---
title:  "The State of React UI libraries in 2024"
date:   2024-06-27
categories:
  - Web
tags:
  - react
  - javascript
  - comparison
---

As I mentioned in my [last post]({% post_url 2024/2024-06-25-react-state-mgmt %}), I've been away from React development for a while, and I'm intending on writing a web application. I'm comparing the various libraries that I can use for my application.  In this post, I'm going to be comparing UI component libraries.

Thus far, my tech stack is:

* Base web framework: [React](https://react.dev)
* Frontend tooling: [Vite](https://vitejs.dev)
* Hosting: [Azure Static Web Apps](https://aka.ms/swa)
* State management: [Redux Toolkit](https://redux-toolkit.js.org)

React UI component libraries are collections of ready-to-use UI elements that help accelerate your application development. There is always the possibility of using primitive web UI design sysems like [Bootstrap](https://getbootstrap.com/) or [Tailwind](https://tailwindcss.com/).  However, I'm then left with writing a whole bunch of HTML and CSS myself, turning them into React components, and then using those to construct my UI.  I much prefer to cut out the basic stuff and jump straight to React components (knowing that I can always revert down to HTML/CSS and my own components if necessary).

So, what are my requirements?

* Like the [state management comparison]({% post_url 2024/2024-06-25-react-state-mgmt %}), I'm going to require great documentation and examples.
* It should be accessible out of the box.
* It should support TypeScript.
* It should be performant.
* It should be visually good out of the box, but still support themes.  I don't want to have to do too many tweaks to get a compelling visual style.
* It should have both a dark and light theme and be able to switch between them in React.
* It should cover the components I am likely to need:
  * Basic components like buttons, menus, form elements, and badges.
  * More complex components like avatars, date pickers, sliders, spinners, and popovers.
  * Data display components like calendars, data grids, and graphs.
  * A great icon set.

Who are the contenders?

* [Ant Design](https://ant.design)
* [Blueprint UI](https://blueprintjs.com/)
* [Chakra UI](https://chakra-ui.com)
* [Core UI](https://coreui.io/react/)
* [Fluent UI](https://react.fluentui.dev)
* [Grommet](https://v2.grommet.io/)
* [Mantine](https://mantine.dev/)
* [Material UI](https://mui.com)
* [Next UI](https://nextui.org/)
* [React Bootstrap](https://react-bootstrap.netlify.app/)
* [Semantic UI](https://react.semantic-ui.com/)

## Documentation and examples

When I look at documentation, I want the following:

* A quickstart that covers how to install the library within my project.
* A basics document that shows off some of the principles - like how to do theming or how to compose your UI.
* A comprehensive listing of components that shows off the design aesthetic (out of the box) and all the parameters.
* Some good examples of the UI framework in use.

The libraries that met all these requirements?  Ant Design, Chakra UI, Fluent UI, Material UI, React Bootstrap and Semantic UI.  However, it should be noted that Ant Design documentation is clearly written by a non-native English speaker, so the documentation is a rough read in places for English-only readers.

What about the others?  Blueprint UI, Core UI, Mantine, and Next UI did not have any examples, which I think are critical to understanding how the components look together. It can be in the form of "here is someone using our stuff" or "here is an example project".  In the case of React Bootstrap, it was an implementation of the base UI framework starter kits.

Grommet had examples (all nicely placed inside a code sandbox), but they were generally weaker than the other libraries.

## Performance, Accessibility, TypeScript, and performance

How do you determine if a library is "performant"?  That's very subjective.  I did, however, have a basic test.  I clicked through the documentation set (on the assumption that the component docs were using the framework in question) to figure out if any frameworks felt slow.  This is subjective, but if I'm clicking on a link and nothing is happening (no loading screen or interactivity to show I clicked), then the user experience is a bad one.  If you don't prioritize user experience in your own docs, then that's a problem.  Generally, the "icons" page in the documentation was the best way of showing this off.

Most of the libraries were performant (with the notably exception of Ant Design).  I could even see how to make the worst offenders work better.  For example, Blueprint UI had an icon list that was painfully slow to load.  However, I could see that doing the basic content first and then loading the icons would have improved the document load times and made the page more responsive, so I chalked this one up to "they didn't design the page well enough for peformance."

Accessibility can also be subjective.  Most (but not all) of the libraries supported aria labels, for example.  However, I wanted something more than the basics.  I wanted a library that discussed accessibility and how they solved for it - maybe even a little something extra in the library beyond aria labels (such as basic color choices, or WCAG tracking).

Libraries that had "above and beyond aria" accessibility:

* Blueprint UI has built-in functionality for highlighting the current target when using keyboard navigation.
* Chakra UI has some of the best accessibility documentation and describes HOW each component is modified for accessiblity.
* Fluent UI had a good discussion of the defaults they provide to make accessible UI and how each component supports accessibility.
* Material UI also indicated how each component supported accessibility (not all of them did).
* Many components for Next UI supported keyboard events.

Most of the libraries were written in TypeScript, so they supported TypeScript even if they didn't explicitly spell it out.  My requirement here was that the library had to either be written in TypeScript or have TypeScript types and a discussion on TypeScript support built into the documentation set.  Grommet and Semantic UI were the only libraries that were not written in TypeScript, and both of them had open issues around TypeScript types.

## Themes

To judge this section, I looked through the documentation again.  Is there good documentation on creating themes? Then I went looking at the code to determine if I could easily switch the theme in code (e.g. through a redux store), use the "system default" theme (which can switch automatically) and if the library provided good out-of-the-box light and dark themes.

It was easier to decide who did this "well".  Lots of libraries relied on your knowledge of CSS, SASS, or Less to do the styling and did not provide an easy way of changing themes.  The libraries that had good theme support include:

* Ant Design (but their documentation was a little complex).
* Fluent UI (the best theme support).
* Mantine.
* Material UI.

The common elements here - each of them had an ability to define your own themes in code with an ability to construct themes by overriding specific elements from other themes, a ThemeProvider context provider that pushed the theme through the entire system, well built light and dark themes that were designed with accessibility in mind, and documentation on how to work with the theme constructs.

## Components

I have some mock-ups of my UI, so I was looking for specific components to be available.  Every UI framework covered the basics, so I'm not going to cover those.  I found Blueprint, Grommet, and Semantic UI missed some of the more complex components I was looking for (most notably, not all libraries had an Avatar component).  Where it got interesting was in what I would consider the "value-added" components - real calendar displays, sortable data grids, and charting.

* Charting: Mantine, Material UI
* Data grid: Blueprint UI, Fluent UI, Material UI
* Date / calendar pickers: Ant Design, Blueprint UI, Fluent UI, Grommet, Mantine, Material UI, Next UI

## Final thoughts

I didn't take styling into account here.  There are several different design systems, each with their own aesthetic.  The only style that was missing was actually "Cupertino" - popularized as the style that Apple products use.  

My top two contenders for UI component libraries implement well known design guidelines: [Fluent UI](https://react.fluentui.dev) follows the Microsoft Design Guidelines and [Material UI](https://mui.com/) follows the Android Design Guidelines.  There are also a couple of up-and-comers that I really liked - [Blueprint UI](https://blueprintjs.com/) and [Mantine](https://mantine.dev/) both cover pretty much everything, although I would have to fill in some gaps.  There are also wrappers around common CSS frameworks, like [React Bootstrap](https://react-bootstrap.netlify.app/) that is built on top of Bootstrap, and [Next UI](https://nextui.org/docs/guide/introduction) that is built on top of Tailwind CSS.

There are actually very few "hard no" libraries in the reviewed set.  There were aspects of each I liked.  However, the purpose of this review was to choose a library that would accelerate my productivity and allow me to quickly prototype a UI for my application.  In case you are wondering, I'm going to prototype my UI with my top two contenders and see which one I like more from a developer experience and visual point of view.
