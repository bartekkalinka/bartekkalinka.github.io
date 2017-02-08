# Akka-streams Source: run it, publish it, then run it again #

## Problem 1: obtaining materialized value, and sending stream's blueprint at the same time ##

I was working on a side project.  It took some text data stream source, ran in through a sliding window, for which wordcounts were calculated, and top n words list would be emitted as output.  I wanted to use akka-streams.  The input source and output display were pluggable: text file or [twitter sample stream](https://dev.twitter.com/streaming/reference/get/statuses/sample) as input, console stdout or websockets with little client as output.  I served the websockets with akka-http.  For twitter stream handling, I chose [HBC](https://github.com/twitter/hbc) (because it handles reconnects etc.), with hbc-twitter4j module for twitter json handling.  It's a java library, leveraging callbacks to handle incoming tweets, so to combine this approach with akka-stream, I used Source.actorRef construct.  It gives you an ActorRef, to which you can send elements, and in this way they enter the stream.

In akka-streams the stream is first constructed as a *blueprint*.  That means, when it's put together using Flow API or Graph DSL, it's only a recipe of a stream.  To get any computation done with it, it needs to be run or *materialized*.  Such a blueprint may be materialized many times, each time processing different physical set of data.  Often, during materialization, there are additional objects emitted for different stages of the stream.  These objects are some handlers which allow runtime control of those stages.  In our case, Source.actorRef's *materialized value* is a reference to an actor (ActorRef) for sending messages entering the stream.

When we use akka-http websockets API, `handleWebSocketMessages` directive takes `Flow[Message, Message, Any]` type parameter.  That means, it takes the blueprint of the stream.  The materialization happens somewhere inside websockets handling library.  Now that's a problem, because we need materialized value of our Source.actorRef and we are not running the stream ourselves.

I found a solution in [following post](http://loicdescotte.github.io/posts/play-akka-streams-twitter/) by Loïc Descotte.  The author had a similar problem when connecting twitter4j callback to `Source.actorRef` used to push data into play framework `EventSource`.  The key piece of code is this:

```scala
val (actorRef, publisher) =
 Source.actorRef[TweetInfo](1000, OverflowStrategy.fail)
   .toMat(Sink.publisher)(Keep.both).run()
```

Here, we construct a Source.actorRef, directing its elements into Sink.publisher and then running it.  To keep both materialized values of Source and Sink, we have to use `toMat` method, that takes `Keep.both` parameter, to instruct the stream blueprint, to return a pair of materialized values upon materialization.  Then we run it, getting `actorRef: ActorRef` and `publisher: Publisher` values.  The Publisher class comes from reactive streams specification, and it's possible to obtain another instance of Source from it:

```scala
val newSource = Source.fromPublisher(publisher)
```

Now we have both the reference to input actor, and brand new Source which we can run again later!  The source is going to emit exactly the elements that we send to the actor.  We can first run some aggregations on this Source (with map/filter/sliding methods for example).  Then, resulting Source (called `dataSource`) is packed into a flow (I didn't need websockets client input in my project):

```scala
Flow.fromSinkAndSource(Sink.ignore, dataSource)
```

and may be used for emitting data with akka-http websockets.  It's still a *blueprint* of stream, but with a Source that has another Source running inside it.

As you will see, I need this trick of running and "publishing" a Source into a new Source more than once, so I prepared a generalized function: 

```scala
object RunWithPublisher {
  def source[A, M](normal: Source[A, M])(implicit fm: Materializer, system: ActorSystem): (Source[A, NotUsed], M) = {
    val (normalMat, publisher) = normal.toMat(Sink.asPublisher(fanout = true))(Keep.both).run
    (Source.fromPublisher(publisher), normalMat)
  }
}
```

## Problem 2: creating a live source, identical to all websocket clients ##

When passing stream's blueprint to akka-http websockets handler, each connected client has this blueprint executed separately.  It may be illustrated by taking the second source type in my side project: a text file.  Sliding window aggregation that passes through the input file, runs separately for each connected client.  For example after refresh in the browser, it runs from the beginning again.  That's behavior we can't have when using twitter source obviously.  The data cannot be replayed, because it's live.  As it turns out, the solution of first problem (getting materialized value by publishing running source) is the solution for the second problem (having same data transmitted for all stream executions).

At first I was getting strange results.  Two clients running in separate browser windows displayed different wordcounts of top n words from twitter data (I actually tracked top tweeting users).  The reason for such behavior was that the aggregations after Source.publisher were still run separately for each client:

```scala
//(simplification)
val (twitterSource, actorRef) = RunWithPublisher.source(Source.actorRef)
Future(runTwitterClient(actorRef))
twitterSource.sliding.scan(someAggregation)
handleWebSocketMessages(Flow.fromSinkAndSource(Sink.ignore, twitterSource))
```

The solution was to use RunWithPublisher on whole aggregated source.  This time we don't care about materialized value, the important part is running one instance of whole stream for all clients.

```scala
//(simplification)
val (twitterSource, actorRef) = RunWithPublisher.source(Source.actorRef)
Future(runTwitterClient(actorRef))
val (finalSource, _) = RunWithPublisher.source(twitterSource.sliding.scan(someAggregation))
handleWebSocketMessages(Flow.fromSinkAndSource(Sink.ignore, finalSource))
```

After such treatment, each client sees identical data being transmitted.  There was just one last problem - when all clients disconnected (which should be possible), the server crashed with exception: `WebSocket handler failed with Cannot subscribe to shut-down Publisher`.  A workaround for this was adding `finalSource.to(Sink.ignore).run` before passing finalSource independently to websockets handler.  It makes the Publisher alive even after all clients disconnect.