# Joint Analysis

## 1. Data type tradeoffs

### Dict in Python

Language and built-in feature:
A python dictionary handles key-value pairs storage, hashing,collision handling and resizing. It stores every entry as a (cached hash, key and value), it only holds pointers to its values. It puts them in resizable boxes so it resizes itself as it grows. It also resizes to keep lookups constant time. 

Our implementation and Comparison:
Our implementation is that our dt_map stores those dt_value inline in each node with a copied key and a next pointer. It also has a fixed 16 bucket with separate chaining which means that it never resizes. Since it is also unboxed which means that it uses less memory per entry but since it never resizes, lookup is slowed down as the map grows. We traded the automatic resizing and O(1) deletion for less memory per entry and a simpler structure. We also keep a side array for insertion order


When the difference matters:
When the program inserts and looks up many entries, since python resizes so the chaining stays short and lookups stay on O(1) but ours do not.



### Struct in C

Language and built-in feature:
C's own `struct`, a compile-time declaration that gives each field a fixed offset chosen by the compiler.

Comparison with our implementation:
`person.age` compiles to one load from a fixed offset. The exact offset depends on field order and alignment. Our `dt_record_get` walks a copied field-name array with `strcmp`, up to 8 comparisons. We also allocate storage for the copied names, which the C struct doesn't have. We trade compile-time type checking for runtime field enumeration via `dt_record_field_name`.

When the difference matters:
A loop reading `person.age` many times. The struct does one load per read. Ours does a short `strcmp` walk per read. The compiled field offset favors repeated access. Runtime field names favor code that must print, serialize, or build a record from a config file, since the C struct cannot list its own fields at runtime.


### Python str

Language and built-in feature:
Python's str is a built-in immutable text object that already provides a stored length, memory management, and full Unicode handling. Each string is an object with a header of tens of bytes, and its characters are packed at 1, 2, or 4 bytes each depending on the widest code point it contains.

Comparison with our implementation:
Our dt_str is a raw byte buffer with only a length and capacity beside it. Both make len a field read, and both can hold a \0 byte. We trade the cached hash, immutability, and Unicode generality for a smaller, mutable buffer.


When the difference matters:
It is noticeable when building a string piece by piece, it is because Python strings are immutable, so appending with s += t allocates a new string and copies both parts each time and it's the reason why python uses "".join(). It is also noticeable when changing characters in place since Python has no s[i] = c just to edit one character, you have to rebuild the whole string with s = s[:i] + c + s[i+1:] which copies all n bytes for a single-character change.  One major difference is that our dt_str is mutable, our dt_str_append doubles the buffer and writes in place for amortized O(1) appends, at the cost of up to half the buffer sitting empty as slack and holds a mutable buffer, so editing one byte does not need actually rebuilding the whole string since we carry both the buffer and the capacity together


## 2. Tagged unions

### Read a member with no tag check.
- Because the union and the tag are public fields in dt.h, any caller can write v.as.string on a value tagged DT_INT and it compiles cleanly, which should not be allowed. The readers are just a label and are not enforced so nothing stops a caller from bypassing them.

### Put a value into a state that contradicts itself.
- Nothing prevents me from setting tag = DT_STR while its value is actually an integer. The value no longer matches what it actually holds. In other languages such as Rust, Swift, and ML, this combination is not possible. The data and its tag are inseparable and must stay consistent.

### Let a missing case slip through.
- The switch on the tag is not strictly enforced. If you were to add an 11th tag to dt_tag, the printer's switch would still compile and the new case would fall through unhandled, caught only if a warning like -Wswitch happens to flag it, since there's no default to fall back on.

### Read the bytes as something else on purpose.
- This is used in low-level code, since it lets you read the same bytes through a different union member for purposes such as serialization, float-bit tricks, NaN-boxing, and tagged pointers. It becomes dangerous when you act on that reinterpreted value as if the bytes were really that type. One example is dereferencing an int as a pointer.

### Is it worth wanting?
- Mostly no. In the example 'as str 42', it hands the printer an integer where a pointer is expected, which is undefined behavior, not just an error. The same happens the other way which is 'as array' on an integer value would call dt_array_len on a garbage pointer and crash.

- The thing that is useful here is freedom. The freedom to control the memory layout and when the checks run. But that is a job better left to the compiler than handled by hand. One thing that is also useful here is the low-level escape hatch, a deliberate way to bypass the safety rules when you have a good low-level reason to. It is useful in systems code such as allocators, kernels, VMs, embedded targets, and binary formats, where you need exact layout control and byte reinterpretation. Everywhere else, the Rust-style enforced check is the better option.


## 3. Map insertion order

### Alternative Design
- Drop the order array, including the order capacity. We would keep only the buckets, chains, and the count. Iteration would traverse the buckets and their chains directly.

### What breaks
- The insertion order is lost, since with no order array, iterating the buckets returns keys in hash order, which is effectively random to the caller and shifts if the hash function or bucket count ever changes.

- For example, inserting alpha, beta, gamma could print as gamma, alpha, beta. The output is no longer reproducible in any way the caller controls, so cases like map_basics that expect insertion-order printing would fail. dt_map_key_at(index) also loses its O(1) access, since reaching the i-th key now means walking the buckets and counting, going from O(1) to O(N).

### Trade-off
- It saves memory and simplifies the implementation, and removal goes from O(N) to O(1), but all of it is traded for predictable output.

### Would we ship it
- No. Even though it saves memory, simplifies the implementation, and is not needed for lookup, the order is part of what makes printing predictable and testable. The memory saving and the O(1) removal are a good tradeoff, but they are small next to losing reproducible output. The extra memory is a reasonable cost for this predictability.



## 4. Released references and leaks

**In a long-running server:**
Use-after-release means `dt_ref_borrow` reads through `p->cell` after
`dt_ref_release` freed it. The freed block can be reused by another
request, so the read may return another request's data, or crash if
the bytes are not a valid `dt_value`. The damage can cross requests.
An unreleased reference leaks its cell and anything it points at. A
server that never exits accumulates leaks until it runs out of memory.

**In a CLI tool that exits in a second:**
Both failures are still present. Use-after-release is still undefined
behavior. It can crash, return stale bytes, or expose data if the freed
block is reused before the read. A short process limits how long the
process has to trigger it, not whether it is safe. The leak sweep still
calls `dt_ref_is_released` and reports `DT_ERR_LEAK`. Exit reclaims the
leaked cell, but the leak still consumed resources while the program
ran, and CI sanitizers may catch it before exit.

**The difference:**
The server's long lifetime lets both failures accumulate: leaked cells
compound and freed memory can carry data across requests. The CLI's
short lifetime limits how long a leak persists. It does not make the
leak free, and it does not make use-after-release safe. The CLI just
has less time to manifest the consequences.
