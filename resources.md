---
layout: default
title: "Resources"
category: "resources"
---

# x86-64 assembly language resources

* [Brown x64 cheat sheet](https://cs.brown.edu/courses/cs033/docs/guides/x64_cheatsheet.pdf)
* [Brown gdb cheat sheet](https://cs.brown.edu/courses/cs033/docs/guides/gdb.pdf)
* [CMU summary of gdb commands for x86-64](http://csapp.cs.cmu.edu/3e/docs/gdbnotes-x86-64.pdf)
* [JHU CSF x86-64 assembly language guide](https://jhucsf.github.io/csfdocs/assembly-tips-v0.1.1.pdf)

<!--
# Example code

## Precedence climbing example

[prec-climb.zip](resources/prec-climb.zip) is an example implementation of precedence
climbing. The `Parser::parse_work` function in `prec_climb.cpp` is the implementation
of the algorithm. It's written in a purely-recursive way, which might make it easier
to understand than the pseudo-code in the
[Wikipedia article on precedence climbing](https://en.wikipedia.org/wiki/Operator-precedence_parser).

Note that the lexer requires spaces between tokens. (E.g., `a+b` would be considered one
token, so you should type `a + b` instead.)

Example run (user input in **bold**):

<div class='highlighter-rouge'><pre>
$ <b>./prec_climb</b>
<b>a + b * 3 - 4 * 2 ^ b ^ 3</b>
OP_MINUS[-]
+--OP_PLUS[+]
|  +--IDENT[a]
|  +--OP_TIMES[*]
|     +--IDENT[b]
|     +--NUMBER[3]
+--OP_TIMES[*]
   +--NUMBER[4]
   +--OP_EXP[^]
      +--NUMBER[2]
      +--OP_EXP[^]
         +--IDENT[b]
         +--NUMBER[3]
</pre></div>

Please [let me know](mailto:daveho@cs.jhu.edu) if you find bugs in this code!
-->

# Exam review materials

Note that in Fall 2020 the exams were take-home exams, so the format is
different than what you will see when you take Exam 1 in class.
However, the questions should still be useful for review purposes.
The Fall 2021 exams were in-person, and the questions are more representative
of the kinds of questions you will see in this course's exams.

For Exam 1:

* [Fall 2020 Exam 1](resources/exam01-628.pdf), [Solution](resources/exam01_soln.pdf) (All questions are relevant)
* [Fall 2020 Exam 2](resources/exam02-628.pdf), [Solution](resources/exam02_soln.pdf) (Questions 1–5 are relevant)
* [Fall 2021 Exam 1](resources/exam01-fall2021.pdf), [Solution](resources/exam01-fall2021-solution.pdf) (All questions are relevant)

For Exam 2:

* [Fall 2020 Exam 2](resources/exam02-628.pdf), [Solution](resources/exam02_soln.pdf) (Question 6 is relevant)
* [Fall 2021 Exam 2](resources/exam02-fall2021.pdf), [Solution](resources/exam02-fall2021-solution.pdf) (Questions 3 and 4 are relevant)
* [Fall 2022 Exam 2](resources/exam02-fall2022.pdf), [Solution](resources/exam02-fall2022-solution.pdf) (Questions 3 and 4 are relevant)

<!--
We're not actually covering attribute grammars any more.

* [Fall 2020 Exam 3](resources/exam03-628.pdf), [Solution](resources/exam03_soln.pdf) (Question 1 is relevant)
-->

For Exam 3:

* [Fall 2020 Exam 3](resources/exam03-628.pdf), [Solution](resources/exam03_soln.pdf) (Question 2–3 are relevant)
* [Fall 2020 Exam 4](resources/exam04-628.pdf), [Solution](resources/exam04_soln.pdf) (Questions 1–4 are relevant)
* [Fall 2021 Exam 3](resources/exam03-fall2021.pdf), [Solution](resources/exam03-fall2021-solution.pdf) (All questions are relevant)
* [Fall 2022 Exam 3](resources/exam03-fall2022.pdf), [Solution](resources/exam03-fall2022-solution.pdf) (All questions are relevant)
