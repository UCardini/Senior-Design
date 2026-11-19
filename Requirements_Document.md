---
title: "Requirements Document"
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
1.0 & 2026-10-30 & Initial release. Functional and non-functional requirements derived from D2 traceability table and user stories. MoSCoW priorities, constraint IDs (CON-01--CON-03), and glossary established. & Carter Smith, Alex Watts \\
1.1 & 2026-11-18 & Revised to align with actual codebase: component names, JPEG quality, NFRs, and constraints corrected. Reformatted to project style guide. & Carter Smith \\
\hline
\end{longtable}
\endgroup

This document traces to Technical Specification v1.1.

***

## Purpose and Scope

**Goal:** Find out if hidden messages are harder to detect in JPEG images than in PNG images.

This document specifies what the system must do and for whom. It consolidates the stakeholder goals from User\_Stories.md, the acceptance criteria from UC-01, and the non-functional requirements from D2 into a single formal requirements baseline. All requirement IDs assigned here (REQ-\#\#, NFR-\#\#, CON-\#\#) are stable and must not be renumbered; withdrawn requirements are marked \textbf{WITHDRAWN} and logged in the change log.

**In scope:** The four-component pipeline C1 $\to$ C2 $\to$ C3 $\to$ C4. C2 (Embedding) is designed to use stegano (E1a) for PNG and outguess (E1b) for JPEG; C3 (Detection) uses chi-square as the baseline detector, with Aletheia (E2) as the planned upgrade (T10). The PoC used inline LSB embedding to validate the pipeline structure; the production implementation follows the D2 design.

**Out of scope:** Graphical user interface, real-time image processing, support for formats other than JPEG and PNG, novel steganographic algorithm development.

***

## Stakeholders and User Goals

\begingroup\small
\begin{longtable}{C{0.10\linewidth} L{0.24\linewidth} L{0.59\linewidth}}
\hline
\textbf{Type} & \textbf{Stakeholder} & \textbf{Goal} \\
\hline
\endhead
Primary & Digital forensics analyst & Know whether JPEG images are harder to detect than PNG so they can identify which images are vulnerable (US-01). \\
Secondary & Image security researcher & See JPEG and PNG detection results side by side to decide which format to use in a demonstration (US-02). \\
Hidden & Security software developer & Know which file type hides information better so that scanning systems can prioritize higher-risk formats (US-03). \\
\hline
\end{longtable}
\endgroup

***

## Glossary

\begingroup\small
\begin{longtable}{L{0.24\linewidth} L{0.71\linewidth}}
\hline
\textbf{Term} & \textbf{Definition} \\
\hline
\endhead
BER (bit-error rate) & Fraction of embedded bits that differ from the original payload bits; range 0.00--1.00. BER 0.00 means the payload is intact; BER $\approx$ 0.47 means compression destroyed roughly half the payload. \\
BOSSBase & Break Our Steganographic System Base; a publicly available dataset of 10,000 $512 \times 512$ grayscale images used as the standard cover source in steganalysis research. \\
Chi-square steganalysis & The Westfeld-Pfitzmann statistical test; detects LSB embedding by testing whether pairs of adjacent palette values $(2k, 2k+1)$ have equal frequency. Returns a score 0--100 (higher = more likely stego). \\
Cover image & An unmodified source image used as the carrier for a hidden message. \\
DCT & Discrete Cosine Transform; the compression domain used by JPEG. LSB bits written to the pixel domain are destroyed by DCT quantization during save. \\
Detection rate & Fraction of stego images in a group with a chi-square score $>$ 50; expressed as a float 0--1. \\
Grayscale & Single-channel 8-bit image with pixel values 0--255. \\
JPEG & Joint Photographic Experts Group; a lossy image format using DCT compression. \\
LSB embedding & Least significant bit embedding; hides data by overwriting the least significant bit of each selected pixel value. \\
Mean score & Arithmetic mean of chi-square detection scores for a group; expressed on a 0--100 scale. \\
Payload fraction & Ratio of message bits to total pixel count; controls how much of the image is modified. \\
PNG & Portable Network Graphics; a lossless image format that preserves LSB-embedded bits exactly. \\
Stego image & An image that contains a hidden message embedded via steganography. \\
Steganalysis & The practice of detecting the presence of steganographic content in digital media. \\
\hline
\end{longtable}
\endgroup

***

## Functional Requirements

Each statement uses the form "The system shall \ldots" with a measurable condition. IDs are stable; a dropped requirement is marked \textbf{WITHDRAWN} in the change log and not renumbered.

\begingroup\small
\begin{longtable}{C{0.07\linewidth} L{0.33\linewidth} C{0.09\linewidth} C{0.10\linewidth} L{0.27\linewidth}}
\hline
\textbf{ID} & \textbf{Statement} & \textbf{MoSCoW} & \textbf{Source} & \textbf{Verification} \\
\hline
\endhead
REQ-01 & The system shall download exactly \texttt{N\_IMAGES} (default: 50) grayscale cover images from BOSSBase and save each as a PNG and a JPEG pair at \texttt{IMG\_SIZE $\times$ IMG\_SIZE} pixels (default: 256 $\times$ 256). & Must & US-01 & Count files in \texttt{data/original/}; assert 100 files, 50 per format. \\
REQ-02 & The system shall embed a fixed message (\texttt{MESSAGE}) into every PNG cover image using stegano (E1a) and every JPEG cover image using outguess (E1b), producing one stego image per cover, saved to \texttt{data/encoded/}. & Must & US-01 & Count files in \texttt{data/encoded/}; assert 100 files, 50 per format. \\
REQ-03 & The system shall compute a chi-square steganalysis score (0--100) for every clean and stego image and write one row per image to \texttt{detection\_results.csv} with columns \texttt{file}, \texttt{format}, \texttt{label}, \texttt{score}. & Must & US-01, US-02, AC-01.1 & Load CSV; assert 200 rows (100 clean + 100 stego), no missing scores. \\
REQ-04 & The system shall compute the bit-error rate (BER) for every stego image and record it in the \texttt{msg\_ber} column of \texttt{detection\_results.csv}; the \texttt{msg\_ber} column shall be empty for clean images. & Must & US-03 & Load CSV stego rows; assert all \texttt{msg\_ber} values are numeric and in range 0.0--1.0. \\
REQ-05 & When an image file cannot be opened, the system shall skip that file, append its filename and error message to an errors log, and continue processing the remaining images without raising an unhandled exception. & Must & AC-01.2, EC-04 & Inject a corrupt file; assert pipeline exits normally, errors log contains the filename, CSV contains the remaining images. \\
REQ-06 & The system shall aggregate \texttt{detection\_results.csv} by format and label, compute detection rate (fraction of stego images with score $>$ 50) and mean score per group, write \texttt{comparison.csv}, and render \texttt{comparison.png}. & Must & US-02, US-03 & Assert \texttt{comparison.csv} has 4 rows (JPEG/PNG $\times$ clean/stego) and \texttt{comparison.png} exists and is non-zero in size. \\
\hline
\end{longtable}
\endgroup

***

## Non-functional Requirements

\begingroup\small
\begin{longtable}{C{0.07\linewidth} L{0.33\linewidth} C{0.09\linewidth} C{0.10\linewidth} L{0.28\linewidth}}
\hline
\textbf{ID} & \textbf{Statement} & \textbf{MoSCoW} & \textbf{Source} & \textbf{Verification} \\
\hline
\endhead
NFR-01 & Given the same \texttt{KEY} (default: 1234) and the same cached dataset, the system shall produce bit-for-bit identical \texttt{detection\_results.csv} on repeated runs on the same machine. & Should & D2 NFR-01 & Run twice, diff the two CSV files; assert zero differences. \\
NFR-02 & After the first \texttt{kagglehub} download, the system shall complete a full pipeline run with no network connection. & Must & D2 NFR-02 & Disable network, run full pipeline; assert no network-related exception. \\
NFR-03 & All third-party libraries used by the pipeline shall have BSD, Apache 2.0, PSF, or HPND licenses. & Must & D2 NFR-03 & Run \texttt{pip-licenses}; assert no license outside the allowed set. \\
NFR-04 & The system shall complete a 50-image-per-format pipeline run in under 60 seconds on a workstation with at least 4 GB RAM and a standard SSD. & Should & D2 NFR-04 & Time the run on the reference machine; assert elapsed $<$ 60 s. \\
\hline
\end{longtable}
\endgroup

***

## MoSCoW Summary

**Must (MVP - required for a valid experiment):**
REQ-01, REQ-02, REQ-03, REQ-04, REQ-05, REQ-06, NFR-02, NFR-03.

**Should (important but not MVP-blocking):**
NFR-01 (reproducibility), NFR-04 (60-second run time).

**Could (desirable, deferred to real-tools track):**
CLI parameter flags for \texttt{N\_IMAGES}, \texttt{JPEG\_QUALITY}, and \texttt{PAYLOAD\_FRACTION}; Aletheia (E2) real-tools integration (T10); outguess (E1b) JPEG real-tools integration (T09); JPEG quality sweep across 50/75/90/95 (T11).

**Won't (explicitly out of scope for this release):**
Graphical user interface; real-time image streaming; support for image formats other than JPEG and PNG; development of novel steganographic algorithms; network-based deployment.

***

## Inherited Constraints

These constraints derive from the Week 3 essay (\texttt{Constraints\_Essay.md}) and bound all design decisions across the project.

\begingroup\small
\begin{longtable}{C{0.09\linewidth} L{0.17\linewidth} L{0.67\linewidth}}
\hline
\textbf{ID} & \textbf{Area} & \textbf{Constraint} \\
\hline
\endhead
CON-01 & Professional & The team are generalist CS students, not credentialed security researchers. The experimental scope is limited to publicly available, published steganographic methods and standard datasets. No contribution requires specialized security expertise beyond the team's background. \\
CON-02 & Ethical / Security & Steganography research has dual-use potential. The project publishes detection findings, not attack tooling. No novel embedding algorithm will be developed or released. Results are presented in an academic context with responsible-disclosure framing. \\
CON-03 & Budget & No paid software, cloud infrastructure, or dataset licenses. All tools, datasets, and libraries must be freely available under open-source or free-tier terms. \\
\hline
\end{longtable}
\endgroup
