---
title: "Easy EditText content validation with Kotlin"
categories:
  - Mobile
tags:
  - android
  - kotlin
---

One of the things that I always deal with in my apps is the need to validate the input fields of a form. I can easily do this with this sample fragment of code (for example, for a length check):

{% highlight kotlin %}
field.addTextChangeListener(object: TextWatcher {
  override fun afterTextChanged(s: Editable?) {
    val content = s?.text.toString()
    s?.error = if (content.length >= 6) null else "Minimum length = 6"
  }
  override fun beforeTextChanged(s: Editable?) { }
  override fun onTextChanged(s: Editable?) { }
})
field.error = if (field.text.toString().length >= 6) null else "Minimum length = 6"
{% endhighlight %}

The TextWatcher is used to detect changes in the EditText control — roughly akin to “if the text changes, do something”. I use `EditText.setError()` (well, the Kotlin equivalent) to alert the UI that there is an issue with the input field. Android renders this as part of the UI with an alert icon and a message box.

This is mildly horrible code and an artifact of the Java classes that are used underneath. I’ve got boilerplate code (from the TextWatcher class) and duplicated code (because I have to set the error initially as well as validate the text along the way).

What would an ideal form look like? How about something like this?

{% highlight kotlin %}
field.validate("Minimum length is 6") { s -> s.lemngth >= 6 }
{% endhighlight %}

Instead of 10 lines of code, I only need to write 1 line of code plus bring in an extension function or two (which I can make part of my standard library of extension functions).

## The afterTextChanged extension

I like to split this problem up. For instance, the `addTextChangeListener` piece is a useful function in itself. I have this in my standard library of Kotlin extensions:

{% highlight kotlin %}
fun EditText.afterTextChanged(afterTextChanged: (String) -> Unit) {
    this.addTextChangedListener(object: TextWatcher {
        override fun afterTextChanged(s: Editable?) {
            afterTextChanged.invoke(s.toString())
        }

        override fun beforeTextChanged(s: CharSequence?, start: Int, count: Int, after: Int) { }

        override fun onTextChanged(s: CharSequence?, start: Int, before: Int, count: Int) { }
    })
}
{% endhighlight %}

Note the definition of the Kotlin lambda function `afterTextChanged`. It takes a string and doesn’t need a return value. This simplifies my original example to the following:

{% highlight kotlin %}
field.afterTextChanged {
  val content = it.text.toString()
  it.error = if (content.length >= 6) null else "message"
}
field.error = if (content.length >= 6) null else "message"
{% endhighlight %}

This is much improved already, but I still have duplicated code.

## The validate extension

I can abstract this duplicate code out with another extension function:

{% highlight kotlin %}
fun EditText.validate(message: String, validator: (String) -> Boolean) {
    this.afterTextChanged {
        this.error = if (validator(it)) null else message
    }
    this.error = if (validator(this.text.toString())) null else message
}
{% endhighlight %}

My validator is a lambda (or function) that takes a string and returns a boolean. I can also deal with the duplication here so my main code-path is not duplicating the code.

I have one more useful extension function to check that a string is a valid email address:

{% highlight kotlin %}
fun String.isValidEmail(): Boolean
  = this.isNotEmpty() && Patterns.EMAIL_ADDRESS.matcher(this).matches()
{% endhighlight %}

Put these together, and I can do the following:

{% highlight kotlin %}
user_field.validate("Valid email address required") { s -> s.isValidEmail() }
{% endhighlight %}

This is must more readable than the original 10–12 lines of code (depending on how much compression you do to avoid multiple lines). When you run an app with this code, you may see the following:

![]({{ site.baseurl }}/assets/images/2018/2018-04-11-image1.png)

When the user types in a valid email address, the icon and message go away.

Should you always use extension functions to make your code more readable? Probably not. However, you will find a number of utility functions that you use repeatedly. These are candidates for your standard extensions. You can find more on GitHub (check out [https://github.com/jitinsharma/Kotlin.someExtensions](https://github.com/jitinsharma/Kotlin.someExtensions) as an example)

Lambdas and extension functions are just two of the many reasons I love Kotlin!

{% include links.md %}
