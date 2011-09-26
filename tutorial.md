---
layout: default
title: Positronic Net Tutorial
summary: Getting started with Positronic Net.
---

# Positronic Net Tutorial

<img src="images/pnettodo.png" style="float:right">

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
the library, with the simple todo-list application you can see at
right.  (The initial UI is
that users fill in a text field and push an "Add" button to add an
item to the list, and tap on the item to remove it.  Which is pretty
crude; we'll do better later.)

The payoff is [here](https://github.com/rst/positronic_tutorial_todo/blob/phase2/src/main/scala/Todo.scala) --- an application in which the entire UI
code is reduced to this:

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

For this first version, we'll start by presenting the backing database
support which is necessary to support this (which is even shorter than
the UI code --- 21 lines).  Then we'll show a first version of the UI
code, which explicitly does a lot of the plumbing chores which are
oddly absent from the sample above:  updating the `ListView` when the
list of items changes, closing the database when the `Activity` shuts
down and so forth.  Then we'll go though and see how the library can
take care of that plumbing for us.

Broken down into individual sections, that's as follows, if you'd
like to skip ahead --- or, just start at the beginning:

<div id="home">
  <ul class="posts">
    {% for post in site.categories.tut_sections %}
      <li>
        <a href="{{ post.url }}">{{ post.title }}</a>
        <p>{{ post.summary }}</p>
      </li>
    {% endfor %}
  </ul>

</div>

