---
title: "Azure Mobile Apps: Supporting tables with int Id"
categories:
  - Azure
tags:
  - ASP.NET Core
  - Azure Mobile Apps
---

One of the persistent questions I get asked about Azure Mobile Apps is how to support older tables.  Let's say you have an older application that was never designed for mobile applications.  It likely has a model like this:

```csharp
public class Movie
{
    /// <summary>
    /// A application specific unique identifier for the movie.
    /// </summary>
    [Key]
    public int Id { get; set; }

    /// <summary>
    /// True if the movie won the oscar for Best Picture
    /// </summary>
    public bool BestPictureWinner { get; set; }

    /// <summary>
    /// The running time of the movie
    /// </summary>
    public int Duration { get; set; }

    /// <summary>
    /// The MPAA rating for the movie, if available.
    /// </summary>
    public MovieRating Rating { get; set; } = MovieRating.Unrated;

    /// <summary>
    /// The release date of the movie.
    /// </summary>
    public DateOnly ReleaseDate { get; set; }

    /// <summary>
    /// The title of the movie.
    /// </summary>
    public string Title { get; set; } = "";

    /// <summary>
    /// The year that the movie was released.
    /// </summary>
    public int Year { get; set; }
}
```

This is actually the model I used when learning ASP.NET6 MVC and Razor back in the day.  I recently recreated the application and you can find it [at the `before` tag on my GitHub repository](https://github.com/adrianhall/zumo-intid-server/releases/tag/before) that I set up for this blog post.  The application is runnable.  After you run it, click on the "Movies App" at the top to get into the list and you can follow along as I modify this application to have a mobile ready API.

It's a good idea to explore the code before you start.  You'll see it is the tutorial code from the ASP.NET6 MVC tutorial, with two exceptions:

1. I changed the model to support the same properties I use in the Azure Mobile Apps testing.  This was done because I wanted to seed the database and had a corpus of data for that model.
2. I updated the `DbContext` to seed data into the database the first time it is used.

What I've got at this point is a fully functional web application with a database backend.

## What are we going to do?

Eventually, we'll want a database model that supports both the web and the mobile access methods and two view models - one that supports the web and one that supports the mobile side of things.  In addition, we'll have to update the database appropriately to ensure that the mobile specific properties are updated automatically when the web side of things updates the database.  Once that is done, we can easily implement the mobile side of things.  My order of operation is:

* Stage 1: Modify the web side to use view models.
* Stage 2: Update the database model to support mobile entities.
* Stage 3: Write a repository and table controller to expose the right form for the mobile entities.

## Stage 1 : Modifying the web application

Part of the plan here is to have two views into the same database model.  In the `before` tag, the database model is used in the views.  This isn't a really good way of doing things anyhow and I recommend using view models whenever you set up one of these projects.  Firstly, I created an `IMovie` interface that has all the data fields in the `Movie` database model.  I adjusted the `Movie` database model so that it implements `IMovie`, and I removed all the data annotations that weren't database specific.  Then I created a `MovieViewModel`:

```csharp
public class MovieViewModel : IMovie
{
    public MovieViewModel()
    {
    }

    public MovieViewModel(Movie m)
    {
        Id = m.Id;
        BestPictureWinner = m.BestPictureWinner;
        Duration = m.Duration;
        Rating = m.Rating;
        ReleaseDate = m.ReleaseDate;
        Title = m.Title;
        Year = m.Year;
    }

    public Movie ToMovie()
    {
        return new Movie()
        {
            Id = Id,
            BestPictureWinner = BestPictureWinner,
            Duration = Duration,
            Rating = Rating,
            ReleaseDate = ReleaseDate,
            Title = Title,
            Year = Year
        };
    }

    /// <summary>
    /// A application specific unique identifier for the movie.
    /// </summary>
    public int Id { get; set; }

    /// <summary>
    /// True if the movie won the oscar for Best Picture
    /// </summary>
    public bool BestPictureWinner { get; set; }

    /// <summary>
    /// The running time of the movie
    /// </summary>
    [Required]
    [Range(30, 360)]
    public int Duration { get; set; }

    /// <summary>
    /// The MPAA rating for the movie, if available.
    /// </summary>
    public MovieRating Rating { get; set; } = MovieRating.Unrated;

    /// <summary>
    /// The release date of the movie.
    /// </summary>
    [DisplayName("Release Date")]
    [DataType(DataType.Date)]
    [Required]
    public DateOnly ReleaseDate { get; set; }

    /// <summary>
    /// The title of the movie.
    /// </summary>
    [Required]
    [StringLength(250, MinimumLength = 1)]
    public string Title { get; set; } = "";

    /// <summary>
    /// The year that the movie was released.
    /// </summary>
    public int Year { get; set; }
}
```

One of the main things I'm going to point out here is that I've deliberately created a constructor that takes a `Movie` database model and a `.ToMovie()` method that converts the data back to a database model.  With this in hand, I updated all the views to use the `MovieViewModel` and then updated the `MoviesController` to convert to/from the view model when retrieving or storing data in the database.

You can see all the changes I made by [looking at the `stage1` tag](https://github.com/adrianhall/zumo-intid-server/releases/tag/stage1), or by looking at [the difference between the `before` and `stage1` tags](https://github.com/adrianhall/zumo-intid-server/commit/41838503a82c9e4a92acc128f1fabaf9708dc4cf).

## Stage 2 : Update the database model

The mobile requirements for a model are codified in the [`ITableData`](https://github.com/Azure/azure-mobile-apps/blob/6.0.7/sdk/dotnet/src/Microsoft.AspNetCore.Datasync.Abstractions/Interfaces/ITableData.cs) interface.  It comes down to:

* The `Id` field that is exposed should be a string.
* The `Version` field is a byte array that changes on every write.
* The `UpdatedAt` field is the last date/time that the record was updated.
* The `Deleted` field is a boolean that indicates if the record has been deleted.

The `Id` field is set by the client so that we don't have collisions between clients.  The `Version` field is used in concurrency checks to ensure two disconnected clients have not updated the record at the same time.  The `UpdatedAt` field is used in incremental synchronization, allowing the client to pull only the updated records.  Finally, the `Deleted` field is used to "soft-delete" records, allowing other clients to be notified that a record has been deleted.

Since we have the web side (and, potentially, other clients) updating the database and they have no understanding of these requirements, we need the database itself to set these when they are not set appropriately.  The first thing I did was to add these fields to the database model:

```csharp
public string MobileId { get; set; } = Guid.NewGuid().ToString("N");

[Timestamp]
public byte[] Version { get; set; } = [];

public DateTimeOffset? UpdatedAt { get; set; }

public bool Deleted { get; set; }
```

I also need to add indices on the MobileId, Deleted, and UpdatedAt properties, which I do in the `OnModelCreating()` method of the database context.  

```csharp
  modelBuilder.Entity<Movie>().Property(m => m.MobileId)
      .HasDefaultValueSql("NEWID()");
  modelBuilder.Entity<Movie>().Property(m => m.UpdatedAt)
      .HasDefaultValueSql("GETUTCDATE()");
  modelBuilder.Entity<Movie>()
      .HasIndex(m => m.MobileId).IsUnique();
  modelBuilder.Entity<Movie>()
      .HasIndex(m => m.UpdatedAt);
  modelBuilder.Entity<Movie>()
      .HasIndex(m => m.Deleted);
```

This is still not enough, however.  That's because I've got a lot of records without this information.  So, first I use `Add-Migration Stage2` to create a migration, but don't apply it yet.  I'm going to make some adjustments.  

The first change is to ensure that the `UpdatedAt` field is correctly updated irrespective of how the change to the row is made.  I do this with a trigger:

```csharp
  migrationBuilder.Sql(@"
      CREATE TRIGGER [trg_Movie_Mobile] ON [Movies]
      AFTER UPDATE AS
          UPDATE [Movie]
          SET [UpdatedAt] = GETUTCDATE()
          WHERE ID IN (SELECT [Id] FROM Inserted)
  ");
```

The initial value of the property is already set to `GETUTCDATE()` when inserted, so I only need to adjust the value if the row has been updated. The second change (and it must follow the trigger) is to update any row that does not have a the mobile fields set.  Since `UpdatedAt` is nullable, I can assume that any field that has a null `UpdatedAt` has not been adjusted, and the adjustment can be made.  Note that I only have to update the `MobileId` and the `UpdatedAt` fields - the `Version` and `Deleted` fields are already handled by the database.

```csharp
  migrationBuilder.Sql(@"
      UPDATE [Movie]
      SET
          [MobileId] = NEWID(),
          [UpdatedAt] = GETUTCDATE()
      WHERE [UpdatedAt] IS NULL
  ");
```

Finally, I remove all those calls to `migrationBuilder.UpdateData()` as those are not needed since I am bulk-updating all records.  This makes the migration much smaller (and much faster as well).

> **Tip**
> Don't forget to update the `Down()` method in your migration to drop the trigger as well!

At this point, you can run `Update-Database` to udpate the database.  Then peek at the data to make sure everything is set up properly.  I'm not a solid SQL person, so getting the specific calls took me a few attempts to get right.  I basically used the SQL Server console to run SQL commands inside a transaction until I got it right.  The transaction allowed me to roll back my changes if I screwed up (which I did - a few times!)

You can see the code thus far [in the `stage2` tag](https://github.com/adrianhall/zumo-intid-server/releases/tag/stage2) on the GitHub repository.  You should definitely run the application and make sure you can still make edits tot he dataset.  Also use the SQL Server Object Explorer to make sure that "the right thing" happens when you edit a record.  The `UpdatedAt` field and `Version` fields should be updated, and the `MobileId` field should not be updated.

> **Bonus**
> Now that you have "soft-delete", you should do the following to your web application:
>
> * Update the `Index` page to filter for `.Where(m => !m.Deleted)`.
> * Update the `Delete` post action to set the `Deleted` property to true instead.
> * Add an editor on the `Edit` page to "un-delete" a record if it is deleted.
>
> This will ensure that the web side "plays nice" with the mobile side.

## Stage 3: Mobile Table Controllers

Now that I have a database with a mobile-compatible table, I can work on creating the table controller.  For this project, that means going through all four steps:

* Update the web project to bring in the Azure Mobile Apps libraries.
* Create a model entity based on `ITableData`.
* Create a repository based on `IRepository{T}`.
* Create the table controller that uses the repository.

If you've used Azure Mobile Apps before, this should be familiar.  I'm not going to go through updating the web project since the documentation covers that pretty well.  My mobile model looks like this:

```csharp
public class MobileMovie : ITableData, IMovie
{
    public MobileMovie()
    {
    }

    public MobileMovie(Movie m)
    {
        Id = m.MobileId;
        Version = [.. m.Version];
        UpdatedAt = m.UpdatedAt ?? DateTimeOffset.UtcNow;
        Deleted = m.Deleted;
        m.CopyTo(this);
    }

    public Movie ToMovie() 
    {
        return new Movie() 
        {
            MobileId = Id,
            Version = [.. Version],
            UpdatedAt = UpdatedAt,
            Deleted = Deleted,
            BestPictureWinner = BestPictureWinner,
            Duration = Duration,
            Rating = Rating,
            ReleaseDate = ReleaseDate,
            Title = Title,
            Year = Year
        };
    }

    #region IMovie
    /// <summary>
    /// True if the movie won the oscar for Best Picture
    /// </summary>
    public bool BestPictureWinner { get; set; }

    /// <summary>
    /// The running time of the movie
    /// </summary>
    [Required]
    [Range(30, 360)]
    public int Duration { get; set; }

    /// <summary>
    /// The MPAA rating for the movie, if available.
    /// </summary>
    public MovieRating Rating { get; set; } = MovieRating.Unrated;

    /// <summary>
    /// The release date of the movie.
    /// </summary>
    [Required]
    public DateOnly ReleaseDate { get; set; }

    /// <summary>
    /// The title of the movie.
    /// </summary>
    [Required]
    [StringLength(250, MinimumLength = 1)]
    public string Title { get; set; } = string.Empty;

    /// <summary>
    /// The year that the movie was released.
    /// </summary>
    public int Year { get; set; }
    #endregion

    #region ITableData
    /// <inheritdoc />
    public string Id { get; set; } = Guid.NewGuid().ToString("N");

    /// <inheritdoc />
    public byte[] Version { get; set; } = [];

    /// <inheritdoc />
    public DateTimeOffset UpdatedAt { get; set; } = DateTimeOffset.UtcNow;

    /// <inheritdoc />
    public bool Deleted { get; set; }
    #endregion

    public bool Equals(ITableData? other)
        => other != null && Id == other.Id && Version.SequenceEqual(other.Version);
}
```

This model has all the data properties from the original database model plus the properties needed to implement the `ITableData` interface.  In addition, I've decorated the properties with the same data annotations for data validation that I used in the web view model.  Finally, I added a constructor and the `.ToMovie()` method that explicitly converts between database model and the mobile model, including the conversion from `MobileId` to `Id`.  

> **Can't I use Automapper?**
> Yes, of course you can use Automapper.  I tend to skip using Automapper in favor of coding explicit conversions.  If you use Automapper, then you will need to do adjustments to do the conversions for you in the right places.

Next, I need to write a repository.  The repository pattern is [relatively easy](https://github.com/Azure/azure-mobile-apps/blob/6.0.7/sdk/dotnet/src/Microsoft.AspNetCore.Datasync.Abstractions/Interfaces/IRepository.cs).  There are also several versions already created, but those don't work directly for this project because they all take something that implements `ITableData` and the database model does not follow that.  We can, however, use [the `EntityTableRepository{T}` class](https://github.com/Azure/azure-mobile-apps/blob/6.0.7/sdk/dotnet/src/Microsoft.AspNetCore.Datasync.EFCore/EntityTableRepository.cs) as a model and just adjust it for our needs.

In this case:

* We'll make a repository that is explicitly dependent on the mobile model.
* Each method will convert to/from the database model before returning data.

First off, let's take a look at the head of the new repository:

```csharp

/// <summary>
/// Create a new <see cref="EntityTableRepository{TEntity}"/> for accessing the database.
/// This is the normal ctor for this repository.
/// </summary>
/// <param name="context">The <see cref="DbContext"/> for the backend store.</param>
public class MovieRepository(MvcMovieContext context) : IRepository<MobileMovie>
{
    /// <summary>
    /// The EF Core <see cref="DbContext"/> for requests to the backend.
    /// </summary>
    internal MvcMovieContext Context { get; } = context ?? throw new ArgumentNullException(nameof(context));

    /// <summary>
    /// The <see cref="DbSet{TEntity}"/> for the data set within EF Core.
    /// </summary>
    internal DbSet<Movie> DataSet { get => Context.Movie; }

    // ... Rest of the class
}
```

I'm using one of my favorite new features of C# - the primary constructor.  It makes single constructor classes that just assign variables so much more compact.  I'm setting up the context and the dataset, and I've removed the generic arguments in this case.

> **Can I do this and still have a generic repository?**
> Yes, you can, but it takes additional interfaces (primarily for the three different metadata types) and code.  Unless you have more then 3 or 4 mobile tables, it's not worth it.

Reading is relatively simple:

```csharp
  public IQueryable<MobileMovie> AsQueryable() => DataSet.Select(m => new MobileMovie(m)).AsQueryable();

  public Task<IQueryable<MobileMovie>> AsQueryableAsync() => Task.FromResult(AsQueryable());

  public async Task<MobileMovie> ReadAsync(string id, CancellationToken token = default)
  {
    if (string.IsNullOrEmpty(id))
    {
      throw new BadRequestException();
    }
    Movie movie = await DataSet.FirstOrDefaultAsync(m => m.MobileId == id, token)
      ?? throw new NotFoundException();
    return new MobileMovie(movie);
  }
```

One of the notes about writing repositories is that entities must be "disconnected from the store."  This means that if I change the return type after returning from a repository method, it should not change the underlying store.  I accomplish this as part of creating the `MobileMovie`.  The data is automatically disconnected since I copy it into a new object.

I'm not going to show off all three of the write operations (Create, Delete, Replace).  Instead, I'm going to focus on one of them.  You can go check out [the source code](https://github.com/adrianhall/zumo-intid-server/releases/tag/stage3) to see the changes I made to the other two.  Let's focus on the create operation:

```csharp
  public async Task CreateAsync(MobileMovie entity, CancellationToken token = default)
  {
      ArgumentNullException.ThrowIfNull(entity);
      try
      {
          if (entity.Id != null && DataSet.Any(x => x.MobileId == entity.Id))
          {
              Movie conflictingEntity = await DataSet.FirstAsync(x => x.MobileId == entity.Id, token).ConfigureAwait(false);
              throw new ConflictException(new MobileMovie(conflictingEntity));
          }
          Movie movie = entity.ToMovie();
          DataSet.Add(movie);
          await Context.SaveChangesAsync(token).ConfigureAwait(false);
      }
      catch (DbUpdateException ex)
      {
          throw new RepositoryException(ex.Message, ex);
      }
  }
```

As far a code goes, this is pretty simple stuff!  The database takes care of setting the `Id` when adding a record here.  One particular note is that I can't use `FindAsync()` any more to grab a copy of the entity.  The `FindAsync()` method uses the primary key and `MobileId` is not the primary key.  Instead, I have to use `FirstAsync()` or `FirstOrDefaultAsync()` method to grab the record.

Finally, I can create a completely normal `MobileMovieController` which is based on `TableController<MobileMovie>` and that uses my repository.

> **Tip**
> You can add repositories to the service collection in your program.  For Entity Framework Core based repositories, make sure you use `.AddScoped<>`.

Here is my controller:

```csharp
[Route("api/mobile/movie")]
public class MobileMovieController : TableController<MobileMovie>
{
    public MobileMovieController(MvcMovieContext context) : base()
    {
        Repository = new MovieRepository(context);
    }
}
```

You can add access control, options, and do anything else you would normally do to a table controller.  It works just the same way.  The best way to test this is with a REST API tester like [Postman](https://getpostman.com).  For instance, I ran the `GET /api/mobile/movie` to see what came up:

![The postman results from running a query]({{ site.baseurl }}/assets/images/2023/2023-12-22-125311.png)

The things to check here:

* Is the `id` field a string GUID?
* Is the `updatedAt` field set?
* Is the `version` field set?

This one looks good!  Don't forget to test POST (for creating), DELETE (for, well, deleting) and PUT (for replacing) as well.  Finally, if using this in production, don't forget to add unit tests for the conversions to/from the database models and for the repository.  There is a great starter set of xUnit based tests in the `Datasync.Integration.Tests` project in the [azure-mobile-apps repository](https://github.com/azure/azure-mobile-apps).

I hope this has been informative for you.  If you have questions, don't hesitate to reach out on the [azure-mobile-apps repository](https://github.com/azure/azure-mobile-apps).  I'm active in both discussions and issues!
