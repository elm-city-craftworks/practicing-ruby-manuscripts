> **NOTE:** This issue of Practicing Ruby was one of several content experiments 
that was run in Volume 6. It uses a cookbook format (e.g. problem -> solution -> discussion)
instead of the traditional long-form article format we use in most Practicing Ruby articles.

**Problem: It is hard to work on side projects without obsessing over technical
details and infrastructure decisions.**

There are lots of reasons to work on projects in your spare time, but there are 
two that stand out above the rest: scratching a personal itch by solving a 
real problem, and gaining a better understanding of various programming tools 
and techniques. Because these two motivating factors are competing interests, 
it pays to set explicit goals before working on a new side project.

That said, remembering this lesson is always a constant struggle for me. 
Whenever I'm brainstorming about a new project while taking a walk or sketching
something on a white board, I tend to develop big dreams that extend far
beyond what I can realistically accomplish in my available free time. To show
you exactly what I mean, I can share the back story on what that lead me to 
write the article you're reading now:

> Because I have a toddler to take care of at home,
meal planning can be a major source of stress for me. My wife and I are 
often too distracted to do planning in advance -- so we often need to make a 
decision on what to eat, put together a shopping list, go to the grocery 
store, and then come home and cook all in a single afternoon. 
Whenever this proves to be too much of a challenge for us, we order 
takeout or heat up some frozen junk food. Unsurprisingly,
this happens far more often than we'd like it to.

> To make matters worse, our family cookbook has historically consisted of a 
collection of haphazardly formatted recipes from various different sources. Over time, we've
made changes to the way we cook these recipes, but these revisions almost
never get written down. So for the most part, our recipes are inaccurate, 
hard to read, and can only be cooked by whichever one of us knows its quirks.
Most of them aren't even labeled with the name of the dish, so you need to
skim the instructions to find out what kind of dish it is!

> On one of my afternoon walks, I decided I wanted to build a program
that would help us solve some of these problems, so that we could make fewer
trips to the grocery store each week, while reducing the friction and cognitive
load involved in preparing a decent home cooked meal. It all seemed so simple in
my head, until I started writing out my ideas!

By the time I got done with my brain dump, the following items were on the 
wish list of things I wanted to accomplish in this side project:

* I figured this would be a great time to try out Rails 4, because this project
would obviously need to be a web application of some sort.

* It would be another opportunity for me to play around with Bootstrap.
I am weak at frontend development, but I am also bothered by poor visual 
design and usability, so it seems to be a toolset that's worth learning for
someone like me.

* I had been meaning to explore using the Pandoc toolchain from within Ruby programs
to produce HTML and PDF output from Markdown files, so this would be a perfect 
chance to try that out. This would allow me to have recipes look nice both
on the web and in print.

* It would be really cool if the meal planner would look for patterns in our
eating habits and generate recommendations for us once it had enough data to
draw some interesting conclusions.

* It would be nice to have a way of standardizing units of measures so that we
could trivially scale recipes and combine multiple recipes into a shopping list
automatically.

* It would be neat to support revision control and variations on recipes within
the web application, in addition to basic CRUD functionality and search.

* It would be great to be able to input a list of ingredients we have on hand
and get back the recipes that match them.

I won't lie to you, the system described above still sounds awesome to
me. Building it would involve lots of fun technological challenges, and
it'd be amazing to have such a powerful tool available to me. But it also
represents a completely unreasonable set of goals for someone who has so little
productive free time that even cooking dinner seems like too much work.
Sadly, it's easy to forget that sometimes.

To make a long story short: my initial brainstorming session proved to be 
a pleasant day dream, but it wasn't a real solution to my problems. Instead, 
what I needed was an approach that could deliver modest results in fractions 
of an hour rather than expecting to put in weeks of hard work. To do that, 
I'd have to radically scale back my expectations and set out in search of 
some low hanging fruit.

---

**Solution: Build a single useful feature and see how well it works in practice 
before attempting to design a full-scale application or library.**

When I catch myself getting caught up in *architectural astronaut* mode, 
I tend to bring myself back down to earth by completely inverting my approach. 
I drop the notion of building a perfect system, and instead focus on
building a single useful feature as quickly as possible without
any concern for elegance.

Like paratroopers in the night, the goal is not to find the exact right
place to start from, but instead to dive head first into unknown territory
and try to secure a foothold. Although there were many possible starting
points for working on my meal planner, I decided to start with the one that
seemed most simple in my mind: generating randomized selections of dishes 
to cook over a three day timespan.

Because this feature involved automating a small part of what was originally 
a completely manual process, the first step was to do a bit of tedious
data entry work. I thumbed through our binder of recipes and pulled out 16 
of them that we had cooked recently. I then used a felt-tipped pen to
number each recipe in ascending order, which yielded a rudimentary
way of looking up recipes by number.

This may seem like an ugly way of doing things, but I did it to save myself 
the trouble of figuring out how to convert my haphazardly printed recipes
into text-based source files. I also wanted to defer the decision of what to 
"officially" name each dish, and this way of labeling things allowed me to 
do that in the same way that an autoincrementing primary key does for 
database records.

Once I finished manually indexing my recipes, I compiled a CSV file 
that looked something like what you see below:

```
name,label
"Veggie Cassarole w. Swiss Chard + Baguette",1
"Stuffed Mushroom w. Leeks + Shallots",2
"Lentil Soup w. Leeks + Kale",3
"Spinach + White Bean Soup",4
```

This dataset introduces some human-readable names for the dishes, because 
I didn't want to have to thumb through the recipe book to see what 
"Dish #4" is actually made out of. This system also has an advantage of 
being truly arbitrary, unlike alphabetical order in which 
"Spinach + White Bean Soup" is just as reasonable a label as 
"White Bean + Spinach Soup", but the two would appear in totally 
different positions in the book. Although this may have been premature 
optimization, it came at a low cost and gave me some peace of mind, 
so that made it worthwhile to me.

Before writing any code, I manually tested the index to see how easy it
would be to look up a recipe by number. It proved to be no more complicated
than flipping to a particular page of a book, so it turned out to be a 
good enough system to start with. After that quick usability test,
I hacked together the following script to give me randomized meal selections:


```ruby
require "csv"

candidates = []

CSV.foreach("recipes.csv", :headers => true) { |row| candidates << row }

puts "How about this menu?\n\n" + candidates
  .sample(3)
  .map { |e| "* #{e['name']} (#{e['label']})" }
  .join("\n")
```

When run, this script produces the following output:

```
How about this menu?

* Tunisian Chickpea Stew (10)
* Tomato + Feta w. Green Bean Salad (13)
* Stuffed Mushroom w. Leeks + Shallots (2)
```

The first time I used this new tool, I had to run it a couple times 
in order to come up with three dishes that appealed to me at that moment.
However, this was still far less daunting than trying to choose three
dishes directly from our disorganized cookbook. With only about 30 minutes 
of work invested into this project (not counting the ridiculously 
ambitious brainstorming session), I already had a tool that
was doing something useful for me. Content with my progress for the day,
I plucked my chosen recipes from the binder we keep them in and headed off 
to the grocery store.

While shopping for ingredients and cooking the meals, I was reminded how 
terribly organized most of our recipes truly were. Some even made it hard
to see exactly what ingredients were needed, and nearly all of them listed
steps in a semi-arbitrary sequence of muddled paragraphs. Almost none of the
recipes were at the scale we tended to cook them at, so we'd need to do mental
math both when cooking and when shopping which occasionally lead us to make
mistakes.

I knew I didn't have the available free time to build a full-blown content management 
system for our recipes, but I wondered whether I could apply the lesson learned
from earlier that day to improve things in a low cost way. I eventually realized
that my idea of using Pandoc to convert markdown formatted recipes into PDFs 
wouldn't be so bad if I didn't need to build a whole system around it, so I 
decided to take a few recipes and manually format them in a way that was 
appealing to me. 

My personal preferences for organizing recipes is not
especially important here, but if you're curious, 
[check out this sample document](http://notes.practicingruby.com/barley_risotto.pdf).
The main goal I had was to limit the amount of information I needed to keep in my
mind at any given point in time, and to make the different transition points in
the cooking process explicitly clear.

The process of formatting the recipes this way was time consuming, 
and actually took longer than writing the randomizer program and preparing 
its data. With that in mind, I decided that I would work on this as I found 
time for it, rather than trying to get everything normalized into this
format all at once. The improved formatting definitely made a difference,
but I had to consider whether my time might be better used elsewhere.

Despite the mixed results, the lesson I learned from this experiment is that if
had I focused on solving the content management problem first, I may have spent
a good chunk of time building a complex system without
gaining an appreciation for the actual data entry costs. I also came to
realize that markdown files in a git repository seemed to be every bit
as comfortable for me as a web application could be, and I didn't need
to build anything in order to use them. This would be a terrible UI for
a general purpose application, but it worked great for me.

Over the course of a couple weeks, I kept using the meal randomizer with 
some degree of success, finding small opportunities to improve it along 
the way. Two main issues surfaced fairly early on in my use of the program:

1. Without some way of filtering recommended meals based on how much effort they 
required, I had to mentally ignore our more time-consuming dishes most
of the time.

2. Sixteen dishes is too small of a selection to get enough variety to avoid
duplicate suggestions and repeatedly seeing dishes you've ate recently.

For the first issue, I decided that I didn't need something as precise as preparation 
time in minutes, but instead could use a simple subjective rating system from 1-5 
where the low end represents dishes that can be made almost instantaneously 
(like a grilled cheese sandwich), the middle represents a dish we'd cook on a regular
evening, and the high end represents an all-day cooking session. I'd set up
the program to select dishes with an effort score of 3 or lower by default, but allow
for the limit to be set via an argument.

But it's easy to see that fixing the first problem would only make the second
issue worse. I briefly thought through some clever solutions to the variety problem, like
keeping track of a history or doing other things to make the selection process smarter,
but eventually decided that simply increasing the number of dishes in the data set
would be easiest. So I dug back into some of our other recipes that we had online,
and also added things like sandwiches and other quick meals that we don't cook
from a recipe. Most of these didn't have a printout in our cookbook, so I just
labeled them with an "X", to indicate that they'd need to be imported later.

Thanks to my active laziness, the script only required very minor changes. The
updated version is shown below:

```ruby
require "csv"

candidates = []
effort     = ARGV[0] ? Integer(ARGV[0]) : 3

CSV.foreach("recipes.csv", :headers => true) { |row| candidates << row }

puts "How about this menu?\n\n" + candidates
  .select { |e| Integer(e['effort']) <= effort }
  .sample(3)
  .map { |e| "* #{e['name']} (#{e['label']})" }
  .join("\n")
```

Similarly, the CSV file only required a tiny bit of rework to add the effort
ratings:

```
name,label,effort
"Veggie Cassarole w. Swiss Chard + Baguette",1,3
"Stuffed Mushroom w. Leeks + Shallots",2,3
"Lentil Soup w. Leeks + Kale",3,3
"Spinach + White Bean Soup",4,2
...
```

With a dataset including over 30 dishes, and a filter that removed the most 
complex ones by default, the variety of the recommendations got a lot 
better. This greatly reduced the number of times I needed to run the script
before I could put together a meal plan. A smarter selection algorithm
could definitely make the tool even more helpful, but these small changes 
made a huge difference on their own.

Another week passed, and I eventually realized that I don't particularly
like having to pop open a terminal and run a command line program simply
to decide what I want to have for dinner. After another half hour of work, 
I wrapped the script in a minimal web interface using Sinatra. Throwing 
that app up onto Heroku allowed me to do my meal planning via the web 
browser. The UI is nothing special, but it gets the job done:

![](http://i.imgur.com/Y1C3sxt.png)

As you might expect, the code that implements this UI isn't
especially exciting, it's just basic glue code and an ERB 
template:

```ruby
require "sinatra"
require "csv"

def meal_list(candidates, effort)
  "<ul>" + 
    candidates.select { |e| Integer(e['effort']) <= effort }
              .sample(3)
              .map { |e| "<li>#{e['name']} (#{e['label']})</li>" }
              .join + 
  "</ul>"
end

get "/" do
  candidates = []
  effort     = Integer(params.fetch("effort", 3))
  meal_list  = "#{File.dirname(__FILE__)}/../recipes.csv"

  CSV.foreach(meal_list, :headers => true) do |row| 
    candidates << row 
  end

  @selected = meal_list(candidates, effort)
  
  erb :index
end

__END__

@@index
<html>
  <body>
    <h1>How about these meals?</h1>
    <%= @selected %>
  </body>
</html>
```

It's worth noting that the code above is about at the level of complexity
where more formal development practices start to pay off. But since I 
managed to squeeze three weeks of active use out of the tool before
getting to this point, I definitely won't mind doing some cleanup work 
if and when I decide to add more features to it.

**Discussion**

The main thing I hope you will take away from this article is that "keeping
things simple" is term that we say often but rarely practice. This can have a
painful effect on our daily work, but is disasterous for our side projects,
because we often work on them with a tight time budget. 

Speaking from personal experience, I've lost count of how many Rails applications 
skeletons I've built that started with big dreams and ended up with nothing more 
than a couple database models, a few half-finished CRUD forms, and an
authentication system, but no actual features to speak of. I guess I am
just extremely good at overestimating how much time and motivation I'll
have for building the things I think of day to day.

Increasingly, I've been trying to think of software as a support system for
solving human problems, rather than some sort of artifact that holds intrinsic
value. Software is extremely expensive to build and maintain, so it pays to
write as little code as possible. This does not just mean writing terse
programs: it means spending more time and creativity on practical problem
solving, so that you can focus your energy on making people's lives
easier rather than obsessing over technical issues. Adopting this mindset
can lead you to being more thoughtful when you build software for others,
but it also serves as a reminder that you can and should enjoy the fruits 
of your own labor.

Even though programming can be fun in its own right, you don't need to view
every software project as an opportunity to solve interesting coding puzzles.
When measured in terms of functional value rather than implementation details,
sometimes the most elegant solution is a script that you cobbled together during
a lunch break, because it cost you almost nothing but still managed to do
something useful for you. These opportunities appear around every corner, you
just need to be prepared to take advantage of them when they arise. I hope
the story I've shared in this article has helped you learn what to look out for.

*Do you have an example of some code you wrote that took very little effort but
still ended up being very useful for you? If so, please share your story in
the comments section below.*
