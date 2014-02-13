*This article was written by Avdi Grimm. Avdi is a [Ruby Rogue][rogue], a
consulting pair programmer, and the head chef at [RubyTapas][tapas]. He writes
about software development at [Virtuous Code.][virtuous]*

When you think of the term *object-oriented programming*, one of the
first associated words that springs to mind is probably *classes*. For
most of its history, the OOP paradigm has been almost inextricably
linked with the idea of classes. Classes serve as *object factories*:
they hold the blueprint for new objects, and can be called upon to
manufacture as many as needed. Each object, or *instance*, has its
state, but each derives its behavior from the class. Classes, in turn,
share behavior through inheritance. In most OO programs, the class
structure is the primary organizing principle.

Even though classes have gone hand-in-hand with OOP for decades, they
aren't the only way to build families of objects with shared behavior.
The most common alternative to *class-based* programming is
*prototype-based* programming. Languages that use prototypes rather than
classes include [Self][self], [Io][io], and (most well known of all) JavaScript.

Ruby comes from the class-based school of OO language design. But it's
flexible enough that with a little cleverness, we can experiment with
prototype-style coding. In this article that's just what we'll do.

[self]: http://en.wikipedia.org/wiki/Self_(programming_language
[io]: http://en.wikipedia.org/wiki/Io_(programming_language)
[rogue]: http://rubyrogues.com/
[tapas]: http://devblog.avdi.org/rubytapas/
[virtuous]: http://devblog.avdi.org/

## Getting started

So how do we write OO programs without classes? Let's explore this
question in Ruby. We'll use the example of a text-adventure game in the
style of "[Colossal Cave
Adventure](http://en.wikipedia.org/wiki/Colossal_Cave_Adventure)". This
is one of my favorite programming examples for object-oriented systems,
since it involves modeling a virtual world of interacting objects,
including characters, items, and interconnected rooms.

We open up an interactive Ruby session, and start typing. We begin with
an `adventurer` object. This object will serve as our avatar in the
game's world, translating our commands into interactions between
objects:

```ruby
adventurer = Object.new
```

The first ability we give to our adventurer is the ability to look at
its surroundings. The `look` command will cause the adventurer to output
a description of its current location:

```ruby
class << adventurer
  attr_accessor :location

  def look
    puts location.description
  end
end
```

Then we add a starting location, called `end_of_road`, and put the
adventurer in that location:

```ruby
end_of_road = Object.new
def end_of_road.description
  <<END
You are standing at the end of a road before a small brick building.
Around you is a forest.  A small stream flows out of the building and
down a gully.
END
end

adventurer.location = end_of_road
```

Now we can tell our adventurer to take a look around:

```console
> adventurer.look

You are standing at the end of a road before a small brick building.
Around you is a forest.  A small stream flows out of the building and
down a gully.
```

## Adding some conveniences

So far we've created an adventurer and a starting room without any kind
of `Adventurer` or `Room` classes. This adventure is getting off to a
good start! Although, if we're going to be creating a lot of these
objects we'd like for the process to be a little less verbose. We decide
to take a step back and build some syntax sugar before moving onward.

We start with an `ObjectBuilder` helper class. Yes, this is a class, when
we are supposed to be using only prototypes. However, Ruby doesn't offer
a lot of support for prototype-based programming out of the box. So we
have to build our tools with the class-oriented materials at hand. This
is intended to be behind-the-scenes support code. In other words, pay no
attention to the man behind the green curtain!

```ruby
class ObjectBuilder
  def initialize(object)
    @object = object
  end

  def respond_to_missing?(missing_method, include_private=false)
    missing_method =~ /=\z/
  end

  def method_missing(missing_method, *args, &block)
    if respond_to_missing?(missing_method)
      method_name = missing_method.to_s.sub(/=\z/, '')
      value       = args.first
      ivar_name   = "@#{method_name}"
     if value.is_a?(Proc)
        define_code_method(method_name, ivar_name, value)
      else
        define_value_method(method_name, ivar_name, value)
      end
    else
      super
    end
  end

  def define_value_method(method_name, ivar_name, value)
    @object.instance_variable_set(ivar_name, value)
    @object.define_singleton_method(method_name) do
      instance_variable_get(ivar_name)
    end
  end

  def define_code_method(method_name, ivar_name, implementation)
    @object.instance_variable_set(ivar_name, implementation)
    @object.define_singleton_method(method_name) do |*args|
      instance_exec(*args, &instance_variable_get(ivar_name))
    end
  end
end
```

There's a lot going on in this class. Going over it line-by-line might
be interesting in its own right, but it wouldn't advance our
understanding of prototype-based programming all that much. Suffice to
say for now that this class can help us add new attributes and methods
to a singleton object using a concise assignment-style syntax. This will
make more sense when we start to make use of it.

We add another bit of syntax sugar: a global method named `Object` (not
to be confused with the class of the same name):

```ruby
def Object(&definition)
  obj = Object.new
  obj.singleton_class.instance_exec(ObjectBuilder.new(obj), &definition)
  obj
end
```

This method takes a block, instantiates a new object, and evaluates the
block in the context of the object's singleton class, passing an
`ObjectBuilder` as a block argument. Then it returns the new object.

Now we recreate our adventurer using this new helper:

```ruby
adventurer = Object { |o|
  o.location = end_of_road

  attr_writer :location

  o.look = ->(*args) {
    puts location.description
  }
}
```

The combination of the `Object` factory method and the `ObjectBuilder`
gives us a convenient, powerful notation for creating new ad-hoc
objects. We can create attribute reader methods and assign the value of
the attribute all at once:

```ruby
o.location = end_of_road
```

We can use standard Ruby class-level code:

```ruby
attr_writer :location
```

And finally we can define new methods by assigning a lambda to an 
attribute:

```ruby
o.look = ->(*args) { puts location.description }
```

We've deliberately avoided defining methods using `def` or
`define_method`. We'll get into the reasons for that later on.

Before we move on, let's take a moment to make sure our shiny new adventurer still works the
same as before:

```console
> adventurer.look

You are standing at the end of a road before a small brick building.
Around you is a forest.  A small stream flows out of the building and
down a gully.
```

## Moving around

It's time to let our adventurer object stretch its legs a bit.
We want to give it the ability to move from location to location. First,
we make a small modification to our `Object()` method:

```ruby
def Object(object=nil, &definition)
  obj = object || Object.new
  obj.singleton_class.instance_exec(ObjectBuilder.new(obj), &definition)
  obj
end
```

Now along with creating new objects, `Object()` can also augment an
existing object which is passed in as an argument.

We pass the `adventurer` to `Object()`, and add a new `#go` method. This
method will take a direction (like `:east`), and attempt to move to the
new location using the `exits` association on its current location:

```ruby
Object(adventurer) { |o|
  o.go = ->(direction){
    if(destination = location.exits[direction])
      self.location = destination
      puts location.description
    else
      puts "You can't go that way"
    end
  }
}
```

We add a destination room to the system:

```ruby
wellhouse = Object { |o|
  o.description = <<END
You are inside a small building, a wellhouse for a large spring.
END
}
```

Then we add an `exits` Hash to `end_of_road`, with an entry saying that
the `wellhouse` is to the `:north` of it:

```ruby
Object(end_of_road) { |o| o.exits = {north: wellhouse} }
```

With that done, we are now ready to set off on our journey!

```console
> adventurer.go(:north)

You are inside a small building, a wellhouse for a large spring.
```

## Cloning prototypes

We try to go north again, expecting to see the admonition "You can't go
that way" as we bump into the wall:

```console
> adventurer.go(:north)
```

Instead, we get an exception:

```console
-:82:in `block (2 levels) in <main>': undefined method `exits' for 
#<Object:0x0000000434d768> (NoMethodError)
        from -:56:in `instance_exec'
        from -:56:in `block (2 levels) in define_code_method'
        from -:100:in `<main>'
```

This is because we never got around to adding an `exits` Hash to
`wellhouse`. We could go ahead and do that now. But as we think about
it, we realize that now that our adventurer is capable of travel, it
would make sense if all rooms started out with an empty `exits` Hash,
instead of us having to add it manually every time.

Toward that end, we create a *prototypical room*:

```ruby
room = Object { |o| o.exits = {} }
```

We then experiment with creating a new `wellhouse`, this one based on
the `room` prototype. We do this by simply cloning the `room` object. We
use `#clone` rather than `#dup` because `#clone` copies singleton class
methods:

```ruby
new_wellhouse = room.clone

new_wellhouse.exits[:south] = end_of_road
```

We quickly uncover a problem with this naive cloning technique. Because
Ruby's `#clone` (as well as `#dup`) are *shallow copies*, `room` and
`new_wellhouse` now share the same `exits`:

```ruby
require 'pp'

puts "new_wellhouse exits:"
pp new_wellhouse.exits
puts "room exits:"
pp room.exits
```

```console
new_wellhouse exits:
{:south=>
  #<Object:0x0000000482c8d8
   @exits=
    {:north=>
      #<Object:0x0000000482bcd0
       @description=
        "You are inside a small building, a wellhouse for a large spring.\n">}>}
room exits:
{:south=>
  #<Object:0x0000000482c8d8
   @exits=
    {:north=>
      #<Object:0x0000000482bcd0
       @description=
        "You are inside a small building, a wellhouse for a large spring.\n">}>}
```

To fix this, we could possibly customize the way Ruby does cloning by overriding
the [Object#initialize_clone](http://jonathanleighton.com/articles/2011/initialize_clone-initialize_dup-and-initialize_copy-in-ruby/)
method, but that would be an invasive change with broad reaching effects.
Because extending core objects is a bit safer than modifying them, we opt to
define our own `Object#copy` method which does a one-level-deep copying of
instance variables:

```ruby
class Object
  def copy
    prototype = clone

    instance_variables.each do |ivar_name|
      prototype.instance_variable_set(
        ivar_name,
        instance_variable_get(ivar_name).clone)
    end

    prototype
  end
end
```

Then we recreate `room` and `new_wellhouse`, and confirm that they no
longer share exits:

```ruby
room = Object { |o| o.exits = {} }

# Use the newly defined Object#copy here instead of Object#clone
new_wellhouse = room.copy

new_wellhouse.exits[:south] = end_of_road

puts "new_wellhouse exits:"
pp new_wellhouse.exits
puts "room exits:"
pp room.exits
```

```console
new_wellhouse exits:
{:south=>
  #<Object:0x00000002ea85d8
   @exits=
    {:north=>
      #<Object:0x00000002ea79d0
       @description=
        "You are inside a small building, a wellhouse for a large spring.\n">}>}
room exits:
{}
```

Cloning a prototypical object in order to create new
objects is the most basic form of prototype-based programming. In fact,
the "Kevo" research language (I'd link to it, but all the information
about it seems to have fallen off the Internet) used copying as the sole
way to share behavior between objects.

## Building dynamic prototypes

There are drawbacks to copying, however. It's a very static way to share
behavior between objects. Clones of `room` only share the behavior which
was defined at the time of the copy. If we were to modify `room`, we'd
have to recreate the `new_wellhouse` object once again in order to take
advantage of any new methods added to it.

Cloning also implies single inheritance. An object can only be a clone
of one "parent" object.

Finally, we also can't add any new behavior to our existing `wellhouse`
object this way. We'd have to throw away our program's state and rebuild
it, this time cloning our `end_of_road` and `wellhouse` objects from
`room`.

In Ruby, we're used to being able to make changes to a live session and
see how they play out. Thus far, we've done this all in a live
interpreter session. It seems a shame to have to lose our state and
start again. So we decide to find out if we can come up with a more
dynamic form of prototypical inheritance than plain copying.

We start by adding a helper method called `#implementation_of` to
Object. Given a method name that the object supports, it will return a
`Proc` object containing the code of that method. We make it aware of
the style of method definition used in `ObjectBuilder`, where the
implementation `Procs` of new methods were stored in instance variables
named for the methods:

```ruby
class Object
  def implementation_of(method_name)
    if respond_to?(method_name)
      implementation = instance_variable_get("@#{method_name}")
      if implementation.is_a?(Proc)
        implementation
      elsif instance_variable_defined?("@#{method_name}")
        # Assume the method is a reader
        ->{ instance_variable_get("@#{method_name}") }
      else
        method(method_name).to_proc
      end
    end
  end
end
```

We then define a new kind of `Module`, called `Prototype`:

```ruby
class Prototype < Module
  def initialize(target)
    @target = target
    super() do
      define_method(:respond_to_missing?) do |missing_method, include_private|
        target.respond_to?(missing_method)
      end

      define_method(:method_missing) do |missing_method, *args, &block|
        if target.respond_to?(missing_method)
          implementation = target.implementation_of(missing_method)
          instance_exec(*args, &implementation)
        else
          super(missing_method, *args, &block)
        end
      end
    end
  end
end
```

A `Prototype` is instantiated with a prototypical object. When a
`Prototype` instance is added to an object using `#extend`, it makes the
methods of the prototype available to the extended object. It does this
by implementing `#method_missing?` (and the associated
`#respond_to_missing?`). When a message is sent to the extended object
that matches a method on the prototype object, the `Prototype` grabs the
implementation `Proc` from the prototype. Then it uses `#instance_exec`
to evaluate the `prototype`'s method in the context of the extended
object. In effect, the extended object "borrows" a method from the
prototype object for just long enough to execute it.

Note that this is different from delegation. In delegation, one object
hands off a message to be handled by another object. If object `a`
delegates a `#foo` message to object `b`, using, for instance, Ruby's
`forwardable` library, `self` in that method will be object `b`. This is
easily demonstrated:

```ruby
require 'forwardable'

class A
  extend Forwardable
  attr_accessor :b
  def_delegator :b, :foo
end

class B
  def foo
    puts "executing #foo in #{self}"
  end
end

a = A.new
a.b = B.new
a.foo
# >> executing #foo in #<B:0x00000003295e20>
```

But delegation is not what we want. We want to execute the methods from
prototypes as if they had been defined on the inheriting object. We want
this because we want them to work with the instance variables of the
inheriting object. If we send `wellhouse.exits`, we want the reader
method to show us the content of `wellhouse`'s `@exits` instance
variable, not `room`'s instance variable.

Remember how, in `ObjectBuilder`, we stored the implementations of
methods as `Procs` in instance variables rather than defining them
directly as methods? This need to call prototype methods on the
inheriting object is the reason for that. In Ruby, it is not possible to
execute a method from class A on an instance of unrelated class B. Since
in this program we are using the singleton classes of objects to define
all of their methods, Ruby considers all of our objects as belonging to
different classes for the purposes of method binding. We can see this if
we try to rebind a method from `room` onto `wellhouse` and then call it:

```ruby
room.method(:exits).unbind.bind(wellhouse)
```

```console
-:115:in `bind': singleton method called for a different object (TypeError)
        from -:115:in `<main>'
```

By storing the implementation of methods as raw `Procs`, without any
association to a specific class, we are able to take the implementations
and `instance_exec` them in other contexts.

The last change we make to support dynamic prototype inheritance is to
add a new `#prototype` method to our `ObjectBuilder`:

```ruby
class ObjectBuilder
  def prototype(proto)
    # Leave method implementations on the proto object
    ivars = proto.instance_variables.reject{ |ivar_name|
      proto.respond_to?(ivar_name.to_s[1..-1]) &&
      proto.instance_variable_get(ivar_name).is_a?(Proc)
    }
    ivars.each do |ivar_name|
      unless @object.instance_variable_defined?(ivar_name)
        @object.instance_variable_set(
          ivar_name,
          proto.instance_variable_get(ivar_name).dup)
      end
    end
    @object.extend(Prototype.new(proto))
  end
end
```

This method does two things:

1.  It copies instance variables from a prototype object to the object
    being built.
2.  It extends the object being built with a `Prototype` module
    referencing the prototype object.

We can now use all of this new machinery to dynamically add `room` as a
prototype of `wellhouse`. We are then able to set the south exit to
point back to `end_of_road`, using the `exits` association that
`wellhouse` now inherits from `room`:

```ruby
Object(wellhouse) { |o| o.prototype room }

wellhouse.exits[:south] = end_of_road

adventurer.location = wellhouse
```

Then we can move around again to make sure things are working as expected:

```ruby
puts "* trying to go north from wellhouse"
adventurer.go(:north)

puts "* going back south"
adventurer.go(:south)
```

```console
* trying to go north from wellhouse
You can't go that way
* going back south
You are standing at the end of a road before a small brick building.
Around you is a forest.  A small stream flows out of the building and
down a gully.
```

## Carrying items around

We now have some powerful tools at our disposal for composing objects
from prototypes. We quickly proceed to implement the ability to pick up
and drop items to our game. We start by creating a prototypical
"container" object, which has an array of items and the ability to
transfer an item from itself to another container:

```ruby
container = Object { |o|
  o.items = []
  o.transfer_item = ->(item, recipient) {
    recipient.items << items.delete(item)
  }
}  
```

We then make the `adventurer` a container, and add some commands for
taking items, dropping items, and listing the adventurer's current
inventory:

```ruby
Object(adventurer) {|o|
  o.prototype container

  o.look = -> {
    puts location.description
    location.items.each do |item|
      puts "There is #{item} here."
    end
  }

  o.take = ->(item_name) {
    item = location.items.detect{|item| item.include?(item_name) }
    if item
      location.transfer_item(item, self)
      puts "You take #{item}."
    else
      puts "You see no #{item_name} here"
    end
  }

  o.drop = ->(item_name) {
    item = items.detect{|item| item.include?(item_name) }
    if item
      transfer_item(item, location)
      puts "You drop #{item}."
    else
      puts "You are not carrying #{item_name}"
    end
  }

  o.inventory = -> {
    items.each do |item|
      puts "You have #{item}"
    end
  }
}
```

For convenience, we've implemented `#take` and `#drop` so that they can
accept any substring of the intended object's name.

Next we make `wellhouse` a container, and add a list of starting items
to it:

```ruby
Object(wellhouse) { |o|
  o.prototype container
  o.items = [
    "a shiny brass lamp",
    "some food",
    "a bottle of water"
  ]
  o.exits = {south: end_of_road}
}
```

As you may recall, `wellhouse` already has a prototype: `room`. But this
is not a problem. One of the advantages of our dynamic prototyping
system is that objects may have any number of prototypes. Since
prototyping is implemented using specialized modules, when an object is
sent a message that it can't handle itself, Ruby will keep searching up an
object's ancestor chain, from one `Prototype` to the next, looking for a
matching method. (This also puts us one-up on JavaScript's
single-inheritance prototype system!)

Finally, we make `end_of_road` a container:

```ruby
Object(end_of_road) { |o| o.prototype(container) }
```

We then proceed to tell our adventurer to pick up a bottle of water from
the wellhouse, and put it down at the end of the road:

```console
> adventurer.go(:north)
You are inside a small building, a wellhouse for a large spring.
> adventurer.take("water")
You take a bottle of water.
> adventurer.inventory
You have a bottle of water
> adventurer.look
You are inside a small building, a wellhouse for a large spring.
There is a shiny brass lamp here.
There is some food here.
> adventurer.go(:south)
You are standing at the end of a road before a small brick building.
Around you is a forest.  A small stream flows out of the building and
down a gully.
> adventurer.drop("water")
You drop a bottle of water.
> adventurer.look
You are standing at the end of a road before a small brick building.
Around you is a forest.  A small stream flows out of the building and
down a gully.
There is a bottle of water here.
```

And with that, we now have a small but functional system which allows us to move
around the game world and interact with it.

## Reflections

We've written the beginnings of a text adventure game in a
prototype-based style. Now, let's take a step back and talk about what
the point of this exercise was.

There is a strong argument to be made that prototype-based inheritance
more closely maps to how humans normally think through problems than
does class-based inheritance. Quoting the paper "[Classes vs.
Prototypes: Some Philosophical and Historical
Observations](http://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.56.4713)":

> A typical argument in favor of prototypes is that people seem to be a
> lot better at dealing with specific examples first, then generalizing
> from them, than they are at absorbing general abstract principles
> first and later applying them in particular cases, ... the ability to
> modify and evolve objects at the level of individual objects reduces
> the need for a priori classification and encourages a more iterative
> programming and design style.

As we built up our adventure game, we immediately added concrete objects
to the system as soon as we thought them up. We added an `adventurer`,
and then an `end_of_road` for the adventurer to start out in. Then
later, as we added more objects, we generalized out commonalities into
objects like `room` and `container`. Our program design emerged
completely organically, and our abstractions emerged as soon as we
needed them, but no sooner. This kind of emergent, organic design
process is one of the ideals of agile software development, and
prototype-based systems seem to encourage it.

Of course, the way we jammed prototypes into a class-based language here
is a horrendous hack: please don't use it in a production system!
But the experience of writing code in a prototyped style can teach
us a lot. We can use what we've learned to influence our daily
coding. We might prototype (heh) a system's design by writing one-off
objects at first, adding methods to their singleton classes. Then, as
patterns of interaction emerge, we might capture the design using
classes. Prototypes can also teach us to do more with delegation and
composition, building families of collaborating objects rather than
hierarchies of related behavior.

Now that we've reached the end of our journey, I hope you've found 
this trip through prototype-land illuminating and thought-provoking. 
I'm still a relative newb to this way of thinking, so if you
have anything to add‚ i.e. other benefits of using prototypes; subtle gotchas;
experiences from prototype-based languages, or alternative implementations of
any of the code above, please don't hesitate to pipe up in the comments. Also,
if you want clarifications about any of the gnarly metaprogramming I used to
bash Ruby into a semblance of a prototype-based language, feel free to ask --
but I can't guarantee that the answers will make any more sense than the
code :-)

> **NOTE:** If you had fun reading this article, you may also enjoy reading Advi's 
> blog post on the [Prototype Pattern](http://devblog.avdi.org/?p=5560), a design pattern that takes 
> ideas from prototype-based programming and applies them to class-based
> modeling. That post started as a section of this article that gained a life
> of its own.
