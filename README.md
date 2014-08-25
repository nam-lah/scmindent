# Editing Lisp and Scheme files in vi

> But then, when was the last time you heard
  of a lisp programmer that used vi?
  -- Paul Fox, vile developer

---------------------------------------------------

The ability to automatically indent code is indispensable when editing
files containing s-expression-based code such as Racket, Common Lisp, Scheme,
ART*Enterprise, and other Lisps.

The text editor family vi provides the option `lisp`, which in
conjunction with the options `autoindent` and `showmatch`
provides a form of Lisp indenting, but except in the improved vi clone
Vim, this support is poor in at least two respects:

1. escaped
parentheses and double-quotes are not treated correctly; and

2. all
head-words are
treated identically.

Even the redoubtable Vim, which has improved its Lisp editing
support over the years, and provides the `lispwords` option, continues to fail in
[strange ways](./vim-indent-error.lisp).

Fortunately, both vi and Vim let you delegate the responsibility for indenting such
code to an external filter program of your choosing.  I provide here two
such filtering scripts:
[scmindent.rkt](./scmindent.rkt) written in PLT Racket, and
[lispindent.lisp](./lispindent.lisp) written in Common Lisp.  The two scripts are
operationally identical: you can use either script to indent any Lisp.
Henceforth, I will refer to just `scmindent.rkt` with the understanding that
everything mentioned applies equally to `lispindent.lisp`.

`scmindent.rkt` takes
Lisp text from its standard input and produces an indented version
thereof on its standard output.  (Thus, it is possible to use
`scmindent.rkt` as a command-line filter to "beautify" Lisp code, even if
you don't use vi.)

Put `scmindent.rkt` in your `PATH`.

In Vim, set the `equalprg` option to the filter name, which causes the
indenting command `=` to invoke the filter rather than the built-in
indenter.

You might want to make the `equalprg` setting local to the filetypes
that need it:

```
autocmd bufread,bufnewfile *.lisp,*.scm setlocal equalprg=scmindent.rkt
```

In vi's other than Vim, use the `!` command to invoke the filter on part or all of
your buffer: Type `!` to declare you'll be filtering; a movement command
to scoop up the lines you'll be filtering; then the filter name
(`scmindent.rkt`) followed by `Return`.

## How subforms are indented

Lisp indentation has a tacit, widely accepted convention that is not
lightly to be messed with, so `scmindent.rkt` strives to provide the same
style as emacs, with the same type of customization.

By default, the indentation procedure treats
a form split over two or more lines as
follows.  (A form, if it is a list, is considered to have a head subform and zero or
more argument subforms.)

1.  If the head subform is followed by at
least one other subform on the same line, then subsequent lines in the
form are indented to line up directly under the first argument subform.

    ```
    (some-user-function-1 arg1
                          arg2
                          ...)
    ```

2. If the head subform is a list and is on a line by itself, then
subsequent lines in the form are indented to
line up directly under the head subform.

    ```
    ((some-user-function-2)
     arg1
     arg2
     ...)
    ```

3. If the head subform is a symbol and is on a line by itself, then
subsequent lines in the form are indented one column past the beginning
of the head symbol.

    ```
    (some-user-function-3
      arg1
      arg2
      ...)
    ```

4. If the head form can be deduced to be a literal, then subforms on
subsequent lines line up directly under it, e.g.

    ```
    (1 2 3
     4 5 6)

    '(alpha
      beta
      gamma)
    ```

## Keywords

However, some keyword symbols are treated differently.  Each such
keyword has a number N associated with it called its Lisp indent number,
which influences how its subforms are indented.  This is almost exactly
analogous to emacs's `lisp-indent-function`, except I'm using numbers
throughout.

If
the i'th argument subform starts
on a subsequent line, and i <= N, then it is indented 3 columns past the
keyword.  All subsequent
subforms are indented simply one column past the keyword.

```
(defun some-user-function-4 (x y)   ;defun is a 2-keyword
  body ...)

(defun some-user-function-5
    (x y)
  body ...)

(if test                            ;if is also a 2-keyword
    then-branch
  else-branch)
```

`scmindent.rkt` pre-sets the indent numbers of many well-known
Lisp keywords.  In addition, any symbol that starts with `def` and whose
indent number has not
been explicitly set is assumed to
have an indent number of 0.

## Customization

You can specify your own Lisp indent numbers for keywords in the file
`.lispwords` in your home directory.  `~/.lispwords` can contain any number of
lists: The first element of each list is the Lisp indent number that is
applied to the symbols in the rest of the list.  (Note that in contrast
to Vim's flat list of `lispwords`, `~/.lispwords`
allows for different categories of lispwords.  Vim's `lispwords` are
all of Lisp indent number 0.)

For example, a lot of users prefer the keyword `if` to have its then-
and else-clauses indented the same amount of 3 columns.  I.e.,
they want it to be a 3-keyword.  A `.lispwords` entry that would
secure this is:

```
(3 if)
```

To remove the keywordness of a symbol, you can assign it a Lisp indent
number < 0.  E.g.

```
(-1 if)
```

would also cause all of `if`'s subforms to be aligned.  (This is because
-1 causes subforms on subsequent lines to line up against the first
argument subform on the first line, and that happens to be 3 columns
past the beginning of a 2-column keyword like `if`.  The only difference
between -1 and 3 here is what happens when the `if` is on a line by
itself, with the test on the line following.  -1 indents subsequent
lines one column past the beginning of the `if`, whereas 3 continues to
indent them three columns past the beginning of the `if`.  Further
differences emerge between 3 and -1 when the `if` has more than three
argument subforms, as allowed by emacs lisp, where 2 and -1 immediately
prove to be better choices than 3.  The author has made 2 the default
because it is the only option that has the merit of indenting the then-
and else-subforms by differing amounts.)
