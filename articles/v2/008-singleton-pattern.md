Many design patterns that originated in other object-oriented languages have elegant Ruby translations. However, the [Singleton](http://en.wikipedia.org/wiki/Singleton_pattern) stands out as a construct that seems to have no good way to implement in Ruby. In this article, I will walk through the different options and explain why they all have something wrong with them. But first, we need a working definition of the singleton pattern to make sure we're on the same page.

Put briefly, the singleton pattern is a clever way of implementing global objects that you never need to explicitly instantiate. Not to be confused with Ruby's mostly unrelated concept of a "singleton class," the singleton pattern is applied when only a single instance of an object is needed across an entire application. Typical examples include objects that represent configuration data, global logging systems, and other similar structures. But there are also some subtle use cases in Ruby due to the fact that classes and modules are objects. For example, we have the `Math` module on which we can call methods such as `Math.sin()`, and `Math.cos()`. The `Math` module is acting as a singleton object in this context, even if it's not immediately obvious to the user. Keep in mind while reading this article that I've lumped this sort of use case in with the more traditional ones, as it shifts the perspective somewhat.

There are a lot of different ways to implement this general pattern in Ruby, but as I've already mentioned, they all pretty much suck. That said, studying this problem can teach us a thing or two about the subtleties (and warts) of the Ruby object system. As you read along, try figuring out the downsides of each implementation before moving on to read my explanations. This exercise will make the article more interesting and may even uncover some fresh ideas that I haven't considered yet.

### Using the `Singleton` module provided by the standard library

Ruby provides a standard library to assist in implementing the singleton pattern. The following code (originally from Practicing Ruby 1.25) illustrates how you can use this library to build a simple logger object.

```ruby
require "singleton"

class SimpleLogger
  include Singleton

  def initialize
    @output = []
  end

  attr_reader :output

  def error(message)
    output << formatted_message(message, "ERROR")
  end

  def info(message)
    output << formatted_message(message, "INFO")
  end

  def write(filename)
    File.open(filename, "a") { |f| f << output.join }
  end

  private

  def formatted_message(message, message_type)
    "#{Time.now} | #{message_type}: #{message}\n"
  end
end
```

By including the `Singleton` module, we make it so that it is no longer possible to create an instance of the `SimpleLogger` class in an ordinary way.

```
>> logger = SimpleLogger.new
NoMethodError: private method `new' called for SimpleLogger:Class
  from (irb):2
```

This behavior makes sense, because the point of the singleton pattern is to prevent multiple instances of a given object from being created. This example code shows how to get at a `SimpleLogger` instance in a way that guarantees that only one will be created.

```ruby
logger = SimpleLogger.instance

logger.error("Some serious problem")
logger.info("Something you might want to know")
logger.write("log.txt")
```

This interface is a bit cumbersome to work with, but it gets the job done, and on its own isn't too bad. However, disabling `new` and adding an `instance` method isn't all that the `Singleton` module does. It also does all of the following things:

* Overrides `inherited()` on the class to ensure that subclasses also retain `Singleton` behavior
* Overrides `dup()`/`clone()` on the class to ensure that copied classes also retain `Singleton` behavior
* Overrides `_load()` to call `instance()`, modifying `Marshal` loading behavior to return the single instance
* Overrides `_dump()` to strip state information when serializing via `Marshal`
* Overrides `dup()`/`clone()` on the instance to raise a `TypeError`, preventing cloning or duping of the instance

When you think about what a singleton object is actually meant to be, these changes make sense. However, many Rubyists look at this and see a whole lot of complexity without a lot of direct benefits. This impression causes many folks to avoid the use of the `Singleton` module in favor of implementations that are a bit more low ceremony. These implementations tend to ignore some of the edge cases that the `Singleton` module accounts for but are much easier to understand.

### Using a class consisting of only class methods

The following code uses ordinary class methods as an alternative to the previous approach. We explicitly call `undef_method` to make it so that instances of this class cannot be created, but otherwise this code is a vanilla Ruby class definition.

```ruby
class SimpleLogger
  class << self
    undef_method :new

    def output
      @output ||= []
    end

    def error(message)
      output << formatted_message(message, "ERROR")
    end

    def info(message)
      output << formatted_message(message, "INFO")
    end

    def write(filename)
      File.open(filename, "a") { |f| f << output.join }
    end

    private

    def formatted_message(message, message_type)
      "#{Time.now} | #{message_type}: #{message}\n"
    end
  end
end
```

Using this class is very simple, as the entire API consists of class method calls.

```ruby
SimpleLogger.error("Some serious problem")
SimpleLogger.info("Something you might want to know")
SimpleLogger.write("log.txt")
```

This approach isn't too bad, but it has its own set of caveats. The use of `undef_method` to disable the `new` method makes our `Class` object into something that is some ways class-like, but isn't quite a class anymore. From a purity standpoint, something just feels wrong about a construct that can exist in an inheritance hierarchy but cannot be instantiated. There is also the question of whether it ever really makes sense to create a subclass of a singleton object.

For a number of reasons, these philosophical issues tend to push folks in the direction of Ruby's `Module` construct, which at first glance seems to address some of these problems.

### Using a module consisting of only module methods

Every module is an object that cannot exist in a hierarchy and cannot be instantiated but otherwise holds similar properties of `Class` objects. Note how similar this code is to our previous example.

```ruby
module SimpleLogger
  class << self
    def output
      @output ||= []
    end

    def error(message)
      output << formatted_message(message, "ERROR")
    end

    def info(message)
      output << formatted_message(message, "INFO")
    end

    def write(filename)
      File.open(filename, "a") { |f| f << output.join }
    end

    private

    def formatted_message(message, message_type)
      "#{Time.now} | #{message_type}: #{message}\n"
    end
  end
end
```

The two approaches are so similar that they look identical from the end user's perspective:

```ruby
SimpleLogger.error("Some serious problem")
SimpleLogger.info("Something you might want to know")
SimpleLogger.write("log.txt")
```

Of course, if we look under the hood, we find that these two implementations are quite different. Although it's true that we've effectively made it impossible to create a subclass of `SimpleLogger` and that we didn't have to explicitly disable the `new` method because `Module` does not provide one, we now are faced with the problem that this module can be mixed into other objects.

Just as a class can have methods at the class level and the instance level, a module can have methods at the module level and the "mixin" level. Our `SimpleLogger` code defines all of its methods at the module level, which means that mixing it into an object via `include` or `extend` will not add any new functionality to the object it gets mixed into. From a purity standpoint, this approach is pretty much identical to the "useless instances" that would be possible for us to create if we allowed calls to the `new` method in our class-based `SimpleLogger`. Modules therefore don't actually give us much of an advantage over classes after all.

To make matters more confusing, Ruby provides us with a couple additional ways to use modules to implement the singleton pattern that bring new kinds of complexity into the mix.

### Using a module with `module_function`

If I were to create a list of Ruby's most confusing features, `module_function` would be near the top. It is a keyword (like `private` and `protected`), which allows you to specify certain methods within a module to be callable at the module level. This feature seems useful at a glance and is even used by Ruby's `Math` module. The interesting thing about module methods is that they serve as public methods on the module itself but get mixed into other objects as private methods. 

This example code demonstrates directly calling methods on the `Math` module, which looks similar to our previous module-based singleton pattern example.

```ruby
class Point
  def initialize(x,y)
    @x = x
    @y = y
  end

  attr_reader :x, :y

  def distance_to(other_point)
    Math.hypot(other_point.x - x, other_point.y - y)
  end
end

point_a = Point.new(0,0)
point_b = Point.new(4,3)

p point_a.distance_to(point_b)
```

If we instead include the `Math` module into the `Point` class, we see that the behavior is different than defining methods directly on the `Math` module because its functionality does get mixed into `Point`.

```ruby
class Point
  include Math

  def initialize(x,y)
    @x = x
    @y = y
  end

  attr_reader :x, :y

  def distance_to(other_point)
    hypot(other_point.x - x, other_point.y - y)
  end
end

point_a = Point.new(0,0)
point_b = Point.new(4,3)

p point_a.distance_to(point_b)
```


This pattern of having a module that doubles as a mixin and a singleton object probably has limited applications, but it seems reasonable for the `Math` module because each method provided by `Math` is purely functional and is also unlikely to clash with other features within a given class. But even if mixing in the `Math` module is convenient and relatively safe, we wouldn't want our `Point` object to expose the features that the `Math` object provides via its public API. This is where we notice that `module_function` anticipates this potential problem and attempts to solve it by making all mixed-in methods private.

```
>> point_a.hypot(4,3)
NoMethodError: private method `hypot' called for #<Point:0x0000010083bd68 @x=0, @y=0>
    from (irb):20
    from /Users/seacreature/.rvm/rubies/ruby-1.9.3-rc1/bin/irb:16:in `<main>'
```

Though this is potentially a useful feature, we now must keep in mind that modules that utilize `module_function` do not have ordinary mixin behavior. However, this is not the main reason why I said that `module_function` is confusing. To generate our proper "WTF?" moment, we can attempt to use `module_function` to implement our `SimpleLogger` object.

```ruby
module SimpleLogger
  module_function
  
  def output
    @output ||= []
  end

  def error(message)
    output << formatted_message(message, "ERROR")
  end

  def info(message)
    output << formatted_message(message, "INFO")
  end

  def write(filename)
    File.open(filename, "a") { |f| f << output.join }
  end

  private

  def formatted_message(message, message_type)
    "#{Time.now} | #{message_type}: #{message}\n"
  end
end
```

That implementation *almost* works, but ends up failing with an error that is quite surprising unless you know exactly how `module_function` works. 

```
>> SimpleLogger.error("This won't actually succeed")
NoMethodError: undefined method `formatted_message' for SimpleLogger:Module
    from (irb):29:in `error'
    from (irb):50
    from /Users/seacreature/.rvm/rubies/ruby-1.9.3-rc1/bin/irb:16:in `<main>'
```

The reason this happens is multifaceted. When using `module_function` with no arguments, as we do, the public methods that are defined after the `module_function` call are treated as module functions and get copied onto the module itself. However, once the private keyword is reached, the methods are no longer treated as module functions and thus don't end up getting copied onto the module. It is therefore effectively impossible to use `module_function` if you want to have your module methods call any private methods. If we accept this limitation and remove the `private` keyword from our `SimpleLogger` definition, things will work as expected using the original runner code from the previous examples.

If you made it through the previous example without becoming incredibly confused, try guessing what this code will do before running it, and then prepare to be surprised.

```ruby
module A
  def x
    10
  end

  module_function :x 

  def x
    12
  end
end

p A.x

class B
  include A
end

p B.new.x
```

If you managed to run this example without thinking that `module_function` is an abomination that should be removed from the Ruby language post-haste, please leave a convincing argument in the comments section. But you may want to first look at `module_function`'s slightly less awkward cousin, `extend self`.

### Using a module with `extend self`

The main problem with `module_function` is that it has so many moving parts. You need to really understand a fairly broad range of Ruby concepts in order to use it effectively. But if we accept the notion that it's sometimes useful for a singleton object to double as a mixin, we can try another approach that behaves similar to `module_function` without too many special cases to consider.

We can start by exploring a contrived example that demonstrates what happens when you use `extend` to mix a module into itself. If it's not immediately obvious what the `extend self` line does, treat it as a black box for now and focus on how the objects behave as we call methods on them.

```ruby
module A
  extend self

  def x
    y
  end
  
  private

  def y
    "yay!"
  end
end

class B
  include A
end

A.x      #=> "yay!"
A.y      #=> raises NoMethodError: private method `y' called for A:Module
B.new.x  #=> "yay!"
B.new.y  #=> raises NoMethodError: private method `y' called for #<B:...>
```

Speaking purely from the perspective of the externally visible behavior, we see that the key difference between `module_function` and `extend self` is that `extend self` results in identical behavior at both the module level and the mixin level when it comes to access control. Both private methods and public methods get mixed into the target object, and their access control is kept the same as whatever it was in the module definition. This is good because it means that your module can actually define private methods without any consequences. The main technical downside of this approach is that if `Math` were implemented in this way, including the `Math` module in a given object would add all the functions that can be called on the `Math` module to the public API of that object. There are workarounds to get that sort of behavior without `module_function`, but they're cumbersome and not really worth talking about. Assuming that we don't care about this subtle distinction, the following code will implement a working `SimpleLogger` singleton object that's a bit easier to reason about than the `module_function` version:

```ruby
module SimpleLogger
  extend self

  def output
    @output ||= []
  end

  def error(message)
    output << formatted_message(message, "ERROR")
  end

  def info(message)
    output << formatted_message(message, "INFO")
  end

  def write(filename)
    File.open(filename, "a") { |f| f << output.join }
  end

  private

  def formatted_message(message, message_type)
    "#{Time.now} | #{message_type}: #{message}\n"
  end
end
```

Even if the `extend self` approach is more fundamentally simple than `module_function`, it is not necessarily easy to learn or easy to understand. I go into great detail explaining exactly how this technique works in [Practicing Ruby 1.10](http://blog.rubybestpractices.com/posts/gregory/040-issue-10-uses-for-modules.html), but the two pages it takes me to explain it at a very high level serve as a hint that we're probably trying too hard to be clever when we write code this way. 

So far, we've gone down a deep bunny hole because each alternative approach we've attempted was about solving a problem with the previous implementation. But for every improvement we make, we lose something in return. Much of our struggle has to do with the costs involved in trying to implement a singleton object that conforms to the expectations we have about classes and modules. To sidestep this issue, we can think about a solution that works directly with an individual object instead. 

### Using a bare instance of `Object`

The following code shows how to implement the singleton pattern by adding methods to a bare instance of `Object`. This code may look a bit strange at first but is at its core the same as defining methods on any other Ruby object, including instances of the `Module` and `Class` classes.

```ruby
SimpleLogger = Object.new
class << SimpleLogger
  def output
    @output ||= []
  end

  def error(message)
    output << formatted_message(message, "ERROR")
  end

  def info(message)
    output << formatted_message(message, "INFO")
  end

  def write(filename)
    File.open(filename, "a") { |f| f << output.join }
  end

  private

  def formatted_message(message, message_type)
    "#{Time.now} | #{message_type}: #{message}\n"
  end
end
```

Although we didn't have to store the object in the `SimpleLogger` constant, doing so makes the familiar runner code we've been using over and over work exactly as expected.

```ruby
SimpleLogger.error("Some serious problem")
SimpleLogger.info("Something you might want to know")
SimpleLogger.write("log.txt")
```

The code looks and feels the same from the end-user's perspective, but we know that there is something quite different hiding under the hood. The good news is that this approach makes it so that our `SimpleLogger` is not a factory for creating new objects like classes are and cannot be mixed into other objects or part of a hierarchy. The bad news is that the resulting object is very opaque. When we inspect a class or module object, it at least gives us back its name. However, when we inspect this object, what we get is the following:

```
>> SimpleLogger
=> #<Object:0x0000010084bc18>
```

Documenting this object using something like RDoc would be similarly frustrating, as it wouldn't be able to infer much about the object without lots of explicit directives. Although Ruby is designed in terms of ordinary objects, its infrastructure is surely defined in terms of classes and modules.

We can't do anything about the documentation problem without major changes to Ruby, but we might be able to build our own custom construct that takes these basic ideas and adds better debugging support.

### Using a hand-rolled construct

A few days before writing this article, I complained on Twitter about the lack of a good way to implement the singleton pattern in Ruby and suggested that perhaps we needed a new first-order construct for these purposes. Someone was quick to point out that [Scala has such a construct](http://hestia.typepad.com/flatlander/2009/01/scala-for-c-programmers-part-2-singletons.html), which got the wheels turning in my head. As a point of reference, here's what the construct looks like in Scala:

```
object Universe {
  def contains(obj: Any): Boolean = true  
}

val v = Universe.contains(42)
```

Now, there is a big difference between having built-in support for something in a language and building some sort of hand-rolled approximation. However, I couldn't resist implementing a construct in Ruby that works roughly the same way as this Scala code. After some tinkering, I settled on this syntax:

```ruby
object "Universe" do
  def contains?(anything)
    true
  end
end

p Universe.contains?(42) #=> true
p Universe #=> #<Universe:2156157600>  
```

Because the call to `object` hides the actual object creation, I was able to add nice `inspect` output in a way that is transparent to the user. Similar debugging and introspection features could be added just as easily. Under the hood, I used an approach similar to working with bare objects, so I retain all the benefits of that approach while getting rid of some of the downsides. (NOTE: I've decided to leave implementing this construct as a homework exercise, but please let me know if you get stuck and want to see how I did it.)

Looking back on this code, I like the way the experiment went, but I am stuck wondering whether it makes sense to take it any farther. Without first-order support in the Ruby language for this construct, documentation would still be a struggle. Also, the awkward syntax breaks consistency with `Class` and `Module`. What we'd really want to be able to type is something like the following definition:

```ruby
object Universe 
  def contains?(anything)
    true
  end
end
```

Although the previous example is syntactically pleasing to me, I wonder if encouraging us to use more global functionality (and on a related note, more constants) is a good idea. At a minimum, such a change would need to also be mirrored in `Object.new` by adding a block form similar to the way that `Module` and `Class` work. This approach would end up looking something like this:

```ruby
universe = Object.new do
  def contains?(anything)
    true
  end
end
```

The thing we have to ask ourselves is whether these features would really make Ruby a better language to work in. To answer that question, we need to consider the costs and benefits of avoiding the singleton pattern entirely.

### Avoiding the singleton pattern entirely

Pretty much everything discussed so far is about avoiding explictly instantiating objects, which makes it possible for us to put off potentially expensive setup work and also makes it easier for us to prevent multiple instances of a given object from being instantiated within our applications. Some of our implementations do a good job of communicating these desires to the user by preventing them from creating instances. Others encourage a limited form of code reuse through module mixins, but with a number of caveats attached.

But in the end, we must not forget that the singleton pattern is essentially just a fancy way of managing global state. If we converted our `SimpleLogger` into an ordinary class and then did something like `$logger = SimpleLogger.new`, there would be marginal practical difference in the way things worked in our codebase. Things change slightly when we think of function bags like the `Math` module, but not as much as you might think. We must remember that no matter what form our singleton objects take, each one we add to our system is by definition less reusable and less testable due to its singular, global nature.

The question of whether to implement the singleton pattern really depends on the context, but it's safe to say that it's a bad default. However, this is a genuinely hard problem in object-oriented programming, which may explain why we've seen so many different attempts in Ruby without a real consensus on which way is best. We've also been unable to eliminate the pattern entirely, which is a sign that we can't simply write it off as one of those bad Java imports that real Rubyists freely ignore.

### Reflections

The process of writing this article has taught me a few things. First of all, most of the approaches we take to implement the singleton pattern are way too complicated. Although this grail quest for object-oriented purity is entertaining from an academic perspective, it isn't something we should need to think about in our daily coding lives.

That said, it seems like a first-order construct that lets us define individual objects in an elegant way would be an interesting addition to Ruby. If we gravitated toward a more prototype-based design style via these standalone objects while using modules for our code reuse, we might end up with very nice solutions that would make this "singleton object" problem just disappear. But then again, that would be a huge shift in the way we write Ruby code, and I'm not sure the juice is worth the squeeze.

In the end, it amazes me that I was able to write so much on this topic, but not in a good way. As powerful as Ruby-the-language is, it seems that we're still far from being able to balance that power with responsibility. Every approach I criticized in this article is one I've advocated for at some earlier point, and now I'm not so sure I like any of them.

My hope is that you got two things from reading this article: a deeper understanding of the complexity of Ruby's object system and an awareness of the trade-offs of various approaches to this problem, so that you can be better equipped when you encounter this sort of design in the wild. If I've accomplished that, then this article was well worth the effort it took to write up. If not, don't worry: we'll return to more practical content next week! :)
