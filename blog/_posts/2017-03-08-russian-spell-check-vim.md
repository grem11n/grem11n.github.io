---
title: How to add Russian spell check to VIM
featured: images/blog/vim.svg
date: 2017-03-08 21:07:30 +0200
categories: blog
layout: post
---

You can implement a spell check in your VIM installation in a moment, it you need it (I do).

Just type a command:

```
:setlocal spell spelllang=en_us
```

Or add it into your vimrc file.

However, you may want to check a few more languages. This is simple as well:

1.  Go to [VIM's spell archive](http://ftp.vim.org/vim/runtime/spell/) and find desired language pack
2.  Download it, for example, into ~/.vim/spell. Create a directory, if it doesn't exist.
3.  Update your vimrc settings. In my case it is:
```
setlocal spell spelllang=en_us,ru_ru
```

Congratulations! You're all set!

