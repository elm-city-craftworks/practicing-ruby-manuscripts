Ruby is best known as a web development language, but in its early days it was
mainly used on the command line. In this article, we'll get back to those roots by building a partial implementation of the standard Unix command `cat`.

The core purpose of the `cat` utility is to read in a list of input files, concatenate them, and output the resulting text to the command line. You can also use `cat` for a few other useful things, such as adding line numbers and suppressing extraneous whitespace. If we stick to these commonly used features, the core functionality of `cat` is something even a novice programmer would be able to implement without too much effort.

The tricky part of building a `cat` clone is that it involves more than just
some basic text manipulation; you also need to know about some 
stream processing and error handling techniques that are common in Unix
utilities. The [acceptance tests](https://gist.github.com/1293709) 
that I've used to compare the original `cat` utility to my Ruby-based `rcat` 
tool reveal some of the extra details that need to be considered when
building this sort of command line application.

If you are already fairly comfortable with building command line tools, you may
want to try implementing your own version of `rcat` before reading on. But don't
worry if you wouldn't even know where to start: I've provided a 
detailed walkthrough of my solution that will teach you everything 
that you need to know.

> **NOTE:** You'll need to have the source code for [my implementation of rcat](https://github.com/elm-city-craftworks/rcat) easily accessible as you work through the rest of this article. Please either clone the repository now or keep the GitHub file browser open while reading.

### Building an executable script

Our first task is to make it possible to run the `rcat` script without having to type something like `ruby path/to/rcat` each time we run it. This task can be done in three easy steps.

**1) Add a shebang line to your script.**

If you look at `bin/rcat` in my code, you'll see that it starts with the following line:

```
#!/usr/bin/env ruby
```

This line (commonly called a shebang line) tells the shell what interpreter to use to process the rest of the file. Rather than providing a path directly to the Ruby interpreter, I instead use the path to the standard `env` utility. This step allows `env` to figure out which `ruby` executable is present in our current environment and to use that interpreter to process the rest of the file. This approach is preferable because it is [more portable](http://en.wikipedia.org/wiki/Shebang_line#Portability) than hard-coding a path to a particular Ruby install. Although Ruby can be installed in any number of places, the somewhat standardized location of `env` makes it reasonably dependable.

**2) Make your script executable.**

Once the shebang line is set up, it's necessary to update the permissions on the `bin/rcat` file. Running the following command from the project root will make `bin/rcat` executable:

```
$ chmod +x bin/rcat
```

Although the executable has not yet been added to the shell's lookup path, it is now possible to test it by providing an explicit path to the executable.

```
$ ./bin/rcat data/gettysburg.txt
Four score and seven years ago, our fathers brought forth on this continent a
new nation, conceived in Liberty and dedicated to the proposition that all men
are created equal.

... continued ...
```

**3) Add your script to the shell's lookup path.**

The final step is to add the executable to the shell's lookup path so that it can be called as a simple command. In Bash-like shells, the path is updated by modifying the `PATH` environment variable, as shown in the following example:

```
$ export PATH=/Users/seacreature/devel/rcat/bin:$PATH
```

This command prepends the `bin` folder in my rcat project to the existing contents of the `PATH`, which makes it possible for the current shell to call the `rcat` command without specifying a direct path to the executable, similar to how we call ordinary Unix commands:

```
$ rcat data/gettysburg.txt
Four score and seven years ago, our fathers brought forth on this continent a
new nation, conceived in Liberty and dedicated to the proposition that all men
are created equal.

... continued ...
```

To confirm that you've followed these steps correctly and that things are working as expected, you can now run the acceptance tests. If you see anything different than the following output, retrace your steps and see whether you've made a mistake somewhere. If not, please leave a comment and I'll try to help you out.

```
$ ruby tests.rb 
You passed the tests, yay!
```

Assuming that you have a working `rcat` executable, we can now move on to talk about how the actual program is implemented.

### Stream processing techniques

We now can turn our focus to the first few acceptance tests from the _tests.rb_ file. The thing that all these use cases have in common is that they involve very simple processing of input and output streams, and nothing more. 

```ruby
cat_output  = `cat #{gettysburg_file}`
rcat_output = `rcat #{gettysburg_file}`

fail "Failed 'cat == rcat'" unless cat_output == rcat_output

############################################################################

cat_output  = `cat #{gettysburg_file} #{spaced_file}`
rcat_output = `rcat #{gettysburg_file} #{spaced_file}`

fail "Failed 'cat [f1 f2] == rcat [f1 f2]'" unless cat_output == rcat_output

############################################################################

cat_output  = `cat < #{spaced_file}`
rcat_output = `rcat < #{spaced_file}`

fail "Failed 'cat < file == rcat < file" unless cat_output == rcat_output
```

If we needed only to pass these three tests, we'd be in luck. Ruby provides a special stream object called `ARGF` that combines multiple input files into a single stream or falls back to standard input if no files are provided. Our entire script could look something like this:

```ruby
ARGF.each_line { |line| print line }
```

However, the real `cat` utility does a lot more than what `ARGF` provides,
so it was necessary to write some custom code to handle stream processing:

```ruby
module RCat
  class Application
    def initialize(argv)
      @params, @files = parse_options(argv)

      @display        = RCat::Display.new(@params)
    end

    def run
      if @files.empty?
        @display.render(STDIN)
      else
        @files.each do |filename|
          File.open(filename) { |f| @display.render(f) }
        end 
      end
    end

    def parse_options(argv)
      # ignore this for now
    end
  end
end
```

The main difference between this code and the `ARGF`-based approach is that `RCat::Application#run` creates a new stream for each file. This comes in handy later when working on support for empty line suppression and complex line numbering but also complicates the implementation of the `RCat::Display` object. In the following example, I've stripped away the code that is related to these more complicated features to make it a bit easier for you to see the overall flow of things:

```ruby
module RCat
  class Display
    def render(data)
      lines = data.each_line
      loop { render_line(lines) }
    end

    private

    def render_line(lines)
      current_line = lines.next 
      print current_line
    end
  end
end
```

The use of `loop` instead of an ordinary Ruby iterator might feel a bit strange here, but it works fairly well in combination with `Enumerator#next`. The following irb session demonstrates how the two interact with one another:

```
>> lines = "a\nb\nc\n".each_line
=> #<Enumerator: "a\nb\nc\n":each_line>
>> loop { p lines.next }
"a\n"
"b\n"
"c\n"
=> nil

>> lines = "a\nb\nc\n".each_line
=> #<Enumerator: "a\nb\nc\n":each_line>
>> lines.next
=> "a\n"
>> lines.next
=> "b\n"
>> lines.next
=> "c\n"

>> lines.next
StopIteration: iteration reached an end
  from (irb):8:in `next'
  from (irb):8
  from /Users/seacreature/.rvm/rubies/ruby-1.9.3-rc1/bin/irb:16:in `<main>'

>> loop { raise StopIteration }
=> nil
```

Using this pattern makes it possible for `render_line` to actually consume more
than one line from the input stream at once. If you work through the logic that
is necessary to get the following test to pass, you might catch a glimpse of the
benefits of this technique:

```ruby
cat_output  = `cat -s #{spaced_file}`
rcat_output = `rcat -s #{spaced_file}`

fail "Failed 'cat -s == rcat -s'" unless cat_output == rcat_output
```

Tracing the executation path for `rcat -s` will lead you to this line of code in
`render_line`, which is the whole reason I decided to use this
`Enumerator`-based implementation:

```ruby
lines.next while lines.peek.chomp.empty?
```

This code does an arbitrary amount of line-by-line lookahead until either a nonblank line is found or the end of the file is reached. It does so in a purely stateless and memory-efficient manner and is perhaps the most interesting line of code in this entire project. The downside of this approach is that it requires the entire `RCat::Display` object to be designed from the ground up to work with `Enumerator` objects. However, I struggled to come up with an alternative implementation that didn't involve some sort of complicated state machine/buffering mechanism that would be equally cumbersome to work with.

As tempting as it is to continue discussing the pros and cons of the different
ways of solving this particular problem, it's probably best for us to get back on
track and look at some more basic problems that arise when working on
command-line applications. I will now turn to the `parse_options` method that I asked you 
to treat as a black box in our earlier examples.

### Options parsing

Ruby provides two standard libraries for options parsing: `GetoptLong` and `OptionParser`. Though both are fairly complex tools, `OptionParser` looks and feels a lot more like ordinary Ruby code while simultaneously managing to be much more powerful. The implementation of `RCat::Application#parse_options` makes it clear what a good job `OptionParser` does when it comes to making easy things easy:

```ruby
module RCat
  class Application
    # other code omitted

    def parse_options(argv)
      params = {}
      parser = OptionParser.new 

      parser.on("-n") { params[:line_numbering_style] ||= :all_lines         }
      parser.on("-b") { params[:line_numbering_style]   = :significant_lines }
      parser.on("-s") { params[:squeeze_extra_newlines] = true               }
      
      files = parser.parse(argv)

      [params, files]
    end
  end
end
```

The job of `OptionParser#parse` is to take an arguments array and match it against the callbacks defined via the `OptionParser#on` method. Whenever a flag is matched, the associated block for that flag is executed. Finally, any unmatched arguments are returned. In the case of `rcat`, the unmatched arguments consist of the list of files we want to concatenate and display. The following example demonstrates what's going on in `RCat::Application`:

```ruby
require "optparse"

puts "ARGV is #{ARGV.inspect}"

params = {}
parser = OptionParser.new 

parser.on("-n") { params[:line_numbering_style] ||= :all_lines         }
parser.on("-b") { params[:line_numbering_style]   = :significant_lines }
parser.on("-s") { params[:squeeze_extra_newlines] = true               }

files = parser.parse(ARGV)

puts "params are #{params.inspect}"
puts "files are #{files.inspect}"
```

Try running this script with various options and see what you end up with. You should get something similar to the output shown here:

````
$ ruby option_parser_example.rb -ns data/*.txt
ARGV is ["-ns", "data/gettysburg.txt", "data/spaced_out.txt"]
params are {:line_numbering_style=>:all_lines, :squeeze_extra_newlines=>true}
files are ["data/gettysburg.txt", "data/spaced_out.txt"]

$ ruby option_parser_example.rb data/*.txt
ARGV is ["data/gettysburg.txt", "data/spaced_out.txt"]
params are {}
files are ["data/gettysburg.txt", "data/spaced_out.txt"]
```

Although `rcat` requires us to parse only the most basic form of arguments, `OptionParser` is capable of a whole lot more than what I've shown here. Be sure to check out its [API documentation](http://ruby-doc.org/stdlib-1.9.2/libdoc/optparse/rdoc/OptionParser.html#method-i-parse) to see the full extent of what it can do.

Now that I've covered how to get data in and out of our `rcat` application, we can talk a bit about how it does `cat`-style formatting for line numbering.

### Basic text formatting 

Formatting text for the console can be a bit cumbersome, but some things are easier than they seem. For example, the tidy output of `cat -n` shown here is not especially hard to implement:

<pre style="font-size: 0.8em">
$ cat -n data/gettysburg.txt 
   1  Four score and seven years ago, our fathers brought forth on this continent a
   2  new nation, conceived in Liberty and dedicated to the proposition that all men
   3  are created equal.
   4  
   5  Now we are engaged in a great civil war, testing whether that nation, or any
   6  nation so conceived and so dedicated, can long endure. We are met on a great
   7  battle-field of that war. We have come to dedicate a portion of that field as a
   8  final resting place for those who here gave their lives that that nation might
   9  live. It is altogether fitting and proper that we should do this.
  10  
  11  But, in a larger sense, we can not dedicate -- we can not consecrate -- we can
  12  not hallow -- this ground. The brave men, living and dead, who struggled here
  13  have consecrated it far above our poor power to add or detract. The world will
  14  little note nor long remember what we say here, but it can never forget what
  15  they did here. It is for us the living, rather, to be dedicated here to the
  16  unfinished work which they who fought here have thus far so nobly advanced. It
  17  is rather for us to be here dedicated to the great task remaining before us --
  18  that from these honored dead we take increased devotion to that cause for which
  19  they gave the last full measure of devotion -- that we here highly resolve that
  20  these dead shall not have died in vain -- that this nation, under God, shall
  21  have a new birth of freedom -- and that government of the people, by the people,
  22  for the people, shall not perish from the earth.
</pre>

On my system, `cat` seems to assume a fixed-width column with space for up to six digits. This format looks great for any file with fewer than a million lines in it, but eventually breaks down once you cross that boundary.

```
$ ruby -e "1_000_000.times { puts 'blah' }" | cat -n | tail
999991    blah
999992    blah
999993    blah
999994    blah
999995    blah
999996    blah
999997    blah
999998    blah
999999    blah
1000000    blah
```

This design decision makes implementing the formatting code for this feature a whole lot easier. The `RCat::Display#print_labeled_line` method shows that it's possible to implement this kind of formatting with a one-liner:

```ruby
def print_labeled_line(line)
  print "#{line_number.to_s.rjust(6)}\t#{line}" 
end
```

Although the code in this example is sufficient for our needs in `rcat`, it's worth mentioning that `String` also supports the `ljust` and `center` methods. All three of these justification methods can optionally take a second argument, which causes them to use an arbitrary string as padding rather than a space character; this feature is sometimes useful for creating things like ASCII status bars or tables.

I've worked on a lot of different command-line report formats before, and I can tell you that streamable, fixed-width output is the easiest kind of reporting you'll come by. Things get a lot more complicated when you have to support variable-width columns or render elements that span multiple rows and columns. I won't get into the details of how to do those things here, but feel free to leave a comment if you're interested in hearing more on that topic.

### Error handling and exit codes

The techniques we've covered so far are enough to get most of `rcat`'s tests passing, but the following three scenarios require a working knowledge of how Unix commands tend to handle errors. Read through them and do the best you can to make sense of what's going on.

```ruby
`cat #{gettysburg_file}`
cat_success = $?

`rcat #{gettysburg_file}`
rcat_success = $?

unless cat_success.exitstatus == 0 && rcat_success.exitstatus == 0
  fail "Failed 'cat and rcat success exit codes match"
end

############################################################################

cat_out, cat_err, cat_process    = Open3.capture3("cat some_invalid_file")
rcat_out, rcat_err, rcat_process = Open3.capture3("rcat some_invalid_file") 

unless cat_process.exitstatus == 1 && rcat_process.exitstatus == 1
  fail "Failed 'cat and rcat exit codes match on bad file"
end

unless rcat_err == "rcat: No such file or directory - some_invalid_file\n"
  fail "Failed 'cat and rcat error messages match on bad file'"
end

############################################################################


cat_out, cat_err, cat_proccess  = Open3.capture3("cat -x #{gettysburg_file}")
rcat_out,rcat_err, rcat_process = Open3.capture3("rcat -x #{gettysburg_file}") 

unless cat_process.exitstatus == 1 && rcat_process.exitstatus == 1
  fail "Failed 'cat and rcat exit codes match on bad switch"
end

unless rcat_err == "rcat: invalid option: -x\nusage: rcat [-bns] [file ...]\n"
  fail "Failed 'rcat provides usage instructions when given invalid option"
end
```

The first test verifies exit codes for successful calls to `cat` and `rcat`. In Unix programs, exit codes are a means to pass information back to the shell about whether a command finished successfully. The right way to signal that things worked as expected is to return an exit code of 0, which is exactly what Ruby does whenever a program exits normally without error.

Whenever we run a shell command in Ruby using backticks, a `Process::Status` object is created and is then assigned to the `$?` global variable. This object contains (among other things) the exit status of the command that was run. Although it looks a bit cryptic, we're able to use this feature to verify in our first test that both `cat` and `rcat` finished their jobs successfully without error.

The second and third tests require a bit more heavy lifting because in these scenarios, we want to capture not only the exit status of these commands, but also whatever text they end up writing to the STDERR stream. To do so, we use the `Open3` standard library. The `Open3.capture3` method runs a shell command and then returns whatever was written to STDOUT and STDERR, as well as a `Process::Status` object similar to the one we pulled out of `$?` earlier. 

If you look at _bin/rcat_, you'll find the code that causes these tests to pass:

```ruby
begin
  RCat::Application.new(ARGV).run
rescue Errno::ENOENT => err
  abort "rcat: #{err.message}"
rescue OptionParser::InvalidOption => err
  abort "rcat: #{err.message}\nusage: rcat [-bns] [file ...]"
end
```

The `abort` method provides a means to write some text to STDERR and then exit with a nonzero code. The previous code provides functionality equivalent to the following, more explicit code:

```ruby
begin
  RCat::Application.new(ARGV).run
rescue Errno::ENOENT => err
  $stderr.puts "rcat: #{err.message}"
  exit(1)
rescue OptionParser::InvalidOption => err
  $stderr.puts "rcat: #{err.message}\nusage: rcat [-bns] [file ...]"
  exit(1)
end
```

Looking back on things, the errors I've rescued here are somewhat low level, and
it might have been better to rescue them where they occur and then reraise
custom errors provided by `RCat`. This approach would lead to code similar to
what is shown below:

```ruby
begin
  RCat::Application.new(ARGV).run
rescue RCat::Errors::FileNotFound => err
  # ...
rescue RCat::Errors::InvalidParameter => err
  # ..
end
```

Regardless of how these exceptions are labeled, it's important to note that I intentionally let them bubble all the way up to the outermost layer and only then rescue them and call `Kernel#exit`. Intermingling `exit` calls within control flow or modeling logic makes debugging nearly impossible and also makes automated testing a whole lot harder.

Another thing to note about this code is that I write my error messages to `STDERR` rather than `STDOUT`. Unix-based systems give us these two different streams for a reason: they let us separate debugging output and functional output so that they can be redirected and manipulated independently. Mixing the two together makes it much more difficult for commands to be chained together in a pipeline, going against the [Unix philosophy](http://en.wikipedia.org/wiki/Unix_philosophy).

Error handling is a topic that could easily span several articles. But when it comes to building command-line applications, you'll be in pretty good shape if you remember just two things: use `STDERR` instead of `STDOUT` for debugging output, and make sure to exit with a nonzero status code if your application fails to do what it is supposed to do. Following those two simple rules will make your application play a whole lot nicer with others.

### Reflections

Holy cow, this was a hard article to write! When I originally decided to write a `cat` clone, I worried that the example would be too trivial and boring to be worth writing about. However, once I actually implemented it and sat down to write this article, I realized that building command-line applications that respect Unix philosophy and play nice with others is harder than it seems on the surface.

Rather than treating this article as a definitive reference for how to build good command-line applications, perhaps we can instead use it as a jumping-off point for future topics to cover in a more self-contained fashion. I'd love to hear your thoughts on what topics in particular interested you and what areas you think should have been covered in greater detail.

> NOTE: If you'd like to learn more about this topic, consider doing the Practicing Ruby self-guided course on [Streams, Files, and Sockets](https://practicingruby.com/articles/study-guide-1?u=dc2ab0f9bb). You've already completed one of its reading exercises by working through this article!
