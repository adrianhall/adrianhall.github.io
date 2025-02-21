---
title: "Bottom Bar Tabs for your Xamarin Forms app"
categories:
  - Xamarin
tags:
  - "Xamarin Forms"
---

Following on from [last weeks article]({% post_url 2021/2021-05-09-xamarin-forms-glyph-fonts %}), I am continuing on my UI journey for my contacts app.  My app design calls for a bottom app bar, with three sections - contacts, groups, and profile.  The contacts and groups will be lists, and the profile will be a detail page for your own details with an editor.

To start, I created the three pages as `ContentPage` views, complete with XAML layout.  For example, here is my `ListContactsPage` XAML file:

``` xml
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage xmlns="http://xamarin.com/schemas/2014/forms"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             x:Class="ContactsApp.Views.ListContactsPage"
             Title="Contacts"
             BackgroundColor="{DynamicResource WindowBackgroundColor}">
    <ContentPage.Content>
        <StackLayout>
            <Label Text="Contacts Page" 
                   VerticalOptions="CenterAndExpand" 
                   HorizontalOptions="CenterAndExpand" />
        </StackLayout>
    </ContentPage.Content>
</ContentPage>
```

The `WindowBackgroundColor` is defined in the application level `App.xaml` file:

``` xml
<Application xmlns="http://xamarin.com/schemas/2014/forms"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             xmlns:controls="clr-namespace:ContactsApp.Controls"
             x:Class="ContactsApp.App">

    <Application.Resources>
        <ResourceDictionary>
            <!-- Colors -->
            <Color x:Key="WindowBackgroundColor">#1F1D2B</Color>

            <!-- ... rest of the resource dictionary ... -->

        </ResourceDictionary>
    </Application.Resources>
</Application>
```

The `ListContactsPage.xaml.cs` is nothing special at this point.  In fact, it's the default when you create a `ContentPage`.

Now I've got my three pages, how do I make them into three tabs?

## Step 1: Create an App Shell

Xamarin Forms introduced the concept of an app shell a while ago.  It's a near construct that allows you to centralize the navigation for your app into one place.  It understands navigation stacks and how different pages are constructed.  When I looked at all the navigation mechanisms that are available for Xamarin.Forms, this one "clicked" in my head.  The first step is to wire it up.  If you've created a new Xamarin Forms project, you can just select the Tab bar template.  However, that comes with a lot of baggage, so let's rewind and wire it in from a blank canvas.

Create a new `xaml`/`xaml.cs` file combination in your shared project - it can be a `ContentPage` or `ContentView` - it really doesn't matter.  By convention, this is called `AppShell.xaml`, but it is really up to you what you call it.  The `AppShell.xaml.cs` file can be "normal":

``` csharp
namespace ContactsApp
{
    public partial class AppShell : Xamarin.Forms.Shell
    {
        public AppShell()
        {
            InitializeComponent();
        }
    }
}
```

The important thing here is that it inherits from the `Xamarin.Forms.Shell` and you initialize the component (which is common to all Xamarin Forms components).

## Step 2: Create the tabs

The work is done in the XAML for the app shell:

``` xml
<Shell xmlns="http://xamarin.com/schemas/2014/forms"
       xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
       xmlns:views="clr-namespace:ContactsApp.Views"
       Title="Contacts"
       x:Class="ContactsApp.AppShell"
       BackgroundColor="{DynamicResource WindowBackgroundColor}">
    <Shell.Resources>
        <Style TargetType="TabBar">
            <Setter Property="Shell.TabBarBackgroundColor" Value="{DynamicResource TabBarBackgroundColor}" />
            <Setter Property="Shell.TabBarDisabledColor" Value="{DynamicResource TabBarDisabledColor}"/>
            <Setter Property="Shell.TabBarForegroundColor" Value="{DynamicResource TabBarForegroundColor}" />
            <Setter Property="Shell.TabBarTitleColor" Value="{DynamicResource TabBarTitleColor}"/>
            <Setter Property="Shell.TabBarUnselectedColor" Value="{DynamicResource TabBarUnselectedColor}" />
        </Style>
    </Shell.Resources>

    <TabBar>
        <ShellContent ContentTemplate="{DataTemplate views:ListContactsPage}" Route="ListContacts" Title="Contacts">
            <ShellContent.Icon>
                <FontImageSource FontFamily="{DynamicResource IconFontFamily}" Glyph="{StaticResource ContactsIcon}"/>
            </ShellContent.Icon>
        </ShellContent>
        <ShellContent ContentTemplate="{DataTemplate views:ListGroupsPage}" Route="ListGroups" Title="Groups">
            <ShellContent.Icon>
                <FontImageSource FontFamily="{DynamicResource IconFontFamily}" Glyph="{StaticResource GroupsIcon}"/>
            </ShellContent.Icon>
        </ShellContent>
        <ShellContent ContentTemplate="{DataTemplate views:ProfilePage}" Route="Profile" Title="Profile">
            <ShellContent.Icon>
                <FontImageSource FontFamily="{DynamicResource IconFontFamily}" Glyph="{StaticResource ProfileIcon}"/>
            </ShellContent.Icon>
        </ShellContent>
    </TabBar>
</Shell>
```

There are two sections here.  First, the resources.  We're setting up a style for the tab bar that uses colors defined in the `App.xaml` file.  I like to centralize all these definitions.  It makes it easier to make changes in one color.  You can define the colors like this:

``` xml
    <!-- Colors for the tab bar -->
    <Color x:Key="TabBarBackgroundColor">#252836</Color>
    <Color x:Key="TabBarTitleColor">#6FCF97</Color>
    <Color x:Key="TabBarUnselectedColor">#828282</Color>
```

Select whatever colors you want.  The `TabBarTitleColor` is the color of the selected icon, and the `TabBarUnselectedColor` is the color of the rest of the icons.

The second section is the tabs themselves.  You can create multiple tab bars, each with their own navigation stack.  The tab bars will be swapped out when you navigate to a route defined by the `Route`.  My tab bar has three tabs - Contacts, Groups, and Profile.  Here is the profile in isolation:

``` xml
    <ShellContent ContentTemplate="{DataTemplate views:ProfilePage}" Route="Profile" Title="Profile">
        <ShellContent.Icon>
            <FontImageSource FontFamily="{DynamicResource IconFontFamily}" Glyph="{StaticResource ProfileIcon}"/>
        </ShellContent.Icon>
    </ShellContent>
```

Each tab definition contains four things:

* A `ContentTemplate` which points to your content page for the tab.
* A `Route`, which needs to be unique and is used in navigation.
* A `Title`, which is text shown below the icon in the tab bar.
* An `Icon`, which is a glyph shown in the tab bar.

The icon can be an image (like a PNG), or - as I am doing here - a web font glyph.

## Step 3 - Start the App Shell

The final step is to wire up the app shell so that it is displayed when the application starts.  This is done within the `App.xaml.cs` file:

``` csharp
    public App()
    {
        InitializeComponent();
        MainPage = new AppShell();
    }
```

Just replace the default `MainPage` with `AppShell`.  Yes - it really is that simple.

## The final product

Here is my final product:

![The tab bar from a shell]({{ site.baseurl }}/assets/images/2021/2021-05-15-image1.png)

As you can see, it looks pretty professional.  The use of a web font for the icons and a suave dark set of colors makes for an awesome look.
