In this article, I've listed five people worth knowing about if you're involved in Ruby. If you're reading this in July 2011, please note that I wrote this article over 7 months ago, and so the descriptions you see below are slightly outdated. That having been said, I still think these five people are on the top of my list when it comes to interesting folks in our community.

### Wayne Seguin ([@wayneeseguin](http://twitter.com/wayneeseguin))

Wayne gave us [RVM](http://rvm.beginrescueend.com), the Ruby enVironment Manager. This tool quickly evolved from a bunch of crude shell script hacks to something that makes working with multiple Ruby versions and implementations a breeze. A tool which simply allowed manually switching between versions and implementations of Ruby would be useful on its own, but the thing that makes RVM special are all the shiny extras that come with it.

In addition to basic version switching, RVM provides gemsets which are sandboxes for your gem installation environment. This makes it possible for each of your projects to have its own gemset, eliminating concerns about different projects having dependencies that clash with one another. While this is a problem that can often be solved by version locking, having an extra layer of protection and organization is great.

Another neat feature of RVM is the ability to include a `.rvmrc` in any of your project roots, which causes `rvm` to automatically switch to the desired ruby version, implementation, and gemset that you specify in that file. This reduces the amount of manual switching needed, and makes commands like `ruby`, `irb`, `rake`, and `gem` 'just work' without having to think about what context you are in.

Another thing that is amazing about RVM is the amount of support Wayne offers for it. He is nearly infamous for his availability on IRC, and he seems to genuinely want to help anyone who is trying to use RVM. I've seen him cornered at least a few times at Ruby conferences by folks asking questions about how to do this or that with RVM, and he always seems to handle those situations gracefully. This is exactly the kind of spirit that makes me appreciate someone's work and makes me want to keep watching them to see what great things they'll come up with.

<i>UPDATE 2011.07.19: You should also check out Wayne's [BDSM framework](http://bdsm.beginrescueend.com).</i>

### Eleanor McHugh ([@feyeleanor](http://twitter.com/feyeleanor))

Eleanor McHugh is an incredibly clever and entertaining hacker who has a deep interest in concurrency and low level UNIX plumbing. She spent a lot of time in 2010 working on [GoLightly](http://github.com/feyeleanor/GoLightly), a lightweight virtual machine running on top of the Go programming language. Her original goal was to re-build miniruby on top of Go, but building the vm became a priority in of itself rather than just a stepping stone once she had a chance to dig into the problem.

What interests me about Eleanor is that she is the kind of person that decides to work on a project first and then figure out how to make it all come together later. I know she has been making some significant personal sacrifices so that she can work on GoLightly, and that sort of attitude is something I really like to see.

Eleanor was one of the guest speakers at Mendicant University in 2010, doing a Q&A session with me and the students. We touched on how pretty much every modern language handles concurrency, and then somehow deviated to discussing Eleanor's background in avionics, in which we collectively decided that TDD in that field worked something like "Whoops, the plane crashed, guess that's red." This of course lead us to a more serious discussion about testing and testability, but was a pretty hilarious diversion along the way.

Where one can really learn a ton from Eleanor is in a small group or one on one conversation. She is the ideal person to catch up with on the hallway track of a conference, or to grab a drink with after an event. Each time I've met up with her I've been consistently entertained and inspired by her stories, and find myself fortunate to be able to call her a friend.

<i>UPDATE 2011.07.19: Eleanor, like me, spends most of her time hacking on community projects. [She can use some help with her travel expenses](http://pledgie.com/campaigns/15689), so if you like what she's doing, please do contribute what you can.</i>

### Brian Ford ([@brixen](http://twitter.com/brixen))

Brian is one of the key [Rubinius](http://rubini.us) team members and also was instrumental in the creation and adoption of the [RubySpec](http://github.com/rubyspec/rubyspec), an executable specification of the Ruby language written in RSpec-like syntax.

While I do not closely follow Rubinius, I studied it a bit when researching for a talk on Ruby versions and implementations. In the process, I came to learn about RubySpec and the specialized testing framework they've built for it called [mspec](https://github.com/rubyspec/mspec). This stuff is seriously cool.

As you can imagine, building a testing framework to test Ruby itself is a harder problem than simply testing code you write using Ruby. To account for this, mspec does all sorts of neat things, allowing tests to be restricted to particular versions, implementations, and even specific patch levels of different Ruby packages. Another interesting aspect of mspec's implementation is that because it's designed to help Ruby implementers test their work, the code for implementing the testing framework intentionally uses a minimal subset of Ruby functionality. As someone interested in tricky design problems, I found myself consistently impressed by how mspec is implemented. While I'm not sure exactly how much of this is Brian's handiwork, he is one of the key folks who set the project in motion.

RubySpec itself is really impressive. If you haven't looked through it before, I strongly encourage that you do so. It provides comprehensive unit tests for a huge amount of Ruby's behavior, covering each feature in minute detail. I guarantee you that if you spend a little time reading through the specs, you'll find an edge case about some Ruby feature that you didn't know about, no matter how solid your understanding of Ruby is.

<strike>While we haven't officially announced the details, Brian and I will be working together to run Ruby Mendicant University's first Free Software Clinic. This will be a chance for some of our students to work with me as we contribute something interesting that should make RubySpec even more useful than it already is. More information will come about this topic soon.</strike>

In addition to his work on Rubinius and RubySpec, Brian happens to be an incredible teacher. While most of my interactions with him have been over IRC, he is capable of explaining complex and deep computer science topics in a way that makes them feel natural and manageable. I finally had a chance to see him give a talk in person at RubyConf 2010, and by watching [this video](http://confreaks.net/videos/454-rubyconf2010-poisoning-rubinius-the-_why-and-how), I think you'll get a sense of what I mean.

<i>UPDATE 2011.07.19: Brian and I haven't had a chance to work on open source projects together with the Mendicant University students yet, but I hope we'll have a chance to do so some time in the not-too-distant future. I struck the mention of our plans out in the description above to make it clear this original plan didn't pan out.</i>

### Tony Arcieri ([@bascule](http://twitter.com/bascule))

Tony is another Ruby hacker interested in concurrency, particularly the Actor model of concurrency. He has built a number of concurrency tools in Ruby, including [revactor](http://github.com/tarcieri/revactor), but eventually decided that what he really wanted was the syntax of Ruby with the baked in concurrency model of Erlang. This lead him to begin work on his own language, [Reia](http://github.com/tarcieri/reia).

For those who haven't seen it before, Reia is a fascinating language, even in its infancy. The syntax does look and feel like Ruby, but everything is Erlang under the hood. The functionality is mapped more towards Erlang than it is towards Ruby, which means that Reia is not aiming to be a feature complete Ruby implementation. Working in Reia is an interesting exercise in wondering what a smaller, more basic subset of Ruby's functionality might look like.

The neat thing about Reia is that a lot of its code is self hosting, similar to Rubinius. This, combined with the fact that you can easily reach down to the Erlang runtime and call functions provided in Erlang's core modules, makes it very easy to contribute to Reia's high level feature set. During RubyConf 2010 I decided to dip my toe in and help wrap a number of the methods in Erlang's List API to make them look and feel like the features provided by Ruby's Enumerable module, and I found contributing to the project very easy.

Tony is another hacker who is gifted at being a bit irreverent towards what are typically considered 'hard problems', and like Brian Ford, he is good at helping you understand that building a programming language isn't quite as hard as you might think. You can check this out for yourself by watching his [RubyConf 2010 talk](http://confreaks.net/videos/457-rubyconf2010-rev-revactor-reia).

<i>UPDATE 2011.07.19: Tony's projects move fast. I wouldn't be surprised if everything above is now out of date, but hunt down whatever he's working on now and you won't be disappointed.</i>

### Eric Hodel ([@drbrain](http://twitter.com/drbrain))

Eric has been in the Ruby community for as long as I can remember, and as a member of the Seattle Ruby Group, he automatically can be recognized as an insanely capable hacker.

What I feel Eric lacks is enough appreciation from the community for the very thankless work he was doing. Anyone who was around in Ruby before Rails knows that RubyGems greatly outgrew its initial design a long time ago. The code, originally hacked together at a conference, was never really meant to live in a world in which gem downloads are measured in the millions rather than the hundreds.

Similar arguments could be made about projects such as RDoc. Being able to autogenerate documentation is an important part of any language's infrastructure, but when Dave Thomas first put together RDoc, I doubt he could have anticipated how big Ruby would be and how long that code would still remain in active use.

Most people didn't want to touch RubyGems or RDoc, both because of how outdated the code was, and because any small change to either of them could easily piss off the entire Ruby world. But the more that Ruby's ecosystem evolved, the more it became clear that fighting against old, janky architecture was a huge waste of time.

Little by little, Eric worked towards fixing up both of these projects. Now, both RDoc and RubyGems are much, much better than what they were before. Each have extension systems that make it so that the core code can continue to get smaller and simpler over time, rather than the other way around. In the case of RubyGems, that extension system brought us Gemcutter (now rubygems.org), which is now the official means of distributing gems to the Ruby community. While we have Nick Quaranto to thank for this innovation, we have Eric to thank for making RubyGems better so that Gemcutter could actually come into existence in the first place.

If there is one person in the Ruby community that deserves thanks for taking our old and busted tooling and making it serviceable again, it's Eric.

<i>UPDATE 2011.07.19: Even despite the RubyGems turbulence over the last several months, I stand by this opinion of Eric's contributions 100%</i>

### Who's interesting to you?

These are the folks who caught my interest over the last year or so. Who is someone you think is worth knowing about?

  
> **NOTE:** This article has also been published on the Ruby Best Practices blog. There [may be additional commentary](http://blog.rubybestpractices.com/posts/gregory/048-issue-16-interesting-ruby-hackers.html#disqus_thread) 
over there worth taking a look at.
