Inheritance is a key concept in most object-oriented languages, but applying it
skillfully can be challenging in practice. Back in 1989, [M.
Sakkinen](http://users.jyu.fi/~sakkinen/) wrote a paper called [Disciplined
inheritance](http://scholar.google.com/scholar?cluster=5893037045851782349&hl=en&as_sdt=0,7&sciodt=0,7)
that addresses these problems and offers some useful criteria for working around
them. Despite being over two decades old, this paper is extremely relevant to
the modern Ruby programmer.

Sakkinen's central point seems to be that most traditional uses of inheritance lead to poor encapsulation, bloated object contracts, and accidental namespace collisions. He provides two patterns for disciplined inheritance and suggests that by normalizing the way that we model things, we can apply these two patterns to a very wide range of scenarios. He goes on to show that code that conforms to these design rules can easily be modeled as ordinary object composition, exposing a solid alternative to traditional class-based inheritance.

These topics are exactly what this two-part article will cover, but before we can address them, we should establish what qualifies as inheritance in Ruby. The general term is somewhat overloaded, so a bit of definition up front will help start us off on the right foot. 

### Flavors of Ruby inheritance

Although classical inheritance is centered on the concept of class-based hierarchies, modern object-oriented programming languages provide many different mechanisms for code sharing. Ruby is no exception: it provides four common ways to model inheritance-based relationships between objects.

* Classes provide a single-inheritance model similar to what is found in many other object-oriented languages, albeit lacking a few privacy features.

* Modules provide a mechanism for modeling multiple inheritance, which is easier to reason about than C++ style class inheritance but is more powerful than Java's interfaces.

* Transparent delegation techniques make it possible for a child object to dynamically forward messages to a parent object. This technique has similar effects as class-/module-based modeling on the child object's contract but preserves encapsulation between the objects.

* Simple aggregation techniques make it possible to compose objects for the purpose of code sharing. This technique is most useful when the subobject is not meant to be a drop-in replacement for the superobject.

Although most problems can be modeled using any one of these techniques, they each have their own strengths and weaknesses. Throughout both parts of this article, I'll point out the trade-offs between them whenever it makes sense to do so.

### Modeling incidental inheritance 

Sakkinen describes **incidental inheritance** as the use of an inheritance-based modeling approach to share implementation details between dissimiliar objects. That is to say that child (consumer) objects do not have an _is-a_ relationship to their parents (dependencies) and therefore do not need to provide a superset of their parent's functionality.

In theory, incidental inheritance is easy to implement in a disciplined way because it does not impose complex constraints on the relationships between objects within a system. As long as the child object is capable of working without errors for the behaviors it is meant to provide, it does not need to take special care to adhere to the [Liskov Substitution Principle](http://blog.rubybestpractices.com/posts/gregory/055-issue-23-solid-design.html). In fact, the child needs only to expose and interact with the bits of functionality from the parent object that are specifically relevant to its domain.

Regardless of the model of inheritance used, Sakkinen's paper suggests that child objects should rely only on functionality provided by immediate ancestors. This is essentially an inheritance-oriented parallel to the [Law of Demeter](http://en.wikipedia.org/wiki/Law_of_Demeter) and sounds like good advice to follow whenever it is practical to do so. However, this constraint would be challenging to enforce at the language level in Ruby and may not be feasible to adhere to in every imaginable scenario. In practice, the lack of adequate privacy controls in Ruby make traditional class hierarchies or module mixins quite messy for incidental inheritance, which complicates things a bit. But before we discuss that problem any further, we should establish what incidental inheritance looks like from several different angles in Ruby.

In the following set of examples, I construct a simple `Report` object that computes the sum and average of numbers listed in a text file. I break this problem into three distinct parts: a component that provides functionality similar to Ruby's `Enumerable` module, a component that uses those features to do simple calculations on numerical data, and a component that outputs the final report. The contrived nature of this scenario should make it easier to examine the structural differences between Ruby's various ways of implementing inheritance relationships, but be sure to keep some more realistic scenarios in the back of your mind as you work through these examples. 

The classical approach of using a class hierarchy for code sharing is worth looking at, even if most practicing Rubyists would quickly identify this as the wrong approach to this particular problem. It serves as a good baseline for identifying the problems introduced by inheritance and how to overcome them. As you read through the following code, think of its strengths and weaknesses, as well as any alternative ways to model this scenario that you can come up with.

```ruby
class EnumerableCollection
  def count
    c = 0
    each { |e| c += 1 }
    c
  end

  # Samnang's implementation from Issue 2.4
  def reduce(arg=nil) 
    return reduce {|s, e| s.send(arg, e)} if arg.is_a?(Symbol)

    result = arg
    each { |e| result = result ? yield(result, e) : e }

    result
  end
end

class StatisticalCollection < EnumerableCollection
  def sum
    reduce(:+) 
  end

  def average
    sum / count.to_f
  end 
end

class StatisticalReport < StatisticalCollection
  def initialize(filename)
    self.input = filename
  end

  def to_s
    "The sum is #{sum}, and the average is #{average}"
  end

  private 

  attr_accessor :input

  def each
    File.foreach(input) { |e| yield(e.chomp.to_i) }
  end
end

puts StatisticalReport.new("numbers.txt")
```

Through its inheritance-based relationships, `StatisticalReport` is able to act as a simple presenter object while relying on other reusable components to crunch the numbers for it. The `EnumerableCollection` and `StatisticalCollection` objects do most of the heavy lifting while managing to remain useful for a wide range of different applications. The division of responsibilities between these components is reasonably well defined, and if you ignore the underlying mechanics of the style of inheritance being used here, this example is a good demonstration of effective code reuse.

Unfortunately, the devil is in the details. When viewed from a different angle, it's easy to see a wide range of problems that exist even in this very simple application of class-based inheritance:

1. `EnumerableCollection` and `StatisticalCollection` can be instantiated, but
it is not possible to do anything meaningful with them as they are currently
written. Although it's not always a bad idea to make use of abstract
classes, valid uses of that pattern typically invert the relationship shown
here, with the child object filling in a missing piece so that its parent can do
a complex job.

2. Although `StatisticalReport` relies on only two relatively generic methods from `StatisticalCollection` and `StatisticalCollection` similarly relies on only two methods from `EnumerableCollection`, the use of class inheritance forces a rigid hierarchical relationship between the objects. Even if it's not especially awkward to say a `StatisticalCollection` is an `EnumerableCollection`, it's definitely weird to say that a `StatisticalReport` is also an `EnumerableCollection`. What makes matters worse is that this sort of modeling prevents `StatisticalReport` from inheriting from something more related to its domain such as a `HtmlReport` or something similar. As my [favorite OOP rant](http://lists.canonical.org/pipermail/kragen-tol/2011-August/000937.html) proclaims, class hierarchies do not exist simply to satisfy our inner Linnaeus.

3. There is no encapsulation whatsoever between the components in this system. The purely functional nature of both `EnumerableCollection` and `Statistics` make this less of a practical concern in this particular example but is a dangerous characteristic of all code that uses class-based inheritance in Ruby. Any instance variables created within a `StatisticalReport` object will be directly accessible in method calls all the way up its ancestor chain, and the same goes for any methods that `StatisticalReport` defines. Although a bit of discipline can help prevent this from becoming a problem in most simple uses of class inheritance, deep method resolution paths can make accidental collisions of method definitions or instance variable names a serious risk. Such a risk might be mitigated by the introduction of class-specific privacy controls, but they do not exist in Ruby. 

4. As a consequence of points 2 and 3, the `StatisticalReport` object ends up with a bloated contract that isn't representative of its domain model. It'd be awkward to call `StatisticalReport#count` or `StatisticalReport#reduce`, but if those inherited methods are not explicitly marked as private in the `StatisticalReport` definition, they will still be callable by clients of the `StatisticalReport` object. Once again, the stateless nature of this program makes the effects less damning in this particular example, but it doesn't take much effort to imagine the inconsistencies that could arise due to this problem. In addition to real risks of unintended side effects, this kind of modeling makes it harder to document the interface of the `StatisticalReport` in a natural way and diminishes the usefulness of Ruby's reflective capabilities.

At least some of these issues can be resolved through the use of Ruby's module-based mixin functionality. The following example shows how our class-based code can be trivially refactored to use modules instead. Once again, as you read through the code, think of its strengths and weaknesses as well as how you might approach the problem differently if it were up to you to design this system.

```ruby
module SimplifiedEnumerable
  def count
    c = 0
    each { |e| c += 1 }
    c
  end

  # Samnang's implementation from Issue 2.4
  def reduce(arg=nil) 
    return reduce {|s, e| s.send(arg, e)} if arg.is_a?(Symbol)

    result = arg
    each { |e| result = result ? yield(result, e) : e }

    result
  end
end

module Statistics
  def sum
    reduce(:+) 
  end

  def average
    sum / count.to_f
  end 
end

class StatisticalReport
  include SimplifiedEnumerable
  include Statistics

  def initialize(filename)
    self.input = filename
  end

  def to_s
    "The sum is #{sum}, and the average is #{average}"
  end

  private 

  attr_accessor :input

  def each
    File.foreach(input) { |e| yield(e.chomp.to_i) }
  end
end

puts StatisticalReport.new("numbers.txt")
```

Using module mixins does not improve the encapsulation of the components in the system or solve the problem of `StatisticalReport` inheriting methods that aren't directly related to its problem domain, but it does alleviate some of the other problems that Ruby's class-based inheritance causes. In particular, it makes it no longer possible to create instances of objects that wouldn't be useful to use as standalone objects and also loosens the dependencies between the components in the system.

Although the `Statistics` and `SimplifiedEnumerable` modules are still not capable of doing anything useful without being tied to some other object, the relationship between them is much looser. When the two are mixed into the `StatisticalReport` object, an implicit relationship between `Statistics` and `SimplifiedEnumerable` exists due to the calls to `reduce` and `count` from within the `Statistics` module, but this relationship is an implementation detail rather than a structural constraint. To see the difference yourself, think about how easy it would be to switch `StatisticalReport` to use Ruby's `Enumerable` module instead of the `SimplifiedEnumerable` module I provided and compare that to the class-based implementation of this scenario.

The bad news is that the way that modules solve some of the problems that we discovered about class hierarchies in Ruby ends up making some of the other problems even worse. Because modules tend to provide a whole lot of functionality based on a very thin contract with the object they get mixed into, they are one of the leading causes of child obesity. For example, swapping my `SimplifiedEnumerable` module for Ruby's `Enumerable` method would cause a net increase of 42 new methods that could be directly called on `StatisticalReport`. And now, rather than having a single path to follow in `StatisticalReport` to determine its ancestry chain, there are two. A nice feature of mixins is that they have fairly simple rules about how they get added to the method lookup path to avoid some of the complexities involved in class-based multiple inheritance, but you still need to memorize those rules and be aware of the combinatorial effects of module inclusion.

As it turns out, modules are a pragmatic compromise that is convenient to use but only slightly more well-behaved than traditional class inheritance. In simple situations, they work just fine, but for more complex systems they end up requiring an increasing amount of discipline to use effectively. Nonetheless, modules tend to be used ubiquitously in Ruby programs despite these problems. A naïve observer might assume that this is a sign that we don't have a better way of doing things in Ruby, but they would be mostly wrong.

All the problems discussed so far with inheritance can be solved via simple aggregation techniques. For strong evidence of that claim, take a look at the refactored code shown here. As in the previous examples, keep an eye out for the pros and cons of this modeling strategy, and think about what you might do differently.

```ruby
class StatisticalCollection
  def initialize(data)
    self.data = data
  end

  def sum
    data.reduce(:+) 
  end

  def average
    sum / data.count.to_f
  end 

  private

  attr_accessor :data
end

class StatisticalReport
  def initialize(filename)
    self.input = filename
    
    self.stats = StatisticalCollection.new(each)
  end

  def to_s
    "The sum is #{stats.sum}, and the average is #{stats.average}"
  end

  private 

  attr_accessor :input, :stats

  def each
    return to_enum(__method__) unless block_given?

    File.foreach(input) { |e| yield(e.chomp.to_i) }
  end
end

puts StatisticalReport.new("numbers.txt")
```

The first thing you'll notice is that the code is much shorter, as if by magic, but really it's because I completely cheated here and got rid of my counterfeit `Enumerable` object so that I could expose a potentially good idiom for dealing with iteration in an aggregation-friendly way. Feel free to mentally replace the object passed to `StatisticalCollection`'s constructor with something like the code shown here if you don't want me to get away with parlor tricks:

```ruby
require "forwardable"

class EnumerableCollection
  extend Forwardable

  # Forwardable bypasses privacy, which is what we want here.
  delegate :each => :data

  def initialize(data)
    self.data = data
  end

  def count
    c = 0
    each { |e| c += 1 }
    c
  end

  # Samnang's implementation from Issue 2.4
  def reduce(arg=nil) 
    return reduce {|s, e| s.send(arg, e)} if arg.is_a?(Symbol)

    result = arg
    each { |e| result = result ? yield(result, e) : e }

    result
  end

  private

  attr_accessor :data
end
```

Regardless of what iteration strategy we end up using, the following points are worth noting about the way we've modeled our system this time around:

1. There are three components in this system, all of which are useful and testable as standalone objects.

2. The relationships between all three components are purely indirect, and the coupling between the objects is limited to the names and behavior of the methods called on them rather than their complete surfaces.

3. There is strict encapsulation between the three components: each have their own namespace, and each can enforce their own privacy controls. It's possible of course to side-step these protections, but they are at least enabled by default. The issue of accidental naming collisions between methods or variables of objects is completely eliminated.

4. As a result of points 2 and 3, the surface of each object is kept narrowly in line with its own domain. In fact, the public interface of `StatisticalReport` has been reduced to its constructor and the `to_s` method, making it as thin as possible while still being useful. 

There are certainly downsides to using aggregation; it is not a golden hammer by any means. But when it comes to **incidental inheritance**, it seems to be the right tool for the job more often than not. I'd love to hear counterarguments to this claim, though, so please do share them if you have something in mind that you don't think would gracefully fit this style of modeling.

### Reflections

Although it may be a bit hard to see why disciplined inheritance matters in the trivial scenario we've been talking about throughout this article, it become increasingly clear as systems become more complex. Most scenarios that involve incidental inheritance are actually relatively horizontal problems in nature, but the use of class-based inheritance or module mixins forces a vertical method lookup path that can become very unwieldy, to say the least. When taken to the extremes, you end up with objects like `ActiveRecord::Base`, which has a path that is 43 levels deep, or `Prawn::Document`, which has a 26-level-deep path. In the case of Prawn, at least, this is just pure craziness that I am ashamed to have unleashed upon the world, even if it seemed like a good idea at the time.

In a language like Ruby that lacks both multiple inheritance and true class-specific privacy for variables and methods, using class-based hierarchies or module mixins for complex forms of incidental inheritance requires a tremendous amount of discipline. For that reason, the extra effort involved in refactoring towards an aggregation-based design pales in comparison to the maintenance headaches caused by following the traditional route. For example, in both `Prawn` and `ActiveRecord`, aggregation would make it possible to flatten that chain by an order of magnitude while simultaneously reducing the chance of namespace collisions, dependencies on lookup order, and accidental side effects due to state mutations. It seems like the cost of somewhat more verbose code would be well worth it in these scenarios.

In Issue 3.8, we will move on to discuss an essential form of inheritance that Sakkinen refers to as **completely consistent inheritance**. Exploring that topic will get us closer to the concept of mathematical subtypes, which are much more interesting at the theoretical level than incidental inheritance relationships are. But because Ruby's language features make even the simple relationships described in this issue somewhat challenging to manage in an elegant way, I am still looking forward to hearing your ideas and questions about the things I've covered so far.

A major concern I have about incidental inheritance is that I still don't have a
clear sense of where to draw the line between the two extremes I've outlined in
this article. I definitely want to look further into this area, so please leave
a comment if you don't mind sharing your thoughts.


