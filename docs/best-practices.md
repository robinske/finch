## Best Practices

* [The big picture](best-practices.md#the-big-picture)
* [Picking a JSON library](best-practices.md#picking-a-json-library)
* [Do not block an endpoint](best-practices.md#do-not-block-an-endpoint)
* [Use TwitterServer](best-practices.md#use-twitterserver)
* [Monitor your application](best-practices.md#monitor-your-application)
* [Picking HTTP statuses for responses](best-practices.md#picking-http-statuses-for-responses)
* [Configuring Finagle](best-practices.md#configuring-finagle)

--

### The big picture

There were a lot of discussions and thoughts on "What an idiomatic Finch program look like?" (see
the [Best Practices and Abstractions][issue263] issue and the [Future Finch][future-finch] writeup),
but we're not sure we should end up sticking with "one-size-fits-all" style in Finch. In fact, zero
lines of Finch's code were written with some particular style in mind, rather than reasonable
composition and reuse of its building blocks. That said, Finch will always be a _library_ that
doesn't promote any concrete style of organizing a code base (frameworks usually do that), but does
promote composability and compile-time guarantees for its abstractions.

Given that major Finch's abstractions are pretty generic, you can make them
[to have any faces you like][faceless] as long as it makes programming fun again: some of the Finch
users write [Jersey-style][diar] programs, some of them stick with [CRUD-style][issue263] ones.

Note that all of the Finch examples are written in the "Vanilla Finch" style, when no additional
levels of indirections are layered on top of endpoints.

### Picking a JSON library

It's highly recommended to use [Circe][circe], whose purely functional nature aligns very well with
Finch (see [JSON docs](json.md#circe) on how to enable Circe support in Finch). In addition to a
[compile-time derived decoders][generic-decoders] and encoders, Circe comes with
[a great performance][circe-performance] and very useful features such as case class patchers and
incomplete decoders.

Case class patchers are extremely useful for `PATCH` and `PUT` HTTP endpoints, when it's required to
_update_ the case class instance with new data parsed from a JSON object. In Finch, Circe's case
class patchers are usually represented as `RequestReader[A => A]`, which

1. parses an HTTP request for a partial JSON object that contains fields need to be updated and
2. represents that partial JSON object as a function `A => A` that takes a case class and updates
   it with all the values from a JSON object

```scala
import io.finch._
import io.finch.circe._
import io.circe.generic.auto._

case class Todo(id: UUID, title: String, completed: Boolean, order: Int)
val patchedTodo: Endpoint[Todo => Todo] = body.as[Todo => Todo]
val patchTodo: Endpoint[Todo] = patch("todos" :: uuid :: patchedTodo) { (id: UUID, pt: Todo => Todo) =>
  val oldTodo = ??? // get by id
  val newTodo = pt(oldTodo)
  // store newTodo in the DB
  Ok(newTodo)
}
```

[Incomplete decoders][incomplete-decoders] are generated by Circe and Finch to decode JSON objects
with some fields missing. This is very useful for `POST` HTTP endpoints that create new entries from
JSON supplied by user and some internally generated ID. A Circe's incomplete decoder is represented
in Finch as a `Endpoint[(A, B, ..., Z) => 0ut]`, where `A, B, ..., Z` is a set of missing
fields from the `Out` type. This type signature basically means that an endpoint, instead of a
final type (a complete JSON object), gives us a function, to which we'd need to supply some missing
bits to get the final instance.


```scala
import io.finch._
import io.finch.circe._
import io.circe.generic.auto._

val postedTodo: RequestReader[Todo] = body.as[UUID => Todo].map(_(UUID.randomUUID()))
val postTodo: Endpoint[Todo] = post("todos" :: postedTodo) { t: Todo =>
  // store t in the DB
  Ok(t)
}
```

By default Finch uses Circe's `Printer` to serialize JSON values into strings, which is quite
convenient given that it's possible to _configure_ and enable some extra options (eg., to drop null
keys in the output string, replace the `io.finch.circe._` import with `io.finch.circe.dropNullKeys._`
one), but it's not the most performant printer in Circe. Always use
[Jackson-powered printer with Circe][circe-jackson] (i.e., replace the `io.finch.circe._` import
with `io.finch.circe.jacksonSerializer` one) unless you absolutely not happy with its output format
(i.e., want to drop null keys).

### Do not block an endpoint

Finagle is very sensitive to whether or not its worker threads are blocked. In order to make sure
your HTTP server always makes progress (accepts new connections/requests), do not block Finch
endpoints. Use `FuturePool`s to wrap expensive computations.

```scala
import io.finch._
import com.twitter.util.FuturePool

val expensive: Endpoint[BigInt] = get(int) { i: Int =>
  FuturePool.unboundedPool {
    BigInt(i).pow(i)
  }
}
```

### Use TwitterServer

Always serve Finch endpoints within [TwitterServer][twitter-server], a lightweight server template
used in production at Twitter. TwitterServer wraps a Finagle application with a bunch of useful
features such as command line flags, logging and more importantly HTTP admin interface that can tell
a lot on what's happening with your server. One of the most powerful features of the admin interface
is [Metrics][metrics], which captures a snapshot of all the system-wide stats (free memory, CPU
usage, request success rate, request latency and many more) exported in a JSON format.

Use the following template to empower your Finch application with TwitterServer.

```scala
import io.finch._

import com.twitter.finagle.param.Stats
import com.twitter.server.TwitterServer
import com.twitter.finagle.{Http, Service}
import com.twitter.finagle.http.{Request, Response}
import com.twitter.util.Await

object Main extends TwitterServer {

  val api: Service[Request, Response] = ???

  def main(): Unit = {
    val server = Http.server
      .configured(Stats(statsReceiver))
      .serve(":8081", api)

    onExit { server.close() }

    Await.ready(adminHttpServer)
  }
}
```

### Monitor your application

Use [Metrics][metrics] to export domain-specific metrics from your application and export them
automatically with [TwitterServer][twitter-server]. Metrics are extremely valuable and helps you
better understand your application under different circumstances.

One of the easiest things to export is a _counters_ that captures the number of times some event
occurred.

```scala
import io.finch._
import io.finch.circe._
import io.circe.generic.auto._

import com.twitter.server.TwitterServer
import com.twitter.finagle.stats.Counter

object Main extends TwitterServer {
  val todos: Counter = statsReceiver.counter("todos")
  val postTodo: Endpoint[Todo] = post("todos" :: postedTodo) { t: Todo =>
    todos.incr()
    // add todo into the DB
    Ok(t)
  }
}
```

It's also possible to export histograms over the random values (latencies, number of active users,
etc).

```scala
import io.finch._
import io.finch.circe._
import io.circe.generic.auto._

import com.twitter.server.TwitterServer
import com.twitter.finagle.stats.Stat

object Main extends TwitterServer {
  val getTodosLatency: Stat = statsReceiver.stat("get_todos_latency")
  val getTodos: Endpoint[List[Todo]] = get("todos") {
    Stat.time(getTodosLatency) { Ok(Todo.list()) }
  }
}
```

Both Finagle and user defined stats are available via TwitterServer's HTTP admin interface or HTTP
endpoint `/admin/metrics.json`.

### Picking HTTP statuses for responses

There is no one-size-fits-all answer on what HTTP status code to use for a particular response, but
there are [best practices][rest-api-bb] established by community as well as
[world-famous APIs][statuses]. Finch is trying to take a neutral stand in this question by providing
an `Output` API abstracted over the HTTP statuses so any of them might be used to construct a
payload, a failure of an empty output. On the other hand, there is an _optional_ and lightweight API
(i.e., methods `Ok`, `BadRequest`, `NoContent`, etc) providing a reasonable mapping between response
type (payload, failure, empty) and its status code. This API (mapping) shouldn't be blindly followed
since there are always exceptions (specific applications, opinionated design principles) from the
general rule. Simply speaking, you're more than welcome to use the mapping we believe makes sense,
but you don't have to stick with that and can always drop down to the `Output.*` API.

The current Finch's mapping is following.

Output type | Status codes
------------| ------------
Payload     | 200, 201
Empty       | 202, 204
Failure     | 4xx, 5xx

### Configuring Finagle

Finch uses Finagle to serve its endpoints (converted into Finagle `Service`s) so it's important to
know what Finagle can do for you in order to improve the resiliency of your application. While
[Finagle servers][finagle-servers] are quite simple compared to Finagle clients, there are still
useful server-side features that might be useful for most of the use cases.

 All Finagle servers are configured via a collection of [`with`-prefixed methods][finagle-config]
 available on their instances (i.e., `Http.server.with*`). For example, it's always a good idea to
 put bounds on the most critical parts of your application. In case of Finagle HTTP server, you
 might think of overriding a [concurrency limit][finagle-concurrency] (a maximum number of
 concurrent requests allowed) on it (disabled by default).

 ```scala
 import com.twitter.finagle.Http

 val server = Http.server
   .withAdmissionControl.concurrencyLimit(
     maxConcurrentRequests = 10,
     maxWaiters = 10,
   )
   .serve(":8080", service)
 ```

[issue263]: https://github.com/finagle/finch/issues/263
[future-finch]: https://gist.github.com/vkostyukov/411a9184f44a136e2ad9
[faceless]: http://gameofthrones.wikia.com/wiki/Faceless_Men
[diar]: https://github.com/jeremyrsmith/dair
[examples]: https://github.com/finagle/finch/tree/master/examples/src/main/scala/io/finch
[circe]: https://github.com/travisbrown/circe
[circe-performance]: https://github.com/travisbrown/circe#performance
[circe-jackson]: https://github.com/travisbrown/circe/pull/111
[generic-decoders]: https://meta.plasm.us/posts/2015/11/08/type-classes-and-generic-derivation/
[incomplete-decoders]: https://meta.plasm.us/posts/2015/06/21/deriving-incomplete-type-class-instances/
[twitter-server]: https://twitter.github.io/twitter-server/
[metrics]: https://twitter.github.io/twitter-server/Features.html#metrics
[statuses]: https://gist.github.com/vkostyukov/32c84c0c01789425c29a
[rest-api-bb]: http://www.vinaysahni.com/best-practices-for-a-pragmatic-restful-api
[finagle-servers]: http://twitter.github.io/finagle/guide/Servers.html
[finagle-config]: http://twitter.github.io/finagle/guide/Configuration.html
[finagle-concurrency]: http://twitter.github.io/finagle/guide/Servers.html#concurrency-limit
