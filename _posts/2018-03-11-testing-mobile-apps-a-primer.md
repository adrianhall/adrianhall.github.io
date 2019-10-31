---
title: "Testing Mobile Apps: A Primer"
categories:
  - Mobile
tags:
  - Testing
---

Why should you test your mobile app? A [recent study](http://info.localytics.com/blog/23-of-users-abandon-an-app-after-one-use) showed that almost a quarter of users only use a mobile app once, and a shocking 95% abandon an app within the first month. Some of the reasons that users abandon an app are due to content and engagement. The biggest non-content reasons for abandonment are application crashes and security concerns. No one can prevent all application crashes. The mobile ecosystem is too broad and unpredictable to provide a 100% guarantee. However, testing ensures that your mobile app has been stressed on as many devices as possible , which enables you to identify and fix bugs early.

## Types of mobile testing

There are seven types of scenarios that you should consider when testing a client-server application such as a mobile app:

* Unit testing
* UI testing
* Fuzz testing
* Performance testing
* End-to-end testing
* Pre-production testing
* Canary (post-production) testing

Tests are organized into test suites — sets of tests that together test the entire functionality of your app. Let’s look at each of these types.

### Unit testing

Unit testing tests individual parts of your code for correctness, usually using an automated test suite. A good unit test assures you that the functionality that you are expecting out of the unit (your code; usually a single method) is correct. A good unit test is:

* Repeatable&mdash;you can run it several times and it produces the same result.
* Fast&mdash;you don't want it to interrupt the flow of your work.
* Readable&mdash;you can understand what the test is doing.
* Independent&mdash;you can run a single test from a suite.
* Comprehensive&mdash;you have enough tests to test all the main cases and corner cases of inputs and cover all code paths.

A good unit test suite augments the developer documentation for your app. This helps new developers come up to speed by describing the functionality of specific methods. When coupled with good code coverage, a unit test acts as a safeguard against regressions. Unit tests are important for anything that does not produce a UI. For example, consider a Notes app that has a set of notes stored in a SQLite database on a mobile device. For this app, there is a class that accesses the notes and a set of classes that implement the UI. The class that accesses the notes has unit tests.

Each platform has its own unit testing framework:

* Android Studio has [JUnit](https://developer.android.com/training/testing/unit-testing/local-unit-tests.html) built in.
* XCode has the [XCTest framework](https://developer.apple.com/library/content/documentation/DeveloperTools/Conceptual/testing_with_xcode/chapters/01-introduction.html).
* Ionic uses the [Karma](https://karma-runner.github.io/1.0/index.html) and [Jasmine](https://jasmine.github.io/) testing frameworks.
* React Native uses the [Jest](https://facebook.github.io/jest/docs/tutorial-react-native.html) testing framework.

Each of these frameworks supports all of the features you need to start testing, including support for async functionality (a common pattern in mobile apps). Each testing framework has its own appearance, but in essence all testing frameworks operate the same. You specify some inputs, call the method to test, and then verify that the output is what you expect it to be.

### UI testing

UI testing takes a flow that the user might follow and ensures it produces the right output. It can be done on real devices or on emulators and simulators. Given the same state (including backing stores and starting point), the UI test always produces the same output. A UI test can be considered similar to a unit test. The input to the test is the user clicks and the output of the test is the screen. UI testing is more closely associated with the device than with the platform language. There is a preferred testing framework and each framework has a test recorder. This enables you to record a UI flow on one device and then replay the test on many other devices. The test recorder allows you to get productive quickly:

* Android uses [Espresso](https://google.github.io/android-testing-support-library/docs/espresso/) and the [Espresso Test Recorder](https://developer.android.com/studio/test/espresso-test-recorder.html).
* iOS uses [XCUITest and the XCode UI Test Recorder](https://developer.apple.com/library/content/documentation/DeveloperTools/Conceptual/testing_with_xcode/chapters/09-ui_testing.html).

In addition, there are cross-platform UI test frameworks to consider, primarily [Appium](http://appium.io/) and [Calabash](http://calaba.sh/). With Appium, you can choose a language. This is useful if you are writing Ionic or React Native apps because you can use JavaScript as the testing language. Calabash requires that the tests are written in Ruby. Neither of these frameworks provide a recorder to aid in writing tests. Both frameworks are open source projects.

Finally, [AWS Device Farm] has a feature called Explorer for Android apps. This feature investigates your UI and finds things to interact with. When it finds a control, it interacts with the control. This feature works with login screens, allowing you to validate authenticated sessions as well.

![Screenshot of AWS Device Farm]({{ site.baseurl }}/assets/images/2018-03-11-image1.jpg){: .center-image}

Ionic and React Native perform UI testing against compiled code, so your UI tests should not include Ionic or React Native code. Instead, you are testing how the app acts on a device.

> Apps are used by humans and humans don't always do the expected thing

### Fuzz testing

Unit tests and UI tests are used to ensure that the expected output happens when the expected input is used. However, apps are used by humans and humans don’t always do the expected thing. For that case, you can introduce a random stream of events into your app and see what happens. This is known as monkey testing or fuzz testing. It’s a stress test that simulates what happens when the user randomly presses the screen, for example. This is similar to a UI test. However, you don’t need to write any tests since random events are created.

* For Android, try [monkeyrunner](https://developer.android.com/studio/test/monkeyrunner/index.html).
* For iOS, try [UI AutoMonkey](https://github.com/jonathanpenn/ui-auto-monkey).

It is also interesting to record a fuzz test run. It can be replayed later to reproduce any issue found, verify that the issue was fixed, and to scale testing across more devices. Fuzz testing takes off when you take testing to the cloud with [AWS Device Farm], allowing you to test on many more devices than you may have access to.

### Performance testing

Gather performance metrics while running the UI and fuzz tests to ensure that your application does not take up significant resources on the device. Such performance metrics include:

* Battery drain and energy usage.
* Appropriate usage of the GPS and other features that drain battery.
* Network bandwidth usage.
* Memory usage.

This data can be gathered during development using a profiling tool such as the [Android Monitor](https://developer.android.com/studio/profile/android-monitor.html) (built into Android Studio) or [Instruments](https://developer.apple.com/library/content/documentation/DeveloperTools/Conceptual/InstrumentsUserGuide/MeasuringCPUUse.html#//apple_ref/doc/uid/TP40004652-CH84-SW1) (built into XCode). During this process, you can also use network shaping, which simulates lower bandwidth network connections (for example, 3G connections) or spotty Wi-Fi connections. This enables you to experience adverse network conditions as your users would and ensure an appropriate user experience. If you are developing a mobile game, you can also measure the FPS (frames per second) for your mobile game when the device is stressed (low memory or restricted network capabilities).

For performance testing during the test phase (and beyond), use application performance monitoring (APM) tools such as [New Relic Mobile](https://newrelic.com/mobile-monitoring) or [Splunk MINT](https://www.splunk.com/en_us/solutions/solution-areas/application-delivery/mobile-intelligence.html).

### Integration (end-to-end) testing

After you test your mobile app in isolation, you test it with a mobile backend. This is generally known as integration testing or end-to-end testing. Take the same UI and fuzz testing that we already described, and then recompile your code with the live cloud backend. Many organizations produce a separate test environment for this purpose.

If you are upgrading the mobile app (rather than releasing a new app) and the upgrade involves an upgrade to the backend resources — the database schema or API responses — then you also should test the upgrade scenarios during integration testing. Users do not upgrade their mobile apps immediately, so multiple versions of your app will use the same cloud backend resources (even if only the database is affected).

If you are using a third-party cloud service (for example, a Weather API), then make sure you check their rules for throttling. Test failures can result if the third-party cloud service detects you are making too many API calls.

### Pre-launch or pre-submission testing

You are just about to launch your app to the public. You’ve done all the appropriate testing on the latest devices and OS versions. Run the UI and fuzz tests on as wide a pool of devices as you possibly can. There is one final test before submitting your app to the app store. You want to test your app on as large a community of devices and OS combinations as you possibly can.

An enterprise usually has the luxury of dictating a support matrix of devices. If you are producing a mobile app for the general public, then the situation is a little more complex. [OpenSignal estimates that there are over 24,000 distinct Android device types](https://opensignal.com/reports/2015/08/android-fragmentation/) in the world. Over 18,000 of these were used in the last year. The [statistics from Google](https://developer.android.com/about/dashboards/index.html) indicate that API level 23 (which is a couple of years old at this point) has only reached 58% of consumers. The iOS device types are a little more limited — there are only a handful of models, and iOS devices tend to be kept up to date. That’s still numerous device / OS combinations.

One option is to maintain a device farm of your own, buying one of each device. However, that is expensive to buy and maintain. A better option is to rent the devices you need for the pre-production run and let someone else worry about maintaining the latest device and OS versions. [AWS Device Farm] helps with this problem as well, running your UI and fuzz tests across a wide variety of devices and charging only for what you use.

### Canary (post-production) testing

After your app is in production, use a canary test to ensure that the mobile app and the backend are running harmoniously. A canary test is a set of UI tests that run using the same mobile app that you distributed to your users and using the same (production) services on the backend.

You can generate a (different, smaller) test suite for an individual (canary) user and run this test suite on [AWS Device Farm](https://aws.amazon.com/device-farm/) on a schedule to implement canary testing.

## Best practices in testing

You don’t need to run all tests all the time. The following are some best practices:

* Architect your application so that the mobile application and backend can be tested independently. Compile your app with “stub” methods that simulate the cloud services. Also, try mock cloud services with frameworks like [Mockito](http://site.mockito.org/) (Android) or [Cuckoo](https://github.com/Brightify/Cuckoo) (Swift).
* All changes to the code base of your mobile app or backend should include appropriate unit tests or UI tests to check the new functionality.
* Run unit tests with every build and UI tests on an emulator or simulator before checking in your code change.
* Run a complete set of UI and fuzz tests on a pre-defined set of the most popular devices for your users on a regular basis. It is ideal to do this as part of a continuous integration pipeline (for example, using Jenkins). At minimum, you should run these tests on a nightly basis. Monitor for application crashes and test failures.
* Run your UI tests on as many devices as possible at intervals throughout the development process. At minimum, run a full set of tests on as many devices as possible before release. Any test failures or application crashes should generate bugs to be fixed by the engineers.
* Include enough information in your bugs to reproduce the failure. Include the test case, device and OS combination, whether the test was against the cloud services or the stubs, and video or screen captures of the test failure.
* Analyze the test results. A single test failure is useful. Knowing that the same error or crash occurred on a set of devices with a common characteristic is more useful to the developers who need to diagnose the problem.

[AWS Device Farm] is a mobile app testing service that lets you test and interact with your Android, iOS, and web apps on many devices at once. It enables you to capture video, screenshots, logs, and performance data to pinpoint and fix issues before shipping your app. You can use it to automate a large portion of the suggested mobile app testing capabilities described in this article.

With these best practices, you are on your way to producing a quality product that can be enjoyed by the maximum number of users.

{% include links.md %}
