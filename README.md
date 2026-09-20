# PS4 13.52 host — three builds

A copy of the [raw13g](https://github.com/raw13g/raw13g.github.io) PS4
13.02 / 13.04 / 13.50 / 13.52 WebKit + kexec host, plus two ways of getting an
FTP server onto the console.

| path | what it loads | JS changed? |
|------|---------------|-------------|
| `/` | PS4-HEN | no — upstream, untouched |
| `/ftp/` | the FTP server *instead of* PS4-HEN | **no** |
| `/chain/` | PS4-HEN **and** the FTP server | yes, one added stage |

## How the payload mechanism works

From `jb.js`:

1. **kernel patch** — `sysent[661].sy_call` is pointed at the `ff 26` (`jmp rsi`)
   gadget, then `syscall(661, mapAddr)` runs `patches/<fw>.bin` in kernel mode.
2. **payload** — fetched over HTTP, copied into anonymous RWX memory in the
   browser process, started with libkernel `pthread_create`.
3. **gate** — the only check on the blob is `payloadBlob[0] === 0xe9`, a flat
   position-independent `jmp rel32`.

That is why an ELF such as `ftpsrv` can never work here — there is no ELF loader.
It is what the upstream "supports no other modules" message refers to.

The old payload began `e9 91 49 00`. hippie68's FTP server begins
`e9 6b 53 00` — the same shape. hippie68's README also says the payload is safe
to stop by "close the browser", i.e. it is meant to run in the browser process.

## `/ftp/` — smallest possible change

No JavaScript is modified at all. `ps4_offsets.js` already reads
`payload: "payload2.bin"`, so this build simply puts the FTP server at that
filename. Everything the exploit executes is byte-for-byte the build that
already works on the console.

Cost: you get FTP instead of PS4-HEN.

## `/chain/` — both, and why the first attempt froze the console

The first attempt started the FTP thread *before* the exploit repaired
`td_ucred`, and did an `await fetch()` in the middle of the teardown. `jb.js`
warns about precisely this:

> `w1.td_ucred must equal the real ucred before this thread is torn down at
> process exit (crfree runs on it)`

A thread that can crash the process inside that window produces a **kernel
panic** — which is what a console with a dead power button looks like.

This build fixes both:

- the blob is fetched **up front**, alongside the HEN payload — no `await` during
  teardown;
- the thread is started **after** the `JB-TDUCRED` repair and the end-of-run
  restore hook. Verified ordering in the shipped `jb.js`:

  ```
  2591  await fetch(PAYLOAD_FILE)      HEN
  2599  await fetch(PAYLOAD2_FILE)     FTP, fetched early
  3120  "PAYLOAD-RUN"                  HEN thread starts
  3139  "JB-TDUCRED"                   ucred repaired
  3205  "PAYLOAD2-RUN"                 FTP thread starts - after the repair
  3215  "EG-VERDICT"
  ```

`?payload2=0` disables the second payload, so this one build can also reproduce
upstream behaviour exactly. `?payload2=<file>` overrides the blob.

### The trade-off this creates

Line 3139 sets `td_ucred` back to the real ucred. A thread started *after* that
point may therefore not inherit the elevated credentials an FTP server needs to
reach `/system_data/priv/`. Starting it earlier inherits them, but re-opens the
panic window.

That trade-off is exactly why `/ftp/` is the first thing to try: there the FTP
blob is created by the exploit's own payload slot, in the very place PS4-HEN
would have been, so it gets the same credentials with no new failure mode at all.

A note on an earlier revision of `chain/jb.js`: it fetched the second blob at
line 3114 and started the thread at 3158, both *before* the repair at 3183. That
ordering reproduces the original panic hazard and should not be used.

## Suggested order

1. `/ftp/jb.html` — proves whether the FTP blob can run in that process at all.
   Then `curl -s --user anonymous: ftp://<PS4-IP>:1337/`
2. `/chain/jb.html` — only if you want PS4-HEN back as well.

If `/ftp/` hangs too, the blob cannot run there, and the answer is a payload
that is definitely relocatable rather than more attempt-and-reboot.

## Recovering a hung console

Hold the power button ~15 s; if that fails, cut power. Nothing here writes to
flash — the exploit only patches RAM — so a power cycle recovers the console.
The next boot may run a filesystem check; let it finish.

## Credits

- WebKit + kexec host, PS4-HEN, kernel patches: **raw13g**
- FTP payload: **hippie68/ps4-ftp** v1.08b (default port 1337)
- ftpsrv (not usable here; it is an ELF): **drakmor**, **ps5-payload-dev**
