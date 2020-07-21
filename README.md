# `Intl.Segmenter`: Unicode segmentation in JavaScript

Stage 3 proposal, champion Richard Gibson

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

For flexibility and advanced use cases, they also support direct random access.

```js
// ┃0 1 2 3 4 5┃6┃7┃8┃9
// ┃A l l o n s┃-┃y┃!┃
let input = "Allons-y!";

let segmenter = new Intl.Segmenter("fr", {granularity: "word"});
let segments = segmenter.segment(input);
let current = undefined;

current = segments.containing(0)
// → { index: 0, segment: "Allons", isWordLike: true }

current = segments.containing(5)
// → { index: 0, segment: "Allons", isWordLike: true }

current = segments.containing(6)
// → { index: 6, segment: "-", isWordLike: false }

current = segments.containing(current.index + current.segment.length)
// → { index: 7, segment: "y", isWordLike: true }

current = segments.containing(current.index + current.segment.length)
// → { index: 8, segment: "!", isWordLike: false }

current = segments.containing(current.index + current.segment.length)
// → undefined
```

## API

[polyfill](https://gist.github.com/inexorabletash/8c4d869a584bcaa18514729332300356) for a historical snapshot of this proposal

### `new Intl.Segmenter(locale, options)`

Creates a new locale-dependent Segmenter.
If `options` is provided, it is treated as an object and its `granularity` property specifies the segmenter granularity ("grapheme", "word", or "sentence", defaulting to "grapheme").

### `Intl.Segmenter.prototype.segment(string)`

Creates a new [<i>Iterable</i>](https://tc39.es/ecma262/#sec-iterable-interface) `%Segments%` instance for the input string using the Segmenter's locale and granularity.

### Segment data

Segments are described by plain objects with the following data properties:
* `segment` is the string segment.
* `index` is the code unit index in the string at which the segment begins.
* `input` is the string being segmented.
* `isWordLike` is `true` when granularity is "word" and the segment is _word-like_ (consisting of letters/numbers/ideographs/etc.), `false` when granularity is "word" and the segment is not _word-like_ (consisting of spaces/punctuation/etc.), and `undefined` when granularity is not "word".

### Methods of %Segments%.prototype:

#### `%Segments%.prototype.containing(index)`

Returns a segment data object describing the segment in the string including the code unit at the specified index, or `undefined` if the index is out of bounds.

#### `%Segments%.prototype[Symbol.iterator]`

Creates a new `%SegmentIterator%` instance which will lazily find segments in the input string using the Segmenter's locale and granularity, keeping track of its current position within the string.

### Methods of %SegmentIterator%.prototype:

#### `%SegmentIterator%.prototype.next()`

The `next` method implements the <i>Iterator</i> interface, finding the next segment and returning a corresponding <i>IteratorResult</i> object whose `value` property is a segment data object as described above.

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

### Why is random-access stateless?

It would be possible to expose methods on %SegmentIterator%.prototype that mutate internal state (e.g., `seek([inclusiveStartIndex = thisIterator.index + 1])` and `seekBefore([exclusiveLastIndex = thisIterator.index])`, and in fact these were part of earlier designs.
They were dropped for consistency with other ECMA-262 iterators (whose movement is always forward and without gaps).
If real-world use reveals that their absence is an ergonomic and/or performance flaw, they can be added in a followup proposal.

### Why is this an Intl API instead of String methods?

All of these boundary types are actually locale-dependent, and some allow complex options. The result of the `segment` method is a SegmentIterator. For many non-trivial cases like this, analogous APIs are put in ECMA-402's Intl object. This allows for the work that happens on each instantiation to be shared, improving performance. We could make a convenience method on String as a follow-on proposal.

### What exactly does the index refer to?

An index *n* refers to the code unit index within a string that is potentially the start of a segment.
For example, when iterating over the string "Hello, world💙" by words in English,segments will start at indexes 0, 5, 6, 7, and 12 (i.e., the string gets segmented like `┃Hello┃,┃ ┃world┃💙┃`, with the final segment consisting of a surrogate pair of two code units encoding a single code point).
The definition of these boundary indexes does not depend on whether forwards or backwards iteration is used.

### What happens when segmenting an empty string?

No segments will be found, and iterators will complete immediately upon first `next()` access.

### What happens when I try to use random access with non-Number values?

_Someone's_ in QA. 😉
Arguments are processed into integer Numbers—`null` becomes 0, Booleans become 0 or 1, Strings are parsed as string numeric literals, Objects are cast to primitives, and Symbols, BigInts, `undefined`, and `NaN` fail. Fractional components are truncated, but infinite Numbers are accepted as-is (although they are always out of bounds and will therefore never find a segment).
