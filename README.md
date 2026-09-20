# RAW GAME — PS4 13.52 host with a chained FTP payload

A copy of the [raw13g](https://github.com/raw13g/raw13g.github.io) PS4
13.02 / 13.04 / 13.50 / 13.52 WebKit + kexec jailbreak host, with one addition:
after the bundled PS4-HEN payload starts, a second native payload (`ftp.bin`)
is started the same way, which gives you an FTP server on the console.

## Use it

Open this on the PS4 browser:

```
https://<USERNAME>.github.io/<REPO>/jb.html?payload=1
```

Watch the on-screen console for:

```
KEXEC syscall(661)=0
PAYLOAD-RUN pthread_create=0 handle=...
PAYLOAD2-BLOB ftp.bin len=37824 e9=1
PAYLOAD2-RUN ftp.bin pthread_create=0
```

then from a PC:

```
curl -s --user anonymous: ftp://<PS4-IP>:1337/
```

`?payload=0` disables the whole payload stage; `?payload2=other.bin` picks a
different second payload.

## Why a second payload works

`jb.js` runs the payload slot named in `ps4_offsets.js`
(`payload: "payload2.bin"`, PS4-HEN) by mapping the file into anonymous RWX
memory **in the browser process** and starting it with libkernel's
`pthread_create`. The only check it makes on that blob is:

```js
payloadBlob[0] === 0xe9        // a flat position-independent `jmp rel32`
```

`ftp.bin` (hippie68/ps4-ftp v1.08b) begins `e9 6b 53 00`, so it satisfies the
same check and is started as a second userland thread. The kernel patch
(`patches/1352.bin`) is a separate kexec step and is untouched.

This is also why an ELF such as ftpsrv cannot be used here: there is no ELF
loader in this host. That is what the upstream "supports no other modules"
message means.

## Enabling GitHub Pages

Settings → Pages → Source: **Deploy from a branch** → `main` / `(root)`.
Both site layouts work, because every path here is relative:

| repo | URL |
|------|-----|
| `USERNAME.github.io` | `https://USERNAME.github.io/jb.html?payload=1` |
| any other repo | `https://USERNAME.github.io/<repo>/jb.html?payload=1` |

## Troubleshooting

- `PAYLOAD2-BLOB ... e9=0` or `PAYLOAD2-FETCH-FAIL` — the browser served a
  cached copy, or `ftp.bin` 404s. Open
  `https://<USERNAME>.github.io/<REPO>/ftp.bin` in a normal browser; it should
  download 37 KB starting `e9 6b 53 00`.
- `PAYLOAD2-RUNNING` never appears — the blob may not be position-independent
  enough for an arbitrary mmap address. Test it on its own with the swap build.
- Application Cache is per-origin, so a fresh origin starts clean; bump the
  `# rev` line in `cache.appcache` whenever a cached file changes, or the
  console will keep serving the old one.

## Credits

- WebKit + kexec host, PS4-HEN payload, kernel patches: **raw13g**
- `ftp.bin`: **hippie68/ps4-ftp** v1.08b — default port 1337
- ftpsrv (not usable here; it is an ELF): **drakmor**, **ps5-payload-dev**

The upstream host ships no licence. This is a republished copy with one patch,
so keep the credit to `raw13g` if you leave it public.
