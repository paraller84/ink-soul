---
name: ocr-and-documents
description: "Extract text from PDFs/scans (pymupdf, marker-pdf)."
version: 2.3.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [PDF, Documents, Research, Arxiv, Text-Extraction, OCR]
    related_skills: [powerpoint]
---

# PDF & Document Extraction

For DOCX: use `python-docx` (parses actual document structure, far better than OCR).
For PPTX: see the `powerpoint` skill (uses `python-pptx` with full slide/notes support).
This skill covers **PDFs and scanned documents**.

## Step 1: Remote URL Available?

If the document has a URL, **always try `web_extract` first**:

```
web_extract(urls=["https://arxiv.org/pdf/2402.03300"])
web_extract(urls=["https://example.com/report.pdf"])
```

This handles PDF-to-markdown conversion via Firecrawl with no local dependencies.

Only use local extraction when: the file is local, web_extract fails, or you need batch processing.

## Step 2: Choose Local Extractor

| Feature | pymupdf (~25MB) | marker-pdf (~3-5GB) |
|---------|-----------------|---------------------|
| **Text-based PDF** | ✅ | ✅ |
| **Scanned PDF (OCR)** | ❌ | ✅ (90+ languages) |
| **Tables** | ✅ (basic) | ✅ (high accuracy) |
| **Equations / LaTeX** | ❌ | ✅ |
| **Code blocks** | ❌ | ✅ |
| **Forms** | ❌ | ✅ |
| **Headers/footers removal** | ❌ | ✅ |
| **Reading order detection** | ❌ | ✅ |
| **Images extraction** | ✅ (embedded) | ✅ (with context) |
| **Images → text (OCR)** | ❌ | ✅ |
| **EPUB** | ✅ | ✅ |
| **Markdown output** | ✅ (via pymupdf4llm) | ✅ (native, higher quality) |
| **Install size** | ~25MB | ~3-5GB (PyTorch + models) |
| **Speed** | Instant | ~1-14s/page (CPU), ~0.2s/page (GPU) |

**Decision**: Use pymupdf unless you need OCR, equations, forms, or complex layout analysis.

If the user needs marker capabilities but the system lacks ~5GB free disk:
> "This document needs OCR/advanced extraction (marker-pdf), which requires ~5GB for PyTorch and models. Your system has [X]GB free. Options: free up space, provide a URL so I can use web_extract, or I can try pymupdf which works for text-based PDFs but not scanned documents or equations."

---

## pymupdf (lightweight)

```bash
pip install pymupdf pymupdf4llm
```

**Via helper script**:
```bash
python scripts/extract_pymupdf.py document.pdf              # Plain text
python scripts/extract_pymupdf.py document.pdf --markdown    # Markdown
python scripts/extract_pymupdf.py document.pdf --tables      # Tables
python scripts/extract_pymupdf.py document.pdf --images out/ # Extract images
python scripts/extract_pymupdf.py document.pdf --metadata    # Title, author, pages
python scripts/extract_pymupdf.py document.pdf --pages 0-4   # Specific pages
```

**Inline**:
```bash
python3 -c "
import pymupdf
doc = pymupdf.open('document.pdf')
for page in doc:
    print(page.get_text())
"
```

---

## marker-pdf (high-quality OCR)

```bash
# Check disk space first
python scripts/extract_marker.py --check

pip install marker-pdf
```

**Via helper script**:
```bash
python scripts/extract_marker.py document.pdf                # Markdown
python scripts/extract_marker.py document.pdf --json         # JSON with metadata
python scripts/extract_marker.py document.pdf --output_dir out/  # Save images
python scripts/extract_marker.py scanned.pdf                 # Scanned PDF (OCR)
python scripts/extract_marker.py document.pdf --use_llm      # LLM-boosted accuracy
```

**CLI** (installed with marker-pdf):
```bash
marker_single document.pdf --output_dir ./output
marker /path/to/folder --workers 4    # Batch
```

---

## Arxiv Papers

```
# Abstract only (fast)
web_extract(urls=["https://arxiv.org/abs/2402.03300"])

# Full paper
web_extract(urls=["https://arxiv.org/pdf/2402.03300"])

# Search
web_search(query="arxiv GRPO reinforcement learning 2026")
```

## Split, Merge & Search

pymupdf handles these natively — use `execute_code` or inline Python:

```python
# Split: extract pages 1-5 to a new PDF
import pymupdf
doc = pymupdf.open("report.pdf")
new = pymupdf.open()
for i in range(5):
    new.insert_pdf(doc, from_page=i, to_page=i)
new.save("pages_1-5.pdf")
```

```python
# Merge multiple PDFs
import pymupdf
result = pymupdf.open()
for path in ["a.pdf", "b.pdf", "c.pdf"]:
    result.insert_pdf(pymupdf.open(path))
result.save("merged.pdf")
```

```python
# Search for text across all pages
import pymupdf
doc = pymupdf.open("report.pdf")
for i, page in enumerate(doc):
    results = page.search_for("revenue")
    if results:
        print(f"Page {i+1}: {len(results)} match(es)")
        print(page.get_text("text"))
```

No extra dependencies needed — pymupdf covers split, merge, search, and text extraction in one package.

---

## Local Vision Model Analysis (Fallback)

When `browser_vision` tool fails with "Connection error" (common in WSL/local setups), fall back to direct Ollama API calls for image analysis. This also works for any image analysis task beyond OCR — health reports, screenshots, diagrams, etc.

### API Format (Verified Working)

Only **qwen3-vl:8b** reliably supports vision via Ollama API. qwen3.5-256k:latest claims vision capability but does NOT accept the `images` field at the message level.

**Correct payload format** — `images` field **inside** the user message object:

```python
import base64, json, urllib.request

with open('image.jpg', 'rb') as f:
    img_b64 = base64.b64encode(f.read()).decode('utf-8')

payload = {
    'model': 'qwen3-vl:8b',
    'messages': [{
        'role': 'user',
        'content': '请详细描述这张图片的所有内容',  # Or your analysis prompt
        'images': [img_b64]
    }],
    'stream': False,
    'options': {'num_predict': 2000}
}

req = urllib.request.Request(
    'http://localhost:11434/api/chat',
    data=json.dumps(payload).encode('utf-8'),
    headers={'Content-Type': 'application/json'}
)
resp = urllib.request.urlopen(req, timeout=180)
result = json.loads(resp.read().decode('utf-8'))
print(result['message']['content'])
```

### Common Failure Modes

| Error | Likely Cause | Fix |
|-------|-------------|-----|
| `images` field at top level of payload | Wrong format for qwen3.5-256k | Move `images` inside the message object (works with qwen3-vl:8b only) |
| `json: cannot unmarshal array into Go struct field` | Content sent as array instead of string | Use string content + separate `images` field, not `content: [type, text]` format |
| Model claims "I cannot see images" | Model doesn't support `images` field | Switch to qwen3-vl:8b |
| Connection refused | Ollama not running or wrong port | Check `curl localhost:11434/api/tags` |
| HTTP 400 | Malformed payload | Verify JSON structure matches exactly |

### Model Compatibility

| Model | Vision via `images` field | Notes |
|-------|--------------------------|-------|
| **qwen3-vl:8b** | ✅ | Primary vision model. Use this. |
| qwen3.5-256k:latest | ❌ | Claims vision in metadata, does NOT accept `images` field in Ollama API |
| qwen3-vl:4b | ⚠️ | Older, smaller. Replaced by 8B version. |

### Environment Note

Ollama runs on **Windows side** (not inside WSL), so the `ollama` CLI is not available in WSL PATH. All interactions must go through the HTTP API at `localhost:11434`.

### When to Use This vs OCR

- **Local Vision Model** → Best for: extracting structured data from complex document images (reports, forms with tables), understanding visual layout, reading handwritten text in context, analyzing charts/diagrams
- **marker-pdf** → Best for: batch OCR of scanned PDFs with 90+ language support, equations, forms
- **pymupdf** → Best for: text-based PDFs (digital-born, not scanned), fast extraction with no model dependencies

---

## Notes

- `web_extract` is always first choice for URLs
- pymupdf is the safe default — instant, no models, works everywhere
- marker-pdf is for OCR, scanned docs, equations, complex layouts — install only when needed
- Both helper scripts accept `--help` for full usage
- marker-pdf downloads ~2.5GB of models to `~/.cache/huggingface/` on first use
- For Word docs: `pip install python-docx` (better than OCR — parses actual structure)
- For PowerPoint: see the `powerpoint` skill (uses python-pptx)
- For local vision model analysis (fallback when `browser_vision` fails): see `references/local-vision-api.md` for working code and troubleshooting guide
