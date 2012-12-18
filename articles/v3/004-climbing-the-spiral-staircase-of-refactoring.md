In [Issue 3.3](http://practicingruby.com/articles/shared/bwgflabwncjv), I presented a proof-of-concept Ruby implementation of the [Brainfuck programming language](http://en.wikipedia.org/wiki/Brainfuck) and challenged Practicing Ruby readers to improve upon it. After receiving several patches that helped move things along, I sat down once again to clean up the code even further. What I came to realize as I worked on my revisions is that the refactoring process is very similar to climbing a spiral staircase. Each structural change to the code simultaneously left the project back where it started along one vector while moving it forward along another.

Because we often look at the merits of a given refactoring technique within the context of a single transition from worse code to better code, it's easy to mistakenly assume that the refactoring process is much more linear than it actually is. In this article, I've tried to capture a much wider angle view of how refactoring really works in the wild. The end result is a story which I hope will spark some good discussions about how we can improve our code quality over time.

### Prologue. Everything has to start somewhere

I decided to name my interpreter [Turing Tarpit](http://en.wikipedia.org/wiki/Turing_tarpit), because that term is perfectly apt for describing languages like Brainfuck. In a nutshell, the term refers to any language which is infinitely flexible, yet nearly impossible to use for anything practical. It turns out that building this sort of mind trap for programmers is quite easy to do.

My first iteration was easy enough to build, and consisted of three objects: a `Tape`, an `Interpreter`, and a `Scanner`. The rough breakdown of responsibilities was something like this:

* The [Tape object](https://github.com/elm-city-craftworks/turing_tarpit/blob/starting_point/lib/turing_tarpit.rb#L103-149) implemented something similar to the storage mechanism in a [Turing machine](http://en.wikipedia.org/wiki/Turing_machine#Informal_description). It provided mechanisms for accessing and modifying numeric values in cells, as well as a way to increment and decrement the pointer that determined which cell to operate on.

* The [Interpreter object](https://github.com/elm-city-craftworks/turing_tarpit/blob/starting_point/lib/turing_tarpit.rb#L7-34) served as a mapping between Brainfuck's symbolic operators and the operations provided by the `Tape` object. It also implemented the I/O functionality required by Brainfuck.

* The [Scanner object](https://github.com/elm-city-craftworks/turing_tarpit/blob/starting_point/lib/turing_tarpit.rb#L36-69) was responsible for taking a Brainfuck source file as input and transforming it into a stream of operations that could be handled by the `Interpreter` object. For the most part this simply meant reading the source file one character at a time, but this object also needed to account for Brainfuck's forward and backward jump operations.

While my initial implementation was reasonably clean for a proof-of-concept, it definitely had room for improvement. I decided to ask for feedback early in the hopes that folks would find and fix the things I knew were problematic while simultaneously checking my blindspots for issues that I hadn't noticed myself.

### Act I. Getting a fresh perspective on the problem

Some of the issues brought up by contributors were fairly obvious housekeeping chores, but nonetheless made the project nicer to work with:

* Steve Klabnik [requested a way to run the whole test suite at once](https://github.com/elm-city-craftworks/turing_tarpit/pull/3) instead of file by file. He had provided a patch with a Rakefile, but since the project didn't have any immediate need for other rake tasks, we ended up deciding that a simple _test/suite.rb_ file would be sufficient. Notes were added to the README on how to run the tests.

* Renato Riccieri [broke the classes out into individual files](https://github.com/elm-city-craftworks/turing_tarpit/pull/6). The original implementation had everything in _lib/turing_tarpit.rb_, simply for convenience reasons while spiking. Breaking the classes into individual files brought the project more in line with [standard Ruby packaging conventions](http://chneukirchen.github.com/rps/).

* Benoit Daloze [refactored some ugly output code](https://github.com/elm-city-craftworks/turing_tarpit/pull/2) to use `putc(char)` instead of `print("" << char)`. Since the latter was obviously a hack due to my lack of awareness of the `putc` method, this was a welcome contribution.

After this initial round of cleanup, we ended up thinking through a pair of more substantial problems: the inconsitent use of private accessors, and a proposed refactoring to break up the `Scanner` object into two separate objects, a `Tokenizer` and a `Scanner`.

**The story behind my recent private accessor experiments**

Ryan LeCompte was the one to bring up [the question about private accessors](https://github.com/elm-city-craftworks/turing_tarpit/issues/1), and was curious about why I had used them in some places but referenced instance variables directly in others. The main reason for this was simply that the use of private accessors is a new experiment for me, and so in my haste of getting a first version out the door, I remembered to use them in some places but not in others.

This project in particular posed certain challenges for using private accessors conveniently. A specific example of where I ran into some weird edge cases can easily be seen in the `Tape` object:

```ruby
module TuringTarpit
  class Tape
    def initialize
      self.pointer_position = 0
      # ...
    end

    def increment_pointer
      self.pointer_position = pointer_position + 1
    end

    # ...

    private

    attr_writer :pointer_position
  end
end
```

If you just glance quickly at this class definition, it is very tempting to try to refactor `increment_pointer` so that it uses convenient `+=` syntax, resulting in something like the code below:

```ruby
def increment_pointer
  self.pointer_position += 1
end
```

In most cases, this refactoring would be a good one because it makes the code slightly less verbose without sacrificing readability. However, it turns out that Ruby does not extend the same private method special casing to `self.foo += something` as it does to `self.foo = something`. This means that if you attempt to refactor this code to use `+=` it ends up raising a `NoMethodError`. Because this is definitely a downside of using private accessors, it's reasonable to ask why you'd bother to use them in the first place rather than using public accessors or simply referring to instance variables directly.

The best reason I can find for making use of accessors in general vs. instance variables is simply that the former are much more flexible. New behavior such as validations or transformations can be added later by changing what used to be vanilla accessors into ordinary method definitions. Additionally, if you accidentally introduce a typo into your code, you will get a `NoMethodError` right away rather than having to track down why your attribute is `nil` when you didn't expect it to be in some completely different place in your code.

The problem with making accessors public is that it hints to the consumer that it is meant to be touched and used, which is often not the case at all, especially for writers. While Ruby makes it trivial to circumvent privacy protections, a private method communicates to the user that it is meant to be treated as an implementation detail and should not be depended on. So the reason for using a private accessor is the same as the reason for using a private method: to mark the accessor as part of the internals of the object.

The interesting thing I stumbled across in this particular project is that if you take this technique to the extreme, it is possible to build entire applications without ever explicitly referencing an instance variable. It comes at the cost of the occasional weird edge case when calling private methods internally, but makes it possible to treat instance variables as a whole as a _language implementation detail_, rather than an _application implementation detail_. Faced with the opportunity to at least experiment with that idea, I decided to make the entire Turing Tarpit codebase completely free of instance variables, which ended up taking very little effort.

The jury is still out on whether or not this is a good idea, but I plan to keep trying the idea out in my projects and see whether I run into any more issues. If I don't experience problems, I'd say this technique is well worth it because it emphasizes message-passing rather than state manipulation in our objects. 

**Splitting up the Scanner object**

After helping out with a few of the general housekeeping chores, Steve Klabnik then turned his attention to one of the weakest spots in the code, the `Scanner` object. He pointed out that having an object with dependencies on a whole lot of private methods is a bit of a code smell, and focused specifically on the `Scanner#next` method. The original implementation looked like this:

```ruby
module TuringTarpit
  class Scanner
    # ...

    def next(cell_value)
      validate_index

      element = @chars[@index]
      
      case element
      when "["
        jump_forward if cell_value.zero?

        consume
        element = @chars[@index]
      when "]"
        if cell_value.zero?
          while element == "]"
            consume
            element = @chars[@index]
            validate_index
          end
        else
          jump_back
          consume
          element = @chars[@index]
        end
      end
      
      consume
      element
    end
  end
end
```

Steve pointed out that the `Scanner#next` method was really doing more of a tokenizing operation, and that most of the scanning work was actually being done by the various private methods that were being used to traverse the underlying string. He prepared a patch which made this relationship explicit by introducing a `Tokenizer` object which would provide a method to replace `Scanner#next`. His newly introduced object allowed for a re-purposing of the `Scanner` object which allowed its methods to become public:

```ruby
module TuringTarpit
  class Tokenizer
    # ...

    def next(cell_value)
      scanner.validate_index

      element = scanner.current_char

      case element
      when "["
        scanner.jump_forward if cell_value.zero?

        scanner.consume
        element = scanner.current_char
      when "]"
        if cell_value.zero?
          while element == "]"
            scanner.consume
            element = scanner.current_char
            scanner.validate_index
          end
        else
          scanner.jump_back
          scanner.consume
          element = scanner.current_char
        end
      end

      scanner.consume
      element
    end
  end
end
```

The thing in particular I liked about this patch is that it abstracted away some of the tedious index operations that were originally present in `Scanner#next`. As much as possible I prefer to isolate anything that can cause off-by-one errors or other such nonsense, and this refactoring did a good job of addressing that issue.

The interesting thing about this refactoring is that while I intended to work on the same area of the code if no one else patched it, I had planned to approach it in a very different way. My original idea was to implement some sort of generic stream datastructure and reuse it in both `Scanner` and `Tape`. However, seeing that Steve's patch at least partly addressed my concerns while possibly opening some new avenues as well, I abandoned that idea and merged his work instead.

### Act II. Building a better horse

After applying the various patches from the folks who participated in this challenge, the code was in a much better place than where it started. However, much work was still left to be done!

In particular, the code responsible for turning Brainfuck syntax into a stream of operations still needed a lot of work. The `Tokenizer` class that Steve introduced was an improvement, but without further revisions would simply serve as a layer of indirection rather than as an abstraction. Zed Shaw describes the difference between these two concepts very eloquently in his essay [Indirection Is Not Abstraction](http://zedshaw.com/essays/indirection_is_not_abstraction.html) by stating that _"Abstraction is used to reduce complexity. Indirection is used to reduce coupling or dependence."_

As far as the `Tokenizer` object goes, Steve's patch reducing coupling somewhat by pushing some of the implementation details down into the `Scanner` object. However, the procedure is pretty much identical with the exception of the lack of explicit indexing code, and so the baseline complexity actually increases because what was once done by one object is now split across two objects.

To address this problem, the dividing lines between the two objects needed to be leveraged so that they could interact with each other at a higher level. It took me a while to think through the problem, but in doing so I realized that I could now push more functionality down into the `Scanner` object so that `Tokenizer#next` ended up with fewer moving parts. After some major gutting and re-arranging, I ended up with a method that looked like this:

```ruby
module TuringTarpit
  class Tokenizer
    # ...

    def next(cell_value)
      case scanner.next_char
      when Scanner::FORWARD_JUMP
        if cell_value.zero?
          scanner.jump_forward
        else
          scanner.next_char
        end
      when Scanner::BACKWARD_JUMP
        if cell_value.zero?
          scanner.skip_while(Scanner::BACKWARD_JUMP)
        else
          scanner.jump_back
        end
      end

      scanner.current_char
    end
  end
end
```

After this refactoring, the `Tokenizer#next` method was a good deal more abstract in a number of ways:

* It expected the `Scanner` to handle validations itself rather than telling it when to check the index 

* It no longer referenced Brainfuck syntax and instead used constants provided by the `Scanner`

* It eliminated a lot of cumbersome assignments by reworking its algorithm so that `Scanner#current_char` always referenced the right character at the end of the scanning routine.

* It expected the `Scanner` to remain internally consistent, rather than handling edge cases itself.

These reductions in complexity made a hugely positive impact on the readability and understandability of the `Tokenizer#next` method. While all of these changes could have technically been made before the split between the `Scanner` and `Tokenizer` happened, cutting the knot into two pieces certainly made untangling things easier. This is why indirection and abstraction often go hand in hand, despite the fact that they are very different concepts from one another.

### Act III. Mountains are once again merely mountains

After building on top of Steve's work to simplify the syntax-processing code even further, I finally felt like that part of the project was in decent shape. I then decided to turn my attention back to the `Interpreter` object, since it had not received any love from the challenge participants. The original code for it looked something like this:

```ruby
module TuringTarpit
  class Interpreter
    def run
      loop do
        case tokenizer.next(tape.cell_value)
        when "+"
          tape.increment_cell_value
        when "-"
          tape.decrement_cell_value
        when ">"
          tape.increment_pointer
        when "<"
          tape.decrement_pointer
        when "."
          putc(tape.cell_value)
        when ","
          value = STDIN.getch.bytes.first
          next if value.zero?

          tape.cell_value = value
        end
      end
    end
  end
end
```

While this implementation wasn't too bad, there were two things I didn't like about it. The first issue was that it directly referenced Brainfuck syntax, which sort of defeated the purpose of having the tokenizer be syntax independent. The second problem was that I found the case statement to feel a bit brittle and limiting. What I really wanted was a dynamic dispatcher similar to the following method:

```ruby
def run
  loop do
    if operation = tokenizer.next(evaluator.cell_value)
      tape.send(operation)
    end
  end
end
```

In order to introduce this kind of functionality, I'd need to find a place to introduce a simple mapping from Brainfuck syntax to operation names. I already had the keys and values in mind, I just needed to find a place to put them:

```ruby
OPERATIONS = { "+" => :increment_cell_value,
               "-" => :decrement_cell_value,
               ">" => :increment_pointer,
               "<" => :decrement_pointer,
               "." => :output_cell_value,
               "," => :input_cell_value }
```

Figuring out how to make this work was surprisingly challenging. I found that the extra layers of indirection between the `Tape` and the `Scanner` meant that any change made too far down the chain would need to be echoed all the way up it, and that changes made towards the top felt tacked on and out of place. This eventually led me to question what the separation between `Scanner` and `Tokenizer` was really gaining me, as well as the separation between `Interpreter` and `Tape`.

After a fair amount of ruminating, I decided to take my four objects and join them together at the seams so that only two remained. The `Scanner` and `Tokenizer` ended up getting joined back together to form a new `Interpreter` class. The job of the `Interpreter` is to take Brainfuck syntax and turn it into a stream of operations. You can get a rough idea of how it all came together by checking out the following code:

```ruby
module TuringTarpit
  class Interpreter
    FORWARD_JUMP = "["
    BACKWARD_JUMP = "]"

    OPERATIONS = { "+" => :increment_cell_value,
                   "-" => :decrement_cell_value,
                   ">" => :increment_pointer,
                   "<" => :decrement_pointer,
                   "." => :output_cell_value,
                   "," => :input_cell_value }

    def next_operation(cell_value)
      case next_char
      when FORWARD_JUMP
        if cell_value.zero?
          jump_forward
        else
          skip_while(FORWARD_JUMP)
        end
      when BACKWARD_JUMP
        if cell_value.zero?
          skip_while(BACKWARD_JUMP)
        else
          jump_back
        end
      end

      OPERATIONS[current_char]
    end

    # ... lots of private methods are back, but now fine-tuned.
  end
end
```

The old `Interpreter` object and `Tape` object were also merged together, forming a single object I ended up calling `Evaluator`. The job of the `Evaluator` object is to take a stream of operations provided by the newly defined `Interpreter` object and then execute them against a Turing Machine like data structure. In essence, the `Evaluator` object is nothing more than the original `Tape` object I implemented along with a few extra methods which account for the things the original `Interpreter` object was meant to do:

```ruby
module TuringTarpit
  class Evaluator 
    def self.run(interpreter)
      evaluator = new

      loop do
        if operation = interpreter.next_operation(evaluator.cell_value)
          evaluator.send(operation)
        end
      end
    end

    def output_cell_value
      putc(cell_value)
    end

    def input_cell_value
      value = $stdin.getch.ord
      return if value.zero?

      self.cell_value = value
    end

    # other methods same as original Tape methods
  end
end
```

I had mixed feelings about recombining these objects, because to some extent it felt like a step backwards to me. In particular, I think this refactoring resulted in some minor violations of the [Single Responsibility Principle](http://en.wikipedia.org/wiki/Single_responsibility_principle), and increased the overall coupling of the system somewhat. However, the independence of the four different objects the system previously consisted of seemed artificial at best. To the extent that they could be changed easily or swapped out for one another, I could not think of a single practical reason why I'd actually want that kind of flexibility. In this particular situation it turned out that recombining the objects greatly reduced their communications overhead, and so was worth the loss in generality.

### Epilogue. Sending the ship out to sea

I was really tempted to keep noodling on the design of this project, because even in my final version of the code I still felt that I could have done better. But at a certain point I decided that I could end up getting caught in this trap forever, and the only way to free myself from it was to wrap up my work and just ship the damn thing. This ultimately meant that I had to take care of several chores that neither I nor the various participants in this challenge bothered to work on earlier:

* I added a [set of integration tests](https://github.com/elm-city-craftworks/turing_tarpit/blob/act3/test/integration/evaluator_test.rb) which ran the `Evaluator` against a couple sample Brainfuck programs to make sure we had some decent end-to-end testing support. Found a couple bugs that way.

* I set up and ran [simplecov](https://github.com/colszowka/simplecov) to check whether my tests were at least *running* all the implementation code, and ended up spotting a faulty test which wasn't actually getting run.

* I added a [bin/turing_tarpit](https://github.com/elm-city-craftworks/turing_tarpit/blob/act3/bin/turing_tarpit) file so that you can execute Brainfuck programs without building a Ruby shim first. 

* Did the usual gemspec + Gemfile dance and pushed a 1.0.0 gem to rubygems.org. Typically I'd call a project in its early stages a 0.1.0 release, but I honestly don't see myself working on this much more so I might as well call it 'production ready'.

After I wrapped up all these chores, I decided to go back and check out what my [flog](https://github.com/seattlerb/flog) complexity scores were for each stage in this process. It turns out that the final version was the least complex, with the lowest overall score, lowest average score, and lowest top-score by method. The original implementation came in second place, and the other two iterations were in a distant third and fourth place. While that gave me some reassurances, it doesn't mean much except for that Flog seems to really hate external method calls.

### Reflections

This has been one of my favorite articles to write for Practicing Ruby so far. It forced me to look at the refactoring process in a much more introspective way than I have typically done in the past, and gave me a chance to interact with some of our awesome readers. I do think it ended up raising more questions and challenges in my mind than it did give me answers and reassurances, but I suppose that's a sign that learning happened.

While I found it very hard to summarize the refactoring lifecycle for this project, my hope is that I've at least given you a glimpse of the spiral staircase metaphor I chose to name this article after. If it didn't end up making you feel too dizzy, I'd love to hear your thoughts about this exercise as well as what your own process is like when it comes to refactoring code.
