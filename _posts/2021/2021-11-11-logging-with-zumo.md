---
title: "Logging and Options for Azure Mobile Apps with ASP.NET 6"
categories:
  - Azure
tags:
  - ASP.NET Core
  - Azure Mobile Apps
---

Over the past three articles, I've covered a lot of the ins and outs of a typical Azure Mobile Apps service.  I've covered [scaffolding]({% post_url 2021/2021-11-08-azure-mobile-apps-intro %}), [authentication]({% post_url 2021/2021-11-09-adding-auth-to-zumo %}), and [complex authorization]({% post_url 2021/2021-11-10-complex-zumo-auth %}).  There are just a few more topics to cover on the basic cases.

## Logging

One of the big areas is logging.  You obviously want to see what the library is doing when it is running, especially because it is running unattended all the time.  Fortunately, it's an ASP.NET Core application, which means [logging is built-in](https://docs.microsoft.com/aspnet/core/fundamentals/logging/?view=aspnetcore-6.0).  Which isn't to say there is no work to be done - there is.  It's just that it works the same way as everything else.

To add logging, inject an `ILogger` into the constructor of the table controller, then assign it to the `Logger` property.

{% highlight csharp %}
[Route("tables/[controller]")]
public class TodoItemController : TableController<TodoItem>
{
  public TodoItemController(AppDbContext context, ILogger<TodoItemController> logger) : base() 
  {
    Repository = new EntityTableRepository<TodoItem>(context);
    Logger = logger;
  }
}
{% endhighlight %}

If you want to control the amount of logging that happens, you can do that in your `appsettings.json`:

{% highlight json %}
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Microsoft.AspNetCore.Datasync": "Information"
    }
  }
}
{% endhighlight %}

You can tune it to the level you want.  Logging is done at warning, information, and debug levels, so you can tune the amount of logging that happens.  By default, the logs will go to the console and the windows event log.  However, you configure logging for the entire service, so it can just as easily go to syslog, Azure Monitor, or any other logging system.

## Table Options

There are a number of table controller options that you can use to control the table controller:

* `PageSize` (default: 100) is the number of items returned in a single page of results.
* `MaxTop` (default: 512000) is the maximum number of items any specific query can return, even with paging.
* `EnableSoftDelete` (default: false) marks an item as deleted when requested rather than deleting the record.
* `UnauthorizedStatusCode` (default: 401 Unauthorized) is the status code that is returned when the access control provider indicates an unauthorized operation.

These are encapsulated by the `TableControllerOptions` class, and set within the table controller by the `Options` property.  For example:

{% highlight csharp %}
[Route("tables/[controller]")]
public class TodoItemController : TableController<TodoItem>
{
  public TodoItemController(AppDbContext context, ILogger<TodoItemController> logger) : base() 
  {
    Repository = new EntityTableRepository<TodoItem>(context);
    Logger = logger;
    Options = new TableControllerOptions
    {
      EnableSoftDelete = true,
      UnauthorizedStatusCode = StatusCodes.Status418ImATeapot
    };
  }
}
{% endhighlight %}

All of the options have a reasonable default.  You only need to set the ones you want to change.

Yes, `StatusCodes.Status418ImATeapot` is a valid status code. 
{: .notice--info}

Soft-delete, in particular, is a great thing to use when you are doing offline synchronization.  Soft-delete allows you to inform other clients that the record is gone and should be removed from the offline cache. However, it does require that you purge the records on a regular basis to keep the database clean.

## Next steps

In the next article, I'm going to go into what happens inside the `EntityTableRepository` and how you can build your own repository when your requirements are not met by a single table managed by Entity Framework Core.
