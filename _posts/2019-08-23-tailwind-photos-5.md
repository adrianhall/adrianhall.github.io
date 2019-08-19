---
title: "Tailwind Photos: Twitter Login"
categories:
  - Android
  - "Tailwind Photos"
tags:
  - Kotlin
---

Thus far in our story, we've covered [Facebook]({% post_url 2019-08-17-tailwind-photos-2 %}), [Google]({% post_url 019-08-19-tailwinds-photos-3 %}), and [Microsoft]({% post_url 2019-08-21-tailwind-photos-4 %}) authentication.  There is one more to do - Twitter.  Unfortunately, Twitter doesn't have a nice vendor-provided SDK to do the work.  Twitter implements OAuth 2.0, though, so it's about as standard as it gets in terms of authentication mechanisms.

## The Twitter side of things

Shocker - you need to do some work on the vendor side of things first.  In this case:

* Go to the [Twitter developers console](https://developer.twitter.com/en/apps).
* Click **Create an app**.
* If you haven't already, you'll be prompted to apply for a developer account.

> Twitter has the most obnoxious requirements.  You cannot use the API in any form until they have approved your request.  This can take some time, so be patient - log off and go do something else for a couple of days.  The other platforms I integrated were more open to being used by developers.

