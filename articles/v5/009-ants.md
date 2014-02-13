*This article is based on a [heavily modified Ruby port][rubyantsim] 
of Rich Hickey's [Clojure ant simulator][hickey]. Although I didn't directly collaborate with Rich on this issue of 
Practicing Ruby, I learned a lot from his code and it provided
me with a great foundation to start from.*

Watch as a small ant colony identifies and completely consumes its four nearest
food sources:

<div align="center">
<iframe width="720" height="480"
src="//www.youtube.com/embed/f2IX1Y5o6pc?rel=0" frameborder="0" allowfullscreen></iframe>
</div>

While this search effort may seem highly organized, it is the
result of very simple decisions made by individual ants. On each
tick of the simulation, each ant decides its next action based only on its
current location and the three adjacent locations ahead of it. But 
because ants can indirectly communicate via their environment, complex 
behavior arises in the aggregate.

Emergence and self-organization are popular concepts in programming, but far too many
developers start and end their explorations into these ideas with [Conway's Game of Life][conway]. 
In this article, I will help you see these fascinating properties in a new
light by demonstrating the role they play in [ant colony optimization (ACO)][aco] algorithms.

> **NOTE:** There are many ways to simulate ant behavior, some of which can be quite useful
for a wide range of search applications. For this article, I have built
a fairly naïve simulation that is meant to loosely mimic the kind of ant
behavior you can observe in the natural world. This article *may* be useful as a 
brief introduction to ACO, but be sure to dig deeper if you are interested in
practical applications. My goal is to provide a great example of emergent 
behavior, NOT a great reference for nature-inspired search algorithms.

## Modeling the state of an ant colony

This simulated world consists of many cells: some are food sources, 
some are part of the colony's nest, and the rest are an
open field that needs to be traversed. Each cell can contain a single 
ant facing in one of the eight directions you'd find on a compass. 
As the ants move around the world, they mark the cells they visit with
a trail of pheromones that helps them find their way between their 
nest and nearby food sources. Pheromones accumulate as more ants 
travel across a given trail, but they also gradually evaporate. 
The combination of these two properties of pheromones helps 
ants find efficient paths to nearby food sources.

Subtle changes to any of these rules can yield very different outcomes, 
and finding an optimal result will necessarily involve some
experimentation. Knowing that, it makes sense for the simulator to 
have a data model that is divorced from its domain logic. Many
behavioral changes can be made without altering the
underlying data model, and that allows the `Ant`, `Cell`, and `World` constructs to
be defined as simple value objects as shown below:

```ruby
module AntSim 
  class Ant
    def initialize(direction, location)
      self.direction = direction
      self.location  = location
    end

    attr_accessor :food, :direction, :location
  end

  class Cell
    def initialize(food, home_pheremone, food_pheremone)
      self.food           = food 
      self.home_pheremone = home_pheremone
      self.food_pheremone = food_pheremone
    end

    attr_accessor :food, :home_pheremone, :food_pheremone, :ant, :home
  end

  class World
    def initialize(world_size)
      self.size = world_size
      self.data = size.times.map { size.times.map { Cell.new(0,0,0) } }
    end

    def [](location)
      x,y = location

      data[x][y]
    end

    def sample
      data[rand(size)][rand(size)]
    end

    def each
      data.each_with_index do |col,x| 
        col.each_with_index do |cell, y| 
          yield [cell, [x, y]]
        end
      end
    end

    private

    attr_accessor :data, :size
  end
end
```

These classes are somewhat peculiar in that they are very state-centric and 
do not encapsulate any interesting domain logic. Although it won't win us
object-oriented style points, designing things this way decouples the state of 
the simulated world from both the events that happen within it and the 
optimization algorithms that run against it. These objects
represent only the nouns of our system, leaving it up to their collaborators 
to supply the verbs.

## Moving around the world

The ants in this system are surprisingly limited in their behavior. On each 
and every iteration, their entire decision making process can result 
in exactly one of the following outcomes:

![Ant movement rules](http://i.imgur.com/VsBkn.png)

Most of these actions are extremely localized. Turning does not affect any
cells, while moving only affects the cell the ant currently occupies
and the one immediately in front of it. However, taking or dropping food
triggers a pheromone update, affecting every cell the ant has 
visited since the last time it updated its trails. This can have far-reaching
effects on the behavior of the rest of the colony, even though each individual
ant can only sense the pheromone levels of its own cell and the three cells
directly in front of it. While natural ants must drop pheromone
continuously as they walk, artificial ants can improve upon nature by
updating entire paths instantaneously.

An object that implements these behaviors needs to know about the structure of
the `Ant`, `Cell`, and `World` objects, but it still does not
need to know much about the core domain logic of the simulator. What we want is
an `Actor` that understands its world and how to play specific roles within it, 
but does not attempt to define the broader story arc:

```ruby
require "set"

module AntSim
  class Actor
    DIR_DELTA   = [[0, -1], [ 1, -1], [ 1, 0], [ 1,  1],
                   [0,  1], [-1,  1], [-1, 0], [-1, -1]]

    def initialize(world, ant)
      self.world   = world
      self.ant     = ant

      self.history = Set.new
    end

    attr_reader :ant

    def turn(amt)
      ant.direction = (ant.direction + amt) % 8

      self
    end

    def move
      history << here

      new_location = neighbor(ant.direction)

      ahead.ant = ant
      here.ant  = nil

      ant.location = new_location

      self
    end

    def drop_food
      here.food += 1
      ant.food   = false

      self
    end

    def take_food
      here.food -= 1
      ant.food   = true

      self
    end

    def mark_food_trail
      history.each do |old_cell|
        old_cell.food_pheremone += 1 unless old_cell.food > 0 
      end

      history.clear

      self
    end

    def mark_home_trail
      history.each do |old_cell|
        old_cell.home_pheremone += 1 unless old_cell.home
      end

      history.clear

      self
    end

    def foraging?
      !ant.food
    end

    def here
      world[ant.location]
    end

    def ahead
      world[neighbor(ant.direction)]
    end

    def ahead_left
      world[neighbor(ant.direction - 1)]
    end

    def ahead_right
      world[neighbor(ant.direction + 1)]
    end
    
    def nearby_places
      [ahead, ahead_left, ahead_right]
    end

    private

    def neighbor(direction)
      x,y = ant.location

      dx, dy = DIR_DELTA[direction % 8]

      [(x + dx) % world.size, (y + dy) % world.size]
    end

    attr_accessor :world, :history
    attr_writer   :ant
  end
end
```

Of course, now that we have crossed the line from pure data models to an object
which actually does something, it is impossible to implement meaningful behavior
without making certain assumptions that will affect the capabilities of the 
rest of the system. The `Actor` class draws two significant lines in the sand that
are easy to overlook on a quick glance:

1. Storing history data in a `Set` rather than an `Array` makes it so
that when this object updates pheromone trails, it only takes into account
what cells were visited, not how many times they were visited or in what order
they were traversed.

2. The modular arithmetic performed in the `neighbor` function treats the world
as if it were a [torus][torus], instead of a plane. This means that the
leftmost column and the rightmost column of the map are adjacent to one 
another, as are the top and bottom rows. This allows ants to easily wrap around
the edges of the map, but also establishes connections between cells that you
may not intuitively think of as being close to one another. Without a
three-dimensional visualization, it is hard to show that the top right corner of
the map and the bottom left corner are actually adjacent to one another.

Of course, the purpose of the `Actor` class is to hide these details from
the rest of the system. As long as its collaborators can operate within these 
constraints, the `Actor` object can be treated as a magic black box that knows
how to make ants move around the world and do interesting things. To see why
that is useful, check out the `Simulator#iterate` function which drives the
simulator's main event loop:

```ruby
module AntSim
  class Simulator
    # ... other functions ...

    def iterate
      actors.each do |actor|
        optimizer = Optimizer.new(actor.here, actor.nearby_places)
        
        if actor.foraging?
          action = optimizer.seek_food
        else
          action = optimizer.seek_home
        end

        case action
        when :drop_food
          actor.drop_food.mark_food_trail.turn(4)
        when :take_food
          actor.take_food.mark_home_trail.turn(4)
        when :move_forward
          actor.move
        when :turn_left
          actor.turn(-1)
        when :turn_right
          actor.turn(1)
        else
          raise NotImplementedError, action.inspect
        end
      end

      sleep ANT_SLEEP
    end
  end
end
```

Here we can see that the `Simulator` acts as a bridge that translates
the `Optimizer` object's very abstract suggestions into concrete
actions for the `Actor` to carry out. The design of the `Actor` object gives the
`Simulator` just enough control to make some small adjustments to the process,
but not so much that it needs to be bogged down with the details.

## Finding food and bringing it home

Now that we know the state of the world and how it can be manipulated, it is
time to discuss how to produce the kind of behavior that you saw in the
video at the beginning of this article. Perhaps unsurprisingly, the life of the
everyday worker ant is actually fairly mundane.

Every ant in this simulation is always either searching for food to bring back
to the nest, or trying to return home with the food it found. As soon 
an ant accomplishes one of these tasks, it immediately transitions to the other,
not bothering to take even a moment to bask in fruits of its labor. The
following outline describes what the ants in this simulation are "thinking" 
at any given point in time, assuming that they haven't managed to 
become self-aware...

**When searching for food:**

1. If the current cell has food in it and it is NOT part of the nest, 
pick up some food.

2. Otherwise, check the cell directly in front of me. If it has food in it, is
not part of the nest, and it is not occupied by another ant, move there.

3. If not, rank the three adjacent cells in front of me based
on the amount of food they contain, and how intense their `food_pheremone`
levels are. I will *usually* choose to move or turn towards the cell with
highest ranking, but I will randomly deviate from this pattern on occasion
so that I can explore some uncharted territory.

**When searching for the nest:**

1. If the current cell is part of the nest, drop the food I am carrying.

2. Otherwise, check the cell directly in front of me. If it is part of the nest,
and it is not occupied by another ant, move there.

3. If not, rank the three adjacent cells in front of me based
on whether or not they are part of the nest, and how intense their `home_pheremone`
levels are. I will *usually* choose to move or turn towards the cell with
highest ranking, but I will randomly deviate from this pattern on occasion
so that I can explore some uncharted territory.

Translating these ideas into code is very straightforward, especially
if you treat the underlying mathematical formulas as a black box:

```ruby
module AntSim
  class Optimizer
    # ...

    def seek_food
      if here.food > 0 && (! here.home)
        :take_food
      elsif ahead.food > 0 && (! ahead.home ) && (! ahead.ant )
        :move_forward
      else
        food_ranking = rank_by { |cell| cell.food }
        pher_ranking = rank_by { |cell| cell.food_pheremone }

        ranks = combined_ranks(food_ranking, pher_ranking)
        follow_trail(ranks)
      end
    end

    def seek_home
      if here.home
        :drop_food
      elsif ahead.home && (! ahead.ant)
        :move_forward
      else
        home_ranking = rank_by { |cell| cell.home ? 1 : 0 }
        pher_ranking = rank_by { |cell| cell.home_pheremone }

        ranks = combined_ranks(home_ranking, pher_ranking)
        follow_trail(ranks)
      end
    end

    def follow_trail(ranks)
      choice = wrand([ ahead.ant ? 0 : ranks[ahead],
                       ranks[ahead_left],
                       ranks[ahead_right]])

      [:move_forward, :turn_left, :turn_right][choice]
    end
    

    # ...
  end
end
```

If you understand the general idea behind this algorithm, don't worry about the
exact computations that the `Optimizer` uses unless you are
planning on researching Ant Colony Optimization in much greater detail. While I
understand what my own code is doing, I'll admit that I mostly 
cargo-cult copied the probabilistic methods 
from [Rich Hickey's simulator][hickey] while sprinkling in a few minor tweaks 
here and there. That said, if you want to see exactly how I hacked things
together, feel free to check out 
the [full Optimizer class definition][optimizer].

What I personally find much more interesting than the nuts and bolt of
*how* this algorithm works is to think about *why* it works.

## How the hive mind emerges

As we discussed in the previous section, ants are attracted to pheromone, and
that makes them more likely to follow the trails left behind by other ants than
they are to venture out on their own. However, when ants first start exploring
a new space, there are no trails to follow and so they are forced to wander
around randomly until a food source is found.

Generally speaking, ants that take a shorter path from the nest to a food
source will arrive there sooner than ants that take a longer path. If they
follow their own pheromone trail back to the nest, they will also return home
sooner than those who are traversing longer paths. By the time ants who have
taken a longer path return home, the ants on the shortest paths have already
went back out in search of additional food, which increases the pheromone levels
on their trails.

This process on its own would bias the ant colony to prefer shorter paths over
longer ones, but the optimization would be somewhat sluggish and might tend to
produce solutions that work well locally but aren't nearly as attractive
globally. To get better results, the system needs a bit of entropy thrown into
the mix.

Because the behavior of ants has a certain amount of randomness to it,
the occasional deviation from established paths are fairly common. Even if the
fluctuations are small, each tiny shortcut that allows an ant to get between two
points along a path in a shorter amount of time ultimately contributes to
finding an optimal solution. This means that even an ant who goes wildly off
course and starves to death nowhere near the nest can make a meaningful
contribution to the colony if even some tiny segment of its path serves to
shorten an existing well-worn trail.

When you add in the fact that pheromones are volatile and tend to evaporate over
time, an upper limit emerges for how much a bad path or a local optimization can
influence the colony's decision making. Evaporation is also a key part of what
allows the ants to change course when a food source is exhausted, or an obstacle
stands in the way of an established path.

Pheromone decay is something that can be modeled in many ways, but the easiest
way of simulating it is to gradually reduce the pheromone at every cell in the 
world on a regular interval. For an example of this approach, check out
`Simulator#evaporate`:

```ruby
module AntSim
  class Simulator
    def evaporate
      world.each do |cell, (x,y)| 
        cell.home_pheremone *= EVAP_RATE 
        cell.food_pheremone *= EVAP_RATE
      end
    end
  end
end
```

So if you take the basic positive feedback loop caused by pheromone attraction
and mix in a bit of probabilistic exploration and the gradual evaporation of trails, you end
up with a fairly robust optimization process. It truly is remarkable that 
these basic factors can combine to create a very
effective search heuristic, especially when you consider the fact that what
we've discussed here is only a crude approximation of the tip of the iceberg
when it comes to [Ant Colony Optimization][aco].

## Reflections

Emergent behaviors in computing problems have always fascinated me, even though I
have not spent nearly enough time studying them to understand them well. I feel
similarly about a lot of other things in life, ranging from the board game Go,
to the spread of memes throughout communities both online and offline.

There is something deep and almost spiritual in the realization that the
extremely complex behaviors can emerge from very simple systems with very few
rules, and a complete lack of central organization. It forces us to call into
question everything we experience and to wonder whether there is some elegant
explanation for it all!

[conway]: http://en.wikipedia.org/wiki/Conway%27s_Game_of_Life
[aco]: http://en.wikipedia.org/wiki/Ant_colony_optimization
[torus]: http://en.wikipedia.org/wiki/Torus
[hickey]: https://gist.github.com/1093917
[rubyantsim]: https://github.com/elm-city-craftworks/practicing-ruby-examples/tree/master/v5/009
[optimizer]: https://github.com/elm-city-craftworks/practicing-ruby-examples/blob/master/v5/009/lib/ant_sim/optimizer.rb
