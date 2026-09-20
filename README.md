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
`ps4_offsets.js`, `rpc_worker.js` and `patches/*.bin` are byte-for-byte
identical to upstream (verified by SHA-256 on every build).

The one deliberate change on top of upstream is the payload: `payload2.bin` is
now **GoldHEN v2.4b18.11** (`goldhen.bin`, 13.52) instead of PS4-HEN. It is a
flat position-independent blob beginning `e9 36 0e 00`, so it satisfies the
loader's only gate (`payloadBlob[0] === 0xe9`) exactly like the old payload did;
no JavaScript was changed.

## What changed

| | upstream | here |
|---|---|---|
| branding | "RAW GAME" bar + logo | removed, no replacement |
| loading screen | spinning ring | `PLEASE WAIT, ENABLING GOLDHEN` + cycling `. .. ...` |
| result screen | blank on success, "Restart your console" on failure | green `GOLDHEN LOADED SUCCESSFULLY` / red `GOLDHEN FAILED - RESTART YOUR CONSOLE` |
| page title | `RAW GAME` | `PS4 GOLDHEN` |
| `logo_raw.png` | shipped | removed (was only referenced by the manifest) |
| payload | PS4-HEN | GoldHEN v2.4b18.11 (`payload2.bin`) |

The status screen is **pure CSS — no JavaScript was added**. `jb.js` already
publishes its outcome through `document.body.className` (`done` / `fail` /
`log`), so the stylesheet just reacts to that and the exploit code path is
untouched.

The dots animate `transform: scale()` rather than `opacity`, and that is
deliberate. The exploit owns the main thread for most of the run, and only a
composited transform keeps moving under that load — which is why the original
spinner (a `transform: rotate()`) kept spinning while opacity keyframes freeze
mid-cycle and read as a static `...`. Each dot keeps its own slot so the line
never reflows, and with no animation support at all they simply stay visible.

`done` means the payload thread started; `fail` means it did not, so the red
screen is a real signal rather than decoration.

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
payload2.bin        GoldHEN v2.4b18.11 payload (13.52)
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
