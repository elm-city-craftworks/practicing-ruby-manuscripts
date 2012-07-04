When I first came to Ruby, one of the things that impressed me the most was the killer features provided by the `Enumerable` module. I eventually also came to love `Enumerator`, even though it took me a long time to figure out what it was and what one might use it for.

As a beginner, I had always assumed that these features worked through some dark form of magic that was buried deep within the Ruby interpreter. With so much left to learn just in order to be productive, I was content to postpone learning the details about what was going on under the hood. After some time, I came to regret that decision, thanks to David A. Black.

David teaches Ruby to raw beginners not only by showing them what `Enumerable` can do, but also by making them implement their own version of it! This is a profoundly good exercise, because it exposes how nonmagical the features are: if you understand `yield`, you can build all the methods in `Enumerable`. Similarly, the interesting features of `Enumerator` can be implemented fairly easily if you use Ruby's `Fiber` construct.

In this article, we're going to work through the exercise of rolling your own subset of the functionality provided by `Enumerable` and `Enumerator`, discussing each detail along the way. Regardless of your skill level, an understanding of the elegant design of these constructs will undoubtedly give you a great source of inspiration that you can draw from when designing new constructs in your own programs.

### Setting the stage with some tests

I've selected a small but representative subset of the features that `Enumerable` and `Enumerator` provide and written some tests to nail down their behavior. These tests will guide my implementations throughout the rest of this article and serve as a roadmap for you if you'd like to try out the exercise on your own.

If you have some time to do so, try to get at least some of the tests to go green before reading my implementation code and explanations, as you'll learn a lot more that way. But if you're not planning on doing that, at least read through the tests carefully and think about how you might go about implementing the features they describe.

```ruby
class SortedList
  include FakeEnumerable

  def initialize
    @data = []
  end

  def <<(new_element)
    @data << new_element
    @data.sort!
   
    self
  end
  
  def each
    if block_given?
      @data.each { |e| yield(e) }
    else
      FakeEnumerator.new(self, :each)
    end
  end 
end

require "minitest/autorun"

describe "FakeEnumerable" do
  before do
    @list = SortedList.new

    # will get stored interally as 3,4,7,13,42
    @list << 3 << 13 << 42 << 4 << 7
  end

  it "supports map" do
    @list.map { |x| x + 1 }.must_equal([4,5,8,14,43])  
  end

  it "supports sort_by" do
    # ascii sort order
    @list.sort_by { |x| x.to_s }.must_equal([13, 3, 4, 42, 7])
  end

  it "supports select" do
    @list.select { |x| x.even? }.must_equal([4,42])
  end

  it "supports reduce" do
    @list.reduce(:+).must_equal(69)
    @list.reduce { |s,e| s + e }.must_equal(69)
    @list.reduce(-10) { |s,e| s + e }.must_equal(59)
  end
end

describe "FakeEnumerator" do
  before do
    @list = SortedList.new

    @list << 3 << 13 << 42 << 4 << 7
  end

  it "supports next" do
    enum = @list.each

    enum.next.must_equal(3)
    enum.next.must_equal(4)
    enum.next.must_equal(7)
    enum.next.must_equal(13)
    enum.next.must_equal(42)

    assert_raises(StopIteration) { enum.next }
  end

  it "supports rewind" do
    enum = @list.each

    4.times { enum.next }
    enum.rewind

    2.times { enum.next }
    enum.next.must_equal(7)
  end

  it "supports with_index" do
    enum     = @list.map
    expected = ["0. 3", "1. 4", "2. 7", "3. 13", "4. 42"]  

    enum.with_index { |e,i| "#{i}. #{e}" }.must_equal(expected)
  end
end
```

If you do decide to try implementing these features yourself, get as close to the behavior that Ruby provides as you can, but don't worry if your implementation is different from what Ruby really uses. Just think of this as if it's a new problem that needs solving, and let the tests guide your implementation. Once you've done that, read on to see how I did it.

### Implementing the `FakeEnumerable` module

Before I began work on implementing `FakeEnumerable`, I needed to get its tests to a failure state rather than an error state. The following code does exactly that:

```ruby
module FakeEnumerable
  def map 
  end

  def select
  end

  def sort_by
  end

  def reduce(*args)
  end
end
```

I then began working on implementing the methods one by one, starting with `map`. The key thing to realize while working with `Enumerable` is that every feature will build on top of the `each` method in some way, using it in combination with `yield` to produce its results. The `map` feature is possibly the most straightforward nontrivial combination of these operations, as you can see in this implementation:

```ruby
def map 
  out = []

  each { |e| out << yield(e) }

  out
end
```

Here we see that `map` is simply a function that builds up a new array by taking each element and replacing it with the return value of the block you provide to it. We can clean this up to make it a one liner using `Object#tap`, but I'm not sure if I like that approach because it breaks the simplicity of the implementation a bit. That said, I've included it here for your consideration and will use it throughout the rest of this article, just for the sake of brevity.

```ruby
def map
  [].tap { |out| each { |e| out << yield(e) } }
end
```

Implementing `select` is quite easy as well. It builds on the same concepts used to implement `map` but adds a conditional check to see whether the block returns a `true` value. For each new yielded element, if the value returned by the block is logically true, the element gets added to the newly built array; otherwise, it does not.

```ruby
def select
  [].tap { |out| each { |e| out << e if yield(e) } }
end
```

Implementing `sort_by` is a little more tricky. I cheated and looked at the API documentation, which (perhaps surprisingly) describes how the method is implemented and even gives a reference implementation in Ruby. Apparently, `sort_by` uses a [Schwartzian transform](http://en.wikipedia.org/wiki/Schwartzian_transform) to convert the collection we are iterating over into tuples containing the sort key and the original element. It then uses `Array#sort` to put these in order, and it finally uses `map` on the resulting array to convert the array of tuples back into an array of the elements from the original collection. That's definitely more confusing to explain than it is to implement in code, so just look at the following code for clarification:

```ruby
def sort_by
  map { |a| [yield(a), a] }.sort.map { |a| a[1] }
end
```

The interesting thing about this implementation is that `sort_by` is dependent on `map`, both on the current collection being iterated over as well as on the `Array` it generates. But after tracing it down to the core, this method is still expecting the collection to implement only the `each` method. Additionally, because `Array#sort` is thrown into the mix, your sort keys need to respond to `<=>`. But for such a powerful method, the contract is still very narrow.

Implementing `reduce` is a bit more involved because it has three different ways of interacting with it. It's also interesting because it's one of the few `Enumerable` methods that isn't necessarily designed to return an `Array` object. I'll let you ponder the following implementation a bit before providing more commentary, because reading through it should be a good exercise. 

```ruby
def reduce(operation_or_value=nil)
  case operation_or_value
  when Symbol
    # convert things like reduce(:+) into reduce { |s,e| s + e }
    return reduce { |s,e| s.send(operation_or_value, e) }
  when nil
    acc = nil
  else
    acc = operation_or_value
  end

  each do |a|
    if acc.nil?
      acc = a
    else
      acc = yield(acc, a)
    end
  end

  return acc
end
```

First, I have to say I'm not particularly happy with my implementation; it seems a bit too brute force and I think I might be missing some obvious refactorings. But it should have been readable enough for you to get a general feel for what's going on. The first paragraph of code is simply handling the three different cases of `reduce()`. The real operation happens starting with our `each` call.

Without a predefined initial value, we set the initial value to the first element in the collection, and our first yield occurs starting with the second element. Otherwise, the initial value and first element are yielded. The purpose of `reduce()` is to perform an operation on each successive value in a list by combining it in some way with the last calculated value. In this way, the list gets reduced to a single value in the end. This behavior explains why the old alias for this method in Ruby was called `inject`: a function is being injected between each element in the collection via our `yield` call. I find this operation much easier to understand when I'm able to see it in terms of primitive concepts such as `yield` and `each` because it makes it possible to trace exactly what is going on.

If you are having trouble following the implementation of `reduce()`, don't worry about it. It's definitely one of the more complex `Enumerable` methods, and if you try to implement a few of the others and then return to studying `reduce()`, you may have better luck. But the beautiful thing is that if you ignore the `reduce(:+)` syntactic sugar, it introduces no new concepts beyond that what is used to implement `map()`. If you think you understand `map()` but not `reduce()`, it's a sign that you may need to brush up on your fundamentals, such as how `yield` works.

If you've been following along at home, you should at this point be passing all your `FakeEnumerable` tests. That means it's time to get started on our `FakeEnumerator`.

### Implementing the `FakeEnumerator` class

Similar to before, I needed to write some code to get my tests to a failure state. First, I set up the skeleton of the `FakeEnumerator` class.

```ruby
class FakeEnumerator
  def next
  end

  def with_index
  end

  def rewind
  end
end
```

Then I realized that I needed to back and at least modify the `FakeEnumerable#map` method, as my tests rely on it returning a `FakeEnumerator` object when a block is not provided, in a similar manner to the way `Enumerable#map` would return an `Enumerator` in that scenario.

```ruby
module FakeEnumerable
  def map 
    if block_given?
      [].tap { |out| each { |e| out << yield(e) } }
    else
      FakeEnumerator.new(self, :map)
    end
  end
end
```

Although, technically speaking, I should have also updated all my other `FakeEnumerable` methods, it's not important to do so because our tests don't cover it and that change introduces no new concepts to discuss. With this change to `map`, my tests all failed rather than erroring out, which meant it was time to start working on the implementation code.

But before we get started, it's worth reflecting on the core purpose of an `Enumerator`, which I haven't talked about yet. At its core, an `Enumerator` is simply a proxy object that mixes in `Enumerable` and then delegates its `each` method to some other iterator provided by the object it wraps. This behavior turns an internal iterator into an external one, which allows you to pass it around and manipulate it as an object. 

Our tests call for us to implement `next`, `rewind`, and `each_index`, but before we can do that meaningfully, we need to make `FakeEnumerator` into a `FakeEnumerable`-enabled proxy object. There are no tests for this because I didn't want to reveal too many hints to those who wanted to try this exercise at home, but this code will do the trick:

```ruby
class FakeEnumerator
  include FakeEnumerable

  def initialize(target, iter) 
    @target = target
    @iter   = iter
  end

  def each(&block)
    @target.send(@iter, &block) 
  end

   # other methods go here...
end
```

Here we see that `each` uses `send` to call the original iterator method on the target object. Other than that, this is the ordinary pattern we've seen in implementing other collections. The next step is to implement our `next` method, which is a bit tricky.

What we need to be able to do is iterate once, then pause and return a value. Then, when `next` is called again, we need to be able to advance one more iteration and repeat the process. We could do something like run the whole iteration and cache the results into an array, then do some sort of indexing operation, but that's both inefficient and impractical for certain applications. This problem made me realize that Ruby's `Fiber` construct might be a good fit because it specifically allows you to jump in and out of a chunk of code on demand. So I decided to try that out and see how far I could get. After some fumbling around, I got the following code to pass the test:

```ruby
# loading the fiber stdlib gives us some extra features, including Fiber#alive?
require "fiber" 

class FakeEnumerator
  def next
    @fiber ||= Fiber.new do
      each { |e| Fiber.yield(e) }

      raise StopIteration
    end

    if @fiber.alive?
      @fiber.resume 
    else
      raise StopIteration
    end
  end
end
```

This code is hard to read because it isn't really a linear flow, but I'll do my best to explain it using my very limited knowledge of how the `Fiber` construct works. Basically, when you call `Fiber#new` with a block, the code in that block isn't executed immediately. Instead, execution begins when `Fiber#resume` is called. Each time a `Fiber#yield` call is encountered, control is returned to the caller of `Fiber#resume` with the value that was passed to `Fiber#yield` returned. Each subsequent `Fiber#resume` picks up execution back at the point where the last `Fiber#yield` call was made, rather than at the beginning of the code block. This process continues until no more `Fiber#yield` calls remain, and then the last executed line of code is returned as the final value of `Fiber#resume`. Any additional attempts to call `Fiber#resume` result in a `FiberError` because there is nothing left to execute.

If you reread the previous paragraph a couple of times and compare it to the definition of my `next` method, it should start to make sense. But if it's causing your brain to melt, check out the [Fiber documentation](http://ruby-doc.org/core-1.9/classes/Fiber.html), which is reasonably helpful. 

The very short story about this whole thing is that using a `Fiber` in our `next` definition lets us keep track of just how far into the `each` iteration we are and jump back into the iterator on demand to get the next value. I prevent the `FiberError` from ever occurring by checking to see whether the `Fiber` object is still alive before calling `resume`. But I also need to make it so that the final executed statement within the `Fiber` raises a `StopIteration` error as well, to prevent it from returning the result of `each`, which would be the collection itself. This is a kludge, and if you have a better idea for how to handle this case, please leave me a comment.

The use of `Fiber` objects to implement `next` makes it possible to work with infinite iterators, such as `Enumerable#cycle`. Though we won't get into implementation details, the following code should give some hints as to why this is a useful feature:

```ruby
>> row_colors = [:red, :green].cycle
=> #<Enumerator: [:red, :green]:cycle>
>> row_colors.next
=> :red
>> row_colors.next
=> :green
>> row_colors.next
=> :red
>> row_colors.next
=> :green
```

As cool as that is, and as much as it makes me want to dig into implementing it, I have to imagine that you're getting tired by now. Heck, I've already slept twice since I started writing this article! So let's hurry up and finish implementing `rewind` and `each_index` so that we can wrap things up.

I found a way to implement `rewind` that is trivial, but something about it makes me wonder if I've orphaned a `Fiber` object somewhere and whether that has weird garbage collection mplications. But nonetheless, because our implementation of `next` depends on the caching of a `Fiber` object to keep track of where it is in its iteration, the easiest way to rewind back to the beginning state is to simply wipe out that object. The following code gets my `rewind` tests passing:

```ruby
def rewind
  @fiber = nil
end
```

Now only one feature stands between us and the completion of our exercise: `with_index`. The real `with_index` method in Ruby is much smarter than what you're about to see, but for its most simple functionality, the following code will do the trick:

```ruby
def with_index
  i = 0
  each do |e| 
    out = yield(e, i)
    i += 1
    out
  end
end
```

Here, I did the brute force thing and maintained my own counter. I then made a small modification to control flow so that rather than yielding just the element on each iteration, both the element and its index are yielded. Keep in mind that the `each` call here is a proxy to some other iterator on another collection, which is what gives us the ability to call `@list.map.with_index` and get `map` behavior rather than `each` behavior. Although you won't use every day, knowing how to implement an around filter using `yield` can be quite useful.

With this code written, my full test suite finally went green. Even though I'd done these exercises a dozen times before, I still learned a thing or two while writing this article, and I imagine there is still plenty left for me to learn as well. How about you?

### Reflections

This is definitely one of my favorite exercises for getting to understand Ruby better. I'm not usually big on contrived practice drills, but there is something about peeling back the magic on features that look really complex on the surface that gives me a great deal of satisfaction. I find that even if my solutions are very much cheap counterfeits of what Ruby must really be doing, it still helps tremendously to have implemented these features in any way I know how, because it gives me a mental model of my own construction from which to view the features.

If you enjoyed this exercise, there are a number of things that you could do to squeeze even more out of it. The easiest way to do so is to implement a few more of the `Enumerable` and `Enumerator` methods. As you do that, you'll find areas where the implementations we built out today are clearly insufficient or would be better off written another way. That's fine, because it will teach you even more about how these features hang together. You can also discuss and improve upon the examples I've provided, as there is certainly room for refactoring in several of them. Finally, if you want to take a more serious approach to things, you could take a look at the tests in [RubySpec](https://github.com/rubyspec/rubyspec) and the implementations in [Rubinius](https://github.com/rubinius/rubinius). Implementing Ruby in Ruby isn't just something folks do for fun these days, and if you really enjoyed working on these low-level features, you might consider contributing to Rubinius in some way. The maintainers of that project are amazing, and you can learn a tremendous amount that way.

Of course, not everyone has time to contribute to a Ruby implementation, even if it's for the purpose of advancing their own understanding of Ruby. So I'd certainly settle for a comment here sharing your experiences with this exercise.
