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
