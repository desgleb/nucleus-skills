---
name: pdf-reading
description: "Use this skill when you need to read, inspect, or extract content from PDF files — especially when file content is NOT in your context and you need to read it from disk. Covers content inventory, text extraction, page rasterization for visual inspection, embedded image/attachment/table/form-field extraction, and choosing the right reading strategy for different document types (text-heavy, scanned, slide-decks, forms, data-heavy). Do NOT use this skill for PDF creation, form filling, merging, splitting, watermarking, or encryption — use the pdf skill instead."
license: Proprietary. LICENSE.txt has complete terms
---

# PDF Processing Guide

## Overview

This guide covers PDF reading operations using Python libraries available on-device (`python`). For advanced features (pypdfium2 rendering details, pdfplumber table settings, encrypted PDF handling), see REFERENCE.md.

## Reading & Inspecting PDFs

Before doing anything with a PDF, understand what you're working with.

### Content inventory

```python
from pypdf import PdfReader

reader = PdfReader("document.pdf")
meta = reader.metadata
print(f"Pages: {len(reader.pages)}")
print(f"Title: {meta.title}, Author: {meta.author}")
print(f"Encrypted: {reader.is_encrypted}")
```

Quick text check — is this a text PDF or a scan?

```python
import pypdfium2 as pdfium

doc = pdfium.PdfDocument("document.pdf")
page = doc[0]
tp = page.get_textpage()
sample = tp.get_text_range(index=0, count=200)
print(repr(sample))  # empty or garbled → likely scanned
```

This tells you:
- **Page count and metadata** — how big is the job?
- **Text extractability** — does pypdfium2 return real text, or is it empty (scanned) or garbled (broken font encoding)?
- **Encryption status** — whether pypdf can decrypt with a password

### Text extraction

**pypdfium2** (primary — fast, accurate):
```python
import pypdfium2 as pdfium

doc = pdfium.PdfDocument("document.pdf")
for i, page in enumerate(doc):
    tp = page.get_textpage()
    text = tp.get_text_range()
    print(f"--- Page {i+1} ---")
    print(text)
```

Or use the ready script:
```bash
python ~/.claude/skills/pdf/scripts/extract_text.py document.pdf
```

**pdfplumber** for layout-aware extraction (multi-column, positioned text):
```python
import pdfplumber

with pdfplumber.open("document.pdf") as pdf:
    for page in pdf.pages:
        print(page.extract_text())
```

Note: pdfplumber fails on encrypted PDFs — decrypt with pypdf first, or rasterize and read visually.

### Visual inspection (rasterize pages)

Text extraction is **blind** to charts, diagrams, figures, equations, multi-column layout, and form structures. When any of these matter, rasterize the relevant page and Read the image:

```python
import pypdfium2 as pdfium

doc = pdfium.PdfDocument("document.pdf")
page = doc[2]  # page 3 (0-based)
bitmap = page.render(scale=150/72)  # 150 DPI
img = bitmap.to_pil()
img.save("/tmp/page_3.png")
# Then: Read /tmp/page_3.png with vision
```

Or use the ready script for all pages:
```bash
python ~/.claude/skills/pdf/scripts/convert_pdf_to_images.py document.pdf /tmp/pages/
# → /tmp/pages/page_1.png, page_2.png, ...
```

Then Read the resulting image file. This gives you full visual understanding of that page — layout, charts, equations, everything.

**When to rasterize vs. text-extract:**
- **Content/data questions → text extraction** (cheaper, searchable)
- **Figures, charts, visual layout → rasterize the page**
- **Tables → try pdfplumber first, rasterize if garbled**
- **Precision matters → do both** (extract text AND rasterize; use text for data, image for context)

**Token cost awareness:**
- Text extraction: ~200–400 tokens per page
- Rasterized image: ~1,600 tokens per page (at 150 DPI)
- Both together: ~2,000–2,400 tokens per page

For a 100-page PDF, rasterizing everything would consume ~160K tokens.
Only rasterize pages that matter for the question at hand.

### Choosing your reading strategy

**Text-heavy documents** (reports, articles, books):
→ Text extraction is primary. Rasterize only for specific figures or pages where layout matters.

**Scanned documents** (no extractable text):
→ Rasterize pages with `convert_pdf_to_images.py` and Read them visually. No OCR engine available on-device — vision is the fallback for scans.

**Slide-deck PDFs** (exported presentations):
→ Every page is primarily visual. Rasterize individual pages on demand. Text extraction gives you bullet-point text but loses all layout.

**Form-heavy documents**:
→ Extract form field values programmatically first (see below). Rasterize the form page for visual context if needed.

**Data-heavy documents** (tables, charts, figures):
→ Use pdfplumber for tables. Rasterize pages with charts/figures. Extract text for surrounding narrative.

### Extracting embedded images

```python
from pypdf import PdfReader

reader = PdfReader("document.pdf")
for i, page in enumerate(reader.pages):
    for j, img_obj in enumerate(page.images):
        with open(f"/tmp/img_p{i+1}_{j}.png", "wb") as f:
            f.write(img_obj.data)
```

**Gotcha — vector graphics:** `page.images` extracts only raster image objects. Charts and diagrams drawn as vector graphics (common in matplotlib, Excel, and R exports) will NOT appear. For these, rasterize the whole page instead.

**Gotcha — empty images:** Sometimes produces tiny background masks or decorative elements. Filter by file size to find real content images.

### Extracting file attachments

PDFs can contain embedded files — spreadsheets, data files, other documents. Common in business reports and PDF/A-3 compliance documents.

```python
import os
from pypdf import PdfReader

reader = PdfReader("document.pdf")
for name, content_list in reader.attachments.items():
    safe_name = os.path.basename(name)  # sanitize
    for content in content_list:
        with open(f"/tmp/{safe_name}", "wb") as f:
            f.write(content)
```

### Extracting form field data

```python
from pypdf import PdfReader

reader = PdfReader("form.pdf")

# Text input fields only:
fields = reader.get_form_text_fields()
for name, value in fields.items():
    print(f"{name}: {value}")

# All field types (checkboxes, radio buttons, dropdowns too):
all_fields = reader.get_fields() or {}
for name, field in all_fields.items():
    print(f"{name}: {field.get('/V', '')} (type: {field.get('/FT', '')})")
```

`get_form_text_fields()` returns only text input fields. For forms with checkboxes, radio buttons, and dropdowns, use `get_fields()` to see all field types.

For anything beyond reading form data — filling forms, creating forms — use the pdf skill at `~/.claude/skills/pdf/SKILL.md`.

### Font diagnostics / garbled text

If text extraction produces garbled output (wrong characters, missing text, mojibake), the PDF likely has broken font encoding or non-embedded fonts. No CLI font diagnostic tool is available on-device.

→ Rasterize the page with `convert_pdf_to_images.py` and read visually — this bypasses font encoding entirely.

---

## Quick Reference

```
┌──────────────────────────┬────────────────┬─────────────────────────────────────────────────┐
│ Task                     │ Tool           │ Command/Code                                    │
├──────────────────────────┼────────────────┼─────────────────────────────────────────────────┤
│ Page count + metadata    │ pypdf          │ PdfReader; len(reader.pages); reader.metadata   │
│ Extract text             │ pypdfium2      │ page.get_textpage().get_text_range()            │
│ Extract text (script)    │ extract_text.py│ python .../scripts/extract_text.py doc.pdf      │
│ Extract text (layout)    │ pdfplumber     │ page.extract_text()                             │
│ Extract tables           │ pdfplumber     │ page.extract_tables()                           │
│ See page visually        │ pypdfium2+PIL  │ page.render(scale=150/72).to_pil() → .png       │
│ Rasterize all pages      │ script         │ python .../scripts/convert_pdf_to_images.py     │
│ Extract raster images    │ pypdf          │ page.images[i].data                             │
│ Extract attachments      │ pypdf          │ reader.attachments                              │
│ Read form fields         │ pypdf          │ reader.get_fields()                             │
│ Scanned PDF / OCR        │ pypdfium2+vision│ rasterize → Read image (no tesseract on device)│
└──────────────────────────┴────────────────┴─────────────────────────────────────────────────┘
```

## PDF Form Filling, Creation, Merging, Splitting, and Other Operations

This skill covers **reading and inspection** only. For filling forms, creating, merging, splitting, rotating, watermarking, encrypting, or other PDF manipulation tasks, use the pdf skill at `~/.claude/skills/pdf/SKILL.md`.
