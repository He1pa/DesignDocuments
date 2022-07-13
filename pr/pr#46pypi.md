<!-- Thank you for contributing to KusionStack!

Note: 

1. With pull requests:

    - Open your pull request against "main"
    - Your pull request should have no more than two commits, if not you should squash them.
    - It should pass all tests in the available continuous integration systems such as GitHub Actions.
    - You should add/modify tests to cover your proposed code changes.
    - If your pull request contains a new feature, please document it on the README.

2. Please create an issue first to describe the problem.

    We recommend that link the issue with the PR in the following question.
    For more info, check https://kusionstack.io/docs/governance/contribute/
-->

#### 1. Does this PR affect any open issues?(Y/N) and add issue references (e.g. "fix #123", "re #123".):

- [ ] N
- [x] Y fix #46 https://github.com/KusionStack/KCLVM/issues/46

<!-- You can add issue references here. 
    e.g. 
    fix #123, re #123, 
    fix https://github.com/XXX/issues/44
-->

#### 2. What is the scope of this PR (e.g. component or file name):

<!-- You can add the scope of this change here. 
    e.g. 
    /src/server/core.rs,
    kusionstack/KCLVM/kclvm-parser 
-->
add:
+ kclvm/internal/MANIFEST.in
+ kclvm/internal/setup.py
+ empty '\_\_init\_\_.py' files in dir: kclvm_py

modify:
+ scripts/build-kclvm/requirements.release.txt
+ kclvm/internal/kclvm_py/scripts/requirements.txt

#### 3. Provide a description of the PR(e.g. more details, effects, motivations or doc link):

<!-- You can choose a brief description here -->
- [ ] Affects user behaviors
- [ ] Contains syntax changes
- [ ] Contains variable changes
- [ ] Contains experimental features
- [ ] Performance regression: Consumes more CPU
- [ ] Performance regression: Consumes more Memory
- [x] Other

Use setuptools and twine to build KCLVM python package, and upload it to PyPI.
steps:
1. cd internal
2. modify kclvm version in setup.py
3. run `python setup.py sdist` to build package
4. run `twine upload dist/kclvm-0.x.x` to upload package to PyPI
5. input username and password of PyPI

PyPI project: https://pypi.org/project/kclvm/

<!-- You can add more details here.
    e.g. 
    Call method "XXXX" to ..... in order to ....,
    More details: https://XXXX.com/doc......
-->

#### 4. Are there any breaking changes?(Y/N) and describe the breaking changes(e.g. more details, motivations or doc link):

- [x] N
- [ ] Y 

<!-- You can add more details here.
    e.g. 
    Calling method "XXXX" will cause the "XXXX", "XXXX" modules to be affected.
    More details: https://XXXX.com/doc......
-->

#### 5. Are there test cases for these changes?(Y/N) select and add more details, references or doc links:

<!-- You can choose a brief description here -->
- [ ] Unit test
- [ ] Integration test
- [x] Manual test (add detailed scripts or steps below)
- [ ] Other
steps:
1. cd internal
2. modify kclvm version in setup.py
3. run `python setup.py sdist` to build package
4. run `twine upload dist/kclvm-0.x.x` to upload package to PyPI
5. input username and password of PyPI

<!-- You can add more details here.
e.g. 
The test case in XXXX is used to .....
test cases in /src/tests/XXXXX
test cases https://github.com/XXX/pull/44
-->

#### 6. Release note

<!-- compatibility change, improvement, bugfix, and new feature need a release note -->

Please refer to [Release Notes Language Style Guide](https://kusionstack.io/docs/governance/release-policy/) to write a quality release note.

```release-note
None
```