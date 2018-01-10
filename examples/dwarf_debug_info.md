Primer vzet neposredno iz [How debuggers work - Part 3 - Debugging information](https://eli.thegreenplace.net/2011/02/07/how-debuggers-work-part-3-debugging-information)

First let's take a glimpse of where the DWARF info is placed inside ELF files. ELF defines arbitrary sections that may exist in each object file. A section header table defines which sections exist and their names. Different tools treat various sections in special ways - for example the linker is looking for some sections, the debugger for others.

We'll be using an executable built from this C source for our experiments in this article, compiled into tracedprog2:

```c
#include <stdio.h>

void do_stuff(int my_arg)
{
    int my_local = my_arg + 2;
    int i;

    for (i = 0; i < my_local; ++i)
        printf("i = %d\n", i);
}

int main()
{
    do_stuff(2);
    return 0;
}
```

Dumping the section headers from the ELF executable using `objdump -h` we'll notice several sections with names beginning with .debug_ - these are the DWARF debugging sections:

```
26 .debug_aranges 00000020  00000000  00000000  00001037
                 CONTENTS, READONLY, DEBUGGING
27 .debug_pubnames 00000028  00000000  00000000  00001057
                 CONTENTS, READONLY, DEBUGGING
28 .debug_info   000000cc  00000000  00000000  0000107f
                 CONTENTS, READONLY, DEBUGGING
29 .debug_abbrev 0000008a  00000000  00000000  0000114b
                 CONTENTS, READONLY, DEBUGGING
30 .debug_line   0000006b  00000000  00000000  000011d5
                 CONTENTS, READONLY, DEBUGGING
31 .debug_frame  00000044  00000000  00000000  00001240
                 CONTENTS, READONLY, DEBUGGING
32 .debug_str    000000ae  00000000  00000000  00001284
                 CONTENTS, READONLY, DEBUGGING
33 .debug_loc    00000058  00000000  00000000  00001332
                 CONTENTS, READONLY, DEBUGGING
```

The first number seen for each section here is its size, and the last is the offset where it begins in the ELF file. The debugger uses this information to read the section from the executable.

Now let's see a few practical examples of finding useful debug information in DWARF.
## Finding functions ##

One of the most basic things we want to do when debugging is placing breakpoints at some function, expecting the debugger to break right at its entrance. To be able to perform this feat, the debugger must have some mapping between a function name in the high-level code and the address in the machine code where the instructions for this function begin.

This information can be obtained from DWARF by looking at the .debug_info section. Before we go further, a bit of background. The basic descriptive entity in DWARF is called the Debugging Information Entry (DIE). Each DIE has a tag - its type, and a set of attributes. DIEs are interlinked via sibling and child links, and values of attributes can point at other DIEs.

Let's run:
```
objdump --dwarf=info tracedprog2
```
The output is quite long, and for this example we'll just focus on these lines [3]:
```
<1><71>: Abbrev Number: 5 (DW_TAG_subprogram)
    <72>   DW_AT_external    : 1
    <73>   DW_AT_name        : (...): do_stuff
    <77>   DW_AT_decl_file   : 1
    <78>   DW_AT_decl_line   : 4
    <79>   DW_AT_prototyped  : 1
    <7a>   DW_AT_low_pc      : 0x8048604
    <7e>   DW_AT_high_pc     : 0x804863e
    <82>   DW_AT_frame_base  : 0x0      (location list)
    <86>   DW_AT_sibling     : <0xb3>

<1><b3>: Abbrev Number: 9 (DW_TAG_subprogram)
    <b4>   DW_AT_external    : 1
    <b5>   DW_AT_name        : (...): main
    <b9>   DW_AT_decl_file   : 1
    <ba>   DW_AT_decl_line   : 14
    <bb>   DW_AT_type        : <0x4b>
    <bf>   DW_AT_low_pc      : 0x804863e
    <c3>   DW_AT_high_pc     : 0x804865a
    <c7>   DW_AT_frame_base  : 0x2c     (location list)
```
There are two entries (DIEs) tagged `DW_TAG_subprogram`, which is a function in DWARF's jargon. Note that there's an entry for do_stuff and an entry for main. There are several interesting attributes, but the one that interests us here is `DW_AT_low_pc`. This is the program-counter (EIP in x86) value for the beginning of the function. Note that it's `0x8048604` for do_stuff. Now let's see what this address is in the disassembly of the executable by running `objdump -d`:
```
08048604 <do_stuff>:
 8048604:       55           push   ebp
 8048605:       89 e5        mov    ebp,esp
 8048607:       83 ec 28     sub    esp,0x28
 804860a:       8b 45 08     mov    eax,DWORD PTR [ebp+0x8]
 804860d:       83 c0 02     add    eax,0x2
 8048610:       89 45 f4     mov    DWORD PTR [ebp-0xc],eax
 8048613:       c7 45 (...)  mov    DWORD PTR [ebp-0x10],0x0
 804861a:       eb 18        jmp    8048634 <do_stuff+0x30>
 804861c:       b8 20 (...)  mov    eax,0x8048720
 8048621:       8b 55 f0     mov    edx,DWORD PTR [ebp-0x10]
 8048624:       89 54 24 04  mov    DWORD PTR [esp+0x4],edx
 8048628:       89 04 24     mov    DWORD PTR [esp],eax
 804862b:       e8 04 (...)  call   8048534 <printf@plt>
 8048630:       83 45 f0 01  add    DWORD PTR [ebp-0x10],0x1
 8048634:       8b 45 f0     mov    eax,DWORD PTR [ebp-0x10]
 8048637:       3b 45 f4     cmp    eax,DWORD PTR [ebp-0xc]
 804863a:       7c e0        jl     804861c <do_stuff+0x18>
 804863c:       c9           leave
 804863d:       c3           ret
```
Indeed, `0x8048604` is the beginning of do_stuff, so the debugger can have a mapping between functions and their locations in the executable.
