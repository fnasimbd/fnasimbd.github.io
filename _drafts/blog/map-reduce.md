---
layout: post
title: "Map-Reduce Pattern"
modified:
categories: articles
excerpt: "Introduction to the _map-reduce_ pattern and its elegant usage."
tags: ["map-reduce", "distributed-processing"]
comments: true
share: true
---

_Map_ and _Reduce_ functions, initially part of functional languages like _Lisp_, later of many others, greatly simplify computations on sequences. Later they inspired a model of distributed computation called _MapReduce_, introduced by Google and made ubiquitous with Apache's [Hadoop MapReduce](https://hadoop.apache.org/docs/stable/hadoop-mapreduce-client/hadoop-mapreduce-client-core/MapReduceTutorial.html) framework. This write-up explores the use of map and reduce functions, how they relate to the MapReduce pattern, and how MapReduce pattern is implemented in different libraries.

{% include toc %}

What are Map and Reduce Functions
=================================
Map and reduce functions are convenience functions for simplifying specific list operations, available in all major language libraries. In some languages, they are available as static functions and in others as members of list types. Commonly, both map and reduce functions take a list and a user-defined function as input; <s>map's input function takes one parameter and reduce's</s>; on invocation, both iterates over the list from left to right, and apply the function on each element. Map keeps appending the value returned by the function for each value to a new list, and returns that new list, its length being the same as the input list, at the end (**maps the list** to a new one); reduce, on the other hand, cumulatively keeps updating a single value and returns it at the end (**reduces the list** to a value.) Many list _transformation_ operations can be easily viewed as map operations, and similarly, list _aggregate_ operations (e.g. sum, average, max-min, etc.) can be viewed as reduce operations.

Here, as a representative, follows how map and reduce functions are implemented in JavaScript. Outside the syntactic differences, other languages implement map and reduce more or less the same way. The following example maps an input array to its squares.

{% highlight javascript linenos %}
var arr = [1, 2, 3, 4, 5];

var res = arr.map((value, ind, arr) => {
    return value * value;
});

// output: [ 1, 4, 9, 16, 25 ]
{% endhighlight %}

The following example reduces an input array to the product of its elements.

{% highlight javascript linenos %}
var arr = [1, 2, 3, 4];

var res = arr.reduce((pre, cur, arr) => {
    return pre * cur;
});

// output: 24
{% endhighlight %}

**Map-Reduce Input Functions and _Closures_**<br><br>As we have seen, map and reduce takes functions as input. An obvious question arises when the input functions access data (variables) out of their scope. That brings us to _closures_. As I said earlier, map and reduce are concepts imported from functional languages where all data are passed as parameters. Closure itself is a complex idea and deserves separate treatment. You may consult the following references: this stackoverflow [answer](https://stackoverflow.com/a/7464475/615119) for a short introduction and MDN [reference](://developer.mozilla.org/en-US/docs/Web/JavaScript/Closures) for extensive treatment.
{: .notice--info}

Map and reduce together give a cleaner, **scalable** abstraction for various computations over collections. It is called **map-reduce pattern**; in this pattern, a task is divided into two phases: **map phase** and **reduce phase**; during the map phase, the **source collection** is mapped to an **intermediate collection** and during the following reduce phase, some final **aggregate result** is computed from the intermediate collection. Map and reduce do not _make direct references to the collection_ and also make _no assumption on the collection length_---define the computation to be done as map function, it applies to collections anywhere and of any length. As we will see, these properties make map-reduce scalable.

_Map-Reduce Index_ in RavenDB
=============================================
The index and query operation of [RavenDB](https://ravendb.net/docs/article-page/4.0/csharp), the NoSQL document database, is a compelling real-world use of map-reduce pattern simplifying a complex job. Before going into its details, a very brief overview of how RavenDB and its index and query operation works is necessary: RavenDB organizes _documents_ into _collections_ (for simplicity, you may view documents as objects with a mandatory unique id field and collections as list of all objects in the database of a certain object type); each time a new document is added to the database, it is appended to its corresponding collection if the collection already exists, or a new collection is created for it.

In RavenDB, users can perform queries to find documents satisfying some criteria, however, performing queries require creating corresponding _indexes_ prior to that. Let's assume we have a collection called `Persons` with fields `Name`, `City`, `Country`, and maybe others, and we want to find all persons from a certain city in that collection; to achieve this, we have to create an index on the field `City` and query on it with desired city name (called _querying an index_ in RavenDB terminology.)

{% highlight csharp linenos %}
public class Person
{
    public string Name { get; set; }
    public string City { get; set; }
    public string Country { get; set; }
}
{% endhighlight %}

Querying fields is straightforward (RavenDB even automatically creates indexes for simple queries on fields.) Now, what if we want to query results of some aggregate operation over a collection? For example, the count of persons from a certain city in the collection of persons? It can be modelled as a map-reduce problem with a small tweak: first, in the map phase, map each document to a _<city, count>_ pair with _count_ = 1; by setting _count_ = 1, we _pretend to have a collection with only one person, the aggregate count for the corresponding city being 1_. Next, during the reduce phase, group the resultant _<city, 1>_ pairs by city as an additional step, and reduce each group---that is city---to a single pair containing the _sum of all counts in that group_. Once reduce phase finishes, for each city, we get a _<city, count>_ pair containing the total person count.

RavenDB supports creating indexes in [map-reduce pattern](https://ravendb.net/docs/article-page/4.0/csharp/indexes/map-reduce-indexes) out-of-the-box; user just needs to pass their custom map and reduce functions as [template methods](https://sourcemaking.com/design_patterns/template_method). The map function iterates over the entire collection and transforms each document into the aggregate result for itself (the _<city, 1>_ pair); the reduce function computes the same aggregate result for each city (from the temporary results computed by map.) Here follows sample implementation of a map-reduce index for count of `Person` from each city in the `Persons` collection; implementing the same functionality without Map-Reduce Index would be tedious.

**Caution.** The remainder of this section implements an actual map-reduce index according to the conceptual description in the preceding paragraph and assumes little bit familiarity with C# and its LINQ library. You may, however, skip to the next section without loosing the essence of the main theme.
{: .notice--warning}

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

Notice that in the index's constructor, user passes the map and reduce functions as `Map` and `Reduce` properties respectively. In the first pass, `Map` iterates over the entire `Persons` collection and transforms each `Person` document into a `Result` instance containing the person count as 1 (the _<city, 1>_ pair); `Reduce` iterates over the Results collection produced by Map earlier, groups them by `City`, and produces one `Result` instance containing the aggregate count (sum of all `Result` with the same `City` name) for each city. Once the index is prepared, for each query on it with a city specified, the index returns a `Result` object containing the sum of all person from there. For more complex queries, user may augment the `Result` type to store relevant data from `Person` for computing the aggregate result during reduce.

The _MapReduce_ Distributed Computation Model
=============================================
Let's assume that some aggregate result (e.g. computing sum of all integers, counting occurrence of all words, creating inverted index of words, etc.) is to be computed on a large chunk of documents---practically impossible for a single computer---and you also have a cluster of computing nodes at your disposal. In these situations, where data size is mammoth and cluster of computers is available, [_MapReduce_](https://static.googleusercontent.com/media/research.google.com/en//archive/mapreduce-osdi04.pdf) framework comes in; as its name suggests, it employs a **model of distributed computation**, exploiting map-reduce pattern.

A typical MapReduce setup consists of a mandatory _master node_ and several _worker nodes_; the master node accepts computation jobs (**MapReduce jobs**) defined by user and orchestrates the job's execution until end. Just like RavenDB Map-Reduce Index, a MapReduce job consists of a user-defined pair of map and reduce functions. <s>Map and reduce functions are slightly modified to make them suitable for distributed processing, and are orchestrated to work together</s>. During map phase, the master node splits the input documents into chunks and assigns each chunk to a worker node for mapping. The map function receives _input key-value pairs_ (e.g. _\<document_001, word_1\>_), where the key identifies the document, and for each input key-value pair, it outputs a set of _intermediate key-value_ pairs (e.g. _\<word_1, 1\>_); those intermediate key-value pairs (e.g. _\<word_1, 1, 1, 1\>_), possibly from different nodes in a cluster, are grouped together by the MapReduce library, and are then forwarded to the reduce function to produce the final _result key-value pairs_ (e.g. _\<word_1, 3\>_). Usually reduce outputs are kept in place and successive MapReduce jobs are run on them to reach some final result. The following pseudocode example <s>from Dean and Ghemawat paper</s> counts the occurrence of each word in a set of documents:

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

MapReduce jobs are defined as implementations of some interface defined by the MapReduce library, and are submitted to master node through network; rest of the task (e.g. division, scheduling, storing-retrieving data, error control, etc.) happens within the library. The MapReduce library partitions the map and reduce tasks and assigns them to different cluster nodes to run in parallel; the nodes where map operation executes also contains the data, exploiting locality to improve performance.

**Commutativity and Associativity of Reduce and the Resultant Optimization**<br><br>As an optional optimization, where the reduce function is both _commutative_ and _associative_, user can submit a _combiner function_ (usually the reduce function itself) that eliminates duplicates before invoking reduce, thus reducing bandwidth usage; commutativity and assoicativity property ensures that reduce function can be called in any order hence applying reduce function once at the mapping sites, before the actual call, produces the same result.
{: .notice--info}

The Apache Hadoop MapReduce Framework
-------------------------------------

Apache Hadoop MapReduce is perhaps the most popular open source implementation of MapReduce for JVM platform. It provides user with interfaces to submit their map and reduce functions as JARs, concurrency safe data structures to store the computation results. The following snippet, written in MapReduce's Java API (it has APIs for other languages too,) shows how the map and reduce functions for a job that computes inverted index of words in documents may look like.

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
