# P2P RTC Tunnel Proxy

> [!CAUTION]
> **⚠️ DANGER:** this tool is intentionally weak. Do not use it in the real world for anything than can take bring to you. ***PLEASE, DO TAKE CARE WITH THIS TOOL***.
```diff
- DANGER: this tool is intentionally weak. Do not use it in the real world for anything than can take risk to you. PLEASE, DO TAKE CARE WITH THIS TOOL.
```
<div style="padding: 15px; border: 1px solid red; border-radius: 4px; color: #721c24; background-color: #f8d7da;">
    <b>&#9888; DANGER:</b> this tool is intentionally weak. Do not use it in the real world for anything than can take risk to you. <em>PLEASE, DO TAKE CARE WITH THIS TOOL</em>.
</div><br>

## What

A modular P2P proxy that uses WebRTC (*aiortc*) to create encrypted tunnels between peers behind NAT. Data is transported as custom RTP payloads over SRTP, making  traffic indistinguishable from a video call to deep packet inspectors (this is **critical** for this particular project).

Local browsers connect to a regular HTTP proxy on localhost (8080/tcp, 8081/tcp); requests are tunneled through the P2P connection and served transparently.

## Architecture

```
Browser A ──► HTTP Proxy A ──► [Crypto Pipeline] ──► RTP/SRTP ──► ICE/NAT
                                                                      │
                                                          Signaling Server
                                                                      │
Browser B ◄── HTTP Proxy B ◄── [Crypto Pipeline] ◄── RTP/SRTP ◄── ICE/NAT
```

## Key Features

The idea of chaining cryptoalgorithms is taken from Truecrypt tool and I want to give them credit here :)

- **Modular crypto pipeline**: Chain multiple encryption algorithms (like TrueCrypt)
  e.g. AES-256-GCM → ChaCha20-Poly1305 → Camellia-256-GCM
- **Modular compression**: Chain compressors (zstd → brotli → lz4)
- **Custom RTP payload**: Reliable transport over SRTP with own ACK/retransmit
- **NAT traversal**: Full ICE via aiortc (STUN/TURN) (**CAREFUL**: any STUN/TURN dependence is a big security risk)
- **Traffic mimicry**: Packets look like standard VP8 video RTP to DPI
- **HTTP proxy**: Transparent to browsers via localhost proxy

## Project Structure

```
p2p-rtc-tunnel/
├── main.py                  # Entry point
├── config.py                # Configuration and CLI
├── crypto/
│   ├── __init__.py
│   ├── base.py              # Abstract cipher/compressor interfaces
│   ├── ciphers/
│   │   ├── __init__.py
│   │   ├── aes_gcm.py       # AES-256-GCM
│   │   ├── chacha20.py      # ChaCha20-Poly1305
│   │   ├── camellia.py      # Camellia-256-GCM (placeholder)
│   │   └── null.py          # No-op cipher for testing
│   ├── compressors/
│   │   ├── __init__.py
│   │   ├── zstd_comp.py     # Zstandard
│   │   ├── brotli_comp.py   # Brotli
│   │   ├── lz4_comp.py      # LZ4
│   │   └── null.py          # No-op compressor
│   └── pipeline.py          # Chained crypto/compression pipeline
├── protocol/
│   ├── __init__.py
│   ├── frames.py            # Wire protocol frame definitions
│   ├── reliable.py          # Reliability layer (ACK, retransmit, ordering)
│   ├── multiplexer.py       # Channel multiplexing over single stream
│   └── flow_control.py      # Flow control / windowing
├── transport/
│   ├── __init__.py
│   ├── peer.py              # WebRTC peer connection management
│   ├── rtp_channel.py       # Custom RTP payload send/receive
│   └── srtp_wrapper.py      # SRTP encryption context
├── signaling/
│   ├── __init__.py
│   ├── server.py            # WebSocket signaling server
│   └── client.py            # WebSocket signaling client
├── proxy/
│   ├── __init__.py
│   ├── http_server.py       # Local HTTP proxy server
│   └── request_handler.py   # Request routing and response assembly
├── tests/
│   ├── test_crypto_pipeline.py
│   ├── test_protocol.py
│   └── test_integration.py
└── requirements.txt
```

## Usage

### 1. Start signaling server (on a public host): this should be running before the others (*clients*).

> ⚠️ This is one of the main critical weaknesses of the project: having an external dependence on a STUN/TURN service is **dangerous**.,

It will be called ***signal-server***.

```bash
$ python main.py signaling --host 0.0.0.0 --port 9000
```

### 2. Start peer A (offering side)

It is not relevant which side is offering or answering, but offering should start first. Not dedicated time to implement a *protocolo* for this.  

```bash
python main.py peer \
  --role offer \
  --signaling ws://signal-server:9000 \
  --room myroom \
  --secret "my-shared-secret" \
  --proxy-port 8080 \
  --crypto aes-256-gcm,chacha20-poly1305 \
  --compression zstd
```

### 3. Start peer B (answering side)

```bash
python main.py peer \
  --role answer \
  --signaling ws://signal-server:9000 \
  --room myroom \
  --secret "my-shared-secret" \
  --proxy-port 8081 \
  --crypto aes-256-gcm,chacha20-poly1305 \
  --compression zstd
```

### 4. Browse through the tunnel

Configure browser to use `localhost:8080` as HTTP proxy, or:

```bash
curl --proxy http://localhost:8080 http://example.com
```

## Crypto Pipeline

> ⚠️ CAREFUL with this crypto strategy as it might not be evident, but can be broken.

The crypto pipeline applies transformations in order:

```
Plaintext
  → Compress (zstd → brotli)          # compression chain
  → Encrypt (AES-256-GCM → ChaCha20)  # encryption chain
  → Wire
```

Decryption reverses the order:

```
Wire
  → Decrypt (ChaCha20 → AES-256-GCM)  # reverse encryption chain
  → Decompress (brotli → zstd)         # reverse compression chain
  → Plaintext
```

Each algorithm derives its own key from the shared secret using *HKDF* with an unique label, so chaining is cryptographically sound.

## Requirements

```
aiortc>=1.6.0
aiohttp>=3.9.0
cryptography>=42.0
zstandard>=0.22
brotli>=1.1
lz4>=4.3
```

## Pryor art

There are several projects in this league. For example:

https://sourceforge.net/projects/steganrtp/

The idea is not new, but the difference here is it could be enhanced.

## TODO and future work

- Create standalone binaries for "aunt Mary": these projects must be focused in citizens, not in hackers.
- Evolve the tool to be actually strong.
- Use Google Meet, Microsoft Teams and other RTC solutions as transport layer (***think about this, please :(***).
- Dedicate time to educate and promote **PRIVACY**, **ENCRYPTION** and **ANONYMITY**.
