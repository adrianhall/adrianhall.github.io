---
title: "Lessons in Kotlin Threading: An Animated Splash Screen"
categories:
  - Mobile
tags:
  - android
  - kotlin
---

I find quite a lot of Android apps have splash screens. Some splash screens are for showing off the logo; others for hiding the extensive data load times. Whatever the reason, it’s a way for a little bit of creativity to shine through.

## My first effort

My new app needs a splash screen. Splash screens aren’t hard until you need to do something that takes some time, and there are many tutorials on how to produce one out on the Internet. I’m going to focus on the one problem I had. Let’s take a look at my first attempt at the splash screen:

{% highlight kotlin %}
class SplashActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_splash)
    }

    override fun onResume() {
        super.onResume()

        AnalyticsClient.initialize(this)
        AnalyticsClient.startSession()

        // Do any other initialization here
        Thread.sleep(5000L)

        val startupTime = System.currentTimeMillis() - ApplicationWrapper.startTime
        val event = AnalyticsClient.createEvent("startup")
              .withMetric("time", startupTime.toDouble())
        AnalyticsClient.recordEvent(event)
    } catch (e: InterruptedException) {
        e.printStackTrace()
    } finally {
        val intent = Intent(this, MainActivity::class.java)
        startActivity(intent)
    }
}
{% endhighlight %}

This is, at first review, a reasonable way to do things. Placing the startup code in the `onResume()` means that my UI will be shown to the user. However, the UI components in the layout don’t get displayed until AFTER `onResume()` has ended. The result is that my background is shown, but my nice indeterminate progress bar (aka activity spinner) is not shown.

## Add threading

For my second attempt, I dug into my Java days. I can create a thread and put the initialization code in there:

{% highlight kotlin %}
var context: Context = this
var thread = object : Thread() {
  override fun run() {
    try {
      // My code here
    } catch (e: InterruptedException) {
      e.printStackTrace()
    } finally {
      // Transition to the new activity
    }
  }
}
thread.start()
{% endhighlight %}

Does this seem, well, not Kotlin’ish enough?

## Kotlin async

Kotlin has some great async facilities. One of these is the [thread operator](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.concurrent/thread.html). You can place all the code within the thread operator instead of stuffing it in a lambda:

{% highlight kotlin %}
thread(start = true) {
    try {
        AnalyticsClient.initialize(this)
        AnalyticsClient.startSession()

        // Do any other initialization here
        Thread.sleep(5000L)

        val startupTime = System.currentTimeMillis() - ApplicationWrapper.startTime
        val event = AnalyticsClient.createEvent("startup")
                .withMetric("time", startupTime.toDouble())
        AnalyticsClient.recordEvent(event)
    } catch (e: InterruptedException) {
        e.printStackTrace()
    } finally {
        startActivity(Intent(this, MainActivity::class.java))
    }
}
{% endhighlight %}

With this code, the UI widgets get displayed, the animations start, and all the initialization happens. I don’t need to temporarily store the context so that the thread can get at it, and I’m using the in-built async commands that Kotlin has. This is a vast improvement to my original codebase.

{% include links.md %}
