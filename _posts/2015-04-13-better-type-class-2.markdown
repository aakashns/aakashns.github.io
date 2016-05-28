---
layout: post
title:  "Scrap Your Type Class Boilerplate (2/2)"
date:   2015-04-13 11:30:14
categories: scala
permalink: better-type-class-2.html
comments: true
---

For the **TL; DR** version, go straight to the [Summary](#summary).

* TOC
{:toc}

Recap
------
The [previous post][better-type-class] covered some ways to make type classes easier to use. For a quick recap, see the [summary][better-type-class-summary]. In this post, we will attempt to eliminate some of the boilerplate associated with creating and using type classes and their instances. Here is the implementation of the `Group` type class that we had at the end of the previous post :

{% gist aakashns/a69d8e602c3093356b7c Group.scala %}

Inside the `Group` companion object, we define instances of `Group` for `Int`, `Double` and pairs :

{% gist aakashns/a69d8e602c3093356b7c GroupInstances.scala %}

And here, the type class is used to define some generic functions :
{% gist aakashns/a69d8e602c3093356b7c sum.scala %}

More Improvements
-----------------

One of the concerns raised earlier but not addressed was that the code for defining type class instances is rather verbose and repetitive, and if the type class has lots of abstract methods or if you're defining many instances, this can lead to a lot of boilerplate. I alluded to something called *the pain of overriding*, which is the root of much of this boilerplate. Here is the definition <s>from Wikipedia</s> :

<blockquote class="twitter-tweet tw-align-center" lang="en"><p>The pain of overriding : The <a href="https://twitter.com/hashtag/Scala?src=hash">#Scala</a> compiler can infer batshit crazy types, still demands the full signature to override/implement methods.</p>&mdash; Aakash N S (@aakashns) <a href="https://twitter.com/aakashns/status/584373978849943552">April 4, 2015</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>
<p></p>


It is annoying to write all these types, and it becomes much more annoying when you have to change something, because you have to make the same change in *SO* many places, and so you end up copy-pasting things all over the place, but that leaves you with a bunch of compilation errors, and then you spend a day figuring out the types to fix those errors, and then you finally fix them, and you reward yourself with a cookie or whatever, but then two days later you realize you've made an error in the logic because you were too busy trying to tell the compiler what it already knows!

![All those types!][frustrated-gif]{: .center-image .responsive-image }


### Helper Class ###

To help eliminate some of this boilerplate, let's define a class `GroupAux` as follows :

{% gist aakashns/c33c8cdd4b2d63bab60a GroupAux.scala %}

This class does nothing special. It simply moves the methods to be implemented into constructor arguments. Using this helper class, the code for defining instances of `Group` for `Int` and `Double` looks like this :
{% gist aakashns/c33c8cdd4b2d63bab60a IntGroup.scala %}

Isn't this much nicer? By using anonymous functions as constructor arguments to `GroupAux`, we're letting the compiler do all the hard work of figuring out the right types. By the way, anonymous functions are also called lambdas (λ). And here's the definition for `pairGroup` :
{% gist aakashns/c33c8cdd4b2d63bab60a pairGroup.scala %}

You can make the anonymous functions more descriptive, [if you're not into the whole brevity thing][dude-brevity] :
{% gist aakashns/c33c8cdd4b2d63bab60a GroupInstancesLong.scala %}

Admittedly, this is not a huge improvement if you only have a few instances of the type class. Some people would argue that this not an improvement at all, because it compromises on readability. Here are a few arguments in favor of this style :

1. It avoids the *pain of overriding*. You no longer have to tell the compiler what it can figure out on its own. If the instances of your type class go into dozens, or if you have many abstract methods to be implemented, this can save you the trouble of having to write all the types all the time.
3. Anonymous functions are not unreadable if you already know the types. That's precisely why we use them. When both the reader and the compiler already know the types, they're just noise in the code. Moreover, you can optionally provide the types, if you *really* want to.
4. It's not all or nothing. You don't have to use it all the time. Use it when it saves you precious lines without compromising on readability, like in the case of `IntGroup` and `DoubleGroup`. If you think the definition of `PairGroup` feels unreadable, don't use `GroupAux`! Extend `Group` and override the methods directly.

Note that this is a general technique that you can apply to any trait / abstract class. Just define a helper class that takes function values as constructor arguments, and uses them to implement the abstract methods.

### The TypeClassCompanion Trait ###

This is another minor improvement, that can help you avoid having to write the `apply` method for your type class companion object. We define a trait called `TypeClassCompanion` that companion objects can extend, to automatically get the `apply` method :
{% gist aakashns/340068446e3fd59196be TypeClassCompanion.scala %}

Note that the apply method is marked as `@inline`, which tells the compiler to replace calls like `Group[Int]` with `implicitly[Group[Int]]`. Not surprisingly, the implicitly method in Predef is [inlined by the complier][scala-implicitly-doc] too, so the compiler will replace `implicitly[Group[Int]]` with an implicit value of type `Group[Int]` *summoned from the nether world*. Long story short, the compiler will replace the expression `Group[Int]` with `IntGroup`.

Using the `TypeClassCompanion` trait, the `Group` companion object looks like this :
{% gist aakashns/340068446e3fd59196be Group.scala %}

There isn't much more to it than that. It saves you one line of code for each of your type classes.

### Implicit Class for Pretty Syntax (or not) ###

With context bounds and the `apply` method on the companion object, the syntax for using type classes is already pretty clean. However, it is often the case that the methods defined in the type class are unary (a function of type `T => T`) or binary (a function of type `(T, T) => T`) operations. This is certainly true for `Group` : `plus` and `minus` are binary operations, and `inverse` is a unary operation. Wouldn't it be nice if we could simply write `(1, 2) + (3, 4)` instead of `Group[(Int, Int)].plus((1, 2), (3, 4))`? Here's some advice I just made up :

<blockquote class="twitter-tweet tw-align-center" lang="en"><p>As a rule of thumb, the answer to every question of the form “Can I do XYZ in <a href="https://twitter.com/hashtag/Scala?src=hash">#Scala</a>?” is “Yeah, totally man! Just use implicits.”</p>&mdash; Aakash N S (@aakashns) <a href="https://twitter.com/aakashns/status/587687158032412672">April 13, 2015</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>
<p></p>

Armed with this newfound wisdom, let's define a class called `GroupOps` that wraps objects and adds methods for group operations on the underlying objects :
{% gist aakashns/f6c61a5f8af587dde756 GroupAnyVal.scala %}

By making `GroupOps` an implicit class, we're simply telling the compiler to add an implicit conversion from `T` to `GroupOps` for types `T` that are members of the type class `Group`. The above piece of code is equivalent to the following :
{% gist aakashns/f6c61a5f8af587dde756 GroupAnyVal2.scala %}

Since we cannot write a `def` in the global scope, implicit classes can only be defined inside another class / trait / object, in this case `GroupSyntax`. Let's use the new syntax to simplify the code for `pairGroup` and the sum functions :
{% gist aakashns/f6c61a5f8af587dde756 Test.scala %}

Clearly, this technique substantially cleans up the interface for using group operations. But despite the benefits, you may not want to do this all the time, because :

1. It uses an implicit conversion. It is not easy to tell where the operator `|+|` came from unless you know where to look. Many people find this confusing, and are going to avoid using it, no matter what you tell them.
3. It may potentially create a new instance of the type class each time you use it. If you find yourself using `Group[(Int, Int)]` 10 times in a block, you're better off defining a local val `intPairGroup = Group[(Int, Int)]` and using that, because `Group[(Int, Int)]` results in a call to `pairGroup[Int, Int]`, which instantiates a new object each time it is called. However, if you're writing `a |+| b` 10 times in a block, there is no easy way to avoid calling `pairGroup[Int, Int]` that many times.

Although the pretty syntax is nice to have, it comes at a cost. You should consider the use-cases of your type classes, and evaluate how they are affected by these costs. This might upset a few people, but I personally think that explicit is indeed better that implicit in this particular instance. So think twice before doing this.

Summary
--------

Type classes are awesome and you should use them everywhere. Make sure you do them right, and everyone will be happy. In addition to the techniques described in [the previous post][better-type-class-summary] :

- Define a helper class to help reduce boilerplate for defining type class instances.
{% gist aakashns/c33c8cdd4b2d63bab60a GroupAux.scala %}

- Use a companion trait to automatically add the `apply` method to your type class companion objects.
{% gist aakashns/340068446e3fd59196be TypeClassCompanion.scala %}

- You may define implicit classes to provide some nice syntax and pretty operators, but be aware of the costs, and also be aware that many people just won't use them. This is best provided as an optional feature.
{% gist aakashns/f6c61a5f8af587dde756 GroupSyntax.scala %}

[That's all folks!][thats-all-folks]

[frustrated-gif]: http://media.tumblr.com/tumblr_mcl3yteMCb1rw8654.gif
[putin-cookie]: http://i3.kym-cdn.com/photos/images/original/000/610/825/eec.jpg
[better-type-class]: /better-type-class.html
[better-type-class-summary]: /better-type-class.html#summary
[dude-brevity]: https://www.youtube.com/watch?v=vgUHqk-4omc
[scala-implicitly-doc]: https://github.com/scala/scala/blob/v2.11.6/src/library/scala/Predef.scala#L130
[thats-all-folks]: https://www.youtube.com/watch?v=b9434BoGkNQ