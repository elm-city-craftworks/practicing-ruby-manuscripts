Many people are attracted to Ruby because of its flexible syntax. Through various tricks and techniques, it is possible for Ruby APIs to blend seamlessly into a wide range of domains. 

In this article, we will investigate how domain-specific APIs are constructed by implementing simplified versions of popular patterns seen in the wild. Hopefully, this exploration will give you a better sense of the tools you have at your disposal as well as a chance to see various Ruby metaprogramming concepts being used in context.

### Implementing `attr_accessor`

One of the first things every beginner to Ruby learns is how to use `attr_accessor`. It has the appearance of a macro or keyword, but is actually just an ordinary method defined at the module level. To illustrate that this is the case, we can easily implement it ourselves.

```ruby
class Module
  def attribute(*attribs)
    attribs.each do |a|
      define_method(a) { instance_variable_get("@#{a}") }
      define_method("#{a}=") { |val| instance_variable_set("@#{a}", val) }
    end
  end
end

class Person
  attribute :name, :email
end

person = Person.new
person.name = "Gregory Brown"

p person.name #=> "Gregory Brown"
```

In order to understand what is going on here, you need to think a little bit about what a class object in Ruby actually is. A class object is an instance of the class called `Class`, which is in turn a subclass of the `Module` class. When an instance method is added to the `Module` class definition, that method becomes available as a class method on all classes.

At the class/module level, it is possible to call `define_method`, which will in turn add new instance methods to the class/module that it gets called on. So when the `attribute()` method gets called on `Person`, a pair of methods get defined on `Person` for each attribute. The end result is that functionally equivalent code to what is shown below gets dynamically generated:

```ruby
class Person
  def name
    @name
  end

  def name=(val)
    @name = val
  end

  def email
    @email
  end

  def email=(val)
    @email = val
  end
end
```

This is powerful stuff. As soon as you recognize that things like `attr_accessor` are not some special keywords or macros that only the Ruby interpreter can define, a ton of possibilities open up.

### Implementing a Rails-style `before_filter` construct

Rails uses class methods all over the place to make it look and feel like its own dialect of Ruby. As a single example, it is possible to register callbacks to be run before a given controller action is executed using the `before_filter` feature. The simplified example below is a rough approximation of what this functionality looks like in Rails.

```ruby
class ApplicationController < BasicController
  before_filter :authenticate

  def authenticate
    puts "authenticating current user"
  end
end

class PeopleController < ApplicationController 
  before_filter :locate_person, :only => [:show, :edit, :update]

  def show
    puts "showing a person's data"
  end

  def edit
    puts "displaying a person edit form"
  end

  def update
    puts "committing updated person data to the database"
  end

  def create
    puts "creating a new person"
  end

  def locate_person
    puts "locating a person"
  end
end
```

Suppose that `BasicController` provides us with the `before_filter` method as well as an `execute` method which will execute a given action, but first trigger any `before_filter` callbacks. Then we'd expect the `execute` method to have the following behavior. 

```ruby
controller = PeopleController.new

puts "EXECUTING SHOW"
controller.execute(:show)

puts

puts "EXECUTING CREATE"
controller.execute(:create)

=begin -- expected output --

EXECUTING SHOW
authenticating current user
locating a person
showing a person's data

EXECUTING CREATE
authenticating current user
creating a new person 

=end 
```

Implementing this sort of behavior isn't as trivial as implementing a clone of `attr_accessor`, because in this scenario we need to manipulate some class level data. Things are further complicated by the fact that we want filters to propagate down through the inheritance chain, allowing a given class to apply both its own filters as well as the filters of its ancestors. Fortunately, Ruby provides facilities to deal with both of these concerns, resulting in the following implementation of our `BasicController` class:

```ruby
class BasicController
  def self.before_filters
    @before_filters ||= []
  end

  def self.before_filter(callback, params={})
    before_filters << params.merge(:callback => callback)
  end

  def self.inherited(child_class)
    before_filters.reverse_each { |f| child_class.before_filters.unshift(f) }
  end

  def execute(action)
    matched_filters = self.class.before_filters.select do |f| 
      f[:only].nil? || f[:only].include?(action) 
    end

    matched_filters.each { |f| send f[:callback] }
    send(action)
  end
end
```

In this code, we store our filters as an array of hashes on each class, and use the `before_filters` method as a way of lazily initializing that array. Whenever a subclass gets created, the parent class copies its filters to the front of list of filters that the child class will continue to build up. This allows downward propagation of filters through the inheritance chain. If that sounds confusing, exploring in irb a bit might help clear up what ends up happening as a result of this `inherited` hook.

```
>> BasicController.before_filters.map { |e| e[:callback] }
=> []
>> ApplicationController.before_filters.map { |e| e[:callback] }
=> [:authenticate]
>> PeopleController.before_filters.map { |e| e[:callback] }
=> [:authenticate, :locate_person]
```

From here, it should be pretty easy to see how the `execute` method works. It simply looks up this list of filters and selects the ones relevant to the provided action. It then uses `send` to call each callback that was matched, and finally calls the target action. 

While we've only gone through two examples of class level macros so far, the techniques used between the two of them cover most of what you'll need to know to understand virtually all uses of this pattern in other scenarios. If we really wanted to dig in deeper, we could go over some other tricks such as using class methods to mix modules into classes on-demand (a common pattern in Rails plugins), but instead I'll leave those concepts for you to explore on your own and move on to some other interesting patterns.

### Implementing a cheap counterfeit of Mail's API

Historically, sending email in Ruby has always been an ugly and cumbersome process. However, the Mail gem changed all of that not too long ago. Using Mail, sending a message can be as simple as the code shown below.

```ruby
Mail.deliver do
  from    "gregory.t.brown@gmail.com"
  to      "test@test.com"
  subject "Hello world"
  body    "Hi there! This isn't spam, I swear"
end
```

The nice thing about Mail is that in addition to this convenient syntax, it is still possible to work with a more ordinary looking API as well.

```ruby
mail         = Mail::Message.new

mail.from    = "gregory.t.brown@gmail.com"
mail.to      = "test@test.com"
mail.body    = "Hi there! This isn't spam, I swear"
mail.subject = "Hello world"

mail.deliver
```

If we ignore the actual sending of email and focus on the interface to the object, implementing a dual purpose API like this is surprisingly easy. The code below defines a class that provides a matching API to the examples shown above.

```ruby
class FakeMail
  def self.deliver(&block)
    mail = Message.new(&block)
    mail.deliver
  end

  class Message
    def initialize(&block)
      instance_eval(&block) if block
    end

    attr_writer :from, :to, :subject, :body

    def from(text=nil)
      return @from unless text 

      self.from = text
    end

    def to(text=nil)
      return @to unless text

      self.to = text
    end

    def subject(text=nil)
      return @subject unless text

      self.subject = text
    end

    def body(text=nil)
      return @body unless text

      self.body = text
    end

    # this is just a placeholder for a real delivery method
    def deliver
      puts "Delivering a message from #{from} to #{to} "+
      "with the subject '#{subject}' and "+
      "the body '#{body}'"
    end
  end
end
```

There are only two things that make this class definition different from that of the ordinary class definitions we see in elementary Ruby textbooks. The first is that the constructor for `FakeMail::Message` accepts an optional block to run through `instance_eval`, and the second is that the class provides accessor methods which can act as both a reader and a writer depending on whether an argument is given or not. These two special features go hand in hand, as the following example demonstrates:

```ruby
  FakeMail.deliver do 
    # this looks ugly, but would be necessary if using ordinary attr_accessors
    self.from = "gregory.t.brown@gmail.com"

  end

  mail = FakeMail::Message.new

  # when you strip away the syntactic sugar, this looks ugly too.
  mail.from "gregory.t.brown@gmail.com"
```

This approach to implementing this pattern is fairly common and shows up in a lot of different Ruby projects, including my own libraries. By accepting a bit more complexity in our accessor code, we end up with a more palatable API in both scenarios, and it feels like a good trade. However, the dual purpose accessors always felt like a bit of a hack to me, and I recently found a different approach that is I think is a bit more solid. The code below shows how I would attack this problem in new projects:

```ruby
class FakeMail

  def self.deliver(&block)
    mail = MessageBuilder.new(&block).message
    mail.deliver
  end

  class MessageBuilder
    def initialize(&block)
      @message = Message.new
      instance_eval(&block) if block
    end

    attr_reader :message

    def from(text)
      message.from = text
    end

    def to(text)
      message.to = text
    end

    def subject(text)
      message.subject = text
    end

    def body(text)
      message.body = text
    end
  end

  class Message
    attr_accessor :from, :to, :subject, :body

    def deliver
      puts "Delivering a message from #{from} to #{to} "+
      "with the subject '#{subject}' and "+
      "the body '#{body}'"
    end
  end
end
```

This code is a drop-in replacement for what I wrote before, but is quite different under the hood. Rather than putting the syntactic sugar directly onto the `Message` object, I create a `MessageBuilder` object for this purpose. When `FakeMail.deliver` is called, the `MessageBuilder` object ends up being the target context for the block to be evaluated in rather than the `Message` object. This effectively splits the code the implements the sugary interface from the code that implements the domain model, eliminating the need for dual purpose accessors.

There is another benefit that comes along with taking this approach, but it is more subtle. Whenever we use `instance_eval`, it evaluates the block as if you were executing your statements within the object it was called on. This means it is possible to bypass private methods and otherwise mess around with objects in ways that are typically reserved for internal use. By switching the context to a simple facade object whose only purpose is to provide some domain specific API calls for the user, it's less likely that someone will accidentally call internal methods or otherwise stomp on the internals of the system's domain objects.

It's worth mentioning that even this improved approach to implementing an `instance_eval` based interface comes with its own limitations. For example, whenever you use `instance_eval`, it makes it so that `self` within the block points to the object the block is being evaluated against rather than the object in the the surrounding scope. The closure property of Ruby code blocks makes it possible to access local variables, but if you reference instance variables, they will refer to the object your block is being evaluated against. This can confuse beginners and even some more experienced Ruby developers. 

If you want to use this style of API, your best bet is to reserve it for things that are relatively simple and configuration-like in nature. As things get more complex the limitations of this approach become more and more painful to deal with. That having been said, valid use cases for this pattern occur often enough that you should be comfortable implementing it whenever it makes sense to do so.

The next pattern is one that you probably WON'T use all that often, but is perhaps the best example of how far you can stretch Ruby's syntax and behaviors to fit your own domain.

### Implementing a shoddy version of XML Builder

One of the first libraries that impressed me as a Ruby newbie was Jim Weirich's XML Builder. The fact that you could create a single Ruby object that magically knew how to convert arbitrary method calls into an XML structure seemed like pure voodoo to me at the time. 

```ruby
require "builder"

builder = Builder::XmlMarkup.new

xml = builder.class do |roster|
  roster.student { |s| s.name("Jim");    s.phone("555-1234") }
  roster.student { |s| s.name("Jordan"); s.phone("123-1234") }
  roster.student { |s| s.name("Greg");   s.phone("567-1234") }
end

puts xml

=begin -- expected output --

<class><student><name>Jim</name><phone>555-1234</phone></student>
<student><name>Jordan</name><phone>123-1234</phone></student>
<student><name>Greg</name><phone>567-1234</phone></student></class>  

=end
```

Some time much later in my career, I was impressed again by how easy it is to implement this sort of behavior if you cut a few corners. While it's mostly smoke and mirrors, the snippet below is sufficient for replicating the behavior of the previous example.

```ruby
module FakeBuilder
  class XmlMarkup < BasicObject
    def initialize
      @output = ""
    end
    
    def method_missing(id, *args, &block)
      @output << "<#{id}>"
      
      block ? block.call(self) : @output << args.first

      @output << "</#{id}>"

      return @output
    end
  end
end
```

Despite how compact this code is, it gives us a lot to talk about. The heart of the implemenation relies on the use of a `method_missing` hook to convert unknown method calls into XML tags. There are few special things to note about this code, even if you are already familiar with `method_missing`.

Typically it is expected that if you implement a `method_missing` hook, you should be as conservative as possible about what you handle in your hook and then use `super` to delegate everything else upstream. For example, if you were to write dynamic finders similar to the ones that ActiveRecord provides (i.e. something like `find_by_some_attribute`), you would make it so that your `method_missing` hook only handled method calls which matched the pattern `/^find_by_(.*)/`. However, in the case of Builder all method calls captured by `method_missing` are potentially valid XML tags, and so `super` is not needed in its `method_missing` hook.

On a somewhat similar note, certain methods that are provided by `Object` are actually valid XML tag names that wouldn't be too rare to come across. In my example, I intentionally used XML data representing a class of students to illustrate this point, because it forces us to call `builder.class`. By inheriting from `BasicObject` instead of `Object`, we end up with far fewer reserved words on our object, which decreases the likelihood that we will accidentally call a method that does exist. While we don't think about it often, all `method_missing` based APIs hinge on the idea that your hook will only be triggered by calls to undefined methods. In many cases we don't need to think about this, but in the case of Builder (and perhaps when building proxy objects), we need to work with a blank slate object.

The final thing worth pointing out about this code is that it uses blocks in a slightly surprising way. Because the `method_missing` call yields the builder object itself whenever the block is given, it does not serve a functional purpose. To illustrate this point, it's worth noting that the code below is functionally equivalent to our original example:

```ruby
xml = builder.class do 
  builder.student { builder.name("Jim");    builder.phone("555-1234") }
  builder.student { builder.name("Jordan"); builder.phone("123-1234") }
  builder.student { builder.name("Greg");   builder.phone("567-1234") }
end

puts xml
```

However, Builder cleverly exploits block local variable assignment to allow contextual abbreviations so that the syntax more closely mirrors the underlying structure. These days we occasionally see `Object#tap` being used for similar purposes, but at the time that Builder did this it was quite novel.

While it's tempting to write Builder off as just a weird little bit of Ruby magic, it has some surprisingly practical benefits to its design. Unlike my crappy prototype, the real Builder library will automatically escape your strings so that they're XML safe. Also, because Builder essentially uses Ruby to build up an abstract syntax tree (AST) internally, it could possibly be used to render to multiple different output formats. While I've not actually tried it out myself, it looks like someone has already made a [JSON builder](https://github.com/nov/jsonbuilder) which matches the same API but emits JSON hashes instead of XML tags.

With those benefits in mind, this is a good pattern to use for problems that involve outputting documents that nicely map to Ruby syntax as an intermediate format. But as I mentioned before, those circumstances are rare in most day to day programming work, and so you shouldn't be too eager to use this technique as often as possible. That having been said, you could have some fantastic fun adding this sort of freewheeling code to various classes that don't actually need it in your production applications and then telling your coworkers I told you to do it. I'll leave it up to you to decide whether that's a good idea or not :)

With four tough examples down and only two more to go, we're on the home stretch. Take a quick break if you're feeling tired, and then let's move on to the next pattern. 

### Implementing Contest on top of MiniTest

When I used to write code for Ruby 1.8, I liked using the Test::Unit standard library for testing, but I wanted context support and full text test cases similar to what was found in RSpec. I eventually settled on using the [contest](https://github.com/citrusbyte/contest) library, because it gave me exactly what I needed in a very simple an elegant way.

When I moved to Ruby 1.9 and MiniTest, I didn't immediately invest the time in learning `MiniTest::Spec`, which provides similar functionality to contest as well as few other RSpec-style goodies. Instead, I looked into porting contest to MiniTest. After finding a [gist](https://gist.github.com/25455) from Chris Wanswrath and customizing it heavily, I ended up with a simple little test helper that made it possible for me to write tests in minitest which looked like this.

```ruby
context "A Portal" do
  setup do
    @portal = Portal.new
  end

  test "must not be open by default" do
    refute @portal.open?
  end

  test "must not be open when just the orange endpoint is set" do
    @portal.orange = [3,3,3]
    refute @portal.open?
  end

  test "must not be open when just the blue endpoint is set" do
    @portal.blue = [5,5,5]
    refute @portal.open?
  end

  test "must be open when both endpoints are set" do
    @portal.orange = [3,3,3]
    @portal.blue = [5,5,5]

    assert @portal.open?
  end

  # a pending test
  test "must require endpoints to be a 3 element array of numbers"
end
```

Without having to install any third party libraries, I was able to support this kind of test syntax via a single function in my test helpers file.

```ruby
def context(*args, &block)
  return super unless (name = args.first) && block

  context_class = Class.new(MiniTest::Unit::TestCase) do
    class << self
      def test(name, &block)
        block ||= lambda { skip(name) }

        define_method("test: #{name} ", &block)
      end

      def setup(&block)
        define_method(:setup, &block)
      end

      def teardown(&block)
        define_method(:teardown, &block)
      end

       def to_s
         name
       end
    end
  end

   context_class.singleton_class.instance_eval do
     define_method(:name) { name }
   end

  context_class.class_eval(&block)
end
```

If you look past some of the dynamic class generation noise, you'll see that a good chunk of this is quite similar to how I implemented a clone of `attr_accessor` earlier. The `test`, `setup`, and `teardown` methods are nothing more than class methods which delegate to `define_method`. The only slightly interesting detail worth noting here is that in the `test` method I define methods which are not callable using ordinary Ruby method calling syntax. The use of `define_method` allows us to bypass the ordinary syntactic limits of using `def`, and because these methods are only ever invoked dynamically, this works without any issues. The reason I don't bother to normalize the strings is because you end up getting more humanized output from the test runner this way.

If you turn your focus back onto the dynamic class generation, you can see that this code creates an anonymous subclass of `MiniTest::Unit::TestCase` and then eventually uses `class_eval` to evaluate the provided block in the context of this class. This code is what enables us to write `context "foo" do ... end` and get something that works similar to the way an ordinary class definition works.

If you're focusing on really subtle details, you'll notice that this code goes through a bunch of hoops to define meaningful `name` and `to_s` methods on the class it dynamically generates. This is in part a bunch of massaging to get better output from MiniTest's runner, and in part to make it so that our anonymous classes don't look completely anonymous during debugging. The irb session below might make some sense of what's going on here, but if it doesn't you can feel free to chalk this up as an edge case you probably don't need to worry about.

```
>> Class.new
=> #<Class:0x00000101069260>
>> name = "A sample class"
=> "A sample class"
>> Class.new { singleton_class.instance_eval { define_method(:to_s) { name } } }
=> A sample class
```

Getting away from these ugly little details and returning to the overall pattern, it is relatively common to see domain-specific APIs which dynamically create modules or classes and then wrap certain kinds of method definitions in block based APIs as well. It's a handy pattern when used correctly, and could be useful in your own projects. But even if you never end up using it yourself, it's good to know how this all works as it'll make code reading easier for you.

While this example was perfect for having a discussion about the pattern of dynamic class creation in general, I'd strongly recommend against using my helper in your MiniTest code at this point. You'll find everything you need in `MiniTest::Spec`, and that is a much more standard solution than using some crazy hack I cooked up simply because I could.

With that disclaimer out of the way, we can move on to our final topic.

### Implement your own Gherkin parser, or criticize mine!

So far, we've talked about various tools which enable the use of domain specific language (DSL) within your Ruby applications. However, there is a whole other category of DSL techniques which involve parsing external languages and then converting them into meaningful structures within the host language. This is a topic that deserves an entire article of its own.

But because it'll be a while before I get around to writing that article, we can wrap up this article with a little teaser of things to come. To do so, I am challenging you to implement a basic story runner that parses the Gherkin language which is used by [Cucumber](http://cukes.info/) and other similar tools.

Your mission, if you chose to accept it, is to take the following feature file and process it with Cucumber-style step definitions. You can feel free to simplify your prototype as much as you'd like, as long as you capture the core idea of processing the steps in the feature file and executing arbitrary code for each of those steps.

```
Feature: Addition
  Scenario: Add two numbers
    Given I have entered 70 into the calculator
    And I have entered 50 into the calculator
    When I press add
    Then the result should be 120
```

If that sounds like too much work for you, you can take on a slightly easier task instead. In preparation for this article, I build two different implementations that capture the essence of the way that that Cucumber story runner works. [One implementation uses global functions](https://github.com/elm-city-craftworks/dsl_construction/blob/master/cucumber/global-dsl/fake_cuke.rb), and the [other implementation uses eval() with a binding](https://github.com/elm-city-craftworks/dsl_construction/blob/master/cucumber/binding-dsl/fake_cuke.rb). I'd like you to examine these two approachs and share your thoughts on what the functional differences between them are.

While I know not everyone will have the time to try out this exercise, if a few of you work on this and share your results, it will lead to a good discussion which could help me gauge interest in a second article about external DSLs. So if you have a few spare moments, please participate! 

### Reflections

We've now reached the end of a whirlwind tour of several powerful tools Ruby provides for bending its syntax and semantics to meet domain-specific needs. While I tried to pick examples which illustrated natural uses of domain specific API construction patterns, I am left feeling that these are advanced techniques which even experienced developers have a hard time evaluating the tradeoffs of.

There are two metrics to apply before trying out anything you've seen in this article in your own projects. The first thing to remember is that any deviation from ordinary method definitions and ordinary method calls should offer a benefit that is at least proportional to how exotic your approach is. Cleverness for the sake of cleverness can be a real killer if you're not careful. The second thing to remember is that whenever if you provide nice domain-specific APIs for convenience or aesthetic reasons, you should always make sure to build it as a facade over a boring and vanilla API. This will help make sure your objects are easier to test and easier to work with in scenarios that your domain specific interface did not anticipate.

If you follow these two bits of advice, you can have fun using Ruby's sharp knives without getting cut too often. But if you do slip up from time to time, don't be afraid to abandon fancy interfaces in favor of having something a bit dull but easier to maintain and understand. It can be tempting to layer dynamic features on top of one another to "simplify" things, but that only hides the underlying problem which is that perhaps you were trying too hard. This is something that used to happen to me all the time, so don't feel bad when it happens to you. Just do what you can to learn from your mistakes as you try out new designs.

_NOTE: If you want to experiment with the examples in this article a bit more, you can find all of them in [this git repository](https://github.com/elm-city-craftworks/dsl_construction). If you fork the code and submit pull requests with improvements, I will review your changes and eventually make a note of them here if we stumble across some good ideas that I didn't cover._
