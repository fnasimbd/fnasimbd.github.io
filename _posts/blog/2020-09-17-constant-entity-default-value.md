---
layout: post
title: "Don't Use Constants as Default Values in Entities"
modified:
categories: blog
excerpt: "An issue while using constants as initial values for entity fields and how to resolve it."
tags: ["ddd", "value-object", "orm"]
comments: true
share: true
---

Immutable value objects are a nice abstraction for many scenarios. Some value objects can take only a finite, small set of values. Maintaining static constants for all possible values of these value object is a nice idea. As a further enhancement, marking the value object constructor as private, thereby **sealing the type for instantiation,** rules out the possibility of creating a wrong value by mistake and also makes sure that **only one instance per state** is created.

{% highlight java linenos %}
public class ValueType {
    public final static ValueType One = new ValueType("One");
    public final static ValueType Two = new ValueType("Two");

    private String value;

    public String getValue() {
        return this.value;
    }

    private ValueType(String value) {
        this.value = value;
    }
}
{% endhighlight %}

Value objects might be used to represent states of an entity. In most occasions, providing initial values to such fields as constants might be fitting: upon creation, an entity starts at some initial state and transitions to other states as operations are performed on it. The initialization might happen both in field initializer or in the default constructor.

{% highlight java linenos %}
public class Entity {
    private ValueType state1;
    private ValueType state2 = ValueType.One;

    public Entity() {
        state1 = ValueType.One;
    }

    public void someOperation() {
        state1 = ValueType.Two;
        state2 = ValueType.Two;
    }
}
{% endhighlight %}

---

This approach, however, has a dire consequence if you are using an Object-Relational Mapper(ORM) to fetch persisted entities. While rehydrating entities from database, ORMs don't always respect private field access restriction. They usually create the entity instance with the default empty constructor and instrument private field values into it using reflection. Therefore, immutability of value objects is not preserved. During a rehydration, ORM initializes the value object field with the designated initial constant and then **mutates the constant** with the value fetched from database. If a query returns multiple entity instances, probably in various states, the same constant **keeps mutating** and **settles at the last---and probably wrong---value**.

The consequence is twofold. Firstly, all entities refer to the same constant instance, thereby the same and incorrect state. Secondly, the constant may end up with a wrong value in it; as a result, new entity instances might get persisted with wrong value for the corresponding field; moreover, conditions based on the constant yield wrong results too.

---

Creating new instances, instead of using constants, during field initialization might seem to be the most obvious remedy: that way, each entity has its own value object field instance and ORM mutates them with their respective values in database. That, however, requires letting create the value object with a public constructor and as a result, compromises the safety due to the sealed instantiation. Initializing the value object field with a public setter goes away with constants. But using public setters is not desirable; entities should change state with intention-revealing methods.

The most appropriate resolution, without loosing any benefits of value objects, is to create the entity in a **static factory method within the entity** from where the private setter is accessible and to initialize the value object field immediately after creation.

{% highlight java linenos %}
public class Entity {
    private ValueType state;

    public ValueType getState() {
        return this.state;
    }

    public void someOperation() {
        state = ValueType.Two;
    }

    public static createInstance() {

        Entity instance = new Entity();
        instance.state = ValueType.One; // set initial state
        return instance;
    }
}
{% endhighlight %}
