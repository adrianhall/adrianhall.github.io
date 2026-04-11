---
title:  "Effective prompts for agentic engineering"
date:   2026-04-11
categories:
  - Devtools
tags:
  - ai
---

This is the third and final part of my series on agentic engineering.  If you want to read the first two articles, see the following links:

- [Ten rules for spec-driven design]({% post_url 2026/2026-04-07-agentic-engineer-1 %})
- [Using Jobs-to-be-done to improve your agentic spec]({% post_url 2026/2026-04-09-agentic-engineer-2 %})

In the previous article, I covered the spec, which documents the "what" and the "why" of what you want to build using [JTBD](https://www.productplan.com/glossary/jobs-to-be-done-framework/) as a primary product requirements framework to use.  This is recommeneded because the alternatives rely on a conversation between a product manager and an engineer to fill in the gaps - the agentic developer can't rely on engineer intuition to determine the answers to all the edge cases you are going to uncover - you have to specify them.  By focusing on the objective, you can use best practices and actually imbue the agent with a lot of that intuition.

![Effective prompts for agentic engineering]({{site.baseurl}}/assets/images/2026/Apr11-banner.png)

While a spec will tell an agent what to build, the agent has a primary objective to satisfy the spec with the least resistance, frequently sacrificing the "invisible" pillars of professional software - for example, observability, security, and maintainability - to get to a visual result faster.  By codifying these requirements, you bridge the intuition gap.  You are effectively hard-coding tje senior-level rigor that a human engineer would apply instinctively - such as error handling, schema migrations, and unit tests - ensuring that the agent doesn't just deliver code that works, but code that is production-grade and maintainable.

An agent without a constitution is like a junior dev on three energy drinks: they'll move fast, but you'll spend the next week cleaning up the mess.  The constitution gives them the "Senior" conscience they weren't born with.

## Generating the Constitution with MoSCoW

Left to themselves, agents often "lazy-code" (for example, skipping error handling or comments) because they are optimized for completion speed.  A constitution is a set of rules that prevents agents from taking short cuts for the sake of speed. The framework I most often reach for is the [**MoSCoW**](https://en.wikipedia.org/wiki/MoSCoW_method) framework.  MoSCoW stands for:

- Must-haves
- Should-haves
- Could-haves
- Won't-have

Anyone who has read an Internet RFC will be familiar here: **DO**, **SHOULD**, *SHOULDN'T**, and **DON'T** map to the same concepts.  I have a basic prompt:

```text
Review the spec in @.spec/spec.md.  Create a constitution in .spec/constitution.md using the MoSCoW framework.  This constitution must define the technical standards and engineering intuition that a senior developer work use.

* **Must Have**: List critical Cloudflare-specific standards (for example, D1 migrations, drizzle type-safety, turnstile verification, vitest coverage >80%, and structured logging).
* **Should/Could Have**: List technical enhancements (e.g. KV caching, R2 image optimization)
* **Rules of Engagement**: Explicitly state that the agent MUST NOT skip observability, security, maintainability, error handling to save tokens.
```

This will produce another living document (don't forget *Rule 10: The spec is a living document*) and that it's your responsibility to read and edit it.  Some of the things I normally end up adding include:

- **MUST HAVE**: Abstract service interfaces as close to the service as possible.
  - Adding an abstraction at the service level is a great way to simplify mocking for unit tests.  This addition is something a senior dev would likely do automatically irrespective of if a PM asked or not - it just makes sense once you've been in industry for a while.  
- **SHOULD HAVE**: Automated cache-aside pattern using Cloudflare Workers KV.
  - While a "Must Have" would be the raw D1 database connection, a "Should Have" like this ensures the system isn't hitting the database for every single page load.  This tells the agent that while the system needs to work, it should be optimized for the edge.  If the agent is running low on context (or time), it can priorize the drizzle logic first, but it knows the architectural expectation is to include a KV caching layer for a read-heavy operation.
- **COULD HAVE**: Real-time 'User is typing' indicators via Cloudflare durable objects.
  - This adds polish to an experience, but it isn't required for a functional comment validation and hosting system.  By labeling this as "Could have", you prevent the agent from over-engineering the initial state-management logic.  It signals to the agent: "If we have extra budget in our work breakdown, this is the direction we want to head" without letting it distract from the security of the API.
- **WONT HAVE**: Cross-platform (non-D1) database drivers.
  - Explicitly stating that the system will not support PostgreSQL, MySQL, or other external databases.  This is a crucial guardrail.  Without it, an agent might try to write "agnostic" code or suggest heavy libraries that work in any environment.  This "Won't have" forces the agent to stay lean and hyper-focused on my Cloudflare native stack, ensuring the code remains lightweight and specific to my architecture.

It's the selective ignorance that keeps the agent from hallucinating complex integrations or pulling in unnecessary NPM packages that "might be useful later".

## Generating the work breakdown with GIST

Now that I have the tech stack, the constitution, and the spec available, I have everything I need to start building.  Complex projects, however, will exhaust the context pretty quickly.  Our spec and constitution are not small and they cost tokens.  Breaking the work down into logical phases is another key skill that senior developers use often prior to touching the keyboard.  The key framework here is GIST:

- GOAL
- IDEA
- STEP-PROJECT
- TASK

I focus in on the STEP-PROJECT and TASK (or more appropriately, "Phase" and "Task") where each task can be considered a single GitHub issue.  This is what allows the LLM to focus their attention on the right part of the project.  Fortunately, LLMs (particularly MoE thinking LLMs like Opus 4.6) are really good at this step.  I use the following prompt:

```text
Convert the @.spec/spec.md ionto a work breakdown using the GIST framework.  Use the @.spec/constitution.md and @.spec/stack.md to frame the requirements for the Cloudflare dev platform.  

For each step-project:

1. **Single File**: Create a file in `.spec/work` for the step-project, named `xxx-short-title.md` where xxx is an incrementing zero-padded number.
2. **Atomic Scope**: Each step must be small enough to be completed in a single LLM chat session without compacting the context.
3. **Definition of Done**: Each task must include a definition of done. 
4. **Indicate Parallelization**: Indicate which tasks can be parallelized within a step-project.
4. **Quality Gate**: At the end of each step-project:
  - Additional unit tests for new code should be written
  - TypeScript type check must be clean
  - Eslint should show no errors
  - All tests should pass
  - Minimum 80% test coverage
  - Wrangler dry-run should ensure no more than 1MB bundle size
```

Step-projects protect the context window, ensuring the agent doesn't (for example) forget the database schema while trying to write the CSS.  What the LLM will produce is a flight map - a set of files you can now execute through chat or feed into Squad and tell it to go ahead and build the project.  By forcing the agent to output each step-project into its own zero-padded file, you are creating a chain of custody for the project evolution.  It also prevents the agent from getting "lazy" or "helpful" by trying to cram a 40-task breakdown into a single message.

Let's look at what one of these files looks like:

```text
File: 0002-auth-turnstile.md
Prerequisites: 001-db-setup.md
Status: Pending
Goal: Implement Cloudflare Turnstile verification and basic user identity logic to ensure only humans can post.
Alignment:
- Must use wrangler secret for keys
- Must include vitest integration tests for the verification worker.

Tasks:

1. Infrastructure & Secrets
  - [ ] Provision a Cloudflare Turnstile site key and secret via Terraform
  - [ ] Run `wrangler secret put TURNSTILE_SECRET_KEY` for the dev environment
  - [ ] Update `wrangler.jsonc` to include the Turnstile site key as a public variable
2. Worker Implementation
  - [ ] Create a utility function `verifyTurnstileToken(token, ip)` using the fetch API to hit Turnstile siteverify endpoint
  - [ ] Modify `POST /comment` endpoint to extract the cf-turnstile-response header
  - [ ] Implement a 403 Forbidden response if the token is missing or the verification fails
3. Constitution checks
  - [ ] Error Handling: Ensure the worker doesn't crash if the Turnstile API is unreachable; default to fail-closed
  - [ ] Observability: Log successful and failed attempts, including Turnstile error code
  - [ ] Tests: Write a vitest suite that mocks the Turnstile API respone to test both success and failed scenarios

Definition of done:

- [ ] curl request to the endpoint without a token returns a 403
- [ ] curl request with a valid (mocked) token returns the expected next-step response
- [ ] All tests pass with 80% statement coverage
```

Because this file is focused and small, it's a great context anchor.  You aren't concerning the LLM with all the other things it COULD be doing - just do this one thing.  This keeps the context smaller, even when it gets fed the entire spec, stack, and constitution to figure out what this task means.  Finally, checkbox logic - agents are remarkably good at following markdown checklists.  It provides a clear success metric for the session.

## Some final cleanup

Don't forget to include a `AGENTS.md` or a skill that you use to execute a step-project.  This allows you to integrate a process that the agent MUST follow.  Mine includes:

- **ALWAYS** create a new, dedicated git worktree (eg. `git worktree ad ../00X-branch 00X-branch`)
- **ALWAYS** commit, following the Conventional Commits specification
- **ALWAYS** run `npm run quality-gate` - fix any errors found and ensure a minimum of 80% code coverage
- **ALWAYS** Push the branch and generate a PR description that references the original JTBD from .spec/spec.md once the quality-gate is green

Obviously, there is more that goes into a skill or AGENTS.md file than this, but putting the process at the front ensures that MULTIPLE agents can be operating together and that all the check-ins and pull-requests will be generated for you.  You become the code reviewer (unless you want to delegate that to an LLM as well).

## Final thoughts

When I started this series, the goal was to find a way to move from vibe-coding to agentic engineering - to make AI agents more than just sophisticated copy-paste machines.  We've moved through the three distinct layers of discipline:

1. The Spec - replacing vague user stored with Jobs-to-be-done to give the agent a deep understanding of why the product exists.
2. The Constitution - using MoSCoW to codify the senior engineer intuition that agents natually lack, forcing common complaints like observability, security, and maintainability into the project "must-haves".
3. The Work Brekdown - leveraging GIST to break the work into verifiable step-projects that keep the agent grounded in a manageable context window.
4. The Cleanup - creating a skill that codifies the process of writing code, allowing parallelization of effort.

The transition from vibe-coder to an agentic engineer isn't about writing less code; it's about writing better instructions.  We are shifting our primary output from lines of syntax to the creation of high-fidelity blueprints.

By applying these product management frameworks to our technical specs, we aren't just making it easier for the agent to execute; we are making it possible for us to scale our own expertise.  We've turned the intuition gap into a documented standard.  Whether you (like me) are building a native comment system on Cloudflare or a complex enterprise backend, the lesson remains the same.

- If you can't specify it, the agent can't build it.
- If you can't put in basic guard rails, you can't trust the agent to lead

This framework doesn't remove the human from the loop; it elevates the human to the role of architect.

One thing I've found useful is to have a robust repository template that is AI focused.  By including the technology stack and constitution along with a solid set of skills and agent definitions for YOUR environment, you'll find the process becomes "write the spec and let the agent do the work".  The repository template is both technology stack and organization specific.  What works for the Cloudflare dev platform won't work for a mobile app, and we can't expect it to.  However, what works for your group of engineers in a company does not necessarily translate to someone elses group either.  I suspect we'll get to the point where the "Cloudflare dev platform AI repository" template can be created and then modified on a per-organization basis.

But that's something for another time.

### AI Disclosure

I wrote this article (as I do all my articles) by hand.  However, the images are generated by Google Gemini.
