In [Issue 3.7](http://practicingruby.com/articles/24), I started to explore the criteria laid out by Sakkinen's
[Disciplined Inheritance](http://scholar.google.com/scholar?cluster=5893037045851782349&hl=en&as_sdt=0,7&sciodt=0,7), 
a language-agnostic paper published more than two decades ago that is surprisingly 
relevant to the modern Ruby programmer. In this issue, we continue where Issue 3.7 
left off: on the question of how to maintain complete compatibility between
parent and child objects in inheritance-based domain models. Or, to put it another way,
this article explores how to reuse code safely within a system—
without it becoming a maintenance nightmare.

After taking a closer look at what Sakkinen exposed regarding this topic, I came to
realize that the ideas he presented were strikingly similar to the [Liskov Substitution
Principle](http://en.wikipedia.org/wiki/Liskov_Substitution_Principle). In fact,
the extremely dynamic nature of Ruby makes 
establishing [a behavioral notion of subtyping](http://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.39.1223) (Liskov and Wing 1993)
a prerequisite for developing disciplined inheritance practices. 
As a result, this article refers to Liskov's work more than Sakkinen's, 
even though both papers have extremely interesting things to say on this topic. 

### Defining a behavioral subtype 

Both Sakkinen and Liskov describe the essence of the inheritance relationship as 
the ability of a child object to serve as a drop-in replacement wherever
its parent object can be used. I've greatly simplified the concept by
stating it in such a general fashion, but this is the thread that ties
their independent works together. 

Liskov goes a step farther than Sakkinen by defining two kinds of 
behavioral subtypes: children that extend the behavior specified by their 
parents, and children that constrain the behavior specified by their parents. 
These concepts are not mutually exclusive, but because each brings up
its own set of challenges, it is convenient to separate them in this
fashion.

Both Sakkinen and Liskov emphasize that the abstract concept of subtyping 
is  much more about the observable behavior of objects than it is about
what exactly is going on under the hood. This concept is a natural way of thinking
for Rubyists, and it is worth keeping in mind as you read through the rest
of this article. In particular, when we talk about the type of an object,
we are focusing on what that object *does*, not what it *is*.

Although the concept of a behavioral subtype sounds like a direct analogue for
what we commonly refer to as "duck typing" in Ruby, the former is about
the full contract of an object rather than how it acts under certain
circumstances. I go into more detail about the differences between
these concepts toward the end of this article,
but before we can discuss them meaningfully, we need to take a look
at Liskov's two types of behavioral subtyping and how they can
be implemented.

### Behavioral subtypes as extensions

Whether you realize it or not, odds are good that you are already familiar with using behavioral subtypes as extensions. Whenever we inherit from `ActiveRecord::Base` or mix `Enumerable` into one of our objects, we're making use of this concept. In essence, the purpose of an extension is to bolt new behavior on top of an existing type to form a new subtype.

To ensure that our child objects maintain the substitution principle, we need to make sure that any new behavior and modifications introduced by extensions follow a few simple rules. In particular, all new functionality must be either completely transparent to the parent object or defined in terms of the parent object's functionality. Changing the signature of a method provided by the parent object would be considered an incompatible change, as would directly modifying instance variables referenced by the parent object. These strict rules may seem like overkill, but they are the only way to guarantee that your extended subtypes will be drop-in replacements for their supertypes.

In practice, obeying these rules is not as hard as it seems. For example, suppose we wanted to extend `Prawn::Document` so that it implements some helpers for article typesetting:

```ruby
Prawn::Article.generate("test.pdf") do
  h1 "Criteria for Disciplined Inheritance"
 
  para %{
    This is an example of building a Prawn-based article
    generator through the use of a behavioral subtype as
    an extension. It's about as wonderful and self-referential
    as you might expect.
  }

  h2 "Benefits of behavioral subtyping"

  para %{
    The benefits of behavioral subtyping cannot be directly
    known without experiencing them for yourself.
  }

  para %{
    But if you REALLY get stuck, try asking Barbara Liskov.
  }
end
```

The most simple way to implement this sort of domain language would be to create a subclass of `Prawn::Document`, as shown in the following example:

```ruby
module Prawn
  class Article < Document
    include Measurements

    def h1(contents)
      text(contents, :size => 24)
      move_down in2pt(0.3)
    end

    def h2(contents)
      move_down in2pt(0.1)
      text(contents, :size => 16)
      move_down in2pt(0.2)
    end

    def para(contents)
      text(contents.gsub(/\s+/, " "))
      move_down in2pt(0.1)
    end
  end
end
```

As far as Liskov is concerned, `Prawn::Article` is a perfectly legitimate extension because instances of it are drop-in substitutes for `Prawn::Document` objects. In fact, this sort of extension is trivial to prove to be a behavioral subtype because it is defined purely in terms of public methods that are provided by its parents (`Prawn::Document` and `Prawn::Measurements`). Because the functionality added is so straightforward, the use of subclassing here might just be the right tool for the job. 

The downside of using subclassing is that even minor alterations to program requirements can cause encapsulation-related issues to become a real concern. For example, if we decide that we want to add a pair of instance variables that control the fonts used for headers and paragraphs, it would be hard to guarantee that these variables wouldn't clash with the data contained within `Prawn::Document` objects. We can assume that calls to public methods provided by the parent object are safe, but we cannot say the same about referencing instance variables, so a delegation-based model starts to look more appealing.

Suppose we wanted to support the following API, but via delegation rather than subclassing:

```ruby
Prawn::Article.generate("test.pdf") do
  header_font    "Courier"
  paragraph_font "Helvetica"

  h1 "Criteria for Disciplined Inheritance"
 
  para %{
    This is an example of building a Prawn-based article
    generator through the use of a behavioral subtype as
    an extension. It's about as wonderful and self-referential
    as you might expect.
  }

  h2 "Benefits of behavioral subtyping"

  para %{
    The benefits of behavioral subtyping cannot be directly
    known without experiencing them for yourself.
  }

  para %{
    But if you REALLY get stuck, try asking Barbara Liskov.
  }
end
```

Using a `method_missing` hook and the `Prawn::Article.generate` class method, it
is fairly easy to implement this DSL:

```ruby
module Prawn
  class Article
    def self.generate(*args, &block)
      Prawn::Document.generate(*args) do |pdf|
        new(pdf).instance_eval(&block)
      end
    end

    def initialize(document)
      self.document = document      
      document.extend(Prawn::Measurements)

      # set defaults so that @paragraph_font and @header_font are never nil.
      paragraph_font "Times-Roman"
      header_font    "Times-Roman"
    end

    def h1(contents)
      font(header_font) do
        text(contents, :size => 24)
        move_down in2pt(0.3)
      end
    end

    def h2(contents)
      font(header_font) do
        move_down in2pt(0.1)
        text(contents, :size => 16)
        move_down in2pt(0.2)
      end
    end

    def para(contents)
      font(paragraph_font) do
        text(contents.gsub(/\s+/, " "))
        move_down in2pt(0.1)
      end
    end

    def paragraph_font(font=nil)
      return @paragraph_font = font if font

      @paragraph_font
    end

    def header_font(font=nil)
      return @header_font = font if font

      @header_font
    end

    def method_missing(id, *args, &block)
      document.send(id, *args, &block)
    end

    private

    attr_accessor :document
  end
end
```

Taking this approach involves writing more code and adds some complexity. However, that is a small price to pay for the peace of mind that comes with cleanly separating the data contained within the `Prawn::Article` and `Prawn::Document` objects. This design also makes it harder for `Prawn::Article` to have name clashes with `Prawn::Document`'s private methods and forces any private method calls to `Prawn::Document` to be done explicitly. Because transparent delegation exposes the full contract of the parent object, it is still necessary for the child object to maintain full compatibility with those methods in the same manner that a class-inheritance-based model would. Nonetheless, this pattern provides a safer way to implement subtypes because it avoids incidental clashes, which could otherwise occur easily.

Although the examples we've looked at so far—combined with your own experiences—should give you a good sense of how to extend code via behavioral subtypes, there are some common pitfalls I have glossed over in order to keep things simple. I'll get back to those before the end of the article, but for now let's turn our attention to the other kind of subtypes Liskov describes in her paper. She refers to them as _constrained subtypes_, but I call them _restriction subtypes_ as an easy-to-remember mirror image of the _extension subtype_ concept.

### Behavioral subtypes as restrictions

Just as subtypes can be used to extend the behavior of a supertype, they can also be used to restrict generic behaviors by providing more specific implementations of them. The example Liskov uses in her paper illustrates how a stack structure can be viewed as a restriction on the more general concept of a bag.

In its most simple form, a bag is essentially nothing more than a set that can contain duplicates. Items can be added and removed from a bag, and it is possible to determine whether the bag contains a given item. However, much like with a set, order is not guaranteed. The following code, which is somewhat of a contrived example, implements a `Bag` object similar to the one described in Liskov's paper:

```ruby
ContainerFullError  = Class.new(StandardError)
ContainerEmptyError = Class.new(StandardError)

class Bag
  def initialize(limit)
    self.items  = [] 
    self.limit = limit
  end

  def push(obj)
    raise ContainerFullError unless data.length < limit

    data.shuffle!
    data.push(obj)
  end

  def pop
    raise ContainerEmptyError if data.empty?

    data.shuffle!
    data.pop
  end

  def include?(obj)
    data.include?(obj)
  end

  private

  attr_accessor :items, :limit
end
```

The challenge in determining whether a `Stack` object can meaningfully be considered a subtype of this sort of structure is that we need to find a way to describe the functionality of a bag so that it is general enough to allow for interesting subtypes to exist but specific enough to allow the `Bag` object to be used on its own in a predictable way. Because Ruby lacks the design-by-contract features that Liskov depends on in her paper, we need to describe this specification verbally rather than relying on our tools to enforce them for us. Something like the following list of rules is roughly similar to what she describes more formally in her work:

1) A bag has `items` and a size `limit`.

2) A bag has a `push` operation, which adds a new object to the bag's `items`.

* If the current number of `items` is less than the `limit`, the new object is added to the bag's `items`.

* Otherwise, a `ContainerFullError` is raised.

3) A bag has a `pop` operation, which removes an object from the bag's `items` and returns it as a result.

* If the bag has no `items`, a `ContainerEmptyError` is raised.

* Otherwise, one object is removed from the bag's `items` and returned.

4) A bag has an `include?` operation, which indicates whether the provided object is one of bag's `items`.

* If the bag's `items` contains the provided object, `true` is returned.

* Otherwise, `false` is returned.

With these rules in mind, we can see that the following `Stack` object satisfies the definition of a bag while simultaneously introducing a predictable ordering to `items`:

```ruby
ContainerFullError  = Class.new(StandardError)
ContainerEmptyError = Class.new(StandardError)

class Stack
  def initialize(limit)
    self.items  = [] 
    self.limit = limit
  end

  def push(obj)
    raise ContainerFullError unless data.length < limit

    data.push(obj)
  end

  def pop
    raise ContainerEmptyError if data.empty?

    data.pop
  end

  def include?(obj)
    data.include?(obj)
  end

  private

  attr_accessor :items, :limit
end
```

With this example code in mind, we can specify the behavior of a stack in the following way:

1) A stack is a bag.

2) A stack's `pop` operation follows a last-in, first-out (LIFO) order.

Because the ordering requirements of a stack don't conflict with the defining characteristics of a bag, a stack can be substituted for a bag without any issues. The key thing to keep in mind here is that restriction subtypes can create additional constraints on top of what was specified by their supertypes but cannot loosen the constraints put upon them by their ancestors in any way. For example, based on the way we defined bag objects, we would not be able to return `nil` instead of raising `ContainerEmptyError` when `pop` is called on an empty stack, even if that seems like a fairly innocuous change.

Once again, maintaining this sort of discipline may seem on the surface to be more trouble than it is worth. However, these kinds of assumptions are baked into useful patterns such as the [template method pattern](http://en.wikipedia.org/wiki/Template_method_pattern) and are also key to designing type hierarchies for all sorts of data structures. A good example of these concepts in action can be found in the way Ruby organizes its numeric types. The class hierarchy is shown here, but be sure to check out Ruby's documentation if you want to get a sense of how exactly these classes hang together.

<img src="http://i.imgur.com/ObKrf.jpg" width=800/>

Whether you are designing extension subtypes or restriction subtypes, it is unfortunately easier to get things wrong than it is to get them right, due to all the subtle issues that need to be considered. For that reason, we'll now take a look at a few examples of flawed behavioral subtypes and how to go about fixing them.

### Examples of flawed behavioral subtypes

To test your understanding of behavior subtype compatibility while simultaneously exposing some common pitfalls, I provide the following three flawed examples for you to study. As you read through them, try to figure out what the subtype compatibility problem is and how you might go about solving it.

1) Suppose we want to add an equality operator to the bag structure. A sample operator is provided here for the `Bag` object, which conforms to the following newly specified
feature: "Two bags are considered equal if they have equivalent items and size limits". What problems will we encounter in implementing a bag-compatible equality operator for the `Stack` object? 

```ruby
class Bag
  # other code similar to before

  def ==(other)
    [data.sort, limit] == [other.sort, other.limit]
  end

  protected 
  
  # NOTE: Implementing == is one of the few legitimate uses of 
  # protected methods / attributes
  attr_accessor :data, :limit
end
```

2) Suppose we have two mutable objects, a `Rectangle` and a `Square`, and we wish to implement `Square` as a restriction of `Rectangle`. Given the following implementation of a `Rectangle` object, what problems will be encountered in defining a `Square` object?

```ruby
class Rectangle
  def area
    width * height
  end

  attr_accessor :width, :height
end
```

3) Suppose we have a `PersistentSet` object that delegates all method calls to the `Set` object provided by Ruby's standard library, as shown in the following code. Why is this not a compatible subtype, even though it does not explicitly modify the behavior of any of `Set`'s operations?

```ruby
require "set"
require "pstore"

class PersistentSet 
  def initialize(filename)
    self.store = PStore.new(filename)

    store.transaction { store[:data] ||= Set.new }
  end

  def method_missing(name, *args, &block)
    store.transaction do 
      store[:data].send(name, *args, &block)
    end
  end

  private

  attr_accessor :store
end
```

To avoid spoiling the fun of finding and fixing the defects with these examples yourself, I've hidden my explanation of the [problems](https://gist.github.com/15b50f918c88bccd6eac) and [solutions](https://gist.github.com/3f53d4094759c0508e19) on a pair of gists. Please spend some time on this exercise before reading the spoilers, as you'll learn a lot more that way!

A huge hint is that the first problem is based on an issue discussed in [Liskov's paper](http://www.cs.cmu.edu/~wing/publications/LiskovWing94.pdf) and the second and third problems are discussed in an [article about LSP](http://www.objectmentor.com/resources/articles/lsp.pdf) by Bob Martin. However, please note that their solutions are not exactly the most natural fit for Ruby, so there is still room for some creativity here.

### Behavioral subtyping versus duck typing

Between this article and the topics discussed in [Issue 3.7](http://practicingruby.com/articles/24), this two-part series offers a fairly comprehensive view of disciplined inheritance practices for the Ruby programmer. However, as I hinted toward the beginning of this article, there is the somewhat looser concept of duck typing that deserves a mention if we really want to see the whole picture.

What duck typing and behavioral subtypes have in common is that both concepts rely on what an object can do rather than what exactly it is. They differ in that behavioral subtypes seem to be more about the behavior of an entire object and duck typing is about how a given object behaves within a certain context. Duck typing can be a good deal more flexible than behavioral subtyping in that sense, because typically it involves an object implementing a meaningful response to a single message rather than an entire suite of behaviors. You can find a ton of examples of duck typing in use in Ruby, but perhaps the easiest to spot is the ubiquitous use of the `to_s` method.

By implementing a `to_s` method in our objects, we are able to indicate to Ruby that our object has a meaningful string representation, which can then be used in a wide range of contexts. Among other things, the `to_s` method is automatically called by irb when an `inspect` method is not also provided, called by the `Kernel#puts` method on whatever object you pass to it, and called automatically on the result of any expression executed via string interpolation. Implementing a meaningful `to_s` method is not exactly a form of behavioral subtyping but is still a very useful form of code sharing. [Issue 1.14](http://blog.rubybestpractices.com/posts/gregory/046-issue-14-duck-typing.html) and [Issue 1.15](http://blog.rubybestpractices.com/posts/gregory/047-issue-15-duck-typing-2.html) cover duck typing in great detail, but this single example is enough to point out the merits of this technique and how much simpler it is than the topics discussed in this article.

### Reflections

A true challenge for any practicing Rubyist is finding a balance between the free-wheeling culture of Ruby development and the more rigorous approaches of our predecessors. Disciplined inheritance techniques will make our lives easier, and knowing what a behavioral subtype is and how to design one will surely come in handy on any moderately complex project. However, we should keep our eyes trained on how these issues relate to maintainability, understandability, and changeability rather than obsessing about how they can lead us to mathematically pure designs.

I think there is room for another article on the practical applications of these ideas, in which I might talk about applying some design-by-contract concepts to Ruby programs or how to develop shared unit tests that make it easier to test for compatibility when implementing subtypes. But I don't plan to work on that article immediately, so for now we can sort out those issues via comments on this article. If you have any suggestions for how to tie these ideas back to real problems, or questions on how to apply them to the things you've been working on, please share your thoughts. 
