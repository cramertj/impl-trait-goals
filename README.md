# impl-trait-goals
An (experimental) RFC repo devoted to Rust's `impl Trait`.

This repo is an effort to collect and organize the discussion around
future `impl Trait` RFCs.

## Current RFCs and overall vision
`impl Trait` allows functions to leave their return types "abstract."
Abstract return types allow a function to hide a concrete return type behind a
trait interface similar to trait objects, while still generating the same
statically-dispatched code as with concrete types.

The initial conservative proposal for `impl Trait` was accepted as
[RFC 1522 - Conservative `impl Trait`.](https://github.com/rust-lang/rfcs/blob/master/text/1522-conservative-impl-trait.md)
This proposal was expanded upon in
[RFC 1951 - Expand `impl Trait`](https://github.com/rust-lang/rfcs/blob/master/text/1951-expand-impl-trait.md),
which settled the `impl Trait` syntax design, resolved questions around type
and lifetime parameter scopes, and added `impl Trait` to argument position.

However, there are still more features left to be added before the `impl Trait`
story can be considered complete. This repo aims to track information and
discussion of these additional features, and to address them through a followup
RFC (possibly multiple).

## How the issues are organized

### Help-wanted
The [`help-wanted` label](https://github.com/cramertj/impl-trait-goals/issues?q=is%3Aopen+is%3Aissue+label%3A%22help+wanted%22)
indicates issues where a PR would be most welcome.

### Priority (the P labels)
- [P-essential](https://github.com/cramertj/impl-trait-goals/issues?utf8=%E2%9C%93&q=is%3Aopen%20is%3Aissue%20label%3A%22P-essential%22)
means that we have to resolve this.
- [P-nice-to-have](https://github.com/cramertj/impl-trait-goals/issues?utf8=%E2%9C%93&q=is%3Aopen%20is%3Aissue%20label%3A%22P-nice-to-have%22)
means that it might be nice to include this feature.
- [P-wild-and-crazy](https://github.com/cramertj/impl-trait-goals/issues?utf8=%E2%9C%93&q=is%3Aopen%20is%3Aissue%20label%3A%22P-wild-and-crazy%22)
means that theses are far-out ideas that may or may not make sense.

### Resolution (the Z labels)
Once an issue is closed, it ought to have a resolution tag.

- `Z-updated-rfc` -- text was added to the RFC resolving this question
- `Z-open-question` -- text was added to the RFC making this an "open question" to be resolved before stabilization
- `Z-alternative` -- text was added to the RFC describing this as an alternative but not endorsing it
- `Z-no-go` -- nothing was added to the RFC. This is out of scope or just something we decided against.
- `Z-duplicate` -- a duplicate of some other issue
