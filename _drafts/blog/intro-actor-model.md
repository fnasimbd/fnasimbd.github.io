---
layout: post
title: "What is Actor Model and Akka?"
modified:
categories: blog
excerpt: "Introduction to Actor Model architecture."
tags: ["distributed-processing", "actors", "actor-model", "reactive"]
comments: true
share: true
---

Actor Model is a new programming model, a departure from traditional method invocation-based model, particulary suited for high-concurrency environments. I discuss the following aspects of Actor Model:

- Overview of the Actor architecture.
- How to use implement basic Actors in Akka.

Overview of Actor Model Architecture
====================================

Actor Model architecture consists of **actors** capable of processing **messages**. Each actor has a **mail box** associated with it. Unlike traditional method invocation, actors pass messages to each other to accomplish tasks. Instead of waiting for method invocation to return, actors **subscribe** for messages to be returned.

Each actor **wakes up** whenever its mail box is non-empty and starts processing messages one after another. This approach eliminates the need for **synchronization locks**. As a result, leads to improved performance. The model also promotes **scalability** by allowing individual components grow without knowledge of others.

The architecture may look like anarchy at first (so many actors communicating to each other without coordination), but in practice it is highly convenient.

Besides the performance limitation, sequential method execution has its benifits too. Most importantly, they are easier to reproduce, predictable, and debug.

Actor Model is nothing radically new. In traditional systems, it is common finding part of the system implemented in Actor-inspired model. Basing entire application, however, in Actor model is pretty recent.

**Actors in UI Frameworks.**<br><br>UI frameworks employ an Actor-like architecture. The UI thread cannot be accessed by other threads; instead other threads pass messages to UI thread for processing.
{: .notice--info}

**NodeJS I/O Handling as Actor.**<br><br>NodeJS queues I/O requests from user before processing them one after another.
{: .notice--info}

Implementing Actors in Akka
===========================

Akka is a popular Actor architecture implementation for JVM platform. The following snippet demonstrates how to define an Actor.

{% highlight java linenos %}
public static class ReadResult extends AbstractActor {
    final LoggingAdapter log = Logging.getLogger(getContext().system(), this);

    @Override
    public Receive createReceive() {
        return receiveBuilder()
	        .match(ReadEventCompleted.class, m -> {
		    final Event event = m.event();
		    log.info("event: {}", event);
		    context().system().terminate();
                })
		.match(Status.Failure.class, f -> {
		    final EsException exception = (EsException) f.cause();
		    log.error(exception, exception.toString());
		    context().system().terminate();
		})
		.build();
    }
}
{% endhighlight %}

The following snippet shows how one actor passes message to another.

{% highlight java linenos %}
ActorRef connection = system.actorOf(ConnectionActor.getProps(settings));

final ActorRef readResult = system.actorOf(Props.create(ReadEventExample.ReadResult.class));

final ReadEvent readEvent = new ReadEventBuilder("newstream")
        .first()
        .resolveLinkTos(false)
        .build();

connection.tell(readEvent, readResult);
{% endhighlight %}

