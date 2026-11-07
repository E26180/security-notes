# Web Lab #1 – HTTP requests, headers, status codes and cookies with `curl -v` and Wireshark

**Date:** 7 Nov 2026
**Setup:** Ubuntu VM (`enp0s1`, `192.168.64.2/24`) in UTM as the HTTP server, my Mac as the HTTP client
**Tools:** `python3 -m http.server`, a small Python handler (`cookie_server.py`), `curl` (`-v`, `-i`, `-H`, `-c`, `-b`), Wireshark, `ss`

## Goal

In [Network Lab #1](2026-10-09-network-lab-1-http-wireshark.md) I looked at HTTP only as packets. This lab starts my web-security month from the application side: I wanted to see the **request and response** in `curl -v`, compare them with the same packets in Wireshark, and understand **headers, status codes and cookies** before moving on to HTTPS and sessions.

## What I did and what I saw

### 1. HTTP server

On the VM I created a folder with a tiny page and started Python's built-in server on port 8000:

```bash
mkdir ~/web-lab1 && cd ~/web-lab1
# create index.html with a simple HTML page
python3 -m http.server 8000 --bind 0.0.0.0
```

The Mac was the client, so the HTTP connection really crossed between two systems.

### 2. Request and response with `curl -v`

```bash
curl -v http://192.168.64.2:8000/
```

| Line in the output | Meaning |
|--------------------|---------|
| `> GET / HTTP/1.1` | The request line: method, path, HTTP version |
| `> Host`, `> User-Agent`, `> Accept` | Request **headers** sent by curl |
| `< HTTP/1.0 200 OK` | The status line of the response |
| `< Content-Type: text/html`, `< Content-Length: 31` | Response headers describing the body |
| the HTML after the headers | The response **body** |

`>` marks what the client **sends** and `<` marks what it **receives**. Interesting detail: curl sent an `HTTP/1.1` request, but the simple Python server answered with `HTTP/1.0`. The two sides do not have to use the same version.

### 3. Status codes: 200 vs 404

```bash
curl -i http://192.168.64.2:8000/olmayan-dosya.html
```

The server answered `404 File not found`. The TCP connection and the HTTP exchange both worked; only the **resource** did not exist. So `200 OK` means the server found and returned the resource, and `404` means it answered correctly that it does not have it.

### 4. Cookies and `Set-Cookie`

First I sent a cookie by hand:

```bash
curl -H "Cookie: lab_session=test123" http://192.168.64.2:8000/
```

Then I wrote `cookie_server.py`, a small handler based on `SimpleHTTPRequestHandler` that adds `Set-Cookie: lab_session=test123; Path=/` to every response. From the Mac:

```bash
curl -c web-lab1-cookies.txt http://192.168.64.2:8000/      # save the cookie the server sets
curl -b web-lab1-cookies.txt http://192.168.64.2:8000/      # send the saved cookie back
```

| Header | Direction | Job |
|--------|-----------|-----|
| `Set-Cookie` | server → client | "Store this value" |
| `Cookie` | client → server | "Here is the value you gave me" |

The cookie was stored and then sent back on the next request. This lab only shows the **mechanism** with a test value. It is not real authentication or session management.

### 5. The same traffic in Wireshark

On the VM I captured on `enp0s1` and used the display filter `tcp.port == 8000`.

| Frames | What I saw |
|--------|------------|
| 49–51 | TCP three-way handshake (`SYN`, `SYN/ACK`, `ACK`) |
| 52 | `GET / HTTP/1.1` |
| 55 | `HTTP/1.0 200 OK` |

In the request I could read `Host`, `User-Agent`, `Accept` and the `Cookie` header. In the response I could read `Content-Type`, `Content-Length`, `Set-Cookie` and the HTML. Wireshark showed about **3.89 ms** between the request and the response on this local network. The terminal output of `curl -v` and the packets showed the same exchange.

### 6. Mistakes I made and fixed

- I typed a wrong IP address in one connection attempt. With `192.168.64.2` the request worked.
- I mistyped the name of the Python script and got `No such file or directory`. After correcting the file name the server started.
- In the cookie file path I had to watch the `~/` part. A path is only correct when it points to the right folder.

### 7. Cleanup

- Stopped the Python server with `Ctrl + C` and checked that nothing listens on the port: `ss -ltn '( sport = :8000 )'` returned no listener.
- Deleted the temporary cookie file on the Mac.
- Kept `index.html` and `cookie_server.py` on the VM for later labs.

## What I observed

- A request is a **request line + headers**, and a response is a **status line + headers + body**.
- `404` is a correct answer from a working server, not a network failure.
- `Set-Cookie` and `Cookie` are plain headers, and Wireshark shows them in clear text.
- The terminal view (`curl -v`) and the packet view (Wireshark) describe the same conversation.

## Security angle

- **Plain HTTP exposes everything:** headers, body and **cookies** are readable by anyone who can capture the traffic. For a real session cookie that would allow **session hijacking**. This is the reason HTTPS/TLS matters.
- **Cookies carry trust:** a server believes a client that sends the right cookie value. That is why real session cookies should be random, protected with flags such as `Secure` and `HttpOnly`, and sent only over HTTPS. I have not tested those flags yet.
- **Status codes leak information:** `404` versus other answers tell a client what exists on a server. Attackers use this when they probe for hidden files.
- **Headers reveal software:** the `Server` header of a basic server can show which software and version is running.
- **`0.0.0.0` is wide open:** binding to all interfaces makes the test server reachable from the network. That was fine on my private lab network, but it should be restricted on a real host.

## What I learned

- How to read `curl -v`: `>` is what I send, `<` is what I receive.
- The parts of an HTTP request and response, and the meaning of `200` and `404`.
- How `Set-Cookie` and `Cookie` work and how to test them with `curl -c` and `-b`.
- How to find the TCP handshake, the `GET` and the response in Wireshark with `tcp.port == 8000`.
- That a simple server and a client can talk different HTTP versions.

## Commands used

| Command | Purpose |
|---------|---------|
| `python3 -m http.server 8000 --bind 0.0.0.0` | Start a simple HTTP server |
| `curl -v <url>` | Show the full request and response |
| `curl -i <url>` | Show the response headers with the body |
| `curl -H "Cookie: name=value" <url>` | Send a custom header (a cookie) |
| `curl -c <file> <url>` | Save cookies from the response |
| `curl -b <file> <url>` | Send saved cookies |
| `tcp.port == 8000` | Wireshark display filter |
| `ss -ltn '( sport = :8000 )'` | Check that nothing listens on port 8000 |

## Next

- Do the same with **HTTPS** and look at what Wireshark can no longer read.
- Add `HttpOnly` and `Secure` flags to the cookie and see what `curl -v` shows.
- Add a simple login form and compare a cookie-based session with a plain request.
- Continue with HTTP methods (`POST`) and the first steps toward authentication vs authorization.
