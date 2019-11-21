---
layout: post
title: "CQRS+ES: How to Implement an Event Store with EventStore"
modified:
excerpt: "Introduction to implementing an Event Sourcing event store with [EventStore](https://eventstore.org)."
tags: ["map-reduce", "map", "reduce", "distributed-processing", "ravendb", "hadoop", "mapreduce"]
comments: true
share: true
---

Event Sourcing is becoming inreasingly popular.

- Characteristics of an Event Sourcing event store.
- How to implement an event store in Java with EventStore.

{% include toc %}

What is an Event Store
======================

An event store contains events in serialized BLOBs and their type (in order to facilitate deserializing) with an event id. Events in a event store can be queried only with event ids. Event stores are not designed for complex querying.

As a representative example, here follows how map and reduce functions are implemented in Java. Outside some syntactic differences, other languages implement map and reduce more or less the same way. The following example maps an input array to its squares.

{% highlight java linenos %}
int[] arr = new int[] { 1, 2, 3, 4, 5 };

List<int> res = arr.stream().map((value) -> {
    return value * value;
}).collect(Collectors.toList());

// res: [ 1, 4, 9, 16, 25 ]
{% endhighlight %}

The following example reduces an input array to the product of its elements.

{% highlight java linenos %}
int[] arr = new int[] { 1, 2, 3, 4 };

int res = arr.stream().reduce(1, (pre, cur) -> {
    return pre * cur;
});

// res: 24
{% endhighlight %}

The _Map-Reduce Pattern_
========================

Map and reduce together give an elegant abstraction for various computations over collections, called **map-reduce pattern**; in this pattern, a task is divided into two phases: **map phase** and **reduce phase**. During the map phase, the **source collection** is mapped to an **intermediate collection** and during the following reduce phase, the intermediate collection is **grouped** by some criterion and each group is reduced to some **aggregate result**. It may sound bit restrictive but in practice, map-reduce pattern is, in fact, capable of highly versatile computations.

Careful look reveals that though map and reduce's input functions define the actual computation, they neither _make direct references to the collection_ nor make any _assumption on the collection's length_ (define the computation once, it applies to collections _anywhere_ and of _any length_.) These properties have two consequences: firstly, user may **pass computations**---defined in map-reduce pattern---for data whose location is unknown to them and secondly, computation may be executed on lists of arbitrary length and split into multiple independent partitions, making computation easily **scalable**. The next section shows how a database system exploits the first facility and later we will see how MapReduce exploits both.

_Map-Reduce Index_ in RavenDB
=============================================
The index and query operation of [RavenDB](https://ravendb.net/docs/article-page/4.0/csharp), the NoSQL document database, is a compelling real-world use of map-reduce pattern simplifying a complex job. Before going into its details, a very brief overview of how RavenDB and its index and query operation works is necessary: RavenDB organizes **documents** into **collections** (for simplicity, you may view documents as objects with a mandatory unique id field and collections as list of all objects in the database of a certain object type); each time a new document is added to the database, it is appended to its corresponding collection if the collection already exists, or a new collection is created for it.

Unlike relational databases, RavenDB _pre-computes_ query results into **indexes**, thereby achieving faster query performance. Each query on a collection corresponds to an index; indexes consists of key-value **index entry** pairs where keys correspond to all possible parameter values found in the collection for that query and values correspond to data matching the parameter value. Let's assume we have a collection called `Persons` with fields `Name`, `City`, `Country`, and maybe others (see snippet below), and we want to find all persons from a certain city; in order to achieve this, we create an index for the `City` field on `Persons` collection where index entries are city names and list of all `Person` ids from that city. Now querying the city index with a city name returns list of all person ids from that city. Indexes are created by a RavenDB **client** and [deployed](https://ravendb.net/docs/article-page/4.1/csharp/indexes/creating-and-deploying) to the RavenDB **server**. Once an index is deployed, it is updated by server each time a new document is added to its collection.

{% highlight csharp linenos %}
public class Person
{
    public string Id { get; set; }
    public string Name { get; set; }
    public string City { get; set; }
    public string Country { get; set; }
}
{% endhighlight %}

Creating indexes for field values, like `City`, is straightforward: client just needs to specify the parameter field name and the field to be stored. Now, what if we want to index results of some aggregate operation over a collection? For example, the _count of persons from a certain city_ in the `Persons` collection? It is clear that for each city, we need to store its person count as values into the index (instead of list of person ids, as we did in the last example) so that upon querying with a city name, count of the persons from that city is returned. Now, how does client _define a complex operation like counting_? Conceptually, it can be modelled in map-reduce pattern: during the map phase, map each person to a $$\langle city, count \rangle$$ pair with $$count = 1$$ (by setting $$count \leftarrow 1$$, we _pretend to have a collection with only one person, the aggregate count for the corresponding city being 1_); then group the resultant $$\langle city, 1 \rangle$$ pairs by $$city$$ and reduce each group---that is $$city$$---to a single pair containing the _sum of all counts in that group_. Once reduce phase finishes, for each city, we get a $$\langle city, count \rangle$$ pair containing the total person count. Now storing the $$\langle city, count \rangle$$ pairs as index entries does the job.

RavenDB supports the conceptual framework outlined earlier [out-of-the-box](https://ravendb.net/docs/article-page/4.0/csharp/indexes/map-reduce-indexes): client simply needs to pass their custom map and reduce functions as overriden [template methods](https://sourcemaking.com/design_patterns/template_method) of an index creation class. The map function implements the logic for mapping each document to the aggregate result for itself (the $$\langle city, 1 \rangle$$ pairs); the reduce function, in turn, implements the logic for reducing the same aggregate result for each group ($$\langle city, count \rangle$$) from the intermediate results produced by map. RavenDB server internally invokes the map and reduce functions whenever needed in appropriate order with appropritate parameters and stores the output into index. This pattern is extraordinarily capable of implementing versatile computations in a compact, portable format.

Here follows sample implementation of a map-reduce index for count of `Person` from each city in the `Persons` collection. It assumes little familiarity with C# and its LINQ library. You may, however, skip to the next section without loosing the essence.

{% highlight csharp linenos %}
public class Persons_Count_ByCity : 
    AbstractIndexCreationTask<Person, Persons_Count_ByCity.Result>
{
    public class Result
    {
        public string City { get; set; }

        public int PersonCount { get; set; }
    }

    public Persons_Count_ByCity()
    {
        Map = persons => from person in persons
                         let cityName = LoadDocument<Person>(person.City).Name
                         select new
                         {
                             City = cityName ,
                             PersonCount = 1
                         };

        Reduce = results => from result in results
                            group result by result.City into g
                            let personCount = g.Sum(x => x.PersonCount)
                            select new
                            {
                                City = g.Key,
                                PersonCount = personCount
                            };
    }
}
{% endhighlight %}

The `AbstractIndexCreationTask` class implements details of index creation and serves as base class for indexes. Notice that in the index's constructor, user passes the map and reduce functions by overriding `Map` and `Reduce` properties respectively. `Map` iterates over the entire `Persons` collection and maps each `Person` document to a `Result` instance containing the person count as 1 (the $$\langle city, 1 \rangle$$ pair); `Reduce` iterates over the intermediate `Result` collection produced by Map, groups them by `City`, and produces one `Result` instance containing the aggregate count (sum of all `Result` with the same `City` name) for each city. Once the index is deployed, for each query on it with a city specified, the index returns a `Result` object containing the sum of all person from that city.

The _MapReduce_ Distributed Computation Model
=============================================

The RavenDB Map-Reduce Indexing shows how map-reduce pattern enables user to pass a specific computation---indexing---for remote data, stored in server. That capability can be extended for computations in general and for remote data split into multiple partitions. Let's assume that we have a large chunk of text documents and we want to count occurrence of all words in them---practically intractable for a single computer---and you also have a cluster of computing nodes at your disposal. In these situations, where data size is mammoth and cluster of computers is available, [_MapReduce_](https://static.googleusercontent.com/media/research.google.com/en//archive/mapreduce-osdi04.pdf) framework comes in. It is a model of distributed computation, proposed by Google researchers Jeffrey Dean and Sanjay Ghemawat in 2004.

Used defines a MapReduce job in map-reduce pattern as a **user program** that is capable of both map and reduce tasks and job coordination as well. As the program starts, prior to map or reduce, input documents are split into number of partitions. Afterwards, copies of the user program starts simultaneously in multiple cluster nodes following master-worker topology: one copy of the user program is designated as **master**, which coordinates a job's execution until its end, and others being **workers** under it. The master assigns each partition to a worker node for mapping in parallel. In a mapper node, the map function receives documents, iterates through their contents, maps them to suitable intermediate key-value result pairs, and writes the result pairs in the mapper node's local disk. Once a worker node finishes mapping, master node is notified and it forwards locations of the intermediate outputs to a worker node for reducing. Before reducing, intermediate outputs are grouped together by their keys---by sorting the keys, then reduce function reduces each group to the final result key-value pairs again in parallel. Once master node receives finish notification from all mapper-reducer nodes, it returns control to the initial call. Usually reduce outputs are kept in place and successive MapReduce jobs are run on them to reach some final result.

For our word counting example, map function would iterate over the tokenized words in a document and for each word, produce a $$\langle word, 1 \rangle$$ pair, meaning one occurrence of $$word$$ is found. Before reduction, the intermediate output pairs would be grouped by $$word$$. Reduce function would add up all 1's for a word (e.g. $$\langle word_1, 1,1,1 \rangle$$), thereby producing that word's total occurrence (e.g. $$\langle word_1, 3 \rangle$$). Compare the process with that of creating Map-Reduce Index in RavenDB for similarity.

Specifics of MapReduce user programs vary from implementation to implementation. However, a MapReduce framework usually provides core library targeting a platform (e.g. C++, Java, etc.) with MapReduce facilities: map-reduce interfaces, data partitioning, scheduling, storing-retrieving data, error control, job coordination, etc. User programs are written extending the core library.

**Commutativity and Associativity of Reduce and the Resultant Optimization**<br><br>As an optional optimization, where the reduce function is both _commutative_ and _associative_, user can submit a _combiner function_ (usually the reduce function itself) that eliminates duplicates before invoking reduce, thus reducing bandwidth usage; commutativity and assoicativity property ensures that reduce function can be called in any order hence applying reduce function once at the mapping sites, before the actual call, produces the same result.
{: .notice--info}

MapReduce in _Apache Hadoop_
-------------------------------------

Apache Hadoop is an open-source ecosystem providing infrastructure for distributed storage and computation. Hadoop's MapReduce, targeted for JVM platform, is perhaps the most popular implementation of MapReduce. It lets user write user programs in Java, packaged as JARs, concurrency safe data structures to store the computation results, etc. As a motivation, the following snippet, written using MapReduce's Java API (it has APIs for other languages too), shows how the map and reduce functions for a job that counts the occurrence of words in documents may look like.

{% highlight java linenos %}
public static class TokenizerMapper extends Mapper<Object, Text, Text, IntWritable> {
    private final static IntWritable one = new IntWritable(1);
    private Text word = new Text();

    public void map(Object key, Text value, Context context)
            throws IOException, InterruptedException {
        StringTokenizer itr = new StringTokenizer(value.toString());

        while (itr.hasMoreTokens()) {
            word.set(itr.nextToken());
            context.write(word, one);
        }
    }
}

public static class IntSumReducer extends Reducer<Text, IntWritable, Text, IntWritable> {
    private IntWritable result = new IntWritable();

    public void reduce(Text key, Iterable<IntWritable> values, Context context)
            throws IOException, InterruptedException {
        int sum = 0;

        for (IntWritable val : values) {
            sum += val.get();
        }

        result.set(sum);
        context.write(key, result);
    }
}
{% endhighlight %}

As parameters, the `map` function receives a document name as `key` and its content as `value`; it tokenizes the document's content into words, iterates over the words, and writes `one` for each word. The `reduce` function, on the other hand, receives a word as `key` and list of all its counts, produced by map, as `values`; it sums up all the counts (`ones`) and writes them out.

**MapReduce and _Apache Spark_**<br><br>The more recent [Apache Spark](https://spark.apache.org/) framework for distributed processing also supports [map](https://spark.apache.org/docs/2.2.0/rdd-programming-guide.html#transformations) and [reduce](https://spark.apache.org/docs/2.2.0/rdd-programming-guide.html#actions) tasks among many others. Spark, however, replaces MapReduce's file system-based, inefficient I/O operations with a more capable data structure called _Resilient Distributed Datasets (RDD)_.
{: .notice--info}

MapReduce in Multi-Core, Shared-Memory Environment
--------------------------------------------------

Though MapReduce is targeted for large-scale distributed data processing, the same model can be employed even in multi-core, shared-memory environment. In this case, multiple threads are employed instead of multiple independent map-reduce nodes, thereby eliminating need for coordination among nodes and replacing communication over network with faster memory read-writes. .NET Framework's [_Parallel LINQ (PLINQ)_](https://docs.microsoft.com/en-us/dotnet/standard/parallel-programming/parallel-linq-plinq) library provides parallel equivalents to LINQ methods for processing collections; user may write their own custom MapReduce extension method like the following on top of them:

{% highlight csharp linenos %}
public static ParallelQuery<TResult> MapReduce<TSource, TMapped, TKey, TResult>(
    this ParallelQuery<TSource> source, Func<TSource, IEnumerable<TMapped>> map, 
    Func<TMapped, TKey> keySelector, 
    Func<IGrouping<TKey, TMapped>, IEnumerable<TResult>> reduce)
{
    return source.SelectMany(map).GroupBy(keySelector).SelectMany(reduce);
}
{% endhighlight %}

The following snippet counts words in some files in a directory with the preceding PLINQ `MapReduce` method.

{% highlight csharp linenos %}
var files = Directory.EnumerateFiles(dirPath, "*.txt").AsParallel();
var counts = files.MapReduce(
    path => File.ReadLines(path).SelectMany(line => line.Split(delimiters)), 
    word => word, 
    group => new[] { new KeyValuePair<string, int>(group.Key, group.Count()) });
{% endhighlight %}

