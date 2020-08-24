---
layout: post
title:  "Null-safety Part 2: Working With Null in Scala"
date:   2020-08-13 13:42:24 -0400
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

Most Scala programmers would opt to wrap their code in `Option` in order to avoid NPEs, however sometimes you don't have the *option* (pun intended) to do this.  I frequently ran into this situation at my work, where we needed to pull data out of highly nested [Avro](https://en.wikipedia.org/wiki/Apache_Avro) classes, in some cases 6 levels deep.  [Avro schemas](https://avro.apache.org/docs/current/idl.html) get compiled into regular Java objects, that don't use `Option`.  So then what, if your code is not designed around `Option`, is the best approach to extract deeply nested values in a null-safe way?  Essentially, what is missing from Scala here is a [safe navigation operator](https://en.wikipedia.org/wiki/Safe_navigation_operator) similar to Groovy or Kotlin.§

In order to find the best solution to this problem, I began by enumerating all of the possible ways that I could think of to implement null-safe access.

#### 1. Explicit null safety
{% highlight scala %}
if(a != null){
   val b = a.b
   if(b != null){
       b.c
   } else null
} else null
{% endhighlight %}

**Pros:** Very efficient, just comprised of null-checks

**Cons:** Poor readability and writability
    
#### 2. Option flatmap

{% highlight scala %}
Option(a)
   .flatMap(a => Option(a.b))
   .flatMap(b => Option(b.c))
{% endhighlight %}

**Pros:** Better read/writability, but still not great.
   
**Cons:** High performance overhead, object allocation + virtual method calls per level of drilldown

#### 3. For Loop
{% highlight scala %}
for {
 aOpt <- Option(a)
 b <- Option(aOpt.b)
 c <- Option(b.c)
} yield c
{% endhighlight %}

Similar to the approach #2, but slightly slower and worse readability.
   
#### 4. Null-safe navigator extension method
{% highlight scala %}
implicit class nullSafe[A](val a: A) extends AnyVal {
 def ?[B <: AnyRef](f: A => B): B = if (a == null) null else f(a)
}

a.?(_.b).?(_.c)
{% endhighlight %}

**Pros:** Pretty readable syntax.  No object allocation.

**Cons:** Syntax still not perfect.  1 function call per level of drilldown

#### 5. Try Catch NPE
{% highlight scala %}
try { a.b.c } catch {
 case _: NullPointerException => null
}
{% endhighlight %}

**Pros:** Syntax could be very nice if abstracted out to a function

**Cons:** Harsh performance penalty in case of NPE.  Could intercept other NPEs

#### 6. Monocle Lenses

{% highlight scala %}
import monocle.Optional
val aGetB = Optional[A,B]{
    case A(b) if b != null => Some(b)
    case _ => None
}(b => { case A(_) => A(b) })
val bGetC = Optional[B,C]{
    case B(c) if c != null => Some(c)
    case _ => None
}(c => { case B(_) => B(c) })
val aGetC = aGetB composeOptional bGetC

aGetC.getOption(a)
{% endhighlight %}

I didn't really consider using lenses when I began this project, but someone asked me one time, "Why not use lenses, like in Monocle?" So I tried it out, but it didn't succeed in either read/writability, or performance, in this use case.

#### 7. § com.thoughtworks NullSafe DSL

So actually, when I began this project I didn't know that [this library](https://users.scala-lang.org/t/nullsafe-kotlin-groovy-flavored-null-safe-operator-now-in-scala/3244) existed.  Essentially what it does is add in the missing safe navigation operator to Scala, via a compiler plugin.

{% highlight scala %}
import com.thoughtworks.dsl.keywords.NullSafe._

a.?.b.?.c
{% endhighlight %}

It has nice syntax, but it does introduce some performance overhead.

## Comparing Approaches

Here are some benchmarks of the different approaches.

{% include image.html url="/images/posts/nullSafe/throughput 1.png" description="Performance of different null-safe implementations" %}

<details>
  <summary>Data in tabular form</summary>
{% highlight text %}
[info] Benchmark                             Mode  Cnt    Score   Error   Units
[info] Benchmarks.fastButUnsafe             thrpt   20  230.157 ± 0.572  ops/us
[info] Benchmarks.explicitSafeAbsent        thrpt   20  429.090 ± 0.842  ops/us
[info] Benchmarks.explicitSafePresent       thrpt   20  231.400 ± 0.660  ops/us
[info] Benchmarks.optionSafeAbsent          thrpt   20  139.369 ± 0.272  ops/us
[info] Benchmarks.optionSafePresent         thrpt   20  129.394 ± 0.102  ops/us
[info] Benchmarks.loopSafeAbsent            thrpt   20  114.330 ± 0.113  ops/us
[info] Benchmarks.loopSafePresent           thrpt   20   59.513 ± 0.097  ops/us
[info] Benchmarks.nullSafeNavigatorAbsent   thrpt   20  274.222 ± 0.441  ops/us
[info] Benchmarks.nullSafeNavigatorPresent  thrpt   20  181.356 ± 1.538  ops/us
[info] Benchmarks.tryCatchSafeAbsent        thrpt   20  254.158 ± 0.686  ops/us
[info] Benchmarks.tryCatchSafePresent       thrpt   20  230.081 ± 0.659  ops/us
[info] Benchmarks.monocleOptionalAbsent     thrpt   20   77.755 ± 0.800  ops/us
[info] Benchmarks.monocleOptionalPresent    thrpt   20   36.446 ± 0.506  ops/us
[info] Benchmarks.nullSafeDslAbsent         thrpt   30  228.660 ± 0.475  ops/us
[info] Benchmarks.nullSafeDslPresent        thrpt   30  119.723 ± 0.506  ops/us
[success] Total time: 3909 s, completed Feb 24, 2019 3:03:02 PM
{% endhighlight %}
</details><br/>

In order to summarize the pros and cons of each approach, let's evaluate them based on null-safty, read/writability, and efficiency.

|                      	| Null-safe 	| Readable / Writable 	| Efficient 	|
|----------------------	|-----------	|-------------------	|-----------	|
| Normal access        	| :no_entry:         	| :heavy_check_mark:️            | :heavy_check_mark:️    |
| Explicit null-checks 	| :heavy_check_mark:️    | :no_entry:                 	| :heavy_check_mark:️    |
| Option flatMap       	| :heavy_check_mark:️    | :no_entry:                 	| :no_entry:         	|
| For loop flatMap     	| :heavy_check_mark:️    | :warning:️                 	| :no_entry:         	|
| Null-safe navigator  	| :heavy_check_mark:️    | :warning:️                 	| :warning:️         	|
| Try-catch NPE        	| :heavy_check_mark:️    | :heavy_check_mark:️            | :warning:️         	|
| Monocle Optional (lenses)| :heavy_check_mark:️ | :skull:                       | :warning:         	|
| thoughtworks NullSafe DSL| :heavy_check_mark:️ | :heavy_check_mark:            ️| :warning:️         	|

Key: :heavy_check_mark:️ = Good, :warning: = Sub-optimal, :no_entry: = Bad

After evaluating all of the options available, I wasn't quite satisfied with any of them, so I decided to create a new way via Scala's [blackbox macros](https://docs.scala-lang.org/overviews/macros/blackbox-whitebox.html).