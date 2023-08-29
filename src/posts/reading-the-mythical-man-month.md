---
layout: layouts/post.njk
title: Reading The Mythical Man-Month
metaTitle: Reading The Mythical Man-Month
metaDesc: Book review and my takeaways from the classic software management book
socialImage: https://www.oreilly.com/api/v2/epubs/0201835959/files/graphics/f0016-01.jpg
date: 2023-08-29T10:46:41.829Z
tags:
  - dev
  - books
---
![The Mythical Man-Month by Frederick P. Brooks, Jr.](https://www.oreilly.com/api/v2/epubs/0201835959/files/graphics/f0016-01.jpg "The Mythical Man-Month by Frederick P. Brooks, Jr.")

[*The Mythical Man-Month* (*MM-M*)](https://www.goodreads.com/book/show/13629.The_Mythical_Man_Month) is hailed as a classic in the software industry: a series of essays addressing software project management.

It wasn't until I started to read *[The Cathedral & The Bazaar](https://www.goodreads.com/book/show/134825.The_Cathedral_the_Bazaar)*, which referenced *MM-M* extensively in the notes and bibliography, that I decided a tangent of pre-requisite reading was called for. Since I'd had *MM-M* sitting on my shelf for a while, and it was written in 1975, it felt overdue.

I have the 20th anniversary edition which features four extra chapters, including a retrospective on what had changed in 1995 since the original 1975 edition.
Being 2023 we're now closer to a 50th anniversary of *MM-M* and nearly 30 years since the anniversary edition. Perhaps even more has changed since 1995 than between the first and second edition!

The original chapters have a good amount of relevant commentary and that shouldn't be too much of a surprise since any seasoned professional soon realises the hardest aspect of software development is communication and organisation of people. Technology may have [dramatically](https://www.goodreads.com/book/show/3532021-the-mighty-micro) changed in 50 years, but people have not.

There's also a fair amount of irrelevant content especially some of the examples regarding memory space, productivity measures and "The Surgical Team" but, a couple of chapters aside, it doesn't detract from the book's readability.

I liked that some of the chapters are backed up by evidence, although the book is largely based on Brooks's anecdotal experience building the IBM System/360 in the 1960s (so it's actually 60 year old advice!)

My biggest **takeaways** were:

1. #### Brooks's Law: "Adding manpower to a late software project makes it later"

The central point of the eponymous chapter. This certainly jives with my experience and it's good to see the idea considered in depth. There's little evidence to support this claim in the original chapter but his retrospective references some subsequent studies. They suggest the reality is more nuanced: dependent on the people who are added and when they are added. Brooks still stands by his original claim as a good rule of thumb.

2. #### Conceptual Integrity and the Architect

Over several chapters, Brooks emphasises the importance of a software system's conceptual integrity which he asserts leads to simpler software and therefore less time debugging. This seems reasonable to me and Brooks continues to advise it. 
This implies top-down design and Brooks recommends minimising the number of people involved, hence the architect role.

An interesting comparison to draw here is the concept of emergent design from the Agile school of thought.
In my experience, architect roles have been on the wane in recent years¹ and perhaps the rise of Agile teams is a relevant factor. I'm not convinced that this decline is an entirely good thing and I'm skeptical of emergent design.

The following Dave Thomas quote sums it up well: 
> Big design up front is dumb, but doing no design up front is even dumber

I'm sure most Agile experts would agree with this sentiment but I wonder if the Agile approach can too easily lead teams into a trap of eschewing upfront design entirely. This being enabled by the lack of an architect role; instead relying on engineers to voluntarily fight against the path of least resistance: churning out features in the existing architecture.

3. #### Estimating software tasks has always been challenging

Estimation has been like a [Sword of Damocles](https://en.wikipedia.org/wiki/Damocles) hanging above software engineers for as long as I've been in the industry. Considering *MM-M* proves estimation woes were pretty well known in 1975 it's disheartening this hasn't improved over the years, or at least, the issue isn't more widely recognised. Unless you're lucky enough to find yourself at an organisation that holds no weight to estimates, then maybe we are doomed to repeat this charade, with managers and programmers both complicit. Afterall, all programmers are optimists, as Brooks points out early on in the book.

Despite me feeling it in my bones, the studies Brooks cites were the first I'd actually seen. The most interesting claiming that jobs take twice as long as estimated. This largely being down to only 50% of employees' time actually getting spent on programming/debugging, the rest being spent on machine issues, short unrelated jobs, meetings, paperwork, company business, sickness, personal time, etc.

Since I don't believe we've made much progress in this field, Brooks's comment bears repeating:

> Until estimating is on a sounder basis, individual managers will need to stiffen their backbones and defend their estimates with the assurance that their poor hunches are better than wish-derived estimates

Finally, I particularly like this quote:

> The incompleteness and inconsistencies of our ideas become clear only during implementation

which both justifies why estimation is hard as well as the need to do some upfront design including writing things down and communicating them early on in the process to tease out the finer details.

S﻿ince both *The Cathedral & The Bazaar* and the newer chapters of *MM-M* sing the praises of *[Peopleware: Productive Projects and Teams](https://www.goodreads.com/book/show/67825.Peopleware)*, I am continuing my tangent in that direction.

### F﻿ootnotes

¹ I was keen to fact-check my impression about the decline of the architect role and Google does seem to agree:

* <https://trends.google.com/trends/explore?date=all&q=software%20architect,software%20developer,business%20analyst,software%20engineer,solution%20architect&hl=en-GB>
* Googling for "Are software architects declining" and "are software architects increasing" yields similar results:

  * <https://www.infoq.com/articles/care-about-architecture/>
  * <https://www.linkedin.com/pulse/diminishing-role-software-architect-raghu-kishore-vempati/>
  * <https://jero786.medium.com/what-is-a-software-architect-a9627241813a>

ChatGPT actually suggests the opposite: that the number of architect roles is increasing, but it couldn't back up its claims and I'm not sure I trust it more than Google in this instance.