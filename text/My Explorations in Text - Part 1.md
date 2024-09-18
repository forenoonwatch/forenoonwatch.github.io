## Introduction
This article series covers the world of text rendering and text editing, particularly from the perspective of game engine development, as well as the design decisions I chose to make in my own text rendering and editing assistance library, [RichText.](https://github.com/forenoonwatch/rich-text) The articles are not particularly academic and I am by no means an expert, although I will try to leave citations where possible; my research primarily covers an investigation of how well-established platforms such as Windows, Linux, Chromium, or Firefox handle *interactive text rendering in the general case*, as well as how to utilize popular text support libraries such as FreeType, HarfBuzz, and ICU in order to build these systems.
## Motivation
While text rendering is generally well researched, there do not seem to be any comprehensive articles on rendering arbitrary formatted, multi-lingual text, nor are there any libraries available that act as a one-stop solution. This is especially true when dealing with game engine needs, where you do not want the text layout and editing to affect your architecture for UI or glyph rendering.
## Prior Work
Aside from the work of the Unicode organization and the contributors to the many libraries I will reference throughout the article, several important articles stand out:
- [Text Rendering Hates You](https://faultlore.com/blah/text-hates-you/) and [Text Editing Hates You Too](https://lord.io/text-editing-hates-you-too/), both of which are great resources to illustrate the depth and complexity of rendering text, and wonderful for finding edge cases you need to support. These articles serve to ask the "questions" about implementing a text library that my articles hope to answer.
- [UTF-8 Everywhere](https://utf8everywhere.org/), a manifesto about the values of UTF-8 as the superior Unicode encoding scheme, a very informative article and a big motivator to accept the additional challenge of making my text library work in UTF-8.
## Scope Overview
So what exactly does "interactive text rendering in the general case" mean? At its simplest, it means "being able to do whatever the average text box can do", such as being able to display any valid sequence of Unicode characters correctly given that a font is available for them, and being able to select, add, or delete characters with a keyboard and mouse. The particulars of how "the average text box" handles text editing, and what features that entail are discussed in a later article. 

In addition to this, I personally want to support the following feature list, which covers most of the major text formatting features available in rich text editors:
- Character and word aware line breaking to fit in a box of a given size
- Multiple font sizes acting on a single string
- Text coloration
- Italic
- Bold and other font weights
- Underline
- Strikethrough
- Smallcaps
- Subscript and superscript
- Text stroke
Each of these features will be defined and discussed in more detail in their respective sections. The next section will cover everything that is necessary for displaying an unformatted string.
## Definitions
Before going any further, I will list out some definitions for common terminology as I understand them:

| Keyword                      | Definition                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| ---------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Codepoint                    | Any numerical value in the Unicode codespace.                                                                                                                                                                                                                                                                                                                                                                                                      |
| Code unit                    | The individual values comprising a codepoint in a particular Unicode encoding, for example in UTF-8 a code point would be each individual 8-bit value.                                                                                                                                                                                                                                                                                             |
| Character/Abstract Character | A unit of information used for the organization, control, or representation of textual data. (From the Unicode standard definition for "Abstract Character".                                                                                                                                                                                                                                                                                       |
| Glyph                        | "The specific shape, design, or representation of a character." [Wikipedia](https://en.wikipedia.org/wiki/Glyph). In terms of text rendering, it is most helpful to think of a glyph as whatever individual shape you are able to request from a font.                                                                                                                                                                                             |
| Grapheme                     | The smallest functional unit of a writing system. [Wikipedia](https://en.wikipedia.org/wiki/Grapheme). It is important to note that in text rendering, there is not necessarily any direct correlation between glyphs and graphemes. A grapheme may be represented by multiple glyphs or a glyph may represent multiple graphemes.                                                                                                                 |
| Grapheme Cluster             | A sequence of coded characters that "should be kept together". Grapheme clusters approximate the notion of user-perceived characters in a language independent way.                                                                                                                                                                                                                                                                                |
| Glyph Cluster                | A sequence of glyphs meant to represent 1 or more characters.                                                                                                                                                                                                                                                                                                                                                                                      |
| Script                       | A collection of letters and other written signs used to represent textual information in one or more writing systems. For example, Russian is written with a subset of the Cyrillic script; Ukrainian is written with a different subset. The Japanese writing system uses several scripts. [Unicode](https://www.unicode.org/glossary/).                                                                                                          |
| Face/Typeface/Font Face      | A design of letters, numbers and other symbols, to be used in printing or for electronic display. [Wikipedia](https://en.wikipedia.org/wiki/Typeface). In text rendering terms this means the *unscaled* definition of all the symbol shapes for a particular font. In most cases a typeface is designed around supporting some particular script or scripts.                                                                                      |
| Font                         | A term with a slightly overloaded definition. Strictly speaking, a Font is an instance of a typeface with a particular size, weight, and style. However in common parlance it can also refer to a typeface or a set of typefaces sharing a common design.                                                                                                                                                                                          |
| Font Family                  | A named set of typefaces with a similar design, separated by weight and style. These usually all represent characters within the same set of scripts.                                                                                                                                                                                                                                                                                              |
| Font Collection              | A collection of multiple font families, usually to provide coverage of multiple sets of scripts with typefaces of similar design. An example is [Noto Fonts](https://github.com/notofonts), a collection of individual font families with distinct names based on the scripts they represent, such as "Noto Serif Devanagari" or "Noto Naskh Arabic". These families in turn have individual files for each weight/style combination they support. |
| Font File                    | A file storing data for the representation of glyphs, as well as character maps that map codepoints to glyphs. Some font file formats such as TrueType (.ttf) or OpenType (.otf) store scalable vector data and thus can represent any font within a typeface for the given weight and style, while other formats may store only rasterized bitmap glyphs and thus represent only a finite set of fonts.                                           |
| Font Style                   | Generally refers to whether a particular font is italic or not italic ("normal").                                                                                                                                                                                                                                                                                                                                                                  |
| Font Weight                  | The relative thickness of the glyphs' lines. These include common named thicknesses such as Light, Regular, Medium, Bold, or Black.                                                                                                                                                                                                                                                                                                                |
# Part 1 - Dealing with Any Text
With those definitions out of the way, let's break down what is involved in displaying any particular string. Let's start by implementing a simple `draw_text` function, for the simplest case you need a total of 2 parameters, a Font and a String:
```cpp
void draw_text(Font font, String string);
```
In terms of practical implementation, this `Font` might be a wrapper around accessing a single font file. To display characters you would iterate the codepoints of the string, request a glyph from the font for each, and use your favorite rasterization library such as FreeType to render it. [LearnOpenGL](https://learnopengl.com/In-Practice/Text-Rendering) has a very simple implementation of this technique using FreeType and rendering Latin ASCII strings.

Let's look at a basic C++ implementation that is slightly improved upon the LearnOpenGL example:
```cpp
void draw_text(FT_Face face, const std::string& str) {
	int32_t i = 0;
	UChar32 c = 0;
	int cursorX = 0;
	int cursorY = 0;

	while (i < str.size()) {
		U8_NEXT_OR_FFFD(str.data(), i, str.size(), c);
		unsigned glyphIndex = FT_Get_Char_Index(face, c);
		FT_Load_Glyph(face, glyphIndex, FT_LOAD_ADVANCE_ONLY);

		draw_glyph(glyphIndex, cursorX, cursorY);

		cursorX += face->glyph->advance.x;
		cursorY += face->glyph->advance.y;
	}
}
```

This basic implementation is fine for many European languages, but there are immediately a few limitations:
1. A Unicode string might have multiple scripts, some of which might not be represented by the given font file.
2. There may not be a 1:1 mapping between the codepoints in the string and the glyphs in the font file.
## Enter Harfbuzz
Harfbuzz is the gold standard for _text shaping_, the problem of interpreting a font file's metadata combined with the content of a string in order to generate a list of glyphs and their positions. Perfect, right? Indeed, with this dependency problem #2 listed above is completely solved with no excess mention of kerning, combining characters, or substitution tables, but let's look at how harfbuzz changes the code to see what assumptions we're still relying on:
```cpp
void draw_text(hb_font_t* font, const std::string& str) {
	hb_buffer_t* buf = hb_buffer_create();
	hb_buffer_add_utf8(buf, str.data(), str.size(), 0, -1);
	hb_buffer_set_direction(buf, HB_DIRECTION_LTR);
	hb_buffer_set_script(buf, HB_SCRIPT_LATIN);
	hb_buffer_set_language(buf, hb_language_from_string("en", -1));

	int cursorX = 0;
	int cursorY = 0;

	hb_shape(font, buf, nullptr, 0);

	unsigned glyphCount;
	hb_glyph_info_t* glyphInfo = hb_buffer_get_glyph_infos(buf, &glyphCount);
	hb_glyph_position_t* glyphPos = hb_buffer_get_glyph_positions(buf, &glyphCount);

	for (unsigned i = 0; i < glyphCount; ++i) {
		draw_glyph(glyphInfo[i].codepoint, cursorX + glyphPos[i].x_offset, cursorY + glyphPos[i].y_offset);
		cursorX += glyphPos[i].x_advance;
		cursorY += glyphPos[i].y_advance;
	}

	hb_buffer_destroy(buf);
}
```
As you can see, we still assume that the entire string flows in a single direction, and uses a single font, script, and language. In practice this produces a mostly passable result already, since most languages in use today are Left-to-Right and don't suffer from too many substitution or positioning issues due to the wrong script or language, however for glyphs outside of the scope of your font file your text might look a bit ������.
In my experience, this is where many hobbyist game engines stop, however this is still an insufficient result (especially if you speak Arabic or Hebrew), and the inability of this solution to extract Unicode metadata from the string severely limits our options for more complex features, so where do we go from here?
## Enter the ICU
The International Components for Unicode (ICU) is the official Unicode support library, and contains the majority of the remaining tools for implementing fully featured text rendering. For now, the two things we are interested in are the implementation of the [Unicode Bidirectional Algorithm (UBA)](https://www.unicode.org/reports/tr9/), and [UAX #24](https://www.unicode.org/reports/tr24/), the Unicode script run detection algorithm.
### UAX #24 Script Detection
UAX #24 is the simpler of the two algorithms, it iterates the string and finds runs (contiguous subregions) of text that are part of the same script. Common symbols which are not part of any script are added to the previous script.
There is an implementation of UAX #24 in icu4c in `common/usc_impl.h`, however this is technically an internal ICU header, so use it at your own risk. However, it is a great starting reference for your own implementation.
### Unicode Bidirectional Algorithm
The UBA splits a string into runs of single directions, either Left-to-Right (LTR) or Right-to-Left (RTL). It operates in multiple stages, first splitting a string into multiple paragraphs, and then computing the directionality runs. After computing the logical directionality runs, the UBA calculates the visual runs, which are substrings of the logical runs, not necessarily displayed in logical run order. With the combination of multiple scripts and directional control characters, visual runs may end up far removed from their logical position in the source string. Tools to use this algorithm can be found in icu4c's `<common/ubidi.h>`.
## Putting it all together - Logical and Visual Runs
To review, in order to build the final text string we need to send unique combinations of font, direction, and script to the shaper in order to generate the necessary glyph indices and positions. The results of these shaped logical runs must then be displayed in visual order based on the results of the UBA.
```cpp
struct LogicalRun {
	Font font;
	UBiDiLevel level;
	UScriptCode script;
};

// Initialize the ICU bidirectional iterator for the paragraph
UBiDi* pParaBidi = ubidi_open();
ubidi_setPara(pParaBidi, str.data(), str.size(), baseParagraphLevel, nullptr, &err);

ValueRuns<UScriptCode> scriptRuns = compute_script_runs(str); // UAX #24
ValueRuns<Font> fontRuns = compute_font_runs(str); /* covered later */
ValueRuns<UBiDiLevel> levelRuns = compute_logical_bidi_runs(pParaBidi, str); // UBA

ValueRuns<LogicalRun> logicalRuns = /*Intersection of the above runs*/

// Shape based on the logical runs
int cursorX = 0;
int cursorY = 0;
std::vector<unsigned> codepoints;
std::vector<ivec2> positions;

int start = 0;
for (auto [logicalRun, limit] : logicalRuns) {
	// Appends to the codepoint and position arrays, using the same cursor
	shape_with_harfbuzz(logicalRun, start, limit);
	start = limit;
}

// Compute line visual runs
UBiDi* pLineBiDi = ubidi_open();
ubidi_setLine(pParaBiDi, 0, str.size(), pLineBiDi, &err);
int32_t runCount = ubidi_countRuns(pLineBiDi, &err);

for (int32_t i = 0; i < runCount; ++i) {
	int32_t logicalStart, runLength;
	UBiDiDirection runDir = ubidi_getVisualRun(state.pLineBiDi, i, &logicalStart, &runLength);
	
	for (int32_t j = logicalStart; j < logicalStart + runLength; ++j) {
		LogicalRun& run = find_logical_run_by_index(j);
		draw_glyph(run.font, codepoints[j], positions[j].x, positions[j].y);
	}
}
```
This is considerably longer than the previous 2 implementations, but now the script and directionality handling is correct, and there is even a placeholder for handling multiple fonts. 
### Paragraphs and hard line breaks
The above code still has an important limitation: it errors when provided with a hard line break character. Specifically, the ICU implementation of the UAX #24 algorithm is unable to tolerate detecting scripts across hard line break boundaries.
Luckily, this is easily fixed by splitting your text into substrings that break on and exclude the following 4 characters:
- LF (U+A)
- CR (U+D)
- LSEP (U+2028)
- PSEP (U+2029)
Doing this is actually very convenient, as this will assist your line breaking later on, and assists with the first step of the UBA, which is breaking the text into paragraphs following this same logic.
### But what about the language?
Interestingly enough, from the sources I looked at (LayoutEx from icu4c, and Chromium), the language is specified as the default locale unless explicitly overriden by the user. With ICU this is done by doing:
```cpp
hb_language_from_string(icu::Locale::getDefault().getLanguage(), -1);
```
### But what about the character encoding?
One of the biggest limitations of ICU's UBA and UAX #24 implementations are that they are UTF-16 exclusive. This is not necessarily a deal-breaker depending on your goals, and Chromium, Firefox, and Windows all accept this limitation, for various reasons. Thus, if you would like to use ICU for text layout, keep in mind that all of your text must be in UTF-16.
Since my personal goals are to explicitly support native-UTF-8 text editing, having these algorithms maintain indices relative to the original UTF-8 string is a priority, and as such I chose [SheenBidi](https://github.com/Tehreer/SheenBidi) as my UBA library instead, and have my own custom UAX #24 implementation (although SheenBidi also provides one).
# Conclusion of Part 1
After reading this article, you should:
- Be familiar with basic typography and Unicode terminology as it pertains to text rendering
- Be able to correctly display any one-line horizontal Unicode string (minus some �'s from missing glyphs in your font file)

In part 2 we will explore font collections and selecting the right fonts for your text.