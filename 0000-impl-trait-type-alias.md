- Feature Name: liberal_impl_trait
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Add the ability to create type aliases for `impl Trait` types,
support `impl Trait` in `let`, `const`, and `static` declarations,
and allow module-local ascription of types to `impl Trait` expressions:

```rust
// `impl Trait` type alias:
type Adder = impl Fn(usize) -> usize;
fn adder(a: usize) -> Adder {
    |b| a + b
}

// `impl Trait` type aliases in associated type position:
struct MyType;
impl Iterator for MyType {
    type Item = impl Debug;
    fn next(&mut self) -> Option<Self::Item> {
        Some("Another item!")
    }
}

// `impl Trait` in `let`, `const`, and `static`:

const ADD_ONE: impl Fn(usize) -> usize = |x| x + 1;

static MAYBE_PRINT: Option<impl Fn(usize)> = Some(|x| println!("{}", x));

fn my_func() {
    let iter: impl Iterator<Item = i32> = (0..5).map(|x| x * 5);
    ...
}


// Module-local ascription of types to `impl Trait` expressions:
mod foo {
    pub const X: impl Debug = 5i32;

    pub fn fn_in_foo() {
        println!("{:?}", X); // `foo::X` is of type `impl Debug`

        // Because `X` is declared  as `impl Debug` within the same module,
        // we can assert that `X` is actually of type `i32`:
        let x: i32 = X;
        println!("{:?}", x + 5);
    }
}

fn fn_outside_foo() {
    // This works because `foo::X` is of type `impl Debug`:
    println!("{:?}", foo::X);

    // This fails because `X` is declared as `impl Debug` inside a different
    // module.
    let x: i32 = X; // ERROR: expected type `i32`, found type `impl Debug`
    println!("{:?}", x + 5);
}
```

# Motivation
[motivation]: #motivation

This RFC proposes three expansions to Rust's `impl Trait` feature.
`impl Trait`, first introduced in [RFC 1522][1522], allows functions to return
types which implement a given trait, but whose concrete type remains anonymous.
`impl Trait` was expanded upon in [RFC 1951][1951], which added `impl Trait` to
argument position and resolved questions around syntax and parameter scoping.
In its current form, the feature makes it possible for functions to return
unnameable or complex types such as closures and iterator combinators.
`impl Trait` also allows library authors to hide the concrete type returned by
a function, making it possible to change the return type later on.

However, the current feature has some severe limitations.
Right now, it isn't possible to return an `impl Trait` type from a trait
implementation. This is a huge restriction which this RFC fixes by making
it possible to create a type alias for an `impl Trait` type using the syntax
`type Foo = impl Trait;`. The type alias syntax also makes it possible to
guarantee that two `impl Trait` types are the same:

```rust
// `impl Trait` in traits:
struct MyStruct;
impl Iterator for MyStruct {
    // Here we can declare an associated type whose concrete type is hidden.
    // External users only know that `Item` implements the `Debug` trait.
    type Item = impl Debug;

    fn next(&mut self) -> Option<Self::Item> {
        Some("hello")
    }
}

// `impl Trait` type aliases allow us to declare multiple items which refer to
// the same `impl Trait` type:

// Type `Foo` refers to a type which is hidden, and which is only known to to
// implement the `Debug` trait.
type Foo: impl Debug;

const FOO: Foo = 5;

fn foo1() -> Foo {
    5
}

fn foo2() -> Foo {
    10
}

fn opt_foo() -> Option<Foo> {
    Some(foo1())
}

// Since we know that all `Foo`s have the same (hidden) concrete type, we can
// write a function which returns `Foo`s acquired from different places.
fn one_of_the_foos(which: usize) -> Foo {
    match which {
        0 => FOO,
        1 => foo1(),
        2 => foo2(),
        3 => opt_foo().unwrap(),

        // It also allows us to make recursive calls to functions with an
        // `impl Trait` return type:
        x => one_of_the_foos(x - 4),
    }
}
```

Separately, this RFC makes it possible to store an `impl Trait` type in a
`let`, `const` or `static`.
This makes it possible to store types such as closures
or iterator combinators in `const`s and `static`s, and also makes it possible
to write more concise type declarations for `const`s and `static`s.
Writing out the full type of `const`s and `static`s has already been mentioned
as a pain point in the [`const`/`static` type annotation elison RFC][2010].
However, omitting the types of `const` and `static` items could make it harder
to tell what sort of value is being stored in a `const` or `static`.
Additionally, inferring the type of `const`s and `static`s is in conflict
with Rust's current convention of requiring explicity-written types for
module-level items.
`impl Trait` `const`s and `static`s would help to alleviate frustration and
enable use of unnameable types without requiring full omission of types:

Finally, this RFC adds the ability to ascribe concrete types to module-local
`impl Trait` values. This allows modules to hide their concrete types from the
outer world with `impl Trait` without sacrificing expressiveness:

```rust
mod foo {
    // The author of module `foo` wants to hide the concrete type of
    // `SomeType` (`i32`):
    type SomeType = impl Debug;
    
    pub fn foo() -> SomeType {
        0i32
    }

    // Without this feature, `enlarge_foo` is impossible to write-- it must take
    // `SomeType` as its argument to maintain the API boundary, but it can't
    // do anything that requires knowing the concrete type of `SomeType`.
    pub fn enlarge_foo(x: &mut SomeType) -> SomeType {
        // In order to add to `x`, we have to assert that it is `&mut i32`.
        // This requires that `SomeType` is `i32`.
        let x: &mut i32 = x;
        x + 5
    }
}

fn outside() {
    // Outside the module, we can call `foo` and `enlarge_foo`, but the only
    // thing we know about `SomeType` is that it implements the `Debug` trait.
    let mut x = foo();
    enlarge_foo(&mut x);
    println!("{:?}", x);
}

```

[1522]: https://github.com/rust-lang/rfcs/blob/master/text/1522-conservative-impl-trait.md
[1951]: https://github.com/rust-lang/rfcs/blob/master/text/1951-expand-impl-trait.md
[2010]: https://github.com/rust-lang/rfcs/pull/2010

# Detailed design
[design]: #detailed-design

## `impl Trait` in `let`, `const`, and `static` Declarations
[design-let-const-static]: #design-let-const-static

The idea of this feature is to simply allow specifying `impl Trait` in
type declarations:

```rust
const ADD_ONE: impl Fn(usize) -> usize = |x| x + 1;

static MAYBE_PRINT: Option<impl Fn(usize)> = Some(|x| println!("{}", x));

fn my_func() {
    let iter: impl Iterator<Item = i32> = (0..5).map(|x| x * 5);
    ...
}
```

When an `impl Trait` type is declared in a `const`, or `static` binding,
the type can only be used as something that implements `Trait`-- its concrete
type is no longer known. This allows modules to declare `const`s and `static`s
whose inner type is private. As we'll explain later, `let` bindings are handled
differently, and their concrete type remains public within the item (function)
in which they're written.

Since the type of `let` bindings can already be fully inferred, `impl Trait`
doesn't add any additional expressiveness to `let` bindings.
However, `const`s and `static`s current must have their full types manually
specified.
`const` and `static`s don't currently have any type inference, so we must add
support for inferring the types of `impl Trait` consts/statics.
In order to prevent the types of variables changing as a result of moving
from a concrete bound to an `impl Trait` bound,
no integer or float fallback will be applied when
inferring the types of `const`s or `static`s.

## `impl Trait` Type Aliases and Obligations
[design-type-aliases]: #design-type-aliases

For the most part, an `impl Trait` type alias can be thought of as the same as
any other type alias-- it introduces a new type name which is equivalent to
some other type. The major differences are are that `impl Trait` type aliases
hide the concrete type to which they refer, and their concrete type isn't
specified at the declaration site (i.e. `type Foo = impl Debug;` alone gives no
information about the concrete type of `Foo` other than that it implements the
`Debug` trait). This second point introduces an issue: how to resolve the
concrete type of `Foo`.

Rust doesn't currently support module-level inference, and adding it would pose
a significant implementation and compile-time burden. So `Foo`, which is a
module-level item, can't be "inferred" in the traditional sense. Instead,
the concrete type of `Foo` is determined from how `Foo` is used in the module
in which it is declared. When a concrete type is used as `Foo`, or when an
expression with type `Foo` is ascribed a type, an "obligation" is created.

```rust
type Foo = impl Debug;

fn foo() -> Foo {
    5i32 // Creates an obligation that `Foo` == `i32`
}

fn bar() {
    // Ascription of `Foo` to `i32` creates an obligation:
    let _: i32 = foo();
}
```

Obligations are visible to the item in which the obligation occurs. This
means that, for example, a function which returns `i32` as a `Foo` can
rely on other values of type `Foo` being of type `i32`:

```rust
type Foo = impl Debug;

fn foo() -> Foo {
    0i32
}

// Since `bar` returns `i32` as `Foo`, it knows that `Foo` must be `i32`,
// so it can use all expressions of type `Foo` as though they were of type
// `i32`.
fn bar() -> Foo {
    println!("{:?}", foo() + 5);
    5i32
}

fn baz() {
    // Creates an obligation that `Foo` == `i32`:
    let x: i32 = foo();
    println!("{:?}", x + 5);
}

// This function, however, creates no obligations on the type of `Foo`,
// and so cannot see into the concrete type of `Foo`.
fn bad1() {
    println!("{:?}", foo() + 5); // ERROR
}

// This function passes function-local typechecking, but creates an
// obligation `Foo` == `i64` which conflicts with the other obligations
// in the module, resulting in an error.
fn bad2() {
    let _: i64 = foo();
}

// `const`s and `static`s can also create obligations:

// Creates an obligation `Foo` == `i32`:
const MY_FOO: Option<Foo> = Some(5i32);

// Creates an obligation `Foo` == `i32`:
static MY_FOO_2: Foo = 5i32;
```

Each function is fully typechecked using only the knowledge of its own
local obligations. Once all of the obligations have been created, they
are checked against each other for conflicts.

## More on Obligations
[design-more-obligations]: #design-more-obligations

It's worth noting that obligations apply to more than just type-aliased
`impl Trait` types. They apply to _all_ occurrences of `impl Trait`.
When a non-aliased `impl Trait` is used, it is resolved just the same
way. This allows for recursive `impl Trait` functions and other things
not possible under the current `impl Trait` rules:

```rust
fn foo(x: usize) -> impl Debug {
    match x {

        // Returning a value of type `i32` creates an obligation
        // `typeof(foo)::Return` == `i32`.
        0 => 5i32,

        // We can use that information in the rest of the function body:
        _ => foo(x - 1) + 5,
    }
}

fn bar(x: bool) -> impl Debug {
    if x {
        // Creates an obligation `bar::Return` == `&str`
        "hello!"
    } else {
        // That obligation allows the type of `bar(true)` to unify with the type
        // of `"hello!"`
        bar(true)
    }
}
```

Without this logic, `foo` would fail because `foo(x - 1)` is only guaranteed to
implement `Debug`, and so cannot be added to `5`.

`bar` also fails under the current system because the if/else branches aren't
guaranteed to be the same type. By creating the obligation that `bar::Return`
be `&str`, `bar(true)` and `"hello!"` are known to be the same type within the
body of `bar` and can be returned from the if/else branches.

Full list of obligations:
1. `let x: MyImplTraitAlias = val_with_concrete type;` creates an obligation on
`MyImplTraitAlias` to match the type of `val_with_concrete_type`. Note that
this is why we can "see through" `impl Trait` in `let` bindings like
`let x: impl Fn(i32) -> i32 = |x| x + 5;`.
2. `let x: MyConcreteType = val_with_impl_trait_type;` creates an obligation on
the `impl Trait` type of `val_with_impl_trait_type` to be equal to
`MyConcreteType`.
3. Same as 1 and 2, but with `static` and `const`.
4. Returning `val_with_impl_trait_type` as a concrete type.
5. Returning `val_with_concrete_type` as an `impl Trait` type.

Other usages of a variable, such as passing it to a function, calling a method,
or accessing a field do _not_ create obligations.

In any of the above obligations, a more specific `impl Trait` type can be
substituted for a concrete type, and that will still create a new obligation.
For example:

```
type Foo = impl Debug;

fn foo() -> Foo {
    0i32
}

fn foo_times_two(x: Foo) -> Foo {
    // Creates an obligation `Foo: Add<Output=Foo> + Copy`
    let x: impl Add<Output=Foo> + Copy = x;
    x + x
}
```

## Ascription of `impl Trait` Types

Ascription of `impl Trait` types is really just one case of the
"obligations" logic used above. The key thing to note here is that, within
the same module as the `impl Trait` type is declared,
`let _: MyType = my_impl_trait_val;` can be used to create an obligation on
the concrete type of `my_impl_trait_val`, allowing it to be used more
permissively.

# How We Teach This
[how-we-teach-this]: #how-we-teach-this

`impl Trait` in `let`, `const`, `static`s and type aliases should feel like
straightforward and natural extensions of the language, and should be
introduced alongside `impl Trait`.

`impl Trait` ascription and the obligations features should be considered
advanced features, and are designed to make `impl Trait` "just work" in
most of the ways you'd naturally want to use it.
A full explanation of the obligations system should be reserved for
advanced documentation. The full explanation would likely resemble the
"detailed design" section of this RFC, rewritten to simplify the language
and omit compiler implementation concerns such as the desire to avoid
module-level inference.

# Drawbacks
[drawbacks]: #drawbacks

This feature significantly expands the number of locations in which `impl Trait`
can be used. While primarily an upside, this also means that newcomers to Rust
will be exposed to the feature early on, potentially increasing the complexity
of the onboarding experience.

This proposal also introduces a whole new concept of "obligations" which can be
challenging to reason about precisely.

# Alternatives
[alternatives]: #alternatives

We could instead expand `impl Trait` in a more focused but limited way,
such as specifically extending `impl Trait` to work in traits without
allowing full "`impl Trait` type alias".
A draft RFC for such a proposal can be seen
[here](https://github.com/cramertj/impl-trait-goals/blob/impl-trait-in-traits/0000-impl-trait-in-traits.md).

# Unresolved questions
[unresolved]: #unresolved-questions

The following extensions should be considered in the future:

- Conditional bounds. Even with this proposal, there's no way to specify
the `impl Trait` bounds necessary to implement traits like `Iterator`, which
have functions whose return types implement traits conditional on the input,
e.g. `fn foo<T>(x: T) -> impl Clone if T: Clone`.
- Associated-type-less `impl Trait` in trait declarations and implementations,
such as the proposal mentioned in the alternatives section.
