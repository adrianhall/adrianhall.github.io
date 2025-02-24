---
title:  "TypeScript, ES Modules, and root-relative imports"
date:   2024-07-10
categories: Devtools
tags:
    - TypeScript
    - Tips
---

As you might have gathered from my [last article]({% post_url 2024/2024-07-06-commonjs-to-esm %}), I'm currently working in the [TypeScript](https://typescriptlang.org) world.  My experience with converting from CommonJS to ES Modules got me thinking - what is state of the art right now?  So I delved in.

I want to build a CLI tool using TypeScript and ES Modules but I want to use non-relative roots.

What are non-relative roots?  Well, if you want to import a module within your project with ES Modules, you might write something like this:

{% highlight typescript %}
import { something } from '../../libs/core/something.js';
{% endhighlight %}

This has two problems for me.  Firstly, it's importing '.js' but my files are '.ts'.  It honestly messes with my head that I have to change the extensions.  I am not going to be able to fix that with just TypeScript (more on that later).

Also, there is that relative path.  In smaller projects, relative paths are fine.  However, in larger projects, they can become cumbersome.  You have to remember how many '..' elements to put in to get to the root, and then remember to update all the references when you move that source file.  It's better to have a path that is relative to the project root.  I'd prefer importing like this:

{% highlight typescript %}
import { something } from '#root/libs/core/something.js';
{% endhighlight %}

The import remains the same no matter where my source file is in the project. 

## Creating a TypeScript project.

Let's start by creating a basic TypeScript project:

{% highlight bash %}
mkdir myproject
cd myproject
npm init -y
npm install -D typescript
npx tsc --init
{% endhighlight %}

There is nothing new or unique here. This should be how you create any TypeScript project. I've created a `project.json` file with all the defaults, installed TypeScript as a devDependency, then used that to create a `tsconfig.json` file.  I'm also going to use Visual Studio Code to create a `src` directory and create a simple `index.ts` file:

{% highlight typescript %}
console.log('Hi from the CLI!');
{% endhighlight %}

## Set up the `tsconfig.json` file.

Before you can compile TypeScript, you need to set up the `tsconfig.json` file.  A basic one is created for you with `npx tsc --init`.  Let's modify it for ES modules:

{% highlight json %}
{
  "compilerOptions": {
    "target": "ESNext",
    "lib": [
      "ESNext"
    ],
    "module": "NodeNext",
    "baseUrl": "./src", 
    "types": [
      "node"
    ],
    "resolveJsonModule": true, 
    "outDir": "./dist",
    "isolatedModules": true,
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true, 
    "strict": true, 
    "skipLibCheck": true
  },
  "include": [
    "src/**/*.ts"
  ],
  "exclude": [
    "node_modules"
  ]
}
{% endhighlight %}

Since I am including `node` in the types section, I also need the following module:

{% highlight bash %}
npm install -D @types/node
{% endhighlight %}

I can now run the compilation with `npx tsc`.  It will output the runnable files in the `./dist` directory.  You can run the script with the following:

{% highlight bash %}
node ./dist/index.js
{% endhighlight %}

### Update the `package.json` scripts.

I set up my build, deploy, and test infrastructure in `package.json`, allowing me to define dependencies in one place.  Let's start by adding some more packages:

{% highlight bash %}
npm install -D rimraf npm-run-all
{% endhighlight %}

The [rimraf](https://npmjs.org/packages/rimraf) package is a cross-platform cleanup utility similar to running `rm -rf` on Linux systems.  The [npm-run-all](https://npmjs.org/packages/npm-run-all) package allows you to construct cross-platform pipelines of commands.  Talking of which, let's look at my "scripts" section in `package.json`:

{% highlight json %}
"scripts": {
    "build": "run-s build:compile",
    "build:compile": "tsc",
    "clean": "rimraf -fr dist"
},
{% endhighlight %}

I build up each separate step of the build process with it's own script, then I bundle them together with the `run-s` command (from the `npm-run-all` package).  It doesn't make sense now, but will when I add linting, testing, deployments, and so on.

## Getting complicated with modules

Up until this point, I haven't used any modules.  The `index.ts` file doesn't import anything.  Let's create a new module in `./src/modules/hello.ts`:

{% highlight typescript %}
export function sayHello(): void {
    console.log('Hello from the hello module!');
}
{% endhighlight %}

I can also alter the `index.ts` to import it:

{% highlight typescript %}
import { sayHello } from './modules/hello.js';

sayHello();
{% endhighlight %}

Now that I'm using ES modules, I need to tell Node that I'm using ES modules.  This is done by adding `"type": "module",` into the `package.json` file.

Run the build, then run `node ./dist/index.js` again.  It still works, but I am still using a relative path for my module.  My design aim here is to get rid of the relative path.  I want something like `#root/modules/hello.js` instead:

{% highlight typescript %}
import { sayHello } from '#root/modules/hello.js';

sayHello();
{% endhighlight %}

There are red squiggly lines in Visual Studio code.  If you just run the compile step (`npm run build`), you'll get the following error:

{% highlight text %}
src/index.ts:1:26 - error TS2307: Cannot find module '#root/modules/hello.js' or its corresponding type declarations.
{% endhighlight %}

Let's start by making TypeScript happy and get rid of those red squiggly lines.  I'm going to take advantage of subpaths in TypeScript.  Edit the `tsconfig.json` file and add the following in the compilerOptions section:

{% highlight json %}
{
    "compilerOptions": {
        "paths": {
            "#root/*": [ "./*" ]
        },
        /* Rest of the tsconfig.json */
    }
}
{% endhighlight %}

I put this section right below the `baseUrl` setting.  That's because the two work in tandem.  When I specify `#root/modules/hello.js`, the paths setting will translate that to `./src/modules/hello.js`; TypeScript will find that, so it won't complain.  Everything is relative to the baseUrl here.

That change has made Visual Studio happy and TypeScript accepted the code but the code won't run.  The error is:

{% highlight text %}
node:internal/modules/esm/resolve:291
  return new ERR_PACKAGE_IMPORT_NOT_DEFINED(
         ^

TypeError [ERR_PACKAGE_IMPORT_NOT_DEFINED]: Package import specifier "#root/modules/hello.js" is not defined in package .../package.json imported from .../dist/index.js
{% endhighlight %}

I need to set up [Node subpath imports](https://nodejs.org/api/packages.html#subpath-imports).  This is something that works with both JavaScript and TypeScript since it's a part of Node.  Subpath imports are defined in `package.json`:

{% highlight json %}
"imports": {
    "#root/*.js": "./dist/*.js"
},
{% endhighlight %}

If I run `npm run build` and then `node ./dist/index.js`, my application works! 

## Final thoughts

Let's go through what I've covered today:

* How to set up a TypeScript + Node application.
* How to get root-relative imports working.

I'm still not 100% happy because I'm referencing the `.js` version inside the TypeScript file.  There are two features that I wish to explore.  One is "directory importing".  Instead of specifying the actual TypeScript file, I could reference the directory and it would automatically use the index.ts file underneath it.  The second is importing the `.ts` file instead of the `.js` file. There is a feature of TypeScript called "allowImportingTsExtensions" which does what I want, but it only allows you to use it if you are not compiling (type checking only).  That means I'm hunting for another tool - one that compiles TypeScript while transforming my imports on the fly.

This is also really only the start of the journey.  Some other things I want to get to:

* How to build a solid CLI.
* How to test an ES module.
* How to do static analysis (linting).
* How to move towards a great contributor experience so others can work on this code.

The code I developed today (and in future articles) is available in a [template repository](https://github.com/adrianhall/esm-typescript-library) so you can go and try it out for yourself.  This will be frequently updated as I write more articles about this topic.

Until next time, happy hacking!

## Further reading

* [TypeScript](https://typescriptlang.org)
* [ES Module Syntax](https://www.typescriptlang.org/docs/handbook/2/modules.html#es-module-syntax)
* [Node subpath imports](https://nodejs.org/api/packages.html#subpath-imports)