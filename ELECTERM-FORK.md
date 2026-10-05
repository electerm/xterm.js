# electerm fork of xterm.js

This repository is the [electerm](https://github.com/electerm/electerm) fork of
[xterm.js](https://github.com/xtermjs/xterm.js). It exists to carry one fix, and to be consumed
directly as a dependency while that fix is not in a release.

## The branch

`fix/mouse-report-not-user-input`

**Base:** `904ae935269eef5ec6a1415b64463c3d02eff1eb` — verified to be the exact source of the
published `@xterm/xterm@6.1.0-beta.292` (105 of the 106 files in the browser bundle's sourcemap are
byte-identical; the only difference is `src/common/Version.ts`, which the publish script rewrites
from `package.json`). Pinning the base to the release electerm already uses keeps the diff to the fix
alone, so a behaviour change can be attributed.

`package.json` and `Version.ts` are set to `6.1.0-beta.292` so the peer range
`^6.1.0-beta.292` declared by the `@xterm/addon-*` packages still resolves.

## The change

`src/browser/services/MouseService.ts`, in `_triggerMouseEvent`:

```diff
       if (this._mouseStateService.isDefaultEncoding) {
         this._coreService.triggerBinaryEvent(report);
       } else {
-        this._coreService.triggerDataEvent(report, true);
+        this._coreService.triggerDataEvent(report, false);
       }
```

**Why.** Sending a mouse report with `wasUserInput = true` makes `CoreService` fire `onUserInput`,
and `SelectionService` clears the selection on that event. `clearSelection()` also calls
`_removeMouseDownListeners()`, so a drag in progress is aborted — not merely de-highlighted.

That is user-visible with `mouseEventsRequireAlt`: the option deliberately leaves the wheel ungated
so it keeps scrolling the application (tmux scrollback), but the resulting wheel report then destroys
the terminal-side selection the user is dragging out, so a selection cannot be extended across
screens.

The default-encoding branch immediately above already uses `triggerBinaryEvent()`, which never fires
`onUserInput` — so the two encodings disagreed about the same event, and only the SGR path (the one
tmux enables) broke selection.

`scrollOnUserInput` is unaffected in practice: while mouse reporting is active the local viewport
cannot be scrolled at all (`_handleWheel` always prevents default, and `_handlePassiveWheel` returns
early once the protocol requests wheel events). `WriteBuffer.handleUserInput()` is only a "flush on
the next frame" latency hint.

## Built output is committed

Upstream's `.gitignore` excludes `lib/`, and the root `package.json` has **no `prepare` script**, so
a plain git dependency would install a package with no `main`. `lib/` is therefore committed here
(`git add -f lib`). It was produced with the repository's own toolchain:

```bash
npm install --ignore-scripts     # native/browser postinstalls are not needed for the build
npm run package                  # tsc + webpack + esbuild --prod
```

The rebuilt `lib/xterm.js` and `lib/xterm.mjs` differ from the published `6.1.0-beta.292` bundles by
**exactly one character** (`triggerDataEvent(t,!0)` → `triggerDataEvent(t,!1)`); byte lengths are
identical. That is the intended way to check this fork is not carrying anything else.

## Using it

```json
"devDependencies": {
  "@xterm/xterm": "github:electerm/xterm.js#fix/mouse-report-not-user-input"
}
```
