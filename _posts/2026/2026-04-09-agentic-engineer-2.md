---
title:  "Using Jobs-to-be-done to improve your agentic spec"
date:   2026-04-09
categories:
  - Devtools
tags:
  - ai
---

In [my last article]({% post_url 2026/2026-04-07-agentic-engineer-1 %}), I introduced ten rules to help you move from vibe-coder to AI Product Architect by building specs.  However, I glossed over a few of the details.  The basic problem is that most people write specs for themselves - a human.  You aren't writing for a human, so why do you think it would look the same? In this article, I'm going in depth into using product management techniques to write a spec that agents love.

![Transition to JTBD Image]({{site.baseurl}}/assets/images/2026/Apr09-banner.png)

Rule 1 (refined) is "**Don't just tell the agent what to build; tell it the circumstance in which the user is struggling.**  The "Job" is the prompt; the "Code" is the solution to that job."

Product managers use frameworks to help organize data from users so that the spec is more meaningful.  We often use "user stories", which get put into JIRA or another issue tracking system, to communicate the requirements.  Weeks of research are done to make these really concise.  Some of the common PM frameworks that you shouldn't use include:

* The [**PR-FAQ**](https://productstrategy.co/working-backwards-the-amazon-prfaq-for-product-innovation/) - a favorite at Amazon; it starts with a really good press release that describes the customer benefits from the product or feature and follows on with a number of frequently asked questions that would describe sales scenarios.  This is **NOT** a good fit for agentic development as it doesn't go into enough detail about the product or feature to be reasonable, even with tight writing.  It's more at home as a mechanism to decide if a project should be approved.
* [**Hypothesis Progression Framework (HPF)**](https://medium.com/@tlowdermilk/customer-driven-engineering-part-1-the-culture-97601b5f65ed) - a favorite at Microsoft Developer Division; it starts with a hypothesis and then you run experiments (data analysis, customer interviews, user studies, etc.) to disprove the hypothesis.  If it's standing at the end, it's a good hypothesis.  These are great for gathering information about features, but lack finesse when directing an LLM.
* [**User Stories**](https://www.productplan.com/glossary/user-story/) - definitely useful, and I started down this path myself.  The classic "As a \[user\], I want to \[do something\], so that \[value occurs\]".  User stories are for humans.  They are great for a JIRA board where a human can fill in the gaps with intuition and discussion with a product manager.  But an LLM lacks intuition; it only has context. The AI focuses on the **Feature**.

So, what **SHOULD** you use:

* [**Jobs-to-be-done**](https://www.productplan.com/glossary/jobs-to-be-done-framework/) (JTBD) - "When I am \[situation\], I want \[outcome\] so I can \[progress\]" - this is the central framework I recommend for agentic specifications.  The LLM focuses on the **Situation** and **Outcome**, instead of a basic understanding of a feature.
* [**CIRCLES**](https://www.productplan.com/glossary/circles-method/) (Comprehend context, identify customers, report needs, cut through priorities, list solutions, evaluate tradeoffs, summarize).  While we use JTBD for the 'Why,' we use the **Comprehend Context** step of CIRCLES to define 'Who' (which is rule 3). Knowing the 'Who' and the 'Why' together creates an unbreakable context for the LLM. The **Evaluate Tradeoffs** step is the secret sauce that makes it useful for agentic specifications.  If you explicitly list tradeoffs in your spec (e.g. "Prioritize developer readability over micro-optimizations"), the AI will stop using overly complex code and stick to clean maintainable patterns.

These are the ones worth knowing while writing an agentic specification.  Frameworks like [MoSCoW](https://en.wikipedia.org/wiki/MoSCoW_method) and [GIST](https://www.productplan.com/glossary/gist-planning/) are better suited for the constitution and work breakdown, which I will tackle in future articles.

Let's take a couple of examples.  I am currently building an AI-driven comment system for [EmDash - the Cloudflare Wordpress replacement](https://blog.cloudflare.com/emdash-wordpress/).

## Example 1: Submitting the comment

I might write this as a user story: `As [a reader], I want to [write comments] so I can [engage with the community]`.  It's snappy, tells the story.  The software engineer will generally have a discussion with the PM asking about design of the feature and the nature of how comments flow through the system and come up with the right thing.  The LLM will build a standard CRUD form and add a submit button.

Using **Jobs to be done**: `When I [finish a compelling article], I want to [share my rebuttal immediately without losing my flow], so that [I feel my voice is heard in real-time].`  It's a completely different vibe to the user story.  The LLM can act on this completely differently.  This feature needs streaming responses or optimistic UI because "flow" and "real-time" are the priorities.

## Example 2: Rejecting Spam Comments

Again, let's write a user story: `As [an author], I want to [reject spammy comments] so [my readers feel safe engaging with the community].`  Again, it's snappy and tells the story.  The LLM will add an `IsSpam` boolean to the database and add a delete button to the comments.  Technically, it meets the requirement.  We don't get to discuss the expectations with an LLM - they read the requirement and act.

Let's re-write this using **Jobs to be done**: `When I [receive a submitted comment], I want to [automatically classify the comment as spam or not-spam and quarantine the spam comments], so that [my readers are not overwhelmed by irrelevant comments]`.  Here, the goal is "automate triage".  The agent will be incentivized to create a pre-save hook, inject a classification step, and route noise to a quarantine table.

## Example 3: Using State Machines

We can also illustrate the state machine for this feature.  When you add a state machine to the spec, you are essentially telling the agent "The comment doesn't just exist; it moves through a lifecycle." and that can help frame the HOW in addition to the WHY.  Instead of just asking for an Insert function, you define the transitions:

```text
stateDiagram-v2
    [*] --> New: Reader Submits
    New --> Triage: classifyComment()
    
    state Triage {
        [*] --> Analyzing
        Analyzing --> Approved: High Signal
        Analyzing --> Quarantined: Low Signal
        Analyzing --> ManualReview: Fail/Ambiguous
    }
    
    Approved --> UI_Live: Optimistic Sync
    ManualReview --> UI_Pending: Notify Reader
    Quarantined --> [*]: Silent Remove
```

As you can see, there can be a lot of transitions to discuss when you consider a full lifecycle of a comment.  I haven't handled all of them - manual review can also move to quarantined or approved through a button press - what happens then? Defining the flow via state machines is great for nailing the logic when it matters most.  

If you are looking for a better tool to represent state machines, use the [mermaid diagram language](https://mermaid.js.org/syntax/stateDiagram.html) (as I have here).  It supports state machines, is AI readable, but renders inside the documents readily for human readers.  It also forces logical rigor (unlike a drawing tool like Figma or LucidChart).  If there is a logic error, the mermaid chart won't compile into an image.

## Example 4: Trade-offs

If you don't give the LLM some guidance in the spec, it will likely re-use other parts of the spec or previous projects (via memory) when building the comments entry box.  You might end up with a heavy weight markdown editor instead of a simple clean text box.  Using the "Evaluate Tradeoffs" from the CIRCLES framework is a great way to put some guard rails on this.

* We priortize **Speed (Low Latency)** and **Data Integrity** over **Rich Text Features**.  The agent should focus on optimistic UI updates and instant Cloudflare D1 writes; markdown support is a 'Should-Have', not a 'Must-Have'.

The agent now knows that if it has to choose between a heavy library for emojis or a lightweight library for speed, it must choose speed.

## Final thoughts

When you shift from a user story to a JTBD, you move from requesting a feature to requesting a system behavior.  You provide more context.  LLMs do a better job when you provide context over expecting engineering intuition.  You move from prescriptive specs to descriptive architectures.

If you did start with user stories, all is not lost.  Remember rule 10: _The spec is a living document_.  

Vibe-coding way: "Actually, don't build the form that way.  Start with an AI-driven spam classification middleware" during a chat session.

Product Architect way:

* Stop the chat session
* Revert the change
* Fix the intent by putting a better Job-to-be-done in the spec.  

Most of the time, the agent isn't dumb; it's just working towards the wrong job. Fixing the job ensures that the agent doesn't re-hallucinate the exact same form next time because the job has been updated.

### AI Disclosure

I wrote this article (as I do all my articles) by hand.  However, the images are generated by Google Gemini.
