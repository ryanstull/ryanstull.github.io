---
layout: post
title:  "Null-safety Part 4: The Future of Scala"
date:   2019-03-10 13:42:24 -0400
categories: blog
tag: scala
---

As I mentioned in the first article in this series, if a programming language's type system supports certain features, such as generic types, or union types,  it can safely represent nullable types.  When Java was created, in 1996,  it didn't have either of these features which is why it has the problem with null-safety that it does today.  Generics were added later in Java 5, in 2004,  but the fact that `null` was a subtype of all objects didn't change.  When Scala was created, around the same time as Java 5, it had generics from the start, which is why `Option` [existed from the beginning](https://github.com/scala/scala/blob/v1.0.0-b5/sources/scala/Option.scala#L21) and has been the default way to handle potentially absent values.
> Fun fact: [Martin Odersky](https://en.wikipedia.org/wiki/Martin_Odersky), the creator of Scala, was one of the people who helped create the [initial design for generics in Java](https://homepages.inf.ed.ac.uk/wadler/gj/)

With the coming of a new major version of Scala, [Scala 3 AKA Dotty](https://dotty.epfl.ch/), Scala will be introducing [Union Types](https://dotty.epfl.ch/docs/reference/new-types/union-types.html).  Union types allow for the description of types that are the union (in set theory terms) of two or more types.  For example:

{% highlight scala%}
val x: String|Int = 3

println(x match {
    case i: Int => "It's an Int"
    case s: String => "It's a String"
})
{% endhighlight %}

This will open up a lot of interesting opportunities for expressiveness in the type system.  One such possibility, which is actually being added to Scala 3 as an opt-in feature, is to change where `Null` fits into the type system so that reference types are not nullable by default.

The modified version would look like this:

{% include image.html url="/assets/images/posts/51210362-9bf86900-18e0-11e9-9485-f40dc9061527.png" description="New Scala type hierarchy with explicit null enabled" %}

Which would mean the following code would no longer typecheck

{% highlight scala%}
val x: String = null // error: found `Null`,  but required `String`
{% endhighlight %}

Instead, you would have to use a type union to assign `null` here

{% highlight scala%}
val x: String|Null = null // ok
{% endhighlight %}

Which is great!  Because this solves the original and fundamental problem we were discussing in part 1, which is being able to distinguish between nullable and non-nullable references.  

I could go in depth about the exciting changes in this project and how it would work, but I'd just be reiterating the [project description](https://gist.github.com/abeln/9f79774bac111d99b3ae2cb9016a33e6), so I'll just encourage you to check that out.  There's also the description of this feature [on the dotty website](https://dotty.epfl.ch/docs/reference/other-new-features/explicit-nulls.html).  I'm very excited to see these features getting added to Scala, and I think it's a big step forward for the language.  

Hopefully you've found this series useful, and you will find the ScalaNullSafe macro useful for the rest of the time you're using Scala 2 :sweat_smile:.  Thanks!

<div class="PageNavigation">
  {% if page.previous.url %}
    <a class="prev" href="{{page.previous.url}}">&laquo; {{page.previous.title}}</a>
  {% endif %}
  {% if page.next.url %}
    <a class="next" href="{{page.next.url}}">{{page.next.title}} &raquo;</a>
  {% endif %}
</div>
