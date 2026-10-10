# 10 Oct 2026 – TryHackMe Pre Security

Today I completed four rooms in TryHackMe's Pre Security path. These are my own notes on the main ideas.

## What I covered

- Data Representation
- Data Encoding
- What is Networking?
- Intro to LAN

## Key ideas

### Data representation

- Computers store everything as **binary** (0s and 1s). One binary digit is a bit; 8 bits make a byte.
- **Hexadecimal** (base 16, digits 0–9 and A–F) is a shorter way to write binary. One hex digit = 4 bits, so one byte = two hex digits.
- The same value can be written in different bases:
  `65 (decimal) = 0x41 (hex) = 01000001 (binary)`
- **Colour codes** use hex too: `#FF0000` is red, because it means Red = FF (255), Green = 00, Blue = 00.
- **Assembly** is a low-level language that sits just above machine code. It gave me an idea of how instructions and data look to the CPU.

### Data encoding

- **ASCII** maps characters to numbers (for example `A` = 65), but it only covers basic English letters, digits and symbols.
- **Unicode** gives a number to almost every character in every writing system: Latin, Cyrillic (for example `Ж`), and even emojis.
- **UTF-8** is a common way to store Unicode. ASCII characters still take 1 byte, while other characters, like Cyrillic letters or emojis, take more bytes.
- Takeaway: text is just numbers plus an agreed encoding. If two systems use different encodings, the text breaks.

### What is networking?

- A network is devices connected so they can share data.
- **IP address:** a logical address that identifies a device on a network and can change, for example when you join a different Wi-Fi.
- **MAC address:** a physical address burned into the network card by the manufacturer. It is used inside the local network (it can be spoofed, though).
- **Router:** connects different networks together and decides where packets should go.
- **Switch:** connects devices inside the same local network and forwards traffic to the right device using MAC addresses.

### Intro to LAN

- A **LAN** (Local Area Network) is a network in a small area, such as a home, office or university building.
- Devices in a LAN can be arranged in different **topologies** (for example star, bus or ring), and each has trade-offs in cost, reliability and how easy it is to grow.
- Switches and routers are the main building blocks that connect a LAN and link it to other networks, such as the internet.

## Why this matters for security

You can't protect what you don't understand. Encoding comes up in malware analysis and web attacks, for example hex dumps and encoded payloads. MAC and IP addresses are the basis for understanding attacks like spoofing and how traffic moves through a network.

## Next steps

- TCP vs UDP, ports, NAT and firewalls
- Basic Linux terminal commands
- Repeat my Network Lab #1

*No flags or room answers are included, only my own notes.*
