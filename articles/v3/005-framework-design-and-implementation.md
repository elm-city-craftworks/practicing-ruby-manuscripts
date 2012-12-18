Ruby is a great language for building application frameworks, particularly
micro-frameworks. The sad thing is that by the time most frameworks become
popular, they end up quite complicated. To discover the joy of building reusable
scaffolding for others, it's necessary to take a look at where the need for that
scaffolding comes from in the first place.

In the January 2012 core skills session at Mendicant University, I asked our
students to each build multi-user email based applications. While the students
were working on very different projects, there was a ton of boilerplate that was
common between them all. Because it was too painful to watch the same bits of
code get written again and again in slightly different ways, I decided to build
a tiny framework to solve this problem.

In this issue of Practicing Ruby and the one that follows it, I'm going to have
you work through the code I wrote and help me figure out what goes into building
a good application framework. The goal for this issue is to generate ideas and
questions about the codebase. All of what we learn from this exercise will be
neatly packaged up and synthesized in time for Issue 3.6, but for now I'm
looking for folks to get their hands dirty.

## The Challenge

I would like you to spend at least the same amount of time you'd ordinarily
spend reading a Practicing Ruby article actively reading and working through
[Newman 0.1.1](https://github.com/mendicant-original/newman/tree/v0.1.1), my
micro-framework for email based applications. I have intentionally left the
source uncommented for two reasons: to get you to practice your code reading
skills and to get your candid feedback on the strengths and weaknesses of my
overall design without influencing you too much.

As you read the code, don't just passively click through files on github!
Instead, pull down the source and play with it: Run the examples if you can, or
even better, build your own examples. Try to break stuff if you think you might
be able to find a bug or two, or try to add a new feature you find interesting.
This is an open sandbox to play in!

Once you have managed to find your way around, you're encouraged to start
actively collaborating. I'll be available via the #newman IRC channel and
[newman@librelist.org](newman@librelist.org) to listen to any ideas or questions
you have. Of course, feel free to use Github for bug reports, feature requests,
and comments on pull requests / commits.

In this code I've tried to apply pretty much everything I've ever taught via
Practicing Ruby whenever there was an opportunity to do so. I've also broken
away from established Ruby conventions in places to explore new ideas. Reading
it will be worth your time, and if you actively involve yourself in the
conversations around it, you'll be sure to level up your Ruby skills in no time. 

**One last thing: Don't be afraid to ask where to get started if you feel stuck.
The purpose of this exercise is to learn, and I will do what I can to help you
get a lot out of this challenge.**
