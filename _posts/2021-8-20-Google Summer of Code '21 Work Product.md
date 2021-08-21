---
layout: post
title: Google Summer of Code '21 Work Product
---

# My project: integrating Tokio `tracing` into Kanidm
> Some clarification: my project is not at all related to my proposal. My mentor, William, made it clear before day 1 that I wasn't strictly bound to my proposal, and I decided I wanted to pursue other avenues.

My main project, Tracing-Tree, was inspired by [🪵 The 🪵 Great 🪵 Log 🪵 Issue 🪵 #453](https://github.com/kanidm/kanidm/issues/453), which outlined the necessity to upgrade Kanidm's current logging framework to support configurable JSON logs and still maintain correlatability, meaning events _and_ their context within logs.

I decided to take on this task, which consisted of research on:
1. Existing logging frameworks.
2. Kanidm's requirements for logging (e.g. performance profiling, structured logging)

Once I was familiar enough with Kanidm and the framework I chose, [`tracing`](https://crates.io/crates/tracing), With William's mentorship, I used `tracing`'s core features to build a logger capable of flexible, structured, and concurrent logging that fit seamlessly into Kanidm's [`tide`](https://crates.io/crates/tide) backend.

The result was a flexible, lightning fast logger that, most importantly, abstracts away many previously existing logging details from other areas of the code base.

# Code I wrote ⌨️
For details on how I spent my time, technical challenges I overcame, and interesting ideas I had along the way, see my previous blog posts at qnnokabayashi.github.io.

## Pull requests
In the beginning of my project, I started with many minor changes to familiarize myself with the code base. Here are the PR's I sent for those:
* Renamed fields in dbvalue [#477](https://github.com/kanidm/kanidm/pull/477)
* `kanidm_client` bool/return values [#479](https://github.com/kanidm/kanidm/pull/479)
* `kanidm_tools` cleanup [#482](https://github.com/kanidm/kanidm/pull/482)
* Removed `OperationResponse` [#489](https://github.com/kanidm/kanidm/pull/489)

This is the main PR corresponding to my project. It contains information on the process of integrating my `tracing` interface with Kanidm directly.
* Customized `tracing` for `tide::Middleware` logging [#544](https://github.com/kanidm/kanidm/pull/544)

## Related repositories
When I was developing my logging tool using `tracing`'s core functionality, I created my own repo to keep compilation times down while I was experimenting.
* https://github.com/QnnOkabayashi/tracing-tests

# Moving forward 🚀
The core functionality combining `tracing` to Kanidm is complete, and already merged into Kanidm. However, the full integration into the code base isn't quite complete yet, but the process is very straight forward (and tedious).

If you're looking to extend my work, great! The idea right now is to integrate my work alongside `AuditScope`, and once that's finished, eventually wipe `AuditScope`'s existance from the face of the Earth. TL:DR: don't delete `AuditScope` stuff (yet).

Here's the workflow for integrating my changes:

## `use` the `tracing` log macros
Import `tracing`'s log macros wherever a regular log is used, so that `tracing` macro is called and not the `log` variant.

If you see this:
```rust
error!("Something went very wrong");
```
You should make sure that this is at the top:
```rust
use tracing::error;
```
This ensures that it's the `tracing` macro that gets invoked, and not `log`'s. I'm pretty sure theres a way to get around doing this so that `tracing` can pick up on `log` macros, but that's an area I haven't researched much.

## Change the contextual logging macros
Change `AuditScope`'s special logs (e.g. `lfilter_info`) to the `tracing-tree` macros (e.g. `filter_info`). All of my custom macros are included in `kanidmd::prelude`, so no need for manual imports. Note that my macros support fields of key-value pairs, so a log that may initially be:
```rust
lfilter_error!(au, "failed -> {:?}", e);
```
should be turned into:
```rust
filter_error!(?e, "failed");
```
Note that the `?` sigil means the value will use its `fmt::Debug` representation, and the `%` sigil corresponds to the `fmt::Display` representation. `tracing`'s documentation contains more formatting, but the macros essentially just wrap the [`event` macro](https://docs.rs/tracing/0.1.26/tracing/index.html#using-the-macros).

## Switch the `AuditScope` segment macros with `spanned`
This one is straightforward, but there's one important detail. `spanned` can take either a block or a closure, difference being that a block containing `return` or `?` can short-circuit the outer function, while a a closure will only exit the closure. For `spanned` calls that wrap an entire function, this doesn't matter, but this comes in handy when wrapping a small piece of logic that may intend to short-circuit its outer function.

Note that the closure variant desugars to:
```rust
trace_span!(/* args before closure here */).in_scope(/* closure here */);
```

Whereas the block variant desugars to:
```rust
{
    let _entered_span = trace_span!(args before block here).entered();
    /* block here */
    // `_entered_span` dropped, span exits
}
```

This means they can still accept key-value pairs just like `tracing`'s other built in macros. However, fields other than `uuid` are ignored. (You can change the source code for this if you have good reason for it).

## Instrument `async` functions
`async` functions that wrap their body in one span must be `instrument`ed, since spans and `async` don't play very nicely. 
```rust
#[instrument(
    level = "trace",
    name = "search",
    skip(self, uat, req, eventid)
    fields(uuid = ?eventid)
)]
pub async fn handle_search(
    &self,
    uat: Option<String>,
    req: SearchRequest,
    eventid: Uuid,
) -> Result<SearchResponse, OperationError> {
```
Note that `instrument` takes many arguments. Let's break down what's happening:
* Set the level to `TRACE`
* Set the span's name to `search`
* Tell `instrument` to not store `self`, `uat`, `req`, and `eventid` as key-value pairs.
* Use a key-value pair with `uuid` and the `fmt::Debug` representation of `eventid`.

In my implementation, providing a span with the field `uuid` will notify that span to use that as its operation ID, which shows up during logging. If none is provided _and_ the span is a root span, a random one is generated. If it's not the root and none is provided, it will use that parent span's operation ID.

More documentation for `instrument` can be found [here](https://docs.rs/tracing-attributes/0.1.15/tracing_attributes/attr.instrument.html).
