# Claude_Snapshot_Maker-Updater
packs entire projects into a single snapshot text file.  Drag the root folder and drop.  Excludes large files automatically.  Add ONE single file containing all your code and keep up to date in Claude Projects easily.
# Terran's Claude Project Packer

A single-file, browser-based tool that snapshots a project's source code into a structured `.txt` file you can upload to a Claude Project's knowledge base. No build step, no install, no server — open the HTML file (or visit the GitHub Pages URL) and drag in a folder.

> **Status:** Rust projects are the only project type that has been thoroughly tested end-to-end. Every other language preset (Python, JS/TS, C/C++, C#, Go, Java, Kotlin, Swift, Ruby, PHP, VB .NET, Dart, Elixir, Haskell) ships with the same filtering logic but has **not** been validated against real-world projects yet. Expect to need manual overrides (right-click → pack/reference/skip) on non-Rust projects until these are battle-tested. Bug reports and corrections to the per-language extension/directory lists are welcome.

---

## What this is for

Claude Projects let you attach a knowledge base of files that Claude references throughout a conversation. For a coding project, the most useful thing you can give Claude is your actual source — but most projects also contain huge amounts of stuff Claude doesn't need to see: compiled binaries, `node_modules`, virtual environments, build caches, lockfiles, images, and so on. Uploading all of it either fails outright (file size/count limits) or buries your real code in noise.

This tool lets you:

1. Drop a project folder in
2. Automatically filter out build artifacts and binaries before they ever load into memory
3. Pick a project-type preset that knows which files are actually relevant for that language
4. Manually fine-tune anything the preset got wrong (force-pack a file, mark something reference-only, or skip it entirely)
5. Build a single `.txt` snapshot and upload that to your Claude Project

---

## Quick start

1. Open `index.html` in Chrome or Edge (the save dialog feature works best in Chromium browsers; Firefox falls back to a standard download)
2. **Drag your project's root folder** onto the drop zone — do not use the "Browse…" picker for anything but small projects (see [Drag-and-drop vs. folder picker](#drag-and-drop-vs-folder-picker) below)
3. Pick a **project type** (e.g. Rust) from the grid — this applies a whitelist filter
4. Review the file tree. Adjust any marks you disagree with
5. Click **Build snapshot**
6. Click **Save .txt** and upload the resulting file to your Claude Project's knowledge base

---

## How the filtering actually works

There are three possible fates for any file: **pack**, **reference**, or **skip**.

| Fate | Meaning |
|---|---|
| **Pack** | Full file content is written into the snapshot |
| **Reference** | Only the file's path and size are written — Claude knows the file exists and roughly how big it is, but never sees its contents |
| **Skip** | The file is omitted entirely. It does not appear anywhere in the snapshot |

Fate is resolved by a single function that checks, in order:

1. **Your manual override** (right-click mark) — always wins, no exceptions
2. **Global always-skip filters** — the directory/extension/filename lists in the "Manual filter overrides" panel
3. **No project type selected** → everything passes through and gets packed (subject to the size cap)
4. **Project type whitelist** — see below

### Project type whitelist logic

When you select a project type (e.g. Rust), the tool applies a strict whitelist:

- **`packExts`** — file extensions that are eligible to be packed (for Rust: `.rs`, `.toml`)
- **`packNames`** — exact filenames always packed regardless of extension (for Rust: `Cargo.lock`)
- **`refDirs`** — directories whose contents are always reference-only (e.g. `node_modules` for JS/TS, `.venv` for Python)
- **`refExts`** — extensions that are always reference-only (typically generated/compiled artifacts that survived intake filtering)
- **Everything else** — any file that doesn't match the above falls through to your **"Unknown files →"** toggle, set to either **Reference** or **Skip**

This means a Rust project with `.rs`, `.toml`, `.html`, `.css`, and `.json` files will pack the `.rs`/`.toml` files and send everything else to whichever bucket your unknown-files toggle is set to — **unless you manually force-pack those other files**, which always works regardless of project type.

### Manual marks always override the filter

Right-click any file or folder (or multi-select with Shift/Cmd-click first) and choose **Force pack**, **Mark reference**, or **Mark skip**. This is stored separately from the project-type logic and takes priority over it unconditionally. A manually packed `.js` file in a Rust-typed project will be packed in full, regardless of what the Rust whitelist says.

Marking a **folder** propagates the mark to everything inside it (shown with a faded "↑" badge on children indicating an inherited mark).

---

## Automatic binary and build-artifact exclusion (intake filtering)

This is the most important efficiency feature, and it **only works when you drag-and-drop**, not when you use the folder picker. Details below in [Drag-and-drop vs. folder picker](#drag-and-drop-vs-folder-picker).

When you drag a folder onto the drop zone, the tool walks the directory tree using the browser's `FileSystemEntry` API. As it walks, it makes reject decisions **before** ever instantiating a `File` object for the entry:

### Directories that are never recursed into

```
target          .git            .cargo          .cache
node_modules    __pycache__     .mypy_cache     .pytest_cache
.ruff_cache     .hypothesis     DerivedData     .stack-work
dist-newstyle   .dart_tool      .pub-cache      Pods
Carthage        .bundle         .gradle         .idea
.vs             CMakeFiles      .next           .turbo
dist            build           out             bin
cmake-build-debug   cmake-build-release   priv/static   _build
```

If the walker encounters any of these directory names, it does not descend into them at all. None of their contents — however many thousands of files — ever get read, instantiated, or counted. For a Rust project, this is what keeps `target/` (which can contain tens of thousands of compiled artifacts) from ever touching memory.

### File extensions rejected before they're read

```
Compiled/object files:  .rlib .rmeta .d .pdb .ilk .exp .o .obj .a .lib
                         .so .dylib .dll .exe .wasm
JVM bytecode:            .class .jar .war .ear .aar
Python bytecode:         .pyc .pyo .pyd
Haskell/Erlang:          .beam .hi .dyn_hi .dyn_o
Archives:                .zip .tar .gz .bz2 .xz .rar .7z .tgz
Images:                  .png .jpg .jpeg .gif .bmp .ico .webp .tiff
Video/audio:             .mp4 .mp3 .wav .ogg .flac .aac .mov .avi .mkv .webm
Fonts:                   .woff .woff2 .ttf .otf .eot
Office/binary docs:      .pdf .doc .docx .xls .xlsx .ppt .pptx
Apple build artifacts:   .framework .ipa .xcarchive
Dart:                    .dill .snapshot
Misc package formats:    .nupkg .gem .whl .egg
```

These checks run against the filename only — no file content is read, no `File` object is created, and no memory is consumed for a rejected file.

### Large files become reference-only automatically

Any file that survives the directory and extension checks but is larger than **2MB** is stored as a lightweight `{path, sizeKB}` object instead of a full `File` object. It shows up in the tree tagged `large·ref` and gets listed by path and size in the snapshot's reference section — but its content is never read into memory, ever.

This three-layer intake filter is what makes the difference between a Rust project loading 50 relevant files almost instantly versus a naive approach that tries to instantiate 75,000+ File objects from `target/` and crashes the tab.

---

## Drag-and-drop vs. folder picker

**Use drag-and-drop.** It's not just a recommendation — it's a hard technical requirement if you want the intake filtering described above to actually run.

### Why the folder picker doesn't get filtered

When you click "Browse…" and pick a folder through the native OS file picker, the browser pre-instantiates a `File` object for **every single file in that directory tree** before handing control back to the page's JavaScript. By the time the tool's code runs, the browser has already done all the work of reading the entire `target/` folder, every binary, every cached dependency — the damage to memory and tab responsiveness is already done. The tool can still filter the resulting array down before building the tree, but the expensive part already happened.

### Why drag-and-drop works correctly

Dropping a folder onto the page gives access to the `DataTransferItem.webkitGetAsEntry()` API, which returns lightweight directory/file *entries* rather than fully-read files. The tool can inspect each entry's name and decide whether to skip it **before** calling `.file()` to materialize the actual content. This is the only browser API that allows reject-before-read behavior, which is why it's the primary, recommended path in the UI.

If you click "Browse…" anyway, a warning modal explains this and asks you to confirm — it's left available for genuinely small projects where the distinction doesn't matter, but it is not recommended for anything with a `target/`, `node_modules/`, or similar heavy build directory.

---

## The resulting snapshot file

### Filename

The output file is named automatically based on the **root folder name** you dropped, sanitized to remove special characters:

```
my_rust_project          →  my_rust_project_snapshot.txt
Tauri App (v2)           →  Tauri_App_v2_snapshot.txt
```

The sanitization strips anything that isn't a letter, digit, underscore, or hyphen and replaces it with an underscore.

### Structure

The snapshot is plain text with two sections:

**1. Reference files block** (only present if at least one file is reference-only or large-ref):

```
=== REFERENCE FILES (known to project, not packed) ===
  src-tauri/target/debug/app.exe  (14823 KB)
  node_modules/react/index.js  (2 KB)
=== END REFERENCE FILES ===
```

This tells Claude these files exist, where they live, and roughly how large they are — without spending any context budget on their actual contents.

**2. Packed file blocks** — one per packed file, in this format:

```
--- START FILE: src/main.rs ---
fn main() {
    println!("Hello, world!");
}
--- END FILE: src/main.rs ---

--- START FILE: Cargo.toml ---
[package]
name = "my-app"
version = "0.1.0"
--- END FILE: Cargo.toml ---
```

Paths are written relative to the dropped root folder, matching what you'd see in the file tree.

### Saving the file

If you're on Chrome or Edge, clicking **Save .txt** opens the native OS save dialog (`showSaveFilePicker`) so you can choose exactly where it goes and rename it if you like. On Firefox or other browsers without that API, it falls back to a standard browser download into your downloads folder.

---

## Sessions — saving and resuming your work

Manual marks, the active project type, your unknown-files default, and your filter overrides can all be exported as a session file so you can close the tab and come back later without redoing your work.

### Why this is necessary

Browsers cannot retain a reference to a folder on your disk between page loads — there's no "remember this folder" mechanism for security reasons. Every time you reopen the tool, you **must** drag the project folder in again. A session file solves the *configuration* half of that problem (your marks, your type selection, your filters) but you will always need to re-drop the actual folder.

### Saving a session

Click **Save session** in the session bar at the top. This downloads a `.json` file named after your project root, e.g. `my_rust_project_packer_session.json`. It contains:

```json
{
  "version": 1,
  "savedAt": "2026-06-21T...",
  "projectRoot": "my_rust_project",
  "activeType": "rust",
  "unknownDefault": "ref",
  "treeDefault": "depth1",
  "manualState": {
    "my_rust_project/src/main.rs": "pack",
    "my_rust_project/assets": "skip"
  },
  "filterSettings": { "dirs": [...], "exts": [...], "names": [...], "maxKB": 500 }
}
```

The button is enabled as soon as you have either a folder loaded or any manual marks set.

### Importing a session

Click **Import session** and select a previously saved `.json` file. Two outcomes are possible:

- **No folder loaded yet** — the session is staged. The status line shows *"Session staged — drop \[project name\] folder to activate."* Your settings (type, toggles, filters) apply immediately; manual marks apply automatically the moment you drop the matching folder.
- **A folder is already loaded** — the session reconciles against it immediately, assuming the root folder name matches.

### Root name mismatch

If the session's saved `projectRoot` doesn't match the root folder name you're dropping (for example, you renamed the project folder since you last saved), a modal appears with three choices:

- **Remap & apply** — rewrites every manual mark's path to use the new root folder name, preserving all your marks
- **Apply as-is** — loads the marks unchanged, which will likely fail to match any current file paths (use only if you know what you're doing)
- **Cancel** — backs out and lets you re-check the folder you're dropping

---

## Updating a project as it grows (re-drop / rescan)

Once a folder is loaded, the drop zone is replaced by a persistent **"Drop updated folder here"** strip that never goes away as long as a project is loaded. This is the mechanism for keeping your snapshot current as the project evolves — there is no other way to rescan, because browsers cannot hold a live handle to a folder on disk between drops.

### What happens on re-drop

1. Drag the same project's root folder onto the re-drop strip (or click it to use the folder picker, with the same filtering caveat as above)
2. The tool diffs the new file list against the previous one
3. A diff bar appears summarizing:
   - **`+N new`** — files that exist now but didn't before
   - **`−N removed`** — files that existed before but are gone now
   - **`N marks preserved`** — manual marks whose file paths still exist and still apply
   - **`N orphaned`** — manual marks whose paths no longer exist (the file was renamed, moved, or deleted)

### What's preserved vs. reset

- **Manual marks are never cleared automatically on re-drop.** They persist exactly as you left them, matched by path string.
- New files are picked up and run through the active project type filter immediately.
- Deleted files simply disappear from the tree; their orphaned mark (if any) becomes harmless dead data that doesn't affect anything.
- If you rename or move a file, its old manual mark becomes orphaned (the path no longer matches) and the file at its new path starts fresh, subject only to the type filter.

To fully wipe manual marks instead of preserving them, use the **"Clear manual marks"** button next to the project type controls — it shows a live count badge of how many marks currently exist and is disabled when there are none.

---

## The file tree

### Selection

- **Click** — select a single item
- **Shift+click** — select a range from your last click to this one
- **Cmd/Ctrl+click** — add or remove a single item from the current selection

### Right-click context menu

Right-clicking any selected item(s) opens a menu with:

- **Force pack** — overrides everything, always packs full content
- **Mark reference** — path + size only, no content
- **Mark skip** — omitted entirely
- **Clear marks** — removes manual override, falls back to type-filter logic
- **Select all visible** / **Deselect all**

A floating selection bar with the same actions also appears above the tree whenever anything is selected, so you don't have to right-click if you'd rather click a button directly.

### Expand/collapse

Above the tree:

- **Expand all** / **Collapse all** buttons act immediately on the current tree
- **Default** toggle (Collapsed / Expanded / Depth 1) controls what state new folder loads start in. **Depth 1** (the default) shows top-level directories open with everything inside them collapsed — this is usually the most navigable starting point for a typical project.

### Badges

- **pack** (green) / **ref** (amber) / **skip** (red/grey) — the resolved fate of that file
- **manual** (blue) — this file or an ancestor folder has a manual override applied
- **↑** suffix — this mark is inherited from a parent folder rather than set directly on this item
- **\*** suffix — this mark came from the project-type's automatic directory rules, not a manual override

---

## Known limitations

- **Only Rust has been properly tested.** All other project type presets are best-effort based on common project conventions and have not been validated against real codebases. If a preset is excluding something it shouldn't (or including something it shouldn't), the fix is either a manual override in the moment, or — better — open an issue/PR with the correction so the preset improves for everyone.
- **The folder picker bypasses intake filtering entirely.** This is a browser API limitation, not a bug — see the dedicated section above.
- **No persistent directory handle.** You must re-drop the folder every time you reopen the tool, even with a saved session.
- **Renamed/moved files lose their manual mark** on re-drop, since marks are keyed by path string.
- **2MB size cutoff for ref-only-by-default is fixed** in this version (not yet user-configurable) — files larger than this always become reference-only metadata regardless of project type, on top of the separate "max file size to pack" setting in the filter panel.

---

## Hosting on GitHub Pages

This is a single self-contained HTML file with no build step and no external dependencies beyond a CDN-hosted icon font. To host it:

1. Push `index.html` (rename the tool's HTML file if needed) to your repository
2. In your repo settings, enable **GitHub Pages**, pointing at the branch/folder containing `index.html`
3. Visit the published URL — the tool runs entirely client-side; no file you drop ever leaves your browser

No server, no API keys, no build process required.

---

## License / attribution

Add your preferred license here before publishing.
