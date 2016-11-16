# `Intl.Segmenter`: Unicode segmentation in JavaScript

Stage 1 proposal, champion Daniel Ehrenberg (Google)

## Motivation

A code point is not a "letter" or a displayed unit on the screen. That designation goes to the grapheme, which can consist of multiple code points (e.g., including accent marks, conjoining Korean characters). Unicode defines a grapheme segmentation algorithm to find the boundaries between graphemes. This may be useful in implementing advanced editors/input methods, or other forms of text processing.

Unicode also defines an algorithm for finding breaks between words and sentences, which CLDR tailors per locale. These boundaries may be useful, for example, in implementing a text editor which has commands for jumping or highlighting words and sentences. There is an analogous algorithm for opportunities for line breaking.

Grapheme, word and sentence segmentation is defined in [UAX 29](http://unicode.org/reports/tr29/). Line breaking is defined in [UAX 14](http://www.unicode.org/reports/tr14/). Web browsers need an implementation of both kinds of segmentation to function, and shipping it to JavaScript saves memory and network bandwidth as compared to expecting developers to implement it themselves in JavaScript.

Chrome has been shipping its own nonstandard segmentation API called `Intl.v8BreakIterator` for a few years. However, [for a few reasons](https://github.com/tc39/ecma402/issues/60#issuecomment-194041835), this API does not seem suitable for standardization. This explainer outlines a new API which attempts to be more in accordance with modern, post-ES2015 JavaScript API design.

## Example

```js
// Create a segmenter in your locale
let segmenter = Intl.Segmenter("fr", {type: "word"});

// Get an iterator over a string
let iterator = segmenter.segment("Ceci n'est pas une pipe");

// Iterate over it!
for (let {segment, breakType} of iterator) {
  console.log(`segment: ${segment} breakType: ${breakType}`);
  break;
}

// logs the following to the console:
// index: Ceci breakType: letter
```

## API

[polyfill](https://gist.github.com/inexorabletash/8c4d869a584bcaa18514729332300356) for a historical snapshot of this proposal

### `Intl.Segmenter(locale, options)`

Interpretation of options:
- `granularity`, which may be `grapheme`, `word`, `sentence` or `line`.
- `strength`, valid only for `line` granularity, which may be `'strict'`, `'normal'`, or `'loose'`, following CSS Text Module Level 3.

### `Intl.Segmenter.prototype.segment(string)`

This method creates a new `%SegmentIterator%` over the input string, which will lazily find breaks.

### `%SegmentIterator%`

This class iterates over segment boundaries of a particular string.

### Methods on %SegmentIterator%:

#### `%SegmentIterator%.prototype.next()`

The `next` method, to use finds the next boundary and returns an `IterationResult`, where the `value` is an object with fields `segment` and `breakType`. The `segment` contains the substring between the previous break location and the newly found break location; the `breakType` describes which sort of segment it is (TODO: define possible values, not part of UTS). This method defines the iteration protocol support for SegmentIterators, and is present for convenience; other methods expose a richer API.

#### `%SegmentIterator%.prototype.advance()`

Move the iterator to the next break position, returning `undefined`.

#### `%SegmentIterator%.prototype.rewind()`

Move the iterator to the previous break position, returning `undefined`.

#### `%SegmentIterator%.prototype.end()`

Move the iterator to the index after the end of the string, returning `undefined`. This is useful for reverse iteration with `rewind()`.

#### `get %SegmentIterator%.prototype.index`

Return the index of the most recently discovered break position, as an offset from the beginning of the string. Initially the `index` is 0, and it is the length of the string after a call to `end()`.

#### `get %SegmentIterator%.prototype.breakType`

The `breakType` of the most recently discovered segment.

## FAQ

Q: Why should we pass a locale and options bag for grapheme breaks? Isn't there just one way to do it?

A: The situation is a little more complicated, e.g., for Indic scripts. Work is ongoing to support grapheme break options for these scripts better; see See [this bug](http://unicode.org/cldr/trac/ticket/2142), and in particular [this CLDR wiki page](http://cldr.unicode.org/development/development-process/design-proposals/grapheme-usage). Seems like CLDR/ICU don't support this yet, but it's planned.

Q: Shouldn't we be putting new APIs in built-in modules?

A: If built-in modules come out before this gets to Stage 3, that sounds like a good option. However, so far the idea in TC39 has been in TC39 not to block either thing on the other. Built-in modules still have some big questions to resolve, e.g., how/whether polyfills should interact with them.

Q: Why is hyphenation not included?

A: Hyphenation is expected to have a different sort of API shape for various reasons:
- Adding a hyphenation break may change the spelling of the affected text
- There may be hyphenation breaks of different priorities
- Hyphenation plays into line layout and font rendering in a more complex way, and we might want to expose it at that level (e.g., in the Web Platform rather than ECMAScript)
- Hyphenation is just a less well-developed thing in the internationalization world. CLDR and ICU don't support it yet; certain web browsers are only getting support for it now in CSS. It's often not done perfectly. It could use some more time to bake. By contrast, word, grapheme, sentence and line breaks have been in the Unicode specification for a long time; this is a shovel-ready project.
