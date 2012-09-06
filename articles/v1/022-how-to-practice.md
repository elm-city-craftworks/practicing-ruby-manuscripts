> **NOTE:** I generally remove temporal references in Practicing Ruby 
articles to keep them disconnected from a particular point in time, but
in this case I intentionally left them in because the post is written
in a sort of "diary" style. It was originally posted in January 2011.

In the [previous issue](http://practicingruby.com/articles/50), I provided a
series of questions and instructions that outlined the way I practice. While
some may have been expecting code katas or other indirect exercises, my style is
more geared towards learning on the job. You start by figuring out what's
important to you, figure out a baby-step that you can make, and then execute.
Once you've brought yourself one step closer to your goal, you reflect a bit on
how things are going, in particular, what parts of the project still scare you.

I use this technique often, in particular, when I want to get started on a new
project or explore a new area that I'm not that familiar with. As luck would
have it, I actually have a new project I need to start, and so today I'll give
you the chance to metaphorically look over my shoulder as I work through this
exercise myself. If you haven't read [Issue
#21](http://practicingruby.com/articles/50), now is a good time to do that, so
that the rest of this article makes sense.

### Step 1: Find out what's important 

What interesting problems do you need to solve?

Lately, this question has been one that has caused me great anxiety.The success
of Mendicant University and even Practicing Ruby to a certain extent has caused
an explosion of ideas that all feel worthwhile and important to me. But they're
fairly easy to separate into wants and needs, and thankfully, much of what I've
come up with falls in the former category.

When I think about it, there is something that I feel I need to do sooner rather
than later. While our courses are booked up until May, we will need to start
admissions for our second trimester soon. Towards the end of last year, we
decided we wanted to do something more fun and lighthearted than an
entrance exam, but we didn't take much action since then. With the clock
ticking down, making headway on this project would surely help give me some
peace of mind.

What I'd like to build is a programming quiz site that is inspired by the
[Internet Problem Solving Contest](http://ipsc.ksp.sk) and [Project
Euler](http://projecteuler.net), but with an MU-themed twist. I'll have
Mendicant's co-founder [Jordan Byron](http://twitter.com/Jordan_Byron) to help
me with the frontend, but since he's busy with 100 other tasks for
MU, I'm the one who needs to build out the backend for this new app. I'll use my
need to write this article as motivation to help me break ground on this new
project today.

### Step 2: Make a commitment

I made the broad commitment to our students that we'd have a nice replacement
for MU's currently dull admissions process before the next trimester began. But
broad commitments don't particularly inspire action, so I needed to make a
specific commitment as well.

With that in mind, I told Jordan I'd have something for him to look
at today, even if it was just a small start. Since he'll be arriving at my
home within an hour of me finishing this article, I am already feeling the
pressure of having something to show for myself, which is a good thing.

### Step 3: Identify a baby step

The next step in this process is to come up with a small step to get you just a
little bit closer to your goal. I knew by the time I finished Issue
#21 that I'd be working on this project, so I've been subconsciously chewing on
my baby-step for a couple days now. This to me is totally fine, it gives my
brain a chance to think things through and makes actually sitting down and
coding something easier. Of course, the key thing is that my delivery time was
still boxed in. If you leave things open ended, you may end up
talking yourself out of building anything at all.

When coming up with a tiny step, I try to focus on something that is core to the
underlying project, to maximize the amount I learn from the mini spike. In the
case of our quiz application (which we're calling PuzzleNode), validating user
submissions is one of the most important pieces of functionality.

What I'll do today is do a rough proof of concept of the submission validation
system, which compares the expected output to the actual file
uploaded by a user. I like to subdivide my tasks even when working
only for an hour, so I'm going to attack this in three phases:

1. A simple function that compares two files using a SHA1 hash and returns true
or false depending on whether they match.

1. A tiny sinatra application that does the same, but introduces file uploads
into the picture.

1. A minimal Rails app that actually records whether a submission was valid or
invalid, and properly links puzzles with their expected output.

I'm setting my time limit for an hour, so I'm not sure how far I'll actually
get. No matter what happens, I'll try to jot down some notes to give you a feel
for my though process as I work through this exercise.

### Step 4: Get one step closer

[06:40] I've got my clock set now, and I'm ready to get started. Please excuse
me while I go heads down for a bit. I'll pop up with some brief notes here each
time I reach a transition point, and then go into more detail in the reflections
phase.

[06:45] Basic [github project](https://github.com/sandal/pr-issue-22) set up for
this experiment.

[06:48] Add three text files, a reference which is meant to act as the expected
solution, a good file which is just a copy of the reference, and a bad file with
some modifications to make it not match the reference.

[06:51] Phase 1 complete!

```ruby
$ ruby check_solution.rb samples/reference.txt samples/good.txt 
GOOD

$ ruby check_solution.rb samples/reference.txt samples/bad.txt
BAD
```

Source code is dead simple, just a few lines.

```ruby
require "digest/sha1"

expected = Digest::SHA1.hexdigest(File.read(ARGV[0]))
actual   = Digest::SHA1.hexdigest(File.read(ARGV[1]))

puts(expected == actual ? "GOOD" : "BAD")
```

[06:54] Next step is to remind myself how file uploads work in Sinatra, an
indicator of how rusty my frontend webdev knowledge is...

[06:57] Google for "File uploads sinatra" and find Peter Cooper talking about
[this blog
post](http://technotales.wordpress.com/2008/03/05/sinatra-the-simplest-thing-that-could-possibly-work/)
via Ruby Inside.

Outdated, but worth a shot since it's just a one liner.

[07:00] File uploads working via curl. Time to integrate the phase 1 code
into my sinata app.

[07:06] Have something I think should work but found some unexpected
bugs. Drat!

[07:08] Oh, apparently I just don't know how to use curl, working now! (albiet
with a little echo hack to add a newline)

```ruby
$ curl -F "data=@samples/bad.txt" 127.0.0.1:4567/reference.txt; echo
BAD
$ curl -F "data=@samples/good.txt" 127.0.0.1:4567/reference.txt; echo
GOOD
```

Source is still quite simple, so I can inline it here.

```ruby
require "rubygems"
require "sinatra"
require "digest/sha1"

ACCEPTED_FILES = ["reference.txt"]

post "/:expected" do
  raise unless ACCEPTED_FILES.include?(params[:expected])

  expected = 
    Digest::SHA1.hexdigest(File.read("samples/#{params[:expected]}"))

  actual   = Digest::SHA1.hexdigest(params[:data][:tempfile].read)

  expected == actual ? "GOOD" : "BAD"
end
```

Off we go to phase 3!

[07:11] I need to think up a few AR models. Off to the whiteboard, back in a
moment.

[07:15] I've decided to cheat a bit. I realized that for a very basic demo, I
don't actually need to store the uploaded files anywhere, but instead, I just
need each puzzle to store its SHA1 fingerprint. Then, when a new submission is
made, you just hash the file uploaded by the user and compare it to the
associated puzzle.

This data model omits a lot, and would need a lot of love to actually be used in
our application, but it is sufficient for demonstrating just the validation
step.

```
Puzzle(name: text, fingerprint: text) 
Submission(puzzle_id: integer, correct: boolean)
```

Time to go spit out a Rails skeleton, I suppose. The key thing this has saved me
is a trip through paperclip's documentation, and a host of questions about
whether that's still the right tool for the job and whether it works with Rails
3 smoothly. I roughly assume that the answer to each of those questions is yes,
but better to not have to answer them right now.

[07:22] Only 18 minutes to go and rails is still installing, sloooooow.

[07:24] Still installing! Should have used --no-rdoc --no-ri!

[07:25] Finally finished installing, while waiting I stumbled upon [this post on
disabling documentation by
default](http://stackoverflow.com/questions/1381725/how-to-make-no-ri-no-rdoc-default-for-gem-install).
Will need to try that out later.

[07:26] Doh, never going to undo my stupid muscle memory

$ rails puzzlenode
Usage:
  rails new APP_PATH [options]

[07:29] Hmm, rails comes with a .gitignore file now? That's handy. Though I'm
pretty sure I just accidentally checked in my config/database.yml. Not
a big deal, this is just a spike, right?

[07:30] Wow, now is not the time to be punished by the fact that I aliased mvim
to sl on my Gentoo box in an effort to stop typing mvim where it doesn't work.
That seemed like a good idea at the time, of course.

[07:32] Toot toot! Time to switch consoles, this is taking forever. Dear reader,
you *have* googled sl by now, right? :)

[07:39] Ran out of time, so just messed with the data models a bit in the
console to imagine their interactions. Will need to save a proper implementation
for later.

```ruby
>> Puzzle.create(:name => "Reference", :fingerprint =>
>> Digest::SHA1.hexdigest(File.read("#{RAILS_ROOT}/samples/reference.txt")))

=> #<Puzzle id: 1, name: "Reference", fingerprint:
"a59eb2c51e07e2b7369baef8a0c3cb3b5d7ed3d9", created_at: "2011-01-28
12:37:41", updated_at: "2011-01-28 12:37:41">

>> Submission.create(:puzzle_id => 1, :correct => false)

=> #<Submission id: 1, puzzle_id: 1, correct: false, 
     created_at: "2011-01-28 12:38:41", updated_at: "2011-01-28 12:38:41">
>> Submission.create(:puzzle_id => 1, :correct => false)

=> #<Submission id: 2, puzzle_id: 1, correct: false, 
   created_at: "2011-01-28 12:38:42", updated_at: "2011-01-28 12:38:42">

>> Submission.create(:puzzle_id => 1, :correct => true)

=> #<Submission id: 3, puzzle_id: 1, correct: true, 
   created_at: "2011-01-28 12:38:45", updated_at: "2011-01-28 12:38:45">

>> Puzzle.find(1).submissions.where(:correct => true).count
=> 1
>> Puzzle.find(1).submissions.where(:correct => false).count
=> 2
```

Hah, at least my associations seem to be working correctly. I can has rails!

### Step 5: Reflect on your progress

This exercise went more or less as I expected it to, with a couple surprises
here and there. One thing that didn't dawn on me until I reached stage 3 is that
I don't necessarily need to worry about file attachments in this application.
While certain features such as having the ability to review user submissions or
display the reference output would require it, a simple alpha product could be
shipped without those features and still be quite usable.

The exercise hopefully also reflects a bit of realism, as I didn't rehearse it
ahead of time and ran into some stupid things that slowed me down, which is what
might happen to anyone. That's really okay, because in the process, I learned
some things worth looking into later on.

Now that I'm an hour into this project, my instructions call for me to reflect
on what scares me about it. I actually have a lot of general fears, but in order
to explain them I'd need to give a lot of context about the project and those
ideas are still fuzzy even in my own mind. That having been said, there is a 
concern that I can share which this small spike keeps reminding me of.

### What about this project scares me?

I'm not sure that I like fingerprinting as a method for
determining the validity of a solution. It scares me to think that if a problem
called for you to generate some XML, alterations to whitespace could
result in an otherwise perfectly valid submission getting rejected.

The way that IPSC and Project Euler solve this problem is by restricting
the submission format. In the case of IPSC this consists of bits of numbers or
text separated by newlines, and for Project Euler the solutions are always to
compute a simple number. I could adopt this strategy, but it makes me
worried that it'll limit the kinds of problems I can run at PuzzleNode.

I want to avoid making the problems at PuzzleNode too academic in
nature, with a focus more on practical problem solving and creative thinking.
Both Project Euler and IPSC do a good job of this within a subset of their
problems, but most of them are algorithmic. I wonder if that's due to the 
input constraints, and if it is, that would be bad for MU.

One possibility is that rather than doing a bitwise matching via a fingerprint,
I could force users to provide JSON data which I could then process and compare
based on the object structure. This would allow for much greater flexibility in
the way I validate submissions, and eliminate the failure-by-formatting issue I
pointed out before, but it'd both increase the overhead of submitting a solution
and make the backend functionality a good deal more complex.

I think that what I need to do is draft up a few puzzles and see how much the
current fingerprinting validation restrictions get in the way. I may be worrying
about nothing, but the only way to tell is to produce some content and see where
that brings me.

### Step 6: Rinse and Repeat

My next step is to actually flesh out the Rails backend, since I didn't get that
far with it. I'm glad to have found that I can defer file uploads until a bit
later, this is something I don't think I would have realized if I started
directly by jumping into the Rails boilerplate.

Once I have a minimal system functioning, my next step will be to come up with
a few more problems to test it against. I already have one idea in mind;
generating more should be easy.

With my next step planned, I feel confident that this project will keep 
moving forward.

### Closing Thoughts

This is a real outline of how I practice. At first, when I wrote up the set of
instructions, I thought formalizing it would make it feel artificial to me. But
honestly, once I got rolling on the spike, things happened pretty much the way
they always would, and the comments I left were just the thoughts that came up
in my mind as I went along. In that sense, it didn't feel like practice.

You'll notice that I start with what I know and work outwards from there. I
rarely try to think too hard about what I need to know ahead of time, because I
find it causes me to study the wrong things at the wrong time. A more formal
approach might have lead me to study paperclip up front, because this process
involves file uploads. But the 20-30 minutes that might have costed me we found
through experimentation is something that I can put off for several weeks
without it affecting my progress.

I tend not to plug into the firehose of information coming from books, blogs,
and reddit/HN for the same reason. Soaking up that material is begging to find a
solution in search of a problem, rather than the other way around. It's always
easy to ask for a recommendation at the time you actually need something, and
Google is pretty good at digging up well read blog posts or articles about
whatever tool you might need, and so I put off studying until it is necessary.

I don't do a whole lot of code katas, or little practice exercises that I can't
actually use for something. I will certainly do those things for entertainment,
but I don't schedule 'practice time' in my day to day life and honestly, I never
have. There is no shortage of necessary learning that takes place when chasing
practical goals, and the reward is much greater than just having an abstract
feeling of learning a bit more, you end up with something you can use.

The more I can make my life my practice, the less I need to be disciplined about
making time for formal academic exercises. I admit that there are
a lot of things about my lifestyle and circumstances that make me especially
blessed, but I wasn't always in a fortunate position and would give 
similar advice even when I was struggling to make ends meet.

So in closing, it may be true that the way to Carnegie Hall is via the
"Practice! Practice! Practice!" path, but in my mind, that means less time
playing with yourself in the comfort of your own home, and more time on small
stages until they lead you to a slightly bigger stage which you can then occupy
until it too, becomes too small.

This is how I practice. I hope hearing about it has been useful to you.

<b>UPDATE 2011.09.09</b>: <i> The [PuzzleNode website](http://puzzlenode.com)
was successfully launched on time, and has been used to conduct three entrance
exams for Mendicant University already. The puzzles there are language agnostic,
and may be fun to try out even if you aren't planning to apply to MU. But I'd be
just as happy to hear that you're too busy working on real projects that you
care a lot about instead.</i>
 
> **NOTE:** This article has also been published on the Ruby Best Practices blog. 
There [may be additional commentary](http://blog.rubybestpractices.com/posts/gregory/054-issue-22-how-to-practice.html#disqus_thread) 
over there worth taking a look at.
