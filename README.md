# MusicSheetConvert

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)
[![Python 3.13+](https://img.shields.io/badge/python-3.12+-blue.svg)](https://www.python.org/downloads/)
[![DOI](https://img.shields.io/badge/DOI-zenodo.20604622-blue.svg)](https://doi.org/10.5281/zenodo.20604622)

> A Python toolkit for converting between **staff notation**, **numbered notation
> (jianpu)**, **MIDI**, and **MusicXML**, with built-in optical music recognition
> (Audiveris) and high-fidelity rendering (MuseScore 4).

---

## Table of Contents

- [Features](#features)
- [Demo](#demo)
- [Architecture](#architecture)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [CLI Reference](#cli-reference)
- [Jianpu Text Grammar](#jianpu-text-grammar)
- [Testing](#testing)
- [Known Limitations](#known-limitations)
- [Contributing](#contributing)
- [License](#license)

---

## Features

- **End-to-end OCR pipeline** — drop in a sheet image or PDF, get MIDI, jianpu
  text, and a re-rendered PNG in one command.
- **Bidirectional conversion** between MusicXML, MIDI, and jianpu (numbered)
  text via `music21` and `pretty_midi`.
- **Complex-score builders** for grand staff, multiple key signatures
  (C / G / D / F / Bb / Eb / Am / Em / Dm), ii-V-I 7th-chord progressions, and
  the first four measures of Bach's Prelude in C major (BWV 846).
- **High-fidelity staff rendering** via MuseScore 4 (PNG, PDF, SVG).
- **Pure-Python core** — no daemon, no web server, just a CLI and a library.

---

## Demo

```
$ python -m src.main --demo bach_prelude --render --out data/demo_bach

MusicXML : data\demo_bach\bach_prelude_m1_m4.musicxml
MIDI     : data\demo_bach\bach_prelude_m1_m4.mid
jianpu   : data\demo_bach\jianpu\bach_prelude_m1_m4.txt
PNG      : data\demo_bach\rendered\bach_prelude_m1_m4.png
```

The generated Bach prelude is recognized correctly by Audiveris 5.10.2 when
fed back through the OCR pipeline:

```
$ python -m src.main data/ocr_e2e/preprocessed/bach_prelude-1.png --render

MusicXML : data\ocr_e2e\musicxml\bach_prelude-1.mxl
MIDI     : data\ocr_e2e\midi\bach_prelude-1.mid
jianpu   : 3+ 5+ 7+ 3+ 1+ 2+ 1+ 6+ 2+ 5+ 7 2+ 3+ 5+ 1++ 3+ 5---
PNG      : data\ocr_e2e\rendered\bach_prelude-1.png
```

---

## Architecture

```
                +------------------+
                |  PDF / PNG image |
                +---------+--------+
                          |
                  preprocess (OpenCV)
                          |
                          v
                  +-------+--------+
                  |   binarized   |
                  |    PNG        |
                  +-------+--------+
                          |
                  Audiveris OMR (batch)
                          |
                          v
                  +-------+--------+
                  |  MusicXML/.mxl|
                  +-------+--------+
                          |
        +-----------------+------------------+------------------+
        |                 |                  |                  |
   music21           core (jianpu)      MuseScore 4           ...
        |                 |                  |
        v                 v                  v
   MIDI / .mid      jianpu / .txt     staff PNG / PDF
```

| Module | Responsibility |
|--------|----------------|
| `src/main.py` | CLI + pipeline orchestration (no conversion logic). |
| `src/music_convert/core.py` | Pure-Python MusicXML <-> MIDI <-> jianpu. |
| `src/music_convert/advanced.py` | Grand-staff, key-signature, chord, Bach-prelude builders. |
| `src/ocr/__init__.py` | Audiveris 5.x adapter with auto-discovery. |
| `src/preprocess/__init__.py` | Image / PDF preprocessing (grayscale, adaptive threshold, scaling). |
| `src/render/__init__.py` | MuseScore 4 rendering (PNG, PDF, SVG, MusicXML). |

See [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) for more detail.

---

## Installation

### 1. Clone the repository

```bash
git clone https://github.com/Unk1ndledAC/MusicSheetConvert.git
cd MusicSheetConvert
```

### 2. Create a Python venv and install dependencies

```bash
# Modify the directories in these two files to your actual Python and virtual environment paths first.
# Windows
scripts\install.bat

# Unix / Git Bash
./scripts/install.sh
```

### 3. Install external tools

| Tool | Why | Where to get it |
|------|-----|-----------------|
| **Audiveris 5.x** | Optical music recognition | <https://github.com/Audiveris/audiveris/releases> |
| **MuseScore 4** | Staff-notation rendering (PNG / PDF / SVG) | <https://musescore.org/download> |
| **Poppler** | PDF -> image (used by `pdf2image`) | <https://github.com/oschwartz10612/poppler-windows/releases> |

Set the `AUDIVERIS_EXE` and `MUSESCORE_EXE` environment variables to point at
non-default locations. The `find_audiveris()` and `find_musescore()` helpers
fall back to PATH lookup, then to this machine's default install paths.

---

## Quick Start

### Run the full pipeline on a sheet image

```bash
python -m src.main path/to/score.png --out data/output --render --bpm 100
```

This produces, under `data/output/`:

```
data/output/
├── preprocessed/   # binarized copies of the inputs
├── musicxml/       # .mxl outputs from Audiveris
├── midi/           # .mid files
├── jianpu/         # jianpu text files
└── rendered/       # staff-notation PNGs (only when --render is passed)
```

### Run the full pipeline on a PDF

```bash
python -m src.main path/to/scorebook.pdf --out data/output --render --bpm 90
```

Each page becomes a separate result with the suffix `_p<NNN>`.

### Skip OCR when the input is already MusicXML

```bash
python -m src.main path/to/score.musicxml --out data/output --skip-ocr --render
```

### Generate a complex score from a preset (no input file)

```bash
# Two-hand grand staff in any of the supported keys
python -m src.main --demo grand_staff --key Bb --render --out data/demo_bb

# ii-V-I 7th-chord progression
python -m src.main --demo chord_progression --key C --render --out data/demo_c

# All 9 supported key signatures in one piece
python -m src.main --demo key_demonstration --render --out data/demo_keys

# First 4 measures of Bach's Prelude in C major
python -m src.main --demo bach_prelude --render --out data/demo_bach
```

### Use as a Python library

```python
from pathlib import Path
from src.music_convert import core
from src.music_convert.advanced import build_grand_staff, NoteSpec
from src.ocr import image_to_musicxml
from src.render import musicxml_to_png

# 1) MusicXML -> MIDI
core.musicxml_to_midi("in.musicxml", "out.mid", core.ConvertOptions(bpm=100))

# 2) MusicXML -> jianpu text
print(core.musicxml_to_jianpu_text("in.musicxml"))

# 3) Build a complex score from scratch
score = build_grand_staff(
    right_hand_measures=[[NoteSpec(["C4"], 1.0), NoteSpec(["D4"], 1.0),
                         NoteSpec(["E4"], 1.0), NoteSpec(["F4"], 1.0)]],
    left_hand_measures=[[NoteSpec(["C3", "E3", "G3"], 4.0)]],
    key_name="C", title="Demo"
)

# 4) OMR (image -> MusicXML)
xml_path = image_to_musicxml("score.png", Path("data/output/musicxml"))

# 5) Render MusicXML to PNG via MuseScore
musicxml_to_png(xml_path, Path("data/output/rendered/score.png"))
```

---

## CLI Reference

```
python -m src.main [INPUT] [OPTIONS]
```

| Option | Description |
|--------|-------------|
| `INPUT` | Path to a PDF / PNG / JPG / MusicXML file. Omit when using `--demo`. |
| `--out DIR` | Output root directory (default `./data/output`). |
| `--bpm N` | Default MIDI tempo (default 90). |
| `--render` | Additionally render staff-notation PNG via MuseScore. |
| `--skip-ocr` | Skip Audiveris OCR (use when the input is already MusicXML). |
| `--audiveris-bin PATH` | Explicit path to the Audiveris executable. |
| `--demo {grand_staff,chord_progression,key_demonstration,bach_prelude}` | Generate a preset complex score instead of reading a file. |
| `--key {C,G,D,F,Bb,Eb,Am,Em,Dm}` | Key signature for the generated score. |
| `--title TEXT` | Title for the generated score. |

---

## Jianpu Text Grammar

The numbered-notation (`jianpu`) text format used by this project is
intentionally minimal:

| Token | Meaning |
|-------|---------|
| `1` `2` `3` `4` `5` `6` `7` | do, re, mi, fa, sol, la, ti (middle C = 1) |
| `1+` `1++` | One / two octaves up |
| `1-` | One octave down |
| `#1` `b3` | Sharp / flat |
| `/` | Chord separator (e.g. `1/3/5`) |
| `\|` | Bar separator (kept for readability, otherwise ignored) |
| whitespace | Token separator |

Each token defaults to one quarter-note beat. See
`src/music_convert/core.py:_emit` if you want to extend the grammar (e.g. for
explicit duration tokens like `1-` / `1.` / `1_`).

---

## Testing

The repository ships with four test suites. They auto-skip when an external
dependency is missing, so they can be run anywhere.

```bash
# 1) MusicXML <-> MIDI <-> jianpu roundtrip
python tests/smoke.py

# 2) End-to-end CLI pipeline (skip OCR)
python tests/e2e_cli.py

# 3) Complex scores: grand staff, 9 key signatures, chords, Bach prelude
python tests/complex_e2e.py

# 4) Real Audiveris + MuseScore pipeline (skipped if either is missing)
python tests/ocr_e2e.py
```

The final regression on a fully configured machine prints:

```
ALL TESTS PASSED          # smoke
E2E OK                    # e2e_cli
ALL E2E PASSED            # complex_e2e
ALL OCR E2E PASSED        # ocr_e2e
```

---

## Known Limitations

- **Jianpu durations** are emitted as fixed quarter-note beats. Real
  durations, ties, and triplets are decoded from MusicXML but not yet
  re-encoded into the compact jianpu text.
- **Audiveris accuracy** on synthesized/clean scores is good; on real
  hand-written or noisy scans it can drop considerably, especially for dense
  broken-chord patterns (the OCR step may collapse two adjacent eighth notes
  into a single quarter).
- **MuseScore 4 CLI** writes the output filename as `<basename>-1.png` for
  PNG; the wrapper normalizes this back to the user-requested path.

---

## Contributing

We welcome issues and pull requests. See [CONTRIBUTING.md](CONTRIBUTING.md) for development setup, code style, and the contribution workflow.

---

## License

This project is released under the **MIT License**. See [LICENSE](LICENSE)
for the full text.
