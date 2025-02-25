---
title:  "The State of JavaScript Testing Frameworks 2024"
date:   2024-08-03
categories:
  - Devtools
tags:
  - JavaScript
  - TypeScript
  - Comparison
---

This week, I am adding a test framework to my ESM module template. I use a testing framework for both [unit testing](https://www.geeksforgeeks.org/unit-testing-software-testing/) and [integration testing](https://www.geeksforgeeks.org/software-engineering-integration-testing/). Like everything, test frameworks have evolved, and no test framework is suitable for all situations.  However, it is one of those things you have to do if your project is destined to be long-lived.  You should get into the habit of adding tests even if your project is not going to be long-lived.  It's a good habit.

The words you want to look for are "test orchestrator" or "test runner".  A testing framework consists of three parts:

1. A test runner - something that you run in order to get the testing framework to execute each test.
2. An assertion library - something that allows you to check results to ensure that the test passes.
3. A test reporter - something that consolidates and displays the results from the multiple tests

You also want to integrate with a code coverage library - something that shows how much of your library you are executing.

So, what are my options?  Let's look at my comparison table:

| Library        | Test runner      | Assertions       | Reporter         | TypeScript       | ESM              | Unit tests       | UI tests         |
|:---------------|:----------------:|:----------------:|:----------------:|:----------------:|:----------------:|:----------------:|:----------------:|
| [CucumberJS]   |:white_check_mark:|:white_check_mark:|:white_check_mark:|:x:               |:white_check_mark:|:x:               |:x:               |
| [Cypress]      |:white_check_mark:|:white_check_mark:|:white_check_mark:|:x:               |:x:               |:white_check_mark:|:white_check_mark:|
| [Jasmine]      |:white_check_mark:|:white_check_mark:|:white_check_mark:|:x:               |:white_check_mark:|:white_check_mark:|:white_check_mark:|
| [Jest]         |:white_check_mark:|:white_check_mark:|:white_check_mark:|:white_check_mark:|:x:               |:white_check_mark:|:x:               |
| [MochaJS]      |:white_check_mark:|:x:               |:white_check_mark:|:x:               |:white_check_mark:|:white_check_mark:|:x:               |
| [NightwatchJS] |:white_check_mark:|:white_check_mark:|:white_check_mark:|:white_check_mark:|:white_check_mark:|:x:               |:white_check_mark:|
| [Playwright]   |:white_check_mark:|:white_check_mark:|:white_check_mark:|:white_check_mark:|:white_check_mark:|:x:               |:white_check_mark:|
| [Vitest]       |:white_check_mark:|:white_check_mark:|:white_check_mark:|:white_check_mark:|:white_check_mark:|:white_check_mark:|:x:               |

Looking for a TL;DR?  

* Use [Vitest] for testing API, CLI, and library projects.
* Use [Playwright] for testing UI browser projects.
* Use [Istanbul] for code coverage.

## What should I look for in a testing framework?

The most important things are in the table above.  But there are some others:

* Does your test runner support ES Modules out of the box?  I've found that if the test runner doesn't support ESM, then it's painful to set up and maintain.
* Does your test runner allow you to write tests in TypeScript?  This isn't a requirement, but if you want to write everything in TypeScript, this is a consideration.
* Do you need to set up spies, stubs, or mocks? In any reasonable project, this is a requirement.  Fortunately, you can always use [sinon](https://npmjs.com/package/sinon) if your test runner doesn't support it out of the box.
* Do you need to test async code? It's likely, and your test runner needs to support it.
* Can you easily run UI tests on a headless browser?  If you are writing a library, this is not important.  If you are writing a browser app, it's required.
* Does it integrate with the Visual Studio Code test runner?  If it doesn't, you'll be running your tests outside of your normal IDE.  This is also useful when you need to attach a debugger to a specific test and run just that test to step through the code.

Note that you should always write your tests to import the transpiled code.  If you are working in TypeScript, it's really easy to fall into the trap of importing your TypeScript module into the test file instead of the compiled JavaScript version.  The problem there is that you aren't testing what your users are going to be using.  You want to be testing the code that your users will be running, not the pre-compiled code.

Most of these frameworks support a "standard" form for writing tests.  This follows the form:

{% highlight js %}
import { something } from '../dist/something.js';
import { expect } from 'chai';

describe('it should do something', () => {
    const x = something();
    expect(x).to.equal('it');
});
{% endhighlight %}

The `expect(x).to.equal('it')` is an assertion.  There are three basic flavors of assertions:

{% highlight js %}
// Assert style
assert.equal(x, 'it');

// Expect style
expect(x).to.equal('it');

// Should style
x.should().equal('it');
{% endhighlight %}

Most people like one form and dislike the others. It's a good idea for your tests to be consistent, so pick whichever you want to use and stick with it.

## Diving into the testing frameworks

Let's do a quick dive into each framework I'm covering.

### CucumberJS

CucumberJS is not a package I have personally used.  As such, a lot of this information is ripped from their website.  However, I would be remiss if I didn't mention the top test runners.

First off, CucumberJS does NOT use the standard form for tests.  Most test runners in this list use the standard form of tests.  Instead, CucumberJS is using behavior driven testing.  A test might look like this:

{% highlight text %}
Feature: Is it Friday yet?
  Everybody wants to know when it's Friday

  Scenario: Sunday isn't Friday
    Given today is Sunday
    When I ask whether it's Friday yet
    Then I should be told "Nope"
{% endhighlight %}

This is called a scenario and it maps directly to what you want to happen in plain language (technically, it's [Gherkin](https://cucumber.io/docs/gherkin/), but it's still readable as english.)  This change in testing focus makes it really hard to compare to other test runners.  I can group all the unit test runners together, and I can group all the UI test runners together.  Where do I put this one?  It's kind of out on its own island.

On the technical side, it requires the use of ts-node for TypeScript support, but does support ES modules.  Since you aren't actually writing tests in TypeScript, this is probably not a big deal.  Just test the JavaScript output (which is what you should be doing anyhow).

Does that mean its not worthy of consideration.  I suspect there are specific scenarios where you would use CucumberJS in conjunction with a UI test runner like Playwright to write specific behavioral tests.  That means it's "in addition to", not "instead of".  It's probably not of much use to my API/library focused requirements.

### Cypress

Cypress is both a UI-based (electron-based) app and a cloud service.  I'm going to ignore the (for-pay) cloud service for the purposes of this comparison.  The app is open source.  Since it is an app, it doesn't work on every platform (although it is supported on the majority of platforms that developers actually use).  It can handle end-to-end testing, component testing (where you test individual components of a UI application to ensure they produce the right HTML/CSS and interactivity), and unit tests.  It supports TypeScript out of the box, and I had no problem running ESM tests either. Code coverage is supplied by a [Istanbul], which is pretty standard.

It has its own helper library and has a built-in "expect" style assertion library.  You have to use the Cypress library extensively.  This means that tests written in Cypress will need a major porting project if you want to replace Cypress with something else.

While I like the fully featured capabilities of Cypress, I don't like it for it's standalone nature. I need tests to run in three places - in my CI pipeline, on the command line, and in my IDE.  I don't want "yet another UI tool" competing for desktop space. You can set up launch settings to launch the cypress app, but it's difficult to set it up for debugging.  This makes Cypress relatively low down on my list.

### Jasmine

Jasmine is a long time contender in the testing space and was an improvement over MochaJS because it integrated browser testing. It supports the standard form of tests and "expect" style assertions. This means that tests are easily written and transportable to other test frameworks if you bump into an unresolvable wall with Jasmine.  It supports ES modules out of the box, but does not support TypeScript tests.  You will need to use ts-node or tsx to pre-compile TypeScript tests. Code coverage is supplied by a [istanbul](https://istanbul.js.org/), which is pretty standard.

There is a [Visual Studio Code Test Adapter](https://marketplace.visualstudio.com/items?itemName=hbenl.vscode-jasmine-test-adapter) for Jasmine, which means you can run your tests easily either on the command line, in a CI pipeline, or within the IDE with debugging.  This is a major benefit of Jasmine over Cypress (but in common with others on the list).

### Jest

Jest was kind of the spiritual successor to MochaJS and was released to be a "batteries included" test runner. Jest is definitely more opinionated than MochaJS and just works out of the box while still being flexible. Jest supports the same standard form of tests and expect style assertions.  It works explicitly with TypeScript through babel or an alternate ts-jest command line.  It does not support ES modules out of the box (and it is painful to make it work).  Code coverage is provided by [Istanbul].  There is a [Visual Studio Code Test Adapter](https://marketplace.visualstudio.com/items?itemName=kavod-io.vscode-jest-test-adapter), so it works with Visual Studio Code debugging as well.

Jest pairs really well with the React Testing Library (this was where I first encountered Jest).  Combined with the React Testing Library, you can test individual React components from the user side of things really easily.  Jest also supports snapshots.  You can say "this output must match this snapshot", then only update the snapshot when you know the output has been validated.  This is a great feature for validating content generally.

With the current state of JavaScript (and specifically the move over to ES modules), the lack of support for ES modules in Jest is the major glaring problem.

### MochaJS

MochaJS has been around almost forever.  Unlike most of the frameworks on this list, this one does not come with an assertion library. Instead, you get to pick an assertion library. I recommend picking [chai] for this as it is a super flexible assertion package that will allow you to write tests the way you want.  It also doesn't have in-built support for spies, mocks, and stubs - another common requirement for testing.  I recommend [sinon](https://npmjs.com/package/sinon) for this.  There is a [test runner for Visual Studio Code](https://marketplace.visualstudio.com/items?itemName=hbenl.vscode-mocha-test-adapter)

So, where are the problems.

* It does not support TypeScript out of the box.  You need to set it up with [tsx] if you want to write tests in TypeScript.
* MochaJS cannot act as a test runner for UI tests without some additional help (which is usually provided by Puppeteer).

The number of packages that you have to add to make MochaJS "functional" is a problem.  This gave rise to more "batteries included" style test runners like Jest and Jasmine (and more recently, Vitest).

### NightwatchJS

(Direct from the web site): Nightwatch.js is an integrated framework for performing automated end-to-end testing on web applications and websites, across all major browsers. It is written in Node.js and uses the W3C WebDriver API to interact with various browsers.

Well, that sounds nice.  It's one of four frameworks that supports UI testing and it definitely excels at that.  It supports all the major web browsers and mobile web.  It has a [Visual Studio Code test runner](https://marketplace.visualstudio.com/items?itemName=browserstackcom.nightwatch).  It also integrates with other test runners like MochaJS and CucumberJS.  This means that you can add UI testing to those frameworks that don't support it.

It also relies on a third party assertion library.  They lean heavily on [chai] in their demos, which is a solid choice.  Since it is geared towards browser and mobile web testing, it runs the tests in that environment.

### Playwright

It's apropos that Playwright comes right below NightwatchJS in the list as these two go head-to-head in target audience, functionality, and capabilities.  Playwright uses browser debugger APIs to communicate with the browser, whereas NightwatchJS uses a web driver.  That means Playwright does not support mobile, whereas NightwatchJS can support mobile web.  Playwright supports mobile web testing via emulation instead.  Is that better?  I've always found that emulations do 95% of the job, but that 5% thats missing is generally important for complex apps.

UI tests do have their own semantics - how do you go to a page, click on a button, and so on.  Just like Cypress, this is handled by a Playwright specific library.  Playwright has a test recorder.  You run your app in the test recorder and Playwright will write the code for the test.  This does mean you are writing tests after your code is finished.  However, this is relatively common in UI testing.  "Tests first" is great in non-UI libraries, but constrains the development too much in UI libraries.

There is a [Visual Studio Code integration](https://playwright.dev/docs/getting-started-vscode), has TypeScript support out of the box, and supports ESM (although there is extra setup required).

### Vitest

Vitest is released by the same people who release [vite](https://vitejs.dev/), and it is the spiritual successor to Jest.  When I ran into problems with ES modules and Jest, this was the framework I turned to.  It has "everything" you need.  It support TypeScript and ES modules out of the box, has Chai built-in but also supports Jest expect-style assertions.  It can integrate with [Istanbul] for code coverage. It has [a plugin for Visual Studio code](https://marketplace.visualstudio.com/items?itemName=vitest.explorer).  If you want to do UI testing, it has its own experimental driver or it integrates directly with Playwright.

If I track the evolution of testing frameworks, it would be Mocha -> Jasmine -> Jest -> Vitest.  Each evolution has "more sensible defaults" and requires you to do less to get testing working - irrespective of the environment you are working in.

## So, which do I choose?

I've got a split here:

* For my API or library projects, including the project I am doing research for, I'm going to be using [Vitest].
* I've got a UI-heavy project coming up and for UI testing, I'm going to lean on [Playwright].

I'm choosing these two test runners because:

* They both support TypeScript and ES Modules.
* They both have great Visual Studio Code support.
* They both can use the same type of assertions.
* They both integrate with [Istanbul] for code coverage.
* They work together.

<!-- Links -->
[CucumberJS]: https://cucumber.io/docs/installation/javascript/
[Cypress]: https://www.cypress.io/
[Istanbul]: https://istanbul.js.org/
[Jasmine]: https://jasmine.github.io/
[Jest]: https://jestjs.io/
[MochaJS]: https://mochajs.org/
[NightwatchJS]: https://nightwatchjs.org/
[PlayWright]: https://playwright.dev/
[Vitest]: https://vitest.dev/
[chai]: https://npmjs.com/package/chai