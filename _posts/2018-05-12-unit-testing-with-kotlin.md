---
title: "Unit Testing in Android Studio with Kotlin"
categories:
  - Android
tags:
  - Kotlin
  - Testing
---

I made myself a promise a couple of months ago. My next app would be fully unit tested for the non-UI components and fully instrument-tested for the UI components. That’s a tall order, especially since I’m using the latest and greatest Android Studio Canary. I ran into a few problems. Some can’t be solved and some can.

Let’s start with a simple test. I’ve got a class that looks this:

{% highlight kotlin %}
data class Note(val id: String = UUID.randomUUID().toString()) {
  var updated: Long = System.currentTimeMillis()
    private set(value) { field = value }
  var title: String = ""
    set(value) {
      updated = System.currentTimeMillis()
      field = value
    }
  // Some more fields here
  init {
    if (id.isBlank()) throw IllegalArgumentException()
  }
}
{% endhighlight %}

This is a fairly simple model class, but I wanted to ensure that the updated property was automatically updated when I set the title. So I wrote a unit test:

{% highlight kotlin %}
package com.amazonaws.mobile.samples.mynotes.models

import org.junit.Test
import org.junit.Assert.*
class NoteUnitTests {
  @Test
  fun `setting the title sets the updated date`() {
    val note = Note()
    val updated = note.updated
    Thread.sleep(100L)
    note.title = "test"
    assertNotSame("T10-1", updated, note.updated)
    assertTrue("T10-2", note.updated > updated)
  }
}
{% endhighlight %}

It’s in among some 21 tests each with a few assertions to check. The assertion message is just unique to the assert so that I can find it in larger test suites.

Good things:

* Note how I can name the test something reasonable. I love this feature. You can’t use dots or other special characters in the function name, but pretty much anything else goes. This is also how it is reported.
* I can right-click on the unit test and run-debug and get the test results, set breakpoints, etc.

Ok, that’s good, but there are a lot of bad things.

## Configuring JUnit 5

Firstly, I wanted to switch to JUnit 5, which came out in 2016. Yep — that’s over 2 years ago. There still isn’t a standard way to just pick JUnit 5. What are they waiting for over in Android Studio land?

Fortunately, community members have stepped up. Here is how you get Android Studio (and your project) to use JUnit 5. First, edit the top-level `build.gradle` file and add the following classpath (with the others):

{% highlight gradle %}
classpath "de.mannodermaus.gradle.plugins:android-junit5:1.0.32"
{% endhighlight %}

This is a plugin for JUnit5 that really aids in setup. Now, edit your app `build.gradle` file. Add the following plugin at the top:

{% highlight gradle %}
apply plugin: "de.mannodermaus.android-junit5"
{% endhighlight %}

Then remove the existing test implementations and add these ones instead:

{% highlight gradle %}
// Testing
testImplementation junit5.unitTests()
testImplementation junit5.parameterized()
androidTestImplementation junit5.instrumentationTests()
{% endhighlight %}

Finally, make sure you set up JDK 1.8. While you are editing the app `build.gradle`, add the following to the android closure:

{% highlight gradle %}
compileOptions {
    sourceCompatibility JavaVersion.VERSION_1_8
    targetCompatibility JavaVersion.VERSION_1_8
}
{% endhighlight %}

Then change the Kotlin dependency to this:

{% highlight gradle %}
// Kotlin
implementation "org.jetbrains.kotlin:kotlin-stdlib-jdk8:$kotlin_version"
{% endhighlight %}

Clean and rebuild. Your unit and instrumented tests will now be using JUnit 5 as the test runner. This is in large part due to the work of Marcel Schnelle, who wrote the plugin. The instructions are on his GitHub repository.

JUnit 5 isn’t a big deal in Android development. JUnit 4 works just fine, thank you. However, JUnit5 includes support for JDK 1.8. A lot of people write their tests in Java (more on why in a moment), so you may want to write Java lambda expressions, as an example. You also get tagging, disabled tests, better exception testing, and much more. For a good overview of the new features, see [this article on baeldung.com](http://www.baeldung.com/junit-5-preview).

## The problems

So, what’s not so great:

* Android Studio Canary has a bug (say it ain’t so!) that prevents code coverage from running within the UI. Various people have suggested fixes for this on Stack Overflow. However, all the suggested remedies have failed me.
* You can’t run all the tests in the app from the UI. That means you don’t get the nice IDE-driven pass/fail test with debug/run and breakpoints. If you run all the tests, do it from the terminal with `./gradlew test`.
* When you run a single package (or directory) worth of tests, the IDE generates a new configuration. That causes a proliferation of configurations (one for each class you run individually plus one for each package).
* When the tests finish, the task does not stop. If you go over to the stop button, you may notice several tasks running — one for each test run you ran. They don’t stop when you click Stop All either. You have to restart the IDE.
* It looks (and feels) like there is a major memory leak in the IDE functionality associated with testing. I’ve noticed that I regularly run out of memory (despite increases) when I am in the process of writing code with TDD (where I write the tests first then write the code and continually run the tests until all the tests pass). Again, shutting down and restarting the IDE seems to be the only way to fix this.

Android Studio Canary is bleeding edge, so these things will happen. It’s definitely not a polished product yet. However, if you are writing Kotlin apps, you need to be on Canary as that is where all the good stuff is happening on a regular basis.

## Wrap up

Despite these setbacks, I am now finding writing tests before the code is starting to be second nature and I have much more confidence in the code I am writing. My general process is:

* Sit down and think and write a rough specification.
* Write the tests to exercise the specification.
* Write stubs for the code.
* Run the tests from the command line — yay! everything fails!
* Start writing the code, augmenting the tests where I feel it’s a good idea to catch edge cases. (Don’t reduce the number of tests — always increase)
* Run the tests again, potentially with debug to step through the code to see where I’ve been an idiot.
* Repeat the last two steps as often as is required.

I wrote my first class (the data model for my app) before the tests and before I had thought about what I wanted to do with it. It didn’t work out too well, so I had to delete it, write the spec and then the tests.

Don’t let the problems with Android Studio prevent you from testing. Get in the habit of unit testing your code even for the simple projects. It will make you a better coder. I know it has improved my coding ability.

{% include links.md %}
