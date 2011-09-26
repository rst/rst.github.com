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
    ...
    listView.setOnItemClickListener {
      new OnItemClickListener {
        override def onItemClick( parent: AdapterView[_], view: View, 
                                  posn: Int, id: Long ) = {
          TodoItems ! Delete( adapter.getItem( posn ) )
          // Fetch is no longer needed here.
        }
      }
    }
    ...
    button.setOnClickListener{
      // Fetch is no longer needed here either...
    }
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
`StopWatcher` is called (that's the `AddWatcher` part).  As a result,
the explicit `Fetch` operations after changes are no longer needed.

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
        this, TodoItems,
        itemViewResourceId = android.R.layout.simple_list_item_1 )
    ...
  }
{% endhighlight %}

This `IndexedSeqSourceAdapter` is a derivative of `IndexedSeqAdapter`,
whose only added functionality is to arrange the updates (and to ask
the `activity` to shut them down at its `onDestroy`, which is why the
adapter takes the `Activity` as the first argument to its
constructor).

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

Another thing that's awkward, looking at the activity code that we've
got so far, is declaring handlers for various user actions.  To show
what we've got so far, here's deletion:

{% highlight scala %}
    listView.setOnItemClickListener {
      new OnItemClickListener {
        override def onItemClick( parent: AdapterView[_], view: View, 
                                  posn: Int, id: Long ) = {
          TodoItems ! Delete( adapter.getItem( posn ) )
        }
      }
    }
{% endhighlight %}

(That's the code as in the last section, without the `Fetch` of the
revised list, which the adapter is now doing on its own.)

Now, the last section mentioned in passing that the views in the
[layout](https://github.com/rst/positronic_tutorial_todo/blob/79a79c8dedd529e3c61684c89e723418e1198f3c/src/main/res/layout/todo_items.xml)
aren't the standard android `ListView`, `Button`, and so forth,
but enhanced versions of them.  (Enhanced just by mixing in a
library trait:  the definitions from the library source
are pretty much

{% highlight scala %}
class PositronicButton( context: Context, attrs: AttributeSet = null )
  extends android.widget.Button( context, attrs ) 
  with org.positronicnet.ui.PositronicHandlers

class PositronicListView( context: Context, attrs: AttributeSet = null )
  extends android.widget.ListView( context, attrs ) 
  with org.positronicnet.ui.PositronicHandlers 
  with org.positronicnet.ui.PositronicItemHandlers
{% endhighlight %}

where `PositronicHandlers` and `PositronicItemHandlers` declare a
whole bunch of extra convenience methods.  It's time to put a few
of them to use.

For starters, the anonymous `OnItemClickListener` in the code above is
really just a wrapper around a function definition, which has to be
written that way because Java doesn't allow function definitions to
stand on their own (or to be anonymous).  Scala does, and the
`PositronicItemHandlers` let us take advantage:

{% highlight scala %}
    listView.onItemClick{ (view, posn, id) =>
      TodoItem ! Delete( adapter.getItem( posn ))
    }
{% endhighlight %}

There go three lines of pure boilerplate.  Similarly for the `Button`.
Instead of 

{% highlight scala %}
    button.setOnClickListener{
      new OnClickListener {
        override def onClick(v: View) = {
        ...
      }
    }
{% endhighlight %}

we can write

{% highlight scala %}
    button.onClick {
      ...
    }
{% endhighlight %}

Lastly, there's the question of adding an item on an 'Enter'
keypress in the textfield.  There are two relevant shorthands.
The one that gets access to any key event would look like this:

{% highlight scala %}
    val textView = findViewById( R.id.newItemText ).asInstanceOf[TextView]

    textView.onKey{ (keyCode, keyEvent) => {
      if (keyCode == KeyEvent.KEYCODE_ENTER
          && ev.getAction == KeyEvent.ACTION_DOWN) 
      {
        addItem
        return true
      }
      return false
    }
{% endhighlight %}

However, there's a particularly terse shorthand for the common case of
declaring that you always want an `ACTION_DOWN` keypress of a
particular key in a particular view to be handled in a particular way:

{% highlight scala %}
    textView.onKey( KeyEvent.KEYCODE_ENTER ){ addItem }
{% endhighlight %}

<div class="qanote">
 <a class="question">Why not use implicit conversions?</a>
 <div class="answer">
   <p>
     People who already know Scala will have spotted that in some of these
     cases, we didn't really <em>need</em> to define traits and
     mix them into the widgets; we could instead have defined an
     "implicit conversion" from the widget to a wrapper class
     which supports the extra methods.
   </p>
   <p>
     In particular, it would be possible for the
     `onClick` and `onListItemClick` methods we're using here
     can be defined in terms of the standard versions --- by
     wrapping their arguments in instances of the Java
     `Listener` classes, and feeding those to the standard
     API.  And that even goes for the simple version of the
     `onKey` handler.
   </p>
   <p>
     Things get more awkward, though, when we're trying to define
     extensions that need to refer to state of their own --- for
     instance, adding extra dispatch state, as in the per-keycode
     version of the `onClick` handler, which has to deal with the
     case of declaring handlers for, say, all of the arrow kepresses.  
   </p>
   <p>
     So, we could have
     two sets of extensions:  one available through implicit
     conversions, and another, those requiring state, available 
     only through mixin traits.  Or we could have one set, with
     one way to get at it.  For now, we're trying the latter
     approach.
   </p>
 </div>
</div>

### Button is a button is a button...

Lastly, there's the awkward Scala cast syntax mentioned in the previous
section:

{% highlight scala %}
    val button = findViewById( R.id.addButton ).asInstanceOf[ Button ]
{% endhighlight %}

I mentioned there that there's a better way.  It's time to say what it is.

One of the fringe benefits that you get with the SBT-Android plugin
that we're using is a set of `TypedResource`s, generated from the
layouts in a process analogous to the standard Android build tooling
that generates `R.java`.  The file can be found a few levels deep in a
`src_managed` subdirectory once you've finished a build, and looks
something like this:

{% highlight scala %}
package org.positronicnet.tutorial.todo
import android.app.Activity
import android.view.View

case class TypedResource[T](id: Int)
object TR {
  val newItemText = 
    TypedResource[org.positronicnet.ui.PositronicEditText](R.id.newItemText)
  val addButton = 
    TypedResource[org.positronicnet.ui.PositronicButton](R.id.addButton)
  val listItemsView = 
    TypedResource[org.positronicnet.ui.PositronicListView](R.id.listItemsView)
}
...
{% endhighlight %}

That defines a `TypedResource` type, with a generic type parameter,
and a bunch of constants of that type.  So, for instance, `TR.addButton`
is defined as a `TypedResource[PositronicButton]`, and `TR.addButton.id`
is the same as `R.id.addButton`.  With that, we can define a utility trait:

{% highlight scala %}
trait ViewHolder {
  def findViewById( id: Int ): View
  def findView[T]( t: TypedResource[T] ) = findViewById( t.id ).asInstanceOf[T]
}
{% endhighlight %}

and mix it into our activity --- giving it a `findView` method which
eliminates the casts, without sacrificing type safety:

{% highlight scala %}
class TodoItemsActivity 
  extends Activity with PositronicActivityHelpers with ViewHolder
{
  onCreate {
    ...
    findView( TR.listItemsView ).setAdapter( adapter )

    findView( TR.listItemsView ).onItemClick{ (view, posn, id) =>
      TodoItem ! Delete( adapter.getItem( posn ))
    }

    findView( TR.addButton ).onClick {
      ...
    }
  }
}
{% endhighlight %}

This `findView` method deduces its type parameter, `T`, from the
`TypedResource` it takes as an argument.  So, the type of
`findView(TR.listItemsView)` is `PositronicListView` because
that's the type parameter of `TR.listItemsView`.  Similarly, the
compiler knows that the result of `findView(TR.addButton)` is a
`PositronicButton`.  If you tried to do a

{% highlight scala %}
    findView( TR.adddButon ).onItemClick{ ... }
{% endhighlight %}

you'd get a type error at compile time, because buttons can't do that.
Not even with Positronic Net enhancements.

### Putting it all together

With all the above changes, we've finally gotten to the version of the
`Activity` source that you've already seen on the tutorial front page:

{% highlight scala %}
class TodoItemsActivity 
  extends Activity with PositronicActivityHelpers with ViewHolder
{
  onCreate {
    setContentView( R.layout.todo_items )
    useAppFacility( TodoDb )

    val adapter: IndexedSeqSourceAdapter[ TodoItem ] = 
      new IndexedSeqSourceAdapter(
        this, TodoItem,
        itemViewResourceId = android.R.layout.simple_list_item_1 )
  
    findView( TR.listItemsView ).setAdapter( adapter )

    findView( TR.listItemsView ).onItemClick{ (view, posn, id) =>
      TodoItem ! Delete( adapter.getItem( posn ))
    }

    findView( TR.addButton ).onClick { addItem }
    findView( TR.newItemText ).onKey( KeyEvent.KEYCODE_ENTER ){ addItem }

    def addItem = {
      val text = findView( TR.newItemText ).getText.toString.trim
      if (text != "") {
        TodoItem ! Save( new TodoItem( text ))
        findView( TR.newItemText ).setText( "" )
      }
    }
  }
}
{% endhighlight %}

It's about half the size of the original version, but compacting
code doesn't always make it clearer.  What's more important is that
the bits we've eliminated were boilerplate.  With that stuff gone,
the structure of the code that's left, and how it relates to what
the user is trying to do, is a lot easier to perceive.  And that's
the real goal here.
