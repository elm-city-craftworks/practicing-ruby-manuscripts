Ruby makes it easy to quickly put together a proof-of-concept for almost any kind of project, as long as you have some experience in rapid application development. In this article, I will go over how I build prototypes, sharing the tricks that have worked well for me.

Today we'll be walking through a bit of code that implements a small chunk of a falling blocks game that is similar to Tetris. If you're not familiar with Tetris, head over to [freetetris.org](http://freetetris.org) and play it a bit before reading this article.

Assuming you're now familiar with the general idea behind the game, I'll walk you through the thought process that I went through from the initial idea of working on a falling blocks game to the small bit of code I have written for this issue.

### The Planning Phase

After running through a few ideas, I settled on a falling blocks game as a good example of a problem that's too big to be tackled in a single sitting, but easy enough to make some quick progress on.

The next step for me was to come up with a target set of requirements for my
prototype. To prevent the possibilities from seeming endless, I had to set a
time limit up front to make this decision making process easier. Because 
very small  chunks of focused effort can get you far in Ruby, I settled on
coming up with something I felt I could build within an hour or two.

I knew right away this meant that I wasn't going to make an interactive demo. Synchronizing user input and screen output is something that may be easy for folks who do it regularly, but my concurrency knowledge is very limited, and I'd risk spending several hours on that side of things and coming up empty if I went down that path. Fortunately, even without an event loop, there are still a lot of options for building a convincing demo.

In my initial optimism, I thought what I'd like to be able to do is place a piece on the screen, and then let gravity take over, eliminating any completed lines as it fell into place. But this would require me to implement collision detection, something I didn't want to tackle right away.

Eventually, I came up with the idea of just implementing the action that happens when a piece collides with the junk on the grid. This process involved turning the active piece into inactive junk, and then removing any completed rows from the grid. This is something that I felt fit within the range of what I could do within an hour or two, so I decided to sleep on it and see if any unknowns bubbled up to the surface.

I could have just started hacking right away, but ironically that's a practice I typically avoid when putting together rapid prototypes. If this were a commercial project and I quoted the customer 2-4 hours, I'd want to use their money in the best possible way, and picking the wrong scope for my project would be a surefire way to either blow the budget or fail to produce something interesting. I find a few hours of passive noodling helps me see unexpected issues before they bite me.

Fortunately, this idea managed to pass the test of time, and I set out to begin coding by turning the idea into a set of requirements.

### The Requirements Phase

A good prototype does not come from a top-down or bottom-up design, but instead comes from starting in the middle and building outwards. By taking a small vertical slice of the problem at hand, you are forced to think about many aspects of the system, but not in a way that requires you consider the whole problem all at once. This allows most of your knowledge and often a good chunk of your code to be re-used when you approach the full project.

The key is to start with a behavior the user can actually observe. This means that you should be thinking in terms of features rather than functions and objects. Some folks use story frameworks such as Cucumber to help them formalize this sort of inside-out thinking, but personally, I prefer just to come up with a good, clear example and not worry about shoehorning it into a formal setting.

To do this, I created a simple text file filled with ascii art that codified two cases: One in which a line was cleared, and where no lines were cleared. Both cases are shown below.


### CASE 1: REMOVING COMPLETED LINES

```
==========
           
           
           
           
           
           
   #       
   #|    | 
  |#||  ||
|||#||||||
==========
```

BECOMES:

```
==========
           
           
           
           
           


   |       
   ||    | 
  ||||  ||
==========
```

### CASE 2: COLLISION WITHOUT ANY COMPLETED LINES

```
==========
           
           
           
           
           
          
  #       
  ##|    |
  |#||  ||
||| ||||||
==========
```

BECOMES:

```
==========
           
           
           
           
           
          
  |       
  |||    | 
  ||||  ||
||| ||||||
==========
```

---------------------------------------------------------------------

With the goals for the prototype clearly outlined, I set out to write a simple program that would perform the necessary transformations.

### The Coding Phase

One thing I'll openly admit is that when prototyping something that will take me less than a half day from end to end, I tend to relax my standards on both testing and writing clean code. The reason for this is that when I'm trying to take a nose-dive into a new problem domain, I find my best practices actually get in the way until I have at least a basic understanding of the project.

What I'll typically do instead is write a single file that implements both the objects I need and an example that gets me closer to my goal. For this project, I started with a canvas object for rendering output similar to what I outlined in my requirements.

Imagining this canvas object already existed, I wrote some code for generating the very first bit out output we see in the requirements.

```ruby
canvas = FallingBlocks::Canvas.new

(0..2).map do |x|
  canvas.paint([x,0], "|")
end

canvas.paint([2,1], "|")

(0..3).map do |y|
  canvas.paint([3,y], "#")
end

(4..9).map do |x|
  canvas.paint([x,0], "|")
end

[4,5,8,9].map do |x|
  canvas.paint([x,1], "|")
end

canvas.paint([4,2], "|")
canvas.paint([9,2], "|")

puts canvas 
```

While I use a few loops for convenience, it's easy to see that this code does little more than put symbols on a text grid at the specified (x,y) coordinates. Once `FallingBlocks::Canvas` is implemented, we'd expect the following output from this example:

```
==========
           
           
           
           
           
           
   #       
   #|    | 
  |#||  ||
|||#||||||
==========
```

What we have done is narrowed the problem down to a much simpler task, making it easier to get started. The following implementation is sufficient to get the example working, and is simple enough that we probably don't need to discuss it further.

```ruby
module FallingBlocks
  class Canvas
    SIZE = 10

    def initialize
      @data = SIZE.times.map { Array.new(SIZE) }
    end

    def paint(point, marker)
      x,y = point
      @data[SIZE-y-1][x] = marker
    end

    def to_s
      [separator, body, separator].join("\n")
    end

    def separator
      "="*SIZE
    end

    def body
      @data.map do |row|
        row.map { |e| e || " " }.join
      end.join("\n")
    end
  end
end
```

However, things get a little more hairy once we've plucked this low hanging fruit. So far, we've built a tool for painting the picture of what's going on, but that doesn't tell us anything about the underlying structure. This is a good time to start thinking about what Tetris pieces are.

While a full implementation of the game would require implementing rotations and movement, our prototype looks at pieces frozen in time. This means that a piece is really just represented by a collection of points. If we define each piece based on an origin of [0,0], we end up with something like this for a vertical line:

```ruby
line = FallingBlocks::Piece.new([[0,0],[0,1],[0,2],[0,3]])
```

Similarly, a bent S-shaped piece would be defined like this:

```ruby
bent = FallingBlocks::Piece.new([[0,1],[0,2],[1,0],[1,1]])
```

In order to position these pieces on a grid, what we'd need as an anchor point that could be used to translate the positions occupied by the pieces into another coordinate space.

We could use the origin at [0,0], but for aesthetic reason, I didn't like the mental model of grasping a piece by a position that could potentially be unoccupied. Instead, I decided to define the anchor as the top-left position occupied by the piece, which could later be translated to a different position on the canvas. This gives us an anchor of [0,3] for the line, and an anchor of [0,2] for the bent shape. I wrote the following example to outline how the API should work.

```ruby 
line = FallingBlocks::Piece.new([[0,0],[0,1],[0,2],[0,3]])
p line.anchor #=> [0,3]

bent = FallingBlocks::Piece.new([[0,1],[0,2],[1,0],[1,1]])
p bent.anchor #=> [0,2]
```

Once again, a simple example gives me enough constraints to make it easy to write an object that implements the desired behavior.

```ruby
class Piece
  def initialize(points)
    @points = points
    establish_anchor
  end

  attr_reader :points, :anchor

  # Gets the top-left most point
  def establish_anchor
    @anchor = @points.max_by { |x,y| [y,-x] }
  end
end
```

As I was writing this code, I stopped for a moment and considered that this logic, as well as the logic written earlier that manipulates (x,y) coordinates to fit inside a row-major data structure are the sort of things I really like to write unit tests for. There is nothing particularly tricky about this code, but the lack of tests makes it harder to see what's going on at a glance. Still, this sort of tension is normal when prototyping, and at this point I wasn't even 30 minutes into working on the problem, so I let the feeling pass.

The next step was to paint these pieces onto the canvas, and I decided to start
with their absolute coordinates to verify my shape definitions. The following example 
outlines the behavior I had expected.

```ruby
canvas = FallingBlocks::Canvas.new

bent_shape = FallingBlocks::Piece.new([[0,1],[0,2],[1,0],[1,1]])
bent_shape.paint(canvas)

puts canvas
```

OUTPUTS:

```
==========
          
          
          
          
          
          
          
#         
##        
 #        
==========
```

Getting this far was easy, the following definition of `Piece` does the trick:

```ruby
class Piece
   SYMBOL = "#"

  def initialize(points)
    @points = points
    establish_anchor
  end

  attr_reader :points, :anchor

  # Gets the top-left most point
  def establish_anchor
    @anchor = @points.min_by { |x,y| [y,-x] }
  end

  def paint(canvas)
    points.each do |point|
      canvas.paint(point, SYMBOL)
    end
  end
end
```

This demonstrates to me that the concept of considering pieces as a collection of points can work, and that my basic coordinates for a bent piece are right. But since I need a way to translate these coordinates to arbitrary positions of the grid for this code to be useful, this iteration was only a stepping stone. A new example pushes us forward.

```ruby
canvas = FallingBlocks::Canvas.new

bent_shape = FallingBlocks::Piece.new([[0,1],[0,2],[1,0],[1,1]])

canvas.paint_shape(bent_shape, [2,3])

puts canvas
```

OUTPUTS

```
==========
          
          
          
          
          
          
  #       
  ##      
   #      
          
==========
```

As you can see in the code above, I decided that my `Piece#paint` method was probably better off as `Canvas#paint_shape`, just to collect the presentation logic in one place. Here's what the updated code ended up looking like.

```ruby
class Canvas
 # ...

 def paint_shape(shape, position)
   shape.translated_points(position).each do |point|
     paint(point, Piece::SYMBOL)
   end
 end
end
```

This new code does not rely directly on the `Piece#points` method anymore, but instead, passes a position to the newly created `Piece#translated_points` to get a set of coordinates anchored by the specified position.

```ruby
class Piece
  #...
  
  def translated_points(new_anchor)
    new_x, new_y = new_anchor
    old_x, old_y = anchor

    dx = new_x - old_x
    dy = new_y - old_y
    
    points.map { |x,y| [x+dx, y+dy] }
  end
end
```

While this mapping isn't very complex, it's yet another point where I was
thinking 'gee, I should be writing tests', and a couple subtle bugs that
cropped up while implementing it confirmed my gut feeling. But with the light
visible at the end of the tunnel, I wrote an example to unify piece objects 
with the junk left on the grid from previous moves.

```ruby
game = FallingBlocks::Game.new
bent_shape = FallingBlocks::Piece.new([[0,1],[0,2],[1,0],[1,1]])
game.piece = bent_shape
game.piece_position = [2,3]
game.junk += [[0,0], [1,0], [2,0], [2,1], [4,0],
              [4,1], [4,2], [5,0], [5,1], [6,0],
              [7,0], [8,0], [8,1], [9,0], [9,1],
              [9,2]]

puts game
```

OUTPUTS:

```
==========






  #
  ##|    |
  |#||  ||
||| ||||||
==========
```

The key component that tied this all together is the `Game` object, which essentially is just a container that knows how to use a `Canvas` object to render itself.

```ruby
class Game
  def initialize
    @junk = []
    @piece = nil
    @piece_position = []
  end

  attr_accessor :junk, :piece, :piece_position

  def to_s
    canvas = Canvas.new

    junk.each do |pos|
      canvas.paint(pos, "|")
    end

    canvas.paint_shape(piece, piece_position, "#")

    canvas.to_s
  end
end
```

I made a small change to `Canvas#paint_shape` so that the symbol used to display pieces on the grid was parameterized rather than stored in `Piece::SYMBOL`. This isn't a major change and was just another attempt at moving display code away from the data models.

After all this work, we've made it back to the output we were getting out of our first example, but without the smoke and mirrors. Still, the model is not as solid as I'd hoped for, and some last minute changes were needed to bridge the gap before this code was ready to implement the two use cases I was targeting.

Since the last iteration would be a bit cumbersome to describe in newsletter form, please just "check out my final commit":http://is.gd/jbvdB for this project on github. With this new code, it's possible to get output identical to our target story through the following two examples.

### CASE 1: line_shape_demo.rb

```ruby
require_relative "falling_blocks"

game = FallingBlocks::Game.new
line_shape = FallingBlocks::Piece.new([[0,0],[0,1],[0,2],[0,3]])
game.piece = line_shape
game.piece_position = [3,3]
game.add_junk([[0,0], [1,0], [2,0], [2,1], [4,0],
              [4,1], [4,2], [5,0], [5,1], [6,0],
              [7,0], [8,0], [8,1], [9,0], [9,1],
              [9,2]])

puts game

puts "\nBECOMES:\n\n"

game.update_junk
puts game
```

### CASE 2: bended_shape_demo.rb

```ruby
require_relative "falling_blocks"

game = FallingBlocks::Game.new
bent_shape = FallingBlocks::Piece.new([[0,1],[0,2],[1,0],[1,1]])
game.piece = bent_shape
game.piece_position = [2,3]
game.add_junk([[0,0], [1,0], [2,0], [2,1], [4,0],
              [4,1], [4,2], [5,0], [5,1], [6,0],
              [7,0], [8,0], [8,1], [9,0], [9,1],
              [9,2]])

puts game

puts "\nBECOMES:\n\n"

game.update_junk
puts game
```

### Reflections

Once I outlined the story by drawing some ascii art, it took me just over 1.5 hours to produce working code that performs the transformations described. Overall, I'd call that a success.

That having been said, working on this problem was not without hurdles. While it turns out that removing completed lines and turning pieces into junk upon collision is surprisingly simple, I am still uneasy about my final design. It seems that there is considerable duplication between the grid maintained by `Game` and the `Canvas` object. But a refactoring here would be non-trivial, and I wouldn't want to attempt it without laying down some tests to minimize the amount of time hunting down subtle bugs.

For me, this is about as far as I can write code organically in a single sitting without either writing tests, or doing some proper design in front of whiteboard, or a combination of the two. I think it's important to recognize this limit, and also note that it varies from person to person and project to project. The key to writing a good prototype is getting as close to that line as you can without flying off the edge of a cliff.

In the end though, what I like about this prototype is that it isn't just an illusion. With a little work, it'd be easy enough to scale up to my initial ambition of demonstrating a free falling piece. By adding some tests and doing some refactoring, it'd be possible to evolve this code into something that could be used in production rather than just treating it as throwaway demo-ware.

Hopefully, seeing how I decomposed the problem, and having a bit of insight into what my though process was like as I worked on this project has helped you understand what goes into making proof-of-concept code in Ruby. I've not actually taught extensively about this process before, so describing it is a bit of an experiment for me. Let me know what you think!

  
> **NOTE:** This article has also been published on the Ruby Best Practices blog. There [may be additional commentary](http://blog.rubybestpractices.com/posts/gregory/044-issue-12-rapid-prototyping.html#disqus_thread) 
over there worth taking a look at.
