# Web Lab #2 – Cookie flags: `HttpOnly` and `Secure`, tested with `curl -v`

**Date:** 18 Nov 2026
**Setup:** Raspberry Pi 5 (16 GB), Linux, Python 3.13.5, curl 8.14.1. Server and client on the same machine (`127.0.0.1:8000`)
**Tools:** `python3`, `http.server` (`BaseHTTPRequestHandler`), `curl` (`-v`, `-c`, `-b`, `--resolve`, `--noproxy`)

## Goal

[Web Lab #1](2026-11-07-web-lab-1-http-curl-cookies-wireshark.md) showed that `Set-Cookie` and `Cookie` are plain headers. In this lab I added the `HttpOnly` and `Secure` flags to a cookie and checked with `curl -v` what changes: what the server sends, what the client stores, and whether the cookie is sent back over plain HTTP. The main point I wanted to understand is that writing `Secure` in a header does **not** mean the connection is encrypted.

## What I did and what I saw

### 1. A small HTTP server that sets a flagged cookie

```bash
mkdir -p ~/labs/web-lab-02 && cd ~/labs/web-lab-02
python3 --version
nano server.py
```

```python
from http.server import BaseHTTPRequestHandler, HTTPServer

class MyHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.send_header("Content-Type", "text/plain")
        self.send_header("Set-Cookie", "session_id=lab123; Path=/; HttpOnly; Secure")
        self.end_headers()
        self.wfile.write(b"Web Lab #2 - Hello from Raspberry Pi 5!")

server = HTTPServer(("127.0.0.1", 8000), MyHandler)
print("Server running at http://127.0.0.1:8000")
server.serve_forever()
```

The server only listens on `127.0.0.1`, so it is not reachable from the network. I started it with `python3 server.py` and sent a request from a second terminal:

```bash
curl -v http://127.0.0.1:8000/
```

The response contained `HTTP/1.0 200 OK` and `Set-Cookie: session_id=lab123; Path=/; HttpOnly; Secure`.

### 2. Reading the headers

| Line in the output | Meaning |
|--------------------|---------|
| `> GET / HTTP/1.1` | Request sent by the client |
| `< HTTP/1.0 200 OK` | Status line of the response |
| `< Set-Cookie: ...` | The server tells the client to store a cookie |
| `> Cookie: ...` | The client sends a stored cookie back |

`send_header()` creates a real HTTP response header, while `print()` only writes to the server's terminal. A header being visible does not prove that the browser or client will treat the cookie securely.

### 3. What each part of the cookie means

| Part | Meaning |
|------|---------|
| `session_id=lab123` | Name and value (a test value, not a real session) |
| `Path=/` | The cookie applies to all paths of the site |
| `HttpOnly` | Browser JavaScript should not be able to read the cookie through `document.cookie` |
| `Secure` | The client should normally send the cookie only over a secure connection (HTTPS) |

`HttpOnly` does not remove an XSS vulnerability, it only makes it harder to steal the cookie with injected JavaScript. `curl` ignores `HttpOnly` because it is not a browser and has no JavaScript.

### 4. Cookie jar: save and send back

```bash
curl -v -c ~/labs/web-lab-02/cookies.txt http://127.0.0.1:8000/
cat ~/labs/web-lab-02/cookies.txt
curl -v -b ~/labs/web-lab-02/cookies.txt http://127.0.0.1:8000/
```

The jar contained the line `#HttpOnly_127.0.0.1  FALSE  /  TRUE  0  session_id  lab123`. The `#HttpOnly_` prefix marks the flag, `TRUE` is the Secure column and `0` means a session cookie without an expiry date. On the second request curl sent `> Cookie: session_id=lab123`, so the cookie was stored and returned.

This was surprising at first: the cookie is marked `Secure`, but it was stored and sent over plain `http://127.0.0.1`. That is a special case for loopback addresses, not a general rule.

### 5. Secure cookie on a normal HTTP hostname

To avoid the loopback exception without leaving my machine, I mapped a made-up hostname to `127.0.0.1` for a single request:

```bash
rm -f ~/labs/web-lab-02/secure-test.txt
curl -v --noproxy '*' \
  --resolve cookie.lab.test:8000:127.0.0.1 \
  -c ~/labs/web-lab-02/secure-test.txt \
  http://cookie.lab.test:8000/
cat ~/labs/web-lab-02/secure-test.txt
```

The server still answered with `Set-Cookie: ...; HttpOnly; Secure`, but the clean `secure-test.txt` contained **no** `session_id` entry. For a normal HTTP hostname curl refused to store the Secure cookie. `--resolve` maps the name to an IP address only for this command, and `--noproxy '*'` makes sure no proxy is used.

### 6. Mistakes I made and fixed

- `self.self_response(200)` instead of `self.send_response(200)` made the handler fail and curl printed `curl: (52) Empty reply from server`.
- I wrote `session_id=lab123: Path=/`. Cookie attributes are separated with `;`, not `:`.
- The cookie file path `~/labs/web-lab-02.cookies.txt` was wrong and gave `failed to open cookie file`. The correct path was `~/labs/web-lab-02/cookies.txt`.
- `cat ~` tried to read a directory. I need to give the full path of a file.
- Looking at the old `cookies.txt` mixed up my results, so I repeated the test with a new, differently named file (`secure-test.txt`).

### 7. Cleanup

- Stopped the server with `Ctrl + C`.
- The test cookies (`cookies.txt`, `secure-test.txt`) only contain a fake value and stay in the lab folder.

## What I observed

- `Set-Cookie` (server → client) and `Cookie` (client → server) are different directions of the same mechanism.
- A flag in a header is only an instruction. Whether it is respected depends on the client.
- `Secure` was ignored for loopback but enforced for a normal HTTP hostname.
- A clean test file matters: an old cookie jar can hide what you are really testing.

## Security angle

- **`HttpOnly` limits damage from XSS:** injected JavaScript cannot read the session cookie directly, but it can still make requests as the user.
- **`Secure` is a transport policy, not encryption:** it stops the client from sending the cookie over plain HTTP, but it does not encrypt anything. Real confidentiality needs HTTPS/TLS.
- **Session cookies need all the flags:** a session cookie sent over HTTP can be read by anyone who captures the traffic, as seen in Web Lab #1.
- **Test results depend on the environment:** loopback behaves differently from a real hostname, so a "working" result on `localhost` can be misleading.

## What I learned

- How to add `HttpOnly` and `Secure` to a `Set-Cookie` header in a Python `http.server` handler.
- How to read `curl -v` output for `Set-Cookie` and `Cookie`, and how to inspect a cookie jar.
- The meaning of `HttpOnly` and `Secure`, and what they do **not** protect against.
- How to use `--resolve` and `--noproxy '*'` to test a hostname locally.
- How to debug a failing server from the client side (`Empty reply from server`).

## Commands used

| Command | Purpose |
|---------|---------|
| `python3 server.py` | Start the lab HTTP server on `127.0.0.1:8000` |
| `curl -v <url>` | Show the full request and response |
| `curl -c <file> <url>` | Save cookies from the response |
| `curl -b <file> <url>` | Send saved cookies |
| `curl --resolve host:port:ip <url>` | Map a hostname to an IP for one request |
| `curl --noproxy '*' <url>` | Do not use a proxy |

## Next

- Set up a local **HTTPS/TLS** server and inspect the certificate and connection with `curl`.
- Compare HTTP and HTTPS traffic in Wireshark (Network Lab #10).
- Test `HttpOnly` in a browser with JavaScript (`document.cookie`).
- Continue with HTTP methods (`GET` vs `POST`) and authentication vs authorization.
