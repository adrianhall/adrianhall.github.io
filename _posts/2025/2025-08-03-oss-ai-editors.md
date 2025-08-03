---
title:  "AI Assisted Editors: A Comparison (Part 2)"
date:   2025-08-01
categories:
  - Devtools
tags:
  - ai
---

In [my last article]({% post_url 2025/2025-08-01-ai-editors %}), I provided a break down of the three commercial AI assisted editors that I use on a regular basis.  However, not everyone can afford a monthly subscription ($20/month is the standard), or maybe you prefer different models or a more local development environment.  There are still options that you can use to get AI assisted editing yet avoid the subscription charge.

<!-- more -->

## What is an AI Assisted Editor?

As a developer, you open up an editor and write code.  AI Assisted editors generally come in the forms of plugins that allow a large language model to take over the writing of the the code for you.  They also generally provide information on the code base through a chat interface.  This allows you to come up to speed on a new code base quickly by allowing the AI to focus your attention on the important bits.

I've personally used three AI assisted code editor plugins to create the same project (a web UI for Azurite) three times.  The projects ended up very different even if they used the same AI models underneath.  That's not unusual and a natural part of the development process with AI.  If you don't like the solution, you can ask the AI to change or you can just wipe it out and start over.

I started off writing the project myself without AI.  This took me a month, mostly because I'm not the best at designing UI and TailwindCSS is complex.  But that's a baseline.

## The plugins

So, who are the contenders?

* [GitHub Copilot](https://github.com/features/copilot) - yes, again!
* [Cline](https://cline.bot/)
* Others I've tried.

These are all available as extensions on Visual Studio Code.  If you want to get going with local models, you're also going to need [Ollama](https://ollama.com/) running on a GPU-enabled desktop with plenty of memory, and a model.  I used qwen2.5-coder:1.5b for my tests, but you will need to explicitly choose a model.

You can easily install Ollama like this:

```bash
winget install ollama.ollama
```

Then install and run the model:

```bash
ollama run qwen2.5-coder:1.5b
```

Unlike the subscription services, not one of the options allowed me to complete my project without a significant amount of coding.  The OSS AI assistants were more helpers on the side than a serious AI coding partner.

> **A note to the Enterprise IT Admins who turn off options in Visual Studio Code**
>
> Why?
>
> If you allow downloading extensions, but turn off features of GitHub Copilot, you are just asking your developers to work around your restrictions.  They will do what is needed to get their job done.

## GitHub Copilot

Didn't we do this last time?  Well, yes - I did talk about the subscription services last time, but I glossed over the experience when you go local.  When you use the drop-down (err - drop-up?) to edit the model, you get this:

![Model selector in GitHub Copilot Chat](/assets/images/2025/08/2025-08-03-copilot-models.png)

Notice the "Manage Models" option.  That allows you to install new models from providers that aren't GitHub Copilot. If you are running Ollama, you can select that and see the list of models that are available to you.  Once selected, you can then run Ask, Edit, or Agent mode.  If you have Azure credits, you can run a model in the Azure AI Factory and connect to it using the Azure option.  There is also support for OpenRouter (which is a pay-as-you-go option for consuming LLMs).  You can also choose models from any of the major frontier providers if you have an API key (and hence are paying directly for them).

I've picked Ollama and qwen2.5-coder as my model.  So, how did it fare?  Well, let's split this into the experience and the model accuracy.  I started by wondering if it could do something as simple as write unit tests for a function.  As soon as I entered the prompt, I heard the fans on my desktop shift into high gear - it was working hard!  My GPU was pegged.  I waited.

And waited.

And waited.

Seven minutes later, my simple prompt produced an answer.  It had worked, but something that produced an almost immediate response took an eternity when running locally.  What's worse, the results left a lot to be desired. 

I didn't have enough horsepower in my desktop to run the latest qwen3-coder models, and stable-coder (another model that was suggested as being acceptable in this scenario) was not any better than the qwen2.5-coder.  I had asked the model to create unit tests for a single test file that I added as context.  It produced code that didn't work and added in extra tests for methods in other files that I had not asked for.

My learning from this - you either need a beefy box that costs several thousand dollars (due to the GPU requirements) so that you can run the best models available or you are stuck using remote models.

As to the rest of the experience - it was exactly the same as GitHub Copilot with a remote model.  You have the three modes (ask, edit, and agent).  You can configure and use tools when the model you are using supports tool use.  You can add context and use `copilot-instructions.md` just like the subscription version.

## Cline

Next, I installed [Cline](https://cline.bot).  If you ever tried Linux after being on a Windows PC, you were likely overwhelmed by the sheer number of "knobs" you can turn to configure it.  I had the same shock when I tried Cline.  It exposes more knobs than any of the other extensions I tried.  It also provides more feedback than any of the other extensions I tried.  This was properly OSS land.

It also failed more often than any of the other extensions I tried.

My first step was to just try it out.  I signed up for a Cline account.  They give you $0.50 worth of credits to play with (enough for 3-4 small tasks), and then you can add more credits as you wish.  The Cline account suggests you use Claude Sonnet, and it works just like you would expect - very well.


I then switched over to Ollama.  Despite having good experience with the configuration in GitHub Copilot, the same setup would not work in Cline.  I tried multiple models (in case it was a model incompatibility) with no success.  The API call would just time out.

![Cline model selection](/assets/images/2025/08/2025-08-03-cline-models.png)

Cline did have the largest model provider selection of all the extensions I tried.  In addition to model routers at Cline, you can also try LiteLLM (for a local flavor), OpenRouter (for a paid option), and VS Code itself (to use Cline on top of the Copilot models).  There is a plethora of options you can use directly from OpenAI, Google, AWS, Azure, Mistral, and others.  You will find YOUR model in this set.

As you can see above, you also get to set the Model Context Window and the Request Timeout.  Qwen 2.5 supports a 256K window and LLama 3.1 supports a 128K window - you can take advantage of those.

Let's switch over to the experience though.  Like all the other extensions, you can just chat.  However, Cline also has a "Plan and Act" mode.  This is most similar to the Kiro spec-driven design model (although Kiro is much more polished even in preview).  It did a reasonable job of turning my project description into an actionable plan and then executing each task. I have no doubt that if I were using Cline accounts and the Claude Sonnet 4 model, I'd be able to complete the project.  I was after a "free" option though, and the models I had access to were not up to the task.  The other good thing I found in the UI was the "MCP marketplace".  Admittedly, this is heavily skewed towards AWS usage.  I did appreciate the one-click install of the [GitHub MCP Server](https://github.com/github/github-mcp-server), for example.  I think I spent an hour just scrolling through the list figuring out if any of the other MCP servers were useful to me.

The other place Cline falls down is a big one.  It's just too complex for the average person.  You can adjust absolutely everything.  However, understanding why you would want to use a bigger context window or a specific model is not something the average developer even wants to know unless they are trying to become an expert on AI topics.  The cognitive load is high without an obvious benefit.

## Other tools

I'm going to lump a lot of tools here, since I tried a bunch when researching this article.  Here is a partial list:

* CodeGPT
* Ollama Chat
* Ollama Dev Companion
* Kilo Code AI Agent
* Zencoder

They all had "something" a little extra for themselves.  CodeGPT was a paid alternative to GitHub Copilot, but it wasn't really that much different.  Ollama Chat was a nice way to interact with the model outside of an IDE, but it didn't seem to offer anything beyond what you could already do in GitHub Copilot, or in the terminal for that matter.  Kilo Code AI Agent is a combination of Cline with Roo - it was an easier combined set up but didn't really offer anything not already within Cline, and I didn't see the benefit of combining them.  It did give you $20 in tokens though.  Zencoder was another sign up for a paid service.  It provided access to Claude Sonnet.  It's little extra was indexing your code base and providing specific agents (e.g. for Q&A, Coding, Test generation, etc), which is just another layer of complexity. I tried them all, but none of them offered anything that I couldn't do elsewhere.

Another problem - really in the usability area, and one I've gotten used to when using AI assitance.  I expect the chat interface to be in the secondary chat area - to the right of my editor.  I'm almost programmed to look for it there.  Instead, all of these tools put the chat in the primary editor space.  This means you can see the chat OR the tests OR the file browser.  It was surprising to me how much of a difference this makes the usability.  I wanted to see the tests and type something in chat while I'm looking at the test output, or I wanted to drag and drop the files onto the context area.

## Wrap-up

So, would I use any of these tools?

Nope.

For the amount of coding I do, I'm going to pay for one of the subscription services.  I'll get a better model, faster responses, and less frustration.

Using local models may be fine for simple tasks, but it's not worth the delays and inaccuracies you need to deal with.  The cost of the GPU you need will fund several years of subscription costs, and the best models are all subscription based, not local models.

Using OSS or non-core extensions all come with their own issues and annoyances.  For me, the benefits are not worth the frustration.

This got me thinking on what an ideal "AI assitant" should be:

* Be everywhere I am - not in a new application.  (Winner: GitHub Copilot)
* Live in the secondary chat area to the right of the editor. (Winners: Copilot, Kiro, Cursor)
* Transparently use the best model unless I want to specify it (Winner: Cursor)
* Minimize the prompt engineering I need to do, allowing me to focus on the job (Winner: Kiro)
* Allow the use of local models, OpenRouter, and direct API connections (Winner: Copilot)
* Provide recommendations for MCP servers (No winner) from a marketplace (Winner: Cline)
* It should allow me to auto-approve while working (Winner: Kiro)
* It should have a separate mode for pull request-like reviews - aka "review my changes" (No winner)
* It should have automatic memory, so it learns my style and requirements as we work together (Winner: Cursor)

As you can see, there is no "best candidate" that does everything - it's sort of a bit of all of them.  Who knows, the next hot new editor may just provide the AI assistant I need.  Do you have something you really want the next hot new editor to help you with?  Let me know in the comments.

In the next (and final) article, I'm going to give my thoughts on using the models available through the subscription services. I'll also give you the model I most often use (and why).  It's not the latest Claude model.


