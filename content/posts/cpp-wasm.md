---
title: "Compile C++ into WebAssembly"
date: 2022-09-29T15:36:14+08:00
lastmod: 2022-10-04T18:06:56-07:00
draft: false
keywords: []
description: ""
tags: []
categories: ["WebAssembly"]
author: Chuxi

# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: true
autoCollapseToc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright:
reward: false
mathjax: false
---

The support for WebAssembly (abbreviated Wasm) is a critical part of the VGG engine. Due to performance and cross-platform support considerations, the VGG engine is written in C++. It could be compiled into WebAssembly, so that we are able to run it in browsers. More importantly, it also supports user-generated Wasm files to be plugged into the designs.

By definition,
> WebAssembly is an executable binary format for a stack-based virtual machine.

There have been plenty of previous work, including blog posts, papers, books, etc, that help us understand the WebAssembly format. However, few of them focus on the compiling process of C++ to WebAssembly.

In this post, we share the process of using [Emscripten](https://emscripten.org/), a *de-facto* compiler toolchain for WebAssembly, to compile C++ code into WebAssembly. Hope you enjoy it!

<!--more-->

## Background of WebAssembly

Let's start the story with JVM[^jvm], the most famous virtual machine in programming history. JVM-based languages are a collection of languages that obey the JVM specification, including Java, Clojure, Scala, and Kotlin, so that they can run on JVM without cross-platform issues, while getting the power of JVM including GC[^gc], exception handling, multithreading, atomic operands, etc.

WebAssembly format actually borrows a lot from the JVM specification. Some details can be peeked in the paper *Bringing the Web up to Speed with WebAssembly*[^wasm-paper]. The authors also implemented a virtual machine to execute the WebAssembly format. As long as a programming language could be compiled into WebAssembly, it can be executed in this WebAssembly virtual machine. This is exactly what JVM does.

[^jvm]: Short for Java-Virtual-Machine
[^gc]: Short for Garbage-Collection
[^wasm-paper]: https://dl.acm.org/doi/10.1145/3062341.3062363

WebAssembly is designed for speeding up the Web. For example, we can use Photoshop in the browser. The ability of WebAssembly is greatly extended. It could help build portable standards for embedded devices so that we can narrow the gap between embedded hardware and software. As a consequence, more developers using high-level programming languages are able to deploy their products on tiny embedded hardware. It is the last puzzle of making everything intelligent.

Besides, the blockchain community is paying more attention to WebAssembly. For example, Ethereum executes contracts in its own virtual machine called EVM[^evm]. These contracts are written by Solidity language, a derivative programming language from Go. Developers cannot port the contracts to another chain unless it also supports EVM. What will happen if we use WebAssembly as the virtual machine format? Developers can write contracts in any programming language, then compile them to WebAssembly format. The contract is portable to any chains which support WebAssembly virtual machine. And it is the next virtual machine generation for most newer blockchain projects. We believe it will be the future for DeFi infrastructure.

[^evm]: Short for Ethereum-Virtual-Machine

Rust and C++, the two programming languages, are the primarily supported languages for WebAssembly generation. WebAssembly has the same linear memory model and reference table design as the C++ language. And the compiling to WebAssembly binaries is supported by **Clang**, which contains the WebAssembly target for LLVM[^llvm] framework. Clang is the C++ compiler constructed with modules from LLVM. It focuses on translating C++ languages to LLVM IR[^ir] and then uses some toolkits to generate executable binary files for the target platform.

[^llvm]: Short for Low-Level-Virtual-Machine
[^ir]: Short for Intermediate-Representation

To use the WebAssembly technology, we need to generate a wasm file and then execute it in the target environment. Generating a wasm file includes two steps, compiling and linking, which is the same as the C++ building process. Then the wasm file can be adapted with JavaScript and run in a browser, or just run in standalone runtime as a normal native process.

## Preparation

It's very convenient to set up a C++ WebAssembly building environment with `emsdk`, which helps us collect all the toolkits for compiling WebAssembly.

```bash
$ git clone https://github.com/emscripten-core/emsdk.git
```

After downloading `emsdk`, we will get all the tools under the directory `upstream`. We can check the installation by the command:

```bash
$ /path/to/emsdk/upstream/bin/clang -v
```

The `clang` executable is downloaded directly from official LLVM releases, unmodified. If we want to build LLVM, we can follow the instructions from [`test_release.sh`](https://github.com/llvm/llvm-project/blob/main/llvm/utils/release/test-release.sh).

Please note, `upstream/cache/sysroot` is a very important directory. It contains the header and library files for the subsequent compiling and linking.

## Compile C++ to WebAssembly with Clang

It is not easy to be a master at compiling C++ programs because the compiler has thousands of options to control the compiling and linking process. As a curious developer, we can debug the LLVM and clang source code to understand how it works. Let's start with the famous `hello world` program.

```cpp
#include <stdio.h>

int main() {
  printf("hello world\n");
  return 0;
}
```

Here we include the header file of `stdio.h`, which is contained in the C language library `libc`. Emscripten uses `musl` library and `libc` library customized for `wasi` environment, and copys the `libcxx`, `libcxxabi`, `libunwind` from LLVM, which are used to support the C++ language features.

To show how to compile the program, we split the progress into four stages: preprocessing, generating LLVM IR, generating assembly target object file, and linking.

### Preprocessing

```bash
$ /path/to/emsdk/upstream/bin/clang --target=wasm32 -E hello.c -v
```

- `--target=wasm32` defines the wasm32 target
- `-E` indicates that we only run preprocessing of the `hello.c` file.
- `-v` will print the details of the execution process.

In the end, we will encounter the error

```
hello.c:1:10: fatal error: 'stdio.h' file not found
#include <stdio.h>
```

It is about the missing header file `stdio.h`. C++ defines all function interfaces in header files. In the preprocessing step, the compiler will search the directories to find the header files. Here we are using the clang from `emsdk`. So it can not find the correct header file. And we notice that the current header search directories only include:

```
ignoring nonexistent directory "/include"
#include "..." search starts here:
#include <...> search starts here:
 /path/to/emsdk/upstream/lib/clang/15.0.0/include
 ```

 Then we can add `-I` to append the header search directory to resolve the problem. As usual, we can add the system header include directory. But the system-integrated headers and libraries are not adapted for WebAssembly. Emscripten offers all the basic headers and libraries required to build the wasm file. We can use the `--sysroot=/path/to/emscripten/cache/sysroot` instead.

 ```bash
$ /path/to/emsdk/upstream/bin/clang --target=wasm32 --sysroot /path/to/emsdk/upstream/emscripten/cache/sysroot -E hello.c -v
 ```

 `--sysroot` is only required with option `--target=wasm32`. So we can not use it in the normal C++ compilation process, otherwise it fails.

### LLVM Intermediate Presentation (IR)

It is the most important design in LLVM, that any languages compiled to IR format, could reuse the target platform code generation and assembling, with lots of optimization libraries. So WebAssembly is derived from the IR format and other LLVM-based programming languages could also easily be transformed into WebAssembly format.

```bash
$ /path/to/emsdk/upstream/bin/clang --target=wasm32 -I /path/to/emsdk/upstream/emscripten/cache/sysroot/include -emit-llvm -S hello.c -o hello.ll -v
```

- `-I` is used to prepend header search directories
- `-S` indicates to generate LLVM assembly file
- `-emit-llvm` will produce the LLVM intermediate representation
- `-o hello.ll` the `.ll` file format is the text format of IR

Besides, to understand how the optimization process works, we can add `-O1`or `-O2` and check the output. If we run the command without `-emit-llvm`, we can get a pure assembly format text file.

As to Emscripten, we find it adapts some header files to support WebAssembly. The wasm file is executed in a virtual machine. Currently, the WebAssembly instruction set does not support some system devices and kernel interfaces. So Emscripten needs to replace these functions when compiling.

### Target Object File

In this stage, the clang will use the LLVM IR to generate target platform object files. So LLVM will handle all the following work to construct an executable binary file.

```bash
$ /path/to/emsdk/upstream/bin/clang --target=wasm32 -I /path/to/emsdk/upstream/emscripten/cache/sysroot/include -c hello.c -o hello.o -v
```

- `-c` option will produce the target compiled object file.

It produces a `hello.o` binary file, which the linker could resolve the symbols with and do some optimization work in the linking stage.

### Linking

Let's run compiling and linking by separate command tools. Clang only chains the tools to complete building the executable target file. To show the linking progress, we only need to use the tool `wasm-ld`, instead of `ld`. More details could be found in [driver.cpp](https://github.com/llvm/llvm-project/blob/main/clang/tools/driver/driver.cpp)

```bash
$ /path/to/emsdk/upstream/bin/wasm-ld -o hello.wasm hello.o -L/path/to/emsdk/upstream/emscripten/cache/sysroot/lib/wasm32-emscripten -lc -lcompiler_rt --no-entry
```

- `wasm-ld` is the wasm linker in clang tools.
- `-o hello.wasm` indicates that the output is a wasm file.
- `hello.o` is the compiled object file.
- `-L...` prepends the object library search directory in the linking stage.
- `-lc` and `-lcompiler_rt` are used to add `libc` and `compiler_rt` library to linker when resolving symbols.
- `--no-entry` is used to avoid `entry symbol not defined _start` error.

If we look into the wasm file, we will find it has some Emscripten symbols inside.
```
wasm2wat hello.wasm -o hello.wat
```

where `wasm2wat` is a tool one of [Wabt](https://github.com/WebAssembly/wabt) toolkits, which is used for transforming a wasm binary file to a human-readable text format file.

## Compile C++ to WebAssembly with Emscripten

In the last chapter, we have compiled C++ programs to WebAssembly with clang toolchains. Actually, Emscripten helps organize the **driver process** to generate WebAssembly files, with a python tool called `emcc.py`.

```bash
$ /path/to/emsdk/upstream/emscripten/emcc.py -o hello.wasm hello.c -v
```

The parameter `-o hello.wasm` is very important as it will not generate the JavaScript glue codes.

We dumped out the commands in `emcc`, as it shows the best practice for compiling WebAssembly with Emscripten toolchains. By following the python debugger, we find the `emcc` tool split the whole process into three main phases: `phase_compile_inputs`, `phase_link`, `phase_post_link`.

The compile command in `emcc`

```
/path/to/emsdk/upstream/bin/clang
  -target wasm32-unknown-emscripten
  -DEMSCRIPTEN
  -D__EMSCRIPTEN_major__=3
  -D__EMSCRIPTEN_minor__=1
  -D__EMSCRIPTEN_tiny__=10
  -fignore-exceptions
  -fvisibility=default
  -mllvm -combiner-global-alias-analysis=false
  -mllvm -enable-emscripten-sjlj
  -mllvm -disable-lsr
  -Werror=implicit-function-declaration
  -Xclang -iwithsysroot/include/SDL
  --sysroot=/path/to/emsdk/upstream/emscripten/cache/sysroot
  -Xclang -iwithsysroot/include/compat
  hello.c -c -o hello.o
```

The link command in `emcc`

```
/path/to/emsdk/upstream/bin/wasm-ld -o he.wasm hello.o
  -L/path/to/emsdk/upstream/emscripten/cache/sysroot/lib/wasm32-emscripten
  /path/to/emsdk/upstream/emscripten/cache/sysroot/lib/wasm32-emscripten/crt1.o
  -lGL -lal -lhtml5 -lstandalonewasm -lstubs-debug -lc-debug -ldlmalloc -lcompiler_rt -lc++-noexcept -lc++abi-noexcept -lsockets
  -mllvm -combiner-global-alias-analysis=false
  -mllvm -enable-emscripten-sjlj
  -mllvm -disable-lsr
  --import-undefined
  --strip-debug
  --export-if-defined=__start_em_asm
  --export-if-defined=__stop_em_asm
  --export=emscripten_stack_get_end
  --export=emscripten_stack_get_free
  --export=emscripten_stack_get_base
  --export=emscripten_stack_init
  --export=stackSave
  --export=stackRestore
  --export=stackAlloc
  --export=__errno_location
  --export-table
  -z stack-size=5242880
  --initial-memory=16777216
  --max-memory=16777216
  --global-base=1024
```

Apparently, `emcc` has done a lot of optimization work in the compiling and linking process.

### Binaryen - the post link phase

Binaryen is another optimization tool integrated into Emscripten. It will extract the wasm binary file into a new AST (Abstract Syntax Tree), rather than the wasm plain stack format. Then this AST helps optimize the wasm file further in `-O3` level.

Binaryen will not be launched unless `-O3` parameter is passed to the `emcc`.

### WebAssembly API with Emscripten

The Emscripten documents have no update with the usage of WebAssembly API in JavaScript. Instead, it posts the solution with `cwrap` and `ccall` from `Module`. Apparently, it is easier and more convenient to use the `WebAssembly API`.

Let's create a `math.c` file.

```cpp
#include <stdlib.h>

extern void consoleLog(int arg);

int* alloc(int length) {
 return malloc(4 * length);
}

int factorial(int *arr, int length) {
 if (length <= 2) {
   return 0;
 } else if (length == 1) {
   return arr[0];
 } else if (length == 2) {
   return arr[1];
 }

 for (int i = 2; i < length; ++i) {
   arr[i] = arr[i - 2] * arr[i - 1];
   consoleLog(arr[i]);
 }
 return arr[length - 1];
}
```

In `math.c` we define an external function `consoleLog` which is imported from JavaScript, an `alloc` function used to do memory allocation, and a `factorial` function to test the operations. Then let's build the wasm file.

```bash
$ /path/to/emsdk/upstream/bin/clang --target=wasm32 -c -o math.o -I /path/to/emsdk/upstream/emscripten/cache/sysroot/include math.c -v
```

And in the linking step, we make use of the full command from `Emscripten`.

```bash
$ /path/to/emsdk/upstream/bin/wasm-ld -o math.wasm  math.o -L/path/to/emsdk/upstream/emscripten/cache/sysroot/lib/wasm32-emscripten /path/to/emsdk/upstream/emscripten/cache/sysroot/lib/wasm32-emscripten/crt1_reactor.o -lGL -lal -lhtml5 -lstandalonewasm -lstubs-debug -lnoexit -lc-debug -ldlmalloc -lcompiler_rt -lc++-noexcept -lc++abi-noexcept -lsockets -mllvm -combiner-global-alias-analysis=false -mllvm -enable-emscripten-sjlj -mllvm -disable-lsr --import-undefined --strip-debug --export-if-defined=factorial --export-if-defined=alloc --export-if-defined=__start_em_asm --export-if-defined=__stop_em_asm --export-if-defined=emscripten_stack_get_end --export-if-defined=emscripten_stack_get_free --export-if-defined=emscripten_stack_get_base --export-if-defined=emscripten_stack_init --export-if-defined=stackSave --export-if-defined=stackRestore --export-if-defined=stackAlloc --export-if-defined=__errno_location --export-table -z stack-size=5242880 --initial-memory=16777216 --entry=_initialize --max-memory=16777216 --global-base=1024
```

Using the option `--export-all` will make the linker import function `wasi_snapshot_preview1`, which leads to errors in JavaScript. So do not use the option in production.

Instead, we can use the `emcc` command directly to complete the whole building process.

```bash
$ emcc math.c --no-entry -sEXPORTED_FUNCTIONS=_factorial,_alloc -sERROR_ON_UNDEFINED_SYMBOLS=0 -o math-v2.wasm -O3 -v
```

- `-O3` will use `wasm-opt` to optimize the wasm file, which is a tool from `binaryen`
- `-sEXPORTED_FUNCTIONS=_factorial,_alloc` makes the `wasm-ld` add `--export-if-defined=factorial` and `--export-if-defined=alloc`
- `-sERROR_ON_UNDEFINED_SYMBOLS=0` is used to avoid the `node` `compiler.js` processing bug in `emcc`.

Then we add the following code in a file `index.html`

```html
<!DOCTYPE html>
<html lang="en">
<head>
   <meta charset="UTF-8">
   <title>Title</title>
</head>
<body>

<script>
   fetch("math-v1.wasm").then(response => response.arrayBuffer())
       .then(bytes => WebAssembly.instantiate(bytes, {
           env: {
               consoleLog: function (arg) {
                   console.log(arg);
               }
           }
       }))
       .then(target => {
           const ptr = target.instance.exports.alloc(10);
           const memory = new Int32Array(target.instance.exports.memory.buffer, ptr, 40);
           memory[0] = 2;
           memory[1] = 3;
           console.log("ptr = %d", ptr);

           const result = target.instance.exports.factorial(ptr, 8);
           console.log("result: %d", result);
       });
</script>

</body>
</html>

```

Open the `index.html` in chrome and we can get the output in the console.

```
ptr = 5244424
6
18
108
1944
209952
408146688
result: 408146688
```

## Conclusion

In this article, we reviewed how the `emcc` script works and extracted the `clang` commands. Then by using the WebAssembly API, we can easily integrate the packaged wasm file with the front-end JavaScript code. Thanks to the `LLVM` and `WebAssembly` text format, which helps a lot in the whole debugging process. And we post some suggestions on developing C/C++ with WebAssembly.
The `emcc` is a helpful tool for compiling C++ to wasm, but it's complicated to understand both `emcc` and `clang` options.
It's better to avoid using Emscripten to generate JavaScript glue codes. Instead, we can map all the interfaces in a Module with WebAssembly Module API.

## Contact us
* Discord: https://discord.gg/89fFapjfgM
<img width="595" alt="Group-Eng" src="https://user-images.githubusercontent.com/111478642/196371289-6e2b651a-e97b-4f93-adda-a9b8fa4cac58.png">

