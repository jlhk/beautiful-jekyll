---
layout:     post
title:      "Cheatsheet for radare2"
subtitle:   "Cheatsheet and tricks for radare2, a reverse engineering framework"
date:       2018-04-16 11:00:00
categories: [reversing]
---
This is a collection of helpful commands in the framework radare2.
Mostly for personal use to not having to keep a cheatsheet of commands I find useful in my notes on my phone and to be able to access it anywhere.

But I'm sure it'll be useful for others. Will be adding more and more commands as I discover them.
So enjoy and good luck reversing!

# Variables
```bash
# Renaming a variable
> afvn <old_name> <new_name>
Example: > afvn local_111h counter

# Inspecting value at renamed variable
> .afvd <new_name>
Example: > .afvd counter
```

# Functions
```bash
# Rename the function currently in where name = the new name you want
> afn <name>

# Print pseudocode of the function
> pdc

# Print variables in current function
> afv
```

# Strings
```bash
# Print string found at location
> ps @ <location>

# Search for a string
> izzq~<string>

# Print every string found and the function it's found in
> axt @@ str.*
```
# Imports
```bash
# Prints imports
> ii
```

# Debugging
r2 -d \<file>
```bash
# Print registers
> dr=

# Fancy debugging visual mode
> V!
Shortcuts in the fancy debugging mode:
F9 - Run the program
F8 - Single-step over function
F7 - Single-step into function
```
# Configuration
~/.radare2rc
```bash
# Intel syntax
e asm.syntax=intel

# Description of every instruction (good if you are learning assembly)
e asm.describe=true

# Use colors
e scr.color=true

# Show comments to the right of the disassembled code if it fits
e asm.cmtright=true

# Solarized theme
eco solarized

# Use utf-8
e scr.utf8=true
```
