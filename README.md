# Master's Thesis - working project

Source for the Master's thesis on the QCar sim-to-real navigation
migration, formatted against the Deggendorf Institute of Technology
(DIT) "Thesis Guideline" (Version 1.0, 27.06.2023).

## How to compile (Overleaf)

1. Overleaf -> **New Project -> Upload Project** -> select
   `qcar-master-thesis.zip` (not a blank project + drag-and-drop,
   which flattens the folder structure).
2. Confirm the main document is
   `Master_Thesis_Report_Amit_Sajeev_22306894.tex` (project menu ->
   Main document; usually auto-detected).
3. Compiler: **pdfLaTeX** (default). The bibliography uses plain
   BibTeX (`\bibliographystyle{plain}` + `\bibliography{...}`), not
   biblatex/biber, and the abbreviations index is a plain table, not
   the `glossaries` package - both deliberately, to keep the compile
   chain short (`pdflatex -> bibtex -> pdflatex -> pdflatex`) instead
   of the much longer `pdflatex -> biber -> makeglossaries -> pdflatex
   -> pdflatex` chain, which is what was tripping the free plan's
   compile-time limit. Overleaf runs `bibtex` automatically; no manual
   step needed beyond a normal Recompile.
4. If you still hit a compile-timeout on the free plan (e.g. after
   adding a lot more content or large figures), the usual next levers
   are: compress/downsize any image you add under `figures/`, or drop
   `microtype` from `preamble.tex` (cosmetic justification polish
   only, not required by the guideline) - both reduce per-pass time
   without changing the document's structure or content.

## What's done vs. what's left

**Done - full structural compliance with the guideline:**
page geometry, font sizes, line spacing, heading style, page
numbering (roman front matter / arabic body), section numbering
depth, figure/table caption style, and a numeric DIN-style
bibliography. All required sections exist: title page, statement of
authorship, index of abbreviations, abstract, introduction,
groundwork, four methodology chapters, results, summary & prospects,
bibliography, appendix.

**Done - substantial real technical content**, pulled directly from
the underlying project's development history (session memory,
READMEs, CHANGELOGs): the Ackermann-drive migration, the Gazebo
Classic/Humble conversion, the four-bug "not moving forward" pattern,
the TCP/JSON hardware bridge design, the LiDAR/odometry/steering-trim
fixes, the full stop-distance investigation (including the active-
braking safety incident and the `readMode=1` HAL buffering root
cause), and the Nav2/MPPI integration and display-lag investigation.

**Done - 14 code listings with explanation**, verbatim from the
actual `qcar_updated` source (URDF/xacro, the behaviour-tree XML, the
onboard bridge and relay Python, and the `EarlyCommitCritic` C++
plugin), each introduced by a paragraph explaining what it does and
why - spread across Chapters 3-6, one or two per major fix discussed
in the prose. `\usepackage{listings}` (already in `preamble.tex`)
renders these with syntax highlighting and line numbers; no extra
Overleaf configuration needed.

**TODO - search every chapter file for the literal string `TODO`**
(`grep -rn TODO .` from this directory) - each marks something only
you can fill in or decide:

- Title page fields (`main.tex`): title, programme, supervisor,
  matriculation number, and submission date are filled in. Only the
  exact degree title (M.Eng./M.Sc./etc.) is still a TODO - confirm
  with the Centre for Studies.
- Acknowledgement (optional, `frontmatter/acknowledgement.tex`) - not
  currently included in the build; uncomment its `\input` line in
  `main.tex` if you want one.
- Statement of Authorship: sign the exported/printed copy by hand.
- Two measurements flagged in Chapters 6-7 as taken *before* the
  final `readMode=0` fix (Section 6.5 / 7.3) - re-running
  `monitor_goal_timing.py` and the six-point sim-vs-real sweep
  post-fix, if you can still get QCar lab time, would let those
  sections report final rather than provisional numbers.
- Bibliography (`bibliography.bib`): verify every entry's exact
  volume/issue/page numbers against the publisher before submission.
- Figures: several `% TODO: add figure` comments mark where a
  diagram, screenshot, or plot would strengthen a chapter - none of
  the current content depends on them being added, they're additive.
- Appendix source-code/data availability pointer (`appendix/appendix.tex`).

## Page count

Guideline target for a Master's thesis: ~80 text pages (±10%),
excluding TOC/bibliography/appendix. Starting from an earlier ~50-page
draft, every chapter was substantially deepened in one pass: Chapter 2
gained full derivations/mechanism explanations for DDS discovery and
QoS compatibility, the true two-wheel Ackermann angle (matching the
actual Gazebo plugin source, not just the single-track approximation),
PID control theory, the Coulomb/friction-cone model, the full MPPI
softmax-weighting math, and the full AMCL particle-filter + augmented-MCL
recovery math; Chapters 3-6 each gained "how the code actually works"
walkthroughs for every bug/fix already introduced - worked numerical
traces, a formal ring-buffer lag derivation, a formal discrete-time
instability proof for the rejected active-braking controller, and
more. Body text alone is now roughly 23,000 words (up from ~15,500) -
compile it and check the actual page count, since code listings,
equations, and tables render less densely than prose and a words/page
heuristic will underestimate them.

**One real correctness fix made during this expansion, not just
additions:** reading the actual `gazebo_ros_ackermann_drive.cpp`
source (Chapter 3, Section "A Quiet Regression") found that the
final simulation drive plugin's `/odom` is computed from
`model_->WorldPose()`/`WorldLinearVel()` - literal Gazebo ground
truth - not the genuine wheel-encoder dead reckoning the thesis
originally (and incorrectly) claimed for it. Every place that claim
appeared (Chapter 3's second-migration section and summary, Chapter
8's summary) has been corrected; only the physical QCar's own
odometry (Chapter 4) is actually dead reckoning. This is now presented
as a real finding (a structural, not just measured, source of the
sim-to-real localisation gap) rather than silently patched over.

## Source reference

`docs/guideline-source.md` (in this directory) has a page-by-page
summary of the DIT guideline this draft was built against, for quick
reference without re-opening the original PDF.
