# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [Unreleased]

---

## [3.1.0] — 2026-04-13

### Changed
- Replaced `claude-opus-4-5` with `claude-sonnet-4-6` as default OCR model; `claude-opus-4-5` remains available via the `MODEL` config parameter
- Refactored `ocr_document()` signature: added `hint`, `model`, and `max_tokens` parameters for explicit per-call control
- Renamed internal image loader to `_load_image()` (private); extracted from inline logic in `ocr_document()`
- Renamed OCR prompt constant to `_OCR_PROMPT` (private)
- Refactored PDF builder: renamed `flush_table()` to `flush()`, tightened internal state management
- Updated `build_pdf()` signature to accept `page_size`, `font_size`, and `margins_mm` as explicit parameters
- Adjusted default `FONT_SIZE` from 9 pt to 8 pt and `MARGINS_MM` from 15 mm to 12 mm
- Increased default `H1` heading font size by 1 pt relative to 3.0

### Added
- `landscape()` page size support (`landscape(A4)`, `landscape(letter)`)
- `DOCUMENT_HINT` config variable replacing `DOCUMENT_CONTEXT`; description and examples updated in-notebook
- `PageBreak` import from `reportlab.platypus`
- `_esc()` helper for ReportLab XML character escaping in table cells
- Two-page spread transcription rule: left page transcribed in full before right page
- `max_tokens` parameter on `ocr_document()` with default 8192; documented as adjustable for long documents
- Per-call token usage reported as formatted string `(N chars | in N tok out N tok)`

### Fixed
- Table cell padding corrected to 2 pt top/bottom, 3 pt left/right (was inconsistent across row types in 3.0)
- `GRID` style now applied uniformly before per-row background overrides, preventing style ordering conflicts

---

## [3.0.0] — 2026-04-13

### Changed
- **Architecture pivot:** replaced image-processing pipeline (OpenCV / scipy) with Claude Vision API OCR + ReportLab PDF typesetter
- Pipeline input is now a document image (JPEG/PNG/TIFF/BMP/WebP); output is a clean typeset PDF rather than a processed raster image
- `Config` dataclass removed; replaced with flat module-level constants (`INPUT_IMAGE`, `OUTPUT_PDF`, `PAGE_SIZE`, `FONT_SIZE`, `MARGINS_MM`, `DOCUMENT_CONTEXT`)
- Dependencies reduced to `anthropic`, `reportlab`, `Pillow`, `tqdm`; `opencv-python-headless`, `scipy`, `ipywidgets` removed

### Added
- `ocr_document(image_path, context)` — sends image to Claude Vision API (`claude-opus-4-5`, `max_tokens=8192`); returns structured plain-text transcription
- `BASE_OCR_PROMPT` — structured transcription protocol defining eight prefixed line types: `HEADING`, `SUBHEADING`, `SECTION`, `FIELD`, `TABLE_HEADER`, `TABLE_ROW`, `TEXT`, `NOTE`
- `image_to_base64(path)` — loads image file and returns `(base64_data, media_type)`; auto-converts TIFF/BMP to JPEG
- `build_pdf(transcription, output_path, page_size, font_size, margins_mm)` — parses structured transcription and renders typeset PDF via ReportLab `SimpleDocTemplate`
- `flush_table()` — internal helper; accumulates and emits buffered `TABLE_HEADER` / `TABLE_ROW` blocks as a `reportlab.platypus.Table` with grid lines and shaded header rows
- `DOCUMENT_CONTEXT` config variable; optional one-sentence document description appended to OCR prompt to improve accuracy on specialist vocabulary and unusual layouts
- `IFrame` display for inline PDF preview in Jupyter
- API key resolution: environment variable → direct assignment → `input()` prompt fallback

---

## [2.1.0] — 2026-04-13

### Changed
- **`black_on_white` pipeline:** replaced Gaussian-background illumination correction with Sauvola adaptive thresholding
  - Previous method: large-Gaussian background estimation → division normalisation → uniform_filter adaptive threshold
  - New method: per-pixel local mean + standard deviation via `uniform_filter(arr²)` → Sauvola threshold `T = mean × (1 + k × (std/R − 1))`
  - Eliminates blur artefacts introduced by background subtraction; produces sharper ink edges on high-contrast documents
- `bow_bg_sigma` parameter removed from `black_on_white` mode (background estimation step eliminated)
- `bow_final_threshold` default lowered from 180 to 128 to match Sauvola output range

### Added
- `bow_sauvola_k` config parameter (default `0.20`; range `0.10–0.35`): controls ink sensitivity in Sauvola threshold
- `bow_sauvola_R` config parameter (default `128.0`): Sauvola normalisation constant
- `bow_adaptive_window` default updated to 51 px (odd value required for symmetric neighbourhood); documented as controlling local window for both mean and std computation

### Removed
- `bow_bg_sigma` config parameter (Gaussian background step eliminated in `black_on_white` mode)

---

## [2.0.0] — 2026-04-13

### Added
- `black_on_white` processing mode — full pipeline: background illumination correction → percentile histogram stretch → adaptive local thresholding → median denoise → stroke thickening → hard binary threshold
- `restore_black_on_white(pil_img, cfg)` core function implementing the above pipeline
- `Config` parameters for `black_on_white` mode:
  - `bow_upscale_processing` (default `3`) — internal processing upscale factor
  - `bow_output_upscale` (default `2`) — final output upscale relative to original
  - `bow_bg_sigma` (default `80`) — Gaussian sigma for background estimation
  - `bow_adaptive_window` (default `60`) — uniform_filter neighbourhood size
  - `bow_threshold` (default `15`) — ink detection sensitivity
  - `bow_noise_size` (default `3`) — median filter kernel size
  - `bow_stroke_size` (default `2`) — minimum_filter stroke thickening kernel
  - `bow_final_threshold` (default `180`) — hard binary cutoff (0–255)
- `PRESET_E` — `black_on_white` standard configuration
- `PRESET_F` — `black_on_white` heavy-damage configuration (`bow_bg_sigma=120`, `bow_threshold=10`, `bow_stroke_size=3`, `bow_final_threshold=160`)
- Preview panel label `⬛ Black-on-White` in `preview_result()` output

### Changed
- Default `Config.mode` changed from `"binarize"` to `"black_on_white"`
- `restore_document()` branching updated to handle `"black_on_white"` mode; output suffix `_black_on_white`
- `PRESET_D` (batch folder) default mode updated to `"black_on_white"`

---

## [1.0.0] — 2026-03-12

### Added
- `Config` dataclass with full parameter set for input path, output directory, processing mode, output format, deskew, illumination correction, binarization, enhance, and PDF options
- `detect_skew_angle(img_gray, max_angle)` — Hough line transform skew detection; returns median angle in degrees
- `deskew(pil_img, max_angle)` — BICUBIC rotation correction; no-op below 0.1° threshold
- `restore_binarize(pil_img, cfg)` — binarization pipeline: illumination correction → percentile histogram stretch → adaptive local threshold → median speckle removal → stroke thickening → auto-contrast → hard binary output
- `restore_enhance(pil_img, cfg)` — sharpen/contrast pipeline: unsharp mask → contrast enhance → brightness enhance → multi-pass sharpen → auto-contrast; no binarization
- `save_as_pdf(pil_img, pdf_path, cfg)` — single-page PDF output via ReportLab `Canvas`; supports A4 and letter with proportional fit and 5% margin
- `get_input_files(input_path)` — resolves single file or directory to sorted list of supported image paths
- `restore_document(image_path, cfg)` — orchestrates deskew + selected pipeline(s); returns result dict with output paths and PIL images
- `restore_all(cfg)` — batch processor over all files at `cfg.input_path`; skips failed files with warning
- `pil_to_b64(img, max_width)` — PIL-to-base64 PNG converter for HTML preview
- `preview_result(original_path, result)` — side-by-side HTML comparison widget (original / binarized / enhanced)
- Processing modes: `"binarize"`, `"enhance"`, `"both"`
- Output formats: `"png"`, `"pdf"`, `"both"`
- Upscale factor: 1×, 2×, 3× (LANCZOS resampling)
- Presets A–D: historical documents, modern office scan, photo/mixed content, batch folder
- Supported input formats: JPEG, PNG, TIFF, BMP, WebP
- Dependencies: `Pillow`, `scipy`, `numpy`, `opencv-python-headless`, `reportlab`, `ipywidgets`, `tqdm`

---

[Unreleased]: https://github.com/username/document-restoration-tool/compare/v3.1.0...HEAD
[3.1.0]: https://github.com/username/document-restoration-tool/compare/v3.0.0...v3.1.0
[3.0.0]: https://github.com/username/document-restoration-tool/compare/v2.1.0...v3.0.0
[2.1.0]: https://github.com/username/document-restoration-tool/compare/v2.0.0...v2.1.0
[2.0.0]: https://github.com/username/document-restoration-tool/compare/v1.0.0...v2.0.0
[1.0.0]: https://github.com/username/document-restoration-tool/releases/tag/v1.0.0
