In the [last issue](http://practicingruby.com/articles/48), I encouraged everyone to read Martin Fowler's classic article [Mocks Aren't Stubs](http://martinfowler.com/articles/mocksArentStubs.html). Since this article is a bit dated and leans heavily towards Java style development practices, I also offered my own commentary to hopefully bridge the gap between Fowler's insights and the modern Ruby world. Now that we have the theories behind us, today we can focus on putting these ideas into practice.

There is a style of behavior driven development that encourages mocking everything except the object under test. Fowler calls folks who follow this methodology _mockists_, and more-or-less presents this approach as a completely valid alternative to classic TDD, in which test doubles of any variety are only used when absolutely necessary. While I think that such an assessment is valid in the context Fowler originally wrote his article (2004/Java), I personally feel that the _mockist_ style that Fowler describes has no place in modern Ruby development.

That having been said, when used in moderation, mocking frameworks can make testing a whole lot easier. Today, I'll be sharing my thoughts on when to use mocks and when not to. While these are not meant to be taken as strict rules to follow, they may shed some light on a middle ground between Fowler's classicist and mockist categories.

> **NOTE:** I'm using [citrusbye/contest](http://github.com/citrusbyte/contest) and [mocha](https://github.com/floehopper/mocha) in the tests shown in this article, but the ideas should apply to any testing framework + mocking system.

### Good uses for mocks

When I think back on my testing habits, I find that virtually all of my use of mock objects falls into one or more of the following three categories:

 * Testing code which depends on an external resource of some sort (a web service, the filesystem, mail server, etc.)
 * Testing code which would involve a large amount of non-reusable setup and fixture data if you didn't mock at a high level.
 * Testing code which relies on features which are particularly computationally expensive.

Each of these scenarios has their caveats, but odds are, most moderate to large
size projects I work on hit at least one of them, and it isn't rare
to deal with all three of these issues simultaneously. That alone tells me that
having a good understanding of how to use mocks is a key part of TDD. I'll now
share some examples that hopefully help drive that point home.

<b>Isolation from external resources</b>

It would be great if our projects were completely self-contained, not having to deal with any shared resources, but this isn't realistic. Most projects need to deal with at least some external resources and may even have to tackle some systems integration problems. This often makes automating testing considerably more challenging than we would like it to be.

Thankfully, mock objects provide some shortcuts for us. While they won't help us with testing the code that needs to interact with the outside world, they can easily be used as stand-ins for our integration points when we are testing code that depends on outside resources. This makes it possible to test our high level logic without having to access whatever external resources our code needs to integrate with.

To demonstrate how useful this can be, we'll look at some simple tests from a tool I built which uses _win32ole_ to integrate with some Windows based truck routing software. Below, you can see a bit of test code that ensures a particular error gets raised when an invalid stop is added to the trip object.

```ruby
test "trip must be able to detect an invalid stop" do 
  trip = MilesDavis::Trip.create
  expect_an_invalid_stop

  error = assert_raises(MilesDavis::InvalidStopError) do 
    trip.stops << "Fakeville, FK" 
  end

  assert_equal "Cannot Find: Fakeville, FK", error.message
end 
```

If you guessed that the `expect_an_invalid_stop` method introduces a mock into
these tests, you were right! While it might look a bit like magic on a first
glance, I usually try to separate all but the most trivial mock logic into its
own helper methods to make the tests easier to maintain. Here's what
`expect_an_invalid_stop` actually does:

```ruby
def expect_an_invalid_stop
  server = mock()
  server.expects(:CheckPlaceName).returns(0)
  MilesDavis.expects(:server).returns(server)
end
```

We can now take a look at the implementation code that these tests run against. It is a simple module that gets mixed into the stops array when a new `Trip` is created.

```ruby
module StopValidation
  def <<(place)
    unless MilesDavis.server.CheckPlaceName(place) > 0
      raise InvalidStopError, "Cannot Find: #{place}"
    end

    super
  end
end
```

If you go back and re-read the test and mock code, it should be pretty clear what is going on here. When this system is actually running in production, `MilesDavis.server` refers to a _win32ole_ object, which explains the crappy camel case method names. But when running this particular test, we swap out the server call to return a mock object of our own creation.

By crafting our tests to mock out any interaction with the server, our test suite still works fine outside of the production environment. Even though the core purpose of this library is to integrate with a proprietary bit of Windows code running on a particular machine, we were able to develop all but the lowest layer entirely within our Linux and Mac-based development environments without needing any direct access to the software we were integrating with.

It's worth mentioning that although this use case was extracted directly from a real world project, it was hand picked to demonstrate the value of mocks in this sort of context. Other interactions with external resources are not so black and white. For example, if you're doing something like manipulating files on a system, it might make more sense to use temporary files than it would be to introduce mock objects. There are many other scenarios like this, so it's usually best to weigh out the costs and benefits before going full steam ahead with mocks.

That having been said, mocking external resources is almost always a valid use case, if not the most optimal one in certain situations.

<b>Avoiding complex setup + fixtures</b>

The main reason why integration with external resources is a pain is because it often requires lots of configuration and setup just to get things running. A similar phenomenon occurs internally when projects get large enough to have some complex object relations and/or advanced datastructures.

What follows is a bit of test code for a decorator that we built to wrap some low level geospatial data that we were storing via PostGIS.

```ruby
test "retreive a valid US postal area" do
  expect_postal_area_search("06511")
  geom = GeoRegion.by_postal("06511")
  assert_equal :postal, geom.interpreted_type
end
```

The mocking actually happens in `expect_postal_area_search`, which is shown
below:

```ruby
def expect_postal_area_search(zip)
  PostalArea.expects(:find_by_zcta).with(zip).returns(record_stub)
end
```

This mock emulates a simple `ActiveRecord` search, returning a stubbed out record which implements the bare minimum functionality required by our `GeoRegion` class. While somewhat uninteresting, below is the definition of `record_stub()`, for those curious.

```ruby
def record_stub
  stub(:the_geom => Object.new)
end
```

The guts of `GeoRegion` are actually a little bit complex, but our test was only meant to show that `GeoRegion.by_postal` returns an object that responds to `interpreted_type()` and returns the value `:postal`. This means we can focus on just that part of things without losing anything important.

The part of the code that does the geometry lookup is a simple delegator to `PostalArea.find_by_zcta`, which is what `expect_postal_area_search` mocks out for us. The stubbed out record it returns ends up being used in a helper method that defines the `interpreted_type` on the record via a mixin and then sets its value.

```ruby
def geom_for(record, type = nil)
  geom = (record && record.the_geom) or 
    raise UnknownFormatError, "Not a valid #{type}"

  geom.extend(Meta)
  geom.interpreted_type = type 
  geom.record = record 
  
  return geom 
end         
```

I won't bother tracing the longish execution path that lies on either side of this helper method, but the key takeaway here is that we're able to avoid to skip two layers of complexity by mocking out our call to `PostalArea` and stubbing out the actual geometry object that is associated with that `PostalArea`.

We could have loaded fixture data into our testing environment which had the relevant geospacial data to perform the sort of search we needed for this feature, but doing so would certainly be more complicated than the two simple lines we used to create our mock and stub.

Part of the reason mocks work out well here is that they allow you to focus on the behavior of `GeoRegion` rather than its implementation details. Even though under the hood a bunch of complex object manipulation is going on, we only really care about a very narrow set of functionality that `GeoRegion`'s adds as metadata to the geometry objects looked up through its search methods. If we had to actually populate the database with geometry data and concern ourselves with the messy relationships between these objects, our tests would be far less clear.

Of course, this technique only really makes sense when understanding and maintaining the mock object's interface is easier than creating the necessary setup code and fixtures to run the tests with real objects. Often times, the scales are tipped in the other direction, which I'll talk about a little later in this article. But before we get into the bad ideas, we have one more good one to cover.

<b>Mocking for performance reasons</b>

The first two techniques both had something in common: They made life easier by preventing certain code from actually being run. If we take that idea and apply it to performance, we find that running less code is usually faster than running more code.

Let's consider the following simple code that sends an email message to a group each time a new member is added.

```ruby
class Group

  def initialize(name, admin)
    @name    = name
    @admin   = admin
    @members = []
  end

  attr_reader :members, :name, :admin

  def <<(new_user)
    raise if members.include?(new_user)

    members << new_user
    broadcast("New user added", "#{new_user} joined the #{name} "+
              "group on #{Date.today}.")
  end

  def broadcast(title, content)
    mail = Mail.new

    mail.from(admin)
    mail.to(members)
    mail.subject(title)
    mail.body(content)

    mail.deliver
  end

end
```

Because `Group#broadcast` is almost entirely calls to the external Mail library, it arguably doesn't need unit tests, and instead could be covered by integration tests that set up a test mail server or something like that. However, `Group#<<` is a different story.

If we focus on the behavior of appending a user to the group, we don't actually need to focus on how `broadcast()` is implemented, we only need to verify that it is called. The following test demonstrates how to apply that line of thinking.

```ruby
test "adding users" do
  group = Group.new("Practicing Ruby", "greg@practicingruby.com")

  expect_broadcast(group, 2)

  group << "joe@example.com"
  group << "matz@example.com"

  assert_equal ["joe@example.com", "matz@example.com"], group.members
end
```

The most simple mock that reasonably covers the necessary functionality for `expect_broadcast()` is shown below.

```ruby
def expect_broadcast(group, count)
  group.expects(:broadcast).times(count)
end
```

We could actually go much farther here and verify the particular subject and content being passed to `broadcast()`, but as I said in [issue #18's mini-rant on testing](http://practicingruby.com/articles/47), I don't particularly like testing presentation logic that needs to be hand verified due to frequent superficial change. But personal preferences aside, even with a more complex set of expectations, using a mock object here is sure to be faster than actually sending an email.

This is a bit contrived example, but imagine a group object with many more methods that send broadcast emails. Add to that all the email enabled features across an application, and you'll quickly see the clock ticking longer and longer even if you do have a mail server that pipes everything to _/dev/null_.

This sort of scenario will come up in a number of different domains, and whenever it does, mock objects might be the right way to go. The main downside of using this sort of approach is that it eliminates the possibility of using your tests as a performance benchmark for your project. It is also worth noting that without proper integration tests, your mocks will happily go green in places that your real code may never be able to run. But since these issues tend to get spotted very quickly in manual testing and ordinary application use, it's usually okay to wait until this becomes a problem before worrying about it.

The three types of scenarios I've covered so far pretty much completely describe the valid use cases for mocks that have come up in my work. It isn't likely to be an exhaustive list, but I've working in a fairly large amount of projects across diverse domains and have yet to see another need for mocks that I didn't cover here. I did run up against a couple anti-patterns though, so let's take a look at those now before we wrap up.

### Bad uses for mocks

Two very popular use cases for mocks should actually be considered harmful:

* Using mocks for complete isolation of internal dependencies
* Using mocks as contracts for unwritten objects

To be sure, there are fairly strong arguments for each of these ideas, Fowler alone goes to great lengths making the case for them, and he is a moderate on these issues. But I'd argue the line of thinking is really geared towards languages that punish users from creating lots of objects with simple APIs connecting them together, such as Java. Let's take a look at some Ruby examples so that we can consider that point.

<b>Using mocks for complete isolation of internal dependencies</b>

Consider this simple variation on the theme of a user group, in which `Group#<<` constructs Person objects for each new member of a group.

```ruby
class Group
  def initialize
    @members = []
  end

  attr_reader :members

  def <<(person_name)
    members << Person.new(person_name)
  end

  def member_names
    members.map { |e| e.name }
  end
end
```

A mockist would not think about whether `Person` has external dependencies, complex setup requirements, or performance issues. They would just have started with a mock right away, perhaps something like this.

```ruby
class GroupTest < Test::Unit::TestCase
  test "adding members to a group" do
    group = Group.new

    expect_new_member("Gregory Brown")
    group << "Gregory Brown"

    expect_new_member("Jia Wu")
    group << "Jia Wu"

    assert_equal ["Gregory Brown", "Jia Wu"], group.member_names
  end

  def expect_new_member(member_name)
    Person.expects(:new).returns(stub(:name => member_name))
  end
end
```

The neat thing about the code above is that it really does create some major isolation, in that it will still allow you to test `Group#<<` and `Group#member_names` with nothing more than a bare class definition for `Person`. If we wanted to be hardcore, you could even create a `Group#new_person` method and mock that instead, and then you wouldn't even need a defined `Person` constant!

But before we get too excited, let's assume `Person` is just a trivial container method, such as the one shown below.

```ruby
class Person
  def initialize(name)
    @name = name
  end

  attr_reader :name
end
```

This code doesn't require any complex setup, it isn't using any external resources, and it doesn't have any performance intensive characteristics to it. That means that in order to test it directly, all we need to do is remove a bunch of lines from our previous test case.

```ruby
test "adding members to a group" do
  group = Group.new

  group << "Gregory Brown"
  group << "Jia Wu"

  assert_equal ["Gregory Brown", "Jia Wu"], group.member_names
end
```

By comparison, the above code is much more simple. But some smart folks still write it the other way. This is not without reason, and in fact has something to do with what happens when a change is made that causes tests to fail. To illustrate this, suppose that Person has a simple test that looks something like this.

```ruby
test "a user has a name attribute" do
  user = User.new("Gregory Brown")
  assert_equal "Gregory Brown", user.name
end
```

With the code we've seen so far, this test easily passes. But consider what happens when the implementation of User is changed to something like the code below.

```ruby
class Person
  def initialize(name)
    @name = name.upcase
  end

  attr_reader :name
end
```

The version of our test suite which uses mock objects will have one failure in the test case that is specifically checking what `Person#name` returns. It will not cause our `Group` tests to fail, because a stubbed person object is used there instead. I've included the output of a test run using that approach so you can see what that looks like.

```
  1) Failure:
test_adding_members_to_a_group(GroupTest)
<["Gregory Brown", "Jia Wu"]> expected but was
<["GREGORY BROWN", "JIA WU"]>.

  2) Failure:
test_a_user_has_a_name_attribute(PersonTest)
<"Gregory Brown"> expected but was
<"GREGORY BROWN">.
```

This is exactly what mockists don't like to see. The argument is that as your programs get more complex, the dependencies between objects get larger and larger and you end up with tens or hundreds of failing tests all because of a change in one place. This phenomena can and does occur, and it happens in smaller projects than you might think.

But still, doesn't something smell fishy?  The mock objects that are now being constructed in the tests for `Group#member_names` are now completely out of synchronization with the real specifications of the application. It isn't possible to get the output they test against in real uses of the application, and so while they adequately test the behavior of `Group#member_names`, the isolation has caused the mocks to diverge from reality, making them untrustworthy as 'living documentation' for the real system.

Personally, when I make a change that has potential system-wide affects, I prefer my tests to be verbose. Testing objects directly prevents this sort of out of sync representation of object behavior from being even possible, and so increases the reliability of the tests as both an integration testing safety net and as a documentation source.

As for sifting through the sea of information that gets spit out when you *don't* use mocks, there are ways of effectively sifting through it so as to not have problems even in very complex applications. But that is a topic more related to general debugging and may be better off described in another article.

We still have one more point to cover before we wrap up here, and this is now edging on being a massive article, so let's get to it.

<b>Using mocks as contracts for unwritten objects</b>

When writing code test first, it is possible to use mock objects as stand ins for objects that have not been defined yet. As I had mentioned before, with minor alterations we wouldn't even need to have a `Person` class defined in order to effectively test `Group#<<` and `Group#member_names`.

This is sort of neat, because it forces a radical form of behavior driven development. Since you're not working with the real collaborator objects at all in your tests, you are absolutely forced to work with their expected behaviors and not their implementations.

We've already hinted at some of the downsides of this approach though, in particular, that it is possible for our mocks can get out of sync with reality. We've seen an example of tests that don't fail, even though they describe invalid output from `User#name`. Now let's see an example of a change that does cause our original mock-based tests to fail, even though there is nothing wrong with the code itself.

```ruby
# replace the Person object with this definition, which simply renames
# Person#name to Person#full_name
#
class Person
  def initialize(full_name)
    @full_name = full_name
  end

  attr_reader :full_name
end

class Group
  # update to call the renamed Person#full_name method
  def member_names
    members.map { |e| e.full_name }
  end
end
```

When we run the non-mocked version of our tests, nothing fails, because it never explicitly mentions the name attribute on `Person`. But the same cannot be said for our mocked code, which explicitly creates stubs with a name attribute, as shown below.

```ruby
  def expect_new_member(member_name)
    Person.expects(:new).returns(stub(:name => member_name))
  end
```

You can see the test output below as evidence that our mock is now indeed broken.

```
  1) Failure:
test_adding_members_to_a_group(GroupTest)
    [/home/sandal/devel/practicing-ruby/group.rb:14:in `member_names'
     /home/sandal/devel/practicing-ruby/group.rb:14:in `map'
    ...
unexpected invocation: #<Mock:0x7ff71166e6c0>.full_name()
satisfied expectations:
- expected exactly once, already invoked once: Person.new(any_parameters)
- expected exactly once, already invoked once: Person.new(any_parameters)
- allowed any number of times, not yet invoked:
  #<Mock:0x7ff71166e6c0>.name(any_parameters)
- allowed any number of times, not yet invoked:
  #<Mock:0x7ff71166aac0>.name(any_parameters)
```

So here we see the knife cuts both ways. While it's true that our mocked code doesn't need to worry about the implementations of anything except the object under test, it does tightly bind to the interface, even when changes to those interfaces don't affect the object under test.

This allows us to make the same argument that mockists make about cascading errors, from the other side of the fence. As projects grow bigger, the amount of red tests due to brittle mock objects grows larger and larger, making it harder to see what is actually broken and what needs to be changed. But unlike the problem of noisy directly tested objects, these sort of failures only indicate a problem with the tests, not the code.

In languages where creating new objects is hard and time consuming, such a trade is probably worth considering. If we had to hand tune a Makefile, set up headers, declare variables, and consider memory management just to add a Person object like we might in C++, there might be a strong argument for how using mocks for driving tests helps you be more agile.

But in Ruby, in which our first tests can be made to pass with just a single line like the one below, you have to wonder whether the juice is worth the squeeze.

```ruby
  Person = Struct.new(:name)
```

One important thing to note is that despite my criticisms, there are folks out there who use very elegant design techniques and testing practices that can minimize the problems I have pointed out. But personally, I feel like these folks succeed in spite of the path they've chosen rather than because of it. The idea that using mocks to force you to think about design may work well as a gateway drug, but then once you've learned how to think about object design on its own, you can chuck out the training wheels and just focus on writing good code.

The examples I've shown here might be a bit biased towards demonstrating my arguments, but at least should give a starting point for considering these issues on your own.

### Reflections

We've simultaneously shown in this article that mock objects are both really damn useful and ridiculously annoying at the same time. Personally, I tend to shy away from tooling that requires you to swallow a large amount of dogma and a boatload of theory before you can even make use of it, and that is the main reason why I'm concerned about the whole mockist approach to things. From what I've seen, while a stereotypical _classicist_ is hard to come by, these _mockist_ folks that Fowler describes do exist and in my opinion, do more harm than good in getting folks to write clear, easy to understand Ruby code.

Mocking frameworks are big guns, and should be treated as such. They can be life
savers when used in moderation, but can make you pull your hair out if you use them inappropriately.

In summary, it's a bad idea to swallow bad tasting medicine with the abstract promise that it will be better for you in the end. If you can see clear benefits from the use of mocks and have weighed them out on a case by case basis against your other options, you should be fine. But if you are mostly using them because the RSpec team tells you to, you're basically screwed :)

My final disclaimer about what I've said here is that it is entirely based on my own experiences. You've worked on different problems in different environments than I have, and I'd love to know how those experiences have influenced your own thoughts on mocking.
  
> **NOTE:** This article has also been published on the Ruby Best Practices blog. There [may be additional commentary](http://blog.rubybestpractices.com/posts/gregory/052-issue-20-thoughts-on-mocking.html#disqus_thread) 
over there worth taking a look at.
