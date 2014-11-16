*This issue of Practicing Ruby was contributed by Magnus Holm ([@judofyr][judofyr]), 
a Ruby programmer  from Norway. Magnus works on various open source 
projects (including the [Camping][camping] web framework),
and writes articles over at [the timeless repository][timeless].*

Working with network I/O in Ruby is so easy: 

```ruby
require 'socket'

# Start a server on port 9234
server = TCPServer.new('0.0.0.0', 9234)

# Wait for incoming connections
while io = server.accept
  io << "HTTP/1.1 200 OK\r\n\r\nHello world!"
  io.close
end

# Visit http://localhost:9234/ in your browser.
```

Boom, a server is up and running! Working in Ruby has some disadvantages, though: we
can handle only one connection at a time. We can also have only one *server*
running at a time. There's no understatement in saying that these constraints
can be quite limiting. 

There are several ways to improve this situation, but lately we've seen an
influx of event-driven solutions. [Node.js][nodejs] is just an event-driven I/O-library
built on top of JavaScript. [EventMachine][em] has been a solid solution in the Ruby
world for several years. Python has [Twisted][twisted], and Perl has so many that they even
have [an abstraction around them][anyevent].

Although these solutions might seem like silver bullets, there are subtle details that
you'll have to think about. You can accomplish a lot by following simple rules
("don't block the thread"), but I always prefer to know precisely what I'm
dealing with. Besides, if doing regular I/O is so simple, why does
event-driven I/O have to be looked at as black magic?

To show that they are nothing to be afraid of, we are going to implement an 
I/O event loop in this article. Yep, that's right; we'll capture the core 
part of EventMachine/Node.js/Twisted in about 150 lines of Ruby. It won't 
be performant, it won't be test-driven, and it won't be solid, but it will 
use the same concepts as in all of these great projects. We will start 
by looking at a minimal chat server example and then discuss 
how to build the infrastructure that supports it.

## Obligatory chat server example

Because chat servers seem to be the event-driven equivalent of a
"hello world" program, we will keep with that tradition here. The
following example shows a trivial `ChatServer` object that uses
the `IOLoop` that we'll discuss in this article:

```ruby
class ChatServer
  def initialize
    @clients = []
    @client_id = 0
  end

  def <<(server)
    server.on(:accept) do |stream|
      add_client(stream)
    end
  end

  def add_client(stream)
    id = (@client_id += 1)
    send("User ##{id} joined\n")

    stream.on(:data) do |chunk|
      send("User ##{id} said: #{chunk}")
    end

    stream.on(:close) do
      @clients.delete(stream)
      send("User ##{id} left")
    end

    @clients << stream
  end

  def send(msg)
    @clients.each do |stream|
      stream << msg
    end
  end
end

# usage

io     = IOLoop.new
server = ChatServer.new

server << io.listen('0.0.0.0', 1234)

io.start
```

To play around with this server, run [this script][chatserver] and then open up
a couple of telnet sessions to it. You should be able to produce something like the
following with a bit of experimentation:

```
# from User #1's console:
$ telnet 127.0.0.1 1234

User #2 joined
User #2 said: Hi
Hi
User #1 said: Hi
User #2 said: Bye
User #2 left

# from User #2's console (quits after saying Bye)
$ telnet 127.0.0.1 1234

User #1 said: Hi
Bye
User #2 said: Bye
```

If you don't have the time to try out this code right now,
don't worry: as long as you understand the basic idea behind it, you'll be fine.
This chat server is here to serve as a practical example to help you 
understand [the code we'll be discussing][chatserver] throughout this article.

Now that we have a place to start from, let's build our event system.

## Event handling

First of all we need, obviously, events! With no further ado:

```ruby
module EventEmitter
  def _callbacks
    @_callbacks ||= Hash.new { |h, k| h[k] = [] }
  end

  def on(type, &blk)
    _callbacks[type] << blk
    self
  end

  def emit(type, *args)
    _callbacks[type].each do |blk|
      blk.call(*args)
    end
  end
end

class HTTPServer
  include EventEmitter
end

server = HTTPServer.new
server.on(:request) do |req, res|
  res.respond(200, 'Content-Type' => 'text/html')
  res << "Hello world!"
  res.close
end

# When a new request comes in, the server will run:
#   server.emit(:request, req, res)

```

`EventEmitter` is a module that we can include in classes that can send and
receive events. In one sense, this is the most important part of our event
loop: it defines how we use and reason about events in the system. Modifying it
later will require changes all over the place. Although this particular
implementation is a bit more simple than what you'd expect from a real 
library, it covers the fundamental ideas that are common to all
event-based systems.

## The IO loop

Next, we need something to fire up these events. As you will see in
the following code, the general flow of an event loop is simple:
detect new events, run their associated callbacks, and then repeat
the whole process again.

```ruby
class IOLoop
  # List of streams that this IO loop will handle.
  attr_reader :streams

  def initialize
    @streams = []
  end
  
  # Low-level API for adding a stream.
  def <<(stream)
    @streams << stream
    stream.on(:close) do
      @streams.delete(stream)
    end
  end

  # Some useful helpers:
  def io(io)
    stream = Stream.new(io)
    self << stream
    stream
  end

  def open(file, *args)
    io File.open(file, *args)
  end

  def connect(host, port)
    io TCPSocket.new(host, port)
  end

  def listen(host, port)
    server = Server.new(TCPServer.new(host, port))
    self << server
    server.on(:accept) do |stream|
      self << stream
    end
    server
  end

  # Start the loop by calling #tick over and over again.
  def start
    @running = true
    tick while @running
  end

  # Stop/pause the event loop after the current tick.
  def stop
    @running = false
  end

  def tick
    @streams.each do |stream|
      stream.handle_read  if stream.readable?
      stream.handle_write if stream.writable?
    end
  end
end
```

Notice here that `IOLoop#start` blocks everything until `IOLoop#stop` is called.
Everything after `IOLoop#start` will happen in callbacks, which means that the
control flow can be surprising. For example, consider the following code:

```ruby
l = IOLoop.new

ruby = i.connect('ruby-lang.org', 80)  # 1
ruby << "GET / HTTP/1.0\r\n\r\n"       # 2

# Print output
ruby.on(:data) do |chunk|
  puts chunk   # 3
end

# Stop IO loop when we're done
ruby.on(:close) do
  l.stop       # 4
end

l.start        # 5
```

You might think that you're writing data in step 2, but the
`<<` method actually just stores the data in a local buffer.
It's not until the event loop has started (in step 5) that the data
actually gets sent. The `IOLoop#start` method triggers `#tick` to be run in a loop, which
delegates to `Stream#handle_read` and `Stream#handle_write`. These methods 
are responsible for doing any necessary I/O operations and then triggering
events such as `:data` and `:close`, which you can see being used in steps 3 and 4. We'll take a look at how `Stream` is implemented later, but for now 
the main thing to take away from this example is that event-driven code 
cannot be read in top-down fashion as if it were procedural code.

Studying the implementation of `IOLoop` should also reveal why it's 
so terrible to block inside a callback. For example, take a look at this 
call graph:

```
# indentation means that a method/block is called
# deindentation means that the method/block returned

tick (10 streams are readable)
  stream1.handle_read
    stream1.emit(:data)
      your callback

  stream2.handle_read
    stream2.emit(:data)
      your callback
        you have a "sleep 5" inside here

  stream3.handle_read
    stream3.emit(:data)
      your callback
  ...
```

By blocking inside the second callback, the I/O loop has to wait 5 seconds 
before it's able to call the rest of the callbacks. This wait is
obviously a bad thing, and it is important
to avoid such a situation when possible. Of course, nonblocking
callbacks are not enough—the event loop also needs to make use of nonblocking
I/O. Let's go over that a bit more now.

## IO events

At the most basic level, there are only two events for an `IO` object:

1. Readable: The `IO` is readable; data is waiting for us. 
2. Writable: The `IO` is writable; we can write data.

These might sound a little confusing: how can a client know that the server
will send us data? It can't. Readable doesn't mean "the server will send us
data"; it means "the server has already sent us data." In that case, the data
is handled by the kernel in your OS. Whenever you read from an `IO` object, you're
actually just copying bytes from the kernel. If the receiver does not read 
from `IO`, the kernel's buffer will become full and the sender's `IO` will 
no longer be writable. The sender will then have to wait until the 
receiver can catch up and free up the kernel's buffer. This situation is
what makes nonblocking `IO` operations tricky to work with.

Because these low-level operations can be tedious to handle manually, the 
goal of an I/O loop is to trigger some more usable events for application
programmers:

1. Data: A chunk of data was sent to us.
2. Close: The IO was closed.
3. Drain: We've sent all buffered outgoing data.
4. Accept: A new connection was opened (only for servers).

All of this functionality can be built on top of Ruby's `IO` objects with
a bit of effort.

## Working with the Ruby IO object

There are various ways to read from an `IO` object in Ruby:

```ruby
data = io.read
data = io.read(12)
data = io.readpartial(12)
data = io.read_nonblock(12)
```

* `io.read` reads until the `IO` is closed (e.g., end of file, server closes the
connection, etc.) 

* `io.read(12)` reads until it has received exactly 12 bytes.

* `io.readpartial(12)` waits until the `IO` becomes readable, then it reads *at
most* 12 bytes. So if a server sends only 6 bytes, `readpartial` will return
those 6 bytes. If you had used `read(12)`, it would wait until 6 more bytes were
sent.

* `io.read_nonblock(12)` will read at most 12 bytes if the IO is readable. It
raises `IO::WaitReadable` if the `IO` is not readable.

For writing, there are two methods:

```ruby
length = io.write(str)
length = io.write_nonblock(str)
```

* `io.write` writes the whole string to the `IO`, waiting until the `IO` becomes
writable if necessary. It returns the number of bytes written (which should
always be equal to the number of bytes in the original string).

* `io.write_nonblock` writes as many bytes as possible until the `IO` becomes
nonwritable, returning the number of bytes written. It raises `IO::WaitWritable`
if the `IO` is not writable.

The challenge when both reading and writing in a nonblocking fashion is knowing 
when it is possible to do so and when it is necessary to wait.

## Getting real with IO.select

We need some mechanism for knowing when we can read or write to our
streams, but I'm not going to implement `Stream#readable?` or `#writable?`. It's 
a terrible solution to loop over every stream object in Ruby and check whether it's
readable/writable over and over again. This is really just not a job for Ruby;
it's too far away from the kernel.

Luckily, the kernel exposes ways to efficiently detect readable and writable
I/O streams. The simplest cross-platform method is called select(2) 
and is available in Ruby as `IO.select`:

```
IO.select(read_array [, write_array [, error_array [, timeout]]])

Calls select(2) system call. It monitors supplied arrays of IO objects and waits
until one or more IO objects are ready for reading, ready for writing, or have
errors. It returns an array of those IO objects that need attention. It returns 
nil if the optional timeout (in seconds) was supplied and has elapsed.
```

With this knowledge, we can write a much better `#tick` method:

```ruby
class IOLoop
  def tick
    r, w = IO.select(@streams, @streams)
    r.each do |stream|
      stream.handle_read
    end
  
    w.each do |stream|
      stream.handle_write
    end
  end
end
```

`IO.select` will block until some of our streams become readable or writable
and then return those streams. From there, it is up to those streams to do 
the actual data processing work.

## Handling streaming input and output 

Now that we've used the `Stream` object in various examples, you may 
already have an idea of what its responsibilities are. But let's first take a look at how it is implemented:

```ruby
class Stream
  # We want to bind/emit events.
  include EventEmitter

  def initialize(io)
    @io = io
    # Store outgoing data in this String.
    @writebuffer = ""
  end

  # This tells IO.select what IO to use.
  def to_io; @io end

  def <<(chunk)
    # Append to buffer; #handle_write is doing the actual writing.
    @writebuffer << chunk
  end
  
  def handle_read
    chunk = @io.read_nonblock(4096)
    emit(:data, chunk)
  rescue IO::WaitReadable
    # Oops, turned out the IO wasn't actually readable.
  rescue EOFError, Errno::ECONNRESET
    # IO was closed
    emit(:close)
  end
  
  def handle_write
    return if @writebuffer.empty?
    length = @io.write_nonblock(@writebuffer)
    # Remove the data that was successfully written.
    @writebuffer.slice!(0, length)
    # Emit "drain" event if there's nothing more to write.
    emit(:drain) if @writebuffer.empty?
  rescue IO::WaitWritable
  rescue EOFError, Errno::ECONNRESET
    emit(:close)
  end
end
```

`Stream` is nothing more than a wrapper around a Ruby `IO` object that
abstracts away all the low-level details of reading and writing that were
discussed throughout this article. The `Server` object we make use of 
in `IOLoop#listen` is implemented in a similar fashion but is focused
on accepting incoming connections instead:

```ruby
class Server
  include EventEmitter

  def initialize(io)
    @io = io
  end

  def to_io; @io end
  
  def handle_read
    sock = @io.accept_nonblock
    emit(:accept, Stream.new(sock))
  rescue IO::WaitReadable
  end

  def handle_write
    # do nothing
  end
end
```

Now that you've studied how these low-level objects work, you should
be able to revisit the full [source code for the Chat Server
example][chatserver] and understand exactly how it works. If you
can do that, you know how to build an evented I/O loop from scratch.

### Conclusions

Although the basic ideas behind event-driven I/O systems are easy to understand, 
there are many low-level details that complicate things. This article discussed some of these ideas, but there are many others that would need
to be considered if we were trying to build a real event library. Among
other things, we would need to consider the following problems:

* Because our event loop does not implement timers, it is difficult to do
a number of important things. Even something as simple as keeping a 
connection open for a set period of time can be painful without built-in
support for timers, so any serious event library must support them. It's
worth pointing out that `IO#select` does accept a timeout parameter, and
it would be possible to make use of it fairly easily within this codebase.

* The event loop shown in this article is susceptible to [back pressure][bp],
which occurs when data continues to be buffered infinitely even if it
has not been accepted for processing yet. Because our event loop 
provides no mechanism for signaling that its buffers are full, incoming
data will accumulate and have a similar effect to a memory leak until
the connection is closed or the data is accepted.

* The performance of select(2) is linear, which means that handling 
10,000 streams will take 10,000x as long as handling a single stream. 
Alternative solutions do exist at the kernel, but many are not 
cross-platform and are not exposed to Ruby by default. If you have 
high performance needs, you may want to look into the [nio4r][nio4r] 
project, which attempts to solve this problem in a clean way by 
wrapping the libev library.

The challenges involved in getting the details right in event loops
are the real reason why tools like EventMachine and Node.js exist. These systems
allow application programmers to gain the benefits of event-driven I/O without
having to worry about too many subtle details. Still, knowing how they work under the hood
should help you make better use of these tools, and should also take away some
of the feeling that they are a kind of deep voodoo that you'll never
comprehend. Event-driven I/O is perfectly understandable; it is just a bit 
messy.

[chatserver]: https://gist.githubusercontent.com/sandal/3612925/raw/315e7bfc5de7a029606b3885d71953acb84f112e/ChatServer.rb 
[timeless]: http://timelessrepo.com
[camping]: https://github.com/camping
[judofyr]: http://twitter.com/judofyr
[nodejs]: http://nodejs.org
[em]: http://rubyeventmachine.com
[twisted]: http://twistedmatrix.com
[anyevent]: http://metacpan.org/module/AnyEvent
[libev]: http://software.schmorp.de/pkg/libev.html
[libuv]: https://github.com/joyent/libuv
[nio4r]: https://github.com/tarcieri/nio4r
[bp]: http://en.wikipedia.org/wiki/Back_pressure#Back_pressure_in_information_technology
