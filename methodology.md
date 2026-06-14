# Methodology

This document describes the technical approach, algorithmic design, and architectural evolution of the Document Restoration Tool across all six research versions. Two distinct processing paradigms are covered: a raster image-processing pipeline (v1.0–v2.1) and a Vision API transcription and PDF typesetting pipeline (v3.0–v3.1).

---

## 1. Research Problem

Historical document scans degrade along several axes simultaneously: uneven illumination from physical warping and shadow; age-related discolouration (yellowing, foxing, staining); ink fading and bleed-through from opposite pages; mechanical skew from flatbed or camera capture; and speckle noise from scanner sensors and deteriorated paper fibre. Standard image enhancement — contrast boost, sharpening — addresses none of these at the source. Each artifact requires a targeted correction at the appropriate stage of a processing pipeline.

The primary test corpus for all versions is a British West African Meteorological Services monthly observation record, Station Lungi, January 1955. The document presents all degradation types simultaneously: heavy background yellowing, shadow gradients from physical curl, mixed print and handwriting, and tabular data requiring column alignment to be useful. It was selected as the reference corpus because passing it through any pipeline variant provides an unambiguous quality signal.

---

## 2. Era 1 — Image Processing Pipeline (v1.0–v2.1)

### 2.1 Design Premise

The image-processing approach treats document restoration as a signal separation problem: the goal is to isolate the ink signal from the background noise introduced by age, damage, and capture conditions. All processing operates on grayscale pixel arrays using NumPy and SciPy; no machine learning inference is involved. The output is a processed raster image (PNG or PDF-embedded image) at a specified upscale factor relative to the original.

### 2.2 Pipeline Architecture

All three image-processing versions share a common stage sequence, with differences concentrated in the thresholding step (Stage 3):

```
Input image
    │
    ▼
[Stage 0]  Format normalisation
           Convert to grayscale (PIL L mode)
           Upscale N× via LANCZOS resampling
    │
    ▼
[Stage 1]  Deskew
           Hough line transform → median skew angle → BICUBIC rotation
           No-op below 0.1° threshold
    │
    ▼
[Stage 2]  Illumination correction   ← varies by pipeline
           (binarize / v2.0 black_on_white only)
    │
    ▼
[Stage 3]  Adaptive thresholding     ← primary algorithmic variation across versions
    │
    ▼
[Stage 4]  Speckle removal
           Median filter (configurable kernel size)
    │
    ▼
[Stage 5]  Stroke thickening
           minimum_filter expansion of ink pixels
    │
    ▼
[Stage 6]  Auto-contrast
           PIL ImageOps.autocontrast (1% cutoff)
    │
    ▼
[Stage 7]  Output normalisation
           Downscale to final upscale factor
           Hard binary threshold → pure black / white
    │
    ▼
Output PNG / PDF
```

The `enhance` mode is a parallel, lighter pipeline without binarization: unsharp mask → contrast enhance → brightness enhance → multi-pass sharpen → auto-contrast. It is designed for photographs and mixed-content pages where a clean binary output is inappropriate.

### 2.3 Stage Detail: Deskew

Skew detection uses OpenCV's probabilistic Hough line transform (`cv2.HoughLinesP`) applied to a Canny edge map of the grayscale input. The transform returns line segment endpoints; the angle of each segment is computed via `math.atan2` and filtered to the range `±max_skew_angle` (default 15°). The median of retained angles is taken as the document skew estimate, providing robustness against non-text edges (borders, rules, noise).

Correction is a BICUBIC rotation by the negative of the detected angle, with `expand=True` to prevent clipping and `fillcolor=255` (white) for introduced border pixels. Segments with fewer than `img_width / 4` pixels are discarded to reject short local features that do not reflect global page orientation.

### 2.4 Stage Detail: Illumination Correction (v1.0 `binarize`, v2.0 `black_on_white`)

The illumination model assumes that background intensity varies slowly across the page — an assumption that holds for physical artifacts (shadow from curl, gradient from staining) but not for fine-grained ink. A large Gaussian blur (`sigma=80` default, tunable up to 120 for severe cases) applied to the full array produces a smooth estimate of the background luminance field. Each pixel is normalised by dividing by this estimate and rescaling to a reference level of 200:

```
corrected[i,j] = arr[i,j] / (background[i,j] + ε) × 200
```

The additive ε (1×10⁻⁶) prevents division by zero in fully black regions. This step removes broad illumination gradients but introduces a specific artifact: at ink boundaries, the Gaussian background estimate is pulled toward the ink value, causing the normalised output to darken at stroke edges. This artifact motivated the replacement of illumination correction with Sauvola thresholding in v2.1.

### 2.5 Stage Detail: Adaptive Thresholding — Versions 1.0 and 2.0

Following illumination correction, a percentile histogram stretch maps the 2nd–98th percentile of the corrected array to the 0–255 range, providing consistent input to the thresholding step regardless of residual tonal variation.

The threshold itself is a local mean comparison. For each pixel, the mean intensity within a square neighbourhood of side `adaptive_window` (default 60 px) is computed via `scipy.ndimage.uniform_filter`. A pixel is classified as ink (black) if it falls more than `binarize_threshold` (default 15) units below its local mean:

```
binary[i,j] = 0   if  local_mean[i,j] − arr[i,j]  >  threshold
binary[i,j] = 255 otherwise
```

This formulation captures pixels that are darker than their surroundings by a fixed margin, making it responsive to both dense printed regions and isolated handwritten strokes. The weakness is sensitivity to `adaptive_window` size: too small and large ink features are partially classified as background; too large and the method loses responsiveness to fine strokes in densely illuminated regions.

### 2.6 Stage Detail: Sauvola Adaptive Thresholding (v2.1 `black_on_white`)

Version 2.1 replaces the illumination-correction + uniform-mean pipeline with Sauvola thresholding, which incorporates local standard deviation into the threshold computation. The per-pixel threshold is:

```
T[i,j] = mean[i,j] × (1 + k × (std[i,j] / R − 1))
```

where `mean` and `std` are the local mean and standard deviation computed over a `bow_adaptive_window` × `bow_adaptive_window` neighbourhood, `k` is a sensitivity parameter (default 0.20), and `R` is a normalisation constant (default 128.0).

Local standard deviation is derived from the identity `Var(X) = E[X²] − E[X]²`, computed via two `uniform_filter` passes:

```python
lm  = uniform_filter(arr, size=bow_adaptive_window)
lm2 = uniform_filter(arr ** 2, size=bow_adaptive_window)
std = sqrt(max(lm2 − lm², 0))
```

The critical difference from the v2.0 approach is that the threshold adapts to local contrast, not only local mean. In high-contrast regions (dense print), the threshold is pushed higher, demanding a darker pixel to be classified as ink. In low-contrast regions (faded handwriting), the threshold drops, recovering faint strokes that the mean-only method would suppress. This eliminates the need for a preceding illumination-correction step and removes the blur artifacts it introduced at ink boundaries.

The `bow_sauvola_k` parameter controls the trade-off between ink recovery (lower values) and background cleanliness (higher values). The range 0.10–0.35 covers most document types without re-tuning other parameters.

### 2.7 Stage Detail: Stroke Thickening and Final Binarization

After denoising via `MedianFilter`, `scipy.ndimage.minimum_filter` applies a morphological erosion to the binary image in the ink domain. Because ink pixels are represented as 0 (black) and paper as 255 (white), a minimum filter expands the zero-valued regions, effectively thickening strokes. The kernel size `bow_stroke_size` (default 2) is intentionally small; the purpose is to restore stroke weight lost during processing rather than to artificially enlarge glyphs.

The final hard threshold after downscaling to the output resolution eliminates any anti-aliased grey values introduced by LANCZOS resampling. Pixels below `bow_final_threshold` become 0 (black); pixels at or above become 255 (white). The threshold was 180 in v2.0 (matching the illumination-corrected range) and lowered to 128 in v2.1 to match the Sauvola output distribution.

### 2.8 Configuration and Presets

The `Config` dataclass exposes every tunable parameter. Six presets cover the most common document types without manual tuning:

| Preset | Mode | Primary Use Case |
|--------|------|-----------------|
| A | `binarize` | Aged / yellowed historical documents |
| B | `binarize` | Modern office scans, minor shadow |
| C | `enhance` | Photographs, mixed-content pages |
| D | `black_on_white` | Batch folder processing |
| E | `black_on_white` | Standard — reference corpus parameters |
| F | `black_on_white` | Heavy damage: severe staining, faint ink |

Preset E reproduces the exact parameter values applied to the meteorological record. Preset F increases `bow_bg_sigma` to 120, reduces `bow_threshold` to 10, widens `bow_stroke_size` to 3, and lowers `bow_final_threshold` to 160 — changes that collectively favour ink recovery over background suppression.

### 2.9 Limitations of the Image-Processing Approach

The raster pipeline preserves the spatial layout of the original document unconditionally, including errors, misalignments, and physical artifacts. It does not interpret content. Where ink is genuinely absent — washed out by water damage, physically torn — no processing pipeline recovers it; the output faithfully reproduces the gap. For documents where content reconstruction is required rather than image legibility improvement, the raster approach is the wrong tool.

---

## 3. Era 2 — Vision API Transcription and PDF Typesetting (v3.0–v3.1)

### 3.1 Architectural Pivot

Version 3.0 replaced the image-processing pipeline entirely. The fundamental recognition was that the goal — producing a document that is readable and usable — does not require the output to be an improved version of the input image. If the content of the degraded image can be read and re-rendered, the output can be indistinguishable from a freshly printed document regardless of source quality.

The pipeline becomes: `image → Claude Vision API → structured transcription → ReportLab PDF typesetter → clean PDF`. The raster processing stages are removed. The dependencies reduce from seven packages to four (`anthropic`, `reportlab`, `Pillow`, `tqdm`).

The trade-off is explicit: the image-processing pipeline guarantees layout fidelity (spatial positions of text are preserved) but not content fidelity (illegible text is not recovered). The API pipeline guarantees content fidelity (every legible character is captured) but not layout fidelity (the output is a reflowed typeset document, not a facsimile). For the meteorological record use case, content fidelity is the priority.

### 3.2 OCR via Claude Vision API

The `ocr_document()` function encodes the input image as a base64 string and submits it to the Claude Vision API with a structured transcription prompt. TIFF and BMP formats are converted to JPEG before submission, as they are not natively supported by the API.

The prompt defines eight line-prefixed output types that together cover all structural elements found in tabular scientific records:

| Prefix | Semantic Role |
|--------|--------------|
| `HEADING:` | Primary document title |
| `SUBHEADING:` | Secondary title or subtitle |
| `SECTION:` | Named section divider |
| `FIELD:` | Label / value pair, pipe-separated |
| `TABLE_HEADER:` | Table column headers, pipe-separated |
| `TABLE_ROW:` | Table data row, pipe-separated |
| `TEXT:` | Free-form paragraph or sentence |
| `NOTE:` | Footnote, marginal annotation, or small-print remark |

The structured format is a deliberate constraint. Free-form transcription output cannot be reliably parsed into a typeset layout; prefixed lines can be processed deterministically by the PDF builder without heuristic parsing. Empty table cells are represented as a single space between pipes; illegible text is marked `[illegible]` in position. These conventions allow the typesetter to round-trip all structural information without ambiguity.

The `DOCUMENT_HINT` / `DOCUMENT_CONTEXT` parameter (optional) appends a one-sentence description of the document type to the prompt. This improves accuracy on specialist abbreviations, domain-specific notation, and unusual column structures where general-purpose OCR would interpret ambiguous characters differently.

### 3.3 PDF Typesetting via ReportLab

The `build_pdf()` function parses the structured transcription line by line and maps each prefix to a ReportLab flowable:

| Prefix | ReportLab Flowable | Style |
|--------|-------------------|-------|
| `HEADING` | `Paragraph` | Helvetica-Bold, centred, `font_size + 5` pt |
| `SUBHEADING` | `Paragraph` | Helvetica-Bold, centred, `font_size + 3` pt |
| `SECTION` | `Paragraph` + `HRFlowable` above and below | Helvetica-Bold, centred, `font_size + 1` pt |
| `FIELD` | `Paragraph` | Bold label + regular value, `font_size` pt |
| `TABLE_HEADER` / `TABLE_ROW` | Buffered → `Table` | Grid with 0.4 pt borders; header rows shaded `#D5D5D5` |
| `TEXT` | `Paragraph` | Helvetica regular, `font_size` pt |
| `NOTE` | `Paragraph` | Helvetica-Oblique, `font_size − 1` pt, `#444444` |

Table rows are buffered in a `tbl_rows` list and flushed to a `Table` flowable when a non-table line is encountered. Column widths are computed as equal subdivisions of the usable page width (`page_width − 2 × margins`). Tables use `repeatRows=1` to repeat header rows across page breaks.

The `_esc()` helper (introduced in v3.1) sanitises cell content for ReportLab's XML-based `Paragraph` renderer, escaping `&`, `<`, and `>` characters that would otherwise produce parse errors in cells containing meteorological notation or mathematical expressions.

### 3.4 Model Selection and Token Budget

Version 3.0 used `claude-opus-4-5` exclusively. Version 3.1 changed the default to `claude-sonnet-4-6` and exposed `model` and `max_tokens` as explicit parameters on `ocr_document()`. The rationale: for clean, well-structured historical records, the Sonnet model produces transcription quality equivalent to Opus at lower latency and cost. Opus remains available for documents with severe degradation, dense handwriting, or non-Latin script where the additional capacity is warranted.

The default `max_tokens` of 8192 is sufficient for single-page documents of the meteorological record type. Multi-page documents or records with large continuous tables may require values up to 16384; the parameter is documented accordingly.

### 3.5 Two-Page Spread Handling

Version 3.1 added an explicit transcription rule for bound volumes photographed open: the left page is transcribed completely from top to bottom before the right page. Without this rule, the model tends to interleave columns from both pages when the two pages share a common visual grid, producing a transcription that cannot be split cleanly by the typesetter. The rule is enforced in the prompt; no additional post-processing is required.

### 3.6 Limitations of the API Approach

Content that is physically absent from the image cannot be recovered. Where the image-processing pipeline represents a gap faithfully, the API pipeline represents it as `[illegible]` — a semantic label rather than a visual gap. Downstream consumers should treat `[illegible]` as a missing value requiring manual verification against the source scan.

Layout fidelity is not guaranteed. The typeset PDF reflects reading order and structural semantics, not the pixel-level spatial arrangement of the original. Documents where spatial position carries meaning (maps, diagrams, forms with precise field placement) are not suitable candidates for this pipeline without additional post-processing.

The pipeline requires an Anthropic API key and incurs per-token cost. Single-page documents of the meteorological record type consume approximately 1,500–2,500 input tokens and 1,000–2,000 output tokens per call, depending on image resolution and document density.

---

## 4. Corpus Reference

**British West African Meteorological Services — Monthly Record of Meteorological Observations**
Station: Lungi | Month: January | Year: 1955

The record contains five structural sections: instrument exposure details, instrument inventory with correction factors, a daily observation table (temperature, humidity, rainfall, wind, sunshine, cloud cover), a summary statistics section, and observer sign-off. It presents print and handwriting in the same document, tabular and free-form text in the same section, and domain-specific abbreviations throughout. It was used as the reference corpus for all pipeline development from the original `Meteorological Record Restoration.ipynb` through v3.1.

---

## 5. Dependency Summary

### Era 1 (v1.0–v2.1)

| Package | Role |
|---------|------|
| `Pillow` | Image load, save, PIL-native transforms |
| `numpy` | Array operations, thresholding logic |
| `scipy` | `gaussian_filter`, `uniform_filter`, `minimum_filter` |
| `opencv-python-headless` | Canny edge detection, Hough line transform (deskew) |
| `reportlab` | PDF output |
| `ipywidgets` | Interactive preview widget |
| `tqdm` | Batch processing progress bar |

### Era 2 (v3.0–v3.1)

| Package | Role |
|---------|------|
| `anthropic` | Claude Vision API client |
| `Pillow` | Image load, format conversion for API submission |
| `reportlab` | PDF typesetting |
| `tqdm` | Batch processing progress bar |

---

## 6. Version–Method Cross-Reference

| Version | Notebook | Thresholding Method | Architecture |
|---------|----------|--------------------|-|
| 1.0 | `Document Restoration Tool 1.0.ipynb` | Uniform-filter local mean | Image processing |
| 2.0 | `Document Restoration Tool 2.0.ipynb` | Gaussian background normalisation + uniform-filter local mean | Image processing |
| 2.1 | `Document Restoration Tool 2.1.ipynb` | Sauvola per-pixel adaptive (mean + std) | Image processing |
| 3.0 | `Document Restoration Tool 3.0.ipynb` | Claude Vision API OCR | API + typesetting |
| 3.1 | `Document Restoration Tool 3.1.ipynb` | Claude Vision API OCR | API + typesetting |
| — | `Meteorological Record Restoration.ipynb` | Gaussian background + uniform-filter (reference implementation) | Image processing |
