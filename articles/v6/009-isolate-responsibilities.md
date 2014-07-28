> **NOTE:** This issue of Practicing Ruby was one of several content experiments 
that was run in Volume 6. It uses a cookbook format (e.g. problem -> solution -> discussion)
instead of the traditional long-form article format we use in most Practicing Ruby articles.

**Problem: Code for data munging projects can easily become brittle.**

Whenever you work on a project that involves a significant amount of [data
munging](http://en.wikipedia.org/wiki/Data_munging), you can expect to get some mud on your boots. Even if the individual
aggregation and transformation steps are simple, complexity arises from
messy process of assembling a useful data processing pipeline. With each
new change in requirements, this problem can easily be compounded in
brittle systems that have not been designed with malleability in mind.

As an example, imagine that you are implementing a tool that
delivers auto-generated email newsletters by aggregating and 
filtering links from Reddit. The following workflow provides
a rough outline of what that sort of program would need to
do in order to complete its task:

1. Map the raw JSON data from Reddit's API to an intermediate format that can be
used throughout the rest of the program.

3. Apply filters to ignore links that have already been included in a previous
newsletter, or fall below a minimum score threshold. 

4. Convert the curated list of links into a human readable format.

5. Send out the formatted list via email using GMail's SMTP servers.

Some will look at this set of steps and see a standalone script as the right
tool for the job: the individual steps are simple, and the time investment is
small enough that you could throw the entire script away and start again if you
end up facing significant changes in requirements.

Others will see this as a perfect opportunity to put together an elegant domain
model that supports a classic object-oriented design style. By encapsulating all
of these ideas in generalized abstractions, endless changes would be possible in
the future, thus justifying the upfront design cost.

Both of these perspectives have merit, but it would be unwise to set up a
false dichotomy between formal design and skipping the design process entirely. 
Interesting solutions to this problem also exist in the space between these two extremes,
and so we'll take a look at one of them now.

---

**Solution: Reduce the cost of rework by organizing your codebase into
isolated single-purpose components.**

Unlike the typical web application which has a wide range of end-points serving 
orthogonal concerns, the workflow for data munging projects often more closely 
resembles a flow-chart, with clearly defined beginning and end points. The
step-by-step nature of data munging projects makes them most naturally fit the 
procedural programming paradigm. This is a source of tension in Ruby,
because of its heavy object-oriented bias at the language level.

A reasonable compromise is to embrace "procedural programming with
objects". Rather than discussing this technique in the abstract,
we will instead explore what it looks like in practice by seeing
how it can be used to build the Reddit curation tool we
discussed earlier.

Let's start with the script that implements the core workflow of the program:

```ruby
require_relative "../lib/spyglass/actions/load_history_file"
require_relative "../lib/spyglass/actions/fetch_links"
require_relative "../lib/spyglass/actions/format_message"
require_relative "../lib/spyglass/actions/deliver_message"

basedir = File.dirname(__FILE__)

history   = Spyglass.load_history_file("#{basedir}/history.store")     #1
min_score = 20

selected_links = Spyglass.fetch_links("ruby").select do |link|         #2
  link.score >= min_score && history.new?(link)                        #3
end

history.update(selected_links)                                         #4

message = Spyglass.format_message(links: selected_links, 
                                  template: "#{basedir}/message.erb")  #5

Spyglass.deliver_message(subject: "Links for you!!!!!!",               #6
                         message: message)
```

This code looks a bit different than the typical Ruby snippet, because rather
than instantiating objects directly and then calling methods on them, it is
simply calling methods on the `Spyglass` module. It is obvious from the
`require_relative` calls that these features have been individually enabled,
which is also a non-standard way of doing thing.

If you set aside the quirks of this code for a moment, it should still be fairly
easy to read. Here's a rough English translation of what's going on:

1. A history log is being loaded from a file.
2. Links in the "ruby" sub-reddit are being fetched and filtered
3. Links are filtered out if they're below a score threshold or have been
selected in a previous run of the program.
4. The history log is updated with the newly selected links
5. The selected links are formatted into a human readable message
6. The message is delivered with the subject "Links for you!!!!!!"

Because the six steps above pretty much directly line up with the high-level
requirements of the project, it is safe to say that this code is sufficiently
expressive. But to properly evaluate the overall design, we'll need to dig into
the code that implements these features. Let's proceed by walking through the
features in the order that they are used.

First up is the `Spyglass.load_history_file` procedure:

```ruby
require_relative "../data/history"

module Spyglass
  def self.load_history_file(filename)
    Data::History.new(filename)
  end
end
```

This method is a trivial stub that creates an instance of the
`Spyglass::Data::History` class shown below:

```ruby
require "pstore"

module Spyglass
  module Data
    class History
      def initialize(filename)
        @store = PStore.new(filename)
      end

      def new?(link)
        @store.transaction { @store[link.url].nil? }
      end

      def update(links)
        @store.transaction do
          links.each { |link| @store[link.url] = true }
        end
      end
    end
  end
end
```

From this definition, we can infer that the job of the `History` 
object is to keep track of which URLs have been selected in 
previous runs of the program. It uses a `PStore` object as
its persistence method, but that is mostly an implementation detail.

With an understanding of how `Spyglass.load_history_file` works and 
what type of object it returns, we can now move on to investigating
the `Spyglass.fetch_links` procedure:

```ruby
require "json"
require "open-uri"

require_relative "../data/link"

module Spyglass
  def self.fetch_links(category)
    document = open("http://api.reddit.com/r/#{category}?limit=100").read

    JSON.parse(document)["data"]["children"].map do |e|
      e = e["data"]

      Data::Link.new(url: e["url"], score: e["score"], title: e["title"])
    end
  end
end
```

This method is responsible for making an HTTP request to the Reddit API to
capture a JSON document representing the raw data about links in a particular
subreddit. It then parses that document and transforms it into `Data::Link`
objects. A quick look at the class definition for `Data::Link` reveals that it
is a straightforward value object with no interesting business logic:

```ruby
module Spyglass
  module Data
    class Link
      def initialize(url: raise, score: raise, title: raise)
        @url   = url
        @score = score
        @title = title
      end

      attr_reader :url, :score, :title
    end
  end
end
```

As simple as it is, the `Data::Link` object is a very important part of this 
program, because every other feature that refers to links assumes that
they conform to this interface. In other words, we've set in stone here
that the data our program is interested in when it comes to links are
its `score`, `url`, and `title`. Any changes to this interface would
require widespread changes throughout our program.

Based on what you've seen so far, you should be able to understand exactly how
this program works up to step #4 in its main script. Only two steps remain:
formatting the list of curated links into a human readable message, and
delivering it to someone.

The formatting procedure (i.e. `Spyglass.format_message`) is extremely basic, 
as it is nothing more than a minimal wrapper around the `ERB` standard 
library:

```ruby
require "erb"

module Spyglass
  def self.format_message(links: raise, template: raise)
    ERB.new(File.read(template), nil, "-").result(binding)
  end
end
```

This code is somewhat generalized, allowing an arbitrary template to
present the list of links. In the case of this particular script, we use a
simple text-based template that looks like this:

```
Here are some links you might enjoy!
<% links.each do |link| -%>

  <%= link.title %>:
  <%= link.url   %>
<% end %>
Have fun!
-greg
```

When evaluated, this template spits out plain-text output that looks similar to
what you see below:

```
Here are some links you might enjoy!

  _why updated his site:
  http://whytheluckystiff.net

  Teabag: A Javascript test runner built on top of Rails:
  https://github.com/modeset/teabag

  Ruby 2.0 Works Hard So You Can Be Lazy:
  http://patshaughnessy.net/2013/4/3/ruby-2-0-works-hard-so-you-can-be-lazy

Have fun!
-greg
```

From here, all that remains is to fire this message out via email, which is
handled by the `Spyglass.deliver_message` procedure:

```ruby
require "mail"

Mail.defaults do
  delivery_method :smtp, {
    :address              => 'smtp.gmail.com',
    :port                 => '587',
    :user_name            => ENV["GMAIL_USER"],
    :password             => ENV["GMAIL_PASSWORD"],
    :authentication       => :plain,
    :enable_starttls_auto => true
  }
end

module Spyglass  
  def self.deliver_message(message: raise, subject: raise)
    mail = Mail.new

    mail.from = ENV["GMAIL_USER"]
    mail.to   = ENV["SPYGLASS_RECIPIENT"]

    mail.subject = subject
    mail.body    = message 

    mail.deliver!
  end
end
```

This is not as easy to read as many of the previous procedures, because it
involves some configuration code. However, on closer investigation we can easily
see that this is a thin wrapper around the `Mail` gem, and that it uses three
environment variables for its settings: `GMAIL_USER`, `GMAIL_PASSWORD`, and
`SPYGLASS_RECIPIENT`. This means that the main script for this program needs to
have these values set before it can be run, as in the example below:

```console
$ GMAIL_USER="test@gmail.com" GMAIL_PASSWORD="password" \
  SPYGLASS_RECIPIENT="test@test.com" ruby examples/reddit.rb
```

If you have a GMail account, you can actually give this a try by cloning
the [practicing-ruby-examples
repository](https://github.com/elm-city-craftworks/practicing-ruby-examples/tree/master/v6/009)
and running something similar to the line shown above in the *v6/009* folder.
But as long as you understand the general idea behind this program, don't worry
if you can't test it for yourself right now.

Assuming that you have been able to understand this walk-through, you may already
have some sense of why this solution is a reasonable middle ground between ad
hoc scripting and formal object-oriented design. However, we should discuss the
benefits and costs in more detail before we wrap things up here.

---

**Discussion**

The primary difference between object-oriented programming and "procedural
programming with objects" is that the former binds certain behaviors to
encapsulated data, and the latter decouples its data from its behavior.

Object-oriented design is best suited for problems where most of the interesting
details exist in the messages that are passed between objects. In other words,
when you have a complex set of interactions between a network of communicating
objects, it makes good sense to tightly bind together state and behavior.
However, this comes at the cost of indirection, and so it becomes hard to keep a
mental model in your mind of what the call graph looks like for even a single
request.

By contrast, data munging projects are procedural in nature, and so you have a
good sense of what needs to happen at each step in the process. The final
program represents a chain of transformations and filters on relatively simple
data structures, with some side effects thrown in along the way. Because each
step tends to be a very concrete action, the abstraction benefit that objects
can offer is negated by the fact that so much is subject to change in the whole
system.

If you go back and read through the [codebase we discussed in this
article](https://github.com/elm-city-craftworks/practicing-ruby-examples/tree/master/v6/009),
you will find that the data objects are trivially understandable, and the
actions are context-independent. Although they are not pure functions, each of
the actions can be fully understood in terms of its inputs, outputs, and
external dependencies. This makes it possible to make changes to the internals
without thinking about their impact on the overall program, as long as the
return values do not change.

Another interesting benefit of "procedural programming with objects" is that a
lack of internal behavioral dependencies makes it so that you can easily change
the signature of a single action without requiring a cascade of changes
throughout the system. The main script might need to be updated, but such
revisions would be trivial.

However, it is important to remember that all of these benefits come from the
fact that data munging projects occupy a special domain where certain benefits
of object-oriented programming are not especially important. You may want to
consider adopting a traditional object-oriented design if any of the following
conditions apply:

* You have actions that need to store data in instance variables, rather than
simply returning value objects or using repository objects like the `History`
object in this example.

* You have actions that need to call other actions in order to get their job
done, rather than relying on simple data objects that rarely change.

* You have actions that need to operate as a multi-step state machine, rather
than a single-purpose procedure that you can fire and forget.

All of the above are symptoms that the benefits of object-oriented design will
outweigh its costs, and Ruby *is* a deeply object-oriented language, so you
won't lose out by heading in that direction. However, if you are stuck in the
place between a throwaway script and a full object-oriented program, the example
shown in this article might help you find a nice compromise.
