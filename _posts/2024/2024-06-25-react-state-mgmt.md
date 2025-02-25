---
title:  "The State of React state management in 2024"
date:   2024-06-25
categories:
  - Web
tags:
  - react
  - javascript
  - comparison
---

I've been away from React development for a while.  I stupidly asked what the best way to create a React app was in 2024 on the [React subreddit](https://reddit.com/r/react), and found that reddit is not a friendly or welcoming community. For those wondering, there are three ways of creating a React app - [Vite](https://vitejs.dev), [Remix](https://remix.run), and [NextJS](https://nextjs.org) - but the community suggests Vite.

Given the somewhat frosty reception I got on the React subreddit, I decided to do my own research across a number of topics for a new project - a social media calendar and scheduling app for my blog. I looked across the main web frameworks (React, Vue, Angular, and Svelte) and chose React as my first one because of my familiarity with its concepts, but also the large ecosystem of tools and libraries.  The next question is "what libraries should I use?"

And here is the first one - state management.

I'm going to limit myself to just 1 hour per library.  This will enable me to see how easy it is to get started in a real world project.  I'm going to score each library across a number of factors:

* Tutorial quality.
* Documentation quality - beginner.
* Documentation quality - intermediate to advanced.
* API documentation.
* Ease of use / developer experience.
* Ease of persisting to in-browser local storage.
* Ease of data-fetching from a remote service.
* Availability of in-browser debugging tools.
* Quality of official sample code.
* Community resource availability.

I'm going to assume all the libraries do what they purport to do - state management.  However, there are changes in mental model and how they do some common tasks.  Everything is weighted equally in this model, but I've included calculations in [my spreadsheet](/assets/images/2024/state-management-2024.xlsx) so you can weight things differently if you so choose.

Who are the contenders?  When I left, Redux was the king of the state management services.  In 2024, I had no clue, so I went to the NPM Weekly Downloads and took the top 7 spots:

* [redux-toolkit] (reducer)
* [zustand] (reducer)
* [xstate] (finite-state-machine)
* [mobx] (observable)
* [jotai] (atom)
* [recoil] (atom)
* [valtio] (proxy)

To that I added the React [useContext() hook](https://react.dev/reference/react/useContext) - i.e. "no real state management" - to see if I even needed a state management library. For each library being considered, I went through whatever tutorials they provided, then I tried to integrate the library into my own project (which has a number of state requirements, including local persistence and data fetching requirements). Interestingly, only two libraries ([react-redux] and [mobx]) are in the list from the last time I did this research.

There are different styles of state management libraries, and I've indicated that in the list above.  The styles relate to the mental model you must adopt to work with the library.  Reducer-style libraries ([redux-toolkit] and [zustand]) use immutable state and the flux pattern (which is a one-way data flow) pattern around a large central state store.  Atom-style libraries ([jotai] and [recoil]) split the central store into many "atoms" - they are more granular and can be updated independently across components.  Comparing these two, atom style libraries are simpler to understand, offer fine-grained control on re-rendering components, and tend to have better performance.  Reducer-style libraries have a single source of truth and are easier to debug.  

Then we have [XState][xstate], which models after a finite state machine.  If you haven't come from academia, you probably haven't bumped into finite state machines before.  The problem, of course, is that JavaScript and React adopt more functional programming techniques, so finite state machines are an unfamiliar pattern that will take some research to understand.  [MobX][mobx], by contrast, uses observables.  Observables are used in reactive programming.  Again, reactive programming is not common in JavaScript and React, so it will take some time and research to understand.  [Valtio][valtio] uses proxies to wrap the state.  I find this confusing and can't say I understand the why behind it yet.  

Everyone has their own favorite library and my ratings won't give the complete picture, especially as I am learning some of these libraries for the first time.  Just looking at the list, you will note that the points scoring will punish any library without great documentation (intentionally). A mentor once told me that if you can't think of anything to do, write some documentation.  I have followed that axiom for most of my career as it is so critical to others understanding what you are trying to do, and vital to the adoption and developer love for a library.

For those that can't wait, here is the table of results:

| Category          | Redux | Zustand | XState | MobX | Jotai | Recoil | Valtio | useContext |
|:------------------|:-----:|:-------:|:------:|:----:|:-----:|:------:|:------:|:----------:|
| Tutorial          | 9     | 1       | 1      | 5    | 4     | 8      | 7      | 1          |
| Beginner docs     | 9     | 1       | 5      | 7    | 3     | 4      | 4      | 4          |
| Intermediate docs | 7     | 7       | 7      | 6    | 8     | 6      | 6      | 7          |
| API docs          | 4     | 0       | 5      | 9    | 0     | 9      | 7      | 5          |
| Dev experience    | 5     | 6       | 6      | 5    | 6     | 7      | 8      | 6          |
| Persistence       | 7     | 3       | 4      | 3    | 6     | 7      | 6      | 0          |
| Data-fetching     | 8     | 0       | 0      | 0    | 6     | 6      | 0      | 0          |
| Debugging tools   | 9     | 0       | 10     | 2    | 8     | 0      | 7      | 10         |
| Sample code       | 2     | 1       | 8      | 4    | 8     | 0      | 4      | 4          |
| Community         | 10    | 4       | 4      | 8    | 4     | 6      | 5      | 10         |
|-------------------|-------|---------|--------|------|-------|--------|--------|------------|
| **TOTAL**         | 61    | 22      | 49     | 44   | 49    | 45     | 47     | 46         |

Now, onto the comparisons!

## Tutorial quality

Every single library on the list has some sort of getting started experience.  A good getting started experience will give the new (to the library) developer an early win of "something working" while demonstrating the basics of using the library - taking the developer from a "context-free" knowledge level to understanding if the library will work for them.  Ideally, the tutorial can be completed in less than 15 minutes.

For the best tutorials, look to [Redux Toolkit][redux-toolkit], which has both a basic "quick start" walk through, an "essentials" tutorial that builds a real-world example application using best practices, then a "fundamentals" tutorial that covers the concepts.  This is the way to do tutorials!  Honerably mentions go to [Recoil][recoil] and [Valtio][valtio], both of which had a basic tutorial that implemented a "todo app" to demonstrate the basics.

The worst ones?  [Zustand][zustand] and [XState][xstate] had nothing more than "here is how to install the library" with no tutorial at all.  Zustand was more interested in saying "here is how we compare to other libraries" rather than educating newcomers, while XState decided that it would wait until later to define any new terms (and there were a lot of concepts you need to understand to use XState).

## Beginner documentation

You've now been through the basic tutorial and figured out that this library will do what you want it to.  Next up, you want to determine how to integrate it into your own application.  You don't want the in-depth details - just enough to get it working without being overly complex.

Again, the stand out here is [Redux Toolkit][redux-toolkit].  It is literally the only one that has a page dedicated to answering the question "What does this library do for you?"  That's a major failing of most libraries I come across.  Aside from that basic question, there is a great tutorial on the fundamentals and a usage guide that goes into details where necessary but gets you up and running in your own app.  Honerably mention to [MobX][mobx] which has "The gist of MobX" that walks through the basics.

Who is bad?  Basically, everyone else seems to write for the developers who wrote the library.  Their documentation is not geared to someone learning the library for the first time.  Again, [Zustand][zustand] is the worst of the bunch - the author doesn't seem to want the beginner to understand the library.

## Intermediate documentation

At some point, you are going to graduate from beginner to intermediate.  You start to want to know how the library works, generally because something did not work the way you were expecting and you had to debug a problem.  Fortunately, intermediate documentation is something just about every library does somewhat well.  To distinguish between the libraries, I looked for specific things - does the library have a good collection of troubleshooting docs; does the library cover concepts and how the library works effectively; does the documentation cover testing.

[Jotai][jotai] was the real standout here with a series of guides on various topics, including performance, debugging, testing, and internals.  The guides were not organized well, but the information was there.  Honorable mention to [Recoil][recoil] which covered both testing and diagnostics, but was lacking in advanced topics.  

The worst, somewhat surprisingly because they have been around for so long, was [Redux Toolkit][redux-toolkit] and [MobX][mobx]. I can somwhat understand Redux because it is actually three libraries (although quite why that should matter to the reader is a problem they have to solve) and a lot of the "good stuff" is not documented alongside Redux Toolkit - you have to go elsewhere on their site. MobX has no excuse.

## API documentation

Once you get through the concepts, eventually you have to get down to the question "how do I use this specific method?"  Questions like what are the arguments?; what is the expected response?; do you have a short example of the method in context?  do you have a good description of errors that can occur?  To judge the API documentation, I value a method by method view of the entire public API surface in one place, with good descriptions, complete argument and response coverage, and in-context examples (or links to examples).  Bonus points for a specific search engine for the API documentation and cross-links from the regular docs to the API docs for when you mention specific methods.

The best API documentation is done by [MobX][mobx] and [Recoil][recoil].  They both meet the general requirements for the API browser, but miss on having a specific API search engine and cross-linkage from the main documentation set.

The worst API documentation?  [Zustand][zustand] and [Jotai][jotai] decided their main documentation set was enough and decided to forego the API documentation.

## Persistence

For some state, you are going to want to persist the state in the browser - generally session storage or local storage.  Consider, for example, the state of your page router.  When the user reloads the page, you want them to go back to the same page.  You have to persist your state somewhere to do that.  The best libraries have persistence built-in or have an officially supported library that works alongside the main library.  The worst don't do persistence themselves and require you to write code.  In the middle - libraries that document how to do it but don't actually do it.

So, who are the best ones? [Redux Toolkit][redux-toolkit] and [Recoil][recoil] both have good documentation for built-in solutions.  I also like [Valtio][valtio], which has a buried documentation section on how to do persistence (and it's easy).  [Jotai][jotai] has a specific atom type with storage, so the API is not good, but it is built in.

Everyone else?  At worst, you have to figure it out yourself.  Sometimes, it's easy.  Most times, you have to rely on third party libraries because there is no direction in the documentation.

## Data fetching

Most applications are data driven.  Eventually, applications will need to interact with some remote API to get data and push it into the local state.  React and GraphQL go hand-in-hand, but REST/CRUD HTTP APIs, tRPC, and other technologies are relatively common.  While state management is fundamentally a different set of problems to data fetching, it's aligned because of the close linkage.  This section is all about how easy it is to integrate data fetching into the state management library.  Is it documented?  Is there a preferred library?

Once again, [Redux Toolkit][redux-toolkit] shows off its maturity because this is solved via [RTK Query][rtk-query] - a library dedicated to this problem.  It's a solid library that has considered most of the problems - caching, optimistic updates, loading state, etc. 

Next step down is [Jotai][jotai] and [Recoil][recoil] - both have a set of integrations that allow them to easily work with external systems.  Jotai has specific integrations on a per-protocol basis (i.e. one for tRPC, one for Tanstack Query, one for URQL, etc.) whereas Recoil as a single integration (Recoil Sync), but the effect is the same.

The other libraries?  You are on your own and there is no documentation to help you along the way.  Some of the libraries do document "async state providers" (although they call them different things), but the end result is the same - you have to do it yourself.

## In-browser debugging tools

State management is also about mutating the state of the application.  Eventually, something will go wrong and you need to inspect the current state of the application to figure out what went wrong.  In addition, you'll find it advantageous to "time travel" - replay the state changes to see when things went wrong.  I'm assuming that the developer has React DevTools installed (so using Edge, Chrome, or Firefox for debugging).

[Redux Toolkit][redux-toolkit] has [Redux DevTools](https://github.com/reduxjs/redux-devtools?tab=readme-ov-file) available and several of the libraries have an integration with this tool.  Others, like [XState][xstate] have their own solid UI-based tooling for handling this functionality.  Others (like [useContext]) integrate quite handily with React DevTools.

The worst experience - [Zustand][zustand] and [Recoil][recoil] have basically nothing or require you to write code in order to debug the state store, although Zustand does have a couple of third-party libraries (I liked [zusty](https://github.com/oslabs-beta/Zusty)) for doing the same.

## Official sample code

Every single library has sample code that you can refer to, but official sample code is something different.  It's maintained along side the main source code for the library and should show off the way that the library is meant to be used.  Ideally, there is a good selection of examples of varying levels of complexity and the library usage is commented so the reader can see what is going on.  To find the examples, I looked in two places.  First, the GitHub repo for the library might have a "samples" or "examples" folder - that's the best scenario.  If the library is a part of an organization, the organization might have a separate "samples" repo for holding these (which is less discoverable and therefore less desirable).

Universally, the example code is not commented.  Good example sets can be found with [Jotai][jotai] and [XState][xstate] in their GitHub repos.  Most of the others rely on the community for providing the samples or just don't bother, assuming the example code in the documentation is "good enough" (clue: it isn't).

## Community resources

Every library in the set has videos and blogs covering the beginner introductions.  Sometimes, they are linked on the main documentation site (and hence have an air of approval from the authors).  In many cases, these introductions are better than the tutorials provided by the library authors.  So I'm not considering those.  However, I do think a strong community is important for any library.  First off, is there a place where developers can ask questions and get answers?  GitHub discussions, Gitter, or Discord is common.  Secondly, is there a more in-depth training course for the library?  I love [FreeCodeCamp](https://www.freecodecamp.org/), but PluralSight, UDemy, Egghead, and others are all in the mix here.  Paid courses are a mechanism by which the authors can share their knowledge and get paid to continue investing in the library.  Finally, does the documentation set highlight the community contributions?  It should!

Pro tip: Look for "github.com awesome-_libraryname_" on Google.  Lots of libraries have a community member that maintains a list of good resources for a library.

So, who has the best community resources? Unsurprisingly, the libraries that have been around the longest are also best at community engagement.  [Redux Toolkit][redux-toolkit] and [MobX][mobx] have figured it out and have strong communities.  Of the rest, [Recoil][recoil] does a great job. The rest are just learning and expanding.  No-one does a particularly bad job here.  It's a case of the age of the library in a lot of cases.


## Final thoughts

So, what would I recommend?

* For large or more complex applications, use [Redux Toolkit](https://redux-toolkit.js.org/) or [Mobx][mobx] depending on your mental model preference (reducers vs. observables).

* For smaller projects, use [useContext] - there is no need for complex libraries.

* If you want to learn something new in a new project, take a look at [Jotai][jotai].

If you want to do your own weightings for the things that are important to you, check out the [Excel spreadsheet](/assets/images/2024/state-management-2024.xlsx) that I used for this post.

<!-- Links -->
[react-redux]: https://react-redux.js.org/
[redux-toolkit]: https://redux-toolkit.js.org/
[rtk-query]: https://redux-toolkit.js.org/tutorials/rtk-query
[zustand]: https://docs.pmnd.rs/zustand/getting-started/introduction
[xstate]: https://xstate.js.org/
[mobx]: https://mobx.js.org/README.html
[jotai]: https://jotai.org/
[recoil]: https://recoiljs.org/
[valtio]: https://valtio.pmnd.rs/
[useContext]: https://react.dev/reference/react/useContext
