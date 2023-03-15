---
title: "Kotlin is `fun` - Function types, lambdas, and higher-order functions"
seoDescription: "Kotlin functions are first-class citizens. Let's understand together what it means, and what are function types, higher-order functions, and lambdas."
datePublished: Mon Mar 13 2023 15:42:02 GMT+0000 (Coordinated Universal Time)
cuid: clf6zt3xo000209l42btbc2ym
slug: kotlin-is-fun-function-types-lambdas-and-higher-order-functions
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1678724457375/e92a4a62-5995-4670-bdb6-0ec454f1c437.png
tags: functional-programming, kotlin, kotlin-beginner

---

Kotlin treats functions as first-class citizens.

[First-class](https://en.wikipedia.org/wiki/First-class_function) means functions can be stored in variables and data structures and can be passed as arguments to and returned from other functions ([higher-order functions](https://en.wikipedia.org/wiki/Higher-order_function)). This is one of the idioms of functional programming, a programming style that I love.

To better understand what it means, and why it is so cool, let's go through the theory: what are function types, why they matter, and how they relate to lambdas and higher order functions.

*Did I lose you? Then read on! Everything will become clear, I promise!* 😊

---

In this part:

* [Function types](#heading-function-types)
    
    * [Instantiating function types](#heading-instantiating-function-types)
        
    * [Invoking a function type](#heading-invoking-a-function-type)
        
* [Higher-order functions](#heading-higher-order-functions)
    
* [Bonus: function types under the hood](#heading-bonus-function-types-under-the-hood)
    
* [Quiz time](#heading-quizz-time)
    

<sub>🔖 I created this Table of Contents using </sub> [<sub>BitDownToc</sub>](https://derlin.github.io/bitdowntoc/)<sub>.</sub>

Previous articles in the [Series](https://blog.derlin.ch/series/kotlin-is-fun):

1. [Some cool stuff about functions](https://blog.derlin.ch/kotlin-is-fun-some-cool-stuff-about-kotlin-functions)
    

---

## Function types

A function type is a special notation Kotlin uses to represent a function - basically its signature - that can be used to declare a function (e.g. as a parameter to another function) or as the type of a variable holding a function reference. It looks like this:

```kotlin
(TypeArg1, TypeArg2, ...) -> ReturnType
```

You put the type(s) of the parameter(s) in the left-side parenthesis and the return type on the right side. A function type that doesn't return anything must use the return type `Unit`. For example:

* a function without arguments that doesn't return anything: `() -> Unit`
    
* a function with two string arguments that returns a boolean: `(String, String) -> Boolean`
    

Now, can you guess this one?

```kotlin
(String) -> (Int) -> Boolean
```

This looks confusing, right? It would be clearer if I rewrite it like this (this is equivalent):

```kotlin
(String) -> ((Int) -> Boolean)
```

This function type simply denotes a function that takes a `String` as a parameter and returns another function, this time taking an `Int` as a parameter and returning a `Boolean`.

### Instantiating function types

Function types can be instantiated in many ways.

Let's take a variable declared with the following function type: `(String) -> Boolean` (something taking a `String` as an argument, and returning a `Boolean`). To initialize such a variable, we could use:

* a *lambda expression* (often just called a *lambda*), expressed with curly braces:
    
    ```kotlin
    // lambda with explicit type
    val lambdaE: (String) -> Boolean = { s -> s != null }
    
    // lambda with explicit type and implicit parameter
    val lambdaE: (String) -> Boolean = { it != null }
    
    // lambda with implicit type
    // (as anything can be null, s's type must be declared explicitly)
    val lambdaI = { s: String -> s != null }
    ```
    
* an *anonymous function*, that is a function without any explicit name:
    
    ```kotlin
    val anon = fun(s: String): Boolean = s != null
    ```
    
* a *reference* to an existing function:
    
    ```kotlin
    // a "normal" function
    fun isNotNull(s: String): Boolean = s != null
    
    // a ref to the function above
    val ref1 = ::isNotNull
    // a ref to an existing String function
    val ref2 = String::isNotBlank
    ```
    

*Note*: lambdas and anonymous functions are known as [**function literals**](https://kotlinlang.org/docs/lambdas.html#lambda-expressions-and-anonymous-functions) - functions that are not *declared* but are passed immediately as an expression.

Once a function type is instantiated, it can be called (=*invoked*), or passed around, for example to other functions.

### Invoking a function type

A function type can be called using its method `invoke`, or more conveniently using the famous `()`:

```kotlin
val printHelloLambda = { println("hello !") }

printHelloLambda()
printHelloLambda.invoke()
```

Those calls can even be chained. For example:

```kotlin
// hint: (String) -> ((String) -> Unit)
val doubleLambda: (String) -> (String) -> Unit = { s1 ->
 { s2 -> println("$s1, $s2!") }
}

doubleLambda("Hello")("World") // prints "Hello, World!"
```

In short, however the function type has been instantiated, it behaves like a regular function.

## Higher-order functions

Higher-order functions are just functions that have one or more parameters and/or return types that are function types. In other words, they are functions that receive other functions as parameters or return other functions.

Here is a useless higher-order function:

```kotlin
// a simple function
fun myName(): String = "derlin"

// a higher order function
fun higherOrderFunc(getName: () -> String) {
    println("name is " + getName())
}

higherOrderFunc(::myName) // -> "My name is derlin"
```

Let's use a more interesting example. The following is a higher-order function that filters items from a list based on a condition (passed as a parameter) and also prints the items that were dropped to stdout:

```kotlin
fun evinceAndPrint(
  lst: List<String>, 
  condition: (String) -> Boolean
): List<String> {
    val (keep, drop) = lst.partition { condition(it) }
    println("Evinced items: $drop")
    return keep
}
```

*Note*: this function could work on any kind of list, not just strings, so it would better be **generic**.

```kotlin
fun <T> evinceAndPrint(
  lst: List<T>, 
  condition: (T) -> Boolean
): List<T> {
    val (keep, drop) = lst.partition { condition(it) }
    println("Evinced items: $drop")
    return keep
}
```

Here is how we could use it:

```kotlin
val mixedCase = listOf("hello", "FOO", "world", "bAr")
evinceAndPrint(mixedCase, { s -> s == s.lowercase() })
// > Evinced items: [FOO, bAr]
```

Note, however, that this syntax is heavy (and ugly!). Fortunately, we can make it better. If you are using IntelliJ IDE, you should get two suggestions:

1. move the [trailing lambda](https://kotlinlang.org/docs/lambdas.html#passing-trailing-lambdas) out of the parentheses:
    
    > *According to Kotlin convention, if the last parameter of a function is a function, then a lambda expression passed as the corresponding argument <s>can</s>* ***should*** *be placed outside the parentheses*
    
2. Use the [implicit name `it`](https://kotlinlang.org/docs/lambdas.html#it-implicit-name-of-a-single-parameter) for the single parameter
    
    > *If the compiler can parse the signature without any parameters, the parameter does not need to be declared and* `->` can be omitted. The parameter will be implicitly declared under the name `it`.
    

The call can then be rewritten as:

```kotlin
evinceAndPrint(mixedCase) { it == it.lowercase() }
```

And this is how you end up with so many constructs like:

```kotlin
listOf(1, 2, 3)
  .filter { it % 2 == 0 }
  .forEach { println(it) }
```

`filter` and `forEach` are simply higher-order functions with a single parameter and a trailing lambda! This is way easier to read than:

```kotlin
listOf(1, 2, 3)
  .filter (fun(i: Int): Boolean = i % 2 == 0)
  .forEach ({ i -> println(i) })
```

## Bonus: function types under the hood

As explained in [function-types.md](http://function-types.md), function types are implemented as interfaces:

```kotlin
package kotlin.jvm.functions

interface Function1<in P1, out R> : kotlin.Function<R> {
    fun invoke(p1: P1): R // <- inherits from Function
}
```

These interfaces are named `FunctionN`, where `N` denotes the number of arguments. They all inherit from `kotlin.Function`, which defines the `invoke` method.

When you instantiate a function type (through lambdas or other means), you are thus actually creating an instance of one of those functional interfaces (thus callable from Java).

Why is this interesting to know? Well, as you may have guessed, the Kotlin team didn't write an infinity of those interfaces. They settled for 23, going from `Function0` to `Function22`. In other words, a ***function type (and thus a lambda) can only have up to 22 parameters***.

## Quiz time

Just for fun: what does this function do and how would you invoke (use) it?

```kotlin
fun <T, R, S> x(a: (T) -> R, b: (R) -> S): (T) -> S = 
  { t: T -> b(a(t)) }
```

This function is the well-known *function composition* in the functional paradigm, which takes two functions `A -> B` and `B -> C` and returns a function `A -> C`.

Here is an example:

```kotlin
val trimAndParseInt = x(String::trim, String::toIntOrNull)

listOf("1 ", "  asdf", " 100")
   .mapNotNull(trimeAndParseInt)
   .let(::println)
// > prints [1, 100]
```

Written directly, `trimAndParseInt` is equivalent to `parse(trim(s))`:

```kotlin
// first argument
val trim: (String) -> String = String::trim
// second argument
val parse: (String) -> Int? = String::toIntOrNull

// what the x function body does
val trimAndParseInt: (String) -> Int? = 
  { s -> parse(trim(s)) }
```

And that concludes our second article in the series! This was the most theoretical of all. Stay tuned to learn one of my favourite features of Kotlin: extension functions.