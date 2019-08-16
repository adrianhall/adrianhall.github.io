---
title: "Tailwind Photos: Facebook Login"
categories:
  - Android
  - "Tailwind Photos"
tags:
  - Kotlin
---

In [my last post on Tailwind Photos](tailwind-photos-1.md), I set up the splash screen and the buttons I want to use for signing in to the app - social media buttons for Facebook, Google, Microsoft, and Twitter.  In this article, I'm going to go through the process of authenticating Facebook.

## The Facebook side of things

All social media companies have their own methods for setting up authentication, and Facebook is no different.  The instructions I have here are correct as of writing, but you should realize the aim of these instructions:

* Register a native mobile app for authentication.
* Get the codes needed by the SDK to configure Facebook Auth.

These may change in the future.  Let's get started!

1. Log on to [developers.facebook.com](https://developers.facebook.com)
2. If necessary, agree to any terms and conditions to become a Facebook developer.
3. Select **My Apps** > **Create App** in the top-right corner.
4. Enter a display name and your contact email.  I'm using `Tailwind Photos` for my display name.
5. Click **Create App ID**.
6. Click **Integrate Facebook Login** in the scenari selector, then click **Confirm**.
7. Click **Facebook Login** > **Quickstart**.
8. Click **Android**.

What you will get next is a step-by-step process for integrating the Facebook Button for Android, including downloading the SDK, integrating it into your app, and generating all the keys needed.  We'll go through the same process, but in a different order.  That's mostly because I like to separate the "what to do on the website" stuff from "what to do in code".

* Download the Facebook SDK for Android: Click **Next** - we don't need to do this.
* Import the Facebook SDK: Click **Next** - we'll do this during the code section.
* Tell Us about Your Android Project:
    * Enter the package name (mine is `com.tailwind.app.photos`).
    * Enter the deep-linking activity (I'm using `com.tailwind.app.photos.activities.AuthenticatorActivity`).
    * Click **Save**
    * Click **Use this package name** (since your package is not on Google Play yet).
    * Click **Continue**.
* Generate a development key hash (ignore the requirement for a release key hash).

Hmm - how can we do this from Android Studio?  The easiest way is to add some code temporarily to the `SplashActivity` class:

{% highlight kotlin %}
class SplashActivity : AppCompatActivity() {
	override fun onCreate(savedInstanceState: Bundle?) {
		super.onCreate(savedInstanceState)

		printhashkey()

		startActivity(Intent(this, AuthenticatorActivity::class.java))
		finish()
	}

	fun printhashkey() {
		try {
			val info = packageManager.getPackageInfo("com.tailwind.app.photos", PackageManager.GET_SIGNATURES)
			for (signature in info.signatures) {
				val md = MessageDigest.getInstance("SHA")
				md.update(signature.toByteArray())
				Log.d("KeyHash:", Base64.encodeToString(md.digest(), Base64.DEFAULT))
			}
		} catch (e: Exception) {
			e.printStackTrace()
		}
	}
}
{% endhighlight %}

> **DO NOT** leave this code in your application.  It's a deprecated security risk.  There is also a command line version (which the instructions provide) if you never want to include temporary code in your app.  Always get the release key hash using the command line tool.

Run your app and watch the LogCat window:

{% highlight text %}
2019-08-15 11:35:45.656 2990-2990/? I/InstantRun: starting instant run server: is main process
2019-08-15 11:35:45.890 2990-2990/? D/KeyHash:: HXObbXXX8wVtfDHiT6jm14bCCC0=
2019-08-15 11:35:46.520 2990-2990/? W/wind.app.photo: Accessing hidden method Landroid/view/View;->computeFitSystemWindows(Landroid/graphics/Rect;Landroid/graphics/Rect;)Z (light greylist, reflection)
2019-08-15 11:35:46.520 2990-2990/? W/wind.app.photo: Accessing hidden method Landroid/view/ViewGroup;->makeOptionalFitsSystemWindows()V (light greylist, reflection)
2019-08-15 11:35:46.883 2990-2990/com.tailwind.app.photos W/wind.app.photo: Accessing hidden method Landroid/graphics/FontFamily;-><init>()V (light greylist, reflection)
{% endhighlight %}

The second line contains the development key hash.  Copy it into the box provided, then click **Save** followed by **Continue**.  Don't forget to remove this code from your application.

> You need to add the key hashes from every single machine you use for developing this app.  To add further key hashes (including an eventual release key hash):
> * Go back to the [Facebook Developers page](https://developers.facebook.com).
> * Click **My Apps** > _Your App Name_
> * Click **Settings** > **Basic** in the left hand nav (right below the dashboard).
> * Scroll down to the bottom of the page.
> * Copy the hash into the **Key Hashes** field (making sure you don't delete the ones that are already there).
> * Click **Save Changes** at the very bottom of the page.

* Enable Single Sign On for Your App: I generally turn this on because I'm bound to use it eventually.
    * Click the switch icon to turn it to **Yes**.
    * Click **Save**, then **Next**.

In the next step, your **Facebook App ID** and **Login Protocol Scheme** are shown.  

## The App side of things

You're basically done with the Facebook side of things.  Everything else is done in the app, so you could just click **Next** a bunch of times.  However, the SDK does change from time to time, so you can also follow the instructions.

### Add the Facebook SDK to your library

This step was #2 in the list! Edit the project-level `build.gradle`:

{% highlight gradle %}
buildscript {
	ext.kotlin_version = '1.3.41'
	repositories {
		google()
		jcenter()
		mavenCentral()

	}
	dependencies {
		classpath 'com.android.tools.build:gradle:3.4.2'
		classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlin_version"
	}
}

allprojects {
	repositories {
		google()
		jcenter()
		mavenCentral()
	}
}

task clean(type: Delete) {
	delete rootProject.buildDir
}
{% endhighlight %}

Don't click **Sync now** just yet.  We need to add dependencies first.  Next, edit the module-level `build.gradle` file.  Add the Facebook SDK to the `dependencies` section:

{% highlight gradle %}
dependencies {
	implementation fileTree(dir: 'libs', include: ['*.jar'])
	implementation "org.jetbrains.kotlin:kotlin-stdlib-jdk7:$kotlin_version"
	implementation 'androidx.appcompat:appcompat:1.0.2'
	implementation 'androidx.core:core-ktx:1.0.2'
	implementation 'androidx.constraintlayout:constraintlayout:1.1.3'
	implementation 'androidx.cardview:cardview:1.0.0'
	implementation 'com.facebook.android:facebook-android-sdk:[5,6)'
}
{% endhighlight %}

Now you can use that **Sync now** button.

### Edit Your Resources and Manifest

This piece is just just a copy-paste from the Facebook instructions, so I won't bore you with the details.  You need to adjust `res/values/strings.xml` and the `AndroidManifest.xml` file.

### Log app events

The Facebook Login code suggests you should log app activations.  I'm not sure my users want Facebook tracking them any more than they already do.  Since I'm doing "multi-authentication", I'll want all the data in one place anyhow (that's a topic for another day).  

In addition, the code snippet that is shown suggests you should initialize the SDK.  You no longer need to do this, so this entire step can be skipped.

### Add the Facebook Login Button

Well, we've already done this, so let's skip this.  If you came here to learn how to do this, you need to add an `ImageButton` to your layout.

### Register a Callback

Finally, let's get to some code.  "Register a Callback" really means "write all the code that you need to write to make this work".  If you follow the instructions from Facebook, then your authenticator activity looks like spagetti code, and no-one like spagetti code.  I prefer to isolate each authentication provider in its own manager class.  This allows me to do things behind the scenes.  The API surface of the manager is fairly simple:

* Set up `onSuccess`, `onCancel`, and `onFailure` callbacks on initialization so that we can do async responses.
* Handle the `onActivityResult` response, since the SDK will go "elsewhere" and then come back with a response.
* Initiate the sign-in from an `onClickHandler` for the button.

Here is my `AuthenticatorActivity` class:

{% highlight kotlin %}
class AuthenticatorActivity : AppCompatActivity() {
	private val fbManager = FacebookManager()

	override fun onCreate(savedInstanceState: Bundle?) {
		super.onCreate(savedInstanceState)
		setContentView(R.layout.activity_authenticator)

		/* FACEBOOK AUTH INITIALIZATION */
		fbManager
			.onSuccess { user -> moveToNextActivity(user) }
			.onFailure { error -> displayErrorAlert("Facebook", error) }
		facebook_login.setOnClickListener {
			fbManager.beginSignin(this@AuthenticatorActivity)
		}
	}

	override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
		super.onActivityResult(requestCode, resultCode, data)
		fbManager.onActivityResult(requestCode, resultCode, data)
	}

	override fun onResume() {
		super.onResume()
		// Do the same stuff in onResume() - it hasn't changed!
	}

	private fun moveToNextActivity(user: AuthenticatedUser) {
		startActivity(Intent(this, MainActivity::class.java))
		finish()
	}

	private fun displayErrorAlert(provider: String, error: Exception?) {
		val alert = AlertDialog.Builder(this@AuthenticatorActivity)
			.setMessage(error?.message ?: "Unknown Error")
			.setCancelable(false)
			.setPositiveButton("OK") { _, _ -> finish() }
			.create()
		alert.setTitle("$provider Login")
		alert.show()
	}
}
{% endhighlight %}

I can probably implement the same API surface for the other three providers.  In addition, I can likely take the `FacebookManager` with me to other applications.  Code re-use is a wonderful thing.  The `onSuccess` callback returns with an `AuthenticatedUser` object:

{% highlight kotlin %}
package com.tailwind.app.photos.models

enum class AuthenticationProvider {
	FACEBOOK,
	GOOGLE,
	MICROSOFT,
	TWITTER
}

data class AuthenticatedUser(
	val accessToken: String,
	val authProvider: AuthenticationProvider,
	val fullname: String,
	val emailAddress: String)
{% endhighlight %}

In Kotlin parlance, the `AuthenticatedUser` is a common DTO.  Using a data class means I get several methods created for me.  I may decide to change it to a class since I don't actually need the extra methods that are generated on a DTO.  However, the savings are not significant in doing that.

Back to the problem at hand - I now have to write the `FacebookManager` that conforms to the API surface I've described.  Since the `FacebookManager` is only used once, I don't need to use a singleton.  I'm interested in the `AuthenticatedUser` class since I'm abstracting away authentication provider.  If I were repeatedly using the access token and data returned by Facebook, I'd make this class a singleton so that the data is available across multiple activities.

{% highlight kotlin %}
typealias OnSuccessCallback = (AuthenticatedUser) -> Unit
typealias OnCancelCallback = () -> Unit
typealias OnFailureCallback = (Exception?) -> Unit

class FacebookManager {
	private val fbLoginManager = LoginManager.getInstance()
	private val fbCallbackManager = CallbackManager.Factory.create()
	private var onSuccessCallback: OnSuccessCallback? = null
	private var onCancelCallback: OnCancelCallback? = null
	private var onFailureCallback: OnFailureCallback? = null


	init {
		fbLoginManager.registerCallback(fbCallbackManager, object : FacebookCallback<LoginResult> {
			override fun onSuccess(result: LoginResult?) {
				if (result != null) {
					getProfile(result.accessToken) { user -> onSuccessCallback?.invoke(user) }
				} else {
					onFailureCallback?.invoke(RuntimeException("Auth succeeded but no result - this should never happen"))
				}
			}

			override fun onCancel() { onCancelCallback?.invoke() }

			override fun onError(error: FacebookException?) { onFailureCallback?.invoke(error) }
		})
	}

	fun onSuccess(callback: OnSuccessCallback): FacebookManager {
		this.onSuccessCallback = callback
		return this
	}

	fun onCancel(callback: OnCancelCallback): FacebookManager {
		this.onCancelCallback = callback
		return this
	}

	fun onFailure(callback: OnFailureCallback): FacebookManager {
		this.onFailureCallback = callback
		return this
	}

	fun getProfile(accessToken: AccessToken, callback: OnSuccessCallback) {
		val request = GraphRequest.newMeRequest(accessToken) { jsonObject, response ->
			if (response.error != null) {
				onFailureCallback?.invoke(response.error.exception)
			} else {
				val name = jsonObject.optString("name", "Unknown")
				val email = jsonObject.optString("email", "")
				val user = AuthenticatedUser(accessToken.token, AuthenticationProvider.FACEBOOK, id, name, email)
				callback.invoke(user)
			}
		}
		request.parameters = Bundle().apply { putString("fields","id,name,email") }
		request.executeAsync()
	}

	fun beginSignin(activity: Activity) {
		fbLoginManager.logInWithReadPermissions(activity, mutableListOf("public_profile", "email"))
	}

	fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
		fbCallbackManager.onActivityResult(requestCode, resultCode, data)
	}
}
{% endhighlight %}

Let's go through it:

* The `onSuccess`, `onCancel`, and `onFailure` methods establish callbacks for later use.  They return `this` so I can use them in a fluent manner.
* The `getProfile` method is an interesting one that calls the Facebook Graph API to get the name and email address.  Only the name is available in `Profile.getCurrentProfile()`.  There are cases where the user has not provided an email (for example, when the user signs up for Facebook with a phone number from a mobile device).  We'll deal with that wrinkle later on.
* The `beginSignin` method is used in the button click-handler to initiate a sign-in.
* The `onActivityResult` is the callback method.

Most of the code in here is exactly what you would expect if you followed the Facebook instructions.  The `getProfile()` method shows how to do a graph request.  I love to use the [Facebook Graph Explorer](https://developers.facebook.com/tools/explorer/) to determine what fields I need to ask for and whether I have permissions to read them.  However, you have to be prepared for instances where the information is not available.  

## Prep your emulator

You can run this app now in the emulator.  When you press the Facebook logo, it will switch to a web browser, then go through the Facebook authentication before switching back to your app and moving to the MainActivity.   This is pretty awesome already, as it means that a user does not need Facebook on their phone to use your app.  However, you should also install the Facebook app on the emulator and then sign in to the Facebook app.   Once that is complete, restart your app and go through the process again.  Note how the app now switches seamlessly to the Facebook app to ask for permissions, then switches back again.  It's a much better experience.

## Next steps

I've got three authentication providers to go, although the first one is always the hardest since you have to work out how the code flow will happen.  You can find the [code for this step on my GitHub repository](https://github.com/adrianhall/tailwind-photos-for-android/tree/blog-2).  
