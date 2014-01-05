---
layout: post
title:  "Play Framework's WebSocket API: from imperative events to functional streams (of streams)"
date:   2013-12-25 18:25:21
categories: play
---

Play 2.2's Scala WebSocket API makes you deal with functional abstractions for stream-oriented processing: Iteratees and Enumerators. They allow to have an unified stream API across the whole framework (HTTP body parsing, Comet, WebSockets...). As pointed in the discussion that followed the article [Play, Scala, and Iteratees vs. Node.js, JavaScript, and Socket.io](http://brikis98.blogspot.fr/2013/11/play-scala-and-iteratees-vs-nodejs.html?m=1), Iteratees are very powerful, actually *too* powerful for some basic WS use cases. They are also *stream-oriented*, whereas some WS use cases are more naturally *event-oriented*. This leads to a not-so-simple API for basic event-oriented use cases, whereas all other APIs in Play manage to satisfy one of its goals: a *progressive* learning curve towards functional programming for Web developers.

In this post, I'll show how to obtain a simple *event-oriented* API from Iteratees, and then I'll show when and how to use a more advanced and *stream-oriented* approach with **back-pressure** (one of the main advantages of the stream-oriented Iteratee approach).

## A simple event-oriented example
This approach is the simplest if you come from the imperative world and/or you think that the WS part of your app if more event-oriented than stream-oriented. What I mean by event-oriented is that all messages are independent and concurrent, unlike streams where messages/chunks are sequential. In other words, the fact that the processing of message N has not ended does not prevent to finish the processing of message N+1. Below is a simple imperative example:

```scala
def ws = ImperativeWebsocket.using[String](
  onOpen = channel => {
    Logger.info("connected")
    channel.push("Hello from server")
    // optionally send channel to an actor that pushes some events
    PusherActor ! channel
  },
  
  onMessage = (message, channel) => {
    message match {
      case "event1" =>
        val future1 = // call service 1
        future1 foreach (result => channel.push(result))
      case "event2" =>
        val future2 = // call service 2
        future2 foreach (result => channel.push(result))
      case "event3" =>
        SomeActor ! SomeMessage
    }
  },
  
  onClose = Logger.info("disconnected")
) 
```

`ImperativeWebsocket` is a helper object that can be defined on top of the Iteratee API like this:

```scala
object ImperativeWebsocket {

  def using[E: WebSocket.FrameFormatter](
          onOpen: Channel[E] => Unit, 
          onMessage: (E, Channel[E]) => Unit, 
          onClose: => Unit,
          onError: (String, Input[E]) => Unit = (_: String, _: Input[E]) => ()
  ): WebSocket[E] = {
    val promiseIn = promise[Iteratee[E, Unit]]

    val out = Concurrent.unicast[E](
      onStart = channel => {
        onOpen(channel)
        val in = Iteratee.foreach[E] { message =>
          onMessage(message, channel) 
        } map (_ => onClose) 
        promiseIn.success(in)
      },
      onError = onError
    )

    WebSocket.using[E](_ => (Iteratee.flatten(promiseIn.future), out))
  }

}
```

The internal definition may look like quite complex, but it's because we are twisting Iteratees to get an imperative API from them. We'll see than in stream-oriented use cases, the Iteratee code becomes much more natural and straight-forward.

If you want to go further in the event/message-oriented approach, take a look at [play-actor-room](http://mandubian.com/2013/09/22/play-actor-room/) library which uses actors to hide the Iteratees stuff and exposes a very simple message-oriented API with built-in rooms, broadcasts and more.

As said before, it is important to notice that unlike stream-oriented processing, our imperative event-oriented code handles all messages concurrently. While we are waiting for the future to be fulfilled in `event1`, `event2` can be fully processed and the result of `future2` can be sent to the client before `future1`. As a consequence, we can not control the possible congestions of messages in the server.

## A stream-oriented use case
Sometimes, the IO that you want to model is more stream-like. You really have a flow of sequential (big) data chunks going through your application. In this case, it is very important for performance to handle back-pressure to avoid congestion of data chunks everywhere in your application. Back-pressure allows the slowest part of your stream (source(s), filter(s), sink(s)...) to impose its speed to the whole stream so that there are no accumulation of data chunks in your server memory.

It turns out that Iteratees are perfect for that job. They handle back-pressure transparently and they have a lot of tools to modify and compose streams (see [Enumeratee](http://www.playframework.com/documentation/2.2.x/api/scala/index.html#play.api.libs.iteratee.Enumeratee$) and [Concurrent](http://www.playframework.com/documentation/2.2.x/api/scala/index.html#play.api.libs.iteratee.Concurrent$)).

As an example, we suppose that your WS clients need to continuously send big data chunks (say String-encoded here) to the server, then the server has to call an external service to process each chunk, and then it sends them back to the client in the same order. Here is how we can implement it:

```scala
def ws = WebSocket.using[String] { httpReq =>
  val (in, inStream) = Concurrent.joined[String]
  
  val asyncTransformer = Enumeratee.mapM[String] { data =>
    val futureProcessedData = // call a webservice to process data
    futureProcessedData
  }

  val out = inStream through asyncTransformer
  
  (in, out)
}
```

`Concurrent.joined` allows to transform the `in` Iteratee (data coming from the client) to an Enumerator (stream source).
`Enumeratee.mapM` allows to map a data chunk asynchronously by returning a Future (the `M` stands for Monad, as Future is a Monad).

Now suppose that the external web service suddenly crashes and recovers only 5 s after. Back-pressure prevents the Web client to push more data into `asyncTransformer` before the current data chunk has been transformed, so incoming data chunks won't be stuck in your server filling up your memory while waiting for the external service to recover.

Note that here we do a circle with the stream (from the Web client back to the Web client), but we could also have send the `out` stream to a database for example. Using a reactive (back-pressure enabled) driver like [ReactiveMongo](http://reactivemongo.org/), we get back-pressure all along the way from the Web client to the database! So if the database is temporary slow, the clients' web browsers will automatically push data slowly!

Furthermore, functional composition enables to very simply plug different strategies for back-pressure using Enumeratees (stream transformers).
Maybe we want to be able to buffer 20 data chunks maximum before going through `asyncTransformer` (in case this one is sometimes slow):

```scala
val buffer = Concurrent.buffer[String](20)
val out = inStream through buffer through asyncTransformer
```

Note that `through` has an alias `&>`:

```scala
val out = inStream &> buffer &> asyncTransformer
```

Or we want to simply drop data server-side after a timeout:

```scala
val dropper = Concurrent.dropInputIfNotReady[String](1, TimeUnit.SECONDS)
val out = inStream &> dropper &> asyncTransformer
```

And we want before to re-chunk to bigger chunks (concatenating 10 chunks into one):

```scala
val reChunker = Enumeratee.grouped[String] {
  Enumeratee.take(10) transform Iteratee.consume()
} 
val out = inStream &> reChunker &> dropper &> asyncTransformer
```

Check Play's Iteratee/Enumerator/Enumerator [doc](http://www.playframework.com/documentation/2.2.x/Iteratees), [API](http://www.playframework.com/documentation/2.2.x/api/scala/index.html#play.api.libs.iteratee.Enumeratee$) and [extra](https://github.com/jroper/play-iteratees-extras) for more possibilities, and some Web apps that use a lot of this: [Zound](http://greweb.me/2012/08/zound-a-playframework-2-audio-streaming-experiment-using-iteratees/) (with an example of stream broadcast) and [Ztream](https://github.com/atamborrino/ztream). There are also some very interesting blog posts on the subject, for example on [Klout's engineering blog](http://engineering.klout.com/2013/01/iteratees-in-big-data-at-klout/) and on [LinkedIn's engineering blog](http://engineering.linkedin.com/play/play-framework-democratizing-functional-programming-modern-web-programmers).

Of course, we can also mix the two approach. For example, we could handle the in-coming WS client messages in an imperative way with `Iteratee.foreach` but return a functional Enumerator for out-coming messages (so we have back-pressure from the server to the client).

## Sub-streams composition with back-pressure
For the following example, we suppose that each message (chunk) sent by a WS client contains information to fetch a stream. So each chunk of the main stream produces a stream of messages (a sub-stream). We'll see 3 alternatives to compose sub-streams.

### Sequential sub-streams
If a chunk of a stream produces a sub-stream, then we want the whole sub-stream to be processed before processing the next chunk of the main stream. All of this with back-pressure, so the chunks from the main stream will not fill up server memory waiting for a sub-stream to end!

Sub-streams are Enumerators that can come from databases, WebService's HTTP chunked responses... But here, for testing purposes, I've defined a function `chunkToStream` that takes a chunk and returns a stream that sends each 2 seconds a numbered message with the original chunk (it sends 3 messages in total, and then the stream ends):

```scala
val chunkToStream: String => Enumerator[String] = { chunk =>
  Enumerator.unfoldM[Int, String](1) {
    case i if i <= 3 =>
      Promise.timeout(Some((i + 1, chunk + " " + i.toString)), 2 seconds)
    case _ =>
      Future.successful(None)
  } 
}
```

The WS code is very straight-forward:

```scala
def ws = WebSocket.using[String] { httpReq =>
  val (in, inStream) = Concurrent.joined[String]
  val out = inStream &> Enumeratee.mapFlatten(chunkToStream)
  (in, out)
}
```

Yep, that's it! Iteratees really shine in these kind of use cases.

### Replaceable sub-streams
Now, when a new chunk comes from the client, we want to replace the old sub-stream (even if it wasn't ended) with a new one. For this use case, we use `Concurrent.patchPannel` which allows precisely to do that. It has an imperative signature, so we will define a helper function to keep our code clean.

The WS code is:

```scala
def ws = WebSocket.using[String] { httpReq =>
  replaceableOutStream { inChunk =>
    // we replace the previous stream with the one returned by chunkToStream
    val newOutStream = chunkToStream(inChunk) 
    newOutStream
  }
}
```

I defined this helper function:

```scala
def replaceableOutStream[E, F](handler: E => Enumerator[F]): (Iteratee[E, Unit], Enumerator[F]) = {
  val promiseIn = promise[Iteratee[E, Unit]]

  val out = Concurrent.patchPanel[F] { patcher =>
    val in = Iteratee.foreach[E] { chunk =>
      // patchIn stash the previous stream, and plug the new one returned by handler
      patcher.patchIn(handler(chunk)) 
    }
    promiseIn.success(in)
  }

  (Iteratee.flatten(promiseIn.future), out)
}
```

And we still have back-pressure from the server to the client!

### Mixable sub-streams
As you may have understood with the title, now we just want the new stream returned by chunkToStream to interleave with the previous stream of sub-streams. Sub-streams will end by themselves.

WS code:

```scala
def ws = WebSocket.using[String] { httpReq =>
  mixableOutStream { inChunk =>
    val newSubstream = chunkToStream(inChunk) 
    newSubstream
  }
}
```

Helper function:

```scala
def mixableOutStream[E, F](handler: E => Enumerator[F]): (Iteratee[E, _], Enumerator[F]) = {
  val promiseIn = promise[Iteratee[E, Enumerator[F]]]

  val out = Concurrent.patchPanel[F] { patcher =>
    val in = Iteratee.fold[E, Enumerator[F]](Enumerator.empty) { (currentStream, chunk) =>
      val substream = handler(chunk)
      val newStream = Enumerator.interleave(currentStream, substream)
      patcher.patchIn(newStream)
      newStream
    }
    promiseIn.success(in)
  }

  (Iteratee.flatten(promiseIn.future), out)
}
```

## Final thoughts
We saw that in stream-oriented functional use cases, the Iteratee code stays simple and clear while handling efficiently complex problems. Iteratees are perfect for stream-oriented use cases of WebSocket, but I think Play should propose a built-in API for simple event-oriented use cases. As you saw in my `
ImperativeWebsocket` definition, it is quite easy to provide an imperative API on top of Iteratees once you understand them, but maybe not for new-comers. 

NB: Play's *Java* Websocket API is event-oriented by default.
