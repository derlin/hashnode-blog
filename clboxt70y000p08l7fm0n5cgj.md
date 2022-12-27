# bitdowntoc: markdown TOC for all (even BitBucket 😉)

[![favicon](https://cdn.hashnode.com/res/hashnode/image/upload/v1671099842024/rO_0AJdS7.png align="left")](https://derlin.github.io/bitdowntoc/)

## Kezaco ?

[bitdowntoc](https://github.com/derlin/bitdowntoc) (**bit**bucket mark**down** **toc**) is a flexible **Markdown TOC Generator**, available both as a [standalone jar](https://github.com/derlin/bitdowntoc/releases/tag/nightly) and as a [free online service](https://derlin.github.io/bitdowntoc/).

## Why use it?

Contrary to other tools such as [GitHub Wiki TOC generator](https://ecotrust-canada.github.io/markdown-toc/), it supports many different hosting platforms. More importantly, it can also generate working TOCs in **BitBucket Servers** that lack even the ability to add anchor links to titles in READMEs.

Here are the options to use for the most well-known platforms (see [bitdowntoc's README](https://github.com/derlin/bitdowntoc) for more info):

*   **BitBucket Server** → generate anchors
    
*   **GitLab** → concat spaces, do not generate anchors
    
*   **GitHub** → do not concat spaces, do not generate anchors
    

You can use the `[TOC]` placeholder to control where the TOC will be located. Header(s) above it will be ignored.

You can regenerate your TOC anytime, bitdowntoc is smart enough to detect the changes.

:no\_entry\_sign: :wink: *Everything is generated locally,* ***no tracking*** *or anything of the sort !*

## Web interface tricks

**Go fast** Paste your markdown content, click *generate* then *copy*. Done, the TOC-ified markdown content is in your clipboard.

**Save your preferences** Have a look at the options (*options* button) and select your preferences once, then click *save* to save them to local storage. They will be loaded automatically next time.

**Light/dark theme** If you prefer dark interfaces, click on the sun/moon icon on the top right.

## Why I did it

I got the motivation from the lack of existing tools supporting BitBucket Server.

Indeed, BitBucket Server (at least at version 6), doesn't generate heading IDs for Markdown. The only way I found to have a nice TOC was to create the anchors myself, using: `<a name="some-heading">`. This is tiresome to do manually. I found [this blog](https://rderik.com/blog/generate-table-of-contents-with-anchors-for-markdown-file-vim-plugin/) talking about a Vim plugin doing it for you, but this requires Vim (obviously) and is just a branch in some repo. Hence bitdowntoc!

## What I learned

This project was a good excuse to play with **Kotlin MPP** (**M**ulti **P**latform **P**roject). I implemented the TOC generation logic in a *common module*, which is then used by a JVM module for the CLI and a JS module for the web interface.

I was amazed by how great it felt to "write JS" using all the wonderful features of kotlin (type-safety, extension functions, generics, ...). I didn't encounter any real difficulties (which is amazing), except maybe two things:

*   regexes (used in the common module) will work differently depending on the runner (JS/browser vs JVM), and
    
*   tests in common module are limited (e.g. not possible to read a file).
    

To find out more, [check the code](https://github.com/derlin/bitdowntoc)!

* * *

Written with ❤️ by [derlin](https://github.com/derlin)