---
layout:     post
title:      "Binary patching and intro to assembly with r2"
subtitle:   "An introduction to reverse engineering and binary patching with the help of radare2"
date:       2018-06-18 18:00:00
categories: [reversing]
---

In this article we will take a look at radare2 and binary patching. Starting out to learn reverse engineering, especially with radare2, can
be a daunting task. So hopefully this article will help you with your journey towards reversing and binary hacking. I am simply trying to pass
my knowledge to people with the same interest as me.

# Radare2
r2 is an amazing tool for analyzing, debugging and disassemble binaries but it got a bit of a steep learning curve. It's similar to
[IDA](https://www.hex-rays.com/products/ida/index.shtml) and [Binary Ninja](https://binary.ninja/) but it got no GUI by default and the best
part is that it's *free and open source*. If you really want to use a GUI, you can check out [Cutter](https://github.com/radareorg/cutter)
but I will be using the command line in this article.

## Installing

### GNU/Linux & OS X
Installation on GNU/Linux and Mac are pretty straight forward, we just need to *git clone* the repository and execute the installation script.
First change the directory to where you want to save and then execute:
```
$ git clone https://github.com/radare/radare2
$ cd radare2
$ sudo sys/install.sh
```
This will install the latest version of radare2 and this way is highly recommended. To then **update** (I suggest you update every week) you
run the same command you installed with *(First change to your radare2 directory)*:
```
$ sudo sys/install.sh
```

### Windows
For installation on Windows you find the executable [here](http://radare.org/r/down.html)

I will be using *gcc* in this article so if you are on windows I suggest you to compile the binary using [MinGW](http://mingw.org/)

## Uninstallation
If you for some reason want to uninstall you need to first change your directory to where you installed and then run:
```
$ sudo make uninstall
$ sudo make purge
$ cd .. && rm -rf radare2
```

# Challenge
I wrote a basic program in C in order for us to learn about patching. To be able to follow along you can clone the repository:
```
$ git clone https://github.com/jlhk/patchme
$ cd patchme
```

First of all let us try to run the *patchme* binary and see what happens. I have compiled two version, Linux and Mac, so execute with whatever
OS you are using. I'll be using the Linux version throughout the whole article.
```
$ ./patchme_linux
Usage: ./patchme_linux <KEY>
```
We see that it requires one argument and that is the key for the program, so lets try and test with a random key:
```
$ ./patchme_linux TEST-KEY-1234
Wrong key! Expected: DFKE-IDHD-MCMP-8606
```
Hmm the key is written to standard output.. lets just try that key then.
```
$ ./patchme_linux DFKE-IDHD-MCMP-8606
Wrong key! Expected: EGOB-LFIO-EKKJ-4862
```
Wrong key again? Seems like the key changes everytime it's being executed. Lets open it up in radare2 to see whats going on.
```
$ r2 patchme_linux
 -- THE ONLY WINNING MOVE IS NOT TO PLAY.
[0x00000730]>
```

Under the message is a *hexadecimal* value surrounded by brackets. This value is your *current memory address* in the binary. Every command
in radare2 has been shortened to a single character but luckily it's super easy to look up commands with the help of the questionmark (**?**).
```
[0x00000730]> ?
Usage: [.][times][cmd][~grep][@[@iter]addr!size][|>pipe] ; ...
Append '?' to any char command to get detailed help
Prefix with number to repeat command N times (f.ex: 3x)
|%var =valueAlias for 'env' command
| *[?] off[=[0x]value]    Pointer read/write data/values (see ?v, wx, wv)
| (macro arg0 arg1)       Manage scripting macros
| .[?] [-|(m)|f|!sh|cmd]  Define macro or load r2, cparse or rlang file
| =[?] [cmd]              Send/Listen for Remote Commands (rap://, http://, <fd>)
| <[...]                  Push escaped string into the RCons.readChar buffer
| /[?]                    Search for bytes, regexps, patterns, ..
| ![?] [cmd]              Run given command as in system(3)
| #[?] !lang [..]         Hashbang to run an rlang script
| a[?]                    Analysis commands
| b[?]                    Display or change the block size
| c[?] [arg]              Compare block with given data
| C[?]                    Code metadata (comments, format, hints, ..)
| d[?]                    Debugger commands
| e[?] [a[=b]]            List/get/set config evaluable vars
| f[?] [name][sz][at]     Add flag at current address
| g[?] [arg]              Generate shellcodes with r_egg
| i[?] [file]             Get info about opened file from r_bin
| k[?] [sdb-query]        Run sdb-query. see k? for help, 'k *', 'k **' ...
| L[?] [-] [plugin]       list, unload load r2 plugins
| m[?]                    Mountpoints commands
| o[?] [file] ([offset])  Open file at optional address
| p[?] [len]              Print current block with format and length
| P[?]                    Project management utilities
| q[?] [ret]              Quit program with a return value
| r[?] [len]              Resize file
| s[?] [addr]             Seek to address (also for '0x', '0x1' == 's 0x1')
| S[?]                    Io section manipulation information
| t[?]                    Types, noreturn, signatures, C parser and more
| T[?] [-] [num|msg]      Text log utility
| u[?]                    uname/undo seek/write
| V                       Visual mode (V! = panels, VV = fcngraph, VVV = callgraph)
| w[?] [str]              Multiple write operations
| x[?] [len]              Alias for 'px' (print hexadecimal)
| y[?] [len] [[[@]addr    Yank/paste bytes from/to memory
| z[?]                    Zignatures management
| ?[??][expr]             Help or evaluate math expression
| ?$?                     Show available '$' variables and aliases
| ?@?                     Misc help for '@' (seek), '~' (grep) (see ~??)
| ?>?                     Output redirection
[0x00000730]>]]
```

This can look a little overwhelming at first, but after a while I promise you. It'll get a lot more easier and understandable.

# Identifying
The first thing we will do to be able to patch this binary is to identify where and which instruction to change. So start by auto analyze the binary.
```
[0x00000730]> aaaa
[x] Analyze all flags starting with sym. and entry0 (aa)
[x] Analyze len bytes of instructions for references (aar)
[x] Analyze function calls (aac)
[x] Emulate code to find computed references (aae)
[x] Analyze consecutive function (aat)
[x] Constructing a function name for fcn.* and sym.func.* functions (aan)
[x] Type matching analysis for all functions (afta)
[0x00000730]>
```
This already outputs all it's doing, but to check what every **a** is doing just type **aa?**.
A short explaination:
```
aa = analyze all (fcns + bbs)
aaa = same as 'aa' but autoname functions as well
aaaa = more experimental analysis, like 'Type matching analysis for all functions'
```
I have not dived into more specifically details and compared the differents between every **a**. I usually just do **aaaa**.

Now it's time to disassemble, a good point to start is always the main function. Because that's where the interesting execution starts. To disassemble main, we can first
start by seeking to that function and then print it out.
```
[0x00000730]> s main
[0x00000933]> pdf
            ;-- main:
┌ (fcn) sym.main 175
│   sym.main ();
│           ; var int local_30h @ rbp-0x30
│           ; var int local_24h @ rbp-0x24
│           ; var int local_20h @ rbp-0x20
│           ; DATA XREF from 0x0000074d (entry0)
│           ; STRING XREF from 0x0000074d (entry0)
│           0x00000933      55             push rbp
│           0x00000934      4889e5         mov rbp, rsp
│           0x00000937      4883ec30       sub rsp, 0x30               ; '0'
│           0x0000093b      897ddc         mov dword [local_24h], edi
│           0x0000093e      488975d0       mov qword [local_30h], rsi
│           0x00000942      837ddc02       cmp dword [local_24h], 2    ; [0x2:4]=0x102464c
│       ┌─< 0x00000946      7425           je 0x96d
│       │   0x00000948      488b45d0       mov rax, qword [local_30h]
│       │   0x0000094c      488b00         mov rax, qword [rax]
│       │   0x0000094f      4889c6         mov rsi, rax
│       │   0x00000952      488d3d1f0100.  lea rdi, str.Usage:__s__KEY ; 0xa78 ; "Usage: %s <KEY>\n" ; const char * format
│       │   0x00000959      b800000000     mov eax, 0
│       │   0x0000095e      e86dfdffff     call sym.imp.printf         ; int printf(const char *format)
│       │   0x00000963      bf01000000     mov edi, 1                  ; int status
│       │   0x00000968      e893fdffff     call sym.imp.exit           ; void exit(int status)
│       │   ; JMP XREF from 0x00000946 (sym.main)
│       └─> 0x0000096d      bf00000000     mov edi, 0                  ; time_t *timer
│           0x00000972      e879fdffff     call sym.imp.time           ; time_t time(time_t *timer)
│           0x00000977      89c7           mov edi, eax                ; int seed
│           0x00000979      e862fdffff     call sym.imp.srand          ; void srand(int seed)
│           0x0000097e      488d45e0       lea rax, [local_20h]
│           0x00000982      4889c7         mov rdi, rax
│           0x00000985      e8d6feffff     call sym.key_gen
│           0x0000098a      488b45d0       mov rax, qword [local_30h]
│           0x0000098e      4883c008       add rax, 8
│           0x00000992      488b00         mov rax, qword [rax]
│           0x00000995      488d4de0       lea rcx, [local_20h]
│           0x00000999      ba14000000     mov edx, 0x14               ; size_t n
│           0x0000099e      4889ce         mov rsi, rcx                ; const char * s2
│           0x000009a1      4889c7         mov rdi, rax                ; const char * s1
│           0x000009a4      e807fdffff     call sym.imp.strncmp        ; int strncmp(const char *s1, const char *s2, size_t n)
│           0x000009a9      85c0           test eax, eax
│       ┌─< 0x000009ab      7422           je 0x9cf
│       │   0x000009ad      488d45e0       lea rax, [local_20h]
│       │   0x000009b1      4889c6         mov rsi, rax
│       │   0x000009b4      488d3dce0000.  lea rdi, str.Wrong_key__Expected:__s ; 0xa89 ; "Wrong key! Expected: %s\n"
│       │   0x000009bb      b800000000     mov eax, 0
│       │   0x000009c0      e80bfdffff     call sym.imp.printf         ; int printf(const char *format)
│       │   0x000009c5      bf01000000     mov edi, 1
│       │   0x000009ca      e831fdffff     call sym.imp.exit           ; void exit(int status)
│       │   ; JMP XREF from 0x000009ab (sym.main)
│       └─> 0x000009cf      488d3dd20000.  lea rdi, str.Congratz__You_just_learned_patching ; 0xaa8 ; "Congratz! You just learned patching" ; const char * s
│           0x000009d6      e8e5fcffff     call sym.imp.puts           ; int puts(const char *s)
│           0x000009db      b800000000     mov eax, 0
│           0x000009e0      c9             leave
└           0x000009e1      c3             ret
[0x00000933]>
```
As you probably already can guess, the **s** command is for seeking to a function or address in the binary. Then 'pdf' **P**rints **D**issasembled **F**unction, as you can
see all the single character commands starts to make sense. If you don't want to seek to the function, just print a disassembled function, you can use **pdf @main**

We can see a call to time and srand at address **0x00000972** and **0x00000979** when you see those
two functions being called, you know there is some kind of random seed being used. And indeed it is,
It's used in the key_gen function called at **0x00000985**. That's why the key is changed every time
the program is being runned.

Generally when you are doing basic challenges, you want to look out for a specific jump where an
condition is met. In our case it's a **strncmp** which is a C function that suprisingly compares
two strings! Most built-in C functions like this has a very descriptional name but if you ever want to read more about a function all you need to do is open up a terminal on your favorite GNU/Linux
distro and type **man \<function\>** so for in this example: **man strncmp**.

# Analysis
For this challenge, the goal is clear. To print the string "Congratz! You just learned patching".
To achieve this we need to, as the title says, patch this binary. The first thing you need to do
is to make a copy of the file for obvious reasons:
```
$ cp patchme patchme.patch
```
then open the file in radare2 with the **-AAAw** flag.

**-AAA** - for analyze all and autoname functions (It's like the **aaa** command)

**-w** - Open in write mode
```
$ r2 -AAAw patchme.patch
```
We already printed the function above and if we take a closer look at the assembly we can see two
jump if equal(je) the first one at address **0x00000946**
```
[...]
│           0x00000942      837ddc02       cmp dword [local_24h], 2    ; [0x2:4]=0x102464c
│       ┌─< 0x00000946      7425           je 0x96d
│       │   0x00000948      488b45d0       mov rax, qword [local_30h]
│       │   0x0000094c      488b00         mov rax, qword [rax]
│       │   0x0000094f      4889c6         mov rsi, rax
│       │   0x00000952      488d3d1f0100.  lea rdi, str.Usage:__s__KEY ; 0xa78 ; "Usage: %s <KEY>\n" ; const char * format
│       │   0x00000959      b800000000     mov eax, 0
│       │   0x0000095e      e86dfdffff     call sym.imp.printf         ; int printf(const char *format)
│       │   0x00000963      bf01000000     mov edi, 1                  ; int status
│       │   0x00000968      e893fdffff     call sym.imp.exit           ; void exit(int status)
│       │   ; JMP XREF from 0x00000946 (sym.main)
│       └─> 0x0000096d      bf00000000     mov edi, 0                  ; time_t *timer
│           0x00000972      e879fdffff     call sym.imp.time           ; time_t time(time_t *timer)
[...]
```
We can get a good grasp of what this part does, just by looking at the string it prints. But to
analyze more, the first thing it does is a compare on a double word at location **local_24h** with
2\. From this information we can make the conclusion that **local_24h** is **argc** and that is being
compared to 2. So basically this part checks if the arguments specified by the user is 2, if it's
not it then prints the usage string with printf and exits with code 1.

Right before the exit call we see a:
```
mov edi, 1
call sym.imp.exit
```
This is because on 64bit binaries the registers is being used as parameters on the function in the
order: EDI, ESI, EDX etc.

As in 32bit binaries the arguments is being pushed onto the stack in opposite order. So First In
First Out (FIFO). The last argument you push onto the stack is the first argument in the function
and the first argument you push on the stack is the last argument in the function.

# Difference between 32bit and 64bit
We can easily test this with a simple function in C:
```
1  #include <stdio.h>
2  int main(void) {
3    int num0 = 0;
4    int num1 = 1;
5    int num2 = 2;
6    printf("Num0: %d, Num1: %d, Num2: %d", num0, num1, num2);
7
8  }
```
We just declare three integers and print them, a very basic program but perfect for learning. Lets
take a look at the assembly:
```
;64BIT
  [...]
1  mov DWORD PTR [rbp-4], 0 ; num0
2  mov DWORD PTR [rbp-8], 1 ; num1
3  mov DWORD PTR [rbp-12], 2 ; num2
4  mov ecx, DWORD PTR [rbp-12] ; num2
5  mov edx, DWORD PTR [rbp-8] ; num1
6  mov eax, DWORD PTR [rbp-4] ; num0
  [...]
8  call printf
```
We can see the resemblance very clear. In our C program line 3-5 is what we do in the assembly at
1-3. So **rbp-4** is num0 and so forth. But before we call the printf function, our integers is
being moved into the registers mentioned above in that order as well. Hopefully 64bit assembly is a
little bit easier to understand now.

Now onto 32bit:
```
;32BIT
  [...]
1  mov    DWORD PTR [ebp-0x14],0x0 ; num0
2  mov    DWORD PTR [ebp-0x10],0x1 ; num1
3  mov    DWORD PTR [ebp-0xc],0x2 ; num2
4  push   DWORD PTR [ebp-0xc] ; num2
5  push   DWORD PTR [ebp-0x10] ; num1
6  push   DWORD PTR [ebp-0x14] ; num0
  [...]
8  call printf
```
Now instead of moving the values into a specific order of registers the program just pushes the
values on the stack. With the num0 as the last push because it's being printed first.

# Winning
So now that we established that the first jump is a check if we specified two arguments or not, we
only have one jump left to analyze:
```
│           0x000009a4      e807fdffff     call sym.imp.strncmp        ; int strncmp(const char *s1, const char *s2, size_t n)
│           0x000009a9      85c0           test eax, eax
│       ┌─< 0x000009ab      7422           je 0x9cf
│       │   0x000009ad      488d45e0       lea rax, [local_20h]
│       │   0x000009b1      4889c6         mov rsi, rax
│       │   0x000009b4      488d3dce0000.  lea rdi, str.Wrong_key__Expected:__s ; 0xa89 ; "Wrong key! Expected: %s\n"
│       │   0x000009bb      b800000000     mov eax, 0
│       │   0x000009c0      e80bfdffff     call sym.imp.printf         ; int printf(const char *format)
│       │   0x000009c5      bf01000000     mov edi, 1
│       │   0x000009ca      e831fdffff     call sym.imp.exit           ; void exit(int status)
│       │   ; JMP XREF from 0x000009ab (sym.main)
│       └─> 0x000009cf      488d3dd20000.  lea rdi, str.Congratz__You_just_learned_patching ; 0xaa8 ; "Congratz! You just learned patching" ; const char * s
```
In this case and in many others we can learn so much from just looking at the strings being printed.
Again a strncmp is being called with what we know is a random key we need to specify. To bypass this
the easiest way is just changing the jump if equal to a constant jump so we win everytime! No matter
what we specify. To do this we first need to seek to the address we wish to change, in my case it's
in the address **0x000009ab** where the second jump is.
```
[0x00000933]> s 0x000009ab
[0x000009ab]>
```
Now with the command **wa** we can write opcode and change the je to a jmp. And as you can see, the
je is already pointing to the jump we need to take! So we want to use the same address:
```
[0x000009ab]> wa jmp 0x9cf
Written 2 byte(s) (jmp 0x9cf) = wx eb22
[0x000009ab]>
```
2 bytes written, nice! This means our binary is successfully patched. Why 2 bytes you wonder? That
is because the jump instruction takes up two bytes.

# Trying it out
Alright all that's left is to try it out, moment of truth..
```
$ ./patchme.patch TEST-KEY-THATS-WRONG
Congratz! You just learned patching
$
```
We got the success string to print and successfully learned basic patching with radare2


The sourcecode to the patchme challenge is available
[here](https://github.com/jlhk/patchme/blob/master/patchme.c) on my Github and in your cloned
patchme directory.

Thank you for reading!
