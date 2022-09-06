---
layout: post
title: Diplomat @ Google
---

This summer, I had the privilege of interning at Google on the i18n team under
the guidance of Manish Goregaokar and Shane Carr. My team worked on a
project called [ICU4X](https://github.com/unicode-org/icu4x), which is an open
source internationalization library written in Rust. Although I contributed to
this a little near the end, I spent most of my time working on the supplemental
FFI tool, [Diplomat](https://github.com/rust-diplomat/diplomat).

# Diplomat: FFI bindings to anything

Diplomat is an open source project started at Google, and was motivated by the
lack of good FFI tools that satified ICU4X's requirements. This post focuses on
how it works, but details on the motivation can be found in the [design document](https://github.com/rust-diplomat/diplomat/blob/main/docs/design_doc.md).

Diplomat is a unidirectional FFI tool that allows programs in other languages
to call Rust libraries via automatically generated, idiomatic APIs. At its core
lays the abstract representation of types and methods, which represents the
lowest common denominator for what types and methods look like across most
modern programming languages. Since the abstract representation is so general,
Diplomat is theoretically capable of generating bindings to any language that
Rust has FFI support for, namely JavaScript via WebAssembly, and languages that
support C FFI. Currently, it has support for generating bindings to C, C++, C#,
and JavaScript/TypeScript.  Diplomat also generates native-feeling types and
methods in the target language that call into the raw WASM/C interface.

The cost of creating bindings to all these languages is just one set of FFI
bindings, which are exposed through a module marked with the
`#[diplomat::bridge]` proc macro. So if you have some large Rust library like
ICU4X, you can attach Diplomat bindings and get native feeling APIs in every
supported language at no extra cost.

# Abstract Representation

In general, there are two kinds of structs in programming. The first kind of
struct has values that are semantically disjoint, where fields can typically be
mutated without breaking things. In Rust, the most common examples are tuples,
but it also shows up in config types frequently. The second kind are opaque
structs which usually have some more advanced functionality. An example is
Rust's `HashMap`, where the user shouldn't particulary care about the exact
inner fields, and only cares about the functionality.

Diplomat distinguishes between these two kinds of structs, and the programmer
can communicate this by marking opaque structs with the `#[diplomat::opaque]`
attribute. Structs marked as opaque have special properties in Diplomat's model:
1. Fields are private from the other language, only methods can be used.
2. Fields can be _any_ valid Rust values, since Diplomat doesn't examine them.
3. Can only cross the FFI boundary behind a pointer, meaning there is no
conversion cost.
4. Boxed opaques can only be returned, but not accepted as a parameters because
most other languages don't have a way to give up ownership.

On the other hand, structs not marked with the `#[diplomat::opaque]` attribute
have nearly opposite properties:
1. Fields are public to the other language.
2. Fields can only be primitive types like numbers, booleans, characters,
owned opaques, and other structs. My project extended this to also allow for
borrowed opaques, string slices, and slices of primitives.
3. Can only cross the FFI boundary by value, meaning that every field is also
converted at the boundary.

Additionally, Diplomat also supports fieldless enums, which are just passed
as their discriminents across the boundary and then converted appropriately.

# My Intern Project

Rust's type system allows for expressing mutability, ownership and lifetimes,
concepts that most other languages simply do not have. My work this summer was
to extend Diplomat's abstract model to figure out how we can take advantage of
Rust's precise control over these factors across the FFI boundary, particularly
with returning references to borrowed data.

One of the problems that Rust's lifetime system solves is dangling pointers,
and it does this statically by preventing references from outliving their owners.
But rustc can only do static analysis on pure Rust code, and this is the main
problem with returning pointers across FFI (aside from XOR aliasing, which is
[being worked on](https://github.com/rust-diplomat/diplomat/issues/225)).

The solution is to instead make Diplomat enforce whatever technique the other
language uses to prevent dangling references. In garbage collected languages
like JavaScript, this involves attaching an edge to the owner to stop it from
being GCd early. In manually memory-managed languages like C++, this involves
generating documentation notifying the programmer of what lifetime relationships
must be upheld.

Let's walk through a simple example:
```rust
#[diplomat::bridge]
mod ffi {
    #[diplomat::opaque]
    pub struct Bar {
        // ...
    }

    #[diplomat::opaque]
    pub struct Foo<'a> {
        bar: &'a Bar,
    }

    impl<'a> Foo<'a> {
        pub fn get_bar(&self) -> &'a Bar {
            &self.bar
        }
    }
}
```
When Diplomat sees `Foo::get_bar`, it sees the expanded `self: &Foo<'a>`
parameter and the returned `&'a Bar` type, and infers that the output shouldn't
outlive the `&self` parameter because they both have `'a`.

In the case that this API requires JavaScript bindings, Diplomat would use this
lifetime relationship to attach an edge from the output to the input that it
borrows from to generate an API that resembles the following:
```js
class Foo {
    // ...

    get_bar() {
        // `this.underlying` is the ptr to the Rust opaque
        const bar = wasm.Foo_get_bar(this.underlying);
        bar.fooRef = this;
        return bar;
    }
}
```
On the other hand, it's up to the programmer to keep track of these "outlives"
relationships in manually memory-managed languages. In C++, Diplomat generates
documentation that notifies the programmer of what these relationships are:
```cpp
class Foo {
    // ...
public:
    /* Lifetimes: the returned value must be outlived by: `this`.
     */
    const Bar& get_bar() { /* ... */ }
}
```
Fundementally, this idea is very straightforward and not too difficult to
implement. Right? _Right?_

## Challenge 1: Lifetime Coercion

Unfortunately, not all code lives on this happy path. Check this out:
```rust
// `Foo` and `Bar` same as above
impl<'a> Foo<'a> {
    pub fn get_bar<'b>(&self) -> &'b Bar
    where
        'a: 'b,
    {
        &self.bar
    }
}
```
To be honest, I've never had to use lifetime bounds outside of borrowing
`Deserialize` impls, and Diplomat doesn't support generic types, so I'm not sure
why you'd need bounded lifetimes. But they exist, and I didn't want to leave it
as someone else's problem.

The problem here is that the output looks like this:
```rust
&'b Bar
```
While the input looks like this:
```rust
self: &Foo<'a>
```
**These are not the same lifetime**. If you just look in the input for `'b`,
you'll find that it doesn't borrow from the input, and this can and will
eventually lead to dangling pointers. But this works in Rust because the
`'a: 'b` bound allows`'a` to coerce into `'b`. Therefore, we have to be
cognizant of these bounds as well. But it goes even further:
```rust
impl<'a> Foo<'a> {
    pub fn get_bar<'b, 'c>(&self) -> &'c Bar
    where
        'a: 'b,
        'b: 'c,
    {
        &self.bar
    }
}
```
Continuing the theme from above, I have no idea why you would ever write this,
especially in a set of FFI bindings. But it's legal Rust code and I wanted to
be consistent with the language.

Here, the output has lifetime `'c`, and the input has lifetime `'a`, but it
doesn't explicitly say `'a: 'c`. Instead, it's _inferred_ via transitivity.
Do you see where this is going?

Here's an exaggurated example of a `where` clause:
```rust
'a: 'b,
'b: 'c,
'c: 'e,
'd: 'b,
'e: 'd + 'f,
```
This looks lot like an adjacency list. That's right, this is a directed graph
problem.

If you think of lifetimes as vertices and bounds as edges pointing from longer
to shorter lifetimes, then determining possible coercions becomes a
simple reachability problem. The diagram below shows the exaggurated example
as a directed graph:
<figure>
    <p align="center">
        <img src="{{site.baseurl}}/images/google-internship/directed-graph.png" alt="A black-on-white image of a directed graph with 6 vertices labeled A through F. A points to B, B points to C, C points to E, E points to D and F, and D points back to B. It clearly shows that B, C, E, and D form a cycle." width="400"/>
        <figcaption>Graph representation of lifetimes and bounds</figcaption>
    </p>
</figure>

With this model, we can easily generalize the definition of what it means for
some lifetime A to be able to coerce into some lifetime B. That is, lifetime A
can coerce into lifetime B if and only if there exists a path from A to B on
the directed graph of lifetimes and bounds. In practice, this also mean that
all lifetimes in a cycle are all the same lifetime.

So here's the gameplan:
1. Traverse the output type, collecting any lifetimes encountered along the way.
2. Traverse the input types. When some lifetime A is encountered, perform a
depth-first search on shorter lifetimes while tracking visiting to break cycles.
If one of the output lifetimes is encountered, add the parameter containing
lifetime A to the list of parameters that the output borrows from.
3. In code generation, use this information to communicate to the other language
that there's a borrowing relationship as described in the previous section.

Using this strategy, I was able to make Diplomat understand arbitrary lifetime
bounds. Hey, that wasn't so bad!

## Challenge 2: Borrowing from Struct Fields
In the type philosophy section, I mentioned how non-opaque struct fields are
publically accessible across the FFI boundary. What happens when we copy a
borrow from a struct field?
```rust
#[diplomat::bridge]
mod ffi {
    #[diplomat::opaque]
    pub struct Opaque {
        // ...
    }

    pub struct Input<'a> {
        data: &'a Opaque,
    }

    impl<'a> Input<'a> {
        pub fn create(data: &'a Opaque) -> Self {
            Self { data }
        }

        pub fn extract(input: Input<'a>) -> &'a Opaque {
            let out = input.data;
            out
        }
    }
}
```
When rustc sees `Input::extract`, it requires that `input.data ≤ 'a ≥ out`, and
it's able to statically enforce this because rustc is omniscient on pure Rust code.
Diplomat, on the other hand, does FFI, where nothing is statically omniscient.
With the current solution, Diplomat would require `'a ≥ input.data ≥ input ≥ out`,
since the current idea doesn't detect which field is being borrowed from.
This is an overly conservative approach to enforcing `'a ≥ out`.

Being overly conservative can prevent some painfully obvious programs from being
accepted. In C++ for example, Diplomat would tell you that you could have a
dangling reference if some wrapper struct goes out of scope, even it's clearly
not true:
```cpp
const Opaque& do_nothing(const Opaque& data) {
    // Put `data` into the struct
    auto input = Input::create(data);

    // Extract `data` from the struct
    auto out = Input::extract(input);

    // STOP! Diplomat says you'll have a dangling ref
    // if `input` goes out of scope before `out` does.
    return out;
}
```
This code is completely safe since it just returns the reference that was
originally passed in, but Diplomat will still generate documentation saying that
this should leave a dangling pointer since `Input::extract` thinks that `input`
must outlive `out`. Equivalent generated JavaScript bindings would just add an
edge and chug along normally, consuming slightly more memory than usual.

Ideally, we want to be able to know that the output of `Input::extract` isn't
bound to `input`, but instead bound to whatever `input.data` is bound to.
This would eliminate the requirement of having `input` outlive `out`, and
making Diplomat only have to enforce `'a ≥ input.data ≥ out`.

This is a lot easier said than done though. For starters, checking for lifetime
relationships now involves recursing down non-opaque structs and their fields,
and if we encounter a borrowed string, slice, or opaque, we have to remember
not only which lifetime it corresponds to, but also where in the type tree it
appeared. Take the following example:
```rust
#[diplomat::bridge]
mod ffi {
    #[diplomat::opaque]
    pub struct Opaque {
        // ...
    }

    pub struct Second<'a> {
        data: &'a Opaque,
    }

    pub struct First<'a> {
        second: Second<'a>,
    }

    impl<'a> First<'a> {
        fn get_data(first: First<'a>) -> &'a Opaque {
            first.second.data
        }
    }
}
```
Remember that Diplomat only looks at function signatures, so we can't just look
at the `first.second.data` staring the user in the face. What happens instead
is that we see that `'a` is in the output type, `&'a Opaque`. Then, we traverse
the inputs and see `first: First<'a>`. We see that it's a non-opaque struct,
so we look into its fields and find `second: Second<'a>`. That's also a
non-opaque struct, so we look into _its_ fields and see `data: &'a Opaque`.
Since `Opaque` is opaque, Diplomat isn't concerned with its fields, so we're
done recursing. Once we reach this point, we then have to remember that
the output borrows from `first.second.data`. In the code generation, we can
then reflect this information, making the C++ example above not generate
overly conservative documentation.

This is only half the problem, though. What about non-opaque borrowing structs
in the output?
```rust
#[diplomat::bridge]
mod ffi {
    #[diplomat::opaque]
    pub struct Opaque {
        // ...
    }

    pub struct Input<'a> {
        data: &'a Opaque,
    }

    pub struct Output<'a> {
        data: &'a Opaque,
    }

    impl<'a> Input<'a> {
        fn get_data(input: Input<'a>) -> Output<'a> {
            let out = Output { data: input.data }
            out
        }
    }
}
```
This follows similarly to the previous example, except we observe that
`out.data` has to live `'a`, and that `input.data` is guaranteed to live `'a`,
so we can just enforce that `input.data` has to outlive `out.data` to uphold
the invariant that `out.data` lives at least `'a`.

## Challenge 3: Index-based Lifetimes
Tracking the "outlives" relationship on a field-by-field basis is great, but
it introduces another challenge: lifetimes can go by different names in
different declarations:
```rust
#[diplomat::bridge]
mod ffi {
    #[diplomat::opaque]
    pub struct Opaque {
        // ...
    }

    pub struct Input<'i> {
        data: &'i Opaque,
    }

    pub struct Output<'o> {
        data: &'o Opaque,
    }

    impl<'a> Input<'a> {
        fn get_data(input: Input<'a>) -> Output<'a> {
            let out = Output { data: input.data }
            out
        }
    }
}
```
The lifetimes in the declarations of `Input` and `Output` have completely
different names. We can't just track `'a` all the way down, which sucks because
that would have been easy. Instead, we have to track lifetimes by the index they
appear at in generic arguments, since that's what rustc does.

## The Big Picture
Let's put this all together and walk through an example of a method with
lifetime bounds, non-opaque structs in the input and output positions,
and renamed lifetimes:
```rust
#[diplomat::bridge]
mod ffi {
    #[diplomat::opaque]
    pub struct Opaque {
        // ...
    }

    pub struct Input<'i> {
        data: &'i Opaque,
    }

    pub struct Output<'o> {
        data: &'o Opaque,
    }

    impl<'a> Input<'a> {
        fn get_data<'b>(input: Input<'a>) -> Output<'b>
        where
            'a: 'b,
        {
            let out = Output { data: input.data }
            out
        }
    }
}
```
The first thing that Diplomat does when it encounters `Input::get_data` is
examine the return type signature, `Output<'b>`. Since it's a non-opaque struct,
we look in the declaration of `Output` and store a mapping
saying that the position of `'o` in the context of `Output`'s declaration
corresponds with the position of `'b` in `Output<'b>` in the method. Next, we
see the `data: &'o Opaque` field, and use the mapping to determine that it's
actually `&'b Opaque` in the context of the method. We then remark that
`out.data` relies on `'b`, see that there's no more fields to explore in the
field traversal, and finish.

Once we know which lifetimes to look out for (just `'b`), we traverse the types
of the
inputs, which consists of just `Input<'a>`. Since `Input` is a non-opaque struct,
we look in the declaration of `Input` and store a mapping saying that the
position of `'i` in the context of `Input`'s declaration corresponds with the
position of `'a` in `Input<'a>` in the method. Inside of `Input`s declaration
we see the `data: &'i Opaque` field, so we map the lifetime `'i` back to `'a`,
and then perform a depth-first search in the lifetime graph to find all shorter
lifetimes.  This search results in finding `'b`. After confirming that `'b`
appears in the output, we remark that `input.data` is one of the input fields
that lives at least `'b`.

In code generation, we put these pieces of information together to determine
that `out.data` must be outlived by `input.data`, which is exactly what we
were aiming for.

# Results
That's a lot of details, and unfortunately it's not 100% integrated yet.
Thus far, the lifetime coercion detection using depth-first search
is entire implemented, but tracking borrows on a field-by-field basis is only
implemented in the JavaScript backend. Aside from not understanding the full
scope of the problem before starting, the biggest challenge was that Diplomat's
abstract representation was not designed with lifetimes in mind, and also
wasn't condusive to writing robust code.

For example, traversing down structs recursively and maintaining a mapping from
lifetimes in a declaration to lifetimes in the method was bolted on at the end
after I'd already written a significant amount of code. Additionally, Diplomat's
model allows for a bunch of illegal states like an opaque struct passed by value.
Obviously there are validity checks, the loose type system eventaully pollutes
backends with `.unwrap()` and `unreachable!()` statements. On top of this, the
existing JavaScript code generation was really challenging to understand and
work with, and I ended up spending a week rewriting it to be more modular.

# Next Steps
Diplomat is relatively a new project, and we're still making a lot of decisions
that we're unsure about. Since I was adding such a large feature to Diplomat's
core model, a lot of these older design decisions came back to bite me, and
I began to consider how I would have implemented Diplomat knowing what I know
now. Additionally, Manish and I had been discussing how to more concretely
define Diplomat's type model, particularly around input and output types which
is a whole other topic. This was the beginning of the high-level intermediate
representation (HIR).

The HIR is an ongoing endeavor that attempts improve on the suboptimal design
decisions of the past by making everything more coherent. I designed it from
the ground up around three core ideas:
1. Make invalid states unrepresentable.
2. Make types more convenient for writing backends.
3. Make lifetime tracking a first-class feature.

It will make all the challenges I listed above non-issues for folks implementing
Diplomat backends, along with many other quality-of-life changes by serving as
an interface between Diplomat's naive abstract representation and the language
backends.

So far, my first iteration of the model is in the repository, as well as the
lowering code for converting Diplomat's old representation to this new
representation, but there's continuous ongoing work to make it the standard
in Diplomat's model.

