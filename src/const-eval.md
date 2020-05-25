# Constant Evaluation

Constant evaluation is the process of computing values at compile time. For a
specific item (constant/static/array length) this happens after the MIR for the
item is borrow-checked and optimized. In many cases trying to const evaluate an
item will trigger the computation of its MIR for the first time.

Prominent examples are:

* The initializer of a `static`
* Array length
    * needs to be known to reserve stack or heap space
* Enum variant discriminants
    * needs to be known to prevent two variants from having the same
      discriminant
* Patterns
    * need to be known to check for overlapping patterns

Additionally constant evaluation can be used to reduce the workload or binary
size at runtime by precomputing complex operations at compiletime and only
storing the result.

Constant evaluation can be done by calling the `const_eval_*` functions of `TyCtxt`.
They're the wrappers of the `const_eval` query.

The `const_eval_*` functions use a [`ParamEnv`](./param_env.html) of environment
in which the constant is evaluated (e.g. the function within which the constant is used)
and a [`GlobalId`]. The `GlobalId` is made up of an `Instance` referring to a constant
or static or of an `Instance` of a function and an index into the function's `Promoted` table.

Constant evaluation returns a [`ConstEvalResult`] with either the error, or the simplest
representation of the constant. "simplest" meaning if the `const` item is representable as an
integer or fat pointer, it will directly yield the value (via [`ConstValue::Scalar`]
or [`ConstValue::Slice`]), instead of referring to the [`miri`](./miri.html) virtual
memory allocation (via [`ConstValue::ByRef`]). This means that the `const_eval_*`
functions cannot be used to create miri-pointers to the evaluated constant or
static. If you need the value of a constant inside Miri, you need to directly work with
[`eval_const_to_op`].

[`GlobalId`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/mir/interpret/struct.GlobalId.html
[`ConstValue::Scalar`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/mir/interpret/value/enum.ConstValue.html#variant.Scalar
[`ConstValue::Slice`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/mir/interpret/value/enum.ConstValue.html#variant.Slice
[`ConstValue::ByRef`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/mir/interpret/value/enum.ConstValue.html#variant.ByRef
[`ConstEvalResult`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/mir/interpret/error/type.ConstEvalResult.html
[`eval_const_to_op`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/interpret/struct.InterpCx.html#method.eval_const_to_op
