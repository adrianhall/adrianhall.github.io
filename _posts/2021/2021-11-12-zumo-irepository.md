---
title: "Building repositories for Azure Mobile Apps with ASP.NET 6"
categories:
  - Azure
tags:
  - ASP.NET Core
  - Azure Mobile Apps
---

Over the last four days, I've delved into how to build a data sync service using Azure Mobile Apps on ASP.NET Core 6.  I've covered [the template]({% post_url 2021/2021-11-08-azure-mobile-apps-intro %}), [authentication]({% post_url 2021/2021-11-09-adding-auth-to-zumo %}), [authorization]({% post_url 2021/2021-11-10-complex-zumo-auth %}), and [logging]({% post_url 2021/2021-11-11-logging-with-zumo %}).  The basic setup is really good when you have a 1:1 relationship between the table and the DTO and you don't need to do anything special, like support a non-EF Core ORM, or integrate real-time alerting.  There are all sorts of reasons you don't want to use the standard repository.

So you get to write your own.

## Introducing ITableData and IRepository

Whenever I wrote a web API for a database service in the past, it inevitably started with a repository pattern.  Not unsurprising, then, Azure Mobile Apps uses a repository pattern.  It starts with a pair of interfaces:

* `ITableData` provides additional properties that you must implement in your DTO.
* `IRepository<T>` provides the basic functionality for a repository for a specific DTO.

You don't need to make your repository generic (even though all of the standard repositories are generics - they are general purpose, so they need to work with any DTO).

When you are writing a new repository, you may have to write a new abstract class to implement `ITableData` as well.  You will always have to write a class that implements `IRepository<T>`.  Let's take a look at `IRepository<T>` (stripped of the comments in the code):

{% highlight csharp %}
public interface IRepository<TEntity> where TEntity : ITableData
{
    IQueryable<TEntity> AsQueryable();
    Task CreateAsync(TEntity entity, CancellationToken token = default);
    Task DeleteAsync(string id, byte[] version = null, CancellationToken token = default);
    Task<TEntity> ReadAsync(string id, CancellationToken token = default);
    Task ReplaceAsync(TEntity entity, byte[] version = null, CancellationToken token = default);
}
{% endhighlight %}

It's not a complex interface.  In fact, it suspiciously simple. There are some things you must be aware of:

1. The `Id` field in `ITableData` is a globally unique string within the table, and that the user can set the `Id` to a known value when creating the record.  If you are translating the `Id` to a long, for example, it's best to create a new property for it.
1. You need to be careful to ensure that the entities (identified by `TEntity` throughout) are "disconnected" - that is, changes to the entity after the call is completed do not affect the underlying database.
1. You need to ensure you throw `ConflictException<TEntity>` if the version (if not null) does not match.
1. The `AsQueryable()` method returns the complete dataset if the queryable is executed.  The service adds onto this to do all the OData stuff it needs for filtering, ordering, and paging.
1. You are responsible for updating the `UpdatedAt` and `Version` properties in the DTO before storage.  Your database storage layer (like EFCore) may do this for you.
1. You don't need to worry about soft-delete, filtering, ordering, and paging.  This is handled for you.

## An example

Let's look at an example.  Let's say you have a set of documents, and the document list is large.  You want to allow users to subscribe to documents and they then get the updates to those documents.  However, maintaining a list on the client is not feasible - there are too many of them to list them all and synchronize each one.  Instead, you create a new subscriptions field that just has the UserId of the user and the DocumentId of the record.  Something like this:

{% highlight csharp %}
public class Subscription : EntityTableData, IUserId
{
  public string UserId { get; set; }
  public string DocumentId { get; set; }
}

public class Document : EntityTableData
{
  public string Title { get; set; }
  public string Notes { get; set; }
}
{% endhighlight %}

You can use a regular Entity Framework Core table to support the `Subscription` table:

{% highlight csharp %}
[Authorize]
[Route("tables/[controller]")]
public class SubscriptionController : TableController<Subscription>
{
  public SubscriptionController(AppDbContext context, IHttpContextAccessor accessor) : base()
  {
    Repository = new EntityTableRepository<Subscription>(context);
    Options = new TableControllerOptions { EnableSoftDelete = true };
    AccessControlProvider = new PersonalAccessControlProvider<Subscription>(accessor);
  }
}
{% endhighlight %}

We've seen this pattern before in our earlier work.  The repository for the `Document` is a little different though.  We want to combine two different tables using EF Core.  Here is the updated DTO:

{% highlight csharp %}
public class DocumentDTO : EntityTableData
{
  public string Title { get; set; }
  public string Notes { get; set; }
  public bool IsSubscribed { get; set; }
}
{% endhighlight %}

The DTO is the same as the `Document` model, but I've added a new computed property.

The controller will look like this:

{% highlight csharp %}
[Authorize]
[Route("tables/[controller]")]
public class DocumentController : TableController<DocumentDTO>
{
  public DocumentController(AppDbContext context, IHttpContextAccessor accessor) : base()
  {
    Repository = new DocumentTableRepository(context, accessor);
    Options = new TableControllerOptions { EnableSoftDelete = true };
    AccessControlProvider = new PersonalAccessControlProvider<DocumentDTO>(accessor);
  }
}
{% endhighlight %}

The `DocumentTableRepository` doesn't exist yet.

## Implementing the DocumentTableRepository

Since both of my tables are in EF Core, I'm using the regular `EntityTableData` as the basis for my DTO as well.  Let's look at the basics:

{% highlight csharp %}
public class DocumentTableRepository : IRepository<DocumentDTO>
{
  private readonly AppDbContext context;
  private readonly IHttpContextAccessor accessor;

  public DocumentTableRepository(AppDbContext context, IHttpContextAccessor accessor)
  {
    this.context = context;
    this.accessor = accessor;
  }

  public string UserId { get => accessor.HttpContext.Identity?.User?.Name; }
}
{% endhighlight %}

I haven't implemented the actual methods yet. Let's consider all of the mutation methods.  We don't mutate the DTO - we mutate the underlying document, so we have to create a `Document` from a `DocumentDTO`.  Conveniently, this also "disconnects" the document from our input nicely:

{% highlight csharp %}
internal static DocumentExtensions
{
  internal static Document ToDocument(this DocumentDTO dto)
  {
    return new Document 
    {
      Id = dto.Id,
      UpdatedAt = dto.UpdatedAt,
      Version = dto.Version,
      Deleted = dto.Deleted,
      Title = dto.Title,
      Notes = dto.Notes
    };
  }
}
{% endhighlight %}

I've written similar helpers for other parts.  For example, my `DocumentDTO` has a constructor for a `Document` so I can convert back and all I need to do is fill in the `IsSubscribed` property.  I can now write the mutation methods basic on the Entity Framework Core ones.  So, for example, here is the `ReplaceAsync` method:

{% highlight csharp %}
public async Task ReplaceAsync(DocumentDTO entity, byte[] version = null, CancellationToken token = default)
{
  if (entity == null) 
  {
    throw new ArgumentNullException(nameof(entity));
  }
  if (string.IsNullOrEmpty(entity.Id))
  {
    throw new BadRequestException();
  }
  
  try
  {
    var document = entity.ToDocument();
    var storedEntity = await LookupAsync(document.Id, token).ConfigureAwait(false);
    if (storedEntity == null)
    {
      throw new NotFoundException();
    }
    if (version != null && storedEntity.Version?.SequenceEqual(version) != true)
    {
      throw new PreconditionFailedException(Disconnect(storedEntity));
    }
    document.UpdatedAt = DateTimeOffset.UtcNow;
    Context.Entry(storedEntity).CurrentValues.SetValues(document);
    await Context.SaveChangesAsync(token).ConfigureAwait(false);
  }
  catch (DbUpdateException ex)
  {
    throw new RepositoryException(ex.Message, ex);
  }
}
{% endhighlight %}

It's worth figuring out what is happening here.  However, the main change from the standard version to this one is that I change the provided entity into a document prior to storing it.  The rest is the same as the regular code.  I've left out the `Disconnect` and `LookupAsync` code that is a part of the standard codebase because it is the same.

## ReadAsync and AsQueryable

The above mechanism is similar for the create and delete functionality.  However, `ReadAsync` and `AsQueryable` are different because I need to combine two different tables.  Let's start with `AsQueryable`:

{% highlight csharp %}
public IQueryable<DocumentDTO> AsQueryable()
{
  return context.Documents.Select(model => new DocumentDTO(model) {
    IsSubscribed = context.Subscriptions.Any(sub => sub.UserId == UserId && sub.DocumentId == model.Id)
  }).AsQueryable();
}
{% endhighlight %}

This is probably not the most efficient way to do this in LINQ.  I'm returning a queryable of the list of documents, but augmented with the `IsSubscribed` property, which is found if there is a record for the user ID and document ID in the subscriptions table.  Note that I don't say "this is subscribed" or "this is not subscribed" within the queryable return value - I just return the entire data set.  This will allow the mobile app to have a huge list and toggle between the "subscribed documents" and "all documents" views.

The `ReadAsync` method has the same logic for generating the `IsSubscribed` property:

{% highlight csharp %}
public async Task<DocumentDTO> ReadAsync(string id, CancellationToken token = default)
{
  if (string.IsNullOrEmpty(id)) 
  {
    throw new BadRequestException();
  }

  var document = await LookupAsync(id, token).ConfigureAwait(false);
  var dto = new DocumentDTO(document)
  {
    IsSubscribed = context.Subscriptions.Any(sub => sub.UserId == UserId && sub.DocumentId == id)
  };
  return dto;
}
{% endhighlight %}

Now, I've left off a lot of the auxiliary code in this, but it should be fairly easy to follow along and fill in the gaps.

## Next steps

You should definitely take a look at the two implementations of the table repository I have written:

* [EntityTableRepository](https://github.com/Azure/azure-mobile-apps/blob/main/sdk/dotnet/src/Microsoft.AspNetCore.Datasync.EFCore/EntityTableRepository.cs)
* [InMemoryTableRepository](https://github.com/Azure/azure-mobile-apps/blob/main/sdk/dotnet/src/Microsoft.AspNetCore.Datasync.InMemory/InMemoryRepository.cs)

You will see that, for all that they do, they are actually quite simple to follow.  My own experience is that table repositories should be straight forward.  We aren't doing a lot of work in there - just storing and reading data.

If you find you can't do what you need without significant complexity in your table repository, perhaps Azure Mobile Apps is the wrong library.  Azure Mobile Apps is an awesome and simple way to spin up an offline-aware REST endpoint for your data, but doesn't solve every problem with data synchronization.

Let me know how you use Azure Mobile Apps on [the discussion boards](https://github.com/Azure/azure-mobile-apps/discussions).  I'd love to hear what you are doing!