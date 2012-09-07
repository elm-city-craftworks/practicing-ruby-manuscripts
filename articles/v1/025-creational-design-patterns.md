In this issue and the next one, I look at several design patterns laid out by
the Gang of Four and explore their relevance to modern day Ruby programming. My
goal is not so much to teach the design patterns themselves, but instead give
practical examples using real code so that you can consider what their strengths
and weaknesses are.

In this article, I focus on the creational design patterns, all of which have to
do with instantiating new objects. They are listed below in no particular order:

  * Singleton
  * Multiton
  * Factory Method
  * Abstract Factory
  * Builder
  * Prototype  

An important thing to keep in mind is that while the original GoF book was
written with C++ and Smalltalk examples, none of the patterns were written with
Ruby's semantics in mind. This makes it necessary to use a little poetic license
to explore how these patterns apply in Ruby. That having been said, the examples
we'll look at attempt to preserve the spirit of the patterns they implement even
if they're not semantically identical.

### Singleton

<i>UPDATE 2011-12-20: Be sure to check out what [Practicing Ruby
2.8](http://practicingruby.com/articles/shared/jleygxejeopq) has to say about
implementing Singleton pattern in Ruby. My feelings on this have changed over
time</i>

The [Singleton pattern](http://en.wikipedia.org/wiki/Singleton_pattern) is used
in situations where a single instance of a class is all you need. Singleton
objects are meant provide an effective way of organizing global state and
behavior, such as configuration data, logging support, or other similar needs.
This pattern is common enough that Ruby provides a module in its standard
library that makes it easier to implement.

In the code below, I've implemented a simple logger using Ruby's singleton
library, which on the surface should look quite like an ordinary Ruby class
definition.

```ruby
require "singleton"

class SimpleLogger
  include Singleton

  def initialize
    @output = []
  end

  attr_reader :output

  def error(message)
    output << formatted_message(message, "ERROR")
  end

  def info(message)
    output << formatted_message(message, "INFO")
  end

  def write(filename)
    File.open(filename, "w") { |f| f << output.join("\n") }
  end

  private

  def formatted_message(message, message_type)
    "#{Time.now} | #{message_type}: #{message}"
  end

end
```

Nothing here looks out of the ordinary, but you can begin to see the impact of the Singleton mixin as soon as you try to create a SimpleLogger instance.

```ruby
>> logger = SimpleLogger.new
NoMethodError: private method `new' called for SimpleLogger:Class
  from (irb):2
```

Since the purpose of the singleton pattern is to allow for the creation of only a single instance, it makes sense that you might not be able to expect construction to work as normal. The example below demonstrates what you need to do instead.

```ruby
logger = SimpleLogger.instance

logger.error("Some serious problem")
logger.info("Something you might want to know")
logger.write("log.txt")
```

The class method `instance()` gets added by the Singleton mixin, and takes responsibility for initializing the SimpleLogger object. The first time this method is called, an instance of SimpleLogger is created, and initialize gets called as usual. Each subsequent call refers to that exact instance, preventing additional instances of this class from being created.

At this point, you might be wondering what the advantage of this approach is
over using ordinary class definitions and global variables, such as in the code
below:

```ruby
$LOGGER = SimpleLogger.new
$LOGGER.error("Some serious problem")
$LOGGER.info("Something you might want to know")
```

This is a reasonable question to ask, because this approach is just as straightforward and has similar strengths and weaknesses. If Ruby did not have an open class system, you could argue that it's less likely that you'd run into side effects with different definitions of SimpleLogger than you would with someone reassigning $LOGGER, but that clearly does not apply here. It does seem like it'd be marginally more likely to see a variable reassigned than a class stomped on, particularly if proper namespacing was used, but it's important to notice that in Ruby, this particular benefit of Singleton objects is marginal at best.

The real benefit that using the Singleton pattern provides in Ruby over its alternatives is that instantiation is lazy evaluated, and enforces the single instance limitation. The former provides a potential performance and memory bonus when the object never ends up getting used, and the latter helps prevent accidental object creation. Both of these things are nice to have, and it only takes a bit of extra effort make them happen.

The standard library implementation of the Singleton pattern is reasonable, and fairly clever. However, I feel it probably leans a bit too closely to a direct translation, and instead typically use a different technique in my own code. Below you can see how I might write SimpleLogger if left to my own devices.

```ruby
module SimpleLogger
  extend self

  def error(message)
    output << formatted_message(message, "ERROR")
  end

  def info(message)
    output << formatted_message(message, "INFO")
  end

  def write(filename)
    File.open(filename, "w") { |f| f << output.join("\n") }
  end

  private

  def formatted_message(message, message_type)
    "#{Time.now} | #{message_type}: #{message}"
  end

  def output
    @output ||= []
  end
end
```

This code preserves the lazy evaluation component while removing the concept of an instance entirely. The module itself becomes a singleton object, allowing you to call methods directly on it which affect the state stored on the module.

```ruby
SimpleLogger.error("Some serious problem")
SimpleLogger.info("Something you might want to know")
SimpleLogger.write("log.txt")
```

I like to use this approach because what I end up with is truly a single object, rather than a class who's job is to provide a single instance. I also like to be able to call my singleton methods directly on that object without having to retrieve the singular instance.

The downside of this code, and the reason why I showed both approaches instead of just my preferred one, is that it requires a much greater amount of Ruby knowledge to actually understand how it works. Additionally, through the loss of the `initialize()` hook, you need to either do something like create a setup method that you explicitly call before any other methods, or do lazy initialization of all data as I've done in the `output()` method. These idioms are fairly common, but again, take you a bit farther away from the look and feel of an 'ordinary class definition', which may be offputting to some.

No matter which approach you choose, it's worth keeping in mind that the Singleton pattern should be used in moderation. Due to Ruby's open class system, singleton objects are essentially nothing more than overengineered global variables. As with any form of global state, this makes them more difficult to test, easier to corrupt, and harder to isolate when it comes time to debug issues with them. However, when used sparingly and for the right reasons, they can be a good tool for managing global state and interactions.

### Multiton

The [Multiton pattern](http://en.wikipedia.org/wiki/Multiton) is an extension to the Singleton pattern which maps unique keys to specific instances of a class. The idea is that there should only be one instance of an object for each unique key in use, limiting the number of objects that need to be created. For a practical example, we can look at some code in the PDF generation library Prawn which implements this sort of behavior.

As a PDF generation library that operates at the very low level, we need to process and utilize font metrics information extensively. Because font files can become quite large, the processing cost of initializing one of our Font objects can be very computationally expensive. However, since this information is unique for each font that we use, there is no need to re-instantiate fonts once they have been initialized.

While the actual source code is more complex than what I'll show here, the basic idea of applying a multiton here is quite simple. We can start off by assuming that our `Font` class has an ordinary constructor which takes a font file and then processes it into a meaningful dataset. Initializing a font directly then looks something like this.

```ruby  
font = Font.new("/path/to/times.ttf")
```

On top of this basic interface, we layer a mapping mechanism that when used, looks something like the code below.

```ruby
Font.map("Times New Roman" => "/path/to/times.ttf")
Font["Times New Roman"] #=> A font instance
```

To bring all of the above together, we can take a look at a skeletal `Font` class which implements the Multiton pattern.

```ruby
class Font
  class << self
    def file_names
      @file_names ||= {}
    end

    def instances
      @instances ||= {}
    end

    def map(params)
      file_names.update(params)
    end

    def [](name)
      instances[name] ||= new(file_names[name])
    end
  end

  def initialize(filename)
    puts "processing #{filename}"
  end

  # details omitted
end
```

While the mapping is a bit complex here because it's a two stage process, the core idea of the Multiton still shines through. What we have are lazy evaluated Font instances which do not get created until the first time `Font[some_font_name]` is called. Each subsequent call will result in the original instance being returned rather than a new instance of Font being created.

This basic pattern can be used any time you have a scenario in which each unique key can be mapped to exactly one object. This sort of structure can be a super effective caching technique, but should also be used with caution as it too introduces global state and added complexity that should not be taken lightly.

### Factory Method

The [Factory Method pattern](http://en.wikipedia.org/wiki/Factory_method) is used for putting a layer of abstraction on top of object creation so that directly working with its constructor is no longer necessary. This process can lead to more expressive ways of building new objects, and can also allow for the creation of new objects without explicitly referencing their class.

In a Mendicant University course, one of our students (Carol Nichols) ran into a design issue that we were able to improve by introducing factory methods. She was building an `AdjacencyMatrix` datastructure for storing graph data, and her original API looked like this:

```ruby
class AdjacencyMatrix
  def initialize(data, directed=false)
    #...
  end
end

undirected_matrix = AdjacencyMatrix.new(data) 
directed_matrix   = AdjacencyMatrix.new(data, true)
```

Using boolean switches as arguments aren't especially expressive, and so the suggestion was quickly made to move towards a hash-based argument allowing for the following usage.

```ruby
undirected_matrix = AdjacencyMatrix.new(data) 
directed_matrix   = AdjacencyMatrix.new(data, :directed => true)
```

Generally speaking, this isn't a bad strategy, but the more we thought about it, the more we realized that there isn't really a need for a dynamic option here. Consumers would always know whether a graph they were trying to represent was directed or undirected at the time they constructed their object, and the default call without the `:directed` argument still wasn't very expressive about this detail. After some discussion, we settled on the following design, which introduces the Factory method pattern.

```ruby
class AdjacencyMatrix
  class << self
    def undirected(data)
      new(data)
    end

    def directed(data)
      new(data,true)
    end

    private :new
  end

  def initialize(data, directed=false)
    #...
  end
end

undirected_matrix = AdjacencyMatrix.undirected(data) 
directed_matrix   = AdjacencyMatrix.directed(data)
```

While this code does still have its original wart intact internally, the interface that consumers interact with has been greatly improved. By making the `new()` method private at the class level, we force users to call the factory methods, and it becomes immediately clear what type of graph is being processed each time a new `AdjacencyMatrix` is constructed.

This type of scenario comes up often, and the use of factory methods can be used
to simplify or at least hide the complexity of constructor calls by giving an
expressive name to a certain way of constructing a given object. However,
sometimes factories go even farther by completely decoupling the object creation
process from the class of the object being created. While I won't go into much
detail about this rare use case, we see this sort of factory very
often in test code, particularly when dealing with object mappers in web 
applications.

The following example from `factory_girl` demonstrates how code which does not
reference a particular class can be used to instantiate records 
via ActiveRecord:

```ruby
factory :user do
  first_name 'John'
  last_name  'Doe'
  admin false
end

FactoryGirl.build(:user) #=> an instance of the User model
```

I won't elaborate on how this works, but it should serve as food for thought,
and perhaps would be a good project to [dig into the
source](https://github.com/thoughtbot/factory_girl) so that you can get a sense
of how factory methods can simplify the object creation process while also making 
it more flexible.

### Abstract Factory

Due to the flexible nature of Ruby's type system, we don't need an actual language construct for abstract classes. With that in mind, when we note that an [Abstract Factory](http://en.wikipedia.org/wiki/Abstract_factory) is simply an abstract interface for concrete Factory objects to conform to, this pattern pretty much falls away. This is perhaps easier to show than to explain, so I will go ahead and build out a Ruby version of the example shown in the wikipedia article I've linked to above.

```ruby
module OSXGuiToolkit
  extend self

  def button
    Button.new
  end
  
  class Button
    def paint
      puts "I'm an OSX Button"
    end
  end
end

module WinGuiToolkit
  extend self
  
  def button
    Button.new
  end

  class Button
    def paint
      puts "I'm a WINDOWS button"
    end
  end
end

class Application
  def initialize(gui_toolkit)
    button = gui_toolkit.button
    button.paint
  end
end

# this check is a very quick hack, not reliable.
if PLATFORM =~ /darwin/
  Application.new(OSXGuiToolkit)
else
  Application.new(WinGuiToolkit)
end
```

In this example, you'll see that we've eliminated the explicit Abstract Factory interface. Instead, what we've done is created two concrete object factories, `OSXGuiToolkit` and `WinGuiToolkit`, that implement a common API. We then create a simple Application stub class which shows that the GUI toolkit factory should be injected into the Application. The reason for this is seen in the final bit of code which determines which toolkit to use based on the platform the code is running on.

The notion of using dependency injection in combination with object factories is an interesting one to me. I honestly don't find myself writing code like this often at all (and so couldn't share a relevant example of my own), but I'm not sure if that's just because I don't think of it in times where it might be the right strategy to use. In any case, this hopefully demonstrates that the notion of an Abstract Factory in Ruby is a conceptual one that has no need for actual abstract classes or interface objects.

### Builder

The purpose of the [Builder pattern](http://en.wikipedia.org/wiki/Builder_pattern) is to create an abstract blueprint describing the steps of creating an object, and then allow many different implementations to actually carry out that process as long as they provide the necessary steps. This is once again a pattern that in Ruby is purely conceptual due to the lack of a need for explicit interfaces or abstract classes. However, unlike the Abstract Factory pattern, I have actually used the ideas behind builder in some real code of my own, and have an example handy that captures the spirit of the pattern.

A while back, I was experimenting with coming up with some generalized domain language for producing output in a number of different formats. While I had more complex goals in mind, the basic usage ended up looking something like what you see below.

```ruby
class ListFormatter < Fatty::Formatter
  format :text do
    def render
      params[:data].map { |e| "* #{e}" }.join("\n")
    end
  end

  format :html do
    def render
      list_elements = params[:data].map { |e| "<li>#{e}</li>" }.join
      "<ul>#{list_elements}</ul>"
    end
  end
end
```

With some formats defined, the ListFormatter class can be used in the following manner.

```ruby
  data = %w[foo bar baz]
  [:html, :text].each do |format|
    puts ListFormatter.render(format, :data => data)
  end
```

OUTPUTS:

```
  * foo
  * bar
  * baz
  <ul><li>foo</li><li>bar</li><li>baz</li></ul>   
```

While this all may look a bit magical on the surface, the implementation is just a few lines of relatively dull dynamic Ruby code.

```ruby
module Fatty
  class Formatter
    
    class << self    
      def formats
        @formats ||= {}
      end
    
      def format(name, options={}, &block)
        formats[name] = Class.new(Fatty::Format, &block)
      end
        
      def render(format, params={})        
        format_obj = formats[format].new
        format_obj.params = params
        format_obj.render
      end   
    end
  end
  
  class Format
    attr_accessor :params 
  end
end
```

The bulk of this code looks like my Multiton example, albeit one that operates on anonymous Class objects. But instead of focusing on that detail, you should turn your attention to the `render()`, which is where the builder process comes in.

Each time render is called, a new instance of an anonymous Format subclass is created, and then customized. The params attribute is set and the render method is called to return the finally constructed object, our output data. As long as each format object that is implements all the required steps, they can be used interchangeably, which is the key feature that the Builder pattern emphasizes.

This example perhaps would be more convincing if you took a look at it in its real setting, in which I do things like call a validations hook, mix in helper methods to the format instances, and otherwise perform more interesting operations. But since we're running on the long side already with this article, I'll invite you to investigate this on your own by checking out [the source code](https://github.com/sandal/fatty) I wrote a couple years ago for this experiment.

It's worth noting that this isn't a perfect example of Builder because I think the pattern doesn't apply directly to Ruby, and that the code I've demonstrated is a bit of a hack. But it might be a good conversation starter at least for those who are interested in investigating this pattern further, since it tries to attack a problem that the Builder pattern would be well suited for in a static language.

### Prototype 

I've included a reference to the [Prototype pattern](http://en.wikipedia.org/wiki/Prototype_pattern) here for completeness, but unfortunately am unsure whether or not it is remotely applicable to Ruby. The purpose of the Prototype pattern is to provide an alternative to manually constructing new objects by allowing for customization through the cloning of existing objects. Presumably this is a good idea when the setup of an object is more costly than it would be to create a copy of the initial post-processed data. However, I'm coming up with a hard time seeing when this approach would be better than ordinary object composition combined with some side effects free operations done on the source data.

I think what's interesting about the Prototype pattern is that it gives you a different way of looking at object oriented programming which allows you to envision an object system without needing the concept of classes. There are two languages I know of that are specifically designed to implement that kind of environment, [Self](http://en.wikipedia.org/wiki/Self_%28programming_language%29) and [Io](http://en.wikipedia.org/wiki/Io_%28programming_language%29).

Because I feel out of my depth when it comes to suggesting a good use of this pattern in Ruby, I welcome readers to submit their own ideas. Personally, I feel like this pattern isn't a good fit for Ruby, but I could be wrong and would love to see evidence of my own ignorance!

### Reflections

Every example I've shown in this article reflects a design approach that I feel is elegant compared to the obvious alternatives. But each and every one of them requires a lot of explanation and assume more than a fair bit of Ruby and object oriented programming knowledge. For this reason, it always baffles me when beginners or even some intermediate developers bother to learn about patterns at all. Since they don't really make sense to think about until you run into fairly complex problems, and when you do need to apply them you need to be very careful not to overengineer things, they seem like a tool to be used sparingly by strongly skilled developers rather than gratuitously by the masses.

That said, I think that it's nice to be able to recognize a pattern when you see it, regardless what your level of experience is. It's also good to have at least a working knowledge of how to implement patterns so that when someone is discussing design with you, you can have a common vocabulary to work with. For these reasons, the study of patterns might be relevant to all active developers.

While there are clearly lots of different ways to abstract object creation, there are even more ways to create interesting compositions of object clusters. In the next article we'll explore that topic by looking at the structural design patterns.
  
> **NOTE:** This article has also been published on the Ruby Best Practices blog. There [may be additional commentary](http://blog.rubybestpractices.com/posts/gregory/059-issue-25-creational-design-patterns.html#disqus_thread) 
over there worth taking a look at.
