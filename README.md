# MBASIC-Protect
This is information on the CP/M MBASIC interpreter's protect mode and how to "get around" it.

Let me start with a spoiler alert. The source code for MBASIC 5.2 is available and, while I dug as deep as I could I did eventually consult the source code. This document is roughly in the order I "discovered" things and this is an interesting puzzle to look at.

Ulitmately, I will provide source code (in BASIC designed to be compiled with BASCOM) that will protect or unprotect any tolkenized MBASIC 5.2 program.

And also a disclaimer... This is done strickly for educational purposes. There was at least one utility already widely available that did unprotect files, there was no source code or documentation about how they went about it.

## UNPRO.COM
In looking through the Walnut Creek CP/M Repository, I found a couple of things.

The most useful was a file UNPRO.COM that, when ran, would unprotect a BASIC file that has been saved as protected. The program was 1) proof that there is a way to unprotect a protected file and 2) that this capability has been around for a while. (I don't feel that I'm compromising any deep secrets by looking at this around four decades after the interpreter was released. But this means I'm not fundamentally breaking something that hasn't been broken before. I also didn't realize that the source code for MBASIC itself was available as I started this.) Unfortunately, there is just a simple text file with the command showing its use and urging people not to unprotect anything they shouldn't. There is no source code and no author is listed. When run with no argument, you see the following:

```
A>unpro
UNPRO version 1.0 for MBASIC 5.21
USAGE:  unpro [d:]filename.ext
Unprotects and rewrites the file.
A>
```

## Pokes and Peeks
In the same area of the Walnut Creek Repository, I also found two files with other ideas on how to approach this.

One was a short BASIC program named UNPROTCT.BAS that POKEs a machine language program into memory and then CALLs it. It claimed that once you do this, you can load other protected programs and CALL the routine. While I never could use this method to open any other protected program, it is kind of interesting that it works and is probably worth digging into sometime. Simplifying things a bit, you can enter a program like this:
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
But I was still left wondering how UNPRO.COM worked. With help, I ultimately found the answer and will share the journey that got me there.

I created a program called PATTERN.BAS in a text editor. It was just a line number, a REMark statement, one space, and then a series of upper case As. I saved this both as a regular tolkenized file and also as a tolkenized file with protection. My goal was to make this so the lines would somewhat line up when looked at as a hex dump. One thing that was immediately obvious was:

  - The pattern is position dependent. A series of characters such as "AAAAA" would be encoded in the protected save as something like ```A6 69 65 6E 6B```.

I also tried changing just a single character which led me to find:

  - Changing any one byte in the program (whether a character or a tolken) changes only one byte in both the tolkenized save and protected save.

To do further checking, I used the file mentioned above. The file is in this repository with the ASCII source (PATTERN.BAS), the tolkenized format (PATTERN1.BAS), and the protected format (PATTERN2.BAS). I also created a hex dump of the protected save and looked for repetition, I found the following...

![Pattern in protected file...](https://github.com/w4jbm/MBASIC-Protect/raw/master/pattern.png)

From this you can see:

  - The encoded pattern in the protect file repeats every 8F (143 decimal) postions. (At least for text that is part of a REMark statement.)

It is interesting that 143 is both 12^2-1 and 11x13. At this point I was not sure either fact had any significance...

It also became apparent that MBASIC could, with some effort, act as a tool to build tables that would allow decryption. If you find locations in the series of 143 bytes that are within a REMark statement, you could write a program that would POKE that location with values from 00 to FF, save a protected copy of itself to a file, open that file, and retrieve the encoded value.

Put another way, I would need 143 indvidual tables, one for each possition in the repeatative sequence. Each table would map the possible values I might find in an encrypted file (from 00 to FF) to it's unencrypted value (also from 00 to FF). This set of tables would require close to 36K of memory, but it would be almost trivial to write a program to construct. At that size, it would not be practical to include the table as part of a CP/M command file.

Since I knew a much smaller program actually exists and works (the UNPRO.COM program discussed earlier) and since MBASIC itself does not need this type of table to encrype and decrypt a protected file, there must be a simpler equation that would allow you to encrypt things "on the fly".

At this point, I developed two additional working hypothesis:

  - The first byte of the program is not encrypted. It is simply a flag with a value of FF for an unencrypted program or FE for an encrypted program.
  - Given the speed and the 143 byte repetation rate, it seem likely that the encryption uses something along the lines of ```Encrypted Byte = [(Unencrypted Byte +/- Pre-Counter) XOR Key (for position 1 to 143)]``` or ```Encrypted Byte = [((Unencrypted Byte) XOR Key (for position 1 to 143)) +/- Post-Counter```.
  
If that second hypothesis was correct, I believed one counter (either the pre-counter or post-counter) ran from 11 to 1 (or 10 to 0) while the other counter ran from 13 to 1 (or 12 to 0). (Counting down to 1 and then using the zero flage to determine when the counter needed to be reset made the most sense to me.) Either counter could count up or count down as well as be added to or subtracted from the byte being encrypted. I was assuming the cycle of 143 bytes happened when the two counters return to being "in synch" every 11x13 or 143 counts.

The fact that there was a KEY for positions 1 to 143 turned out to be a workable hypothesis and overall this felt like something that could be "brute forced".

I used my earlier program (with the "As") and also generated a second program with a "message" that I saved in protected format. I loaded the protected copys of the known file and the unknown file into an array (and also created an area for the "decoded" value of the known file.) Then looking at an individual byte, I could do three things:

  - Subtract or add or do nothing with a "post-XOR" value from one of the two counters
  - XOR the value against the known value (in this case, the ASCII value of the character "A")
  - Subtract or add or do nohting with a "pre-XOR" value from one of the two counters

After these operations, I would have a "key" for that particular position in the 143 byte cycle and then I could apply that key (along with the pre and/or post operations) to the "test message" in my second BASIC program.

The proof-of-concept program I put together is here as BRUTEF.BAS. The data from two protected files are included as DATA statements.

There were several things to tinker with including the order of the counters; whether they counted up (increment), down (decrement), or maybe one of each; what their range of values were (did they start at 0 or 1 or something else?); and whether they were used a before (pre) or after (post) the XOR operation.

I tried a couple of combinations before hitting on the right one. In the process I found there is both a Pre and Post adder and that the counters did count down to 1 (with hitting zero likely causing the zero flag to reset the value).

Below is the output that shows the squence number (realative to where I started trying to decode a message), the PRE and POST add values (relative to the start of the file), the "known" message value and protected stream of that message, the KEY that was calculated for that position, the protected stream from the message I was trying to decode and finally the decrypted result (both as decimal and as a character):

```
LOAD "BRUTEF.BAS"
Ok
RUN
Load arrays and initialize data...

SEQ  PR PO   T1  P1  KEY   P2  T2
  1   7  5   65 166  155  219  84 T
  2   6  4   65 105   94   64 104 h
  3   5  3   65 101   94   65 101 e
  4   4  2   65 110   81   79  32  
  5   3  1   65 107   84   59 113 q
  6   2 11   65 210  248  150 117 u
  7   1 10   65 132   58   92 105 i
  8  13  9   65  52   31   82  99 c
  9  12  8   65 209  252  171 107 k
 10  11  7   65 152  167  185  32  
 11  10  6   65 115   90    8  98 b
 12   9  5   65 103   90   56 114 r
 13   8  4   65 202  255  156 111 o
 14   7  3   65 155  162  213 119 w
 15   6  2   65 204  241  155 110 n
 16   5  1   65  20   47   53  32  
 17   4 11   65 159  169  214 102 f
 18   3 10   65 235  223  189 111 o
 19   2  9   65  31   41  104 120 x
 20   1  8   65 200  128  167  32  
 21  13  7   65  88  101   63 106 j
 22  12  6   65 226  233  134 117 u
 23  11  5   65  28   33   72 109 m
 24  10  4   65 127   76   46 112 p
 25   9  3   65  94   99   66 101 e
 26   8  2   65 107   80   14 100 d
 27   7  1   65 234  211  203  32  
 28   6 11   65  61    9  107 111 o
 29   5 10   65 162  164  223 118 v
 30   4  9   65  78  120   34 101 e
 31   3  8   65  99  101   18 114 r
 32   2  7   65 115   83   84  32  
 33   1  6   65 219  149  236 116 t
 34  13  5   65 220  227  189 104 h
 35  12  4   65 206  255  170 101 e
 36  11  3   65  49   24   16  32  
 37  10  2   65 214  227  131 108 l
 38   9  1   65  43   18   75  97 a
 39   8 11   65 156  168  229 122 z
 40   7 10   65 108   88   52 121 y
 41   6  9   65 236  216  203  32  
 42   5  8   65  42   30   75  98 b
 43   4  7   65  70    2  115 114 r
 44   3  6   65  84  112   34 111 o
 45   2  5   65 239  213  165 119 w
 46   1  4   65 199  131  242 110 n
 47  13  3   65 241  218  204  32  
 48  12  2   65 103   80   10 100 d
 49  11  1   65  96  105   14 111 o
 50  10 11   65  55   27   81 103 g
 51   9 10   65 171  153  198  46 .

The quick brown fox jumped over the lazy brown dog.
Ok
```
The program I was trying to decrypt was just a REMark statement of "The quick brown fox jumped over the lazy brown dog." (I know, the fox is supposed to be red...)

Note: At this point you should be able to create your own file of the form ```10 REM Any comment up to 50 characters...```, save it as a protected (encrypted) file, use a utility like DUMP.COM to snag the contents, place these in the second set of data statements in BRUTEF.BAS, and decode the statement.

It did take some tinkering of the pointers, but I was able to use this as a foundation to deriving a list of the 143 individual keys and, at this point, I had enough information to unprotect a program.

The thing that still bothered me was that it seemed larger than the method UNPRO.COM was probably using (I didn't see the 143 position table in the dump of that program). Also, it didn't seem likely that Microsoft had just stuck a random 143 bytes into their code to act as a key.

My first thought was that this might be machine code from within MBASIC somewhere. Feeding it through a disassembler, that didn't seem to be the case. The other problem with that approach would be that it would mean that for the protect function to work on various version, a sizable chunk of code would have had to remained unchanged in MBASIC from version to version. (Remember earlier we couldn't even get a POKE that worked in V5.2 to work in V5.21--clear the code tended to shift around at least in spots.)

So the big spoiler alert... At this point I decided to look at the MBASIC 5.2 source code.

It turns out that there are actually two different look ups--one for each counter. The PRE add counter points to one of 13 values in SINCON (the sin lookup table) and the POST add counter points to one of 11 values in ATNCON (the atan look up table). So there is the PRE adder, an XOR from the ATNCON table (using POST as the index), an XOR from the SINCON table (using PRE as the index), and finally the POST addition that gives the final result.

Martin (from the cpm mailing list) came to my aid again by disassembling UNPRO.COM and verifying that it actually used these two seperate tables.

The source code also revealed that an interesting approach was used in MBASIC. If you SAVE with the ",P" option, the entire contents of the program space are encrypted in place, then saved to disk, and finaly unencrypted.

Knowing all of this this, it is possible to decryt with an 11 value table (the ATNCON table using the POST value as the pointer) and a 13 value table (the SINCON table using the PRE value as the pointer).

So we've gone from 36K with no intelligence (just a raw lookup based on position of a byte and its value) to 143 bytes with a single lookup table constructed using brute force, and finally to 24 (11+13) bytes of table space knowing that we actually just have a pair of tables creating values for XOR functions.

I planned to write a program testing all of this, but was sidetracked for a few weeks. During that time I came across a copy of an article entitled "A Chosen-Plaintext Attack on the Microsoft BASIC Protection" by Rene van den Assem and Willem-Jan van Elk from 1986. (I'm not sure of the actual publication the article originally appeared in.)

They do a much better job of figuring out what is going on using some rigour at least to the point of being able to come up with an elegant way of creating the 143 byte key table for decryption. What they do is pure decription and they don't disassemble the source code or do anything else of that nature. (And, realistically, if you were simply interested in unprotecting something it probably isn't worth the extra effort of trying to understand where the keys come from.)

But having successfully "brute forced" the 143 keys, I was curious if you might be able to "brute force" identifying the location of the SINCON and ATNCON tables without any previous knowledge of where they were or that they were being used.

The answer is "yes". Taking the tables created by van den Assem and van Elk's approach, you can find the table by looking for three consecutive locations in two different locations that XOR to the first three values in their larger table. How to find these tables is shown in FNDTBLE.BAS. This program takes a very, very long time to crank through every pair of locations, do an XOR and then decide if it is worth doing a second, a third, and eventually a fourth XORs.

The results look like this:

```
RUN
 14612         14691 
 14691         14612 
Ok
```
Testing any fewer than four positions of the table yields some spurious results. Also, I have no doubt that if van den Assem and van Elk had seen the results, they would have immediately recognized the significance and been able to refine their program.

I used my previous UNIX2CPM.BAS program as the basis for handlign the filename on the command line and then van dem Assem and van Elk's KRAK.BAS program (modified to use the two tables that are known instead of having to create a 143 key table using yet another program) for the actual code to protect or unprotect a file. (I had started to code something up before this but had issues that their opening files for 1 byte records resolved.) The result is UNPRO2.BAS with is compiled into UNPRO2.COM. This command will protect or unprotect an MBASIC tolkenized file.

## And the fine print...

These programs are intended for personal use only. Any material will be removed at the request of the copyright holder or those holding other rights to it.

To the extent applicable, any original work herein is:

Copyright 2020 by James McClanahan and made available under the terms of The MIT License.

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
