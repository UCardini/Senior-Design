---
title: "Technical Specification"
author: Carter Smith, Alex Watts
date: 2026-11-18
due: "2026-10-30"
assignment: "Assignment 09 - Requirements Document and Technical Specification"
autopdf: true
pdfdir: pdfs
header: |
  \usepackage[margin=1in]{geometry}
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

## Version and Change Log

\begingroup\small

\begin{longtable}{C{0.03\linewidth} C{0.15\linewidth} L{0.49\linewidth} L{0.23\linewidth}}
\hline
\textbf{Version} & \textbf{Date} & \textbf{Change} & \textbf{Author} \\
\hline
\endhead
1.0 & 2026-10-30 & Initial release. Architecture summary, component interfaces, stack justification, and traceability matrix derived from D0--D2. Traces to Requirements Document v1.0. & Carter Smith, Alex Watts \\
1.1 & 2026-11-18 & Revised to align with actual implementation: component interfaces, stack, and traceability matrix corrected. Advisor review pending. & Carter Smith \\
\hline
\end{longtable}
\endgroup

This document traces to Requirements Document v1.1.

***

## Architecture Summary

The system is a sequential four-component research pipeline. It takes BOSSBase grayscale images and a fixed message as inputs and produces per-format detection scores and a statistical comparison as output.

**Pipeline sequence:** C1 (Image Preparation) $\to$ C2 (Embedding) $\to$ C3 (Steganography Detection) $\to$ C4 (Detection Score Comparison). C3 uses chi-square (Westfeld-Pfitzmann) as the baseline detector.

**Deployment model:** A single-machine Python environment. All components run as Python modules invoked sequentially by \texttt{run.py}. No server, no network dependency after the first \texttt{kagglehub} download (NFR-02). The pipeline is designed for a research workstation, not production deployment.

**Data flow:** BOSSBase images are downloaded and saved to \texttt{data/original/} (I1). C2 reads those images, embeds the message, and saves stego images to \texttt{data/encoded/} (I3). C3 reads both directories and writes one scored row per image to \texttt{data/results/detection\_results.csv} (I5). C4 reads that CSV, aggregates by format and label, and writes \texttt{comparison.csv} and \texttt{comparison.png}.

**Diagram reference:** \texttt{Design\_Diagrams/D0\_High\_Level\_Design.md} (block diagram), \texttt{Design\_Diagrams/D2\_Detailed\_Design.md} (pipeline with error paths).

***

## Component Interfaces and Data Formats

### C1: Image Preparation

**Input:** Kaggle dataset download via \texttt{kagglehub.dataset\_download("lunan0/bossbase-1-01")} (external; no defined schema).

**Output (I1):**

- Directory: \texttt{data/original/}
- Files: \texttt{img\_000.png} \ldots \texttt{img\_049.png} and \texttt{img\_000.jpeg} \ldots \texttt{img\_049.jpeg}
- Format: 8-bit grayscale, \texttt{IMG\_SIZE $\times$ IMG\_SIZE} pixels (default: 256 $\times$ 256)
- Count: $2 \times$ \texttt{N\_IMAGES} files (default: 100)

**Error behavior:**

- Kaggle credentials missing $\to$ \texttt{kagglehub} raises; pipeline stops. User must supply \texttt{\textasciitilde{}/.kaggle/kaggle.json}.
- No images found after download $\to$ \texttt{SystemExit("C1: no images found under \textless{}path\textgreater{}")}; fatal.

---

### C2: Embedding

**Input (I1):** \texttt{data/original/*.png} and \texttt{data/original/*.jpeg}.

**Output (I3):** \texttt{data/encoded/\{filename\}}: same filename as the input, one stego image per cover.

**Output (I2):** \texttt{data/results/embed\_errors.log}: tab-separated lines of \texttt{filename\textbackslash{}terror\_message}. Empty if no errors. (EC-03 gap: not yet written; fix tracked in T01.)

**Embedding behavior (D2 C2):**

1. For each \texttt{.png} in \texttt{data/original/} (sorted): call stegano (E1a) to embed \texttt{MESSAGE} using least significant bit (LSB) embedding with key \texttt{KEY}; save output to \texttt{data/encoded/}.
2. For each \texttt{.jpeg} in \texttt{data/original/} (sorted): call outguess (E1b) to embed \texttt{MESSAGE} via DCT-domain embedding; save output to \texttt{data/encoded/}.
3. On tool failure: log filename and error to I2 (\texttt{embed\_errors.log}); continue (EC-03).

**Error behavior (designed):** Embedding tool exits with non-zero code: log to \texttt{embed\_errors.log}, continue (EC-03). Handler: \texttt{C2.main()}. Implementation pending T01.

---

### C3: Steganography Detection

**Input:** \texttt{data/original/} (clean images, label = \texttt{clean}) and \texttt{data/encoded/} (stego images, label = \texttt{stego}).

**Output (I5):** \texttt{data/results/detection\_results.csv}

\begingroup\small
\begin{longtable}{L{0.13\linewidth} L{0.17\linewidth} L{0.62\linewidth}}
\hline
\textbf{Column} & \textbf{Type} & \textbf{Description} \\
\hline
\endhead
\texttt{file} & string & Filename including extension, e.g.\ \texttt{img\_000.jpeg}. \\
\texttt{format} & string & \texttt{JPEG} or \texttt{PNG} (derived from file extension, uppercased). \\
\texttt{label} & string & \texttt{clean} or \texttt{stego}. \\
\texttt{score} & float & Chi-square steganalysis score, 0.0--100.0, rounded to 1 decimal place. Higher values indicate a higher probability of steganographic content. \\
\texttt{msg\_ber} & float or empty & Bit-error rate for stego images, 0.000--1.000 rounded to 3 decimal places. Empty string for clean images. \\
\hline
\end{longtable}
\endgroup

**Output (I4):** \texttt{data/results/errors.log}: tab-separated lines of \texttt{filename\textbackslash{}terror\_message}. Written only when at least one image cannot be opened.

**Chi-square (Westfeld-Pfitzmann) algorithm:** For the first 25\% of pixels (\texttt{frac=0.25} in \texttt{\_chi()}; this is a fixed sample parameter, independent of \texttt{PAYLOAD\_FRACTION}), bin pixel values 0--255 into 128 even/odd pairs. For each pair $(2k, 2k+1)$ where the expected count exceeds 5, compute the chi-square statistic. Return $100 \times (1 - \text{CDF}(\chi^2, \text{df}))$. Complexity: $O(N \cdot 0.25P)$ where $N$ = images, $P$ = pixels per image.

**Error behavior:** Corrupt or unreadable file $\to$ appended to \texttt{errors.log}, batch continues (EC-04, satisfies REQ-05 and AC-01.2).

---

### C4: Detection Score Comparison

**Input:** \texttt{data/results/detection\_results.csv} (I5 schema above).

**Output:** \texttt{data/results/comparison.csv}: one row per (format, label) group with columns \texttt{mean\_score}, \texttt{detection\_rate} (fraction with score $>$ 50), and \texttt{n} (count).

**Output:** \texttt{data/results/comparison.png}: bar chart of detection rate by format and label, 120 DPI.

**Error behavior:** Required column missing $\to$ \texttt{ValueError} naming the missing column; pipeline stops (EC-06). No valid rows for a group $\to$ group absent from summary with no warning (open gap; fix tracked in T07).

---

### Shared Configuration: \texttt{common.py}

All pipeline constants are centralized in \texttt{src/common.py}. No persistent state; imported by all four components.

\begingroup\small
\begin{longtable}{L{0.23\linewidth} L{0.17\linewidth} L{0.52\linewidth}}
\hline
\textbf{Constant} & \textbf{Default} & \textbf{Effect} \\
\hline
\endhead
\texttt{N\_IMAGES} & 50 & Cover images downloaded and processed per format. \\
\texttt{IMG\_SIZE} & 256 & Pixel dimension (width = height) for all prepared images. \\
\texttt{JPEG\_QUALITY} & 90 & JPEG save quality passed to Pillow. Affects BER. \\
\texttt{PAYLOAD\_FRACTION} & 0.5 & Fraction of pixels whose LSB carries a message bit. \\
\texttt{MESSAGE} & \texttt{"The quick brown fox..."} & Fixed plaintext payload embedded by C2. \\
\texttt{KEY} & 1234 & Seed for the XOR keystream RNG. Changing this changes the BER. \\
\hline
\end{longtable}
\endgroup

***

## Technology Stack

\begingroup\small
\begin{longtable}{L{0.15\linewidth} L{0.20\linewidth} L{0.57\linewidth}}
\hline
\textbf{Layer} & \textbf{Choice} & \textbf{Justification} \\
\hline
\endhead
Language & Python 3.11 & Team skill fit (CON-01); large scientific library ecosystem (REQ-03); no licensing cost (CON-03). Alternative: Julia was rejected due to lower team familiarity. \\
Image I/O & Pillow 10.x & BSD license (NFR-03); standard for grayscale manipulation in Python; Pillow's JPEG quality parameter maps directly to \texttt{JPEG\_QUALITY} (REQ-02). \\
Numerical & NumPy 1.26 & BSD license (NFR-03); vectorized LSB operations in C2 and BER calculation in C3 complete in milliseconds (NFR-04). \\
Statistics & SciPy 1.12 & BSD license (NFR-03); \texttt{chi2.cdf} provides the closed-form CDF needed by the Westfeld-Pfitzmann test (REQ-03). \\
Data analysis & pandas 2.1 & BSD license (NFR-03); \texttt{groupby} aggregation for per-format summary in C4 (REQ-06). \\
Visualization & matplotlib 3.8 & PSF-compatible license (NFR-03); bar chart of detection rates (REQ-06). \\
Dataset & kagglehub + BOSSBase & Free tier, no cost (CON-03); BOSSBase is the standard steganalysis cover set, making results comparable to published work (DR-04). Alternative: sklearn sample images rejected because the sample set is too small and non-standard. \\
\hline
\end{longtable}
\endgroup

***

## Known Constraints

\begingroup\small
\begin{longtable}{C{0.09\linewidth} L{0.17\linewidth} L{0.67\linewidth}}
\hline
\textbf{ID} & \textbf{Area} & \textbf{What it rules out or forces} \\
\hline
\endhead
CON-01 & Professional & Rules out novel attack algorithm development. Forces the experimental scope to published methods: inline LSB for PoC, outguess for the real-tools track. \\
CON-02 & Ethical / Security & Rules out publishing embedding tools or payloads usable as attack aids. Forces results to be framed as detection findings for defenders. \\
CON-03 & Budget & Rules out paid cloud GPU instances (Aletheia training requires compute time), paid dataset licenses, and proprietary steganalysis tools. Forces all choices to open-source or free-tier alternatives. \\
\hline
\end{longtable}
\endgroup

***

## Traceability Matrix

Status values: \textbf{Designed} = design element is specified in D0--D2; \textbf{In progress} = implementation started, not all conditions met; \textbf{Planned} = not yet implemented.

\begingroup\footnotesize
\begin{longtable}{C{0.08\linewidth} L{0.21\linewidth} L{0.30\linewidth} C{0.11\linewidth} C{0.17\linewidth}}
\hline
\textbf{Req ID} & \textbf{Description} & \textbf{Design Component (D0/D1/D2)} & \textbf{Status} & \textbf{Test Case} \\
\hline
\endhead
REQ-01 & Download N\_IMAGES covers, save as PNG and JPEG pairs & C1 \texttt{main()}, I1 output directory, D2 C1 behavior steps 1--4 & Designed & TC-01 \\
REQ-02 & Embed message via stegano (E1a) for PNG and outguess (E1b) for JPEG & C2 \texttt{main()}, I3 output, D2 C2 behavior steps 1--3 (stegano for PNG, outguess for JPEG) & Designed & TC-02 \\
REQ-03 & Chi-square score for all clean and stego images, written to CSV & C3 \texttt{main()}, \texttt{\_chi()}, I5 CSV schema, D1 algorithm section & Designed & TC-03, TC-04 \\
REQ-04 & BER per stego image in msg\_ber column & C3 \texttt{\_ber()}, I5 \texttt{msg\_ber} column, D2 C3 interface & Designed & TC-05 \\
REQ-05 & Skip-and-log corrupt files; pipeline continues & C3 \texttt{try/except} block (I4 errors.log); C2 EC-03 (designed, implementation pending T01) & In progress & TC-06 \\
REQ-06 & Aggregate by format/label, produce comparison.csv and .png & C4 \texttt{main()}, \texttt{groupby} aggregation, D1 C4 algorithm & Designed & TC-07 \\
NFR-01 & Reproducible results given same KEY and dataset & \texttt{KEY=1234} in \texttt{common.py}; seeded \texttt{np.random.default\_rng(KEY)} in C2 and C3 & Designed & TC-08 \\
NFR-02 & Local execution after first download & \texttt{kagglehub} local cache; all processing in pure Python with no server & Designed & N/A (satisfied by design) \\
NFR-03 & Open-source permissive licenses & Build/reuse table in D1; all libraries BSD/Apache/PSF/HPND & Designed & N/A \\
NFR-04 & Run completes in under 60 seconds & C3 \texttt{\_chi()} O(N$\cdot$P) with 25\textbackslash{}\% sample; C4 O(N) aggregation & Designed & TC-09 \\
CON-01 & Published methods only & D2 DR-02 (inline LSB), DR-04 (BOSSBase); tool choices in stack table & Designed & N/A \\
CON-02 & Detection framing; no attack tooling & D2 design rationale; scope section of this document & Designed & N/A \\
CON-03 & Free-tier tools and datasets only & Stack table licenses; BOSSBase free tier (DR-04) & Designed & N/A \\
\hline
\end{longtable}
\endgroup

**Reverse check:** All design elements in D0--D2 (C1--C4 components, I1--I5 interfaces, \texttt{common.py} configuration) trace to at least one requirement or constraint above. \texttt{common.py} constants are infrastructure supporting NFR-01 and REQ-01--REQ-06; they are not scope creep.

***

## Advisor Review Record

\begingroup\small
\begin{longtable}{L{0.21\linewidth} L{0.74\linewidth}}
\hline
\textbf{Field} & \textbf{Detail} \\
\hline
\endhead
Advisor & Justin Zhan, CEAS Computer Science \\
Review date & Pending \\
Documents reviewed & Pending \\
Items discussed & Pending \\
Changes produced & Pending \\
\hline
\end{longtable}
\endgroup

## References <!-- nopdf -->
- [Requirements Document](Requirements_Document.md) <!-- nopdf -->
- [User Stories and Use Cases](User_Stories.md) <!-- nopdf -->
- [D0 High-Level Design](Design_Diagrams/D0_High_Level_Design.md) <!-- nopdf -->
- [D1 Detailed Design](Design_Diagrams/D1_Detailed_Design.md) <!-- nopdf -->
- [D2 Detailed Design](Design_Diagrams/D2_Detailed_Design.md) <!-- nopdf -->
