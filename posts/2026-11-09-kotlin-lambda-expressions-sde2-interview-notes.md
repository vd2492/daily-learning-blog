# Kotlin Lambda Expressions: SDE 2 Interview Notes

Today I revised Kotlin lambda expressions end to end: fundamentals, function types, syntax shortcuts, higher-order functions, closures, lambdas with receiver, scope functions, returns, `inline`/`noinline`/`crossinline`/`reified`, SAM conversion, runtime cost, and the interview questions that keep coming up.

## 1. Fundamentals

Simple meaning:
A lambda is an anonymous function literal: a function with no name that you can store, pass, and return like any other value.

One-line interview definition:
"A lambda is a function literal. Kotlin functions are first-class, so a lambda can be stored in a variable, passed as an argument, or returned from a function."

Why it exists:

- Less boilerplate than anonymous classes.
- Enables functional style (`map`, `filter`).
- Makes callbacks and DSLs possible.

```kotlin
val name: (ParamTypes) -> ReturnType = { params -> body }
val add: (Int, Int) -> Int = { a, b -> a + b }
```

How to read the code:
"`add` is of type function taking two Ints and returning an Int; its value is the lambda that names the inputs `a`, `b` and returns `a + b`."

Symbols:

- `:` means "is of type" (name on the left, type on the right).
- `->` means "returns" or "goes to" (inputs on the left, output or body on the right).
- `=` means "is" (assigns the value).
- `{ }` marks the lambda itself.

Rules to remember:

- Always wrapped in curly braces.
- The last expression is the return value; no `return` keyword needed.
- Parameter types can be omitted when Kotlin can infer them.
- Function type parameters always need parentheses: `(Int) -> Int`, never `Int -> Int`.
- `val` holding a lambda means the variable cannot be reassigned; it can still be called any number of times.

Two equivalent forms:

```kotlin
val add: (Int, Int) -> Int = { a, b -> a + b }   // type on the variable
val add = { a: Int, b: Int -> a + b }            // types inside, inferred
```

Calling it: `add(2, 3)` or `add.invoke(2, 3)` both return `5`.

## 2. Function Types

A function type describes a function's inputs and output: `(InputTypes) -> OutputType`.

| Type | Read as | Example value |
| --- | --- | --- |
| `() -> Unit` | no input, returns nothing | `{ println("hi") }` |
| `(Int) -> Int` | one Int in, Int out | `{ it * 2 }` |
| `(Int, Int) -> Int` | two Ints in, Int out | `{ a, b -> a + b }` |
| `(String) -> Boolean` | String in, Boolean out | `{ it.isEmpty() }` |
| `(() -> Unit)?` | a function OR null (note the extra parentheses) | `null` |
| `() -> Unit?` | a function that returns `Unit?` (different from above) | `{ null }` |
| `String.(Int) -> String` | called on a String, takes an Int, returns a String (receiver) | `{ this.repeat(it) }` |
| `(Int) -> (Int) -> Int` | takes an Int, returns a function `(Int) -> Int` | `{ a -> { b -> a + b } }` |
| `suspend () -> Unit` | a suspending function type (coroutines) | `{ delay(100) }` |

Notes:

- Named parameters in a type are allowed for documentation only: `(x: Int, y: Int) -> Int`.
- Type alias shortens long types: `typealias Op = (Int, Int) -> Int`.
- `->` is right-associative: `(Int) -> (Int) -> Int` means `(Int) -> ((Int) -> Int)`.
- Receiver vs parameter: `A.(B) -> C` and `(A, B) -> C` are interchangeable when passing values; the only difference is `this` vs a named parameter inside the lambda.
- On the JVM, a function type compiles to an interface `Function0`, `Function1`, `Function2`, and so on (number = parameter count), each with an `invoke` method.

## 3. Syntax Shortcuts

Five conventions make lambdas short, and interviewers expect each by name.

**1. `it`: implicit single parameter.** When a lambda has exactly one parameter, you may skip declaring it.

```kotlin
val square: (Int) -> Int = { it * it }   // same as { x -> x * x }
```

**2. Trailing lambda.** If the last parameter is a function, the lambda can go outside the parentheses. If it is the only argument, drop the parentheses.

```kotlin
list.filter({ it > 2 })   // valid, not idiomatic
list.filter() { it > 2 }
list.filter { it > 2 }    // idiomatic
```

**3. Last expression is the result.** Multi-line lambdas return their final line.

```kotlin
val describe = { n: Int ->
    val type = if (n % 2 == 0) "even" else "odd"
    "$n is $type"   // returned
}
```

**4. `_` for unused parameters.**

```kotlin
map.forEach { (_, value) -> println(value) }
```

**5. Destructuring in parameters.** Works for any type with `componentN()` (data classes, `Pair`, `Map.Entry`).

```kotlin
map.forEach { (key, value) -> println("$key=$value") }
```

Gotchas:

- `{ a, b -> }` takes two parameters; `{ (a, b) -> }` takes ONE parameter and destructures it.
- Nested lambdas both using `it` shadow each other; name the outer parameter for readability.

## 4. Where Lambdas Are Used

Because functions are values, a lambda can go anywhere a value can.

| # | Situation | Example |
| --- | --- | --- |
| 1 | Stored in a variable | `val sq: (Int) -> Int = { it * it }` |
| 2 | Passed as an argument | `list.map { it * 2 }` |
| 3 | Returned from a function | `fun mul(f: Int): (Int) -> Int = { it * f }` |
| 4 | Stored in a collection | `mapOf<String, (Int, Int) -> Int>("add" to { a, b -> a + b })` |
| 5 | Class property / callback | `class Button(val onClick: () -> Unit)` |
| 6 | Default parameter value | `fun greet(fmt: (String) -> String = { "Hi $it" })` |
| 7 | Nullable (optional callback) | `onDone: (() -> Unit)? = null` then `onDone?.invoke()` |
| 8 | Invoked immediately | `{ a: Int, b: Int -> a + b }(3, 4)` or `run { ... }` |
| 9 | With a receiver (DSL) | `buildString { append("hi") }` |
| 10 | In place of a SAM interface | `Runnable { println("run") }` |

Variable vs parameter, same syntax split in two:

- Variable: name, type, and value together: `val op: (Int, Int) -> Int = { x, y -> x + y }`.
- Parameter: only name and type in the declaration (`op: (Int, Int) -> Int`); the lambda value arrives at the call site.

Operation fixed vs operation chosen by the caller:

```kotlin
val add = { x: Int, y: Int -> x + y }                          // operation decided now, values later
fun calculate(a: Int, b: Int, op: (Int, Int) -> Int) = op(a, b) // operation and values decided at the call
calculate(10, 5, add)                                          // they combine
```

## 5. Higher-Order Functions (HOF)

Simple meaning:
A higher-order function takes a function as a parameter, returns a function, or both.

Takes a function:

```kotlin
fun calculate(a: Int, b: Int, op: (Int, Int) -> Int): Int = op(a, b)

calculate(10, 5) { x, y -> x + y }   // 15
calculate(10, 5) { x, y -> x * y }   // 50
```

How to read the header:
"`calculate` takes `a` (Int), `b` (Int) and `op` (a function from two Ints to an Int), and returns an Int, which is `op(a, b)`."

- Three inputs: two numbers and one function.
- `-> Int` inside the parentheses is `op`'s return type; `): Int` after them is `calculate`'s return type.
- `= op(a, b)` is an expression body, shorthand for `{ return op(a, b) }`.

Flow:

```text
10 -> a -> x
5  -> b -> y
lambda -> op -> body calls op(10, 5)
```

Returns a function:

```kotlin
fun multiplier(factor: Int): (Int) -> Int = { it * factor }
val triple = multiplier(3)
triple(5)   // 15
```

Why HOFs matter:
They separate what varies (the lambda) from the fixed algorithm (looping, sorting, retrying, timing).

Standard library HOFs to name in interviews: `map`, `filter`, `forEach`, `fold`, `reduce`, `flatMap`, `groupBy`, `sortedBy`, `any`, `all`, `first`, `associateBy`, `takeWhile`, `let`, `apply`, `repeat`, `use`, `synchronized`, `lazy`.

Real-world example (common SDE 2 ask: "write a retry / timing utility"):

```kotlin
inline fun <T> measure(tag: String, block: () -> T): T {
    val start = System.nanoTime()
    return block().also { println("$tag took ${System.nanoTime() - start} ns") }
}

fun <T> retry(times: Int, block: () -> T): T {
    repeat(times - 1) {
        try { return block() } catch (e: Exception) { /* log */ }
    }
    return block()   // last attempt lets the exception propagate
}
```

## 6. Closures (Capturing Variables)

Simple meaning:
A closure is a lambda that captures variables from the scope where it was created and keeps access to them after that scope ends.

```kotlin
fun counter(): () -> Int {
    var count = 0
    return { ++count }
}

val c = counter()
c(); c()
println(c())   // 3, count lives on inside the lambda
```

Kotlin vs Java (frequent question):

- Java lambdas can capture only effectively final variables.
- Kotlin lambdas can capture and modify `var`s.

Internal idea:
A captured `var` is wrapped in a `Ref` object (`Ref.IntRef`, `Ref.ObjectRef`) so the lambda and the outer scope share one mutable box. Captured `val`s are copied into fields of the lambda object.

Cost:
A capturing lambda allocates a new object on every evaluation; a non-capturing lambda is reused as a singleton instance.

Pitfalls to mention:

- Memory leaks: a lambda that captures an Activity, View, or `this` and is stored in a long-lived object (singleton, static listener) keeps that object alive.
- Thread safety: a captured `var` modified from several threads or coroutines is a shared mutable state race; use `AtomicInteger` or a `Mutex`.
- Stale capture: a lambda captures the variable, not a snapshot, so it sees later changes.

```kotlin
var x = 1
val show = { println(x) }
x = 2
show()   // prints 2, not 1
```

## 7. Lambdas with Receiver and DSLs

Simple meaning:
A lambda with receiver has type `T.() -> R`. Inside it, `this` is a `T`, so you call `T`'s members directly. This is the foundation of Kotlin DSLs.

```kotlin
val shout: String.() -> String = { uppercase() + "!" }   // this = the String
"hi".shout()   // "HI!"
shout("hi")    // also valid: receiver passed as first argument
```

How to read the code:
`String.() -> String` is "a function called on a String that returns a String."

Writing a builder (classic interview task):

```kotlin
class Html {
    private val sb = StringBuilder()
    fun p(text: String) { sb.append("<p>$text</p>") }
    override fun toString() = sb.toString()
}

fun html(init: Html.() -> Unit): Html = Html().apply(init)

val page = html {
    p("Hello")     // this.p(...), this is Html
    p("World")
}
```

Where you see it:
`apply`, `run`, `with`, `buildString`, `buildList`, Gradle Kotlin DSL, Ktor routing, Jetpack Compose (`Column { }` content uses `ColumnScope.() -> Unit`), and coroutine builders (`CoroutineScope.() -> Unit` in `launch`).

Notes:

- `@DslMarker` stops nested DSL blocks from implicitly calling an outer receiver's methods, preventing confusing scope leaks.
- Extension function vs lambda with receiver: same idea (`this` = receiver); an extension function is named and declared once, a lambda with receiver is a value passed at the call site.

## 8. Scope Functions

Simple meaning:
Scope functions are five small inline higher-order functions that run a lambda in the context of an object. They differ on two axes only: how you refer to the object (`it` or `this`) and what they return (the lambda result or the object).

| Function | Object as | Returns | Extension? | Typical use |
| --- | --- | --- | --- | --- |
| `let` | `it` | lambda result | yes | null checks `x?.let { }`, transform a value |
| `run` | `this` | lambda result | yes (also a non-extension form) | configure and compute a result |
| `with` | `this` | lambda result | no: `with(obj) { }` | group calls on an object |
| `apply` | `this` | the object | yes | object initialisation / builder |
| `also` | `it` | the object | yes | side effects: logging, validation |

Signatures (worth memorising, they explain everything):

```kotlin
inline fun <T, R> T.let(block: (T) -> R): R
inline fun <T, R> T.run(block: T.() -> R): R
inline fun <T, R> with(receiver: T, block: T.() -> R): R
inline fun <T> T.apply(block: T.() -> Unit): T
inline fun <T> T.also(block: (T) -> Unit): T
```

Memory trick:
`(T) ->` gives `it`, `T.() ->` gives `this`. Returns `R` = lambda result, returns `T` = the object.

Examples:

```kotlin
val len = name?.let { it.length } ?: 0
val user = User().apply { name = "A"; age = 30 }
val list = mutableListOf(1).also { println("created $it") }
val area = rect.run { width * height }
with(canvas) { drawLine(); drawCircle() }
```

Common follow-ups:

- `apply` vs `also`: both return the object; `apply` uses `this` (configure), `also` uses `it` (side effect).
- `let` vs `run`: both return the result; `let` uses `it`, `run` uses `this`.
- `run` vs `with`: same behaviour; `run` is an extension (`obj.run { }`, works with `?.`), `with` takes the object as an argument.
- Avoid deep nesting: nested `it`/`this` gets ambiguous and hurts readability.

## 9. Returns in Lambdas

Simple meaning:
A bare `return` inside a lambda returns from the enclosing function, not the lambda, and is only allowed when the lambda is passed to an inline function. This is called a non-local return.

| Form | Exits | Allowed when |
| --- | --- | --- |
| last expression | the lambda (normal result) | always |
| `return@label` | the lambda only (like `continue` in `forEach`) | always |
| `return` (bare) | the enclosing function | lambda passed to an inline function, not marked `crossinline` |
| `return` in anonymous function | the anonymous function | always |

```kotlin
fun findFirstEven(list: List<Int>) {
    list.forEach {
        if (it % 2 == 0) {
            println("found $it")
            return            // exits findFirstEven (non-local)
        }
    }
    println("none")           // skipped if an even number is found
}

list.forEach {
    if (it == 2) return@forEach   // skips just this item, loop continues
    println(it)
}

list.forEach loop@{             // custom label
    if (it == 2) return@loop
}
```

Why bare `return` needs `inline`:
After inlining, the lambda body sits inside the caller, so returning from the caller is real code. A non-inline lambda is a separate object that might run later or on another thread, so a return to a finished caller makes no sense.

Notes:

- `break` from `forEach` is not directly possible. Use a regular `for` loop, `run loop@{ list.forEach { if (...) return@loop } }`, or `firstOrNull` / `takeWhile`.
- `val f = { return 5 }` does not compile; use `return@label` or an anonymous function `fun(): Int { return 5 }`.

## 10. `inline`, `noinline`, `crossinline`, `reified`

Simple meaning:
`inline` copies a function's body and its lambda arguments into the call site, removing the lambda object allocation and the virtual `invoke` call. The other three keywords adjust that behaviour. This is the most-asked advanced lambda topic in senior Kotlin and Android rounds.

| Keyword | Applies to | What it does | Use when |
| --- | --- | --- | --- |
| `inline` | function | inlines the body and all lambda params at the call site; enables non-local return | small HOFs called often with lambdas (`map`, `filter`, `let`) |
| `noinline` | one lambda param of an inline fn | that lambda stays a real object; no non-local return | you need to store it, or pass it to a non-inline function |
| `crossinline` | one lambda param of an inline fn | still inlined, but bare `return` is forbidden | the lambda runs in another context (a nested lambda, `Runnable`, another thread) |
| `reified` | type param of an inline fn | keeps `T` available at runtime (`is T`, `T::class`) | generic helpers like `inline fun <reified T> Gson.fromJson(json: String)` |

```kotlin
inline fun runTasks(a: () -> Unit, noinline b: () -> Unit) {
    a()                 // inlined
    saveForLater(b)     // b is an object, can be passed on
}

inline fun onUi(crossinline block: () -> Unit) {
    handler.post { block() }   // called inside another lambda: needs crossinline
}

inline fun <reified T> List<Any>.filterType(): List<T> = filter { it is T }.map { it as T }
listOf(1, "a", 2).filterType<Int>()   // [1, 2]
```

Why lambdas cost something without `inline`:

- Each lambda compiles to a `FunctionN` object.
- Capturing lambdas allocate on every call.
- Calls go through `invoke` (virtual call, boxing of primitives for generic `FunctionN`).

Downsides of `inline`:

- Larger bytecode (body copied at every call site).
- Cannot inline recursive functions.
- Public inline functions cannot use private/internal members (they would leak into callers).
- No benefit without lambda params (the compiler warns).

Why `reified` needs `inline`:
JVM generics are erased at runtime; inlining substitutes the real type at each call site, so `T` is known.

`noinline` vs `crossinline` in one line:
`noinline` = not inlined at all (becomes an object); `crossinline` = inlined, but no non-local return.

## 11. Lambda vs Anonymous Function vs Function Reference

All three produce a value of a function type; they differ in syntax, return behaviour, and return-type declaration.

| | Lambda | Anonymous function | Function reference |
| --- | --- | --- | --- |
| Syntax | `{ x: Int -> x * 2 }` | `fun(x: Int): Int { return x * 2 }` | `::double` |
| Bare `return` | exits the enclosing function (inline only) | exits the anonymous function itself | n/a |
| Explicit return type | no, inferred from last expression | yes, can be declared | from the referenced function |
| Best for | short inline logic | multiple exit points, explicit types | reusing an existing named function |

```kotlin
fun isEven(n: Int) = n % 2 == 0

list.filter { it % 2 == 0 }               // lambda
list.filter(fun(n: Int): Boolean {        // anonymous function
    if (n < 0) return false
    return n % 2 == 0
})
list.filter(::isEven)                     // top-level function reference
list.map(String::length)                  // member reference (receiver becomes the param)
val p = user::greet                       // bound reference to an instance
val make = ::User                         // constructor reference
```

Notes:

- Lambda vs regular function: a `fun` is a named declaration; a lambda is an expression/value. A `fun` can be turned into a value with `::name`.
- Lambda vs Java lambda: Kotlin has real function types (`(Int) -> Int`); Java lambdas only target functional interfaces. Kotlin lambdas can modify captured `var`s; Java's cannot.

## 12. SAM Conversion and `fun interface`

Simple meaning:
SAM (Single Abstract Method) conversion lets you pass a lambda wherever an interface with exactly one abstract method is expected.

Java interfaces: automatic.

```kotlin
val r = Runnable { println("running") }
executor.execute { println("task") }
button.setOnClickListener { view -> /* ... */ }
```

Kotlin interfaces: must be declared `fun interface` (since Kotlin 1.4).

```kotlin
fun interface Validator {
    fun validate(input: String): Boolean
}

val notEmpty = Validator { it.isNotEmpty() }
notEmpty.validate("")   // false
```

`fun interface` vs function type `(String) -> Boolean`:

- `fun interface`: has a name, can have default methods and extra members, clearer in public APIs, Java-friendly.
- Function type: lighter, no declaration needed, composes easily.

Without SAM you would write `object : Validator { override fun validate(input: String) = input.isNotEmpty() }`.

Gotcha:
An `object :` expression creates a new instance each time and has its own `this`; a SAM-converted lambda's `this` refers to the enclosing class.

## 13. Under the Hood and Performance

SDE 2 rounds probe whether you know what a lambda costs at runtime, not just how to write one.

Compilation:
A function type `(A, B) -> R` becomes `Function2<A, B, R>` with `invoke(a, b)`. Since Kotlin 2.0, lambdas are compiled with JVM `invokedynamic` (`LambdaMetafactory`) by default instead of generating a class per lambda.

Allocation:

- Non-capturing lambda: one shared instance, effectively free after the first use.
- Capturing lambda: a new object each time the lambda expression is evaluated.
- Captured `var`: wrapped in a `Ref` box (another allocation).

Boxing:
Generic `FunctionN` takes and returns `Object`, so `(Int) -> Int` boxes primitives on each call unless the call is inlined.

`inline` fixes all three for lambdas passed to inline functions: no object, no `invoke`, no boxing.

Collections vs sequences:
Chained `list.filter { }.map { }` creates an intermediate list per step (eager). `list.asSequence().filter { }.map { }.toList()` processes element by element lazily; prefer it for large data or when you stop early (`first`, `take`). Sequence operations are not inline, so their lambdas are objects.

Other notes:

- Hot paths: avoid creating capturing lambdas inside tight loops or `onDraw`; hoist them or use inline functions.
- Compose: unstable lambdas that capture changing values can cause extra recompositions; `remember { }` a lambda when needed.
- Debugging cost: long lambda chains are harder to step through and give less readable stack traces; balance elegance with debuggability.

## 14. Interview Q&A

**Basics**

1. **What is a lambda?** An anonymous function literal. Kotlin functions are first-class, so a lambda can be stored, passed, and returned like any value.
2. **What is a function type?** A type describing inputs and output, e.g. `(Int, Int) -> Int`. It compiles to `Function2<Int, Int, Int>`.
3. **What does `it` mean?** The implicit name of the single parameter when a lambda has exactly one.
4. **What is a trailing lambda?** If the last parameter is a function, its lambda can be written outside the parentheses: `list.map { it * 2 }`.
5. **What does a lambda return?** The value of its last expression.
6. **What is a higher-order function?** A function that takes a function as a parameter and/or returns one. Examples: `map`, `filter`, `let`.

**Intermediate**

7. **What is a closure?** A lambda that captures variables from its surrounding scope. Kotlin can modify captured `var`s (via `Ref` wrappers); Java requires effectively final.
8. **Difference between `let`, `run`, `with`, `apply`, `also`?** `it` vs `this`, and lambda result vs object. `apply`/`also` return the object; `let`/`run`/`with` return the lambda result.
9. **What is a lambda with receiver?** A lambda of type `T.() -> R` where `this` is a `T`. It powers `apply`, `buildString`, and DSLs.
10. **Lambda vs anonymous function?** An anonymous function can declare its return type, and its `return` exits only itself. A lambda's bare `return` exits the enclosing function and only works inside inline calls.
11. **What is a function reference?** `::name` turns an existing function, property, or constructor into a function-type value.
12. **What is SAM conversion?** Passing a lambda where a single-abstract-method interface is expected. Works for Java interfaces and Kotlin `fun interface`.
13. **How do you exit only the current iteration of `forEach`?** `return@forEach`. For a full break, use a `for` loop or a labeled `run` block.

**Advanced (SDE 2 focus)**

14. **Why use `inline`?** It removes the lambda object allocation, the `invoke` call, and boxing by copying code into the call site, and it allows non-local returns. Use it for small HOFs; avoid it for large bodies.
15. **`noinline` vs `crossinline`?** `noinline`: that lambda is not inlined, becomes an object, can be stored or passed on. `crossinline`: still inlined, but non-local return is banned because it runs in another context.
16. **What is `reified` and why does it need `inline`?** It keeps a generic type available at runtime (`is T`, `T::class`). Generics are erased on the JVM, so only inlining can substitute the real type at each call site.
17. **Why can't a non-inline lambda use a bare `return`?** The lambda is a separate object that may run later or on another thread, after the enclosing function has finished.
18. **What does a lambda cost at runtime?** A `FunctionN` instance (new per evaluation if capturing), a virtual `invoke`, boxing for primitives, and a `Ref` box for captured `var`s. Since Kotlin 2.0, lambdas compile with `invokedynamic`.
19. **How can lambdas leak memory on Android?** A stored lambda that captures an Activity, Fragment, View, or `this` keeps it alive. Fix: clear listeners in `onDestroy`, use lifecycle-aware scopes, avoid capturing context in singletons.
20. **How does Jetpack Compose or a Kotlin DSL use lambdas?** Content parameters are lambdas with receiver (e.g. `ColumnScope.() -> Unit`), so children call scope members like `Modifier.weight` directly; `@DslMarker` limits scope leakage.
21. **`(A) -> B` vs `fun interface`: when would you pick each?** Function types for lightweight internal callbacks; `fun interface` for public or Java-facing APIs needing a name, docs, or default methods.
22. **Sequence vs collection chains?** Collections are eager and allocate an intermediate list per step; sequences are lazy, element by element, better for large inputs or early termination.
23. **What are suspending lambdas?** Lambdas of type `suspend () -> T`, as taken by `launch` and `async`, which can call suspend functions inside.

Coding tasks commonly asked:

- Implement your own `map`, `filter`, or `forEach` as an extension HOF.
- Write a `retry(times) { }` or `measureTime { }` utility.
- Write a debounce / throttle that returns a lambda wrapping another lambda.
- Build a tiny HTML or config DSL with a lambda with receiver.
- Implement `memoize` for `(Int) -> Int`.

```kotlin
inline fun <T, R> List<T>.myMap(transform: (T) -> R): List<R> {
    val result = ArrayList<R>(size)
    for (item in this) result.add(transform(item))
    return result
}

fun memoize(f: (Int) -> Int): (Int) -> Int {
    val cache = HashMap<Int, Int>()
    return { n -> cache.getOrPut(n) { f(n) } }
}
```

## 15. Predict-the-Output Questions

These tricky snippets test returns, captures, and scope functions. Cover the answer and try each one first.

**Q1. Non-local return**

```kotlin
fun test() {
    listOf(1, 2, 3).forEach {
        if (it == 2) return
        print(it)
    }
    print(" done")
}
```

Output: `1`. The bare `return` exits `test()`, so `" done"` never prints.

**Q2. Labeled return**

```kotlin
listOf(1, 2, 3).forEach {
    if (it == 2) return@forEach
    print(it)
}
print(" done")
```

Output: `13 done`. `return@forEach` skips only the current item.

**Q3. Capture sees later changes**

```kotlin
var x = 1
val f = { x }
x = 5
println(f())
```

Output: `5`. The lambda captures the variable, not its value at creation.

**Q4. Lambda defined but not called**

```kotlin
val greet = { println("Hello") }
println("Bye")
```

Output: `Bye`. Defining a lambda does not run it; `greet()` would.

**Q5. Type of `{ 5 }`**

```kotlin
val a = { 5 }
fun b() = { 5 }
```

`a` is `() -> Int`, not `Int`. `b()` returns a lambda; you need `b()()` to get `5`. Classic mistake: writing `fun b() = { 5 }` when you meant `fun b() = 5`.

**Q6. Scope function return values**

```kotlin
val r1 = "hi".let { it.length }
val r2 = "hi".also { it.length }
val r3 = "hi".run { length }
val r4 = "hi".apply { length }
```

`r1 = 2`, `r2 = "hi"`, `r3 = 2`, `r4 = "hi"`. `let`/`run` return the result; `also`/`apply` return the object.

**Q7. Null-safe `let`**

```kotlin
val s: String? = null
println(s?.let { "len ${it.length}" } ?: "empty")
```

Output: `empty`. `?.let` is skipped for null, then Elvis provides the fallback.

**Q8. Nested `it`**

```kotlin
listOf(1, 2).forEach {
    listOf(10).forEach { print(it) }
}
```

Output: `1010`. The inner `it` shadows the outer one.

**Q9. Closure counter**

```kotlin
fun counter(): () -> Int { var c = 0; return { ++c } }
val a = counter(); val b = counter()
println("${a()} ${a()} ${b()}")
```

Output: `1 2 1`. Each call to `counter()` creates its own captured `c`.

**Q10. Compile error or not?**

```kotlin
fun run2(block: () -> Unit) = block()
fun test() { run2 { return } }
```

Compile error: `run2` is not inline, so a bare `return` is not allowed. Fix with `return@run2` or by marking `run2` as `inline`.

## Sources

- Kotlin docs: Scope functions
- Kotlin docs: Higher-order functions and lambdas
- ProAndroidDev: `inline`, `noinline`, `crossinline` and `reified` explained
- ProAndroidDev: Android Interview Series 2024, Kotlin basics
- Android interview prep: Higher-order functions and lambdas
- Hyring: Top 60 Kotlin interview questions (2026)
- Second Talent: Top 30 Kotlin interview questions (2026)
- Senior Kotlin developer interview questions
