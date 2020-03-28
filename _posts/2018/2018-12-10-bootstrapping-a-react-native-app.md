---
title: "Bootstrapping a React Native App: A Comparison"
categories:
  - "React Native"
---

It's been a while since I have worked on a React Native app. In that time, `create-react-native-app` (CRNA) has been deprecated, and `expo-cli` has taken its place as the advised route. There is still `react-native init`, and Infinite Red has `ignite`. That is four different ways of bootstrapping a React Native app.

Before I continue, let me be clear on what I want:

* I want to use "best practices" in my app architecture - for state, communication with my AWS-based backend, navigation, and so on.
* I want to develop, debug, and test on emulators or real devices. My app will not be running inside a "walled garden".
* I want to be able to distribute my apps on the Apple App Store and Google Play Store.
* I want the bootstrap solution to be supported by the community or an organization that isn't going anywhere.
* It needs to "work with" my chosen IDE - Visual Studio Code.

Let's start with the obvious statement. All of the solutions for bootstrapping do the job that is proposed. It's really a case of what opinions they have when you bootstrap a project.

Note that throughout this article, I'm going to use the term "emulator" to indicate both simulators (via XCode for iOS) and emulators (via Android Studio for Android).

## The crowd opinion

Naturally, I took to Twitter to start the investigation:

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">So CRNA is dead, <a href="https://twitter.com/expo?ref_src=twsrc%5Etfw">@expo</a> CLI is new, but requires you to sign up.  How do folks like to bootstrap a <a href="https://twitter.com/reactnative?ref_src=twsrc%5Etfw">@ReactNative</a> app now?</p>&mdash; Adrian Hall (@FizzyInTheHall) <a href="https://twitter.com/FizzyInTheHall/status/1070386921623318528?ref_src=twsrc%5Etfw">December 5, 2018</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

It looks like a horse race between `react-native init` and `expo-cli`! After experiencing all of them, I can see why this is the case.

## react-native init

When you install the development environment for React Native, you probably installed this:

```bash
$ [sudo] npm install -g react-native-cli
```

That's it for setup. Now, how about bootstrapping?

```bash
$ react-native init RNIApp
```

Here, `RNIApp` is the name of the directory (and the app) that I am developing. When done, here is what the layout looks like:

![]({{ site.baseurl }}/assets/images/2018-12-10-picture1.png)

There are minimal opinions here and all are easily changeable:

* A single "App" entry-point (rather than one per platform).
* Jest for testing.
* Babel for compilation of ES2016 to JavaScript.
* Yarn for package management.

The Good Things:
* It's all set up for Android and iOS out of the box - no need to eject to compile.
* It's minimalist - there is very little opinion, and everything is easily changed.
* You can easily include native code.

The Bad Things:
* It takes a long time for the first compilation.
* If React Native updates the android/ios directories, your app won't get the changes.
* It isn't "expo" ready, so you have to use emulators to run the code.
* There is a lot of boiler-plate code that you will have to establish prior to actually getting a viable app.
* I've found that the VS Code debugging against emulators is a little problematic. Sometimes, the debugging session won't be able to connect to the emulator even though the app runs.

I would use `react-native init` if I was planning on adding a library that isn't supported in Expo. (AWS Amplify - one of my go to libraries for connecting to the AWS cloud - is supported by Expo).

## expo-cli

My next one to try was the `expo-cli`. This is the replacement for `create-react-native-app` (or CRNA). CRNA was maintained by the community and had no ties to another organization. Now that Expo has taken it over, they've got ties for publishing Expo apps and building on their backend. These are all, in the whole, good things. You don't need to sign up for an Expo account if you aren't using these things. That means if you were using CRNA, just use `expo-cli` instead and you won't lose functionality. 

Installing the `expo-cli` is as easy as all the others:

```bash
$ [sudo] npm install -g expo-cli
```

Then you can easily initialize a project:

```bash
$ expo init ExpoApp
```

This gives you a choice. Do you want a minimal setup (which is much like the CRNA option) or a better setup with tabs and react-navigation. I opted for the minimal setup to see what that looks like first:

![]({{ site.baseurl }}/assets/images/2018-12-10-picture2.png)

In the minimal configuration, there is very little opinion:

* Expo is the preferred method of running the app.
* Babel with an "Expo" set of presets is used.
* Yarn is preferred but not required for package management

The Good Things:
* It works with Expo.
* No android/ios directories for a simplified view of the app.

The Bad Things:
* Uses a "custom" version of react-native pulled from Expo rather than the standard version of react-native.
* The "entry point" is not App.js, but something in a node_module, which makes it more opaque.
* You need to eject to do native library development.

The "tabs" template gives you more stuff:

![]({{ site.baseurl }}/assets/images/2018-12-10-picture3.png)

This template adds `@expo/samples` as a dependency, which has no GitHub repository (so I can't check out the source code prior to downloading) and no README. I had to go and check out what was in it in the `node_modules` directory. It's not a particularly big nor important dependency, but the lack of transparency is disturbing. Other than this glaring point, it also adds react-navigation, which is fairly standard as the screen navigation solution.

The Good Things:
* It's given you a lot of structure right in the template.
* The fact that you have templates opens up the way for future different templates (in which case, why not just use Yeoman, but I digress).
* It's added testing with Jest.

The Bad Things:
* It doesn't explain what is in that samples package. I realize that it's just the content for two sample screens, but a README and GitHub repo would have been nice.
* It uses a custom version of react-native libraries. This is bad because it hides the version of react-native that you are using.
* It loads `Icon` from `expo` instead of `react-native-vector-icons`. I appreciate this is because it doesn't require linking, but it's yet another hard dependency with versioning problem to solve.

I'm sure there are good reasons for all of this, and I trust that the react-native version will be kept up to date.  Still, this is causing me more investigation to ensure I know what I am playing with.

In terms of "time" that will be saved in bootstrapping, the tabs template will save about a days worth of development time, but the blank template won't save anything.

## ignite

New (to me) is [ignite by Infinite Red](https://infinite.red/ignite). Installing this is as easy as the others:

```bash
$ [sudo] npm install -g ignite-cli
```

Then initialize the app:

```bash
ignite new IgniteApp
```

Gone is the minimal look - you get **Andross** (which includes React Navigation, Redux, and Redux Saga) or **Bowser** (which includes React Navigation, MobX State Tree and TypeScript). The ignite process for Bowser crashed. In addition, there were some quirks in the ignition process. I selected Andross because of these problems (even though I wanted the TypeScript support).

The ignition process also asks you about which libraries you want to use, with (in general) only one supported, so you get to include it or not. After all is done (and it takes a long time), this is what the file structure looks like:

![]({{ site.baseurl }}/assets/images/2018-12-10-picture4.png)

Pre-installed dependencies (not including the libraries that you asked for during the ignition process) include:
* apisauce (Axios plus request/response transforms)
* ramda (A functional library)
* redux (with redux-persist and redux-saga and reduxsauce)
* Storybook (a development environment for UI components)
* enzyme (testing utilities for React)
* snazzy (stylish linter output)
* reactotron (remote control for your app)

That's a lot of boiler plate. In addition, a lot of the structure of the app is taken care of for you.

The Good Things
* There are a lot of wiring boilerplate done for you.
* You will probably knock off 2+ weeks of development time off your project by using ignite IF you want to use the environment that is set up
* It is set up as if it were "react-native init" with a template, so it's easy to convert out to do native development.

The Bad Things
* There is so much done for you that you may find it difficult to change the things you don't like.
* The ignite CLI seems to be very buggy.
* There are only two templates, which means not a lot of selection for the things you want to control.

## So, what will I use?

I'm a believer in understanding what goes into the code you write.  As a result of that, I prefer to not use templates that do a lot.  I might use the tabs template from `expo-cli`, but that would be as much template as I would want.

I'm more likely to look at `react-native init` or` expo-cli` with the blank template.

In terms of my debugging, etc. situation, relying on expo early on seems like a good idea.  However, I have concerns (which I should probably just get over) as to that react-native library stored on GitHub.  That being said, I can always eject from the expo version and revert to `react-native init` form.  As a result of all that, I'm going to go with the `expo-cli` version.

