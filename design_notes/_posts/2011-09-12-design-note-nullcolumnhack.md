---
layout: design_note
title: Dealing with "NullColumnHack"
summary: Dealing gracefully with a SQLite glitch
disqus_id: nullcolumnhack
---
The native SQLite APIs for Android have a glitch:  inserting a completely
empty `ContentValues` doesn't work.  The reason is that `SQLite`
requires at least one column to be specified on an insert.

To get around that, the native `SQLiteDatabase` `insert` helper has an
odd extra argument, `nullColumnHack`.  The idea is that this should be
the name of a nullable column in the table into which you're doing the
`insert`.  So, if you want to `insert` a completely empty
`ContentValues`, you also supply a column name in `nullColumnHack`.
It then behaves as if you'd explicitly set the value of the column
named in `nullColumnHack` to `null`, in the `ContentValues`, thereby
meeting `SQLite`'s requirement that you name at least one column.

(So, why not just do that?  Never mind.)

The problem I'm trying to address here is that the Positronic Net
generic `ContentRepository` framework doesn't allow callers to
supply this odd extra argument on an `insert`, and there's no
obvious place to put it.  Right now, we just supply a `null` as
the value of `nullColumnHack` --- which is always fine, so long
as the argument `ContentValues` isn't empty.  But what if it is? 

Fortunately, to work, it just has to be the name of _some_ nullable
column in the relevant table --- meaning that the _same_ value will
work for any insert into a given table.  Which means that if the
caller can't supply it at the point of the insert (because the API has
no place for it), it could be supplied at database creation:

{% highlight scala %}
object TodoDb extends Database( 
  filename = "todos.sqlite3",
  nullColumnHack = Map( "todo_items" -> "sign_off_date", ...)
 ) 
{
  def schemaUpdates = ...
}
{% endhighlight %}

and we could rig `insert` on a DB-backed `ContentQuery`
to look up the proper value for its table in the map.  As a fringe 
benefit, this lets the programmer write the set of names down just
once, as opposed to once per insert.

(If the overhead for one lookup in a tiny hash per insert got to be
significant, the implementation support for this could be bundled into
a subclass of `Database`.)

As another fringe benefit, writing this down is a quick way to check
the formatting of design notes.