# Network Lab #1 – HTTP server and TCP handshake in Wireshark

**Date:** 9 Oct 2026
**Setup:** Ubuntu VM (server) + macOS host (client), on the same virtual network
**Tools:** Python `http.server`, `curl`, Safari, Wireshark

## Goal

See what client–server communication looks like as real network traffic, instead of only reading about it.

## What I did

1. **Created a folder with a simple `index.html` page on the Ubuntu VM**

2. **Started an HTTP server on port 8000**
   ```bash
   python3 -m http.server 8000
   ```

3. **Found the VM's IP address:** `192.168.64.2`, on interface `enp0s1`.

4. **Connected from the Mac (client)**
   ```bash
   curl http://192.168.64.2:8000
   ```
   I also opened the same address in Safari.

5. **Checked the server logs.** Each request showed up like this:
   ```
   "GET / HTTP/1.1" 200 -
   ```
   `GET /` is the client asking for the main page, and `200` means OK: the request succeeded.

6. **Captured the traffic in Wireshark**
   - Interface: `enp0s1`
   - Display filter: `tcp.port == 8000` (to show only the packets for my server)

## What I observed

- **Source and destination:** each packet showed the Mac's IP and a random high source port on the client side, and `192.168.64.2:8000` on the server side.
- **TCP three-way handshake** before any HTTP data was sent:
  1. `SYN`: the client asks to open a connection
  2. `SYN/ACK`: the server agrees
  3. `ACK`: the client confirms, and the connection is established
- After the handshake, the **HTTP request** (`GET /`) went from client to server, and the **HTTP response** (`200 OK` + the HTML) came back.

## What I learned

- An **IP address** identifies the machine. A **port** identifies which service on that machine to talk to (here, 8000).
- The **client** starts the connection; the **server** listens and responds.
- HTTP runs on top of TCP. A TCP connection has to be set up with the handshake before any web data is exchanged.
- HTTP without TLS is **plaintext**, so traffic like this can be read by anyone on the network path. That is why HTTPS matters.

## Next

- Lab #2: enable the `ufw` firewall on Ubuntu, block and allow port 8000, and see what changes in Wireshark and `curl`.
