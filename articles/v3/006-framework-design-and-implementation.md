In [Issue 3.5](http://practicingruby.com/articles/22), I  challenged Practicing Ruby subscribers to read through and play with the uncommented source code of [Newman 0.1.1](https://github.com/mendicant-original/newman/tree/v0.1.1), the first release of my micro-framework for building email-centric applications. My hope was that by looking through the implementation of a framework in its very early stages of development, readers would be able to familiarize themselves with the kinds of challenges involved in building this sort of project.

If you didn't participate in that challenge, I recommend spending an hour or two working through it now before reading the rest of this article. My feeling was (and is) that because framework development is about taking care of a thousand tiny details, it's important to see where this kind of project begins before you can really appreciate where it ends up. Assuming you've gone ahead and done that, we can move on to this week's exercise.

### The challenge revisited

I had originally planned to provide a nice annotated walk-through of Newman's implementation up front, but then decided it'd be better if you had a chance to explore it without much guidance before sharing my own explanations of how it all hangs together.

However, this would be little more than an exercise in code reading if I didn't revisit that challenge and provide you with comprehensive implementation notes. With that in mind, you can now read [the fully documented source code](http://mendicant-original.github.com/newman/lib/newman.html), complete with little bits of design ruminations peppered throughout the code. This ought to answer some of the questions that were rattling around in the back of your mind, and even if it didn't, it may spark new questions or ideas that you can share with me as they arise.

Just reading through the documented codebase should teach you a lot about how a micro-framework can be built in Ruby. But the process of doing so might still leave you feeling a bit disoriented because it provides a view of the big picture in terms of dozens of microscopic snapshots rather than exposing a handful of bright-line items to focus on.  With that in mind, I've decided to outline a few of the recurring obstacles I kept running into even in the first week of development on this project, as I think they'll be friction points for frameworks of all varieties.

### Lessons learned the hard way

There is a ton of content to read in that source walk-through, so I'll try to keep these points brief in the hopes that they'll spark some discussions that might possibly lead to further investigation in future articles. But these are the main things to watch out for if you're designing your own framework or contributing to someone else's early stage project:

**1) Dealing with global state sucks**

In the context of Newman, I had to deal with global state in the form of configuration settings, logging, mailer objects, and persistence layers. The first version of Newman had to resort to turning many of its objects into singleton objects because any object which manipulates global state can have global side effects. 

The viral nature of singleton objects was something that I rarely encountered in application or even library code, but became blindingly apparent in working on this framework. For example, Newman's first version shipped with a `Newman::Mailer` object, but because this object was using Mail's global settings functionality, it was not practical to ever create more than one `Newman::Mailer` object. This was really annoying, because it meant that Newman would be limited to monitoring a single inbox per process, which seems like an artificial restriction. But because a `Newman::Server` object is essentially a router designed to work bridge our mailer object to our application objects, it too needed to become a singleton object!

We eventually were able to work around this for the most part by using some low level APIs provided by the mail gem, and this will pave the way for multi-inbox support in Newman in the near future. But I was sort of shocked at how much impact depending on a singleton object can have on the overall flexibility of a framework, because I had not experienced this problem in my usual work on applications and libraries.

For the things that'd be ordinarily implemented as singleton objects, such as loggers or configuration objects, I took care to make it so that even if in practice the system makes use of globally available state, the structure allows for that to be changed with minimal impact. As an example of this, you can see that Newman actually passes its setting objects, data storage objects, and loggers down the call chain to any object that needs them rather than having those objects reference a constant or global variable. This makes it possible for isolation to occur at any point in the chain, and makes it so that Newman has very few hard dependencies on shared state, with its use only being a matter of convenience. The unfortunate side effect of this sort of design is a bit of repetitive code, but I've tried to minimize that where possible by providing convenience constructors and factory methods that make this job easier.

**2) Handling application errors in server software is hard**

In the first version of Newman, any application error would cause the whole server to come crashing down with it. This is definitely not the right way to do things, at least not by default, as it means that a problem with a single callback in a single application can bring down a whole process that is otherwise working as expected.

You might think that the solution to this is to simply rescue application errors and log them, and that is more-or-less the approach I chose to solving this problem. However, I quickly ran into an issue with this in testing. I didn't want a bunch of verbose log output while running my integration tests, so I ran the server with logging turned off. But soon enough, I was getting tests which were failing but giving me no feedback at all as to why that was happening. I eventually discovered that this was due to application errors being swallowed silently, causing the application to fail to respond but not give any feedback that would help with debugging them. The code was not raising an exception, so the tests were not halting with an error, they were just failing.

To solve this issue, I added a server configuration object which allowed toggling exception raising on and off. When that setting was enabled, the server would halt, which is exactly the behavior I wanted in my tests. This did the trick and it's been mostly smooth sailing since then. But the question remains: what are the best defaults to use for this sort of thing? I'm thinking that for test mode, logging should be off and exception raising should be on, and for the default runtime behavior, it should be exactly the opposite. Is that sane? I don't really know, but I suppose it's worth a try.

Another open question I have here is how much effort should be put into only rescuing certain kinds of errors. In most cases, "log and move on" seems like the right behavior from the server's perspective, but could this lead to weird edge cases? As the framework designer, it's my job to help make it hard for the application developer to shoot himself in the foot. But unfortunately right now I don't really know where to aim. More research is required here.

**3) Frameworks are tricky to test**

The whole point of a framework is to tie together a bunch of loose odds and ends to produce a cohesive environment for application development. Writing tests for applications developed within a framework should be *easier* than writing tests for applications developed standalone, but writing tests for the framework itself present new and interesting challenges that aren't typically encountered in ordinary application development.

When I first started working on Newman, I wasn't writing any tests at all
because I had no faith that any of the objects I was creating were going to
survive first contact with real use cases. Instead, I focused on building little
example applications which moved the functionality along and served as a way of
manual testing the code in a fairly predictable way. But after even a modest
amount of functionality was built, the lack of automated testing made it so that
each new change to the system involved a long, cumbersome, and error prone
manual testing session, to the point where it was no longer practical.

From there, I decided to build some simple "live tests" that would essentially
script away the manual checking I was doing, running the example programs
automatically, and automating the sending and checking of email to give me a
simple red/green check. The process of introducing these changes required me to
make some changes to the system, such as allowing the system to run tick by tick
rather than in a busy-wait loop, among other things. This cut down some of the
manual testing time and made the test plan more standardized, but was still very
brittle because cleaning up after a failed test was still a manual process.

Sooner or later, we introduced some surface-level test doubles, such as a simple
TestMailer which could serve as a stand-in replacement for the real mail object.
With this object in place it was possible to convert some of the live tests to a
set of acid tests which were capable of testing the system from end to end
reaching all systems except for the actual mail interactions. This was a huge
improvement because it made basically the same tests run in fractions of a
second rather than half a minute, but I'm still glad it's not where we started
at. Why? Simply because email is a super messy domain to work in and five
minutes of manual testing will expose many problems that hours of work on finely
crafted automated tests would never catch unless you happen to be an email
expert (I'm not). The only thing I regret is that I should have developed these
tests concurrently with my manual testing, rather than waiting until the pain
became so great that the project just ground to a halt.

Even after these improvements, Newman 0.2.0 still ended up shipping with no unit
tests. Part of this is because of the lack of available time I had to write
them, but the other part is that it still feels like a challenge for me to
meaningfully test isolated portions of the framework since they literally will
never be useful in isolation. I'm stuck in between a rock and a hard place,
because the use of mock objects feels too artificial, but dragging in the real
dependencies is an exercise in tedium. I'd love some guidance on how to test
this sort of code effectively and will be looking into how things like Sinatra
and Rails do their testing to see if I can learn anything there.

But one thing is for sure, I wouldn't suggest trying to build a framework unit
test by unit test. The problem domain is way too messy with way too many
interlocking parts for that to be practical. I think if I took the TDD approach
I'd still be working on getting the most basic interactions working in Newman
today rather than talking about it's second major release. Still, maybe I'm just
doing it wrong?

**4) Defining a proper workflow takes more effort than it seems**

This obstacle is hard to describe succinctly, so I won't try to do so. But the
main point I want to stress is that many frameworks are essentially nothing more
than a glorified tool for ferrying a bunch of request data around and building
up a response in the end. But deciding where to sneak in extension points, and
where to give the application developer control vs. where to make a decision for
them is very challenging.

Try to start with one of Newman's examples and trace the path from the point
where an email is received to the point where a response gets sent out. Then let
me know what you think of what I've done, and perhaps give me some ideas for how
I can do it better. I'm still not happy with the decomposition as it is, but I'm
struggling to figure out how to fix it.

**5) Modularity is great, but comes at a cost**

As soon as you decide to make things modular, you have to be very careful about
baking assumptions into your system. This means making interfaces between
objects as simple as possible, limiting the amount of dependencies on shared
state, and providing generic adapter objects to wrap specific implementation
details in some contexts. This is another thing that's hard to express in a
concise way, but the point is that modularity is a lot more complicated when you
are not just concerned about reusability/replaceability within a single
application, but instead within an entire class of applications, all with
somewhat different needs.

I've been trying to ask myself the question of whether a given bit of
functionality really will ever be customized by a third-party extension, and if
I think it will be, I've been trying to imagine a specific use case. If I can't
find one, I decide to avoid generalizing my constructs. However, this is a
dangerous game and finding the right balance between making a system highly
customizable and making it cohesive is a real challenge. It's where all the fun
problems come from with framework design, but is also the source of a lot of
headaches.

### Reflections

I'm afraid dear Practicing Rubyist that once again I've raised more questions
than answers. I always worry when I find myself over my own head that perhaps
I've lost some of you in the process. But I want to emphasize the fact that this
journal is meant to chronicle a collective learning process for all of us, not
just a laundry list of developer protips that I can pull off the top of my head.
Even if this series of articles was hard to digest, it will pave the way for
more neatly synthesized works in the future.

Also, don't underestimate your ability to contribute something to this
conversation! If any ideas or questions popped up in your head while reading
through these notes, please share them without thinking about whether or not
your insights would be worth sharing. I've been continuously impressed by the
quality of the feedback around here, so I'd love to hear what you think.
