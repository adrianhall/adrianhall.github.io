---
title:  "ASP.NET Identity deep dive - Part 1 (Project setup)"
date:   2024-09-11
categories:
  - "ASP.NET Core"
tags:
  - "ASP.NET Core"
  - "ASP.NET Identity"
---

You may have noticed that I included ASP.NET Identity in a project [a couple of posts ago]({% post_url 2024/2024-09-05-aspire-identity %}).  I'm currently doing a deep dive into ASP.NET Identity with an eye towards an OIDC identity service based on [OpenIddict](https://documentation.openiddict.com/).

Identity is a complex topic and I still recommend that developers integrate another service rather than write their own: 

* [Keycloak](https://www.keycloak.org/) is a good option if you have to store your own data, 
* [Auth0](https://auth0.com/) is a good option when you just want a bunch of social providers,
* [Corbado](https://www.corbado.com/) has support for Face ID, Touch ID, and PassKeys for a more modern approach.

All of these options will have you writing less code and getting to the real meat of your application faster but still provides a secure identity solution. If you don't have a good reason for writing your own identity service, then take a look at these options.

But you (like me for this project) have decided to go it alone. How is it done? This article is one of a number of articles I will write over the coming month and will go into depth about the [ASP.NET Identity](https://learn.microsoft.com/aspnet/core/security/authentication/identity) system.  My outline thus far:

{% include_relative includes/aspnet-identity-series.md %}

I'll then move onto OIDC support from there.  However, I'm unlikely to cover the OIDC stuff on this blog.  As I mentioned earlier, I don't think it's a good idea.

## Required knowledge

ASP.NET Identity is a simple API, but it does rely on pre-requisite knowledge.  Specifically, you should start with a solid understanding of the following topics:

* [Entity Framework Core](https://learn.microsoft.com/ef/core/)
* [ASP.NET MVC](https://dotnet.microsoft.com/apps/aspnet/mvc)

If you find yourself uncomfortable with databases and web development, perhaps you should review the linked content before continuing with ASP.NET Identity.  You'll find [a great tutorial](https://learn.microsoft.com/aspnet/core/tutorials/first-mvc-app/start-mvc) which takes you through developing a basic MVC database driven application.

## Database support

Where do you store the identity data? Microsoft provides support for Entity Framework Core (which is what pretty much everyone uses), and I've seen a third party MongoDB provider. However, you can write your own stores.  I could certainly see a high performance store which also uses Redis Cache, for instance, or - if you are into writing your own SQL statements - using Dapper.

For this project, I'm going to use a PostgreSQL database and Entity Framework Core.  You may have caught onto this if you read [my earlier article on Aspire]({% post_url 2024/2024-09-05-aspire-identity %}). With Aspire, I can set up the database and the identity service as containers and run them on Docker Desktop.

If you do want to write your own data access layer, see [the documentation about custom storage providers](https://learn.microsoft.com/aspnet/core/security/authentication/identity-custom-storage-providers).

Add the NuGet packages for your Entity Framework Core setup and the following two packages:

* [Microsoft.AspNetCore.Identity.EntityFrameworkCore](https://www.nuget.org/packages/Microsoft.AspNetCore.Identity.EntityFrameworkCore)
* [Microsoft.AspNetCore.Identity.UI](https://www.nuget.org/packages/Microsoft.AspNetCore.Identity.UI)

## Users and roles

Users sign in to applications, and they are given permissions via role-based access controls. All the basic model requirements for ASP.NET Identity are built into [`IdentityUser`](https://learn.microsoft.com/dotnet/api/microsoft.aspnetcore.identity.identityuser) and [`IdentityRole`](https://learn.microsoft.com/dotnet/api/microsoft.aspnetcore.identity.identityrole).  You can create your own models, but they must derive from these classes. It's normal, for instance, to add profile data to the user model.

My own situation requires that the user provide a "Display Name" field.  I've created a new user model class:

{% highlight csharp %}
public class ApplicationUser : IdentityUser
{
  public string? DisplayName { get; set; }
}
{% endhighlight %}

## Setting up the database context

Entity Framework Core requires that you set up a class that derives from `DbContext`.  ASP.NET Identity requires that you set up a class that derives from `IdentityDbContext` instead (which, in turn, derives from `DbContext`).  Here is the most basic version:

{% highlight csharp %}
public class ApplicationDbContext(DbContextOptions<ApplicationDbContext> options) 
  : IdentityDbContext<ApplicationUser>(options)
{
}
{% endhighlight %}

This will actually define a number of tables for storing users, roles, and supporting metadata.  You can, of course, add your own tables if you are incorporating identity into a monolith.

> **Use a separate database for your identity data**<br/>
> By separating your identity data from your application data, you get a separation of concerns.  This makes it easier to move your identity stuff to a separate microservice later on and it allows you to restrict access to the database separately.
> {: .notice--success}

If you need to adjust your model (for example, to add an index, change table names, or similar), you can do this within the `OnModelCreating()` method.  However, you must call the base version BEFORE making your changes.  i.e.

{% highlight csharp %}
public class ApplicationDbContext(DbContextOptions<ApplicationDbContext> options) 
  : IdentityDbContext(options)
{
    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);

        // Do any changes to the identity models here.
    }
}
{% endhighlight %}

Aside from these changes, it's a normal database context.  You can do anything you would normally do in a database context. 

If you are using a separate database for identity data (which is something I highly recommend), you should have two database contexts - one for your normal application data and one for your identity data.

## Database initialization

You can (and should) use data migrations or SQL scripts in production to initialize the database.  However, you may want to seed the database with some users when running in development.  Use the `UserManager<TUser>` object from dependency injection to add users properly.  Here is a code snippet for you:

{% highlight csharp %}
internal async Task EnsureUserExistsAsync(SeedUser userRecord)
{
    ApplicationUser? user = await userManager.FindByNameAsync(userRecord.UserName);
    if (user is not null)
    {
        return;
    }

    ApplicationUser newUser = new()
    {
        Id = Guid.NewGuid().ToString(),
        Email = $"{userRecord.UserName.ToLowerInvariant()}@contoso-email.com",
        EmailConfirmed = true,
        UserName = userRecord.UserName,
        DisplayName = userRecord.DisplayName
    };
    IdentityResult? result = await userManager.CreateAsync(newUser, defaultPassword);
    ThrowIfNotSuccessful(result, $"Could not create user '{userRecord.UserName}'");
}

internal record SeedUser(string UserName, string DisplayName, List<string>? Roles = null);
{% endhighlight %}

Check out [my ApplicationDbInitializer](https://github.com/adrianhall/samples/blob/0911/identity/Samples.Identity/Data/ApplicationDbInitializer.cs) for more reusable methods for seeding the database and for a full example that includes role assignments as well.

## Configuring identity

I've got an MVC application that I created with `dotnet new mvc`.  Here is my `Program.cs`

{% highlight csharp %}
var builder = WebApplication.CreateBuilder(args);

// .NET Aspire
builder.AddServiceDefaults();

// Database services
builder.AddNpgsqlDbContext<ApplicationDbContext>("identitydb", configureDbContextOptions: options =>
{
    if (builder.Environment.IsDevelopment())
    {
        options.EnableDetailedErrors();
        options.EnableSensitiveDataLogging();
        options.EnableThreadSafetyChecks();
    }
});

builder.Services.AddScoped<IDbInitializer, ApplicationDbInitializer>();

// ASP.NET Identity
builder.Services
    .AddIdentity<ApplicationUser, IdentityRole>(options => 
    {
      options.SignIn.RequireConfirmedAccount = true;
      options.User.RequireUniqueEmail = true;
    })
    .AddEntityFrameworkStores<ApplicationDbContext>()
    .AddDefaultTokenProviders();

// ASP.NET MVC
builder.Services.AddControllersWithViews();

// =========================================================
// HTTP Pipeline
// =========================================================
var app = builder.Build();

// .NET Aspire
app.MapDefaultEndpoints();

// Database initialization
using (IServiceScope scope = app.Services.CreateScope())
{
  IDbInitializer initializer = scope.ServiceProvider.GetRequiredService<IDbInitializer>();
  await initializer.InitializeDatabaseAsync();
}

app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseRouting();

// ASP.NET Identity
app.UseAuthentication();
app.UseAuthorization();

// ASP.NET MVC
app.MapDefaultControllerRoute();

// =========================================================
// Run the Service
// =========================================================
app.Run();
{% endhighlight %}

Let's focus in on the `builder.Services.AddIdentity()` call.  There are lots of options, including:

* [Password policy](https://learn.microsoft.com/dotnet/api/microsoft.aspnetcore.identity.passwordoptions).
* [Account lockout](https://learn.microsoft.com/dotnet/api/microsoft.aspnetcore.identity.lockoutoptions).

One of the most common requirements for an identity service is account confirmation.  As we will see in the next article, we send the user a link that they need to use to confirm the account.  The `options.SignIn.RequireConfirmedAccount = true` option sets up ASP.NET Identity so that a confirmed account is required before they can sign in.  I'm going to be using the email address for my user ID, so I need to ensure that each user has a unique email address.

The `.AddDefaultTokenProviders()` is also a part of the account confirmation and reset password capabilities.  When we go through the process of confirming an account, we ask the system to generate a token that the user then must submit back to the system to validate that the email address is correct.  This is done with a token provider.  ASP.NET Identity provides default implementations of the token provider so that you don't have to worry about it.  If you want to use short and easily entered tokens (for example, if you are doing authentication for a set-top box), then you will want to override the default token providers.

> **Use extension methods to simplify setup**
> I don't put all this setup in `Program.cs`.  Instead I put the setup in extension methods.  This allows me to store the database setup with the database context and initializer, for instance.  Using extension methods also improves readability of your code.
> {: .notice--success}

## Scaffolding the UI Pages

ASP.NET Identity comes with a set of UI pages that you can scaffold into your project.  The UI pages are based on Razor Pages and there is a lot of them.  If you do use the scaffolding pages, don't forget to configure your project to use Razor Pages.  However, they do get the job done and will get you started quickly.  It's also good reference material as you work to implement your own UI.  Part of the allure of ASP.NET Identity is that you get to control every aspect of the sign in experience, including the UI.  As a result, I rarely use the stock pages.

You can scaffold the default UI by command line:

{% highlight bash %}
dotnet aspnet-codegenerator identity --useDefaultUI
{% endhighlight %}

Or you can use Visual Studio:

* Right-click on the project node.
* Select **Add** > **New Scaffolded item...**
* Select **Identity**.
* Complete the rest of the wizard.

I don't like the models, logic, and UI all munged together.  As a result, I inevitably roll my own UI using MVC (model-view-controller).  We'll get onto that in future articles.

## Final thoughts

Firstly, don't use ASP.NET Identity unless you really need to. You are re-inventing the wheel and are bound to introduce security vulnerabilities that you will not be aware of. There are solid options implemented as containers or services that you can use to avoid writing your own identity service.  Just because you need your own identities doesn't mean you need your own identity service.

However, if you've decided that you absolutely must have your own identity service, then ASP.NET Identity is a solid choice (and also the Microsoft approved choice).  It's easy to get started with a lot of helper code that is generated for you.  You should, however, be cogniscent of the problems with implementing your own.  The [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html) is a great starting point for this.

But, seriously, don't roll your own identity for a product unless the product is authentication.

## Further reading

* [The project so far](https://github.com/adrianhall/samples/tree/0911/identity)
* [ASP.NET Identity](https://learn.microsoft.com/aspnet/core/security/authentication/identity)
* [OAuth 2.0](https://oauth.net/2/)
* [OpenID Connect (OIDC)](https://openid.net/developers/how-connect-works/)
* [OpenIddict](https://documentation.openiddict.com/)
* [Keycloak](https://www.keycloak.org/) 
* [Auth0](https://auth0.com/)
* [Corbado](https://www.corbado.com/)
