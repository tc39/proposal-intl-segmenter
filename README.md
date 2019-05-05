# `Intl.Segmenter`: Unicode segmentation in JavaScript

Stage 2 proposal, champion Daniel Ehrenberg (Igalia)

## Motivation

A code point is not a "letter" or a displayed unit on the screen. That designation goes to the grapheme, which can consist of multiple code points (e.g., including accent marks, conjoining Korean characters). Unicode defines a grapheme segmentation algorithm to find the boundaries between graphemes. This may be useful in implementing advanced editors/input methods, or other forms of text processing.

Unicode also defines an algorithm for finding breaks between words and sentences, which CLDR tailors per locale. These boundaries may be useful, for example, in implementing a text editor which has commands for jumping or highlighting words and sentences.

Grapheme, word and sentence segmentation is defined in [UAX 29](http://unicode.org/reports/tr29/). Web browsers need an implementation of this kind of segmentation to function, and shipping it to JavaScript saves memory and network bandwidth as compared to expecting developers to implement it themselves in JavaScript.

Chrome has been shipping its own nonstandard segmentation API called `Intl.v8BreakIterator` for a few years. However, [for a few reasons](https://github.com/tc39/ecma402/issues/60#issuecomment-194041835), this API does not seem suitable for standardization. This explainer outlines a new API which attempts to be more in accordance with modern, post-ES2015 JavaScript API design.

## Examples

### Boundary iteration

Objects returned by the `segment` method of an Intl.Segmenter instance expose segmentation boundaries via the <i>Iterable</i> interface.

```js
// Create a locale-specific word segmenter
let segmenter = new Intl.Segmenter("fr", {granularity: "word"});

// Use it to get a boundary iterator for a string
let input = "Moi?  N'est-ce pas.";
let boundaries = segmenter.segment(input);

// Use that for segmentation!
let lastIndex = 0;
for (let {index, precedingSegmentType} of boundaries) {
  let segment = input.slice(lastIndex, index);
  console.log("segment at [%d, %d) of type %o: «%s»",
    lastIndex, index,
    precedingSegmentType,
    input.slice(lastIndex, index)
  );
  lastIndex = index;
}
// console.log output:
// segment at [0, 3) of type "word": «Moi»
// segment at [3, 4) of type "none": «?»
// segment at [4, 6) of type "none": «  »
// segment at [6, 11) of type "word": «N'est»
// segment at [11, 12) of type "none": «-»
// segment at [12, 14) of type "word": «ce»
// segment at [14, 15) of type "none": « »
// segment at [15, 18) of type "word": «pas»
// segment at [18, 19) of type "none": «.»
```

For performance and flexibility, they also support direct random access.

```js
// ┃0 1 2 3 4 5┃6┃7┃8┃9
// ┃A l l o n s┃-┃y┃!┃
let input = "Allons-y!";

let segmenter = new Intl.Segmenter("fr", {granularity: "word"});
let boundaries = segmenter.segment(input);
let done = false;

boundaries.index                // → 0
boundaries.precedingSegmentType // → undefined

done = boundaries.following()   // → false
boundaries.index                // → 6
boundaries.precedingSegmentType // → "word" (describing "Allons")

done = boundaries.following(5)  // → false
boundaries.index                // → 6
boundaries.precedingSegmentType // → "word" (describing "Allons")

done = boundaries.following()   // → false
boundaries.index                // → 7
boundaries.precedingSegmentType // → "none" (describing "-")

done = boundaries.following(8)  // → true
boundaries.index                // → 9
boundaries.precedingSegmentType // → "none" (describing "!")

done = boundaries.following()   // → RangeError
boundaries.index                // → 9

done = boundaries.preceding()   // → false
boundaries.index                // → 8
boundaries.precedingSegmentType // → "word" (describing "y")

done = boundaries.preceding(3)  // → false
boundaries.index                // → 0
boundaries.precedingSegmentType // → undefined

done = boundaries.preceding()   // → RangeError
boundaries.index                // → 0
```

## API

[polyfill](https://gist.github.com/inexorabletash/8c4d869a584bcaa18514729332300356) for a historical snapshot of this proposal

### `new Intl.Segmenter(locale, options)`

Interpretation of options:

- `granularity`, which may be `grapheme`, `word`, or `sentence`.

### `Intl.Segmenter.prototype.segment(string)`

This method creates a new `%SegmentIterator%` over the input string, which will lazily find breaks, starting at index 0.

### `%SegmentIterator%`

This class iterates over segment boundaries of a particular string.

#### Iteration result data

* `index` is the code point index immediately following the last found boundary.
* `precedingSegmentType` is a value characterizing the segment preceding last found boundary. Some particularly important values are:
  * For `word` granularity, "word" for letter/number/ideograph segments vs. "none" for spaces/punctuation/etc.
  * For `sentence` granularity, "term" for segments with terminating punctuation vs. "sep" for those without it.

### Methods on %SegmentIterator%:

#### `%SegmentIterator%.prototype.next()`

The `next` method implements the <i>Iterator</i> interface, finding the next boundary and returning an `IteratorResult` object relating to it. The object includes `index` and `precedingSegmentType` fields corresponding to iteration result data.

#### `%SegmentIterator%.prototype.following(index)`

Move the iterator to the next break position after the given code unit index _index_, or if no index is provided, after its current index. Returns *true* if the end of the string was reached.

#### `%SegmentIterator%.prototype.preceding(index)`

Move the iterator to the previous break position before the given code unit index _index_, or if no index is provided, before its current index. Returns *true* if the beginning of the string was reached.

#### `get %SegmentIterator%.prototype.index`

Return the code unit index of the most recently discovered break position, as an offset from the beginning of the string. Initially the `index` is 0.

#### `get %SegmentIterator%.prototype.precedingSegmentType`

The type of the segment which precedes the current iterator location in logical order. If there is no preceding segment (e.g., a just-instantiated SegmentIterator), or if the granularity is "grapheme", then this will be `undefined`.

## FAQ

Q: Why should we pass a locale and options bag for grapheme breaks? Isn't there just one way to do it?

A: The situation is a little more complicated, e.g., for Indic scripts. Work is ongoing to support grapheme break options for these scripts better; see [this bug](http://unicode.org/cldr/trac/ticket/2142), and in particular [this CLDR wiki page](http://cldr.unicode.org/development/development-process/design-proposals/grapheme-usage). Seems like CLDR/ICU don't support this yet, but it's planned.

Q: Shouldn't we be putting new APIs in built-in modules?

A: If built-in modules had come out before this gets to Stage 3, that sounds like a good option. However, so far the idea in TC39 has been not to block either thing on the other. Built-in modules still have some big questions to resolve, e.g., how/whether polyfills should interact with them.

Q: Why is line breaking not included?

Line breaking was provided in an earlier version of this API, but it is excluded because simply a line breaking API would be incomplete: Line breaking is typically used when laying out text, and text layout requires a larger set of APIs, e.g., determining the width of a rendered string of text. For this reason, we suggest continued development of a line breaking API as part of the [CSS Houdini](https://github.com/w3c/css-houdini-drafts/wiki) effort.

Q: Why is hyphenation not included?

A: Hyphenation is expected to have a different sort of API shape for various reasons:
- Adding a hyphenation break may change the spelling of the affected text
- There may be hyphenation breaks of different priorities
- Hyphenation plays into line layout and font rendering in a more complex way, and we might want to expose it at that level (e.g., in the Web Platform rather than ECMAScript)
- Hyphenation is just a less well-developed thing in the internationalization world. CLDR and ICU don't support it yet; certain web browsers are only getting support for it now in CSS. It's often not done perfectly. It could use some more time to bake. By contrast, word, grapheme, sentence and line breaks have been in the Unicode specification for a long time; this is a shovel-ready project.

Q: Why is this API stateful?

It would be possible to make a stateless API without a SegmentIterator, where instead, a Segmenter has two methods, with two arguments: a string and an offset, for finding the next break before or after. This method would return an object `{precedingSegmentType, index}` similar to what `next()` returns in this API. However, there are a few downsides to this approach:
- Performance:
  - Often, JavaScript implementations need to take an extra step to convert an input string into a form that's usable for the external internationalization library. When querying several break positions on a single string, it is nice to reuse the new form of the string; it would be difficult to cache this and invalidate the cache when appropriate.
  - The `{precedingSegmentType, index}` object may be a difficult allocation to optimize away. Some usages of this library are performance-sensitive and may benefit from a lighter-weight API which avoids the allocation.
- Convenience: Many (most?) usages of this API want to iterate through a string, either forwards or backwards, and get all of the appropriate breaks, possibly interspersed with doing related work. A stateful API may be more terse for this sort of use case--no need to keep track of the previous break position and feed it back in.

It is easy to create a stateless API based on this stateful one, or vice versa, in user JavaScript code.

Q: Why is this an Intl API instead of String methods?

A: All of these break types are actually locale-dependent, and some allow complex options. The result of the `segment` method is a SegmentIterator. For many non-trivial cases like this, analogous APIs are put in ECMA-402's Intl object. This allows for the work that happens on each instantiation to be shared, improving performance. We could make a convenience method on String as a follow-on proposal.

Q: What exactly does the index refer to?

An index *n* refers to the boundary preceding the code unit *n*. For example, when iterating over the string "Hello, world💙" by words in English, there will be boundaries at 0, 5, 6, 7, 12, and 14. The definition of these boundary indices does not depend on whether forwards or backwards iteration is used.
