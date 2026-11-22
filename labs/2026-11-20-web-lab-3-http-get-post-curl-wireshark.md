# Web Lab #3 – HTTP methods: GET vs POST with `curl` and Wireshark

**Date:** 20 Nov 2026
**Setup:** Ubuntu VM, server and client on the same machine (`127.0.0.1:8000`), my Mac for analysing the capture
**Tools:** Python 3 (`BaseHTTPRequestHandler`), `curl` (`-v`, `-X`, `-d`, `-fL`), Wireshark (`lo` interface), `find`, `realpath`, `mv`

## Goal

[Web Lab #1](2026-11-07-web-lab-1-http-curl-cookies-wireshark.md) and [Web Lab #2](2026-11-18-web-lab-2-cookie-flags-httponly-secure-curl.md) used only `GET` requests. In this lab I compared **`GET` and `POST`**: where the data travels (URL or body), how it looks in `curl -v`, and what Wireshark shows on the wire.

## What I did and what I saw

### 1. A Python server that handles GET and POST

I checked that the folder `~/web-lab1` from Web Lab #1 still had `index.html` and `cookie_server.py`, then wrote a new server, `post_server.py`, based on `BaseHTTPRequestHandler`:

- `do_GET()` reads the **query parameters** from the URL.
- `do_POST()` reads the request **body**, using the `Content-Length` header to know how many bytes to read.
- `HTTPServer` listens on TCP port 8000.

Client and server both ran on the VM, so I used the loopback address `127.0.0.1`.

### 2. GET request

From a second terminal:

```bash
curl -v "http://127.0.0.1:8000/?username=testuser&action=login"
```

| What I saw | Meaning |
|------------|---------|
| `> GET /?username=testuser&action=login HTTP/1.1` | The data is part of the request line |
| `< HTTP/1.0 200 OK` | The server answered successfully |
| server output `username=testuser&action=login` | The server parsed the query string correctly |

Everything after the `?` is the **query string**. With `GET` the parameters travel inside the URL.

### 3. POST request

The same data sent as a POST:

```bash
curl -v -X POST -d "username=testuser&action=login" http://127.0.0.1:8000/
```

In the request I saw:

- `POST / HTTP/1.1`
- `Content-Type: application/x-www-form-urlencoded`
- `Content-Length: 30`

The server read `username=testuser&action=login` from the **body** and showed it in its response, again with `200 OK`. I also noticed that `-d` makes curl choose `POST` by default, so `-X POST` is optional here, but explicit.

### 4. The same traffic in Wireshark

Because the traffic never leaves the VM, I captured on the **`lo` (loopback)** interface instead of `enp0s1`, with the display filter `tcp.port == 8000`.

- Three separate HTTP connections, each with a TCP three-way handshake (`SYN`, `SYN/ACK`, `ACK`) and a `200 OK` response.
- In the POST request packet I confirmed: `Request Method: POST`, `Request URI: /`, `Content-Length: 30` and `Content-Type: application/x-www-form-urlencoded`.
- Wireshark's **HTML Form URL Encoded** section split the body into two fields: `username = testuser` and `action = login`.

So the form data really is carried in the HTTP **request body** and Wireshark can read it in clear text.

### 5. Comparing the GET and POST packets

When I looked through my saved capture I noticed that two GET requests looked different:

- In one GET request the values were in the URL **path** (no `?`), so they were not a query string.
- In another GET request the query string was empty even though a body was present.

That helped me separate three things: the **URL path**, the **query string** and the **request body**. I also learned to be careful with what the server's *response* prints (`Method: POST`, `Body: ...`): that is the server repeating what it received, not proof of what was on the wire. To check the real POST body I looked at the **request** packet and its form fields.

### 6. Saving the capture and moving it to the Mac

- I saved the capture as `.pcapng`, but the file name ended up as `web-lab3.pcapng .pcapng` by mistake.
- I found it with `find`, `realpath` and `ls` (176,420 bytes) and fixed the name with `mv` to `web-lab3.pcapng`.
- I briefly started a temporary Python HTTP file server on port 8765 on the VM and downloaded the file on the Mac with `curl -fL` (saved as `~/Downloads/web-lab3.pcapng`).
- I opened it directly in Wireshark on the Mac and analysed the packets there.

### 7. Mistakes I made and fixed

- I placed the backslashes (`\`) of a multi-line command wrongly and got `curl: (3) URL rejected`. Running the POST command on a single line fixed it.
- The capture file was saved under the wrong name. `find`, `realpath` and `mv` solved it.
- Lesson: check command syntax and file paths before assuming the tool is broken.

### 8. Cleanup

- Stopped the temporary file server on port 8765.
- Stopped the lab server on port 8000 and shut down the VM.
- Kept `post_server.py` for later labs and the capture on my Mac.

## What I observed

- With **GET** the data is in the URL (query string). With **POST** it is in the request body and the URL stays clean.
- `Content-Type: application/x-www-form-urlencoded` and `Content-Length` describe a form body.
- A server's response can repeat the request, so it is not a reliable way to prove what was sent. The request packet is.
- Wireshark can decode form fields and shows them in clear text over plain HTTP.

## Security angle

- **POST is not encryption.** The values are hidden from the URL, but over plain HTTP they are still readable in the packet, as Wireshark showed for `username` and `action`.
- **GET data ends up in more places:** URLs can be stored in browser history, server logs and proxy logs, so passwords and tokens should never be put in a query string.
- **Sensitive data needs HTTPS/TLS** in addition to using POST.
- **Server-side input:** both the query string and the body are user-controlled input and must be validated.

## What I learned

- How to write `do_GET()` and `do_POST()` handlers and read the body with `Content-Length`.
- How to send GET and POST requests with `curl -v`, `-X` and `-d`.
- How to capture loopback traffic with the `lo` interface and the filter `tcp.port == 8000`.
- How to read POST form fields in Wireshark (HTML Form URL Encoded).
- How to tell apart URL path, query string and request body.
- How to find and rename a file, and move it from a VM to the Mac.

## Commands used

| Command | Purpose |
|---------|---------|
| `python3 post_server.py` | Start the GET/POST lab server on port 8000 |
| `curl -v "<url>?a=b"` | GET request with a query string |
| `curl -v -X POST -d "a=b" <url>` | POST request with a form body |
| `tcp.port == 8000` | Wireshark display filter (capture on `lo`) |
| `find`, `realpath`, `ls`, `mv` | Locate, inspect and rename the capture file |
| `curl -fL <url> -o <file>` | Download a file from the VM to the Mac |

## Next

- Do the same comparison over **HTTPS** and see what Wireshark can no longer read (Network Lab #10).
- Add a simple login form and compare cookie-based sessions with plain requests.
- Authentication vs authorization.
