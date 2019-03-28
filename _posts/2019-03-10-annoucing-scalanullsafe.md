---
layout: post
title:  "Announcing ScalaNullSafe!"
date:   2019-03-10 13:42:24 -0400
categories: blog
tag: scala
---

NullPointerException.

What Java/Scala programmer hasn't had to deal with this issue that never seems to go away fully.


WHY?
The issue is that Java and Scala's type systems don't differentiate between nullable and non-nullable references.

This leads to the situation where one can call methods or access properties of something that is actually null.

> Why did I make the NPE?  
-Tony Hoare

Let's a quick look at Scala's type system to see this problem visually

<SCALA TYPE SYSTEM PICTURE>

So we have to ensure our code is null-safe if we want to avoid the dreaded NPE in Scala.

Let's take a look at all of the different ways we can do this.

There's:

CHART

So unfortunately there's at least one issue with all of these different approaches.  The best one
in terms of performance is the explicit null-checks.  Now the only problem with that is that it's very hard to read
and write.  But what if we could make it easy to do so?

This is where the idea for the macro-based approach comes in.

It allows us to, what is essentially, a source code -> source code level transformation.

Here it is 

{% highlight scala %}
case class A(b: B)
case class B(c: C)
class C

val a: A = null

?(a.b.c)
 
if(a != null){
  val b = a.b
  if(b != null){
    b.c
  } else null
} else null
{% endhighlight %}

asdasd