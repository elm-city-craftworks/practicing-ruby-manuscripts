In this two part series, I've been looking at the classical design patterns laid out by the Gang of Four and exploring their relevance to modern day Ruby programming. Rather than teaching the design patterns themselves, my goal is to give practical examples using real code so that you can consider what the strengths and weaknesses of these techniques are.

In this issue, I'll be focusing on the structural design patterns, all of which have to do with the relationships between clusters of code. The seven patterns listed below are the ones we'll focus on in this article.

  * Adapter
  * Bridge
  * Composite
  * Proxy
  * Decorator
  * Facade
  * Flyweight

An important thing to keep in mind is that while the original GoF book was written with C++ and Smalltalk examples, none of the patterns were written with Ruby's semantics in mind. This makes it necessary to use a little poetic license to explore how these patterns apply in Ruby. That having been said, the examples we'll look at attempt to preserve the spirit of the patterns they implement even if they're not semantically identical.

### Adapter 

An [Adapter](http://en.wikipedia.org/wiki/Adapter_pattern) is used when you want to provide a unified interface to a number of different objects that implement similar functionality. This pattern is easy to spot in the wild for a Ruby developer, as most of us have to use ActiveRecord in our day to day work. Under the hood, ActiveRecord talks to a large amount of different databases, but wraps them up in a common interface its implementation code can avoid worrying about their individual differences. This is exactly the sort of thing the Adapter pattern is useful for.

Increasingly, Rubyists are finding this pattern to be useful in other settings as well. In particular, things like Intridea's [multi_json](https://github.com/intridea/multi_json) gem allow libraries and applications to be built against an abstract interface rather than requiring some particular JSON library that might conflict with other dependencies. Inspired by the ideas behind multi_json, a Mendicant University student (Mitko Kostov) built a similar adapter library for Markdown processors called [Marky](https://github.com/mytrile/marky). The base implementation is very simple, and gives us a good opportunity to look at one way to implement an Adapter in Ruby from the ground up.

The basic idea is that we want to be able to use a common interface while easily
configuring which backend is used. The following example shows how Marky might
be used:

```ruby  
# using RDiscount as a backend
Marky.adapter = :rdiscount

Marky.to_html("hello world") #=> "<p>hello world</p>

# using BlueCloth as a backend
Marky.adapter = :bluecloth

Marky.to_html("hello world") #=> "<p>hello world</p>"
```

To make this work, it is necessary to map the name of an adapter to a class which wraps the underlying engine and implements a `to_html()` method. The code that actually wires up the adapter is shown below.

```ruby
module Marky
  extend self

  def adapter
    return @adapter if @adapter
    self.adapter = :rdiscount
    @adapter
  end
   
  def adapter=(adapter_name)
    case adapter_name
    when Symbol, String
      require "adapters/#{adapter_name}"
      @adapter = Marky::Adapters.const_get("#{adapter_name.to_s.capitalize}")
    else
      raise "Missing adapter #{adapter_name}"
    end
  end

  def to_html(string)
    adapter.to_html(string)
  end
end
```

While this uses a tiny bit of dynamic Ruby magic to look up the right module name, when we see the actual adapters, it all comes together.

```ruby
# adapters/bluecloth.rb

require "bluecloth"

module Marky
  module Adapters
    module Bluecloth
      extend self
      def to_html(string)
        ::BlueCloth.new(string).to_html
      end
    end
  end
end
```

```ruby
# adapters/rdiscount.rb

require "rdiscount"

module Marky
  module Adapters
    module Rdiscount
      extend self
      def to_html(string)
        ::RDiscount.new(string).to_html
      end
    end
  end
end
```

Since all the adapters implement a `to_html()` method that share a common
contract, `Marky.to_html()` will work regardless of what adapter gets loaded.
The win here is that if that libraries, applications and frameworks rely on
adapters rather than concrete implementations, it will be easier to swap
one engine out for another when necessary.

While not every problem domain needs added level of indirection that Adapters introduce, they can come in handy when there are several competing implementations solving a common problem but you don't want to forced to choose one over the other.

### Bridge

The idea behind a [Bridge](http://en.wikipedia.org/wiki/Bridge_pattern) is to place a physical separation between the interface of a given type of object and its implementation. In languages in which the type system gets in the way, this kind of pattern is important for reducing the proliferation of various type permutations by making hierarchies of interfaces and implementations orthogonal to one another. If that sounds like academic jibberish to you, there is a [SourceMaking article](http://sourcemaking.com/design_patterns/bridge) that might help sort the concepts out for you a bit.

I had a hard time thinking through where this pattern has its place in Ruby, mainly because there is a built in separation of interface and implementation in Ruby via duck typing semantics. The best example I could come up with was to port the painfully complex C++ example that SourceMaking uses in their article to something that is a bit more natural looking in Ruby. Read it over, and think about what it might be gaining us as you go.

```ruby
##Concrete Implementations

class BasicTimeData
  def initialize(hour, minutes)
    @hour     = hour
    @minutes  = minutes
  end

  def formatted_output
    "Time is #{@hour}:#{@minutes}"
  end
end

class TimeWithMeridianData
  def initialize(hour, minutes, meridian)
    @hour     = hour
    @minutes  = minutes
    @meridian = meridian
  end

  def formatted_output
    "Time is #{@hour}:#{@minutes} #{@meridian}"
  end
end

##Bridge

module TimeFormatter
  def to_s
    @time_data.formatted_output
  end
end

## Abstract Objects linked to Concrete Implementations through Bridge

class BasicTime
  include TimeFormatter
  
  def initialize(*a, &b)
    @time_data = BasicTimeData.new(*a, &b)    
  end
end

class TimeWithMeridian
  include TimeFormatter

  def initialize(*a, &b)
    @time_data = TimeWithMeridianData.new(*a, &b)
  end
end

## Example Usage

time1  = BasicTime.new("10","30")
time2  = TimeWithMeridian.new("10","30","PM")

[time1, time2].each { |t| puts t }
```

While it might just due to the nature of the example, I feel this code is still quite contrived, and that hides its benefits. The takeway here is mostly that it's possible for both `BasicTimeData` and `TimeWithMeridianData` to change without breaking `BasicTime` and `TimeWithMeridian`, as long as `TimeFormatter` is updated. Similarly, a change in the needs of `BasicTime` and `TimeWithMeridian` will not affect the concrete implementations of the data structures, as long as the bridge provides the necessary wiring.

The thing that I struggle with in this pattern is understanding what unique behaviors the `BasicTime` and `TimeWithMeridian` objects could implement that justify their existence. Is this pattern an artifact of static languages that isn't really needed in Ruby? Or am I just missing some important detail that could be cleared up with a good example or two? Feel free to leave a comment letting me know what you think.

### Composite 

The [Composite pattern](http://en.wikipedia.org/wiki/Composite_pattern) is useful when you want to treat a group of objects as if it were a single, unified object. To explore this pattern, we can look at some experimental code I wrote for Prawn which was designed to make it possible to treat all objects drawn in the document as compositions of primitive PDF instructions. We can start with an example of rendering a curve, working from the outside in.

When `curve()` is called on a Prawn drawing, the PDF content is generated as a
side effect and the user does not need to do anything with the return value of
that method. However, if someone wanted to play with the internals, they could
call `curve!()` instead and get themselves an object that implements a
`to_pdf()` method, as in the following example:

```ruby
chunk = canvas.curve!(:point1 => [100,100],
                      :point2 => [50,50], 
                      :bound1 => [60,90],
                      :bound2 => [60,90])

puts chunk.to_pdf

...........................................................OUTPUTS..

  100.000 100.000 m
  60.000 90.000 60.000 90.000 50.000 50.000 c
```

We can catch a glimpse of how this `to_pdf()` method is actually implemented via several composed primitive objects by taking a look at the source for the `curve!()` method.

```ruby
def curve!(params)
  chunk(:curve, params) do |c|
    [ move_to!(:point => c[:point1]),
      curve_to!(:point => c[:point2],
                :bound1 => c[:bound1],
                :bound2 => c[:bound2]) ]

  end
end
```

In the above, `chunk()` is a helper method which builds a `Prawn::Core::Chunk` object, which serves as the composite object in this system. We'll look at how chunks are implemented in a bit, but note that the block given to the `chunk()` method defines what the individual chunk is composed of. In this case, a curve consists of two subchunks, one responsible for generating a move instruction, and one for rendering a curve from the current drawing location to another point. The definitions for both of those methods are shown below.

```ruby
def move_to!(params)
  chunk(:move_to, params) do |c|
    raw_chunk("%.3f %.3f m" % c[:point])
  end
end

def line_to!(params)
  chunk(:line_to, params) do |c|
    raw_chunk("%.3f %.3f l" % c[:point])
  end
end
```

Similar to `chunk!()`, these methods can called directly, producing an object with a `to_pdf()` method, as shown below.

```ruby
chunk = canvas.move_to!(:point => [100,100])
puts chunk.to_pdf

chunk2 = canvas.curve_to!(:point => [50,50],
                          :bound1 => [60,90],
                          :bound2 => [60,90])

puts chunk2.to_pdf

# combined output is identical to calling curve!() directly.
```

Through these two methods, we encounter the leaf nodes in our composition, `Prawn::Core::RawChunk` objects. These objects are where the actual text that is in our `to_pdf()` is stored. With that in mind, we can now look at the actual objects that this graphics system is built on.

```ruby
module Prawn
  module Core
    class RawChunk
      def initialize(content)
        @content = content
        @command = :raw_pdf_text
        @params = {}
      end

      attr_accessor :content, :command, :params
      alias_method :to_pdf, :content
    end

    class Chunk
      def initialize(command, params={}, &action)
        @command = command
        @params = params
        @action = action
      end

      attr_reader :command, :params, :action

      def content
        action.call(self)
      end

      def to_pdf
        case results = content
        when Array
          results.map { |sub_chunk| sub_chunk.to_pdf }.join("\n")
        when Prawn::Core::Chunk, Prawn::Core::RawChunk
          results.to_pdf
        else
          raise "Bad Chunk: #{results.class} not supported"
        end
      end
    end
  end
end
```

From this we see that a chunk's content can be an array of children, a chunk, or a raw chunk object. The `to_pdf()` method is responsible for traversing downwards through the composition until it reaches the raw chunk objects which simply return the raw content. But because the APIs match, we can look at the chunks at any level of the system and burrow down to their raw data.

While this might be a bit of an intense example, it shows how the Composite pattern can be used to make a complex set of objects look and feel as if they were a single unified entity. It may be a bit much to take in on a first glance, but try to think of how you could apply these ideas to your own code and you might gain some useful insights.

### Proxy

A [Proxy](http://en.wikipedia.org/wiki/Proxy_pattern) is any object that acts as a drop-in replacement object that does a bit of work and then delegates to some other underlying object. This is another pattern that's easy to recognize for Rails developers, because it is used extensively in ActiveRecord associations support. The following example shows a typical interaction between a base model and one of its associations.

```ruby
quiz = Quiz.first
quiz.questions.class #=> Array

quiz.questions.count #=> 10

quiz.questions.create(:question_text => "Who is the creator of Ruby?",
                      :answer        => "Matz")

quiz.questions.count #=> 11

quiz.questions[-1].answer #=> "Matz" 
```

While the questions association claims to be an array, it also provides some of the common helpers found in `ActiveRecord::Base`. If we stick to the core idea and ignore some of the Rails specific details, creating such an association proxy is fairly easy to do using Ruby's delegate standard library. The code below more or less does the trick.

```ruby
require "delegate"

class Quiz
  def questions
    @questions                  ||= HasManyAssociation.new([])
    @questions.associated_class ||= Question

    @questions
  end
end

class Question
  def initialize(params)
    @params = params
  end

  attr_reader :params

  def answer
    params[:answer]
  end
end

class HasManyAssociation < DelegateClass(Array)
  attr_accessor :associated_class

  def initialize(array)
    super(array)
  end

  def create(params)
    self << associated_class.new(params)
  end
end

quiz = Quiz.new

# grab the proxy object
questions = quiz.questions


# use custom create() method on proxy object

questions.create(:question_text => "Who is the creator of Ruby?",
                 :answer        => "Matz")
questions.create(:question_text => "Who is the creator of Perl?",
                 :answer        => "Larry Wall")


# use array like behavior on proxy object

p questions[0].answer #=> "Matz"
p questions[1].answer #=> "Larry Wall"
p questions.map { |q| q.answer }.join(", ") #=> "Matz, Larry Wall"
```

Interestingly enough, while Ruby provides a standard library for building Proxy objects, most people tend to implement them in a different way, through the use of an explicit `method_missing()` hook on a blank slate object such as Ruby 1.9's BasicObject. For example, we could have written our HasManyAssociation code in the manner shown below and things would still work as expected.

```ruby
class HasManyAssociation < BasicObject
  attr_accessor :associated_class

  def initialize(array)
    @array = array
  end

  def create(params)
    self << associated_class.new(params)
  end

  def method_missing(*a, &b)
    @array.send(*a, &b)
  end
end
```

Without looking at the source, I'm almost sure that Rails does something similar to this, because doing some_association.class returns Array rather than the name of the proxy object. This is the only noticeable difference between this approach and the DelegateClass approach.

Personally, I've written proxies in both ways, and I tend to prefer the `DelegateClass()` approach slightly, simply because it's more explicit and doesn't require me to explicitly define a `method_missing()` hook. But on the other hand, we can see that rolling your own proxy implementation is trivial in Ruby, and some may prefer the self contained nature of doing the delegation work yourself. It'd be interesting to hear what readers have to say on this topic, so please feel free to post to the mailing list if you prefer one approach over the other.

### Decorator

While there is a clear distinction between a [Decorator](http://en.wikipedia.org/wiki/Decorator_pattern) and a Proxy in static languages, in Ruby the two concepts almost merge, except that a Decorator is used for the purpose of adding / extending behavior of a target object, and a Proxy is a more general concept.

Since I've already written up a cool example of using decorators on this blog, I think what I'll do is point you over there in the interest of keeping this article from being even more incredibly long than it already is. Check out the [Decorator Delegator Disco](http://blog.rubybestpractices.com/posts/gregory/008-decorator-delegator-disco.html) if you want to see some interesting code samples that implement this pattern.

### Facade

The [Facade pattern](http://en.wikipedia.org/wiki/Facade_pattern) simplifies how users interact with a codebase by implementing an interface that hides many implementation details for the most common behaviors needed by consumers. Looking at it from the outside, a perfect example of this is the open-uri standard library. When using open-uri, it is possible to do a simple HTTP get request with just a single line of code, as shown below.

```ruby
require "open-uri"

puts open("http://www.google.com").read
```

The purpose of open-uri is to make reading a resource via an HTTP GET request look and feel like opening an ordinary file. This is a very common use case, so open-uri makes it easy to work with. To see what this facade is hiding, we can write the equivalent code using the libraries it wraps, `Net::HTTP` and `URI`.

```ruby
require 'net/http'
require 'uri'

url = URI.parse('http://www.google.com')

res = Net::HTTP.start(url.host, url.port) {|http|
  http.get('/index.html')
}

puts res.body
```

While the code above isn't especially difficult to read, it certainly feels like more work than the previous example. This is primarily because of the flexibility and functionality tradeoff between the two approaches. The open-uri library can afford to be much more high level and limited in scope because its sole purpose is to help make a single particular task easier. On the other hand, `Net::HTTP` and `URI` are both complex tools that can be used for a number of different things. The use of the Facade pattern allows for both kinds of objects to coexist peacefully within a single system.

It's worth keeping in mind that pretty much every DSL you encounter is a Facade of some form or another. If you're interested in seeing how simple interfaces can mask highly complex networks of objects, consider doing a source dive into your favorite tool that utilizes a domain specific language, such as Rake, RSpec, or Sinatra. You'll find a number of different techniques at work depending on which project you explore, but all have the common characteristic of providing a simplified way to interact with a deep system.

### Flyweight

The [Flyweight pattern](http://en.wikipedia.org/wiki/Flyweight_pattern) is a way to represent what would seem like a large amount of data in a lightweight way. This is one of those patterns that mostly goes away in Ruby due to built in language constructs, but it's worth taking a look at just for the sake of completeness.

The wikipedia article linked above discusses font metrics as a common application of the flyweight pattern, in which you may want to associate each character in a string with a large amount of information describing how that character should be rendered. The basic idea is that it'd be far too inefficient memory-wise to create a new instance of font metrics data for every character in a document. So instead, using the Flyweight pattern, it is possible to map the index of characters in a string to a single instance for each unique character, vastly reducing the amount of memory consumed. This is a problem I've actually had to solve before within Prawn, but it's a bit of a tough one demonstrate briefly if we attempt to show real code.

However, if we stub out the actual font metrics generation, it's easy to see that this problem can be solved by wrapping a simple Ruby hash that has an initializer block.

```ruby
class FontMetrics

  def initialize(string)
    @string = string
  end

  def [](index)
    glyph_data[@string[index]]
  end

  def glyph_data
    @glyph_data ||= Hash.new { |h,k| h[k] = metrics_for(k) }
  end

  # stubbed out, but would typcially refer to something
  # largish and time consuming to generate
  # 
  def metrics_for(k)
    { :bbox => [rand(100), rand(100), rand(100), rand(100)] }
  end

end

string = "Hello World"

metrics = FontMetrics.new(string)

p metrics[0]  #=> {:bbox=>[86, 44, 88, 31]}
p metrics[2]  #=> {:bbox=>[52, 7, 38, 98]}
p metrics[3]  #=> {:bbox=>[52, 7, 38, 98]}
p metrics[-2] #=> {:bbox=>[52, 7, 38, 98]}

p metrics[2].object_id == metrics[3].object_id #=> true

p metrics[0] == metrics[1] #=> false
p metrics[2] == metrics[3] #=> true
```

From the above code, we see that the `FontMetrics` object gives the appearance of having data for each and every character in the string, but checking the `object_id` for each proves that only one object per unique character has been created. While I suppose we could call this a Flyweight, I think that in Ruby we'd just say that we were caching the metrics data using a hash with an initializer. But perhaps using this vocabulary wouldn't hurt, if we want to keep our minds focused on the high level concepts.

### Reflections

After writing two articles on the the topic, I'm finding myself getting sick of design patterns. But when I look back on the code I've shared with you, I realize that when I built these things, I never really thought consciously about what pattern they were, and it took me until the time of writing this article to put a name on them.

A major concern I have about classical patterns is that in order to see the resemblance of the code I've shared to the original GoF patterns, you really need to look at things sideways and squint a bit. It'd be great for me to be able to just say 'Use a flyweight here' and have it mean something, but if you say that to someone without a strong background in Ruby, you may end up with hundreds of lines of a Java-esque monstrosity.

To be sure, I'm not saying that this exploration has not been worthwhile. Forcing myself to think of how many of the classic GoF patterns might materialize themselves in Ruby has been a very interesting experience, because it's really making me think about how we build and design our code. But the problem of whether we can actually have a common vocabulary about concepts that really get distorted in translation makes me wonder about the merits of stacking up pattern definitions, at least without giving them new names.

I'm quite curious about what folks have been thinking as they read through the last two articles, in particular, I wonder if seeing me try to attack patterns from a purely pragmatic perspective has changed anything about the way readers look at these concepts. I'm also kind of curious if 'that guy' is out there, silently thinking to himself "Well... actually" after seeing my somewhat liberal interpretation of these classic patterns. Please leave a comment letting me know what you think!
  
> **NOTE:** This article has also been published on the Ruby Best Practices blog. There [may be additional commentary](http://blog.rubybestpractices.com/posts/gregory/060-issue-26-structural-design-patterns.html#disqus_thread) 
over there worth taking a look at.
