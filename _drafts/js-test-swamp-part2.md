---
layout: post
title:  "JS test swamp Part 2: Decoupling runner and reporter with TAP"
date:   2019-01-09 14:20:00 +0800
categories: update
author: Tamas Herman
---

It turns out there is a long history of this decoupled architecture and there
is already a convention for conveying test execution results *produced* by a
test runner to be *consumed* by a test reporter program.

This convention is called TAP, abbreviated from
[<u>T</u>est <u>A</u>nything <u>P</u>rotocol][testanything], and it's designed
to be both reasonably human-readable and easily parsable by computers.

<!--more-->

In TAP literature, reporters are called [test harness] or [TAP consumers],
and programs emitting TAP formatted output are [TAP producers].

The TAP format is quite simple, hence both producer and consumer
implementations can be very simple.

Producers and consumers can even be programmed in different programming
languages and connected in arbitrary combinations by a simple unix pipe.

You can learn more about the history and details of TAP at
http://testanything.org.

This article is part of a series. For the motivation and fundamentals, see
[Part 1].

# JavaScript TAP implementations

JavaScript land has 2 major contenders in the TAP producer space:

1. [tape]
1. [node-tap] (confusingly published as just [tap] on npmjs.com) 

and there are tons of consumers.

## Run tests with `node` directly

While TAP promotes producer/consumer decoupling, both of these implementations
couple test plan declaration, assertions and test execution.

You might think such coupling has a negative consequence, but surprisingly
it yields a more concise, unobtrusive test DSL, despite being more explicit
than alternatives.

In fact they pushed being explicit, while maintaining unobtrusiveness so far
that you can run your test files directly with `node`, since they are just
self-contained, vanilla Node.js programs.

Most JS-aware IDEs provide conveniences around running individual JS files,
even in **debug mode**, so you don't necessarily need special IDE plugins.

Nothing is without a trade-off though... If we run our tap test with `node`,
the test output will be the semi-human-readable, raw, TAP format. :(

It's an empowering feature to run tests simply with `node`, the reality is
that tests are organized into multiple files, so to run a test suite, a
developer might desire some way to run all or some of them. That is still
very much possible using `node` and leaning on your shell and filesystem utils,
however running them in parallel is more complicated as we lamented earlier.

For that reason, both implementations ship with simple test runner scripts.

## Assertions doesn't throw

The other peculiarity is that the bundled assertion system, does NOT conform
to the above described, exception-throwing convention.

While it is generally considered a testing anti-pattern, this non-throwing
behaviour allows placing multiple assertions into a test.

In TAP world though, a `test` is more like a `describe`/`context` block in
other frameworks, so we are not necessarily violating the 1 assertion per
test recommendation.

Worth mentioning that assertions also accept a parameter which can explain the
meaning of the expectation. This parameter alleviates the need for a
BDD style `it`, further toning down the noise in the code, by omitting an
extra level of code nesting.

## Structured data in test output
 
The TAP format is rather minimal and liberal to stay forward compatible.
Originally it was up to the TAP consumers how they handle lines with
unrecognized syntax. [TAP v13][v13] codified how arbitrary
[YAML] data should be embedded into a TAP output, but

> Currently (2007/03/17) the format of the data structure represented 
> by a YAML block has not been standardized.

In practice, it means TAP consumers are not necessarily compatible with
both `tape` and `tap` or the assertion library being used. :(

Now let's zoom in on the individual implementations and collect their pros and
cons.


# Tape

This option is very attractive because of its moderate size, but if we look
deeper, the situation is not so rosy.

I've noticed, that many packages (including `tape` itself) includes test code
into their published packages, which is rather unnecessary, since it's just
dead code and wasted download time and bandwidth in dependent projects.

My biggest sorrow is the lack of loose object comparison, which would permit
the presence of extra attributes in the actual value, like
`expect().toMatchObject()` in jest or `t.match()` in node-tap. 

But as we mentioned, the built-in assertion system doesn't throw, so all
exceptions simply abort the test process and it's not being counted as a test
failure. This closes the door towards plugging in a more sophisticated
assertion libraries and providing more comforting matchers. :( 

While async tests can be expressed using a traditional, call-back style
via `t.plan()` and `t.end()`, a more convenient `Promise` support is absent
from `tape`.

This shortcoming is remedied by [blue-tape], which is a very thin, very
straightforward wrapper around `tape`, thanks to tape's simple internals. 

## Pros

* Relatively small
* Browser support
* Async support

## Cons

* No `Promise` support
* Coverage support is not documented
* Built-in assertions are extremely simple
* Most TAP consumers have various issues with its output 
* Poor IDE support
* Unhelpful, [verbose stack traces][ugly-traces]


# Node-tap (`npm i -D tap`)

While the built-in assertions in `node-tap` doesn't throw, it still recognizes
`AssertionError`s and counts them as test failures, opening up the avenue
for fusing it with our favorite assertion tool.

Initially I was happy to find the `t.match()` assertion, which doesn't stumble
over extra attributes in the actual value. Disappointingly when it fails as
expected, the YAML structure it emits, is not recognized by the built-in
TAP consumers and simply swallowed, leaving the programmer clueless about
how to fix it. I wish it would have such a nice colored output as a
`t.equals()` failure.

Fixing this issue is not obvious though, because the YAML data doesn't
contain a diff, so it's the TAP consumer's responsibility to calculate it.

Furthermore, the stack traces are formatted differently in the `classic`
and the `spec` reporters, so the file references are not turned into links
in IntelliJ, when the `at` prefix is omitted...

Investigating the cause of its unreasonable footprint revealed that it
directly depends on the *28MB* `nyc` coverage CLI package. It would have
been nice if that would be just an optional [peerDependency].

Similarly, [tap-mocha-reporter] is a good candidate for being a
[peerDependency] too and the README could just contain:

> For getting started simply:
> 
> ```
> npm install --save-dev   tap   tap-mocha-reporter
> ```
> and later you can replace `tap-mocha-reporter` with a TAP consumer
> which suites you or your IDE better.

It would only make 404KB optional, but still...

## Pros

* Native `Promise` support
* Decent bundled assertions
* Exception-throwing assertion library compatibility
* Optional `mocha`/BDD style test structure
* Parallel test execution
* Snapshots
* Stack traces without framework internals
* Coverage support
* Browser support

## Cons

* Unreasonable size
* Issues with the default [tap-mocha-reporter]s
* Poor IDE support


# TAP verdict

The concept of TAP is quit attractive, but unfortunately the actual 
implementations are bleeding from multiple (small) wounds.

For most programmers, the convenience provided by `mocha` or `jest`
IDE plugins, outweigh the upside of the simpler, leaner, tap-style execution
model.

Both `tape` and `node-tap` are taking backwards compatibility with both node
and browser runtimes seriously, which contributes to their bloated dependency
tree.

In summary, in their current form, they don't have a clear-cut advantage over
`mocha`.


# Author
[Tamas Herman](http://github.com/onetom/)

[testanything]: http://testanything.org
[Part 1]: js-test-swamp-part1.html
[test harness]: http://testanything.org/tap-specification.html#harness-behavior
[TAP consumers]: http://testanything.org/consumers.html
[TAP producers]: http://testanything.org/producers.html

[tape]: https://github.com/substack/tape
[node-tap]: https://www.node-tap.org
[tap]: https://www.npmjs.com/package/tap
[glob]: https://www.npmjs.com/package/glob
[v13]: http://testanything.org/tap-version-13-specification.html
[YAML]: http://testanything.org/tap-version-13-specification.html#yaml-blocks

[ugly-traces]: https://remysharp.com/2016/02/08/testing-tape-vs-tap 
[blue-tape]: https://github.com/spion/blue-tape
[peerDependency]: https://docs.npmjs.com/files/package.json#peerdependencies

[tap-mocha-reporter]: https://github.com/tapjs/tap-mocha-reporter
