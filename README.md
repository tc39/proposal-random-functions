# Simple Random Functions

* Stage 1
* Authors: WhosyVox and Tab Atkins-Bittner
* Champions: Tab Atkins-Bittner
* Spec Text: currently this README

-----

Historically, JS has offered a very simple API for randomness: a `Math.random()` function that returns a random value from a uniform distribution in the range [0,1), with no further details. This is a perfectly adequate primitive, but most actual applications end up having to wrap this into other functions to do something useful, such as obtaining random *integers* in a particular range, or sampling a *normal* distribution. While it's not hard to write many of these utility functions, doing so *correctly* can be non-trivial: off-by-one errors are easy to hit even in simple "simulate a die roll" cases.

This proposal aims to add a handful of new simple uniform-distribution random number methods. (At the request of the committee, further use-cases are being offloaded to separate proposals, notably [Random Collection Functions](https://github.com/tabatkins/proposal-random-collection-functions) and [Random Distributions](https://github.com/tabatkins/proposal-random-distributions).)

This proposal builds on the (now Stage 2) [Seeded Random](https://github.com/tc39/proposal-seeded-random) proposal, which adds a new `Random` namespace object, both to host the `Random.Seeded` class itself and to host the `Random.Seeded` random methods, so you can call them with a randomly-seeded PRNG created by the User Agent rather than always having to create a `Random.Seeded` yourself.


# The New Random Functions

There is a very large family of functions we could *potentially* include.
This proposal is intentionally pared down to the basic set of commonly-written random number functions: random numbers in a range other than 0-1, random integers, random bigints, and random *bytes*, all drawn from a uniform distribution.

## `Random.number(lo: Number, hi: Number, stepOrOptions: (Number or RandomOptions)?): Number` ##

If `step` is omitted,
returns a random `Number` in the range `(lo, hi)`
(that is, not containing either `lo` or `hi`)
with a uniform distribution.

If there are no floats between `lo` and `hi`,
returns `lo` unless the `excludeMin` option is true;
otherwise, returns `hi` unless the `excludeMax` option is true;
otherwise, throws a {{RangeError}}.

> [!NOTE]
> [Issue 19](https://github.com/tc39/proposal-random-functions/issues/19) discusses the exact planned algorithm, using 2 64-bit chunks of randomness.

If `step` is passed (directly, or as the `step` option)
returns a random `Number` of the form `lo + N*step`,
in the range `[lo, hi]`
(that is, potentially containing `lo` or `hi`).
If the `excludeMin` or `excludeMax` options are passed,
it avoids returning `lo` or `hi`, respectively;
if this results in there being no possible values to return,
throws a {{RangeError}}.

Specifically:

1. Let `epsilon` be a small value related to `step`, chosen to match [`Iterator.range` issue #64](https://github.com/tc39/proposal-iterator.range/issues/64#issuecomment-2881243363)).
2. Let `minN` be 0 if the `excludeMin` option is false, or 1 if it's true.
3. Let `maxN` be the largest integer such that `lo + maxN*step` is less than or equal to `hi`, if the `excludeMax` option is false, or less than `hi`, if it's true.
4. If the `excludeMax` option is false,
    and `lo + maxN*step` is not within `epsilon` of `hi`,
    but `lo + (maxN+1)*step` is
    (even if it's greater than `hi`),
    set `maxN` to `maxN+1`. 
5. If `minN` is larger than `maxN`, throw a RangeError.
6. Let `N` be a random integer between `minN` and `maxN`, inclusive.
7. If `N` is `maxN`, the `excludeMax` option is false, and `lo + maxN*step` is within `epsilon` of `hi`,
    return `hi`. Otherwise, return `lo + N*step`.

> [!NOTE]
> This `step`/`epsilon` behavior is taken directly from [CSS's `random()` function](https://drafts.csswg.org/css-values-5/#random).
> It's also [being proposed for `Iterator.range()`](https://github.com/tc39/proposal-iterator.range/issues/64#issuecomment-2881243363).


## `Random.range(...)` ##

This is an accompaniment to [`Iterator.range()`](https://github.com/tc39/proposal-iterator.range/),
and will live either in this proposal or the `Iterator.range` proposal, whichever advances last.

The arguments to `Random.range()` exactly match those of `Iterator.range()`,
and are interpreted identically.
The function returns one of the values in the range, chosen uniformly at random.
If the equivalent range would be infinite,
instead throws a RangeError.

> [!NOTE]
> This is just a convenience function for what I expect will be a commonly-desired need.
> It's sugar for some equivalent `Random.number()`.


## `Random.int(lo: Number, hi: Number, stepOrOptions: (Number or RandomOptions)?): Number` ##

Returns a random integer Number in the range `[lo, hi]`
(that is, containing `lo` and `hi`),
with a uniform distribution.
If `excludeMin` or `excludeMax` options are passed and true,
it excludes `lo` and `hi`.
If the range thus contains no possible values,
throws a RangeError.

If `step` is passed (directly, or as the `step` option),
returns a random integer Number of the form `lo + N*step`
in the range `[lo, hi]`.
(This might not be capable of returning the `hi` value,
depending on the chosen `hi` and `step`.)
If `excludeMin` or `excludeMax` options are passed and true,
it excludes `lo` and `hi`.
If the range thus contains no possible values,
throws a RangeError.


> [!NOTE]
> [Issue 19](https://github.com/tc39/proposal-random-functions/issues/19) discusses what algorithm to use. Current plan uses 2 64-bit chunks if the lo-hi range contains less than 2^63 values. If the range is larger, the algorithm uses approximately N+1 64-bit chunks, where N is the number of 64-bit chunks it takes to represent the range in the first place (with an ignorable chance of rejection, requiring another set of chunks). Either way, *every* int in the range is possible and has a uniform chance, unlike `Random.number(lo, hi, {step:1})` for large ranges.


## `Random.bigint(lo: BigInt, hi: BigInt, stepOrOptions: (BigInt or RandomOptions)?): BigInt` ##

Identical to `Random.int()`, except it returns BigInts instead.


## `Random.bytes(n: Number): Uint8Array` ##

Returns a `Uint8Array` of length `n`,
filled with random bytes with a uniform distribution.

## `Random.fillBytes(buffer: BufferType, start: Number?, end: Number?): BufferType`

Fills the passed `TypedArray` or `ArrayBuffer`
with random bytes with a uniform distribution.
If `start` and/or `end` are passed,
only fills between those positions,
identical to `TypedArray.prototype.fill()`.

> [!NOTE]
> Note that "random bytes" produces a uniform distribution of values for the *integer types*, like `Uint8Array`, `Int32Array`, etc.
> It does *not* do the same for the float types like `Float64Array`.
> Those types do not have any straightforward definition of "uniform" over their range of possible values.
> If you need something smarter, you'll need to write it yourself to meet your exact use-cases.

# Interaction with `Random.Seeded`

All of the above functions will also be defined on the [`Random.Seeded` class](https://github.com/tc39/proposal-seeded-random/) as methods, with identical signatures and behavior. That is, `Random.number(...)` and `new Random.Seeded(...).number(...)` will both work.

*Precise* generation algorithms will be defined for the `Random.Seeded` methods, to ensure reproducibility. It's recommended that the `Random` versions use the same algorithm, but not strictly required; doing so just lets you use an internal `Random.Seeded` object and avoid implementing the same function twice.

# Expected Questions

> Why do some functions use an open range and others use a closed range?

The open/closed-ness of the ranges expressed in the algorithms is somewhat incidental. They should all be *thought of* as closed ranges, that include both endpoints. This allows us to offer an identical set of options across the functions (`excludeMin` and `excludeMax`).

Being open/closed/half-open is *extremely important and visible* for most random *int* use-cases - if simulating a d6, for example, if `Random.int(1,6)` uses an open or half-open range it'll give extremely incorrect statistics.

This is *not true* for *almost all* random *numbers*. Most of the time, a `Random.number()` range will cover *at least* one full exponent regime of floats (the range between two consecutive powers of 2), meaning you'll have *at least* 2^52 possible return values (four *quadrillion*!). (The planned algorithm maxes out at 2^54, ~18 quadrillion, possible return values.) Even if it doesn't cover a full regime, any realistic scenario is almost certain to cover a *large fraction* of one, still guaranteeing an *extremely large* number of possible values, almost certainly in the billions to trillions.

This means it is *extremely unlikely* for any *particular* value to be returned, such as the actual min or max value. Except in weird corner cases (ranges where the two endpoints are very close together), you simply can't ever depend on seeing those values in the first place, so whether they're actually possible to see or not is irrelevant. The fact that `excludeMin` and `excludeMax` don't actually do anything most of the time in `Random.number()` is simply undetectable, most of the time.

On a slightly different topic, sometimes *most* values in a range are perfectly fine, but *some* cause problems. In general we can't automatically handle this, and you'll just have to do rejection sampling on your own. For example, if you're generating `Random.number(1, 10)` but have to avoid the pure integers, that's on you. That sort of thing is fairly rare, anyway.

What's *not* rare is the *endpoints* being special in some way. For example, `1 / Random.number(0, 1)` is fine for *almost every possible value*, except for 0 itself, which'll result in Infinity. You'll only trigger that issue less than 1 quadrillionth of the time - too rare to ever depend on it happening, but not so rare that it's impossible to see at scale. By omitting the endpoints by default in `Random.number()`, we avoid an entire class of extremely-rare-but-possible bugs like this. And if the author *does* specify the `excludeMin`/`excludeMax` options, we make sure the endpoints don't show up even when the range would otherwise be empty. (This argument doesn't apply to random *ints*, because if an endpoint is problematic you can just... use the next int over, instead. "The next float over" is much more difficult to express.)


# Prior Art

* [Python's `random` module](https://docs.python.org/3/library/random.html)
    * random float, with any min/max bounds (and any step)
    * random ints, with any min/max bounds (and any step)
    * random bytes
    * random selection (or N selections with replacement) from a list, with optional weights
    * random sampling (or N samples, *without* replacement) from a list, with optional counts
    * randomly shuffle an array
    * sample various random distributions: binomal, triangular, beta, exponential, gamma, normal, log-normal, von Mises, Pareto, Weibull
* [.Net's `Random` class](https://learn.microsoft.com/en-us/dotnet/api/system.random?view=net-8.0)
    * random ints, with any min/max bounds
    * random floats, with any min/max bounds
    * random bytes
    * randomly shuffle an array
* [Haskell's `RandomGen` interface](https://hackage.haskell.org/package/random-1.2.1.2/docs/System-Random.html)
    * random u8/u16/u32/u64, either across the full range or between 0 and max
    * <s>generate two RNGs from the starting RNG</s> (seeded-random use-case)
    * random from any type with a range (like all the numeric types, enums, etc), or tuples of such
    * random bytes
    * random floats between 0 and 1
* [Ruby's `Random` class](https://ruby-doc.org/core-2.4.0/Random.html)
    * random floats between 0 and max
    * random ints between 0 and max
    * random bytes
* [Common Lisp's `(random n)` function]([https://www.cs.cmu.edu/Groups/AI/html/cltl/clm/node133.html](http://clhs.lisp.se/Body/f_random.htm))
    * random ints between 0 and max
    * random floats between 0 and max
* [Java's `Random` class](https://docs.oracle.com/javase/8/docs/api/java/util/Random.html)
    * random doubles, defaulting to [0,1) but can be given any min/max bounds
    * random ints (or longs), with any min/max bounds
    * random bools
    * random bytes
    * sample Gaussian distribution
* [JS `genTest` library](https://www.npmjs.com/package/gentest)
    * random ints (within certain classes)
    * random characters
    * random "strings" (relatively short but random lengths, random characters)
    * random bools
    * random selection (N, with replacement) from a list
    * custom random generators

# History

* 2024-04: Initial Stage 0 proposal authored.
* 2025-05: Proposal was accepted for Stage 1, somewhat stripped down (TODO: link to notes)
