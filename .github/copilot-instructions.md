# MillPresenter AI Instructions

## Project Overview
MillPresenter is a Windows desktop application for analyzing and presenting grinding mill videos. It performs a **one-time detection pass** over a 1080p@59.94 fps clip, caches results, and then supports **real-time playback** with instant per-size overlays (4/6/8/10 mm). The app is **presentation-first**: smooth playback, simple toggles, and visually trustworthy overlays matter more than perfect detection.

## Documentation & Maintenance Rules
You must proactively maintain the project documentation. **Do not wait for the user to ask.**

### 1. Plan & Roadmap (`PLAN.md`)
- **Update Immediately**: Mark tasks as completed (`[x]`) as you finish them.
- **Link to Tests**: Every completed task must cite the test file that verifies it (e.g., `*Verified by*: tests/test_xyz.py`).
- **Granularity**: Break large tasks into small, verifiable sub-tasks. This ensures we can track progress on specific edge cases and accuracy improvements.
- **Scope**: Add new tasks if the scope expands.

### 2. Testing Criteria (`docs/testing_criteria.md`)
- **Update Trigger**: Must be updated after every test run or new test suite.
- **User Alignment**: Before writing a new test suite, **ask the user** to confirm the test plan/criteria. This ensures alignment on what "Success" looks like.
- **Content**: Use plain language to explain **What** (functionality/edge case), **Why** (risk/problem), and **How** (solution).
- **Accuracy Focus**: Explicitly cover edge cases (glare, occlusion, hollow rings) to ensure high accuracy.
- **Format**: Keep entries chronological with short headings.

### 3. Architecture & Decisions
- **`docs/design_decisions.md`**: Record the "Why" behind architectural choices (e.g., "Why NVDEC", "Why JSONL").
- **`docs/technical_primer.md`**: Keep module descriptions synced with code.

### 4. FAQ & Knowledge Capture (`docs/faq.md`)
- **Capture Questions**: Whenever the user asks "Why?" or "How does this work?", or when a non-obvious decision is made during conversation, **immediately** add it to the FAQ.
- **Context**: Include the context of the question (e.g., "During CLI testing...").
- **Goal**: Build a knowledge base for future contributors so they don't have to re-ask the same questions.

**Philosophy**: Code explains *how*, documentation explains *why*. Documentation updates are **part of the commit**, not an afterthought.

## Architecture & Data Flow
- **Separation of Concerns**: Detection is fully decoupled from rendering and playback.
  - Detection: **one offline/pre-playback pass**, writes detections to `detections.jsonl`.
  - Playback: reads from cache only; **toggling classes never re-runs detection**.
- **Pipeline**: Video Source -> `FrameLoader` (PyAV, NVDEC if available) -> `ProcessorOrchestrator` -> `ResultsCache` (JSONL + RAM ring buffer) -> `OverlayRenderer` (used by **UI Player** and **Exporter**).
- **Tech Stack**: Python, PyQt/PySide (`QOpenGLWidget` + `QPainter`), OpenCV, PyAV/FFmpeg.

## Critical Implementation Details
- **Performance Budget**: ≈16.7 ms per frame at 59.94 fps.
  - Decode: prefer NVDEC; fallback to CPU.
  - Rendering: `QOpenGLWidget` + `QPainter`, **reuse pens/brushes**, avoid per-frame allocations.
  - On overload: **skip overlay drawing, never drop decoded video frames**.
- **Video Ingestion (Raw .MOV Support)**:
  - **Rotation**: `FrameLoader` must respect metadata rotation flags (e.g., iPhone/Nikon `Rotate: 90`) before processing/ROI.
  - **Interlacing**: Detect 1080i; apply deinterlacing if needed to prevent "combing" artifacts on moving beads.
- **Caching Strategy**:
  - Disk: `detections.jsonl` (one JSON object per frame: `FrameDetections`).
  - Runtime: **ring buffer ≈240 frames (~4 s)** in RAM for smooth scrubbing.
  - Playback and export must consume detections **only through this cache**.

## Vision Pipeline (Detection Engine)
- **Preprocessing**: grayscale -> **bilateral filter** (preserve edges, tame noise/glare) -> **CLAHE** (local contrast; keep ring edges visible under uneven lighting).
- **Dual-Path Proposals**:
  - `HoughCircles`: robust on dense bead piles and partially occluded rings.
  - `ContourFilter`: Canny + morphology -> circularity, solidity, minEnclosingCircle fit.
- **Annulus Handling (CRITICAL)**:
  - Beads are **hollow rings**, not solid disks.
  - Always use **outer diameter** for size; inner hole must not become a “small bead.”
  - Guard explicitly against misclassifying a 10 mm bead’s hole as a 4 mm bead.
  - **Calibration Safety**: Calibration tools must share this logic—if a user clicks a hole, snap to the outer ring automatically.
- **NMS / Fusion**:
  - Merge overlapping proposals from Hough + Contours using IoU + center distance.
  - Exactly one `Ball` per physical bead per frame.
- **Classification**:
  - Convert radius to mm via calibrated `px_per_mm`.
  - Assign bins 4/6/8/10 mm by configurable ranges; store `cls` and `conf`.
  - Detection is **best-effort for convincing overlays**, not metrology-grade.

## ROI, Clutter, and Glare
- **ROI Mask (`roi_mask.png`)**:
  - Mandatory: white = valid, black = ignore.
  - Discard any detection whose center lies on a black pixel.
  - Use to aggressively remove bolts, flanges, gear teeth, wheels, and frame edges.
- **Static Background**:
  - Favor edge/ring evidence over filled areas; down-weight static, non-moving structures.
- **Specular Highlights**:
  - Tune bilateral + CLAHE so ring contours stay continuous while highlights are softened.
  - Avoid thresholds/morphology that fragment the circular rim into arcs.

## OverlayRenderer & Playback
- **Shared Renderer Pattern**:
  - One `OverlayRenderer` module, used by:
    - UI Player (`QOpenGLWidget` + `QPainter`).
    - Exporter (drawing onto frames for MP4).
  - Do not duplicate overlay logic in UI or exporter; they should call into `overlay.py`.
- **Toggles**:
  - Size toggles (4/6/8/10 mm) **only** decide which cached `Ball`s are drawn.
  - No new detection, no reclassification, no extra disk I/O when toggling.
- **No Tracking Requirement**:
  - Per-frame detections are enough; **no bead identity tracking over time**.
  - Any optional smoothing must not break scrubbing or introduce long-lived state.

## Project Structure & Conventions
- **Entry Point**: `src/mill_presenter/app.py`.
- **Core Modules** (names may vary; roles must not):
  - `playback.py`: decode via PyAV/NVDEC; drive frame clock.
  - `orchestrator.py` / `processor.py`: run the one-time detection pipeline.
  - `cache.py`: JSONL writer/reader + RAM ring buffer.
  - `overlay.py`: **only place** that turns detections + toggles into drawn primitives.
  - `exporter.py`: reuses `overlay.py` for annotated exports; never runs detection.
- **Config & Paths**:
  - Use `configs/*.yaml` for calibration, bins, performance settings.
  - Use `utils/paths.py` for resource resolution; **no hard-coded absolute paths**.
  - Respect `.env` overrides (e.g., `MP_DECODE_MODE`, preview downscale, overlay mode).

## Data Contracts
- **`Ball`**: `x:int`, `y:int`, `r_px:float`, `diameter_mm:float`, `cls:int`, `conf:float`.
- **`FrameDetections`**: `frame_id:int`, `timestamp:float`, `balls:list[Ball]`.
- **`detections.jsonl`**: exactly one JSON object per frame; frame IDs align with decode order.

## Common Pitfalls to Avoid
- Never re-run detection because of:
  - Overlay toggles, seeking, pausing/resuming, or export.
- Never ignore `roi_mask.png` (or treat black as valid area).
- Never classify using inner ring radius when the outer rim is visible.
- Do not block the UI thread with anything beyond lightweight overlay drawing.
- Do not add heavyweight per-frame tracking that complicates scrubbing or cache semantics.

## Visual Acceptance Checklist (Main Video Frames)
- Each physical bead has **one overlay circle**; no double-drawing from Hough+Contours duplicates.
- Large annular beads **do not** show an extra “inner small bead” where the hole is.
- Overlays stay strictly **inside the valid mill region**; no circles on bolts, flange, wheels, or outside the drum.
- In dense piles, circles are reasonably centered and not exploding into many tiny detections; some overlap is acceptable, but not cluttered noise.
- Under glare, circles follow the outer metallic ring, not random highlight streaks or reflections.
- Toggling 4/6/8/10 mm classes changes only which circles are drawn; video motion remains smooth, with **no noticeable lag or recomputation** on toggles.
