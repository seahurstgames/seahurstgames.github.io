# PS4 jailbreak host — 13.02 / 13.04 / 13.50 / 13.52

Load in the PS4 browser:

```
https://seahurstgames.github.io/
```

`index.html` reads the firmware from the user agent, checks the Application
Cache, and forwards itself to `jb.html` automatically on a supported firmware.
On anything else it shows a tap prompt instead of launching, so a
user-agent quirk can never silently run the wrong build.

## What this is

The `raw13g` host with its branding removed and the loading screen replaced.
The jailbreak itself is untouched: `jb.js`, `core.js`, `mem.js`, `int64.js`,
`ps4_offsets.js`, `rpc_worker.js`, `payload2.bin` and `patches/*.bin` are
byte-for-byte identical to upstream (verified by SHA-256 on every build).

## What changed

| | upstream | here |
|---|---|---|
| branding | "RAW GAME" bar + logo | removed, no replacement |
| loading screen | spinning ring | `PLEASE WAIT, ENABLING HEN` + cycling `. .. ...` |
| page title | `RAW GAME` | `PS4 HEN` |
| `logo_raw.png` | shipped | removed (was only referenced by the manifest) |

The status line is pure CSS. `jb.js` never touches it — it only ever sets
`body.done` / `body.fail` / `body.log`, and the stylesheet reacts to those, so
none of the exploit code path was modified.

Note that the success and failure screens behave the same as upstream:
`done` clears the screen (reboot to load HEN), `fail` shows
"Restart your console". A failure means the payload thread never started.

## Debug

- `?log=1` — full step-by-step log instead of the status screen
- `?verbose=1` — don't truncate the log lines
- `?force=1` — run even if the firmware isn't in the kernel table

## Files

```
index.html          landing / firmware gate / app-cache handling
jb.html             status screen, imports jb.js
jb.js               the exploit (upstream, unchanged)
core.js mem.js int64.js ps4_offsets.js rpc_worker.js
payload2.bin        PS4-HEN payload
patches/1302.bin patches/1350.bin patches/1352.bin
cache.appcache      offline cache manifest, SHA-256 per file
.nojekyll           serve files verbatim
```

## Rebuilding

`../build_site.py` regenerates this directory from a pristine `../upstream/`
copy: it removes the branding, swaps the loading screen, and recomputes every
`cache.appcache` hash. Each edit is asserted, so drift fails the build instead
of shipping quietly.

## Rollback

Every file is byte-identical to upstream except `index.html`, `jb.html` and
`cache.appcache` (plus the two dotfiles). To go back, restore those three from
`raw13g.github.io` and drop `.nojekyll` / `.gitattributes`.

If the PS4 browser serves you a stale build, clear its browser data — the
Application Cache is per-origin.
