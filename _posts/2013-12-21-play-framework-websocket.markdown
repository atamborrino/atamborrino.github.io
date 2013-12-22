---
layout: post
title:  "Play Framework's WebSocket API: from imperative events to functional streams"
date:   2013-12-21 18:25:21
categories: play
---

Play's Scala WebSocket API makes you by default deal with functional abstractions for stream-oriented processing: Iteratees and Enumerators. They allow to have an unfified stream API across the whole framework (HTTP body parsing, Comet, WebSockets, ...). As pointed in the discussion that followed the article [Play, Scala, and Iteratees vs. Node.js, JavaScript, and Socket.io](http://brikis98.blogspot.fr/2013/11/play-scala-and-iteratees-vs-nodejs.html?m=1), Iteratees are very powerful, actually *too* powerful for some basic WS use cases. They are also *stream-oriented*, whereas some WS use cases are more naturally *event-oriented*. This leads to a not-so-simple API for basic event-oriented use cases, whereas all other APIs in Play manage to satisfy one of its goals: a *progressive* learning curve towards functional programming for Web developpers.

In this post, I'll show how to obtain a simple *event-oriented* API from Iteratees, and then I'll show when and how to use a more advanced and *stream-oriented* approach with back-pressure (one of the main advantages of the stream-oriented Iteratee approach).

## A simple event-oriented example
This approach is the simplest if you come from the imperative world and/or you think that the WS part of your app if more event-oriented than stream-oriented. What I mean by event-oriented is that messages are not sequential, ie. the fact that the processing of message N has not ended does not prevent to finish processing message N+1 (everything is concurrent).

<div class="code">
{% highlight scala %}
def ws = WebSocket.using[String] { httpReq =>

  imperativeHandler[String](
    onOpen = channel => {
      Logger.info("start")
      channel.push("hello from server")
    },
    
    onMessage = (channel, message) => {
      Logger.info(message)
      message match {
        case "event1" => 
          val future1 = // call WebService 1
          future1 foreach (channel.push(_))
        case "event2" =>
          val future2 = // call WebService 2
          future2 foreach (channel.push(_))
      }
    },
    
    onClose = Logger.info("close")
  )
}
{% endhighlight %}
</div>

`imperativeHandler` is a helper function that can be defined on top of the Iteratee API like this:
<div class="code">
{% highlight scala %}
def imperativeHandler[E](onOpen: Channel[E] => Unit, 
                         onMessage: (Channel[E], E) => Unit, 
                         onClose: => Unit): (Iteratee[E, Unit], Enumerator[E]) = {

  val promiseIn = promise[Iteratee[E, Unit]]
  val out = Concurrent.unicast[E]( channel => {
    onOpen(channel)
    val in = Iteratee.foreach[E](onMessage(channel, _)) map (_ => onClose) 
    promiseIn.success(in)
  })
      
  (Iteratee.flatten(promiseIn.future), out)
}
{% endhighlight %}
</div>

The internal definition may look like quite complex, but it's because we are tweaking Iteratees to get an imperative API from them. We'll see than in stream-oriented use cases, the Iteratee code becomes much more clear and straight-foward.

If you want to include the possibility of broadcasting to all members, just use `Concurrent.broadcast` on top of it:
<div class="code">
{% highlight scala %}
  lazy val (broadcastOut, broadcastChannel) = Concurrent.broadcast[String]

  def ws = WebSocket.using[String] { httpReq =>
    val (in, out) = imperativeHandler[String](
      onOpen = channel => {
        Logger.info("start")
        channel.push("hello from server")
      },
      
      onMessage = (channel, message) => {
        Logger.info(message)
        message match {
          case "event1" => 
            val future1 = // call WebService 1
            // broadcast to every connect WS clients
            future1 foreach (broadcastChannel.push(_))
          case "event2" =>
            val future2 = // call WebService 2
            // broadcast to every connect WS clients
            future2 foreach (broadcastChannel.push(_))
        }
      },
      
      onClose = Logger.info("close")
    )
    
    (in, Enumerator.interleave(out, broadcastOut))
  } 
{% endhighlight %}
</div>
Here we use some of the power of functional composition with `Enumerator.interleave` which creates a new Enumerator by mixing two other ones. But we still have our imperative event-oriented API by broadcasting trough `broadcastChannel.push(message)`.

If you want to go further in the event/message-oriented approach, take a look at the [play-actor-room](http://mandubian.com/2013/09/22/play-actor-room/) library which use actors to hide the Iteratees stuff and expose a very simple message-oriented API with built-in rooms, broadcasts and more.

It is important to notice that unlike stream-oriented processing (functional Iteratee), our imperative code handles all messages concurrently. While we are waiting for the future to be fullfiled in `event1`, `event2` can be fully processed and the result of `future2` can be sent to the client.

## A stream-oriented use case
Sometimes, the IO that you want to model is more stream-like. You really have a flow of sequential (big) data chunks going through your application. In this case, it is very important for performance to handle back-pressure to avoid congestion of data chunks everywhere in your application. Back-pressure allows the slowest part of your stream (source, filter, sink...) to impose his speed to the whole stream, so that there are no accumulation of data chunks in-memory.

It turns out that Iteratees are perfect or that job. They handles back-pressure transparently and they have a lot of tools to modify and compose streams (see [Enumeratee](http://www.playframework.com/documentation/2.2.x/api/scala/index.html#play.api.libs.iteratee.Enumeratee$) and [Concurrent](http://www.playframework.com/documentation/2.2.x/api/scala/index.html#play.api.libs.iteratee.Concurrent$)).

As an example, we suppose that your WS client needs to continuously send big data chunks (say String-encoded here) to the server, then the server has to call a external service to process each chunk, and then it sends them back to the client in the same order. Here is how we can implement it:

<div class="code">
{% highlight scala %}
def ws = WebSocket.using[String] { httpReq =>
  val (in, inStream) = Concurrent.joined[String]
  
  val asyncTransformer = Enumeratee.mapM[String] { data =>
    // call a webservice to process data
    futureProcessedData
  }

  val out = inStream through asyncTransformer
  
  (in, out)
}
{% endhighlight %}
</div>

`Concurrent.joined` allows to transform the `in` Iteratee (data coming from the client) to an Enumerator.
`Enumeratee.mapM` allows to map a data chunk asynchronously by returning a Future (the `M` stands for the future Monad).

Now suppose that the external web service suddenly crash and recover only 5s after. Back-pressure prevents the Web clients to push more data in `asyncTransformer` before the current data chunk has been transformed, so incoming data chunks won't be stuck in your server filling up your memory waiting for the external service to recover.

Note that here we do a circle with the stream (from the Web client back to the Web client), but we could also have send the `out` stream to a database for example. Using a reactive (back-pressure enabled) driver like [ReactiveMongo](http://reactivemongo.org/), we get back-pressure all along the way, from the Web client to the database! So if the database is slow, the clients' web browsers will automatically push slowly!

Furthermore, functional composition allows use to very simply plug different strategies for back-pressure. 
Maybe we want to be able to buffer 20 data chunks maximum before going through `asyncTransformer` (in case this one is sometimes slow):
<div class="code">
{% highlight scala %}
val buffer = Concurrent.buffer[String](20)
val out = inStream through buffer through asyncTransformer
{% endhighlight %}
</div>
Or we want to simply drop data server-side after a timeout:
<div class="code">
{% highlight scala %}
val dropper = Concurrent.dropInputIfNotReady[String](1, TimeUnit.SECONDS)
{% endhighlight %}
</div>
Or we want to re-chunk to bigger chunks (concatenating 10 strings):
<div class="code">
{% highlight scala %}
val reChunker = Enumeratee.grouped[String] {
  Enumeratee.take(10) transform Iteratee.consume()
} 
{% endhighlight %}
</div>

Check Play's Iteratee/Enumerator/Enumerator [doc](http://www.playframework.com/documentation/2.2.x/Iteratees), [API](http://www.playframework.com/documentation/2.2.x/api/scala/index.html#play.api.libs.iteratee.Enumeratee$) and [extra](https://github.com/jroper/play-iteratees-extras) for more possibilites, and some Web apps that use a lot of this: [Zound](http://greweb.me/2012/08/zound-a-playframework-2-audio-streaming-experiment-using-iteratees/) and [Ztream](https://github.com/atamborrino/ztream). There are also some very interesing blog posts on the subject, for example on [Klout's engineering blog](http://engineering.klout.com/2013/01/iteratees-in-big-data-at-klout/) and on [LinkedIn's engineering blog](http://engineering.linkedin.com/play/play-framework-democratizing-functional-programming-modern-web-programmers).

## Final thoughts
We saw that in stream-oriented use cases, the Iteratee API code stays very simple and clear (much more than imperative code would be) while handling complex problems. Iteratees are perfect for stream-oriented use cases of WebSocket or HTTP body parsing, but I think Play should propose a built-in API for simple event-oriented use cases of WebSocket. As you saw in my `
imperativeHandler` definition, it is quite easy to provide an imperative API on top of Iteratees once you undersand them, but maybe not for new-comers.

