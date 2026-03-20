---
layout: post
title:  "Soul Searching in Software Land  "
date:   2026-03-10 20:00:20 +0000
image:  '/images/soul-searching-software-land.png'
description: "Soul Searching in Software Land"
tags: [Software Engineering]
---

There is a lot of soul-searching going on in Software land.

We've gone from a repudiation of the merits of AI ("Claude will never write better code than _me_") to a frantic justification of our existence ("OK, it's better, but my hard-won experience is still needed").

I agree. The justification of the need for strong software engineering skills and practices is not...unjustified. But then I would, I'm a software engineer. Articles such as this, making the case for the continuation of our careers, are popping up all over the web.

There is another school of thought: that before long LLMs will be able to develop a taste for good engineering practices themselves, and churn out faultless codebases with no supervision, whilst erstwhile software developers kick up dust in the queue for the Job Centre.

I wanted to take an unbiased look at both opinions. But then I found I couldn't do that, so here is a biased one instead.

### The Case for Software Engineering

A brittle codebase is a brittle codebase. Its brittleness will cause problems, regardless of who is working on it.

Defining brittleness is challenging. Some hallmarks are high coupling (strong inter-dependencies make it difficult to change one part without changing another); low cohesion (modularised components make little logical sense); and poorly designed interfaces (such that invoking parts of the code becomes complicated). The worst types of brittleness are _unknown unknowns_ - traps that cannot be inferred through reading the code, and are usually bound up in ‘institutional knowledge’.

But how does a codebase become brittle?

For this I’m going to quote one of my favourite books on software: A Philosophy of Software Design by John Ousterhout.

The whole book is framed as a battle against _complexity_, which can be thought of as a leading cause of brittleness (not all complexity is bad, but the type detailed in the book _is_). The central theme is that complexity builds up over time, unintentionally, until such a point where the project becomes so complex that making changes is an ordeal (i.e. it is brittle, easy to break).

In the book, John Ousterhout identifies 2 types of programmer: the Tactical Programmer and the Strategic Programmer. 

The purview of the Tactical Programmer is to make shit work, at all costs; they churn out features with little regard to the ongoing maintenance of the project. 

The Strategic Programmer on the other hand, like the proverbial tortoise, makes slower progress but deeply considers the structure and maintainability of the codebase.

This graph is reproduced from the book:

![Tactical vs Strategic Programmer](/images/Tactical-vs-Strategic.png){:width="80%" }

The Tactical Programmer makes rapid progress initially, but soon gets mired down in complexity. The Strategic Programmer is slower to begin with, but over time is able to add features more quickly. Mr Ousterhout estimates the crossover point to be about 18 months.

Coding agents, then, are the ultimate Tactical Programmer. How can they be otherwise? They have no idea if they are working on a long-term project or a throwaway vibe-coding effort. It could almost be thought of as their strength - "make this thing work, don't bother me until it does."

A case in point: as I'm typing this I'm having Claude generate a bash script (a very impressive bash script, I must admit). But when I check the script, I find it riddled with Python shell-outs. It works wonderfully, of course. But now I have a _dependency_ on Python. Claude obligingly removed all Python calls when prompted, proving that the dependency is superfluous.

Ah, you might say, see, you just need to explicitly tell Claude _not_ to do that. Yes! But that is my point: I only recognise this as an anti-pattern because of the *taste* that has been developed over a whole career. If we were to erase software engineers from the gene pool, who would be left to define these behaviours away?

### The Case Against

Is it so far-fetched to believe that LLMs will not simply imbibe the collective knowledge of the software industry, and be able to apply the same judgements as experienced software engineers?

It's not. In fact you'd be foolish to bet against it. Imbibing knowledge is what they do best. 

LLMs do what humans tell them. Humans with no engineering _taste_ will inevitably build complex and brittle solutions. Will LLMs eventually learn to push back, and suggest alternatives? Yes, probably. Maybe they already do, with the correct setup.

So where does that leave us?

Well, here's the quiet part said out loud: despite all our fulminations about good engineering practices, the truth is, many engineers don't follow them anyway! There are died-in-the-wool Tactical Programmers out there causing mayhem but, at times, we've all been Tactical Programmers (when circumstances demand). The result is millions of existing codebases which are already a complete mess; no tests or documentation, riddled with unknown unknowns and completely in hock to institutional knowledge.

It is fanciful to believe that unsupervised LLMs will magically tame these codebases and be able to successfully improve or add to them. It does however seem eminently possible that _with_ human supervision, technical debt can be rapidly paid off: the cost of servicing the debt has just dropped dramatically.

It could be argued that the very evolution that LLMs will drive is that more fleet-footed competitors will arise, and those products mired in existing _bad_ code will be driven out of the market. It is certainly true that businesses starting now have a massive advantage over incumbents. But businesses are far more than their codebase, there are many many other reasons why incumbents will remain incumbent.

And what about those new businesses: can you really imagine a software business without software engineers? No, but it is increasingly easy to imagine one with just a handful. The power conferred by LLMs on each engineer is a massive force multiplier, and will have a big effect on organisational structure.

So where does this leave software engineers? Not obsolete, but transformed. The value we bring is shifting; from the ability to _produce_ code to the ability to _recognise_ good code. To smell complexity before it metastasises.

LLMs are the ultimate Tactical Programmers. They'll make shit work, beautifully, every time. But they don't know if they're building a cathedral or a sandcastle. That judgement — that _taste_ — is what decades of experience actually buys you. The question isn't whether we'll still need software engineers. It's whether we'll use these tools to finally become the Strategic Programmers we always claimed to be.