---
layout: post
title: Better notifiers and futures
summary: Monads and actors and stuff, oh my!
---
## What might change, and what won't.

This note describes some reframing I'm thinking about in terms of the
Positronic Net APIs for concurrency support and messaging.  I think,
going forward, this would make for neater design all around.

However, I would plan to <em>keep the existing APIs around</em>, at 
least for enough time to let people transition (which should involve
only one-liner changes in any event).  That said, if there is cause
for breaking changes, earlier is usually better...

## We're we've been --- adding futures

One of the things I've tried to do with Positronic Net is use the
concurrency idioms that come with Akka, without necessarily dragging
all of Akka in.  (This is in part due to wanting to have the details
of the messaging implementation deal more directly to Android's native
notions of "Handler threads", and the requirements of the Activity and
Service lifecycles, and in part to simply avoid unnecessary bulk.
(The size of even the Scala standard library is an issue; that by
itself made me think that cut-down versions of some of the other stuff
might be a better fit.)

Initially, this meant that only certain library-provided entities could
act as message receivers, and callbacks were embedded directly in the
messages:

{% highlight scala %}
  TodoItems ! Fetch{ items => ... }
{% endhighlight %}

In this case, the behavior was as if the `Fetch` action constructor
would, in effect, remember what Android `HandlerThread` it was called
from, and arrange for the callback to be invoked with the result (when
available) on the same thread.

(In fact, the actual mechanics were more of a mess, due to my
first-cut API for running queries immediately on the current thread.
And there are use cases that call for that.  Consider, for instance, a
`Service`.  If its `onStartCommand` posts code for later execution, as
in the above examples, then `onStartCommand` itself would return
immediately --- at which point, the Android framework would consider
the command-handling complete, and the process containing the Service
would be eligible for summary termination, whether or not the background
thread had done the work.  The simplest approach in these cases is to
have _some_ API which keeps everything on one thread, though we'll see
alternatives in a minute.)

Where this broke down was in the Contacts-manager sample app.  (As 
expected.  One of the reasons I did a contacts manager as a sample app
was that it would force me into handling an interesting set of corner
cases.  This was one of them.)  

Among other motiviations: let's say that you have a bunch of
`RawContact` objects, and that you'd like to do something with the
associated `ContactData` of all of them.  If you know that you have
exactly two of them, you can do something like:

{% highlight scala %}
  rawContact1.data ! Fetch{ data1 =>
    rawContact2.data ! Fetch{ data2 =>
      // ... do something with data1 and data2
    }
  }
{% endhighlight %}

But if you have a vector of `RawContact` objects, and would like a
vector of `IndexedSeq[Data]` to be processed when they're all available,
without knowing the size of the vector in advance... things are very
awkward.

The solution I chose is to borrow some machinery from Akka:  create
an alternate series of query operators that return `Future` objects,
and implement combinators on the `Future`s.  (Which can be done in
very little code --- if you look at the Akka source, the combinators
are the tip of an iceberg which goes much, much deeper, and it's not
yet obvious that the deep parts are useful on Android.)

Thus, to retrieve all the `ContactData`, pair each up with the `RawContact`
that it came from (when available), and do something with the result:

{% highlight scala %}
    val dataQueries: Seq[Future[IndexedSeq[ContactData]]] =
      rawContacts.map { rawContact => (rawContact.data ? Query) }

    Future.sequence( dataQueries ).onSuccess { data => 
      val myState = new AggregateContactEditState( rawContacts.zip( data ))
      ...
    }
{% endhighlight %}

where `Future.sequence` is the operator that takes a collection of 
`Future`s, and returns a `Future` of a collection.  (Which, in turn,
is easy to implement in terms of simpler combinators on `Future`.)

## Where we could go --- better notifiers.

So, making this change to notation for simple queries has some
benefits.  (Or rather, supporting this alternative --- the old API
still works, at least for now.)  It would be nice to secure similar
benefits for some of the other messages-with-callbacks that Positronic
Net currently supports.  Specifically, there is the notification-style
query:

{% highlight scala %}
  TodoItems ! AddWatcher( key ){ items => ... }
  TodoItems ! AddWatcherAndFetch( key ){ items => ... }
{% endhighlight %}

Both of these treat `TodoItems` as a sourceof a stream of sequences
of items, which invoke the handler.  The `key` is something that can
be used to control the lifetime of the stream, i.e., tell the event
source --- `TodoItems` in this case --- when this particular watcher
no longer cares about the events:

{% highlight scala %}
  TodoItems ! StopWatcher( key )
{% endhighlight %}

Applying a similar transformation would yield:

{% highlight scala %}
  val watcher: FutureStream[ IndexedSeq[ TodoItem ]] = TodoItems ?? Values

  watcher.withValue{ values => ... }
{% endhighlight %}

Here we choose `??` as a message-sending variant which yields a
`FutureStream` instead of just a `Future`.  The idea is then that
each time the `TodoItems` table is updated, the `withValue` handlers
are called with the new sequence of items as an argument.

The tricky part of this is how to arrange transparent lifetime control
--- another part of the API that is less elegant than might be
desired.  The current way that that happens is that our
`ActivityHelper`s have a method that effectively does this (via a yet
older version of the API, internally):

{% highlight scala %}
  def manageListener[T]( listener: AnyRef, source: Notifier[T])( handler: T => Unit ) = {
    source ! AddWatcherAndFetch( listener ){ handler }
    this.onDestroy{ source.stopNotifier( listener ) }
  }
{% endhighlight %}

Which hides the concurrency operators from the invoking source code,
which is bad --- we really want to keep the existence of concurrency
up front.

Two possible options are:

{% highlight scala %}
  val lifetime: Duration = myActivity.whileActive     # new helper method
  val myStream = TodoItems ?? (Values( during = lifetime ))
{% endhighlight %}

and

{% highlight scala %}
  val lifetime: Duration = myActivity.whileActive
  val myStream = (TodoItems ?? Values).during( lifetime )
{% endhighlight %}

where, either way, `lifetime` is an object which the `FutureStream`s
to coordinate when they should be active.  (It could even be a 
`FutureStream[ Boolean ]` itself.)

## Other changes

Coupled with this, I'd also like it to be easier to declare an actor-like
entity that could respond to messages (looking at the Akka API for the
actors as one inspiration), though in a way that perhaps couples more
directly to Android's HandlerThreads.

I'm also thinking about improvements to the way the type system interacts
with messaging (adding covariance and contravariance where they arguably
belonged already), though that's likely to be the subject of a different
note.  This one's long enough!
