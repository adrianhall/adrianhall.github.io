---
title: "Tailwind Photos: Registration (Authentication on Android)"
categories:
  - Android
  - "Tailwind Photos"
tags:
  - Kotlin
  - "Azure Functions"
---

We are almost at the end of "registration and authentication" - and it's been a seriously long way, which just goes to show the amount of complexity in the subject.  Thus far, we've authenticated with [Facebook][1], [Google][2], and [Microsoft][3], handled [silent login][4] and [configured the backend resources][5].  In the [last article][6], we configured the backend resources to handle authentication, allowing us to swap a social identity provider token with a ZUMO token that we can use for further authentication.

> A **ZUMO token** is just a JWT that the Azure App Service or Azure Functions generates.  It's called a ZUMO token because it was created for A**ZU**re **MO**bile.  It's name is legacy at this point.

Today, we are going to add the code necessary to get the ZUMO token from the backend within the Android code.  I have a little bit of a confession to make though.  I re-architected the authenticator activity structure again.  I wasn't happy with the way that the code flowed through the repository.  Since the authenticator activity is the only place where the social identity providers get handled, I decided to merge them all back into one activity and get rid of the repository.

## Create a service class

When I interact with a backend service, I tend to isolate the service into a service class.  This service class has a 1:1 relationship with the service.  Every single function you can do with the service has a method that just calls the service and decodes the response.  In this case, I've created `AzureFunctionsAuthService`, a class for dealing with the `/.auth` endpoints.  It starts like this:

{% highlight kotlin %}
class AzureFunctionsAuthService(private val baseUrl: URL) {
  // ...
}
{% endhighlight %}

It takes a `baseUrl`, which I store in the `strings.xml` file:

{% highlight xml %}
<!-- Tailwind Photos backend -->
<string name="backend_url">https://twprodfunctions.azurewebsites.net</string>
{% endhighlight %}

This URL is the output from my ARM template, so I can do an automatic build whereby I deploy my ARM template, gather the outputs, update the `strings.xml` file, and then build the Android app.  I doubt I will ever do the complete run - the backend and the frontend code are completely separate.  However, I **can** do it if I needed to.

For the HTTP client, there are a couple of options:

* The Apache HttpClient for Android.
* The Google HttpClient.
* Square OkHttp.
* Retrofit2 with one of the other providers.

There are others as well.  Let's ignore the Apache and Google Http clients.  The Google one is fairly new.  The Apache one is a good option, but I'm intending on using Retrofit2 later on.  Retrofit is great at modeling a REST service in code.  For this job, however, it's a little too much.  The job I want to do is simple.  [OkHttp](https://square.github.io/okhttp/) is a great open-source project with a lot of community behind it and I'm used to coding with it, so that's my selection.

> Even if you have experience with a library, don't be afraid to go looking for something better.  You never know when you are going to find a library that has a more usable API or better support.  Libraries can and do fall out of fashion and get abandoned, even with significant usage.  You should try to use the best library for the job at the point you write the app.

To add `okHttp` to your app, just add a dependency into the app-level `build.gradle` file:

{% highlight gradle %}
dependencies {
  # Other dependencies here
  implementation 'com.squareup.okhttp3:okhttp:3.10.0'
}
{% endhighlight %}

Sync your project to download the library.  If this were a new app, then you would need to add the `INTERNET` permission to the app.  We've already got a lot of network calls happening with authentication, so it's already done.

Next, we will want to store the return values from the service into our user model.  We could create a whole new user model.... or just extend the existing one, which is what I opted to do:

{% highlight kotlin %}
data class SocialAuthUser(val provider: String) {
  var accessToken: String? = null
  var idToken: String? = null
  var zumoToken: String? = null
  val map = HashMap<String,String>()
}
{% endhighlight %}

Now, let's return to the `AzureFunctionsAuthService` class.  In the [last article][6], I provided the format of the JSON block needed to be sent to the login endpoint and what the service responded with.  The code will do this translation for us, but it also needs to handle error conditions.

Before I get to that, I need to decide "async or sync".  If I go with async code, I need to set up callbacks that handle the response and errors, but I don't need to create an additional thread manually so that my network code is not happening on my UI thread.  If I go with sync code, the code I write is a lot simpler and I can do everything as a single function with no callbacks.  However, I have to wrap the call in an AsyncTask or similar thread.  For this job, I am going to go with synchronous code.  It's only called in one place, and I can significantly simplify the code.  I do, however, have to be careful when I call it from the `AuthenticatorActivity`.

{% highlight kotlin %}
class AzureFunctionsAuthService(private val baseUrl: URL) {
  private val client = OkHttpClient()
  private val jsonMediaType = MediaType.parse("application/json")

  /**
   * Send the social user authentication token to the back end for validation
   *
   * @param user the social user record
   * @return the social user record (updated)
   * @throws UnauthenticatedException if the service returns a 401 Unauthenticated
   * @throws ServiceException if the call fails for some other reason
   */
  fun login(user: SocialAuthUser): SocialAuthUser {
    // Create the appropriate URL from the base URL
    val url = URL(baseUrl, "/.auth/login/${user.provider}")

    // Create the JSON Object payload
    val jObject = JSONObject()
      .putOpt("id_token", user.idToken)
      .putOpt("access_token", user.accessToken)
      .toString()

    // Create the POST request with the JSON Object as the body
    val request = Request.Builder()
        .url(url)
        .post(RequestBody.create(jsonMediaType, jObject))
        .build()

    // Do the actual POST operation
    val response = client.newCall(request).execute()

    // If it isn't successful, throw something appropriate
    if (!response.isSuccessful) {
      if (response.code() == 401) {
        throw UnauthenticatedException("User is unauthenticated")
      } else {
        throw ServiceException("Backend service failure", response)
      }
    }

    // If it is successful, extract the token and user ID field.
    val responseBody = response.body()!!.string()
    val rj = JSONObject(responseBody)
    user.zumoToken = rj.getString("authenticationToken")
    user.map["user_id"] = rj.getJSONObject("user").getString("userId")
    return user
  }
}
{% endhighlight %}

When an error occurs, I need to decide if I am going to throw an error or not.  For "expected normal conditions", I will just trap it and return a value.  For "expected errors", I will throw a unique exception (such as the `UnauthenticatedException` here).  Finally, all the others get a common `ServiceException`.  This basically says "something went wrong but I couldn't handle it".  These are just regular exceptions with potentially additional arguments that are stored as properties.

## Update the AuthenticatorActivity

There are two places that I need to update within the `AuthenticatorActivity` to use this service class.  Firstly, I need to initialize the service within the `onCreate` method:

{% highlight kotlin %}
  // Initialize the Tailwind Service
  val tailwindUrl = URL(resources.getString(R.string.backend_url))
  tailwindService = AzureFunctionsAuthService(tailwindUrl)
{% endhighlight %}

The `tailwindService` variable is just a class-level private variable.  

I have a method called `authenticateUser()`.  Eventually, all the social identity provider flows end up in `authenticateUser()` with a `SocialAuthUser` object filled in with the provider information.  Before adding this functionality, it just stored the provider in a shared preference so I could call the appropriate silent login method.  The new code looks like this:

{% highlight kotlin %}
  /**
   * When we have a valid social auth user object, call this to authenticate to the tailwind
   * photos backend to get an authenticated user object
   *
   * @param {SocialAuthUser} user the user object from the social auth provider.
   */
  private fun authenticateUser(user: SocialAuthUser) {
    // Store the provider in shared prefs so we do silent auth next time
    storeAuthPref(user.provider)

    // Exchange the IdP token for a ZUMO token
    thread(start = true) {
      try {
        tailwindService.login(user) // User is updated here
        vm.setUser(user)
        startActivity(Intent(this, MainActivity::class.java))
      } catch (error: Exception) {
        runOnUiThread { onError(R.string.service_auth, error) }
      }
    }
  }
{% endhighlight %}

First, I call the new service code to swap the social IdP token for a ZUMO token.  Then I update a repository (via a view model) with that new information.  Lots of other activities will eventually need to have that information.  Finally, I switch to the new activity (in this case, the `MainActivity`).

## Running the app

The best way to run this app to ensure everything is working is in an emulator with the Facebook app already installed and configured.  Set a breakpoint at the `authenticateUser` method, and another at the `onError` method.  Now run in debug mode.

You will want to inspect the `user` object immediately after the `tailwindService.login()` call and note that the `zumoToken` property is filled in.  In addition, note that a `user_id` field has been added to the map for the user.

If something goes wrong, then the `onError` will be called and you can inspect the exception to see what is going on.  I almost got this code working the first time.  Setting a breakpoint on `onError` allowed me to quickly identify the error.  A quick google later (and reference to the excellent `okHttp` documentation) allowed me to fix the error.

## Next steps

We are just one article away from having a complete social identity provider based authentication and registration process for Android.  In the next article, I'll cover the Azure Function that interacts with the Cosmos database to do registration.  The article after that will cover detecting registration is required and displaying the registration screen.

Until then, you can take a look at [my latest code](https://github.com/adrianhall/tailwind-photos-for-android/tree/blog-8) on my GitHub repository.

<!-- Links -->
[1]: {% post_url 2019/2019-08-17-tailwind-photos-2 %}
[2]: {% post_url 2019/2019-08-19-tailwinds-photos-3 %}
[3]: {% post_url 2019/2019-08-21-tailwind-photos-4 %}
[4]: {% post_url 2019/2019-08-23-tailwind-photos-5 %}
[5]: {% post_url 2019/2019-09-03-tailwind-photos-6 %}
[6]: {% post_url 2019/2019-09-10-tailwind-photos-7 %}
