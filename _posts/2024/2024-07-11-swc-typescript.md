---
title:  "Building TypeScript projects with the swc compiler"
date:   2024-07-11
categories:
    - Devtools
tags:
    - TypeScript
    - Tips
---

In my [last article]({% post_url 2024/2024-07-10-esm-typescript %}), I set up a small project that I'm going to use for TypeScript development using ES modules that are "root-relative" - i.e. I don't have to provide a relative path.  I can use a path like `#root/relative/path.js` instead so that the code doesn't change if I decide to move the source file I'm working on.

I want to switch from importing ".js" files to importing ".ts" files.  To do that, I need to set `allowImportingTsExtensions`.  The `tsc` tool says I can't do that unless I also set `noEmit`.  So I need an alternative compiler. 

Today, I thought I would check out something that has been making the rounds recently - an alternative TypeScript compiler called [swc](https://swc.rs).  Swc does not do any type checking, so you still need to install the `typescript` module, especially if you are using Visual Studio Code.  So what's the point then?

There are a number of ways of compiling TypeScript into JavaScript - a step you have to do in order to use your application in a Node-native environment.

1. [tsc](https://typescriptlang.org) is the default option.  It is written and supported by Microsoft.  It's been around as long as TypeScript has been around.
2. [swc](https://swc.rs) is written and supported by Vercel and is designed to be very fast when compiling large code bases.  It's the relative newcomer.
3. [babel](https://babeljs.io/docs/en/) is designed to transpile newer versions of JavaScript and TypeScript into older versions of JavaScript for maximum compatibility.  It's the grand-daddy of them all.

There are others (esbuild and sucrose, for example).  However, these three represent the best of breed when it comes to pure compilation tasks.  In terms of speed, swc is the fastest, tsc is next, and babel is the slowest.  In terms of maturity, swc is very new with few plugins (and thus little configurability) whereas babel has a vast array of plugins to allow you to tweak it any way you want.

## Integrating swc

First, I just want to get the two-step compilation process working. 

### 1. Install the swc compiler in your project.

This is a simple `npm` command:

{% highlight bash %}
npm install -D @swc/cli @swc/core
{% endhighlight %}

### 2. Update your tsconfig.json file.

At a minimum, you need to set `noEmit` to true.  The [swc documentation](https://swc.rs/docs/migrating-from-tsc) also suggests a number of other changes.  Here are all the changes I made:

{% highlight json %}
{
    "compilerOptions": {
        "noEmit": true,
        "isolatedModules": true, 
        "esModuleInterop": true,
        "verbatimModuleSyntax": true,
        "useDefineForClassFields": true,
        /* Rest of your tsconfig.json */
    }
}
{% endhighlight %}

You can see the complete [tsconfig.json on my GitHub repository](https://github.com/adrianhall/esm-typescript-library/blob/v2/tsconfig.json).

### 3. Create a .swcrc file

I copied this from the swc repository.  Based on the number of blogs and repositories I reviewed, everyone else did too.  You might as well copy it as well:

{% highlight json %}
{
    "$schema": "https://swc.rs/schema.json",
    "jsc": {
        "baseUrl": "./src",
        "parser": {
            "syntax": "typescript"
        },
        "target": "esnext"
    },
    "module": {
        "type": "nodenext"
    }
}
{% endhighlight %}

The `baseUrl`, `target`, and `module` must match the same settings in your `tsconfig.json` file.  Otherwise, the settings are pretty straight forward.

### 4. Update the build definition

Here is the relevant part of my build definition in the `package.json` file:

{% highlight json %}
  "main": "dist/index.js",
  "type": "module",
  "imports": {
    "#root/*.js": "./dist/*.js"
  },
  "scripts": {
    "build": "run-s build:typecheck build:compile",
    "build:typecheck": "tsc",
    "build:compile": "swc ./src --out-dir dist --strip-leading-paths",
    "clean": "rimraf -fr dist"
  },
{% endhighlight %}

Swc does not do type checking, so I've added an explicit type-check as part of the build pipeline. You can find the code thus far on [my GitHub repository](https://github.com/adrianhall/esm-typescript-library/blob/v2).

## Adding support for `.ts` imports

My next step is to get `.ts` imports working.  If I change my `./src/index.ts` file to the following:

{% highlight typescript %}
import { sayHello } from '#root/modules/hello.ts';

sayHello();
{% endhighlight %}

I want that to work.  It makes a lot more sense (to me) to be importing TypeScript files in a TypeScript world.  However, as soon as you make that change, you get the dreaded red squiggly lines in Visual Studio Code.  Update your `paths` section in the `tsconfig.json` file:

{% highlight json %}
{
    "compilerOptions": {
        "allowImportingTsExtensions": true,
        /* Rest of your tsconfig.json file */
    }
}
{% endhighlight %}

Now that the red squiggly lines have gone away, I can run the build. However, there is nothing to transform the import into the equivalent `.js` import (which is needed when running the application from Node).  I get the following error:

{% highlight text %}
node:internal/modules/esm/resolve:291
  return new ERR_PACKAGE_IMPORT_NOT_DEFINED(
         ^

TypeError [ERR_PACKAGE_IMPORT_NOT_DEFINED]: Package import specifier "#root/modules/hello.ts" is not defined in package /.../esm-typescript-library/package.json imported from /.../esm-typescript-library/dist/index.js
{% endhighlight %}

Fortunately, swc has a concept of plugins.  Unfortunately, they are experimental and I've found them rather temperamental at times.  The plugin I am using for this is `@swc/plugin-transform-imports`.  It has no documentation, so I had to copy the code from [another blog](https://dev.to/a0viedo/nodejs-typescript-and-esm-it-doesnt-have-to-be-painful-438e), which turned out to be super helpful.  First off, install the plugin:

{% highlight bash %}
npm install -D @swc/plugin-transform-imports
{% endhighlight %}

Here is the new `.swcrc` file:

{% highlight json %}{% raw %}
{
    "$schema": "https://swc.rs/schema.json",
    "jsc": {
        "baseUrl": "./src",
        "parser": {
            "syntax": "typescript"
        },
        "target": "esnext",
        "experimental": {
            "plugins": [
                [
                    "@swc/plugin-transform-imports",
                    {
                        "^(.*?)(\\.ts)$": {
                            "skipDefaultConversion": true,
                            "transform": "{{matches.[1]}}.js"
                        }
                    }
                ]
            ]
        }
    },
    "module": {
        "type": "nodenext"
    }
}
{% endraw %}{% endhighlight %}

Take a look at the plugins section. The syntax is horrible.  However, it converts any imports that match the given regular expression `^(.*?)(\.ts)$`.  You have to quote backslashes with a backslash when you put the regular expression inside a JSON property.  When it parses the regular expression, it gets a set of matches - these become `matches.[index]` in the transform statement.  The transform statement is a "handlebars" style statement.  

Let's take our one match.  `#root/modules/hello.ts` is matched as follows:

* matches.[0] == `#root/modules/hello.ts` (the zero index is always the whole match)
* matches.[1] == `#root/modules/hello`
* matches.[2] == `.ts`

I am not good at regular expressions, but I've found [regex101](https://regex101.com/) to be an excellent resource.

Clean and build your project, then analyze the `./dist/index.js` file.  This is the compiled file:

{% highlight js %}
import { sayHello } from "#root/modules/hello.js";
sayHello();
{% endhighlight %}

You can see that the import has been modified correctly.  You can also run `node ./dist/index.js` and see that the application runs correctly.

## Final thoughts

I'm not sure that swc is useful for my project yet.  The relative immaturity of the tool, lack of documentation for plugins, and general lack of examples is a lot of risk just so I can import `.ts` files.  My project won't be big enough to make the speed of compilation make a difference either.

That said, I'm glad I went through this process.  You can find the project on [my GitHub repository](https://github.com/adrianhall/esm-typescript-library/blob/v3).

## Further reading

* [TypeScript](https://typescriptlang.org).
* [swc](https://swc.rs).
* [NodeJS, TypeScript, and ESM: It doesn't have to be painful](https://dev.to/a0viedo/nodejs-typescript-and-esm-it-doesnt-have-to-be-painful-438e) by Alejandro Oviedo.