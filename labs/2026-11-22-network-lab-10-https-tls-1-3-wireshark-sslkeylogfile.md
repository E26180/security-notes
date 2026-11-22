# Network Lab #10 – HTTPS, TLS 1.3 and Wireshark: what stays visible and what is encrypted

**Date:** 22 Nov 2026
**Setup:** Ubuntu VM (`enp0s1`, `192.168.64.2`) in UTM, capturing my own HTTPS traffic to `example.com`
**Tools:** `curl` (`-v`), Wireshark (`tcp.port == 443`), `SSLKEYLOGFILE`

## Goal

In [Web Lab #1](2026-11-07-web-lab-1-http-curl-cookies-wireshark.md) and [Web Lab #3](2026-11-20-web-lab-3-http-get-post-curl-wireshark.md) I could read HTTP headers, cookies and form data directly in Wireshark. This lab is the HTTPS side of the same story: how a TLS 1.3 connection is set up, **what Wireshark can still see** and **what it can no longer read**. As a bonus I decrypted *my own* session with a key log file to understand the difference between breaking TLS and being allowed to decrypt a session.

## What I did and what I saw

### 1. Preparing the capture

I started the Ubuntu VM and checked its network interfaces: `enp0s1` with `192.168.64.2`. In Wireshark I started a live capture on `enp0s1` and used the display filter `tcp.port == 443`.

### 2. An HTTPS request with TLS 1.3

From the VM I sent an HTTPS request with `curl`:

```bash
curl -v https://example.com
```

I configured the connection to use TLS 1.3 and HTTP/1.1. In the `curl -v` output I saw the TLS handshake lines, the server certificate being verified and `HTTP/1.1 200 OK`. So the connection worked at the TLS layer **and** at the HTTP layer.

### 3. The TLS handshake in Wireshark

| Packet | What I saw |
|--------|-----------|
| **ClientHello** | The client's TLS offer. The **SNI** (Server Name Indication) field shows `example.com` in plain text |
| **ServerHello** | The server picked cipher suite `TLS_AES_256_GCM_SHA384` and a key share with `X25519MLKEM768` |

Two things stood out:

- **SNI is visible.** Even with HTTPS, anyone on the path can see *which host name* I connect to. HTTPS hides the content, not every piece of metadata.
- **Version fields can mislead.** TLS 1.3 packets still carry older "legacy" version values for compatibility. The version that is really used is negotiated in the `supported_versions` extension.

### 4. The encrypted traffic

After the handshake Wireshark only shows **Encrypted Application Data**. The HTTP request and the response, including headers and body, are not readable in the normal packet view.

Compared with my earlier labs:

| | Plain HTTP (Web Lab #1, #3) | HTTPS (this lab) |
|---|-----------------------------|------------------|
| Request line, headers, cookies | Readable | Encrypted |
| Form data in POST body | Readable | Encrypted |
| Host name | Readable (`Host` header) | Readable in SNI only |
| IP addresses, ports, packet sizes and timing | Readable | Readable |

### 5. Bonus: decrypting my own session with `SSLKEYLOGFILE`

After finishing the normal lab goals I tried one more experiment:

1. I set `SSLKEYLOGFILE` so that the client application wrote the **TLS session keys** of my own connection to a file on the VM.
2. I told Wireshark to use this file in its TLS settings and made a new HTTPS capture.
3. With the filter `http && tcp.port == 443` I could now read `GET / HTTP/1.1` and `HTTP/1.1 200 OK` inside the previously encrypted connection.
4. When I removed the key file from the Wireshark settings, the decrypted HTTP view disappeared again.

This showed me that I had **not broken TLS**. TLS stayed secure. I was able to read the traffic only because I, as one of the two endpoints, exported the session keys myself.

### 6. Files and cleanup

- Saved the capture as `network-lab10-https.pcapng`. It stays on my machine and is **not** part of this repository.
- Removed the key log file from Wireshark's TLS settings and **deleted the key file** from the VM with the terminal.
- Shut down the Ubuntu VM.

## What I observed

- TLS 1.3 starts with a ClientHello and ServerHello that are readable, and then everything after the handshake is **Encrypted Application Data**.
- SNI exposes the destination host name, even though the content is encrypted.
- A packet capture alone does not give access to the data. Content can only be read with the session keys.
- Removing the key file immediately removed the decrypted view, so the keys really are what makes decryption possible.

## Security angle

- **HTTPS protects content, not all metadata:** IP addresses, ports, SNI, packet sizes and timing stay visible to an observer.
- **Session keys are secrets.** A key log file lets anyone with the matching capture decrypt that session, including cookies and passwords. I treat key log files and the captures made with them as sensitive: they never go into a public repository, a screenshot or a shared folder, and I delete the key file when the experiment is over.
- **Decrypt only what you are allowed to.** Using `SSLKEYLOGFILE` on my own client and my own test traffic is a normal debugging and analysis technique. Doing it on someone else's traffic would not be authorised.
- **Why it matters:** analysts use this to debug applications and to investigate incidents when they own the endpoint. It also shows why compromising a client machine can defeat HTTPS for that client.
- **HTTPS is not a guarantee that a site is trustworthy:** it means the connection is encrypted and the server presented a valid certificate, not that the site is safe.

## What I learned

- The order of a TLS 1.3 handshake and what ClientHello and ServerHello contain.
- What SNI is and why it is visible.
- What a cipher suite and a key share are, and how to find them in ServerHello.
- Why packets show **Encrypted Application Data** and what an observer can still see.
- How `SSLKEYLOGFILE` and Wireshark's TLS settings decrypt my own session.
- The difference between breaking encryption and decrypting with legitimate keys.
- How to keep secrets (key files, captures) out of public repositories.

## Commands used

| Command / setting | Purpose |
|-------------------|---------|
| `curl -v https://example.com` | HTTPS request, shows the TLS handshake and the certificate check |
| `tcp.port == 443` | Wireshark display filter for HTTPS |
| `http && tcp.port == 443` | Show decoded HTTP inside TLS once keys are loaded |
| `SSLKEYLOGFILE` | Client option to write TLS session keys to a file (my own sessions only) |

## Next

- Test the same HTTPS request with a certificate error and see how curl reports it.
- Look at the certificate chain in more detail.
- Authentication vs authorization and how sessions and cookies work together.
