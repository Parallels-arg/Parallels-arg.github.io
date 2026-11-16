---
title: "Binary Gecko Academy 2026: the qualifying challenge"
description: "Four chained levels in a MinGW crackme: username hashes, RC4, anti-debugging and a bytecode VM, solved without patching and with a keygen for each level."
date: 2026-11-16 12:00:00 -0300
categories: [Writeups, Reversing]
tags: [crackme, windows, keygen, x64dbg, binary-ninja, anti-debugging, vm]
translated: true
media_subpath: /assets/img/posts/binary-gecko-academy-2026
---

This is the writeup I submitted for the qualifying challenge of the [Binary Gecko](https://binarygecko.com/) Academy 2026, published now that the selection process is over. The analysis is the same one I sent; for the blog I turned the screenshots of my scripts into code blocks you can copy, and left the challenge binary out.

## Summary

`GeckoAcademy2026_7_challenge.exe` is a 64-bit console program compiled with MinGW, with four chained levels and anti-debugging checks along the way. I solved all four without patching the binary, and wrote a Python keygen for each one:

| Level | What it checks | My input |
| --- | --- | --- |
| 1 | An 8-digit hex serial that must match a hash of the username | Username `Bruno`, serial `0038D6D8` |
| 2 | A XXXX-XXXX-XXXX-YYYY serial: three 16-bit hashes of the same username with different seeds, plus the XOR of the three | `D31B-84B1-76D4-217E` |
| 3 | A login: the password is built from a new username and an 8-byte key decrypted with RC4 | Username `Bruno`, password `AFB0685485B42B8B` |
| 4 | A 12-character password validated by a small bytecode VM. There are multiple possible passwords. | `Bruno_W!59*.` |

Completion code: `937F16DE-EB8C1201`

The completion code depends on everything entered in the run, so it's only valid for the inputs above (details in [the completion code](#the-completion-code) section).

## Initial assessment

I first downloaded the challenge on September 21st. The zip (password `a`) contained `GeckoAcademy2026_7_challenge.exe` and a `notas.txt`:

![Notepad++ showing notas.txt from the first download](img-001.webp){: w="886" h="253" }

When I downloaded it again on September 22nd, the note had been replaced by `Qualifying requirements.txt`, so it looks like the original note was just a slip-up by the organizers, oops! The executable also had a different file date, so I compared the SHA-256 hashes to check whether the binary itself had changed:

![SHA-256 hashes of the executables from the first two downloads](img-002.webp){: w="1880" h="248" }

Both copies had the same SHA-256: `C7868C66050F75AE9BAE7E8116D74BF04BA8B5A1C5D0A1B9CD0B7457BB02ED2F`. A third download on October 14th matched as well:

![SHA-256 hash of the third download, matching the others](img-003.webp){: w="1880" h="267" }

All three executables were identical. I used the copy from the second download for the analysis below.

### Ground rules

Before starting, I set some rules for myself:

1. **Deep understanding**. This year I caught the challenge early and there's plenty of time before the deadline, so I'm going to take my time with it. For every level I'll try to fully understand what the check does, document it, and try to find all the valid inputs not just one that happens to get through.
1. **No patching as a solution**. Last year I NOPed out some instructions to get past a crash in the challenge executable, without ever understanding why the crash was happening in the first place. This year I will leave the binary untouched, and if something crashes or gets in my way, the goal is to understand why.
1. **Confirmation through dynamic analysis**. I'll read the code in the disassembler first and use the debugger to check my assumptions.
1. **Environment**. Windows 11 x64. Given the organizers' reputation, I'm comfortable running the executable directly on my machine, without a VM or any further security measures.

### Tooling

| Tool | Used for |
| --- | --- |
| Detect It Easy 3.21 | Initial file identification |
| PEStudio 9.61 | Initial file identification |
| Binary Ninja 6.0 | Main disassembler / decompiler |
| Calltree (Binary Ninja plugin) | Caller/callee trees to get the big picture of the program |
| x64dbg | Dynamic analysis |
| Python 3.13 | Keygens and solution scripts |

### Documentation I relied on

| Source | Used for |
| --- | --- |
| [mingw-w64 CRT source, stdio/](https://mingw.googlesource.com/mingw-w64/+/refs/heads/master/mingw-w64-crt/stdio/) | MinGW-w64's own implementations of the C stdio functions (printf, sscanf, snprintf...), to identify them in the binary |
| [Stack Overflow: Using msvcrt with Mingw-w64](https://stackoverflow.com/questions/64158676/using-msvcrt-with-mingw-w64) | Why MinGW-w64 uses its own implementations instead of the ones in msvcrt.dll (msvcrt.dll only covers C89, GCC compiles as C11) |
| [Binary Ninja docs: basic types](https://docs.binary.ninja/guide/types/basictypes.html) | Changing the type of a function |
| [binaryninja-printf](https://github.com/nshp/binaryninja-printf) | Confirming that Binary Ninja needs help with variadic printf-like functions |
| [cppreference: scanf](https://en.cppreference.com/w/c/io/fscanf) | How %x parses a hex number (spaces, 0x prefix, where it stops) |
| [Microsoft Learn: chcp](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/chcp) | The console code page, and why accented letters reach the binary in cp850 |
| [Microsoft Learn: GetOEMCP](https://learn.microsoft.com/en-us/windows/win32/api/winnls/nf-winnls-getoemcp) | The system OEM code page used by my first fix, which is not necessarily the console input code page |
| [Microsoft Learn: GetConsoleCP](https://learn.microsoft.com/en-us/windows/console/getconsolecp) | Reading the current console input code page for the keygens |
| [Wikipedia: RC4](https://en.wikipedia.org/wiki/RC4) | Recognizing the key scheduling and the generator of RC4 in level 3 |
| [x64dbg docs: HideDebugger](https://help.x64dbg.com/en/latest/commands/misc/HideDebugger.html) | Hiding x64dbg from IsDebuggerPresent in level 3 |
| [Cracking a virtualized binary with Angr](https://noh.am/en/posts/vm/) | How a VM works inside a crackme (bytecode, dispatcher, handlers) and the manual way to reverse it, for level 4 |
| [x64dbg docs: conditional breakpoints](https://help.x64dbg.com/en/latest/introduction/ConditionalBreakpoint.html) | Breakpoints that log a line instead of stopping, to trace the VM of level 4 |
| [x64dbg docs: string formatting](https://help.x64dbg.com/en/latest/introduction/Formatting.html) | The {expression} syntax of the log text, printed in hex by default |
| [x64dbg docs: values](https://help.x64dbg.com/en/latest/introduction/Values.html) | Reading memory in an expression with [addr] and dword:[addr] |
| [Baeldung: meet-in-the-middle attack](https://www.baeldung.com/cs/security-meet-in-the-middle-attack) | The trick used to find level 4 passwords, going forwards and backwards and meeting in the middle |
| [Wikipedia: ASCII printable characters](https://en.wikipedia.org/wiki/ASCII) | The range of characters that can be typed in a level 4 password (0x21 to 0x7E) |
| [Wikipedia: Fowler–Noll–Vo hash function](https://en.wikipedia.org/wiki/Fowler%E2%80%93Noll%E2%80%93Vo_hash_function) | Recognizing the FNV-1a constants (0x811c9dc5, 0x1000193) in the completion code |

### Basic analysis

First stop, Detect It Easy:

![Detect It Easy: 64-bit console PE, MinGW GCC 13, C](img-004.webp){: w="727" h="532" }

A 64-bit console PE, but this time compiled with `MinGW (GCC 13)`, language C, and **stripped**.

Before going further I checked the entropy, just to make sure I'm not looking at a packed binary:

![Detect It Easy entropy view, 6.16 and not packed](img-005.webp){: w="2560" h="1032" }

6.16 overall, and DIE says "not packed". Going section by section, the highest one is `.text` with 6.27. Nothing gets close to 7-8, which is what is expected from compressed or encrypted data.

Next, the sections:

![Section table of the binary](img-006.webp){: w="1566" h="269" }

A quick Google search showed that this is the standard section layout of a binary built with MinGW, so nothing added by hand.

The binary also has **TLS callbacks**, two of them, both pointing into `.text`:

![Detect It Easy listing two TLS callbacks](img-007.webp){: w="2560" h="1032" }

I wrote down `0x140002960` and `0x140002930` to check them later in the disassembler.

PEStudio confirms what DIE showed and also gives the compile timestamp:

![PEStudio overview with the compile timestamp](img-008.webp){: w="1234" h="741" }

### Imports

Only two libraries: `KERNEL32.dll` and `msvcrt.dll`. The KERNEL32 imports are where it gets interesting:

![KERNEL32.dll imports in PEStudio, with the anti-debugging APIs flagged](img-009.webp){: w="854" h="405" }

- `IsDebuggerPresent` and `CheckRemoteDebuggerPresent`: the names say it all, debugger checks. Very consistent with last year's challenge.
- `GetThreadContext` together with `GetCurrentThread`: looking it up, that's a way for a program to read its own debug registers (DR0-DR3), which is where hardware breakpoints live.
- `QueryPerformanceCounter` / `QueryPerformanceFrequency`: most likely used for the debugger checks too, since last year's challenge also had timing checks.
- `VirtualProtect` and `VirtualQuery` are flagged, but I can't say much about them yet.

The `msvcrt.dll` side is the C runtime:

![msvcrt.dll imports in PEStudio](img-010.webp){: w="854" h="781" }

### Strings

PEStudio found 1129 strings, most of them noise:

![PEStudio strings with the level banners and anti-debugging messages](img-011.webp){: w="1178" h="781" }

Looking through them, I can see that the challenge has 4 levels, along with anti-debugger messages, format strings and the existence of a "completion code".

With this I have a good idea of what I'm up against: 4 levels in plain C, one or more anti-debugging checks, two TLS callbacks still to check, and a completion code. Time to open it in Binary Ninja.

## Getting oriented

### Setting up Binary Ninja

Binary Ninja found `main` on its own as soon as the analysis finished, which is a nice start. Before reading any level, though, I wanted a way to see the whole program at a glance, meaning who calls who, which helpers are shared between levels, and so on. For that I installed **Calltree** from the Extension Manager. It adds a sidebar with the incoming and outgoing calls of the current function, and a button that turns those trees into a graph:

![Binary Ninja with the Calltree sidebar](img-012.webp){: w="1282" h="832" }

### A first look at `main`

This is how `main` looked right after opening the binary:

![Decompiled main before renaming, with nested calls](img-013.webp){: w="1282" h="832" }

The structure is already there. Some initialization, a banner, four calls (the first two share the same `var_88`), and a last function that prints the completion code. But the calls to `sub_14000fb30` end up nested inside each other and even inside the first level's arguments, which makes no sense for a banner.

Every line of the banner goes through the same function, `sub_14000fb30`. Clicking on it highlights all its uses, and the Cross References panel shows it's called 40 times across the program, always with a string as the first argument: the banner, `"Username: "`, `"Serial (8 hex digits): "`, `"\n[!] %s\n"`, and some of them have format specifiers like `%s` or `%08X`:

![Cross references of sub_14000fb30, always called with a format string](img-014.webp){: w="1210" h="638" }

That looks like some kind of printf, so I opened it:

![sub_14000fb30 and the two wrappers below it](img-015.webp){: w="670" h="459" }

It saves the extra arguments on the stack and calls another function with `data_140010100(1)`, the format string and a pointer to the arguments. The two functions right below it have the exact same shape: they take a format plus a variable number of arguments and forward everything to another function. So this looked like a whole family of stdio functions, but none of them is an import. Going back to the `msvcrt.dll` imports in PEStudio, the only ones from that family were `fprintf` and `vfprintf`.

Looking it up, I found out that MinGW-w64 ships its own implementations of the C stdio functions, which get linked into the binary instead of using the ones from `msvcrt.dll` (the source is in the [mingw-w64 CRT, stdio/](https://mingw.googlesource.com/mingw-w64/+/refs/heads/master/mingw-w64-crt/stdio/)). With the source at hand, telling them apart was a matter of comparing each wrapper with how it's used in the binary:

- `sub_14000fb30` is `printf`: every console message goes through it, and like mingw-w64's version it just hands a stream (`data_140010100(1)`), the format and the arguments to the function that does the actual formatting. MinGW-w64 uses its own version because the one in `msvcrt.dll` doesn't fully follow the C99 standard, so its [stdio.h](https://mingw.googlesource.com/mingw-w64/+/refs/heads/master/mingw-w64-headers/crt/stdio.h) redirects `printf` to `__mingw_printf` when C99 conformance is requested.
- `sub_14000fb90` is `sscanf`. In the first level, right after reading the serial, it's called with the user's input, `"%8x"` and a pointer to a variable, and its cross references show it's also used with `"%4x-%4x-%4x-%4x"`. mingw-w64's `sscanf` does exactly that, it passes the input string, the format and the arguments to `__mingw_vsscanf`. It's there for the same reason, the same `stdio.h` redirects `sscanf` to `__mingw_sscanf` so that input formats also follow C99 instead of `msvcrt.dll`'s behaviour.

![sscanf identified in Binary Ninja](img-016.webp){: w="1686" h="923" }

- `sub_14000fbe0` is `snprintf`: its cross references show it receiving a buffer, a size (`0x20`) and `"%08X"`, and mingw-w64's `snprintf` passes the buffer, its size, the format and the arguments to `__vsnprintf`. Same story as the other two: `stdio.h` redirects `snprintf` to `__mingw_snprintf` so that its output follows C99 instead of `msvcrt.dll`'s behaviour.

![snprintf identified in Binary Ninja](img-017.webp){: w="1686" h="923" }

Even after identifying these functions, I still didn't understand why `main` looked nested. Digging a bit, I found out that Binary Ninja has to be told that a function takes a variable number of arguments. In a stripped binary it can only guess how many arguments each call has, and with these functions it guessed wrong, so it ended up treating some calls as arguments of others. It's common enough that there's even a plugin, [binaryninja-printf](https://github.com/nshp/binaryninja-printf), whose whole job is annotating variadic printf-like functions, including the ones statically linked into a binary like here. With only three functions I did it by hand. I renamed them and changed their types with `y` ([Binary Ninja docs](https://docs.binary.ninja/guide/types/basictypes.html)) to `int32_t printf(char const* fmt, ...)` and so on, and the nesting was gone. Then I went through the four calls in `main` to confirm what each one was before naming them:

- `sub_140001877` prints *"LEVEL 1 / 4 - Difficulty: MEDIUM"*.
- `sub_140001ae5` doesn't print its title directly but it says *"using the same username from Level 1"* and asks for a XXXX-XXXX-XXXX-YYYY serial, so it's clearly level 2.
- `sub_140001fca` prints *"LEVEL 3 / 4 - Difficulty: HARD (login)"*.
- `sub_140002468` prints *"LEVEL 4 / 4 - Difficulty: VERY HARD (final)"*.
- `sub_14000262e` prints *"CHALLENGE COMPLETE"* and the completion code.

After renaming, `main` reads a lot better:

![main after renaming the stdio functions and the levels](img-018.webp){: w="1686" h="923" }

Something was still bugging me, though. The banner says the levels are chained, but in the decompiled `main` nothing checks what each level returns. It just calls all four and returns 0, so I decided to check the code in disassembly mode:

![Disassembly of main, checking the return value after each level](img-019.webp){: w="1090" h="1052" }

After each `call level_N` there's a `xor eax, 0x1` / `test al, al` / `je`: if a level returns anything other than 1, `main` jumps out with `eax = 1`. So the checks are there, the decompiler just simplified them away. Good to know before trusting the decompiler blindly in the levels.

I also took a quick look at the first call, `sub_140002900`, since it runs before any level. It walks a small table of function pointers and calls each one, which is just initialization code with nothing related to the challenge, so I named it `run_init_table` and moved on.

### The TLS callbacks

Back to the two TLS callbacks I wrote down in the basic analysis:

![The two TLS callbacks in Binary Ninja](img-020.webp){: w="1686" h="923" }

Both of them end up in `sub_140003400`, and Calltree shows it only goes a couple of calls deeper, to `free` and two small functions. I skimmed through them and nothing caught my attention. There are no anti-debugging APIs and nothing related to the levels, so whatever the binary does against debuggers, it doesn't happen before `main`.

### Where are the anti-debugging checks?

To find them I went the other way around. Starting from the imports. Selecting each entry in the import table shows where it's used:

![Import table entries and where each one is used](img-021.webp){: w="1686" h="923" }

Each anti-debugging API is called from exactly one small function. I named them after what they do:

![The anti-debugging helpers after renaming](img-022.webp){: w="1686" h="923" }

- `check_debugger`: just `IsDebuggerPresent()`.
- `check_remote_debugger`: `CheckRemoteDebuggerPresent()` on its own process.
- `check_hw_breakpoints`: gets the context of the current thread and returns 1 if any of Dr0-Dr3 is set.

![check_hw_breakpoints reading Dr0 to Dr3](img-023.webp){: w="1686" h="923" }

- `check_timing`: takes a `QueryPerformanceCounter` before and after a loop of `0x30d40` iterations, and returns 1 if it took more than 250 ms. A loop like that should take way less, so it should only trigger if someone is single-stepping (or has a breakpoint) inside it.

When any of them returns true, the level calls another small function that prints `"\n[!] %s\n"` with the message and calls `exit(1)`. I named it `die`. The start of `level_4` shows the whole pattern:

![Start of level_4 calling the checks and die](img-024.webp){: w="1686" h="923" }

With the cross references of each helper I put together which level uses which check:

| Level | `check_debugger` | `check_remote_debugger` | `check_hw_breakpoints` | `check_timing` |
| --- | --- | --- | --- | --- |
| 1 |  |  |  |  |
| 2 | YES |  |  |  |
| 3 | YES |  | YES | YES |
| 4 | YES | YES | YES | YES |

Level 1 has no checks at all, while level 4 runs all four before even printing its banner. So the debugger is going to get less and less welcome as I move forward.

### Other helpers

While going through the levels to name them, a few small functions kept showing up:

| Function | What it does | Name |
| --- | --- | --- |
| `sub_1400015bf` | `fgets` from `stdin` and cuts the string at the first `\r` or `\n` | `read_line` |
| `sub_140001555` | XORs every byte of a buffer with a key and copies it to a destination | `xor_decode` |
| `sub_140001480` | Binary Ninja already shows it as a ROLD, a rotate left | `rol32` |

Two of them are worth keeping in mind for later:

- `read_line` is how every level reads its input, always into a `0x80`-byte buffer. Since `fgets` reads one character less than the size, anything longer than 127 characters gets cut, and the string ends at the first newline. That's going to matter when looking for border cases.
- `rol32` isn't only used by `check_timing`. Its cross references show it in `sub_140001801` (which `level_1` calls with the username), in two of the functions under `level_3`, and in `sub_140002161`, the function `level_4` passes the password to. So rotations are going to show up a lot.

`xor_decode` explains the missing level 2 title. `level_2` calls it twice with the key `0x5a`, once to decode its banner (printed right after with `" %s\n"`) and once to decode the message it passes to `die` if a debugger is attached. That's why there was no *"LEVEL 2 / 4"* among the strings.

![level_2 decoding its strings with xor_decode](img-025.webp){: w="1686" h="923" }

With the key and the data right there, I didn't need to run anything to read them: I copied the bytes of both buffers from Binary Ninja and XORed them with `0x5a` in a small Python script.

```python
# level 2 strings, xor key 0x5a
title = bytes.fromhex("1b191b1e1f17037a081f770d13147a60607a163f2c3f367a68")
msg = bytes.fromhex("1e3f382f3d3d33343d7a3f342c33283534373f342e7a3e3f2e3f392e3f3e")

for buf in (title, msg):
    print(bytes(b ^ 0x5a for b in buf).decode())
```
{: file="level2_strings.py" }

```text
ACADEMY RE-WIN :: Level 2
Debugging environment detected
```

The level 2 title is *"ACADEMY RE-WIN :: Level 2"*, and the message `level_2` dies with when it finds a debugger is *"Debugging environment detected"*.

### The map

With everything renamed, this is the final picture I have before starting with the levels, the Calltree graph of `main`:

![Calltree graph of main with everything renamed](img-027.webp){: w="1297" h="876" }

The four levels hang from `main`, and the shared helpers sit below them with arrows coming from several levels. Time to start with level 1.

## Level 1

### First look

Level 1 is fairly short:

![Decompiled level_1](img-028.webp){: w="823" h="764" }

After the banner, `level_1` loops forever asking for two things, both read with `read_line` into `0x80`-byte buffers: a username into `_Source` and a serial, `"8 hex digits"`, into `var_118`.

![level_1 reading the username and the serial](img-029.webp){: w="636" h="98" }

Right after that comes the actual check:

![The serial check in level_1](img-030.webp){: w="767" h="322" }

It calls `sub_140001801` with the username, takes the `strlen` of the serial and, only if it's exactly 8 characters long, parses it with `sscanf(&var_118, "%8x", &var_11c)`. The loop only breaks when all three conditions hold.

- the serial is 8 characters long (`rax_2 == 8`),
- sscanf managed to read a hex number from it (`rax_3 == 1`),
- and that value is equal to what `sub_140001801` returned for the username (`rax_1 == var_11c`).

Otherwise it prints *"Invalid serial. Try again."* and asks for both again. There's no attempt counter and no anti-debugging here, which matches the table I created in the previous section.

![level_1 copying the username and saving the hash](img-031.webp){: w="669" h="98" }

When the check passes, it copies the username into `arg1`, a buffer it gets from `main`, stores the value from `sub_140001801` in the global `data_140015040`, and returns 1. The copy explains why level 2 said it was *"using the same username from Level 1"*.

So, the serial is just a number computed from the username, written in hex. Everything depends on `sub_140001801`.

### The username hash

![sub_140001801, the username hash](img-032.webp){: w="823" h="220" }

It's short and the decompiler output reads almost like plain C. It walks the username and, for each character, adds its value (cast to `uint8_t`) multiplied by its position starting at 1. Then it XORs the 32-bit sum with `0xc001c0de` (cool code?) and rotates the result left by 5 with `rol32`, the helper I had already found in the previous section:

`hash = rol32( sum(name[i] * (i + 1)) ^ 0xc001c0de, 5 )`

Since it's a hash of the username, I renamed it `username_hash`.

Two details I wrote down for the border cases:

- Nothing checks the username: there's no minimum length and no allowed character set. Even an empty username should work: the sum stays at 0, `0 ^ 0xc001c0de` is just `0xc001c0de`, and rotating it left by 5 gives `0x00381BD8`, so the serial would be `00381BD8`.
- Each character is read as an unsigned byte (`uint8_t`), a number from 0 to 255. For plain letters and digits that's just their ASCII code, but I'm not sure what would happen with letters that aren't plain ASCII, like á or ñ, so that's something to test at runtime.

With the hash understood I renamed the rest of the variables in `level_1`:

![level_1 with its variables renamed](img-033.webp){: w="823" h="743" }

One thing was still loose: right before returning, `level_1` saves the hash in a global variable, `data_140015040`, and nothing in level 1 reads it back.

![Cross references of data_140015040](img-034.webp){: w="731" h="153" }

Cross references show that only two places touch it: `level_1` writes it and `print_completion_code` reads it with `snprintf(&buf, 0x20, "%08X", ...)`. So, the level 1 hash ends up as part of the completion code printed at the end. I renamed the global `level1_hash`.

At this point level 1 boils down to computing `username_hash` for any username and writing it as 8 hex digits, so the next step is a keygen.

### Keygen

The keygen is just `username_hash` rewritten in Python. I replicated the same sum, the same XOR and the same rotation. I wrote it as a small console program, `keygen1.py`, that asks for the username and prints the serial:

```python
def rol32(x, n):
    return ((x << n) | (x >> (32 - n))) & 0xffffffff

def lvl1(name):
    s = 0
    for i, c in enumerate(name):
        s += c * (i + 1)
    return rol32(s ^ 0xc001c0de, 5)

name = input("Username: ")
print("Serial: %08X" % lvl1(name.encode()))
input("Press Enter to exit")
```
{: file="keygen1.py" }

Running it with my name:

```text
Username: Bruno
Serial: 0038D6D8
Press Enter to exit
```

To test it against the binary I first tried a random serial, to see the error, and then the one from the keygen:

![Level 1 rejecting 12345678 and accepting 0038D6D8](img-037.webp){: w="965" h="380" }

`12345678` is rejected and the loop asks again, while `0038D6D8` passes and the binary moves on to level 2, which already shows *"Bruno"* as the username it took from level 1.

### Border cases

With a working keygen I went back to the notes I had taken while reading the code.

#### Empty username

Nothing stops it, so I just pressed Enter at the username prompt and used `00381BD8`, the serial I had worked out by hand for the empty name (the keygen gives the same if you just press Enter):

![Level 1 passing with an empty username and 00381BD8](img-038.webp){: w="983" h="166" }

It passes, and level 2 then shows an empty username between the parentheses.

#### How strict is `"%8x"`?

The length check only counts characters, and the number itself is read by sscanf. Reading the [scanf reference](https://en.cppreference.com/w/c/io/fscanf), `%x` skips spaces at the start, accepts a `0x` prefix and stops reading at the first character that isn't hex, so I tried it. My serial, `0038D6D8`, starts with two zeros, so I can drop them and use those two characters for something else. Each of these is 8 characters long and passed:

- `0x38d6d8`, with the `0x` prefix (and lowercase):

![Level 1 accepting 0x38d6d8](img-039.webp){: w="983" h="70" }

- `38D6D8`, with two spaces in front:

![Level 1 accepting the serial with two leading spaces](img-040.webp){: w="983" h="70" }

- `38D6D8zz`, with garbage at the end:

![Level 1 accepting 38D6D8zz](img-041.webp){: w="983" h="70" }

So, for any hash that starts with 00 there are several valid serials besides the "official" one. With a hash that uses all 8 digits only the plain form fits in 8 characters (plus upper and lowercase variations).

#### Usernames longer than 127 characters

Here I went back to `read_line`:

![Decompiled read_line](img-042.webp){: w="823" h="352" }

It calls `fgets` with the size it receives (`0x80`) and then cuts the string at the first `\r` or `\n`. If the line is longer than 127 characters, `fgets` stops there and the rest of the line stays in the input buffer, and nothing clears it. So the next `read_line`, the one for the serial, gets the leftovers without waiting for me. To test it I ran the keygen with 127 As, which gives `0139E3D8`, and pasted those 127 As followed directly by that serial, all on one line (`AAAA...AAAA0139E3D8`). The first 127 characters become the username and the remaining 8 are what `read_line` returns for the serial:

![127 As followed by the serial on one line, and level 1 passes](img-043.webp){: w="983" h="183" }

The binary prints the serial prompt but doesn't wait, because the serial was already there, and the level passes. Level 2 also shows the 127 As as the username. The same thing would happen with any username longer than 127 characters, the extra characters become the serial, which most of the time just means an *"Invalid serial"*.

#### Accented letters

For `Brunó` the keygen gave me `00399C18`:

```text
Username: Brunó
Serial: 00399C18
Press Enter to exit
```

and the binary rejected it. My keygen was encoding the name with Python's default, UTF-8, where ó takes two bytes. The console used for these tests was on code page 850 ([chcp docs](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/chcp)), where ó is a single byte, `0xA2`. Encoding the name with cp850 gives `0038F738`, which passed:

![Brunó rejected with the UTF-8 serial and accepted with the cp850 one](img-045.webp){: w="983" h="147" }

So, the serial for a name with accents depends on the code page of the console it's typed in. My first fix used [GetOEMCP](https://learn.microsoft.com/en-us/windows/win32/api/winnls/nf-winnls-getoemcp), called from Python with ctypes, to avoid hardcoding cp850. That gave the right result here, but nearing the end of the challenge I realized by chance I had mixed up the system's OEM code page and the console's input code page. I hadn't realized they can be different, for example after changing the console code page with `chcp`.

When I revisited this, the documentation for [GetConsoleCP](https://learn.microsoft.com/en-us/windows/console/getconsolecp) pointed me to the input code page of the console attached to the calling process. I switched all four keygens to that. I run each keygen and the challenge in the same console, without changing its code page between them, so the keygen encodes the input with the code page the challenge receives. This is the corrected version:

```python
import ctypes

def rol32(x, n):
    return ((x << n) | (x >> (32 - n))) & 0xffffffff

def lvl1(name):
    s = 0
    for i, c in enumerate(name):
        s += c * (i + 1)
    return rol32(s ^ 0xc001c0de, 5)

# input code page of the current console
cp = "cp%d" % ctypes.windll.kernel32.GetConsoleCP()

name = input("Username: ")
print("Serial: %08X" % lvl1(name.encode(cp)))
input("Press Enter to exit")
```
{: file="keygen1.py" }

With the console still on cp850, the serial for `Brunó` stays the same:

```text
Username: Brunó
Serial: 0038F738
Press Enter to exit
```

Level 1 is done. Next up is level 2, which reuses this username.

## Level 2

### First look

The beginning of `level_2` is the part I had already seen while looking for its missing title:

![Start of level_2](img-048.webp){: w="1216" h="345" }

It decodes its two strings with `xor_decode` and then calls `check_debugger`. If a debugger is attached, it dies with *"Debugging environment detected"*, so this is the first level that doesn't want to run under a debugger, although for now it only uses `IsDebuggerPresent`. Then it prints the decoded title and the username, which comes in `arg1` and it's the same buffer main passed to `level_1`, where level 1 copied the username at the end.

Then comes the loop:

![Serial loop of level_2](img-049.webp){: w="1216" h="535" }

- It reads the serial with `read_line`, again into a `0x80`-byte buffer.
- `sscanf` parses it with `"%4x-%4x-%4x-%4x"` and has to return 4, so the serial is four groups of up to 4 hex digits separated by dashes.
- Each value is cut to 16 bits (`int16_t`) and stored in four locals that sit next to each other: `var_120`, `var_11e_1`, `var_11c_1` and `var_11a_1`. That's an array of four 16-bit words, and its address is what goes to `sub_140001a34`, together with the username.
- If `sub_140001a34` returns non-zero, the level passes, the first word is saved in the global `data_140015044` and it returns 1. Otherwise, it prints *"Invalid serial. Try again."* and asks only for the serial again, since the username is fixed from level 1.

### The serial check

![sub_140001a34 comparing the four groups](img-050.webp){: w="1216" h="251" }

`sub_140001a34` calls `sub_1400019b1` three times with the username and a different 16-bit constant each time: `0x1234`, `0xabcd` and `0x7777`. The serial is valid when the first three groups are equal to those three results and the fourth one is the XOR of the three. That explains the `YYYY` in the level prompt since the last group is a checksum of the other three. I renamed it `check_serial2`.

### The 16-bit hash

![sub_1400019b1, the 16-bit hash](img-051.webp){: w="1216" h="235" }

`sub_1400019b1` is a small hash. It starts from the constant it receives and, for each character of the username, mixes the character in with an XOR, multiplies by `0x9e37` and passes the result through `sub_14000149c`. Everything works on 16-bit values.

`sub_14000149c` sits right below `rol32` and looks a lot like it, it's also a rotation, but on 16 bits.

![sub_14000149c next to rol32](img-052.webp){: w="1216" h="274" }

One line of the decompiled code confused me: `arg1` is an `int16_t` (a signed 16-bit number), and before shifting it the code casts it to `uint32_t`. I wasn't sure what that cast does with a negative value. Does it pad the new 16 bits with zeros, or with ones to keep the sign? If it pads with ones, those ones would get shifted into the result and it wouldn't be a real rotation anymore. I looked at the disassembly:

![Disassembly of the rotation, using movzx](img-053.webp){: w="1216" h="444" }

The instruction it uses is `movzx`, "move with zero extension", so the value is padded with zeros, and this is a plain 16-bit rotation. I renamed `sub_14000149c` to `rol16`, and `sub_1400019b1` to `name_hash16`. After also renaming its variables, `name_hash16` reads like this, where `start` is the constant it receives:

![name_hash16 after renaming](img-054.webp){: w="1216" h="213" }

So the serial is the username hashed three times, starting from `0x1234`, `0xabcd` and `0x7777`, plus a fourth group with the three hashes XORed together.

Like in level 1, the level saves something in a global before returning, so I checked who uses `data_140015044`:

![Cross references of data_140015044](img-055.webp){: w="954" h="154" }

Again only two places. `level_2` writes the first word of the serial, and `print_completion_code` reads it with `snprintf(&buf, 0x20, "%04X", ...)`. So the first group of the level 2 serial is also used for the completion code. I renamed the global `level2_word`.

With the functions and the global named, I also renamed the variables of `level_2` and turned the four 16-bit locals into an array, `int16_t words[4]`, by changing the type of the first one with `y`. This is how the serial loop reads after that:

![level_2 serial loop with int16_t words[4]](img-056.webp){: w="965" h="649" }

### Keygen

The keygen follows the same idea as in level 1. A replica of `name_hash16` rewritten in Python (with the 3 constants) plus the XOR of the three results. I kept the code page detection from level 1, since level 2 hashes the same username:

```python
import ctypes

def rol16(x, n):
    return ((x << n) | (x >> (16 - n))) & 0xffff

def name_hash16(name, start):
    h = start
    for c in name:
        h = rol16(((h ^ c) * 0x9e37) & 0xffff, 3)
    return h

# console input code page, like in level 1
cp = "cp%d" % ctypes.windll.kernel32.GetConsoleCP()

name = input("Username: ").encode(cp)
a = name_hash16(name, 0x1234)
b = name_hash16(name, 0xabcd)
c = name_hash16(name, 0x7777)
print("Serial: %04X-%04X-%04X-%04X" % (a, b, c, a ^ b ^ c))
input("Press Enter to exit")
```
{: file="keygen2.py" }

Running it with my name:

```text
Username: Bruno
Serial: D31B-84B1-76D4-217E
Press Enter to exit
```

And against the binary, first with a random serial and then with the one from the keygen:

![Level 2 rejecting a random serial and accepting the keygen's](img-059.webp){: w="986" h="178" }

### Border cases

#### Empty username

Level 1 accepts an empty username and level 2 just reuses it. With no characters, `name_hash16` never enters its loop and returns the constant it started from, so the serial is just the three constants and their XOR:

```text
Username:
Serial: 1234-ABCD-7777-CE8E
Press Enter to exit
```

![Level 2 accepting 1234-ABCD-7777-CE8E with an empty username](img-061.webp){: w="986" h="113" }

It passes, so for an empty username the level 2 serial is sitting right there in `check_serial2`.

#### How strict is `"%4x-%4x-%4x-%4x"`?

Like in level 1, `%x` skips the spaces before each number and doesn't care about uppercase or lowercase. And this time nothing checks the length of the serial, so once `sscanf` has read its four numbers, whatever comes after them is ignored:

- In lowercase and with spaces after the dashes:

![Level 2 accepting the serial in lowercase and with spaces](img-062.webp){: w="986" h="50" }

- With some text at the end:

![Level 2 accepting the serial with text at the end](img-063.webp){: w="986" h="54" }

Level 2 is done. Next up is level 3, which asks for a new username and a password.

## Level 3

### First look

![Start of level_3 with three anti-debugging checks](img-064.webp){: w="1282" h="516" }

Level 3 starts by calling three of the anti-debugging helpers I had found while getting oriented, one after the other. I'm not using a debugger for now, but it's something to keep in mind if I need one later.

After the checks it copies an 8-byte global, `data_140010000`, into a local and calls `sub_140001c74`. Then it prints the banner and goes into the login loop:

![Login loop of level_3](img-065.webp){: w="1282" h="498" }

- It asks for a username, and it has to be at least 5 characters long (`strlen > 4`), otherwise it prints *"Invalid username (minimum length 5)."*.
- Then it asks for a password, again with `read_line` and `0x80`.
- `sub_140001f2b` receives the username and the 8-byte local, and writes something into a `0x20-byte` buffer, `_Str2`. If the password is equal to that buffer (`strcmp`), the loop ends.
- After *"Level 3 PASSED"*, it copies 16 characters of the password into another global, `data_140015050`, and returns 1.

So the password is built from the username and those 8 bytes. Unlike the first two levels, this time the comparison is a `strcmp` against a string, so there's no `sscanf` to be lenient with the input, and the password has to match exactly.

### The 8-byte key

Before looking at the login, I wanted to know what `sub_140001c74` does with those 8 bytes, since it runs before anything else:

![sub_140001c74 before renaming](img-066.webp){: w="1282" h="669" }

What stood out first were the loops up to `0xff`. The first one just fills a 256-byte table with the numbers 0 to 255, in order. The second one goes over that table again and swaps its entries around, and to decide what to swap it uses the bytes of the first argument, the `"RE-WIN!!%08X%08X"` string. The last loop runs once per byte of the data, 8 times here, and XORs each byte with a value taken from the table.

The table of 256 bytes filled with 0 to 255 and then shuffled with a key, coincidentally, I had run into something like this in a small CTF at DEF CON this year. Looking it up confirmed it's RC4, and the pseudocode on [Wikipedia](https://en.wikipedia.org/wiki/RC4) has the same three loops.

After renaming the arguments and variables, the function reads like this:

![rc4 after renaming](img-067.webp){: w="1282" h="630" }

One small detail, the position in the key is `i_1 % key_len`, and `key_len` is the `8` that `level_3` passes. So the index goes back to the start after the 8th character, and out of the whole "`RE-WIN!!%08X%08X`" string only `RE-WIN!!` is ever used. The `%08X%08X` part is never read, at least not here.

These are the 8 encrypted bytes:

![The 8 encrypted bytes shown as a 64-bit value](img-068.webp){: w="653" h="45" }

Binary Ninja shows them as a single 64-bit number, `0x5c044c603c508d4a`. Since RC4 works on bytes, I switched to the hex view to see them in the order they are in memory:

![The 8 encrypted bytes in the hex view](img-069.webp){: w="1143" h="24" }

Instead of decrypting them by hand, I wanted to see the program decrypt them, so this was the moment to open x64dbg, keeping in mind that the level starts with the anti-debugging checks.

I started the challenge as usual, passed level 1 and attached x64dbg while level 2 was waiting for its serial. My first idea was to go straight to the call to `rc4` with the address from Binary Ninja, `0x14000204f`, but x64dbg didn't like it:

![x64dbg rejecting the Binary Ninja address](img-070.webp){: w="542" h="123" }

The Symbols tab showed that x64dbg had the program loaded at `0x7FF77E120000`, not at `0x140000000` like Binary Ninja, so the addresses didn't match. Instead of doing the math, I looked for the `RE-WIN!!` string.

![The RE-WIN!! string in x64dbg](img-071.webp){: w="1159" h="54" }

Double-clicking it took me to `level_3`, with the call to `rc4` two instructions below, so I put a breakpoint there. I also followed one of the calls to `die` from `level_3` and put another breakpoint at its start, so that if a check caught me I would at least see which one. After entering the level 2 serial, one did:

![Breakpoint hit in die after IsDebuggerPresent](img-072.webp){: w="1118" h="134" }

The message says it was `IsDebuggerPresent`. According to the [x64dbg docs](https://help.x64dbg.com/en/latest/commands/misc/HideDebugger.html), its `hide` command modifies the process so that `IsDebuggerPresent()` returns false. So I restarted the challenge, attached again and ran `hide` before entering the level 2 serial:

![Running hide in x64dbg](img-073.webp){: w="931" h="149" }

This time the three checks passed and the program stopped at the call to `rc4`. The third argument, `r8`, is the address of the 8 bytes, so I followed it in the dump. There were the encrypted bytes, in the same order as in the hex view:

![Encrypted bytes in the x64dbg dump before rc4](img-074.webp){: w="1004" h="75" }

I stepped over the call and the same 8 bytes changed:

![The same bytes decrypted after rc4](img-075.webp){: w="1004" h="75" }

The decrypted key is `13 37 c0 ff ee 42 99 01` (elite coffee?).

### Building the password

Now the function that builds the password, `sub_140001f2b`. It's short:

![sub_140001f2b, which builds the password](img-076.webp){: w="1282" h="214" }

It does three things:

1. It calls `sub_140001e9f` with the username and keeps the result, a 32-bit number.
1. For each of the 8 decrypted bytes, it calls `sub_140001ee9` with that number and the byte, and keeps the new result.
1. It writes two 32-bit numbers in hex into the buffer with `"%08X%08X"`: the final result and a variation of it `rol32(result ^ 0xdec0ded1, 11)`. That's why the password is 16 hex characters long.

![sub_140001e9f, djb2 with XOR](img-077.webp){: w="1282" h="271" }

`sub_140001e9f` walks the username one character at a time. It starts from `0x1505` and, for each character, multiplies by `0x21` and XORs the character in. A search for `0x1505` led me to find out that it's `djb2`, a variant that uses XOR instead of an addition. I renamed it `djb2`.

![sub_140001ee9, which mixes in each key byte](img-078.webp){: w="1282" h="116" }

`sub_140001ee9` is a single line, but it does several things to the number: it XORs the byte in, rotates it left by 7, subtracts `0x61c88647` and XORs it with itself shifted 13 bits to the right. I didn't find a matching algorithm when looking online even though the constant `0x61c88647` is mentioned on many hashing algorithms. I renamed it `mix_key_byte`, and `sub_140001f2b` became `make_password3`.

Like in the previous levels, I also checked who uses the global the password is copied to:

![Cross references of data_140015050](img-079.webp){: w="954" h="175" }

Besides `level_3`, only `print_completion_code` uses it, passing it to `sub_1400025c6`. So the level 3 password is also part of the completion code. I renamed it `level3_password`, and the encrypted bytes `level3_key_enc`. With everything renamed, `level_3` reads like this:

![level_3 after renaming](img-080.webp){: w="1282" h="612" }

x64dbg was still attached and stopped just after the call to `rc4`, so I used it to get a correct password to test my keygen against later. I scrolled down from the call to `rc4` to the call to `strcmp`, put a breakpoint there, let the program run and entered `Bruno` with a random password:

![Registers at strcmp, with the expected password in rdx](img-081.webp){: w="1494" h="67" }

`rcx` has the password I typed and `rdx` the one the level expects for Bruno: `AFB0685485B42B8B`.

To see the whole level at a glance, this is its Calltree graph with everything renamed:

![Calltree graph of level_3](img-082.webp){: w="1249" h="764" }

Everything that is specific to this level hangs from `rc4` and `make_password3`, the rest are the same helpers the other levels use.

### Level 3, step by step

1. Before anything else, it looks for a debugger in three different ways and exits if it finds one.
1. It decrypts 8 bytes stored in the binary with `RC4`, using `RE-WIN!!` as the key. Those bytes are always the same, so they work like a fixed secret that only shows up while the program runs.
1. It asks for a username of at least 5 characters, and then for a password.
1. It builds the expected password from the username, in three steps:
   - `djb2` goes over the username one character at a time and turns it into a single 32-bit number.
   - `mix_key_byte` takes that number and mixes the 8 decrypted bytes into it, one by one. That way the result depends on both the username and the key.
   - The final number is written in hex twice, 8 characters each time: first as it is, and then XORed with `0xdec0ded1` and rotated left by 11 bits. Together they make the 16 characters of the password.
1. If the password I typed is exactly that string, the level passes and the password is saved for the completion code.

The good news is that nothing depends on the machine or on the time. The only input is the username, so a keygen just has to repeat these steps in Python.

### Keygen

This keygen has a few more pieces than the previous ones, but each of them is a direct copy of what I saw in Binary Ninja. `rc4` with its three loops, `djb2`, `mix_key_byte` and the two `%08X` at the end of `make_password3`. The 8 encrypted bytes go in just as they appear in the hex view, and I also made it print the decrypted key, to compare it with what I saw in x64dbg. I kept the code page detection from the previous levels, because the username is hashed byte by byte here too:

```python
import ctypes

def rol32(x, n):
    return ((x << n) | (x >> (32 - n))) & 0xffffffff

def rc4(key, data):
    S = list(range(256))
    j = 0
    for i in range(256):
        j = (j + S[i] + key[i % len(key)]) & 0xff
        S[i], S[j] = S[j], S[i]
    out = []
    i = j = 0
    for b in data:
        i = (i + 1) & 0xff
        j = (j + S[i]) & 0xff
        S[i], S[j] = S[j], S[i]
        out.append(b ^ S[(S[i] + S[j]) & 0xff])
    return bytes(out)

def djb2(name):
    h = 0x1505
    for c in name:
        h = (c ^ (h * 0x21)) & 0xffffffff
    return h

def mix_key_byte(h, b):
    t = (rol32(h ^ b, 7) - 0x61c88647) & 0xffffffff
    return t ^ (t >> 13)

# level3_key_enc, decrypted with RE-WIN!!
key = rc4(b"RE-WIN!!", bytes.fromhex("4a8d503c604c045c"))
print("Key:", key.hex())

cp = "cp%d" % ctypes.windll.kernel32.GetConsoleCP()

name = input("Username: ").encode(cp)
h = djb2(name)
for b in key:
    h = mix_key_byte(h, b)
print("Password: %08X%08X" % (h, rol32(h ^ 0xdec0ded1, 11)))
input("Press Enter to exit")
```
{: file="keygen3.py" }

Running it with my name:

```text
Key: 1337c0ffee429901
Username: Bruno
Password: AFB0685485B42B8B
Press Enter to exit
```

Both the key and the password match what x64dbg showed me.

And against the binary, first with a random password and then with the one from the keygen:

![Level 3 rejecting a random password and accepting the keygen's](img-085.webp){: w="736" h="199" }

### Border cases

Unlike the first two levels, there's no `sscanf` here, so the password has much less room to play with. Still, I tried a few things:

#### Short username

The 5 character minimum is checked before asking for the password, so with 4 characters the level doesn't even get there:

![Level 3 rejecting a 4-character username](img-086.webp){: w="736" h="52" }

#### Lowercase password

In levels 1 and 2 `%x` didn't care about uppercase or lowercase, but here the password is compared with `strcmp`, character by character. The expected password is printed in uppercase with `%08X`, so the same password in lowercase fails:

![Level 3 rejecting the password in lowercase](img-087.webp){: w="736" h="68" }

#### A username made only of spaces

The only check on the username is its length, and `read_line` only cuts the line break, so five spaces are a valid username. I selected them in the console so they can be seen:

![Keygen output for a username of five spaces, selected](img-088.webp){: w="982" h="107" }

![Level 3 accepting a username of five spaces](img-089.webp){: w="736" h="68" }

Level 3 is done. Next up is level 4, the final one, where the password is checked by an internal VM.

## Level 4

### First look

![Start of level_4 with all four checks](img-090.webp){: w="1282" h="557" }

The last level starts with all four anti-debugging checks, then it prints the banner, this time with a warning and a tip, and goes into its loop:

![Password loop of level_4](img-091.webp){: w="1282" h="550" }

- It asks for a password, with `read_line` and `0x80` like the other levels.
- It passes the password and the address of a local, `var_89`, to `sub_140002161`. The level only passes if that function returns something other than 0 and `var_89` also ends up as something other than 0.
- After *"LEVEL 4 PASSED"*, it copies up to `0x3f` characters of the password into another global, `data_140015080`, and returns 1.

The banner says the password is validated by an internal VM, and the tip asks to understand the bytecode. I had a rough idea of what that meant, but before going further I looked up how a VM works in a crackme. The clearest explanation I found was [Cracking a virtualized binary with Angr](https://noh.am/en/posts/vm/), by Noham. The idea is that the real check isn't written as normal x86 code, instead it's turned into bytecode for an instruction set the authors made up, and the program carries a small interpreter that runs it. That interpreter is a loop that reads an opcode, looks it up in a table, and jumps to the piece of code (the handler) for that instruction, keeping the state of the VM in a few variables of its own. The post solves its binary with angr, but it also describes a manual way. Go through every handler, understand the format of the bytecode and write a disassembler for it. I followed the same approach to understand the handlers and the bytecode format, then decoded the bytecode by hand.

There's no username this time and no `strcmp` against an expected password. Everything is decided inside `sub_140002161`, which, going by the banner, must be the VM.

### Inside `sub_140002161`

![sub_140002161 before renaming](img-092.webp){: w="1282" h="557" }

This function is longer than the others, so I went through it in parts, from top to bottom.

**First, a loop that fills a buffer:**

![Loop that decrypts the bytecode](img-093.webp){: w="1000" h="153" }

It goes from 0 to `0x88`, so it runs `0x89` (137) times. Each time, it takes one byte from `data_140011380`, XORs it with `var_9`, and stores the result in the same position of a local buffer, `var_b8`. `var_9` starts at `0xa5` and goes up by one after every byte (`0xa5`, `0xa6`, `0xa7`...), so every byte is XORed with a different value. In other words, `data_140011380` holds 137 encrypted bytes, and after this loop `var_b8` has them decrypted.

**Then, a check:**

![CRC check against 0x69f27969](img-094.webp){: w="1000" h="39" }

It calls `sub_1400014d4` with `data_140011380` and `0x89`, the same bytes and the same length, and if the result isn't `0x69f27969` the function returns 0 right there. I didn't know yet what `sub_1400014d4` computes, but it looked like a way to make sure those bytes are the expected ones.

**Then, a few variables start at 0:**

![Variables initialized to 0](img-095.webp){: w="1000" h="77" }

Two locals, `var_14_1` and `var_18_1`, are set to 0. The length of the password is saved in `rax_10`, and a counter, `var_20_1`, also starts at 0. I didn't know what the first two were for yet, but that would come from seeing how they're used.

**Finally, the main loop:**

![Main loop of the VM with the switch](img-096.webp){: w="1000" h="172" }

It keeps going while `var_20_1` isn't past `0x88`, the last position of `var_b8`. In each round it reads the byte of `var_b8` at the position of `var_20_1`, moves `var_20_1` one position forward, and uses the byte it just read to pick a case in a switch.

That last part is exactly the loop the post described, first read an opcode, pick its handler through the switch, run it, repeat. So `var_b8` should be the bytecode the banner talks about. These are the bytes it comes from:

![Encrypted bytecode in data_140011380](img-097.webp){: w="1282" h="114" }

On their own they don't say much, since what's stored in the file is the encrypted version. I could decrypt them by hand with `var_9`, but this time I wanted to see the program decrypt them itself, so it was the moment to bring back x64dbg.

`level_4` starts with the anti-debugging checks, four of them this time, including the new `check_remote_debugger`. Looking at `level_4` again, I saw that they run at the very start of the function, before the password loop.

![level_4 graph, with the checks before the password loop](img-098.webp){: w="1591" h="428" }

This meant that instead of fighting the checks, I could simply arrive after them. I ran the challenge normally without any debugger, passed levels 1 to 3 with my keygens, and only attached x64dbg when level 4 was already showing `Password:`

![The challenge waiting at the level 4 password prompt](img-099.webp){: w="736" h="225" }

![x64dbg attached, with the process still running](img-100.webp){: w="996" h="193" }

The process kept running after the attach, with no `die` anywhere. By then the four checks had already run with nothing attached, so they had nothing to find, and this time I didn't even need `hide`.

From level 3 I knew x64dbg loads the program at `0x7FF77E120000`, so I went to the instruction right after the decryption loop (`0x1400021b3` in Binary Ninja, `0x7FF77E1221B3` in x64dbg), put a breakpoint there and typed `test` as the password. This is the same instruction in both tools, the `mov edx, 0x89` that comes right after the loop's `jle`:

![mov edx, 0x89 in Binary Ninja](img-101.webp){: w="794" h="78" }

![The same instruction in x64dbg](img-102.webp){: w="1118" h="34" }

When it stopped, `var_b8` was already decrypted, and since the disassembly accesses it as `rbp-0xB0`, I opened that address in the dump:

![The decrypted bytecode in the x64dbg dump](img-103.webp){: w="996" h="123" }

There they are, the 137 bytes decrypted. I'll come back to them once I know what each instruction does.

**Then, the check from step 2:**

![sub_1400014d4, CRC32 with 0xedb88320](img-104.webp){: w="1282" h="287" }

`sub_1400014d4` goes over the bytes it gets and, for each one, does 8 rounds of shifts and XORs with `0xedb88320`. A search for that constant in Google outright told me it's the polynomial of CRC32, so I renamed it `crc32`.

So what is this check for?

1. Before running anything, `sub_140002161` computes the CRC of `data_140011380`, the encrypted bytecode exactly as it's stored in the file.
1. It compares the result with a fixed value written in the code, `0x69f27969`.
1. If they don't match, it returns 0 before running a single instruction, and the level fails.

That protects the bytecode against changes. Someone could think of editing those bytes in the exe so that the VM accepts any password, but that would change the CRC, and the VM wouldn't even start. It doesn't affect me, since I decided not to patch the binary anyway, but it confirms that the way through this level is to understand the bytecode, just like the tip says.

Since I was stopped right before that call, I stepped over it with F8 to see the result:

![eax equal to 0x69F27969 after the call](img-105.webp){: w="1494" h="157" }

`eax` is exactly `0x69F27969`, so the check passes and the VM goes on.

With that, I renamed `var_b8` to `code`, `data_140011380` to `code_enc` and `var_9` to `key`.

### The instructions

Next were the variables of the loop. I started with the two that were easier to figure out:

- `var_20_1` starts at 0, the loop stops when it goes past `0x88` (the last position of `code`), and in each round it's used to read the next byte of `code` and then moves one position forward. Skimming the cases, many of them move it forward a few more positions (`+= 4`, `+= 2`), and two of them give it a completely new value. So it always points to the next instruction to run, which is typically what a CPU calls the program counter. I renamed it `pc`.
- `rax_10` is the result of `strlen(arg1)`, and `arg1` is the password that `level_4` read with `read_line`, so it holds the length of the password. I renamed it `pass_len`.

To understand `var_14_1`, I clicked on it, and the Cross References panel listed every place that uses it:

![Cross references of var_14_1](img-106.webp){: w="557" h="250" }

Looking at the list, almost every line puts a new value into `var_14_1`. It gets a number (`= rax_18`), something is added to it (`+=`), it's XORed (`^=`), rotated (`rol32`) or multiplied (`*=`), or it gets the length of the password. Only one line, the `if`, just reads it to compare it with something. So, the instructions don't work with several different variables, they all keep updating this same one, so it basically works like a CPU register. I called it `acc`.

`var_18_1` was simpler. It has only four references: it starts at 0, `case 6` writes the result of a comparison into it, and `case 7` reads it to decide something. A value that only remembers whether a comparison was true is a flag, so it became `flag`.

The arguments were easier. `arg1` is what `level_4` read with `read_line`, so it became `password`. `arg2` only ever gets a 1 or a 0, and `level_4` needs it to be 1 to pass, so I called it `accepted`. The function itself became `vm_run`:

![vm_run after renaming](img-107.webp){: w="1282" h="557" }

Then came the `switch`. This is all of it after renaming, in three parts:

![The VM switch, part 1 of 3](img-108.webp){: w="1282" h="732" }

![The VM switch, part 2 of 3](img-109.webp){: w="1282" h="693" }

![The VM switch, part 3 of 3](img-110.webp){: w="1282" h="732" }

It's a lot of code at once, so I read it one case at a time. For each one, I looked at what it reads from the bytecode, what it does with `acc`, and how far it moves `pc`, and renamed its temporary variables as I went.

`case 1:`

![VM handler for case 1](img-111.webp){: w="900" h="133" }

When `case 1` starts, the main loop has already read the `opcode` and moved `pc` one position forward, so `pc` points to the byte right after the `opcode`. Then case 1:

1. Reads the 4 bytes starting at `pc` as one 32-bit number. That's what `*(uint32_t*)(pc + &code)` means, and Binary Ninja had already named the result `acc_1`.
1. Moves `pc` 4 positions forward.
1. Puts that number in `acc`.

Step 2 is what revealed that those 4 bytes aren't instructions but data that belongs to this one, and the next instruction starts right after them, so an instruction can take more than one byte. First comes the opcode, which says what to do, and then its argument, the value it works with.

`case 2:`

![VM handler for case 2](img-112.webp){: w="900" h="285" }

This one has a 1-byte argument. Step by step, case 2:

1. Reads the byte at `pc` and moves `pc` one position forward.
1. Uses that byte as a position inside the password. Since it's an index into the password, I called it `idx`.
1. Checks that `idx` isn't past the end of the password (`idx >= pass_len`). If it is, the result is 0. If not, the result is the character of the password at that position. Since it holds a character, I called it `ch`.
1. Adds `ch` to `acc`.

This is the first case that looks at the password. With it the bytecode can pick any character of the password and add it to `acc`.

`case 3:`

![VM handler for case 3](img-113.webp){: w="900" h="133" }

The same as case 1, but instead of replacing `acc` with the 4-byte number, it XORs it in. I called that number `imm`, for immediate.

`case 4:`

![VM handler for case 4](img-114.webp){: w="900" h="133" }

It reads one byte and rotates `acc` to the left by that many bits, with the same `rol32` from the previous levels.

`case 5:`

![VM handler for case 5](img-115.webp){: w="900" h="96" }

No argument at all, it just puts the length of the password in `acc`.

`case 6:`

![VM handler for case 6](img-116.webp){: w="900" h="210" }

It reads 4 bytes, which I called `expected`, and compares them with `acc`. The decompiler writes it backwards (`acc != expected ? 0 : 1`), but it means that flag is 1 if they're equal and 0 if they're not. Nothing else changes.

`case 7:`

![VM handler for case 7](img-117.webp){: w="900" h="252" }

It reads 2 bytes, which I called `target`, and moves `pc` past them. Then it looks at `flag`. If it's 1, it does nothing more and the VM goes on with the next instruction. If it's 0, it sets `pc` to `target`, so the next instruction comes from somewhere else in the bytecode. In other words, it's a `jump` that only happens when the last comparison failed.

`case 8:`

![VM handler for case 8](img-118.webp){: w="900" h="115" }

Also a `jump`, but this one always happens, it reads 2 bytes and puts them straight into `pc`.

`case 9:`

![VM handler for case 9](img-119.webp){: w="900" h="115" }

It writes a 1 in `accepted` and returns 1. This is the only place where the password can be accepted.

`case 0xb:`

![VM handler for case 0xb](img-120.webp){: w="900" h="134" }

Like case 1 and case 3, but it multiplies `acc` by the 4-byte number, which I called `factor`.

`case 0xc:`

![VM handler for case 0xc](img-121.webp){: w="900" h="271" }

The twin of case 2. It reads a position, takes that character of the password (or 0), and XORs it into `acc` instead of adding it. Its variables became `idx2` and `ch2`.

`case 0xa:`

![VM handler for case 0xa](img-122.webp){: w="900" h="96" }

I left this one for the end because it confused me. All the decompiler shows for it is two breaks, so the question was where those breaks lead. At the end of `vm_run` there are two possible places:

![The two possible exits at the end of vm_run](img-123.webp){: w="849" h="142" }

1. `return 0`, right after the switch but still inside the loop. If case `0xa` ends up here, `vm_run` returns 0.
1. `accepted = 0; return 1`, after the whole loop. If it ends up here, `vm_run` returns 1, but with `accepted` at 0.

Both make the level fail, but they're different results, and from the two breaks alone I couldn't tell which exit case `0xa` takes. So I switched to the disassembly, where Binary Ninja labels where each case of the jump table lands:

![Jump table targets in the disassembly](img-124.webp){: w="1081" h="338" }

So case `0xa` takes the second exit. It writes a 0 in `accepted`, sets `eax` (the return value) to 1, and jumps straight to the end of the function.

Putting it all together, this is the instruction set of the VM:

| Opcode | Argument | What it does |
| --- | --- | --- |
| 0x01 | 4 bytes | acc = value |
| 0x02 | 1 byte, i | acc += password[i] (0 if i is past the end) |
| 0x03 | 4 bytes | acc ^= value |
| 0x04 | 1 byte, n | acc = rol32(acc, n) |
| 0x05 | - | acc = length of the password |
| 0x06 | 4 bytes | flag = (acc == value) |
| 0x07 | 2 bytes | if flag is 0, jump to that position |
| 0x08 | 2 bytes | jump to that position |
| 0x09 | - | accept: accepted = 1, return 1 |
| 0x0a | - | reject: accepted = 0, return 1 |
| 0x0b | 4 bytes | acc *= value |
| 0x0c | 1 byte, i | acc ^= password[i] (0 if i is past the end) |
| 0x00 | - | return 0 |

It took me quite a while to see how all these pieces fit together, but in the end I understood a few things:

1. The only instruction that accepts the password is `0x09`, so the bytecode has to reach one to pass.
1. The only instruction that can choose between two paths is `0x07`, the conditional jump. Every other instruction always does the same thing. The math ones run and move on to the next one, and `0x08` always jumps.
1. `0x07` decides by looking at `flag`, and the only instruction that writes `flag` is `0x06`, by comparing `acc` with a number from the bytecode.
1. The password only gets into `acc` through `0x02` and `0x0c`, one character at a time, and through `0x05`, which loads its length.

Putting those together, the program must look something like this: compute a number from the password into `acc`, compare it with `0x06`, and use `0x07` to jump away when it doesn't match, probably to a `0x0a`.

With the table in hand, I went back to the dump and read the bytes the same way the VM would. I couldn't just pick any byte and look it up in the table, because a byte can be an opcode or part of an argument depending on where you start reading. For example, the third byte is a `0C`: on its own it would look like opcode `0x0c`, but it's actually part of the number that comes after the `06`. So, I started where `pc` starts, at position 0, and went one instruction at a time, using the table to know how many bytes each one takes:

1. **Position 0** is a `05`. In the table, `0x05` has no argument, so this instruction is just 1 byte: `acc = length of the password`. The next one starts at position 1.
1. **Position 1** is a `06`, which has a 4-byte argument: `0C 00 00 00`, the number 12 (x86 stores the lowest byte first, so the bytes are read backwards). So `flag = (acc == 12)`. The instruction takes 5 bytes, and the next one starts at position 6.
1. **Position 6** is a `07`, with a 2-byte argument: `88 00`, the number `0x88`. If `flag` is 0, it jumps to position `0x88`. It takes 3 bytes, so the next one starts at position 9.
1. **Position 9** is a `01`, with a 4-byte argument: `05 15 00 00`, the number `0x1505`. It puts that number in `acc`.

I kept going the same way until the end of the 137 bytes following always the same rule: the next instruction starts at the current position, plus 1 for the opcode, plus the size of its argument. From the table of opcodes, that means an instruction takes 1 byte with no argument (0x05, 0x09, 0x0a), 2 bytes with a 1-byte argument (0x02, 0x04, 0x0c), 3 bytes with a 2-byte one (0x07, 0x08) and 5 bytes with a 4-byte one (0x01, 0x03, 0x06, 0x0b). Most of the program turned out to be the same three instructions repeated, 0x04, 0x02 and 0x03, which take 2 + 2 + 5 = 9 bytes, so each round starts 9 positions after the previous one (0E, 17, 20, 29...). And the count closed exactly: the last instruction is at 0x88, so the program ends at 0x89, the 137 bytes. If I had gotten any size wrong, I would have known because I would have ended up short or past the end.

Written down as a table, with the position (in hex), the bytes and what each instruction does, this is the whole program:

| Position | Bytes | What it does |
| --- | --- | --- |
| 00 | 05 | acc = length of the password |
| 01 | 06 0C 00 00 00 | flag = (acc == 0xC) |
| 06 | 07 88 00 | if flag == 0: jump to 88 |
| 09 | 01 05 15 00 00 | acc = 0x1505 |
| 0E | 04 05 | acc = rol32(acc, 5) |
| 10 | 02 00 | acc += password[0] |
| 12 | 03 00 01 00 00 | acc ^= 0x100 |
| 17 | 04 05 | acc = rol32(acc, 5) |
| 19 | 02 01 | acc += password[1] |
| 1B | 03 01 01 00 00 | acc ^= 0x101 |
| 20 | 04 05 | acc = rol32(acc, 5) |
| 22 | 02 02 | acc += password[2] |
| 24 | 03 02 01 00 00 | acc ^= 0x102 |
| 29 | 04 05 | acc = rol32(acc, 5) |
| 2B | 02 03 | acc += password[3] |
| 2D | 03 03 01 00 00 | acc ^= 0x103 |
| 32 | 04 05 | acc = rol32(acc, 5) |
| 34 | 02 04 | acc += password[4] |
| 36 | 03 04 01 00 00 | acc ^= 0x104 |
| 3B | 04 05 | acc = rol32(acc, 5) |
| 3D | 02 05 | acc += password[5] |
| 3F | 03 05 01 00 00 | acc ^= 0x105 |
| 44 | 04 05 | acc = rol32(acc, 5) |
| 46 | 02 06 | acc += password[6] |
| 48 | 03 06 01 00 00 | acc ^= 0x106 |
| 4D | 04 05 | acc = rol32(acc, 5) |
| 4F | 02 07 | acc += password[7] |
| 51 | 03 07 01 00 00 | acc ^= 0x107 |
| 56 | 04 05 | acc = rol32(acc, 5) |
| 58 | 02 08 | acc += password[8] |
| 5A | 03 08 01 00 00 | acc ^= 0x108 |
| 5F | 04 05 | acc = rol32(acc, 5) |
| 61 | 02 09 | acc += password[9] |
| 63 | 03 09 01 00 00 | acc ^= 0x109 |
| 68 | 04 05 | acc = rol32(acc, 5) |
| 6A | 02 0A | acc += password[10] |
| 6C | 03 0A 01 00 00 | acc ^= 0x10A |
| 71 | 04 05 | acc = rol32(acc, 5) |
| 73 | 02 0B | acc += password[11] |
| 75 | 03 0B 01 00 00 | acc ^= 0x10B |
| 7A | 03 BE BA FE CA | acc ^= 0xCAFEBABE |
| 7F | 06 5B 17 EB 71 | flag = (acc == 0x71EB175B) |
| 84 | 07 88 00 | if flag == 0: jump to 88 |
| 87 | 09 | accept |
| 88 | 0A | reject |

Read like that, the program is shorter than it looked:

1. **Positions 00 to 06:** the `05` at `00` loads the length of the password, the `06` at `01` compares it with 12, and the `07` at `06` jumps to `88` if they're different. At `88` there's a `0x0a` that rejects it. So the password must have exactly 12 characters, and my guess about the jump was right.
1. **Position 09:** the `01` starts `acc` at `0x1505`. That number is the same starting value `djb2` had in level 3.
1. **Positions 0E to 75:** one round per character, always the same three instructions. In the first round, the `04` at `0E` rotates `acc` left by 5 bits, the `02` at `10` adds `password[0]`, and the `03` at `12` XORs `acc` with `0x100`. The second round does the same at `17`, `19` and `1B` with `password[1]` and `0x101`, and so on, until the last round at `71`, `73` and `75`, with `password[11]` and `0x10B`.
1. **Positions 7A to 88:** the `03` at `7A` XORs `acc` with `0xCAFEBABE` (second coffee mention), the `06` at `7F` compares it with `0x71EB175B`, and the `07` at `84` jumps to `88` if they're different. If they match, the jump isn't taken and the VM reaches the `09` at `87`, which accepts the password. If not, it lands on the `0A` at `88` and the password is rejected.

To check my hand decoding, I wanted to watch the VM run. My first idea was a normal breakpoint on the switch, looking at the opcode and `acc` every time it stopped. But with a 12-character password the VM could run dozens of instructions, and pressing F9 and writing down values that many times sounded slow and easy to get wrong. So I looked for a way to make x64dbg take the notes for me. Here's what I found:

- [Conditional breakpoints](https://help.x64dbg.com/en/latest/introduction/ConditionalBreakpoint.html): a breakpoint can have a break condition, and if it evaluates to 0 the program doesn't stop. It can also have a log text, which is written to the Log tab every time the breakpoint is hit.
- [String formatting](https://help.x64dbg.com/en/latest/introduction/Formatting.html): to put values in that text, anything between braces, like `{eax}`, is evaluated and printed, in hex by default.
- [Values](https://help.x64dbg.com/en/latest/introduction/Values.html): inside those braces, `[addr]` reads memory, and `dword:[addr]` reads exactly 4 bytes.

With that, I set up the breakpoint like this:

- **Where:** on the `cmp eax, 0xc` of the switch, where `eax` already holds the opcode.
- **Break condition:** 0, so it never stops.
- **Log text:** the position of the instruction, the opcode and `acc` before it runs. It took me a few runs to figure out exactly how to format it. In the disassembly, `pc` is at `rbp-0x18` (already one past the opcode at that point, hence the -1) and `acc` is at `rbp-0xC`.

Back in the program, I typed the password `test`:

![Logging breakpoint trace for the password test](img-125.webp){: w="735" h="70" }

Only four instructions, exactly the ones I had decoded by hand: `0x05` loads the length (4), `0x06` compares it with 12, `0x07` jumps to position `0x88` because they're different, and there the VM finds a `0x0a`, the reject. I tried again with 12 characters, `abcdefghijkl`:

![Logging breakpoint trace for abcdefghijkl](img-126.webp){: w="735" h="634" }

This time the length check passes and the whole program runs, in the same order as my listing. The positions in the log are the same ones I had written down by hand. After loading `0x1505` into `acc` with `0x01`, the same three instructions repeat 12 times, `0x04`, `0x02` and `0x03`, one round per character. At the end there's one more `0x03`, the comparison `0x06`, the jump `0x07`, and the same `0x0a` at `0x88`, so `vm_run` returns 1 with `accepted` at 0, and `level_4` prints the *"Incorrect password"* message.

Like in the other levels, I also checked who uses the global the password is copied to:

![Cross references of data_140015080](img-127.webp){: w="627" h="256" }

`print_completion_code` passes it to `sub_1400025c6` and also to `crc32`, so the level 4 password is part of the completion code too. I renamed it `level4_password`, and with that `level_4` reads like this:

![level_4 after renaming](img-128.webp){: w="1282" h="555" }

With the whole program understood, what's left is finding a password that gets through it.

### Working backwards

Something important about this program: it never compares the password with a stored password. It turns the 12 characters into a single 32-bit number, `acc`, and the only thing it checks is whether that number ends up being `0x71EB175B`. Any 12 characters that produce that number pass. That also means there isn't just one valid password. 12 characters are 96 bits squeezed into a 32-bit number, so, like with a hash, lots of different passwords end up with the same result. The goal wasn't to recover "the" password, but to find any password that produces that number.

The check happens at the very end, so I started from there. To pass, `acc` has to be `0x71EB175B` right after the XOR with `0xCAFEBABE`. Doing the same XOR twice undoes it, so before that XOR, at the end of the 12 rounds, `acc` must be `0x71EB175B ^ 0xCAFEBABE`.

A round can be undone too, as long as I know which character it used. Each of its three steps has an opposite:

1. `acc = rol32(acc, 5)` is undone by rotating 5 bits to the right.
1. `acc += password[i]` is undone by subtracting the character.
1. `acc ^= 0x100 + i` is undone with the same XOR.

Going backwards, they're undone in reverse order. First the XOR, then the subtraction, then the rotation. To try this, I wrote a script with a function, `undo`, that takes a value of `acc` at the end of the 12 rounds and a password, and undoes the rounds one by one from the last to the first, to get the value `acc` had at the start. Then I ran it twice with `abcdefghijkl`:

```python
def ror32(x, n):
    return ((x >> n) | (x << (32 - n))) & 0xffffffff

def undo(acc, pw):
    # undo the 12 rounds, from the last one to the first
    for i in range(11, -1, -1):
        acc = ror32(((acc ^ (0x100 + i)) - pw[i]) & 0xffffffff, 5)
    return acc

pw = b"abcdefghijkl"

# value the trace showed after the rounds for abcdefghijkl
print("from the trace:  %08X" % undo(0x8E7FBD34, pw))

# value needed to pass: 0x71EB175B before the xor with 0xCAFEBABE
print("from the target: %08X" % undo(0x71EB175B ^ 0xCAFEBABE, pw))
```
{: file="backwards.py" }

```text
from the trace:  00001505
from the target: C75EFFD6
```

The first line is good news, it tells that starting from the trace and undoing the 12 rounds, I got back exactly to `0x1505`, so the undoing is right. The second one shows the problem, starting from the value needed to pass I end up at `0xC75EFFD6` instead of `0x1505`. That is what I expected since `abcdefghijkl` isn't the password, but it shows I can't just pick any characters on the way back.

### Brute force

My next idea was to pick the characters myself while going backwards. That works until the very first character, by then, `acc` before it has to be exactly `0x1505`, so its value is no longer my choice, it's forced by the other 11. To see what that forced value looks like, I wrote another script. It picks the last 11 characters at random, goes backwards, and calculates what the first one would have to be:

```python
import random
import string

def rol32(x, n):
    return ((x << n) | (x >> (32 - n))) & 0xffffffff

def ror32(x, n):
    return ((x >> n) | (x << (32 - n))) & 0xffffffff

goal = 0x71EB175B ^ 0xCAFEBABE

for attempt in range(10):
    # pick the last 11 characters at random
    rest = "".join(random.choice(string.ascii_letters) for _ in range(11)).encode()
    acc = goal
    for i in range(11, 0, -1):
        acc = ror32(((acc ^ (0x100 + i)) - rest[i - 1]) & 0xffffffff, 5)
    # now acc is the value after the first round, so the first char is forced
    first = ((acc ^ 0x100) - rol32(0x1505, 5)) & 0xffffffff
    print("?" + rest.decode(), "-> the first char would have to be %08X" % first)
```
{: file="first_char.py" }

```text
?dWFIYfIjTba -> the first char would have to be 4EE20312
?cUBeeKtimgs -> the first char would have to be 6788CE98
?mwAtcdkYDgU -> the first char would have to be 13732425
?UKiAddquIJG -> the first char would have to be AECF27A5
?JUUsMBaUVAP -> the first char would have to be E30DC49F
?KJxoYWylWVL -> the first char would have to be 0A874669
?OKOkSqqrczo -> the first char would have to be E3A1BFE1
?xbYvfyLtmzj -> the first char would have to be AF9F1754
?mGvzAGDboYf -> the first char would have to be 2489B68E
?UkSbdFExWDI -> the first char would have to be 36A99B13
```

Ten tries and none of them is a character. They're 32-bit numbers like `4EE20312`, when a character I can type is somewhere between `0x21` (`!`) and `0x7E` (`~`), the printable characters of the [ASCII table](https://en.wikipedia.org/wiki/ASCII), without the space. Since `acc` is a 32-bit number, the forced value can be any of 2^32 = 4,294,967,296 values, and only 94 of them are characters I can type.

Still, nothing stops me from trying again and again until one of them falls in that range, so I wrote a script that does exactly that and measured how long it takes:

```python
import random
import time

def rol32(x, n):
    return ((x << n) | (x >> (32 - n))) & 0xffffffff

def ror32(x, n):
    return ((x >> n) | (x << (32 - n))) & 0xffffffff

goal = 0x71EB175B ^ 0xCAFEBABE
start = rol32(0x1505, 5)
chars = range(0x21, 0x7f)  # printable, without the space

t = time.time()
tries = 0
while True:
    tries += 1
    rest = bytes(random.choices(chars, k=11))
    acc = goal
    for i in range(11, 0, -1):
        acc = ror32(((acc ^ (0x100 + i)) - rest[i - 1]) & 0xffffffff, 5)
    first = ((acc ^ 0x100) - start) & 0xffffffff
    if 0x21 <= first <= 0x7e:
        break

print("password: " + chr(first) + rest.decode())
print("tries: %d" % tries)
print("time: %.1f seconds" % (time.time() - t))
```
{: file="brute_first_char.py" }

```text
password: }p:AS~h+TV/)
tries: 3723181
time: 22.4 seconds
```

It found `}p:AS~h+TV/)` after 3.7 million tries and 22 seconds, but that run was lucky. With 94 good values out of 4,294,967,296, the average is around 45 million tries, and at about 8 microseconds per try in Python, that's several minutes for every password. So, technically, I already had a way to get passwords.

### A password with my name

Two things didn't convince me, though. The first one is that the passwords come out as random garbage like `}p:AS~h+TV/)`, while the keygens of the other levels started from my name. I wanted something similar here, a password that starts with `Bruno_`. The second one is that every password costs minutes.

Fixing a prefix raised a question: if 6 of the 12 characters are already decided, are the other 6 enough for a valid password to exist? Each combination of the free characters ends up as one of the 4,300 million possible values of `acc`, more or less at random, so on average only one in 4,300 million combinations is a valid password. Dividing the number of combinations by that, I could estimate how many valid passwords to expect for each number of free characters:

| Free characters | Possible combinations (94^n) | Valid passwords to expect (÷ 4,300 million) |
| --- | --- | --- |
| 1 | 94 | practically none |
| 4 | about 78 million | about 0.02, so most likely none |
| 5 | about 7,300 million | about 2 |
| 6 | about 690,000 million | about 160 |

With 4 free characters there probably isn't a single valid password, no matter how well I search. With 6 there should be around 160. So, I left 6 free and used the other 6 for my name.

Then the question was how to find them quickly. Guessing those 6 characters at random would take about 4,300 million tries on average, hours in Python. Calculating one of them from the others, like in the brute force, would bring it back to minutes per password.

Here I got a bit stuck. I didn't see a way to do better than minutes. I kept thinking about the one thing I knew for sure, that the rounds could be undone, but I couldn't turn that into a plan. So I started reading about how people attack this kind of problem, where you know the start and the end of a calculation but not the values in between, and that's how I found the meet-in-the-middle attack. [Baeldung's explanation](https://www.baeldung.com/cs/security-meet-in-the-middle-attack) has a sentence that made sense for my problem:

> The attack takes advantage of the fact that the encryption and decryption processes are inverse operations of each other.

That was exactly my situation. The rounds of the VM play the part of the encryption, and my undo function is their inverse. So instead of going all the way from one end, I could go forwards from the start and backwards from the end, brute-forcing a few characters on each side, and look for the point where both meet. Applied to this password, it looks like this:

```text
position:  0 1 2 3 4 5 | 6 7 8 | 9 10 11
password:  B r u n o _ | ? ? ? | ?  ?  ?
                          -->      <--
```

1. **Forwards:** from `0x1505`, I run the rounds of `Bruno_`, then try every combination for positions 6 to 8, and save each resulting `acc` in a table.
1. **Backwards:** from the value needed to pass, I undo the rounds of positions 11, 10 and 9, trying every combination of those 3 characters.
1. **The meeting point:** when a value from the backwards side is in the table, both halves fit together, and that's a valid password.

I wrote it as a script, with `Bruno_` as the prefix:

```python
import time

def rol32(x, n):
    return ((x << n) | (x >> (32 - n))) & 0xffffffff

def ror32(x, n):
    return ((x >> n) | (x << (32 - n))) & 0xffffffff

goal = 0x71EB175B ^ 0xCAFEBABE
prefix = b"Bruno_"
chars = range(0x21, 0x7f)  # printable, without the space

t = time.time()

# forwards: the prefix, then every combination for positions 6 to 8
acc = 0x1505
for i, c in enumerate(prefix):
    acc = ((rol32(acc, 5) + c) & 0xffffffff) ^ (0x100 + i)

middle = {}
for a in chars:
    acc6 = ((rol32(acc, 5) + a) & 0xffffffff) ^ 0x106
    for b in chars:
        acc7 = ((rol32(acc6, 5) + b) & 0xffffffff) ^ 0x107
        for c in chars:
            acc8 = ((rol32(acc7, 5) + c) & 0xffffffff) ^ 0x108
            middle[acc8] = bytes([a, b, c])

# backwards: undo positions 11, 10 and 9 with every combination
found = 0
for z in chars:
    acc11 = ror32(((goal ^ 0x10B) - z) & 0xffffffff, 5)
    for y in chars:
        acc10 = ror32(((acc11 ^ 0x10A) - y) & 0xffffffff, 5)
        for x in chars:
            acc9 = ror32(((acc10 ^ 0x109) - x) & 0xffffffff, 5)
            if acc9 in middle:
                pw = prefix + middle[acc9] + bytes([x, y, z])
                print("%s  meets at %08X" % (pw.decode(), acc9))
                found += 1

print("found %d passwords in %.1f seconds" % (found, time.time() - t))
input("Press Enter to exit")
```
{: file="mitm.py" }

The forwards part fills a table, `middle`, with the value of `acc` after position 8 for every combination of positions 6 to 8. The backwards part undoes positions 11, 10 and 9 for every combination, and every time the result is in the table, both halves fit and it prints the password. Running it:

```text
Bruno_ZCu9*.  meets at 90817621
Bruno_ZCtY*.  meets at 90817620
Bruno_ZCcy*.  meets at 9081761F
Bruno_ZCu6J.  meets at 90817621
Bruno_ZCtVJ.  meets at 90817620
Bruno_ZCcvJ.  meets at 9081761F
Bruno_ZCu7j.  meets at 90817621
Bruno_ZCtWj.  meets at 90817620
Bruno_ZCcwj.  meets at 9081761F
Bruno_ZCu9-N  meets at 90817621
Bruno_ZCtY-N  meets at 90817620
Bruno_ZCcy-N  meets at 9081761F
Bruno_ZCu6MN  meets at 90817621
Bruno_ZCtVMN  meets at 90817620
Bruno_ZCcvMN  meets at 9081761F
Bruno_ZCu7mN  meets at 90817621
Bruno_ZCtWmN  meets at 90817620
Bruno_ZCcwmN  meets at 9081761F
Bruno_ZCu9,n  meets at 90817621
Bruno_ZCtY,n  meets at 90817620
Bruno_ZCcy,n  meets at 9081761F
Bruno_ZCu6Ln  meets at 90817621
Bruno_ZCtVLn  meets at 90817620
Bruno_ZCcvLn  meets at 9081761F
Bruno_ZCu7ln  meets at 90817621
Bruno_ZCtWln  meets at 90817620
Bruno_ZCcwln  meets at 9081761F
found 27 passwords in 0.7 seconds
Press Enter to exit
```

27 passwords in 0.7 seconds, instead of minutes for a single one. Take the first one, `Bruno_ZCu9*.`: going forwards with `Bruno_ZCu`, `acc` is `0x90817621` after position 8, and going backwards from the value needed to pass, undoing `.`, `*` and `9`, `acc` is also `0x90817621` before position 9. Both halves meet at the same value, so the whole password works.

Two things caught my attention, though. All of them start with `Bruno_ZC`, and there are only 27, when I expected around 160. After some tinkering, I realized my table keeps only one combination for each value of `acc`, so if several combinations end up at the same value, the last one overwrites the others. To check it, I changed the table to keep a list of combinations for each value, and made the script print how many different values there are and a sample of the passwords it finds:

```python
import time

def rol32(x, n):
    return ((x << n) | (x >> (32 - n))) & 0xffffffff

def ror32(x, n):
    return ((x >> n) | (x << (32 - n))) & 0xffffffff

goal = 0x71EB175B ^ 0xCAFEBABE
prefix = b"Bruno_"
chars = range(0x21, 0x7f)  # printable, without the space

t = time.time()

# forwards: the prefix, then every combination for positions 6 to 8
acc = 0x1505
for i, c in enumerate(prefix):
    acc = ((rol32(acc, 5) + c) & 0xffffffff) ^ (0x100 + i)

middle = {}
for a in chars:
    acc6 = ((rol32(acc, 5) + a) & 0xffffffff) ^ 0x106
    for b in chars:
        acc7 = ((rol32(acc6, 5) + b) & 0xffffffff) ^ 0x107
        for c in chars:
            acc8 = ((rol32(acc7, 5) + c) & 0xffffffff) ^ 0x108
            # keep every combination, not just the last one
            middle.setdefault(acc8, []).append(bytes([a, b, c]))

print("combinations: %d, different values: %d" % (94 ** 3, len(middle)))

# backwards: undo positions 11, 10 and 9 with every combination
found = []
for z in chars:
    acc11 = ror32(((goal ^ 0x10B) - z) & 0xffffffff, 5)
    for y in chars:
        acc10 = ror32(((acc11 ^ 0x10A) - y) & 0xffffffff, 5)
        for x in chars:
            acc9 = ror32(((acc10 ^ 0x109) - x) & 0xffffffff, 5)
            for mid in middle.get(acc9, []):
                found.append(prefix + mid + bytes([x, y, z]))

print("found %d passwords in %.1f seconds" % (len(found), time.time() - t))
for pw in found[::25]:
    print(pw.decode())
input("Press Enter to exit")
```
{: file="mitm_all.py" }

```text
combinations: 830584, different values: 102704
found 243 passwords in 0.8 seconds
Bruno_W!59*.
Bruno_ZBCy*.
Bruno_YccvJ.
Bruno_Ya#wj.
Bruno_W"Cy-N
Bruno_ZCtVMN
Bruno_ZA4WmN
Bruno_YbTY,n
Bruno_W#tVLn
Bruno_W!4Wln
Press Enter to exit
```

That was it. The 830,584 combinations of the forwards side give only 102,704 different values, so lots of them share a value, and the first version was throwing most of them away. Keeping all of them, the same search finds 243 passwords in 0.8 seconds, much closer to the estimate, and with all kinds of characters after `Bruno_`.

At the end I realized I could have just used the meet in the middle attack without a prefix, but having a prefix makes the password much more significant, so that's the version I used for the keygen.

### Level 4, step by step

Before writing the keygen, this is the whole level put together:

1. `level_4` runs its four anti-debugging checks once, then asks for a password and passes it to `vm_run`.
1. `vm_run` decrypts 137 bytes of bytecode (XOR with `0xa5`, `0xa6`, `0xa7`...), verifies the CRC32 of the original encrypted bytes, and runs the decrypted bytecode in a small VM with one register, `acc`, a `flag`, and 12 instructions.
1. The bytecode checks that the password has exactly 12 characters.
1. It starts `acc` at `0x1505` and, for each character, rotates `acc` left by 5 bits, adds the character, and XORs `acc` with `0x100` plus the position of the character.
1. At the end, it XORs `acc` with `0xCAFEBABE`, and the password passes only if the result is `0x71EB175B`.
1. The rounds can be undone, so a meet-in-the-middle search can find passwords that start with 6 characters of my choice.

### Keygen

The keygen is the second version of the search, with two changes. It asks for the first 6 characters instead of always using `Bruno_`, and it uses the code page of the console, like the keygens of the other levels, so the characters I type in the keygen are the same bytes the challenge receives. It also tells me when it can't find any password, but I'll come back to that in the border cases:

```python
import ctypes

def rol32(x, n):
    return ((x << n) | (x >> (32 - n))) & 0xffffffff

def ror32(x, n):
    return ((x >> n) | (x << (32 - n))) & 0xffffffff

# acc right before the final xor with 0xCAFEBABE
goal = 0x71EB175B ^ 0xCAFEBABE
chars = range(0x21, 0x7f)    # printable, without the space

# console input code page, like in the other levels
cp = "cp%d" % ctypes.windll.kernel32.GetConsoleCP()
prefix = input("First 6 characters: ").encode(cp)

if len(prefix) != 6:
    print("It has to be exactly 6 characters")
else:
    # forwards: the prefix, then every combination for positions 6 to 8
    acc = 0x1505
    for i, c in enumerate(prefix):
        acc = ((rol32(acc, 5) + c) & 0xffffffff) ^ (0x100 + i)

    middle = {}
    for a in chars:
        acc6 = ((rol32(acc, 5) + a) & 0xffffffff) ^ 0x106
        for b in chars:
            acc7 = ((rol32(acc6, 5) + b) & 0xffffffff) ^ 0x107
            for c in chars:
                acc8 = ((rol32(acc7, 5) + c) & 0xffffffff) ^ 0x108
                middle.setdefault(acc8, []).append(bytes([a, b, c]))

    # backwards: undo positions 11, 10 and 9 and look for a match
    found = []
    for z in chars:
        acc11 = ror32(((goal ^ 0x10B) - z) & 0xffffffff, 5)
        for y in chars:
            acc10 = ror32(((acc11 ^ 0x10A) - y) & 0xffffffff, 5)
            for x in chars:
                acc9 = ror32(((acc10 ^ 0x109) - x) & 0xffffffff, 5)
                for mid in middle.get(acc9, []):
                    found.append(prefix + mid + bytes([x, y, z]))

    if not found:
        print("No password starts with that, try other 6 characters")
    else:
        print("Found %d passwords, for example:" % len(found))
        for pw in found[::max(1, len(found) // 10)][:10]:
            print(pw.decode(cp))

input("Press Enter to exit")
```
{: file="keygen4.py" }

Running it with my name:

```text
First 6 characters: Bruno_
Found 243 passwords, for example:
Bruno_W!59*.
Bruno_ZA#y*.
Bruno_Ya#vJ.
Bruno_W!#wj.
Bruno_ZA4Y-N
Bruno_Ya4VMN
Bruno_W!4WmN
Bruno_ZA59,n
Bruno_Ya56Ln
Bruno_W!57ln
Press Enter to exit
```

Check inside the binary, first with a random password and then with the first one the keygen gave me:

![Level 4 rejecting a random password and accepting Bruno_W!59*.](img-141.webp){: w="736" h="193" }

Level 4 passed, and with it the whole challenge. Right after that, the program prints the completion code, which I'll come back to at the end:

![Challenge complete, with the completion code](img-142.webp){: w="736" h="141" }

### Border cases

#### Prefixes without passwords

While trying the keygen with other prefixes, I found something I didn't expect. Some of them give no password at all, like `Gecko!`:

```text
First 6 characters: Gecko!
No password starts with that, try other 6 characters
Press Enter to exit
```

After trying a few more, it turned out to be all or nothing. `Bruno_` gives 243 passwords, `abcdef` 216, `Gecko_` 213 and `RE-WIN` 162, but `Gecko!`, `Binary`, `Hello!`, `AAAAAA`, `Bruno!` and `Academ` give none. So the estimate of about 160 was only an average, and valid passwords are not evenly distributed across prefixes. That's why I adjusted the keygen so that, when it doesn't find anything, it says so and asks for other 6 characters, instead of printing an empty list.

#### Wrong length

The first thing the bytecode checks is the length, so a password from the keygen with one character less, or one more, fails right away:

![Level 4 rejecting passwords of 11 and 13 characters](img-144.webp){: w="736" h="111" }

#### Many valid passwords

Since there isn't a single password, any of the ones the keygen prints should work, not only the first one. `Bruno_Ya#vJ.`, another one from the same list, also passes:

![Level 4 accepting Bruno_Ya#vJ. with a different completion code](img-145.webp){: w="736" h="168" }

The completion code is different from the previous run. It includes the level 4 password, so it changes with every password. I'll come back to this at the end.

#### Accented characters

Like in the other levels, the keygen encodes what I type with the code page of the console. With `Brunó_` it also finds 243 passwords:

```text
First 6 characters: Brunó_
Found 243 passwords, for example:
Brunó_W!55'6
Brunó_ZA#u'6
Brunó_Ya#rG6
Brunó_W!#sg6
Brunó_ZA4U&V
Brunó_Ya4RFV
Brunó_W!4SfV
Brunó_ZA55!v
Brunó_Ya52Av
Brunó_W!53av
Press Enter to exit
```

And `Brunó_Ya#rG6` passes in the binary:

![Level 4 accepting Brunó_Ya#rG6](img-147.webp){: w="736" h="49" }

In cp850 the ó is a single byte, so the password is 12 bytes long. If the keygen had used UTF-8, the ó would take two bytes, and the 13-byte password wouldn't even pass the length check. Level 4 is done, and with it the whole challenge.

## The completion code

While testing level 4 I got three different completion codes, one for each password, so before choosing which one to submit I wanted to know exactly what goes into it. Back in Binary Ninja, `print_completion_code` was already easy to read, since every global it touches had been renamed while going through the levels:

![print_completion_code](img-148.webp){: w="1586" h="380" }

It starts a 32-bit value, `var_c`, at `0x811c9dc5`, and then passes a pointer to it to `sub_1400025c6` five times, each time with a different string:

1. `arg1`, which in `main` is `&var_88`, the same buffer `level_1` and `level_2` get (the username).
1. `level1_hash`, printed with `"%08X"`, so the level 1 serial in uppercase hex.
1. `level2_word`, printed with `"%04X"`, so the first group of the level 2 serial.
1. `level3_password`.
1. `level4_password`.

At the end it prints `var_c` and, after a dash, the `crc32` of the level 4 password.

`sub_1400025c6` was the only part I hadn't seen yet:

![sub_1400025c6, FNV-1a](img-149.webp){: w="1586" h="384" }

It goes through the string byte by byte until the null terminator and, for each byte, XORs the value with it and multiplies it by `0x1000193`. Just like with the CRC32 in level 4, a quick search for `0x811c9dc5` and `0x1000193` told me they are the offset basis and the prime of the 32-bit [FNV-1a hash](https://en.wikipedia.org/wiki/Fowler%E2%80%93Noll%E2%80%93Vo_hash_function). Since the same value goes through all five calls, the first half of the code is a single `FNV-1a` hash of the five strings one after the other. I renamed it `fnv1a`.

So the completion code is something like a fingerprint of the whole run. It depends on the username and on what I entered in every level, and the second half depends only on the level 4 password. That explains why each level 4 password gave me a different code. To make sure I had understood it, I rebuilt it in Python with the values of my first complete run:

```python
import zlib

def fnv(h, s):
    for c in s:
        h = ((h ^ c) * 0x1000193) & 0xffffffff
    return h

username = b"Bruno"
lvl1 = b"0038D6D8"           # level1_hash with %08X
lvl2 = b"D31B"               # level2_word with %04X
lvl3 = b"AFB0685485B42B8B"
lvl4 = b"Bruno_W!59*."

h = 0x811c9dc5
for s in [username, lvl1, lvl2, lvl3, lvl4]:
    h = fnv(h, s)

print("%08X-%08X" % (h, zlib.crc32(lvl4)))
```
{: file="completion.py" }

```text
937F16DE-EB8C1201
```

`937F16DE-EB8C1201`, the same code the binary printed after `Bruno_W!59*.`. That's the one I submitted, from this run:

| Level | Input |
| --- | --- |
| 1 | Username `Bruno`, serial `0038D6D8` |
| 2 | Serial `D31B-84B1-76D4-217E` |
| 3 | Username `Bruno`, password `AFB0685485B42B8B` |
| 4 | Password `Bruno_W!59*.` |

Completion code: `937F16DE-EB8C1201`

## Conclusions

Level 4 was by far the hardest part of the challenge. Understanding the VM already took me a while. I had to decrypt the bytecode, figure out the instructions one case at a time, decode it by hand and trace it in x64dbg. But even once I understood exactly what the bytecode did, I still spent hours looking for a way to get a password out of it. Working backwards let me calculate the first character from the other eleven. Brute force eventually found a valid password, but I wanted a faster way to get one starting with my name. I was fairly lost until I came across the meet-in-the-middle attack. Being able to choose the first 6 characters and get hundreds of valid passwords in less than a second was definitely the most satisfying moment of the whole challenge.

What I'm taking away from it:

- MinGW-w64 links its own stdio functions into the binary, so `printf` and friends show up as local functions instead of imports.
- Binary Ninja has to be told when a function is variadic; otherwise calls end up nested inside each other.
- A bytecode VM can be read one handler at a time, and once the opcode table exists, the bytecode can be decoded by hand.
- Logging breakpoints in x64dbg can trace a whole VM run without stopping it.
- The console code page decides which bytes a program receives, so a keygen has to encode its input the same way.

The method mattered as much as any of those: read the code first, confirm each assumption in the debugger, leave the binary unpatched, and look for every valid input instead of just one. Most of the border cases in this writeup came from that last rule.

Finally, thanks to the Binary Gecko team for putting this challenge together. It was a lot of fun to solve, and I learned more from it than I expected.
