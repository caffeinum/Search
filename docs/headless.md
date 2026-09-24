# Headless Search, and a CLI for agents

A proposal, not a build. Agents drive Search with no window on anybody's screen; the person signs in once, in a real Search window, and everything after that is headless.

Search already has most of the substrate: `./bench` and `Bench.swift` speak a JSON-per-line protocol over a Unix socket, per world, with off-screen bench tabs, `open/wait/text/eval/click/type/submit/shot`, real mouse and key events (`tap`, `key`), and extension control. This proposal is about the gaps between that dev tool and something agents can use every day.

## What was measured

All on this Mac, macOS 27, WKWebView from the system, 1280×800, a page counting `requestAnimationFrame` calls and a 10 ms `setInterval` for 2 s:

| Setup | `visibilityState` | rAF / 2 s | 10 ms ticks / 2 s |
|---|---|---|---|
| No window at all | hidden | 0 | 8 |
| Off-screen borderless window (what `Bench.makeRoom` does today) | hidden | 0 | 8 |
| Same, `_setWindowOcclusionDetectionEnabled:` **false** on the web view | **visible** | 119 | 167 |
| Same, but `NSApp.hide` (what `open -j` gives you) | hidden | 0 | 8 |
| SPI off, no window | hidden | 0 | 8 |

Activation policy (`.regular`, `.accessory`, `.prohibited`) made no difference. What matters is a window plus occlusion detection turned off, with the app not hidden. `document.hasFocus()` stays false in every setup.

So bench tabs today are backgrounded pages: no rAF, timers clamped. `personal` saw the same `hidden` in the installed app. The fix is one call in `Bench.house(_:)`. `Bench`'s `picture` already makes that call, so the SPI is already in the product.

A ~40-line windowless harness (`.prohibited`, off-screen window, SPI off, Safari user agent) loaded Hacker News, built the element tree below (227 refs, 6 KB against 39 KB of HTML), clicked `new` by ref and snapshotted the next page. It took 0.95 s wall and 0.13 s CPU.

## 1. Architecture

**Search.app is the daemon.** No separate engine binary. A world's cookies live in a `WKWebsiteDataStore(forIdentifier:)` that belongs to the app's bundle (`~/Library/WebKit/com.officecommun.search/WebsiteDataStore/<id>`). A second binary gets a store of its own, and then the login the person did in Search isn't there. The same goes for the ad shield, extensions and the keychain passwords, which are all tied to the app's identity. So headless is a mode of the app, not a new program.

- **One process per world**, as now. The socket path is per world.
- **Headless launch**: `SEARCH_HEADLESS=1` (or `--headless`). It sets `.accessory` (no Dock icon or menu bar, but it can still show a window when login needs one), makes no browser window, turns the bench on, and keeps only the room. `.prohibited` can never show a window, so it is the wrong policy for a process that sometimes needs one for login. Never `NSApp.hide`: the table shows it throttles pages again.
- **The room** keeps its off-screen window and gains the occlusion call in `house(_:)`. Bench tabs are then `visible` and run at full speed.
- **GUI and headless are the same process.** If the person opens Search on a world that is running headless, LaunchServices reactivates the running process. It should then switch to `.regular` and summon its window, not start a second process on the same session files. This is the riskiest part of the work (see §5).
- **The CLI is a thin client.** It connects to the world's socket. If nothing is listening, it launches `Search.app` headless for that world (`open -n -g --env SEARCH_HEADLESS=1 --env SEARCH_PROBE=<world>`, never `-j`) and retries.

**Worlds are profiles.** Agents get their own world by default (`--world agent`, `--world work`), not the person's everyday browser. They can't close someone's tabs or trample their session, and one `rm` resets them. Driving the person's own world stays possible, behind the existing Settings switch.

Today "world" means "test": `Store.testing` gates `tap`/`key`/`select`/`resize` and `--yes`. We'd split that into three kinds of world:
- **user**: the default world. Page commands only.
- **agent**: named, made with `search world new`. It gets real input and auto-answered dialogs.
- **test**: for developing Search itself.

## 2. The CLI

One binary, `search`, shipped inside the app bundle (`Contents/MacOS/search`) and linked by the brew cask's `binary` stanza. It speaks the existing bench protocol. `./bench` stays as the dev script, or becomes a wrapper around `search`.

Global flags: `--world NAME` (like Aside's `--account`), `--json` (default output is short plain text), `--tab ID` or `SEARCH_TAB`. A bare `search open` prints a tab id that later commands take.

```bash
# lifecycle
search world new work                    # an agent world, bench on
search login work https://github.com     # a real Search window; sign in, close it, done
search status                            # worlds, which are running, headless or not, tab counts
search stop work

# pages
t=$(search --world work open https://github.com/notifications)
search wait $t --idle 10                 # load, then network quiet (or: --selector 'css', --text 'Inbox')
search snapshot $t                       # the element tree with refs (below)
search click $t e14                      # by ref...
search click $t 'button[type=submit]'    # ...or CSS selector, or text='Sign in'
search type $t e7 'hello' --enter
search press $t Escape
search select $t e9 'Weekly'
search scroll $t down                    # or: to e40
search text $t                           # innerText, as today
search eval $t 'document.title'
search shot $t /tmp/n.png [--full]
search go $t https://… ; search back $t ; search reload $t
search tabs ; search close $t

# agents
search mcp [--world work]                # stdio MCP server, same verbs
search skill install                     # the SKILL.md into Claude Code / Codex, as `aside skills install` does
```

**The snapshot** is the one thing agents need that bench lacks. It is a flat, indented list of what can be seen and acted on: role, accessible name, value, state, and a ref.

```
Hacker News
https://news.ycombinator.com/
[e2] link "Hacker News"
[e3] link "new"
[e12] link "Linux support is coming to Snapdragon X2 Series"
[e30] textbox "q" = ""
[e31] button "Search" disabled
```

A ref is stamped onto the element (`data-search-ref`), so `click e3` resolves straight back to the element. It stays valid until the next snapshot or navigation, and the CLI says `stale ref` when it isn't. The first version is plain JS (`role`/`aria-label`/`labels`/`alt`/`placeholder`/innerText, visible only). The prototype that produced the example above is ~20 lines. Later it should walk same-origin iframes and open shadow roots, and cross-origin frames through `callAsyncJavaScript(in: WKFrameInfo)`. `--full` adds headings and text landmarks for reading.

**Input**: in an agent world, `click`/`type`/`press` go through the real-event path (`tap`/`key` today). Pages then see `isTrusted: true`, which some sites require. The JS path stays as a fallback for user worlds.

**First-class commands vs `repl`**: everything above is first-class. It is small, agents discover it through `--help`, and it maps 1:1 onto MCP tools. **No Playwright-style `repl` in v1.** Aside's repl is a JS runtime on the driver's side with `openTab`/`page` objects, which is a second API to design and keep compatible. Agents already have a shell, and `eval` covers "run this in the page". Revisit only if agents keep writing loops of 20 CLI calls.

**Copy from Aside**: one binary with subcommands, `mcp` over stdio, installing the skill into coding agents, an account flag (our `--world`), JSON output.

**Don't copy**: `exec` and its model/effort/provider flags (Search is the substrate, not the agent), `memory`, `session steer/queue` (they belong to an agent's conversation, not a browser), permission modes (a world is the boundary), `--host` (local only).

## 3. Auth

The person sees one window, once per world:

1. `search login work https://mail.google.com`. The world's process (headless or not) switches to `.regular` and opens a normal Search window on that URL, in that world. It has an address field, passkeys, and passwords from the keychain.
2. The person signs in and closes the window. The process drops back to `.accessory`. Cookies are in the world's persistent store on disk, so they survive restarts.
3. Everything after that is headless in the same process and store. Nothing crosses a process boundary, so there is no store-sharing question to get wrong.
4. When a session expires, the agent sees a sign-in page in the snapshot. The CLI exits with a distinct code (`needs login`, found by a login-form heuristic). The agent asks the person to run `search login` again. Agents never get passwords: no autofill in agent worlds.

**Google, honestly**: `disallowed_useragent` targets WKWebView's default user agent, the one without `Version/… Safari/…`. Search sends `Version/26.5 Safari/605.1.15` (`Tab.swift:17`), so Google sees Safari. People already sign in to Google in Search today: CHANGELOG #2 fixed a reload loop *during* Google sign-in. The login window is a full browser window, which is what Google's policy asks for. What I did **not** verify: a fresh Google sign-in end to end (no account on this box). That is the first manual check. The residual risk is Google adding embedded-browser detection beyond the user agent. That is a policy risk, not a technical wall, and every WebKit browser that isn't Safari shares it.

**Importing Chrome cookies: no, not in v1.**
- Reading them means the "Chrome Safe Storage" keychain prompt and decrypting Chrome's cookie DB. That is fragile, and it is the kind of thing users rightly distrust.
- For Google specifically, Device Bound Session Credentials make it pointless. Chrome keeps the session alive with short-lived cookies it refreshes by signing a challenge with a key held in the Secure Enclave/TPM. Lifted cookies die within minutes, and nothing outside that Chrome can refresh them.
- It would work for sites that don't bind sessions, but login-once already covers them without the trust cost. Search already imports *passwords* from Chrome, and that is the right level.

## 4. MCP

Yes. `search mcp` runs over stdio, and its tools are generated from the same command table as the CLI, so the two can't drift: `open`, `snapshot`, `click`, `type`, `press`, `select`, `scroll`, `wait`, `text`, `eval`, `shot` (returns image content), `tabs`, `close`, `go`. There is no `login` tool. It returns an instruction for the person instead, because a login needs a human. It talks to the same socket and auto-launches headless the same way.

## 5. What's hard, ranked. And what's cut

1. **One process that is both GUI and headless.** It has to be born with no window, grow one for login or when the person opens the app, and shrink back. It touches the SwiftUI scene lifecycle in `App.swift`/`Links.swift`, which already fights hidden launches. Double launches and window restoration will produce bugs. This is where the time goes.
2. **Dialogs and prompts in headless.** alert/confirm/prompt, file pickers, downloads, camera/location/notification permissions, and extension asks all wait for a person who isn't there. Agent worlds need a policy (auto-dismiss or answer, and report it in the command's output). `ext-answer` is the seed.
3. **Snapshot quality.** Frames, shadow DOM, virtualized lists, canvas apps. The 20-line version handles ordinary pages. Gmail- and Notion-class apps will need iteration.
4. **Private SPI.** `_setWindowOcclusionDetectionEnabled:` could change or disappear. It is already used, and Search ships with Developer ID, not the App Store. The fallback (a window on screen at alpha 0, or 1 px) is untested. `hasFocus()` stays false everywhere. Sites that check focus are an open question.
5. **Google policy drift.** Low odds, high impact, and out of our hands.

**Cut**: an agent or model inside Search, `repl`, memory, steer/queue, remote hosts, cookie import, password autofill for agents, and driving the person's everyday world by default.

## 6. Smallest first slice

Keep `./bench` as the client. Change the app only:

1. The occlusion call in `Bench.house(_:)`. Verify with `./bench eval $id 'document.visibilityState'` → `"visible"`.
2. `SEARCH_HEADLESS=1` on a named world: `.accessory`, no browser window, bench on, room only.
3. `snapshot`, plus `click`/`type` accepting a ref.
4. `login URL`: show a window in this world. Closing it goes back to headless.

The proof to show is `SEARCH_PROBE=agent`: log in to GitHub once, then with zero windows on screen run open `github.com/notifications` → `snapshot` → `click` a notification. The page reports `visible`, and the person was never interrupted.

Slice 2 is the installed `search` binary, `search mcp` and the skill install. Slice 3 is GUI↔headless in one process for the person's own world.
