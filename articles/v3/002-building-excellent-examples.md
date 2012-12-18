Good code examples are the secret sauce that makes Practicing Ruby a high-quality learning resource. That said, the art of building excellent examples is one that I think all programmers should practice, not just those folks out there trying to teach for a living. The ability to express ideas clearly through well-focused snippets of code is key to writing good tests, documentation, bug reports, code reviews, demonstrations, and a whole lot of other stuff, too.

In this article, I've identified five patterns I use for expressing ideas through code examples. These techniques run the gamut from building contrived "Hello World" programs to crafting full-scale sample applications using a literate programming style. Each technique has its own strengths and weaknesses, and I've done what I could to outline them where possible.

Although this isn't necessarily a comprehensive list, it can help you start to improve the way you write your examples and also serves as a good jumping-off point for further discussion on the topic.

### Contrived examples

For any given programming language or software library, the odds are pretty good that the first example you'll run is a contrived "Hello World" program. The following example is taken from the Sinatra web framework but is similar in spirit to pretty much every other "Hello World" application out there:

```ruby
require 'sinatra'

get '/hi' do
  "Hello World!"
end
```

This kind of example seems quite useless on the surface and is neither interesting nor educational. As it turns out, these characteristics are precisely what make this "Hello World" program perfect! The contrived nature of the example allows it to serve as a simple sanity check for someone who is trying out Sinatra for the first time.

If you try to run this example and find that it doesn't work correctly, there are only a few possible points of failure. In most cases, not even getting a "Hello World" program to run correctly can be blamed on one of three things: out-of-date documentation, issues with your environment, or user error. The fact that there are very few moving parts makes it much easier for you to determine the source of your problem than it would be if the example were significantly more complex. This ease of debugging is precisely why most introductory tutorials start off with a "Hello World" program rather than something more exciting.

Although the most common use case for contrived examples is to construct "Hello World" applications, there are other use cases for this technique as well. In particular, contrived examples are a good fit for discussions about syntactic or structural differences between two pieces of code. As an example, consider a short tutorial that explains why a user might want to use Ruby's `attr_reader` functionality. It could start by showing a `Person` class that implements accessors explicitly:

```ruby
class Person
  def initialize(name, email)
    @name  = name
    @email = email
  end

  def name
    @name
  end

  def email 
    @email
  end
end
```

A followup example could then be provided to show how to simplify the code via `attr_reader`:

```ruby
class Person
  def initialize(name, email)
    @name  = name
    @email = email
  end

  attr_reader :name, :email
end
```

This scenario is very simplistic when compared to the class definitions we write in real projects, but the absence of complicated functionality makes it easier for the reader to focus on the syntactic differences between the two examples. It also allows the novice Ruby programmer to think of the difference between explicitly defining accessors and using `attr_reader` as a simple structural transformation rather than something with complex semantic differences. Although this mental model is not 100 percent accurate, it emphasizes the big picture, which is what actually matters for a novice programmer. The simplicity of these examples makes the general pattern much easier to remember, which justifies hiding a few things behind the curtain to be revealed later.

Unfortunately, the ability of contrived examples to hide the semantics of our programming constructs is just as often a drawback as it is an asset. The more complex a concept is, the more dangerous it is to present simplistic examples rather than working through more realistic scenarios. For example, it is common for object-oriented programming tutorials to use real-world objects and hierarchies to explain how class inheritance works, but the disconnect between these models and the kinds that real software projects implement is so great that this approach completely obfuscates the real power and purpose of object-oriented programming. By choosing a scenario that may feel natural to the reader but does not fit naturally with the underlying programming constructs, this sort of tutorial fails to emphasize the right details and leaves the door open for a wide range of misconceptions. These incorrect assumptions end up getting in the way of learning real object-oriented programming techniques rather than helping develop an understanding of them.

I could easily rant on this topic, but someone else did it for me by writing a great mailing list post entitled [Goodbye, shitty Car extends Vehicle object-orientation tutorial](http://lists.canonical.org/pipermail/kragen-tol/2011-August/000937.html). Despite the somewhat inflammatory title, it is a very insightful post, and I strongly recommend reading it if you want to see a strong argument for the limitations of contrived examples as teaching tools.

Figuring out where to draw the line between when it is appropriate to use a contrived example and when to use one that is based on a practical application is tricky. In general, I try to keep in mind that the purpose of a contrived example is specifically to remove context from the picture. Outside of "Hello World" programs and simple syntactic transformations, a lack of context hurts more than it helps, and so I try to avoid contrived examples as much as I can for pretty much every other use case. 

### Cheap counterfeits

One of my favorite techniques for teaching programming concepts is to construct cheap counterfeits that emulate the surface-level behavior of a more complicated structure. These "poor man's implementations" are similar to contrived examples in that they can hide as much complexity as they'd like from the reader but, because they are grounded by some realistic scenario, do not suffer from being totally disconnected from practical applications.

I have used this technique extensively throughout Practicing Ruby and my other written works, and it almost always works out well. In fact, the issue on [Implementing Enumerable and Enumerator in Ruby](http://practicingruby.com/articles/4) was entirely based on this strategy and turned out to be one of the most popular articles I've written for this journal. Although you are probably already very familiar with this pattern as a Practicing Ruby reader, I can still provide a bit of extra insight by decomposing it for you.

The purpose of building a cheap counterfeit is not to gain a deep understanding of how a certain construct actually works. Instead, the purpose of a counterfeit is to teach people how to steal ideas from other interesting bits of code for their own needs. For example, take the previous `attr_reader` example:

```ruby
class Person
  def initialize(name, email)
    @name  = name
    @email = email
  end

  attr_reader :name, :email
end
```

This is a great feature, because it replaces tedious boilerplate methods with a concise declarative statement. But without some sort of explanation as to how it works, `attr_reader` feels pretty magical and might be perceived as a special case that the Ruby internals are responsible for handling. This misconception can easily be cleared up by showing how to implement a cheap counterfeit version of `attr_reader` in application code:

```ruby
class Module
  def my_attr_reader(*args)
    args.each do |a|
      define_method(a) { instance_variable_get("@#{a}") }
    end
  end
end

class Person
  def initialize(name, email)
    @name  = name
    @email = email
  end

  my_attr_reader :name, :email
end
```

If teaching programmers how to use `attr_reader` is like treating them to a nice fish dinner, teaching them how to implement it is like giving them a fishing pole and showing them how to catch their own meals. Seeing a practical use of `define_method` opens the doors for a huge range of other applications, all of which hinge on the simple concept of dynamic method definition. For example, a similar technique could be used to convert hideous method names like `test_a_user_must_be_able_to_log_in` into the elegant syntax shown here:

```ruby
test "A user must be able to log in" do
  # your test code here
end
```

There are countless other applications of dynamic method definition, many of which I expect Practicing Ruby readers are already familiar with. The point here is that a single example that demystifies a certain technique can make a huge difference in what possibilities someone sees in a given system. This payoff is what makes cheap counterfeits such a tremendously good teaching tool.

An important thing to keep in mind, however, is that this technique is useful mostly for teaching concepts, as opposed to showing someone how a feature is really implemented. If you actually look into the implementation of `attr_reader`, you'll find a number of edge cases that this cheap counterfeit example does not take into consideration. Although these subtleties are not especially relevant if you're just trying to give a contextualized example of how `define_method` can be used, they would be important to point out if you were trying to write a specification for how `attr_reader` is meant to work, which is why cheap counterfeits are not a substitute for case studies of real code but instead serve a different purpose entirely.

### Simplified examples

Digging directly into the source code of a project is the most direct way to understand how its features are implemented, but it can be a somewhat disorienting process. Production code in all but the most trivial projects tends to accumulate edge cases, error-checking code, and other bits of cruft that make it harder to see what the core ideas are. When giving a talk about how something is implemented or writing documentation for potential contributors, it is sometimes helpful to provide simplified examples that demonstrate the key functionality while minimizing distractions.

Suppose I want to do a lightning talk about how [MiniTest](https://github.com/seattlerb/minitest) is implemented, with the goal of attracting new contributors to the project. In a talk like that, I'd definitely need to discuss a bit about how assertions work. A logical place to start might be the `Assertions#assert` method:

```ruby
def assert test, msg = nil
  msg ||= "Failed assertion, no message given."
  self._assertions += 1
  unless test then
    msg = msg.call if Proc === msg
    raise MiniTest::Assertion, msg
  end
  true
end
```

The implementation of `assert` is simple enough that I could probably show it as-is without losing my audience. But if I keep in mind that this code is going to be shown on a slide for just a few seconds, I might show the following simplified example instead:

```ruby
def assert(test, msg=nil)
  msg ||= "Failed assertion, no message given."

  raise(MiniTest::Assertion, msg) unless test
  true
end
```

This code omits some implementation details, but it preserves the main idea, which is that MiniTest's assertions work by raising an exception when a test fails. The fact that `assert` is where the number of assertions is counted is fairly obvious and only adds noise when you want to get a rough idea for how the code works at a glance. Likewise, the fact that a message can be passed in as a `Proc` object rather than a string is an interesting but obscure edge case that does not need to be emphasized. By removing these two statements from the method definition, the core behavior is easier to notice.

The process of creating a simplified example starts with looking at the original source code and then determining which details are essential to expressing the idea you want to express and which details can be considered background noise. The next step is to construct an example that serves as a functional subset of the original implementation when used within a certain context. You don't want to deviate too much from the original idea, but you can clean up the syntax a bit where appropriate to make the example easier to understand. From there, you can treat the example as a substitute for the real implementation for the purposes of demonstration; you just need to make sure to point out that you have simplified things a bit.

With this MiniTest example, the simplified version of the code is only slightly less complicated than the original, so the benefits of using this technique are a bit subdued. In practice, you're much more likely to run into situations in which there are dozens of lines of implementation code but only a handful of them are central to the idea that you are trying to express. In those situations, this pattern is especially effective at cutting through the cruft to get at the real meat of the problem you want to focus on. However, it's worth keeping in mind that even relatively small and easy-to-understand chunks of code can be simplified if they happen to include statements that are not directly relevant to the point you are trying to make.

### Reduced examples

A reduced example is one that reproduces a certain behavior in the most simple possible way. This technique is most commonly used for putting together bug reports and is one of the most important skills you can have as a software developer.

In the [Ruby Best Practices](http://rubybestpractices.com/) book, I told a story about a bug that was spotted in Prawn and how we reduced the original report to something much more simple in order to discover the root cause of the problem. Because this is still the best example I've found of this process in action, I'll summarize that story here rather than telling a new one.

A developer sent us the following bug report to demonstrate that our code for generating fixed-width columns of text had a problem that was causing page breaks to be inserted unnecessarily:

```ruby
Prawn::Document.generate("span.pdf") do
  span(350, :position => :center) do 
    text "Here's some centered text in a 350 point column. " * 100
  end
  
  text "Here's my sentence."

  bounding_box([50,300], :width => 400) do 
    text "Here's some default bounding box text. " * 10 

    pos = bounds.absolute_left - margin_box.absolute_left
    span(bounds.width, :position => pos) do
      text "The rain in Spain falls mainly on the plains. " * 300 
    end
  end

  text "Here's my second sentence."
end
```

In this example, he expected all of his text to be rendered on one page and was trying to show that each time he used the `span` construct, an unnecessary page break was created. He showed that this was the case both within the default page boundaries and within a manually specified bounding box. As far as user reports go, this example was pretty good, because it was specifically designed to show the problem he was having and was clearly not just some broken production code that he wanted help with.

That having been said, an understanding of how Prawn works under the hood made it possible to simplify this example quite a bit, even before investigating further. Because the default page boundaries in Prawn are implemented in terms of bounding boxes and the `bounding_box` method just temporarily swaps those dimensions with new ones, the second part of this report was superfluous. Removing it got the reproducible sample down to the example shown here:

```ruby
Prawn::Document.generate("span.pdf") do
  span(350) do 
    text "Here's some text in a 350pt wide column. " * 20
  end

  text "This text should appear on the same page as the spanning text"
end
```

In making this reduction, I also did some other minor cleanup chores such as reworking the text to be self-documenting and removing the `:position => :center` option for `span`, because it didn't affect the outcome. At this point, even someone without experience in how Prawn works would be able to more easily spot the problem in the example.

Although `span` is not a trivial construct, it had only two possible points of failure: the `bounding_box` method and the `canvas` method. Because `bounding_box` is fundamental to pretty much everything Prawn does and we had plenty of evidence that it was working as expected, we turned our attention to `canvas`.

The purpose of `canvas` is  to execute the contents of a block while ignoring all document margins and bounding boxes in place, essentially converting everything to absolute coordinates on the page. After the block is executed, it is supposed to keep the text pointer wherever it left off, which means that it should not trigger a pagebreak unless the text flows beyond the bottom of the page. To test this behavior, we coded up the following example:

```ruby
Prawn::Document.generate("canvas_sets_y_to_0.pdf") do 
  canvas { text "Some text at the absolute top left of the page" }
  text "This text should not be after a pagebreak" 
end
```

After running this example, we noticed that it exhibited the same defect that we saw in the user's bug report. Because this method is almost as deep down the Prawn call chain as you can go, it became clear that at this point we had our reduced example. The benefit of drilling down like this became apparent when we converted our sample code into a regression test:

```ruby
class CanvasTest < Test::Unit::TestCase
  def setup 
    @pdf = Prawn::Document.new
  end

  def test_canvas_should_not_reset_y_to_zero 
    after_text_position = nil

    @pdf.canvas do 
      @pdf.text "Hello World" 
      after_text_position = @pdf.y
    end

    assert_equal after_text_position, @pdf.y 
  end
end
```

After seeing this test fail and applying a quick patch that got it to go green, we went back and ran the original bug report the user provided us with. As predicted, the bad behavior went away and things were once again working as expected.

The benefits of reducing the example before writing a regression test were tremendous. Not only was the test easier to write, but it also ended up capturing the problem at a much lower level than it would have if we immediately set in with codifying the bug report as a unit test. In addition to these benefits, the reduction process itself greatly simplified the debugging process, as it allowed us to proceed methodically to find the root of the problem.

I've mostly used this reduction technique while debugging, but it can also be a useful way to find your way around a complex codebase. By starting with a practical example that exercises the system from the outermost layer, you can drill down into the code and trace your way through the call chains to find how some particular aspect of the software works. Although this is a less direct approach than just reading the documentation, it will give you a better fundamental understanding of how the system hangs together, and it's a fun way to practice code reading.

Whether you are exploring a new codebase or tracking down a bug, the reduction process limits the scope of the things you need to think about, allowing you to dedicate your attention in a more focused way. This effect is somewhat similar to what we find when we make use of simplified examples but is more of a drilling down process than it is a pruning process. Both have their merits, and they can even be used in combination at times.

### Sample applications 

Although all of the techniques I've discussed so far can be quite useful for studying, investigating, and teaching about specific issues, none of them are particularly suitable for demonstrating big-picture topics. When you want to emphasize how things come together, as opposed to how each individual part works, nothing beats the combination of a sample application with a good walkthrough tutorial. Several issues from Practicing Ruby Volume 2 made use of this format and were very well received by the readers here.

In [Learning new things step-by-step](http://practicingruby.com/articles/6), I built a small game for the purpose of demonstrating how to develop software in tiny bite-sized chunks. The nice thing about this approach is that it allows the reader to follow along at home (either mentally or by literally running the code themselves), while proceeding at their own pace. I hope it also encourages folks to experiment and draw their own conclusions rather than just rigidly following a predefined script.

In [Building Unix-style command line applications](http://practicingruby.com/articles/9), I tackled the creation of a Ruby clone of the Unix `cat` utility by focusing on distinct areas of functionality, one at a time. This is a slightly less linear format than the step-by-step approach of the game development article, but it allows readers to look at the complete application from several different angles, depending on the topics that interested them most.

Finally, in [Designing business reporting applications](http://practicingruby.com/articles/13), I totally break away from linearity by presenting the source code of a full application in literate programming style. This approach allows readers to study the real implementation code and my commentary side by side and to bounce around as they see fit. This lack of explicit structure encourages readers to explore in a free-form fashion rather than focusing on some predefined areas of interests.

Although these articles were some of the most successful ones that I've published here at Practicing Ruby, they were also among the most challenging to write. I had to apply a much higher standard of writing clear and concise code than I would if I were simply trying to make a project easy enough for me to maintain on my own. The prose was tricky to organize, because it's hard to decide which areas to emphasize and which to gloss over in a complete application. For these reasons, sample applications can be a cumbersome and time-consuming learning resource to produce. However, the investment seems to be well worth it in the end.

### Reflections

Writing good examples can be seriously hard work. This is why all too often we see people overusing contrived examples or simply attempting to pass off unrefined snippets of production code as learning materials. However, code examples in all of their myriad forms lay the foundation for how we communicate our ideas as software developers.

An important thing to remember when writing code examples is that the process is in many ways similar to writing prose. If we simply spit out a brain dump without thinking about how it will be interpreted and understood by others, we will end up with crappy results. But if we remember that the main goal of writing our examples is to communicate an idea to our fellow programmers, we naturally begin to ask the questions that lead us to improve our work.

I hope that by sharing these few patterns with you, I've given you some useful ideas for how to improve your code communication skills. Following the patterns I've outlined here will lead you to writing better examples for your documentation, bug reports, unit tests, tutorials, and quite a few other things as well. 

Though the techniques I've shown here are ones that work well in a wide range of contexts, I am sure there are other approaches worth learning about. If you've seen a novel use of code examples in the wild, please let me know! I'd also be happy to hear any other thoughts you have on this topic, and I wouldn't mind helping a few folks come up with good examples for the projects they're working on. If you've got something you want me to take a look at, just leave a comment and I'll be sure to get back to you.
