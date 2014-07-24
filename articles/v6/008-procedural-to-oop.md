> **NOTE:** This issue of Practicing Ruby was one of several content experiments 
that was run in Volume 6. It uses a cookbook format (e.g. problem -> solution -> discussion)
instead of the traditional long-form article format we use in most Practicing Ruby articles.

**Problem: An adhoc script has devolved into an unmaintainable mess**

Imagine that you're working on a shipping cost estimation program for a small
business that uses a courier service for regional deliveries. Part of the task
for building that tool would involve importing pricing information from
some data source, such as this CSV file:

```
06770,$12.00
06512,$14.00
06510,$15.30
06701,$12.15
```

A real dataset would be more complex, but this minimal example exposes the
information we're interested in: what it costs to ship something from our
facility to somewhere else, based on the destination's zip code.

Now suppose that we want to build a simple data store which will be updated
daily with the latest pricing information. We then could easily write a script
using a few of Ruby's standard libraries (`PStore`, `BigDecimal`, and `CSV`),
which would normalize the data in a way that could be used by the user-facing
cost estimation program. If we could assume the source CSV data was validated
before we processed it, the program could be as simple as what you see below:

```ruby
require "csv"
require "pstore"
require "bigdecimal"

store = PStore.new("shipping_rates.store")

store.transaction do
  CSV.foreach(ARGV[0] || "rates.csv") do |r|
    zip    = r[0]
    amount = BigDecimal.new(r[1][1..-1])
    
    store[zip] = amount
  end
end
```

But in reality, most businesses environments do not make things like this easy
for you. You'd probably quickly discover that the source data could have
any number of problems with it, ranging from duplicate entries to inconsistently
formatted fields. Because this kind of data often originates from people who are
entering information into Excel by hand, they can even be littered with typos!

To help mitigate these issues somewhat, you need a combination of
sanity-checking validations and basic logging so that when something goes wrong
you know why it happened. After adding those features, your simple script might
collapse into the mess you see below:

```ruby
require "csv"
require "pstore"
require "bigdecimal"

store = PStore.new("shipping_rates.store")

store.transaction do
  processed_zipcodes  = []
  
  CSV.foreach(ARGV[0] || "rates.csv") do |r|
    raise unless r[0][/\A\d{5}\z/]
    raise unless r[1][/\A\$\d+\.\d{2}\z/]
    
    zip    = r[0]
    amount = BigDecimal.new(r[1][1..-1])

    raise "duplicate entry: #{zip}" if processed_zipcodes.include?(zip)
    processed_zipcodes << zip
    
    next if store[zip] == amount

    if store[zip].nil?
      STDERR.puts("Adding new entry for #{zip}: #{'%.2f' % amount}")
    elsif store[zip] != amount
      STDERR.puts("Updating entry for #{zip}: "+
                  "was #{'%.2f' % store[zip]}, now #{'%.2f' % amount}")
    end
    
    store[zip] = BigDecimal.new(amount)
  end
end
```

Once your code ends up like this, it becomes increasingly difficult to 
add new features or make any sort of change without breaking 
something. Because this style of program is fairly difficult to test,
the maintenance problems can be made even worse by the fact that bugs may 
end up not being discovered until long after they're introduced.

Procedural scripts are great when you can throwaway the code once you've
completed your task, or for solving simple problems that you are reasonably
sure the requirements will never change for. For everything else,
more structure pays off in the long run. It's clear that this program
is in the latter category, so how do we fix it?

---

**Solution: Redesign the script as an object-oriented program**

The thing that makes ad-hoc scripts complicated to reason about
as they grow is that they blend all their concerns together -- both 
logically and conceptually. For that reason, it is worthwhile to
start thinking in terms of functions and objects as soon as your
program exceeds more than a paragraph or two of code.

Imagine that the script portion of your importer tool was reduced
to the following code:

```ruby
require "csv"

Importer.update("shipping_rates.store") do |store|
  CSV.foreach(ARGV[0] || "rates.csv") do |r|
    info = PriceInformation.new(zipcode: r[0], shipping_rate: r[1])
    
    store[info.zipcode] = info.shipping_rate
  end
end
```

This brings us back to about the same level of detail expressed in the
naïve implementation of the importer script, albeit with a few custom classes
thrown into the mix. It hides a lot of detail
from the reader, but its core purpose is obvious: it iterates over a CSV file
to create a mapping of zipcodes to shipping rates in a datastore. 

To see where the real work is being done, we need to look at the
`PriceInformation` and `Importer` class definitions. We'll start by taking a
look at the former, because it has fewer moving parts to consider:

```ruby
require "bigdecimal"

class PriceInformation
  ZIPCODE_MATCHER = /\A\d{5}\z/
  PRICE_MATCHER   = /\A\$\d+\.\d{2}\z/

  def initialize(zipcode: raise, shipping_rate: raise)
    raise "Zipcode validation failed"       unless zipcode[ZIPCODE_MATCHER]
    raise "Shipping rate validation failed" unless shipping_rate[PRICE_MATCHER]
    
    @zipcode       = zipcode 
    @shipping_rate = BigDecimal.new(shipping_rate[1..-1])
  end

  attr_reader :zipcode, :shipping_rate
end
```

Here we see that `PriceInformation` applies the same validations and
transformations as shown in the script version of this program, but
encapsulates them in its constructor. This makes sure that a `PriceInformation`
object will either represent valid data or not be instantiated at all, 
which makes it so that the main script does not need to concern itself 
with these issues. Even if these validations or transformations become
more complex over time, the calling code should not need to change.

In a similar vein, the `Importer` class attempts to encapsulate the details
about some lower level concepts at a higher level of abstraction. It's
functionality is a bit more involved than the `PriceInformation` class,
so take a few minutes to study it before moving on:

```ruby
require "pstore"

class Importer
  def self.update(filename)
    store = PStore.new(filename)

    store.transaction do
      yield new(store)
    end
  end

  def initialize(store)
    self.store    = store
    self.imported = []
  end

  def []=(key, new_value)
    raise_if_duplicate(key)

    old_value = store[key]

    return if old_value == new_value # nothing to do!

    if old_value.nil?
      ChangeLog.new_record(key, new_value)
    else
      ChangeLog.updated_record(key, old_value, new_value)
    end

    store[key] = new_value
  end

  private

  attr_accessor :store, :imported

  def raise_if_duplicate(key)
    raise "Duplicate key in import data: #{key}" if imported.include?(key)
    imported << key
  end
end
```

Despite the complexity of its implementation, this class presents a very minimal
user interface, consisting of only `Importer.update` and `Importer#[]=`. The
`Importer.update` method is responsible for instantiating a `PStore` object,
initiating a transaction, and then wrapping it in an `Importer` instance to
limit access to its internals. From there, the only method available to the user
is `Importer#[]=`, which wraps `PStore#[]=` with two important features:

1. Single-assignment semantics: once a key has been set to particular value, it
cannot be reset from within the same `Importer` instance. This is because we
want to raise an exception whenever we encounter duplicate keys in the data
we're importing.

2. Update notifications: For debugging purposes, we want to know whether a
record is introducing a new key, or updating the value associated with
an old one. Rather than cluttering up this class with the particular log
messages associated with those events, we delegate to a `ChangeLog` helper
object, which is shown below:

```ruby
class << (ChangeLog = Object.new)
  def new_record(key, value)
    STDERR.puts "Adding #{key}: #{f(value)}"
  end

  def updated_record(key, old_value, new_value)
    STDERR.puts "Updating #{key}: Was #{f(old_value)}, Now #{f(new_value)}" 
  end

  private

  def f(value)
    '%.2f' % value
  end
end
```

With this last detail exposed, you've walked through the complete 
object-oriented solution to this problem. It is much longer than the
script version, but also much more organized. Before we wrap things up, 
let's talk a bit more about the costs and benefits involved in introducing
more structure into your programs.

---

**Discussion**

The best thing about unstructured code is that nothing is hidden from view. 
To understand a script, you start at the top of the file and read downwards, 
mentally evaluating the state changes and iterators you encounter along the way.

Object-oriented programs are much more logically complex, because they 
represent a network of collaborators rather than a linear set of instructions.
For example, whenever we make a call to `Importer#[]=`, messages are sent to the
`ChangeLog` helper object as well as to an instance of `PStore`, but these
details are not at all visible when you read the caller code. The more objects
that exist within a system, the more complex their interactions get, and so
it is not uncommon to end up with call graphs that are both wide and deep.

But when it comes to visibility, the strength of scripted solutions is also their 
weakness, and the weakness of object-oriented programs is also their strength:

* In an adhoc script, you cannot make simple decisions about your code
without considering the entire program. Even something as straightforward
as renaming a variable used for temporary storage must be carefully considered,
because everything exists within a single namespace; anything more involved
than that is simply inviting trouble unless you can keep the entire program
in your head at once.

* In an object-oriented program, the walls erected between different objects give
you freedom to make sweeping changes to internal structures, as long as their
interfaces are preserved. You can even rewire entire subnetworks of functionality
from your programs, as long as you know what features depend on them. When
done well, the fact that you cannot keep an entire object-oriented program
in your head is not much of a concern, because the layered abstractions
make it so you don't have to.

The real challenge involved in writing object-oriented programs is that they'll
only be as useful as the mental model they represent. This is why it can
actually be helpful to start off with less structure (even none at all!), and
gradually work your way towards something more organized. After all,
there is nothing worse than an abstract solution in search of a concrete problem!
