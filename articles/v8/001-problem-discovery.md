Imagine you're a programmer for a dental clinic, and they need your help to build a vacation scheduling system for their staff. Among other things, this system will display a calendar to staff members that summarizes all of the currently approved and pending vacation requests for the clinic, grouped by role.

The basic idea here is simple: If a dental assistant wants to take a week off some time in July, it'd be more likely to get time off approved for a week where there was only one other assistant out of the office than it would be for a week when five others were on vacation. Rather than waiting for a manager to review their request (which might take a while), this information can be supplied up front to make planning easier for everyone.

Your vacation request system already has been implemented weeks ago, so you can easily get all the data you need on who is requesting what time off, and who has already had their time off approved. Armed with this information, building the request summary calendar should be easy, right? Just take all the requests and then group them by employee roles, and then spit them out in chronological order. You'll be able to roll out this new feature into production by lunch time!

You grab your morning coffee, and sit down to work. Before you can even open your text editor, an uncomfortable realization weighs heavily upon you: Roles are actually a property of shifts, not employees. Your clinic is understaffed, and so some employees are cross-trained and need to wear multiple hats. To put it bluntly, there's at least one employee that's not precisely a receptionist, and would be more adequately described as "receptionish". She helps out in the billing office at times, and whenever she's working there, the clinic is down a receptionist.

You do have access to some data about individual shifts, so maybe that could be used to determine roles. By the time a shift is approved, the role is set, and you know for sure what that employee is doing for that day. 

You think for a little while. You uncover a few annoying problems that will need to be solved if you decide to go this route.

The shift data is coming from a third party shift planning system, and the import window is set out only ten weeks into the future.  In practice, shifts aren't really firmly committed to until four weeks out, so that makes the practical window even smaller. 

The idea that a given employee's shift in July would be set in stone by March is a fantasy, and so even if you could get at that data, it wouldn't be perfectly accurate. There's also no guarantee that attempting to import five times more data than what you're currently working with won't cause problems… the whole synchronization system was built in a bit of a hurry, and could be fragile in places.

Feeling the anxiety start to set in, you go for a quick walk around the block, and come to the realization that you've gone into problem solving mode already, when you really should be more in the problem discovery phase of things. You haven't even answered the question of how many employees work in multiple different roles, and you're already assuming that's a problem that needs a clear solution.

An idea pops into your head. You rush to your desk, and pop open a Rails console in production. You write a crude query and then massage the data with an ugly chain of Enumerable methods, and end up with a report that looks like this:

```ruby
 ["Nikko Bergnaum", [["Hygienists", 5]]],
 ["Anderson Miller", [["Billing", 50]]],
 ["Bell Effertz", [["Hygienists", 14]]],
 ["Vicky Okuneva", [["Receptionists", 30]]],
 ["Lavern Von", [["Assistants", 37]]],
 ["Crawford Purdy", [["Receptionists", 40]]],
 ["Valentin Daugherty", [["Hygienists", 61]]],
 ["Eudora Bauch", [["Receptionists", 40]]],
 ["Jaeden Bashirian", [["Assistants", 28]]],
 ["Roel Hammes", [["Dentists", 36]]],
 ["King Schowalter", [["Hygienists", 20]]],
 ["Liam Kovacek", [["Receptionists", 55]]],
 ["Elaina Von", [["Hygienists", 25]]],
 ["Susie Watsica", [["Hygienists", 31]]],
 ["Oswaldo Boyer", [["Hygienists", 20]]],
 ["Gardner Fay", [["Hygienists", 10]]],
 ["Joanny Beatty", [["Assistants", 52]]],
 ["Beth Yost", [["Hygienists", 34]]],
 ["Gerry Torphy", [["Hygienists", 10]]],
 ["Maureen Terry", [["Hygienists", 9]]],
 ["Maritza Kemmer", [["Billing", 25]]],
 ["Morton Hudson", [["Dentists", 61]]],
 ["Santino Parker", [["Hygienists", 49]]],
 ["Jesse Friesen", [["Hygienists", 31]]],
 ["Dillan Krajcik", [["Hygienists", 44]]],
 ["Travon Koch", [["Hygienists", 16]]],
 ["Audreanne Hand", [["Billing", 47]]],
 ["Coralie Predovic", [["Receptionists", 45]]],
 ["Jovani Schulist", [["Management", 50]]],
 ["Tanner D'Amore", [["Dentists", 41]]],
 ["Jace Nitzsche", [["Dentists", 21]]],
 ["Carolina Waters", [["Receptionists", 40]]],
 ["Terence Howell", [["Dentists", 39]]],
 ["Leann Pacocha", [["Assistants", 2]]],
 ["Alvah Rippin", [["Dentists", 50]]],
 ["Lorenzo West", [["Hygienists", 27]]],
 ["Gideon McKenzie", [["Dentists", 41]]],
 ["Katrine O'Reilly", [["Dentists", 51]]],
 ["Briana Ziemann", [["Dentists", 40]]],
 ["Jerome Harris", [["Dentists", 10]]],
 ["Misael Pagac", [["Assistants", 51]]],
 ["Krista Predovic", [["Assistants", 32]]],
 ["Carole O'Hara", [["Assistants", 42]]],
 ["Adalberto Doyle", [["Management", 49], ["Receptionists", 2]]],
 ["Noel Ortiz", [["Management", 28], ["Receptionists", 1]]],
 ["Monique McLaughlin", [["Receptionists", 43], ["Assistants", 1]]],
 ["Jaleel Graham", [["Billing", 50], ["Receptionists", 18]]],
 ["Ned Reilly", [["Receptionists", 50], ["Assistants", 1]]],
 ["Enrico Schowalter", [["Receptionists", 1], ["Assistants", 55]]],
 ["Caesar Goldner", [["Management", 30], ["Receptionists", 16]]],
 ["Kirstin Weissnat", [["Receptionists", 26], ["Assistants", 28]]],
 ["Guillermo Klein",
  [["Assistants", 41], ["Hygienists", 2], ["Receptionists", 3]]]]
```

This listing shows all the shifts planned for the next ten weeks, with counts for each employee by role. You copy and paste it into a text editor, and delete any of the lines for employees that have a single role. Here's what you end up with:

```ruby
   ["Adalberto Doyle", [["Management", 49], ["Receptionists", 2]]],
 ["Noel Ortiz", [["Management", 28], ["Receptionists", 1]]],
 ["Monique McLaughlin", [["Receptionists", 43], ["Assistants", 1]]],
 ["Jaleel Graham", [["Billing", 50], ["Receptionists", 18]]],
 ["Ned Reilly", [["Receptionists", 50], ["Assistants", 1]]],
 ["Enrico Schowalter", [["Receptionists", 1], ["Assistants", 55]]],
 ["Caesar Goldner", [["Management", 30], ["Receptionists", 16]]],
 ["Kirstin Weissnat", [["Receptionists", 26], ["Assistants", 28]]],
 ["Guillermo Klein",
  [["Assistants", 41], ["Hygienists", 2], ["Receptionists", 3]]]]
```

Now you've whittled the list down to only 9 people. In a business that employees over 50 people, this is 20% of the workforce, which isn't a tiny number. But taking a closer look at the data, you realize something else: even on this list of cross-trained employees, most staff members work in a single role the majority of the time, and very rarely fill in for another role.

Filtering the list again, you remove anyone who works in a single role at least 90% of the time. After this step, only three employees remain on your list:

```ruby
 ["Jaleel Graham", [["Billing", 50], ["Receptionists", 18]]],
 ["Caesar Goldner", [["Management", 30], ["Receptionists", 16]]],
 ["Kirstin Weissnat", [["Receptionists", 26], ["Assistants", 28]]]]]]]
```

Because these employees represent only about 5% of the total staff, you've revealed this problem as an edge case. For the other six employees that substitute for a different role once in a blue moon, you'd have at least 90% accuracy by always labeling them by their primary role. It'd be confusing to refer to them as anything else, at least for the purposes of vacation planning.

In the process of this ad-hoc exploration, you've discovered a reasonably accurate method of predicting employee roles far out into the future: if at least 90% of the shifts they're assigned are for a particular role, assume that is their primary role. Otherwise, label them as cross-trained, listing out all the roles they commonly fill in for. For example, 
Jaleel could be listed as "X-Trained (Billing, Receptionist)",
Kirsten as "X-Trained (Receptionist, Assistant)", and Caesar as "X-Trained (Receptionist, Management)".

Taking this approach will be at least as reliable as using the raw shift data, and requires no major technical changes to the system's under-plumbing. It's also dynamic, in the sense that the system will adaptively relabel employees as cross trained when they're doing more than one role on a regular basis.

Happy with this re-evaluation of the problem, you start working, and you manage to get the feature into production before lunch after all. In the worst case scenario, you can always come back to this and do more careful analysis, peering into the technological and philosophical rabbit hole that made you nervous in the first place. But there's a very good chance that this solution will work just fine, and so it's worth trying it out before venturing out into the darkness.

From this small exercise, you come to a powerful realization:

> Software isn't mathematically perfect reality, it's a useful fiction meant to capture some aspect of reality that is interesting or important to humans. Although our technical biases may aim for logical purity in the code we write, the humans that use our work mainly care about the story we're trying to tell them. We should seek the most simple solutions that allow us to tell that story, even if those solutions lack technical elegance.

In other words, feel free to ignore the man behind the curtain.

