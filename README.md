# PS4 13.52 host — three builds

A copy of the [raw13g](https://github.com/raw13g/raw13g.github.io) PS4
13.02 / 13.04 / 13.50 / 13.52 WebKit + kexec host, plus two attempts at getting
an FTP server onto the console.

| path | what it is | status |
|------|------------|--------|
| `/` | upstream, untouched | known good on this console |
| `/ftp/` | upstream code, payload slot repointed at an FTP server | **try this** |
| `/chain/` | HEN **and** FTP as two threads | **froze the console — do not use** |

## `/ftp/` — why this one is different

It changes **no JavaScript**. `ps4_offsets.js` already reads
`payload: "payload2.bin"`, so `/ftp/` just puts the FTP server at that filename.
Everything the exploit actually executes is byte-for-byte the build that already
works on the console; the only difference is which bytes get mapped and started.

What the file is, and why it fits:

- `jb.js` maps the payload into anonymous RWX memory in the browser process and
  starts it with libkernel `pthread_create`. Its only check is
  `payloadBlob[0] === 0xe9`.
- The old PS4-HEN blob began `e9 91 49 00`; hippie68's FTP server begins
  `e9 6b 53 00` — the same flat position-independent `jmp rel32` shape.
- hippie68's own README says the payload is safe to stop with "close the
  browser", i.e. it is explicitly meant to be run inside the browser process.

An ELF such as ftpsrv can never work here: there is no ELF loader, which is what
the upstream "supports no other modules" message refers to.

## What `/chain/` did

It started FTP as a *second* thread after PS4-HEN and added an `await fetch()`
in the middle of the exploit's teardown — after the kernel patch and payload
start, but before `td_ucred` is restored. That build hung the console. Do not
use it. If you want a combined build later, it has to be built without
perturbing the exploit's flow, and that is a separate piece of work.

## Testing `/ftp/`

Open on the PS4 browser, with the browser cache cleared first:

```
https://seahurstgames.github.io/ftp/jb.html
```

Then from the PC:

```
curl -s --user anonymous: ftp://<PS4-IP>:1337/
```

hippie68's server defaults to **port 1337**.

If the console hangs instead, that tells us something useful: the FTP blob
cannot run in that process at that address, and the next step is a payload that
is definitely position-independent rather than more attempt-and-reboot.

## Recovering a hung console

Hold the power button ~15 s; if that fails, cut power. Nothing here writes to
flash — the exploit only patches RAM — so a power cycle recovers the console.
Note that when it hangs, the PS4 may spend a moment checking the filesystem.

## Credits

- WebKit + kexec host, PS4-HEN, kernel patches: **raw13g**
- FTP payload: **hippie68/ps4-ftp** v1.08b
- ftpsrv (not usable here; it is an ELF): **drakmor**, **ps5-payload-dev**
