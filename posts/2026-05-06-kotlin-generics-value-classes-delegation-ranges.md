# Kotlin Concepts: Generics Variance, Value Classes, Delegation, Ranges, and More

Today I revised a set of Kotlin concepts that show up often while reading Android code: generic variance, value classes, property delegation, function references, ranges, closures, extension properties, sealed interfaces, and function composition.

## 1. Generics Variance: Invariant, Covariant, and Contravariant

Simple meaning:

- Invariant means exact type only.
- `out` means read-only producer.
- `in` means write-only consumer.

Real-world analogy:

- A box can both give and take items, so it is invariant.
- A juice machine gives juice, so it is `out`.
- A dustbin takes things, so it is `in`.

```kotlin
class Box<T>(var value: T)           // Invariant
class Producer<out T>(val value: T)  // Covariant
class Consumer<in T>                 // Contravariant
```

How to read the code:

- `T` is the generic type.
- `out T` can be returned safely.
- `in T` can be accepted safely.

Internal idea:
The compiler restricts unsafe operations.

- `out` prevents unsafe writes.
- `in` prevents unsafe reads.

Flow:

```text
Producer -> gives safe values
Consumer -> accepts safe values
```

Android use:

- `LiveData<out T>` exposes values.
- `Comparator<in T>` consumes values to compare them.
- `MutableList<T>` is invariant because it supports both reading and writing.

Common mistakes:

- Thinking `List<String>` can always be treated like `List<Any>`.
- Adding `in` or `out` without checking whether the type is produced or consumed.

## 2. Inline Value Classes

Simple meaning:
An inline value class is a wrapper without extra object memory in many common cases.

Analogy:
A label on a primitive value.

```kotlin
@JvmInline
value class UserId(val id: Int)
```

How to read the code:

- `UserId` wraps one value.
- The wrapper gives type safety.
- The runtime representation can use the underlying value directly.

Internal idea:
The compiler can compile `UserId` down to its underlying `Int` value.

Flow:

```text
UserId(101) -> 101
```

Android use:
IDs become safer because `UserId` and `OrderId` cannot be mixed accidentally.

Common mistakes:

- Adding multiple properties.
- Expecting normal object identity.

## 3. Property Delegation

Simple meaning:
Property delegation outsources property logic to another object.

Analogy:
A manager handles the work for you.

```kotlin
val name by lazy { "Vishruth" }
```

How to read the code:

- `by` means the property uses a delegate.
- `lazy` computes the value once.

Internal idea:
Kotlin calls `getValue()` behind the scenes.

Flow:

```text
first access -> compute
next access  -> reuse
```

Android use:

- `by viewModels()`
- Lazy database initialization

Common mistake:
Thinking `lazy` runs immediately. It runs only on first access.

## 4. Function References: `::`

Simple meaning:
A function reference lets us pass a function as a value.

Analogy:
Giving instructions instead of doing the work immediately.

```kotlin
val ref = ::greet
```

How to read the code:
`::greet` references the function. It does not call it.

Internal idea:
The reference can be used like a lambda.

Flow:

```text
call ref -> original function runs
```

Android use:

```kotlin
button.setOnClickListener(::handleClick)
```

Common mistake:
Writing `::greet()` as if the function reference itself should be called.

## 5. `downTo`

Simple meaning:
`downTo` creates a descending range.

Analogy:
A countdown.

```kotlin
for (i in 5 downTo 1) {
    println(i)
}
```

How to read the code:
Start at `5` and move backward to `1`.

Internal idea:
The range uses a negative step.

Flow:

```text
5 -> 4 -> 3 -> 2 -> 1
```

Android use:
Countdown timers and reverse loops.

Common mistake:
Expecting the loop to run in ascending order.

## 6. Closures

Simple meaning:
A closure is a function that remembers variables from its outer scope.

Analogy:
A backpack carrying data.

```kotlin
var count = 0
val inc = { count++ }
```

How to read the code:
The lambda uses `count`, even though `count` was declared outside the lambda.

Internal idea:
The outer variable is captured, so the same variable can be updated every time.

Flow:

```text
call inc -> update same count variable
```

Android use:
Closures are common in click listeners, callbacks, and coroutine blocks.

Common mistake:
Thinking the value is copied into the lambda every time.

## 7. `until`

Simple meaning:
`until` creates a range that excludes the end value.

Analogy:
Entry is allowed until the gate, but not including the gate.

```kotlin
for (i in 1 until 5) {
    println(i)
}
```

How to read the code:
The loop runs from `1` to `4`. The value `5` is excluded.

Internal idea:
`1 until 5` behaves like `1..4`.

Flow:

```text
1 -> 2 -> 3 -> 4
```

Android use:
RecyclerView loops and index-based iteration.

Common mistake:
Expecting the end value to be included.

## 8. Extension Properties

Simple meaning:
Extension properties make an existing class look like it has a new property.

Analogy:
Attaching a new feature without modifying the original class.

```kotlin
val String.lastChar: Char
    get() = this[length - 1]
```

How to read the code:
This adds a readable `lastChar` property to `String`.

Internal idea:
The extension property is compiled into a generated getter function.

Flow:

```text
"Kotlin".lastChar -> generated getter -> 'n'
```

Android use:

```kotlin
val View.isVisible: Boolean
    get() = visibility == View.VISIBLE
```

Common mistake:
Thinking an extension property stores new data inside the original object.

## 9. Sealed Interfaces

Simple meaning:
A sealed interface defines a restricted type hierarchy.

Analogy:
A closed system with no outside cases.

```kotlin
sealed interface Result
```

How to read the code:
Only known types are allowed to implement `Result` directly.

Internal idea:
The compiler knows the possible result types and can check exhaustive `when` expressions.

Flow:

```text
when -> match known type -> run branch
```

Android use:
Sealed interfaces are useful for UI states, API results, and one-time screen events.

Common mistake:
Trying to implement a sealed interface from a place Kotlin does not allow.

## 10. Function Composition

Simple meaning:
Function composition combines functions so one output becomes the next input.

Analogy:
An assembly line.

```kotlin
val result = multiply(add(5))
```

How to read the code:
`add(5)` runs first. Its output is passed to `multiply`.

Internal idea:
Functions can be treated as values and connected together.

Flow:

```text
5 -> add -> multiply -> result
```

Android use:
Function composition appears in `Flow`, Rx chains, and transformation pipelines.

Common mistake:
Reading the order from left to right when nested calls execute from the inside out.
