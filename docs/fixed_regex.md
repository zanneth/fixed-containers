# FixedRegex

Include `fixed_containers/fixed_regex.hpp`. `FixedRegex` owns a compiled regular
expression in fixed storage. It does not delegate to `std::regex`. Compilation,
matching, searching, result formatting, replacement, and iteration do not allocate
with the default traits and checking policy. These operations can also be evaluated
at compile time.

```cpp
#include "fixed_containers/fixed_regex.hpp"

using namespace fixed_containers;

constexpr FixedRegex<64> EXPRESSION{R"(([a-z]+)=([0-9]+))"};
static_assert(regex_match("answer=42", EXPRESSION));

FixedMatchResults<const char*, 2, 80> matches;
if (regex_search("answer=42;", matches, EXPRESSION))
{
    // Captures are iterator pairs referring to the original input.
    // str() makes an owning FixedString<80> only when requested.
    auto key = matches.str(1);    // "answer"
    auto value = matches.str(2);  // "42"
    auto position = matches.position(); // 0
}

auto replaced = regex_replace<80>("answer=42;", EXPRESSION, "$1: $2");
// FixedString<80>{"answer: 42;"}
```

## Capacities and errors

The expression template parameters are:

```cpp
FixedRegex<MAXIMUM_PATTERN_LENGTH,
           MAXIMUM_STACK_SIZE = 1024,
           Traits = FixedRegexTraits,
           Checking = customize::RegexAbortChecking,
           MAXIMUM_OPERATIONS = 1'000'000>
```

The pattern limit counts bytes, including explicitly supplied embedded nulls.
Pattern storage and the compiled program are embedded in the expression. Copies
own independent programs and remain valid after the original is destroyed or
reassigned. The default type is trivially copyable and a structural type.

The matcher uses a local array for its backtracking choices and capture/repetition
undo records. It does not recurse with input length. The stack capacity counts
records, not characters: a repeated capturing group can require several records
per iteration. Input length is not separately bounded, but a match can exhaust
this workspace. Increasing the capacity increases automatic storage use; size it
for the workload and the application's available thread stack. The parser's
recursion depth is bounded by the pattern size.

The operation limit counts matching instructions and backtracking work across
all candidate start positions in one search. It prevents an exponential
backtracking expression from running indefinitely. Iterator increments perform
separate searches with separate limits. This is a work budget, not a wall-clock
deadline.

`FixedMatchResults<Iterator, MAXIMUM_CAPTURES, MAXIMUM_STRING_LENGTH = 256>` stores
the whole match plus up to `MAXIMUM_CAPTURES` capturing groups. The string capacity
limits owning results from `str()` and `format()`, not the length of the input or
the captured iterator ranges. Replacement's return-value overload has an
independent output capacity, defaulting to 256. Its output-iterator overload
writes to caller-supplied storage and does not impose an output length limit.

Invalid expressions and resource exhaustion call
`Checking::regex_error(std::regex_constants::error_type, source_location)`.
The default policy aborts, as do the library's other default checking policies.
A custom policy must not return. It may throw in applications that enable
exceptions; assignment commits the new expression only after compilation succeeds.

| Condition | Error |
| --- | --- |
| Pattern/program or capture-result capacity exhausted | `error_space` |
| Matching workspace exhausted | `error_stack` |
| Matching work limit exhausted | `error_complexity` |
| Invalid expression | Corresponding standard syntax error code |

`FixedString`'s checking policy handles owning output overflow. Capacity and
complexity failures are errors, never reported as an ordinary unsuccessful match.

## Operations and syntax

The constructors, assignment operations, `assign`, `mark_count`, `flags`, `imbue`,
`getloc`, and `swap` follow the `std::regex` operation names. Pattern arguments can
be C strings, pointer/count pairs, strings, string views, initializer lists, or
iterator pairs. `regex_match` and `regex_search` accept bidirectional iterator
ranges, C strings, and strings, with or without results. String views and
`FixedString` inputs are also supported. Results and iterators borrow their input;
keep it alive and unchanged while using them. Overloads reject temporary owning
strings when results would retain their iterators.

The six standard syntax flags are recognized: `ECMAScript` (the default), `basic`,
`extended`, `awk`, `grep`, and `egrep`. Features include alternatives, capturing
groups, character classes and ranges, negated classes, POSIX named classes,
collating names and equivalence classes, anchors, and bounded/unbounded repetition.
ECMAScript additionally provides non-capturing groups, lazy quantifiers, word
boundaries, positive/negative lookahead, and its character escapes. Backreferences
are available in ECMAScript, basic, and grep syntax. Like libc++, extended and
egrep also accept single-digit backreferences as an extension. POSIX syntaxes select a
leftmost longest match; ECMAScript selects the first successful ordered alternative.

Use `std::regex_constants` or its `fixed_containers::regex_constants` alias for
syntax and match flags. `icase`, `nosubs`, `collate`, and ECMAScript `multiline`
are supported. `optimize` is accepted as a hint. Replacement supports the standard
ECMAScript and sed formatting rules, `format_no_copy`, and `format_first_only`.

```cpp
constexpr FixedRegex<8> SEPARATOR{"[,;]+"};
constexpr std::string_view INPUT = "one,two;;three";

for (FixedRegexIterator it{INPUT.begin(), INPUT.end(), SEPARATOR};
     it != std::default_sentinel; ++it)
{
    auto separator = it->str();
}

for (FixedRegexTokenIterator it{INPUT.begin(), INPUT.end(), SEPARATOR, -1};
     it != std::default_sentinel; ++it)
{
    auto word = it->str();
}
```

`FixedRegexIterator<Iterator, RegexType, StringCapacity = 256>` and
`FixedRegexTokenIterator<Iterator, RegexType, SubmatchCapacity = 16,
StringCapacity = 256>` support deduction from constructor arguments. Token
selection accepts an integer, an initializer list, an array, or an input range
of integers. The selector `-1` emits unmatched text. Copies of a token iterator
refer to their own stored results. A default-constructed iterator is the end
iterator; `std::default_sentinel` is also accepted. The expression must outlive
its iterators. Zero-length matches advance according to the standard iterator rules.
As with the standard regex iterators, `iterator_category` is forward but the
C++20 `iterator_concept` is input: dereferencing refers to the iterator's own result.

## Allocation-free compatibility boundary

The API uses `fixed_containers` algorithms and fixed-capacity companion types;
it cannot be passed to the existing `std::regex_*` algorithms or use
`std::match_results` for output. Owning result strings are `FixedString`, and
allocator accessors are unnecessary. Error handling uses a checking policy instead
of requiring exceptions.

The character type is `char`, like `std::regex`. Matching operates on 8-bit code
units, not decoded Unicode characters. The default traits use the classic ASCII
character classes. They own a `FixedRegexLocale` value containing 256-entry tables
for case folding, character classes, collation, and primary equivalence. The
default primary-equivalence table folds ASCII case. `imbue`
accepts this value type, not a heap-backed `std::locale`; it invalidates the compiled
expression until the next assignment. The tables can be customized without heap
allocation. Collating names supplied by the default traits designate single bytes.

An alternative traits type can implement the `regex_traits` protocol with its own
value locale and bounded strings. Character classes are compiled to byte sets;
collating names must resolve to single bytes. Caller-provided traits, checking
policies, input/output iterators, and output streams must themselves avoid
allocation if the entire calling operation must remain allocation-free.

Tests compare standard operations against `std::regex` and also check specified
ECMAScript behavior directly. For example, a skipped capture's backreference
matches an empty string, and captures inside a repeated atom are cleared before
each iteration. Some STL implementations differ on these cases; this library
follows [the ECMAScript matching rules](https://262.ecma-international.org/5.1/#sec-15.10.2.5)
used by [the C++ regex grammar](https://eel.is/c++draft/re.grammar).

The [ported libc++ suite](../test/libcxx_regex/README.md) adds 147 executable tests
and four compile-failure tests. Its inventory documents unsupported APIs and
intentional differences from libc++ expectations, including the unresolved
`match_prev_avail` anchor semantics. Long repetitions of a single character,
dot, or character class use one resumable choice instead of one entry per byte;
other backtracking and operation limits still apply.
