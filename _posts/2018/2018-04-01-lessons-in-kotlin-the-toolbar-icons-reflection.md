---
title: "Lessons in Kotlin: Toolbar Icons and Reflection"
categories:
  - Mobile
tags:
  - android
  - kotlin
---

There are many tutorials online on how to produce an Android app bar with an options menu — so much so that it can be boiled down to a few steps, and I’ll reproduce them here:

## Step 1: Create resources

You actually need two resources. The first is a menu:

{% highlight xml %}
<?xml version="1.0" encoding="utf-8"?>
<menu xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto">
    <item
        android:id="@+id/mainActionCamera"
        android:icon="@drawable/ic_camera_48dp"
        android:title="@string/mainActionCamera"
        app:showAsAction="ifRoom"/>
</menu>
{% endhighlight %}

Notice that I include an icon.  This is the second resource.  I create it by making a vector icon in the `drawable` resource directory with a 48x48dp size. I also need a string resource for the title. The `app:showAsAction` takes one of several values. The important ones here are:

* `always` will always put an icon on the toolbar
* `never` will always put a menu item in the overflow options menu
* `ifRoom` will put an icon on the toolbar if it fits, but otherwise place it in the overflow options menu.

## Step 2: Add a toolbar to your layout

I’m using ConstraintLayout these days, so this is my entry at the top of my layout:

{% highlight xml %}
<android.support.v7.widget.Toolbar
    android:id="@+id/mainToolbar"
    android:layout_width="match_parent"
    android:layout_height="?attr/actionBarSize"
    android:background="?attr/colorPrimary"
    android:elevation="4dp"
    android:theme="@style/ToolbarTheme"
    app:popupTheme="@style/ThemeOverlay.AppCompat.Light"/>
{% endhighlight %}

I also create a style and modify my main app theme:

{% highlight xml %}
<style name="AppTheme" parent="Theme.AppCompat.Light.NoActionBar">
    <item name="colorPrimary">@color/colorPrimary</item>
    <item name="colorPrimaryDark">@color/colorPrimaryDark</item>
    <item name="colorAccent">@color/colorAccent</item>
</style>
<style name="ToolbarTheme" parent="ThemeOverlay.AppCompat.Dark.ActionBar">
    <item name="android:textColorSecondary">@android:color/black</item>
</style>
{% endhighlight %}

I match the parent of the `ToolbarTheme` to the color of the background. if the background is dark, then I use `ThemeOverlay.AppCompat.Dark.ActionBar`. If the background is light, then I use `ThemeOverlay.AppCompat.ActionBar`. This ensures the title of the page on the action bar is the right color. My options menu is always a light background and I set the appropriate text color in the theme.

## Step 3: Add code to your activity

You need to add a single line to the `onCreate()` fun:

{% highlight kotlin %}
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_main)
    setSupportActionBar(mainToolbar)
}
{% endhighlight %}

Also, add an `onCreateOptionsMenu()` fun:

{% highlight kotlin %}
override fun onCreateOptionsMenu(menu: Menu?): Boolean {
    menuInflater.inflate(R.menu.main_menu, menu)
    return true
}
{% endhighlight %}

Run this and you have a collapsible menu. Change the `app:ShowAsAction` between `always` and `never` to see both states.

> What if you want the icon to appear next to the menu item? Material Design says this is a big no today. Later APIs have removed the ability to enable icons next to menu items. Thanks Google!

## Step 4: Customize the toolbar

We could go completely with a completely custom toolbar. After all, the toolbar is just a container with some default content. Replace the content and you can do whatever you want. However, the underlying capabilities like displaying a menu with icons are still in the code. They are just hidden so we can’t see them. We can use reflection to expose them.

So, let’s display the icons. The `onCreateOptionsMenu()` fun is called only once when the activity is started. The `onPrepareOptionsMenu()` fun is called whenever it is needed, so it handles typical things that may adjust the menu (like orientation changes). This makes it an appropriate place to put out adjustments.

Put a breakpoint on the `onCreateOptionsMenu()` fun and inspect the `menu` object that is passed to the fun. Note that it is actually a `MenuBuilder` object. The variable `mOptionalIconsVisible` within the `MenuBuilder` class determines if the icons are visible. We can make that variable accessible using the built-in reflection capabilities:

{% highlight kotlin %}
override fun onPrepareOptionsMenu(menu: Menu?): Boolean {
    menu?.let {
        if (menu is MenuBuilder) {
            try {
                val field = menu.javaClass.getDeclaredField("mOptionalIconsVisible")
                field.isAccessible = true
                field.setBoolean(menu, true)
            } catch (ignored: Exception) {
                // ignored exception
            }
        }
    }
    return super.onPrepareOptionsMenu(menu)
}
{% endhighlight %}

I guess you could put this in a pair of extension methods to `MenuBuilder`, but it’s only used in this one place, so why bother? Set the `app:ShowAsAction` value to never in your menu, then run the app. You will have icons, but the icons will be white (because that’s what goes on the toolbar). If your toolbar has a dark background and the menu has a light background, you need to alter the tint of the icon.

Fortunately, altering the tint of the background is easy. I have an extension function on `Drawable` to do this:

{% highlight kotlin %}
import android.graphics.PorterDuff
import android.graphics.drawable.Drawable

fun Drawable.setIconColor(color: Int) {
    mutate()
    setColorFilter(color, PorterDuff.Mode.SRC_ATOP)
}
{% endhighlight %}

Now I can write something like:

{% highlight kotlin %}
menuItem.icon.setIconColor(Color.BLACK)
{% endhighlight %}

and it will produce a black icon instead of a white icon.

The next problem is that there is no standard way of determining if the icon will be shown as an action (on the toolbar) or in the menu. When Google deprecated the ability to show icons on the menus, they also removed the ability to detect whether a menu item will be an action or not. (Thanks again Google!)

As before, there is a variable — this time on the `MenuItem`. But it’s hidden. I use the following extension method to expose it:

{% highlight kotlin %}
import android.view.MenuItem

fun MenuItem.getShowAsAction(): Int {
    var f = this.javaClass.getDeclaredField("mShowAsAction")
    f.isAccessible = true
    return f.getInt(this)
}
{% endhighlight %}

`getShowAsAction()` returns 1 if it is shown as an action or 0 if it is in a menu. Now I can use this method to update the menu. Here is my final `onPrepareOptionsMenu()`:

{% highlight kotlin %}
override fun onPrepareOptionsMenu(menu: Menu?): Boolean {
    menu?.let {
        if (menu is MenuBuilder) {
            try {
                val field = menu.javaClass.getDeclaredField("mOptionalIconsVisible")
                field.isAccessible = true
                field.setBoolean(menu, true)
            } catch (ignored: Exception) {
                logger.debug("ignored exception: ${ignored.javaClass.simpleName}")
            }
        }

        for (item in 0 until menu.size()) {
            val menuItem = menu.getItem(item)
            menuItem.icon.setIconColor(
                if (menuItem.getShowAsAction() == 0) Color.BLACK
                else Color.WHITE
            )
        }
    }
    return super.onPrepareOptionsMenu(menu)
}
{% endhighlight %}

## Wrap Up
Firstly, thank you Google for making this stuff difficult. It’s almost forcing us to go hunting for a new library. It’s well worth taking a look at some of these libraries. However, using reflection will fix the problems with the current toolbar. A new library will increase the size of your binary, so it’s not without cost.

Using reflection isn’t without its problems. The most major problem is that Google is signalling that these APIs will go away in the future, so your app could break with any new release. Set a maximum API level and test as each new release of Android comes out.

