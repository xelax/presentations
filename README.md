Kafka and Akka for high-performance real-time behavioral data
========================================================
Alex Cozzi
2014-11-10



Overview of the talk
========================================================

* Benefits of the queue-based architecture.
* Actor computation model.
* Listening to behavioral events.


Kafka
===========================================================
What is it: Publish-subscribe messaging rethought as a distributed commit log

![producer consumer](http://kafka.apache.org/images/producer_consumer.png)

![topic anatomy](http://kafka.apache.org/images/log_anatomy.png)
![consumer groups](http://kafka.apache.org/images/consumer-groups.png)


Actor model (Hewitt, 1973)
====================

* Send a finite number of messages to other actors
* Create a finite number of new actors
* Designate the behavior to be used for the next message it receives

Akka-Kafka Integration
=====================

https://github.com/sclasen/akka-kafka



Receive messages from Kafka
================================
Configuration
```scala
val consumerProps = AkkaConsumerProps.forContext(
      context = context,
      zkConnect = "zookeeper cluster",
      topic = "SaaS topic",
      group = hostname,
      streams = 1, 
      keyDecoder = new DefaultDecoder(),
      msgDecoder = new DefaultDecoder(),
      receiver = pulsarListener,
      commitConfig = CommitConfig())
```

Inititialization
================
```json
kafka.consumer {
  auto.offset.reset = "smallest" // default: "largest"
}
```
```scala
ZkUtils.maybeDeletePath(zk, s"/consumers/$group")

val consumer = new AkkaConsumer(consumerProps)

consumer.start()
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
