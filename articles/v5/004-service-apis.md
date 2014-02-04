*This article was contributed by Carol Nichols
([@carols10cents](http://twitter.com/carols10cents),
[carols10cents@rstat.us](https://rstat.us/users/Carols10cents)), one
of the active maintainers of [rstat.us](https://rstat.us). Carol is
also involved in the Pittsburgh Ruby community, and is a co-organizer of the
[Steel City Ruby Conf](http://steelcityrubyconf.org/).*

[Rstat.us](https://rstat.us) is a microblogging site that is similar to Twitter, but 
based on the [OStatus](http://ostatus.org/about) open standard. It's designed to be federated so
that anyone can run an instance of rstat.us on their own domain while still being
able to follow people on other domains. Although rstat.us is an active project 
which has a lot to offer its users, the lack of an API has limited its
adoption. In particular, an API would facilitate the development of mobile
clients, which are a key part of what makes microblogging convenient for many people.

Two different types of APIs have been considered for possible implementation
in rstat.us: a hypermedia API using an open microblogging spec and a JSON API that is
compatible with Twitter's API. In this article, we'll compare these two API styles 
in the context of rstat.us, and discuss the decision that the project's
developers have made after weighing out the options.

## Hypermedia API

Hypermedia APIs currently have a reputation for being complicated and hard to
understand, but they're really nothing to be scared of. There are many, many
articles about what hypermedia is or is not, but the general definition that
made hypermedia click for me is that a hypermedia API returns links in its
responses that the client then uses to make its next calls. This means that the
server does not have a set of URLs with parameters documented for you up front;
it has documentation of the controls that you will see within the responses.

The specific hypermedia API type that we are considering for rstat.us is one
that complies with the [Application-Level Profile Semantics (ALPS) microblogging
spec](http://amundsen.com/hypermedia/profiles/). This spec is an experiment
started by Mike Amundsen to explore the advantages and disadvantages of multiple
client and server implementations agreeing only on what particular values for
the XHTML attributes `class`, `id`, `rel`, and `name` signify. The spec does not
contain any URLs, example queries, or example responses.

Here is a subset of the ALPS spec attributes and definitions; these have to do
with the rendering of one status update and its metadata:

- li.message - A representation of a single message
- span.message-text - The text of a message posted by a user
- span.user-text - The user nickname text
- a with rel 'message' - A reference to a message representation

This is one way you could render an update that is compatible with these attributes:

```html
<li class="message">
  <span class="message-text">
    I had a fantastic sandwich at Primanti's for lunch.
  </span>
  <span class="user-text">Carols10cents</span>
  <a rel="message" href="http://rstat.us/12345">(permalink)</a>
</li>
```

And this is another way that is also compatible:

```html
<li class="message even">
  <p>
    <a rel="permalink message" href="http://rstat.us/update?id=12345">
      <span class="user-text">Carols10cents</span> said:
    </a>
    <span class="message-text">
      I had a fantastic sandwich at Primanti's for lunch.
    </span>
  </p>
</li>
```

Notice some of the differences between the two:

- All the elements being siblings vs some nested within each other
- Only having the ALPS attribute values vs having other classes and rels as well
- Only having the ALPS elements vs having the `<p>` element 
between the `<li>` and the rest of the children
- Simple resource-based routing vs. passing the id as a parameter

All of these are perfectly fine! If a client only depends on the values of the
attributes and not the exact structure that's returned, it will be flexible
enough to handle both responses. For example, you can extract the username 
from either fragment using the following CSS selector:

```ruby
require 'nokogiri'

# Create a Nokogiri HTML Document from the first example, the second example 
# could be substituted and the result would be the same
html = <<HERE
  <li class="message">
    <span class="message-text">
      I had a fantastic sandwich at Primanti's for lunch.
    </span>
    <span class="user-text">Carols10cents</span>
    <a rel="message" href="http://rstat.us/12345">(permalink)</a>
  </li>
HERE

doc = Nokogiri::HTML::Document.parse(html)

# Using CSS selectors
username = doc.css("li.message span.user-text").text 
```

With this kind of contract, we can change the representation
of an update by the server from the first format to the second without breaking
client functionality. While we will discuss the tradeoffs involved in using
hypermedia APIs in more detail later, it is worth noting
that structural flexibility is a big part of what makes them attractive
from a design perspective.

## JSON API

JSON APIs are much more common than hypermedia APIs right now. This style of API
typically has a published list of URLs, one for each action a client may want to
take. Each URL also has a number of documented parameters through which a client can
send arguments, and the requests return data in a defined format. This style is 
similar to a Remote Procedure Call (RPC) --
functions are called with arguments, and values are returned, but the work is
done on a remote machine. Because this style matches the way we code locally,
it feels familiar, and that may explain why the technique is so popular.

[Twitter's API](https://dev.twitter.com/docs/api) is currently implemented in
this RPC-like style. There is a lot of documentation about all the URLs
available, what parameters they take, and what the returned data or resulting
state will be. For example, here is how you would get the text of the 3 most
recent tweets made by user @climagic with Twitter's JSON API ([relevant
documentation](https://dev.twitter.com/docs/api/1/get/statuses/home_timeline)):

```ruby
require 'open-uri'
require 'json'

# Make a request to the home_timeline resource with the format json.
# Pass the parameter screen_name with the value climagic and the 
# parameter count with the value 3.

base = "http://api.twitter.com/1/statuses/user_timeline.json"
uri  = URI("#{base}?screen_name=climagic&count=3")

# The response object is a list of tweets, which is documented at
# https://dev.twitter.com/docs/platform-objects/tweets

response = JSON.parse(open(uri).read)

tweets = response.map { |t| t["text"] }
```

Rendering JSON from the server is usually fairly simple as well, and
the simplicity of providing and consuming JSON in many different languages
is another one of the big reasons why JSON APIs are gaining in popularity. Twitter
actually decided to [drop support for XML, RSS, and
Atom](https://dev.twitter.com/docs/api/1.1/overview#JSON_support_only) in
version 1.1 of their API, leaving ONLY support for JSON. [According to
Programmable
Web](http://blog.programmableweb.com/2011/05/25/1-in-5-apis-say-bye-xml/) 20%
of new APIs released in 2011 offered only JSON support.

That said, popularity is neither the best nor the only metric for evaluating
design strategies; costs and benefits of different approaches 
can only be weighed out in the context of a real project. To illustrate that point, we can consider how 
each of these API styles would impact the development of rstat.us.

### Comparing and contrasting the two styles

There are many clients that have been built against Twitter's current API. There
are even some clients that allow you to change the root URL of all the requests
(ex:
[Twidere](https://play.google.com/store/apps/details?id=org.mariotaku.twidere))
If rstat.us implemented the same parameters and response data,
people could use those clients to interact with both Twitter and rstat.us. 
Even if rstat.us doesn't end up having this level of compatibility with
Twitter's API, a close approximation to it would still feel a lot more 
familiar to client developers, which may encourage them to support rstat.us.

But is it really a good idea to be coupled to Twitter's API design? If Twitter changes a
parameter name, or a URL, or the structure of the data returned, rstat.us will
need to implement those changes or risk breaking its Twitter-compatible clients.
Because one of the reasons rstat.us was developed was to reduce this kind of
dependency of Twitter, this is a big price to pay, and hypermedia APIs can help
guard against this kind of brittleness.

In addition to flexibility in representation on both the client and server side,
another advantage of a hypermedia API is that
it uses XHTML as its media type, and we just so happen to already have an XHTML
representation of rstat.us' functionality: the web interface itself! If
you take a look at the source of [http://rstat.us](http://rstat.us), you can see
that the markup for an update contains the attribute values we've been talking
about. We haven't made rstat.us completely compliant with the ALPS spec yet, 
but adding attributes to our existing output [has been fairly
simple](https://github.com/hotsh/rstat.us/commit/4e234556c73426dc16526883661b3feb1e2f7d9f).
By contrast, building out a Twitter-compatible JSON API would mean reimplementing an almost
entirely separate interface to rstat.us that would need to maintain a mapping
between its core functionality and the external behavior of Twitter's API.

But looking at the source of http://rstat.us again, you'll also see a lot of
other information in the source of the page. Most of it isn't needed for the use
of the API, so we're transferring a lot of unnecessary data back and forth. The
JSON responses are very compact in comparison; over time and with scale, this
could make a difference in performance.

I am also concerned that some operations that are straightforward with a
Twitter-style JSON API (such as getting one user's updates given their username)
seem complex when following the ALPS spec. With the JSON API, there is a
predefined URL with the username as a parameter, and the response contains
the user's updates. With the ALPS spec, starting from the root URL (which is the
only predefined URL in an ideal hypermedia API), we would need to do a minimum
of 4 HTTP requests. That would lead to some very tedious client code:

```ruby
require 'nokogiri'
require 'open-uri'

USERNAME = "carols10cents"
BASE_URI = "https://rstat.us/"

def find_a_in(html, params = {})
  raise "no rel specified" unless params[:rel]

  # This XPath is necessary because @rels could have more than one value.
  link = html.xpath(
    ".//a[contains(concat(' ', normalize-space(@rel), ' '), ' #{params[:rel]} ')]"
  ).first
end

def resolve_relative_uri(params = {})
  raise "no relative uri specified" unless params[:relative]
  raise "no base uri specified" unless params[:base]

  (URI(params[:base]) + URI(params[:relative])).to_s
end

def request_html(relative_uri)
  absolute_uri = resolve_relative_uri(
    :relative => relative_uri,
    :base     => BASE_URI
  )
  Nokogiri::HTML::Document.parse(open(absolute_uri).read)
end

# Request the root URL
# HTTP Request #1
root_response = request_html(BASE_URI)

# Find the `a` with `rel=users-search` and follow its `href`
# HTTP Request #2
users_search_path = find_a_in(root_response, :rel => "users-search")["href"]
users_search_response = request_html(users_search_path)

# Fill out the `form` that has `class=users-search`,
# putting the username in the `input` with `name=search`

search_path = users_search_response.css("form.users-search").first["action"]
user_lookup_query = "#{search_path}?search=#{USERNAME}"

# HTTP Request #3
user_lookup_response = request_html(user_lookup_query)

# Find the search result beneath `div#users ul.search li.user` that has
# `span.user-text` equal to the username
search_results = user_lookup_response.css("div#users ul.search li.user")

result = search_results.detect { |sr|
  sr.css("span.user-text").text.match(/^#{USERNAME}$/i)
}

# Follow the `a` with `rel=user` within that search result
# HTTP Request #4
user_path = find_a_in(result, :rel => "user")["href"]
user_response = request_html(user_path)

# Extract the user's updates using the update attributes.
updates = user_response.css("div#messages ul.messages-user li")
puts updates.map { |li| li.css("span.message-text").text.strip }.join("\n")
```

This workflow could be cached so that the next time we try to get a user's
updates, we wouldn't have to make so many requests. The first two
requests for the root page and the user search page are unlikely to change
often, so when we get a new username we can start with the construction
of the `user_lookup_query` with a cached `search_path` value. That way, we would
only need to make the last two requests to look up subsequent users.
However, if the root page or the user search page do change, subsequent 
requests could fail. In that case, we'd need error handling code that clears 
the cache and and starts from the root page again. Unfortunately, doing 
so would make the client code even more complicated.

We could simplify things by extending the ALPS spec to include a URI
template on the root page with a `rel` attribute to indicate that it's a
transition to information about a user when the template is filled out with
the username. The ALPS spec path would still work, but the shortcut would
allow clients to get at this data in fewer requests.
However, since it wouldn't be an official part of the spec, we'd need to
document it, and all clients that wanted to remain compatible with ALPS would
still need to implement the long way of doing things.

As you can see, there are significant tradeoffs between the two API styles,
and so it isn't especially easy to decide what to do. But because rstat.us
really needs an API in order to be a serious alternative to Twitter, we must 
figure out a way forward!

### Making a decision

After weighing all these considerations, we've decided to concentrate first on
implementing a Twitter-compatible JSON API, because it may allow our users
to interact with rstat.us using the clients they are already familiar with. Even
if those clients end up requiring some modifications, having an API that is easily 
understood by many developers will still be a big plus. For the long term, having a more flexible and
scalable solution is important, but those problems won't need to be solved
until there is more adoption. We may implement a hypermedia API (probably an
extension of the ALPS spec) in the future, but for now we will take the
pragmatic route in the hopes that it will encourage others to use
rstat.us and support its development.

### References

- [rstat.us](http://rstat.us) and its [code on github](https://github.com/hotsh/rstat.us)
- [ALPS microblogging spec](http://amundsen.com/hypermedia/profiles/)
- [Designing Hypermedia APIs](http://designinghypermediaapis.com) by Steve Klabnik
- [A Shoes hypermedia client for ALPS microblogging](https://gist.github.com/2187514)
- [Twitter API docs](https://dev.twitter.com/docs/api)
- [REST APIs must be hypertext-driven](http://roy.gbiv.com/untangled/2008/rest-apis-must-be-hypertext-driven)
