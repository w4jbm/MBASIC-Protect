# MBASIC-Protect
This is information on the CP/M MBASIC interpreter's protect mode and how to "get around" it.

Let me start with a spoiler alert. The source code for MBASIC 5.2 is available and, while I dug as deep as I could I did eventually consult the source code. This document is roughly in the order I "discovered" things and this is an interesting puzzle to look at.

And also a disclaimer... This is done strickly for educational purposes. There was at least one utility already widely available that did unprotect files, there was no source code or documentation about how they went about it.

## UNPRO.COM
In looking through the Walnut Creek CP/M Repository, I found a couple of things.

The most useful was a file UNPRO.COM that, when ran, would unprotect a BASIC file that has been saved as protected. The program is 1) proof that there is a way to unprotect a protected file and that 2) this capability has been around for a while. (I don't feel that I'm compromising any deep secrets by looking at this around four decades after the interpreter was released. But this means I'm not fundamentally breaking something that hasn't been broken before.) Unfortunately, there is just a simple text file with the command showing its use and urging people not to unprotect anything they shouldn't. There is no source code and no author is listed. When run with no argument, you see the following:

```
A>unpro
UNPRO version 1.0 for MBASIC 5.21
USAGE:  unpro [d:]filename.ext
Unprotects and rewrites the file.
A>
```

## Pokes and Peeks
In the same area of the Walnut Creek Repository, I also found two files with other ideas on how to approach this.

One was a short BASIC program named UNPROTCT.BAS that POKEs a machine language program into memory and then CALLs it. It claims that once you do this, you can load other protected programs and CALL the routine. While I never could use this method to open any other protected program, it is kind of interesting that it works and probably worth digging into sometime. Simplifying things a bit, you can enter a program like this:
```
10 POKE 2051,49:POKE 3052,0
```
If you save this in protected mode and then reload it, you find that you cannot list it. But, if you run it, you can now list the program and save an unprotected copy of it. If you are running an emulator and can tinker with memory locations on the fly, this would provide a possilbe way to unprotect a program. (I have not tried this yet, so your mileage may vary with that idea...)

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

Going a step further, Martin found the same routine in MBASIC 5.21 and disassembled it.
```
5d65 f5       PUSH AF
5d66 3a ec 0b LD A,(0bec)
5d69 b7       OR A
5d6a c2 18 14 JP NZ,1418
5d6d f1       POP AF
5d6e c9       RET
```
With things in a slightly different location, under MBASIC 5.21 the command ```POKE 23913,175``` from the comand prompt before loading a protected file will let you list and save the file after loading it.

## Analysis
But I was still left wondering how DEPRO.COM worked. I think I found the answer and will share the journey to get there.

I created a program called PATTERN.BAS in a text editor. It was just a line number, a REMark statement, one space, and then a series of upper case As. I saved this both as a regular tolkenized file and also as a tolkenized file with protection. My goal was to make this so the lines would somewhat line up when looked at as a hex dump. One thing that was immediately obvious was:

  - The pattern is position dependent. A series of characters such as "AAAAA" would be encoded in the protected save as something like ```A6 69 65 6E 6B```.

I also tried changing just a single character which led me to find:

  - Changing any one byte in the program (whether a character or a tolken) changes only one byte in both the tolkenized save and protected save.

To do further checking, I used the file mentioned above. The file is in this repository with the ASCII source, the tolkenized format, and the protected format. I also created a hex dump of the protected save and looked for repetition, I found the following...

![Pattern in protected file...](https://github.com/w4jbm/MBASIC-Protect/raw/master/pattern.png)

From this you can see:

  - The ecoded pattern in the protect file repeats every 8F (143 decimal) postions. (At least for text that is part of a REMark statement.)

It is interesting that 143 is both 12^2-1 and 11x13. At this point I'm not sure either fact has any significance...

At this point, the fact that MBASIC can act as a tool to build tables that would allow decryption is apparent. If I find locations in the series of 143 that are within a REMark statement, I could write a program that would POKE that location with values from 00 to FF, save a protected copy of the source, open that file, and retrieve the encoded value.

Put another way, I would need 143 indvidual tables, one for each possition in the repeatative sequence. Each table would map the possible values I might find in an encrypted file (from 00 to FF) to it's unencrypted value (also from 00 to FF). This set of tables would require close to 36K of memory, but would be almost trivial to write a program to construct. At that size, it would not be practical to include the table as part of a CP/M command file.

Since I know a much smaller program actually exists and works (the UNPRO.COM program discussed earlier) and since MBASIC itself does not need this type of table to encrype and decrypt a protected file, there must be a simpler equation that would allow you to encrypt "on the fly".

At this point, I have two additional working hypothesis:

  - The first byte of the program is not encrypted. It is simply a flag with a value of FF for an unencrypted program or FE for an encrypted program.
  - Given the speed and the 143 byte repetation rate, it seem likely that the encryption uses something along the lines of ```Encrypted Byte = [(Unencrypted Byte +/- Pre-Counter) XOR Key (for position 1 to 143)] +/- Post-Counter```.
  
If that second hypothesis is correct, I believe one counter (either the pre-counter or post-counter) runs from 0 to 10 or 1 to 11 while the other counter runs from 0 to 12 or 1 to 13. Either counter could count up or count down as well as be added to or subtracted from the byte being encrypted. I'm assuming the cycle of 143 bytes happens when the two counters return to being "in synch" every 11x13 or 143 counts.

This feels like something that can be "brute forced" somehow. I can generate a file and save both an unprotected copy and a protected copy. Looking at an individual byte, I could subtract (or add) the post-XOR amount from the encrypted value to reverse that part of the operation and also add (or subtract) the pre-XOR amount to the unencrypted value to "anticipate" that part of the operation. That would give me the before and after value of the byte as it goes through the XOR operation. Knowing that, I should be able to determine the value for the "key" for that position.


