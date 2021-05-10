---
title: "Adding a Glyph Font to your Xamarin Forms apps"
categories:
  - Xamarin Forms
---

I've started building a new app using Xamarin Forms from scratch.  The idea is that I have a list of contacts stored in the cloud.  There isn't anything revolutionary about that, but I'm going to allow sharing of specific lists with others.  You could, for example, create a tag called "Cycling Buddies", and then tag your friends with that.  Everyone who subscribes to the same list gets the same information.  Even better, you can "broadcast" your information to certain groups, allowing other people to pick up changes to your profile as needed.  It's a "more complicated than a Todo List" application that I'm going to be using for a bunch of cloud backend tutorials in the future.

Of course, I have to write it first, so I thought I would document my journey.  Each blog post will have a small tip, with the code to make it happen.

## Icon Fonts

In the old days, we had to create graphic assets for icons.  Each platform had their own format and resolution requirements.  You ended up needing several copies of the same icon to create a set.  Along came Font Awesome for the web, and the world seemed to change overnight.  Gone was the need to create several copies of the same icon.  Instead, you had a icon font that you added to the application.  There are several to choose from, each with their own visual style.  If you want to follow style guides, then the [Material Icon Font](https://materialdesignicons.com/) for Android, and the [SF Symbols Font](https://developer.apple.com/sf-symbols/) are for you.  Unfortunately, SF Symbols is not available as a standalone font from Apple, but there are several "close" icon sets to choose from.  You can also pick [Ionicons](https://ionic.io/ionicons/v4), which is built by Ionic (and they have a vested interest in getting close to Apple design aesthetics).

Aside from those, there is [Font Awesome](https://fontawesome.com/), [Octicons](https://octicons.github.com/), and many others.  Take a look at [this big list](https://acodez.in/web-icon-fonts/) for more ideas.  The important thing is to take a look at your icon needs, then select an icon font you like that has all the icons you need.

Once you've done that, find the TrueType font (.TTF file) and download it.

### Adding an Icon Font to your project

There are three steps to adding an Icon Font.

1. Add the TrueType font to each platform.
1. Include a font style within the application resources.
1. Create glyph resources to identify each icon you want to use.

Let's take an example.  I am going to be using the MaterialDesignIcons font in this app.  The font is available [on GitHub](https://github.com/Templarian/MaterialDesign-Webfont/tree/master/fonts) - just download the .ttf font.  It needs to be stored in each platform project:

* In Android projects, it is stored in the `Assets` folder.
* In iOS projects, it is stored in the `Resources` folder.
* In UWP projects, it is stored in the `Assets/Fonts` folder.

Ensure you have set the **Build Action** appropriately.  Right-click on the font, then select **Properties**.  Android requires the build action to be _AndroidAsset_, and iOS requires it to be _BundleResource_.

You will also need to know the name of the font.  On Windows, this can be found by double-clicking on the TTF file:

![How to find the name of the font]({{ site.baseurl }}/assets/images/2021/2021-05-09-image1.png)

In this case, the name is _Material Design Icons_.  Now, let's define the font in the shared project `App.xaml` file:

```xml
<?xml version="1.0" encoding="utf-8" ?>
<Application xmlns="http://xamarin.com/schemas/2014/forms"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             x:Class="ContactsApp.App">
    <Application.Resources>
        <ResourceDictionary>
            <!-- Main Icon Font -->
            <OnPlatform x:Key="IconFontFamily" x:TypeArguments="x:String">
                <On Platform="Android" Value="materialdesignicons-webfont.ttf#Material Design Icons"/>
                <On Platform="iOS" Value="Material Design Icons"/>
                <On Platform="UWP" Value="Assets/Fonts/materialdesignicons-webfont.ttf#Material Design Icons"/>
            </OnPlatform>
        </ResourceDictionary>
    </Application.Resources>
</Application>
```

You can see that each platform requires a different value for the font family.

> **Use a friendly name for your icon font**
>
> I've seen a lot of people use the name of the font as the key.  However, this doesn't let you change the font later on.  By defining all the icons and fonts with "friendly names", I get to swap out the icon font I am using at a later date easily by just changing one file.  This helps if the font you choose suddenly requires licensing fees.

### Defining Icons

When defining an icon, you need to know it's ID.  All icon fonts are based on the web, so the icon ID is stored in CSS.  However, some icon fonts show the icon IDs on their website.  [Material Design Icons](https://pictogrammers.github.io/@mdi/font/5.4.55/) and [Font Awesome](https://fontawesome.com/icons?d=listing&p=2) both provide the icon IDs on their website.  

Your first step is to find the icon you want.  Once you have the icon, look it up within the CSS or the cheatsheet.  You are looking for a four or five hex-digit number.  For example, from CSS:

```css
.mdi-access-point-check::before {
  content: "\F1538";
}
```

The digits you are looking for are `F1538` here.  The cheatsheets have the same number.  I define the icons I am going to use as resources in `App.xaml`:

```xml
<?xml version="1.0" encoding="utf-8" ?>
<Application xmlns="http://xamarin.com/schemas/2014/forms"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             x:Class="ContactsApp.App">
    <Application.Resources>
        <ResourceDictionary>
            <!-- Main Icon Font -->
            <OnPlatform x:Key="IconFontFamily" x:TypeArguments="x:String">
                <On Platform="Android" Value="materialdesignicons-webfont.ttf#Material Design Icons"/>
                <On Platform="iOS" Value="Material Design Icons"/>
                <On Platform="UWP" Value="Assets/Fonts/materialdesignicons-webfont.ttf#Material Design Icons"/>
            </OnPlatform>

            <!-- Icons used by the app -->
            <x:String x:Key="IconSettings">&#xf08bb;</x:String>
            <x:String x:Key="IconAddContact">&#xf0415;</x:String>
        </ResourceDictionary>
    </Application.Resources>
</Application>
```

The number you got for the icon ID is just a hexadecimal character entity and is used as an index to locate the correct character.  You get to do all the normal things here - for instance, you can easily use different icons on iOS and Android.

> **Use different names for all the icons**
>
> I create a distinct icon name for each place I use an icon.  It allows me to differentiate between the locations where the icon is used and update them easily later on.

### Using the icons

Using the icons is as easy as using a [FontImageSource](https://docs.microsoft.com/dotnet/api/xamarin.forms.fontimagesource?view=xamarin-forms) with the color, size, font family, and glyph defined.  Let's take a look at some examples:

```xml
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage xmlns="http://xamarin.com/schemas/2014/forms"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             x:Class="ContactsApp.MainPage">
    <ContentPage.ToolbarItems>
        <ToolbarItem Command="{Binding RefreshCommand}" Text="Show Settings">
            <ToolbarItem.IconImageSource>
                <FontImageSource FontFamily="{DynamicResource IconFontFamily}" Glyph="{StaticResource IconSettings}" />
            </ToolbarItem.IconImageSource>
        </ToolbarItem>
    </ContentPage.ToolbarItems>
    <Grid>
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
        </Grid.RowDefinitions>

        <ImageButton
            Command="{Binding NewCommand}"
            BackgroundColor="OrangeRed"
            HorizontalOptions="FillAndExpand" VerticalOptions="Center">
            <ImageButton.Source>
                <FontImageSource
                    Size="Large"  Color="White"
                    FontFamily="{DynamicResource IconFontFamily}"
                    Glyph="{StaticResource IconAddContact}" />
            </ImageButton.Source>
        </ImageButton>
    </Grid>
</ContentPage>
```

Since it's a font, you can also put the icons in other places, like labels.  It's really up to you.
