# Yomitan compatibility with a transparent text overlay

Research for issue #3, which is part of map #1 (meikai V1 architecture).

**Question:** Can Yomitan reliably look up text rendered as a transparent DOM overlay on top of a manga Page image (the mokuro approach)? What constraints does that put on the reader, and which DOM patterns work or break hover lookup?

**Short answer:** Yes. Yomitan's hover scan is built on `document.caretPositionFromPoint` / `caretRangeFromPoint` plus its own DOM walker. Any real DOM text node that is the top hit-testable thing under the cursor can be scanned, and that includes vertical text, absolutely positioned boxes and CSS-transformed (zoomed) text. Lookups break when the text cannot be hit-tested, when the walker decides the text is "invisible", or when the walker sees a hard break inside a Text Box. That last case is the one that bites manga readers in practice.

## Sources

All claims were read from these primary sources (source code at the commit noted, plus issue threads):

| Ref | Source |
|---|---|
| Y-gen | `yomidevs/yomitan` @ `64b4eba` (2026-11-26), `ext/js/dom/text-source-generator.js` |
| Y-scan | same commit, `ext/js/dom/dom-text-scanner.js` |
| Y-util | same commit, `ext/js/dom/document-util.js` |
| Y-range | same commit, `ext/js/dom/text-source-range.js` |
| Y-ts | same commit, `ext/js/language/text-scanner.js` |
| Y-popup | same commit, `ext/js/app/popup.js` |
| Y-schema | same commit, `ext/data/schemas/options-schema.json` |
| Y-manifest | same commit, `dev/data/manifest-variants.json` |
| M-legacy | `kha-white/mokuro` @ `9f79b12`, `mokuro/legacy/overlay_generator.py`, `styles.css`, `script.js` |
| MR-zxy | `ZXY101/mokuro-reader` @ `7a1ff6b`, `src/lib/components/Reader/TextBoxes.svelte`, `MangaPage.svelte` |
| MR-gn | `Gnathonic/mokuro-reader` @ `1b0f755` (active fork behind reader.mokuro.app), `src/lib/components/Reader/TextBoxes.svelte` |
| Issues | yomidevs/yomitan #1044, #423, #855, #1636, #2311; Gnathonic/mokuro-reader #254, #124, #101, #141, #7, #24 |

Note: `yomitan/yomitan` does not exist. The canonical repository is `yomidevs/yomitan`, formerly `themoeway/yomitan`.

## How Yomitan finds the text under the cursor

1. **Event capture.** Pointer, mouse and touch listeners are registered on `window` with `capture = true` (Y-ts L1128-1166, `node: window` in `frontend.js`). The reader's own `stopPropagation` in bubble or target phase therefore does not block Yomitan.
2. **Top element check.** `document.elementFromPoint(x, y)` runs first. If "deep content scan" is on (`deepDomScan`, default `false`, Y-schema) it uses `elementsFromPoint` instead. If the top element is an `IMG`, `BUTTON` or `SELECT`, Yomitan **returns that element itself** and scans its `alt` text instead of any text (Y-gen L201-206; `text-source-element.js` L231-235). `INPUT`/`TEXTAREA` get an "imposter" div.
3. **Caret position.** `caretPositionFromPoint` (Firefox, Chrome 128+) runs, falling back to `caretRangeFromPoint` (Y-gen L410-429). Open shadow roots are discovered via `element.shadowRoot` and passed as `{shadowRoots}` (Y-gen L435-476).
4. **Validation.** The range must start in a **text node**, and the point must lie inside `range.getClientRects()` of the adjacent non-whitespace character (Y-gen L363-402; `isPointInAnyRect` in Y-util). If the hit is not text, and deep scan is on, Yomitan temporarily sets `pointer-events: none` on "transparent" covering elements and retries. A covering element counts as transparent if it has opacity 0, `visibility: hidden`, or no background image and a transparent background colour (Y-gen L577-681). With deep scan off (the default), there is no retry.
5. **Reading text.** `DOMTextScanner` walks forward from the caret by `firstChild`/`nextSibling` and collects up to `scanning.length` characters (default 16). The result is sent to the dictionary (Y-ts L1254-1270). The walker rules, all in Y-scan `getElementSeekInfo` and `isStyleVisible`, are:
   - It never enters `RT`, `SCRIPT`, `STYLE`, `HEAD`, `TEXTAREA`, `INPUT` or `BUTTON`, so ruby furigana is skipped.
   - It does not enter elements that are `display:none`, `visibility:hidden`, `opacity <= 0`, `font-size <= 0`, or both **unselectable** (`user-select:none`) **and** transparent-coloured (`color` or `-webkit-text-fill-color` is `rgba(...,0)`).
   - With **layout-aware scanning** on (`layoutAwareScan`, schema default `true`; Y-schema L806), entering or leaving a `position:absolute|fixed|sticky` element inserts **2 newlines**, and a block, flex, grid, table or list element inserts **1 newline**. A `<br>` inserts 1 newline. `inline` and `inline-block` insert nothing.
   - The search text is passed to the translator as-is. Nothing in `translator.js` strips `\n`, so **a newline ends the lookup**: a word that straddles a newline cannot match.
6. **Highlight and popup.** The matched range is highlighted with the real `Selection` API (`selectText`). The popup iframe is appended to `document.body`, or to the fullscreen element unless that element is an iframe or a shadow host (Y-popup L863-884). The popup is placed using the range's client rects and `getComputedStyle(parent).writingMode`. In vertical modes it is placed beside the column (`popupVerticalTextPosition: 'before'`).

## What works

| Pattern | Verdict | Why / evidence |
|---|---|---|
| Absolutely positioned Text Box `div`s over the page | Works | mokuro, ZXY101 and Gnathonic all do this (M-legacy `get_box_style`, MR-zxy, MR-gn). The boundary also acts as a sentence break with layout-aware scan (2 newlines), which keeps neighbouring bubbles out of `{sentence}`. |
| `writing-mode: vertical-rl` on the box | Works | Used by all three mokuro implementations. Yomitan reads `writingMode` for popup placement (Y-util `getElementWritingMode`, Y-range `getWritingMode`). Hit-testing and `getClientRects` are writing-mode-aware in browsers. |
| Zoom/pan with a CSS `transform` (scale/translate matrix) on an ancestor | Works | mokuro legacy and ZXY101 use `panzoom`, which sets a `transform` matrix. The browser hit-tests transformed glyphs, and `getClientRects` returns viewport-space boxes. Gnathonic additionally translates and **rotates** individual lines with `transform` and documents that `elementFromPoint`/`caretRangeFromPoint` follow the transformed glyphs (MR-gn comments L401-420). |
| Page image as CSS `background-image` on the page container | Works, and is the safest option | All mokuro implementations do this (M-legacy `get_container_style`, MR-zxy `MangaPage.svelte`, MR-gn). It means the top element is never an `<img>`, so Yomitan's IMG/alt-text branch (step 2) never triggers. |
| `<img>` under the overlay | Works only if the text is on top | If any point over text resolves to the `IMG` via `elementFromPoint`, Yomitan scans the image's `alt` instead of the text. Give the `<img>` an empty `alt=""` and make sure text boxes sit above it with a higher z-index. |
| `color: transparent` with `user-select: text` | Works | The walker treats transparent text as invisible only when it is also unselectable (Y-scan `isStyleVisible`). Transparent glyphs still hit-test, and the Selection highlight still paints, so the reader can style `::selection` to show what was matched. |
| Lines of a box as **inline** spans in one block, with visual breaks from CSS | Works, and is the recommended pattern | Gnathonic uses this to fix #254 and #124 ("words can't carry over to the next line"). Each line is an inline or inline-block `span` inside one `<p>`. The visual break comes from `.ocr-line:not(:last-child)::after { content: '\A'; white-space: pre; }`. Generated content is not in the DOM, so the walker sees one continuous run (MR-gn L854-862). In per-line mode, lines are `inline-block` in normal flow and moved onto their OCR quad with `transform` (MR-gn L398-420, L826-846). |
| Reveal-on-hover text (mokuro style) | Works | mokuro and ZXY101 hide lines with `display:none` and show them on `.textBox:hover`. Gnathonic does the same with `visibility:hidden`. This works only because the hover reveal happens before Yomitan's scan, so the text is visible when scanned. It is not a transparent overlay. |
| Same-document React SPA | Works | Content scripts run with `all_frames`, `match_about_blank` and `match_origin_as_fallback` (Y-manifest L47-49), but a plain single-document SPA avoids frame issues entirely. |

## What breaks or degrades

| Pattern | Effect | Evidence |
|---|---|---|
| Text drawn on `<canvas>` or baked into the image | No lookup. There is no DOM text node, so the caret lands on the canvas element and the "must be a text node" check fails. | Y-gen L363-368 |
| `pointer-events: none` on the text or its box | No hover lookup. The element is skipped by hit-testing, so the caret resolves to whatever is underneath, normally the page container, which has no text. Deep scan does not help, because it only disables covering layers and cannot re-enable the text. | Y-gen L223, L577-631 (hit-test based) |
| A transparent layer **above** the text, such as a gesture-capture div or a pan/zoom hit area | No lookup with default settings (deep scan off). With deep scan on, only layers that look transparent are bypassed. | Y-gen L577-631, Y-schema `deepDomScan: false` |
| `opacity: 0` on the box or its lines | Unreliable. Opacity-0 elements still hit-test, but the walker refuses to *enter* any element whose own computed opacity is 0. Multi-span text gets truncated, and the highlight is invisible. | Y-scan `isStyleVisible` |
| `visibility: hidden` or `display: none` while not hovered | No lookup until revealed. The element is not hit-testable and the walker will not enter it. | Y-scan `getElementSeekInfo` |
| `color: transparent` **plus** `user-select: none` | Treated as invisible. The walker skips such elements, and the Selection highlight cannot apply. | Y-scan `isStyleVisible` |
| Lines of a box as **block** elements (`<p>` per line, `display:block/table`), `<br>`, or `position:absolute` per line | Words split across columns are not found, and `{sentence}` is cut to one line. This is the mokuro legacy markup (one `<p>` per line, shown as `display:table`) and was the cause of Gnathonic #124, #254 and #101. | Y-scan newline rules; MR-gn comment L398-404; M-legacy `overlay_generator.py` |
| Literal `\n` characters in a text node with `white-space: pre*` | Same problem as `<br>`: the newline ends the lookup. | Y-scan `_getWhitespaceSettings` / `getCharacterAttributes` |
| Shadow DOM | Fragile. Open roots are found by brute-force `querySelectorAll('*')`. Closed roots are invisible. Firefox throws "operation is insecure" when a range is created in a shadow root, so the lookup returns null. Chromium shadow-root scanning was broken until the `caretPositionFromPoint` path landed. Also, the walker uses `firstChild`/`nextSibling` and never descends into a shadow root. | Y-gen L435-505; yomidevs/yomitan #1044 |
| Iframes | Each frame needs its own Yomitan instance. Sandboxed, `srcdoc` or blob frames have historically failed (e.g. Protonmail). Avoid putting reader pages in iframes. | Y-manifest; yomidevs/yomitan #423 |
| Text clipped by `overflow: hidden`, or pushed outside its box by overflow | Clipped glyphs cannot be hit-tested, and glyphs pushed off position fail the "point inside the character's client rect" check. If text overflows into the margins in vertical mode, a whole paragraph can become unscannable. | Y-gen L384-399; yomidevs/yomitan #855 |
| Reader UI text near the overlay in DOM order (page counter, zoom %, toasts, route announcer) | Leaks into the `{sentence}` Anki field. | Gnathonic #141 (zoom % prepended), #7/#24 and yomidevs/yomitan #1636 (Svelte announcer "Volume" appended), #2311. mokuro legacy works around it by inserting a 1px `。` (M-legacy L176-178). |
| CSS `zoom` property for page zoom | Probably works, but is riskier than `transform`. Yomitan divides coordinates by the computed `zoom` of ancestors when `normalizeCssZoom` is on (default `true`; Y-util `computeZoomScale`). Browsers changed `zoom` semantics in 2024 ("standardized CSS zoom"), so the correction may not match the rects. *Not verified in a browser; inference from source.* | Y-util L34-92, Y-schema L836 |

## Font sizing and geometry

- Hover scanning needs the **DOM glyph under the cursor to coincide with the printed glyph on the image**, because the hit-test point must fall inside the character's client rect. Size the text so that each line spans its OCR line extent. A mis-sized font still scans, but the hotspot no longer matches what the Reader sees.
- mokuro's block `font_size` overstates the true size, because it measures the quad including furigana. Gnathonic therefore derives per-line font size and letter-spacing from `lines_coords` quads and uses `font-kerning: none` for fixed-pitch print (MR-gn L95-104, L826-846). mokuro legacy simply clamps the font size to 12-32px and uses `white-space: nowrap` (M-legacy `get_box_style`).
- Use `white-space: nowrap` (or explicit inline-block line sizing) so that the browser's own wrapping does not reflow lines differently from the print.
- If furigana is ever rendered, use `<ruby><rt>`. The walker skips `RT`, so lookups read only the base text (Y-scan).

## Recommendations for meikai's overlay

1. **Page container.** Render the page as a `div` with the image as `background-image`. If you use an `<img>`, give it `alt=""` and keep it strictly below the text layer. Never put a hit-testable layer above the text layer. Pan/zoom gesture handling belongs on the container itself, or on an ancestor via capture listeners, not on an overlay sheet.
2. **One absolutely positioned element per Text Box**, positioned in image-pixel coordinates inside the page container, with `writing-mode: vertical-rl` for vertical boxes and `horizontal-tb` otherwise. Order the boxes in the DOM in reading order. The absolute positioning gives free sentence isolation between bubbles.
3. **Lines inside a box must be inline in one text run.** Use `<span class="line">` children of a single block, with visual breaks from `::after { content: '\A'; white-space: pre }`, or with inline-block lines moved by `transform`. Do **not** use a block element, a `<br>`, an absolutely positioned element or a literal `\n` per line.
4. **Invisibility.** Use `color: transparent` (and `-webkit-text-fill-color: transparent` if needed) with `user-select: text`. Do not use `opacity: 0`, `visibility: hidden`, `display: none`, `pointer-events: none` or `user-select: none` on text. Style `::selection` so that the Yomitan highlight is visible on the transparent text. A mokuro-style "reveal on hover with a white background" mode is also compatible, as an option.
5. **Zoom and fit.** Apply a single `transform: translate() scale()` to the page or spread container. Prefer this over the CSS `zoom` property and over re-laying out text at new font sizes. Text then scales with the image and stays aligned at every fit mode. Single and double spreads and RTL paging are just layout of page containers and do not affect scanning.
6. **Font sizing.** Compute the font size and letter-spacing per line from the OCR line geometry so that glyphs land on the printed glyphs. Use a CJK font with predictable metrics, `font-kerning: none` and `line-height: 1`. Avoid `overflow: hidden` on boxes so that glyphs are never clipped.
7. **No canvas, iframes or shadow DOM** for the overlay or for page rendering.
8. **Keep reader UI text away from the overlay.** Do not place page numbers, zoom percentages, toasts or live-region announcers next to page text in DOM order. Render transient labels outside the page subtree, after it in a separate top-level node, or as CSS generated content. Verify the Anki `{sentence}` field during QA.
9. **QA matrix.** Check Chrome and Firefox; vertical and horizontal boxes; a word straddling two columns; zoomed in and out; a double-page spread; layout-aware scan on and off; and the Anki `{sentence}` field contents.
