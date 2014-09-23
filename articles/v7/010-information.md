Suppose that you want catch up with your friends 
Alice, Bob, and Carol. To do this, you might log into your favorite
chat service, join a group chat room, and then type in some
sort of friendly greeting and hit the enter key. Moments later, your friends would
see your message appear on their screens, and soon after that
they would probably send you some sort of response. As long as
their reply was somewhat intelligible, you could be reasonably 
certain that your message was successfully communicated, without
giving much thought to the underlying delivery mechanism.

Just beneath the surface of this everyday activity, we find a world 
of precise rules and constraints governing our 
communications. In the world of chat clients and servers, the 
meaning of your message does not matter, but its structure is 
of critical importance. Protocols define the format for messages 
to be encoded in, and even small variations will result 
in delivery failures. 

Even though much of this internal complexity is hidden by user interfaces,
message format requirements are not a purely technical
concern -- they can also directly affect human behavior. On Twitter, 
a message needs to be expressed in 140 characters or less, and on 
IRC the limit is only a few hundred characters. This single
constraint makes Twitter and IRC fundamentally different from
email and web forums, so it's hard to overstate the impact
that constraints can have on a communicaitons medium.

In addition to intentional restrictions on message structure,
there are always going to be incidental technical limitations
that need to be dealt with -- the kinds of quirks that arise
from having too much or too little expressiveness[^1] in a given
message format. These unexpected obstacles are among the most 
interesting problems in information exchange, because they are 
not an essential part of the job to be done but rather an
emergent property of the way we've decided to do the job. 

As programmers, we're constantly working to bridge the gap
between people and the machines that serve them. This article
explores the boundary lines between those two disjoint worlds, 
and the complicated decisions that need to be made 
in order to cross the invisible chasm that lies between
computational structures and human meaning.

## The medium is the message

To see the impact a communication medium can have on its messages,
let's work through a practical example. The line of text below is
representative of what an IRC-based chat message look like when it 
get sent over a TCP socket:

```
PRIVMSG #practicing-ruby-testing :Seasons greetings to you all!\r\n
```

Even if you've never used IRC before or looked into its implementation
details, you can extract a great deal of meaning from this single line 
of text. The structure is very simple, so it's fairly obvious that
`PRIVMSG` represents a command, `#practicing-ruby-testing` represents
a channel, and that the message to be delivered is 
`"Seasons greetings to you all!"`. If I asked you to parse this
text to produce the following array, you probably would have
no trouble doing so without any further instruction:

```ruby
["PRIVMSG", "#practicing-ruby-testing", "Seasons greetings to you all!"]
```

But if this were a real project and not just a thought experiment,
you might start to wonder more about the nuances of the protocol. Here
are a few questions that might come up after a few minutes of
careful thought:

* What is the significance of the `:` character? Does it always signify 
the start of the message body, or does it mean something else?

* Why does the message end in `\r\n`? Can the message body contain newlines,
and if so, should they be represented as `\n` or `\r\n`, or something
else entirely?

* Will messages always take the form `"PRIVMSG #channelname :Message Body\r\n"`, 
or are their cases where additional parameters will be used?

* Can channel names include spaces? How about `:` characters?

Try as we might, no amount of analyzing this single example will answer 
these questions for us. That leads us to a very important point: 
Understanding  the *meaning* of a message doesn't necessarily mean that 
we know how to process the information contained within it.

## The meaning of a message depends on its level of abstraction

At first glance, the text-based IRC protocol made it 
easy for us to identify the structure and meaning of the various
parts of a chat message. But when we thought a little more about what 
it would take to actually implement the protocol, we quickly ran 
into several questions about how to construct well-formed messages.

A lot of the questions we came up with had to do with basic syntax
rules, which is only natural when exploring an unfamiliar information
format. For example, we can guess that the `:` symbol is a special character 
in the following text, but we can't reliably guess its meaning without 
reading the formal specification for the IRC protocol:

```
PRIVMSG #practicing-ruby-testing :Seasons greetings to you all!\r\n
```

To see the effect of syntax on our interpretation of information
formats, consider what happens when we shift the representation 
of a chat message into a generic structure that
we are already familiar with, such as a Ruby array: 

```ruby
["PRIVMSG", "#practicing-ruby-testing", "Seasons greetings to you all!"]
```

Looking at this information, we still have no idea whether it 
constitutes a well-formed message to be processed by 
our hypothetical IRC-like chat system. But because we know Ruby's 
syntax, we understand what is being communicated here at
a primitive level.

Before when we looked at the `PRIVMSG` command expressed in
the format specified by the IRC protocol, we weren't able to
reliably determine the rules for breaking the message up
into its parts by looking at a single example. Because
we didn't already have its syntax memorized, we wouldn't even
be able to reliably parse IRC commands, let alone process them.
But as Ruby programmers, we know what array and string literals
look like, and so we know how to map their syntax to the concepts
behind them.

The mundane observation to be made here is that it's easier 
to understand a format you're familiar with than it is to 
interpret one you've never seen before. A far more interesting
point to discover is that these two examples have fundamental
differences in meaning, even if they can be interpreted in
a way that makes them equivalent to one another.

Despite their superficial similarities, the two examples
we've looked at operate at completely different
levels of abstraction. The IRC-based example directly 
encodes the concept of a *chat message*, whereas 
our Ruby example encodes the concept of an *array of strings*. 
In that sense, the former is a direct representation of a 
domain-specific concept, and the latter is a indirect 
representation built up from general-purpose data structures.
Both can express the concept a chat message, but they're not
cut from the same cloth.

Let's investigate why this difference
in structure matters. Consider what might happen if we attempted
to allow whitespace in chat channel names, i.e. 
`#practicing ruby testing` instead of `#practicing-ruby-testing`.
By directly substituting this new channel name into our `PRIVMSG`
command example, we get the text shown below:

```
PRIVMSG #practicing ruby testing :Seasons greetings to you all!\r\n
```

Here we run into a syntactic hiccup: If we allow for channel
names to include whitespace, we need to come up with more complex
rules for splitting up the message into its different parts. But
if we decide this is an ill-formed string, then we need to come
up with a constraint that says that the channel parameter
cannot include spaces in it. Either way, we need to come up
with a formal rule that will be applied at parse time,
before processing even begins.

Now consider what happens when we use Ruby syntax instead:

```ruby
["PRIVMSG", "#practicing ruby testing", "Seasons greetings to you all!"]
```

This is without question a well-formed Ruby array, and it will
be successfully parsed and turned into an internal data structure.
By definition, Ruby string literals allow whitespace in them, 
and there's no getting around that without writing our own 
custom parser. So while the IRC example *must* consider the meaning
of whitespace in channel names during the parsing phase, our
Ruby example *cannot*. Any additional constraints placed on the 
format of channel names would need to be done via logical 
validations rather than syntactic rules.

The key insight here is that the concepts we're expressing
when we encode something in one syntax or another have meaning
beyond their raw data contents. In the IRC protocol
a channel is a defined concept at the symbolic level, with a 
specific meaning to it. When we encode a channel name 
as a Ruby string, we can only approximate the concept by starting with
a more general structure and then applying logical rules to
it to make it a more faithful representation of a concept
it cannot directly express. This is not unlike translating
a word from one spoken language to another which cannot
express the same exact concept using a single word.

## Every expressive syntax has at least a few corner cases

Consider once more our fascinating Ruby array:

```ruby
["PRIVMSG", "#practicing-ruby-testing", "Seasons greetings to you all!"]
```

We've seen that because its structure is highly generic, its
encoding rules are very permissive. Nearly any sequence of
printable characters can be expressed within a Ruby string literal,
and so there isn't much ambiguity in expression of ordinary strings.

Despite its general-purpose nature, there are edge cases in Ruby's
string literal syntax that could lead to ambiguous or incomprehensible messages. 
For example, consider strings which have `"` characters within them:

```
"My name is: "Gregory"\n"
```

The above will generate a syntax error in Ruby, becasuse it ends up
getting parsed as the string `"My name is: "`, followed immediately
by the constant `Gregory`, followed by the string `"\n"`. Ruby
understandably has no way of interpreting that nonsense, so
the parser will fail.

If we were only concerned with parsing string literals, we could 
find a way to resolve these ambiguities by adding some special 
parsing rules, but Ruby has a much more complex grammar across
its entire featureset. For that reason, it expects you to be
a bit more explicit when dealing with edge cases like this one.
To get our string to parse, we'd need to do something like this:

```
"My name is: \"Gregory\"\n"
```

By writing `\"` instead of `"`, we tell the parser
to treat the quote character as just another character in the string
rather than a symbolic *end-of-string* marker. The `\` acts
as an escape character, which is useful for resolving these sorts
of ambiguities. The cost of course is that `\` itself
becomes a potential source of ambiguity, so you end up having to write
`\\` instead of `\` to express backslashes in Ruby
string literals.

Edge cases of this sort arise in any expressive text-based format.
They are often easy to resolve by adding a few more rules, but in many
cases the addition of new processing rules add an even more subtle layer
of corner cases to consider (as we've seen w. the `\` character).
Resolving minor ambiguities comes naturally to humans because we can
guess at the meaning of a message, but cold-hearted computers
can only follow the explicit rules we've given them.

## Can we free ourselves from the limitations of syntax?

One solution to the syntactic ambiguity problem is to represent information in
a way that is convenient for computers, rather than optimizing for
human readability. For example, here's the same array of strings
represented as a raw sequence of bytes in [MessagePack format]:

```
93 a7 50 52 49 56 4d 53 47 b8 23 70 72 61 63 74 69 63 69 6e 67 2d 72 75 62 
79 2d 74 65 73 74 69 6e 67 bd 53 65 61 73 6f 6e 73 20 67 72 65 65 74 69 6e 
67 73 20 74 6f 20 79 6f 75 20 61 6c 6c 21
```

At first, this looks like a huge step backwards, because it smashes our
ability to intuitively extract meaning from the message by simply
reading its contents. But when we discover that the vast majority of
these bytes are just encoded character data, things get a little
more comprehensible:

```ruby
"\x93\xA7PRIVMSG\xB8#practicing-ruby-testing\xBDSeasons greetings to you all!"
```

Knowing that most of the message is the same text we've seen in the other
examples, we only need to figure out what the few extra bytes of information
represent:

![](http://i.imgur.com/YAh5olr.png)

Like all binary formats, MessagePack is optimized for ease of processing
rather than human readability. Instead using text-based symbols to describe 
the structure of data, MessagePack uses an entirely numeric encoding format.

By switching away from brackets, commas, and quotation marks to arbitrary
values like `93`, `A7`, `B8`, and `BD`, we immediately lose the ability to
visually distinguish between the different structural elements of the 
message. This makes it harder to simply look at a message and know whether
or not it is well-formed, and also makes it harder to notice the connections
between the symbols and their meaning while reading an encoded message.

If you squint really hard at the yellow boxes in the above diagram, you might
guess that `93` describes the entire array, and that `A7`, `B8`, and `BD`
all describe the strings that follow them. But `A7`, `B8`, and `BD` need to
be expressing more than just the concept of a *string*, otherwise there
would be no need to use three different values. You might be able to
discover the underlying rule by studying the example for a while, but
it doesn't just jump out at you the way a pair of opening and closing
brackets might.

To avoid leaving you in suspense, here's the key concept: MessagePack
attempts to represent seralized data structures using as few bytes 
as possible, while making processing as fast as possible. To do this,
MessagePack uses type headers that tell you exactly what type of
data is encoded, and exactly how much space it takes up in 
the message. For small chunks of data, it conveys both of these
pieces of information using a single byte!

Take for example the first byte in the message, which has the
hexadecimal value of `93`. MessagePack maps the values `90-9F`
to the concept of *arrays with up to 15 elements*. This
means that an array with zero elements would have the type code 
of `90` and an array with 15 elements would have the type code
of `9F`. Following the same logic, we can see that `93` represents 
an array with 3 elements.

For small strings, a similar encoding process is used. Values in 
the range of `A0-BF` correspond to *strings with up to 31 bytes of data*.
All three of our strings are in this range, so to compute
their size, we just need to subtract the bottom of the range
from each of them:

```ruby
# note that results are in decimal, not hexadecimal
# String sizes are also computed explicitly for comparison

>> 0xA7-0xA0
=> 7
>> "PRIVMSG".size
=> 7

>> 0xB8-0xA0
=> 24
>> "#practicing-ruby-testing".size
=> 24

>> 0xBD-0xA0
=> 29
>> "Seasons greetings to you all!".size
=> 29
```

Piecing this all together, we can now see the orderly structure
that was previously obfuscated by the compact nature of the
MessagePack format:

![](http://i.imgur.com/H9lOSex.png)

Although this appears to be superficially similar to the structure
of our Ruby array example, there are significant differences that
become apparent when attempting to process the MessagePack data:

* In a text-based format you need to look ahead to find closing
brackets to match opening brackets, to organize quotation marks
into pairs, etc. In MessagePack format, explicit sizes for each
object are given so you know exactly where its data is stored
in the bytestream.

* Because we don't need to analyze the contents of the message
to determine how to break it up into chunks, we don't need
to worry about ambiguous interpretation of symbols in the data.
This avoids the need for introducing escape sequences for the
sole purpose of making parsing easier.

* The explicit separation of metadata from the contents of the
message makes it possible to read part of the message without
analyzing the entire bytestream. We just need to extract all
the relevant type and size information, and then from there
it is easy to compute offsets and read just the data we need.

The underlying theme here is that by compressing all of the
structural meaning of the message into simple numerical values,
we convert the whole problem of extracting the message into
a series of trivial computations: read a few bytes to determine
the type information and size of the encoded data, then
read some content and decode it based on the specified type,
then rinse and repeat.

## Separating structure from meaning via abstract types

Even though representing our message in a binary format allowed
us to make information extraction more precise, 
the data type we used still corresponds to concepts that don't exactly
fit the intended meaning of our message.

One possible way to solve this conceptual mapping problem is to completely 
decouple structure from meaning in our message format. To do that,
we could utilize MessagePack's application-specific type mechanism;
resulting in a message similar to what you see below:

![](http://i.imgur.com/s3Rjgzz.png)

The `C7` type code indicates an abstract type, and is followed
by two additional bytes: the first provides an arbitrary type
id (between 0-127), and the second specifies how many bytes
of data to read in that format. After applying these rules,
we end up with the following structure:

![](http://i.imgur.com/AubaxCk.png)

The contents of each object in the array is the same as it always
has been, but now the types have changed. Instead of an
array composed of three strings, we now have an array that
consists of elements that each have their own type.

Although I've illustrated the contents of each object as text-based
strings for the sake of readability,
the MessagePack format does not assume that the data associated
with abstract types will be text-based. The decision of
how to process this data is left up to the decoder.

Without getting into too many details, let's consider how abstract
data types might be handled in a real Ruby program[^3] that processed
MessagePack-based messages. You'd need to make an explicit mapping
between type identifiers and the handlers for each type, perhaps
using an API similar to what you see below:

```ruby
data_types = { 1 => CommandName, 2 => Parameter, 3 => MessageBody }

command = MessagePackDecoder.unpack(raw_bytes, data_types)
#  [ CommandName <"PRIVMSG">, 
#    Parameter   <"#practicing-ruby-testing">, 
#    MessageBody <"Season greetings to you all!"> ]
```

Each handler would be responsible for transforming raw byte arrays
into meaningful data objects. For example, the following class might
be used to convert message parameters (e.g. the channel name) into
a text-based representation:

```ruby
class Parameter
  def initialize(byte_array)
    @text = byte_array.pack("C*")

    raise ArgumentError if @text.include?(" ")
  end

  attr_reader :text
end
```

The key thing to note about the above code sample is that
the `Parameter` handler does not simply convert the raw binary into
a string, it also applies a validation to ensure that the
string contains no space characters. This is a bit of a
contrived example, but it's meant to illustrate the ability
of custom type handlers to apply their own data integrity
constraints.

Earlier we had drawn a line in the sand between the 
array-of-strings representation and the IRC message format
because the former was forced to allow spaces in strings
until after the parsing phase, and the latter was forced
to make a decision about whether to allow them or not
before parsing could be completed at all. The use
of abstract types removes this limitation, allowing us to choose when and where to
apply our validations, if we apply them at all.

Another dividing wall that abstract types seem to blur for
us is the question of what the raw contents of our message
actually represent. Using our own application-specific type
definitions make it so that we never need to consider the
contents of our messages to be strings, except as an
internal implementation detail. However, we rely
absolutely on our decoder to convert data that has been
tagged with these arbitrary type identifiers
into something that matches the underlying meaning of 
the message. In introducing abstract types, we have 
somehow managed to make our information format more precise 
and more opaque at the same time.

## Combining human intuition with computational rigor 

As we explored the MessagePack format, we saw that by coming up with very
precise rules for processing an input stream, we can interpet messages by
running a series of simple and unambiguous computations. But in the
process of making things easier for the computer, we complicated
things for humans. Try as we might, we aren't very good at
rapidly extracting meaning from numeric sequences like
`93`, `C7 01 07`, `C7 02 18`, and `C7 03 1D`.

So now we've come full circle in our explorations, realizing that we really do
want to express ourselves using something like the text-based IRC message 
format. Let's look at it one last time to reflect on its strengths
and weaknesses:

```
PRIVMSG #practicing-ruby-testing :Seasons greetings to you all!\r\n
```

The main feature of representing our message this way is that because we're
familiar with the concept of *commands* as programmers, it is easy to see
the structure of the message without even worrying about its exact syntax 
rules: we know intuitively that `PRIVMSG` is the command being sent,
and that `#practicing-ruby-testing` and `Seasons greetings to you all!`
are its parameters. From here, it's easy to extract the underlying
meaning of the message, which is: "Send the message 'Seasons greetings to you
all!' to the #practicing-ruby-testing channel".

The drawback is that we're hazy on the details: we can't simply guess the rules
about whitespace in parameters, and we don't know exactly how to interpret 
the `:` character or the `\r\n` at the end of the message. Because a correct 
implementation of the IRC protocol will need to consider
various edge cases, attempting to precisely describe the message format
verbally is challenging. That said, we could certainly give
it a try, and see what happens...

* Messages consist of a valid IRC command and its parameters
(if any), followed by `\r\n`.

* Commands are either made up solely of letters, or are
represented as a three digit number.

* All parameters are separated by a single space character.

* Parameters may not contain `\r\n` or the null character (`\0`).

* All parameters except for the last parameter must not contain
spaces and must not start with a `:` character.

* If the last parameter contains spaces or starts with a `:`
character, it must be separated from the rest of the
parameters by a `:` character, unless there are exactly
15 parameters in the message. 

* When all 15 parameters are present, then the separating `:` 
character can be omitted, even if the final parameter
includes spaces.

This ruleset isn't even a complete specification of the message format, 
but it should be enough to show you how specifications written in
prose can quickly devolve into the kind of writing you might expect 
from a tax attorney. Because spoken language is inherently fuzzy and 
subjective in nature, it makes it hard to be both precise and 
understandable at the same time.

To get around these communication barriers, computer scientists
have come up with *metalanguages* to describe the syntactic rules
of protocols and formats. By using precise notation with well-defined 
rules, it is possible to describe a grammar in a way that is both
human readable and computationally unambiguous.

When we look at the real specification for the IRC message format,
we see one of these metalanguages in use. Below
you'll see a nearly complete specification[^2] for the general form
of IRC messages expressed in [Augmented Backus–Naur Form][ABNF]:

```
message    =  command [ params ] crlf
command    =  1*letter / 3digit
params     =  *14( SPACE middle ) [ SPACE ":" trailing ]
           =/ 14( SPACE middle ) [ SPACE [ ":" ] trailing ]

nospcrlfcl =  %x01-09 / %x0B-0C / %x0E-1F / %x21-39 / %x3B-FF
                ; any octet except NUL, CR, LF, " " and ":"

middle     =  nospcrlfcl *( ":" / nospcrlfcl )
trailing   =  *( ":" / " " / nospcrlfcl )

SPACE      =  %x20        ; space character
crlf       =  %x0D %x0A   ; "carriage return" "linefeed"
letter     =  %x41-5A / %x61-7A       ; A-Z / a-z
digit      =  %x30-39                 ; 0-9
```

If you aren't used to reading formal grammar notations, this example may appear
to be a bit opaque at first glance. But if you go back and look at the
rules we listed out in prose above, you'll find that all of them are expressed
here in a way that leaves far less to the imagination. Each rule tells us
exactly what should be read from the input stream, and in what order.

Representing syntactic rules this way allows us to clearly understand
their intended meaning, but that's not the only reason for the formality. 
BNF-based grammar notations express syntactic rules so precisely that we can 
use them not just as a specification for how to build a parser
by hand, but as input data for a code generator that can build
a highly optimized parser for us. This not only saves development effort,
it also reduces the likelihood that some obscure edge case will be
lost in translation when converting grammar rules into raw
processing code.

To demonstrate this technique in use, I converted the
ABNF representation of the IRC message format into a grammar that is 
readable by the [Citrus parser generator][]. Apart from a few lines of 
embedded Ruby code used to transform the input data, the following code look 
conceptually similar to what you saw above:

```
grammar IRC
  rule message
    (command params? endline) {
      { :command => capture(:command).value,
        :params  => capture(:params).value }
    }
  end

  rule command
    letters | three_digit_code 
  end

  rule params
    ( ((space middle)14*14 (space ":"? trailing)?) |
      ((space middle)*14 (space ":" trailing)?) ) {
      captures.fetch(:middle, []) + captures.fetch(:trailing, [])
    }
  end

  rule middle
    non_special (non_special | ":")*
  end

  rule trailing
    (non_special | space | ":")+
  end

  rule letters
    [a-zA-Z]+
  end

  rule three_digit_code
    /\d{3}/ { to_str.to_i }
  end

  rule non_special
    [^\0:\r\n ]
  end

  rule space
    " "
  end

  rule endline
    "\r\n"
  end
end
```

Loading this grammar into Citrus, we end up with a parser that can correctly
extract the commands and paramaters from our original `PRIVMSG` example:

```ruby
require 'citrus'
Citrus.load('irc')

msg = "PRIVMSG #practicing-ruby-testing :Seasons greetings to you all!\r\n"

data = IRC.parse(msg).value

p data[:command] 
#=> "PRIVMSG"

p data[:params]
#=> ["#practicing-ruby-testing", "Seasons greetings to you all!"]
```

In taking this approach, we're forced to accept certain constraints
(like a set of complicated rules about where a `:` character can appear), but
we avoid turning our entire message format into meaningless streams of numbers
like `93` and `C7 01 08`. Even if there is a bit more magic going on in the
conversion of a Citrus grammar into a functioning parser, we can still see
the telltale signs of a deterministic process lurking just beneath the surface.

The decision to express a message in a text-based format or a binary format
is one rife with tradeoffs, as we've already seen from this single example.
Now that you've seen both approaches, consider how you might implement
a few different types of message formats. Would an audio file be better
represented as binary file format, or a text-based format? How about
a web page? Before you read this article you probably already knew the 
answers to those questions, but now hopefully you have a better sense of 
the tradeoffs involved in how we choose to represent information in
software systems.

## The philosophical conundrum of information exchange

Computers are mindless automatons, and humans are bad at numbers. This
friction between people and their machines runs so deep that
it's remarkable that any software gets built
at all. But because there is gold to be found at the
other side of the computational tarpit, we muddle through our differences 
and somehow manage to make it all work.

To work together, computers and humans need a bridge between their mutually
exclusive ways of looking at the world. And this is what coding is all about!
We *encode* information into data and source code for computers to process,
and then after the work is done, we *decode* the results of a computation back
into a human-friendly message format. 

Once everything is wired up, human users of software can think mostly 
in terms of meaningful information exchange, and software systems only need to 
worry about moving numbers around and doing basic arithmetic operations. 
Although it isn't especially romantic, this is how programmers trick computers 
and humans into cooperating with each other. When done well, people barely
notice the presence of the software system at all, and focus entirely on
their job to be done. This suits the computer just fine, as it does not
care at all what puny humans think of it.

As programmers, we must concern ourselves with the needs of both people 
and machines. We are responsible for connecting two seemingly incompatible worlds,
each with their own set of rules and expectations. This is what makes 
our job hard, but is also what makes it rewarding and almost magical 
at times. We've just explored some examples of the sorts of challenges that
can arise along the boundary line between people and machines, 
but I'm sure you can think of many more that are present in your own work. 

The next time you come across a tension point in your software design
process, take a moment to  reflect on these ideas, and see what kind of 
insights arise. Is the decision you're about to make meant to
benefit the people who use your software, or the machines that run your code?
Consider the tradeoffs carefully, but when in doubt, always choose to 
satisfy the humans. :grin:

> **NOTE:** While writing this article, I was also reading "Gödel, Escher, Bach"
in my spare time. Though I don't directly use any of its concepts here, Douglas
Hofstadter deserves credit (and/or blame) for getting me to think deeply
on *the meaning of meaning* and how it relates to software development.

[^1]: Having too much or too little expressiveness in a format is pretty much a guarantee, because even as we get closer to the *Goldilocks Zone*, increasingly subtle edge cases tend to proliferate. Since we can't expect perfection, we need to settle for expressiveness that's "good enough" and the tradeoffs that come along with it.

[^2]: For the sake of simplicity, I omitted the optional prefix in IRC messages which contains information about the sender of a message, because it involves somewhat complicated URI parsing. See [page 7 of the IRC specification](http://tools.ietf.org/html/rfc2812#page-7) for details.

[^3]: The abstract types API shown in this article is only a theoretical example, because the [official MessagePack library](https://github.com/msgpack/msgpack-ruby) for Ruby does not support application-specific types as of September 2014, even though they're documented in the specification. It may be a fun patch to write if you want to explore these topics more, though!

[MessagePack format]: https://github.com/msgpack/msgpack/blob/master/spec.md
[Citrus parser generator]: https://github.com/mjackson/citrus
[ABNF]: http://en.wikipedia.org/wiki/Augmented_Backus%E2%80%93Naur_Form
