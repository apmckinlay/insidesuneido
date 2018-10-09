- [Introduction](#introduction)
- [History](#history)
  - [Past](#past)
  - [Present](#present)
  - [Future](#future)
- [Motivation](#motivation)
- [Implementations](#implementations)
- [Source Code Layout](#source-code-layout)
- [Coding Style](#coding-style)
- [Tests & Benchmarks](#tests--benchmarks)
- [Portable Tests](#portable-tests)
- [Language](#language)
  - [Values](#values)
  - [Numbers](#numbers)
    - [Decimal Floating Point](#decimal-floating-point)
    - [Integers](#integers)
  - [Strings](#strings)
  - [Exceptions](#exceptions)
  - [Functions & Methods](#functions--methods)
    - [Arguments](#arguments)
    - [Parameters](#parameters)
    - [Passing Arguments to Parameters](#passing-arguments-to-parameters)
- [Compiler](#compiler)
  - [Interpreter](#interpreter)
  - [Concurrency](#concurrency)
- [Database](#database)
  - [Storage](#storage)
  - [Shutdown & Startup](#shutdown--startup)
  - [Recovery](#recovery)
  - [Packing](#packing)
  - [Records](#records)
  - [Indexes](#indexes)
  - [Queries](#queries)
    - [Query Optimization](#query-optimization)
  - [Transactions](#transactions)
  
# Introduction

This document explains the high level design and implementation of Suneido, including cSuneido (C++), jSuneido (Java), gSuneido (Go), suneido.js (JavaScript).

This is not a guide or reference to using Suneido. See the Users Manual for that.

This is a work in progress. It will likely never be "complete". But it may be useful to maintainers (including me) and might be of interest to people implementing or interested in this kind of software.

# History

## Past

The original implementation of Suneido was in C++. The first open source release (on SourceForge) was in 2001.

Suneido grew out of several earlier projects - Lucid (trademark long since sold) was a simple personal information system written for CPM and MS-Dos (remember those?). From selling that product we ended up doing custom programming based on it. Which led to C4 (fourth generation language written in and based on C). C4 expanded on Lucid, adding a language and more of a database. This was around the time that Smalltalk and object-oriented programming started. So C4's language was object-oriented, although it had a C-like syntax. The language was also influenced somewhat by Lisp and Scheme. C4 was originally called Lucid-86 (copying Smalltalk-80's naming) so I must have started on it sometime around 1986. Although it used a mouse and pull-down menus (cutting edge!) it was still text based.

After a number of years using C4 to develop custom software for clients I was ready to do something better and started working on Suneido. C++ was just becoming available and I could finally use an object-oriented programming language to implement Suneido.

Axon decided to open source the project, partly to give back to the community, and partly in hopes a getting contributions. The open source project was quite active for a number of years with multiple contributors. But as other languages like Ruby became dominant and the web took over, the community dwindled. (Suneido was and mostly remains a desktop client-server platform, not web based.)

However, Axon, the parent company of Suneido, uses Suneido to implement its commercial software so Suneido has continued to be used and developed.

I made the decision early on to use garbage collection with C++. At first I wrote my own series of garbage collectors, getting increasingly more sophisticated. The final version used bitmaps for allocation and mark-sweep. It was replaced by the Boehm-Demers-Weiser garbage collector. This is conservative garbage collection which occasionally leads to spurious retention of memory, but works well the majority of the time. I think using garbage collection was a good decision. I would not want to implement something like Suneido without it.

The Java implementation was started in 2008 and went into production roughly three years later in 2011, although there was substantial work on it after that, especially on the database.

The main motivation for jSuneido was multi-threading so the server could handle more clients. cSuneido uses Windows fibers, cooperative multi-tasking, but it is essentially single threaded. (The big advantage of this is that there are almost no concurrency issues to deal with.) At the time, Java had the best concurrency support, albeit the traditional error prone threads + locks + shared memory.

The database implementation was redesigned for jSuneido, again largely to handle concurrency better.

I started playing with a Go implementation (gSuneido) in 2014. The advantages of Go over Java are that it doesn't require a large runtime and that it allows programming at a lower level, while still including garbage collection and concurrency support. The main drawback (for implementing Suneido) is that it does not include a VM with a JIT compiler like Java has. The main advantage of Go over C++ (for me, for implementing Suneido) is that Go includes garbage collection and it's "safe" (e.g. no unchecked pointers or array accesses).

The JavaScript implementation (suneido.js) was started in 2015. Unlike gSuneido, the goal was not to reimplement all of Suneido in JavaScript. The database especially would not make sense to implement in JavaScript. Instead, we would still use jSuneido as the server and transpile Suneido code to JavaScript to run on the browser. Transpiling is largely complete, although not all the built-in functions. The big remaining challenge is to figure out how to "port" the Suneido user interface (currently Win32 specific) to run on the browser, while minimizing the amount of work to port our existing large application code base. One of the challenges is that the current design is quite "chatty", making a lot of client-server requests. This is reasonable on a local area network, but probably not over the internet because latency will be much higher.

## Present

Axon is currently using jSuneido for all production servers, with cSuneido for the desktop clients. The cSuneido database implementation is still maintained but we only use it for development.

## Future

Looking forward, one possible path would be to replace the back end (currently jSuneido) with gSuneido and the front end (currently cSuneido) with suneido.js

# Motivation

There are two sides to "why". Why did I write Suneido? And why did Axon (Suneido's parent company) implement their commercial software with Suneido.

# Implementations

**_cSuneido_** - C++

- Conservative garbage collection with Boehm-Demers-Weiser
- Has its own byte code and interpreter
- 32 bit
- Single threaded but multi-tasking with Win32 fibers
- All Suneido values inherit from SuValue

**_jSuneido_** - Java

- Compiles Suneido code to JVM byte code
- Multi-threaded
- Used for production servers
- Uses bare JVM types for integers and strings

**_gSuneido_** - Go

- Incomplete, work in progress
- Has its own byte code and interpreter
- Suneido values (including integers and strings) implement the Value interface
- Uses panic and recover to implement exceptions
- Aiming to work in both 32 and 64 bit

**_suneido.js_** - JavaScript

- Incomplete, work in progress
- Transpiles from Suneido code to JavaScript, currently (2018-01-30) using AST from jSuneido parser
- Does not include the database
- Written in TypeScript
- One challenge is to somehow run existing user interfaces (which are win32 specific and assume low latency)
- Another challenge is that current code assumes blocking/synchronous, but browsers want async.
- Although it's not specifically targeted, most of the development is with V8 with node.js

# Source Code Layout

**_cSuneido_**

cSuneido does not use subdirectories for source code.

Source files starting with an "su" prefix generally implement things that are visible to the user, such as data types and built-in functions and classes.

**_jSuneido_**

- **code** - compiled Suneido code goes in this package
- **compiler** - lexing, parsing, and byte code generation
- **database**
  - **immudb** - current storage engine used by database
  - **query** - parsing and execution of queries
  - **expr** - expressions for where and extend
  - **server** - client/server communications
- **runtime** - run-time support code
  - **builtin** - Suneido built-in functions and classes
- **util** - code that is not specific to Suneido

**_gSuneido_**

**_suneido.js_**

# Coding Style

**_cSuneido_**

Use `<cstdint>`

Use `int64_t` instead of `long long`

Avoid “long” because of confusion over whether it is 32 or 64 bits. On Win32 long is 32 bits but in jSuneido/Java long is 64 bits. Use int or int32_t or int64_t.

Code is formatted with clang-format. (I have Visual Studio set up to format on save.)

**_jSuneido_**

**_gSuneido_**

**_suneido.js_**

# Tests & Benchmarks

**_cSuneido_**

Uses its own simple test framework contained in testing.h/cpp.

Tests look like:

```
TEST(dnum_add)
    {
    assert_eq(...);
    }
```

The -t(est) command line option is used to run all the tests. It takes an optional prefix to run only the tests that match.

e.g. "suneido -test dnum" would run all the tests with names starting with "dnum"

Normally the first part of the test name matches the source file name.

Test results are displayed in an alert with more detail written to test.log.

Unfortunately, cSuneido's dependencies are so tangled that the only practical way to run tests is as part of the entire exe. Rather than compiling separate test and production executables, the tests are always included. Although this makes the executable slightly larger, it means you're testing the actual production version.

Visual Studio is normally configured to run all the tests after a successful build.

Benchmarks look like:

```
BENCHMARK(dnum_add)
    {
    while (nreps-- > 0)
        ...
    }
```

**_jSuneido_**

Uses jUnit.

It's not required, but it's nice to run Infinitest within Eclipse to run the tests automatically after every change. See infinitest.args and infinitest.filters

There is a simple benchmark function. This is normally called by a jUnit test but it will be skipped by Infinitest.

**_gSuneido_**

Uses the Go testing framework.

**_suneido.js_**

# Portable Tests

To reduce the duplication of tests in the different implementations, there are "portable tests". These tests are contained in text files in their own repository (<https://github.com/apmckinlay/suneido_tests>)

The files are designed to be read by the Suneido lexer/scanner and executed by "fixtures" that must be written for each implementation.

<http://thesoftwarelife.blogspot.ca/2014/05/portable-tests.html>

# Language

## Values

Suneido is a dynamically typed language. Variables may hold different types at different times.

**_cSuneido_**

SuValue (see suvalue.h/cpp) is the base class for SuString, SuNumber, SuObject, etc. that are normally allocated on the heap and referenced by pointer.

Value (see value.h/cpp) is a union of a pointer to an SuValue or a small immediate integer. (See Integers) Value delegates to SuValue.

**_jSuneido_**

For efficiency, some Java types are used directly, without being wrapped, e.g. Boolean, String, and Integer. SuValue is the base class for other types such as SuContainer.

Because there is no common base class or interface for values, operations (see ops.java) must do type checks (e.g. instanceof).

**_gSuneido_**

Values implement the Value interface (see value.go). Like Java, some types are used without wrapping (e.g. strings) but in Go we can define a new type based on a builtin type and support the Value interface. Therefore, fewer explicit runtime type checks are required.

**_suneido.js_**

## Numbers

Don't narrow results - too much overhead.

Divide handles even integer division specially. Running the ETA tests found roughly 1/3 of divides were even integer divisions. Benchmark went from 43 ns to 14 ns and the extra case doesn't seem to slow down normal usage.

Many things treat false and "" as zero. However, the first argument to ranges and string subscripts do not.

### Decimal Floating Point

Numbers in Suneido are stored in **decimal** floating point so that decimal fractions (like .1) can be represented exactly.

This means we can't use the **binary** floating point numbers normally provided by hardware and languages.

As of 2018-08 cSuneido, jSuneido, and gSuneido all use an implementation with a 64 bit coefficient and 8 bit sign and exponent. Only 16 of the possible 19 decimal digits are used. This allows splitting into two 8 decimal digit halves which fit into 32 bit for faster processing.  (cSuneido is 32 bit so it cannot take advantage of 64 bit instructions. In Visual C++ most 64 bit operations are subroutine calls.) The coefficient is kept “maximized” i.e. “shifted” left as far as possible.

*cSuneido* previously had an implementation of decimal floats that used four 16 bit integers, each holding four decimal digits (i.e. 0 to 9999) plus a sign and an exponent. The intent was to provide 16 digits of precision. However, because the exponent had 4 digit granularity, the actual precision could be as low as 13 digits.

*jSuneido* previously used Java BigDecimal. This worked fine, but BigDecimal is arbitrary precision which is overkill for Suneido.

Previously there were some minor issues with inconsistent results between *cSuneido* and *jSuneido* due to the different implementations and precisions. This should no longer be an issue - they should now give identical results.

Unfortunately, JavaScript does not have 64 bit integers, it only has binary floating point numbers, with bitwise operations defined as 32 bit.  (Internally, JavaScript implementations usually use integers.) To work around this, *suneido.js* uses JavaScript floats as 52 bit integers. This provides a little over 15 digits of precision - not quite the 16 that Suneido nominally provides.

[Bit Twiddling (Dnum)](https://thesoftwarelife.blogspot.com/2018/03/bit-twiddling.html)

### Integers

Although Suneido only has one kind of number (decimal floating point), for performance integers are internally handled separately.

**_cSuneido_**

The Value class holds (as a union) either a pointer to an SuValue, or a small integer. Integers are identified by have the high 16 bits set to 0xffff. This assumes there are no valid pointers with those bits set. So far that assumption has worked, but it might be safer to identify integers by using the low few bits which are more reliably zero for pointers due to alignment. Assuming 32 bit pointers (cSuneido is targeted at 32 bit) this gives 16 bit immediate integers.

Larger numbers could be stored this way (up to 31 bits) but there would be little advantage. With 16 bits we can avoid any bit operations by defining Value as:

```C++
union {
    SuValue* p;
    struct {
        short n;
        short type;
    } im;
};
```

Normally Value delegates methods to its SuValue. In the case of an immediate integer, a temporary SuNumber is created on the stack, **not** heap allocated,  (see NUM macro) and delegated to. These values must not be kept around. And it makes little sense to modify them.

**_jSuneido_**

Java doesn't allow this kind of low level optimization. Although Java has primitive integers they cannot be handled “generically” without boxing them to heap objects.

However, JIT compiling may be able to eliminate some boxing.

**_gSuneido_**

In earlier versions of Go, integers were automatically stored within interfaces - perfect for Suneido - but this was removed so the garbage collector would not have to deal with non-pointer values.

At first glance, it appears Go can't support immediate integers but it turns out to be possible.

The way to work within Go’s type safety is to use pointers into a large array of bytes. If we make the array 64 kB then we can handle 16 bit integers like cSuneido. We don't actually store anything in the array and we don't initialize it, so it doesn't take up any physical memory. So a small immediate integer type can be created with:

```Go
type smi byte
const range = 1 << 16
var space [range]smi
func NewSmi(i int) interface{} {
    return &space[i]
}
```

We can now define methods on *smi to support the Value interface. (See suint.go)

To get the integer back out of the pointer we have to use the unsafe package, but in a safe way :-)

[Go Interfaces and Immediate Integers](https://thesoftwarelife.blogspot.com/2017/09/go-interfaces-and-immediate-integers.html)

**_suneido.js_**

Integers are handled as JavaScript numbers, relying on the JavaScript implementation to optimize.

## Strings

Suneido does not require the use of a “string builder”. It optimizes concatenation so it can be used to build large strings efficiently.

**_cSuneido_**

Strings are implemented with three main classes:

**gcstring** is the low level string class. It is intended for garbage collection - it has no destructors and it shares portions of strings. Substrings simply point to a section of the original string. However, this can have the bad side effect of a small substring keeping a large source string from being garbage collected.

**Concat** is private to gcstring. It is used to optimize concatenation. Instead of allocating a new string for each concatenation, if the result size is greater than LARGE (64) we build a tree of Concats with gcstrings at the leaves. These are "flattened" to a normal string lazily, when required.

**SuString** is the high level SuValue. User accessible Suneido string methods are implemented here.

**_jSuneido_**

Native Java String's are used. (See Values).

Ops.cat will return a Concats if the combined length is greater than LARGE (256).

Concats wraps a StringBuilder. However, we don't want to allocate a new StringBuilder every time so if we concatenate onto a Concats we try to append to the StringBuilder. To detect modification Concats stores its own length, which may be shorter than the StringBuilder's length.

```
s = 'x'.Repeat(200) $ 'x'.Repeat(200)
// >256 so returns a Concats

t = s + 'a'
// s and t share the same StringBuilder
// s.length is 400, t.length is 401

u = s + 'b'
// can't share the StringBuilder
// because s.length < stringbuilder length
```

Concats extends String2 which is an abstract base class for string-like values. (Except also extends String2.) Used for type check in Ops.is\_ and for common implementations of SuValue and Comparable methods.

**_gSuneido_**

**_suneido.js_**

suneido.js does not attempt to optimize concatenation. This is left up to the JavaScript implementation.

## Exceptions

Caught exceptions can be treated as strings, but they also have other information attached.

Concatenating onto an exception maintains the exception information.

**_cSuneido_**

**_jSuneido_**

Except extends String2

**_gSuneido_**

**_suneido.js_**

## Functions & Methods

### Arguments

We need to handle passing named arguments and @args while efficiently handling simple calls (the majority) with minimal overhead.

Ideally we want to be able to call functions from native code relatively easily.

**_cSuneido_**

Arguments are passed on Suneido's value stack (not the native stack), described by separate nargs, nargnames, argnames, and each parameters.

**_jSuneido_**

Although the JVM has variadic arguments, they require boxing as an allocated array on the heap. To avoid this in simple cases, up to four unnamed arguments are passed individually as normal JVM arguments. Otherwise, they are passed as an array.

No explicit description of the arguments is used. Instead, special values are inserted into the arguments.

```
fn(@args) =\> EACH, args

fn(1, 2, a: 3, b: 4) =\> 1, 2, NAMED, "a", 3, NAMED, "b", 4
```

**_gSuneido_**

Arguments are passed on Suneido's value stack (not the native stack) like cSuneido. They are described by ArgSpec (see ArgSpec.go) containing:

**unnamed** - the number of unnamed arguments or the special values EACH or EACH1

**spec** - a list of indexes into the strings, specifying the argument names. The spec is sliced directly from the byte code (not copied).

**strings** - usually a reference to the string list from the function containing the call (a reference, not copied)

**_suneido.js_**

JavaScript is even looser than Suneido on argument passing. Any number of arguments may be passed to any function.

Unnamed arguments are passed as regular JavaScript arguments. Named arguments are passed in an object as the last argument.

Mandatory arguments are verified by using "= mandatory()". If the argument is not passed, mandatory throws an exception.

Multiple adapter functions are created for each function/method, either by the transpiler or by a separate code generation tool for built-in functions.

### Parameters

We need to handle default values (arg = constant), dynamic (\_arg), member (.arg), and gathering (@args).

We also need to handle both interpreted Suneido functions and methods as well as built-in ones.

**_cSuneido_**

**_jSuneido_**

See FunctionSpec.java

**_gSuneido_**

base.Func (in callable.go) describes the parameters for a function with Nparams, Ndefaults, Flags, Strings, and Values.

**_suneido.js_**

### Passing Arguments to Parameters

This process includes:

- expanding @args (like JavaScript "spread")
- constructing @params (like JavaScript "rest")
- filling in default values
- filling in dynamic defaults
- matching up named arguments
- checking for excess or missing arguments.

As usual, we want minimal overhead for the simple cases while still allowing for the more complex cases. We don’t want much overhead on most calls.

Optimizations:

- @args =\> @params should not expand args and then build params. It should just copy (or slice for @+1 args) the args object.
- Simple calls e.g. fn(1,2) should bypass argument massage and go directly to function(a,b)

|             | simple params | default params   | @params | raw        |
| :---------- | :------------ | :--------------- | :------ | :--------- |
| simple args | no massage    | fillin           | gather  | no massage |
| named args  | shuffle       | shuffle & fillin |
| @args, @+1  | expand        | expand & fillin  | slice   |

**_cSuneido_**

**_jSuneido_**

Args.massage converts arguments from what the call supplies to what the function expects.

We specialize for up to five unnamed arguments.

**_gSuneido_**

**_suneido.js_**

Special comments in the code are used by a code generator to generate adapters for each function

# Compiler

Suneido uses a hand-written lexical scanner and recursive descent parser.

**_cSuneido_**

Compiling is done in a single pass, with byte code generated during parsing. This is fast and memory efficient, but not as flexible as using an intermediate AST.

**_jSuneido_**

Parsing produces an AST which is then used to generate Java byte code using the Asm library.

**_gSuneido_**

Parsing produces an AST which is then used to generate byte code. (Similar to jSuneido)

**_suneido.js_**

Currently (2018) jSuneido is used to parse to an AST which is then used by Suneido code to generate JavaScript.

[Precedence Climbing Expression Parsing](https://thesoftwarelife.blogspot.com/2018/10/precedence-climbing-expression-parsing.html)

## Interpreter

Only cSuneido and gSuneido implement their own interpreters.

**_cSuneido_**

**_jSuneido_**

jSuneido compiles to JVM byte code which is run by the JRE.

**_gSuneido_**

**_suneido.js_**

suneido.js compiles to JavaScript which is run by e.g. the browser.

## Concurrency

# Database

Currently only cSuneido and jSuneido implement the database. They use different database file formats. However, the format used by dump and load is the same. The normal method of converting a database between cSuneido and jSuneido is to dump on one and load on the other.

## Storage

Suneido memory maps the database file. The file is mapped in chunks (of 64mb. Additional chunks are added as the database expands.

Although the mapping will work with physical memory smaller than the database file, the best performance is when physical memory is larger than the database i.e. no swapping. Depending on the operating system, the process information may not show the mapped space as part of the process memory usage. This can make it (incorrectly) appear that Suneido does not require much memory.

Suneido currently does not reclaim file space during operation. This means you need to periodically compact the database off-line.

**_cSuneido_**

The database is stored in a single file: suneido.db

To handle databases larger than the address space of the 32 bit process (2gb) chunks are mapped as needed and unmapped LRU style. We rely on Windows to cache the file contents in memory even when we unmap it.

Although cSuneido is not completely append-only like jSuneido it only updates indexes in place. Data records are never updated, new versions are added.

**_jSuneido_**

The database is stored in three files:

- suneido.dbd - data
- suneido.dbi - indexes
- suneido.dbc - a hash of the data size (see Shutdown & Startup)

Since Java does not support it, unlike cSuneido, we never unmap chunks.

Previously jSuneido mapped the chunks lazily, but to avoid concurrency issues, this was changed to map all chunks at startup. Since mapping does not actually read/load the file contents, this does not delay startup by much.

jSuneido's database implementation is append-only i.e. the dbd and dbi files are only appended to. Once written, the file contents is never updated. This has multiple benefits:

- Crash recovery is simplified
- Multi-version transactions are simple
- Appends are optimal for performance, especially on SSD

The data file is written when a transaction commits. Up to that point all updates are done in memory, private to the transaction. Aborting a transaction simply discards the in memory information. This makes it less likely that a transaction will be partially written, but it is still possible e.g. due to a crash. We don't have to worry about a transaction being partially rolled back. (Since rollback doesn't touch the database files.)

For performance, the dbi (indexes) file is written lazily, once per minute. If jSuneido crashes or is killed pending dbi writes may be lost. This will be detected at startup (see Shutdown & Startup) and will be repaired.

Compaction removes both old versions of data records, old versions of btree index nodes, and old versions of schema information.

## Shutdown & Startup

Both cSuneido and jSuneido attempt to detect database corruption and to repair it if possible. (See Recovery)

**_cSuneido_**

**_jSuneido_**

During shutdown the suneido.dbc file is updated with a hash of the size of the suneido.dbd file. (A hash is used to avoid anyone editing the file by hand.)

At startup, we check if the dbc file matches the dbd file. If not, we assume the database was not shutdown properly (e.g. crashed or was killed) and we do a full check of the database. Otherwise, normally, only a fast check of the tail of the file is done, in case the OS did not complete all the writes.

At startup we also check that the dbi (indexes) file matches the dbd (data) file by comparing the last commit hashes.

## Recovery

At startup (see Shutdown & Startup) if corruption is detected, then the database will automatically be rebuilt.

Corruption can occur for several reasons:

- Crash of process or operating system
- Hardware or operating system error e.g. disk or memory
- Suneido process killed during io
- Bugs in Suneido

This process will attempt to recover the database as of some previous point in time.

Recover maintains atomic transactions. i.e. either a transaction will be recovered completely or not at all, never partially.

**_cSuneido_**

Recovery rebuilds the indexes from scratch, it does not use any of the old index information.

**_jSuneido_**

Even with jSuneido's append-only design, old parts of the files may be corrupted by hardware errors.

jSuneido can recover the database from just the dbd (data) file, however for performance, recovery will use (copy) as much of the existing dbi (index) file as it can.

## Packing

Data values are "packed" to store in the database and to transfer client-server.

When a database or table is dumped, the values are in packed format. Since dumps are compatible between cSuneido and jSuneido, this means the packed format must also be compatible.

Each packed value starts with a tag byte that indicates the type of value. In the case of true and false, the tag specifies the actual value.

Packed values are comparable as raw unsigned bytes. This allows indexing, sorting, and selecting in the database without unpacking values. This starts with tag values having the same order as types compare in Suneido.

The length of packed values is tracked externally, it is not stored in the packed format. A zero length packed value (not even a tag) is taken as an empty zero length string.

Packing is a two stage process. First, a packsize method is called on the value. Then the pack method itself is called, passing it a buffer with sufficient room.

Unpacking looks at the tag and then calls static methods for the type.

**_cSuneido_**

Packed values are in char sequences, often in gcstring's to track size.

**_jSuneido_**

Packed values are in ByteBuffer's.

**_gSuneido_**

**_suneido.js_**

## Records

## Indexes

## Queries

### Query Optimization

Lexical scanning uses the same code as for the language with a different set of keywords.

Parsing builds an AST which is used to optimize and execute the query.

Each query operation e.g. where or join is a node class. Optimization and execution is done by methods. All the node types inherit from Query.

The first stage of optimization is **transform**. It makes changes that are assumed to be always beneficial. It does not estimate any costs. Transform rearranges

The second stage is the cost based **optimize** which chooses strategies and indexes. The costs are estimated very roughly based on number of bytes read.

Query operations implement multiple strategies.

## Transactions

Suneido implements "ACID" transactions - atomic, consistent, isolated, and durable.

Transactions see the database as of their start time.

Suneido uses optimistic multi-version concurrency control. Update transactions do not lock records. However, they may fail to comit if there is a  conflict with another update transaction.

Read-only transactions do not do any locking and do not conflict with any other transactions.

**_cSuneido_**

Versioning is handled by tagging index entries with a timestamp. Transactions will ignore index entries later than their as-of time. Deleted index entries are left in the index but tagged with the time of their deletion. Earlier transactions will still see these entries.

**_jSuneido_**

Versioning is handled by the append-only database. A given version is simply a prefix of the database.

Update transactions operate in memory. They do not update the data file until they commit.
