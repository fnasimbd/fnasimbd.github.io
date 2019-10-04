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

_Map_ and _Reduce_ functions, initially part of functional languages like _Lisp_, later of many others, greatly simplify computations on lists. In this write-up, I explore the following aspects of map and reduce functions:
- Their definitions, usage, and properties.
- How they can be used together to model seemingly complex computations, with a real-world application.
- How they inspire a general-purpose, scalable distributed computation framework, namely _MapReduce_.

{% include toc %}

What are Map and Reduce Functions
=================================
Map and reduce functions are convenience functions for simplifying two specific list operations. Both operates on a list with a user-defined **input function**. The input function for map is _single-parameter_ and of _non-void return type_. On invocation on a list, map iterates over the list from left to right, and while doing so, applies the input function on each element, thereby producing a new value for each element. Map keeps appending the new value produced to a new list, and returns that new list, its length being the same as the input list, at the end (**maps the list** to a new one). List _transformation_ operations can easily be viewed as map operations.

Reduce, on the other hand, takes a _two-parameter_ and _non-void return type_ function as input. Reduce starts with a **partial result** value (usally the first element in the list, but user may explicity specify it too) and, like map, iterates over the list from left to right. Reduce applies the input function on each list element and the current partial result and replaces the partial result with the value returned, making it the new partial result. That way, reduce keeps updating the partial result and returns it at the end (**reduces the list** to a value.) List _aggregate_ operations (e.g. sum, average, max-min, etc.) can be viewed as reduce operations.

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
------------------------

Map and reduce together give an elegant abstraction for various computations over collections, called **map-reduce pattern**; in this pattern, a task is divided into two phases: **map phase** and **reduce phase**. During the map phase, the **source collection** is mapped to an **intermediate collection** and during the following reduce phase, some final **aggregate result** is computed from the intermediate collection. It may sound bit restrictive but in practice, map-reduce pattern is, in fact, capable of highly versatile computations.

Careful look reveals that though map and reduce's input functions define the actual computation, they neither _make direct references to the collection_ nor make any _assumption on the collection's length_ (define the computation once, it applies to collections _anywhere_ and of _any length_.) These properties have two consequences: firstly, user may **pass computations**---defined in map-reduce pattern---for data whose location is unknown to them and secondly, computation may be executed on lists of arbitrary length and split into multiple chunks, making computation easily **scalable**. The next section shows how a database system exploits the first facility and later we will see how MapReduce exploits both.

_Map-Reduce Index_ in RavenDB
=============================================
The index and query operation of [RavenDB](https://ravendb.net/docs/article-page/4.0/csharp), the NoSQL document database, is a compelling real-world use of map-reduce pattern simplifying a complex job. Before going into its details, a very brief overview of how RavenDB and its index and query operation works is necessary: RavenDB organizes **documents** into **collections** (for simplicity, you may view documents as objects with a mandatory unique id field and collections as list of all objects in the database of a certain object type); each time a new document is added to the database, it is appended to its corresponding collection if the collection already exists, or a new collection is created for it.

Unlike relational databases, RavenDB _pre-computes_ query results into **indexes** hence achieving faster query performance. Each query on a collection corresponds to an index; indexes consists of key-value **index entry** pairs where keys correspond to all possible parameter values found in the collection for that query and values correspond to data matching the parameter value. Let's assume we have a collection called `Persons` with fields `Name`, `City`, `Country`, and maybe others (see snippet below), and we want to find all persons from a certain city; in order to achieve this, we create an index for the `City` field on `Persons` collection where index entries are city names and list of all `Person` ids from that city. Now querying the city index with a city name returns list of all person ids from that city. Indexes are created by a RavenDB **client** and [deployed](https://ravendb.net/docs/article-page/4.1/csharp/indexes/creating-and-deploying) to the RavenDB **server**. Once an index is deployed, it is updated by server each time a new document is added to its collection.

{% highlight csharp linenos %}
public class Person
{
    public string Id { get; set; }
    public string Name { get; set; }
    public string City { get; set; }
    public string Country { get; set; }
}
{% endhighlight %}

Creating indexes for field values, like `City`, is straightforward: client just needs to specify the parameter field name and the field to be stored. Now, what if we want to index results of some aggregate operation over a collection? For example, the _count of persons from a certain city_ in the `Persons` collection? It is clear that for each city, we need to store its person count as values into the index (instead of list of person ids, as we did in the last example) so that upon querying with a city name, count of the persons from that city is returned. Now, how does client _define a complex operation like counting_? Conceptually, it can be modelled in map-reduce pattern: during the map phase, map each person to a $$\langle city, count \rangle$$ pair with $$count = 1$$; by setting $$count \leftarrow 1$$, we _pretend to have a collection with only one person, the aggregate count for the corresponding city being 1_; before reducing, as an additional step, group the resultant $$\langle city, 1 \rangle$$ pairs by city then reduce each group---that is city---to a single pair containing the _sum of all counts in that group_. Once reduce phase finishes, for each city, we get a $$\langle city, count \rangle$$ pair containing the total person count. Now storing the $$\langle city, count \rangle$$ pairs as index entries does the job.

RavenDB supports the conceptual framework outlined earlier [out-of-the-box](https://ravendb.net/docs/article-page/4.0/csharp/indexes/map-reduce-indexes): client simply needs to pass their custom map and reduce functions as overriden [template methods](https://sourcemaking.com/design_patterns/template_method) of an index creation class. The map function implements the logic for mapping each document to the aggregate result for itself (the $$\langle city, 1 \rangle$$ pairs); the reduce function, in turn, implements the logic for reducing the same aggregate result for each group ($$\langle city, count \rangle$$) from the intermediate results produced by map. RavenDB server internally invokes the map and reduce functions whenever needed in appropriate order with appropritate parameters and stores the output into index. This pattern is extraordinarily capable of implementing versatile computations in a compact, portable format.

**Caution.** The remainder of this section implements an actual map-reduce index according to the conceptual description in the preceding paragraph and assumes little bit familiarity with C# and its LINQ library. You may, however, skip to the next section without loosing the essence of the main theme.
{: .notice--warning}

Here follows sample implementation of a map-reduce index for count of `Person` from each city in the `Persons` collection.

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

The `AbstractIndexCreationTask` class implements details of index creation hence serves as base class for indexes. Notice that in the index's constructor, user passes the map and reduce functions by overriding `Map` and `Reduce` properties respectively. `Map` iterates over the entire `Persons` collection and maps each `Person` document to a `Result` instance containing the person count as 1 (the $$\langle city, 1 \rangle$$ pair); `Reduce` iterates over the intermediate `Result` collection produced by Map, groups them by `City`, and produces one `Result` instance containing the aggregate count (sum of all `Result` with the same `City` name) for each city. Once the index is deployed, for each query on it with a city specified, the index returns a `Result` object containing the sum of all person from that city.

The _MapReduce_ Distributed Computation Model
=============================================

The RavenDB Map-Reduce Indexing shows how map-reduce pattern enables user to pass a specific computation---indexing---for remote data, stored in server. That capability can be extended for computations in general and for remote data split into multiple chunks. Let's assume that we have a large chunk of text documents and we want to count occurrence of all words in them---practically intractable for a single computer---and you also have a cluster of computing nodes at your disposal. In these situations, where data size is mammoth and cluster of computers is available, [_MapReduce_](https://static.googleusercontent.com/media/research.google.com/en//archive/mapreduce-osdi04.pdf) framework comes in. It is a model of distributed computation, proposed by Google researchers Jeffrey Dean and Sanjay Ghemawat in 2004.

MapReduce builds on master-worker architecture: one node in the cluster is designated as **master node** and others being **worker nodes** under it. Clients submit computation jobs defined in map-reduce pattern and input data to the master node; master node coordinates the job's execution until its end. Prior to map phase, the master node splits the input documents into chunks, then assigns each chunk to a worker node for mapping in parallel. In a mapper node, the map function receives documents, iterates through their contents, and maps them to suitable intermediate key-value pairs. Once a worker node finishes mapping, master node forwards the intermediate outputs to a worker node for reducing. Before reducing, intermediate outputs are grouped together by their keys, then reduce function reduces each group to the final result key-value pairs again in parallel. Usually reduce outputs are kept in place and successive MapReduce jobs are run on them to reach some final result.

For our word counting example, map function would iterate over the tokenized words in a document and for each word, produce a $$\langle word, 1 \rangle$$ pair, meaning one occurrence of $$word$$ is found. Before reduction, the intermediate output pairs would be grouped by $$word$$. Reduce function would add up all 1's for a word (e.g. $$\langle word_1, 1,1,1 \rangle$$) hence producing that word's total occurrence (e.g. $$\langle word_1, 3 \rangle$$) The following pseudocode example from the originial MapReduce paper counts the occurrence of each word in a set of documents:

{% highlight linenos %}
map(String key, String value):
    // key: document name
    // value: document contents
    for each word w in value:
        EmitIntermediate(w, "1");

reduce(String key, Iterator values):
    // key: a word
    // values: a list of counts
    int result = 0;
    for each v in values:
        result += ParseInt(v);
    Emit(AsString(result));
{% endhighlight %}

MapReduce jobs are defined as implementations of some interface that may vary from implementation to implementation. Jobs are submitted to master node through network; rest of the tasks (e.g. splitting, scheduling, storing-retrieving data, error control, etc.) are taken care of by the library. The nodes where map operation executes also contains the data, exploiting locality to improve performance.

**Commutativity and Associativity of Reduce and the Resultant Optimization**<br><br>As an optional optimization, where the reduce function is both _commutative_ and _associative_, user can submit a _combiner function_ (usually the reduce function itself) that eliminates duplicates before invoking reduce, thus reducing bandwidth usage; commutativity and assoicativity property ensures that reduce function can be called in any order hence applying reduce function once at the mapping sites, before the actual call, produces the same result.
{: .notice--info}

MapReduce in _Apache Hadoop_
-------------------------------------

Apache Hadoop is an open-source ecosystem providing infrastructure for distributed storage and computation (e.g. its distributed file system HDFS). Hadoop's MapReduce, targeted for JVM platform, is perhaps the most popular implementation of MapReduce. It provides user with interfaces to submit their map and reduce functions written in Java and packaged as JARs, concurrency safe data structures to store the computation results. As a motivation, the following snippet, written in MapReduce's Java API (it has APIs for other languages too,) shows how the map and reduce functions for a job that computes inverted index of words in documents may look like.

{% highlight java linenos %}
public static class InvertedIndexMapper extends Mapper<LongWritable, Text, Text, Text> {
    private final Text document = new Text();
    private Text word = new Text();

    public void map(LongWritable key, Text value, Context context)
            throws IOException, InterruptedException {
        String documentName = ((FileSplit) context.getInputSplit()).getPath().getName();
        document.set(documentName);

        StringTokenizer itr = new StringTokenizer(value.toString());

        while (itr.hasMoreTokens()) {
            word.set(itr.nextToken());
            context.write(word, document);
        }
    }
}

public static class InvertedIndexReducer extends Reducer<Text, Text, Text, Text> {
    public void reduce(Text key, Iterable<Text> values, Context context)
            throws IOException, InterruptedException {
        StringBuffer buffer = new StringBuffer();

        for (Text val : values) {
            if (buffer.length() != 0) {
                buffer.append(" ");
            }

            buffer.append(val.toString());
        }

        Text documentList = new Text();
        documentList.set(buffer.toString());

        context.write(key, documentList);
    }
}
{% endhighlight %}

**MapReduce and _Apache Spark_**<br><br>MapReduce got synonymous with Apache MapReduce. It is just one of the many MapReduce implementations: just like Google's original C++-based one (introduced in 2004, by Google's Jeffrey Dean and Sanjay Ghemawat) had been.<br><br>The [Apache Spark](https://spark.apache.org/) framework---de facto replacement of Apache MapReduce---also supports [map](https://spark.apache.org/docs/2.2.0/rdd-programming-guide.html#transformations) and [reduce](https://spark.apache.org/docs/2.2.0/rdd-programming-guide.html#actions) tasks among many others. Spark, however, replaces MapReduce's file system-based, inefficient operations with a more capable data structure called _Resilient Distributed Datasets (RDD)_.
{: .notice--info}

MapReduce in non-distributed multi-core environment
---------------------------------------------------
Though MapReduce is mainly suited for large-scale distributed processing, many tasks even in single-node multi-core environment can be simplified with it. Microsoft's _Parallel LINQ_ library supports Map-Reduce processing for multi-core.

{% highlight csharp linenos %}
public static ParallelQuery<TResult> MapReduce<TSource, TMapped, TKey, TResult>(
    this ParallelQuery<TSource> source, Func<TSource, IEnumerable<TMapped>> map, 
    Func<TMapped, TKey> keySelector, 
    Func<IGrouping<TKey, TMapped>, IEnumerable<TResult>> reduce)
{
    return source.SelectMany(map).GroupBy(keySelector).SelectMany(reduce);
}
{% endhighlight %}

{% highlight csharp linenos %}
var files = Directory.EnumerateFiles(dirPath, "*.txt").AsParallel();
var counts = files.MapReduce(
    path => File.ReadLines(path).SelectMany(line => line.Split(delimiters)), 
    word => word, 
    group => new[] { new KeyValuePair<string, int>(group.Key, group.Count()) });
{% endhighlight %}

References
----------
1. [A reference](https://courses.cs.washington.edu/courses/cse454/05au/slides/08-map-reduce.pdf).
2. [Another reference](https://lintool.github.io/MapReduceAlgorithms/MapReduce-book-final.pdf).
3. [Aggregating with Apache Spark](https://www.javaworld.com/article/3184109/aggregating-with-apache-spark.html)
