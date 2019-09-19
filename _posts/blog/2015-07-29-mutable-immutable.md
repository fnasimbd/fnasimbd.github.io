---
layout: post
title: "Mutable and Immutable Objects"
redirect_from:
  - /blog/mutable-immutable
  - /blog/mutable-immutable/
modified:
categories: blog
excerpt: "Difference between mutable and immutable objects."
tags: ["object-oriented-programming", "random-notes"]
comments: true
date: 2015-07-29T02:58:00-05
---

One of the earliest object oriented programming concepts I had trouble with for long time. The anomaly arises from the naming: I couldn't associate the names *mutable* and *immutable* to something reasonable. Sometimes it seemed they relate to *mutuality*, sometimes to *muting*, and so on. Moreover, briefly I often confused *mutable* with *mutex*. Later came to realize they have rather more to do with *mutation*. Yes, mutation from biology. Wikipedia says

> In biology, a mutation is a permanent change of the nucleotide sequence of the genome of an organism, virus, or extrachromosomal DNA or other genetic elements.

In the same manner, in OOP, *mutable objects are on which mutation or modification is possible* and *immutable objects---on the other hand---are on which mutation is not posssible*. To put more simply, attributes of a mutable object may change after its creation, but an immutable object's attributes cannot change once it is created. This mutation connection makes meaning of the terminology more obvious.

A very obvious example of immutable and mutable classes are the `String` and `StringBuilder`(also called *mutable string*) classes respectively in Java, C#, and some other languages.

An assumption on an immutable object remains true throughout its lifetime; a very crucial property in many situations. Immutable objects are intrinsically thread-safe.

And the downside is that you may have to sacrifice performance for immutability: difference between the `String` and `StringBuilder` concatenation methods illustrate the fact.

### External Links
1. [Mutable and Immutable Objects](http://www.javaranch.com/journal/2003/04/immutable.htm) tutorial at *Java Ranch* by David O'Meara.
2. [C# StringBuilder source code](http://referencesource.microsoft.com/#mscorlib/system/text/stringbuilder.cs,adf60ee46ebd299f) at *Microsoft Reference Source*.

