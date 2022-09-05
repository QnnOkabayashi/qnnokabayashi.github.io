---
layout: post
title: Rewriting tracing-tree
---

Among other things, I had three goals in mind:
1. Allow for ergonomically passing `Uuid`'s into spans as a pair of `u64`'s.
2. Allow for ergonomically reading `Uuid`'s from spans.
3. Allow easy configuration of log levels.

## Motivation
[#628](https://github.com/kanidm/kanidm/pull/628) on 
Kanidm regarding composing with an `EnvFilter` with
my `TreeSubscriber` made me realize that my project
would have been much better implemented on a
[`Layer`](https://docs.rs/tracing-subscriber/latest/tracing_subscriber/layer/trait.Layer.html).
`tracing-tree` is designed to just do one small
task, and semantically should be composable, making
it the perfect candidate to be a `Layer`.

I initialized a new library crate, set up my tracing
dependencies, and began the rewrite from scratch.

## The deferred `Debug` debackle
Every event that gets logged can have fields, which 
are key-value pairs containing relevant information. 
The keys are of type `'static str`, but the values
were stored as `String`s. A lot of events with a 
couple fields each adds up quickly, and I wanted 
out of heap hell no matter how long it took to 
figure out the right lifetimes.

When an event is logged in tracing, its fields are 
accessed via the visitor pattern,
where an object "visits" the event, and "records" 
fields via a restricted interface. In tracing, you 
can visit the primitive numeric types, `&str`, and 
`&dyn Debug`.

The reason I stored field values as 
`String`s was because provided a `&dyn Debug` value,
there's not much to do but format it. But for most 
of the types I cared about, I knew the erased type 
of the value was [`Arguments`](https://doc.rust-lang.org/std/fmt/struct.Arguments.html), 
so the first thing I did was naïvely replace all the 
`String`s with `Arguments` among my types,
adding the necessary lifetime parameters. I then got 
hit with an error saying that my 
types were no longer `Send + Sync`, which was 
critical because I wanted to send my log trees 
across threads for processing. Since `Arguments<'a>`
are constructed at compile time, I figured I could
get away with an `unsafe impl`, but that didn't sit
right with me. Furthermore, when I tried to downcast 
the `&dyn Debug` argument, I couldn't because 
`Arguments` isn't `'static`, and therefore cannot be 
downcasted to. To circumvent downcasting, I tried  
to `format_args!("{:?}", value)`, but was
then restricted to the same lifetime as the value, 
which was no longer than the duration of the  function.

At this point, I finally realized that no 
amount of clever lifetime handling could fix this, 
because by the time I was going to process the logs,
everything would be out of scope.

So I switched gears. Instead of avoiding allocation 
altogether, I considered how little I could get away 
with. I wanted to format everything for a log into 
a single buffer. However, this requires different 
fields being able to point into it. I considered 
[`owning_ref`](https://docs.rs/owning_ref/latest/owning_ref/),
but it looked overcomplicated and also doesn't 
support an arbitrary number of references for my 
arbitrary number of fields.
Another idea I had was to create my own 
`StringBuffer` type that somehow allowed access to 
multiple `&str` values. Something like using 
`GhostCell` but that seemed like 
overengineering that wasn't worth the potential 
benefits.
<!-- TODO: add link for ghostcell -->

In the end, I gave up and went back to `format!()`.
Sadness.

## Many `expect`s, all the same reasons
There were many times where I used `Option::expect`
in areas that were semantically identical. For
example, the `Layer` trait provides an `on_new_span` 
function that takes a `Context` and an `Id`, 
and in order to get the spans info from the 
`Context`, you have to look it up in the `Context`, 
which returns an `Option<SpanRef<_>>`. I'm pretty 
sure this is because a deeper layer could have 
removed the span first, making it not show up,
but I assumed this never happened. This meant that
every time I wanted to get the `SpanRef<_>`, I was
calling `Option::expect` and writing the same error
message every time. Now, there's nothing wrong with
this, especially because it's never supposed to 
happen. But I was bothered: what if they didn't
all say the exact same thing, even though it was the
same bug? Consistency could've been solved with a few
`const ERROR_MSG: &'static str = ...`. And although
this was more than enough, it didn't express the 
semantics behind what I was conveying. Instead, I 
opted to define custom error types with the help
of `thiserror`, so I could have a variant 
correspond which each type of error.

```rust
#[derive(Debug, Error)]
enum LayerError {
    #[error("Span not in context, this is a bug")]
    SpanNotInContext,
    #[error("Span extension doesn't contain `TreeSpanOpened`, this is a bug")]
    SpanNotInExtensions,
}
```
So instead of writing
```rust
let tree_span = span
    .extensions_mut()
    .remove::<TreeSpanOpened>()
    .expect("Span extension doesn't contain `TreeSpanOpened`, this is a bug");
```
I could instead write
```rust
let tree_span = span
    .extensions_mut()
    .remove::<TreeSpanOpened>()
    .ok_or(LayerError::SpanNotInExtensions)
    .unwrap();
```

I thought this was a great idea! But a few days 
later when I was testing the errors, I found out 
that it only prints the debug info (variant name), 
and not the 
custom error messages I made. I briefly considered 
making the variant names more verbose like
`TreeSpanOpenedNotInExtensions` so 
the debug formatting would still be clear to the 
dev what happened. But then I remembered:
this is Rust, we're allowed to have nice things.

I had overcomplicated things. Even if 
using `unwrap` showed the `Display` formatting, 
having a special custom type for it was overkill. 
The solution? A few standalone functions packaged 
in a `fail` module. This allowed me 
to have it panic with the exact error message I 
wanted, which also eliminating using `thiserror`. 
Syntactically, this was also superior, as I could 
now just say
```rust
let tree_span = span
    .extensions_mut()
    .remove::<TreeSpanOpened>()
    .unwrap_or_else(fail::span_not_in_extensions);
```
This still maintains my objectives of semantically 
separating usage from definition, but also shows 
the correct error. I also find it extremely 
satisfying to pass a non-closure function as an 
argument that is typically a closure, so I get a 
little serotonine from it as well.

<!-- TODO: maybe move to patterns section? -->
An interesting challenge that I faced was that 
`panic!()` yields the never type, which satisfies 
any type requirements because it can technically be 
called where any value is expected. However, when I 
specified my functions having the return type `!`, I 
got an error saying that it wasn't the expected type. 
I initially began by putting the exact type expected 
from the context where I called the function as the 
return type, but there was a case where one required 
a `&TreeSpanOpened` and another that required a 
`&mut TreeSpanOpened`, and I couldn't make it work.
I was able to resolve this by adding a generic `T` 
type, and returning that. This way, my errors could 
be placed in any context, and inferred to return the 
expected type, even though it will panic every time.

For `fail::subscriber_not_found`, I wanted it to take 
a generic for the subscriber so I could print the name 
using `std::any::type_name`. However, the context I was 
using it in required the return type to be `&S`, and I 
had no way of knowing what the lifetime of the return 
type would be. My first approach was to make a second 
generic, `T`, and just return that. However, this meant 
I had to write
```rust
    .unwrap_or_else(fail::subscriber_not_found<S, _>)
```
which I really didn't like. I wanted to have the second 
generic default to something so I could avoid writing it,
but I realized I could simulate this behavior by instead 
having a generic lifetime and returning `S` with that lifetime:
```rust
fn subscriber_not_found<'a, S>() -> &'a S { ... }
```
This made it so I could get away with
```rust
    .unwrap_or_else(fail::subscriber_not_found<S>)
```
Yay!

## Features, `cfg!`, and serialization
* Made everything a feature (each default processor
is its own feature, both formatters are features, 
uuids and timestamps are features)
* Stole idea for `cfg_feature! {}` macros from Tokio
* Adding custom wrapper types that have desired 
`Serialize` implementation
* `Ser` type and `Serialize2`
* Ending up using `#[serde(serialize_with = "path")]`

## Tags... with a side of generics
* Wanted to have custom tags for different types of 
projects
* Ended up with a `Tag` trait, and making almost 
every type have a generic because I didn't want to 
erase the type with `Box<dyn Tag>` because I don't 
like heap allocations.
* Actually put generics... EVERYWHERE
* Got stuck because `serde_json::to_writer` claimed 
that `Serialize` wasn't implemented for 
`Tree<T: Tag>`, even though I'm pretty sure it was.
* Couldn't figure out why it wasn't working, but I 
also wasn't happy that `operation_id` also required 
specifying more generics, since I wanted it to be 
as minimal as possible. The person shouldn't have 
to import the tag type they originally initialized 
with each time they call it!
```rust
// not fun
tracing_tree::operation_id::<Registry, MyTag>()
```
* Realized that the only reason I need it is to 
figure out how to decode the `u64` during visiting.
* Now I'm just storing a function pointer on how to 
decode it, so I can erase all the generics.
* It's funny because I tried putting generics 
everywhere in the original 
impl over the summer, and it _also_ didn't work out.
* It fucking worked! I am a genius :)
* I initially had it as a feature because I got a 
little to excited with features, but then I realized 
too many features is a bad thing, so I decided to 
remove it. Until...

## `#[derive(Tag)]`
But I later realized that I should make 
a custom proc macro to automatically derive on 
tagged types, so users couldn't accidentally screw 
up matching a unique `u64` to each variant. And any 
time you're adding a proc macro to a crate, I 
learned it's important that it has to be an optional 
feature because some people claim it has a noticable 
impact on compilation time.
* I also decided that instead of being super fancy 
and making Kanidm's implementation of a tag be super 
ergonomic, I decided to add the bare minimum for not 
making errors, and leaving the rest for downstream 
users.

Before researching how to write derive macros, I 
layed how exactly what I wanted the end product to 
look like. Here's what I came up with:
```rust
#[derive(Tag)]
enum KanidmTag {
    #[tag(info: "admin.info")]
    AdminInfo,
    #[tag(error: "request.error")]
    RequestError,
    #[tag(custom('🔐'): "security.critical")]
    SecurityCritical,
}
```
My objective was to introduce minimal complexity and 
maximum flexibility. Tags, at the abstract level, 
are made of two parts: an icon for pretty 
formatters, and a message. I wanted to minimize the 
amount of emoji copy-pasting for downstream users, 
so I added keywords for each log level that correspond 
to their appropriate icons. I didn't want to restrict 
what the messages could look like, nor did I want to 
tie them to the actual variant name, so I made it an 
additional parameter.

This is the first derive macro I've ever written,
so I needed a place to start. The docs are never a 
bad first guess, but I had an even better idea: read 
`thiserror`s source directly. VSCode's rust-analyzer 
allows me to easily find definitions of functions 
and types with just a click, allowing me to comb 
through the `#[derive(Error)]` implementation with ease.

Being able to read the source code and find 
definitions for literally anything I encounter is 
easier for me to determine how things work than just 
looking at the docs, which don't give a good idea 
of implementation details.

I initially didn't intend to be able to derive on 
structs, but I decided there could be some cases. 
This turned out to be an amazing idea, because it 
helped me recognize all the places in the macro 
where I should be moving abstract behavior into 
multiple functions. This ultimately made both 
implementations _much_ cleaner.

## Testing nice-to-haves
When making an `init` function to set up the global 
subscriber, I realized there were a few parameters
that could be configured: tag type, formatter, and 
writer. I considered making a bunch of config 
functions for each combination, before 
remembering the builder pattern common in Rust. 
But I was reluctant because builder patterns are 
verbose, and init functions are supposed to be 
quick and easy. I realized this was the perfect 
opportunity to build a macro. They're not very 
robust or composable, but are great for simple 
quick-and-dirty options, which an test init 
function absolutely is.

I also realized that you don't need to specify a 
writer, because if no formatter is provided, then 
nothing should be written. If a formatter is 
provided, it should go to 
[`TestWriter`](https://docs.rs/tracing-subscriber/latest/tracing_subscriber/fmt/struct.TestWriter.html).

Like I did with the `Tag` derive macro, the first 
thing I did was model what I wanted the syntax to 
look like.
```rust
// No tags or printing
init!{};
// Using a custom tag without printing
init!{<KanidmTag>}
// Printing without custom tags
init!{(Json::new())}
// Printing with custom tags
init!{<KanidmTag>(Json::new())}
```
My reasoning for this choice was because in Rust's 
generic functions, type parameters are wrapped in 
angle brackets, while value parameters are wrapped 
in parentheses. I wanted to provide the most 
consistency and intuition, and so I mimicked that 
idea.

But then I thought back to Tokio, and their 
beautiful `#[tokio::test]` macros... Wow I wanted 
that.
I really like the idea of being able to have a 
`tokio::test` inside of the macro.

### `tracing_forest::test`
* Config options:
    1. `tag`: defaults to NoTag
    2. `fmt`: defaults to pretty
    3. You always get the processory type associated with the asyncness of the function.
* If `sync` isn't enabled, then compile error if the function is `async`.
* Always writes to `TestWriter`. If you want it to be quiet, tell cargo
to ignore test output.
* Sets the default subscriber using a drop guard.

### `tracing_forest::main`
* Config options:
    1. `tag`: defaults to NoTag
    2. `fmt`: defaults to pretty
    3. `writer` (a MakeWriter): defaults to `io::stdout`
* If `sync` isn't enabled, then compile error if the function is `async`.
* Checks that the name is `main`.
* Sets the subscriber as a global default

I think I'm gonna restrict this to purely tokio, 
since tracing is part of the tokio family.

```rust
#[tracing_forest::test]
fn test_spans() {
    // no tags, print with TestWriter using pretty formatting
}

#[tokio::test]
#[tracing_forest::test]
async fn test_async_spans() {
    // print with TestWriter using pretty formatting, using the async processor
}

#[tokio::test]
#[tracing_forest::test(tag = "KanidmTag", fmt = "json")]
async fn test_async_everything() {
    // print with TestWriter using json formatting and KanidmTag, using async processor
    // I cannot think of ANY reason where you would want to specify a blocking processor
}


#[tokio::main]
#[tracing_forest::main]
async fn main() {
    // tokio runtime
}
```
If you want a processor to both print and log, 
just use one of tracing-subscriber's composable 
`MakeWriter`s.

## If only there was a `Visit::record_any`...
I was thinking about the lengths at which I've 
gone to send `Uuid`s and tags through tracing's
field-value API. If I were to design it myself,
I would have added a `record_any` method, so I
could downcast to the types that I wanted.
Now I'm looking into writing a PR for tracing, 

## `tracing-tree` is taken on crates.io, I need a new name
* treecing
* tracing-deep (because you can go deep in nested spans)
* tracing-2d
* tracing-forest -> I'm going with this!

## Magic with lifetimes
When I was working on an `AsyncProcessor::spawn` 
function to automatically start a processing 
thread, I ran into a strange issue:
```
error[E0597]: `make_writer` does not live long enough
  --> src/proc.rs:77:21
   |
77 |                     make_writer.make_writer().write_all(&buf[..]).unwrap();
   |                     ^^^^^^^^^^^^^^^^^^^^^^^^^
   |                     |
   |                     borrowed value does not live long enough
   |                     argument requires that `make_writer` is borrowed for `'static`
78 |                 }
79 |             });
   |             - `make_writer` dropped here while still borrowed
```
Here's my code:
```rust
pub fn spawn<F, W>(formatter: F, make_writer: W) -> (Self, impl Future<Output = Result<(), JoinError>>)
where
    F: 'static + Formatter + Send,
    W: 'static + MakeWriter<'static> + Send + Sync,
{
    let (tx, mut rx) = mpsc::unbounded_channel();

    let handle = tokio::spawn(async move {
        while let Some(tree) = rx.recv().await {
            let buf = formatter.fmt(tree);
            #[allow(clippy::unwrap_used)]
            make_writer.make_writer().write_all(&buf[..]).unwrap();
        }
    });

    let processor = AsyncProcessor::from(tx);

    (processor, handle)
}
```
The compiler was complaining that a value didn't 
live long enough, even though I owned it and `move`d 
it into the async closure.
I tried googling it, and asking my gsoc mentor,
but neither were much help.
I remembered somewhere I'd seen this pattern:
```rust
W: for<'a> MakeWriter<'a>
```
tbh I don't really know what the `for<'a>` thing does.
But I tried it and it worked!
```rust
W: 'static + for<'a> MakeWriter<'a> + Send + Sync,
```

## Disabling early exits
When using the async logger for tests, it's critical 
that the processing thread is `await`ed, otherwise the 
handle is dropped and no logs are ever processed. This 
meant adding a `handle.await` at the end of the body. 
However, I realized that if the function `return`ed 
early, it would skip the `await` call and exit the 
test without doing any log processing. To circumvent 
this, I wrapped the body in an inner function, and
`await` a call to that before `await`ing the processer 
handle. This means that if the test returns early, it 
just leaves the inner function, and the rest can safely 
clean up everything. Yay!

## How to write good documentation
Copy existing crates. So far I've copied from tracing_subscriber, tokio, and std.
I basically just copy the same style and type of information that's provided,
including links, examples, and other notes.

If I didn't copy from them, my documentation would be terrible.

```rust
let id = Uuid::new_v4();
let _entered = uuid_span!(id, Level::TRACE, "my span").entered();
```

## Format Immediate
Sometimes, something goes catastrophically wrong and it's important that 
it goes to stderr immediately.

sara:
loroush pozay
sara vi
shisaito (rly good)
dr dennis gross
dr jart
olice choice
skin suiticals (expensive but results)
ask about kdramas

gifts:
handmade art
sweet cards
useful things (?)
dates / date plans

