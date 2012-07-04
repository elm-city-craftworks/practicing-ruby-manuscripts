When Mike Burns outlined his vision of [Unobtrusive Ruby](http://robots.thoughtbot.com/post/10125070413/unobtrusive-ruby), I initially thought it was going to be a hit with the community. However, a lack of specific examples led to a critical backlash and caused the post to generate more heat than light. This is unfortunate, because the ideas he outlined are quite valuable and shouldn't be overlooked.

In this article, I share my own interpretation of what Unobtrusive Ruby means, based on the the points that Mike outlined. I can't guarantee that my take on this is what Mike had in mind, but it should be interesting to those who wanted a better-defined roadmap than what he provided. To get this most out of this article, I recommend going back and [reading what Mike wrote](http://robots.thoughtbot.com/post/10125070413/unobtrusive-ruby) before continuing. Try to think about what these concepts mean to you, and then compare them to what I've outlined here.

The following guidelines are the ones Mike laid out, but I've replaced his explanations with my own in the hopes that attacking these ideas from a second angle will be useful.

## Take objects, not classes 

Because Ruby lets us pass classes around like any other object, we have a way to do dependency injection that isn't as common in other languages. For example, we can write something like the following code:

```ruby
class Roster
 def initialize          
    @participants = []  
  end  
  
  def <<(new_participant)  
    @participants << new_participant  
  end  
  
  def participant_names  
    @participants.map { |e| e.full_name }  
  end  
  
  def print(printer=RosterPrinter)
    printer.new(participant_names).print
  end    
end

class RosterPrinter  
  def initialize(participant_names)  
    @participant_names = participant_names  
  end  
  
  def print
    puts "Participants:\n" +  
    @participant_names.map { |e| "* #{e}" }.join("\n")  
  end  
end 
```

This feels clever, but this form of dependency injection is more brittle than it
needs to be. A [recent conversation with Derick
Bailey](http://blog.rubybestpractices.com/posts/gregory/055-issue-23-solid-design.html#comment-317367342)
on my [SOLID
article](http://blog.rubybestpractices.com/posts/gregory/055-issue-23-solid-design.html)
about a very similar example made me realize just how much we tend to pass
classes around unnecessarily when we could be directly passing the objects our
class depends on. With that in mind, Derick helped me refactor the previous example to the more flexible design shown here. 

```ruby
class Roster
 def initialize          
    @participants = []  
  end  
  
  def <<(new_participant)  
    @participants << new_participant  
  end  
  
  def participant_names  
    @participants.map { |e| e.full_name }  
  end  
  
  def print(printer=RosterPrinter.new)
    printer.print(@participants)
  end    
end

class RosterPrinter    
  def print(participant_names)
    puts "Participants:\n" +  
    participant_names.map { |e| "* #{e}" }.join("\n")  
  end  
end 
```

Although this is a subtle change, it has a major impact on the way `Roster` and its printer object relate to one another. The `Roster#print` method originally had a dependency on both the constructor of its printer object and its `print()` instance method. Our new code reduces that coupling by depending only on the existence of a `print()` method in the general case. The following examples demonstrate the added flexibility that this new approach offers us.

```ruby
# does not provide a .new() method
module FunctionalPrinter
  def self.print(participant_names)
    puts "Participants:\n" +  
    participant_names.map { |e| "* #{e}" }.join("\n")  
  end
end

require "prawn"

# has a different constructor than RosterPrinter
class PDFPrinter
  def initialize(filename)
    @document = Prawn::Document.new
    @filename = filename
  end

  def print(participant_names)
    @document.text("Participants", :size => 16)

    participant_names.each do |e|
      @document.text("- #{e}")
    end

    @document.render_file(@filename)
  end
end

roster = Roster.new
roster << "Gregory Brown" << "Jia Wu" << "Jordan Byron"

puts "USING DEFAULT"
roster.print

puts "USING FUNCTIONAL PRINTER"
roster.print(FunctionalPrinter)

puts "USING PDF PRINTER (see roster.pdf)"
roster.print(PDFPrinter.new("roster.pdf"))
``` 

Both `FunctionalPrinter` and `PDFPrinter` demonstrate corner cases that our original code did not account for. By following the guideline of accepting objects rather than classes as the arguments to our methods, our new design can accomodate these types of objects without modification of the `Roster#print` code. As you can see from these examples, this way makes life easier for us. 

## Never require inheritance

Some libraries that we use strongly encourage us to create subclasses of the objects they provide; others flat-out force us to do so. `ActiveRecord` is an example of a library that is almost useless without inheritance but is also very complicated and would be hard to use as an example. The following code is a much simpler example of a tool that expects you to use subclasses of its provided `Plugin` class to get the job done.

```ruby
module Inspector
  def self.analyze(data)
    Plugin.registered_plugins.each { |e| e.new.analyze(data) }
  end

  class Plugin
    def self.inherited(base)
      registered_plugins << base
    end

    def self.registered_plugins
      @registered_plugins ||= []
    end

    def analyze(data)
      raise NotImplementedError
    end
  end
end

class WordCountPlugin < Inspector::Plugin
  def analyze(data)
    word_count = data.split(/ /).length
    puts "Content contained #{word_count} words"
  end
end

class WordLengthPlugin < Inspector::Plugin
  def analyze(data)
    longest = data.split(/ /).map { |e| e.length }.max
    puts "Longest word contained #{longest} characters"
  end
end

Inspector.analyze("This is a test of the watcher plugins")

## OUTPUTS
Content contained 8 words
Longest word contained 7 characters   
```

Using the `inherited` hook is a clever way to implictly register plugins, but this approach comes with a number of downsides. For example, you are forced to make all your plugins inherit from `Inspector::Plugin`, which means your class can't be a subclass of anything else. Additionally, you need to use a class even if a module would make more sense. This is a direct consequence of the "interface taking classes rather than ordinary objects" issue and cannot be easily avoided. If you throw in things like having to be aware of possible Liskov Substitution Principle violations, it becomes clear that an API that forces you to use subclassing is not exactly flexible.

This code shows a much more flexible alternative that completely removes the dependency on class inheritance:

```ruby
module Inspector
  def self.analyze(data)
    registered_plugins.each { |e| e.analyze(data) }
  end

  def self.registered_plugins
    @registered_plugins ||= []
  end
end

module WordCountPlugin
  def self.analyze(data)
    word_count = data.split(/ /).length
    puts "Content contained #{word_count} words"
  end
end

module WordLengthPlugin
  def self.analyze(data)
    longest = data.split(/ /).map { |e| e.length }.max
    puts "Longest word contained #{longest} characters"
  end
end

Inspector.registered_plugins << WordCountPlugin << WordLengthPlugin
Inspector.analyze("This is a test of the watcher plugins")
```

Now any object that has a valid `analyze` method will work as a plugin, as long as you explicitly register it with `Inspector`. This approach results in much cleaner-looking code for this trivial implementation, but the general strategy can also can be used in situations where inheritance is still a part of the picture. The following code shows plugins that inherit from a parent class coexisting with plugins built from scratch.

```ruby
module Inspector
  def self.analyze(data)
    registered_plugins.each { |e| e.analyze(data) }
  end

  def self.registered_plugins
    @registered_plugins ||= []
  end

  class Plugin
    def self.verify(&block)
      validations << block
    end

    def self.validations
      @validations ||= []
    end

    def self.analyze(&block)
      define_method :analyze do |data|
        validate(data) 
        block.call(data)
      end
    end

    def validate(data)
      raise unless self.class.validations.all? { |v| v.call(data) }
    end
  end
end

class WordCountPlugin < Inspector::Plugin
  verify { |data| data.is_a?(String) }

  analyze do |data|
    word_count = data.split(/ /).length
    puts "Content contained #{word_count} words"
  end
end

module WordLengthPlugin
  # same as before, not inheriting from anything
end

Inspector.registered_plugins << WordCountPlugin.new << WordLengthPlugin
Inspector.analyze("This is a test of the watcher plugins")
```

This small change makes a big difference in flexibility and leaves some of the decision making up to your users rather than forcing them down a particular path. Using the default approach of inheriting from `Inspector::Plugin` will feel nearly identical to an inheritance-only approach to the user. However, if more customizations are needed, this design provides an easy way to build plugins from scratch. Because it is so easy to implement this pattern, it is probably worth doing even if your immediate needs don't call for such flexibility.

## Inject dependencies

I covered dependency injection and the dependency inversion principle at length in [Practicing Ruby 1.23](http://blog.rubybestpractices.com/posts/gregory/055-issue-23-solid-design.html). Rather than repeating those details here, I'd like you to read that article as well as the comments from [Derick Bailey](http://blog.rubybestpractices.com/posts/gregory/055-issue-23-solid-design.html#comment-317367342). My conversation with Derick taught me a thing or two about the subtle distinctions between dependency injection (a technique) and dependency inversion (a design principle) that are hard to summarize but still well worth working through if you want to solidify your understanding of these two very important concepts.

As an example of some practical code that uses dependency injection, let's look at the Highline command-line library. In ordinary usage, Highline outputs everything to `STDIN` and `STDOUT`. You can install HighLine via RubyGems and run the following example to get a feel for how it works.

```ruby
require "highline"

console = HighLine.new
name    = console.ask("What is your name?") 
console.say("Hello #{name}")
```

In ordinary cases, this is exactly what we want: ordinary input and output from your console. However, to test HighLine, we made it possible to inject input and output objects to use in place of `STDIN` and `STDOUT`. The next example shows the use of `StringIO` objects, which is what we use in our unit tests. 

```ruby
require "stringio"
require "highline"

input   = StringIO.new("Gregory\n")
input.rewind # set seek pos back to 0

output  = StringIO.new

console = HighLine.new(input, output)
name = console.ask("What is your name?")
console.say("Hello #{name}!")

output.rewind
puts output.read
```

Interestingly enough, this 'feature' of HighLine has caused it to be used in a number of contexts that we didn't anticipate. For example, it is occasionally used in GUI programs for its input validation features and is sometimes used in noninteractive scripts for its text formatting features. If we had directly worked against `STDIN` and `STDOUT`, these ways of using HighLine would not be possible without ugly hacks.

## Unit tests are fast

Personally, I am not obsessive about unit test performance. Many Rubyists care a lot about this and advocate the heavy use of mock objects to speed up your tests. Mike points out in his article that the combination of "mock objects, a lack of inheritance, and injected dependencies" will make your tests fast, and that's basically true. 

Dependency injection does facilitate the use of mock objects, and our HighLine example demonstrates why that is the case. A lack of inheritance might imply that your method call chains are shorter and that you have fewer dependencies, but it's not as strong of a metric as he makes it seem. Eventually, object composition can end up being more or less the same as inheritance in complexity, and in those cases you may still end up with slow tests. Composed object systems are easier to decouple, which is probably what he's getting at here.

For some practical examples of how you can use mocks to decouple your tests from your code and speed things up a bit, see [Practicing Ruby 1.20](http://blog.rubybestpractices.com/posts/gregory/052-issue-20-thoughts-on-mocking.html). If you read through that article, you'll find out why I don't care so much about limiting my dependencies to the object under test. But when all else is considered equal, fewer dependencies mean less code, which probably means faster tests.

Having a performant test suite is certainly ideal; it's just a matter of weighing he costs of fine-tuning your test suite versus the benefits that faster test runs provide. Personally, I felt that Mike somewhat overemphasized this particular point, but many well-known Rubyists would disagree with me on this one.

## Require no interface

Mike suggests that your library code should allow your users to name their methods however they want and should be designed to consume rather than be consumed. He then recognizes that this design may not always be possible and concedes that if you need the user to implement certain features, you should limit this functionality to one or two methods at most. This is great general advice, and if you look at Ruby itself, we can find some good examples.

The `Enumerable` module is capable of providing the vast majority of its features if the user implements only an `each` method. If you want to use things like `sort`, you just need the yielded elements to implement a `<=>` method. All features in `Enumerable` can thus be supported by "one or two methods" implemented by the user, which makes it a very good API, considering the great wealth of functionality it provides.

However, `Enumerator` takes things a step further by requiring no interface at all. You can name the methods of the target collection anything you want; you merely tell `Enumerator` which method it should delegate its `each` method to when you initialize an `Enumerator`. See the following example to see how flexible this approach is.

```ruby
class RandomInfiniteList
  def generate
    loop do
      yield rand
    end
  end
end

enum = Enumerator.new(RandomInfiniteList.new, :generate)
p enum.next
p enum.take(10)
```

In this way, `Enumerator` can be made to turn any object with an iterator method into an `Enumerable` object, regardless of what the name of that method is. This feature can be useful when you're working with unidiomatic Ruby objects that provide an iterator but do not mix in `Enumerable`.

We can also look at an example that's a bit less abstract. If we look back at the `Inspector` example that we were using to discuss how to avoid hard inheritance requirements, we can see that it requires only a small interface for plugins to conform to. Although it isn't so bad that each plugin needed an `analyze` method in our previous iteration, we can make some modifications so that it depends on no interface at all, which may bring us a step closer to what Mike was hinting at.

The following example shows what an `Inspector` class that implements an "expects no interface" API style might look like. To keep things interesting, I've left the implementation of this class up to you. If you get stuck, feel free to leave a comment asking for hints on how to build this out.

```ruby
# delegates to the WordLengthPlugin module
Inspector.report("Word Length") do |data|
  WordLengthPlugin.analyze(data)
end

# implements the report as an inline function
Inspector.report("Word Count") do |data|
  word_count = data.split(/ /).length
  puts "Content contained #{word_count} words"
end

Inspector.analyze("This is a test of the watcher plugins")
```

## Data, not action

This guideline seems to boil down to the idea that your API calls should be simple "data in, data out" operations whenever that level of simplicity is easily within reach. After thinking about this concept a bit, I realized that a pair of common Ruby operations serve as perfect examples.

Suppose that you want to open a binary file and read its contents into a `String`. You could write the following code, which will get the job done:

```ruby
file = File.open("foo.jpg", "rb")
image_data = file.read
file.close
```

But this approach is the correct way to do things only if you want to work explicitly with the `File` object and control exactly when it gets opened and closed. If all you want to do is read the contents of a binary file, this is three actions for what should be one "data in, data out" operation. Fortunately, Ruby recognizes this as a common use case and provides a nicer API that you can use instead:

```ruby
image_data = File.binread("foo.jpg")
```

In this example, we pass the filename and get back the contents of that file data. Though we have less control over the overall process, we also get to ignore irrelevant details like exactly when the file is opened and closed. This is what I think that Mike probably meant when he said "data, not action," and it can be applied when designing your own APIs. To drive the point home, let's look at another example.

The following code shows the generic form of doing a GET request via the `Net::HTTP` standard library. It is not the most terse way to use `Net::HTTP`, but it is one of the most common.

```ruby
require 'net/http'

url = URI.parse('http://www.google.com')
res = Net::HTTP.start(url.host, url.port) do |http|
  http.get('/')
end

puts res.body
```

There is a whole lot going on here, including explicit `URI` parsing and explicit calls to `get()`. But as you saw with `File.open` versus `File.binread`, Ruby provides a convenient alternative for this very common operation. The open-uri standard library makes it possible to write the following code instead:

```ruby
require "open-uri"

puts open("http://google.com").read
```

Once again, we see a series of complex actions being converted into a simple "data in, data out" operation. In this case, we are converting a string that represents a URI into an IO object via the globally available `open()` method. This approach makes it possible for us to not think about explicitly parsing our string into a `URI` object and lets us ignore the details about starting a HTTP connection and explicitly making a GET request. When all we want is the source of a web page, it's great to be able to ignore these details.

## Always be coding

As I reflect on Mike's guidelines for Unobtrusive Ruby, it all seems to come down to making life easier for your users by limiting the impact your system has on them. Give your users small, flexible APIs that do not demand much of their systems, and they will have better luck using your code. This seems like a noble set of goals to me, and I hope my examples demonstrate the same spirit that Mike wanted to encourage in his post.

However, I always worry about using design guidelines like these as some sort of absolute set of commandments. There were a whole lot of words like "always" and "never" in the original Unobtrusive Ruby post that, if left unchecked, could cause more harm than good. For me, context is king, and these ideas seem much more important for those who are writing code that is intended for widespread reuse than they might be for ordinary application development. That said, the examples I've shown here demonstrate that you can often be on the right side of these guidelines with little to no additional effort. Therefore, if we can keep the ideas behind Unobtrusive Ruby in the back of our minds and apply them when it's an easy fit, we may well end up improving our code.

This article is meant to be a conversation starter, not gospel. Please share your own thoughts on what Unobtrusive Ruby means to you, either via a comment here or on your own blog. I think it's a conversation worth having, even if we haven't quite nailed down all the definitions yet. If we end up getting an interesting discussion going, I'll invite Mike to check out what we've discussed and see what he thinks about it.

So what do you think? Was my code unobtrusive enough for you? If not, why not? ;)
