In Issue #6, you got to see my intentionally bad implementation of Tic Tac Toe. For today, I have promised to show you some better code and the steps I took to get there. But before we move forward, let's take a quick look back at where we started.

To start this exercise, I had challenged myself to implement this simple game without using any user defined classes or methods. Given that I wanted to make sure I produced *bad* code to start with, I got a little nervous when my back-of-the-napkin proof of concept didn't come out looking that bad. Here it is again below, for those who forgot about it.

```ruby
board = [[nil,nil,nil],
         [nil,nil,nil],
         [nil,nil,nil]]

players = [:X, :O].cycle

loop do
  current_player = players.next
  puts board.map { |row| row.map { |e| e || " " }.join("|") }.join("\n")
  print "\n>> "
  row, col = gets.split.map { |e| e.to_i }
  puts
  board[row][col] = current_player
end
```

The above code is good demo-ware, as long as you type really carefully and conveniently forget to finish a game before hitting ctrl+c. But to make a real, playable implementation, some end game conditions and basic validations are necessary. To my great joy, adding those new features caused this tight little script to explode into a hot mess of intertwined logic and nasty little hacks. Check out [the source tree](https://github.com/sandal/tictactoe/tree/7fd72a33aec33f75909d8c9d59a43423b0f66b24) that we ended up with at the end of Issue #6 to see how things turned.

While concise at less than 60 lines of code, it's pretty easy to see that this isn't the kind of software we should aspire to be writing. So the challenge was to start here and end up somewhere better.

Whenever I do this exercise with my students, there is a roadmap I follow that tends to lead to some decent insights. It roughly goes like this:

* Get some basic file structures and namespaces in place so that you get yourself out of the global namespace and open the doors for scripting some examples or running things in irb without firing off a procedure automatically.

* Break down the procedure into some separable chunks so that you can think about smaller parts of the problems, and more easily see the dependencies between the steps in the procedure.

* Re-think the design by identifying areas where objects can put an abstraction barrier between different layers of data and logic. Strive to have each bit of code do one thing and one thing well.

* Identify the leaky abstractions and dangly bits that didn't get ironed out by the last step. Aim for beautiful solutions, but be skeptical of over-engineering at this point. No problem can be modeled perfectly

* Reflect on the exercise, and ask yourself whether you've gone far enough with your cleanup. If you feel like so, then be sure to think about whether you've gone *too* far!

This is the approach I took as I worked on this problem myself, and you'll be able to see it step by step in the git logs. I tried to write good log messages, so I will link to them rather than repeat what was said, but I'll also share some more big-picture oriented thoughts as I walk you through my work.

### Basic organization first

Here is my [first commit](https://github.com/sandal/tictactoe/commit/5af96941d74f8014a3276b77fe67c17e0ed5e2df) of the evening. And this is the [source tree](https://github.com/sandal/tictactoe/tree/5af96941d74f8014a3276b77fe67c17e0ed5e2df).

Tiny changes really, but it's the first thing I do as soon as I've exited 'spike
mode' on any project, no matter how small. I've used a standard structure, and
it does two things for me:

1. Allows me to load my whole library with a single require. (See app.rb for example and note how it doesn't change throughout this walkthrough)

1. Places 100% of what I build under a single constant's namespace (i.e. `TicTacToe`)

These two points pretty much guarantee me that I won't have any naming clashes or unexpected collisions with other people's code unless I plan on loading a library that might clobber the name `TicTacToe` or the `require` path of <i>"tictactoe/*"</i>. It also makes it easy for me to start interacting with my code from scripts I write, from irb, and from unit tests. For so little work, we get a ton of benefit, and this is a great place to start when doing any sort of cleanup.

### Basic Slicing and Dicing

My next goal is to start breaking my monolithic procedure into some smaller chunks so I can get a sense of what parts go well together and how they need to interact with each other.

I start by realizing that using a singleton pattern for `Game`, while possible, isn't a great idea. A function bag approach in which we pass board and player information around like crazy also wouldn't be great, so I decide to make `Game` an ordinary class in this [commit](https://github.com/sandal/tictactoe/commit/2579626bd73fc7ad9e7d0a87419d5ecab2aacdda).

Read the message, and then if you'd like, have a look at the [updated source tree](https://github.com/sandal/tictactoe/tree/2579626bd73fc7ad9e7d0a87419d5ecab2aacdda).

I immediately make use this refactoring by breaking down the original game procedure into several smaller, simpler methods. ([commit](https://github.com/sandal/tictactoe/commit/286724de5328fda779caa500ccc76a0ad5de2bd7), [source](https://github.com/sandal/tictactoe/tree/286724de5328fda779caa500ccc76a0ad5de2bd7))

At this point, it's not uncommon for folks to think they're done refactoring. By giving things nicer names and distributing the pain points so that they're not all crammed together in one place, the code feels cleaner. But upon further investigation of this code, while perhaps understandability and organization have improved, flexibility and abstraction have not. This is what I like to call 'procedural programming with objects', and we can do better than this.

The good news is, with the code cleaned up a bit, we see where some of the pain points are. When it seems like a large amount of your code is dedicated to handling a particular concept, that means you have an object begging to be born. Our handling of the game board logic in this code is a prime example.

### Sneaking in Domain Models

A key principle of object oriented design is to do one thing and do it well. But what does that mean? Hopefully, this refactored `Board` class gives you an idea!
([commit](https://github.com/sandal/tictactoe/commit/efcbf51bcc1f7d4d094c671b60761229aec3dded), [source](https://github.com/sandal/tictactoe/tree/efcbf51bcc1f7d4d094c671b60761229aec3dded))

If you look at the `Board` class, you'll see that it takes the concept of a Tic Tac Toe board and solidifies it so that when `Game` works with it, `Board` does the heavy lifting and `Game` mostly just calls the methods it needs to get its job done. This lets `Game` forget about some of the finer points like what the individual kinds of illegal moves are, or how to compute the intersecting lines that cross through a given point. This sort of black box effect gives us some real abstraction, which is exactly why object oriented programming is as good as they say it is.

With this complex board logic out of the way and some updates to the way flow is handled in game, it's obvious that `Game` is now something like a controller, and `Board` is a model. But there are still some loose ends in `Game`, things that actually look like logic rather than just flow control and dispatch. The majority of the code you see in this class has to do with implementing a user interface and basic event loop. So, methods like `check_move`, `check_win`, and `check_draw` feel a little bit out of place, since they implement actual logic about the rules of the game rather than just how players interact with it.

Sometimes, little leaks like this aren't a big deal. In fact, the code looks reasonable to me at this point and if I were doing this for my day job and wasn't trying to get in the record books for 'World's Best Tic Tac Toe Implementation', I'd probably stop here.

But we're already cruising now, so why don't we try to shoot for the stars?

### Grail Quests

I really wanted to find a way to rip that last bit of domain logic out of `Game`, and after wrestling a little bit, I came up with something.
([commit](https://github.com/sandal/tictactoe/commit/0fef18d320af2bd1a08f5115a2b94e552205f218), [source](https://github.com/sandal/tictactoe/tree/0fef18d320af2bd1a08f5115a2b94e552205f218))

The thing I kept wrestling with was how to manage the screen output stuff. I wrestled with a bunch of ideas, including defining a simply `display()` method on `Game` like this:

```ruby
def display(message)
  puts message
end
```

The reason why I wanted this is so my Rules mixin could rely on a method that `Game` provided for display rather than directly assuming console output. But I think that what I ended up with is better.

Imagine that my `check_draw` method in Rules was written like this:

```ruby
def check_draw
  if @board.covered?
     display "It's a draw"
     game_over
  end
end
```

It's almost a trivial difference *except* that now we have a leak on the Rules side. If `TicTacToe::Game` is now meant to exclusively be a UI event loop, having the messages that are displayed to the user caught up in some module seems a bit ugly.

But instead, I chose to let `Game` fill in the blanks with an implementation like this:

```ruby
def check_draw
  if @board.covered?
     yield
     game_over
  end
end
```

This allows the draw logic to live in `Rules`, with calling code in `Game`
that looks like this:

```ruby
check_draw { puts "It's a draw" }
```

A place for everything and everything in its place! Time to go hang some banners on aircraft carriers, because well, Mission Accomplished.

### Fear, Uncertainty, and Doubt

Is this final implementation an example of good Ruby code? Yeah, probably. Is it excellent? I really have no idea. At the very least, it's almost certainly not 'The Best Tic Tac Toe Implementation Ever'.

But really, the kind of perfection I was trying to seek in this exercise is not really what we should be looking for in our day to day work. Right now I have the amps cranked up to 11, when 7 or 8 would really do fine. But as I said before, this is one of my favorite exercises for learning and teaching. Here's why: It really gets me thinking.

I'm still trying to decide on whether extracting out the `Rules` module was really necessary, and I also have some areas about this I still don't like. For example, I'm not sure whether `Board` should know more about the rules of the game, or even less. I don't like the hard coding I did of all the parameters of the game in there, but I can't put my finger on why. After all, it's very unlikely that Tic Tac Toe is suddenly going to become Chess and need to expand to an NxN board. Even if it did, wouldn't it need to change a whole lot to accommodate it?

Still, I don't like things like these constants:

```ruby
LEFT_DIAGONAL_POSITIONS  = [[0,0],[1,1],[2,2]]
RIGHT_DIAGONAL_POSITIONS = [[2,0],[1,1],[0,2]]
SPAN                     = (0..2)
CELL_COUNT               = 9
```

There is a natural connascence between all four of these values, but the code to generalize their creation would be longer and much uglier to read than the above. So maybe it's a good choice to do it this way, but it makes the mathematician in me uneasy.

Another thing I don't like about my design is `Board#to_s`, because putting presentation logic on domain logic is nasty. But to make a view object or otherwise promote one line of code to something more complex seems to be a cure that is worse than the disease.

But on the bright side of things, I really like the callback scheme for doing the bits of game logic like `check_win` and `check_draw` and passing in a block with the rendering code. This is actually a formal design pattern just hiding in a line of code, and things like that remind me of why Ruby is so beautiful.

Also, I've never used `throw` / `catch` before in real code. Never really saw why I'd need it. But at a glance, my use of it here actually seems pretty expressive and appropriate given the situation. But because I've never used it before, I'm still glancing at it sideways with considerable doubt. I even had to wrap it in a method called `game_over` to hide the throw keyword to get over my fear of its relative novelty. But now, my `game_over` method is like some sort of crazy goto call... and that makes me not so sure that this was a good idea afterall.

Oh yeah, and I also didn't write any tests while working on this code. I thought about writing them, but I felt that it'd cause me to think about the tests themselves more than the coding practices I was experimenting with. But then again, maybe if I wrote tests, I wouldn't be pondering the relative merits of my fancy `game_over()` goto.

And this is how this exercise always ends. It doesn't come together in a beautiful blossom of Ruby awesomeness, it just kind of falls off a cliff. But really, that's okay! Not every question needs to be answered, and as I said before, if this were something I was working on just to get a job done, I would happily make concessions where needed to avoid letting perfect become the enemy of the good.

Still, this sort of practice gnaws on your subconscious, and I've seen it lead to great progress in my own studies and in my students as well. Hopefully you've enjoyed seeing this process in action, and will give it a try soon if you weren't able to try it out this week.

### Submissions from our readers

I haven't had a chance to review them in depth, but a few readers did share
their own explorations with us. Check out the [github network graph](https://github.com/sandal/tictactoe/network) to see what others have done.

Looking forward to hearing your thoughts on this exercise, and whether it seems like something you could make good use of. Until next time, happy hacking!

  
> **NOTE:** This article has also been published on the Ruby Best Practices blog. There [may be additional commentary](http://blog.rubybestpractices.com/posts/gregory/036-issue-7-good-and-bad-code.html#disqus_thread) 
over there worth taking a look at.
