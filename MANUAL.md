# Fl_Highlight_Editor manual

## Table of contents

* [Introduction](#introduction)
* [Initializing widget and interpreter](#initializing-widget-and-interpreter)
* [Some obligatory terms](#some-obligatory-terms)
* [Adding your own mode](#adding-your-own-mode)
  * [Mode and rule details](#mode-and-rule-details)
* [Adding new faces](#adding-new-faces)

## Introduction

This manual will try to give you some introduction on how to use
Fl_Highlight_Editor widget and how to extend the builtin language with
own constructs, modes and highlight styles. No previous Emacs
knowlege is assumed (e.g. you don't have to know what are faces or
styles) but basic Scheme, C++ and FLTK knowledge is required.

If you are not familiar with the Scheme language, please read
[Yet Another Scheme Tutorial](http://www.shido.info/lisp/idx_scm_e.html)
or any other online tutorial you find suitable.

## Initializing widget and interpreter

Here is example on how to initialize Fl_Highlight_Editor, load
interpreter and read some file:

```cpp
#include <FL/Fl.H>
#include <FL/Fl_Double_Window.H>
#include <FL/Fl_Text_Buffer.H>

/* our library */
#include "FL/Fl_Highlight_Editor.H"

int main(int argc, char **argv) {
  /* create initial window */
  Fl_Double_Window *win = new Fl_Double_Window(400, 400, "Test #1");
  win->begin();
    /* initialize editor object */
    Fl_Highlight_Editor *editor = new Fl_Highlight_Editor(10, 10, 380, 350);

    /*
     * Initialize interpreter by pointing it to folder where are
     * Scheme source files for Fl_Highlight_Editor. These files
     * implements modes, syntax higlighting logic and many more.
     *
     * Unless you initialize interpreter or point it to folder where
     * are not Scheme source files, Fl_Highlight_Editor will behave
     * just like Fl_Text_Editor: it will display text allowing you
     * basic operation on that text.
	 *
	 * You should probably initialize interpreter as soon as possible so
	 * interpreter can run hooks when file is loaded.
     */
    editor->init_interpreter("./scheme");

    /* load some example */
	editor->loadfile("test/example.cxx");
  win->end();

  /* show the window and enter event loop */
  win->show(argc, argv);
  return Fl::run();
}
```

## Some obligatory terms

Before we continue explaining widget details and internals, let we
describe some terms used in this manual. If you used Emacs before, you
will find these terms quite familiar.

### mode

Mode is state of editor loaded on some action; in our case mode
represents state of widget editing capabilities. For example, mode can
alter default editor behavior (e.g. changing keys, replacing tab
character with spaces), load syntax highlighting and etc.

Comparing to Emacs modes where each mode can change almost any editor
option, Fl_Highlight_Editor modes are more limited.

### face

Face is detail how to draw text in widget or parts of text. Each face
has name, font name, type, size and color. For now, face does not have
option to set background color.

### context

Context is set of expressions used to find certain words or chunk of
text to be painted with given face (or highlighted). For example,
context can be regular expression or tokens that will designate start
and end of the block.

### hook

Hook is a function (Scheme function) called when something
happens. For example, when you a load file (via ''loadfile()'' member),
a hook will be called; the same applies when you save file with ''savefile()''.

Hooks represents efficient way to extend or alter widget behavior,
without writing C++ code.

## Adding your own mode

All modes should exist in ''FOLDER/modes'', where FOLDER is location
you loaded with C++ function 'init_interpreter(FOLDER)'. Default name,
distributed with this library is *scheme*, but you can change it to
suit your needs.

Mode files should be name as ''MODE-mode.ss'', where MODE is name of
your mode file, e.g. `make-mode.ss`. Let we write make-mode, showing
details behind it (Fl_Highlight_Editor comes with make-mode, but for
the sake of this example, we will assume writing it from scratch).

In editor open file ''FOLDER/modes/make-mode.ss'' and write:

```scheme
(define-mode make-mode
  "Mode for editing Make files."
  (syn 'default #f 'default-face)

  (syn 'eol   "#" 'comment-face)

  ;; make rule syntax
  (syn 'regex "^[^:]*:" 'keyword-face)

  ;; make keywords
  (syn 'regex "^\\s*(ifeq|endif|include|echo)\\s+" 'keyword-face)

  ;; magic variable
  (syn 'regex "\\$[@<#\\^]" 'keyword-face)

  ;; user written variables
  (syn 'regex "\\$\\(\\w+\\)" 'type-face)

  ;; @echo and etc
  (syn 'regex "@\\w+" 'keyword-face)

  ;; external command in form $(command params)
  (syn 'regex "\\$\\([a-z]+\\s+[^\\)]*\\)" 'keyword-face))

;; now to invoke this mode on certain file types, you place
(add-to-list-once! *editor-auto-mode-table* '("([mM]akefile|\\.make)$" . make-mode))
```

First we are starting with `define-mode` which accept mode name and
documentation string, explaining what mode does and details. For now
documentation handling is not implemented, but this feature is planned
in future releases. Next we are adding `rules`, marked with
*(syn ...)* construct.

Before going into details about `rules`, let quickly describe a line:

```scheme
(add-to-list-once! *editor-auto-mode-table* '("([mM]akefile|\\.make)$" . make-mode))
```

Every mode you create should be invoked on some action and in this
case the action is matching against file
name. `*editor-auto-mode-table*` is Fl_Highlight_Editor global
variable containing mappings of filenames (or regular expressions
matching against loaded file path) and modes: when loaded path has
successful match, associated mode will be loaded.

Details about how this works check in
[Mode and rule details](#mode-and-rule-details).

Now, lets get back to rules: rules will tell to Fl_Highlight_Editor
what to look for in text and with what face to paint it. There are a
couple of builtin rule types:

#### default

Default face for all unmatched (default) rules. This face uses only
face name, without matching details so form:

```scheme
(syn 'default #f 'default-face)
```

should be set first.

#### regex

regex will tell matcher to use regular expressions (GNU style) to look
for text parts. For now, regular expression implementation doesn't
handle grouping.

To use it, you will say something like this:

```scheme
(syn 'regex "\\$[@<#\\^]" FACE-NAME)
```

where `FACE-NAME` is one of known (or yet unknown) faces. Note that
for escaping, you will need to use **double** backslashes; one will be
seen by interpreter as escaping character and one by regular
expression engine.

#### eol

This is `end of line` matcher: it will search for exact string token
and paint everything up to the end of the line. It is primarly used
for comments or headings, where full line can be painted with single
or multiple characters.

For example, to paint line with C++-like comments, you can use:

```scheme
(syn 'eol "//" 'comment-face)
```

String used for matching is exact string and matcher will search for
first occurence in the line. Given above rule, it is easily applied on
cases like:

```cpp
//////// this is some c++ comment /////////
```

#### block

`block` should target code blocks, which usually has staring and
ending markers. This option also supports blocks with the same
starting and ending markers.

`block` accepts quoted pair, where first element is staring marker
should be looked for and last element is ending marker. For example:

```scheme
(syn 'block '("<!--" . "-->") 'comment-face)
```

In Scheme parlance, constructs like `'(X . Y)` are pairs and **are
not** the same as `'(X Y)` so please be careful about it.

This rule will behave similarily to `eol`: it will search first
occurence of starting block marker, then will search first occurence
of ending block marker and so on. With this beavior, things like:

```html
<!--
  this is multiline
  comment with <!-- YIKES
  -->
```

should work as expected.

#### exact

This rule type if for search exact strings inside text. Let say you
are going to highlight all *FIXME:* words; to do that, you can use:

```scheme
(syn 'exact "FIXME:" 'important-face)
```

### Mode and rule details

This part will quickly explain internals and caveats about modes and
rules.

Fl_Highlight_Editor use hooks to determine when to load a mode; two of
them are important:

* `*editor-before-loadfile-hook*` - called before the file was loaded in
  widget
* `*editor-after-loadfile-hook*` - called after the file content was put
  in widget

`*editor-before-loadfile-hook*` will call
`(editor-try-load-mode-by-filename)` function, which will try to match
(using regular expressions) loaded filename against
`*editor-auto-mode-table*` mapping table; if match was found,
associated mode will be loaded.

This means on each file load above logic will be repeated, which is
intentional: idea is to have ability to reload modes in runtime so
they can be changed without deleting the widget.

Interpreter is involved only during mode loading and constructing
matching rules; from there C++ code (including FLTK paint engine) will
continue the work using compiled expression, which will keep
highlighting fast.

#### Rule executing order

Rules inside `define-mode` are executed *in order*, from top to bottom
and that can affect painting strategy. If used smartly, can do things
that would require much more complex stuff in backend.

## Adding new faces

Fl_Highlight_Editor organize faces in much simpler manner than
modes. Faces are stored in global variable `*editor-face-table*` and
are in form of list of vectors. They can look like this:

```scheme
;; this is part of (default-lite-theme)
(set! *editor-face-table*
  (list
    (vector 'default-face FL_BLACK 12 FL_HELVETICA)
    (vector 'comment-face FL_BLUE  12 FL_HELVETICA)
    (vector 'keyword-face FL_BLUE  12 FL_HELVETICA)
    (vector 'important-face FL_BLUE 12 FL_HELVETICA_BOLD)
    (vector 'type-face FL_DARK_GREEN 12 FL_HELVETICA)
    (vector 'attribute-face FL_DARK_CYAN 12 FL_HELVETICA)
    (vector 'string-face FL_DARK_RED 12 FL_HELVETICA)))
```

Each vector is in form:

```scheme
(vector FACE-NAME FLTK-COLOR-CODE FONT-SIZE FLTK-FONT-CODE)
```

You can freely mix FLTK color/font codes (they are defined in
`constants.ss` file) or their equivalent number values.

Face names, unlike in Emacs, are plain symbols and they can be
anything; Fl_Highlight_Editor will just use them as keys to find
appropriate color/size/font triplets. To add new face, you can use
this snippet:

```scheme
(add-to-list-once! *editor-face-table* (vector 'my-cool-face FL_RED 12 FL_HELVETICA))
```
