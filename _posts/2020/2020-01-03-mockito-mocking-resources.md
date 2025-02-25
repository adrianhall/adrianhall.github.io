---
title: "Mocking Android resources with Mockito and Kotlin"
categories:
  - Mobile
tags:
  - android
  - kotlin
---

I bumped into an issue that was a little harder than I expected to solve, so this is the documentation.

Requirement: Load a JSON file from the `res/raw` resource directory.

Actually, that wasn't the problem.  The problem was how do you test that functionality?

## The library

I have a basic configuration library that is constructed like this:

{% highlight kotlin %}
class Configuration internal constructor(jsonContext: String): Map<String, Any> {
  internal constructor(stream: InputStream)
    : this(stream.bufferedReader(Charsets.UTF_8)).readText()

  constructor(context: Context, @RawRes resourceId: Int)
    : this(context.resources.openRawResource(resourceId))
  
  constructor(context: Context, resourcesName: String)
    : this(context, context.resources.getIdentifier(resourceName, "raw", context.packageName))

  init {
    val mapper = jacksonObjectMapper()
    configuration = mapper.readValue(jsonContent)
  }

  // Rest of my class here
}
{% endhighlight %}

All the secondary constructors call one another in a cascade.  If I provide a resource name, it looks it up, calling the constructor above it (which has a resource ID) that opens the resource before calling the constructor above it (which takes a stream) that loads the JSON, which calls the primary constructor, which parses the JSON.

I need to mock two methods within the context:

* `context.resources.openRawResource()`
* `context.resources.getIdentifier()`

With my new best friend, [Mockito](https://site.mockito.org/), this should be easy.

## Add Mockito to the build.gradle file

I use both `mockito` and [`mockito-kotlin`](https://github.com/nhaarman/mockito-kotlin) in my testing.  You will see why in a moment.  Here are my dependencies:

{% highlight gradle %}
dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    implementation "org.jetbrains.kotlin:kotlin-stdlib-jdk8:$kotlin_version"
    implementation 'androidx.core:core-ktx:1.1.0'
    implementation "com.fasterxml.jackson.module:jackson-module-kotlin:2.10.1"

    testImplementation 'junit:junit:4.12'
    testImplementation 'org.mockito:mockito-core:2.23.0'
    testImplementation "com.nhaarman.mockitokotlin2:mockito-kotlin:2.2.0"
    testImplementation "androidx.test:core:1.2.0"
}
{% endhighlight %}

## Create a BaseTest class

I tend to have a bunch of utility functions for loading JSON files and mocking.  They are used throughout the tests, so I put them in an abstract `BaseTest` class:

{% highlight kotlin %}
abstract class BaseTest {
    fun openJsonFile(filename: String): InputStream
        = javaClass.classLoader!!.getResource(filename).openStream()

    /**
     * Produces a context that supplies a resource for testing.
     */
    fun getTestContext(id: Int, resource: String): Context {
        val resources = mock<Resources> {
            on { openRawResource(eq(id)) } doReturn(openJsonFile("${resource}.json"))
            on { getIdentifier(eq(resource), eq("raw"), any()) } doReturn(id)
        }

        val context = mock<Context> {
            on { getResources() } doReturn(resources)
        }

        return context
    }
}
{% endhighlight %}

The `getTestContext()` method returns a mocked Android context that will allow you to interact with a resource.  This allows me to test different files at different times.

I can then write my test as follows:

{% highlight kotlin %}
class AzureConfigurationTest: BaseTest() {
    @Test
    fun `can load a configuration from a resource name`() {
        val context = getTestContext(1001, "flat")
        val actual = Configuration(context, "flat")
        assertNotNull(actual)
    }
}
{% endhighlight %}

The test is good, but what went wrong?

1. The `openRawResource` method always said the stream was closed when attempting to read from it.
2. The `getIdentifier` method always returned 0, indicating an error.

## Fix the mock

The problem, in both cases, is in how I constructed the mock.  There are two mechanisms for returning data through the mock.  `doReturn` is for static data.  `doAnswer` is for dynamic data.  A stream is always dynamic data.  I can adjust the `openRawResource` call as follows:

{% highlight kotlin %}
  val resources = mock<Resources> {
      on { openRawResource(eq(id)) } doAnswer {
          val file = "${resource}.json"
          openJsonFile(file)
      }
      //...
  }
{% endhighlight %}

In this interim state, I changed `eq(id)` to `any()` to see if it worked.  This was actually how I discovered that the `getIdentifier()` call was returning 0. 

The second problem was a little harder to track down.  I adjusted the mock again to the following:

{% highlight kotlin %}
  val resources = mock<Resources> {
      on { openRawResource(eq(id)) } doAnswer {
          val file = "${resource}.json"
          openJsonFile(file)
      }

      on { getIdentifier(eq(resource), eq("raw"), any()) } doAnswer {
          id
      }
  }
{% endhighlight %}

Now, set a breakpoint on the second `doAnswer` block.  It never gets executed.  Looking at the code and thinking through the process, I thought - "hmmm - I'm not mocking `context.packageName`.  What happens when the packageName is null?"  

Short version: `any()` does not match null.  So I have to also mock the packageName:

{% highlight kotlin %}
fun getTestContext(id: Int, resource: String): Context {
    val resources = mock<Resources> {
        on { openRawResource(eq(id)) } doAnswer {
            val file = "${resource}.json"
            openJsonFile(file)
        }

        on { getIdentifier(eq(resource), eq("raw"), any()) } doReturn(id)
    }

    val context = mock<Context> {
        on { getResources() } doReturn(resources)
        on { packageName } doReturn(javaClass.canonicalName)
    }

    return context
}
{% endhighlight %}

With this version, I learned two valuable lessons:

1. Use `doAnswer` during the writing process - it allows you to set breakpoints.
2. Make sure you mock everything you are going to use.  Everything else will return null.

> Yes, I looked at [Robolectric](http://robolectric.org/) for this.  While I like the idea of Robolectric for applications, it doesn't provide enough control over how the test files are injected for me to use it.  Mockito was just easier.
