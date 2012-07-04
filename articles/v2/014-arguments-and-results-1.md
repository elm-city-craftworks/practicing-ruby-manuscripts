Back in 1997, James Noble published a paper called [Arguments and Results](http://www.laputan.org/pub/patterns/noble/noble.pdf) which outlined several useful patterns for designing better object protocols. Despite the fact that this paper was written nearly 15 years ago, it addresses design problems that programmers still struggle with today. In this two part article, I will show how the patterns James came up with can be applied to modern Ruby programs.

<u>Arguments and Results</u> is written in such a way that it is natural to split the patterns it describes into two separate groups: patterns about method arguments and patterns about the results returned by methods. I've decided to split this Practicing Ruby article in the same manner in order to make it easier for me to write and easier for you to read. 

In this first installment, we will explore the patterns James lays out for working with method arguments, and in Issue 2.15 we'll look into results objects. If you read this part, be sure to read the second part once it comes out, because the two concepts complement each other nicely.

### Establishing a context 

It is very difficult to study design patterns without applying them within a particular context. When I am trying to learn new patterns, I tend to start by looking for a realistic scenario that the pattern might be applicable to. I then examine the benefits and drawbacks of the design changes within that context. James uses a lot of graphics programming examples in his paper and this is for good reason: it's an area where designing good interfaces for your objects can quickly become challenging.

I've decided to follow in James's footsteps here and use a trivial [SVG](http://www.w3.org/TR/SVG/) generator as the common theme for the examples in this article. The following code illustrates the interface that I started with before applying any special patterns:

```ruby
# image dimensions are provided to `Drawing` in cm, 
# all other measurements are done in units of 0.01 cm

drawing = Drawing.new(4,4)

drawing.line(:x1 => 100, :y1 => 100, :x2 => 200, :y2 => 250,
             :stroke_color => "blue", :stroke_width => 2)

drawing.line(:x1 => 300, :y1 => 100, :x2 => 200, :y2 => 250,
             :stroke_color => "blue", :stroke_width => 2)

File.write("sample.svg", drawing.to_svg)
```

The implementation details are not important here, but if you would like to see how this code works, you can check out the [source code for the Drawing class](https://github.com/elm-city-craftworks/pr-arguments-and-results/blob/7656768680b6a940a5ccf569fc0e0dce48a5dbfe/drawing.rb). The interface for `Drawing#line` uses keyword-style arguments in a similar fashion to most other Ruby libraries. Because keyword arguments are easier to remember and more flexible than ordinal arguments, this style of interface has become very popular among Ruby programmers. However, the more arguments a method takes, the more unwieldy this sort of API becomes. That tipping point is where design patterns about arguments come into play.

### Arguments object

As the number of arguments to a method increase, the amount of code within the method to handle those arguments tends to increase as well. This is because complex protocols typically require  arguments to be validated and transformed before they can be operated on. By introducing new objects to wrap related sets of arguments, it is possible to keep your argument processing logic somewhat separated from your business logic. The following code demonstrates how to use this concept to simplify the interface of the `Drawing#line` method:

```ruby
drawing = Drawing.new(4,4)

line1 = Drawing::Shape.new([100, 100], [200, 250])
line2 = Drawing::Shape.new([300, 100], [200, 250])

line_style = Drawing::Style.new(:stroke_color => "blue", :stroke_width => "2")

drawing.line(line1, line_style)

drawing.line(line2, line_style)

File.write("sample.svg", drawing.to_svg)
```

This approach takes a single complex method call on a single object and replaces it with several less complex method calls distributed across several objects. In the early stages of development, applying this pattern feels ugly because it involves writing a lot more code for both the library developer and application developer. However, as the complexity of the argument processing increases, the benefits of this approach begin to shine. The following example demonstrates how the newly introduced arguments objects raise the `Drawing#line` code up to a higher level of abstraction.

```ruby
def line(data, style)
  unless data.bounded_by?(@viewbox_width, @viewbox_height)
    raise ArgumentError, "shape is not within view box"
  end

  @lines << { :x1 => data[0].x.to_s, :y1 => data[0].y.to_s,
              :x2 => data[1].x.to_s, :y2 => data[1].y.to_s,
              :style => style.to_css }
end
```

The cost of making `Drawing#line` so concise is a big chunk of boilerplate code that on the surface feels a bit overkill at this stage in the game. However, it does not take a very wild imagination to see how these new objects set the stage for future extensions:

```ruby
class Point
  def initialize(x, y)
    @x, @y = x, y
  end

  attr_reader :x, :y
end

class Shape
  def initialize(*point_data)
    @points = point_data.map { |e| Point.new(*e) }
  end

  def [](index)
    @points[index]
  end

  def bounded_by?(x_max, y_max)
    @points.all? { |p| p.x <= x_max && p.y <= y_max }
  end
end

class Style
  def initialize(params)
    @stroke_width  = params.fetch(:stroke_width, 5)
    @stroke_color  = params.fetch(:stroke_color, "black")
  end

  attr_reader :stroke_width, :stroke_color

  def to_css
    "stroke: #{@stroke_color}; stroke-width: #{@stroke_width}"
  end
end
```

The interesting thing about these objects is that they actually represent domain models even though their original purpose was simply to wrap up some arguments to a single method defined on the `Drawing` object. James mentions in his paper that this phenomena is common and would call these "Found objects", i.e. objects that are part of the domain model that were found through refactoring rather than accounted for in the original design.

You might have noticed that in the previous example, I set some default values for some of the variables on the `Style` object. If you compare this to setting defaults directly within the `Drawing#line` method itself, it becomes obvious that there is a benefit here. Properties like
the color and thickness of the lines drawn to form a shape are universal properties, not things specific to straight lines only. Centralizing the defaults makes it so that they do not need to be repeated for each type of shape that the `Drawing` object supports.

### Selector object

Sometimes we end up with objects that have many methods that take similar arguments. While these methods may actually do different things, the only difference in the object protocol is the name of the message being sent. After adding a method for rendering polygons to my `Drawing` object, I ended up in exactly this situation. The following example shows just how similar the `Drawing#line` interface is to the newly created `Drawing#polygon` method:

```ruby
drawing = Drawing.new(4,4)

line1 = Drawing::Shape.new([100, 100], [200, 250])
line2 = Drawing::Shape.new([300, 100], [200, 250])

triangle = Drawing::Shape.new([350, 150], [250, 300], [150,150])

style = Drawing::Style.new(:stroke_color => "blue", :stroke_width => 2)

drawing.line(line1, style)

drawing.line(line2, style)

drawing.polygon(triangle, style)

File.write("sample.svg", drawing.to_svg)
```

Taking a look at the implementation of both methods, it is easy to see that there are deep similarities in structure between the two:

```ruby
class Drawing
  # NOTE: other code omitted, not important...

  def line(data, style)
    unless data.bounded_by?(@viewbox_width, @viewbox_height)
      raise ArgumentError, "shape is not within view box"
    end

    @elements << [:line, { :x1    => data[0].x.to_s, 
                           :y1    => data[0].y.to_s, 
                           :x2    => data[1].x.to_s, 
                           :y2    => data[1].y.to_s,
                           :style => style.to_css }] 
  end

  def polygon(data, style)
    unless data.bounded_by?(@viewbox_width, @viewbox_height)
       raise ArgumentError, "shape is not within view box"     
    end

    @elements << [:polygon, { 
      :points => data.each.map { |point| "#{point.x},#{point.y}" }.join(" "),
      :style  => style.to_css
    }]
  end
end
```

To make this code more DRY, James recommends converting our arguments object into what he calls a selector object. A selector object is an object which uses similar arguments to do different things depending on the type of message it is meant to represent. James recommends using double dispatch or multi-methods to implement this pattern, but that approach is not appropriate for Ruby because the language does not provide built-in semantics for function overloading. The good news is that he also mentions that inheritance can be used as an alternative, and in this case it was a perfect fit.

To simplify and clean up the previous example, I introduced `Line` and `Polygon` which inherit from `Shape`. I then combined the `Drawing#line` method and `Drawing#polygon` method into a single method called `Drawing#draw`. The following example demonstrates what the API ended up looking like as a result of this change:

```ruby
drawing = Drawing.new(4,4)

line1 = Drawing::Line.new([100, 100], [200, 250])
line2 = Drawing::Line.new([300, 100], [200, 250])

triangle = Drawing::Polygon.new([350, 150], [250, 300], [150,150])

style = Drawing::Style.new(:stroke_color => "blue", :stroke_width => 2)

drawing.draw(line1, style)
drawing.draw(line2, style)
drawing.draw(triangle, style)

File.write("sample.svg", drawing.to_svg)
```

The changes to the API are small but make the code a lot easier to read. This rearrangement introduces even more objects into the system, but simplifies the protocol between those objects. In large systems, this leads to greater maintainability and learnability at the cost of having a few more moving parts.

In order to implement this new interface, some non-trivial changes needed to be made under the hood. You can check out the [exact commit](https://github.com/elm-city-craftworks/pr-arguments-and-results/commit/47924901552d0509f97a3083737709980139feba) to see the details about what changed implementation-wise between this example and the last one, but most of the changes were just boring housekeeping. The general idea is that the `Drawing#draw` method now simply asks each shape object to represent itself as a hash which ultimately ends up getting converted into an XML tag within the SVG document. As an example, here is what the definition for the `Line` object looks like:

```ruby
class Drawing
  class Line < Shape
    def to_hash(style)
      { :tag_name => :line,
        :params => { :x1    => self[0].x.to_s,
                     :y1    => self[0].y.to_s,
                     :x2    => self[1].x.to_s,
                     :y2    => self[1].y.to_s,
                     :style => style.to_css } }
    end
  end
end
```

As you can imagine, the `Polygon` object uses a similar approach and this general pattern would be applicable for new types of shapes as well.


### Curried object

While method arguments exist to allow us to vary the objects we pass in, its not uncommon for the same method to be called many times with some of its arguments being held constant. In fact, all of the examples in this article have shown the same `Style` object being passed to the same method again and again, with only the shape varying. This has resulted in some repetitive code that looks ugly, and could be improved.

James recommends creating a curried object to deal with this sort of problem. The curried object acts as a lightweight proxy over the original object, but keeps the constant data stored in variables so that you do not need to keep repeating it. The following code applies this concept to clean up our previous example:

```ruby
line1 = Drawing::Line.new([100, 100], [200, 250])
line2 = Drawing::Line.new([300, 100], [200, 250])

triangle = Drawing::Polygon.new([350, 150], [250, 300], [150,150])

drawing = Drawing.new(4,4)
style   = Drawing::Style.new(:stroke_color => "blue", :stroke_width => 2)
pen     = Drawing::Pen.new(drawing, style)

pen.draw(line1)
pen.draw(line2)
pen.draw(triangle)

File.write("sample.svg", drawing.to_svg)
```

While introducing the new `Pen` object requires a change in the calling code so that `Pen#draw` gets called instead of `Drawing#draw`, no change to the implementation of `Drawing` was needed to introduce this new object. The following class definition will do the trick:

```ruby
class Drawing
  class Pen
    def initialize(drawing, style)
      @drawing, @style = drawing, style
    end

    def draw(shape)
      drawing.draw(shape, style)
    end
    
    private

    attr_reader :drawing, :style
  end
end
```

In this particular case, `Pen` is easy to write because the interface on `Drawing` is so small. In more complicated cases, it would make sense to use some of Ruby's metaprogramming features to implement a dynamic proxy of some sort. However, if you find yourself simultaneously facing a broad interface that has arguments that often remain constant in many of its functions, you may want to evaluate whether you have a flawed design before going down that road.

An interesting thing to note is that curried objects are not necessarily limited to arguments that remain constant. This pattern can also be applied in situations where method calls made in sequence have a clear pattern in the way that one or more arguments are varied. The example James gives in his paper describes some logic for a text editor in which lines of text are rendered to the screen with all the same style attributes from line to line, but with the line number incremented as each new line is rendered. Taking inspiration from that example, I decided to build a simple turtle graphics system to demonstrate how curried objects can be used for predictably varying arguments as well as constant arguments. The code below generates an image of an X when run:

```ruby
drawing = Drawing.new(4,4)
style   = Drawing::Style.new(:stroke_color => "blue", :stroke_width => 2)
turtle  = Drawing::Turtle.new(drawing, style)

turtle.move_to([0, 400])

turtle.pen_down
turtle.move_to([400, 0])

turtle.pen_up
turtle.move_to([0,0])

turtle.pen_down
turtle.move_to([400,400])

File.write("sample.svg", drawing.to_svg)
```

The implementation code to make the previous example work was very easy to write and required no changes to the rest of the system:

```ruby
class Drawing
  class Turtle
    def initialize(drawing, style)
      @drawing  = drawing
      @style    = style
      @inked    = false
      @position = [0,0]
    end

    def move_to(next_position)
      if inked
        drawing.draw(Line.new(position, next_position), style)
      end
      
      self.position = next_position
    end

    def pen_up
      self.inked = false
    end

    def pen_down 
      self.inked = true
    end

    private

    attr_reader   :drawing, :style
    attr_accessor :position, :inked 
  end
end
```

After taking a look at the finished `Turtle` object, I did wonder a little bit about whether the idea of a curried object in Ruby is nothing more than an ordinary object making use of object composition. However, because the name of the pattern is helpful for describing the intent of this sort of object in a succinct way, it may be a good label for us to use when discussing the merits of different design options.

### Reflections

Applying these various argument patterns to a realistic example made it much easier for me to see the power behind these ideas. I have gradually picked up bits and pieces of the various techniques shown here before reading this paper largely due to my trial and error work on the Prawn PDF generator. 

In lots of places in Prawn, we let hash arguments grow to an insanely large size and it created a lot of problems for us. We also ignored using curried objects in a lot of places by instead placing instance variables directly on the target objects and then mutating the state within them over time to vary things. This led to complicated transactional code and made it easy for things to end up in an inconsistent state. The solutions to these problems tended to be refactorings that are quite similar to what you've seen in this article, even if we didn't call them by a special name at the time.

Still, I do have some concern that these patterns might be overkill for any interfaces that you are reasonably sure won't get too complex over time. If we apply these patterns overzealously, you might end up needing to go through level after level of indirection just to accomplish anything useful, and that will make Ruby start to feel like Java. However, it seems like using some sort of formalized arguments object is obviously beneficial for highly complex interactions, and likely to be at least somewhat useful for medium complexity protocols as well.

No matter what the complexity of the problem I was working on, it's unlikely that I would make it so that the application developer needed to jump through so many hoops just to use my library. Instead, I would probably build a simple facade or DSL that made their life easier, even if a rich object structure was lurking under the hood. If I were really building an SVG generator, I might end up building a DSL for it that looked something like this:

```ruby
drawing do
  style :stroke_color => "blue", :stroke_width => 2 do
    line    [100, 100], [200, 250]
    line    [300, 100], [200, 250]
    polygon [350, 150], [250, 300], [150,150]
  end

  save_as "sample.svg"
end
```

If I implemented this as a thin veneer on top of code similar to what we ended up with in this article, I think that would be a pretty well designed library. The end user gets convenience for the normal case, but the underlying system would be easier to maintain, test, and learn. It would also give the user flexibility to interact with the system in ways I didn't anticipate.

Be sure to tune in next week for the second part of this article, where I'll focus on the results side of the method interface. Until then, I'd love to hear any questions or thoughts you have about this topic.

