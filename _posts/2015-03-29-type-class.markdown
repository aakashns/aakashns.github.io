---
layout: post
title:  "Type Classes in Scala"
date:   2015-03-27 11:30:14
categories: scala
permalink: type-class.html
comments: true
---

Consider the following functions :

- `sum` : which takes a list of integers and returns their sum.
{% gist aakashns/4599e5db4b28e02f79b9 sum.scala %}

- `sumDifference` : which takes two lists of integers and returns the difference of the their sums.
{% gist aakashns/4599e5db4b28e02f79b9 sumDifference.scala %}

- `sumNonEmpty` : which takes a list of integers and returns an `Option[Int]`, which is `None` if the list is empty, and a `Some` containing the sum of the members otherwise.
{% gist aakashns/4599e5db4b28e02f79b9 sumNonEmpty.scala %}

Suppose we want to generalize these functions to work on lists of some other type, such as `Double`s, pairs of integers i.e. `(Int, Int)` with `(x1, x2) + (y1, y2)` defined as `(x1 + y1, x2 + y2)`, or pairs of `Double`s. Take a moment and think about how you would do it.
All these types seem to naturally support the operations addition and subtraction, but this fact is not encoded in their inheritance hierarchy, which makes it extremely difficult to write generic functions that work for all of them.

To solve this problem, let's define a trait called `Group` :
{% gist aakashns/108da2dfc2b6d772f544 Group.scala %}

`Group` is a generic trait, which takes a type parameter `T` and declares the operations plus, minus and inverse for objects of type `T`, as well as an element zero. Note that minus is defined using plus and inverse because `x - y = x + (-y)`(how clever!).

Groups are a [real thing][group-wikipedia], and the operation plus needs to be associative i.e. `(x + y) + z` should be equal to `x + (y + z)`, otherwise we may get weird results.

Instances of the trait `Group` should ideally be defined as *implicit* members of the object `Group`, which lives is the same file as the trait. It is called the *companion object* of the trait. When the Scala compiler needs to supply an implicit parameter of type `Group[T]`, it looks inside the `Group` companion object. So, we won't need to pass around `Group` instances explicitly while writing and using functions that depend on group operations.

Let's define an instance of the trait for integers (inside the `Group` object, of course):
{% gist aakashns/108da2dfc2b6d772f544 IntGroup.scala %}

The instance for `Double` looks almost identical, except for the types :
{% gist aakashns/108da2dfc2b6d772f544 DoubleGroup.scala %}

Let's try something a little more interesting. Here's a method that produces a an instance of `Group` for the pair `(T1, T2)`, if it can find instances of `Group` for `T1` and `T2` :
{% gist aakashns/108da2dfc2b6d772f544 PairGroup.scala %}

This single method will automatically provide `Group` instances for `(Int, Int)`, `(Int, Double)`, `(Double, Int)` and `(Double, Double)` whenever we need them. Isn't that neat?

Alright, now that we've done all the hard work, let's use the trait to make our `sum` functions generic :
{% gist aakashns/108da2dfc2b6d772f544 sum.scala %}

With these small modifications, you can now use these functions on lists of any type `T` for which the compiler can find an implicit object of type `Group[T]`:
{% gist aakashns/108da2dfc2b6d772f544 test.scala %}

That's it! That's the type class pattern. `Group` is called a type class (because it represents a *class of types* that support group operations), and the types `Int`, `Double`, `(Int, Int)` etc. are called members of the type class `Group` because the compiler can find instances of the type class for these types.

This post merely scratches the surface of the use cases and benefits of using type classes in Scala. Type classes are awesome and you should use them all over your code base. There are many great resources (much better than this one) for learning about type classes in Scala, like [this][dan-rosen-tutorial], [this][daniel-westheide-tutorial] and [this][safari-books-tutorial]. The only reason this post exists is so that I can point out the deficiencies in this naive implementation, and show you how to make it much better [in my next post][type-class-boilerplate].

[group-wikipedia]: http://en.wikipedia.org/wiki/Group_(mathematics)
[type-class-boilerplate]: /better-type-class.html
[dan-rosen-tutorial]: https://www.youtube.com/watch?v=sVMES4RZF-8
[daniel-westheide-tutorial]: http://danielwestheide.com/blog/2013/02/06/the-neophytes-guide-to-scala-part-12-type-classes.html
[safari-books-tutorial]: https://blog.safaribooksonline.com/2013/05/28/scala-type-classes-demystified/