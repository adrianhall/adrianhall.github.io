---
title: "Building Stylesheets for Blazor with SASS"
categories:
  - "ASP.NET Core"
tags:
  - Blazor
  - SCSS
---

As I mentioned in [my last article]({% post_url 2022/2022-08-25-building-a-serverless-mud %}), I'm building a cloud-based MUD using all the modern techniques that I have available to me.  One of the things I decided was that I was going to use an ASP.NET Core application hosting a single-page web application written in Blazor and I'm going to be running that inside a Docker image so I can deploy it onto Azure Container Apps.  Scaffolding the app out is easy:

{% highlight bash %}
$ mkdir cloudmud
$ cd cloudmud
$ dotnet new blazorwasm
$ .\cloudmud.sln
{% endhighlight %}

This opens up Visual Studio for me with three projects - a `Client`, a `Server`, and a `Shared` project - so far, so good.  

> If you want to learn more about the basics of Blazor, I recommend the following:
>
> * [Beginning Blazor](https://dotnet.microsoft.com/apps/aspnet/web-apps/blazor)
> * [The Blazor Tutorial with Jeff Fritz](https://www.youtube.com/playlist?list=PLdo4fOcmZ0oUJCA3DCzKT79Oe3kdKEceX)
> * [Blazor in Action](https://www.manning.com/books/blazor-in-action) (Manning)
>
> Also check out the [dotnet channel](https://www.youtube.com/c/dotNET) on Youtube - it always has great content!

I have started working on getting my basic home page lit up, but I came across a small nit.

Blazor uses straight-up CSS.

I hate straight-up CSS.

Ok - maybe "hate" is a strong word, but I avoid writing CSS as much as I can.  I find it difficult to get CSS to do the things I want it to do without help.  That's where [SASS](https://sass-lang.com/) comes in.  SASS is completely compatible with CSS (so anything CSS can do, SASS can do better), has a large community, and is better suited for large projects.  This isn't a large project, but I'll still benefit from its capabilities.

Of course, there is always a problem.  It's not supported "out of the box" in Visual Studio.

## Web Compiler to the rescue

I have a few needs for SASS.  Firstly, I need to be able to write SASS (or more correctly, `.scss` files) in Visual Studio and get all the help that Visual Studio is known for - syntax highlighting, snippets, etc.  Ideally, I'd like to keep the SASS files separate from the other files so that I am messing with the `wwwroot` directory as little as possible.  I should be able to build the SASS files into CSS files and have them auto
populate the `wwwroot` directory.  Finally, and most importantly, I need to be able to execute a build within a CI/CD platform without relying on Visual Studio and the existance of a specific setup.

That's where [Web Compiler](https://github.com/madskristensen/WebCompiler) comes in.  Mads Kristensen is known for his excellent extensions.  This particular one has been taken over by Jason Moore and [the new one](https://marketplace.visualstudio.com/items?itemName=Failwyn.WebCompiler64&ssr=false) supports both Dart SASS (the reference implementation) and Node SASS (which lags behind in capabilities).

## Installing Web Compiler

Open up Visual Studio and select **Extensions** > **Manage Extensions**.  Enter **Web Compiler** in the search box, then press Enter.  The **Web Compiler 2022+** by Jason Moore is the first match.  Install it just like you would any other extension (which probably involves restarting Visual Studio).

## Using Web Compiler within Visual Studio

Once you are back in Visual Studio, open your `Client` project and create a new directory called `Styles`.  No, this isn't a special name - it's just a name I picked.  You can pick your own location.  The point is that it is outside of the `wwwroot`. Right-click on the new folder and select **Add**, then find the **SCSS Style Sheet (SASS)** type and create a new style sheet.  

![Adding an SCSS file to the project]({{ site.baseurl }}/assets/images/2022/08-26/image1.png)

You can put whatever you want inside the style sheet - SASS has a lot to offer in terms of a pre-processor for your style sheets, so experiment.  Let's look at a basic setup that I tend to use.  Firstly, I have a `Theme.scss` that includes a whole bunch of variables that I can use:

{% highlight scss %}
@import url('https://fonts.googleapis.com/css2?family=Poppins:wght@300;400;700&display=swap');

:root {
    --footer-background: #404040;
    --header-background: #404040;
    --header-font: Poppins,sans-serif;
    --navbar-brand-background: #dd6e0f;
    --navbar-brand-color1: white;
    --navbar-brand-color2: #808080;
    --page-background: #f9f9f9;
}
{% endhighlight %}

I have also moved (and re-formatted) the standard blazor UI stylesheet into `Blazor.scss`:

{% highlight scss %}
/*
** Standard Blazor Error Controls.
*/
#blazor-error-ui {
    background: lightyellow;
    bottom: 0;
    box-shadow: 0 -1px 2px rgba(0, 0, 0, 0.2);
    display: none;
    left: 0;
    padding: 0.6rem 1.25rem 0.7rem 1.25rem;
    position: fixed;
    width: 100%;
    z-index: 1000;

    .dismiss {
        cursor: pointer;
        position: absolute;
        right: 0.75rem;
        top: 0.5rem;
    }
}

.blazor-error-boundary {
    background: url(data:image/svg+xml;base64,....=) no-repeat 1rem/1.8rem, #b32121;
    padding: 1rem 1rem 1rem 3.7rem;
    color: white;

    &::after {
        content: "An error has occurred";
    }
}
{% endhighlight %}

Just copy the data block from the `wwwroot/css/app.css` file - I've shortened it here for clarity.  Note how SASS allows me to embed the sub-elements, which makes the SASS version of the stylesheet much easier to read.  Now, let's pull this together into the `StyleSheet.scss` file:

{% highlight scss %}
@import url('open-iconic/font/css/open-iconic-bootstrap.min.css');

@import "Theme.scss";
@import "Blazor.scss";

html, page {
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    margin: 0;
    padding: 0;
}
{% endhighlight %}

Now, compile the file by right-clicking on the `StyleSheet.scss` file, then selecting **Web Compiler** > **Compile file**.

![Using Web Compiler to compile SASS]({{ site.baseurl }}/assets/images/2022/08-26/image2.png)

This generates a `StyleSheet.css` file right in the `Styles` directory (which is the wrong place - we'll fix that in a moment), and a `compilerconfig.json` (together with an embedded `compilerconfig.json.defaults` file) that is used to configure the Web Compiler extension.

![The updated project layout]({{ site.baseurl }}/assets/images/2022/08-26/image3.png)

Take a look at the generated `compilerconfig.json` file:

{% highlight json %}
[
  {
    "outputFile": "Styles/StyleSheet.css",
    "inputFile": "Styles/StyleSheet.scss"
  }
]
{% endhighlight %}

It's basically a list of objects with an input and output file.  This is where I can adjust the location of the generated file.  Set the `outputFile` to `wwwroot/css/app.css` and it will overwrite the existing CSS file that the project uses.  I like to rename it at this point so I don't overwrite the existing file.  The only requirement is that the build file be placed in the `wwwroot` directory somewhere so that it is copied "as is" to the built site. If you rename the file, don't forget to adjust the `index.html` file to load the style sheet from the new location.

Now you can right-click on the `StyleSheet.scss` file and select **Web Compiler** > **Re-compile file**, and the generated file will be placed in the right place. 

> Two versions of the CSS file get generated - a normal CSS file and a minified CSS file.  You can load whichever you choose in the `index.html` file.

When you change ANY file in the project that is handled by the Web Compiler, then the Web Compiler runs again and re-generates the file.  This includes `.less`, `.scss`, `.styl`, `.jsx`, `.es6`, and `.coffee` files.  A recompilation is also done when the `compilerconfig.json` file is altered.

## Changing the defaults

Take a look at the `compilerconfig.json.defaults` file.  It contains all the settings that will be used to compile each type of file.  I'm only interested in the SASS and minifiers at the moment, so I can remove the other sections.  (I won't because they don't really affect things and I may need them later on).

{% highlight json %}
{
  "compilers": {
    "sass": {
      "autoPrefix": "",
      "loadPaths": "",
      "style": "expanded",
      "relativeUrls": true,
      "sourceMap": false
    },
  },
  "minifiers": {
    "css": {
      "enabled": true,
      "termSemicolons": true,
      "gzip": false
    },
  }
}
{% endhighlight %}

I can set the defaults here, or I can edit the `compilerconfig.json` file.  For instance, If I want to enable source maps (which is a good idea for debugging purposes), I can set `sourceMap` to true above, or I can set it in `compilerconfig.json` like this:

{% highlight json %}
[
  { 
    "inputFile": "Styles/StyleSheet.scss", 
    "outputFile": "wwwroot/StyleSheet.css",
    "options": {
      "autoPrefix": "last 2 versions",
      "sourceMap": true
    }
  }
]
{% endhighlight %}

The choice is yours.  I prefer to set the defaults for all the files.  The options that can be set in the SASS section are:

* `autoPrefix` sets up auto-prefixing based on [a browserslist query](https://github.com/browserslist/browserslist#query-composition).
* `loadPaths` is list of paths (semi-colon separated) that will be searched when you import SASS modules.
* `style` can be `compressed` or `expanded`.  Compressed style removed as many extra characters as possible and writes the entire stylesheet on a single line.
* `relativeUrls` tells the SASS compiler to emit relative URLs when writing out the source map.
* `sourceMap` tells the SASS compiler to emit an embedded source map into the CSS file.

> It's a good idea to add `*.css` to a `.gitignore` file in the `Styles` directory.  This prevents accidental check-ins of build artifacts.  You may also consider adding a similar `.gitignore` to the `wwwroot` directory once you have integrated compilation into the build.

## Compile SASS during builds

Everything above is really nice if (and only if) you are using Visual Studio.  It doesn't work on Linux, Macs, or if you are using Rider (as an example).  It also won't work when you are building the project in a CI/CD pipeline because Visual Studio is unlikely to have the extension installed when that happens.  You are also requiring that other people who download the project and try to build it have the extension.

We need to integrate SASS compilation into the build step.

Fortunately, this is really easy.  Right-click on the `compilerconfig.json` file and select **Web Compiler** > **Enable compile on build...**.

![Enable compile on build]({{ site.baseurl }}/assets/images/2022/08-26/image4.png)

This will prompt you to approve the addition of a NuGet package to the project:

![A NuGet package is required to enable compilation]({{ site.baseurl }}/assets/images/2022/08-26/image5.png)

Press **Yes**.  Once done, the `Client.csproj` file will look like this:

{% highlight xml %}
<Project Sdk="Microsoft.NET.Sdk.BlazorWebAssembly">

  <PropertyGroup>
    <TargetFramework>net6.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

  <ItemGroup>
    <Content Remove="compilerconfig.json" />
  </ItemGroup>

  <ItemGroup>
    <None Remove="Shared\Navigation.razor.css" />
  </ItemGroup>

  <ItemGroup>
    <Content Include="Shared\Navigation.razor.css" />
  </ItemGroup>

  <ItemGroup>
    <None Include="compilerconfig.json" />
  </ItemGroup>

  <ItemGroup>
    <PackageReference Include="BuildWebCompiler2022" Version="1.14.8" />
    <PackageReference Include="Microsoft.AspNetCore.Components.WebAssembly" Version="6.0.8" />
    <PackageReference Include="Microsoft.AspNetCore.Components.WebAssembly.DevServer" Version="6.0.8" PrivateAssets="all" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\Shared\cloudmud.Shared.csproj" />
  </ItemGroup>

</Project>
{% endhighlight %}

Note the addition of the `<PackageReference Include="BuildWebCompiler2022" Version="1.14.8" />` - this enables the build step.  If you are on a Linux or Mac system, you can add this to your `.csproj` file for the same effect.  Once you've done this change, delete the CSS files that were created in `wwwroot` and use **Ctrl+B** to build the client.  Take a look at the output:

{% highlight text %}
2>------ Rebuild All started: Project: cloudmud.Client, Configuration: Debug Any CPU ------
2>
2>WebCompiler: Begin cleaning output of compilerconfig.json
2>WebCompiler: Done cleaning output of compilerconfig.json
2>
2>WebCompiler: Begin compiling compilerconfig.json
2>	Compiled wwwroot/StyleSheet.css
2>	Minified wwwroot/StyleSheet.min.css
2>WebCompiler: Done compiling compilerconfig.json
2>cloudmud.Client -> D:\GitHub\cloudmud\Client\bin\Debug\net6.0\cloudmud.Client.dll
2>cloudmud.Client (Blazor output) -> D:\GitHub\cloudmud\Client\bin\Debug\net6.0\wwwroot
{% endhighlight %}

This shows the two files being built and they will appear back in the `wwwroot` directory.  The same thing will happen when you build the project through a CI/CD pipeline, so now you don't actually need Web Compiler installed to take advantage of the automatic build.

The only issue you will have is when you are using Hot Reload.  When using Hot Reload, the SCSS build doesn't happen automatically.  You need to use the **Web Compiler** > **Re-compile file** command, then "force-refresh" the browser to make it load the style sheet again.  

## Next steps

Once I have my main page looking nice (and that may take a couple of days - I think I mentioned I'm not good at this stuff!), I'm going to move onto authentication in Blazor.  

Until the next time, happy hacking!