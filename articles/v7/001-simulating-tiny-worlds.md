As programmers we are very comfortable with the idea of using
software to solve concrete problems. However, it is easy to
underestimate the role that code can play in problem-solving itself, and that blindspot can hinder our creativity in a number of ways.

In this article, I will walk you through some fun examples that demonstrate how code can be used as an exploratory thinking tool, and then reflect upon how that kind of workflow might be applied to everyday programming tasks.

## Setting the stage

The source of my motivation for writing on this topic is the [StarLogo](http://education.mit.edu/starlogo/) programming environment and Mitchel Resnick's excellent book "Turtles, Termites, and Traffic Jams", both of which illustrate the potential for software to be used as a mind-expanding thinking tool.

As the title of the book implies, StarLogo is an environment that facilitates simplistic modeling of scenarios that occur in the natural world. The purpose of the tool is not to create environments that closely emulate reality, but instead, to encourage exploration and discovery in simple, tightly constrained microcosms. Apart from being an intellectual curiosity, this sort of toolset provides a powerful way to intuitively experience deep concepts that range from self-organization and emergent behavior to massive parallelism.

In the spirit of exploration, I won't attempt to make a case for those claims by way of a top-down explanation. Instead, we'll now walk through a few scenarios that are easily implemented using StarLogo-style modeling. The examples I've chosen are based on ideas from StarLogo and Resnick's book, but I have ported them to JRuby to allow you to explore the concepts without having to familiarize yourself with a new environment first. The engine I built is called [Terrarium](https://github.com/sandal/terrarium), and it is very much a rough prototype, but it should still be good enough to introduce you to these ideas with minimal friction.

## Scenario I: Forest fires

The environment in the StarLogo model consists of a two-dimensional grid of "patches", which are in some ways similar to cellular automata models such as Conway's Game of Life. 

Using only patch color to represent state, we could apply the following ruleset to simulate a rough sketch of a forest fire:

1. Start by building a forest. For the sake of simplicity, we can begin with an empty grid and then randomly paint some of its patches green.

2. To ignite our fire, we can pick a random patch in the grid and paint it red.

3. Each green patch then needs to repeatedly check to see if any of its neighbors are red, and if so, it becomes red itself, spreading the fire.

Applying these three trivial rules results in the following behavior:

![](http://i.imgur.com/MtAbXPF.gif)

Although this animation should be fairly straightforward to understand, it is worth pointing out one small detail about the geometry of a StarLogo-style world: rather than being an infinite grid like Conway's Game of Life, it is a torus, where the left side is connected to the right, and the top is connected to the bottom. This explains why the fire (which starts at the bottom of the screen) quickly overflows onto the top of the screen in this animation.

The code that was used to generate this visualization is shown below, and is nearly a direct translation of the rules shown above:

```ruby
Terrarium::Scenario.define do
  # Rule 1: Build the forest
  patches do
    with_probability(0.5) { set_color :green } 
  end

  # Rule 2. Start the fire
  random_patch { set_color :red }

  # Rule 3. Spread the fire
  patches! do
    if color == :green && neighbors.any? { |e| e.color == :red }
      set_color :red
    end
  end
end
```

It is here where you can catch the first glimpse of what I meant by "code as a thinking tool". With Terrarium as our engine and StarLogo-style data modeling, we don't need to think at all about the structure or inner workings of our program, but instead can immediately turn our ideas into code. This takes what would cost us hours in upfront modeling and reduces it to minutes of effort.

Being able to work at this very high level of abstraction allows us to try variations and experiments as soon as we think of them. A simple idea to try out with this model is to see how the fire spreads at various levels of tree density. You will find that at 50% (which is what is shown above), the fire will pretty much always spread across the forest, but at 30%, the opposite is true. Is there a critical tipping point between those two bounds? If so, why is it there? These are the kinds of thoughts that arise when you can focus on ideas rather than code.

## Scenario II: Infectious disease

As you may have guessed from the name of the language, StarLogo also implements the *turtle graphics* model found in the Logo programming language. Both languages were developed by same research group at MIT, and so if you are familiar with Logo turtles, you will find that StarLogo's creatures move around the world in a similar way to their classical ancestors.

However, that is where the similarities end. While the average Logo turtle lives a solitary life, StarLogo's creatures can be commanded en-masse, in groups of hundreds or thousands. Where the Logo turtle is mostly used for drawing lines (albeit in some very clever ways), the StarLogo creature is capable of having much more complex interactions with its world, including the other creatures in it.

Take for example the problem of modeling the spread of a contagious disease through a population of creatures. If we allow ourselves to paper over any inconsistencies with reality by using a bit of imagination, the following rules are sufficient for emulating this scenario:

1. Arrange a group of healthy creatures into a crowd
3. Infect some of the creatures with the disease
4. Allow the creatures to slowly move about their world
5. The disease will spread from sick to healthy creatures whenever they come into contact with each other.
6. After a set period of "sick time", the creature will either die or recover, based on probability. (Recovered creatures can be re-infected if they come into contact with sick creatures, dead creatures simply disappear.)

When applied to a population of 200 StarLogo creatures, these rules produce a pattern similar to what is shown in the following animation:

![](http://i.imgur.com/dZ6czuf.gif)

Here we see the disease quickly spreading from a few infected individuals to the majority of the population. However, the rate of infection then dampens due to the following factors:

* As the creatures wander around, they become less densely packed together, which reduces the frequency at which they transmit disease to one another.

* If a creature eventually dies from an infection, that stops it from continuing to spread the disease, because it gets removed from the world upon its death.

* If the creature recovers, it can be reinfected, but by then the creatures have already spread out enough to prevent rapid chain reactions from occuring.

All of these conditions are effected by a number of variables, including population size, population density, duration of sick time, number of initially infected creatures, speed of movement of the creatures, and the probability of death vs. recovery in the infected population. In addition to this, the whole system is subject to some degree of fluctuation due to the randomness in both the movement and initial layout of the population.

Taking a purely analytical approach towards thinking through the relationships between all of these variables would be a challenging task to say the least. However, it does not take much specialized knowledge at all to model this problem using StarLogo-style creatures. In fact, the code below is all you need to implement this scenario. Try reading it one rule at a time while looking at the animation, and you should be able to piece together the main concepts even if you've never heard of StarLogo before reading this article:

```ruby
Terrarium::Scenario.define do
  healthy_color = :cyan
  sick_color    = :yellow

  initial_population = 200
  crowd_range        = 5..15
  sick_time          = 5
  infection_density  = 0.02
  movement_speed     = 0.2
  
  create_creatures(initial_population)
  
  # rule 1: arrange a group of healthy creatures into a crowd
  creatures do 
    lt rand(0..359)
    fd rand(crowd_range)

    data[:sick_time] = 0
    set_color healthy_color
  end

  # rule 2: infect some creatures
  creatures do
    with_probability(infection_density) do
      set_color sick_color
      
      data[:sick_time] = sick_time
    end
  end

  # rule 3: allow the creatures to move about randomly
  creatures! do
    lt rand(1..40)
    rt rand(1..40)

    fd movement_speed
  end

  # rule 4: spread disease on contact
  creatures! do
    next unless color == healthy_color

    if nearby_creatures.any? { |e| e.color == sick_color }
      set_color sick_color
      
      data[:sick_time] = sick_time
    end
  end

  # rule 5: recover or die based on probability 
  creatures!(1) do
    next unless color == sick_color

    if data[:sick_time] > 0
      data[:sick_time] -= 1 
    else
      coinflip ? set_color(healthy_color) : destroy
    end
  end
end
```

Because it's the live interactions in this system that are complex and not its rules, you cannot easily predict the patterns that will emerge from this program by simply reading its source code. However, by repeatedly running the program and testing various assumptions you have about the system, you can rapidly gain an intuitive sense for the patterns that arise. In that sense, exploratory programming environments can have an effect similar to that of plotting a mathematical formula: although they can't give you a precise answer to your question, they can very quickly communicate the main points of a story.

## Scenario III: Rabbits in a cabbage patch

As you may have already guessed, StarLogo's data model doesn't just give you creatures and patches, but it also supports interactions between the two. Because both the creatures and patches can encapsulate arbitrarily complex data, and because StarLogo provides a solid API for various kinds of common tasks, the richness of behavior that can be expressed through these interactions is mind boggling.

The full StarLogo environment can tackle problems like ant foraging behavior with ease, a problem that I labored with for weeks and spent two issues of Practicing Ruby on ([Issue 5.8](https://practicingruby.com/articles/92) and [Issue 5.9](https://practicingruby.com/articles/93)). However, the features I've ported from StarLogo into the Terrarium project are somewhat limited, so we'll tackle a more basic scenario that will still give you a sense of how creatures and patches can interact with one another.

We'll now take a stab at implementing a simple ecosystem in which hungry rabbits wander around doing what rabbits tend to do: eating, procreating, and dying. This is the sort of predator/prey modeling problem that you might find on a school math test, but we'll approach it informally rather than brushing up on our differential equations.

Here are the rules that will get our ecosystem up and running:

1. Create a cabbage patch by randomly coloring some patches green
2. On each iteration of the simulation, give each patch a small chance to sprout cabbage, facilitating regrowth.
3. Arrange a crowd of rabbits in the cabbage patch.
4. Allow the rabbits to wander randomly around the cabbage patch
5. Rabbits eat any cabbage they encounter. This sets the patch color back to black, and increases the energy of the rabbits.
6. Rabbits gradually lose energy over time. If their energy is fully depleted, they die.
7. Rabbits also breed (asexually!) when they have enough energy. The parent's energy is reduced, and then it produces an exact clone of itself at its current location.

Once set into action, these constaints give rise to the dynamic system you see in the animation below. To make sense of what's going on, ignore the rabbits and focus on the oscillating growing and shrinking of the cabbage patch:

![](http://i.imgur.com/3iese1f.gif)

What you're seeing happen here is a basic cycle that tends to proceed in the following fashion:

* Whenever the rabbits have plenty of cabbage to eat, they breed, and their population numbers rise.

* As the rabbit population rises, the cabbage gets eaten more rapidly, reducing the amount of total food available to the rabbits in the cabbage patch.

* As food sources dwindle, rabbits tend to stop breeding and some also die of starvation, causing their population levels to drop.

* A smaller rabbit population leads to slower cabbage consumption, which results in rapid regrowth and plenty of cabbage for the rabbits to eat.

* This in turn leads the rabbits to stop dying from starvation and start breeding again, starting the cycle all over again.

The fact that we've reproduced this cycle is not a particularly profound result: you could have guessed it without ever bothering to create a simulation. However, if you treat the basic problem as a starting point and then continue your explorations from there, many more surprising results can be found. 

In my casual experiments I found that the system is surprisingly tolerant to singular catastrophic events (such as killing off 90% of the rabbits or the cabbage), because the two populations naturally force each other into balance. However, very small changes to the rate of cabbage regrowth, or to the amount of energy the rabbits gain from eating the cabbage can have disasterous effects that lead to extinction. I found these patterns interesting, because they were opposite to my intuition. 

Perhaps a more significant point though is that I doubt I would have even thought to try out those ideas if I were working with a formal equation rather than a dynamic and lively visualization. Because I'm not a visually-oriented learner, this really surprised me!

The full source code for this scenario is shown below, and you're should skim it at least, but you don't need to get bogged down in the details unless you plan to play around with StarLogo or my Terrarium engine after you're done reading this article. If you're feeling a bit tired by now, you can skip right past it to the next section without losing too much.  

```ruby
Terrarium::Scenario.define do
  cabbage_density    = 0.5 
  regrowth_rate      = 0.02
  initial_population = 200
  initial_energy     = 8
  food_energy        = 5
  hatch_threshold    = 10
  hatched_energy     = 0.25

  cabbage_color = :green
  rabbit_color  = :white
  soil_color    = :black

  # rule 1: create cabbage patch
  patches do 
    set_color soil_color
    with_probability(cabbage_density) { set_color(cabbage_color) } 
  end

  # rule 2: cabbage regrowth
  patches! do
    with_probability(regrowth_rate) { set_color(cabbage_color) } 
  end

  create_creatures(200)

  # rule 3: arrange a crowd of rabbits
  creatures do 
    lt rand(0..359)
    fd rand(5..25)

    data[:energy] = initial_energy

    set_color rabbit_color
  end

  # rule 4: let the rabbits wander
  creatures! { rt(rand(1...40)); lt(rand(1..40)); fd(1) }
  
  # rule 5: rabbits eat any cabbage they encounter, gaining energy
  creatures! do 
    update_patch do |patch| 
      if patch.color == cabbage_color
        patch.set_color soil_color
        data[:energy] += food_energy
      end
    end
  end
  
  # rule 6: the rabbits are always losing energy
  creatures! { data[:energy] -= 1 }

  # rule 7: when the rabbits run out of energy, they die
  creatures! { destroy if data[:energy] < 1 }

  # rule 8: when rabbits have enough energy, they clone themselves
  #         (but it costs them some energy)
  creatures! do 
    if data[:energy] > hatch_threshold
      data[:energy] *= hatched_energy

      hatch
    end
  end
end
```

## Exploratory programming as a first-class paradigm?

Even though we've managed to pack a lot of interesting behavior into a small amount of code, the examples I've shown here barely scratch the surface of StarLogo and capabilities. While my Terrarium engine is nothing more than a poor man's implementation of a few of StarLogo's features, the full StarLogo language is elegantly designed and carefully thought out.

But the goal of this article was not to introduce you to a shiny piece of technological infrastucture, it was meant to get you thinking about a different kind of workflow than what we tend to use day to day.  Even through the smudged window I've had you look through, it should be clear to see that the style of programming used in StarLogo has several powerful benefits:

1. Thoughts can be expressed directly
2. Feedback is given continuously 
3. Failure comes at a very low cost
4. The problem domain is well constrained
5. Objects can be directly acted upon

While most of the tools I use when I'm programming have at least some of these positive traits, it's rare to experience the effect of all of them simultaneously. However, a few positive examples do come 
to mind. In particular, the various web browser development tools (like Firebug or the tools that ship with Chrome) support this kind of workflow.

When it comes to frontend web development tools, I've always been amazed at how much it is possible to incrementally evolve a design by tweaking various page elements until you're happy with them. I think that much of the effectiveness of this technique is due to the benefits listed above. Here is a specific example to illustrate that point:

1. If you want to change a font size of a given block of text, it's as easy as clicking that text and editing a single attribute.

2. You see the results immediately on your screen. 

3. If you don't like the results, you can easily revert your changes. And if you made a mistake when you were editing things, it should be immediately obvious based on what does (or doesn't) get displayed on the screen.

4. Although the environment is very sophisticated, the scope is constrained enough where the available actions are fairly clear at any given point in time. 

5. Finally, because you are often looking at things within the scope of a single element that you are working with directly, you can use extremely localized thinking without harmful consequences.

Unfortunately, I can't easily come up with similar examples when it comes to backend web frameworks. If you narrow the scope, similar workflows can be applied to very simple HTTP services running on Sinatra, but once you need anything more complex than that it becomes much too broad of a problem to solve.

To be fair, Rails has some elements baked into it that facilitate a certain amount of exploratory programming (the console, scaffolding, etc.). However, these features have always felt to me as if they were not taken nearly far enough, and that there is still room for a much higher level toolkit, even if it would only be useful for rapid prototyping. 

In an ideal world, I would love to be able to describe a useful full-stack feature in a web application in a dozen lines or less, but I've never seen anything that gets me even close to that level of abstraction. Of course, web architecture is sufficiently obtuse to make this a genuinely hard problem to solve, so I'm not surprised that there isn't an obvious solution out there just yet.

But web programming (particularly general-purpose web programming) is really at a lower level than where this paradigm really could shine. It seems to me that there is nearly infinite possibility for what one might call "domain-specific development environments". For example, could we build programmable tools for book publishers that sit somewhere between a WYSIWYG editor and DocBook XML? Could we build drop-in management panels for business metrics that can be programmed at a high enough level that an analyst could use them with minimal help from their programming team? Is there hope that we can put these kinds of high-powered but easy-to-use tools into the hands of musicians, artists, teachers, and charity volunteers?

Perhaps the best use of a general purpose programming language it to build domain-specific environments that help cross a bridge from low-level infrastructure to high-level ideas. But because this is all just a pie-in-the-sky dream that may never end up becoming a reality, I will let you be the judge! Please share your thoughts in the comments below.






