---
title: "Inline Variables in C++17"
date: 2020-08-03T19:26:20+02:00
draft: false
---

I watched Nir Friedman's [cppcon talk](https://www.youtube.com/watch?v=xVT1y0xWgww) on globals and the linker. This post is a hands-on to try `inline` variables in C++.

# What are inline variables?

[cppreference on inline specifier](https://en.cppreference.com/w/cpp/language/inline) says

> An inline function or variable (since C++17) with external linkage (e.g. not declared static) has the following additional properties: 
> 
> - There may be more than one definition of an inline function or variable (since C++17) in the program as long as each definition appears in a different translation unit and (for non-static inline functions and variables (since C++17)) all definitions are identical.
> - It must be declared inline in every translation unit.
> - It has the same address in every translation unit. 

If I were to compile an inline variable into a static library and a shared library, link an executable with it, I should end up with only one copy in the end.

# How does this work?

Let's compile this snippet and inspect how it appears to `nm`. We have a non-inlined variable `first` and an inlined variable `second`.

```cpp
#include <string>

namespace Test {
    std::string first = "first";
    inline std::string second = "second";
}
```

```shell script
# static library
g++ -std=c++17 test.cpp -c -o test.cpp.o
ar rcs libtest.a test.cpp.o

# shared library
g++ -std=c++17 test.cpp -c -fPIC -o test.cpp.o
g++ -shared -Wl,-soname,libtest.so -o libtest.so test.cpp.o
```

Run `nm -C` on the static library and `nm -CD` on the shared library to inspect the symbols. We `grep` for symbols under namespace `Test`.

## Static library

```
0000000000000000 u guard variable for Test::second[abi:cxx11]
0000000000000000 B Test::first[abi:cxx11]
0000000000000000 u Test::second[abi:cxx11]
```

## Shared library

```
00000000000051a0 u guard variable for Test::second[abi:cxx11]
0000000000005160 B Test::first[abi:cxx11]
0000000000005180 u Test::second[abi:cxx11]
```

The `nm` [manual](https://linux.die.net/man/1/nm) lists the types of symbols. 

> Lower case *b* is local, Upper case *B* is global
> "B" The symbol is in the uninitialized data section (known as BSS), global.
>
> "u" The symbol is a unique global symbol. This is a GNU extension to the standard set of ELF symbol bindings. For such a symbol the dynamic linker will make sure that in the entire process there is just one symbol with this name and type in use. 
>
> "U" The symbol is undefined. This should not be confused with lower case "u".

# Creating a binary

We write `main.cpp` to use the symbols and run `nm -C` on the `main` executable.

```cpp
#include <string>
#include <iostream>

namespace Test {
	extern std::string first;
	extern std::string second;
}

int main() {
	std::cout << Test::first << std::endl;
	std::cout << Test::second << std::endl;
}
```

## Only static library

```shell script
g++ -std=c++17 ./main.cpp -o main -L. -l:./libtest.a
```

```
0000000000002492 t _GLOBAL__sub_I__ZN4Test5firstB5cxx11E
0000000000005260 u guard variable for Test::second[abi:cxx11]
0000000000005220 B Test::first[abi:cxx11]
0000000000005240 u Test::second[abi:cxx11]
```

## Only shared library

```shell script
g++ -std=c++17 ./main.cpp -o main -L. -l:libtest.so -Wl,-rpath=./
```

```
00000000000041a0 B Test::first[abi:cxx11]
00000000000041c0 B Test::second[abi:cxx11]
```

Running `objdump` to list dynamic relocations,

```shell script
$ objdump -R main | grep Test
00000000000041a0 R_X86_64_COPY     Test::first[abi:cxx11]
00000000000041c0 R_X86_64_COPY     Test::second[abi:cxx11]
```

These symbols will be relocated at runtime. We run gdb to verify and these are indeed from the shared library.

```
0x5555555581b0 <_ZN4Test5firstB5cxx11E+16>:     102 'f' 105 'i' 114 'r' 115 's' 116 't'
0x5555555581d0 <_ZN4Test6secondB5cxx11E+16>:    115 's' 101 'e' 99 'c'  111 'o' 110 'n' 100 'd'
``` 

## Both static and shared library

### Static library first 

```shell script
g++ -std=c++17 ./main.cpp -o main -L. -l:libtest.a -l:libtest.so -Wl,-rpath=./
```

```
0000000000005260 u guard variable for Test::second[abi:cxx11]
0000000000005220 B Test::first[abi:cxx11]
0000000000005240 u Test::second[abi:cxx11]
```

`objdump -RC ./main | grep Test` is empty. This means there are no relocations. I rebuilt the shared library with `2` suffixed at the end of both strings. When we run `./main`, we now know the origin of a symbol.

```shell script
./main 
first
second2
```

### Shared library first

```shell script
g++ -std=c++17 ./main.cpp -o main -L. -l:libtest.a -l:libtest.so -Wl,-rpath=./

```

```
00000000000041a0 B Test::first[abi:cxx11]
00000000000041c0 B Test::second[abi:cxx11]
```

```shell script
./main 
first2
second2
```

Both the non-inlined variable `first` and the inlined variable `second` from the **static library** have their raw strings embedded in the binary in `.rodata` section. 

```
objdump -sC -j .rodata ./main

./main:     file format elf64-x86-64

Contents of section .rodata:
 3000 01000200 00000000 62617369 635f7374  ........basic_st
 3010 72696e67 3a3a5f4d 5f636f6e 73747275  ring::_M_constru
 3020 6374206e 756c6c20 6e6f7420 76616c69  ct null not vali
 3030 64006669 72737400 7365636f 6e6400    d.first.second.
```

> The linker searches from left to right, and notes unresolved symbols as it go. If a library resolves the symbol, it takes the object files of that library to resolve the symbol.
>
> [stackoverflow](https://stackoverflow.com/a/409470/3951920)

The non-inlined variable chooses the first occurrence in the library order, whereas the inlined variable chooses the dynamic linked symbol in both cases. What happens when we remove the dependency on `libtest.so` from `main` by touching the `DT_NEEDED` dynamic section? It falls back to the embedded `.rodata` definition from the static library.

```shell script
patchelf --remove-needed libtest.so ./main

patchelf --print-needed ./main
libstdc++.so.6
libm.so.6
libgcc_s.so.1
libc.so.6

./main 
first
second
``` 

Let's add it back and we are once again depending on the symbol from the dynamic library.

```shell script
patchelf --add-needed libtest.so ./main

./main 
first
second3
```
