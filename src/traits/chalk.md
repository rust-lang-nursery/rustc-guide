# Chalk-based trait solving (new-style)

> 🚧 This chapter describes "new-style" trait solving. This is still in the
> [process of being implemented][wg]; this chapter serves as a kind of
> in-progress design document. If you would prefer to read about how the
> current trait solver works, check out
> [this other subchapter](./resolution.html). 🚧
>
> By the way, if you would like to help in hacking on the new solver, you will
> find instructions for getting involved in the
> [Traits Working Group tracking issue][wg]!

[wg]: https://github.com/rust-lang/rust/issues/48416

The new-style trait solver is based on the work done in [chalk][chalk]. Chalk
recasts Rust's trait system explicitly in terms of logic programming. It does
this by "lowering" Rust code into a kind of logic program we can then execute
queries against.

The key observation here is that the Rust trait system is basically a
kind of logic, and it can be mapped onto standard logical inference
rules. We can then look for solutions to those inference rules in a
very similar fashion to how e.g. a [Prolog] solver works. It turns out
that we can't *quite* use Prolog rules (also called Horn clauses) but
rather need a somewhat more expressive variant.

[Prolog]: https://en.wikipedia.org/wiki/Prolog

You can read more about chalk itself in the
[Chalk book](https://rust-lang.github.io/chalk/book/) section.

## Ongoing work
The design of the new-style trait solving currently happens in two places:

**chalk**. The [chalk][chalk] repository is where we experiment with new ideas
and designs for the trait system.

**rustc**. Once we are happy with the logical rules, we proceed to
implementing them in rustc. This mainly happens in
[`librustc_traits`][librustc_traits]. We map our struct, trait, and impl declarations
into logical inference rules in the [lowering module in rustc](./lowering-module.md).

[chalk]: https://github.com/rust-lang/chalk
[librustc_traits]: https://github.com/rust-lang/rust/tree/master/src/librustc_traits
