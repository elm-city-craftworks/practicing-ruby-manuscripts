> **NOTE:** This issue of Practicing Ruby was one of several content experiments 
that was run in Volume 6. It uses a cookbook format (e.g. problem -> solution -> discussion)
instead of the traditional long-form article format we use in most Practicing Ruby articles.

**Problem: A method has many parameters, making it hard to remember its
interface.**

Suppose we were building a HTTP client library called `HyperClient`. A trivial
request might look like this:

```ruby
http = HyperClient.new("example.com")
http.get("/")
```

But we would probably need to support some other features as well, such as 
accessing HTTP services running on non-standard ports, and routing 
requests through a proxy. If we simply add these features 
without careful design consideration, we may end up
with the following bloated interface for `HyperClient.new`: 

```ruby
http = HyperClient.new("example.com", 1337, 
                       "internal.proxy.example.com", 8080, 
                       "myuser", "mypassword")
```

If the above code looks familiar to you, it's because it is modeled directly
after Ruby's `Net::HTTP` standard library; a codebase which
is often critized for it's poor API design! There are many reasons 
why this style of interface is bad, but three obvious issues stand out:

* Without a single unambiguous way of sorting the parameters, it is very
difficult to remember their order.

* This style of interface makes it hard to set defaults for parameters in a
flexible way. For example, consider the difficulty of setting default values for
the `service_port` and `proxy_port` in the code above.

* If the `HyperClient` API changes and a new optional parameter is introduced, 
it must either be added to the end of the arguments list or risk breaking 
all calls that relied on the previous order of the parameters.

Fortunately, all of the above points can be addressed by designing a better
method interface.

---

**Solution: Use a combination of keyword arguments and parameter objects to
create interfaces that are both memorable and maintainable.**

Whenever a method's interface accumulates several related arguments, it is a
sign that introducing a parameter object might be helpful. In this 
particular example, we can easily group together the proxy-related arguments 
as shown below:

```ruby
proxy = HyperClient::Proxy.new("internal.proxy.example.com",
                               :port     =>  8080,
                               :username => "myuser",
                               :password => "mypass")
```

By switching to keyword arguments, it becomes obvious what
each of these parameters represent, and there is no need to list them
in a particular order. This basic idea can also be extended to simplify 
the interface of the original `HyperClient` object:

```ruby
http = HyperClient.new("example.com", :port  => 1337, :proxy => proxy) 
```

This new constructor looks and feels more comfortable to use, because it
introduces some structure to separate essential parameters from
optional ones while grouping related concepts together. This
makes it easier to recall the right bits of knowledge at the right time.

---

**Discussion**

Both interfaces for `HyperClient.new` handle the most common use case 
in the same way:

```ruby
http = HyperClient.new("example.com")
```

Where they differ is when you have extra parameters. Dealing with
default values in the former is *much* uglier. For example, if
`HyperClient` provided default ports for both the service and the
proxy, you'd need to do something like this when using a username
and password:

```ruby
http = HyperClient.new("example.com", nil, 
                       "internal.proxy.example.com", nil,
                       "myuser", "mypassword")
```                       

In the improved code, those parameters could simply be omitted:

```ruby
proxy = HyperClient::Proxy.new("internal.proxy.example.com",
                               :username => "myuser",
                               :password => "mypass")

http = HyperClient.new("example.com", :proxy => proxy)
```

But this is a consequence of using keyword arguments -- it has 
little to do with the fact that we've introduced the `HyperClient::Proxy` 
parameter object. For example, if the following API were used instead,
it would be trivial to fall back to default values for `:service_port` and
`:proxy_port` if they were not explicitly provided:

```ruby
http = HyperClient.new("google.com",
                       :proxy_address   => "internal.proxy.example.com",
                       :proxy_username  => "myuser",
                       :proxy_password  => "mypass")
```

The following signature supports this kind of behavior, using Ruby 2.0's 
keyword arguments:

```ruby
class HyperClient
  def initialize(service_address, service_port: 80, 
                 proxy_address:  nil, proxy_port: 8080, 
                 proxy_username: nil, proxy_password: nil)

    # ...        
  end
end
``` 

This style of design isn't especially painful to work with for the end-user, 
and it has a fairly wide precedent in Ruby library design. However, taking this
approach comes with three significant drawbacks:

* An interface with many similarly named parameters that are 
differentiated only by a prefix (e.g. `service_port` vs. `proxy_port`)
is still intention-revealing and memorable, but the repetition 
introduces line noise that hurts readability.

* Validating and transforming inputs becomes increasingly complex 
as method interfaces become bloated. Think about the various
checks that would need to be done in the previous example to
verify what proxy settings should be used, if any.

* Each and every new parameter introduced into a method's interface 
creates a new set of branches that need to be covered by tests,
and considered during debugging.

To see how these issues are mitigated by the introduction of the
`HyperClient::Proxy` object, think through what the validation
and transformation work might look like in both the example shown
above, and in the code shown below:

```ruby
class HyperClient
  def initialize(service_address, port: 80, proxy: nil)
    # ...
  end

  class Proxy
    def initialize(address, port: 8080, username: nil, password: nil)
      # ...
    end
  end
end
```

Although the two implementations will end up sharing a lot of code in 
common, introducing a formal parameters object allows you to hide
some of the ugly details from the `HyperClient` class that would
otherwise end up in its constructor. This is good for both testability
and maintainability.

Despite its utility, it is possible to take this technique too far. 
For example, introducing a `HyperClient::Service` object to wrap the service 
address and port is probably more trouble than its worth, because it does not
hide enough complexity to have a net positive impact on maintainability.That said,
design decisions are highly context dependent and need to 
be revisited as requirements grow and change. Suppose that wanted to support
both SSL and HTTP basic authentication were in this library; 
then adding a `HyperClient::Service` object might start to make sense!
This rise in necessary complexity shifts the balance of things to make
an extra layer of indirection seem worthwhile, where it may not have before.

The thing to remember is that being influenced by features that will soon be 
implemented is part of the design process, but considering vague scenarios 
that may or may not happen in the far future is more akin to gazing into a 
crystal ball. The former is productive; the latter is potentially harmful.

---

**Conclusions**

When designing method interfaces, don't bother trying to get them perfect,
because they will eventually end up changing anyway. However, don't just ignore
their design either -- keep in mind that good APIs makes easy things easy and hard 
things possible. The techniques we've discussed in this recipe should help you
avoid some of the most common mistakes people make, but the rest is up to you!

If you want to learn more about method-level interface design, James Noble wrote
a great paper on the topic called [Arguments and
Results](http://www.laputan.org/pub/patterns/noble/noble.pdf). I strongly
recommend reading his work, as well as [Issue 2.14](https://practicingruby.com/articles/shared/vpxpovppchww) 
and [Issue 2.15](https://practicingruby.com/articles/shared/mupuergickjz) of
Practicing Ruby, which cover the same topic with some Ruby-specific examples.
