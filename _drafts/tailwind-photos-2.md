---
title: "Tailwind Photos: The Splash Screen"
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

```kotlin
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
```

> **DO NOT** leave this code in your application.  It's a deprecated security risk.  There is also a command line version (which the instructions provide) if you never want to include temporary code in your app.  Always get the release key hash using the command line tool.

Run your app and watch the LogCat window:

```text
2019-08-15 11:35:45.656 2990-2990/? I/InstantRun: starting instant run server: is main process
2019-08-15 11:35:45.890 2990-2990/? D/KeyHash:: HXObbXXX8wVtfDHiT6jm14bCCC0=
2019-08-15 11:35:46.520 2990-2990/? W/wind.app.photo: Accessing hidden method Landroid/view/View;->computeFitSystemWindows(Landroid/graphics/Rect;Landroid/graphics/Rect;)Z (light greylist, reflection)
2019-08-15 11:35:46.520 2990-2990/? W/wind.app.photo: Accessing hidden method Landroid/view/ViewGroup;->makeOptionalFitsSystemWindows()V (light greylist, reflection)
2019-08-15 11:35:46.883 2990-2990/com.tailwind.app.photos W/wind.app.photo: Accessing hidden method Landroid/graphics/FontFamily;-><init>()V (light greylist, reflection)
```

The second line contains the development key hash.  Copy it into the box provided, then click **Save** followed by **Continue**.  Don't forget to remove this code from your application.

> You need to add the key hashes from every single machine you use for developing this app.  To add further key hashes (including an eventual release key hash):
> * TODO: Add in instructions on adding the additional developer codes
> *

* Enable Single Sign On for Your App: I generally turn this on because I'm bound to use it eventually.
    * Click the switch icon to turn it to **Yes**.
    * Click **Save**, then **Next**.

In the next step, your **Facebook App ID** and **Login Protocol Scheme** are shown.  

## The App side of things

You're basically done with the Facebook side of things.  Everything else is done in the app, so you could just click **Next** a bunch of times.  However, the SDK does change from time to time, so you can also follow the instructions.

### Add the Facebook SDK to your library

This step was #2 in the list! Edit the project-level `build.gradle`:

```gradle hl_lines="6,19"
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
```

Don't click **Sync now** just yet.  We need to add dependencies first.  Next, edit the module-level `build.gradle` file.  Add the Facebook SDK to the `dependencies` section:

```gradle hl_lines="8"
dependencies {
	implementation fileTree(dir: 'libs', include: ['*.jar'])
	implementation "org.jetbrains.kotlin:kotlin-stdlib-jdk7:$kotlin_version"
	implementation 'androidx.appcompat:appcompat:1.0.2'
	implementation 'androidx.core:core-ktx:1.0.2'
	implementation 'androidx.constraintlayout:constraintlayout:1.1.3'
	implementation 'androidx.cardview:cardview:1.0.0'
	implementation 'com.facebook.android:facebook-android-sdk:[5,6)'
}
```

Now you can use that **Sync now** button.

### Edit Your Resources and Manifest

This piece is just just a copy-paste from the Facebook instructions, so I won't bore you with the details.  You need to adjust `res/values/strings.xml` and the `AndroidManifest.xml` file.

### Log app events

The Facebook Login code suggests you should log app activations.  I'm not sure my users want Facebook tracking them any more than they already do.  Since I'm doing "multi-authentication", I'll want all the data in one place anyhow (that's a topic for another day).  

In addition, the code snippet that is shown suggests you should initialize the SDK.  You no longer need to do this, so this entire step can be skipped.

### Add the Facebook Login Button

Well, we've already done this, so let's skip this.  If you came here to learn how to do this, you need to add an `ImageButton` to your layout.

### Register a Callback

Finally, let's get to some code.  "Register a Callback" really means "write all the code that you need to write to make this work".  This includes:

* Creating a `LoginManager` and `CallbackManager`
* Register a facebook callback to handle the request
* Setting up a click-listener to trigger the login
* Handling the response from the request

This is all handled in the following contents of the `AuthenticatorActivity`:

```kotlin hl_lines="2-3,9-29,32-35"
class AuthenticatorActivity : AppCompatActivity() {
	private val fbLoginManager = com.facebook.login.LoginManager.getInstance()
	private val fbCallbackManager = com.facebook.CallbackManager.Factory.create()

	override fun onCreate(savedInstanceState: Bundle?) {
		super.onCreate(savedInstanceState)
		setContentView(R.layout.activity_authenticator)

		/* FACeBOOK AUTH INITIALIZATION */
		fbLoginManager.registerCallback(fbCallbackManager, object : com.facebook.FacebookCallback<com.facebook.login.LoginResult> {
			override fun onSuccess(result: com.facebook.login.LoginResult?) {
				Log.d("AuthenticatorActivity", "Successfully logged in with Facebook")
			}

			override fun onCancel() { }

			override fun onError(e: com.facebook.FacebookException) {
				val alert = AlertDialog.Builder(this@AuthenticatorActivity)
					.setMessage("Error signing in with Facebook")
					.setCancelable(false)
					.setPositiveButton("OK") { _, _ -> finish() }
					.create()
				alert.setTitle("Facebook Login")
				alert.show()
			}
		})
		facebook_login.setOnClickListener {
			fbLoginManager.logInWithReadPermissions(this@AuthenticatorActivity, arrayListOf("email", "public_profile"))
		}
	}

	override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
		super.onActivityResult(requestCode, resultCode, data)
		fbCallbackManager.onActivityResult(requestCode, resultCode, data)
	}

    // Leave onResume() alone
}
```

Most of this is boiler-plate code.  Where you do get some leeway is in how to handle `onSuccess` and `onError`.  These callbacks are called when the Facebook process returns.  For the error condition, I've created a dialog.  I'll probably replace this with a prettier dialog later on.  For right now, I'm just outputting the success to logcat, but I can set a break point to see what is returned:

![]({{ site.baseurl}}/assets/tailwinds-2-image1.png)

The important part is the access token.  What I want to do with this is:

1. Get the users email address and name from the profile.
2. Call my own `IdentityManager` with this information to say "I'm logged in"
3. Move to a new activity with this information.

The identity manager will be a singleton service that the rest of the app can use for storing identity information.

