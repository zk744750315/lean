Design notes for Lean4
----------------------

# Goals

- Move more code from C++ to Lean.
- New compiler and C++ code generator.
- New runtime (support for unboxed values and FFI).
- New parser and macro expander (in Lean).
- New monad for accessing primitives that are only available in C++ (e.g., `type_context`).
- Fix critical issues (e.g., issue #1601).
- Fix language design issues.
- Reduce clutter in the core lib and code base.

# Plan

- Create Lean4 branch
- Disable most tests (they will be incrementally added back as we make progress on Lean4).
- Dramatically reduce the size of core lib. We should only keep the basics that
are needed to execute Lean programs. Remove most theorems and lemmas,
algebraic hierarchy, all non basic tactics, etc. Motivations: reduce clutter,
and increase agility. We will copy `library` into `old_library` and incrementally
rebuild core lib.
- Remove dead C++ code, and disable all but the most basic tactics.
- Add abstraction layers to isolate modules. Example: module manager should not depend on the parser;
  equation compiler should not depend on the elaborator.
- Make Lean object thread safe (see Memory management section for performance issues),
and remove related clutter. Example: we will not need ts_vm_obj anymore.
Remove unnecessary closure objects that were added for the previous C++ code generator
that was discontinued.
- Split tactic state into backtrackable and non-backtrackable parts using the
new monad transformers. The plan is to have the following monads for meta programming:
  a) `elab`: it is morally state_t and except_t where the state is `elab_state`.
     `elab_state` is defined in C++, and contains: `environment`, cache data structures,
     `name_generator`, `options` and `metavar_context`. Only `metavar_context` is backtrackable, and
     we define `<|>` as
     ```
     meta def elab.orelse {α : Type} (e₁ : elab α) (e₂ : elab α) : elab α :=
     ⟨λ s, let mctx := s.get_mvar_ctx in
        match e₁.run s with
        | elab_result.exception α ex s' := e₂.run (s'.set_mvar_ctx mctx)
        | r                             := r
        end⟩
     ```
     Note that, `elab_state` is used linearly in the definition above, and we just save
     `metavar_context`.

  b) `tactic`: it built on top of `elab` using `state_t`. In the new state we store
      the list of goals (i.e., metavariables) and the main metavariable.

  We will implement (most of) the C++ primitive tactics in the `elab` monad.
  One of the motivations for the approach above is that C++ and Lean code use `elab` in very similar ways.
  For example, the `type_context` object in C++ just gets a reference for the `elab_state` object.

  Remark: `elab` monad will be defined in Lean, but we will mark it private. All tactics that
  have access to the private definition will use `elab_state` linearly. The idea is to make sure
  we can use destructive updates when implementing C++ primitive tactics. This is a big
  advantage with respect to the approach used in Lean3 where we keep creating (and deleting) `tactic_state` objects
  all over the place.

  We have considered splitting `elab` into two monads. The first one `elab_core` would have a state
  without `metavar_context`. `elab_core` would morally be `except_t ex (state elab_core_state)`. And then,
  `elab` would be defined as `state_t metavar_context elab_core`. So, `metavar_context` would be backtrackable
  for free, and we would not need to define a custom `orelse`. This change is fine on the Lean side, but it would
  be messy when programming in C++ because we would have to continue to propagate the `metavar_context` manually as
  we do in Lean3. Recall that a few bugs in Lean3 are due to incorrect propagation of the `metavar_context`. These bugs
  will all disappear with the new design.
  We also have additional performance overhead with the `elab`+`elab_core` approach.
  For example, we want to support the following API:
  ```
  mk_type_context : elab type_context
  infer_type : type_context -> expr -> elab expr
  whnf : type_context -> expr -> elab expr
  ...
  ```
  If we use `elab`+`elab_core` approach, the `infer_type` primitive would have to be implemented as:
  ```
  vm_obj infer_type(vm_obj const & ctx, vm_obj const & e, vm_obj const & mctx, vm_obj const & s) {
  try {
    ctx.set_mctx(mctx); // copy current metavar_context to ctx
    auto r = ctx.infer(e);
    return mk_success_result(r, ctx.mctx(), s); // build result using updated metavar_context
  } catch (exception & ex) {
    ....
  } }
  ```
  In the `elab` only approach, we save one argument `mctx`, and we don't have to copy the `metavar_context`.

- Implement support objects in Lean: options, format, structure trace messages, syntax object, etc.
- Add parser infrastructure in Lean.
- Compiler and C++ code generator. The C++ code generator will avoid many bootstrapping
problems we have. The idea is to write several Lean modules in Lean, emit C++ code, save
the generated C++ code in our repo.
- New IR with support for non uniform memory layout for Lean objects (see details on the #backend
Slack channel).
- Develop a tool in Lean that given a Lean inductive datatype (or structure) generates C++
code for retrieving fields and creating Lean objects. The goal is to isolate primitives implemented
in C++ from the way we represent Lean objects in memory. For example, most Lean functions implemented
in C++ use the C++ function `cfield` which assumes objects have a uniform memory layout.
- Develop a tool for generating glue code for interfacing Lean and C++ code. Again, the goal
is to isolate the primitives from the way we handle boxed/unboxed values.
For example, suppose we have a builtin function `foo` that takes two Lean `bool` values.
Right now, this function takes boxed values `c_foo(vm_obj const & a, vm_obj const & b)`.
It feels weird to have to box a Lean value to be able to invoke the builtin implementation for `foo`.
After we have the tool, we would write `c_foo(bool a, bool b)` and describe its signature in a Lean file.
The tool then generates the wrappers for invoking `c_foo` from the interpreter and generated C++ code.

# Language and library issues

- `private` declarations are not reliable. Users can easily subvert them
using meta programming. This is problematic for several optimizations
we want to use. For example, suppose we define a state-like monad
where every primitive uses the state linearly. The code
generator cannot rely on that since users can currently access
the internal implementation, and use the state in a non linear way.
This is just an example. We have many more.

- `parameter`s are currently simulated in Lean. For example,
when we declare `foo` in a section with a parameter `A`,
`foo` is automatically abstracted and an alias `foo => @foo A` is created.
This creates many problems, most of them are documented in the issue tracker.
This approach has one advantage: users can use the abstracted (`_root_.foo`)
and non-abstracted (`foo`) version simultaneously. Another advantage is that
we don't have to type check `foo` more than once in the kernel.
That being said, the disadvantages far outweigh the advantages.
We plan to go back to the approach used in Lean1 and Coq.

- Coercion resolution (see issue #1402).

- Name resolution for `[ ... ] tactic blocks.
The `[ ... ] notation allows us to use interactive tactic notation
when writing reusable tactics. This is very convenient, but the current
implementation uses dynamic name resolution, and is a source of many
bugs.

- `if-then-else` using `bool` instead of `Prop`.
As soon as we started programming with Lean (version 3), it became clear
that `if-then-else` with `Prop` creates more problems that it solves.
The elaborator already has support for a coercion from `Prop` to `bool`
(for decidable propositions). The dependent `if H : p then t else e`
may look cute, but it is unnecessary now that we have `match`.

- `decidable` type class. A recurrent problem in Lean occurs when
users perform dependent elimination on `decidable` instances.
The problem occurs when we have `[h : decidable p]` in the context
and a goal `G[@f p h]`, that is, a goal containing the term `@f p h`.
Then, we perform `cases h`, and obtain `G[@f p (dedicable.is_true h')]` in
one branch and `G[@f p (decidable.is_false h)]` in another. Then, we apply a
lemma that gives us `G[@f p h]` where `h` is the synthesized instance,
but we cannot use it to close the goal because we get a type error.
We have discussed this problem with Tahina and he pointed out that
we should never perform dependent elimination on type class instances in proofs,
and that this is an anti-idiom. He told us that every type class is a
structure in Coq. That is, everything is wrapped in a structure.
In Coq, they would use a custom eliminator for performing case analysis
on decidable instances. They don't face this problem because the custom eliminator
is more convenient to use them manually destructing the wrapper structure,
and then the actual data. He also strongly suggested that we
should decouple the program that computes whether a proposition is true or false
from the proof the result is correct. The Lean type class combines both in one single definition.
He said this will be a problem in the future for users that want to compute in the kernel.
 The kernel computations will have to deal with these proof terms. In Coq, `decidable` is now defined as:
```
Class Decidable (P : Prop) := {
  Decidable_witness : bool;
  Decidable_spec : Decidable_witness = true <-> P
}.
```
He strongly recommended we define `decidable` using this approach.
He said this is not the original definition used in Coq. The first one was a structure wrapping a
sum type (which is closer to our definition), and the Coq developers had to change it
because of performance problems in proofs by reflection.

- Interactive tactics. In Lean4 we will have a much more extensible and flexible
parser, and it will be written in Lean. We will be able to write a custom parser for the tactic
interactive mode. So, the current argument-type driven parser we use will not
be needed anymore.

- Quotations. Do we need all of them? I find the one with three backticks very inconvenient to use.
Moreover, as soon as we implement the new parser, we will want quotations for building
the new syntactic object and Lean expressions.

# Compiler

- The new compiler will use a System-F like intermediate representation.
It will be similar to Haskell core language. Inductive datatypes will be represented
using a constant for each constructor and a `cases` eliminator. If `cases` is encoded
using a expr-macro, we can easily support `default/other` case.

- Code inlining will occur at the System-F level after we have applied
simplifications.  This is relevant for the performance issues we have
observed when a long chain of functions need to be unfolded (e.g., new
monad transformer library).

- The first compilation step applies compiler specific simplification rules provided by users.
For example, we will be able to mark `map g (map f l) = map (g o f) l` as an optimization
rule for the compiler.

Issue: many opportunities for applying simplification rules only appear after we have inlined
definitions. However, we want to inline after we have converted into System-F and have
erased computationally irrelevant code and applied basic simplifications (e.g., erase trivial structures).
This is a problem since user provided simplification rules are not applicable here since they have
been described at the Lean level.

- Basic types (scalars, bool, char, uint32, uint64, int64, int32, ...) and C++ types
can be stored in unboxed form. The unboxed version are prefixed with `#` as they do
in Haskell.

- When we convert a Lean function to System-F, we will generate two versions: boxed and unboxed.
The boxed version is needed when passing this function to polymorphic higher-order functions.
As in Haskell, polymorphic functions always take boxed values.
Both versions are stored in the Lean environment as `meta` functions.
Example: the function `def inc (a : int32) := a + 1` is converted into two versions
`meta def _SystemF.boxed.inc (a : int32) := a + 1`, and `meta def _SystemF.unboxed.inc (a : #int32) := a #+ #1`.
Now, suppose we want to compile `twice inc a` where `def twice {A : Type} (f : A -> A) (a : A) := f (f a)`.
Then, since `twice` is polymorphic, we need to pass `inc` boxed version, and we generate
`@_SystemF.boxed.twice int32 _SystemF.boxed.inc a`.

- We want to implement monomorphisation as an additional optimization step. The idea is specialize functions like `twice`.
In the previous example, monomorphisation would generate `@_SystemF.unboxed.twice_int32 (f : #int32 -> #int32) (a : #int32)`.
We are considering caching monomorphised functions into the .olean files. If we do this, we have to consider the situation
where more than one .olean contains the same monomorphised function. We see two options: we have a canonical way to generate
names for monomorphised functions; we generate unique names, and accept the fact the environment will contain duplicates.
It is just a space issue.
Remark: see comment `closure` bullet point below. We may not use unboxed values in closures.

- In Lean3, `name`, `level` and `expr` are all implemented in C++. To expose these objects in Lean, we have to wrap them
using a subclass of `vm_external`. This generates a significant performance overhead. For example, suppose we have a lean
function that traverses a big expression; for each visited expression `e`, we need to create a `vm_expr` object for wrapping `e`.
Moreover, we have two layers of reference counting: one at `expr` and another at `vm_obj`.
This design decision is fine in Lean3 because the most expensive tasks are implemented in C++, and Lean code is only
used to "glue" together existing procedures implemented in C++. This is not the case in Lean4.
So, in Lean4, all these objects will be implemented directly in Lean.
As described above, we will have a tool that will generate C++ functions for creating and accessing these objects.
We believe this will not affect much how we code in C++, and it will eliminate a lot of boilerplate code we currently use.

# Object memory layout

- Constructor objects: they will contain pointers and unboxed data. We will use all-pointers first approach, and the header will contain 8 bytes:
  a) reference counter: 4 bytes
  b) kind: 1 byte
  c) tag (aka constructor index): 1 byte
  d) size (aka number of pointer objects): 2 bytes
  After the header we have `size` pointers and then all unboxed data. In debug mode, we want to store the size of the space used for
  unboxed data, and use this information for implementing sanity checks.
  Remark: the header of all composite Lean objects start with the reference counter and `kind`.
  Note that this representation supports only inductive datatypes with at most 256 constructors and 2^16 pointer fields.
  This is sufficient for our needs.
  If one day we want to support inductive datatypes with more than 256 constructors and/or 2^16 pointer fields, we can add
  a new kind of constructor object, and add new opcodes for manipulating them. We say this new kind is a "fat" constructor object
  since its header is bigger.

- Array: we support arrays of pointers and arrays of unboxed data. We will have opcodes for reading and updating arrays.
The updates are destructive when the array reference counter is 1.

- Closures: we need two kinds of closures: one that stores the bytecode id (for interpreted code); and another that stores a function pointer (for compiled code).
We need to decide whether we will support closures that store unboxed data or not. The simple solution is to support only boxed data
in closures. This may not be a huge performance overhead since many higher order functions such as `map` and `fold` will
be specialized during the monomorphisaton step, and we will not even create the closures.
A possible compromise is to support unboxed data only for closures that store function pointers. The idea is the following, whenever we emit C++ code for a function,
we also generate a `run` function for it s.t. given the closure data, it invokes the function. Then, we store the pointer to the `run` function in the closure.
For example, suppose we have generated
```
vm_obj foo(bool b1, vm_obj o, bool b2) { ... }
```
Then, we would also generate
```
vm_obj run_foo(closure_data d) {
  return foo(get_closure_bool(8, d), get_closure_obj(0, d), get_closure_bool(9, d));
}
```
`get_closure_bool(offset, d)` retrieves the Boolean stored at the given offset in `d`.
Note that, as in constructor objects, unboxed data is stored after pointers at `closure_data`.
So, we use `get_closure_bool(8, d)` to retrieve `b` which is stored after `o` (the pointer to `o` consumes 8 bytes) in `d`.
It is not clear this approach is a good one since we would always need to create a `closure_data` object that contains all arguments
before executing a closure. It is not clear how to support the optimization we use in Lean3 that avoids the allocation of the last `closure_data`
in most cases (https://github.com/leanprover/lean/blob/master/src/library/vm/vm.cpp#L1698).

- MPZ (multiprecision integers)

- String. Remark: internally it is not an array of char, but an array of bytes encoded in UTF8. The key difference with respect to Lean3 is that it will
not use the `vm_external` wrapper approach, but have an object kind for strings.

# IR

- Register based.
- Explicit reference counting instructions. We use reference counting for: composite objects, closures and
  C++ unboxed data (e.g., `expr`). Remark: for each C++ primitive the VM needs to know how to increase/decrease the reference counter.
- Support for unboxed values.
- Instructions for accessing unboxed values in non-uniform structures.
  We will have instructions for operations such as `get_scalar_<sz>(obj, offset)`,
  where `obj` is a (potentially non-uniform) Lean object, `offset` is the offset
  inside of this object, and `sz` is the number of bytes needed to store the object.
  In practice, we would have `get_scalar_1`, `get_scalar_2`, `get_scalar_4` and `get_scalar_8`.
  These scalars are unboxed. We would have registers for storing the different kinds of scalars,
  and basic operations on them (e.g., comparison, arithmetic, etc). So, we will
  have instructions such as `GETS_<sz> r_o r_i offset` where `r_o` and `r_i` are registers, and
  it corresponds to `r_o := get_scalar_<sz>(r_i, offset)`. Moreover, `r_o` must be a scalar
  register of size `sz` and `r_i` is a register for storing Lean objects.

Remark: for each constructor datatype, we will have a table that maps fields to the
operation needed to retrieve them. We will use this table when converting the SystemF
representation into the IR.

Open issue: should we use SSA or SIL?
BTW, most of the benefits of SSA/SIL seem to be irrelevant for Lean.
The paper https://arxiv.org/pdf/1507.05762.pdf send by Nuno describes the pros/cons for SSA.
Most of them seem to be related to static analysis procedures. I think we will need very few
static analysis steps, most optimizations will be implemented at System-F before we convert the code into IR.

# VM

- We need a new VM for the new IR.

- The VM should be able to invoke primitives hand written in C++ and
C++ code emitted by the Lean compiler.

- A few hand written C++ primitives and C++ code emitted by the Lean compiler
need to invoke Lean functions. We should be careful to avoid a mismatch here where
a C++ function F for Lean version `X` is trying to invoke bytecode for
a Lean function G for version `X+1`. If we allow this to happen the system
may crash because the data representation for version `X+1` may be
different from version `X`.

We can try to address this issue by breaking core lib into two parts.
The first part (bootstrapping) contains all the infrastructure needed by the parser, compiler,
tactic framework, and Lean runtime. If we make a change here, we should
compile it again using the previously emitted C++ code, and then generate
a new version of the C++ code, compile it, and check whether it works or not.
In principle, it is not safe to invoke bytecode generated during the current compilation
from previously emitted C++ code since they may be using different representations.
Of course, the changes may be harmless, but to avoid problems we should minimize
the number of tactics used in this part of the core lib. Ideally, tactics should
not be used in the bootstrapping part.

We may also emit C++ code for non essential functionality that is implemented in Lean,
and then link it with the Lean executable. Example: a decision procedure, a parser extension.
The idea is to provide a more efficient version to users. Here we can use a more
relaxed approach since this functionality is not part of the compiler. We can store a hash code
for each of these functions. When we import the .olean file that generated the function
we compare whether the hash code there matches the one in the emitted C++ code.
If it does, we use the C++ version, otherwise we use the bytecode.

# Explicit reference counting

As described above, we will have opcodes for increasing/decreasing `vm_obj`s reference counters.
In Lean3, we perform too many unnecessary increments/decrements.
A function is responsible for decreasing the reference counter of each argument (i.e., it consumes the argument), and
for increasing the reference counter of the result. Each argument should be viewed as a resource that is consumed by the function.
Many optimizations are possible. For example, consider the function
```
def proj2 (a : A) (b : B) := b
```
A naive compilation into bytecode would produce:
```
def proj2 a b :=
  r := b;
  inc r;
  dec a;
  dec b;
  return r
```
This can be optimized as
```
def proj2 a b :=
  dec a;
  return b
```
We need to explicitly decrement `a` because it was not use by `proj2`.
The function
```
def ex1 (x : X) (y : Y) :=
  let v1 := f x x,
      v2 := f x y
  in h v1 v2
```
is compiled as
```
def ex1 x y :=
  inc x;
  inc x;
  v1 := f x x;
  v2 := f x y;
  r  := h v1 v2;
  return r
```
The reference counter for `x` is incremented twice because it is used 3 times in this function.
Note that, we don't have to decrement `v1` nor `v2` since they are consumed by `h`.
We can an instruction `inc2 x` for executing `inc x; inc x` in a single step.
Now consider a map-like function
```
def my_map : list A -> list A
| []     := []
| (h::t) := g h :: my_map t
```
This function will be compiled as
```
def my_map t :=
switch (cidx t) {
case 0:
  return t;
case 1:
  h1 := get#0 t;
  t1 := get#1 t;
  dec t;
  h2 := g h1;
  t2 := my_map t2;
  r  := mk_list_cons h2 t2;
  return r
}
```
The instruction `get#<idx> t` returns the field `idx` of the object `t`. It bumps the reference counter of the resulting object, but does not update the reference counter of `t`.
`mk_list_cons h2 t2` is a function that creates a constructor object with tag `#1` and two pointers `h2` and `t2`. We don't need to decrement
the reference counters of `h2` and `t2` since they were "consumed" by `mk_list_cons`. We can inline `mk_list_cons` too.
We can also add a special `dec` instruction for reusing memory cells, and write `my_map` as:
```
def my_map t :=
switch (cidx t) {
case 0:
  return t;
case 1:
  h1 := get#0 t;
  t1 := get#1 t;
  t_cell := dec_core t;
  h2 := g h1;
  t2 := my_map t2;
  r  := mk_list_cons_reusing h2 t2 t_cell;
  return r
}
```
The `dec_core t` instruction decrements the reference counter of `t`, but does not delete the memory cell if it is zero.
Then, `mk_list_cons_reusing` will reuse `t`'s memory cell if the counter is 0, and will create a new cell otherwise.
With this trick, `my_map` will not allocate any constructor object if the input list is not shared.
To avoid memory leaks, we have to make sure that in each path after `dec_core t`, `t_cell` is used in a `mk_*_reusing` function
and/or we explicitly delete it using `del t_cell`.
When the reference counter of `t` is one in `case 1`, the `get#0 t` and `get#1 t` unnecessarily increase the reference counter of the result value just to decrease it again at `dec_core t`. We can avoid this overhead by using the following alternative formulation
```
def my_map t :=
switch (cidx t) {
case 0:
  return t;
case 1:
  if (ref_count t == 1) {
    h1     := steal#0 t;
    t1     := steal#1 t;
    t_cell := t;
  } else {
    h1     := get#0 t;
    t1     := get#1 t;
    dec t;
    t_cell := 0;
  }
  h2 := g h1;
  t2 := my_map t2;
  r  := mk_constructor_reusing #1 2 t_cell;
  set#0 r h2;
  set#1 r t2;
  return r
}
```
The instruction `steal#<idx> t` is similar to `get#<idx> t`, but it does not increase the reference counter of the resulting object.
`mk_constructor_reusing #1 2 t_cell` creates a constructor object with tag `#1` and size 2 and reusing `t_cell` if different from 0. The instructions `set#<idx>` are used to initialize the resulting fields.
The optimization above can be used whenever `t` is dead after the `case`, and an object of same size is created.
Note that if `t` is a list of arrays and it is not shared, then `g h1` will also be able to perform destructive updates.
Remark: suppose the `i`-th field in `case` branch is not used, then instead of using `steal#i t`, we use `dec#i t` to decrement the reference counter of the `i`-th field.

TODO: create experiments to check whether the optimization above is relevant or not.

# Tail recursion

TODO

# Unboxed products and sums

TODO

# Memory management

Lean3 VM objects are not thread safe: they do not use atomic
operations for updating the reference counter, and we use a small
object memory allocator. The main motivation was performance.
We evaluated these design decisions again, and did not observe any
performance impact when we used atomic operations for updating
the reference counter and removed the small object allocator.
The experiments were conducted using OSX and Linux.
We considered two benchmarks: core lib compilation, and a small Lean
program attached in the end of this section.
In both platforms and benchmarks no significant difference was observed.
Then, we disable the `memory_pool` object, and again no difference
in performance was observed.
We believe the memory allocators in the C++ runtime have been improved.
This is consistent with our observation that building Lean with `tcmalloc`
does not improve the performance significantly anymore.
However, it is not clear why using std::atomic does not impact performance
anymore.

```
def foo (n : nat) : nat :=
(((list.iota n).map (+10)).map (+30)).length

#eval nat.repeat (λ i _, foo i) 4000 0
```
