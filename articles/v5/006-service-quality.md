Software projects need to evolve over time, but they also need to avoid
collapsing under their own weight. This balancing
act is something that most programmers understand, but it is often 
hard to communicate its importance to nontechnical stakeholders. 
Because of this disconnect, many projects operate under the
false assumption that they must stagnate in order to stabilize. 

This fundamental misconception about how to maintain a stable codebase has some
disastrous effects: it causes risk-averse organizations to produce stale 
software that quickly becomes irrelevant, while risk-seeking organizations ship 
buggy code in order to rush features out the door faster than their 
competitors. In either case, the people who depend on the software produced by
these teams give up something they shouldn't have to.

I have always been interested in this problem, because I feel it is at the 
root of why so many software projects fail. However, my work on Practicing Ruby
has forced me to become much more personally invested in solving it. As someone
attempting to maintain a very high-quality experience on a shoestring budget, I
now understand what it is like to look at this problem from a stakeholder's
point of view. In this article, I will share the lessons that Jordan Byron and 
I have learned from trying to keep Practicing Ruby's web application stable as
it grows.

### Lesson 1: Work incrementally

Inspired by [Lean Software Development][lean] practices, we now
view all work-in-progress code as a form of waste. This way of looking at things 
has caused us to eschew iteration planning in favor of shipping a single
improvement or fix at a time. This workflow may seem a bit unrealistic at 
first glance, but with some practice it gets easier to break very 
complicated features into tiny bite-sized chunks. We now work this way by
habit, but our comment system was the first thing we approached in 
this fashion.

When we first implemented comments, we had Markdown support, but not much else. 
Later, we layered in various improvements one by one, including syntax 
highlighting, email notifications, Twitter-style mentions, and Emoji support. 
With so little development time available each week, it would have taken 
months to ship our discussion system if we attempted to build it all at once.
With that in mind, our adoption of a Lean-inspired deployment strategy was not just a
workflow optimization—it was an absolute necessity. We also eventually came to 
realize that this constraint was a source of strength rather than weakness for
us. Here's why:

> Developing features incrementally reduces the number of moving parts 
to integrate on each deploy. This reduction in turn limits the number of new
defects introduced during development.

> When new bits of functionality do fail, finding the root cause of the problem is usually easy, and even when it isn't, rolling the system
back to a working state is much less traumatic. Together, these approaches result in
a greatly reduced day-to-day maintenance cost, which means that more time can
be spent on value-producing work.

As you read through the rest of the guidelines in this article, you'll find that
although they are useful on their own, they are made much more effective by this
shift in the way we ship things.

### Lesson 2: Review everything

It is no secret that code reviews are useful for driving up quality and
reducing the number of defects that are introduced into production in the first
place. However, figuring out how to conduct a good review is something that
takes a bit of fine-tuning to get right. Here is the set of steps we eventually
settled on:

1. The reviewer attempts to actually use the new feature while its developer 
answers any questions that come up along the way. Whenever an unanticipated
edge case or inconsistency is found, we immediately file a ticket for it. We
repeat this process until all open questions or unexpected issues have been 
documented.

    Unless the feature's developer has specific technical questions for the
    reviewer, we don't bother with in-depth reviews of implementation details until
    all functional issues have been addressed. This prevents us from spending time
    on bikeshed arguments about refactorings or hypothetical sources of
    failure at the code level. Doing things this way also reminds us that the
    external quality of our system is our highest priority and that although clean
    code makes building a better product easier, it is a means, not an end in itself.

2. Once a feature seems to work as expected by both the developer and
the reviewer, we next turn our attention to the tests. It is the
reviewer's responsibility to make sure that the tests cover the issues brought
up during the review and to verify that they exercise the 
feature well enough to prevent it from silently breaking. Sometimes the reviewers will ask the developer
of the feature to write the tests; other times it is easier for the reviewers to
write the tests themselves rather than trying to explain what is needed. 

    In either case, the end result of this round of changes is that the feature's
    requirements become clearer as the tests are updated to cover more subtle
    details. Because many of these tests can be written at the UI level, it is
    common to have not yet discussed implementation details at this stage of a
    review.

3. By now, the feature is tested well enough, and its functionality has been 
exercised more than a few times. That means that a spot check of its source code 
is in order. The goal is not to make the code perfect, 
but to identify both low-cost improvements that can be done right away 
and any serious warning signs of potential problems that may make the 
code hard to maintain or prone to error. Everything else is 
something that can be dealt with later—if and when a feature needs to be 
improved or modified.

Even though these items are listed in order, it's better to think of them as layers
rather than procedural steps. You need to start at the outermost layer, then
dig down as needed to fully answer each question that comes up during a review.
This may sound like a very rigorous procedure, but it isn't as daunting as it
seems. You can get an idea of what this process looks like in practice by reading through 
the conversation on [this pull request][pr-76]. Here's why we invest the extra
effort:

> Reviewing functionality first, tests second, and
> implementation last helps ensure that the right kinds of
> conversations happen at the right time. If a feature isn't implemented
> correctly or is poorly usable, it doesn't matter how well written its tests
> are. Likewise, if test coverage is inadequate, it isn't wise to recommend 
> major refactorings to production code. This simple prioritization keeps
> the focus on improving the *application* rather than the *implementation*.

Even with a very good review process, bad things still happen. That's
why the remaining four lessons focus on what to do when things go wrong, but keep
in mind that actively reviewed projects help prevent unexpected failures from
happening in the first place.

### Lesson 3: Stay alert

When something breaks, we want to know about it as soon as possible.
We rely on many different ways of detecting problems, and we automate as much as
we can.

Our first line of defense is our continuous integration (CI) system. 
We use [Travis CI][travis], but for our purposes pretty much any CI tool would
work. Travis does a great job of catching environmental issues for us: things
such as unexpected changes in dependencies, application configuration problems,
platform-specific failures, and other subtle things that would be hard to 
notice in development. More important, it helps protect us from ourselves: even if we 
forget to run the entire test suite before pushing a set of changes, 
Travis never forgets, and will complain loudly if we've broken the 
build. Most of the mistakes that CI can detect are quite 
trivial, but catching them before they make it to production helps us 
keep our service stable.

For the bugs that Travis can't catch (i.e., most of them), we rely on
the [Exception Notifier][exception-notification] plugin for Rails. Most
notification systems would probably do the trick for us, but we like that Exception
Notifier is email-based; it fits into our existing workflow nicely. The default
template for error reports works great for us because it provides everything
you tend to get from debugging output during development: session data,
environment information, the complete request, and a stack trace. If we start to
notice exception reports rolling in soon after we've pushed a change to the system, this
information is usually all we need in order to find out what caused the problem.

Whenever we're working on features that are part of the critical path of our
application, we tend to use the UNIX `tail -f` command to watch our production 
logs in real time. We also occasionally write ad hoc reports that give us 
insight into how our system is working. For example, we built the following 
report to track account statuses when we rolled out a partial replacement 
for our registration system. We wanted to make sure it was possible for folks to
successfully make it to the "payment pending" status, and the report showed
us that it was:

![Account status report](http://i.imgur.com/NOI0A.png)

Our proactive approach to error detection means that we can rely less on
bug reports from our subscribers and more on automated reports and alerts. This approach
works fairly well most of the time, and we even occasionally send messages
to people who were affected by bugs with either an apology and a note that we
fixed the problem, or a link to a ticket where they can track our progress on
resolving the issue. We do display our email address on all of our error pages,
but we place a high priority on making sure that subscribers need to use
it only to provide extra context for us, rather than to notify us that a
problem exists.

Before we move on to the next lesson, here are a few things to remember about
this topic:

> The main reason to automate error detection as much as possible is that the
people who use your application should not be treated like unpaid QA testers.
The need for an active conversation with your users every time something goes
wrong is a sign that you have poor visibility into your application's failures,
and it will pay off to fix this problem. However, every automated error
detection system requires some fine-tuning to get it right, and you may need
to make pragmatic compromises from time to time.

> Automated error detection is almost always a good thing: the main question is how extensive
> you want it to be. For small projects, something as simple as maintaining a
> detailed log file is enough; for larger projects, much more sophisticated
> systems are needed. The key is to choose a strategy that works for your
> particular context, rather than trying to find a one-size-fits-all
> solution.

If automated error detection interests you, please post a
comment about your experiences after you finish reading this article. It
is a very complex topic, and I feel like I've only scratched the surface
of it in my own work, so I'd love to hear some stories from our readers.

### Lesson 4: Roll back ruthlessly

Working on one incremental improvement at a time makes it easy 
to revert newly released functionality immediately if we find 
out that it is defective. At first, we got into the habit of 
rolling things back to a stable state because we didn't
know when we'd get around to fixing the bugs we encountered. Later, we discovered that this approach allows us to take
our time and get things right rather than shipping quick
fixes that felt like rushed hacks.

In order to make rollbacks painless, good revision control processes
are essential. We started out by practicing [GitHub Flow][gh-flow]
in its original form, which consisted of the following steps:

1. Anything in the master branch is deployable.

2. To work on something new, create a descriptively named branch off of the master.

3. Commit to that local branch and regularly push your work to the server.

4. When you need feedback or help, or you want to merge, open a pull request.

5. After someone else has reviewed the feature, you can merge it into the master.

6. Once it is merged and pushed to the master, you can and should deploy
immediately.

Somewhere down the line, we made a small tweak to the formula by deploying
directly from our feature branches before merging them into the master branch. This
approach allows every improvement we ship to get some live testing time in
production before it gets merged, greatly increasing the stability
of our mainline code. Whenever trouble strikes, we redeploy from our master branch, 
which executes a rollback without explicitly reverting any
commits. As it turns out, this approach is very similar to [GitHub's more recent
deployment practices][gh-deploy-aug-2012], minus their fancy robotic
helpers.

Although this process significantly reduces the amount of defects on our master branch,
we do occasionally come across failures that are in old code rather than in our
latest work. When that happens, we tend to fix the issues directly on the master, 
verify that they work as expected in production, and then attempt to merge those changes into any active
feature branches. Most of the time, these merges can be cleanly applied, so
it doesn't interrupt our work on new improvements all that much. But when things
get messy, it is a reminder for us to take a step back and look at the big
picture:

> In a healthy system, rollbacks should be easy, particularly when feature
branches are used. When this process does not go smoothly, it is usually a 
sign of a deeper problem:

> 1) If lots of bugs need to be fixed on the master branch,
it is a sign that features may have been merged prematurely, or that
the application's integration points have become too brittle and need some 
refactoring.

> 2) If a new feature repeatedly fails in production despite attempts to fix
it, it may be a sign that the feature isn't very well thought out and that
a redesign is in order.

> Although neither of these situations are pleasant to deal with, addressing them
right away helps prevent them from spiraling out of control. This approach makes sure
that small flaws do not evolve into big ones and minimizes the project's
pain points over the long haul.

Despite these benefits, this practice does feel a bit ruthless at times, 
and it definitely takes some getting used to. However, by treating rollbacks 
as a perfectly acceptable response to a newly discovered defect rather 
than an embarrassing failure, a totally different set of priorities are 
established that help keep things in a constant state of health. 

### Lesson 5: Minimize effort

Every time we find a defect in one of our features, we ask ourselves whether
that feature is important enough to us to be worth fixing at all. Properly
fixing even the simplest bugs takes time away from our work on other
improvements, so we are tempted to cut our losses by removing
defective code rather than attempting to fix it. Whether we can get away with
that ultimately depends on the situation.

---

**Critical defects:** Sometimes bugs are severe enough that they need to be dealt with right away,
and in those cases we [stop the line][autonomation] to give the issue the
attention it deserves. The best example of this practice that we've encountered in recent
times was when we neglected to update our omniauth dependency before GitHub shut
down an old version of their API, which disabled logins temporarily for all
Practicing Ruby subscribers. We had an emergency fix out within hours, but it
predictably broke some stuff. Over the next couple days, we added fixes for
the edge cases we hadn't considered until the system stabilized again. Because
this wasn't the kind of defect we could easily work around or roll back from, we
were working under pressure, and attempting to work on other things during that
time would have just made matters worse.

**Trivial defects:** At the other extreme, some bugs are so easy to fix that it makes
sense to take care of them as soon as you notice them. A few weeks before this
article was published, I noticed that our broadcast email system was treating our
plain-text messages as if they were HTML that needed to be escaped, which caused
some text to be mangled. If you don't count the accompanying test, fixing this
problem was [a one-line change][htmlescape] to our production code. Tiny bugs 
like this should be fixed right away to prevent them from accumulating 
over time.

**Moderate defects:** Most of the bugs we discover fall somewhere 
between these two extremes, and figuring out how to deal with them is not nearly so
straightforward. We've gradually learned that it is
better to assume that a feature can be either cut or simplified and then try to
prove ourselves wrong rather than thinking that it absolutely must be fixed.

One area where we failed to keep things simple at first was in our work on
account cancellation. Because we were in the middle of a transition to a 
new payment provider, this feature ended up being more complicated to
implement than we expected. After several hours of discussion and
development, we ended up with something that almost worked but still had
many kinks to be ironed out. Almost immediately after we deployed the feature
to production, we noticed that it wasn't working as expected and
immediately rolled it back.

We thought for some time about what would be needed in order to fix the
remaining issues and eventually came to realize that we had overlooked an
obvious shortcut: instead of fully automating cancellations, we could make it so
the unsubscribe button sent us an email with all the details necessary to close
accounts upon request. This process takes only a few seconds to do manually and
happens only a few times a week. Most important, the semi-automatic
approach was easy to understand with few potential points of failure and could
be designed, implemented, and tested in less time than it took for us to think
through the issues of the more complicated system. In other words, it required
less effort to ship this simple system than it would have taken to fix the
complicated one, so we scrapped the old code.

---

Every situation is different, but hopefully these examples have driven home the
point that dealing with bugs requires effort that might or might not be better spent
elsewhere. In summary:

> Critical flaws and trivial errors both deserve immediate attention: the former
> because of their impact on people, the latter due to the fact that they get
> harder to fix as they accumulate. Unfortunately, most bugs are in-between these two extremes and must be evaluated on a case-by-case basis.

> You can't just decide whether a bug is worth fixing based on the utility of the
> individual feature it affects: you need to think about whether your time would be
> better spent working on other things. It is worth resolving defects only
> if the answer to that question is "No!". Even if it is emotionally challenging
> to do so, sometimes it makes sense to kill off a single buggy feature if doing
> so improves the overall quality of your system.

Of course, if you do decide to fix a bug, you need to do what you can to prevent
that time investment from going to waste. Regression testing can help with that,
and that's why we've included it as the sixth and final lesson in this article.

### Lesson 6: Prevent regressions 

One clear pattern that time has taught us is that all bugs that are not covered by
a test eventually come back. To prevent this from happening, we
try to write UI-level acceptance tests to replicate defects as the first step 
in our bug-fixing process rather than the last.

Adopting this practice was very tedious at first. Even though [Capybara][capybara]
made it easy to simulate browser-based interactions with our application, 
dropping down to that level of abstraction every time we found a new
defect both slowed us down and frustrated us. We eventually realized that we
needed to reduce the friction of writing our tests if we wanted this good habit
to stick. To do so, we started to experiment with some ideas I hinted at
back in [Issue 4.12.1][pr-4.12.1]: application-specific helper objects for 
end-to-end testing. We eventually ended up with tests that look something like
the following example:

```ruby
class ProfileTest < ActionDispatch::IntegrationTest
  test "contact email is validated" do
    simulate_user do
      register(Support::SimulatedUser.default)
      edit_profile(:email => "jordan byron at gmail dot com")
    end

    assert_content "Contact email is invalid"
  end

  # ...
end
```

If you strip away the syntactic sugar that the `simulate_user` method provides,
you'll find that this is what is really going on under the hood:

```ruby
test "contact email is validated" do
  user = Support::SimulatedUser.new(self)

  user.register(Support::SimulatedUser.default)
  user.edit_profile(:email => "jordan byron at gmail dot com")

  assert_content "Contact email is invalid"
end
```

Even without reading the [implementation of Support::SimulatedUser][simulated-user],
you have probably already guessed that it is a simple wrapper around Capybara's
functionality that provides application-specific helpers. This object provides
us with two main benefits: reduced duplication in our tests, and a vocabulary
that matches our application's domain rather than its delivery mechanism. The
latter feature is what reduces the pain of assembling tests to go along 
with our bug reports.

Let's take a moment to consider the broader context of how this email
validation test came into existence in the first place. Like many changes we
make to Practicing Ruby, this particular one was triggered by an exception
report that revealed to us that we had not been sanity-checking email 
addresses before updating them. This problem was causing a 500 error to be 
raised rather than failing gracefully with a useful failure message, which pretty
much guaranteed a miserable experience for anyone who encountered it. The steps
to reproduce this issue from scratch are roughly as follows:

1. The user registers for Practicing Ruby.
2. The user attempts to edit his or her profile with a badly formatted email address.
3. The user *should* see a message saying that the email is invalid but
instead encounters a 500 error and a generic "We're sorry, something went wrong"
message.

If you compare these steps to the ones that are covered by the test, you'll see
that they are almost identical to one another. Although the verbal description is
something that may be easier to read for nonprogrammers, the tests communicate
the same idea at nearly the same level of abstraction and clarity to anyone who
knows how to write Ruby code. Because of this, it isn't as easy for us to
come up with a valid excuse for not writing a test or putting it off until
later.

Of course, old habits die hard, and occasionally we still cut corners when
trying to fix bugs. Every time we encounter an interaction that our
`SimulatedUser` has not yet been programmed to handle, we experience the same
friction that makes it frustrating to write acceptance tests in the first place.
When that happens, it's tempting to put things off or to cobble together a test
in haste that verifies the behavior, but in a sloppy way that doesn't make
future tests easier to write. The lesson here is simple: even the most
disciplined processes can easily break down when life gets too busy or too
stressful.

To mitigate these issues, we rely once again on the same practice that allows 
us to let fewer bugs slip into production in the first place: active peer
review. Whenever one of us fixes a bug, the other person reviews it for quality and
completeness. This process puts a bit of peer pressure on both of us to not be sloppy
about our bug fixes and also helps us catch issues that would otherwise hide
away in our individual blind spots. 

In summary, this approach towards regression testing has taught us the following
lesson:

> Any time not spent hunting down old bugs or trying to pin down new ones is 
time that can be spent on value-producing work. Automated testing can really
help in this context, but only if the friction of writing new high-level tests
is minimized.

> Even with convenient application-level test helpers, it can still be tedious
to test behaviors that haven't been considered before, which makes it tempting
to cut corners or to leave out testing entirely in the hopes that someone else 
will get to it later. To keep us from doing this, bug fixes should be reviewed for 
quality just as improvements are, and their tests should be augmented as
needed whenever they seem to come up short.

It does require a little bit of will power, but this habit can work
wonders over time. The trick is to make practicing it as easy as 
possible so that it doesn't bog you down.

### Reflections

Do we follow all of these practices completely and consistently without fail? Of
course not! But we do try to follow them most of the time, and we have
found that they work best when done together. That's not to say
that removing or changing any one ingredient would spoil the soup, only that
it's hard for us to guess what their effects would be like in isolation.

It's important to point out that we adopted these ideas organically rather
than carefully designing a process for ourselves to rigidly follow. This article
is more of a description of how we viewed things at the time it was
published than a prescription for how people ought to approach all
projects all the time. We've found that it's best to maintain a consistent
broad-based goal (ours is to make the best possible user experience with the
least effort) and to continuously tweak your processes as needed to meet that
goal. Working habits need to be treated with a bit of fluidity because brittle
processes can kill a project even faster than brittle code can.

In the end, much of this is very subjective and context dependent. I've shared
what works for us in the hopes that it'll be helpful to you, but I want to
hear about your own experiences as well. Because our process is
nothing more than an amalgamation of good ideas that other people have come up
with, I'd love to hear what you think might be worth adding to the mix.

> **UPDATE**: Although this article recommends using `tail -f` to watch logs in real
> time, it may be [better to use less +F][less], because it makes scrollbacks
> easier and can resume real-time monitoring at any time. Thanks to @sduckett for the suggestion.

[mendicant]: http://mendicantuniversity.org
[travis]: http://about.travis-ci.org/docs/user/getting-started/
[lean]: http://en.wikipedia.org/wiki/Lean_software_development 
[exception-notification]: https://github.com/smartinez87/exception_notification
[gh-flow]: http://scottchacon.com/2011/08/31/github-flow.html
[capybara]: https://github.com/jnicklas/capybara
[pr-4.12.1]: http://practicingruby.com/articles/66
[simulated-user]: https://github.com/elm-city-craftworks/practicing-ruby-web/blob/f00f89b0a547829aea4ced523a3d23a136f1a6a7/test/support/simulated_user.rb
[autonomation]: http://en.wikipedia.org/wiki/Autonomation
[htmlescape]: https://github.com/elm-city-craftworks/practicing-ruby-web/commit/223ca92a0b769713ce3c2137de76a8f34f06647e
[gh-deploy-aug-2012]: https://github.com/blog/1241-deploying-at-github
[pr-76]: https://github.com/elm-city-craftworks/practicing-ruby-web/pull/76
[less]: http://blog.libinpan.com/2009/07/less-is-better-than-tail/
