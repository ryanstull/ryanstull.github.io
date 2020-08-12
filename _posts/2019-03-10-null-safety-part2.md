---
layout: post
title:  "Null-safety Part 2: Working With Null in Scala"
date:   2019-03-10 13:42:24 -0400
categories: blog
tag: scala
---

So how then, given this problem, are we to try to write null-safe code in Scala?  There are two different things we need to do when working with potentially absent values, keep track of them, and perform operations on them.

One approach many languages have come up with a solution using a generic wrapper type, `Optional` in Java, `Option` in Scala and Rust, `Nullable` in C#, or `Maybe` in Haskell.  We'll use Scala for all the following examples.  A simplified definition of the `Option` type would look something like this:

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

Now, everywhere you used to have a `String`, you have an `Option[String]`, so it is apparent, when you're expecting a possibly absent value.  If your codebase and the libraries you use are consistent, and use this strategy throughout, it can work very well for keeping track of absent values.

However, there are some shortcomings to this approach in Scala:

* Since references can still be `null`, this system "rides on top of" the default system for representing absent values (Unlike in Rust, C# and Haskell). This means there's
somewhat of a duplication of effort, and thus two "holes" in this approach. 

{% highlight scala %}
val option: Option[Any] = null //An Option that is actually null

val option: Option[Null] = Some(null) //An Option that contains null
{% endhighlight %}

* This can use more memory, since you have to store an extra reference for each optional value

Though, these shortcomings are generally not a big problem in practice. 

## Operations

While code written with `Option` can be more verbose than regular code, it is still usually pretty straight-forward.

{% highlight scala %}
def getPossiblyNullString(): String = ...

val option: Option[String] = Option(getPossiblyNullString())

option match {
     case Some(_) => //Value is present
     case None => //No value
}
{% endhighlight %}

There are [many methods](https://www.scala-lang.org/api/current/scala/Option.html) available on `Option` for operating on the value stored inside.

Say you wanted to take a possibly null `String`, trim it, keep it if it's not empty, and then change it to upper case.  You could accomplish it like so:

{% highlight scala %}
option.map(_.trim).filter(_.length > 0).map(_.toUpperCase)
{% endhighlight %}

Despite all of these capabilities, there is one very common scenario though, where the optional code feels quite unsatisfactory.

{% highlight scala %}
case class A(b: B)
case class B(c: C)
case class C(string: String)

val a = A(B(C("Hello")))
{% endhighlight %}

Say you wanted to extract the value of `string`. Using `Option` is somewhat clunky:

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