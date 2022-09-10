---
layout: post
title: Safe borrowing across FFI with Diplomat
---

This summer, I had the privilege of interning at Google on the i18n team under
the guidance of [Manish Goregaokar](https://twitter.com/ManishEarth) and
[Shane Carr](https://twitter.com/_sffc). My team worked on a project called
[ICU4X](https://github.com/unicode-org/icu4x), which is an open source
internationalization library written in Rust. Although I contributed to this a
little near the end, I spent most of my time working on the supplemental FFI tool,
[Diplomat](https://github.com/rust-diplomat/diplomat).

# Diplomat: FFI bindings to anything

Diplomat is an open source project started by the ICU4X project, designed by
Manish and originally implemented by [Shadaj](https://twitter.com/ShadajL),
a past Google intern, and was motivated by the lack of good FFI tools that
satified ICU4X's requirements. This post focuses on how it works, but details
on the motivation can be found in the [design document](https://github.com/rust-diplomat/diplomat/blob/main/docs/design_doc.md).

Diplomat is a unidirectional[^1] Rust FFI tool that answers the question: "What
if we had FFI tools like CXX, but pluggable to any language?" At its core
lays the abstract representation of types and methods, which represents the
lowest common denominator for what types and methods look like across most
modern programming languages. This enables it to be theoretically capable of
generating bindings to any language that can support binding to C or WASM,
since it works through generated `extern "C"` functions. To top things off,
it also generates native-feeling types and methods in the target language that
wrap the WASM/C interface. Currently, Diplomat has support for generating
bindings to C, C++, C#, and JavaScript/TypeScript.

[^1]: Unidirectional FFI is when one language can invoke APIs from another
language, but not the other way around. This is most common when exposing a
library to another language.

The cost of creating bindings to all these languages is just the work involved
in writing one set of FFI API declarations, which looks mostly like normal Rust
code exposed through a module marked with the `#[diplomat::bridge]` proc macro,
which is inspired by `#[cxx::bridge]`.  So if you have some large Rust library
like ICU4X, you can attach Diplomat bindings and get native feeling APIs in every
supported language at no extra cost. We'll explore some examples together below.

## Abstract Representation

Since Diplomat aims to be language agnostic, it requires an abstract representation
that makes sense to many languages. For this reason, it's abstract representation
consists of only primitives (integers, booleans, strings), fieldless enums,
structs, and methods. By supporting these ideas that are representable to most
programming languages, Diplomat is able to reason about the types and generate
conversion code between the two over FFI when calling Rust methods.

This introduces a problem: sometimes we don't want to convert certain values
at the boundary because their internals are, well, _internal_, and we only want
to expose their functionality. This is where opaque structs come into play.

### Opaque vs Non-Opaque Structs

In general, there are two kinds of structs in programming. The first kind of
struct is the "bag 'o stuff", and has fields that are semantically disjoint.
In Rust, the most common examples are tuples, but it also shows up in config
types frequently. The second kind are opaque structs which usually have some
more advanced functionality. An example is Rust's `HashMap`, where the user
shouldn't particulary care about the exact inner fields (which are private anyway!),
and only cares about the functionality it provides.

Diplomat distinguishes between these two kinds of structs, and the programmer
can communicate this by marking opaque structs with the `#[diplomat::opaque]`
attribute. Structs marked as opaque have special properties in Diplomat's model:
1. Fields are private from the other language, only methods can be used.
2. Fields can be _any_ concrete type, since Diplomat doesn't examine them.
3. Can only cross the FFI boundary behind a pointer, meaning conversion across
the boundary is constant and minimal.
4. Boxed opaques can only be returned, but not accepted as a parameters because
many other languages don't have a way to give up ownership.

On the other hand, structs not marked with the `#[diplomat::opaque]` attribute
have nearly opposite properties:
1. Fields are public to the other language.
2. Fields can only be primitive types like numbers, booleans, characters,
owned opaques, and other structs. My project extended this to also allow for
borrowed opaques, string slices, and slices of primitives.
3. Can only cross the FFI boundary by value, meaning that every field is also
converted at the boundary.

In JavaScript, for example, "bag 'o stuff" structs are converted into classes
with public fields and methods associated to the Diplomat-exposed methods, but
it's worth noting that it will accept any `Object` with the right fields in
method arguments.

# My Intern Project: Lifetimes in Diplomat

Among other things, Rust's type system allows for expressing mutability,
ownership, and lifetimes, concepts that most other languages simply do not have.
My work this summer was to extend Diplomat's abstract model to figure out how we
can take advantage of Rust's precise control over these factors across the FFI
boundary, particularly with returning references to borrowed data.

One of the problems that Rust's lifetime system solves is dangling pointers[^2],
and it does this statically by preventing references from outliving their owners.
But rustc can only do static analysis on pure Rust code, and this is the main
problem with returning pointers across FFI: it's not just pure Rust code.

[^2]: XOR aliasing is also solved with Rust's lifetime system, and this is
[being worked on](https://github.com/rust-diplomat/diplomat/issues/225).

The solution, instead, is to make Diplomat enforce whatever technique the other
language uses to prevent dangling references. In garbage collected languages
like JavaScript, this involves holding a reference to the owner to stop it from
being cleaned up early. In manually memory-managed languages like C++, this
involves generating documentation notifying the programmer of what lifetime
relationships must be upheld.

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
parameter and the returned `&'a Bar` type, and sees that they both contain `'a`.
Here, `'a` is the lifetime of some `Bar` value, but Diplomat has no way of
knowing where that `Bar` value is, or how long it lives for. But it does know
that however long it lives for, `Foo<'a>` is guaranteed to outlive it.
Therefore, by enforcing that `&self` value outlives the return value,
Diplomat can ensure that there are no dangling references.

In the case that this API requires JavaScript bindings, Diplomat would use this
lifetime relationship to hold a reference from the output to the input that it
borrows from to generate an API that resembles the following:
```js
class Foo {
    // ...

    get_bar() {
        // `this.underlying` is the ptr to the Rust opaque
        const bar = wasm.Foo_get_bar(this.underlying);
        // hold a reference so the GC doesn't clean up `this` before `bar`
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
Fundementally, this idea is straightforward and not too difficult to implement...
right?

## Challenge 1: Lifetime Subtyping

Wrong.

Some people want to watch the world burn. Some people want to use lifetime bounds.
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
Unless your writing borrowing iterators or zero-copy `Deserialize` implementations
by hand, you will probably never need this.

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
eventually lead to dangling pointers. This works in Rust because the `'a: 'b`
bound allows`'a` to coerce into `'b`, meaning that if we're looking for all
inputs that live at least `'b`, then we should also count types with `'a`.
Therefore, we have to be cognizant of these bounds as well.
But it goes even further:
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
_Why? Why would you ever write this?_

Here, the output has lifetime `'c`, and the input has lifetime `'a`, but it
doesn't explicitly say `'a: 'c`. Instead, it's _inferred_ via transitivity.
Do you see where this is going?

Here's an exaggerated example of a `where` clause:
```rust
'a: 'b,
'b: 'c,
'c: 'e,
'd: 'b,
'e: 'd + 'f,
```
This looks lot like an adjacency list...

Ah. It's a directed graph problem.

If you think of lifetimes as vertices and bounds as edges pointing from longer
to shorter lifetimes, then determining possible coercions becomes a
simple reachability problem. The diagram below shows the exaggerated example
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

With what we've discussed to far, this is what the algorithm looks like:
1. Traverse the output type, collecting any lifetimes encountered along the way.
2. Traverse the input types. When some lifetime A is encountered, perform a
depth-first search on shorter lifetimes while tracking visiting to break cycles.
If one of the output lifetimes is encountered, add the parameter containing
lifetime A to the list of parameters that the output borrows from.
3. In code generation, use this information to communicate to the other language
that there's a borrowing relationship as described in the previous section.

This is all great, but take a moment to hear me out. I'm not saying I don't
trust my implementation (it's tested), but if you're relying on the transitivity
of lifetime bounds across arbitrarily many lifetimes, your house is eventually
getting egged by someone forced to read your code. So let's move on.

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
        pub fn extract(self) -> &'a Opaque {
            let out = self.data;
            out
        }
    }
}
```
When rustc sees `Input::extract`, it requires that `self.data ≤ 'a ≥ out`, and
it's able to statically enforce this because rustc is omniscient on pure Rust code.
Diplomat, on the other hand, cannot perform static analysis since it cannot peek
into the other side of the code.
With the lifetime design discussed so far, Diplomat would require
`'a ≥ self.data ≥ self ≥ out`, since the current idea doesn't detect which
field is being borrowed from. This is an overly conservative approach to
enforcing `'a ≥ out`.

Being overly conservative can prevent some painfully obvious programs from being
accepted. In C++ for example, Diplomat would generate documentation saying that
you that you could have a dangling reference if some wrapper struct goes out of
scope, even it's clearly not true:
```cpp
const Opaque& do_nothing(const Opaque& data) {
    // Put `data` into the struct
    Input input;
    input.data = data;

    // Extract `data` from the struct
    const Opaque& out = Input::extract(input);

    // STOP! Diplomat says you'll have a dangling ref
    // if `input` goes out of scope before `out` does.
    return out;
}
```
This code is completely safe since it just returns the reference that was
originally passed in, but Diplomat will still generate documentation saying that
this should leave a dangling pointer since `Input::extract` thinks that `self`
must outlive `out`. Equivalent generated JavaScript bindings would just hold a
reference and chug along normally, consuming slightly more memory than usual.

Ideally, we want to be able to know that the output of `Input::extract` isn't
bound to `self`, but instead bound to whatever `self.data` is bound to.
This would eliminate the requirement of having `self` outlive `out`, and
making Diplomat only have to enforce `'a ≥ self.data ≥ out`.

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
        fn get_data(self) -> Output<'a> {
            let out = Output { data: self.data }
            out
        }
    }
}
```
This follows similarly to the previous example, except we observe that
`out.data` has to live `'a`, and that `self.data` is guaranteed to live `'a`,
so we can just enforce that `self.data` has to outlive `out.data` to uphold
the invariant that `out.data` lives at least `'a`.

## Challenge 3: Naming Lifetimes
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
        fn get_data(self) -> Output<'a> {
            let out = Output { data: self.data }
            out
        }
    }
}
```
The lifetimes in the declarations of `Input` and `Output` have completely
different names. We can't just track `'a` all the way down since `'a` is being
mapped to other lifetimes. Instead, we have to track lifetimes by the index they
appear at in generic arguments, and use that to determine which lifetime is which.
This is similar to what rustc does.

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
        fn get_data<'b>(self) -> Output<'b>
        where
            'a: 'b,
        {
            let out = Output { data: self.data }
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
of the inputs, which consists of just `Input<'a>`. Since `Input` is a non-opaque
struct, we look in the declaration of `Input` and store a mapping saying that the
position of `'i` in the context of `Input`'s declaration corresponds with the
position of `'a` in `Input<'a>` in the method parameter. Inside of `Input`s
declaration we see the `data: &'i Opaque` field, so we map the lifetime `'i`
back to `'a`. At this point, we have to consider that `'a` may coerce into
other lifetimes, so we perform a depth-first search in the lifetime graph to
find all shorter lifetimes. This search results in finding `'b`. After confirming
that `'b` appears in the output, we remark that `self.data` is one of the input
fields that lives at least `'b`.

In code generation, we put these pieces of information together to determine
that `out.data` must be outlived by `self.data`. With this information, for
example, JavaScript would handle this by making `out.data` hold an reference to
`self.data` to ensure that values are cleaned up in the right order, and C++
would have generated documentation specifying which the lifetime invariant that
needs to be upheld.

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

Each of these challenges made me consider how I would have done things differently
to make everything easier given the opportunity to. This lead to me designing the
high-level intermediate representation (HIR), which will eventually be the
interface that language backends interface with to generate bindings.

# Next Steps: HIR
The HIR is an ongoing endeavor that attempts improve on the suboptimal design
decisions of the past by making everything more coherent. I designed and
implemented it over the course of my internship from the ground up around three
core ideas:
1. Make invalid states unrepresentable to reduce `.unwrap()`s.
2. Make types more convenient for writing backends by breaking apart large `enum`s.
3. Make lifetime tracking a first-class feature.

It will make all the challenges I listed above non-issues for folks implementing
Diplomat backends, along with many other quality-of-life changes by serving as
an interface between Diplomat's naive abstract representation and the language
backends.

At this point, the type system is well defined and implemented, and the code
to convert Diplomat's current abstract representation into the HIR is implemented
but not thoroughly tested. But the roadmap for full integration is there, and
there's continuous ongoing work to make it the standard in Diplomat's model.

