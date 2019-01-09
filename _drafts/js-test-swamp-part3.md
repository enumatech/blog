---
layout: post
title:  "JS test swamp Part 3: Enhance assertion ergonomics"
date:   2019-01-09 14:20:00 +0800
categories: update
author: Tamas Herman
---

Our other pain-point with `mocha` was the boilerplate needed to pull in `chai`
assertions and some plugins for it, even for typical stuff, like expecting a
`Promise` to be resolved of rejected (provided by [`chai-as-promised`]).
For Ethereum projects we would need a custom matcher for `BigNumber` or `BN`
too, but the `chai` extension mechanism is not exactly the most obvious
interface to grasp and the existing [`chai-bignumber`] plugin is the not the
most concise to use...

<!--more-->

# Unexpected: elaborate, colored, assertion failures

Amongst the many assertion libraries, [`unexpected`] hits a sweet spot.
It's size is a bit over the top (12MB), it requires 0 configuration. It's
extensibility is astonishingly powerful, yet simple. Assertions are nestable
out of the box. The bundled functionality is extensive and very logical.
It has opted to use strings instead of method chains to denote assertions.
That gives it a serious performance boost at runtime, but at the cost of
auto completion.

Where it really shines is the expectation failures. It's explanations of what
went wrong, is spectacularly clear and helps a lot to fix the cause.

As we saw earlier, we can't really combine it with the lean `tape` framework,
because it's not aware of `AssertionErrors`.a


# Author
[Tamas Herman](http://github.com/onetom/)

[`chai-as-promised`]: https://www.chaijs.com/plugins/chai-as-promised/
[`chai-bignumber`]: https://www.chaijs.com/plugins/chai-bignumber/
[`unexpected`]: https://unexpected.js.org
