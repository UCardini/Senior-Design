---
title: "Ethics and Responsible-Use Statement"
author: Carter Smith, Alex Watts
date: 2026-11-11
due: "2026-11-12"
assignment: "Assignment 11 - Ethics and Responsible-Use Statement"
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

Team: Carter Smith, Alex Watts

## 1. Privacy Summary

This system is a local research pipeline. It has no network connectivity and no user accounts. The only people interacting with the system are the two team members. Source images will be screened for metadata before use; any image containing personal inforomation will be excluded.

\begingroup
\footnotesize
\setlength{\tabcolsep}{4pt}
\begin{longtable}{|L{2.2cm}|L{2.0cm}|L{3.0cm}|L{1.4cm}|L{2.4cm}|L{1.3cm}|L{2.2cm}|}
\hline
\textbf{Data item} & \textbf{Source} & \textbf{Purpose} & \textbf{Consent} & \textbf{Storage} & \textbf{Access} & \textbf{Retention} \\
\hline
\endhead
Image files (source, carrier, stego) &
Team / public image set &
Experiment inputs and intermediates &
N/A &
Local VM &
Team only &
Deleted at project end \\
\hline
\texttt{payload.txt} &
Created by team &
Fixed test message embedded in every carrier &
N/A &
Local VM (gitignored) &
Team only &
Deleted at project end \\
\hline
\texttt{config.json} &
Created by team &
Pipeline settings (quality, threshold, timeout) &
N/A &
Version-controlled in repo &
Team only &
Kept with source code \\
\hline
Pipeline outputs (\texttt{results.csv}, \texttt{summary.json}, manifests, log) &
Pipeline scripts &
Detection scores and run metadata &
N/A &
Local VM; results and summary committed to repo &
Team only &
Results and summary retained as research output; intermediates and log deleted at project end \\
\hline
\end{longtable}
\endgroup

**Removed fields:** None.

## 2. Security Summary

**Secrets in source control:** No secrets exist in this system. The only sensitive item is \texttt{payload.txt}, which is gitignored and has never been committed. No API keys, passwords, or tokens are present. Git history was searched with \texttt{git log -S} for 'password', 'secret', 'api\_key', and 'token'. Only 'secret' returned matches; both commits added Design\_Diagrams files containing the phrase "a secret message" in a diagram legend, not a credential.

**Mechanisms:** File paths passed to external tools are constructed as list arguments to subprocess, not shell strings, to prevent command injection. Image files are validated before being passed to Pillow. File permissions are verified before the pipeline starts.


| Question | Answer |
|---|---|
| Default or hard-coded credentials? | No. No credentials exist in this system. |
| Unauthenticated endpoints that touch user data? | No. No network endpoints exist; all execution is local CLI. |
| Secrets committed to the repository? | No. Git history searched (git log -S) for 'password', 'secret', 'api_key', 'token'. Only 'secret' matched; both hits are diagram legend text, not credentials. |
| HTTPS enforced? | Not applicable. No network communication. |
| Rate limiting or abuse-prevention mechanism? | Not applicable. Single-user local tool. |

## 3. Licensing Summary

\begin{center}
\small
\setlength{\tabcolsep}{4pt}
\begin{tabular}{|L{3.2cm}|L{1.5cm}|L{2.5cm}|L{2.5cm}|L{4.5cm}|}
\hline
\textbf{Name} & \textbf{Version} & \textbf{License (SPDX)} & \textbf{Permissive or Copyleft} & \textbf{Obligation} \\
\hline
Python & 3.11 & PSF-2.0 & Permissive & Retain copyright notice \\
\hline
Pillow & 10.4.0 & HPND & Permissive & Retain copyright notice \\
\hline
pandas & 2.2.3 & BSD-3-Clause & Permissive & Retain copyright notice; no endorsement \\
\hline
NumPy & 1.26.4 & BSD-3-Clause & Permissive & Retain copyright notice; no endorsement \\
\hline
stegano (E1a) & 0.11.3 & GPL-3.0-or-later & Copyleft & Comply; invoked as subprocess only \\
\hline
outguess (E1b) & 0.2.0 & BSD (LICENSE) & Permissive & Retain copyright notice \\
\hline
Aletheia (E2) & 0.3.0 & MIT & Permissive & Retain copyright notice \\
\hline
Source images & N/A & CC0-1.0 & Permissive & None; attribution optional \\
\hline
\end{tabular}
\end{center}

**Project license:** MIT. Compatible with all permissive entries; see Copyleft note below.

**Copyleft:** stegano (E1a) is GPL-3.0-or-later. It is invoked exclusively as a subprocess and is not imported, linked, or distributed with this source code; the GPL does not propagate under this use. E1b and E2 are also invoked as subprocesses. No imported Python dependency is copyleft.


## 4. Attribution List

**Libraries:**

- Python 3.11, https://www.python.org
- Pillow (HPND), https://python-pillow.org
- pandas (BSD-3-Clause), https://pandas.pydata.org
- NumPy (BSD-3-Clause), https://numpy.org

**External tools:**

- stegano (GPL-3.0-or-later), https://github.com/cedricbonhomme/Stegano/tree/master?tab=GPL-3.0-1-ov-file, PNG LSB embedding (E1a)
- outguess (BSD; LICENSE), https://github.com/resurrecting-open-source-projects/outguess/blob/master/LICENSE, JPEG DCT embedding (E1b)
- Aletheia (MIT), https://github.com/cedricbonhomme/Stegano/tree/master, detection (E2)

**Previously considered tools (not used):**

- steghide, considered as E1 but could not be adapted for our needs
- StegExpose, considered as E2 but could not be adapted for our needs

**Adapted code:** None.

**Datasets:** Source images will be drawn from a freely licensed or public-domain image collection. Any image must be confirmed as public domain or free-use with no redistribution restrictions. Where a suitable licensed dataset is unavailable, images may be synthetically generated. Confirmed image licenses are recorded in `NOTICE` in the repo root.

**Assets:** None.

## 5. AI-Use Disclosure

"Generative AI tools may be used for drafting and code assistance but not for the design decisions or written analysis that are assessed in this course."

\begingroup
\footnotesize
\setlength{\tabcolsep}{4pt}
\begin{longtable}{|L{2.2cm}|L{1.8cm}|L{4.2cm}|L{3.0cm}|L{4.0cm}|}
\hline
\textbf{Tool} & \textbf{Used By} & \textbf{Task} & \textbf{Column} & \textbf{How Verified} \\
\hline
\endhead
Claude (Anthropic) &
Carter Smith &
Document drafting, formatting &
Generally OK (drafting assistance) &
Reviewed each section; all component selection, interface definitions, and design decisions confirmed as independently derived \\
\hline
Gemini (Google) &
Alex Watts &
Document drafting, formatting &
Generally OK (drafting assistance) &
Reviewed each section; all component selection, interface definitions, and design decisions confirmed as independently derived \\
\hline
\end{longtable}
\endgroup

## Verification Confirmation

**Carter Smith** - The architecture, data model, component priorities, and design rationale in all assessed deliverables are my own independent work.

**Alex Watts** - The architecture, data model, component priorities, and design rationale in all assessed deliverables are my own independent work.

