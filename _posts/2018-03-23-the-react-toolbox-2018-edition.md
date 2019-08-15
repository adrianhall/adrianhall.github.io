---
title: "The React Toolbox - 2018 Edition"
categories:
  - "Web Development"
tags:
  - React
  - JavaScript
---

It’s been a while since I’ve been in web development. I’ve mostly been concentrating on mobile development instead. But this week I had cause to delve again. It’s amazing how much change there has been in the web world in just 2 years. React has become the force in web frameworks (Sorry Angular — I never liked you anyway) and developers are starting to coalesce around libraries and tools that make their life easier. I thought I’d put down my thoughts on the libraries that are making it into my latest project.

## Scaffolding: cerate-react-app

There are so many scaffolding options for react, mostly driven by either [Yeoman](http://yeoman.io/) or Ignite. However, the scaffolding options tend to be very opinionated and those opinions don’t match my own. As a result, I use [create-react-app](https://www.npmjs.com/package/create-react-app). It produces a very basic version of the app and allows me to get started very quickly, which is what I tend to want. It’s got packaging with Webpack, support for the latest features of BabelJS, and PWA support built right in. I love that the latest features of JavaScript (like async/await or spread operators) can be used right out of the box.

That isn’t to say it isn’t without problems. Among the many problems it has — no support for TypeScript, CSS Modules, SCSS/LESS, or really any pre-processors. If I’m doing a more complex app, I’ll want something like that. These support issues are sometimes correctable (for example, I can use TypeScript relatively easily using a different set of scripts). However, if I’m doing a big project, I’ll eject from the project and make it a typical project.

## State Management: Redux

Since my last delve into web development, Redux has emerged the victor in the local state management world. I used to consider other libraries (either write my own, [MobX](https://mobx.js.org/), or [Alt](http://alt.js.org/)), but I’ve been using Redux extensively in my React Native work and I don’t see any reason to change for the web. It seems that every other library that needs to implement state management integrates with Redux first and anyone else is a maybe but probably not. This makes the choice for state management easy.

I have moved over to a slightly weird / non-traditional setup for my Redux boilerplate. I place my reducer and action creators in the same file. I think this will require some explaining, so I’ll do that in another blog post some time.

## Routing: react-router v4

If ever there was a case of “game over — there is a winner” — this is it. Internally, we all want apps with different pages and to the ability to route between them. [React Router v4](https://reacttraining.com/react-router/) does one job and does it well. I found the leap from react-router v3 to react-router v4 a little jarring, but it wasn’t that hard to make the jump over all.

## CSS Library: Bootstrap

This is one decision that has not changed for me. When I’m looking for an overall CSS library, [BootStrap](https://getbootstrap.com/) is still the go-to for me. However, I no longer write HTML and CSS for this. Instead, I use [React-Bootstrap](https://react-bootstrap.github.io/). This turns all the CSS classes and components into React components. That makes Bootstrap much more easily consumed.

## CSS Styling: The jury is still out

I did a re-spin of all the ways to style a component — from importing CSS and CSS Modules to libraries like [Radium](http://formidable.com/open-source/radium/), [Glamor](https://github.com/threepointone/glamor) and (most recently) [styled-components](https://www.styled-components.com/). They all “feel” wrong. Part of the problem here is that I like [SCSS](http://sass-lang.com/) or [LESS](http://lesscss.org/) as pre-processors. Partly, I like separated stylesheets. (embedded style sheets in React Native feel wrong as well).

Unfortunately, I’m forced to choose. As a result, I’m likely to just import a CSS file (potentially generated from SCSS or LESS, per the instructions within create-react-app). However, it just feels wrong. I’d love to hear what others are doing and why you chose the library / mechanism you did.

## Development Tool I can't be without: Storybook

One of the great benefits of using React is that every component can be used and tested on its own. One of the great problems of using React is developing components in isolation is hard, requiring that you use a separate app for each component just to develop it. Storybook is a solution to that problem. It allows you to write a component in isolation.

## Component Library: There are too many (but use js.coach)

Confession time here. I’m a UI Component Library junkie. I love trying out all the new ones. There are some that rise to the surface, mostly because of my mobile work. I love [Material-UI](http://www.material-ui.com/), for example, and I’ve already mentioned React-Bootstrap.

When I am looking for a component to do something (even if I know something exists in one of my favorite libraries), then I go to [js.coach](https://js.coach/) and look for it. I will normally find something new to try out and that may be more performent, look better, or fit better with my functionality requirements.

What is rare these days is to write my own components from scratch — at least for the basic ones. I still do a lot of component composition for higher level, app specific components, but the lower-level components are always pulled from a library.

## Communication to the backend: GraphQL

In the beginning, I would write awesome REST APIs in the cloud to manage my data in between runs. For data, I would consider the complex world of OData. Then I would have HTTP POST/GET for the random things that happen in the middle (like submitting to a queue, for example).

In the interim, GraphQL has come out. GraphQL is a nice little language that is optimized for user interfaces. Each element of the UI can ask for what it needs, and only that information gets returned. This is amazingly useful for mobile apps where bandwidth and data consumption is an issue, but it’s also useful for web applications as well. GraphQL also includes mutations (something that OData never did) which means that when I am working with data tables, this is a preferred way to do it. It can replace both the HTTP POST and PATCH for data mutations, the HTTP GET for data retrieval and the random HTTP POST/GET requests for doing random stuff (although doing this latter stuff within GraphQL doesn’t seem like it is a benefit — merely using the same API instead of spinning up another API).

## Cloud Services: AWS Mobile

No secret here as I work for them. AWS is the biggest cloud provider out there and, especially with the introduction of [AWS AppSync], has all the facilities you need to build a complete Serverless mobile backend, complete with identity, analytics, structured and unstructured data, push notifications, machine learning, AR/VR, chat bots, etc. For accessing these resources, use [AWS Amplify] — an open source library for doing all the cloud operations in a standard way. Given the one-stop shop for all the services, I don’t really see a need to go elsewhere.

Web development has been solidifying for a number of years now, and its great to see the tooling, libraries, and patterns progress as the React ecosystem matures.

{% include links.md %}
