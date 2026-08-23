# SuperGrok fork notes

Personal rebaseable fork of official Zed **stable**. Adds SuperGrok OAuth next to the xAI API-key provider.

Install: `/Applications/Zed Grok.app`

Shares official settings and DB:

- `~/.config/zed/`
- `~/Library/Application Support/Zed/` (including `db/stable`)

Do not run official `Zed.app` and `Zed Grok.app` at the same time. Both are `RELEASE_CHANNEL=stable`, so they share one SQLite file and the macOS single-instance port.

## Remotes and branch

```
origin    https://github.com/thibaudgg/zed.git
upstream  https://github.com/zed-industries/zed.git
```

Branch: `feat/zed-grok-stable` from tag `v1.16.1` (current official stable, 2026-08-19).

Wednesday rebase:

```sh
git fetch --tags --force upstream
# newest non-pre v1 tag, today that is v1.16.1
git rebase --onto v1.16.1 "$(git merge-base HEAD upstream/main)"   # only if you still have main-based commits

# usual case after this branch exists:
NEW=$(git tag --sort=-v:refname | rg '^v1\.[0-9]+\.[0-9]+$' | head -1)
git rebase "$NEW"
cargo test -p x_ai_subscribed --offline
script/bundle-mac
# then copy the bundled app to /Applications/Zed Grok.app
```

If the tag moved a week, rebase onto that tag, not onto `upstream/main`. `main` is the next preview line and will fight this fork.

## Bundle

`crates/zed/Cargo.toml` `[package.metadata.bundle-stable]`:

- name: `Zed Grok`
- identifier: `dev.thibaudgg.ZedGrok` (not `dev.zed.Zed`, so Finder and keychain do not steal official Zed.app)
- no `zed://` URL scheme (official Zed keeps that)
- `APP_NAME` stays `"Zed"` so paths stay official

Auto-update from zed.dev is off:

- `assets/settings/default.json` → `"auto_update": false`
- `.cargo/config.toml` sets `ZED_UPDATE_EXPLANATION` so the poller never starts

Release rebuild from another terminal (quit Zed Grok first):

```sh
.agents/scripts/build-zed-grok
```

`--force-quit` asks Zed Grok to quit, then overwrites `/Applications/Zed Grok.app`.
`--skip-install` only writes `target/.../Zed Grok.app`.
`--debug` is an unoptimized bundle. Official `Zed.app` is never touched. The script also copies `cli` into the bundle for the fish `zed` alias.

## Run from source

```sh
# still a stable-channel binary, same DB as official Zed
cargo run --release
```

Needs full Xcode + Metal toolchain:

```sh
sudo xcode-select --switch /Applications/Xcode.app/Contents/Developer
xcodebuild -downloadComponent MetalToolchain
```

## Sign in

Agent Settings → LLM Providers → SuperGrok → Sign In.

Public xAI PKCE client `b1a00492-073a-47ea-816f-4c329264a828`, loopback `http://127.0.0.1:56121/callback`. Tokens in keychain under `https://auth.x.ai/zed-supergrok`.

If login works and inference is 403: SuperGrok Heavy is more reliable. Do not invent a fallback API.

Default model: Grok 4.6. Default thinking effort: medium.

## Architecture

| Piece | Where |
|---|---|
| Auth + completions | `crates/x_ai_subscribed/src/x_ai_subscribed.rs` |
| Sign in/out UI | `crates/language_models/src/provider/x_ai_subscribed.rs` |
| Provider id | `x_ai_subscribed` |
| Display name | SuperGrok |
| Settings | `language_models.x_ai_subscribed` |

Leave `x_ai` BYOK alone.
