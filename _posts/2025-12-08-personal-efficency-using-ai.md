---
layout: post
title: >
  Personal Efficiency gains using AI (Dec'25)
feature_image: "/images/israel-palacio-ImcUkZ72oUs-unsplash.jpg"
---

In recent times, it has become common for people to post about how they're becoming more efficient with the use of AI.
I want to follow that trend and also provide a quick overview of where I've been using AI in my workflow over the last months to ensure I'm as efficient as possible.
Some of the gains have been in place for months; some of the things I'm using are actually ones I discovered in the last week, and make me a lot more productive.
Sadly, there are still many places where I wish I could utilise AI to be a bit more productive, but we're making progress.

## Writing Text
Writing text, especially in English, is one of the most common activities I engage in during my daily work.
For that, I have been using Grammarly for a long time to correct many of the grammatical mistakes I make as a non-native English speaker.
This has helped me improve significantly, both in the end result and in the text I write before it gets corrected.

Additionally, I've switched to using WisprFlow to dictate some of my text.
This is particularly useful if you need to write large chunks of text without typing them, or if you're on the go and only have a mobile phone.
Then typing the amount of text will be really a hassle.

Sadly, what Wispr is going to add as text is not perfect, so especially for blog posts and longer content, you need to review it once or twice.
Besides some typos or misspelt words, especially things like programming-specific abbreviations, are not well matched and often can't come up in a quite weird way.
Still, some of the typical ones, like MCP, are perfectly recognised by Wispr.
There is always a pass with Grammarly required afterwards.
Luckily, LLMs are not very picky about the correctness of your text, and thus, this works perfectly for prompting them.

## Research
My web search has shifted mostly to using Perplexity if I need a piece of information.
I can type in a rough search statement and don't need to worry about using the correct terms to get a match.
Instead, Perplexity is quite good at finding search terms that match my question very well.
In comparison to other LLM-assisted web searches, Perplexity is much more snappy, and I also perceive their citations as a lot better.
For things where I know exactly what the terms are, I'm still sticking to Google, as a simple web search is much faster.
This is particularly true for shopping; here, I often already know exactly what I want and thus don't really need or want AI assistance.

If I want to conduct in-depth research on a topic, I often start by searching using both the standard and deep research features simultaneously.
Often, Deep Research leads to a much better understanding of a topic than a quick web search;
however, there have been several instances where Deep Research took a peculiar turn and presented me with a lot of information that was tangential to my question but not at its core.
In these cases, the normal search feature of Perplexity actually led to a much better answer.

My most common usage for Perplexity is actually looking up solutions for error messages.
Due to my work on [conda-forge](https://conda-forge.org/), I frequently encounter errors in third-party software.
This also happens in using this software during development.
While a standard LLM can often reason quite well about software errors, it is limited by its training dataset.
My errors are in over 80% of all cases connected to software releases made in the last two months. Thus, if you do a web search and even include the version changes, the explanations for possible fixes improve drastically.

## LLMs in software development
The most common area where LLMs have brought improvements, as a software engineer, is in coding.
Due to my role as a manager, I no longer make significant contributions to larger codebases.
With that in mind, someone who does such work might use a completely different stack.

While still a significant chunk of my work day is spent on github.com, I rarely use GitHub Copilot.
In most cases, there is a tool that does the job better for me.
Better in this case could mean faster or with a better UX.
I have not yet compared the quality of each of these tools, as in my cases, there have not been significant differences on this side.

Producing new code is a small part of my workday.
With LLMs, the share of time I spent on this didn't change; simply, the amount of produced code increased.
This is mainly due to me now letting Claude Code write a lot of small automation scripts that ease tasks that I never considered automating before.
These tasks don't take that long, and without the LLM-assisted coding tools, I would have probably taken up an hour or more to write the automation.
With prompting, reviewing, and testing these automations, the time required is now typically within a maximum of five minutes per task.
This makes many more tasks in my daily life worthwhile to automate.

If I do make a more significant change in a codebase, I tend to use Cursor.
The UI allows me to review the proposed changes more effectively than in a TUI like Claude Code.
It also has the advantage of being significantly faster than GitHub Copilot.
Its speed is one of the surprising aspects/benefits of it.
Many of my colleagues initially seem reluctant to try out Cursor, as it looks similar to Copilot, but they are then positively surprised by its speed.

If I want to understand large code bases and use the model for querying them, I switch back to Claude Code.
This is mostly due to my love for the terminal (I'm a NeoVim power user), but also because of the distraction-free presentation of everything in Claude Code.
Querying a codebase in my workday can involve either finding a specific functionality within a larger codebase or initiating a larger review of the codebase.
For the latter, you need to structure the prompt according to the type of review you want to conduct.
However, it really helps to break down a project into segments, build an understanding together with the LLM, and ask more specific questions in certain parts of the code.
Here, better documentation or a good CLAUDE.md (which is essentially documentation) really helps to achieve better, more targeted results.

LLMs (not a particular one) are great at finding the actual error in a very large CI log.
While these errors generally occur at the end of a log file, they are not always at the very end, as some cleanup occurs once the actual job fails.
Additionally, there are many cases where a single core error occurs, but multiple exceptions are raised on top (e.g., in the case of a failed compilation) that obscure the search for the initial failure.
Using an LLM can save you anywhere from 30 to 60 seconds, but if you do this often or in an automated fashion, the time savings also add up.
LLMs can then also help and explain the reason for the error.
In the case of some C++ errors, this also helped me to understand which line actually explained the error.
For certain failure scenarios, this is sadly non-trivial to understand.

If the error is in code I'm working on, I usually switch to local development using Claude Code and let it figure out how to fix it.
If the error occurs in a third-party tool (as is the case in my conda-forge work), I'll use, as mentioned above, LLM-assisted web search to obtain a suggestion on how to resolve it.

The combination of LLM-assisted error detection in logs and web search for finding a solution to the problem works so well that I have used it as an onboarding task for an intern.
This is a neat project that allows one to learn the frameworks for dealing with LLMs, understand how different LLMs yield different results, and identify the engineering challenges that arise in this context.
One of the findings we observed was that the mini variants of the most common LLMs identified errors in the log file; they only listed a set of errors, whereas the larger models continuously pinpointed the actual error that caused the failure and could extract that specific information.
We look forward to sharing more information on this soon.

## Summary
Overall, LLMs help me to solve a lot of small things more efficiently in several areas.
There is not one single thing that they have made a massive difference in.
It's challenging to quantify the improvement, but overall, it has a very noticeable and positive impact on my workflow.
While many things are faster now, I also get distracted by them, as I want to see whether they can solve other problems or spend time understanding how they work.

My general takeaway from all the experience I gained with using LLMs in my daily work is that you should be in a constant testing phase.
New tools or models appear on a daily basis and can help solve some challenges in a different way.
Without a testing routine, it is hard to follow, and it will also be hard to catch up.
I don't think that it pays off to wait until the "dust has settled" and until clear techniques have emerged that don't get updated so frequently.
You will need to catch up on all the development anyway, and that won't be easy if you don't do it incrementally.
At the same time, if you wait and don't constantly test new things, you will also miss out on a lot of efficiency gains that (accumulated) make an enormous impact on the daily work.
