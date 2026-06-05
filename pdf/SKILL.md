---
name: pdf
description: Read and extract text from PDF files on the device using the bundled Python runtime + pypdfium2. Use this when the user shares a PDF or asks to read, summarize, search, or extract text from a PDF — works fully offline, no upload.
---

The device has a real CPython 3.14 runtime: the `python` command is already in PATH (no setup, no internet needed for parsing). The `pypdfium2` library is bundled — it wraps the PDFium engine via ctypes and reads PDFs natively.

## Extract all text (most common)

```sh
python -c '
import sys, pypdfium2 as pdfium
pdf = pdfium.PdfDocument(sys.argv[1])
n = len(pdf)
for i in range(n):
    tp = pdf[i].get_textpage()
    print(f"===== page {i+1}/{n} =====")
    print(tp.get_text_range())
' "/absolute/path/to/file.pdf"
```

## Page count / quick metadata

```sh
python -c '
import sys, pypdfium2 as pdfium
d = pdfium.PdfDocument(sys.argv[1])
print("pages:", len(d))
' "/absolute/path/to/file.pdf"
```

## Extract text from a specific page (e.g. page 3)

```sh
python -c '
import sys, pypdfium2 as pdfium
pdf = pdfium.PdfDocument(sys.argv[1])
print(pdf[2].get_textpage().get_text_range())
' "/absolute/path/to/file.pdf"
```

## Notes

- Always pass an **absolute path** to the PDF.
- For large PDFs, process page-by-page (loop above) instead of loading everything into one string.
- Searching: extract text first, then grep/filter in Python or shell.
- **Image rendering to PNG** needs `Pillow` (not yet bundled) — text extraction here does NOT need it and works standalone.
- If `python` reports «рантайм ещё разворачивается» — the Python runtime is still unpacking on first launch; wait a few seconds and retry.
