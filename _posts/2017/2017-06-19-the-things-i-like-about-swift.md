---
title: "The things I like (and don't like) about Swift?"
categories:
  - Mobile
tags:
  - swift
---

Recently, I’ve given myself the task of learning the “native” mobile development platforms.  That means Java or Kotlin for Android and Swift or Objective-C for iOS development.  Kotlin is a ways behind Java for Android development and I already knew (somewhat) the Java language, so that one was relatively easy.  Swift is the new kid on the block, but it’s obviously the way to go for iOS Development.

This is not a blog post about how to learn Swift.  In fact, I’m probably not going to teach you anything at all.  If you really want to learn Swift, then I recommend the [official Apple documentation](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/) and the [Stanford University iTunesU course](https://itunes.apple.com/us/course/developing-ios-10-apps-with-swift/id1198467120).  I’ve reviewed both over the last week and they got me pointed in the right place and to my goal of writing my first apps.  Of course, it helps if you have gone through learning a programming language a few times – you will notice the same structures repeated over and over again.

Swift is different from most other languages – I struggled to even read the code for a while until I grokked the optional stuff (more on that later).  I’m not a fanboy, however.  I don’t “like” Swift – any more than I like Java, C#, Python, Perl or JavaScript.  All languages have their quirks – some quirks I like, and some I don’t. I’m not going to suddenly start jamming Swift into places it shouldn’t go. I like using the right tool for the right job. Swift feels like a good tool for writing iOS apps; it doesn’t feel good for writing backend code.

So, what about Swift?

There are a bunch of things I like about Swift.  The biggest one is proper support for Unicode.  Normally, a language that “supports” Unicode just means that the String can hold Unicode characters, and that the characters in a String are double-byte to effect that. However, I’ve not seen any other language that can write the following:

{% highlight swift %}
let π = 3.14159
let r = 2.0
let area = π * r * r
{% endhighlight %}

Yes – you can [use Unicode characters for variable names](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/TheBasics.html#//apple_ref/doc/uid/TP40014097-CH5-ID309). The other major thing that Unicode “support” generally lacks is the combination of Unicode characters. For example café – the é can be written as either a single Unicode character or a combination of an e followed by a combinator acute symbol. In Swift, both are valid and both will compare the same.

I like [tuples](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/TheBasics.html#//apple_ref/doc/uid/TP40014097-CH5-ID309). Tuples are a set of information that can be returned from a function (or just grouped together) so that they act like a single unit. In most other languages, one creates a model for this and then an instance of a model. Using tuples feels a bit more friendly to the developer. It allows the developer to say “I want this function to return three bits of information – not just one”.

I like the built-in [collections](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/CollectionTypes.html#//apple_ref/doc/uid/TP40014097-CH8-ID105) – particularly the Set type. Normally, I’m stuck with array and dictionary types, which are fine. However, sometimes I need to do set operations, which on arrays are difficult. It’s good having the option to have an unordered list with set operators (like intersection and union).

Finally, I like [preconditions and assertions](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/TheBasics.html#//apple_ref/doc/uid/TP40014097-CH5-ID309). Most developers, when writing a function, will have a set of if/then statements at the top of their function to ensure that the parameters are not just the right type; they are also have valid values. Preconditions and assertions allow the developer to do the same thing. However, it “feels” much more intuitive to say “this is a precondition”.

This brings me to my final point. Once you get beyond the quirks that I don’t like, Swift is a very readable language. If the developer has taken the time to name his variables correctly, then it’s a pleasure to read. Unfortunately, a lot of developers don’t take that time. You can write bad code in any language.

There are a few things I don’t like about Swift.

The first is [optionals](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/TheBasics.html#//apple_ref/doc/uid/TP40014097-CH5-ID309). You will see a lot of code that looks like this:

{% highlight swift %}
func foo(_ sender: UIButton?) -> String {
    return sender!.currentTitle!
}
{% endhighlight %}

What is with all the ? and !? Well, the Swift designers decided that they wanted to be explicit about nullable variables and ensuring that all variables have values. The ? after a type means “nullable” – something that can take the nil value. It’s called “optional” by Swift, because they wanted to be different, I guess. The ! means “I want the value, not the optional value”; a process called unwrapping. But wait, you can unwrap in an if statement!

{% highlight swift %}
func foo(_ sender: UIButton?) -> String {
    if let unwrapped = sender {
        return unwrapped.currentTitle!
    } else {
        // Do something with nil
    }
}
{% endhighlight %}

That’s less readable. What was wrong with comparing with nil and then doing something with the value? Was that so bad?

Swift supports [multi-line strings](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/StringsAndCharacters.html#//apple_ref/doc/uid/TP40014097-CH7-ID285) as well – arguably a good feature. However, their implementation has some quirky behavior. How do you get a CR at the beginning of your multi-line string? A blank line after the start makes me think 2 CRs. Indenting rules are also problematic? I hate significant white space (which is why I loath Python). It seems that the team were trying to make beautiful looking code a design criteria.

Swift functions have [parameters that have an internal name and an external name](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/Functions.html#//apple_ref/doc/uid/TP40014097-CH10-ID158). This is good in theory. However, they are bucking the trend here of being able to have readable code. If I pass in `color: UIColor.RED`, then I want it to be `color` in the function as well – not something else. The underscore parameter name as a special case is also annoying.

Let’s talk object-orientated programming, and protocols, structs, types, and enums. There are just too many things for a new object. It seems that the designers of the language could have pushed enums, classes, structs and protocols and called them all classes and interfaces – just like a whole bunch of other languages. Sometimes, being different just to be different isn’t the best idea, and this feels like it.

Finally, why did they have to continue with callbacks as a predominant asynchronous event model? There is the async/await pattern and the Promise pattern – both of which are better and more readable than callbacks. I hope to see one of these patterns in a future version of the language, which is still very much in flux.

Overall Impressions?

Well, it’s no worse than Java, C#, or TypeScript – the languages that I find myself using the most currently. It’s got a big name behind it, which means it will live and be improved for many years to come. It’s got some good quirks and some bad quirks, but overall I can live with them all.

There are two questions on my mind here:

1. Will I be writing mobile apps in Swift long-term? I still think [progressive web apps](https://www.smashingmagazine.com/2016/08/a-beginners-guide-to-progressive-web-apps/) with service workers and cross-platform native apps using something like [React Native](https://facebook.github.io/react-native/) will win out over proprietary languages in the long term. I don’t see Swift being used to write Android apps (any more than I see iOS apps being written in Kotlin – not going to happen!), so this will always be an iOS only thing. This is fine if all you are writing is iOS apps. I write for everything.

2. Will I be writing backend code in Swift? I don’t think so. There are too many other good languages that are already good at multi-tiered architectures, database access, concurrency, etc. Backend concerns and frontend concerns are different. Use the right tool for the right job.
