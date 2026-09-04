# Coding Assignment 2: TinyLang Semantic Analysis and LLVM IR

## 1. Overview

Coding Assignment 2 continues directly from Coding Assignment 1.
Assignment 1 answered: `Does the program follow TinyLang's grammar?`
Assignment 2 asks: `Does the syntactically valid program make sense?`
and, if it does: `How can we translate it into LLVM IR?`

The pipeline is:

```text
source
  |
  v
Lexer
  |
  v
Parser
  |
  v
AST
  |
  v
+--------------------+
| Semantic Analyzer  |   Part 1
+--------------------+
  |
  | semantically valid AST
  v
+--------------------+
| LLVM IR Generator  |   Part 2
+--------------------+
  |
  v
LLVM IR
```

You will primarily edit:
`src/Semantic.cpp` and 
`src/CodeGen.cpp`

Search for:

```cpp
TODO(A2)
```

The assignment is intentionally weighted toward **semantic analysis**. LLVM code generation is kept small so you can experience the complete compiler pipeline without building a large backend.

---

# 2. Before starting

Your frontend must be able to construct an AST before semantic analysis can run.
Use either:
- your working Assignment 1 implementation, or
- an instructor-provided reference frontend

Do not redesign the parser for Assignment 2.
The important input to this assignment is the AST.

---

## 2.1 Syntax versus semantics

Consider the following code in TinyLang:

```text
fn main(): int {
    let x: int = 10;
    x = true;
    return x;
}
```

The program is grammatically well formed. Therefore the parser can construct an AST.
However:
`x = true;`
is invalid because:

```text
x    : int
true : bool
```

This is a **semantic error** (type mismatched).
Another example:

```text
fn main(): int {
    y = 10;
    return 0;
}
```

The grammar allows:
`Identifier "=" expression ";"`
so the statement is syntactically valid.
But `y` was never declared, so it is semantically invalid.

---

## 2.2 Static semantic analysis

TinyLang performs semantic checking before execution.
Required checks include:
- variable declared before use
- no duplicate declaration in the same scope
- assignment type compatibility
- valid operator operand types
- boolean `if` conditions
- return type compatibility

The semantic analyzer does not execute the program. It reasons about the AST and the rules of the language.

---

# 3. Files for Assignment 2

Read:

```text
include/AST.h
include/Semantic.h
include/CodeGen.h
src/main.cpp
```

Primarily edit:

```text
src/Semantic.cpp
src/CodeGen.cpp
```

The parser and AST representation should already be working.


---
# 4. Part 1: Semantic Analyzer

---

## 4.1 Symbol tables

The starter contains:
`std::vector<std::unordered_map<std::string, Type>> scopes_;`
This is a stack of symbol tables.
Each table maps:
`variable name -> declared type`

For example:
```text
x     -> int
flag  -> bool
```

When the analyzer later encounters:
`x = x + 1;`,
the symbol table tells it:
- `x` exists
- `x` has type `int`

---

## 4.2 Lexical scope

Blocks (i.e., { ... }) create scopes.
For example:

```text
fn main(): int {
    let x: int = 10;

    if (x < 20) {
        let y: int = 5;
        x = y;
    }

    return x;
}
```

Inside the `if` block, both `x` and `y` are visible.
After leaving that block, `x` remains visible but `y` does not.
Conceptually:

```text
main block:
    x : int

    if block:
        y : int
```

Lookup begins in the innermost scope and proceeds outward.

---

## 4.3 Scope operations

The starter provides: 
`enterScope()` and 
`leaveScope()`. 
These push and pop symbol tables.
For nested blocks:

```text
// outer
{ // block1
    let x: int = 10;

    { // block2
        let y: int = 20;
    }
}
```

the stack may evolve as:
`[outer]` (outside of any blocks) 
then:
`[outer, block1]`
then:
`[outer, block1, block2]`
After leaving `block2`:
`[outer, block1]`, and
`y` is no longer visible.

---

## 4.4 Implement `declare()`

Complete the function:
`bool SemanticAnalyzer::declare(const std::string& name, Type type)`
It should insert into the **current scope**.
Reject duplicate declarations in that same scope.
An invalid example:

```text
let x: int = 10;
let x: bool = true;
```

Do not necessarily search every outer scope when checking duplicates. A nested scope may shadow an outer variable unless the instructor specifies otherwise.

Example:

```text
let x: int = 10; // outer definition of x

if (x < 20) {
    let x: int = 5; // inner definition of x
}
```

can be treated as valid shadowing.

---

## 4.5 Implement `lookup()`

Complete the function:
`Type SemanticAnalyzer::lookup(const std::string& name) const`
The order to search the proper scope is:
```text
innermost scope -> ... -> outermost scope
```

Return the first matching declaration.
If not found, return:
`Type::Error`. 
The caller can then report an undeclared-variable error.

---

## 4.6 Implement `checkExpr()`

Complete:
`Type SemanticAnalyzer::checkExpr(const Expr& expr)`
This function determines the static type of an expression.
For examples:

```text
10       -> int
true     -> bool
x        -> type found in symbol table
x + 1    -> int if x is int
x < 1    -> bool if x is int
```

---

## 4.7 Literal and variable types

Since TinyLang only has two types, you will need to handle: 
- Integer: `IntegerExpr -> Type::Int`
- Boolean: `BoolExpr -> Type::Bool`
- Variable: Use function `lookup(name)`. If no declaration exists:
    1. report an error
    2. return `Type::Error`. An example error message is: `use of undeclared variable 'x'`.

---

## 4.8 Binary type rules

### Arithmetic

Consider operators:
```text
+
-
*
/
```

which require the following form to use the operators:
`int op int`,
and produce the result:
`int`. 

### Comparison

Consider operators:
```text
<
>
```

which require the following form to use the operators:
`int op int`,
and produce the result:
`bool` 

### Equality
Consider operator:
`==` 
which requires both operands to have the same type and produces the result:
`bool`

Foe examples:

```text
10 == 20       valid
true == false  valid
10 == true     invalid
```

### Logical operators
Consider operators:
```text
&&
||
```

which require the following form to use the operators:
`bool op bool`, 
and produce the result:
`bool`.


---

## 4.9 Type-rule table
Use this table directly when implementing `checkExpr()`.

| Expression | Required operand types | Result |
|---|---|---|
| `a + b` | `int`, `int` | `int` |
| `a - b` | `int`, `int` | `int` |
| `a * b` | `int`, `int` | `int` |
| `a / b` | `int`, `int` | `int` |
| `a < b` | `int`, `int` | `bool` |
| `a > b` | `int`, `int` | `bool` |
| `a == b` | same type | `bool` |
| `a && b` | `bool`, `bool` | `bool` |
| `a || b` | `bool`, `bool` | `bool` |


---

## 4.10 Implement `checkStmt()`

Complete function: 
`void SemanticAnalyzer::checkStmt(const Stmt& stmt)`
It should handle the supplied AST statement types.

---

## 4.11 Variable declaration rules

For example: `let x: int;`,
the analyzer should:
1. verify `x` is not already declared in the current scope
2. record `x : int`

For example: `let x: int = 10;`,
the analyzer should do two more tasks based on the above two:
3. type-check the initializer
4. require the initializer type to match the declared type

An valid example:
`let x: int = 10;`

An invalid example:
`let x: int = true;`

---

## 4.12 Assignment rules
For example: `x = expression;`, 
the analyzer should:
1. look up `x`
2. reject the assignment if `x` is undeclared
3. determine the right-hand-side type
4. require both types to match

An valid example:

```text
let x: int = 10;
x = 20;
```

An invalid example if type mismatched:

```text
let x: int = 10;
x = false;
```

Another invalid example if undeclared:

```text
y = 10;
```

---

## 4.13 Return rules

TinyLang's function is of the following form: `fn main(): int`. 
Therefore the return expression must have type `int`.

An valid example:
`return 0;`

An invalid example:
`return true;`

The starter stores the expected return type in:
`currentReturnType_`. 
Use this value rather than scattering hard-coded type assumptions through your code.

---

## 4.14 `if` rules

The condition of:

```text
if (condition) {
    ...
}
```

must have type: `bool`. 

An valid example:

```text
if (x < 10) {
    ...
}
```

An invalid example:

```text
if (10) {
    ...
}
```

Notice that `if (10)` was grammatically valid in Assignment 1. It becomes invalid only during semantic analysis.

Analyze both the `then` block (true branch) and optional `else` block (false branch) using proper nested scopes.

---

## 4.15 Semantic error reporting

The starter provides function:
`report(...)` to generate useful error messages.
For examples:

```text
duplicate declaration of variable 'x'
```

```text
use of undeclared variable 'y'
```

```text
cannot assign bool to variable 'x' of type int
```

```text
operator '+' requires int operands
```

```text
if condition must have type bool
```

```text
return expression has type bool; expected int
```

Where possible, continue checking after an error so one compilation can report multiple semantic problems.

---

## 4.16 Semantic tests
Run semantic tests with:

```bash
./tinylang program.tl --check
```

You can try your code with the following test cases: 

### Undeclared variable

```text
fn main(): int {
    x = 10;
    return 0;
}
```

Must fail.

### Duplicate declaration

```text
fn main(): int {
    let x: int = 10;
    let x: int = 20;
    return x;
}
```

Must fail.

### Assignment mismatch

```text
fn main(): int {
    let x: int = 10;
    x = true;
    return x;
}
```

Must fail.

### Operator mismatch

```text
fn main(): int {
    let x: int = true + 1;
    return 0;
}
```

Must fail.

### Incorrect `if` condition

```text
fn main(): int {
    if (10) {
        return 1;
    }
    return 0;
}
```

Must fail.

## Incorrect return type

```text
fn main(): int {
    return false;
}
```

Must fail.


---

# 5. Part 2: LLVM IR generation

After semantic analysis succeeds, you will translate the AST into LLVM IR.
Relevant implementation is in:
`src/CodeGen.cpp`. 
This is intentionally a small backend.
The purpose is to understand how AST operations map to lower-level IR instructions, not to implement all LLVM features.

---

## 5.1 LLVM objects already supplied

The starter creates:

```cpp
llvm::LLVMContext
llvm::Module
llvm::IRBuilder<>
```

where:
- `LLVMContext`: Owns LLVM's core type/constant infrastructure.
- `Module`: Represents the LLVM IR module being generated.
- `IRBuilder`: Creates LLVM instructions at the current insertion point.

You do not need to initialize LLVM infrastructure from scratch.

---

## 5.2 The LLVM `main` function

`CodeGenerator::generate()` creates an LLVM function corresponding to: `fn main(): int`,
with type conceptually: `i32 ()` and creates an entry basic block.
Your main job is to translate TinyLang statements and expressions inside that body.

---

## 5.3 Variables: `alloca`, `store`, and `load`

In TinyLang:
`let x: int = 10;`
can be represented as:

```llvm
%x = alloca i32
store i32 10, ptr %x
```

To read `x`, you need to create an LLVM IR instruction:
```llvm
%tmp = load i32, ptr %x
```

The starter has:
`std::unordered_map<std::string, llvm::AllocaInst*> variables_;` which
conceptually means:
```text
"x" -> pointer to %x stack slot
"y" -> pointer to %y stack slot
```

---

## 5.4 Generating declarations

For example:
`let x: int = 10;`, 
generates the following tasks:
1. an `alloca i32`
2. an entry in `variables_`
3. the initializer value
4. a `store`

The starter includes a suggested `CreateAlloca` call.
Because semantic analysis already succeeded, the code generator may assume source-level type rules have been checked.

---

## 5.5 Generating assignment

For example:
`x = x + 1;` 
conceptually introduces the following tasks:

1. locate `x`'s stack slot
2. generate `x + 1`
3. store the result back

A set of possible generated LLVM IR is:

```llvm
%1 = load i32, ptr %x
%2 = add i32 %1, 1
store i32 %2, ptr %x
```

---

## 5.6 Generating integer literals

For example:
`10`
uses an LLVM constant.
The starter suggests to use:
`llvm::ConstantInt::get(llvm::Type::getInt32Ty(*context_), value, true)`.

In TinyLang, type `int` maps to: `i32`.

---

## 5.7 Generating variable expressions

For example:
`x`
finds its slot in:
`variables_` and 
then generate a load:
`builder_->CreateLoad(...)`.
The load result is an LLVM `Value*` usable by later instructions.

---

## 5.8 Arithmetic instructions

| TinyLang | LLVM builder |
|---|---|
| `+` | `CreateAdd` |
| `-` | `CreateSub` |
| `*` | `CreateMul` |
| `/` | `CreateSDiv` |

For exmple:
`x + y`, 
you may generate conceptually:
```llvm
%1 = load i32, ptr %x
%2 = load i32, ptr %y
%3 = add i32 %1, %2
```
where the last `add` instruction is created by calling `CreateAdd`. 

---

## 5.9 Comparison instructions

Similar to arithmetic operations, other useful operations are:

| TinyLang | LLVM builder |
|---|---|
| `<` | `CreateICmpSLT` |
| `>` | `CreateICmpSGT` |
| `==` | `CreateICmpEQ` |

An integer comparison usually produces LLVM type: `i1` which naturally represents a boolean result.

---

## 5.10 Generating return

For example:
`return x;`, 
you will firstly generate `x`'s value, then call:
`builder_->CreateRet(value);`. 
Because TinyLang `main` returns `int`, the returned LLVM value must be compatible with:
`i32` (LLVM type). 

---

## 5.11. Required and extended backend features

The semantic analyzer should understand the TinyLang subset described above.
The LLVM portion is intentionally smaller.
Core code-generation work focuses on:
- integer literals
- integer variables
- declarations
- assignment
- arithmetic operators including `+`, `-`, `*`, `/`
- required integer comparisons
- return

Depending on your schedule and potential, these may be *extra credit*:
- stored boolean variables
- `&&`
- `||`
- full `if/else` control-flow generation

where `if/else` generation is more involved because it introduces:
- multiple basic blocks
- conditional branches
- merge blocks
- builder insertion-point management

---

## 6. Example translation

TinyLang:

```text
fn main(): int {
    let x: int = 10;
    let y: int = 20;
    x = x + y * 2;
    return x;
}
```

Possible LLVM IR:

```llvm
define i32 @main() {
entry:
  %x = alloca i32
  %y = alloca i32

  store i32 10, ptr %x
  store i32 20, ptr %y

  %x1 = load i32, ptr %x
  %y1 = load i32, ptr %y

  %mul = mul i32 %y1, 2
  %add = add i32 %x1, %mul

  store i32 %add, ptr %x

  %x2 = load i32, ptr %x
  ret i32 %x2
}
```

Your temporary names (e.g., `%x1`) may differ, which is normal.

---

# 7. Inspecting generated LLVM IR (the most fun part!!!)

After building your code in starter, you will see the executatble `tinylang`. Run:

```bash
./tinylang ../examples/codegen.tl --emit-llvm
```

Redirect to a file:

```bash
./tinylang ../examples/codegen.tl --emit-llvm > output.ll
```

If LLVM command-line tools are installed (should be installed together with all other LLVM packages), validate it with:

```bash
llvm-as output.ll -o output.bc
```

You can also compile suitable LLVM IR:

```bash
clang output.ll -o output
```

and run:

```bash
./output
```

The return value is normally observed as the process exit status rather than printed text.

---

# 8. Recommended implementation order

## Semantic analysis first

1. Implement `declare()`.
2. Implement `lookup()`.
3. Handle integer/boolean literals in `checkExpr()`.
4. Handle variable expressions.
5. Implement binary-expression type rules.
6. Implement declaration checking.
7. Implement assignment checking.
8. Implement return checking.
9. Implement `if` checking and nested scopes.
10. Test thoroughly with `--check`.

## LLVM generation second

11. Emit integer constants.
12. Emit integer variable `alloca`.
13. Emit initializer `store`.
14. Emit variable `load`.
15. Emit arithmetic expressions.
16. Emit assignment stores.
17. Emit returns.
18. Add required comparisons.
19. Add any instructor-assigned extension.

Do not start with LLVM before semantic analysis works.

---

# 9. Debugging by compiler phase

Use the pipeline to localize bugs.

```text
Wrong token stream
    -> Lexer.cpp

Correct tokens but wrong AST
    -> Parser.cpp

Correct AST but semantic result is wrong
    -> Semantic.cpp

Semantic analysis succeeds but IR is wrong
    -> CodeGen.cpp
```

This is much more effective than changing several phases at once.

---

# 10. Important architecture principle

Do not use LLVM code generation as the source-language type checker.
The intended architecture is:

```text
AST
 |
 v
Semantic Analyzer
 |
 | only if valid
 v
Code Generator
```

For example:

```text
let x: int = true;
```

should be rejected before code generation.
The semantic analyzer enforces TinyLang's source-language rules. The LLVM back end assumes those rules have already been checked.

---

# 11. Common mistakes

## Looking up names only in the current scope

Nested blocks must still see outer variables.
Lookup searches outward.

## Rejecting shadowing by searching all scopes during declaration

Duplicate declaration checking should normally inspect only the current scope.

## Treating comparison results as `int`

Source-level comparisons produce `bool`.

## Restricting `==` to integers only

TinyLang permits:

```text
10 == 20
true == false
```

but rejects:

```text
10 == true
```

## Forgetting nested blocks

Both `thenBlock` and `elseBlock` must be semantically analyzed.

## Forgetting to load a variable

`variables_` stores the pointer created by `alloca`. Arithmetic usually requires the loaded value.

## Confusing `store` and SSA values

For this simple back end:

```text
alloca -> storage location
store  -> write/update
load   -> read current value
```

---

# 12. Completion checklist

## Semantic analyzer

- [ ] `declare()` inserts into the current scope
- [ ] duplicate declarations in one scope are rejected
- [ ] `lookup()` searches inner to outer scopes
- [ ] integer literals have type `int`
- [ ] boolean literals have type `bool`
- [ ] undeclared variables are rejected
- [ ] arithmetic requires integers
- [ ] comparisons require integers and return `bool`
- [ ] equality requires compatible matching types
- [ ] logical operators require booleans
- [ ] initializer types are checked
- [ ] assignment types are checked
- [ ] return types are checked
- [ ] `if` conditions require `bool`
- [ ] nested blocks use proper scopes
- [ ] useful semantic errors are reported

## LLVM generator

- [ ] integer constants can be emitted
- [ ] integer variables receive stack storage
- [ ] initializers are stored
- [ ] variable references generate loads
- [ ] assignments generate stores
- [ ] `+` generates addition
- [ ] `-` generates subtraction
- [ ] `*` generates multiplication
- [ ] `/` generates signed division
- [ ] required comparisons generate correct LLVM comparisons
- [ ] `return` generates a valid return instruction
- [ ] `--emit-llvm` prints the generated module

Most importantly, be able to explain:

```text
AST
 |
 | source-language semantic rules
 v
semantically valid AST
 |
 | translation rules
 v
LLVM IR
```

and why semantic analysis and code generation are separate compiler phases.
