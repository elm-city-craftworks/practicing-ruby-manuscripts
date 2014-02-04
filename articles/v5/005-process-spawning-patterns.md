*This article was contributed by [Jesse Storimer](http://jstorimer.com). He is
the author of [Working with Unix Processes](http://workingwithunixprocesses.com)
and [Working with TCP Sockets](http://workingwithtcpsockets.com), a pair of
ebooks providing fundamental Unix knowledge to Ruby developers. When he's not at
the keyboard, he's often enjoying the great Canadian outdoors with his family.*

Like many of you, I discovered Ruby via Rails and web development. That was my
"in." But before it was popular for writing web apps, Ruby was known for its
object-oriented fundamentals and for being a great scripting language. One of the reasons for
this latter benefit is that it's so easy to marry Ruby with command-line
utilities. Here's an example:

```ruby
task :console do
  `irb -r my_app`
end
```

There's something simple and beautiful in the combination of Ruby and the
command line here--the backticks are barely detectable. This code will technically 
accomplish what you think it will: it will drop you into an app-specific console  that is
basically an `irb` session with your app already required. But do you know what's 
going on inside that backtick method? 

Ruby provides many ways of spawing processes. Why use backticks instead of
`system`?

```ruby
task :console do
  system('irb -r my_app')
end
```

Or what about `exec`? Would that have been better?

```ruby
task :console do
  exec('irb', '-r', 'my_app')
end
```

In order to make this decision, you need to understand what these methods are
doing under the hood. The differences may be trivial for spawning a development
console, but picking one of these methods over another in a production environment can
have major implications.

In this article, we're going to reimplement the key parts of these process-spawning
primitives to get a better understanding of how they work and where they're most
applicable. Afterward, you'll have a greater understanding of how process
spawning works regardless of programming language and you'll have a grip on
which methods are most applicable in different situations.

## Starting somewhere

I have already hinted at a few different process-spawning methods--Ruby has
a ton of them. Off the top of my head, there's: `Kernel#system`,
<code>Kernel#\`</code>, `IO.popen`, `Process.spawn`, `open`, `shell`, `open3`,
`pty`, and probably more. All of these ship with Ruby, some in the core and
others in the standard library.

All of these spawning methods boil down to the same pattern, but we're not going
to implement them all. To save time, we'll stick with implementing `system` and
the backtick method. Either of these methods can be called
with a shell command as the argument. Both handle the command in slightly
different ways with slightly different outputs:

``` 
system('ls -l') #=> true
system('ls -l *.rb | ack Product') #=> true
system('boohoo') #=> nil
`git log -n1 --format=%h^` #=> 51e7a1c
`hostname` #=> jessebook
```

Let's start building them.

## Harnessing ourselves with tests

Before we dive into spawning process head first, let's rein ourselves in a
bit. If we're going to reimplement what Ruby already provides, we're going to
need a way to test our implementation and make sure that it performs the same
way that Ruby does. Enter [Rubyspec](http://rubyspec.org).

> The RubySpec project aims to write a complete executable specification for the
> Ruby programming language that is syntax-compatible with RSpec. RSpec is
> essentially a DSL (domain-specific language) for describing the behavior of
> code. This project contains specs that describe Ruby language syntax, core
> library classes, and standard library classes.

RubySpec provides a specification for the Ruby language itself, and we want to
reimplement a part of the Ruby language; therefore, we can use RubySpec
to test our implementation.

To use these specs to drive our implementation, we need to get two
things: RubySpec itself, and its testing library mspec. You can check
out [this README](https://github.com/rubyspec/rubyspec/blob/master/README) 
for installation instructions. To verify that things are working as 
expected, try running the kernel tests from within the RubySpec project
directory:

```bash
$ mspec core/kernel
```

To run our custom code against these tests, we can use
the familiar `-r` option with `mspec` to require a file that redefines
the methods we want to override. Let's do that, while at the same time 
running the `Kernel.system` specs:

```bash
$ touch practicing_spawning.rb
$ mspec -r ./practicing_spawning.rb core/kernel/system_spec.rb
```

Should be all green so far!

## Breaking the test

Let's begin our implementation by causing the tests to fail:

```ruby
# practicing_spawning.rb
module Kernel
  def system(*args)
  end

  private :system
end
```

The very first spec says that `system` should be private. I set that up right
away because it's not the interesting part. If we run the `system` specs again,
we get our first of several failures:

```console
1)
Kernel#system executes the specified command in a subprocess FAILED
Expected (STDOUT): "a\n"
          but got: ""
```

This failure directly relates to the following spec:

```ruby
it "executes the specified command in a subprocess" do
  lambda { @object.system("echo a") }.should output_to_fd("a\n")
end
```

If you've ever used the `system` method, this test should be easy to
understand. It says that shelling out to `echo` should output the echoed string.
If you [dig into](https://github.com/rubyspec/mspec/blob/master/lib/mspec/matchers/output_to_fd.rb#L68-70)
 the `output_to_fd` method that's part of `mspec`, you'll see that it's
expecting this output on `STDOUT`.

## fork and subprocesses

The failing spec title says that `system` spawns a subprocess. If you're
creating new processes on a Unix system, that means using `fork`:

> ------------------------------------------------------------------------------
>   Kernel.fork  [{ block }]   -> fixnum or nil  
>   Process.fork [{ block }]   -> fixnum or nil
>    
> ------------------------------------------------------------------------------
> 
> Creates a subprocess. If a block is specified, that block is run in the
> subprocess, and the subprocess terminates with a status of zero. Otherwise,
> the fork call returns twice, once in the parent, returning the process ID of
> the child, and once in the child, returning nil.

This bit of Ruby documentation gives you an idea of what `fork` does. It's
conceptually similar to going on a hike and coming to a fork in the trail. The
trail represents the execution of a process over time. Whereas humans can only
pick one path, when a process is forked it literally continues down both
branches of the trail in parallel. What was one process becomes two independent 
processes. This behavior is specified by the fork(2) manpage:

> Fork() causes creation of a new process.  The new process (child process) is
> an exact copy of the calling process (parent process) [...]

When you `fork`, you start with one process and end up with two processes that
are *exactly the same*. In some cases, this means that everything is copied from
one process to the other. But if [copy-on-write
semantics](http://en.wikipedia.org/wiki/Copy-on-write) are implemented,
the two processes may physically share memory until one of them tries to
modify it; then each gets its own copy written out.

Although understanding `fork` is certainly helpful, we still haven't quite figured
out how to implement the `system` method. We know that we can take our Ruby 
process and create a copy of it with `fork`, but how do we then turn the 
new child process into an `echo` process?

## fork + exec

The `fork` + `exec` pattern for spawning processes is the blueprint upon which
most process spawning is built. We've already looked at `fork`, so what
about `exec`?

`exec` transforms the current process into another process. Using
`exec`, you can transform a Ruby process into an `ls` process, another Ruby
process, or an `echo` process:

```ruby
puts 'hi from Ruby'
exec('ls')
puts 'bye from Ruby' # will never be reached
```

This program will never get to the last line of Ruby code. Once it has performed
`exec('ls')`, the Ruby program no longer exists. It has been transformed to `ls`.
So there's no possible way for it to get back to this Ruby program and finish
execution.

## Finally, a passing test

With `fork` and `exec`, we now have the building blocks that we need to implement
our own `system` method. Here's the most basic implementation:

```ruby
# practicing_spawning.rb
module Kernel
  def system(*args)

    # Create a new subprocess that will just exec the requested program.
    pid = fork { exec(*args) }

    # Because fork() allows both processes to work in parallel, we must tell the
    # parent process to wait for the child to exit. Otherwise, the parent would
    # continue in parallel with the child and would be unable to process its
    # return value.
    _, status = Process.waitpid2(pid)
    status.success?
  end

  private :system
end
```

If we run this against the same spec as before, more tests pass, but
not all of them. Still, getting that initial spec to pass means that we're headed
in the right direction.

There are three very simple Unix programming primitives in use here: `fork`,
`exec`, and `wait`. We've already talked about `fork` and `exec`, the
cornerstone of Unix process spawning. The third player here, `wait`, is often
used in unison with these two. It tells the parent process to wait for the child
process before continuing, rather than continuing execution in parallel. This is
a pretty common pattern when spawning shell commands, because you usually want to
wait for the output of the command.

In this case, we collect the status of the child when it exits and return the
result of `success?`. This result is `true` for a successful exit status code (i.e., 0)
and `false` for any other value.

## Getting back to green

Now we need to get the rest of the `system` specs passing. In
the remainder of the failures, we see the following output:

```console
1) 
Kernel#system returns nil when command execution fails FAILED
Expected false to be nil
<snipped backtrace...>

2)
Kernel#system does not write to stderr when command execution fails FAILED
Expected (STDERR): ""
         but got: "/[...]/practicing_spawning.rb:8:in `exec': No such 
         file or directory - sad (Errno::ENOENT)
<snipped backtrace...>
```

These failures relate to the following specs:

```ruby
ruby_version_is "1.9" do
  it "returns nil when command execution fails" do
    @object.system("sad").should be_nil
  end
end

it "does not write to stderr when command execution fails" do
  lambda { @object.system("sad") }.should output_to_fd("", STDERR)
end
```

Both of these specs are testing the same situation: trying to `exec` a command
that doesn't exist. When this happens, it actually raises an exception in
the subprocess, as is evidenced by the previously listed failure #2, which prints an
exception message along with a stacktrace on its `STDERR`, whereas the spec
expected that `STDERR` would be empty.

So when the subprocess raises an exception, we need to notify the parent process
of what went wrong. Note that we can't use Ruby's regular exception handling in
this case because the exception is happening inside the subprocess. The
subprocess got a copy of everything that the parent had, including the Ruby
interpreter. So although all of the code is sourced from the same file, we can't
depend on regular Ruby features because the processes are actually running on
their own separate copies of the Ruby interpreter!

To solve this problem, we need some form of interprocess communication (IPC).
Keeping with the general theme of this article, we'll use a Unix pipe.

## The pipe

A call to `IO.pipe` in Ruby will return two `IO` objects, one readable and
one writable. Together, they form a one-way data 'pipe'. Data is written
to one `IO` object and read from the other:

```ruby
rd, wr = IO.pipe
wr.write "ping"
wr.close

rd.read #=> "ping"
```

A pipe can be used for IPC by taking advantage of `fork` semantics. If you
create a pipe before forking, the child process inherits a copy of the pipe
from its parent. As both have a copy, one process can write to the pipe while
the other reads from it, enabling IPC. Pipes are
backed by the kernel itself, so we can use them to communicate between our independent
Ruby processes.

## Implementing system() with a pipe

Now we can roll together all of these concepts and write our own implementation
of `system` that passes all the specs:

```ruby
# practicing_spawning.rb
module Kernel
  def system(*args)

    rd, wr = IO.pipe

    # Create a new subprocess that will just exec the requested program.
    pid = fork do
      # The subprocess closes its copy of the reading end of the pipe
      # because it only needs to write.
      rd.close

      begin
        exec(*args)
      rescue SystemCallError

        # In case of failure, write a byte to the pipe to signal that an exception
        # occurred and exit with an unsuccessful code.
        wr.write('.')
        exit 1
      end
    end

    # The parent process closes its copy of the writing end of the pipe
    # because it only needs to read.
    wr.close

    # Tell the parent to wait.
    _, status = Process.waitpid2(pid)

    # If the reading end of the pipe has no data, there was no exception
    # and we fall back to the exit status code of the subprocess. Otherwise,
    # we return nil to denote the error case.
    if rd.eof?
      status.success?
    else
      nil
    end
  end

  private :system
end
```

All green!

## Implementing backticks

Now that you've got the fundamentals under your belt, we can apply these concepts to the
implementation of other process-spawning methods. Let's do backticks:

```ruby
# practicing_spawning.rb
module Kernel
  def `(str)
    rd, wr = IO.pipe

    # Create a new subprocess that will exec just the requested program.
    pid = fork do
      # The subprocess closes its copy of the reading end of the pipe
      # because it only needs to write.
      rd.close

      # Anything that the exec'ed process would have written to $stdout will
      # be written to the pipe instead.
      $stdout.reopen(wr)

      exec(str)
    end

    # The parent process closes its copy of the writing end of the pipe
    # because it only needs to read.
    wr.close

    # The parent waits for the child to exit.
    Process.waitpid(pid)

    # The parent returns whatever it can read from the pipe.
    rd.read
  end

  private :`
end
```

Now we can run the backticks spec against our implementation and see that it's
all green!

```console
$ mspec -r ./practicing_spawning.rb core/kernel/backtick_spec.rb
```

The full source for our `practicing_spawning.rb` file is available [as a gist](https://gist.github.com/3730986). 

## Closing notes

I find something special in spawning processes. You get to dig down
below the top layer of your programming language to the lower layer where All
Things Are One. When dealing with things such as `fork`, `exec`, and `wait`, your
operating system treats all processes equally. Any Ruby process can transform
into a C program, or a Python process, or vice versa. Similarly, you can `wait`
on processes written in any language. At this layer of abstraction, there are only
the system and its primitives.

We spend a lot of our mental energy worrying about good principles such as
abstraction, decoupling, efficiency. When digging down a layer and learning what
your operating system is capable of, you see an extremely robust and abstract
system. It cares not how you implement your programs but offers the same
functionality for any running program. Understanding your system at this level
will really show you what it's capable of and give you a good mental
understanding of how your system sees the world. Once you really grasp the
`fork` + `exec` concepts, you'll see that these are right at the core of a Unix system.
Every process is spawned this way. The simplest example is your shell, which uses
this very pattern to launch programs.

I'll leave you with two more tips:

1. Use `exec()` at the end of scripts to save a process. Remember the early example
in which a rake task spawned an `irb` session? The obvious
choice in that case is to use `exec`.

    Any other variant will require forking a new process that then execs and
    has the parent wait for it. Using `exec` directly eliminates the need for an extra
    process by transforming the `rake` process directly into an `irb` process.
    This trick obviously won't work in situations where you need to shell out and then
    work with the output, but keep it in mind if the last line of your script
    just shells out.

2. Pass an `Array` instead of a `String`. The backticks method always takes a
string, but the `system` method (and many other process spawning methods) will
take an array or a string. 

    When passed a string, `exec` may spawn a shell to interpret the
    command, rather than executing it directly. This approach is handy for stuff like
    `system('find . | ack foobar -l')` but is very dangerous when user input is
    involved. An unescaped string makes shell injection possible. Shell
    injection is like SQL injection, except that a compromised shell could provide an
    attacker with root access to your entire system! Using an array will never
    spawn a shell but will pass the elements directly as the `ARGV` of the exec'ed process. 
    Always do this.

Finally, if you enjoyed these exercises, try to implement some of
the other process spawning primitives I mentioned. With RubySpec as your guide,
you can try reimplementing just about anything with confidence. Doing so will
surely give you a better understanding of how process spawning works in Ruby--or 
any Unix environment.

Please leave a comment and share your code if you implement some pure-Ruby versions 
of these spawning methods. I'd love to see them!
