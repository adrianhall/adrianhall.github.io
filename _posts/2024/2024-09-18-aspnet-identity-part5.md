---
title:  "ASP.NET Identity deep dive - Part 5 (Sending email)"
date:   2024-09-18
categories:
  - Web
tags:
  - aspnetcore
  - aspnet_identity
mermaid: true
---

This article is one of a number of articles I will write over the coming month and will go into depth about the [ASP.NET Identity](https://learn.microsoft.com/aspnet/core/security/authentication/identity) system.  My outline thus far:

{% include_relative includes/aspnet-identity-series.md %}

Today is the final spot for the basic flows.  Up to this point, I've been logging links in places where I should be sending emails.  Sending emails in development is scary.  First, things could go wrong and your site could potentially send non-functional links to unsuspecting people.  Second, there has been a gradual lock down of the email systems which makes sending transactional emails difficult.  You need an external service.  (Are you regretting skipping the service options for identity yet?)

So, now you need an email service.  The words to google for are "[transactional email API](https://www.google.com/search?q=transactional+email+API)", and there are several options for you.  You may have something you can already use.  If you have a pre-configured SMTP server, then use [MailKit]().  If you have Office 365 and access to the Microsoft Graph, you can use the [sendMail API](https://learn.microsoft.com/graph/api/user-sendmail?view=graph-rest-1.0&tabs=http).  Many organizations use services like [MailChimp](https://mailchimp.com/developer/transactional/api/messages/send-new-message/).  Finally, you can use a transactional API service, like [MailerSend](https://mailersend.com) or [SendGrid](https://sendgrid.com/en-us/solutions/email-api).  These latter two offer a small free tier and so are ideal for developer samples.

In this tutorial, I'm going to be using MailerSend.  You can find their API documentation [on their site](https://developers.mailersend.com/api/v1/email.html#send-an-email) - it's typical of the API you get.  It allows you to send HTML mails, which I personally don't like.  A HTML mail needs to be self-contained.  All styles and images need to be included with the document.  That makes it relatively hard to create responsive emails that look good on both mobile and on a desktop.  However, HTML mails also look more professional, so all the transactional email APIs support them.

In order to send good emails as part of a transaction, we need two things:

1. An email sender API.
2. A HTML mail template rendering engine.

## Rendering HTML mail templates

My first stop, therefore is the "HTML template engine".  Fortunately, I'm already working with one - Razor.  I can adapt it so that instead of sending the HTML to the browser, the engine renders the output to a string.  Let's start with a template.  Create a Razor layout file in `Views/Shared/_EmailLayout.cshtml`.  I found mine [at litmus.com](https://www.litmus.com/email-templates) and then adapted it.  You can find it [on my GitHub repository](https://github.com/adrianhall/samples/blob/0918/identity/Samples.Identity/Views/Shared/_EmailLayout.cshtml).  Note that it doesn't depend on any local files, including style sheets or images.  Everything is included or remotely loaded.  Even when things are remotely loaded, I've planned for the case where the user doesn't allow remote content to be loaded.

Next, let's create a couple of templates in `Views/EmailTemplates`.  My first one is `SendConfirmationLink.cshtml`:

```html
@model SendConfirmationEmailViewModel

<table width="100%" border="0" cellspacing="0" cellpadding="0">
    <tr>
        <td bgcolor="#ffffff" align="center" style="padding: 20px 30px 60px 30px;">
            <table border="0" cellspacing="0" cellpadding="0">
                <tr>
                    <td align="center" style="border-radius: 3px;" bgcolor="#539be2">
                        <a href="@Model.Url" target="_blank" style="font-size: 20px; font-family: Helvetica, Arial, sans-serif; color: #ffffff; text-decoration: none; color: #ffffff; text-decoration: none; padding: 15px 25px; border-radius: 2px; border: 1px solid #539be2; display: inline-block;">
                            @Model.Text
                        </a>
                    </td>
                </tr>
            </table>
        </td>
    </tr>
</table>
```

The `SendPasswordResetLink.cshtml` file is similar - just with a different view model.  These work exactly the same as the MVC views and view models we've been using to this point - they are just focused on email messages instead of web pages.  Similarly, the view models are relatively simple:

```csharp
public record SendConfirmationEmailViewModel
{
  public required string Url { get; set; }
  public string Text { get; set; } = "Confirm registration";
  public string EmailTitle { get; set; } = "Confirm your account";
  public string EmailContent { get; set; } = "Click the button below to confirm your account";
}
```

I've pre-defined the strings for a lot of these - it allows me to re-use templates if I want to.  Normally, I would put these strings in a resource file instead of defining them explicitly.  However, this mechanism is good enough to demonstrate the practice.

Now that I've got the templates, I need to create a service to render the templates into content I can use.  Start with the interface since I'll be injecting this as a service in my ASP.NET Core application:

```csharp
public interface IRazorViewToStringRenderer
{
    Task<string> RenderViewToStringAsync(string viewName);
    Task<string> RenderViewToStringAsync<TModel>(string viewName, TModel model);
}
```

There are two methods - one for rendering with a model (which is the normal path) and one for rendering without a model (not normally used).  Let's look at the implementation:

{% highlight csharp linenos %}
public class RazorViewToStringRenderer(
    IRazorViewEngine viewEngine,
    ITempDataProvider tempDataProvider,
    IServiceProvider serviceProvider
    ) : IRazorViewToStringRenderer
{
    public async Task<string> InternalRenderViewToStringAsync(string viewName, ViewDataDictionary viewData)
    {
        HttpContext httpContext = new DefaultHttpContext() { RequestServices = serviceProvider };
        ActionContext actionContext = new(httpContext, new RouteData(), new ActionDescriptor());
        IView view = FindView(actionContext, viewName);

        using var output = new StringWriter();
        TempDataDictionary tempData = new(actionContext.HttpContext, tempDataProvider);
        ViewContext viewContext = new(actionContext, view, viewData, tempData, output, new HtmlHelperOptions());
        await view.RenderAsync(viewContext);
        return output.ToString();
    }

    public Task<string> RenderViewToStringAsync(string viewName)
    {
        ViewDataDictionary viewData = new(new EmptyModelMetadataProvider(), new ModelStateDictionary());
        return InternalRenderViewToStringAsync(viewName, viewData);
    }

    public Task<string> RenderViewToStringAsync<TModel>(string viewName, TModel model)
    {
        ViewDataDictionary<TModel> viewData = new(new EmptyModelMetadataProvider(), new ModelStateDictionary())
        {
            Model = model
        };
        return InternalRenderViewToStringAsync(viewName, viewData);
    }

    internal IView FindView(ActionContext context, string viewName)
    {
        var getViewResult = viewEngine.GetView(null, viewName, isMainPage: true);
        if (getViewResult.Success)
        {
            return getViewResult.View;
        }

        var findViewResult = viewEngine.FindView(context, viewName, isMainPage: true);
        if (findViewResult.Success)
        {
            return findViewResult.View;
        }

        var searchedLocations = getViewResult.SearchedLocations.Concat(findViewResult.SearchedLocations);
        var errorMessage = string.Join(Environment.NewLine, [
            $"Unable to find view '{viewName}'.  The following locations were searched:",
            ..searchedLocations
        ]);
        throw new EmailTemplateNotFoundException(errorMessage) { SearchedLocations = searchedLocations.ToList() };
    }
}
{% endhighlight %}

This is perhaps the most complicated piece of code I've written for this series, but it's not hard to understand.  The two entry points set up a `ViewDataDictionary` - the version that takes the model includes the model in the dictionary.  Then both of them end up calling the internal version.  That internal method then does what is necessary to find the view you asked for and renders it into a string.

The "magic" happens in the `FindView()` method; responsible for translating the view you give the method to a view construct that the rendering engine can use. It looks in the current assembly / project for the view.  If it doesn't find it there, then it goes looking for it where you've defined all the other views.  While it doesn't work with this precise code, you can easily modify this to specify a file system location for the view.  This will allow you to place the email templates in a separate project (defined as a Razor Class Library).

I've created a custom exception `EmailTemplateNotFoundException()` so that missing templates can be handled differently from other missing content.  However, it should never be thrown.

Now I've got my rendering engine, I can add it into my services pipeline, which is defined in `Program.cs`, but I put this stuff in an extension method:

```csharp
builder.Services.AddScoped<IRazorViewToStringRenderer, RazorViewToStringRenderer>();
```

## Sending email

Next, I need to send an email.  Since this is API specific, I'm going to create a few models:

```csharp
public class EmailAddress(string emailAddress = string.Empty)
{
    [JsonPropertyName("email")]
    public string Email { get; set; } = emailAddress

    [JsonPropertyName("name")]
    public string? DisplayName { get; set; }
}

public class EmailMessage
{
    [JsonPropertyName("from")]
    public EmailAddress? FromAddress { get; set; }

    [JsonPropertyName("to")]
    public IList<EmailAddress> ToAddresses { get; set; } = [];

    [JsonPropertyName("subject")]
    public required string Subject { get; set; }

    [JsonPropertyName("text")]
    public required string TextContent { get; set; }

    [JsonPropertyName("html")]
    public string? HtmlContent { get; set; }
}
```

These two models result in the same JSON content that the API requires when sent to MailerSend.  I'm using `System.Text.Json` for serialization here.  I also have an `EmailResult` class that I use for reporting the results back.  If I need to change the JSON content for some other API, I can easily update the `JsonPropertyName` values or I can map this object into the required object.  As an example, [SendGrid supplies a C# SDK](https://www.twilio.com/docs/sendgrid/for-developers/sending-email/email-api-quickstart-for-c) which I can use.  I can easily map between these classes and the SendGrid API using a helper class.

I want to inject the email sender API into my services.  I also want to be able to mock the email sending, so I'm going to use an interface:

```csharp
public interface ISendEmailApi
{
    Task<EmailResult> SendEmailAsync(EmailMessage message, CancellationToken cancellationToken = default);
}

public static class ISendEmailApiExtensions
{
    public static Task<EmailResult> SendEmailAsync(this ISendEmailApi api, EmailAddress address, string subject, string textContent, string htmlContent, CancellationToken cancellationToken = default)
    {
        EmailMessage message = new()
        {
            ToAddresses = [address],
            Subject = subject,
            TextContent = textContent,
            HtmlContent = htmlContent
        };

        return api.SendEmailAsync(message, cancellationToken);
    }
}
```

The extension method allows me to skip the bit where I construct the `EmailMessage` object - it does it for me.  All that is left is to create the actual implementation for MailerSend:

{% highlight csharp linenos %}
public class MailerSendApi(
    IOptions<MailerSendOptions> options,
    ILogger<MailerSendApi> logger
    ) : ISendEmailApi
{
    internal const string MailerSendUri = "https://api.mailersend.com/v1/email";
    internal static MediaTypeHeaderValue jsonMediaType = MediaTypeHeaderValue.Parse( "application/json" );
    internal static JsonSerializerOptions serializerOptions = GetSerializerOptions();
    internal HttpClient client = new();

    internal string? ApiKey { get => string.IsNullOrWhiteSpace(options.Value?.ApiKey) ? null : options.Value.ApiKey; }
    internal string? FromEmail { get => string.IsNullOrWhiteSpace(options.Value?.FromEmail) ? null : options.Value.FromEmail; }
    internal string? FromName { get => string.IsNullOrWhiteSpace(options.Value?.FromName) ? null : options.Value.FromName; }

    public bool IsConfigured { get => ApiKey is not null && FromEmail is not null; }

    public async Task<EmailResult> SendEmailAsync(EmailMessage message, CancellationToken cancellationToken = default)
    {
        logger.LogTrace("SendEmailAsync: {message}", message.ToJsonString());
        if (!IsConfigured)
        {
            logger.LogError("MailerSendApi has not been configured.");
            throw new InvalidOperationException("MailerSend is not configured correctly.");
        }

        // Force the From Address, irrespective of what the email message says.
        message.FromAddress = new EmailAddress(FromEmail!) { DisplayName = FromName };

        using HttpRequestMessage request = new(HttpMethod.Post, new Uri(MailerSendUri));
        request.Content = JsonContent.Create(message, jsonMediaType, serializerOptions);
        request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", ApiKey);

        using HttpResponseMessage response = await client.SendAsync(request, cancellationToken);
        EmailResult result = new()
        { 
            Succeeded = response.IsSuccessStatusCode,
            ResultCode = (int)response.StatusCode,
            Messages = [
                await response.Content.ReadAsStringAsync(cancellationToken)
            ]
        };

        if (response.IsSuccessStatusCode)
        {
            logger.LogInformation("Email successfully submitted: {result}", result.ToJsonString());
        }
        else
        {
            string messageId = response.Headers.GetValues("x-message-id").FirstOrDefault() ?? "not-returned";
            logger.LogError("Email submission failed. {statusCode} {reasonPhrase} x-message-id={messageId}", 
                response.StatusCode, response.ReasonPhrase, messageId);
        }

        return result;
    }

    internal static JsonSerializerOptions GetSerializerOptions()
    {
        JsonSerializerOptions options = new(JsonSerializerDefaults.Web)
        {
            AllowTrailingCommas = false,
            DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull,
            DictionaryKeyPolicy = JsonNamingPolicy.CamelCase,
            NumberHandling = JsonNumberHandling.Strict,
            PropertyNamingPolicy = JsonNamingPolicy.CamelCase
        };

        return options;
    }
}
{% endhighlight %}

Let's go through it:

* Line 2 defines a set of options for this API that I need to pass in.  I will need to add these to the configuration.
* Lines 6-8 define the HTTP request parameters that I'm going to use.  MailerSend has a specific URI and passes data as JSON content.
* Line 9 creates a new HTTP client for the service.  This is generally a bad idea - you want to use a singleton or a `IHttpClientFactory`, so this is something I'll refactor later on.
* Lines 11-13 are the properties pulled from configuration.  I make sure they are null if not defined or empty.
* Line 17 starts the actual entry-point.  I create the `HttpRequestMessage` according to the API specification (lines 29-31), send it to the remote service (line 33), and process the result (lines 34-52).
* The `GetSerializerOptions()` method generates an appropriate set of `JsonSerializerOptions` to ensure that the `EmailMessage` object is translated according to the requirements of the MailerSend API.

I can now inject this service into the services provider:

```csharp
builder.Services.Configure<MailerSendOptions>(builder.Configuration.GetSection("Services:MailerSend"));
builder.Services.AddScoped<ISendEmailApi, MailerSendApi>();
```

I also need to make sure I have added the following to my configuration:

```json
"Service": {
  "MailerSend": {
    "ApiKey": "<API Key for the MailerSend Service>",
    "FromEmail": "<MailerSend From: email address - must be registered with MailerSend>",
    "FromName": "<The name of the account associated with FromEmail>"
  }
}
```

I add these to user-secrets in development.  In production, I use Azure App Configuration and Azure Key Vault.  The API Key is stored in Key Vault (with a reference in App Configuration) and the FromEmail and FromName are stored in App Configuration.

## Joining the bits together

Finally, I need to implement an `IEmailSender<TUser>` that uses the `ISendEmailApi` interface.  I started with a copy of the `LoggingEmailSender` class, then I expanded it to add the code to use the `ISendEmailApi` and `IRazorViewToStringRenderer`:

{% highlight csharp linenos %}
public class MailerSendEmailSender(
    ISendEmailApi sendEmailApi,
    IRazorViewToStringRenderer templateRenderer,
    ILogger<MailerSendEmailSender> logger
    ) : IEmailSender<ApplicationUser>
{
    public async Task SendConfirmationLinkAsync(ApplicationUser user, string email, string confirmationLink)
    {
        logger.LogInformation("SendConfirmationLink: email={email},link={confirmationLink}", email, confirmationLink);

        SendConfirmationEmailViewModel viewModel = new() { Url = confirmationLink };
        string htmlContent = await templateRenderer.RenderViewToStringAsync("/Views/EmailTemplates/SendConfirmationLink.cshtml", viewModel);
        string textContent = $"Your confirmation link is {confirmationLink} - click or copy into your browser.";
        string subject = "Your confirmation link from Samples.Identity";

        EmailAddress address = new(email) { DisplayName = string.IsNullOrWhiteSpace(user.DisplayName) ? null : user.DisplayName };
        EmailResult result = await sendEmailApi.SendEmailAsync(address, subject, textContent, htmlContent);
        result.LogResult(logger);
    }

    public Task SendPasswordResetCodeAsync(ApplicationUser user, string email, string resetCode)
    {
        logger.LogInformation("SendPasswordResetCode: email={email},code={resetCode}", email, resetCode);
        return Task.CompletedTask;
    }

    public async Task SendPasswordResetLinkAsync(ApplicationUser user, string email, string resetLink)
    {
        logger.LogInformation("SendPasswordResetLinkAsync: email={email},link={confirmationLink}", email, resetLink);

        SendPasswordResetEmailViewModel viewModel = new() { Url = resetLink };
        string htmlContent = await templateRenderer.RenderViewToStringAsync("/Views/EmailTemplates/SendPasswordResetLink.cshtml", viewModel);
        string textContent = $"Your password reset link is {resetLink} - click or copy into your browser.";
        string subject = "Your password reset link from Samples.Identity";

        EmailAddress address = new(email) { DisplayName = string.IsNullOrWhiteSpace(user.DisplayName) ? null : user.DisplayName };
        EmailResult result = await sendEmailApi.SendEmailAsync(address, subject, textContent, htmlContent);
        result.LogResult(logger);
    }
}
{% endhighlight %}

Each method is similar.  I create a view model, then use `RenderViewToStringAsync()` to render the correct template to a string using the view model.  Then I call the extension method `sendEmailApi.SendEmailAsync()` to send the email and log the results.  As a better implementation, I should check the result and throw an error - this allows my registration or password reset code to back out the change to the database, and display an appropriate message to the user instead of "check your email".

## Final thoughts

There is a lot of reusable code here for transactional emails, so I hope its useful outside of ASP.NET Identity as well.  However, you are using an external service - whether it's your own SMTP server, Microsoft Graph, or a transactional email service like MailerSend or SendGrid.  In any case, you should now have a fully functional identity system with registration and password reset via email.

## Further reading

* [The project so far](https://github.com/adrianhall/samples/tree/0918/identity)
* [MailerSend API Documentation](https://developers.mailersend.com/)
* [SendGrid API Documentation](https://www.twilio.com/docs/sendgrid/api-reference/how-to-use-the-sendgrid-v3-api/authentication)
* [MailKit API Documentation](https://mimekit.net/docs/html/Introduction.htm)
* [Microsoft Graph Documentation](https://learn.microsoft.com/en-us/graph/use-the-api)
* [RazorViewToStringAsync](https://learn.microsoft.com/aspnet/core/blazor/components/render-components-outside-of-aspnetcore)
