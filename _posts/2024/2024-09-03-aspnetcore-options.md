---
title:  "Making ASP.NET Core applications readable - the options patterns"
date:   2024-09-03
categories:
  - "ASP.NET Core"
tags:
  - C#
---

Applications are read more often than they are written.  The normal situation when a developer comes onto a project is that anything from a couple of weeks to several months is requried to come "up to speed" on the code base. Making the efforts required for readability of the code is important, and I spend a ton of time up front to ensure my applications are understandable without needing an in-depth lesson from me on how it works.

Let's take an application I am starting (today actually).  I'm building a Microservices based application with a built-in OIDC identity service. Anyone who has played around with identity knows that there are some shared bits required.  The identity service needs to know about the clients and each client needs to know something about the identity service.  I'm not going to go into the specifics of how the identity service works.  I hope to do that in subsequent posts.  However, I do think it's worthwhile looking at the various options for handling application settings and to make the web application readable.

## Configuring the web application

Let's say I have an ASP.NET Core web application (which you can create with `dotnet new webapp` or via Visual Studio).  Usually, the `Program.cs` is a mass of configurable bits, but you can make it more readable by using extension methods and the options pattern. Let's start with extension methods.  Normally, adding an external identity service to your `Program.cs` will look something like this:

{% highlight csharp %}
JwtSecurityTokenHandler.DefaultInboundClaimTypeMap.Clear();

string myEndpoint = builder.Configuration["Services:Identity:Endpoint"];
string myClientId = builder.Configuration["Services:Identity:ClientId"];
string myClientSecret = builder.Configuration["Services:Identity:ClientSecret"];

builder.Services
    .AddAuthentication(config =>
    {
        config.DefaultScheme = CookieAuthenticationDefaults.AuthenticationScheme;
        config.DefaultChallengeScheme = OpenIdConnectDefaults.AuthenticationScheme;
    })
    .AddCookie(CookieAuthenticationDefaults.AuthenticationScheme)
    .AddOpenIdConnect(OpenIdConnectDefaults.AuthenticationScheme, options =>
    {
        options.Authority = settings.Endpoint;
        options.ClientId = settings.ClientId;
        options.ClientSecret = settings.ClientSecret;
        options.ResponseType = "code";
        options.CallbackPath = "/signin-oidc";
        options.SaveTokens = true;
        options.TokenValidationParameters = new()
        {
            ValidateIssuerSigningKey = false,
            SignatureValidator = (token, validationParameters) => new JwtSecurityToken(token)
        };
    });
{% endhighlight %}

This is a long piece of configuration and the details aren't important unless you happen to be working on the identity service.  When you add database configuration, view configuration, and any other services into the mix, you end up with a really long `Program.cs` that takes mental energy to decompose.  What you really need to do is to encapsulate this lot into an extension method.

{% highlight csharp %}
public static class IdentityServiceExtensions
{
  public const string IdentityServiceSection = "Services:Identity";

  public static IHostApplicationBuilder AddIdentityServices(this IHostApplicationBuilder builder)
  {
    IdentityServiceSettings settings = builder.Configuration.GetSection(IdentityServiceSection).Get<IdentityServiceSettings>()
      ?? throw new ApplicationException($"Configuration section {IdentityServiceSection} is not available");

    JwtSecurityTokenHandler.DefaultInboundClaimTypeMap.Clear();

    builder.Services
      .AddAuthentication(config =>
      {
        config.DefaultScheme = CookieAuthenticationDefaults.AuthenticationScheme;
        config.DefaultChallengeScheme = OpenIdConnectDefaults.AuthenticationScheme;
      })
      .AddCookie(CookieAuthenticationDefaults.AuthenticationScheme)
      .AddOpenIdConnect(OpenIdConnectDefaults.AuthenticationScheme, options =>
      {
        options.Authority = settings.Endpoint;
        options.ClientId = settings.ClientId;
        options.ClientSecret = settings.ClientSecret;
        options.ResponseType = "code";
        options.CallbackPath = "/signin-oidc";
        options.SaveTokens = true;
        options.TokenValidationParameters = new()
        {
          ValidateIssuerSigningKey = false,
          SignatureValidator = (token, validationParameters) => new JwtSecurityToken(token)
        };
      });

      return builder;
  }

  public static WebApplication UseIdentityServices(this WebApplication app)
  {
    app.UseAuthentication();
    app.UseAuthorization();
    return app;
  }

  public record IdentityServiceSettings(
    string Endpoint,
    string ClientId,
    string ClientSecret
  );
}
{% endhighlight %}

This is the same code, but wrapped into an extension method with some extra error handling thrown in.  One of the main pieces of error handling is the inclusion of an options pattern to get the identity service settings.  The options pattern allows you to use strongly-typed model classes instead of grabbing strings from the application configuration.  The actual format of the settings is defined in the record at the end - stored along with the identity services (so I don't have to hunt for it).  If the settings cannot be retrieved, an exception is thrown.  The section name is stored as a constant.  Again, it's in one place (stored alongside the extension methods which use it) and re-used throughout.

> Don't create separate "model" classes for one-off models.  Create a record type that is inside the same class that uses it so that you don't have to go hunting for it.
> {: .notice--success}

Now the code in `Program.cs` can become:

{% highlight csharp %}
using WebApp.Extensions;

WebApplicationBuilder builder = WebApplication.CreateBuilder(args);

builder.AddIdentityServices();
builder.Services.AddControllersWithViews();

WebApplication app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Home/Error");
    app.UseHsts();
}

app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseRouting();
app.UseIdentityServices();
app.MapDefaultControllerRoute();

app.Run();
{% endhighlight %}

I'll also add extension methods for database services and other services as necessary. Note that I also include an accompanying `app.UseIdentityServices()`.  Many service additions have a HTTP pipeline addition as well, so I add a mirror extension method even if it doesn't do anything.  This allows me to put any code I want to include later on without adjusting the `Program.cs` file.

## Using options in dependency injection

If you have to use a set of options in controllers or services where the object is constructed using dependency injection, you can use the `IOptions<>` pattern instead.  I need to know the external URI of the request so that I can re-use it when producing documents.  Since there might be an API gateway or front door in the way, I specify this in the application settings.  The app setting is read during startup and injected into the services collection.

Start with a model record:

{% highlight csharp %}
public record ServiceInformation
{
  public required Uri Endpoint { get; set; }
}
{% endhighlight %}

I love using records for models. A lot of the hard work is done for you. You just have to specify the shape of the information.  In the previous section, I used a record where each property of the record was defined in a primary constructor.  You cannot do that for the dependency injection case.  You must have a "parameterless constructor".

Next, inject the service information into the services collection (inside `Program.cs`):

{% highlight csharp %}
builder.Services.Configure<ServiceInformation>(builder.Configuration.GetSection("Service"));
{% endhighlight %}

Here, I'm expecting the following bit of JSON inside `appsettings.json`:

{% highlight json %}
{
  "Service": {
    "Endpoint": "https://localhost:7133"
  }
}
{% endhighlight %}

The actual URI is the URI of the local https endpoint inside my `launchSettings.json`.  I'll replace this when I publish this using an environment variable.  However, this allows me to "just run" the application from Visual Studio.

Finally, let's take a look at a typical controller.  The identity service I am writing requires that I create a discovery endpoint, so I have a `DiscoveryController`:

{% highlight csharp %}
Route("~/.well-known")]
public class DiscoveryController(IOptions<ServiceInformation> serviceInfo) : Controller
{
    [HttpGet("openid-configuration")]
    public JsonResult GetConfiguration()
    {
        DiscoveryResponse response = new(serviceInfo.Value.Endpoint);
        return Json(response);
    }
}
{% endhighlight %}

That `IOptions<ServiceInformation>` parameter is injected from the services collection.

> Use primary constructors where you can.  While it is seen by many as one of the less useful improvements to C#, primary constructors improve the readability of the code by removing constructors that just assign the parameters to internal fields.
> {: .notice--success}

## Final thoughts

I always do development in two passes.  Pass #1 is "make it work".  The code is yours, so you need to do whatever is necessary to make it work.  It will be messy sometimes but your first order task is to get something working.

After that, I do a second pass where my primary concern is "does the code I wrote make semantic sense when I am reading it?" I've found that I can't do this at the same time as when I writing the code.  I need a little bit of space from writing the code to doing the readability pass.  In between the two, I generally write unit tests for the code.  When I return for the readability pass, I end up using extension methods and options patterns a lot to improve the code readability.

I'm not satisfied until I can understand the code just by reading it.  Does the code describe the "what"?  Spending time in this step will pay dividends the moment someone else works on the code.

## Further reading

* [ASP.NET Core Configuration](https://learn.microsoft.com/aspnet/core/fundamentals/configuration/)
* [The Options pattern](https://learn.microsoft.com/aspnet/core/fundamentals/configuration/options)