---
layout: tut_section
title: Tightening things up
summary: Letting the framework take care of some plumbing (and shrinking the UI code by half).
---

If you look at the complete example, there isn't a great deal to it,
but there still is a fair amount of avoidable bookkeeping.  If you've
opened a database when starting an activity, you'll almost certainly
want to close it when the activity terminates; can't something take
care of this for you?  Likewise, after _any_ modification to the list
of items (we have two; we'll have more), we need to call for the
display to be refreshed; is there a way to avoid writing that after
each one?  And we've already seen examples where the way some
particular thing is expressed in code is just clumsy.

In this section, we'll be hammering away at these issues, cutting the
size of our `Activity` class roughly in half.  It's not a huge gain in
this case, since there wasn't all that much code to begin with, but
the gains in other cases are proportionately greater.  Moreover, it
gives us a chance to introduce a bunch of other Positronic Net features
which are interesting in their own right.

### Managing the database

We'll start with a simple one: opening and closing the database.  That's
not really a big deal in this example, but it lets us introduce the 
