# Erm – Build Plan

Nov 3, 2026 · Based on *Erm – Design Document* (Draft v0.1, Nov 2, 2026) · Status: Draft v0.1

This plan turns the design document into an ordered set of work, with exit criteria for each milestone. Section numbers like "§5" refer to the design document. The size estimates are rough guesses for one developer working full time. Scale them to your actual hours.

---

## 1. Approach

1. **Prove the risky parts first.** M0 measures the two things that could change the stack: render and turn speed on the heaviest scanned score, and pen pressure on the actual stand computer. Nothing else gets built until M0 passes.
2. **Fix the contracts before the features.** `PageGeometry`, `CommandBus` and the turn guard, the `Renderer` interface and annotation file schema v1 are written and unit-tested at the start of M1. Every later feature depends on them, and changing them later would break saved annotations.
3. **Measure all the time.** A latency harness (input timestamp to frame on screen) and a memory readout are built in M1 and kept in debug builds. Every milestone's exit criteria include the §9 numbers.
4. **Package from M1.** The Inno Setup installer runs in CI from the first viewer build, so the 30 MB install limit is checked from the start instead of being discovered at the end.
5. **Keep it usable from M1 on.** After each milestone you should be able to play from the app. M1 already replaces a basic PDF reader with a pedal.

---

## 2. Pre-work (before M0, about 2 days)

| Task | Notes |
| --- | --- |
| **Move the project to the Windows file system** | This folder is currently at `/home/natet/erm-pdf` in WSL. §8 says to build Windows code from the Windows side. Suggested location: `C:\dev\erm`, reached from WSL as `/mnt/c/dev/erm` for scripts only. |
| `git init`, `.gitignore`, `.gitattributes` | LFS (or an ignored folder) for large test PDFs. Never commit copyrighted scores. |
| Toolchain on Windows | MSVC 2022, CMake ≥ 3.28, Ninja, Qt 6.8 LTS (via the Qt online installer or `aqtinstall`) with the **Qt PDF**, Qt WebSockets, Qt SerialPort and Qt Quick modules. Python 3.12 with PySide6 for M0. |
| **Test corpus** (`tests/fixtures/`) | A `tools/make_fixtures.py` script (pikepdf + reportlab) generates: mixed page sizes, `/Rotate` 90/180/270, a CropBox whose origin isn't (0, 0) and that differs from the MediaBox, a password-protected file, a damaged file (truncated, bad xref), a file with existing Ink/FreeText annotations, blank pages, a 200-page 600 DPI colour scan (synthetic noise images), and a landscape fold-out inside a portrait file. Add your real heaviest score locally, outside git. |
| Target device inventory | Write down the stand computer's model, screen resolution and scaling, pen type (Windows Ink / Wacom AES / MPP), RAM and GPU. M0 results only count when measured on that machine. |

---

## 3. Repository layout

```
erm/
  CMakeLists.txt, CMakePresets.json      # presets: win-debug (ASan + clang-tidy), win-release
  src/
    core/        PageGeometry, CommandBus, TurnGuard, Command, Settings
    render/      Renderer (interface), QtPdfRenderer, [PdfiumRenderer], PageCache
    annot/       AnnotationModel, AnnotationStore (annotation file I/O), AnnotationLibrary (central store + index), undo commands
    input/       InputService, Binding, BindingStore, sources: Key, Mouse, Wheel, Serial, Remote
    remote/      RemoteBridge (HTTP + WebSocket), Pairing, QR (qrcodegen)
    platform/    ScreenAwake (interface + win32), FileAccess (interface + desktop)
    pdftools/    library (merge, split, extractPages, imagesToPdf) + erm-pdftools CLI
    app/         main.cpp, QML type registration, ImageProvider / PageItem
  qml/           Viewer, Toolbar, Settings, InputMapping, Tools, Setlist screens
  web/remote/    index.html, remote.js, remote.css    # phone page, embedded via .qrc
  firmware/pico/ boot.py, code.py                     # CircuitPython HID controller
  prototype/     M0 PySide6 spike (frozen after M0)
  tests/cpp/     Qt Test unit tests
  tests/py/      pytest: CLI, annotation file schema, WebSocket protocol, end-to-end
  tests/fixtures/
  tools/         make_fixtures.py, bench/ (latency and memory scripts)
  packaging/     erm.iss (Inno Setup), windeployqt script
```

Dependency rule: `core` depends on nothing but QtCore. `render`, `annot`, `input`, `remote` and `pdftools` depend on `core` and not on each other. Only `app` and `qml` know about the UI. This keeps the command bus testable without a window (§4, "Command bus").

---

## 4. Contracts settled at the start of M1

These are written first, reviewed, and covered by unit tests before any feature uses them.

### 4.1 `PageGeometry`
- Built per page from the CropBox, the page's `/Rotate` and the CropBox origin.
- `pageToScreen` / `screenToPage` for a given fit mode (fit page, same zoom), user crop rect, two-up slot and device pixel ratio.
- Unit tests: a round trip within 0.01 pt for every fixture page, at all four rotations, with a non-zero origin, and with crop.
- **M0 must check first:** whether `QPdfDocument::pagePointSize()` reports the CropBox and applies `/Rotate`. If it doesn't, PageGeometry reads the boxes itself (via qpdf, which M6 needs anyway, or the PDFium fallback).

### 4.2 `Command`, `CommandBus`, `TurnGuard`
- `Command { type: Next | Prev | NextHalf | Jump(id) | ToggleLayer(id) | StageMode | Goto(page); source; timestamp; seq? }`
- The `CommandBus::command(Command)` signal is the only way into the viewer.
- `TurnGuard` sits inside the bus: it drops Next/Prev/NextHalf commands within the lockout window (default 300 ms, configurable), and drops auto-repeat. Remote `seq` de-duplication also happens here.
- Every command carries the original input event timestamp, so the latency harness can measure the full path from input to screen.

### 4.3 `Renderer`
```cpp
class Renderer : public QObject {               // lives on the single render thread
  virtual int pageCount() const = 0;
  virtual PageInfo pageInfo(int page) const = 0; // boxes, rotation
  virtual void requestRender(RenderRequest r) = 0; // page, target px size, flags (annots on/off)
signals: void rendered(RenderResult);            // QImage + request id
};
```
`QtPdfRenderer` comes first. A `PdfiumRenderer` behind the same interface is built only if M0 or M1 shows it's needed.

### 4.4 Annotation storage and schema v1

**Decided Nov 3:** annotations are kept in **one central annotation library**, not in sidecar files next to each PDF. This replaces design doc §6 ("sidecar next to the PDF") and uses the same design §8 already planned for tablets.

```
Documents\Erm\Annotations\       default location; change it in Settings → Storage
  index.json                     cache: hash → last path, file name, page count, page sizes, last opened
  <sha256>.json                  one file per score: annotations, layers, bookmarks, view settings
  <sha256>.json.1 … .5           the last 5 versions (backups)
```
- **Documents, not AppData,** so you can see the files and normal backups (and OneDrive, if Documents is synced) include them. The location comes from `QStandardPaths::DocumentsLocation`, so it follows a redirected Documents folder.
- **The index can always be rebuilt.** Each `<sha256>.json` also records the PDF's last path, file name, page count and page sizes, so a missing or broken `index.json` is regenerated by scanning the folder.
- **One storage mode only.** There's no "keep it next to the PDF" option, which would double the testing.
- **The uninstaller never deletes the library.**
- **Synced folders:** if the atomic rename fails because OneDrive or antivirus has the file locked, retry with backoff and keep the edits in memory. Never drop them.

Write the schema down (`docs/annotations-v1.md` plus a JSON Schema file used by pytest) before M2. The design doc's draft (§6) leaves these open. Proposed answers:
- Page numbers are **1-based** (matching the draft example and what musicians say).
- `width` is in PDF points. `points` are `[x, y, pressure]` in PageGeometry page space.
- Add a `highlighter` type (or `ink` with `"blend": "multiply"`), `text` (`at`, `text`, `size`, `color`) and `eraser` (no stored item: partial erase splits a stroke into new items).
- Add a `pdf` block with the hash, last path, file name, page count and page sizes, so the index can be rebuilt and the reattach flow works.
- Add a `view` section for settings that belong to the score (decided Nov 3, §9.5): per-page crop rects, colour inversion, fold-out handling, two-up "cover alone", skip blank pages. Device preferences (dark/sepia page mode, fit mode, tap zones) stay in app settings (QSettings).
- Backups: keep the last 5 versions as `<sha256>.json.1` … `.5` (Risks table), rotated on the first save of each session rather than on every stroke.

---

## 5. Milestones

### M0 – PySide6 spike (1–2 weeks, from §9)

**Goal:** a go/no-go on Qt PDF and on pen input, measured on the target machine.

Tasks
1. A minimal QML viewer (QML, not widgets, so the measurements carry over): `QPdfDocument` on a worker thread, an image provider, a 4-page cache (current, +1, +2, −1), PageDown/PageUp turning.
2. Timing: cold open to first page, render time per page at screen resolution, turn latency (key event timestamp to `QQuickWindow::frameSwapped`), and peak memory with the heaviest file.
3. Pen test: a `PointHandler` (accepted devices: Stylus) that draws a pressure-scaled stroke, while a `TapHandler` (Finger | Mouse) turns pages. Check pressure range, hover, and whether palm contact triggers turns.
4. Check Qt PDF details: CropBox/rotation reporting (§4.1), the render flag for showing or hiding existing annotations, behaviour with the password and damaged fixtures.
5. If Qt PDF misses the targets: repeat steps 1–2 with pypdfium2.

Exit criteria (all on the target device, using the heaviest real score)
- Cached turn ≤ 50 ms at the 95th percentile, ≤ 100 ms worst case.
- Cold open to first page < 2 s for a 50-page scan.
- Memory < 300 MB with a 100 MB PDF open.
- Pressure values vary smoothly from 0 to 1. Finger and palm never draw, and the pen never turns pages.
- A one-page write-up in `prototype/RESULTS.md`: the numbers, the chosen renderer, and any surprises.

Decision point: pass → C++ from M1. Qt PDF fails but PDFium passes → C++ with `PdfiumRenderer`. Pen fails → stop and look at the Windows Ink / Wintab settings before going any further, because F4 is a Must.

---

### M1 – Viewer (≈ 3 weeks)

**Goal:** a C++ app you could use at a rehearsal with a pedal.

Tasks
1. CMake skeleton, presets, CI (§8 of this plan), an empty QML window, and an Inno Setup installer built in CI.
2. The §4 contracts: `PageGeometry`, `CommandBus` + `TurnGuard`, `Renderer`/`QtPdfRenderer` on a dedicated `QThread`.
3. `PageCache`: current, +1, +2 and −1, each rendered at its own target size (never above screen resolution, §5 "Mixed page sizes"). It's cancellable, re-prioritised when you turn, and re-renders on resize, rotation or a DPI change, showing a scaled copy of the old image until the new one is ready (§11).
4. Page display: start with `QQuickImageProvider`. If the measured turn includes a slow texture upload, switch to a custom `PageItem` (a `QQuickItem` that holds pre-uploaded `QSGTexture`s for the cached pages), so a turn really is just swapping a pointer.
5. Fit modes: fit page (default) and keep same zoom. Single-page display only. Two-up and half-page come in M5.
6. Tap zones (F2): the default 1/3 and 2/3 split, configurable including top and bottom zones, `TapHandler` for finger and mouse only.
7. Keyboard input: PageDown/PageUp, arrow keys and Space mapped through `InputService` with a hard-coded default binding table. An app-wide event filter that ignores `isAutoRepeat()`.
8. Edge flash on the first and last page (§11).
9. F1: open a file or folder, a recent-files list (stored with QSettings), reopen at the last page.
10. Error handling: password prompt (never stored), damaged file opens whatever pages render, never crash.
11. A debug overlay (toggled with a key) showing the last turn latency, the cache state and the process memory.
12. *(Deferred, §9.10)* A **page navigator** (thumbnail strip or go-to-page). This isn't in the design doc, but you need some way to reach page 37. Thumbnails are generated lazily at low priority.

Exit criteria
- PageGeometry unit tests pass on every fixture.
- Turn latency meets the M0 numbers in the C++ build, through both the image-provider and the tap paths.
- A held key turns once. Two presses 150 ms apart turn once, and 400 ms apart turn twice (automated test through `CommandBus`).
- Installer size is recorded in CI. If it's over 30 MB, fix it now (trim plugins, skip software OpenGL, check the `windeployqt` flags).
- Rotating a 2-in-1 mid-piece keeps the same page.

---

### M2 – Annotations (≈ 4.5 weeks, split in two)

The design doc lists pen, highlighter, eraser, sidecar and undo for M2, but F4 (a Must) also includes **text and music stamps**, and F5 layers aren't assigned to any milestone. This plan adds them as M2b.

**M2a – Ink, storage, annotation mode (≈ 3 weeks)**
1. `AnnotationModel` (items per page, layer ids) and an annotation file schema v1 reader and writer, validated against the JSON Schema in pytest.
2. `AnnotationLibrary` (§4.4): the central store in `Documents\Erm\Annotations`, `index.json` with rebuild-on-scan, a configurable location (moving it copies, verifies and then switches), and atomic writes with retry. The same code is used for tablets later (§8).
3. Content hash (§6). Computed on open, off the UI thread.
4. **Annotation mode** (decided Nov 3, §9.6). You enter and leave it with a dedicated toolbar button or a mapped input, never by accident. Inside annotation mode:
   - **Tap zones are off.** A finger pans, pinch-zooms and selects. The pen draws, erases and places items. The pen never turns pages.
   - **Page turns** come only from an **edge swipe** or the **dedicated Next/Prev buttons** on the annotation toolbar. External inputs (pedal, keys, mouse buttons, phone remote) keep turning pages, so you can practise and mark up without leaving the mode (§2, "Practice").
   - **Edge swipe vs. pan:** a swipe only counts as a turn if it starts in the outer edge strip (about 8% of the width), moves mostly horizontally, and travels more than about 25% of the width. Any other one-finger drag pans or selects. If the page is zoomed in and not yet at its edge, the edge swipe pans first. All the thresholds are settings, and the swipe goes through `CommandBus` and `TurnGuard` like any other input.
   - Outside annotation mode, the pen behaves like a finger (taps turn pages) and nothing can be drawn.
5. **Zoom and pan** (needed for precise marking): pinch-zoom up to about 4× around the fingers, with a higher-resolution render of the visible area on a lower-priority render queue. Zoom resets to fit when you turn the page. The page cache and the turn path are unaffected.
6. Live-stroke rendering: the stroke in progress is drawn on its own lightweight item. Finished strokes for each visible page are cached in a per-page overlay image, redrawn only when that page changes, so a turn doesn't repaint every stroke. Ink width varies with pressure. Highlighter uses multiply blending.
7. Eraser: whole-stroke erase, and partial erase that splits a stroke.
8. Undo and redo per page (`QUndoStack` per page inside a `QUndoGroup`).
9. Autosave: debounced (about 500 ms after a stroke ends) through `QSaveFile`, plus backup rotation. Flush on close and when the page changes.
10. Hash mismatch flow (§11): when a PDF opens with an unknown hash, look in the index for an entry with the same last path or file name. If its page count and sizes match, ask whether to reattach (this re-keys the annotation file to the new hash). Otherwise leave the old annotations in the library and say they're there. Never delete them.
11. "Back up all annotations": a zip of the whole library to a location you choose.

**M2b – Text, stamps, layers (≈ 1.5 weeks)**
1. Text tool (a tap places it, with an inline editor).
2. Stamp palette: Bravura loaded with `QFontDatabase` and SMuFL glyph names mapped to codepoints from the SMuFL `glyphnames.json`. Fingerings 1–5, pp–ff, breath mark, up-bow/down-bow, accidentals, fermata, glasses (Bravura has no "glasses" glyph, so use a small SVG for it).
3. Select, move, scale and delete for stamps and text.
4. Layers (F5): create, rename, show or hide, choose the active layer. Existing PDF annotations appear as a read-only pseudo-layer that can be hidden (§11).

Exit criteria
- Annotations line up within 1 px after zoom, crop, rotating the device, moving to another monitor and reopening on a different screen resolution (an automated test through PageGeometry, plus a manual check).
- Killing the process mid-stroke loses at most the stroke in progress, and the annotation file always parses (a pytest kill-loop test).
- Drawing doesn't change turn latency with 500 strokes on the page.
- Pen pressure and palm rejection work on the target device. In annotation mode, finger taps never turn the page, while edge swipes, the toolbar buttons and the pedal always do (automated gesture tests plus a manual check).
- Zooming in and panning on a 600 DPI scan stays smooth and doesn't slow the next turn.

---

### M3 – Phone remote (≈ 1.5 weeks)

1. `RemoteBridge`: a `QWebSocketServer` and a minimal static HTTP server for the phone page (see §9 of this plan, question 1, about QHttpServer licensing), bound to the LAN interface the user chooses.
2. **Interface selection (under review, §9.13):** this laptop has Wi-Fi plus Tailscale and NordVPN adapters, so automatic choice is unreliable. Rank physical adapters first, show the IP, let the user change it, and listen only on that adapter.
3. Pairing: a one-time token in the QR URL (qrcodegen), exchanged for a session token. One controller at a time (decision of Nov 2), and a new pairing replaces the old one.
4. Protocol (§7): `{"cmd","seq"}` → `{"ack","page","of"}`. `seq` de-duplication is per session and survives reconnects. The server sends page changes to the phone even when the turn came from another input.
5. Phone page (vanilla JS, no build step): large Next and Prev buttons that work one-handed and when propped on the stand, optional Half and Jump, page number, a small thumbnail, automatic reconnect with a visible connection state.
6. Keeping the phone awake: the Wake Lock API **only works in secure contexts (HTTPS)**, and a LAN `http://` page isn't one. **Decision deferred** (§9.2). Decide before M3 starts. The working proposal is the hidden looping-video fallback (the NoSleep.js approach), trying the Wake Lock API first in case the browser allows it.
7. Windows Firewall: the installer adds an inbound rule for private networks only. In the app, detect a "Public" network profile and explain why the phone can't connect.

Exit criteria
- Phone to screen in < 100 ms on home Wi-Fi (measured with the overlay, using the phone's send timestamp corrected for clock offset).
- A retry storm (the same seq sent 10 times) turns once (pytest WebSocket test).
- A second phone is rejected while the first is paired.
- The server is off by default and listens only after the user turns it on.

---

### M4 – Switches and input mapping (≈ 1.5 weeks)

1. `BindingStore`: many bindings per command, saved in settings. Binding types: key (with modifiers), mouse button (including `Qt::BackButton` and `Qt::ForwardButton`), wheel up/down (one notch = one command, accumulating the delta), serial line, remote command.
2. Input mapping screen (F8) with learn mode ("Assign Next", then wait for any source). Conflict warnings.
3. Bluetooth mouse in stage mode: clicks are captured anywhere on the window.
4. Pico firmware: `firmware/pico/code.py` from §7, plus a `boot.py` that turns off the CIRCUITPY drive and serial console in "performance" mode (a jumper or a held button at boot re-enables them for editing). Wiring guide with the TRS pinout.
5. Optional serial channel: `QSerialPort`, auto-detecting the Pico by VID/PID, line protocol `NEXT`/`PREV`/`HALF`, sending `PAGE n/m` back for an LED or display. Off by default.
6. **Basic stage mode early:** `ScreenAwake` (`SetThreadExecutionState(ES_CONTINUOUS | ES_DISPLAY_REQUIRED)` from the main thread) and touch locked outside the turn zones. F10 is a Must and costs little, so it shouldn't wait until M5. The full stage mode is finished in M5.
7. Document known commercial pedal modes (AirTurn, PageFlip, Donner) in the help.

Exit criteria
- Every source in §7's table turns pages through the same `CommandBus` path.
- Learn mode can bind each source type.
- A bouncing switch (simulated at 5 ms intervals) turns once.

---

### M5 – Performance features (≈ 4 weeks)

1. **Stage mode (F10), complete:** hide all chrome, 2 s long press to exit, restore the same page after sleep or wake. Windows press-and-hold and edge gestures can interfere (deferred, §9.12).
2. **Half-page turn (F3):** the split happens in screen space after scaling (§5). The first press shows a composite (the top half of page n+1 over the bottom half of page n). The second press completes the turn. Works with mixed page sizes and crop.
3. **Two-up (F3):** same-height scaling, pair centering, "cover alone", advance by one or two pages.
4. **Crop (F11):** auto-detect margins at low resolution (threshold plus a bounding box of non-white pixels), adjustable handles, stored per page in the annotation file's `view` section.
5. **Jump links and bookmarks (F6):** a jump rect is drawn in annotation mode and fired by a tap or a `Jump(id)` binding. Bookmarks are listed in the navigator.
6. **Setlists (F7):** a setlist file (format deferred, §9.11; proposal: a `.ermset` JSON file (an ordered list of `{path, hash, pageRange?}`)), crossing from the last page into the next item, preloading the first pages of the next item, and recovering from missing files without crashing mid-setlist (§11).
7. **Readability:** dark and sepia page modes, per-file colour inversion, skip blank pages, fold-out handling (§11).
8. **Export (F12, Could):** a flattened copy, with annotations written as vector paths and text into a new content stream with qpdf (no re-rendering of the scan). Standard Ink/FreeText annotation export is optional.
9. **Moving and sharing annotations:** "Export annotations" writes a small `.ermannot` file (the score's annotation file, optionally only some layers) to send to a teacher or carry to another computer. "Import annotations" attaches one to an open PDF, checking the hash and page sizes. "Export score with annotations" bundles the PDF and its annotations in one file.
10. **Library housekeeping:** a screen listing "annotations for missing files", where you can reattach them to a PDF or delete them.

Exit criteria
- A full mock gig: a 3-piece setlist, stage mode, pedal + phone + tap, 30 minutes, no wrong or missed turns, the screen never sleeps.
- Half-page and two-up are correct on every mixed-size fixture.

---

### M6 – PDF tools (≈ 2.5 weeks)

1. A `pdftools` library and the `erm-pdftools` CLI: `merge`, `split` (at pages / every N / at bookmarks), `extract`, `reorder`, `rotate`, `delete`, `images2pdf`.
2. **Use qpdf for images to PDF too** (§9 of this plan, question 3): JPEGs are written as-is as `/DCTDecode` image streams (no re-encode), PNGs are decoded with Qt and written as Flate streams, and EXIF orientation is applied through the page's `/Rotate` or matrix. Page size: fit to image, A4 or Letter.
3. Annotations follow their pages: the output PDF gets its own annotation file in the library, remapped (page renumbering, rotation transform, dropped pages), with pytest checks through the CLI.
4. Tools screen: worker thread, progress bar, Cancel, safe output names, password prompt.
5. Merging a setlist into one PDF (connects to F7).

Exit criteria
- Byte-level check that JPEGs are embedded unchanged.
- Round trips: split then merge gives the same page count, sizes and annotation positions as the original.
- The original file is never modified (a pytest check on mtime and hash).

---

## 6. Requirements traceability

| Req | Milestone | | Req | Milestone |
| --- | --- | --- | --- | --- |
| F1 Open, recent | M1 | | F9 Phone remote | M3 |
| F2 Tap zones | M1 | | F10 Stage mode | M4 (basic), M5 (full) |
| F3 Single / two-up / half | M1 / M5 / M5 | | F11 Crop | M5 |
| F4 Pen, highlighter, eraser | M2a | | F12 Export flattened | M5 |
| F4 Text, stamps | M2b | | F13 Images to PDF | M6 |
| F5 Layers | M2b | | F14 Merge | M6 |
| F6 Bookmarks, jumps | M5 | | F15 Split, page edits | M6 |
| F7 Setlists | M5 | | Dark/sepia (NFR) | M5 |
| F8 Mapping + learn | M4 | | Install < 30 MB (NFR) | checked from M1 |

---

## 7. Test strategy

- **C++ unit tests (Qt Test):** PageGeometry, TurnGuard, CommandBus, BindingStore, AnnotationModel and undo, annotation file I/O, library index rebuild, page remapping. Run on every commit.
- **Python black-box tests (pytest)** (§4 says Python stays for tests): annotation file JSON Schema, `erm-pdftools` CLI, WebSocket protocol (pairing, seq dedupe, single controller), kill-during-save.
- **Performance tests:** `tools/bench/` opens fixtures through a `--bench` flag in the app that scripts turns and prints JSON (latency percentiles, peak working set). CI tracks the trend on its runner. Pass/fail only counts on the target device, run before each milestone ends.
- **Manual device checklist** per milestone: pen, palm, rotation, multi-monitor, sleep/wake, real pedals.
- **Debug builds** use ASan and clang-tidy (§4 engineering rules). MSVC supports `/fsanitize=address`. clang-tidy runs through the CMake preset.

## 8. CI and packaging

- GitHub Actions `windows-latest` runner: install Qt with `jurplel/install-qt-action`, configure and build both presets, run Qt Test and pytest, run `windeployqt` and Inno Setup, upload the installer, and record its size in the job summary.
- Link Qt as shared libraries (LGPL, Risks table). Ship the licence texts for Qt, qpdf (Apache-2.0), qrcodegen (MIT) and Bravura (OFL) in the installer.
- Version scheme: `0.<milestone>.<n>` until v1.

---

## 9. Problems found in the design document

Status as of Nov 3, 2026: **Accepted** = the proposed fix is adopted in this plan. **Decided** = the user chose an approach. **Deferred** = the note is kept and the decision is postponed. **Under review** = still being discussed.

These need a decision, ideally before the milestone shown.

1. **Accepted.** **QHttpServer licensing (before M3).** Qt HTTP Server is offered under GPLv3 or a commercial licence, not the LGPL. That conflicts with the plan to stay LGPL by linking Qt dynamically. Recommendation: serve the 3 static files of the phone page with a small HTTP handler on `QTcpServer` (about 150 lines), and keep `QWebSocketServer` (LGPL) for the protocol. Check the current licence terms before you decide.
2. **Deferred, decide before M3.** **Wake Lock on the phone (M3).** The Wake Lock API needs HTTPS, and a LAN page is plain HTTP. The working proposal is the hidden-video fallback (M3 task 6). Self-signed HTTPS would cause certificate warnings on every phone.
3. **Accepted.** **PDFium for images to PDF (M6).** The `FPDFImageObj_*` APIs in §10 aren't exposed by Qt PDF. Using them means shipping a second, standalone copy of PDFium. qpdf, which is already a dependency, can embed JPEGs without re-encoding. Recommendation: use qpdf only, unless the PDFium renderer fallback is adopted anyway.
4. **Accepted.** **Text, stamps and layers have no milestone.** F4 (Must) and F5 aren't listed under any milestone in §9. This plan adds them as M2b.
5. **Decided (Nov 3).** **Per-file view settings have no storage location.** Crop, inversion, fold-out choice, two-up cover and skip-blank go in the `view` section of each score's annotation file. Device preferences (dark/sepia, fit mode, tap zones) stay in app settings. Annotations themselves move from sidecars next to the PDF to a central library in `Documents\Erm\Annotations` (§4.4). **Update design doc §6 and §11 to match.** The trade-offs accepted: annotations no longer travel with a copied PDF (covered by export/import, M5) and the content hash becomes the only link (covered by the index and reattach flow, M2a).
6. **Decided (Nov 3).** **Finger input while annotating.** §5 says pen input never turns pages in annotation mode, and §6 says fingers turn pages when a stylus is in use, which would leave no finger gesture for pan and select. Decision: in annotation mode, page turns need an edge swipe or a dedicated button (toolbar Next/Prev, or a mapped pedal, key or remote). Finger taps and drags are for pan, zoom and select. Details are in M2a task 4. Update §5 and §6 of the design doc to match.
7. **Accepted.** **Hash coverage.** Hashing the size plus the first and last 1 MB won't notice an edit in the middle that keeps the same size. That's rare, and acceptable given the reattach flow, but record it in the doc.
8. **Accepted.** **"Pointer Events" (§6)** is browser terminology. In Qt this is `QPointerEvent`/`QTabletEvent` through `PointHandler`. This only matters to keep the doc accurate.
9. **Deferred.** **Memory on high-DPI 2-in-1s.** At 3000×2000, one ARGB32 page is about 24 MB, so a 4-page cache in two-up uses around 190 MB before Qt and the PDF itself. Consider `Format_Grayscale8` for pages detected as greyscale (most scans) to cut that by 4×. Measure this in M0.
10. **Deferred.** **Missing page navigation.** There is no requirement for go-to-page or thumbnails, but they're needed in practice. Added to M1.
11. **Deferred.** **Setlist file format** isn't specified. This plan proposes `.ermset` JSON (M5).
12. **Deferred.** **Windows press-and-hold (M5).** Windows turns a long touch press into a right-click, and edge swipes open system panels. Both can interfere with the 2 s stage-mode exit and the annotation-mode edge swipe. Proposed fix: turn off press-and-hold and edge gestures for the Erm window.
13. **Under review.** **Network adapter for the phone remote (M3).** The QR code has to contain an IP address the phone can reach, but this laptop has several adapters: Wi-Fi, Tailscale, NordLynx (NordVPN) and other virtual ones (`ipconfig`, Nov 3). NordLynx also has a default gateway, so even "pick the adapter with the default route" could choose the VPN. While NordVPN is connected, it may also block incoming LAN connections unless its LAN-access setting is on. Proposal: list adapters, rank physical Wi-Fi/Ethernet first and skip known virtual and VPN adapters, show the chosen IP next to the QR code, let the user pick another, remember the choice per network, and listen only on that adapter. If the phone never connects, show a hint about VPNs and the firewall.

---

## 10. First two weeks, concretely

- [ ] Day 1: move the project to `C:\dev\erm`, `git init`, install the toolchain, write down the target device's specs.
- [ ] Day 2: `tools/make_fixtures.py` and the fixture set. Find the heaviest real score.
- [ ] Days 3–5: M0 viewer (QML + QPdfDocument on a worker thread, 4-page cache, key turns, latency overlay).
- [ ] Days 6–7: M0 pen test on the target device. Check CropBox/rotation and the annotation render flag.
- [ ] Day 8: pypdfium2 comparison if needed.
- [ ] Days 9–10: `prototype/RESULTS.md`, go/no-go, then start the M1 skeleton (CMake, CI, installer, PageGeometry tests).
