<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/WorkHaH/SheetWeave/main/assets/sheetweave-banner.svg">
    <img src="https://raw.githubusercontent.com/WorkHaH/SheetWeave/main/assets/sheetweave-banner.svg" alt="SheetWeave: vector PDF drawing stitching skill" width="100%">
  </picture>
</p>

<p align="center">
  <a href="README.zh-CN.md">简体中文</a> | <a href="README.md">English</a>
</p>

<p align="center">
  <a href="LICENSE"><img alt="License: MIT" src="https://img.shields.io/badge/license-MIT-1F7A6D.svg"></a>
  <img alt="Agent Skill" src="https://img.shields.io/badge/agent%20skill-SKILL.md-F1C376.svg">
  <img alt="Codex ready" src="https://img.shields.io/badge/Codex-ready-1F7A6D.svg">
  <img alt="Vector output" src="https://img.shields.io/badge/output-vector%20PDF-F1C376.svg">
</p>

<h1 align="center">SheetWeave</h1>

<p align="center">
  Weave tiled drawing sheets into one complete vector PDF.
</p>

---

SheetWeave is an agent skill. Give your agent a drawing PDF and tell it to use `$sheetweave`; the skill provides the workflow, scripts, and review checkpoints needed to recover the layout and produce a merged vector PDF.

<p align="center">
  <img src="assets/sheetweave-overview.svg" alt="SheetWeave high-level workflow: from tiled PDF input, through layout recovery, route selection, review, to vector PDF export" width="100%">
</p>

## 🧭 What Problem Does It Solve?

Many construction, architecture, and engineering PDFs are split into local/detail sheets. The hard part is not just stitching images together; it is recovering where each sheet belongs while keeping the final drawing sharp and vector-based.

| Your situation | What SheetWeave helps the agent do |
| --- | --- |
| The PDF includes an overview/index page | Check the likely overview pages first, usually page 1 or the last page. |
| Overview labels are machine-readable | Match detail pages by extracted sheet codes or numeric markers. |
| Labels are visible but code cannot extract them | Ask a vision model or human to read the labels and produce a layout JSON. |
| There is no usable overview page, or the overview has no labels | Use overlap-based neighbor matching between detail sheets. |
| Some sheets connect only through small overlaps | Use targeted bridge recovery instead of slow full high-DPI rendering. |
| The final result must stay vector | Place original PDF pages onto a larger LaTeX/TikZ canvas. |

> SheetWeave is a drawing-layout recovery skill, not a raster screenshot stitcher.

## 🟢 Easiest Install

Open your agent and say:

```text
Please install the SheetWeave skill from https://github.com/JoinferLI/SheetWeave-skill, then use it to merge my drawing PDF into one vector PDF.
```

Or in Chinese:

```text
请帮我安装 https://github.com/JoinferLI/SheetWeave-skill 这个 skill，然后用它把我的图纸 PDF 拼成一张完整的矢量 PDF。
```

## ⚙️ Manual Install

```bash
npx skills add JoinferLI/SheetWeave-skill
```

Or clone directly:

```bash
git clone https://github.com/JoinferLI/SheetWeave-skill.git ~/.agents/skills/sheetweave
```

Restart or reload your agent after installation.

## 💬 Ask Your Agent To Use It

```text
Use $sheetweave to merge this drawing PDF into one vector PDF: ./drawings.pdf
```

```text
用 $sheetweave 处理这个 PDF，把里面的局部图纸拼成一张完整的矢量 PDF。
```

## 🧵 How It Works — Decision Flow

<p align="center">
  <img src="assets/sheetweave-decision-flow.svg" alt="SheetWeave decision flow: overview detection → label routes → overlap fallback → bridge recovery → vector export" width="100%">
</p>

1. **Render previews** — Low-resolution page images and extracted text for fast layout recovery.
2. **Check for overview** — Usually page 1 or the last page. Machine-readable labels become an automatic page-to-region map.
3. **Choose a route**:
   - **Machine-readable labels** → auto label mapping
   - **Visible-only labels** → Manual or VLM layout JSON → rerun with `--overview-layout-json`
   - **No overview / no labels** → Overlap-based neighbor matching
4. **Build graph and solve placements** — Selected edges become page transforms.
5. **Single connected component?** → **Vector merged PDF** (original pages on TikZ canvas)
6. **Still split?** → **Bridge recovery** (targeted high-DPI checks) → re-solve
7. **Unresolved components** → **Groups and summary** (diagnostics preserved)

## 📦 Output Structure

```text
output/run/
  summary.json                 # mapping, edges, components, final paths
  final/
    full-merged.pdf            # vector result when one component is solved
    full-merged.tex            # generated LaTeX/TikZ source
    full-merged.png            # raster review preview
    layout-contact.png         # overview-guided contact sheet when available
  groups/group-XX/             # written when disconnected components remain
  vlm-request.json             # written when overview mapping needs help
```

## 🔍 Review The Result

Always inspect `summary.json` and the PNG preview before treating the vector PDF as final.

## 🛠️ Runtime Environment

| Requirement | Purpose |
| --- | --- |
| Python 3.10+ | Runs the bundled scripts. |
| `numpy`, `opencv-python`, `Pillow`, `pypdf` | Image matching and PDF manipulation. |
| `pdfinfo`, `pdftoppm`, `pdftotext` | PDF metadata, preview rendering, text extraction. |
| `pdflatex` | Final vector PDF assembly. |

```bash
pip install -r scripts/requirements.txt
```

## 👁️ Manual / VLM Overview Mapping

When labels are visible but code cannot extract them, read [`references/overview_layout_prompt.md`](references/overview_layout_prompt.md), ask a VLM or human to map labels to PDF pages, and rerun with `--overview-layout-json`. If the overview truly has no labels, use overlap matching instead.

## ✅ Quality Bar

- Single `final/full-merged.pdf` when the drawing set is connected
- Vector-based output (original PDF pages embedded)
- `summary.json` with auditable edges
- Review artifacts for human verification
- Graceful degradation into groups or VLM request on failure

## 🗂️ Repository Layout

```text
sheetweave/
  SKILL.md                         # agent-loaded skill entry
  README.md                        # English docs
  README.zh-CN.md                  # Chinese docs
  assets/
    sheetweave-overview.svg        # five-step workflow diagram
    sheetweave-decision-flow.svg   # detailed decision flowchart
  agents/openai.yaml               # Codex UI metadata
  scripts/
    sheetweave.py                  # main helper script
    merge_drawings.py              # overlap scoring
    merge_pdf_drawings.py          # PDF helpers
    vector_pdf_export.py           # vector assembly
    requirements.txt               # Python deps
  references/
    overview_layout_prompt.md      # VLM/human mapping prompt
```

## ⚠️ Current Limitations

- Very large canvases may hit LaTeX page-size limits.
- Synthetic bridge edges are geometric inferences; review `summary.json` and `full-merged.png` for critical work.
- The repository intentionally excludes real drawing PDFs and generated outputs.
