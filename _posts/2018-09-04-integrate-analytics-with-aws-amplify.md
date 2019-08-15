---
title: "Integrate Analytics into your Android applications with AWS Amplify"
categories:
  - Android
tags:
  - Kotlin
---

I’ve become a big fan of [Kotlin development](https://kotlinlang.org/) for my Android apps. I also think that analytics should be integrated into every single app I write. I’ve covered integrating Amazon Pinpoint before via AWS Mobile Hub. Recently, [AWS Amplify] announced an updated CLI that provides support for native applications and bypasses the need to use the AWS web console for creating resources. Naturally, [I wanted to try it out]({% post_url 2018-08-27-native-development-with-aws-amplify %}). Integrating analytics is now really easy. Here is how I do it:

## Start Local

You don’t need to create any services in the cloud to get started with analytics. I start by defining an interface for recording analytics events:

{% highlight kotlin %}
interface AnalyticsService {
  fun recordEvent(eventName: String, attributes: Map<String,String>? = null, metrics: Map<String,Double>? = null)
}
{% endhighlight %}

Where there is an interface, there is also a concrete implementation:

{% highlight kotlin %}
class LocalAnalyticsService : AnalyticsService {
  override fun recordEvent(eventName: String, attributes: Map<String,String>?, metrics: Map<String,Double>?) {
    val event = StringBuilder("")
    attributes?.let {
      for ((k, v) in it) { event.append(", $k=\"$v\"") }
    }
    metrics?.let {
      for ((k, v) in it) { event.append(", $k=${String.format("%.2f",v)}") }
    }
    if (event.isNotEmpty())
      event[0] = ':'
    Log.v(TAG, "recordEvent($eventName)$event")
  }
}
{% endhighlight %}

Before I can use it, I have to instantiate a version of it. This should be a singleton, since many cloud services need a singular context to handle authentication and to minimize the number of outbound connections that are created. This isn’t using a cloud service yet, but I am going to do that later. I use [Koin dependency injection](https://insert-koin.io/) throughout my app for dealing with this, but you can use a regular singleton pattern, [Dagger2](https://google.github.io/dagger/), [Kodein](http://kodein.org/Kodein-DI/), or any other DI framework. In my case, I edit my application entry point to add the analytics service to the DI module:

{% highlight kotlin %}
private val servicesModule : Module = applicationContext {
  bean { LocalAnalyticsService() as AnalyticsService }
}
{% endhighlight %}

To use it, I edit my activities to inject the analytics service:

{% highlight kotlin %}
private val analyticsService : AnalyticsService by inject()
{% endhighlight %}

I then liberally sprinkle my code with `recordEvent()` calls. In this case, they become debug log statements that I can monitor while I am developing the app.

## Build an Analytics Service with Amazon Pinpoint

Building a backend service for your Android or iOS app is easy with the [AWS Amplify] CLI.  First, [install and configure the AWS Amplify CLI][amplify-get-started].   There are detailed instructions plus a video on the AWS Amplify website.

Open a terminal and change directory to your project directory (which is the one with top-level `build.gradle` file). First, initialize the project:

{% highlight bash %}
amplify init
{% endhighlight %}

You’ll be guided through the process, which includes naming the project, confirming that you are developing an Android project and choosing appropriate credentials for deploying resources to AWS. This command will create an amplify directory to store the details of the backend. Now, let’s add an Amazon Pinpoint resource for storing analytics:

{% highlight bash %}
amplify add analytics
{% endhighlight %}

You can press enter to progress through the process. All the defaults are good for this app. Finally, let’s deploy the resources:

{% highlight bash %}
amplify push
{% endhighlight %}

You’ll be asked to confirm the deployment and then the resources are created. More importantly, this will create the `awsconfiguration.json` file in the `res/raw` directory. This JSON file contains a description of the resources that you can access via your app. Without this file, you would have to log into the AWS console and take a look at each resource, copying the access information from the console into a constants file, and then use those constants to initialize the SDK. This is a major friction point for developing the app.

With the AWS Amplify CLI, the `awsconfiguration.json` is maintained in the proper location for you.

## Add the AWS SDK to your project

The AWS SDK contains all the code necessary to connect to AWS cloud resources, including [Amazon Pinpoint]. You can easily download the libraries using gradle. I add the following to my dependencies:

{% highlight gradle %}
dependencies {
  // Other dependencies here
  implementation 'com.amazonaws:aws-android-sdk-pinpoint:2.6.29'
  implementation ('com.amazonaws:aws-android-sdk-mobile-client:2.6.29@aar') { transitive = true }
}
{% endhighlight %}

You will also need to add the following permissions to your `AndroidManifest.xml` file:

{% highlight xml %}
<uses-permission android:name="android.permission.INTERNET"/>
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>
{% endhighlight %}

## Create a new AWSAnalyticsService class

We can now replace the `LocalAnalyticsService` with one that talks to Amazon Pinpoint:

{% highlight kotlin %}
import android.content.Context
import com.amazonaws.mobile.auth.core.IdentityManager
import com.amazonaws.mobile.config.AWSConfiguration
import com.amazonaws.mobile.samples.picturefeed.services.AnalyticsService
import com.amazonaws.mobileconnectors.pinpoint.PinpointConfiguration
import com.amazonaws.mobileconnectors.pinpoint.PinpointManager

class AWSAnalyticsService(context: Context): AnalyticsService {
    private var manager: PinpointManager? = null

    init {
        val awsConfiguration = AWSConfiguration(context)
        val identityManager = IdentityManager(context, awsConfiguration)
        val config = PinpointConfiguration(context, identityManager.credentialsProvider, awsConfiguration)
        manager = PinpointManager(config)
        manager?.sessionClient.startSession()
        manager?.analyticsClient.submitEvents()
    }

    /**
     * Record a custom event into the analytics stream
     *
     * @param name the custom event name
     * @param [attributes] a list of key-value pairs for recording string attributes
     * @param [metrics] a list of key-value pairs for recording numeric metrics
     */
    override fun recordEvent(name: String, attributes: Map<String, String>?, metrics: Map<String, Double>?) {
        manager?.let {
            val event = it.analyticsClient.createEvent(name)
            for ((k, v) in attributes.orEmpty()) {
                event.addAttribute(k, v)
            }
            for ((k,v) in metrics.orEmpty()) {
                event.addMetric(k, v)
            }
            it.analyticsClient.recordEvent(event)
            it.analyticsClient.submitEvents()
        }
    }
}
{% endhighlight %}

The `AWSConfiguration` and `IdentityManager` objects should be shared among all AWS service libraries. If you are doing more than analytics, then you should probably abstract those into their own class that is injected into this class. I automatically record a start-session event so I can always get usage analytics.

You can easily inject this using Koin:

{% highlight kotlin %}
private val servicesModule : Module = applicationContext {
  bean { AWSAnalyticsService(get()) as AnalyticsService }
}
{% endhighlight %}

Here, the `get()` will inject the application context for me.

## Check out the analytics

Once you run your app and do some stuff to generate appropriate analytics traffic, you can open up the Pinpoint console:

{% highlight bash %}
amplify analytics console
{% endhighlight %}

You will probably be asked to log in to the console — just use your regular AWS crednetials. You will be placed directly in the analytics tab and can start exploring the graphs that are provided out of the box.

## Recording other events

Amazon Pinpoint also allows you to [record revenue events](https://aws.github.io/aws-sdk-android/docs/reference//index.html?com/amazonaws/mobileconnectors/pinpoint/analytics/monetization/MonetizationEventBuilder.html). There are several versions — for Google Play, Amazon and custom events. You can also make your own version up. This is good if you are doing in-app purchases. Just add a method to the interface and concrete implementation. You can then get a live update (on the Pinpoint console) of how much money you made today (or last week, month or year).

There are specific events that you can record for authentication events:

* `_userauth.sign_in`
* `_userauth.sign_up`
* `_userauth.authfail`

This allows you to record some information about authentication events for both Amazon Cognito and 3rd party authentication libraries using just the `recordEvent()` method I presented here. I’ll be doing [a series of in-depth authentication articles with AWS Amplify]({% post_url 2018-09-18-auth-with-aws-amplify-1 %}) later on, so stay tuned for that.

My next article will show off how to record user-specific information (for example, tagging with a username or location) and user-provided information (for example, topics of interest) so that we can produce segments of users, all with the aim of doing marketing campaigns and communicating with the user base. Until that point, start getting into cloud backends with [AWS Amplify].

{% include links.md %}
