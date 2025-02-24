---
title: "Introducing Azure Mobile Apps for ASP.NET 6"
categories:
  - Azure
tags:
  - ASP.NET Core
  - Azure Mobile Apps
---

It's release day for .NET 6, and I couldn't be happier to introduce Azure Mobile Apps for ASP.NET 6 being released on the same day.  When I started developing the new code-base, I had several aims:

* Backwards compatibility with the older clients.
* Develop for ASP.NET 6 so I can develop anywhere - Windows, Mac, or Linux.
* Actually run anywhere - containers, App Service, or even a VM.
* Bring your own authentication provider and database provider.

That meant some radical changes in the way that code is presented.  Over the next few blog posts, I'll introduce you to a lot of the new concepts and code that you will want to write.  I've also written a handy [HOW TO guide](https://docs.microsoft.com/en-us/azure/developer/mobile-apps/azure-mobile-apps/howto/server/dotnet-cor).  There are also [discussion boards](https://github.com/Azure/azure-mobile-apps/discussions) for more in depth conversations.

## Getting your environment set up

Obviously, you will want to [install .NET 6](https://docs.microsoft.com/dotnet/core/install/).  I'm writing this entire blog on an Ubuntu 21.04 based system, and I've installed dotnet through snap.  I've also run through this tutorial on Windows and Mac, so it should run everywhere.  You can validate your toolchain is set up properly like this:

{% highlight bash %}
$ dotnet sdk check
.NET SDKs:
Version      Status     
------------------------
5.0.403      Up to date.
6.0.100      Up to date.

.NET Runtimes:
Name                          Version      Status     
------------------------------------------------------
Microsoft.AspNetCore.App      5.0.12       Up to date.
Microsoft.NETCore.App         5.0.12       Up to date.
Microsoft.AspNetCore.App      6.0.0        Up to date.
Microsoft.NETCore.App         6.0.0        Up to date.


The latest versions of .NET can be installed from https://aka.ms/dotnet-core-download. For more information about .NET lifecycles, see https://aka.ms/dotnet-core-support.
{% endhighlight %}

Secondly, if you want to follow along, [install the latest Azure CLI](https://docs.microsoft.com/cli/azure/install-azure-cli).  Yes, I know you can deploy anywhere, but Azure makes this easy.  If you want to deploy to AWS, Google Cloud, or any other provider, then make sure you can run the ASP.NET 6 application.  The broad steps are the same between clouds, but the way of achieving them is not.  

Once you've install the Azure CLI, make sure you run `az login` and any commands to select a subscription.  The instructions are in the installation guides.  You can check to see if it's working using `az account show`:

{% highlight bash %}
$ az account show
{
  "environmentName": "AzureCloud",
  "homeTenantId": "{redacted-guid}",
  "id": "{redacted-guid}",
  "isDefault": true,
  "managedByTenants": [],
  "name": "Visual Studio Enterprise",
  "state": "Enabled",
  "tenantId": "{redacted-guid}",
  "user": {
    "name": "{your-login-id}",
    "type": "user"
  }
}
{% endhighlight %}

Now you are ready to go!

# Install the Azure Mobile Apps template

Azure Mobile Apps is an ASP.NET Core service that exposes an Entity Framework Core based table via a Web API, with full OData for searching and ordering the table, plus CRUD operations with ETag support for conflict resolution.  You could create a Web API (using `dotnet new webapi`), then add Entity Framework Core, then add the Azure Mobile Apps libraries, before creating a data transfer object and controller.

Or you could just use the template as a starting point. 

To install the template:

{% highlight bash %}
$ dotnet new -i Microsoft.AspNetCore.Datasync.Template.CSharp
The following template packages will be installed:
   Microsoft.AspNetCore.Datasync.Template.CSharp

Success: Microsoft.AspNetCore.Datasync.Template.CSharp::5.0.0-beta.8 installed the following templates:
Template Name                 Short Name       Language  Tags                   
----------------------------  ---------------  --------  -----------------------
ASP.NET Core Datasync Server  datasync-server  [C#]      Cloud/Web/Mobile/WebAPI
{% endhighlight %}

# Create a project

Next step - let's create a project!

{% highlight bash %}
$ mkdir myproject
$ cd myproject
$ dotnet new datasync-server
The template "ASP.NET Core Datasync Server" was created successfully.
{% endhighlight %}

Let's take a look at the project.  Open it with Visual Studio Code or your favorite editor.  Things to note:

* There is a `Program.cs` that contains the startup code.
* There is an `Models\AppDbContext.cs` that contains the Entity Framework Core context object.
* There is a `Models\TodoItem.cs` that contains the **Data Transfer Object** (DTO) that I want to project to clients.
* There is a `Controllers\TodoitemController.cs` that contains the code for the endpoints.

Let's look at `Models\TodoItem.cs` first - our DTO:

{% highlight csharp %}
using Microsoft.AspNetCore.Datasync.EFCore;
using System.ComponentModel.DataAnnotations;

namespace myproject.Db
{
    public class TodoItem : EntityTableData
    {
        [Required, MinLength(1)]
        public string Title { get; set; } = "";

        public bool IsComplete { get; set; }
    }
}
{% endhighlight %}

All DTOs are based on a concrete implementation of `ITableData` that contains the fields necessary to implement incremental sync, soft delete, and conflict resolution.  The `EntityTableData` class is the implementation for Entity Framework Core.  So you don't have to implement `Id`, `UpdatedAt`, `Version`, or `Deleted` - those are specified for you.

Now, let's move on to the controller:

{% highlight csharp %}
using Microsoft.AspNetCore.Datasync;
using Microsoft.AspNetCore.Datasync.EFCore;
using Microsoft.AspNetCore.Mvc;
using myproject.Db;

namespace myproject.Controllers
{
    [Route("tables/todoitem")]
    public class TodoItemController : TableController<TodoItem>
    {
        public TodoItemController(AppDbContext context)
            : base(new EntityTableRepository<TodoItem>(context))
        {
        }
    }
}
{% endhighlight %}

Yep - that's it.  An entire set of endpoints anchored on the specified route that stores data in the provided Entity Framework context and handles full versioning, conflict resolution, and incremental synchronization.  Note that you have to specify the route to be "tables/{dtoname}` in order to be backwards compatible.  However, it isn't required when you are using the newest client.  Also, this implements an open fully read-write table with no authentication, which is probably a bad thing and not what is wanted.  I'll get onto authentication later on.

Now, let's look at that startup code in `Program.cs`:

{% highlight csharp %}
using Microsoft.AspNetCore.Datasync;
using Microsoft.EntityFrameworkCore;
using myproject.Db;

var builder = WebApplication.CreateBuilder(args);
var connectionString = builder.Configuration.GetConnectionString("DefaultConnection");

builder.Services.AddDbContext<AppDbContext>(options => options.UseSqlServer(connectionString));
builder.Services.AddDatasyncControllers();

var app = builder.Build();

// Initialize the database
using (var scope = app.Services.CreateScope())
{
    var context = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    await context.InitializeDatabaseAsync().ConfigureAwait(false);
}

// Configure and run the web service.
app.MapControllers();
app.Run();
{% endhighlight %}

This probably looks a little strange if you've been used to the older style ASP.NET Framework and ASP.NET Core startup styles.  There isn't a class!  However, you should be able to identify the setup for Entity Framework Core.  The only additional line you need here is the `builder.Services.AddDatasyncControllers();` and `app.MapControllers();`.  The first sets up all the services necessary to run Azure Mobile Apps, and the second links the controllers into the route map so they can be accessed.

## Change to an InMemory Store

There is another mechanism other than Entity Framework Core.  I've also provided a basic in-memory store.  Here is how to use it.

1. Add the `Microsoft.AspNetCore.Datasync.InMemory` package to the project:

    {% highlight bash %}
    $ dotnet add package Microsoft.AspNetCore.Datasync.InMemory --prerelease
    {% endhighlight %}

2. In `TodoItem.cs` change `EntityTableData` to `InMemoryTableData`.
3. Change `TodoitemController.cs` to the following:

    {% highlight csharp %}
    using Microsoft.AspNetCore.Datasync;
    using Microsoft.AspNetCore.Mvc;
    using myproject.Db;

    namespace myproject.Controllers
    {
        [Route("tables/todoitem")]
        public class TodoItemController : TableController<TodoItem>
        {
            public TodoItemController(IRepository<TodoItem> repository)
                : base(repository)
            {
            }
        }
    }
    {% endhighlight %}

4. Remove the Entity Framework Core stuff from `Program.cs` and add a singleton for the store:

    {% highlight csharp %}
    using Microsoft.AspNetCore.Datasync;
    using Microsoft.AspNetCore.Datasync.InMemory;
    using Microsoft.EntityFrameworkCore;
    using myproject.Db;

    var builder = WebApplication.CreateBuilder(args);

    builder.Services.AddSingleton<IRepository<TodoItem>>(new InMemoryRepository<TodoItem>());
    builder.Services.AddDatasyncControllers();

    var app = builder.Build();

    // Configure and run the web service.
    app.MapControllers();
    app.Run();
    {% endhighlight %}

Entity Framework Core likes to have a context object per controller, and handles the singleton datastore for us.  However, the in-memory repository doesn't have that luxury, so we have to create a singleton and then inject it into the controller.

## Run the app

Now we can run the app everywhere, and not have to worry about provisioning a database.  Make sure you have saved all the files, then run:

{% highlight bash %}
$ dotnet run
info: Microsoft.Hosting.Lifetime[14]
      Now listening on: http://localhost:5000
info: Microsoft.Hosting.Lifetime[14]
      Now listening on: https://localhost:5001
info: Microsoft.Hosting.Lifetime[0]
      Application started. Press Ctrl+C to shut down.
info: Microsoft.Hosting.Lifetime[0]
      Hosting environment: Production
info: Microsoft.Hosting.Lifetime[0]
      Content root path: /home/adrian/GitHub/myproject/
{% endhighlight %}

You will be able to get some output by visiting `https://localhost:5001/tables/todoitem?ZUMO-API-VERSION=3.0.0`.  However, if you really want to exercise the service, you are going to have to use [Insomnia](https://insomnia.rest/) or [Postman REST client](https://www.postman.com/product/rest-client/) to send some REST commands!  Supported commands (where `{endpoint}` is the `/tables/todoitem` right now)

* `GET {endpoint}` to get a list of all entities (paged, plus you can use OData v4 syntax)
* `GET {endpoint}/{id}` to get a single entity
* `POST {endpoint}` (payload is an object with the properties in your DTO) to create an entity
* `PUT {endpoint}/{id}` (payload is the complete object with updated fields) to replace an entity
* `PATCH {endpoint}/{id}` (payload is either the replaced fields as an object or a JSON Patch object) to alter an entity
* `DELETE {endpoint}/{id}` to delete the entity.

Make sure you also send `ZUMO-API-VERSION: 3.0.0` as a header (or `ZUMO-API-VERSION=3.0.0` as a query param) to select the Azure Mobile Apps protocol level.

Experiment!

## Deploy to Azure

Now, let's deploy the service to Azure.  This will create a new (free) Azure App Service with the same service running:

{% highlight bash %}
$ az webapp up --name myawesomeproject --location westus --sku FREE --runtime "DOTNET|6.0"
{% endhighlight %}

That's a deceptively simple command that does all the work of creating and deploying a web application for you.  It won't connect a database (but we don't need one because we've switched over to an in-memory database), so there is still work to be done if you want to provision a full web app connected to a database.

> The original Azure Mobile Apps only ran on App Service.  The new Azure Mobile Apps is "just an ASP.NET Core application", so you can run it wherever ASP.NET Core applications run - including container based systems like Azure Kubernetes Service or Azure Container Apps.

## Some things to try out

Note that we are not tied to SQL Azure any more.  You can try anything - SQLite needs a little more work than most (since it has a deficient DateTime implementation that doesn't have millisecond resolution), but [Cosmos DB](https://docs.microsoft.com/ef/core/providers/cosmos/?tabs=dotnet-core-cli), [MySQL](https://github.com/PomeloFoundation/Pomelo.EntityFrameworkCore.MySql), or [PostgreSQL](https://www.npgsql.org/efcore/) are all possibilities, along with others.  Check out [the database notes in the HOW TO guide](https://docs.microsoft.com/azure/developer/mobile-apps/azure-mobile-apps/howto/server/dotnet-core) for important support notes.

## Backwards compatible - almost

One of the goals of the project was to be backwards compatible with the older clients.  And we got there - almost.  If you send `ZUMO-API-VERSION: 2.0.0` as a header, you'll get the older functionality.  It's "mostly" compatible because there are some differences between OData v3 and OData v4 we couldn't work around.  Most notably, the ability to do substring searches (either as part of a sync query or as a LINQ query on the table) is gone.  If you rely on this functionality, you will have to stay on the .NET Framework version for now.  You won't be able to do substring searches until the new clients are released.

## Next steps

In the next blog post, I'll show how incredibly easy it is to set up Azure App Service Authentication, and other authentication implementations that are possible with the new codebase.  Stay tuned!
