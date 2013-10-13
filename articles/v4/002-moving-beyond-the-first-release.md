In Issue 2.10, I described the path I took [from requirements discovery to
release](http://practicingruby.com/articles/10) for a small game I created,
including various corners I had cut to get an initial release out the door.
While that article was very well received, it left an important question
unanswered: **How do you transform a prototype into a product?**

To answer that question, I have built a new little game for us to play with. But
this time around, rather than outlining the path from the idea phase to a basic
proof-of-concept, I will instead focus on what it takes to turn a rough prototype into a
game that might actually be fun to play. Because that is a pretty big topic, I
have split this article into two parts. This first part describes the initial
prototype and my plans for improving it, and the second part will reflect on the
challenges I encountered while trying to make those improvements. 

### Blind: A Minesweeper-inspired game with a massive twist

The game I created for this article is based on a very simple concept:
Navigating a mine field in search of an exit. While the premise is a bit
different from the classic Minesweeper game, the basic idea of carefully trying
to figure out where the mines are without getting yourself blown up is
preserved. The graphic below gives a rough approximation what the structure of
the Blind world is like:

<div align="center">
<img src="http://i.imgur.com/lvRaj.png">
</div>

The game starts with the player positioned at the center of the safe zone, a
buffer zone that exists to prevent mines from being detonated as soon as the
player spawns. Within the mine field itself, a large quantity of mines are
randomly spawned and there is no way for the player to know in advance exactly
where they will be. This area is also where the exit gets randomly positioned,
forcing the player to navigate the mine field to find their way out of the
level. 

There is one way to win and two ways to lose a game of Blind. As you might imagine,
getting too close to a mine is one of the ways to lose; the other is to wander off
the edge of the map into deep space. To do this, the player needs to ignore a
pretty obvious warning sign, but this loss condition helps make sure the player
stays within a well defined perimeter throughout the game. The only way to win
is to find the exit, which can be easy or hard depending on the positions and
quantities of mines in the minefield.

From this description, you might be imagining some sort of simple
two-dimensional arcade game, complete with a hard to control little ship
(perhaps similar to Asteroids), but as I mentioned before, this game has a
massive twist: It has no graphics at all! Instead, it relies entirely on
[positional audio](http://en.wikipedia.org/wiki/3D_audio_effect) to represent its world. To get a taste of what the gameplay is like, grab a pair of headphones and play the video shown below.

> **NOTE:** I have turned on the debugging output so that this video can be played
on mute by those who either dislike loud noises or are reading this article
in a place where they can't play audio. However, it is worth noting that the
game is designed to be played without any visual feedback at all,
and that while testing it I have typically played with my eyes closed! 

<div align="center">
<iframe width="640" height="480" src="//www.youtube.com/embed/cM0WHWgdmQk" frameborder="0" allowfullscreen></iframe>
</div>

The "audio only" twist is enough to inject some excitement into a very boring
game concept, but right now the game is not particularly enjoyable to play because it is poorly
balanced. I will talk about ideas I have for improving that later, but for now
we should move on to discuss the implementation code that got me to the point
where I could make this short video.

### An overview of Blind's initial implementation

I was originally going to use Blind as a demonstration how to implement
layered applications that have clean separation between their business logic and
UI code, so the code quality is better than what I usually start out with in proof-of-concept projects. 
While you might be able to read the [full implementation]() without too much effort, 
the following outline will give you an overview of how the codebase is structured without 
dragging you too far out into the weeds.

**[Blind::Game]() triggers events in response to the player's movements**

One decision I made early on in the process of building Blind was that I wanted
the main `Blind::Game` object to be based on a publish/subscribe model. Whenever
I have worked on games in the past, I have always struggled to figure out how to
encapsulate the rules without writing extremely brittle code, and this time
around I think I found a nice happy medium. 

If you look at the `Game#move`
method below, you can see that while it captures all the different kinds of
events that can occur within the game, it leaves it up to someone else to
determine what should happen in response to those events. This flexibility will
hopefully come in handy as the game rules evolve over time.

```ruby
module Blind
  class Game
    # ...
    
    def move(dx, dy)
      x,y = world.current_position.to_a

      r1 = world.current_region
      r2 = world.move_to(x + dx, y + dy)

      if r1 != r2
        broadcast_event(:leave_region, r1)
        broadcast_event(:enter_region, r2)
      end

      mines = world.mine_positions

      if mines.find { |e| world.distance(e) < MINE_DETONATION_RANGE }
        broadcast_event(:mine_detonated)
      end

      if world.distance(world.exit_position) < EXIT_ACTIVATION_RANGE
        broadcast_event(:exit_located)
      end
    end
  end
end
```

While I will show some more examples of this event system when we discuss Blind's
presentation layer, the following tests hint at how the publish/subscribe
system works. In a nutshell, the `Game#on_event` method sets up callbacks that get executed whenever `Game#broadcast_event` is called with a matching key.

```ruby
  it "must trigger an event when a mine is detonated" do
    detonated = false

    game.on_event(:mine_detonated) { detonated = true }

    mine = world.mine_positions.first

    game.move(mine.x - Blind::Game::MINE_DETONATION_RANGE, mine.y)

    refute detonated, "should not be detonated before " +
                      "player is in the mine's range"

    game.move(1, 0)

    assert detonated, "should detonate when player is in the mine's range"
  end
```

While I am fairly happy with the implementation of `Blind::Game`, both the
implementation code and the tests hint at the current dependency on a pair of
magic numbers stored in the `EXIT_ACTIVATION_RANGE` and
`MINE_DETONATION_RANGE` constants. This was done purely for the sake of
convenience, but I imagine they will need to be parameterized at some point in
future to make the game rules more customizable.

Another potential pitfall of this design is that its open-ended flexibility may
prove to be a double edged sword. I will be looking at this closely as I
increase the complexity of the game rules, but what I want to avoid is having a
big chunk of the game logic spill over into the client code unnecessarily. This
wasn't a major concern with the initial implementation, but it is definitely
something to look out for later.

Lastly, the event system as it currently is implemented assumes that there is a
single subscriber for each published event. This is an arbitrary limitation that
can easily be lifted, but right now is a limitation to be aware that may
need to be dealt with later.

**[Blind::World]() models the layout of the game world**

In practice, the `Blind::World` class has been working out reasonable well.
However, it is a bit of a structural method due to its broad spectrum of
responsibilities. In particular, `Blind::World` can be used to:

* Track where the mines, exit, and player are in the world
* Compute the distance between the player and any other object in the world
* Determine the region the player is currently in
* Move the player to an arbitrary location in the world
* Generate random locations within the minefield

While you might be able to group some of these concepts together, it is clear
that when taken as a set, these features fail to represent a single cohesive
object. In particular, the methods provided by this object don't have much in
common when it comes to the level of abstraction they operate at. For example,
the `World#distance` method and the `World#random_minefield_position` have so
little in common that it is hard to look at them side by side without getting
a bit of a headache.

```ruby
module Blind
  class World

    # simple delegation to the underlying Point object
    def distance(other)
      current_position.distance(other)
    end

    # ...

    # a non-trivial trigonometric function
    def random_minefield_position
      angle = rand(0..2*Math::PI)
      length = rand(MINE_FIELD_RANGE)

      x = length*Math.cos(angle)
      y = length*Math.sin(angle)

      Blind::Point.new(x.to_i,y.to_i)
    end
  end
end
```

Another major limitation of the `Blind::World` object is that similar to 
`Blind::Game`, it is chock full of magic numbers representing the sizes
and position of the various regions. While it might be reasonable to 
provide sensible defaults, locking these values down to exact numbers
will make world customization harder. I imagine this is something 
I will need to refactor sooner rather than later if I want to make
the game more interesting to play.

With all of these structural problems, you might expect the `Blind::World`
object to be quite cumbersome to work with, but so far I haven't really
had problems with it. I expect that the next round of improvements to Blind 
will change that, but I prefer to refactor based on needs rather than
gut feelings about what a good design might look like.

**[Blind::Point]() implements a simple generic point structure**

This object is simple enough where you can read its full implementation before we discuss it further:

```ruby
require "matrix"

module Blind
  class Point
    def initialize(x,y)
      @data = Vector[x,y]
    end

    def x
      data[0]
    end

    def y
      data[1]
    end

    def distance(other)
      (self.data - other.data).r
    end

    def ==(other)
      distance(other).zero?
    end

    def to_a
      [x,y]
    end

    def to_s
      "(#{x}, #{y})"
    end

    protected
    
    attr_reader :data
  end
end
```

The idea of having a generic class for representing points makes perfect sense for this game, because we need to do point math all over the place. I like that `Blind::Point` is an immutable object, because it reduces the possibility for weird corruptions to happen. However, the current implementation is not really putting the dependency on Ruby's `Vector` object to good use. I had originally built the class this way because I expected to be doing a lot of calculations, but then somehow got in the habit of manually doing the math on the individual components. 

I assume that as Blind gets more complex, I will do more and more point math, and that will inspire me to delegate a few more operations to the `Vector` object. I may also refactor several of Blind's API calls to take `Blind::Point` objects rather than explicit x and y arguments, which would encourage better use of this object. This kind of inconsistency is common in the early stages any project, because the boundary lines between the various objects in the system have not quite solidified yet. The good news is that these tensions tend to work themselves out gradually over time.


**[Blind::UI::JukeBox]() is responsible for constructing the various sounds used in the game**

The highly dynamic nature of the sounds in Blind made it worthwhile to introduce a small abstraction for loading and manipulating audio files. The example below demonstrates how both simple and complex sounds can be created by `Blind::UI::JukeBox`:

```ruby
module Blind
  module UI
    JukeBox = Object.new
    
    class << JukeBox 
      def explosion
        new_sound("grenade")
      end

      def phone(position)
        new_sound("telephone") do |s|
          s.pos     = [position.x, position.y, 0]
          s.looping = true

          s.play
        end
      end

      def mines(positions)
        step      = 0
        step_size = 1/positions.count.to_f

        positions.map do |pos|
          new_sound("beep") do |s|
            s.pos     = [pos.x, pos.y, 1]
            s.looping = true
            s.pitch   = step + rand(step_size)

            s.play

            step += step_size
          end
        end
      end

      # ... several other sounds
    end
  end
end
```

One thing I like about this code is that it gave me a chance to use one of my favorite techniques: hiding ugly code via block-based APIs. While it looks pretty, the `JukeBox#new_sound` method is in reality nothing more than a tiny bit of syntactic sugar built on top of the underlying `Ray::Sound` object:

```ruby
module Blind
  module UI
    JukeBox = Object.new
    
    class << JukeBox
      def new_sound(name) 
        filename = "#{File.dirname(__FILE__)}/../../../data/#{name}.wav"
        
        Ray::Sound.new(filename).tap do |s|
          yield s if block_given?
        end
      end
    end
  end
end
```

When I had first built this method, it was designed to store the `Ray::Sound` objects in a hash that was keyed by the sound name. However, I eventually ended up deciding that it'd be best to let the client determine if and how sound objects should be cached, and so this method (and the `JukeBox` object as a whole) became a bit more simple as a result of that. Of course, it does complicate things for the `Blind::UI::GamePresenter` object, which has essentially become a dumping ground for all the functionality that didn't fit well in the other components that Blind is made up of.

**[Blind::UI::GamePresenter]() bridges the gap between the game logic and the
UI**

Similar to `Blind::World`, the `Blind::UI::GamePresenter` object suffers from a
bit of an identity crisis. On the one hand, some of its methods do look like
simple presentation-related features:

```ruby
module Blind
  module UI
    class GamePresenter
      def lose_game(message)
        silence_sounds

        sound = sounds[:explosion]
        sound.play

        self.game_over_message = message
      end

      def win_game(message)
        silence_sounds

        sound = sounds[:celebration]
        sound.play

        self.game_over_message = message
      end
    end
  end
end
```

However, there are just as many examples of methods that seem to be tacked
on to this object that perhaps would have been better off on another object:

```ruby
module Blind
  module UI
    class GamePresenter

      # requires domain knowledge about Blind::UI::JukeBox
      def silence_sounds
        sounds.each do |name, sound|
          case name
          when :mines
            sound.each { |s| s.stop }
          else
            sound.stop
          end
        end
      end

      # touches every attribute provided by Blind::World
      def to_s
        "Player position #{world.current_position}\n"+
        "Region #{world.current_region}\n"+
        "Mines\n #{world.mine_positions.each_slice(5)
                         .map { |e| e.join(", ") }.join("\n")}\n"+
        "Exit\n #{world.exit_position}"
      end
    end
  end
end
```

And just for good measure, `Blind::UI::GamePresenter` implements a few
methods that seem to be closer to the logical layer rather than the
presentation layer:

```ruby
module Blind
  module UI
    class GamePresenter
      
      # if we want to change a game rule, we'd need to update
      # the GamePresenter object. That seems a bit strange.
      def setup_events
        game.on_event(:enter_region, :danger_zone) do
          self.in_danger_zone = true
        end

        game.on_event(:leave_region, :danger_zone) do
          self.in_danger_zone = false
        end

        game.on_event(:enter_region, :deep_space) do
          lose_game("you drifted off into deep space! you lose!")
        end

        game.on_event(:mine_detonated) do
          lose_game("you got blasted by a mine! you lose!")
        end

        game.on_event(:exit_located) do
          win_game("you found the exit! you win!")
        end
      end

      # this triggers a "SURPRISE MATH ATTACK!" as in Blind::World
      def detect_danger_zone
        if in_danger_zone
          min = Blind::World::DANGER_ZONE_RANGE.min
          max = Blind::World::DANGER_ZONE_RANGE.max

          sounds[:siren].volume = 
            ((world.distance(world.center_position) - min) / max.to_f) * 100
        else
          sounds[:siren].volume = 0
        end
      end
    end
  end
end
```

This extremely messy design is a consequence of trying to make good design
decisions elsewhere. Whenever I was in doubt about whether something was a
logical concern or a presentation concern, I error on pushing the code out of
the domain models and into the UI. This helped the objects which implemented
pure game logic stay simple and lean, but without finding a good place for all
this other code to go, it ended up getting slapped together in a haphazard 
"procedural programming with objects" style. I am hoping to clean up this
object eventually, but I didn't have any brilliant ideas for how to do so
before publishing this article.

**The [bin/blind]() executable implements trivial UI boilerplate code**

The main benefit of the `Blind::UI::GamePresenter` class is that it makes it
possible for the Ray-based UI code to be almost entirely logic-free. This 
leads to a very clear main program loop:

```ruby
always do
  if game.finished?
    message = game.game_over_message
  else
    game.detect_danger_zone
    
    game.move( 0.0, -0.2) if holding?(:w)
    game.move( 0.0, 0.2)  if holding?(:s)
    game.move(-0.2, 0.0)  if holding?(:a)
    game.move( 0.2, 0.0)  if holding?(:d)

    position = game.player_position

    Ray::Audio.pos = [position.x, position.y, 0]
  end
end
```

My hope is that I will be able to preserve the simplicity of this runner file
even if I end up having to radically restructure the `Blind::UI::GamePresenter`
object. Because the interface used by this script is very narrow, I don't expect
that will be a problem.
 
### What would make Blind a more enjoyable game?

Now that I have given you a ton of context about the various strengths and
weaknesses of Blind's codebase, we can talk about some ways to improve its
gameplay.

**Customizable world maps**

While randomization can help make games have a higher replay value, full
randomization can result in a pretty inconsistent gaming experience. I would
like to add either a mechanism for loading in pre-defined world maps, or
build a more customizable random world generator that allows to define
a bunch of different factors that affect gameplay.

> **CHALLENGES:**  In order to implement this feature, I am going to need to deal with reducing
the dependency on hard-coded numeric values throughout the system. I will
want to be able to control things like the size of the regions or the blast
radius of a mine, and currently those things are not configurable at runtime.

**Level-based organization**

Once I have the ability to support multiple different maps in the game, I would
like to be able to chain them together in a sequence to form levels. This will
make it possible to start with easy maps and progress to harder ones, which may
be a bit less of a disorienting experience for the player than the game
currently is.

> **CHALLENGES:** This feature does not require a ton of rework to the base
implementation, because we will be to easily modify the handler for the `:exit_located`
event and have it advance to the next level rather than end the game. However, I
want to make sure to re-think the `Blind::UI::GamePresenter` object before doing
this so that I can avoid accumulating even more logic in the wrong place.

**Multiple lives per game**

Having to start over from the beginning will become more and more annoying as
more levels are added, so I will introduce the concept of "lives" in some form
or another. I haven't decided yet exactly what the mechanics for this feature
will be like, but it should make death somewhat less tragic and irritating
for the player.

> **CHALLENGES:**  Similar to adding support for levels, this probably won't 
require many structural changes to the game's current implementation. But as
I mentioned before, we need a better home for our event handling code moving
forward.

**Moving enemies that chase the player**

Having to slowly navigate a mine field in two dimensional space is
nerve-wracking enough, but getting chased through one would be terrifying!
I want to add some sort of flying baddies that will chase the player around
the minefield. There are lots of different ways I could possibly implement
this, but those are more game design questions moreso than technical questions.

> **CHALLENGES:** Adding more game elements that require their position to be 
updated will force me to think harder about the limitations of the 
current `Blind::Point` class and the overall design of 
the `Blind::World` class. I may need to end up refactoring both of
those objects, depending on how complex the functionality for these new
game elements are. Additionally, new event types and event handlers will
be created, and new sounds will need to be added. In other words, making
this change will force me to touch pretty much every object in the system.

**A defense mechanism for the player**

I would like to add some way for the player to protect themselves from harm.
This will likely be some sort of limited-use shield that is effective against
mine detonations, flying baddies, or both.

> **CHALLENGES:** Adding this functionality will require some sort of new event
type, and probably some modification to existing events. It will involve less
rework than the flying baddies feature, but will require more changes to the
existing system than most of the other features I have proposed. The good news
is that apart from those changes, the feature itself should be easy to
implement.

I may not get around to implementing all of these features by the time Issue 4.3
is published, but I will definitely tackle at least a few of them between now
and then. Regardless of how things turn out, I think we will end up with some
interesting problems to discuss.

### Some things you can do to help me

**If you have a few seconds to spare** and know someone who has some experience
with game development, please share this article and ask them to get in touch
with me.

**If you have a few minutes to spare**, please share your thoughts about this
article. I would be happy to hear whatever is on your mind, whether it is about
the game itself, its codebase, or just a gut reaction to something I have said
in this article.

**If you can spare an extra hour or two of your time**, please pull down the
code and try to get the game up and running, and then take a closer look
at its implementation. Once you have done that, get in touch with me with
any questions and suggestions about points I didn't manage to cover in
this article.

And lastly, if you want to stay on top of changes as I work towards implementing
the code that will be discussed in Issue 4.3, please hang out in the
**#mendicant** channel on Freenode. I may occasionally ask the folks there to test things
for me or give me feedback on small snippets of code while I am working.
