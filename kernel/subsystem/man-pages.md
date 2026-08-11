# Introduction
Manual pages are the canonical type of documentation for Unix and Lustre.

## Conventions in manual pages

The formatted page follows manpage typesetting and other conventions, so that readers can efficiently extract the needed information from them. This bit is important: manpages are reference documentation, intended to quickly answer questions like "what is the purpose of this command" or "is there an option to show more information about files".

## Some of the more important conventions:

- the page should be short and to the point, without sacrificing clarity
- use only the usual sections, in the usual order; see `man-pages(7)` for details
- in particular, they should contain a brief, readable summary of command line syntax in `SYNOPSIS`; if there are a lot of options, list them in the `OPTIONS` section
- format the `OPTIONS` section so that options are in bold and their optional arguments are in italic, to improve fast skimming of the page
- for anything that is complicated to use, add an `EXAMPLE` section
-- add examples for the common cases, not just special ones: you may think your program is obvious to use, but not everyone will find it so
- don't forget `ENVIRONMENT` and `FILES` sections for commands that need them
- the `SEE ALSO` section is almost always useful to add at the end

## The title: .TH

Every manual page should start by specifying its title and section number:
```
.TH TITLE 1
```
You can additionally add three more pieces of information: the date of this revision of the manual page, where the program it documents came from, and the title of the whole book to which this page belongs to. The date should be relatively uptodate (within a few days of when the patch was last updated).

## NAME section
```
.SH NAME
```
The `NAME` section declares the name of command that is being documented. It also gives a very brief explanation of what it does (under 60 characters). These two parts are separated by backslash-dash. That's a magic combination, the man command requires it.

## SYNOPSIS section
```
.SH SYNOPSIS
```
This section gives the user a summary of how the command line syntax of the program looks like. Font usage is important here, and carries information. All the parts that are in bold are things that the user is expected to write verbatim. Italic UPPERCASE indicates values the user is expected to replace. Normal font is used for syntax meta-characters: for the brackets that indicate optionality, and the ellipsis that indicates repetition.

### Using fonts
Fonts can be set in two ways: either using the dot-commands (preferred), or the backslash-f escapes. The `.B` command typesets the rest of the line in bold face. Similarly, `.I` typesets in italics (but terminals show that as underline), and `.R` in what troff calls the roman font, and the rest of us call the normal font. You can combine these, `.BR` typesets the first word on the line in bold, the second in normal font. The output will have no space between the words: `.BR manpagename (7)` would be the usual way to refer to another manual page. If a word has spaces in it, use double quotes: `.B "far and away"` for example.

Backslash-f escapes (`\fB`, `\fI`, `\fR`) work anywhere on a line, and in some cases they're easier to use than the dot-commands. Their effect also does not end at the end of a line.

Dashes in options should be prefixed by backslashes. A naked `-` means a hyphen, whereas `\-` means a minus sign. Typographically these are distinct, and they are also distinct in Unicode. The typesetter is free to break a line at a hyphen, but not at a minus. For dashes in options, you should thus use minuses, but in normal text, for normal words, the hyphen.

## DESCRIPTION section
```
.SH DESCRIPTION
```
The `DESCRIPTION` section describes what the program does, in more detail than the `NAME` section. There are no artificial size limits here, but it's still good style to avoid being long-winded. At the same time, it is perhaps best to not be quite as terse as the example.

### Paragraphs and line structure

If you write more than one paragraph, start the other paragraphs with the `.PP` command. Do not just leave an empty line; this makes troff sometimes do the wrong thing. In fact, the manual page source should have no empty lines at all.

troff prefers you to start every sentence on a new line. This lets it typeset end-of-sentence whitespace better, when it produces output using proportional fonts. Also, this makes it easier to compare versions of a manpage with diff.

### More about fonts

There are some more font conventions.

- use bold for the command you are documenting
- refer to other manual pages like this: `man-pages(7)` (easiest to achieve like this: `.BR man-pages (7))`
- variable arguments in CAPITALS and italics
- filenames in italics

See `man-pages(7)` for more details.

## OPTIONS section
```
.SH OPTIONS
.TP
.BR \-n ", " \-\-number =\fIVALUE\fR
```
Options are perhaps the most tedious part to document. It is also the part where proper formatting gives the most benefit to the reader.

The list structure is achieved with the `.TP` command takes the next line, and does not indent it, and then indents the rest of the paragraph.

The un-indented line describes the option, giving its name, and indicates if it gets an argument or not. If there are several names for an option (a long one, and a one-letter one, for example), they should be on the same line, separated by commas. The one-letter option name does not get the option argument, to keep things short.

Fonts are again used to clarify things:

- the name of the option (including any dashes at the beginning) is in bold
- arguments in UPPERCASE in italic, but the equals sign (if any) is normal font
- the comma between options is in normal font

### Comments in troff source

Comments start a line with `.\"`.

## Marking up examples
```
.SH EXAMPLES
Example usage of command:
.RS
.EX
.B command with options
output of command
.EE
.RE
```

Most manual pages benefit from an `EXAMPLES` section, which shows basic use of the command.

Marking up examples of command line use should have a textual description of the example in regular font, use the `.RS` and `.EX` macros to start the example with the command-line or other literal text in bold, output in regular font, then end the example with `.EE` and `.RE`.

### A few new troff commands:

- `.nf` turns off paragraph filling mode: we don't want that for showing command lines.
- `.fi` turns it back on.
- `.RS1 starts a relative margin indent: examples are more visually distinguishable if they're indented.
- `.RE` ends the indent.
- `\\` puts a backslash in the output.
