# gRPC Kotlin - Coroutine based gRPC for Kotlin

[![CircleCI](https://img.shields.io/circleci/project/github/rouzwawi/grpc-kotlin.svg)](https://circleci.com/gh/rouzwawi/grpc-kotlin)
[![Maven Central](https://img.shields.io/maven-central/v/io.rouz/grpc-kotlin-gen.svg)](https://search.maven.org/#search%7Cga%7C1%7Cg%3A%22io.rouz%22%20grpc-kotlin-gen)

gRPC Kotlin is a [protoc] plugin for generating native Kotlin bindings using [coroutine primitives] for [gRPC] services.

## Why?

The asynchronous nature of bidirectional streaming rpc calls in gRPC makes them a bit hard to implement
and read. Getting your head around the `StreamObserver<T>`'s can be a bit tricky at times. Specially
with the method argument being the response observer and the return value being the request observer, it all
feels a bit backwards to what a plain old synchronous version of the handler would look like.

In situations where you'd want to coordinate several request and response messages in one call, you'll and up
having to manage some tricky state and synchronization between the observers. There's some [reactive bindings]
for gRPC which make this easier. But I think we can do better!

Enter Kotlin Coroutines! By generating native Kotlin stubs that allows us to use [`suspend`] functions and 
[`Channel`], we can write our handler and client code in idiomatic and easy to read Kotlin style.

## Quick start

note: This has been tested with `gRPC 1.15.1`, `protobuf 3.5.1`, `kotlin 1.3.0` and `coroutines 1.0.0`.

Add a gRPC service definition to your project

`greeter.proto`

```proto
syntax = "proto3";
package org.example.greeter;

option java_package = "org.example.greeter";
option java_multiple_files = true;

message GreetRequest {
    string greeting = 1;
}

message GreetReply {
    string reply = 1;
}

service Greeter {
    rpc Greet (GreetRequest) returns (GreetReply);
    rpc GreetServerStream (GreetRequest) returns (stream GreetReply);
    rpc GreetClientStream (stream GreetRequest) returns (GreetReply);
    rpc GreetBidirectional (stream GreetRequest) returns (stream GreetReply);
}
```

Run the protoc plugin to get the generated code, see [build tool configuration](#maven-configuration)

### Server

After compilation, you'll find the generated Kotlin stubs in an `object` named `GreeterGrpcKt`. Both
the service base class and client stub will use `suspend` and `Channel<T>` instead of the typical
`StreamObserver<T>` interfaces.

All functions have the [`suspend`] modifier so they can call into any suspending code, including the
[core coroutine primitives] like `delay` and `async`.

All the server streaming calls return a `ReceiveChannel<TReply>` and can easily be implemented using
`produce<TReply>`.

All client streaming calls receive an argument of `ReceiveChannel<TRequest>` where they can `receive()`
messages from the caller.

Here's an example server that demonstrates how each type of endpoint is implemented.

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.ReceiveChannel
import kotlinx.coroutines.channels.produce

class GreeterImpl : GreeterGrpcKt.GreeterImplBase(
  coroutineContext = newFixedThreadPoolContext(4, "server-pool")
) {

  // unary rpc
  override suspend fun greet(request: GreetRequest): GreetReply {
    return GreetReply.newBuilder()
        .setReply("Hello " + request.greeting)
        .build()
  }

  // server streaming rpc
  override suspend fun greetServerStream(request: GreetRequest) = produce<GreetReply> {
    send(GreetReply.newBuilder()
        .setReply("Hello ${request.greeting}!")
        .build())
    send(GreetReply.newBuilder()
        .setReply("Greetings ${request.greeting}!")
        .build())
  }

  // client streaming rpc
  override suspend fun greetClientStream(requestChannel: ReceiveChannel<GreetRequest>): GreetReply {
    val greetings = mutableListOf<String>()

    for (request in requestChannel) {
      greetings.add(request.greeting)
    }

    return GreetReply.newBuilder()
        .setReply("Hi to all of $greetings!")
        .build()
  }

  // bidirectional rpc
  override suspend fun greetBidirectional(requestChannel: ReceiveChannel<GreetRequest>) = produce<GreetReply> {
    var count = 0

    for (request in requestChannel) {
      val n = count++
      launch {
        delay(1000)
        send(GreetReply.newBuilder()
            .setReply("Yo #$n ${request.greeting}")
            .build())
      }
    }
  }
}
```

### Client

The generated client stub is also fully implemented using `suspend` functions, `Deferred<TReply>`
and `SendChannel<TRequest>`.

```kotlin
import io.grpc.ManagedChannelBuilder
import kotlinx.coroutines.delay
import kotlinx.coroutines.launch
import kotlinx.coroutines.runBlocking

fun main(args: Array<String>) {
  val localhost = ManagedChannelBuilder.forAddress("localhost", 8080)
      .usePlaintext(true)
      .build()
  val greeter = GreeterGrpcKt.newStub(localhost)

  runBlocking {
    // === Unary call =============================================================================

    val unaryResponse = greeter.greet(req("Alice"))
    println("unary reply = ${unaryResponse.reply}")

    // === Server streaming call ==================================================================

    val serverResponses = greeter.greetServerStream(req("Bob"))
    for (serverResponse in serverResponses) {
      println("server response = ${serverResponse.reply}")
    }

    // === Client streaming call ==================================================================

    val manyToOneCall = greeter.greetClientStream()
    manyToOneCall.send(req("Caroline"))
    manyToOneCall.send(req("David"))
    manyToOneCall.close()
    val oneReply = manyToOneCall.await()
    println("single reply = ${oneReply.reply}")

    // === Bidirectional call =====================================================================

    val bidiCall = greeter.greetBidirectional()
    launch {
      var n = 0
      for (greetReply in bidiCall) {
        println("r$n = ${greetReply.reply}")
        n++
      }
      println("no more replies")
    }

    delay(200)
    bidiCall.send(req("Eve"))

    delay(200)
    bidiCall.send(req("Fred"))

    delay(200)
    bidiCall.send(req("Gina"))

    bidiCall.close()
  }
}
```

## RPC method details

### Unary call

> `rpc Greet (GreetRequest) returns (GreetReply);`

#### Service

A suspendable function which returns a single message.

```kotlin
override suspend fun greet(request: GreetRequest): GreetReply {
  // return GreetReply message
}
```

#### Client

Suspendable call returning a single message.

```kotlin
val response: GreetReply = stub.greet( /* GreetRequest */ )
```

### Streaming request, Unary response

> `rpc GreetClientStream (stream GreetRequest) returns (GreetReply);`

#### Service

A suspendable function which returns a single message, and receives messages from a `ReceiveChannel<T>`.

```kotlin
override suspend fun greetClientStream(requestChannel: ReceiveChannel<GreetRequest>): GreetReply {
  // receive request messages
  val firstRequest = requestChannel.receive()
  
  // or iterate all request messages
  for (request in requestChannel) {
    // ...
  }

  // return GreetReply message
}
```

#### Client

Using `send()` and `close()` on `SendChannel<T>`.

```kotlin
val call: ManyToOneCall<GreetRequest, GreetReply> = stub.greetClientStream()
call.send( /* GreetRequest */ )
call.send( /* GreetRequest */ )
call.close() //  don't forget to close the send channel

val responseMessage = call.await()
```

### Unary request, Streaming response

> `rpc GreetServerStream (GreetRequest) returns (stream GreetReply);`

#### Service

Using `produce` and `send()` to send a stream of messages.

```kotlin
override suspend fun greetServerStream(request: GreetRequest) = produce<GreetReply> {
  send( /* GreetReply message */ )
  send( /* GreetReply message */ )
  // ...
}
```

Note that `close()` or `close(Throwable)` should not be used, see [Exception handling](#exception-handling).

In `kotlinx-coroutines-core:1.0.0` `produce` is marked with `@ExperimentalCoroutinesApi`. In order
to use it, mark your server class with `@UseExperimental(ExperimentalCoroutinesApi::class)` and
add the `-Xuse-experimental=kotlin.Experimental` compiler flag.

#### Client

Using `receive()` on `ReceiveChannel<T>` or iterating with a `for` loop.

```kotlin
val responses: ReceiveChannel<GreetReply> = stub.greetServerStream( /* GreetRequest */ )

// await individual responses
val responseMessage = serverResponses.receive()

// or iterate all responses
for (responseMessage in responses) {
  // ...
}
```

### Full bidirectional streaming

> `rpc GreetBidirectional (stream GreetRequest) returns (stream GreetReply);`

#### Service

Using `produce` and `send()` to send a stream of messages. Receiving messages from a `ReceiveChannel<T>`.

```kotlin
override suspend fun greetBidirectional(requestChannel: ReceiveChannel<GreetRequest>) = produce<GreetReply> {
  // receive request messages
  val firstRequest = requestChannel.receive()
  send( /* GreetReply message */ )
  
  val more = requestChannel.receive()
  send( /* GreetReply message */ )
  
  // ...
}
```

Note that `close()` or `close(Throwable)` should not be used, see [Exception handling](#exception-handling).

In `kotlinx-coroutines-core:1.0.0` `produce` is marked with `@ExperimentalCoroutinesApi`. In order
to use it, mark your server class with `@UseExperimental(ExperimentalCoroutinesApi::class)` and
add the `-Xuse-experimental=kotlin.Experimental` compiler flag.

#### Client

Using both a `SendChannel<T>` and a `ReceiveChannel<T>` to interact with the call.

```kotlin
val call: ManyToManyCall<GreetRequest, GreetReply> = stub.greetBidirectional()
launch {
  for (responseMessage in call) {
    log.info(responseMessage)
  }
  log.info("no more replies")
}

call.send( /* GreetRequest */ )
call.send( /* GreetRequest */ )
call.close() //  don't forget to close the send channel
```

## Exception handling

The generated server code follows the standard exception propagation for Kotlin coroutines as described
in the [Exception handling] documentation. This means that it's safe to throw exceptions from within
the server implementation code. These will propagate up the coroutine scope and be translated to
`responseObserver.onError(Throwable)` calls. The preferred way to respond with a status code is to
throw a `StatusException`.

Note that you should not call `close(Throwable)` or `close()` from within the `ProducerScope<T>`
blocks you get from `produce` as the producer will automatically be closed when all sub-contexts are
closed (or if an exception is thrown).

## Maven configuration

Add the `grpc-kotlin-gen` plugin to your `protobuf-maven-plugin` configuration (see [using custom protoc plugins](https://www.xolstice.org/protobuf-maven-plugin/examples/protoc-plugin.html))

```xml
<protocPlugins>
    <protocPlugin>
        <id>GrpcKotlinGenerator</id>
        <groupId>io.rouz</groupId>
        <artifactId>grpc-kotlin-gen</artifactId>
        <version>0.0.5</version>
        <mainClass>io.rouz.grpc.kotlin.GrpcKotlinGenerator</mainClass>
    </protocPlugin>
</protocPlugins>
```

Add the kotlin dependencies

```xml
<dependency>
  <groupId>org.jetbrains.kotlin</groupId>
  <artifactId>kotlin-stdlib</artifactId>
  <version>1.3.0</version>
</dependency>
<dependency>
  <groupId>org.jetbrains.kotlinx</groupId>
  <artifactId>kotlinx-coroutines-core</artifactId>
  <version>1.0.0</version>
</dependency>
```

## Gradle configuration

Add the `grpc-kotlin-gen` plugin to the plugins section of `protobuf-gradle-plugin`

```gradle
def protobufVersion = '3.5.1-1'
def grpcVersion = '1.15.1'

protobuf {
    protoc {
        // The artifact spec for the Protobuf Compiler
        artifact = "com.google.protobuf:protoc:${protobufVersion}"
    }
    plugins {
        grpc {
            artifact = "io.grpc:protoc-gen-grpc-java:${grpcVersion}"
        }
        grpckotlin {
            artifact = "io.rouz:grpc-kotlin-gen:0.0.5:jdk8@jar"
        }
    }
    generateProtoTasks {
        all()*.plugins {
            grpc {}
            grpckotlin {}
        }
    }
}
```

Add the kotlin dependencies

```gradle
def kotlinVersion = '1.3.0'
def kotlinCoroutinesVersion = '1.0.0'

dependencies {
    compile "org.jetbrains.kotlin:kotlin-stdlib:$kotlinVersion"
    compile "org.jetbrains.kotlinx:kotlinx-coroutines-core:kotlinCoroutinesVersion"
}
```


[protoc]: https://www.xolstice.org/protobuf-maven-plugin/examples/protoc-plugin.html
[`suspend`]: https://kotlinlang.org/docs/reference/coroutines-overview.html
[coroutine primitives]: https://github.com/Kotlin/kotlinx.coroutines
[core coroutine primitives]: https://github.com/Kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/README.md
[Exception handling]: https://kotlinlang.org/docs/reference/coroutines/exception-handling.html
[`Channel`]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/-channel/index.html
[`Deferred`]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-deferred/index.html
[`async`]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/async.html
[`produce`]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/produce.html
[gRPC]: https://grpc.io/
[reactive bindings]: https://github.com/salesforce/reactive-grpc

