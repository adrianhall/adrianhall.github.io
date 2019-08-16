---
title: "Authentication with AWS Amplify and Android: Facebook Login"
categories:
  - Android
tags:
  - Kotlin
---

This will be an in-depth series on authentication with [AWS Amplify]. Here are the topics I am going to cover, and I will update each blog with the links as I complete the articles.

* [The basics - a username/password system]({% post_url 2018-09-18-auth-with-aws-amplify-1 %}).
* [Customizing the UI]({% post_url 2018-09-23-auth-with-aws-amplify-2 %}).
* [Authenticating with Facebook]({% post_url 2018-10-01-auth-with-aws-amplify-3 %}).
* [Authenticating with Google]({% post_url 2018-10-08-auth-with-aws-amplify-4 %}).
* [Using third-party authentication providers]({% post_url 2018-10-15-auth-with-aws-amplify-5 %}).
* [Using Time-based One-time passwords (TOTP)]({% post_url 2018-10-22-auth-with-aws-amplify-6 %}).
* [Using Biometric authentication]({% post_url 2018-10-29-auth-with-aws-amplify-7 %}).
* [Doing fraud protection and analytics]({% post_url 2018-11-05-auth-with-aws-amplify-8 %}).

This is part 3 — authenticating with Facebook. It builds on what we have done in parts 1 and 2. If you haven’t looked at the identity repository and how to build a custom UI, you should definitely go do that first.

So far, we’ve covered implementing a username and password authentication scheme with Amazon Cognito user pools. We’ve looked at how to get the basics going, connecting the sign in action to a button, and customizing all the flows — sign-in, sign-up and self-service password reset.

Most users don’t want to remember yet another password, however. Giving the user the option to use Facebook, Google, or some other authentication provider is a good practice. In the new couple of articles, I’m going to walk through configuring the authentication system so that we can use these third party authentication providers, starting with Facebook.

The process is fairly simple, but it is a multi-step process:

1. Get an Application ID from the Facebook Developers console.
2. Add your application to the Facebook Application registration.
3. Configure the Amazon Cognito identity pool to federate using the application ID you just received.
4. Integrate the Facebook authentication provider into your app.
5. Send the Facebook authentication token you receive from signing in with Facebook to the Amazon Cognito identity pool for federation.

## Step 1: Get an Application ID from Facebook

I should note here that every single web site on the planet changes over time. While I will put a specific process down here, it is “at time of writing”. Things may change by the time you read this. The end result is important — you want an application ID for logging in with Facebook.

First, log in to the [Facebook Developers portal](https://developers.facebook.com/). If you have never visited this portal before, there is a sign-up agreement to become a Facebook developer. Depending on whether you have created an app before:

* If you don’t have any apps registered, click the big friendly green **Add a New App** button.
* If you have registered an app, click the **My Apps** menu, then **Create a New App**.

The form will have already filled in the contact email — you just have to fill in the display name, then click **Create App ID**. There may be a security check, and then you are onto app configuration.

## Step 2: Add your application to the Application registration

Facebook needs to know which specific applications are going to be using the Facebook login. To do that, you need to give Facebook the key hash for your mobile app. First, you need to generate a development key for your Android environment:

{% highlight bash %}
keytool -exportcert -alias androiddebugkey -keystore ~/.android/debug.keystore | openssl sha1 -binary | openssl base64
{% endhighlight %}

You will be prompted to enter a key store password (which can be anything — you are generating a password), then you get a string of characters ending in the equals sign. You need this later on, so don’t lose it.

Back in the Facebook developers console:

* Click **Settings** > **Basic** in the left-hand menu.
* Scroll to the bottom of the page, then click **Add Platform**.
* Select **Android** from the list.
* Turn the **Single Sign On** switch to **Yes**.
* Set the **Google Play Package Name** to be the package name of your app (in my case, it is _com.amazonaws.mobile.samples.picturefeed_)
* Set the **Class Name** to be the activity that will receive the deep link for when Facebook returns control to your app. In my case, this is _com.amazonaws.mobile.samples.picturefeed.ui.auth.AuthenticatorActivity_.
* Set the **Key Hashes** to be the result of the keytool output from earlier in this step.
* Click **Save Changes**.
* You probably have not registered the name space with Google Play yet, so click **Use this package name** when prompted. (This prompt will not appear if you have registered the name space with Google Play).

There are two modes for Facebook apps — Development and Production. Only users who have been registered as testers will be able to authenticate with your app while it is in development. You and your friends are automatically testers, but if you need to add someone else:

* Click **Roles** > **Roles** in the left-hand menu.
* Click **Add Testers**.
* Enter the name, fbid or username of the person, then click **Submit**.

You can find the users username easily enough. Go to their profile on the main Facebook site. Their username is listed in the URL.

## Step 3: Add Facebook to the Amazon Cognito identity pool

Since we are using [AWS Amplify CLI](https://aws-amplify.github.io/) for our backend configuration, this becomes easily accomplished:

{% highlight bash %}
amplify update auth
{% endhighlight %}

The answers to the questions are the same, except for this one:

```
? Do you want to enable 3rd party authentication providers in your identity pool (Use arrow keys)
```

Answer **Yes** here. Then select **Facebook**. Cut and paste the App ID into the appropriate slot.

At this point, the Facebook configuration is done. You still have to do the rest of the authentication setup. However, we have previously set all this up, so you can press Enter the rest of the way.

The final step is to deploy the backend:

{% highlight bash %}
amplify push
{% endhighlight %}

> Want to take a look at what is happening behind the scenes? All the configuration is placed in an Amazon CloudFormation template (with one template per Amplify category). Take a look in `amplify/backend/auth` for the details on the authentication settings.

## Step 4: Integrate the Facebook SDK into your app

You should always follow the [Facebook documentation](https://developers.facebook.com/docs/facebook-login/android) for the latest instructions. However, here is how I integrated it into my app. Start by adding the SDK into your app dependencies within `build.gradle`:

{% highlight gradle %}
implementation 'com.facebook.android:facebook-login:4.36.0'
{% endhighlight %}

Ensure you sync your project. Now, add the following two lines to your `strings.xml` file:

{% highlight xml %}
<string name="facebook_app_id">YOUR-APP-ID</string>
<string name="fb_login_protocol_scheme">fbYOUR-APP-ID</string>
{% endhighlight %}

Replace _YOUR-APP-ID_ with your Facebook App ID from the Facebook Developers portal. Now, edit your `AndroidManifest.xml` You should already have the `INTERNET` permission (but add it if you don’t). Add the following meta-data element inside the application node:

{% highlight xml %}
<!-- Facebook Requirements -->
<meta-data
    android:name="com.facebook.sdk.ApplicationId"
    android:value="@string/facebook_app_id"/>
<activity
    android:name="com.facebook.FacebookActivity"
    android:configChanges="keyboard|keyboardHidden|screenLayout|screenSize|orientation"
    android:label="@string/app_name"/>
{% endhighlight %}

There are two more steps that we need to do. Firstly, we need to add a Facebook button to the login screen. This is included in the XML for my layout:

{% highlight xml %}
<com.facebook.login.widget.LoginButton
    android:id="@+id/loginFormFacebookLoginButton"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:layout_gravity="start" />
{% endhighlight %}

Then, in the `onCreate()` method of the `AuthenticatorActivity`, I’ve included the following:

{% highlight kotlin %}
loginFormFacebookLoginButton.setReadPermissions(listOf("email"))
{% endhighlight %}

This pulls the email address of the user in addition to their public profile. Be careful what you ask for as asking for some items trigger an additional app review.

The second thing you must do is to register a callback to receive the result of the login. Create a callback manager within the `AuthenticatorActivity` and handle the activity result return:

{% highlight kotlin %}
// Facebook Requirement
private val callbackManager: CallbackManager = CallbackManager.Factory.create()

override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
    // Handle response from Facebook SDK
    callbackManager.onActivityResult(requestCode, resultCode, data)
    super.onActivityResult(requestCode, resultCode, data)
}
{% endhighlight %}

Then, in the `onCreate()` method, register the callback:

{% highlight kotlin %}
// Register the callback for the Facebook authentication
LoginManager.getInstance().registerCallback(callbackManager, object : FacebookCallback<LoginResult> {
    override fun onSuccess(result: LoginResult?) {
        Log.d(TAG, "Facebook token = ${result?.accessToken?.token}")
        toast("Access Token received")
   }

    override fun onCancel() {
        Log.d(TAG, "Facebook authentication is cancelled")
        toast("Facebook Authentication is cancelled")
    }

    override fun onError(error: FacebookException?) {
        Log.e(TAG, "Facebook Authentication failed: ${error?.message}")
        toast("Facebook Authentication failed: ${error?.message}")
    }
})
{% endhighlight %}

You should now be able to run your app and authenticate with Facebook, but it doesn’t actually authenticate with the app as the callback doesn’t do anything with the access token.

## Step 5: Federate with Amazon Cognito identity pools

The final piece of the puzzle is to make Amazon Cognito aware that we have logged in with Facebook. Firstly, I am going to add a method to our Identity Repository called `federateWithFacebook` that takes an access token and sends it to Amazon Cognito. Here is the AWS implementation:

{% highlight kotlin %}
/**
  * Federate with Facebook authentication
  */
override fun federateWithFacebook(accessToken: AccessToken) {
    Log.d(TAG, "Federating with Facebook")
    thread(start = true) {
        with(service.identityManager.underlyingProvider) {
            clear()
            withLogins(mapOf("graph.facebook.com" to accessToken.token))
            refresh()
        }

        val user = User().apply {
            username = accessToken.userId
            tokens[TokenType.ACCESS_TOKEN] = accessToken.token
            userAttributes["provider"] = "graph.facebook.com"
        }
        Log.d(TAG, "Federated result: ${service.identityManager.isUserSignedIn}")
        runOnUiThread { mCurrentUser.postValue(user) }
    }
}
{% endhighlight %}

I also need to add a little bit of code into the `init` block of the identity repository to read the Facebook access token:

{% highlight kotlin %}
// Check to see if Facebook is authenticated
// if it is, then federate with facebook
val accessToken = AccessToken.getCurrentAccessToken()
if (accessToken != null && !accessToken.isExpired) {
    federateWithFacebook(accessToken)
}
{% endhighlight %}

Finally, I can adjust the `onSuccess()` method of my Facebook login callback to close the activity is the authentication is successful:

{% highlight kotlin %}
// Register the callback for the Facebook authentication
LoginManager.getInstance().registerCallback(callbackManager, object : FacebookCallback<LoginResult> {
    override fun onSuccess(result: LoginResult?) {
        result?.let { model.federateWithFacebook(it.accessToken) }
        this@AuthenticatorActivity.finish()
    }
    override fun onCancel() {
        Log.d(TAG, "Facebook authentication is cancelled")
        toast("Facebook Authentication is cancelled")
    }
    override fun onError(error: FacebookException?) {
        Log.e(TAG, "Facebook Authentication failed: ${error?.message}")
        toast("Facebook Authentication failed: ${error?.message}")
    }
})
{% endhighlight %}

Phew!

There is a lot more to do here and subtleties with the Facebook authentication that must be taken care of. Note, however, the following:

* You must use an Amazon Cognito identity pool. Even though Amazon Cognito user pools can do Facebook authentication, you can’t combine that functionality with a user pool in a mobile app.
* There is no “sign out” functionality with Facebook. You have to sign out of Amazon Cognito (using the sign out button), then sign out again when you click on Facebook again! If you close and re-open the app, Facebook is still authenticated and you will be re-signed in. (Yes, not ideal).
* In an emulator, you don’t have Facebook installed, so you will be prompted for a username and password (and potentially a code). However, in a “real world” scenario, the app will switch to Facebook, ask you to approve the app login, then switch back to your app.
* If you are using the Amazon `auth-ui` package and the basic UI, then you can easily add Facebook authentication with just a couple of lines of code. Check [the documentation](https://docs.aws.amazon.com/aws-mobile/latest/developerguide/add-aws-mobile-user-sign-in.html) for detailed instructions.

Next time, I’ll be covering Google authentication, which looks remarkably similar. 