# Kotlin Concepts: Functional Programming, Delegates, Sealed Classes, Constructors, and Generics

Kotlin becomes much easier to read when each concept has a simple mental model. Today I revised a set of core Kotlin ideas that appear often in Android apps: functional programming, delegates, sealed classes, constructors, constants, generics, and type aliases.

## 1. Functional Programming

Functional programming means writing code by using functions and data transformation instead of changing shared state.

Simple analogy:
Cooking with recipes. Each recipe takes an input, produces an output, and does not secretly modify the original ingredient list.

```kotlin
val result = listOf(1, 2, 3, 4)
    .filter { it % 2 == 0 }
    .map { it * 2 }
```

How to read it:

- `filter` keeps only even numbers.
- `map` multiplies each remaining number.
- Data flows step by step.

Internal idea:
Each step creates new data instead of mutating the original list.

Flow:

```text
input -> filter -> map -> result
```

Android use:
Functional style appears often in `Flow`, coroutines, and MVVM transformations.

Common mistake:
Functional programming does not mean "no classes." Kotlin uses both functional programming and object-oriented programming.

## 2. OOP vs. FP

OOP focuses on objects and state.
FP focuses on functions and transformations.

Simple analogy:

- OOP is like restaurant staff. Each person has responsibilities and state.
- FP is like a cooking pipeline. Ingredients move through transformation steps.

| OOP | FP |
| --- | --- |
| Objects | Functions |
| Often mutable | Often immutable |
| State changes over time | Data is transformed step by step |

## 3. Delegates

A delegate means letting another object handle work for a property.

Simple analogy:
A manager assigns work to an employee.

```kotlin
var name by NameDelegate()
```

How to read it:

- `name` is not a normal backing variable.
- The delegate controls how the value is read and written.

Internal idea:

```text
read name  -> delegate.getValue()
write name -> delegate.setValue()
```

Android use:
Delegates are useful with `SharedPreferences`, ViewModels, and Compose state.

Common mistake:
Do not assume the variable itself stores the value. The delegate may be storing or computing it.

## 4. The `by` Keyword

The `by` keyword means outsourcing work.

Simple analogy:
Hiring a driver instead of driving yourself.

```kotlin
val x by delegate
```

Internally, property access becomes delegate access:

```text
get() = delegate.getValue()
```

Flow:

```text
access -> delegate called
assign -> delegate called
```

## 5. Built-In Delegates

Kotlin includes useful built-in delegates.

### `lazy`

`lazy` creates a value only when it is first used.

```kotlin
val adapter by lazy {
    OrdersAdapter()
}
```

Internal idea:

```text
first access -> compute
next access  -> return cached value
```

Important:
`lazy` does not update automatically. It computes once, then reuses the cached value.

Android use:
`lazy` is useful for adapters, bindings, and objects that should be created only when needed.

### `observable`

`observable` runs code whenever a value changes.

```kotlin
var username by Delegates.observable("") { _, old, new ->
    println("Changed from $old to $new")
}
```

Android use:
It can help connect property changes to UI updates in simple cases.

### `vetoable`

`vetoable` allows or rejects a value change.

```kotlin
var age by Delegates.vetoable(0) { _, _, new ->
    new >= 0
}
```

If the condition returns `false`, the new value is rejected.

## 6. Sealed Class and `when`

A sealed class represents a fixed set of possible types. The compiler knows all direct subclasses.

Simple analogy:
A traffic light has only known states: red, yellow, and green.

```kotlin
sealed class Result {
    data class Success(val data: String) : Result()
    data class Error(val message: String) : Result()
    object Loading : Result()
}
```

How to read it:
`Result` can only be one of its known subclasses.

```kotlin
fun render(result: Result) {
    when (result) {
        is Result.Success -> println(result.data)
        is Result.Error -> println(result.message)
        Result.Loading -> println("Loading")
    }
}
```

Internal idea:
The compiler checks that all cases are handled.

Flow:

```text
when -> match type -> execute branch
```

Android use:
Sealed classes are great for UI states such as loading, success, and error.

Common mistake:
If a case is missing, Kotlin can produce a compile-time error when `when` is used as an expression.

## 7. Primary Constructor

A primary constructor initializes properties directly in the class header.

Simple analogy:
Filling a form once while creating an object.

```kotlin
class User(val name: String)
```

How to read it:
Every `User` object must receive a `name`.

Internal idea:

```text
pass value -> assign property -> create object
```

Android use:
Primary constructors are common in data classes, request models, response models, and UI state classes.

## 8. `const` vs. `val`

`val` is a read-only value assigned at runtime.
`const` is a compile-time constant.

Simple analogy:

- `val` is like salary. It is fixed after assignment, but known at runtime.
- `const` is like gravity in a formula. It is already known before the program runs.

```kotlin
val userName = getUserName()

const val MAX_RETRY_COUNT = 3
```

Internal idea:
For `const`, the compiler can replace usages with the actual value.

Flow:

```text
val   -> computed at runtime
const -> already known at compile time
```

Android use:
Constants are often used for keys, tags, action names, and fixed configuration values.

Common mistake:
`const` cannot be assigned from a function call because it must be known at compile time.

## 9. `init` vs. Constructor

The constructor receives input.
The `init` block runs setup logic.

Simple analogy:

- Constructor: receiving the order.
- `init`: cooking the order.

```kotlin
class User(name: String) {
    init {
        println(name)
    }
}
```

How to read it:
The `init` block runs immediately when the object is created.

Flow:

```text
create object -> constructor input received -> init runs
```

Android use:
`init` is useful for validation or setup logic in classes and ViewModels.

## 10. Generics

Generics let us write code once and use it with different types.

Simple analogy:
A box that can hold anything, while still knowing what type is inside.

```kotlin
class Box<T>(val value: T)
```

How to read it:
`T` is a placeholder type.

```kotlin
val intBox = Box(10)
val stringBox = Box("Kotlin")
```

Internal idea:
Generics give compile-time type checking. On the JVM, generic type information is usually erased at runtime.

Flow:

```text
write generic code -> compiler checks type -> runtime uses erased type
```

Android use:
Generics appear in Retrofit responses, adapters, repositories, and reusable UI/state wrappers.

Common mistake:
Do not assume normal generic type information always exists at runtime.

## 11. `typealias`

`typealias` creates a shorter name for a complex type.

Simple analogy:
A nickname for a long name.

```kotlin
typealias UserMap = Map<String, List<Int>>
```

How to read it:
`UserMap` means `Map<String, List<Int>>`.

Internal idea:
The compiler treats it as the original type. It does not create a new runtime type.

Android use:
`typealias` is useful for callbacks, listener types, and long nested generic types.

Common mistake:
`typealias` is not a new type. It is only another name for an existing type.

## Final Takeaway

These Kotlin concepts become easier when read as data flow and responsibility:

- Functional programming transforms data.
- OOP organizes behavior around objects and state.
- Delegates outsource property behavior.
- Sealed classes model fixed possibilities.
- Constructors and `init` separate input from setup logic.
- Generics make reusable type-safe code.
- `typealias` makes complex types easier to read.
