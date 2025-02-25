---
title:  "ASP.NET Identity deep dive - Part 2 (Registration)"
date:   2024-09-13
categories:
  - Web
tags:
  - aspnetcore
  - aspnet_identity
mermaid: true
---

This article is one of a number of articles I will write over the coming month and will go into depth about the [ASP.NET Identity](https://learn.microsoft.com/aspnet/core/security/authentication/identity) system.  My outline thus far:

{% include_relative includes/aspnet-identity-series.md %}

As you may remember from the [last article]({% post_url 2024/2024-09-11-aspnet-identity-part1 %}), the first user journey I am going to implement is the registration journey.  This is actually one of the more complex journeys with several parts to it.

<pre class="mermaid">
flowchart TD
  id1([User selects Register])
  id2[User Registration Form displayed]
  id3(User submits Registration Form)
  id4[User already exists]
  id5([User must be confirmed via email])
  id6(User clicks on email link)
  id7[User is confirmed]

  id1 --> id2
  id2 --> id3
  id3 --> id4
  id3 --> id5
  id5 --> id6
  id6 --> id7
</pre>

## Triggering registration

Let's get over some of the pre-requisites first.  When a user goes to my home page, I need to be able to trigger a registration event.  I started by gutting the `Views/Shared/_Layout.cshtml` file and establishing [my own Bootstrap configuration]({% post_url 2024/2024-08-08-bootstrap-in-aspnetcore %}).  Part of that process was to create a navigation partial, which then includes the `Views/Shared/_LoginPartial.cshtml` file.  This uses ASP.NET Identity to decide what to display:

{% highlight html %}
@using Samples.Identity.Data
@using Microsoft.AspNetCore.Identity

@inject SignInManager<ApplicationUser> SignInManager
@inject UserManager<ApplicationUser> UserManager

@{
    bool isSignedIn = SignInManager.IsSignedIn(User);
    ApplicationUser? userRecord = await UserManager.GetUserAsync(User);
}

<ul class="navbar-nav ms-auto me-4 my-3 my-lg-0">
  @if (isSignedIn)
  {
    <a class="btn btn-primary rounded-pill px-3 mb-2 mb-lg-0" asp-controller="Account" asp-action="Logout">
      <span class="d-flex align-items-center">
        <i class="bi-person-circle me-2"></i>
        <span class="small">@userRecord?.DisplayName</span>
      </span>
    </a>
  }
  else
  {
    <a class="btn btn-primary px-3 mb-2 mx-2 mb-lg-0" asp-controller="Account" asp-action="Login">
      <span class="d-flex align-items-center">
        <i class="bi-box-arrow-in-right me-2"></i>
        <span class="small">Sign in</span>
      </span>
    </a>
    <a class="btn btn-warning px-3 mb-2 mb-lg-0" asp-controller="Account" asp-action="Register">
      <span class="d-flex align-items-center">
        <i class="bi-person-add me-2"></i>
        <span class="small">Register</span>
      </span>
    </a>
  }
</ul>
{% endhighlight %}

This is simple enough to follow.  I use the `SignInManager` (a part of ASP.NET Identity) to determine if the user is signed in or not.  I also retrieve the user record for the logged in user.  This will be null if the user is not logged in.  For the display part, I'm going to display the users display name (which I will capture during registration) and allow the user to sign out.  If the user is not signed in, then I'll display a sign in and a register button.

All three links go to actions within an `AccountController` - something I have not written yet.  MVC (model-view-controller) requires three parts - a model, a view, and a controller.  The `AccountController` is the controller part of this.

For the rest of this project, I'm going to be working on a number of files:

* `Controllers/AccountController.cs` is a C# class for handling the business logic for account operations.  We'll be adding a lot to this.
* `Models/Account/*.cs` is a set of model classes for passing data to and from the views.
* `Views/Account/*.cshtml` are a set of views, written in Razor syntax, for displaying the output.

## Display the registration form

Let's start with the most basic sequence.  When a user pressed the register button, the `AccountController.Register()` method is called.  Let's take a look at the full `AccountController` at this point:

{% highlight csharp %}
using Microsoft.AspNetCore.Identity;
using Microsoft.AspNetCore.Mvc;
using Samples.Identity.Data;
using Samples.Identity.Models.Account;

namespace Samples.Identity.Controllers;

[AutoValidateAntiforgeryToken]
public class AccountController(
    UserManager<ApplicationUser> userManager,
    SignInManager<ApplicationUser> signInManager,
    ILogger<AccountController> logger
    ) : Controller
{
    #region Register
    /// <summary>
    /// Displays the registration form
    /// </summary>
    [HttpGet]
    public IActionResult Register() => View(new RegisterViewModel());
    #endregion
}
{% endhighlight %}

The `UserManager<TUser>` and `SignInManager<TUser>` are from ASP.NET Identity. Finally, we have a logger since I do a lot of logging within the identity system.  For the actual page, I just display the view with a blank register view model.  I am using the standard anti-forgery token validation that is recommended for non-API scenarios across the entire controller.  You can read about this in [the ASP.NET Core documentation](https://learn.microsoft.com/aspnet/core/security/anti-request-forgery).  It's a critical part of the security story for identity.

I've got a specific way of dealing with view models.  I create an input model and a view model inside the same file, where the view model derives from the input model.  The register view model will show you what I mean:

{% highlight csharp %}
namespace Samples.Identity.ViewModels.Account;

public record RegisterInputModel
{
  public RegisterInputModel()
  {
  }

  public RegisterInputModel(RegisterInputModel model)
  {
    Email = model.Email;
    Password = model.Password;
    ConfirmPassword = model.ConfirmPassword;
    DisplayName = model.DisplayName;
  }

  [Required, EmailAddress]
  public string? Email { get; set; }

  [Required, MinLength(1), MaxLength(100)]
  public string? DisplayName { get; set; }

  [Required, DataType(DataType.Password)]
  public string? Password { get; set; }

  [Required, DataType(DataType.Password)]
  [Compare(nameof(Password), ErrorMessage = "Password and confirmation must match")]
  public string? ConfirmPassword { get; set; }
}

public record RegisterViewModel : RegisterInputModel
{
  public RegisterViewModel() : base()
  {
  }

  public RegisterViewModel(RegisterInputModel inputModel) : base(inputModel)
  {
  }
}
{% endhighlight %}

Why do I do it this way?  It's a standard I have adopted to split the classes between an input model (which is just the properties I need when submitting a form) and an output (or view) model (which is the properties I need to render the form).  By providing a constructor that takes the input model, I can easily clone the data for the next display when there is an error.

Now, let's take a look at the view:

{% highlight html %}
@using Samples.Identity.Models.Account
@model RegisterViewModel
@{
    ViewBag.BodyClass = "layout--account";
    ViewBag.Title = "Create account";
    Layout = "_AccountLayout";
}

@section StyleSheets {
    <link rel="stylesheet" href="https://unpkg.com/bs-brain@2.0.4/components/registrations/registration-6/assets/css/registration-6.css"/>
}

<div class="row">
    <div class="col-12">
        <div class="mb-5">
            <h2 class="h3">Registration</h2>
            <h3 class="fs-6 fw-normal text-secondary m-0">Enter your details to register</h3>
        </div>
    </div>
</div>
<form method="post">
    <div asp-validation-summary="ModelOnly" class="text-danger"></div>
    <div class="row gy-3 overflow-hidden">
        <div class="col-12">
            <div class="form-floating mb-3">
                <input asp-for="DisplayName" class="form-control" placeholder="Display Name" required>
                <label asp-for="DisplayName" class="form-label">Display Name</label>
                <span asp-validation-for="DisplayName" class="text-danger"></span>
            </div>
        </div>

        <div class="col-12">
            <div class="form-floating mb-3">
                <input asp-for="Email" class="form-control" placeholder="name@example.com" required>
                <label asp-for="Email" class="form-label">Email</label>
                <span asp-validation-for="Email" class="text-danger"></span>
            </div>
        </div>
        <div class="col-12">
            <div class="form-floating mb-3">
                <input asp-for="Password" class="form-control" placeholder="Password" required>
                <label asp-for="Password" class="form-label">Password</label>
                <span asp-validation-for="Password" class="text-danger"></span>
            </div>
        </div>
        <div class="col-12">
            <div class="form-floating mb-3">
                <input asp-for="ConfirmPassword" class="form-control" placeholder="Confirm Password" required>
                <label asp-for="ConfirmPassword" class="form-label">Password</label>
                <span asp-validation-for="ConfirmPassword" class="text-danger"></span>
            </div>
        </div>
        <div class="col-12">
            <div class="d-grid">
                <button class="btn bsb-btn-2xl btn-primary" type="submit">Sign up</button>
            </div>
        </div>
    </div>
</form>
<div class="row">
    <div class="col-12">
        <hr class="mt-5 mb-4 border-secondary-subtle">
        <p class="m-0 text-secondary text-center">
            Already have an account?
            <a asp-controller="Acocunt" asp-action="Login" class="link-primary text-decoration-none">Sign in</a>
        </p>
    </div>
</div>
{% endhighlight %}

This is a very basic Razor form. I've got a unique layout for the account section, but the rest of this is standard MVC view stuff.  You can (and should) do more here.  Some of the things I look at are password strength meters and inline validation capabilities to ensure that as much is done by the browser as possible before sending the form to the backend for processing.

You should be able to run the project at this point, click on the Register button and see your form.  Now you can play with your view and layout as much as is needed to get it to display the way you want.  Here are some sites that I came across while developing this:

* [Material UI Kit](https://mdbootstrap.com/docs/standard/extended/login/)
* [Bootstrap example login form](https://getbootstrap.com/docs/5.3/examples/sign-in/)
* [Bootstrap brain registration form](https://bootstrapbrain.com/component/bootstrap-registration-form-code/)

Obviously, I used one of the registration form examples from this last site.  I found their code to be great to follow.

## Handling the Register form

Obviously, since we now have a registration form, we need to handle it.  I'm referring to my flow chart above when I write this.

{% highlight csharp linenos %}
/// <summary>
/// Handles the result from the POST of the registration form.
/// </summary>
[HttpPost]
public async Task<IActionResult> Register([FromForm] RegisterInputModel input)
{
    logger.LogTrace("Register Form Submission: {input}", JsonSerializer.Serialize(input));
    if (!ModelState.IsValid || input.Email is null || input.Password is null || input.DisplayName is null)
    {
        logger.LogDebug("Register Form Submission: Form was invalid");
        return View(new RegisterViewModel(input));
    }

    // TODO: If you need to validate DisplayName - e.g. to check it isn't a profane username
    //  - then do it at this point

    ApplicationUser? existingUser = await userManager.FindByEmailAsync(input.Email);
    if (existingUser is not null)
    {
        logger.LogDebug("Register Form Submission: User {email} already exists as {id}", input.Email, existingUser.Id);
        ModelState.AddModelError(string.Empty, "User already registered.");
        return View(new RegisterViewModel());
    }

    ApplicationUser newUser = new()
    {
        Id = Guid.NewGuid().ToString("N"),
        UserName = input.Email.ToLowerInvariant(),
        Email = input.Email.ToLowerInvariant(),
        DisplayName = input.DisplayName
    };

    logger.LogTrace("Register: Creating new user {newUser}", JsonSerializer.Serialize(newUser));
    IdentityResult result = await userManager.CreateAsync(newUser, input.Password);
    if (!result.Succeeded)
    {
        logger.LogError("Register: Could not create {id}: {errors}", newUser.Id, JsonSerializer.Serialize(result.Errors.Select(e => e.Description)));
        foreach (IdentityError error in result.Errors)
        {
            ModelState.AddModelError(string.Empty, error.Description);
        }
        return View(new RegisterViewModel(input));
    }

    // If an email confirmation is NOT required, then sign the user in and redirect to the home page
    if (!userManager.Options.SignIn.RequireConfirmedAccount)
    {
        logger.LogDebug("Register: RequireConfirmedAccount = false; sign-in {email} automatically", newUser.Email);
        await signInManager.SignInAsync(newUser, isPersistent: false);
        return RedirectToHomePage();
    }

    // email confirmation IS required; send the email and display the email confirmation required page
    await SendConfirmationLinkAsync(newUser);
    return RedirectToAction(nameof(ResendEmailConfirmation), new { newUser.Email });
}
{% endhighlight %}

I've made sure to comment this throughout so you can follow along.  However, there are a few other controller actions (which we will get onto in a bit) and some 
helper methods that are needed.  Explicitly, there is a method of sending a confirmation link to a user, which is not yet implemented.

This is a great point at which to take a look at some of the APIs that are provided by ASP.NET Identity.  You have three basic objects that are injected by dependency injection:

* `UserManager<TUser>` handles user records.
* `RoleManager<TUser>` handles roles.
* `SignInManager<TUser>` handles sign in and sign out actions.

Within `UserManager<TUser>`, there are a number of methods for finding users - by email, ID, username, etc.  For example, I'm using the email address as my primary login method, so I can use the following method:

{% highlight csharp %}
ApplicationUser? user = userManager.FindByEmailAsync(emailAddress);
{% endhighlight %}

The user manager also has the normal CRUD methods for creating, updating, and deleting users.

The `SignInManager<TUser>` deals with the actual user claims.  The logged in user is stored in `HttpContext.User` as a `ClaimsPrincipal`.  This is generally not something you want to be using when you are developing with ASP.NET Identity.  You want to be using `ApplicationUser` (or whatever your user model is).  To get the user model from the claims principal, use `userManager.GetUserAsync(HttpContext.User)`.  To determine if the user is logged in, use `signInManager.IsSignedIn(HttpContext.User)`.  We'll get more into the sign in manager in the next article when we cover authentication.

We'll cover the role manager in greater detail when we talk about roles and authorization later on in the series.

## Sending the confirmation link

ASP.NET Identity provides a nice interface - `IEmailSender{TUser}` - that you can use with dependency injection.  It gives you three methods that can be used for sending templated emails to users.  I'm going to be covering sending email later on.  For right now, I just want to continue with development.  I've created a simple implementation of the interface that just logs the request.  It looks like this:

{% highlight csharp %}
public class LoggingEmailSender(ILogger<LoggingEmailSender> logger) : IEmailSender<ApplicationUser>
{
    /// <inheritdoc />
    public Task SendConfirmationLinkAsync(ApplicationUser user, string email, string confirmationLink)
    {
        logger.LogInformation("SendConfirmationLink: email={email},link={confirmationLink}", email, confirmationLink);
        return Task.CompletedTask;
    }

    /// <inheritdoc />
    public Task SendPasswordResetCodeAsync(ApplicationUser user, string email, string resetCode)
    {
        logger.LogInformation("SendPasswordResetCode: email={email},code={resetCode}", email, resetCode);
        return Task.CompletedTask;
    }

    /// <inheritdoc />
    public Task SendPasswordResetLinkAsync(ApplicationUser user, string email, string resetLink)
    {
        logger.LogInformation("SendPasswordResetLink: email={email},link={resetLink}", email, resetLink);
        return Task.CompletedTask;
    }
}
{% endhighlight %}

I've added it to my startup:

{% highlight csharp %}
builder.Services.AddScoped<IEmailSender<ApplicationUser>, LoggingEmailSender>();
{% endhighlight %}

And I've added it to the constructor of the `AccountController`.  Now I can implement the `SendConfirmationLinkAsync()` method in the `AccountController`:

{% highlight csharp %}
private async Task SendConfirmationLinkAsync(ApplicationUser user)
{
    logger.LogTrace("SendConfirmationLink: {json}", JsonSerializer.Serialize(user));
    string userId = await userManager.GetUserIdAsync(user);
    string confirmationToken = await userManager.GenerateEmailConfirmationTokenAsync(user);
    string? callbackUrl = Url.ActionLink(
        action: nameof(ConfirmEmail),
        values: new { userId, token = EncodeToken(confirmationToken) },
        protocol: Request.Scheme);
    await emailSender.SendConfirmationLinkAsync(user, user.Email!, callbackUrl!);
}
{% endhighlight %}

I also need to take the token that is generated via the `userManager` and encode it so that the user can click on a link in their email.  I'll need a corresponding
decode method later on when I write the `ConfirmEmail` handler.

{% highlight csharp %}
internal string DecodeToken(string token)
    => Encoding.UTF8.GetString(WebEncoders.Base64UrlDecode(token));

internal string EncodeToken(string token)
    => WebEncoders.Base64UrlEncode(Encoding.UTF8.GetBytes(token));
{% endhighlight %}

## Supporting views

There are two supporting views that I need:

* `ConfirmEmail` is called when the user clicks on the link in their email.
* `ResendEmailConfirmation` will be called if the email confirmation fails for some reason.

As you add the code that links to these views, make sure you don't accidentally bring in the Blazor Identity version of the UI.  Visual Studio is only too happy to
direct you to these packages if you aren't careful.

Here are the actions that support these views:

{% highlight csharp linenos %}
[HttpGet]
public async Task<IActionResult> ConfirmEmail([FromQuery] string? userId, [FromQuery] string? token)
{
    if (string.IsNullOrEmpty(userId) || string.IsNullOrEmpty(token))
    {
        return RedirectToHomePage();
    }

    ApplicationUser? user = await userManager.FindByIdAsync(userId);
    if (user is null)
    {
        return NotFound($"No such user {userId}.");
    }

    IdentityResult result = await userManager.ConfirmEmailAsync(user, DecodeToken(token));
    if (result.Succeeded)
    {
        return View(new EmailConfirmationViewModel() { Email = user.Email! });
    }

    return RedirectToAction(nameof(ResendEmailConfirmation), new { user.Email });
}

[HttpGet]
public IActionResult ResendEmailConfirmation([FromQuery] string? email)
{
    if (string.IsNullOrEmpty(email))
    {
        return RedirectToAction(nameof(Register));
    }

    return View(new EmailConfirmationViewModel() { Email = email });
}
{% endhighlight %}

I'll also need a form processor for the `ResendEmailConfirmation` form so that it triggers the `SendEmailConfirmationAsync()` method again.  I'll leave the implementation of that up to you, but you can check out my version of the actions [on my GitHub project][github].

The `EmailConfirmationViewModel` (and associated `EmailConfirmationInputModel`) are similar to the `RegisterViewModel` and `RegisterInputModel`, but only have one property - the email address of the user.

## The supporting views

I created a number of supporting views for the registration process as well.  I'll leave these to you since they are basically HTML and CSS.  There wasn't any real functionality - just status messages. Again, check out the versions [on my GitHub project][github] to see what I did.

Once this is done, you should be able to run your project and go through the registration process.  A few notes about that:

* The "send confirmation email" shows up in your logs, so make sure you are watching the logs for the link.  You'll need to copy and paste the link into your browser to confirm the email.
* Check out the resulting database updates as you go through the process.  You'll see the changes that are made at each stage.  I've included PgWeb in my Aspire AppHost for this purpose.
* Make sure you validate all the error paths as well.  What happens if you try to re-register an existing account, or if you put in the wrong code during confirmation?

MVC does lend itself to better unit testing, so you can test the controller in isolation to ensure each error condition is handled appropriately. When testing
the controller, I can mock each of the components that are introduced with dependency injection - allowing me to simulate failure conditions that includes database
failures.  Check out [the docs on ASP.NET Core controller testing](https://learn.microsoft.com/aspnet/core/mvc/controllers/testing) for more information.

## Final thoughts

The registration process is a lot of code.  Since I'm not a (good) frontend developer, I find the hardest part is writing the HTML and CSS that drives the UI. The actual logic feels straight forward to me.  ASP.NET Identity also provides a lot of assistance through its API surface.

I've also added a couple of "nice-to-haves":

* It's possible to spam the resend email confirmation button.  I've added a very basic rate limiting version so that you can't just click the button.  Yes, someone who is really evil could bypass the form and see what I'm submitting, so it's not in any way fool-proof.  It's designed to get the "rage clickers" to wait.
* I've also put a pretty basic profanity detector on the DisplayName property during registration.  Again, it's not fool proof (but, then again, no profanity filter is foolproof).  However, it will pause the most egregious situations.

With all this being said, are you sure your logic is secure?  This is a lot of code that is very tied up with the security of your application that doesn't add real value to the experience. There is still time to consider NOT doing an identity service!

## Further reading

* [Mozilla Developer Network](https://developer.mozilla.org/) - essential reading for frontend devs.
* [ASP.NET MVC Forms tag helpers](https://learn.microsoft.com/aspnet/core/mvc/views/working-with-forms).
* [ASP.NET Identity Overview](https://learn.microsoft.com/aspnet/core/security/authentication/identity)

<!-- Links -->
[github]: https://github.com/adrianhall/samples/tree/0913/identity
