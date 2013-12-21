---
layout: post
title:  "Play Framework's WebSocket API: from imperative events to functional streams"
date:   2013-12-21 18:25:21
categories: play
---

The default Scala WebSocket API of Play makes you deal with functional abstractions for stream-oriented processing: Iteratees and Enumerators. They allow to have an unfified stream API in the whole framework (HTTP body parsing, Comet, WebSockets, ...). As pointed in the discussion that followed the article [Play, Scala, and Iteratees vs. Node.js, JavaScript, and Socket.io](http://brikis98.blogspot.fr/2013/11/play-scala-and-iteratees-vs-nodejs.html?m=1), Iteratees are very powerful, actually *too* powerful for some basic WS use cases. They are also *stream-oriented*, whereas some WS use cases are more naturally *event-oriented*. This leads to a complex API for basic event-oriented use cases, whereas all other APIs in Play manage to satisfy one of Play's goal: a progressive learning curve towards functional programming for Web developpers.

In this post, I'll show how to obtain a simple *event-oriented* API à la node.js from Iteratees, and then I'll show when and how to use a more advanced and *stream-oriented* approach with back-pressure (one of the main advantages of the stream-oriented Iteratee approach).

## A simple event-oriented example
The approach is the simplest if you come from the imperative world and/or you think that the WS part of your app if more event-oriented than stream-oriented.

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
      channel.push("echo: " + message)
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
        broadcastChannel.push(message) 
        // broadcasting to every connected members, including ourself.
      },
      
      onClose = Logger.info("close")
    )
    
    (in, Enumerator.interleave(out, broadcastOut))
  } 
{% endhighlight %}
</div>
Here we use some of the power of functional composition with `Enumerator.interleave` which creates a new Enumerator by mixing two other ones. But we still have our simple event-oriented API by broadcasting trough `broadcastChannel.push(message)`.

If you want to go further in the event/message-oriented approach, take a look at the [play-actor-room](http://mandubian.com/2013/09/22/play-actor-room/) library which use actors to hide the Iteratees stuff and expose a very simple message-oriented API with built-in rooms, broadcasts and more.

## A stream-oriented use case
Sometimes, the IO that you want to model is more stream-like. You really have a flow of data going through your application. In this case, it is very important for performance to handle back-pressure to avoid congestion of data chunks everywhere in your application. Back-pressure allows the slowest part of your stream (source, filter, sink...) to impose his speed to the whole stream, so that there are no accumulation of data chunks in-memory.

It turns out that the Iteratee API are perfect or that job. It handles back-pressure transparently, and has lots of tools to modify and compose streams (see [Enumeratee](http://www.playframework.com/documentation/2.2.x/api/scala/index.html#play.api.libs.iteratee.Enumeratee$) and [Concurrent](http://www.playframework.com/documentation/2.2.x/api/scala/index.html#play.api.libs.iteratee.Concurrent$)).

As an example, we suppose that your WS client needs to continuously send data (say String here) to the server, then the server has to call a external service to process this data, and then it sent it back to the client. Here is how we can implement it:

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

Now suppose that the external web service suddenly crash and recover only 5s after. Back-pressure prevents the Web clients to push more data before the current data chunk is processed, so incoming data chunk won't be stuck in your server filling up your memory waiting for the external service to recover.

Note that here we do a circle with the stream (from the Web client back to the Web client), but we could also have send the `out` stream to a database for example. Using a reactive (back-pressure enabled) driver like [ReactiveMongo](http://reactivemongo.org/), we get back-pressure all along the way, from the Web client to the database! So if the database is slow, the clients' web browsers will automatically push slowly!

Furthermore, functional composition allows use to very simply plug different strategies for back-pressure. 
Maybe we want to buffer 20 data chunks if `asyncTransformer` is too slow:
<div class="code">
{% highlight scala %}
val buffer = Concurrent.buffer[String](20)
val out = inStream through buffer through asyncTransformer
// OR, using the &> alias of through
val out = inStream &> buffer &> asyncTransformer
{% endhighlight %}
</div>
Or we want to simply drop data server-side after a timeout:
<div class="code">
{% highlight scala %}
val dropper = Concurrent.dropInputIfNotReady[String](2, TimeUnit.SECONDS)
{% endhighlight %}
</div>
Or we want to re-chunk to bigger chunks (concat 10 strings):
<div class="code">
{% highlight scala %}
val reChunker = Enumeratee.grouped[String] {
  Enumeratee.take(10) transform Iteratee.consume()
} 
{% endhighlight %}
</div>

Check Play's Iteratee/Enumerator/Enumerator [doc](http://www.playframework.com/documentation/2.2.x/Iteratees) and [API](http://www.playframework.com/documentation/2.2.x/api/scala/index.html#play.api.libs.iteratee.Enumeratee$) for more possibilites, and some Web apps that use a lot of this: [Zound](http://greweb.me/2012/08/zound-a-playframework-2-audio-streaming-experiment-using-iteratees/) and [Ztream](https://github.com/atamborrino/ztream). There are also some very interesing blog posts on the subject, for example on [Klout's engineering blog](http://engineering.klout.com/2013/01/iteratees-in-big-data-at-klout/) and on [LinkedIn's engineering blog](http://engineering.linkedin.com/play/play-framework-democratizing-functional-programming-modern-web-programmers).

## Final thoughts
We saw that in stream-oriented use cases, the Iteratee API code stays very simple and clear (much more than imperative code would be) while handling complex problems. Iteratees are perfect for stream-oriented use cases of WebSocket or HTTP body parsing, but I think Play should propose a built-in API for simple event-oriented use cases of WebSocket, even if we lose composability and back-pressure. As you saw in my `
imperativeHandler` definition, it is quite easy to provide an imperative API on top of Iteratees once you undersand them, but maybe not for new-comers.

