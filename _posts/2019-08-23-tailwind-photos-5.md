---
title: "Tailwind Photos: Silent Login"
categories:
  - Android
  - "Tailwind Photos"
tags:
  - Kotlin
---

Thus far in our story, we've covered [Facebook]({% post_url 2019-08-17-tailwind-photos-2 %}), [Google]({% post_url 2019-08-19-tailwinds-photos-3 %}), and [Microsoft]({% post_url 2019-08-21-tailwind-photos-4 %}) authentication.  There is one more to do - Twitter.  Unfortunately, Twitter doesn't have a nice vendor-provided SDK to do the work.  In fact, Twitter is fairly hostile to app developers, so I decided to forego the Twitter login (sorry!).  Instead, I'm going to cover the changes I made to support silent login.

Up until now, everything has been in "managers" - one for each authentication provider.  This has done all the work for each provider.  However, to support silent login, I need to provide two paths to the same information - one silently (during the time when the spinner is active) and one interactively (when the user clicks on a button).  The easiest way to do this is to use an observer pattern.  In an observer pattern, you establish a variable that can send events to observers when its value changes.  In our case, each path will post the sign in information to the observable, and then the UI can react to it by observing the changes.

Android comes with an observable as part of its [Android Architecture Components](https://developer.android.com/topic/libraries/architecture) called `LiveData`.  This is one of a trio of components that make up the architecture components infrastructure.  The others are `Repository` and `ViewModel`.  Since my app is going to evolve extensively, I'm going to restructure my app around the architecture components.

> Step 0: **Integrate a dependency injection system**.  There are multiple dependency injection systems available, and I'm not going to tell you which one to use.  The most popular is [Dagger2](https://dagger.dev/) since it's produced by Google.  I don't like it, and prefer [Koin](https://insert-koin.io/).  Use whichever you like.

Once you have your preferred dependency injection system selected, you will need to:

* Create a repository interface
* Create an implementation of the repository interface
* Create a view model for your authenticator activity
* Update the authenticator activity to use the new view model

Let's take each of these in turn:

## Create a repository interface.

Here is mine:

{% highlight kotlin %}
interface IdentityRepository {
  val authenticatedUser: LiveData<AuthenticatedUser?>
  val error: LiveData<Exception?>

  fun silentlySignIn(context: Context)
  fun interactivelySignIn(activity: Activity, provider: AuthenticationProvider)
  fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?)
}
{% endhighlight %}

I'm using two observable variables here - one that will be updated when the user logs in, and another that updates when an error condition occurs.  I'll eventually tie the former to the code that moves to the next activity and the latter to something that pops up an alert.

After that, my repository has three methods - one used to sign in silently, one that produces an interactive sign-in for a particular provider and another that handles the result from the external authentication.  Each provider (Facebook, Google, and MSAL) uses a web view to complete the transaction, and then redirects back to your app.  When the redirect back to your app occurs, the `onActivityResult()` methods captures it to complete the authentication.

## Create a repository implementation

I thought about re-using the individual auth provider managers I had previously written, but realized pretty quickly that it was a bad idea for a variety of reasons.  Most notably, each provider has a different way of reporting silent logins back to you.  One does it synchronously (no callback); one does it in the activity result and another does it in a different callback.  Since there is no standard way, it's best to do it all in one repository.  So my repository covers all three providers.

Most of the code is practically identical to the original code.  Instead of calling the callbacks (which won't exist any more as we are using observables), the callback handlers update the observable variables.  The only difference is in the `silentlySignIn()` method.  This needs to handle cases when the user is not signed in and cases when the user is signed in.  Here is my code:

{% highlight kotlin %}
  override fun silentlySignIn(context: Context) {
    if (!prefs.contains(PREFS_KEY)) {
      postAnonymousUser()
    } else {
      val provider = prefs.getString(PREFS_KEY, null)
      if (provider == null) {
        postAnonymousUser()
      } else {
        when (provider.toLowerCase()) {
          "facebook" -> {
            val accessToken = AccessToken.getCurrentAccessToken()
            if (accessToken != null && !accessToken.isExpired) {
              getFacebookProfile(accessToken)
            } else {
              postAnonymousUser()
            }
          }

          "google" -> {
            val task = googleClient.silentSignIn()
            if (task != null) {
              try {
                val account = task.getResult(ApiException::class.java) ?: throw RuntimeException("account is null")
                val idToken = account.idToken ?: throw RuntimeException("account ID is null")
                val user = AuthenticatedUser(idToken, AuthenticationProvider.GOOGLE,
                  account.displayName ?: "Unknown", account.email ?: "")
                mutableUser.postValue(user)
              } catch (error: Exception) {
                postAnonymousUser()
              }
            } else {
              postAnonymousUser()
            }
          }

          "microsoft" -> {
            val account = if (msalClient.accounts.isEmpty()) null else msalClient.accounts[0]
            if (account != null) {
              msalClient.acquireTokenSilentAsync(msalScopes, account, object : AuthenticationCallback {
                override fun onSuccess(authenticationResult: AuthenticationResult?) {
                  try {
                    authenticationResult?.run { getMicrosoftProfile(this) }
                  } catch (ex: Exception) {
                    postAnonymousUser()
                  }
                }
                override fun onCancel() { postAnonymousUser() }
                override fun onError(exception: MsalException?) { postAnonymousUser() }
              })
            } else {
              postAnonymousUser()
            }
          }
        }
      }
    }
  }
{% endhighlight %}

We first read from the preferences file to see if an authentication has happened that has completed all the way through to the end.  If there is, we call the appropriate "silent sign in" process for that provider.  In all cases where something doesn't exist (there is no preference file; there is no preference; the key is expired; there was an error), we post the "anonymous" user:

{% highlight kotlin %}
  private fun postAnonymousUser() {
    mutableUser.postValue(AuthenticatedUser("", AuthenticationProvider.ANONYMOUS, "", ""))
  }
{% endhighlight %}

The `mutableUser` variable is the mutable version of the `authenticatedUser` in the interface.  They are defined like this:

{% highlight kotlin %}
  private val mutableUser: MutableLiveData<AuthenticatedUser?> = MutableLiveData()

  override val authenticatedUser: LiveData<AuthenticatedUser?>
    get() = mutableUser
{% endhighlight %}

When you post a value to the `mutableUser`, anything that establishes an observer on the `authenticatedUser` property gets notified of the change.  In other code paths, I post an `AuthenticatedUser` that is fully formed for the signed in case. 

This is a rather long code re-org, but you can find the complete code on [my GitHub repository](https://github.com/adrianhall/tailwind-photos-for-android/blob/blog-5/app/src/main/java/com/tailwind/app/photos/repositories/impl/IdentityRepositoryImpl.kt)

## Create a view model

The view model is the intermediary between the activity and the repositories.  View models might compose for multiple different repositories, or transform results according to needs.  In this case, there is a 1:1 between the needs of the activity and the repository, so most things are "pass-through".  Here is the code:

{% highlight kotlin %}
class AuthenticatorViewModel(private val identityRepository: IdentityRepository): ViewModel() {
  val authenticatedUser = identityRepository.authenticatedUser
  val identityError = identityRepository.error

  fun silentlySignIn(context: Context)
    = identityRepository.silentlySignIn(context)

  fun interactivelySignIn(activity: Activity, provider: AuthenticationProvider)
    = identityRepository.interactivelySignIn(activity, provider)

  fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?)
    = identityRepository.onActivityResult(requestCode, resultCode, data)
}
{% endhighlight %}

## Update the activity

The activity is actually much simpler, so I'm just going to throw the code for `onCreate()` first and walk through it:

{% highlight kotlin %}
  private val vm : AuthenticatorViewModel by viewModel()

  override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_authenticator)

    val distance = resources.getDimensionPixelSize(R.dimen.social_media_button_offset).toFloat()

    // Handle the login events
    vm.authenticatedUser.observe(this, Observer { user ->
      user?.run {
        if (user.authProvider == AuthenticationProvider.ANONYMOUS) { // There is no logged in user - show the auth buttons
          auth_progress_bar.visibility = View.GONE
          social_media_login_buttons
            .animate()
            .translationY(-distance)
            .setDuration(500L)
            .alpha(1.0f)
            .setListener(null)
        } else { // There is a logged in user - move to the next step
          social_media_login_buttons
            .animate()
            .translationY(distance)
            .setDuration(250L)
            .alpha(0.0f)
            .setListener(null)
          auth_progress_bar.visibility = View.VISIBLE
          // SEE IF REGISTERED
        }
      }
    })

    // Handle the error events
    vm.identityError.observe(this, Observer { error ->
      error?.run {
        val alert = AlertDialog.Builder(this@AuthenticatorActivity)
          .setMessage(error?.message ?: "Unknown Error")
          .setCancelable(false)
          .setPositiveButton("OK") { _, _ -> finish() }
          .create()
        alert.setTitle("Sign In Error")
        alert.show()
      }
    })

    // Wire up the various sign-in provider buttons
    facebook_login.setOnClickListener  { vm.interactivelySignIn(this, AuthenticationProvider.FACEBOOK) }
    google_login.setOnClickListener    { vm.interactivelySignIn(this, AuthenticationProvider.GOOGLE) }
    microsoft_login.setOnClickListener { vm.interactivelySignIn(this, AuthenticationProvider.MICROSOFT) }
  }
{% endhighlight %}

First, I bring in the view model I just wrote via dependency injection.  How you do this depends on your preferred DI mechanism.  I like Koin for its simplicity.  Now, let's look at the code for the `onCreate()` method.  There are two distinct areas - setting up the observers for our observable data, and setting up the click handlers for the interactive sign in buttons.  Concentrate on the `vm.authenticatedUser` observer.  It basically says "when the authenticated user changes, do something.  If the user is ANONYMOUS, when show the sign in buttons.  If the user is anything else, when hide the sign in buttons and move to the next activity".

I haven't written the next activity yet, but I'm intending to put a registration step there.

Now, let's look at the (much simpler) `onResume()` and `onActivityResult()` methods:

{% highlight kotlin %}
  override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
    super.onActivityResult(requestCode, resultCode, data)
    vm.onActivityResult(requestCode, resultCode, data)
  }

  override fun onResume() {
    super.onResume()

    thread(start = true) {
      vm.silentlySignIn(this)
    }
  }
{% endhighlight %}

This is much easier now.  The `onActivityResult()` just calls the view model version, which will call the repository version in turn.  The `onResume()` method initiates the silent sign-in process.

So, what will happen when you run the app?  There are two cases - one where you have previously signed in successfully and have a valid token, and another where you have not previously signed in OR do not have a valid token.

In the latter case:

* The activity calls `silentlySignIn()` on the repository (via the view model).
* The repository posts the anonymous user to the `authenticatedUser` property.
* The activity notices that and shows the sign in buttons.
* The user clicks on a sign-in button to initiate a sign-in.
* The auth provider brings up a web view to complete the auth, redirecting back to the app.
* The activity redirects into the repository to complete the authentication.
* The auth provide callback is called with a new token.
* An `AuthenticatedUser` object is created and posted to the `authenticatedUser` property.
* The activity notices that and hides the sign in buttons before "doing something else".

In the former case:

* The activity calls `silentlySignIn()` on the repository (via the view model).
* The repository posts the stored user to the `authenticatedUer` property.
* The activity notices that and ensures the sign in buttons are hidden before "doing something else".

Short version: if the user has signed-in already, they will not see a sign-in button.  They will just be taken to the app.

## Next steps

I've left the code as "registration happens here", and that is my next step.  Part of that process is to ensure I've created a user on my own cloud backend and completed a sign in process completed before writing the auth provider to the preferences file (so that silent sign-in will work).  Next step is to do the registration process.

Until then, check out the latest code on [my GitHub repository](https://github.com/adrianhall/tailwind-photos-for-android/tree/blog-5).
