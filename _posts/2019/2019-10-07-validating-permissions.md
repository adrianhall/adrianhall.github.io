---
title: "Validating permissions on Android with Kotlin"
categories:
  - Mobile
tags:
  - android
  - kotlin
---

I'm continuing my educational coding exercise, developing a new photo sharing app.  Recently, I completed my user authentication and registration process and I'm now quite happy with it, so I moved on to taking photographs.  I've got an activity with a toolbar and a floating action button:

{% highlight xml %}
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:id="@+id/main_container"
    tools:context=".activities.MainActivity">

    <include
        layout="@layout/main_toolbar"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

    <com.google.android.material.floatingactionbutton.FloatingActionButton
        android:id="@+id/main_fab_camera"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginEnd="32dp"
        android:layout_marginBottom="32dp"
        android:clickable="true"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:srcCompat="@drawable/ic_camera_white_48dp" />

</androidx.constraintlayout.widget.ConstraintLayout>
{% endhighlight %}

In the `MainActivity`, I've set up an `onClickListener` as follows:

{% highlight kotlin %}
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        setSupportActionBar(findViewById(R.id.toolbar))

        main_fab_camera.setOnClickListener { validatePermissions() }
    }
{% endhighlight %}

The question before me is this: How do I ask for permissions on Android?

## The hard way

Naturally, Android has its [own system of doing things](https://developer.android.com/training/permissions/requesting).  It boils down to this:

1. Declare the permissions you want in your `AndroidManifest.xml` file.
2. Use `ContextCompat.checkSelfPermission()` to see if you have the permission.
3. If you don't have the permission, show the rationale, and request permissions.
4. Define a callback to receive the result of the request permissions.

It's actually a lot of boiler plate code, which I won't show here because the [android.developer.com](https://developer.android.com/training/permissions/requesting) has already done it.

## The easy way

This left me searching for an alternate way, and I found it in a small library called [Dexter](https://github.com/Karumi/Dexter) - contributed to open source (Apache 2.0 licensed) by [Karumi](http://karumi.com).  Basically, it simplifies the way you ask for permissions.

In my case, I need to ask for multiple permissions, so let's go through the process.  First, we declare the permissions in the `AndroidManifest.xml` file:

{% highlight xml %}
    <uses-permission android:name="android.permission.CAMERA" />
    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
{% endhighlight %}

Next, add the library to the app-level `build.gradle`:

{% highlight gradle %}
dependencies {
    // Other dependencies here
    implementation "com.karumi:dexter:6.0.0"
}
{% endhighlight %}

Don't forget to sync.

Finally, call Dexter in your app to check for licenses:

{% highlight xml %}
private fun validatePermissions() {
  Dexter
    .withActivity(this)
    .withPermissions(
      Manifest.permission.CAMERA,
      Manifest.permission.READ_EXTERNAL_STORAGE,
      Manifest.permission.WRITE_EXTERNAL_STORAGE
    )
    .withListener(object : MultiplePermissionsListener {
      override fun onPermissionsChecked(report: MultiplePermissionsReport?) {
        launchCamera()
      }

      override fun onPermissionRationaleShouldBeShown(permissions: MutableList<PermissionRequest>?, token: PermissionToken?) {
        Timber.d("Skipping rationale request in validatePermissions()!")
      }
    })
    .check()
}
{% endhighlight %}

We declare the list of permissions we want in the `.withPermissions()` and then add a callback that is called when it wants something.  In this case, once I've got all the permissions approved, I call `launchCamera()`.  If the permissions failed, then it drops through and returns - nothing actually happens.

There is another version of this that provides an `onFailure` callback for checking an individual permission.  This allows you, for example, to disable the camera button when you get a declined to prevent the user from clicking the button in the future.

You can see the code for this in my [GitHub repository](https://github.com/adrianhall/tailwind-photos/tree/7oct2019).
