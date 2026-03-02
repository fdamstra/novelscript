# NovelScript Tutorial

A hands-on introduction for experienced programmers.

---

## 1. Hello World

Create a file called `hello.ns`:

```
print("Hello, World!")
```

Run it:

```
$ novelscript hello.ns
Hello, World!
```

That's it. Top-level expressions execute in order. No boilerplate, no main function
required.

---

## 2. Values and Bindings

NovelScript has four primitive types:

```
let answer = 42              -- Num
let pi = 3.14159             -- Num (same type, integers and floats unified)
let greeting = "hello"       -- Text
let active = true            -- Bool
```

Bindings are immutable by default. Use `mut` for mutable bindings:

```
let x = 10
-- x <- 20  -- ERROR: cannot mutate immutable binding

let mut counter = 0
counter <- counter + 1       -- OK: counter is now 1
```

The mutation operator is `<-`, not `=`. This makes mutation visually distinct from
binding.

### Collections

```
let numbers = [1, 2, 3, 4, 5]             -- List[Num]
let config = #{"host": "localhost", "port": "8080"}  -- Map[Text, Text]
```

Lists and maps are immutable. Operations on them return new values:

```
let more = numbers |> append(6)       -- [1, 2, 3, 4, 5, 6]
-- numbers is still [1, 2, 3, 4, 5]
```

### String Interpolation

Expressions inside `{}` in strings are evaluated:

```
let name = "NovelScript"
let version = 1
print("Welcome to {name} v{version}!")
-- Welcome to NovelScript v1!
```

---

## 3. Functions

Functions are declared with `fn`:

```
fn greet(name: Text) -> Text do
  "Hello, {name}!"
end

greet("Alice") |> print    -- Hello, Alice!
```

The last expression in the body is the return value. No `return` keyword needed
(though it exists for early returns).

### Contracts

Functions can declare preconditions and postconditions:

```
fn divide(a: Num, b: Num) -> Num
  require b != 0
do
  a / b
end

divide(10, 0)   -- Runtime error: Contract violation: require b != 0
```

Postconditions use the special name `result`:

```
fn abs(n: Num) -> Num
  ensure result >= 0
do
  match n >= 0
    case true => n
    case false => -n
  end
end
```

---

## 4. Pipelines

Pipelines are the heart of NovelScript. The `|>` operator passes a value as the first
argument to the next function:

```
-- Without pipelines (nested, hard to read):
print(join(sort(split("banana,apple,cherry", ",")), ", "))

-- With pipelines (left-to-right, clear):
"banana,apple,cherry"
  |> split(",")
  |> sort
  |> join(", ")
  |> print
-- apple, banana, cherry
```

You can build complex transformations as readable chains:

```
let result = [4, 2, 7, 1, 9, 3]
  |> filter(n => n > 2)       -- [4, 7, 9, 3]
  |> map(n => n * 10)         -- [40, 70, 90, 30]
  |> sort                     -- [30, 40, 70, 90]
  |> take(3)                  -- [30, 40, 70]
```

### Debugging Pipelines

Use `inspect` to see intermediate values without breaking the chain:

```
[1, 2, 3, 4, 5]
  |> inspect                   -- prints: [inspect] [1, 2, 3, 4, 5]
  |> filter(n => n % 2 == 0)
  |> inspect                   -- prints: [inspect] [2, 4]
  |> map(n => n * n)
  |> inspect                   -- prints: [inspect] [4, 16]
```

### Tap for Side Effects

`tap` runs a function for its side effect but passes the value through unchanged:

```
data
  |> tap(d => print("Processing {d |> length} items"))
  |> transform
```

---

## 5. Pattern Matching

Pattern matching is the **only** way to branch in NovelScript. There is no `if/else`.

### Matching on Values

```
fn describe(n: Num) -> Text do
  match n
    case 0 => "zero"
    case 1 => "one"
    case n when n > 0 => "positive"
    case _ => "negative"
  end
end
```

### Matching without a Subject (Conditionals)

When you omit the subject, each case is a boolean guard:

```
fn classify_age(age: Num) -> Text do
  match
    case age < 13 => "child"
    case age < 20 => "teenager"
    case age < 65 => "adult"
    else => "senior"
  end
end
```

### Destructuring

Patterns can destructure complex values:

```
-- Lists
match items
  case [] => "empty"
  case [only] => "just one: {only}"
  case [first, ..rest] => "first is {first}, {rest |> length} more"
end

-- Options
match find_user(id)
  case some(user) => "Found: {user.name}"
  case none => "Not found"
end

-- Results
match parse_number(input)
  case ok(n) => "Parsed: {n}"
  case err(msg) => "Error: {msg}"
end

-- Shapes
match event
  case Click { x: x, y: y } => "Clicked at ({x}, {y})"
  case KeyPress { key: "Enter" } => "Enter pressed"
  case KeyPress { key: k } => "Key: {k}"
end
```

### Why No If/Else?

Pattern matching is always exhaustive — the interpreter warns if you miss a case.
This eliminates an entire class of bugs where a condition is unhandled. Using one
consistent mechanism also means there's never a question about which construct to use.

For simple boolean checks, `match` on the condition:

```
let label = match x > 0
  case true => "positive"
  case false => "non-positive"
end
```

Or use the guard form:

```
let label = match
  case x > 0 => "positive"
  else => "non-positive"
end
```

---

## 6. Error Handling

NovelScript has no exceptions. Errors are values.

### Result Type

Functions that can fail return `Result[T, E]`:

```
fn parse_port(s: Text) -> Result[Num, Text] do
  match s |> to_num
    case ok(n) when n > 0 and n <= 65535 => ok(n)
    case ok(n) => err("Port out of range: {n}")
    case err(e) => err("Not a number: {s}")
  end
end
```

### Conditional Pipeline `|?>`

The `|?>` operator makes error handling in pipelines seamless:

```
fn load_server_config(path: Text) -> Result[Config, Text] do
  read_file(path)
    |?> parse_json
    |?> extract_field("server")
    |?> validate_config
end
```

If `read_file` returns `err(...)`, the entire pipeline short-circuits. No nested match
expressions needed.

Compare with explicit error handling (equivalent but verbose):

```
fn load_server_config(path: Text) -> Result[Config, Text] do
  match read_file(path)
    case err(e) => err(e)
    case ok(text) => match parse_json(text)
      case err(e) => err(e)
      case ok(json) => match extract_field(json, "server")
        case err(e) => err(e)
        case ok(server) => validate_config(server)
      end
    end
  end
end
```

### Option Type

For values that may or may not exist:

```
let users = [
  #{"name": "Alice", "email": "alice@example.com"},
  #{"name": "Bob"}
]

users
  |> map(u => u |> get("email"))
  |> each(email =>
    match email
      case some(e) => print("Email: {e}")
      case none => print("No email")
    end
  )
```

### Unwrapping

For cases where you're certain a value exists:

```
let value = some(42) |> unwrap             -- 42 (panics on none)
let safe = some(42) |> unwrap_or(0)        -- 42 (returns 0 on none)
let lazy = none |> unwrap_or_else(() => compute_default())
```

---

## 7. Shapes

Shapes define structured data types:

```
shape Task {
  title: Text,
  done: Bool,
  priority: Num
}

let task = Task {
  title: "Write tutorial",
  done: false,
  priority: 1
}

print(task.title)   -- Write tutorial
```

### Updating Shapes

Since everything is immutable, use `with` to create modified copies:

```
let completed = task |> with(done: true)
-- completed.done is true, title and priority unchanged
```

### Associated Functions

Define a module with the same name as a shape to add methods:

```
shape Stack {
  items: List[Num]
}

module Stack do
  fn new() -> Stack do
    Stack { items: [] }
  end

  fn push(s: Stack, value: Num) -> Stack do
    s |> with(items: s.items |> prepend(value))
  end

  fn pop(s: Stack) -> Result[(Num, Stack), Text] do
    match s.items
      case [top, ..rest] => ok((top, s |> with(items: rest)))
      case [] => err("Stack is empty")
    end
  end

  fn peek(s: Stack) -> Option[Num] do
    match s.items
      case [top, ..] => some(top)
      case [] => none
    end
  end
end

-- Usage:
let s = Stack.new()
  |> Stack.push(10)
  |> Stack.push(20)
  |> Stack.push(30)

s |> Stack.peek |> print   -- some(30)
```

---

## 8. Streams and Iteration

There are no `for` or `while` loops. All iteration uses streams and pipelines.

### Ranges

```
1 to 5           -- stream: 1, 2, 3, 4, 5  (inclusive)
1 until 5        -- stream: 1, 2, 3, 4     (exclusive end)
```

### Iterating

```
-- Print numbers 1 through 10
1 to 10 |> each(n => print(n))

-- Sum of first 100 squares
1 to 100 |> map(n => n * n) |> sum   -- 338350

-- Find first even number greater than 10
1 to 100 |> find(n => n > 10 and n % 2 == 0)   -- some(12)
```

### Infinite Streams

```
-- Generate powers of 2
let powers = stream(1, n => n * 2)   -- 1, 2, 4, 8, 16, ...
powers |> take(10) |> collect        -- [1, 2, 4, 8, 16, 32, 64, 128, 256, 512]

-- Collatz sequence
fn collatz(n: Num) -> Stream[Num] do
  stream(n, x => match
    case x == 1 => 1
    case x % 2 == 0 => x / 2
    else => 3 * x + 1
  end)
    |> take_while(x => x != 1)
    |> append(1)
end

collatz(27) |> collect |> length |> print   -- 112 steps
```

### Recursion

For complex iteration patterns, use recursion:

```
fn gcd(a: Num, b: Num) -> Num do
  match b
    case 0 => a
    case b => gcd(b, a % b)
  end
end
```

The interpreter optimizes tail calls, so tail-recursive functions don't overflow the
stack.

---

## 9. Bijections

Bijections define reversible transformations — define once, use in both directions.

```
biject rot13(plain: Text) <-> (coded: Text)
  forward do
    plain |> chars |> map(c => shift_char(c, 13)) |> join("")
  end
  reverse do
    coded |> chars |> map(c => shift_char(c, 13)) |> join("")
  end
end

-- ROT13 is its own inverse, but the concept generalizes:
"Hello" |> rot13           -- "Uryyb"
"Uryyb" |> rot13.rev       -- "Hello"
```

A more interesting example — unit conversion:

```
biject km_miles(km: Num) <-> (mi: Num)
  forward do km * 0.621371 end
  reverse do mi / 0.621371 end
end

biject meters_km(m: Num) <-> (km: Num)
  forward do m / 1000 end
  reverse do km * 1000 end
end

-- Compose: meters to miles
let meters_to_miles = meters_km >> km_miles

1500 |> meters_to_miles            -- ~0.932
0.932 |> meters_to_miles.rev       -- ~1500
```

### When to Use Bijections

- Unit conversions (temperature, distance, weight)
- Encoding/decoding (Base64, ROT13, Caesar cipher)
- Serialization/deserialization
- Format transformations
- Any operation where you need the inverse

---

## 10. Proofs

Proofs are inline tests that verify your code works correctly. They run with your
program by default.

```
fn fibonacci(n: Num) -> Num
  require n >= 0
do
  match n
    case 0 => 0
    case 1 => 1
    case n => fibonacci(n - 1) + fibonacci(n - 2)
  end
end

proof "fibonacci sequence" do
  assert fibonacci(0) == 0
  assert fibonacci(1) == 1
  assert fibonacci(10) == 55
  assert fibonacci(20) == 6765
end
```

When a proof fails, you get a clear error:

```
Error: Assertion failed in proof "fibonacci sequence"
  assert fibonacci(10) == 55
  Left:  56
  Right: 55
  at fibonacci.ns:15
```

### Proofs as Documentation

Proofs serve double duty — they verify correctness AND document expected behavior:

```
fn parse_csv_line(line: Text) -> List[Text] do
  line |> split(",") |> map(s => s |> trim)
end

proof "CSV parsing" do
  assert parse_csv_line("a, b, c") == ["a", "b", "c"]
  assert parse_csv_line("hello") == ["hello"]
  assert parse_csv_line("") == [""]
end
```

Skip proofs in production: `novelscript app.ns --no-proofs`

---

## 11. Putting It All Together

Here's a complete program: a simple word-frequency counter.

```
-- word_freq.ns
-- Count word frequencies in a text file

fn count_words(text: Text) -> Map[Text, Num] do
  text
    |> lower
    |> replace(",", "")
    |> replace(".", "")
    |> replace("!", "")
    |> replace("?", "")
    |> split(" ")
    |> filter(w => w |> length > 0)
    |> reduce(#{}, (counts, word) =>
      match counts |> get(word)
        case some(n) => counts |> put(word, n + 1)
        case none => counts |> put(word, 1)
      end
    )
end

fn top_words(counts: Map[Text, Num], n: Num) -> List[(Text, Num)] do
  counts
    |> entries
    |> sort_by((_, count) => -count)
    |> take(n)
end

proof "count_words basics" do
  let counts = count_words("the cat sat on the mat")
  assert counts |> get("the") == some(2)
  assert counts |> get("cat") == some(1)
  assert counts |> get("dog") == none
end

fn main(args: List[Text]) do
  match args
    case [path] => do
      match read_file(path)
        case ok(text) => do
          let counts = count_words(text)
          let top = top_words(counts, 10)
          print("Top 10 words:")
          top |> each((word, count) =>
            print("  {word}: {count}")
          )
        end
        case err(msg) => print("Error reading file: {msg}")
      end
    end
    case _ => print("Usage: novelscript word_freq.ns <file>")
  end
end
```

---

## Quick Reference

| Concept              | Syntax                                        |
|----------------------|-----------------------------------------------|
| Binding              | `let x = 42`                                  |
| Mutable binding      | `let mut x = 0`                               |
| Mutation             | `x <- x + 1`                                  |
| Function             | `fn f(x: Num) -> Num do x * 2 end`            |
| Lambda               | `x => x * 2`                                  |
| Pipeline             | `value |> fn1 |> fn2`                          |
| Conditional pipeline | `value |?> fn1 |?> fn2`                        |
| Pattern match        | `match v case p => e end`                      |
| Conditional          | `match case guard => e else => e end`          |
| Shape                | `shape S { field: Type }`                      |
| Shape update         | `s |> with(field: value)`                      |
| List                 | `[1, 2, 3]`                                   |
| Map                  | `#{"key": value}`                              |
| Range (inclusive)    | `1 to 10`                                      |
| Range (exclusive)    | `1 until 10`                                   |
| Proof                | `proof "name" do assert expr end`              |
| Contract             | `require cond` / `ensure cond`                 |
| Bijection            | `biject f(a) <-> (b) forward do ... end ...`   |
| Comment              | `-- single line` / `{- multi line -}`          |
| String interpolation | `"Hello, {name}!"`                             |
