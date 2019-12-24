---
title: "Two tips for unit testing Android libraries"
categories:
  - Android
tags:
  - Kotlin
---

I'm busy writing a networking library for one of my Android apps.  The question today is "how do I properly test the library?"  There are a few problem areas, and I'll cover two of them today.

1. How do I properly mock classes that aren't really Android specific, like `android.location.Location` and `android.net.Uri`?
2. How do I properly mock the Android context?

Android Studio already integrates the excellent JUnit packages for unit testing.  I don't want to have to write a visual app just to test my library. 

## Tip #1: Use Unmock to mock Android library classes

There are quite a few classes in the Android standard library that are not Android specific.  When you consider it, the `android.location.Location` is storage for the result of an Android operation (finding location via the built in GPS), but it doesn't touch the Android OS.  My library takes a `Location` object as a parameter to one of the methods.  I want to test the method, so I have to mock the object.

To do this, use `Unmock` - a library that does this for a whole host of standard classes.  First, edit the project-level `build.gradle` file, and add the appropriate line:

```gradle
buildscript {
    ext.kotlin_version = '1.3.61'
    repositories {
        google()
        jcenter()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:3.5.3'
        classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlin_version"
        classpath 'de.mobilej.unmock:UnMockPlugin:0.7.3'
    }
}
```
{:.line-numbers data-start="1" data-line="10"}

Then edit the `build.gradle` for the module containing the tests.  Add the plugin at the top:

```gradle
apply plugin: 'com.android.library'
apply plugin: 'kotlin-android'
apply plugin: 'kotlin-android-extensions'
apply plugin: 'de.mobilej.unmock'

unMock {
    keep "android.location.Location"
    keep "android.net.Uri"
    keepStartingWith "org."
    keepStartingWith "libcore."
    keepAndRename "java.nio.charset.Charsets" to "xjava.nio.charset.Charsets"
}

# Rest of the build.gradle file
```
{:.line-numbers data-start="1" data-line="4"}

I want to mock `android.net.Uri` as well since I parse a Uri.  Using this class instead of the more normal `java.net.Uri` class allows me to pass it directly to an Intent to start the web browser.  The other classes that are included were required by the two main classes.  Unmock can mock a whole host of classes.  There is an exhaustive list on [their web site](https://github.com/bjoernQ/unmock-plugin).

Now you can write your unit tests to use these two classes that wouldn't normally be available.

## Tip #2: Use Mockito for mocking the Android Context

Quite a few libraries - mine included - use the Android `Context` to read strings from a `values` file (like `strings.xml`).  I use this to localize the error messages produced by the library, for instance.  To do this, I call `context.getString(id)`.  Unfortunately, Unmock does not mock the context.  I'm guessing most libraries don't need the complete support of that the context provides - just a small part of it.

Enter `Mockito` - a library that mocks objects for you.  Using this library in Kotlin is not straight forward because the major method is a reserved word in Kotlin.  However, without it, you tend to jump through hoops.  For example, you might create an interface with just `getString()` in it, then create another object that just wraps the context.  This is ugly and means you can't pass the context directly to the library - something that most Android developers expect to do.

Start by adding the [Mockito library](https://site.mockito.org/) as a test dependency in your module-level `build.gradle` file.

```gradle
dependencies {
    # Rest of the gradle dependencies

    testImplementation 'org.mockito:mockito-core:2.23.0'
}
```
{:.line-numbers data-start="22"}

You can create the mock anywhere during the test creation.  I have a string referenced by the id `R.string.dslib_api_key`, which I mock like this:

```kotlin
@Test(expected = InvalidApiKey::class)
fun test_api_key() {
    va api_key = "1234"
    var context = mock(Context::class.java)
    `when`(context.getString(R.string.dslib_api_key))
        .thenReturn(api_key)
    
    val actual = DSLibrary(context)
    // This should throw
}
```

You have to surround the `when` method with back-ticks because `when` is a reserved word in Kotlin.  Thanks Mockito!  I hope they return to `whenever` as a method name or provide both as alternatives, as this tripped me up for a while.

There is one more problem I am going to be solving - how to mock the network connectivity stack.  However, that is a bigger blog post, so I'll leave that for next time.
