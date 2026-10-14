# 14 Oct 2026 – TryHackMe: DNS in Detail and HTTP in Detail

Today I completed two more rooms in TryHackMe's Pre Security path. These are my own short notes on the main ideas. They connect directly to my [Network Lab #3](../labs/2026-10-13-network-lab-3-subnetting-dhcp-dns.md), where I checked DNS on my own VM.

## What I covered

- DNS in Detail
- HTTP in Detail

## DNS in Detail

### How a domain name is built

Reading `admin.tryhackme.com` from right to left:

| Part | Example | Notes |
|------|---------|-------|
| **TLD** | `.com` | gTLD (generic: `.com`, `.org`, `.online`) or ccTLD (country: `.co.uk`, `.ca`) |
| **Second-level domain** | `tryhackme` | Max 63 characters; a–z, 0–9 and hyphens |
| **Subdomain** | `admin` | Same rules; can be chained (`a.b.example.com`); full name max 253 characters |

### Record types

| Record | Points to | Example use |
|--------|-----------|-------------|
| `A` | IPv4 address | `www` → `104.26.10.229` |
| `AAAA` | IPv6 address | same idea for IPv6 |
| `CNAME` | another domain name | `store.example.com` → a hosting provider's name, then a second lookup for its IP |
| `MX` | mail servers, with a **priority** | lower number is tried first; others are backups |
| `TXT` | free text | domain ownership checks, and email security such as **SPF** and **DMARC** |

### What happens during a lookup

1. The computer checks its **local cache**.
2. If the answer is not there, it asks a **recursive resolver** (usually the ISP's or one I choose), which also has a cache.
3. If the resolver doesn't know, it asks a **root server**, which points it to the right **TLD server** (for example `.com`).
4. The TLD server points to the domain's **authoritative name server**, which holds the real records.
5. The answer travels back to the resolver and then to me, and is **cached for its TTL** (in seconds) to avoid repeating the whole journey.

### Practice with `nslookup`

I queried different record types with `nslookup --type=A`, `--type=CNAME`, `--type=MX` and `--type=TXT`:

- `127.0.0.53` as the server again meant the local `systemd-resolved` stub, as in Lab #3.
- **`NXDOMAIN`** means the name (or that record type) does not exist.
- **"Non-authoritative answer"** means the reply came from a resolver's cache, not directly from the authoritative server.

## HTTP in Detail

### HTTP vs HTTPS

- **HTTP** is the set of rules a browser uses to fetch web content.
- **HTTPS** adds encryption, so others can't read the traffic, and it proves you are talking to the real server.

### Parts of a URL

`http://user:password@tryhackme.com:80/view-room?id=1#task3`

| Part | Example | Meaning |
|------|---------|---------|
| Scheme | `http` | Protocol to use |
| User | `user:password` | Optional credentials (a bad idea in practice) |
| Host | `tryhackme.com` | Domain or IP |
| Port | `80` | 80 for HTTP, 443 for HTTPS by default |
| Path | `/view-room` | Resource on the server |
| Query string | `?id=1` | Extra parameters |
| Fragment | `#task3` | A position on the page; handled by the browser |

### Requests and responses

- The first line of a request is **method + path + version**, for example `GET / HTTP/1.1`, followed by headers and then a **blank line**.
- The first line of a response is **version + status code**, for example `HTTP/1.1 200 OK`, then headers, a blank line and the body.

### Methods

| Method | Purpose |
|--------|---------|
| `GET` | Read data |
| `POST` | Send data / create something (for example a login form) |
| `PUT` | Update existing data |
| `DELETE` | Remove data |

### Status codes

| Range | Meaning | Common examples |
|-------|---------|-----------------|
| 1xx | Information | rarely seen now |
| 2xx | Success | `200 OK`, `201 Created` |
| 3xx | Redirect | `301` permanent, `302` temporary |
| 4xx | Client error | `400`, `401` (not logged in), `403` (forbidden), `404`, `405` (wrong method) |
| 5xx | Server error | `500`, `503` (overloaded or maintenance) |

### Headers

- **Request:** `Host` (which site on a shared server), `User-Agent` (browser), `Content-Length` (size of the body), `Accept-Encoding` (supported compression), `Cookie`.
- **Response:** `Set-Cookie`, `Cache-Control`, `Content-Type` (HTML, image, PDF…), `Content-Encoding`.

### Cookies

HTTP is **stateless**: each request stands alone. A server sends `Set-Cookie`, and the browser sends that value back in a `Cookie` header with every later request, so the server can recognise the user. For logins the value is normally a random **session token**, not the password. Cookies can be inspected in the browser's developer tools (**Network** tab).

### Practice

In the room's request builder I sent `GET`, `GET` with a query parameter, `PUT` and `DELETE` requests, and a `POST` login with form data (`Content-Type: application/x-www-form-urlencoded`), then read each response.

## Why this matters for security

- **DNS** is a target and a tool: spoofed answers can send users to the wrong server, and `MX`/`TXT` records (SPF, DMARC) help stop email spoofing.
- **HTTP** is the base of web security: methods, parameters, headers and cookies are exactly what attacks like broken authentication and session hijacking target, which leads into the OWASP Top 10.
- Credentials in a URL or a non-HTTPS login send secrets in a form others may be able to read.

## Next steps

- Look at real cookies and headers in my browser's developer tools.
- Compare an HTTP and an HTTPS request in Wireshark.
- Continue the Pre Security path.

*No flags or room answers are included, only my own notes.*
