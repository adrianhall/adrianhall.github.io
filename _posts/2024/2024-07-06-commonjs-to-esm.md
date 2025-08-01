---
title:  "Converting a TypeScript project from CommonJS to ESM"
date:   2024-07-06
categories:
  - Web
tags:
  - javascript
  - typescript
---

I haven't made much progress on my own projects recently because of a project at work.  Specifically, I am currently maintaining a CLI tool written in TypeScript about five years ago.  It hasn't really been looked after on a consistent basis, but some of the libraries that it uses (specifically, [update-notifier](https://www.npmjs.com/package/update-notifier) and [wait-on](https://www.npmjs.com/package/wait-on)) have some security issues.  Now, this is a development CLI tool, so the actual vulnerabilities don't affect production code.  Still, many people don't like to use tools that are flagged for high risk vulnerabilities (and I can't say I blame them).  The CLI tool is transpiled into JavaScript using the [CommonJS](https://en.wikipedia.org/wiki/CommonJS) module system.

Not a problem, I thought.  I'll use rev the relevant libraries, build, test, and be done inside of 15 minutes.  Oh, how wrong was I?  What should have been a simple update instead turned into a multi-day adventure that has, at its root, a transition between CommonJS and [ECMAScript Modules](https://nodejs.org/api/esm.html).

The short version is that CommonJS and ESM are totally different standards.  If you are using one, you stick with it throughout.  So if even one library is ESM, then they all have to be.  Bear in mind that your code is a module as well.  In my case, [update-notifier](https://www.npmjs.com/package/update-notifier) was the culprit and caused the wholesale change of the project to ESM.

This is how I did it.

## Step 1: Change package.json

By default, your code will be CommonJS.  Add the following to `package.json` to move your code over to be recognized as ESM:

{% highlight json %}
  "type": "module",
{% endhighlight %}

While you are there, also make sure you are using the latest version of TypeScript.  This ensures you have the best support for ESM.  You may also need to update the `engine` section of your `package.json` to support Node v18 or later.  My CLI was originally written for Node v14 and some things just didn't work.

Yes, maintaining legacy apps is a bear - but there is way more legacy than new stuff.

## Step 2: Change tsconfig.json

I made the following changes to my `tsconfig.json` file:

{% highlight json %}
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "Node16", // ESM Module resolution
    "moduleResolution": "Node16", // ESM module resolution
    // ... rest of your tsconfig.json file
  }
}
{% endhighlight %}

Several pieces of online advice suggest using `ESNext` and `NodeNext`.  These monikers are substitutes "use the latest thing".  I like to lock versions down so that I avoid any hidden problems when my tooling changes.  The values I am using are identical to those used by `ESNext` and `NodeNext` right now.  They may not be in the future (including when you read this).

## Step 3: Update all your imports

With CommonJS, you use `import package from "./mylibdir/mysource";`.  With ESM, you use `import package from "./mylibdir/mysource.js";` - yes, even with TypeScript.  There are ways you can use `.ts` instead (and let the TypeScript transpiler do the work for you) by adding additional directives inside the `tsconfig.json` file , but the effect is the same - you need the extension on the end.

So, go through each and every source code file and add the extension on the end of all your imports.

## Step 4: Remove reliance on aggregator `index.ts` files

I had a number of aggregator `index.ts` files that looked like this:

{% highlight typescript %}
export * from "./myfile.ts";
export * from "./myotherfile.ts";
{% endhighlight %}

This isn't allowed any more.  You have to be specific about what you are exporting:

{% highlight typescript %}
export {
    MyClass,
    myfunc,
    MYCONSTANT
} from "./myfile.ts";
{% endhighlight %}

I found it actually easier to just forego the `index.ts` files and go direct to the source.  If I move a function from one file to another, I need to change a whole bunch of files anyhow.  Maybe I'll figure out a better way using namespacing where I don't have to specify a relative path, but that day is not today.

## Step 5: Update JSON file handling

In CommonJS, you would use `require()` to bring in a JSON file:

{% highlight typescript %}
const pkg = require('../../package.json');
{% endhighlight %}

In ESM, the syntax is different:

{% highlight typescript %}
import pkg from '../../package.json' with { type: 'json' };
{% endhighlight %}

I brought in the `package.json` in several places throughout my code.  I created a new module `package.ts` that did the import for me, then included that everywhere.  In this way, I can isolate the special syntax in one file.

## Step 6: Replace __dirname references

There is no `__dirname` in ESM.  For Node v20.11 / v21.2 and later, you can use the following:

{% highlight typescript %}
const __dirname = import.meta.dirname
{% endhighlight %}

If you are not lucky enough to be able to rev the Node version easily (hello legacy maintainers!), you can use the following:

{% highlight typescript %}
import { dirname } from 'node:path';
import { fileURLToPath } from 'node:url';
    
const __dirname = dirname(fileURLToPath(import.meta.url));
{% endhighlight %}

## Step 7: Update incompatible libraries

By now, you've likely run your tests a bunch of times and figured out some libraries need updating.  I was actually quite lucky in that I only had four or five libraries that needed to be updated.  Most of the time, there was a new major version with the ESM exports in it and everything "just worked".

Then there were the "difficult" libraries.  For me, this was [Ajv](https://ajv.js.org).  To get the ESM version of the module, I needed to upgrade to the latest version.  However, I was using the Ajv library to ensure a file corresponded a JSON schema and that JSON schema was written in JSON Schema draft-04 format - something the latest version of the library did not support.  There are quite a few libraries that have combined the CommonJS to ESM module change with a breaking change in functionality, so it's likely you will run into one.  At this point, you have three options:

1. Bring the older version of the library (with the functionality) into your own code and commit to maintaining the code forever.  
2. Convince the maintainer of the package to restore the functionality you need.  This may mean you get to update the relevant code and submit a PR.  It's likely the maintainer dropped the code for a reason, so don't expect them to welcome the submission.
3. Find another library that has the same functionality you are looking for.

For my situation, I chose option number 3.  The library was replaced with [json-schema-library](https://www.npmjs.com/package/json-schema-library).  This is still being maintained and supports the specification I need. 

While I was doing wholesale library updates, I took a look at `npm outdated` and `npm audit` to see if anything else needed to be updated because of security issues.

## Step 8: Update jest to vitest

As I got to step 4 or 5, I was feeling really good about my work.  Then I ran the tests and **everything** broke.  Every single test was a fail.  However, the CLI itself worked just fine.

Jest is not compatible with ESM.

Sure, they will tell you exactly how you can run Jest to be compatible, but it's jumping through hoops.  Jest is not compatible with ESM out of the box.  You have to do the work necessary to change it.  Throw in TypeScript tests (and the `ts-jest` module) and you quickly realise that it's not going to be a quick change.

Fortunately [vitest](https://vitest.dev/) is compatible with jest (there is even a [migration guide](https://vitest.dev/guide/migration.html#jest)), and it supports ESM and TypeScript.  The migration from jest to vitest takes time.  The migration guide is not as step-by-step as you would want.

While testing, I used the VSCode Jest plugin to run tests manually.  I had to swap this plugin with [Vitest Explorer](https://marketplace.visualstudio.com/items?itemName=vitest.explorer).  The Vitest Explorer has some nice features that weren't available in the Jest plugin. For example, Vitest Explorer allows you to only run the tests that are open in the editor, allowing you to limit test runs to just the tests you are working on.

## Final thoughts

Maintaining legacy code can turn simple requirements (like "just upgrade the library version") into multi-day rabbit holes.  I believe the code I leave behind is better for it.  It isn't more maintainable, but it's more up to date with the standards and that allows the next bug to be fixed that much faster.

And hopefully, now I can get back to my projects!

## Further reading

* [The official ESM guide for TypeScript](https://www.typescriptlang.org/docs/handbook/modules/reference.html#node16-nodenext)
* [A history of JavaScript modules](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules)
* [A guide for converting to ESM](https://gist.github.com/sindresorhus/a39789f98801d908bbc7ff3ecc99d99c#how-can-i-make-my-typescript-project-output-esm)
* [Vitest](https://vitest.dev)
* [Jest to Vitest migration guide](https://vitest.dev/guide/migration.html#jest)
