The [Law of Demeter](http://www.ccs.neu.edu/home/lieber/LoD.html) is a well-known 
software design principle for reducing coupling between collaborating objects. 
However, because the law exists in many forms, it often means different things 
to different people. As far as laws go, Demeter has been flexible in practice, 
which has lead to some interesting evolutions in its application over time. 
In this article, I will discuss an interpretation of the law that is quite 
literally out of this world.

### An introduction to Smyth's Law of Demeter

[David Smyth](http://mars.jpl.nasa.gov/zipcodemars/bio-contribution.cfm?bid=1018&cid=393&pid=377&country_id=0), 
a scientist who worked on various Mars missions for NASA's Jet 
Propulsion Laboratory, came up with this seemingly innocuous definition
of the Law of Demeter:

> A method can act only upon the message arguments and the state of the receiving object.

On the surface, this formulation is essentially [the object form of the Law
of Demeter](http://www.ccs.neu.edu/research/demeter/demeter-method/LawOfDemeter/object-formulation.html)
stated in much less formal terms. However, Smyth's law is different in the way in which
he interprets it: he assumes that the Law of Demeter implies that 
methods should not have return values. This small twist causes 
the law to have a much deeper effect than its originators had 
anticipated. 

Before we discuss the implications of building systems entirely out of methods
without return values, it is important to understand why Smyth assumed
that value-returning methods were forbidden in the first place. To explore
that point, consider the following trivial example:

```ruby
class Person < ActiveRecord::Base
  def self.in_postal_area(zipcode)
    where(:zipcode => zipcode)  
  end
end
```

The `Person.in_postal_area` method does not violate the 
Law of Demeter itself, as it is nothing more than a simple delegation
mechanism that passes the `zipcode` parameter to a 
lower-level function on the same object. But because it
returns a value, this function makes it easy for its callers
to violate the Law of Demeter, as shown here:

```ruby
class UnsolicitedMailer < ActionMailer::Base
  def spam_postal_area(zipcode)
    people = Person.in_postal_area(zipcode)

    emails = people.map { |e| e.email }

    mail(:to => emails, :subject => "Offer for you!")
  end
end
```

Because the value returned by `Person.in_postal_area` is neither 
a direct part of the `UnsolicitedMailer` object nor a parameter
of the `spam_postal_area` method, sending messages
to it results in a Demeter violation. Depending on the project's 
requirements, breaking the law in this fashion could be 
reasonable, but it is a code smell to watch out for.

In the context of the typical Ruby project, methods that 
return values are common because the convenience of implementing
things this way often outweighs the cost of doing so. However,
whenever you take this approach, you make two fundamental 
assumptions that those who write code for Mars rovers 
cannot: that your value-returning methods will respond
in a reasonable amount of time, and that they will not fail 
in all sorts of complicated ways.

Although these basic assumptions often apply to the bulk of what we do,
even those of us who aren't rocket scientists occasionally
need to work on projects for which temporal coupling is considered
harmful and robust failure handling is essential. In such
scenarios, it is worth considering what Smyth's interpretation
of the Law of Demeter (LoD) has to offer.

### The implications of Smyth's Law of Demeter

Smyth's unique interpretation of how to apply LoD eventually 
caught the eye of Karl Lieberherr, a member of the
Demeter project who published some of the earliest papers
on the topic. Lieberherr took an interest in Smyth's approach 
because it was clearly different than what the Demeter 
researchers had intended—yet potentially useful. 
A correspondence between the two led Smyth to share his 
thoughts about what his definition of LoD brings to 
the table. His six key points from the [original discussion](http://www.ccs.neu.edu/research/demeter/demeter-method/LawOfDemeter/Smyth/LoD-revisited2) 
are listed in an abridged form here:

```
There are actually several wonderful properties that fall out 
from this definition of LoD:

     A method can act only upon the message arguments and the
     existing state of the receiving object.

1. Method bodies tend to be very close to straight-line code. Very
   simple logic, very low complexity.

2. There must be no return values; if there are, the sender of the message
   is not obeying the law.

3. There cannot be tight synchronization, as the sender cannot tell whether
   the message is acted on or not within any "small" period of time
   (perhaps the objects collaborate with a two-way protocol and the
   sender can eventually detect a timeout).

4. Because there are no return values, the objects need to be
   "responsible" objects: they need to handle both nominal and
   foreseeable off-nominal cases. This requirement has the wonderful effect of
   localizing failure handling within the object that has the
   best visibility and understanding of whatever went wrong.
   It also dramatically reduces the complexity of protocols and
   clients.

   ...

5. The law requires an object to subscribe to information so that it has
   what it needs whenever it gets a message. Thus lazy
   evaluation can't be used. Although this requirement may seem like an 
   inefficiency, it becomes one in practice only if the objects don't have 
   concise responsibilities. In such a case, efficiency of communication
   bandwidth isn't the real problem.

   ...

6. Because tight synchronization is out of the picture, the responsible
   objects should be goal oriented. A goal is different from a method
   in that a goal is pursued over some expanse of time and does not
   seem instantaneous. By thinking of goals rather than discrete
   actions, people can derive solutions that don't require tight
   temporal coupling. This sounds like hand waving, and it is—but
   seven years of doing it shows that it really does work.
```

These are deep claims, but the remainder of the discussion between Smyth
and Lieberherr did not elaborate much further on them. However, it is 
fascinating to imagine the kind of programming style that Smyth
is advocating here: it boils down to a highly robust form of
[responsibility-driven development](http://practicingruby.com/articles/64) with 
concurrent (and potentially distributed) objects that communicate almost 
exclusively via callback mechanisms. If Smyth were not an established
scientist working on some of the world's most challenging problems,
it would almost seem as if he were playing object-oriented buzzword bingo.

Although I don't know nearly enough about any of these ideas to speak 
authoritatively on them, I think that they form a great starting point 
for a very interesting conversation. However, if you're like me, you
would benefit from having these ideas brought back down to earth
a bit. With that in mind, I've put together a little example 
program that will hopefully help you do exactly that.

### Smyth's Law of Demeter in practice

Software design principles can be interesting to study in the abstract, but
there is no substitute for trying them out in concrete applications. If you 
can find a project that is a natural fit for the technique you are 
trying to investigate, even the most simple toy application will teach you
more than pure thought experiments ever could.

Smyth's approach to the Law of Demeter originated from his work on software for
Mars rovers, an environment where tight temporal coupling and a lack of 
robust interactions between distributed systems can cause serious problems.
Because it takes about 14 minutes for light to travel between Earth and Mars, 
even the most trivial system interactions require careful design consideration. 
With so much room for things to go wrong, a programming style that claims to 
make it easier to manage these kinds of problems definitely sounds promising.

Of course, you don't need to land robots on Mars to encounter these kind of
challenges. I can easily imagine things such as payment processing systems 
and remote system administration toolchains having a good
degree of overlap with the issues that Smyth's LoD is meant to
address. Still, those problems are not nearly as exciting as driving a
remote control car around on a different planet. Knowing that, I decided
to test Smyth's ideas by building a very unrealistic Mars rover 
simulation. The video below shows me interacting with it over IRC:

<div align="center">
<iframe width="800" height="600"
src="//www.youtube.com/embed/Yqofx6MbYFU?vq=480&rel=0" frameborder="0" allowfullscreen></iframe>
</div>

In the video, the communications delay is set at only a couple of seconds, but it
can be set arbitrarily high, which makes it possible to simulate the full 14-
minute-plus delay between Earth and Mars. No matter what the delay is set at, the
rover queues up commands as they come in and sends its responses one 
at a time as its tasks are completed. The entire simulator is only a couple of
pages of code. It consists of the following objects and responsibilities:

* [SpaceExplorer::Radio](https://github.com/elm-city-craftworks/space_explorer/blob/pr-5.2/lib/space_explorer/radio.rb) relays messages on a time delay.
* [SpaceExplorer::MissionControl](https://github.com/elm-city-craftworks/space_explorer/blob/pr-5.2/lib/space_explorer/mission_control.rb) communicates with the rover.
* [SpaceExplorer::Rover](https://github.com/elm-city-craftworks/space_explorer/blob/pr-5.2/lib/space_explorer/rover.rb) communicates with mission control and updates the map.
* [SpaceExplorer::World](https://github.com/elm-city-craftworks/space_explorer/blob/pr-5.2/lib/space_explorer/world.rb) implements the simulated world map.

As I implemented this system, I took care to abide by Smyth's recommendation
that methods not return meaningful values. Although I wasn't so pedantic as
to explicitly return `nil` from each function, I treated them as void functions
internally, so none of the simulator's features depend on the return value 
of the methods I implemented. To see the effect this approach had on
overall system design, we can trace a command's execution from end to end while
paying attention to what is going on under the hood.

I'd like to walk you through how `SNAPSHOT` works, simply because it has the
largest number of moving parts to it. As you saw in the video, 
`SNAPSHOT` is used to get back a 5x5 ASCII "picture" of the area around the 
rover, which can be used to aid navigation. In the following example, 
`@` is the rover, `-` represents empty spaces, and `X` represents boulders:

```
20:35|  seacreature| !SNAPSHOT
20:35|  roboseacreature| X - - X -
20:35|  roboseacreature| X X - X X
20:35|  roboseacreature| - X @ X X
20:35|  roboseacreature| - - X X -
20:35|  roboseacreature| - - - - -
```

As you may have already guessed, the user interface for this project is
IRC-based, which is a convenient (if ugly) medium for experimenting with
asynchronous communications. A bot that is responsible for running
the simulation monitors the channel for commands, which can be any
message that starts with an exclamation point. When these messages are
detected, they are passed on a `MissionControl` object for processing. The
callback that monitors the channel and passes messages along to that 
object is shown here:

```ruby
bot.on(:message, /\A!(.*)/) do |m, command|
  mission_control.send_command(command)
end
```

The `MissionControl` object is nothing more than a bridge between the UI 
and a `Radio` object, so the `send_command` method passes the 
command along without modification:

```ruby
module SpaceExplorer
  class MissionControl
    def send_command(command)
      @radio_link.transmit(command)
    end
  end
end
```

The `Radio` instance that `@radio_link` points to holds a
reference to a `Rover` object, which is where the `SNAPSHOT` command will be
processed. Before it gets there, `Radio#transmit` enforces a
transmission delay through the use of a very coarse-grained timer mechanism:

```ruby
module SpaceExplorer
  class Radio
    def transmit(command)
      raise "Target not defined" unless defined?(@target)

      Thread.new do
        start_time = Time.now

        sleep 1 while Time.now - start_time < @delay

        @target.receive_command(command) 
      end
    end
  end
end
```

It's important to point out here that `Radio#transmit` is designed to work with
an arbitrary delay, so it isn't practical for it to block execution and
return a value. Instead, it spins off a background thread that will eventually
call the `receive_command` callback method on its `@target` object, which in this case is a
`Rover` instance.

The implementation of the `Rover` object is more interesting than the
objects we've looked at so far because it implements the
[Actor model](http://en.wikipedia.org/wiki/Actor_model). 
Whenever `Rover#receive_command` is called, commands are not
processed directly but are instead placed on a threadsafe queue that then gets
acted upon in a first-come, first-serve basis. This approach allows the `Rover` to do 
its tasks sequentially while continuing to accept requests as they come in. To
understand how that works, think about how `SNAPSHOT` gets handled by the 
following code:

```ruby
require "thread"

module SpaceExplorer
  class Rover
    def initialize(world, radio_link)
      @world      = world
      @radio_link = radio_link

      @queue = Queue.new

      Thread.new { loop { process_command(@queue.pop) } }
    end

    def receive_command(command)
      @queue.push(command)
    end

    def process_command(command)
      case command
      when "PING"
        @radio_link.transmit("PONG")
      when "NORTH", "SOUTH", "EAST", "WEST"      
        @world.move(command)
      when "SNAPSHOT"
        @world.snapshot { |text| @radio_link.transmit("\n#{text}") }
      else
        # do nothing
      end
    end
  end
end
```

When the `receive_command` callback is triggered by the `Radio` object,
the method pushes that command onto a queue, which should happen nearly
instantaneously in practice. At this point, the command has finished its
outbound trip and is ready to be processed.

After the `Rover` object handles any tasks that were already queued up,
`SNAPSHOT` is passed to the `process_command` method, where the 
following line gets executed:

```ruby
@world.snapshot { |text| @radio_link.transmit("\n#{text}") }
```

This code looks a little weird because it isn't immediately obvious why a block
is being used here. Instead, we might expect the following code under 
ordinary circumstances:

```ruby
@radio_link.transmit("\n#{@world.snapshot}") 
```

However, taking this approach would be a subtle violation of Smyth's LoD,
because it would require `World#snapshot` to have a meaningful return value,
introducing additional coupling. In this case, the coupling is
temporal rather than structural, which makes it harder to spot.

The main difference between the two examples is that the latter has a strong
connascence of timing and the former does not. In the value-returning example,
if `@world.snapshot` were not simply generating a trivial ASCII diagram but
actually controlling hardware on a Mars rover to take an image, we might expect
it to take some amount of time to respond. If it were a large enough amount of
time, it wouldn't be practical to block while waiting for a response, so the
call to `RadioLink#transmit` would need to be backgrounded. This would also be
true for any caller that made use of `World#snapshot`.

By using a code block (which is really just a lightweight, anonymous callback
mechanism), we can push the responsibility of whether to run the
computations in a background thread into the `World` object, making that
decision completely invisible to its callers. As an added bonus, `World` can
also be more responsible about failure handling as well, because it decides 
if and when to execute the callback and how to handle unexpected situations.

In practical scenarios, the advantages and disadvantages of whether 
violate Smyth's law would need to be weighed out, but in this case I've
intentionally tried to apply it first and then attempt to justify it. For this
particular example, I can see the approach as being worthwhile even if it 
makes for slightly more ugly code.

Of course, no attempt at purity is ultimately successful, and if you take a look
at `World#snapshot`, you will see that this is where I finally throw Smyth's LoD
out the window for the sake of practicality. Feel free to focus on the structure
of the code rather than the algorithm used to process the map, as that is what 
matters most in this article:

```ruby
module SpaceExplorer
  class World
    DELTAS = (-2..2).to_a.product((-2..2).to_a)

    # ...

    def snapshot
      snapshot = DELTAS.map do |rowD, colD|
        if colD == 0 && rowD == 0
          "@"
        else
          @data[@row + rowD][@col + colD]
        end
      end

      text = snapshot.each_slice(5).map { |e| e.join(" ") }.join("\n")

      yield text
    end
  end
end
```

Among other things, we see here the familiar chain of `Enumerable` methods 
slammed together, all of which return values that are not immediate parts of
the `World` object:

```ruby
text = snapshot.each_slice(5).map { |e| e.join(" ") }.join("\n")
```

Although I could probably have written some cumbersome adapters to make this
code conform to Smyth's LoD, I think that would be a wasteful attempt to follow
the letter of the law rather than its spirit. This is especially true when you
consider that Smyth and many other early adopters of the classical Law of
Demeter were working in languages that had a clear separation between objects
and data structures, so they would not necessarily have considered core
structures to be "objects" in the proper sense. In Ruby, our core structures are
full-blown objects, but that does not mean they need to follow the same rules 
as our domain objects.

I would love it if you'd share a comment with your own thoughts about 
the philosophical divide between data structures and domain objects, and
also encourage you to read [this post from Bob Martin](https://sites.google.com/site/unclebobconsultingllc/active-record-vs-objects) 
on the topic, but I won't dwell on the point for now. We still have
work to do!

With the output in hand, all that remains to be done is to ferry it back to the
IRC channel that requested it. Looking back at the relevant portion of `Rover#process_command`, 
you can see that the yielded text from `World#snapshot` is passed on to 
another `Radio` object:

```ruby
@world.snapshot { |text| @radio_link.transmit("\n#{text}") }
```

This `Radio` object holds a reference to the `MissionControl` object that sent
the original `SNAPSHOT` command, and the path back to it is identical to the
path the command took to get to the `Rover` object, just in reverse. I won't
explain that process again in detail, as all that really matters is that
`MissionControl#receive_command` eventually gets run. This method is just as
boring as the `send_command` method we looked at earlier, serving as a direct
bridge to the UI. I've used a Cinch-based IRC bot in this example, but anything
with a `msg()` method will do:

```ruby
module SpaceExplorer
  class MissionControl
    # ...

    def receive_command(command)
      @narrator.msg(command)
    end
  end
end
```

At this point, a message is sent to the IRC channel and the out-and-back trip is
completed. Despite being a fairly complicated feature, Smyth's LoD was mostly
followed throughout, and things got weird in only a few places. That said,
if you have a devious mind, you are likely to have already realized that the
relative simplicity of this code is deceptive, because there are
so many places things can go wrong. Let's talk a little more about 
that now.

### GROUP PROJECT: Exploring our options for failure handling

Smyth's Law of Demeter promises three main consequences: less complex method
definitions, a decrease in temporal coupling, and a robust way of handling
failures. Although the example I've been using provides some evidence for the
first two claims, I intentionally avoided working on error handling to leave
something for us to think through together.

Your challenge, if you choose to accept it, is to think about what can go wrong
in this simulation and to come up with ways to handle those problems without
violating Smyth's LoD. Off the top of my head, I can think of several trivial
problems that exist in this code, but I'm sure there are many other things that I
haven't considered.

If you want to start with some low-hanging fruit, think about what happens when
an invalid command is sent, or what happens when the rover moves off the edge of
the fixed-size map it is currently using. If you want to get fancy, think about
whether the rover ought to have some safety mechanism that will prevent it from
driving into boulders, which it is currently perfectly happy to do. Or, if you
want to get creative, find your own way of breaking things, and feel free to ask
me clarifying questions as you go.

Any level of participation is welcome, ranging from asking a "What if?"
question after reading through the code a bit to grand-scale patches that make
our clunky little rover bulletproof. As I said at the beginning of this
article, my purpose in introducing Smyth's LoD to you was to start a
conversation, and I think this is a fun way to do exactly that.

The [full source for the simulator](https://github.com/elm-city-craftworks/space_explorer) 
is ready for you to tinker with, so go forth and break stuff!

### Reflections

Although I am fairly happy with how the simulator experiment turned out, it is 
hard to draw very many conclusions from it. In very small greenfield 
projects, it is hard  to see how any design principle will ultimately 
influence the full software development lifecycle. That having been said,
it did serve as a great testbed for exploring these ideas and can be a 
stepping stone toward trying these techniques in more practical settings.

I tend to think of software principles as being descriptive rather than
prescriptive; they provide us with convenient labels for particular approaches
to problems that already exist in the wild. If you've seen or worked on some
code that reminds you of the ideas that Smyth's Law of Demeter attempts to
capture, I'd love to hear about it.

I'd also love to hear about whatever doubts have been nagging you as
you worked your way through this article. Every software design strategy has its
strengths and weaknesses, and sometimes we make the mistake of emphasizing the
good parts while downplaying the bad parts, especially when we study new things.
With that in mind, your curmudgeonly comments are most welcome, as they tend to 
bring some balance along with them.

> **NOTE:** I owe a huge hat-tip to [David
> Black](http://twitter.com/david_a_black), as he was the inspiration for
this article. He and I were collaborating on a more traditional
treatment of the Law of Demeter; we each found our own divergent ideas to
investigate, but I definitely would not have written this article if he hadn't
shared his thoughtful explorations with me.
