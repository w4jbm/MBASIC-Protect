# MBASIC-Protect
This is information on the CP/M MBASIC interpreter's protect mode and how to "get around" it.

This is done strickly for educational purposes. There is at least one utility already widely available that does this, but there is no source code or documentation about how they go about it.

## UNPRO.COM
In looking through the Walnut Creek CP/M Repository, I found a couple of things.

The most useful was a file UNPRO.COM that, when ran, would unprotect a BASIC file that has been saved as protected. The program is 1) proof that there is a way to unprotect a protected file and that 2) this capability has been around for a while. (I don't feel that I'm compromising any deep secrets by looking at this around four decades after the interpreter was released. But this means I'm not fundamentally breaking something that hasn't been broken before.) Unfortunately, there is just a simple text file with the command showing it's use and urging people not to unprotect anything they shouldn't. There is no source code and no author is listed. When run with no argument, you see the following:

```
A>unpro
UNPRO version 1.0 for MBASIC 5.21
USAGE:  unpro [d:]filename.ext
Unprotects and rewrites the file.
A>
```

## Pokes and Peeks
In the same area of the Walnut Creek Repository, I also found two files with other ideas on how to approach this.

One is a short BASIC program named UNPROTCT.BAS that will poke a machine language program into memory and then CALL it. It claims that once you do this, you can load other protected programs and CALL the routine. I could never make that work. Simplifying things a bit, you can enter a program like this:

```
10 POKE 2051,49:POKE 3052,0
```
If you save this in protected mode and then reload it, you find that you cannot list it. But, if you run it, you can now list the program and save an unprotected copy of it. If you are running an emulator and can tinker with memory locations on the fly, this is likely the easiest way to unprotect a program. (I have not tried this yet, so your mileage may vary with that idea...)

The is a second file named MBASIC-P.DOC that suggests that if you ```POKE 23899,175``` before loading your protected program in MBASIC 5.2, you will be able to list or save the program. I could _not_ make this work in MBASIC 5.21 and was not able to find a copy of MBASIC 5.2 to try it with. Martin in the Google comp.os.cpm had better luck and the rest of this section is from information in his reply.

The source for MBASIC 5.2 is available and if you look at the area impacted by the POKE, you find this in FIVDSK.MAC:
```
PROCHK: PUSH PSW   ;Save flags
        LDA PROFLG ;Is this a protected file?
        ORA A      ;Set CC's
        JNZ FCERR  ;Yes, give error
        POP PSW    ;Restore flags
        RET
```
So ```POKE 23899,175``` effectively replaces ```ORA A``` with ```XRA A```.

Going a step further, he found where the same routine in MBASIC 5.21 and disassembled it.
```
5d65 f5 PUSH AF
5d66 3a ec 0b LD A,(0bec)
5d69 b7 OR A
5d6a c2 18 14 JP NZ,1418
5d6d f1 POP AF
5d6e c9 RET
```
With things in a slightly different location, under MBASIC 5.21 the command ```POKE 23913,175``` from the comand prompt before loading a protected file will let you list and save the file after loading it.


## Analysis
I created a program called PATTERN.BAS in a text editor. It has a line number, a REMark statement, one space, and then a series of upper case As. I saved this as a regular tolkenized file and with protection. I had tried to make this so the lines would somewhat line up when looked at as a hex dump. One thing that was immediately obvious was:

  - The pattern is position dependent. A series of characters such as "AAAAA" would be encoded in the protected save as something like ```A6 69 65 6E 6B```.

I also tried changing just a single character which led me to find:

  - Changing any one byte in the program (whether a character or a tolken) changes only one byte in both the tolkenized save and protected save.

When I looked at the protected save and looked for repetition, I found the following...

![Pattern in protected file...](https://github.com/w4jbm/MBASIC-Protect/raw/master/pattern.png)

From this you can see:

  - The ecoded pattern in the protect file repeats every 8F (143 decimal) postions. (At least for text that is part of a REMark statement.)

It is interesting that 143 is both 12^2-1 and 11x13. At this point I'm not sure either fact has any significance...
