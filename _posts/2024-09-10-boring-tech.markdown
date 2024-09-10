---
layout: post
title:  "The Return of Boring Tech"
date:   2024-03-05 10:00:20 +0000
image:  '/images/zirp-boring.webp'
description: "In times of economic woe, an unlikely champion lumbers into view - Boring Tech"
tags: [Software Engineering]
---

Are we seeing a return to boring architecture?

Recently I watched a [talk by Gergely Orosz](https://youtu.be/VpPPHDxR9aM?si=okquzuxr0E_Jr9BS) of 
[Pragmatic Engineer](https://www.pragmaticengineer.com/) fame (great talk, you should watch it). In it he bemoans the 
end of the Zero Interest Rate Period (ZIRP), and the glut of private equity investment that came about because of it. 
The thinking goes like this: during a period of zero-percent interest rates, people in charge of _lots_ of money (think 
fund managers or eccentric billionaires) will not see much of a return by consigning it to a bank account.

Now, if there's one thing people with lots of money like, its more money; so instead they find other ways to improve 
their lot. One of those ways is to bankroll tech companies: throw a few million each at a few hundred start-ups and, 
like baby turtles ambling their way to the sea, hope that at least one of them survives.

BUT, when interest rates go up, this model makes less sense. Now, investors can achieve returns of ~5% simply by 
_leaving their money in the bank_. There is no risk involved, and the investors know exactly what they'll get back. As 
there is now less money sloshing around, there is less earmarked for funding risky tech ventures. Established tech 
companies are not immune either: with their shareholders eyeing up cosy bank accounts, they too are forced to maximise 
profit, and return.

So what has all this to do with architecture?

In the talk, Gergely posits that businesses of all shapes and sizes will have to learn to do more with less. Less money 
means less hiring means fewer engineers. With fewer engineers, the smart move is to narrow the scope of the technologies 
they need to work with. Or, rather, the number of technologies they need to know. Not only does this reduce the 
cognitive load on the engineers, it also means that each individual is able to work on multiple areas of the stack.

An example cited in the talk is Typescript. Whilst Typescript itself is not boring, perhaps using *solely* Typescript 
could be considered so (if set against the plethora of other 'cool' technologies). Typescript is an inspired choice: we 
pretty much _have_ to use it for the web, so why not then for mobile, and the backend? We can even use it for 
infrastructure - AWS' Cloud Development Kit (CDK) uses Typescript natively (and compiles to Terraform so is not limited 
to just AWS).

Similar sentiments have been raised by others:

[This article](https://www.robinwieruch.de/react-full-stack-framework/) by Robin Wieruch takes the Typescript argument 
one abstraction further and proposes using React as a common framework for front- and back-end. This is a pattern 
popularised by Next.JS and is fast becoming part of the React mainstream via Server Components and Server Actions.

Also [this post](https://www.amazingcto.com/postgres-for-everything/) by Stephan Schmidt entreats us to stop using 
different technologies for different types of data storage, and simply use Postgres for everything. This holds up to
scrutiny, particularly for startups trying to move quickly. Good developers will structure their code in such a way
that the storage layer can be switched out if/when the business grows.

Now, none of this should be particularly unwelcome, especially if it leads to a reduction in *complexity*. It's long been 
known that complexity is the silent killer, a form of organisational fat clogging up the arteries of tech teams everywhere. 
Yet trends over the last decade (the ZIRP) have tended towards more complexity, not less.

We've witnessed the rise of architectural patterns such as Microservices, event-driven Serverless and even Kubernetes 
that often do little to ameliorate the head-ache of complexity. I'm not saying that these things are _bad_ per se. If 
done right, and in the right business context, they can be genuinely transformative and add real value.

But they have also been part of the mania of the ZIRP. Companies sprang up to productize these concepts, eco-systems 
were birthed, conferences materialised. There was gold in them hills and in order to divine that gold, a particular 
group of people needed convincing: software engineers.

It's unarguably true that a corollary of the red-hot tech sector has been a red-hot jobs market. Talented 
engineers have been able to hop between jobs every couple of years.

All of this creates a push and a pull incentive to use less boring tech. In a seller's market, businesses may opt to 
choose 'cool tech' in order to attract the best engineers from a dwindling pool; this is the pull incentive. The push 
incentive comes via those same engineers _seeing_ cool tech all around them (including on those very job specs) and 
believing they need to be using it in order to further their careers.

All the while the handle of complexity is silently being cranked.

Whilst the pitfalls of complexity have long [been known](https://louismccormack.com/accidental-vs-essential-complexity), 
the tide of public opinion is steadily resurfacing arguments against it. A book released by John Ousterhout, [A 
Philosophy of Software Design](https://a.co/d/ecEG4y0), has gained a lot of traction. The book is framed as a battle 
against complexity, where vigilance must be continually maintained. Also, we have seen robust discussion around the 
return of monoliths, a direct riposte against the [sprawl of microservices](https://world.hey.com/dhh/how-to-recover-from-microservices-ce3803cc). 
We have concepts such as [Radical Simplicity](https://www.radicalsimpli.city/) and [The Boring Technology Club](https://boringtechnology.club/). 
The world turns, and what's old is new again.

Taking all of the above into account, there is every chance that we will start to see a tendency towards boring 
technology. However, that should be seen as no bad thing. Swap out the word 'boring' for 'less complex'. There is an 
elegance in designing software architectures with as few moving parts as possible, that can be readily understood and 
supported for years to come.
