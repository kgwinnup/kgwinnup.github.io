---
layout: post
title:  "Exploring Radare2, unpacking self modifying code and binary patching"
date:   2017-04-27 23:59:59
categories: reversing
---

[Radare2](https://radare.org) is a project I've been watching with some interest for a while. Although, being a little removed from reversing for almost a year now, getting comfortable with Radare2 was never something on the top of my Todo stack. That has sinse changed and wow! Radare2 is awesome. I am still going through and figuring out all the commands and exploring capabilities, but so far I am very impressed.

This binary lab in particular is interesting because it contains self modifying code that makes a callback to RPI servers. The self modifying code is xor encoded and makes for a cool example to try out some of Radare2's features!

##### Downloads

You can download the [original binary](/assets/radare2emu/bomb) or the [patched binary](/assets/radare2emu/bomb.mod) with a callback to the 127.0.0.1.

- [Identifying and extracting self modifying code](#finding)
- [Unpacking with Radare2 Emulator](#radare2)
- [Patching with Radare2](#patching)

<a name="finding"></a>

## Identifying and extracting the self modifying code

When spinning this binary up, entering an incorrect answer triggers the `kaboom` function. A few instructions after, there is an call to a local stack variable which is the self modifying code section. 

Radare2 also comes with some cool utility applications. Rabin2 to get general information, symbols, strings, binary info, etc... Rax2 is handy too and there are several others. See the Radare2 book, obviously the book is extremely helpful while learning Radare.

### The screencast below uses the following commands

- `aaa` upon loading a binary, this command will run various analyses. use a? to see more options.
- `vv` will jump into visual mode with function menu, single `v` will drop you into visual mode without the menu at the current cursor location (prompt address)
- navigation is basic vi/vim movement keys which is awesome!
- `c` while in visual mode pressing `c` will drop the into the cursor mode instead of moving line by line.
- `p` while navigating around, pressing the `p` button in visual mode will toggle different views, hex, disassembly, debug, etc...
- `space` while in visual mode `space` will drop you into ascii graph view
- `s` seek cursor to address. There are a couple ways to do this, while in visual mode press o then the address to jump to that address (`u` brings you back)
- `;` this is a keybinding might be familiar, it will allow you to type comments

<asciinema-player src="/assets/radare2emu/screencast1.json" cols="120" rows="29"></asciinema-player>

Now we could emulate the code in place using the same method as you'll see below. Or we could isolate the binary code into its own file for further analysis. Extracting sections of code using Radare is done using the `wtf` command. Typing `wt?` will give you usage menu for these commands. 

We know that the self modifying code is 0x8c bytes long. And due to the `call 0x804a08a` instruction at 0x0804a09c, the next instruction address is pushed onto the stack. This value is then popped into eax which is used as part of the byte indexing loop to xor. 

Doing some basic math we can calculate the entire code block to be 0xa1 bytes long.

```config
0x0804a08c      eb0e           jmp 0x804a09c           
0x0804a08e      58             pop eax
0x0804a08f      31c9           xor ecx, ecx
0x0804a091      b18c           mov cl, 0x8c
0x0804a093      807401ff7f     xor byte [ecx + eax - 1], 0x7f
0x0804a098      e2f9           loop 0x804a093             
0x0804a09a      ffd0           call eax
0x0804a09c      e8edffffff     call 0x804a08e              
0x0804a0a1      b87b5b7d7f     mov eax, 0x7f7d5b7b
```

### Dumping the packed code to a file

This step is quite easy in Radare2. `wtf packed.bin 0xa1 @ 0x804a08c`. The code will dump 0xa1 bytes starting at 0x804a08c into the file packed.bin. This step is not necessary as you can do all the emulation and patching from the main binary.

<a name="radare2"></a>

## Unpacking with Radare2 Emulator

Now to decoding this blob of bytes in Radare. The emulator and the ESIL language built into Radare2 is definitely a cool feature. Radare2 also supports emulation with [Unicorn](https://github.com/radare/radare2-extras/tree/master/unicorn) although I have not tried it yet. 

Anyway, lets get to unpacking this binary. First, we needed to enable emulation mode with `aei`. Second, we'll initialize the stack. The stack in the decoding is only used for the value in `pop eax` to set part of the indexing calculation. `aeim 0x2000 0xffff` will set up our stack (these addresses are arbitrary just needed a range somewhere). Next, define where to stop the emulation. We know that once the code is unpacked it begins executing at the value in eax. So that seems like a reasonable place to stop the emulation. If you don't set a stop address, the emulator will execute continuously until you press ctrl-c. So executing and stopping after the encoder finishes can be done with `aecu 0x00000015` (you'll have to change the address if you are decoding in the original binary).

The sequence of commands is as follows:

- `aei` initialize ESIL VM state
- `aeim 0x2000 0xffff` initialize ESIL VM stack
- `e io.cache=true` this will ensure the self modification happening in memory can actually happen
- `aeip` set EIP to be at the cursor address
- `aecu 0x00000015` continue stepping through and emulating code until you reach this point. This is where the unpacking stops.

<asciinema-player src="/assets/radare2emu/screencast2.json" cols="120" rows="30"></asciinema-player>

<a name="patching"></a>

## Patching the binary

First step here is to calculate the new bytes that we'll use to replace with. These bytes need to be an IP address that when xor'd by 0x7f result in `127.0.0.1`. As mentioned above, `rax2` is another cool tool in the Radare suite ([link to book](https://radare.gitbooks.io/radare2book/content/introduction/rax2.html)). Once we get the bytes above with our loopback address, we then need to find where to place them. Below are the `rax2` commands for calculating the bytes we need.

We also know based on analyzing the decoded instructions that the address being pushed onto the stack for the socket creation is at 0x58 from the start. Give that address and what the bytes look like we should easily be able to find the values and replace them with our new ones.

```rax2 $((0x7f ^ `rax2 128`))```

```rax2 $((0x7f ^ `rax2 213`)```

```rax2 $((0x7f ^ `rax2 33`))```

```rax2 $((0x7f ^ `rax2 6`))```

```rax2 $((0x7f ^ `rax2 127`))```

```rax2 $((0x7f ^ `rax2 0`))```

```rax2 $((0x7f ^ `rax2 1`))```

Now that we've calucated our new bytes and figured out what the old encoded bytes look like, it is just a matter of swaping the new with the old. The following screencast shows how.

<asciinema-player src="/assets/radare2emu/screencast3.json" cols="120" rows="30"></asciinema-player>





