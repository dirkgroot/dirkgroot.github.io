---
title: "Robot Worlds 3: Client"
date: 2022-06-21T20:40:34Z
draft: false
series: [Robot Worlds]
tags: [walking-skeleton, robot-worlds, kotlin, tdd, refactoring]
---
{{< summary >}}
Now that we have a server that can receive a request over TCP, handle it and send a response back, it looks like good
time to start working on the client. We'll also think about how we want to test our walking skeleton end-to-end.
{{< /summary >}}

<!--more-->

## Progress

Let's grab the crude sequence diagram we drew in [part 1](../01-walking-skeleton) of this series and mark the parts we
already covered in the two previous posts with a green color.

{{< figure src="launch-sequence.svg" class="dg-figure-border dg-figure-padding" >}}

As you can see, our walking skeleton is almost an end-to-end skeleton. The only things left to do are to create a
minimal client application, and write a `main` method for our server, so it can be started as a standalone program.

## Decisions

Before we start coding, I'd like to make some decisions.

### How will the player interact with the client?

The specification doesn't require any particular way of presenting the game to the player. As I'm the only stakeholder
for this little project, and I happen to like ASCII-art based roguelike games, and I don't feel like programming with
Swing, or JavaFX or some other UI/graphics library, we'll create a text-based user interface for the client. The player
will control the his/her robot with the keyboard.

There, that was easy. Let's think about testing.

### Testing end-to-end

Up until now, we've created small tests, focused on small parts of the code. However, because we didn't use _test
doubles_[^1], every step up the call stack resulted in tests which covered more code. Now our goal with our walking
skeleton is to get to a point where we can perform an end-to-end test of the tiny slice of functionality we created.
That way we extend our test coverage to the deployment of our system and the integration of the various standalone
components of the system[^2].

With an end-to-end test, rather than calling specific parts of the application code directly from our test code, we want
to execute the test against the actual program. In our case, an end-to-end test scenario could look something like
this:

1. Install and configure the server, and start it up.
2. Install and configure the client, and start it up.
3. Perform a small, well chosen, number of scenarios to make sure that all components of the system are working
   together nicely.

If this were a "real" project, I would set up automated end-to-end tests as part of the walking skeleton, using test
frameworks like Cypress or Robot Framework (in the case of web applications). For this little hobby project, I don't
mind having to execute end-to-end tests manually.

I will, however, keep on using the [Humble Object](../02-communication#humble-object) pattern to separate code that
performs keyboard and screen I/O from code that can be automatically tested with JUnit tests.

## Let's get to it

### Humble socket client

The server has a humble `SocketListener`, which accepts incoming connections, passes requests to `MessageReceiver` and
sends the response back to the client. Let's mirror this in our client by creating a humble `SocketClient`.

So, I guess the test for our `SocketClient` should look something like our test for `MessageReceiver`. We need to set up
a `SocketListener` for this, so our `client` module needs a test dependency on the `server` module. Let's do that first.

```kotlin
dependencies {
    testImplementation(kotlin("test"))
    testImplementation("org.jetbrains.kotlin:kotlin-reflect:1.7.0")
    testImplementation("com.willowtreeapps.assertk:assertk:0.25")
    testImplementation(project(":server"))
}
```

Now, let's write a test.

```kotlin
@Test
fun `invoke launch command with JSON message`() {
    val socketListener = SocketListener(MessageReceiver(World()))
    val socketClient = SocketClient(socketListener.port)
    val result = socketClient.send("""{ "command": "launch" }""")
    assertThat(result).isEqualTo("""{"result":"OK"}""")
}
```

Make it compile by creating the `SocketClient` class with a stubbed `send` method.

```kotlin
class SocketClient(port: Int) {
    fun send(message: String): String {
        return ""
    }
}
```

Now the test compiles, and it fails: `expected:<"[{"result":"OK"}]"> but was:<"[]">`. Let's make it pass. We'll need to
upgrade the constructor parameter `port` to a private property.

{{< anchor id="socket-client-class" >}}

```kotlin
class SocketClient(private val port: Int) {
    fun send(message: String): String {
        return Socket("127.0.0.1", port).use {
            sendMessage(it, message)
            receiveResponse(it)
        }
    }

    private fun sendMessage(socket: Socket, message: String) {
        socket.getOutputStream().writer().apply {
            write("$message\n")
            flush()
        }
    }

    private fun receiveResponse(socket: Socket) =
        socket.getInputStream().bufferedReader().readLine()
}
```

That's quite a chunk of code to write to make one single test pass. Truth is, I've committed a horrible sin by
copy/pasted and adjusted some code from the `SocketListenerTest` class we wrote in
the [previous post](../02-communication#are-you-being-served). This means we've now got some duplication to eliminate.
Also, I don't like the way we start a `SocketListener` in our test. Let's address the last issue first.

### Decoupling

To start the `SocketListener` from our test, we need to set up all the internals of the `server`.

```kotlin
val socketListener = SocketListener(MessageReceiver(World()))
```

We've introduced coupling[^3] between the `client` module's test code and the internal structure of the `server`. This
means that if we change the internal structure of the server, we **must** change this code as well. That's not good. We
prefer our tests to be sensitive to the behaviour of the system, not to the structure of the system[^4].

Let's refactor step by step. First, we'll make a bit more explicit that the test needs to start the server, and needs to
know what port the server is running on. We'll introduce a `port` variable.

```kotlin
val socketListener = SocketListener(MessageReceiver(World()))
val port = socketListener.port
val socketClient = SocketClient(port)
val result = socketClient.send("""{ "command": "launch" }""")
assertThat(result).isEqualTo("""{"result":"OK"}""")
```

Then inline the `socketListener` variable.

```kotlin
val port = SocketListener(MessageReceiver(World())).port
val socketClient = SocketClient(port)
val result = socketClient.send("""{ "command": "launch" }""")
assertThat(result).isEqualTo("""{"result":"OK"}""")
```

Now we can remove the knowledge about the `server` module's internals from the test method by extracting a method.

```kotlin
@Test
fun `invoke launch command with JSON message`() {
    val port = startServerApplication()
    val socketClient = SocketClient(port)
    val result = socketClient.send("""{ "command": "launch" }""")
    assertThat(result).isEqualTo("""{"result":"OK"}""")
}

private fun startServerApplication() = SocketListener(MessageReceiver(World())).port
```

Finally, let's move the newly created `startServerApplication` function to the `server` module. I've put it in a new
source file called `ServerApplication.kt` in the root package of the `server` module.

```kotlin
package nl.dirkgroot.robotworlds

fun startServerApplication() = SocketListener(MessageReceiver(World())).port
```

There, we've encapsulated all knowledge about the internal structure of the `server` module into
the `startServerApplication` function. By moving said function to the `server` module, we completely decoupled the test
code in the `client` module from the internal structure of the `server` module.

### Duplication

Let's look at the duplication we introduced. Look back at the [`SocketClient`](#socket-client-class) class earlier in
this post, and compare it to what we have in `SocketListenerTest`.

```kotlin
@Test
fun `handles a command`() {
    val socketListener = SocketListener(MessageReceiver(World()))
    val port = socketListener.port

    assertTimeoutPreemptively(Duration.ofSeconds(1)) {
        Socket("127.0.0.1", port).use { socket ->
            sendLaunchCommand(socket)
            val response = receiveResponse(socket)

            assertThat(response).isEqualTo("""{"result":"OK"}""")
        }
    }
}

private fun sendLaunchCommand(socket: Socket) {
    socket.getOutputStream().writer().apply {
        write("""{ "command": "launch" }""" + "\n")
        flush()
    }
}

private fun receiveResponse(socket: Socket) =
    socket.getInputStream().bufferedReader().readLine()
```

Now this is not a 100% duplicate, but I think it's fairly obvious that both classes create a socket connection, send a
message and receive a response. If only we could use `SocketClient` in `SocketListenerTest`...

### Simplification

I'm tempted to introduce a new module, named `tcp-communication`, move `SocketClient` to that module, and use this new
module from the `client` and `server` module. But wait, why did we create separate modules for the server and the
client? I don't remember it being a very conscious decision. I guess I did it because I drew 2 separate containers in
the [container diagram](../01-walking-skeleton#what-is-robot-worlds). But now I think about it, having two separate
containers just means that there are two separate processes. This doesn't necessarily mean that they also need to be
separate executables. We could just as well create one executable which can be started either in client or in server
mode using a command-line switch.

Let's simplify the module structure, so we have just one module: `game`. I'll do it step by step, like this:

1. Move all code from `client/src/main/kotlin` to `server/src/main/kotlin`
2. Move all code from `client/src/test/kotlin` to `server/src/test/kotlin`
3. Remove the `client` module
4. Rename the `server` to `game`

Now there's one thing I notice. In the `client` module I created a package called `nl.dirkgroot.robotworlds.client`, but
in the `server` module I put everything in the `nl.dirkgroot.robotworlds` package. Let's move the server code to
`nl.dirkgroot.robotworlds.server`. All tests still pass, so everything went well.

{{< figure src="game-module-structure.png" class="dg-figure-border" >}}

### Deduplication

Now that we have everything in one module, let's see what we can do about the duplication we created. We'll change
the `handles a command` test in `SocketListenerTest` so it uses `SocketClient`.

```kotlin
@Test
fun `handles a command`() {
    val socketListener = SocketListener(MessageReceiver(World()))
    val port = socketListener.port

    assertTimeoutPreemptively(Duration.ofSeconds(1)) {
        val response = SocketClient(port).send("""{ "command": "launch" }""")
        assertThat(response).isEqualTo("""{"result":"OK"}""")
    }
}
```

The test passes, so we can delete the `sendLaunchCommand` and `receiveResponse` methods from `SocketListenerTest`, as
those are not used anymore. Hmm, now there's still duplication left. This test is basically the same as the test we
wrote for `SocketClient`.

```kotlin
@Test
fun `invoke launch command with JSON message`() {
    val port = startServerApplication()
    val socketClient = SocketClient(port)
    val result = socketClient.send("""{ "command": "launch" }""")
    assertThat(result).isEqualTo("""{"result":"OK"}""")
}
```

The only differences are in how the socket listener is started and whether or not the test is timing out if no response
is received for more than 1 second. One of these tests can be deleted, for sure. There's no point in having two tests
test exactly the same thing.

I like the one with the timeout better, because it's a little more robust than the one without, so let's keep that one.
This means that we can delete the `SocketClientTest` class entirely, because it only contains the test we're going to
delete.

This leaves us with a bit of an awkward situation, because now `SocketListenerTest` is responsible for testing
both `SocketClient` and `SocketListener`. I don't think this is necessarily a bad thing. After all, to me, the "unit" in
unit testing is primarily a unit of behaviour - communication between client and server in this case - not a unit of
structure (a particular class or function). The awkwardness is in the names of the test class and the tests.

So let's just change the name of the test class to `ClientServerCommunicationTest`, and move it from the
`nl.dirkgroot.robotworlds.server` package to `nl.dirkgroot.robotworlds`. Also, let's change the initialisation part of
the two tests to use the `startServerApplication` function, and rename `handles a command`
to `client can send a command to server over a TCP socket`.

```kotlin
@Test
fun `listens on free TCP port when no port is given`() {
    val port = startServerApplication()

    Socket("127.0.0.1", port).use {
        assertThat(it.isConnected).isTrue()
    }
}

@Test
fun `client can send a command to server over a TCP socket`() {
    val port = startServerApplication()

    assertTimeoutPreemptively(Duration.ofSeconds(1)) {
        val response = SocketClient(port).send("""{ "command": "launch" }""")
        assertThat(response).isEqualTo("""{"result":"OK"}""")
    }
}
```

I think I like where this is going. Let's do a retrospective.

## Retrospective

### Horrible sin

I committed a horrible sin by copy/pasting code, but in my defense, we were in the _red_ stage of the red/green/refactor
cycle. When we're in the red stage, our goal is to get to _green_ as fast as possible, by any means necessary. And
that's what we did. In the _refactor_ stage, we cleaned up our mess.

Working like this creates a very clear distinction between getting code to work and designing our code. Furthermore, as
you can see from this little project so far, most of the design work is done _after_ we get some code to work. We did
very little design up front, and we did it just to get an idea of where we're heading.

Why does TDD work like this? Why do we refactor afterwards, instead of designing it "the right way" up front? In my
experience, no matter how much design you do up front, refactoring afterwards will always be necessary to keep the code
clean and well-factored. We just cannot foresee and account for every design issue we will face. When we're writing the
code, and when we're mindful of what we're creating, that's when the most valuable design insights occur, because the
code is right in front of us, and not in some fantasy in our minds.

### Integration

By eliminating the duplication we introduced, we ended up with a test class which tests the communication between the
client and the server. We started out by testing the `SocketListener` by creating some stub client code, because that's
all we could do at that point. When we created the `SocketClient` class, we didn't need the stub code anymore and
replaced it with the actual client code. The result is what I would call an integration test.

I've seen code bases where "integration tests" consisted of client code being tested by stubbing or mocking the server
in some way, and of server code being tested by stubbing the client. To me, this is not integration testing. If we want
to test whether two components integrate well, the best possible way to do that is by letting them actually "talk" to
each other. As we've seen, this doesn't just result in testing the actual integration between components. It also
results in the elimination of duplication and all the risks that come with it.

Let's see if we can finish our walking skeleton next time, so we can start building some real end-to-end features.
Thanks for reading, see you next time!

PS: Here's the source code so far: https://github.com/dirkgroot/robot-worlds

[^1]: Or mocks, as most people call them. In reality, a mock is only one of many kinds of test doubles. Take a look at
the [Test Double patterns](http://xunitpatterns.com/Test%20Double%20Patterns.html) on <http://xunitpatterns.com> if you
want to know more about the terminology around test doubles.
[^2]: There's a lot more to say about end-to-end testing and testing strategy in general, but that's beyond the scope of
this series.
[^3]: Watch Kent Beck [explain what coupling is](https://youtu.be/3gib0hKYjB0?t=865).
[^4]: Again, watch Kent
Beck: [Test Desiderata 2/12 Tests Should be Structure-Insensitive](https://youtu.be/3gib0hKYjB0?t=867).
