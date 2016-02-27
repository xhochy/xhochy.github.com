---
layout: post
title: Akka Streams for extracting Wikipedia Articles
---

*We use Akka Streams as a new technique to extract specific articles from the Wikipedia xml dump into single files without the need to fit all data into RAM.*

One of the best data sources to derive information on certain topics is the Wikipedia.
Each month, they publish a [data dump of the whole site](https://dumps.wikimedia.org/enwiki/) for each available language.
For a side project, I wanted to obtain the country of origin for music artists from the information stored in the articles.
Mostly the information is inside a special Infobox directly at the beginning of the article but in some cases, it is part of the main text.
Also as I often run other types of analytics on Wikipedia articles, I wanted to have a reproducible way to extract the relevant articles from a new dump without loading it fully into RAM.

Although you could simply use a streaming XML parser and just directly dump out the contents of an articles once you pass over it, with [Akka Streams](http://doc.akka.io/docs/akka-stream-and-http-experimental/1.0/scala/stream-introduction.html) you can build a pipeline that is easily extended to more logic.
Akka Streams is an implementation of the [Reactive Streams](http://www.reactive-streams.org/) initiative that allows you to asynchronous process data streams while adhering to bounded resources.
For handling those streams, you are provided with the already known interfaces of the `map`, `filter`, `reduce`, .. functions of Scala.

A pipeline in Akka Streams is made up of a `Source` that emits elements into the pipeline, an arbitrary number (none is OK) of `Flows` that process the data and a `Sink` that consumes the output of the pipeline, e.g. stores it to files.
An important aspect of Reactive Streams is that each of the stages in the pipeline controls the velocity of the flow of data through the notion of back pressure.
This is to ensure that not too many elements are buffered in memory.
With this technique we can control exactly the amount of RAM used and do not suffer `OutOfMemoryErrors`.

For extracting articles from the (English) Wikipedia Dump, our main source is the `enwiki-20151002-pages-articles.xml.bz2` file.
To integrate this in the Akka Stream system, we need a `Source` that reads this file and emits article-by-article while parsing the file.
Akka Streams provides a constructor to create a `Source` from an `Iterator` that will evaluated in line with the back pressure from the later stages in the pipeline.
This `Iterator` is [implemented through streaming XML parsing](https://github.com/xhochy/open-data-dump-analyses/blob/e4e67d42b7a8d3b1d262d7fb75e03b3e017a996f/wikipedia/akka-streams/src/main/scala/com/xhochy/WikiArticleIterator.scala).
To use it in Akka Streams, we simply instantiate the `Iterator` and pass it to the according `Source` constructor:

{% highlight scala %}
def source(filename:String): Source[WikiArticle, Unit] = {
  val fis = new FileInputStream(filename)
  val bcis = new BZip2CompressorInputStream(fis)
  val iter = new WikiArticleIterator(bcis)
  Source(() => iter)
}
{% endhighlight %}

For our pipeline, we model two simple steps: First we heuristically determine the type of the article and then re-compress its contents if its possibly music related, then we filter on only the articles that are about an artist and not e.g. only a song.
To utilise the multi-core compute power of concurrent CPUs, we utilise Akka Stream's capability to asynchronously process some steps in the pipeline.
Asynchronous (or parallel) steps are implemented in Akka Streams as transformers that return Future and for each of these step, the amount of parallelism is fixed so that the flow and the memory usage is kept at a constant level.

In our pipeline, the object returned from the `WikiArticleIterator` contains a title property which we simply pass on to the new `Article` object.
Whereas we use the `text` property to determine the type of the article, in this case based on the presence of an infobox, and then store the compressed content in the object.
We do the compression here in the future so that we can utilise the multicore power of the CPUs in our machines (This step will be executed several times in parallel).
As only the music related articles are relevant for us later on, we just store an empty bytearray in the articles of the type `Other`.

{% highlight scala %}
{% raw %}
def guessType(content: WikiArticle)(implicit ec: ExecutionContext):Future[Article] = {
  Future {
    if (content.text.contains("{{Infobox musical artist")) {
      new Article(content.title, ArticleType.Artist, compressContent(content))
    } else if (content.text.contains("{{Infobox album")) {
      new Article(content.title, ArticleType.Album, compressContent(content))
    } else if (content.text.contains("{{Infobox single")) {
      new Article(content.title, ArticleType.Song, compressContent(content))
    } else {
      new Article(content.title, ArticleType.Other, new Array[Byte](0))
    }
  }
}
{% endraw %}
{% endhighlight %}

For the compression, we utilise the BZip2 implementation in the [Apache Common Compress](https://commons.apache.org/proper/commons-compress/) package.
Using a `ByteArrayOutputStream`, we can compress the text data on-the-fly in memory. 

{% highlight scala %}
{% raw %}
def compressContent(content: WikiArticle) = {
  val baos = new ByteArrayOutputStream()
  val bcos = new BZip2CompressorOutputStream(baos)
  bcos.write(content.text.getBytes)
  bcos.close()
  baos.toByteArray
}
{% endraw %}
{% endhighlight %}

The second step of the pipeline is the filter on only the artist related pages.
This filter is written as we would be dealing with a normal (everything in memory) list: `.filter(_.articleType == ArticleType.Artist)`.

The final stage of the whole stream is a `Sink` where we define where the results of the pipeline shall go / be stored.
In this implementation, we store each article as a file in the local filesystem.
To handle special characters in the article title that the filesystem might not be able to store, we use the base64 encoded article title as the filename.
Furthermore to have a progress monitoring, we print the current number of saved articles on the command line.
As the sink is a special kind of reduction, we return in each call to our sink function the current number of articles.

{% highlight scala %}
{% raw %}
  def storeArticle(x:Int, article:Article) = {
    val filename = "articles/" +
      Base64.getUrlEncoder()
        .encodeToString(article.title.getBytes())
        .replace("/", "_") +
      ".txt.bz2"
    val fos = new FileOutputStream(filename)
    fos.write(article.compressedContent)
    fos.close()
    print(x.toString + "\r")
    x + 1
  }
{% endraw %}
{% endhighlight %}

With all the parts of the pipeline now implemented, we can plug them together.
We need as the basis an `ActorSystem` and an `ActorMaterializer` and also import the default dispatcher into the current scope as implicts which will be picked up by the respective Akka Streams calls.
For the asynchronous tasks, we need to specify how much parallelism we want. We simply default here to the number of available CPUs.

For the pipeline, the first components we initialise are the input, i.e. the source, and the output, i.e. the `Sink` which is in our case a routine that stores the final articles on disk and counts the number of processed articles.
To connect source and sink, we add the intermediate steps of the pipeline.
The main step is the `guessType` function which we chain with `mapAsyncUnordered(numCPUs)` into the pipeline so that we run it several times in parallel.
Afterwards, we apply a filter to only pass the articles about artists to the sink.
With `toMat` we start the materialisation of the stream with the `storeArticle` function as the destination.

The stream returns a future to the result of the sink function fold.
On the completion of this future, we shutdown Akka's actor system and print the total number of artist pages.

{% highlight scala %}
{% raw %}
  def main(args: Array[String]) {
    implicit val system = ActorSystem("music-articles")
    implicit val materializer = ActorMaterializer()
    import system.dispatcher

    val numCPUs = Runtime.getRuntime().availableProcessors()

    val sink = Sink.fold(0)(storeArticle)
    val src = source(args(0))
    val counter = src.mapAsyncUnordered(numCPUs)(guessType)
        .filter(_.articleType == ArticleType.Artist)
        .toMat(sink)(Keep.right)
    val sum: Future[Int] = counter.run()
    sum.andThen({
        case _ =>
          system.shutdown()
          system.awaitTermination()
      })
    sum.foreach(c => println(s"Total artist pages found: $c"))
  }
{% endraw %}
{% endhighlight %}
