---
layout: post
title:  "PDF processing with Nix"
date:   2018-07-31 09:38:08 +0800
categories: update
author: Tamas Herman
---

Have you ever wished if you could just convert a logo you got in a PDF file
into an SVG, so you can put up on your website and it would look good on
Retina screens too?

Have you ever marvelled the beautifully formatted research papers in PDFs,
but you didn't want to invest into learning TeX or get tangled in installing TeX?

Nix to the rescue!

<!--more-->

It's worth emphasising that you DO NOT need NixOS to do the following.
A macOS or most Linux kernel based systems will work.

You don't have to be a guru to use Nix AND on top of that your knowledge
is OS agnostic.

If you happen to hit some obstacles with Nix in your specific use-case,
you can always go back to solving the problem in an OS / distro
specific way, because having Nix on your system will not interfere
with your underlying OS setup (beyond eating up extra disk space).

# PDF to SVG conversion

Here is a real-life example from the other day:

I had to convert a logo sent to me in a PDF file into SVG.
I googled for _"open source command line  pdf to svg"_
Turns out there is indeed a `pdf2svg` tool, so I just did

```
$ nix-shell -p pdf2svg  --run 'pdf2svg --help'
these paths will be fetched (5.74 MiB download, 31.79 MiB unpacked):
  /nix/store/2li666k7669n3i8n0wbgda41znxzif6s-libjpeg-turbo-1.5.3
...
Usage: pdf2svg <in file.pdf> <out file.svg> [<page no>]
```

then problem solved:

```
$ nix-shell -p pdf2svg  --run 'pdf2svg logo.pdf logo.svg'
```

Everyone should know at least this much about Nix...

I forgot to mention in my previous article one more practicality.
Install a faster, nicer package search utility with `nix-env -i nox`,
so you can find package names easily:

```
$ nox pdf2
1 pdf2djvu-0.9.9 (nixpkgs.pdf2djvu)
    Creates djvu files from PDF files
2 pdf2odt-20170207 (nixpkgs.pdf2odt)
    PDF to ODT format converter
3 pdf2svg-0.2.3 (nixpkgs.pdf2svg)
    PDF converter to SVG format
4 pdf2xml (nixpkgs.pdf2xml)

5 python2.7-pdf2image-0.1.14 (nixpkgs.python27Packages.pdf2image)
    A python module that wraps the pdftoppm utility to convert PDF to PIL Image object
6 python2.7-PyPDF2-1.26.0 (nixpkgs.python27Packages.pypdf2)
    A Pure-Python library built as a PDF toolkit
7 python3.6-pdf2image-0.1.14 (nixpkgs.python36Packages.pdf2image)
    A python module that wraps the pdftoppm utility to convert PDF to PIL Image object
8 python3.6-PyPDF2-1.26.0 (nixpkgs.python36Packages.pypdf2)
    A Pure-Python library built as a PDF toolkit
Packages to install:
```

# Turn a Markdown document into PDF

Here comes another, maybe even more powerful example.

We wrote a grant application in [Notion](https://www.notion.so).
Exported the document into markdown and turned it into a professional looking PDF:

```
nix-shell -p texlive.combined.scheme-small pandoc \
    --run 'pandoc grant.md -V geometry:margin=1in -o grant.pdf'
```

My colleague did it the 1st time on his Arch Linux, then after I amended the Notion page,
did the export again and I ran the exact same command on my macOS and got an updated
PDF.

Now it's your turn; give it a try!

Let's pick the ClojureScript `README.md` from github for example and see it for yourself how would it look if you would have printed it!

```
$ curl -s https://raw.githubusercontent.com/clojure/clojurescript/master/README.md | \
   nix-shell -p texlive.combined.scheme-small pandoc \
         --run 'pandoc <(cat) -o README.pdf'; open README.pdf
```

_Note: It will take a few minutes for the first time to download ~40MB of TeX of course, but it happens unattended and will work for sure!_