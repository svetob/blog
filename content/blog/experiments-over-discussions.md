---
title: "Experiments Over Discussions"
date: 2018-07-06T15:51:10+02:00
draft: true
---

Remember the last time you had a long, heated discussion and found a perfect solution? Me neither.

Wouldn't it be amazing to have a better way to resolve arguments?

I recall a particularly exhausting discussion that went on for over a month. A few years ago, our team was working on a large scale storage backend. At some point we found a potential issue that could lead to lost updates.

We had two solution ideas. One simple bare-minimum fix, the other a major coordination overhaul. In many ways they were direct opposites.

"We've got to find the best solution", said someone. And just like that, all possible middle ground disintegrated into dust.

We wrote design documents. We held meetings. We discussed and argued. But we couldn't find agreement. As weeks went on, frustration and resentment grew.

What came of it? Nothing. After a month with no agreement, we jumped onto a new project, leaving the issue to gather mold in some corner. Our attempt to find the best solution through discussion had failed.

What could we have done differently?

![Photo by John Matychuk on Unsplash](/img/john-matychuk-479013-unsplash.jpg)

# Searching for an Experiment

Engineering is about risks and unknowns, learning through trial and error when applying ideas to the real world. After testing and gaining information, you can rethink your approach and make it better. Why miss out on this process by defining everything in advance? Why handicap yourself?

Instead of looking for the perfect solution, search for the most valuable _experiment_. What can you try quickly that will answer your biggest questions? How can you easily find useful information?

In our discussion, we had many unknowns. Would the issue be common or rare? How much money would it cost us? If we had said “Let’s find out how common this issue is” or “Let’s find out how if customers would notice the issue” we could have taken action.

We would learn much later that the issue was very rare and had no proven customer impact. In short, it was not worth spending time on. If we had begun by gathering these facts, we could have saved ourselves a month of headache.


# Consent over Consensus

There was a fundamental issue at play in our months-long debate. For one solution to be “the best”, the other solution had to be less good. One side would have to lose face. There was no motivation for either side to yield any ground.

Not only was the pursuit of agreement meaningless in finding the best solution. It was causing resentment, as team members risked losing face. It was wasting time, preventing us from learning.

When looking for an experiment instead of a solution, you don’t actually need consensus. You just need _consent_. It’s okay not to agree, as long as a you find a way forward.

If you’re having problems agreeing on an experiment, here are some suggestions for moving forward:

- __Which experiment will take least time?__ Small, effort-less experiments mean little risk. If a test will take less than an hour, just sit down and do it.

- __Who is willing to actually do the work?__ Trust your team mates to make the right call. If Matt suggest a great experiment but only Lisa can actually spend time experimenting, let her choose what to spend time on.

- __Try both!__ If all else fails and you still can’t agree, just try all your ideas! Remember that the point is to learn and gather facts.


# Putting it all together

Fast forward to a few months later. We were facing incidents and performance issues with an unstable key-value store. Maintenance costs were through the roof and attempts to scale it up had failed. Again, there were many paths forward. As discussion arose, I was eager to go anywhere but back to the same state we had been a few months ago.

That afternoon, I suggested we try out the approach that was easiest to test. We hacked together a local test, replacing the database with an alternative one. It outperformed our production cluster on my laptop. There and then, discussion died. It didn't matter if this was the best way forward or not. What mattered was we had proof this would solve our problems.

We were still unsure how it would scale. So, we began with what was most likely to fail - large-scale performance testing. After that, we would test fault tolerance, then maintenance tasks like adding nodes, and so on. Our big project became a series of small experiments. A third of the way through the project, we _knew for a fact_ it would work at scale, handling failures gracefully.

By testing in practice, assumptions became facts. As "I think" became "I know", risk vanished and discussion with it.

The next time you find yourselves at odds, aim for experiments over discussions. Look for consent over consensus. Spend your time learning instead of worrying.
