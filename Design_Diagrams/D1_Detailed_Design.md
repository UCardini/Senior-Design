---
title: "D1 Detailed Design Document"
author: Carter Smith, Alex Watts
date: 2026-10-08
due: "2026-10-08"
assignment: "Assignment 06 - Detailed System Design, Part 1 (Design D1)"
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

## Header, Scope, and Conventions

**Goal:** Find out if hidden messages are harder to detect in JPEG than in PNG.

**Scope:** This document details C3 (Detection) and C4 (Comparison). C1 (Image Preparation) and C2 (Embedding) are deferred to D2.

### Diagram Conventions

* Boxes are entities; rows list key attributes only. `[PK]` = primary key, `[FK]` = foreign key.
* Solid lines are relationships; cardinality is labeled on each end.
  * `1->M`: one to many (zero or more)
  * `1->1..N`: one to one-or-more
  * `M->1`: many roll up into one

***

## Data Model

\begin{center}
\resizebox{\textwidth}{!}{%
\begin{tikzpicture}[
  font=\sffamily,
  entity/.style={
    rectangle split, rectangle split parts=2, draw, thick,
    rectangle split part fill={black!8,white},
    rectangle split part align={center,left},
    align=left, minimum width=4.6cm, font=\small\sffamily, inner sep=5pt},
  rel/.style={thick},
  card/.style={font=\footnotesize\itshape\sffamily},
  lab/.style={font=\small\bfseries\sffamily, fill=white, inner sep=1.5pt},
]

  % ---- Title and goal ----
  \node[anchor=west, font=\bfseries\sffamily] at (0,1.25)
    {JPEG vs.\ PNG Steganography: D1 Data Model (C3 Detection, C4 Comparison)};
  \node[anchor=west, font=\small\sffamily] at (0,0.7)
    {Goal: Find out if hidden messages are harder to detect in JPEG than in PNG.};

  % ---- Entities ----
  \node[entity, anchor=north west] (img) at (0,0) {%
    \textbf{IMAGE}
    \nodepart{two}
    filename [PK]\\
    format (JPEG $|$ PNG)};

  \node[entity, anchor=north west] (res) at ($(img.north east)+(2.2,0)$) {%
    \textbf{DETECTION\_RESULT}
    \nodepart{two}
    filename [FK]\\
    format (JPEG $|$ PNG)\\
    label (clean $|$ stego)\\
    score (float 0--100)\\
    msg\_ber (float, stego only)};

  \node[entity, anchor=north west] (sum) at ($(res.north east)+(2.2,0)$) {%
    \textbf{COMPARISON\_SUMMARY}
    \nodepart{two}
    format [PK]\\
    label [PK]\\
    mean\_score (float)\\
    detection\_rate (float 0--1)\\
    n (int)};

  \node[entity, anchor=north] (err) at ($(res.south)+(0,-1.6)$) {%
    \textbf{ERROR\_LOG}
    \nodepart{two}
    filename\\
    message};

  % ---- Relationships ----
  \draw[rel] (img.east) -- (img.east -| res.west)
    node[lab, midway, above=2pt] {1->1..N}
    node[card, pos=0.06, below=1pt] {1}
    node[card, pos=0.84, below=1pt] {1..N};

  \draw[rel] (res.east) -- (res.east -| sum.west)
    node[lab, midway, above=2pt] {M->1}
    node[card, pos=0.06, below=1pt] {M}
    node[card, pos=0.94, below=1pt] {1};

  \draw[rel] (img.south) |- (err.west)
    node[card, pos=0.04, right=2pt] {1}
    node[lab, pos=0.72, above=2pt] {1->M}
    node[card, pos=0.97, below=1pt] {M};

  % ---- Conventions ----
  \node[draw, thick, anchor=north west, align=left, font=\small\sffamily,
        inner sep=6pt, text width=5.6cm]
    at ($(sum.south west)+(0,-1.2)$) {%
    \textbf{Conventions}\\[2pt]
    box = entity; rows = key attributes\\
    {[PK]} primary key, {[FK]} foreign key\\
    line = relationship\\
    \textbf{1->M}: one has many (0 or more)\\
    \textbf{1->1..N}: one generates 1 or more\\
    \textbf{M->1}: many roll up into one};

\end{tikzpicture}%
}
\smallskip
\end{center}

**Storage:** CSV files. One for detection records, one for the comparison summary. The initial run uses around 200 records (50 images $\times$ 2 formats $\times$ 2 labels); everything fits in memory and every query is a single pandas pass. We use a fixed count for this experiment, though the design scales to larger datasets without changing the schema. The relational shape (primary and foreign keys) maps directly to a database if needed.

**Structural Decisions**

\begingroup\small
\begin{longtable}{L{0.20\linewidth} L{0.15\linewidth} L{0.41\linewidth} L{0.14\linewidth}}
\hline
\textbf{Decision} & \textbf{Choice} & \textbf{Justification} & \textbf{Req.} \\
\hline
\endhead
Storage format
  & CSV
  & Tabular, human-readable, pandas-native; no server required
  & I4, I5 \\
Score metric
  & Float 0--100
  & Finer confidence than a binary detected/not flag
  & US-01 \\
\texttt{label} on \texttt{DETECTION\_RESULT}
  & Attribute
  & C3 scores both clean and stego images; \texttt{label} lets C4 report true-positive and false-positive rates separately
  & US-02 \\
\texttt{format} as attribute, not entity
  & Attribute
  & Two values only, no data of its own. A separate entity adds a join with no benefit.
  & US-02 \\
Error log separate from results
  & Separate entity
  & Errors carry a message and no score; keeping them apart keeps the results table clean for C4
  & AC-01.2 \\
\hline
\end{longtable}
\endgroup

***

## Core Algorithms

**C3: Chi-Square Detection (Westfeld-Pfitzmann)**

* **Problem:** Detect LSB steganography by testing whether pixel histogram bins show the even/odd equalization that LSB embedding produces.
* **Inputs/Outputs:** A labeled set of images (clean and stego). Produces one scored row per image: filename, format, label, score (float 0--100), and bit-error rate (stego images only).
* **Steps:**
  1. Load each image as grayscale.
  2. Sample the first 25\% of pixel values and build a 256-bin histogram.
  3. Compute the chi-square statistic comparing even/odd bin pairs against their expected equalized distribution.
  4. Convert the p-value to a 0--100 score. High score = likely stego.
  5. On error, log the filename and skip to the next image.
* **Complexity:** $O(N \cdot P)$ where $N$ = images and $P$ = pixels per image. At 200 images the full run takes under 5 seconds.
* **Why:** No training data or external model required; runs on any machine with scipy. Provides the statistical baseline for the JPEG vs.\ PNG comparison.
* **Edge cases:** Empty input directory produces a header-only results file. Corrupt image is logged and skipped. Bit-error rate is computed only for stego images.

**C4: Aggregation and Comparison**

* **Problem:** Summarize detection results and compare JPEG vs.\ PNG side by side.
* **Inputs/Outputs:** Reads the detection results file. Produces a grouped summary (mean score, detection rate, count per format/label combination) and a bar chart.
* **Steps:**
  1. Load detection results. Flag each row as detected if \texttt{score > 50}.
  2. Group by format and label; compute mean score, detection rate, and count.
  3. Write the summary CSV and bar chart.
* **Complexity:** $O(N)$; runs in under a second at any realistic scale.
* **Edge cases:** Missing required column raises a named error. No rows for a format/label group means that group is absent from the summary.

***

## Build-versus-Reuse Decisions

\begingroup\small
\begin{longtable}{L{0.17\linewidth} L{0.11\linewidth} L{0.15\linewidth} L{0.12\linewidth} L{0.32\linewidth}}
\hline
\textbf{Component} & \textbf{Build/Reuse} & \textbf{Library} & \textbf{License} & \textbf{Reason} \\
\hline
\endhead
Dataset download  & Reuse & kagglehub  & Apache 2.0 & Downloads BOSSBase from Kaggle in one call; caches locally \\
Image loading     & Reuse & Pillow     & HPND       & PNG and JPEG I/O with error detection \\
Array / bit ops   & Reuse & numpy      & BSD        & Pixel array manipulation and histogram computation \\
Chi-square CDF    & Reuse & scipy      & BSD        & Converts the chi-square statistic to a 0--100 score \\
Data aggregation  & Reuse & pandas     & BSD        & Groupby aggregation and CSV I/O \\
Chart output      & Reuse & matplotlib & PSF        & Bar chart of detection rates per format \\
C3 / C4 scripts   & Build & N/A        & Internal   & Pipeline logic and experiment orchestration \\
\hline
\end{longtable}
\endgroup

***

## Component Interfaces

**C3 (Detection)**

\begingroup\small
\begin{longtable}{L{0.15\linewidth} L{0.25\linewidth} L{0.25\linewidth} L{0.25\linewidth}}
\hline
\textbf{Interface} & \textbf{Input} & \textbf{Output} & \textbf{On Error} \\
\hline
\endhead
Detection & Labeled image set (clean and stego images, format determined by extension) & \texttt{detection\_results.csv}: columns \texttt{file}, \texttt{format}, \texttt{label}, \texttt{score} (float 0--100, 1 d.p.), \texttt{msg\_ber} (float 0--1, 3 d.p., blank for clean) & Corrupt or unreadable file: row written to error log, batch continues \\
\hline
\end{longtable}
\endgroup

**C4 (Comparison)**

\begingroup\small
\begin{longtable}{L{0.15\linewidth} L{0.25\linewidth} L{0.25\linewidth} L{0.25\linewidth}}
\hline
\textbf{Interface} & \textbf{Input} & \textbf{Output} & \textbf{On Error} \\
\hline
\endhead
Comparison & \texttt{detection\_results.csv} (I5 schema above) & \texttt{comparison.csv}: columns \texttt{format}, \texttt{label}, \texttt{mean\_score}, \texttt{detection\_rate} (float 0--1), \texttt{n}; and \texttt{comparison.png}: bar chart & Missing required column raises named error; pipeline stops \\
\hline
\end{longtable}
\endgroup

**Example output** (\texttt{comparison.csv}):

>```csv
>format,label,mean_score,detection_rate,n
>JPEG,clean,27.81,0.30,50
>JPEG,stego,48.61,0.52,50
>PNG,clean,25.34,0.26,50
>PNG,stego,99.74,0.98,50
>```

***

## Technology Choices with Justification

**Python:** Both team members know Python from coursework and co-ops and can write and debug it without ramp-up time (team skill fit). It is free and open-source under the PSF license (licensing). Python has the largest ecosystem for image processing and data science, with active maintenance on every library we need (community support). The heavy work in this pipeline runs inside compiled NumPy and SciPy code, so Python interpreter overhead is not a bottleneck (performance). Everything runs locally on a standard laptop with no server or hosting cost (cost and hosting). We considered C++ for the pixel-level operations, but the actual bottleneck is in library code that is already compiled, so a rewrite would add maintenance cost with no measurable gain.

**Scientific libraries (NumPy, SciPy, pandas, matplotlib, Pillow):** Every team member has used these in prior coursework (team skill fit). All are BSD or HPND licensed and free (licensing). These are the most widely maintained scientific Python libraries, with millions of users and active support (community support). NumPy and SciPy run compiled C underneath, pandas processes 200 rows in milliseconds, and Pillow handles both JPEG and PNG natively (performance). There are no hosting costs and no package fees (cost and hosting). We looked at the R ecosystem and MATLAB as alternatives. R would require switching the entire pipeline to a different language. MATLAB has equivalent numerical tools but costs money, which rules it out.

**Data storage (CSV):** Both team members are comfortable with the format (team skill fit). CSV requires no license and no server (licensing). It is universally supported and readable with any text editor or spreadsheet tool (community support). At \textasciitilde{}200 rows, a single pandas read takes milliseconds, and every query is one pass (performance). No server infrastructure or hosting is needed (cost and hosting). We considered SQLite and PostgreSQL. At our scale they add no query power that pandas does not already provide, and they introduce a dependency without benefit.

**Dataset (BOSSBase via kagglehub):** BOSSBase is the standard grayscale cover image set used across steganalysis research (community support). Using it makes our detection rates directly comparable to published results. \texttt{kagglehub} downloads and caches the dataset in one call, requires no manual steps, and is Apache 2.0 licensed (licensing, cost). We considered the sample images bundled with scikit-learn as an alternative. That set is too small and not representative of real photographic cover images, which would make our results meaningless against published benchmarks.

## References <!-- nopdf -->
- [Design D0: High-Level System Design](D0_High_Level_Design.md) <!-- nopdf -->
- [User Stories and Use Cases](../User_Stories.md) <!-- nopdf -->
