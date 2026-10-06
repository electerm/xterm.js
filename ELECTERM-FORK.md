# electerm fork of xterm.js

This repository is the [electerm](https://github.com/electerm/electerm) fork of
[xterm.js](https://github.com/xtermjs/xterm.js). It carries one small fix and is published to npm as
[`@electerm/xterm`](https://www.npmjs.com/package/@electerm/xterm).

## The branch

`fix/mouse-report-not-user-input`

**Base:** `904ae935269eef5ec6a1415b64463c3d02eff1eb` — verified to be the exact source of the
published `@xterm/xterm@6.1.0-beta.292` (105 of the 106 files in the browser bundle's sourcemap are
byte-identical; the only difference is `src/common/Version.ts`, which the publish script rewrites
from `package.json`). Pinning the base to the release electerm already uses keeps the diff to the fix
alone, so a behaviour change can be attributed.

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

**Why.** A mouse report is not text input. Sending it with `wasUserInput = true` makes `CoreService`
fire `onUserInput`, and `SelectionService` clears the selection on that event — `clearSelection()`
also calls `_removeMouseDownListeners()`, so a drag in progress is aborted, not merely
de-highlighted. The default-encoding branch immediately above already uses `triggerBinaryEvent()`,
which never fires `onUserInput`, so the two encodings disagreed about the same event and only the SGR
path (the one tmux enables) destroyed selection.

**What this does *not* fix.** It does not give you "hold the button and wheel to extend a selection
across screens" under `tmux set -g mouse on`. Measured against a real tmux, the selection was never
being cleared in that scenario — `getSelectionPosition()` is constant across wheel ticks and
`onSelectionChange` never fires with an empty selection. tmux runs the terminal in the alternate
buffer and owns the wheel; each wheel repaints the screen **in place**, so an xterm.js selection
stays pinned to fixed *screen rows* while the text slides out from under it. Whoever owns the wheel
must own the selection, so that gesture has to be tmux's own (press → drag → wheel → release), which
tmux delivers to the terminal via OSC 52. See electerm's
`src/client/components/terminal/osc52-addon.js`.

`scrollOnUserInput` is unaffected in practice: while mouse reporting is active the local viewport
cannot be scrolled at all (`_handleWheel` always prevents default, and `_handlePassiveWheel` returns
early once the protocol requests wheel events). `WriteBuffer.handleUserInput()` is only a "flush on
the next frame" latency hint.

## Publishing

Published as **`@electerm/xterm`** only. Version scheme is `<upstream version>-electerm.<n>`, e.g.
`6.1.0-beta.292-electerm.1` — it keeps the upstream base obvious, leaves room to re-release without
rebasing, and still satisfies the `^6.1.0-beta.292` peer range that every `@xterm/addon-*` declares.

```bash
npm run publish-fork          # npm publish --access public --tag beta
npm run publish-fork -- --dry-run
```

`prepublishOnly` runs `npm run package` (tsc + webpack + esbuild), so `lib/` is built from source at
publish time.

**Do not use `bin/publish.js`.** It is upstream's release orchestrator: it also publishes
`headless/` (the `@xterm/headless` package name), every addon whose files changed, and on a
"stable" release runs `bin/update-website.sh`. From a fork that means trying to publish packages
owned by the xterm.js org, which 403s. `.github/workflows/release.yml` has been rewritten for the
same reason — it is manual-only and defaults to a dry run.

`.npmignore` whitelists exactly `lib/`, `css/`, `typings/` and `src/` (minus tests), so no addons or
`headless/` end up in the tarball.

**One-time npm setup:** the `@electerm` scope has to exist on npm and the publisher needs rights to
it, then configure a trusted publisher (npmjs.com → package → Settings → Trusted publisher) for
repository `electerm/xterm.js` and workflow `release.yml`. Until the package exists, npm offers a
"pending trusted publisher" for it.

## Consuming it

Use an **npm alias**, so the addons' `@xterm/xterm` peer resolves to this package and only one copy
of xterm.js is installed:

```json
"devDependencies": {
  "@xterm/xterm": "npm:@electerm/xterm@6.1.0-beta.292-electerm.1"
}
```

Verified: with the alias, `npm install` adds exactly two packages and `npm ls @xterm/xterm` reports
the addon's peer as `deduped` onto the alias — no second, upstream copy.

Do **not** point a `github:` or `git+https://` spec at this repo. npm rewrites the shorthand to
`git+ssh://` in `package-lock.json`, which CI cannot fetch.

## Rebase onto a new upstream release

```bash
git fetch upstream            # github.com/xtermjs/xterm.js
git rebase <upstream release tag or commit>
# resolve only src/browser/services/MouseService.ts if it conflicts
# then bump both places and republish:
#   package.json + package-lock.json + src/common/Version.ts  -> <new version>-electerm.1
npm run publish-fork -- --dry-run
```

Sanity check that the fork carries nothing but the fix: rebuild and diff the bundles against the
published upstream ones — they should differ by exactly one character
(`triggerDataEvent(t,!0)` → `triggerDataEvent(t,!1)`) with identical byte lengths.

## `lib/` is committed (for now)

Upstream's `.gitignore` excludes `lib/`, and the root `package.json` has no `prepare` script, so a
plain **git** dependency installs a package with no `main`. `lib/` is therefore committed here
(`git add -f lib`) to keep the git-dependency path working.

Once electerm has switched to the npm alias above, that fallback is dead weight and `lib/` can be
dropped from git (`git rm -r --cached lib`) — `prepublishOnly` already builds it for publishing.
