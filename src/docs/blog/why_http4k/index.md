# {{WORK IN PROGRESS}}

# Server as a Function. In Kotlin. Typesafe. Without the Server.

##### [@daviddenton](http://github.com/daviddenton) / november 2017

## Meet [**http4k**](https://http4k.org).

Whenever (yet another) new JVM HTTP framework is released, the inevitable question that rightly get asked is **"How it this different to X?"**. In this post, I'm going to briefly cover what [**http4k**](https://http4k.org) is, how we think it's different, and address some of those **bold claims** from the title of this post.

Here's a quick rundown of what we think those differences are:

* [**http4k**](https://http4k.org) is small. Written in pure functional Kotlin, with zero dependencies.
* [**http4k**](https://http4k.org) is simple. Like, really simple. No static API magic, no annotations, no reflection.
* [**http4k**](https://http4k.org) is immutable. It relies on an immutable HTTP model, which makes it a snap to test and debug.
* [**http4k**](https://http4k.org) is symmetric. The remote HTTP model is the same as the incoming model.
* [**http4k**](https://http4k.org) is typesafe. Say goodbye to validation and marshalling boilerplate and hello to auto-generated documentation.
* [**http4k**](https://http4k.org) is serverless. Or rather - server independent. Test an app locally and then deploy it into AWS Lambda with no changes.

### Oh god not another framework! Why does this even exist?!?
Firstly - we don't consider [**http4k**](https://http4k.org) to be a framework - it's a set of libraries providing a toolkit to serve and consume HTTP services, focussing on simple, consistent, and testable APIs. Hence, whilst it does provide support for various APIs *relevant to serving and consuming HTTP*, it does not provide every integration under the sun - merely simple points to allow those integrations to be hooked in.

Another thing to say is that (not very much) of [**http4k**](https://http4k.org) is new - it's rather the distillation of 15 years worth of experience of using various server-side libraries and hence most of the good ideas are stolen. For instance - the routing module is inspired by [UtterlyIdle](https://github.com/bodar/utterlyidle), the basic "Server as a function" model is stolen from [Finagle](https://twitter.github.io/finagle/), and the contract module OpenApi/Swagger generator is ported from [Fintrospect](http://fintrospect.io/). 

With the growing adoption of Kotlin, we wanted something that would fully leverage the functional features of the language and it felt like a good time to start something from scratch, whilst avoiding the *magic* that plagues other frameworks and focusing on simple, consistent, functional (and therefore) testable APIs. Hence, [**http4k**](https://http4k.org) is primarily designed to be a Kotlin-first library.

## Claim A: Small, simple, immutable.
Based on the awesome ["Your Server as a Function"](https://monkey.org/~marius/funsrv.pdf) paper from Twitter, [**http4k**](https://http4k.org) apps are modelled by composing 2 types of simple, independent function. 

### Function 1: HttpHandler
An `HttpHandler` and represents an HTTP endpoint. It's not even an Interface, modelled merely as a [typealias](https://kotlinlang.org/docs/reference/type-aliases.html):
```kotlin
typealias HttpHandler = (Request) -> Response
```
Here's a entire [**http4k**](https://http4k.org) application, which echoes request body back a the user. It only relies on the `http4k-core` module, which itself has zero dependencies:
```kotlin
val app = { request: Request -> Response(OK).body(request.body) }
val server = app.asServer(SunHttp(8000)).start()
```
The `Request` and `Response` objects in there are immutable data classes/POKOs, so testing the app requires absolutely no extra infrastructure - just call the function, so is as easy as:
```kotlin
class AppTest {
    @Test
    fun `responds as expected`() {
        assertThat(app(Request(POST, "/").body("hello")), equalTo(Response(OK).body("hello")))
    }
}
```
To plug it into a different Server-backend, just depend on the relevant module (Jetty, Undertow, Netty) and change the call to `asServer()`.

### Function 2: Filter
`Filters` provides pre and post Request processing:
```kotlin
interface Filter : (HttpHandler) -> HttpHandler
```
For discoverability reasons this is modelled as an Interface and not a [typealias](https://kotlinlang.org/docs/reference/type-aliases.html) - it also has a couple of Kotlin [extension methods](https://kotlinlang.org/docs/reference/extensions.html) to allow you to compose `Filters` with `HttpHandlers` and other `Filters`:
```kotlin
val setContentType = Filter { next ->
        { request -> next(request).header("Content-Type", "text/plain") }
    }
val repeatBody = Filter { next ->
        { request -> next(request.body(request.bodyString() + request.bodyString() }
    }
val composedFilter: Filter = repeatBody.then(setContentType)
val decoratedApp: HttpHandler = composedFilter.then(app)
```
Filters are also trivial to test independently, because they are generally just stateless functions.

### Routing
[http4k](https://http4k.org)'s nestable routing looks a lot like every other Sinatra-style framework these days, and allows for infinitely nesting `HttpHandlers` - this just exposes another `HttpHandler` so you can easily extract, test and reuse sets of routes as easily as you could with one:
```kotlin
val app: HttpHandler = routes(
    "/app" bind GET to decoratedApp,
    "/other" bind routes(
        "/delete" bind DELETE to { _: Request -> Response(OK) },
        "/post/{name}" bind POST to { request: Request -> Response(OK).body("you POSTed to ${request.path("name")}") }
    )
)
```

## Claim B. Symmetric HTTP
Out of the multitude of JVM http frameworks out there, not many actually consider how you app talks to other services, yet in this Microservice™ world that's an absolutely massive part of what many apps do!

As per a core principle behind "Server as a Function", [**http4k**](https://http4k.org) provides a symmetric API for HTTP clients - ie. it's *exactly* the same API as is exposed in [**http4k**](https://http4k.org) server applications - the `HttpHandler`. Here's that entire API again, just in case you've forgotten:
```kotlin
typealias HttpHandler = (Request) -> Response
```
What does that mean in practice? Well - for one thing, it's less for your brain to think about because you already know the API:
```kotlin
val client: HttpHandler = ApacheClient()
val response: Response = client(Request(GET, "http://server/path"))
```
For another, it means that since clients are just functions you can easily stub them for testing, and since applications and clients are interchangeable, they can be plugged together in memory without putting them on the network - which makes testing insanely fast:

```kotlin
fun MyApp1(): HttpHandler = { Response(OK) }
fun MyApp2(app1: HttpHandler): HttpHandler = { app1(it) }

val app1: HttpHandler = MyApp1()
val app2: HttpHandler = MyApp2(app1)
```
[**http4k**](https://http4k.org) provides a HTTP client adapters for both Apache and OkHttp, all with streaming support.

## Claim C. Typesafe HTTP
{{tumbleweed}}

## Claim D. Serverless
{{tumbleweed}}

## The final word(s)!
The `http4k-core` module rocks in at <1000 lines of code (about 600kb), and has zero dependencies (other than the Kotlin language itself). Additionally, everything in the core is functional and predictable - there is no static API magic going on under the covers (making it impossible to have multiple apps in the same JVM), no compiler-plugins, and no reflection. It also provides:

* Support for static file serving with HotReload
* A bunch of useful Filters for stuff like [Zipkin](http://zipkin.io/) Request Tracing
* A RequestContext mechanism for
* Facilities to record and replay HTTP traffic

There are also a bunch of other modules available, all presented with the same concentration on Testability, API simplicity and consistency:
 
* Templating engines with HotReload
* Popular JSON/XML library support
* Typesafe, multipart forms processing
* [AWS](https://aws.amazon.com/) request signing
* [Resilience4j](http://resilience4j.github.io/resilience4j/) integration, including Circuit Breakers & Rate Limiting
* Testing support via [Hamkrest](https://github.com/npryce/hamkrest) matchers and an in-memory [WebDriver](https://github.com/SeleniumHQ/selenium) implementation.

Lastly, [http4k](https://http4k.org) is proven in production, it has been adopted in at least 2 investment banks (that we know of) and is driving traffic for a major publishing website (easily serving 10s of million hits per day on a few nodes) since March 2017. You can see a few example applications [here](/in_action/).

##### Footnotes
* "But... but... but... asynchronous! And Webscale!"*, I heard them froth. Yes, you are correct - "Server as a Function" is based on asynchronous functions and [**http4k**](https://http4k.org) is synchronous. However, we tried this already and found that for 99% of apps it actually makes things harder unless you've got async all the way down. We found that this plainly didn't matter for our use-case so went for Simple™... maybe Kotlin co-routines will make this simpler - we'll see.