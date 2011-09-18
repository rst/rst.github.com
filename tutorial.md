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
Positronic Net seriously cuts down on the line count, give examples
of usage in an `Activity` that uses only standard APIs, and then show
how the shorthand helpers introduced by Positronic Net can also make
pure UI code easier to write and easier to read.

Discussion is in sections, as follows, if you'd like to skip ahead for
a quick preview of an individual topic:

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

