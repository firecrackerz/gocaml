GoCaml
======
[![Linux and macOS Build Status][]][Travis CI]
[![Windows Build Status][]][Appveyor]
[![Coverage Status][]][Coveralls]

GoCaml is a [MinCaml][] implementation in Go using [LLVM][]. MinCaml is a minimal subset of OCaml for educational purpose ([spec][MinCaml spec]).

This project aims my practices for understanding type inference, closure transform and introducing own intermediate language (IL) to own language.

Example:

```ocaml
let rec gcd m n =
  if m = 0 then n else
  if m <= n then gcd m (n - m) else
  gcd n (m - n) in
print_int (gcd 21600 337500)
```

## Tasks

- [x] Lexer -> ([doc][lexer doc])
- [x] Parser with [goyacc][] -> ([doc][parser doc])
- [x] Alpha transform ([doc][alpha transform doc])
- [x] Type inference (Hindley Milner monomorphic type system) -> ([doc][typing doc])
- [x] GoCaml intermediate language (GCIL) ([doc][gcil doc])
- [x] K normalization from AST into GCIL ([doc][gcil doc])
- [x] Closure transform ([doc][closure doc])
- Optimizations
  - [ ] Beta reduction
  - [ ] Inlining
  - [ ] Folding constants
  - [ ] Striping unused variables
  - [ ] Reduce `()` type to `void`
  - [x] LLVM IR optimization passes
- [x] Code generation using [LLVM][]
- [x] Garbage collection with [Boehm GC][]
- [ ] Debug information (DWARF) using LLVM's Debug Info builder

## Difference from original MinCaml

- MinCaml assumes external symbols' types are `int` when it can't be inferred. GoCaml does not have such an assumption.
  GoCaml assumes unknown return type of external functions as `()` (`void` in C), but in other cases, falls into compilation error.
  When you use nested external functions call, you need to clarify the return type of inner function call. For example, when `f` in
  `g (f ())` returns `int`, you need to show it like `g ((f ()) + 0)`.
- MinCaml allows `-` unary operator for float literal. So for example `-3.14` is valid but `-f` (where `f` is `float`) is not valid.
  GoCaml does not allow `-` unary operator for float values totally.

## Requirements

- Go 1.2+ (Go 1.7+ is recommended)
- make
- Clang
- cmake (for building LLVM)
- Git

## Installation

```sh
$ go get -d github.com/rhysd/gocaml
$ cd $GOPATH/src/github.com/rhysd/gocaml

# Full-installation with building LLVM locally
$ make

# Use system-installed LLVM. You need to install LLVM in advance (see below)
$ USE_SYSTEM_LLVM=true make
```

If you want to use `USE_SYSTEM_LLVM`, you need to install LLVM 4.0.0 in advance.

If you use Debian-family Linux, use [LLVM apt repository][]

```sh
$ sudo apt-get install libllvm4.0 llvm-4.0-dev
```

If you use macOS, use [Homebrew][]. GoCaml's installation script will automatically detect LLVM
installed with Homebrew.

*Note:* LLVM 4.0 is now on an RC stage. So it doesn't come to Homebrew yet.

```sh
$ brew install llvm
```

And you need to install [libgc][] as dependency.

```sh
# On Debian-family Linux
$ sudo apt-get install libgc-dev

# On macOS
$ brew install bdw-gc
```

## Usage

`gocaml` command is available to compile sources. Please refer `gocaml -help`.

Compiled code will be linked to [small runtime][]. In runtime, some functions are defined to print values and it includes
`<stdlib.h>` and `<stdio.h>`. So you can use them from GoCaml codes.

## How to Work with C

All symbols not defined in source are treated as external symbols. So you can define it in C source and link it to compiled GoCaml
code after.

Let's say to write C code.

```c
#include <stdint.h>

// Please use int64_t for int in GoCaml, double for float in GoCaml, int for bool
int64_t plus100(int64_t const i)
{
    return i + 100;
}
```

Then compile it to an object file:

```
$ clang -Wall -c plus100.c -o plus100.o
```

Then you can refer the function from GoCaml code:

```ml
println_int ((plus100 10) + 0)
```

`println_int` is a function defined in runtime. So you don't need to care about it.
The `+ 0` is necessary to tell a compiler that the type of returned value of `plus100` is `int`. A compiler can know the type
via type inference.

Finally comile the GoCaml code and the object file together with `gocaml` compiler. You need to link `.o` file after compiling
GoCaml code by passing the object file to `-ldflags`.
```
$ gocaml -ldflags plus100.o test.ml
```

After the command, you can find `test` executable. Executing by `./test` will show `110`.

[MinCaml]: https://github.com/esumii/min-caml
[goyacc]: https://github.com/cznic/goyacc
[LLVM]: http://llvm.org/
[Linux and macOS Build Status]: https://travis-ci.org/rhysd/gocaml.svg?branch=master
[Travis CI]: https://travis-ci.org/rhysd/gocaml
[lexer doc]: https://godoc.org/github.com/rhysd/gocaml/lexer
[parser doc]: https://godoc.org/github.com/rhysd/gocaml/parser
[typing doc]: https://godoc.org/github.com/rhysd/gocaml/typing
[alpha transform doc]: https://godoc.org/github.com/rhysd/gocaml/alpha
[gcil doc]: https://godoc.org/github.com/rhysd/gocaml/gcil
[closure doc]: https://godoc.org/github.com/rhysd/gocaml/closure
[MinCaml spec]: http://esumii.github.io/min-caml/paper.pdf
[Boehm GC]: https://github.com/ivmai/bdwgc
[Coverage Status]: https://coveralls.io/repos/github/rhysd/gocaml/badge.svg
[Coveralls]: https://coveralls.io/github/rhysd/gocaml
[Windows Build Status]: https://ci.appveyor.com/api/projects/status/7lfewhhjg57nek2v/branch/master?svg=true
[Appveyor]: https://ci.appveyor.com/project/rhysd/gocaml/branch/master
[small runtime]: ./runtime/gocamlrt.c
[LLVM apt repository]: http://apt.llvm.org/
[Homebrew]: https://brew.sh/index.html
[libgc]: https://www.hboehm.info/gc/
