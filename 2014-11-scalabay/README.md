Kafka and Akka for high-performance real-time behavioral data
===========================================================
by Alex Cozzi, 2014-11-10



##Overview of the talk

* Benefits of the queue-based architecture.
* Actor computation model.
* Listening to behavioral events.


Kafka
===========================================================
What is it: Publish-subscribe messaging rethought as a distributed commit log

![producer consumer](http://kafka.apache.org/images/producer_consumer.png)

![topic anatomy](http://kafka.apache.org/images/log_anatomy.png)

![consumer groups](http://kafka.apache.org/images/consumer-groups.png)


Akka, an implementation of the Actor model (Hewitt, 1973)
====================
A deceptively simple computational model

An actor can:
* Send a finite number of messages to other actors
* Create a finite number of new actors
* Designate the behavior to be used for the next message it receives


Akka-Kafka Integration
=====================
Receive messages from Kafka using https://github.com/sclasen/akka-kafka

```scala
val consumerProps = AkkaConsumerProps.forContext(
      context = context,
      zkConnect = "zookeeper cluster",
      topic = "topic name",
      group = "my listener group",
      streams = 1, 
      keyDecoder = new DefaultDecoder(),
      msgDecoder = new DefaultDecoder(),
      receiver = pulsarListener,
      commitConfig = CommitConfig())
```

Process messages
===============
Message parsing and throttling

```scala
class PulsarListener extends Actor {
  def receive = {
    case b: Array[Byte] =>
        val event = PulsarDecoder.decode(b)
        ... do something ...
        sender ! StreamFSM.Processed
  }
}
```


REST API
==========
How to create a stream of server-side Events (SSE) using [spray](http://spray.io).

we delegate the stream handling to another actor
```scala
class PscraperSvc extends RouteDefinition {
  def route =
    path("events") {
      get { ctx =>
        context.actorOf(Props(classOf[Mediator], ctx)
      }
    }
}
```

Mediator (simplified)
========

```scala
class Mediator(ctx: RequestContext) extends Actor {
  context.actorSelection("/user/pscraper/pulsar") ! Register(self)

  ctx.responder ! ChunkedResponseStart(HttpResponse(entity = HttpEntity(`text/event-stream`, "event: start\n")))

  def toSSE(msg: String) = "event: message\ndata: " + msg.replace("\n", "\ndata: ") + "\n\n"

  def receive = {
    case msg: SaasPulsarEvent =>
      ctx.responder ! MessageChunk(toSSE(msg))

    case ev: Http.ConnectionClosed =>
      context.actorSelection("/user/pscraper/pulsar") ! Deregister(self)
      self ! PoisonPill
 }
}
```


Mediator, with throttling
========
Actor as a state machine
```scala
class Mediator(ctx: RequestContext) {
  context.actorSelection("/user/pscraper/pulsar") ! Register(self)

  val responseStart = HttpResponse(entity = HttpEntity(`text/event-stream`, "event: start\n"))

  ctx.responder ! ChunkedResponseStart(responseStart)

  def toSSE(msg: String) = "event: message\ndata: " + msg.replace("\n", "\ndata: ") + "\n\n"

  def receive = {
    case msg: SaasPulsarEvent =>
      ctx.responder ! MessageChunk(toSSE(eventMapper(msg))).withAck(ChunkSent)
      // switch to skipping mode until we get confirmation that the chunck has been received. 
      context.become(skipping)

    case ev: Http.ConnectionClosed =>
      context.actorSelection("/user/pscraper/pulsar") ! Deregister(self)
      self ! PoisonPill
      context.become(skipping)
  }

  def skipping: Actor.Receive = {
    case ChunkSent => context.unbecome()
    case msg: SaasPulsarEvent =>  // do nothing
    case ev: Http.ConnectionClosed =>
      context.actorSelection("/user/pscraper/pulsar") ! Deregister(self)
      self ! PoisonPill
  }
}
```

Distributiong messages: The [Akka Event Bus](http://doc.akka.io/docs/akka/snapshot/scala/event-bus.html)
=================
```scala
  // a message bus to deliver the messages to all interested mediators
  val messageBus = new ScanningBusImpl()

  def commonBehavior: Actor.Receive = {
    case Register(listener, classifier) =>
      messageBus.subscribe(listener, classifier)

    case Deregister(listener) =>
      messageBus.unsubscribe(listener)
  }

  def receive = commonBehavior orElse {
  }
  
```

Actor as State machines (2)
=========================
how to maintain state

```
class PulsarListener extends Actor {
  var kafkaConsumer : AkkaConsumer[Array[Byte], Array[Byte]]] = null
  
  def receive =  {
    case StartListening =>
      kafkaConsumer = createConsumer()
      context.become(listening)
  }

  def listening: Actor.Receive =  {
    case b: Array[Byte] =>
        val event = PulsarDecoder.decode(b)
        sender ! StreamFSM.Processed
    
    case StopListening =>
      kakfaConsumer = null
      context.unbecome()
  }      
}
```

But this is ugly! And we do not want `null`s in Scala.
Instead using argument passing:

```
class PulsarListener extends Actor {

  def receive =  {
    case StartListening =>
      context.become(listening(createConsumer(context)))
  }

  def listening(consumer: AkkaConsumer[Array[Byte], Array[Byte]]]): Actor.Receive =  {
    case b: Array[Byte] =>
        val event = PulsarDecoder.decode(b)
        sender ! StreamFSM.Processed
    
    case StopListening =>
      context.unbecome()
  }      
}
```

Behavior stacking
======================
```scala
class PulsarListener extends Actor {
  // a message bus to deliver the messages to all interested mediators
  val messageBus = new ScanningBusImpl()

  def commonBehavior: Actor.Receive = {
    case Register(listener, classifier) =>
      messageBus.subscribe(listener, classifier)

    case Deregister(listener) =>
      messageBus.unsubscribe(listener)
  }

  def receive = commonBehavior orElse {
    case StartListening(pool, streams) =>
      sender ! s"Started Listening to $pool"
      context.become(listening(Map(pool -> createConsumer(context, self, pool, streams))), discardOld = true)

    case StopListening(pool) => sender ! s"$pool already stopped"
  }

  def listening(consumers: Map[String, AkkaConsumer[Array[Byte], Array[Byte]]]): Actor.Receive = commonBehavior orElse {
    case b: Array[Byte] =>
        val event = PulsarDecoder.decode(b)
        sender ! StreamFSM.Processed

    case StartListening(pool, streams) =>
      if (consumers.contains(pool))
        sender ! s"Already Listening to $pool:  ${consumers.keySet}"
      else {
        val newConsumers = consumers + (pool -> createConsumer(context, self, pool, streams))
        sender ! s"Now Listening to ${newConsumers.keySet}"
        context.become(listening(newConsumers), discardOld = true)
      }

    case StopListening(pool) =>
      if (consumers.contains(pool)) {
        consumers(pool).stop()
        val newConsumers = consumers - pool

        if (newConsumers.nonEmpty) {
          sender ! s"Now Listening to ${newConsumers.keySet}"
          context.become(listening(newConsumers), discardOld = true)
        } else {
          sender ! s"Stopped Listening to $pool. Not listening to anything: ${newConsumers.keySet}"
          context.become(receive, discardOld = true)
        }
      } else {
        sender ! s"No such pool. Currently listening to: ${consumers.keySet}"
      }
  }
}
```

Event Bus Implementation
============================

```scala
class ScanningBusImpl extends ActorEventBus with ScanningClassification with PredicateClassifier {
  type Event = SaasPulsarEvent

  // is needed for determining matching classifiers and storing them in an
  // ordered collection
  override protected def compareClassifiers(a: Classifier, b: Classifier): Int = a.hashCode().compareTo(b.hashCode())

  // determines whether a given classifier shall match a given event; it is invoked
  // for each subscription for all received events, hence the name of the classifier
  override protected def matches(classifier: Classifier, event: Event): Boolean = classifier(event)

  // will be invoked for each event for all subscribers which registered themselves
  // for a classifier matching this event
  override protected def publish(event: Event, subscriber: Subscriber): Unit = {
    subscriber ! event
  }
}
```

Instrumentation
===============
How to measure the performance of your Akka app?

I used the scala wrapper for (Coda Hale metrics)[https://github.com/erikvanoosten/metrics-scala], but I also found (kamon)[http://kamon.io] very intersting.

Be careful of the cost of logging!
```scala
trait Instrumented extends InstrumentedBuilder with FutureMetrics {
  val metricRegistry = PulsarListener.metricRegistry
}

class PulsarListener extends Actor with ActorLogging with Instrumented {
  private[this] val messages = metrics.meter("kafka-messages")
  private[this] val messageLag = metrics.histogram("message-lag")
  private var lastLag = 0L
  private var lastTimestamp = 0L
  private var lastMessage = SaasPulsarEvent()

  def receive {
    case Status => 
      sender ! s"listening to ${consumers.keySet}. ${messages.oneMinuteRate} messages/sec in the last minute, current lag: $lastLag mean ${messageLag.mean / 1000.0} lag in seconds, timestamp ${new java.util.Date(lastTimestamp)}\n"

   case b: Array[Byte] =>
      messages.mark()
      val event = PulsarDecoder.decode(b)
      lastMessage = event
      lastTimestamp = event.timestamp
      lastLag = (System.currentTimeMillis() - lastTimestamp)
      messageLag += lastLag
      messageBus.publish(event)
      sender ! StreamFSM.Processed
  }
}
```

Recovery and Performance Testing
================
How do you recover from system crashes? 
How fast can you go?
```
kafka.consumer {
  auto.offset.reset = "smallest"  # default: "largest"
}
```

```scala
ZkUtils.maybeDeletePath(zk, s"/consumers/$group")

val consumer = new AkkaConsumer(consumerProps)

consumer.start()
```



Performance
========================

configuration    | messages/sec
-----------------|-------------
spray json 1.2.6 | 1K
spray json 1.3.0 | 22K 
lazy parsing     | 34K
play json        | 36K 


Conclusions
=======================
* Queue based architectures are simple, performant.
* Recovery from crashes is seamless.
* Easy performance testing.
* Akka is an excellent match for Kafka processing.


Next Steps
====================
* Look carefully at [reactive streams](http://www.reactive-streams.org)
* Statistical sketches [
