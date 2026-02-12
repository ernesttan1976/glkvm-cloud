# Advanced Clipboard / Patch-Paste UI — Specification

## 1. Overview and goals

**Purpose:** Support the patch-transfer workflow (commit range → patches → empty files on target → paste via UI) with minimal errors and cognitive overhead when syncing patch sets from a source machine to a remote device through the glkvm-cloud web terminal.

**Target users:** Developers syncing patch sets from a Mac (or other source) to a Windows machine (or other target) via the glkvm-cloud web UI, using the remote terminal to paste patch content into pre-created empty files.

**Goals:**
- Minimize mistakes (wrong patch, duplicate paste, skipped patch).
- Reduce cognitive load (clear “next” patch, progress, optional validation).
- Keep the workflow compatible with a single-command creation of empty `.patch` files on the target.

---

## 2. Workflow summary (reference)

1. **Source (e.g. Mac):** Developer has a branch and a commit range (commit A → commit B). They generate one or more `.patch` files per commit (commit → patches = 1:M).
2. **Target (e.g. Windows):** For each patch file that exists on the source, create a corresponding empty `.patch` file using a **single** PowerShell or Bash command (patch on source → patch on target = 1:1).
3. **Transfer:** Using the glkvm-cloud UI (remote terminal to the target device), the developer pastes each patch’s **content** into the terminal (e.g. redirect into the empty file or into an editor), in the same order as the patch list.
4. The **same ordered list** of patch filenames is used both for (2) creating empty files and (3) the paste queue in the UI.

---

## 3. Single command for empty .patch files on target

The UI may later reference or generate these one-liners. The **patch list** (ordered list of `.patch` filenames) is the same list the UI uses for the paste queue.

### PowerShell

- From a file (e.g. `patch-list.txt`), one line per filename:
  ```powershell
  Get-Content patch-list.txt | ForEach-Object { New-Item -ItemType File -Path $_ -Force }
  ```
- From clipboard (list of filenames, one per line):
  ```powershell
  Get-Clipboard | ForEach-Object { New-Item -ItemType File -Path $_ -Force }
  ```

### Bash (e.g. WSL on Windows)

- From a file:
  ```bash
  while IFS= read -r f; do touch "$f"; done < patch-list.txt
  ```

---

## 4. Current behavior (baseline)

- **Paste:** Context menu “Paste” (and Shift+Insert) reads `navigator.clipboard.readText()` and sends the text to the terminal via WebSocket (`sendTermData`). There is no queue, no progress indicator, and no association with patch names.
- **File upload:** Separate flow via a modal; user selects a file and it is transferred in chunks over the WebSocket. Not used for the patch-paste workflow described here.
- **Copy:** Selection in the terminal is copied to the system clipboard via `vue-clipboard3` / `toClipboard()`.

---

## 5. Functional requirements for the advanced paste UI

### 5.1 Patch queue

- The user can define an **ordered list** of patch identifiers (e.g. filenames such as `0001-foo.patch`, `0002-bar.patch`).
- **Entry methods:** At least one of:
  - Paste a list of names (e.g. from clipboard; one filename per line).
  - Add entries one by one (e.g. type or paste a single name and add to queue).
- **Persistence (optional):** Queue may be persisted in the session (e.g. `sessionStorage`) or kept in memory for the tab so it survives page refresh if desired; the spec leaves this as an implementation choice.

### 5.2 Progress and “current patch”

- Each queue entry has a state: **pending** / **current** / **done** (or equivalent).
- The UI clearly indicates the **next patch to paste** (e.g. name and index: “3 of 12 – 0003-baz.patch”).
- After a successful “Paste next” action, the pasted entry is marked **done** and the **current** pointer advances to the next **pending** item.

### 5.3 One-action paste

- **Primary action:** “Paste next patch” (or “Paste current”).
- **On trigger:**
  1. Read clipboard via `navigator.clipboard.readText()`.
  2. Optionally validate (e.g. non-empty; optional: basic “looks like a patch” heuristic).
  3. Send the content to the terminal using the existing mechanism (e.g. `sendTermData`).
  4. Mark the current queue entry as **done** and advance **current** to the next pending entry.
- **User responsibility:** The user is expected to have copied the patch content on the source machine before clicking “Paste next” in the UI.

### 5.4 Error reduction

- Show the **name of the patch** that will be pasted before the action (and after, as “just pasted”).
- **Optional:** Show approximate size (character count) or line count of clipboard content before sending so the user can sanity-check.
- **Optional:** Simple “patch header” check (e.g. first line contains `diff --git` or `From `) and warn if not.
- **Avoid double-paste:** After “Paste next,” advance immediately; optional confirmation (e.g. “Paste 0003-baz.patch?”) before sending may be offered.

### 5.5 Integration with existing UI

- **Entry point:** Either extend the rtty **context menu** with an item that opens a “Patch paste” panel/drawer, or add a small **toolbar/panel** near the terminal (e.g. “Patch queue” or “Clipboard – Patch mode”) that is visible when the user chooses.
- **Panel/drawer contents:**
  - Queue list: index, name, status (pending / current / done).
  - “Paste next” button (primary action).
  - Optional: “Set queue from clipboard” (paste list of filenames to define the queue).
  - Optional: Clear queue, edit queue (add/remove/reorder).

---

## 6. Non-goals / out of scope

- **Backend changes:** No server-side patch queue is required for this spec; optional future extension may store the queue on the backend.
- **Automatic creation of empty files on the device:** The spec only documents the single command; the UI is not required to execute it on the device.
- **Full file upload of patches:** The primary path is paste-of-content; the existing file upload feature remains as-is and is not replaced.

---

## 7. UI mockup / wireframe (reference)

```
+------------------------------------------+  +---------------------------+
|                                          |  | Patch queue               |
|  Terminal (rtty)                         |  | ------------------------- |
|                                          |  | [1] 0001-foo.patch   done |
|  $ cat > 0003-baz.patch << 'EOF'         |  | [2] 0002-bar.patch   done |
|  (paste here)                             |  | [3] 0003-baz.patch  current|
|                                          |  | [4] 0004-qux.patch pending|
|                                          |  | ...                       |
|                                          |  | [Paste next patch]        |
|                                          |  | (3 of 12)                 |
+------------------------------------------+  +---------------------------+
```

- Terminal occupies main area; patch panel is a small side panel or drawer.
- Queue shows index, filename, status; “Paste next” sends clipboard to terminal and advances.

---

## 8. Implementation notes

- **Frontend-only:** All new behavior can be implemented in the Vue frontend.
- **Reuse:** Use existing clipboard reading and `sendTermData` (or equivalent) in [ui/src/views/device/components/rttyTerm.vue](ui/src/views/device/components/rttyTerm.vue); no WebSocket protocol change is required for paste.
- **Components:** Implement in rttyTerm.vue and/or a new sibling component (e.g. `PatchPastePanel.vue`) used by [ui/src/views/device/rttyPage.vue](ui/src/views/device/rttyPage.vue) or rttyTerm.
- **State:** Queue and current index can live in a composable or component state; optional persistence via `sessionStorage` as noted in 5.1.

---

## 9. i18n

All user-facing strings (e.g. “Paste next patch”, “Patch queue”, “3 of 12”, “Set queue from clipboard”, status labels) must be localizable. Add keys to [ui/src/lang/locales/en.json](ui/src/lang/locales/en.json) and [ui/src/lang/locales/zh.json](ui/src/lang/locales/zh.json) as part of implementation.
