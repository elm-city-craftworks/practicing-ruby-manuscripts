Back in 1997, James Noble published a paper called [Arguments and Results](http://www.laputan.org/pub/patterns/noble/noble.pdf) which outlined several useful patterns for designing better object protocols. Despite the fact that this paper was written nearly 15 years ago, it addresses design problems that programmers still struggle with today. In this two part article, I show how the patterns James came up with can be applied to modern Ruby programs.

<u>Arguments and Results</u> is written in such a way that it is natural to split the patterns it describes into two separate groups: patterns about method arguments and patterns about the results returned by methods. I've decided to split this Practicing Ruby article in the same manner in order to make it easier for me to write and easier for you to read. 

In [Issue 2.14](http://practicingruby.com/articles/14) I outlined various kinds of arguments objects that can be used to simplify the messages being sent within a system. In this issue, I will show how results objects can provide similar flexibility on the response side of things.

### Results objects

Results objects are similar to argument objects in that they simplify the interface of one object at the cost of introducing new objects into the system. The `Report` class I built for [Issue 2.13](http://practicingruby.com/articles/13) is a good example of this sort of object. If we start with its class definition and work our way backwards to what the code would have looked like without it, we will be able to see the reason why this object was introduced in the first place.

```ruby
module SubscriptionCounter
  class Report
    def initialize(series)
      @series = series

      @issue_numbers = series.map(&:number)
      @weekly_counts = series.map(&:count)
      @weekly_deltas = series.map(&:delta)
      @average_delta = Statistics.adjusted_mean(@weekly_deltas)
    end
 
    attr_reader :issue_numbers, :weekly_counts, :weekly_deltas, 
                :average_delta, :summary, :series

    def table(*fields)
      series.map { |e| fields.map { |f| e.send(f) } }
    end
  end
end
```

The following code outlines how `Report` is actually used by client code. It essentally serves as a bridge between `Series` and `Report::PDF`.

```ruby
campaigns = SubscriptionCounter::Campaign.all
series    = SubscriptionCounter::DataSeries.new(campaigns, 10)
report    = SubscriptionCounter::Report.new(series)

SubscriptionCounter::Report::PDF.new(report).save_as("pr-subscribers.pdf")
```

If we pretend that `Report` never existed and that its methods were implemented directly on `DataSeries`, we would end up with something similar to the following client code:

```ruby
campaigns = SubscriptionCounter::Campaign.all
series    = SubscriptionCounter::DataSeries.new(campaigns, 10)

SubscriptionCounter::Report::PDF.new(series).save_as("pr-subscribers.pdf")
```

This example actually looks a bit cleaner than the previous one, but results in six new methods getting added to `DataSeries` and introduces tighter coupling between the `DataSeries` and `PDF` objects. Due to the increased coupling, a change in the interface of the `DataSeries` object will directly impact the presentation code in the `PDF` object, whereas before the `Report` object provided a buffer zone between the two classes.

While it is still possible to substitute a `DataSeries` object with some other object that provides an equivalent interface, we have lost the flexibility of reusing the code that actually does the aggregation work. Before removing the `Report` object, it was possible to use it to wrap any pretty much any `Enumerable` collection of objects which provided `number`, `count`, and `delta` methods. Now we must either do the aggregation ourselves or use a `DataSeries` object directly.

These downsides are why I introduced the `Report` object in the first place, and they make the case for using an object that exists simply to aggregate some results based on the data contained in another object. If I wanted to make the integration of this results object a bit tighter and simplify the client code, I could have introduced a `DataSeries#report` method such as the one shown below:

```ruby
module SubscriptionCounter
  class DataSeries
    def report
      Report.new(self)
    end
  end
end
```

With this method added, I could either have the `Report::PDF` accept an object that responds to `report`, or call the method explicitly in my client code. If I went with the former approach I could use the same client code as shown in the previous example, making the `Report` object completely transparent to the end user. However, the latter approach still looks a bit cleaner than what I had originally without introducing too much coupling into the system:

```ruby
campaigns = SubscriptionCounter::Campaign.all
series    = SubscriptionCounter::DataSeries.new(campaigns, 10)

SubscriptionCounter::Report::PDF.new(series.report).save_as("pr-subscribers.pdf")
```

While this pattern certainly has its benefits, it may feel a bit unexciting to Ruby developers. When results objects are introduced simply to reduce coupling between two different subsystems in your project or to provide a bit of encapsulation for some cached values, they feel like ordinary objects that don't require a special name. However, explictly thinking of results objects as an abstraction opens the door for more interesting techniques as well.

### Lazy objects

One interesting aspect of introducing results objects into a system is that it helps facilitate lazy evaluation. Ruby's own `Enumerator` object provides an excellent example of how powerful this combination can be. Laziness allows `Enumerator` objects to efficiently chain different transformations together. This makes it possible to do things like `map.with_index` without having to iterate over the collection multiple times or store an intermediate representation of the indexed data:

```ruby
>> [1,2,3,4].map.with_index { |e,i| "#{i}. #{e}" }
=> ["0. 1", "1. 2", "2. 3", "3. 4"]
```

Lazy objects can also represent infinite or repeating sequences in a very elegant way. The examples below show some bits of functionality baked into `Enumerator` that make modeling these kinds of sequences a whole lot easier.

```ruby
>>  players = [:red, :black].cycle
=> #<Enumerator: [:red, :black]:cycle>
>> players.next
=> :red
>> players.next
=> :black
>> players.next
=> :red
>> players.next
=> :black
>> odds = Enumerator.new { |y| k = 0; loop { y << 2*k + 1; k += 1 } }
=> #<Enumerator: #<Enumerator::Generator:0x00000100972b50>:each>
>> odds.next
=> 1
>> odds.next
=> 3
>> odds.next
=> 5
>> odds.take(10)
=> [1, 3, 5, 7, 9, 11, 13, 15, 17, 19]
```

While infinite sequences may seem like a very academic topic, they show up in some practical applications as well. For example, some video games use procedural generation to produce seemingly infinite randomly generated maps. The video below demonstrates that technique being used in a very crude manner, but the same general approach could be used to build rich three dimensional environments as well, such as the ones found in [MineCraft](http://www.minecraft.net/). (_NOTE: I accidentally uploaded this video with ambient sounds rather than muted, and won't be able to fix this until I return from vacation after December 15th. If you don't like the sound of keyboard motions, heavy breathing, and some weird beeping noise: mute your audio before playing this video. Sorry!_)

<div align="center">
<iframe width="640" height="480" src="//www.youtube.com/embed/fg-dYZfd6Y4?rel=0" frameborder="0" allowfullscreen></iframe>
</div>

To implement the map generation code, I put together a simple `Location` object which is essentially an infinite two dimensional doubly linked list. Notice how the class definition below makes extensive use of the common `||=` idiom to handle the lazy evaluation and caching.

```ruby
class Location
  def self.[](x,y)
    @locations        ||= {}
    @locations[[x,y]] ||= new(x,y)
  end 

  def initialize(x,y)
    @x     = x
    @y     = y
    @color = [:green, :green, :blue].sample
  end

  def ==(other)
    [x,y] == [other.x, other.y]
  end  

  attr_reader :x, :y, :color

  def north
    @north ||= Location[@x,@y-1] 
  end

  def south
    @south ||= Location[@x,y+1]
  end

  def east
    @east ||= Location[@x+1, @y] 
  end

  def west
    @west ||= Location[@x-1, @y] 
  end

  def neighbors
    [north, south, east, west]
  end
end
```

While this technique works fine and is the traditional way to achieve lazy evaluation in Ruby, it feels a bit primitive. Ruby does not provide a general purpose construct for lazy evaluation, but if it did, it would allow us to write code similar to what you see below:

```ruby
class Location
  def self.[](x,y)
    @locations        ||= {}
    @locations[[x,y]] ||= new(x,y)
  end 

  def initialize(x,y)
    @x     = x
    @y     = y
    @color = [:green, :green, :blue].sample

    @north = LazyObject.new { Location[@x,@y-1] }
    @south = LazyObject.new { Location[@x,y+1] }
    @east  = LazyObject.new { Location[@x+1, @y] }
    @west  = LazyObject.new { Location[@x-1, @y] }
  end

  def ==(other)
    [x,y] == [other.x, other.y]
  end  

  def neighbors
    [north, south, east, west]
  end

  attr_reader :x, :y, :color, :north, :south, :east, :west
end
```

Such an object can be implemented as a simple proxy which delays the execution of a callback until the results are actually needed. The following code illustrates one way to do that:

```ruby
class LazyObject < BasicObject
  def initialize(&callback)
    @callback = callback
  end

  def __result__
    @__result__ ||= @callback.call
  end

  def method_missing(*a, &b)
    __result__.send(*a, &b)
  end
end
```

Another option would be to use [lazy.rb](http://moonbase.rydia.net/software/lazy.rb/), which provides similar functionality via `Lazy::Promise` objects that get instantiated via the `Lazy.promise` method:

```ruby
require "lazy"

class Location
  def self.[](x,y)
    @locations        ||= {}
    @locations[[x,y]] ||= new(x,y)
  end 

  def initialize(x,y)
    @x     = x
    @y     = y
    @color = [:green, :green, :blue].sample

    @north = Lazy.promise { Location[@x,@y-1] }
    @south = Lazy.promise { Location[@x,y+1] }
    @east  = Lazy.promise { Location[@x+1, @y] }
    @west  = Lazy.promise { Location[@x-1, @y] }
  end

  def ==(other)
    [x,y] == [other.x, other.y]
  end

  def neighbors
    [north, south, east, west]
  end

  attr_reader :x, :y, :color, :north, :south, :east, :west
end
```

This approach provides a thread safe solution and prevents us from having to reinvent the wheel. The only downside is that _lazy.rb_ is a bit dated and generates some warnings on Ruby 1.9 due to the way it implements its core proxy object. But whether you use _lazy.rb_ or roll your own lazy object, it is important to understand that the difference between this pattern and the common Ruby idiom of delaying execution via cached method calls is more than just a matter of aesthetics. To illustrate why that is the case, we can can consider the difference in behavior between the two approaches when `Location#neighbors` is called.

In the original example that explictly defines the `north`, `south`, `east`, and `west` methods, the first time the `neighbors` method is called, four `Location` objects are created. This means that the following line of code will generate all four neighboring `Location` objects even if not all of them are needed to answer the question it asks:

```ruby
green_neighbor = location.neighbors.find { |loc| loc.color == :green }
```

By contrast, a `Location` object that uses some form of lazy object would behave differently here. Because `Enumerable#find` returns as soon as it finds a single object which matches its conditions, the `Location#color` method will not necessarily get called on each of the neighboring locations. This means that in the best case scenario, only one new `Location` object would end up getting created. While this particular example is a bit contrived, it's not hard to see why this is a desireable characteristic of lazy objects that cannot be easily emulated via the standard Ruby idiom for delayed execution.

### Future objects

Lazy objects provide certain performance benefits in the sense that they make it possible to avoid unnecessary computation, but they don't do anything to improve the perceived waiting time for any computations that actually need to be run. This is where future objects come in handy.

A future object is essentially an object which immediately begins doing some processing in the background but only blocks if the results are demanded before the thread has finished executing. The example below demonstrates how this sort of object can come in handy for building a simple non-blocking download manager:

```ruby
require "open-uri"
require "lazy"

class DownloadManager
  def initialize
    @downloads = []
  end

  def save(url, filename)
    downloads << Lazy.future { File.binwrite(filename, open(url).read) }
  end

  def finish_all_downloads
    downloads.each { |d| Lazy.demand(d) }
  end

  private

  attr_reader :downloads
end

downloader = DownloadManager.new 

downloader.save("http://prawn.majesticseacreature.com/manual.pdf", "manual.pdf")
puts "Starting Prawn manual download"

downloader.save("http://sandal.github.com/rbp-book/pdfs/rbp_1-0.pdf", "rbp_1-0.pdf")
puts "Starting download of Ruby Best Practices book"

puts "Waiting for downloads to finish..."
downloader.finish_all_downloads
```

In this particular example the callback doesn't return a meaningful value, and so the `DownloadManager#finish_all_downloads` method makes use of `Lazy.demand` to force each future to wrap up its computations. However, the following example demonstrates that the future objects that _lazy.rb_ provides can also be used as transparent proxy objects:

```ruby
require "open-uri"
require "lazy"

class Download
  def initialize(url, filename)
    @filename = filename
    @contents = open(url).read
  end

  def save
    File.binwrite(@filename, @contents)
  end
end

class DownloadManager
  def initialize
    @downloads = []
  end

  def save(url, filename)
    downloads << Lazy.future { Download.new(url, filename) }
  end

  def finish_all_downloads
    downloads.each { |d| d.save }
  end

  private

  attr_reader :downloads
end

downloader = DownloadManager.new 

downloader.save("http://prawn.majesticseacreature.com/manual.pdf", 
                "manual.pdf")
puts "Starting Prawn manual download"

downloader.save("http://sandal.github.com/rbp-book/pdfs/rbp_1-0.pdf", 
                "rbp_1-0.pdf")
puts "Starting download of Ruby Best Practices book"

puts "Waiting for downloads to finish..."
downloader.finish_all_downloads
```

In both examples, the future object will block as long as necessary to allow the computations to complete, but only when it is forced to do so. Until this occurs, other operations can continue in parallel and/or block the execution of these future objects. This means in the best case scenario, a computation will end up being completed before it is actually needed, and the results will be returned from the future object's cache at that time.

While implementing a generic future object from scratch would not be difficult for anyone who has experience working with threads, concurrency is a weak point for me and I rather not confuse folks by giving potentially bad advice through a naive implementation of my own. Those who are really itching to see how such an object is implemented should look at the [lazy.rb source code](https://github.com/mental/lazy/blob/master/lib/lazy.rb#L138-146), but if you treat future objects as black boxes you just need to know a few basic things about Ruby's thread model to make use of this construct effectively.

The most important thing to keep in mind is that thread scheduling in standard Ruby is affected by a global interpreter lock (GIL) which makes it so that most computations end up blocking the execution of other threads. Alternative implementations such as JRuby and Rubinius remove this lock, but in standard Ruby this basically means that threads are mostly useful for backgrounding operations such as file and network I/O. This is because unlike most computations, I/O operations will give other threads a chance to run while waiting on their data to become available. Because lazy.rb's implementation is thread based, future objects inherit the same set of restrictions. The other thing to be aware of is that Ruby does not explicitly join all of its threads once the main execution thread completes. This means that if I did not explicitly call `downloader.finish_all_downloads` in the previous example, the threads spun up by my future objects would be terminated if the main thread finished up before the downloads were completed. This may be obvious to anyone with a background in concurrency, but I scratched by head for a bit because of this issue.

Other than those issues, future objects pretty much allow you to solve some basic concurrency problems without knowing a whole lot about how to work with low level concurrency primitives. While the example I've shown here is a bit dull, I can imagine this technique might come in handy for things like sending emails or doing some time intensive computations that are part of an interactive reporting system. Both of these are problems I've had to solve before using tools like [Resque](https://github.com/defunkt/resque), but simple future objects might prove to be a lightweight 
alternative. I'd be curious to hear from our readers who have some concurrency experience whether that seems like a good idea or not, and also whether you have ideas for other potential applications of future objects.

## Reflections

The general concept of wrapping data in results objects isn't that exciting, but the notion of lazy objects and future objects show that results objects can be imbued with rich behaviors that can make our code more flexible and easier to understand. 

While Rubyists are no strangers to benefits of lazy evaluation, the process of writing this article has lead me to believe that we can probably benefit from having some higher level constructs to work with. However, explaining my thoughts on that would take a whole other article.

Similarly, it seems that Ruby provides all the basic tooling necessary for concurrency, even when you take into account the limitations of standard Ruby due to its GIL. It would be nice if we could establish some good patterns and constructs for making this kind of programming more accessible to the amateur. Such constructs may end up hiding some of the details that experienced developers care about, but would likely lead to more performant code without sacrificing maintainability or learnability.

On a closing note, the fact that there are two interesting subcategories of results objects hints that there may be more left to discover. This is similarly true for patterns about arguments. I can feel in my gut that there are other patterns out there just waiting to be discovered, but cannot think of any off the top of my head at the moment. Have you seen anything in the wild that hints at how we can expand on these ideas? If so, please leave a comment! 

