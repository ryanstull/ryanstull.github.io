---
layout: post
title:  "Null-safety Part 3: Announcing ScalaNullSafe!"
date:   2019-03-10 13:42:24 -0400
categories: blog
tag: scala
---

## Existing Options

Let's review the pros and cons of all the ways of doing this:

|                      	| Null-safe 	| Readable/Writable 	| Efficient 	|
|----------------------	|-----------	|-------------------	|-----------	|
| Normal access        	| ⛔         	| ✅                 	| ✅         	|
| Explicit null-checks 	| ✅         	| ⛔                 	| ✅         	|
| Option flatMap       	| ✅         	| ⛔                 	| ⛔         	|
| For loop flatMap     	| ✅         	| ⚠️                 	| ⛔         	|
| Null-safe navigator  	| ✅         	| ⚠️                 	| ⚠️         	|
| Try-catch NPE        	| ✅         	| ✅                 	| ⚠️         	|

So unfortunately there's at least one issue with all of these different approaches.  The best one in terms of performance is the explicit null-checks, and the only problem with it is that it's hard to read and very hard to write; but what if we could make it easy to do so?

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

//^ Which gets transformed into:
if(a != null){
  val b = a.b
  if(b != null){
    b.c
  } else null
} else null

val a2 = A(B(C))

?(a2.b.c) //returns C
{% endhighlight %}

With this approach we get the best possible solution, something that is null-safe, easy to read and write, and efficient!

## How to get it

I've published the [source code for the macro on github](https://github.com/ryanstull/ScalaNullSafe),  and I've also published the [jar on maven](https://mvnrepository.com/artifact/com.ryanstull/scalanullsafe), so it can easily be incorporated into other projects.

To add it to your project, just add the dependency

{% highlight scala %}
libraryDependencies += "com.ryanstull" %% "scalanullsafe" % "1.2.5"
{% endhighlight %}

and then import

{% highlight scala %}
import com.ryanstull.nullsafe._
{% endhighlight %}

and you're good to go!

## More features

### Opt macro
There's also two other variants of the macro, `opt`, which is useful for interoping with Java code and work as follows:

{% highlight scala %}

opt(a.b.c) //returns None

//^ Which gets transformed into:
if(a != null){
  val b = a.b
  if(b != null){
    Option(b.c)
  } else None
} else None

val a2 = A(B(C))

opt(a2.b.c) //returns Some(C)
{% endhighlight %}

### notNull macro

and `notNull` which works like this:

{% highlight scala %}

notNull(a.b.c) //returns false

//^ Which gets transformed into:
if(a != null){
  val b = a.b
  if(b != null){
    b.c != null
  } else false
} else false

val a2 = A(B(C))

notNull(a2.b.c) //returns true
{% endhighlight %}

### Safe translation

All of the above work for method invocation as well as property access, and the two can be intermixed. For example:

{% highlight scala %}
?(someObj.methodA().field1.twoArgMethod("test",1).otherField)
{% endhighlight %}

will be translated properly.

Also the macro makes the arguments to method and function calls null-safe as well.  So in the case of:

{% highlight scala %}
?(a.b.c.method(d.e.f))
{% endhighlight %}

you don't have to worry if `d` or `e` would be `null`.

### Custom default for `?`

For the `?` macro, you can also provide a custom default instead of `null`, by passing it in as the second parameter. For example

{% highlight scala %}
case class Person(name: String)

val person: Person = null

assert(?(person.name,"") == "")
{% endhighlight %}

### ?? macro

There's also a `??` ([null coalesce operator](https://en.wikipedia.org/wiki/Null_coalescing_operator)) which is used to select the first non-null value from a var-args list of expressions.

{% highlight scala %}
case class Person(name: String)

val person = Person(null)

assert(??(person.name)("Bob") == "Bob")

val person2: Person = null
val person3 = Person("Sally")

assert(??(person.name,person2.name,person3.name)("No name") == "Sally")
{% endhighlight %}

The null-safe coalesce operator also rewrites each arg so that it's null safe. So you can pass in `a.b.c` as an expression without worrying if `a` or `b` are `null`. To be more explicit, the `??` macro would translate `??(a.b.c,a2.b.c)(default)` into

{% highlight scala %}
{
    val v1 = if(a != null){
      val b = a.b
      if(b != null){
        val c = b.c
        if(c != null){
          c
        } else null
      } else null
    } else null
    if(v1 != null) v1
    else {
        val v2 = if(a2 != null){
          val b = a2.b
          if(b != null){
            val c = b.c
            if(c != null){
              c
            } else null
          } else null
        } else null
        if (v2 != null) v2
        else default
    }
}
{% endhighlight %}

Compared to the `?` macro in the case of a single arg, the `??` macro checks that the entire expression is not null. Whereas the `?` macro would just check that the preceding elements (e.g. `a` and `b` in `a.b.c`) aren't `null` before returning the default value.

### Efficient null-checks

The macro is also smart about what it checks for null, so anything that is `<: AnyVal` will not be checked for null.  For example

{% highlight scala %}
case class A(b: B)
case class B(c: C)
case class C(s: String)

?(a.b.c.s.asInstanceOf[String].charAt(2).*(2).toString.getBytes.hashCode())
{% endhighlight %}

Would be translated to:

{% highlight scala %}
if (a != null)
  {
    val b = a.b;
    if (b != null)
      {
        val c = b.c;
        if (c != null)
          {
            val s = c.s;
            if (s != null)
              {
                val s2 = s.asInstanceOf[String].charAt(2).$times(2).toString();
                if (s2 != null)
                  {
                    val bytes = s2.getBytes();
                    if (bytes != null)
                      bytes.hashCode()
                    else
                      null
                  }
                else
                  null
              }
            else
              null
          }
        else
          null
      }
    else
      null
  }
else
  null
{% endhighlight %}

## Performance

Here's the result of running the included jmh benchmarks:

{% include image.html url="/assets/images/posts/throughput.png" description="Performance of different null-safe implementations" %}

Here's the same data in tabular form

```
[info] Benchmark                             Mode  Cnt    Score   Error   Units
[info] Benchmarks.fastButUnsafe             thrpt   20  230.157 ± 0.572  ops/us
[info] Benchmarks.ScalaNullSafeAbsent       thrpt   20  428.124 ± 1.625  ops/us
[info] Benchmarks.ScalaNullSafePresent      thrpt   20  232.066 ± 0.575  ops/us
[info] Benchmarks.explicitSafeAbsent        thrpt   20  429.090 ± 0.842  ops/us
[info] Benchmarks.explicitSafePresent       thrpt   20  231.400 ± 0.660  ops/us
[info] Benchmarks.optionSafeAbsent          thrpt   20  139.369 ± 0.272  ops/us
[info] Benchmarks.optionSafePresent         thrpt   20  129.394 ± 0.102  ops/us
[info] Benchmarks.loopSafeAbsent            thrpt   20  114.330 ± 0.113  ops/us
[info] Benchmarks.loopSafePresent           thrpt   20   59.513 ± 0.097  ops/us
[info] Benchmarks.monocleOptionalAbsent     thrpt   20   77.755 ± 0.800  ops/us
[info] Benchmarks.monocleOptionalPresent    thrpt   20   36.446 ± 0.506  ops/us
[info] Benchmarks.nullSafeNavigatorAbsent   thrpt   20  274.222 ± 0.441  ops/us
[info] Benchmarks.nullSafeNavigatorPresent  thrpt   20  181.356 ± 1.538  ops/us
[info] Benchmarks.tryCatchSafeAbsent        thrpt   20  254.158 ± 0.686  ops/us
[info] Benchmarks.tryCatchSafePresent       thrpt   20  230.081 ± 0.659  ops/us
[success] Total time: 3909 s, completed Feb 24, 2019 3:03:02 PM
```

You can find the source code for the JMH benchmarks [here](https://github.com/ryanstull/ScalaNullSafe/blob/ebc0ed592fa5997a9c7b868cf8cdcea590e8ae07/benchmarks/src/test/scala/com/ryanstull/nullsafe/Benchmarks.scala#L18). All of the 'Present' benchmarks are where the value was actually defined and the 'Absent' ones are where one of the intermediate values, was `null`; or in other words, where an NPE would have occurred.

If you want to run the benchmarks yourself, just run `sbt bench`.

## Conclusion

These benchmarks compare all of the known ways (or at least the ways that I know of) to handle null-safety in scala. It demonstrates that the explicit null-safety is the highest performing and that the ScalaNullSafe macro has equivalent performance.

<br/>

***

<br/>

In the next section we'll examine how the usage of `null` will evolve in the next major version of Scala, Scala 3, AKA Dotty.

<div class="PageNavigation">
  {% if page.previous.url %}
    <a class="prev" href="{{page.previous.url}}">&laquo; {{page.previous.title}}</a>
  {% endif %}
  {% if page.next.url %}
    <a class="next" href="{{page.next.url}}">{{page.next.title}} &raquo;</a>
  {% endif %}
</div>