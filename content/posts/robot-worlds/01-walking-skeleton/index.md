---
title: "Robot Worlds 1: Walking Skeleton"
date: 2022-06-14T17:44:08Z
draft: true
tags: [walking-skeleton, robot-worlds, kotlin, tdd]
---
{{< summary >}}
Let's make a game! I have a big graveyard of unfinished hobby projects, so I
have no idea how far I'll get. But as long as learning ensues, I'm fine with
that.
{{< /summary >}}

<!--more-->

## Intro

I created this blog site 6 months ago, so I guess it's about time to actually
start writing 🙄. Here goes nothing!

[Geepaw Hill](https://www.geepawhill.org/series/robot-worlds/) and
[Ron Jeffries](https://ronjeffries.com/categories/robot/) both started a
(v|b)log series on a fun programming exercise called Robot Worlds. From what I
understand from these blogs, this exercise is part of an education offered
by [WeThinkCode_](https://www.wethinkcode.co.za/), which helps young South
Africans train their digital skills.

I really like doing these kinds of exercises, mainly because they are an
excellent way of practicing my test-driven development skills, and a nice
programming challenge in general. So let's give it a go, shall we?

You can view the source code on
GitHub: <https://github.com/dirkgroot/robot-worlds>.

## What is Robot Worlds?

Robot Worlds is a multiplayer game in which the player controls a robot in a
world full of obstacles, pits, and other robots. Robots can move around, scan
their surroundings, shoot, and place mines. It consists of a client and a
server, which communicate over a TCP socket connection, using messages in JSON
format. Check out
[Hill's GitHub repository](https://github.com/GeePawHill/robot-worlds/tree/main/notes)
for the specifications.

The _container diagram_[^1] below shows my current understanding of the
architecture of the game.

{{< figure src="container.svg" class="figure-border" >}}

## Walking skeleton

Both Hill and Jeffries start out by building what's called a "walking skeleton",
and I'll be doing that as well. The term "walking skeleton" was coined by
[Alistair Cockburn](https://twitter.com/TotherAlistair/), and the
[GOOS book](https://www.amazon.com/Growing-Object-Oriented-Software-Guided-Tests/dp/0321503627)
describes it as follows:

> A “walking skeleton” is an implementation of the thinnest possible slice of
> real functionality that we can automatically build, deploy, and test
> end-to-end. It should include just enough of the automation, the
> major components, and communication mechanisms to allow us to start working on
> the first feature.

Hill and Jeffries have also given some valuable insights about what a walking
skeleton is, and why it's so useful. Check out their articles, they are really
good!

My plan is to follow the definition of the GOOS book rather closely. This means
I'll create a client and a server application with just enough functionality to
justify their existence and the need for communication between the client and
the server.

## Let's get started!

### Project setup

First things first, let's set up a basic project. I'll be using Kotlin as my
programming language, and IntelliJ IDEA as my IDE. I've set up a multi-module
Gradle project with 2 modules to start with: `client` and `server`:

```text
robot-worlds
|...
|- client
|  |- src/main/kotlin
|  |- src/test/kotlin
|  |- build.gradle.kts
|...
|- server
|  |- src/main/kotlin
|  |- src/test/kotlin
|  |- build.gradle.kts
|...
|- settings.gradle.kts
```

### Launch!

So, what's the "thinnest possible slice of real functionality" we could
implement in our walking skeleton? Launching a robot into the world seems like a
good candidate. To launch a robot into the world, the client must send
a `launch` command to the server with some arguments, describing the robot it
wants to launch. If all is well, the server then responds with a message
containing an `OK` result and the current state of the robot (position,
orientation, etc).

I'd like to make this slice even thinner. Let's start by implementing a very
minimal version of the `launch` command and response. We're leaving out the
arguments from the command, and the state from the response. Furthermore, we'll
only implement the happy flow.

This is how the first version of the `launch` command will look:

```json
{
  "command": "launch"
}
```

The response will look like this:

```json
{
  "result": "OK"
}
```

### So, where to begin?

Although this is indeed a very thin slice of real functionality, it's still
functionality that requires all major parts of the application to be in place.
That way we can test it end-to-end. We need a client and a server, both need
some kind of user interface, and the client needs to send commands to the server
over a TCP socket.

Let's start from the bottom of the call stack, and step by step work our way
upwards. What do I mean by this? This sequence diagram is a rough sketch of the
call stack I currently have in mind.

{{< figure src="launch-sequence.svg" class="figure-border" >}}

Just rotate your screen 90 degrees clockwise, and you'll see what I mean. The
top of the call stack is where everything starts: the `Player`. With every arrow
we dive a level deeper into the call stack, until we arrive at `World`. So the
bottom of the call stack in this case is `World`.

### World-class

Let's start nice and simple. First, write a test.

```kotlin
class WorldTest {
    @Test
    fun `no robots in a new world`() {
        val world = World()
        assertThat(world.robotCount).isZero()
    }
}
```

Making it pass is easy.

```kotlin
class World {
    val robotCount: Int = 0
}
```

For our slice, we need a way to launch a robot into the world.

```kotlin
@Test
fun `launch a robot into the world`() {
    val world = World()
    world.launchRobot()
    assertThat(world.robotCount).isEqualTo(1)
}
```

We don't have a `launchRobot` function yet, so make it compile first.

```kotlin
class World {
    val robotCount: Int = 0

    fun launchRobot() {
    }
}
```

Our test fails, of course: `expected:<[1]> but was:<[0]>`. Let's make it pass.
We'll make the `robotCount` a `var` instead of a `val`, to be able to update
it's value.

```kotlin
var robotCount: Int = 0

fun launchRobot() {
    robotCount++
}
```

Now the `robotCount` property can be updated by everyone. Let's make the setter
private, so only `World` can update it.

```kotlin
var robotCount: Int = 0
    private set
```

Our tests still pass, so I guess the head of our walking skeleton is done 💀. It
doesn't have much of a brain yet, but with the amount of intelligence it
currently has, I'd call it "undead".

### Request to launch

Looking at the protocol specification, we can see that every interaction between
client and server is initiated by the client. The communication style is
[request-response](https://en.wikipedia.org/wiki/Request–response). The client
requests the server to execute a command, and the server replies with a result.

So, if we move one step up the call stack from `World::launch`, what would we
need? I think it's a function that takes a request and executes the requested
command. Let's implement this without worrying about (de)serializing from/to
JSON yet. We'll start by adding a test to `WorldTest`.

```kotlin
@Test
fun `execute a launch command`() {
    val world = World()
    world.handleRequest(Request(command = "launch"))
    assertThat(world.robotCount).isEqualTo(1)
}
```

This doesn't compile, because we don't have a `Request` class, and `World`
doesn't have a `handleRequest` method. Let's start by letting IntelliJ generate
a `Request` class for us (I'm lazy).

```kotlin
class Request(command: String)
```

Easy. Now, IntelliJ can generate the `World::handleRequest` method as well.

```kotlin
fun handleRequest(request: Request) {
}
```

Now our test compiles, but it fails: `expected:<[1]> but was:<[0]>`, so let's
make it pass.

```kotlin
fun handleRequest(request: Request) {
    launchRobot()
}
```

Yep, that's all we need for now.

### Really?

Remember the [description from the GOOS book](#walking-skeleton) earlier in this
post. Our goal here is not to build an entire feature:

> [...] It should include just enough of the automation, the major components,
> and communication mechanisms to allow us to start working on the first
> feature.

We just want to get all the stuff in place that we need to build and test our
first feature end-to-end. In a real-world project this would typically include
setting up build scripts, CI/CD, deployment
to [DTAP](https://en.wikipedia.org/wiki/Development,_testing,_acceptance_and_production)
environments, etc. By doing this first, we benefit from having all this from the
very start. Immediately, we have everything in place to make sure that every bit
of functionality we add is well-tested, well-factored and potentially shippable.

## Summary

We started out by drawing a little architecture diagram to get an overview of
the major parts of the game that we're building. Later on, the sequence diagram
helped us choose a starting point for writing some actual code. I feel like that
was just enough "design up front" to get me started.

We got some of the boring project set up stuff out of the way. The project is
set up and a tiny part of a tiny slice of functionality is in place. The `World`
class will almost certainly be split up into the actual game logic and one or
more other classes which handle requests from the client. We could have chosen
to do that right now, but we can also do it later. I don't think it's really
important right now.

For now, our goal is to put the major components in place, and `World` is one of
them. The next steps up the call stack will most likely be the TCP socket
connection and (de)serialization of messages.

So there it is, 6 months after setting it up, this blog has finally officially
been kicked off. I hope you enjoyed my first post. I certainly enjoyed writing
it. Stay tuned for the next one!

[^1]: The container diagram is one of the 4 core diagrams of the
[C4 model](https://c4model.com) for visualising software architecture.
