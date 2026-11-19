---
title: "D2 Detailed Design Document"
author: Carter Smith, Alex Watts
date: 2026-10-14
due: "2026-10-14"
assignment: "Assignment 07 - Detailed System Design, Part 2 (D2)"
autopdf: true
pdfdir: pdfs
header: |
  \usepackage[margin=1in]{geometry}
  \usepackage{tikz}
  \usetikzlibrary{shapes.multipart,positioning,calc}
  \usepackage{longtable}
  \usepackage{array}
  \newcolumntype{L}[1]{>{\raggedright\arraybackslash}p{#1}}
  \newcolumntype{C}[1]{>{\centering\arraybackslash}p{#1}}
  \renewcommand{\arraystretch}{1.2}
  \makeatletter\@beginparpenalty=10000\makeatother
---
**Project:** JPEG vs. PNG Steganography
**Team:** Carter Smith, Alex Watts

***

## Header, Conventions, and D2 Overview

**Goal:** Find out if hidden messages are harder to detect in JPEG than in PNG.

**What D1 covered:** C3 (chi-square detection) and C4 (aggregation and comparison), including the data model, core algorithms, build-versus-reuse decisions, component interfaces, and technology choices.

**What D2 adds:** C1 (image preparation) and C2 (embedding) are detailed here for the first time. This document also adds a full edge-case table, a requirements traceability table, design rationale, and **NEEDS** peer review notes.

### Diagram Conventions

* Solid boxes: components built by the team. Dashed boxes: external systems.
* Solid arrows: data flow, labeled with D0 interface ID where applicable.
* Dashed arrows: error paths.

***

## Finished Components

### C1: Image Preparation

**Data model:** No persistent data. Outputs are files in `data/original/`.

**Behavior:**

1. Download BOSSBase via `kagglehub.dataset_download("lunan0/bossbase-1-01")`.
2. Collect the first `N_IMAGES` files matching known image extensions.
3. For each source image: convert to grayscale, resize to `IMG_SIZE x IMG_SIZE` (LANCZOS), save as both `.png` and `.jpeg` (quality = `JPEG_QUALITY`).
4. Exit with `SystemExit` if no images are found after download.

**Interface (I1 output):**

* Output directory: `data/original/`
* File naming: `img_000.png`, `img_000.jpeg`, ..., `img_049.png`, `img_049.jpeg`
* Format: 8-bit grayscale, `IMG_SIZE x IMG_SIZE` pixels
* Count: `2 x N_IMAGES` files (default: 100)

**Error paths:**

* Kaggle auth not configured or network unavailable: `kagglehub` raises; pipeline stops. Subsequent runs use the local cache.
* Zero images found after download: `SystemExit("C1: no images found under <path>")`.

---

### C2: Embedding

**Data model:** No persistent data. Reads from `data/original/`, writes to `data/encoded/`.

**Behavior:**

1. For each `.png` in `data/original/` (sorted): call stegano (E1a) to embed `MESSAGE` with key `KEY`; save output to `data/encoded/`.
2. For each `.jpeg` in `data/original/` (sorted): call outguess (E1b) to embed `MESSAGE`; save output to `data/encoded/`.
3. On tool failure, log the filename and error to `data/results/embed_errors.log`; continue.

**Interface:**

* Input (I1): `data/original/*.png`, `data/original/*.jpeg`
* Output (I3): `data/encoded/*.png`, `data/encoded/*.jpeg`
* Output (I2): `data/results/embed_errors.log`: one line per skipped file (filename, error message). Empty if no errors.

**Error paths:**

* Embedding tool exits with non-zero code: log to `embed_errors.log`, continue. Handler: `C2.main()`. Recoverable.

**Pipeline diagram with error paths:**

\begin{center}
\resizebox{\textwidth}{!}{%
\begin{tikzpicture}[
  x=0.025cm, y=-0.025cm, font=\sffamily, >=latex,
  comp/.style={rectangle, draw, minimum width=3cm, minimum height=1.5cm,
               align=center, font=\small\sffamily},
  ext/.style={comp, dashed},
  err/.style={comp, fill=red!10},
  fix/.style={comp, fill=green!10},
  lbl/.style={font=\scriptsize\sffamily},
  earrow/.style={->, densely dashed},
]
  % Outer frame, title and goal
  \draw (-100,-119) rectangle (710,251);
  \node[anchor=west, font=\footnotesize\sffamily] at (-98,-110)
    {\textbf{Title:} JPEG vs.\ PNG Steganography, with Error Paths};
  \node[anchor=west, font=\footnotesize\sffamily] at (-98,-94)
    {\textbf{Goal:} Find out if hidden messages are harder to detect in JPEG than in PNG, and show where each component can fail and how each failure is handled.};

  % Components (built by the team)
  \node[comp] (c1) at (160,91) {\textbf{C1}\\(image preparation)};
  \node[comp] (c2) at (320,91) {\textbf{C2}\\(embedding)};
  \node[comp] (c3) at (480,91) {\textbf{C3}\\(detection)};
  \node[comp] (c4) at (640,91) {\textbf{C4}\\(comparison)};

  % External tools and systems
  \node[ext] (kag) at (-30,91) {\textbf{Kaggle}\\(BOSSBase images)};
  \node[ext] (e1)  at (320,211) {\textbf{E1}\\(stegano / outguess)};
  \node[ext] (e2)  at (480,211) {\textbf{E2}\\(detection\\engine)};

  % Interfaces
  \draw[->]  (kag) -- node[above, lbl] {download} (c1);
  \draw[->]  (c1) -- node[above, lbl] {I$_1$} (c2);
  \draw[->]  (c2) -- node[above, lbl] {I$_3$} (c3);
  \draw[->]  (c3) -- node[above, lbl] {I$_5$} (c4);
  \draw[<->] (c2) -- node[right, lbl] {I$_2$} (e1);
  \draw[<->] (c3) -- node[right, lbl] {I$_4$} (e2);

  % Error outcomes
  \node[err] (x1) at (160,-29) {\textbf{SystemExit}\\(fatal)};
  \node[fix] (x2) at (320,-29) {\textbf{Skip + log}\\(I$_2$ errors.log)};
  \node[fix] (x3) at (480,-29) {\textbf{Skip + log}\\(I$_4$ errors.log)};
  \node[fix] (x4) at (640,-29) {\textbf{ValueError}\\(named column)};

  % Error paths
  \draw[earrow] (c1) -- node[right, lbl] {auth/net} (x1);
  \draw[earrow] (c2) -- node[right, lbl] {tool fail} (x2);
  \draw[earrow] (c3) -- node[right, lbl] {corrupt} (x3);
  \draw[earrow] (c4) -- node[right, lbl] {bad CSV} (x4);

  % Legend (bottom-left, sharing the frame edges, right edge beside E1)
  \draw (-100,141) rectangle (255,251);
  \node[align=left, text width=8.7cm, inner sep=1pt, font=\scriptsize\sffamily] at (77.5,196) {%
    \centerline{\textbf{Legend}}
    \begin{tabular}{@{}l@{\hspace{1.5em}}l@{}}
      \textbf{Solid box} = part we build & \textbf{Red box} = fatal / unhandled error\\
      \textbf{Dashed box} = outside tool or system & \textbf{Green box} = handled error
    \end{tabular}\\
    \textbf{Solid arrow} = data moving (labeled I, followed by a number)\\
    \textbf{Dashed arrow} = error path (labeled with its trigger)\\[6pt]
    \textbf{Input:} BOSSBase images from Kaggle and a secret message.\\
    \textbf{Output:} Detection score per format, and an error outcome per component.};
\end{tikzpicture}%
}
\end{center}

---

### Configuration and Logging

**`.env` / `common.py`:** Centralizes all pipeline constants (`N_IMAGES`, `IMG_SIZE`, `JPEG_QUALITY`, `PAYLOAD_FRACTION`, `MESSAGE`, `KEY`). No persistent state; imported by all four components. Changing a value here affects the entire pipeline consistently.

**Error logs:** Plain text, one `filename\tmessage` line per skipped file. C3 produces `data/results/errors.log`; C2 produces `data/results/embed_errors.log`. Written only when errors occur.

---

### D0 Interface Revisions

I1, I3, and I5 are unchanged. **I2** (C2 error output) was undefined in D0 and is defined here: `data/results/embed_errors.log`, same format as I4. No other interfaces needed revision.

***

## Edge Cases and Error Handling

\begingroup\small
\begin{longtable}{L{0.06\linewidth} L{0.07\linewidth} L{0.12\linewidth} L{0.31\linewidth} L{0.31\linewidth}}
\hline
\textbf{ID} & \textbf{Comp.} & \textbf{Category} & \textbf{Edge Case} & \textbf{Handler} \\
\hline
\endhead
EC-01 & C1 & External dep & Kaggle unavailable or auth missing & Pipeline stops with error. Fatal; subsequent runs use the local cache. \\
EC-02 & C1 & Bad input & No images found after download & \texttt{SystemExit} with path. Fatal. \\
EC-03 & C2 & Bad input & Embedding tool fails on a file & Log to \texttt{embed\_errors.log}; continue. Handler: \texttt{C2.main()}. Recoverable. \\
EC-04 & C3 & Bad input & Corrupt or unreadable image file & Skip and log to \texttt{errors.log}; continue. Handler: \texttt{C3.main()}. Recoverable. (AC-01.2) \\
EC-05 & C3 & Boundary & Empty input directory & Header-only CSV written; warning printed. Recoverable. \\
EC-06 & C4 & Bad input & Required column missing from \texttt{detection\_results.csv} & \texttt{ValueError} naming the missing column. Fatal until corrected. Handler: pandas raises on \texttt{groupby}. \\
\hline
\end{longtable}
\endgroup

***

## Requirements Traceability

\begingroup\small
\begin{longtable}{L{0.08\linewidth} L{0.20\linewidth} L{0.09\linewidth} L{0.26\linewidth} L{0.26\linewidth}}
\hline
\textbf{ID} & \textbf{Requirement} & \textbf{Priority} & \textbf{Design Elements} & \textbf{How Satisfied} \\
\hline
\endhead
REQ-01 & Download N\_IMAGES covers, save as PNG and JPEG pairs & Must & C1 behavior steps 1--4, I1 output directory & C1 downloads BOSSBase, resizes to \texttt{IMG\_SIZE}, saves 50 PNG + 50 JPEG pairs \\
REQ-02 & Embed via stegano (E1a) for PNG and outguess (E1b) for JPEG & Must & C2 behavior steps 1--3, I3 output & C2 calls stegano for each \texttt{.png} and outguess for each \texttt{.jpeg}; saves to \texttt{data/encoded/} \\
REQ-03 & Chi-square score for all clean and stego images, written to CSV & Must & C3 chi-square (D1), I5 CSV schema & C3 scores all 200 images and writes one row per image to \texttt{detection\_results.csv} \\
REQ-04 & BER per stego image in \texttt{msg\_ber} column & Must & C3 \texttt{\_ber()} (D1), I5 \texttt{msg\_ber} column & C3 computes BER for stego images; empty string for clean rows \\
REQ-05 & Skip-and-log corrupt files; pipeline continues & Must & EC-04, C3 \texttt{try/except}, I4 \texttt{errors.log} & C3 appends filename and error to \texttt{errors.log} and continues (AC-01.2) \\
REQ-06 & Aggregate by format/label, produce \texttt{comparison.csv} and \texttt{.png} & Must & C4 \texttt{groupby} (D1), comparison outputs & C4 groups by format and label; reports mean score, detection rate, and count \\
NFR-01 & Reproducible results given same KEY and dataset & Should & \texttt{KEY=1234} in \texttt{common.py}; seeded RNG & Seeded RNG produces identical bit streams and chi-square scores across runs \\
NFR-02 & Local execution after first download & Must & All components; \texttt{kagglehub} cache & Pure Python pipeline; Kaggle dataset cached after first run \\
NFR-03 & Open-source permissive licenses & Must & All libraries (see D1) & numpy, scipy, pandas, matplotlib, Pillow, kagglehub; all BSD/Apache/PSF/HPND \\
NFR-04 & Completes in under 60 seconds & Should & C3 $O(N \cdot P)$, C4 $O(N)$ & 200-image run completes in under 60 seconds \\
CON-01 & Published methods only & N/A & DR-02 (stegano/outguess), DR-04 (BOSSBase); tool choices in D1 stack & Scope limited to published steganographic methods and standard datasets \\
CON-02 & Detection framing; no attack tooling & N/A & Design rationale; project scope & Project publishes detection findings only; no novel embedding algorithms developed \\
CON-03 & Free-tier tools and datasets only & N/A & All libraries BSD/Apache/PSF/HPND; BOSSBase free tier (DR-04) & All tools and libraries are free or open-source; no paid services used \\
\hline
\end{longtable}
\endgroup

***

## Design Rationale

**DR-01: Chi-square as the baseline detector.** We chose the chi-square statistical test over Aletheia (E2) because it requires no training data, no model weights, and no external installation; it runs anywhere with scipy. We accepted the trade-off of a weaker detector because the goal is a reproducible statistical baseline, not state-of-the-art detection accuracy. Aletheia remains the planned upgrade path for the final experiment.

**DR-02: stegano (E1a) for PNG, outguess (E1b) for JPEG.** We chose format-appropriate embedders over inline LSB because JPEG compression overwrites LSB-modified pixel values, destroying the payload before detection. Format-aware tools preserve the embedded message across the compression step, producing a valid JPEG vs.\ PNG comparison. We accepted the trade-off of external tool dependencies because they are required for the experiment to be valid.

**DR-03: Flat CSV over SQLite.** We chose CSV files over SQLite because the full dataset fits in memory at \textasciitilde{}200 records and every query is a single pandas pass. We accepted the trade-off of no query language or transactions because they add no value at this scale.

**DR-04: BOSSBase over sklearn sample images.** We chose BOSSBase via kagglehub over sklearn's bundled images because BOSSBase is the standard cover set for steganalysis research, making results directly comparable to published benchmarks. We accepted the trade-off of a Kaggle dependency because it is free, one-time, and cached after the first run.

**DR-05: Sequential pipeline over parallel execution.** We chose a sequential C1$\to$C2$\to$C3$\to$C4 runner over parallel stages because the bottleneck at \textasciitilde{}200 images is I/O, not CPU, and sequential execution is simpler to debug. We accepted the trade-off of longer wall time only because it does not matter at our scale.

**DR-06: JPEG quality 90.** We chose quality 90 over lower values because it represents a typical high-quality setting and applies measurable but not aggressive compression, consistent with steganalysis literature. We accepted the trade-off of less visible compression because the goal is a fair baseline, not an adversarial one.

**DR-07: Grayscale over RGB.** We chose grayscale conversion because BOSSBase is grayscale and most steganalysis literature uses grayscale, which removes chroma subsampling as a confounding variable. We accepted the trade-off of discarding color information because color is not part of the research question.

**DR-08: `common.py` constants over CLI flags.** We chose centralised constants over argparse CLI flags because the pipeline is a research script run by the team, not a tool with external users. We accepted the trade-off of requiring a code edit to change parameters, which is fine for a two-person team.

<!-- *** -->
<!---->
<!-- ## Peer Design Review Notes -->
<!---->
<!-- | ID | Criterion | Issue | Team | Action | Owner | Priority | Due | -->
<!-- |---|---|---|---|---|---|---|---| -->
