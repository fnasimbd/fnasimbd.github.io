---
layout: post
title: "Introduction to Map-Reduce Pattern"
modified:
categories: articles
excerpt: "Introduction to map and reduce functions, how they lead to _map-reduce pattern_ and its elegant real-world application, namely in _RavenDB_ and _MapReduce_."
tags: ["map-reduce", "map", "reduce", "distributed-processing", "ravendb", "hadoop", "mapreduce"]
comments: true
share: true
---

_Map_ and _Reduce_ functions, initially pertaining to functional languages like _Lisp_, later adopted by many others, greatly simplify computations on lists. In this write-up, I explore the following aspects of map and reduce functions:
- Their definitions, usage, and properties.
- How they can be used together to model seemingly complex computations, with a real-world application.
- How they inspire a general-purpose, scalable distributed computation framework, namely _MapReduce_.

{% include toc %}

What are Map and Reduce Functions
=================================

Map and reduce are two convenience functions for simplifying two specific list operations. Both operates on a list with a user-defined **input function**. The input function for map is _single-parameter_ and of _non-void return type_. On invocation on a list, map iterates over the list from left to right, and while doing so, applies the input function on each element, thereby producing a new value for each element. Map keeps appending the new value produced to a new list, and returns that new list, its length being the same as the input list, at the end (**maps the list** to a new one). List _transformation_ operations can easily be viewed as map operations.

Reduce, on the other hand, takes a _two-parameter_ and _non-void return type_ function as input. Reduce starts with a **partial result** value (usally the first element in the list, but user may explicity specify it too) and, like map, iterates over the list from left to right. Reduce applies the input function on each list element and the current partial result and replaces the partial result with the value returned, making it the new partial result. That way, reduce keeps updating the partial result and returns it at the end (**reduces the list** to a value.) List _aggregate_ operations (e.g. sum, average, max-min, etc.) can be viewed as reduce operations.

As a representative example, here follows how map and reduce functions work in Java. Outside some syntactic differences, they work more or less the same way in other languages. The following example maps an input array to its squares.

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

Map and reduce together give an elegant abstraction for various computations over collections, called the **map-reduce pattern**. In this pattern, a task is divided into two phases: **map phase** and **reduce phase**. During the map phase, the **source collection** is mapped to an **intermediate collection** and during the following reduce phase, the intermediate collection is **grouped** by some criterion and each group is reduced to some **aggregate result**. Depending on the computation problem, multiple successive mappings might be necessary before reducing. It may sound bit restrictive but in practice map-reduce pattern is capable of modeling diverse kind of computations.

Careful look reveals that though map and reduce's input functions define the actual computation, they neither _make direct references to the collection_ nor make any _assumption on the collection's length_ (define the computation once, it applies to collections _anywhere_ and of _any length_.) These properties have two consequences: firstly, user may **pass computations**---defined in map-reduce pattern---for data whose location is unknown to them and secondly, computation may be executed on lists of arbitrary length and split into multiple independent partitions, making computation easily **scalable**. The next section shows how a database system exploits the first facility and later we will see how the MapReduce exploits both.

_Map-Reduce Index_ in RavenDB
=============================================
The index and query operation of [RavenDB](https://ravendb.net/docs/article-page/4.0/csharp), the NoSQL document database, is a compelling real-world use of map-reduce pattern simplifying a complex job. Before going into its details, a very brief overview of how RavenDB and its index and query operation works is necessary: RavenDB organizes **documents** into **collections** (for simplicity, you may view documents as objects with a mandatory unique id field and collections as list of all objects in the database of a certain object type); each time a new document is added to the database, it is appended to its corresponding collection if the collection already exists, or a new collection is created for it.

Unlike relational databases, RavenDB _pre-computes query results_ into **indexes**, thereby achieving faster query performance. Each query on a collection corresponds to an index; **index entries** are key-value pairs where keys correspond to all possible query parameters in the collection and values correspond to data matching the respective parameter values. Let's assume we have a collection called `Persons` with fields `Name`, `City`, `Country`, etc. (see snippet below) and we want to find all persons from a certain city; in order to achieve this, we create an index for the `City` field on `Persons` collection where index entries are pairs of city names and list of all `Person` ids from that city. Now querying the city index with a city name returns list of all person ids from that city. RavenDB **clients** define indexes and [deploys](https://ravendb.net/docs/article-page/4.1/csharp/indexes/creating-and-deploying) them to the RavenDB **server**; the server updates an index each time a new document is added to its collection.

{% highlight csharp linenos %}
public class Person
{
    public string Id { get; set; }
    public string Name { get; set; }
    public string City { get; set; }
    public string Country { get; set; }
}
{% endhighlight %}

Creating indexes for field values, like `City`, is straightforward: client just needs to specify the parameter field name and the field to be stored. Now, what if we want to index results of some _aggregate operation_ over a collection? For example, the _count of persons from a certain city_ in the `Persons` collection? It is clear that for each city, we need to store its person count as values into the index (instead of list of person ids, as we did in the last example) so that queried with a city name, the index returns the count of the persons from that city. Now, how does client _define a complex operation like counting_? It is not as straightforward as specifying a field as it is in the previous case. Here map-reduce pattern comes in: during the map phase, map each person to a $$\langle city, count \rangle$$ pair with $$count = 1$$ (by setting $$count \leftarrow 1$$, we _pretend to have a collection with only one person, the aggregate count for the corresponding city being 1_); then group the resultant $$\langle city, 1 \rangle$$ pairs by $$city$$ and reduce each group to a single pair containing the _sum of all counts in that group_. Once reduce phase finishes, for each city, we get a $$\langle city, count \rangle$$ pair containing the total person count. Now storing the $$\langle city, count \rangle$$ pairs as index entries does the job.

RavenDB supports the conceptual map-reduce framework outlined earlier [out-of-the-box](https://ravendb.net/docs/article-page/4.0/csharp/indexes/map-reduce-indexes): client simply needs to pass their custom map and reduce functions as overriden [template methods](https://sourcemaking.com/design_patterns/template_method) of an index creation class. The map function implements the logic for mapping each document to the aggregate result for itself (the $$\langle city, 1 \rangle$$ pairs); the reduce function, in turn, implements the logic for reducing the same aggregate result for each group ($$\langle city, count \rangle$$) from the intermediate results produced by map. RavenDB server internally invokes the map and reduce functions whenever needed in appropriate order with appropritate parameters and stores the output into index. This pattern is extraordinarily capable of implementing versatile computations in a compact, portable format.

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

The `AbstractIndexCreationTask` class that implements the details of index creation serves as base class for indexes. Notice that in the index's constructor, user passes the map and reduce functions by overriding `Map` and `Reduce` properties respectively. `Map` iterates over the entire `Persons` collection and maps each `Person` document to a `Result` instance containing the person count as 1 (the $$\langle city, 1 \rangle$$ pair); `Reduce` iterates over the intermediate `Result` collection produced by Map, groups them by `City`, and produces one `Result` instance containing the aggregate count (sum of all `Result` with the same `City` name) for each city. Once the index is deployed, queried for a city, the index returns a `Result` object containing the sum of all person from that city.

The _MapReduce_ Distributed Computation Model
=============================================

The RavenDB Map-Reduce Indexing shows how map-reduce pattern enables user to pass a specific computation---indexing---for remote data, stored in a server. That capability can be extended for computations in general and for remote data split into multiple partitions. Let's assume that we have a large chunk of text documents and we want to count occurrence of all words in them---practically intractable for a single computer---and you also have a cluster of computing nodes at your disposal. In these situations, where data size is mammoth and cluster of computers is available, [_MapReduce_](https://static.googleusercontent.com/media/research.google.com/en//archive/mapreduce-osdi04.pdf) framework comes in. It is a model of distributed computation, proposed by Google researchers Jeffrey Dean and Sanjay Ghemawat in 2004.

User defines a MapReduce job as a **user program** that is at the same time capable of both map and reduce tasks and job coordination. Copies of the user program starts simultaneously in multiple cluster nodes following the _master-worker topology_: one copy of the user program is designated as **master**, which coordinates a job's execution until its end, and others being **workers** under it. As the program starts, prior to map or reduce, input documents are split into number of partitions. The master assigns each partition to a worker node for mapping in parallel. In a mapper node, the map function receives documents, iterates through their contents, maps them to suitable intermediate key-value result pairs, and writes the result pairs in the mapper node's local disk. Once a mapper node finishes mapping, master node is notified and it forwards the intermediate output locations to a reducer node for reducing. Before reducing, the reducer node groups together the intermediate outputs by their keys---by sorting the keys---then reduces each group to the final result key-value pairs by applying the reduce function, again in parallel. Once master node receives finish notification from all mapper and reducer nodes, it returns control to the initial call. Usually reduce outputs are kept in place and successive MapReduce jobs are run on them to reach some final result.

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

Though MapReduce is targeted for large-scale distributed data processing, the same model suits well even in multi-core, shared-memory environment. In this case, multiple threads are employed instead of multiple independent map-reduce nodes, thereby eliminating need for coordination among nodes and replacing communication over network with faster memory read-writes. .NET Framework's [_Parallel LINQ (PLINQ)_](https://docs.microsoft.com/en-us/dotnet/standard/parallel-programming/parallel-linq-plinq) library provides parallel equivalents to LINQ methods for processing collections; user may write their own custom MapReduce extension method like the following on top of them:

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

