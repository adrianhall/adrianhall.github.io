---
title: "Why I'm refactoring to Kotlin"
categories:
  - Mobile
tags:
  - android
  - kotlin
---

One of the things that is really cool about software development is that we are continually learning and adjusting. I spent a day recently learning [Kotlin](https://kotlinlang.org/). I was so impressed with the language, I decided to refactor my entire Family Photos app. It did not take long.

## Why do I love Kotlin?

I have to admit, I was skeptical at first. After all, I tried Swift for iOS as an alternate language and it felt foreign. Admittedly, Objective-C isn’t much better, but Swift definitely had some issues for me. Maybe I’m just not an iOS person. For me, Kotlin is more readable. It’s more readable than Swift and more readable than Java. Let’s have a look at some of the basics:

### val for constants and var for variables

I get this in about 10 seconds. I think I’m a little slow on the uptake. `val` is a value and `var` is a variable. You get type inference and where that isn’t possible, you can specify a type. Kotlin is still strongly typed, so everything has a type. So, for example, here is equivalent code in Java:

{% highlight java %}
private static final String TAG = "MainActivity";
private Timestamp lastUpdated = new Timestamp(model.getTime());
public String wtfError = null;
{% endhighlight %}

And the same thing in Kotlin:

{% highlight kotlin %}
private val TAG = "MainActivity"
private var lastUpdated = Timestamp(model.time)
public var wtfError: String? = null
{% endhighlight %}

No reduction in lines, but much more readable. I love the getter/setter inference and the fact that I can specify the type when necessary. I’m also a fan of the null safety. I have to declare something can take null — it isn’t taken for granted.

I’m not a fan of the lack of semi-colons. I like to know when a statement has ended. There are worse languages for this (hello, Python) and it isn’t a show stopper for me.

### Easy callbacks (aka lambdas)

In Android, you do callbacks all. the. time. It’s a fact of life, and it is mostly because Java hasn’t come out of the dark ages and embraced async/await or even promises — two things that C# and JavaScript get right. The problem with Java is that the code is so verbose. For instance, here is some code for dealing with getting an Amazon Pinpoint client:

{% highlight java %}
AWSMobileClient.initialize(context, new AWSStartupCallback() {
  @Override
  public void onSuccess(AWSStartupResult result) {
    /* do something */
  }
}).execute();
{% endhighlight %}

{% highlight kotlin %}
AWSMobileClient.initialize(context, { _ -> {
  override fun onSuccess(result) {
    /* do something */
  }
}).execute()
{% endhighlight %}

That callback is called a lambda or function expression. In fact, since the callback object only has one callback function, I can even further simplify this:

{% highlight kotlin %}
AWSMobileClient.initialize(context, { _ -> { /* Do Something */} })
  .execute()
{% endhighlight %}

If there is more than one callback function, you have to specify all of them.

### Better Type Safety

You don’t see this, except very occasionally. I’m setting up Amazon Pinpoint and there is a possibility that the `pinpointManager` is not set up before I decide to click back and destroy the activity, but the `onDestroy()` method has something to record an event to the `pinpointManager`. In Java:

{% highlight java %}
@Override
public void onDestroy() {
  super.onDestroy();
  if (pinpointManager != null) {
    pinpointManager.recordStopEvent();
  }
}
{% endhighlight %}

That guard around the `recordStopEvent()` call is necessary because if it isn’t there, I leave open the prospect of a crash. If the app crashes on something as inconsequential as analytics, there is a direct user impact. Fortunately, Kotlin comes with override checking, nullability checks, and generally better type safety. My new code is this:

{% highlight kotlin %}
override public fun onDestroy() {
  super.onDestroy()
  pinpointManager?.let { it.recordStopEvent() }
}
{% endhighlight %}

If I have multiple calls (for example, I may have to do a `submitEvents()` call as well), it's just as easy:

{% highlight kotlin %}
override public fun onDestroy() {
  super.onDestroy()
  pinpointManager?.let {
    it.recordStopEvent()
    it.submitEvents()
  }
}
{% endhighlight %}

We can even get more concise:

{% highlight kotlin %}
override public fun onDestroy() {
  super.onDestroy()
  pinpointManager?.recordStopEvent()
}
{% endhighlight %}

It's nice and concise while still being readable and produces the intent of my original code.

### The == operator does what you expect

Ok, this is just a placeholder for “Kotlin fixed all the annoyances of Java”. When I say ==, I mean “this object is equal to another”. Take these:

{% highlight java %}
String a, b;
// Do some work with a and b
if (a == b) { /* Never */ }
if (a.equals(b)) { /* Possibly */ }
{% endhighlight %}

`a` and `b` are two different strings. If I use the `==` operator, they will never be equal even if they hold the same value. That’s because `==` in Java assumes referential equality — the two objects are actually the same object. Let’s look in Kotlin:

{% highlight kotlin %}
var a = "foo", b = "foo"
if (a == b) { /* Definitely */ }
if (a === b) { /* Never */ }
{% endhighlight %}

Here, the `==` operator does what you expect. There is a completely different operator for referential integrity.

It isn’t the only place where Kotlin has done “the right thing”. They’ve liberally borrowed from other languages when those other languages have done the right thing. See how [Kotlin does generics](https://kotlinlang.org/docs/reference/generics.html), for example. Note how similar it is to C#?

### Async promises

Nothing annoys me more than async callbacks in Java. Fortunately, [Kotlin has an answer](http://kovenant.komponents.nl/). Ok, so it’s not in the core Kotlin libraries, but neither are a lot of Java things either:

{% highlight kotlin %}
task {
    //some (long running) operation, or just:
    1 + 1
} then {
    i -> "result: $i"
} success {
    msg -> println(msg)
}
{% endhighlight %}

**Edit**: Antonio D’Souza pointed out that [Java8 does in fact have promises](http://www.deadcoderising.com/java8-writing-asynchronous-code-with-completablefuture/), but they are called `CompletableFuture`. I still like the syntax of Kotlin for async work better.

### Kotlin has better community support

Ok, this might be arguable. When I look at the number of people developing in Kotlin vs. Java, this shouldn’t happen. Kotlin developers are more community focused than Java developers. They love Kotlin and aren’t shy about sharing that love.

However, I’m going to point to [Awesome Kotlin](https://github.com/KotlinBy/awesome-kotlin) and call it a day. Seriously — I spent three hours just on this one site. It was primarily why Kotlin took a day to learn. I could probably have been done with my first app in a couple of hours if it hadn’t been for the wealth of information out there that just kept drawing me in.

## Wrap Up

In two years time, all new Android apps will be written in Kotlin. It’s easier to learn, easier to read and now, with the [introduction of Kotlin projects in Android Studio 3.2](https://android-developers.googleblog.com/2017/11/update-on-kotlin-for-android.html), it’s easier to use.

{% include links.md %}
