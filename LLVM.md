# Why We Use LLVM

LLVM is a widely used compiler infrastructure that provides a reusable **intermediate representation (LLVM IR)** and a large collection of optimization and code-generation tools. Instead of requiring us to generate machine code directly for a specific processor, our TinyLang compiler translates programs into LLVM IR. LLVM can then handle many lower-level tasks for us.

Conceptually, our compiler pipeline becomes:

```text
TinyLang source code
        |
        v
      AST
        |
        v
   LLVM IR
        |
        v
      LLVM
   /    |     \
  v     v      v
x86   ARM   other targets
```

This separation is powerful because the front end of the compiler can focus on the **source language**—its syntax, types, variables, and semantics—while LLVM handles much of the machine-specific complexity.

# Why LLVM Is Powerful

LLVM provides several important capabilities:

- **Portable intermediate representation.** LLVM IR is independent of a particular source language and can be translated to many processor architectures.
- **Reusable code generation.** A compiler does not need to implement separate machine-code generators for x86, ARM, and other architectures.
- **Optimization infrastructure.** LLVM includes many optimization passes for simplifying and improving generated programs.
- **Well-defined low-level operations.** Instructions such as `add`, `load`, `store`, `icmp`, and `ret` make the connection between high-level language constructs and machine-level execution easier to study.
- **Real-world relevance.** LLVM is used by major compiler and programming-language projects, including Clang and the Rust compiler.

For example, TinyLang code such as:

```text
let x: int = 10;
let y: int = 20;
return x + y;
```

may eventually become LLVM IR similar to:

```llvm
%x = alloca i32
%y = alloca i32

store i32 10, ptr %x
store i32 20, ptr %y

%x.value = load i32, ptr %x
%y.value = load i32, ptr %y
%result = add i32 %x.value, %y.value

ret i32 %result
```

Once this LLVM IR has been generated, LLVM can translate it further into native machine code.

Using LLVM therefore lets this project demonstrate an important real-world compiler architecture:

```text
Source Language
      |
      v
Compiler Front End
      |
      v
Intermediate Representation
      |
      v
Compiler Back End
      |
      v
Machine Code
```

In this project, we implement the TinyLang-specific front end and a small translation to LLVM IR, while relying on LLVM for the lower-level compiler infrastructure. This allows us to study the complete compiler pipeline without having to build an entire machine-code backend from scratch.

