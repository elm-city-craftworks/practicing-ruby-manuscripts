> Note: This article series on modules is also available as a [PDF download]. The
> PDF version has been revised and is more up-to-date than what you see here.

[PDF download]:https://github.com/elm-city-craftworks/pr-monthly/blob/gh-pages/b5e5a89847701c4aa7c170cf/sept-2012-modules.pdf?raw=true

In the last two issues, we covered mixins and namespacing, two of the most common uses for modules. In the second half of this series, we'll look at some other ways to use modules that are not quite so obvious.

We can now focus on the question that caused me to write this series in the
first place. Many readers were confused by my use of `extend self` within
earlier Practicing Ruby articles, and this lead to a number of interesting
questions on the mailing list at the time these articles were originally
published. While I tried my best to answer them directly, I think we're in better
shape to study this topic now that the last two articles have laid a 
foundation for us.

### Review of how `extend()` works

To understand this trick of mixing modules into themselves, one first must understand how `extend()` works. We covered this concept at the end of the last article, but we can touch on it again for good measure. Start by considering the trivial module shown below.

```ruby
module Greeter
  def hello
    "hi"
  end
end 
```

We had shown that unlike `include()` which is especially designed for augmenting class definitions so that a mixin can add instance methods to some target class, `extend()` has a much more simple behavior and works with any object.

```ruby
obj = Object.new
obj.extend(Greeter)
obj.hello #=> "hi"
```

From this, we can see that mixing in a module by using extend simply mixes the methods defined by the module directly at that object's level. In this way, the methods defined by the module are mixed into the receiver, no matter what that object is.

In Ruby, classes and modules are ordinary objects. We can confirm this by doing a tiny bit of introspection on `Greeter`.

```ruby
>> Greeter.object_id
=> 212500
>> Greeter.class
=> Module
>> Greeter.respond_to?(:extend)
=> true
```

While this may be a mental leap for some, you might be able to find peace with it by considering the ordinary module definition syntax to be a bit of sugar that is functionally equivalent to the following bit of code.

```ruby  
Greeter = Module.new do
  def hello
    "hi"
  end
end
```

When written in this way, it becomes far more obvious that `Greeter` is actually just an instance of the class Module, making it an ordinary Ruby object at its core. Once you feel that you understand this point, consider what happens when the following line of code is run.

```ruby
Greeter.extend(Greeter)
```

If we compare this to previous examples of `extend()`, it should be clear now that despite the seemingly circular reference, this line does exactly what it would if called on any other object: It mixes the methods defined by `Greeter` directly into the `Greeter` object itself. A simple test confirms this to be true.

```ruby
Greeter.hello #=> "hi"
```

If we unravel things a bit, we find that we could have written our `extend()` call slightly differently, by doing it from within the module definition itself:

```ruby
module Greeter
  extend Greeter

  def hello
    "hi"
  end
end
```

The reason `extend()` works here is because `self == Greeter` in this context.
Noticing this detail allows us to use slightly more dynamic approach, resulting
in the following code.

```ruby
module Greeter
  extend self

  def hello
    "hi"
  end
end
```

You'll find this new code to be functionally identical to the previous example, but slightly more flexible. Now, if we change the name of our module, we won't need to update our `extend()` call. This is why folks tend to write `extend self` rather than `extend TheCurrentModule`.

Hopefully by now, it is clear that this trick does not involve any sort of special casing for modules, and is an ordinary application of the `extend()` method provided by every Ruby object. The only thing that might be confusing is the seemingly recursive nature of the technique, but this issue disappears when you recognize that modules are not mixed into anything by default, and that modules themselves are not directly related to the methods they define. If you understand the difference between class and instance methods in Ruby, this isn't a far stretch from that concept.

While the inner workings of modules are an interesting academic topic, my emphasis is always firmly set on practical applications of programming techniques rather than detached conceptual theory. So now that we've answered 'how does this work?', let's focus on the much more interesting 'how can I use it?' topic.

### Self-Mixins as Function Bags

A fascinating thing about Ruby is the wide range of different software design paradigms it supports. While object-oriented design is heavily favored, Ruby can do a surprisingly good job of emulating everything from procedure programming to prototype-based programming. But the one area that Ruby overlaps most with is functional programming.

Now, before you retire your parenthesis for good and herald Ruby as a replacement for LISP, be warned: There is a lot about Ruby's design that makes it a horrible language for functional programming. But when used sparingly, techniques from the functional world fit surprisingly well in Ruby programs. The technique I find most useful is the ability to organize related functions together under a single namespace.

When we create class definitions, we tend to think of the objects we're building as little structures which manage state and provide behaviors which manipulate that state. But sometimes, a more stateless model makes sense. The closer you get to pure mathematics, the more a pure functional model makes sense. We need to look no farther than Ruby's own `Math` module for an example:

```ruby
>> Math.sin(Math::PI/2.0)
=> 1.0
>> Math.log(Math::E)
=> 1.0
```

It seems unlikely that we'd want to create an instance of a `Math` object, since
it doesn't really deal with any state that persists beyond a single function
call. But it might be desirable to mix this functionality into another object so
that you can call math functions without repeating the `Math` constant
excessively. For this reason, Ruby implements `Math` as a module.

```ruby
>> Math.class
=> Module
```

For another great example of modular code design in Ruby itself, be sure to check out the `FileUtils` standard library, which allows you to execute basic *nix file operations as if they were just ordinary function calls.

After seeing how Ruby is using this technique, I didn't find it hard to stumble upon scenarios in my own code that could benefit from a similar design. For example, when I was working on building out the backend for a trivia website, I was given some logic for normalizing user input so that it could be compared against a predetermined pattern.

While I could have stuck this logic in a number of different places, I decided I wanted to put it within a module of its own, because its logic did not rely on any persistent state and could be defined independently of the way our questions and quizzes were modeled. The following code is what I came up with:

```ruby
module MinimalAnswer
  extend self

  def match?(pattern, input)
    pattern.split(/,/).any? do |e| 
      normalize(input) =~ /\b#{normalize(e)}/i 
    end
  end

  private

  def normalize(input)
    input.downcase.strip.gsub(/\s+/," ").gsub(/[?.!\-,:'"]/, '')
  end
end
```

The nice thing about the code above is that using a modular design doesn't force you to give up things like private methods. This allows you to keep your user facing API narrow while still being able to break things out into helper methods.

Here is a simple example of how my `MinimalAnswer` module is used within the application:

```ruby
>> MinimalAnswer.match?("Cop,Police Officer", "COP")
=> true
>> MinimalAnswer.match?("Cop,Police Officer", "police officer")
=> true
>> MinimalAnswer.match?("Cop,Police Officer", "police office")
=> false
>> MinimalAnswer.match?("Cop,Police Officer", "police officer.")
=> true
```

Now as I said before, this is a minor bit of functionality and could probably be shelved onto something like a `Question` object or somewhere else within the system. But the downside of that approach would be that as this `MinimalAnswer` logic began to get more complex, it would begin to stretch the scope of whatever object you attached this logic to. By breaking it out into a module right away, we give this code its own namespace to grow in, and also make it possible to test the logic in isolation, rather than trying to bootstrap a potentially much more complex object in order to test it.

So whenever you have a bit of logic that seems to not have many state dependencies between its functions, you might consider this approach. But since stateless code is rare in Ruby, you may wonder if learning about self-mixins really bought us that much.

As it turns out, the technique can also be used in more stateful scenarios when you recognize that Ruby modules are objects themselves, and like any object, can contain instance data.

### Self-Mixins for Implementing Singleton Pattern

Ruby overloads the term 'singleton object', so we need to be careful about terminology here. What I'm about to show you is how to use these self-mixed modules to implement something similar to the [Singleton design pattern](http://en.wikipedia.org/wiki/Singleton_pattern).

I've found in object design that objects typically need zero, one, or many instances. When an object doesn't really need to be instantiated at all because it has no data in common between its behaviors, the modular approach we just reviewed often works best. The vast majority of the remaining cases fall into ordinary class definitions which facilitate many instances. Virtually everything we model fits into this category, so it's not worth discussing in detail. However, there are some cases in which a single object is really all we need. In particular, configuration systems come to mind.

The following example shows a simple DSL I wrote for the trivia application I had mentioned earlier. It may look familiar, and that is because it appeared in our discussion on writing configuration systems some weeks ago. This time around, our focus will be on how this system actually works rather than what purpose it serves.

```ruby
AccessControl.configure do
  role "basic",
    :permissions => [:read_answers, :answer_questions]

  role "premium",
    :parent      => "basic",
    :permissions => [:hide_advertisements]

  role "manager",
    :parent      => "premium",
    :permissions => [:create_quizzes, :edit_quizzes]

  role "owner",
    :parent      => "manager",
    :permissions => [:edit_users, :deactivate_users]
end 
```

To implement code that allows the definitions above to be modeled internally, we need to consider how this system will be used. While it is easy to imagine roles shifting over time, getting added and removed as needed, it's hard to imagine what the utility of having more than one `AccessControl` object would be.

For this reason, it's safe to say that `AccessControl` configuration data is global information, and so does not need the data segregation that creating instances of a class provides.

By modeling `AccessControl` as a module rather than class, we end up with an object that we can store data on that can't be instantiated.

```ruby
module AccessControl
  extend self

  def configure(&block)
    instance_eval(&block)
  end

  def definitions
    @definitions ||= {}
  end

  # Role definition omitted, replace with a stub if you want to test
  # or refer to Practicing Ruby Issue #4
  def role(level, options={})
    definitions[level] = Role.new(level, options)
  end

  def roles_with_permission(permission)
    definitions.select { |k,v| v.allows?(permission) }.map { |k,_| k }
  end

  def [](level)
    definitions[level]
  end 
end
```

There are two minor points of potential confusion in this code worth discussing, the first is the use of `instance_eval` in `configure()`, and the second is that the `definitions()` method refers to instance variables. This is where we need to remind ourselves that the scope of methods defined by a module cannot be determined until it is mixed into something.

Once we recognize these key points, a bit of introspection shows us what is really going on.

```ruby
>> AccessControl.configure { "I am #{self.inspect}" }
=> "I am AccessControl"
>> AccessControl.instance_eval { "I am #{self.inspect}" }
=> "I am AccessControl"
>> AccessControl.instance_variables
=> ["@definitions"]
```

Since `AccessControl` is an ordinary Ruby object, it has ordinary instance variables and can make use of `instance_eval` just like any other object. The key difference here is that `AccessControl` is a module, not a class, and so cannot be used as a factory for creating more instances. In fact, calling `AccessControl.new` raises a `NoMethodError`.

In a traditional implementation of Singleton Pattern, you have a class which disables instantiation through the ordinary means, and creates a single instance that is accessible through the class method `instance()`. However, this seems a bit superfluous in a language in which classes are full blown objects, and so isn't necessary in Ruby.

For cases like the configuration system we've shown here, choosing to use this approach is reasonable. That having been said, the reason why I don't have another example that I can easily show you is that with the exception of this narrow application for configuration objects, I find it relatively rare to have a legitimate need for the Singleton Pattern. I'm sure if I thought long and hard on it, I could dig some other examples up, but upon looking at recent projects I find that variants of the above are all I use this technique for.

However, if you work with other people's code, it is likely that you'll run into someone implementing Singleton Pattern this way. Now, rather than scratching your head, you will have a solid understanding of how this technique works, and why someone might want to use it.

### Reflections

In Issue 11, we'll wrap up with some even more specialized uses for modules, showing how they can be used to build plugin systems as well as how they can be used as a replacement for monkey patching. But before we close the books on today's lesson, I'd like to share some thoughts that were rattling around in the back of my mind while I was preparing this article.

The techniques I've shown today can be useful in certain edge case scenarios
where an ordinary class definition might not be the best tool to use. In my own
code, I tend to use the first technique of creating function bags often but sparingly, 
and the second technique of building singleton objects rarely and typically only 
for configuration systems.

Upon reflection, I wonder to myself whether the upsides of these techniques outweigh the cost of explaining them. I don't really have a definitive answer to that question, but it's really something I think about often.

On the one hand, I feel that users of Ruby should have an ingrained understanding of its object system. After all, these are actually fairly straightforward techniques once you understand how things work under the hood. It's also true that you can't really claim to understand Ruby's object system without fully understanding these examples. Having a weak understanding of how Ruby's objects work is sure to rob you of the joy of working in Ruby, so for this reason, I feel like 'dumbing down' our code would be a bad thing.

On the other hand, I think that for the small gains yielded by using these techniques, we require those who are reading our code to understand a whole score of details that are unique to Ruby. When you consider that by changing a couple lines of code, you can have a design which is not much worse but is understandable by pretty much anyone who has programmed in an OO language before, it's certainly tempting to cater to the lowest common denominator.

But this sort of split-mindedness is inevitable in Ruby, and comes up in many scenarios. The truth of the matter is that it's going to take many more years before Ruby is truly understood by the programming community at large. But as more people dive deeper into Ruby, Ruby is starting to come into its own, and the mindset that things should be done as they are in other languages is not nearly as common as it was several years ago. For this reason, it's important to stop thinking of Ruby in terms of whatever language you've come from, and start thinking of it as its own thing. As soon as you do that, a whole range of possibilities open up.

At least, that's what I think. What about you?

  
> **NOTE:** This article has also been published on the Ruby Best Practices blog. There [may be additional commentary](http://blog.rubybestpractices.com/posts/gregory/040-issue-10-uses-for-modules.html#disqus_thread) 
over there worth taking a look at.

