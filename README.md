# claude-alt

Run more than one Claude Desktop account on the same Mac, each as a separate app
with its own Dock icon, name, colour, login, and local session history.

macOS only. No modification of `/Applications/Claude.app` — it stays
Developer-ID signed, notarised, and auto-updating.

## How it works

Each extra account gets an **APFS copy-on-write clone** of `/Applications/Claude.app`
(instant, ~0 extra disk) with:

| Change | Why |
|---|---|
| `CFBundleDisplayName` → `Claude <Name>` | what Dock and ⌘-Tab show |
| `CFBundleIdentifier` → `local.claude<name>` | separate app identity for LaunchServices/TCC |
| `Contents/MacOS/Claude` → compiled launcher | pins `--user-data-dir` (see trap 3) |
| `electron.icns` recoloured | tell the apps apart at a glance |
| re-signed with a stable local identity | so the login survives rebuilds (trap 4) |

## Install

```bash
curl -o ~/.local/bin/claude-alt <raw-url> && chmod +x ~/.local/bin/claude-alt
```

Requires `python3` with Pillow (icon recolouring) and Xcode command line tools (`cc`).

Create the signing identity once (no sudo, no trust settings needed):

```bash
openssl req -x509 -newkey rsa:2048 -nodes -sha256 -days 7300 \
  -keyout key.pem -out cert.pem -config ext.cnf     # CN="Claude Alt Local Signing"
                                                     # basicConstraints=critical,CA:true
                                                     # keyUsage=critical,digitalSignature
                                                     # extendedKeyUsage=critical,codeSigning
openssl pkcs12 -export -inkey key.pem -in cert.pem \
  -name "Claude Alt Local Signing" -out bundle.p12 -passout pass:transport
security import bundle.p12 -k ~/Library/Keychains/login.keychain-db \
  -P transport -T /usr/bin/codesign -f pkcs12
rm -f key.pem bundle.p12          # private key now lives only in the keychain
```

Without it `claude-alt` still works, but falls back to ad-hoc signing and every
rebuild forces a fresh login.

## Usage

```
claude-alt list                                   original + clones, versions, drift
claude-alt new <name> <hex>                       create "Claude <Name>.app"
claude-alt refresh [name]                         re-clone after a Claude update
claude-alt self-update [name] [--force]           update a clone from inside its own chat
claude-alt doctor [name]                          verify — including runtime routing
claude-alt run <name>                             launch
claude-alt remove <name>                          delete the app, keep account data
claude-alt accounts                               account UUID → email map
claude-alt import-sessions <name> --email <addr>  copy one account's sessions into a clone
```

Clones do **not** auto-update. After Claude updates itself, run `claude-alt refresh`
then `claude-alt doctor`. `list` flags `<< STALE` and `<< CLOBBERED`.

## Four traps, learned the hard way

**1. `CFBundleName` must stay `"Claude"`.**
Electron derives the helper-app path from it, so renaming it makes the app hunt for
a nonexistent `Claude <Name> Helper.app` and die with
`FATAL electron_main_delegate_mac.mm:66 Unable to find helper app`.
Put the new name in `CFBundleDisplayName` — that is what Dock and ⌘-Tab read anyway.

**2. Editing `Info.plist` breaks the signature, and ad-hoc can't carry everything.**
The CodeDirectory seals an Info.plist hash, so any edit forces a re-sign. Re-signing
drops `com.apple.security.virtualization` and the team keychain groups, so VM-backed
features are unverified in clones and TCC permissions prompt separately.
`embedded.provisionprofile` must be deleted or launch is rejected.

**3. `CLAUDE_USER_DATA_DIR` alone is not enough (Claude ≥ 1.34493.1).**
It still moves Electron's paths via `app.setPath('userData')`, but Chromium's own
directories — IndexedDB, WebStorage, SharedStorage, Partitions, GPUCache, DIPS —
stay in the **default** profile. Chats and routines live in IndexedDB, so the clone
looks like it lost all history while quietly sharing the default profile.
Only `--user-data-dir` fixes the Chromium side, and argv cannot come from `Info.plist`,
hence the launcher. The launcher **must be a compiled Mach-O**: launchd refuses to
spawn a shell script as `CFBundleExecutable` under hardened runtime
(`RBSRequestErrorDomain Code=5, Launchd job spawn failed`) — while running that same
script from a terminal works, which makes it easy to misdiagnose.

**4. Ad-hoc signing logs you out on every rebuild.**
Ad-hoc yields `designated => cdhash H"..."`, which changes each build and invalidates
the keychain ACL on `Claude Safe Storage`. A stable self-signed identity yields
`designated => identifier "local.claude<name>" and certificate root = H"..."`,
identical across rebuilds. The certificate does not need macOS trust —
`codesign` uses it despite `CSSMERR_TP_NOT_TRUSTED`.

## Verify

`claude-alt doctor` checks version, display name, signature authority, that the
launcher is a Mach-O, and — at runtime — that the clone has **zero files open in the
default profile** and its helpers are pinned to the right `--user-data-dir`.
That last pair is the important one: bundle-level checks pass happily while
routing is broken.

## Notes

Clone updaters still poll and fail harmlessly with
`SQRLUpdaterErrorDomain: Could not locate update bundle for local.claude<name>` —
Squirrel aborts at the bundle-identifier lookup, before any signature check.
The ShipIt cache self-cleans, so there is no disk leak.

Session history is **local and per-profile**; it does not follow the account on
login (only claude.ai conversations are server-side). A new clone therefore starts
with an empty session list — use `accounts` + `import-sessions` to bring one
account's history across.
