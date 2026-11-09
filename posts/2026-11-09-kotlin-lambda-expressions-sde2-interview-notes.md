# Kotlin Lambda Expressions: SDE 2 Interview Notes

Today I revised Kotlin lambda expressions end to end: fundamentals, function types, syntax shortcuts, higher-order functions, closures, lambdas with receiver, scope functions, returns, `inline`/`noinline`/`crossinline`/`reified`, SAM conversion, runtime cost, and the interview questions that keep coming up.

## 1. Fundamentals

A lambda is an anonymous function literal: a function with no name that you can store, pass, and return like any other value.

One-line interview definition:
"A lambda is a function literal. Kotlin functions are first-class, so a lambda can be stored in a variable, passed as an argument, or returned from a function."

Why it exists:

- Less boilerplate than anonymous classes.
- Enables functional style (`map`, `filter`).
- Makes callbacks and DSLs possible.

Analogy:
A lambda is a sticky note with instructions. It has no title, but you can hand it to someone and they can follow it later.

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

Simple meaning:
Because functions are values, a lambda can go anywhere a value can: a variable, an argument, a return value, a collection, a property, or a default parameter.

Analogy:
A recipe card is just a piece of paper. You can keep it in a drawer, hand it to a friend, or let the friend choose which card to use.

```kotlin
val sq: (Int) -> Int = { it * it }                              // variable
list.map { it * 2 }                                             // argument
fun mul(f: Int): (Int) -> Int = { it * f }                      // return value
class Button(val onClick: () -> Unit)                           // property / callback
fun greet(fmt: (String) -> String = { "Hi $it" })               // default parameter
fun load(onDone: (() -> Unit)? = null) { onDone?.invoke() }     // optional callback
```

How to read the code:

- Variable: name, type, and value together: `val op: (Int, Int) -> Int = { x, y -> x + y }`.
- Parameter: only name and type in the declaration (`op: (Int, Int) -> Int`); the lambda arrives at the call site.

Internal idea:
There are two ways to decide the operation.

```kotlin
val add = { x: Int, y: Int -> x + y }                          // operation decided now, values later
fun calculate(a: Int, b: Int, op: (Int, Int) -> Int) = op(a, b) // operation and values decided at the call
calculate(10, 5, add)                                          // they combine
```

Flow:

```text
lambda created -> stored / passed / returned -> invoked later
```

Android use:

- Click listeners: `button.setOnClickListener { }`.
- Callbacks: `onSuccess`, `onError`.
- Compose: `onClick = { }` and content lambdas.

Common mistake:
Forgetting to call a stored lambda. `val greet = { println("Hi") }` does nothing until `greet()` runs.

## 5. Higher-Order Functions (HOF)

Simple meaning:
A higher-order function takes a function as a parameter, returns a function, or both.

Analogy:
A coffee machine with a swappable recipe. The machine (fixed algorithm) stays the same; you decide what drink to make by handing it a different recipe.

```kotlin
fun calculate(a: Int, b: Int, op: (Int, Int) -> Int): Int = op(a, b)

calculate(10, 5) { x, y -> x + y }   // 15
calculate(10, 5) { x, y -> x * y }   // 50
```

How to read the code:
"`calculate` takes `a` (Int), `b` (Int) and `op` (a function from two Ints to an Int), and returns an Int, which is `op(a, b)`."

- Three inputs: two numbers and one function.
- `-> Int` inside the parentheses is `op`'s return type; `): Int` after them is `calculate`'s return type.
- `= op(a, b)` is an expression body, shorthand for `{ return op(a, b) }`.

Internal idea:
HOFs separate what varies (the lambda) from the fixed algorithm (looping, sorting, retrying, timing).

Flow:

```text
10 -> a -> x
5  -> b -> y
lambda -> op -> body calls op(10, 5) -> 15
```

Returning a function:

```kotlin
fun multiplier(factor: Int): (Int) -> Int = { it * factor }
val triple = multiplier(3)
triple(5)   // 15
```

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

Android use:

- `map`, `filter`, `groupBy`, `sortedBy` on lists.
- `lazy { }`, `synchronized(lock) { }`, `use { }` for resources.
- Retry and timing wrappers around network calls.

Common mistakes:

- Confusing `-> Int` inside the parentheses (the lambda's result) with `: Int` after them (the function's result).
- Writing a custom loop where `map`/`filter`/`fold` already express the intent.

## 6. Closures (Capturing Variables)

Simple meaning:
A closure is a lambda that captures variables from the scope where it was created and keeps access to them after that scope ends.

Analogy:
A backpack. When the lambda leaves its home function, it carries the variables it needs inside the backpack.

```kotlin
fun counter(): () -> Int {
    var count = 0
    return { ++count }
}

val c = counter()
c(); c()
println(c())   // 3, count lives on inside the lambda
```

How to read the code:
`counter()` creates `count`, returns a lambda that increments it, and then finishes. The lambda still remembers `count`.

Internal idea:
A captured `var` is wrapped in a `Ref` object (`Ref.IntRef`, `Ref.ObjectRef`) so the lambda and the outer scope share one mutable box. Captured `val`s are copied into fields of the lambda object.

Flow:

```text
counter() -> count lives in a Ref box -> lambda holds the box -> each call updates the same box
```

Kotlin vs Java (frequent question):

- Java lambdas can capture only effectively final variables.
- Kotlin lambdas can capture and modify `var`s.

Android use:

- Click listeners capturing a counter or state.
- Callbacks capturing a ViewModel, Context, or a View.

Common mistakes:

- Memory leaks: a lambda that captures an Activity, View, or `this` and is stored in a long-lived object (singleton, static listener) keeps that object alive.
- Thread safety: a captured `var` modified from several threads or coroutines is shared mutable state; use `AtomicInteger` or a `Mutex`.
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

Analogy:
Handing a chef a ready-made workstation. Inside the station you just say "chop" and "boil" without naming the kitchen each time.

```kotlin
val shout: String.() -> String = { uppercase() + "!" }   // this = the String
"hi".shout()   // "HI!"
shout("hi")    // also valid: receiver passed as first argument
```

How to read the code:
`String.() -> String` is "a function called on a String that returns a String."

Builder example (classic interview task):

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

Internal idea:
The compiler passes the receiver as a hidden first argument, so `Html.() -> Unit` behaves like `(Html) -> Unit`, but the body sees `this`.

Flow:

```text
html { } -> Html() created -> lambda runs with this = Html -> p("Hello"), p("World") -> Html returned
```

Android use:

- `apply`, `run`, `with`, `buildString`, `buildList`.
- Gradle Kotlin DSL, Ktor routing.
- Jetpack Compose: `Column { }` content uses `ColumnScope.() -> Unit`.
- Coroutine builders: `launch` takes `CoroutineScope.() -> Unit`.

Notes:

- `@DslMarker` stops nested DSL blocks from implicitly calling an outer receiver's methods.
- Extension function vs lambda with receiver: same idea (`this` = receiver); an extension function is named and declared once, a lambda with receiver is a value passed at the call site.

Common mistake:
Nesting DSL blocks without `@DslMarker` and accidentally calling an outer builder's function from an inner block.

## 8. Scope Functions

Simple meaning:
Scope functions are five small inline higher-order functions that run a lambda in the context of an object. They differ on two axes only: how you refer to the object (`it` or `this`) and what they return (the lambda result or the object).

Analogy:
Different ways to work on a parcel. Some let you talk about "this parcel" directly, some call it "it"; some hand back the parcel, some hand back a receipt.

| Function | Object as | Returns | Typical use |
| --- | --- | --- | --- |
| `let` | `it` | lambda result | null checks `x?.let { }`, transform a value |
| `run` | `this` | lambda result | configure and compute a result |
| `with` | `this` | lambda result | group calls on an object |
| `apply` | `this` | the object | object initialisation / builder |
| `also` | `it` | the object | side effects: logging, validation |

```kotlin
val len = name?.let { it.length } ?: 0
val user = User().apply { name = "A"; age = 30 }
val list = mutableListOf(1).also { println("created $it") }
val area = rect.run { width * height }
with(canvas) { drawLine(); drawCircle() }
```

How to read the code:
Ask two questions: "does the block use `it` or `this`?" and "do I want the result or the object back?"

Internal idea:
The signatures explain everything (worth memorising):

```kotlin
inline fun <T, R> T.let(block: (T) -> R): R
inline fun <T, R> T.run(block: T.() -> R): R
inline fun <T, R> with(receiver: T, block: T.() -> R): R
inline fun <T> T.apply(block: T.() -> Unit): T
inline fun <T> T.also(block: (T) -> Unit): T
```

Memory trick:
`(T) ->` gives `it`, `T.() ->` gives `this`. Returns `R` = lambda result, returns `T` = the object.

Flow:

```text
object -> scope function -> lambda runs on object -> result or object returned
```

Android use:

- `intent.apply { putExtra(...) }`.
- `savedState?.let { restore(it) }`.
- `also { Log.d(...) }` inside a chain.
- `with(binding) { title.text = ...; subtitle.text = ... }`.

Common follow-ups:

- `apply` vs `also`: both return the object; `apply` uses `this` (configure), `also` uses `it` (side effect).
- `let` vs `run`: both return the result; `let` uses `it`, `run` uses `this`.
- `run` vs `with`: same behaviour; `run` is an extension (works with `?.`), `with` takes the object as an argument.

Common mistake:
Deep nesting of scope functions, where `it` and `this` become ambiguous and the code gets hard to read.

## 9. Returns in Lambdas

Simple meaning:
A bare `return` inside a lambda returns from the enclosing function, not the lambda, and is only allowed when the lambda is passed to an inline function. This is called a non-local return.

Analogy:
A fire alarm inside a room. A bare `return` evacuates the whole building (enclosing function); `return@label` only leaves the room (the lambda).

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

How to read the code:
In `findFirstEven`, the first even number ends the whole function. In the `return@forEach` version, only the current item is skipped.

Internal idea:
After inlining, the lambda body sits inside the caller, so returning from the caller is real code. A non-inline lambda is a separate object that might run later or on another thread, so a return to a finished caller makes no sense.

Flow:

```text
inline forEach -> lambda body copied into caller -> bare return = return from caller
```

Android use:

- Early exits from `list.forEach` inside a validation function.
- `return@launch` inside coroutine builders.

Common mistakes:

- Expecting bare `return` in `forEach` to behave like `continue`.
- `break` from `forEach` is not possible. Use a `for` loop, `run loop@{ list.forEach { if (...) return@loop } }`, or `firstOrNull` / `takeWhile`.
- `val f = { return 5 }` does not compile; use `return@label` or an anonymous function `fun(): Int { return 5 }`.

## 10. `inline`, `noinline`, `crossinline`, `reified`

Simple meaning:
`inline` copies a function's body and its lambda arguments into the call site, removing the lambda object allocation and the virtual `invoke` call. The other three keywords adjust that behaviour. This is the most-asked advanced lambda topic in senior Kotlin and Android rounds.

Analogy:
Instead of calling a helper who brings a tool (object) every time, you photocopy the helper's instructions straight into your own notebook (call site).

| Keyword | Applies to | What it does |
| --- | --- | --- |
| `inline` | function | inlines the body and all lambda params at the call site; enables non-local return |
| `noinline` | one lambda param | that lambda stays a real object; no non-local return |
| `crossinline` | one lambda param | still inlined, but bare `return` is forbidden |
| `reified` | type param | keeps `T` available at runtime (`is T`, `T::class`) |

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

How to read the code:

- `a` is inlined and can use non-local return; `b` stays an object so it can be stored or passed on.
- `block` runs inside `handler.post { }`, a different context, so a bare `return` would be unsafe: `crossinline` bans it.
- `reified T` lets `it is T` work even though generics are erased.

Internal idea:

- Without `inline`: each lambda compiles to a `FunctionN` object, capturing lambdas allocate on every call, and calls go through `invoke` (virtual call, boxing of primitives).
- With `inline`: the code is pasted at the call site, so none of that happens.
- `reified` needs `inline` because JVM generics are erased at runtime; inlining substitutes the real type at each call site.

Flow:

```text
call site -> inline function body + lambda body copied in -> no object, no invoke
```

Android use:

- `Gson.fromJson<T>()` style helpers with `reified`.
- `runOnUiThread`-style helpers that need `crossinline`.
- `measure { }` and `let`/`map` style utilities.

Common mistakes:

- Inlining large bodies: bytecode grows at every call site.
- Inlining recursive functions: not allowed.
- Using private/internal members inside a public inline function: they would leak into callers.
- Marking a function `inline` with no lambda params: no benefit, the compiler warns.

`noinline` vs `crossinline` in one line:
`noinline` = not inlined at all (becomes an object); `crossinline` = inlined, but no non-local return.

## 11. Lambda vs Anonymous Function vs Function Reference

Simple meaning:
All three produce a value of a function type. They differ in syntax, return behaviour, and return-type declaration.

Analogy:
Three ways to give directions: shout a quick instruction (lambda), write a detailed letter with a stated destination (anonymous function), or point to an existing signpost (function reference).

| | Lambda | Anonymous function | Function reference |
| --- | --- | --- | --- |
| Syntax | `{ x: Int -> x * 2 }` | `fun(x: Int): Int { return x * 2 }` | `::double` |
| Bare `return` | exits the enclosing function (inline only) | exits the anonymous function itself | n/a |
| Explicit return type | no, inferred | yes, can be declared | from the referenced function |

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

How to read the code:

- The anonymous function has several exits, and `return` leaves only itself.
- `String::length` turns a member into a function where the receiver becomes the parameter.
- `user::greet` is bound to one instance; `::User` is the constructor as a function.

Internal idea:
A named `fun` is a declaration; a lambda is an expression. `::name` converts a declaration into a value.

Flow:

```text
named fun -> ::name -> function value -> passed like a lambda
```

Android use:

- `list.map(::toUiModel)` to reuse mapper functions.
- `viewModel::onClick` as a bound reference for callbacks.

Common mistakes:

- Using a bare `return` in a lambda and expecting it to return only from the lambda.
- Lambda vs Java lambda: Kotlin has real function types; Java lambdas only target functional interfaces, and Kotlin lambdas can modify captured `var`s while Java's cannot.

## 12. SAM Conversion and `fun interface`

Simple meaning:
SAM (Single Abstract Method) conversion lets you pass a lambda wherever an interface with exactly one abstract method is expected.

Analogy:
A job role with exactly one duty. Anyone who can do that one duty (a lambda) can fill the role without a formal contract.

```kotlin
val r = Runnable { println("running") }          // Java interface: automatic
button.setOnClickListener { view -> /* ... */ }

fun interface Validator {                         // Kotlin interface: must be fun interface
    fun validate(input: String): Boolean
}

val notEmpty = Validator { it.isNotEmpty() }
notEmpty.validate("")   // false
```

How to read the code:
`Validator { it.isNotEmpty() }` creates a `Validator` whose single method `validate` runs the lambda.

Internal idea:

- Java interfaces: SAM conversion is automatic.
- Kotlin interfaces: need the `fun interface` marker (since Kotlin 1.4).
- Without SAM you would write `object : Validator { override fun validate(input: String) = input.isNotEmpty() }`.

Flow:

```text
lambda -> wrapped as interface instance -> interface method call runs the lambda
```

`fun interface` vs function type `(String) -> Boolean`:

- `fun interface`: has a name, can have default methods and extra members, clearer in public APIs, Java-friendly.
- Function type: lighter, no declaration needed, composes easily.

Android use:

- `OnClickListener`, `Runnable`, `Comparator`, `Callable`.
- Public library callbacks that Java code must also call.

Common mistake:
An `object :` expression creates a new instance each time and has its own `this`; a SAM-converted lambda's `this` refers to the enclosing class.

## 13. Under the Hood and Performance

Simple meaning:
SDE 2 rounds probe whether you know what a lambda costs at runtime, not just how to write one.

Analogy:
Writing a note is free, but delivering a fresh envelope every time (a new object per call) adds up on a busy day.

Internal idea:

- Compilation: `(A, B) -> R` becomes `Function2<A, B, R>` with `invoke(a, b)`. Since Kotlin 2.0, lambdas compile with `invokedynamic` (`LambdaMetafactory`) by default instead of a class per lambda.
- Non-capturing lambda: one shared instance, effectively free after the first use.
- Capturing lambda: a new object each time the lambda expression is evaluated.
- Captured `var`: wrapped in a `Ref` box (another allocation).
- Boxing: generic `FunctionN` takes and returns `Object`, so `(Int) -> Int` boxes primitives unless the call is inlined.

Flow:

```text
lambda expression -> (capturing? new object : shared instance) -> invoke -> maybe boxing
inline: none of the above
```

Collections vs sequences:

```kotlin
list.filter { }.map { }                      // eager: one intermediate list per step
list.asSequence().filter { }.map { }.toList() // lazy: element by element
```

Prefer sequences for large data or early termination (`first`, `take`). Sequence operations are not inline, so their lambdas are objects.

Android use:

- Avoid capturing lambdas inside tight loops or `onDraw`; hoist them or use inline functions.
- Compose: unstable lambdas that capture changing values can cause extra recompositions; `remember { }` a lambda when needed.

Common mistakes:

- Assuming lambdas are free everywhere. A capturing lambda in a hot path allocates repeatedly.
- Long lambda chains that are hard to step through and give unreadable stack traces. Balance elegance with debuggability.

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
