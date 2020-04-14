# Overview of the Compiler

This chapter is about the overall process of compiling a program -- how
everything fits together.

The rust compiler is special in two ways: it does things to your code that
other compilers don't do (e.g. borrow checking) and it has a lot of
unconventional implementation choices (e.g. queries). We will talk about these
in turn in this chapter, and in the rest of the guide, we will look at all the
individual pieces in more detail.

## What the compiler does to your code

So first, let's look at what the compiler does to your code. For now, we will
avoid mentioning how the compiler implements these steps except as needed;
we'll talk about that later.

**TODO: Would be great to have a diagram of this once we nail down the details...**

**TODO: someone else should confirm this vvv**

- The compile process begins when a user writes a Rust source program in text
  and invokes the `rustc` compiler on it. The work that the compiler needs to
  perform is defined by command-line options. For example, it is possible to
  enable nightly features (`-Z` flags), perform `check`-only builds, or emit
  LLVM-IR rather than executable machine code. The `rustc` executable call may
  be indirect through the use of `cargo`.
- Command line argument parsing occurs in the [`librustc_driver`]. This crate
  defines the compile configuration that is requested by the user and passes it
  to the rest of the compilation process as a [`rustc_interface::Config`].
- The raw Rust source text is analyzed by a low-level lexer located in
  [`librustc_lexer`]. At this stage, the source text is turned into a stream of
  atomic source code units known as _tokens_. (**TODO**: chrissimpkins - Maybe
  discuss Unicode handling during this stage?)
- The token stream passes through a higher-level lexer located in
  [`librustc_parse`] to prepare for the next stage of the compile process. The
  [`StringReader`] struct is used at this stage to perform a set of validations
  and turn strings into interned symbols.
- (**TODO**: chrissimpkins - Expand info on parser) We then [_parse_ the stream
  of tokens][parser] to build an Abstract Syntax Tree (AST).
  - macro expansion (**TODO** chrissimpkins)
  - ast validation (**TODO** chrissimpkins)
  - nameres (**TODO** chrissimpkins)
  - early linting (**TODO** chrissimpkins)

- We then take the AST and [convert it to High-Level Intermediate
  Representation (HIR)][hir]. This is a compiler-friendly representation of the
  AST.  This involves a lot of desugaring of things like loops and `async fn`.
- We use the HIR to do [type inference]. This is the process of automatic
  detection of the type of an expression. **TODO: how `ty` module fits in
  here**
- **TODO: Maybe some other things are done here? I think initial type checking
  happens here? And trait solving?**
- The HIR is then [lowered to Mid-Level Intermediate Representation (MIR)][mir].
  - Along the way, we construct the HAIR, which is an even more desugared HIR.
    HAIR is used for pattern and exhaustiveness checking. It is also more
    convenient to convert into MIR than HIR is.
- The MIR is used for [borrow checking].
- **TODO: const eval fits in somewhere here I think**
- We (want to) do [many optimizations on the MIR][mir-opt] because it is still
  generic and that improves the code we generate later, improving compilation
  speed too. (**TODO: size optimizations too?**)
  - MIR is a higher level (and generic) representation, so it is easier to do
    some optimizations at MIR level than at LLVM-IR level. For example LLVM
    doesn't seem to be able to optimize the pattern the [`simplify_try`] mir
    opt looks for.
- Rust code is _monomorphized_, which means making copies of all the generic
  code with the type parameters replaced by concrete types. To do
  this, we need to collect a list of what concrete types to generate code for.
  This is called _monomorphization collection_.
- We then begin what is vaguely called _code generation_ or _codegen_.
  - The [code generation stage (codegen)][codegen] is when higher level
    representations of source are turned into an executable binary. `rustc`
      uses LLVM for code generation.  The first step is the MIR is then
    converted to LLVM Intermediate Representation (LLVM IR). This is where
    the MIR is actually monomorphized, according to the list we created in
    the previous step.
  - The LLVM IR is passed to LLVM, which does a lot more optimizations on it.
    It then emits machine code. It is basically assembly code with additional
    low-level types and annotations added. (e.g. an ELF object or wasm).
    **TODO: reference for this section?**
  - The different libraries/binaries are linked together to produce the final
    binary. **TODO: reference for this section?**

[`librustc_lexer`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_lexer/index.html
[`librustc_driver`]: https://rustc-dev-guide.rust-lang.org/rustc-driver.html
[`rustc_interface::Config`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_interface/interface/struct.Config.html
[lex]: https://rustc-dev-guide.rust-lang.org/the-parser.html
[`StringReader`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_parse/lexer/struct.StringReader.html
[`librustc_parse`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_parse/index.html
[parser]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_parser/index.html
[hir]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_hir/index.html
[type inference]: https://rustc-dev-guide.rust-lang.org/type-inference.html
[mir]: https://rustc-dev-guide.rust-lang.org/mir/index.html
[borrow checking]: https://rustc-dev-guide.rust-lang.org/borrow_check.html
[mir-opt]: https://rustc-dev-guide.rust-lang.org/mir/optimizations.html
[`simplify_try`]: https://github.com/rust-lang/rust/pull/66282
[codegen]: https://rustc-dev-guide.rust-lang.org/codegen.html

## How it does it

Ok, so now that we have a high-level view of what the compiler does to your
code, let's take a high-level view of _how_ it does all that stuff. There are a
lot of constraints and conflicting goals that the compiler needs to
satisfy/optimize for. For example,

- Compilation speed: how fast is it to compile a program. More/better
  compile-time analyses often means compilation is slower.
  - Also, we want to support incremental compilation, so we need to take that
    into account. How can we keep track of what work needs to be redone and
    what can be reused if the user modifies their program?
    - Also we can't store too much stuff in the incremental cache because
      it would take a long time to load from disk and it could take a lot
      of space on the user's system...
- Compiler memory usage: while compiling a program, we don't want to use more
  memory than we need.
- Program speed: how fast is your compiled program. More/better compile-time
  analyses often means the compiler can do better optimizations.
- Program size: how large is the compiled binary? Similar to the previous
  point.
- Compiler compilation speed: how long does it take to compile the compiler?
  This impacts contributors and compiler maintenance.
- Compiler implementation complexity: building a compiler is one of the hardest
  things a person/group can do, and Rust is not a very simple language, so how
  do we make the compiler's code base manageable?
- Compiler correctness: the binaries produced by the compiler should do what
  the input programs says they do, and should continue to do so despite the
  tremendous amount of change constantly going on.
- Compiler integration: a number of other tools need to use the compiler in
  various ways (e.g. cargo, clippy, miri, RLS) that must be supported.
- Compiler stability: the compiler should not crash or fail ungracefully on the
  stable channel.
- Rust stability: the compiler must respect rust's stability guarantees by not
  breaking programs that previously compiled despite the many changes that are
  always going on to its implementation.
- Limitations of other tools: rustc uses LLVM in its backend, and LLVM has some
  strengths we leverage and some limitations/weaknesses we need to work around.

So, as you read through the rest of the guide, keep these things in mind. They
will often inform decisions that we make.

### Constant change

Keep in mind that `rustc` is a real production-quality product.
As such, it has its fair share of codebase churn and technical debt. A lot of
the designs discussed throughout this guide are idealized designs that are not
fully realized yet. And things keep changing so that it is hard to keep this
guide completely up to date on everything!

The compiler definitely has rough edges, but because of its design it is able
to keep up with the requirements above.

### Intermediate representations

As with most compilers, `rustc` uses some intermediate representations (IRs) to
facilitate computations. In general, working directly with the source code is
extremely inconvenient and error-prone. Source code is designed to be human-friendly while at
the same time being unambiguous, but it's less convenient for doing something
like, say, type checking.

Instead most compilers, including `rustc`, build some sort of IR out of the
source code which is easier to analyze. `rustc` has a few IRs, each optimized
for different purposes:

- Token stream: the lexer produces a stream of tokens directly from the source
  code. This stream of tokens is easier for the parser to deal with than raw
  text.
- Abstract Syntax Tree (AST): the abstract syntax tree is built from the stream
  of tokens produced by the lexer. It represents
  pretty much exactly what the user wrote. It helps to do some syntactic sanity
  checking (e.g. checking that a type is expected where the user wrote one).
- High-level IR (HIR): This is a sort of desugared AST. It's still close
  to what the user wrote syntactically, but it includes some implicit things
  such as some elided lifetimes, etc. This IR is amenable to type checking.
- HAIR: This is an intermediate between HIR and MIR. It is like the HIR but it
  is fully typed and a bit more desugared (e.g. method calls and implicit
  dereferences are made fully explicit). Moreover, it is easier to lower to MIR
  from HAIR than from HIR.
- Middle-level IR (MIR): This IR is basically a Control-Flow Graph (CFG). A CFG
  is a type of diagram that shows the basic blocks of a program and how control
  flow can go between them. Likewise, MIR also has a bunch of basic blocks with
  simple typed statements inside them (e.g. assignment, simple computations,
  etc) and control flow edges to other basic blocks (e.g., calls, dropping
  values). MIR is used for borrow checking and other
  important dataflow-based checks, such as checking for uninitialized values.
  It is also used for a series of optimizations and for constant evaluation (via
  MIRI). Because MIR is still generic, we can do a lot of analyses here more
  efficiently than after monomorphization.
- LLVM IR: This is the standard form of all input to the LLVM compiler. LLVM IR
  is a sort of typed assembly language with lots of annotations. It's
  a standard format that is used by all compilers that use LLVM (e.g. the clang
  C compiler also outputs LLVM IR). LLVM IR is designed to be easy for other
  compilers to emit and also rich enough for LLVM to run a bunch of
  optimizations on it.

### Queries

The first big implementation choice is the _query_ system. The rust compiler
uses a query system which is unlike most textbook compilers, which are
organized as a series of passes over the code that execute sequentially. The
compiler does this to make incremental compilation possible -- that is, if the
user makes a change to their program and recompiles, we want to do as little
redundant work as possible to produce the new binary.

In `rustc`, all the major steps above are organized as a bunch of queries that
call each other. For example, there is a query to ask for the type of something
and another to ask for the optimized MIR of a function. These
queries can call each other and are all tracked through the query system.
The results of the queries are cached on disk so that we can tell which
queries' results changed from the last compilation and only redo those. This is
how incremental compilation works.

In principle, for the query-fied steps, we do each of the above for each item
individually. For example, we will take the HIR for a function and use queries
to ask for the LLVM IR for that HIR. This drives the generation of optimized
MIR, which drives the borrow checker, which drives the generation of MIR, and
so on.

... except that this is very over-simplified. In fact, some queries are not
cached on disk, and some parts of the compiler have to run for all code anyway
for correctness even if the code is dead code (e.g. the borrow checker). For
example, [currently the `mir_borrowck` query is first executed on all functions
of a crate.][passes] Then the codegen backend invokes the
`collect_and_partition_mono_items` query, which first recursively requests the
`optimized_mir` for all reachable functions, which in turn runs `mir_borrowck`
for that function and then creates codegen units. This kind of split will need
to remain to ensure that unreachable functions still have their errors emitted.

[passes]: https://github.com/rust-lang/rust/blob/45ebd5808afd3df7ba842797c0fcd4447ddf30fb/src/librustc_interface/passes.rs#L824

Moreover, the compiler wasn't originally built to use a query system; the query
system has been retrofitted into the compiler, so parts of it are not
query-fied yet. Also, LLVM isn't our code, so that isn't querified
either. The plan is to eventually query-fy all of the steps listed in the
previous section, but as of this writing, only the steps between HIR and
LLVM-IR are query-fied. That is, lexing and parsing are done all at once for
the whole program.

One other thing to mention here is the all-important "typing context",
[`TyCtxt`], which is a giant struct that is at the center of all things.
(Note that the name is mostly historic. This is _not_ a "typing context" in the
sense of `Γ` or `Δ` from type theory. The name is retained because that's what
the name of the struct is in the source code.) All
queries are defined as methods on the [`TyCtxt`] type, and the in-memory query
cache is stored there too. In the code, there is usually a variable called
`tcx` which is a handle on the typing context. You will also see lifetimes with
the name `'tcx`, which means that something is tied to the lifetime of the
`TyCtxt` (usually it is stored or _interned_ there).

[`TyCtxt`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/ty/struct.TyCtxt.html

### `ty::Ty`

Types are really important in Rust, and they form the core of a lot of compiler
analyses. The main type (in the compiler) that represents types (in the user's
program) is [`rustc::ty::Ty`][ty]. This is so important that we have a whole chapter
on [`ty::Ty`][ty], but for now, we just want to mention that it exists and is the way
`rustc` represents types!

Oh, and also the `rustc::ty` module defines the `TyCtxt` struct we mentioned before.

[ty]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc/ty/type.Ty.html

### Parallelism

Compiler performance is a problem that we would like to improve on
(and are always working on). One aspect of that is parallelizing
`rustc` itself.

Currently, there is only one part of rustc that is already parallel: codegen.
During monomorphization, the compiler will split up all the code to be
generated into smaller chunks called _codegen units_. These are then generated
by independent instances of LLVM. Since they are independent, we can run them
in parallel. At the end, the linker is run to combine all the codegen units
together into one binary.

However, the rest of the compiler is still not yet parallel. There have been
lots of efforts spent on this, but it is generally a hard problem. The current
approach is (**TODO: verify**) to turn `RefCell`s into `Mutex`s -- that is, we
switch to thread-safe internal mutability. However, there are ongoing
challenges with lock contention, maintaining query-system invariants under
concurrency, and the complexity of the code base. One can try out the current
work by enabling parallel compilation in `config.toml`. It's still early days,
but there are already some promising performance improvements.

### Bootstrapping

`rustc` itself is written in Rust. So how do we compile the compiler? We use an
older compiler to compile the newer compiler. This is called _bootstrapping_.

Bootstrapping has a lot of interesting implications. For example, it means that one
of the major users of Rust is Rust, so we are constantly testing our own
software ("eating our own dogfood"). Also, it means building the compiler can
take a long time because one must first build the compiler and then use that to
build the new compiler (sometimes you can get away without the full 2-stage
build, but for release artifacts you need the 2-stage build).

Bootstrapping also has implications for when features are usable in the
compiler itself. The build system uses the current beta compiler to build the
stage-1 bootstrapping compiler. This means that the compiler source code can't
use some features until they reach beta (because otherwise the beta compiler
doesn't support them). On the other hand, for compiler intrinsics and internal
features, we may be able to use them immediately because the stage-1
bootstrapping compiler will support them.

# Unresolved Questions

**TODO: find answers to these**

- Does LLVM ever do optimizations in debug builds?
- How do I explore phases of the compile process in my own sources (lexer,
  parser, HIR, etc)? - e.g., `cargo rustc -- -Zunpretty=hir-tree` allows you to
  view HIR representation
- What is the main source entry point for `X`?
- Where do phases diverge for cross-compilation to machine  code across
  different platforms?

# References

- Command line parsing
  - Guide: [The Rustc Driver and Interface](https://rustc-dev-guide.rust-lang.org/rustc-driver.html)
  - Driver definition: [`rustc_driver`](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_driver/)
  - Main entry point: [`rustc_session::config::build_session_options`](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_session/config/fn.build_session_options.html)
- Lexical Analysis: Lex the user program to a stream of tokens
  - Guide: [Lexing and Parsing](https://rustc-dev-guide.rust-lang.org/the-parser.html)
  - Lexer definition: [`librustc_lexer`](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_lexer/index.html)
  - Main entry point: [`rustc_lexer::tokenize`](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_lexer/fn.tokenize.html)
- Parsing: Parse the stream of tokens to an Abstract Syntax Tree (AST)
  - Guide: [Lexing and Parsing](https://rustc-dev-guide.rust-lang.org/the-parser.html)
  - Parser definition: [`librustc_parse`](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_parse/index.html)
  - Main entry point: **TODO**
  - AST definition: [`librustc_ast`](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_ast/ast/index.html)
  - Expansion: **TODO**
  - Name Resolution: **TODO**
  - Feature gating: **TODO**
  - Early linting: **TODO**
- The High Level Intermediate Representation (HIR)
  - Guide: [The HIR](https://rustc-dev-guide.rust-lang.org/hir.html)
  - Guide: [Identifiers in the HIR](https://rustc-dev-guide.rust-lang.org/hir.html#identifiers-in-the-hir)
  - Guide: [The HIR Map](https://rustc-dev-guide.rust-lang.org/hir.html#the-hir-map)
  - Guide: [Lowering AST to HIR](https://rustc-dev-guide.rust-lang.org/lowering.html)
  - How to view HIR representation for your code `cargo rustc -- -Zunpretty=hir-tree`
  - Rustc HIR definition: [`rustc_hir`](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_hir/index.html)
  - Main entry point: **TODO**
  - Late linting: **TODO**
- Type Inference
  - Guide: [Type Inference](https://rustc-dev-guide.rust-lang.org/type-inference.html)
  - Guide: [The ty Module: Representing Types](https://rustc-dev-guide.rust-lang.org/ty.html) (semantics)
  - Main entry point: **TODO**
- The Mid Level Intermediate Representation (MIR)
  - Guide: [The MIR (Mid level IR)](https://rustc-dev-guide.rust-lang.org/mir/index.html)
  - Definition: [`librustc/mir`](https://github.com/rust-lang/rust/tree/master/src/librustc/mir)
  - Definition of source that manipulates the MIR: [`librustc_mir`](https://github.com/rust-lang/rust/tree/master/src/librustc_mir)
  - Main entry point: **TODO**
- The Borrow Checker
  - Guide: [MIR Borrow Check](https://rustc-dev-guide.rust-lang.org/borrow_check.html)
  - Definition: [`rustc_mir/borrow_check`](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/borrow_check/index.html)
  - Main entry point: [`mir_borrowck` query](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/borrow_check/fn.mir_borrowck.html)
- MIR Optimizations
  - Guide: [MIR Optimizations](https://rustc-dev-guide.rust-lang.org/mir/optimizations.html)
  - Definition: [`rustc_mir/transform`](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/transform/index.html)
  - Main entry point: [`optimized_mir` query](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/transform/fn.optimized_mir.html)
- Code Generation
  - Guide: [Code Generation](https://rustc-dev-guide.rust-lang.org/codegen.html)
  - Guide: [Generating LLVM IR](https://rustc-dev-guide.rust-lang.org/codegen.html#generating-llvm-ir) - **TODO: this is not available yet**
  - Generating Machine Code from LLVM IR with LLVM - **TODO: reference?**
  - Main entry point MIR -> LLVM IR: [`MonoItem::define`](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_codegen_llvm/mono_item/enum.MonoItem.html#method.define)
  - Main entry point LLVM IR -> Machine Code: [`rustc_codegen_ssa::base::codegen_crate`](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_codegen_ssa/base/fn.codegen_crate.html)
