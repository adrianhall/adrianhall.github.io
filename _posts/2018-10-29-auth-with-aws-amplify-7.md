---
title: "Authentication with AWS Amplify and Android: Integrating Biometrics"
categories:
  - Android
  - AWS
tags:
  - Kotlin
  - "AWS Amplify"
  - "Amazon Cognito"
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

This is the seventh in the series covering how to authenticate with the backend service using Biometrics - specifically, fingerprints. 

I should note here that I am Android introduced a new API called `BiometricPrompt` in API level 28 (Pie). I find this API to be a bit problematic in many respects. Because Pie is so new and because the new API is such a problem, I am not using that API. Instead, I am using the older API. This means my target API level is 27 for this part of the project.

If you went looking for biometric authentication on the web, you probably found lots of ways to authenticate your user using their fingerprint and not one of them allows you to authenticate to a backend service. That's because you are looking for the wrong thing. You need to look for examples for storage secured by biometrics. In essence, you don't authenticate the user. You store the users password in secured storage, and retrieve it when needed. To access the secure area, you need to use biometric authentication.

> You don't need anything special on the server side to handle biometric storage of credentials.

In this article, I'm going to use a reactive secure storage library written by Square called Whorlwind. Handling secure storage is time consuming and a lot of the code is boiler plate. By utilizing the library, we get out of the business of writing boiler plate code.
So, what do we need to do?

1. Add the Whorlwind library to the app
2. Add permissions to the app for handling fingerprints
3. Ask for permission to use fingerprints in our app
4. Initialize the Whorlwind library
5. Save the password to Secure Storage on a successful login
6. Load the password from Secure Storage if it exists

The Whorlwind library takes care of the rest.

## Add Whorlwind to the app

Whorlwind is based on [RxJava](https://github.com/ReactiveX/RxJava), so you will actually need to add three libraries to your dependencies:

```gradle
implementation "io.reactivex.rxjava2:rxjava:2.1.3"
implementation "io.reactivex.rxjava2:rxandroid:2.1.0"
implementation "com.squareup.whorlwind:whorlwind:2.0.0"
```

Don't forget to synchronize your IDE so you can use the new libraries

## Add permissions to the app

Permissions are handled in the `AndroidManifest.xml` file:

```xml
<!-- Biometrics -->
<uses-feature
    android:name="android.hardware.fingerprint"
    android:required="false"/>
<uses-permission
    android:name="android.permission.USE_FINGERPRINT"/>
```

The feature states that we don't "require" a fingerprint reader, but we declare that we use it so that it displays the requirement within the Google Play Store if you distribute your app.

## Ask for permissions to use the fingerprint reader

Just because you have the permission listed in the `AndroidManifest.xml` doesn't mean that your users have given the app that permission. We need to ask for permission if it has not been granted. I've done this in the `AuthenticatorActivity`. First, let's set some things up:

```kotlin
class AuthenticatorActivity : AppCompatActivity() {
    companion object {
        private const val REQUEST_PERMISSIONS_FINGERPRINT = 90001
    }

    private val model by viewModel<AuthenticatorViewModel>()
    private val analyticsService by inject<AnalyticsService>()

    // For permissions checks
    private var checkedPermissions = false
    private var hasPermissions = false

    // For fingerprint storage
    private var whorlwind: Whorlwind? = null
    private val mDisposable = CompositeDisposable()

    // Rest of class
}
```

Permissions are handled by calling out to the OS and then handling the response - much the same way that dealing with the camera, as an example, is done. To handle the request, we need a request code which is located in the companion object. In addition, we've got a couple of booleans to ensure that we don't perpetually ask for permissions when they have been denied. 

I've also shown off the private variables I need to handle the Whorlwind library - we'll be using these shortly.

In the `onCreate()` method, we need to check for permissions and initiate a permissions request if we don't have them. I put this at the bottom of the `onCreate()` method so that it's the last thing that happens:

```kotlin
// Ask for permission to use the fingerprint scanner
if (fingerprintManager.isHardwareDetected) {
    if (checkSelfPermission(Manifest.permission.USE_FINGERPRINT) != PackageManager.PERMISSION_GRANTED) {
        if (checkedPermissions) {
            hasPermissions = false
        } else {
            requestPermissions(arrayOf(Manifest.permission.USE_FINGERPRINT), REQUEST_PERMISSIONS_FINGERPRINT)
        }
    } else {
        hasPermissions = true
    }
}
```

Here, `fingerprintManager` is retrieved using `getSystemService()`. I also need to handle the response from the OS:

```kotlin
/**
  * Callback for when the permissions has been requested and responded to.
  */
override fun onRequestPermissionsResult(requestCode: Int, permissions: Array<out String>, grantResults: IntArray) {
    when (requestCode) {
        REQUEST_PERMISSIONS_FINGERPRINT -> {
            checkedPermissions = true
            hasPermissions = (grantResults.isNotEmpty() && grantResults[0] == PackageManager.PERMISSION_GRANTED)
        }

        else -> {
            super.onRequestPermissionsResult(requestCode, permissions, grantResults)
        }
    }
}
```

This is fairly standard boiler-plate code for handling permissions checks. You should be able to re-use this.

## Initialize the Whorlwind library

In the `onCreate()` method, add the following:

```kotlin
whorlwind = Whorlwind.create(this,
  SharedPreferencesStorage(this, "amazon.cognito"), "cognito")
```

This will initialize the Whorlwind library.

## Save the password securely

We want biometric storage to be available whenever it is needed, so we need to store the password securely whenever there is a successful login. In the `handleLogin()` method, I updated the `SUCCESS` case with the following:

```kotlin
IdentityRequest.SUCCESS -> {
    analyticsService.recordSuccessfulLogin(loginFormUsernameField.text.toString())
    model.updateStoredUsername(loginFormUsernameField.text.toString())
    saveToBiometricStore(loginFormPasswordField.text.toString())
    this@AuthenticatorActivity.finish()
}
```

This just calls a new method to store the password:

```kotlin

/**
 * Save the current form data to the biometric store
 */
private fun saveToBiometricStore(password: String) {
   if (whorlwind?.canStoreSecurely() == true) {
        val disposable = Observable.just(password)
               .observeOn(Schedulers.io())
               .flatMapCompletable {value ->
                    whorlwind?.write("password", ByteString.encodeUtf8(value))
               }
               .subscribe()
        mDisposable.add(disposable)
    } else {
       toast("Biometric storage is not available")
    }
}
```

You can "fail silently" if you wish, but I've added a toast (from Anko) so that you can see when biometrics is not available.

Biometrics are available when:

* The hardware is present
* Permission to use the fingerprint sensor has been approved
* There is an enrolled finger print
* A secure device lock-screen has been configured

The actual code is almost a direct copy from the Whorlwind sample app and is a good example on how to securely store data.

## Load the password from secure storage

The final step is to load the data. I added a "fingerprint icon" to the UI with an ID of `loginFormFingerprintButton`. This is wired up within the `onCreate()` method to call my data loader:

```kotlin
loginFormFingerprintButton.onClick { loadFromBiometricStorage() }
```

This is similar to all the other buttons on the page. Now, let's load the data:

```kotlin
/**
  * Load the data from the biometric store and populate the right fields
  */
private fun loadFromBiometricStore() {
    if (whorlwind?.canStoreSecurely() == true) {
        val disposable = whorlwind!!.read("password")
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe { result ->
                    when (result.readState) {
                        ReadResult.ReadState.NEEDS_AUTH -> {
                            toast("WHORLWIND - NEEDS_AUTH")
                        }

                        ReadResult.ReadState.UNRECOVERABLE_ERROR,
                        ReadResult.ReadState.AUTHORIZATION_ERROR,
                        ReadResult.ReadState.RECOVERABLE_ERROR -> {
                            toast("WHORLWIND - ERROR")
                        }

                        ReadResult.ReadState.READY -> {
                            Log.d(TAG, "WHORLWIND - READY - password = ${result.value?.utf8() ?: "null"}")
                            val password = result.value?.utf8() ?: ""
                            with (loginFormPasswordField.text) {
                                clear()
                                append(password)
                            }
                        }

                        else -> {
                            toast("WHORLWIND - EEEK!")
                        }
                    }
                }
        mDisposable.add(disposable)
    } else {
        toast("Biometric storage is not available")
    }
}
```

When Whorlwind reads the storage, it can find itself in two states - it either needs authentication or it can provide the data. All other conditions are errors which you can silently eat or print errors for. I produce toasts for each of these conditions.

Let's talk about those two states though.

* If the state is `NEEDS_AUTH`, then you should prompt for a fingerprint. There are lots of tutorials on how to do fingerprint authentication in a dialog. There are even [plenty of helper libraries](https://android-arsenal.com/tag/238?sort=created). For now, I'm just popping up a toast to say "you need to touch the fingerprint sensor"
* If the state is `READY` and the value is non-null, then the data is available. At this point, you can fill in the password and then call handleLogin() to submit the form. No further input is required from the user.
* If the state is `READY` and the value is null, there is no value stored in the secure storage. You should continue on as if nothing had happened and prompt for a password.

Right now, I'm just filling in the password field, but there is nothing stopping you from calling `handleLogin()` immediately afterwards.

## Try it out!

If you are using the emulator, you will need to register a fingerprint before continuing. You can do this by swiping down to get into the settings and doing the fingerprint the normal way.

To access the emulated fingerprint manager, open the extended controls (the triple dots at the bottom of the menu), then select Fingerprint. There is a button to simulate a finger print touch.

First, authenticate with your regular username and password. Then sign out. Finally, click the sign-in button, but this time touch the fingerprint marker. You'll get the toast (`NEEDS_AUTH`). Touch the fingerprint reader (or press the button to emulate a sensor touch) and you will notice the password is filled in for you.

## Wrap up

Note that if you have multi-factor authentication configured, then the user will still be prompted for their MFA token (provided via SMS or generated by a TOTP application). We are only storing the credentials securely on device - we aren't bypassing the security provided by the backend service.

Biometric security is awesome and adds a feature to your app that users will appreciate. It means remembering less passwords, and an easier login experience. The Whorlwind library, despite its lack of documentation, makes it easy to configure.

You can even support "just biometrics" by generating a random password during sign-up, then storing it in secure storage. The user then only has to touch the fingerprint reader to authenticate and doesn't have a password to remember.

As always, you can check out the code at [my GitLab repository](https://gitlab.com/adrianhall/aws-mobile-android-kotlin-photos/tree/auth-biometrics).
