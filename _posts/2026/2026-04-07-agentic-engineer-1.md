---
title:  "Ten rules for spec-driven design"
date:   2026-04-07
categories:
  - Devtools
tags:
  - ai
---

Recently, I posted an article on the [AI Maturity Model]({% post_url 2026/2026-04-03-ai-discussion %}).  In that article, I proposed six stages of growth - from tactician or accidental editor to the agentic engineer.  The biggest difference between the vibe coder and the agentic engineer is the transition to spec-driven design.

But what does that really mean?

The "vibe-coder" sends one massive prompt and hopes for a miracle.  The agentic engineer understands that an LLM is a reasoning engine, not a magician.  Spec-driven design provides three distinct documents: a **Constitution**, a **Specification**, and a **Work Breakdown**.

![The three pillars of agentic engineering]({{site.baseurl}}/assets/images/2026/Apr07-agentic-engineer-1.png)


* Without the Spec: The agent will build a "technically correct" feature that doesn't actually solve the user's problem.
* Without the Constitution: The agent will suggest a library you hate or use an insecure pattern.
* Without the Breakdown: The agent tries to write thousands of lines of code at once and hallucinates halfway through.

It doesn't matter if you are using [OpenSpec](https://openspec.dev/), [SpecKit](https://github.blog/ai-and-ml/generative-ai/spec-driven-development-with-ai-get-started-with-a-new-open-source-toolkit/), or [AWS Kiro](https://kiro.dev/) for your spec-driven design.  The concepts and the requirements are the same.

The real insight here is that you have to be a cross between an architect and a product manager to be really good at this.  Technical Product Manager (and I have been one for many years) are great at this stuff.  I believe _Product Architect_ is the role that will be advertised for this specific skill set in the future.

## Pillar 1: The Spec (The Intent)

This is the "North Star" for the agent, defining the specific feature or product value and behaviour.

### Rule 1: Use a PM Framework

Before defining a single UI element, anchor the spec in the user's struggle.  Use a framework like **Jobs-to-be-Done** to explain the context: "When I am [X], I want to [Y], so I can [Z]".  When the agent understands the intent, it makes better autonomous decisions when it hits a logical fork in the road that the spec didn't explicitly cover.

This is so important that I'm going to follow up on this topic in a future article.

### Rule 2: Explicit State Machine Mapping

Agents excel at the "happy path" code but struggle with edge cases.  This is why vibe-coding a proof of concept works wonderfully but taking it to production fails in so many cases.  You must explicitly define the feature states, like idle, loading, success, and error.  By mapping these transitions in the spec, you force the agent to write defensive code and robust error handling rather than optimistic, fragile snippets.

### Rule 3: The "User Persona" Context

An agent building a tool for a "Senior devops engineer" should write different code (and UI) than one building for a "First-time retail customer."  Providing the persona ensures the agent defaults to the correct level of complexity, terminology, and UX friction without you having to prompt for it every time.

## Pillar 2: The Constitution (The Constraints)

These are the "global rules" that ensure the agent stays within your specific tech stack and architectural style.

### Rule 4: The "Single Source of Truth" Schema.

My first spec forgot to define the data schema.  I ended up with one table with an auto-incrementing `user_id` and another with a `unique_user_id` that was a UUID.  Needless to say, things broke.  Including the Drizzle `schema.ts` directly would have solved this.

You constitution should define the data shapes before any logic is written.  By grounding the agent, you prevent logic drift where the agent hallucinates database columns or variable types that don't exist.

### Rule 5: Explicit Tech Stack and Dependency Declaration

Specify your version (e.g. _Next.js 15, Drizzle ORM, CloudFlare D1_).  If the constitution is vague, the agent will default to its oldest training data.  Hard-coding your stack preferences ensures the agent doesn't introduce hallucinated libraries or outdated patterns that break your build.

You should also provide coding style decisions - e.g. JSDoc requirements, one class per file, or whatever else your organization (or you) decides is important.

### Rule 6: The "Out of Scope" List.

Telling an agent what **not** to do is often more important than telling it what to do.  Use the constitution to set "negative constraints" - such as _No external CSS libraries_ or _No Node.js built-ins (v8 isolates only)_ (which is great for my current focus in CloudFlare development).  This keeps the agent output lean and compatible with your tech stack.

## Pillar 3: The Work Breakdown

This is the "Task Graph" that prevents the agent from losing its place during complex builds.

### Rule 7: Modular Decomposition

Never ask an agent to build a "Page".  Ask it to build a "Module".  Break your feature into atomic units - schema, data access layer, and UI components.  This prevents context collapse, where the agents reasoning degrades because the context size has exceeded its cognitive window.

### Rule 8: The Definition of Done (Verification)

The work breakdown must include explicit acceptance criteria for every task.  This allows the agent (or a secondary reviewer agent) to verify the work.  If the agent can't test its output against your criteria, it isn't finished.

### Rule 9: Error handling and Edge Cases

Every task in the breakdown should account for failure states.  By itemizing specific edge cases (e.g. "what if the D1 query times out?"), you ensure the agent builds a resilient system rather than a "vibe-based" prototype that only works under perfect conditions.

## Rule 10: The Living Document

Spec-Driven Design is not "waterfall" development.  As the agent codes, it will discover ambiguities or technical blockers.  Rule 10 dictates that you **update the spec or constitution first**, then let the agent retry.  Never patch the code manually to fix a logic error; fix the instruction so the agentic loop remains clean and reproducible.

You are responsible for each stage.  It's alright to defer some of the work to a reasoning LLM, but you are responsible for the output.  Each document should be as concise as possible while still meeting the requirements for the document.  I have found that LLMs love to be verbose.  One project I was on, the LLM decided to say "Support RFC x" and then include the RFC in the document.  Remember, size is context and context is token cost.

Of course, after every change, you should ask the LLM to analyze the work breakdown, particularly the things that have not been started yet, to ensure that other changes are not needed.

## Final thoughts

As I was writing this, I realized that this is just scraping the surface of this topic, and I will go into detail in future articles, especially the in-depth analysis for the rules around the spec itself.  Thinking like a product manager does not come naturally to most people, so it's time I wrote it down.

If you are doing this right, you will be spending much more time on these documents than on running the AI.  A recent project I completed is a good example.  I spent over three weeks working through all the issues with the spec.  Once I was happy, the product was done in less than two days, and could have been done faster if I had enabled a squad to help me orchestrate the work.

Product thinking is not free, but it is the difference between a vibe-coded proof-of-concept and a maintainable production-ready product.

### AI Disclosure

I wrote this article (as I do all my articles) by hand.  However, the images are generated by Google Gemini.
