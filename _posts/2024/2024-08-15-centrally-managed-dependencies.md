---
title:  "Centrally managing dependencies in your C# solutions"
date:   2024-08-15
categories:
  - Tips
tags:
  - csharp
  - dependencies
---

Today, I'd like to talk about the best way to centrally manage dependencies in your dotNET solutions.  It's common for a single solution to comprise multiple projects.  The happy path for maintaining dependencies in Visual Studio involves right-clicking on the project and selecting "Manage NuGet packages...".  Once you have a set of packages, you can keep them in sync by right-clicking on the solution and doing the same thing.  By using "Manage NuGet packages" for the solution, you can update multiple projects at the same time.

So, why is this bad?  If you happen to have a project that uses v1.0 of a library, then you include a common library in your project that uses v2.0 of a library, they will conflict.  You won't be able to build your solution until you repair the conflict.  Also, some libraries need to be updated together.  ASP.NET Core, Aspire, and Entity Framework Core are three common examples of collections of NuGet packages that need to be versioned together.

So, if the happy path doesn't lead to happiness, how should you do it?

## Step 1: Create a Directory.Build.props

One of the best things you can do is to consolidate your common project settings into a common props file.  Let's say you have all your projects in the `src` directory, and they all use the same target framework, but you also want to ensure that nullables are enabled everywhere, you are using implicit using statements, and that all your projects share a common UserSecretsId so that you can consolidate the development configuration for all projects in one place.  You can place a `Directory.Build.props` file in the `src` directory, then fill it with the following information:

{% highlight xml %}
<Project>
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <UserSecretsId>1880ee15-3d6a-4399-91bc-3f314c8199ab</UserSecretsId>
  </PropertyGroup>
</Project>
{% endhighlight %}

Any common settings in a property group or item group can be placed in this file.  You can then remove the same settings from your `.csproj` files as they will be incorporated automatically.  Technically, you don't need to do this step.  However, if you are consolidating these things, you might as well do it right.

## Step 2: Create a Directory.Packages.props

This file is where the magic happens.  There are three sections to this file:

* The properties that enable centralized package versioning.
* A set of variables for substituting common version numbers.
* A set of `<PackageVersion>` elements to provide the versions of each NuGet package.

Let's take a look:

{% highlight xml %}
<Project>
  <!-- Enable centralized package versioning -->
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
    <CentralPackageTransitivePinningEnabled>true</CentralPackageTransitivePinningEnabled>
  </PropertyGroup>

  <!-- Common versions -->
  <PropertyGroup>
    <AspnetVersion>8.0.8</AspnetVersion>
    <EntityFrameworkVersion>8.0.8</EntityFrameworkVersion>
  </PropertyGroup>

  <!-- Package versions -->
  <ItemGroup>
    <!-- Version together with ASP.NET -->
    <PackageVersion Include="Microsoft.AspNetCore.Authentication.JwtBearer" Version="$(AspnetVersion)" />
    <PackageVersion Include="Microsoft.AspNetCore.Authentication.OpenIdConnect" Version="$(AspnetVersion)" />
    <PackageVersion Include="Microsoft.AspNetCore.Identity.EntityFrameworkCore" Version="$(AspnetVersion)" />
    <PackageVersion Include="Microsoft.AspNetCore.Identity.UI" Version="$(AspnetVersion)" />
    <PackageVersion Include="Microsoft.AspNetCore.OpenApi" Version="$(AspnetVersion)" />
    <PackageVersion Include="Microsoft.Extensions.Identity.Stores" Version="$(AspnetVersion)" />

    <!-- Version together with EF -->
    <PackageVersion Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="8.0.4" />
    <PackageVersion Include="Microsoft.EntityFrameworkCore" Version="$(EntityFrameworkVersion)" />
    <PackageVersion Include="Microsoft.EntityFrameworkCore.SqlServer" Version="$(EntityFrameworkVersion)" />
    <PackageVersion Include="Microsoft.EntityFrameworkCore.Tools" Version="$(EntityFrameworkVersion)" />

    <!-- Other packages -->
    <PackageVersion Include="Automapper" Version="13.0.1" />
  </ItemGroup>
</Project>
{% endhighlight %}

You can add as many variable versions to the common versions section as you want, and you can add as many PackageVersion references as you want.  This also has the advantage of centralizing all the known packages that you are using in one handy place.

**Add the Directory.\*.props to your solution**<br/>
You can add the additional solution files by right-clicking on the solution (or a solution folder) and using "Add > Existing Item..." to add the files for editing.
{: .notice--warning}

## Step 3: Remove the versions from your .csproj files

Now that you have all the versions centralized, your builds will start spitting out warnings about having versions in your .csproj file while centrally managing versions.  You can fix these warnings by simply removing the `Version="version"` property from each `<PackageReference>` element in your .csproj files.

## Step 4: Updating packages centrally

You will find that your handy "Manage NuGet Packages..." gesture for both the solution and individual projects now updates the central `Directory.Packages.props` file.  You don't need to do anything different.

There is one big proviso here, though.  If you have defined a variable for the version, that is not updated automatically.  Instead, the `<PackageVersion>` entry for the package is re-written with the actual version.  If you want to update something with a version variable, you need to edit the `Directory.Packages.props` file instead.

## Final thoughts

Centrally managing package dependencies is harder work than using the happy path that Visual Studio provides. However, the benefits for centrally managing packages far outweighs the inconvenience in larger and more complex projects.

I don't centrally manage package dependencies all the time, but a good number of my solutions are multi-project.  As soon as I move to multiple projects, I start centrally managing package versions.

## Further reading

* [Central package management with NuGet](https://learn.microsoft.com/nuget/consume-packages/Central-Package-Management)