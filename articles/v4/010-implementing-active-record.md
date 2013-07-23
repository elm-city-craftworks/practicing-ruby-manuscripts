> This two part article explores the challenges involved in
> building a minimal implementation of the Active Record pattern. 
> [Part 1 (Issue 4.8)](http://practicingruby.com/articles/60) provides
> some basic background information about the problem and
> walks through some of the low level structures that are
> needed to build an ORM. Part 2 (this issue) builds on top of
> those structures to construct a complete Active Record
> object.

### Building object-oriented mixins

One thing that makes the Active Record pattern challenging to implement is that
involves shoehorning a bunch of persistence-related functionality into model
objects. In the case of Rails, models inherit from `ActiveRecord::Base` which
has dozens of modules mixed into it. This inheritance-based approach is the
common way of doing complex behavior sharing in Ruby, but comes at a [high
maintainence cost](http://practicingruby.com/articles/62). This is one of the
main design challenges that
[BrokenRecord](https://github.com/elm-city-craftworks/broken_record) attempts to solve.

Because this is a tricky problem, it helps to explore these ideas by
solving an easier problem first. For example, suppose that you have the following trivial
`Stack` object and you want to extend it with some `Enumerable`-like
functionality without mixing `Enumerable` directly into the `Stack` object:

```ruby
class Stack
  def initialize
    @data = []
  end

  def push(obj)
    data.push(obj)
  end

  def pop
    data.pop
  end

  def size
    data.size
  end

  def each
    data.reverse_each { |e| yield(e) }
  end

  private

  attr_reader :data
end
```

You could use an `Enumerator` for this purpose, as shown in the following
example:

```ruby
stack = Stack.new

stack.push(10)
stack.push(20)
stack.push(30)

enum  = Enumerator.new { |y| stack.each { |e| y.yield(e) } }
p enum.map { |x| "Has element: #{x}" } #=~
# ["Has element: 30", "Has element: 20", "Has element: 10"]    
```

This is a very clean design, but it makes it so that you have to interact with
both a `Stack` object and an `Enumerator`, which feels a bit tedious. With a
little effort, the two could be unified under a single interface while keeping
their variables and internal method calls separated:

```ruby
class EnumerableStack
  def initialize
    @stack = Stack.new
    @enum  = Enumerator.new { |y| @stack.each { |e| y.yield(e) } }       
  end

  def respond_to_missing?(m, *a)
    [@stack, @enum].find { |e| e.respond_to?(m) }
  end

  def method_missing(m, *a, &b)
    obj = respond_to_missing?(m)

    return super unless obj
    obj.send(m, *a, &b)
  end
end
```

From the external perspective, `EnumerableStack` still looks and 
feels like an ordinary `Enumerable` object:

```ruby
stack = EnumerableStack.new

stack.push(10)
stack.push(20)
stack.push(30)

p stack.map { |x| "Has element: #{x}" } #=~
# ["Has element: 30", "Has element: 20", "Has element: 10"]  
```

Unfortunately, it is painful to implement objects this way. If you
applied this kind of technique throughout a codebase without introducing some
sort of abstraction, you would end up having to write a ton of very boring
`respond_to_missing?` and `method_missing` calls. It would be better to have
an object that knows how to delegate methods automatically, such as 
the `Composite` object in the following example:

```ruby
class EnumerableStack
  def initialize
    stack = Stack.new
    enum  = Enumerator.new { |y| stack.each { |e| y.yield(e) } }      
    
    @composite = Composite.new
    @composite << stack << enum
  end

  def respond_to_missing?(m, *a)
    @composite.receives?(m)
  end

  def method_missing(m, *a, &b)
    @composite.dispatch(m, *a, &b)
  end
end
```

The neat thing about this approach is that the `EnumerableStack`
object now only needs to keep track of a single variable, even though it is
delegating to multiple objects. This makes it safe to extract some
of the functionality into a mix-in without the code becoming too brittle:

```ruby
class EnumerableStack
  include Composable

  def initialize
    stack = Stack.new
    enum  = Enumerator.new { |y| stack.each { |e| y.yield(e) } }      

    # features is a simple attribute containing a Composite object
    features << stack << enum
  end
end
```

The end result looks pretty clean, but using the `Composable` 
mixin to solve this particular problem is massively overkill. 
Mixing the `Enumerable` module directly into the `Stack` object
is not that hard to do, and is unlikely to have any adverse
consequences. Still, seeing how `Composable` can be used to
replace one of the most common applications of mixins makes
it much easier to understand how this technique can be 
applied in more complex scenarios. The good news is 
that as long as you have a rough idea of how `Composable` 
works in this context, you will have no trouble understanding
how it is used in BrokenRecord.

To test whether or not you understand the basic pattern, take a look at the
following code and see if you can figure out how it works. Don't worry about
the exact implementation details, just compare the following code to the other
examples in this section and think about what the purpose of this module is:

```ruby
module BrokenRecord
  module Mapping
    include Composable

    def initialize(params)
      features << Record.new(params)
    end

    def self.included(base)
      base.extend(ClassMethods)
    end

    module ClassMethods
      include Composable

      def map_to_table(table_name)
        features << Relation.new(:name         => table_name,
                                 :db           => BrokenRecord.database,
                                 :record_class => self)
      end
    end
  end
end
```

If you guessed that mixing `BrokenRecord::Mapping` into a class will cause any
unhandled messages to be delegated to `BrokenRecord::Relation` at the class 
level and to `BrokenRecord::Record` at the instance level, then you guessed
correctly! If you're still stuck, it might help to recall how this mixin 
is used:

```ruby
class Article
  include BrokenRecord::Mapping

  map_to_table :articles
end

article = Article.create(:title => "Great article", :body => "Wonderful!")
p article.title.upcase #=> "GREAT ARTICLE"
```

If you consider that the definition of `BrokenRecord::Mapping` above is its
complete implementation, it becomes clear that the methods being called in this
example need to come from somewhere. Now, it should be easier to see that
`Relation` and `Record` are where those methods come from.

You really don't need to know the exact details of how 
the `Composable` module works, because it is based entirely on the 
ideas already discussed in this article. However, if `Composable` still feels a
bit too magical, go ahead and [study its
implementation](https://github.com/elm-city-craftworks/broken_record/blob/master/lib/broken_record/composable.rb)
before reading on. For bonus points, pull the code down and try to 
recreate the `EnumerableStack` example on your own machine.

Once you feel that you have a good grasp on how `Composable` works, you can
continue on to see how it can be used to implement an Active Record object.

### Implementing basic CRUD operations

The complex relationships that Active Record objects depend upon make them a bit
challenging to understand and analyze. But like any complicated system,
you can gain some foundational knowledge by starting with a very simple example as an
entry point and digging deeper from there.

In the case of BrokenRecord, a good place to start is with a somewhat trivial
model definition:

```ruby
class Article
  include BrokenRecord::Mapping

  map_to_table :articles

  def published?
    status == "published"
  end
end
```

You found out earlier when you looked at `BrokenRecord::Mapping` that it exists
primarily to extend classes with functionality provided by
`BrokenRecord::Relation` at the class level, and `BrokenRecord::Record` at the
instance level. Because `BrokenRecord::Mapping` provides a fairly complicated 
`initialize` method, it is safe to assume that `Article` objects should be
created by factory methods rather than instantiated directly. The following 
code demonstrates how that works:


```ruby
Article.create(:title  => "A great article",
               :body   => "The rain in Spain...",
               :status => "draft")

Article.create(:title  => "A mediocre article",
               :body   => "Falls mainly in the plains",
               :status => "published")

Article.create(:title  => "A bad article",
               :body   => "Is really bad!",
               :status => "published")

Article.all.each do |article|
  if article.published?
    puts "PUBLISHED: #{article.title}"
  else
    puts "UPCOMING: #{article.title}"
  end
end
```

If you ignore what is going on inside the `each` block for the moment, it is
easy to spot two factory methods being used in the previous example:
`Article.create` and `Article.all`. To track down where these methods are coming
from, you need to take a look at `BrokenRecord::Relation`, because that is where
class-level method calls on `Article` are forwarded to if `Article` does not
handle them itself. But before you do that, keep in mind that this is how
that object gets created in the first place:

```ruby
  def map_to_table(table_name)
    features << Relation.new(:name         => table_name,
                             :db           => BrokenRecord.database,
                             :record_class => self)
  end
```

If you note that `map_to_table :articles` is called within the `Article`
class, you can visualize the call to `Relation.new` in the previous example as
being essentially the same as what you see below:

```ruby
features << Relation.new(:name          => :articles,
                         :db            => BrokenRecord.database,
                         :record_class  => Article)
```

Armed with this knowledge, it should be easier to make sense of the
`BrokenRecord::Relation` class, which is shown in its entirety below. Pay
particular attention to the `initialize` method, and just skim the rest of the
method definitions; it isn't important to fully understand them until later.

```ruby
module BrokenRecord
  class Relation
    include Composable

    def initialize(params)
      self.table = Table.new(:name => params.fetch(:name),
                             :db   => params.fetch(:db))

      self.record_class = params.fetch(:record_class)

      features << CRUD.new(self) << Associations.new(self)
    end

    attr_reader :table

    def attributes
      table.columns.keys
    end

    def new_record(values)
      record_class.new(:relation => self,
                       :values   => values,
                       :key      => values[table.primary_key])
    end

    def define_record_method(name, &block)
      record_class.send(:define_method, name, &block)
    end

    private

    attr_reader :record_class
    attr_writer :table, :record_class
  end
end
```

The main thing to notice about `BrokenRecord::Relation` is that its main purpose
is to glue together a `BrokenRecord::Table` object with a user-defined record
class, such as the `Article` class we've been working with in this example. The
rest of its functionality is provided by the `Relation::CRUD` and
`Relation::Associations` objects via composition. Because `Article.all` and
`Article.create` are both easily identifiable as CRUD operations, the `Relation::CRUD` 
object is the next stop on your tour:

```ruby
module BrokenRecord
  class Relation
    class CRUD
      def initialize(relation)
        self.relation = relation
      end

      def all
        table.all.map { |values| relation.new_record(values) }
      end

      def create(values)
        id = table.insert(values)    
      
        find(id)
      end

      def find(id)
        values = table.where(table.primary_key => id).first

        return nil unless values

        relation.new_record(values)
      end

      # ... other irrelevant CRUD operations omitted
      
      private

      attr_accessor :relation

      def table
        relation.table
      end
    end
  end
end
```

At this point, you should have noticed that both `create()` and
`all()` are defined by `Relation::CRUD`, and it is ultimately these 
methods that get called whenever you call `Article.create` 
and `Article.all`. Whether you trace `Relation::CRUD#create` or `Relation::CRUD#all`, you'll find
that they both interact with the `Table` object provided by `Relation`, and that
they both call `Relation#new_record`, and they don't do much more than that. 
To keep things simple, we'll follow the path that `Relation::CRUD#all` takes:

```ruby
def all
  table.all.map { |values| relation.new_record(values) }
end
```

This method calls `BrokenRecord::Table#all`, which as you saw in 
[Issue 4.8](http://practicingruby.com/articles/60) returns an
array of hashes representing the results returned from the 
database when a trivial `select * from articles` query is issued. 
For this particular data set, the following results get 
returned:

```ruby
[
 { :id     => 1, 
   :title  => "A great article", 
   :body   => "The rain in Spain...", 
   :status => "draft" }, 

 { :id     => 2, 
   :title  => "A mediocre article", 
   :body   => "Falls mainly in the plains", 
   :status => "published"}, 

 { :id     => 3, 
   :title  => "A bad article", 
   :body   => "Is really bad!", 
   :status => "published" }
]
```

Taking a second look at the `Relation::CRUD#all` method, it is easy to
see that this is being transformed by a simple `map` call which passes each of
these hashes to `Relation#new_record`. I had asked you to skim over that
method earlier, but now would be a good time to take a second look at its
definition:

```ruby
module BrokenRecord
  class Relation
    def new_record(values)
      record_class.new(:relation => self,
                       :values   => values,
                       :key      => values[table.primary_key])
    end
  end
end
```

If you recall that in this context `record_class` is a reference to `Article`,
it becomes easy to visualize this call as something similar to what is 
shown below:

```ruby
values = { :id     => 1, 
           :title  => "A great article", 
           :body   => "The rain in Spain...", 
           :status => "draft" }

Article.new(:relation => some_relation_obj,
            :values   => values,
            :key      => 1)
```

As you discovered before, `Article` does not provide its own `initialize`
method, and instead inherits the definition provided by 
`BrokenRecord::Mapping#initialize`:

```ruby
module Mapping
  include Composable

  def initialize(params)
    features << Record.new(params)
  end
end
```

If you put all the pieces together, you will find that calls to
`Article.all` or `Article.create` return instances of `Article`, but
those instances are imbued with functionality provided by a `Record`
object, which in turn hold a reference to a `Relation` object 
that ties everything back to the database. By now you're probably feeling like
the Active Record pattern is a bit of a 
[Rube Goldberg machine](http://www.youtube.com/watch?v=qybUFnY7Y8w), and that
isn't far from the truth. Don't worry though, the next section should help 
tie everything together for you.

### Implementing the Active Record object itself

Earlier, I had asked you to ignore what was going on in the `each` block of the
original example that kicked off this exploration, because I wanted to show you
how `Article` instances get created before discussing how they work. Now that you
have worked through that process, you can drop down to the instance level to
complete the journey. Using the same code reading strategy as what you used
before, you can start with the `Article#published?` and `Article#title` 
calls in the following example and see where they take you:

```ruby
Article.all.each do |article|
  if article.published?
    puts "PUBLISHED: #{article.title}"
  else
    puts "UPCOMING: #{article.title}"
  end
end
```

A second look at the `Article` class definition reveals that it implements
the `published?` method but does not implement the `title` method; the latter call gets 
passed along to `BrokenRecord::Record` automatically. Similarly, the internal call 
to `status` gets delegated as well:

```ruby
class Article
  include BrokenRecord::Mapping

  map_to_table :articles

  def published?
    status == "published"
  end
end
```

To understand what happens next, take a look at how the `BrokenRecord::Record` class works:

```ruby
module BrokenRecord
  class Record
    include Composable

    def initialize(params)
      self.key      = params.fetch(:key, nil)
      self.relation = params.fetch(:relation)

      # NOTE: FieldSet (formally called Row) is a simple Struct-like object
      features << FieldSet.new(:values     => params.fetch(:values, {}),
                               :attributes => relation.attributes)
    end

    # ... irrelevant functionality omitted ...

    private

    attr_accessor :relation, :key
  end
end
```

By now you should be able to quickly identify `BrokenRecord::FieldSet` as the object that
receives any calls that `Record` does not answer itself. The good
news is that you already know how `FieldSet` works, because it was discussed in
detail in [Issue 4.8](http://practicingruby.com/articles/60). But if you need a
refresher, check out the following code:

```ruby
values = { :id     => 1, 
           :title  => "A great article", 
           :body   => "The rain in Spain...", 
           :status => "draft" }

fieldset = BrokenRecord::FieldSet.new(:values     => values,
                                      :attributes => values.keys)

p fieldset.title  #=> "A great article"
p fieldset.status #=> "draft"
```

If you read back through the last few examples, you should be able to see how
the data provided by `Relation` gets shoehorned into one of these `FieldSet`
objects, and from there it becomes obvious how the `Article#title` and `Article#status`
messages are handled.

If `FieldSet` is doing all the heavy lifting, you may be wondering why the
`Record` class needs to exist at all. Those details were omitted from the
original example, so it is definitely a reasonable question to ask. To find your
answer, consider the following example of updating database records:

```ruby
articles = Article.where(:status => "draft")

articles.each do |article|
  article.status = "published"
  article.save
end
```

In the example you worked through earlier, data was being read and not written,
and so it was hard to see how `Record` offered anything more than a layer of
indirection on top of `FieldSet`. However, the example shown above changes that
perspective significantly by giving a clear reason for `Record` to hold a
reference to a `Relation` object. While `Article#status=` is provided by
`FieldSet`, the `Article#save` method is provided by `Record`, and is defined as
follows:

```ruby
module BrokenRecord
  class Record
    
    # ... other details omitted ...

    def save
      if key
        relation.update(key, to_hash)
      else
        relation.create(to_hash)
      end
    end 
  end
end
```

From this method (and others like it), it becomes clear that `Record` is
essentially a persistent `FieldSet` object, which forms the essence of what an
Active Record object is in its most basic form.

### EXERCISE: Implementing minimal association support 

The process of working through the low level foundations built up in [Issue
4.8](http://practicingruby.com/articles/60) combined with this article's 
extensive walkthrough of how BrokenRecord implements some basic CRUD 
functionality probably gave you enough learning moments to make you want to 
quit while you're ahead. That said, if you are looking to dig a little deeper, I'd recommend
trying to work your way through BrokenRecord's implementation of associations
and see if you can make sense of it. The following example should serve as a
good starting point:

```ruby
class Article
  include BrokenRecord::Mapping

  map_to_table :articles

  has_many :comments, :key   => :article_id, 
                      :class => "Comment"

  def published?
    status == "published"
  end
end

class Comment
  include BrokenRecord::Mapping

  map_to_table :comments

  belongs_to :article, :key   => :article_id,
                       :class => "Article"
end


Article.create(:title  => "A great articles",
               :body   => "The Rain in Spain",
               :status => "draft")


Comment.create(:body => "A first comment",  :article_id => 1)
Comment.create(:body => "A second comment", :article_id => 1)


article = Article.find(1)

puts "#{article.title} -- #{article.comments.count} comments"
puts article.comments.map { |e| "  * #{e.body}" }.join("\n")
```

Because not all the features used by this example are covered in this article,
you will definitely need to directly reference the [full source of
BrokenRecord](https://github.com/elm-city-craftworks/broken_record) to complete
this exercise. But don't worry, by now you should be familiar with most of its
code, and that will help you find your way around. If you attempt this 
exercise, please let me know your thoughts and questions about it!

### Reflections

Object-oriented mixins seems very promising to me, but also full of open 
questions and potential pitfalls. While they seem to work well in this toy 
implementation of Active Record, they may end up creating as many problems as
they solve. In particular, it remains to be seen how this kind of modeling would
impact performance, debugging, and introspection of Ruby objects. Still, the
pattern does a good enough job of handling a very complex architectural
pattern to hint that some further experimentation may be worthwhile.

Going back to the original question I had hoped to answer in the first part of
this article about whether or not the Active Record pattern is inherently complex, I
suppose we have found out that there isn't an easy answer to that question. My
BrokenRecord implementation is conceptually simpler than the Rails-based
ActiveRecord, but only implements a tiny amount of functionality. I think that
the closest thing to a conclusion I can come to here is that the traditional
methods we use for object modeling in Ruby are certainly complex, and so any
system which attempts to implement large-scale architectural patterns in Ruby
will inherit that complexity unless it deviates from convention.

That all having been said, reducing complexity is about  more than just
preferring composition over inheritance and reducing the amount of magic in our
code. The much deeper questions that we can ask ourselves is whether these very
complicated systems we build are really necessary, or if they are a
consequence of piling [abstractions on top of abstractions](http://timelessrepo.com/abstraction-creep) 
in order to fix some fundamental low-level problem.

While this article was a fun little exploration into the depths of a
complex modeling problem in Ruby, I think its real point is to get us to
question our own tolerance for complexity at all levels of what we do. If you
have thoughts to share about that, I would love to hear them.
