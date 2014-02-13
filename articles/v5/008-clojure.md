An interesting thing about learning new programming languages is that it takes
much less time to learn how to read programs than it does to write them. While
building non-trivial software in a language you are not familiar with can take weeks
or months of dedicated practice, the same software could be read and understood
in a fraction of that time. 

Because programming languages are much more similar to the formal language of 
mathematics than they are to natural languages, people from diverse backgrounds 
can communicate complex ideas with a surprising lack of friction. Unfortunately,
we often forget this point because we are overwhelmed by the memories of how
hard it is to *write* elegant code in a new language. This tendency puts us at a
tremendous disadvantage, because it causes us to artifically limit our access to
valuable learning resources.

In this article, I will walk you through an example of how I was plagued by this
very fear, how I overcame it, and how that lead me to learn a lot 
about [Clojure][clojure] in a very short period of time. My hope is that by following 
in my footsteps, you'll be able to learn the technique I used and possibly apply
it to your own studies.

## How I finally learned about Ant Colony Optimization

For a few weeks before this article was published, I was busy
researching  [swarm intelligence][swarm]. I have always been fascinated by 
how nature-inspired algorithms can be used to solve surprisingly complex 
computing problems, and I decided that I wanted to try implementing some 
of them myself. I started off by implementing the [Boids algorithm][boids], 
and was surprised at how quickly I was able to get something vaguely 
resembling a flock of birds to appear on my screen. Motivated by that small
win, I decided to try my hand at simulating an [Ant Colony][aco].

On the surface, ant behavior is deceptively simple, even intuitive. At least,
that is what the description provided by Wikipedia would have you believe:

1. An ant (called "blitz") runs more or less at random around the colony;
2. If it discovers a food source, it returns more or less directly to the nest, leaving in its path a trail of pheromone;
3. These pheromones are attractive; nearby ants will be inclined to follow, more or less directly, the track;
4. Returning to the colony, these ants will strengthen the route;
5. If there are two routes to reach the same food source then, in a given amount of time, the shorter one will be traveled by more ants than the long route;
6. The short route will be increasingly enhanced, and therefore become more attractive;
7. The long route will eventually disappear because pheromones are volatile;
8. Eventually, all the ants have determined and therefore "chosen" the shortest route.

Unfortunately, it is hard to find resources that precisely describe the rules
that govern each of these behaviors, and those that do exist are highly abstract
and mathematical. While I'm not one to shy away from theoretical papers, I
usually like to approach them once I understand a concept fairly well in
practice. For this particular problem, I was unable to find the materials 
that would get to that point, and it felt like I was hitting a brick wall.

Although I found tons of examples of applying a generalized form of ant colony 
optimization to the traveling salesman problem, I wanted to start with a more 
direct simulation of the natural behavior. After digging around for a bit, I
found [Rich Hickey's ant simulator][sim], which is implemented 
in [Clojure][clojure]. Check out the video below to see what it looks like in action:

<div align="center">
<iframe width="720" height="480"
src="//www.youtube.com/embed/shm7QcJMvig?rel=0" frameborder="0" allowfullscreen></iframe>
</div>

I knew right away that this was exactly the kind of simulation I wanted to
build, but it honestly didn't even cross my mind to attempt to read the Clojure
code and port it to Ruby. One quick glance at its [source code][sim] reminded me
just how much I wanted to learn a Lisp dialect some day, but it definitely
wasn't going to be today! I didn't have time to go dust off the books on my
shelf that I never read, or to watch the [2.5 hour long video][hickey] of the
talk that this code came from. 

So instead of doing all that, I set off to build my own implementation from
scratch by cobbling together the bits of information I had collected
into something that sort of worked. Using the general description of ant
behavior as my guide, I left it up to my imagination to fill in the details, and
within an hour or so I had built something that had ants moving around the
screen. Unfortunately, my little family of ants seemed to have come from a
failed evolutionary branch, because they didn't do what they were supposed to
do! They'd wander around randomly, get stuck, choose the wrong food sources, and
generally misbehave in all sorts of painful ways. Thinking that I needed a
break, I stepped away from the project for a day so that I could come back to it
with a fresh perspective.

The next day, I did end up getting something vaguely resembling an ant colony to
appear on my screen. The behavior was not perfect, but it illustrated the main 
idea of the algorithm:

<div align="center">
<iframe width="720" height="480"
src="//www.youtube.com/embed/p_XmuRHs57g?rel=0" frameborder="0" allowfullscreen></iframe>
</div>

It was fairly easy to get to this point, but then it became extremely hard to 
improve upon the simulator. Ant colony optimization has a lot of variables to it
(i.e. things like the size of the world, number of ants, number of food sources,
pheremone decay rate, amount of pheremone dropped per iteration, etc). Changing
any one of these things can influence the effectiveness of the others. When you
combine this variability with an implementation where the actual behaviors were 
half-baked and possibly buggy, you end up with a big mess that is hard to debug,
and even harder to understand. Knowing that my code was in really bad shape,
I was ready to give up.

Although it took a lot of rumination to get me there, the lightbulb eventually
turned on: maybe reading the Clojure implementation wasn't such a bad idea after
all! I had initially thought that learning the algorithm would be much easier
than learning the semantics and syntax of a new language, but two days of hard
work and mediocre results lead me to re-evaluate that assumption. At the very
least, I could spend an afternoon with the Clojure code. Even if the ants still frightened
and confused me in the end, I'd at least learn how to read some code from a language 
that I had always wanted to study anyway. 

I think you can guess what happened next: within a couple of hours, I not only
fully understood Rich Hickey's implementation, but I had also learned dozens
upon dozens of Clojure features, including a few that have no direct analogue in
Ruby. While it may have been a result of frustration 
driven development, I was genuinely surprised at what a great way this was to
learn a new language while also studying a programming problem that I was
interested in.

Throughout the rest of this article, I will attempt to demonstrate that given
the right example, even a few dozen lines of code can teach you a tremendous
amount of useful things about a language that you've never worked with before.
If you are new to Clojure programming, you'll be able to follow along
and experience the same benefits that I did; if you already know the
language, you can use this as an exercise in developing a beginner's mindset. In
either case, I think you'll be surprised at how much we can extract
from such a small chunk of code.

To keep things simple, we won't bother to read the complex bits of code that 
implements ant behavior. Instead, we'll start from the bottom up and
take a look at how this simulation models its world and the ants within it.
Although this won't help us understand how things get set into action, it will
give us plenty of opportunities to learn some interesting Clojure
features.

## Modeling the world

The following code is responsible for creating a blank slate world with 80x80
dimensions:

```clojure
(defstruct cell :food :pher) ;may also have :ant and :home

;dimensions of square world
(def dim 80)

;world is a 2d vector of refs to cells
(def world 
   (apply vector 
     (map (fn [_] 
            (apply vector (map (fn [_] (ref (struct cell 0 0))) 
                               (range dim)))) 
          (range dim))))

(defn place [[x y]]
  (-> world (nth x) (nth y)))
```

Even if this is the first time you've ever seen a Clojure program, you could take an
educated guess at what is going on in at least a few of these lines of code:

```clojure
(defstruct cell :food :pher)      ; this defines a Struct-like thing

(def dim 80)                      ; this defines a named value, setting dim=80

(defn place [[x y]]
  (-> world (nth x) (nth y)))     ; this looks like an accessor into a
                                  ; two-dimensional grid
```

The code in the `world` definition is much more complicated, but it 
has a helpful comment that describes what it is: a 2D vector of 
refs to cells. This hint gives us some useful keywords to search for 
in [Clojure's API docs][clojure-doc]. With a bit of effort, it is possible
to use this code sample and Clojure's documentation to learn all the 
following things about the language:

1. The [Map][Map] collection is Clojure's equivalent to Ruby's `Hash` object.
1. The [StructMap][StructMap] collection is a `Map` with some predefined keys
that cannot be removed. They can be defined using `defstruct`, and are instantiated
via `struct`.
1. The `(def ...)` construct is a [special form][def] that defines global variables 
within a namespace, but it is considered bad style to treat these variables as
if they were mutable.
1. The `(defn ...)` construct is a macro which among other things provides
syntactic sugar for defining functions with named parameters.
1. The [Vector][Vector] collection has core functionality which is similar to
Ruby's `Array` object. Vectors can be instantiated using the `vector` function or
via the `[]` literal syntax,  and their elements are accessed using 
the `nth` function.
1. All collections in Clojure implement a [Sequence][Sequence]
interface that is similar to Ruby's `Enumerable` module. It provides various
functions that Ruby programmers are already familiar with, such as `map`, `reduce`, 
`sort` But because most of these functions return lazy sequences, they
behave slightly differently than their Ruby counterparts.
1. The [Ref][Ref] construct is a transactional reference, which is one of
Clojure's concurrency primitives. In a nutshell, wrapping state in a `Ref`
makes it so that state can only be modified from within a transaction, ensuring
thread safety.
1. Among other things, the `range` function provides behavior similar to the enumerator 
form of Ruby's `Integer#times` method. 
1. The `apply` function provides functionality similar to Ruby's splat operator
(`*`), passing the elements of a sequence as arguments to a function.
1. The [-> macro][->] provides syntactic sugar for function composition, which
can make chaining function calls easier.

Based on this laundry list of concepts to learn, it is easy to see from this
example alone that much like Ruby, Clojure is a very rich language that is
capable of concisely expressing very complex ideas. With that in mind, it is
helpful to use Clojure's REPL to experiment while learning, much as we'd do
with `irb` in Ruby. Once again using the code sample as a guide, an 
exploration such as the one that follows can go a long way towards 
verifying our understanding of what we learned from the documentation:

```clojure
user=> (defstruct cell :food :pher)
; #'user/cell
user=> (struct cell 1 4)
; {:food 1, :pher 4}
user=> [ [ :a :b :c ] [ :d :e :f ] ]
; [[:a :b :c] [:d :e :f]]
user=> (def data [[:a :b :c] [:d :e :f]])
; #'user/data
user=> (nth (nth data 1) 2)
; :f
user=> (nth (nth data 2) 1)
; IndexOutOfBoundsException   clojure.lang.PersistentVector.arrayFor 
; (PersistentVector.java:106)
user=> (nth (nth data 0) 1)
; :b
user=> (-> data (nth 1) (nth 2))
; :f
user=> (map (fn [x] (* x 2)) [1 2 3])
; (2 4 6)
user=> (vector (map (fn [x] (* x 2)) [1 2 3]))
; [(2 4 6)]
user=> (apply vector (map (fn [x] (* x 2)) [1 2 3]))
; [2 4 6]
user=> (range 5)
; (0 1 2 3 4)
user=> (apply vector (map (fn [x] (struct cell 0 0)) (range 5)))
; [{:food 0, :pher 0} {:food 0, :pher 0} {:food 0, :pher 0} 
; {:food 0, :pher 0} {:food 0, :pher 0}]
```

Knowing what we now know, it is possible to imagine a loose translation of the
original Clojure code sample into Ruby, if we account for a few cavaets:

1. Most `Enumerable` methods return `Array` objects, which are not lazily
evaluated. Some support for lazy sequences exist in Ruby 2.0, but we'll
not bother with that in our translation because it'd only create more
work for us.

2. We don't have a direct analogy to Clojure's `Ref` construct, but we can
pretend that we do for the purposes of this example.

3. We don't have anything baked into the language which implements a `Hash` with
some required keys and some optional ones. But such behavior could be
emulated by building a custom `Cell` object. 

4. We don't have destructuring in the parameter lists for our
functions, so we need to handle destructuring manually within the bodies
of our methods rather than their signatures.

Keeping these points in mind, here's a semi-literal translation of Clojure code
to Ruby:

```ruby
DIM    = 80                                          
WORLD  = DIM.times.map do                                 # 1
           DIM.times.map { Ref.new(Cell.new(0, 0)) }      # 2,3
         end

def place(pos)       
  x, y = pos                                              # 4
  WORLD[x][y]
end
```

While the two languages cannot be categorically compared by such a coarse
exercise in syntactic gymnastics, it does help the similarities and 
differences between the languages stand out a bit more. This allows us to reuse
the knowledge we already have, and also exposes the gaps in our 
understanding that need to be filled in.

> **SIDE QUEST:** The remaining two sections in this article will repeat this
same basic process on two more small chunks of code from the ant simulator. If you have some
free time and an interest in learning Clojure, you may want to start
with the initial code samples in each section and try to figure them out on
your own, and *then* come back to read my notes. If you decide to
try this out, please share a comment with what you've learned.

Now that we've tackled one concrete feature from this program, it will be much
easier to understand the rest. There's a lot left to learn, so let's keep
moving!

## Modeling an ant

The following code is responsible for initializing an ant at a 
given location within the world:

```clojure
(defstruct ant :dir) ;may also have :food

(defn create-ant 
  "create an ant at the location, returning an ant agent on the location"
  [loc dir]
    (dosync
      (let [p (place loc)
            a (struct ant dir)]
        (alter p assoc :ant a)
        (agent loc))))
```

Because we already have a rudimentary understanding of how `StructMap` works,
and how to define functions, we can skip over some of the boilerplate
and get right to the good stuff:

```clojure
(dosync                               ; 1
  (let [p (place loc)                 ; 2
        a (struct ant dir)]
    (alter p assoc :ant a)            ; 3
    (agent loc)))                     ; 4
```

Digging back into Clojure's API docs, we can learn four new things 
from this code sample:

1. The [dosync][dosync] macro starts a transaction,
which among other things, makes it possible to modify `Ref`
structures in a thread-safe way.

1. The [let][let] macro allows you to make use of named values within
a lexical scope. This construct appears to be roughly similar to the 
concept of block-local variables in Ruby.

1. The [alter][alter] function is used for modifying the contents of a `Ref`
structure, and can only be called within a transaction.

1. The [Agent][Agent] construct is another one of Clojure's concurrency
primitives. This structure provides an interesting state-centric alternative
to the actor model of concurrency: rather than encapsulating behavior that acts
upon external state, agents encapsulate state which is *acted upon* by external
behaviors.

Of course, in order to verify that we understand what the documentation is
telling us, nothing beats a bit of casual experimentation in the REPL:

```clojure
user=> (let [x 10 y 20] (+ x y))
; 30
user=> (let [x 10] (let [y 20] (+ x y)))
; 30
user=> (let [x 10] (let [y 20]) y)
; CompilerException java.lang.RuntimeException: Unable to resolve 
; symbol: y in this context, compiling:(NO_SOURCE_PATH:3) 
user=> (def foo (ref { :x 1 :y 1}) )
; #'user/foo
user=> foo
; #<Ref@6762ba99: {:y 1, :x 1}>
user=> (assoc foo :z 2)
; ClassCastException clojure.lang.Ref cannot be cast to clojure.lang.Associative  
; clojure.lang.RT.assoc (RT.java:691)
user=> (assoc @foo :z 2)
; {:z 2, :y 1, :x 1}
user=> @foo
; {:y 1, :x 1}
user=> (alter foo assoc :z 2)
; IllegalStateException No transaction running  
; clojure.lang.LockingTransaction.getEx (LockingTransaction.java:208)
user=> (dosync (alter foo assoc :z 2))
; {:z 2, :y 1, :x 1}
user=> (def bar (agent [1 2 3]))
; #'user/bar
user=> bar
; #<Agent@3445378f: [1 2 3]>
user=> @bar
; [1 2 3]
user=> (send bar reverse)
; #<Agent@3445378f: [1 2 3]>
user=> bar
; #<Agent@3445378f: (3 2 1)>
user=> @bar
; (3 2 1)
user=> (reverse @bar)
; (1 2 3)
user=> @bar
; (3 2 1)
```

The ant creation code sample consists mostly of features that don't exist in
Ruby, so a direct translation isn't possible. However, it doesn't hurt to
imagine what the syntax for these features might look like in Ruby if we did
have Clojure's concurrency primitives:

```ruby
  def create_ant(loc, dir)
    Ref.transaction do 
      p = place(loc)
      a = Ant.new(dir)
    
      p.ant = a

      Agent.new(loc)
    end
  end
```

Assuming that Clojure's semantics were maintained, either all mutations that
happen within the `Ref.transaction` block would be applied, or none of them
would be. Furthermore, thread-safety would be handled for us ensuring state
consistency for the duration of the block. Language-level transactions seem like
seriously powerful stuff, and it will be interesting to see if Ruby ends up
adopting them in the future.

## Populating the world

The following code populates the initial state of the world with ants and food:

```clojure
;number of ants = nants-sqrt^2
(def nants-sqrt 7)
;number of places with food
(def food-places 35)
;range of amount of food at a place
(def food-range 100)

(def home-off (/ dim 4))
(def home-range (range home-off (+ nants-sqrt home-off)))

(defn setup 
  "places initial food and ants, returns seq of ant agents"
  []
  (dosync
    (dotimes [i food-places]
      (let [p (place [(rand-int dim) (rand-int dim)])]
        (alter p assoc :food (rand-int food-range))))
    (doall
     (for [x home-range y home-range]
       (do
         (alter (place [x y]) 
                assoc :home true)
         (create-ant [x y] (rand-int 8)))))))
```

As in the ant initialization code, this snippet includes a mixture of new
concepts and old ones. If we focus on the body of the `setup` definition, there
are five new things for us to learn:

```clojure
(dosync
    (dotimes [i food-places]                             ;1
      (let [p (place [(rand-int dim) (rand-int dim)])]   ;2
        (alter p assoc :food (rand-int food-range))))
    (doall                                               ;3
     (for [x home-range y home-range]                    ;4
       (do                                               ;5
         (alter (place [x y]) 
                assoc :home true)
         (create-ant [x y] (rand-int 8))))))
```

1. The [dotimes][dotimes] macro is a simple iterator that is comparable to the
block form of `Integer#times` in Ruby. 

1. The [rand-int][rand-int] function returns a random integer between 0 and
a given number, which is similar to calling Ruby's `Kernel#rand` with an 
integer argument.

1. The [doall][doall] macro is used to force a lazy sequence to be fully
evaluated.

1. The [for][for] macro implements list comprehensions, which are a very
powerful form of iterator that does not have a direct analogue in Ruby.

1. The [do][do] special form executes a series of expressions in sequence and
returns the result of the last expression. This is roughly equivalent to Ruby's
`do...end` block syntax.

One last trip back to the REPL is needed to confirm that once again, the
documentation is not lying, and we have not misunderstood its explanations:

```clojure
user=> (dotimes [i 5] (println i))
; 0
; 1
; 2
; 3
; 4
; nil
user=> (rand-int 10)
; 3
user=> (rand-int 10)
; 6
user=> (rand-int 10)
; 6
user=> (rand-int 10)
; 2
user=> (for [x (range 5) y (range 5)] [x y])
; ([0 0] [0 1] [0 2] [0 3] [0 4] [1 0] [1 1] [1 2] [1 3] [1 4] 
; [2 0] [2 1] [2 2] [2 3] [2 4] [3 0] [3 1] [3 2] [3 3] [3 4] 
; [4 0] [4 1] [4 2] [4 3] [4 4])
user=> (for [x (range 5) y (range 5)] (+ x y))
; (0 1 2 3 4 1 2 3 4 5 2 3 4 5 6 3 4 5 6 7 4 5 6 7 8)
user=> (do (print "hello world\n") (+ 1 1))
; hello world
; 2
user=> (realized? (for [x (range 5) y (range 5)] [x y])) 
; false
user=> (realized? (doall (for [x (range 5) y (range 5)] [x y])))
; true
```

Because many of the Clojure features used for populating the simulation's 
world either already exist in Ruby or are irrelevant due to implementation
differences, this code sample translates fairly well. Apart from the fact
that the `Ref` construct in this example is imaginary, the only 
noticeable thing that is lost in translation is the conciseness
of Clojure's list comprehensions. But in this particular use case,
`Array#product` gets us part of the way there:

```ruby
NANTS_SQRT  = 7
FOOD_PLACES = 35
FOOD_RANGE  = 100

HOME_OFF   = DIM / 4
HOME_RANGE = (HOME_OFF..NANTS_SQRT + HOME_OFF)

def setup
  Ref.transaction do
    FOOD_PLACES.times do
      p      = place([rand(DIM), rand(DIM])
      p.food = rand(FOOD_RANGE)
    end

    HOME_RANGE.to_a.product(HOME_RANGE.to_a).map do |x,y|
      place([x,y]).home = true
    
      create_ant([c, y], rand(8))
    end
  end
end
```

At this point, you should now completely understand the structure of the initial
state of the world in [Rich Hickey's ant simulator][sim], and if you're new to
Clojure, you probably know a lot more about the language than you did when you
started reading. If you have enjoyed the journey so far, definitely consider
reading the entire program; this article only covers tip of the iceberg! 

## Reflections

[XKCD] sums up how I feel about this exercise much better than I could on my own:

[![](http://imgs.xkcd.com/comics/lisp_cycles.png)](http://xkcd.com/297/)

That said, I'm sure that more than a few people would be happy to tell you that 
many of the pragmatic compromises that Clojure has made are blasphemic in some
way. Truth be told, I don't know nearly enough about functional languages to
weigh in on any of those claims.

The real takeaway for me was that by stepping outside of my comfort zone for
even a few hours, I was able to look back at Ruby with a fresh perspective. I
was also able to gain an understanding of a programming problem that I couldn't
find a good Ruby example for. Both of these things were a huge win for me. I
hope that you will find a way to try this exercise out on one of your own 
problems, and I look forward to hearing what you think of it.

Learning to read code in a language you are not familiar with takes practice,
but it is easier than it seems. If you step outside the
bubble from time to time, only good things will come of it.

> **NOTE**: You may want to try out [4Clojure][4Clojure] if you want to hone
> your Clojure skills at a more gradual pace than what we attempted in this
> article. It's a quiz site similar to [RubyKoans].
 
[swarm]:       http://en.wikipedia.org/wiki/Swarm_intelligence
[boids]:       http://en.wikipedia.org/wiki/Boids
[aco]:         http://en.wikipedia.org/wiki/Ant_colony_optimization
[sim]:         https://gist.github.com/1093917
[clojure]:     http://clojure.org/
[clojure-doc]: http://clojure.org/documentation
[hickey]:      http://blip.tv/clojure/clojure-concurrency-819147
[xkcd]:        http://xkcd.com
[4Clojure]:    http://www.4clojure.com
[RubyKoans]:   http://rubykoans.com

[def]:     http://clojure.org/special_forms#Special%20Forms--%28def%20symbol%20init?%29
[Map]: http://clojure.org/data_structures#Data%20Structures-Maps%20%28IPersistentMap%29
[StructMap]: http://clojure.org/data_structures#Data%20Structures-StructMaps
[Vector]: http://clojure.org/data_structures#Data%20Structures-Vectors%20%28IPersistentVector%29
[Sequence]: http://clojure.org/sequences
[Ref]: http://clojure.org/refs
[->]: http://blog.fogus.me/2009/09/04/understanding-the-clojure-macro/
[dosync]: http://clojure.github.com/clojure/clojure.core-api.html#clojure.core/dosync
[let]: http://clojure.github.com/clojure/clojure.core-api.html#clojure.core/let
[alter]: http://clojure.github.com/clojure/clojure.core-api.html#clojure.core/alter
[agent]: http://clojure.org/agents
[dotimes]: http://clojure.github.com/clojure/clojure.core-api.html#clojure.core/dotimes
[for]: http://clojure.github.com/clojure/clojure.core-api.html#clojure.core/for
[doall]: http://clojure.github.com/clojure/clojure.core-api.html#clojure.core/doall
[rand-int]: http://clojure.github.com/clojure/clojure.core-api.html#clojure.core/rand-int
[do]: http://clojure.org/special_forms#Special%20Forms--%28do%20exprs*%29
