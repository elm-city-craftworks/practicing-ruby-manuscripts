SOLID is a collection of five object oriented design principles that go nicely
together. Here's a super brief summary pulled from [the wikipedia
page](http://en.wikipedia.org/wiki/SOLID) on the topic:

* Single responsibility principle: an object should have only a single
responsibility.

* Open/closed principle: an object should be open for extension, but closed for
modification.

* Liskov substitution principle: objects in a program should be replaceable with
instances of their subtypes without altering the correctness of that program.

* Interface segregation principle: many client specific interfaces are better
than one general purpose interface.

* Dependency inversion principle: depend upon abstractions, do not depend upon
concretions

The term SOLID was coined by Uncle Bob Martin to group together these important
concepts. I had heard of each of the design principles SOLID covers over the
years, but didn't really think much of them until I attended a great talk by
Sandi Metz at GoRuCo 2009. Fortunately, Confreaks recorded [Sandi's
talk](http://confreaks.net/videos/240-goruco2009-solid-object-oriented-design)
so I won't need to try to summarize it here.

I'd strongly recommend watching that video before moving on, because it will go
through SOLID in a lot more detail than what I plan to do in this article. You
might also watch [another video on the same topic by Jim
Weirich](http://confreaks.net/videos/185-rubyconf2009-solid-ruby), which like
pretty much any other talk Jim has done, is likely to blow your mind.

Rather than giving a tutorial on these principles, I'm going to trust you to
either read up on them or watch the videos I've linked to. This way, we can
focus on what I think is a much more interesting problem: How to apply these
ideas to real code.

### Single responsibility principle

The idea that an object should have only a single responsibility shouldn't come
as a surprise. This concept is one of the selling points for object oriented
languages that sets them apart from the more procedural systems that preceded
them. The hard part about putting this idea into practice is figuring out just
how wide to cast the 'single responsibility' net.

In my experience, most objects are born with just one goal in mind, and so
adhere to this principle at least superficially in the early stages of
greenfield projects. It's later when systems get more complex that our objects
lose their identity. To demonstrate this phenomena, we can look at the life
cycle of the Document object from my PDF generation library Prawn.

Back in early 2008, when the project was just beginning, my idea was that the
job of the Document class would be to wrap the low level concept of a Document
at the PDF layer, with a few extra convenience functions at the high level. For
a sketch of what that looked like at the time, we can take a look at the
object's public methods.

```
Directly implemented on Prawn::Document
  start_new_page, page_count, page_size, page_layout, render, render_file

Mixed in via Prawn::Document::Graphics
  line_width=, line, line_to, curve_to, curve, circle_at, ellipse_at,
  polygon, rectangle, stroke, fill, fill_color, stroke_color

Mixed in via Prawn::Document::PageGeometry
  page_dimensions
```

This is so early in Prawn's history that it didn't even have text support yet.
While the API wasn't perfectly well factored at this point in time, the fact
that almost all the above methods directly produced PDF instructions or
manipulated low level structures made me feel that it was a reasonably cohesive
set of features.

Fast forward by a year, and we end up with feature explosion on `Document`.
Here's what shipped in Prawn 0.4.1:

```
Directly implemented on Prawn::Document
  start_new_page, page_count, cursor, render, render_file, bounds,
  bounds=, move_up, move_down, pad_top, pad_bottom, pad, mask,
  compression_enabled?, y, margin_box, margins, page_size,
  page_layout, font_size

Included via Prawn::Document::Text
  text, text_options, height_of (via Prawn::Document::Text::Wrapping),
  naive_wrap (via Prawn::Document::Text::Wrapping)

Included via Prawn::Document::PageGeometry
  page_dimensions

Included via Prawn::Document::Internals
  ref, add_content, proc_set, page_resources, page_fonts, 
  page_xobjects, names

Included via Prawn::Document::Annotations
  annotate, text_annotation, link_annotation

Included via Prawn::Document::Destinations
  dests, add_dest, dest_xyz, dest_fit,  dest_fit_horizontally, 
  dest_fit_vertically, dest_fit_rect, dest_fit_bounds,
  dest_fit_bounds_horizontally, dest_fit_bounds_vertically

Included via Prawn::Graphics
  move_to, line_to, curve_to, rectangle, line_width=, line_width,
  line, horizontal_line, horizontal_rule, vertical_line, curve,
  circle_at, ellipse_at, polygon, stroke, stroke_bounds, fill,
  fill_and_stroke, fill_color (via Prawn::Document::Color), 
  stroke_color (via Prawn::Document::Color)

Included via Prawn::Images
  image
```

The above list of methods is almost embarrassingly scattershot, but it was due
to an oversight. The mistake I made was thinking that splitting different
aspects of functionality into modules was a valid way of respecting the single
responsibility principle. But this is deeply flawed thinking, because the end
result of pulling in roughly 50 methods into a single object by mixing in 8
modules results in a single object, `Prawn::Document` having 60+ public methods
all sharing the same state and namespace. Any illusion of a physical separation
of concerns is all smoke and mirrors here.

Once an object gets this fat, thinking about the cohesiveness of the interface
is the most minor detail to be worried about. I've focused on the 60 public
methods here, but if we count private methods, they would easily exceed 100.
Sometimes folks think that private methods in mixins don't actually get mixed
into the base object, but that's an incorrect assumption, making this problem
much, much worse.

Having close to two hundred methods living in one space causes you to run into
really basic, fundamental problems such as namespace clashes on method names and
variables. It also makes data corruption downright easy, because it's hard to
keep track of how a couple hundred methods manipulate a common dataset. Once you
reach this point, you're back in procedural coding land where all manners of bad
things can happen.

Now that I've sufficiently kicked my own ass, I can tell you the solution to
this problem is simple, if not easy to refactor towards once you've already made
the mess: you just introduce more objects. To do so, we need to identify the
different concerns and group them together, putting abstraction barriers between
their implementations and the behaviors they provide.

An easy realization to make is that over time, Prawn's `Document` became two
different things at the conceptual level. When we see methods like
`page_xobjects`, `ref`, and `proc_set`, we know that there are some low level
tools in use here. But what about methods like move_up, move_down, text, image,
and others like them? These are clearly meant for something that resembles a
domain specific language, and Prawn does look gorgeous at the high level, just
see the simple example below to see what I mean.

```ruby
Prawn::Document.generate('hello.pdf') do 
  text "Hello Prawn!"
end 
```

With 20/20 hindset, the solution to this problem is obvious: Produce a whole
layer of low level tooling that closely follows the PDF spec, creating objects
for managing things like a PDF-level page, the rendering of raw PDF strings,
etc. Make as many objects as necessary to do that, and then maybe provide a
facade that makes interacting with them a bit easier.

Then, for the higher level features, do the same thing. Have an object who's job
is to provide nice looking methods that rely on Prawn's lower level
objects to do the dirty work. Dedicate whole objects or even clusters of objects
to text, images, graphics, and any other cluster of functionality that
previously was mixed into Document directly. The objects might require a bit
more wiring, but the facade can hide that by doing things like the pseudo-code
below.

```ruby
def text(contents, options={})
  text_element = Prawn::TextElement.new(contents, options)
  text_element.render_on(current_page)
end
```

Naming the benefits of this over the previous design would take a long time, but
we've at least cut out those pesky namespace and data corruption concerns
while providing a cohesive API.

While I don't think that the scale of our design problem in Prawn is comparable
to what most Ruby hackers are likely to experience in their day to day work, it
does show just how bad things can get when you start dealing with very complex
systems. Prawn has improved a lot since its 0.4.1 release, but undoing the
damage that was done by neglecting this for so long has been a slow and painful
process for us.

The real lesson here is that you can't respect SRP without real abstraction
barriers. SRP is about more than just creating a cohesive API, you actually need
to create a physical separation of concerns at the implementation level of your
system.

Since it's very likely that you're experiencing this sort of issue on a smaller
scale in the projects you're working on, keeping the story about what happened
to me in Prawn in mind may help you learn from my mistakes instead of your own.

### Open/closed principle

The open/closed principle tells us that an object should be open for extension,
but closed for modification. This can mean a lot of different things, but the
basic idea is that when you introduce a new behavior to an existing system,
rather than modifying old objects you should create new objects which inherit
from or delegate to the target object you wish to extend. The theoretical payoff
is that taking this approach improves the stability of your application by
preventing existing objects from changing frequently, which also makes
dependency chains a bit less fragile because there are less moving parts to
worry about.

Personally, I feel that treating this principle as an absolute law would lead to
the creation of a lot of unnecessary wrapper objects that could make your
application harder to understand and maintain, so much that it might outweigh
the stability benefits you'd gain. But that doesn't mean these ideas don't have
their value, in fact, they provide an excellent alternative to extensive
monkeypatching of third party code.

To illustrate this, I'd like to talk about 
[i18n_alchemy](https://github.com/carlosantoniodasilva/i18n_alchemy), a project
by Carlos Antonio da Silva that was built as a student project for his Mendicant
University core course. The goal of this project was to make it easy to add
localizations for numeric, time, and date values in ActiveRecord.

Early on in the course, Carlos came to me with an implementation that
more-or-less followed the standard operating procedure for developing Rails
plugins. While Carlos shouldn't be faulted for following community trends here,
the weapon of choice was a shotgun blast into an `ActiveRecord::Base` object's
namespace, via a mixin which could be used on a per-model level. By including
this module, you would end up with behavior that looked a bit like this:

```ruby
some_model = SomeModel.where(something)
some_model.a_number     #=> <a localized value>
some_model.a_number_raw #=> <the original numeric value>
```

Now, there are pros and cons to this approach, but I felt pretty sure that we
could do better, and through conversations with Carlos, we settled on a much
better design that didn't make such far reaching changes to the model objects.
Before I explain how it works, I'd like to show an example of how i18n_alchemy
works now:

```ruby  
some_model = SomeModel.where(something)
some_model.a_number     #=> <the original numeric value>

localized_model = some_model.localized
localized_model.a_number #=> <a localized value>
```

In this new implementation, you do have to explicitly ask for a localized
object, but that small change gains us a lot. The module that gives us
`SomeModel#localized` only introduces that one method, rather than a hook that
gets run for every `ActiveRecord::Base` method. That means that
ordinary calls to models extended by i18n_alchemy still work as they always did.

Our localized model act differently, but it's actually not an instance of
SomeModel at all. Instead, it is a simple proxy object that defines special
accessors for the methods that i18n_localized, delegating everything else to the
target model instance.

This makes it possible for the consumer to choose when it'd be best to work with
the localized object, and when it'd be best to work with the model directly.
Unlike the first implementation which breaks the ordinary expected
behavior of an ActiveRecord model, this approach creates a new entity which can
have new behaviors while reusing old functionality.

We were both pretty proud of the results here, because it gives some of the
convenient feel of mixing in some new functionality into an existing Ruby object
without the many downsides. This of course is only a single example of how you
can use OCP in your own code, but I think it's a particularly good one.

### Liskov substitution principle

The idea behind Liskov substitution is that functions that are designed operate
on a given type of object should work without modification when they operate on
objects that belong to a subtype of the original type. In many object oriented
languages, the type of an object is closely tied to its class, and so in those
languages, this principle mostly describes a rule about a relationship between a
subclass and a superclass. In Ruby, this concept is a bit more fluid, and
probably requires a bit more explanation up front.

When we talk about the type of an object in Ruby, we're concerned with
what messages that object responds to rather than what class that object is an
instance of. This seems like a subtle difference, but it has a profound
impact on how we think about thing. In Ruby, type checking can range from very
strict to none at all, as shown by the examples below.

```ruby
  ## Different ways of type checking, from most to least coarse ##

  # verify the class of an object matches a specific class
  object.class == Array

  # verify object's class descends from a specific class
  object.kind_of?(Array)   

  # verify a specific module is mixed into this object
  object.kind_of?(Enumerable)
  
  # verify object claims to understand the specified message
  object.respond_to?(:sort)   

  # don't verify, trust object to either behave or raise an error
  object.sort                 
```

Regardless of the level of granularity of the definition, objects that are meant
to be treated as subtypes of a base type should not break the contracts of the
base type. This is a very hard standard to live up to when dealing with ordinary
class inheritance or module mixins, since you basically need to know the
behavior specifications for everything in the ancestry chain, and so the rule of
thumb is basically not to inherit from anything or mix in a module unless you're
fairly certain that the behavior you're implementing will not interfere with the
internal operations of your ancestors.

To demonstrate a bit of a weird LSP issue, let's think about what happens when
you subclass an `ActiveRecord::Base` object. Technically speaking, if we give
ourselves a pass for breaking signature of methods provided by Object, we'd
still need to keep track of all the behaviors `ActiveRecord::Base` provides, and
take care not to violate them. Here's a brief list of method names, but keep in
mind we'd also need to match signatures and return values.

```ruby
>> ActiveRecord::Base.instance_methods(false).sort
=> ["==", "[]", "[]=", "attribute_for_inspect", "attribute_names",
"attribute_present?", "attribute_types_cached_by_default", "attributes",
"attributes=", "attributes_before_type_cast", "becomes", "cache_key",
"clone", "colorize_logging", "column_for_attribute", "configurations",
"connection", "connection_handler", "decrement", "decrement!",
"default_scoping", "default_timezone", "delete", "destroy",
"destroy_without_callbacks", "destroy_without_transactions",
"destroyed?", "eql?", "freeze", "frozen?", "has_attribute?", "hash",
"id", "id=", "id_before_type_cast", "include_root_in_json", "increment",
"increment!", "inspect", "lock_optimistically", "logger",
"nested_attributes_options", "new_record?", "partial_updates",
"partial_updates?", "pluralize_table_names", "primary_key_prefix_type",
"quoted_id", "readonly!", "readonly?", "record_timestamps", "reload",
"reload_without_autosave_associations", "reload_without_dirty", "save",
"save!", "save_without_dirty", "save_without_dirty!",
"save_without_transactions", "save_without_transactions!",
"save_without_validation", "save_without_validation!", "schema_format",
"skip_time_zone_conversion_for_attributes", "store_full_sti_class",
"store_full_sti_class?", "table_name_prefix", "table_name_suffix",
"time_zone_aware_attributes", "timestamped_migrations", "to_param",
"toggle", "toggle!", "update_attribute", "update_attributes",
"update_attributes!", "valid?", "valid_without_callbacks?",
"write_attribute", "write_attribute_without_dirty"]
```

Hopefully your impression after reading this list is that LSP is basically
impossible to be a purist about, but let's try to come up with a plausible
violation that isn't some obscure edge case. For example, what happens if we're
building a database model for describing a linux system configuration, which has
a field called logger in it? You can certainly at least get away with the
migration for it without Rails complaining, using something like the code shown
below.

```ruby
class CreateLinuxConfigs < ActiveRecord::Migration
  def self.up
    create_table :linux_configs do |t|
      t.text :logger
      t.timestamps
    end
  end

  def self.down
    drop_table :linux_configs
  end
end
```

The standard behavior of `ActiveRecord`'s models is to provide dynamic accessors
to a record's database fields, which means we should expect the following
behavior:

```ruby
config        = LinuxConfig.new
config.logger = "syslog-ng"
config.logger #=> "syslog-ng"
```

But because `ActiveRecord::Base` also implements a method called `logger`, and the
dynamic attribute lookup is just a method_missing hack, we end up with a
different behavior:

```ruby
config        = LinuxConfig.new
config.logger = "syslog-ng" 
config.logger #=> #<ActiveSupport::BufferedLogger:0x00000000b6de38 
              #     @level=0, @buffer={}, @auto_flushing=1, 
              #     @guard=#<Mutex:0x00000000b6dde8>,
              #     @log=#<File:/home/x/demo/log/development.log>>
```

If you've been following closely, you probably saw this coming from a mile away,
even if you couldn't predict the exact behavior. It's worth mentioning that even
Rails knows that this sort of setup will lead to bad things, but their checks
which raise an error when they spot this LSP violation apparently aren't
comprehensive. But to be fair, if we try to set this at the time our record was
initialized, or if we try to use write_attribute, we get a pretty decent error
message.

```ruby
>> config = LinuxConfig.new(:logger => "syslog-ng")
ActiveRecord::DangerousAttributeError: logger is defined by ActiveRecord
```

```ruby
>> config = LinuxConfig.new
=> #<LinuxConfig id: nil, logger: nil, created_at: nil, updated_at: nil>
>> config.write_attribute(:logger, "syslog-ng")
ActiveRecord::DangerousAttributeError: logger is defined by ActiveRecord
```

This sort of proactive error checking is actually more than we should expect
from most parent classes, `ActiveRecord::Base` just takes special consideration
because it is so widely used. You can't expect every object you might subclass
to even try to catch these sorts of violations, and it's not a great idea to
introduce this sort of logic into your own base classes without carefully
considering the context. Of course, that doesn't mean that there aren't measures
you can take to avoid LSP violations in code that you design yourself.

I don't want to go into too much detail here, but there are two techniques I
like to use for mitigating LSP issues. The first one is object composition, and
the second is defining per-object behavior. Just as an experiment, I've thrown
together a rethinking of how `ActiveRecord` could handle dynamic accessors in a
slightly more robust way.

```ruby
require "delegate"

module DynamicFinderProxy

  extend self

  def build_proxy(record)
    proxy = SimpleDelegator.new(record)
    record.attribute_names.each do |a|
      proxy.singleton_class.instance_eval do
        define_method(a) { read_attribute(a) }
        define_method("#{a}=") { |v| write_attribute(a,v) }
      end
    end

    proxy
  end

end

class FakeActiveRecord

  class << self
    def new
      obj = allocate
      obj.send(:initialize)
      DynamicFinderProxy.build_proxy(obj)
    end

    def column_names(*names)
      @column_names = names unless names.empty?
      @column_names
    end
  end

  def attribute_names
    self.class.column_names
  end

  def read_attribute(a)
    logger.puts("Reading #{a}")
    instance_variable_get("@#{a}")
  end

  def write_attribute(a,v)
    logger.puts("Writing #{a}")
    instance_variable_set("@#{a}",v)
  end

  def logger
    STDOUT
  end
end

class LinuxConfig < FakeActiveRecord
  column_names "logger", "crontab"
end

record = LinuxConfig.new
record.logger = "syslog-ng"
p record.logger
```

Now, I'll admit that there is some deep voodoo in this code, but it at least
indicates to me that we should be thinking differently about our options in
Ruby. We have more than just vanilla inheritance to play with, and even ordinary
mixins have their limitations, so maybe we need a whole new set of design
principles that take Ruby's deeply dynamic nature into account? Or perhaps I've
just passed the midway point in a very long article and have decided to go off
on a little tangent to keep myself entertained. I'll let you be the judge.

### Interface segregation principle

I've seen a couple different interpretations of the interface segregation
principle, with the most narrow ones almost directly outlining the use case for
Java-style interfaces, which is to prevent code from specifying that an object
must be a specific type when all that is required is a certain set of methods to
have a meaningful implementation.

Ruby offers a lot of flexibility and its dynamic typing makes a lot of interface
segregation principle violations just go away on their own. That having been
said, we still see a lot of `is_a?()` and `respond_to?()` checks which are both
a form of LSP violation.

To protect against those violations, the best bet is to embrace duck typing as
much as possible. Since this article is already super long and we've already
covered duck typing extensively in issues
[#14](http://practicingruby.com/articles/43) and
[#15](http://practicingruby.com/articles/44) of Practicing Ruby, It would be
sufficient to simply re-read those articles if you need a refresher and then
promptly move on to the next principle. But in case you want to dig deeper, here
are a couple more articles related to this topic that you should definitely read
if you haven't seen them before. All three are about how to get around
explicitly naming classes in case statements, which is a form of LSP violation.

* [Ruby case statements and kind_of?(Sandi Metz)](http://sandimetz.com/2009/06/ruby-case-statements-and-kindof.html)

* [The Double Dispatch Dance (Aaron Patterson)](http://blog.rubybestpractices.com/posts/aaronp/001_double_dispatch_dance.html)

* [The Decorator Delegator Disco (Gregory Brown)](http://blog.rubybestpractices.com/posts/gregory/008-decorator-delegator-disco.html)

That should add an extra hour or so of homework for you. This is getting a bit
crazy though, so let's hit that last principle and call it a day.

### Dependency inversion principle

You probably already know about the values of dependency inversion (aka
dependency injection) if you've been working in Ruby for a while now. You also
probably know that unlike some other languages, there really isn't a need for DI
frameworks because it implements all the necessary tools for good DI at the
language level. But in case you didn't get the memo, I'll go through a quick
example of how dependency inversion can come in handy.

Suppose we have a simple object, like a `Roster`, which keeps track of a list of
people, and we have a `RosterPrinter` which creates formatted output from that
list. Then we might end up with some code similar to what is shown below.

```ruby
class Roster
  def initialize        
    @participants = []
  end

  def <<(new_participant)
    @participants << new_participant
  end

  def participant_names
    @participants.map { |e| e.full_name }
  end

  def to_s
    RosterPrinter.new(participant_names).to_s
  end
end

class RosterPrinter
  def initialize(participant_names)
    @participant_names = participant_names
  end

  def to_s
    "Participants:\n" +
    @participant_names.map { |e| "* #{e}" }.join("\n")
  end
end
```

The nice thing about this code is that it separates the presentation of a roster
from its data representation, bringing it in line with the single
responsibility principle. But the problem with it is that `Roster` and
`RosterPrinter` are needlessly coupled, which limits the value of
separating the objects in the first place. Modifying `Roster#to_s()` can
solve this problem.

```ruby
class Roster
  # other methods same as before

  def to_s(printer=RosterPrinter)
    printer.new(participant_names).to_s
  end
end

# usage
roster.to_s 
```

This new code is functionally equivalent to our previous example when called
with no arguments, but opens a whole host of new opportunities. For example, we
can trivially swap in any printer object we'd like now.

```ruby  
class HTMLRosterPrinter
  def initialize(participant_names)
    @participant_names = participant_names
  end

  def to_s
    "<h3>Participants</h3><ul>"+
    @participant_names.map { |e| "<li>#{e}</li>" } +
    "</ul>
  end
end

# usage
roster.to_s(HTMLRosterPrinter)
```

By injecting the printer object into `Roster`, we avoid resorting to
something as uncouth as creating a `Roster` subclass for the sole purpose of
wiring up the `HTMLRosterPrinter`.

Of course, the most common place that talk about dependency inversion comes up
is when folks are thinking about automated testing. While Ruby makes it possible
to mock out calls to pretty much any object, it's a whole lot cleaner to pass in
raw mock objects than it is to set expectations on real objects.

Dependency inversion can really come in handy, but it's important to provide
sensible defaults so that you don't end up forcing consumers of your API to do a
lot of tedious wiring. The trick is to make it so you can swap out
implementations easily, it's not as important for your code to have no opinion
about which implementation it should use. Folks sometimes forget this and as a
result their code gets quite annoying to work with. However, Ruby makes it easy
to provide defaults, so there is no real reason why this issue can't be averted.

### Reflections

This article is much longer than I expected it would be, but I feel like I've
just scratched the surface. An interesting thing about the SOLID principles is
that they all sort of play into each other, so you tend to get the most out of
them by looking at all five concepts at once rather than each one in isolation.

One thing I want to emphasize is that when I make use of SOLID or any other set
of design principles, I use them as a metric rather than a set of
constructive rules. I don't typically set out designing a system with all of
these different guidelines in mind, as that would give me a claustrophobic
feeling. However, when the time comes to sanity check a new design or make
incremental improvements to an old one during a refactoring session, SOLID
provides a good checklist for pinpointing areas of my code that might deserve
some rethinking.

Sometimes you break these rules by accident, and that's okay. Sometimes you
break them because you are making a conscious trade to avoid some other bad
thing from happening, and that's okay too. As long as you're regularly checking
your assumptions about things and actually caring about the overall design of
your system, you shouldn't feel guilty for not following these guidelines
perfectly. In fact, it is more dangerous to blindly follow design principles to
the letter than it is to completely ignore them.

We have much, much more design discussion to come, so hopefully you enjoyed this
article. :)

> **NOTE:** This article has also been published on the Ruby Best Practices blog. There 
[may be additional commentary](http://blog.rubybestpractices.com/posts/gregory/055-issue-23-solid-design.html#disqus_thread) 
over there worth taking a look at.

