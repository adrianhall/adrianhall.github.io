---
title: "Azure Active Directory Authentication for Blazor WASM (Part 3: Production)"
categories:
  - ASP.NET
tags:
  - Blazor
---

Recently, I've been working on integrating authentication with Azure Active Directory into my Blazor app.  I've covered [the server side]({% post_url 2022/2022-09-01-blazor-wasm-aad-auth-part-1 %}) and [the client side]({% post_url 2022/2022-09-02-blazor-wasm-aad-auth-part-2 %}), so what's left?  Well, firstly, I left "secrets" in my client app in the form of an `appsettings.json` file.  They aren't exactly secret values, but having specific values in there means I can't have different values for different environments.  Similarly, I can't change those values over time - I have to re-build the app in order to change them.  Secondly, I've got my app running in development (with secrets manager storage for my app settings), but there is nothing similar when I publish my app to the cloud, so I want to sort that out as well.

## Storing client settings in the service

Let's start by looking at fixing the `appsettings.json` problem.  The MSAL documentation (and everyone else) has you place this file in the `wwwroot`.  This makes me think that it is loaded at runtime by the browser.  Opening up the browser developer tools and checking the network tab confirms this:

![Screen shot of the browser developer tools network tab]({{ site.baseurl }}/assets/images/2022/09-03/image1.png)

Because it is loaded from my server at runtime, I can create a controller that produces that information from the configuration.  Ideally, the same controller would provide the same settings for each client that I write - event if they had different settings.

To support this, I'm going to create a `ClientConfiguration` model with the information needed, then I'll create a `ClientConfigurationManager` that handles the storage of `ClientConfiguration`, and (finally), I'll create an initial version of the `AppSettingsController` that returns the data to the client.

Let's look at the model first.  I've created this in `Models/ClientConfiguration.cs` within the server project:

{% highlight csharp %}
using System.Text.Json.Serialization;

namespace cloudmud.Server.Models
{
  public class AzureAdSettings
  {
    [JsonPropertyName("Authority")]
    public string? Authority { get; set; }
    [JsonPropertyName("ClientId")]
    public string? ClientId { get; set; }
    [JsonPropertyName("ValidateAuthority")]
    public bool ValidateAuthority { get; set; }
    [JsonPropertyName("Scope")]
    public string? Scope { get; set; }
  }

  public class ClientConfiguration
  {
    [JsonPropertyName("AzureAd")]
    public AzureAdSettings? AzureAd { get; set; }

  }
}
{% endhighlight %}

If you take a look at the `appsettings.json` file that you created in the client project, then you will see that the structure of `ClientConfiguration` is exactly the same.  The idea is that when the service sends this object encoded as JSON, it looks exactly like the `appsettings.json` file.  I've included the `JsonPropertyName` on each property because the JSON serializer lower-cases every property name normally and I need the capitalized property name.

I can actually construct most of the client configuration from the existing settings that I have set in the server-side `appsettings.json` file.  The only extra thing I need is the client application ID.  That is added as follows:

{% highlight csharp %}
{
  "AzureAd": {
    "Instance": "https://login.microsoftonline.com",
    "Domain": "{DOMAIN}",
    "TenantId": "{TENANT_ID}",
    "ClientId": "{APPLICATION_ID_FOR_SERVICE}",
    "CallbackPath": "/signin-oidc"
  },
  "ClientConfiguration": {
    "Common": {
      "ClientId": "{APPLICATION_ID_FOR_CLIENT}"
    }
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*"
}
{% endhighlight %}

As I did in [part 1]({% post_url 2022/2022-09-01-blazor-wasm-aad-auth-part-1 %}), I can use the secrets manager to store the actual client ID.  I've used "Common" as a special tag.  I'm also going to allow the user to use a query string - something like `/appsettings.json?client=Android` to get different configurations.  If no query string is present, then I'll use the "Common" set.  

The next step is to create the client configuration manager.  This is just a class that is injected (via dependency injection) into the `AppSettingsController` that returns the right data for the application settings.  First, create an interface that you will implement:

{% highlight csharp %}
using cloudmud.Server.Models;

namespace cloudmud.Server.Services
{
    public interface IClientConfigurationManager
    {
        ClientConfiguration GetClientConfiguration(string? clientType);
    }
}
{% endhighlight %}

Then create a concrete implementation:

{% highlight csharp %}
using cloudmud.Server.Models;

namespace cloudmud.Server.Services
{
    public class ClientConfigurationManager : IClientConfigurationManager
    {
        private readonly Dictionary<string, ClientConfiguration> _clients = new();

        public ClientConfigurationManager(IConfiguration configuration)
        {
            var aadSettings = configuration.GetSection("AzureAd").Get<AzureAd>();
            var clientTypes = configuration.GetSection("ClientConfiguration");
            foreach (var clientType in clientTypes.GetChildren())
            {
                _clients[clientType.Key] = new ClientConfiguration()
                {
                    AzureAd = new()
                    {
                        Authority = $"{aadSettings.Instance}/{aadSettings.TenantId}",
                        ClientId = clientType.GetValue<string>("ClientId"),
                        ValidateAuthority = true,
                        Scope = $"api://{aadSettings.ClientId}/API.Access"
                    }
                };
            }
        }

        public ClientConfiguration GetClientConfiguration(string? clientType)
        {
            if (clientType != null && _clients.ContainsKey(clientType))
            {
                return _clients[clientType];
            }
            else
            {
                return _clients["Common"];
            }
        }

        /// <summary>
        /// The form of the <c>AzureAd</c> section in the server-side appsettings.json
        /// </summary>
        private class AzureAd
        {
            public string? Instance { get; set; }
            public string? Domain { get; set; }
            public string? TenantId { get; set; }
            public string? ClientId { get; set; }
            public string? CallbackPath { get; set; }
        }
    }
}
{% endhighlight %}

The constructor builds a map of all the client configuration types that I have specified in the application settings.  In this case, I have one entry in the map once it is constructed.  The `GetClientConfiguration()` method is from the interface and just reads the map.  Finally, the class at the end is a mirror image of the configuration section.

You might wonder (given the relative complexity) why I am building a configuration manager and doing all this work when I could just put the same code inside the controller.  First, I can create the map once and use it for the life of the process.  This becomes more important if your configuration is dynamic or on the larger side.  Second, this is more testable - I can write unit tests for the manager to ensure that the right data is being returned given the right configuration.

Now that I have the client configuration manager, I need to inject it into the services of the app so I can actually use it.  This is a single line in the `Program.cs` (right below the `AddHttpContextAccessor` call):

{% highlight csharp %}
builder.Services
  .AddSingleton<IClientConfigurationManager>(new ClientConfigurationManager(builder.Configuration));
{% endhighlight %}

Finally, let's add a controller:

{% highlight csharp %}
using cloudmud.Server.Services;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;

namespace cloudmud.Server.Controllers
{
    [AllowAnonymous]
    [ApiController]
    [Route("appsettings.json")]
    public class AppSettingsController : ControllerBase
    {
        public AppSettingsController(IClientConfigurationManager manager)
        {
            Manager = manager;
        }

        public IClientConfigurationManager Manager { get; }

        [HttpGet]
        public IActionResult GetConfiguration([FromQuery] string? client)
        {
            var config = Manager.GetClientConfiguration(client);
            return Ok(config);
        }
    }
}
{% endhighlight %}

Move your client side `appsettings.json` file out of the way, and you are done? Now, before you get all excited that you are done, I'm going to warn you this won't work. You can use [Postman](https://getpostman.com) to check the output of your controller.  Make sure it matches the content of your manually created `appsettings.json` file.

Now, why didn't it work.  The configuration module within a Blazor WASM app reads the `appsettings.json` file **ONLY IF IT EXISTS IN `wwwroot` WHEN YOU BUILD THE APP**.  If you want to store it on the server, you have to load it manually.  Fortunately, this is easy to do.  Since `appsettings.json` is "special", I switch the route for the `AppSettingsController` to be `clientconfiguration.json`.  This allows me to use both capabilities.  

Add the following code in the client `Program.cs` (right above where you add the scoped HttpClient):

{% highlight csharp %}
/*
** Load the clientconfiguration.json from the service.
*/
using var http = new HttpClient() 
{ 
  BaseAddress = new Uri(builder.HostEnvironment.BaseAddress) 
};
using var response = await http.GetAsync("clientconfiguration.json");
using var stream = await response.Content.ReadAsStreamAsync();
builder.Configuration.AddJsonStream(stream);
{% endhighlight %}

If you run your app now, it should authenticate - despite no client configuration.  The configuration is read from the server as your app starts.

> You can add any other client configuration you need into this model and it will be delivered dynamically as your app starts.  You only need to restart the server to provide new configuration content to your clients.

## Storing client secrets in the cloud

Thus far, I've only run my service locally on my dev box.  When I move it to the cloud, the secrets manager goes away and I have to think about where to store the stuff that is in the secrets manager now.  There are three basic options:

1. Environment variables
2. Azure Key Vault
3. Azure App Configuration

Let's look at each one in turn.

I like to use either Azure App Service or Azure Container Apps to host my apps.  Both of them allow you to set environment variables.  Azure App Service has [app settings](https://docs.microsoft.com/azure/app-service/configure-common) and Azure Container Apps has [secrets](https://docs.microsoft.com/azure/container-apps/manage-secrets).  ASP.NET Core configuration natively [reads environment variables](https://docs.microsoft.com/aspnet/core/fundamentals/configuration/?view=aspnetcore-6.0#default-host-configuration-sources), so you don't need to do anything special.  You just need to set the following:

* `AzureAd__Domain`
* `AzureAd__TenantId`
* `AzureAd__ClientId`
* `ClientConfiguration__Common__ClientId`

At this stage, this isn't a problem.  You can even set these during deployment in your CI/CD platform of choice. Since they are tied to the service, you have no problem keeping different environments up to date. Life is good.  However, as the application configuration grows more complex, you will want to move to a service.  For one, each service in your application will need their own secrets, so now you have a coordination problem.  Secondly, you end up having to embed the secrets in the deployment scripts, which are - you guessed it - checked into source code.  That means the secrets have the potential to be leaked.

Of course, this isn't a problem with the AAD settings, but it is a problem with other settings (like database connection strings).  My app is definitely going to have some of these, so I'd rather solve it once and not have the problem later on.

Key Vault is good for common secrets that are shared among many apps.  For example, if I have a product database, then that database might be shared among multiple apps, so I will want to ensure it is the same everywhere.  I can have a Key Vault for staging and a Key Vault for production - each with the same set of keys in it.  That way the only thing I have to do is change the KeyVault name between environments and I've changed the keys.  Key Vault is also properly secure - the data at rest is encrypted and the keys are stored on a HSM in the cloud that only you can access.

But it doesn't work out of the box.  You have some work to do:

1. Create the Key Vault in Azure.
2. Create an Azure App Service with a Managed Identity.
3. Give the Managed Identity permission to use Key Vault.
4. Place the secrets in the Key Vault.
5. Update your server code to use the Key Vault.

I'm not going to cover the first three items as there are many ways to do it, most of which are [covered in the documentation](https://docs.microsoft.com/aspnet/core/security/key-vault-configuration?view=aspnetcore-6.0). 

To add the secrets into the key vault, use the following:

{% highlight powershell %}
az keyvault secret set --vault-name {KEY VAULT NAME} --name "AzureAd--Domain" --value "{value}"
{% endhighlight %}

Repeat for the other values you need to set, replacing the colon or double-underscore with double-dash.  You will also need to set an environment variable on your hosting provider for the key vault name - so you don't get away without environment variables here either.  The bonus is that you can set it in the CI/CD pipeline and it doesn't change (unlike the values of the secrets).

Now you can add the following to your servers `Program.cs` (right under the call to `CreateBuilder(args)`):

{% highlight csharp %}
var keyVaultName = builder.Configuration["KeyVaultName"];
if (!string.IsNullOrEmpty(keyVaultName)) 
{
  builder.Configuration.AddAzureKeyVault(
    new Uri($"https://{keyVaultName}.vault.azure.net/"),
    new DefaultAzureCredential()
  );
}
{% endhighlight %}

The `DefaultAzureCredential` class comes from `Azure.Identity`, and the `AddAzureKeyVault()` method comes from `Microsoft.Extensions.Configuration.AzureKeyVault` - both are available on NuGet.  

Note that I use the existance of the `KeyVaultName` in the configuration as a trigger for adding it. The `DefaultAzureCredential` will use the login to the Azure CLI, Azure PowerShell, Visual Studio, or similar to access the key vault on a development box.  This means that you can test the key vault functionality without deploying to the cloud simply by setting the key vault name in the environment.

The final mechanism - Azure App Configuration - is perhaps my favorite of the three.  Azure App Configuration provides a couple of advantages over the Azure Key Vault (but includes a couple of minuses as well).  Both Key Vault and App Configuration support the storage of key/value pairs.  Key Vault deals with certificates; App Configuration doesn't.  App Configuration deals with multiple configuration sets and feature flags; Key Vault does neither.  Key Vault is backed by a HSM; App Configuration is not.  When you are in an enterprise, the guaranteed secure storage backed by a HSM is probably important.  For a side project like mine, it isn't.  I also don't need to manage quantities of certificates.

Finally, and this is kind of arbitrary - Key Vault costs money (although admittedly, not a lot of money).  App Configuration has a free tier.  For my side projects, I prefer free.

The same steps to link an App Configuration instance to your hosting platform are required (and I still won't go through them because they are [well documented](https://docs.microsoft.com/azure/azure-app-configuration/quickstart-aspnet-core)!) and you can still use Managed Identity to provide that access.  Let's look at storing keys and accessing them in the app.

To create the key, use something like the following:

{% highlight powershell %}
az appconfig kv set --name {App Config instance} --key "AzureAd:Domain" --value "{DOMAIN}"
{% endhighlight %}

One of the cool things I like about App Configuration is that if you have some values that you have to store in Key Vault (because they really need to be stored securely or they are common to a bunch of apps) but most of the configuration is separate, then you can link the two together.  Let's say the AzureAd:ClientId was one of those secrets.  You could do this:

{% highlight powershell %}
az appconfig ky set-keyvault --name {App Config instance} --key "AzureAd:ClientId" --secret-identifier "AzureAd--ClientId"
{% endhighlight %}

That secret is now exposed via App Configuration but stored in Key Vault.  This allows you to mix and match as you see fit.  

For the code, I can use something similar to Key Vault:

{% highlight csharp %}
var appConfigurationConnectionString = builder.Configuration.GetConnectionString("AppConfiguration");
if (!string.IsNullOrEmpty(appConfigurationConnectionString))
{
  builder.Configuration.AddAzureAppConfiguration(options => 
  {
    options.ConnectionString = appConfigurationConnectionString;
    options.Credential = new DefaultAzureCredential()
  });
}
{% endhighlight %}

The code is remarkably similar, and the effect is the same - the App Configuration settings get merged into the configuration set and made available to you.

Which do I use?  When I am starting a new project, I use environment variables.  These are easy to manage, but not very scalable.  Once I reach my threshold for complexity (i.e. I am bored with setting so many environment variables when I set  up the service), I switch to App Configuration.  If I need to store something securely (which is rare, but does happen occassionally), I store just that item in Key Vault and link it into the configuration set within App Configuration.  This really is a case of "it depends"

## Next steps

You can browse [the project thus far](https://github.com/adrianhall/cloudmud/tree/191da597a88453171e854c1250acd04e1aec65f3) and see the code for yourself.  I'll be moving on to a new part of the project in the near future, so I hope you'll continue watching this space for more tips and tricks as I develop the code.
