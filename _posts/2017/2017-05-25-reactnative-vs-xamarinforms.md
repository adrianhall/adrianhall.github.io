---
title: "Which is better - React Native or Xamarin Forms?"
categories:
  - "React Native"
  - Xamarin
tags:
  - Comparisons
---

Let’s talk about a loaded question.  After my recent forays into both React Native and Xamarin Forms, I got asked on Twitter – which is better, React Native or Xamarin Forms?  Further, I should answer this for JavaScript experts, C# XAML experts and for developers with experience in both.  After all, both produce native apps and both use a common codebase.

## The Short Version

There isn’t a clear winner here.  If I am a JavaScript expert, I’ll gravitate towards React Native.  If I’m a C# expert, I’ll gravitate towards Xamarin Forms.  If I’m equally experienced in both (which is unlikely), then I’m likely to use React Native for everything but Microsoft-centric projects.

## The Long Version

There are many facets that one must consider when starting a mobile project.  Here are the ones I looked at:

### Legal Requirements

If I am producing a commercial product, then the open source components that I use must not be encumbered or there must be a commercially friendly license available.  Xamarin Forms meets these requirements.  It’s [licensed under the MIT license](https://github.com/xamarin/Xamarin.Forms/blob/master/LICENSE) and you get a permissive license to use Xamarin Forms when you purchase an MSDN license.  It really can’t get better than that.  React Native, on the other hand, has an [awkward patent clause](https://github.com/facebook/react/blob/master/PATENTS) that makes it [less than desirable](https://shellmonger.com/2016/08/18/follow-up-to-why-im-leaving-react-behind/) in commercial products.  I would not use React or React Native in commercial products without some solid legal advice.

> **UPDATE (2018-02-19)** Facebook has removed the PATENTS requirement from both the React and React Native code base.  This has removed any hesitation I have had in the past about recommending React Native.

Winner: A Tie

### Framework

Xamarin Forms pretty much uses MVVM as a framework style.  React Native uses the more modern one-way data flow that React provides.  Personally, I find myself more productive in the React style than I am in the MVVM style.  However, this is marginal.  I suspect that if you are already familiar with modern JavaScript web development, then you will like React style.   If you are an ASP.NET MVC, Web Forms, or other MV* developer, you will prefer Xamarin Forms.

Winner: A Tie

### Development Environment

Xamarin Forms requires Visual Studio, which is a well respected IDE, but it’s only one possibility.  I know that someone will correct me and say you don’t HAVE to use Visual Studio, but let’s be realistic.  If you are developing Xamarin Forms, you are going to be using Visual Studio.  React Native, on the other hand, can use pretty much any text editor.  The command line tools are written in node, so you can use whatever platform you want and whatever editor you want.  This flexibility is awesome

Winner: React Native

### Cross-Platform Development

You can develop Xamarin Forms apps on a PC and compile on a Mac easily using an agent.  This allows you to “rent” a Mac in the cloud when you are building your apps, so you don’t need to own one.  Even if you do go for owning one, you can own a small Mac Mini to get the job done.  You can also run the iOS Simulator via a proxy and display it on the PC.

React Native doesn’t have the same experience.  You can (and should) use Expo for the majority of your coding.  Expo is an app that you run on your phone (Android or iOS).  However, if you need to build an iOS app, you still need to do it from a Mac (which can be yours or in the cloud) or as a build service (from the cloud).  You can use command line tools, but the experience is not as seamless as the Xamarin Forms experience.

Winner: Xamarin Forms

### Ready Made Components

I found pretty much equal quantity of quality components and libraries in both Xamarin Forms and React Native.  The Xamarin Forms ones were split between NuGet and the component store, and the documentation for them was, in general, worse.  The React Native ones were all in one place – npm – and the documentation was, in general, better.

Winner: React Native

### Time to Productive

If I am starting from a blank mac with Android Studio and XCode already installed, and I want to produce my first app, the installation process for Xamarin Forms (installing Visual Studio, downloading the components, File -> New Project and a couple of screens) is several hours.  I completed the same process in around 3 hours for React Native.

> **UPDATE (2018-02-19)** I use Expo and create-react-native-app primarily these days.  The “time to productive” was about 15 minutes.  React Native was the winner before and it just got better.

Winner: React Native

### Expert Assistance

If I look for assistance on the Internet – Stack Overflow, blogs, etc. – then I am more likely to find an answer for Xamarin Forms.  It’s been around longer and there is a community of MVPs to assist.  React Native is younger.  There is still a lot of support out there, especially for beginners.  However, you won’t find the depth of knowledge as readily available.

Winner: Xamarin Forms

### Type Safety

Xamarin Forms uses C#.  It has type safety by default.  React Native uses JavaScript.  You can use TypeScript to get type safety at compile time, but it’s an optional extra.  I actually like this feature of React Native.  People coming from a static type language like C# or Java likely won’t like this feature.

Winner: React Native (for me)

### Testing Capabilities

You can do UI testing in either platform.  I found unit tests to be easier to write in Jest than XUnit.    One big difference I found was that I could debug in the XCode Simulator for iOS when using Xamarin Forms.  I could only debug in the Android Emulator on React Native.  Even there, I had many issues with the emulator.  This is obviously an area that is evolving, so I expect the answer may change.

> **UPDATE (2017-10-31)** This has improved considerably.  Check out the Visual Studio Code debugging capabilities for React Native.    I’m impressed with React Native now

Winner: A Tie!

In the end, the choice is up to you.  There is no clear winner and it will depend on your individual requirements as to which one to select.  My preference right now is for React Native.
