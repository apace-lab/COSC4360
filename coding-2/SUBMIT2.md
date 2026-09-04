# Coding Assignment 2: What To Submit

Submit the source files that you modified for Coding Assignment 2, primarily `src/Semantic.cpp` and `src/CodeGen.cpp`. Your implementation should work with the provided TinyLang front end and should not require changes to the supplied compiler interfaces. Do not submit build directories, executables, generated LLVM bitcode, or temporary output files.

You must also submit a `tests/` directory containing **your own TinyLang programs** for testing semantic analysis and LLVM IR generation. Include both semantically valid and invalid programs, and include programs that exercise different symbol-table, scope, type-checking, and code-generation behaviors. Include a short `README.md` describing the purpose of each test, whether semantic analysis should succeed or fail, and, for code-generation tests, the expected result or behavior. Your test suite will be considered as part of the assignment grade because a good compiler implementation should be accompanied by tests that demonstrate understanding of both common and corner cases.


# Grading Policy

| Category | Weight |
|---|---:|
| Baseline cases from starter | 30% |
| Student-created test quality | 15% |
| Additional common cases | 20% |
| Corner cases | 20% |
| LLVM execution cases | 15% |


## Extra credits

Extra 20 pts if you implemented the following correctly with your test cases: 

- `bool` LLVM values
- `&&` and `||`
- `if/else` control-flow generation
