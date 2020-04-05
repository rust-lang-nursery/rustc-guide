# About this guide

This guide is meant to help document how rustc – the Rust compiler –
works, as well as to help new contributors get involved in rustc
development.

There are six parts to this guide:

1. Contains information that should be useful no matter how you are
   contributing, such as procedures for contribution, building the compiler,
   etc.
2. Discusses the high-level architecture of the compiler, especially the query
   system.
3. Discusses the compiler frontend and internal representations.
4. Discusses the type system.
5. Discusses the compiler backend, code generation, linking, and debug info.
6. Appendices at the end with useful reference information.

The guide itself is of course open-source as well, and the sources can
be found at the [GitHub repository]. If you find any mistakes in the
guide, please file an issue about it, or even better, open a PR
with a correction!

## Other places to find information

You might also find the following sites useful:

- [Rustc API docs] -- rustdoc documentation for the compiler
- [Forge] -- contains documentation about rust infrastructure, team procedures, and more
- [compiler-team] -- the home-base for the rust compiler team, with description
  of the team procedures, active working groups, and the team calendar.

[GitHub repository]: https://github.com/rust-lang/rustc-dev-guide/
[Rustc API docs]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/
[Forge]: https://forge.rust-lang.org/
[compiler-team]: https://github.com/rust-lang/compiler-team/
