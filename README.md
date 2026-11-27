# PudimTSL

> **Status: on hold.** I'm still evaluating whether this library is useful. If I come back to it, I will most likely rewrite it from scratch.

A small Lua helper that creates a **local certificate authority** and uses it to sign **HTTPS certificates for development servers**.

> **A note on the name:** despite "TSL", this is about **TLS** (Transport Layer Security). The code's own log channels say `TLS`.

## Why it exists (and why it's paused)

It was born inside [PudimServer](https://github.com/pessoa736/PudimServerAPIs), my attempt at a web framework in Lua, and was moved out into its own repo (the commit is literally "moving TLS.lua from PudimServer to here"). I now think a better design is to leave the TLS part to a **Docker-based backend** instead of handling certificates from Lua, so the reason this library exists is in question.

## What it does

`TSL.lua` exposes three functions:

| Function | What it does |
|---|---|
| `TSL:ensureCA(dir)` | Loads `dir/ca.key` + `dir/ca.crt` if they exist and are readable; otherwise creates a new self-signed CA and saves both files. Returns the key and certificate objects. |
| `TSL:ensure(dir, DNSs, IPs)` | Calls `ensureCA`, then creates a server key and certificate signed by that CA, saves them as `dir/server.key` and `dir/server.crt`, and returns the two paths. `DNSs` and `IPs` are optional lists for the Subject Alternative Name. |
| `TSL:createSan(DNSs, IPs)` | Builds the SAN string, e.g. `DNS:localhost,DNS:pudim.local,IP:127.0.0.1`. Defaults to `localhost` and `127.0.0.1`. |

What gets generated:

| | CA | Server |
|---|---|---|
| Key | EC, curve `prime256v1` | EC, curve `prime256v1` |
| Subject | `CN=pudim.local, O=DaviPassos, C=BR` | `CN=localhost, O=PudimServer, C=BR` |
| Validity | 1 year | 1 year |
| Extensions | `CA:TRUE`, `keyCertSign, cRLSign` | `CA:FALSE`, `digitalSignature, keyEncipherment`, `serverAuth`, SAN |
| Signature | self-signed, SHA-256 | signed by the CA, SHA-256 |

## Requirements

- Lua 5.4
- [`openssl`](https://github.com/zhaozg/lua-openssl) for Lua: `luarocks install openssl`
- [`loglua`](https://github.com/pessoa736/loglua) (my logging library)
- `PS.utils`: the helpers from PudimServer ([`PudimServer/utils.lua`](https://github.com/pessoa736/PudimServerAPIs/blob/main/PudimServer/utils.lua)). The `PS.utils` module name predates the current layout of that repo, so you have to make that file available under that name.

## Usage

```lua
local TSL = require("TSL")

-- creates ./certs/ca.{key,crt} (if missing) and ./certs/server.{key,crt}
local keyPath, crtPath = TSL:ensure("./certs", { "localhost", "pudim.local" }, { "127.0.0.1" })
```

Checking the result with the OpenSSL command line:

```sh
openssl verify -CAfile certs/ca.crt certs/server.crt     # certs/server.crt: OK
openssl x509 -in certs/server.crt -noout -ext subjectAltName
```

The returned paths are PEM files of the kind PudimServer's `Https.Key` / `Https.Certificate` options expect; I did not test that integration. A browser will only trust the server certificate if you add `ca.crt` to its trust store.

## Known limitations

Verified by running it (Lua 5.4, `openssl` 0.11.1) unless noted:

- **Private keys are written world-readable.** Files are created with the default umask (mode `644` in my test). Run `chmod 600` on the keys yourself.
- **The server certificate is regenerated on every `ensure()` call.** Only the CA is reused.
- **No expiry handling** (from reading the code): an existing CA is reused without checking whether it has expired, and nothing renews anything. Both certificates last 1 year.
- **Serial numbers are `os.time()`.** Two certificates issued by the same CA in the same second would get the same serial.
- **Subject names are hard-coded** (see the table), and if you pass your own `DNSs`, `localhost` is not added automatically.
- **Mixed key formats:** `ca.key` is a SEC1 `EC PRIVATE KEY`, `server.key` is PKCS#8 `PRIVATE KEY`.
- **No tests, no rockspec, no documentation** beyond this file.
- Keys and certificates are in `.gitignore` (`*.key`, `*.crt`), so they never get committed.

## License

MIT, Copyright (c) 2026 Davi. See [`LICENSE`](LICENSE).
