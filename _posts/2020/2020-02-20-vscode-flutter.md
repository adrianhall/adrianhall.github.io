---
title: "Pre-build steps for Flutter using Visual Studio Code"
categories:
  - Flutter
---

Flutter is not a build system.

I know, it's a terrible way to start, but you have to accept that you need to produce a build system for Flutter.  There are build systems written in Dart (such as [this one](https://github.com/dart-lang/build)), and I'm sure this situation will get better, but right now?  There isn't a good build system.  So I use the node package manager.

Here is my basic flow:

1. Run `npm init -y`.
2. Edit the produced `package.json` file to use flutter under the covers.
3. Run flutter with `npm start` instead of the usual flutter commands.

Here is my `package.json` file as an example:

```json
{
  "name": "My App - Build",
  "version": "1.0.0",
  "description": "Build system for my flutter app",
  "private": true,
  "scripts": {
    "start": "flutter run",
    "build": "flutter build"
  },
  "author": "Adrian Hall <photoadrian@outlook.com>",
  "license": "MIT"
}
```

It's nice and simple.  Now, I have a requirement for a pre-build step.  I need to copy some JSON configuration files from my backend to my front end so that the code knows where to look.  I can easily add it to the `package.json`:

```json
{
  "name": "My App - Build",
  "version": "1.0.0",
  "description": "Build system for my flutter app",
  "private": true,
  "scripts": {
    "copyClientConfiguration": "cpy ../../service/output/*.json ./config",
    "prestart": "run-s copyClientConfiguration",
    "start": "flutter run",
    "prebuild": "run-s copyClientConfiguration",
    "build": "flutter build"
  },
  "author": "Adrian Hall <photoadrian@outlook.com>",
  "license": "MIT",
  "devDependencies": {
    "cpy-cli": "^3.1.0",
    "npm-run-all": "^4.1.5"
  }
}
```

There are lots of CLI utilities you can use for handling the pre-build steps you want (or you can build your own). The `cpy-cli` package copies files into a directory (cross-platform, so it works on both Mac and Windows), and the `npm-run-all` package gives me the `run-s` command for running NPM tasks sequentially (and thus allowing me to attach the pre-build task to multiple other tasks).

Now when I run `npm start`, the configuration files are first copied over, and then my application is built and run.

### Visual Studio Code

I don't actually develop on the command line, however.  When developing using Flutter, I use Visual Studio Code.  This has a flutter debugger which I find useful.  To run flutter from the debugger, you add a launch configuration.  It looks like this:

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Flutter",
            "request": "launch",
            "type": "dart",
        }
    ]
}
```

The problem with this is that it bypasses my nice build system.  However, Visual Studio Code can also execute tasks right before launching the debugger.  First, define a task.  Press Ctrl-Shift-P to bring up the command palette, and select **Tasks: Configure Task**.  You should be able to go through the cofniguration to add the `npm: copyClientConfiguration` task.  In your `.vscode/tasks.json` file, this is defined thusly:

```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "type": "npm",
            "script": "copyClientConfiguration",
            "problemMatcher": []
        }
    ]
}
```

You can now go back to the launch configuration (stored in `.vscode/launch.json`), and add it as a `preLaunchTask`:

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Flutter",
            "request": "launch",
            "type": "dart",
            "preLaunchTask": "npm: copyClientConfiguration"
        }
    ]
}
```

When you launch the flutter application from the debug menu, it will first copy your configuration files into the project before building and launching your app.  This duplicates the effect that I have configured with the command line build.
