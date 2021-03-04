---
title: "Unit testing asynchronous Android network libraries"
categories:
  - Android
tags:
  - Kotlin
---

I'm writing a network library for Android at the moment, and specifically looking at unit tests.  In [my last article]({% post_url 2019/2019-12-24-unit-testing-android-libraries %}), I looking at mocking the Android context and other Android specific libraries.  Since I am writing a network client library, I need to go a step further and deal with the network connection itself.  

How can I test the asynchronous network calls in a repeatable manner?

Fortunately, there's a library for that.  Square, the same people that brought you [OkHttp](https://square.github.io/okhttp/), also produce a mock web server that you can use to mock the network connection.  Since I've already got the relevant JSON (which I use to test the model decoding), I can use that to produce a fake service.

Doing this requires an understanding of three steps:

1. Adjust your library so it is testing friendly.
2. Create a mock web service.
3. Deal with the asynchronous testing.

## Make your library testing friendly

My library always connects to the same URL and uses the context to grab the API key.  I'm expecting my users to call something like:

```kotlin
val client = NetworkClient(context)
```

This is not testing friendly because I cannot mock everything - including the server.  However, I can provide a public constructor that calls an internal constructor, like this:

```kotlin
class NetworkClient internal constructor(
    context: Context,
    apiKey: String? = null,
    serviceUri: String
) {
    companion object {
        /**
         * The base URI for our calls when we aren't mocking
         */
        private const val BASE_URI = "https://my.baseurl.com"
    }

    constructor(context: Context, apiKey: String? = null): this(context, apiKey, BASE_URI)

    // ...
}
```

The user will call the secondary constructor.  If they are relying on Intellisense within Android Studio, then that is the only constructor that they will see.  In the mean time, my tests are in the same package as the class under test, so the tests can use the internal constructor.  This allows me to adjust the `serviceUri` according to needs.

You should make all your internal properties "internal" as well.  This allows you to ensure that they are set properly.  For example, let's say you have mocked the context.  You can do the following test:

```kotlin
@Test
fun test_context_constructor() {
    val apiKey = "whatever-your-api-key-is"
    val context = getMockContext(apiKey)
    val client = NetworkClient(context)
    assertEquals(apiKey, client.internalApiKey)
}
```

Now that you know your client construction is good, you don't need to test that part of it later on.

## Create a mock web service

This gets to the code for using that `serviceUri` parameter to mock the server.  First, add the `mockwebserver` package as a test dependency:

```gradle
dependencies {
    testImplementation "com.squareup.okhttp3:mockwebserver:4.2.1"
}
```

Then, add a method to create an appropriate web service locally.  I have a "response.json" file that contains a valid response.  Here is my method for creating a custom web service:

```kotlin
private fun createServerAndEnqueue(path: String: MockWebServer {
    val server = MockWebServer()
    val response = MockResponse()
        .setResponseCode(200)
        .setBody(readJsonFromFile(path))
    server.enqueue(response)

    return server
}
```

When you connect to the server and send it a request, it will respond with the queued response - in this case, a 200 OK with a JSON body.  It doesn't matter what request you send to the service - you will always get the same thing.

Now you can use this in a test:

```kotlin
@Test
fun test_network_request_with_callback() {
    val server = createServerAndEnqueue()
    val client = createGoodClient(server)

    client.callNetwork(params) { response -> 
        // Ensure my library thinks request is successful
        assertTrue(response.isSuccessful)
        assertEquals(200, response.httpStatusCode)
        // Ensure exactly 1 network request took place
        assertEquals(1, server.requestCount)

        // Check what request was actually sent
        val request = server.takeRequest()
        val url = request.requestUrl!!

        // Do other checks as neccessary here
    }
}
```

What other checks?  That depends on the requirements of your library.  My library ensures that the options I pass in are encoded properly, and that the request is turned into the appropriate URL.  The `requestUrl` is a `HttpUrl` object, which is part of the okhttp3 library.

This doesn't work.  The asynchronous worker is not finished by the time the assertions are done.  You have to deal with the async nature of the API.

## Deal with asynchronous tests

JUnit tests generally run synchronously.  How do you ensure that your test runs to completion (with a timeout, obviously) when you are running in an asynchronous context?  Fortunately, we have the capabilities of Kotlin coroutines in the tests, which makes this remarkably easy.  For a good writeup on this topic, check out the [documentation on replacing callbacks with suspendCoroutine](https://github.com/Kotlin/KEEP/blob/master/proposals/coroutines.md#wrapping-callbacks).

I have a method in my client class with the following signature:

```kotlin
fun doNetworkRequest(options: OptionsBag, callback: (ClientResponse) -> Unit)
```

I call the `doNetworkRequest` method, and it calls my callback when it is completed.  This is done on a background thread so my method carries on executing in the background.

Tests are run synchronously.

The easiest mechanism to support testing is to use a wrapper to place the request into a suspending coroutine:

```kotlin
suspend fun doNetworkRequestSync(client: NetworkClient, options: OptionsBag): ClientResponse
    = suspendCoroutine { cont -> client.doNetworkRequest(options) { cont.resume(it) } }
```

You can put this in your test class so it doesn't pollute the namespace of your network client class.  Similarly, if your network request returns a `LiveData<T>`, you can observe the `LiveData` and then call `cont.resume` when you have a response:

```kotlin
suspend fun doNetworkRequestSync(client: NetworkClient, options: OptionsBag): ClientResponse
    = suspendCoroutine { cont ->
        val observable = client.doNetworkRequest(options)
        observable.observeForever { cont.resume(it) }
    }
```

This is the "simple" version.  However, you may run into problems when you have many tests.  Each observable is observed forever, which isn't what you want.  You want to detach the observer when the response has been received.

To run a test, you need to create a coroutine launcher.  Fortunately, there is a package for this, so add the following to your dependencies:

```gradle
testImplementation 'org.jetbrains.kotlinx:kotlinx-coroutines-core:1.3.0'
```

What does a test look like?

```kotlin
@Test
fun client_location_options_callback() {
    val server = createServerAndEnqueue()
    val client = createGoodClient(server)
    val location = createLocation(46.6062, -122.3321)

    runBlocking {
        val response = doNetworkRequestSync(client, location)

        assertTrue(response.isSuccessful)
        assertEquals(200, response.httpStatusCode)
        assertEquals(1, server.requestCount)

        val request = server.takeRequest()
        val url = request.requestUrl

        assertEquals("GET", request.method)
        // Do any other asserts you want here
    }
}

suspend fun doNetworkRequestSync(client: NetworkClient, location: Location): NetworkResponse
    = suspendCoroutine { cont -> client.doNetworkRequest(location) { cont.resume(it) }}
```

With these notes, I can now finish off the tests for my network client and appropriately test failure and success conditions with a real HTTP request.
