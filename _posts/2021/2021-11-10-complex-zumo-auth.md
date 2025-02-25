---
title: "Controlled access for Azure Mobile Apps for ASP.NET Core"
categories:
  - Cloud
tags:
  - aspnetcore
  - azure_mobile_apps
---

In the last two articles, I've gone over how you can [create a basic datasync service]({% post_url 2021/2021-11-08-azure-mobile-apps-intro %}) and [add authentication]({% post_url 2021/2021-11-09-adding-auth-to-zumo %}) to the service.  What if you want to do something more complex?  Authorization that is an on/off switch is reasonable as a first pass, but rarely allows you to handle the cases you actually need to implement.

## Introducing the Access Control Provider

To handle cases where your needs are a little more complex, we've introduced a new interface: `IAccessControlProvider<T>`.  This requires you to implement three methods:

* `GetDataView` is a func used to limit what data the user can see.
* `IsAuthorizedAsync` determines if the user is allowed to do the requested operation.  If not, a `401 Unauthorized` response is returned.
* `PreCommitHookAsync` is called right before each database write to allow you to modify the stored entity.

## A quick example

Let's say you have the following model:

{% highlight csharp %}
public class Model : EntityTableData
{
  [JsonIgnore]
  public bool IsApproved { get; set; }
  
  public string Title { get; set; }

  public string Notes { get; set; }
}
{% endhighlight %}

As models go, this isn't too bad.  We've used the `JsonIgnore` to ensure that the user connecting from the mobile app never gets to see if the record is approved.  However, we want our internal code to see it so we can make decisions based on the approval state.

Now, we want to provide a table controller that

* Only shows the user approved records.
* Doesn't allow the user to modify records.

To do this, we create a new class that implements `IAccessControlProvider<Model>`:

{% highlight csharp %}
public class ApprovalAccessControlProvider : IAccessControlProvider<Model>
{
  public Func<T, bool> GetDataView() 
    => model => model.IsApproved;

  public Task<bool> IsAuthorizedAsync(TableOperation op, T? entity, CancellationToken token = default) 
    => Task.FromResult(op == TableOperation.Query || op == TableOperation.Read);

  public Task PreCommitHookAsync(TableOperation op, T entity, CancellationToken token = default)
    => Task.CompletedTask;
}
{% endhighlight %}

The `GetDataView()` is used in a LINQ `.Where(view)` call to limit the data.  In this case, IsApproved must be set to true to show the data.  The `IsAuthorizedAsync()` method returns true if the operation is a query or read operation - create, update, or delete operations are disallowed.  We don't need to do any changes to the pre-commit hook since we aren't doing any writes.

> We use async methods for the authorization and pre-commit hook because you may want to do additional database lookups to provide the right functionality.

You can use this in the controller as follows:

{% highlight csharp %}
[Authorize]
[Route("tables/[controller]")]
public class ModelController : TableController<Model>
{
  public ModelController(AppDbContext context) 
  {
    AccessControlProvider = new ApprovalAccessControlProvider();
    Repository = new EntityTableRepository<Model>(context);
  }
}
{% endhighlight %}

> You can also embed the access control provider in the controller.  Just implement the interface in the controller, then set `AccessControlProvider` to `this`.  Implementing it as a separate class allows code re-use.

## Using authentication in an access control provider

Of course, eventually, you will want a multi-user table.  Start by implementing an interface for the user ID:

{% highlight csharp %}
internal interface IUserId
{
  public string UserId { get; set; }
}
{% endhighlight %}

Now, let's create a model with the user ID:

{% highlight csharp %}
public class TodoItem : EntityTableData, IUserId
{
  [JsonIgnore]
  public string UserId { get; set; }

  [Required, MinLength(1)]
  public string Title { get; set; }

  public bool IsComplete { get; set; }
}
{% endhighlight %}

As is common with security information, we decorate the `UserId` with `JsonIgnore` so it doesn't get sent to the client.  Now, let's write a basic but reusable access control provider that allows any authenticated user to create or query the table, but limits modifications to the same user.

{% highlight csharp %}
internal class PersonalAccessControlProvider<T> : IAccessControlProvider<T>
  where T : ITableData where T : IUserId
{
  private readonly IHttpContextAccessor _accessor;

  public PersonalAccessControlProvider(IHttpContextAccessor accessor)
  {
    _accessor = accessor;
  }

  /// <summary>
  /// Obtains the user ID, or null if not authenticated
  /// </summary>
  private string? UserId { get => _accessor.HttpContext.User?.Identity?.Name; }

  public Func<T, bool> GetDataView()
  {
    return UserId == null
      ? _ => false
      : model => model.UserId == UserId;
  }

  public Task<bool> IsAuthorizedAsync(TableOperation op, T? entity, CancellationToken token = default)
  {
    switch (op)
    {
      case TableOperation.Create:
      case TableOperation.Query:
        return Task.FromResult(UserId != null);
      default:
        return Task.FromResult(UserId != null && entity?.UserId == UserId);
    }
  }
However, for example, pbleOperation op, T entity, CancellationToken token = default) 
  {
    entity.UserId == UserId;
    return Task.CompletedTask;
  }
}
{% endhighlight %}

There is a lot going on here.  Let's start at the interface level.  This class only accepts models that implement `IUserId`.  We accept a [HttpContextAccessor](https://docs.microsoft.com/aspnet/core/fundamentals/http-context?view=aspnetcore-6.0#use-httpcontext-from-a-controller) that allows us to access the `HttpContext` from within an async controller or middleware (more on that in a moment).  Then we have our three methods.  We take care in the first two methods to ensure the user is authenticated before returning "the right thing".  In the pre-commit hook, we set the `UserId` to the ID of the authenticated user.  At this point, we know the user is already authenticated (since the `IsAuthorizedAsync` method has been called and returned true), so it's safe to do the assignment.

To use this, we need to first declare the dependency on `HttpContextAccessor`.  Edit `Program.cs` and add the line:

{% highlight csharp %}
builder.Services.AddHttpContextAccessor();
{% endhighlight %}

Then adjust the controller:

{% highlight csharp %}
[Authorize]
[Route("tables/[controller]")]
public class TodoItemController : TableController<TodoItem>
{
  public TodoItemController(AppDbContext context, IHttpContextAccessor accessor) 
  {
    AccessControlProvider = new PersonalAccessControlProvider<TodoItem>(accessor);
    Repository = new EntityTableRepository<TodoItem>(context);
  }
}
{% endhighlight %}

The `IHttpContextAccessor` is passed in via depedency injection, and we use it to create the access control provider.

## Another example

Let's say I wanted to have a model that could be read by anonymous users, but only updated by authenticated users.  I could do something like this:

{% highlight csharp %}
[AllowAnonymous]
[Route("tables/[controller]")]
public class ModelController : TableController<Model>
{
  public ModelController(AppDbContext context, IHttpContextAccessor accessor)
  {
    AccessControlProvider = new AnonymousReadAccessControlProvider<Model>(accessor);
    Repository = new EntityTableRepository<Model>(context);
  }
}
{% endhighlight %}

I'm using `AllowAnonymous` here to ensure the user identity is populated in the `HttpContext`.  If you don't do some sort of authorization, then it isn't populated.  Now, let's take a look at the access control provider:

{% highlight csharp %}
public class AnonymousReadAcessControlProvider<T> : IAccessControlProvider<T> where T : ITableData
{
  private readonly IHttpContextAccessor _accessor;

  public AnonymousReadAccessControlProvider(IHttpContextAccessor accessor)
  {
    _accessor = accessor;
  }

  private bool IsAuthenticated {get => _accessor.HttpContext.Identity?.User?.Name != null; }

  public Func<T, bool> GetDataView() => _ => true;

  public Task PreCommitHookAsync(TableOperation op, T entity, CancellationToken token = default)
    => Task.CompletedTask;

  public Task<bool> IsAuthorizedAsync(TableOperation op, T? entity, CancellationToken token = default)
  {
    if (op == TableOperation.Query || op == TableOperation.Read)
      return Task.FromResult(true);
    else
      return Task.FromResult(IsAuthenticated);
  }
}
{% endhighlight %}

## Next steps

That's it for authentication and authorization.  But there is more to go over.  There are a bunch of options you can also supply to the table controller to modify the behavior.  In the next article, I'll go over those options to round out the basic operations for a table.
