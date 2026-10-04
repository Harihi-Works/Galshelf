# GalShelf 0.3 Development Notes

Status: Draft  
Started: 2026-10-05

This document tracks confirmed issues and planned engineering work for the GalShelf 0.3 development cycle.

## P0 — Broken launch-file relocation in packaged build

### Summary

In GalShelf 0.2.8, when a saved game launch path is no longer valid and the user chooses to relocate the executable, the packaged desktop build can fail with:

```text
No module named 'tkinter'
```

The UI then also surfaces an `Internal Server Error`. The same error dialog includes a generic note saying Microsoft Edge WebView2 Runtime is required, which is misleading in this case because the application UI itself is already running inside WebView2.

This is a P0 item for 0.3 because users who reinstall Windows, move games, reconnect storage, or rebuild their library must be able to repair stale launch paths reliably.

### Observed user flow

1. A game's previously saved executable path no longer exists or its disk is unavailable.
2. GalShelf correctly reports that the saved launch file cannot be found.
3. The user clicks "Relocate" / "重新定位".
4. The backend attempts to open a file picker.
5. The packaged runtime raises `No module named 'tkinter'`.
6. The backend request fails and the frontend shows `Internal Server Error`.
7. The dialog additionally shows a generic WebView2 requirement message that does not describe the actual failure.

### Likely root cause

The relocation flow appears to depend on Python `tkinter` for a native file/folder picker, but the frozen/release runtime does not include the required Tk/Tcl components.

This must be confirmed in code before implementation. Do not treat the hypothesis as a substitute for tracing the actual packaged execution path.

### Required fix

- Remove the hard runtime dependency on bare `tkinter` from the relocation flow where practical.
- Prefer a reliable Windows-native picker or another picker implementation that is explicitly supported by the packaged desktop architecture.
- If Tk is intentionally retained, the release build must package and validate all required Tcl/Tk runtime components.
- Picker launch failures must be converted into a typed, user-readable application error rather than an HTTP 500.
- Do not show the generic "Microsoft Edge WebView2 Runtime required" message for unrelated backend exceptions.
- Keep the existing stale-path detection, but make the recovery path complete:
  - choose replacement executable;
  - validate it;
  - update the game directory when appropriate;
  - persist the new launch path;
  - refresh the detail page without restarting GalShelf.
- Preserve non-ASCII / Japanese paths and long Windows paths.
- Canceling the picker must be a normal no-op and must not produce an error toast.

### Error handling requirements

The frontend should distinguish at least:

- saved executable missing;
- picker could not be opened;
- selected file is invalid or unsupported;
- selected path is inaccessible;
- save/update failed;
- user canceled selection.

Backend implementation details and raw Python exceptions must be written to the GalShelf log, but the user-facing dialog should contain a concise actionable message.

### Acceptance criteria

1. Reproduce the original failure on a packaged 0.2.8-compatible environment before the fix, or provide equivalent regression coverage if the old package is unavailable.
2. In the fixed packaged build, clicking "Relocate" opens a usable file picker without requiring a separately installed Python/Tk environment.
3. Selecting a valid game executable updates the stored launch path and the game can be launched from GalShelf afterward.
4. Paths containing Japanese characters, spaces, punctuation, and deep directory nesting work correctly.
5. Canceling the picker produces no error.
6. A picker/backend failure produces a specific application error and never a generic `Internal Server Error` toast.
7. The WebView2 requirement message is shown only when WebView2 is actually unavailable or initialization fails.
8. Automated regression coverage exists for the backend/API error mapping and path update logic.
9. The packaged Windows smoke test explicitly exercises the relocation/picker flow, not only source-tree development mode.

### Release gate

Do not mark this item complete based only on unit tests or running from a Python development environment. Acceptance requires verification against the actual packaged Windows artifact.

### Evidence

First reported from a GalShelf 0.2.8 Windows packaged build on 2026-10-05. The visible failure included `No module named 'tkinter'` together with an `Internal Server Error` notification while attempting to repair a missing launch-file path.
