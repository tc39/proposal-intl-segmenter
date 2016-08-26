# `Intl.Segmenter`: Unicode segmentation in JavaScript

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
for (let {index, breakType} of iterator) {
  console.log(`index: ${index} breakType: ${breakType}`);
  break;
}

// logs the following to the console:
// index: 3 breakType: letter
```

## API

### `Intl.Segmenter(locale, options)`

The only option here, for now, is `type`, which may be `grapheme`, `word`, `sentence` or `line`.

### `Intl.Segmenter.prototype.segment(string)`

This method creates a new `%SegmentIterator%` over the input string, which will lazily find breaks.

### `%SegmentIterator%`

This class iterates over segment boundaries of a particular string.

### `%SegmentIterator%.prototype.next`

The `next` method finds the next boundary and returns an `IterationResult`, where the `value` is an object with fields `index` and `breakType`.

### Additional low-level way to read segments

#### `%SegmentIterator%.prototype.advance`

Move to the next segment (same as `next` but returns `undefined`).

#### `%SegmentIterator%.prototype.index`

Return the index of the current segment (which is the last one returned by `next`).

#### `%SegmentIterator%.prototype.breakType`

The `breakType` of the current segment.

## Further work

- Hyphenization API
