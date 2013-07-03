Type systems are a fundamental part of every programming language. In fact, the way a language designer approaches typing goes a long way towards outlining the way that thoughts are expressed in that language.

Statically typed languages like C++ and Java make us tend to think of objects as abstract data structures that fit within a neatly defined hierarchy. In these languages, there isn't a major distinction between an object's class and its type, as the two concepts are tied together directly at the implementation level. But the marriage of class and type found in these languages is not a universal law shared by all object oriented programming languages.

By contrast, Ruby's dynamic nature facilitates a style of type system known as duck typing. In particular, duck typing breaks the strong association between an object's class and its type by defining types based on what an object can do rather than what class it was born from. This subtle shift in semantics changes virtually everything about how you need to think about designing object oriented systems, making it a great topic for Practicing Ruby to cover.

While duck typing is possible in many other languages, Ruby is designed from the ground up to support this style of objected oriented design. In this issue, we will cover some of the options that are available to us for doing Ruby-style type checking. 

### Type Checking Techniques

There are three common ways to do type checking in Ruby, two of which involve
duck typing, and one that does not. Here's an example of the approach 
that does *not* involve duck typing.

```ruby
def read_data(source)
  case source
  when String
    File.read(source)
  when IO
    source.read
  end
end
```

If you've been working with Ruby for a while, you've probably written code that
did type checking in this fashion. Ruby's case statement is powerful, and
makes this sort of logic easy to write. Our `read_data()` function works as
expected in the following common scenarios:

```ruby
filename = "foo.txt"
read_data(filename) #=> reads the contents of foo.txt by calling 
                    #   File.read()


input = File.open("foo.txt")
read_data(input) #=> reads the contents of foo.txt via 
                 #   the passed in file handle
```
  
But things begin to fall apart a bit when we decide we'd like `read_data()` to
work with a `Tempfile`, or with a `StringIO` object, or perhaps with a mock
object we've defined in our tests. We have baked into our logic the assumption that the input is always either a descendent of `String` or a descendent of `IO`. The purpose of duck typing is to remove these restrictions by focusing only on the messages that are being passed back and forth between objects rather than what class they belong to. The code below demonstrates one way you can do that.

```ruby
def read_data(source)
  return source.read if source.respond_to?(:read)
  return File.read(source.to_str) if source.respond_to?(:to_str)
  raise ArgumentError
end
```

With this modification, our method expects far less of its input. The passed in
object simply needs to implement either a meaningful `read()` or `to_str()`
method. In addition to being backwards compatible with our non-duck-typed code,
this new approach gives us access to many useful standin objects, including: `StringIO`, `Tempfile`, mock objects for testing, and any user defined objects that are either IO-like or String-like but not a descendent of either.

However, the following contrived example illustrates a final corner case that calls for a bit of extreme duck typing to resolve. Try to spot the problem before reading about how to solve it.

```ruby
class FileProxy
  def initialize(tempfile)
    @tempfile = tempfile
  end

  def method_missing(id, *args, &block)
    @tempfile.send(id, *args, &block)
  end
end
```

This code implements a proxy which forwards all of its messages to the wrapped `tempfile` object. However, like many hastily coded proxy objects in Ruby, it does not properly forward `respond_to?()` calls to the object it wraps. The irb session below illustrates the resulting false negative in our test.

```ruby
# Populate our tempfile through the proxy

>> proxy = FileProxy.new(Tempfile.new("foo.txt"))
=> #<FileProxy:0x39461c @tempfile=#<File:/var/f..foo.txt.7910.3>>
>> proxy << "foo bar baz"
=> #<File:/var/folders/sJ/sJo0IkPYFWCY3t5uH+gi0++++TQ/-Tmp-/foo.txt.7910.3>
>> proxy.rewind
=> 0

# Unsuccessfully test for presence of read() method

>> proxy.respond_to?(:read)
=> false

# But read() works as expected!

>> proxy.read
=> "foo bar baz"
```

This issue will cause `read_data()` to raise an `ArgumentError` when passed a `FileProxy`. In this case, the best solution is to fix `respond_to?()` so that it works as expected, but since you may often encounter libraries with bad behaviors like this, it's worth knowing what the duck typing fundamentalist would do in this situation.

```ruby
def read_data(source)
  begin 
    return source.read 
  rescue NoMethodError
    # do nothing, just catch the specific error you'd expect if
    # read() was not present.
  end

  begin
    File.read(source.to_str)
  rescue NoMethodError
    raise ArgumentError # now we've run out of valid cases, so let's
                        # raise a meaningful error
   end
end
```

With this final version, we preserve all the benefits of the previous duck
typing example, but we can work with objects that have dishonest `respond_to?()`
methods. Unfortunately, the cost for such flexibility includes code that is less
pleasant to read and is almost certainly going to run slower than either of our
previous implementations. Using the exception system for control flow isn't cheap, 
even if this is the most 'pure' form of type checking we can do.

While we've talked about the benefits and drawbacks of each of these approaches, I haven't given any direct advice on whether one way of doing type checking is better than the others, simply because there is no simple answer to that question.

I will paint a clearer picture in the next article by showing several
realistic examples of why duck typing can come in handy. Until then, I will
leave you with a few things to think about.

### Questions / Study Topics

* Is explicit class checking ever absolutely necessary? Are their situations in which even if other options are available, checking the class of an object is still the best thing to do?

* Name something weird that can happen when you write your contracts on the messages your objects respond to rather than what class of object they are.

* Try to identify some feature of Ruby that relies on duck typing either for its basic functionality or as an extension point meant to be customized by application programmers.

* Share a bit of code which does explicit class comparison that you think would be very difficult to convert to a duck-typing style.

* Share a bit of code (either your own or from a OSS project you like) that you feel uses duck typing effectively.

Feel free to leave a comment below if any of the above topics interest you.

  
> **NOTE:** This article has also been published on the Ruby Best Practices blog. There [may be additional commentary](http://blog.rubybestpractices.com/posts/gregory/046-issue-14-duck-typing.html#disqus_thread) 
over there worth taking a look at.
