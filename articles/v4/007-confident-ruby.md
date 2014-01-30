*This article was contributed by [Avdi Grimm](http://avdi.org). Avdi has been wrangling Ruby code for over a
decade and shows no signs of slowing down. He is the author of
[*Exceptional Ruby*](http://exceptionalruby.com) and
[*Objects on Rails*](http://objectsonrails.com). His next book,
[*Confident Ruby*](http://confidentruby.com), focuses on writing Ruby
code with a confident and straightforward style.*

### Losing the plot

Have you ever read a "choose your own adventure" book? Nearly every page ends with a question like this:

> * If you fight the angry troll with your bare hands, turn to page 137.
> * If you try to reason with the troll, turn to page 29.
> * If you don your invisibility cloak, turn to page 6.

You'd pick one option, turn to the indicated page, and the story would
continue.

Did you ever try to read one of those books from front to back? It's a
surreal experience. The story jumps forward and back in
time. Characters appear out of nowhere. One page you're crushed by the
fist of an angry troll, and on the next you're just entering the
troll's realm for the first time.

What if _each individual page_ was this kind of mish-mash? What if
every page read like this:

>   You exit the passageway into a large cavern. Unless you came from
>   page 59, in which case you fall down the sinkhole into a large
>   cavern. A huge troll, or possibly a badger (if you already visited
>   Queen Pelican), blocks your path. Unless you threw a button down the
>   wishing well on page 8, in which case there nothing blocking your
>   way. The [troll or badger or nothing at all] does not look happy to
>   see you.
> 
> * If you came here from chapter 7 (the Pool of Time), go back to the
>   top of the page and read it again, only imagine you are watching the
>   events happen to someone else.
> 
> * If you already received the invisibility cloak from the aged
>   lighthouse-keeper, and you want to use it now, go to page 67. Otherwise, forget you read anything about an invisibility cloak.
> 
> * If you are facing a badger (see above), and you choose to run away,
>   turn to page 93…

Not the most compelling narrative, is it? The story asks you to carry
so much mental baggage for it that just getting through a page is
exhausting.

### Code as narrative

What does this have to do with software? Well, code can tell a story
as well. It might not be a tale of high adventure and intrigue. But
it's a story nonetheless; one about a problem that needed to be
solved, and the path the developer(s) chose to accomplish that task.

A single method is like a page in that story. And unfortunately, a lot
of methods are just as convoluted, equivical, and confusing as that
made-up page above.

In the following sections, we'll take a look at some examples of code
that unnecessarily obscures the storyline of a method. We'll also
explore some techniques for minimizing distractions and writing
methods that straightforwardly convey their intent.

### Secure the borders

Here is some code that's having some trouble sticking to the plot:

```ruby
require 'date'

class Employee
  attr_accessor :name
  attr_accessor :hire_date

  def initialize(name, hire_date)
    @name      = name
    @hire_date = hire_date
  end

  def due_for_tie_pin?
    raise "Missing hire date!" unless hire_date
    ((Date.today - hire_date) / 365).to_i >= 10
  end

  def covered_by_pension_plan?
    # TODO Someone in HR should probably check this logic
    ((hire_date && hire_date.year) || 2000) < 2000
  end

  def bio
    if hire_date
      "#{name} has been a Yoyodyne employee since #{hire_date.year}"
    else
      "#{name} is a proud Yoyodyne employee"
    end
  end
end
```

We can speculate about the history of this class. It looks like over
the course of development, three different developers discovered that
`#hire_date` might sometimes be `nil`. They each chose to handle this
fact in a slightly different way. The one who wrote
`#due_for_tie_pin?` added a check that raises an exception if the hire
date is missing. The developer responsible for
`#covered_by_pension_plan` substituted a (seemingly arbitrary) default
value for `nil`. And the writer of `#bio` went with an `if` statement
switching on the presence of `#hire_date`.

This class has serious problems with second-guessing itself. And the
root of all this insecurity is the fact that the `#hire_date`
attribute is unreliable—even though it's clearly important to the operation of the class!


One of the purposes of a constructor is to establish an object's
invariant: a set of properties which should always hold true for that
object. In this case, it really seems like one of those invariants should be: *employee hire date is a `Date`*.

But the constructor, whose job it is to stand guard against initial
values which are not compatible with the class invariant, has fallen
asleep on the job. As a result, every other method dealing with hire
dates is burdened with the additional responsibility of checking
whether the value is present.

This is an example of a class which needs to set some
boundaries. Since there is no obvious "right" way to handle a missing
hire date, it probably needs to simply insist on having a valid hire
date, thereby forcing the cause of these spurious `nil` values to be
discovered and sorted out. To do this, it should guard its own
integrity by checking the value wherever it is set, either in the
constructor or elsewhere:

```ruby
require 'date'

class Employee
  attr_accessor :name
  attr_reader :hire_date  

  def initialize(name, hire_date)
    @name          = name
    self.hire_date = hire_date
  end

  def hire_date=(new_hire_date)
    raise TypeError, "Invalid hire date" unless new_hire_date.is_a?(Date)
    @hire_date = new_hire_date
  end

  def due_for_tie_pin?
    ((Date.today - hire_date) / 365).to_i >= 10
  end

  def covered_by_pension_plan?
    hire_date.year < 2000
  end

  def bio
    "#{name} has been a Yoyodyne employee since #{hire_date.year}"
  end
end
```

In this version, the `hire_date` attribute is protected by type check
in the setter method. Since the constructor now delegates to this
setter method to initialize the attribute, it is no longer possible to
construct new `Employee` objects without a valid hire date. Now that
the "borders" of the object are guarded, all the other methods can
focus on telling their own stories, without being distracted by
a potentially missing `hire_date`.

### Be assertive

In the last section we saw how assertions in a class' constructor or
setter methods can help keep the other methods focused. But
uncertainty and convoluted code can come from sources
other than input parameters.

Let's say you're working on some budget management software. The next
user story requires the application to pull in transaction data from a
third-party electronic banking API. According to the meager
documentation you can find, you need to use the
`Bank#read_transactions` method in order to load bank
transactions. The first thing you decide to do is to stash the loaded
transactions into a local data store.

```ruby
class Account
  def refresh_transactions
    transactions = bank.read_transactions(account_number)
    # ... now what?
  end
end
```

Unfortunately the documentation doesn't say what the
`#read_transactions` method returns. An `Array` seems likely. But what
if there are no transactions found? What if the account is not found?
Will it raise an exception, or perhaps return `nil`? Given enough time
you might be able to work it out by reading the API library's source
code, but it's pretty convoluted and you might still miss some edge
cases.

You decide to make an assumption… but as insurance, you document your assumption with an assertion.

```ruby
class Account
  def refresh_transactions
    transactions = bank.read_transactions(account_number)
    transactions.is_a?(Array) or raise TypeError, "transactions is not an Array"
    transactions.each do |transaction|
      # ...
    end
  end
end
```

You manually test the code against a test account and it doesn't blow
up, so it seems your suspicion was correct. Next, you move on to
pulling amount information out of the individual transactions.

You ask your teammate, who has had some experience with this API, what
format transactions are in. She says she thinks they are hashes with
string keys. You decide to tentatively try looking at the "amount"
key.

```ruby
transactions.each do |transaction|
  amount = transaction["amount"]
end
```

You look at this for a few seconds, and realize that if there is no
"amount" key, you'll just get a `nil` back. Then you'd have to check
for the presence of `nil` everywhere the amount is used. You'd prefer
to document your assumption more explicitly. So instead, you make an
assertion by using the `Hash#fetch` method:

```ruby
transactions.each do |transaction|
  amount = transaction.fetch("amount")
end
```

`Hash#fetch` will raise a `KeyError` if the given key is not found,
signaling that one of your assumptions about the `Bank API` was
incorrect.

You make another trial run and you don't get any exceptions, so you
proceed onward. Before you can store the value locally, you want to
make sure the transaction amount is in a format that your local
transaction store can understand. Nobody in the office seems to know
what format the amounts come in as. You know that many financial
system store dollar amounts as an integer number of cents, so you
decide to proceed with the assumption that it's the same with this
system. In order to once again document your assumption, you make
another assertion:

```ruby
transactions.each do |transaction|
  amount = transaction["amount"]
  amount.is_a?(Integer) or raise TypeError, "amount not an Integer"
end
```

You put the code through it's paces and… BOOM. You get an error.

```
TypeError: amount not an Integer
```

You decide to drop into the debugger on the next round, and take a
look at the transaction values coming back from the API. You see this:

```ruby
[
 {"amount" => "1.23"},
 {"amount" => "4.75"},
 {"amount" => "8.97"}
]
```

Well that's… interesting. The amounts are reported as decimal strings.

You decide to convert them to integers, since that's what
your internal `Transaction` class uses.

```ruby
transactions.each do |transaction|
  amount = transaction.fetch("amount")
  amount_cents = (amount.to_f * 100).to_i
  # ...
end
```

Once again, you find yourself questioning this code as soon as you
write it. You remember something about `#to_f` being really forgiving
in how it parses numbers. A little experimentation proves this to be
true.

```ruby
"1.23".to_f                     # => 1.23
"$1.23".to_f                    # => 0.0
"a hojillion".to_f              # => 0.0
```

Only having a small sample of demonstration values to go on, you're
not confident that the amounts this API might return will always be in
a format that `#to_f` understands. What about negative numbers? Will
they be formatted as "-4.56"? Or as "(4.56)"? Having an unrecognized
amount format silently converted to zero could lead to nasty bugs down
the road.

Yet again, you want a way to state in no uncertain terms what kind of
values the code is prepared to deal with. This time, you use
Kernel#Float to assert that the amount is in a format Ruby can parse
unambiguously as a floating point number:

```ruby
transactions.each do |transaction|
  amount = transaction.fetch("amount")
  amount_cents = (Float(amount) * 100).to_i
  cache_transaction(:amount => amount_cents)
end
```

`Kernel#Float` is much stricter than `String#to_f`:

```ruby
Float("$1.23")
# ~> -:1:in `Float': invalid value for Float(): "$1.23" (ArgumentError)
# ~>    from -:1:in `<main>'
```

The final code is chock full of assertions:

```ruby
class Account
  def refresh_transactions
    transactions = bank.read_transactions(account_number)
    transactions.is_a?(Array) or raise TypeError, "transactions is not an Array"
    transactions.each do |transaction|
      amount = transaction.fetch("amount")
      amount_cents = (Float(amount) * 100).to_i
      cache_transaction(:amount => amount_cents)
    end
  end
end
```

This code clearly states what it expects. It communicates a great deal
of information about your understanding of the external API at the
time you wrote it. It explicitly establishes the parameters within
which it can operate confidently; as soon as any of its expectations
are violated it fails quickly, with a meaningful exception message.

Unfortunately, as a result it is really telling two stories: one about
refreshing transactions (remember, that's nominally what this method is
about) and one about the format of an external data source. This is
quickly mended, however:

```ruby
class Account
  def refresh_transactions
    fetch_transactions do |transaction_attributes|
      cache_transaction(transaction_attributes)
    end
  end

  # Yields a hash of cleaned-up transaction attributes for each transaction
  def fetch_transactions
    transactions = bank.read_transactions(account_number)
    transactions.is_a?(Array) or raise TypeError, "transactions is not an Array"
    transactions.each do |transaction|
      amount = transaction.fetch("amount")
      amount_cents = (Float(amount) * 100).to_i
      yield(:amount => amount_cents)
    end
  end
end
```

By failing early rather than allowing misunderstood inputs to
contaminate the system, it reduces the need for type-checking and
coercion in other methods. And not only does this code document your
assumptions now, it also sets up an early-warning system should the
third-party API ever change unexpectedly in the future.

### Represent special cases with objects

The most common causes of code that tells a confusing story are
special cases. Let's look at an example of a special case, in the
context of our budgeting application.

You've implemented transaction import and it's working great. Except
for one little problem: users have been reporting bugs about the
reported balances being off. And not just the balances; in fact, all
of the reports seem to have incorrect numbers for some accounts.

You do some investigation into the system logs, and eventually
discover the culprit. It turns out that some banks, when they receive
a authorization for a credit card charge, immediately report it as a
pending transaction in the transaction list. The data looks something
like this:

```ruby
{"amount" => "55.08", "type" => "pending", "id" => "98765"}
```

Then, when the charge is completed or "captured", another transaction
is recorded:

```ruby
{"amount" => "55.08", "type" => "charge", "id" => "98765"}
```

> **Aside:** if you've ever written code to deal with actual banking APIs,
> you've probably figured out by now that I have not. I'm making this up for the
> sake of example. I expect real banking APIs are just as idiosyncratic, though,
> in their own ways.

The result of these "double entries" is that your calculations get
thrown off. Your application has routines for summing transactions,
averaging them, breaking them down by month and quarter, and many
more. And every one of these calculations uses the amount field to
arrive at its results.

You briefly consider simply throwing out pending transactions. But
after a quick consultation with your team you realize this would only
introduce more problems. There is sanity-checking code in place which
checks that the bank servers and the local cache have the same
transaction count and contain the same transaction IDs. And not only
that, you might actually want to use the pending transaction
information for upcoming features.

Your second option is to handle the special case… *specially*,
everywhere that the amount field is referenced. For example:

```ruby
def account_balance
  cached_transactions.reduce(starting_balance) do |balance, transaction|
    if transaction.type == "pending"
      balance
    else
      balance + transaction.amount
    end
  end
end
```

You'll have to carefully audit the code base, adding conditionals to
every use of amount. Not only that, you'll have to make sure anyone
else who works on this code understands the special case.

That doesn't seem like a very attractive option. Thankfully, there is
a third way. And the code above actually gives you the hint you needed
to discover it.

Let's look at that conditional again:

```ruby
if transaction.type == "pending"
```

This code is branching on the type of a value. This is a huge clue. In
an object-oriented language, anytime we branch on an object's type,
we're doing work that the language could be doing for us.

You realize that this special case calls for a special type of object
to represent it.

You decide to try this approach out. You find the code where
transaction objects are being instantiated:

```ruby
# Note: we expect that transaction attributes have already been
# converted to use Symbol keys at this point.
def cache_transaction(attributes)
  cached_transactions << Transaction.new(attributes)
end
```

You change it to instantiate a different kind of object for pending
transactions. Because you want to quickly spike this approach, you use
an `OpenStruct` to create a rough-and-ready ad-hoc object:

```ruby
def cache_transaction(attributes)
  transaction = 
    case attributes[:type]
    when "pending"
      pending_attributes = {
        :amount         => 0,
        :pending_amount => attributes[:amount]
      }
      OpenStruct.new(attributes.merge(pending_attributes))
    else
      Transaction.new(attributes)
    end
  cached_transactions << transaction
end
```

This switches on type as well, but it only does it once. After that,
the transaction can be used as-is in all of your existing algorithms.

You run some tests, and discover this fixes the problem! You consider
simply leaving the code as it is, since it's working now. But on
reflection you decide that the concept of a pending transaction would
be best represented by a proper class. That way you have a place to
put documentation about this special case, as well as any more special
logic you realize you need down the road.

```ruby
# A pending credit-card transaction
class PendingTransaction
  attr_reader :id, :pending_amount

  def initialize(attributes)
    @id             = attributes.fetch(:id)
    @pending_amount = attributes.fetch(:amount)
  end

  def amount
    # Pending transactions duplicate finished transactions, thus
    # throwing off calculations. For the purpose of calculations and
    # reports, a pending transaction always has a zero amount. The
    # real amount is available from #pending_amount.
    0
  end
end
```

You then rewrite `#cached_transactions` to use this new class.

```ruby
def cache_transaction(attributes)
  transaction = 
    case attributes[:type]
    when "pending"
      PendingTransaction.new(attributes)
    else
      Transaction.new(attributes)
    end
  cached_transactions << transaction
end
```

This code solves the immediate problem of a special type of
transaction, without duplicating logic for that special case all
throughout the codebase. But not only that, it is *exemplary*: it sets
a good example for code that follows. When, inevitably, another
special case transaction type turns up, whoever is tasked with dealing
with it will see this class and be guided towards representing the new
case as a distinct type of object.

### Conclusion

Ruby is a language which values expressiveness over just about
everything else. It is optimized to help us programmers say exactly
what we mean, without any extraneous fluff, to both the computer and
to future readers of our code. This is what makes it so much fun to
code in.

When we allow our methods to become cluttered up with ifs and maybes
and provisos and digressions, we let go of that expressiveness. We
start to lose the clear, confident narrative voice. We force the
future maintainers of the code to navigate through a twisty path full
of logical forks in the road in order to understand the purpose of a
method. Reading and updating the code stops being fun.

My challenge to you is this: when you are writing a new method, keep a
clear idea in mind of the story you are trying to tell. When detours
and diversions start to show up along the way, figure out what you
need to do to restore the narrative, and do it. You might get rid of
repetitive data integrity checks by introducing preconditions in the
initializer of a method. Maybe you can surround an external API in
assertions that document your beliefs about it, rather than trying to
handle anything it throws at you. Or perhaps you can eliminate a
family of often-repeated conditionals by representing a special case
as a class in its own right.

However you do it, keep your focus on telling a straightforward
tale. Not only will the future readers of your code thank you for it,
but I think you'll find that it makes your code more robust and easier
to maintain as well.
