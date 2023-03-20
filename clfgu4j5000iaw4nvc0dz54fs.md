---
title: "Kotlin is `fun` - extension functions"
seoDescription: "Let's discover what are extension functions in Kotlin, and why they are so useful."
datePublished: Mon Mar 20 2023 13:00:39 GMT+0000 (Coordinated Universal Time)
cuid: clfgu4j5000iaw4nvc0dz54fs
slug: kotlin-is-fun-extension-functions
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1678724508012/282fc323-90f5-4d2f-bcc5-b6c77e57b16d.png
tags: beginners, functional-programming, kotlin, kotlin-beginner

---

Kotlin provides the ability to **extend a class or an interface with new functionality without having to inherit from the class or use design patterns such as Decorator**. This is done via special declarations called [*extensions*](https://kotlinlang.org/docs/extensions.html).

For example, it is possible to write new functions for a class or an interface from a third-party library that you can't modify, or from built-in types. Such functions can be called in the usual way **as if they were methods of the original class**. This mechanism is called an [*extension function*](https://kotlinlang.org/docs/extensions.html#extension-functions).

This is an awesome feature and will shed light on the Kotlin standard library. Read on!

---

In this part:

* [The basics](#heading-the-basics)
    
* [They are still functions](#heading-they-are-still-functions)
    
* [A compile-time sugarcoating](#heading-a-compile-time-sugarcoating)
    
* [The magic of Kotlin's standard library explained](#heading-the-magic-of-kotlins-standard-library-explained)
    

<sub>🔖 I created this Table of Contents using </sub> [**<sub>BitDownToc</sub>**](https://derlin.github.io/bitdowntoc/)<sub>.</sub>

Previous articles in the [Series](https://blog.derlin.ch/series/kotlin-is-fun):

1. [Some cool stuff about functions](https://blog.derlin.ch/kotlin-is-fun-some-cool-stuff-about-kotlin-functions)
    
2. [**Kotlin is** `fun` **- Function types, lambdas, and higher-order functions**](https://blog.derlin.ch/kotlin-is-fun-function-types-lambdas-and-higher-order-functions)
    

---

## The basics

Consider the following piece of code:

```kotlin
capitalize(trimSpaces(myString))
```

Wouldn't this be more readable like this?

```kotlin
myString.trimSpaces().capitalize()
```

This is exactly what extension functions let you do! Let's implement the example above.

Instead of this dull regular function:

```kotlin
// regular function
fun trimSpaces(s: String): String =
    s.replace("\\s+".toRegex(), " ").trim()
```

We can define `trimSpaces` as an extension to the `String` class like this:

```kotlin
// extension function
fun String.trimSpaces(): String =
   this.replace("\\s+".toRegex(), " ").trim()
```

Or by omitting the `this` receiver:

```kotlin
// same extension function,
// but using the implicit receiver (no "this")
fun String.trimSpaces(): String =
   replace("\\s+".toRegex(), " ").trim()
```

As you can see, an extension function simply prefixes the method name with a **receiver type** (`ReceiverType.methodName`), which refers to the type being extended. When called, the method can access the **receiver** - the instance it is called on - directly or using the `this` keyword. That's it.

The *receiver type* can be a built-in type (`String`, `Int`, `Any`, ...), a collection type (`List<Int>`, `Map<String, Map<String, Any>>`, ...), a nullable type (`String?`, `Any?`), or even a generic type (`T`).

Long story short, it can be applied to pretty much anything. This makes this concept very powerful and versatile.

Here is another, more complex one:

```kotlin
fun <T> List<T>?.prettyPrint() {
    if (this.isNullOrEmpty()) {
        println("Empty list")
    } else {
        println("List content ($size elements):")
        println(joinToString("\n") { "* $it" })
    }
}
```

The latter may be called on any list:

```kotlin
// an empty list
emptyList<String>().prettyPrint()

// a list of `Any`
listOf("x", 1).prettyPrint()

// or even a null
val lst: List<Int>? = null
lst.prettyPrint()
```

For real-life examples, see the end of [my text utils in goodreads-metadata-fetcher](https://github.com/derlin/goodreads-metadata-fetcher/blob/main/src/main/kotlin/ch/derlin/grmetafetcher/internal/text.kt), or [my MiscUtils class (Android) in easypass](https://github.com/derlin-easypass/easypass-android-mse/blob/master/app/src/main/java/ch/derlin/easypass/helper/MiscUtils.kt).

Cherries on the cake, IDEs auto-suggest extensions functions for you, so they pop up on the auto-complete dropdowns 💖.

## They are still functions

As extension functions are, *in fine*, functions, they support the same [visibility modifiers](https://kotlinlang.org/docs/visibility-modifiers.html) as regular functions and can be imported around.

```kotlin
package ch.derlin.utils

fun List<String>.doStuff() { /*...*/ }
```

```kotlin
package ch.derlin.foo

import ch.derlin.utils.doStuff

listOf("a", "b").doStuff()
```

They can even be made *scope local* (declared inside another method), for example:

```kotlin
fun fuzzyCompare(expected: String, actual: String): Boolean {
  // here, this complex chain is used twice in the body
  // instead of copy-pasting, we can write it once ...
  fun String.cleaned() = lowercase()
      .removeDiacritics()
      .removeInitials()
      .replace("[^a-z0-9]".toRegex(), "")
      .replace(" +".toRegex(), " ")
      .trim()

  // ... and use it twice
  return actual.cleaned() == expected.cleaned()
}
```

## A compile-time sugarcoating

It is essential to understand that extensions do not modify the classes they extend. By defining an extension, **we are not inserting new members into a class, only making new functions callable with the dot-notation** on variables of this type.

More importantly, extension functions are *statically resolved*. In other words, the magic happens at **compile time**. This has some consequences.

Consider the following:

```kotlin
open class Shape
class Rectangle: Shape()

fun Shape.getName() = "Shape"
fun Rectangle.getName() = "Rectangle"

fun printName(s: Shape) {
    println(s.getName())
}
```

In this case, what would be the output of:

```kotlin
printName(Rectangle()) // 1
printName(Shape())     // 2
```

Well, this will actually print `Shape` twice. Why? Because in `printName`, the parameter is declared as `Shape`, so even if the parameter is of another type at runtime, at build time it is known only as a shape.

(I ignored it for a very long time and never encountered such a situation, but it is something to keep in the back of your mind.)

## The magic of Kotlin's standard library explained

The Kotlin Standard Library makes heavy use of extension functions. Ever had a look at the signature of [`joinToString()`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/join-to-string.html), [`all { ... }`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/all.html) or [`lines()`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.text/lines.html)?

```kotlin
// lines() is actually an extension function working
// on any CharSequence, which is a base class for String
fun CharSequence.lines(): List<String>
```

They are all extension functions! Just look at their definitions, you'll see :)

---

There is so much more to say about extension functions. I strongly encourage you to read the [kotlin doc on extensions](https://kotlinlang.org/docs/extensions.html#extension-functions) and start playing around with them yourselves!

But beware, it's addictive 😉.