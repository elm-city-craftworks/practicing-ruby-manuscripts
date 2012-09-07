My article on the [SOLID design
principles](http://practicingruby.com/articles/52) was inspired by a great talk
I saw by Sandi Metz at GoRuCo 2009. Coincidentally, this  article is inspired by
another great talk I saw in 2009, called [The Building Blocks of
Modularity](http://confreaks.net/videos/77-mwrc2009-the-building-blocks-of-modularity).
This talk was given by Jim Weirich at MWRC, and if you haven't seen it yet, I
urge you to stop what you're doing and watch it right now.

In the talk, Jim jokingly claims he's presenting on the "Grand Unified Theory of
Software Development". Personally, I think that isn't too far off the mark,
because connascence is a fundamentally simple concept when compared to things
like the SOLID principles or any of other design concepts we'll be studying in
this series.

### Brief introduction to connascence for the uninitiated

Since I didn't know the concept of connascence even existed before seeing Jim's
talk, and because it's not a super common discussion topic even among design
geeks, we should at least steal some [content from
Wikipedia](http://en.wikipedia.org/wiki/Connascent_software_components) to frame
our discussion around:

><i>"Two software components are connascent if a change in one would require the other to be modified in order to maintain the overall correctness of the system. Connascence is a way to characterize and reason about certain types of complexity in software systems."</i>

If you haven't watched Jim's talk yet, I'll remind you to go ahead and do that
now. But if some reason you can't or won't, you should know that the kinds of
complexity that connascence can be used to reason about all have
to do with coupling. The relationship between the concept of connascence to the
concept of coupling becomes a little more clear when you look at the various
kinds of connascence that can be found in software systems. Below I've listed
out the various kinds of connascence in order from weakest to strongest:

* <b>Name:</b> when multiple components must agree on the name of an entity.
* <b>Type:</b> when multiple components must agree on the type of an entity.
* <b>Meaning:</b> when multiple components must agree on the meaning of specific values.
* <b>Position:</b> when multiple components must agree on the order of values.
* <b>Algorithm:</b> when multiple components must agree on a particular algorithm.
* <b>Execution (order):</b> when the order of execution of multiple components is important.
* <b>Timing:</b> when the timing of the execution of multiple components is important.
* <b>Identity:</b> when multiple components must reference the entity. 

Knowing the various kinds of connascence gives us a metric for determining the characteristics and severity of the coupling in our systems. The idea is simple: The more remote the connection between two clusters of code, the weaker the connascence between them should be.

Good design principles encourages us to move from tight coupling to looser
coupling where possible. But connascence allows us to be much more specific
about what kinds of problems we're dealing with, which makes it easier to reason
about the types of refactorings that can be used to weaken the connascence
between components.

In this article, I will show how you can convert instance of Type, Meaning, 
Position, and Algorithm-based
connascence down to Connascence of Name. While all forms of connascence are
worth studying, these ones are the most likely to appear in your daily work.

### Connascence of Name

Name based coupling exists when a name change in one place requires a code
change in other places. Being the weakest form of connascence, it's also by far
the most common. Every module, class, method and variable we create introduces
connascence of name, assuming it is actually used for something. As an example,
consider the following code:

```ruby
mailer = Mailer.new
mailer.deliver(:to      => "gregory.t.brown@gmail.com", 
               :from    => "fake@fake.com",
               :subject => "You have won a lifetime supply of...",
               :body    => "Dear Sir, I am pleased to inform...")
```

In just this script, we see an incredible amount of name based coupling. Any of
the following trivial changes to Mailer would cause all code that depends on it
to break immediately.

* Wrapping the Mailer class definition in a namespace, e.g. `FancyUnicorn::Mailer`

* Renaming the `deliver()` method to `send_message()`

* Renaming any of the keys in the hash passed to `deliver()`, i.e. changing
the `:to` key so that it reads `:recipient`

But the fact is, there isn't really any way around this sort of coupling in most
scenarios, and it's not necessarily a sign of a problem. That having been said,
the reason why naming things is so important in computer science is because even
loosely coupled, highly cohesive systems have copious amounts of name based
coupling, which have widespread effects that only increase as systems get more
complex.

Sometimes, it is possible to eliminate Connascence of Name and the the coupling
that comes along with it. For example, consider this way of defining class
methods in Ruby:

```ruby
class Mailer
  def Mailer.configure(*args)
    #...
  end
end
```

There is a clear dependence in this code between the second line of code and the
first, in which if the first line changes, so too must the second line. We can
rewrite this to avoid that coupling, if we just take advantage of Ruby's `self`
keyword here:

```ruby
class Mailer
  def self.configure(*args)
    #...
  end
end
```

But while eliminating Connascense of Name is desireable when it is both possible
and convenient to do so, it's not always realistic. We need to accept that
because names don't change all that often, a little bit of CoN is often
harmless. In fact, when given the choice between CoN and other forms of
connascence, name based coupling is preferable. We will now take
a look at the other forms of connascence to see why that is the case.

### Connascence of Type

Folks like to think that Ruby is immune to typing issues, but that 
assumption is often far too optimistic. The following code is a 
direct example of Connascence of Type:

```ruby
def average(values)
   raise ArgumentError unless values.kind_of?(Array)

   values.inject(:+) / values.size.to_f
end
```

One might attempt to resolve the problem by moving away from strict class
checking and instead use a `respond_to?()` check:

```ruby
def average(values)
   unless values.respond_to?(:inject) && values.respond_to?(:size)
     raise ArgumentError 
   end

   values.inject(:+) / values.size.to_f
end
```

This certainly loosens the type coupling, but does not eliminate it. If we
accept the notion that the type of a Ruby object is defined by what that object
can do, `respond_to?()` is still a form of type check, done at the method level
instead of at the level of the class hierarchy. It can sometimes even result in
false negatives, because not all code which implements dynamic behavior through
`method_missing()` updates `respond_to?()` to add those methods. This can lead
to code similar to previous example to fail with certain kinds of proxy objects,
even though they implement all necessary behaviors.

To truly free ourselves from Connascence of Type, one option is to just remove
the guard clause and let Ruby bubble up with an exception for objects that don't
work as our code expects them to. But sometimes, we want to make sure our
debugging isn't harder than it needs to be. Here's an alternative that preserves
the error handling in a way that is free of type dependencies:

```ruby
def average(values)
   values.inject(:+) / values.size.to_f
rescue NoMethodError
   raise ArgumentError, "The average() function can only " +
                        "operate on collections which implement " +
                        "the inject() and size() methods"
end
```

If this feels a bit overkill, it's because it probably is. But the general idea
of removing the `kind_of?()` and `respond_to?()` checks is a good one, because
it puts us squarely back in the realm of Connascence of Name. Our dependency is
now simply that the values object has a pair of methods with the names
`inject()` and `size()`.

### Connascence of Meaning

In its most basic form, Connascence of Meaning is all about magic values. For
example, consider a legacy access control system in
which an admin is given the value 0, a manager 1, and an ordinary user 2. You
could end up writing code like this:

```ruby
if user.access_level == 0
  shoot_nukes_at_moon
else
  raise AccessDeniedError
end
```

The trouble is, once you've littered your system with hard-coded numeric values,
you will have a hard time remembering what they do, and will have a hard time
hunting them down when they need to be changed. To fix this issue, we can modify
our hypothetical `User` object:

```ruby
class User
  ACCESS_LEVELS = { 0 => :admin, 1 => :manager, 2 => :user }

  def admin?
    ACCESS_LEVELS[access_level] == :admin
  end
end
```

We try to avoid repeating Connascence of Meaning even in the more local context
of the `User` class by storing the actual role mappings in a constant. We then
provide a convenience method `User#admin?` to be used externally, resulting in
newly minted caller code that looks like this:

```ruby
if user.admin?
  shoot_nukes_at_moon
else
  raise AccessDeniedError
end
```

Now I don't know about you, but I think I'd be much less likely to accidentally
nuke the moon if the code were written this way. We haven't totally 
eliminated the `Connascence of Meaning`, but we've moved it to a hyper-local 
context within a single constant on the User model. Because all of the 
calling code is now just exhibiting Connascence of Name, this is a great 
refactoring.

### Connascence of Position

Connascence of Position is something that we see every day in Ruby because
method parameters are ordered. If we go to our mailer example, we could have
just as easily written the `Mailer#deliver()` method to use explicitly ordered
parameters, similar to what is shown in the example below.

```ruby
class Mailer
  def deliver(to, from, subject, message)
    # ...
  end
end
```

APIs like this annoy the heck out of me, because the calling code typically
doesn't give any hints at why the arguments are specified in a particular order.
Take a look at how opaque things get when we just try to reproduce our previous
example using this slightly different API:

```ruby
mailer.deliver("gregory.t.brown@gmail.com", 
               "fake@fake.com",
               "You have won a lifetime supply of...",
               "Dear Sir, I am pleased to inform...")
```

Looking at this code, it's very difficult to determine who is the sender and who
is the recipient, and even more difficult to think about how you might introduce
default values into the mix. Every change to the ordering or length of the list
of arguments can lead to broken code in remote places in your codebase. For all
of these reasons, Rubyists tend to prefer hash-based pseudo-keyword arguments
for all but the most straightforward method signatures.

However, introducing keyword arguments isn't the only way to reduce CoP in
method signatures to CoN. Another alternative that is perhaps underused is to
simply create objects that provide all the necessary attributes that you would
typically use a hash for. In this case, we can envision a simple `Message`
object being introduced:

```ruby
message = Message.new
message.to      = "gregory.t.brown@gmail.com"
message.from    = "fake@fake.com"
message.subject = "You have won a lifetime supply of..."
message.body    = "Dear Sir, I am pleazed to inform..."

mailer.deliver(message)
```

Assuming that the `Mailer#deliver` method just depends on the attribute readers
for those attributes, this is functionally equivalent to the hash based code but
offers a number of advantages. `Message` is now a reusable, independently
testable entity that can do things like validations internally. This moves some
of the error checking and simple transformation code that might be needed to use
a parameters hash into a more local setting. With a little creativity, it's
relatively easy to make the API look a little nicer by letting `Mailer#deliver`
create the message object for you.

```ruby
mailer.deliver do |message|
  message.to      = "gregory.t.brown@gmail.com"
  message.from    = "fake@fake.com"
  message.subject = "You have won a lifetime supply of..."
  message.body    = "Dear Sir, I am pleazed to inform..."
end
```

This sort of API is fairly common in Ruby as well, but probably not as common as
it should be. So next time you're faced with the CoP problem when dealing with
method arguments, consider fixing it by putting a nice shiny new object in
place.

It's worth noting that Connascence of Position is certainly not limited to
method arguments in Ruby. Anywhere in which a change in position of some data
requires you to change code elsewhere, you've got a CoP issue, and should think
about how to reduce it if possible.

### Connascence of Algorithm

Connascence of Algorithm often looks and smells a bit like the DRY principle.
But there are many cases in which code that violates DRY doesn't have a CoA
problem, and some rare cases where the opposite is true. The key thing about CoA
is the dependency between two or more clusters of code.

The following example is a CoA example from the wild, from a programming quiz
site that we're working on as part of Mendicant University's admission process.
First, you can see a fairly DRY model which is meant to compare uploaded
solutions to the actual answer for a given puzzle.

```ruby
class Puzzle < ActiveRecord::Base
  def file=(tempfile)
    write_attribute :fingerprint, sha1(tempfile)
  end

  def valid_solution?(tempfile)
    fingerprint == sha1(tempfile)
  end

  private

  def sha1(tempfile)
    Digest::SHA1.hexdigest(tempfile.read)
  end
end
```

Internally, this code is fairly free of CoA, particularly because the algorithm
for fingerprinting solutions has been extracted into the `Puzzle#sha1()` helper.
But because this is a private method, I ended up with tests that explicitly do
the hashing themselves to verify that the `Puzzle#file=()` method is working as
expected.

```ruby
test "must be able to create a fingerprint for a file" do
  tempfile = Tempfile.new("puzzle_sample")
  tempfile << "Sample Text"
  tempfile.rewind

  expected_fingerprint = Digest::SHA1.hexdigest(tempfile.read)
  tempfile.rewind

  puzzle = Puzzle.new(:file => tempfile)

  assert_equal expected_fingerprint, puzzle.fingerprint
end
```

This has an upside in that it sanity checks the exact behavior, ensuring that
the tempfile is actually hashed via SHA1. But since the focus of the test is
more on ensuring that a hash was generated rather than the way it was generated,
we might be able to improve this by extracting the fingerprinting code into its
own module.

```ruby
module Fingerprint
  extend self

  def [](stream)
    Digest::SHA1.hexdigest(stream.read)
  end
end
```

Then, I could rewrite the code and tests to look like they do below:

```ruby
class Puzzle < ActiveRecord::Base
  def file=(tempfile)
    write_attribute :fingerprint, Fingerprint[tempfile]
  end

  def valid_solution?(tempfile)
    fingerprint == Fingerprint[tempfile]
  end
end
```

```ruby
test "must be able to create a fingerprint for a file" do
  tempfile = Tempfile.new("puzzle_sample")
  tempfile << "Sample Text"
  tempfile.rewind

  expected_fingerprint = Fingerprint[tempfile]
  tempfile.rewind

  puzzle = Puzzle.new(:file => tempfile)

  assert_equal expected_fingerprint, puzzle.fingerprint
end
```

The end result would be that the algorithm for how I was generating digital
fingerprints for the solutions could change, and I would not need to update my
tests, as long as the names of everything stayed the same.

In this case, arguably just fully applying the DRY principle would lead us to
the same place, but the concept of connascence lets us think about the
consequences of DRY in a less abstract way. Like all the other types of
connascence, there is a lot more we could talk about here, but in the interest
of time, we'll skip the details for now.

### Reflections

While we dug deep into some heavy theory in [last week's SOLID
article](http://practicingruby.com/articles/52), I tried to keep the connascence
examples simple, practical, and common. But that is not to say that connascence
isn't every bit as deep a concept as SOLID, and your investigations should not
stop at the examples I've shown here.

In the two articles to follow this one, we'll be looking at particular patterns
and techniques that can help you design better code. Now that you're armed with
both the SOLID principles and the metrics of connascence, you have a solid basis
for thinking about these problems in more specific contexts. I encourage you to
re-read these first two articles as you continue on with this series, and get
back to me with any questions you might have.
  
> **NOTE:** This article has also been published on the Ruby Best Practices blog. 
There [may be additional commentary](http://blog.rubybestpractices.com/posts/gregory/056-issue-24-connascence.html#disqus_thread) 
over there worth taking a look at.
