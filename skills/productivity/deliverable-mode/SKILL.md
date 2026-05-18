---
name: deliverable-mode
description: When the user is in a messaging surface (Slack, Discord, Telegram, etc.), produce native artifacts (charts, PDFs, spreadsheets, images) and reference their absolute path in the response so the gateway delivers them inline. Bias toward artifacts over walls of text for comparative, numeric, or long-form content.
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [deliverable, slack, gateway, charts, artifacts, messaging]
    category: messaging
triggers:
  - user is on slack / discord / telegram / signal / whatsapp and asks for a comparison
  - request involves numeric data, tabular data, or research output
  - request would otherwise produce a long wall of text
  - user explicitly asks for a "report", "chart", "deck", "spreadsheet", "pdf"
---

# Deliverable Mode

When the user reaches you through a messaging surface, the gateway automatically
delivers any absolute file path you mention in your response as a native file
upload. Use this. Prefer producing artifacts over prose for the cases below,
then reference the artifact path in plain text. The gateway strips the path
from the message text and uploads the file natively (image embed for
`.png`/`.jpg`, file attachment for `.pdf`/`.csv`/`.xlsx`/etc.).

## When to reach for an artifact

| User asks for | Produce | Tool to use |
|---|---|---|
| Comparison of N options across M criteria | Chart (PNG) and/or table | `execute_code` with `matplotlib` |
| Numeric trends over time | Line/bar chart (PNG) | `execute_code` with `matplotlib` |
| Long-form research summary | PDF report | `latex-pdf-report` skill + tectonic |
| Tabular data the user will reuse | CSV or XLSX | `execute_code` with `pandas` or stdlib `csv` |
| Slide deck or presentation | PPTX | `powerpoint` skill |
| Diagram / architecture sketch | SVG or PNG | `excalidraw` or `architecture-diagram` skills, or `image_generate` |
| Spatial / visual concept | Generated image | `image_generate` tool |
| Voice-message style reply | Audio (MP3/OGG) | `text_to_speech` tool |

## When to stay text-first

- The question is conversational ("hey, what time is it?", "thanks", "got it")
- The user explicitly asks "just summarize", "give me a quick answer", "TL;DR"
- The output is a single number, fact, or short answer
- The user is on the CLI / TUI (not a messaging gateway) — they read text inline anyway
- You're mid-conversation and the user is steering with rapid follow-ups
- The artifact would be smaller or less useful than the prose summary

## How to ship an artifact

1. Generate the file in `/tmp/` or another writable location with a descriptive name. Avoid generic names like `chart.png` — use `q3-revenue-comparison.png`.
2. After the file exists on disk, write the absolute path into your response as plain text. Examples:
   - `Here's the comparison: /tmp/q3-revenue-comparison.png`
   - `Full report attached: /tmp/competitor-analysis.pdf`
   - `Raw data: /tmp/sales-by-region.csv`
3. Do NOT wrap the path in inline code (backticks) or fenced code blocks — those are excluded from extraction so code samples aren't mutilated. Bare text is the trigger.
4. Do NOT use a `MEDIA:` tag for these files. Bare paths are the simpler primitive; `MEDIA:` is reserved for tool-emitted media (image_generate, video_generate, TTS).

## Supported extensions

Images embed inline on most platforms: `.png .jpg .jpeg .gif .webp .bmp .tiff .svg`

Video embeds inline where supported: `.mp4 .mov .avi .mkv .webm`

Audio routes to voice/audio attachments: `.mp3 .wav .ogg .m4a .flac`

Documents upload as file attachments: `.pdf .docx .doc .odt .rtf .txt .md`

Spreadsheets and data: `.xlsx .xls .csv .tsv .json .xml .yaml .yml`

Presentations: `.pptx .ppt .odp`

Archives: `.zip .tar .gz .tgz .bz2 .7z`

Web: `.html .htm`

## Examples

### Comparison → chart

User asks: "compare Q1, Q2, Q3 revenue for our top 3 products"

Wrong (text only):
```
Q1: ProductA $1.2M, ProductB $800K, ProductC $400K
Q2: ProductA $1.5M, ProductB $1.1M, ProductC $600K
Q3: ProductA $1.8M, ProductB $1.4M, ProductC $900K
```

Right (artifact + brief prose):

Generate via `execute_code`:
```python
import matplotlib.pyplot as plt
products = ['ProductA', 'ProductB', 'ProductC']
q1 = [1.2, 0.8, 0.4]
q2 = [1.5, 1.1, 0.6]
q3 = [1.8, 1.4, 0.9]
# ... bar chart code ...
plt.savefig('/tmp/q3-revenue-comparison.png', dpi=150, bbox_inches='tight')
```

Then in your response:
```
Quarterly revenue is climbing across all three products, with ProductC
showing the strongest growth (125% Q1→Q3). Full chart:
/tmp/q3-revenue-comparison.png
```

### Research → PDF

User asks: "research the top 5 vector databases and give me a report"

Use the `latex-pdf-report` skill to build the PDF with tectonic. Save it to a
descriptive path, then reference that path in your response.

### Data dump → CSV

User asks: "give me a list of all our customers by region"

Generate the data, write to `/tmp/customers-by-region.csv`, reference the path
in your reply. Don't paste 500 rows of CSV into the message.

## Quality bar

- **Charts have axis labels and a title.** Bare line plots are not a deliverable.
- **PDFs have a heading.** Even one section header beats a wall of paragraphs.
- **Spreadsheets have column headers.** Always.
- **File names describe content.** `comparison.png` is bad; `q3-revenue-by-product.png` is good.
- **Don't ship debugging output.** If matplotlib emitted a warning, the file still rendered — but verify visually before announcing it.

## When the platform is the CLI

Skip all of this. CLI users want text in their terminal. They can ask for an
artifact explicitly if they want one.

## When the user asks for "just text"

Respect it. This skill biases toward artifacts as a default, not as a rule.
The user's explicit preference always wins.
