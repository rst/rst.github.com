---
layout: tut_section
title: Tightening things up
summary: Letting the framework take care of some plumbing (and shrinking the UI code by half).
---

If you look at the complete example, there isn't a great deal to it,
but there still is a fair amount of avoidable bookkeeping.  After
_any_ modification to the list of items (we have two; we'll have
more), we need to call for the display to be refreshed; is there a way
to avoid writing that after each one?  Likewise, if you've opened a
database when starting an activity, you'll almost certainly want to
close it when the activity terminates; can't something take care of
this for you?  And we've already seen examples where the way some
particular thing is expressed in code is just clumsy.

In this section, we'll be hammering away at these issues, cutting the
size of our `Activity` class roughly in half.  It's not a huge gain in
this case, since there wasn't all that much code to begin with, but
the gains in other cases are proportionately greater.  Moreover, it
gives us a chance to introduce a bunch of other Positronic Net features
which are interesting in their own right.

### Automatically dealing with database updates

Let's start with something fairly pervasive:  updating views when
the underlying data that they're viewing changes.  Right now, that's
at only two points in the code, but it could potentially be more,
and we have to go through the ceremony of

{% highlight scala %}
    TodoItem ! Fetch{ items => adapter.resetSeq( items ) }
{% endhighlight %}

each time.  What's more, if there were other UI components which
also displayed something about the items, they would need to be
updated as well, in all those places.

What we'd like to do is arrange for the `TodoItem` record manager to
automatically run the update on any change.  One way to do this is a
`Watch` message which is like `Fetch`, but runs the query on every
change, not just the first time it's called.  Once we've taken care
of some details, things are slightly more elaborate:

{% highlight scala %}
  override def onCreate( savedInstanceState: Bundle ) = {
    ...
    TodoItems ! AddWatcherAndFetch( this ){ items => adapter.resetSeq(items) }
  }

  override def onDestroy = {
    TodoItems ! StopWatcher( this )
    TodoDb.close
  }
{% endhighlight %}

What this does is that it will run the 

{% highlight scala %}
  { items => adapter.resetSeq(items) }
{% endhighlight %}

change-handler the first time it's called (that's the `Fetch` part of
`AddWatcherAndFetch`), and _on every subsequent change_ until 
`StopWatcher` is called (that's the `AddWatcher` part).  

(The reason for `StopWatcher` is that in an application with multiple
activities, the database can stay open after one of the application's
activities shuts down.  In this circumstance, we obviously want the
notifications to stop.  And since a notifier can have multiple
listeners, it needs to know which one it is being asked to shut down;
the `Activity` uses itself as a marker for that.  Any `Object` (in
Scala terms, any `AnyRef`) will do; the only thing that matters is
that you use the same one in `AddWatcher` and `StopWatcher`.)

## Cleaning up the cleanups.

At this point, we have two instances of a common pattern: setting
something up in `onCreate`, and tearing it down in `onDelete`, which
can be pretty far away in the code (perforce, if `onCreate` is long,
as it often winds up being).  Code is easier to read and deal with when
related sections can be physically close together.  So let's do that:

{% highlight scala %}
class TodoItemsActivity extends Activity with PositronicActivityHelpers {
  override def onCreate(savedInstanceState: Bundle) {
    super.onCreate( savedInstanceState )
    ...
    TodoDb.openInContext( this )
    onDestroy{ TodoDb.close }
    ...
    TodoItems ! AddWatcherAndFetch( this ){ items => adapter.resetSeq(items) }
    onDestroy{ TodoItems ! StopWatcher( this ) }
    ...
  }
  ...
}
{% endhighlight %}

In this example, we've done a couple of things.  First off, we've added
`with PositronicActivityHelpers`.  What that does is to make a number of
Positronic Net facilities available within our `Activity`.  

The second change was to use one, in the `onDestroy{ TodoDB.close }`,
and the similar wrapper around the `StopWatcher` message.  What's going
on here?

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

However, these pairing of lines still look a bit odd.  If you're
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

On to the adapter.  What we have right now is an adapter for an
`IndexedSeq` of `TodoItem`s, which has to be explicitly refreshed 
at startup, and every time the underlying sequence changes:

{% highlight scala %}
    val adapter: IndexedSeqAdapter[ TodoItem ] = 
      new IndexedSeqAdapter(
        IndexedSeq.empty,
        itemViewResourceId = android.R.layout.simple_list_item_1 )
{% endhighlight %}

What might be preferable is to have an adapter for an `IndexedSeq`
notifier, which registers _itself_ to get the notifications.  As you've
guessed, we've got one:

{% highlight scala %}
  onCreate {
    ...
    val adapter: IndexedSeqSourceAdapter[ TodoItem ] = 
      new IndexedSeqSourceAdapter(
        this, TodoItems.records,
        itemViewResourceId = android.R.layout.simple_list_item_1 )
    ...
  }
{% endhighlight %}

This `IndexedSeqSourceAdapter` is a derivative of `IndexedSeqAdapter`,
whose only added functionality is to arrange the updates (and to ask
the `activity` to shut them down at its `onDestroy`, which is why the
adapter takes the `Activity` as the first argument to its
constructor).

<div class="qanote">
 <a class="question">Why <tt>TodoItems.records</tt>?</a>
 <div class="answer">
   <p>
      You're right.  That's silly.  I'll fix it.
   </p>
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

### Handling user actions with less boilerplate

TBD