---
title: "Implement Search-on-type in Android with RxJava"
categories:
  - Android
tags:
  - Kotlin
---

I’m working on a new sample which, as is typical, communicates with a backend service for data through a serverless API. In this particular example, it’s a search capability that I am developing and one of the features I want to implement is “search while you type”.

Not a problem, you might think. Just put a search box on the page (probably in the action bar), wire up the `onTextChange` event handler, and do the search. So, that’s what I did:

{% highlight kotlin %}
override fun onCreateOptionsMenu(menu: Menu?): Boolean {
    menuInflater.inflate(R.menu.menu_main, menu)
    val searchView = menu?.findItem(R.id.action_search)?.actionView as SearchView

    // Set up the query listener that executes the search
    searchView.setOnQueryTextListener(object : SearchView.OnQueryTextListener {
        override fun onQueryTextSubmit(query: String?): Boolean {
            Log.d(TAG, "onQueryTextSubmit: $query")
            return false
        }

        override fun onQueryTextChange(newText: String?): Boolean {
            Log.d(TAG, "onQueryTextChange: $newText")
            return false
        }
    })

    return super.onCreateOptionsMenu(menu)
}
{% endhighlight %}

## The problem

If I am doing “search-on-type”, then whenever the `onQueryTextChange()` event handler fires, I will kick off an API call to return the first set of results. The log looks like this:

```
D/MainActivity: onQueryTextChange: T
D/MainActivity: onQueryTextChange: TE
D/MainActivity: onQueryTextChange: TES
D/MainActivity: onQueryTextChange: TEST
D/MainActivity: onQueryTextSubmit: TEST
```

Even though I’m just typing, I would kick off five API calls, each of which would do a search. In the serverless cloud, you pay for executions — i.e. API calls. If I am just pressing buttons to complete my search term, I want to de-bounce this and only do one API call.

Now let’s say I want to search for something else. I delete the TEST and type something else:

```
D/MainActivity: onQueryTextChange: TES
D/MainActivity: onQueryTextChange: TE
D/MainActivity: onQueryTextChange: T
D/MainActivity: onQueryTextChange:
D/MainActivity: onQueryTextChange: S
D/MainActivity: onQueryTextChange: SO
D/MainActivity: onQueryTextChange: SOM
D/MainActivity: onQueryTextChange: SOME
D/MainActivity: onQueryTextChange: SOMET
D/MainActivity: onQueryTextChange: SOMETH
D/MainActivity: onQueryTextChange: SOMETHI
D/MainActivity: onQueryTextChange: SOMETHIN
D/MainActivity: onQueryTextChange: SOMETHING
D/MainActivity: onQueryTextChange: SOMETHING
D/MainActivity: onQueryTextChange: SOMETHING E
D/MainActivity: onQueryTextChange: SOMETHING EL
D/MainActivity: onQueryTextChange: SOMETHING ELS
D/MainActivity: onQueryTextChange: SOMETHING ELSE
D/MainActivity: onQueryTextChange: SOMETHING ELSE
D/MainActivity: onQueryTextSubmit: SOMETHING ELSE
```

20 API calls! Most of these will get taken care of by the “de-bounce”. I also want to de-duplicate so that the trimmed text does not cause duplicate submissions. Also, this is a search page, so I probably want to filter out some items. For example, do I want to allow blank searches? How about short searches (one letter)?

There are a couple of things that I can do here, but the one I want to concentrate on right now is a technique broadly known as reactive programming and the RxJava library. When I first got introduced to reactive programming, I saw this description:

> [ReactiveX](http://reactivex.io/) is an API that focuses on asynchronous composition and manipulation of observable streams of data or events by using a combination of the Observer pattern, Iterator pattern, and features of Functional Programming.

That totally doesn’t explain what it does or its power. Well, it does — but only to those who already know what it does. I also saw diagrams like this:

![Marble chart courtesy of ReactiveX]({{ site.baseurl }}/assets/images/2018/2018-06-20-image1.png){: .center-image}

Which explains the role of the operator, but not so much on how to set things up. So let’s see if I can do a better job by way of a simple example that’s also useful.

## Configure the project

Let’s get the project ready first. You are going to need a new library in your app `build.gradle` dependencies:

{% highlight gradle %}
implementation "io.reactivex.rxjava2:rxjava:2.1.14"
{% endhighlight %}

Don’t forget to sync the project so that the library is downloaded.

## Create an observable stream

Now, on to our new technique. Under the old method, I called the API whenever a new character was typed. In the new version I am going to create an `Observable` and submit to that:

{% highlight kotlin %}
override fun onCreateOptionsMenu(menu: Menu?): Boolean {
    menuInflater.inflate(R.menu.menu_main, menu)
    val searchView = menu?.findItem(R.id.action_search)?.actionView as SearchView

    // Set up the query listener that executes the search
    Observable.create(ObservableOnSubscribe<String> { subscriber ->
        searchView.setOnQueryTextListener(object : SearchView.OnQueryTextListener {
            override fun onQueryTextChange(newText: String?): Boolean {
                subscriber.onNext(newText!!)
                return false
            }

            override fun onQueryTextSubmit(query: String?): Boolean {
                subscriber.onNext(query!!)
                return false
            }
        })
    })
    .subscribe { text ->
        Log.d(TAG, "subscriber: $text")
    }

    return super.onCreateOptionsMenu(menu)
}
{% endhighlight %}

This code does exactly the same thing as the old code — the log looks like this:

```
D/MainActivity: subscriber: T
D/MainActivity: subscriber: TE
D/MainActivity: subscriber: TES
D/MainActivity: subscriber: TEST
D/MainActivity: subscriber: TEST
```

However, the major difference is that we have a reactive stream to play with. The stream is an `Observable`. The text handler (or in this case the query handler) submits elements into the stream using `onNext()`. The observable has subscribers that consume those elements (after whatever pipeline we have deemed appropriate has been cleared).

We can place a chain of methods in front of the `subscribe` call to adjust the list of strings that the `subscribe` method sees.

## Working with the stream

Let’s start by adjusting our stream so that the text that is submitted is always lower-case and it is trimmed of whitespace at the beginning and end:

{% highlight kotlin %}
Observable
        .create(ObservableOnSubscribe<String> {
          // The same code here
        })
        .map { text -> text.toLowerCase().trim() }
        .subscribe { text -> Log.d(TAG, "subscriber: $text" }
{% endhighlight %}

I’ve shortened the methods to show just the relevant bit. Now the same log looks like this:

```
D/MainActivity: subscriber: t
D/MainActivity: subscriber: te
D/MainActivity: subscriber: tes
D/MainActivity: subscriber: test
D/MainActivity: subscriber: test
```

Next, let’s de-bounce the stream by waiting for more content for up to 250ms:

{% highlight kotlin %}
Observable
        .create(ObservableOnSubscribe<String> {
          // The same code here
        })
        .map { text -> text.toLowerCase().trim() }
        .debounce(250, TimeUnit.MILLISECONDS)
        .subscribe { text -> Log.d(TAG, "subscriber: $text" }
{% endhighlight %}

Next, let’s de-duplicate the stream so that only the first unique request gets processed — subsequent identical requests will get ignored

{% highlight kotlin %}
Observable
        .create(ObservableOnSubscribe<String> {
          // The same code here
        })
        .map { text -> text.toLowerCase().trim() }
        .debounce(250, TimeUnit.MILLISECONDS)
        .distinct()
        .subscribe { text -> Log.d(TAG, "subscriber: $text" }
{% endhighlight %}

Finally, let's filter out blank requests:

{% highlight kotlin %}
Observable
        .create(ObservableOnSubscribe<String> {
          // The same code here
        })
        .map { text -> text.toLowerCase().trim() }
        .debounce(250, TimeUnit.MILLISECONDS)
        .distinct()
        .filter { text -> text.isNotBlank() }
        .subscribe { text -> Log.d(TAG, "subscriber: $text" }
{% endhighlight %}

At this point, you will note that you only get one (or maybe two) log messages, resulting in less API calls to your backend. Yet your app will remain responsive. In addition, the case where you type something and then delete it to type something else will also result in less API calls.

There are [more operators you can add](http://reactivex.io/documentation/operators.html) to this pipeline for different needs. I find these ones are good for dealing with input fields that do something on an API. The complete code looks like this:

{% highlight kotlin %}
// Set up the query listener that executes the search
Observable
    .create(ObservableOnSubscribe<String> { subscriber ->
        searchView.setOnQueryTextListener(object : SearchView.OnQueryTextListener {
            override fun onQueryTextChange(newText: String?): Boolean {
                subscriber.onNext(newText!!)
                return false
            }

            override fun onQueryTextSubmit(query: String?): Boolean {
                subscriber.onNext(query!!)
                return false
            }
        })
    })
    .map { text -> text.toLowerCase().trim() }
    .debounce(250, TimeUnit.MILLISECONDS)
    .distinct()
    .filter { text -> text.isNotBlank() }
    .subscribe { text ->
        Log.d(TAG, "subscriber: $text")
    }
{% endhighlight %}

I can now replace the log message with a call to my viewModel to initiate the API call. That’s a subject for another blog post, however.

With this simple technique of wrapping your text controls in an observable and using RxJava, you can reduce the number of API calls you make for doing backend operations and improve the responsiveness of your app. This is just scratching the surface of the RxJava world, so I’ll leave you with some additional reading:

* [Grokking RxJava](http://blog.danlew.net/2014/09/15/grokking-rxjava-part-1/) by Dan Lew. (this is the site that got me pointed in the right direction)
* The [ReactiveX website](http://reactivex.io/). (I refer to this one often when constructing pipelines)

There are also additional libraries for Android and data binding with Kotlin available. I’ll cover those another time.

{% include links.md %}
