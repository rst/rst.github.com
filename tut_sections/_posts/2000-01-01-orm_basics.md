---
layout: tut_section
title: Database basics
summary: How to set up data storage for a simple list in less than two dozen lines of code
---
## Data representation.

As is conventional in Android, we're going to ultimately be storing
our to-do list in a table in a SQLite database.  Positronic Net's
facilities for doing that are heavily inspired by the Ruby on Rails
web development framework, though, as we'll see, it has some twists
of its own in order to accommodate its requirements (most notably,
that database I/O and UI management happen on different threads).

### Setting things up

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
contents.  

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

### Using it

So, how do we use this?  Well, so long as we aren't planning on
persisting our `TodoItem`s into the database, nothing has changed;
we're free to create and copy them exactly as we normally would.

What we've added is the ability to ask the `RecordManager` to store
`TodoItem`s into the `Database`, and to retrieve them (so long as the
`Database` has been opened; we'll cover that in detail in the next
section).  

<div class="qanote">
  <a class="question">How does the <tt>RecordManager</tt> know which
                      field goes with which column?</a>
  <div class="answer">
   <p>
    The <tt>RecordManager</tt> can figure out on its own which fields should be
    matched to which database columns based on naming conventions --- the
    operative convention being that a field with a <tt>camelCased</tt> name is
    mapped to the column whose name is the same thing with underscores
    (thus <tt>camel_cased</tt>, if anyone was actually going to name a field
    that), with the special case that a field named `id` maps to a
    column named <tt>_id</tt>.
   </p>

   <p>
    The <tt>RecordManager</tt> knows how to map fields of all numeric types
    (<tt>Int</tt>, <tt>Long</tt>, <tt>Float</tt>, etc.), <tt>String</tt>s, and <tt>Boolean</tt>s.  Since
    SQLite lacks native Boolean support, a <tt>Boolean</tt> field is expected
    to be mapped to an <tt>Int</tt>-valued column, with values 0 or 1 for 
    <tt>false</tt> or <tt>true</tt>, respectively.
   </p>
  </div>
</div>

<div class="qanote">
  <a class="question">What if there isn't a matching field or column?</a>
  <div class="answer">

   <p>Fields with no obviously matching column (or columns with no obviously
   matching field) are not mapped by default, though in the declaration of
   the <tt>RecordManager</tt> instance, you can request nonstandard mappings
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

Go to [Usage examples](/tut_sections/1999/01/01/simple_activity.html)