---
name: herdrm-contrib
description: "Contribute to herdrm (native macOS console for herdr) from this fork. Use when working in ~/Sites/herdrm: building, running, testing HerdrKit, or opening a PR upstream to missuo/herdrm. Covers the local code-signing override this machine needs, the herdr socket protocol gotchas, and the fork/PR flow."
---

# Contributing to herdrm

Swift/SwiftUI macOS app (`Sources/HerdrM`) over a socket-RPC library
(`Packages/HerdrKit`). Read `CLAUDE.md` in the repo root first — it is the
upstream author's own brief and carries the architecture and the verified herdr
protocol notes. This skill only adds what CLAUDE.md does not: how to build on
*this* machine, and the fork/PR flow.

Upstream has no CONTRIBUTING.md, no PR CI, and no linter. The contract is:
it builds, `swift test` is green, and the commit message follows the
convention below.

## Build & run

`make build` and `make run` both **fail on this machine** (`run` depends on
`build`) — `project.yml` pins
`CODE_SIGN_IDENTITY` to the maintainer's Developer ID (MOE AI LLC). Override it
with the local Apple Development cert instead of editing `project.yml`
(editing it would put signing noise in the PR):

```sh
cd ~/Sites/herdrm
xcodegen generate
xcodebuild -project HerdrM.xcodeproj -scheme HerdrM -configuration Debug \
  -derivedDataPath build build -skipPackagePluginValidation \
  CODE_SIGN_IDENTITY="Apple Development" DEVELOPMENT_TEAM=D2HZ8U62PA
open build/Build/Products/Debug/herdrm.app
```

Team `D2HZ8U62PA` = el `OU` del cert local "Apple Development: Manuel Ramirez",
válido hasta abril de 2027.

**El team ID es el campo `OU` del certificado, NO el paréntesis del nombre.** El
paréntesis es el ID del cert y confunde: `Apple Development: Manuel Ramirez (LYF5NV372Q)`
tiene `OU=D2HZ8U62PA`, y pasar `LYF5NV372Q` como `DEVELOPMENT_TEAM` falla.

**Si el build falla con `No signing certificate "Mac Development" found ... with a private
key`, sospecha caducidad antes que cualquier otra cosa.** El team anterior
(`JXGN27PTN9`, cert "manuel@unit1gear.com") **caducó el 2026-08-21 a las 22:40:05 UTC**, en
mitad de una sesión: el mismo comando compilaba verde y seis minutos después no. No es
keychain bloqueado ni Xcode. Diagnóstico en un comando — lista los certs con su `OU` y su
fecha real de caducidad, que `security find-identity` NO muestra:

```sh
security find-certificate -a -c "Apple Development" -p ~/Library/Keychains/login.keychain-db \
  > /tmp/certs.pem
# y por cada bloque:  openssl x509 -noout -subject -enddate
```

`security find-identity -v -p codesigning` miente por omisión aquí: lista una identidad como
válida sin decir de qué team es ni cuándo expira.
Re-derive it if it ever changes:
`security find-certificate -c "Apple Development: manuel" -p | openssl x509 -noout -subject`
(the `OU=` field is the team).

`-skipPackagePluginValidation` is mandatory — SwiftTerm ships a build plugin.

Verified working 2026-08-19: build succeeds (~18s incremental) and the app
launches and stays up.

Same bundle ID (`dev.bybee.herdrm`) as the brew-installed herdrm, and they share
`~/Library/Application Support/HerdrM/devices.json`. Quit the installed one
(`osascript -e 'quit app "herdrm"'`) before launching the dev build, or they
fight over that file. Quitting the console does not touch the agents — those
live in the herdr server.

### Prereqs already installed on this machine
- `xcodegen` (brew) — `HerdrM.xcodeproj` is generated, never committed
- Xcode Metal Toolchain — SwiftTerm compiles `Shaders.metal`. If a fresh Xcode
  build dies with `cannot execute tool 'metal'`, run
  `xcodebuild -downloadComponent MetalToolchain` (~700 MB).

### Ad-hoc signing is a trap
`CODE_SIGN_IDENTITY="-"` builds fine but UserNotifications silently break — the
Debug config comment in `project.yml` says as much. Any work touching
`NotificationManager.swift` must be verified with a real identity.

## Tests

```sh
cd ~/Sites/herdrm/Packages/HerdrKit && swift test          # 36 tests
```

Needs a **running local herdr server** — the LocalSocket/LocalServer suites talk
to `~/.config/herdr/herdr.sock` for real. Start one with `herdr` in another pane
if they fail to connect.

The 4 `RemoteSSHTests` skip unless a real SSH target is given. Use the VPS:

```sh
HERDRM_E2E_SSH_TARGET=kupavo@srv1759591 swift test --filter RemoteSSHTests
```

Verified green 2026-08-19 (3 pass, 1 still skips). `srv1759591` is a configured
`~/.ssh/config` host on the tailnet running herdr 0.8.0 — it is the reference
remote for anything touching `SSHTunnel`, `Device`, or `SSHCredentialStore`.

## Where things live

| Change | File |
|---|---|
| Sidebar rows, Spaces/Agents list | `Sources/HerdrM/SidebarView.swift` |
| Sheets (New Agent, New Space, Add Device) | `Sources/HerdrM/ContentView.swift` |
| App state, RPC orchestration | `Sources/HerdrM/AppModel.swift` |
| Terminal embed (SwiftTerm) | `Sources/HerdrM/TerminalView.swift` |
| Colors, spacing, hairlines | `Sources/HerdrM/Theme.swift` |
| Socket RPC / protocol | `Packages/HerdrKit/Sources/HerdrKit/SocketRPC.swift` |
| SSH tunnel, devices | `.../SSHTunnel.swift`, `.../Device.swift` |
| Wire models | `.../Models.swift` |

New Swift files under `Sources/HerdrM` need no project edit — XcodeGen globs the
directory; just re-run `xcodegen generate`.

## Protocol work

Do not guess the herdr socket API. It is a live server: query it.

```sh
herdr api --help                # socket API metadata
herdr --skill                   # herdr's own agent skill, current CLI surface
```

`CLAUDE.md` records what was verified against protocol 19 (params must always be
present even if `{}`; pane-scoped events can't be subscribed globally;
`tab.create` returns `result.root_pane.pane_id`). If a change contradicts those
notes, re-verify against the socket and update `CLAUDE.md` in the same PR.

## PR flow

`origin` = alejodelosrios/herdrm (the fork), `upstream` = missuo/herdrm.

`.claude/` (this skill) **is committed on the fork's `main`**, so PR branches
must be cut from `upstream/main` — never from local `main` — or the skill ends up
in the diff:

```sh
git -C ~/Sites/herdrm fetch upstream
git switch -c fix/short-slug upstream/main    # <- not from local main
# work, build, swift test
git push -u origin fix/short-slug
gh pr create --repo missuo/herdrm --base main --fill
```

Check before pushing: `git diff --stat upstream/main` must show no `.claude/`.

- **Commits: Conventional Commits, lowercase, optional scope.** Match the log
  exactly: `feat: …`, `fix(ui): …`, `docs(readme): …`, `ci: …`.
- **`CHANGELOG.md`: add a bullet under `## [Unreleased]`** in the right
  Added/Fixed/Changed group. It is not optional theatre — release CI extracts the
  version section for GitHub release notes *and* the Sparkle update description,
  and fails if it is missing. Do not invent a version heading; only the
  maintainer tags releases.
- Never commit `HerdrM.xcodeproj/`, `build/`, or `design/` — gitignored.
- The project is explicitly "early stage software without full test coverage —
  PRs are very welcome". Small, single-purpose PRs land best.

## Repo is a fork of an active upstream
Rebase on `upstream/main` before starting anything. Releases move fast (0.3.0
shipped the same week) and the maintainer squash-merges.
