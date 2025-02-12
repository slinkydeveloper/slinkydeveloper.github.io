---
title: Rust macros are not just about DRY
description: "Or how to win the \"I think we should use a macro here\" argument."
date: 2025-02-05 15:10:02
tags: [rust, macros]
---

In the last weeks I've been re-working some internals of [Restate](https://restate.dev/) that enable the SDKs used by our users to interact with the Restate server. At a glance, the Restate server talks with the SDKs using a protocol on top of HTTP that transports few Protobuf messages, prefixed by a header containing message type, length and flags. Easy enough.

If you've ever found yourself implementing something similar, you know there's a lot of boring `match`es involved in the process: convert from the message type in the code to some `MessageType` enum, then from `MessageType` enum to the `prost::Message` implementation, and so on and so forth. And often you also have some additional intrinsic properties of the Message themselves that you need to represent in the `MessageType` enum, leading to again more `match`es.

Sounds like a perfect case for a nice shiny macro right?

## The argument against macros

Let's face it, unless you're a macro magician, macros aren't super approachable to develop and maintain for all Rust developers. And I think this is true both for the world of procedural macros and for `macro_rules!`. 

In order to develop a procedural macro, you need to dig deep into the Rust syntax tree data model, requiring a lot of code to handle all the different cases correctly, especially when adding generics to the mix. On top, it requires to set up a different crate, perhaps leading you to other issues that you didn't need to deal with in the first place. If you've never developed a Rust procedural macro, I suggest to look at [`thiserror`](https://github.com/dtolnay/thiserror/tree/master/impl), it's one of the best procedural macro implementations I've read so far.

When it comes to `macro_rules!`, well well well... Substitutions can be challenging to understand, both from a syntax perspective but also because of the matching/expansion rules Rust applies, that sometimes are sort of unexpected. There's a reason why the Rust Macro book has a section only about [Minutiae](https://veykril.github.io/tlborm/decl-macros/minutiae.html), which you definitely encounter and exploit too.

At the end of the day, procedural macros and `macro_rules!` are respectively a framework and a separate language within Rust, and as such it needs people that know them in order to mantain them.

Because of that, it is fair that some people refrain from creating macros, especially when they don't bring enough value. You probably heard or read somewhere arguments like "if you need 100 lines of `macro_rules` to generate 120 lines of code, then it's not worth it". I think there's more to macros though, than just ratio between macro LOCs vs generated LOCs.

## Macros are not just about DRY

Macros have way more benefits than just DRY, which I can sum up as follows: macros are an excellent tool to develop a DSL evaluated at compile time!

Let's go back to our protocol: we need to describe the message types with their various intrinsic properties, and whether we achieve that through `if`s, `match`es, a bunch of traits with dynamic dispatch, that's just an implementation detail. This is exactly what I would use a DSL for!

By skimming through the [protocol message macro](https://github.com/restatedev/restate/blob/faa2376b5c0f8eb46aeff500d04107ee212d91da/crates/service-protocol-v4/src/message_codec/mod.rs) here:

```rust
// Syntax: <Message type> <Message kind> <flags>* = <code>,
gen_message!(
    Start Control = 0x0000,
    Call Command allows_ack = 0x040D,
    Call CompletionNotification noparse = 0x800D,
    Signal Notification noparse = 0xFBFF,
    // [...]
);
```

You can quite easily deduct for a given message type what is the message kind, the message code, the flags, and so on.
With the help of this macro, I can define with a simple DSL my data model with all its properties and the macro generates for me the code implementing it, including the `MessageType` enum, methods like `MessageType::code()`, `MessageType::requires_ack()`, encoding/decoding of messages, and so on. Even better, I can now change and/or expand the generated code, without needing to go through the `match` statements one by one fixing the individual bits.

At the end of the day, macros are an awesome tool to define DSLs to describe your data model at compile time!

We have other good examples of macros like this in Restate, for example the `define_table!` macro, that we use to define our SQL tables in Datafusion:

```rust
define_table!(sys_journal(
    /// [Invocation ID](/operate/invocation#invocation-identifier).
    id: DataType::LargeUtf8,
    /// The index of this journal entry.
    index: DataType::UInt32,
    /// The entry type. You can check all the available entry types in [`entries.rs`](https://github.com/restatedev/restate/blob/main/crates/types/src/journal/entries.rs).
    entry_type: DataType::LargeUtf8
    
    // [...]
));
```

This macro generates for us few data structures, including the `Schema` used by DataFusion, and the `RowBuilder` to fill the DataFusion rows during a table scan.

What's even more interesting in this macro is that we reuse the `///` doc comments by copying them in a static string (compiled behind a feature gate) and we read them later with a tool to generate our SQL documentation.

## Use with moderation

I hope I gave you some good examples of how to use Rust macros. As every "advanced" feature in every programming language, you should ponder pragmatically whether your problem can use a macro or not.

Few tips I can give are:

* Refrain from overcomplicated macro input syntaxes when you can and rather try to stick to familiar things such as assignment like syntax or struct/enum definition like syntax.
* Document the macro input syntax and the generated output (**of course!**), no matter if it's an internal macro or something you expose in a Rust crate you publish/maintain.
* Before delving into proper `macro_rules!` implementations, read the [Rust macro book](https://veykril.github.io/tlborm/decl-macros/macros-methodical.html), and generally stick to the patterns described in the book. Most of the `macro_rules!` I developed so far are a combination of push down accumulators and TT munchers.
* If you use RustRover/Intellij for Rust, crank up max heap before touching a macro code!

I hope you liked this post, I plan to do more of that in the future, so stay tuned for updates!
