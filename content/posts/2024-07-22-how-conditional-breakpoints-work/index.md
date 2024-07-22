---
title: "How conditional breakpoints work 🐢"
date:   2024-07-22 17:42:00 +0200
tags:
   - debugging
   - debuggers
   - visual-studio
   - lldb
   - expression-evaluation
   - lldb-eval
   - raddbg
   - delve
---

Conditional breakpoints are extremely useful, but everyone knows [citation needed] that they're super slow, to the point where people stop using them. Visual Studio recently did some [good improvements](https://x.com/VS_Debugger/status/1800617097623675381) and [@ryanjfleury](https://x.com/ryanjfleury) still [dunked on it](https://x.com/ryanjfleury/status/1801685216001724548) for being too slow. But even `raddbg` takes ~2 seconds to execute 10000 iterations of a simple loop with conditional breakpoints inside. For comparison, the same loop without breakpoints takes less than 1ms. So why is it so damn slow?

Let's explore how conditional breakpoints are typically implemented in modern debuggers, where the performance problems come from and what can be done to make things go fast.

<!--more-->

> Note: this article is about debuggers that work with "native" code, e.g. GDB, LLDB, Visual Studio C++. Debuggers for managed and scripting languages work similarly, but the implementation details may differ.

## Just check the condition?

When I say "conditional breakpoint" I mean a regular-looking breakpoint with a user-defined condition -- the program stops only when this condition evaluates to `true`. The condition is usually defined as an expression which uses the same (or very similar) language as the original program and can access the local/global variables. Therefore, you can do things like:

```golang
   1. func main() {
   2.   for i, user := range getUsers() {
🔴 3.     lastName := getLastName(user)        // condition: (i % 10 == 0)
🔴 4.     fmt.Printf("Hello, %s\n", lastName)  // condition: (lastName == "Hippo")
   5.   }
   6. }
```

If somebody asked you to implement this feature in an existing debugger, you'd probably quickly come up with a solution -- simply put a regular breakpoint, check the condition every time it's triggered and if the condition is false, silently resume the process. This doesn't require any architectural changes and can be implemented fairly easily in any debugger that already supports regular boring breakpoints.

### Follow the code

Let's follow the code in [LLDB](https://lldb.llvm.org/) and see how it's actually implemented there. To speed things up, we'll start at the API level. We quickly notice that there are lots of ways to create breakpoints (e.g. put it on a line, a symbol, or via regex), but none of them actually let us specify the condition ([SBTarget.h#L596](https://github.com/llvm/llvm-project/blob/b4f3a9662d308c869ac97e4f147edb38bc4f0626/lldb/include/lldb/API/SBTarget.h#L596)):

```c++
lldb::SBBreakpoint BreakpointCreateByLocation(
   const char *file, uint32_t line);
lldb::SBBreakpoint BreakpointCreateByName(
   const char *symbol_name, const char *module_name = nullptr);
lldb::SBBreakpoint BreakpointCreateByRegex(
   const char *symbol_name_regex, const char *module_name = nullptr);
lldb::SBBreakpoint BreakpointCreateForException(
   lldb::LanguageType language, bool catch_bp, bool throw_bp);
```

The breakpoint object, however, has a method with a telling name ([SBBreakpoint.h#L76](https://github.com/llvm/llvm-project/blob/b4f3a9662d308c869ac97e4f147edb38bc4f0626/lldb/include/lldb/API/SBBreakpoint.h#L76)):

```c++
class LLDB_API SBBreakpoint {
public:
  void SetCondition(const char *condition);
};
```

This method doesn't do much, though. It simply saves the condition as a string in the breakpoint options:

> [SBBreakpoint::SetCondition](https://github.com/llvm/llvm-project/blob/b4f3a9662d308c869ac97e4f147edb38bc4f0626/lldb/source/API/SBBreakpoint.cpp#L271) -> [Breakpoint::SetCondition](https://github.com/llvm/llvm-project/blob/b4f3a9662d308c869ac97e4f147edb38bc4f0626/lldb/source/Breakpoint/Breakpoint.cpp#L398) -> [BreakpointOptions::SetCondition](https://github.com/llvm/llvm-project/blob/b4f3a9662d308c869ac97e4f147edb38bc4f0626/lldb/source/Breakpoint/BreakpointOptions.cpp#L461)

So far it matches our first guess: conditional breakpoint is a regular breakpoint with a condition saved next to it. What happens next is also what we expect -- when the breakpoint is triggered, the process is stopped and the debugger checks if there's a condition associated with the breakpoint. If yes, then it tries to evaluate it and automatically continues the process if the condition is false. The code for this is quite long, so here’s a relevant snippet (slightly reformatted from [Target/StopInfo.cpp#L447](https://github.com/llvm/llvm-project/blob/b4f3a9662d308c869ac97e4f147edb38bc4f0626/lldb/source/Target/StopInfo.cpp#L447)):

```c++
if (bp_loc_sp->GetConditionText() == nullptr)
   actually_hit_any_locations = true;
else {
   Status condition_error;
   bool condition_says_stop =
   bp_loc_sp->ConditionSaysStop(exe_ctx, condition_error);
   ...
   if (condition_says_stop)
   actually_hit_any_locations = true;
   ...
}
...
if (!actually_hit_any_locations) {
   // In the end, we didn't actually have any locations that passed their
   // "was I hit" checks.  So say we aren't stopped.
   GetThread()->ResetStopInfo();
   LLDB_LOGF(log, "Process::%s all locations failed condition checks.", __FUNCTION__);
}
```

The `ConditionSaysStop()` is an interesting function here. So far we haven't clarified what a condition _is_, we've only passed it as a string. Well, turns out the condition can be (almost) any C++ code! The official documentation is not very explicit about this, but [one of the examples](https://lldb.llvm.org/use/map.html#set-a-conditional-breakpoint) features an expression with a cast and a function call. How does LLDB handle this? Following the implementation of `ConditionSaysStop()`, we can see that the debugger [parses the expression](https://github.com/llvm/llvm-project/blob/b4f3a9662d308c869ac97e4f147edb38bc4f0626/lldb/source/Breakpoint/BreakpointLocation.cpp#L264) and then [executes it](https://github.com/llvm/llvm-project/blob/b4f3a9662d308c869ac97e4f147edb38bc4f0626/lldb/source/Breakpoint/BreakpointLocation.cpp#L295). It has a nice optimization of caching the parsed expression between the executions.

For parsing the expression the debugger [uses `clang` compiler](https://github.com/llvm/llvm-project/blob/f65d7fdcf81f1fb01df3446b254f3304589f19c4/lldb/source/Plugins/ExpressionParser/Clang/ClangUserExpression.cpp#L645), which enables it to support very complex expressions. The string expression is converted to LLVM IR, which can be sent for execution. If the produced IR is ["simple enough"](https://github.com/llvm/llvm-project/blob/f65d7fdcf81f1fb01df3446b254f3304589f19c4/lldb/source/Expression/IRInterpreter.cpp#L523), then it can be [interpreted by the debugger](https://github.com/llvm/llvm-project/blob/f65d7fdcf81f1fb01df3446b254f3304589f19c4/lldb/source/Expression/LLVMUserExpression.cpp#L95). However, if it contains some non-trivial instructions, then the expression is JIT-compiled, injected into the target process and executed there.

## Why so slow?

You might be surprised, but that's exactly how most modern debuggers implement conditional breakpoints. For example, a Golang debugger [`delve`](https://github.com/go-delve/delve) accepts [Go-like expressions as conditions](https://github.com/go-delve/delve/blob/3ae22627df02ecd12b7daf7ac919ee70fb7093e0/service/debugger/debugger.go#L923) and then evaluates them [whenever the breakpoint is hit](https://github.com/go-delve/delve/blob/3ae22627df02ecd12b7daf7ac919ee70fb7093e0/pkg/proc/breakpoints.go#L450). Another example is the recently open-sourced C/C++ debugger [`raddbg`](https://github.com/EpicGamesExt/raddebugger) -- it stores the conditions as [expression strings](https://github.com/EpicGamesExt/raddebugger/blob/d143fec0d146ef0463b18cfd76ef094c81ac8250/src/ctrl/ctrl_core.h#L216) and then evaluates them [every time the breakpoint is triggered](https://github.com/EpicGamesExt/raddebugger/blob/d78afc50002e981ca63467494231d0abfebe3baa/src/ctrl/ctrl_core.c#L4628).

There are two operations in this approach that look like they could be slow -- stopping the process at the breakpoint and evaluating the condition. Let's dig deeper into each one of them and see where the performance degradation happens.

### Stopping the world

A quick recap of how breakpoints work. The debugger patches the process code -- it writes a special `int3` trap instruction at the address where the process should stop. When the CPU executes this instruction, the interrupt is raised, the process is stopped and the control is given over to the debugger. Check <https://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints> or <http://www.nynaeve.net/?p/=80> for better and longer explanations.

The problem with checking the condition whenever the breakpoint is hit is that the debugger must actually stop at the breakpoint every single time. Even if the condition will later turn out to be false, we're still paying the price of stopping and resuming the process. This may not be as noticeable for the code paths that don't trigger often, but breakpoints in tight loops will definitely affect the performance. A single stop-resume cycle requires several back and forths with the debugger:

0. (assume we already have a breakpoint set somewhere)
1. The breakpoint is hit -- an interrupt is raised and the processed is paused
2. Overwrite `int3` with the original instruction
3. Resume the process in the execute-single-instruction mode
4. Write `int3` back again
5. Resume the process

Things get much worse when debugging remote processes, as in this case the network overhead will play a major role. Imagine you have 1ms round-trip latency to the remote process. This means stopping and resuming the process already takes ~5ms without accounting for the processing time. So if you have a breakpoint inside a loop, you won't get more than 200 iterations per second 🐢🐢🐢.

### Executing code while executing code

Evaluating breakpoint conditions can be tricky as well. Most debuggers already have this capability in some form, as it's often used for scripting and data visualization. Typically not the whole "source" language is supported, but rather its subset or even a similar-looking scripting language. Sigh, implementing a fully fledged C++ interpreter just for evaluating the conditions [is far from trivial](https://werat.dev/blog/blazing-fast-expression-evaluation-for-c-in-lldb/) :D

The performance of the expression evaluation varies A LOT across different debuggers. As discussed earlier, LLDB actually tries to support (almost) any valid C++ code and it uses `clang` to compile and execute it -- as you can imagine, things get slow real fast. In `delve` the conditions are Go code, which is much easier to parse than C++. The produced expression ASTs are re-used, but if the user enters a long and "heavy" expression, then things will get slow. I don't know the exact syntax of the expression language supported by `raddbg`, but I suspect it won't let you enter anything outrageous. Hence it can afford doing parsing and evaluation from scratch every time.

In theory, the expression evaluation can be optimized to be almost negligible, but the overhead of stopping the process doesn't go away. But if we want to improve the performance significantly, we should think of a way _not_ to stop the process every time we need to check the condition.

## Journey to the Center of the Process

We know already that the debugger can patch the process code -- this is how software breakpoints work. What if instead of inserting the trap instruction, we inject some code to check the condition and break only when it's true?

This sounds great in theory, but comes with a few of challenges:

* The trap instruction (`int3`) is a single byte opcode, so it can be inserted anywhere. The "code to check the condition" can be arbitrarily large in size, so we can't just put it at an arbitrary location.
* The process code is machine code, so we first need to compile our conditional expression down to the machine code. This is a highly platform-dependent exercise and we basically need to implement a real compiler.

The first problem doesn't really have a perfect solution. The best we can do is to allocate some space in the program for storing our code and then inject the `jmp` instructions in places where we need to check the condition. A jump takes more than one byte, so it can't be used _everywhere_, but it could be enough in many cases. Allocating memory in the program can cause side-effects, but hey, you wanted to go fast, right?

For the second problem we have options. Some debuggers provide "agents" or "sidecars" that can be injected into the process to host some logic. A good example is GDB and its [In-Process Agent](https://sourceware.org/gdb/current/onlinedocs/gdb.html/In_002dProcess-Agent.html#In_002dProcess-Agent) capability. It was designed specifically for situations where it's not desirable to stop the process and some form of in-process execution is preferred.

The agent is a shared library that is loaded into the target process and has a small virtual machine that can evaluate simple [byte-code based expressions](https://sourceware.org/gdb/current/onlinedocs/gdb.html/Agent-Expressions.html). This means we don't need to compile our expression down to the machine code, only to the byte-code the agent can understand. Of course, for complex expressions this may still be complicated, but much easier than dealing with platform- and architecture-specific details.

> Similar techniques are used by tools that leverage dynamic instrumentation. For example, the performance profiler [Orbit](https://github.com/google/orbit) can dynamically patch process code to collect _precise_ profiling information instead of relying on sampling.

Executing the break conditions directly in the process code can dramatically (and I mean DRAMATICALLY) improve the performance. The `UDB` debugger leverages the GDB's in-process agent for evaluating breakpoint conditions and the measured performance improvements are ~1000x -- <https://www.youtube.com/watch?v=gcHcGeeJHSA>.

If we want to squeeze the last bits of perf and go _really_ fast, then we have to compile the expressions after all. I'm not aware of any native debugger that actually does that, but LLDB comes pretty close. It has a built-in JIT compiler that's already used for evaluating breakpoint conditions, but it still follows the standard stop-evaluate-continue model. The idea of using JIT to speed up conditional breakpoints has been [flying in the air](https://lldb.llvm.org/resources/projects.html#use-the-jit-to-speed-up-conditional-breakpoint-evaluation) for a while now, however, even though many of the building blocks are already there, there are still enough non-trivial implementation details. Maybe one day...

---

Discuss this article:

* <https://lobste.rs/s/woj8tt/how_conditional_breakpoints_work>
* <https://news.ycombinator.com/item?id=41036522>
* <https://www.reddit.com/r/programming/comments/1e9j2ah/how_conditional_breakpoints_work/>
