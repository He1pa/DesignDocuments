# KCL Lint 
## 1 Background
- An indispensable tool for programming languages: code checking tools. Code checking is available for most programming languages, and generally compilers have built-in checking tools.
- KCL Lint is designed to help users to coding more compliant with the KCL code specification, detect potential problems in the code, help enforce KCL coding standards, and provide features such as simple refactoring suggestions or automatic fixes.
- Use the lint tool to build ci tests for internal and external [Konfig repositories](https://github.com/KusionStack/konfig)  to ensure the quality of the code.

## 2 Design
### 2.1 Difference with python ver.

The python version of KCL Lint follows the design of pylint and is a compiler-independent tool. The Lint has indepent command line parsing. In main fuction, Linter calls the `parse_program()` function to obtain the AST, and then checks the AST, generates Lint messages and output them. Therefore, the python version of KCL Lint needs to deal with many problems by itself, such as parameter parsing, path processing, file traversal, etc. In fact, these work has already been handled once in KCLVM. At the same time, the Lint tool also maintains another error messages and error output system, which is somewhat separated from the main body of KCLVM, and also required cost to maintain. Moreover, for some Warning level information, the user will not be prompted during the compilation process. They need to run the lint command separately to see it.

![](../images/KCL_Lint_tool(Rustver.)/kcllint_py.jpg)

The KCL Lint rust version is based on the design of Rustc's Resolver and Lint, and executes lint checking during the semantic phase of compilation. Lint shares a common error handling system with the rest of the KCLVM. Lint tool checks AST, generates diagnostics and inserts them into `handler.diagnostics`, which are handled by the KCLVM. This avoids the need to maintain an additional set of error messages, and also throws the Lint-checked problems at compile time without the need to perform additional lint checks. If a separate lint check is required, the diagnostics are thrown after the lint check and the program exits. This avoids maintaining an additional set of error messages, and also emit the problems detected by Lint at compile time. If only lint checking is required, KCLVM will emit diagnostics after lint checking and exits the main program.

![](../images/KCL_Lint_tool(Rustver.)/kcllint_rust.jpg)

When checking, the python ver. Lint divides different checks into multiple checkers by AST type, such as `ImportCheck`, `BaseChecker` etc. Each checker needs to traverse AST at least once. The Rust version is no longer divided into multiple checkers, but collects all lint checks by AST type in a `CombinedLintPass` structure. When traversing the AST node, Lint calls the check method of `CombinedLintPass`, which run all lint checks in one traversal.

### 2.2 Overall
Lint is executed during the semantic analysis phase, and the main structure consists of `Lint`, `LintPass`, `CombinedLintPass` and `Linter`, which implements the `walker` methods for traversing AST. When traversing AST node, the corresponding check method in `CombinedLintPass` is called. Static information and check methods for each lint are defined in `Lint` and `LintPass` respectively. `CombinedLintPass` aggregates the checks in these LintPasses according to the type of AST node.

### 2.3 Specific design
#### 2.3.1 Lint

`Lint` is a struct type that defines a lint. It is a global identifier and a description of the lint, which contains some stastic information about the lint (name, level, error message, error code, examples, etc.).

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

The `LintPass` is an implementation of the specific check logic for Lint, containing the check methods that need to be called when traversing the AST tree. LintPass is defined as a trait, which needs to be implemented for each definition of lintpass. Not every Lint needs to check all ASTs, but only the check methods required by the Lint need to be overridden. So when defining trait `LintPass`, we give all methods a default implementation, i.e. a null check, and when defining LintPass, just override the partial check function.

Every LintPass needs to implement the get_lint() method to generate the corresponding Lint structure.

```rust
pub trait LintPass{
    fn name(&self);
    fn get_lint();
    fn check_ident(&mut self, a: ast::Ident, &mut diags: IndexSet<diagnostics>){}
    fn check_module(&mut self, a: ast::Module, &mut diags: IndexSet<diagnostics>){}
    fn check_stmt(&mut self, a: ast::Stmt, &mut diags: IndexSet<diagnostics>){}
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

Each LintPass defined separate check function for AST node. Traversing the AST once for each Lint individually would cause a relatively large performance overhead. Therefore, the `CombinedLintPass` structure is defined to aggregate the check methods of all defined LintPasses. By calling the check method of `CombinedLintPass` when traversing the AST, all Lint checks can be done in one traversal.

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
    fn check_ident(&mut self, a: Ident, diags: &mut IndexSet<diagnostics>){
        self.LintPassA.check_ident(a, diags);
        self.LintPassB.check_ident(a, diags);
        ...
    }
    fn check_stmt(&mut self, a: &ast::Stmt, diags: &mut IndexSet<diagnostics>){
        self.LintPassA.check_stmt(a, diags);
        self.LintPassB.check_stmt(a, diags);
        ...
    }
}
```
#### 2.3.4 Linter
`Linter` is a structure that traverses the AST and required to implement mothods of `Walker`. Linter calls the check method of `CombinedLintPass` when traversing the AST.

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
Lint errors are consistent with KCLVM error and are stored as `diagnostics` types, which are handled by the error handling module.

#### 2.3.6 cli参数

#### 2.3.7 lint配置
There are some lint checks that support custom parameters, such as naming convention, code length, etc., and the lint tool itself requires some configuration, such as allowing the user to ignore some checks. Lint configuration is set according to the following priorities.

1. the config file specified by the cli parameter
2. the `.kcllint` file in the directory of the file or folder being checked.
3. default config

The configuration file is written in yaml format, e.g.
```yaml
ignore: ["E0501"]
max_line_length: 120

```
#### 2.3.8 others
##### macro
The definitions of Lint, LintPass and CombinedLintPass have a lot of repetitive code that can be generated using macro definitions. For example, `declare_lint!` and `declare_lint_pass!` are used in rustc to define `Lint` (WHILE_TRUE) and `LintPass` (WHILE_TRUE).
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

And the definition of `BuiltinCombinedEarlyLintPass`
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
- Konfig: [https://github.com/KusionStack/konfig](https://github.com/KusionStack/konfig)


