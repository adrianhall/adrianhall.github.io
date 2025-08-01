---
title:  "Organizing service injection in ASP.NET Core Minimal APIs"
date:   2024-08-07
categories:
  - Web
tags:
  - aspnetcore
  - minimal_apis
---

For the longest time, the Controller was the only way to introduce an API into your application.  With the latest versions of ASP.NET Core, [Minimal APIs](https://learn.microsoft.com/aspnet/core/fundamentals/minimal-apis/overview) became available.  These offered the potential to write less code and be more efficient since they didn't carry the baggage of the controller with them.  That does come with some drawbacks, however. For me, one of the main ones is that it is so much easier to write unreadable code. WIth some basic practices, however, you can use minimal APIs yet still have the power of dependency injection AND readable code.

If you've spent some time with minimal APIs, you've likely written some unreadable code.  Something like this:

{% highlight csharp %}
app.MapGet("/api/articles", async (
  [FromServices] IHttpContextAccessor contextAccessor,
  [FromServices] ILogger<MyApiService> logger,
  [FromServices] IFileProvider fileProvider,
  [FromServices] ApplicationDbContext dbContext,
  [FromQuery] int? skip,
  [FromQuery] int? top
) =>
{
  // Your code here.
});
{% endhighlight %}

It's ugly and quickly becomes unreadable and hard to maintain. You also likely have a whole bunch of mappings that take exactly the same services and you probably have common models for queries, such as pagination (like I do here).  The more services and parameters you introduce, the longer that argument list becomes and the less readable the code becomes.

There is a better way, and I'm sad to say I only learned it this week.

## Create a services collection class.

My class for the above mapping looks like this:

{% highlight csharp %}
public class ArticleServices(
  IHttpContextAccessor contextAccessor,
  ILogger<MyApiService> logger,
  IFileProvider fileProvider,
  ApplicationDbContext dbContext
)
{
  public HttpContext HttpContext { get => contextAccessor.HttpContext; }
  public ILogger Logger { get; } = logger;
  public IFileProvider FileProvider { get; } = fileProvider;
  public ApplicationDbContext DbContext { get; } = dbContext;
}
{% endhighlight %}

It's nice and simple.  It also allows me to make any other temporary services that depend on these services.  Repository patterns are not unheard of as a level of abstraction over the `DbContext` for example.  I can create a new repository based on the `DbContext` while I am at it.

## Create a query parameter collection class.

I have the following struct in my code:

{% highlight csharp %}
public struct PaginationRequest
{
  public int? Skip { get; }
  public int? Top { get; }
}
{% endhighlight %}

Again - nice and simple.  It also has the advantage of being reusable.  If I want to add a new thing to the pagination request, I can do it in one place.

## Update the mapping

So, what does my mapping look like now?

{% highlight csharp %}
app.MapGet("/api/articles", async ([AsParameters] ArticleServices services, [AsParameters] PaginationRequest request) => {
  // My code here
});
{% endhighlight %}

I find this to be much more readable!

## Creating an API the right way?

I don't actually do this in my code any more.  Sure, it's good for simple cases.  However, mappings quickly explode in "real" applications.  I now write an extension method to define the API **without the prefix**:

{% highlight csharp %}
public static partial class ApiDefinitions
{
  public static IEndpointRouteBuilder MapMyApi(this IEndpointRouteBuilder api)
  {
    api.MapGet("/", MyApi.GetAllItemsAsync);
    api.MapGet("/{id:string}", MyApi.GetItemAsync);
    api.MapPost("/", MyApi.PostBulkOperationAsync);
    api.MapPut("/{id:string}", MyApi.UpsertItemAsync);
    api.MapDelete("/{id:string}", MyApi.RemoveItemAsync);
  }
}
{% endhighlight %}

It's in a partial class because I put all my API definitions in one place but I split each definition into its own file.  In `Program.cs`, I have something like this:

{% highlight csharp %}
app.MapGroup("/api/articles").MapMyApi();
{% endhighlight %}

If I decide to add versioning, then I can do so easily:

{% highlight csharp %}
app.MapGroup("/api/articles").HasApiVersion(1.0).MapMyApi();
{% endhighlight %}

Finally, you'll notice that I defined the actual underlying methods for the API in a separate static class called `MyApi`:

{% highlight csharp %}
using Microsoft.AspNetCore.Http.HttpResults;

namespace MyApp.Apis.MyApi
{
  using GetItemResponse = Results<Ok<MyThing>, BadRequest<string>, NotFound>;

  public static class MyApi
  {
    public static async Task<GetItemResponse> GetItemAsync([AsParameters] MyApiServices, [FromRoute] string id, CancellationToken ct = default)
    {
      // My code here
    }
  }
}
{% endhighlight %}

A few things:

* I define a type alias for the response.  The typed results list gets unwieldy pretty quickly in my real applications, and that contributes to unreadable code.  Define the response up front and then use it everywhere.
* Since I am using a type alias with "short names", I have to place it inside a namespace block.  If I use fully qualified types, I can place it above the namespace statement (and use file-level namespaces again).  However, the type names become huge and make it even more unreadable than before.
* If I only have one parameter, I just include it in the method signature.  If I have multiple parameters, I define a struct (generally in the same file).
* If it's async, I always include a cancellation token.  I also call the async method if it's available and pass the cancellation token along.

## Final thoughts

Following these basic rules allows me to write minimal APIs using vertical slice easily.  I create an Apis folder, then each API has its own folder inside that.  Inside the API folder are the definition of the services, all the DTOs for the API, and the definition of the API itself.  Within the Apis folder, I have an extension method that defines all the APIs.

By following these best practices:

* Each API is self-contained within a single folder.
* The complete set of minimal APIs are defined in an extension method within the Apis folder as well.
* I'm only touching one (isolated) place when working on an API.

## Further reading

* [Parameter binding in Minimal APIs](https://learn.microsoft.com/aspnet/core/fundamentals/minimal-apis/parameter-binding)
