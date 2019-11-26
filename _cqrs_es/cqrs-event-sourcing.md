---
layout: post
title: "CQRS+ES: How to Implement an Event Store with EventStore"
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

CQRS divides the application into two sides: the **write model** and the **read model**. Only write models can update the domain model. For each write requests, current state of a domain model is reconstructed by **replaying** all events pertaining to that domain model instance. Read models subscribe to an event stream. With each domain event, interested read models update themselves and store the state in a secondary store (aka **projection**) for serving on user's read requests.

A convenient way for storing and retrieving events is necessary. That is where _event store_ comes in.

Events and the Event Store
==========================

An event store stores domain events as serialized BLOBs with an unique event id. Events in a event store can be queried only with event ids. Event stores are not designed for complex querying.

An application usually contains many different domain events. Before storing in an event store, however, for convenience, all domain events are converted to something like the following `StoredEvent` type. The `eventBody` contains the serialized domain event and the `typeName` field contains the fully qualified type name of the event to facilitate deserializing during read.

{% highlight java linenos %}
public class StoredEvent {

    private String eventBody;
    private long eventId;
    private Date occurredOn;
    private String typeName;

    // ... various constructors, methods, and properties
}
{% endhighlight %}

An event store interface follows the following basic format.

{% highlight java linenos %}
public interface EventStore {

    public void appendWith(EventStreamId aStartingIdentity, List<DomainEvent> anEvents);

    public void close();

    public List<DispatchableDomainEvent> eventsSince(long aLastReceivedEvent);

    public EventStream eventStreamSince(EventStreamId anIdentity);

    public EventStream fullEventStreamFor(EventStreamId anIdentity);

    public void purge(); // mainly used for testing

    public void registerEventNotifiable(EventNotifiable anEventNotifiable);
}
{% endhighlight %}

Vaughn Vernon implements event store in MySQL and that is sufficient for basic usage. That approach has several limitations for practical use.

_EventStore_ as an Event Store
==============================

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

Reading Events
--------------

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

Subscribing to Domain Events
----------------------------

For a single event stream, it is simple; because stream is consistent after each transaction.

In EventStore, we have to subscribe to the changes taking place in a projection.

**Commutativity and Associativity of Reduce and the Resultant Optimization**<br><br>As an optional optimization, where the reduce function is both _commutative_ and _associative_, user can submit a _combiner function_ (usually the reduce function itself) that eliminates duplicates before invoking reduce, thus reducing bandwidth usage.
{: .notice--info}
