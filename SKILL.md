---
name: miniscript-2
description: Write, review, and debug MiniScript 2 (.ms) code in this project. Covers syntax and scoping, lists/maps and frozen values, functions and prototype classes, error values, intrinsics, command-line extras (file, exec, import), and standard library modules. Use whenever editing, running, or explaining MiniScript code.
license: MIT
metadata:
  category: language
  docs: https://github.com/JoeStrout/miniscript2/tree/main/docs
---

# MiniScript 2

MiniScript is a dynamically typed, prototype-based scripting language. Source files use the
`.ms` extension. It is line-oriented and keyword-blocked: no braces, no semicolons, and
indentation is purely cosmetic (tabs are the convention).

Authoritative docs (read these when a detail matters):
- Language manual: `https://github.com/JoeStrout/miniscript2/blob/main/docs/MiniScript-Manual.pdf`
- What changed from 1.x: `https://github.com/JoeStrout/miniscript2/blob/main/docs/NewInMS2.md`
- Command-line extras: `https://github.com/JoeStrout/miniscript2/blob/main/docs/CommandLine.md`
- Bundled modules: `https://github.com/JoeStrout/miniscript2/tree/main/lib`

## This project

- Entry point: `src/main.ms` (set by `ENTRY_POINT` in `.env`; defaults to `src\main.ms`).
- Run it with `run.bat` from the repo root. It reads `.env`, checks
  `MINISCRIPT_EXE_PATH`, and runs `"<exe>" "<entry point>"`.
- The interpreter is not assumed to be on `PATH`. Use `run.bat` rather than calling
  `miniscript` directly.
- Ad-hoc checks: `"<exe>" -c "print 6*7"`, or `"<exe>" -i src/main.ms` to drop into the
  REPL with the script's globals still loaded.
- `test/` is currently empty; there is no test harness. Verify behavior by running the
  program and printing what you need.

## A first look

```miniscript
// FizzBuzz
for i in range(1, 15)
	s = "Fizz" * (i % 3 == 0) + "Buzz" * (i % 5 == 0)
	if s == "" then s = str(i)
	print s
end for
```

Booleans are numbers: `true` is 1 and `false` is 0, so multiplying by a comparison is a
normal MiniScript idiom.

## Syntax rules that trip people up

- One statement per line. Use `;` only to squeeze several statements onto one line.
- Blocks open with `if`, `for`, `while`, or `function` and close with `end if`,
  `end for`, `end while`, `end function`.
- Short-form `if` takes exactly one statement per branch:
  `if x == null then x = 1` and `if x >= 0 then print "pos" else print "neg"` (no `end if`).
- A statement continues onto the next line when the last token before any comment is an
  opening `(`, `[`, or `{`, a comma, or a binary operator.
- No parentheses around `if`/`while` conditions.
- Omit `()` when calling a function with no arguments: `print`, `f.close`, `lst.sort`.
- Omit parentheses around arguments when the call is the first thing on the statement:
  `print x`, `f 1, 2`, `lst.push 4`.
- Identifiers are case-sensitive. `Print` is not `print`.
- Comments are `//`, to end of line. A first line starting with `#!` is ignored.
- Do not begin identifiers with `_`; those names are reserved for the host. (The REPL
  exposes `_in` and `_out`.)
- `^` is right-associative in 2.0: `2^3^2` is 512, not 64.
- `and`, `or`, and `not` are fuzzy logic on numbers, not boolean/bitwise operators:
  `a and b` is `a*b` clamped to [0,1], `a or b` is `a+b-a*b` clamped, `not a` is
  `1-abs(a)` clamped. Use `bitAnd`/`bitOr`/`bitXor` for integer bit operations.

## Truthiness

`0` is false and every other number is true. Empty strings, lists, and maps are false;
non-empty ones are true. `null` is always false. Comparisons produce 1 or 0.

## Variables and scope

Variables are dynamically typed and local by default: assigning inside a function creates a
local. Reach outside explicitly:

- `globals.x` — the global map (`globals` is also the whole global map).
- `locals.x` — the current call frame's locals.
- `outer.x` — the enclosing function's locals, for nested functions.
- `self`, `super` — object and prototype, inside methods.

```miniscript
count = 0
bump = function
	"Adds one to the global count."
	globals.count = globals.count + 1
end function
```

MiniScript 2 rejects `count = count + 1` inside that function at compile time
(`illegal assignment to unqualified local 'count' based on nonlocal`) when `count` is not
already a local. It also rejects the implicit form, `count += 1`. Qualify the left side
with `globals.` (or use `locals.`), or read the global into a differently named local.

Math-assignment forms exist for `+ - * / % ^`: `x += 1`, `globals.x *= 5`.

## Data types

Numbers, strings, lists, maps, functions, `null`, and (new in 2.0) `error`. Classes and
objects are maps.

### Numbers

All numbers are doubles. Printing follows fixed rules: `NaN`, `Inf`, `-Inf` print as shown;
integers with magnitude ≤ 2^53 print without a decimal point; `-0` prints `0`; non-integers
with magnitude > 1E10 or < 1E-6 use scientific notation; other non-integers print with 1–6
decimal places, trailing zeros trimmed.

### Strings

Immutable, UTF-8/Unicode code-point indexed. Literal quotes are doubled:
`"She said ""hi""."`.

| Operator | Meaning |
|---|---|
| `s + t` | concatenate (`"a" + null` is `"a"`) |
| `s - t` | chop: remove trailing `t` if present |
| `s * n` | repeat `s` (fractional repeats allowed) |
| `s / n` | equivalent to `s * (1/n)` |
| `s[i]` | character at index, negative counts from the end |
| `s[:n]`, `s[n:]`, `s[n:m]` | slices, end exclusive |
| `s == t`, `s < t`, ... | case-sensitive comparison |

`char(i)` and `.code` convert between code points and characters. `s.len` counts code
points, so an emoji counts as one.

String methods: `.len`, `.upper`, `.lower`, `.indexOf(s, after=null)`, `.insert(index, s)`,
`.remove(s)`, `.replace(old, new, maxCount=null)`, `.split(delimiter=" ", maxCount=null)`,
`.hasIndex(i)`, `.indexes`, `.values`, `.val`, `.code`.

```miniscript
parts = "a,b,c".split(",")
print parts.join("-")          // a-b-c
print "MiniScript".upper       // MINISCRIPT
```

### Lists

Mutable, ordered, zero-based. `[1, 2, 3]`, concatenation with `+`, replication with `*`,
and the same slice syntax as strings.

Methods: `.len`, `.push(x)`, `.pop`, `.pull`, `.insert(index, value)`, `.remove(i)`,
`.indexOf(x, after=null)`, `.hasIndex(i)`, `.indexes`, `.values`, `.join(delimiter=" ")`,
`.sort(key=null, ascending=1)`, `.shuffle`, `.sum`, `.replace(old, new, maxCount=null)`.

`push`, `pop`, `pull`, `insert`, `remove`, `sort`, and `shuffle` mutate in place. `pop` is
stack-like (last), `pull` is queue-like (first). `range(x, y=0, step=null)` is inclusive of
both ends: `range(1, 5)` is `[1, 2, 3, 4, 5]`, and `range(10, 1)` counts down.

Lists from `range` and `[x] * n` are stored in computed form, so
`for i in range(1, 1000000)` does not build a million-element list.

### Maps

Key-value pairs with any value as a key. `{"host": "localhost", "port": 8080}`. Access with
`m["key"]` or `m.key` (dot form requires an identifier-shaped string key). Keys are unique;
reassigning a key replaces its value. Iteration order is insertion order, including after
removals (a re-added key moves to the end).

Methods: `.len`, `.hasIndex(x)`, `.indexes`, `.indexOf(value, after=null)`, `.values`,
`.push(x)` (set to 1, i.e. use as a set), `.pop`, `.remove(x)`, `.replace`,
`.shuffle`, `.sum`.

```miniscript
for kv in m
	print kv.key + " = " + kv.value
end for
```

### Frozen values

`freeze x` marks a list or map (recursively) immutable; `frozenCopy(x)` returns a frozen
copy, or `x` itself when already frozen; `isFrozen(x)` tests it. Mutating a frozen value is
a runtime error. A list or map used as a map key is stored as a frozen copy automatically.
List/map default parameter values are frozen too. The core type maps (`list`, `string`,
`map`, `number`, `funcRef`) are never frozen.

## Control flow

```miniscript
if n < 0 then
	print "negative"
else if n == 0 then
	print "zero"
else
	print "positive"
end if

for i in range(1, 100)
	if i == 42 then continue
	if i > 50 then break
end for

while not done
	done = step
end while
```

`for` iterates lists, strings (characters), and maps (`kv.key` / `kv.value`). `break` exits
only the innermost loop; `continue` skips to the next iteration.

## Functions

```miniscript
add = function(a, b=2)
	"Adds two numbers."          // first constant string is the docstring
	return a + b
end function

print add(1)                     // 3
details = info(@add)
print details.note               // Adds two numbers.
```

- Parameters may have defaults; defaults are evaluated at compile time, so `6*7` and
  literal lists/maps are allowed (maps/lists are frozen automatically). Missing arguments
  become `null`.
- `@f` refers to the function itself; a bare identifier invokes it.
- `function ... end function` is an expression, so functions can be passed or returned
  inline.
- Functions defined by assignment get a `name`, and their first-line string becomes their
  `note`; `info(@f)` reports `type`, `name`, `note`, `params`, `closure`, `sourceLoc`.
- Nested functions see the enclosing function's locals; assign to them via `outer.name`.
- Functions can be added to the built-in type maps to extend them, e.g.
  `string.capitalized = function; ...; end function`, where `self` is the string.

## Classes and objects

Prototype-based: a class is a map, `new` sets `__isa` to the prototype, and lookup walks
the chain. `isa` walks the same chain. Methods invoked with dot syntax get `self`;
`super.method` calls the parent implementation with `self` unchanged.

```miniscript
Shape = {"sides": 0}
Shape.degrees = function
	return 180 * (self.sides - 2)
end function

Square = new Shape
Square.sides = 4
x = new Square
print x.degrees                  // 360

MagicSquare = new Square
MagicSquare.degrees = function
	return super.degrees + 42
end function
print (new MagicSquare).degrees  // 402
```

Setters intercept assignment: name the method `"prop="` and use the bracket form (the `=`
is not legal in an identifier). `super.prop = value` performs the ordinary store into
`self`; returning an error rejects the assignment and halts; assigning `null` instead of a
function makes the property read-only.

```miniscript
Counter = {"n": 0}
Counter["n="] = function(value)
	if value < 0 then return err("Counter cannot go negative")
	super.n = value
end function

c = new Counter
c.n = 5
print c.n                        // 5
```

`new A` binds tighter than arithmetic and unary minus; `@A`, `^`, indexing, calls, and
dot all bind tighter still. Dot syntax also works on numeric literals
in 2.0: given `number.double = function; return self * 2; end function`, `4.double` is 8.

## Errors are values

`err(message, inner=null)` creates an `error`; `e.message`, `e.inner`, and `e.stack` expose
its details. Errors propagate through operators, indexing, and most intrinsics (`e + 5` is
`e`), but `e or fallback` yields `fallback`. Specialize a base error by calling `.err` on it
and test with `isa`:

```miniscript
NetErr = err("network trouble")
e = NetErr.err("host unreachable")
print e isa NetErr               // 1
print e isa error                // 1
```

An error that is never caught halts the program, printing `Runtime Error: ...`. That
includes a bare expression statement evaluating to an error (`file.readLines(path)` alone on
a line), `if e then`, `while e`, and reading any member other than `message`, `inner`,
`stack`, or `__isa`. Assign the result and check it:

```miniscript
lines = file.readLines(path)
if lines isa error then
	print "Could not read: " + lines.message
	return
end if
```

Operators on unsupported types now return errors instead of `null`: `1 + null`,
`[1, 2] + 3`, and `1 < "42"` are all errors. `==` and `!=` still compare any two values;
`<`, `<=`, `>`, `>=` work only between two numbers or two strings.

## Intrinsics

Dot-syntax intrinsics can also be called as globals with the receiver first: `s.len` and
`len(s)` are equivalent.

Numbers/math: `abs`, `acos`, `asin`, `atan(y, x=1)`, `ceil`, `floor`, `log(x, base=10)`,
`pi`, `range`, `round(x, d=0)`, `rnd(seed=null)`, `sign`, `sin`, `cos`, `tan`, `sqrt`,
`bitAnd`, `bitOr`, `bitXor`, `char`, `str`.

Strings/lists/maps: see the method lists above, plus `slice(x, from, to)`.

Errors: `err(msg, inner=null)`, `e.err(msg, inner=null)`.

System: `globals`, `locals`, `intrinsics`, `print(x, delim)`, `input(prompt)`,
`refEquals(a, b)`, `stackTrace`, `time`, `wait(x=1)`, `yield`, `freeze x`,
`frozenCopy(x)`, `isFrozen(x)`, `info(x)`, `gc`.

## Command-line MiniScript extras

- `shellArgs` — frozen list of arguments: script path first, then arguments (with `-c`,
  `shellArgs[0]` is `"-c"`).
- `env` — environment variable map; changes propagate to `exec`.
- `exec(cmd, timeout=30)` — returns `{output, errors, status}`; timeout gives status 124.
- `exit(resultCode=0)` — exit with a status.
- `file` — `curdir`, `setdir(path)` (or `file.curdir = path`), `makedir`, `children`,
  `name`, `parent`, `child`, `exists`, `info`, `copy`, `move`, `delete`, `readLines`,
  `writeLines`, `loadRaw`, `saveRaw`, `open(path, mode="r+")`.
- `FileHandle` — `write`, `writeLine`, `read(byteCount=-1)`, `readLine`, `atEnd`,
  `position` (assignable), `seek`, `isOpen`, `close`. Modes are `"r"`, `"w"`, `"a"`,
  `"r+"`.
- `RawData` — byte buffer: `len`/`resize`, `littleEndian`, `byte`/`setByte`,
  `sbyte`/`setSbyte`, `ushort`/`setUshort`, `short`/`setShort`, `uint`/`setUint`,
  `int`/`setInt`, `float`/`setFloat`, `double`/`setDouble`, `utf8`, `setUtf8`.
- `key` — `key.available`, `key.get`, `key.raw` for single-keystroke input.
- `gc` — `gc.collect`, `gc.stats`.
- REPL: `_in`/`_out` history and `reset`; `!help`, `!?`, `!N` metacommands.

`import "name"` runs `name.ms` and stores its globals in a map variable named `name`;
`importUtil.ensureImport` imports only once. Search order is the script's folder, its
`lib` subfolder, then the `lib` folder next to the executable; override with
`MS_IMPORT_PATH` (`:`/`;`-separated, `$VAR` expansion).

Bundled modules: `dateTime`, `grfon`, `importUtil`, `json`, `listUtil`, `mapUtil`,
`mathUtil`, `matrixUtil`, `qa`, `stringUtil`, `tsv`, `vt`. Each source file documents its
own API; `listUtil`/`stringUtil`/`mapUtil` add methods to the built-in types, so importing
one is often enough to use them.

## Debugging

- `print` with labels, or use `str()` on values.
- `info(x)` for type and metadata; `info(@f)` for a function's name/note/params.
- `stackTrace` for the current call stack; an error's `.stack` for where it was created.
- `"<exe>" -d script.ms` prints compiled code and VM diagnostics; `-i` leaves you in the
  REPL after the run.
- In the REPL, `_out[n]` reuses an earlier result and `reset` clears globals and history.

## MiniScript 2 gotchas

- `^` is right-associative.
- Unqualified assignment that mixes a local and a nonlocal is a compile error; qualify it.
- `super.x = v` stores into `self`, not into the prototype.
- Operators on unsupported types return errors instead of `null`.
- Strings count code points on every platform.
- Frozen lists/maps reject mutation with a runtime error; `shellArgs` and list/map defaults
  are frozen.
- Map iteration is insertion-ordered.
- An uncaught error value stops the program; catch it with `isa error` or `or`.
- Reading a key that a map does not have is a fatal `Key Not Found` runtime error, not
  `null`. Test first with `m.hasIndex("key")`, or fill optional keys in when the map is
  built (`if not m.hasIndex("x") then m.x = 0`).
- A multi-line map or list literal only continues the statement when its last entry ends
  with a comma. Leave it off and the close brace on the next line ends the statement, and
  you get `Compiler Error: Expected '}' after map entries`.
- A multi-line *call* has to close on the last argument's line. Continuation needs a comma
  or operator as the last token, so a `)` on its own line ends the statement early
  (`Compiler Error: Expected ')' after arguments`), and a trailing comma before the `)`
  does not rescue it (`Compiler Error: Unexpected token: )`). Note the contrast with a map
  or list literal, where a trailing comma is what lets the closer start a new line:
  `rule(spo(var("x"), "child", var("y")),` newline `spo(var("y"), "parent", var("x")))`.

## Style

- Tabs for indentation, one statement per line, `//` comments.
- Lowercase variables and functions; capitalized class names (`Shape`, `Counter`).
- Put a one-line docstring as the first line of non-trivial functions.
- Prefer the built-in operators over manual loops (`"Fizz" * (i % 3 == 0)`).
- Build strings from lists with `split`/`join` rather than repeated concatenation.
- Extend built-in types sparingly; keep host-reserved `_`-prefixed names out of your code.
