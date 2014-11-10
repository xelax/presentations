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

Akka, an implementation of the Actor model (Hewitt, 1973)
====================
A deceptively simple computational model

An actor can:
* Send a finite number of messages to other actors
* Create a finite number of new actors
* Designate the behavior to be used for the next message it receives


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
Server-side Events

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

Mediator
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


Mediator (2)
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

Message Bus
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

Message Bus Implementation
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

Recovery and Performance Testing
================
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
* Queue based architecture are simple and performant
* No worries about recovery
* Easy performance testing 
* Akka is an excellent match for Kafka processing
