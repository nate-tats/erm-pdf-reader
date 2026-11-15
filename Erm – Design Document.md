# Erm – Design Document

Nov 2, 2026 · @Nathan · Status: Draft v0.2 (Nov 14: updated to match the decisions in *Erm – Build Plan* §9)

## 1. Overview

Erm (temporary name) is a lightweight, distraction-free PDF reader for sheet music, used on a laptop, 2-in-1 or tablet sitting on a music stand. Pages turn with one tap or a hands-free physical switch, and annotations (fingerings, dynamics, cuts, reminders) sit in a layer over the PDF without modifying the original file.

### Goals

- Turn pages in under 100 ms from any input, with no missed or double turns.
- Annotate quickly with pen, highlighter, text and music stamps, and show or hide annotation layers.
- Accept input from a phone, a wired embedded controller and a Bluetooth mouse or pedal through a single input abstraction.
- Open large scanned scores (100 MB+) quickly and work fully offline.
- Stay small: one window, a few tools, a small install.

### Non-goals (v1)

- Editing notation, MusicXML, optical music recognition or playback.
- Cloud sync, cloud storage, accounts, or syncing annotations between devices. (The annotation library lives in Documents, so OneDrive or another backup tool may copy it. That's intended as a backup, not a sync feature; moving annotations to another computer is done with export and import.)
- A full library manager. Folders and setlists are enough.

## 2. User scenarios

- **Practice:** The player marks fingerings and breath marks with a stylus, then plays through and turns pages with a foot pedal.
- **Performance (stage mode):** Annotation tools are hidden, every touch except the turn zones is ignored, and the screen never sleeps. A bumped tablet never jumps to the wrong page.
- **Repeats and D.S./coda:** The player places a jump link on page 4 ("to page 2") and fires it with a tap or a dedicated switch.
- **Assisted turning:** A second person (teacher, page turner) turns pages from their own phone.
- **Setlist gig:** Several pieces queued in order. Going past the last page of one piece opens the next.

## 3. Functional requirements

| ID | Requirement | Priority |
| --- | --- | --- |
| F1 | Open PDFs from local files or folders, with a recent-files list | Must |
| F2 | One-tap turn zones (default: right two-thirds = next, left third = previous), configurable | Must |
| F3 | Display modes: single page, two-up (landscape), half-page turn | Must / Should |
| F4 | Annotations with stylus support (pressure, palm rejection): freehand pen, highlighter, eraser, text, music stamps | Must |
| F5 | Annotation layers (e.g. "My part", "Teacher") that can be shown or hidden | Should |
| F6 | Bookmarks and jump links for repeats | Should |
| F7 | Setlists: an ordered list of PDFs, with optional page ranges | Should |
| F8 | Input mapping screen with a learn mode ("press your switch now") | Must |
| F9 | Phone remote over the local network, with no app install | Must |
| F10 | Stage mode: locked UI, screen kept awake | Must |
| F11 | Per-page margin crop, to enlarge the music | Should |
| F12 | Export a flattened, annotated PDF | Could |
| F13 | Images (JPG, PNG) to PDF (section 10) | Should |
| F14 | Merge PDFs, carrying annotations over (section 10) | Should |
| F15 | Split and extract pages; reorder, rotate and delete pages (section 10) | Should |

## 4. Architecture and tech stack

### Components

1. **Renderer:** wraps the PDF engine and renders pages to bitmaps off the UI thread.
2. **Page cache:** keeps the current page, the next two and the previous one pre-rendered at screen resolution, so a turn only swaps a bitmap.
3. **Annotation store:** loads, saves and undoes annotations kept in a central annotation library, one file per score (section 6).
4. **Input service:** turns every source (touch, keys, mouse, serial, network) into commands through the input mapping. Debounce happens in the command bus's turn guard.
5. **Remote bridge:** a small local WebSocket and HTTP server for the phone remote.
6. **UI shell:** viewer, toolbars, settings and the input-mapping screen.

### Command bus

Input sources never touch the viewer directly. They emit commands, and the viewer only listens for commands. That keeps every switch interchangeable and testable.

```
[Touch] [Keyboard/pedal] [BT mouse] [USB serial] [Phone (WebSocket)]
     \         |             |           |            /
      +------> Input service: map -> command <-----------+
                              |
             Command bus + turn guard (lockout, no auto-repeat)
                              |
            NEXT | PREV | NEXT_HALF | JUMP(id) | TOGGLE_LAYER | STAGE_MODE
                              |
                Viewer  <-->  Page cache  <-->  Renderer
                  |
           Annotation store  -->  Annotations\<sha256>.json
```

### Stack options

### Chosen stack: Qt 6, prototyped in PySide6, built in C++

- **Prototype (M0):** Python with PySide6 and the Qt PDF module. Goal: prove rendering speed on the heaviest scanned score and pen pressure on the real device. Write it "C++-shaped": Qt types only (`QImage`, `QJsonDocument`, `QSaveFile`), type hints, and the same class boundaries as the final app.
- **Product (M1 onward):** C++17/20 with Qt 6.8 LTS. Core logic in C++, screens in QML. Built with CMake. Developed and unit-tested on Linux (WSL); the Windows release is built with MSVC 2022 in GitHub Actions (section 8) and packaged with `windeployqt` and Inno Setup.
- **Python stays for:** tests, tooling scripts, and CircuitPython firmware for the Pico controller.
- **Why:** PySide6 is a thin wrapper over C++ Qt and QML files carry over unchanged, so the port from prototype to product is mostly mechanical. Qt has built-in pieces for every part of the design and a path to Android and iPad. It also matches the existing C/C++ and Python skills.
- **Engineering rules:** Modern C++ (RAII, smart pointers or Qt parent ownership, no raw `new`/`delete`). AddressSanitizer and clang-tidy in debug builds. One dedicated render thread, because PDFium isn't thread-safe.

#### Qt building blocks

| Component | Qt pieces |
| --- | --- |
| Renderer | `QPdfDocument::render()` → `QImage`, on one worker thread |
| Page display | QML `Image` + `QQuickImageProvider` |
| Tap zones and stylus | QML `TapHandler` (finger and mouse only); `PointHandler` with pressure |
| Strokes and stamps | `QPainterPath` in page coordinates, drawn in a `QQuickPaintedItem` overlay; Bravura font via `QFontDatabase` |
| Annotation file | `QJsonDocument` + `QSaveFile` (atomic writes) |
| Keys, pedals, BT mouse | `Keys.onPressed` or an app-wide event filter (ignore auto-repeat); `Qt::BackButton` / `Qt::ForwardButton` |
| USB serial (optional) | `QSerialPort` |
| Phone remote | `QWebSocketServer` for the protocol; a small static-file HTTP handler on `QTcpServer` for the phone page (`QHttpServer` is GPLv3 or commercial, not LGPL); QR code via qrcodegen (MIT) |
| Screen awake | Win32 `SetThreadExecutionState`, behind an interface |
| Command bus | `CommandBus` QObject with a `command` signal; the turn guard lives here |

### Backup options

| Option | Switch to it if… | Trade-offs |
| --- | --- | --- |
| B1. PySide6 as the product (skip the port) | The prototype is good enough and a tablet build is no longer wanted | Faster development; 80–150 MB installer, slower startup, no real tablet path |
| B2. Tauri 2 (Rust core, web UI) + PDF.js or PDFium | A web-based UI becomes more attractive | Small install, good mobile support; two new languages (Rust, TypeScript) |
| B3. Flutter + PDFium binding | A tablet becomes the main device | Best touch and tablet experience; new language (Dart); weaker desktop serial support |
| Renderer fallback: PDFium directly (pypdfium2 in the prototype) | Qt PDF is too slow or lacks a feature | Same renderer interface; a little more build setup |

Not recommended: Electron (too heavy), Dear ImGui (not built for touch or stylus), and MuPDF/PyMuPDF unless the project is open source or licensed (AGPL).

## 5. Page turning and display modes

- **Tap zones:** By default the right two-thirds of the screen turns forward and the left third turns back. Zones are configurable, including top and bottom zones for portrait use. In annotation mode the tap zones are off, so neither pen nor finger taps turn pages (section 6, Input).
- **Instant turns:** No animation by default (an optional fade is under 80 ms). The page is already in the cache, so a turn is a bitmap swap.
- **Half-page turn:** The first press replaces the top half of the screen with the top half of the next page while you finish the bottom half of the current one. The second press completes the turn. This lets players read ahead and is a key feature for musicians.
- **Two-up:** In landscape, show pages *n* and *n+1*. A turn advances one or two pages (setting).
- **Crop:** Trim white margins per page (auto-detected, adjustable) so the staves fill the screen.
- **Stage mode:** Hides all chrome, locks gestures except turn zones, keeps the screen awake, and requires a long press (about 2 s) to exit.
- **Turn guard:** Commands arriving within the lockout window (default 300 ms) are dropped, and held-key auto-repeat is ignored. This prevents double turns from switch bounce or a heavy foot.

### Mixed page sizes

A single PDF can contain pages of different sizes and orientations. Every display rule works per page:

- **Fit (default):** Each page is scaled on its own to fit the screen (or the crop area) and centered, with blank space as needed. An optional **Keep same zoom** setting uses one scale for the whole file, so staff lines stay one size, at the cost of some pages not fitting.
- **Two-up:** Both pages are scaled to the same height, then centered as a pair. A **Cover alone** setting shows page 1 by itself so book spreads pair as left/right pages.
- **Half-page turn:** The split happens in screen space, after each page is scaled, not at half of each page's height.
- **Page cache:** Each page is rendered at its own target size, never above screen resolution.
- **Page geometry:** Use each page's CropBox (what other readers show), not the MediaBox. Apply the page's built-in rotation and allow for a CropBox origin that isn't (0, 0). All of this lives in one `PageGeometry` class that converts between screen and page coordinates. Viewer, annotations and crop all use it, and it is settled in M1, before annotations depend on it.
- **Landscape pages in a portrait score** (fold-outs): per-file choice to fit them as they are, or show them rotated to fill the screen.

## 6. Annotation system and data model

- **Non-destructive:** Annotations live in a central annotation library, `Documents\Erm\Annotations\` by default (changeable in Settings → Storage), with one `<sha256>.json` file per score. The original PDF is never modified and nothing is written next to it.
- **Library:** An `index.json` caches hash → last path, file name, page count, page sizes and last opened. It can always be rebuilt by scanning the folder. The last 5 versions of each file are kept as `<sha256>.json.1` … `.5`. Documents rather than AppData, so you can see the files and normal backups include them. The uninstaller never deletes the library.
- **Identity:** Each annotation file is keyed by a content hash of the PDF (SHA-256 of the file size plus the first and last 1 MB), so annotations survive renames and moves. Known limit: an edit in the middle of the file that keeps the same size isn't detected. That's rare, and the reattach flow (section 11) covers the common cases. Annotations no longer travel with a copied PDF; use export and import for that.
- **Coordinates:** Stored in PDF page space (points, relative to each page's CropBox, with the page's rotation applied, via `PageGeometry`). They stay correct at any zoom level, crop, screen size or device.
- **Input:** Qt's pointer events (`QPointerEvent` / `QTabletEvent`, through `PointHandler`) give pressure and tilt. Annotation mode is entered and left only with a dedicated toolbar button or a mapped input, never by accident. Inside it:
  - The pen draws, erases and places items, and never turns pages.
  - Fingers pan, pinch-zoom (up to about 4×) and select. Tap zones are off (palm rejection).
  - Pages turn only by an edge swipe (starting in the outer edge strip and moving mostly horizontally), the Next/Prev buttons on the annotation toolbar, or an external input (pedal, key, mouse, phone remote), so you can practise and mark up without leaving the mode.
  - Outside annotation mode, the pen acts like a finger (taps turn pages) and nothing can be drawn.
- **Tools:** Pen, highlighter, eraser (stroke and partial), text, and music stamps: fingerings 1–5, dynamics (pp–ff), breath mark, up-bow and down-bow, accidentals, fermata, and the "watch out" glasses. Draw stamps with a SMuFL font such as Bravura (OFL-licensed).
- **Saving:** Autosave about 500 ms after each stroke ends (atomic write), with undo and redo per page.
- **Interop (Could):** Export as standard PDF annotations (Ink, FreeText) so other readers can show them, or flatten them into a copy.

### Annotation file format (draft v1)

```json
{
  "version": 1,
  "pdf": {
    "hash": "sha256:...",
    "lastPath": "C:\\Users\\me\\Music\\Sonata.pdf",
    "fileName": "Sonata.pdf",
    "pages": 12,
    "pageSizes": [[595.3, 841.9], [595.3, 841.9]]
  },
  "view": {
    "crop": { "3": [36, 40, 560, 800] },
    "invert": false,
    "foldOuts": "rotate",
    "coverAlone": true,
    "skipBlank": false
  },
  "layers": [
    { "id": "me", "name": "My part", "visible": true },
    { "id": "teacher", "name": "Teacher", "visible": false }
  ],
  "items": [
    { "id": "a1", "layer": "me", "page": 3, "type": "ink",
      "color": "#d33", "width": 1.5,
      "points": [[102.4, 511.0, 0.6], [104.1, 512.2, 0.7]] },
    { "id": "a2", "layer": "me", "page": 3, "type": "stamp",
      "glyph": "dynamicForte", "at": [220.0, 498.5], "scale": 1.0 },
    { "id": "a3", "layer": "me", "page": 4, "type": "jump",
      "rect": [40, 700, 120, 730], "target": { "page": 2, "y": 650 } },
    { "id": "a4", "layer": "me", "page": 3, "type": "highlighter",
      "color": "#fd0", "width": 8.0,
      "points": [[90.0, 480.0, 1.0], [180.0, 480.5, 1.0]] },
    { "id": "a5", "layer": "teacher", "page": 5, "type": "text",
      "at": [300.0, 620.0], "text": "slower here", "size": 10, "color": "#226" }
  ],
  "bookmarks": [ { "name": "Coda", "page": 9 } ]
}
```

- Page numbers are **1-based**. `width` is in PDF points. `points` are `[x, y, pressure]` in `PageGeometry` page space.
- `pdf` has the file's last path, name, page count and page sizes (one entry per page; shortened above), so the index can be rebuilt and a changed file can be reattached.
- `view` holds settings that belong to the score: per-page crop rects, colour inversion, fold-out handling, two-up "cover alone", skip blank pages. Device preferences (dark/sepia page mode, fit mode, tap zones) stay in app settings.
- The eraser stores nothing. A partial erase splits a stroke into new items.
- Still open: whether the highlighter is its own type (as above) or `ink` with `"blend": "multiply"`. The final schema is written in `docs/annotations-v1.md` plus a JSON Schema file, before M2 (build plan §4.4).

## 7. Physical switch input

**Design principle: wherever possible, make the switch look like a keyboard.** If a device sends standard HID key presses (PageDown/PageUp, arrow keys, space), it works with no drivers on every OS, including iPad and Android. Commercial Bluetooth page-turner pedals (AirTurn, PageFlip, Donner and others) already work this way. The app then only needs a good key and button mapping screen. Richer channels (serial, WebSocket) are added only where they give something extra, such as feedback.

| Source | Transport | What the app receives | Notes |
| --- | --- | --- | --- |
| Bluetooth mouse | BT HID | Left/right click, wheel, side buttons 4 and 5 | Map each button in settings. In stage mode, capture clicks anywhere on the window so the cursor position does not matter. |
| Commercial BT pedal | BT HID keyboard | Key events | No extra work needed. Document known pedal modes. |
| Wired embedded controller | USB HID keyboard (recommended) | Key events | See below. Plug and play everywhere. |
| Wired controller, advanced | USB serial (CDC) | Text lines such as `NEXT` | Two-way: the app can light an LED or show a page number. Needs serial-port access in the app. |
| Phone | Wi-Fi LAN, WebSocket | JSON commands | Phone opens a web page through a QR code. No install. |

### Wired embedded controller

- **Hardware:** A Raspberry Pi Pico (RP2040), an Arduino Pro Micro or Leonardo (ATmega32U4), or an ESP32-S3. All can act as a native USB keyboard. (An ESP32 with BLE HID gives a wireless version later.)
- **Switches:** Momentary, normally-open footswitches wired between a GPIO pin and GND, using the internal pull-up. A 3.5 mm TRS jack carries two switches on one cable: tip = next, ring = previous, sleeve = GND.
- **Debounce:** 20 ms in firmware, plus the app's 300 ms turn guard.
- **Firmware sketch (CircuitPython on a Pico):**

```python
import time, board, digitalio, usb_hid
from adafruit_hid.keyboard import Keyboard
from adafruit_hid.keycode import Keycode

kbd = Keyboard(usb_hid.devices)
mapping = {board.GP14: Keycode.PAGE_DOWN, board.GP15: Keycode.PAGE_UP}
buttons = []
for pin, key in mapping.items():
    b = digitalio.DigitalInOut(pin)
    b.switch_to_input(pull=digitalio.Pull.UP)
    buttons.append({"io": b, "key": key, "released": True})

while True:
    for btn in buttons:
        pressed = not btn["io"].value
        if pressed and btn["released"]:
            kbd.send(btn["key"])      # one tap per press, no auto-repeat
        btn["released"] = not pressed
    time.sleep(0.02)                   # 20 ms debounce / poll
```

### Phone remote

- The app runs a small HTTP and WebSocket server on the LAN and shows a QR code that contains the URL plus a one-time pairing token.
- The phone page shows large Next and Prev buttons (and optionally Half and Jump), the current page number and a thumbnail.
- **Keeping the phone awake:** The Wake Lock API only works on secure (HTTPS) pages, and this is a plain HTTP page on the LAN. The page tries Wake Lock first in case the browser allows it, and otherwise falls back to a hidden looping video (the NoSleep.js approach). Final decision before M3 (build plan §9.2).
- **Protocol:** `{"cmd":"next","seq":42}` goes to the app, and `{"ack":42,"page":5,"of":12}` comes back. A duplicate `seq` is ignored, so a retry never turns twice.
- **Security:** LAN only, token required, one controller at a time. While a phone is paired, any other phone is rejected. To switch phones, unpair the current one in the app, which makes a new QR code.
- **Limitation:** A web page cannot reliably capture the phone's volume buttons. If volume-button turning is wanted, that needs a native companion app or a phone app that acts as a Bluetooth keyboard.

### Learn mode

Settings → Inputs → "Assign Next": the app waits for the next key, mouse button, serial line or remote command, then saves it as the binding. Several bindings can map to the same command.

## 8. Platform decision: Windows or Linux (with WSL)

**Recommendation: ship a native Windows build, use a cross-platform stack, and use WSL for development, not as the place the app runs.**

Why not run the reader inside WSL:

- WSL is a developer environment. WSLg can display Linux GUI windows, but a reader running there is a guest on your Windows machine, with weaker touch and pen support and no direct link to your hardware.
- **Bluetooth is not available inside WSL**, so the Bluetooth mouse and pedals would only work through Windows anyway.
- **USB devices are not visible in WSL by default.** They have to be attached with `usbipd-win` every time, which is fine for flashing firmware but bad for a performance setup.
- The machine on your music stand is a Windows laptop or 2-in-1 (or a tablet), not a WSL instance.

How to use what you have (decided Nov 3, build plan §9.14):

- **Develop in WSL.** The project lives in WSL (`/home/natet/erm-pdf-reader`). The app is built there against the Linux Qt for daily work, unit tests, AddressSanitizer and clang-tidy, with quick visual checks through WSLg. Scripts, tests and controller firmware toolchains (the Pico SDK and PlatformIO) also run there.
- **Get the Windows `.exe` from CI.** GitHub Actions builds it with MSVC on every push. It's downloaded into the git-ignored `dist/` folder and copied to Windows by hand. Nothing is built or run through `/mnt/c`, which is slow from WSL.
- **Only Windows can test** pen and palm input, screen-awake, the firewall prompt, DPI and multi-monitor handling, Bluetooth pedals, and all the performance numbers in section 9. WSLg results don't count for these.
- **Trade-offs:** there's no local Windows debugger (Windows-only bugs are investigated through CI logs, crash dumps or a temporary local MSVC install), and every Windows test needs a push and a 5–10 minute build.
- With option A or B, a Linux build comes almost free later, for example for a Raspberry Pi with a portrait monitor as a dedicated stand display.

| Device you'll play from | Build for |
| --- | --- |
| Windows laptop or Surface-style 2-in-1 | Windows (native) |
| Android tablet or iPad | Qt for Android or iPad from the C++ codebase (Flutter as a backup); Windows as a bonus |
| Raspberry Pi or Linux mini-PC with a monitor | Linux (native on the device, not in WSL) |

### Does a tablet need a completely different build?

No. With Qt in C++, a tablet is another build target from the same codebase, not a rewrite. (The PySide6 prototype is desktop-only; tablet builds come from the C++ version.) The viewer, page cache, annotations, command bus and phone-remote server are all shared. What changes is a thin platform layer:

- **Build tools:** Android builds need the Android SDK and NDK (these work on Windows). iPad builds need a Mac with Xcode.
- **Switch input:** Bluetooth pedals, Bluetooth mice and the USB HID controller (through a USB-C adapter) work on tablets unchanged, as keyboards and mice. USB serial does not carry over: it isn't available on iPad and needs a plugin on Android. This is another reason to make HID the main path.
- **Files:** Tablets use a system file picker and an app sandbox rather than open folder access. The annotation library moves into app storage. It's already keyed by the PDF's hash, so nothing else changes.
- **Layout:** Portrait by default, larger touch targets, a toolbar instead of desktop menus. Design the UI to be responsive from M1.
- **Stylus:** Apple Pencil and Android styluses arrive as the same pointer events with pressure, so they use the same code path.

**Plan:** Build for Windows now. Keep platform-specific code (file access, serial) behind small interfaces, so a tablet target can be added later without restructuring.

## 9. Non-functional requirements, milestones, risks

### Non-functional requirements

- Input-to-screen page turn: under 100 ms (target 50 ms).
- Cold open of a 50-page scanned score: under 2 s to the first page.
- Memory: under 300 MB with a 100 MB PDF open.
- Install size: under 30 MB.
- Fully offline. No network except the optional LAN remote.
- Readability: large touch targets, plus dark and sepia page modes for dim stages.

### Milestones

**M0 – PySide6 spike (1–2 weeks):** open the heaviest scanned score with Qt PDF, measure render and turn times, and test pen pressure on the target computer. If it passes, M1 onward is built in C++. If not, try the PDFium renderer fallback before moving on.

1. **M1 – Viewer:** open a PDF, page cache, tap zones, keyboard and PageDown mapping, turn guard, page navigator (go-to-page or thumbnails).
2. **M2 – Annotations,** in two parts. **M2a:** annotation mode, pen, highlighter, eraser, zoom and pan, the central annotation library, undo. **M2b:** text, music stamps, layers.
3. **M3 – Phone remote:** WebSocket server, QR pairing, phone web page.
4. **M4 – Switches:** Pico HID controller, Bluetooth mouse mapping, learn mode, optional serial channel, and basic stage mode (screen kept awake, touch locked outside the turn zones).
5. **M5 – Performance features:** full stage mode, half-page turns, two-up, crop, setlists, jump links and bookmarks, dark and sepia modes, export, annotation export and import.
6. **M6 – PDF tools:** images to PDF, merge, split and page edits, with annotations carried over (section 10).

Tasks and exit criteria for each milestone are in the build plan (§5).

### Risks and mitigations

| Risk | Mitigation |
| --- | --- |
| Qt PDF is slow on huge scanned scores | Pre-render cache; switch to the PDFium backend |
| Bluetooth pedal or mouse sleeps, so the first press is lost | Document the behavior; optional "keep-alive" setting; prefer wired for performances |
| Accidental or double turns | Turn guard, stage mode, no auto-repeat |
| Lost annotations | Atomic writes (temp file, then rename, retrying with backoff if OneDrive or antivirus has the file locked, keeping edits in memory meanwhile), keep the last 5 versions |
| Licensing (MuPDF is AGPL) | Use Qt PDF / PDFium (BSD-style); link Qt as shared libraries to meet the LGPL |

### Decisions (Nov 2, 2026)

- **Primary device:** A Windows computer (laptop or 2-in-1) for v1. A tablet may come later, and the stack keeps that option open (section 8).
- **Stylus:** Required. Pen input with pressure and palm rejection is a Must (F4).
- **Phone remote:** For the performer's own use. One controller at a time, no shared mode. The phone page should work one-handed or propped on the stand.
- **Sync, cloud storage, accounts:** Out of scope. Annotations stay local.
- **Notation editing, OMR, playback:** Out of scope, handled by MuseScore or a separate project.
- **Stack:** Qt 6. Prototype in PySide6, build the product in C++ with QML. Tauri, Flutter and a Python-only product are backups (section 4).
- **PDF tools:** Images to PDF, merge and split are in scope (M6). Compression is out of scope.

### Decisions (Nov 3 and Nov 14, 2026)

Details and trade-offs are in build plan §9.

- **Annotation storage (Nov 3):** one central annotation library in `Documents\Erm\Annotations` instead of sidecars next to each PDF, with per-score view settings in the same file (section 6).
- **Fingers in annotation mode (Nov 3):** fingers pan, zoom and select; pages turn by edge swipe, toolbar buttons or external inputs (section 6).
- **Development (Nov 3):** develop and unit-test in WSL; the Windows `.exe` comes from GitHub Actions (section 8).
- **Phone page server:** `QTcpServer` for static files instead of `QHttpServer`, to stay LGPL (section 4).
- **Images to PDF:** qpdf only, no standalone PDFium (section 10).
- **Second phone (Nov 14):** rejected while another phone is paired (section 7).

## 10. PDF tools

A separate **Tools** screen, not part of the performance viewer, for preparing scores. Every tool writes a new file and never modifies the original. Compression is out of scope.

### Tools

- **Images to PDF:** JPG or PNG photos and scans become one PDF, one image per page, in the order you choose. JPEGs are embedded as-is (no re-compression, no quality loss), and the photo's EXIF rotation is applied. Page size: fit to image, A4 or Letter.
- **Merge:** Combine several PDFs in a chosen order, for example one PDF for a whole setlist.
- **Split and extract:** Split at chosen pages, every N pages or at bookmarks; or extract a page range, such as one étude from a book.
- **Page edits:** Reorder, rotate and delete pages. These come almost free with merge and split.

### Libraries

| Task | C++ product | Python prototype |
| --- | --- | --- |
| Images to PDF | qpdf: JPEGs written as-is as `/DCTDecode` image streams (no re-encode); PNGs decoded with Qt and written as Flate streams; EXIF orientation applied through the page's `/Rotate` or matrix. (Qt PDF doesn't expose PDFium's image APIs, and shipping a second PDFium isn't worth it.) | img2pdf |
| Merge, split, page edits | qpdf (C++, Apache-2.0) | pikepdf (wraps qpdf) |

### Design

- **Separate library:** a `pdftools` module (`merge`, `split`, `extractPages`, `imagesToPdf`) with no UI code. It is also exposed as a small command-line tool, which makes it easy to test from Python.
- **Background work:** Tools run on a worker thread with a progress bar and a Cancel button, never on the page-turn path.
- **Annotations follow their pages:** The output file gets its own annotation file in the library. Each page's annotations are copied and renumbered (page 3 of file B becomes page 15 of the merged file), annotations on rotated pages are rotated with them, and deleted pages drop theirs.
- **Safe output names:** never overwrite, for example `Sonata (merged).pdf` or `Etudes pp12-15.pdf`.
- **Protected PDFs:** Ask for the password, or refuse with a clear message.

## 11. Edge cases

### Page content

| Case | Handling |
| --- | --- |
| Mixed page sizes and orientations | Per-page rules in section 5 (Mixed page sizes) |
| Scans with big or uneven margins | Per-page auto-crop, adjustable (F11). Deskewing is a possible later tool. |
| Blank pages left for page turns | Option to skip blank pages (detected when the page is almost entirely white) |
| Heavy scans (600 DPI, full color, 200+ pages) | Render only at screen resolution; bounded page cache; thumbnails generated lazily |
| Annotations already in the PDF (Acrobat, forScore, a teacher) | Shown by default as a read-only pseudo-layer that can be hidden. Erm's own annotations stay in the annotation library. |
| Dark mode on scans | Color inversion is a per-file setting, because it looks wrong on photo-like scans |

### Files

| Case | Handling |
| --- | --- |
| File changed on disk (edited elsewhere, re-downloaded) | The hash no longer matches. Look in the library index for an entry with the same last path or file name. If its page count and sizes match, ask whether to reattach the old annotations. Otherwise leave them in the library (listed under "annotations for missing files") and say they're there. Never discard them silently. |
| Same content under two names | Both copies share one set of annotations (same hash). Treated as a feature; documented. |
| Damaged PDF | Show a clear error; open whatever pages the renderer can render; never crash, even mid-setlist |
| Password-protected PDF | Ask for the password; never store it |
| File moved or deleted while open | Keep showing the cached pages; warn after the piece |

### Performance moments

| Case | Handling |
| --- | --- |
| Next pressed on the last page | Open the next setlist item if there is one, otherwise do nothing. Either way, a brief edge flash confirms the press. |
| Prev pressed on the first page | Same rule in reverse |
| 2-in-1 rotates or window resizes mid-piece | Stay on the same page; re-render the cache in the background; show a scaled copy of the old render until the new one is ready |
| Multiple monitors with different scaling | Render for the monitor the window is actually on; re-render if it moves |
| Laptop sleeps or the lid closes | Keep the screen awake in stage mode; restore the same page after wake |
| Switch held down or bouncing | Turn guard and no auto-repeat (section 5) |
