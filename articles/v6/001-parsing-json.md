*This article was written by Aaron Patterson, a Ruby
developer living in Seattle, WA.  He's been having fun writing Ruby for the past
7 years, and hopes to share his love of Ruby with you.*

Hey everybody!  I hope you're having a great day today!  The sun has peeked out
of the clouds for a bit today, so I'm doing great!

In this article, we're going to be looking at some compiler tools for use with Ruby.  In
order to explore these tools, we'll write a JSON parser.  I know you're saying,
"but Aaron, *why* write a JSON parser?  Don't we have like 1,234,567 of them?".
Yes!  We do have precisely 1,234,567 JSON parsers available in Ruby!  We're
going to parse JSON because the grammar is simple enough that we can finish the
parser in one sitting, and because the grammar is complex enough that we can
exercise some of Ruby's compiler tools.

As you read on, keep in mind that this isn't an article about parsing JSON, 
its an article about using parser and compiler tools in Ruby.

## The Tools We'll Be Using

I'm going to be testing this with Ruby 2.1.0, but it should work under any
flavor of Ruby you wish to try.  Mainly, we will be using a tool called `Racc`,
and a tool called `StringScanner`.

**Racc**

We'll be using Racc to generate our parser.  Racc is an LALR parser generator
similar to YACC.  YACC stands for "Yet Another Compiler Compiler", but this is
the Ruby version, hence "Racc".  Racc converts a grammar file (the ".y" file)
to a Ruby file that contains state transitions.  These state transitions are
interpreted by the Racc state machine (or runtime).  The Racc runtime ships
with Ruby, but the tool that converts the ".y" files to state tables does not.
In order to install the converter, do `gem install racc`.

We will write ".y" files, but users cannot run the ".y" files.  First we convert
them to runnable Ruby code, and ship the runnable Ruby code in our gem.  In
practical terms, this means that *only we install the Racc gem*, other users
do not need it.

Don't worry if this doesn't make sense right now.  It will become more clear
when we get our hands dirty and start playing with code.

**StringScanner**

Just like the name implies, [StringScanner](http://ruby-doc.org/stdlib-1.9.3/libdoc/strscan/rdoc/StringScanner.html)
is a class that helps us scan strings.  It keeps track of where we are
in the string, and lets us advance forward via regular expressions or by
character.

Let's try it out!  First we'll create a `StringScanner` object, then we'll scan
some letters from it:

```ruby
require 'strscan'

ss = StringScanner.new 'aabbbbb' #=> #<StringScanner 0/7 @ "aabbb...">
ss.scan /a/ #=> "a"
ss.scan /a/ #=> "a"
ss.scan /a/ #=> nil
ss #=> #<StringScanner 2/7 "aa" @ "bbbbb">
```

Notice that the third call to
[StringScanner#scan](http://ruby-doc.org/stdlib-1.9.3/libdoc/strscan/rdoc/StringScanner.html#method-i-scan)
resulted in a `nil`, since the regular expression did not match from the current
position.  Also note that when you inspect the `StringScanner` instance, you can
see the position of the scanner (in this case `2/7`).

We can also move through the scanner character by character using
[StringScanner#getch](http://ruby-doc.org/stdlib-1.9.3/libdoc/strscan/rdoc/StringScanner.html#method-i-getch):

```ruby
ss #=> #<StringScanner 2/7 "aa" @ "bbbbb">
ss.getch #=> "b"

ss #=> #<StringScanner 3/7 "aab" @ "bbbb">
```

The `getch` method returns the next character, and advances the pointer by one.

Now that we've covered the basics for scanning strings, let's take a 
look at using Racc.

## Racc Basics

As I said earlier, Racc is an LALR parser generator.  You can think of it as a
system that lets you write limited regular expressions that can execute
arbitrary code at different points as they're being evaluated.

Let's look at an example.  Suppose we have a pattern we want to match:
`(a|c)*abb`.  That is, we want to match any number of 'a' or 'c' followed by
'abb'.  To translate this to a Racc grammar, we try to break up this regular
expression to smaller parts, and assemble them as the whole.  Each part is
called a "production".  Let's try breaking up this regular expression so that we
can see what the productions look like, and the format of a Racc grammar file.

First we create our grammar file.  At the top of the file, we declare the Ruby
class to be produced, followed by the `rule` keyword to indicate that we're
going to declare the productions, followed by the `end` keyword to indicate the
end of the productions:

```
class Parser
rule
end
```

Next lets add the production for "a|c".  We'll call this production `a_or_c`:


```
class Parser
rule
  a_or_c : 'a' | 'c' ;
end
```

Now we have a rule named `a_or_c`, and it matches the characters 'a' or 'c'.  In
order to match one or more `a_or_c` productions, we'll add a recursive
production called `a_or_cs`:

```
class Parser
rule
  a_or_cs
    : a_or_cs a_or_c
    | a_or_c
    ;
  a_or_c : 'a' | 'c' ;
end
```

The `a_or_cs` production recurses on itself, equivalent to the regular
expression `(a|c)+`.  Next, a production for 'abb':

```
class Parser
rule
  a_or_cs
    : a_or_cs a_or_c
    | a_or_c
    ;
  a_or_c : 'a' | 'c' ;
  abb    : 'a' 'b' 'b' 
end
```

Finally, the `string` production ties everything together:


```
class Parser
rule
  string
    : a_or_cs abb
    | abb
    ;
  a_or_cs
    : a_or_cs a_or_c
    | a_or_c
    ;
  a_or_c : 'a' | 'c' ;
  abb    : 'a' 'b' 'b';
end
```

This final production matches one or more 'a' or 'c' characters followed by
'abb', or just the string 'abb' on its own.  This is equivalent to our original
regular expression of `(a|c)*abb`.

**But Aaron, this is so long!**

I know, it's much longer than the regular expression version.  However, we can
add arbitrary Ruby code to be executed at any point in the matching process.
For example, every time we find just the string "abb", we can execute some
arbitrary code:

```
class Parser
rule
  string
    | a_or_cs abb
    | abb         
    ;
  a_or_cs
    : a_or_cs a_or_c
    | a_or_c
    ;
  a_or_c : 'a' | 'c' ;
  abb    : 'a' 'b' 'b' { puts "I found abb!" };
end
```

The Ruby code we want to execute should be wrapped in curly braces and placed
after the rule where we want the trigger to fire.

To use this parser, we also need a tokenizer that can break the input
data into tokens, along with some other boilerplate code. If you are curious
about how that works, you can check out [this standalone
example](https://gist.githubusercontent.com/sandal/9532497/raw/8e3bb03fc24c8f6604f96516bf242e7e13d0f4eb/parser_example.y).

Now that we've covered the basics, we can use knowledge we have so far to build 
an event based JSON parser and tokenizer.

## Building our JSON Parser

Our JSON parser is going to consist of three different objects, a parser, a
tokenizer, and document handler.The parser will be written with a Racc grammar, 
and will ask the tokenizer for input from the input stream.  Whenever the parser 
can identify a part of the JSON stream, it will send an event to the document 
handler.  The document handler is responsible for collecting the JSON 
information and translating it to a Ruby data structure. When we read in 
a JSON document, the following method calls are made:

![method calls](//i.imgur.com/HZ0Sa.png)

It's time to get started building this system. We'll focus on building the 
tokenizer first, then work on the grammar for the parser, and finally implement 
the document handler.

## Building the tokenizer

Our tokenizer is going to be constructed with an IO object.  We'll read the
JSON data from the IO object.  Every time `next_token` is called, the tokenizer
will read a token from the input and return it. Our tokenizer will return the 
following tokens, which we derived from the [JSON spec](http://www.json.org/):

* Strings
* Numbers
* True
* False
* Null

Complex types like arrays and objects will be determined by the parser.

**`next_token` return values:**

When the parser calls `next_token` on the tokenizer, it expects a two element
array or a `nil` to be returned.  The first element of the array must contain
the name of the token, and the second element can be anything (but most people
just add the matched text).  When a `nil` is returned, that indicates there are
no more tokens left in the tokenizer.

**`Tokenizer` class definition:**

Let's look at the source for the Tokenizer class and walk through it:

```ruby
module RJSON
  class Tokenizer
    STRING = /"(?:[^"\\]|\\(?:["\\\/bfnrt]|u[0-9a-fA-F]{4}))*"/
    NUMBER = /-?(?:0|[1-9]\d*)(?:\.\d+)?(?:[eE][+-]?\d+)?/
    TRUE   = /true/
    FALSE  = /false/
    NULL   = /null/

    def initialize io
      @ss = StringScanner.new io.read
    end

    def next_token
      return if @ss.eos?

      case
      when text = @ss.scan(STRING) then [:STRING, text]
      when text = @ss.scan(NUMBER) then [:NUMBER, text]
      when text = @ss.scan(TRUE)   then [:TRUE, text]
      when text = @ss.scan(FALSE)  then [:FALSE, text]
      when text = @ss.scan(NULL)   then [:NULL, text]
      else
        x = @ss.getch
        [x, x]
      end
    end
  end
end
```

First we declare some regular expressions that we'll use along with the string
scanner.  These regular expressions were derived from the definitions on
[json.org](http://www.json.org).  We instantiate a string scanner object in the
constructor.  String scanner requires a string on construction, so we read the
IO object.  However, we could build an alternative tokenizer that reads from the
IO as needed.

The real work is done in the `next_token` method.  The `next_token` method
returns nil if there is nothing left to read from the string scanner, then it
tries each regular expression until it finds a match.  If it finds a match, it
returns the name of the token (for example `:STRING`) along with the text that
it matched.  If none of the regular expressions match, then we read one
character off the scanner, and return that character as both the name of the
token, and the value.

Let's try feeding the tokenizer a JSON string and see what tokens come out:

```ruby
tok = RJSON::Tokenizer.new StringIO.new '{"foo":null}'
#=> #<RJSON::Tokenizer:0x007fa8529fbeb8 @ss=#<StringScanner 0/12 @ "{\"foo...">>

tok.next_token #=> ["{", "{"]
tok.next_token #=> [:STRING, "\"foo\""]
tok.next_token #=> [":", ":"]
tok.next_token #=> [:NULL, "null"]
tok.next_token #=> ["}", "}"]
tok.next_token #=> nil
```

In this example, we wrap the JSON string with a `StringIO` object in order to
make the string quack like an IO.  Next, we try reading tokens from the
tokenizer.  Each token the Tokenizer understands has the name as the first value of
the array, where the unknown tokens have the single character value.  For
example, string tokens look like this: `[:STRING, "foo"]`, and unknown tokens
look like this: `['(', '(']`.   Finally, `nil` is returned when the input has
been exhausted.

This is it for our tokenizer.  The tokenizer is initialized with an `IO` object, 
and has only one method: `next_token`.  Now we can focus on the parser side.

## Building the parser

We have our tokenizer in place, so now it's time to assemble the parser.  First
we need to do a little house keeping.  We're going to generate a Ruby file from
our `.y` file.  The Ruby file needs to be regenerated every time the `.y` file
changes.  A Rake task sounds like the perfect solution.

**Defining a compile task:**

The first thing we'll add to the Rakefile is a rule that says *"translate .y files to
.rb files using the following command"*:

```ruby
rule '.rb' => '.y' do |t|
  sh "racc -l -o #{t.name} #{t.source}"
end
```

Then we'll add a "compile" task that depends on the generated `parser.rb` file:

```ruby
task :compile => 'lib/rjson/parser.rb'
```

We keep our grammar file as `lib/rjson/parser.y`, and when we run `rake
compile`, rake will automatically translate the `.y` file to a `.rb` file using
Racc.

Finally we make the test task depend on the compile task so that when we run
`rake test`, the compiled file is automatically generated:

```ruby
task :test => :compile
```

Now we can compile and test the `.y` file.

**Translating the JSON.org spec:**

We're going to translate the diagrams from [json.org](http://www.json.org/) to a
Racc grammar.  A JSON document should be an object or an array at the root, so
we'll make a production called `document` and it should be an `object` or an
`array`:

```
rule
  document
    : object
    | array
    ;
```

Next we need to define `array`.  The `array` production can either be empty, or
contain 1 or more values:

```
  array
    : '[' ']'
    | '[' values ']'
    ;
```

The `values` production can be recursively defined as one value, or many values
separated by a comma:

```
  values
    : values ',' value
    | value
    ;
```

The JSON spec defines a `value` as a string, number, object, array, true, false,
or null.  We'll define it the same way, but for the immediate values such as
NUMBER, TRUE, and FALSE, we'll use the token names we defined in the tokenizer:

```
  value
    : string
    | NUMBER
    | object
    | array
    | TRUE
    | FALSE
    | NULL
    ;
```

Now we need to define the `object` production.  Objects can be empty, or
have many pairs:

```
  object
    : '{' '}'
    | '{' pairs '}'
    ;
```

We can have one or more pairs, and they must be separated with a comma.  We can
define this recursively like we did with the array values:

```
  pairs
    : pairs ',' pair
    | pair
    ;
```

Finally, a pair is a string and value separated by a colon:

```
  pair
    : string ':' value
    ;
```

Now we let Racc know about our special tokens by declaring them at the top, and
we have our full parser:

```
class RJSON::Parser
token STRING NUMBER TRUE FALSE NULL
rule
  document
    : object
    | array
    ;
  object
    : '{' '}'
    | '{' pairs '}'
    ;
  pairs
    : pairs ',' pair
    | pair
    ;
  pair : string ':' value ;
  array
    : '[' ']'
    | '[' values ']'
    ;
  values
    : values ',' value
    | value
    ;
  value
    : string
    | NUMBER
    | object
    | array
    | TRUE
    | FALSE
    | NULL
    ;
  string : STRING ;
end
```

## Building the handler

Our parser will send events to a document handler.  The document handler will
assemble the beautiful JSON bits in to lovely Ruby object!  Granularity of the
events is really up to you, but I'm going to go with 5 events:

* `start_object` - called when an object is started
* `end_object`   - called when an object ends
* `start_array`  - called when an array is started
* `end_array`    - called when an array ends
* `scalar`       - called with terminal values like strings, true, false, etc

With these 5 events, we can assemble a Ruby object that represents the JSON
object we are parsing.

**Keeping track of events**

The handler we build will simply keep track of events sent to us by the parser.
This creates tree-like data structure that we'll use to convert JSON to Ruby.

```ruby
module RJSON
  class Handler
    def initialize
      @stack = [[:root]]
    end

    def start_object
      push [:hash]
    end

    def start_array
      push [:array]
    end

    def end_array
      @stack.pop
    end
    alias :end_object :end_array

    def scalar(s)
      @stack.last << [:scalar, s]
    end

    private

    def push(o)
      @stack.last << o
      @stack << o
    end
  end
end
```

When the parser encounters the start of an object, the handler pushes a list on
the stack with the "hash" symbol to indicate the start of a hash.  Events that
are children will be added to the parent, then when the object end is
encountered the parent is popped off the stack.

This may be a little hard to understand, so let's look at some examples.  If we
parse this JSON: `{"foo":{"bar":null}}`, then the `@stack` variable will look
like this:

```ruby
[[:root,
  [:hash,
    [:scalar, "foo"],
    [:hash,
      [:scalar, "bar"],
      [:scalar, nil]]]]]
```

If we parse a JSON array, like this JSON: `["foo",null,true]`, the `@stack`
variable will look like this:

```ruby
[[:root,
  [:array,
    [:scalar, "foo"],
    [:scalar, nil],
    [:scalar, true]]]]
```

**Converting to Ruby:**

Now that we have an intermediate representation of the JSON, let's convert it to
a Ruby data structure.  To convert to a Ruby data structure, we can just write a
recursive function to process the tree:

```ruby
def result
  root = @stack.first.last
  process root.first, root.drop(1)
end

private
def process type, rest
  case type
  when :array
    rest.map { |x| process(x.first, x.drop(1)) }
  when :hash
    Hash[rest.map { |x|
      process(x.first, x.drop(1))
    }.each_slice(2).to_a]
  when :scalar
    rest.first
  end
end
```

The `result` method removes the `root` node and sends the rest to the `process`
method.  When the `process` method encounters a `hash` symbol it builds a hash
using the children by recursively calling `process`.  Similarly, when an
`array` symbol is found, an array is constructed recursively with the children.
Scalar values are simply returned (which prevents an infinite loop).  Now if we
call `result` on our handler, we can get the Ruby object back.

Let's see it in action:

```ruby
require 'rjson'

input   = StringIO.new '{"foo":"bar"}'
tok     = RJSON::Tokenizer.new input
parser  = RJSON::Parser.new tok
handler = parser.parse
handler.result # => {"foo"=>"bar"}
```

**Cleaning up the RJSON API:**

We have a fully function JSON parser.  Unfortunately, the API is not very
friendly.  Let's take the previous example, and package it up in a method:

```ruby
module RJSON
  def self.load(json)
    input   = StringIO.new json
    tok     = RJSON::Tokenizer.new input
    parser  = RJSON::Parser.new tok
    handler = parser.parse
    handler.result
  end
end
```

Since we built our JSON parser to deal with IO from the start, we can add
another method for people who would like to pass a socket or file handle:

```ruby
module RJSON
  def self.load_io(input)
    tok     = RJSON::Tokenizer.new input
    parser  = RJSON::Parser.new tok
    handler = parser.parse
    handler.result
  end

  def self.load(json)
    load_io StringIO.new json
  end
end
```

Now the interface is a bit more friendly:

```ruby
require 'rjson'
require 'open-uri'

RJSON.load '{"foo":"bar"}' # => {"foo"=>"bar"}
RJSON.load_io open('http://example.org/some_endpoint.json')
```

## Reflections 

So we've finished our JSON parser.  Along the way we've studied compiler
technology including the basics of parsers, tokenizers, and even interpreters
(yes, we actually interpreted our JSON!).  You should be proud of yourself!

The JSON parser we've built is versatile. We can:

* Use it in an event driven manner by implementing a Handler object
* Use a simpler API and just feed strings
* Stream in JSON via IO objects

I hope this article has given you the confidence to start playing with parser
and compiler technology in Ruby. Please leave a comment if you have any
questions for me.

## Post Script

I want to follow up with a few bits of minutiae that I omitted to maintain
clarity in the article:

* [Here](https://github.com/tenderlove/rjson/blob/master/lib/rjson/parser.y) is
the final grammar file for our JSON parser.  Notice 
the [---- inner section in the .y file](https://github.com/tenderlove/rjson/blob/master/lib/rjson/parser.y#L53).
Anything in that section is included *inside* the generated parser class.  This
is how we get the handler object to be passed to the parser.

* Our parser actually [does the
translation](https://github.com/tenderlove/rjson/blob/master/lib/rjson/parser.y#L42-50)
of JSON terminal nodes to Ruby.  So we're actually doing the translation of JSON
to Ruby in two places: the parser *and* the document handler.  The document
handler deals with structure where the parser deals with immediate values (like
true, false, etc).  An argument could be made that none or all of this
translation *should* be done in the parser.

* Finally, I mentioned that [the
tokenizer](https://github.com/tenderlove/rjson/blob/master/lib/rjson/tokenizer.rb)
buffers.  I implemented a simple non-buffering tokenizer that you can read
[here](https://github.com/tenderlove/rjson/blob/master/lib/rjson/stream_tokenizer.rb).
It's pretty messy, but I think could be cleaned up by using a state machine.

That's all. Thanks for reading! <3 <3 <3

> NOTE: If you'd like to learn more about this topic, consider doing the Practicing Ruby self-guided course on [Streams, Files, and Sockets](https://practicingruby.com/articles/study-guide-1?u=dc2ab0f9bb). You've already completed one of its reading exercises by working through this article!
