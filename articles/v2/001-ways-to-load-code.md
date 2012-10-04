There are many ways to load Ruby code, and that has lead to confusion over the years. In this article, I will give you the backstory behind several conventions seen in the wild and share some stories about how I use those conventions in my own code.

The topic of code loading breaks up naturally into two subtopics: loading code
within your own project and loading code from third-party libraries. People tend
to struggle more with loading code properly within their own projects than they
do with loading code from third-party libraries, so that's what I'll focus on
exclusively in this issue.

For now, I will focus on the basic mechanics of `load()`, `auto_load()`,
`require()`, and `require_relative()`. I'll discuss how they work so you can
then think about how they can be used within your own projects.

### Kernel#load

Suppose we have a file called _calendar.rb_ that contains the code shown here:

```ruby
class Calendar
  def initialize(month, year)
    @month = month
    @year  = year
  end

  # A simple wrapper around the *nix cal command.
  def to_s
    IO.popen(["cal", @month.to_s, @year.to_s]) { |io| io.read }
  end
end

puts Calendar.new(8, 2011)
```

Given an absolute path to this file, the contents will be loaded and then
executed immediately:

```console
>> load "/Users/seacreature/devel/practicing-ruby-2/calendar.rb"
    August 2011
Su Mo Tu We Th Fr Sa
    1  2  3  4  5  6
 7  8  9 10 11 12 13
14 15 16 17 18 19 20
21 22 23 24 25 26 27
28 29 30 31
```

I can also just specify a path relative to my current working directory and get the same results. That means that if _calendar.rb_ is in the same directory from which I invoked my irb session, I'm able to call `load()` in the manner shown here:

```console
>> load "./calendar.rb"
    August 2011
Su Mo Tu We Th Fr Sa
    1  2  3  4  5  6
 7  8  9 10 11 12 13
14 15 16 17 18 19 20
21 22 23 24 25 26 27
28 29 30 31
```

An interesting thing about `load()` is that it does not do any checks to see
whether it has already loaded a file and will happily reload and reexecute a
file each time you tell it to. So, in practice, the implementation of `load()`
is functionally similar to the code shown here:

```ruby
def fake_load(file)
  eval File.read(file)
  true
end
```

The main benefit of indiscriminately reloading and reexecuting code is that you
can make changes to your files and then `load()` them again within a single
session without having to restart the program that's loading the code. So, for
example, if we changed _calendar.rb_ to output August 2012 instead of August
2011, we could just load it again without restarting irb. But we'd also be
greeted with some warnings in the process:


```console
>> load "./calendar.rb"
/Users/seacreature/devel/practicing-ruby-2/calendar.rb:2: 
warning: method redefined; discarding old initialize
/Users/seacreature/devel/practicing-ruby-2/calendar.rb:2: 
warning: previous definition of initialize was here
/Users/seacreature/devel/practicing-ruby-2/calendar.rb:8: 
warning: method redefined; discarding old to_s
/Users/seacreature/devel/practicing-ruby-2/calendar.rb:8:
warning: previous definition of to_s was here
August 2012
Su Mo Tu We Th Fr Sa
      1  2  3  4
5  6  7  8  9 10 11
12 13 14 15 16 17 18
19 20 21 22 23 24 25
26 27 28 29 30 31
```

If you remember that Ruby classes and modules are permanently open to
modification, these warnings should make a lot of sense. The first time we
called `load()`, it defined the `initialize()` and `to_s()` methods for the
`Calendar` class. The second time we called `load()`, that class and its methods
already existed, so it redefined them. This is not necessarily a sign of a bug,
but Ruby warns you of the possibility that it might be.

Ultimately, these warnings are Ruby telling you that there is probably a better
way for you to do what you're trying to do. One interesting way to get around
the problem is to use `Kernel#load()`'s wrap functionality.  Rather than telling
you directly how it works, I'm going to show you by example and see if you can
guess what's going on.

Suppose we kill our irb session and fire up a new one; we're now back to a blank
slate. We then run the following code and see the familiar calendar output:

```console
>> load "./calendar.rb", true
    August 2012
Su Mo Tu We Th Fr Sa
          1  2  3  4
 5  6  7  8  9 10 11
12 13 14 15 16 17 18
19 20 21 22 23 24 25
26 27 28 29 30 31
```

Then we decide that we want to look a little deeper into the future so that we
know what to plan for in AD 2101. We reload the code using the same command as
before:

```console
>> load "./calendar.rb", true
    August 2101
Su Mo Tu We Th Fr Sa
    1  2  3  4  5  6
 7  8  9 10 11 12 13
14 15 16 17 18 19 20
21 22 23 24 25 26 27
28 29 30 31
```

This time, we don't see any warnings, so obviously something has changed. Here's
a clue:

```console
>> Calendar
NameError: uninitialized constant Object::Calendar
  from (irb):2
  from /.../.rvm/rubies/ruby-1.9.2-p180/bin/irb:16:in `<main>'
```

Surely the `Calendar` class must have been defined *somewhere*, because the
program worked as expected. So what is going on here? Take a look at the
following code; it should give you a clearer picture of what is happening:

```ruby
def fake_load(file)
  Module.new.module_eval(File.read(file))
  true
end
```

In this implementation, our approximation of `load()` is evaluating the loaded
code in the context of an anonymous module, which essentially wraps everything
its own namespace. This step prevents any of the constants defined in the loaded
code from being defined within the global namespace, including any class or
module definitions.

The existence of this option is a hint that although `load()` is suitable for
code loading, it is geared more to implementing customized runners for Ruby code
than to simply loading the classes and modules in your projects. So if you've
been using `load()` on a daily basis, you might be using the wrong tool for the
job at least some of the time. It should be clear by the end of this article why
that is the case.

Now that we have looked at the most simple code loading behavior Ruby has to
offer, we will jump straight into the deep end and explore one of its most
complex options: loading code on demand via `Kernel#autoload`.

### Kernel#autoload

Regardless of whether you've used it explicitly in your own projects, the
concept of automatically loading code on demand should be familiar to anyone
familiar with Rails. In Rails, none of the classes or modules you define are
loaded until the first time they are referenced in your running program. There
are two main benefits to this design: faster startup time and delayed loading of
optional dependencies.

Rails uses its own customized code to accomplish this result, but the basic idea
is similar to what can be done with Ruby's `autoload()` method. To illustrate
how `autoload()` works, let's revisit our `Calendar` class that we began
building while discussing `load()`. This time, we have a file called
_calendar.rb_ that contains only the definition of the `Calendar` class, not the
code that actually calls methods on it:

```ruby
class Calendar
  def initialize(month, year)
    @month = month
    @year  = year
  end

  # A simple wrapper around the *nix cal command.
  def to_s
    IO.popen(["cal", @month.to_s, @year.to_s]) { |io| io.read }
  end
end
```

The following irb session demonstrates the behavior of `autoload()`. 

```console
>> autoload(:Calendar, "./calendar.rb") #1
=> nil
>> defined?(Calendar)                   #2
=> nil
>> puts Calendar.new(8,2011)            #3
    August 2011
Su Mo Tu We Th Fr Sa
    1  2  3  4  5  6
 7  8  9 10 11 12 13
14 15 16 17 18 19 20
21 22 23 24 25 26 27
28 29 30 31

=> nil
>> defined?(Calendar)                   #4
=> "constant"
```

In our first step, we set up the `autoload()` hook, instructing Ruby to load the
file _calendar.rb_ at the time that the first constant lookup happens for the
Calendar constant. In the second step, we check to ensure that `autoload()` does
not actually load the file for you automatically by verifying that Calendar has
not yet been defined. Then, in our third step, we build and output our Calendar.
Last, we see that the constant is now defined.

This exposes us to some cool Ruby voodoo while also raising a lot of questions.
It may help to approximate how `autoload()` might be implemented in order to
wrap your head around the idea. Although the following code is evil and should
never be used for anything but educational purposes, it simulates the load on
demand behavior nicely.

```ruby
$load_hooks = Hash.new

module Kernel
  def fake_autoload(constant_name, file_name)
    $load_hooks[constant_name] = file_name
  end
end

def Object.const_missing(constant)
  load $load_hooks[constant]
  const_get(constant)
end

fake_autoload :Calendar, "./calendar.rb"
p defined?(Calendar)
puts Calendar.new(8,2011)
p defined?(Calendar)
```

After reading the previous example code and playing with it a bit, remember the
dependency on `const_missing()` and forget pretty much everything else about the
implementation. The real `autoload()` handles a lot more cases than this trivial
example gives it credit for.

With the `const_missing()` dependency in mind, try to guess what will happen
when the following code is run:

```ruby
class Calendar; end

autoload :Calendar, "./calendar.rb"
p defined?(Calendar)
puts Calendar.new(8,2011)
p defined?(Calendar)
```

If you guessed that it didn't output a nicely formatted calendar, you guessed
correctly. Below you can see that when I run this script, all the code in
_calendar.rb_ never gets loaded, so the default `Object#initialize` and
`Object#to_s` are being called instead:

```console
"constant"
<Calendar:0x0000010086d6b0>
"constant"
```

Because `autoload()` does not check to see whether a constant is already defined
when it registers its hook, you do not get an indication that the _calendar.rb_
file was never loaded until you actually try to use functionality defined in
that file. Thus `autoload()` is safe to use only when there is a single, uniform
place where a constant is meant to be defined; it cannot be used to
incrementally build up class or module definitions from several different source
files.

This sort of rigidity is frustrating, because unlike load(), which does not care
how or where you define your code, `autoload()` is much more opinionated. What
you've seen here is a single example of the constraints it puts on you, but it
is easy to imagine other scenarios in which `autoload()` can feel like a brittle
way to load code. I'll leave it up to you to try to figure out some of those
issues, but feel free to ask me for some hints if you get stumped.

In the context of Rails—particularly when working in development mode, in which
the whole environment gets reloaded on every request—some form of automatic
loading makes sense. However, outside of that environment, the drawbacks of
`autoload()` tend to outweigh the benefits, so most Ruby projects tend to avoid
it entirely by making heavy use of `require()`.

### Kernel#require()

If you've written any code at all outside of Rails, odds are you've used
`require()` before. It is actually quite similar to `load()` but has a few
additional features that come in handy. To illustrate how `require()` works,
let's revisit our original _calendar.rb_ file, the one that had a bit of code to
be executed in the end of it.

```ruby
class Calendar
  def initialize(month, year)
    @month = month
    @year  = year
  end

  # A simple wrapper around the *nix cal command.
  def to_s
    IO.popen(["cal", @month.to_s, @year.to_s]) { |io| io.read }
  end
end

puts Calendar.new(8, 2011)
```

If we attempt to load this code twice via `require()`, we immediately see an
important way in which it differs from `load()`.

```console
>> require "./calendar.rb" #1
    August 2011
Su Mo Tu We Th Fr Sa
    1  2  3  4  5  6
 7  8  9 10 11 12 13
14 15 16 17 18 19 20
21 22 23 24 25 26 27
28 29 30 31

=> true
>> require "./calendar.rb" #2
=> false
```

When I ran `require()` the first time, the familiar calendar output greeted me,
and then the function returned a true value. The second time I ran it, nothing
happened and the function returned false. This is a feature, and not a bug. The
following code is a crude approximation of what is going on under the hood in
`require()`:

```ruby
$LOADED_BY_FAKE_REQUIRE = Array.new

def fake_require(file)
  full_path = File.expand_path(file)
  return false if $LOADED_BY_FAKE_REQUIRE.include?(full_path)

  load full_path
  $LOADED_BY_FAKE_REQUIRE << full_path

  return true
end
```

This behavior ensures that each file loaded by `require()` is loaded exactly
once, even if the `require()` calls appear in many places. Therefore, updates to
those files will take effect after they have been loaded once. Although this
behavior makes `require()` less suitable than `load()` for quick exploratory
code loading, it does prevent programs from needlessly reloading the same code
again and again, similar to how `autoload()` works once a constant has been
loaded.

Another interesting property of `require()` is that you can omit the file
extension when loading your code. Thus `require("./calendar")` will work just as
well as `require("./calendar.rb")`. Though this may seem like a small feature,
the reason it exists is that Ruby can load more than just Ruby files.
When you omit an extension on a file loaded with `require()`, it will attempt to
load the file with the ".rb" extension first, but will then cycle through the
file extensions used by C extensions as well, such as ".so", ".o", and ".dll".
Despite being an obscure property, it's one that we often take for
granted when we load certain standard libraries or third-party gems. This
behavior is another detail that separates `require()` from `load()`, as the
latter can work only with explicit file extensions.

The main benefit of using `require()` is that it provides the explicit,
predictable loading behavior of `load()` with the caching functionality of
`autoload()`. It also feels natural for those who use RubyGems, as the standard
way of loading libraries distributed as gems is via the patched version of
`Kernel#require()` that RubyGems provides.

Using `require()` will take you far, but it suffers from a pretty irritating
problem—shared by `load()` and `autoload()`—with the way it looks up files. The
`require_relative()` is meant to solve that problem, so we'll take a look at it
now.

### Kernel#require_relative()

Each time I referenced files using a relative path in the previous examples, I
wrote the path to explicitly reference the current working directory. If you're
used to using Ruby 1.8, this may come as a surprise to you. If you've been using
Ruby 1.9.2, it may or may not appear to be the natural thing to do. However, now
is the time when I confess that it's almost always the wrong way to go about
things.

Ruby 1.9.2 removes the current working directory from your path by default for
security reasons. So, in our previous example, if we attempted to write
`require("calendar")` instead of `require("./calendar")`, it would fail on Ruby
1.9.2 even if we invoked irb in the same folder as the _calendar.rb_ file.
Explicitly referencing the current working directory works on both Ruby 1.8.7
and Ruby 1.9.2, which is why this convention was born. Unfortunately, it is an
antipattern, because it forces us to assume that our code will be run from a
particular place on the file system.

Imagine a more typically directory structure, such as this:


```console
lib/
  calendar.rb
  calendar/
    month.rb
    year.rb
bin/
  calendar.rb
```

We could have a _bin/ruby_calendar.rb_ file that looks like this code:

```ruby
require "lib/calendar"

case ARGV.size
when 2
  puts Calendar::Month.new(ARGV[0], ARGV[1])
when 1
  puts Calendar::Year.new(ARGV[0])
else
  raise "Invalid arguments"
end
```

Similarly, our _lib/calendar.rb file_ might include `require()` calls such as
these:

```ruby
require "lib/calendar/year"
require "lib/calendar/month"
```

Now if we run _bin/ruby_calendar.rb_ from the project root, things will work as
expected.

```bash
$ ruby bin/ruby_calendar.rb 2011
# ...
```

But if we ran this file from any other directory, it'd fail to work as expected
because the relative paths would be evaluated relative to wherever you executed
the files from, not relative to where the files live on the file system. That
is, if you execute _ruby_calendar.rb_ in the _bin/_ folder, it would look for a file
called _bin/lib/calendar.rb_.

One way to solve this problem is to use the same mechanism that the Ruby
standard library and RubyGems uses: modify the loadpath.

In _bin/ruby_calendar.rb_, we rewrite our code to match this:

```ruby
$LOAD_PATH.unshift("#{File.dirname(__FILE__)}/../lib")
require "calendar"

case ARGV.size
when 2
  puts Calendar::Month.new(ARGV[0], ARGV[1])
when 1
  puts Calendar::Year.new(ARGV[0])
else
  raise "Invalid arguments"
end
```

Because we've added the _lib/_ folder to the lookup path for all `require()`
calls in our application, we can modify _lib/calendar.rb_ to match the
following:

```ruby
require "calendar/year"
require "calendar/month"
```

This approach makes it possible to run the _ruby_calendar.rb_ program from any
location within the file system, as long as we tell ruby where to find it. That
means you can run it directly from within the _bin/_ folder, or even with an
absolute path.


```bash
# NOTE: this is common in cron jobs.
$ ruby /Users/seacreature/devel/ruby_calendar/bin/ruby_calendar.rb
```

This approach works, and was quite common in Ruby for some time. Then, people
began to get itchy about it, because it is definitely overkill. It effectively
adds an entire folder to the `$LOAD_PATH`, giving Ruby one more place it has to
look on every require and possibly leading to unexpected naming conflicts
between libraries.

The solution to that problem is to not mess with the `$LOAD_PATH` in your code.
Therefore, you expect either that the `$LOAD_PATH` variable will be properly set
by the `-I` flag when you invoke ruby or irb, or that you have to write code
that dynamically determines the proper relative paths to require based on your
current working directory. The latter approach requires less effort from the end
user but makes your code ugly. Below you'll see what people resorted to on Ruby
1.8 before a better solution came along:


```ruby
# bin/ruby_calendar.rb
require "#{File.dirname(__FILE__)}/../lib/calendar"

case ARGV.size
when 2
  puts Calendar::Month.new(ARGV[0], ARGV[1])
when 1
  puts Calendar::Year.new(ARGV[0])
else
  raise "Invalid arguments"
end

# lib/calendar.rb
require "#{File.dirname(__FILE__)}/calendar/year"
require "#{File.dirname(__FILE__)}/calendar/month"
```

Using this approach, you do not add anything to the `$LOAD_PATH` but instead
dynamically build up relative paths by referencing the `__FILE__` variable and
getting a path to the directory it's in. This code will evaluate to different
values depending on where you run it from, but in the end, the right path will
be produced and things will just work.

Predictably, people took efforts to hide this sort of ugliness behind helper
functions, and one such function was eventually adopted into Ruby 1.9. That
helper is predictably called `require_relative()`. Using `require_relative()`,
we can simplify our calls significantly while preserving the "don't touch the
`$LOAD_PATH` variable" ethos:


```ruby
# bin/ruby_calendar.rb
require_relative "../lib/calendar"

case ARGV.size
when 2
  puts Calendar::Month.new(ARGV[0], ARGV[1])
when 1
  puts Calendar::Year.new(ARGV[0])
else
  raise "Invalid arguments"
end

# lib/calendar.rb
require_relative "calendar/year"
require_relative "calendar/month"
```

This code looks and feels like it would work in the way that we'd like to think
`require()` would work. The files we reference are relative to the file in which
the actual calls are made, rather than the folder in which the script was
executed in. For this reason, it is a much better approach than pretty much
anything I've shown so far.

Of course, it is not a perfect solution. In some cases, it does not work as
expected, such as in Rackup files. Additionally, because it's a Ruby 1.9
feature, it's not built into Ruby 1.8.7. The former issue cannot be worked
around, but the latter can be. I'll go into a bit more detail about both of
these issues in the recommendations section, which is coming up right now.

### Conventions and Recommendations

If you remember one thing from this article, it should be that whenever it's
possible to use `require_relative()` and there isn't an obviously better
solution, it's probably the right tool to reach for. It has the fewest 
dark corners and pretty much just works.

That said, take my advice with a grain of salt. I no longer actively
maintain any Ruby 1.8 applications, nor do I have to deal with code that must
run on both Ruby 1.8 and 1.9. If I were in those shoes again, I'd weigh
out four different possible ways of approaching things:

1) Explicitly use `require()` with the `File.dirname(__FILE__)` hack

2) Write my own `require_relative()` implementation leaning on the previous
   hack that gets defined only if `require_relative()` isn't already
   implemented

3) Add a dependency for Ruby 1.8 only on the `require_relative` gem

4) Assume that `$LOAD_PATH` is set for me via the `-I` flag on execution,
   or some other means, and then write ordinary require calls 
   relative to the _lib/_ folder in my project.

I can't give an especially good picture of when I'd pick one of those options
over the other, because it's been about a year since I've last had to think
about it. But any of those four options seem like at least reasonable ideas. I
would *not* employ the common but painfully ugly
`require("./file_in_the_working_dir.rb")` hack in any code that I expected to
use for anything more than a spike or demonstration.

Whether using `require_relative()` explicitly, or one of the workarounds listed
above, I like to use some form of relative require whenever I can. Occasionally,
I do use `load()`, particularly in spikes where I want to  reload files into an
irb session without restarting irb.  But I don't think that `load()` ends up in
production code of mine unless there is a very good reason to use it. A possible
good reason would be if I were building some sort of script runner, such as what
you could find in Rails when it reloads your development environment or in
autotest. In the autotest case in particular in which your test files are
reloaded each time you make an edit to any of your files in your project, it
seems that using `load()` with its obscure second parameter is a good idea. But
these are not tools I'd expect to be building on a daily basis, so `load()`
remains somewhat of an obscure tool for me.

I never use `autoload()`. I've just not run into the issues that some folks in
Rails experience regarding slow startup times of applications in any way that
has mattered to me. I feel like the various gotchas that come along with using
`autoload()` and the strict conventions it enforces are not good things to
impose on general-purpose uses of Ruby. I don't know whether I think that it
makes sense in to context of Rails, but that's a very different question than
whether it should be used in ordinary Ruby applications and libraries. It makes
at least some sense in Rails, but in most Ruby applications, it does not. The
only time I might think about looking into `autoload()` is if I had some sort of
optional dependency that I wanted to be loaded only on demand. I have never
actually run into that issue, and I've found that the following hack provides a
way to do optional dependencies that seems to work just fine:

```ruby
begin
  require "some_external_dependency"
  require "my_lib/some_feature_that_depends_on_dependency"
rescue LoadError
  warn "Could not load some_external_dependency."+
       " Some features are disabled"
end
```

But really, optional dependencies are things I very rarely need to think about.
There are valid use cases for them, but unless something is very difficult to
install or your project is specifically meant to wrap various mutually exclusive
dependencies, I typically will just load up all my dependencies regardless of
whether the user ends up using them. This policy has not caused me problems,
but your mileage will vary depending on the type of work you are doing.

On a somewhat tangential note, I try to avoid things like dynamic require calls
in which I walk over a file list generated from something like `Dir.glob()` or
the like. I also avoid using `Bundler.require()`, even when I use bundler. The
reason I avoid these things is because I like to be able control the exact order
in which my files and my dependencies are being loaded. It's possible to not
have to worry about this sort of thing, but doing so requires a highly
disciplined way of organizing your code so that files can be loaded
independently. 

### Questions / Feedback 

I hope this background story about the various ways to load code along with the
few bits of advice I've offered in the end here have been useful to you. I am
happy to answer whatever questions you have; just leave a comment below.
