---
layout: post
title:  "Null-safety Part 2: Working With Null"
date:   2019-03-10 13:42:24 -0400
categories: blog
tag: scala
---

So how then, given this problem are we to try to write null-safe code in Scala.

Some languages, noticing this problem, have come up with a solution using a generic wrapper type, `Optional` in Java, `Option` in Scala, `Nullable` in C#, or `Maybe` in Haskell. This type usually has two subtypes, one that represents the presence of a value, which conatains that value, and another that represents the absence of a value.  We'll use Scala for this example:  

A simplified definition of the `Option` type would look something like this:

{% highlight scala %}
sealed trait Option[+A]

case class Some[A](val value: A) extends Option[A]
case object None extends Option[Nothing]

object Option {
  def apply[A](x: A): Option[A] = if (x == null) None else Some(x)
}
{% endhighlight %}

One could use this strategy like so:

{% highlight scala %}
def getPossiblyNullReference(): AnyRef = ...

val opt = Option(getPossiblyNullReference())
{% endhighlight %}

The real `Option` class provides many methods for transforming the value that might be contained by the option.  If we were redo the previous `substring` example with Scala's `Option`, it becomes:

{% highlight scala %}
val h = Option("hello").map(_.substring(2)) // returns Some("llo")

val w = Option(null).map(_.substring(2)) // returns None
{% endhighlight %}

<hr/>

However there are some issues with this approach.  The first is that depending on the implementation, the use of a wrapper object may incur an additional heap allocation, which leads to performance issues.  The other is that we end up with two 'holes' in this safety mechanism;  they are:

{% highlight scala %}
val option: Option[Any] = null

val option: Option[Null] = Some(null)
{% endhighlight %}

Since `Option` is just another object in the type system it can been assigned to `null`, and the `Some` subtype could contain null itself instead of being `None`.

<br/>

***

<br/>
## [Part 3: Introducing ScalaNullSafe]({% post_url 2019-03-10-null-safety-part3 %})