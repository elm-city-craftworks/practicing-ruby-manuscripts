Over the last several years, Ruby programmers have gained a reputation of being
*test obsessed* -- a designation that many of our community members consider to
be a badge of honor. While I share their enthusiasm to some extent, I can't help but notice
how dangerous it is to treat any single methodology as if it were a panacea.

Our unchecked passion about [test-driven
development](http://en.wikipedia.org/wiki/Test-driven_development) (TDD) has paved the way for deeply
dogmatic thinking to become our cultural norm. As a result, many vocal members
of our community have oversold the benefits of test-driven development 
while downplaying or outright ignoring some of its costs. While I don't doubt
the good intentions of those who have advocated TDD in this
way, I feel strongly that this tendency to play fast and loose with very complex
ideas ends up generating more heat than light.

To truly evaluate the impact that TDD can have on our work, we need to go 
beyond the anecdotes of our community leaders and seek answers to 
two important questions:

> 1) What evidence-based arguments are there for using TDD? 

> 2) How can we evaluate the costs and benefits of TDD in our own work?

In this article, I will address both of these questions and share with you my
plans to investigate the true costs and benefits of TDD in a more rigorous and
introspective way than I have done in the past. My hope is that by considering a
broad spectrum of concerns with a fair amount of precision, I will be able to
share relevant experiences that may help you challenge and test your own 
assumptions about test-driven development.

### What evidence-based arguments are there for using TDD? 

Before publishing this article, I conducted a survey that collected thoughts
from Practicing Ruby readers about the costs and benefits of test-driven
development that they have personally experienced. Over 50 individuals responded, and
as you might expect there was a good deal of diversity in replies. However, the
following common assumptions about TDD stood out:

```diff
+ Increased confidence in developers working on test-driven codebases
+ Increased protection from defects, especially regressions
+ Better code quality (in particular, less coupling and higher cohesion)
+ Tests as a replacement/supplement to other forms of documentation
+ Improved maintainability and changeability of codebases
+ Ability to refactor without fear of breaking things
+ Ability of tests to act as a "living specification" of expected behavior
+ Earlier detection of misunderstandings/ambiguities in requirements
+ Smaller production codebases with more simple designs
+ Easier detection of flaws in the interactions between objects
+ Reduced need for manual testing
+ Faster feedback loop for discovering whether an implementation is correct
- Slower per-feature development work because tests take a lot of time to write
- Steep learning curve due to so many different testing tools / methodologies
- Increased cost of test maintenance as projects get larger
- Some time wasted on fixing "brittle tests"
- Effectiveness is highly dependent on experience/discipline of dev team
- Difficulty figuring out where to get started on new projects
- Reduced ability to quickly produce quick and dirty prototypes
- Difficulty in evaluating how much time TDD costs vs. how much it saves
- Reduced productivity due to slow test runs
- High setup costs
```

Before conducting this survey, I compiled [my own list of
assumptions](https://gist.github.com/2277788) about test-driven 
development, and I was initially relieved to see that there was a high degree of
overlap between my intuition and the experiences that Practicing Ruby 
readers had reported on. However, my hopes of finding some solid ground to stand
on were shattered when I realized that virtually all of these claims did not have
any conclusive empirical evidence to support them.

Searching the web for answers, I stumbled across a great [three-part
article](http://scrumology.com/the-benefits-of-tdd-are-neither-clear-nor-are-they-immediately-apparent/)
 called "The benefits of TDD are neither clear nor are they immediately
apparent", which presents a fairly convincing argument that we don't know as
much about the effect of TDD on our craft as we think we do. The whole article is
worth reading, but this paragraph in [part
3](http://scrumology.com/the-benefits-of-tdd-why-tdd-part-3/) really grabbed my
attention:

> Eighteen months ago, I would have said that TDD was a slam dunk. Now that I’ve taken the time to look at the papers more closely … and actually read more than just the introduction and conclusion … I would say that the only honest conclusion is that TDD results in more tests and by implication, fewer defects. Any other conclusions such as better design, better APIs, simpler design, lower complexity, increased productivity, more maintainable code etc., are simply not supported.

Throughout the article, the author emphasizes that he believes in the value of
TDD and seems to think that the inconsistency of rigor and quality in the
studies at least partially explain why their results do not mirror the 
expectations of practitioners. He even offers some standards for what he 
believes would make for more reliable studies on TDD:

> My off-the-top-of-my-head list of criteria for such a study, includes (a) a multi year study with a minimum of 3 years consecutive years (b) a study of several teams (c) team sizes must be 7 (+/-2) team members and have (d) at least 4 full time developers. Finally, (e) it needs to be a study of a product in production, as opposed to a study based on student exercises. Given such as study it would be difficult to argue their conclusions, whatever they be.

His points (c) and (d) about team size seem subject to debate, but it is fair
to say that studies should at least consider many different team sizes as
opposed to focusing on individual developers exclusively. All other points he
makes seem essential to ensuring that results remain tied to reality, but he
goes on to conclude that his requirements are so complicated and costly to 
implement that it could explain why all existing studies fall short of this gold
standard.

Intrigued by this article, I went on to look into whether there were other, more
authoritative sources of information about the overall findings of research on
test-driven development. As luck would have it, the O'Reilly book on
evidence-based software engineering ([Making
Software](http://www.amazon.com/Making-Software-Really-Works-Believe/dp/0596808321)) had a chapter on this
topic called "How effective is test-driven development?" which followed a
similar story arc.

In this chapter, five researchers present the result of their systematic review of 
quantitative studies on test driven development. After analyzing what published 
literature says about internal quality, external quality, productivity, 
and correctness testing, the researchers found some evidence that both 
correctness  testing and external quality are improved through TDD. However, 
upon limiting the scope to well-defined studies only, the positive effect 
on external quality disappears, and even the effect on correctness 
testing weakens significantly. In other words, their conclusion matched the
conclusions of the previously mentioned article: <u>*there is simply not a whole lot of
science supporting our feverish advocacy of TDD and its benefits.*</u>

While the lack of rigorous and conclusive evidence is disconcerting, it is not 
necessarily a sign that our perception of the costs and benefits of 
TDD is invalid. Instead, we should treat these findings as an invitation to
slow down and look at our own decision making process in a more careful and
introspective way. 

### How can we evaluate the costs and benefits of TDD in our own work?

Because there are very few evidence-supported generalizations that can be made
about test-driven development, we each have the responsibility to discover for
ourselves what effects the red-green-refactor cycle truly has on our work. But
based on my personal experience, many of us have a long way to go before we can
even begin to answer this question.

In the process of preparing this article, I ended up identifying three
guidelines that I feel are essential for any sort of introspective evaluation. I
have listed them below, along with some brief notes on how I have failed
miserably at meeting these preconditions when it comes to analyzing TDD.

---

**1) We must be aware of our assumptions, and be willing to test them.**

_How I failed to do this:_ As someone who learned TDD primarily because other smart people
told me it was the right way to do things, my personal opinions about testing
were developed reactively rather than proactively. As a result, I have ignored certain 
observations and amplified others to fit a particular mental model that is
mostly informed by gut reactions rather than reasoned choices.

**2) We must be aware of our limitations and try to overcome them.**

_How I failed to do this:_ My mixed feelings towards TDD are in part due to my
own lack of effort to fully understand the methodology. 
While I may have done enough formal practice to have some basic intuitive sense of what
the red-green-refactor cycle is like, I have never been able to sustain 
a pure TDD workflow over the entire lifetime of any reasonably complex
project that I have worked on. As a result, it is likely that I have been 
blaming testing tools and methodologies for my some of my own deficiencies.

**3) We must be continuously mindful of context and avoid over-generalization.**

_How I failed to do this:_ I have always been irked by the lack of sufficient context in literature about
test-driven development, but I have found myself guilty of committing a similar
crime on numerous occasions. Even when I have tried to use specific examples to support
my arguments, I have often failed to consider that my working environment is very
different than that of most programmers. As a result, I have made more than few
sweeping generalizations which could be invalid at best and misleading at worst.

---

If I had to guess why I approached TDD in such a haphazard way despite my
tendency to treat other areas of software development with a lot more 
careful attention, I would say it was a combination of immaturity and a 
deeply overcommitted work schedule. When I first learned Ruby in 2004, I 
studied just enough about software testing and the TDD workflow to get 
by, and then after that only brushed up on my software testing skills 
when it was absolutely essential to do so. There was simply too much to learn
about and not enough time, and so I never ended up giving TDD as much attention 
as it might have deserved.

Like most things that get learned in this fashion, my knowledge of software
testing in the test-driven style is full of gaping holes and 
dark corners. Until recently this is something I have always been able to work
around, but my role as a teacher has forced me to identify this as a real weak 
spot of mine that needs to be dealt with. 

### Looking at TDD from a fresh perspective

Relearning the fundamentals of test-driven development is the only way 
I am ever going to come up with a coherent explanation for [my 
assumptions](https://gist.github.com/2277788) about the costs and benefits of this kind of workflow, 
and is also the only way that I will be able to break free from various 
misconceptions that I have been carrying around for the better part of 
a decade.

For a period of 90 days from 2012-04-10 to 2012-07-09, I plan to follow 
disciplined TDD practices as much as possible. The exact process I want 
to adopt is reflected in the handy-dandy flow chart shown below:

<div align="center">
<img
src="http://upload.wikimedia.org/wikipedia/en/9/9c/Test-driven_development.PNG"
title="Image Credit: Excirial on Wikipedia CC-SA" >
</div>

This is a workflow that I am already quite familiar with and have practiced
before, but the difference this time around is that I'm going to avoid cutting
corners. In the past, I have usually started projects by spiking a rough
prototype before settling into a TDD workflow, and that may have dampened the
effect that writing tests up front could have had on my early design process in
those projects. I have also practicing refactoring in the large rather than the
small fairly often, favoring a Red-Green-Red-Green-...-Red-Green-Refactor
pattern which almost certainly lead to more brittle tests and implementations
than I might have been able to come up with if I were more disciplined.
Throughout this three month trial period, I plan to think long and hard before
making any deviations from standard practice, and will be sure to note whenever
I do so.

The benefit of revisiting this methodology as an experienced developer is that I
have a whole lot more confidence in my ability to be diligent in my efforts. In
particular, I plan to take careful notes during each and every coding session
about my TDD struggles and triumphs, which I will associate with particular
changesets on particular projects. Before writing this article I did a test run
of how this might work out, and you can 
[check out these notes](https://gist.github.com/2286918) to get a sense of what 
I am shooting for. I think the [github compare
view](https://github.com/sandal/puzzlenode-solutions/compare/9070...3b79) will 
really come in handy for this kind of note-taking, as it will allow me to track 
my progress with a high degree of precision. 

I don't plan to simply use these notes for subjectively analyzing my own
progress, but also expect to use them as a way to seek out advice and help from
my friends who seem to have strongly integrated test-driven development into their
working practices. Having particular code samples to share along with some additional 
context will go a long way towards helping me ask the right kinds of 
questions that will move me forward. Each time I reach a stumbling point or
discover a pattern that is influencing my work (for better or for worse), I will
request some feedback from someone who might be able to help. When I was
learning TDD the first time around I might have avoided asking "stupid
questions" as a way to hide my ignorance, but this time I am intentionally
trying to expose my weaknesses so that they can be dealt with.

After this 90 day period of disciplined study and practice of test-driven
development, I will collect my notes and attempt to summarize my findings.
If I have enough interesting results to share, I will publish them in Issue 4.12
of Practicing Ruby towards the end of July 2012. At that time, I will also
attempt to take a slightly more informed guess at the "cost and benefits"
question that lead me to write this article in the first place, and will comment
on how this disciplined period of practice has influenced my assumptions about
TDD.

### Predictions about what will be discovered

While certain things are best left to be a mystery, there are a few predictions
can make about the outcomes of this project. These are mostly "just
for fun", but also may help reveal some of my biases and expectations:

* I expect that I will reverse my position on several criticisms of test-driven
  development as I learn more about practicing it properly.

* I expect that I will understand more of the claims that I feel are either
  overstated or lacking in context, and will either be able a more balanced
  view of them or meaningfully express my reservations about them.

* I expect that I will stop exclusively doing pure test-driven development as
  soon as this trial period is over, but think it is very likely that I will
  use TDD more often and more skillfully in the future.

* I expect to be just as frustrated about the extra work involved in TDD
  by the end of this study as I am now.

* I expect that simply by measuring my progress and reflecting on it, that I
  will learn a lot of interesting things that aren't related to TDD at all,
  and that will help me write better Practicing Ruby articles!

I will do my best not to allow these predictions to become self-fulfilling
prophecies and just go with the flow, but I feel it is important to expose 
the lens that I will be viewing my experiences through.

### Limitations of this method of study

The method I am using to reflect on my studies is to some extent a legitimate
form of qualitative research that may be useful for more than just improving
my own skillset. I am essentially conducting a diary study, which is 
the [same technique that Donald Knuth used](http://books.google.com/books?id=DxuGi5h2-HEC&lpg=PA58&dq=Reading%20Qualitative%20Research%20Knuth&pg=PA58#v=onepage&q=Reading%20Qualitative%20Research%20Knuth&f=false)
in an attempt to categorize the different kinds of errors found in TeX. This 
technique is also used in marketing and usability
research, and can provide interesting insights into the experiences of
individuals with sufficient context to be analyzed in a fairly rigorous way.
However, I am not a scientist and this is not a scientific study, and so there
are a ton of limitations can threaten the validity of any claims made about 
the results of this project.

The first and most obvious limitation is that this is a self-study, and that I
am already chock full of my own assumptions and biases. My main goal is to learn
more about TDD and come up with better reasons for the decisions I make about
how I practice software testing, but it is impossible for me to wipe the slate
completely clean and serve as an objective source of information on this topic.

On top of this, I will be discussing things entirely in terms of my experiences
and won't have many objective measures to work with. My hope is that tagging
my notes with links back to particular changesets will make it possibly to apply 
some quantitative measures after this study is completed, but it is hard to say
whether that will be feasible or whether it would even mean anything if I
attempted to do that. Without hard numbers, my results will not be
directly comparable to anyone else's nor can it say anything about the average
developer's experience.

Lastly, when I look back on my notes from the 90 day period, it may be hard for
me to reestablish the context of the early days of the study. This means that my
final report may be strongly biased by whatever ends up happening towards the 
end of the trial period. While I expect that I will be able to make some high-level 
comparisons across the whole time period, I will not be able to precisely 
compare my experiences on day 5 with my experiences on day 85 even if I take
very detailed notes. This may cause some important insights to get lost in the
shuffle.

My hope is that by staying communicative during this study and by sharing most
or all of my raw data (code, notes, etc.), the effects of these limitations will
be reduced so that others can still gain something useful from my
efforts. At the very least, this transparency will allow individuals 
to decide for themselves to what extent my conclusions match up with my
evidence, and whether my results are relevant to other contexts.

### Some things you can do to help me

One thing I know about Practicing Ruby readers is that you folks really enjoy 
improving the craft of software development. That is the reason why I decided 
to announce my plans for this study via an article here rather than 
somewhere else. If you would like to support this project,
there are a few ways you can contribute.

**If you have a few seconds to spare:** You can spread the word about this
project by sharing this article with your friends and colleagues. This will help
me make sure to get adequate critical review from the community, which is a key
part of the improvement process. To create a share link, just click the handy
dandy robot down in the bottom right corner of the screen.

**If you have a few minutes to spare:** You can leave a comment sharing your
thoughts on this article as well as any questions or suggestions you might have
for me. I take all reader feedback to heart, and comments are one of the best
ways that you can support my work on these articles. 

**If you have a few minutes to spare each week:** You can subscribe to the
[mendicant-research](http://lists.rubymendicant.org/listinfo.cgi/mendicant-research-rubymendicant.org)
mailing list, where I plan to post my questions about TDD as
I study, as well as any interesting problems I run into or helpful learning
resources I end up using. I am also going to invite a few folks from the Ruby
community that I think have specific skills that will help me with this study,
but I feel that every practicing Rubyist could be a meaningful contributor to
these discussions.

**If you have a large amount of free time:** You can try to do this study along with me.
I can't promise that I'll have time during the 90 day period to regularly review
your progress, but I can definitely help you get set up and also would love to
compare notes at the end of the trial period. If this is something that
interests you, please post to the
[mendicant-research](http://lists.rubymendicant.org/listinfo.cgi/mendicant-research-rubymendicant.org) mailing list and I'll provide additional details.

Any little bit of effort you spend on helping me make this project better will
absolutely be appreciated! Our readers are what make this journal what it is, I
just work here. :wink:
