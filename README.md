> **Refactoring** (noun): a change made to the internal structure of software to
> make it easier to understand and cheaper to modify without changing its
> observable behavior.
>
> **Refactoring** (verb): to restructure software by applying a series of
> refactorings without changing its observable behavior.\
> --Martin Fowler, *Refactoring* [¹](#refactoring-footnote)

This file is meant to be a collection of techniques and hacks I've developed to
help me refactor Common Lisp code without so much manual editing.

It's **very hacky**. However, there's code and video for each refactor type except "Add File To System", "Mass Rename", and "Inline Function". Those do have
discussion, however (and mass-rename is doable with vim-grepper), and there's some semi-rants spread throughout. ;)

If you're a vimscript master and want to share an improvement to something here,
or send me an implementation for something not filled in yet, feel free, I'm
very much not a master.

If you just want the code for everything (which may or may not be more up to date than what's in this document) then see [here](https://github.com/Jach/dots/blob/master/.vimrc-lisp-refactoring)

## ToC

* [More Preface](#more-preface)
* [Template](#template)
---
* [Common Functions](#common-functions)
---
* [Export Symbol](#export-symbol)
* [Inline and Import Symbol](#inline-and-import-symbol)
* [Uninline Symbol](#ninline-symbol)
* [Add File To System](#add-file-to-system)
* [Mass Rename](#mass-rename)
* [Symbol To String](#symbol-to-string)
* [Extract Function](#extract-function)
* [Inline Function](#inline-function)
---
* [Refactoring Footnote](#refactoring-footnote)
* [Exporting Footnote](#exporting-footnote)
* [Importing Footnote](#importing-footnote)

## More preface

I almost exclusively edit Lisp with vim using
[slimv](https://github.com/kovisoft/slimv/) -- lately I've even been trying out
neovim, but I go back and forth and nothing here should require one or the
other. As such, I recognize most of these tricks won't exactly be applicable to
most Lisp programmers who are happy using emacs, and maybe not so useful to
other vimmers happy using [vlime](https://github.com/vlime/vlime).

I'm also happy to call them tricks/hacks/kludges/omg-wtfs because at the end of
the day, they're fragile. My vimscript knowledge is pretty weak still, and at
some level they're all based on textual find-replace schemes, not a semantic
understanding of the code.

I'm not convinced emacs+slime/sly can do these out of the box. If they could,
I'd be more motivated to use them. (Feel free to correct me by opening an Issue
or something. I am vaguely aware of an emacs lisp refactoring contrib as part of
slime, but I don't think it's on by default, and what it offered was pretty
barebones. Maybe things have changed. And someone, I forgot who, was supposedly
working on an editor-agnostic library a few years ago to support refactoring,
but I haven't heard any updates about it.)

If I do remember or have heard about slime doing these things, I'll make a note
of it. For example, the first refactor around exporting symbols should be
supported by `C-c x` or `slime-export-class`.

These types of automatic edits are *table stakes* for what programmers of other
languages can expect their editors and IDEs to provide for them. Maybe this
collection will give inspiration to some emacs hackers to improve their own
ecosystems. Or maybe not.

I think some strengths of Lisp are paradoxically at play here to explain why in
the year 2024 "the most powerful" Lisp editors don't have basic automatic
refactoring tools from decades ago.

One, the language itself encourages *loose
coupling* in many ways. That just means that big refactors which could benefit
more from tooling support are more rare.

Two, the language itself has so many powerful features enabling efficient and
correct expression. The pressure to refactor to a better structure will be
smaller, because you're more likely to already have a better structure, and just
the fact it's easy to define things like top-level functions (in addition to
`flet` and friends) means that a lot of refactor-into-function transformations
are trivial to do manually, and hard to do automatically. (Structural editing
support from the editor makes them even easier.)

If you've ever written a lot of Java without auto-complete assistance, then you
know what it feels like to write a new method. Even if you like typing, like I
do (I don't use slimv's paredit stuff, though I do use
[vim-surround](https://github.com/tpope/vim-surround) and
[vim-sexp-mappings-for-regular-people](https://github.com/tpope/vim-sexp-mappings-for-regular-people)),
those things are *weighty*, even in Hello World, and the refactoring guys want
me to make more of them?

## Template

For each refactoring trick, the following is the ideal template of what you should expect to see:

----

## Refactoring name

Should be familiar if relevant to other languages.

If there's an Emacs/slime equivalent I know about, then I'll mention it here.

#### Motivation

More detailed description of what's actually meant and should happen with the
refactoring, including an example or two. Should be somewhat motivating.

#### Video

A short video or gif of a typical usage of the refactor in action.

If the video example is significantly different from what was given in the motivation (for example, it's on a much larger set of code than the tiny example), highlight
some important notes about what happened that may not have been obvious.

#### Code

Code to add to .vimrc to actually do it, and what assumptions or limitations
it has.

----

TLDR: scroll to the bottom of each refactoring to see if there's code implemented for it yet.

## Common Functions

Last section before we get to the refactors themselves.

If you couldn't already tell, this is all pretty hacky, and I used AI tools to help vibe-code a lot as well because I don't really know vimscript well. Here are
some common functions that will be useful for multiple refactoring commands. A list of general pre-requisites beyond using slimv is at the top.

```vim
" Requirements:
" SBCL (reach out if you'd like me to investigate for you how to get these working in another impl)
" slimv https://github.com/kovisoft/slimv/
" vim-sexp https://github.com/guns/vim-sexp
"   (however I also recommend vim-sexp-mappings-for-regular-people https://github.com/tpope/vim-sexp-mappings-for-regular-people)


function! SlimvEvalQuiet(exp)
  " Runs exp in the connected Lisp without showing it or its output in the REPL
  " Need to escape any double quotes in exp...
  if SlimvConnectSwank()
    call SlimvFindPackage()
    let l:result = trim(SlimvCommandGetResponse(':pprint-eval', 'python3 swank_pprint_eval("' . a:exp . '")', 0))
    return l:result
  endif
endfunction

function! UndoubleQuote(str)
  " given string with a value of `"value"`, strip its surrounding double quotes
  return substitute(a:str, '^"\(.\{-}\)"$', '\1', '')
endfunction

function! GetPackageSrc(pkg_name)
  " consider using shinmera's https://shinmera.github.io/definitions/ if you
  " need ccl / clasp support...
  if a:pkg_name == ''
    let l:pkg = '*package*'
  else
    let l:pkg = '(find-package \"' . toupper(a:pkg_name) . '\")'
  endif

  let l:script='
\ (let* ((src (sb-introspect:find-definition-source ' . l:pkg . '))
\        (path (sb-introspect:definition-source-pathname src))
\        (form (sb-introspect:definition-source-form-path src))
\        (form-number (sb-introspect:definition-source-form-number src))
\        (offset (sb-introspect:definition-source-character-offset src)))
\   (declare (ignore form form-number offset))
\   (namestring (or path \"\")))'
  let l:filename=SlimvEvalQuiet(l:script)
  return UndoubleQuote(l:filename)
endfunction

function! GetPackageName()
  return tolower(UndoubleQuote(SlimvEvalQuiet('(package-name *package*)')))
endfunction

function! FindDefPackageLine(pkg_name)
  " Try to find package name for various formats:
  " (defpackage foo
  " (defpackage :foo
  " (defpackage #:foo
  " (defpackage "FOO"
  " (defpackage FOO
  " and above with any spaces/tabs/newlines between the two symbols,
  " and all case-insensitively (the \c),
  " and also allowing variants like (cl:defpackage or (uiop:define-package
let l:patterns = [
    \ '\c(\(cl:\)\?defpackage\_s\+' . a:pkg_name . '\($\|\_s\|)\|(\)',
    \ '\c(\(cl:\)\?defpackage\_s\+:' . a:pkg_name . '\($\|\_s\|)\|(\)',
    \ '\c(\(cl:\)\?defpackage\_s\+#:' . a:pkg_name . '\($\|\_s\|)\|(\)',
    \ '\c(\(cl:\)\?defpackage\_s\+"' . toupper(a:pkg_name) . '"\($\|\_s\|)\|(\)',
    \ '\c(\(uiop:\)\?define-package\_s\+' . a:pkg_name . '\($\|\_s\|)\|(\)',
    \ '\c(\(uiop:\)\?define-package\_s\+:' . a:pkg_name . '\($\|\_s\|)\|(\)',
    \ '\c(\(uiop:\)\?define-package\_s\+#:' . a:pkg_name . '\($\|\_s\|)\|(\)',
    \ '\c(\(uiop:\)\?define-package\_s\+"' . toupper(a:pkg_name) . '"\($\|\_s\|)\|(\)'
    \ ]

  for l:pattern in l:patterns
    let l:pkg_start_line = search(l:pattern, 'w')
    if l:pkg_start_line > 0
      return l:pkg_start_line
    endif
  endfor
  return 0 " failure, like search itself
endfunction

function! GetSymbolPackage(symbol)
  " Gets the package name of the symbol. Assumes symbols are not |caseSensitive|.
  let l:script = '(let ((sym (find-symbol \"' . toupper(a:symbol) . '\"))) (when sym (package-name (symbol-package sym))))'
  let l:result = SlimvEvalQuiet(l:script)
  return UndoubleQuote(l:result)
endfunction

function! FindSymbolWithApropos(symbol)
  " Find symbol using apropos when it's not already imported
  let l:script = '(let ((results ())) (dolist (sym (apropos-list \"' . toupper(a:symbol) . '\")) (when (string= (symbol-name sym) \"' . toupper(a:symbol) . '\") (push (package-name (symbol-package sym)) results))) (nreverse results))'
  let l:result = SlimvEvalQuiet(l:script)

  if empty(l:result) || l:result == 'NIL'
    return ''
  endif

  " Parse the result - it should be a list of package names
  let l:result = UndoubleQuote(l:result)

  " Remove outer parentheses and split by spaces, handling quoted strings
  let l:result = substitute(l:result, '^(\|)$', '', 'g')
  let l:packages = split(l:result, '\s\+')

  " Clean up package names (remove quotes)
  let l:clean_packages = []
  for pkg in l:packages
    let l:clean_pkg = substitute(pkg, '^"\|"$', '', 'g')
    if !empty(l:clean_pkg)
      call add(l:clean_packages, l:clean_pkg)
    endif
  endfor

  if empty(l:clean_packages)
    return ''
  elseif len(l:clean_packages) == 1
    return l:clean_packages[0]
  else
    " Multiple matches - let user choose
    echo "Multiple packages found for " . a:symbol . ":"
    for i in range(len(l:clean_packages))
      echo (i + 1) . ": " . l:clean_packages[i]
    endfor
    let l:choice = input("Choose package (1-" . len(l:clean_packages) . "): ")
    let l:idx = str2nr(l:choice) - 1
    if l:idx >= 0 && l:idx < len(l:clean_packages)
      return l:clean_packages[l:idx]
    else
      echo "Invalid choice"
      return ''
    endif
  endif
endfunction

function! IsSymbolImported(symbol, pkg_name)
  let l:script = '(multiple-value-bind (sym status) (find-symbol \"' . toupper(a:symbol) . '\" \"' . toupper(a:pkg_name) . '\") (values status (when sym (package-name (symbol-package sym)))))'
  let l:result = SlimvEvalQuiet(l:script)
  return l:result !~# 'NIL'
endfunction
```

----

## Export Symbol

Possibly covered by `C-c x` or `slime-export-class` with emacs and slime.
[This](https://github.com/LispCookbook/cl-cookbook/issues/269) page has some
details.

#### Motivation

You've written a new function or several in your package, and you want
it or them to be part of the package's public API, so you need to
export[²](#exporting-footnote) the symbols.

```lisp
(defpackage #:lgame.event
  (:use #:common-lisp)
  (:documentation
    "blah blah"))
(in-package #:lgame.event)

(defun key-scancode (event)
  (ref event :key :keysym :scancode))

(defun key-pressed? (&key key any all) ...)
```

If you're lazy, you use [cl-annot](https://github.com/m2ym/cl-annot) and that
becomes:

```diff
(defpackage #:lgame.event
-  (:use #:common-lisp)
+  (:use #:common-lisp #:annot.std)
  (:documentation
    "blah blah"))
(in-package #:lgame.event)

+(annot:enable-annot-syntax)

+@export
(defun key-scancode (event)
  ;(plus-c:c-ref event sdl2-ffi:sdl-event :key :keysym :scancode))
  (ref event :key :keysym :scancode))

+@export
(defun key-pressed? (&key key any all) ...)
```

It's fine... but it turns the `defpackage` into a soft lie. Isn't it better, in
general, to have the package explicitly list its exports? Is it worth importing
this unmaintained LLGPL-licensed reader macro, that doesn't even use
[named-readtables](https://github.com/melisgl/named-readtables), just to save
some buffer switching and typing?

*Yes*, I used to say, and do. But: no more! That's what this refactor is for.
With the cursor on the function name symbols in the original code above,
invoke the refactor (my current binding: `\ex`) twice and the `defpackage` form
is automatically modified to:

```lisp
(defpackage #:lgame.event
  (:use #:common-lisp)
  (:documentation
    "blah blah")
  (:export #:key-scancode
           #:key-pressed?))
```

#### Video

Here's a short video of me invoking the refactor on various `@export`'d functions.

![export symbol vid](recs/export-symbol.gif)

> [!NOTE]
> Notice that the first file in the video follows the common convention where
> the `defpackage` form lives in a different file. The refactor command
> automatically found and opened it in a buffer and made the modifications
> there. The last file had to be loaded before the export could work.

Also notice that if there's an existing `:export` section it will append to its
list, otherwise it will make a new `:export` section as the last element of the
`defpackage`.

#### Code

```vim
function! ExportSymbol()
  " Adds the symbol under the cursor to the defpackage's export list.

  " Store current position and buffer and view
  let l:cur_pos = getpos('.')
  let l:cur_buf = bufnr('%')
  let l:cur_view = winsaveview()

  " Get the symbol under cursor
  let l:symbol = expand('<cword>')

  " Get package name and source file
  " (note name must come first as it searches current file)
  let l:pkg_name = GetPackageName()
  let l:pkg_file = GetPackageSrc('')
  if empty(l:pkg_file)
    echo "Could not find package source file"
    return
  endif

  " Load package file into new buffer if needed
  let l:pkg_in_other_buf = 0
  if expand('%:p') !=# l:pkg_file
    let l:pkg_in_other_buf = 1
    let l:pkg_buf = bufnr(l:pkg_file)
    if l:pkg_buf == -1
      execute 'edit ' . fnameescape(l:pkg_file)
    else
      execute 'buffer ' . l:pkg_buf
    endif
  endif

  let l:pkg_start_line = FindDefPackageLine(l:pkg_name)

  if l:pkg_start_line == 0
    " Close buffer and restore position:
    execute 'buffer ' . l:cur_buf
    call setpos('.', l:cur_pos)
    call winrestview(l:cur_view)
    echo "Could not find defpackage form"
    return
  endif

  if !l:pkg_in_other_buf
    " Need to account for newly added line when moving cursor back
    let l:cur_pos[1] = l:cur_pos[1]+1
    let l:cur_view['lnum'] = l:cur_view['lnum']+1
    let l:cur_view['topline'] = l:cur_view['topline']+1
  endif

  " Find the end of the defpackage form, move to it
  execute "normal \<Plug>(sexp_move_to_next_bracket)"

  " Look for existing :export clause in the defpackage boundary,
  " moving to it if found
  let l:has_export = search('(:export\_s', 'b', l:pkg_start_line)

  if l:has_export
    execute "normal \<Plug>(sexp_move_to_next_bracket)"
    execute "normal i\n#:" . l:symbol
  else
    execute "normal i\n(:export #:" . l:symbol . ")"
  endif

  " Save changes?
  " write

  " Return to original buffer and position
  execute 'buffer ' . l:cur_buf
  call setpos('.', l:cur_pos)
  call winrestview(l:cur_view)
endfunction

nnoremap <silent> \ex :call ExportSymbol()<cr>
```

Limitations:

* SBCL only. (Portability option available and mentioned in the
common functions.)
* It's assuming you're using slimv and have already LOADed your code into Lisp,
so SBCL knows what file the `defpackage` lives in. If it can't determine that,
or it can't find the `defpackage` in the file, it gives up.
* Therefore, if you make changes to the defpackage file, be sure to reload it
  either by reloading the whole system or executing `,L` in the vim buffer.
* Assumes you haven't run `,g` to override the default package from cl-user when
  slimv doesn't know otherwise.
* Does not automatically export the slots or accessors of a class, or the
  accessors and constructor of a struct.

In theory, vim could search all the files manually itself, and this could be a
pure text refactor that doesn't depend on SBCL, a Lisp running, or slimv.

## Inline and Import Symbol

#### Motivation

You've written code like this:

```lisp
(alexandria:define-constant +width+ 1920)
```

and decide you want to drop the package name. Invoke the refactor (my binding is
currently `\in`) on the symbol and you get:

```lisp
(define-constant +width+ 1920)
; ... elsewhere ...
(defpackage #:foo
  (:use #:cl)
  (:import-from #:alexandria
                #:define-constant))
```

See this [footnote](#importing-footnote) on why you might not want to do this.

#### Video

![inline import vid](recs/inline-and-import.gif)

Like the export symbol refactor, this automatically finds the source location of
the relevant `defpackage`. It tries to be kind of smart and not add the symbol
to the import list if it's already been imported, so you can just spam `\in` in
a series of lines and be good. It will also automatically inline all uses of
that symbol in the buffer. (See in the video how `cl-ppcre:split` gets turned into `split` on both line 27 and 39.)

#### Code

```vim
function! InlineAndImportSymbol()
  " Removes the package prefix from the symbol under the cursor,
  " adds it to the import list,
  " and replaces ALL full package-prefixed names in the current buffer
  " with the plain symbol name.

  let l:cur_pos = getpos('.')
  let l:cur_buf = bufnr('%')
  let l:cur_view = winsaveview()

  " Get the current word with package prefix
  let l:line = getline('.')
  let l:col = col('.')

  " Find package:symbol pattern around cursor
  let l:start_col = l:col
  let l:end_col = l:col

  " Expand backwards to find package prefix
  while l:start_col > 1 && l:line[l:start_col-2] =~# '[a-zA-Z0-9.\-:]'
    let l:start_col -= 1
  endwhile

  " Expand forwards to find full symbol
  while l:end_col < len(l:line) && l:line[l:end_col-1] =~# '[a-zA-Z0-9.\-:?!]'
    let l:end_col += 1
  endwhile

  let l:full_symbol = l:line[l:start_col-1:l:end_col-2]

  " Check if it contains a colon (package separator)
  if l:full_symbol !~# ':'
    echo "No package prefix found, full_symbol was " . full_symbol
    return
  endif

  let l:parts = split(l:full_symbol, ':')
  if len(l:parts) < 2
    echo "Invalid package:symbol format"
    return
  endif

  let l:pkg_name = l:parts[0]
  let l:symbol = l:parts[-1]

  " Get current package info
  let l:cur_pkg_name = GetPackageName()
  let l:pkg_file = GetPackageSrc('')

  if empty(l:pkg_file)
    echo "Could not find package source file. Make sure to LOAD the file if you made defpackage changes. cur_pkg_name: " . cur_pkg_name
    return
  endif

  " Check if symbol is already imported
  if IsSymbolImported(l:symbol, l:cur_pkg_name)
    echo 'Symbol already imported.'
    " Just do the inline part
    "execute l:start_col . ',' . (l:end_col-1) . 's/' . escape(l:full_symbol, '/') . '/' . l:symbol . '/'
    execute '%s/' . escape(l:full_symbol, '/') . '/' . l:symbol . '/g'
    call setpos('.', l:cur_pos)
    return
  endif

  " Open package file
  let l:pkg_in_other_buf = 0
  if expand('%:p') !=# l:pkg_file
    let l:pkg_in_other_buf = 1
    let l:pkg_buf = bufnr(l:pkg_file)
    if l:pkg_buf == -1
      execute 'edit ' . fnameescape(l:pkg_file)
    else
      execute 'buffer ' . l:pkg_buf
    endif
  endif

  let l:pkg_start_line = FindDefPackageLine(l:cur_pkg_name)

  if l:pkg_start_line == 0
    execute 'buffer ' . l:cur_buf
    call setpos('.', l:cur_pos)
    call winrestview(l:cur_view)
    echo "Could not find defpackage form"
    return
  endif

  " Find end of defpackage
  call cursor(l:pkg_start_line, 1)
  execute "normal \<Plug>(sexp_move_to_next_bracket)"

  " Look for existing :import-from clause for this package
  let l:import_pattern = '(:import-from\_s\+[#]\?[:]\?' . l:pkg_name . '\_s'
  let l:has_import = search(l:import_pattern, 'b', l:pkg_start_line)

  if l:has_import
    " Add to existing import list
    execute "normal \<Plug>(sexp_move_to_next_bracket)"
    execute "normal i\n#:" . l:symbol
  else
    " Create new import-from clause
    execute "normal i\n(:import-from #:" . l:pkg_name . "\n#:" . l:symbol . ")"
  endif

  " Return to original buffer and inline the symbol
  execute 'buffer ' . l:cur_buf
  call setpos('.', l:cur_pos)
  call winrestview(l:cur_view)

  " Replace package:symbol with just symbol
  "execute l:start_col . ',' . (l:end_col-1) . 's/' . escape(l:full_symbol, '/') . '/' . l:symbol . '/'
  execute '%s/' . escape(l:full_symbol, '/') . '/' . l:symbol . '/g'
  call setpos('.', l:cur_pos)

  echo "Imported and inlined: " . l:symbol
endfunction

nnoremap <silent> \in :call InlineAndImportSymbol()<cr>
```

Limitations: same as export symbol. Additionally, no support for knowing that a
`:shadowing-import-from` was needed, instead. While it does do a mass
search-replace on the current buffer, it won't touch other files that might be
in the same package.

## Uninline Symbol

#### Motivation

You're looking at code you or someone wrote like this:

```lisp
(define-constant +width+ 1920)
```

The uninline symbol refactor (my binding is currently `\uin`) turns this to:

```lisp
(alexandria:define-constant +width+ 1920)
```

Essentially undoing the inline step from the inline and import symbol refactor.

This refactoring was somewhat inspired by Clojure's
[Slamhound](https://github.com/technomancy/slamhound) back in the day and it
made working with Java stuff a lot less tedious.

#### Video

![uninline vid](recs/uninline.gif)

Here we essentially undo what the last video did, except the `defpackage` form is unchanged.

#### Code

```vim
function! UninlineSymbol()
  " Add package prefix to symbol based on its actual package

  let l:symbol = expand('<cword>')

  if empty(l:symbol)
    echo "No symbol under cursor"
    return
  endif

  " Get the package this symbol belongs to
  let l:pkg_name = GetSymbolPackage(l:symbol)

  " If not found, try apropos as fallback
  if empty(l:pkg_name) || l:pkg_name == 'nil'
    let l:pkg_name = FindSymbolWithApropos(l:symbol)
  endif

  if empty(l:pkg_name) || l:pkg_name == 'nil' || l:pkg_name ==# 'COMMON-LISP-USER'
    echo "Cannot determine package for symbol: " . l:symbol
    return
  endif

  " Replace symbol with package:symbol
  let l:qualified_symbol = tolower(l:pkg_name) . ':' . l:symbol
  execute 's/\<' . l:symbol . '\>/' . l:qualified_symbol . '/'

  echo "\nUninlined: " . l:qualified_symbol
endfunction

nnoremap <silent> \uin :call UninlineSymbol()<cr>
```

Limitations: it does *not* remove the symbol from the `defpackage` list. That would require a
more extensive search and replace for all uses in the current package, which can span
multiple files. Doable in principle, but not covered by this code. And to make it extra clear, as in the video, only the symbol under the cursor changes, not
any other usages of the symbol in the file.

It *does* do a bit of magic guessing. The easiest case is if the symbol is actually imported and interned (i.e. you've loaded the code already). The harder case
is if you've just typed the code but haven't actually evaluated it yet. Still, we try to find the symbol's home package via `apropos`. This can of course have
conflicts, but the script should ask you which one you want, as in the video.

This also does not use any package nicknames, just the main package name. (Shown in the video with the un-inlining of `aref` to `common-lisp:aref` instead of
`cl:aref`.)

## Add File To System

#### Motivation

You've taken some code and split it off into a new file. Now you need to add
that file to your project's system definition, which means you have to go add it
to your ASDF file. You have to consider the proper `load` order for it.

```lisp
...
  :components ((:module "src/"
                :serial t
                :components ((:file "package")
                             (:file "my-new-file")
                             (:file "main"))))
...
```

I'm still thinking this one through, so it's wishlist status. I have some code
that does a dumb thing of just assuming your project has a single `:serial t`
set of components, and that it's safe to add this new file to one from the end
(as in the example) but that's obviously not acceptable in general. At best it
saves you typing the `(:file "filename")` form, and then you can use `>f` and
`<f` to push it around to its proper place.

I think the best solution is probably to just give up and use ASDF's
package-inferred feature, but I detest that they require a package-name
hierarchy separator of a `/`. If it's ever easy to configure that to a `.` I'll
mind it less. Furthermore, there are times it's useful to have files share the
same package.

I bet an 80% solution could be made that just globs existing top-level
class/function/method/macro/defvar/... names as pure text, sends them to
`tsort`, and uses that ordering. (Maybe the glob can be more principled, using
the exported symbols list of known packages.) Difficulties still seem to arise
when using multiple modules, though. I guess the refactor invocation can always
ask which (give a list, don't force typing the name from memory) module to
restrict to.

89-90%+ solutions are valid; consider most Java refactoring tools don't even
bother to try and do anything about the possible presence of reflection, even if
they could do more, and so method names encoded as strings could be missed in a
rename. A lot of refactor tool actions also ask extra questions after you invoke
them, rather than just immediately doing one thing.

#### Video

(video here)

#### Code

(code here)

Note: should at least be smart enough to handle multiple components in the asd
file such that if one specifies a module with a directory name like "src/",
or a module with a name and an explicit :pathname like "src/otherdir", then if
the current file is in a matching directory it should be added to the component
list of the correct module. If not, a new module with the new path should be
added.

## Mass Rename

#### Motivation

Very common refactor. I want to change the name of my function, or class, or
variable, and update all uses of the old name to the new name.

I use [vim-grepper](https://github.com/mhinz/vim-grepper/) for this sort of
thing. A chief concern it helps solve is that you probably don't have all of the
files of your project already open as buffers, and you probably don't want all
such files open as buffers. Nevertheless, this will find usages in all files of
your project, whereas other solutions limit themselves to just open buffers.

#### Video

(video here)

#### Code

Again just use vim-grepper, but I should document what I do in an example video.

## Symbol To String

#### Motivation

Goal is to turn any of `symbol`, `#:symbol`, `#'symbol`, `:symbol`, `'symbol`,
`` `symbol``, `,symbol` into `"symbol"`. This is borderline useful. The main
motivator was when I decided to follow ASDF's guidance that canonical system
names are strings, not keyword symbols or uninterned symbols, despite a lot of
Lispers using the latter two forcing ASDF to maintain compatibility. I used this
to convert my system definition names and their specified dependencies.

I remember when I was first learning Lisp, I was confused about the differences
between systems and packages. I think having the convention of systems being
strings, and packages being uninterned symbols for their `defpackage` form (and
optionally uninterned symbols or keyword symbols for their `in-package` form) is
helpful for distinguishing them further.

#### Video

![sym2string vid](recs/symbol2string.gif)

#### Code

```vim
" symbol -> quoted string
" #:blah -> "blah"
" #'func -> "func"
" :foo -> "foo"
" 'some-symbol -> "some-symbol"
" unquoted:symbol -> "unquoted:symbol"
" `quasi -> "quasi"
" ,escaped -> "escaped"
function! RefactorSymbolToString()
  " Depends on vim-surround / vim-sexp-mappings-for-regular-people...
  " should use n2char(getchar())? though surround.vim has its own s:getchar()
  let cursor_col_idx = col('.') - 1
  let cur_line = getline('.')
  let cur_char = cur_line[cursor_col_idx]
  setlocal iskeyword+=#
  setlocal iskeyword+='
  setlocal iskeyword+=`
  setlocal iskeyword+=,
  let cur_word_col_idx = strridx(cur_line, expand('<cword>'), cursor_col_idx)
  setlocal iskeyword-=,
  setlocal iskeyword-=`
  setlocal iskeyword-='
  setlocal iskeyword-=#
  " note not using <cWORD> since I think that includes too much...
  "let is_on_keyword = (cur_word_col_idx >=0 && (cur_word_col_idx + strlen(expand('<cword>')) >= cursor_col_idx))
  let is_at_start_keyword = (cur_word_col_idx == cursor_col_idx)
  if !is_at_start_keyword
    " move to it
    "execute 'normal! B'
    call sexp#move_to_adjacent_element('n', v:count, 0, 0, 0)
    " don't infinite loop..
    if getline('.')[col('.')-1] == '"'
      return
    endif
    return RefactorSymbolToString()
  endif
  if cur_char == '#' || cur_char == ':' || cur_char == "'" || cur_char == '`' || cur_char == ','
    execute 'normal! x'
    return RefactorSymbolToString()
  endif
  " At this point, any prefixes should have been stripped so it's just a naked
  " symbol we can slap double quotes around.
  "
  " I would like to use ysiw" here, since I'm not sure if <cword> captures all
  " of what counts as a lisp symbol as a 'word', but for some reason it's
  " busted in a weird way. That is, if you replace the next two lines with
  " either:
  " execute 'normal! g@iw"'
  " or
  " execute 'normal! ysiw"'
  " then you'll be left with just the symbol but no surrounding quotes.
  " However, if you manually type ysiw"
  " in your editor, then try this command again on other symbols, it will work
  " and give you the quoted symbol... Well, we'll revisit this when cword
  " isn't good-enough.
  let word = expand('<cword>')
  execute 'normal! ciw"' . word . '"'
endfunction

map \rf2s :call RefactorSymbolToString()<cr>W
````

Limitations: as mentioned in the final comment of the code, there might be some instances where what gets quoted isn't what was expected. Let me know if you run
into one.

## Extract Function

#### Motivation

Perhaps the most famous refactor.

```lisp
(defun decode-param (s)
  (labels ((f (lst)
             (when lst
               (case (first lst)
                 (#\% (cons (let ((code (parse-integer
                                          (coerce (list (second lst) (third lst)) 'string)
                                          :radix 16
                                          :junk-allowed t)))
                              (if code
                                  (code-char code)
                                  #\Space))
                            (f (nthcdr 3 lst))))
                 (#\+ (cons #\space (f (rest lst))))
                 (t (cons (first lst) (f (rest lst))))))))
    (coerce (f (coerce s 'list)) 'string)))
;; Unsure what this does?
(decode-param "foo%3F") ;-> "foo?"
(decode-param "foo%2Fbar") ;-> "foo/bar"
(decode-param "hello+there") ;-> "hello there"
```

If you're thinking a Scheme programmer must have written that, you might be
right. I tend to view `flet` and `labels` as code smells. When you write code
like this, you very quickly start to attack the right margin, and I don't think
that's good.

(Note: independent of how it looks, I don't endorse this code for its seeming
purpose, it just makes a quick example. I think this code originates from *Land
of Lisp* and used `car`s and `caddr`s and so on and might have been more
refactored originally. It's a fine book, my chief criticism is only that it
tends to default to such kinds of code, but as a book part of its whole point is
to demonstrate code from the whole family of Lisp-likes and to do things with
minimal library support and without worrying about excessive consing.)

An ideal refactor command here would let me have my cursor over the `f` and
invoke it, have it ask me for a name, and produce:

```lisp
(defun decode-param (s)
  (coerce (decode-helper (coerce s 'list)) 'string))

(defun decode-helper (lst)
  (when lst
    (case (first lst)
      (#\% (cons (let ((code (parse-integer
                               (coerce (list (second lst) (third lst)) 'string)
                               :radix 16
                               :junk-allowed t)))
                   (if code
                       (code-char code)
                       #\Space))
                 (decode-helper (nthcdr 3 lst))))
      (#\+ (cons #\space (decode-helper (rest lst))))
      (t (cons (first lst) (decode-helper (rest lst)))))))
```

Note that the `f` has been replaced with the name `decode-helper` at both the
original call site and in the recursive calls.

I might go a step further and have my cursor over the `let` in `let ((code ...`
and refactor that, giving in all:

```lisp
(defun decode-param (s)
  (coerce (decode-helper (coerce s 'list)) 'string))

(defun decode-helper (lst)
  (when lst
    (case (first lst)
      (#\% (cons (http-char (second lst) (third lst))
                 (decode-helper (nthcdr 3 lst))))
      (#\+ (cons #\space (decode-helper (rest lst))))
      (t (cons (first lst) (decode-helper (rest lst)))))))

(defun http-char (c1 c2)
  (let ((code (parse-integer
                (coerce (list c1 c2) 'string)
                :radix 16
                :junk-allowed t)))
    (if code
        (code-char code)
        #\Space)))
```

This is a tricky tool to get right. It would have needed to prompt me for not
only a new name (given as `http-char`) but also what arguments I wanted it to
have (`c1`, `c2`), and a mapping of those arguments to the original source
(`(second lst)`, `(third lst)`). It would have been equally valid if I requested
`http-char` takes one argument `lst`, making the parent call be `(http-char
lst)` and the `(second lst)` and `(third lst)` bits being pushed down into
`http-char`.

There's an argument to be made about whether the extracted function should sit
above or below the original function. This should just be a parameter to or
prompt of the refactor tool. Personally I generally prefer it to be below,
because I read top to bottom, but when I'm in pure interactive development mode,
it's nice to have it above so that when I start redefining things piecemeal I
don't get warnings about missing function calls. However, as long as you're
doing a `compile-file` or system load, which you should be doing later
regardless once the file crystallizes more, SBCL is smart enough to fix up
forward-referenced calls and there's no performance penalty or warning.

For a lisper comfortable with manipulating sexps, automating this refactor may
not be such a high priority value. Where it can showcase more value is when the
tool can replace the same pattern occurring elsewhere in the file with the new
function call.

So, what I have implemented below requires a bit of manual effort: the user must select in visual mode the stuff they want to extract.

This selection is replaced by a function call, whose name and other details the user supplies in a prompt.

Thus, as we'll see in the video, we don't get the magic "refactor the `f` function out of `labels` into its own `decode-helper`", but we do get a simple refactor of
the inner `let` within the `cons`. Before that though I also show in the video that it's not that hard to break out the `f` function manually. (It's a bit harder if
it's a closure and captures some of the surrounding variables, but you'll get a compiler warning soon enough, as I intentionally show below when I don't rename
the recursive `f` calls at first.)

#### Video

![extract function vid](recs/extract-function.gif)

#### Code

```vim
function! ExtractFunction(visual)
  " Operates on a visual selection to extract it into a new top-level function.
  if !a:visual
    echo "Error: ExtractFunction must be called in Visual mode."
    return
  endif

  " --- 1. Reliably get the selected text without using registers ---
  let l:start_pos = getpos("'<")
  let l:end_pos = getpos("'>")
  let l:start_line = line("'<")
  let l:end_line = line("'>")
  let l:start_col = col("'<")
  let l:end_col = col("'>")

  let l:lines = getline(l:start_line, l:end_line)
  if empty(l:lines)
    echo "Error: Could not get selected text."
    return
  endif

  if l:start_line == l:end_line
    let l:lines[0] = strpart(l:lines[0], l:start_col - 1, l:end_col - l:start_col + 1)
  else
    " Multi-line selection. Handle first and last lines which may be partial.
    let l:last_idx = len(l:lines) - 1
    let l:lines[l:last_idx] = strpart(l:lines[l:last_idx], 0, l:end_col)
    let l:lines[0] = strpart(l:lines[0], l:start_col - 1)
  endif
  let l:original_selection_text = join(l:lines, "\n")


  " --- 2. Get user input ---
  let l:func_spec_input = input('New function signature (e.g., my-func arg1 arg2): ')
  if empty(l:func_spec_input)
    echo "Extraction cancelled."
    return
  endif

  let l:func_spec_parts = split(l:func_spec_input)
  let l:func_name = l:func_spec_parts[0]
  let l:params = len(l:func_spec_parts) > 1 ? l:func_spec_parts[1:] : []

  let l:call_site_args = []
  let l:new_body = l:original_selection_text
  for l:param in l:params
    let l:sub_expr = input('Enter call site expression for ''' . l:param . ''': ')
    if empty(l:sub_expr)
      echo "Extraction cancelled."
      return
    endif
    call add(l:call_site_args, l:sub_expr)
    let l:escaped_expr = escape(l:sub_expr, '.*[]^%$\~')
    let l:new_body = substitute(l:new_body, l:escaped_expr, l:param, 'g')
  endfor

  " --- 3. Construct new forms ---
  let l:call_form = '(' . l:func_name . (empty(l:params) ? '' : ' ' . join(l:call_site_args, ' ')) . ')'
  let l:new_defun_str = "(defun " . l:func_name . " (" . join(l:params, ' ') . ")\n" . l:new_body . ")"

  " --- 4. Perform buffer manipulations ---
  let l:placement = input("Place function [a]bove or [b]elow current function? (default: below): ")
  let l:place_above = (l:placement ==? 'a' || l:placement ==? 'above')
  call setpos('.', l:start_pos)
  " Find the start of the top-level form (assumes it starts at column 0).
  call search('^(', 'b')
  if l:place_above
    execute "normal! O"
    let l:insertion_line = line('.') - 1
  else
    execute "normal! %"
    execute "normal! o"
    let l:insertion_line = line('.')
  endif

  " Insert the new defun after the current form.
  call append(l:insertion_line, split(l:new_defun_str, "\n"))

  " Indent the newly added function.
  let l:new_func_start_line = l:insertion_line + 1
  let l:new_func_end_line = l:insertion_line + len(split(l:new_defun_str, "\n"))
  execute l:new_func_start_line . ',' . l:new_func_end_line . 'normal! =='

  " Finally, replace the original selection with the new function call.
  " Re-select last visual area
  execute "normal! gv"
  " Change it
  execute "normal! c" . l:call_form . "\<Esc>"

  echo "Extracted function '" . l:func_name . "'"
endfunction

vnoremap <silent> \ef :<C-u>call ExtractFunction(1)<CR>
```

Limitations:

* Needs the user to visually select the thing they're extracting (fortunately in visual mode, `%` works well to span a sexp).
* If renamed variables are substrings of other variables, could break the refactored function.
* If refactoring lines instead of a single paren-wrapped form (like the final test refactor in the video) be careful that the last line doesn't end with a comment, or else the inserted defun won't close properly.
* Does not attempt to search-replace similar patterns within the rest of the file to replace those with the function call as well.

## Inline Function

#### Motivation

Inverse of extract function.

It's especially useful with other people's code if you feel they sometimes have gone too far with extracting functions and you're
finding them annoying rather than helpful. Yes, you can always jump to source and back, but if you have to do this too much to understand a piece of code, the
functions probably aren't well-factored, and it will be easier to read them inline.

Naturally, this refactor should also `macroexpand-1` if the symbol is a macro.

For CLOS methods, however, what should be done? With multiple dispatch, there's not even "the" function to inline unless you already know your types. I suppose
this could be a very interactive refactor, asking the user for such types, or just gathering all defmethods and inlining them as a `labels` with a typecase.
However there's also the complications of any before/after/around methods, and any calls to parent methods with `call-next-method`. I'm going to say, this
refactor only deals with functions (and can `macroexpand-1` macros?) and leave methods for a separate dream refactor. As a proof case, it shouldn't require too
much manual effort to turn the defmethod code in [this](https://news.ycombinator.com/item?id=3389933) example to the comment's Scheme-like inlined version.

This refactor can have two modes to it: local and global.

The local mode only inlines the current call in the current file. It doesn't delete the source function, either. This is mainly for the cases where you just
want to quickly see it with inline context, and will probably undo the inline later.

The global mode is a global search-replace. It would rely on using the cross-referencing capabilities of the Lisp implementation to find all instances of
`who-calls` and go fix up those call sites with the inlined function code, and finally delete the original function definition.

This can be a rather difficult refactor to implement because it also depends on a sort of "smart copy" feature. For example, the function you're inlining might
be referring to symbols that it has privately imported in its own package that you have not. Does the refactor also import the same symbols to your package?
What if there are conflicts? What if you are trying to do it globally, are you going to also change the defpackage's of all the other packages of code that uses
this function?

Perhaps it could be another user input selection. In general I think it's better to convert them to package-qualified names if they aren't already imported by
the package they're getting inlined into. (Have to make sure that they actually are imported from the same parent, not just other imported symbols from other
packages with the same name.) But I can see a lot of circumstances where you want a global inline and also want every package that used the old function to
import what's needed to keep the original function source the same. Potential name conflicts can be resolved by the user outside the refactor command.

Another potential hiccup: if the function was declared inline to the compiler, then it might not show up in cross-referencing who-calls checks.

#### Video

(video here)

#### Code

(code here)

----

That's it for now!

----

## Refactoring Footnote

It's important to take the definitions at the top of this page seriously. A lot of programmers seem to
think that "refactoring" is any kind of code clean up, but it's not. If you have
"refactored" a piece of code, and suddenly things are not working, or you've
introduced a bug, or even introduced a new feature at the same time,  *you have
failed at the refactoring*. To succeed at refactoring means to make your purely
structural changes without changing any of the actual observable side effects
and results that the code had before you messed with it.

(Yes, it's very frequent to have a single commit do both a refactoring and a new
feature or bug fix, but conceptually you should not be doing both at the same
time. It's better to do multiple commits, even if you have a policy of squashing
them later.)

Sometimes this idea of "make sure you don't break/change anything" gives people
trouble. Like, they're not sure how to do that, and so there's fear. Especially
if they are working in a programming language that doesn't have static typing,
and so, "necessarily", as a consequence (right? right?), doesn't have automated
refactoring tools to make sure they haven't missed anything.

First of all, even with tools, you're going to mess up sometimes. You will fail at the refactoring. *That's ok*, so long as it's not too often and you learn from it.
That fear shouldn't prevent the attempt. I've failed a few times myself with Java and Eclipse not finding methods I'm renaming or what have you that were
referenced by string for a reflection call or for PowerMock to intercept. I find out about it later and I fix it up to get a complete non-failure refactoring.

Second of all, Fowler's original *Refactoring* book came out in 1999. In it, he used Java for
the examples. Java is statically typed, and even back then Java IDEs had some
decent automatic refactoring tools. There's a whole chapter on them. However, in 2018, he
released a second edition, but this time in JavaScript. The methods and thinking
are exactly the same, but he assumes you might not have any automatic
refactoring tools, so he shows how everything can be done manually. And you have
JavaScript's not so stellar type system, so you need to be careful.

I'm not saying you should go out and read one of the books -- I haven't even
read either version, just skimmed a digital copy sometimes -- it's on my backlog
to read fully. I *am* saying that the book serves as a proof for skeptics that
refactoring can be done just fine, pretty confidently, even in the absence of static types or
automated tooling, which is largely our situation with Lisp.

I also think, from what I have seen of the book, that it's quite good at
showcasing what refactoring is, can be, and how it's actually done. Having tools
or not, there's a flow, a rhythm, to the process. The book also is quite
level-headed about the value proposition of refactoring overall, including some
thoughts on when *not* to bother with it, and what to do when management is
skeptical.

(As a funny historical aside, the first language which had great automatic
refactoring tools was Smalltalk! The first tool was Refactoring Browser by John
Brant and Don Roberts. Smalltalk is dynamically typed, too. Fowler was even
going to use Smalltalk for his examples in his first book but was convinced to
go with Java.)

He stresses the importance of having a test suite to give you extra confidence
that you haven't inadvertently broken anything. This is useful even in Java.
If the code you want to refactor isn't covered by tests, Fowler says, then the
first thing you should do is add tests.

Now, I think that's all fine, and I think a good test suite helps refactoring
and so many other things. But I also think programmers can and should try to
rely on their pure reason, and not let themselves be paralyzed by fear because
they don't have full test coverage, or some tool like a compiler or refactor
window looking after them. Even when you have all the fancy automation tools,
it's useful to verify that yes, you can do things manually, you can be confident
about a change just by looking at it, you can understand what's going on using
just your brain. Avoid learned helplessness.
["It's more fun to be competent!"](https://www.youtube.com/watch?v=-cEn_83zRFw)

Suppose you had code like:

```lisp
(when (and (equal context-user
                  (comment-author comment))
           (< (comment-age-secs comment) 30))
  (perform-edit ...))
```

You're asked to change the policy to 60 seconds instead of 30. Before you do
that, though, you decide maybe you should encode the intended meaning of this
conditional with a name that will make it more clear what's being checked, and
to some extent why. Use the "Extract Function" refactor a couple times and...

```lisp
(when (can-edit-comment? context-user comment)
  (perform-edit ...))

(defun can-edit-comment? (user comment)
  (and (comment-you-authored? user comment)
       (< (comment-age-secs comment) 30)))

(defun comment-you-authored? (user comment)
  (equal user (comment-author comment)))
```

Before you recoil, can we at least agree that this transformation is trivial,
doesn't take long to do manually, and by inspection (no compiling, no test
running, no live running) we are *very confident* we haven't changed any
behavior yet? We didn't need to solve the halting problem, right?

If so, good, that's all I really want to get agreement on from my fellow
programmers -- that we are intelligent humans and can confidently change code in
many ways while only relying on our brains reasoning through the behavior. If we
have ways to take the pressure off our brains, like tests and compilers and so
on, ways to double-check ourselves, because we're also humans prone to mistakes,
then great! So much the better. Use them, I do. Just don't let the lack of such
things scare you or feel like an insurmountable barrier preventing you from
making a change. (And if you need help getting code under test, which is a
valuable thing to do, Michael Feathers has a great book *Working Effectively
with Legacy Code* that's all about that.)

And if you change the 30 to a 60, and then do the restructuring, except maybe
the excessive `comment-you-authored?` function, and commit it all in one commit,
I'm not going to bother you, despite what I led this section with. I trust your
competence to do such basic stuff flawlessly, every time. The method is there
when things aren't quite so basic, which can happen rapidly.

I do think a lot of programmers, Lispers and non-Lispers alike, will surely
recoil at this refactor. Look how much extra code I just added! So much
repetitive argument passing! And it's doing the same thing! Why on earth did I
do that? I just needed to change a single character and be done with it.

Sure, and sometimes doing that single character change is the right thing to do.
I submit that the tradeoffs are in favor of the refactor though. Once it's done,
it's still just a one character change for the driving issue that made you look
at it to begin with, but we've bought a bit more for our trouble:

* We can add explicit tests on the edit policy, rather than having to setup
  mocks or infer side-effects via `perform-edit` being called or not.
* When business logic like this changes, it's likely to change again. I think
  with this refactor it'll be easier to find the code that needs to change. I
  wouldn't even find fault if someone suggests replacing the hard-coded integer
  with a named constant. If the change is more drastic, we're starting out in a
  nice function call to do new computation in, rather than somewhere in the
  `when` conditional.
* I find it nicer to read. It's not like the original is at all unclear, but the
  nice little name helps contextualize it, and if we find out we need a similar
  conditional check somewhere else, we have a nice example of what the right
  conditional is without having to try and rederive it.

There's plenty of room for disagreement.

## Exporting Footnote

One thing you appreciate coming from Java or C++ or PHP is that Lisp doesn't
have that public/private/protected/friend/package-private nonsense. Python gets
rid of that, too, yay.

But I think what Lisp did do, as far as "public" interfaces go, was wrong, and
Python did it right. That is, in Python, when you import something, all of that
file's top level classes and functions are "exported" and trivially available to
you via the dot syntax, `foo.bar`. Same thing with an object's attributes. Same thing with the imported file's `from foo import *` lines.
But in Lisp, only the exported symbols of the package are trivially available as
`foo:bar`. If you don't export it, you have to type `foo::bar`. It's annoying.

I more or less understand why they did it: symbols get interned automatically to
the package, you don't want to re-export most of them, especially given the giant list of symbols that comes from using the `common-lisp` package.
I still wish things were different.

There's a convention to prefix stuff with `%` when it really is meant to be
non-public and furthermore not to be relied upon never changing. It's not a bad
convention, if you really are communicating that then cool, but it's a bit
unnecessary when the default is such symbols aren't exported to begin with, so
you don't need any extra trick.

Since in Python things are "exported" by default, it does have a trick to hide
some of them a bit, just put a double underscore in front of them and refer to them like
`self.__attrib`. But it's a total hack, you can still get at the property from
the outside if you type `obj._ClassName__attrib`.

In a language like CL where a certain book author put "kludge" in the index
followed by a range covering each page, I wonder if some sort of similar kludge
couldn't have been done at the standardization time to make packages just a bit
nicer to work with at the cost of an ugly and unprincipled hack like Python's
`__` or something else. Like, export everything but the special `cl` symbols, or export everything but symbols that come from imports or used packages, or do
that but allow re-export with an explicit re-export tag somewhere...

## Importing Footnote

It's not a good idea to `:use` most packages, so instead you should import the
symbols you need, right? Sure, but not necessarily always. Having symbols
be imported can hurt readability! How? Because if the context is not clear, a
reader can't from inspection know if this symbol was defined somewhere in the
current package or comes from somewhere else. That distinction matters,
especially if the symbol name is not particularly distinctive or even clashes
with some common names from other libraries or from CL itself.

Excuse me for picking on
[FSet](https://gitlab.common-lisp.net/fset/fset/-/wikis/FSet/Tutorial) for a
moment. The `fset-user` package for the tutorial harms more than aids
understanding, I think. First off, the use of `isetq` is unhelpful. Just use
`defvar`, dude. You can leave the earmuffs off, too, no one will care. Nowhere
in the tutorial are you using `isetq` to redo a binding on an existing variable
to store a new value. If you were, you could use the normal `cl:setq` or
`cl:setf` after the first `defvar`, or just use `defparameter` throughout. More
typing, sure, but we have tab completion. Or at least I do, don't you? So it's
not a burden on the write side, and it's not a burden on the read side.

Any considerations that the variable is special are irrelevant here. If it
actually matters, use a principled approach to making a global binding
non-special like in [global-vars](https://github.com/lmj/global-vars).

And to bully that project a bit: "Global variables are also more efficient than
special variables, especially in the presence of threads." Yeah ok but how much
more efficient? Few people ever do benchmarks... The SBCL manual doesn't even
say, despite it helpfully giving relative performance when it comes to slot
access via `slot-value` vs. an accessor. The LispWorks manual says the speedup
will be "pretty small overall in most cases", as expected, and "not obvious" for
one particular option available.

I'm only willing to believe it can matter overall in some circumstances because
Clojure switched from special-by-default, like CL, to non-special-by-default,
like every other language. But this could have more to do with the JVM than
special vars in general. (It'd be interesting to test ABCL on this, though this
library doesn't wrap any implementation-specific feature for ABCL, so perhaps it
doesn't have any.) Even when you opt-in to using a special var, some Clojure
people will raise a fuss and say you're hurting performance -- do they realize
what language they're using and the cost of their data structures?

Besides the syntactic ugliness of repeatedly threading the same variables
through multiple functions for a sense of functional purity -- ugliness which
special variables can alleviate -- is it actually more efficient? (Don't
underestimate the ugliness! I think part of people's instinctive recoiling from
the extract function refactor example given in the first footnote is largely
that.) I believe the JVM calling convention is stacks everywhere, so you're
repeatedly pushing and popping this thing off. Maybe JVM's JIT inlining will
save you? Not that it matters much because stack memory is typically in L1 cache
and not shared. I'm happy to continue believing it's more efficient in general,
especially if multiple threads are in play (memory barriers are cheap but not
free). Just, again, typically no benchmarks... And it's difficult to reason from
hardware principles here around the special var being in L1 cache or not and any
branch predictions on there being a special binding override being accurate or
not.

Back to FSet. Later in the tutorial, there's this:

> FSet provides a constructor macro which is simply called `set`.
> (We have shadowed `cl:set`, which is archaic anyway

Ok, but do you see how `(isetq s1 (set 1 2 3))` is mixing up two different
meanings of the English word "set"? That's confusing! Call `cl:set` archaic all
you want, but `setq` and `setf` that are built on top are just as archaic, and
here you are using a variant of `setq` anyway.

A curious non-beginner will now wonder what other `cl:` symbols they can no
longer rely on having the standard meaning, if they see them unadorned. From
FSet's `defs.lisp` file:

```lisp
  ;; For each of these shadowed symbols, using packages must either shadowing-
  ;; import it or shadowing-import the original Lisp symbol.
  (:shadow ;; Shadowed type/constructor names
	   #:set #:map
	   ;; Shadowed set operations
	   #:union #:intersection #:set-difference #:complement
	   ;; Shadowed sequence operations
	   #:first #:last #:subseq #:reverse #:sort #:stable-sort
	   #:reduce
	   #:find #:find-if #:find-if-not
	   #:count #:count-if #:count-if-not
	   #:position #:position-if #:position-if-not
	   #:remove #:remove-if #:remove-if-not
	   #:substitute #:substitute-if #:substitute-if-not
	   #:some #:every #:notany #:notevery)
```

Yeah, I'm going to *always* use the `fset` package prefix, thanks. I don't mind
that they've shadowed these, I think it's great to use intuitive names -- but
they're only intuitive if you know you're entirely in the FSet context. When you
say `fset:set` or `fset:count`, there's no missing that context.

Yes, technically if it's an unadorned `count`, if I'm ever confused about
whether it's the normal `cl:count` or not  I can always with a single editor
interaction resolve my confusion quickly. Just describe it. (Or in some cases,
just hover your mouse over it.) So in practice, importing such a symbol isn't
often that big of a deal. But if you're doing this describe action all the time,
instead of just knowing by inspection, something is wrong, you're not living in
habitable code and that will hamper your productivity and enjoyment. Also,
you're not always going to be looking at code through the lens of your editor.
(As an aside, it would be cool to see an AR demo that, using AI or not, gives
you full "IDE boosted intelligence" everywhere code comes up, even if you're
looking at a hard printout.)

Another benefit of keeping the package names, rather than importing everything:
findable relevant symbols through tab completion. It's sort of like a poor-man's
dot notation from other languages. You can type `(alex<TAB>` and get
`(alexandria:`. From there type `d<TAB>` and you'll get a list of all the fuzzy
matches of exported symbols in the alexandria package that match the letter d;
for me the first one is `alexandria:deletef`. More practically I appreciate it
for immediately seeing the useful methods relevant to a class, if they're all
kept in their own package to form an API; much nicer than doing a
cross-reference on who-specializes the class.

Sometimes the package name really is quite long, even with tab complete to rid
most the tedium of writing. Long names eventually raise the tedium of reading,
they turn into an annoyance instead of the intended helping to establish
context. I'm frequently guilty of long package names too, because I think Java
got it right to have package names be reverse-domain-name encoded, so I have
things like `com.thejach.thing...`. Fortunately, package-local-nicknames are
everywhere now, you can alias that to just `thing`. (Alas, slimv doesn't
[play nice](https://github.com/kovisoft/slimv/issues/128) with this yet.)
(Update: slimv now *does* support local nicknames... but in exchange for giving up fuzzy completions. I have these two lines in my vimrc:
`map \fuzz :let g:slimv_completions='fuzzy'<cr>` and `map \nofuzz :let g:slimv_completions='swank'<cr>` that let me toggle between the two modes; usually it's
in fuzzy mode.)
