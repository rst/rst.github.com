---
layout: tut_section
title: Using the ORM in a simple UI
summary: Database usage, using familiar APIs
---

We have our first-cut database.  We're now going to discuss the user
interface.  For the first cut, as discussed above, that's going to be
a single screen, controlled by a single android `Activity`, which
shows the current contents of the `TodoItems` table, as it currently
exists in the database.  

Our user interface is as specified in [this layout](http://...).  Most
of it is a `ListView` which contains the current todo list items;
initially, tapping one of these will delete it.  At the top, there is
an `EditText` to add the text for new items, and a button which adds
an item whose `description` is the current content of the `EditText`.

So, what does the code look like?

### Initialization

We'll begin with the basic mechanics of setting up an `Activity`:

{% highlight scala %}
class TodoItemsActivity extends Activity {
  override def onCreate(savedInstanceState: Bundle) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.todo_items)
  }
}
{% endhighlight %}

The first thing we have to do is arrange for the database to be opened
and closed.  The most direct way of doing this is to add a line to
`onCreate`, and properly clean up in a new `onDestroy` method:

{% highlight scala %}
  override def onCreate(savedInstanceState: Bundle) {
    ...
    TodoDb.openInContext( this )
  }

  override def onDestroy = {
    super.onDestroy
    TodoDb.close
  }
{% endhighlight %}

That adds five lines of code; there are less chatty ways to do it, in
which Positronic Net takes care of a little more of the bookkeeping,
but I'll be going through the more explicit ways first, to try to give
a feel for what's going on behind the scenes, and to dispel any air of
magic around the really short versions.

Going on, the next, basic task is initialize our `ListView` to contain
the current set of items when the `Activity` starts (or restarts).
That information comes out of the ORM as a Scala `IndexedSeq` of our
`TodoItem` objects.  As per usual, for Android, this is done by giving
the `ListView` an `Adapter`, which tells it how to fill in the
sequence from our data source, and then hooking the data source itself
up to the adapter.  

Android supplies a convenience `ArrayAdapter`, but that won't handle
the Scala `IndexedSeq` type.  Positronic Net supplies the obvious
variant:

{% highlight scala %}
  override def onCreate(savedInstanceState: Bundle) {
    ...
    val adapter: IndexedSeqAdapter[ TodoItem ] = 
      new IndexedSeqAdapter(
        IndexedSeq.empty,
        itemViewResourceId = android.R.layout.simple_list_item_1 )

    val listView = findViewById( R.id.listItemsView ).asInstanceOf[ ListView ]
    listView.setAdapter( adapter )
    ...
  }
{% endhighlight %}

(Note the `asInstanceOf`, Scala's somewhat awkward notation for type
casts, which is deliberately awkward to encourage people to try to
find better alternatives.  We'll see one in a bit.)

Initializing the `ListView` with the current contents of the list is
only one line, but it's a line that requires some amount of
explanation:

{% highlight scala %}
  override def onCreate(savedInstanceState: Bundle) {
    ...
    TodoItems ! Fetch{ items => adapter.resetSeq( items ) }
    ...
  }
{% endhighlight %}

What the heck is going on here?  Let's unpack this from the inside out.
What's innermost is this bit:

{% highlight scala %}
    { items => adapter.resetSeq( items ) }
{% endhighlight %}

That's a function which takes a list of items, and tells the adapter
to use that set of items as its current sequence.  Now for the odd
`TodoItems ! Fetch{...}` notation.  What's up with that?

Well, on Android, it's considered good practice for an `Activity` to
avoid doing I/O operations, other than manipulating its own UI
components directly, on its own "main application thread".
Accordingly, Positronic Net does database queries and updates on a
background thread associated with the `Database` (which is started
when the first `Activity` opens it, and shut down when the last one
does a `close`).  The use of `!` is a notational convention to
indicate that the request that follows (`Fetch`, `Save`, `Delete`) is
to be passed off to the "Database thread" for execution.

(There's one more piece of black magic going on here.  It's a bad
idea, leading to laggy UIs and ticked-off users, to do I/O on the UI
thread.  But it's an even _worse_ idea, leading to jumbled displays
and "Force Close"d applications, to manipulate UI components, such as
`ListView`s and their `Adapter`s, on any _other_ thread, as the
Android UI is not thread-safe.  Positronic Net needs to avoid that; it
does so by using an `android.os.Handler` to `post` the body of a
`Fetch` back to the UI thread, for execution there, when the results
are ready.)

Whew.

### Manipulating items.

Having done all that, the code for deleting items on the list should
be straightforward.  Remember, in this quickie version, we want a tap
on an item to delete it.  So:

{% highlight scala %}
  override def onCreate(savedInstanceState: Bundle) {
    ...
    listView.setOnItemClickListener {
      new OnItemClickListener {
        override def onItemClick( parent: AdapterView[_], view: View, 
                                  posn: Int, id: Long ) = {
          TodoItems ! Delete( adapter.getItem( posn ) )
          TodoItems ! Fetch{ adapter.resetSeq( _ ) }
        }
      }
    }
    ...
  }
{% endhighlight %}

So, when the user does a tap, we post a request to `Delete` the item
to the database thread (via the `TodoItems` record manager), and then
re-fetch the revised list and send the result to the `ListView`'s
adapter.

About the only thing remarkable here is to note that the database
thread executes queries in the order they're posted to it.  So, even
though the database operations will be happening on a separate thread,
the revision will show the result of the delete.

With the code to an item, we're finished:

{% highlight scala %}
  override def onCreate(savedInstanceState: Bundle) {
    ...
    val button   = findViewById( R.id.addButton ).asInstanceOf[ Button ]
    button.setOnClickListener{
      new OnClickListener {
        override def onClick(v: View) = {
          val textView = findViewById( R.id.newItemText ).asInstanceOf[TextView]
          val text = textView.getText.toString.trim
          if (text != "") {
            TodoItems ! Save( new TodoItem( text ))
            TodoItems ! Fetch{ adapter.resetSeq( _ ) }
          }
          textView.setText( "" )
        }
      }
    }
  }
{% endhighlight %}

