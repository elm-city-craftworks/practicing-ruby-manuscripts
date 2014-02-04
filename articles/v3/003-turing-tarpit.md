> NOTE: This article describes an interactive challenge that was done in
> realtime at the time it was published. You can still make use of it
> by starting from the initial version of the code, but if you'd
> rather skip to the end results, be sure to read [Issue 3.4](https://practicingruby.com/articles/spiral-staircase-of-refactoring).

A programming language that is endlessly flexible but difficult to use because
of its lack of support for common operations is known as a [Turing
tarpit](http://en.wikipedia.org/wiki/Turing_tarpit). While a range of esoteric
programming languages fall under this category, the [Brainfuck
language](http://en.wikipedia.org/wiki/Brainfuck) is one that stands out for its
extreme minimalism.

Brainfuck manages to be Turing complete in only 8 operations, which means that
despite being functionally equivalent to Ruby in a theoretical sense, it offers
virtually none of the conveniences that a modern programmer has come to expect.
You don't need to go any farther than its "Hello World" program to see that it
isn't a language that any sane person would want to work in:

```ruby
++++++++++[>+++++++>++++++++++>+++>+<<<<-]>++.>+.+++++++..+++.>++.
<<+++++++++++++++.>.+++.------.--------.>+.>.
```

However, the simplicity of the language is not without at least some merit. A
consequence of having a somewhat trivial to parse syntax and very basic
semantics is that while Brainfuck may be one of the hardest languages to *use*,
it simultaneously happens to be one of the easiest languages to *implement*.
While it's not quite as easy a task as working on a code kata, it takes roughly
the same order of magnitude of effort to produce a fully functioning Brainfuck
interpreter. The key difference is that you end up with some software that is
much deeper than your average bowling score calculator after you've completed
the exercise.

A functioning Brainfuck interpreter is complex enough where you can begin to ask
serious questions about code quality and overall design. It is also something
that you can build new functionality on top of in a ton of interesting ways. So
in this exercise, the real payoff comes after you've run your first "Hello
World" example and have a working interpreter to play with. The downside is that
it might take you a day or two of work to get to that point, and after all that
effort, you might need to think about doing something that's just a bit more
meaningful with your life. But like most things that involve this sort of
drudgery, being a Practicing Ruby subscriber can help you skip the boring stuff
and get right to the juicy parts.

Rather than a typical article, this issue is instead an interactive challenge
for our readers to try at home. I've posted a [simple and functional Brainfuck
interpreter](https://github.com/elm-city-craftworks/turing_tarpit/tree/starting_point) on GitHub,
and I'm inviting all of you to look for ways of improving it. The code itself is
good in places and not so good in others, and I've intentionally left in some
things that can be improved. Your challenge, if you choose to accept it, is as
follows:

* If you spot things in the code that can be improved, let me know!
* If you spot a bug, file a bug report via github issues and optionally send a pull request that includes a failing test.
* If you have time to make a refactoring or improvement yourself, fork the project and submit pull requests
* If you want to add documentation patches, those are welcome too!
* Feel free to work on this with friends and colleagues, even if they aren't Practicing Ruby subscribers.

In [Issue 3.4](https://practicingruby.com/articles/spiral-staircase-of-refactoring), I will
go over the improvements we made to this project as a group, and discuss why
they're worthwhile. Because I know there are several things that need
improvement in this code that are pretty general in nature, I'm reasonably sure
that article will be a generally interesting discussion on software design and
Ruby idioms, as opposed to a collection of esoterica. But since I don't know
what to expect from your contributions, the exact contents will be a surprise
even to me.
