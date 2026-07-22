# AGENTS.md — Bunny Tea Parade

Instructions for AI agents (and humans) working in this repository.

## 1. Repository identity

| Field | Value |
|---|---|
| Remote (canonical) | `https://github.com/ekaphop-won/phaser-rhythm-mini-game.git` |
| Branch | `main` |
| Local path | `/Users/jojo/Github/project-phaser-music-game` |
| Default branch on `origin` | `main` |

> **Authoritative source for version-control configuration:** `.git/config` in the
> working tree. Every remote URL, credential helper, signing key, and branch
> mapping below is read from that file — re-derive it from there, do not hard-code.

Read it with:
```bash
git config --list                          # all effective config
cat .git/config                            # local repo config only
git remote -v                              # remote URLs
```

## 2. Author + signing

| Field | Value |
|---|---|
| Committer name | `Jojo` |
| Committer email | `w.ekaphop@gmail.com` |
| GPG format | `ssh` |
| Signing program | `/Applications/1Password.app/Contents/MacOS/op-ssh-sign` (1Password SSH agent) |
| Signing key | `ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIBXQhTIKUvTT1n3rfCvR2VCRlXGXEPOFZNk4tXWcD6om` |
| `commit.gpgsign` | `true` (signing enforced) |

If the 1Password agent is unavailable, sign-commits will fail. Fall back to:
```bash
GIT_TERMINAL_PROMPT=0 git -c gpg.format=openpgp -c commit.gpgsign=false commit
```
…and explain why in the commit body. Do not silently disable signing for convenience.

## 3. Authentication

- **HTTPS remote**, credentials in `remote.origin.url` (PAT embedded).
- Helper chain (from `.git/config`):
  - `credential.https://github.com.helper=!/opt/homebrew/bin/gh auth git-credential`
  - `credential.helper=osxkeychain` (fallback)
- `gh` CLI auth lives in the system keychain — `gh auth status` to verify.
- **Do not print or echo the embedded PAT** in any log/chat/screenshot.

If a push returns 403/404 and `gh auth status` shows correct scopes, the per-repo
ACL on the GitHub side is the real problem (token may have `pull` only on this
specific repo). Tell the user to grant Collaborator access on the repo, then
retry — do not invent alternative routes.

## 4. Tech stack

- **Phaser 3.80.1** via CDN with SRI hash (no bundler)
- Plain HTML/CSS/JS — **no build system**
- Web Audio API (custom `AudioManager`; Phaser audio is `noAudio: true`)
- Beatmap data: inline `BN_L` / `BN_T` arrays in `index.html` + external JSON
  in `audio/bunny-tea-parade-adjusted.json`
- Static server: `python3 -m http.server 8765 --bind 127.0.0.1`
- `node --check` for inline JS syntax validation

## 5. File map (what to touch for what)

| File | Role | When to edit |
|---|---|---|
| `index.html` | Game + all inline scene code (Phaser, AudioManager, embedded beatmap) | Gameplay, scoring, VFX, Settings overlay, scene lifecycle |
| `beatmap-editor.html` | No-build editor (waveform, lanes, drag/snap, export) | Editor features, beatmap import/export, parseIndex regex |
| `audio/bunny-tea-parade-adjusted.json` | External beatmap (149 notes) | Beatmap content, markers, sections |
| `audio/bunny-tea-parade-adjusted.mp3` | Audio asset (60.000 s) | Replace only — verify duration matches `SONG_DURATION_MS` |
| `IDEA.md` | Thai/English game design notes | Design intent, controls, scoring philosophy |
| `CHANGELOG.md` | Release notes (Keep a Changelog 1.1.0) | Every shipped change set |
| `README.md` | User-facing overview | Controls, features, limitations that still apply |
| `AGENTS.md` | This file | Agent operating rules |
| `.gitignore` | VCS exclusions | New artifact types (e.g. `node_modules/`, `__pycache__/`) |
| `.git/config` | **Source of truth for git config** | Re-read before any version-control operation |

## 6. Operational rules for agents

### 6.1 Before editing
1. `git status --short` — know the working-tree state.
2. `git log --oneline -10` — know what's on `main` locally vs. `origin/main`.
3. **Re-read `.git/config`** before any push, release, or tag operation.
4. `node --check` both inline `<script>` blocks after any edit (extract them first
   or use a small Python helper — see §6.5).

### 6.2 Settings overlay (a Phaser 3.80.1 footgun this repo already paid for)

- Phaser config **must** include `dom: { createContainer: true }` —
  without it, `this.add.dom()` throws and the entire scene hangs. The game will
  not boot past start, and there is no error in the Phaser console.
- The settings overlay is **not** a `this.add.dom()` — it is a `Phaser.GameObjects.Container`
  at scene-coord `(cx, cy)`. Always:
  - `this.settingsOverlay = this.add.container(cx, cy).setDepth(70)`
  - Add **both** `wrap` (text/graphics) and `hitZone` (`this.add.zone()`) to the
    container. Bare `this.add.zone()` is **scene-level** — it ignores the
    container's transform, so taps land off-screen.
- Auto-pause: `openSettings()` captures `_settingsWasPlaying`, sets `isPaused=true`,
  and stops audio. `toggleSettings()` (the unified close path) restores both.
- `toggleSettings()` is the **single source of truth** for opening/closing.
  Do not split Space/Esc handling — both should call `toggleSettings()`.

### 6.3 Web Audio

- Browser user-gesture rule: `audio.startSong(0)` will not run from a script
  in a background tab/headless context. Use a Playwright `mouse.click()` (which
  counts as a gesture) before any audio test.
- Audio state during pause: `audio.stopSong()` on pause, `audio.startSong(t)`
  on resume with the captured `pauseStartedAt`. `pauseDelta` accumulates elapsed
  pause time so `gameTime` stays continuous.

### 6.4 No-build verification

- `node --check` does not catch Phaser runtime bugs. Always pair syntax check
  with a runtime exercise (see `verify-without-test-runner` skill pattern).
- The `?debug` query string is the project's gate to expose `window.__game`
  for headless verification. Use `page.evaluate()` on `window.__game.scene.getScene('GameScene')`
  to drive state without DOM dispatch.
- For Phaser input, drive the scene input system directly
  (`scene.input.activePointer` + `zone.emit('pointerdown', …)`) — DOM
  `dispatchEvent` does **not** satisfy `event.isTrusted=true` for the Web Audio
  unlock heuristic, and DOM pointer events don't reach Phaser's hit-test.

### 6.5 Inline script extraction

`index.html` has two `<script>…</script>` blocks (one in `<head>`, one in
`<body>`). To validate with `node --check`:

```bash
python3 - <<'EOF'
import re
src = open('index.html').read()
for i, b in enumerate(re.findall(r'<script>([\s\S]*?)</script>', src), 1):
    open(f'/tmp/btp-block-{i}.js', 'w').write(b)
    print(f"block {i}: {len(b)} chars")
EOF
for f in /tmp/btp-block-*.js; do node --check "$f" && echo "$f: PARSE OK"; done
```

### 6.6 GitHub Release workflow

- `CHANGELOG.md` is the source of truth for release notes. The GitHub Release
  UI object is sugar on top — GitHub's "Create release from tag" reads
  `CHANGELOG.md` directly.
- If `gh release create` returns 404 or 403 with the token showing
  `'pull': True, 'push': False, 'admin': False` for the specific repo, the
  per-repo ACL is the issue — the OAuth scopes of the user are irrelevant.
  Stop and tell the user to fix the grant on GitHub.com.

## 7. Conventions

- **No build system.** Don't introduce a bundler, a transpiler, or a package.json
  unless explicitly asked.
- **No new dependencies** without a heads-up — this is a static HTML game,
  vendoring anything increases the surface area for no real gain.
- **Thai language in UI strings** is the default. English-only strings should
  be marked as such in code comments.
- **Inline comments in Thai are fine**; AI agent reasoning should be in English
  or Thai depending on the user's language.
- **Prefer 4-space indentation** in JS, 2-space in CSS — match the file you're
  editing.
- **`type="text/css"` and inline `<style>`** are first-class. Do not extract
  them to external files unless the project structure changes.

## 8. Tags and releases

The `v1.0-ux` tag points at commit `d195384c`. The full release artifact
process is:

1. Land UX changes on `main`, push via HTTPS remote in `.git/config`.
2. Update `CHANGELOG.md` (Keep a Changelog 1.1.0) — commit separately.
3. Refresh `README.md` for any changed controls or shipped limitations.
4. `git tag -a vN.M-x -m "..."` locally.
5. `git push origin vN.M-x` (the tag goes through the same remote as commits).
6. `gh release create vN.M-x --target <sha> --title "..." --notes-file CHANGELOG.md`
   (only if the token has push on the repo; otherwise step 5 is the ceiling).

## 9. What this file is NOT

- Not a substitute for `.git/config` — `.git/config` is authoritative.
- Not a CI config (there is no CI yet).
- Not a code review checklist — see `06_skills/requesting-code-review` in the
  Mochi profile for that.

---

*This file is read by every agent that opens this repo. Keep it short, keep it
accurate, and re-derive anything that can drift (remote URL, signing key,
release tags) from the filesystem before acting on it.*
