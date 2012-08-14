Today, I've got a handful of neat examples to share, each which demonstrates an interesting use of duck typing. We'll start by looking a feature built into Ruby's core, and then look at a few examples from other open source Ruby projects.

### Type Coercion, Ruby-style

Many dynamically typed languages that offer both integer and floating point
arithmetic are smart about doing the right thing based on whether or not any
floats are used in a given expression. While I assume that you are already 
familiar with Ruby's behavior, the following example demonstrates what 
I've just described.

```ruby
>> 3/2
=> 1
>> 3/2.0
=> 1.5
```

This is an obvious candidate for implementation level special casing, but since all the primitive numeric types in Ruby are actually objects, Ruby prefers something a bit more flexible and consistent. What actually happens when an arithmetic operation is performed on a Ruby number is that a method called `coerce()` is called to do any necessary type modifications so that the computations work as expected. The irb session shown below demonstrates calling `coerce()` directly on both a `Fixnum` and a `Float`.

```ruby
>> 3.coerce(2)
=> [2, 3]
>> 3.coerce(2.0)
=> [2.0, 3.0]
>> 3.0.coerce(3)
=> [3.0, 3.0]
>> 3.0.coerce(2.0)
=> [2.0, 3.0]
```

Note that `Fixnum#coerce` only returns an array of Float values when its argument is a Float, but that `Float#coerce` always does this conversion. While what is shown above only demonstrates how floating point coercion works, we can actually create our own objects that duck type to Ruby numbers by simply defining a `coerce()` method on them.

To demonstrate this, I have created a partial implementation of a `BinaryInteger` object. A `BinaryInteger` is meant to act similar to Ruby's `Fixnum` objects but display itself to the user in binary notation. Here's an example of how such an object might be used:

```ruby
>> int = BinaryInteger.new(40)
=> 0b101000
>> 2 + int
=> 0b101010
>> 2.5 + int
TypeError: BinaryInteger can't be coerced into Float
	from ./binary_integer.rb:49:in `+'
	from (irb):4
	from :0
```

The following class definition does not quite produce a complete `Numeric` work-alike but it is sufficient for making the previous example work as shown. It also serves to demonstrate that `coerce()` is indeed the magic that ties all of Ruby's arithmetic operations together.

```ruby
class BinaryInteger
  def initialize(value)
    @value = value
  end

  attr_accessor :value

  def integer?
    true
  end

  def +(other)
    a,b = coerce(other) # use our own coerce here
    self.class.new(a.value + b.value)
  end

  def coerce(other)
    raise TypeError unless other.integer? 

    if other.respond_to?(:value)
      [self, other] # no coercion needed
    else
      [self, self.class.new(other)]
    end
  end

  def inspect
    "0b#{@value.to_s(2)}"
  end
end
```

While it can be tricky to puzzle through how `coerce()` should work, since you can't know in advance what the calling object will be, it is a lot more dynamic than enforcing class based typing. Getting in the practice of thinking in terms of the interactions between the objects in your project rather than their static definitions can lead to some very good design insights.

In addition to the `coerce()` method for arithmetic, Ruby uses a whole score of other coercion hooks, including `to_int`, `to_str`, and `to_ary`. These methods are called on the arguments passed to a number of `Fixnum`, `String`, and `Array` methods. The neat thing is that there is no strict requirement that these methods actually return `Fixnum`, `String`, or `Array` objects, as long as they act close enough to the real thing where it counts (i.e. for whatever messages that get sent to them).

We could probably spend all day going through other examples of where Ruby uses duck typing for coercion, for extension points, and tons of other uses. This is especially true when you consider that almost every mixin relies on a form of duck typing. For example, all functionality in `Enumerable` can work with anything that implements a sensible `each()` method. Similarly a suitable `<=>` operator unlocks all that `Comparable` has to offer. In both the core and standard library, you will find plenty of examples of this sort of design.

The key point to take away from these observations is that duck-typed APIs aren't some obscure edge case for the extensibility-obsessed, but instead, something baked into Ruby's philosophy from the ground up. This means that you can and should imitate this style in your own libraries when it makes sense to do so.

We'll now take a look at a pair of examples from the wild, one from my own project (Prawn), and another from Aaron Patterson's Rails 3.1 performance tuning adventures. Both involve the use of duck typing not for the purpose of infinite flexibility, but for addressing practical problems that come up in most moderately complex projects.

### Duck typing to avoid scope creep

The first example of duck typing in actual Ruby projects that I want to share is actually quite similar to the contrived `read_data()` example I shared on Tuesday. Today, rather than showing you the usage code first, I want you to take a look at the implementation and try to spot the usage of duck typing and guess at what it gains us before reading on.

```ruby
def image(file, options={})
  Prawn.verify_options [:at, :position, :vposition, :height,
                        :width, :scale, :fit], options

  if file.respond_to?(:read)
    image_content = file.read
  else
    raise ArgumentError, "#{file} not found" unless File.file?(file)
    image_content = File.binread(file)
  end

  # additional implementation details omitted.
end

# FULL IMPLEMENTATION OF image() at:
# https://github.com/sandal/prawn/blob/master/lib/prawn/images.rb#L65
```

If you guessed this code is used to make it so that the `image()` method can be called with either a file name or a file handle, you had the right idea. It does all of the things we discussed yesterday, allowing the use of this code with `StringIO`, `Tempfile`, any mock object that implements a `read()` method, etc. But the really interesting use case is the one that we actually wrote this feature for, shown below.

```ruby
require "open-uri"

Prawn::Document.generate("remote_images.pdf") do
  image open("http://prawn.majesticseacreature.com/images/prawn.png")
end
```

Through the use of `open-uri`, our duck-typed image method provides a nice way
of rendering remote content! While this might not have been an easy feature to
guess without knowing a bit about Prawn, it represents the elegant compromise that such an implementation affords us. Adding support for remote images was something that our users often asked for, but we wanted to avoid giving people the impression that Prawn was web-aware, and didn't want to support a special case for this sort of logic, as it'd require either an API change or an ugly hack to determine whether the provided string was either a URI or a file name.

The approach of accepting anything with a `read()` method combined with Ruby's standard library `open-uri` made for something that is easy to document and easy for our users to remember. While a simple hack, I was very satisfied with how this design turned out because it seemed to mostly eliminate the problem for our users while simultaneously avoiding some overly complex implementation code that might be brittle and hard to test.

These sort of tough design decisions are certainly not unique to Prawn, so we can now turn our eyes to Aaron Patterson's performance optimization work on Rails 3.1.

### Duck typing for performance tuning

One area Aaron Patterson found was a hotspot for many Rails apps are `ActiveRecord` scopes, which allow the users to create custom filters. For example, consider the following example which filters by email address.

```ruby
class Comment < ActiveRecord::Base
  scope :with_email, lambda { |email|
    where(:email => email)
  }
end

# Above code provides functionality shown below
User.with_email("gregory.t.brown@gmail.com").count #=> 1
```

The block syntax is nice and clean for simple things, but can get a bit unwieldy for complex logic. For example, if we wanted to throw in validations for the entered email addresses, our block would end up getting a bit ugly unless we implemented some private class methods to help out. If you're thinking that private class methods sound weird and might be a bit of a code smell, they are, and that's one indication that this API needs to be more flexible than what it is.

That said, Aaron was on a performance tuning mission, not an API overhaul. The
problem he found with the API was initially not an aesthetic one but an
implementation detail: Executing code stored in a `Proc` object is considerably
more computationally expensive than an ordinary method call. While this isn't
likely to be a bottleneck in ordinary situations, it is common for high traffic
Rails applications to really hammer on their scopes, since they're used for
filtering the data that is presented to users. The key insight Aaron had was
that making some other object quack like a `Proc` is as easy as implementing 
a `call()` method.

Shown below is the one line patch that changes the behavior of `scope()` to
allow the use of any object that implements a meaningful `call()` method:

```ruby
# BEFORE
options = filter.is_a?(Proc) ? filter.call(*args) : filter

# AFTER
options = filter.respond_to?(:call) ? filter.call(*args) : filter
```

With this nearly microscopic change, we can write a faster `with_email()` scope that also leaves room for complex logic such as validations in its own neatly defined namespace. The following definition is functionally equivalent to our original code that passes a `Proc` to `scope()`, but has a lot more potential for future growth.

```ruby
class EmailFilter 
  def initialize(model_class)
    @model_class = model_class
  end

  def call(email)
    validate_address(email)
    @model_class.where(:email => email)
  end

  private

  def validate_address(email)
    # do some validation magic here
  end
end

class User < ActiveRecord::Base
  scope :with_email, EmailFilter.new(self)
end
```

The nice thing about this patch is that nothing is lost by doing things this way. Often times, when moving from explicit class checking to behavior based checks, the only overhead is that debugging can be a bit more complicated since there is no easy way to verify that an object implementing `call()` actually does so in a sensible way. However, with adequate unit tests and decent documentation, this kind of fuzziness is rarely a big enough problem in practical applications to outweigh the benefits that come along with utilizing this technique.

Aside from the superficial improvements that come from converting `Proc` calls
to method calls, the general approach of writing duck typed interfaces tends to increase the potential for further performance improvements. When code is written to explicitly avoid assuming too much about how objects are implemented, it is easy to swap out objects that are more performant in edge cases, or implement aggressive caching where appropriate. While it may seem counterintuitive, the same dynamic nature that makes Ruby slow at the implementation level makes a wide range of algorithmic improvements possible. We unfortunately won't be exploring this topic today, but it would be a good topic for a future issue.

  
> **NOTE:** This article has also been published on the Ruby Best Practices blog. There [may be additional commentary](http://blog.rubybestpractices.com/posts/gregory/047-issue-15-duck-typing-2.html#disqus_thread) 
over there worth taking a look at.
