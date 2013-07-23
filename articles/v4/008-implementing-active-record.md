> (ORM) is one of the most complex things you could ever touch, and we choose it
> over and over again without thinking at all because everybody is doing it. It
>  is really complex! You waste an inordinate amount of your time on it, and
>  you need to look at it. -- [Rich Hickey, RailsConf 2012 (video)](http://www.youtube.com/watch?v=rI8tNMsozo0#t=1289s)

Depending on the kind of work you do, the claim that object-relational mapping
is _"one of the most complex things you could ever touch"_ is just 
as likely to be shocking as it is to be blindingly obvious. Because 
ActiveRecord (and other Ruby ORMs) provide highly abstracted ways of solving
common problems, it is easy to ignore the underlying complexity involved in even
the most simple things that we use ORM for. But just as there is a huge
difference between driving a car and repairing one, the cost of 
understanding ORM is much higher than simply making use of it.

In this two-part article, I will walk you through a minimal 
implementation of the [Active
Record](http://en.wikipedia.org/wiki/Active_record) pattern so that you can more 
easily understand what we take for granted when we use this flavor 
of ORM in our projects.

### Is the Active Record pattern inherently complex?

Whenever we talk about an Active Record in Ruby, it is extremely common for us
to immediately tie our thoughts to the Rails implementation of this pattern,
even though the concept itself was around before Rails was invented. If we 
accept the Rails-centric view of our world, the question of whether
ActiveRecord is a complex piece of software is trivial to answer; we only need
to look at the `ActiveRecord::Base` object to see that it has all of the 
following complecting characteristics:

* Hundreds of instance methods
* Hundreds of class methods
* Over a dozen instance variables
* Over a dozen class instance variables
* Several class variables ([a construct that's inherently complex!](http://www.oreillynet.com/ruby/blog/2007/01/nubygems_dont_use_class_variab_1.html))
* A 40 level deep lookup path for class methods
* A 46 level deep lookup path for instance methods
* Dozens of kinds of method_missing hacks
* No encapsulation whatsoever between mixed in modules

But if you look back at how Martin Fowler defined the concept of an Active
Record in his 2003 book "Patterns of Enterprise Application Architecture", you
will find that the pattern does not necessarily require such a massively complex
implementation. In fact, Fowler's definition of an Active Record included any
object that could do most or all of the following things:

* Construct an instance of the Active Record from a SQL result set row
* Construct a new instance for later insertion into the table
* Use static finder methods to wrap commonly used SQL queries and return
  Active Record objects
* Update the database and insert data into the Active Record
* Get and set fields
* Implement some pieces of business logic

Clearly, the Rails-based ActiveRecord library does all of these things, but it
also does a lot more. As a result, it is easy to conflate the
coincidental complexity of this very popular implementation with the inherent 
complexity of its underlying pattern. This is a major source of
confounding in many discussions about software design for Rails developers, and
is something I want to avoid in this article.

With that problem in mind, I built a minimal implementation of the Active Record pattern called
[BrokenRecord](https://github.com/elm-city-craftworks/broken_record) which will
help you understand the fundamental design challenges involved in implementing this
particular flavor of ORM. As long as you keep in mind that BrokenRecord exists
primarily to facilitate thought experiments and is not meant to be used in
production code, it should provide an easy way for you to explore a number
of questions about ORM in general, and the Active Record pattern in particular.

### The ingredients for implementing an Active Record object

Now that you know what an Active Record is in its most generic form, how would you
go about implementing it? To answer that question, it may help to reflect upon
an example of how Active Record objects are actually used. The following code
is a good place to start, because it illustrates some of the most basic 
features you can expect from an Active Record object.

```ruby
## Create an article with a few positive comments.

article1 = Article.create(:title => "A great article",
                          :body  => "Short but sweet!")


Comment.create(:body => "Supportive comment!", :article_id => article1.id)
Comment.create(:body => "Friendly comment!",   :article_id => article1.id)

## Create an article with a few negative comments.

article2 = Article.create(:title => "A not so great article",
                          :body  => "Just as short")

Comment.create(:body => "Angry comment!",      :article_id => article2.id)
Comment.create(:body => "Frustrated comment!", :article_id => article2.id)
Comment.create(:body => "Irritated comment!",  :article_id => article2.id)

## Display all the articles and their comments 

Article.all.each do |article|
  puts %{
    TITLE: #{article.title}
    BODY: #{article.body}
    COMMENTS:\n#{article.comments.map { |e| "    - #{e.body}" }.join("\n")}
  }
end
```

While this example omits a bit of setup code, it is not hard to see that it
produces the following output:

```
    TITLE: A great article
    BODY: Short but sweet!
    COMMENTS:
    - Supportive comment!
    - Friendly comment!


    TITLE: A not so great article
    BODY: Just as short
    COMMENTS:
    - Angry comment!
    - Frustrated comment!
    - Irritated comment!  
```

Despite its simple output, there is a lot going in this little 
code sample. To gain a better sense of what is happening under
the hood, take a look at how the `Article` and `Comment` objects
are defined:

```ruby
class Article
  include BrokenRecord::Mapping
  
  map_to_table :articles

  has_many :comments, :key   => :article_id,
                      :class => "Comment"
end

class Comment
  include BrokenRecord::Mapping

  map_to_table :comments

  belongs_to :article, :key   => :article_id,
                       :class => "Article"
end
```

Because `BrokenRecord::Mapping` does not implement the naming 
shortcuts that `ActiveRecord::Base` uses, the connection between 
these objects and the underlying database schema is much more 
explicit. If you take a look at how the `articles` and `comments` 
tables are defined, it should be straightforward to understand how 
this all comes together:

```sql
 create table articles ( 
   id     INTEGER PRIMARY KEY,
   title  TEXT,
   body   TEXT,
 );

 create table comments (
   id          INTEGER PRIMARY KEY,
   body        TEXT,
   article_id  INTEGER,
   FOREIGN KEY(article_id) REFERENCES articles(id)
 );
```

If you haven't been paying close attention to what kinds of things you would
need to build in order to make this code work, go ahead and quickly re-read this
section with that in mind. Once you've done that, examine the following grocery 
list of Active Record ingredients and see if they match your own:

1. Storage and retrieval of record data in an SQL database. 
  (e.g. `Article.create` and `Article.all`)
2. Dynamic generation of accessors for record data. (e.g. `article.body`)
3. Dynamic generation of associations methods (e.g. `article.comments`),
  including the ability to dynamically look up the associated class.
  (e.g. `:class => "Comments"`)
4. The ability to wrap all these features up into a single module mix-in.

This list easily demonstrates that a fair amount of complicated code is needed 
to support the most basic uses of Active Record objects, even when the
pattern is stripped down to its bare essentials. But it is one thing
to have a rough sense that a problem is complex, and a different thing
entirely to familiarize yourself with its nuances. The former insight leads you to
appreciate your magical tools; the latter helps you master them.

To help you dig deeper, I will guide you through the code that handles each
of these responsibilities in `BrokenRecord`, explaining how it all works along
the way. We will start by exploring some low level constructs that help
simplify the implementation of Active Record objects, and then in [part 2](http://practicingruby.com/articles/63) 
we will look at how the whole system comes together.

### Abstracting away the database

Using the Active Record pattern introduces tight coupling between classes
containing bits of domain logic and the underlying persistence layer. However,
this does not mean that an Active Record ought to directly tie itself to 
a low-level database adapter. With that in mind, introducing a simple
object to handle basic table manipulations and queries is a good way
to reduce the brittleness of this tightly coupled design.

The following example shows how `BrokenRecord::Table` can be used directly to
solve the same problem that was shown earlier. As you read through it, try to
imagine how the `BrokenRecord::Mapping` module might be implemented using this
object as a foundation.

```ruby
## create a couple table objects

articles = BrokenRecord::Table.new(:name => "articles",
                                   :db   => BrokenRecord.database)

comments = BrokenRecord::Table.new(:name => "comments",
                                   :db   => BrokenRecord.database)

## create an article with some positive comments

a1 = articles.insert(:title => "A great article", 
                     :body  => "Short but sweet")

comments.insert(:body => "Supportive comment!", :article_id => a1)
comments.insert(:body => "Friendly comment!",   :article_id => a1)

## create an article with some negative comments

a2 = articles.insert(:title => "A not so great article", 
                     :body  => "Just as short")

comments.insert(:body => "Angry comment!",      :article_id => a2)
comments.insert(:body => "Frustrated comment!", :article_id => a2)
comments.insert(:body => "Irritated comment!",  :article_id => a2)

## Display the articles and their comments

articles.all.each do |article|
  responses = comments.where(:article_id => article[:id])

  puts %{
    TITLE: #{article[:title]}
    BODY: #{article[:body]}
    COMMENTS:\n#{responses.map { |e| "    - #{e[:body]}" }.join("\n") }
  }
end
```

Despite the superficial similarity between the features provided by
the `BrokenRecord::Mapping` mixin and the `BrokenRecord::Table` class,
there are several key differences that set them apart from one another:

1) `Mapping` assumes that `BrokenRecord.database` holds a
reference to an appropriate database adapter, but `Table` requires
the database adapter to be injected. This means that unlike `Mapping`, the
`Table` class has no dependencies on global state. 

2) Most of the methods in `Mapping` return instances of whatever
object it gets mixed into, but `Table` always returns primitive
values such as arrays, hashes, and integers. This means that
`Mapping` needs to make assumptions about the interfaces of other
objects, and `Table` does not.

3) `Mapping` implements a big chunk of its functionality via class methods, 
but `Table` does not rely on any special
class-level behavior. This means that `Table` can be easily tested
without generating anonymous classes or doing awkward cleanup tasks.

The `Mapping` mix-in is convenient to use because it can introduce 
persistence into any class, but it bakes in a few assumptions that you
can't easily change. By contrast, the `Table` object expects you to wire more
things up by hand, but is conceptually simple and very flexible. This is exactly
the kind of tension to expect between higher and lower levels of abstraction,
and is not necessarily a sign of a design problem.

If these two components were merged into a single entity, the 
conflict between their design priorities would quickly lead 
to creating an object with a split-personality. Whenever that happens, 
complexity goes through the roof, and so does the cost
of change. By allowing `Mapping` to delegate much of its functionality to 
a `Table` object, it is possible to sidestep these concerns and gain 
the best of both worlds.

### Encapsulating record data

One of the defining characteristics of an Active Record is that ordinary
accessors can be used to retrieve and manipulate its data. As a
result, basic operations on Active Record objects end up looking like 
plain old Ruby code, such as in the following example:

```ruby
Article.all.each do |article|
  puts %{
    TITLE: #{article.title}
    BODY: #{article.body}
    COMMENTS:\n#{article.comments.map { |e| "    - #{e.body}" }.join("\n")}
  }
end
```

The interesting part about getters and setters for Active Record objects 
is that they need to be dynamically generated. To refresh your memory, take a
second look at the class definition for `Article`, and note that it contains
no explicit definitions for the `Article#title` and `Article#body` methods.

```ruby
class Article
  include BrokenRecord::Mapping
  
  map_to_table :articles

  has_many :comments, :key   => :article_id,
                      :class => "Comment"
end
```

In the above code, `map_to_table` ties the `Article` class to a database 
table, and the columns in that table determine what accessors need to 
be defined. Through a low-level call to `BrokenRecord::Table`, it 
is possible to get back an array of column names, as shown below:

```ruby
  table.columns.keys #=> [:id, :title, :body]
```

If you assume that `Article` will not store field values directly, but instead
delegate to some sort of value object, Ruby's built in `Struct` object might
come to mind as a way to solve this problem. After all, it does make 
dynamically generating a value object with accessors quite easy:

```ruby
  article_container = Struct.new(:id, :title, :body)

  article = article_container.new
  article.title = "A fancy article"
  article.body  = "This is so full of class, it's silly"

  # ... 
```

Using a `Struct` for this purpose is a fairly standard idiom, and it is not
necessarily a bad idea. But despite how simple they appear to be on the surface,
the lesser known features of `Struct` objects make them very complex. In
addition to accessors, using a `Struct` also gives you all of the 
following functionality:

```ruby
  # array-like indexing
  article[1] #=> "A fancy article"

  # hash-like indexing with both symbols and strings
  article[:title] == article[1]       #=> true
  article[:title] == article["title"] #=> true 

  # Enumerability
  article.count       #=> 3
  article.map(&:nil?) #=> [true, false, false]

  # Pair-wise iteration
  article.each_pair { |k,v| p [k,v] }
  
  # Customized inspect output
  p article #=> #<struct id=nil, title="A fancy article", 
                # body="This is so full of class, it's silly">
```

While this broad interface makes `Struct` very useful for certain data
processing tasks, they are much more often used in scenarios in which a simple
object with dynamic accessors would be a much better fit. The
`BrokenRecord::FieldSet` class implements such an object while 
maintaining a minimal API: 

```ruby
module BrokenRecord
  class FieldSet
    def initialize(params)
      self.data = {}

      attributes  = params.fetch(:attributes)
      values      = deep_copy(params.fetch(:values, {}))

      attributes.each { |name| data[name] = values[name] }

      build_accessors(attributes)
    end

    def to_hash
      deep_copy(data)
    end

    private

    attr_accessor :data

    def deep_copy(object)
      Marshal.load(Marshal.dump(object))
    end

    def build_accessors(attributes)
      attributes.each do |name|
        define_singleton_method(name) { data[name] }
        define_singleton_method("#{name}=") { |v| data[name] = v }
      end
    end
  end
end
```

 The most important thing to note about this code is that
 `BrokenRecord::FieldSet` makes it just as easy to create 
 a dynamic value object as `Struct` does:

```ruby
article = BrokenRecord::FieldSet.new(:attributes => [:id, :title, :body])
article.title = "A fancy article"
article.body  = "This is so full of class, its silly"

# ...
```

The similarity ends there, mostly because `BrokenRecord::FieldSet` does not 
implement most of the features that `Struct` provides. Another important difference
is that `BrokenRecord::FieldSet` does not rely on an anonymous intermediate class to
implement its functionality. This helps discourage the use of class inheritance
for code reuse, which in turn reduces overall system complexity.

In addition to these simplifications, `BrokenRecord::FieldSet` also attempts to 
adapt itself a bit better to its own problem domain. Because `FieldSet`
objects need to be used in conjunction with `Table` objects, they need to be 
more hash-friendly than `Struct` objects are. In particular, it must easy 
to set  the values of the `FieldSet` object using a hash, and it must be easy to 
convert a `FieldSet` back into a hash. The following example demonstrates 
that both of those requirements are handled gracefully:

```ruby
article_data = { :id    => 1,
                 :title => "A fancy article",
                 :body  => "This is so full of class, it's silly" }

article = BrokenRecord::FieldSet.new(:attributes => [:id, :title, :body],
                                     :values       => article_data)

p article.title #=> "A fancy title"

p article.to_hash == article_data #=> true

article.title = "A less fancy title"

p article.to_hash == article_data #=> false
p article.to_hash[:title]         #=> "A less fancy title"
```

While it may be a bit overkill to roll your own object for the sole purpose of
removing features from an existing well supported object, the fact that
`BrokenRecord::FieldSet` also introduces a few new features of its own makes it more
reasonable to implement things this way. More could definitely be said about the
trade-offs involved in making this kind of design decision, but they are very 
context dependent, and that makes them a bit tricky to generalize.

### Reflections

The objects described in this article may seem a bit austere,
but they are easy to reason about once you gain some familiarity with them. In
the [second part of this article (Issue
4.10)](http://practicingruby.com/articles/63), you will be able to see these
objects in the context which they are actually used, which will help you
understand them further.

The main theory I am trying to test out here is that I believe simple low level 
constructs tend to make it easier to build simple higher level constructs.
However, there is a very real tension between conceptual simplicity and
practical ease-of-use, and that can lead to some complicated design decisions.

What do you think about these ideas? Are the techniques that I've shown so far more
confusing than they are enlightening? Do you have a better idea for how to
approach this problem? No matter what is on your mind, if you have thoughts on
this topic, I want to hear from you!

> **BONUS CONTENT:** If you're curious about what it looks like for me to put the "finishing touches" on a Practicing Ruby article, see [this youtube video](http://www.youtube.com/watch?v=bojXlV1mFNY). Be warned however, I am barely capable of using a computer, and so it's likely to be painful to watch me work.

