# Writing a Sequence of Documents to Elasticsearch #

I'm writing logs during work.  I keep them in text files, and I've been writing them for many years.  Here's the format:

```
24.04.2012
- some project
  - some task
    - looking at recent changes in X
    I need to modify Y
- some other project
  - some task
    - doing Z
```

So obvious idea for a project came up: let's parse those logs, put them in a database and analyze.  The database of choice would be Elasticsearch, because it takes unformatted data and provides full text search, as well as Kibana analysis tool.

Here's the parser API:

```scala
case class Project(name: String, log: List[String])

case class Workday(date: DateTime, projects: List[Project])

object LogParser extends RegexParsers {
  def apply(input: String): List[Workday]
}
```

*Project* is a list of lines in text files that were logged after a project name.  On a given day there might be several sections logged for different projects, so *Workday* consists of a list of projects.

For inserting records to Elasticsearch, I decided that I'll use a different model:

```scala
case class ProjectDay(date: DateTime, name: String, log: List[String])
```

So I added a `def toProjectDays: Seq[ProjectDay]` method to *Workday*.  Data inserted into the database would come in `Seq[ProjectDay]` form, flatmapped from all workdays.

Here comes first approach at inserting *ProjectDays* into Elasticsearch index, with use of [elastic4s library](https://github.com/sksamuel/elastic4s).  Let's look at ElasticDao importData method:

```scala
class ElasticDao extends JsonProtocol {
  val uri = ElasticsearchClientUri("elasticsearch://localhost:9300")
  val client = ElasticClient.transport(uri)

  implicit object ProjectDayIndexable extends Indexable[ProjectDay] {
    override def json(t: ProjectDay): String = t.toJson.toString
  }

  def importData(dbData: Seq[ProjectDay]): Future[Seq[IndexResult]] = {
    client.execute { create index "log"  }

    Future.traverse(dbData) { projectDay =>
      client.execute {
        index into "log" / "days" source projectDay
      }
    }
  }
}
```

It works fine for small examples.  However, for a real-life work log, accumulated over several years, with around 2600 *ProjectDay* records, following exception occured:

```
EsRejectedExecutionException[rejected execution of 
org.elasticsearch.transport.TransportService$4@6e6db429 on 
EsThreadPoolExecutor[index, queue capacity = 200, 
org.elasticsearch.common.util.concurrent.EsThreadPoolExecutor@34e1db17[
Running, pool size = 8, active threads = 8, queued tasks = 200, completed tasks = 0
]]]
```

What's the matter?  Probably the concurrent write was executed too quickly, so some internal queue (of 200) filled up and crashed the client.

Then I tried to write it all sequentially:

```scala
  def importData(dbData: Seq[ProjectDay]): Seq[IndexResult] = {
    Await.result(client.execute { create index "log"  }, Duration.Inf)

    dbData.map { projectDay =>
      Await.result(client.execute {
        index into "log" / "days" source projectDay
      }, Duration.Inf)
    }
  }
```

Sequential approach works, but reaches quite long execution time:  around 20 seconds for our 1700 records.  It's **slow**.

What could work then?  Elastic4s has additional module offering reactive streams API (it needs to be added as separate dependency to build.sbt).  I tried it:

```scala
  def importData(dbData: List[ProjectDay]) = {
    implicit val system = ActorSystem()
    implicit val materializer = ActorMaterializer()

    val source = Source[ProjectDay](dbData)

    implicit val builder = new RequestBuilder[ProjectDay] {
      def request(projectDay: ProjectDay): BulkCompatibleDefinition =  index into "log" / "days" source projectDay
    }
    val completionFn: () => Unit = { () => system.terminate() }
    val subscriber = client.subscriber[ProjectDay](batchSize = 100, concurrentRequests = 1, completionFn = completionFn)
    val sink = Sink.fromSubscriber(subscriber)

    source.to(sink).run()
  }
```
First, `val source` is defined as a *Source* from in-memory sequence of *ProjectDays*.  Next, thanks to `import com.sksamuel.elastic4s.streams.ReactiveElastic._` and implicit builder with record indexing method, ElasticClient gives us a subscriber to write to.  One of its parameters is *completionFn* closure with `system.terminate` call - without it *importData* would hang after completion.  It's then easy to transform the subscriber into akka-streams *Sink*, combine it with our *Source* and run the stream.

It loads all data successfully, and does it in just few seconds (including parsing, and stream initialization).  Why does it work?  Maybe because of using batches (parameter *batchSize* suggests data is loaded not one record at a time).  Maybe because of backpressure.  I haven't researched it in depth.

There is a simpler solution - using bulk operations on sequence of *index* statements.  It wasn't obvious for me at first, because [elastic4s bulk operations docs](https://github.com/sksamuel/elastic4s/blob/master/guide/bulk.md) doesn't show *bulk* use on Scala *Seq*.  I received a hint from my work colleagues though - thanks!  Here's the code:

```scala
  def importData(dbData: Seq[ProjectDay]): Future[BulkResult] = {
    client.execute { create index "log"  }

    client.execute {
      bulk(dbData.map(index into "log" / "days" source _))
    }
  }
```

It loads the data as quickly as stream implementation.

Here's the [full source code of my small application](https://github.com/bartekkalinka/worklogminer)
