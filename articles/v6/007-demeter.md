*This article was contributed by [David A. Black](https://twitter.com/david_a_black), Lead Developer at Cyrus Innovation. David is a long-time Ruby developer, author, trainer, 
speaker, and community event organizer. He is the author of The Well-Grounded Rubyist (Manning Publications, 2009).* 

A lot has been written and said about the Law of Demeter. I'd read and heard a
lot about the law before I ever went back and looked at the seminal, original
papers that described it. In spite of how much I thought I knew about the law, I
found those original papers quite enlightening and absorbing. 

I've been particularly absorbed in two articles: the 1988 OOPSLA paper
"Object-Oriented Programming: An Objective Sense of Style" by Karl J.
Lieberherr, Ian M. Holland, and Arthur J.Riel, and the 1989 article "Assuring
Good Style for Object-Oriented Programs" by Lieberherr and Holland. 
The two papers are of course closely related. But what I've found interesting,
aside from just the process of studying and absorbing information about the Law
of Demeter at its source, is considering the ways in which they differ. 

Both papers posit that there are different versions of the Law of Demeter. But
the taxonomies they construct for the law differ considerably from each other.
A lot of further thought and work, evidently, went into the law between 1988 and
1989. 

I'm going to put the two taxonomies, and the differences between them, under a
microscope -- at least, a medium-powered microscope. I won't recapitulate
everything in the two articles, but I'll go into enough detail to set the
stage for some reflective and interpretive observations about why the law might
have evolved in the ways it did in a relatively short time. 

I'll then conclude with a couple of speculative, open-ended thoughts about the
Law of Demeter as it relates to general problems of code organization and best
practices in programming -- a probably small-scale but hopefully interesting
perspective that I've dubbed "Metademeter". 

## THE 1988 TAXONOMY

> **Note:** In addition to the type and object versions of the law described here,
the 1988 article talks about the *strong* and *weak* versions of the law. That
distinction has to do with whether or not it's considered permissible to send
messages to inherited instance variables. The strong version says no; the weak
version says yes. I'm not going to go into detail about that aspect of the 1988
taxonomy, but it's certainly worth a look at the original article.

In 1988, the three authors state the Law of Demeter, initially, in the following
terms: 

For all classes C, and for all methods M attached to C, all objects to which M
sends a message must be instances of classes associated with the following
classes: 

```
  1. The argument classes of M (including C).
  2. The instance variable classes of C.

  (Objects created by M, or by functions or methods which M calls, and objects
  in global variables are considered as arguments of M.) 

  (pg. 325)
```

There follows an extensive treatment of the motivation for and implications of
the law. Included in this treatment is consideration of a case where strict
adherence to the law nonetheless runs contrary to its intended effect. Consider
a case where there's a kind of circular structure to the instance variable types
of a set of classes. The following example is adapted from the article, and while Ruby doesn't enforce instance variable classes, the code illustrates the basic difficulty the authors identify: 

```ruby
  class A
    def initialize
      @b, @c, @d, @e = B.new, C.new, D.new, E.new
    end

    def bad_style
      b.d.e
    end

    attr_reader :b, :c, :d, :e
  end

  class B
    def initialize
      @c, @d = C.new, D.new
    end

    attr_reader :c, :d
  end

  class C; end

  class D
    def initialize
      @e = E.new
    end

    attr_reader :e
  end

  class E; end

  a = A.new
  a.bad_style
```

The `bad_style` instance method in class `A`, called at the end of the example, triggers a series of calls. The first, a call to the reader method `b`, returns `a`'s instance variable `@b`, which is an instance of class `B`. Then the message `d` is sent to that `B` instance; the result is an instance of `D`, namely the instance held in the instance variable `@d` of the `B` instance. Sending `d` to a `B` instance is legal, Demeter-wise, because one of `a`'s instance variables is of class `B`. Then the `D` instance gets the message `e`; this is also OK for the same reason. 

So you've only "talked to" objects belonging to classes corresponding to
instance variables of your instance of `A`, but, as the article states, *"the
method looks two levels deep into the structure of instance variable first,
violating the ideals of information-hiding and maintainability."* 

The authors propose a second formulation of the law as a way around this
problem. Note that here the law is stated in terms of objects, not classes:

```
  For all classes C, and for all methods M attached to C, all objects to which M
  sends a message must be:

    * M's argument objects, including the self object or...
    * The instance variable objects of C. 

  (Objects created by M, or by functions or methods which M calls, and objects
  in global variables are considered as arguments of M.) 

  (327)
```

The downside to this object version of the Law of Demeter is that it makes it
hard to do compile-time checking. The conclusion of the authors is that *"to
retain easy compile-time checking we require the Law's formulation in terms of
types. We feel that such path[o]logical cases as the one above will not occur
often enough to cause problems" (327).*

Still, the object version of the law serves as an important guide for
programmers. Toward the end of the article, the authors provide formulations of
the law for several specific object-oriented languages, using the law's object
version. Of the languages for which they offer such formulations, the closest to
Ruby is Smalltalk-80. In that language, the authors state that message-sending
should be restricted to:

* an argument object of [the method] M including objects in pseudo variables
  "self" and "super" or
* an instance variable object of the class to which M is attached. 
  (332)

As before, newly-created objects and objects in global variables count as
argument objects.

The *object* version of the law casts a somewhat wider net, as far as languages
are concerned, than the first, *class* version. Certainly for a dynamic language
like Ruby, where static code analysis can do relatively little for you and
compile-time checking doesn't exist, the object version makes sense. It also
makes sense in languages where there's no such thing as the *type* of an
instance variable; Ruby instance variables, for example, can be assigned any
object and even different objects at different times. The object version of the
law of Demeter, as laid out in 1988, doesn't specifically address the matter of
reassigning to instance variables but might provide enough structure and
discipline to give you pause if you find yourself doing that. 

Let's move a year forward. 

## THE 1989 TAXONOMY

Like the 1988 article, the 1989 article presents the Law of Demeter in two major
versions: the class version and the object version. Here, though, the
definitions of the two versions have changed in interesting ways, and the class
version, in turn, is broken down into the minimization version and the strict
version. 

The 1989 taxonomy of the law rests on the notion of clients and suppliers.
Clients are methods; suppliers are classes. If method M calls method N on an
instance of class C (or on class C itself), then M is a client of both the
method N and the class C. In turn, C is a supplier to M. (There are some further
subtleties but this is the thrust of how clients and suppliers relate to each
other.) 

In the client/supplier relationship, the supplier class may be an *acquaintance*
class (what's often paraphrased as a "stranger"), or it may be a preferred
supplier (sometimes called a "friend"). Preferred suppliers, in brief, include:

* the subcategory *preferred acquaintance*, consisting of:
  * the class(es) of object(s) instantiated inside the client method
  * the class(es) of global object(s)
* the class of an instance variable (or a superclass)
* the class of an argument to the method (or a superclass)

The article summarizes the two sub-versions of the class version of the law as follows:

```
  Minimization version: Minimize the number of acquaintance classes of all
  methods.

  Strict version: All methods may have only preferred-supplier classes. 
  (40-41)
```

As you can see, the 1989 taxonomy involves more terms and definitions than the
1988 taxonomy. It's a denser account of the law. But there's something gained
for the added complexity. Everything is organized from the root of the structure
upward. The categories of newly created objects and global variables, both of
which were literally added via parenthetical addenda to the 1988 versions of the
law, are more smoothly integrated into the model in 1989. Every imaginable
object that might be sent a message falls somewhere on one consistent spectrum,
ranging from mere acquaintance (to be avoided) to preferred acquaintance
(acceptable but still flagged as not quite a full *friend*) to preferred
supplier (the real friends). I have found that the 1989 taxonomy requires
longer and deeper study than the 1988 taxonomy, but that it repays careful
reading.

And that's just the class version of the law. As before, there's also an object
version, summarized as follows:

```
  All methods may have only preferred-supplier objects. 
```

Note the shift, subtle but important, from *classes* to *objects*, as compared
with the strict version of the class version of the law. Focusing on objects
allows for inclusion of such constructs as self and super. Moreover, the authors
make the following interesting point about the object version of the law:

```
    While the object version of the law expresses what is really wanted, it cannot
    be enforced at compile time. The object version serves as an additional guide
    in addition to the class version of the law (42).
```  

There's a kind of "bend before you break" principle at work here. The Law of
Demeter is not all-or-nothing, as regards the ability to do compile-time
checking. It's also something that you can, and in some cases must, bake into
your programming habits as you go along. 

As in 1988, the 1989 authors present a kind of checklist of how to enforce the
law in the cases of several specific languages (C++, CLOS, Eiffel, Flavors, and
Smalltalk-80). Interestingly, the 1989 account of how to apply the language to
C++ recommends the strict version of the class form of the law -- whereas in
1988, the C++ guidelines suggested the object version. For the other languages,
the 1989 guidelines refer to the object version, though there's some explanatory
text suggesting that in any statically-typed language (including Eiffel), "the
class form is most useful because it can be checked by a modified compiler"
(47). 

Once again, the Smalltalk-80 criteria come the closest to what we might
formulate for Ruby:

```
  Smalltalk-80, object form. In all message expressions inside method M the
  receiver must be one of the following objects:
    * an argument object of M, including objects in the pseudovariables Self and
      Super,
    * an immediate part of Self, or
    * an object that is either an object created directly by M or an object in a
      global variable (47).
```

(An "immediate part of Self" can be an instance variable. It is not explicitly stated in the article whether or not the concept of "immediate part" can also include collection elements.) 

The salient point here is that the framers of the Law of Demeter were at pains
to welcome dynamic languages to the fold. This is directly related to the
complexity of the taxonomy of the law. Exploding the law into several versions
and sub-versions allows for close, reasoned analysis of what can and cannot be
checked at compile time, as well as other details and underpinnings of the law's
rationale and logic. In the end, though, everything converges back on the
original purpose: providing programmers using object-oriented languages with a
set of principles that reduce inter-class dependencies. 

## METADEMETER

The Law of Demeter is engineered to help programmers using object-oriented
languages gain a lot of clarity of code for a relatively small price. Of course,
there's a whole world of refactoring out there; the Law of Demeter is not the
only guideline, or set of guidelines, for making code better, clearer, and more
maintainable. It would be a mistake to lump all refactorings as "Demeter-ish";
that does justice neither to the Law of Demeter nor to the other refactorings. 

And yet... I'm intrigued by the possibility that recognizable aspects of the Law
of Demeter might surface in contexts other than those for which the law was
originally formulated. I'm not going to push this point very far. I've got one
example that I find suggestive, and I'll leave it at that. See what you think. 

The 1989 article describes a programming technique that the authors call
*lifting*. To illustrate lifting, here's an example of an acquaintance class,
and a Demeter violation:

```ruby
  class Plane
    attr_accessor :name
  end

  class Flight
    attr_accessor :plane
  end

  class Person
    def itinerary_for(flight)
      "Flight on #{flight.plane.name}"
    end
  end
```

Here, `Plane` is an acquaintance class of `Person`. `Flight` isn't; `Flight` is a
preferred supplier class, because it's the class of an argument. `Flight` is a
friend; `Plane` isn't, and by calling name on a plane object we're operating
outside of the Law of Demeter. 

You can fix this Demeter violation by "lifting" the method that provides the
information into the external class:

```ruby
  class Plane
    attr_accessor :name
  end

  class Flight
    attr_accessor :plane
    def plane_name
      plane.name
    end
  end

  class Person
    def itinerary_for(flight)
      "Flight on #{flight.plane_name}"
    end
  end
```

Note that this code is longer than the original. It's not uncommon for
Demeter-compliant code to have more methods than non-compliant code. The gain,
on the other hand, lies in the way the code is organized, and the ease with
which the code can be maintained and changed. If you change the way `Plane#name`
works, and you want to make sure it's still used consistently in all your code,
you only need to hunt for classes that use `Plane` objects as arguments or
instance variables, and make sure the code is still correct. In the first
version of the plane code, you'd have to dig deep into every class in the
program, since you have no guidelines for figuring out where `Plane#name` is
likely to be called or not called. 

Now for the part about aspects of Demeter cropping up outside the original
context. I'm thinking specifically of programming controllers and view templates
in Rails. Templates are already a bit of an oddity, in terms of object-oriented
programming, because of the way they share instance variables with controller
actions: assign something to `@buyer` in the controller, and you can use `@buyer` in
the view. Instance variables always belong to self, and self in the controller
is different from self in the view -- yet the instance variables resurface. 

In case you've ever wondered, this is brought about by an explicit assignment
mechanism: when a view object is created, it copies over the controller's
instance variables one by one into instance variables of its own. So we've got a
domain-specific and kind of hybrid situation: two self objects sharing, or
appearing to share, instance variables.

So where does lifting come in, in any sense reminiscent of the Law of Demeter?

Consider a view snippet like this:

```erb
  <% @user.friends.each do |friend| %>
    <% friend.items.each do |iitem| %>
      <%= friend.name %> has a(n) <%= item.description %>
    <% end %>
  <% end %>
```

I don't want to get into a whole debate here about whether or not it's ever
acceptable to hit the database from the views. My philosophy has always been
that you should be allowed to send a message to any object that the controller
shares with the view. By that reckoning, @user.friends would be acceptable, and
it's up to the controller to eager-load the friends if it wants to. 

But what about `friend.items`? Here we're wandering out on a limb; we're an extra
level of remove from the original object. I can't claim that this is exactly the
situation envisioned by the framers of the Law of Demeter -- but it reminds me
strongly of Demeter-ish situations. And I would propose a Demeter-ish solution,
based on the lifting technique: "lift" one of the method calls back into the
controller. Here's a simple version:

```ruby
  def show
    @user = current_user
    @friends = @user.friends
  end
```

And then in the view:

```erb
  <% @friends.each do |friend| %>
    <% friend.items.each do |item| %>
      <%= friend.name %> has a(n) <%= item.description %>
    <% end %>
  <% end %>
```
In "metademeter" terms, we're talking only to the immediate parts of the
`@friends` object -- in this case, the elements of a collection. I believe there's
room for debate, within discussions of the law itself, on whether or not
collection elements count as *immediate parts* of an object. But here it seems a
good fit. Again, keep in mind that this is just an observation of what I would
call a Demeter-ish way of thinking about code. The Rails controller/view
relation is not the same as the relation between and among classes and methods
that the Law of Demeter directly addresses. And the object whose immediate
parts I'm restricting myself to is not the self object; it is, itself, an
instance variable object. Still, I think we could do worse in a situation like
this than to be inspired to think of a motto like "talk only to your friends",
understanding "friends" to be objects that lie one method call away from the
original ActiveRecord objects handed off by the controller. 

That's the extent of my metademeter musings. Meanwhile I hope you'll continue to
study and contemplate the Law of Demeter, and explore the many writings and
discussions and debates that you'll find surrounding it. I've presented no more
than a subset of what has been or can be said; but I hope that this trip back
to the original statements on the law has been engaging and worthwhile.

## REFERENCES AND FURTHER READING

* [The 1988 article (in special OOPSLA issue of SIGPLAN Notices)](http://www.ccs.neu.edu/research/demeter/papers/law-of-demeter/oopsla88-law-of-demeter.pdf)

* [The 1989 article, available through IEEE](http://ieeexplore.ieee.org/xpl/login.jsp?tp=&arnumber=35588&url=http%3A%2F%2Fieeexplore.ieee.org%2Fxpls%2Fabs_all.jsp%3Farnumber%3D35588)

* [A somewhat different version of the 1989 article, in PostScript form](ftp://ftp.ccs.neu.edu/pub/research/demeter/documents/papers/LH89-law-of-demeter.ps)

* [Another 1988 document, with some further/interim reflections on the Law of Demeter, by Lieberherr and Holland](http://www.ccs.neu.edu/research/demeter/papers/law-of-demeter/law-formulations/ss.tex)

* [An excellent account of the Law of Demeter and its practical uses, by David Bock](
http://www.ccs.neu.edu/research/demeter/demeter-method/LawOfDemeter/paper-boy/demeter.pdf)

* [Practicing Ruby Issue 5.2: Rocket Science and the Law of Demeter](https://practicingruby.com/articles/shared/gulrqynwlywm)

