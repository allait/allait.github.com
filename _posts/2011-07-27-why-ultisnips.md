---
layout: post
title: Why UltiSnips?
summary: UltiSnips is a Vim plugin that, while being easy to use, is incredibly feature rich.
---

When it comes to snippets, Vim has a somewhat large number of available plugins. Not necessarily
the most popular, but definitely the most mentioned seems to be snipMate. Now, snipMate is
a great plugin: it's minimalistic, it has simple snippets syntax and it does its work really
really well. But it doesn't do much, and judging from the lack of activity in the official
repository, it doesn't look like it will learn to do more any time soon (although, to be fair,
there is some activity on the many forks on github).

To be more specific, let's take a look at snipMate's own docs. "Disadvantages" section lists
some of the features snipMate lacks: `$0`, nested placeholders, multiple-line placeholders and regex
transformations of variables. These are great examples of functions that usually don't get in
the way, but may really come in handy when you need them.

UltiSnips is a Vim plugin that, while being as easy to use as snipMate, is much more feature
rich and, along with many unique features, implements everything listed above.

## Getting UltiSnips

You can download UltiSnips from the [Vim website][vimscripts] or the official bzr
repository on [launchpad][]. Installation is quite straightforward and is explained on the
[script page][vimscripts].

If you're using [pathogen][] and git submodules to manage your Vim plugins, there's the
[vim-scripts github repository][vimscripts-github] and
[official github mirror][github-clone].

UltiSnips is written mostly in python and therefore requires Vim compiled with python support. This
means that it may not work where snipMate did and is expected to be somewhat slower. In practice, I
haven't experienced any slowdowns and UltiSnips feels as fast and snappy as any snippet plugin I've
used.

By default, UltiSnips loads `.snippets` files from `UltiSnips` directories in your `runtimepath`.

[vimscripts]: http://www.vim.org/scripts/script.php?script_id=2715
[launchpad]: https://launchpad.net/ultisnips
[vimscripts-github]: https://github.com/vim-scripts/UltiSnips
[github-clone]: https://github.com/SirVer/ultisnips
[pathogen]: http://www.vim.org/scripts/script.php?script_id=2332

## Features

[UltiSnips documentation][ultisnips-docs]
covers everything the plugin may do, but I'll try to highlight the most interesting features here.

As I've mentioned, UltiSnips supports multi-line and nested placeholders, `$0` (last tabstop) and
placeholder text transformations. Most of the syntax for placeholders is the same as TextMate's.

Other notable features include:

* Ability to add filetype-independent snippets.
* Support for Vim dotted filetype syntax.
* Ability to "include" snippet files.
* Command to show the list of available snippets.
* Support for multi-word and regular expression triggers.

Apart from this UltiSnips has two extremely powerful tools: "options" and interpolation.

[ultisnips-docs]: http://bazaar.launchpad.net/~sirver/ultisnips/trunk/view/head:/doc/UltiSnips.txt

### Options

Options, among other things, allow you to specify for each snippet whether it will be expanded only
at the beginning of a line, in the middle of the word, at word boundary or not. This, combined with
the option to overwrite all previously defined snippets with the same trigger, allows you to use the
same trigger to expand different snippets in different conditions.

### Interpolation

Interpolation allows you to execute arbitrary code and insert it's output in your snippets.
While snipMate allowed you to call a Vim function, UltiSnips supports shell code, VimScript
and python code. There's also a way to create standalone python functions for use inside python
code blocks.

## Usage examples

A list of features sounds really nice, but sometimes it's hard to imagine how all these things may
be useful. In this section I'll try to demonstrate few small examples of some of the
UltiSnips features. Apart from giving you some ideas about how each feature my be used, they also
might serve as additional documentation of the snippet syntax.

### Different beginning-of-line snippet

If you've ever used Django ORM (or any other ORM that requires you to describe model fields)
you might be familiar with the following syntax for field definition:

    name = models.CharField(max_length=255)

Now, if you want to write a snippet `char` for inserting this line, you have two options:

1. Expand `char` trigger to `${1:FIELDNAME} = models.CharField(${2:...})`
2. Expand `char` trigger to `models.CharField(${1:...})`

The first one works great at the beginning of line, but is cumbersome if you've already typed the
desired field name either by mistake or because you haven't decided on the field type yet. The second
definition saves a bit less time and won't allow you to define a default field name in case you need
one.

Now, the good news is, as you might have guessed from the header, it is possible to define both and
make UltiSnips use the first one only on the beginning of line and the second one everywhere else.
To do this, we start with defining the shorter snippet:

    snippet char "char field"
    models.CharField(${1:...})
    endsnippet

Then, we define the beginning-of-line snippet:

    snippet char "char field" !b
    ${1:FIELDNAME} = models.CharField(${2:...})
    endsnippet

As you can see, the second snippet has two characters `!b` after the description. These are snippet
options: `b` means that snippet should be expanded only at the beginning of a line and `!`
overwrites all previously defined snippets with the same trigger. Without `!` there would be
two possible expansions for `char` at the beginning of a line so UltiSnips would show a list of
available snippets before expansion. And since beginning-of-line snippets can't be used after you've
typed a field name, the short snippet remains the only available option there and will be expanded without
further questions. It is important that the beginning-of-line snippet is defined after the short
snippet, since `!` only overwrites previously defined snippets.

### Inserting current date

This is a simple example of interpolation. Let's define a snippet `date` that will insert the
current date:

    snippet date
    `!v strftime("%Y-%m-%d")`
    endsnippet

Backticks indicate interpolation and `!v` means that we're using vimscript. `strftime` is the name
of Vim function that returns date and time in the specified format. If you want this snippet to be
available in files of any type, you can define it inside `all.snippets` file (analogous to
`_.snippets` file in snipMate) in one of the directories in snippet search path.

### Documenting function arguments

There is an advanced example of interpolation usage in [python.snippets][] file bundled with
UltiSnips. `smart class` and `smart def` snippets use python interpolation to insert a special line
into function docstring for each function argument. I won't explain the snippet here, but together
with the `box` snippet from [all.snippets][] it shows how much can be achieved with python
interpolation in UltiSnips. I haven't made much use of it yet, but it seems that such snippets may
be a viable alternative for some types of Vim plugins.

More snippet examples can be found in snippets bundled with UltiSnips.

[python.snippets]: http://bazaar.launchpad.net/~sirver/ultisnips/trunk/view/head:/UltiSnips/python.snippets
[all.snippets]: http://bazaar.launchpad.net/~sirver/ultisnips/trunk/view/head:/UltiSnips/all.snippets

## Porting snipMate snippets

UltiSnips comes with a nice collection of snippets, but if you've been using snipMate for a long
time, there's a chance that you've grown accustomed to snippets that aren't readily available in
UltiSnips. The good news is that, at the moment, pretty much every snippet can be converted from
snipMate to UltiSnips format in few easy steps.

The main difference in snippet file syntax is that snipMate uses indentation to find the end of the
snippet text, while UltiSnips requires and explicit `endsnippet` line at the end. Overcoming this
difference is easy: remove leading indentation from snippet text and add `endsnippets` line at the
end of the snippet, converting

    snippet example
            example snippet text

to

    snippet example
    example snippet text
    endsnippet

The second common difference is the snippet description text, which should be quoted in UltiSnips:
`snippet try Try/Except` should be converted to `snippet try "Try/Except"`.

At last, if you have snippets that call VimScript functions, you should either prepend them
with `!v` to indicate VimScript interpolation (`strftime("%Y-%m-%d")` should be converted to
`!v strftime("%Y-%m-%d")`) or rewrite them using one of the other UltiSnips interpolation types.

This should cover most of the differences and make the transition painless. There's a chance that
some non-trivial snippets may require some additional changes, but the above steps were enough for
everything I've tried to convert.
If you're converting snippets from `_.snippets` file, don't forget that you should rename it to
`all.snippets` to make snippets available regardless of the filetype in UltiSnips.

**UPDATE:** It looks like the latest version of UltiSnips comes with a script for automatic snippet
conversion from both snipMate and TextMate formats.

## Conclusion

[UltiSnips documentation][ultisnips-docs] contains a lot more information about what UltiSnips can
and can't do and you should probably consult it if you have any questions.
