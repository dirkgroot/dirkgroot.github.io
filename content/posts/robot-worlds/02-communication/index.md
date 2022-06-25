---
title: "Robot Worlds 2: Communication"
date: 2022-06-18T13:53:03Z
draft: false
series: [Robot Worlds]
tags: [walking-skeleton, robot-worlds, kotlin, kotlin-serialization, tdd, yagni]
---
{{< summary >}}
Our walking skeleton is no more than an undead skull at the moment. Some more bones need to be added to make it a fully
fledged skeleton. Let's see if we can set up communication between the client and the server.
{{< /summary >}}

<!--more-->

## Recap

So where are we? In the [first post in this series](../01-walking-skeleton), we established a general idea of what the
architecture of the game looks like, and we started creating a walking skeleton. We started at the bottom of the call
stack, and we're working our way up, step by step.

## Communication

Today, I would like to start setting up communication between the client and the server over a TCP socket. For that,
we'll need to serialize and deserialize the messages to/from JSON, and send JSON over a TCP socket. Let's see if we can
invoke the `launch` command by sending it in JSON format to the server. But first...

### Returning a result

Looking at our launch command, I notice that it's not returning a result.

```kotlin
fun handleRequest(request: Request) {
    launchRobot()
}
```

According to the spec, every command must
return a result, so let's fix that first. Remember, for our walking skeleton, we return a very simple result:

```json
{
  "result": "OK"
}
```

So let's first change our test.

```kotlin
@Test
fun `execute a launch command`() {
    val world = World()
    val result = world.handleRequest(Request(command = "launch"))
    assertThat(world.robotCount).isEqualTo(1)
    assertThat(result).isEqualTo(CommandResult(result = "OK"))
}
```

This doesn't compile, because the `CommandResult` class does not exist yet. Let's create that first.

```kotlin
class CommandResult(result: String)
```

Good enough for now. Now our test fails:
`expected:<[nl.dirkgroot.robotworlds.CommandResult@750e2b97]> but was:<[kotlin.Unit]>`. That looks ugly. Let's make it
prettier:

```kotlin
data class CommandResult(val result: String)
```

[Data classes](https://kotlinlang.org/docs/data-classes.html) in Kotlin are a convenient way to create classes that are
meant to hold data. Among other things, the Kotlin compiler automatically provides data classes with `equals`,
`hashCode` and `toString`. The generated`toString` method makes the error message look nicer.

There, that's a lot better: `expected:<[CommandResult(result=OK)]> but was:<[kotlin.Unit]>`. Now, let's make the
test pass.

```kotlin
fun handleRequest(request: Request): CommandResult {
    launchRobot()
    return CommandResult(result = "OK")
}
```

### (De)serialisation

We're still working on the game server. Now that we have a basic `launch` command working, let's see what we need to add
to the server so we can invoke it using a JSON message. Let's ignore the TCP part for now and take a small step up the
call stack. I think we'll need a `MessageReceiver` which translates JSON messages to invocations of
`World::handleRequest`.

No, let's back off a little bit, and first make sure we can create a `Request` from JSON. If we have that in place,
creating a `MessageReceiver` should be trivial.

```kotlin
@Test
fun `create a request from JSON`() {
    val request = Request.fromJSON("""{ "command": "launch" }""")
    assertThat(request).isEqualTo(Request(command = "launch"))
}
```

I'll let IntelliJ generate a stub for `Request::fromJSON`.

```kotlin
class Request(command: String) {
    companion object {
        fun fromJSON(json: String): Request {
            TODO("Not yet implemented")
        }
    }
}
```

Now our test fails, of course: `An operation is not implemented: Not yet implemented`. Let's use
[Kotlin serialization](https://kotlinlang.org/docs/serialization.html) to implement this.

```kotlin
fun fromJSON(json: String): Request {
    return Json.decodeFromString(json)
}
```

Still fails: `Serializer for class 'Request' is not found. Mark the class as @Serializable or provide the serializer
explicitly`. `Request` needs to be serializable.

```kotlin
@Serializable
class Request(command: String)
```

Now we get a compiler error: "This class is not serializable automatically because it has primary constructor parameters
that are not properties". Okay, let's make `command` a property.

```kotlin
@Serializable
class Request(val command: String)
```

Now our test fails with an ugly
message: `expected:<...robotworlds.Request@[4de025bf]> but was:<...robotworlds.Request@[538613b3]>`. I suspect this is
because `Request` doesn't have an `equals` implementation, so let's upgrade it to a `data class`, like we did
with `CommandResult`.

```kotlin
@Serializable
data class Request(val command: String)
```

That was nice and easy. Now let's move on to our `MessageReceiver`.

### Launch with JSON

First, we'll try to invoke the `launch` command using a JSON message.

```kotlin
class MessageReceiverTest {
    @Test
    fun `invoke launch command with JSON message`() {
        val world = World()
        val messageReceiver = MessageReceiver(world)
        messageReceiver.receive("""{ "command": "launch" }""")
        assertThat(world.robotCount).isEqualTo(1)
    }
}
```

Doesn't compile of course, so let's create the `MessageReceiver` class, with a `receive` method.

```kotlin
class MessageReceiver(world: World) {
    fun receive(jsonMessage: String) {
        TODO("Not yet implemented")
    }
}
```

Implement it to make the test pass. For this, we'll have to make the constructor parameter `world` a class member.

```kotlin
class MessageReceiver(private val world: World) {
    fun receive(jsonMessage: String) {
        val request = Request.fromJSON(jsonMessage)
        world.handleRequest(request)
    }
}
```

I'd like `receive` to return the result in JSON format, so let's make sure we can serialize `CommandResult` to JSON.

```kotlin
class CommandResultTest {
    @Test
    fun `serialize to JSON`() {
        val json = CommandResult(result = "OK").toJSON()
        assertThat(json).isEqualTo("""{ "result": "OK" }""")
    }
}
```

Implement it in one fell swoop and use Kotlin serialization again.

```kotlin
@Serializable
data class CommandResult(val result: String) {
    fun toJSON(): String {
        return Json.encodeToString(this)
    }
}
```

The test fails: `expected:<"{[ "result": "OK" ]}"> but was:<"{["result":"OK"]}">`. Ah, apparently `Json.encodeToString`
uses as few spaces as possible. Let's adjust the test.

```kotlin
assertThat(json).isEqualTo("""{"result":"OK"}""")
```

Done. Now let's change `MessageReceiver::receive` to return a result in JSON format.

```kotlin
@Test
fun `invoke launch command with JSON message`() {
    val world = World()
    val messageReceiver = MessageReceiver(world)
    val result = messageReceiver.receive("""{ "command": "launch" }""")
    assertThat(world.robotCount).isEqualTo(1)
    assertThat(result).isEqualTo("""{"result":"OK"}""")
}
```

This fails with `expected:<["{"result":"OK"}"]> but was:<[kotlin.Unit]>`. We'll use the `CommandResult` returned
by `World::handleRequest` as our return value.

```kotlin
fun receive(jsonMessage: String): String {
    val request = Request.fromJSON(jsonMessage)
    return world.handleRequest(request).toJSON()
}
```

### A little refactoring

In the previous article, we already had a suspicion that `World` would eventually need to be split up. Now that we have
a `MessageReceiver` in place, it's becoming obvious that `World::handleRequest` is out of place. Remember, its job is to
take a request, execute the requested command, and return a result. Its responsibility is focused on request and
response messages. I think the primary responsibility of `World` should be to handle game logic. It shouldn't need to
worry about messages, and `MessageReceiver` seems like a much better place for that. So let's move `handleRequest`
to `MessageReceiver`.

I'll start by copy-pasting `handleRequest` to `MessageReceiver`, change `receive` to
use `MessageReceiver::handleRequest` instead of `World::handleRequest`, and make it compile.

```kotlin
fun receive(jsonMessage: String): String {
    val request = Request.fromJSON(jsonMessage)
    return handleRequest(request).toJSON()
}

fun handleRequest(request: Request): CommandResult {
    world.launchRobot()
    return CommandResult(result = "OK")
}
```

Tests still pass. Now we have two similar tests for the `launch` command, one in `WorldTest` and one
in `MessageReceiverTest`. This is the one in `WorldTest`.

```kotlin
@Test
fun `execute a launch command`() {
    val world = World()
    val result = world.handleRequest(Request(command = "launch"))
    assertThat(world.robotCount).isEqualTo(1)
    assertThat(result).isEqualTo(CommandResult(result = "OK"))
}
```

And here's the test in `MessageReceiverTest`.

```kotlin
@Test
fun `invoke launch command with JSON message`() {
    val world = World()
    val messageReceiver = MessageReceiver(world)
    val result = messageReceiver.receive("""{ "command": "launch" }""")
    assertThat(world.robotCount).isEqualTo(1)
    assertThat(result).isEqualTo("""{"result":"OK"}""")
}
```

These tests are basically the same, except that the test for `MessageReceiver` is using JSON, and the test for `World`
is using a `Request` object. Now, we could do two things: We could move the test in `WorldTest` to `MessageReceiverTest`
and change it, so it uses `MessageReceiver::handleRequest`, or we could just delete the test in `WorldTest`. Let's get
rid of this duplication by deleting the test in `WorldTest`.

Tests still pass. Now that the duplicate test is gone, `World::handleRequest` is not used anywhere, so we can delete
that as well. Finally, we can make `MessageReceiver::handleRequest` private, because it's only used
in `MessageReceiver::receive`.

```kotlin
private fun handleRequest(request: Request): CommandResult {
    world.launchRobot()
    return CommandResult(result = "OK")
}
```

### Are you being served?

Now, let's start setting up a TCP socket listener.

```kotlin
class SocketListenerTest {
    @Test
    fun `listens on free TCP port when no port is given`() {
        val socketListener = SocketListener()
        val port = socketListener.port

        val socket = Socket("127.0.0.1", port)

        assertThat(socket.isConnected)
            .isTrue()

        socket.close()
    }
}
```

Create the `SocketListener` class.

```kotlin
class SocketListener {
    val port = 1
}
```

And, as expected, our test fails: `java.net.ConnectException: Connection refused`. Making it pass is simple.

```kotlin
class SocketListener {
    private val serverSocket = ServerSocket(0)

    val port get() = serverSocket.localPort
}
```

We initialize the `ServerSocket` with port `0`, so it will automatically choose an available port. If we pick a fixed
port, there's always a (small) risk that port is already taken, which could lead to flaky tests on the CI/CD pipeline.
We don't want flaky tests, so I'll do whatever I can do to prevent that from happening.

The test code is a little verbose, so let's make it more readable by using Kotlin's handy
dandy [`use`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.io/use.html) extension.

```kotlin
@Test
fun `listens on free TCP port when no port is given`() {
    val socketListener = SocketListener()
    val port = socketListener.port

    Socket("127.0.0.1", port).use {
        assertThat(it.isConnected).isTrue()
    }
}
```

That's a lot better. Now let's see if we can make it handle a request.

```kotlin
@Test
fun `handles a command`() {
    val socketListener = SocketListener()
    val port = socketListener.port

    Socket("127.0.0.1", port).use {
        it.getOutputStream().writer().write("""{ "command": "launch" }""")
        val response = it.getInputStream().bufferedReader().readLine()

        assertThat(response).isEqualTo("""{"result":"OK"}""")
    }
}
```

There's a number of things that can be improved in this code, but let's first see what happens. The `SocketListener` is
not handling any messages, so the test waits indefinitely for a response. That's not ideal. We want our test to fail,
not to wait forever. Let's fix that by using JUnit's `assertTimeoutPreemptively`[^1].

```kotlin
@Test
fun `handles a command`() {
    val socketListener = SocketListener()
    val port: Int = socketListener.port

    assertTimeoutPreemptively(Duration.ofSeconds(1)) {
        Socket("127.0.0.1", port).use {
            it.getOutputStream().writer().write("""{ "command": "launch" }""")
            val response = it.getInputStream().bufferedReader().readLine()

            assertThat(response).isEqualTo("""{"result":"OK"}""")
        }
    }
}
```

Now the test properly fails. The timeout of 1 second is my arbitrary choice. It's always risky to have these kinds of
tests, because they tend to be flaky. For now, I'm okay with this, because I'll probably only be running these tests on
my developer laptop, and I don't think it will be flaky. If it is, I'll just run the test again, or increase the
timeout.

Let's see if we can make the test pass. To do that, we'll need to start a separate thread which handles requests in the
background. We'll also need to hand our `SocketListener` a `MessageReceiver`, so it will be able to handle the messages
it receives. Let's first change the instantiation of `SocketListener` in our test.

```kotlin
val socketListener = SocketListener(MessageReceiver(World()))
```

Now we need to change the `SocketListener` constructor.

```kotlin
class SocketListener(private val messageReceiver: MessageReceiver)
```

And finally we need to start the thread and handle the request.

```kotlin
init {
    Thread {
        serverSocket.accept().use {
            val request = it.getInputStream().bufferedReader().readLine()
            val response = messageReceiver.receive(request)

            it.getOutputStream().writer().write("$response\n")
        }
    }.start()
}
```

This fails with a timeout. Ah, I forgot to send a newline after the request message. Let's change that.

```kotlin
it.getOutputStream().writer().write("""{ "command": "launch" }""" + "\n")
```

It still fails. Apparently I'm doing something wrong, but it isn't immediately obvious to me what that is. I see no
other option than to use the debugger. Okay, so the `SocketListener` keeps blocking
on `val request = it.getInputStream().bufferedReader().readLine()`. Do we need to `flush` the writer? Let's try.

```kotlin
it.getOutputStream().writer().apply {
    write("""{ "command": "launch" }""" + "\n")
    flush()
}
```

Te test still fails: `expected:<"{"result":"OK"}"> but was:<null>`. It looks like we also need to flush
in `SocketListener`.

```kotlin
it.getOutputStream().writer().apply {
    write("$response\n")
    flush()
}
```

Yes, that does the trick. Now, our code can be improved a bit, so let's do that. First of all, let's make our test code
a bit more readable.

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

Now it's a bit clearer what our test is actually doing. Let's do something similar in `SocketListener`.

```kotlin
init {
    Thread {
        serverSocket.accept().use {
            val request = receiveRequest(it)
            val response = messageReceiver.receive(request)

            sendResponse(it, response)
        }
    }.start()
}

private fun receiveRequest(socket: Socket) =
    socket.getInputStream().bufferedReader().readLine()

private fun sendResponse(socket: Socket, response: String) {
    socket.getOutputStream().writer().apply {
        write("$response\n")
        flush()
    }
}
```

I think this is a good moment to call it a day, let's do a little retrospective.

## Retro

Everything went smooth, until we started messing with sockets. I don't frequently work with sockets or input/output
streams, so the need for flushing wasn't immediately obvious to me. This is what typically happens when you're working
on the "edges" of the system. That's where we need to deal with I/O, or databases, or queues, and often times non
intuitive API's.

### Humble object

This is why I keep as much logic as possible separated from the code that has to deal with these kinds of API's. That
way, we maximize the amount of code that can easily be tested and understood. This is what's called the
[Humble Object](https://martinfowler.com/bliki/HumbleObject.html) pattern. Our `SocketListener` is a humble object. Its
only responsibilies are to accept connections, pass messages on to `MessageReceiver` and send the result back to the
client.

### YAGNI

Our `SocketListener` is far from done. It accepts one connection, handles one request and then stops. However, our goal
is not to build a working feature, but to get just enough functionality working to allow us to start working on the
first real feature. Our focus is not on functionality, but on setting up the general structure of the program and making
sure it's all testable.

I'm relentlessly applying the [YAGNI principle](https://en.wikipedia.org/wiki/You_aren%27t_gonna_need_it) to get to our
walking skeleton. I'm happily leaving out anything that is not strictly necessary for our goals. We'll see if I'm going
to regret doing that, but I think I won't. Time will tell 😃.

Thanks for reading, and I'll see you in the next one. You can find my source code on
GitHub: <https://github.com/dirkgroot/robot-worlds>.

[^1]: This assertion asserts that the lambda finishes before a timeout is exceeded. Read the
the [JavaDocs](https://junit.org/junit5/docs/current/api/org.junit.jupiter.api/org/junit/jupiter/api/Assertions.html#assertTimeoutPreemptively(java.time.Duration,org.junit.jupiter.api.function.Executable))
for more information.
