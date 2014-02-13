> **CREDIT:** Although this article is my own work, it is based on ideas I got from
> a very different but interesting [early draft][dna-draft] from Practicing Ruby reader
> [Mike Subelsky][subelsky]. I owe him a huge hat tip for suggesting that we cover
> this topic and for helping me get started with it.

The challenge of sensibly connecting a set of objects together to perform a
complex task is one that confounds programmers of all skill levels. In fact, 
it is hard to reason about the relationships between objects without getting 
trapped by analysis paralysis. With that in mind, it is no surprise that so
many of us struggle with this particular aspect of object-oriented programming. 

But like so many other problems we encounter in our work, this one can
be simplified greatly by introducing a common vocabulary and some rough
heuristics that make thinking and communicating about our code easier.
For reasoning about this particular design challenge, the 
"Object Peer Stereotypes" described in [Growing Object-Oriented Software, Guided
by Tests][GOOS] give us some very useful conceptual tools 
to work with.

In this article, we will explore the three stereotypical relationships 
between an object and its peers that were described in GOOS: 
dependencies, notifications, and adjustments. Taken together, these 
rough categorizations do a good job of identifying the kinds of
connections that exist between objects, which makes it easier
to develop a more nuanced view of how they communicate with each other.

The specific examples in this article are based on code from 
[Newman][newman] (my experimental email-based microframework), but the
general concepts that we'll discuss are relevant to all object-oriented 
software. If you keep your own projects in the back of your mind as you
read on, you'll easily find similarities between Newman's design 
challenges and your own.

## Dependencies

> Services that the object requires from its peers so it can perform its
> responsibilities. The object cannot function without these services. It should
> not be possible to create the object without them. (GOOS, pg. 52)

Whether they are internal or external, dependency relationships need to be
carefully managed in order to prevent brittleness. 
Alistair Cockburn's [ports and adapters][ports-and-adapters] pattern provides
one way of dealing with this problem: define abstract *ports* in the 
application's domain language that covers slices of functionality, and then build 
implementation-specific *adapters* with compatible interfaces. This approach allows dependencies 
to be reasoned about at a higher level of abstraction and makes it so that systems 
can be easily changed.

We applied this pattern (albeit without recognizing it by name) when thinking
through how Newman should handle its email dependency. We knew from the outset
that we'd need to support some sort of test mailer and that it should be a
drop-in replacement for its real mailer. We also anticipated that down the line
we might want to support delivery mechanisms other than the `mail` gem and
figured that some sort of adapter-based approach would be a good fit.

Constructing a port involves thinking through the various ways a 
subsystem will be used within your application and then 
mapping a protocol to those use cases. In Newman, we expected that
our email dependency would need to support the following requirements:

1) Read configuration data from a `Newman::Settings` object if necessary.

```ruby
mailer = AnyMailAdapter.new(settings)
```

2) Retrieve all messages from an inbox, deleting them from the server in the
process.

```ruby
mailer.messages.each do |message|
  do_something_exciting(message) 
end
```

3) Construct a complete message and deliver it immediately.

```ruby
mailer.deliver_message(:to      => "test@test.com",
                       :from    => "gregory@practicingruby.com",
                       :subject => "A special offer for you!!!",
                       :body    => "Send me your credit card number, plz!")
```

4) Construct a message incrementally and then deliver it later, if at all. 

```ruby
message = mailer.new_message(:to   => "test@test.com",
                             :from => "gregory@practicingruby.com")

if bank_account.balance < 1_000_000_000
  message.subject = "Can I interest you in some prescription painkillers?"
  message.body    = "Best prices anywhere on the internets!!!"
  messsage.deliver
end
```

Although you can make an educated guess about how to implement adapters
for this port based on the previous examples, there are many
unanswered questions lurking just beneath the surface. This is where
the difference between *interfaces* and *protocols* becomes important:

> An interface defines whether two things can fit together, a protocol 
defines whether two things can *work together* (GOOS, pg. 58)

If you revisit the code examples shown above, you'll notice that the interface
requirements for a Newman-compatible mail adapter are roughly as follows:

* The constructor accepts one argument (the settings object).
* The `messages` method returns an collection that responds to `each` and yields
an object for each message in the inbox.
* The `deliver_message` accepts one argument (a parameters hash).
* The `new_message` method accepts a parameters hash and returns
an object representing the message. At a minimum, the object allows certain fields
to be set (i.e., `subject` and `body`) and responds to a `deliver` method.

Building an object that satisfies these requirements is trivial, but there is
no guarantee that doing so will result in an adapter that conforms to the
*protocol* that Newman expects. Unfortunately, protocols are much harder
to reason about and define than interfaces are.

Like many Ruby libraries, Newman relies on loose [duck typing][duck typing] 
rather than a formal behavioral contract to determine whether one adapter can 
serve as a drop-in replacement for another. The `Newman::Mailer` object is used
by default, so it defines the canonical implementation that 
other adapters are expected to mimic at the functional level -- even if they 
handle things very differently under the hood. This implicit contract makes 
it possible for `Newman::TestMailer` to stand in for 
a `Newman::Mailer` object, even though it stores all incoming and 
outgoing messages in memory rather than relying on SMTP and IMAP. Because
the two objects respond to the same messages in similar ways, the systems
that depend on them are unaware of their differences in implementation -- they
are just two different adapters that both fit in the same port.

If you read through the source of the [Newman::Mailer][newman-mailer] 
and [Newman::TestMailer][newman-testmailer] objects, you will find that
several compromises have been made for the sake of convenience:

1. Arguments for the `new_message` and `deliver_message` methods on both 
adapters are directly delegated to the `Mail::Message` constructor, and the
return value of `messages` on both adapters is a collection 
of `Mail::Message` objects. This design implicitly ties the interface of those 
methods to the mail gem; it's what GOOS calls a *hidden dependency*.

2. The `Newman::TestMailer` object is a singleton object, but it
implements a fake constructor in order to maintain interface compatibility 
with `Newman::Mailer`. This is an example of how constraints 
from dependencies can spill over into client code.

3. Configuration data is completely ignored by `Newman::TestMailer`. Because
all of its operations are done in memory, it has no need for SMTP and IMAP
settings, but it needs to accept the settings object anyway for the 
sake of maintaining interface compatibility.

All of these warts stem from protocol issues. The first issue is due to
underspecification: Newman has a clear protocol for creating, retrieving, and
sending messages, but it does not clearly define what it expects the messages
themselves to look like. The coupling between the interface of `Newman::Mailer`
and that of `Mail::Message` makes it so that other adapters must also inherit
this hidden dependency. Because `Newman::TestMailer` also explicitly depends 
upon `Mail::Message`, this constraint does not complicate its implementation,
but it certainly does make it harder to build adapters that aren't dependent 
on the mail gem.

On the flip side, the second and third issues are a result of 
overspecification. We didn't want to make `Newman::TestMailer` a singleton, 
but because the underlying `Mail::TestMailer` is implemented that way,
we didn't have much of a choice. Our decision to implement a fake constructor
in order to maintain compatibility with `Newman::Mailer` is something I was
never happy with, but I also couldn't think of a better
alternative. I am somewhat less concerned about `Mailer::TestMailer` having to
accept a settings object that it doesn't actually use, but it does feel like one
extra hoop to jump through simply for the sake of consistency.

Despite these rough edges, Newman's way of handling its email dependency is a
good example of the [ports and adapters][ports-and-adapters] pattern in the 
wild. If anything, it serves as a reminder that the hard part of writing loosely
coupled code is not in the creation of duck-typed adapters, but in clearly
defining the protocol for our ports. This concept takes us beyond the idea of "coding to
an interface rather than an implementation" and is worth ruminating
over.

## Notifications

> Peers that need to be kept up to date with the object’s activity. The object
> will notify interested peers whenever it changes state or performs a
> significant action. Notifications are ‘fire and forget’; the object neither
> knows nor cares which peers are listening. (GOOS, pg. 52)

Because Ruby is a message-oriented programming language, it is easy to model
many kinds of object relationships as notifications. Doing so greatly reduces
the coupling between objects and helps establish a straight-line flow from a 
system's inputs to its outputs.

Notification-based modeling is especially useful when designing framework code,
because it is important for frameworks to know as little as possible about the
applications that are built on top of them. The general design of
the extremely popular [rack web server interface][rack] leverages these ideas to
great effect; it assumes that its applications implement a meaningful
`call` method, but otherwise remains blissfully ignorant of their behaviors.

We have designed Newman using a similar
strategy, and the general idea can be understood by tracing the execution of
the `Newman::Server#tick` method:

```ruby
module Newman
  class Server
    # NOTE: the mailer, apps, logger, and settings dependencies
    # are initialized when a Server instance is instantiated

    def tick
      mailer.messages.each do |request|
        response = mailer.new_message(:to   => request.from,
                                      :from => settings.service.default_sender)

        process_request(request, response) && response.deliver
      end

      # ... error handling code omitted
    end


    def process_request(request, response)
      apps.each do |app|
        app.call(:request  => request,
                 :response => response,
                 :settings => settings,
                 :logger   => logger)
      end

      return true

      # ... error handling code omitted
    end
  end
end
```

Did you figure out how it works? Let's walk through the process step by step to
confirm:

1. The `tick` method walks over each incoming message currently queued by the
`mailer` object (i.e., the `request`).

2. A `response` message is constructed and addressed to the sender of
the `request`.

3. The `process_request` method is called, which iterates over a
collection, executing the `call` method on each element and passing along
several dependencies that can be used to finish building a meaningful
`response` message.

4. Once `process_request` completes successfully, the response is delivered.

Because `Newman::Server` has a notification-based relationship with its
`apps` collection, it does not know or care about the structure of those
objects. In fact, the contract is so simple that a trivial `Proc` object 
can serve as a fully functioning Newman application:

```ruby
Greeter = ->(params) { |params| params[:response].subject = "Hello World!" }

server.apps = [Greeter]
server.tick
```

If we wanted to make things a bit more interesting, we could add request
and response logging into the mix, using Newman's built-in features:

```ruby
Greeter = ->(params) { |params| params[:response].subject = "Hello World!" }

server.apps = [Newman::RequestLogger, Greeter, Newman::ResponseLogger]
server.tick
```

These objects make use of a mixin that simplifies email logging, but as you can
see from the following code, they have no knowledge of the `Newman::Server`
object and rely entirely on the parameters being passed into their `#call`
method:

```ruby
module Newman
  class << (RequestLogger = Object.new)
    include EmailLogger

    def call(params)
      log_email(params[:logger], "REQUEST", params[:request]) 
    end
  end

  class << (ResponseLogger = Object.new)
    include EmailLogger

    def call(params)
      log_email(params[:logger], "RESPONSE", params[:response])
    end
  end
end
```

Taken together, these four objects combined form a cohesive workflow:

1. The server receives incoming emails and passes them on to its `apps` for
processing, along with a placeholder `response` object.

2. The request logger inspects the incoming email and records debugging 
information.

3. The greeter sets the subject of the outgoing response to "Hello World".

4. The response logger inspects the outgoing email and records debugging
information.

5. The server sends the response email.

The remarkable thing is not this semimundane process, but that the
objects involved know virtually nothing about their collaborators and are unaware of their position in the sequence of events. Context-independence
(*GOOS, pg. 54*) is a powerful thing, because it allows each object to be reasoned
about, tested, and developed in isolation.

The implications of notification-based modeling extend far beyond
context independence, but it wouldn't be easy to summarize them in 
a few short sentences. Fortunately, this topic has been covered 
extensively in other Practicing Ruby articles, particularly in 
[Issue 4.11][pr-4.11] and [Issue 5.2][pr-5.2]. Be sure to
read those articles if you haven't already; they are among the finest in our
collection.

## Adjustments

> Peers that adjust the object’s behavior to the wider needs of the system. This
includes policy objects that make decisions on the object’s behalf . . . and
component parts of the object if it’s a composite. (GOOS, pg. 52)

Adjustment relationships are hard to summarize, because they can exist in so 
many forms. But regardless of the form they take on, adjustments tend to be
used to bridge the gap between different levels of abstraction. Some are used
to raise the level of abstraction by wrapping a specific object in a more
generic interface, and others are designed to do the opposite. 

For an example of climbing down the ladder of abstraction, consider 
`Newman::EmailLogger`. It is implemented as a module in Newman for convenience,
but it could easily be reimagined as a stateless peer object of `RequestLogger`
and `ResponseLogger`. Such a redesign would yield something similar to the
following code:

```ruby
module Newman
  class << (EmailLogger = Object.new)
    def log_email(logger, prefix, email)
      logger.debug(prefix) { "\n#{email}" }
      logger.info(prefix) { email_summary(email) }
    end

    private

    def email_summary(email)
      { :from     => email.from,
        :to       => email.to,
        :bcc      => email.bcc,
        :subject  => email.subject,
        :reply_to => email.reply_to }
    end    
  end

  RequestLogger = ->(params) {  
    EmailLogger.log_email(params[:logger], "REQUEST", params[:request])
  }

  ResponseLogger = ->(params) {
    EmailLogger.log_email(params[:logger], "RESPONSE", params[:response])
  }
end
```

Though this is a subtle change, it lifts up and centralizes the concept of
"email logging" into a single object, rather than mixing helper methods into
various objects that need that functionality. This adjustment helps define the borders
between distinct concepts within the code and establishes `EmailLogger` as an
adjustment to the much more general `Logger` object it depends upon.

The philosophical distinction between these two objects is what matters here. 
A `Logger` has very abstract responsibilities: it must record arbitrary strings 
at various levels of severity and then format and output them to various 
streams. `EmailLogger`, on the other hand, is extremely concrete in its
responsibilities: it uses a `Logger` to report debugging information about
an email message. The details of how the actual logging happens are hidden from
`EmailLogger`'s clients, making it easier to treat as a black box.

Simple designs can also emerge from climbing the ladder of abstraction, 
that is, moving from a very specific context to a much more general one. For
example, it might not be a bad idea to introduce an object into Newman 
that encapsulates the concept of an email message but leaves the exact
delivery mechanism up to the individual adapters:

```ruby
# this code would be in an adapter or application code
message = Newman::Message.new { |params| Mail::Message.new(params).deliver }

# elsewhere, no knowledge of the dependency on the mail gem would be necessary:
message.to      = "test@test.com"
message.from    = "gregory@practicingruby.com"
message.subject = "You have won twenty bazillion dollars!"
message.body    = "Please send us a hair sample to confirm your ID"

message.deliver
```

This kind of object is trivial to implement because it is nothing more
than a value object with a simple callback mechanism bolted on top of it:

```ruby
module Newman 
  class Message
    def initialize(&delivery_callback)
      self.delivery_callback = delivery_callback
    end

    attr_accessor :to, :from, :subject, :body

    def deliver
      raise NotImplementedError unless delivery_callback

      delivery_callback.(:to      => to,      :from => from, 
                         :subject => subject, :body => body)
    end

    private

    attr_accessor :delivery_callback
  end
end
```

Despite its simplicity, this object provides a useful benefit: it explicitly
separates the protocol of message delivery from its implementation. If all
mail adapters for Newman were expected to return only `Newman::Message` objects, 
then any message-processing code within Newman (either in the server or in
application code) would have a well-defined interface to work against. Although
this requirement would make adapters slightly more cumbersome to write, it would
completely eliminate the hidden dependency issue discussed earlier.

Regardless of which direction they are pointed in, adjustment relationships are
very closely related to the concept of object composition in general. With that
in mind, the authors of GOOS have a useful rule to consider when designing 
composite objects:

> The API of a composite object should not be more complicated than that of any
> of its components. (GOOS, pg. 54)

Notice that in both the `Newman::EmailLogger` example and the `Newman::Message`
object, the result of composition is that a more complex system is being wrapped
by something with fewer methods and concepts to worry about. When applied
repeatedly, this kind of design causes software to become more simple as it
grows.

## Reflections

The benefit I have gained from being able to explicitly label various 
object relationships as dependencies, notifications, and adjustments is that
it forces me to think about my code in a more fine-grained way. Each
kind of object relationship comes with benefits and costs that are easier to
reason about when you recognize them for what they are.

As with most ideas from [Growing Object-Oriented Software, Guided by Tests][GOOS],
I have not yet had a chance to apply this particular set of heuristics
frequently enough to know the full extent of their usefulness. However, it never
hurts to have specific words to describe ideas that previously were hard for me
to express without relying heavily on intuition.

I would love to hear from you if you can think of ways to connect these ideas
back to your own projects or to the open source projects you've worked with. If
you have an interesting story to share, please leave a comment!

[GOOS]: http://www.growing-object-oriented-software.com/
[rack]: http://rack.github.com/
[pr-4.11]: https://practicingruby.com/articles/64
[pr-5.2]: https://practicingruby.com/articles/71
[ports-and-adapters]: http://alistair.cockburn.us/Hexagonal+architecture
[newman-mailer]: http://elm-city-craftworks.github.com/newman/lib/newman/mailer.html
[newman-testmailer]: http://elm-city-craftworks.github.com/newman/lib/newman/test_mailer.html
[duck typing]: http://en.wikipedia.org/wiki/Duck_typing
[newman]: https://github.com/elm-city-craftworks/newman
[subelsky]: http://www.subelsky.com/
[dna-draft]: http://www.subelsky.com/2012/11/ruby-dependencies-notifications-and.html
