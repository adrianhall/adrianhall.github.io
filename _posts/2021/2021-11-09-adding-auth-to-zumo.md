---
title: "Add authentication to Azure Mobile Apps for ASP.NET Core"
categories:
  - Cloud
tags:
  - aspnetcore
  - azure_mobile_apps
---

In [my last article]({% post_url 2021/2021-11-08-azure-mobile-apps-intro %}), I introduced the new ASP.NET Core edition of Azure Mobile Apps, including how to set up Entity Framework Core and in-memory stores.  Today, we are going to introduce simple authentication.  What do you need to do to secure your entire API?  We'll cover more complex authentication schemes (such as protecting a single API, or doing DTO transforms based on the identity of the user) next time.

## Azure App Service Authentication

Azure App Service includes a feature that most people call "EasyAuth".  It allows you to configure a Google, Microsoft, Facebook, or Twitter authentication service within Azure App Service.  You can read about the configuration process in [their documentation](https://docs.microsoft.com/azure/app-service/overview-authentication-authorization).  If you used Azure App Service to host an Azure Mobile Apps service previously, this will be familiar to you.

The new Azure Mobile Apps service components support Azure App Service directly.  Aside from setting up the Azure App Service (which hasn't changed), here is what you need to do:

1. Configure ASP.NET Core identity with the Azure Mobile Apps provided authentication provider.
2. Update the controllers to require authentication.

### Configure ASP.NET Core identity

I'll assume you have already created a new ASP.NET Core service using the datasync-server template.  Edit the `Program.cs` file as follows:

{% highlight csharp linenos %}
using Microsoft.AspNetCore.Authentication;
using Microsoft.AspNetCore.Datasync;
using Microsoft.EntityFrameworkCore;
using t.Db;

var builder = WebApplication.CreateBuilder(args);
var connectionString = builder.Configuration.GetConnectionString("DefaultConnection");

builder.Services.AddDbContext<AppDbContext>(options => options.UseSqlServer(connectionString));
builder.Services.AddAuthentication(AzureAppServiceAuthentication.AuthenticationScheme)
    .AddAzureAppServiceAuthentication(options => options.ForceEnable = true);
builder.Services.AddDatasyncControllers();

var app = builder.Build();

// Initialize the database
using (var scope = app.Services.CreateScope())
{
    var context = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    await context.InitializeDatabaseAsync().ConfigureAwait(false);
}

// Configure and run the web service.
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
app.Run();
{% endhighlight %}

The first two lines (10-11) configure the Azure App Service Authentication service to be primary.  The second two lines add the standard ASP.NET Core identity services to the application.  Your controllers will now see the users identity in the `HttpContext`.

### Add Authorize tag to controllers

The controller now becomes:

{% highlight csharp linenos %}
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Datasync;
using Microsoft.AspNetCore.Datasync.EFCore;
using Microsoft.AspNetCore.Mvc;
using t.Db;

namespace t.Controllers
{
    [Authorize]
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

Line 9 is the bit that does the work.  Your API is now protected.

## Yes, this is standard ASP.NET Core Identity

We don't do anything special within the new service library. That means you can use any authentication you want.  For example, it's [relatively easy](https://docs.microsoft.com/aspnet/core/security/authentication/?view=aspnetcore-6.0) to secure the API with Azure Active Directory, certificates, or anything else that ASP.NET Identity supports.

## Client Library Support

There are now two sets of libraries - the older (existing) Azure Mobile Apps libraries support Azure App Service Authentication out of the box.  However, you can write a `DelegatingHandler` that provides any other type of authentication. Thus, for example, you can [use MSAL](https://thewissen.io/implementing-msal-authentication-in-xamarin-forms/) to get an access token, then inject that access token as an Authorization header into every single request. Create the client with:

{% highlight csharp %}
var client = new MobileServiceClient(serviceUri, new MyDelegatingHandler());
{% endhighlight %}

There is also the newer client.  This provides an `AuthenticationProvider` that will inject the token for you.  For example:

{% highlight csharp %}
var authProvider = new GenericAuthenticationProvider(() => GetAccessToken());
var client = new DatasyncClient(serviceUri, authProvider);
{% endhighlight %}

The authentication provider is similarly a delegating handler at its heart, but contains additional code to handle refreshing the access token and caching the access token in memory while it is valid.  

If you are starting a new project, use the newer libraries.

## Next steps

In the next blog post, I'll show you how you can use the ASP.NET Core Identity to handle complex authorization capabilities, such as personal tables and read-only tables.
