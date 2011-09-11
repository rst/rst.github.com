---
layout: default
title: Positronic Net Tutorial
summary: Getting started with Positronic Net.
---

# Positronic Net Tutorial

This document assumes that you've got SBT, the android-sbt plugin, and
the library installed, mainly to save me from having to write up 
procedures to do that while I think they aren't stable yet.

## A code sample

We're going to do a to-do list.  We're going to store items in SQLite.
Let's say a few words about that, and then define the schema:

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

<div class="qanote">
  <a class="question">What's up with this?</a>
  <div class="answer"><p>Who knows?</p></div>
</div>

Now that we've done that, we can define a few classes that use the ORM
to map it:

{% highlight scala %}
case class TodoItem( description: String = null, 
                     id: Long            = ManagedRecord.unsavedId 
                   )
  extends ManagedRecord( TodoItems )
{
  def setDescription( s: String ) = this.copy( description = s )
  override def toString = this.description
}

object TodoItems extends RecordManager[ TodoItem ]( TodoDb( "todo_items" ))
{% endhighlight %}

And keep on talking...