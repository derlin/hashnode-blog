---
title: "Kotlin is `fun` - lambdas with receivers"
seoDescription: "Learn what are lambdas with receivers in Kotlin through a simple example."
datePublished: Tue Apr 11 2023 17:09:33 GMT+0000 (Coordinated Universal Time)
cuid: clgcipcr7000f09l74qhhf8bi
slug: kotlin-is-fun-lambdas-with-receivers
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1681232861014/0d45b776-5326-4e6b-af17-201e829cc554.png
tags: beginners, functional-programming, kotlin, kotlin-beginner

---

As we already covered, Kotlin provides the ability to extend a class or an interface with new functionality without having to inherit from the class or use design patterns such as Decorator ⮕ [Kotlin is `fun` - extension functions](https://dev.to/derlin/kotlin-is-fun-extension-functions-5bee).

Kotlin also treats functions as [first-class citizens](https://kotlinlang.org/docs/lambdas.html), meaning they can be passed around. It also, of course, supports generics ⮕ [Kotlin is `fun` - Function types, lambdas, and higher-order functions](https://dev.to/derlin/kotlin-is-fun-function-types-lambdas-and-higher-order-functions-3lb4). <sub>(for more, have a look at the Series)</sub>

This means we can write something like this:

```kotlin
fun <T> T.doWith(op: (T) -> Unit) {
    op(this)
}
```

This extension function can be called on anything and will simply pass this *anything* as an argument to the lambda passed as a parameter. For example:

```kotlin
val someList = listOf(1, 2, 3, 4)

someList.doWith ({ list -> 
    println(list.size)
    println(list.sum())
})
```

This syntax is not following the conventions. Let's rewrite it in proper Kotlin, by [removing parenthesis](https://kotlinlang.org/docs/lambdas.html#passing-trailing-lambdas) and using [the implicit `it` parameter](https://kotlinlang.org/docs/lambdas.html#it-implicit-name-of-a-single-parameter):

```kotlin
someList.doWith { // it -> List<Int>
    println(it.size)
    println(it.sum())
}
```

This is better. Notice though that the `it` is redundant. We would rather be able to write:

```kotlin
someList.doWith { // this -> List<Int>
  println(size)
  println(sum())
}
```

But how? Read on!

## Lambdas with receivers

To make this magic happen, the only necessity is to change the type of the `op` argument from `(T) -> Unit` to `T.() -> Unit`. This special syntax, `A.(B)`, denotes a [*receiver object*](https://kotlinlang.org/docs/lambdas.html#function-literals-with-receiver):

> *Inside the body of the function literal, the receiver object passed to a call becomes an implicit this, so that you can access the members of that receiver object without any additional qualifiers, or access the receiver object using a* `this` expression.

The final implementation is now:

```kotlin
fun <T> T.doWith(op: T.() -> Unit) {
    op(this)
}

listOf(1, 2, 3, 4).doWith {
    println(size)
    println(this.sum()) // "this" is optional 
}
```

It is just a bit of sugar-coating, but admit this syntax looks cool 😎.

---

**Side note**

In the real world, we would also want this `doWith` function to be [*inline*](https://kotlinlang.org/docs/inline-functions.html), to improve performances:

```kotlin
inline fun <T> T.doWith(op: T.() -> Unit) {
    op(this)
}
```

---

For more information on receivers (and more examples), see the docs on [Function literals with receivers](https://kotlinlang.org/docs/lambdas.html#function-literals-with-receiver).