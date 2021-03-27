---
title: "Debugging LLDB with source stepping"
date:   2020-10-11 18:35:00 +0100
tags:
   - lldb
   - debugging
aliases:
  - /2020/10/11/debugging-lldb
  - /2020/10/11/debugging-lldb.html
---

Sometimes you want to (or _need_ to) debug a program that you didn't build yourself and you don't even know _how exactly_ it was built. Depending on the specifics of your setup that could mean many different things:

* Built on a build farm, running on a production server/container
* Installed via `apt` or similar, running locally
* ...

This post is inspired by my experience of debugging [LLDB](https://lldb.llvm.org/). Debugging the debugger is always interesting and tricky, even without the additional difficulties like trying to get the source stepping to work :)

The LLDB I was debugging was installed via `apt` on Ubuntu (namely `lldb-10`). Unfortunately, the binary I built from source myself didn't reproduce the issue, so I was stuck with debugging the prebuilt one. At least the debug symbols are available, you just need to install them separately (you're quite lucky if the already has them!). In my case that means `liblldb-10-dbgsym`.

Now if you try debugging you'll see that the debugger shows you an assembly rather than the source code, even though the debug info has the information about the context (source files, line numbers, etc):

```bash
(lldb) continue
Process 3096063 resuming
Process 3096063 stopped
* thread #1, name = 'exec', stop reason = breakpoint 2.2
    frame #0: 0x00007fffefdcb750 liblldb-10.so.1`lldb::SBTarget::CreateValueFromData(char const*, lldb::SBData, lldb::SBType)
liblldb-10.so.1`lldb::SBTarget::CreateValueFromData:
->  0x7fffefdcb750 <+0>: pushq  %rbp
    0x7fffefdcb751 <+1>: pushq  %r15
    0x7fffefdcb753 <+3>: pushq  %r14
    0x7fffefdcb755 <+5>: pushq  %r13
```

This is because the debugger doesn't have the source code! When you're debugging the binary you've just built yourself this is not an issue, because the debug info references your local source code. But since the binary was built somewhere else, the original source code is not available anymore.

Luckily, all modern debuggers can do "source mapping" -- you can tell where the source code is and the debugger will use it for all matched references. In LLDB one can use `target.source-map`:

```bash
(lldb) settings set target.source-map /buildbot/path /my/path
```

First we need to figure out the _original_ source path, i.e. the one used for building the target binary (`/buildbot/path` in the example above). Since we didn't build the binary, we don't know the path. But that's not a problem, because it's saved somewhere in the debug info (otherwise the debuggers wouldn't be able the source stepping at all).

The easiest way to extract it is to use `image lookup` while debugging:

```bash
(lldb) image lookup -vn CreateValueFromData
2 matches found in /usr/lib/x86_64-linux-gnu/liblldb-10.so.1:
        Address: liblldb-10.so.1[0x0000000000342750] (liblldb-10.so.1.PT_LOAD[0]..text + 1445344)
        Summary: liblldb-10.so.1`::CreateValueFromData() at SBTarget.cpp:1488
         Module: file = "/usr/lib/x86_64-linux-gnu/liblldb-10.so.1", arch = "x86_64"
    CompileUnit: id = {0x00000031}, file = "/build/llvm-toolchain-10-GjIltB/llvm-toolchain-10-10.0.0/lldb/source/API/SBTarget.cpp", language = "c++14"
       Function: id = {0x7fffffff002e1822}, name = "::CreateValueFromData()", range = [0x00007ffff712d750-0x00007ffff712db76)
       FuncType: id = {0x7fffffff002e1822}, byte-size = 0, compiler_type = "void (void)"
         Blocks: id = {0x7fffffff002e1822}, range = [0x7ffff712d750-0x7ffff712db76)
      LineEntry: [0x00007ffff712d750-0x00007ffff712d77c): /build/llvm-toolchain-10-GjIltB/llvm-toolchain-10-10.0.0/lldb/source/API/SBTarget.cpp:1488
         Symbol: id = {0x0000b0fc}, range = [0x00007ffff712d750-0x00007ffff712db76), name="lldb::SBTarget::CreateValueFromData(char const*, lldb::SBData, lldb::SBType)", mangled="_ZN4lldb8SBTarget19CreateValueFromDataEPKcNS_6SBDataENS_6SBTypeE"
...
```

(`CreateValueFromData` is some random function from the binary we're inspecting)

Bingo! The source path prefix we're looking for is `/build/llvm-toolchain-10-GjIltB/llvm-toolchain-10-10.0.0`.

---
Another way is to inspect the debug info directly.

Lookup the location of the debug info:

```bash
> readelf --debug-dump /usr/lib/llvm-10/lib/liblldb.so
Contents of the .gnu_debuglink section:

  Separate debug info file: 617fb09b6875c831220b21d028cc7443dda882.debug
  CRC value: 0x3e9d3f54
```

Peek into the actual DWARF:

```bash
> readelf --debug-dump=info /usr/lib/debug/.build-id/4b/617fb09b6875c831220b21d028cc7443dda882.debug
...
Contents of the .debug_info section:

  Compilation Unit @ offset 0x0:
   Length:        0xdd39 (32-bit)
   Version:       4
   Abbrev Offset: 0x0
   Pointer Size:  8
 <0><b>: Abbrev Number: 1 (DW_TAG_compile_unit)
    <c>   DW_AT_producer    : (indirect string, offset: 0x3bded7): Debian clang version 10.0.0-4
    <10>   DW_AT_language    : 33       (C++14)
    <12>   DW_AT_name        : (indirect string, offset: 0x1de52): /build/llvm-toolchain-10-GjIltB/llvm-toolchain-10-10.0.0/lldb/source/API/SBAddress.cpp
    <16>   DW_AT_stmt_list   : 0x0
    <1a>   DW_AT_comp_dir    : (indirect string, offset: 0x4d5f6): /build/llvm-toolchain-10-GjIltB/llvm-toolchain-10-10.0.0/build-llvm/tools/clang/stage2-bins/
tools/lldb/source/API
    <1e>   DW_AT_low_pc      : 0x0
    <26>   DW_AT_ranges      : 0x3290
```

The path is in the `DW_AT_name` attribute.

---

Now that we have a prefix we can remap the sources in the debugger via `target.source-map`:

```bash
settings set target.source-map /build/llvm-toolchain-10-GjIltB/llvm-toolchain-10-10.0.0/ /home/werat/src/llvm-project/
```

... and starting debugging!

```bash
Process 1967456 stopped
* thread #1, name = 'main', stop reason = step over
    frame #0: 0x00007ffff7039cec liblldb-10.so.1`::Initialize() at SBDebugger.cpp:152:3
   149  }
   150
   151  void SBDebugger::Initialize() {
-> 152    LLDB_RECORD_STATIC_METHOD_NO_ARGS(void, SBDebugger, Initialize);
   153    SBError ignored = SBDebugger::InitializeWithErrorHandling();
   154  }
   155
(lldb) n
Process 1967456 stopped
* thread #1, name = 'main', stop reason = step over
    frame #0: 0x00007ffff7039d55 liblldb-10.so.1`::Initialize() at SBDebugger.cpp:153:21
   150
   151  void SBDebugger::Initialize() {
   152    LLDB_RECORD_STATIC_METHOD_NO_ARGS(void, SBDebugger, Initialize);
-> 153    SBError ignored = SBDebugger::InitializeWithErrorHandling();
   154  }
   155
   156  lldb::SBError SBDebugger::InitializeWithErrorHandling() {
(lldb) s
Process 1967456 stopped
* thread #1, name = 'main', stop reason = step in
    frame #0: 0x00007ffff7039e1f liblldb-10.so.1`::InitializeWithErrorHandling() at SBDebugger.cpp:157
   154  }
   155
   156  lldb::SBError SBDebugger::InitializeWithErrorHandling() {
-> 157    LLDB_RECORD_STATIC_METHOD_NO_ARGS(lldb::SBError, SBDebugger,
   158                                      InitializeWithErrorHandling);
   159
   160    SBError error;
```

The steps above are quite typical when debugging non-locally-built binaries (e.g. debugging your payment service in production, fun 🤠 ). Depending on what kind of debug info is actually available you might not be able to do all the the usual debug operations you're used to -- for example, inspect local variables via `frame variable foo`.

But hey, that's still better than decoding the assembly yourself, right?
