---
title:  "Better data validation exceptions with C#"
date:   2024-08-10
categories: ASP.NET Core
tags:
  - Tips
  - C#
---

There are times when I look at code I have written and think to myself "there has to be a better way."  When I start thinking like this, I start by looking at the documentation - the .NET official documentation is incredibly well written and simple to digest, and the fundamentals section is something I believe every language documentation should aspire to.  Take exceptions, for example.  When I started my career in software development, methods were written with a block of validations at the top.  I couldn't be certain if bad data was going to creep in somewhere, so I checked all arguments religiously.  Then I added tests for each of those validations.

These days, it's much simpler.  Today I want to show you how I leverage data annotations, custom exception handlers, and extension methods to make my code readable and predictable, yet also fully testable without a mountain of tests for validation.

## Parameter validation the hard way

Let's take you back to the beginning of time.  How did I start doing argument validation?  I wrote something like this:

{% highlight csharp %}
public void MyMethod(string arg)
{
  if (arg == null)
  {
    throw new ArgumentNullException(nameof(arg));
  }
  if (string.IsNullOrEmpty(arg))
  {
    throw new ArgumentException("Argument is empty", nameof(arg));
  }
}
{% endhighlight %}

This is an incredibly naive way of writing a method for a number of reasons.  The main one, however, is that you have to test all these conditions individually for every single method you write. This leads to an explosion of tests and your development energy is better spent elsewhere. As the number of arguments (and validations) increases, the code also becomes unreadable.

## Improving data validation

Eventually, I switched to a static class:

{% highlight csharp %}
public static class Arguments
{
  public static IsNotNullOrEmpty(string arg, string paramName)
  {
    if (arg == null)
    {
      throw new ArgumentNullException(paramName);
    }
    if (string.IsNullOrEmpty(arg))
    {
      throw new ArgumentException("Argument is empty", paramName);
    }
  }
}
{% endhighlight %}

Centralizing the argument handling makes the code much more readable.  It also means I can write unit tests for the central arguments class and ignore testing individual methods, resulting in a smaller unit test footprint and more time spent writing the code that matters.  In fact, the core team noticed that there were some common validations, like null checks, and they built them into the exception classes:

{% highlight csharp %}
public void myMethod(string arg)
{
  ArgumentNullException.ThrowIfNullOrEmpty(arg);
}
{% endhighlight %}

## Best practices for parameter validation

Putting the checker for the arguments with the `ArgumentException` makes a whole lot of sense!  Unfortunately, `ArgumentException` is a core class, so I can't extend it with a new static method. It's relatively easy to create new exception classes.  Here is one I use a lot:

{% highlight csharp %}
using System.ComponentModel.DataAnnotations;

public class ArgumentValidationException : ArgumentException
{
  public ArgumentValidationException() : base() { }
  public ArgumentValidationException(string? message) : base(message) { }
  public ArgumentValidationException(string? message, Exception? innerException) : base(message, innerException) { }
  public ArgumentValidationException(string? message, string? paramName) : base(message, paramName) { }
  public ArgumentValidationException(string? message, string? paramName, Exception? innerException) : base(message, paramName, innerException) { }

  public IList<ValidationResult>? ValidationErrors { get; private set; }

  public static void ThrowIfNotValid(object? value, string? paramName)
    => ThrowIfNotValid(value, paramName, "Object is not valid");

  public static void ThrowIfNotValid(object? value, string? paramName, string? message)
  {
    ArgumentNullException.ThrowIfNull(value, paramName);
    List<ValidationResult> results = [];
    if (!Validator.TryValidateObject(value, new ValidationContext(value), results, validateAllProperties: true))
    {
      throw new ArgumentValidationException(message, paramName) { ValidationErrors = results };
    }
  }
}
{% endhighlight %}

I use data annotations a lot - even outside ASP.NET Core.  This exception is designed to be thrown when an input variable with data annotations isn't valid. Let's say I have a model where each value has a range of valid values.  I might write this model class like this:

{% highlight csharp %}
public class PaginationRequest
{
  private int _skip = 0, _take = Constants.MaxPageSize;

  public int Skip
  {
    get => _skip;
    set
    {
      ArgumentOutOfRangeException.ThrowIfLessThan(value, 0, nameof(Skip));
      _skip = value;
    }
  }

  public int Take
  {
    get => _take;
    set
    {
      ArgumentOutOfRangeException.ThrowIfLessThanOrEqual(value, 0, nameof(Take));
      ArgumentOutOfRangeException.ThrowIfGreaterThan(value, Constants.MaxPageSize, nameof(Take));
      _take = value;
    }
  }
}
{% endhighlight %}

The use of a backing variable just for range checking doesn't seem right to me, and I need to write two model classes - one for doing data validation in the ASP.NET Core application that doesn't throw, and one for using in the rest of the system that does throw an exception.

So if this is bad, what's the alternative?  I use data annotations:

{% highlight csharp %}
public class PaginationRequest
{
  [GreaterThanOrEqual(0, ErrorMessage = "The Skip value must be positive")]
  public long Skip { get; set; } = 0;

  [Range(0, Constants.MaxPageSize, ErrorMessage = "The Take value must be between 0 and the maximum page size")]
  public long Take { get; set; } = Constants.MaxPageSize;
}
{% endhighlight %}

Firstly, can we appreciate how much easier to read this code is.  I can easily discern the range of valid values and the default value without jumping away.  Consider a model that is much larger and you will start to see the benefits. What is not so obvious is that I can use the same model class for both a controller and a service.  In a controller, bad input is a fairly common occurence and it's considered bad form to throw exceptions for normal situations.  I can write my controller like this:

{% highlight csharp %}
public async Task<IActionResult> GetModelsAsync([FromQuery] PaginationRequest request, CancellationToken ct = default)
{
  // Good version
  if (!ModelState.IsValid)
  {
    return BadRequest(ModelState)
  }
}
{% endhighlight %}

In a service, I want to throw an exception on a validation error.  I can use the `ArgumentValidationException` I developed earlier:

{% highlight csharp %}
public async Task<IEnumerable<Model>> GetModelsAsync(PaginationRequest request, CancellationToken ct = default)
{
  ArgumentValidationException.ThrowIfNotValid(request, nameof(request));
  // Rest of my method goes here
}
{% endhighlight %}

Because the `ArgumentValidationException` is also an `ArgumentException`, tests that use this (including try/catch blocks elsewhere in the app) will happily understand that an argument was at fault.  If one of my methods actually needs to know the validation errors, then I can see the validation errors in the exception object through the debugger.

> **Avoid using exceptions in ASP.NET Core Controllers and Minimal APIs**
>
> Exceptions should be used for "extraordinary" situations that cannot be resolved in other ways.  Data validation errors are common and expected, so they should be handled as such.  See the [advice on exceptions](https://learn.microsoft.com/dotnet/standard/exceptions/best-practices-for-exceptions) from the dotNET team for more information.

Using data annotations and my validation exception allows me to avoid using backing variables for my models, yet still allow for throwing or validation checks in the right way.  This leads me to my final extension method - the validator:

{% highlight csharp %}
public static ValidatorExtensions
{
  public static bool IsValid(this object value, [NotNullWhen(false)] out IList<ValidationResult>? validationErrors)
  {
    List<ValidationResult> results = [];
    if (!Validator.TryValidateObject(value, new ValidationContext(value), results, validateAllProperties: true))
    {
      validationErrors = results;
      return false;
    }
    return true;
  }
}
{% endhighlight %}

I think this is what `.TryValidateObject()` should look like.  Most of the time, this is the code I want to run when I am validating an object.  I can easily validate three different ways now, depending on my needs:

{% highlight csharp %}
// Throw if not valid
ArgumentValidationException.ThrowIfNotValid(request, nameof(request));

// Test for validation and handle errors
if (!request.IsValid(out var validationErrors))
{
  // Handle validation errors here
}

// Within a controller
if (!ModelState.IsValid) 
{
  // Handle validation errors here
}
{% endhighlight %}

Each form is testable without an explosion of tests that are there just to test validation logic.  The validation logic is also well-known (use data annotations) which makes the code very readable.

## Final thoughts

Extension methods, custom data annotations, and custom exception classes with static validators are three techniques I use on a regular basis to make my code more readable and to ensure that I have great test coverage for everything. I used to have a growing collection of data annotations, but I found that someone has inevitably written a data annotation that I can use in my project.

My final thought of this blog.  Occassionally, I am reminded of a fundamental feature of dotNET that I had forgotten about. It's always worth reading [.NET Fundamentals](https://learn.microsoft.com/dotnet/fundamentals/) on a regular basis. Things do get updated and the updates are inevitably good guidance that improves your productivity.  If you ever stare at your code and think "there has to be a better way to express this!", there probably is and you will likely find it in the documentation.

## Further reading

* [.NET Fundamentals](https://learn.microsoft.com/dotnet/fundamentals/)
* [Best practices for exception handling](https://learn.microsoft.com/dotnet/standard/exceptions/best-practices-for-exceptions)
* [Model validation](https://learn.microsoft.com/aspnet/web-api/overview/formats-and-model-binding/model-validation-in-aspnet-web-api)
