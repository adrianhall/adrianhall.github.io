---
title:  "ASP.NET Identity deep dive - Part 4 (Password reset)"
date:   2024-09-16
categories:
  - Web
tags:
  - aspnetcore
  - aspnet_identity
mermaid: true
---

This article is one of a number of articles I will write over the coming month and will go into depth about the [ASP.NET Identity](https://learn.microsoft.com/aspnet/core/security/authentication/identity) system.  My outline thus far:

{% include_relative includes/aspnet-identity-series.md %}

I've already implemented the registration and login/logout functionality. I've also done some updates since the last check in. Most notably, I've automatically signed the user in after registration rather than forcing the user to go through the login process manually.  Given they have just confirmed their account, it feels like a reasonable thing to do. I've also started working on a set of pages that only the administrator can see that will be used for doing account maintenance work.

Today, I'm going to cover another relatively complex user workflow - the forgot password process.

<pre class="mermaid">
{%raw%}
flowchart TD
  id1([User selects Forgot password])
  id2[User is asked for the email address]
  id3{{Does the email address exist?}}
  id4fail[Display the password reset confirmation]
  id4[Send the reset password link to the email]
  id5[Display the password reset confirmation]
  id6([User selects link in email])
  id7[Reset password form is displayed]
  id8[Users password is reset]

  id1-->id2
  id2-->id3
  id3-- Yes -->id4
  id3-- No -->id4fail
  id4-->id5
  id5-->id6
  id6-->id7
  id7-->id8
{%endraw%}
</pre>

You will note from this chart that I will be displaying the same password reset confirmation irrespective of whether the users email exists or not.  This is a security measure to prevent account harvesting.  Other than that, I have a bunch of views and models to create.  As before, I'm not going to show most of the views - they are just HTML and CSS and you should know how to create them.

## The controller

Let's take a look at the additions to the controller first.  There are quite a lot of views to trigger.  The first one to trigger is the initialization.  This is a link to `GET /Account/ForgotPassword`.  I normally put this on the login page - usually right under the password field.

{% highlight csharp linenos %}
[HttpGet, AllowAnonymous]
public IActionResult ForgotPassword([FromQuery] string? returnUrl)
{
    returnUrl = HomePageIfNullOrEmpty(returnUrl);
    if (signInManager.IsSignedIn(User))
    {
        return RedirectToHomePage();
    }

    EmailConfirmationViewModel viewModel = new()
    {
        ReturnUrl = returnUrl
    };

    return View(viewModel);
}

[HttpPost, AllowAnonymous, ValidateAntiForgeryToken]
public async Task<IActionResult> ForgotPassword([FromForm] EmailConfirmationInputModel model)
{
    model.ReturnUrl = HomePageIfNullOrEmpty(model.ReturnUrl);
    if (!ModelState.IsValid)
    {
        EmailConfirmationViewModel viewModel = new(model);
        LogAllModelStateErrors(ModelState);
        return View(viewModel);
    }

    ApplicationUser? user = await userManager.FindByEmailAsync(model.Email);
    if (user is not null)
    {
        await SendResetPasswordLinkAsync(user);
    }

    return RedirectToAction(
        nameof(AwaitPasswordReset),
        new { model.Email, model.ReturnUrl }
    );
}
{% endhighlight %}

The view for "ForgotPassword" is a basic form with an "Email" field that the user fills in.  The form processor looks up the email in the database and sends the reset password link to the email if it finds it.  In either case, the `AwaitPasswordReset` view is triggered:

{% highlight csharp %}
[HttpGet, AllowAnonymous]
public IActionResult AwaitPasswordReset(string email, string returnUrl)
    => View(new EmailConfirmationViewModel { Email = email, ReturnUrl = returnUrl });
{% endhighlight %}

I used the same view model in the registration page.  The `SendResetPasswordLinkAsync()` method is almost identical to the `SendConfirmationLinkAsync()` method I used in the registration user workflow:

{% highlight csharp linenos %}
internal Task SendResetPasswordLinkAsync(ApplicationUser user, CancellationToken cancellationToken = default)
{
    logger.LogTrace("SendResetPasswordLink: {json}", user.ToJsonString());
    return Task.Run(async () =>
    {
        string userId = await userManager.GetUserIdAsync(user);
        string token = await userManager.GeneratePasswordResetTokenAsync(user);
        logger.LogTrace("SendResetPasswordLink: {userId} {token}", userId, token);
        string? callbackUrl = Url.ActionLink(
            action: nameof(ResetPassword),
            values: new { userId, code = EncodeToken(token) },
            protocol: Request.Scheme
        );
        logger.LogTrace("SendResetPasswordLink: {userId} {callbackUrl}", userId, callbackUrl);
        if (callbackUrl is null || user.Email is null)
        {
            logger.LogError("Failed to generate password reset link for user {userId}", userId);
            throw new ApplicationException("Failed to generate password reset link.");
        }
        await emailSender.SendPasswordResetLinkAsync(user, user.Email, callbackUrl);
    }, cancellationToken);
}
{% endhighlight %}

> **Cancellation and ASP.NET Identity**<br/>
> The ASP.NET Identity methods don't generally allow you to pass a `CancellationToken`, so there is no way to cancel any operation.  Sometimes, you want to ensure you can cancel a long-running task.  In these cases, you can wrap the operation in a `Task.Run()` that takes a cancellation token.

When the user receives the email (or - currently - you see the log message), the user can click on the `ResetPassword` link, which calls this:

{% highlight csharp linenos %}
[HttpGet, AllowAnonymous]
public async Task<IActionResult> ResetPassword([FromQuery] string? userId, [FromQuery] string? code)
{
    if (string.IsNullOrEmpty(userId) || string.IsNullOrEmpty(code))
    {
        return RedirectToHomePage();
    }

    ApplicationUser? user = await userManager.FindByIdAsync(userId);
    if (user is null || user.Email is null)
    {
        return RedirectToHomePage();
    }

    string resetToken = DecodeToken(code);
    ResetPasswordViewModel viewModel = new() { Email = user.Email, Token = resetToken };
    return View(viewModel);
}

[HttpPost, AllowAnonymous, ValidateAntiForgeryToken]
public async Task<IActionResult> ResetPassword([FromForm] ResetPasswordInputModel model)
{
    IActionResult DisplayView()
    {
        ResetPasswordViewModel viewModel = new(model);
        return View(viewModel);
    }

    model.ReturnUrl = HomePageIfNullOrEmpty(model.ReturnUrl);
    if (!ModelState.IsValid)
    {
        LogAllModelStateErrors(ModelState);
        return DisplayView();
    }

    ApplicationUser? user = await userManager.FindByEmailAsync(model.Email);
    if (user is null)
    {
        ModelState.AddModelError(string.Empty, "Invalid email address.");
        return DisplayView();
    }

    IdentityResult result = await userManager.ResetPasswordAsync(user, model.Token, model.Password);
    if (!result.Succeeded)
    {
        foreach (IdentityError error in result.Errors)
        {
            ModelState.AddModelError(string.Empty, error.Description);
        }
        return DisplayView();
    }

    await signInManager.SignInAsync(user, isPersistent: false);
    return Redirect(model.ReturnUrl);
}
{% endhighlight %}

The first method is the GET that is called when the user clicks on the link from their email.  The second one is the password reset form processor.  The magic occurs when you call `ResetPasswordAsync()` - pass in the token (from the email link) and the new password to reset the password.  As with my new registration code, once the password is reset, you can sign the user in - they know the password, so it's not a security issue.

## New models

Since I re-use the `EmailConfirmationViewModel`, there is only one new model, and it drives the `ResetPassword` view:

{% highlight csharp linenos %}
public record ResetPasswordInputModel
{
    public ResetPasswordInputModel()
    {
    }

    public ResetPasswordInputModel(ResetPasswordInputModel inputModel)
    {
        Email = inputModel.Email;
        Password = inputModel.Password;
        ConfirmPassword = inputModel.ConfirmPassword;
        Token = inputModel.Token;
        ReturnUrl = inputModel.ReturnUrl;
    }

    [Required, EmailAddress, StringLength(256, MinimumLength = 3)]
    public string Email { get; set; } = string.Empty;

    [Required, DataType(DataType.Password), StringLength(64, MinimumLength = 3)]
    public string Password { get; set; } = string.Empty;

    [Required, DataType(DataType.Password), StringLength(64, MinimumLength = 3)]
    [Compare(nameof(Password), ErrorMessage = "Password and confirmation must match")]
    public string ConfirmPassword { get; set; } = string.Empty;

    [Required]
    public string Token { get; set; } = string.Empty;

    public string ReturnUrl { get; set; } = string.Empty;
}

public record ResetPasswordViewModel : ResetPasswordInputModel
{
    public ResetPasswordViewModel()
    {
    }

    public ResetPasswordViewModel(ResetPasswordInputModel inputModel) : base(inputModel)
    {
    }
}
{% endhighlight %}

## The ResetPassword view

I'm only going to show off one view - the ResetPassword view.  It's not the only view you need, but the others are straight forward.  If you like, you can [check out all the views in the GitHub repository][github].

{% highlight html linenos %}
@model ResetPasswordViewModel
@{
    ViewBag.Title = "Reset password";
    Layout = "_AccountLayout";
}

<div class="text-center">
    <h1 class="h4 text-gray-800 mb-4">Reset your password</h1>
</div>
<form class="user" method="post">
    <div asp-validation-summary="ModelOnly" class="text-danger"></div>
    <input asp-for="Email" type="hidden" />
    <input asp-for="Token" type="hidden" />
    <div asp-validation-summary="All" class="text-danger"></div>

    <div class="form-floating mb-3">
        <input asp-for="Password" class="form-control" placeholder="Password">
        <label asp-for="Password">Password</label>
    </div>
    <div action="invalid-feedback">
        <span asp-validation-for="Password" class="text-danger"></span>
    </div>

    <div class="form-floating mb-3">
        <input asp-for="ConfirmPassword" class="form-control" placeholder="Confirm password">
        <label asp-for="ConfirmPassword">Confirm password</label>
    </div>
    <div action="invalid-feedback">
        <span asp-validation-for="ConfirmPassword" class="text-danger"></span>
    </div>

    <button type="submit" class="btn btn-primary btn-user btn-block">Reset password</button>
</form>
{% endhighlight %}

So, why did I choose this one to show off?  The decoded token has to be passed to the form processor.  I use a hidden field for both the email address and the token. These are non-editable fields needed to select the right account in the identity database and validate that the password reset is allowed.  However, the user doesn't need to see them.  This makes the form relatively clean.

## Final thoughts

You should now have a "minimal" identity solution, including registration, sign-in, sign-out, and self-service password reset.  It comprises 4 input models, 4 view models and a user model, plus 9 views and a controller - all of which need to be maintained as security issues are identified.  We also haven't finished the final piece which answers the question "how do you send email to the end user?"

We also haven't delved into the deeper aspects of identity like multi-factor authentication and social logins.

It's not too late to decide that doing it yourself is a bad idea?  [Keycloak](https://www.keycloak.org/), [Auth0](https://auth0.com/), and [Corbado](https://www.corbado.com/) are still excellent options for you to integrate into your app that doesn't have the coding complexity of an identity solution.

## Further reading

* [The project so far][github]
* [ASP.NET Identity](https://learn.microsoft.com/aspnet/core/security/authentication/identity)
* [ASP.NET MVC](https://dotnet.microsoft.com/apps/aspnet/mvc)

[github]: https://github.com/adrianhall/samples/tree/0916/identity