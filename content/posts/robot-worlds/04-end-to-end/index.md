---
title: "Robot Worlds 4: End to end"
date: 2022-06-25T16:20:22Z
draft: false
series: [Robot Worlds]
tags: [walking-skeleton, robot-worlds, kotlin, tdd]
---
{{< summary >}}
I feel like we're almost done with our walking skeleton. Today, I'll try to finish it.
{{< /summary >}}

<!--more-->

## Recap

### Design so far

It's been a few days since I wrote the last article, so let's take a moment to review our design so far. Currently, the
game consists of two "chunks", or components: the `client` and the `server`.

{{< figure src="class-diagram.svg" class="dg-figure-border dg-figure-padding" >}}

Step by step, we've been working our way up the call stack, starting with `World`. In the [previous post](../03-client),
we created `SocketClient`, which means we're now able to send a JSON string to the server over TCP, and get a JSON
string back as a reply.

### Terminology

Before we move on, let's look at the terminology we're using. I'm seeing an inconsistency I'd like to address. Making a
little diagram like this often helps to make these kinds of inconsistencies obvious. From this diagram, it's quite
obvious that we're using four words, without making it obvious how these words are related to each other: _message_,
_request_, _command_ and _result_. I'd like to settle on one word.

All of these words are actually used in the protocol specification. In my words, the spec says that the client sends a
_request_ _message_ containing a _command_, and the server replies with a response _message_, containing a _result_.
This structure is not visible in our design. Let's try to make this more obvious.

Let's rename `Request`  to `RequestMesage`, and `CommandResult` to `ResponseMessage`, and create a little diagram again.

{{< figure src="class-diagram-rename.svg" class="dg-figure-border dg-figure-padding" >}}

I think this is a lot better. The concepts from the specification are now all present in our design, and the structure
also reflects the structure in the specs.

Now there's one more thing: I don't think the `Receiver` part of `MessageReceiver` is entirely accurate. It does more
than just receiving a message. I think it's more accurate to say that it _handles_ messages (if you have better ideas,
let me know, naming is hard). So I think `MessageHandler` is a more accurate name. Let's stick with that for now.

## Finishing the client

### Hello, friend[^1]

We have a `SocketClient` that sends newline-terminated strings to the server. I think that's actually enough to start
working on a `Robot`. Let's do that, and see what we'll end up with.

```kotlin
class RobotTest {
    @Test
    fun `initially not launched`() {
        val robot = Robot()
        assertThat(robot.launched).isFalse()
    }
}
```

Make it compile by creating the `Robot` class with a `launch` property.

```kotlin
class Robot {
    val launched: Boolean = false
}
```

Our test immediately passes. Now, let's launch the robot!

```kotlin
@Test
fun `launch the robot`() {
    val port = startServerApplication()
    val client = SocketClient(port)
    val robot = Robot(client)
    robot.launch()
    assertThat(robot.launched).isTrue()
}
```

Make it compile and watch it fail.

```kotlin
class Robot(client: SocketClient) {
    val launched: Boolean = false

    fun launch() {
    }
}
```

I need to adjust our first `Robot` test as well, because the constructor has changed.

```kotlin
@Test
fun `initially not launched`() {
    val port = startServerApplication()
    val client = SocketClient(port)
    val robot = Robot(client)
    assertThat(robot.launched).isFalse()
}
```

Now let's implement `Robot::launch`.

```kotlin
class Robot(private val socketClient: SocketClient) {
    var launched: Boolean = false

    fun launch() {
        socketClient.send(RequestMessage("launch").toJSON())
        launched = true
    }
}
```

Hmm, `RequestMessage` does not yet have a `toJSON` function. Let's comment the `launch` implementation, `@Ignore` the
test we just created, and add that first.

```kotlin
class RequestMessageTest {
    // ...

    @Test
    fun `serialize to JSON`() {
        val json = RequestMessage("launch").toJSON()
        assertThat(json).isEqualTo("""{"command":"launch"}""")
    }
}
```

We already know how to do this, so let's make it compile en make it pass in one go.

```kotlin
data class RequestMessage(val command: String) {
    fun toJSON(): String {
        return Json.encodeToString(this)
    }
    // ...
}
```

Done. Let's go back to `Robot::launch` again. Enable the test again, uncomment our code and watch it pass. That was
easy! But wait, it looks like we did more than was needed to make the test pass, let's check that. Yes, the test still
passes if we remove the call to `SocketClient`.

```kotlin
fun launch() {
    launched = true
}
```

What test could we add to validate that we actually use the `SocketClient`? We could test that `launch` fails if we
don't start the server application. It should fail because the `Socket` created in `SocketClient::send` will throw a
`java.net.ConnectException` when it doesn't succeed at creating a TCP connection. Let's try that.

```kotlin
@Test
fun `launch when the server is not running`() {
    val client = SocketClient(32323)
    val robot = Robot(client)
    assertThat { robot.launch() }.isFailure()
    assertThat(robot.launched).isFalse()
}
```

This test fails, like expected, because `launch` doesn't even try to connect to the server. Let's fix `launch` by adding
the call to `SocketClient` again.

```kotlin
fun launch() {
    socketClient.send(RequestMessage("launch").toJSON())
    launched = true
}
```

Now all tests pass, and all functionality is covered by a test.

### Are we done?

In the first post in this series, we looked
at [what a walking skeleton actually is](../01-walking-skeleton#walking-skeleton). Let's review:

> A “walking skeleton” is an implementation of the thinnest possible slice of real functionality that we can
> automatically build, deploy, and test end-to-end. It should include just enough of the automation, the major
> components, and communication mechanisms to allow us to start working on the first feature.

Reading this, I think we're almost done with our walking skeleton, but not entirely, for two reasons. First, we can't
perform an end-to-end test yet, because we can't start the client and the server as standalone processes. We need
a `main` function for that. Second, I think we're missing a "major component": The robot's view of the world.

### The world through the robot's eyes

We already have a `World` class in the `server` package. This class represents the world, as seen by the server. It
contains all information of the entire map (obstacles, pits, mines) and knows where all robots are and what they are
doing. However, from the spec we know that the robot has a very limited view of the world. It gathers information by
moving around and scanning its surroundings.

Do I want this to be in the walking skeleton? Do we need it to start working on the first feature? I'm tempted to say
yes, but in all honesty I don't think we need it right now. Sure, `Robot` using `SocketClient` directly is something we
will want to change at some point, but I don't think it's necessary for the tiny slice of functionality we chose for our
walking skeleton. Let's skip this for now, and wait until we have an actual need for it.

### End-to-end

That leaves us with the need for an end-to-end test. To be able to do this, we need a `main` function. However, we need
to be able to launch the client and the server as standalone processes. Let's use command-line arguments for that. If
the first argument is `server`, we'll start a server, if it is `client`, we'll require a second argument containing the
port to connect to, and start a client.

Since this is the outer-most edge of the application, I'm not going to use TDD.

```kotlin
fun main(args: Array<String>) {
    if (args.isEmpty())
        notEnoughArguments()

    when (args[0]) {
        "server" -> runServer()
        "client" -> {
            if (args.size < 2)
                notEnoughArguments()

            runClient(args[1].toInt())
        }
    }
}

private fun notEnoughArguments() {
    System.err.println("Not enough arguments!")
    exitProcess(1)
}

private fun runServer() {
    val port = startServerApplication()
    println("Server running on port $port")
}

private fun runClient(port: Int) {
    println("Connecting to server on port $port...")
    Robot(SocketClient(port)).launch()
    println("Robot launched!")
}
```

Next, let's add the `application` plugin to our Gradle configuration and configure a main class, so we can create a
distribution for our game.

```kotlin
plugins {
    kotlin("jvm") version "1.7.0"
    kotlin("plugin.serialization") version "1.7.0"
    application
}

// ...

application {
    mainClass.set("nl.dirkgroot.robotworlds.RobotWorldsKt")
}
```

Now, we can use `./gradlew assemble` to create our distribution. This creates a `game/build/distributions` directory
containing our distribution in TAR and ZIP format. We also get a `game/buid/install` directory containing an
uncompressed distribution, which can be used immediately.

Based on what we built so far, we expect the server to report the port it's listening on and keep running until the
client connects and launches its robot. After that the server should exit. Let's give it a try, shall we?

{{< figure src="end-to-end.gif" class="dg-figure-border dg-figure-padding" >}}

Success 🎉! On the left you can see that I started the server. On the right, you can see that I started the client using
the port number reported by the server. The client successfully connected to the server, the launch command was handled
successfully and the server stopped after that.

When I start the client while the server isn't running I expect to be greeted with a stack trace, because we don't have
proper error handling in place yet.

```text
bash-5.1$ ./game client 62226
Connecting to server on port 62226...
Exception in thread "main" java.net.ConnectException: Connection refused
    ...
	at nl.dirkgroot.robotworlds.client.SocketClient.send(SocketClient.kt:7)
	at nl.dirkgroot.robotworlds.client.Robot.launch(Robot.kt:9)
	at nl.dirkgroot.robotworlds.RobotWorldsKt.runClient(RobotWorlds.kt:35)
	at nl.dirkgroot.robotworlds.RobotWorldsKt.main(RobotWorlds.kt:18)
```

## Walking skeleton: Done!

That's it! We created a tiny slice of end-to-end functionality to set us up for the development of our first feature.
All code, except for the outer-most edge (the `main` method and its siblings) is covered with automated tests, and we
performed a manual end-to-end test.

For all intents and purposes, I consider the game to be potentially shippable. The only reason we're not shipping is
because the game lacks features and polish (we don't want to see a stack trace in the final product).

I think this is a good time for a retrospective.

## Retrospective

### Ubiquitous language

We started by observing that the terminology we used in the code didn't fully match the terminology which is used in the
spec. We fixed this by renaming two classes. I think it's important to make sure that the terminology used in the code
matches the terminology of the problem domain. This makes it makes it easier to understand code, because we don't need
to translate between different sets of terminology.
The [Domain-Driven Design](https://en.wikipedia.org/wiki/Domain-driven_design) method makes this very explicit by
striving for a "ubiquitous language", a language shared by everyone involved in the project, from end users to software
developers.

### YAGNI (again)

I forgot to apply the YAGNI principle when implementing the `Robot::launch` method. Instead of writing just enough code
to make the test pass, I added a line of code I knew I was going to need. It wasn't a big deal, and it was fixed easily,
but it's still.

I think this little example is a nice showcase of why the YAGNI principle is so powerful. By making sure every line of
production code we add is "justified" by a failing test, we don't just achieve high line and branch coverage. We also
achieve high _behavioural coverage_, as I like to call it, which to me is the primary goal to strive for with TDD, when
it comes to coverage. High line and branch coverage are consequences, not goals in and of themselves.

### Privacy

While glossing over the code we created, I noticed that the setter for `Robot::launched` is public. Let's make it
private for good measure.

```kotlin
class Robot(private val client: SocketClient) {
    var launched: Boolean = false
        private set
    // ...
}
```

Tests still pass ➡️ commit.

Thanks for reading! As always, you can view the source code on GitHub: https://github.com/dirkgroot/robot-worlds

[^1]: I couldn't resist a Mr. Robot reference 🤖.
