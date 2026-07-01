---
title:  "Testing in an agentic coding world"
date:   2026-06-07
categories:
  - Devtools
tags:
  - ai
---

I've been writing a lot of code with agents recently.  I've even written about a lot of it here.  However, I began to notice a pattern about testing.  My LLM is great about writing tests to go along with their code and it always ends up with a 100% pass rate.  Most of the time, the LLM will modify their tests when they do fail the first time.

![Testing in an agentic coding world]({{site.baseurl}}/assets/images/2026/Jun07-banner.png)

But what am I really testing here?

Agents write code to get to completion as quickly as possible.  It's built into their DNA - please the user as fast as possible.  So I get it to write tests, and the tests pass.  Agents write tests to validate their version of reality, not the user requirements.  The perform what I call "Confirmation-Driven Development".  Write the tests so that the code you have written gets exercised, but not validated.

## A brief history of testing

Software testing has a very very long history, but it can be broken into four eras.  I'll push forward the idea now that we are entering a fifth era, but I'll get onto that later.

### ERA 1: The birth of software developing and "debugging"

In the earliest days of computing, "testing" wasn't a distinct discipline.  Programs were punches into cards and fed into room-sized mainframes.  This was really the ultimate "test in production".  If the code crashed or produced the wrong math, you debugged it.

Proving correctness was done mathematically on paper before writing the code.  Edsger Dijkstra famously noted that "testing cases can highly effectively show the presence of bugs, but never their absence."

### ERA 2: The waterfall method and the rise of Integration Testing

As software projects grew massive, the industry realized it needed structure.  The term "Software Engineering" was coined and formal testing frameworks emerged.  Winston Royce introduced the Waterfall Model in 1970, and testing became a distinct phase at the _end_ of the development lifecycle.

This period popularized the "V-Model" of development, which explicitly paired development phases with testing phases.  This is where the concept of "unit tests" was introduced: verifying individual modules or subroutines in isolation, written by the developer.  Integration testing (putting modules together to ensure the interfaces between them worked) was often done by a separate QA team.

### ERA 3: The Agile revolution and unit-test frameworks

Until the late '90s, unit testing was highly manual, tedious, and often skipped.  You had to write custom scaffolding code just to test a function.  Then came Kent Beck and the Agile/Extreme Programming (XP) movement.  In 1996, Kent Beck and Erich Gamma wrote JUnit for Java.  This was a massive turning point since it gave developers a standardized, automated framework to write and run unit tests easily.

Kent Beck formalized **Test-Driven Development** (TDD).  TDD flipped the playbook used by the waterfall model on its head.  Instead of testing at the end, you wrote the test before the code.  The tests became the design spec.

### ERA 4: DevOps and the Testing Pyramid

As software moved to the cloud, the industry shifted from releasing software once a year to multiple times a day.  Manual testing became a bottleneck, leading to the rise of DevOps.  The testing pyramid argued for a massive base of fast, cheap, automated unit tests, a smaller middle layer of integration tests, and a tiny top layer of end-to-end UI tests.  Continuous integration suites ran the automated testing suites as developers pushed code into repositories.  If a unit test broke, the deployment halted.  Tests became the gatekeepers of production.

This is where we were six months ago.  In my own work, I use [vitest](https://vitest.dev) for unit tests and integration tests, and [Playwright](https://playwright.dev/) for end-to-end UI tests.  I can test components in isolation from the rest of the UI and get to about 90% coverage with not too many problems.  The extra 10% is likely not worth it because it covers paths that are only achieved when there are environmental problems, defensive guards, and paths that are rarely exercised but complex to test.

## The problem with agentic testing

Then came the coding agents.  2025 was a seminal year for agentic coding.  Up until this point, tests were written to validate that the code met the spec.  As agents took over the test writing, tests came to validate that the code was what the agent meant to write - the spec was no longer considered.  The agents' goal is to maximize the probability of a "green" test suite to signal task completion.

What if the agent misunderstands the intent?  It writes a brilliant, highly deterministic test that perfectly validates its own misunderstanding. AI agents don't write tests to find bugs; they write tests to prove they did what they _thought_ you asked for.  That's why I call it "Confirmation-Driven Development".

## Prevent code rot by returning to Agentic QA

When a single agent handles both the test and the code, the intent shifts to match the easiest path to completion.  When Kent Beck introduced test-driven design, he locked the intent (the specification) into the tests early.

We need to return to the concept of a separated QA department and Test-Driven Development.  We do this by utilizing multiple adversarial agents, each with their own job, with a human as the architect and judge.

My setup includes three agents, and the development cycle flows like this:

1. The human (me) defines the project.
2. **Agent M** (yes, I use James Bond references) takes the definition and creates a solid engineering specification.  I have a skill that determines what is in an engineering spec (and it's the same stuff that was in an engineering spec before coding agents came along).
3. The human (me again) reviews the engineering spec and makes adjustments - I'm still using a chat interface and **Agent M** for this, but the document is carefully reviewed and updated.
4. I ask **Agent M** to also produce the interfaces to all modules - this requires a lot more systems architecture knowledge than vibe-coding requires; take the time to do this properly.
5. I ask **Agent Q** to create integration tests for the specification.  All the tests will fail, but that's the point.
6. I ask **Agent X** to write the code to the specification, and to run the tests at the end - fix the code or identify that the test is wrong (and why the test is wrong).
7. Finally, I go through all the "wrong tests" and make a judgement call - instructing **Agent Q** to fix the tests or telling **Agent X** that they are wrong and to fix the code.

Importantly, **Agent Q** does not touch code and **Agent X** does not touch the tests.  This clear separation between roles with a human arbiter to judge the contradictions provides for a better test suite and gets us back to the original intent of testing in the first place - does the software produced meet the specification written.

## Final thoughts

You may be thinking here "_If both agents use the same LLM, won't they just make the same mistakes?_" It's a great point, and had me using different LLMs for a while.  However, the point here is contextual isolation.  Each agent has their own prompt, their own sequence, their own purpose. For example, the prompt for Agent Q (the test agent) tells them they will be penalized if a bug slips through, while Agent X (the coding agent) is penalized if the test suite fails. Their objective functions are diametrically opposed.  If they disagree, then that is exactly where a human needs to step in as a judge.

Multi-agent setups like these are critical to the production of well-engineered software.  We move testing back to specification verification, allow different personalities of agents to write their bit of the puzzle and move on.

Indeed, multi-agent facilities are now built-in to the major coding platforms, like [OpenCode](https://opencode.ai/), [Claude Code](https://code.claude.com/docs/en/overview), and [GitHub Copilot](https://github.com/features/copilot).  There are also independent packages like [Squad](https://bradygaster.github.io/squad/) that are explicit about the agent breakdown.

Whatever you use, don't let the agents take over your spec.  Software developers need to think more like software architects and drive agents towards implementing and testing the spec.
