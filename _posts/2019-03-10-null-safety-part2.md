---
layout: post
title:  "Null-safety Part 2: Working With Null in Scala"
date:   2019-03-10 13:42:24 -0400
categories: blog
tag: scala
---

So how then, given this problem, are we to try to write null-safe code in Scala?

There are two different things we need to do when working with potentially absent values, keep track of them, and transform them.

One approach many languages have come up with a solution using a generic wrapper type, `Optional` in Java, `Option` in Scala and Rust, `Nullable` in C#, or `Maybe` in Haskell.  We'll use Scala for all the following examples:  

A simplified definition of the `Option` type would look something like this:

{% highlight scala %}
sealed trait Option[+A]

case class Some[A](val value: A) extends Option[A]
case object None extends Option[Nothing]

object Option {
  def apply[A](x: A): Option[A] = if (x == null) None else Some(x)
}
{% endhighlight %}

Now let's take a look at how it fairs at the two functions it serves

## Keeping Track

One could use this strategy like so:

{% highlight scala %}
def getPossiblyNullString(): String = ...

val opt: Option[String] = Option(getPossiblyNullString())

opt match {
    case Some(_) => //Value is present
    case None => //No value
}
{% endhighlight %}

If your codebase and the libraries you use are consistent, and use this strategy throughout, it can work very well for keeping track of absent values.

//TODO More examples

However, there are some shortcomings to this approach in Scala:

* Since references can still be `null`, this system "rides on top of" the default system for representing absent values (Unlike in Rust, C# and Haskell). This means there's
somewhat of a duplication of effort, and thus two "holes" in this approach. 

{% highlight scala %}
val option: Option[Any] = null //An Option that is actually null

val option: Option[Null] = Some(null) //An Option that contains null
{% endhighlight %}

* This can use more memory, since you have to store an extra reference for each optional value

Despite these, `Option` works reasonably well.

## Transforming

While code written with `Option` can be more verbose than regular code, though it is still usually pretty straight-forward.

{% highlight scala %}
def getPossiblyNullString(): String = ...

val opt: Option[String] = Option(getPossiblyNullString())

opt match {
     case Some(_) => //Value is present
     case None => //No value
}
{% endhighlight %}

# Drilldown
For example, say you had the following scenario

{% highlight scala %}
case class A(b: B)
case class B(c: C)
case class C(string: String)

val a = A(B(C("Hello")))
{% endhighlight %}

and you wanted to extract the value of `string`. Using `Option` is somewhat clunky:

{% highlight scala %}
Option(a)
    .flatMap(a => Option(a.b))
    .flatMap(b => Option(b.c))
    .flatMap(c => Option(c.string))
{% endhighlight %}

***

<div class="PageNavigation">
  {% if page.previous.url %}
    <a class="prev" href="{{page.previous.url}}">&laquo; {{page.previous.title}}</a>
  {% endif %}
  {% if page.next.url %}
    <a class="next" href="{{page.next.url}}">{{page.next.title}} &raquo;</a>
  {% endif %}
</div>