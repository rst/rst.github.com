---
layout: default
title: Positronic Net Tutorial
summary: Getting started with Positronic Net.
---

# Positronic Net Tutorial

Positronic Net is a library that aims to make it more convenient to
write Android applications, by taking care of background plumbing
tasks, and letting you concentrate less on the requirements of the
framework, and more on the problem that you're trying to solve.  To
help with that, it's for applications written in Scala, a JVM-based
language that adds type inference and other facilities from functional
programming languages.  (For the moment, we're assuming familiarity
with both Scala and the basics of the Android API; hopefully, that'll
change at some point.)

This tutorial tries to get you acquainted with the basic concepts of
the library, with a simple todo-list application.  (The initial UI is
that users fill in a text field and push an "Add" button to add an
item to the list, and tap on the item to remove it.  Which is pretty
crude; we'll do better later.)

This shouldn't take a whole lot of code, and it doesn't.  The complete
description of the database and items is twenty-one lines.  The first
version of the user interface we'll see is fifty-four lines; we'll
then use various Positronic Net shorthands to cut that in half, by
eliminating boilerplate and letting the library keep track of the
bookkeeping.

We start off with the data storage, since that's an area where
Positronic Net seriously cuts down on the line count, and then show
how the shorthand helpers introduced by Positronic Net can also make
pure UI code easier to write and easier to read.

Complete Scala source code for this example can be found, in one file,
[here](http://github.com/rst/XXX-to-fix), along with its accompanying
[resource definitions](http://github.com/rst/XXX-to-fix).  As you can
see, there isn't much of it at all --- in a way, that's the point.

## Data representation.

As is conventional in Android, we're going to ultimately be storing
our to-do list in a table in a SQLite database.  Positronic Net's
facilities for doing that are heavily inspired by the Ruby on Rails
web development framework, though, as we'll see, it has some twists
of its own in order to accommodate its requirements (most notably,
that database I/O and UI management happen on different threads).

But we'll start here with the description of an item.  There are all
sorts of things you can imagine having here --- an `is_done` status to
indicate that the task has been accomplished, a `todo_list_id` if we
have multiple lists, and so forth --- and we'll see those two in
subsequent installments.  (Though, as of this writing, the text isn't
there yet, the code is; see `sample/todo_app` in the library source.)
But the absolutely bare minimum of a todo-list item record in the
database would be a `String` with the description of the task, and a
`Long` (to match SQLite conventions) for a database row ID.  Thus:

{% highlight scala %}
case class TodoItem( description: String, id: Long )
{
  def setDescription( s: String ) = this.copy( description = s )
  override def toString = this.description
}
{% endhighlight %}

Here, the `case class` bit is a Scala shorthand for declaring a class
with a particular set of extra features --- most notably the typed
fields, and the `copy` method that you can see used here.  (It also
allows class instances to be used with a pattern-matching facility
which we won't be using here, which gives rise to the name.)   The
objects are also immutable --- if you want to change them, you make a
copy with new values.

<div class="qanote">
 <a class="question">Do they really have to be immutable?</a>
 <div class="answer">

   <p>Actually no; that's a convention that I'm following, but if
   you declare ORM-managed records --- using the machinery described
   below --- that have mutable `var` fields that map to database
   columns really should work.</p>

   <p>The main reason I'm doing things this way is that Positronic Net
   does database manipulation in a background thread.  That raises
   the possibility of awkward race conditions where the UI is modifying
   an object which has already been passed off to be saved on a background
   thread.  Treating the objects as immutable makes these race conditions
   impossible.  It does require some extra consing to make the copies,
   but these objects are quite small --- and you can use mutable records,
   if you find in a particular application that this is an issue.</p>

   <p>The same argument would apply with stronger force, of course, if we
   had a tablet app with multiple `Fragment`s presenting different views
   of the same data.</p>

 </div>
</div>

We also have to declare how things are stored in the database itself.
The quickest way to make that happen, with Positronic Net, is as follows:

{% highlight scala %}
object TodoDb extends Database( filename = "todos.sqlite3" ) 
{
  def schemaUpdates =
    List(""" create table todo_items (
               _id integer primary key,
               description string
             )
         """)
}
{% endhighlight %}

This does a few things --- it declares the database as a singleton
object, `TodoDb`, gives the filename for it, and specifies the SQL
used to create the tables that will be used to actually store the
contents.  (We'll see how user interface code deals with the singleton
object in a bit.)

Note that the row ID column is declared as `_id integer primary key`;
this follows Android conventions.

<div class="qanote">
 <a class="question">How does this relate to the standard Android
   `SQLiteOpenHelper` machinery for handling `onCreate` and `onUpdate`?</a>
 <div class="answer">

  <p>It's actually using that machinery under the covers, with
  behavior determined by the supplied list of <tt>schemaUpdates</tt>.
  Specifically, the <tt>version</tt> is, by convention, the length
  of the supplied list of updates; <tt>onCreate</tt> runs them all,
  and <tt>onUpdate</tt> runs the ones between the previous stored
  version at the current end of the list.  So, adding new
  SQL declarations at the end of the list will cause them to
  be run <tt>onUpgrade</tt> when the corresponding version of the app
  is installed.</p>

 </div>
</div>

Having done that, we need to arrange for records to be translated to
and from rows in the database.  In Positronic Net, that's done by a
singleton object of class `RecordManager`, which is tied to the
`TodoItem` class as follows:

{% highlight scala %}
case class TodoItem( description: String = null, 
                     id: Long            = ManagedRecord.unsavedId )
  extends ManagedRecord( TodoItems )
{
  def setDescription( s: String ) = this.copy( description = s )
  override def toString = this.description
}

object TodoItem extends RecordManager[ TodoItem ]( TodoDb( "todo_items" ))
{% endhighlight %}

A few things have been added here:  some obvious, one subtle.

To first state the obvious, we've now made `TodoItem` a subclass of
the library `ManagedRecord` class, and associated it with a
record manager, which I've also named `TodoItem`.  (In Scala terms,
this makes the `RecordManager` the "companion object" of the class,
meaning that the `RecordManager` and its `ManagedRecord`s can
access each others' private data, should that prove useful.)

The subtlety:  in order to pull records out of the database,
the `RecordManager` has to be able to create them, which means (as
is typical for ORMs) that there needs to be a zero-argument
constructor.  However, if there's a constructor with defaults for
all arguments, that's good enough --- so we give them all defaults.
(The `id` attribute defaults to a value which no actual stored 
record will ever have --- in fact, -1.)

The `RecordManager` figures out on its own which fields should be
matched to which database columns based on naming conventions --- the
operative convention being that a field with a `camelCased` name is
mapped to the column whose name is the same thing with underscores
(thus `camel_cased`, if anyone was actually going to name a field
that), with the special case that a field named `id` maps to a
column named `_id`.

The `RecordManager` knows how to map fields of all numeric types
(`Int`, `Long`, `Float`, etc.), `String`s, and `Boolean`s.  Since
SQLite lacks native Boolean support, a `Boolean` field is expected
to be mapped to an `Int`-valued column, with values 0 or 1 for 
`false` or `true`, respectively.

<div class="qanote">
  <a class="question">What if there isn't a matching field or column?</a>
  <div class="answer">

   <p>Fields with no obviously matching column (or columns with no obviously
   matching field) are not mapped by default, though in the declaration of
   the `RecordManager` instance, you can request nonstandard mappings
   explicitly.</p>
   
   <p>There's an example of that in the sample
   <a href="https://github.com/rst/UmbrellaToday"><tt>UmbrellaToday</tt>
   application</a>, where the application's natural representation of the
   <tt>alertAt</tt> time on a <tt>WeatherAlert</tt> is a Java
   <tt>Calendar</tt> object.  This is stored as a string in the
   database; the <tt>RecordManager</tt> is told to map that to an
   attribute named <tt>rawAlertAt</tt>, and the <tt>WeatherAlert</tt>
   objects do the conversions, transparently to the code that needs to
   deal with them.</p>
   
  </div> 
</div>
   
<div class="qanote">
 <a class="question">Why not just deduce the table definition from the record
                     declarations</a>
 <div class="answer">

   <p>A number of other ORM systems do exactly that, and it works
   moderately well for them --- though it would require declaring
   which fields are to be mapped to columns.</p>
   
   <p>What makes me nervous about this sort of arrangement is that it
   gets awkward in situations where a new column is, say, initialized
   based on the contents of data that were already entered.</p>

 </div>
</div>

## Using it

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

Complete code is [here](https://github.com/rst/tuttodo/branches/phase1XXX].

## Tightening things up.

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
