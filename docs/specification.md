# NovelScript Language Specification

**Version 0.1 — Draft**

## 1. Overview & Design Philosophy

NovelScript is a pipeline-oriented, pattern-driven programming language designed for
correctness and clarity. Its primary audience is AI agents generating code, with human
readability as a strong secondary goal.

### Core Principles

1. **Pipeline-first computation** — Data flows left-to-right through transformations.
Nested function calls are replaced by pipelines, eliminating precedence confusion.
2. **One way to branch** — Pattern matching is the sole branching mechanism. There is
no `if`, `else`, `switch`, or `cond`. This eliminates choice paralysis for code
generators and ensures exhaustive case handling.
3. **Contracts are code** — Preconditions (`require`) and postconditions (`ensure`) are
part of function signatures, checked at runtime by default.
4. **Proofs live with code** — Inline test blocks (`proof`) are first-class language
constructs, not a separate framework. They execute as part of the program and halt
on failure.
5. **Bijections** — Bidirectional functions can be defined once and used in either
direction. Useful for encoding/decoding, serialization, unit conversion, and more.
6. **No null** — The language has no null or nil. Optional values use `Option[T]`
(`some(v)` or `none`), errors use `Result[T, E]` (`ok(v)` or `err(e)`).
7. **Immutable by default** — Bindings are immutable unless explicitly declared `mut`.
8. **Explicit over implicit** — No hidden conversions, no implicit returns from

---

## 2. Lexical Structure

### 2.1 Character Set

Source files are UTF-8 encoded. Identifiers and keywords use ASCII letters, digits,
       and underscores. String literals may contain any UTF-8 text.

### 2.2 Comments

       ```
       -- This is a single-line comment

{- This is a
    multi-line comment -}
    ```

    Multi-line comments nest: `{- outer {- inner -} still outer -}`.

### 2.3 Keywords

    ```
    fn  let  mut  match  case  else  do  end  shape  module  use  as
    require  ensure  proof  assert  biject  forward  reverse  with
    true  false  none  some  ok  err  and  or  not  to  until
    return  stream  collect  type
    ```

### 2.4 Identifiers

    Identifiers begin with a letter or underscore, followed by letters, digits, or
    underscores. By convention:
    - Variables and functions: `snake_case`
    - Shapes and type names: `PascalCase`
    - Module names: `PascalCase`
    - Constants: `UPPER_SNAKE_CASE`

### 2.5 Operators and Punctuation

    ```
    |>    pipeline
|?>   conditional pipeline (propagates err/none)
    =>    case arrow / lambda arrow
    <-    mutation
    =     binding
    ++    concatenation
    +  -  *  /  %    arithmetic
    ==  !=  <  >  <=  >=    comparison
    ..    rest pattern / spread
    @     annotation prefix
    ,  :  .  (  )  [  ]  {  }  #
    ```

### 2.6 Operator Precedence (highest to lowest)

    | Precedence | Operators              | Associativity |
    |------------|------------------------|---------------|
    | 7          | unary `-`, `not`       | right         |
    | 6          | `*`, `/`, `%`          | left          |
    | 5          | `+`, `-`               | left          |
    | 4          | `++`                   | left          |
    | 3          | `==`, `!=`, `<`, `>`, `<=`, `>=` | none |
    | 2          | `and`                  | left          |
    | 1          | `or`                   | left          |
    | 0          | `|>`, `|?>`            | left          |

    Comparisons are non-associative: `a < b < c` is a syntax error. Use
    `a < b and b < c`.

    ---

## 3. Types

### 3.1 Primitive Types

    | Type   | Description                  | Literals                    |
    |--------|------------------------------|-----------------------------|
    | `Num`  | Arbitrary-precision number   | `42`, `3.14`, `-7`, `1_000` |
    | `Text` | UTF-8 string                 | `"hello"`, `"line\n"`       |
    | `Bool` | Boolean                      | `true`, `false`             |
    | `Void` | No meaningful value          | (no literal)                |

    `Num` is a unified numeric type. The interpreter uses integers when possible and
    floats when necessary. Integer division is performed with `//` (floor division), while
    `/` is true division.

### 3.2 Composite Types

    | Type           | Description                | Literal / Constructor             |
    |----------------|----------------------------|-----------------------------------|
    | `List[T]`      | Ordered, immutable list    | `[1, 2, 3]`                      |
    | `Map[K, V]`    | Key-value map              | `#{"a": 1, "b": 2}`              |
    | `Option[T]`    | Optional value             | `some(42)`, `none`                |
    | `Result[T, E]` | Success or error           | `ok(42)`, `err("bad")`           |
    | `Stream[T]`    | Lazy, possibly infinite    | constructed via `stream` function |

### 3.3 Shape Types

    Shapes are named structural types (see Section 8).

### 3.4 Function Types

    Function types are written: `(ParamTypes) -> ReturnType`

    ```
    (Num, Num) -> Num
    (Text) -> Option[Text]
    () -> Void
    ```

### 3.5 Type Parameters

    Types and functions can be parameterized with square brackets:

    ```
    shape Pair[A, B] {
first: A,
           second: B
    }

fn identity[T](x: T) -> T do
x
end
```

### 3.6 Type Inference

Type annotations are optional on local bindings. The interpreter infers types where
possible. Function signatures should include type annotations for documentation and
contract checking, but the interpreter will accept unannotated functions.

---

## 4. Expressions

### 4.1 Literals

    ```
    42                    -- Num (integer)
    3.14                  -- Num (float)
1_000_000             -- Num (underscores ignored)
    "hello"               -- Text
    "value is {x + 1}"   -- Text with interpolation
    true                  -- Bool
    false                 -- Bool
    [1, 2, 3]             -- List[Num]
#{"a": 1}             -- Map[Text, Num]
    some(42)              -- Option[Num]
none                  -- Option[T] (type inferred from context)
    ok("success")         -- Result[Text, E]
    err("failure")        -- Result[T, Text]
    ```

### 4.2 String Interpolation

    Curly braces inside double-quoted strings evaluate expressions:

    ```
    let name = "world"
    "Hello, {name}!"          -- "Hello, world!"
    "2 + 2 = {2 + 2}"         -- "2 + 2 = 4"
    "items: {list |> length}"  -- "items: 5"
    ```

    Use `\{` for a literal brace: `"use \{braces\}"`.

### 4.3 Variable Binding

    ```
    let x = 42                -- immutable
    let mut counter = 0        -- mutable
counter <- counter + 1     -- mutation (only on mut bindings)
    ```

    Attempting to mutate an immutable binding is a compile-time error.

### 4.4 Block Expressions

    A `do...end` block is an expression that evaluates to its last expression:

    ```
    let result = do
    let a = 10
    let b = 20
    a + b
    end
    -- result is 30
    ```

### 4.5 Lambda Expressions

    ```
    n => n * 2                         -- single parameter
    (a, b) => a + b                    -- multiple parameters
    () => print("hello")               -- no parameters
    (items) => do                      -- multi-line (block) lambda
    items
    |> filter(n => n > 0)
|> map(n => n * 2)
    end
    ```

    When a lambda is the last argument to a function, it may be written as a trailing
    block:

    ```
    [1, 2, 3] |> map do |n|
    let doubled = n * 2
    doubled + 1
    end
    -- [3, 5, 7]
    ```

### 4.6 Range Expressions

    ```
    1 to 5       -- [1, 2, 3, 4, 5]     (inclusive)
1 until 5    -- [1, 2, 3, 4]         (exclusive of end)
    ```

    Ranges are lazy streams. They produce values on demand and can be used in pipelines.

    ---

## 5. Declarations

### 5.1 Function Declaration

    ```
    fn function_name(param1: Type1, param2: Type2) -> ReturnType
    do
-- body (last expression is the return value)
    end
    ```

    The return type can be omitted and will be inferred.

#### Contracts

    Functions may declare preconditions and postconditions:

    ```
    fn factorial(n: Num) -> Num
    require n >= 0
    require n == n // 1    -- n is an integer
    ensure result >= 1
    do
    match n
    case 0 => 1
case n => n * factorial(n - 1)
    end
    end
    ```

    - `require` — Checked before the function body executes. Failure raises a contract
    violation with the expression that failed.
    - `ensure` — Checked after the body executes. The special name `result` refers to the
    return value.

    Multiple `require` and `ensure` clauses are allowed. All are checked independently.

#### Intent Annotations

    Functions may carry an `@intent` annotation describing their purpose in natural
    language. This has no runtime effect but serves as machine-readable documentation:

    ```
    @intent "Sort a list of numbers in ascending order using merge sort"
    fn merge_sort(items: List[Num]) -> List[Num]
    ensure result |> length == items |> length
    do
    -- implementation
    end
    ```

### 5.2 Module Declaration

    ```
    module Geometry do
    shape Point {
x: Num,
       y: Num
    }

    fn distance(a: Point, b: Point) -> Num do
((a.x - b.x) * (a.x - b.x) + (a.y - b.y) * (a.y - b.y))
    |> sqrt
    end
    end
    ```

    Modules may be nested. Everything inside a module is accessible via dot notation:
    `Geometry.distance(p1, p2)`.

### 5.3 Use (Import) Declaration

    ```
    use Geometry                   -- import module, access via Geometry.distance
    use Geometry.{Point, distance} -- import specific items into current scope
    use Geometry as Geo            -- aliased import: Geo.distance
    ```

### 5.4 Type Alias

    ```
    type Name = Text
    type Point3D = { x: Num, y: Num, z: Num }
    type Predicate[T] = (T) -> Bool
    ```

    ---

## 6. Pattern Matching

    Pattern matching is the **only** branching mechanism in NovelScript.

### 6.1 Match Expression (with subject)

    ```
    match value
    case pattern1 => expr1
    case pattern2 => expr2
    case _ => default_expr
    end
    ```

### 6.2 Match Expression (without subject — conditional)

    When no subject is given, each `case` acts as a boolean guard:

    ```
    match
    case x > 100 => "large"
    case x > 10 => "medium"
    case x > 0 => "small"
    else => "non-positive"
    end
    ```

    Cases are evaluated top-to-bottom. The first matching case wins. `else` is equivalent
    to `case _` or `case true`.

### 6.3 Pattern Forms

    | Pattern                  | Matches                                      |
    |--------------------------|----------------------------------------------|
    | `42`, `"hello"`, `true`  | Literal equality                             |
    | `x`                      | Any value, binds to `x`                      |
    | `_`                      | Any value, no binding (wildcard)             |
    | `[]`                     | Empty list                                   |
    | `[a, b, c]`              | List of exactly 3 elements                   |
    | `[head, ..tail]`         | List with at least 1 element, rest as `tail` |
    | `[_, ..tail]`            | Same, discarding the head                    |
    | `#{"key": v}`            | Map containing "key", binds value to `v`     |
    | `ShapeName { field: p }` | Shape with field matching pattern `p`        |
    | `some(x)`                | Option containing a value                    |
    | `none`                   | Empty option                                 |
    | `ok(x)`                  | Successful result                            |
    | `err(e)`                 | Error result                                 |
    | `(a, b)`                 | Tuple of two elements                        |

### 6.4 Guards

    Patterns can have guards using `when`:

    ```
    match value
    case n when n > 0 => "positive"
    case 0 => "zero"
    case n => "negative: {n}"
    end
    ```

### 6.5 Exhaustiveness

    The interpreter warns if a match expression is not exhaustive (does not cover all
            possible values). A wildcard `_` or `else` branch ensures exhaustiveness.

    ---

## 7. Pipelines

### 7.1 Standard Pipeline `|>`

    The pipeline operator passes the left-hand value as the **first** argument to the
    right-hand function:

    ```
    "hello world"
    |> split(" ")          -- ["hello", "world"]
    |> map(w => w |> upper_first)  -- ["Hello", "World"]
    |> join(" ")           -- "Hello World"
    ```

    Equivalent to: `join(map(split("hello world", " "), ...), " ")`

### 7.2 Conditional Pipeline `|?>`

    The conditional pipeline unwraps `ok`/`some` values and short-circuits on
    `err`/`none`:

    ```
    fn load_config(path: Text) -> Result[Config, Text] do
    read_file(path)              -- Result[Text, Text]
    |?> parse_json             -- only called on ok
    |?> validate_config        -- only called on ok
    |?> apply_defaults         -- only called on ok
    end
    ```

    If any step returns `err(e)`, the entire pipeline evaluates to `err(e)`. If any step
    returns `none`, the pipeline evaluates to `none`.

    Rules:
    - If the input is `ok(v)`, `v` is extracted and passed to the next function.
    - If the input is `err(e)`, the pipeline short-circuits and returns `err(e)`.
    - If the input is `some(v)`, `v` is extracted and passed to the next function.
    - If the input is `none`, the pipeline short-circuits and returns `none`.
    - If the next function returns an unwrapped value (not Result/Option), it is
    automatically wrapped in `ok()`/`some()` to continue the pipeline.

### 7.3 Tap

    The built-in `tap` function executes a side effect without altering the pipeline value:

    ```
    data
    |> tap(d => print("loaded: {d |> length} items"))
    |> transform
    |> tap(d => print("transformed"))
    |> output
    ```

    `tap(value, fn)` calls `fn(value)` for its side effect and returns `value` unchanged.

### 7.4 Inspect

    The built-in `inspect` function prints a debug representation of the value and returns
    it unchanged:

    ```
    data |> inspect |> transform |> inspect |> output
    -- Prints: [inspect] <value representation>
    -- at each inspect point
    ```

    `inspect` can take an optional label: `inspect("after transform")`.

    ---

## 8. Shapes

    Shapes are named, structural record types.

### 8.1 Definition

    ```
    shape Person {
name: Text,
          age: Num,
          email: Option[Text]
    }
```

### 8.2 Construction

```
let alice = Person {
name: "Alice",
          age: 30,
          email: some("alice@example.com")
}
```

All fields must be provided at construction.

### 8.3 Field Access

```
alice.name       -- "Alice"
alice.age        -- 30
```

### 8.4 Update with `with`

Since shapes are immutable, `with` creates a copy with updated fields:

    ```
let older_alice = alice |> with(age: 31)
    -- older_alice.age is 31, all other fields unchanged
    ```

    Multiple fields can be updated:

    ```
let updated = alice |> with(age: 31, email: none)
    ```

### 8.5 Shape Methods

    Functions can be associated with a shape by defining them inside a module of the same
    name:

    ```
    shape Circle {
center: Point,
            radius: Num
    }

module Circle do
fn area(c: Circle) -> Num do
3.14159 * c.radius * c.radius
end

    fn scale(c: Circle, factor: Num) -> Circle do
c |> with(radius: c.radius * factor)
    end
    end

    -- Usage:
    let c = Circle { center: Point { x: 0, y: 0 }, radius: 5 }
    c |> Circle.area       -- 78.53975
    c |> Circle.scale(2)   -- Circle with radius 10
    ```

    ---

## 9. Bijections

    A bijection defines a bidirectional transformation — a function and its inverse,
    declared together to ensure consistency.

### 9.1 Definition

    ```
biject celsius_fahrenheit(c: Num) <-> (f: Num)
    forward do
    c * 9 / 5 + 32
    end
    reverse do
    (f - 32) * 5 / 9
    end
    end
    ```

### 9.2 Usage

    ```
    100 |> celsius_fahrenheit            -- 212.0
    212 |> celsius_fahrenheit.rev        -- 100.0
    ```

    - `celsius_fahrenheit` calls the forward direction.
    - `celsius_fahrenheit.rev` calls the reverse direction.

### 9.3 Composition

    Bijections can be composed to create new bijections:

    ```
biject double(x: Num) <-> (y: Num)
    forward do x * 2 end
    reverse do y / 2 end
    end

biject add_one(x: Num) <-> (y: Num)
    forward do x + 1 end
    reverse do y - 1 end
    end

    let double_then_add = double >> add_one   -- composed bijection
    5 |> double_then_add        -- 11
    11 |> double_then_add.rev   -- 5
    ```

    The `>>` operator composes bijections: `(a >> b).forward = b.forward(a.forward(x))`
    and `(a >> b).rev = a.rev(b.rev(x))`.

### 9.4 Proof of Bijectivity

    The interpreter can optionally verify that forward and reverse are inverses by running
    round-trip checks on proof values:

    ```
biject celsius_fahrenheit(c: Num) <-> (f: Num)
    forward do c * 9 / 5 + 32 end
    reverse do (f - 32) * 5 / 9 end
    end

    proof "celsius_fahrenheit round-trip" do
    [0, 100, -40, 37] |> each(c =>
            assert c |> celsius_fahrenheit |> celsius_fahrenheit.rev == c
            )
    end
    ```

    ---

## 10. Proofs & Contracts

### 10.1 Proof Blocks

    Proof blocks are inline test assertions. They execute as part of the program and abort
    on failure. They can be disabled in production with an interpreter flag (`--no-proofs`).

    ```
    proof "list operations" do
    let xs = [3, 1, 4, 1, 5]
    assert xs |> length == 5
    assert xs |> sort == [1, 1, 3, 4, 5]
    assert xs |> filter(x => x > 2) == [3, 4, 5]
    end
    ```

    A proof block has:
- A descriptive string name (required)
    - A `do...end` body containing `assert` statements
    - No parameters and no return value

### 10.2 Assert

    `assert` takes a boolean expression. On failure, it reports:
    - The proof block name
    - The failed expression as source text
    - The actual values of sub-expressions

    ```
    assert 2 + 2 == 5
    -- Error: Assertion failed in proof "math"
    --   assert 2 + 2 == 5
    --   Left:  4
    --   Right: 5
    ```

### 10.3 Contracts (Require / Ensure)

    See Section 5.1. Contracts are checked at runtime by default. They can be disabled
    with `--no-contracts`.

    ---

## 11. Streams & Iteration

    NovelScript has **no loop constructs** (`for`, `while`, `loop`). All iteration is
    accomplished through streams, pipelines, and recursion.

### 11.1 Creating Streams

    ```
    -- From a range
    1 to 10                              -- stream of 1..10 inclusive
    1 until 10                           -- stream of 1..9

    -- From a generator function
stream(0, n => n + 2)                -- 0, 2, 4, 6, ... (infinite)

    -- From a list
    [1, 2, 3] |> to_stream               -- finite stream
    ```

### 11.2 Stream Operations

    All operations are lazy until a terminal operation is reached:

    ```
-- Transformations (lazy)
    stream |> map(fn)            -- transform each element
    stream |> filter(fn)         -- keep matching elements
    stream |> take(n)            -- first n elements
    stream |> drop(n)            -- skip first n elements
    stream |> take_while(fn)     -- take while predicate holds
    stream |> drop_while(fn)     -- drop while predicate holds
    stream |> zip(other)         -- pair elements from two streams
    stream |> enumerate          -- pair each element with its index
    stream |> flat_map(fn)       -- map then flatten
    stream |> chunk(n)           -- group into chunks of n

-- Terminal operations (force evaluation)
    stream |> collect            -- collect into a List
    stream |> reduce(init, fn)   -- fold into a single value
    stream |> each(fn)           -- execute side effect for each element
    stream |> count              -- count elements
    stream |> any(fn)            -- true if any element matches
    stream |> all(fn)            -- true if all elements match
stream |> find(fn)           -- first matching element (Option)
    stream |> sum                -- sum of numeric stream
    stream |> join(sep)          -- join text stream with separator
    ```

### 11.3 Examples

    ```
    -- Sum of squares of even numbers from 1 to 100
    1 to 100
    |> filter(n => n % 2 == 0)
|> map(n => n * n)
    |> sum
    -- 171700

    -- Fibonacci sequence
    let fibs = stream((0, 1), (a, b) => (b, a + b))
|> map((a, _) => a)
    fibs |> take(10) |> collect
    -- [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]

    -- FizzBuzz
    1 to 100 |> each(n =>
            match
            case n % 15 == 0 => print("FizzBuzz")
            case n % 3 == 0  => print("Fizz")
            case n % 5 == 0  => print("Buzz")
            else              => print(n |> to_text)
            end
            )
    ```

    ---

## 12. Standard Library

### 12.1 IO

    ```
    print(value)                         -- print to stdout with newline
    print_inline(value)                  -- print without newline
    read_line() -> Text                  -- read a line from stdin
    read_file(path: Text) -> Result[Text, Text]      -- read entire file
    write_file(path: Text, content: Text) -> Result[Void, Text]
    append_file(path: Text, content: Text) -> Result[Void, Text]
    ```

### 12.2 Math

    ```
    abs(n: Num) -> Num
    sqrt(n: Num) -> Num
    pow(base: Num, exp: Num) -> Num
    min(a: Num, b: Num) -> Num
    max(a: Num, b: Num) -> Num
    floor(n: Num) -> Num
    ceil(n: Num) -> Num
    round(n: Num) -> Num
    ```

### 12.3 Text

    ```
    length(t: Text) -> Num              -- also works on List
    upper(t: Text) -> Text
    lower(t: Text) -> Text
    trim(t: Text) -> Text
    split(t: Text, sep: Text) -> List[Text]
    contains(t: Text, sub: Text) -> Bool
    replace(t: Text, old: Text, new: Text) -> Text
    starts_with(t: Text, prefix: Text) -> Bool
    ends_with(t: Text, suffix: Text) -> Bool
    chars(t: Text) -> List[Text]        -- split into individual characters
    ```

### 12.4 List

    ```
    length(l: List[T]) -> Num
    get(l: List[T], index: Num) -> Option[T]
    append(l: List[T], item: T) -> List[T]
    prepend(l: List[T], item: T) -> List[T]
    concat(a: List[T], b: List[T]) -> List[T]
    reverse(l: List[T]) -> List[T]
    sort(l: List[T]) -> List[T]
    sort_by(l: List[T], key_fn: (T) -> Num) -> List[T]
    unique(l: List[T]) -> List[T]
    flatten(l: List[List[T]]) -> List[T]
    ```

### 12.5 Map

    ```
    get(m: Map[K, V], key: K) -> Option[V]
    put(m: Map[K, V], key: K, value: V) -> Map[K, V]
    remove(m: Map[K, V], key: K) -> Map[K, V]
    keys(m: Map[K, V]) -> List[K]
    values(m: Map[K, V]) -> List[V]
    entries(m: Map[K, V]) -> List[(K, V)]
    has_key(m: Map[K, V], key: K) -> Bool
    merge(a: Map[K, V], b: Map[K, V]) -> Map[K, V]
    ```

### 12.6 Convert

    ```
    to_text(value) -> Text
    to_num(t: Text) -> Result[Num, Text]
    ```

### 12.7 Functional

    ```
    identity(x: T) -> T                 -- returns its argument
compose(f, g) -> fn                  -- compose two functions: compose(f, g)(x) = f(g(x))
    tap(value: T, fn: (T) -> Void) -> T -- side effect, returns value unchanged
    inspect(value: T) -> T              -- debug print, returns value
    inspect(label: Text, value: T) -> T -- debug print with label
    ```

    ---

## 13. Program Structure

### 13.1 File Extension

    NovelScript source files use the `.ns` extension.

### 13.2 Entry Point

    A NovelScript program consists of one or more `.ns` files. Execution begins at the top
    of the main file and proceeds top-to-bottom. Declarations (functions, shapes, modules,
            bijections) are hoisted — they can be used before they are textually defined.

    If a `main` function is defined with signature `fn main(args: List[Text])`, it is
    called with command-line arguments after top-level code executes.

### 13.3 File Layout Convention

    ```
    -- Imports
    use ModuleName

    -- Shape definitions
    shape MyShape { ... }

    -- Function definitions
    fn my_function(...) do ... end

    -- Bijections
    biject my_conversion(...) <-> (...) ... end

    -- Proofs
    proof "description" do ... end

-- Top-level code (or main function)
    fn main(args: List[Text]) do
    -- entry point
    end
    ```

### 13.4 Multi-File Programs

    Files can import each other using `use`:

    ```
    -- In math_utils.ns
    module MathUtils do
    fn square(n: Num) -> Num do n * n end
    end

    -- In main.ns
    use MathUtils

    MathUtils.square(5) |> print   -- 25
    ```

    The interpreter resolves imports relative to the main file's directory.

    ---

## 14. Annotations

    Annotations provide metadata that does not affect runtime behavior (unless the
            interpreter uses them for diagnostics).

    ```
    @intent "Brief natural-language description of purpose"
    @deprecated "Use new_function instead"
    @todo "Handle edge case for empty lists"
    ```

    Annotations attach to the next declaration (function, shape, module, bijection).

    ---

## 15. Formal Grammar (Simplified)

    ```
    program        ::= declaration*
    declaration    ::= fn_decl | shape_decl | module_decl | use_decl
    | biject_decl | proof_decl | type_decl | expression

    fn_decl        ::= annotation* 'fn' IDENT type_params? '(' params ')' ('->' type)?
    contract* 'do' block 'end'
    contract       ::= 'require' expression | 'ensure' expression
    params         ::= (param (',' param)*)?
    param          ::= IDENT (':' type)?
    type_params    ::= '[' IDENT (',' IDENT)* ']'

    shape_decl     ::= 'shape' IDENT type_params? '{' field (',' field)* '}'
    field          ::= IDENT ':' type

    module_decl    ::= 'module' IDENT 'do' declaration* 'end'

    use_decl       ::= 'use' module_path ('as' IDENT)?
    | 'use' module_path '.{' IDENT (',' IDENT)* '}'
        module_path    ::= IDENT ('.' IDENT)*

            biject_decl    ::= 'biject' IDENT '(' param ')' '<->' '(' param ')'
            'forward' 'do' block 'end'
            'reverse' 'do' block 'end'
            'end'

            proof_decl     ::= 'proof' STRING 'do' block 'end'

            type_decl      ::= 'type' IDENT type_params? '=' type

            expression     ::= pipeline
            pipeline       ::= conditional (('|>' | '|?>') IDENT call_args?)*
            conditional    ::= or_expr
            or_expr        ::= and_expr ('or' and_expr)*
            and_expr       ::= comparison ('and' comparison)*
            comparison     ::= addition (comp_op addition)?
            addition       ::= multiplication (('+' | '-' | '++') multiplication)*
            multiplication ::= unary (('*' | '/' | '//' | '%') unary)*
            unary          ::= ('-' | 'not') unary | postfix
            postfix        ::= primary ('.' IDENT | '[' expression ']' | call_args)*
            primary        ::= literal | IDENT | '(' expression ')' | block_expr
            | match_expr | lambda | list | map

            match_expr     ::= 'match' expression? case_clause+ ('else' '=>' expression)? 'end'
            case_clause    ::= 'case' pattern ('when' expression)? '=>' expression
            pattern        ::= literal | IDENT | '_' | list_pattern | shape_pattern
            | option_pattern | result_pattern | tuple_pattern

            lambda         ::= params '=>' expression
            | params '=>' 'do' block 'end'

            block_expr     ::= 'do' block 'end'
            block          ::= (statement)*
            statement      ::= 'let' 'mut'? IDENT (':' type)? '=' expression
            | IDENT '<-' expression
            | 'assert' expression
            | 'return' expression
            | expression
            ```

            ---

## 16. Interpreter Flags

            ```
            novelscript <file.ns>               -- run program
            novelscript <file.ns> --no-proofs   -- skip proof blocks
            novelscript <file.ns> --no-contracts -- skip require/ensure checks
            novelscript --version                -- print version
            novelscript --help                   -- print usage
            ```

            ---

## Appendix A: Reserved for Future Versions

            The following features are reserved for future specification:

            - **Concurrency** — async/await, channels, parallel pipelines
            - **Traits / Interfaces** — shared behavior across shapes
            - **Pattern-based dispatch** — multi-method functions
            - **Foreign Function Interface** — calling into host language
            - **Package management** — dependency resolution and versioning
