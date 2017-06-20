- Feature Name: impl_trait_in_traits
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Allow `impl Trait` in argument position and return types in trait
definitions and implementations:

```rust
trait Foo {
    fn foo() -> impl Fn();
    fn call_with_num(f: impl Fn(usize));
}

impl Foo for MyType {
    fn foo() -> impl Fn() {
        || println!("foo!")
    }

    fn call_with_num(f: impl Fn(usize)) {
        f(24601)
    }
}
```

Additionally, this RFC introduces associated type inference, which allows
associated types to be inferred from the return types of methods:

```rust
trait Bar {
    type Output: Debug;
    fn bar() -> Self::Output;
}

impl Bar for MyType {
    // `Output` is inferred from the return type of `bar`
    fn bar() -> &'static str { "bar!" }
}
```

# Motivation
[motivation]: #motivation

`impl Trait` allows functions to return values with types like
`impl Iterator<Item = u64>` or `impl Fn(u64) -> bool`.
Return values of type `impl Trait` are guaranteed to implement the specified
trait or traits, but nothing else about their type can be assumed.

This is useful for two main reasons:
- It allows the function author to impose an abstraction barrier, wherein the
user of the function can assume nothing about the return type other than that
it implements a given set of traits.
- It allows for returning complex or unnameable types, such as closures and
combinator types such as those produced by the `Iterator` trait.

Previous RFCs have introduced `impl Trait`, solidified its syntax, and
expanded its use to argument position.
However, `impl Trait` is still not usable in trait definitions or
implementations. This restriction means that: 
- It is still impossible to return closures from trait functions.
- Several popular combinator-heavy traits such as `Iterator` and
`Future` (from futures-rs) still produce hugely unergonomic types that ard
hard to read and understand, and nearly impossible to return from trait
functions.

This RFC expands the use of `impl Trait` to trait definitions and
implementations.

This RFC also introduces associated type inference. This feature allows
`impl Trait`-using and non-`impl Trait`-using traits to work nicely together,
as explained below.

# Detailed design
[design]: #detailed-design

## Argument Position
[detailed-design-argument-pos]: #detailed-argument-pos

Argument-position `impl Trait` in traits has a straightforward desugaring to
generic functions:

```rust
trait Foo {
    fn call_with_num(f: impl Fn(usize));
}

impl Foo for MyType {
    fn call_with_num(f: impl Fn(usize)) {
        f(24601)
    }
}

// Desugars to the following:

trait Foo {
    fn call_with_num<F: Fn(usize)>(f: F);
}

impl Foo for MyType {
    fn call_with_num<F: Fn(usize)>(f: F) {
        f(24601)
    }
}
```

This RFC starts conservatively by allowing only argument-position `impl Trait`
where both the trait definition and the trait implementation use the feature.
If `impl Trait` only appears in argument position in either the definition or
implementation, it isn't clear how many type parameters the trait method
should accept (since it would be different between the definition and
implementation). Example:

```rust
trait Foo {
    fn output_debug<T: Debug>(f: T);
}

impl Foo for MyType {
    fn output_debug(f: impl Debug) {
        println!("{:?}", f);
    }
}

// What goes here? The trait definition says `usize`, while the trait
// implementation suggests that no type parameters are necessary.
MyType::output_debug<???>(5);
```

## Return Position
[detailed-return-pos]: #detailed-return-pos

Returning an `impl Trait` type from a trait function is essentially syntactic
sugar for returning an associated type:

```rust
// In definitions:

trait Foo {
    fn foo() -> impl Fn();
}

// The above desugars to something like this:

trait Foo {
    // This associated type is anonymous-- it's hidden from the end-user.
    type FooReturn: Fn();

    fn foo() -> Self::FooReturn;
}


// In implementations:

impl Foo for MyType {
    fn foo() -> impl Fn() {
        || println!("foo!")
    }
}

// Desugars to:

impl Foo for MyType {
    type FooReturn = my_unnameable_closure_type;

    fn foo() -> Self::FooReturn {
        || println!("foo!")
    }
}
```

The two definitions and implementations above are very similar.
However, there are a few key differences between the associated-type form and
the `impl Trait` form.
The following sections will explain the similarities and differences between
`impl Trait` and associated type trait definitions and implementations.

### Return Position: `impl Trait` in Trait Definitions
[detailed_trait_def]: #detailed_trait_def

`impl Trait` in trait definitions allows you to avoid explicitly specifying
an associated type for each trait-returning method.
`impl Trait` in trait definitions has several benefits:
- It's nicely symmetrical `impl Trait` trait implementations.
- It's convenient-- you don't have to specify a separate associated type
just to allow for implementation-specific return types.
- It allows for default trait method implementations that return
complex or unnameable types:
```rust
trait MyTrait {
    fn foo() -> impl Fn() {
        || println!("Default implementation of foo!")
    }
}
```

However, `impl Trait` in trait definitions has several nuances, which I've
enumerated below.

First, types implementing an `impl Trait`-returning trait cannot name the
associated type returned by their functions.
To give an example, it is not possible to explicitly specify the return type
of `foo` here:

```rust
trait Foo {
    fn foo() -> impl Debug;
}

impl Foo for MyType {
    type XXX? = &'static str; // What do we write here?
    fn foo() -> &'static str { "foo!" }
}
```

One possible solution to this problem would be to only ever specify
`impl Trait` returning implementations for `impl Trait`-returning
method definitions. Using this strategy, the above implementation would look
like this:

```rust
impl Foo for MyType {
    fn foo() -> impl Debug { "foo!" }
}
```

However, users of `MyType` may need to depend on the fact that
`MyType::foo()` returns `&'static str`. Because of this, we also allow users
to specify explicit types as the return type of their methods, and infer
the anonymous associated types from there:

```rust
impl Foo for MyType {
    fn foo() -> &'static str { "foo!" }
}

// From that, we infer:
impl Foo for MyType {
    type FooReturn = &'static str; // Unnameable associated type
    fn foo() -> Self::FooReturn { "foo!" }
}
```

Similarly, we allow implementations to return `impl Trait` types that are
more specific than the kind used in the trait definition:

```rust
impl Foo for MyType {
    fn foo() -> impl Debug + Display + Clone { "foo!" }
}
```

This allows the trait implementation to provide the user with more
capabailities than the minimum requirements specified in the trait
definition.

The second issue with `impl Trait`-returning trait definitions
is that the anonymous associated types cannot be named in trait objects.
Normally, creating a trait object requires you to specify the values of all
of its associated types. However, anonymous associated types make this
impossible (`Box<Foo<???=&'static str>>`). Because of this,
`impl Trait`-returning methods are not object-safe, and a trait with these
methods can only be made into a trait object if all of the
`impl Trait`-returning methods have a `where Self: Sized` bound on them,
excluding them from trait objects.

Finally, it should be noted that traits with `impl Trait`-returning default
are treated equivalently to normal implementations. That is, they
introduce an anonymous associated type that can be overriden in trait
implementations.

### Return Position: `impl Trait` in Trait Implementations
[detailed_trait_impl]: #detailed_trait_impl

This RFC would also allow `impl Trait` to be used in trait implementations:
```rust
impl Foo for MyType {
  fn foo() -> impl Debug { "foo!" }
}
```

This is the most important and valuable part of this RFC.
As described previously, this feature allows users to specify an `impl Trait`
return type from a trait implementation.

The first and most obvious use of this feature is implementing a trait
with a matching `impl Trait` return type:

```rust
trait MyTrait {
    fn foo() -> impl Fn();
}

impl MyTrait for () {
    fn foo() -> impl Fn { println!("()") }
}
```

This should mostly "just work" as you'd expect. The definition will create an
anonymous associated type `FooReturn` and the implementation will provide
the type of `FooReturn`.

### Return Position: Associated Type Inference
[detailed_assoc_type_infer]: #detailed_assoc_type_infer

There's one thing still missing: the ability to turn an `impl Trait`
type from an existing trait which uses associated types.
Because all existing Rust traits use associated types, this is a severely
limiting restriction-- no existing trait would be able to be implemented
with an `impl Trait` type. In order to overcome it, this RFC proposes to allow
associated-type inference. This feature allows associated types to be inferred
from the return types of trait functions:

```rust
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<impl Fn()> {
        Some(|| println!("woo!")
    }
    ...
}

impl Iterator for MyType {
    // The value of Self::Item is inferred from the return type of `next`:
    fn next(&mut self) -> Option<impl Fn()> {
        Some(|| println!("woo!")
    }
}
```

When analyzing trait implementations, the compiler will first look for any
explicit declarations of the associated type. If it finds none, it will attempt
to unify the return types of the trait methods with the the expected signature
with the associated type left as an inference variable. For example, the above
example would try to unify `Option<usize>` with `Option<?Self::Item>`, and would
discover that `Self::Item` must be `usize`.

For the sake of consistency, this RFC also proposes to allow associated type
inference with non-`impl Trait` return types:

```rust
// Associated type inference also works for concrete types:
impl Iterator for MyOtherType {
    fn next(&mut self) -> Option<usize> {
        Some(0)
    }
}
```

This prevents users from having to switch back and forth between providing and
not providing associated types based on whether or not their return type uses
`impl Trait`.

### Return Position: Interaction with Associated Type Constructors
[detailed_atcs]: #detailed_atcs

Associated type constructors, often referred to as ATCs, allow users to specify
an associated type that is generic over one or more type or lifetime parameters.
When using `impl Trait` as the return type of a generic method, the anonymous
associated type is actually an associated type constructor:

```rust
trait Foo {
    fn foo<T>() -> impl Debug;
}

// This is sugar for:

trait Foo {
    type FooReturn<T>: Debug; // Anonymous associated type
    fn foo<T>() -> Self::FooReturn<T>;
}
```

Now that we're well on our way to supporting ATCs in general, this shouldn't
be a problem to support.

One thing worth noting here is that generic parameters in traits follow the
normal capture semantics for `impl Trait` types. That is, generic types are
automatically considered to be in scope for `impl Trait`, but lifetimes
must be manually annotated:

```rust
trait Foo {
    // The `+ 'a` is necessary here to indicate that the lifetime parameter
    // is in scope for the return type.
    fn bar<'a>(&'a self) -> impl Debug + 'a;
}

// This is sugar for:

trait Iterable {
    type BarReturn<'a>: Debug + 'a; // Anonymous associated type
    fn bar<'a>(&'a self) -> Self::BarReturn<'a>; 
}
```

# How We Teach This
[teach]: #teach

`impl Trait` return types and arguments bear a lot of similarity to returning
interfaces in traditional OOP languages.
Many of the same ideas apply--
`impl Trait` is a good way to enforce abstraction barriers,
and to make more code reliant upon the interface rather than a specific
implementing type.

`impl Trait` return types should be taught and discussed as a simple
alternative to associated return types. It may even be beneficial to teach
`impl Trait` return types first.

Associated type inference should be simple to describe to users-- associated
types can be ommitted so long as they appear in the return type of one or
more methods.

# Drawbacks
[drawbacks]: #drawbacks

Associated type inference makes associated types implicit in trait
implementations.
Specifying an associated type often feels like a burden,
but there is a real downside to inference:
it's no longer possible to determine whether
or not a trait uses an associated return type by just looking at an
implementation of the trait in question.

However, disallowing `impl Trait` return types in implementations
of legacy traits severely harms the usability of the feature.
There is the option to only infer `impl Trait`ed associated types,
but this feels like the worst of both worlds since it doesn't
solve the above drawback, and it creates an inconsistency in the language.

# Alternatives
[alternatives]: #alternatives

- Do nothing.
- Don't provide any special syntax for `impl Trait` in traits, and instead
require users to manually specify `impl Trait` associated types via
`type Foo = impl Trait;` or similar syntax. This explored in a different RFC,
and it is likely that a syntax like this will be added at some point.
However, it would eventually be nice to get to a state in which `impl Trait`
can be specified nearly anywhere a concrete type could be specified,
allowing "bare-trait" (no-`impl`) syntax to be more intuitive and in line
with the experience of using Java-esque "interfaces."

# Unresolved questions
[unresolved]: #unresolved-questions

To be resolved.
