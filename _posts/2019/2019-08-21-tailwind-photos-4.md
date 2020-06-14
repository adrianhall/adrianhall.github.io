---
title: "Tailwind Photos: Microsoft Login"
categories:
  - Android
  - "Tailwind Photos"
tags:
  - Kotlin
---

Today, I am continuing with the authentication story for my app - Tailwind Photos - and tackling [Microsoft authentication](https://docs.microsoft.com/en-us/azure/active-directory/develop/). 
The story so far:

* [The Splash Screen]({% post_url 2019/2019-08-15-tailwind-photos-1 %})
* [Facebook Login]({% post_url 2019/2019-08-17-tailwind-photos-2 %})
* [Google Login]({% post_url 2019/2019-08-19-tailwinds-photos-3 %})

 You will see a lot of the same techniques as previous methods - just updated for todays topic.  Let's get started!

The bright side of todays topic is that, with a few twists, you can use this same code if your app targets enterprise users.  There is a point in the configuration where you need to select "Anyone" and "Enterprise users only" - just make the right selection!

## The Azure Active Directory side of things

Start by signing onto the [Azure portal](https://portal.azure.com).  The [account is free](https://azure.microsoft.com/en-us/free/), and doing the actual sign-up is like any other cloud provider.  Once you have got into your account:

* Open the Azure Active Directory blade:
  * Click **All services** in the left-hand menu.
  * Enter _Active Directory_ in the search box
  * Click **Azure Active Directory**.

> You can favorite any resource types you use on a regular basis to get easy access to them in the left-hand menu.

* Click **App registrations** in the blade menu.
* Click **New registration**.
  * Enter a name for this app.  I used `Tailwind Photos for Android`.
  * Select the supported account types.  I used _Accounts in any organizational directory and personal Microsoft accounts (e.g. Skype, Xbox)_.  This is where you can lock your authentication down to just enterpise users if needed.
  * Under **Redirect URI**, select **Public client (mobile & desktop), then enter a redirect URI that is unique to your app.  I'm using `tailwind-photos://auth`.  Pick your own redirect, though.
* Click **Register**.

This will give you an **Application (client) ID** which you will need later.

## The app side of things

As with the other libraries, we need to do a little bit of setup.  Let's start with the library.  We previously added `mavenCentral()` as a repository, so we only need to add the library in the module level `build.gradle` file:

{% highlight gradle %}
dependencies {
  // Rest of the dependencies go here
	implementation 'com.facebook.android:facebook-android-sdk:5.2.0'
	implementation 'com.google.android.gms:play-services-auth:17.0.0'
	implementation 'com.microsoft.identity.client:msal:0.2.2'
}
{% endhighlight %}

Next, we already have the `INTERNET` permission, but we also need the `ACCESS_NETWORK_STATE` permission in the `AndroidManifest.xml` file:

{% highlight xml %}
	<uses-permission android:name="android.permission.INTERNET" />
	<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
{% endhighlight %}

Also in the `AndroidManifest.xml`, we need to add a definition of the activity that the MSAL (Microsoft Authentication Library) uses:

{% highlight xml %}
		<!-- Microsoft Authentication -->
		<activity android:name="com.microsoft.identity.client.BrowserTabActivity">
			<intent-filter>
				<action android:name="android.intent.action.VIEW"/>
				<category android:name="android.intent.category.DEFAULT"/>
				<category android:name="android.intent.category.BROWSABLE"/>
				<data android:scheme="tailwind-photos" android:host="auth"/>
			</intent-filter>
		</activity>
{% endhighlight %}

The `data` values come from the redirect URI that you entered in the configuration.  I entered `tailwind-photos://auth`, so that is where the scheme and host are derived from.  Notice how this activity definition is similar to the Facebook definition!

We need to create an `msal_config.json` file that contains the configuration we have set up.  Here is an example:

{% highlight json %}
{
	"client_id": "12345678-abcd-4dc1-96f0-14c0ccaf829c",
	"authorization_user_agent": "DEFAULT",
	"redirect_uri": "tailwind-photos://auth",
	"authorities": [
		{
			"type": "AAD",
			"audience": {
				"type": "AzureADandPersonalMicrosoftAccount"
			}
		}
	]
}
{% endhighlight %}

To create this:

* Right-click on `res`, select **New** > **Directory**.  Enter the name `raw`.
* Right-click on the newly created `res/raw` directory, select **New** > **File**.  Enter the name `msal_config.json`.
* Copy the above JSON into the newly created file, replacing the `client_id` and `redirect_uri` values with your own.

Let's get on with the code.  As before, I've added calls into `AuthenticatorActivity` to call the abstracted authentication class:

{% highlight kotlin %}
class AuthenticatorActivity : AppCompatActivity() {
	// Facebook and Google manager variables
	private lateinit var msaManager: MicrosoftManager

	override fun onCreate(savedInstanceState: Bundle?) {
		super.onCreate(savedInstanceState)
		setContentView(R.layout.activity_authenticator)

		/* FACEBOOK AUTH INITIALIZATION */
		/* GOOGLE AUTH INITIALIZATION */

		/* MICROSOFT AUTH INITIALIZATION */
		msaManager = MicrosoftManager(applicationContext)
		msaManager
			.onSuccess { user -> moveToNextActivity(user) }
			.onFailure { error -> displayErrorAlert("Microsoft", error) }
		microsoft_login.setOnClickListener {
			msaManager.beginSignin(this@AuthenticatorActivity)
		}
	}

	override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
		super.onActivityResult(requestCode, resultCode, data)

		// Facebook and Google onActivityResult calls
		msaManager.onActivityResult(requestCode, resultCode, data)
	}

  // Rest of the activity
}
{% endhighlight %}

This code should be familiar by now since we've done the same thing for each authentication implementation.  Let's take a look at the `MicrosoftManager` class:

{% highlight kotlin %}
typealias OnMicrosoftSuccessCallback = (AuthenticatedUser) -> Unit
typealias OnMicrosoftCancelCallback = () -> Unit
typealias OnMicrosoftFailureCallback = (Exception?) -> Unit

class MicrosoftManager(context: Context) {
	private var onSuccessCallback: OnMicrosoftSuccessCallback? = null
	private var onCancelCallback: OnMicrosoftCancelCallback? = null
	private var onFailureCallback: OnMicrosoftFailureCallback? = null

	private val scopes = arrayOf(
		"https://graph.microsoft.com/User.Read"
	)

	private val client = PublicClientApplication(context, R.raw.msal_config)

	fun onSuccess(callback: OnMicrosoftSuccessCallback): MicrosoftManager {
		this.onSuccessCallback = callback
		return this
	}

	fun onCancel(callback: OnMicrosoftCancelCallback): MicrosoftManager {
		this.onCancelCallback = callback
		return this
	}

	fun onFailure(callback: OnMicrosoftFailureCallback): MicrosoftManager {
		this.onFailureCallback = callback
		return this
	}

	fun beginSignin(activity: Activity) {
		val account = if (client.accounts.isEmpty()) null else client.accounts[0]
		if (account != null) {
			client.acquireTokenSilentAsync(scopes, account, object : AuthenticationCallback {
				override fun onSuccess(authenticationResult: AuthenticationResult?) {
					userInformationCallback(authenticationResult)
				}

				override fun onCancel() {
					onCancelCallback?.invoke()
				}

				override fun onError(exception: MsalException?) {
					if (exception is MsalUiRequiredException)
						signInInteractively(activity)
					else
						onFailureCallback?.invoke(exception)
				}
			})
		} else {
			signInInteractively(activity)
		}
	}

	private fun signInInteractively(activity: Activity) {
		client.acquireToken(activity, scopes, object : AuthenticationCallback {
			override fun onSuccess(authenticationResult: AuthenticationResult?) {
				userInformationCallback(authenticationResult)
			}

			override fun onCancel() {
				onCancelCallback?.invoke()
			}

			override fun onError(exception: MsalException?) {
				onFailureCallback?.invoke(exception)
			}
		})
	}

  private fun userInformationCallback(authenticationResult: AuthenticationResult?) {
    if (authenticationResult == null) {
      onFailureCallback?.invoke(RuntimeException("auth result is null"))
      return
    }
    Log.d("MicrosoftManager", "SUCCESS!")
  }

	fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
		if (data == null)
			onFailureCallback?.invoke(RuntimeException("response data is null"))
		else
			client.handleInteractiveRequestRedirect(requestCode, resultCode, data)
	}
}
{% endhighlight %}

This is all fairly standard boiler-plate code for MSAL.  However, I've not completed the task.  Specifically, I've left what happens in `userInformationCallback()`.  When you press the Microsoft login button, it will go through the normal authentication mechanism, eventually calling `userInformationCallback()` with an authentication result.  The authentication result only contains the access token.  This is a JWT used for authentication purposes on back end services.

We need the name and email address as well.  Fortunately, the information is contained within the JWT and JSON web tokens are not hard to decode.  If you need to validate a JSON web token, then it is best to use a library.  If you want to decode a JSON web token, they you just have to be aware that they are made up of two JSON sections that are base-64 encoded, plus a signature.  You can replace the `userInformationCallback()` with the following code:

{% highlight kotlin %}
	private fun userInformationCallback(authenticationResult: AuthenticationResult?) {
		if (authenticationResult == null) {
			onFailureCallback?.invoke(RuntimeException("auth result is null"))
			return
		}

		try {
			val b64body = authenticationResult.idToken.split(".")[1]
			val body = JSONObject(String(Base64.decode(b64body, Base64.URL_SAFE)))
			val name = body.optString("name", "")
			val email = authenticationResult.account.username
			val user = AuthenticatedUser(
						authenticationResult.idToken,
						AuthenticationProvider.MICROSOFT,
						name, email)
			onSuccessCallback?.invoke(user)
		} catch (exception: Exception) {
			onFailureCallback?.invoke(exception)
		}
	}
{% endhighlight %}

This will now construct the right `AuthenticatedUser` object without further network calls.  You could, instead, do a Graph lookup to Microsoft Graph for the information.  However, that's an additional network call; mobile apps have enough to do!

## Next Steps

You should be able to run the app at this point and authenticate with Microsoft authentication.  If you stop and start the app again, pressing the Microsoft authentication button will silently authenticate you - there is no need for another prompt if the token is still valid.

In the next article, I'll tackle Twitter authentication.  Until then, the [code is in GitHub](https://github.com/adrianhall/tailwind-photos-for-android/tree/blog-4).
