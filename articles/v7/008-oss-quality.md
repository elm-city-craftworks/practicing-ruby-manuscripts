> This article was written in collaboration with Eric Hodel
> ([@drbrain](http://twitter.com/drbrain)), a developer from Seattle. 
> Eric is a Ruby core team member, and he also maintains RubyGems 
> and RDoc. 

A big challenge in managing open source projects is that their codebases tend
to decay as they grow. This isn't due to a lack of technically skilled
contributors, but instead is a result of the gradual loss of understandability 
that comes along with any long-term and open-ended project 
that has an distributed team of volunteers supporting it.

Once a project becomes more useful, it naturally attracts a more
diverse group of developers who are interested in adapting the codebase to
meet their own needs. Patches are submitted by contributors who do not fully 
understand a project's implementation, and maintainers merge these patches 
without fully understanding the needs of their contributors. Maintainers 
may also struggle to remember the reasoning behind any of their own code that they 
haven't touched in a while, but they still need to be able to work with it.

As a result of both of these influencing factors, mistaken assumptions tend to 
proliferate as a project grows, and with them come bugs and undefined behaviors. 
When direct knowledge of the codebase becomes limited and unreliable, it's easy to 
let code quality standards slip without fully realizing the potential
for future problems. 

If bad code continues to accumulate in this fashion, improving one part of a 
a project usually means breaking something else in the 
process. Once a maintainer starts spending most of their time fixing bugs, 
it gets hard to move their project forward in meaningful 
ways. This is where open source development stops being fun, and starts feeling 
like a painful chore.

Not all projects need to end up this way, though. As long as project maintainers 
make sure to keep the quality arrow pointing upwards over the long haul, 
any bad code that temporarily accumulates in a project can always be replaced with 
better code whenever things start getting painful. The real challenge is to 
establish healthy maintenance practices that address quality issues 
in a consistent and sustainable way. 

### Developing a process-oriented approach towards quality

In this article, we'll discuss three specific tactics we've used in 
our own projects that can be applied at any stage in the software 
development lifecycle. These are not quick fixes; they are helpful 
habits that drive up understandability and code quality more and more
as you continue to practice them. The good news is that even though 
it might be challenging to keep up with these efforts on a daily basis, 
the recommendations themselves are very simple:

1. Let external changes drive incremental quality improvements
2. Treat all code with inadequate testing as legacy code
3. Expand functionality via well-defined extension points 

We'll now take a look at each of these guidelines individually and walk 
you through some examples of how we've put them into practice in RDoc, 
RubyGems, and Prawn -- three projects that have had their own share of 
quality issues over the years, but continue to serve very diverse 
communities of users and contributors.

### 1) Let external changes drive incremental quality improvements

Although there is often an endless amount of cleanup work that can
be done in mature software projects, there is rarely enough
available development time to invest in these efforts. For programmers 
working on open source in their spare time, it is hard enough
to keep up with new incoming requests, so most preventative maintenance 
work ends up being deferred indefinitely. When cleanup efforts do happen,
they tend to be done in concentrated bursts and then things go back
to business-as-usual from there.

A better approach is to pay down technical debts little by little, not as a
distinct activity but as part of responding to ordinary change requests. There
are only two rules to remember when applying this technique in your daily work:

* Try to avoid making the codebase worse with each new change, or at least
minimize new maintenance costs as much as possible.
* If there is an easy way to improve the code while doing everyday work, 
go ahead and invest a little bit of effort now to make future changes easier. 

The amount of energy spent on meeting these two guidelines should be proportional
to the perceived risks and rewards of the change request itself, but typically
it doesn't take a lot of extra effort. It may mean spending an extra 10 minutes on a
patch that would take an hour to develop, or an extra hour on a patch that would
take a day to prepare. In any case, it should feel like an obviously good
investment that is well worth the cost you are paying for it.

There is a great example in Prawn that illustrates this technique being used,
and if you want to see it in its raw form, you can check out [this pull
request](https://github.com/prawnpdf/prawn/pull/587) from Matt Patterson.

Matt's request was to change the way that Prawn's image loading
feature detected whether it was working with an I/O object or a path to
a file on disk. Initially Prawn assumed that any object responding to `read` 
would be treated as an I/O object, but this was too loose of a test and
caused some subtle failures when working with `Pathname` objects.

The technical details of the change are not important here, so don't worry if
you don't understand them. Instead, just look at the method that would need to
be altered to fix this problem, and ask yourself whether you would feel
comfortable making a change to it:

```ruby
def build_image_object(file)
  file.rewind  if file.respond_to?(:rewind)
  file.binmode if file.respond_to?(:binmode)

  if file.respond_to?(:read)
    image_content = file.read
  else
    raise ArgumentError, "#{file} not found" unless File.file?(file)  
    image_content = File.binread(file)
  end
  
  image_sha1 = Digest::SHA1.hexdigest(image_content)

  if image_registry[image_sha1]
    info = image_registry[image_sha1][:info]
    image_obj = image_registry[image_sha1][:obj]
  else
    info = Prawn.image_handler.find(image_content).new(image_content)

    min_version(info.min_pdf_version) if info.respond_to?(:min_pdf_version)

    image_obj = info.build_pdf_object(self)
    image_registry[image_sha1] = {:obj => image_obj, :info => info}
  end

  [image_obj, info]
end
```

Although this probably isn't the absolute worst code you have ever seen, 
it isn't very easy to read. Because it takes on many responsibilities,
it's hard to even summarize what it is supposed to do! Fortunately for Matt,
the part that he would need to change was only the first few lines of the 
method, which are reasonably easy to group together:

```ruby
def build_image_object(file)
  file.rewind  if file.respond_to?(:rewind)
  file.binmode if file.respond_to?(:binmode)

  if file.respond_to?(:read)
    image_content = file.read
  else
    raise ArgumentError, "#{file} not found" unless File.file?(file)  
    image_content = File.binread(file)
  end

   # ... everything else 
end
```

The quick fix would have been to edit these lines directly, but Matt recognized
the opportunity to isolate a bit of related functionality and make the code a
little bit better in the process of doing so. Pushing these lines of code down
into a helper method and tweaking them slightly resulted in the following
cleanup to the `build_image_object` method:

```ruby
def build_image_object(file)
  io = verify_and_open_image(file)
  image_content = io.read

  # ... everything else
end
```

In the newly created helper method, Matt introduced his desired change, 
which is much easier to understand in isolation than it would have been in the 
original `build_image_object` method definition. In particular, he changed
the duck typing test to look for `rewind` rather than `read`, in the hopes
that it would be a more reliable way to detect I/O-like objects. Everything 
else would be wrapped in a `Pathname` instance:

```ruby
def verify_and_open_image(io_or_path)
  if io_or_path.respond_to?(:rewind)
    io = io_or_path

    io.rewind

    io.binmode if io.respond_to?(:binmode)
    return io
  end

  io_or_path = Pathname.new(io_or_path)
  raise ArgumentError, "#{io_or_path} not found" unless io_or_path.file?
  
  io_or_path.open('rb')
end
```

At this point, he could have submitted a pull request, because the tests were
still green and the new behavior was working as expected. However, the issue
he had set out to fix in the first place wasn't causing Prawn's tests to fail,
and that was a sign that there was some undefined behavior at the root of
this problem. Although Prawn had some tests for reading images referenced by
`Pathname` objects, it only had done its checks at a high level, and did not
verify that the PDF output was being rendered correctly.

A test would be needed at the lower level to verify that the output was no
longer corrupted, but this kind of testing is slightly tedious to do in Prawn.
Noticing this rough spot, Matt created an RSpec matcher to make this kind of
testing easier to do in the future:

```ruby
RSpec::Matchers.define :have_parseable_xobjects do
  match do |actual|
    expect { PDF::Inspector::XObject.analyze(actual.render) }.not_to raise_error
    true
  end
  failure_message_for_should do |actual|
    "expected that #{actual}'s XObjects could be successfully parsed"
  end
end
```

Finally, he provided a few test cases to demonstrate that his patch
fixed the problem he was interested in, and also covered some other 
common use cases as well:

```ruby
context "setting the length of the bytestream" do
  it "should correctly work with images from Pathname objects" do
    info = @pdf.image(Pathname.new(@filename))
    expect(@pdf).to have_parseable_xobjects
  end

  it "should correctly work with images from IO objects" do
    info = @pdf.image(File.open(@filename, 'rb'))
    expect(@pdf).to have_parseable_xobjects
  end

  it "should correctly work with images from IO objects not set to mode rb" do
    info = @pdf.image(File.open(@filename, 'r'))
    expect(@pdf).to have_parseable_xobjects
  end
end
```

When you put all of these changes together, the total value of this patch
is much greater than the somewhat obscure bug it fixed. By addressing
some minor pain points as he worked, Matt also improved Prawn in the
following ways:

* The `build_image_object` method is now more understandable because one 
of its responsibilities has been broken out into its own method.

* The `verify_and_open_image` method allows us to group together all the
basic guard clauses for determining how to read the image data, 
making it easier to see exactly what those rules are.

* The added tests clarify the intended behavior of Prawn's image loading
mechanism.

* The newly added RSpec matcher will help us to do more
PDF-level checks in future tests.

None of these changes required a specific and focused effort of refactoring or redesign,
it just involved a bit of attention to detail and a willingness to make minor
improvements that would pay off for someone else in the future.

As a project maintainer, you cannot expect contributors to put this level of
effort into their patches -- Matt really went above and beyond here. However, 
you can definitely look for these kind of opportunities yourself during review 
time, and either ask the contributor to make some revisions, or make them yourself 
before you merge in new changes. No matter who ends up doing the work, little by
little these kinds of incremental cleanup efforts can turn a rough codebase into
something pleasant to work with.

### 2) Treat all code without adequate testing as legacy code

Historically, we've defined legacy code as code that was written long before our
time, without any consideration for our current needs. However, any untested
code can also be considered legacy code[^1], because it often
has many of the same characteristics that make outdated systems difficult to
work with. Open source projects evolve quickly, and even very clean code
can cause a lot of headaches if its intended behavior is left undefined.

To guard against the negative impacts of legacy code, it helps 
to continuously update your project's test suite so 
that it constantly reflects your current understanding of the problem domain
you are working in. A good starting point is to make sure that your project
has good code coverage and that you keep your builds green in CI.
Once you've done that, the next step is to go beyond the idea of just having
lots of tests and start focusing on making your test suite more capable 
of catching problems before they leak out into released code.

Here are some things to keep in mind when considering the potential 
impact that new changes will have on your project's stability:

* Any behavior change introduced without test
coverage has a good chance of causing a defect or 
accidentally breaking backwards-compatibility in a future release.

* A passing test suite is not proof that a change is well-defined
and defect-free.

* The only reliable way to verify that existing features have 
well-defined behavior and good test coverage is to
review their code manually.

* Contributors often don't understand your project's problem domain 
or its codebase well enough to know how to write good tests for
their changes without some guidance.

These points are not meant to imply that each and every pull request
ought to be gone over with a fine-tooth comb -- they're only meant to 
serve as a reminder that maintaining a high quality test suite is
a harder problem than we often make it out to be. The same ideas
of favoring incremental improvements over heroic efforts that
we discussed earlier also apply here. There is no need to
rush towards a perfect test suite all at once, as long as it improves
on average over time.

We'll now look at a [pull request](https://github.com/rubygems/rubygems/pull/781/files) 
that Brian Fletcher submitted to
RubyGems for a good example of how these ideas can be applied 
in practice.

Brian's request was to add support for Base64 encoded usernames and
passwords in gem request URLs. Because RubyGems already supported
the use of HTTP Basic Auth with unencoded usernames and passwords in
URLs, this was an easy change to make. The desired URL decoding functionality
was already implemented by `Gem::UriFormatter`, so the
initial commit for this pull request involved changing just a single line 
of code:

```diff
     request = @request_class.new @uri.request_uri
 
     unless @uri.nil? || @uri.user.nil? || @uri.user.empty? then
-      request.basic_auth @uri.user, @uri.password
+      request.basic_auth Gem::UriFormatter.new(@uri.user).unescape,
+                         Gem::UriFormatter.new(@uri.password).unescape
     end
 
     request.add_field 'User-Agent', @user_agent
```

On the surface, this looks like a fairly safe change to make. Because it only
adds support for a new edge case, it should preserve the original behavior
for URLs that did not need to be unescaped. No new test failures were introduced
by this patch, and a quick look at the test suite shows that `Gem::UriFormatter`
has some tests covering its behavior.

As far as changes go, this one is definitely low risk. But if you dig in a
little bit deeper, you can find a few things to worry about: 

* Even though only a single line of code was changed, that line of code
was at the beginning of a method that is almost 90 lines long. This isn't
necessarily a problem, but it should at least be a warning sign to slow
down and take a closer look at things.

* A quick look at the test suite reveals that although there were tests
for the `unescape` method provided by `GemUri::Formatter`, there were no tests 
for the use of Basic Auth in gem request URLs, which means the behavior
this patch was modifying was not formally defined. Because of this, we can't
be sure that a subtle incompatibility wasn't introduced by this patch, 
and we wouldn't know if one was introduced later due to a change to
`GemUri::Formatter`, either.

* The new behavior introduced by this patch also wasn't verified, which
means that it could have possibly been accidentally removed in a future 
refactoring or feature patch. Another contributor could easily assume 
that URL decoding was incidental rather than intentional without
tests that indicated otherwise.

These are the kind of problems that a detailed review can discover 
which are often invisible at the surface level. However, a much more
efficient maintenance policy is to simply assume one or more of the 
above problems exist whenever a change is introduced without tests, 
and then either add tests yourself or ask contributors to add them
before merging. 

In this case, Eric asked Brian to add a test after giving him some guidance 
on how to go about implementing it. For reference, this was his exact request:

> Can you add a test for this to test/rubygems/test_gem_request.rb?
>
> You should be able to examine the request object through the block #fetch yields to.

In response, Brian dug in and noticed that the base case of
using HTTP Basic Auth wasn't covered by the tests. So rather than simply 
adding a test for the new behavior he added, he went ahead and wrote tests 
for both cases:

```ruby
class TestGemRequest < Gem::TestCase
 def test_fetch_basic_auth
    uri = URI.parse "https://user:pass@example.rubygems/specs." +
                     Gem.marshal_version
    @request = Gem::Request.new(uri, Net::HTTP::Get, nil, nil)
    conn = util_stub_connection_for :body => :junk, :code => 200

    response = @request.fetch

    auth_header = conn.payload['Authorization']

    assert_equal "Basic #{Base64.encode64('user:pass')}".strip, auth_header
  end

  def test_fetch_basic_auth_encoded
    uri = URI.parse "https://user:%7BDEScede%7Dpass@example.rubygems/specs." +
                    Gem.marshal_version
    @request = Gem::Request.new(uri, Net::HTTP::Get, nil, nil)
    conn = util_stub_connection_for :body => :junk, :code => 200

    response = @request.fetch

    auth_header = conn.payload['Authorization']

    assert_equal "Basic #{Base64.encode64('user:{DEScede}pass')}".strip, 
                 auth_header
  end
end
```

It is hard to overstate the difference between a patch with these tests
added to it and one without tests. The original commit introduced a new
dependency and more complex logic into a feature that lacked formal definition
of its behavior. But as soon as these tests are added to the change request,
RubyGems gains support for a new special condition in gem request URLs while 
tightening up the definition of the original behavior. The tests
also serve to protect both conditions from breaking without being noticed 
in the future.

Taken individually, the risks of accepting untested patches are
small enough that they don't seem important enough to worry about when you are pressed
for time. But in the aggregate, the effects of untested code will pile up until your
codebase really does become unworkable legacy code. For that reason, establishing
good habits about reviewing and shoring up tests on each new change can make a
huge difference in long-term maintainability.

### 3) Expand functionality via well-defined extension points 

Most open source projects can benefit from having two clearly defined interfaces:
one for end-users, and one for developers who want to extend its functionality.
This point may seem tangentially related to code quality and maintainability,
but a well-defined extension API can greatly increase a project's stability.

When its possible to add new functionality to a project without patching its
codebase directly, it becomes easier to separate essential features that most
people will need from features that are only relevant in certain rare 
contexts. The ability to support external add-ons in a transparent way also
makes it possible to try experiments outside of your main codebase and then
only merge in features that prove to be both stable and widely used.

Even within the scope of a single codebase, explicitly defining a layer one
level beneath the surface forces you to think about what the common points
of interaction are between your project's features. It also makes testing
easier, because feature implementations tend to get slimmed down as the
extension API becomes more capable. Each part can then be tested in 
isolation without having to think about large amorphous blobs
of internal dependencies.

It may be hard to figure out how to create an extension API when you first start
working on a project, because at that time you probably don't know much
about the ways that people will need to extend its core behavior, and you may
not even have a good sense of what its core feature set should be! This is
completely acceptable, and it makes sense to focus exclusively on your high-level 
interface at first. But as your project matures, you can use the following guidelines to
incrementally bring a suitable extension API into existence:

* With each new feature request, ask yourself whether it could be implemented
as an external add-on without patching your project's codebase. If not, figure 
out what extension points would make it possible to do so.

* For any of your features that have become difficult to work with or overly
complex, think about what extension points would need to be added in
order to extract those features into external add-ons.

* For any essential features that have clearly related functionality, 
figure out what it would take to re-implement them on top of well defined 
extension points rather than relying on lots of private internal code.

At first, you may start by carrying out these design considerations as simple
thought experiments that will indirectly influence the way you implement 
things. Later, you can take them more seriously and seek to support
new functionality via external add-ons rather than merging new features 
unless there is a very good reason to do otherwise. Every project needs to
discover the right balance for itself, but the basic idea is that the value of a
clear extension API increases the longer a project is in active use.

Because RDoc has been around for a very long time and has a fairly decent extension 
API, it is a good library to look at for examples of what this technique has
to offer. Without asking Eric for help, I looked into what it would take to autolink 
Github issues, commits, and  version tags in RDoc output. This isn't something I had 
a practical use for, but I figured it  would be a decent way to test how easily I 
could extend the RDoc parser.

I started with the following text as my input data: 

```
Please see #125, #127, and #159

Also see @bed324 and v0.14.0
```

My goal was to produce the following HTML output after telling RDoc what repository
that these issues, commits, and tags referred to:

```
<p>Please see <a href="https://github.com/prawnpdf/prawn/issues/125">#125</a>,
<a href="https://github.com/prawnpdf/prawn/issues/127">#127</a>, and 
<a href="https://github.com/prawnpdf/prawn/issues/159">#159</a></p>

<p>Also see <a
href="https://github.com/prawnpdf/prawn/commit/bed324">@bed324</a> and 
<a href="https://github.com/prawnpdf/prawn/tree/0.14.0">v0.14.0</a></p>
```

Rendered, the resulting HTML would look like this:

> Please see <a href="https://github.com/prawnpdf/prawn/issues/125">#125</a>,
> <a href="https://github.com/prawnpdf/prawn/issues/127">#127</a>, and 
> <a href="https://github.com/prawnpdf/prawn/issues/159">#159</a></p>
>
> Also see <a href="https://github.com/prawnpdf/prawn/commit/bed324">@bed324</a> and 
> <a href="https://github.com/prawnpdf/prawn/tree/0.14.0">v0.14.0</a></p>

I wasn't concerned about styling or how to fit this new functionality into a
full-scale RDoc run. I just wanted to see if I could take my little
snippet of sample text and replace the GitHub references with their 
relevant links. My experiment was focused solely on finding an
suitable entry point into the system that supported these 
kinds of extensions.

After about 20 minutes of research and tinkering, I was able 
to produce the following example:

```ruby
require 'rdoc'

REPO_URL = "https://github.com/prawnpdf/prawn"

class GithubLinkedHtml < RDoc::Markup::ToHtml
  def handle_special_ISSUE(special)
    %{<a href="#{REPO_URL}/issues/#{special.text[1..-1]}">#{special.text}</a>}
  end

  def handle_special_COMMIT(special)
    %{<a href="#{REPO_URL}/commit/#{special.text[1..-1]}">#{special.text}</a>}
  end

  def handle_special_VERSION(special)
    tag = special.text[1..-1]

    %{<a href="#{REPO_URL}/tree/#{tag}">#{special.text}</a>}
  end
end

markup = RDoc::Markup.new

markup.add_special(/\s*(\#\d+)/, :ISSUE)
markup.add_special(/\s*(@\h+)/,  :COMMIT)
markup.add_special(/\s*(v\d+\.\d+\.\d+)/, :VERSION)

wh = GithubLinkedHtml.new(RDoc::Options.new, markup)

puts "<body>#{wh.convert ARGF.read}</body>"
```

Once I figured out the right APIs to use, this became an easy problem to solve.
It was clear from the way things were laid out that this sort of use case had 
already been considered, and a source dive revealed that RDoc also uses these
extension points internally to support its own behavior. The only challenge I ran
into was that these extension points were not especially well documented, which
is unfortunately a more common problem than it ought to be with open 
source projects. 

It is often the case that extension points are built initially to support 
internal needs rather than external use cases, and so they often lag behind 
surface-level features in learnability and third-party usability. This is
certainly a solvable problem, and is worth considering when working on
your own projects. But even without documentation, explicit and stable extension
points can be a hugely powerful tool for making a project more maintainable.

### Reflections

As you've seen from these examples, establishing high quality standards for open 
source projects is a matter of practicality, not pride. Projects that are made up 
of code that is easy to understand, easy to test, easy to change, and easy to 
maintain are far more likely to be sustainable over the long haul than projects 
that are allowed to decay internally as they grow.

The techniques we've discussed in this article are ones that will
pay off even if you just apply them some of the time, but the more you use them,
the more you'll get in return. The nice thing about these practices is that they
are quite robust -- they can be applied in early stage experimental software as
well as in projects that have been used in production for years.

The hard part of applying these ideas is not in remembering them when things are
painful, but instead in keeping up with them when things are going well with
your project. The more contributions you receive, the more important these
strategies will become, but it is also hard to keep up with them because they do
slow down the maintenance process a little bit. Whenever you feel that pressure,
remember that you are looking out for the future of your project by focusing on
quality, and then do what you can to educate others so that they understand why
these issues matter.

Every project is different, and you may find that there are other ways to keep a
high quality standard without following the guidelines we've discussed in this
article. If you have some ideas to share, please let us know!

[^1]: The definition of legacy code as code without tests was popularized in 2004 by Michael Feathers, author of the extremely useful [Working Effectively with Legacy Code](http://www.amazon.com/Working-Effectively-Legacy-Michael-Feathers/dp/0131177052) book.
