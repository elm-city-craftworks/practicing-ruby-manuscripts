> Note: This article series on modules is also available as a [PDF download]. The
> PDF version has been revised and is more up-to-date than what you see here.

[PDF download]:https://github.com/elm-city-craftworks/pr-monthly/blob/gh-pages/b5e5a89847701c4aa7c170cf/sept-2012-modules.pdf?raw=true

Today we're going to wrap up this series on modules by looking at how mixins can be useful for implementing custom behavior on individual objects. In particular, we'll be looking at how modules can be used both as a replacement for monkey patching, as well as for constructing systems that can be extended without the need for monkey patching. While neither of these techniques are going to be something you'll use every day, they really come in handy when you run into a situation that calls for them.

### Modules instead of Monkey Patches

Back in the bad old days before Prawn, I was working on a reporting framework called Ruby Reports (Ruport), which generated PDF reports via `PDF::Writer`. At the time, `PDF::Writer` was quite buggy, and essentially abandoned, but was the only game in town when it came to PDF generation.

One of the bugs was something fairly critical: Memory consumption for outputting simple PDF tables would balloon like crazy, causing a document with more than a few pages to take anywhere from several minutes to several *hours* to run.

The original author of the library had a patch laying around that inserted a hook which did some caching that greatly reduced the memory consumption, but he had not tested it extensively and did not want to want to cut a release. I had talked to him about possibly monkey patching `PDF::Document` in Ruport's code to add this patch, but together, we came up with a better solution: wrap the patch in a module.

```ruby
module PDFWriterMemoryPatch
  unless self.class.instance_methods.include?("_post_transaction_rewind")
    def _post_transaction_rewind
      @objects.each { |e| e.instance_variable_set(:@parent,self) }
    end
  end
end
```

In Ruport's PDF formatter code, we did something like the following to apply our patch:

```ruby
@document = PDF::Document.new
@document.extend(Ruport::PDFWriterMemoryPatch)
```

Throughout our application, whenever someone interacted with a `PDF::Document` instance we created, they had a patched instance that fixed the memory leak. This meant from the Ruport user's perspective, the bug was fixed. So what makes this different from monkey patching?

Because we were only manipulating the individual objects that we created in our library, we were not making a global change that might surprise people. For example if someone was building an application that only implicitly loaded Ruport as a dependency, and they created a `PDF::Document` instance, our patch would not be loaded. This prevented us from causing unexpected behavior in any code that lived outside of Ruport itself.

While this approach didn't shield us from the risks that a future change to `PDF::Writer` could potentially break our patch in Ruport, it did prevent any risk of global consequences. Anyone who's ever spent a day scratching their head because of some sloppy monkey patch in a third party dependency will immediately be able to see the value of this sort of isolation.

The neat thing is that a similar approach can be used for core extensions as
well. Rather than re-opening Ruby core classes, you can imbue individual
instances with custom behavior, getting many of the benefits of monkey patching
without the disadvantages. For example, suppose you want to add the `sum)()` and
`average()` methods to Array. If we were monkey patching, we'd write something
like the following code:

```ruby
class Array
  def sum
    inject(0) { |s,e| s + e }
  end

  def average
    sum.to_f / length
  end
end

obj = [1,3,5,7]
obj.sum     #=> 16
obj.average #=> 4
```

The danger here of course is that you'd be globally stomping anyone else's definition of `sum()` and `average()`, which can lead to ugly conflicts. All these problems can be avoided with a minor modification.

```ruby
module ArrayMathHelpers
  def sum
    inject(0) { |s,e| s + e }
  end

  def average
    sum.to_f / length
  end
end

obj = [1,3,5,7]
obj.extend(ArrayMathHelpers)
obj.sum     #=> 16
obj.average #=> 4
```

By explicitly mixing in the `ArrayMathHelpers` module, we isolate our changes just to the objects we've created ourselves. With slight modification, this technique can also be used with objects passed into functions, typically by making a copy of the object before working on it.

Because modules mixed into an instance of an object are looked up before 
the methods defined by its class, 
you can actually use this technique for modifying existing behavior of an object as well. 
The example below demonstrates modifying `<<` on strings so that it allows appending 
arbitrary objects to a string through coercion.

```ruby
module LooseStringAppend
  def <<(value)
    super
  rescue TypeError
    super(value.to_s)
  end
end

a = "foo"
a.extend(LooseStringAppend)
a << :bar << :baz #=> "foobarbaz"
```

Of course this (like most core modifications), is a horrible idea. But speaking as a pure technique, this is far better than the alternative global monkey patch shown below:

```ruby
class String
  alias_method :old_append, :<<
  
  def <<(value)
    old_append(value)
  rescue TypeError
    old_append(value.to_s)
  end
end
```

When using per-object mixins as an alternative to monkey patching, what you gain is essentially two things: A first class seat in the lookup path allowing you to make use of `super()`, and isolation on a per-object behavior so that consumers of your code don't curse you for patching things in unexpected ways. While this approach isn't always available, it is definitely preferable whenever you can choose it over monkey patching.

In Ruby 2.0, we may end up with even better option for this sort of thing called refinements, which are also module based. But for now, if you must hack other people's objects, this approach is a civil way to do it.

We'll now take a look at how to produce libraries and applications that actively encourage extensions to be done this way.

### Modules as Extension Points

This last section is not so much about practical advice as it is about taking what we've learned so far and really stretching it as far as possible into new territories. In essence, what follows are my own experiments with ideas that I'm not fully sure are good, but find interesting enough to share with you.

In previous Practicing Ruby issues, I've shown some code from a command line client we've used for time tracking in my consulting work. The tool itself never quite matured far enough to be release ready, but I used it as a testing ground for new design ideas, so it is a good conversation starter at least.

Today, I want to show how we implemented commands for it. Essentially, I want to walk through what happens when someone types the following command into their console:

```ruby
$ turbine start
Timer started at Wed Dec 15 17:55:37 -0500 2010
```

Because we knew this tool would evolve over time, we wanted to make it as hackable as possible. To do this, we set up a system in which commands get installed into a hidden folder in each project, making it trivial to modify existing commands or add new ones. Here's a quick directory listing to show what that structure looks like:

```ruby
$ ls .turbine/commands/standard/
add.rb		project.rb	rewind.rb	status.rb commit.rb push.rb		
staged.rb	stop.rb drop.rb	reset.rb start.rb
```

As you might expect, start.rb defines the start command. Here's what its source
looks like:

```ruby
Turbine::Application.extension(:start_command) do
  def start
    timer = Turbine::Timer.new
    if timer.running?
      prompt.say "Timer already started, please stop or rewind first"
    else
      timer.write_timestamp
      prompt.say "Timer started at #{Time.now}"
    end
  end
end
```

You'll notice that all our commands are direct mappings to method
calls, which are responsible for doing all the work. While I've simplified the
following definition to remove some domain specific callbacks and options 
parsing, the following example shows the basic harness which registers 
Turbine's commands:

```ruby
module Turbine
  class Application
    def self.extensions
      @extensions ||= {}
    end

    def self.extension(key, &block)
      extensions[key] = Module.new(&block)
    end

    def initialize
      self.class.extensions.each do |_, extension|
        extend(extension)
      end
    end
  
    def run(command)
      send(command)
    end
  end
end
```

From this, we see that `Turbine::Application` stores a Hash of anonymous modules
which are created on the fly whenever the `extension()` is called. The
interesting thing about this design is that the commands aren't applied globally
to `Turbine::Application`, but instead, are mixed in at the instance level. This
approach allows us to selectively disable features, or completely replace them 
with alternative implementations.

For example, consider a custom command that gets loaded after the standard commands, which is implemented like this:

```ruby
Turbine::Application.extension(:start_command) do
  def go
    puts "Let's go!"
  end
end
```

Because the module defining the `go()` method would replace the original module in the extensions hash, the original module ends up getting completely wiped out. In retrospect, for my particular use case, this approach seems to be like using a thermonuclear weapon where a slingshot would do, but you can't argue that this fails to take extensibility to whole new limits.

Eventually, when someone falls off the deep end in their study of modules, they ask 'is it possible to uninclude them?', and the short answer to that question is "No", promptly followed up with "Why would you want to do that?". But what we've shown here is a good approximation for unincluding a module, even if we haven't quite figured out the answer to the 'why' part yet.

But sometimes, we have to explore just for the fun of it, right? :)

### Reflections

I have had a blast writing to you all about modules and answering your questions as they come up. Unfortunately, the topic is even bigger than I thought, and there are at least two full articles I could write on the topic,which might actually be more practical and immediately relevant than the materials I've shared today. In particular, we didn't cover things like the `included()` and `extended()` hooks, which can be quite useful and are worth investigating on your own.

Moving forward, my goals for Practicing Ruby are to be able to hit a wide range of topics, so we'll probably move away from the fundamentals of Ruby's object system and go back to some more problem-solving oriented topics in the coming weeks. But if you like this kind of format, please let me know.

  
> **NOTE:** This article has also been published on the Ruby Best Practices blog. There [may be additional commentary](http://blog.rubybestpractices.com/posts/gregory/043-issue-11-uses-for-modules.html#disqus_thread) 
over there worth taking a look at.
