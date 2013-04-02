*This article is written by James Edward Gray II.  James is an old friend of
Greg's, so he was thrilled to contribute. From late 2011 to mid 2012, James 
wrote his own series of programming articles called [Rubies in the Rough][rubies].
Yes, James just stole Greg's good idea.*

[rubies]: http://subinterest.com/rubies-in-the-rough

In this article, we will look at a fun problem that was in a couple of the [Peepcode][peepcode] _Play by Play_ videos. I've played around with this kata a bit and given it to a programming student of mine, so I know it pretty well by now. Its solution touches on a couple of neat programming topics, so dig in and see what you can learn.

[peepcode]: https://peepcode.com

### The challenge

I'm going to simplify the Peepcode task a bit so that we can get right to the heart of the problem. Here's the challenge we're going to work on:

> Write a method that accepts two arguments: an `Array` of five guesses for
> finalists in a race and an `Array` of the five actual finalists.  Each
> position in the lists matches a finishing position in the race, so first place
> corresponds to index `0`.  Return an `Integer` score of the predictions:  `0`
> or more points.  Correctly guessing first place is worth `15` points, second
> is worth `10`, and so on down with `5`, `3`, and `1` point for fifth
> place.  It's also worth `1` point to correctly guess a racer that finishes in
> the top five but to have that racer in the wrong position.

I'm going to jump right into solving this problem, but I encourage everyone to stop and play with the problem a little before reading on.  You'll get more out of what I say if you are familiar with the problem.

OK, ready?

### Complete specing

I test-drove my solution to this code, but probably not as everyone else does it.  Let me show you a trick I like to use for these fixed algorithms. First, let's set up some directories for the code and create the needed files:

```
$ mkdir -p scoring_predictions/{lib,spec}
$ cd scoring_predictions/
$ touch lib/race.rb
$ touch spec/scoring_spec.rb
```

At this point I opened `spec/scoring_spec.rb` in my editor and set to work.  We're supposed to begin with the happy path, so I wrote an example for a set of perfect guesses:

```ruby
require "race"

describe "Race::score" do
  let(:winners) { %w[First Second Third Fourth Fifth] }

  it "add points for each position" do
    Race.score(winners, winners).should eq(15 + 10 + 5 + 3 + 1)
  end
end
```

At this point, most developers would start adding the library code to make this example pass.  However, I don't find that approach very helpful for code like this.

If I do "the right thing," I should just return a hardcoded score.  Then I'll need to write a second example to force me to generalize (or consider the hardcoded score a violation of DRY that I need to refactor).  Either way, the tasks are just busywork that isn't helping me write the code.  Even having the extra example won't raise my confidence that it scores the scenarios correctly.

What would help me is to have an example for each rule in the challenge.  I'm going to need to do some programming to solve this—and nothing is getting me out of that.  All I can do is to make it easier to do that programming.  If the examples codify the rules for me, running them will tell me whether I am getting closer to a right answer just by watching the pass/fail ratio.

With these thoughts in mind, I finished writing examples for the rules of the challenge:

```ruby
require "race"

describe "Race::score" do
  let(:winners) { %w[First Second Third Fourth Fifth] }

  def correct_guesses(*indexes)
    winners.map.with_index { |w, i| indexes.include?(i) ? w : "Wrong" }
  end

  it "add points for each position" do
    Race.score(winners, winners).should eq(15 + 10 + 5 + 3 + 1)
  end

  it "gives 0 points for no correct guesses" do
    all_wrong = correct_guesses  # none correct
    Race.score(all_wrong, winners).should eq(0)
  end

  it "gives 15 points for first place" do
    Race.score(correct_guesses(0), winners).should eq(15)
  end

  it "gives 10 points for second place" do
    Race.score(correct_guesses(1), winners).should eq(10)
  end

  it "gives 5 points for third place" do
    Race.score(correct_guesses(2), winners).should eq(5)
  end

  it "gives 3 points for fourth place" do
    Race.score(correct_guesses(3), winners).should eq(3)
  end

  it "gives 1 point for fifth place" do
    Race.score(correct_guesses(4), winners).should eq(1)
  end

  it "gives one point for a correct guess in the wrong place" do
    guesses = correct_guesses(0)
    guesses.unshift(guesses.pop)  # shift positions by one
    Race.score(guesses, winners).should eq(1)
  end

  it "score positional and misplaced guesses at the same time" do
    guesses                = correct_guesses(0, 3)
    guesses[3], guesses[4] = guesses[4], guesses[3]
    Race.score(guesses, winners).should eq(15 + 1)
  end
end
```

This probably looks like a lot of code, but it's quite trivial.  You already saw the first example.  The next six just specify the score for each position (and one for no positions) with the help of a trivial method I wrote to generate right and wrong guesses.  The next-to-last example is the rule about right guesses in the wrong position.  Finally, I just wanted at least one example testing both scenarios at once.

This gives me plenty of red to work with:


```
$ rspec
FFFFFFFF

Failures:

…

Finished in 0.00417 seconds
9 examples, 9 failures

Failed examples:

rspec ./spec/scoring_spec.rb:8 # Race::score add points for each position
rspec ./spec/scoring_spec.rb:12 # Race::score gives 0 points for …
…
```

From there, I played around with an algorithm until I saw these examples go green.  I could show my process, but the truth is that we all attack this stuff in different ways.

Instead, let's look at a correct but not optimal solution.

### What the iterators can do for you

The first pass my student made at this problem landed on some code like this:

```ruby
module Race
  module_function

  def score(guesses, winners)
    points = 0
    guesses.each_with_index do |guess, i|
      if guess == winners[i]
        points += case i
                  when 0 then 15
                  when 1 then 10
                  when 2 then 5
                  when 3 then 3
                  when 4 then 1
                  end
      else
        winners.each do |winner|
          points += 1 if winner == guess
        end
      end
    end
    points
  end
end
```

The guy has only been studying Ruby a short while, so I thought this was a great first stab at the problem.  I did urge him to refine it, though.

First, I mentioned that you can often tell that you have the wrong iterator if it does extra iterations.  The `else` code in the previous example is a good example of this.  It may find the `guess` in the first position of `winners`, but it would keep looking.  Although it's possible to add a `break` statement to fix this problem, there are iterators that "short-circuit" when they find an answer.  For example, `find()`, which is close to what we want, or `any?()` which is even closer.  What we really want though, is this:

```ruby
module Race
  module_function

  def score(guesses, winners)
    points = 0
    guesses.each_with_index do |guess, i|
      if guess == winners[i]
        points += case i
                  when 0 then 15
                  when 1 then 10
                  when 2 then 5
                  when 3 then 3
                  when 4 then 1
                  end
      elsif winners.include? guess
        points += 1
      end
    end
    points
  end
end
```

Another sign that you're on the wrong track in Ruby is the need to track an index.  Sometimes you really do need one, but that need is quite rare.  Assume that you don't and give in only when you can't find a way around it.

In this case, the path is almost clear.  The first thing you see the index used for is to walk two lists in parallel.  Ruby has an iterator for that.  It's `zip()`.

Unfortunately, we can't switch straight to `zip()`.  If we did, we wouldn't have the score.  It also needs the index in this setup.  That's the problem we need to solve first.

The trick is that `case` statement.  It's really hiding the true nature of those scores.  If you squint hard enough, you'll see that it's really just another `Array`.  It would have been easier to see this if there were more of them (say, 100) because we would be less willing to type that out.

That gives us the first step.  We need to move to something more like this:

```ruby
module Race
  module_function

  def score(guesses, winners)
    points = 0
    guesses.each_with_index do |guess, i|
      if guess == winners[i]
        points += [15, 10, 5, 3, 1][i]
      elsif winners.include? guess
        points += 1
      end
    end
    points
  end
end
```

This code solves one of our problems.  We're now working with `Array` objects all the way down.  That's nice, but I don't really like that change I just made.  It makes it painfully obvious that it's a list of magic numbers.  That makes me want to give them a name:

```ruby
module Race
  SCORES    = [15, 10, 5, 3, 1]
  MISPLACED = 1

  module_function

  def score(guesses, winners, scores = SCORES, misplaced = MISPLACED)
    points = 0
    guesses.each_with_index do |guess, i|
      if guess == winners[i]
        points += scores[i]
      elsif winners.include? guess
        points += misplaced
      end
    end
    points
  end
end
```

That's much better, in my opinion.  The scores now have names.  They are in constants, so you can reflect on them externally.  This approach allows us to update the specs to use these values.  (I'll leave that work as an exercise for the interested reader.)  Finally, because we are passing the constants as defaults to method arguments, they can be overridden as needed, which ends their reign as magic values.

Of course, we took that step to get to this one:

```ruby
module Race
  SCORES    = [15, 10, 5, 3, 1]
  MISPLACED = 1

  module_function

  def score(guesses, winners, scores = SCORES, misplaced = MISPLACED)
    points = 0
    guesses.zip(winners, scores) do |guess, winner, score|
      if guess == winner
        points += score
      elsif winners.include? guess
        points += misplaced
      end
    end
    points
  end
end
```

The switch to `zip()` was straightforward and makes the code read better.  Plus, we're rid of that index.

This code is pretty close to the code I ended up with while fiddling with this problem.

### The point

I don't want to tell you what to get out of this exercise, but I can tell you what I got out of it, which is mainly to remember the true purpose of a thing.  For example:

* Following the proper steps of BDD is meant to **help you write code**.  If it turns into busywork that doesn't help, you are free to go another way.  And maybe you should feel compelled to go another way at that point.
* Iterators are intended to **save you from maintenance and potential error points**, such as:  tracking indexes or other variables and doing too much work.  If you find yourself in either of these scenarios, go spelunking in `Enumerable` to see whether there's a better tool for the job.  Heck, do that anyway... it's fun!  Do you know [what Enumerable#chunk() does][chunk] yet?
* The primary purpose of code is to **communicate with the reader.**  Period.  No, really!  Notice that in all of the steps in this article, I am trying to tease out the underlying meaning of the code, then write the code as close to that intention as possible.  That's when we're at our best, if you ask me.

[chunk]: http://ruby-doc.org/core-1.9.3/Enumerable.html#method-i-chunk

