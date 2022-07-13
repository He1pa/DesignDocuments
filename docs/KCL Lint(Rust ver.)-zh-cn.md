## 1 背景需求
- 语言/代码编写不可或缺的实用工具：代码检查工具。对大多数编程语言来说都会有代码检查，一般来说编译程序会内置检查工具。
- KCL Lint 的设计目标：旨在帮助用户写出更符合 KCL 代码规范的代码，检测出代码中潜在的问题，帮助执行 KCL 编码标准，并提供简单的重构建议或者自动修复等功能。
- 使用 lint 工具，构建内外 Konfig 仓库的 ci 测试，保障仓库代码质量
## 2 设计方案
### 2.1 与py版本的差异
KCL lint 的 python 版本参照了 pylint 的设计，是独立于编译器的工具。Lint 工具有独立的命令行解析，主函数中调用 KCLVM 的 parse 函数获取 AST，然后在 Lint 的系统中对 AST 进行检查、输出。因此， KCL Lint 的 python 版本需要自行处理例如参数解析，跟路径处理，文件遍历等问题。实际上，这些工作在 KCLVM 中已经处理过一次了。同时 Lint 工具还单独维护了错误信息和错误输出模块，与 KCLVM 主体有一些割裂，维护起来也需要成本。而且，对于一些 Warning 级别的信息，编译过程中并不会给用户提示，而是需要单独运行 lint 指令才能看到。
![](../images/KCL_Lint_tool(Rustver.)/kcllint_py.jpg)

KCL Lint rust 版本参考 Rustc 的 Resolver 和 Lint 的设计，把 Lint 工具做到编译的语义检查阶段，与VM 的其他部分共用一套错误处理系统， Lint 检查生成 diagnostics 并插入到`handler.diagnostics`中， 由 VM 统一处理。避免了额外维护一套错误信息，同时也将 Lint 检查出来的问题在编译时抛出，不需要额外执行 lint 检查。如果需要单独进行 lint 检查，则在 lint 检查后抛出 diagnostics 并退出程序。
![](../images/KCL_Lint_tool(Rustver.)/kcllint_rust.jpg)

在执行检查方法，python 版本将 Lint 的检查按功能和种类分为多个 checker，例如 ImportChecker，BaseChecker等。每一个 checker 需要遍历一次 AST。rust 版本不再分为多个checker，而是在一个CombinedLintPass 结构中按照 AST 的种类汇总所有 lint 的检查。在遍历 AST 节点时，调用 CombinedLintPass 的 check 方法，即可在一次遍历中完成所有 lint 的检查。
### 2.2 总体设计
Lint 在语义分析阶段执行，主要结构包含 Lint， LintPass， CombinedLintPass 和 主体 Linter，由 Linter 实现 walker 中遍历 AST 的方法。在遍历每一个 AST 节点时，调用 CombinedLintPass 中对应的 check 方法。Lint 和 LintPass 中分别有每个 lint 的静态信息和检查方法。CombinedLintPass 按照 AST 节点的种类汇总了这些 LintPass 中的检查。
### 2.3 具体设计
#### 2.3.1 Lint
`Lint` 是定义 Lint 的 struct 类型，是 lint 的全局标识和对 lint 的描述，包含该 lint 的一些 stastic 信息(名称、等级、错误信息，错误代码，例子等）。
```rust
pub struct Lint {
    /// A string identifier for the lint.
    pub name: &'static str,

    /// Level for the lint.
    pub level: Level,

    /// Description of the lint or the issue it detects.
    /// e.g., "imports that are never used"
    pub desc: &'static str,
    
    // Error/Warning code
    pub code: DiagnosticId,
}
```
#### 2.3.2 LintPass
`LintPass` 是 Lint 的具体检查、判断逻辑的实现，包含遍历 AST 树时需要调用的 check 方法。LintPass 表现为一个 trait，每一个 lintpass 的定义都需要实现该trait。并非每一个 Lint 都需要检查所有 AST，而是只需要重写该 Lint 所需要的 check 方法。所以在实现时，需要给 LintPass 一个默认的实现方法,即空检查，在定义 LintPass 时，重写部分检查的函数即可。
LintPass 需要实现 get_lint() 方法生成对应的 Lint 结构。
```rust
pub trait LintPass{
    fn name(&self);
    fn get_lint();
    fn check_ident(&mut self, a: ast::Ident, diags: &mut IndexSet<diagnostics>){}
    fn check_module(&mut self, a: ast::Module, diags: &mut IndexSet<diagnostics>){}
    fn check_stmt(&mut self, a: ast::Stmt, diags: &mut IndexSet<diagnostics>){}
    ...
}

pub struct LintPassA{
    name: str
}

pub struct LintPassB{
    name: str
}

impl LintPass for LintPassA{
    fn name(){..}
    fn get_lint(){...}
    fn check_ident(&mut self, a: ast::Ident, diags: &mut IndexSet<diagnostics>){
        ...
    }
}


impl LintPass for LintPassB{
    fn name(){..}
    fn get_lint(){...}
    fn check_stmt(&mut self, a: ast::Stmt, diags: &mut IndexSet<diagnostics>){
        ...
    }
}

```
#### 2.3.3 CombinedLintPass
每一个 LintPass 中定义了单独的对 AST 节点检查的函数。每一个 Lint 单独遍历一次 AST 会造成比较大的性能开销。因此定义 `CombinedLintPass` 结构，对所有定义的 LintPass 的 check 方法进行汇总。在遍历 AST 时，调用 CombinedLintPass 的 check 方法，就可以在一次遍历中完成所有 Lint 检查。
```rust
pub struct CombinedLintPass {
    LintPassA: LintPassA;
    LintPassB: LintPassB;
    ...
}

impl CombinedLintPass{
    pub fn new() -> CombinedLintPass { 
        CombinedLintPass {
            LintPassA: LintPassA::new(),
            LintPassB: LintPassB::new(),
            ...
        } 
    }
}

impl LintPass for CombinedLintPass {
    fn check_ident(&mut self, a: Ident, &mut diags: IndexSet<diagnostics>){
        self.LintPassA.check_ident(a, diags);
        self.LintPassB.check_ident(a, diags);
        ...
    }
    fn check_stmt(&mut self, a: &ast::Stmt, &mut diags: IndexSet<diagnostics>){
        self.LintPassA.check_stmt(a, diags);
        self.LintPassB.check_stmt(a, diags);
        ...
    }
}
```
#### 2.3.4 Linter
`Linter` 是遍历 AST 的结构，需要实现 Walker 的方法。 Linter 在遍历 AST 时调用 CombinedLintPass 的check 方法。
```rust
pub struct Linter<T: LintPass> {
    pass: T,
    diags: diagnostics
}

impl Checker{
    fn new() -> Checker{
        Checker{
            pass: CombinedLintPass::new()
            diags: Default::default()
        }
    }
}

impl ast_walker::Walker for Linter{
    fn walk_ident(&self, a: ast::Ident, diags: &mut IndexSet<diagnostics>){
        pass.check_ident(a, diags);
        walk_subAST();
    }
    fn walk_stmt(&self, a: ast::Ident, diags: &mut IndexSet<diagnostics>){
        pass.check_stmt(a, diags);
        walk_subAST();
    }
}
```
#### 2.3.5 lint message
与 KCLVM 保持一致，存储为 diagnostics 类型，由错误处理模块统一处理。
#### 2.3.6 cli参数

#### 2.3.7 lint配置
有一些 lint 检查支持自定义参数，例如命名规范、代码长度等，同时 lint 工具自身也需要一些配置，例如允许用户忽略某些检查。lint 配置按照以下优先级设置：

1. cli参数指定的配置文件
1. 被检查文件或文件夹的目录下的`.kcllint`文件，
1. 默认配置

配置文件以yaml格式书写，例如：
```yaml
ignore: ["E0501"]
max_line_length: 120

```
#### 2.3.8 其他
##### 宏
Lint, LintPass 以及 CombinedLintPass 的定义中有大量重复代码，可以利用宏定义去生成。例如 rustc 中用`declare_lint!`和`declare_lint_pass!`两个宏去定义 Lint（WHILE_TRUE） 和 LintPass（WHILE_TRUE）。
```rust
declare_lint! {
    /// The `while_true` lint detects `while true { }`.
    ///
    /// ### Example
    ///
    /// ```rust,no_run
    /// while true {
    ///
    /// }
    /// ```
    ///
    /// {{produces}}
    ///
    /// ### Explanation
    ///
    /// `while true` should be replaced with `loop`. A `loop` expression is
    /// the preferred way to write an infinite loop because it more directly
    /// expresses the intent of the loop.
    WHILE_TRUE,
    Warn,
    "suggest using `loop { }` instead of `while true { }`"
}

declare_lint_pass!(WhileTrue => [WHILE_TRUE]);

impl EarlyLintPass for WhileTrue{
  ...
}
```

以及 BuiltinCombinedEarlyLintPass 的定义
```rust
early_lint_passes!(declare_combined_early_pass, [BuiltinCombinedEarlyLintPass]);

// Expand all macros
pub struct BuiltinCombinedEarlyLintPass {
    UnusedParens: UnusedParens;
    UnusedBraces: UnusedBraces;
    ...
}

impl BuiltinCombinedEarlyLintPass{
    pub fn new() -> Self {
        UnusedParens: UnusedParens,
        UnusedBraces: UnusedBraces
        ...
    }
    
    pub fn get_lints() -> LintArray {
        let mut lints = Vec::new();
        lints.extend_from_slice(&UnusedParens::get_lints());
        lints.extend_from_slice(&$UnusedBraces::get_lints());
        ...
        lints
    }
}

impl EarlyLintPass for BuiltinCombinedEarlyLintPass {
    fn check_ident(&mut self, context: &EarlyContext<'_>, a: Ident){
        self.UnusedParens.check_ident(context, a: Ident);
        self.UnusedBraces.check_ident(context, a: Ident);
        ...
    }
    fn check_crats(&mut self, context: &EarlyContext<'_>, a: &ast::Crate){
        self.UnusedParens.check_crats(context, a: Crate);
        self.UnusedBraces.check_crats(context, a: Crate);
        ...
    }
}
```
## Ref

- RustcDevGuide: [https://rustc-dev-guide.rust-lang.org/](https://rustc-dev-guide.rust-lang.org/)
- Rustc: [https://github.com/rust-lang/rust.git](https://github.com/rust-lang/rust.git)



