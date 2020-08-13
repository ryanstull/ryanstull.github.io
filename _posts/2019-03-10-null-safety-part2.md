---
layout: post
title:  "Null-safety Part 2: Working With Null in Scala"
date:   2019-03-10 13:42:24 -0400
categories: blog
tag: scala
---

So how then, given this problem, are we to try to write null-safe code in Scala?  NullPointerExceptions (NPEs) occur when we try to access a field, method, or index, of an object that is actually `null`.

{% highlight scala %}
case class A(b: B)
case class B(c: C)
case object C

val a: A = null

a.b.c //Would result in an NPE
{% endhighlight %}

At my work, we ran into this issue when we had to pull data out of highly nested Avro classes, sometimes 7 levels deep.  Essentially, what is missing from Scala here is a [safe navigation operator](https://www.groovy-lang.org/operators.html#_safe_navigation_operator) similar to Groovy or Kotlin.

In order to find the best solution to this problem, I began by enumerating all of the possible ways that I could think of to implement null-safe access.

1. Explicit null safety
   ```scala
   if(a != null){
       val b = a.b
       if(b != null){
           b.c
       } else null
   } else null
   ```

    Pros: Very efficient, just comprised of null-checks
    
    Cons: Poor readability and writability
    
2. Option flatmap

   ```scala
   Option(a)
       .flatMap(a => Option(a.b))
       .flatMap(b => Option(b.c))
   ```
   
   Pros: Better read/writability, but still not great.
       
   Cons: High performance overhead, object allocation + virtual method calls per level of drilldown

3. For Loop
   ```scala
   for {
     aOpt <- Option(a)
     b <- Option(aOpt.b)
     c <- Option(b.c)
   } yield c
   ```
   
   Similar to the approach #2, but slightly slower and worse readability.
   
4. Null-safe navigator
   ```scala
   implicit class nullSafe[A](val a: A) extends AnyVal {
     def ?[B <: AnyRef](f: A => B): B = if (a == null) null else f(a)
   }
   
   a.?(_.b).?(_.c)
   ```

    Pros: Pretty readable syntax.  No object allocation.
    
    Cons: Syntax still not perfect.  1 function call per level of drilldown

5. Try Catch NPE
   ```scala
   try { a.b.c } catch {
     case _: NullPointerException => null
   }
   ```

    Pros: Syntax could be very nice if abstracted out to a function
    
    Cons: Harsh performance penalty in case of NPE.  Could intercept other NPE

Here are some benchmarks of the different approaches.

{% include image.html url="/assets/images/posts/throughput 1.png" description="Performance of different null-safe implementations" %}

All of the 'Present' benchmarks are where the value was actually defined and the 'Absent' ones are where one of the intermediate values, was `null`; or in other words, where an NPE would have occurred.

In order to summarize the pros and cons of each approach, let's make a chart of each and evaluate them based on null-safty, read/writability, and efficiency.

|                      	| Null-safe 	| Readable/Writable 	| Efficient 	|
|----------------------	|-----------	|-------------------	|-----------	|
| Normal access        	| ⛔         	| ✅                 	| ✅         	|
| Explicit null-checks 	| ✅         	| ⛔                 	| ✅         	|
| Option flatMap       	| ✅         	| ⛔                 	| ⛔         	|
| For loop flatMap     	| ✅         	| ⚠️                 	| ⛔         	|
| Null-safe navigator  	| ✅         	| ⚠️                 	| ⚠️         	|
| Try-catch NPE        	| ✅         	| ✅                 	| ⚠️         	|

After evaluating all of the options available, I wasn't quite satisfied, so I decided to create a new way via Scala's blackbox macros.

***

<div class="PageNavigation">
  {% if page.previous.url %}
    <a class="prev" href="{{page.previous.url}}">&laquo; {{page.previous.title}}</a>
  {% endif %}
  {% if page.next.url %}
    <a class="next" href="{{page.next.url}}">{{page.next.title}} &raquo;</a>
  {% endif %}
</div>