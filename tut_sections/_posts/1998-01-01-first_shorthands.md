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

Let's start with something simple:  the business of opening and closing
the database.  There isn't a whole lot of code here, but there's more
than absolutely necessary.  So, let's cut it to one.  Along the way,
we'll learn a bit about how the library's structured, and the philosophy
behind it.

To begin with, let's remember the way we're doing this now:

{% highlight scala %}
  override def onCreate(savedInstanceState: Bundle) {
    super.onCreate( savedInstanceState )
    ...
    TodoDb.openInContext( this )
    ...
  }

  override def onDestroy = {
    super.onDestroy
    TodoDb.close
  }
{% endhighlight %}

So, we're opening the database at one point in the code, and closing
it at a different point.  Code is easier to read and deal with when
related sections can be physically close together.  So let's do that:

{% highlight scala %}
class TodoItemsActivity extends Activity with PositronicActivityHelpers {
  override def onCreate(savedInstanceState: Bundle) {
    super.onCreate( savedInstanceState )
    ...
    TodoDb.openInContext( this )
    onDestroy{ TodoDb.close }
    ...
  }
  ...
}
{% endhighlight %}

In this example, we've done a couple of things.  First off, we've added
`with PositronicActivityHelpers`.  What that does is to make a number of
Positronic Net facilities available within our `Activity`.  

The second change was to use one, in the `onDestroy{ TodoDB.close }`.
What's going on here?

What it may look like is that we've somehow added a new control structure
to Scala, which is special-purpose and specific to Android activities.
The reality is slightly more prosaic.  Scala allows you to write functions,
and methods, which take other functions as arguments.  The `onDestroy`
that's being called here is a new overload of the `Activity` `onDestroy`
method, which takes a piece of code as an argument (that's the part within
braces), and arranges to run it when its overload for the standard,
no-argument `onDestroy` method is called by the framework.  There are
similar overloads for the other activity lifecycle stages, too, as we'll
see in a minute.  

However, these two lines grouped together still look a bit odd:  if you're
opening a database when an activity starts, you'll almost certainly want to
close it when it shuts down.  Which is a common enough pattern that there's
explicit support for it:

{% highlight scala %}
class TodoItemsActivity extends Activity with PositronicActivityHelpers {
  override def onCreate(savedInstanceState: Bundle) {
    super.onCreate( savedInstanceState )
    ...
    useAppFacility( TodoDb )
    ...
  }
  ...
}
{% endhighlight %}

That does the `openInContext`, and arranges the deferred `close`, in a
single call.  Which is an instance of one of the principles that the
library tries to follow: if there's something you almost certainly
need to do for your app to function correctly, then the framework
ought to do it for you, so you can instead concentrate on the parts
of the job that are unique to your particular app.

<div class="qanote">
 <a class="question">So, what the heck is an <tt>AppFacility</tt>?</a>
 <div class="answer">
   <p>Basically, anything that obeys the same
   open/close protocol.  For now, see the source for details, but it can
   be useful to declare your own to manage background machinery of your
   own.</p>
 </div>
</div>

This is pretty terse --- but we can trim one last bit of fat.  As we've
already mentioned, `PositronicActivityHelpers` can register bits of code
to be executed at any of the activity lifecycle callbacks, not just
`onDestroy`.  One of these is `onCreate`, so we can write:

{% highlight scala %}
class TodoItemsActivity extends Activity with PositronicActivityHelpers {
  onCreate {
    useAppFacility( TodoDb )
    ...
  }
  ...
}
{% endhighlight %}

which does the same job, without nearly so much ceremony.  In
particular, it keeps the necessary business of calling
`super.onCreate(...)` out of the code, which is another example of the
principle that if something must happen for an app to work right, then
the framework ought to do it for you, so you can spend more time
concentrating on the needs of your app, and less on the needs of the
framework.