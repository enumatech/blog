---
layout: post
title:  "JS test swamp Part 1: Why (not) `jest`?"
date:   2019-01-09 14:20:00 +0800
categories: update
author: Tamas Herman
---

In this article series we will survey a few corners of the JavaScript testing
solution and try a few uncommon combinations of components, hoping
to increase testing ergonomics, while taming dependency weight and pushing
the runtime overhead of the complete test harness lower

<!--more-->

This first part of the series we will establish some concepts, to make sure we
have a shared mental model of the problem space, then look into the anatomy
of existing test frameworks and quantify some of their issues.

Further articles will evaluate potential remedies, aiming to reduce
specifically:

* cold-startup time
* size and number of dependencies
* overall complexity

without sacrificing (to much):

* expressiveness of test specifications
* precision and clarity of test failure reports

To achieve this we are going to deploy 2 tactics:

1. trade-off the simplicity of the getting started experience for more
   explicit setup and configuration.
1. increase decoupling, by extracting functionality which is already solved by
   simpler, more robust tools.


# Motivation

We switched from `mocha` to `jest`, because we were tired to add `chai` and
`chai` plugins, `sinon` for mocking, `nyc`/`istanbul` for test coverage
metrics and wire them together every time we started a new project.

`jest` is a very fancy test framework and thanks to its opinionated,
monolithic design, it frees us from the tedious work of fishing out good
packages from the sea of the myriad alternatives available on
https://npmjs.com.

With `jest` some decisions are made for us, saving time spent on hesitation
and arguments. We get `describe`/`context`/`it` BDD format, `expect` style
assertions and a custom, but well-documented module/object/function mocking
toolkit plus snapshot management for maintaining test fixtures, all integrated
with `nyc` to generate test coverage reports. 

All these benefits come at a cost of course: tons of  dependencies, implying
elevated number of bugs, bloated size and high levels of parochialism too.


# Quantification

Let's contrast the metrics of some frameworks and their underlying
building blocks:

| Framework | Version | Size | Packages | Contributors | Open issues  | Closed issues | Age |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| `jest` | 23.6.0 | **62MB** | 618 | 361 | 485 | 3498 | [3 years][jest age] (effectively) |
| `mocha`| 6.0.0-1 | **2.5MB** | 24 | 436 | 250 | 1957 | [7 years][mocha age] |
| `node-tap`| 12.1.1  | **39MB** | 450 | 233 | 66 | 236 | [7 years][node-tap age] |
| `tape`| 4.9.2 | **2.4MB** | 33 | 16 | 80 | 184 | [6 years][tape age] |
| `nyc`| 13.1.0 | **19MB** | 182 | 110 | ___ | ___ | [4 years][nyc age] |
| `istanbul`| 0.4.5 | **15MB** | 48 | 138 | ___ | ___ | [6 years][istanbul age] |
| `chai`| 4.2.0 | **1.1MB** | 7 | 20 | ___ | ___ | [7 years][chai age] |
| `unexpected`| 11.0.0 | **12MB** | 22 | 10 | ___ | ___ | [5 years][unexpected age] |
| `sinon`| 7.2.2 | **7.5MB** | 17 | 435 | ___ | ___ | [8 years][sinon age] |
| `proxyquire`| 2.1.0 | **668KB** | 7 | 10 | ___ | ___ | [6 years][proxyquire age] |

So whats wrong with having so many dependencies?
Typical reaction to that is: "storage is cheap".

Sure, but every time you start the framework, a lot of this code
has to be loaded, parsed and executed.

"Get a decent machine for development!", you would say?

How about a 2014, 4GHz i7?

```
$ npm --version
v10.12.0
$ mkdir /tmp/jest-drive/__tests__; cd /tmp/jest-drive
$ npm init -y
$ time npm i -D jest
       14.83 real        14.34 user         8.20 sys
$ echo 'it("works", () => expect(1).toEqual(1))' > __tests__/test.js
$ time npx jest test.js
        1.23 real         1.64 user         0.31 sys
        0.98 real         0.99 user         0.16 sys
        0.55 real         0.53 user         0.09 sys  // sometimes...
```

I'm not sure when or why it can run twice as fast sometimes, which is already
troubling to experience...

Well, I can run it in watch mode, so I only pay for this startup time once in
a while, right? That helps, but it still takes 0.2s to rerun the test on file
change, which is surprisingly slow. If you consider that your application
is also reliant on a bunch of bloated libraries – like `web3.js` for example,
which adds another 0.3s to the boot time –, then you just crossed the line
between test feedback which is considered instant vs laggy and annoying.

If the test would pull in less dependencies, it could be just started from
scratch every time, alleviating the need for the test framework to deal
with file-watching and ensuring a clean execution environment...

Again, you might argue that a file watcher is not so much overhead, but
don't forget that for a cross-platform solution, `jest` had to include the
`fsevents` library, which needs a C compiler, which is started from the
`node-gyp` Python build environment. That's another half gigabyte of
dependencies at least, which can break or differ between machines and
lead to subtle errors or incompatibilities.

Then CI machines have to obtain this 62MB first along with the whole Node.js,
build environment and upload/download it to their caches between builds.

All developer workstations have to download and have a copy of it too.
IDEs need to index it to provide convenient definition lookup and
auto-completion.

All-in-all it's just ridiculous to have so much cruft lying around, especially
when a project itself might only be a few 10s of kilobytes...


# Test framework anatomy

A testing toolkit consists of the following components:

1. plan
1. assertions
1. test doubles (aka. mocks, stubs, spies)
1. fixtures, factories or generators for fake data
1. runner
1. reporter
1. watcher
1. coverage

While these aspects of running tests are quite independent from each other,
for convenience sake test plan declaration, running and reporting are typically
implemented as one monolithic, integrated package.

Let's review these ingredients in more detail, one by one. 


## The test plan

Many framework separate test specification from running tests.
This makes it possible to pick only certain test cases for execution, even if
they are nested within complex scenarios.

As we will see later, such a strict separation is not essential and
if done well, the ability to nest, skip or focus test-cases can be
preserved.

`mocha` has several DSLs, so called [UI interfaces], but the
`describe`/`context`/`it`, aka BDD style is the most popular and also chosen
by `jest`.

To avoid confusion, we must note, that the BDD style also dictates assertions
being denoted by an `expect` function, which returns a matcher object with
chainable checker methods on it.


## The contract with assertion libraries

Assertion libraries have a simple contract with test runners:

> They throw an `AssertionError` specifically, when an expectation is not met
> and the `message` property of that error object contains the explanation of
> the failure.

This convention enables one to pick an assertion library independently from
the test framework and extend it with application-domain specific matcher
functions, to achieve highly expressive, concise test descriptions and
informative test failure output.

Unfortunately test-run reporters might not handle `message`s with newlines
and ANSI color code escape sequences very well.

Unfortunately there are no established rules about the structure of the
`message`. When it contains newlines or ANSI color code escape sequences,
 some test reporters might get confused and emit garbled output.

An assertion library can be graded by the following properties: 

* amount of setup boilerplate
* depth and breadth of default features
* simplicity of extension interface
* plugin availability
* expectation
    * expressivity
    * verbosity
    * composability
* failure explanation clarity

Most assertion libraries wouldn't score very high on most of these metrics.


## Test doubles and fake data

These test ingredients are usually independent from the test framework,
but nested test scenarios and property-based testing require some support.

Nesting test cases allow us to represent test scenarios concisely, when their
preconditions are similar. It lets us structure scenarios into a tree, where
we prepare preconditions gradually for a specific test case described on
a leaf of that tree.

There are 2 well-known techniques for exposing data, to the assertions, which
was generated during the precondition build up.

One is to use the implicit `this` value during execution as a shared context.

The other approach is to use the nestable lexical scope of the language to
declare shared variables and populate those from `before{All,Each}` hooks
during execution.

Snapshots are another way to deal with fixtures, which do require support
from both the assertion framework to create and update snapshots and
the test runner, so you can control whether you want to run in snapshot
update mode or not.

Later, when we look into the TAP ecosystem, we will see why all this is has a
significance.


## Test runners and test reporters

Again, most of the well-know libraries doesn't separate the test execution
from the reporting of summaries of execution results.

If we run test files sequentially, then their output can be concatenated and
summarized at the end.

However, when tests are run in parallel, running + reporting takes the shape
of a map-reduce operation over the test files.

In the parallel run case, reporting execution-progress requires some kind of
output-stream merging operation too.

Since test runners usually allow some selection/filtering of test cases too,
reporters might want to be aware of the result of the selection, especially
when they want to provide some progress reporting during test execution.
This makes decoupling these 2 steps tricky.  


## File watchers

Watcher functionality is OS dependent, so language ecosystems usually provide
some OS independent abstraction layer for it.

In Node.js world, the `fsevents` package is a popular pick for implementing
file-change detection.

While it very flexible to drive such libraries from the convenience of your
programming language, it doesn't come without its own problems.

As we mentioned earlier, `fsevents` requires a functioning Node.js build
environment too, which can get quite tricky to reproduce across OSes,
especially if we want to use the exact same Node.js versions on both macOS
and Linux, or we have multiple Node.js version "installed" on a developer
workstation.

We mitigated these issues by mandating the usage of the Nix package manager
for all of our projects.

It's easy to get started with. Install and activate nix, then start a
`nix-shell -p nodejs-10_x`. For a more permanent solution declare this
Node.js dependency of your project in a `shell.nix` file:

```nix
with import <nixpkgs> { };

mkShell {
    buildInputs = [nodejs-10_x];
}
```

For more details, check out our former articles on this topic:

* [The Nix package manager in practice][nix-practice]
* [PDF processing with Nix][nix-pdf]

File change detection is a quite independent problem from testing, so why is
it not an independent puzzle piece from test harnesses?

My guess is that developers desire to simplify the way you appoint tests for
execution, regardless of granularity. You can refer groups of tests by
pattern matching either against test files names or test scenario descriptions
and scenario metadata, regardless which files house the matching tests.

If our tests are fast and well-structured, so failures doesn't cascade, we
won't really miss the ability to sieve tests by matching against their
description.

Leaving out the description matching feature leaves us with a simpler,
more familiar problem to solve: selecting files.

For file selection there are a lot of utilities available on our systems
out of the box, like `bash` globs, `ls` and `find` and there are super
advanced, interactive ones, like [`fzf`], the "general-purpose command-line
fuzzy finder".

Building on those primitives and throwing a small, cross-platform utility
called [`entr`] into the mix results in a completely technology agnostic,
reusable, file-change monitoring construct.

While the globing built into shells are not as powerful as the [`minimatch`]
library which typically stands behind many libraries, in practice we lose
very little convenience.

`entr` is not present on most systems though, but if we assume a `nix`
environment being present, it's trivial to add `entr` to the `buildInputs`
list in our `shell.nix`.

Such a move would liberate a test framework from watching duties, further
reducing their complexity and dependency count.


## Coverage

[`nyc`] is a pretty unobtrusive tool, which doesn't really require any
explicit support from a test framework and as such they are better left
decoupled.

(At least at the moment I'm unclear on the motivation behind the existing
framework integrations. Might it be just the (perceived) increase in
convenience?)


# Author
[Tamas Herman](http://github.com/onetom/)


[jest age]: https://www.npmjs.com/package/jest?activeTab=versions
[mocha age]: https://www.npmjs.com/package/mocha?activeTab=versions
[node-tap age]: https://www.npmjs.com/package/tap?activeTab=versions 
[tape age]: https://www.npmjs.com/package/tape?activeTab=versions 
[nyc age]: https://www.npmjs.com/package/nyc?activeTab=versions 
[istanbul age]: https://www.npmjs.com/package/istanbul?activeTab=versions 
[chai age]: https://www.npmjs.com/package/chai?activeTab=versions 
[unexpected age]: https://www.npmjs.com/package/unexpected?activeTab=versions 
[sinon age]: https://www.npmjs.com/package/sinon?activeTab=versions 
[proxyquire age]: https://www.npmjs.com/package/proxyquire?activeTab=versions 
[UI interfaces]: https://mochajs.org/#interfaces
[nix-practice]: https://blog.enuma.io/update/2018/07/12/nix-in-practice.html
[nix-pdf]: https://blog.enuma.io/update/2018/07/31/nix-markdown-pdf-svg.html
[`entr`]: http://eradman.com/entrproject/
[`nyc`]: https://github.com/istanbuljs/nyc 
[`minimatch`]: https://www.npmjs.com/package/minimatch
[`fzf`]: https://github.com/junegunn/fzf
