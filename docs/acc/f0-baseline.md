# F0 baseline: Agent Control Center

Reproducibility record for the unmodified Daintree base, run in WSL on 2026-11-18. No application code or dependency was changed. Failures are recorded, not fixed.

**Base commit:** `1c2d4b51df0905cbfdde9977ba8200279241d43d` (2026-10-27, merge of PR #14895). `package.json` says 0.46.0, but the commit is not a release: `v0.46.0` is not an ancestor of it, and `v0.47.0` is the first tag containing it.

**Outcome:** dependencies install, typecheck passes, both requested test files pass, and the agy headless test works. `npm run build:app` runs out of memory, and under `npm run dev` a window opens but the renderer fails to boot.

## Results

| Step | Command | Result | Status |
| --- | --- | --- | --- |
| 1 | `uname -a`, `agy --version`, `node -v`, `git --version`, `gh auth status` | All ran. gh is logged in with scopes `gist`, `read:org`, `repo`, `workflow`. Versions are listed below. | PASS |
| 2 | `gh repo fork daintreehq/daintree --clone=false`, `git clone`, `git remote add upstream` | Fork `DobleRRmatutino/daintree` created and cloned; `upstream` added and fetched. | PASS |
| 3 | `git merge-base --is-ancestor <commit> upstream/develop`, `git switch -c desarrollo <commit>`, `git switch -c feature/f0-baseline desarrollo` | Commit is in upstream `develop`. Both branches start at the exact commit. | PASS |
| 4 | Read the docs, `nvm install 22.23.2` | Node 22.23.2 and npm 10.9.8 installed; checksum matched. Side effect on nvm aliases: see Notes. | PASS |
| 5a | `npm ci` | Exit 0, 122 s. No native module failed. `node-pty`, `win-job-object` and the `better-sqlite3` prebuilds are present. 39 audit findings (31 moderate, 8 high), left alone. | PASS |
| 5b | `npm run rebuild` | Not run: it is conditional on a native module failure, and none failed. | PASS (not needed) |
| 5c | `npm run build:app` | Exit 134, 171 s. First error: `FATAL ERROR: Ineffective mark-compacts near heap limit Allocation failed - JavaScript heap out of memory`. It died in the `tsc` stage at a heap of about 4 GB, so `vite build` and `build:main` never ran. | FAIL |
| 5d | `npm run typecheck` | Exit 0, 54 s. | PASS |
| 6a | `npm test -- electron/services/mcp-server/__tests__/tierAuth.test.ts` | Exit 0, 2 s. 272 of 272 passed. | PASS |
| 6b | `npm test -- shared/config/__tests__/helpAssistantTierAllowlists.test.ts` | Exit 0, 1 s. 18 of 18 passed. | PASS |
| 7a | `timeout -k 10 -s INT 150 npm run dev` | Exit 124 is the time limit I set, not a crash. A window opens: the X server reports a viewable top-level window titled `Daintree`. The renderer does not boot. First error: `[BabelError] Found 1 error: Compilation Skipped: Existing memoization could not be preserved` at `src/components/Sidebar/SidebarViewHost.tsx:68`, followed by `Bootstrap failed: TypeError: Failed to fetch dynamically imported module: http://127.0.0.1:5173/src/App.tsx`. | FAIL |
| 7b | Windows Shell driving a Host in this distro | Steps are below. Distro prerequisites checked: WSL 2, systemd is PID 1, user session `running`. | HUMAN STEP |
| 8 | `agy -p "create a file hello.txt containing ok, then stop" --output-format stream-json` in a worktree of `~/agent-control/sandbox` | Exit 0, 10 s. `hello.txt` exists and contains `ok` with no trailing newline. No approval was requested, because the agy config is `always-proceed`: see Notes. | PASS |
| 9a | Enable Settings > MCP server, read port and key | Not reachable at this commit in this distro: the dev renderer does not boot (7a) and there is no production build (5c). Port 45454 was not listening during the dev run. | BLOCKED, HUMAN STEP |
| 9b | MCP `initialize` POST from WSL | Not run, depends on 9a. Command is below. | BLOCKED |
| 9c | `Test-NetConnection` from Windows | Not run, depends on 9a. Command is below. | HUMAN STEP |
| 10a | Create `runs/test-task/result.md` and an empty `runs/inbox.md` | Both created. Windows path: `\\wsl.localhost\Ubuntu\home\dn\agent-control\runs` | PASS |
| 10b | Add that path to Claude Desktop's filesystem connector and read the file | Needs Claude Desktop on Windows. | HUMAN STEP |

## Blockers

1. **`npm run build:app` runs out of memory.** The script's bare `tsc` runs with Node's default heap. The repo's own `npm run build` wraps the same script with `NODE_OPTIONS=--max-old-space-size=8192`. I did not run it, because it was not part of this task.
2. **The dev renderer does not boot.** React Compiler rejects `src/components/Sidebar/SidebarViewHost.tsx:68`. `vite.config.ts:100` sets `panicThreshold` to `critical_errors` in dev and `none` in builds, so this may only affect dev. That is untested. It was seen in one dev run.
3. **MCP cannot be enabled yet.** Step 9 needs the Settings UI, which blockers 1 and 2 keep out of reach in this distro.
4. **A Windows Shell cannot yet exercise this exact commit.** The GitHub releases checked (`v0.46.0`, `v0.51.0`) list no assets, and Host setup installs the client's exact released version from the release feed. The base commit is not a release.

## Versions

| Component | Version |
| --- | --- |
| OS | `Linux <hostname> 6.18.58.1-microsoft-standard-WSL2 #1 SMP PREEMPT_DYNAMIC Tue Oct 27 00:24:11 UTC 2026 x86_64 x86_64 x86_64 GNU/Linux` (hostname removed because this file is pushed) |
| Distro | Ubuntu (WSL 2, systemd enabled) |
| agy | 1.5.8 |
| Node | 22.23.2 for this work (from `.nvmrc`); the shell default stays 24.14.1 |
| npm | 10.9.8 |
| git | 2.43.0 |
| gh | 2.45.0 |
| Daintree | 0.46.0 in `package.json` |
| Electron | 42.11.18 |

## agy stream-json shape

Output is NDJSON with 12 lines. Each line has an `event` field. The sequence was `init`, then `step_update` lines, then `result`.

| Event | Fields |
| --- | --- |
| `init` | `conversation_id`, `init.cwd`, `init.permission_mode`, `init.tools[]` |
| `step_update` | `step_update.conversation_id`, `.step_index`, `.step_type`, `.state`, `.duration_seconds`, `.text_delta`, `.tool_name`, `.tool_info.name`, `.tool_info.parameters.Path`, `.usage.{input_tokens, output_tokens, thinking_tokens, cache_read_tokens, total_tokens}` |
| `result` | `result.conversation_id`, `.status`, `.response`, `.num_turns`, `.duration_seconds`, `.usage.{input_tokens, output_tokens, thinking_tokens, cache_read_tokens, total_tokens}` |

Values seen: `step_type` was `user_input`, `agent_response` or `tool`; `state` was `ACTIVE` or `DONE`; `result.status` was `SUCCESS`.

## HUMAN STEP: Windows Shell driving a Host in this distro

From `docs/architecture/remote-hosts.md` (WSL section) and `src/components/Settings/Hosts/AddHostDialog.tsx`. Not executed. With a released Windows client this proves the mechanism only, because the Host it installs is that release, not this commit (blocker 4).

1. On Windows, in PowerShell, confirm the distro is listed as version 2: `wsl.exe --list --verbose`
2. Install and open Daintree for Windows.
3. Open Settings > Hosts and choose **Add host**.
4. Under **WSL distributions**, pick `Ubuntu`, and optionally set a **Name**.
5. On the check step, read **Platform**, **Installed**, **Matches this machine** and **Host mode**.
6. Follow the dialog to install the Linux build in the distro and turn Host mode on, then connect.
7. Keep Daintree open on Windows. About 15 s after it quits, WSL can stop the distro, which ends the Host's terminals and agents.

## HUMAN STEP: MCP reachability (after blocker 3 is cleared)

1. With Daintree running, open Settings > MCP server, turn it on, and note the port (default 45454, incremented on collision) and the **API key**.
2. Run this in WSL. It prompts for the key without echoing it and prints only the HTTP status and whether the session header came back:

```bash
read -rs "DAINTREE_MCP_KEY?MCP key: "; echo
PORT=45454
printf 'Authorization: Bearer %s\n' "$DAINTREE_MCP_KEY" | curl -sS -o /dev/null -D - -X POST "http://127.0.0.1:$PORT/mcp" -H @- -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"f0-baseline","version":"0"}}}' | tr -d '\r' | awk 'NR==1{print "status:", $2} tolower($1)=="mcp-session-id:"{print "mcp-session-id: present"}'
unset DAINTREE_MCP_KEY
```

The `read` syntax is for zsh, this distro's shell. No `mcp-session-id` line means the header did not come back.

3. Run this in PowerShell on Windows, with the same port. The server binds to loopback inside WSL, so this tests whether WSL forwards that port to Windows:

```powershell
Test-NetConnection -ComputerName 127.0.0.1 -Port 45454
```

## HUMAN STEP: runs folder

Add `\\wsl.localhost\Ubuntu\home\dn\agent-control\runs` to Claude Desktop's filesystem connector, then confirm it can read `test-task\result.md`.

## Notes

- **agy ran without approval prompts because of existing user config.** `~/.gemini/antigravity-cli/settings.json` sets `toolPermission` to `always-proceed`, and the `init` event reported `permission_mode: always-proceed`. `--dangerously-skip-permissions` was not passed, but the effect is equivalent. So this run does not show how agy behaves when approval is required. The setting was not changed.
- **`nvm install` changed nvm aliases outside this folder.** It refreshed the LTS aliases, and the default Node stopped resolving in new shells. `~/.nvm/alias/lts/*` and `~/.nvm/alias/lts/krypton` were restored to `lts/krypton` and `v24.14.1`, and a clean shell resolves v24.14.1 again. The other LTS aliases keep their refreshed values. The next `nvm install` or LTS lookup may break the default the same way.
- **Electron's binary was absent after `npm ci`.** `node_modules/electron/dist` did not exist until `npm run dev` first launched Electron, when it appeared.
- **The dev run wrote outside `~/agent-control`.** `~/.config/daintree-dev` and `~/.daintree` exist after it; this is the app's normal data location.
- **`desarrollo` is local only.** Only `feature/f0-baseline` is pushed, and it contains the same base commit.

## F0.1: build fix and base selection

Run on 2026-11-18. No application code, dependency, branch base or agent config was changed. `npm run build` now passes on the pinned snapshot. On `v0.51.0`, everything requested passes and the app reaches its main UI under `npm run dev`, though startup took minutes.

| Step | Command | Result | Status |
| --- | --- | --- | --- |
| 1 | `free -h`, `nproc`, read `.wslconfig` on the Windows side | 15 GiB RAM with 13 GiB available, 4 GiB swap, 12 CPUs. No `.wslconfig` in any folder under `/mnt/c/Users`. | PASS |
| 2 | `npm run build` on `1c2d4b51df` | Exit 0, 292 s. No error and no out-of-memory, so no `.wslconfig` change is needed. | PASS |
| 3 | `git fetch upstream --tags`, list tags, read `CHANGELOG.md` | Newest stable tag is `v0.51.0`; GitHub marks it neither draft nor prerelease. `CHANGELOG.md` and `package.json` both say 0.51.0. No `-rc` or `-beta` tags exist. | PASS |
| 4a | `git clone ~/agent-control/daintree ~/agent-control/daintree-release`, `git checkout v0.51.0` | Both exit 0, at `33813085dd2c96fd74094b3969524f0e41897cdd`. | PASS |
| 4b | `nvm use` | Selected 22.23.2 from the tag's `.nvmrc`, already installed. nvm aliases are byte-identical before and after. | PASS |
| 4c | `npm ci` | Exit 0, 60 s. | PASS |
| 4d | `npm run build` | Exit 0, 133 s. | PASS |
| 4e | `npm run typecheck` | Exit 0, 51 s. | PASS |
| 5a | `npm run dev`, 200 s limit | Window opened. Still on the loading skeleton with "Daintree is taking longer than expected" at the last capture (140 s). | Not conclusive |
| 5b | `npm run dev`, 280 s limit | Window opened. `Renderer loaded` was logged, but the window was still on the skeleton at the last capture (about 255 s). | Not conclusive |
| 5c | `npm run dev`, 600 s limit | Window opened and the main UI was visible at the first capture (240 s): the "Welcome to Daintree" screen, no error screen. It stayed up through the last capture (540 s), with 0 errors logged before shutdown. | PASS |
| 6 | Compare the React Compiler error | **No**, it does not appear on `v0.51.0`: 0 `BabelError` lines in all three runs. First error line on `1c2d4b51df`: `[vite] 7:58:23 PM [vite] (client) Pre-transform error: [BabelError] Found 1 error:` | PASS |
| 7a | Enable Settings > MCP server on `v0.51.0` | Steps are below. | HUMAN STEP |
| 7b | `initialize` from WSL and `Test-NetConnection` from Windows | Commands are below. Not run, because they need the key from 7a. | HUMAN STEP |
| 8 | Recommendation | Base `desarrollo` on `v0.51.0`, after a re-audit. No branch was moved. | PASS |

### Tags

| Tag       | Date       |
| --------- | ---------- |
| `v0.51.0` | 2026-11-13 |
| `v0.50.0` | 2026-11-10 |
| `v0.49.0` | 2026-11-06 |
| `v0.48.0` | 2026-10-30 |
| `v0.47.1` | 2026-10-29 |
| `v0.47.0` | 2026-10-28 |
| `v0.46.0` | 2026-10-22 |
| `v0.45.0` | 2026-10-19 |
| `v0.44.0` | 2026-10-17 |
| `v0.43.0` | 2026-10-11 |

### How the dev runs were judged

- Each run was stopped by my time limit, so exit 124 is expected and is not an app failure.
- I set `DAINTREE_DEV_USER_DATA_DIR` to a folder under `daintree-release/.tmp/f01/` to keep app data under `~/agent-control`. The app therefore started as a first run, with no existing projects or settings.
- UI state comes from captures of the Daintree window only. They were not committed.
- Startup is slow and its time was not measured precisely. The main UI appeared within 240 s in run 5c but had not appeared at about 255 s in run 5b.
- The window in 5c showed a notice that hardware acceleration was disabled after repeated GPU crashes. In 5a and 5b the logged GPU crashes (`launch-failed`) begin only after my time limit sent SIGINT, and 5c logged none before its shutdown. The notice comes from that test method, and the flag lives in the throwaway data folder.

### Recommendation

Base `desarrollo` on **`v0.51.0`** (`33813085dd2c96fd74094b3969524f0e41897cdd`).

| Evidence | `1c2d4b51df` (audited snapshot) | `v0.51.0` |
| --- | --- | --- |
| Stable release | No | Yes |
| `npm ci` | Pass | Pass |
| `npm run build` | Pass, 292 s | Pass, 133 s |
| `npm run typecheck` | Pass | Pass |
| `npm run dev` reaches the main UI | No, React Compiler error stops the renderer | Yes |
| Contains `87429c7912` ("clear React Compiler errors that break dev boot"), which also changed `SidebarViewHost.tsx` | No | Yes |
| Electron | 42.11.18 | 45.2.3 |
| Vite | 8.0.16 | 8.3.6 |

The cost is the audit. The tag is 4,833 commits after the audited snapshot and changes 5,556 files. The MCP server code alone has 86 changed files (+20,305, -1,261), so the audit of `1c2d4b51df` does not carry over and that area needs reviewing again. MCP reachability on the tag is also still untested.

### HUMAN STEP: enable MCP on `v0.51.0`

1. Start the app in WSL, and allow several minutes. The "taking longer than expected" card can appear while it loads; do not press its buttons.

```bash
cd ~/agent-control/daintree-release
nvm use
DAINTREE_DEV_USER_DATA_DIR="$HOME/agent-control/daintree-release/.tmp/f01/userdata-mcp" npm run dev
```

2. In the app, open Settings > MCP server and turn it on. Note the port (default 45454, incremented on collision) and the **API key**. Keep the app running.
3. In a second WSL terminal, run this. It reads the key without echoing it and prints only the HTTP status and whether `mcp-session-id` came back:

```bash
read -rs "DAINTREE_MCP_KEY?MCP key: "; echo
PORT=45454
printf 'Authorization: Bearer %s\n' "$DAINTREE_MCP_KEY" | curl -sS -o /dev/null -D - -X POST "http://127.0.0.1:$PORT/mcp" -H @- -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"f0-baseline","version":"0"}}}' | tr -d '\r' | awk 'NR==1{print "status:", $2} tolower($1)=="mcp-session-id:"{print "mcp-session-id: present"}'
unset DAINTREE_MCP_KEY
```

No `mcp-session-id` line means the header did not come back.

4. In PowerShell on Windows, with the same port:

```powershell
Test-NetConnection -ComputerName 127.0.0.1 -Port 45454
```

`TcpTestSucceeded : True` means the port is reachable from Windows.

### Created or changed outside `~/agent-control`

| Path | What |
| --- | --- |
| `~/.daintree/cli-control.json` | Rewritten by the app during the dev runs. Not opened, because it may hold a credential. |
| `~/.npm/_cacache`, `~/.npm/_logs` | npm cache and logs from `npm ci` and `npm run` |
| `~/.electron-gyp/45.2.3` | Electron headers for native module builds |
| `~/.cache/electron/electron-v45.2.3-linux-x64.zip` | Electron download, 124 MB |
| `~/.cache/ffmpeg-static-nodejs` | Cached download from dependency install |
| Claude Code's session temp folder under `/tmp` | Helper scripts, the agy output from F0 and the window captures. Not committed. |

Claude Code's own `~/.claude.json` and its package under nvm's Node 24.14.1 also changed during this session; no command in this task touched them. `~/.gemini` was not touched, and no Node version was installed.
