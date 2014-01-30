If you are using `send` to test private methods in your tests, you are almost certainly doing it wrong. Most private methods tend to fall into one of the following categories, none of which require `send` to test:

* A method that does not have behavior of its own (a helper function) 
* A method that actually deserves to be public on the current object 
* A method that is only private to hide a design flaw

Take a look at the three objects below and try to match them to the patterns listed above.

```ruby
class Book
  def initialize(name)
    @name = name
  end

  def available_for_purchase?
    copies_remaining > 0     
  end

  private

  def copies_remaining
    Inventory.count(:book, @name)
  end
end

module Inventory
  extend self

  def count(item_type, name)
    item_class(item_type).find_by_name(name).quantity
  end

  def receive(item_type, name, quantity)
    item_class(item_type).create(name, quantity)
  end

  private

  def item_class(item_type)
    case item_type
    when :book
      InStockBook
    when :video
      InStockVideo
    end
  end
end

class InStockBook
  def self.titles
    @titles ||= {}
  end
  
  def self.find_by_name(name)
    titles[name]
  end

  def self.create(name, quantity)
    titles[name] = new(name, quantity)
  end

  def initialize(name, quantity)
    @title     = name
    @quantity  = quantity
  end

  attr_reader :title, :quantity

  def isbn
    @isbn ||= isbn_from_service
  end

  private

  def isbn_from_service
    isbn_service_connect

    isbn = @isbn_service.find_isbn_for(@title)

    isbn_service_disconnect

    return isbn
  end

  def isbn_service_connect
    @isbn_service = IsbnService.new
    @isbn_service.connect
  end

  def isbn_service_disconnect
    @isbn_service.disconnect
  end
end
```

If you guessed that `Inventory` was the object which demonstrated a private method that doesn't implement an external behavior, you guessed right. The sole purpose of `Inventory#item_class` is just to make the code in `Inventory#count` and `Inventory#receive` a bit cleaner to read. Therefore, it'd be wasteful to write an explicit test such as the one below.

```ruby
def test_item_class
  assert_equal InStockBook, Inventory.send(:item_class, :book)
end
```

The following tests implicitly cover the functionality of `Inventory#item_class` while focusing on actual interactions through the public interface.

```ruby
def test_stocking_a_book
  Inventory.receive(:book, "Ruby Best Practices", 100)
  assert_equal 100, Inventory.count(:book, "Ruby Best Practices")
end
```

Because indirectly testing a private method will result in the same code coverage results as testing the method directly, you won't silently miss out on a failure if `Inventory#item_class` does not work as expected. However, by writing your tests this way, you focus primarily on what can be done to the object via its external interface. This leads to clearer, more maintainable tests. If a user is expected to add books through `Inventory#receive`, they should not need to know about `InStockBook`, so it can be regarded as an implementation detail. Changing the definition of `Inventory#item_class` or even removing it entirely will not require a change to these tests as long as you maintain the signature of the objects public API.

Now that we've identified the approach for testing `Inventory`, we are left with `Book` and `InStockBook` to discuss. Of the two, the problem with `Book` is a little more obvious, so we'll tackle it first.

Book implements a method called `available_for_purchase?`, which relies on a private method called `copies_remaining` to operate. The following code demonstrates a poorly implemented test.
 
```ruby
def test_copies_remaining
  book = Book.new("Ruby Best Practices")
  Inventory.receive(book.name, 10)
 
  assert_equal book.send(:copies_remaining), 10 
end
```

The reason why this is poor is because once again, we are relying on `send` to call a private method in our tests. Our theory from the previous example is that private methods do not need to be tested because they don't actually implement behavior. However, `Book#copies_remaining` seems like something you might want to actually make use of. If you imagine a web front-end for an e-commerce site, it's easy to visualize both an indicator of whether an item is in stock, as well as how many of that item are still available.

The rule of thumb here is that if a method provides a sensible behavior that fits the context of your object, it's better off to just make it public. The following test seems very natural to me.

```ruby
def test_copies_remaining
  book = Book.new("Ruby Best Practices")
  Inventory.receive(book.name, 10)
  
  assert_equal book.copies_remaining, 10 
end
```

So far we've seen two extremes: Private methods that are rightfully private and do not need to be tested explicitly, and private methods that ought to be public so that they can be tested explicitly. We will now examine the space between these two opposite ends of the spectrum.  

Let's think a bit about how we could test the `InStockBook#isbn` shown below.

```ruby
class InStockBook

  # .. other features omitted

  def isbn
    @isbn ||= isbn_from_service
  end

end
```

One way to do it the would be to mock out the call to `isbn_from_service` as we do in the following tests.

```ruby
def test_retreive_isbn
  book = InStockBook.new("Ruby Best Practices", 10)
  book.expects(:isbn_from_service).once.returns("978-0-596-52300-8")

  # Verify caching by calling isbn twice but expecting only one service
  # call to be made
  2.times { assert_equal "978-0-596-52300-8", @book.isbn }
end
```

The downside of this approach is that by mocking out the call to `isbn_from_service`, we're bypassing all of the following code, leaving it untested.

```ruby
def isbn_from_service
  isbn_service_connect

  isbn = @isbn_service.find_isbn_for(@title)

  isbn_service_disconnect

  return isbn
end

def isbn_service_connect
  @isbn_service = IsbnService.new
  @isbn_service.connect
end

def isbn_service_disconnect
  @isbn_service.disconnect
end
```

Making these methods public on `InStockBook` doesn't make much sense, but we also can't say that these are mere implementation details that can be ignored. In these situations, typically some redesign is necessary, and in this case, a simple shift of this functionality upstream to the `IsbnService` class makes the most sense.

```ruby 
class IsbnService

  def self.find_isbn_for(title)
    service = new

    service.connect
    isbn = service.find_isbn_for(title) # delegate to instance
    service.disconnect

    return isbn
  end

  # .. other functionality

end
```

This functionality can now easily be tested as a public behavior of the `IsbnService` class, where it won't get jumbled up with `InStockBook`'s logic. All that's left to do is rewrite our `InStockBook#isbn` method so that it delegates to this new class.

```ruby
class InStockBook

  # .. other features omitted

  def isbn
    @isbn ||= IsbnService.find_isbn_for(@title)
  end

end
```

Our updated `isbn` tests only need to change slightly to accommodate this
change:

```ruby
def test_retreive_isbn
  book = InStockBook.new("Ruby Best Practices", 10)
  IsbnService.expects(:find_isbn_for).with(book.title).once.
              returns("978-0-596-52300-8")

  # Verify caching by calling isbn twice but expecting only one service
  # call to be made
  2.times { assert_equal "978-0-596-52300-8", @book.isbn }
end
```

Now, when reading the tests for `InStockBook`, the developer can safely gloss
over `IsbnService`'s implementation until its contract changes. With this
dilemma solved, we've now comprehensively categorized the strategies that allow
you to avoid testing private methods without sacrificing the clarity and
coverage of your test suite.

### Reflections

We've now seen examples of how to deal with all of the following situations that might tempt us to use `send` in our tests unnecessarily:

1. A method that does not have behavior of its own (a helper function) 
1. A method that actually deserves to be public on the current object 
1. A method that is only private to hide a design flaw

Can you think of a situation where none of these approaches seem to work? Please feel free to share them in the comments section below.
  
> **NOTE:** This article has also been published on the Ruby Best Practices blog. There [may be additional commentary](http://blog.rubybestpractices.com/posts/gregory/034-issue-5-testing-antipatterns.html#disqus_thread) 
over there worth taking a look at.
