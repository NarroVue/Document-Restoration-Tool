# 📄 Universal Document Restoration Tool

**Turns any scanned, aged, stained or damaged document image into a clean, typeset PDF — exactly as if it had been retyped from scratch on fresh paper.**

Perfect for: historical archives, handwritten logs, degraded scans, legal documents, medical records, weather notebooks, census forms — anything with text that needs to become a clean digital document.

## How it works

1. Point the notebook at any image file (JPEG, PNG, TIFF, BMP, WebP)
2. Claude Vision reads every word, number, table, and handwritten note
3. ReportLab typesets the result into a brand‑new clean PDF

## Requirements

- Python 3.9+
- An [Anthropic API key](https://console.anthropic.com) (Claude Vision)
- A few GB of disk space (the notebook itself is tiny)

## Quick Start

### 1. Clone the repository

```bash
git clone https://github.com/yourusername/document-restoration-tool.git
cd document-restoration-tool

!pip install anthropic reportlab Pillow tqdm

INPUT_IMAGE  = "path/to/your/scanned_document.jpg"
OUTPUT_PDF   = "path/to/clean_output.pdf"
API_KEY      = "sk-ant-..."          # your Anthropic key

PAGE_SIZE    = A4                    # A4 | letter | landscape(A4)
FONT_SIZE    = 8                     # body text in points
MARGINS_MM   = 12                    # page margins in mm
MODEL        = 'claude-sonnet-4-6'   # or 'claude-opus-4-5' for higher accuracy
DOCUMENT_HINT = ''                   # e.g. 'Meteorological logbook, 1950s'

DOCUMENT_HINT = 'Meteorological observation logbook, 1950s, British West Africa.'

from reportlab.lib.pagesizes import landscape, A4, A3
PAGE_SIZE = landscape(A4)   # wide page for tables with many columns
PAGE_SIZE = A3              # larger paper

MODEL = 'claude-opus-4-5'

restore_folder(
    input_folder='scans/',
    output_folder='restored/',
    hint='Legal contracts, 1990s',
    save_txt=True
)
