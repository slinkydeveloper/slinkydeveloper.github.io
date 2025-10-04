---
title: Designing APIs for vibe coding
description: "Lessons learnt "
date: 2025-10-03
tags: [devex, vibe coding, typescript]
---

Vibe coding is a point of no return. What I like to think as the _fine art_ of designing APIs and crafting the DevEx of a library, a developer tool, needs to adapt to this new reality.

But how can we do so?

## Before vibe coding

I've been doing API design for quite some time now, nitpicking on this parameter name, that function overload, and so on.
Over the years, I learnt from my mentors a few principles that I somewhat consider **Developer Experience canon**.

First, your API needs to be **idiomatic** for the target audience.
Being idiomatic means many things, from more obvious aspects like adopting the appropriate naming convention, to more elaborate or even unwritten rules:
if factory methods in a certain language respect a certain pattern in commonly used libraries, you need to have really good reasons to design it differently.

Designing for **integrability** is another fundamental aspect. Your API rarely lives in isolation, but it's always used together with other APIs:
when you have a function returning a type you can iterate over, better make sure you implement whatever is the standard library type for an iterator.

And last, the aspect I call **discoverability**: the ability for a user to discover a certain API or even functionality exists.
People discover APIs in different ways: skimming through documentation, looking at some examples, or downloading a template project and `CTRL` + `Space` their way through.
It's up to the API designer to optimize for one or more of those funnels.

To get concrete, let's look at the design choices behind the `Context` API from the Restate TypeScript SDK:

```typescript
import * as restate from "@restatedev/restate-sdk";

async function myFn(ctx: restate.Context, input: Input): Promise<Output> {
  await ctx.run(mySideEffect);
}
```

The first problem: **people don't know what Restate can do for them**, unless they read through all the documentation and all the examples.
So let's put all the important features in a single type, and let's call that `Context`, a common name used for such APIs.
The user can just `CTRL` + `SPACE` and here they are, all the features we want users to learn about:

![Vs code autocompletion for Context](/img/vscode-autocompletion.png "Vs code autocompletion for Context")

We got `run` which lets users record a side effect, `sleep` which lets users sleep their function, `serviceClient` which lets users create clients for other services, and the list goes on.

And we go further: when you define a restate function that you want to serve, using `restate.serve({...options})`, you need to provide the `Context`, otherwise you'll get a typing error.
Restate functions don't strictly require the `Context` parameter, but we nevertheless make it mandatory in the type system to make sure the user **knows** about the `Context`.

This design approach works well for us, but we could have taken some alternative approaches.

For example, we could have encapsulated all the features as individual functions you import, and get rid of the `Context`:

```typescript
import * as restate from "@restatedev/restate-sdk";

async function myFn(input: Input): Promise<Output> {
  await restate.run(mySideEffect);
}
```

This looks probably nicer on an example, but it would not be that easily discoverable by autocompletion: `CTRL` + `Space` would give you here a lot of results, including functions that you shouldn't use inside a restate function, like `serve`!
How could we solve that? We would probably need another import, like `import * as restateFeature from "@restatedev/restate-sdk/features";`, but users would have to know this import, and good luck giving a good name to this namespace!

Of course, every design has its set of tradeoffs. In our `Context` design approach, the obvious drawback is that you have to pass it through your functions, as opposed to the design with namespaces. There's no free lunch!

## What changed with vibe coding

Now why am I telling you all of this?

Let's take the example of `ctx.serviceClient`:

![Vs code autocompletion for service client](/img/vscode-autocompletion-2.png "Vs code autocompletion for service client")

This API is another example of optimizing for auto-completion: you pass the service you want to call to `ctx.serviceClient`, and some TypeScript type manipulation plus `Proxy` gives the user a typed client with autocompletion.

Let's do a test now with Copilot, using GPT as a model, prompting:

```
In my function, can you call anotherFn using the context?
```

![First prompt result](/img/prompt-first-result.png "First prompt result")

Let's try another time:

![Second prompt result](/img/prompt-second-result.png "Second prompt result")

Let's try another prompt:

```
In my function, can you send a request to anotherFn using the context?
```

![Fourth prompt result](/img/prompt-fourth-result.png "Fourth prompt result")

Changing model to Claude Sonnet 4, after a few attempts, we get the right result:

![Third prompt result](/img/prompt-third-result.png "Third prompt result")

Before the current generation of models, Copilot would give me even more interesting results, returning some `fetch` like API, something like:

```typescript
ctx.request({
  service: "anotherService",
  handler: "anotherFn",
  body: "my input"
})
```

Of course, LLMs get better over time, at some point they get trained/or get feedback over your API docs/examples, and at the end of the day they're probabilistic models, so they won't be always right.

Yet you want to give a good experience to your users, so here it is a new principle to design APIs for you: maximize the **"LLMs hit rate"**.

This API, while being perfectly fine for auto-completion, it's perhaps not the best choice for LLMs hit rate, as when you say "request" to a LLM in the context of Typescript, `fetch` or similar is what it



In my opinion, idiomatic and easy to integrate APIs are still valid design goals.
Perhaps those are even more important than ever, as these days software development consists essentially in connecting other people APIs in a way that makes sense for your business.

Who reads docs anymore? Only LLMs do!!!
