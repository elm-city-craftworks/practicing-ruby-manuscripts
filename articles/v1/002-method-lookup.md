In [Issue #1](http://practicingruby.com/articles/29) we discussed Ruby's lookup
path and proved by example that class inheritance is only a small part of the 
picture. To recap, Ruby methods are looked up in the following order:

1. Methods defined in the object's singleton class (i.e. the object itself)
1. Modules mixed into the singleton class in reverse order of inclusion
1. Methods defined by the object's class
1. Modules included into the object's class in reverse order of inclusion
1. Methods defined by the object's superclass, i.e. inherited methods

The example we looked at in the previous issue just showed the mechanics of how the above process plays out, it didn't really hint at practical use cases. Today, we'll look at a scenario for each of these options and discuss some of the up and downs that come along with them. Rather than presenting the examples in method lookup order, I'll try to start with the most common ones and work my way out to the more special purpose ones.

> **NOTE:** In a comment on issue #1, <a href="http://twitter.com/jeg2">@JEG2</a> correctly pointed out that this roadmap doesn't account for what happens after the whole class hierarchy is walked. Once `BasicObject` is reached, Ruby starts from the bottom again calling the `method_missing` hook, which is essentially an implicit step 6. I left this detail out for the sake of simplicity, but it's very important to at least be aware of.

### When to use ordinary class definitions

The following code implements a simple timer that can write out a timestamp to a file and read it back later to determine elapsed time. Study it and consider its design.

```ruby
class Timer
  MissingTimestampError = Class.new(StandardError)

  def initialize(dir=Turbine::Application.config_dir)
    @file = "#{dir}/timestamp"
  end

  def write_timestamp
    File.open(@file, "w") { |f| f << Time.now.utc.to_s }
  end
  
  def timestamp
    raise MissingTimestampError unless running?
    Time.parse(File.read(@file)).localtime
  end
  
  def elapsed_time
    (Time.now.utc - timestamp.utc) / 60.0 / 60.0
  end

  def clear_timestamp
    FileUtils.rm_f(@file)
  end

  def running?
    File.exist?(@file)
  end
end 
```

When deciding if just a plain old class definition will do, I often ask myself several questions.

* Is it likely is that I'll need to customize this code later for another purpose? 
* Is this code meant to be interacted with and extended by third party code? 
* Are there any common behaviors in this code I'd want to extract and use elsewhere?

Because this `Timer` class definition comes from a real project of mine, I can tell you that the answer to all of the above questions in the context this code is intended to be used is a simple 'no'. What this indicates to me is that while extension might be necessary at some point down the line, there is no immediate need to design for extensibility, and so we go with the most simple option that could possibly work.

Another indicator that a plain class definition might be appropriate here is the fact that most of the functionality in this class is centered around manipulating a particular bit of state, the <i>timestamp</i> file. The problem we are trying to solve is quite a narrow one, and a single-minded class definition reflects that.

The downside to designing code this way is that it does make third-party modification harder. If for example, you wanted to add some behavior around the `timestamp()` method, you have three options, none of them great:

 * You can create a subclass of `Timer`, but your new class won't be used by the application that defined `Timer` without modification.

 * You can create an instance of `Timer` and then add per-object behavior, but this has the same problem as subclassing.

 * You can use `alias_method` to create a monkeypatch to `Timer`, which will inject your code into the original application, but runs risks of naming clashes and other nasty things.

While it ultimately depends on how the calling code uses this `Timer` class, and what features are provided for making extensions, it's not going to be trivial to modify systems built in this fashion. But because we already determined this was a narrow bit of functionality designed to be used internally within a larger application, it isn't a problem that it isn't super extendable.

Many of the rules that apply to defining your own classes also apply to inheritance based designs, so let's investigate that now.

### When Inheritance Makes Sense

For those working with Rails, you already encounter class inheritance on a daily basis, through the ActiveRecord ORM. Despite the terrible choice of name, `ActiveRecord::Base` is a reasonable example of when class inheritance is a decent option.

Consider the typical ActiveRecord model, which is often extremely simple:

```ruby
class User < ActiveRecord::Base 
  has_many :comments

  validates_presence_of :name, :email
end
```

While it's true that in most interesting applications, models do become more complex, implementing intricate business logic, the amount of functionality added by the user is dwarfed by what ActiveRecord provides.

One key thing to notice about a subclass of `ActiveRecord::Base` is that by design, there is really no incentive to manage your own state. All state manipulation is passed upwards to the parent class to handle, which typically involves using a pre-configured database connection also managed by the parent class to persist whatever state is required.

Inheritance makes sense in situations where complex state manipulations are handled by the parent class. This is especially true if the parent class provides a boat-load of functionality which dwarfs the customization needs of the child class. Since both things are true about a typical ActiveRecord model, the design is certainly a reasonable choice.

However, before you start modeling your own projects after this pattern, you should take a look at the great pains that go into designing an extensible parent class. It's outside the scope of this article, but I'd recommend reading what <a href="http://is.gd/gW558 ">Yehuda Katz has to say about ActiveModel</a>, which provides the bulk of ActiveRecord's functionality under the hood.

Before we move on to other topics, I'd like to offer another example outside of the Rails world, just to help further illuminate the pattern.

The PDF generation library <a href="http://prawn.majesticseacreature.com">Prawn</a> provides a class that's designed to be inherited from, `Prawn::Document`. I made use of this functionality recently to build a small typesetting library for formatting technical articles. While I won't go into much detail here, you can check out the <a href="http://github.com/madriska/jambalaya">implementation and example code</a>.

What you'll find in <a href="https://github.com/madriska/jambalaya/blob/master/lib/jambalaya.rb">lib/jambalaya.rb</a> is that except for a custom factory method for generating the document, Jambalaya introduces no new state, relying on calls to `Prawn::Document` to do all the heavy lifting. You can also see that <a href="https://github.com/madriska/jambalaya/blob/master/example/rbp_ch1.rb">examples/rbp_ch1.rb</a> gives the illusion of a new special purpose DSL, but that in truth, almost all the work is being done by Prawn under the hood.

Unfortunately, the disadvantages of class inheritance become clear the farther away you get from these scenarios in which the subclass truly is analogous to its parent class. You get only one parent class, and chaining to it is a commitment that you must be willing to respect all the way up the hierarchy. For the scenarios we've shown, the benefits outweigh the drawbacks, but for many others, they do not.

In Issue #1, I asked the question of which techniques are special cases, and which are meant to be used commonly. While not rare by any means, inheritance falls closer to being a special case than it does to being the first tool you should reach for. If this comes as a surprise to you, it's about time for us to talk about modules.

### Mixing modules into a class

If you want to see the power of mixins, you need to look no farther than Ruby's
`Enumerable` module. Rather than relying on a common base class to provide
iterators for collections, Ruby mixes in the `Enumerable` module into its core structures, 
including `Array` and `Hash`. This is where a whole host of useful methods come from, 
including `map`, `select`, and `inject`.

The beauty of this design is that it imposes a much lighter contract than the rigid is-a relationship enforced by class inheritance. Instead, mixins focuses on what you can do with an object rather than what that object is. It makes perfect sense to say both `Hash` and `Array` objects have elements that can be enumerated over. As far as Ruby is concerned, the same can be true about any object which defines an `each()` method.

Let's take a look at a custom Ruby class which implements each and mixes in
`Enumerable`. It is a simple file-backed numerical queue, from the same project 
our `Timer` came from.

```ruby
class Queue 
  include Enumerable

  def initialize(file)
    @file = file
  end

  def entries
    return [] if empty?

    File.read(@file).split("\n").map { |e| e.to_f }
  end

  def each
    entries.each { |e| yield(e) }
  end

  # additional unrelated methods omitted
end
```

The data file this queue wraps looked something similar to the data shown below.

```
125.75
100.25
300.50
700
```

Given a properly formatted input file, it's possible to interact with the `Queue` like any other `Enumerable` object.

```ruby
queue = Queue.new("queue.txt")
p queue.map { |x| "Amount: #{x}" }
p queue.inject { |x,y| x + y }
```

If you go ahead and try this yourself, you'll find that it will work identically if you simply replace the first line with an array, as shown below.

```ruby
queue = [125.75, 100.25, 300.50, 700]
p queue.map { |x| "Amount: #{x}" }
p queue.inject { |x,y| x + y }
```

This simple example hints at the real beauty of `Enumerable` in particular, and the mixin technique in general. In reality, my `Queue` object and Ruby's `Array` class have very little in common. But in the context of how you can iterate over the two objects, they can share a matching interface for the things they do have in common.

This is where modules shine. They allow some of the benefits of inheritance in that they allow implementation sharing, but without the requirement of organizing things into a rigid class hierarchy. Things get even more interesting when you remember to tie your understanding of how modules work back to the way Ruby looks up methods.

### Exploiting the lookup order of mixins

Methods are looked up in mixins in reverse order of their inclusion, giving the last module you mixed in a priority spot in the lookup path. A pleasant effect that arises naturally from this rule is that it provides an elegant technique for monkey patching that does not rely on method aliasing. Let's look at a patch that uses method aliasing, and how it could be written differently.

Below is the code that Rubygems uses to patch `require` to add in gem loading functionality. Since `require` is just a method in Ruby, and not a keyword, the patch is relatively straightforward in pure Ruby.

```ruby
module Kernel
  alias gem_original_require require

  def require(path) # :doc:
    gem_original_require path
  rescue LoadError => load_error
    if load_error.message.end_with?(path)
      if Gem.try_activate(path)
        return gem_original_require(path)
      end
    end

    raise load_error
  end
end 
```

At the time this code was written, using method aliasing was the standard way of changing the behavior of an existing method. Aliases are used to make a copy of an existing method before modifying it, which allows customized code to delegate to the original method. This permits re-using the parts of the original method that are needed while (hopefully) preventing issues with backwards compatibility. The general approach works well, but it increases the chances that the copied methods will clash with each other as the chain gets longer, and also adds a number of superfluous methods to objects that are really just implementation details.

Taking advantage of Ruby's method lookup order in modules, we can get around the issues with aliasing by writing a patch similar to the one shown below.

```ruby
module GemCustomRequire
  def require(path) # :doc:
    super
  rescue LoadError => load_error
    if load_error.message.end_with?(path)
      if Gem.try_activate(path)
        return super
      end
    end

    raise load_error
  end   
end 

class Object
  include GemCustomRequire
end
```

Because the original `require()` method is defined within the `Kernel` module and not on `Object` itself, we can include our `GemCustomRequire` module and then use `super` to call the original require. The result is code that looks more natural and ordinary, reducing the amount of magic you need to know in order to understand it. It also completely avoids the possible issue of copied methods clashing with one another.

This ability to do safe monkeypatching that modules affords us has been picking up steam within popular Ruby projects. Rails 3 was in a large extent designed to afford this sort of modularity, for the express purpose of making it easier for third party plugins to hook into the system in a more graceful way than method aliasing. Other projects that require a high degree of extensibility are quickly following in its footsteps, which is a good thing.

While you're less likely to run into this question in application code than you are in library or framework code, knowing what mixins can gain you in terms of extensibility can really come in handy. There are tons of other good things to say about modules, but we'll need to save those for another day.

### Per Object Behavior

I was originally going to go into detail about mixing modules into individual objects as well as defining singleton methods. However, I think that can be a topic all of it's own, and I want to give it a proper treatment rather than tacking it on to the end of an already lengthy newsletter.

I promise we'll revisit it soon, but for those who absolutely want to explore potential uses of these techniques right away, I offer two small challenges.

1) Rather than using a framework for testing stubs, experiment with something like the code below next time you're writing tests.

```ruby
obj = Object.new

class << obj
  def my_stubbed_method

  end
end
```

2) Rather than re-opening a class to add some extra behavior, experiment with mixing modules into individual objects to get the extra features you need.

```ruby
module MathHelpers
  def sum
    inject { |x,y| x + y }
  end

  def average
    sum.to_f / length
  end
end

array = [1,2,3]
array.extend(MathHelpers)
p array.average
```

If you try these ideas out, you'll almost certainly find uses for them in other
contexts, too.

### Reflections

Hopefully you've learned something new about Ruby's method lookup rules, or at least been given some new things to think about and explore. If you've come from a background in which inheritance has been your only tool, you will likely have to retrain yourself a bit to make full use of what Ruby has to offer.

Whenever you compare one of these options to the other, consider your context
and how much the advantages and disadvantages of each technique affect your
particular situation. The correct approach always depends on that context, and
if in doubt, experiment and see what works best.

More discussion on this topic is welcome in the comments section below. While I wrote this article a while ago, I am happy to jump back into the topic as long as folks have interesting ideas and questions to share.
  
> **NOTE:** This article has also been published on the Ruby Best Practices blog. There [may be additional commentary](http://blog.rubybestpractices.com/posts/gregory/031-issue-2-method-lookup.html#disqus_thread) 
over there worth taking a look at.
