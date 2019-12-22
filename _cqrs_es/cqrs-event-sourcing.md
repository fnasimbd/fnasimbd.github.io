---
layout: post
title: "CQRS+ES: EventStore as Event Store"
modified:
excerpt: "Introduction to implementing an Event Sourcing event store with [EventStore](https://eventstore.org)."
tags: ["event-sourcing", "cqrs", "eventstore"]
comments: true
share: true
---

Event Sourcing and CQRS are becoming increasingly popular, mainly among the DDD community. Understanding and implementing an event store can be a suitable first step towards adopting CQRS and Event Sourcing.

* Purpose and characteristics of an event store.
* How to implement an event store in Java with EventStore.

{% include toc %}

Quick Recap of CQRS and Event Sourcing
======================================

Event Sourcing is a departure from the traditional approach of storing and retrieving domain model state. Instead of storing domain model's current state, the entire sequence of events leading to the current state is preserved and whenever needed, currrent state is reconstructed from the events.

CQRS divides the application into two sides: the **write model** and the **read model**. Only write models can update the domain model. For each write requests, current state of a domain model is reconstructed by **replaying** all events pertaining to that domain model instance and the desired change is appended as a new event. Read models subscribe to the event stream corresponding to the domain model. With each domain event, interested read models update themselves and store the state in a secondary store (aka **projection**) for serving on user's read requests.

A convenient way for storing and retrieving events is warranted. That is where **event store** comes in.

Events and the Event Store
==========================

An event store stores domain events as serialized BLOBs with an unique event id. Events in a event store can be queried only with event ids. 

An application usually consists of many different domain events. Before storing in an event store, however, for convenience, all domain events are converted to something like the following `StoredEvent` type. The `eventBody` contains the serialized domain event and the `typeName` field contains the fully qualified type name of the event to facilitate deserializing during read.

{% highlight java linenos %}
public class StoredEvent {

    private String eventBody;
    private long eventId;
    private Date occurredOn;
    private String typeName;

    // ... various constructors, methods, and properties
}
{% endhighlight %}

An event store interface supports the following basic operations.

{% highlight java linenos %}
public interface EventStore {

    void appendWith(EventStreamId aStartingIdentity, List<DomainEvent> anEvents);

    List<DispatchableDomainEvent> eventsSince(long aLastReceivedEvent);

    EventStream eventStreamSince(EventStreamId anIdentity);

    EventStream fullEventStreamFor(EventStreamId anIdentity);

    void registerEventNotifiable(EventNotifiable anEventNotifiable);
}
{% endhighlight %}

**Event Stream and Querying**<br><br>Notice that none of the `EventStore` interface methods implement any complex parameterized queries. Event stores are not designed for complex querying. They are meant to serve the write or command side of a CQRS application.<br><br>Stream projections are there to support that.
{: .notice--info}

_EventStore_ as an Event Store
==============================

<s>Vaughn Vernon implements event store in MySQL and that is sufficient for basic usage. That approach has several limitations for practical use.</s>

EventStore---confusing namesake---is a battle-tested _append-only_ log of _immutable events_, purpose-built for event sourcing, designed by a team lead by long-time CQRS proponent Greg Young. EventStore is written in C#, with .NET and JVM client wrappers for its REST API. I will be using the JVM one.

Events are stored in various **streams**. Each event in a stream is assigned a unique, random **event id** and an incremental integer **event number**.

The EventStore JVM Client
-------------------------

The EventStore JVM client is actor-based.

Connecting to EventStore
------------------------

{% highlight java linenos %}
var credentials = new UserCredentials("admin", "changeit");
var system = ActorSystem.create();
var settings = new SettingsBuilder()
        .address(new InetSocketAddress("localhost", 1113))
        .defaultCredentials(credentials().login(), credentials().password())
        .build();

connection = EsConnectionFactory.create(system, settings);
{% endhighlight %}

A CQRS+ES application usually establishes an EventStore connection on application startup and unlike a relational database connection, retains the connection for the entire application lifecycle.

With the event store connection established, application is ready to serve user requests. For a command request, event-sourced application's write model starts by reading an aggregate root and then appending new events to it if necessary.

Reading Events
--------------

The following snippet shows how to read events from an event stream asynchronously.

{% highlight java linenos %}
Future<ReadStreamEventsCompleted> res = connection.readStreamEventsForward(
                    streamIdentity.streamName(),
                    new EventNumber.Exact(startEventNumber),
                    100,
                    true,
                    credentials());

List<DispatchableDomainEvent> dispatchableEvents = new ArrayList<>();

ReadStreamEventsCompleted r = Await.result(res, Duration.create(10, TimeUnit.SECONDS));
var events = r.eventsJava();
{% endhighlight %}

Once events are read, they must be deserialized to Java classes.

{% highlight java linenos %}
for (var event : events) {
    String jsonString = event.data().data().value().decodeString("UTF-8");

    var storedEvent = this.eventSerializer().
            deserializeStoredEvent(jsonString, StoredEvent.class);

    Class<DomainEvent> eventClass = (Class<DomainEvent>) Class.forName(storedEvent.typeName());

    var eventDeserialized = this.eventSerializer().
            deserialize(storedEvent.eventBody(), eventClass);

//    dispatchableEvents.add(
//            new DispatchableDomainEvent(event.record().number().value(),
//                    eventDeserialized));
}
{% endhighlight %}

Writing Events
--------------

{% highlight java linenos %}
Collection<EventData> writeEvents = new ArrayList<>();

for (var event : anEvents) {
    var eventData = createEventData(aStartingIdentity, event);
    writeEvents.add(eventData);
}

connection.writeEvents(aStartingIdentity.streamName(), null, writeEvents, credentials());
{% endhighlight %}

**Event Stream Strategy**<br><br>A carefully defined event stream strategy is important for an event-sourced application. It decides whether there should be one stream per application, per aggregate, per aggregate id, etc. This decision depends on a number of factors.
{: .notice--info}

Subscribing to Domain Events
----------------------------

For a single event stream, it is simple; because stream is consistent after each transaction.

In EventStore, we have to subscribe to the changes taking place in a projection.
