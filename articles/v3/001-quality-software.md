I ended the second volume of Practicing Ruby by launching an exploration into
the uncomfortable question of what it means to write good code. To investigate
the topic, I began to compile a wiki full of small case studies for each of the
different properties outlined by [ISO/IEC
9126](http://en.wikipedia.org/wiki/ISO/IEC_9126) -- an international standard
for evaluating software quality. While I made good headway on this project
before taking a break for the holidays, I left some of it unfinished and
promised to kick off this new volume by presenting my completed work.

While it is possible to read through the wiki by [starting at the overview on
the homepage](https://github.com/elm-city-craftworks/code_quality/wiki) and then
clicking through it page by page, there is a tremendous amount of content there
on lots of disjoint topics. To help find your way through these materials, I've
summarized their contents below so that you know what to expect.

### Functionality concerns

While we all know that getting our software to work correctly is important, the
functional qualities of our software are often not emphasized as much as they
should be. Issues to consider in this area include:

* The [suitability](https://github.com/elm-city-craftworks/code_quality/wiki/Suitability) 
of our software for serving its intended purpose. As an example, I note the
differences between the _open-uri_ vs. _net/http_ standard libraries and suggest
that while they have some overlap in functionality, they are aimed at very
different use cases.

* The [accuracy](https://github.com/elm-city-craftworks/code_quality/wiki/Accuracy) 
of our software in meeting its requirements. As an example, I discuss how a
small bug in Gruff made the entire library unusable for running a particular
report, even though it otherwise was well suited for the problem.

* The [interoperability](https://github.com/elm-city-craftworks/code_quality/wiki/Interoperability) 
of our software and how it effects our ability to fit seamlessly into the user's
environment. As an example, I discuss at a high level the benefits of using the
Rack webserver interface as compared to writing adapters that directly connect
web frameworks with web servers.

* The [security](https://github.com/elm-city-craftworks/code_quality/wiki/Security) 
of our software and how it affects the safety of our users and the systems our
software runs on. As an example, I discuss a small twitter bot I wrote for
demonstration purposes that had a operating system command injection
vulnerability, and also show how I fixed the problem.

### Reliability concerns

Even if our software does what it is supposed to do, if it does not do so
reliably, it will not do a good job at making users happy. Issues to consider in
this area include:

* The [maturity](https://github.com/elm-city-craftworks/code_quality/wiki/Maturity) 
of our software, i.e. the gradual reduction of unexpected defects over time. As
an example, I discuss some regression tests we've written for the Practicing
Ruby web application.

* The [fault tolerance](https://github.com/elm-city-craftworks/code_quality/wiki/Fault-Tolerance) 
of our software, i.e. how easy it is for us to mitigate the impact of failures
in our code. As an example, I discuss at a high level about how ActiveRecord
implements error handling around failed validations, and show a pure Ruby
approximation for how to build something similar.

* The [recoverability](https://github.com/elm-city-craftworks/code_quality/wiki/Recoverability) 
of our software when dealing with certain kinds of failures. As an example, I
discuss various features that resque-retry provide for trying to recover from
background job failures.

### Usability concerns

Once we have code that does it's job correctly and does it well, we still need
to think about how pleasant of an experience we create for our users. Issue to
consider in this area include:

* The [understandability](https://github.com/elm-city-craftworks/code_quality/wiki/Understandability) 
of our software, in particular how well its functionality is organized and how
well documented it is. As an example, I extract some guidelines for writing a
good README file using Sinatra's README as a reference.

* The [learnability](https://github.com/elm-city-craftworks/code_quality/wiki/Learnability)
of our software, i.e. how easy it is to discover new ways of using the software
based on what the user already knows. As an example, I discuss a change we made
to the Prawn graphics API to make things more consistent and easier to learn.

* The [operability](https://github.com/elm-city-craftworks/code_quality/wiki/Operability) 
of our software, particularly whether we give our users the control and
flexibility they need to get their job done. As an example, I discuss how most
Markdown processors in Ruby function as black boxes, and how RedCarpet 2 takes a
different approach that makes it much easier to customize.

* The [attractiveness](https://github.com/elm-city-craftworks/code_quality/wiki/Attractiveness) 
of our software. As an example, I show the difference between low level and high
level interfaces for interacting with the Cairo graphics library, and illustrate
how the use of syntactic sugar can influence user behavior.

### Efficiency concerns

Ruby has had a reputation for being a slow, resource intensive programming
language. As a result, we need to rely on some special tricks to make sure that
our code is fast enough to meet the needs of our users. Issues to consider in
this area include:

* The [performance](https://github.com/elm-city-craftworks/code_quality/wiki/Performance)
of our software. As an example, I talk at a very high level about the
computationally expensive nature of PNG alpha channel splitting, and how C
extensions can be used to solve that problem.

* The [resource utilization](https://github.com/elm-city-craftworks/code_quality/wiki/Resource-Utilization)
characteristics of our software. While this most frequently means memory and
disk space usage, there are lots of different resources our programs use. As an
example, I talk about the fairly elegant use of file locking in the PStore
standard library.

### Maintainability concerns

No matter how good our software is, it will ultimately be judged by how well it
can change and grow over time. This is the area we tend to spend most of our
time studying, because difficult to maintain projects make us miserable as
programmers. Issues to consider in this area include:

* The [analyzability](https://github.com/elm-city-craftworks/code_quality/wiki/Analyzability) 
of our software, i.e. how easy it is for us to reason about our code. As an
example, I discuss at a high level how the Flog utility assigns scores to
methods based on their complexity, and how that can be used to identify areas of
your code that need refactoring.

* The [changeability](https://github.com/elm-city-craftworks/code_quality/wiki/Changeability)
of our software, which is commonly considered the holy grail of software design.
As an example, I point out connascence as a mental model for reasoning about the
relationships between software components and how easy or hard they are to
change.

* The [stability](https://github.com/elm-city-craftworks/code_quality/wiki/Stability)
of our software, in particular how much impact changes have on users. As an
example, I talk about the merits of designing unobtrusive APIs for reducing the
amount of moving parts in our code.

* The [testability](https://github.com/elm-city-craftworks/code_quality/wiki/Testability)
of our software. As an example, I discuss how useful the SOLID principles are in
making our code easier to test.

### Portability concerns

One thing we don't think about often in Ruby, perhaps not often enough, is how
easy it is for folks to get our software up and running in environments other
than our own. While writing code in a high level language does get us away from
some of the problems that system programmers need to consider, there are still
platform and environment issues that deserve our attention. Issues to consider
in this area include:

* The [adaptability](https://github.com/elm-city-craftworks/code_quality/wiki/Adaptability) 
of our software to the user's environment. As an example, I discuss at a high
level the approach HighLine takes to shield the user from having to write low
level console interaction code.

* The [installability](https://github.com/elm-city-craftworks/code_quality/wiki/Installability) 
of our software. As an example, I discuss some general thoughts about the 
state of installing Ruby software, and look into an interesting approach to 
setting up a Rails application in Jordan Byron's Mission of Mercy clinic 
management project.

* The [co-existence](https://github.com/elm-city-craftworks/code_quality/wiki/Co-existence) 
of our software with other software in the user's environment. As an example, 
I discuss how conflicting monkey patches led me on a wild goose chase in 
one of my Rails applications.

* The [replaceability](https://github.com/elm-city-craftworks/code_quality/wiki/Replaceability) 
of our software as well as the ability for our software to act as a drop in 
replacement for other tools. Because I feel this concept is one baked into the UNIX and open 
source culture, I don't provide a specific case study but instead point out several applications 
of this idea in the wild.

### Reflections

Spending several weeks studying this topic just so I can *start* a discussion
with our readers has been a painful, but enlightening experience for me. As you
can see from the giant laundry list of concerns listed above, the concept of
software quality is much deeper than something like [The Four Simple Rules of
Design](http://www.c2.com/cgi/wiki?XpSimplicityRules) might imply. 

It is no surprise that we yearn for something more simple than what I've
outlined here, but I cannot in good conscience remove any of the focus areas
outlined by ISO/IEC 9126 as being unimportant when it comes to software quality.
While we cannot expect that all of our software will be shining examples of all
of these properties all of the time, we do have a responsibility for knowing how
to spot the tensions between these various concerns and we must do our best to
resolve them in a smart way.

While our intuition and experience may allow us to address most of the issues
I've outlined here at a subconscious level, I feel that more work needs to be
done for us to seriously consider ourselves good engineers. The real challenge
for me personally is to figure out how to continue to study these topics without
stifling my creativity, willingness to experiment, and ability to make decisions
without becoming overwhelmed.

I look forward to hearing your own thoughts on this topic, because it is one
that we probably need to work through together if we want to make any real
progress.
