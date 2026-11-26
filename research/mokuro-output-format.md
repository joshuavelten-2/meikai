# mokuro output format and the Volume upload contract

Research for issue #13 (map: #1). Researched 2026-11-26.

## Sources (primary, pinned)

| Short name | Source |
|---|---|
| **mokuro** | `kha-white/mokuro` at `v0.2.5` (commit `9f79b1281066f953ec6e5d1e7086c401fa8da159`), the latest release (2026-07-20) |
| **ctd** | `kha-white/comic-text-detector` at `92ea867bd85a13105ebcf6bb7f7314784ca89902`, the submodule mokuro pins as `comic_text_detector/` |
| **zxy** | `ZXY101/mokuro-reader` at `7a1ff6b8f2696dbe8a9504b8733b482dcbcb2691` (2025-11-22), the original web reader |
| **gn** | `Gnathonic/mokuro-reader` at `1b0f755cb4fe17b9ce133fd6c4e174249b2d2509` (2026-11-07), the reader behind reader.mokuro.app; mokuro's README links here |
| **releases** | `gh api repos/kha-white/mokuro/releases` |

File references below are `repo:path:line`.

## TL;DR

- A `.mokuro` file is **one UTF-8 JSON object per Volume**. It holds metadata, plus one entry per Page with the image's pixel size and its OCR blocks. It **does not contain images**. Each Page names its image with `img_path`, a POSIX path **relative to the Volume's image folder** (or to the root of the Volume's zip/cbz).
- **There is no schema version.** `version` is just mokuro's package version string (e.g. `"0.2.5"`, `"0.2.0-beta.7"`). It appears at Volume level, and again on every Page with the version that OCR'd that Page, which can be different. The per-page shape (`box`, `vertical`, `font_size`, `lines`, `lines_coords`) has not changed since v0.1.0. The volume wrapper has not changed since v0.2.0. The only real format change is from v0.1.x to v0.2.0: v0.1.x wrote legacy HTML plus `_ocr/` per-page JSON, and v0.2.0 added `.mokuro`.
- Coordinates are **integer-valued pixels of the original image**, origin at the top left. `box` is `[x1, y1, x2, y2]` (ints). `lines_coords` is `[line][4 points][x, y]` (floats with integer values). The points run roughly TL, TR, BR, BL. **Boxes can extend past the image edges**, so clamp them instead of rejecting.
- `title`/`volume` are just **folder/file names**, and `title_uuid`/`volume_uuid` are random UUID4s. There is no author, number, language or reading direction.
- Recommended for meikai: **one `.zip`/`.cbz` containing exactly one `<name>.mokuro` at the root and the images.** Ingest matches images by exact `img_path`, and the match must be a strict bijection. Each block becomes one Text Box, with its lines kept as ordered, separately positioned segments of one text run.

## 1. What mokuro produces on disk

`mokuro` takes volume paths, each a directory, `.zip` or `.cbz` (`mokuro:mokuro/run.py:30-31`). The Series ("title") is the **parent directory** of the volume path (`mokuro:mokuro/volume.py:144-147`, `Title.name = path.name` at `:27`). Output for `manga/SeriesX/vol1/` (or `vol1.cbz`):

```
manga/SeriesX/
├── vol1/                 # input images (or vol1.cbz; zips are unzipped to a temp dir,
│   ├── 001.jpg           #   or in place with --unzip / legacy HTML)
│   └── ch2/002.png       # subfolders allowed: images are globbed recursively
├── vol1.mokuro           # NEW in 0.2.0: the volume JSON, sibling of the image folder/archive
├── vol1.html             # legacy HTML (still on by default in 0.2.5, deprecated)
└── _ocr/vol1/            # per-page OCR cache, mirrors the image tree
    ├── 001.json
    └── ch2/002.json
```

- `.mokuro` path: `<parent>/<dir name>.mokuro` for a dir, `<archive>.with_suffix(".mokuro")` for a zip/cbz (`mokuro:mokuro/volume.py:150-154`).
- `_ocr` cache: `<parent>/_ocr/<volume stem>/<img rel path>.json` (`mokuro:mokuro/volume.py:89-91`, `mokuro/mokuro_generator.py:47`).
- Image discovery is a recursive glob, natsorted, of extensions `.jpg .jpeg .png .webp .avif`, case-insensitive (`mokuro:mokuro/volume.py:99-107`). AVIF was added in v0.2.5 (releases) and WebP in v0.1.8.
- When a zip is unzipped and contains a single top-level folder with the same name as the archive, mokuro strips that folder (`correct_duplicated_root`, `mokuro:mokuro/utils.py:57-72`). So `img_path` is always relative to the **volume content root**, never prefixed with the archive's own name.
- The flag defaults in 0.2.5 are `legacy_html=True` and `as_one_file=True` (`mokuro:mokuro/run.py:23-25`). The run prints a deprecation warning and forces `unzip=True` when legacy HTML is enabled (`:51-59`).

## 2. The `.mokuro` schema (exact)

Written by `MokuroGenerator.generate_mokuro_file` (`mokuro:mokuro/mokuro_generator.py:73-98`). Pages come from `MangaPageOcr.__call__` (`mokuro:mokuro/manga_page_ocr.py:46-88`). It is serialised with `json.dump(ensure_ascii=False)` and a NumPy encoder (`mokuro:mokuro/utils.py:17-33`), so Japanese text is raw UTF-8 and NumPy numbers become plain JSON numbers.

### Annotated example

The values come from mokuro's own test fixture `tests/data/expected_results/test3_convert_legacy_ocr/vol1.mokuro`, trimmed. JSON has no comments; the `//` notes are annotations.

```jsonc
{
  "version": "0.2.0-beta.7",          // mokuro __version__ that WROTE this file (string, not semver-guaranteed)
  "title": "test3_convert_legacy_ocr", // = name of the parent directory  -> Series name candidate
  "title_uuid": "ebc9afa5-c1e7-41ce-b013-1d95671df7e8", // uuid4; shared by all .mokuro in that parent dir
  "volume": "vol1",                    // = stem of the .mokuro file / volume dir / archive
  "volume_uuid": "be16a2d9-1481-4949-9b6c-81422ec92c13", // uuid4, stable across re-runs (read back from existing .mokuro)
  "pages": [                           // natsorted by _ocr json path == natsort of image path minus extension
    {
      "version": "0.2.0-beta.6",       // mokuro version that OCR'd THIS page (may differ from top-level: cached pages are reused)
      "img_width": 827,                // int, pixel width of the decoded image
      "img_height": 1170,              // int, pixel height
      "blocks": [                      // in reading order (see section 4); [] if no text or --disable_ocr
        {
          "box": [37, 0, 863, 235],    // [x1, y1, x2, y2] ints, image px. NOTE x2=863 > img_width=827
          "vertical": false,           // bool: true => vertical text (tategaki), lines read right-to-left
          "font_size": 137.5,          // number, px; usually int, float when blocks were merged
          "lines_coords": [            // one quadrilateral per line, same length/order as "lines"
            [[582.0, 18.0], [785.0, 13.0], [787.0, 53.0], [583.0, 58.0]],
            [[37.0, 0.0], [863.0, 0.0], [863.0, 235.0], [37.0, 235.0]]
          ],
          "lines": ["ダイアリー・", "うちの猫ず日記"]   // OCR text per line (manga-ocr), strings
        }
      ],
      "img_path": "000a.jpg"           // POSIX path relative to the volume content root (backslashes normalised to "/")
    }
  ]
}
```

### Field reference

**Volume (top level).** Every key is always present (`mokuro_generator.py:78-85`):

| key | type | meaning |
|---|---|---|
| `version` | string | mokuro package version (`mokuro/_version.py`) |
| `title` | string | parent directory name |
| `title_uuid` | string (UUID4) | Series id. It is reused if any sibling `.mokuro` already has one, and new otherwise. If siblings disagree, mokuro warns and regenerates it (`volume.py:36-60`). |
| `volume` | string | `.mokuro` stem |
| `volume_uuid` | string (UUID4) | read from the existing `.mokuro` if present, otherwise new (`volume.py:69-75`) |
| `pages` | array of Page | may be empty |

**Page.** This is the per-page `_ocr` JSON plus `img_path`:

| key | type | meaning |
|---|---|---|
| `version` | string | mokuro version that produced the OCR |
| `img_width`, `img_height` | int | image size in px (`H, W = img.shape`; `manga_page_ocr.py:48-49`) |
| `blocks` | array of Block | |
| `img_path` | string | image path relative to the volume root (`mokuro_generator.py:92`) |

**Block** (`manga_page_ocr.py:55-86`):

| key | type | meaning |
|---|---|---|
| `box` | `[int,int,int,int]` | `[xmin, ymin, xmax, ymax]`, from `TextBlock.xyxy` (`ctd:utils/textblock.py:47`) |
| `vertical` | bool | orientation |
| `font_size` | number | px. `int(round(...))` normally (`ctd:utils/textblock.py:322,326`), but a float average after merging (`:351,373`) |
| `lines_coords` | `[[[num,num] ×4] × n]` | per-line quadrilateral. It is stored as int32 and emitted via `lines_array()` as float64 (`ctd:utils/textblock.py:109-110,340`), so the values are integer-valued floats like `582.0` |
| `lines` | `[string × n]` | text per line. `len(lines) == len(lines_coords)` always holds (both are appended in the same loop, `manga_page_ocr.py:85-86`) |

No other keys are written by mokuro.

## 3. Versions and the version field

- **v0.1.0 – v0.1.8 (legacy).** The output is `<vol>.html` plus `_ocr/<vol>/*.json`. The per-page JSON is already `{version, img_width, img_height, blocks:[{box, vertical, font_size, lines_coords, lines}]}` (checked via `gh api` at tags `v0.1.0`, `v0.1.5` and `v0.1.8`, `mokuro/manga_page_ocr.py`). v0.1.x writes **no `.mokuro` file**.
- **v0.2.0 (2024-07-07)** "introduce .mokuro format … deprecate legacy HTML" (releases). It adds the volume wrapper and `img_path`. It **reuses v0.1.x `_ocr` caches**. mokuro's test `test3_convert_legacy_ocr` covers turning old `_ocr` output into a `.mokuro`. It can also regenerate `_ocr` from an existing `.mokuro` (`mokuro_generator.py:34-42`).
- **v0.2.1 – v0.2.5.** No schema changes. The changes are CUDA/MPS support, numpy 2, XDG cache, AVIF input, and a fix for numeric paths (`v0.2.2`, "fix parsing numerical paths": the reason `img_path` is always a string).
- **There is no schema-version field.** `version` is a free-form package version. Pre-releases look like `0.2.0-beta.7`. meikai should **record it but not branch on it**: validate the structure instead. Any `.mokuro` that exists is ≥ 0.2.0 by construction.
- **Legacy HTML.** It is a full reader GUI with JS/CSS inlined (`as_one_file`). Each page is a `div.pageContainer` whose CSS `background-image` is the **relative path** to the image (`<vol dir>/<img_path>`), with one `div.textBox` per block (absolute `left/top/width/height` from `box`, font-size clamped 12–32px, `writing-mode: vertical-rl` if vertical) and one `<p>` per line (`mokuro:mokuro/legacy/overlay_generator.py:249-309`). It **drops `lines_coords`**, and images are referenced, not embedded. Recommendation: **do not accept HTML**. Its information is a subset of `.mokuro`, and the deprecated tool still writes both side by side.

## 4. Coordinate and ordering conventions

- **Space.** Pixels of the original decoded image (`img_width × img_height`), with the origin at the top left, x to the right and y downward. The detector runs at 1024 px, but box and line coordinates are rescaled back to original size before output (`ctd:inference.py:150-174`).
- **`box`** is an axis-aligned `[x1, y1, x2, y2]`, with x2/y2 exclusive-ish (legacy HTML uses `width = x2 - x1`). It is first the YOLO block box, then widened to enclose its lines (`adjust_bbox(with_bbox=True)`, `ctd:utils/textblock.py:89-100,482-485`). **It is not clipped to the image**: the fixture above has `x2 = 863 > 827`. Across mokuro's fixtures, 9 of 678 blocks exceed the bounds and none is negative. Do not assume this always holds; clamp to `[0, W] × [0, H]`.
- **`lines_coords[i]`** is a 4-point quadrilateral from `cv2.minAreaRect`, ordered by `get_mini_boxes` as top-left, top-right, bottom-right, bottom-left (`ctd:utils/db_utils.py:176-195`). Lines can be **rotated** (not axis-aligned), and for a block with no detected lines the quad is the block's own rectangle (`ctd:utils/textblock.py:461-468`). The ordering is heuristic, so treat the points as a polygon and take the AABB/centroid rather than trusting the corner order.
- **Line order inside a block.** Sorted by distance from the reading origin: lines start at the top right (`im_w, 0`) for vertical text and top left for horizontal (`ctd:utils/textblock.py:319-344`, `sort_lines`). So `lines[0]` is the first line to read: right-most for tategaki, top-most for yokogaki.
- **Block order on a page.** `sort_textblk_list` uses a grid-weighted order that approximates manga reading order (`ctd:utils/textblock.py:269-302`). It is a heuristic, so keep the file order as the default Text Box order.
- **`font_size`** is the estimated glyph size in image px (line thickness ÷ lines). Readers clamp it for display.

## 5. How the existing readers accept uploads

**ZXY101/mokuro-reader** (`zxy:src/lib/upload/index.ts`):
- It accepts dropped folders/files, or a single zip/cbz that it unzips and reprocesses (`:174-185`). It looks for `*.mokuro` (by extension) and images limited to jpeg/png/webp.
- It pairs a `.mokuro` with images whose `webkitRelativePath` starts with the `.mokuro` path minus its extension, i.e. the sibling `vol1/` folder (`:46-60,134-163`). The volume is keyed by `title_uuid` + `volume_uuid`, and duplicate `volume_uuid`s are skipped (`:215-233`).
- **It never reads `img_path`.** Pages are matched to images **by position** in sorted file order (`zxy:src/lib/components/Reader/Reader.svelte:265,350-352`), and `lines_coords` is not even in its `Block` type (`zxy:src/lib/types/index.ts:1-6`). This is fragile and not worth copying.

**Gnathonic/mokuro-reader (reader.mokuro.app)** (`gn:src/lib/import/*`):
- Pairing rules in priority order (`gn:src/lib/import/pairing.ts:1-18`): `.mokuro` in the same folder as the images, a `.mokuro` next to a same-name folder, a `.mokuro` next to a same-name archive, "TOC" (a `.mokuro` above chapter subfolders), an archive with an internal `.mokuro`, and finally image-only (no OCR). It accepts zip/cbz/cbr/rar/7z (`gn:src/lib/import/types.ts:334`) and a wide list of images, including gif/bmp/tiff/jxl (`:316-331`).
- Parse validation only checks that `version, title, title_uuid, volume, volume_uuid, pages` are present (`gn:src/lib/import/processing.ts:167-197`). It also reads its own **extension keys** if present: `chars`, `spine_width`, `ocr_engine`, `edited`. Files re-exported by that reader may contain them.
- Image matching (`gn:src/lib/import/processing.ts:233-310`) goes in order: exact `img_path` (case-insensitive, `\`→`/`), then **same stem with a different extension** (for re-encoded images), then, if under 50% matched and the counts agree, **positional fallback**. Missing and extra images are tolerated.
- Its own export (`gn:src/lib/util/compress-volume.ts:101-160,457-512`) writes `<Volume Title>/` with the images inside it, plus `<Volume Title>.mokuro` **at the archive root**, and optionally `series.json` and `<title>.<layer>.mokuro` files beside it. Note that the images sit **under a folder** while `img_path` is relative to that folder. The pairing rule "`.mokuro` next to same-name folder" resolves this.
- Rendering uses `lines_coords` to place each line as an `inline-block` span inside one block element, moved with `transform`. A per-line `position:absolute` "would inject a hard break at every line and split words/sentences across lines" for Yomitan/Migaku (`gn:src/lib/components/Reader/TextBoxes.svelte:398-420`, their issue #254). This matches meikai's one-inline-run requirement.

## 6. Recommended strict upload contract for meikai

meikai can be stricter than both readers because every upload must carry mokuro output and no OCR runs server-side.

**Container.** One file per Volume upload: a ZIP archive with the `.zip` or `.cbz` extension (both are the same format). It must use the stored or deflate methods only, with no encryption. RAR and 7z are rejected.

**Layout.** Exactly one of these two layouts is accepted. Both are what mokuro and Gnathonic naturally produce.

```
A (flat)                      B (mokuro / Gnathonic export layout)
vol1.cbz                      vol1.zip
├── vol1.mokuro               ├── vol1.mokuro
├── 001.jpg                   └── vol1/            <- folder name == .mokuro stem
└── ch2/002.png                   ├── 001.jpg
                                  └── ch2/002.png
```

The **image root** is the archive root in layout A, or `<stem>/` in layout B. Every `img_path` resolves against the image root. Ingest picks B if and only if a top-level directory named exactly like the `.mokuro` stem exists. Users zip `SeriesX/vol1/` plus `SeriesX/vol1.mokuro` to get B.

**Validation rules (reject the upload on any violation and report all errors):**

Archive hygiene:
1. The upload is within the size limit, and the total uncompressed size and entry count are within limits (zip-bomb guard: check the declared sizes, enforce them while streaming, and cap the compression ratio).
2. Entry names: reject absolute paths, `..` segments, backslashes, NUL/control characters and symlinks, and reject duplicates after Unicode NFC normalisation (do **not** case-fold; mokuro matches paths exactly). The names must be UTF-8 (ZIP flag bit 11) or plain ASCII.
3. Ignore (do not count) OS junk such as `__MACOSX/`, `.DS_Store`, `Thumbs.db` and `desktop.ini`. Reject any other file that is not the `.mokuro` or an image, including `_ocr/`, `.html` and any extra `.mokuro`/layer file. This keeps "exactly one" meaningful.
4. There is **exactly one** `*.mokuro` file, at the archive root.

`.mokuro` JSON:
5. It is valid UTF-8 JSON (BOM tolerated) within a size cap. The top level is an object with `version`, `title`, `volume` (non-empty strings), `title_uuid` and `volume_uuid` (UUID strings) and `pages` (a non-empty array). Unknown keys are **ignored** (e.g. Gnathonic's `chars`, `ocr_engine`), never stored as trusted data.
6. Each page has `img_path` (a non-empty string that is a valid relative POSIX path, with the same rules as rule 2), `img_width` and `img_height` (integers from 1 to a max dimension), and `blocks` (an array; empty is allowed). The page `version` is an optional string.
7. Each block has `box` (4 finite numbers with `x1 ≤ x2` and `y1 ≤ y2`), `vertical` (bool), `font_size` (a finite number > 0), and `lines` and `lines_coords` arrays of **equal length ≥ 1**. Each `lines_coords[i]` holds exactly 4 points of 2 finite numbers. Each line is a string, NFC-normalised on import. Coordinates may lie up to some tolerance (e.g. 10% of W/H) outside the image and are **clamped** on import; anything wildly out of range is rejected.
8. `img_path` values are unique across pages.

Pairing:
9. There must be a **strict bijection** between `pages[].img_path` and the image files under the image root, by exact path. Any missing or unreferenced image fails the upload. meikai skips Gnathonic's stem and positional fallbacks, because they hide mismatched OCR.
10. Each image decodes (accepted types: jpeg, png, webp, avif, matching mokuro's set; sniff the magic bytes, don't trust extensions). Its **decoded size must equal `img_width × img_height`**, which catches resized or re-encoded images whose geometry no longer matches. Apply EXIF orientation consistently: mokuro reads images through PIL without `exif_transpose`, so the stored pixel dimensions are the reference.

**Page order** is the `pages` array order. mokuro already natsorts it, and the file defines it explicitly.

**Metadata use.** `title` and `volume` are only *suggestions* for Series and Volume names. They are folder names, often like `SeriesX` or `vol1`, so the Reader confirms or edits them in the UI. Store `title_uuid`/`volume_uuid` to suggest grouping into an existing Series and to warn about duplicate uploads, but never use them as meikai primary keys, because they are user-controlled. Store `version` as provenance.

## 7. Proposed mapping: mokuro block → Text Box

One mokuro **block** becomes one **Text Box**. A block is the detector's speech-bubble or caption region, and its lines are one utterance.

| Text Box field | From | Notes |
|---|---|---|
| page | page index in `pages` | |
| order | block index within the page | keeps mokuro/ctd's reading-order heuristic |
| bbox (x, y, w, h) | `box`, clamped to the image | store in image px. The client scales to rendered size, or meikai stores it normalised (÷W, ÷H). Pick one, ideally px plus the page W/H. |
| writing_mode | `vertical` → `vertical-rl`, else `horizontal-tb` | |
| font_size_px | `font_size` | display hint. The client fits or clamps it, as legacy HTML clamps to 12–32 and Gnathonic fits it |
| lines[] | zip(`lines`, `lines_coords`) | ordered. Each line keeps its text plus its quad (4 points, clamped), or a derived AABB plus rotation angle |
| text | `"".join(lines)` | for search and Yomitan. Join with **no separator**, because Japanese has no spaces between lines; a newline would break lookup across line wraps |

**Rendering contract (reader).** Render each Text Box as one absolutely positioned container with the correct `writing-mode`. Inside it, render one inline (or `inline-block`) span per line in `lines` order, with no `<br>`, no block elements and no per-line `position:absolute`. Each span is moved onto its quad with a CSS `transform`, which does not take it out of flow. The DOM text is then a single continuous run that Yomitan can scan across line breaks, while each glyph still sits over the printed text. This is the technique Gnathonic shipped for exactly this problem (`gn:src/lib/components/Reader/TextBoxes.svelte:398-420`).

**Edge cases to decide during implementation.** These are not blocking for the contract:
- A block whose `lines` are all empty strings: drop it on import.
- Overlapping blocks: legacy HTML z-orders larger boxes underneath (`overlay_generator.py:252-258`). The reader should do the same, so small boxes stay hoverable.
- Pages with `blocks: []` (`--disable_ocr` or textless pages) are valid. A Volume whose pages are *all* empty could get a warning, since it was probably run with OCR disabled.
