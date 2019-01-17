# Salsa Design RFCs

Salsa Design RFCs reflect "in-progress" work on Salsa. The process is quite
a bit different from Rust RFCs. An open Salsa RFC Pr is **not** a "proposed design",
but rather a "design in progress". Furthermore, conversation about the design doesn't
happen on this repository, but rather in the [Salsa Zulip].

## RFC Process

The process goes like this:

- When we want to do something, we discuss on the [Salsa Zulip].
- If there is rough consensus among Salsa contributors to go forward, somebody writes up 
  a design RFC and opens a PR in this repository.
- The RFC thread will be locked; discussion is meant to happen on Zulip.
  - For each RFC, we make a Zulip stream.
- The RFC stays open until it is fully implemented:
  - We adjust the design until we are done.
  - Once the design is fully implemented, we merge the RFC PR.
- If we decide to abandon the design, we close the PR without merging.

The active PRs on the RFCs repo therefore show the things we are actively implementing.

## Making an RFC

RFCs should have be files with names like `RFCNNNN-Foo-Bar.md`. You can start with [the `template.md` file](template.md).

The RFC should include a link to a stream on the [Salsa zulip].

[Salsa Zulip]: https://salsa.zulipchat.com/




