# repo-guard

Scans a git repository for code that **runs itself when you open the folder** in VS Code / Cursor
(or when you run `npm install`), before any of it touches your disk.

Built after a real incident (Sep 2026): a client's repos contained a hidden editor task that, the moment
the folder was opened in Cursor, ran a JavaScript loader disguised as a font file
(`public/fonts/fa-solid-500.woff2`). It connected to a remote server, ran whatever code it received,
and watched the clipboard. The victim saw nothing, because the task hid its own output.

## Install

Needs only Python 3 and git (both come with macOS developer tools).

```sh
curl -fsSL -o /usr/local/bin/repo-guard https://raw.githubusercontent.com/Ankitsaroj94/repo-guard/main/repo-guard
chmod +x /usr/local/bin/repo-guard
repo-guard harden        # one-time: stop VS Code / Cursor from auto-running tasks
```

## Use

```sh
# Instead of `git clone`: files are only written to disk if the scan is clean
repo-guard clone https://github.com/someone/project.git

# Check a repo you already have (do this BEFORE opening it in an editor)
repo-guard scan path/to/project
```

Exit code is `1` when anything HIGH or CRITICAL is found, so it also works in scripts and CI.
Add `--json` for machine-readable output.

## What it catches

| Check | Why |
|---|---|
| Tasks with `"runOn": "folderOpen"` in `.vscode/`, `.cursor/`, `*.code-workspace` (including hidden in `launch.json` / `settings.json`) | The exact trick used: runs a command as soon as you open the folder |
| Editor or npm commands that execute a non-code file (`node font.woff2`) | Real fonts and images are never executed |
| Fonts, images, PDFs, archives that are actually text/JavaScript | The payload was hidden as a `.woff2` |
| Code pushed off-screen by hundreds of tabs/spaces | Hides the payload if you glance at the file |
| `preinstall` / `postinstall` / `prepare` scripts in `package.json` | Run automatically on `npm install` |
| `devcontainer.json` `initializeCommand` | Runs on your machine, not in the container |
| Code that combines: global `require` aliasing, `child_process`, `eval`, hard-coded IP URLs, blockchain/Telegram lookups, clipboard access, browser password / wallet / SSH key paths | Typical loader and info-stealer behaviour |
| Known indicators from this campaign (C2 IP, Ethereum address, marker string) | Direct match |

## `harden`

Sets these in VS Code / Cursor / Windsurf / VSCodium user settings (a `.repo-guard.bak` backup is kept):

```json
"task.allowAutomaticTasks": "off",
"security.workspace.trust.enabled": true
```

Even with this, open unfamiliar repos in **Restricted Mode** when the editor asks whether you trust the authors.

## Limits

A scanner reduces risk; it cannot prove a repo is safe. It checks the default branch (use `-b` for others),
does not inspect dependencies inside `node_modules` / pub / CocoaPods, and a determined attacker can
write code that avoids these patterns. Treat unexpected "test tasks" and repos from job offers with suspicion.

**If you think you were already hit:** see what processes Node is running (`ps aux | grep node`),
then from another device change passwords, sign out of all sessions, and rotate API keys and SSH keys.
