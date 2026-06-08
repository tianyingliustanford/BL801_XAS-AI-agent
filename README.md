# XAS AI Agent

**A conversational, tool-calling AI agent for soft X-ray absorption spectroscopy (XAS) data analysis.**

Ask in plain English — *"plot the TFY of scan 27284 normalized by I0"*, *"smooth it with
an 11-point Savitzky–Golay filter"*, *"normalize it and show E0"*, *"which scans are
Fe2O3?"* — and the agent picks the right analysis tool, runs it on the raw beamline data,
and returns the result with an inline plot. It turns a folder of synchrotron scans into
fast, reproducible analysis without writing code for each step.

## Demo
See **[docs/DEMO.md](docs/DEMO.md)** for a full walkthrough (plotting, smoothing, peak comparison) and the architecture.

## Features

- **Natural-language analysis:** an LLM tool-calling loop maps each request to a concrete
  analysis function and returns plots inline.
- **Robust file handling:** auto-detects the header row, resolves the energy / I0 / TEY /
  TFY columns by name, and finds a scan by its **unique number** even after files are
  renamed (e.g. `27284 pristine.txt`). Plots and reports are labeled with the full
  sample name.
- **Analysis toolkit:** plotting and multi-scan overlays, Savitzky–Golay smoothing,
  Athena-style pre-/post-edge normalization, 1st/2nd derivatives, peak & shoulder
  detection, element/edge identification, batch energy calibration, and data/image export.
- **Persistent experiment memory:** record notes about scans (*"27284 is the pristine
  reference"*) that survive across sessions and are searchable by keyword.
- **Provider-flexible:** works with any OpenAI-compatible LLM endpoint (CBORG / OpenAI /
  Gemini / Claude).

## Project structure

```
801_Agent/
├── run_agent.py              # One-click launcher — starts server + opens browser
├── chat_app.py               # Flask web app — chat UI + tool definitions + agent loop
├── xas_utils.py              # Core XAS utilities (parsing, normalization, smoothing, derivatives, peaks)
├── exp_info.py               # Persistent scan-metadata / long-term memory manager
├── XAS_Agent.ipynb           # Notebook launcher
├── XAS_Widget_Plotter.ipynb  # ipywidgets plotting notebook (see Notes)
├── .env.example              # Template for API key / config (copy to .env)
├── .gitignore
├── requirements.txt
├── 801_Data/                 # Scan data files (SigScan*.txt); a sample scan is included
├── exp_info/                 # Persistent experiment notes (exp_info.txt)
└── exported_data/            # Saved/exported analysis results
```

## Prerequisites

```bash
pip install -r requirements.txt
# flask, flask-cors, numpy, pandas, matplotlib, scipy, python-dotenv, openai
```

### LLM provider

Copy `.env.example` to `.env` and set **one** API key; the provider is auto-detected.

| Provider | Environment variable | Default model | Base URL |
|----------|---------------------|---------------|----------|
| CBORG (LBL) | `CBORG_API_KEY` | `claude-sonnet` | `https://api.cborg.lbl.gov/v1` |
| OpenAI | `OPENAI_API_KEY` | `gpt-4o` | `https://api.openai.com/v1` |
| Google Gemini | `GEMINI_API_KEY` | `gemini-2.0-flash` | `https://generativelanguage.googleapis.com/v1beta/openai` |
| Anthropic Claude | `ANTHROPIC_API_KEY` | `claude-sonnet-4-20250514` | `https://api.anthropic.com/v1` |

If several keys are set the order is CBORG → OpenAI → Gemini → Claude; override with
`LLM_PROVIDER`, `LLM_MODEL`, `LLM_BASE_URL`.


### Directories (optional `.env` overrides)

```bash
XAS_DATA_DIR=/path/to/your/scans     # default: 801_Data
XAS_EXPORT_DIR=/path/to/exports      # default: exported_data
EXP_INFO_DIR=/path/to/exp_info       # default: exp_info
```

## How to run

```bash
# One-click (recommended) — starts the server and opens the browser
python run_agent.py

# Or start the server directly
python chat_app.py        # then open http://localhost:5050

# Or from the notebook: open XAS_Agent.ipynb and run all cells
```

Refer to scans by number (e.g. *"plot scan 27284 TEY"*); the agent searches all
subdirectories automatically.

## Available tools

| Tool | Description |
|------|-------------|
| `list_scans` / `list_exports` | List scan files (with date filtering) / exported files |
| `plot_scan` / `compare_scans` | Plot one scan / overlay several (raw or signal/I0) |
| `plot_file` / `compare_files` | Plot / overlay generic two-column data files |
| `show_scan_info` | Scan metadata, columns, and saved user notes |
| `smooth_scan` | Savitzky–Golay smoothing (overlay raw vs smoothed) |
| `normalize_scan` | Athena-style pre-edge subtraction + post-edge normalization |
| `derivative_scan` | Savitzky–Golay 1st / 2nd derivative |
| `find_peaks_scan` | Peak & shoulder detection with tunable sensitivity |
| `identify_edge` | Identify element / absorption edge from peak energies |
| `calibrate_scans` | Batch energy-calibration shift |
| `rename_scan` | Copy a scan with a descriptive name (original unchanged) |
| `save_data` / `save_image` | Export last plot data (txt/csv/tsv/dat) / PNG |
| `update_exp_info` / `search_exp_info` | Add / search persistent scan notes |

### Highlights

- **Peak sensitivity:** `find_peaks_scan` supports `low` / `normal` / `high` / `very_high`.
- **Auto-scale & offset:** overlay spectra of very different intensity (`overlay` or `offset`).
- **Dual-axis & styling:** plot two signals on left/right axes; control colors, line styles,
  fonts, and labels via natural language.
- **Numbered options:** when the agent offers choices, just type the number to pick one.
- **Multi-format export:** `save_data` honors the file extension (`.txt`, `.csv`, `.tsv`, `.dat`).
- **Long-term memory:** notes are stored in `exp_info/exp_info.txt` (JSON, sorted by scan
  number, timestamped); changing an existing note triggers a confirm dialog.

## Data format

Tab-separated text files with a metadata header, then a column-header row, then data. The
header row is detected automatically (its first column starts with `Time`).

| Quantity | Column |
|----------|--------|
| Photon energy (eV) | `Energy UDP` |
| Incident flux I0 | `Izero` |
| Total Electron Yield (TEY) | `TEY` |
| Total Fluorescence Yield (TFY) | `Channeltron` |

The column resolver accepts a few alternate names per quantity, so scans saved under
slightly different headers still load without code changes.

## Analysis methods

- **Smoothing (Savitzky–Golay):** fits a low-order polynomial (default order 2) to a sliding
  window of points; window size and order are adjustable. Suppresses noise while preserving
  peak position and height.
- **Normalization (Athena-style):** E0 from the max of the smoothed 1st derivative; linear
  pre-edge fit; quadratic post-edge fit; edge step at E0; normalized μ(E). For
  publication-grade work, process in Athena; this is a fast in-browser look.
- **Derivatives:** Savitzky–Golay smoothed 1st/2nd derivative.
- **Peak detection:** `scipy.signal.find_peaks` plus 2nd-derivative shoulder detection, with a
  relative-prominence filter to drop spurious weak peaks.
- **Edge identification:** matches observed peak energies against a built-in edge database
  (±30 eV), prioritizing an element hint parsed from scan metadata.

## Architecture

```
User ↔ Browser (HTML/CSS/JS)
       ↕ AJAX POST /chat
     Flask server (chat_app.py)
       ↕ OpenAI-compatible API (tool calling)
     LLM
       ↕ tool calls
     xas_utils.py (data processing)   exp_info.py (persistent notes)
       ↕ file I/O                       ↕ file I/O
     801_Data/*.txt                    exp_info/exp_info.txt
```

The agent runs an iterative tool-calling loop: the LLM receives the message, decides which
tool(s) to call, the server executes them and returns results (including base64 plot images),
and the LLM writes the final response.

## Notes / known limitations

- `XAS_Widget_Plotter.ipynb` calls `xu.load_all_scans` / `xu.scan_summary`, which are not
  defined in `xas_utils.py`. The chat app and the `xas_utils` analysis functions are the
  maintained path.
- The in-browser `find_e0` (max |dμ/dE|) is a coarse estimate; for edges without a clean
  rising step it can land near a scan boundary. Use Athena for final E0/edge assignment.
