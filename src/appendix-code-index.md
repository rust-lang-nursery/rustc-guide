# Appendix D: Code Index

rustc has a lot of important data structures. This is an attempt to give some
guidance on where to learn more about some of the key data structures of the
compiler.

Item            |  Kind    | Short description           | Chapter            | Declaration
----------------|----------|-----------------------------|--------------------|-------------------
`CodeMap` | struct | The CodeMap maps the AST nodes to their source code | [The parser] | [src/libsyntax/codemap.rs](https://doc.rust-lang.org/nightly/nightly-rustc/syntax/codemap/struct.CodeMap.html)
`CompileState` | struct | State that is passed to a callback at each compiler pass | [The Rustc Driver] | [src/librustc_driver/driver.rs](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_driver/driver/struct.CompileState.html)
`DocContext` | struct | A state container used by rustdoc when crawling through a crate to gather its documentation | [Rustdoc] | [src/librustdoc/core.rs](https://github.com/rust-lang/rust/blob/master/src/librustdoc/core.rs)
`ast::Crate` | struct | Syntax-level representation of a parsed crate | [The parser] | [src/librustc/hir/mod.rs](https://doc.rust-lang.org/nightly/nightly-rustc/syntax/ast/struct.Crate.html)
`hir::Crate` | struct | More abstract, compiler-friendly form of a crate's AST | [The Hir] | [src/librustc/hir/mod.rs](https://doc.rust-lang.org/nightly/nightly-rustc/rustc/hir/struct.Crate.html)
`ParseSess` | struct | This struct contains information about a parsing session | [the Parser] | [src/libsyntax/parse/mod.rs](https://doc.rust-lang.org/nightly/nightly-rustc/syntax/parse/struct.ParseSess.html)
`Session` | struct | The data associated with a compilation session | [the Parser], [The Rustc Driver] | [src/librustc/session/mod.html](https://doc.rust-lang.org/nightly/nightly-rustc/rustc/session/struct.Session.html)
`StringReader` | struct | This is the lexer used during parsing. It consumes characters from the raw source code being compiled and produces a series of tokens for use by the rest of the parser | [The parser] |  [src/libsyntax/parse/lexer/mod.rs](https://doc.rust-lang.org/nightly/nightly-rustc/syntax/parse/lexer/struct.StringReader.html)
`TraitDef` | struct | This struct contains a trait's definition with type information | [The `ty` modules] |  [src/librustc/ty/trait_def.rs](https://doc.rust-lang.org/nightly/nightly-rustc/rustc/ty/trait_def/struct.TraitDef.html)
`Ty<'tcx>` | struct | This is the internal representation of a type used for type checking | [Type checking] | [src/librustc/ty/mod.rs](https://doc.rust-lang.org/nightly/nightly-rustc/rustc/ty/type.Ty.html)
`TyCtxt<'cx, 'tcx, 'tcx>` | type | The "typing context". This is the central data structure in the compiler. It is the context that you use to perform all manner of queries. | [The `ty` modules] | [src/librustc/ty/context.rs](https://doc.rust-lang.org/nightly/nightly-rustc/rustc/ty/struct.TyCtxt.html)

[The HIR]: hir.html
[The parser]: the-parser.html
[The Rustc Driver]: rustc-driver.html
[Type checking]: type-checking.html
[The `ty` modules]: ty.html
[Rustdoc]: rustdoc.html
