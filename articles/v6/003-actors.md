> This issue was a collaboration with [Alberto Fernández Capel][afcapel], a Ruby developer
from Spain. Although it has been through many revisions since
we started, Alberto's ideas, code, and explanations provided
an excellent starting point that lead us to publish this article.

Conventional wisdom says that concurrent programming is hard, especially in 
Ruby. This basic assumption is what lead many Rubyists to take an interest
in languages like Erlang and Scala -- their baked in support for 
the [actor model][actors] is meant to make concurrent systems 
much easier for everyday programmers to implement and understand.

But do you really need to look outside of Ruby to find concurrency primitives
that can make your work easier? The answer to that question probably 
depends on the levels of concurrency and availability that you require, but
things have definitely been shaping up in recent years. In particular, 
the [Celluloid][celluloid] framework has brought us a convenient and clean way to implement
actor-based concurrent systems in Ruby.

In order to appreciate what Celluloid can do for you, you first need to
understand what the actor model is, and what benefits it offers over the
traditional approach of directly using threads and locks for concurrent 
programming. In this article, we'll try to shed some light on those points by
solving a classic concurrency puzzle in three ways: Using Ruby's built-in
primitives (threads and mutex locks), using the Celluloid framework, and using a
minimal implementation of the actor model that we'll build from scratch.

By the end of this article, you certainly won't be a concurrency expert
if you aren't already, but you'll have a nice head start on some
basic concepts that will help you decide how to tackle concurrent programming
within your own projects. Let's begin!

## The Dining Philosophers Problem

The [Dining Philosophers][philosophers] problem was formulated by Edsger Djisktra in 1965 to
illustrate the kind of issues we can find when multiple processes compete to
gain access to exclusive resources.

In this problem, five philosophers meet to have dinner. They sit at a round
table and each one has a bowl of rice in front of them. There are also five
chopsticks, one between each philosopher. The philosophers spent their time
thinking about _The Meaning of Life_. Whenever they get
hungry, they try to eat. But a philosopher needs a chopstick in each
hand in order to grab the rice. If any other
philosopher has already taken one of those chopsticks, the hungry
philosopher will wait until that chopstick is available.

This problem is interesting because if it is not properly solved it can easily
lead to deadlock issues. We'll take a look at those issues soon, but first let's
convert this problem domain into a few basic Ruby objects.

### Modeling the table and its chopsticks

All three of the solutions we'll discuss in this article rely on a `Chopstick`
class and a `Table` class. The definitions of both classes are shown below:

```ruby
class Chopstick
  def initialize
    @mutex = Mutex.new
  end

  def take
    @mutex.lock
  end

  def drop
    @mutex.unlock

  rescue ThreadError
    puts "Trying to drop a chopstick not acquired"
  end

  def in_use?
    @mutex.locked?
  end
end

class Table
  def initialize(num_seats)
    @chopsticks  = num_seats.times.map { Chopstick.new }
  end

  def left_chopstick_at(position)
    index = (position - 1) % @chopsticks.size
    @chopsticks[index]
  end

  def right_chopstick_at(position)
    index = (position + 1) % @chopsticks.size
    @chopsticks[index]
  end

  def chopsticks_in_use
    @chopsticks.select { |f| f.in_use? }.size
  end
end
```

The `Chopstick` class is just a thin wrapper around a regular Ruby mutex 
that will ensure that two philosophers can not grab the same chopstick 
at the same time. The `Table` class deals with the geometry of the problem; 
it knows where each seat is at the table, which chopstick is to the left 
or to the right of that seat, and how many chopsticks are currently in use.

Now that you've seen the basic domain objects that model this problem, we'll
look at different ways of implementing the behavior of the philosophers. 
We'll start with what *doesn't* work.

## A solution that leads to deadlocks

The `Philosopher` class shown below would seem to be the most straightforward
solution to this problem, but has a fatal flaw that prevents it from being
thread safe. Can you spot it?

```ruby
class Philosopher
  def initialize(name)
    @name = name
  end

  def dine(table, position)
    @left_chopstick  = table.left_chopstick_at(position)
    @right_chopstick = table.right_chopstick_at(position)

    loop do
      think
      eat
    end
  end

  def think
    puts "#{@name} is thinking"
  end

  def eat
    take_chopsticks

    puts "#{@name} is eating."

    drop_chopsticks
  end

  def take_chopsticks
    @left_chopstick.take
    @right_chopstick.take
  end

  def drop_chopsticks
    @left_chopstick.drop
    @right_chopstick.drop
  end
end
```

If you're still scratching your head, consider what happens when each
philosopher object is given its own thread, and all the philosophers attempt to
eat at the same time. 

In this naive implementation, it is
possible to reach a state in which every philosopher picks up their left-hand
chopstick, leaving no chopsticks on the table. In that scenario, every
philosopher would simply wait forever for their right-hand chopstick to 
become available -- resulting in a deadlock. You can reproduce the problem
by running the following code:

```ruby
names = %w{Heraclitus Aristotle Epictetus Schopenhauer Popper}

philosophers = names.map { |name| Philosopher.new(name) }
table        = Table.new(philosophers.size)

threads = philosophers.map.with_index do |philosopher, i|
  Thread.new { philosopher.dine(table, i) }
end

threads.each(&:join)
sleep
```

Ruby is smart enough to inform you of what went wrong, so you should end up
seeing a backtrace that looks something like this:

```console
Aristotle is thinking
Popper is eating.
Popper is thinking
Epictetus is eating.
Epictetus is thinking
Heraclitus is eating.
Heraclitus is thinking
Schopenhauer is eating.
Schopenhauer is thinking

dining_philosophers_uncoordinated.rb:79:in `join': deadlock detected (fatal)
  from dining_philosophers_uncoordinated.rb:79:in `each'
  from dining_philosophers_uncoordinated.rb:79:in `<main>
```

In many situations, the most simple solution tends to be the best one, but this
is obviously not one of those cases. Since we've learned the hard way that the
philosophers cannot be safely left to their own devices, we'll need to do more
to make sure their behaviors remain coordinated.

### A coordinated mutex-based solution

One easy solution to this issue is introduce a `Waiter` object into the mix. In this
model, the philosopher must ask the waiter before eating. If the number of chopsticks
in use is four or more, the waiter will make the philosopher wait until someone
finishes eating. This will ensure that at least one philosopher will be able to eat 
at any time, avoiding the deadlock condition.

There's still a catch, though. From the moment the waiter checks the number of chopstick
in use until the next philosopher starts to eat we have a critical region in our
program: If we let two concurrent threads execute that code at the same time there
is still a chance of a deadlock. For example, suppose the waiter checks the number of
chopsticks used and see it is 3. At that moment, the scheduler yields control to
another philosopher who is just picking the chopstick. When the execution flow
comes back to the original thread, it will allow the original philosopher to
eat, even if there may be more than four chopsticks already in use.

To avoid this situation we need to protect the critical region with a mutex, as
shown below:


```ruby
class Waiter
  def initialize(capacity)
    @capacity = capacity
    @mutex    = Mutex.new
  end

  def serve(table, philosopher)
    @mutex.synchronize do
      sleep(rand) while table.chopsticks_in_use >= @capacity 
      philosopher.take_chopsticks
    end

    philosopher.eat
  end
end
```

Introducing the `Waiter` object requires us to make some minor changes to our
`Philosopher` object, but they are fairly straightforward: 

```ruby
class Philosopher

  # ... all omitted code same as before

  def dine(table, position, waiter)
    @left_chopstick  = table.left_chopstick_at(position)
    @right_chopstick = table.right_chopstick_at(position)

    loop do
      think

      # instead of calling eat() directly, make a request to the waiter 
      waiter.serve(table, self)
    end
  end

  def eat
    # removed take_chopsticks call, as that's now handled by the waiter

    puts "#{@name} is eating."

    drop_chopsticks
  end
end
```

The runner code also needs minor tweaks, but is mostly similar to what
you saw earlier:

```ruby
names = %w{Heraclitus Aristotle Epictetus Schopenhauer Popper}

philosophers = names.map { |name| Philosopher.new(name) }

table  = Table.new(philosophers.size)
waiter = Waiter.new(philosophers.size - 1)

threads = philosophers.map.with_index do |philosopher, i|
  Thread.new { philosopher.dine(table, i, waiter) }
end

threads.each(&:join)
sleep
```

This approach is reasonable and solves the deadlock issue, but using mutexes 
to synchronize code requires some low level thinking. Even in this simple 
problem, there were several gotchas to consider. As programs get more
complicated, it becomes really difficult to keep track of critical regions 
while ensuring that the code behaves properly when accessing them.

The actor model is meant to provide a more systematic and natural way of 
sharing data between threads. We'll now take a look at an actor-based 
solution to this problem so that we can see how it compares to this 
mutex-based approach.

## An actor-based solution using Celluloid

We'll now rework our `Philosopher` and `Waiter` classes to make use of 
Celluloid. Much of the code will remain the same, but some important
details will change. The full class definitions are shown below to preserve
context, but the changed portions are marked with comments.

We'll spend the rest of the article explaining the inner workings 
of this code, so don't worry about understanding every last detail. Instead,
just try to get a basic idea of what's going on here:

```ruby
class Philosopher
  include Celluloid

  def initialize(name)
    @name = name
  end

  # Switching to the actor model requires us get rid of our
  # more procedural event loop in favor of a message-oriented
  # approach using recursion. The call to think() eventually
  # leads to a call to eat(), which in turn calls back to think(),
  # completing the loop.

  def dine(table, position, waiter)
    @waiter = waiter

    @left_chopstick  = table.left_chopstick_at(position)
    @right_chopstick = table.right_chopstick_at(position)

    think
  end

  def think
    puts "#{@name} is thinking."
    sleep(rand)

    # Asynchronously notifies the waiter object that
    # the philosophor is ready to eat

    @waiter.async.request_to_eat(Actor.current)
  end

  def eat
    take_chopsticks

    puts "#{@name} is eating."
    sleep(rand)

    drop_chopsticks

    # Asynchronously notifies the waiter
    # that the philosopher has finished eating

    @waiter.async.done_eating(Actor.current)

    think
  end

  def take_chopsticks
    @left_chopstick.take
    @right_chopstick.take
  end

  def drop_chopsticks
    @left_chopstick.drop
    @right_chopstick.drop
  end

  # This code is necessary in order for Celluloid to shut down cleanly
  def finalize
    drop_chopsticks
  end
end


class Waiter
  include Celluloid

  def initialize
    @eating   = []
  end

  # because synchronized data access is ensured
  # by the actor model, this code is much more
  # simple than its mutex-based counterpart. However,
  # this approach requires two methods
  # (one to start and one to stop the eating process),
  # where the previous approach used a single serve() method.

  def request_to_eat(philosopher)
    return if @eating.include?(philosopher)

    @eating << philosopher
    philosopher.async.eat
  end

  def done_eating(philosopher)
    @eating.delete(philosopher)
  end
end
```

The runner code is similar to before, with only some very minor changes:

```ruby
names = %w{Heraclitus Aristotle Epictetus Schopenhauer Popper}

philosophers = names.map { |name| Philosopher.new(name) }

waiter = Waiter.new # no longer needs a "capacity" argument
table = Table.new(philosophers.size)

philosophers.each_with_index do |philosopher, i| 
  # No longer manually create a thread, rely on async() to do that for us.
  philosopher.async.dine(table, i, waiter) 
end

sleep
```

The runtime behavior of this solution is similar to that of our mutex-based
solution. However, the following differences in implementation are worth noting:

* Each class that mixes in `Celluloid` becomes an actor with its own thread of execution.

* The Celluloid library intercepts any method call run through the `async` proxy
object and stores it in the actor's mailbox. The actor's thread will sequentially 
execute those stored methods, one after another.

* This behavior makes it so that we don't need to manage threads and mutex
synchronization explicitly. The Celluloid library handles that under 
the hood in an object-oriented manner.

* If we encapsulate all data inside actor objects, only the actor's
thread will be able to access and modify its own data. That prevents the
possibility of two threads writing to a critical region at the same time,
which eliminates the risk of deadlocks and data corruption.

These features are very useful for simplifying the way we think about
concurrent programming, but you're probably wondering how much magic is involved
in implementing them. Let's build our own minimal drop-in replacement for
Celluloid to find out!

## Rolling our own actor model

Celluloid provides much more functionality than what we can discuss
in this article, but building a barebones implementation of the actor
model is within our reach. In fact, the following 80 lines of code are
enough to serve as a replacement for our use of Celluloid in the 
previous example:

```ruby
require 'thread'

module Actor  # To use this, you'd include Actor instead of Celluloid
  module ClassMethods
    def new(*args, &block)
      Proxy.new(super)
    end
  end

  class << self
    def included(klass)
      klass.extend(ClassMethods)
    end

    def current
      Thread.current[:actor]
    end
  end

  class Proxy
    def initialize(target)
      @target  = target
      @mailbox = Queue.new
      @mutex   = Mutex.new
      @running = true

      @async_proxy = AsyncProxy.new(self)

      @thread = Thread.new do
        Thread.current[:actor] = self
        process_inbox
      end
    end

    def async
      @async_proxy
    end
      
    def send_later(meth, *args)
      @mailbox << [meth, args]
    end

    def terminate
      @running = false
    end

    def method_missing(meth, *args)
      process_message(meth, *args)
    end

    private

    def process_inbox
      while @running
        meth, args = @mailbox.pop
        process_message(meth, *args)
      end

    rescue Exception => ex
      puts "Error while running actor: #{ex}"
    end

    def process_message(meth, *args)
      @mutex.synchronize do
        @target.public_send(meth, *args)
      end
    end
  end

  class AsyncProxy
    def initialize(actor)
      @actor = actor
    end

    def method_missing(meth, *args)
      @actor.send_later(meth, *args)
    end
  end
end
```

This code mostly builds upon concepts that have already been covered in this 
article, so it shouldn't be too hard to follow with a bit of effort. That
said, combining meta-programming techniques and concurrency can
lead to code that makes your eyes glaze over, so we should also make
an attempt to discuss how this module works at the high level. Let's do that
now!

Any class that includes the `Actor` module will be converted into an actor and will be 
able to receive asynchronous calls. We accomplish this by overriding the constructor
of the target class so that we can return a proxy object every time an object of 
that class is instantiated. We also store the proxy object in a
thread level variable. This is necessary because when sending messages between actors, 
if we refer to self in method calls we will exposed the inner target object, 
instead of the proxy. This same [gotcha is also present in Celluloid](https://github.com/celluloid/celluloid/wiki/Gotchas).

Using this mixin, whenever we attempt to create an instance of a `Philosopher`
object, we will actually receive an instance of `Actor::Proxy`. The `Philosopher` 
class is left mostly untouched, and so the actor-like behavior is handled
entirely by the proxy object. Upon instantiation, that proxy creates
a mailbox to store the incoming asynchronous messages and a thread to process those 
messages. The inbox is a thread-safe queue that ensures that incoming message
are processed sequentially even if they arrive at the same time. Whenever the inbox
is empty, the actor's thread will be blocked until a new message needs to
be processed.

This is roughly how things work in Celluloid as well, although its
implementation is much more complex due to the many additional features it
offers. Still, if you understand this code, you're well on your way to having a
working knowledge of what the actor model is all about.

### Actors are helpful, but are not a golden hammer

Even this minimal implementation of the actor model gets the low-level
concurrency primitives out of our ordinary class definitions, and into a
centralized place where it can be handled in a consistent and reliable way.
Celluloid goes a lot farther than we did here by providing excellent fault
tolerance mechanisms, the ability to recover from failures, and lots of other
interesting stuff. However, these benefits do come with their own share of
costs and potential pitfalls.

So what can go wrong when using actors in Ruby? We've already hinted at the potential 
issues that can arise due to the issue of [self schizophrenia][self] in 
proxy objects. Perhaps more complicated is the issue of mutable state: while
using actors guarantees that the state *within* an object will be accessed
sequentially, it does not provide the same guarantee for the messages that are
being passed around between objects. In languages like Erlang, messages consist of immutable parameters, so consistency 
is enforced at the language level. In
Ruby, we don't have that constraint, so we either need to solve this problem by
convention, or by freezing the objects we pass around as arguments -- which is quite
restrictive!

Without attempting to enumerate all the other things that could
go wrong, the point here is simply that there is no such thing as a golden hammer 
when it comes to concurrent programming. Hopefully this article has
given you a basic sense of both the benefits and drawbacks of applying the
actor model in Ruby, along with enough background knowledge to apply some
of these ideas in your own projects. If it has done so, please do share your
story.

### Source code from this article

All of the code from this article is in 
Practicing Ruby's [example repository][examples],
but the links below highlight the main points of interest:

* [A solution that leads to deadlocks](https://github.com/elm-city-craftworks/practicing-ruby-examples/blob/master/v6/003/mutex_uncoordinated/dining_philosophers.rb)
* [A coordinated mutex-based solution](https://github.com/elm-city-craftworks/practicing-ruby-examples/blob/master/v6/003/mutex_coordinated/dining_philosophers.rb)
* [An actor-based solution using Celluloid](https://github.com/elm-city-craftworks/practicing-ruby-examples/blob/master/v6/003/celluloid/dining_philosophers.rb)
* [An actor-based solution using a hand-rolled actor library](https://github.com/elm-city-craftworks/practicing-ruby-examples/blob/master/v6/003/actors_from_scratch/dining_philosophers.rb)
* [Minimal implementation of the actor model](https://github.com/elm-city-craftworks/practicing-ruby-examples/blob/master/v6/003/lib/actors.rb)
* [Chopsticks class definition](https://github.com/elm-city-craftworks/practicing-ruby-examples/blob/master/v6/003/lib/chopstick.rb)
* [Table class definition](https://github.com/elm-city-craftworks/practicing-ruby-examples/blob/master/v6/003/lib/table.rb)

If you see anything in the code that you have questions about, don't hesitate to
ask.

[examples]: https://github.com/elm-city-craftworks/practicing-ruby-examples/tree/master/v6/003
[actors]: http://en.wikipedia.org/wiki/Actor_model
[celluloid]: http://celluloid.io/
[philosophers]: http://en.wikipedia.org/wiki/Dining_philosophers
[self]: http://en.wikipedia.org/wiki/Schizophrenia_%28object-oriented_programming%29
[afcapel]: https://github.com/afcapel
