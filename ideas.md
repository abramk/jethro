# Jethro — Language Design Ideas

## Core Philosophy

- Strictly typed
- Right-to-left evaluation with uniform operator precedence (K/Q-style)
- Compiler written in Go, LLVM backend via llir/llvm

## Syntax

- Infix functions: `a foo b` instead of `foo(a, b)`
- Influences from Kotlin, Go, and Groovy
- Type-before-name: `int x` not `x: int` — no `:` symbol for type annotations
- No `->` for return types — return type follows param list directly
- `void` return type required when function returns nothing
- Function signature is always: `fn name(params) returnType body`
- `=>` for single-expression bodies (fn and closure) — implicit return
- `{ }` for multi-line bodies — explicit `return` required (fn and closure)
- No underscores in identifiers

```
// single-expression fn
fn foo(int x) int => x + 2

// multi-line fn
fn bar(int x) int {
    i32 y = x * 2
    return y + 1
}

// void fn
fn baz(int x) void {
    print(x)
}

// single-expression closure
let double = { int x => x + 2 }

// multi-line closure
let process = { int x =>
    i32 y = x * 2
    return y + 1
}
```

## Variables

- Explicit type required: `Type name = value`
- Mutable by default, `const` for immutability
- `let` for type inference — only valid when RHS has a single unambiguous type
- `let x = 42` is a compiler error — numeric literals are ambiguous across integer/float types
- `let x = Bar<SomeGeneric>()` is ok — constructor return type is unambiguous
- `::` type coercion operator on values — disambiguates literals, casts, or calls user-defined conversion
- No `var` or `:=`

```
// explicit type — always works
i32 x = 42
const f64 PI = 3.14
string name = "bob"

// let — inferred from unambiguous RHS
let user = fetchUser(id)
let orders = Map<string, List<Order>>()
const config = loadConfig()

// let + :: for literal disambiguation
let y = 42::i8
let z = 3.14::f32

// :: as general coercion (cast, assertion, user-defined)
let wide = someInt::f64
let circle = shape::Circle
let euros = dollars::Euro          // calls user-defined ::Euro

// errors
let y = 42                         // ERROR: ambiguous numeric type
```

## Type System

- Algebraic types (sum + product) with exhaustive pattern matching
- Affine types for resource safety (use-at-most-once ownership)
- All types have value semantics (move by default, explicit `.copy()` when needed)
- Compiler decides stack vs heap via escape analysis — no Box/Rc/pointer types exposed
- Primitives: `i8, i16, i32, i64`, `u8, u16, u32, u64`, `f32, f64`, `bool`, `char`, `byte`
- Explicit `return` keyword required in multi-line bodies — no implicit returns
- `=>` single-expression bodies have implicit return

## Null, Void, and Never

## Switch

- Single `switch` keyword for value matching, type matching, and guards
- No destructuring — smart-cast narrows the type, access fields directly
- `=>` for single-expression arms, `{ }` for multi-line (explicit `return` if used as expression)
- First match wins — specific guards before general cases
- Exhaustive for sealed traits — compiler error if a case is missing
- `default` for the default arm
- No fallthrough
- Works as expression or statement
- Bare `if` arm for guard-only matching (no type/value)

```
// value matching
switch status {
    "active" => handleActive()
    "pending" => handlePending()
    "closed", "archived" => handleDone()
    default => handleUnknown()
}

// type matching (sealed trait) — smart-cast, no destructuring
switch shape {
    Circle => PI * shape.radius * shape.radius
    Rect => shape.w * shape.h
}

// type + guard
switch shape {
    Circle if shape.radius > 10 => clampCircle(shape)
    Circle => PI * shape.radius * shape.radius
    Rect if shape.w == shape.h => squareArea(shape)
    Rect => shape.w * shape.h
}

// value + bare if guard
switch code {
    200 => "ok"
    if code >= 400 && code < 500 => "client error"
    if code >= 500 => "server error"
    default => "unknown"
}

// nullable
switch name {
    string if name.length > 0 => "hi ${name}"
    string => "empty name"
    null => "no name"
}

// as expression
string label = switch status {
    "active" => "🟢"
    "pending" => "🟡"
    default => "⚪"
}
```

## Null, Void, and Never

- `void` — function returns normally, no value
- `never` — function never returns (bottom type, subtype of every type)
- `T?` — built-in nullable type, not sugar for any class
- `null` — the only value of the absent case in `T?`
- `?:` — elvis operator: `expr ?: default`
- Non-null by default — `string` can never be null, `string?` can
- Smart casting in branches — compiler narrows `T?` to `T` after null check

```
string? x = "hello"
string? y = null

string a = x ?: "world"   // "hello"
string b = y ?: "world"   // "world"

fn greet(string? name) string {
    if name != null {
        return "hi ${name}"   // compiler narrows to string
    }
    return "hi stranger"
}

fn panic(string msg) never {
    // terminates — never returns
}

fn getUser(UserId id) User {
    return findUser(id) ?: panic("not found")
    // never is subtype of User, so ?: types as User
}
```

## Classes & Traits

- Classes as the organizing unit (data + behavior together)

- No inheritance — classes are always final

- Traits for abstraction — conformance is explicit at the class declaration

- Sealed traits only (not sealed classes) for sum types

- First-class delegation via `by` — compiler generates forwarding methods, replaces inheritance for code reuse

- Trait fields allowed — desugar to getter/setter accessor functions

- Trait methods always qualified by trait name at definition site
  
  - Aliasing for shared implementations: `fn B::execute = A::execute`
  - Delegation generates qualified forwarding: `fn A::foo() => inner.A::foo()`

- Implicit `self` — trait methods are always instance methods, no `self` parameter in signatures

- Static blocks for associated functions (factories, etc.) — no `self`, access private fields

- Call site uses `.` for everything:
  
  - `instance.method()` — unambiguous trait method, no qualification needed
  - `instance.Trait::method()` — required only when multiple traits define same method name
  - `Type.staticMethod()` — static call via type name

- Traits can extend other traits — expresses "this trait requires that trait"
  
  - Name conflicts allowed — resolved at call site with `Trait::method()` qualification
  
  - Overriding a supertrait method uses qualified syntax in the definition
    
    ```
    trait C : A, B {
      fn A::process() void { ... }   // override A's version
      fn B::process() void { ... }   // override B's version
    }
    c.A::process()    // ok
    c.B::process()    // ok
    c.process()       // error — ambiguous
    ```

- Constructors use `new` keyword:
  
  - `new(params)` — default constructor, called as `Type.new(...)`
  
  - `new name(params)` — named constructor, called as `Type.name(...)`
  
  - No sugar — `Point(...)` is not valid, always `Point.new(...)`
    
    ```
    class Point {
      i32 x
      i32 y
    
      new(i32 x, i32 y)
      new polar(f64 r, f64 theta) {
          self.x = (r * cos(theta))::i32
          self.y = (r * sin(theta))::i32
      }
    }
    ```
  
  let p1 = Point.new(10, 20)
  let p2 = Point.polar(5.0, 1.57)
  
  ```
  
  ```

- Open question: value types (move/copy) vs reference types (heap/pointer) for classes

## Properties (Getters/Setters)

- `get` and `set` are contextual keywords (only special after `fn`)
- Dot syntax to bind accessor to field: `get.name`, `set.name`
- Works with trait qualification: `fn A::get.name() string`
- Backing field access inside accessor via `field` keyword (avoids recursion)
- Getter without setter = read-only from outside the class

## Visibility

- Keywords: `public`, `private`
- Three levels: `private` (file-only), default (module-visible), `public` (outside module)
- Default is module-visible — no keyword needed for the common case
- Annotations (`@`) reserved for metaprogramming, not visibility

## Example Sketch

```
trait Serializable { fn serialize() Bytes }
trait Area { fn area() f64 }

class Circle : Serializable, Area {
    f64 radius

    fn Serializable::serialize() Bytes => ...
    fn Area::area() f64 => PI * radius * radius

    static {
        fn unit() Circle => Circle{ radius: 1.0 }
        fn deserialize(Bytes data) Circle => ...
    }
}

// call site — no qualification needed (unambiguous)
Circle c = Circle.deserialize(data)   // static
Circle c = Circle.unit()              // static factory
c.serialize()                      // instance
c.area()                           // instance

// ambiguous case — qualification required
trait A { fn execute() string }
trait B { fn execute() string }

class Bar : A, B {
    fn A::execute() string => "a"
    fn B::execute() string => "b"
}

Bar b = Bar{}
b.execute()        // compile error: ambiguous
b.A::execute()     // ok
b.B::execute()     // ok

// aliasing
fn B::execute() = A::execute

// sealed traits for sum types
sealed trait Shape : Area {
    class Circle { f64 radius }
    class Rect { f64 w, f64 h }
}

// delegation
class ScaledShape : Area by inner {
    Area inner
    f64 scale
    fn Area::area() f64 => inner.Area::area() * scale
}
```

## Loops & Iteration

## Error Handling

- `!` suffix on return type = function can fail
- `error` keyword to return an error — any type, no `Error` trait required
- Compiler infers the set of possible error types from all `error` statements in the function body
- Propagation with `?` unions the called function's error types into the caller's set
- Postfix `switch` on a `!` call — matches error types, success value is implicit
- Exhaustiveness checked by compiler against the inferred error type set
- `?` operator to propagate errors — only valid in functions that return `!`
- Ordering: `T?!` is valid (nullable + failable), `T!?` is a compiler error
- No `catch` keyword — postfix `switch` handles everything
- When used as assignment, every arm must produce a value of the correct type or diverge (`return`/`error`)

### Type algebra

- `T` — always a value
- `T?` — value or null
- `T!` — value or error
- `T?!` — value, null, or error

```
fn readFile(string path) string! {
    if !exists(path) {
        error FileNotFound{ path: path }
    }
    return read(path)
}

fn writeFile(string path, string data) void! { ... }

fn findUser(i64 id) User?! {
    if dbDown() { error DbError{} }     // error path
    if !found(id) { return null }        // null path
    return user                          // success path
}

// postfix switch — success flows to id, arms handle errors
let id = foo() switch {
    default => -1
}

// specific error types
let id = foo() switch {
    NotFound => -1
    DbError => { log(err); return }     // diverges — ok
    default => error it                  // re-throw
}

// do other work AND produce a value
let id = foo() switch {
    default => { someVar = 5; -1 }      // last expression is the value
}

// void! — postfix switch as statement
writeFile("out.txt", data) switch {
    default => println("write failed")
}

// propagate with ?
fn loadConfig() Config! {
    let content = readFile("config.txt")?
    return parse(content)
}

// COMPILE ERROR — unhandled failable call
let content = readFile("data.txt")

// COMPILE ERROR — arm doesn't produce value or diverge
let id = foo() switch {
    default => someVar = 5              // void, not i32
}
```

## Modules & Packages

- File extension: `.jet`
- Directory = package, name derived from directory path — no `package` declaration in files
- `module.jet` at the root marks the module boundary and names the module
- No nested modules
- Module is the compilation unit — compiler sees all packages in a module together

### Directory Structure

```
myproject/
  module.jet                // module com.xyz.myproject
  app.jet                   // package: com.xyz.myproject
  math/
    vector.jet              // package: com.xyz.myproject.math
    matrix.jet              // package: com.xyz.myproject.math
  math/geo/
    transform.jet           // package: com.xyz.myproject.math.geo
  net/http/
    client.jet              // package: com.xyz.myproject.net.http
```

### Imports

- `...` prefix = relative to module root (internal)
- No prefix = absolute (external module)
- Full, aliased, and selective imports allowed — no wildcards

```
import ...math.Vector3                // relative — single type
import ...math.{ Vector3, Matrix4 }  // relative — selective
import ...math.Vector3 as Vec3       // relative — aliased
import com.external.json.Parser      // absolute — external module
```

### module.jet

- Regular `.jet` file — no special DSL or config format
- Annotations on `init()` declare module metadata and dependencies
- `init()` runs once when module is loaded
- Dependency modules' `init()` runs before the dependent module's

```
@module("com.xyz.myproject")
@version("1.0.0")
@dependency("com.other.lib", "2.3.0")
@dependency("org.json.parser", "1.0.0")
fn init() void {
    // module-level initialization
}
```

### Dependencies

- Source distribution — dependencies are fetched as source and compiled locally
- No prebuilt binary artifacts
- Declared via `@dependency` annotations on `init()` in `module.jet`

### Re-exports

- No re-exports — you import from where things actually live
- To expose external types as part of your API, write a wrapper with delegation

### Circular Dependencies

- Allowed within a module (compiler resolves via multi-pass)
- Forbidden between modules

## Strings & Interpolation

- Two string literal forms:
  - `"..."` — plain string, no interpolation
  - `` `...` `` — template string, compiler processes `${}` expressions
- Both produce the same `string` type
- `${expr}` inside backtick strings calls `op.string` on the expression
- Primitives have `op.string` built-in by the compiler
- User types define `op.string` to control their string representation
- Recursive — `op.string` can use backtick strings that call `op.string` on nested types, all the way down to primitives

```
"hello world"                  // plain string
`hello ${name}`                // template — calls name.op.string()

class Money {
    i64 cents
    fn op.string() string => `$${cents / 100}.${cents % 100}`
}

class Order {
    i64 id
    Money total
    fn op.string() string => `Order #${id}: ${total}`
    // total.op.string() called recursively
}

string msg = `Your ${order} has shipped`
```

## Operator Overloading

- Operators are methods named `op.name` — same dot-namespace pattern as `get.name`/`set.name`
- Symbolic operators have a fixed name mapping (table below)
- Word operators are open-ended — any `op.name` method is callable as `a name b`
- Primitive types have built-in operators — cannot be overridden
- Only two unary operators: `-` (`op.neg`) and `!` (`op.not`)
- No `~` operator — bitwise not is `.bitnot()` method on integer primitives
- Derived operators are auto-generated from base definitions
- `+=`, `-=`, etc. are sugar for `a = a.op.X(b)`

### Symbolic Operators

| Operator          | Name          | Notes                   |
| ----------------- | ------------- | ----------------------- |
| `+`               | `op.plus`     |                         |
| `-`               | `op.minus`    |                         |
| `*`               | `op.times`    |                         |
| `/`               | `op.div`      |                         |
| `%`               | `op.mod`      |                         |
| `-` (unary)       | `op.neg`      |                         |
| `!` (unary)       | `op.not`      |                         |
| `==`              | `op.eq`       |                         |
| `!=`              | derived       | `!(a.op.eq(b))`         |
| `<=>`             | `op.compare`  | spaceship — returns i32 |
| `<` `>` `<=` `>=` | derived       | from `op.compare`       |
| `&&`              | `op.and`      |                         |
| `\|\|`            | `op.or`       |                         |
| `&`               | `op.bitand`   |                         |
| `\|`              | `op.bitor`    |                         |
| `^`               | `op.bitxor`   |                         |
| `<<`              | `op.shl`      |                         |
| `>>`              | `op.shr`      |                         |
| `..`              | `op.range`    |                         |
| `[]`              | `op.get`      |                         |
| `[]=`             | `op.set`      |                         |
| `in`              | `op.contains` | called on RHS           |
| `::`              | `op.cast`     | type coercion           |
| `?:`              | `op.elvis`    | nullable default        |
| `<-` (binary)     | `op.put`      | `ch <- value`           |
| `<-` (unary)      | `op.take`     | `<- ch`                 |

### Word Operators (open-ended infix)

- Any `op.name` method is usable as `a name b`
- No fixed set — parser resolves `expr IDENT expr` as infix
- Type-before-name syntax prevents ambiguity with declarations

```
fn op.to(i32 end) Range<i32> => Range{ start: self, end: end }
fn op.by(i32 step) SteppedRange => ...

1 to 10           // 1.op.to(10)
1 to 10 by 2      // (1.op.to(10)).op.by(2)
```

```
class Money {
    i64 cents
    fn op.plus(Money other) Money => Money{ cents: cents + other.cents }
    fn op.compare(Money other) i32 => cents - other.cents
}

Money total = price + tax       // price.op.plus(tax)
if price > limit { ... }        // derived from op.compare
```

## Loops & Iteration

- `for` is the only loop keyword — no `while`, no `loop`
- `for { }` — infinite loop (no condition)
- `for x in thing { }` — iterate (requires `Iterable<T>` trait)
- `break`/`continue` work in both forms
- `Iterable<T>` is the one magic trait — `for x in` desugars to calling it
- Any user type can implement `Iterable<T>` and work with `for`
- Numeric types get `.times`, ranges work with `for`

```
trait Iterable<T> {
    fn iterator() Iterator<T>
}

trait Iterator<T> {
    fn next() T?    // null means done
}
```

```
// infinite loop
for {
    Message msg = ch.recv()
    if msg.isPoison() { break }
    process(msg)
}

// iterate collection
for x in list {
    process(x)
}

// range
for i in 0..10 {
    print(i)
}

// break from iteration
for x in list {
    if x == "stop" { break }
    process(x)
}
```

## Vector Model

- Adverbs modify operators to work over vectors
  - `/` (over) — fold/reduce
  - `\` (scan) — accumulate
  - `'` (each) — map
  - `':` (eachprior) — pairwise

## Compiler Architecture

- Hand-written lexer (Go)
- Hand-written recursive descent parser (Go)
- Typed AST → semantic analysis → LLVM IR → native binary

## Aspects & Annotations (Compile-time AOP)

- Annotations are not passive markers — they are active compile-time code generators
- Defined via `aspect` keyword — the aspect IS the annotation and its transformation
- Compiler executes aspects at compile time to produce final code (GWT-style codegen)
- No separate annotation processor, no runtime reflection needed

### Aspect Definition

- `before { }` — run before the function
- `after { }` — run after the function
- `around(joinpoint) { }` — full control, call `joinpoint.proceed()` for original

### Join Point Metadata

- `joinpoint.name` — function name
- `joinpoint.args` — argument names + values
- `joinpoint.argTypes` — argument types
- `joinpoint.returnType` — return type
- `joinpoint.class` — enclosing class
- `joinpoint.traits` — traits this fn implements
- `joinpoint.annotations` — other annotations on this fn
- `joinpoint.taggedArgs` — args marked with `@tag`
- `joinpoint.proceed()` — invoke the original

### Tagged Variables — Cross-cutting Data Capture

- `@tag` on parameters or local variables marks them as visible to aspects
- Optional name: `@tag("uid") userId` → tag name is "uid"
- Aspects decide what to do with tagged values (span attributes, debug output, metrics dimensions)
- `@tag` is a general mechanism, not specific to any one aspect

### Composability

- Multiple aspects stack: outermost annotation wraps innermost
- Class-level annotation applies to all methods in the class

### Named Regions — Sub-joinpoints Within a Function

- `@"name" { ... }` declares a named code region inside a function body
- Not a closure — transparent, variables flow in and out, no new scope
- Visible to all aspects active on the function/class
- Aspects define a `region(name, block)` handler to react to regions
- If no aspect handles regions, they're zero-cost (compiler strips them)
- Consistent `@` symbol: `@Aspect` = apply aspect, `@"name"` = named region, `@tag` = variable tagging

```
@trace
fn processOrder(Order order) Receipt {

    @"validation" {
        ValidationResult valid = validate(order)
        @tag Errors errors = valid.errors
    }

    @"persistence" {
        SaveResult saved = db.save(order)
        @tag i64 rowId = saved.id
    }

    return buildReceipt(saved)
}

// aspect handles regions as child spans
aspect trace(TraceLevel level = TraceLevel.Info) {
    around(joinpoint) {
        Span span = Span.start(joinpoint.name, level)
        let result = joinpoint.proceed()
        span.end()
        return result
    }

    region(name, block) {
        Span child = Span.start(name)
        block.proceed()
        child.end()
    }
}
```

```
// define an aspect
aspect trace(TraceLevel level = TraceLevel.Info) {
    around(joinpoint) {
        Span span = Span.start(joinpoint.name, level)
        for arg in joinpoint.taggedArgs {
            span.setAttribute(arg.tagName, arg.value)
        }
        let result = joinpoint.proceed { tagged =>
            span.setAttribute(tagged.tagName, tagged.value)
        }
        span.ok()
        return result
    }
}

// apply it
@trace
fn processOrder(@tag Order order, @tag("uid") UserId userId) Receipt {
    @tag Money total = order.calculateTotal()
    Receipt intermediate = someHelper()            // not tagged
    @tag("validated") bool ok = validate(order)
    ...
}
// span attributes: order, uid, total, validated

// stack aspects
@trace
@retry(3)
@cache(ttl: 60)
fn fetchUser(UserId id) User => ...

// class-level — applies to all methods
@trace
class OrderService {
    fn handle(Order order) Receipt => ...
    fn cancel(OrderId id) Result => ...
}
```

## Implicit Context (no parameter passing)

- Every function executes within an implicit context, automatically propagated down the call stack
- `within` blocks create scoped child contexts:
  - `within timeout(5s) { ... }` — deadline
  - `within cancellable { ... }` — returns cancel handle
  - `within value(key, val) { ... }` — typed context values
- Nesting works naturally — inner scope wins
- Cancellation is cooperative — checked at function call boundaries and IO operations
- Replaces Go's `context.Context` without signature pollution

## Concurrency

- Async/await under the hood, but no function coloring — all functions are potentially suspendable
- Compiler transforms to state machines — LLVM-compatible, no custom green thread runtime
- `chan` keyword for channels (buffered/unbuffered), like Go
- `<-` operator for both put and take:
  - Binary `ch <- value` — put (calls `op.put` on left operand)
  - Unary prefix `<- ch` — take (calls `op.take` on left operand)
  - `op.put`/`op.take` are overloadable — any type can define them (sockets, queues, streams)
  - `select` only works with `chan` types
- `spawn` for launching concurrent tasks
- `select` for multiplexing channel operations, including timeout
- Structured concurrency — spawned tasks scoped to their `within` block, auto-cancelled on exit

```
// channel creation
let ch = chan<i32>()       // unbuffered
let ch = chan<i32>(10)     // buffered, capacity 10

within timeout(30s) {
    let ch = chan<Result>(10)

    spawn { ch <- fetchUser(id) }
    spawn { ch <- fetchOrders(id) }

    let user = <- ch
    let orders = <- ch
}

select {
    <- ch1 => val { process(val) }
    <- ch2 => val { handle(val) }
    timeout 5s { fallback() }
}

// op.put/op.take on user types
socket <- packet       // socket.op.put(packet)
let pkt = <- socket    // socket.op.take()
queue <- job           // queue.op.put(job)
let job = <- queue     // queue.op.take()
```

## Type System Decisions

- No union types — if a function needs to accept multiple types, use overloading
- No intersection types as a type expression — use generic bounds instead (`<T : A & B>`)
- `T?` and `T!` remain built-in, not sugar for union types
- Type algebra unchanged: `T`, `T?`, `T!`, `T?!`

## Function Overloading

- Overloads must differ in parameter count or at least one parameter type

- No return-type overloading

- Resolution requires exactly one match — if multiple overloads match, compiler error

- Use `::` to cast and disambiguate at the call site

- No implicit widening — exact type match only

- Default parameters must not create overlap with another overload — compiler error at definition site

- Traits: a class can add overloads beyond what its trait declares

- Operator methods (`op.name`) can be overloaded
  
  ```
  fn format(i32 n) string => `${n}`
  fn format(f64 n) string => `${n}`
  fn format(string s) string => s
  
  format(42::i32)      // ok — exact match
  format("hello")      // ok — exact match
  format(42)           // error — ambiguous literal
  
  // trait conformance ambiguity
  fn highlight(Shape s) void { ... }
  fn highlight(Circle c) void { ... }
  Circle c = Circle.new(10)
  highlight(c)         // error — matches both
  highlight(c::Shape)  // ok — explicit
  
  // default param overlap — caught at definition
  fn foo(i32 x) void { ... }
  fn foo(i32 x, i32 y = 0) void { ... }  // error — overlaps with foo(i32)
  ```

## Closures

- Syntax: `{ params => body }` with `it` as implicit single parameter

- Capture by copy (default) — closure gets its own copy of captured variables

- `ref` keyword for by-reference capture — listed in parameter list before `=>`

- `ref` captures cannot escape — compiler error if closure is passed to `spawn`

- Compiler may optimize non-escaping copy captures to avoid actual copy

- No borrow checker, no escape analysis visible to the user
  
  ```
  // implicit single param
  list.filter { it > 5 }
  
  // explicit param
  list.map { i32 x => x * 2 }
  
  // ref capture for mutation
  i32 sum = 0
  list.each { i32 x, ref sum => sum += x }
  
  // multiple ref captures
  i32 sum = 0
  i32 count = 0
  list.each { i32 x, ref sum, ref count =>
      sum += x
      count += 1
  }
  
  // chained with ref
  i32 total = 0
  list.map { i32 x => x * 2 }.each { i32 x, ref total => total += x }
  
  // spawn — copy only, ref is compiler error
  spawn { doWork(x) }           // ok — x copied in
  spawn { ref sum => sum += 1 } // error: ref capture cannot escape
  ```

## Tuples

- `:` is the tuple operator — works for any arity

- Type syntax mirrors value syntax: `i32:string:f64`

- Access elements with `.0`, `.1`, `.2`, etc.

- No ambiguity — identifiers can't start with digits
  
  ```
  // 2-tuple (pair)
  let entry = "a":1               // string:i32
  
  // 3-tuple
  let point = 10:20:30            // i32:i32:i32
  point.0                         // 10
  point.1                         // 20
  point.2                         // 30
  point.3                         // compiler error
  
  // as function parameter types
  fn foo(string:i32 entry) void { ... }
  fn bar(i32:i32:i32 point) void { ... }
  ```

- `:` in other contexts is unambiguous — trait conformance (`class Foo : Bar`) and generic bounds (`<T : Comparable>`) are syntactically distinct

## Strings

- `string` — lowercase, built-in primitive like `i32`, `f64`, `bool`
- Immutable, UTF-8 encoded
- Methods via traits and operators — no special treatment:
  - `Sized` trait → `.size()` (byte length or codepoint count — TBD)
  - `Iterable<char>` → `for` loops over characters
  - `op.get` → indexing
  - `op.contains` → `in` operator
  - `op.eq` → equality
  - `op.compare` → ordering
  - `op.plus` → concatenation
- Two literal forms (already decided):
  - `"plain string"` — no interpolation
  - `` `template ${expr}` `` — interpolation via `op.string`

## Primitive Arrays

- `T[]` — fixed-size, contiguous memory, compiler-known type

- Size determined at creation, not part of the type

- Interface expressed through traits — no special-case methods:
  
  - `Sized` trait → `.size()`
  - `Iterable<T>` trait → `for` loops
  - `op.get` / `op.set` → indexing

- `[1, 2, 3]` literal creates a `T[]`

- Varargs `T...` is sugar for `T[]` — can pass a `T[]` directly, no spread operator needed
  
  ```
  i32[] arr = [1, 2, 3]
  arr.size()                     // Sized trait
  arr[0]                         // op.get
  arr[0] = 42                   // op.set
  for x in arr { ... }          // Iterable<i32>
  
  i32[] bigger = i32[10]         // size 10, zero-initialized
  
  fn sum(i32... nums) i32 { ... } // nums is i32[]
  sum(1, 2, 3)                   // compiler packs into i32[]
  sum(arr)                       // just works — already i32[]
  ```

- Library collections (`List<T>`, etc.) built on top of `T[]`

## Collection Literals

- Arrays: `[1::i32, 2, 3]` — built-in syntax

- Lists, maps, sets: static factory methods in stdlib, no special syntax
  
  ```
  import ...collection.List.listOf
  import ...collection.Map.mapOf
  import ...collection.Set.setOf
  
  let arr = [1::i32, 2, 3]                // Array<i32> — built-in
  let list = listOf(1::i32, 2, 3)         // List<i32>
  let map = mapOf("a":1, "b":2)           // Map<string, i32> — vararg of string:i32
  let set = setOf(1::i32, 2, 3)           // Set<i32>
  ```

- `mapOf` accepts `K:V...` (vararg of pair tuples)

- No special treatment — any user type can define similar factory methods

## Enums

- `enum` keyword — closed set of named instances, all same shape

- Can have fields, constructor, and methods

- Exhaustive in `switch`

- Cannot be instantiated outside the enum definition

- Sealed traits for sum types where each case has different structure
  
  ```
  enum Color {
      RED("FF0000")
      GREEN("00FF00")
      BLUE("0000FF")
  
      string hex
      new(string hex)
  }
  
  fn toRgb(Color c) i32 => c switch {
      RED => 0xFF0000
      GREEN => 0x00FF00
      BLUE => 0x0000FF
  }
  
  let hex = Color.RED.hex
  ```

## Extensions

- `extend` keyword to add methods to types you don't own

- Can add methods, not state (no new fields)

- Resolved statically (compile-time declared type)

- Must be imported to be visible — no ambient extensions

- Cannot override existing methods on the type

- Visibility: default module-scoped, `public`, or `private` (file-only)

- Generic constraints on the type parameter directly
  
  ```
  // module-scoped (default)
  extend List<T : Multipliable> {
      op.times(T scalar) List<T> => self.map { T x => x * scalar }
  }
  
  // public — importers can use it
  public extend List<T : Summable> {
      fn sum() T => self.fold(T.zero, &T.op.plus)
  }
  
  // caller must import
  import ...math.collections.List
  let result = list * 2       // works — extension imported
  let total = list.sum()      // works — extension imported
  ```

## Function Types & References

- Function type syntax: `fn(ParamTypes) ReturnType`

- `&` for function/method references

- No adverbs — use standard functional methods (`map`, `fold`, `scan`, `filter`) with closures or function references
  
  ```
  // function type as parameter
  fn fold<T, R>(R initial, fn(R, T) R reducer) R { ... }
  
  // closure
  list.fold(0) { i32 acc, i32 x => acc + x }
  
  // function reference with &
  list.fold(0, &i32.op.plus)
  list.map(&i32.op.string)
  
  // reference to a named function
  fn double(i32 x) i32 => x * 2
  list.map(&double)
  
  // store as a variable
  fn(i32, i32) i32 adder = &i32.op.plus
  ```

## Memory Model

- Move by default on assignment — original variable becomes unusable

- Explicit clone via `Cloneable` trait — opt-in, not universal

- `clone()` method — compiler can auto-generate field-by-field deep clone, overridable

- Escape analysis for stack vs heap placement — compiler decides, not the programmer

- Ownership-based cleanup — value freed when owner goes out of scope (like Rust's Drop)

- No GC, no reference counting, no borrow checker

- Circular references impossible with strict move semantics

- Graphs and similar structures: use index-based approach (arena pattern)

- `ref` in closures is a temporary borrow — can't escape to `spawn`
  
  ```
  i32[] a = [1, 2, 3]
  i32[] b = a              // a is moved, unusable
  i32[] c = b.clone()      // explicit clone — both b and c are valid
  
  // index-based graph
  class Graph {
      Node[] nodes
  }
  class Node {
      i32 id
      i32[] neighbors      // indices, not references
  }
  ```

## FFI (Foreign Function Interface)

- `@ffi("name")` annotation on empty function — compiler generates C calling convention glue

- `@ffi.struct` on class — C-compatible memory layout, still a regular Jethro class

- `@ffi.union` on sealed trait — overlapping memory layout for C unions

- `Ptr<T>` built-in for raw pointer interop (unsafe)
  
  ```
  @ffi.struct
  class Point {
      f64 x
      f64 y
  }
  
  @ffi("translate_point")
  fn cTranslate(Point p, f64 dx, f64 dy) Point
  
  @ffi("strlen")
  fn cStrlen(i8[] str) i64
  ```

## Runtime

- Goal: self-contained binaries with no external dependencies (like Zig)
- Bundle a minimal libc (musl for Linux) for static linking
- LLVM backend handles cross-compilation to multiple targets
- Minimal runtime: memory allocator, thread spawning, channel primitives
- Can bypass libc and call syscalls directly where possible
- Ship linker as part of the toolchain (LLVM's LLD)

## Other Decisions

- Comments: C-style only (`//` and `/* */`), no doc comments

- `bigint`: library, not built-in

- No reference equality — with move/copy semantics, no two references to same object

- `op.string`: implicit on every class — compiler generates default (class name + fields), overridable

- `op.eq`: explicit — must implement to use `==`. Without it, `==` is a compiler error

- `Hashable`: explicit — must implement to use as map key or set member

- Type aliases with `alias` keyword — three forms:
  
  - Type alias: `alias Name = SomeType<Foo, Bar>`
  
  - Trait bound combination: `alias Numeric = Summable & Multipliable & Comparable`
  
  - Type set (constraint only): `alias integer = i8 | i16 | i32 | i64`
  
  - Type sets can compose: `alias numeric = integer | floating`
  
  - `|` only valid in alias definitions for type sets — not a general type expression
  
  - Type sets can only be used as generic constraints, not as variable types
  
  - Type sets and trait bounds can be mixed with `&` in constraints, parentheses for grouping
    
    ```
    alias integer = i8 | i16 | i32 | i64
    alias floating = f32 | f64
    alias numeric = integer | floating
    ```
  
  fn sum<T : numeric>(List<T> items) T => ...           // type set constraint
  fn foo<T : numeric & Serializable>(T value) void { }  // type set + trait bound
  fn bar<T : (i32 | f32) & Comparable>(T value) void { } // inline type set + trait
  numeric x = 42                                         // error — not a variable type
  
  ```
  
  ```
  
  ```
  
  ```
