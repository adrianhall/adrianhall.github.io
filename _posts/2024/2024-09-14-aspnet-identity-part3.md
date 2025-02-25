---
title:  "ASP.NET Identity deep dive - Part 3 (Authentication)"
date:   2024-09-14
categories:
    - "ASP.NET Core"
tags:
    - "ASP.NET Core"
    - "ASP.NET Identity"
mermaid: true
---

This article is one of a number of articles I will write over the coming month and will go into depth about the [ASP.NET Identity](https://learn.microsoft.com/aspnet/core/security/authentication/identity) system.  My outline thus far:

{% include_relative includes/aspnet-identity-series.md %}

Today, I'm tackling authentication - that is, signing in and out of an account using the web UI.  You may remember from [my last article]({% post_url 2024/2024-09-13-aspnet-identity-part2 %}) that I set up the initial web UI to include a link to `/Account/Login` in my `_LoginPartial.cshtml` partial view to act as a trigger for the login process.  Compared to the registration process, the login flow is much easier to understand.

Let's expand this a little bit.  When you press the Login button on most UIs, you go through the login process and then you expect to be returned to the same place as before.  In order to do this, you need to know where you came from and that fact needs to be transmitted across the entire login flow.  That means it gets stored in the input and view models and it needs to be passed into the login trigger.  To do this, we alter the login button:

{% highlight html %}
<a class="btn btn-primary px-3 mb-2 mx-2 mb-lg-0" asp-controller="Account" asp-action="Login" asp-route-returnUrl="@Context.Request.GetEncodedUrl()">
    <span class="d-flex align-items-center">
        <i class="bi-box-arrow-in-right me-2"></i>
        <span class="small">Sign in</span>
    </span>
</a>
{% endhighlight %}

I had to add `@using Microsoft.AspNetCore.Http.Extensions` to my view.  You can do this at the top of the view or within the `_ViewImports.cshtml` file.  Now that I've got the correct trigger, lets look at the MVC stuff, starting with the models.

## The login models

As is normal, I've got an input model and a view model:

{% highlight csharp %}
public record LoginInputModel
{
    public LoginInputModel()
    {
    }

    public LoginInputModel(LoginInputModel model)
    {
        Email = model.Email;
        Password = model.Password;
        RememberMe = model.RememberMe;
        ReturnUrl = model.ReturnUrl;
    }

    [Required, EmailAddress]
    public string? Email { get; set; }

    [Required, DataType(DataType.Password)]
    public string? Password { get; set; }

    [Display(Name = "Remember me?")]
    public bool RememberMe { get; set; } = true;

    public string? ReturnUrl { get; set; }
}

public record LoginViewModel : LoginInputModel
{
    public LoginViewModel() : base()
    {
    }

    public LoginViewModel(LoginInputModel model) : base(model)
    {
    }
}
{% endhighlight %}

As I mentioned, I'm passing the `ReturnUrl` across the entire process, so it has to be a part of the input model.  There isn't anything else in the login view model right now, but we'll be changing that later on when we start working on external (social) login providers, so it's a good idea to keep the separation.

## The login view

As before, I'm not going to go through the view.  It's a regular ASP.NET Razor View form.  You can see it [on the GitHub repository](https://github.com/adrianhall/samples/blob/0914/identity/Samples.Identity/Views/Account/Login.cshtml).  The only thing to be aware of (that is non-obvious) is that you should pass the ReturnUrl to the form as a hidden input field.

## The login actions

Now let's take a look at the login actions on the `AccountController`:

{% highlight csharp linenos hl_lines="33" %}
/// <summary>
/// Determines if the user should be locked out according to the rules of lockouts.
/// </summary>
internal bool LockoutOnFailure { get => !environment.IsDevelopment(); }

[HttpGet]
public IActionResult Login([FromQuery] string? returnUrl)
{
    returnUrl ??= Url.Content("~/");
    return View(new LoginViewModel() { ReturnUrl = returnUrl });
}

[HttpPost]
public async Task<IActionResult> Login([FromForm] LoginInputModel model)
{
    logger.LogTrace("Login: form = {json}", JsonSerializer.Serialize(model));
    model.ReturnUrl ??= Url.Content("~/");
    if (!ModelState.IsValid || model.Email is null || model.Password is null)
    {
        logger.LogDebug("Login: form is invalid");
        ModelState.AddModelError(string.Empty, "Form is invalid");
        return View(new LoginViewModel(model));
    }

    ApplicationUser? user = await userManager.FindByEmailAsync(model.Email);
    if (user is null)
    {
        logger.LogDebug("Login: email address {email} not found.", model.Email);
        ModelState.AddModelError(string.Empty, "Unknown user/password combination.");
        return View(new LoginViewModel(model));
    }

    var result = await signInManager.PasswordSignInAsync(user, model.Password, model.RememberMe, lockoutOnFailure: LockoutOnFailure);
    if (result.Succeeded)
    {
        logger.LogDebug("Login: password sign in for {email} succeeded.", model.Email);
        return LocalRedirect(model.ReturnUrl);
    }

    if (result.IsLockedOut)
    {
        logger.LogDebug("Login: email address {email} is locked out", model.Email);
        ModelState.AddModelError(string.Empty, "Locked out.");
        return View(new LoginViewModel(model));
    }

    logger.LogDebug("Login: password sign in for {email} failed.", model.Email);
    ModelState.AddModelError(string.Empty, "Unknown user/password combination.");
    return View(new LoginViewModel(model));
}
{% endhighlight %}

To start with, I have included a property that determines if the system is handling lockouts.  In my case, I don't want to be locked out if I'm running in development mode.  I'll update this at some point so that this sort of thing is driven by app settings and the `IConfiguration` system.

Note how I pass in the return URL all over the place and ensure it is properly propagated throughout the login system.

The main line to concentrate on is line 33.  This is where the sign-in actually happens and it is handled within ASP.NET Identity. I know that, underneath, database lookups happen and the password is hashed so that it is stored securely. However, all that is abstracted away to be consumed by a simple API. Note that the ability to remember the username and password (via a "Remember Me" checkbox) and account lockout (disable the account after a number of wrong logins) are all taken care of within this API without you having to code out the logic.

One big security question you should answer for yourself is what the error messages should be for various failure conditions.  In this version, I log the true error to the system logs.  However, I treat a wrong username and a wrong password as the same error to the user. While this doesn't prevent account harvesting attacks, it makes them much harder.  I do display an "Account locked out" message when the user is locked out.  This allows the user to do something different if they are locked out other than just try more passwords.  It also indicates to an attacker that its a waste of time spending more resources on hacking this account as its not going to work anyway.

## Signing out

Signing out is much simpler than the login process.  Here is the controller action:

{% highlight csharp %}
[HttpGet]
public async Task<IActionResult> Logout([FromQuery] string? returnUrl)
{
    returnUrl ??= Url.Content("~/");
    if (!signInManager.IsSignedIn(HttpContext.User))
    {
        logger.LogTrace("Logout: User is not signed in - redirecting");
        return LocalRedirect(returnUrl);
    }

    ApplicationUser? user = await userManager.GetUserAsync(HttpContext.User);
    if (user is not null)
    {
        logger.LogTrace("Logout: Logging out {email}", user.Email);
    }
    else
    {
        logger.LogTrace("Logout: Cannot determine user record - something is very wrong");
    }

    await signInManager.SignOutAsync();
    return LocalRedirect(returnUrl);
}
{% endhighlight %}

As with the login process, I pass in an optional return URL.  If the return URL is not specified, I redirect to the home page.  If the user is not signed in, then I just redirect wherever they wanted.  If they are logged in, I log the user record and then sign them out.  It's really straight forward code!

Now that you have sign-in and sign-out covered, you should be able to run the project and...

* Register an account, with confirmation via log message.
* Sign in with that account after registration.
* See your username, click on it, and be signed out.

You can (and should) also check out [PgWeb](https://sosedoff.github.io/pgweb/) (which is provided in the Aspire AppHost) and check the changes that are made to the database as you cycle through the various user flows.  Now is also a great time to add a user to the admin role and write some management pages that allow you to view the users within the web UI.

## Final thoughts

This code path does not go into any of the complexities of modern authentication systems, including:

* Multi-factor authentication.
* Passkeys and other passwordless technologies.
* Magic links (as a passwordless alternative)
* Social logins

I'm going to get to those in the future. I'd use this level of authentication for a game or smaller site, but I'd want the complexity in a more mainstream site.  If that sounds too complex for you, I'm taking this opportunity to get you to re-think your life choices and decide on an alternative identity solution.  The ones I've used are:

* [Keycloak](https://www.keycloak.org/) is a good option if you have to store your own data, 
* [Auth0](https://auth0.com/) is a good option when you just want a bunch of social providers,
* [Corbado](https://www.corbado.com/) has support for Face ID, Touch ID, and PassKeys for a more modern approach.

## Further reading

* [The project so far](https://github.com/adrianhall/samples/tree/0914/identity)
* [ASP.NET Identity](https://learn.microsoft.com/aspnet/core/security/authentication/identity)
* [ASP.NET MVC](https://dotnet.microsoft.com/apps/aspnet/mvc)
