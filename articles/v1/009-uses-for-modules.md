> Note: This article series on modules is also available as a [PDF download]. The
> PDF version has been revised and is more up-to-date than what you see here.

[PDF download]:https://github.com/elm-city-craftworks/pr-monthly/blob/gh-pages/b5e5a89847701c4aa7c170cf/sept-2012-modules.pdf?raw=true

### Using Mix-ins to Augment Class Definitions

Although knowing [how to use modules for namespacing](http://practicingruby.com/articles/36) is important, it's really only a small part of what you can do with modules. What modules do best is providing a convenient way to write code that can be mixed into other objects, augmenting their behaviors. Because modules facilitate code sharing in a way that is distinct from both the general OO concept of class inheritance and from things like Java's interfaces, they require you to think about your design in a way that's a bit different from most other object oriented programming languages.

While I imagine that most of our readers are comfortable with using mixins, I'll
refer to some core Ruby mixins to illustrate their power before moving on to more 
subtle points. For example, consider the following bit of code which implements lazily evaluated computations:

```ruby
class Computation

  def initialize(&block)
    @action = block
  end

  def result
    @result ||= @action.call
  end

  def <(other)
    result < other.result
  end

  def >(other)
    result > other.result
  end

  def >=(other)
    result >= other.result
  end

  def <=(other)
    result <= other.result
  end

  def ==(other)
    result == other.result
  end

end

a = Computation.new { 1 + 1 }
b = Computation.new { 4*5 }
c = Computation.new { -3 }

p a < b  #=> true
p a <= b #=> true
p b > c  #=> true
p b >= c #=> true
p a == b #=> false
```

While Ruby makes defining custom operators easy, there is a lot more code here than there needs to be. We can easily clean it up by mixing in Ruby's built in `Comparable` module.

```ruby
class Computation
  include Comparable

  def initialize(&block)
    @action = block
  end

  def result
    @result ||= @action.call
  end

  def <=>(other)
    return  0 if result == other.result
    return  1 if result > other.result
    return -1 if result < other.result
  end
end

a = Computation.new { 1 + 1 }
b = Computation.new { 4*5 }
c = Computation.new { -3 }

p a < b  #=> true
p a <= b #=> true
p b > c  #=> true
p b >= c #=> true
p a == b #=> false
```

We see that our individual operator definitions have disappeared, and in its place are two new bits of code. The first new thing is just an include statement that tells Ruby to mix the `Comparable` functionality into the `Computation` class definition. But in order to make use of the mixin, we need to tell `Comparable` how to evaluate the sort order of our `Computation` objects, and that's where `<=>` comes in.

The `<=>` method, sometimes called the spaceship operator, essentially fills in a template method that allows `Comparable` to work. It codifies the notion of comparison in an abstract manner by expecting the method to return `-1` when the current object is considered less than the object it is being compared to, `0` when the two are considered equal, and `1` when the current object is considered greater than the object it is being compared to.

If you're still scratching your head a bit, pretend that rather than being a core Ruby object, that we've implemented `Comparable` ourselves by writing the following code.

```ruby
module Comparable
  def ==(other)
    (self <=> other) == 0
  end

  def <(other)
    (self <=> other) == -1
  end

  def <=(other)
    self < other || self == other
  end

  def >(other)
    (self <=> other) == 1
  end

  def >=(other)
    self > other || self == other
  end
end
```

Now, if you imagine these method definitions literally getting pasted into your `Computation` class when `Comparable` is included, you'll see that it would provide a behavior that is functionally equivalent to our initial example.

Of course, it wouldn't make sense for Ruby to implement such a feature for us
without using it in its own structures. Because Ruby's numeric classes
all implement `<=>`, we are able to simply delegate our `<=>` call to the 
result of the computations.

```ruby
class Computation
  include Comparable

  def initialize(&block)
    @action = block
  end

  def result
    @result ||= @action.call
  end

  def <=>(other)
    result <=> other.result
  end
end
```

The only requirement for this code to work as expected is that each `Computation`'s result must implement the `<=>` method. Since all objects that mix in `Comparable` have to implement `<=>`, any comparable object returned as a result should work fine here.

While not a technically complicated example, there is surprising power in having a primitive built into your programming language which trivializes the implementation of the Template Method design pattern. If you look at Ruby's `Enumerable` module and the powerful features it offers, you might think it would be a much more complicated example to study. But it too hinges on Template Method and requires only an `each()` method to give you all sorts of complex functionality including things like `select()`, `map()`, and `inject()`. If you haven't tried it before, you should certainly try to roll your own `Enumerable` module to get a sense of just how useful mixins can be.

We can also invert this relationship by having our class define a template, and then relying on the module that we mix in to provide the necessary details. If we look back at a previous example `TicTacToe`, we can see a practical example of this technique by looking at the play method in our `TicTacToe::Game` class.

```ruby
module TicTacToe
  class Game
    def play
      catch(:finished) do
        loop do
          start_new_turn
          show_board

          check_move { |error_message| puts error_message }
          check_win { puts "#{current_player} wins" }
          check_draw { puts "It's a tie" }
        end
      end
    end

    # ...
  end
end
```

In this code, we wanted to keep our event loop abstract, and rely on a mixed in module to provide the logic for executing and validating a move as well as checking end game conditions. As a result, we ended up with the `TicTacToe::Rules` module shown below.

```ruby
module TicTacToe
  module Rules
    def check_move
      row, col = move_input
      board[row, col] = current_player
    rescue TicTacToe::Board::InvalidRequest => error
      yield error.message if block_given?
      retry
    end

    def check_win
      return false unless board.last_move

      win = board.intersecting_lines(*board.last_move).any? do |line|
        line.all? { |cell| cell == current_player }
      end

      if win
        yield
        game_over
      end
    end

    def check_draw
      if @board.covered?
        yield
        game_over
      end
    end
  end
end
```

When we look at this code, we see some basic business logic implementing the rules of Tic Tac Toe, with some placeholder hooks being provided by `yield()` that allows the calling code to inject some logic at certain key points in the process. This is how we manage to split the UI code from the game logic, without creating frivolous adapter classes.

While this is a more complicated example than our walkthrough of `Comparable`, the two share a common thread. In both cases, some coupling exists between the module and the object it is being mixed into. This is a common pattern when using mixins, in which the module and the code it is mixed into have to do a bit of a secret handshake to be able to talk to one another, but as long as they agree on that, neither needs to know about the other's inner workings. The end result is two components which must agree on an interface but do not need to necessarily understand each other's implementations. Code with this sort of coupling is easy to test and easy to refactor.

### Using Mix-ins to Augment Objects Directly

As you may already know, Ruby's mixin capability is not limited to simply including new behavior into a class definition. You can also extend the behavior of a class itself, through the use of the `extend()` method. We can look to the Ruby standard library <i>forwardable</i> for a nice example of how this is used. Consider the following trivial `Stack` implementation.

```ruby
require "forwardable"

class Stack
  extend Forwardable

  def_delegators :@data, :push, :pop, :size, :first, :empty?

  def initialize
    @data = []
  end
end
```

In this example, we can see that after we extend our `Stack` class with the `Forwardable` module, we are provided with a class level method called `def_delegators` which allows us to easily define methods which delegate to an object stored in the specified instance variable. Playing around with the `Stack` object a bit should illustrate what this code has done for us.

```ruby
>> stack = Stack.new
=> #<Stack:0x4f09c @data=[]>
>> stack.push 1
=> [1]
>> stack.push 2
=> [1, 2]
>> stack.push 3
=> [1, 2, 3]
>> stack.size
=> 3
>> until stack.empty?
>>   p stack.pop
>> end
3
2
1
```

As before, it may be helpful to think about how we might implement `Forwardable` ourselves. The following bit of code shows one way to approach the problem.

```ruby
module MyForwardable
  def def_delegators(ivar, *delegated_methods)
    delegated_methods.each do |m|
      define_method(m) do |*a, &b|
        obj = instance_variable_get(ivar)
        obj.send(m,*a, &b)
      end
    end
  end
end
```

While the metaprogramming aspects of this may be a bit noisy to read if you're not familiar with them, this is fairly vanilla dynamic Ruby code. If you've got Ruby 1.9.2 installed, you can actually try it out on your own and verify that it does indeed work as expected. But the practical use case of this code isn't what's important here.

The key thing to notice about this code is that while it essentially implements a class method, nothing in the module's syntax directly indicates this to be the case. The only hint we get that this is meant to be used at the class level is the use of `define_method()`, but we need to dig into the implementation code to notice that.

Before we wrap up, we should investigate why this is the case.

### A Brief Stroking of the Beard

The key thing to recognize is that `include()` mixes methods into the instances of the base object while `extend()` mixes methods into the base object itself. Notice that this is more general than a class method / instance method dichotomy.

Let's explore a few different possibilities using a somewhat contrived example so that we can focus on the mixin mechanics. First, we start with an ordinary module, which is somewhat useless on its own.

```ruby
module Greeter
  def hello
    "hi"
  end
end
```

By including `Greeter` into `SomeClass`, we make it so that we can now call `hello()` on instances of `SomeClass`.

```ruby
class SomeClass
  include Greeter
end

SomeClass.new.hello #=> "hi"
```

But as we saw in the `Forwardable` example, extending `AnotherClass` with `Greeter` would allow us to call the hello method directly at the class level, as in the example below.

```ruby
class AnotherClass
  extend Greeter
end

AnotherClass.hello #=> "hi"
```

Be sure to note at this point that `extend()` and `include()` are two totally
different operations. Because you did not extend `SomeClass` with `Greeter`, you
could not call `SomeClass.hello()`. Similarly, you cannot call
`AnotherClass.new.hello()` without explicitly including `Greeter`.

From the examples so far, it might seem as if `include()` is for defining instance methods, and `extend()` is for class methods. But that is not quite accurate, and the next bit of code illustrates just how much deeper the rabbit hole goes.

```ruby
obj = Object.new
obj.extend(Greeter)
obj.hello #=> "hi"
```

Before you let this example make you go cross-eyed, let's review the key point I made at the beginning of this section: <i>The key thing to recognize is that `include()` mixes methods into the instances of the base object while `extend()` mixes methods into the base object itself.</i>

Since not every base object can have instances, not every object can have modules included into them (in fact, only classes can). But *every* object can be extended by modules. This includes, among other things, classes and modules themselves.

Let's try to bring the two `extend()` examples closer together with the following little snippet:

```ruby
MyClass = Class.new
MyClass.extend(Greeter)
MyClass.hello #=> "hi"
```

If you feel like you understand the lines above, you're ready for the rest
of this mini-series. If not, please ponder the following questions and leave a
comment sharing your thoughts.

### Questions To Consider

  * Why do we have both `include()` and `extend()` available to us? Why not just have one way of doing mixins?

  * When you write `extend()` within a class definition, does it do any sort of special casing? Or is it the same as calling `extend()` on any other object?

  * Except for mixing in class methods, what is `extend()` useful for?

Please feel free to ask for hints on any of these if you're stumped, or share your answers if you'd like to help others and maybe get a bit of feedback to check your assumptions against.

  
> **NOTE:** This article has also been published on the Ruby Best Practices blog. There [may be additional commentary](http://blog.rubybestpractices.com/posts/gregory/038-issue-9-uses-for-modules.html#disqus_thread) 
over there worth taking a look at.
