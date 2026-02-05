---
layout: post
title: >
  Should I write that AI coding strategy post?
feature_image: "/images/israel-palacio-ImcUkZ72oUs-unsplash.jpg"
---

In the last two months, I wanted to get people to experiment more with agentic coding tools.
While I have seen successful uses of them, some people still weren't using them at all,
and others stuck with one tool (mostly GitHub Copilot) and assumed that all other tools were similar.
Given the changes we are currently observing in the software engineering space, the level of experimentation I was seeing at our company was lower than I would have expected.
Still, there is a large crowd trying out everything, experimenting with new techniques and constantly seeking access to new tools.
These should be the ones I want to encourage to keep doing it and help them remove any blockers along the way.

## Why should I write such a post?

Among other actions, I wanted to write a call-to-action blog post to encourage and streamline our efforts to adopt AI-based (agentic) coding tools.
The post should not be as strong as [the one from Brian Armstrong](https://fortune.com/2025/08/25/coinbase-ceo-brian-armstrong-ai-coding-assistants-mandate-tech/), but rather a soft version of [Tobi Lütke's post](https://x.com/tobi/status/1909251946235437514).
The plan is not to reshape/reorganise the company with AI, but rather to guide it toward a sane adoption of AI coding.

Before writing the post, I had to decide whether I should do this in the first place.
I wanted to include some context on the current state, discuss open questions, outline what we are going to do, and include a call to action at the end.
This will turn into a lengthy post, and, from experience, the longer a post is, the less likely people are to read it in full.
Still, this marks a significant shift in how we (will) work, and I was encouraged by several people to address this and ensure we move at the same pace as the leaders in the field.
In addition to the length, the content can simply lead to some backlash.
We are in an experimental phase and have not found the right way to do things.
This means that it is pretty easy to be criticised for pushing immature practices.
In the end, this is something I considered a necessary evil, as I want to push people to look into agentic coding that they are not doing as actively as I wish.

## How to start?

Once I had committed to writing the post, I needed to decide what it would contain.
As I wanted the whole company to be able to read it (80%+ of my colleagues write code), it needed a sufficient introduction to the topic.
To start, I used Karpathy's post from 26th Dec (you need to look it up, I don't link to that site anymore) as an introduction.
This already summarises the madness of choices/possibilities/... a programmer currently has to go through if they want to write code.
In addition, I felt the urge to clarify that producing code is only a subset of the work a software engineer does.
My gut feeling is that once you leave the group of people writing software, people assume a software engineer's sole task is coding.
The other elephant in the room I wanted to touch on in the introduction was the people calling for a "wait-and-see" approach, where we lean back and look again in several months/years' time on what people converged on.
While this would save us a lot on experimentation, it would also mean we would not be able to leverage the benefits for very long.

With the context set, I moved on to the company specifics.
This picked up conversations from Slack and also referenced Notion pages listing the tools we have available and those we are actively experimenting with.
This should directly allow people to take action and try out things that they may not have been aware of.
To emphasise that we are going through a significant change, I then outline the new patterns of software development we are seeing.
While you shouldn't "vibe code" everything from now on, it is important to understand that you can successfully use it for things that you may not have thought of before.
If you have the right guardrails and checks in place, you should be able to create settings/situations where you can run the code blindly.
Still, there is a psychological component to it.
We are used to inspecting the code to ensure it does what we intend, rather than checking it from the outside.
In some cases, we need to unlearn it, whereas in others, we should definitely review and fully own it.
This part is important for me to get right.
I don't want to argue that everything should be vibe-coded, but I do want to encourage people to try new possibilities where you wouldn't have coded something at all in the past.
In contrast, anything that could be considered production or critical code should be carefully reviewed using the four-eyes principle and owned by the person submitting it to a project.

Finally, I wanted to mention the more extreme approaches, such as fleets of agents or OpenClaw.
While I don't think that our software development process can benefit from them or we would trust such things security-wise,
it is important to study what people are building and learning in this space.
If you push stuff to the extremes, you will see what breaks, under which conditions it breaks and what actually holds.
We can then apply these learnings to our processes.

## Laying out next steps

Having a long introduction and providing context doesn't help at all if you don't take action.
Thus, I continued with a section on what actions I am taking.
As I'm the one doing a bold blog post, I should also be the one taking bold next steps.
As this is not a purely technical evolution, I have started preparing some in-person events to talk to people.
In addition, I will share more information on the topic in a virtual company-wide software engineering event.
My main goal with these events is to encourage people to experiment with these tools and discuss their learnings.
To enable experimentation, I not only need to encourage people but also understand what is holding them back and work to remove their blockers.

Before I jump to a call to action for the reader, I enumerate some open discussion points that go beyond choosing the right tool to how the workflow should look.
This can be tied to the tools people are using, but at the moment, it seems orthogonal to the specific coding agent.
While I can't list everything that needs to be discussed, I've listed the most prominent and/or obvious items.
As I'm posting this article internally, people can comment on it and maybe already start a discussion there or link to a deep dive on the topic.
In the context of our company, there is also a major topic that would immediately lead to larger discussions, so I have added a paragraph on it before finally jumping to the call to action.

## Call to Action

The final part of the blog post is then my call to action to all my colleagues.
With my main objective being to have them experiment more, I start by asking them to install the most-hyped tools.
While you do not need to start using them immediately, you should have them installed so that this bit of friction is removed before you encounter a situation where you could apply them later. 

With everything prepared, I encourage them to consider every problem they encounter in the near future and whether they can use these tools to address it.
For matching situations, I strongly suggest that they also try out vibe coding.
While it isn't a good fit for everything, there are many situations where looking at the code isn't necessary.
If you haven't done it in practice, it sounds scarier than it is (if used appropriately).

## Did I cover everything? 

Given the large and fast-moving nature of the topic, it was impossible to cover everything in this blog post.
I didn't go into detail about any specific tools or techniques.
These are ones that will change quite quickly, and some of them even changed during the writing of this blog post.
One section I initially covered was the cost.
In the end, because the section became quite long, I separated it into its own blog post and will post it in about one week.
Agentic coding is not only costly because it consumes a lot of tokens, but it also involves a lot of bureaucracy in onboarding people to the new tools and conducting an information security review.
Furthermore, there's a social and psychological cost because we see dramatic changes in the way work is happening, and you have a lot of social interactions about how this should happen, and what should be done first, and where we shall wait and see what time will tell us.
