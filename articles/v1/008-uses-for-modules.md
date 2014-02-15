> Note: This article series on modules is also available as a [PDF download]. The
> PDF version has been revised and is more up-to-date than what you see here.

[PDF download]:https://github.com/elm-city-craftworks/pr-monthly/blob/gh-pages/b5e5a89847701c4aa7c170cf/sept-2012-modules.pdf?raw=true

Modules are part of what makes Ruby's design beautiful. However, since they do not have a direct analogy in any mainstream programming language, it is easy to get a bit confused about what they should be used for. While most folks quickly encounter at least some of their use cases, typically only very experienced Ruby developers know their true versatilty.

In this four part article series, I aim to demystify Ruby modules by showing many practical use cases, explaining some tricky details along the way. We'll work through some of the fundamentals in the first two issues, and move into more advanced examples in the second two. Today we'll kick off this series by looking at the most simple, but perhaps most important ability modules offer us, the creation of namespaces.

### Modules for Namespacing

Imagine that you are writing an XML generation library, and in it, you have a class to generate your XML documents. Perhaps uncreatively, you choose the name `Document` for your class, creating something similar to what is shown below.

```ruby
class Document
  def generate
    # ...
  end
end
```

On its own, this seems to make a lot of sense; a user could do something simple like the following to make use of your library.

```ruby
require "your_xml_lib"
document = Document.new
# do something with document
puts document.generate
```

But imagine that you were using another library that generates PDF documents, which happens to use similar uncreative naming for its class that does the PDF document generation. Then, the following code would look equally valid.

```ruby
require "their_pdf_lib"
document = Document.new
# do something with document
puts document.generate
```

As long as the two libraries were never loaded at the same time, there would be no issue. But as soon as someone loaded both libraries, some quite confusing behavior would happen. One might think that defining two different classes with the same name would lead to some sort of error being raised by Ruby, but with open classes, that is not the case. Ruby would actually apply the definitions of `Document` one after the other, with whatever file was required last taking precedence. The end result would in all likelihood be a very broken `Document` class that could generate neither XML nor PDF.

But there is no reason for this to happen, as long as both libraries take care to namespace things. Shown below is an example of two `Document` classes that could co-exist peacefully.

```ruby
# somewhere in your_xml_lib

module XML
  class Document
    # ...
  end
end

# somewhere in their_pdf_lib

module PDF
  class Document
    # ...
  end
end
```

Using both classes in the same application is as easy, as long as you explicitly include the namespace when referring to each library's `Document` class.

```ruby
require "your_xml_lib"
require "their_pdf_lib"

# this pair of calls refer to two completely different classes
pdf_document = PDF::Document.new
xml_document = XML::Document.new
```

The clash has been prevented because each library has nested its `Document` class within a module, allowing the class to be defined within that namespace rather than at the global level. While this is a relatively straightforward concept, it's important to note a few things about what is really going on here.

Firstly, namespacing actually applies to the way constants are looked up in Ruby in general, not classes in particular. This means that it applies to modules nested within modules as well as ordinary constants as well.

```ruby
module A
  module B
  end
end

p A::B

module A
  C = 10
end

p A::C
```

Secondly, this same behavior of using modules as namespaces applies just as well to classes, as in the code below.

```ruby
class Blog
  class Comment
    #...
  end
end
```

Be sure to note that in this example, nesting a class within a class does not in any way make it a subclass or establish any relationship between `Blog` and `Blog::Comment` except that `Blog::Comment` is within the `Blog` namespace. In the example below, you can see that a class nested within another class looks the same as a class nested within a module.

```ruby
blog = Blog.new
comment = Blog::Comment.new
# ...
```

Of course, this technique is only really useful when you have a desired namespace for your library that also happens matches one of your class names. In all other situations, it makes sense to use a module for namespacing as it would prevent your users from creating instances of an empty and meaningless class.

Finally, it is important to understand that constants are looked up from the innermost nesting to the outermost, finally searching the global namespace. This can be a bit confusing at times, especially when you consider some corner cases.

For example, examine the following code:

```ruby
module FancyReporter
  class Document
    def initialize
       @output = String.new
    end

    attr_reader :output
  end
end
```

If you load this code into irb and play with a bit on its own, you can inspect an instance of Document to see that its output attribute is a core ruby `String` object, as shown below:

```ruby
>> FancyReporter::Document.new.output
=> ""
>> FancyReporter::Document.new.output.class
=> String
```

While this seems fairly obvious, it is easy for a bit of unrelated code written elsewhere to change everything. Consider the following code:

```ruby
module FancyReporter
  module String
    class Formatter
    end
  end
end
```

While the designer of `FancyReporter` was most likely trying to be well organized by offering `FancyReporter::String::Formatter`, this small change causes headaches because it changes the meaning of `String.new` in `Document`'s initialize method. In fact, you cannot even create an instance of `Document` before the following error is raised:

```ruby
?> FancyReporter::Document.new
NoMethodError: undefined method `new' for FancyReporter::String:Module
	from (irb):35:in `initialize'
	from (irb):53:in `new'
	from (irb):53
```

There are a number of ways this problem can be avoided. Often times, it's
possible to come up with alternative names that do not clash with core objects,
and when that's the case, it's preferable. In this particular case, `String.new`
can also be replaced with `""`, as nothing can change what objects are created
via Ruby's string literal syntax. But there is also an approach that works
independent of context, and that is to use explicit constant lookups from the
global namespace. You can see an example of explicit lookups in the following
code:

```ruby
module FancyReporter
  class Document
    def initialize
       @output = ::String.new
    end

    attr_reader :output
  end
end
```

Prepending any constant with `::` will force Ruby to skip the nested namespaces and bubble all the way up to the root. In this sense, the difference between `A::B` and `::A::B` is that the former is a sort of relative lookup whereas the latter is absolute from the root namespace.

In general, having to use absolute lookups may be a sign that there is an unnecessary name conflict within your application. But if upon investigation you find names that inheritently collide with one another, you can use this tool to avoid any ambiguity in your code.

While we've mostly covered the mechanics of namespacing, all this talk about `::` compels me to share a cautionary tale of mass cargoculting before we wrap up for today. Please bear with me as I stroke my beard for a moment.

### Abusing the Constant Lookup Operator (`::`)

In some older documentation, and some relatively recent code written by folks who learned from old documentation, you may see class methods being called in the manner shown below.

```ruby
YAML::load(File::read("foo.yaml"))
```

While the above code runs fine, it's only a historical accident that it does. In fact, `::` was never meant for method invocation, class methods or otherwise. You can easily demonstrate that `::` can be used to execute instance methods as well, which eliminates any notion that `::` has some special 'class methods only' distinction to it.

```ruby  
"foo"::reverse #=> "oof"
```

As far as I can tell, this style of method invocation actually came about as a documentation convention. In both formal documentation and in mailing list discussions, it can sometimes be difficult to discern whether someone is talking about a class method or instance method, since both can be called just as well with the dot operator. So, a convention was invented so that for a class `Foo`, the instance method `bar` would be referred to as `Foo#bar`, and the class method `bar` would be referred to as `Foo::bar`. This did away with the dot entirely, leaving no room for ambiguity.

Unfortunately, this lead to a confusing situation. Beginners would often type `Foo#bar` to try to call instance methods, but were at least promptly punished for doing so because such code will not run at all. However, typing `Foo::bar` does work! Thus, an entire generation of Ruby developers were born thinking that `::` is some sort of special operator for calling class methods, and to an extent, others followed suit as a new convention emerged.

The fact that `::` will happily call methods for you has to do with internal implementation details of MRI, and so it's actually an undefined behavior, subject to change. As far as I know, there is no guarantee it will actually work as expected, and so it shouldn't be relied upon.

In your code, you should feel free to replace any method calls that use this style with ordinary `Foo.bar` calls. This actually reflects more of the true nature of Ruby, in that it doesn't emphasize the difference between class level calls and instance level calls, since that distinction isn't especially important. In documentation, things are a little trickier, but it is now generally accepted that `Foo.bar` refers to a class method and `Foo#bar` refers to an instance method. In cases where that distinction alone might be confusing, you could always be explicit, as in the example below.

```ruby
obj.bar # obj is an instance of Foo
```

If this argument wasn't convincing enough, you should know that every time you replace a `Foo::bar` call with `Foo.bar`, a brand new baby unicorn is born beneath a magnificent double rainbow. That should be reason enough to reverse this outdated practice, right?

### Reflections 

This article probably gave you more details than you ever cared to know about namespacing. But future articles will be sure to blow your mind with what else modules can do. However, if you have any questions or thoughts about what we've discussed so far, feel free to leave them in the comments section below.
  
> **NOTE:** This article has also been published on the Ruby Best Practices blog. There [may be additional commentary](http://blog.rubybestpractices.com/posts/gregory/037-issue-8-uses-for-modules.html#disqus_thread) 
over there worth taking a look at.
