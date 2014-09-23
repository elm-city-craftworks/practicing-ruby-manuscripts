[ ] Day 1 review
[ ] Day 2 review
[ ] Day 3 review
[ ] Day 4 review
[ ] Day 5 review
[ ] Explain four step system and projects
[ ] Add explanations of code
[ ] Connect togther narrative


----------

http://alistapart.com/article/writing-is-thinking

First of all, why did this excerpt from your experience stand out to you, personally? 
Was this the moment something clicked for you regarding your work?

Secondly, why do you think things turned out the way they did? Were you surprised? 
Do you do things differently now as a result? When you spell this out, 
it’s the difference between journaling for yourself and writing for an audience.

Finally, is this something others in your line of work are prone to miss? 
Is it a rookie error, or something more like an industry-wide oversight? 
If you’ve tried to search online for similar opinions, do you get a lot 
of misinformation? Or is the good information simply not in a place
where others in your field are likely to see it?

----------

**Summarize my preliminaries and the four step structure, then make
case study presentation chronological but free-form narrative 
(i.e. don't put in clear headers for exercise, book, project, etc, make it 
more like a journal -- give the reader a 'riding shotgun' view of the
action, with a brief summary at the end of each day. Include evolution
in thought process and new realizations as I go. Wrap up with a where
to next section.**

** Look up dates for everything to linearize it, but don't be afraid to do mild
editing / resequencing for clarity**

## What is trivial code literacy?

**Starting with fizzbuzz may work, but consider condensing or replacing with a
shorter anecdote (the chinese one?)**

(rewrite to be a bit more positive, and to illustrate
Ruby vs. Erlang, and Reading vs. Writing)

Programmers often joke about how ridiculous it is to use the FizzBuzz problem as
a screening test, because it is so easy to solve. Derived from a word
game that's used to test the division skills of small children, FizzBuzz is
about as conceptually trivial as computing challenges get:

> Write a program that prints the numbers from 1 to 100. But for multiples 
of three print “Fizz” instead of the number and for the multiples of 
five print “Buzz”. For numbers which are multiples of both three and 
five print “FizzBuzz”.

Any working programmer or hobbyist that's built absolutely any software could
solve this problem in their sleep, as long as they were allowed to use a language 
they were already comfortable with. But suppose the interviewer asked a Ruby
programmer who had never worked in a functional programming language before to 
produce an Erlang solution instead. What would that programmer need to learn in
order to pass the test? Let's figure that out by working backwards from the 
following solution:

```erlang
-module(fizzbuzz).
-export([run/0]).

run() -> 
  io:format("~p~n", [lists:map(fun transform/1, lists:seq(1,100))]).

transform(X) when X rem 15 =:= 0 -> "FizzBuzz";
transform(X) when X rem 5  =:= 0  -> "Fizz";
transform(X) when X rem 3  =:= 0  -> "Buzz";
transform(X) -> X.
```

Let's assume that simply producing the source code shown above wouldn't be good
enough to pass the test, the programmer would actually need to execute it and
show that it produces the correct output, too. Right out of the gate, that means
understanding how to install Erlang, compile Erlang modules, and then call
functions on them. If that was all done successfully, the programmer might
produce something like the following output:

```erlang
$ erl
1> c(fizzbuzz).
{ok,fizzbuzz}
2> fizzbuzz:run().
[1,2,"Buzz",4,"Fizz","Buzz",7,8,"Buzz","Fizz",11,"Buzz",13,14,"Fizzbuzz",16,
 17,"Buzz",19,"Fizz","Buzz",22,23,"Buzz","Fizz",26,"Buzz",28,29,"Fizzbuzz",31,
 32,"Buzz",34,"Fizz","Buzz",37,38,"Buzz","Fizz",41,"Buzz",43,44,"Fizzbuzz",46,
 47,"Buzz",49,"Fizz","Buzz",52,53,"Buzz","Fizz",56,"Buzz",58,59,"Fizzbuzz",61,
 62,"Buzz",64,"Fizz","Buzz",67,68,"Buzz","Fizz",71,"Buzz",73,74,"Fizzbuzz",76,
 77,"Buzz",79,"Fizz","Buzz",82,83,"Buzz","Fizz",86,"Buzz",88,89,"Fizzbuzz",91,
 92,"Buzz",94,"Fizz","Buzz",97,98,"Buzz","Fizz"]
```

But imagine the interviewer was not convinced by this alone, and wanted the
programmer to walk through the code statement-by-statement and explain it.

```erlang
-module(fizzbuzz).
```

This code defines the module name, but it also implies what the filename should
be (`fizzbuzz.erl`). If the programmer didn't know the two must match,
autoloading would not work correctly when the Erlang `c()` shell command 
was used.

```erlang
-export([run/0]).
```

This code is necessary to make it possible to call the `fizzbuzz:run()` function
externally, because all Erlang functions are private by default. The programmer
would also need to explain that `run/0` means 
"the run function with zero arguments", demonstrating an understanding of the
concept of *function arity*.

```erlang
run() -> 
  io:format("~p~n", [lists:map(fun transform/1, lists:seq(1,100))]).
```

This line of code has a ton of features crammed into it, including calls to both
the `io` and `lists` standard libraries, along with the syntax for passing
an existing function as an argument to another function 
(e.g. `fun transform/1`).

```erlang
transform(X) when X rem 15 =:= 0 -> "FizzBuzz";
transform(X) when X rem 5  =:= 0  -> "Fizz";
transform(X) when X rem 3  =:= 0  -> "Buzz";
transform(X) -> X.
```

Finally, we see function overloading and guards, both concepts that don't exist
in Ruby, but are commonly used in Erlang.

When you add up all of these points, it takes a whole lot of knowledge for even
an experienced programmer to write such a trivial program in a language they're
unfamiliar with. When you throw in things like familiarizing yourself with new
syntax and grammar rules, it becomes easy to see that trivial code literacy
demands a whole lot more understanding of a language than it appears to
at a first glance.


```
Start every day coding, end every day thinking.

1. Warmup exercise (30 mins)

Make sure to have these ready the night before, pick stuff
that you can work on right away without having to study 
in advance. 

They can either be book exercises or stuff
from other sources, but they should be self-verifiable
for correctness. Goal is not to finish but just to 
learn as much as possible.

2. Book reading and exercises (90 mins)

Jumping around chapters is OK, but reading whole chapters
at a time is encouraged. Read what is most related to
the projects you're working on. 

Give reading a higher priority over exercises during this 
time, and do only exercises related to the current reading.

3. Work on projects (90 mins)

Start with bowling score calculator, then dining philosophers,
then IRC rover bot if time permits. Focus on getting working
code first, before worrying about correct code. But once you
have a working solution, figure out how to make it right,
and get help if necessary for style questions.

4. Review today's work and do next day's prep work (30 mins)

Prepare questions, TODO lists, and exercises for the next
day. Reflect on what was learned today, possibly looking
up tangential points that you didn't have time for in the
day, or seek solutions to exercises already published online,
or write notes asking for help. 

Try to do all four hours in a single day if possible,
otherwise try to do 1+2 and 3+4 in two sessions as close
together as possible.
```

Projects summary (what and why)
-----------------

Book learning
-------------

Typed in, ran, and tinkered with nearly every code snipped from 
CH 1 to CH 14. (First several chapters before the "practice week"
brought me up to fizz-buzz level knowledge, skipped a couple 
chapters, but wrote dozens of functions)

Went off on several tangents (find a couple examples)

Find some book learning highlights from each day, and note tie-ins with
exercises / projects.

(what's shown in this article is actually just a small fraction
of the code I typed, including all book snippets and most exercises
(get a count or rough estimate)

Maybe show the fizzbuzz example?

Daily Review
------------

Cover this by adding a wrap-up paragraph or two at the end of each day entry.
Summarize the most important lessons learned, the pitfalls and triumphs,
and what I had planned to do the next day.

Next actions
------------

What did I leave undone? What could I have done next?

* Process termination still not 100% clear to me
* I still suck at concurrency concepts

Wrapup
------

Re-state the four step system and its benefits in a couple paragraphs,
invite others to try it.

Learning is cyclical. Always go back and see how your new knowledge might have
been applied to old problems, particularly wherever you struggled before.


Raw journal notes + checklists
-------------------------------

Share, don't share, summarize?


Preliminaries: December 26 - Jan 5 (12 hrs)
-------------------------------------------

Summarize what was studied / learned during this time period.

Maybe create a bulleted list of "What I already know about erlang"
based on Ch 1-6 and my prior knowledge.


* structures: Atoms, Integers, Floats, Tuples, Lists, Records, Strings(`*`)
* constructs: Modules, annotations, functions.
* workflow: shell, compilation
* pattern matching, list comprehensions, guards, recursive coding, 
  single assignment
* Using io:format to print out output
* Basic error handling
* Standard library features like `erlang:*` and `lists:*` 

(Probably more)

Consider showing fizzbuzz example for this.

Day 1: January 6 (Monday)
-------------------------------------------

### Finding the smallest element of a list

Good refresheer on pattern matching and recursive coding style (from Journal)

Original solution:

```erlang
-module(mylists).
-export([minimum/1]).

minimum([H|T]) -> minimum(T, H).

minimum([],    Min) -> Min;
minimum([H|T], Min) -> 
  case Smallest < H of
    true  -> minimum(T, Min);
    false -> minimum(T, H)
  end.
```

In retrospect:

```erlang
-module(mylists).
-export([minimum/1]).

minimum([Min|T]) -> minimum(T, Min).

minimum([], Min) -> Min;

minimum([H|T], Min) when Min < H -> minimum(T, Min);

minimum([Min|T], _) -> minimum(T, Min).
```

## (Almost working ping-pong) 

I didn't have much to go on yet, but the file server example and hello world
example were useful in preparing this. It's always good to save all exercises
you work on / projects you work on because they become your library for
looking up features in context.

Almost works, but has a bug in it! Deal with that later.

```erlang
-module(ping_pong).
-export([start/1, loop/1]).

start(Message) -> spawn(ping_pong, loop, [Message]).

loop(Message) ->
  receive
    {Client, N} ->
      io:format("~p Received: ~s~n", [self(), Message]),

      %% FIXME: Find out why this isn't working! %%
      case N of
        1    -> Client ! { self(), N -1 }, exit(self(), ok);
        0    -> exit(self(), ok);
        true -> Client ! { self(), N - 1 }
      end
  end,
  loop(Message).
```

## Reading notes

Already read Ch 1-6 (skimmed 5) during preliminaries doing most of their
exercises. Decided to skip Ch 7 on binary processing, since none of my projects
would need it.

Focus for the day is on Ch 8, a misc. grab bag of erlang features. 

> TODO: Try to find features introduced in this chapter and point them out
if they're shown in other code samples, particularly in this day's project
or the next day's warmup exercise.

Possible points of interest:

* Code loading (pp124-126)
* Macros / preprocessor
* Annotations
* Non-short circuiting logic
* List subtraction
* Process dictionary

## Bowling

Discuss the first data modeling challenges. Note that at this point, I'm not
even sure what the differences between lists and tuples are (see the journal).

Note how pattern matching makes non-uniformity less awkward than in Ruby
(e.g. `{10}` vs `{A, B}`)

Note how I got hung up on `true -> ...` in case for a while, but eventually
came to realize that catchall would be something like `_ -> ...`. Discovering
the same bug in this code helped me realize what I'd need to do to fix
the other ping-pong code, but only after sleeping on it (maybe split
this explanation up into two parts, one in today's review, the other in the
next day). 

Note how it was fun to write simple unit tests this way, if brittle.

```erlang
-module(bowling).
-export([score/1, test/0]).

score([]) -> 0;
score([H|T]) -> 
  case H of
    {A} ->

      [H1|T1] = T,

      case H1 of
        { B, C } -> A + B + C + score(T);
        { B } -> 
          case T1 of
            []    -> 0;
            _ ->
              [H2|_] = T1,
              case H2 of
                { C, _ } -> A + B + C + score(T);
                { C }    -> A + B + C + score(T)
              end
          end
      end;

    {A, B} when 10 =:= A + B ->
      [H1|_] = T,

      case H1 of
        {C, _} -> A + B + C + score(T);
        {C}    -> A + B + C + score(T)
      end;


    {A, B} -> H, A + B + score(T)
  end.

test() -> 
  9  = score([{7, 2}]), 

  %            4     6     8     9     5     0     3     1     0     4
  40 = score([{1,3},{2,4},{3,5},{5,4},{2,3},{0,0},{1,2},{1,0},{0,0},{1,3}]),

  %            4     6     8     S:12  5     0     3     1     0     4
  43 = score([{1,3},{2,4},{3,5},{5,5},{2,3},{0,0},{1,2},{1,0},{0,0},{1,3}]),

  %            4     6     8     S:15 5     0     3     1     0     4
  46 = score([{1,3},{2,4},{3,5},{10},{2,3},{0,0},{1,2},{1,0},{0,0},{1,3}]),

  %            4     6     8     S:14  S:11  1     3     1     0     4
  52 = score([{1,3},{2,4},{3,5},{5,5},{4,6},{1,0},{1,2},{1,0},{0,0},{1,3}]),

  %            4     6     8    S:21 S:13    3   3     1     0     4
  63 = score([{1,3},{2,4},{3,5},{10},{10},{1,2},{1,2},{1,0},{0,0},{1,3}]),

  300 = score([{10},{10},{10},{10},{10},{10},{10},{10},{10},{10},{10},{10}]),

  %             30   30   30   30   29   20   20    30   30   30   
  279 = score([{10},{10},{10},{10},{10},{10},{9,1},{10},{10},{10},{10},{10}]),

  ok.
```

## Day wrapup notes

>  The code I wrote for my bowling score calculator has no error handling, and
is a terrible mess of case statements, but it seems to be working! 

Day 2: January 7 (Monday)
-------------------------------------------

Originally planned to spend a whole extra
day on sequential erlang, but then realized
that concurrency is more interesting to me. 
(unsure to put this point at the head, the summary, or omit it)

## Working ping pong + Refactored ping-pong

Realized `true -> ...` case wasn't working but didn't connect it
to yesterday's notes yet, switch to `if`.

(working example)

```erlang
-module(ping_pong).
-export([start/1, loop/1]).

start(N) -> 
  Ping = spawn(ping_pong, loop, [ping]),
  Pong = spawn(ping_pong, loop, [pong]),

  Ping ! { Pong, N },
  done.

loop(Message) ->
  receive
    {Client, N} ->
      io:format("[~p] ~p Received: ~s ~n", [N, self(), Message]),

      if 
        N > 2   -> Client ! { self(), N - 1 }, loop(Message);
        N =:= 2 -> Client ! { self(), N - 1 };
        true -> void
      end
  end.
```

Then realize that I literally used the same pattern in my bowling example, but
used `case`, and switched back to it:

```erlang
-module(ping_pong).
-export([start/1, loop/1]).

start(N) -> 
  Ping = spawn(ping_pong, loop, [ping]),
  Pong = spawn(ping_pong, loop, [pong]),

  Ping ! { Pong, N },
  done.

loop(Message) ->
  receive
    {Client, N} ->
      io:format("[~p] ~p Received: ~s ~n", [N, self(), Message]),

      case N of
        1 -> done;
        2 -> Client ! { self(), N - 1 }, done;
        _ -> Client ! { self(), N - 1 }, loop(Message)
      end
  end.
```

Here is where I struggled with process termination. Do we explicitly
call exit() when the process is done? Simply let the loop terminate?
Leave the loop running in a refreshed state? Is it a concern to have
processes accumulating in a zombie state of some sort? The exercise indicates
that we should make sure to terminate the processes gracefully, but I'm
unsure what that means in Erlang.

ED NOTE: Did not look this up at the time, but there is some relevant 
discussion here: http://stackoverflow.com/questions/14515480/processes-exiting-normally

Seems that a function simply returning does not terminate a process in erlang.

### Reading notes

Decided to skip Ch 9 on type system and Ch 10 on compiler tools, because neither
were essential for my projects and I was concerned about the limited time I'd
have in the practice week.

Possible points of interest:

* Tuple modules
* Message processing semantics (esp unmatched messages) -- 194
  (contrast this to our actor article, at least my understanding of it
  being roughly equivalent to a work queue. Possible look at Celluloid)
* More confusion around code loading
* Request/response pattern  + RPC style
* Receive timeouts
* Register
* TCO caveat
* Processes are cheap (seems like a broken record on this)
* Boilerplate ??? (maybe not)

Find a way to discuss the above without drawing too much from the book,
maybe pick a few topics and show original examples.

## Refactored bowling

Note the piecewise problem decomposition, and my concerns about being too
golf-ish, brittle. But also not my feeling of how it helps clarity / 
elegance / simplicity, and linerizes the code to eliminate nested conditionals.

```erlang
-module(bowling).
-export([score/1, test/0]).

score([])                          -> 0;
score([{10}|T])                    -> strike(T) + score(T);
score([{A,B}|T]) when 10 =:= A + B -> spare(T) + score(T);
score([{A,B}|T])                   -> A + B + score(T).

strike([])                   -> 0;
strike([{10}])               -> 0;
strike([{10}, {10}|_])       -> 30;
strike([{10}, {Ball2, _}|_]) -> 20 + Ball2;
strike([{Ball1, Ball2}|_])   -> 10 + Ball1 + Ball2.

spare([])                -> 0;
spare([{10}|_])          -> 20;
spare([{NextBall, _}|_]) -> 10 + NextBall.

test() -> 
  9  = score([{7, 2}]), 

  %            4     6     8     9     5     0     3     1     0     4
  40 = score([{1,3},{2,4},{3,5},{5,4},{2,3},{0,0},{1,2},{1,0},{0,0},{1,3}]),

  %            4     6     8     S:12  5     0     3     1     0     4
  43 = score([{1,3},{2,4},{3,5},{5,5},{2,3},{0,0},{1,2},{1,0},{0,0},{1,3}]),

  %            4     6     8     S:15 5     0     3     1     0     4
  46 = score([{1,3},{2,4},{3,5},{10},{2,3},{0,0},{1,2},{1,0},{0,0},{1,3}]),

  %            4     6     8     S:14  S:11  1     3     1     0     4
  52 = score([{1,3},{2,4},{3,5},{5,5},{4,6},{1,0},{1,2},{1,0},{0,0},{1,3}]),

  %            4     6     8    S:21 S:13    3   3     1     0     4
  63 = score([{1,3},{2,4},{3,5},{10},{10},{1,2},{1,2},{1,0},{0,0},{1,3}]),

  300 = score([{10},{10},{10},{10},{10},{10},{10},{10},{10},{10},{10},{10}]),

  %             30   30   30   30   29   20   20    30   30   30   
  279 = score([{10},{10},{10},{10},{10},{10},{9,1},{10},{10},{10},{10},{10}]),

  %             30   30   30   30   29   20   20    29   20   20   
  258 = score([{10},{10},{10},{10},{10},{10},{9,1},{10},{10},{9,1},{10}]),

  %             30   30   30   30   29   20   20    29   20   20   
  257 = score([{10},{10},{10},{10},{10},{10},{9,1},{10},{10},{9,1},{9,1}]),

  ok.
```

(refactoring also applies to list:min, though I didn't know
it at the time)

## Inital very broken dining philosophers

https://github.com/sandal/erlang-practice/commit/df17dddec3588bff7b73417d0290c48beb2cf6bf

(But maybe Chopstick is almost right? **CHECK THIS**)

## Day wrapup notes

Used most of my wrapup time on reading, because I had exercises queued
up for the next day (register race, ring).

Day 3: January 8 (Wednesday) -- cut short
-------------------------------------------

(May want to streamline the discussion for this day because of its lack of consistency with the
other days, but see how it plays out.)

## Register race condition

Note my struggle and lessons learned from this, even though I couldn't solve 
it myself. Note epiphany about Erlang not being totally immune to race
conditions, and my understanding of the problem. Also note request/response
pattern seen again here for sync (point out synthesis used to get from
here to dining philosophers).

```erlang
-module(concurrency).
-export([start/2]).

% Solution is from http://forums.pragprog.com/forums/27/topics/124

start(Atom, Fun) ->
    Registrant = self(),
    spawn(
        fun() ->
            try register(Atom, self()) of
                true ->
                    Registrant ! true,
                    Fun()
            catch
                error:badarg ->
                    Registrant ! false
            end
        end),
    receive
        true  -> true;
        false -> erlang:error(badarg)
    end.
```

( consider cleaning up code )


My understanding after investigation (see journal for more details):

> It's not possible to use `whereis` to verify that a process has been
registered or not, because both processes could pass that test BEFORE their call
to `register` is processed. If one process completes the register process, the
other will fail, but only after `start()` returns, which is not what the
exercise calls for.

> To verify success or failure BEFORE `start()` returns, the spawned processes
communicates back to the process that spawned them about their status. The
parent process uses `receive` to wait for a response, ensuring that failure is
communicated at the time the method returns, not after.

This problem is more general than the race condition, it also hints at how to
write functions that fail sychronously. Unsure whether synchronous failure
is an edge case in Erlang or not (where it would be the default in
single-threaded Ruby)

### Reading notes  (discuss Philo research?)

Possible points of interest:

* Self-calling `Fun`. Why did I want this? For quick spawn examples???
* "Let some other process fix the error" and "Let it crash"
* Corrective vs. defensive programming
* Some extra research into DIning Philoosopher: Chandry/Misra and a waiter.

## Day wrapup notes

Didn't go as planned, mostly just did some work on exercise and a whole lot of
searching around for answers. Also didn't get that far on Dining Philosophers,
except to research strategy. But rather than attempt to make up for lost time,
I just let it go.

Day 4: January 9 (Thursday)
-------------------------------------------

## Process Ring

note dining philosophers synthesis.
(or maybe not, because we didn't go chandry/misra)

note mixed feelings about academic nature of exercise
note usefulness of drawing a picture

```erlang
-module(ring(Probably more).
-export([send/2, loop/1]).

send(N, M) ->
  Head = build(N),
  Head ! { deliver, howdy, N*M }.

start() ->
  spawn(ring, loop, [void]).

connect(From, To) ->
  From ! {observe, To}.

build(N) ->
  First = start(),
  Last  = build(N-1, First),

  connect(Last, First),
  First.

build(0, Current) -> Current;

build(N, Current) ->
  Next = start(),
  connect(Current, Next),
  build(N-1, Next).

loop(Observer) ->
  receive
    {observe, NewObserver} -> 
      io:format("~p is now observing ~p", [self(), NewObserver]),
      loop(NewObserver);
    {deliver, _, 0} ->
      io:format("Done sending messages!~n"),
      loop(Observer);
    {deliver, Message, Count} when Observer =/= void ->
      io:format("[~p], ~p is sending message ~p to ~p~n", 
        [Count, self(), Message, Observer]),
      Observer ! {deliver, Message, Count - 1},
      loop(Observer)
  end.
```

## Reading notes

Points of interest:

* Process linking and monitoring
* Various ways of error signalling
* Linking + monitoring patterns / firewalls

(most time spent on book code samples)


## Dining philosophers


```erlang
-module(philosophers).
-export([dine/0, loop/3]).

dine() ->
  [C1, C2, C3, C4, C5] = [chopstick:start(X) || X <- [1,2,3,4,5]],
  dine(C1, C2, C3, C4, C5).


dine(C1, C2, C3, C4, C5) -> 
  Aristotle    = spawn(philosophers, loop, ["Aristotle",    C1, C2]),
  Popper       = spawn(philosophers, loop, ["Popper",       C2, C3]),
  Epictetus    = spawn(philosophers, loop, ["Epictetus",    C3, C4]),
  Heraclitus   = spawn(philosophers, loop, ["Heraclitus",   C4, C5]),
  Schopenhauer = spawn(philosophers, loop, ["Schopenhauer", C1, C5]),

  Aristotle ! Popper ! Epictetus ! Heraclitus ! Schopenhauer ! think.

loop(Philosopher, LeftChopstick, RightChopstick) ->
  receive
    think -> 
      io:format("~p is thinking.~n", [Philosopher]),
      timer:sleep(1000),

      self() ! eat;
    eat   -> 
      LeftChopstick  ! {take, self()},

      receive
        {cs, FirstChopstick} ->
          io:format("~p picked up chopstick ~p~n", [Philosopher, FirstChopstick]), 

          RightChopstick ! {take, self()},
          receive
            {cs, SecondChopstick} ->
              io:format("~p picked up chopstick ~p~n", [Philosopher, SecondChopstick]),
              io:format("~p is eating.~n", [Philosopher]),
              timer:sleep(1000)
          end
      end,

      LeftChopstick  ! {drop, self()},
      RightChopstick ! {drop, self()},

      io:format("~p is done eating, releases chopsticks ~p and ~p~n",
        [Philosopher, FirstChopstick, SecondChopstick]),

      self() ! think
  end,

  loop(Philosopher, LeftChopstick, RightChopstick).
```

```erlang
-module(chopstick).
-export([start/1, loop/2]).

start(Number) ->
  spawn(chopstick, loop, [Number, nobody]).

loop(Number, Owner) ->
  receive
    {take, Owner} -> loop(Number, Owner);
    {take, NewOwner} when Owner =:= nobody -> 
      NewOwner ! {cs, Number},
      loop(Number, NewOwner);
    {drop, Owner} ->
      loop(Number, nobody)
  end.
```

(note request/response synchronization and similarity to
N-ring problem, and also why I used the naive solution)

Note what I learned about Erlang from this example, even when using the naive
solution.

Never really thought through the many ways of solving this problem.

## wrapup

* Note about syntax errors (journal)
* Thoughts about asynchronous message delivery


Day 5: January 10 (Friday)
-------------------------------------------

### Start with monitor

Note points of interest here, mainly error handling.

```erlang
-module(errors).
-export([my_spawn/3]).

my_spawn(Mod, Func, Args) ->
  Pid = spawn(Mod, Func, Args),
  {T1, _} = statistics(wall_clock),

  spawn(fun() ->
    Ref = monitor(process, Pid),
    receive
      { 'DOWN', Ref, process, Pid, Why } ->
        io:format("~p went down with reason: ~p~n", [Pid, Why]),

        {T2, _} = statistics(wall_clock),

        io:format("~p was alive for ~p seconds~n", [Pid, (T2-T1)/1000])
    end
  end),

  Pid.
```

## Reading notes summarized here in transition
  * Die together workers and parent monitor
  * Keepalive
  * Distributed erlang, session names + cookie based auth
  * KVM uses process dictionary, what about tuple modules? (i guess it'd run
    into register problems and remote RPC issues)
  * Didn't quite get `nl()` to work
  * Recurring problems with stuck processes + testing callbacks
    (move to review instead?)

## Trivial process

(explain tangent)

```erlang
-module(trivial_process).
-export([start/0, loop/1]).

start() -> spawn(?MODULE, loop, [1]).

loop(N) ->
  io:format("Tick ~p.~n", [N]),

  receive
    _ -> error("Boom!")
  after 1000 ->
    loop(N+1)
  end.
```


## Rover


```erlang
-module(world).
-export([start/1, loop/3]).

start(Filename) ->
  spawn(world, loop, [read(Filename), 11, 13]).

loop(MapData, Row, Col) ->
  receive 
    {Caller, MsgID, snapshot} ->
      Caller ! { self(), MsgID, {snapshot, snapshot(MapData, Row, Col)}},
      loop(MapData, Row, Col);
    {Caller, MsgID, move_north} ->
      Caller ! { self(), MsgID, {move_north, Row-1, Col}},
      loop(MapData, Row-1, Col);
    {Caller, MsgID, move_south} ->
      Caller ! { self(), MsgID, {move_south, Row+1, Col}},
      loop(MapData, Row+1, Col);
    {Caller, MsgID, move_east} ->
      Caller ! { self(), MsgID, {move_east, Row, Col+1}},
      loop(MapData, Row, Col+1);
    {Caller, MsgID, move_west} ->
      Caller ! { self(), MsgID, {move_west, Row, Col-1}},
      loop(MapData, Row, Col-1)
  end.


read(Filename) ->
  { ok, MapBinary } = file:read_file(Filename),
  MapText = binary_to_list(MapBinary),
  list_to_tuple(
    [ list_to_tuple(string:tokens(X, " ")) || 
      X <- string:tokens(MapText, "\n")]).

% TODO: add @ sign to indicate current location.
snapshot(Map, Row, Col) ->
  RowWindow = [ element(RowD, Map) || 
                RowD <- lists:seq(Row - 2, Row + 2) ],

  string:join(
    lists:map(fun(RowData) ->
      string:join(
        [element(ColD, RowData) || ColD <- lists:seq(Col - 2, Col + 2)], 
        " ")
      end, 
      RowWindow), 
     "\n") ++ "\n".
```

```erl
-module(radio).
-export([start/1, loop/1]).
-define(TRANSMISSION_DELAY, 5000).

start(Controller) -> spawn(radio, loop, [Controller]).

loop(Controller) ->
  receive
    { transmit, Pid, Message } ->
      erlang:send_after(?TRANSMISSION_DELAY, Pid, {self(), erlang:make_ref(), Message});
    Message ->
      erlang:send_after(?TRANSMISSION_DELAY, Controller, Message)
  end,

  loop(Controller).
```


```erl
-module(controller).
-export([start/0, loop/0]).

start() -> spawn(controller, loop, []).

loop() ->
  receive
    {_, MsgId, {snapshot, MapData} } ->
      io:format("~s~n~n(msg id: ~p)~n", [MapData, MsgId]);
    Any -> io:format("Received message: ~p~n", [Any])
  end,
  loop().
```

Rover (note ugly map parsing code, consider attempting a refactor,
note ease of concurrency stuff)

## Daily wrapup

This project has given me a very visceral lesson in the differences between:

* Reading code
* Running example code
* Trying out coding exercises
* Working on toy projects
* Working on real projects

Learning a language is an N-dimensional activity, it's surprising that we
tend to have a much more simplistic view of what is involved (or at least I do).
So much of a willingness to theorize and discuss that which we have very little
practical experience with.
