---
title: "Design D0: High-Level System Design"
author: Carter Smith, Alex Watts
date: 2026-10-02
due: "2026-10-02"
assignment: "Assignment 05 - High-Level System Design (Design D0)"
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

**Goal:** Find out if hidden messages are harder to detect in JPEG than in PNG.  
**Input:** Images and a secret message.  
**Output:** Detection score for each format.

### Conventions:

* Solid box = part we build
* Dashed box = outside tool
* Arrow = Data moving (labeled I, followed by a number)

## Block Diagram (the D0 diagram)
<!-- ![Block Diagram](D0_Block_Diagram.svg)\  -->
\begin{center}
\resizebox{\textwidth}{!}{%
\begin{tikzpicture}[
  x=0.025cm, y=-0.025cm, font=\sffamily, >=latex,
  comp/.style={rectangle, draw, minimum width=3cm, minimum height=1.5cm,
               align=center, font=\small\sffamily},
  ext/.style={comp, dashed},
  lbl/.style={font=\scriptsize\sffamily},
]
  % Outer frame, title and goal
  \draw (0,1) rectangle (710,251);
  \node[anchor=west, font=\footnotesize\sffamily] at (2,10)
    {\textbf{Title:} Steganography on JPEG vs.\ PNG};
  \node[anchor=west, font=\footnotesize\sffamily] at (2,26)
    {\textbf{Goal:} Find out if hidden messages are harder to detect in JPEG than in PNG.};

  % Components (built by the team)
  \node[comp] (c1) at (160,91) {\textbf{C1}\\(images preparation)};
  \node[comp] (c2) at (320,91) {\textbf{C2}\\(embedding data)};
  \node[comp] (c3) at (480,91) {\textbf{C3}\\(detection)};
  \node[comp] (c4) at (640,91) {\textbf{C4}\\(comparison)};

  % External tools
  \node[ext] (e1) at (320,211) {\textbf{E1}\\(Steganography\\Embedding Tool)};
  \node[ext] (e2) at (480,211) {\textbf{E2}\\(Steganography\\Detection Tool)};

  % Interfaces
  \draw[->]  (c1) -- node[above, lbl] {I$_1$} (c2);
  \draw[->]  (c2) -- node[above, lbl] {I$_3$} (c3);
  \draw[->]  (c3) -- node[above, lbl] {I$_5$} (c4);
  \draw[<->] (c2) -- node[right, lbl] {I$_2$} (e1);
  \draw[<->] (c3) -- node[right, lbl] {I$_4$} (e2);

  % Legend (bottom-left, sharing the frame edges)
  \draw (0,141) rectangle (255,251);
  \node[align=left, text width=6.25cm, inner sep=1pt, font=\scriptsize\sffamily] at (127.5,196) {%
    \centerline{\textbf{Legend}}
    \textbf{Solid box} = part we build\\
    \textbf{Dashed box} = outside tool\\
    \textbf{Arrow} = data moving (labeled I, followed by a number)\\[6pt]
    \textbf{Input:} Images and a secret message.\\
    \textbf{Output:} Detection score for each format.};
\end{tikzpicture}%
}
\end{center}



| ID | Component |
|----|-----------|
| C1 | Images Preparation |
| C2 | Data Embedding |
| C3 | Detection |
| C4 | Comparison |
| E1 | Steganography Embedding Tool *(external)* |
| E2 | Steganography Detection Tool *(external)* |

## Component Responsibility Table

| Component | Responsibility | Interface In | Interface Out | Owner |
|-----------|---------------|--------------|---------------|-------|
| C1 | Prepare images in both file formats | N/A | I1 | Carter |
| C2 | Hide data in images using E1 | I1, I2 | I2, I3 | Carter |
| C3 | Check images for hidden data using E2 | I3, I4 | I4, I5 | Alex |
| C4 | Compare each format's detection rate | I5 | N/A | Alex |

## Interface Specification Table(s)

**Data**

| ID | Input | Output | Format |
|----|-------|--------|--------|
| I1 | Image files | Image files | `*.JPEG`, `*.PNG` |
| I2 | Image file, payload | Encoded image file | `*.JPEG`, `*.PNG`, `*.TXT` |
| I3 | Encoded image files | Encoded image files | `*.JPEG`, `*.PNG` |
| I4 | Encoded image file | Detection results | `*.JPEG`, `*.PNG`, `*.CSV` |
| I5 | Detection results | Comparison results | `*.CSV` |

**Behavior**

| ID | Mechanism | On Error |
|----|-----------|----------|
| I1 | Written to filesystem | C2 rejects image |
| I2 | CLI call to E1 | C2 rejects image |
| I3 | Written to filesystem | C3 rejects image |
| I4 | CLI call to E2 | C3 rejects image |
| I5 | Written to filesystem | C4 rejects data |


**Example payload (I5 output):**

| file | format | label | score | msg_ber |
|------|--------|-------|-------|---------|
| img_000.jpeg | JPEG | stego | 48.6 | 0.470 |
| img_000.png | PNG | stego | 99.7 | 0.000 |


## Data Flow Diagram
<!-- ![Data Flow Diagram](D0_Data_Flow_Diagram.svg)\ -->
\begin{center}
\resizebox{\textwidth}{!}{%
\begin{tikzpicture}[
  x=0.025cm, y=-0.025cm, font=\sffamily, >=latex,
  comp/.style={rectangle, draw, minimum width=3cm, minimum height=1.5cm,
               align=center, font=\small\sffamily},
  ext/.style={comp, dashed},
  lbl/.style={font=\scriptsize\sffamily},
]
  % Outer frame, title and goal
  \draw (0,1) rectangle (840,251);
  \node[anchor=west, font=\footnotesize\sffamily] at (2,10)
    {\textbf{Title:} Steganography on JPEG vs.\ PNG};
  \node[anchor=west, font=\footnotesize\sffamily] at (2,26)
    {\textbf{Goal:} Find out if hidden messages are harder to detect in JPEG than in PNG.};

  % Components (built by the team)
  \node[comp] (c1) at (160,91) {\textbf{C1}\\(images preparation)};
  \node[comp] (c2) at (360,91) {\textbf{C2}\\(embedding data)};
  \node[comp] (c3) at (560,91) {\textbf{C3}\\(detection)};
  \node[comp] (c4) at (760,91) {\textbf{C4}\\(comparison)};

  % External tools
  \node[ext] (e1) at (360,211) {\textbf{E1}\\(Steganography\\Embedding Tool)};
  \node[ext] (e2) at (560,211) {\textbf{E2}\\(Steganography\\Detection Tool)};

  % Data flow
  \draw[->] (c1) -- node[above, lbl] {Images (original)} (c2);
  \draw[->] (c2) -- node[above, lbl] {Images (encoded)}  (c3);
  \draw[->] (c3) -- node[above, lbl] {Scores (CSV file)} (c4);
  \draw[<->] (c2) -- node[right, lbl, align=left]
    {Image (original) + message,\\Returns: encoded image} (e1);
  \draw[<->] (c3) -- node[right, lbl, align=left]
    {Image (encoded),\\Returns: score} (e2);

  % Legend (bottom-left, sharing the frame edges)
  \draw (0,141) rectangle (245,251);
  \node[align=left, text width=5.9cm, font=\scriptsize\sffamily] at (122.5,196) {%
    \centerline{\textbf{Legend}}
    \textbf{Solid box} = part we build\\
    \textbf{Dashed box} = outside tool\\
    \textbf{Arrow} = data moving\\[6pt]
    \textbf{Input:} Images and a secret message.\\
    \textbf{Output:} Detection score for each format.};
\end{tikzpicture}%
}
\end{center}

***

## Architecture Pattern and Justification

**Pattern:** Pipeline, used for the whole system.

**Fit to the problem:** The experiment is a fixed sequence of four steps with no branching or feedback loop. Each step produces a file that the next step consumes. A pipeline pattern maps directly to this structure.

**Performance and timing:** The pipeline must complete a 50-image-per-format run in under 60 seconds on a standard workstation. The bottleneck is I/O and per-pixel chi-square computation, not CPU-bound parallelism.

**Rejected pattern:** Layered architecture. The project has no separate UI, business logic, or storage tiers; a layered pattern would add structure with no benefit.

## Decision log

1. **Use a pipeline pattern** - Alternatives: layered architecture. The experiment is a fixed sequential flow with no feedback loop; a pipeline maps directly to it. Layered architecture would add UI and storage tiers that don't exist.
2. **Write each step in Python** - Alternatives: R, MATLAB. Both team members have Python experience from coursework and co-ops. R requires switching the entire pipeline language; MATLAB costs money.
3. **Use existing embedding and detection tools** Alternatives: writing custom LSB and chi-square implementations. Reusing stegano, outguess, and chi-square via scipy tests tools that practitioners actually use and makes results comparable to published benchmarks.
4. **Measure a continuous detection score per image** Alternatives: binary detected/not. A continuous 0-100 score from the chi-square CDF shows how strongly the detector suspects steganography, enabling mean score comparisons across formats.
