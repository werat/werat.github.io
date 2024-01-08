---
title: "Learning about debuggers"
date:   2024-01-08 13:00:00 +0200
tags:
   - debugging
   - debuggers
   - visual-studio
   - lldb
   - gdb
   - dwarf
---

Today's article is a collection of materials to learn more about debuggers: how they work, which technologies are under the hood, what kind of problems exist in this area. There is of course a big overlap with related components like compilers and linkers, so get ready to learn lots of new things 😃.

Of course, the list is not exhaustive by any means. These are just the links I've accumulated over the years and found useful for myself. If you'd like to add anything to the list, let me know!

> Note: many of the links here are blog posts -- be sure to check out other articles in those blogs! They're often worth reading even if not directly related to the topic.

<!--more-->

---

## How debuggers work

All debuggers share the same basic principles: they need to attach to a process, parse the binary and debug information, handle breakpoints, etc. The implementations may vary significantly between the operating systems, but after you understand one of them, others will look very familiar.

* **Writing a Linux Debugger**
  * <https://blog.tartanllama.xyz/writing-a-linux-debugger-setup/>
  * A 10-part journey of writing a Linux debugger from scratch. You'll learn the basics of `ptrace`, how breakpoints work, how to map source code to machine code and how to unwind the stack. Highly recommend starting with this one, if you wanna learn the fundamentals.
* **Writing a Debugger From Scratch**
  * <https://www.timdbg.com/posts/writing-a-debugger-from-scratch-part-1>
  * A still ongoing series about writing a Windows debugger in Rust. You'll learn the same basic concepts (attaching to process, handling breakpoints, etc), but specific for Windows world.
* **How Does a C Debugger Work? (GDB Ptrace/x86 example)**
  * <https://blog.0x972.info/?d=2014/11/13/10/40/50-how-does-a-debugger-work>
  * Good quick overview of how GDB works. Doesn't go into many details, but provides enough keywords for expanding the search.
* **Debugging with the natives**
  * <https://www.humprog.org/~stephen/blog/2016/02/25/#native-debugging-part-1>
  * <https://www.humprog.org/~stephen/blog/2017/01/30/#native-debugging-part-2>
  * Also a good overview. Part 1 explains things in plain English and part 2 goes into more details about debugger internals.
* **Engineering Record And Replay For Deployability** (aka "how does `rr` work?")
  * <https://arxiv.org/abs/1705.05937>
  * Amazing technical paper about `rr` (<https://rr-project.org/>). It goes into deep details about how record-and-replay works, explaining every bit of it. Highly recommend!
* **Implementation of Live Reverse Debugging in LLDB**
  * <https://arxiv.org/abs/2105.12819>
  * Great detailed paper about the implementation of the reverse debugging in LLDB. Gives a good overview of various options and their trade-offs and explains in details the approach chosed for LLDB.
* **How does gdb call functions?**
  * <https://jvns.ca/blog/2018/01/04/how-does-gdb-call-functions/>
  * Explains how one of the more magical things work in modern debuggers -- calling functions. The article says GDB, but the same applies to LLDB and other debuggers as well.

## Stack unwinding

Producing a call stack is one of the most basic debugger features, yet it's far from trivial. In order to understand how it works, you'll need to learn about frame pointers, call frame information (CFI), DWARF expressions and many other things. Here are some article that explore this topic in great detail.

* **Unwinding the stack the hard way**
  * <https://lesenechal.fr/en/linux/unwinding-the-stack-the-hard-way>
  * Great article about how stack unwinding works. It covers frame pointers (and why you can't rely on them), CFI, DWARF expressions and more.
* **Stack unwinding**
  * <https://maskray.me/blog/2020-11-08-stack-unwinding>
  * Another detailed explanation of stack unwinding. Goes a bit more into ABI and platform differences and the work compiler and linker have to do.
* **The Apple Compact Unwinding Format: Documented and Explained**
  * <https://faultlore.com/blah/compact-unwinding/>
  * Did I hear you say macOS?
* **Unwinding a Stack by Hand with Frame Pointers and ORC**
  * <https://blogs.oracle.com/linux/post/unwinding-stack-frame-pointers-and-orc>
  * Now that we've learned all about elfs and dwarves, it's time to get familiar with orcs. You thought unwinding the stack of your program is hard? Well how about unwinding the stack of the kernel.
* **DWARF-based Stack Walking Using eBPF**
  * <https://www.polarsignals.com/blog/posts/2022/11/29/profiling-without-frame-pointers>
  * Everything becomes better when you add eBPF into the mix.

## Random debugger trivia

I already gave up with the categorization, so dumping the rest under "random trivia".

* **Where Did My Variable Go? Poking Holes in Incomplete Debug Information**
  * <https://arxiv.org/abs/2211.09568>
  * A cool paper exploring how incomplete debug information affects the debugging quality.
* **Separating debug symbols from executables**
  * <https://www.tweag.io/blog/2023-11-23-debug-fission/>
  * Storing the debug information inside the executable is convenient, but may be impractical. This article is a great overview of how to split the binary and debug info and store them separately.
* **Debugger Lies: Stack Corruption**
  * <https://www.timdbg.com/posts/debugger-lies-part-1/>
  * What do you do if your stack is corrupted?
* **Everything Is Broken: Shipping rust-minidump at Mozilla**
  * <https://hacks.mozilla.org/2022/06/everything-is-broken-shipping-rust-minidump-at-mozilla/>
  * If you thought walking the stack of a living process is hard, how about walking the stack of a corpse ☠️?
* **Fuzzing rust-minidump for Embarrassment and Crashes**
  * <https://hacks.mozilla.org/2022/06/fuzzing-rust-minidump-for-embarrassment-and-crashes/>
  * Fuzzing saves the day (again), yay!
* **Crash reporting in Rust**
  * <https://jake-shadle.github.io/crash-reporting/>
  * Collecting crash information is very useful for debugging and root cause investigations, but unfortunately it's far from trivial...
* **So you want to live-reload Rust**
  * <https://fasterthanli.me/articles/so-you-want-to-live-reload-rust>
  * Live-reloading (or hot-reloading) is a fascinating topic and I wish it were more popular. The article goes into great details about how it could work, giving examples and exploring the depth of the abyss.

## General useful knowledge

* **Linkers**
  * <https://www.airs.com/blog/archives/38>
  * A 20-part series about linkers. Everything you wanted (and didn't want to) know about how linkers work. You can find the links for all 20 parts here -- <https://lwn.net/Articles/276782/>.
* **Design and implementation of mold**
  * <https://github.com/rui314/mold/blob/main/docs/design.md>
  * `mold` is the latest ultra-fast linker, beating `gold` and `lld`. The design & implementation section explains why it's so fast and which trade-offs were made to achieve that.
* **The Definitive Guide to Linux System Calls**
  * <https://blog.packagecloud.io/the-definitive-guide-to-linux-system-calls/>
  * Everything you wanted to know about system calls in Linux.
* **System calls in the Linux kernel**
  * <https://0xax.gitbooks.io/linux-insides/content/SysCall/linux-syscall-1.html>
  * How syscalls work from the kernel perspective. If you want to learn more about how the Linux kernel works, be sure the check out the rest of this book.

---

Discuss this article:

* <https://lobste.rs/s/cvdwwr/learning_about_debuggers>
* <https://news.ycombinator.com/item?id=38920207>
* <https://www.reddit.com/r/programming/comments/19200s8/learning_about_debuggers/>
