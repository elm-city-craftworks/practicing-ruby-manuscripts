> Note: This article series on modules is also available as a [PDF download]. The
> PDF version has been revised and is more up-to-date than what you see here.

[PDF download]:https://github.com/elm-city-craftworks/pr-monthly/blob/gh-pages/b5e5a89847701c4aa7c170cf/sept-2012-modules.pdf?raw=true

In the [last issue](http://practicingruby.com/articles/38), we discussed the use of `extend self` in great detail, but neglected to cover a pair of alternatives that seem on the surface to be functionally equivalent. While I don't want to spend too much time rehashing an old topic, I want to at least provide an example of each approach and comment on their quirks.

### Defining methods at the module level

Occasionally folks ask whether mixing a module into itself via `extend()` is equivalent to the code shown below.

```ruby
module Greeter
  def self.hello
    "hi"
  end
end
```

The short answer to that question is "no", but it is easy to see where the confusion comes from, because calling `Greeter.hello` does indeed work as expected. But the important distinction is that methods defined in this way are simply directly defined on the module itself and so cannot be mixed into anything at all. There is really very little difference between the above code and the example below.

```ruby  
obj = Object.new

def obj.hello
  "hi"
end
```

Consider our earlier example of Ruby's `Math` or `FileUtils` modules. With both of these modules, you can envision scenarios in which you would call the functions on the modules themselves. But there are also cases where using these modules as mixins would make a lot of sense. For example, Ruby itself ships with a math mode (-m) for irb which mixes in the `Math` module at the top level so you can call its functions directly.

```ruby
$ irb -m
>> sin(Math::PI/2)
=> 1.0
```

In the above example, if `sin()` were implemented by defining the method
directly on the `Math` module, there would be no way to mix it into anything.
While sometimes it might make sense to force a module to never be used as a
mixin, that use case is rare, and so little is gained by defining methods on
modules rather than using the `extend self` technique.

### Using `module_function`

Before people got in the habit of mixing modules into themselves, they often relied on a more specialized feature called `module_function` to accomplish the same goals.

```ruby
module Greeter
  module_function

  def hello
    "hi"
  end
end
```

This code allows the direct calling of `Greeter.hello`, and does not prevent
`Greeter` from being mixed into other objects. The `module_function` approach
also allows you to choose certain methods to be module functions while 
leaving others accessible via mixin only:

```ruby
module Greeter
  def hello
    "hi"
  end

  def goodbye
    "bye"
  end

  module_function :hello
end
```

With this modified definition, it is still possible to call `Greeter.hello`, but attempting to call `Greeter.goodbye` would raise a `NoMethodError`. This sort of sounds like it offers the benefits of extending a module with itself, but with some added granularity. Unfortunately, there is something about `module_function` that makes it quite weird to work with.

As it turns out, `module_function` works very different under the hood than self-mixins do. This is because `module_function` actually doesn't manipulate the method lookup path, but instead, it makes a direct copy of the specified methods and attaches them to the module itself. If that sounds too weird to be true, check out the code below.

```ruby 
module Greeter
  def hello
    "hi"
  end

  module_function :hello

  def hello
    "howdy"
  end
end

Greeter.hello #=> "hi"

class Foo
  include Greeter
end

Foo.new.hello #=> "howdy"
```

Pretty weird behavior, right? You may find it interesting to know that I was not actually aware that `module_function` made copies of methods until I wrote Issue #10 and was tipped off about this by one of our readers. However, I did know about one of the consequences of `module_function` being implemented in this way: private methods cannot be used in conjunction with `module_function`. That means that the following example cannot be literally translated to use `module_function`.

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

From these examples, we see that `module_function` is more flexible than defining methods directly on your modules, but not nearly as versatile as extending a module with itself. While the ability to selectively define which methods can be called directly on the module is nice in theory, I've yet to see a use case for it where it would lead to a much better design.

### Reflections

With the alternatives to `extend self` having unpleasant quirks, it's no surprise that they're quickly falling out of fashion in the Ruby world. But since no technical decision should be made based on dogma or a blind-faith acceptance of community conventions, these notes hopefully provide the necessary evidence to help you make good design decisions on your own.

  
> **NOTE:** This article has also been published on the Ruby Best Practices blog. There [may be additional commentary](http://blog.rubybestpractices.com/posts/gregory/041-issue-10.5-uses-for-modules.html#disqus_thread) 
over there worth taking a look at.
