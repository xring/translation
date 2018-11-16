# Integrating Akka Streams and Akka Actors: Part I

# 集成 Akka Streams 和 Akka Actors：第一部分

I expand on these concepts in my [Reactive Summit presentation](https://www.youtube.com/watch?v=qaiwalDyayA).

> 我在[反应式峰会](https://www.youtube.com/watch?v=qaiwalDyayA)的演讲中详述了这些观点。

Most people are attracted to Akka with the promise of the actor model providing a better abstraction for building scalable and resilient distributed systems. Since Akka attempts to solve such a challenging set of problems—from concurrency, to distributed computation, to fault tolerance—it takes some time to appreciate the depth and breadth of the Akka toolkit.

很多人因为 actor 模型所承诺的为构建具备弹性和伸缩性的分布式系统提供更好的抽象而被 Akka 所吸引。Akka 尝试解决从决发到分布式计算到容错等一系列极具挑战性的问题，因此需要花一些时间来领会 Akka 工具包的深度和广度。

After getting started with actors and welcoming the benefits of this approach, it is not uncommon for people to encounter traditional concurrent-programming and distributed-systems problems—related to flow control, out-of-memory exceptions, or poor performance—which can be somewhat discouraging. This is usually when people discover the Akka Streams API. In a previous essay, I detailed this journey, providing a motivating example for using the Akka Streams API. Others have detailed similar experiences. In another essay, I demonstrated how the Akka Streams API simplifies patterns that are fundamental to the domain of streaming measurement-data, making these patterns straightforward, scalable, resilient, and reliable.

在开始上手 actors 并从这种方法中受益后，人们普遍会遇到流控、内存溢出、性能低等传统并发编程和分布式系统中让人气馁的问题。通常在这个时间点，人们会发现 Akka Streams API 可以使用。在前一篇文章中，我通过一个相当有意思的例子来详细说明了 Akka Streams API。其他人可能已经有了类似的经验。在另一篇文章中，我演示了 Akka Streams 的 API 是如何简化流式测量数据领域的基本模式，使这些模式明确、可扩展、具备弹性又可靠。


After realizing the power of the Akka Streams API, people are often confused as to how it relates to Akka Actors. Some people question if actors are even necessary. Actors are definitely necessary. Actors solve complimentary problems. In this article, I want to explore some introductory examples for integrating the Akka Streams API with Akka Actors, to solve complimentary problems, and build scalable, reliable, and efficient distributed systems.

在意识到 Akka Streams API 的强大能力后，人们常会疑惑：它应该如何和 Akka Actors 产生关联。甚至有人会问 actors 是否还有必要存在。Actors 当然是必要的。Actors 解决了（和 Akka Streams API）互补的问题。在这篇文章中，我想去探索一些集成 Akka Streams API 和 Akka Actors 的入门级例子来解决互补的问题并且构建可扩展、可靠、高效的分布式系统。

## Maintaining State

## 维护状态

While Akka Streams can maintain mutable state though elements like statefulMapConcat, or a custom graph stage, it is uncommon, and even somewhat unnatural to do so. I might even go as far as saying that it is an anti-pattern, and that it only makes sense to maintain mutable state in a stream if there is no alternative. Actors, on the other hand, are great for encapsulating and maintaining mutable state.

尽管 Akka Streams 可以使用像 statefulMapConcat 或者自定义的 graph stage 来维护状态，但这并不常见，这么做甚至有些不自然。我甚至会说这是反模式的，只有在别无选择时在 stream 中维护状态才是有意义的。而另一方面，使用 Actors 来封状和维护可变状态是很棒的的选择。

In the following example, the WebSocket server is designed to handle connections from a large number of clients. Each client emits an unbounded stream of measurements. The measurementsWebSocket stream computes intermediate sums and sends these intermediate results to the Total actor, at least every second.

在下面的例子中，WebSocket 服务端被设计用来处理大量客户端的连接。每个客户端都发送无限的测量数据流。measurementsWebSocket stream 以每秒至少一次的频率计算中间结果的和，然后将这些中间结果送到 Total actor。
```scala
val total = system.actorOf(Props[Total], "total")

val measurementsWebSocket =  
  Flow[Message]
    .collect {
      case TextMessage.Strict(text) =>
        Future.successful(text)
      case TextMessage.Streamed(textStream) =>
        textStream.runFold("")(_ + _)
          .flatMap(Future.successful)
    }
    .mapAsync(1)(identity)
    .groupedWithin(1000, 1 second)
    .map(messages => (messages.last, Messages.parse(messages)))
    .map {
      case (lastMessage, measurements) =>
        total ! Increment(measurements.sum)
        lastMessage
    }
    .map(Messages.ack)

val route =  
  path("measurements") {
    get {
      handleWebSocketMessages(measurementsWebSocket)
    }
  }

val bindingFuture = Http().bindAndHandle(route, "localhost", 8080)
```
The Total actor maintains the cumulative total for all clients.

Total actor 维护所有客户端的累加和。
```scala
object Total {  
  case class Increment(value: Long)
}

class Total extends Actor {  
  var total: Long = 0

  override def receive: Receive = {
    case Increment(value) =>
      total = total + value
  }
}
```
This is a simple example of how the Akka Streams API can be used to handle the processing of unbounded streaming-data, while an actor can be used to maintain mutable state.

这是一个简单的示例，说明了 Akka Streams API 是如何来处理无限的流数据的和 actor 是如何被用来维护中间状态的。

## Observing State

## 观测状态

A logical extension to this first example is to be able to observe the cumulative total maintained by this actor. An easy way to do this is to add an HTTP route to the server, retrieving the current value through an HTTP GET request.

对上一个例子的逻辑延伸是使得 actor 维护的累加和可以被观测到。一种简单的方法是在服务端添加一个 HTTP 路由，通过 GET 请求来获取这个累加和。
```scala
val route =  
  path("measurements") {
    get {
      handleWebSocketMessages(measurementsWebSocket)
    }
  } ~ path("total") {
    get {
      import akka.pattern.ask
      implicit val askTimeout = Timeout(30 seconds)
      onSuccess(total ? GetTotal) {
        case CurrentTotal(value) =>
          complete(s"The total is : $value")
      }
    }
  }
```
The Total actor can be extended to return the current total.

Total actor 可以被扩展以支持返回当前的 total 值。
```
object Total {  
  case object GetTotal
  case class Increment(value: Long)
  case class CurrentTotal(value: Long)
}

class Total extends Actor {  
  var total: Long = 0

  override def receive: Receive = {
    case Increment(value) =>
      total = total + value
    case GetTotal =>
      sender ! CurrentTotal(total)
  }
}
```
This example uses the ask pattern to query the actor, asynchronously, for the current total. Note that this is not something that can be accomplished with Akka Streams, as there is no way to query a stream directly.

这个例子使用 ask 模式来进行异步查询 actor 中当前 total 的值。需要注意的是，这无法用 Akka Streams 来完成，因为我们无法直接查询一个 stream。

## Flow Control

## 流控
This all looks great, but as systems like this scale, the number of messages the Total actor needs to process can become significant, and lead to performance problems, like timeouts querying the actor using the ask pattern. A subtle aspect of sending the intermediate sums to the Total actor is that these messages are sent asynchronously, and there is no feedback from the actor to the stream—the messages are fire-and-forget. In other words, there is a discontinuity between the flow-controlled, unbounded stream-processing offered by the Akka Streams API, and the asynchronous messaging of actors, which is not flow-controlled. In addition to performance problems, this can lead to out-of-memory exceptions, as I detailed in my essay motivating the need for the Akka Streams API.

目前看起来还不错，但这种规模的系统中 Total actor 需要处理的消息数值得注意，可能会引发性能问题，比如使用 ask 模式来查询会超时。在将中间和送到 Total actor 时有个点很微妙：这些消息是采用 fire-and-forget 异步发送的，而 Total actor 并没有给 stream 任何反馈。换句话说，在 Akka Streams API 提供的无限流处理中的流控和 actors 的异步消息之间存在不连续性，这部分没有进行流控。除了性能问题，这也可能导致 OOM 异常，就像我在[另一篇](http://blog.colinbreck.com/akka-streams-a-motivating-example/)鼓励使用 Akka Stream API 的文章中所说的那样。

It is possible to use actor-based techniques to add linear scalability, for example, adding a pool of actors to handle these requests. While approaches like this remain complimentary, there are more native ways to address these streaming-data challenges that can maintain the streaming interface and flow control, removing this discontinuity between streams and actors.

使用基于 actor 的技术来达到线性扩展是有可能的，比如，添加一堆的 actors 来处理这些请求。尽管这种方法看起来不错，但有更原生的方法来处理这些流数据的挑战，移除 streams 和 actors 之间的不连续性。

## Ask Pattern with mapAsync
Probably the most straightforward and flexible way to achieve flow control between streams and actors is to use the ask pattern, to asynchronously send a message to an actor, combined with the mapAsync stage to await the response, before sending additional messages to the actor. The following example is a safer and more scalable implementation of the first example that I provided. It will only send a single message at a time to the Total actor, backpressuring otherwise.
```scala
val measurementsWebSocket =  
  Flow[Message]
    .collect {
      case TextMessage.Strict(text) =>
        Future.successful(text)
      case TextMessage.Streamed(textStream) =>
        textStream.runFold("")(_ + _)
          .flatMap(Future.successful)
    }
    .mapAsync(1)(identity)
    .groupedWithin(1000, 1 second)
    .map(messages => (messages.last, Messages.parse(messages)))
    .mapAsync(1) {
      case (lastMessage, measurements) =>
        import akka.pattern.ask
        implicit val askTimeout = Timeout(30 seconds)
        (total ? Increment(measurements.sum))
          .mapTo[Done]
          .map(_ => lastMessage)
    }
    .map(Messages.ack)
```
The Total actor is implemented as follows.
```scala
object Total {  
  case class Increment(value: Long)
}

class Total extends Actor {  
  var total: Long = 0

  override def receive: Receive = {
    case Increment(value) =>
      total = total + value
      sender ! Done
  }
}
```
## An Actor as a Sink
If the actor you are interacting with is essentially the termination point of the stream, it can be treated as a fully backpressured sink, using Sink.actorRefWithAck. As usual, the actor processes only one message at a time, but, in this case, the actor must respond with an acknowledgment message to support backpressure. In addition, the actor must also acknowledge an initialization message, to indicate that it is ready to handle messages. It may optionally handle a message when the stream is completed.

Using this approach, it is possible to reimplement the first example that I provided, with a single Total actor exposed as a unique sink for every stream.
```scala
val total = system.actorOf(Props[Total], "total")

val measurementsWebSocket = (sink: Sink[Increment, NotUsed]) =>  
  Flow[Message]
    .collect {
      case TextMessage.Strict(text) =>
        Future.successful(text)
      case TextMessage.Streamed(textStream) =>
        textStream.runFold("")(_ + _)
          .flatMap(Future.successful)
    }
    .mapAsync(1)(identity)
    .groupedWithin(1000, 1 second)
    .map(Messages.parse)
    .map(messages => Increment(messages.sum, messages.last))
    .alsoTo(sink)
    .map(increment => Messages.ack(increment.lastMessage))

val route =  
  path("measurements" / LongNumber) { id =>
    get {
      val sink = Sink.actorRefWithAck(total, Init, Ack, Complete(id))
      handleWebSocketMessages(measurementsWebSocket(sink))
    }
  }
```
The implementation of the actor is as follows.
```scala
object Total {  
  case object Init
  case object Ack
  case class Complete(id: Long)
  case class Increment(value: Long, lastMessage: Long)
}

class Total extends Actor {  
  var total: Long = 0

  override def receive: Receive = {
    case _: Init.type =>
      sender ! Ack
    case Increment(value, _) =>
      total = total + value
      sender ! Ack
    case Complete(id) =>
      println(s"WebSocket terminated for Id : $id")
  }
}
```
## Sending Message to a Stream
Sending messages to an Akka Stream is a very useful pattern, but, at the time of this writing, it is also quite challenging, and it needs to be done with great care. There are two means for sending messages to an Akka Stream. The first is to use Source.actorRef. Messages sent to the actor materialized from this source will be emitted downstream, when there is demand. Otherwise, they will be buffered, up to the specified maximum, in conjunction with the overflow strategy.
```scala
val ref =  
  Source.actorRef[Long](Int.MaxValue, OverflowStrategy.fail)
    .to(Sink.foreach(println))
    .run()

Source(1 to Int.MaxValue)  
  .map(x => ref ! x)
  .runWith(Sink.ignore)
The drawback of this approach is that there is no feedback to the producer of the events. In other words, there is no backpressure, and it is vulnerable to out-of-memory exceptions (ref ! x is fire-and-forget).

The second approach is to use Source.queue, which is an improvement, since it can provide backpressure. The offer method returns a Future, which completes with the result of the enqueue operation.

val queue =  
  Source.queue[Long](Int.MaxValue, OverflowStrategy.backpressure)
    .to(Sink.foreach(println))
    .run()

Source(1 to Int.MaxValue)  
  .mapAsync(1)(x => queue.offer(x))
  .runWith(Sink.ignore)
```
The limitation of this approach is that the Source.queue must only be used from a single thread. Therefore, in order to send messages concurrently, one needs to serialize messages through an actor, or other means. The result is that I cannot send messages to the queue directly from multiple WebSocket connections, as I did in the first example, without additional synchronization. I hope that the Akka Streams API adds a thread-safe mechanism for this in the future.

Note that with both Source.actorRef and Source.queue, there are other options for flow control, like discarding the first element, discarding the last element, or discarding the entire buffer. Given the nature of the systems that I work on, it is usually a requirement to reliably process every single element in the stream, therefore, I rarely use these options.

For completeness, I will briefly mention that ActorPublisher and ActorSubscriber are also means of interfacing actors and streams, however, they may be deprecated in the future, so I will not detail them here. At the time of this writing, the ActorPublisher and ActorSubscriber also do not respect the streams supervision strategy, which can make error handling challenging.

## Looking Ahead
In this article, I provided some introductory examples for integrating Akka Streams and Akka Actors. Rather than simply relying on the native, asynchronous messaging of actors, it is advantageous to build systems that combine streams and actors, using the techniques that I have demonstrated here. This leverages the advantages of streams—like flow control, bounded memory, and complex streaming-semantics—while integrating with the complementary functions of actors—like maintaining state and exposing an asynchronous query interface. In a [future article](http://blog.colinbreck.com/integrating-akka-streams-and-akka-actors-part-ii/), I will explore more sophisticated ways to integrate actors and streams, examining distributed execution and fault tolerance.
