# PS4 13.52 host - FTP from the base URL

A copy of the [raw13g](https://github.com/raw13g/raw13g.github.io) PS4
13.02 / 13.04 / 13.50 / 13.52 WebKit + kexec host.

## Use it

Open on the PS4 browser (clear the browser cache first):

    https://seahurstgames.github.io/

That is the FTP-only build: PS4-HEN is disabled, and hippie68's FTP server is
started instead. Then from a PC:

    curl -s --user anonymous: ftp://<PS4-IP>:1337/

Default port is 1337.

## What is loaded

From `jb.js`:

1. kernel patch, via kexec of `patches/1352.bin` in kernel mode;
2. a payload fetched over HTTP, copied into anonymous RWX memory in the browser
   process and started with libkernel `pthread_create`;
3. the only gate on that blob is `payloadBlob[0] === 0xe9`, a flat
   position-independent `jmp rel32`.

An ELF such as `ftpsrv` can never work here - there is no ELF loader. That is
what the upstream *"supports no other modules"* message refers to.

The old payload began `e9 91 49 00`; hippie68's server begins `e9 6b 53 00` -
the same shape.

## Why it is started where it is

`jb.js` warns about the window itself:

> `w1.td_ucred must equal the real ucred before this thread is torn down at
> process exit (crfree runs on it)`

Before that repair, `td_ucred` points at a credential the process does not hold.
A thread started there that faults makes `crfree` run on freed memory: refcount
corruption, kernel panic, dead power button.

This build starts the FTP thread **after** the repair (`JB-TDUCRED` at 3134,
`PAYLOAD2-RUN` at 3200) with PS4-HEN disabled (`if (false)` at 3054). A fault
costs the browser process, not the console.

One honest trade-off: the repair resets `td_ucred`, so the FTP server may not
inherit the credentials needed to reach `/system_data/priv/`. If FTP comes up but
cannot see that path, credential timing is the next problem, and it is solvable.

## Other builds

| path | loads |
|------|-------|
| `/` | FTP only, started after the repair |
| `/chain/` | PS4-HEN **and** FTP, both safe-window |
| `/baseline/` | untouched upstream |

## Recovering a hung console

Hold the power button ~15 s; if that fails, cut power. Nothing here writes to
flash - the exploit only patches RAM - so a power cycle recovers it. The next
boot may run a filesystem check; let it finish. Reboot before any second attempt,
or a tainted heap makes every later run fail the same way.

## Credits

- WebKit + kexec host, PS4-HEN, kernel patches: **raw13g**
- FTP payload: **hippie68/ps4-ftp** v1.08b
- ftpsrv (not usable here; it is an ELF): **drakmor**, **ps5-payload-dev**
