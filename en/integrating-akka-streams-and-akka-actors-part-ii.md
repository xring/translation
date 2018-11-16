# Integrating Akka Streams and Akka Actors: Part II

I expand on these concepts in my [Reactive Summit presentation](https://www.youtube.com/watch?v=qaiwalDyayA).

In part one of this series, I described the challenges of integrating the Akka Streams API with Akka Actors. I demonstrated the basic patterns for interfacing streams and actors, removing the discontinuity between the two. In this article, I will begin to explore more sophisticated ways to integrate actors and streams, in support of building robust and scalable distributed systems, rather than just simple applications, or stand-alone data-processing pipelines. This article will focus on how Akka Actors compliment the Akka Streams API with regard to life-cycle management and fault tolerance.

## Encapsulating Streams With Actors
It is not always immediately obvious that an actor, being a class, can encapsulate a stream, just as it would another member variable. Consider this trivial example of an actor that materializes a stream, to print the integers from 1 to 10.
```scala
class PrintSomeNumbers(implicit materializer: ActorMaterializer) extends Actor {  
  private implicit val executionContext = context.system.dispatcher

  Source(1 to 10)
    .map(_.toString)
    .runForeach(println)
    .map(_ => self ! "done")

  override def receive: Receive = {
    case "done" =>
      println("Done")
      context.stop(self)
  }
}
```
This actor can be used in a program, as follows.
```scala
object TrivialExample extends App {  
  implicit val system = ActorSystem()
  implicit val materializer = ActorMaterializer()
  system.actorOf(Props(classOf[PrintSomeNumbers], materializer))
}
```
Rather than creating the materializer inside the actor, I passed it as an implicit parameter, so that the caller can reuse it. Sometimes I also pass the execution context as an implicit parameter to allow it to be controlled by the caller, which can be important when calling blocking code, but here I just set it to the context.system.dispatcher within the actor.

An extremely important aspect to understand is that the materialized stream is running as a set of actors on the threads of the execution context on which they were allocated. In other words, the stream is running independently from the actor that allocated it. This becomes very important if the stream is long-running, or even infinite, and we want the actor to manage the life-cycle of the stream, such that when the actor stops, the stream is terminated. Expanding on the example above, I will make the stream infinite and use a KillSwitch to manage the life-cycle of the stream.
```scala
class PrintMoreNumbers(implicit materializer: ActorMaterializer) extends Actor {  
  private implicit val executionContext = context.system.dispatcher

  private val (killSwitch, done) =
    Source.tick(0 seconds, 1 second, 1)
      .scan(0)(_ + _)
      .map(_.toString)
      .viaMat(KillSwitches.single)(Keep.right)
      .toMat(Sink.foreach(println))(Keep.both)
      .run()

  done.map(_ => self ! "done")

  override def receive: Receive = {
    case "stop" =>
      println("Stopping")
      killSwitch.shutdown()
    case "done" =>
      println("Done")
      context.stop(self)
  }
}
```
When the actor is stopped, it will also stop the stream.
```scala
object LessTrivialExample extends App {  
  implicit val system = ActorSystem()
  implicit val materializer = ActorMaterializer()
  implicit val executionContext = system.dispatcher
  val actorRef = system.actorOf(Props(classOf[PrintMoreNumbers], materializer))
  system.scheduler.scheduleOnce(5 seconds) {
    actorRef ! "stop"
  }
}
```
## A More Interesting Example
As a more interesting example of encapsulating a stream within an actor, consider creating an actor to represent a physical, industrial device. The following actor simulates a wind turbine. It will be used for testing the performance and scalability of a service that records and aggregates wind-turbine measurements, by simulating a large number of wind turbines. The actor representing the wind turbine is created with a unique identifier for the physical device, and it creates a WebSocket connection, by calling WebSocketClient(id, endpoint, self).
```scala
object WindTurbineSimulator {  
  def props(id: String, endpoint: String)(implicit materializer: ActorMaterializer) =
    Props(classOf[WindTurbineSimulator], id, endpoint, materializer)

  final case object Upgraded
  final case object Connected
  final case object Terminated
  final case class ConnectionFailure(ex: Throwable)
  final case class FailedUpgrade(statusCode: StatusCode)
}

class WindTurbineSimulator(id: String, endpoint: String)  
                          (implicit materializer: ActorMaterializer)
  extends Actor with ActorLogging {
  implicit private val system = context.system
  implicit private val executionContext = system.dispatcher

  val webSocket = WebSocketClient(id, endpoint, self)

  override def postStop() = {
    log.info(s"$id : Stopping WebSocket connection")
    webSocket.killSwitch.shutdown()
  }

  override def receive: Receive = {
    case Upgraded =>
      log.info(s"$id : WebSocket upgraded")
    case FailedUpgrade(statusCode) =>
      log.error(s"$id : Failed to upgrade WebSocket connection : $statusCode")
      throw WindTurbineSimulatorException(id)
    case ConnectionFailure(ex) =>
      log.error(s"$id : Failed to establish WebSocket connection $ex")
      throw WindTurbineSimulatorException(id)
    case Connected =>
      log.info(s"$id : WebSocket connected")
      context.become(running)
  }

  def running: Receive = {
    case Terminated =>
      log.error(s"$id : WebSocket connection terminated")
      throw WindTurbineSimulatorException(id)
  }
}
```
The WebSocket client is implemented as an Akka Stream, therefore, the WindTurbineSimulator actor is an actor encapsulating a stream. The WebSocket client is used to stream telemetry to the service, once a second. The WebSocket stream is implemented as follows.
```scala
object WebSocketClient {  
  def apply(id: String, endpoint: String, supervisor: ActorRef)
           (implicit
            system: ActorSystem,
            materializer: ActorMaterializer,
            executionContext: ExecutionContext) = {
    new WebSocketClient(id, endpoint, supervisor)(system, materializer, executionContext)
  }
}

class WebSocketClient(id: String, endpoint: String, supervisor: ActorRef)  
                     (implicit
                      system: ActorSystem,
                      materializer: ActorMaterializer,
                      executionContext: ExecutionContext) {
  val webSocket: Flow[Message, Message, Future[WebSocketUpgradeResponse]] = {
    val websocketUri = s"$endpoint/$id"
    Http().webSocketClientFlow(WebSocketRequest(websocketUri))
  }

  val outgoing = GraphDSL.create() { implicit builder =>
    val data = WindTurbineData(id)

    val flow = builder.add {
      Source.tick(1 seconds, 1 seconds, ())
        .map(_ => TextMessage(data.getNext))
    }

    SourceShape(flow.out)
  }

  val incoming = GraphDSL.create() { implicit builder =>
    val flow = builder.add {
      Flow[Message]
        .collect {
          case TextMessage.Strict(text) =>
            Future.successful(text)
          case TextMessage.Streamed(textStream) =>
            textStream.runFold("")(_ + _)
              .flatMap(Future.successful)
        }
        .mapAsync(1)(identity)
        .map(println)
    }

    FlowShape(flow.in, flow.out)
  }

  val ((upgradeResponse, killSwitch), closed) = Source.fromGraph(outgoing)
    .viaMat(webSocket)(Keep.right) // keep the materialized Future[WebSocketUpgradeResponse]
    .viaMat(KillSwitches.single)(Keep.both) // also keep the KillSwitch
    .via(incoming)
    .toMat(Sink.ignore)(Keep.both) // also keep the Future[Done]
    .run()

  val connected =
    upgradeResponse.map { upgrade =>
      upgrade.response.status match {
        case StatusCodes.SwitchingProtocols => supervisor ! Upgraded
        case statusCode => supervisor ! FailedUpgrade(statusCode)
      }
    }

  connected.onComplete {
    case Success(_) => supervisor ! Connected
    case Failure(ex) => supervisor ! ConnectionFailure(ex)
  }

  closed.map { _ =>
    supervisor ! Terminated
  }
}
```
This implementation largely follows the webSocketClientFlow example from the Akka HTTP documentation, with the addition of actor messaging, to manage the life-cycle of the WebSocket stream. If the WebSocket connection cannot be opened, is closed, or encounters an error, it sends a message to the WindTurbineSimulator actor. For instance, when the WebSocket is connected, it sends the actor a Connected message, and if the WebSocket closes, it sends the actor a Terminated message. The actor will handle logging messages and raising exceptions, related to the life-cycle of the stream. If the actor does raise an exception, it will be restarted, and it will restart the WebSocket stream. It is important to remember that these life-cycle messages are sent from Futures and that it is not thread-safe to close over member variables of the actor. It is safe, however, to capture the self actor-reference and send the actor a message. The  KillSwitch is used in the postStop method of the actor to terminate the WebSocket stream when the actor is stopped. The KillSwitch is not used in this article, but it will become important in the next article in this series.

The JSON measurement messages are simulated in the following manner.
```scala
object WindTurbineData {  
  def apply(id: String) = new WindTurbineData(id)
}

class WindTurbineData(id: String) {  
  val random = Random

  def getNext: String = {
    val timestamp = System.currentTimeMillis / 1000
    val power = f"${random.nextDouble() * 10}%.2f"
    val rotorSpeed = f"${random.nextDouble() * 10}%.2f"
    val windSpeed = f"${random.nextDouble() * 100}%.2f"

    s"""{
       |    "id": "$id",
       |    "timestamp": $timestamp,
       |    "measurements": {
       |        "power": $power,
       |        "rotor_speed": $rotorSpeed,
       |        "wind_speed": $windSpeed
       |    }
       |}""".stripMargin
  }
}
```
To simulate a significant load on the service, it is extremely easy to write an application to create a large number of wind-turbine-simulator actors.
```scala
object SimulateWindTurbines extends App {  
  implicit val system = ActorSystem()
  implicit val materializer = ActorMaterializer()

  for (_ <- 1 to 1000) {
    val id = java.util.UUID.randomUUID.toString
    system.actorOf(WindTurbineSimulator.props(id, Config.endpoint), id)
  }
}
```
Starting a large number of WebSocket connections can be resource intensive. If I start too many WebSocket clients, some of them will timeout connecting to the service. Interestingly, the Akka Streams API can be used to conveniently throttle the creation of new connections at a targeted rate. In this case, I will throttle new connections to 100 per second. This is a much more refined approach than adding a magic Thread.sleep(100), or equivalent, to throttle connections within a for-comprehension.
```scala
object SimulateWindTurbines extends App {  
  implicit val system = ActorSystem()
  implicit val materializer = ActorMaterializer()

  Source(1 to 1000)
    .throttle(
      elements = 100,
      per = 1 second,
      maximumBurst = 100,
      mode = ThrottleMode.shaping
    )
    .map { _ =>
      val id = java.util.UUID.randomUUID.toString
      system.actorOf(WindTurbineSimulator.props(id, Config.endpoint), id)
    }
    .runWith(Sink.ignore)
}
scala
This example demonstrates the advantages of encapsulating a stream within an actor to handle the life-cycle management of the stream. The advantages of encapsulating a stream within an actor become even more apparent, however, for fault-tolerance and handling errors within the stream.

## Stream Supervision
The Akka Streams API provides a stream supervision-strategy to handle exceptions within the stream itself. When processing the current element in the stream results in an exception, there are three options: 1) complete the stream with a failure; 2) drop the element and continue processing; or 3) drop the element and continue the stream after restarting the stage, discarding any intermediate state. For a stream that parses messages, for example, if it encounters a message that cannot be parsed, it can log an error, drop the message, and continue processing.
```scala
implicit val system = ActorSystem()

val decider: Supervision.Decider = {  
  case MessageParsingException(message) =>
    logger.error(s"Unable to parse message : $message")
    Supervision.Resume
}

implicit val materializer = ActorMaterializer(  
  ActorMaterializerSettings(system)
    .withSupervisionStrategy(decider)
)

Source(messages)  
  .map(parse)
  .runWith(sink)
```
Supervision strategies for streams are limited, however, in comparison to the supervision strategies for actors. For instance, returning to the example of simulating the wind turbine, presented above, as it stands, the actor supervision-strategy is such that all of the wind-turbine-simulator actors are running directly under the user guardian-actor.

![default-wind-turbine-simulator-supervision.png](/images/default-wind-turbine-simulator-supervision.png)

If a wind-turbine simulator loses its WebSocket connection with the service, the actor will be restarted immediately, according to the default restart policy of the user guardian-actor. If the measurement collection and aggregation service is temporarily unavailable, or is under load, the connection can fail repeatedly, and the actor will be restated in a tight loop. This behaviour can overwhelm the service with a huge number of WebSocket connection attempts, not to mention pollute logs with a large number of messages, and it will not represent the behaviour of the actual client, which uses an exponential-backoff-and-retry mechanism.

To more effectively simulate the wind-turbine clients, an exponential-backoff-and-retry supervisor-actor can be used to supervise the simulated wind-turbine actor. A backoff-and-retry supervision-strategy is not a pattern offered by the Akka Streams API, but it is a pattern offered by Akka Actors.
```scala
object SimulateWindTurbines extends App {  
  implicit val system = ActorSystem()
  implicit val materializer = ActorMaterializer()

  Source(1 to 1000)
    .throttle(
      elements = 100,
      per = 1 second,
      maximumBurst = 100,
      mode = ThrottleMode.shaping
    )
    .map { _ =>
      val id = java.util.UUID.randomUUID.toString

      val supervisor = BackoffSupervisor.props(
        Backoff.onFailure(
          WindTurbineSimulator.props(id, Config.endpoint),
          childName = id,
          minBackoff = 1 second,
          maxBackoff = 30 seconds,
          randomFactor = 0.2
        ))

      system.actorOf(supervisor, name = s"$id-backoff-supervisor")
    }
    .runWith(Sink.ignore)
}
```
The actor supervision hierarchy has evolved to the following. Notice that in order to include backoff-and-retry for the WebSocket connection, I did not need to change the program radically. In fact, all that I needed to do was add one more layer or indirection, through actor supervision, I did not need to change anything else about the existing code or functionality.

![wind-turbine-simulator-backoff-supervision](/images/wind-turbine-simulator-backoff-supervision)

In this article, I demonstrated how Akka Actors compliment the Akka Stream API for fault tolerance and the life-cycle management of streams. In the next article in this series, I will extend this example to show how actors can be used to distribute streaming workloads within an Akka cluster and provide location transparency for streams.