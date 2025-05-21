---
title: "Designing with types #01: Introduction"
date: 2025-05-21T06:12:50Z
draft: true
tags: [ design, "type safety", java ]
series: "Designing with Types"
---

{{< summary >}}

Ever had that sinking feeling when a bug sneaks into production, despite all your testing efforts? Yeah, we've all been
there! While languages like Java, C#, and Kotlin come with powerful type systems, many developers aren't using them to
their full potential.

In this series, we'll explore how we can use types to catch bugs before they even have a chance to become bugs. We'll
look at practical ways to make our designs safer and easier to understand.

{{< /summary >}}

<!--more-->

## Introduction

This is the first part of a series of articles I'm writing about designing with types. In the first two articles we'll
set the stage. In subsequent articles we'll explore some practical ways of using types make our code safer and easier to
understand.

## Feedback loops

Before we dive in, let's consider why this is relevant. While developing software, we are constantly faced with feedback
loops. The diagram below shows a simplified version of a typical development process.

{{< figure src="feedback-loops.svg" class="dg-figure-border dg-figure-padding" width="400px" >}}

This diagram shows three types of feedback we could get after we make a change. The build pipeline could fail, a code
reviewer could give valuable feedback, or someone could find a bug while manually testing the change.

From this small example, we can easily see that we're more productive when we have short feedback loops. Solving the
bugs we inevitably introduce takes longer when more steps are between making the change and receiving feedback. If
receiving feedback takes a long time, we'll go do something else in the meantime. This causes context switching which,
as we know, is a big productivity killer.

### Finding bugs

Development processes like the one shown above typically have multiple ways of finding bugs after they have been
introduced. For example:

- Code review
- Automated tests
- Manual / exploratory tests
- Acceptance test
- Collect end user feedback
- etc...

We need to do stuff like this. If you're doing all this, good job!

{{< figure
src="https://media0.giphy.com/media/v1.Y2lkPTc5MGI3NjExNmxsNHRmd2lkODFvbnhtcjBhdzdyaXMwbTcweGd3bzdpZjN5d2htMiZlcD12MV9pbnRlcm5hbF9naWZfYnlfaWQmY3Q9Zw/mGK1g88HZRa2FlKGbz/giphy.gif"
width="200px" >}}

### Preventing bugs

There are also multiple ways to prevent bugs from occurring in the first place, for example:

- Backlog refinement
- Developing in small, safe steps
- Test-Driven Development
- 🎇 **Design** 🎇

**Design** is what this series is about. We'll explore several ways of using carefully designed types to **make
illegal states unrepresentable**[^1]. Code that introduces an illegal state should not compile.

{{< figure
src="https://media0.giphy.com/media/v1.Y2lkPTc5MGI3NjExeHU0bTZianlqZWJ3NjFzNXFvNTI3bHFodXhzZjUwaDFqajZlMjZsdCZlcD12MV9pbnRlcm5hbF9naWZfYnlfaWQmY3Q9Zw/26ufdipQqU2lhNA4g/giphy.gif"
width="200px" >}}

## Design

### Strategy

As shown before, we typically spend a lot of time and effort on finding bugs in our software. This is a good thing.
Mistakes will always be made and cannot be 100% prevented, not even by the techniques we'll be exploring in this series.
However, from what I've seen "in the wild", I do think we tend to miss a lot of opportunities to design our software in
ways that **prevent** bugs from being introduced.

The overall strategy we'll be exploring is to prevent bugs by carefully designing a **domain model**. This domain model
implements business rules and should be considered an **API** that is used by our application logic. The domain model
should consist of carefully designed types that make it impossible to build application logic that introduces illegal
states in our application. Ideally, mistakes in our application logic are found by the compiler.

### Design goals

In short, the design goals for our domain model are to create an API that:

- is **safe** to use
- reveals **intent**

#### Safety

One common cause of bugs is misusing the API of the domain model. Some examples are:

- Assigning invalid values to properties
- Changing a property of an entity while not allowed
- Changing the "workflow" status of an entity while not allowed
- Calling functions while not allowed
- Having functions without proper input validation and calling them with invalid parameter values

All of this can and **will** happen, when the API's we design allow for such mistakes to be made.

So what do I mean when I say we should design APIs that are safe to use? It means we should aim to design API's that can
be used without having to wonder whether they are safe to use in a particular situation. To achieve this, we should
prefer compile time validation over runtime validation. When you use an API in an invalid way, the code should not
compile.

#### Revealing intent

Problems like the ones mentioned above also tend to happen when developers misunderstand the intended purpose of APIs.
An obvious reason for this is lack of proper naming. For example, `completeOrder` reveals a lot more intent than
`setStatus`.

Another, more subtle, naming problem is that we tend to just not name things. We'll be looking at several examples of
this in subsequent articles. As a preparatory exercise, you can look at the code you're working on and see if you can
find concepts that are implemented, but not explicitly named.

As you will see, safety and revealing intent often go hand in hand. When we improve the safety of your code, we will
likely also make the code easier to understand and vice versa.

## Next up

In the next article we'll do some more setting the stage by reviewing some commonly seen code. In subsequent articles
we'll explore ways of improving the safety and readability of our code.

[^1]: This phrase was coined by [Yaron Minsky](https://blog.janestreet.com/effective-ml-revisited/)
