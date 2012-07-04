Every `Proc` object is a closure, which means that each code block you write captures references to data from its surrounding scope for later use. Although that sounds highly academic, it has some very practical benefits that you're probably already aware of, as well as some drawbacks that you may or may not know about.

### Closures make block-based APIs feel natural

The closure property of `Proc` objects is what makes the following snippet of code possible:

```ruby
class Vector
  def initialize(data)
    @data = data
  end

  def *(num)
    @data.map { |e| e * num }
  end
end

>> Vector.new([1,2,3]) * 7
=> [7, 14, 21]
```

In this example, when we call `@data.map` and provide it with a code block to execute, we have no trouble accessing the `num` variable. However, this local variable is not defined within the block's local scope; it's defined within the enclosing scope (the `Vector#*`) method. To see that these are truly two different scopes, check out the following examples, which clarify the relationship between the `Proc` object's code and its enclosing scope.

```ruby
def proc_can_see_outer_scope_locals
  y = 10
  lambda { p defined?(y) }.call
end

def proc_can_modify_outer_scope_locals
  y = 10
  lambda { y = 20 }.call
  p y
end

def proc_destroys_block_local_vars_on_exit
  lambda { y = 10 }.call
  p defined?(y)
end

proc_can_see_outer_scope_locals          #=> "local-variable"
proc_can_modify_outer_scope_locals       #=> 20
proc_destroys_block_local_vars_on_exit   #=> nil
```

The first example demonstrates that a `Proc` object's code can access the local variables of its enclosing scope, which is exactly what is going on in our `Vector` example. The second example is an answer to a question that arises naturally from the first example, which is whether the `Proc` object's code can modify the contents of the local variables that are defined in its enclosing scope. The third example simply verifies that once the `Proc` object's code has been called, any variables set up within its own code block are wiped out and are not visible from the outer scope.

### Closures make memory management complicated

Though they take some getting used to, the behaviors provided by the closure property in `Proc` objects are relatively easy to understand and have many practical benefits. However, they do give rise to a complex behavior that sometimes leads to surprising results. Check out the following example for a bit of a head trip.

``` ruby
def new_counter
  x = 0
  lambda { x += 1 }
end

counter_a = new_counter
counter_a.call
counter_a.call

p counter_a.call #=> 3

counter_b = new_counter
p counter_b.call #=> 1

p counter_a.call #=> 4
```

In the example code, we see that the two `Proc` objects returned by the `new_counter()` method are referencing two different locations in memory. This behavior is a bit confusing because we can usually count on methods to clean up after themselves them once they wrap up whatever they are doing. But because the purpose of a `Proc` object is in part to be able to delay the execution of code, it's impossible for the `new_counter()` method to do this cleanup task for us. So here's what happens: `counter_a` gets a reference to the local variable `x` that was set up the first time we called `new_counter()`, and `counter_b` gets a reference to a different local variable `x` that was set up the second time we called `new_counter()`.

If used correctly, this behavior can be a feature. It's not one that you or I would use every day, but because this approach can be used to maintain state in a purely functional way, it is at least academically interesting. However, in most ordinary use cases, it is much more likely that this behavior is going to cause a memory leak than that it will do anything helpful for you, as it leads to lots of seemingly throwaway data stored in local variables getting dangling references that prevent that data from being garbage collected.

### Not all closure-based memory leaks are so obvious

Capturing references to locals from the enclosing scope for longer than necessary isn't the only way that you can cause leaks with `Proc` objects. Every `Proc` object also creates a reference to the object that it was defined within, creating a leak that is even harder to notice. Let's take a look at an example of how that can come back to bite you.

Suppose we have a configurable logger module and we want to record a message to the logs each time a new `User` object is created. If we were going for something simple and straightforward, we might end up with code similar to what you see here:

```ruby
module Logger
  extend self

  attr_accessor :output

  def log(msg)
    output << "#{msg}\n"
  end
end

class User
  def initialize(id)
    @id = id
    Logger.log("Created User with ID #{id}")
  end
end
```

But if we wanted to be a bit more fancy, we could build a logger that delayed the writing of the logs until we explicitly asked for them to be written. We could use `Proc` objects for lazy evaluation, giving us a potential speed boost whenever we didn't actually need to view our logs.

```ruby
module LazyLogger
  extend self

  attr_accessor :output

  def log(&callback)
    @log_actions ||= []
    @log_actions << callback
  end

  def flush
    @log_actions.each { |e| e.call(output) }
  end
end

class User
  def initialize(id)
    @id = id
    LazyLogger.log { |out| out << "Created User with ID #{id}" }
  end
end
```

Although this code may look simple, it has a subtle memory leak. The leak can be verified via the following simple script, which shows that 1000 users still exist in the system, even though the objects were created as throwaway objects:

```ruby
LazyLogger.output = ""
1000.times { |i| User.new(i) }

GC.start
p ObjectSpace.each_object(User).count #=> 1000
```

If instead we use our more vanilla code that does not use `Proc` objects, we see that for the most part*, the garbage collector has done its job.

```ruby
Logger.output = ""
1000.times { |i| User.new(i) }

GC.start

# (*): I expected below to be 0, but GC clearly ran. Weird.
p ObjectSpace.each_object(User).count #=> 1
```

Our `LazyLogger` leaks because when `LazyLogger.log` is called with a block from within `User#initialize`, a new `Proc` object is created that holds a reference to that user object. That `Proc` object ends up getting stored in the `@log_actions` array in `LazyLogger` module and needs to be kept alive at least until `LazyLogger.flush` is called in order for everything to work as expected. Thus our `User` objects that we expected to get thrown away still have live references to them, so they don't end up getting garbage collected.

These kinds of problems can be very easy to run into and very hard to work around. In fact, I've have been having trouble figuring out how to preserve the `LazyLogger` behavior in a way that'd plug the leak or at least mitigate it somewhat. In this particular case, it'd be possible to call `clear` on the `@log_actions` array whenever `flush` is called, and that would free up the references to the `User` instances. But that approach still ends up keeping unnecessary references alive longer than you might want, and the pattern doesn't necessarily apply generally to other scenarios.

### Reflections

Because we use code blocks so freely and tend to ignore the closure property, many Ruby applications and libraries have memory leaks in them. Even fairly experienced developers (myself included) don't necessarily design with these issues in mind. Those who do have firm memory constraints to deal with are forced to use a variety of awkward techniques to overcome this problem. 

One possible way to avoid closure-based memory leaks is to use `Method` objects in place of `Proc` objects wherever the closure properties are not required. Another option is to create `Proc` objects in a different context to avoid accidental references to objects that you want garbage collected. In fact, I recently needed to use the latter approach in order to make use of `ObjectSpace.define_finalizer`. Although that's a bit of an obscure topic, it's a good example of what we've just been talking about, so I recommend checking out [this article by Mike Perham](http://www.mikeperham.com/2010/02/24/the-trouble-with-ruby-finalizers/). 

I don't want to give much more advice on handling memory management, because it's not an area in which I'm particularly strong. I welcome any corrections to what I've said here, if you find that I've made a mistake anywhere.

### Questions/Discussion Points

This article wasn't especially long, but the material is quite dense, and I don't want to push my luck by covering too many concepts at once. That said, I've provided a few exercises for those who want to dig a bit deeper and would be happy to continue discussing the topic in general now that we have a starting point. Leave a comment if something is on your mind!

* Find a good example of an API that allows you to use a `Method` object in place of a `Proc` object, or create your own. Investigate the performance and memory differences between the two approaches by writing benchmarks.

* Use two different approaches to implement your own `attr_accessor` function: one using the closure-based `define_method`, and another using `module_eval` with a string. Compare the performance characteristics of calling the dynamically defined methods and try to reason about why one is faster than the other, if they are not comparable to one another.

* Share your thoughts on when you need to worry about the downsides of closures, and when you don't. Come up with some metrics for determining what issues to look out for.

* CHALLENGE: Explain why my `ObjectSpace` example showed 1 `User` instance, not 0!
