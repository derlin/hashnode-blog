---
title: "Kotlin is `fun` - Some cool stuff about Kotlin functions"
seoDescription: "The first article of a series delving into Kotlin functions, starting with some cool stuff about them."
datePublished: Mon Mar 06 2023 13:50:39 GMT+0000 (Coordinated Universal Time)
cuid: clewvqw5v0a8hurnvfiqj62hl
slug: kotlin-is-fun-some-cool-stuff-about-kotlin-functions
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1677740630706/43380a51-2660-457d-a508-c088f149eb77.png
tags: functions, kotlin, kotlin-beginner

---

---

**About the Series**

Let's deep dive into Kotlin functions, from the simple theory to the more advanced concepts of extension functions and lambda receivers. After reading this series, constructs such as the following should not scare you off anymore:

```kotlin
fun <T, R, S> x(a: (T) -> R, b: R.() -> S): (T) -> S =
    { t: T -> a(t).b() }
```

The only thing you need is a basic knowledge of Kotlin - I won't explain the 101. Let's get started!

---

You got me, I am a big fan of Kotlin. Some of the (many) things I love about it are its functional programming support, its compact syntax, and the advanced yet very handy concepts of extension functions, delegates, and more.

Since I won't be able to cover everything offered by the language, let's restrict this (first?) Series to Kotlin functions, starting with some cool stuff about them:

* [A quick reminder (you can skip it)](#heading-a-quick-reminder-you-can-skip-it)
    
* [Optional named arguments](#heading-optional-named-arguments)
    
* [Single expression functions](#heading-single-expression-functions)
    
* [Local functions](#heading-local-functions)
    

---

### A quick reminder (you can skip it)

For those catching up, a function in Kotlin is declared using the `fun` parameter, a function name, optional arguments in parenthesis written `<name>: <type>`, followed by a `:` and an optional return type.

For example:

```kotlin

fun printHello() {
  println("Hello, World!")
}

printHello() // => writes the string to stdout
```

Or:

```kotlin
fun ternary(ifTrue: String, ifFalse: String, test: Boolean): String {
    return if (test) ifTrue else ifFalse
}

ternary("yes", "no", 2 == 1) // => "yes"
```

Functions can, of course, have modifiers (`internal`, `private`, `inline`, etc), but no `static`! "Static" methods are methods declared in [`object`](https://kotlinlang.org/docs/object-declarations.html#object-declarations-overview).

---

### Optional named arguments

Kotlin supports both optional (with defaults) and named arguments, which removes most of the need for the ugly [function overloading](https://stackabuse.com/guide-to-overloading-methods-in-java/) pattern (java).

Take this function:

```kotlin
fun reformat(
    str: String,
    normalize: Boolean = true,
    useTitleCase: Boolean = true,
    wordSeparator: Char = ' ',
) { /*...*/ }
```

It declares one required argument of type `String`, and multiple optional configuration parameters. Note that optional arguments always come *after* required ones.

All the following calls are perfectly valid:

```kotlin
// ↓↓ preferred
reformat("use defaults")
reformat("change useTitleCase only", useTitleCase = false)

// ↓↓ ugly, but works!
reformat(str = "use defaults")
reformat(useTitleCase = false, str = "changing order")
reformat("all params no names", false, true, '-')
reformat(
    "mix",
    false,
    useTitleCase = false,
    '_' // if all params are present, we can omit the name
)
```

So optional named arguments let the caller pick which parameters he wants to specify, leaving the rest to the default implementation. While a lot of syntaxes are allowed, I would advise you to:

* use the same order in the call as in the declaration
    
* always use named arguments for optional parameters
    
* prefer named arguments when there are many parameters or multiple parameters of the same type.
    

This works the same way for constructors, which are also functions!

```kotlin
data class Person(name: String, hasPet: Boolean = false)

Person("Georges")
Person("Wallace", hasPet = true)
```

### Single expression functions

When a function's body is only a single expression, one can omit the curly braces and the `return` by using `=`.

Hence, this:

```kotlin
fun mod2(x: Int): Int {
    return x % 2
}
```

Can be turned into an elegant single-line declaration:

```kotlin
fun mod2(x: Int): Int = x % 2
```

Kotlin type inference is very good, so in this case, you can even omit the return type (though it shouldn't be taken as a good practice):

```kotlin
fun mod2(x: Int) = x % 2
```

Remember, `if/else` and `try/catch` are expressions too! So this also is valid:

```kotlin
fun trySomething() = try { 
    /*...*/ 
  } catch { 
    /*...*/ 
  }

fun conditional() = if (something) a() else b()
```

I usually abuse this feature, as chaining lets you write many (any ?) complex logics into a single statement in Kotlin. Here is one example from [goodreads-metadata-fetcher](https://github.com/derlin/goodreads-metadata-fetcher/blob/92ec6c0ae13f75243fc1857a15b655aa4450591e/src/main/kotlin/ch/derlin/grmetafetcher/details.kt#L134), where I extract a specific text from an HTML document and convert it to int:

```kotlin
fun getNumberOfPages(doc: Document): Int? =
    doc.getElementById("details")
        ?.getElementsByAttribute("itemprop")
        ?.find { it.attr("itemprop") == "numberOfPages" }
        ?.text()?.split(" ")?.first()
        ?.toInt()
```

In case you are wondering, the `?.` is the [safe call operator](https://kotlinlang.org/docs/null-safety.html#safe-calls).

### Local functions

Functions can be defined inside other functions. In this case, the inner function is only available from inside the outer function and can access the local variables of the outer scope. This is called a [*closure*](https://kotlinlang.org/docs/lambdas.html#closures).

Here is a good example of a local function:

```kotlin
fun fuzzyCompare(expected: String, actual: String): Boolean {
    fun clean(s: String) = s
        .lowercase()
        .replace("[^a-z0-9]".toRegex(), "")
        .replace("\\s+".toRegex(), " ")
        .trim()

    return clean(actual) == clean(expected)
}
```

The `clean` function here is very specific to the fuzzy compare functionality. Instead of making it a private utility function in the same file, we can directly encapsulate it inside the fuzzy compare function itself, making the code both clear and clean.

Even better, Kotlin offers *mutable closures*! Thus, contrary to Java, it is possible to mutate an outer variable from inside a local function (or a lambda). Here is an example:

```kotlin
fun sumList(list: List<Int>): List<Int> {
    var sum = 0
    fun add(i: Int): Int { 
      sum += i; return sum 
    }
    return list.map { add(it) }
}

println(sumList(listOf(1, 2, 3))) // [1,3,6]
```

*Note*: this is a very bad example, as the same can be achieved using functional programming built-ins:

```kotlin
list.runningFold(0, Int::plus).drop(1)
```

If you are interested in how local functions are implemented under the hood (or their cost), see [Idiomatic Kotlin: Local functions](https://medium.com/tompee/idiomatic-kotlin-local-functions-4421f86ac864%5D) from Tompee Balauag.

---

That's it for this first article, which only scraped the surface. The next one will be more theoretical, giving you the conceptual overview necessary to truly understand Kotlin functions, higher-order functions, and lambdas. I will try to keep it light and `fun` though, I promise. Stay tuned!