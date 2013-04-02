While it may not seem like it at first, you can learn a great deal about Ruby by building something as simple as a static website generator. Although the task itself may seem a bit dull, it provides an opportunity to practice a wide range of Ruby idioms that can be applied elsewhere whenever you need to manipulate text-based data or muck around with the filesystem. Because text and files are everywhere, this kind of practice can have a profound impact on your ability to write elegant Ruby code.

Unfortunately, there are two downsides to building a static site generator as a learning exercise: it involves a fairly large time commitment, and in the end you will probably be better off using [Jekyll](http://github.com/mojombo/jekyll) rather than maintaining your own project. But don't despair, I wrote this article specifically with those two points in mind!

In order to make it easier for us to study text and file processing tricks, I broke off a small chunk of Jekyll's functionality and implemented a simplified demo app called [Jackal](http://github.com/elm-city-craftworks/jackal). Although it would be a horrible idea to attempt to use this barely functional counterfeit to maintain a blog or website, it works great as a tiny but context-rich showcase for some very handy Ruby idioms.

### A brief overview of Jackal's functionality

The best way to get a feel for what Jackal can do is to [grab it from Github](https://github.com/elm-city-craftworks/jackal) and follow the instructions in the README. However, because it only implements a single feature, you should be able to get a full sense of how it works from the following overview.

Similar to Jekyll, the main purpose of Jackal is to convert Markdown-formatted posts and their metadata into HTML files. For example, suppose we have a file called **_posts/2012-05-09-tiniest-kitten.markdown** with the following contents:

```
---
category: essays
title: The tiniest kitten
---

# The Tiniest Kitten

Is not nearly as **small** as you might think she is.
```

Jackal's job is to split the metadata from the content in this file and then generate a new file called **_site/essays/2012/05/09/tiniest_kitten.html** that ends up looking like this:


```html
<h1>The Tiniest Kitten</h1>

<p>Is not nearly as <strong>small</strong> as you might think she is.</p>
```

If Jackal were a real static site generator, it would support all sorts of fancy features like layouts and templates, but I found that I was able to generate enough "teaching moments" without those things, and so this is pretty much all there is to it. You may want to spend a few more minutes [reading its source](http://github.com/elm-city-craftworks/jackal) before moving on, but if you understand this example, you will have no trouble understanding the rest of this article.

Now that you have some sense of the surrounding context, I will take you on a guided tour of through various points of interest in Jackal's implementation, highlighting the parts that illustrate generally useful techniques.

### Idioms for text processing

While working on solving this problem, I noticed a total of four text processing idioms worth mentioning.

**1) Enabling multi-line mode in patterns**

The first step that Jackal (and Jekyll) need to take before further processing can be done on source files is to split the YAML-based metadata from the post's content. In Jekyll, the following code is used to split things up:

```ruby
if self.content =~ /^(---\s*\n.*?\n?)^(---\s*$\n?)/m
  self.content = $POSTMATCH
  self.data    = YAML.load($1)
end
```

This is a fairly vanilla use of regular expressions, and is pretty easy to read even if you aren't especially familiar with Jekyll itself. The main interesting thing about it that it uses the `/m` modifier to make it so that the pattern is evaluated in multiline-mode. In this particular example, this simply makes it so that the group which captures the YAML metadata can match multiple lines without explicitly specifying the intermediate `\n` characters. The following contrived example should help you understand what that means if you are still scratching your head:

```
>> "foo\nbar\nbaz\nquux"[/foo\n(.*)quux/, 1]
=> nil
>> "foo\nbar\nbaz\nquux"[/foo\n(.*)quux/m, 1]
=> "bar\nbaz\n"
```

While this isn't much of an exciting idiom for those who have a decent understanding of regular expressions, I know that for many patterns can be a mystery, and so I wanted to make sure to point this feature out. It is great to use whenever you need to match a semi-arbritrary blob of content that can span many lines.

**2) Using MatchData objects rather than global variables**

While it is not necessarily terrible to use variables like `$1` and `$POSTMATCH`, I tend to avoid them whenever it is not strictly necessary to use them. I find that using `String#match` feels a lot more object-oriented and is more aesthetically pleasing:

```ruby
if md = self.content.match(/^(---\s*\n.*?\n?)^(---\s*$\n?)/m)
  self.content = md.post_match
  self.data    = md[1]
end
```

If you combine this with the use of Ruby 1.9's named groups, your code ends up looking even better. The following example is what I ended up using in Jackal:

```ruby
if (md = contents.match(/^(?<metadata>---\s*\n.*?\n?)^(---\s*$\n?)/m))
  self.contents = md.post_match
  self.metadata = YAML.load(md[:metadata])
end
```

While this does lead to somewhat more verbose patterns, it helps quite a bit with readability and even makes it possible to directly use `MatchData` objects in a way similar to how we would work with a parameters hash.

**3) Enabling free-spacing mode in patterns**

I tend to be very strict about keeping my code formatted so that my lines are under 80 characters, and as a result of that I find that I am often having to think about how to break up long statements. I ended up using the `/x` modifier in one of Jackal's regular expressions for this purpose, as shown below:

```ruby
module Jackal
  class Post
    PATTERN = /\A(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})-
                (?<basename>.*).markdown\z/x

    # ...
  end
end
```

This mode makes it so that patterns ignore whitespace characters, making the previous pattern functionally equivalent to the following pattern:

```ruby
/\A(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})-(?<basename>.*).markdown\z/x
```

However, this mode does not exist primarily to serve the needs of those with obsessive code formatting habits, but instead exists to make it possible to break up and document long regular expressions, such as in the following example:

```ruby
# adapted from: http://refactormycode.com/codes/573-phone-number-regex

PHONE_NUMBER_PATTERN = /^
  (?:
    (?<prefix>\d)             # prefix digit
    [ \-\.]?                  # optional separator
  )?
  (?:
    \(?(?<areacode>\d{3})\)?  # area code
    [ \-\.]                   # separator
  )?
  (?<trunk>\d{3})             # trunk
  [ \-\.]                     # separator
  (?<line>\d{4})              # line
  (?:\ ?x?                    # optional space or 'x'
    (?<extension>\d+)         # extension
  )?
$/x
```

This idiom is not extremly common in Ruby, perhaps because it is easy to use interpolation within regular expressions to accomplish similar results. However, this does seem to be a handy way to document your patterns and arrange them in a way that can be easily visually scanned without having to chain things together through interpolation.

**4) Making good use of Array#join**

Whenever I am building up a string from a list of elements, I tend to use `Array#join` rather than string interpolation (i.e. the `#{}` operator) if I am working with more than two elements. As an example, take a look at my implementation of the `Jackal::Post#dirname` method:

```ruby
module Jackal
  class Post
    def dirname
      raise ArgumentError unless metadata["category"]

      [ metadata["category"], 
        filedata["year"], filedata["month"], filedata["day"] ].join("/")
    end
  end
end
```

The reason for this is mostly aesthetic, but it gives me the freedom to format my code any way I would like, and is a bit easier to make changes to.

> **NOTE:** Noah Hendrix pointed out in the [comments on this article](http://practicingruby.com/articles/57#comments) that for this particular example, using `File.join` would be better because it would take platform-specific path syntax into account.

### Idioms for working with files and folders

In addition to the text processing tricks that we've already gone over, I also noticed four idioms for doing various kinds of file and folder manipulation that came in handy.

**1) Manipulating filenames**

There are three methods that are commonly used for munging filenames: `File.dirname`, `File.basename`, and `File.extname`. In Jackal, I ended up using two out of three of them, but could easily imagine how to make use of all three.

I expect that most folks will already be familiar with `File.dirname`, but if that is not the case, the tests below should familiarize you with one of its use cases:

```ruby
describe Jackal::Page do
  let(:page) do
    posts_dir = "#{File.dirname(__FILE__)}/../fixtures/sample_app/_posts"
    Jackal::Page.new("#{posts_dir}/2012-05-07-first-post.markdown")
  end

  it "must extract the base filename" do
    page.filename.must_equal("2012-05-07-first-post.markdown")
  end
end
```

When used in conjunction with the special `__FILE__` variable, `File.dirname` is used generate a relative path. So for example, if the `__FILE__` variable in the previous tests evaluates to `"test/units/page_test.rb"`, you end up with the following return value from `File.dirname`:

```ruby
>> File.dirname("test/units/page_test.rb")
=> "test/units"
```

Then the whole path becomes `"tests/units/../fixtures/sample_app/_posts"`, which is functionally equivalent to `"test/fixtures/sample_app/_posts"`. The main benefit is that should you run the tests from a different folder, `__FILE__` would be updated accordingly to still generate a correct relative path. This is yet another one of those idioms that is hardly exciting to those who are already familiar with it, but is an important enough tool that I wanted to make sure to mention it.

If you feel like you understand `File.dirname`, then `File.basename` should be just as easy to grasp. It is essentially the opposite operation, getting just the filename and stripping away the directories in the path. If you take a closer look at the tests above, you will see that `File.basename` is exactly what we need in order to implement the behavior hinted at by `Jackal::Page#filename`. The irb-based example below should give you a sense of how that could work:

```
>> File.basename("long/path/to/_posts/2012-05-09-tiniest-kitten.markdown")
=> "2012-05-09-tiniest-kitten.markdown"
```

For the sake of simplicity, I decided to support Markdown only in Jackal posts, but if we wanted to make it more Jekyll-like, we would need to support looking up which formatter to use based on the post's file extension. This is where `File.extname` comes in handy:

```
>> File.extname("2012-05-09-tiniest-kitten.markdown")
=> ".markdown"
>> File.extname("2012-05-09-tiniest-kitten.textile")
=> ".textile"
```

Typically when you are interested in the extension of a file, you are also interested in the name of the file without the extension. While I have seen several hacks that can be used for this purpose, the approach I like best is to use the lesser-known two argument form of `File.basename`, as shown below:

```
>> File.basename("2012-05-09-tiniest-kitten.textile", ".*")
=> "2012-05-09-tiniest-kitten"
>> File.basename("2012-05-09-tiniest-kitten.markdown", ".*")
=> "2012-05-09-tiniest-kitten"
```

While these three methods may not look especially beautiful in your code, they provide a fairly comprehensive way of decomposing paths and filenames into their parts. With that in mind, it is somewhat surprising to me how many different ways I have seen people attempt to solve these problems, typically resorting to some regexp-based hacks.

**2) Using Pathname objects**

Whenever Ruby has a procedural or functional API, it usually also has a more object-oriented way of doing things as well. Manipulating paths and filenames is no exception, and the example below shows that it is entirely possible to use `Pathname` objects to solve the same problems discussed in the previous section:

```
>> require "pathname"
=> true
>> Pathname.new("long/path/to/_posts/2012-05-09-tiniest-kitten.markdown").dirname
=> #<Pathname:long/path/to/_posts>
>> Pathname.new("long/path/to/_posts/2012-05-09-tiniest-kitten.markdown").basename
=> #<Pathname:2012-05-09-tiniest-kitten.markdown>
>> Pathname.new("long/path/to/_posts/2012-05-09-tiniest-kitten.markdown").extname
=> ".markdown"
```

However, because doing so doesn't really simplify the code, it is hard to see the advantages of using `Pathname` objects in this particular example. A much better example can be found in `Jackal::Post#save`:


```ruby
module Jackal
  class Post
    def save(base_dir)
      target_dir = Pathname.new(base_dir) + dirname
      
      target_dir.mkpath

      File.write(target_dir + filename, contents)
    end
  end
end
```

The main reason why I used a `Pathname` object here is because I needed to make use of the `mkpath` method. This method is roughly equivalent to the UNIX `mkdir -p` command, which handles the creation of intermediate directories automatically. This feature really comes in handy for safely generating a deeply nested folder structure similar to the ones that Jekyll produces. I could have alternatively used the `FileUtils` standard library for this purpose, but personally find `Pathname` to look and feel a lot more like a modern Ruby library.

Although its use here is almost coincidental, the `Pathname#+` method is another powerful feature worth mentioning. This method builds up a `Pathname` object through concatenation. Because this method accepts both `Pathname` objects and `String` objects as arguments but always returns a `Pathname` object, it makes easy to incrementally build up a complex path. However, because `Pathname` objects do more than simply merge strings together, you need to be aware of certain edge cases. For example, the following irb session demonstrates that `Pathname` has a few special cases for dealing with absolute and relative paths:

```
>> Pathname.new("foo") + "bar"
=> #<Pathname:foo/bar>
>> Pathname.new("foo") + "/bar"
=> #<Pathname:/bar>
>> Pathname.new("foo") + "./bar"
=> #<Pathname:foo/bar>
>> Pathname.new("foo") + ".////bar"
=> #<Pathname:foo/bar>
```

Unless you keep these issues in mind, you may end up introducing subtle errors into your code. However, this behavior makes sense as long as you can remember that `Pathname` is semantically aware of what a path actually is, and is not meant to be a drop in replacement for ordinary string concatenation.

**3) Using File.write**

When I first started using Ruby, I was really impressed by how simple and expressive the `File.read` method was. Because of that, it was kind of a shock to find out that simply writing some text to a file was not as simple. The following code felt like the opposite of elegance to me, but we all typed it for years:

```ruby
File.open(filename, "w") { |f| f << contents }
```

In modern versions of Ruby 1.9, the above code can be replaced with something far nicer, as shown below:

```ruby
File.write(filename, contents)
```

If you look back at the implementation of `Jackal::Post#save`, you will see that I use this technique there. While it is the simple and obvious thing to do, a ton of built up muscle memory typically causes me to forget that `File.write` exists, even when I am not concerned at all about backwards compatibility concerns.

Another pair of methods worth knowing about that help make some other easy tasks more elegant in a similar way are `File.binread` and `File.binwrite`. These aren't really related to our interests with Jackal, but are worth checking out if you ever work with binary files.

**4) Using Dir.mktmpdir for testing**

It can be challenging to write tests for code which deals with files and complicated folder structures, but it doesn't have to be. The tempfile standard library provides a lot of useful tools for dealing with this problem, and `Dir.mktmpdir` is one of its most useful methods.  

I like to use this method in combination with `Dir.chdir` to build up a temporary directory structure, do some work in it, and then automatically discard all the files I generated as soon as my test is completed. The tests below are a nice example of how that works:

```ruby
it "must be able to save contents to file" do
  Dir.mktmpdir do |base_dir|
    post.save(base_dir)

    Dir.chdir("#{base_dir}/#{post.dirname}") do
      File.read(post.filename).must_equal(post.contents)
    end
  end
end
```
This approach provides an alternative to using mock objects. Even though this code creates real files and folders, the transactional nature of `Dir.mktmpdir` ensures that tests won't have any unexpected side effects from run to run. When manipulating files and folders is part of the core job of an object (as opposed to an implementation detail), I prefer testing in this way rather than using mock objects for the sake of realism.

The `Dir.mktmpdir` method can also come in handy whenever some complicated work needs to be done in a sandbox on the file system. For example, I [use it in Bookie](https://github.com/sandal/bookie/blob/45e0c4d0a575026deff79732b3c4c737f1c6f15c/lib/bookie/emitters/epub.rb#L19-46) to store the intermediate results of a complicated text munging process, and it seems to work great for that purpose.

### Reflections

Taken individually, these text processing and file management idioms only make a subtle improvement to the quality of your code. However, if you get in the habit of using most or all of them whenever you have an opportunity to do so, you will end up with much more maintainable code that is very easy to read.

Because many languages make text processing and file management hard, and because Ruby also has low level APIs that work in much the same way as those languages, it is often the case that folks end up solving these problems the hard way without ever realizing that there are nicer alternatives available. Hopefully this article has exposed you to a few tricks you haven't already seen before, but if it hasn't, maybe you can share some thoughts on how to make this code even better!
