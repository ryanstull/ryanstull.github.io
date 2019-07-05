---
layout: post
title:  "Null-safety Part 3: Announcing ScalaNullSafe!"
date:   2019-03-10 13:42:24 -0400
categories: blog
tag: scala
---

Let's take a look at all of the different ways we can do this:

|                      	| Null-safe 	| Readable/Writable 	| Efficient 	|
|----------------------	|-----------	|-------------------	|-----------	|
| Normal access        	| ⛔         	| ✅                 	| ✅         	|
| Explicit null-checks 	| ✅         	| ⛔                 	| ✅         	|
| Option flatMap       	| ✅         	| ⛔                 	| ⛔         	|
| For loop flatMap     	| ✅         	| ⚠️                 	| ⛔         	|
| Null-safe navigator  	| ✅         	| ⚠️                 	| ⚠️         	|
| Try-catch NPE        	| ✅         	| ✅                 	| ⚠️         	|

So unfortunately there's at least one issue with all of these different approaches.  The best one in terms of performance is the explicit null-checks.  Now the only problem with that is that it's very hard to read and write.  But what if we could make it easy to do so?

## Enter the macro

This is where the idea for the macro-based approach comes in. It allows us to create, what is essentially, a `source code` -> `source code` level transformation at compile time; or in layman's terms, a code re-writing tool.

With this power, we can specify the property we're trying to access, and have the macro rewrite it for us, as the fully explicitly null-checked version.

An example:

{% highlight scala %}
case class A(b: B)
case class B(c: C)
object C

val a: A = null

?(a.b.c) //returns null

//^Gets transformed into:
//if(a != null){
//  val b = a.b
//  if(b != null){
//    b.c
//  } else null
//} else null

val a2 = A(B(C))

?(a2.b.c) //returns C
{% endhighlight %}

There's also two other variants, `opt` and `notnull`, which work as follows:

{% highlight scala %}

opt(a.b.c) //returns None

//^Gets transformed into:
//if(a != null){
//  val b = a.b
//  if(b != null){
//    Some(b.c)
//  } else None
//} else None

val a2 = A(B(C))

opt(a2.b.c) //returns Some(C)
{% endhighlight %}