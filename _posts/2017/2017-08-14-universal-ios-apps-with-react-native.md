---
title: "Universal iOS Apps with React Native"
categories:
  - iOS
  - "React Native"
---

This is one of those short “tip” posts.

I was developing a nice React Native app and blogging about it. However, I bumped into a problem where my iOS simulator runs always appeared as if they were on an iPhone. If I ran the app on an iPad, it would still act as if it were on an iPhone.

The fix for this is buried inside XCode. Open XCode and then open the .xcodeproj file, which is inside the ios folder of your project. Then click on the project name. You will see a section like this:

![]({{ site.baseurl }}/assets/images/2017/2017-08-14-image1.png)

Note the **Devices** section. It’s set to iPhone. It doesn’t matter what you do in the JavaScript code. This code will always run as if it is on an iPhone. Change the **Devices** to **Universal**, then close Xcode and re-build your React Native app for iOS.

When you run the app on the simulator, it will now act like an iPad on the iPad and an iPhone on the iPhone. Everything is right in the world.

You might be wondering why this isn’t the default setting! Yeah – me too.
