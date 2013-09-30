---
layout: post
title:  "Play JSON API"
date:   2013-09-30 18:25:21
categories: play json
---

# Title 1
You'll find this post in your `_posts` directory - edit this post and re-build (or run with the `-w` switch) to see your changes!
To add new posts, simply add a file in the `_posts` directory that follows the convention: YYYY-MM-DD-name-of-post.ext.

Jekyll also offers powerful support for code snippets:

<div class="code">
<script src="https://gist.github.com/atamborrino/4360663.js"></script>
</div>

# Title 2

Hello hello.

<div class="code">
{% highlight scala %}
def stream = WebSocket.using[JsValue] { req =>
  val out = Concurrent.patchPanel[JsValue] { patcher =>
      // callback called when the enumerator "out" is applied to an iteratee, here the
      // Websocket output Iteratee      
      patcher.patchIn(streamEnumerator1)
      // ...
      patcher.patchIn(streamEnumerator2)
  }
 
  val in = Iteratee.foreach[JsValue] { json =>
    val topic = // read json and get the topic that the user want to subscribe to
  } mapDone { _ => println("Disconnected") }
 
  (in, out)
}
{% endhighlight %}
</div>

Check out the [Play website][play].

[play]: http://www.playframework.com/
