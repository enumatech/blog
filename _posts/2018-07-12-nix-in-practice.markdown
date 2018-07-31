---
layout: post
title:  "The Nix package manager in practice"
date:   2018-07-12 17:38:08 +0800
categories: update
author: Tamas Herman
---

Have you ever wondered why do you need to install broadly used development tools
in different ways on different OSes?

Python, Ruby, Node.js, Clojure, just to name a few.

Wouldn't it be nice if you could learn one method which works on macOS, Ubuntu, Arch,
Amazon Linux, Docker containers?

<!--more-->

# Enter the Nix package manager!

I've noticed a [claim](https://clojureverse.org/t/from-twitter-how-do-we-sell-clojure/1184/15?u=onetom) the other day on a forum from a fellow Clojure enthusiast:

> Agreed. Except that install Node.js on Ubuntu is not hard. Linux users get better command line tools:
> 
> https://nodejs.org/en/download/package-manager/#debian-and-ubuntu-based-linux-distributions
> ```
> curl -sL https://deb.nodesource.com/setup_9.x | sudo -E bash -
> sudo apt-get install -y nodejs
> ```

My replies to him warrant a blog post, so I've assembled them below.

It is indeed not hardto install Node.js like that. I would rather say it's _horrible_! :)
It's not reproducible later and it's also Linux distribution specific!

On the other hand if you are opening yourself up to the dangers of `curl | sudo bash`,
you might as well install the [Nix package manager](https://nixos.org/nix/)
with `curl https://nixos.org/nix/install | sh`.
Once that's done, obtaining a shell process, which provides you with `node`, `npm`, `clojure`, `clj` and `lein` on the `PATH`, becomes ridiculously simple, **regardless of which OS are you using**! It can be macOS, Ubuntu or Amazon Linux (I have no experience with Windows):

```
nix-shell -p clojure leiningen nodejs-10_x
```

It will absolutely not interfere with your existing system, because the PATH is only constructed
ephemerally for this new shell process you just started, eg:

```
$ nix-shell --pure -p coreutils --run 'echo $PATH'

/nix/store/ckq71kkymh1ji2b44xn80wmr7fmi6wr5-clang-wrapper-5.0.2/bin:/nix/store/bcl9zj60h52p47dy85s326mdrqx52417-clang-5.0.2/bin:/nix/store/5q51r2d0xzs4hi8rb661drpf83mhm4b4-coreutils-8.29/bin:/nix/store/qk4fdnzkr8r0gihgyps7x07cpyffvkmy-cctools-binutils-darwin-wrapper/bin:
...
```

The software you declared, with all its supporting files and transitive dependencies,
down to `libc`, are all stored under `/nix/store` in read-only directories.
These directories have a hash in their name to avoid any version conflict,
even between seemingly identical versions of software, which only differ
in the versions of libraries they are linked against! eg.:

```
$ ls -lahd /nix/store/*-bash-4.4-p12*
dr-xr-xr-x 4 root nixbld  128 Jan  1  1970 /nix/store/axikcsz4wh2qpi5zmlfsmm4jx8wm8s1g-bash-4.4-p12/
-r--r--r-- 2 root nixbld 4.6K Jan  1  1970 /nix/store/c8adxvdnrsqa6v48hagqfab6iv4l35sb-bash-4.4-p12.drv
-r--r--r-- 2 root nixbld 4.3K Jan  1  1970 /nix/store/djpy3089x8ij20dw5zvwvr2ym8iryhl1-bash-4.4-p12.drv
dr-xr-xr-x 4 root nixbld  128 Jan  1  1970 /nix/store/pkjmwq7sqrvjg7cjiph6hq0khsmfl6p8-bash-4.4-p12/
dr-xr-xr-x 4 root nixbld  128 Jan  1  1970 /nix/store/rjglqbbmg27dwwyyqsnn62jcz6qwxkli-bash-4.4-p12/
drwxr-xr-x 4 root nixbld  128 Jan  1  1970 /nix/store/s8mff1kmnc63b21ybdid2ni0fw7mzy7r-bash-4.4-p12/
```

Now this process is still not any more reproducible than the `apt-get` above,
but you can just provide a complete package tree archive (which is around ~10MB)
to `nix-shell` (via a URL or file reference) and it's guaranteed it will build the
exact same binaries today and one decade from now.

You can retrieve Nix package tree archives as [tarballs of a very specific commit](https://developer.github.com/v3/repos/contents/#get-archive-link)
from the [nixpkgs](https://github.com/nixOS/nixpkgs) package tree source repo, eg:

https://api.github.com/repos/NixOS/nixpkgs/tarball/1539167d215841a966b8395c1025d66812d63d31

or you can pick such a version combination of all those 6500+ packages in the tree,
which has been tested [to a degree of your liking](https://nixos.org/nixos/manual/#sec-upgrading):

```
curl -sI https://nixos.org/channels/nixpkgs-unstable/nixexprs.tar.xz | awk '/Location:/ {print $2}'
```

then feed this package tree into your `nix-shell`:

```
nix-shell -I nixpkgs=https://d3g5gsiof5omrk.cloudfront.net/nixpkgs/nixpkgs-18.09pre144939.14a9ca27e69/nixexprs.tar.xz -p clojure leiningen nodejs-10_x
```

If you give this command to your friends, they will also get a shell
with the exact same software versions, be it today or next year
and regardless whether they have, macOS or some flavor of Linux...

Since this command line is getting a bit unwieldy, you should just put
this information into a `shell.nix` file at the top of your projects like this:

```
# curl -sI https://nixos.org/channels/nixpkgs-unstable/nixexprs.tar.xz | awk '/Location:/ {print $2}'
with import (builtins.fetchTarball "https://d3g5gsiof5omrk.cloudfront.net/nixpkgs/nixpkgs-18.09pre144939.14a9ca27e69/nixexprs.tar.xz") {};

mkShell rec {
  buildInputs = [
    clojure leiningen nodejs-10_x
  ];

  shellHook = ''
    export PATH="$PATH:$PWD/node_modules/.bin"
    '';
}
```

Then you can just run `nix-shell` from the directory where the `shell.nix` file is.

The extra `PATH` magic is there, so you can just `npm i -D mocha` locally into your project and run your tests by simlpy typing `mocha`.
None of that dirty `npm i -g  ...` nonsense is necessary anymore!

Pro tip:  Run `nix-shell --pure` to drop your default system `$PATH`
and you will see if you are accidentally depending on something from your global, host OS installation.

Further reading:

1. Nix Pills — small intro tutorial chapters into Nix and Nixpkgs
   https://nixos.org/nixos/nix-pills/

1. https://nixos.org/

