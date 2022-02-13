---
layout: post
title: using regular expressions
---


Regular expressions are very handy to search for certain strings.
Note that there are different versions of grep.

source: https://unix.stackexchange.com/questions/498925/grep-not-working-as-expected

GNU grep uses by default Basic Regular Expressions (BRE), but it also let you use Extended Regular Expressions (ERE) and Perl-compatible Regular Expressions (PCRE).

Please note that neither BRE nor ERE support \s nor \d, but they have similar feature

To be able to use for instance `\t` (tab character) you have to invoke `grep -E` as in 

`grep -P "^[ \t]*print" test.txt`

This searches for
* start of line (^)
* followed by zero or more [space or tab]
* followed by the word 'print'


