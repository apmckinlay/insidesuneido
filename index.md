- [Introduction](#introduction)
- [History](#history)
  - [Past](#past)
  - [Present](#present)
  - [Future](#future)
- [Motivation](#motivation)
- [Implementations](#implementations)
- [Low Level](#low-level)
  - [Representation](#representation)
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
    - [Literal Arguments](#literal-arguments)
  - [Iterators and Sequences](#iterators-and-sequences)
- [Compiler](#compiler)
  - [Byte Code](#byte-code)
  - [Interpreter](#interpreter)
  - [Classes](#classes)
    - [Private Class Members](#private-class-members)
    - [Getters](#getters)
- [Windows Interface](#windows-interface)
  - [DLL](#dll)
  - [COM](#com)
  - [SuneidoAPP](#suneidoapp)
- [Concurrency](#concurrency)
- [Database](#database)
  - [Storage](#storage)
  - [Shutdown & Startup](#shutdown--startup)
  - [Recovery](#recovery)
  - [Packing](#packing)
  - [Records](#records)
    - [Old Format (before 2019)](#old-format-before-2019)
    - [New Format (after 2019)](#new-format-after-2019)
  - [Indexes](#indexes)
  - [Optimizations](#optimizations)
    - [Btrees](#btrees)
    - [ixbuf](#ixbuf)
  - [Queries](#queries)
    - [Query Optimization](#query-optimization)
  - [Transactions](#transactions)
  - [Repair](#repair)
  - [Metadata](#metadata)

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

As of Dec. 2020, Axon is migrating customers to using gSuneido for desktop clients.

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

- Client side is complete, now working on the database.
- Has its own byte code and interpreter
- Suneido values (including integers and strings) implement the Value interface
- Uses panic and recover to implement exceptions
- 64 bit

**_suneido.js_** - JavaScript

- Incomplete, work in progress
- Transpiles from Suneido code to JavaScript, currently (2018-01-30) using AST from jSuneido parser
- Does not include the database
- Written in TypeScript
- One challenge is to somehow run existing user interfaces (which are win32 specific and assume low latency)
- Another challenge is that current code assumes blocking/synchronous, but browsers want async.
- Targeting Chrome (or Chromium browsers) initially

# Low Level

## Representation

cSuneido inadvertently used little endian (least significant byte first) for some things (e.g. records/tuples) because that was native byte order on x86.

But to make packed values sort properly, numbers had to be stored big endian (most significant byte first).

gSuneido (in 2019) moved to standardize on big endian.

Although the in-memory and on-disk representations do not need to be compatible, records and packed values are used in the client-server protocol so they must be the same to be interoperable.

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

- makefile
- gsuneido.go
- **runtime** - the interpreter and the core built-in types
- **builtin** - built-in functions and classes
- **compile** - lexer, parser, AST, byte code generation
- **options** - command line options
- **util** - low level general purpose utilities
- **dbms** - the dbms client and server
- **db19** - the gSuneido database implementation
- **res** - windows resources

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

Some types have multiple internal representations. Numbers can be either small integers or decimal floating point. Strings can be either simple strings, deferred concatenations, or exceptions (with an attached call stack). This is mostly invisible at the Suneido language level.

**_cSuneido_**

SuValue (see suvalue.h/cpp) is the base class for SuString, SuNumber, SuObject, etc. that are normally allocated on the heap and referenced by pointer.

Value (see value.h/cpp) is a union of a pointer to an SuValue or a small immediate integer. (See Integers) Value delegates to SuValue.

**_jSuneido_**

For efficiency, some Java types are used directly, without being wrapped, e.g. Boolean, String, and Integer. SuValue is the base class for other types such as SuContainer.

Because there is no common base class or interface for values, operations (see ops.java) must do type checks (e.g. instanceof).

**_gSuneido_**

Values implement the Value interface (see value.go). Like Java, some types are used without wrapping (e.g. strings) but in Go we can define a new type based on a built in type and support the Value interface. Therefore, fewer explicit runtime type checks are required.

**_suneido.js_**

## Numbers

Don't narrow results - too much overhead. This means that we have to canonicalize values in SuObject so ob[5] is the same whether the 5 is an integer or a dnum.

Divide handles even integer division specially. Running the ETA tests found roughly 1/3 of divides were even integer divisions. Benchmark went from 43 ns to 14 ns and the extra case doesn't seem to slow down normal usage.

Many things treat false and "" as zero. However, the first argument to ranges and string subscripts do not.

### Decimal Floating Point

Numbers in Suneido are stored in **decimal** floating point so that decimal fractions (like .1) can be represented exactly.

This means we can't use the **binary** floating point numbers normally provided by hardware and languages.

As of 2018-08 cSuneido, jSuneido, and gSuneido all use an implementation (Dnum) with a 64 bit coefficient and 8 bit sign and exponent. Only 16 of the possible 19 decimal digits are used. This allows splitting into two 8 decimal digit halves which fit into 32 bit for faster processing.  (cSuneido is 32 bit so it cannot take advantage of 64 bit instructions. In Visual C++ 32 bit code most 64 bit operations are subroutine calls.) The coefficient is kept “maximized” i.e. “shifted” left as far as possible.

*cSuneido* previously had an implementation of decimal floats that used four 16 bit integers, each holding four decimal digits (i.e. 0 to 9999) plus a sign and an exponent. The intent was to provide 16 digits of precision. However, because the exponent had 4 digit granularity, the actual precision could be as low as 13 digits.

*jSuneido* previously used Java BigDecimal. This worked fine, but BigDecimal is arbitrary precision which is overkill for Suneido.

Previously there were some minor issues with inconsistent results between *cSuneido* and *jSuneido* due to the different implementations and precisions. This should no longer be an issue - they should now give identical results.

Unfortunately, JavaScript does not have 64 bit integers, it only has binary floating point numbers, with bitwise operations defined as 32 bit.  (Internally, JavaScript implementations usually use integers.) To work around this, *suneido.js* uses JavaScript floats as 52 bit integers. This provides a little over 15 digits of precision - not quite the 16 that Suneido nominally provides.

For compatibility, the packed format was *not* changed with the switch to Dnum. However, the old format is not the most efficient for Dnum. The plan is to switch this (along with several other representation improvements - records/tuples and packed objects) in 2019.

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

```Go
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

gSuneido also has "signatures" on ParamSpec and ArgSpec which provide fast matching. This is mostly for built in functions since interpreted functions already have so much overhead.

**_suneido.js_**

Special comments in the code are used by a code generator to generate adapters for each function

### Literal Arguments

**_cSuneido_** attempt to optimize [...] expressions by compiling any constant arguments into an object and calling a special constructor.

**_jSuneido_** attempts to optimize all call expressions by compiling more than 10 constant arguments into an object and passing it as with @args.

However, these optimizations affect a small number of calls (less than 1% in jSuneido) so **_gSuneido_** does not implement this optimization.

Related to this, as of 2019-05, [...] generates an object (rather than a record) if there are unnamed arguments. This is partly because objects are simpler. It is also a step towards removing unnamed/list elements from records and having only string named members (like instances).

## Iterators and Sequences

A user defined iterator is a class providing a Next method that returns the instance itself when it reaches the end. The reason for using itself rather than e.g. false is that the iterator itself is highly unlikely to be an actual value. Unfortunately, this convention is not used by some things like queries, which return false. (In this case safely, because they otherwise return records.)

A for-in loop uses an iterator.

```
for x in ob
```

is equivalent to:

```
iter = ob.Iter()
while iter != (x = iter.Next())
```

However, user defined classes cannot define their own Iter method because it is built-in on classes and instances to iterate through the members.

A **sequence** is a kind of virtual object that wraps an iterator and instantiates it lazily. Sequences are returns by some built-in methods like Seq and object.Members. They can also be explicitly created using the Suneido language Sequence(iter) function.

The iterator passed to Sequence should also have Infinite? and Dup methods. An infinite sequence (e.g. all the odd numbers) will throw an error if anything tries to instantiate it. Dup should return a copy of the iterator that is reset to start at the beginning.

In most cases, sequences can be used transparently. However there are subtle differences, primarily that before the sequence is instantiated it reflects changes in the underlying data. For the rare case where this is relevant, sequences have an Instantiate() method.

**_gSuneido_**

Internal iterators implement the **runtime.Iter** interface with Next, Infinite, and Dup.

**SuIter** is a Value that adapts anything implementing the runtime.Iter interface so it can be used as a Suneido language iterator.

**wrapIter** adapts a Suneido iterator (a class with Next,Dup,Infinite) to the runtime.Iter interface i.e. the reverse of runtime.SuIter

The Suneido language **Sequence** function returns an SuSequence wrapping a Suneido iterator.

SuSequence is a Value that wraps a runtime.Iter and instantiates it lazily. The Iter is either built-in e.g. Seq or object.Members, or user defined via Suneido Sequence. SuSequence wraps a runtimer.Iter rather than a Suneido iterator so it is more efficient for built-in iterators like object.Members.

# Compiler

Suneido uses a hand-written lexical scanner and recursive descent parser.

Note: There is also a parser written in Suneido (Tdop), however it is quite slow which limits its usefulness.

**_cSuneido_**

Compiling is done in a single pass, with byte code generated during parsing. This is fast and memory efficient, but not as flexible as using an intermediate AST.

Constant folding is done during byte code emit.

**_jSuneido_**

Parsing produces an AST which is then used to generate Java byte code using the Asm library.

Constant folding is done during code generation.

An extra pass is done over the AST to determine which blocks can be compiled as regular functions instead of closures.

**_gSuneido_**

Parsing produces an AST which is then used to generate byte code. (Similar to jSuneido)

gSuneido uses [Precedence Climbing Expression Parsing](https://thesoftwarelife.blogspot.com/2018/10/precedence-climbing-expression-parsing.html), whereas cSuneido and jSuneido just use recursive descent.

Constant folding is done as part of the AST building. This is naturally bottom up, and eliminates the need for a separate tree traversal. It is implemented as a wrapper (decorator) around the node factory, so it is easily bypassed if you want the AST one to one with the source code.

gSuneido also does [constant *propagation*](https://thesoftwarelife.blogspot.com/2020/04/constant-propogation-and-folding.html). Final variables, ones that are assigned once and never modified are identified during parsing. If any are found then an extra pass is made over the AST to propagate. Folding is also done during this pass because propagation may create new opportunities.

If any blocks are found during parsing, an extra pass is done over the AST to determine which of them can be compiled as regular functions instead of closures.

**_suneido.js_**

Currently (2018) jSuneido is used to parse to an AST which is then used by Suneido code to generate JavaScript.

## Byte Code

**_jSuneido_**

Uses Java byte code.

**_cSuneido_**

Uses every possible byte value. This makes the code compact but it complicates the interpreter.

**_gSuneido_**

Does not pack any options or arguments into the op codes or use variable length ints for arguments. This simplifies the interpreter. It also means there are lots of byte codes available, allowing specialized op codes for things like for-in.

## Interpreter

Only cSuneido and gSuneido implement their own interpreters.

**_cSuneido_**

**_jSuneido_**

jSuneido compiles to JVM byte code which is run by the JRE.

**_gSuneido_**

A standard loop containing a giant switch with one case per op code.

**_suneido.js_**

suneido.js compiles to JavaScript which is run by e.g. the browser.

## Classes

Classes and instances of classes were originally just objects with the addition of a class reference. In jSuneido classes and instances became maps from strings (member names) to values. (No unnamed list members.) This approach was migrated to **cSuneido**, and is also used in **gSuneido**.

### Private Class Members

Suneido fakes private members by prefixing them with their class name. e.g. in MyClass a private member foo becomes MyClass_foo. Although this is primarily an implementation detail, these members are still visible to Suneido code and tests take advantage of this. However, we try to avoid bypassing privacy in actual application code.

Anonymous classes are given a system generated internal name (e.g. Class123) for privatization.

Privatization is done at compile time, for class members like:

`foo: 123`

and for member references like:

`.foo`

### Getters

If code reads a member that does not exist, Suneido will look for a "getter" i.e. a method that will provide the value, either Getter_(name) or getter_name or Getter_Name

Private getters e.g. getter_foo() are privatized to Getter_MyClass_foo

Note: Previously, getters were written as Get_ and get_ but this is a common name and was often used for regular methods, making it hard to know whether a method was intended to be a getter or not.

# Windows Interface

cSuneido and gSuneido have interfaces to Windows facilities.

A Windows interface for jSuneido was implemented and was functional, but it was not used and not maintained so eventually it was removed. One of the factors was that having to have a JRE made it much more heavy

## DLL

**_cSuneido_**

cSuneido has a general purpose DLL interface. The DLL functions, structs, and callbacks are defined in Suneido code. (primarily in stdlib)

**_gSuneido_**

gSuneido takes a different approach. The DLL functions, structs, and callbacks are built-in and cannot be defined in Suneido code. This was partly to avoid writing the general purpose facility, although in the end it was probably more work to write them all in Go. The other reason was that the cSuneido approach is inherently dangerous. If a definition is wrong you can easily corrupt memory or crash.

Originally this was written using syscall.SysCall but this turned out to be unstable (see [here](https://thesoftwarelife.blogspot.com/2019/10/gsuneido-roller-coaster.html) and [here](https://thesoftwarelife.blogspot.com/2020/01/recurring-nightmares.html)), probably related to Go's garbage collection, escape analysis, and moving/growing stacks. There were also questions related to threading. cSuneido only ever had a single real thread that everything ran on, so there were no threading issues. (See [Concurrency](#concurrency)) Go not only uses multiple threads but also runs multiple goroutines on a single thread. Each thread in Windows has its own event queue and message loop. The simplest approach is to use one UI thread. gSuneido isolates the DLL calls and the message loop etc. in a single thread created by cgo outside of Go.

## COM

Only the IDispatch interface is implemented.

This is used primarily for the browser control and Image.

It might be better to make Image built-in. It was originally built-in on cSuneido but it was moved to stdlib to work with the jSuneido Windows interface.

It can also be used to automate external applications with COM interfaces.

## SuneidoAPP

This is the suneido: protocol used with the browser control.

# Concurrency

**_cSuneido_**

cSuneido uses one thread with multiple fibers. Since fibers only yield explicitly, we don't have to worry about concurrency or locking. However, blocking IO is an issue.

cSuneido uses a thread to automatically change to the wait cursor if the message loop isn't running.

The database server also used fibers per connection.

**_jSuneido_**

jSuneido implements Suneido Thread's with actual threads. Threads are mostly isolated, but mutable objects can be shared, so concurrency is a concern. There is some locking, but it is probably not complete or sufficient to prevent all possible concurrency problems.

The database server also uses threads. In this area, the concurrency has been carefully thought out and is complete as far as we know.

**_gSuneido_**

Suneido Thread's are Go routines.

Mutable Value's have a concurrent flag. When the concurrent flag is false, no locking is done. When the object propagates to other threads, the SetConcurrent method in the Value interface sets the concurrent flag to true. The flag is set while the value is still single threaded and the flag is not changed again after being set. This makes it safe to access with no locking. SetConcurrent is recursive and deep - it must call SetConcurrent on all reachable child values. And if new child values are added to a concurrent value (e.g. Object) the new child value must be set to concurrent. The global Suneido object is concurrent from the start. Concurrency is contagious and irreversible.

Deep equal and compare are structured to only lock one object at a time to avoid deadlock. This means the result is undefined if there are concurrent modifications. The locking is just to prevent data race errors or corruption.

Since the Go mutex is **not** reentrant, to avoid deadlock a method that locks must **not** call another method that locks. The coding convention is that public (capitalized) methods lock, and private (uncapitalized) methods do not. However, this is not followed 100%. Locking public methods must not call other locking public methods. This means there is often a public method that locks, and a private method that does the work.

Synchronized is a global reentrant mutex. Only one thread can be inside Synchronized at a time. Different Synchronized blocks are not independent.

# Database

![database design](images/suneido&#32;dbms&#32;design.svg)

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

For cSuneido and jSuneido to interoperate client-server, the packed format must be compatible.

When a database or table is dumped, the values are in packed format. Since dumps are compatible between cSuneido and jSuneido, this means the packed format must also be compatible.

Each packed value starts with a tag byte that indicates the type of value. In the case of true and false, the tag specifies the actual value.

Packed values are comparable as raw unsigned bytes. This allows indexing, sorting, and selecting in the database without unpacking values. This starts with tag values having the same order as types compare in Suneido.

The length of packed values is tracked externally, it is not stored in the packed format. A zero length packed value (not even a tag) is taken as an empty zero length string.

Packing is a two stage process. First, a pack size method is called on the value. Then the pack method itself is called, passing it a buffer with sufficient room.

Unpacking looks at the tag and then calls static methods for the type.

**_cSuneido_**

Packed values are in char sequences, often in gcstring's to track size.

**_jSuneido_**

Packed values are in ByteBuffer's.

**_gSuneido_**

Packed values are normally in strings.

**_suneido.js_**

### Objects

Objects and records contain nested packed values.

If there are no members the packed format is just the tag.

Following the tag are:

- unnamed members:
  - varuint count (possibly zero)
  - one or more of:
    - varuint size + packed value
- named members:
  - varuint count (possibly zero)
  - one or more of:
    - varuint size + packed member name value
    - varuint size + packed member value

## Records

Records in the database are stored as a list of packed values in a format that allows indexing individual values directly (like an array).

To minimize space there are three variations using either 1, 2, or 4 byte integers for the offset.

Note: These Records are different from SuRecord which is a Suneido value.

### Old Format (before 2019)

The first byte is the mode, either 'c', 's', or 'l'.

The second byte is 0.

The third and fourth bytes are the number of values (little endian).

Next are the length (including header) and the offsets of the values (little endian).

Next are the packed values stored contiguously.

### New Format (after 2019)

An empty record is a single zero byte.

The first and second bytes are the mode and the number of values (big endian). The mode is in the two high bits.

Next are the length (including header) and the offsets of the values (big endian).

Next are the packed values stored contiguously.

## Indexes

Suneido supports three kinds of indexes:

- keys - do not allow duplicates
- indexes - allows duplicates
- unique indexes - unique like a key, but allows multiple empty values

All three are implemented with the same btrees. Because the btrees require unique values, additional data is added to indexes and empty values in unique indexes. jSuneido added the record address, but that means every update changes the index. gSuneido adds key field(s) instead. Thanks to the encoding that gSuneido btrees use, this does not add much space overhead.

Uniqueness is enforced by lookups prior to outputting or updating records.

## Optimizations

It's possible to have redundant keys e.g. key(a) and key(a,b). gSuneido identifies "primary" keys i.e. ones that do not include any other key and only does uniqueness lookups on these.

Originally, the first, shortest key was used to make non-key indexes unique. This was improved to select the key that required the least number of additional fields. e.g. with key(a,b) key(c,d,e) index(d,e) it's better to use key(c,d,e) because it only requires one additional field (c).

We also identify unique indexes that contain a key. Like a non-primary key, these do not require any uniqueness lookups.

### Btrees

gSuneido btrees use [partial incremental encoding](https://thesoftwarelife.blogspot.com/2019/02/partial-incremental-encoding.html) which greatly reduces the size of the btrees compared to storing full keys. On average, less than 2 bytes per key are stored in the index.

This is a tradeoff between complexity, read speed, and index size. More compact btrees means higher fanout which means faster lookups. However, insertion and deletion are more complex due to the encoding.

When the server is stopped, all the indexes are stored in the btrees in the database file. However, while the server is running, indexes are "layered". In progress transactions store their updates as an in-memory "overlay" on the index. When a transaction commits, this layer is added to the global state. Background processes merge these layers. It is not until the next persist (e.g. once per minute) that the in memory layers are written to the btrees in the database file.

One advantage of only updating the disk btrees once per minute is that it reduces the write amplification from the append-only database.

This is a little bit similar to log structured merge trees. LSM trees are usually used to handle high insert rates. With gSuneido it is used primarily to improve concurrency. Transactions can insert into their own private overlays with no locking, the overlays can quickly be layered onto the global state, and they can be merged lazily in the background.

The downside (as with LSM trees) is increased read overhead since multiple layers may have to be accessed.

### ixbuf

In memory index layers are ixbuf's - simple list of lists. A bit like a btree but only two levels - a root and leaves. The size of the leaves is adaptive. The more entries in the ixbuf, the larger the leaves are allowed to grow before splitting. The policy is designed so that the root node ends up a similar size to the leaves.

A transaction that modifies an index creates a mutable ixbuf layer for it. Mutable ixbuf's are thread contained.  Once a transaction commits the ixbuf becomes read-only so it can be safely used by multiple threads.

Many transactions are small, which means many ixbuf's are small. Transactions are limited in terms of the number of updates they can do (currently 10,000). This means the maximum ixbuf size created by a transaction is 10,000 entries or roughly 100 leaf nodes. Merging of ixbuf's from multiple transactions can result in larger ixbuf's.

ixbuf's are merged by a background process. This merging takes advantage of the ixbuf's being immutable and may reuse nodes in the result. (persistent immutable data structure)

## Queries

Query where and extend expressions are a subset of language expressions. The subset has expanded over time.

Lexical scanning uses the same code as for the language with a different set of keywords.

cSuneido has separate code for parsing query expressions, but this means a lot of duplication.

jSuneido uses the same expression parser for both the language and queries. It uses different "generators" to build different AST nodes. However, the abstract generator complicates the parser somewhat.

Parsing builds an AST which is used to optimize and execute the query.

Each query operation e.g. where or join is a node class. Optimization and execution is done by methods. All the node types inherit from Query.

Query (and query expression) execution is done by a tree walking interpreter. There is no byte code as there is for the language.

Lower level iterators stick at eof bidirectionally. i.e. once they hit eof, both Next and Prev will return eof. However, Query operations rewind after hitting eof. And then SuQuery sticks at eof unidirectionally. i.e. once Next hits eof it will stick, but Prev will give the last row, and vice versa.

### Query Optimization

Query optimization tries to find a good way to execute the query. Optimization does not change the results, just the way they are derived. It does not necessarily find the best strategy. Optimization should not affect Nrows, Header, Columns, or Keys.

The first stage is **transform**. It makes changes that are assumed to be always beneficial. It does not estimate any costs and it does not look at the data. 

The second stage is the cost based **optimize** which chooses strategies and indexes. 

Query operations implement multiple strategies.

#### Transform

Transform does several things: 

- remove unnecessary operations (where, extend)

e.g. `table where true`

=> `table`

- eliminate operations or branches that cannot produce any results

e.g. `table1 join (table2 where false)`

=> `table1`

- combine adjacent equal operations (where, rename, extend, project)

e.g. `table where a = 1 where b = 2` 

=> `table where a = 1 and b = 2`

- move where's and project's closer to the tables

e.g. `table rename b to a where a = 1`

=> `table where b = 1 rename b to a`

Moving operations may make them adjacent so they can be merged.

Moving where's toward the tables helps in two ways:

- it reduces the number of rows that later operations must process
- if the where can be moved adjacent to the table it can make use of table indexes

#### Fixed

In Suneido query processing, "fixed" refers to fields that are known to have one or more specific constant values.

In the simplest case, fixed comes from something like:

```
table where mykey = 123
```

We know that all the results from this query will have mykey = 123 (even though table may have more different values).

Fixed tracks multiple values e.g.

```
(table where mykey = 123) union (table where mykey = 456)
```

However, in most cases we only take advantage of fixed with a single value.

Single value fixed are used, for example, to eliminate fields from sorting. Reducing the fields can make more indexes applicable. For example, if you sort by a,b,c and b is fixed, then you can use an index on a,c. Or vice versa, if you sort by a,c and b is fixed, then you can use an index on a,b,c.

Fixed is one of the methods in the Query interface.

### Mode

Query optimization and strategy choice is affected by the "mode" of the query - read, update, or cursor.

A cursor is a query that is separate from any specific transaction. Instead, a transaction must be supplied to Get.

Cursors do not allow temporary indexes. This is because the information in the temporary index would become outdated and incorrect.

Technically, temporary indexes should also not be allowed in update mode, again because the information in the index could get out of date. However, this was overlooked for many years, and now it would be hard to restrict. In practice, it hasn't  been a problem.

Certain other operations (**project**, **summarize**) also create their own equivalent of temporary indexes. Currently gSuneido **project** correctly only allows this in read mode, but **summarize** allows it in any mode, so it is possible to get outdated information.

## Transactions

Suneido implements "ACID" transactions - atomic, consistent, isolated, and durable.

Transactions see the database as of their start time.

Suneido uses optimistic multi-version concurrency control. Update transactions do not lock records. However, they may fail to commit if there is a  conflict with another update transaction.

Read-only transactions do not do any locking and do not conflict with any other transactions.

**_cSuneido_**

Versioning is handled by tagging index entries with a timestamp. Transactions will ignore index entries later than their as-of time. Deleted index entries are left in the index but tagged with the time of their deletion. Earlier transactions will still see these entries.

**_jSuneido_**

Versioning is handled by the append-only database. A given version is simply a prefix of the database.

ReadTransaction is the simplest. It operates "as of" its start time i.e. on that "version" of the database. It does not need to track actions. It does not block any other activity. Therefore, it has no limits in terms of actions or time.

ReadWriteTransaction is the base class for BulkTransaction and UpdateTransaction

UpdateTransaction operates in memory. They do not update the data file until they commit. UpdateTransactions use OverlayIndex to merge the in-memory changes from the current transaction with the database version (so it sees it's own changes). OverlayIndex.Iter tracks reads (with TransactionReads which uses IndexRange) so they can be validated at commit (to detect conflicts). Changes are also tracked and used to update the actual database state at commit.

Because UpdateTransaction tracks reads and writes, there are limits on how much a single transaction can do, to prevent excessive memory use. SchemaTransaction and RebuildTransaction are excluded from these limits.

There are also limits on the number of overlapping transactions outstanding.

BulkTransaction is used for load, compact, and rebuild. It is exclusive i.e. no other concurrent transactions are allowed. Unlike normal transactions, it writes to the database file incrementally, prior to commit.

***gSuneido***

Like jSuneido, versioning is handled by an append only database.

Like cSuneido, and unlike jSuneido, the database is stored in a single suneido.db file containing both data and indexes. It is only ever appended to. Except that the size is written to the start of the file at shut down to determine at start up whether it was shut down properly (didn't crash). This is similar to jSuneido's .dbc file.

gSuneido is intended to allow more concurrency than jSuneido. The global commit lock in jSuneido is a bottleneck, especially since all the writing to the database files is done during commit while holding the lock. gSuneido improves on this by:

- Writing data records immediately. This means some wasted abandoned space if the transaction aborts, but it means less work for commit.
- Many actions are "fire and forget" i.e. the client does not wait for the action to complete. This allows client processing to proceed concurrently with server processing. 
- Conflict checking is done incrementally, by background threads.
- Index updates are merged by background threads. Unmerged index layers will slow down reads, but in many/most cases the merges will happen immediately after commits so they will not affect most reads. However, there will normally be at least two layers - the on disk indexes, and the in-memory updates since the last persist. (Unless there have been no updates since the last persist.)
- Like jSuneido, indexes are persisted to disk periodically (e.g. once per minute). This reduces write amplification.
- Commit is simple, it must wait for its conflict checking to finish, and then it merges its state with the global master state. This is much less work and faster than jSuneido.

Suneido does not use a log (although in a sense, the append-only database serves as the log). This means that transactions are not necessary durable (the 'D' in ACID) in the case of system failure (OS or hardware). Transactions are guaranteed to be atomic (the 'A' in ACID) however. In modern computer systems, with multiple layers of caching, it is difficult to guarantee that data has been durably written to the storage device, especially when using memory mapped file access. It is also slow to force this and wait for it to complete. Writing a separate log also doubles the amount of writing since every change must be written to the log as well as the database. Suneido trades a small amount of durability for speed. A crash may also lose some committed transactions because indexes are only persisted once per minute. Note that even with a separate log you are not immune to hardware failures that affect both the log and the database.

## Repair

Unlike cSuneido and jSuneido, gSuneido's database file is not intended to be scanned. For example, data records are not tagged with what table they belong to. Nor are deletes recorded explicitly. The only way to find the data records is via the indexes, and the only way to find the indexes is via the meta data that is pointed to by the state root.

Think of the database file as a bucket. Data records are added to the bucket as they are created. During the next persist (every minute) index updates will be added and then a new "state". The state points to the indexes which point to the data records. Everything is reachable from the state (and only reachable from the state).

When the server shuts down there is a final persist so the last thing in the file (the top of the bucket) is the root of the state.

State roots are delimited by magic values. When the server starts it checks if the end of the database is a valid state root. If not, then it searches backwards for the magic values to find a valid state. Each state corresponds to the value of the database at a given point in time. 

Once a state root has been located (either at the end of the file, or by searching) it is checked. For a normal startup, where a valid state is found at the end of the file, a quick check is done. Otherwise, it means the database was not shut down properly and a full check is done. This could be because it crashed, or because the operating system crashed, or the hardware itself (e.g. power failure). 

A quick check takes advantage of the append-only nature of the database. This means that offset in the database file correspond roughly to age. In addition, immutable btree updates mean path copying, so the paths leading to newer records will be later offsets in the file. So the quick check can traverse (and check) the newer data. This is based on the idea that if the server crashes, that it will be recent data that may not be written correctly.

A full check checks entire indexes (in parallel) i.e. that the keys in the index match the values in the data records, and that the indexes match each other in terms of number of records and checksum of record offsets.

If checking fails for a given state, then we search backwards for a previous state. If no valid state can be found then the database is not repairable.

## Metadata

The metadata is split into two parts - the schema and the info. The info is the part that changes more frequently - the number of records and total size of each table (used by the query optimizer) and the indexes. Separating frequently changing information reduces the size of the updates that must be written to disk. The info includes the offsets of the roots of the btrees.

The schema and info are stored in [hash array mapped tries](https://thesoftwarelife.blogspot.com/2020/07/hash-array-mapped-trie-in-go.html). Updates are written to the database file along with the index persists.

Once again, there is a tradeoff between writing and reading. For writing, it's faster to save just the changes but then you end up with a long linked list of changes, which is slow to read.  Note that we only have to read it at startup, after that we  can use the in-memory version. If you save the entire meta data at each persist, that makes it easy to read, but it causes write amplification because you're rewriting all the unchanged data.

Instead, we compromise. Sometimes we write just the changes. Other times, to prevent the linked list from getting too big, we write bigger chunks. This is a bit like a [log structured merge tree](https://en.wikipedia.org/wiki/Log-structured_merge-tree) (LSM tree).

We have all the data in memory, so rather than actually merging data from disk, we track the "age" of each entry (when it was last updated) and then we can write a bigger chunk by going back further in age.

