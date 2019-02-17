---
layout: post
title: "Python and Vim"
comments: True
permalink: "python-and-vim"
---

Now days I am spending a quite a bit of time with Python from doing hobby project to implementing algorithms and data structures to writing Apache Spark code. Python is one of my favorite language. It's versatile easy to grasp and the logic behind the code can be expressed with simplicity and elegance. The Zen of Python from Tim Peters says all about Python. Zen of Python can be read in Python shell with the command ```import this```.

I wrote Python with [PyCharm](https://www.jetbrains.com/pycharm/) and with two current popular text editor [Atom](https://atom.io/) and [Sublime](http://www.sublimetext.com/). PyCharm is best editor though sometime it feels slow. And both Atom and Sublime has good plugins that makes it simple and easier to work with Python.

If you are a Vim user and comfortable with Vim basics Vim could be used as complete Python IDE and there are very good Vim plugins support which makes working with Pyhton in Vim make smooth. Below are the steps for getting started to work with Vim.

Most of the `*nix` systems comes with Vim pre-installed. To check the Vim version and the supported Python:
```
vim --version
```
 The output from the above command should show:

>The Vim version should be at least > 7.3.
> There should be +Python under feature included

One important things to note is the Pyhton version. Vim in Ubuntu 14.04 doesn't come up with Python3 and we cannot use both Python2 and Python3 in same Vim session. To use Vim with Python3, we have to compile it with Python3 support. [Here](http://www.xorpd.net/blog/vim_python3_install.html) is steps to compile Vim with Python support.

### Vim Extension Manger

There are several extension manager available in Vim but the [Vundle](https://github.com/VundleVim) is quite popular one. It makes updating and installing Vim extensions lot easier. Below are the instructions from Vundle github repo to setup Vundle:

```
git clone https://github.com/VundleVim/Vundle.vim.git ~/.vim/bundle/Vundle.vim
```

Now create the `.vimrc` file under user home directory and add the following lines:

```
set nocompatible              " required
filetype off                  " required

" set the runtime path to include Vundle and initialize
set rtp+=~/.vim/bundle/Vundle.vim
call vundle#begin()
" alternatively, pass a path where Vundle should install plugins
"call vundle#begin('~/some/path/here')

" let Vundle manage Vundle, required
Plugin 'VundleVim/Vundle.vim'

" All of your Plugins must be added before the following line
call vundle#end()            " required
filetype plugin indent on    " required
```

Now the Vundle is configured and we can open the Vim and start the Vundle to do all the magic. Inside the Vim type the command:
```
:PluginInstall
```

Now we can start adding extensions to Vim:

### Code Folding
We can have the Code folding feature of most modern IDE in Vim. [SimplyFold](https://github.com/tmhedberg/SimpylFold) is one such Vim extension. Add following line to `.vimrc`:
```
Plugin 'tmhedberg/SimpylFold'
```
And open the `vim` and run `:PluginInstall`.

### Python Indentation
For the `PEP8` indentation add the following line to the `.vimrc`:

```
au BufNewFile,BufRead *.py
    \ set tabstop=4
    \ set softtabstop=4
    \ set shiftwidth=4
    \ set textwidth=119
    \ set expandtab
    \ set autoindent
    \ set fileformat=unix
```
This will give us 4 spaces when hit tab and line length doesn't cross 120 characters(80 char is standard but with wider display we can go little more) and stores in unix format to avoid conversion issues.

### File Browsing
NerdTree(https://github.com/scrooloose/nerdtree) provides the tree like file browser. Again to add using vundle add following lines to `.vimrc`:
```
Bundle 'scrooloose/nerdtree'
```
And open `vim` run `:PluginInstall`.

### Python Mode
Python-mode(https://github.com/klen/python-mode) adds lots of feature such as Lint, codecompletio, document lookup, jump to classes, refactoring tools.

The bundle for `python-mode` is:
```
Bundle 'python-mode/python-mode'
```
And run `:PluginInstall` to install. The complete documentation is provided on github or from inside the vim run `:help python-mode`. Below are the some settings that can be set for `python-mode`

```
let g:pymode_rope = 1

" Documentation
let g:pymode_doc = 1
let g:pymode_doc_key = 'K'

"Linting
let g:pymode_lint = 1
let g:pymode_lint_checker = "pyflakes,pep8"
" Auto check on save
let g:pymode_lint_write = 1

" Support virtualenv
let g:pymode_virtualenv = 1

" Enable breakpoints plugin
let g:pymode_breakpoint = 1
let g:pymode_breakpoint_bind = '<leader>b'

" syntax highlighting
let g:pymode_syntax = 1
let g:pymode_syntax_all = 1
let g:pymode_syntax_indent_errors = g:pymode_syntax_all
let g:pymode_syntax_space_errors = g:pymode_syntax_all

" Don't autofold code
let g:pymode_folding = 0
```

The above settings helps:
- Lookup Python docs using `k`
- Automatically check code on saving, but only use `PyFlint` or `PyFlakes`
- Support virtualenv
- Use <leader>b to add a pdb shortcut (inserts import pdb; pdb.set_trace() ### XXX BREAKPOINT into your code
- Enhanced syntax highlighting and formatting


These are the basic settings that makes using with Python. I found these pointers in different blogs and sites and was very helpful to setup Vim for Python, so I posted here so someone may find it useful.

#### Reference
1. https://wiki.python.org/moin/Vim
2. https://realpython.com/blog/python/vim-and-python-a-match-made-in-heaven/
