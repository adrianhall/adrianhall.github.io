---
title: "Tailwind Photos: Google Login"
categories:
  - Android
  - "Tailwind Photos"
tags:
  - Kotlin
---

Thus far in this series, I've [established a decent splash screen and auth trigger page](2019-08-15-tailwind-photos-1.md), and [integrated Facebook authentication](2019-08-17-tailwind-photos-2.md) into my app.  The next authentication technique is [Google Auth](https://developers.google.com/identity/sign-in/android/start-integrating).  If you followed along with the Facebook auth integration, you will know that I separated the actual Facebook part of it into a separate class, allowing me to clean up the code in the actual activity.  I also established provider-agnostic models for the authenticated user, which I will re-use in this post.

Onto Google!  It follows the same pattern as Facebook - you do some work to configure a project on the Google Developers site, then integrate some code in your app.  Along the way, you will need to provide some data about your app to Google, and take some information from Google to hook the app to the right configuration.

## Getting your signing certificate information

Before we begin with the Google side of things, let's talk about the signing certificate.  This is the same as the "key hash" that was provided for Facebook, so we already have a method of getting the signing certificate.  It's just in the wrong form (which - admittedly - we could correct).  However, I do want to provide an alternate to using code - the `keytool` command line tool.

1. Install a Java JDK.  I use [the Azul Zulu Community Edition](https://www.azul.com/downloads/zulu-community/?&os=windows) since I don't like what Oracle did to the licensing.  However, you can use any version you want.
2. Locate the `keytool`.  Ensure the location is on your `PATH`.  How to do this depends on your platform, and Google is your friend.  For Azul Zulu, the installer adds it to your path.
3. Run the following command:

{% highlight bash %}
keytool -list -v -keystore .android/debug.keystore
{% endhighlight %}

The password is `android` for  debug keystore.  You will see output similar to the following:

![]({{site.baseurl}}/assets/images/tailwinds-3-image1.png)

The highlighted content (next to SHA-1) is what you will need.  You can find [the full instructions in the Google Guides](https://developers.google.com/android/guides/client-auth).

## The Google side

You have a Google account, right?  Then go to the [Google documentation](https://developers.google.com/identity/sign-in/android/start-integrating) and click the **Configure a project** button.

> You can also configure the service side by going to the [Google API Console](https://console.developers.google.com) and creating a project from scratch.  I find this to be more cumbersome.  I like the wizard that the **Configure a project** link provides.

If you already have a project, then you can select it from a drop-down and just use that, alternatively (and this is what I am doing), select **Create a new project**.  If you don't have any Google projects, the project selector is skipped and you get deposited right at the "Create a new project" step.

* Enter a name for the project.  I called mine `Tailwind Photos`.
* Enter a product name.  Again, I entered `Tailwind Photos`.  Since this is the name that the user sees, ensure it matches your brand. 
* In **Configure your OAuth client**, select **Android**.
    * Enter the package name of your app from your `AndroidManifest.xml` - mine is `com.tailwind.apps.photos`
    * Enter the SHA-1 signing certificate you retrieved before you started this section in the box provided.

This will give you a Client ID and Client Secret.  Copy these off somewhere.  Do not put them in your Android app!  Once you have copied them off, click **DONE**.

> The Google client relies on the signing certificate for your app to identify itself.  You need to use the same signing certificate on all your clients.  If you develop (as I do) on multiple machines, you will need to add additional Android clients:
> * Go to the [Google Developers Console](https://console.developers.google.com) and sign in.
> * Select your app, if required.  If you only have one app, it will be automatically selected for you.
> * Click **Credentials** in the left-hand menu.
> * Click **Create credential**.
> * Select **Android**.
> * Fill in the new signing certificate SHA-1 and the same package name.
> * Click **Create**.
> You should now be able to sign in from your new development box.

Finally, when you go through this process, the Google systems will create a web client for you with a unique client ID and client secret.  You need the client ID later on:

* Go to the [Google Developers Console](https://console.developers.google.com) and sign in.
* Select your app, if required.  If you only have one app, it will be automatically selected for you.
* Click **Credentials** in the left-hand menu.
* Look for "Web client (Auto-created for Google Sign-in)".
* Copy the client ID associated with this client.

Now, let's move to the app side!

## The App side

First, I said we needed the client ID for the web client.  Let's put that in our `res/values/strings.xml` file:

{% highlight xml %}
<resource>
	<!-- the rest of the strings are here -->

	<!-- Google Sign-in -->
	<string name="google_client_id">1234567890-abcdefghijklmnopqr.apps.googleusercontent.com</string>
</resources>
{% endhighlight %}


> It's ok to include client IDs in the app - they aren't considered security information.  However, you should never include client secrets in mobile apps.  Mobile apps can be decoded once they are released.  Any string you include in your app will be available for anyone to see.

Just replace my (obviously faked) client ID above with the one your copied from the Google developers console earlier.  Next, as with all these integrations, there is a library to add to the module-level `build.gradle` file:

{% highlight gradle %}
dependencies {
	// Rest of the dependencies go here	
	implementation 'com.facebook.android:facebook-android-sdk:5.2.0'
	implementation 'com.google.android.gms:play-services-auth:17.0.0'
}
{% endhighlight %}

Like the Facebook mechanism, I want to abstract the code for handling Google.  I'm going to try and use a similar API surface.  Here is the code for the `AuthenticatorActivity` (at least the important bits):

{% highlight kotlin %}
class AuthenticatorActivity : AppCompatActivity() {
	private lateinit var fbManager: FacebookManager
	private lateinit var googleManager: GoogleManager

	override fun onCreate(savedInstanceState: Bundle?) {
		super.onCreate(savedInstanceState)
		setContentView(R.layout.activity_authenticator)

		/* FACEBOOK AUTH INITIALIZATION */
		fbManager = FacebookManager()
		fbManager
			.onSuccess { user -> moveToNextActivity(user) }
			.onFailure { error -> displayErrorAlert("Facebook", error) }
		facebook_login.setOnClickListener {
			fbManager.beginSignin(this@AuthenticatorActivity)
		}

		/* GOOGLE AUTH INITIALIZATION */
		googleManager = GoogleManager(this)
		googleManager
			.onSuccess { user -> moveToNextActivity(user) }
			.onFailure { error -> displayErrorAlert("Google", error) }
		google_login.setOnClickListener {
			googleManager.beginSignin(this@AuthenticatorActivity)
		}
	}

	override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
		super.onActivityResult(requestCode, resultCode, data)
		fbManager.onActivityResult(requestCode, resultCode, data)
		googleManager.onActivityResult(requestCode, resultCode, data)
	}

	// The rest of the class is unchanged
}
{% endhighlight %}

I've been deliberate with the API surface here.  I want this side of things to be very readable and as close to the `FacebookManager` API as possible.  It's almost exactly the same.  In face, the only thing that is different is that I have to pass a `Context` to the constructor for Google.  Let's take a look at the manager:

{% highlight kotlin %}
typealias OnGoogleSuccessCallback = (AuthenticatedUser) -> Unit
typealias OnGoogleFailureCallback = (Exception?) -> Unit

class GoogleManager(context: Context) {
	companion object {
		const val GOOGLE_SIGN_IN_RC = 6001
	}

	private var signInClient: GoogleSignInClient
	private var onSuccessCallback: OnGoogleSuccessCallback? = null
	private var onFailureCallback: OnGoogleFailureCallback? = null

	init {
		val options = GoogleSignInOptions.Builder(GoogleSignInOptions.DEFAULT_SIGN_IN)
			.requestIdToken(context.getString(R.string.google_client_id))
			.requestEmail()
			.build()
		signInClient = GoogleSignIn.getClient(context, options)
	}

	fun onSuccess(callback: OnGoogleSuccessCallback): GoogleManager {
		this.onSuccessCallback = callback
		return this
	}

	fun onFailure(callback: OnGoogleFailureCallback): GoogleManager {
		this.onFailureCallback = callback
		return this
	}

	fun beginSignin(activity: Activity) {
		activity.startActivityForResult(signInClient.signInIntent, GOOGLE_SIGN_IN_RC)
	}

	fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
		if (requestCode == GOOGLE_SIGN_IN_RC) {
			val task = GoogleSignIn.getSignedInAccountFromIntent(data)
			try {
				val account = task.getResult(ApiException::class.java) ?: throw RuntimeException("account is null")
				val idToken = account.idToken ?: throw RuntimeException("account ID is null")
				val user = AuthenticatedUser(idToken, AuthenticationProvider.GOOGLE,
					account.displayName ?: "Unknown", account.email ?: "")
				onSuccessCallback?.invoke(user)
			} catch (error: Exception) {
				onFailureCallback?.invoke(error)
			}
		}
	}
}
{% endhighlight %}

This code is pretty much taken from the Google sign-in instructions.  The one wrinkle is that you need to pass the client ID to `.requestIdToken()` to get an OAuth ID token that is used to authenticate with your backend.  We haven't got there yet, but we will.  

There are two possible errors that you may run into:

* An `ApiException` generally indicates that you have messed up the Google console configuration.  Check that the information in the Google developers console matches your app exactly.  The SHA-1 hash and the package identifier have to match.
* If the `idToken` is null, then the web client ID is probably wrong.  Again, check that the client ID you included in the `strings.xml` file matches the client ID in the Google developers console for the `Web client` OAuth credential.

## Next steps

You should go through the process of signing in.  You will need to set up your Google account on the emulator the first time through, but it won't ask for your password after that - it will just switch seamlessly between asking and returning.  In most cases, you won't even get prompted - it will just return.

Next time, I'm going to cover Microsoft authentication (which can cover both enterprise authentication and outlook.com authentication).  Until then, the code is [on my GitHub repository](https://github.com/adrianhall/tailwind-photos-for-android/tree/blog-3).
