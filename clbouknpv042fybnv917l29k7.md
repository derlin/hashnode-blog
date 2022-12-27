# Kotlin: pretty print data classes

Working on a little new [library to fetch book metadata from GoodReads](https://github.com/derlin/goodreads-metadata-fetcher), I stumbled upon a basic, yet tricky need/problem: pretty-printing data classes in Kotlin.

**NOTE**: I am not the only one wishing this feature to be available ↦ https://discuss.kotlinlang.org/t/pretty-print-data-class/19345

## What I wanted

Kotlin data classes have many advantages, one of which being the automatic override of `toString` in an already nice format. For example, given a simple data class:

```kotlin
data class Person(
    val name: String,
    val dateOfBirth: LocalDate,
    val married: Boolean = false,
    val children: List<String> = listOf(),
    val n: Int? = null
)
```

the output of `toString()` would be for example:

```plaintext
Person(name=Dani Chou, dateOfBirth=1988-06-13, married=false, children=[Max, Leo, Soan], n=1)
```

However, to make it more readable (especially when the data class has more properties), I wanted something like:

```plaintext
Person(
  name="Dani Chou",
  dateOfBirth="1988-06-13",
  married=false,
  children=["Max", "Leo", "Sam"],
)
```

The requirements were, in order of importance:

1.  have each property on a new line (preferably with indent),
    
2.  have strings quoted,
    
3.  *nice-to-have* keep the data class property definition order,
    
4.  *nice-to-have* have compilable text, that can be copy-pasted as is in a kotlin file.
    

## Solutions found online (not retained)

From the different threads and searches, three solutions usually popped up: regexes, JSON serialization, and the [pretty-print - pp](https://github.com/snowe2010/pretty-print) library.

### Use regexes

The first basic answer is usually to use some `replace` on the `toString` output. The easiest I could do was:

```kotlin
fun Any.toPrettyString(indentSize: Int = 2) = " ".repeat(indentSize).let { indent ->
    toString()
        .replace(", ", ",\n$indent")
        .replace("(", "(\n$indent")
        .dropLast(1) + "\n)"
}
```

It works quite well when we have empty `children`:

```plaintext
Person(
  name=Dani Chou,
  dateOfBirth=1988-06-13,
  married=false,
  children=[],
  n=1
)
```

But fails miserably once we have non-empty lists, or if any property contains `,` and/or parentheses (for example `name="Dani, Chou (nickname)"`).

I tried other regex-based approaches, but there is always a catch: what if a string contains `prop=value`, or other characters matching perfectly our patterns?

### JSON serialization

Another possibility is to actually serialize the data class into JSON (or another format). The output would be:

```json
{
  "name": "Dani Chou",
  "dateOfBirth": {
    "year": 1988,
    "month": 6,
    "day": 13
  },
  "married": false,
  "children": [
    "Max",
    "Leo",
    "Soan"
  ],
  "n": 1
}
```

This output is nice, but only satisfies requirements 1-2. Moreover, it has some downsides:

*   the name of the data class is missing,
    
*   depending on the library used, the order of properties in the output will vary,
    
*   we are far from the original `toString` output,
    
*   lists are pretty printed as well (which is not bad *per se*, but didn't match my tastes on my specific use case).
    

Moreover, this requires more setup and dependencies, even though my project doesn't need JSON at all.

Lots of libraries are available for JSON serialization, including [jackson](https://www.baeldung.com/kotlin/jackson-kotlin) and [gson](https://www.baeldung.com/kotlin/json-convert-data-class) (both very straightforward).

I ended up trying **kotlinx.serialization**, as it is maintained by the core kotlin team. And gosh, this was hard. After finally understanding how to configure it properly, I annotated my data class with `@Serializable` and got a beautiful exception. The problem? There is no built-in support for `LocalDate`...

For posterity, here is the serializer I had to implement:

```kotlin
object LocalDateSerializer : KSerializer<LocalDate> {
    override val descriptor: SerialDescriptor = PrimitiveSerialDescriptor("LocalDate", PrimitiveKind.STRING)
    override fun serialize(encoder: Encoder, value: LocalDate) = encoder.encodeString(value.toString())
    override fun deserialize(decoder: Decoder): LocalDate = LocalDate.parse(decoder.decodeString())
}
```

The `dateOfBirth` would also need to be annotated with the following to work:

```kotlin
@Serializable(with = LocalDateSerializer::class)
```

Note also that contrary to GSON (used in the example above), kotlinx won't preserve the data class properties order.

### pretty print - pp library

This [post on Reddit](https://www.reddit.com/r/Kotlin/comments/apip8o/i_wrote_a_library_for_pretty_printing_objects/) made me aware of a library that seems to provide *exactly* what I need: https://github.com/snowe2010/pretty-print. It is very lightweight and depends solely on `kotlin-reflect`.

However, this didn't work for me, primarily because of `LocalDate`. Using `pp()`:

```plaintext
Person(
  name = "Dani Chou"
  dateOfBirth = LocalDate(
    MIN = LocalDate.<static cyclic class reference>
    MAX = LocalDate.<static cyclic class reference>
    EPOCH = LocalDate.<static cyclic class reference>
    serialVersionUID = 2942565459149668126
    DAYS_PER_CYCLE = 146097
    DAYS_0000_TO_1970 = 719528
    year = 1988
    month = 6
    day = 13
  )
  married = false
  children = [
               "Max",
               "Leo",
               "Soan"
             ]
  n = 1
)
```

I also didn't like the way lists were shown, with one item per line and a big indent.

## My solution

I ended up writing my own little helpers. Those are not for general use, and only cover my needs:

*   simple data classes (no nesting),
    
*   only basic types except `LocalDate`,
    
*   support for lists only (with the same basic types support).
    

It depends solely on `kotlin.reflect`, which needs to be imported explicitly in the project.

The result is a *perfectly valid* kotlin code:

```plaintext
Person(
  name="Dani Chou",
  dateOfBirth=LocalDate.parse("1988-06-13"),
  married=false,
  children=listOf("Max", "Leo", "Soan"),
  n=1,
)
```

Here is the code, which is easily extensible to other types:

```kotlin
import java.time.LocalDate
import kotlin.reflect.KClass
import kotlin.reflect.full.declaredMemberProperties

@Suppress("UNCHECKED_CAST")
internal fun <T : Any> ppDataClass(data: T, indent: Int = 2): String {
    val klass = data::class as KClass<T>
    // store all declared properties in a map as key=value
    val propsInObject = klass.declaredMemberProperties
        .associate { it.name to it.get(data) }
    // extract the properties present in toString(), preserving order
    val orderedPropsInToString = "([A-Za-z0-9_]+)=".toRegex()
        .findAll(data.toString()).map { it.groupValues[1] }

    return with(StringBuilder()) {
        val spaces = " ".repeat(indent) // indent
        // start with ClassName(
        appendLine(klass.simpleName + "(")
        // all toString properties (ordered) and their formatted values
        orderedPropsInToString.forEach { propName ->
            val value = ppValue(propsInObject[propName])
            appendLine("""$spaces$propName=$value,""")
        }
        appendLine(")") // close the class name
        toString()
    }
}

private fun ppValue(value: Any?): String {
    if (value == null) return "null"
    return when (value) {
        is String -> "\"$value\""
        is LocalDate -> "LocalDate.parse(\"$value\")"
        is List<*> -> "listOf(" + value.joinToString(", ") { ppValue(it) } + ")"
        else -> "$value"
    }
}
```

The only little trick worth mentioning is that I first call `toString()` on the data class, and extract all properties present in the output. The goal is two-fold:

1.  this allows me to preserve the data class properties declaration order, whereas `kotlin.reflect` returns properties in alphabetical order,
    
2.  this allows for a data class to omit some properties, by changing the `toString` method.
    

In [GoodReads Metadata Fetcher](https://github.com/derlin/goodreads-metadata-fetcher), all my data classes offer a `toCompilableString` method, which internally calls the `ppDataClass` above.

## Conclusion

This simple problem got me on an interesting journey that I wanted to share. Even though pretty-printing data classes may seem basic, there is currently no tool perfectly tailored for the task.

It is possible to come up with very nice (though limited) solutions using the power of `kotlin.reflect`, but as the pretty-print library showed, making those solutions truly generic is very difficult, and will always miss some edge cases (e.g. `LocalDate`).

* * *

Written with ❤ by [derlin](https://github.com/derlin)