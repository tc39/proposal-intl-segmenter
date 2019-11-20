# `Intl.Segmenter`: Unicode segmentation in JavaScript

Stage 2 proposal, champion Richard Gibson

## Motivation

A code point is not a "letter" or a displayed unit on the screen. That designation goes to the grapheme, which can consist of multiple code points (e.g., including accent marks, conjoining Korean characters). Unicode defines a grapheme segmentation algorithm to find the boundaries between graphemes. This may be useful in implementing advanced editors/input methods, or other forms of text processing.

Unicode also defines an algorithm for finding boundaries between words and sentences, which CLDR tailors per locale. These boundaries may be useful, for example, in implementing a text editor which has commands for jumping or highlighting words and sentences.

Grapheme, word and sentence segmentation is defined in [UAX 29](http://unicode.org/reports/tr29/). Web browsers need an implementation of this kind of segmentation to function, and shipping it to JavaScript saves memory and network bandwidth as compared to expecting developers to implement it themselves in JavaScript.

Chrome has been shipping its own nonstandard segmentation API called `Intl.v8BreakIterator` for a few years. However, [for a few reasons](https://github.com/tc39/ecma402/issues/60#issuecomment-194041835), this API does not seem suitable for standardization. This explainer outlines a new API which attempts to be more in accordance with modern, post-ES2015 JavaScript API design.

## Examples

### Segment iteration

Objects returned by the `segment` method of an Intl.Segmenter instance find boundaries and expose segments between them via the [<i>Iterable</i> interface](https://tc39.es/ecma262/#sec-iterable-interface).

```js
// Create a locale-specific word segmenter
let segmenter = new Intl.Segmenter("fr", {granularity: "word"});

// Use it to get an iterator for a string
let input = "Moi?  N'est-ce pas.";
let segments = segmenter.segment(input);

// Use that for segmentation!
for (let {segment, index, isWordLike} of segments) {
  console.log("segment at code units [%d, %d): «%s»%s",
    index, index + segment.length,
    segment,
    isWordLike ? " (word-like)" : ""
  );
}
// console.log output:
// segment at code units [0, 3): «Moi» (word-like)
// segment at code units [3, 4): «?»
// segment at code units [4, 6): «  »
// segment at code units [6, 11): «N'est» (word-like)
// segment at code units [11, 12): «-»
// segment at code units [12, 14): «ce» (word-like)
// segment at code units [14, 15): « »
// segment at code units [15, 18): «pas» (word-like)
// segment at code units [18, 19): «.»
```

For performance and flexibility, they also support direct random access.

```js
// ┃0 1 2 3 4 5┃6┃7┃8┃9
// ┃A l l o n s┃-┃y┃!┃
let input = "Allons-y!";

let segmenter = new Intl.Segmenter("fr", {granularity: "word"});
let segments = segmenter.segment(input);
let done = false;

segments.index                   // → -1
segments.segment                 // → null
segments.isWordLike              // → null

segments.following()             // → [object Segmenter String Iterator]
segments.index                   // → 0
segments.segment                 // → "Allons"
segments.isWordLike              // → true

segments.following(4)            // → [object Segmenter String Iterator]
segments.index                   // → 6
segments.segment                 // → "-"
segments.isWordLike              // → false

segments.following()             // → [object Segmenter String Iterator]
segments.index                   // → 7
segments.segment                 // → "y"
segments.isWordLike              // → true

segments.following().following() // → [object Segmenter String Iterator]
segments.index                   // → 9
segments.segment                 // → null
segments.isWordLike              // → null

segments.following()             // → RangeError
segments.index                   // → 9

segments.preceding()             // → [object Segmenter String Iterator]
segments.index                   // → 8
segments.segment                 // → "!"
segments.isWordLike              // → false

segments.preceding(3)            // → [object Segmenter String Iterator]
segments.index                   // → 0
segments.segment                 // → "Allons"
segments.isWordLike              // → true

segments.preceding()             // → [object Segmenter String Iterator]
segments.index                   // → -1
segments.segment                 // → null
segments.isWordLike              // → null
```

## API

[polyfill](https://gist.github.com/inexorabletash/8c4d869a584bcaa18514729332300356) for a historical snapshot of this proposal

### `new Intl.Segmenter(locale, options)`

Creates a new locale-dependent Segmenter.
If `options` is provided, it is treated as an object and its `granularity` property specifies the segmenter granularity ("grapheme", "word", or "sentence", defaulting to "grapheme").

### `Intl.Segmenter.prototype.segment(string)`

Creates a new `%SegmentIterator%` instance which will lazily find segments in the input string using the Segmenter's locale and granularity, keeping track of its current position within the string.

### `%SegmentIterator%.prototype`

#### Iteration result data

The `value` property of an <i>IteratorResult</i> object produced by a `%SegmentIterator%` instance is an object with the following data properties:
* `segment` is the string segment.
* `index` is the code unit index in the string at which the segment begins.
* `isWordLike` is `true` when granularity is "word" and the segment is _word-like_ (consisting of letters/numbers/ideographs/etc.), `false` when granularity is "word" and the segment is not _word-like_ (consisting of spaces/punctuation/etc.), and `undefined` when granularity is not "word".

### Methods of %SegmentIterator%.prototype:

#### `%SegmentIterator%.prototype.next()`

The `next` method implements the <i>Iterator</i> interface, finding the next segment and returning a corresponding <i>IteratorResult</i> object as described above.

#### `%SegmentIterator%.prototype.following(startAfter = this.index)`

Advances the iterator to the first segment in the string starting after the specified code unit index (defaulting to the current index when not provided).
Returns the iterator, or throws if the starting index was out of bounds (e.g., was already past the end of the string).

#### `%SegmentIterator%.prototype.preceding(startBefore = this.index)`

Moves the iterator backward to the first segment in the string starting before the specified code unit index (defaulting to the current index when not provided).
Returns the iterator, or throws if the starting index was out of bounds (e.g., was already negative).

### Properties of %SegmentIterator%.prototype:

#### `get %SegmentIterator%.prototype.segment`

A read-only accessor property that returns the current string segment, as if from an [iteration result](#iteration-result-data).
The initial value is `null`.

#### `get %SegmentIterator%.prototype.index`

A read-only accessor property that returns the code unit index within the iterated string at which the current segment starts, as if from an [iteration result](#iteration-result-data).
The initial value is `-1` (i.e., a position immediately preceding the first segment), and the highest possible value is the length of the iterated string (i.e., a position immediately following the last code unit).

#### `get %SegmentIterator%.prototype.isWordLike`

A read-only accessor property that returns the "word-like" classification of the current string segment, as if from an [iteration result](#iteration-result-data).
The initial value for granularity "word" is `null`, and for all other granularities is `undefined`.

## FAQ

### Why should we pass a locale and options bag for grapheme boundaries? Isn't there just one way to do it?

The situation is a little more complicated, e.g., for Indic scripts. Work is ongoing to support grapheme boundary options for these scripts better; see [this bug](http://unicode.org/cldr/trac/ticket/2142), and in particular [this CLDR wiki page](http://cldr.unicode.org/development/development-process/design-proposals/grapheme-usage). Seems like CLDR/ICU don't support this yet, but it's planned.

### Shouldn't we be putting new APIs in built-in modules?

If built-in modules had come out before this gets to Stage 3, that sounds like a good option. However, so far the idea in TC39 has been not to block either thing on the other. Built-in modules still have some big questions to resolve, e.g., how/whether polyfills should interact with them.

### Why is line breaking not included?

Line breaking was provided in an earlier version of this API, but it is excluded because simply a line breaking API would be incomplete: Line breaking is typically used when laying out text, and text layout requires a larger set of APIs, e.g., determining the width of a rendered string of text. For this reason, we suggest continued development of a line breaking API as part of the [CSS Houdini](https://github.com/w3c/css-houdini-drafts/wiki) effort.

### Why is hyphenation not included?

Hyphenation is expected to have a different sort of API shape for various reasons:
- Adding a hyphenation break may change the spelling of the affected text
- There may be hyphenation breaks of different priorities
- Hyphenation plays into line layout and font rendering in a more complex way, and we might want to expose it at that level (e.g., in the Web Platform rather than ECMAScript)
- Hyphenation is just a less well-developed thing in the internationalization world. CLDR and ICU don't support it yet; certain web browsers are only getting support for it now in CSS. It's often not done perfectly. It could use some more time to bake. By contrast, word, grapheme, sentence and line breaks have been in the Unicode specification for a long time; this is a shovel-ready project.

### Why is this API stateful?

It would be possible to make a stateless API without a SegmentIterator, where instead, a Segmenter has two methods, with two arguments: a string and an offset, for finding the next boundary before or after. This method would return an object similar to what `next()` returns in this API. However, there are a few downsides to this approach:
- Performance:
  - Often, JavaScript implementations need to take an extra step to convert an input string into a form that's usable for the external internationalization library. When querying several positions on a single string, it is nice to reuse the new form of the string; it would be difficult to cache this and invalidate the cache when appropriate.
  - Allocation of the <i>IteratorResult</i> objects may be difficult to optimize away. Some usages of this library are performance-sensitive and may benefit from a lighter-weight API which avoids the allocation.
- Convenience: Many (most?) usages of this API want to iterate through a string, either forwards or backwards, and get all of the appropriate segments, possibly interspersed with doing related work. A stateful API may be more terse for this sort of use case—no need to keep track of the previous position and feed it back in.

It is easy to create a stateless API based on this stateful one, or vice versa, in user JavaScript code.

### Why is this an Intl API instead of String methods?

All of these boundary types are actually locale-dependent, and some allow complex options. The result of the `segment` method is a SegmentIterator. For many non-trivial cases like this, analogous APIs are put in ECMA-402's Intl object. This allows for the work that happens on each instantiation to be shared, improving performance. We could make a convenience method on String as a follow-on proposal.

### What exactly does the index refer to?

An index *n* refers to the code unit index within a string that is potentially the start of a segment.
For example, when iterating over the string "Hello, world💙" by words in English,segments will start at indexes 0, 5, 6, 7, 12, and 14 (i.e., the string gets segmented like `┃Hello┃,┃ ┃world┃💙┃`, with the final segment consisting of a surrogate pair of two code units encoding a single code point).
The definition of these boundary indexes does not depend on whether forwards or backwards iteration is used.

### What happens when segmenting an empty string?

No segments will be found, but random access will succeed once in each direction (e.g., `segmenter.segment("").following()` will return an iterator at index 0 with a null `segment`, and `segmenter.segment("").preceding(Infinity)` will return an iterator at index -1 with a null `segment`).
Attempts to advance forward from 0 or higher, or backward from -1 or lower, will fail with a RangeError.
We essentially synthesize two positions at which there is no segment (not just for empty string but for _all_ input), one before the string (which is also the starting point) and one after it.

### What happens when I try to use random access with non-Number values?

Someone's in QA. 😉
The random access methods treat `undefined` starting index the same as unspecified, and will start from the current index.
All other inputs are processed into integer Numbers—`null` becomes 0, Booleans become 0 or 1, Strings are parsed as string numeric literals, Objects are cast to primitives, and Symbols, BigInts, and `NaN` fail. Fractional components are truncated, but infinite Numbers are accepted as-is.

### Do you feel bad about violating [<i>Iterator</i> interface best practices](https://tc39.es/ecma262/#sec-iterator-interface) by allowing random access to "reset" finished iterators?

No, not really.
