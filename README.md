# srtp-nv

SRTP, the Secure Real-time Transport Protocol, protects the packets of an
RTP media stream: it encrypts the payload, authenticates the packet and
rejects a packet that was sent before. SRTCP does the same for the RTCP
control packets that travel beside the stream. SRTP is specified in
[RFC 3711](https://www.rfc-editor.org/rfc/rfc3711), the 256-bit counter mode
in [RFC 6188](https://www.rfc-editor.org/rfc/rfc6188) and the AES-GCM
profiles in [RFC 7714](https://www.rfc-editor.org/rfc/rfc7714). This package
brings them to novo-lang. It works on the packets of
[rtp-nv](https://novo-lang.org/packages/rtp-nv), and it is the media
protection that WebRTC, SIP phones and RTSP cameras use.

## What it is

A **master key** and a **master salt** come from a key management protocol.
SRTP does not use the master key on the media. It derives from it, with a
pseudo-random function built on AES (RFC 3711 section 4.3), the **session
keys**: an encryption key, an authentication key and a session salt, once for
SRTP and once for SRTCP. A **protection profile** names the algorithms.

| Profile | Master key | Master salt | SRTP tag | SRTCP tag |
| --- | --- | --- | --- | --- |
| AES_CM_128_HMAC_SHA1_80 | 16 bytes | 14 bytes | 10 bytes | 10 bytes |
| AES_CM_128_HMAC_SHA1_32 | 16 bytes | 14 bytes | 4 bytes | 10 bytes |
| AES_CM_256_HMAC_SHA1_80 | 32 bytes | 14 bytes | 10 bytes | 10 bytes |
| AES_CM_256_HMAC_SHA1_32 | 32 bytes | 14 bytes | 4 bytes | 10 bytes |
| AEAD_AES_128_GCM | 16 bytes | 12 bytes | 16 bytes | 16 bytes |
| AEAD_AES_256_GCM | 32 bytes | 12 bytes | 16 bytes | 16 bytes |

An SRTP packet is the RTP header, which stays in the clear, the payload
encrypted, the **master key identifier** (MKI) if the session uses one, and
the authentication tag. The tag of the counter-mode profiles is an
HMAC-SHA1 of the header, the encrypted payload and the 32-bit **roll-over
counter**. The roll-over counter (ROC) counts how often the 16-bit sequence
number has passed 65535. It is not in the packet. Both ends keep it, and the
receiver estimates it from each sequence number (RFC 3711 section 3.3.1).
The AES-GCM profiles authenticate the header as additional data and put a
16-byte tag after the cipher.

An SRTCP packet keeps the first eight bytes of the compound RTCP packet in
the clear and carries a 31-bit **SRTCP index** and an **E bit** after the
payload. The index is in the packet, so the receiver does not estimate it. The
E bit says whether the rest of the packet is encrypted.

The receiver keeps a **replay list** of the 64 most recent indexes. A packet
whose index it has seen, or which is more than 63 below the highest, is
refused. The list moves only for a packet whose tag has been verified.

**Nothing is allocated.** A packet is a `Vec[u8; 1500]` that the caller
owns, protected and unprotected in place. A context is a `@value` struct of
about 1,800 bytes. The package builds for a Cortex-M with no heap.

## Install

```
novo pkg add srtp-nv
```

## Example

A sender and a receiver share a master key and salt. The sender protects an
RTP packet. The receiver unprotects it, and a copy of the same packet is then
refused as a replay.

```novo
use rtpbytes
use srtpaes
use srtpctx
use srtp

fn main() [io]
    var key: srtpaes.SrtpBytes = Vec.new()
    var salt: srtpaes.SrtpBytes = Vec.new()
    for i in 0..16
        let _ = key.push(i as u8)
    for i in 0..14
        let _ = salt.push((0xA0 + i) as u8)

    var tx = srtpctx.empty()
    var rx = srtpctx.empty()
    let _ = srtpctx.init(var tx, srtpctx.SRTP_AES128_CM_SHA1_80, var key, var salt, 0)
    let _ = srtpctx.init(var rx, srtpctx.SRTP_AES128_CM_SHA1_80, var key, var salt, 0)

    // An RTP packet: version 2, payload type 0, sequence 1, SSRC 0x11223344, payload 1 2 3.
    var p: rtpbytes.RtpBuf = Vec.new()
    for b in [0x80, 0x00, 0x00, 0x01, 0, 0, 0, 160, 0x11, 0x22, 0x33, 0x44, 1, 2, 3]
        let _ = p.push(b as u8)

    let sent = srtp.protect_rtp(var tx, var p)
    println("protect ${sent}, ${p.len()} bytes")        // protect 0, 25 bytes
    var copy = p
    println("unprotect ${srtp.unprotect_rtp(var rx, var p)}, ${p.len()} bytes")   // unprotect 0, 15 bytes
    println("again ${srtp.unprotect_rtp(var rx, var copy)}")                      // again 5
```

Build and test with `novo pkg build` and `novo test tests/srtp_tests.nv`;
`tests/coverage.sh` merges the coverage of all the suites.

## What the package contains

| Module | Contents |
| --- | --- |
| `srtp` | `protect_rtp`, `unprotect_rtp`, `protect_rtcp` and `unprotect_rtcp`, the length of an RTP header, and the master key identifier a packet carries. |
| `srtpctx` | The profiles and their key, salt and tag lengths, the context with its session keys and counters, the master key identifier, the SRTCP index, and the initialisation vectors of the counter mode and of AES-GCM. |
| `srtpkdf` | The key derivation function of RFC 3711 section 4.3, with the key derivation rate, for AES-128 and for AES-256 (RFC 6188). |
| `srtpaes` | AES counter mode over a packet, and the loading of a key from bytes. |
| `srtpgcm` | AES-GCM over a packet, with the additional data of RFC 7714 sections 8 and 9. |
| `srtpmac` | HMAC-SHA1 over the bytes of a packet, and the comparison of a tag. |
| `srtpindex` | The packet index and the roll-over counter, and the replay window of 64. |
| `srtpdtls` | The two contexts of a DTLS-SRTP handshake, from the bytes the handshake exports (RFC 5764 section 4.2). |
| `srtpsdes` | The SDES crypto attribute of RFC 4568, read and written. |

The block cipher, the multiplication of GCM and the compression function of
SHA-1 are those of [crypto-nv](https://novo-lang.org/packages/crypto-nv). The
packet buffer and the reads of an RTP header are those of
[rtp-nv](https://novo-lang.org/packages/rtp-nv).

Every `pub` declaration is documented where it is declared, and
[the package's page](https://novo-lang.org/packages/srtp-nv) is that
reference.

## The rules a user needs

1. **A context is one direction of one stream.** The roll-over counter, the
   highest sequence number and the replay list belong to a source. A sender
   holds a context for the SSRC it sends. A receiver holds one for each SSRC
   it accepts, set up from the same master key when the session has one.
   DTLS-SRTP gives each direction its own master key.
2. **The caller supplies a fresh key.** A master key and its salt must be
   random and must not be used for two sessions. The package does not choose
   one.
3. **A refusal changes nothing.** A packet that is refused, for its tag, its
   index or its identifier, is left as it was, and the replay list does not
   move.
4. **The order of the checks is the RFC's.** A receiver checks the master key
   identifier, then the replay list, then the tag, and decrypts only a packet
   whose tag is right (RFC 3711 section 3.3).
5. **A sender never repeats an index.** `protect_rtp` refuses an index it has
   protected, or one more than 63 below the highest, with
   `SRTP_ERR_REPLAY`. A counter-mode keystream or an AES-GCM IV used twice
   gives away the plaintext (RFC 3711 section 9.1, RFC 7714 section 8.4).
6. **A master key protects 2^48 SRTP packets and 2^31 SRTCP packets.** Past
   that, `protect_rtp` and `protect_rtcp` answer `SRTP_ERR_KEY_LIFE`. A new
   context needs a new master key.
7. **The first SRTCP packet carries the index 0.** RFC 3711 section 3.4 sets
   the index to zero before the first packet and says it is not reset after a
   re-key. A caller that builds a new context for a new master key carries the
   counter over with `rtcp_index` and `set_rtcp_index`. libsrtp starts at 1.
8. **A packet grows by its tag.** The buffer holds 1500 bytes. A packet whose
   protection would not fit is refused with `SRTP_ERR_NO_ROOM`.
9. **The master key identifier is sent, not authenticated.** A context holds
   one master key, so it checks that a packet carries its identifier. A
   receiver with several master keys holds a context for each and picks one
   with `packet_mki`. An identifier is up to 8 bytes.
10. **The key derivation rate re-derives the session keys.** With a rate of
    `2^n` the session keys change when `index DIV rate` does. The context
    derives them again for the packet whose quotient it does not hold.
11. **The 12-byte salt of AES-GCM is padded on the right.** RFC 7714 does not
    say where the 96-bit master salt sits in the 112-bit input of the key
    derivation. This package follows libsrtp: the salt is the first 12 bytes
    of a 14-byte salt that ends in two zero bytes.
12. **Tag comparison takes the same time whatever the bytes are.** `srtpmac`
    and `srtpgcm` compare every byte of a tag before they answer.
13. **The RTP header is not decrypted.** Header extensions stay in the clear
    (RFC 6904 encryption of extension elements is not applied).

## Keys from a handshake

`srtpdtls.contexts` takes the profile, the `2 * (key + salt)` bytes a DTLS
endpoint exports with the label `EXTRACTOR-dtls_srtp`, and whether it is the
client, and answers the sending and the receiving context. `srtpsdes.parse`
reads the value of an `a=crypto:` attribute into a `SdesCrypto`, and
`srtpsdes.init_ctx` sets a context up from it. `srtpsdes.write` writes one into
a buffer.

## Running on a microcontroller

All nine modules are `@tier(embedded)` throughout and allocate nothing, and
the package declares no effects. `tests/embedded_probe.nv` is a firmware
program for a device that streams audio. It derives the key of RFC 3711
appendix B.3, seals the packet of RFC 7714 section 16.1.1 and compares it
with the RFC's bytes, protects an AES_CM_128_HMAC_SHA1_80 packet to the bytes
libsrtp produces and refuses its replay, protects and unprotects an SRTCP
packet and refuses an altered one, runs an AES-GCM stream across the
roll-over, keys two ends from exporter bytes and reads the SDES attribute of
RFC 4568. It is compiled for a Cortex-M as part of this package's checks, so
the claim is built rather than asserted, and under QEMU it prints
`PASS: srtp-embedded`.

The image is 85,272 bytes of flash and 520 bytes of static RAM, including the
runtime and the board's startup code. The largest values the probe holds are
a context of 1,808 bytes, a packet buffer of 1,512 bytes and a GCM key of
512 bytes, all in the frame.

The AES of crypto-nv takes the expanded key by value, so each block copies
496 bytes. The package is written for correctness on a device, and the
throughput is that of a part with no AES hardware.

## Timing behaviour

The AES of crypto-nv has no lookup table and no branch or index that depends
on the key or the data. The multiplication of GCM selects with a mask. The
tag comparisons do not stop at the first difference. The package does not
claim resistance to power or fault attacks. The length of a packet is not a
secret.

## What is not included

- **AES-f8 mode, RFC 3711 section 4.1.2: NOT IMPLEMENTED.** The `F8_128_HMAC_SHA1`
  suites of RFC 4568 read as `SDES_ERR_SUITE`.
- **The NULL cipher, unauthenticated SRTP, and `UNENCRYPTED_SRTP`,
  `UNENCRYPTED_SRTCP` and `UNAUTHENTICATED_SRTP`.** The parameters are read and
  a context is refused for them.
- **AES-192.** RFC 6188 defines it. No registry of key management protocols
  names a profile for it that this package follows.
- **AES-GCM with an 8-byte tag, and an authentication tag after an AES-GCM
  packet.** RFC 7714 section 8.2 does not recommend the second.
- **A DTLS handshake.** `srtpdtls` takes the exported bytes. The handshake, and
  the `use_srtp` extension that agrees the profile, are a DTLS package's.
- **Several keys in one SDES attribute, FEC, and an MKI longer than 8 bytes.**
  They read as `SDES_ERR_PARAM`.
- **Header extension encryption**, RFC 6904, and the cryptex extension.
- **The SRTP transport of RTP over a stream, and ZRTP and MIKEY.**
- **A replay window other than 64.** The `WSH` hint of SDES is read and not
  used.
- **Sockets and timers.**

## Related packages

- [rtp-nv](https://novo-lang.org/packages/rtp-nv) reads and writes the RTP and
  RTCP packets that this package protects.
- [crypto-nv](https://novo-lang.org/packages/crypto-nv) holds the AES, GHASH
  and SHA-1 cores that `srtpaes`, `srtpgcm` and `srtpmac` are built over.
- [sdp-nv](https://novo-lang.org/packages/sdp-nv) reads and writes the session
  description that carries an `a=crypto:` line. Give `srtpsdes.parse` the value
  of the attribute.
- [stun-nv](https://novo-lang.org/packages/stun-nv) finds the address a media
  stream runs between.

## Tests

The suite is eight files under `tests/`. `tests/coverage.sh` reports the
coverage of `src/` merged across them.

`tests/srtpkdf_tests.nv` holds the keystream and the key derivation to RFC
3711 appendix B.2 and B.3 and RFC 6188 sections 7.1 and 7.2, byte for byte.
`tests/srtpgcm_tests.nv` holds every vector of RFC 7714 sections 16 and 17:
the RTP packet and the RTCP packet, AES-128 and AES-256, encrypted, tagged
only, and verified. `tests/srtpmac_tests.nv` holds the cases of RFC 2202.
`tests/srtpindex_tests.nv` holds the index estimate and the replay window to
the rules of RFC 3711 section 3.3.

`tests/srtp_tests.nv` compares whole packets with libsrtp 2.5. For each of
the six profiles it protects four RTP packets, one with a CSRC and an
extension, and three SRTCP packets, and one that is authenticated and not
encrypted, and checks both directions. It follows the roll-over of the
sequence number for two profiles. RFC 3711 and RFC 7714 print no complete
packet for the counter-mode profiles, so libsrtp, driven through Python's
ctypes, is the independent implementation. The same suite's key derivation
rate vectors come from a second implementation written from the RFC's text
in Python, which first agreed with libsrtp on the packets of rate 0.

`tests/srtp_edge_tests.nv` holds what the vectors cannot show: each refusal,
the window, the identifier, the key lifetimes and the rate.
`tests/srtp_keys_tests.nv` reads the examples of RFC 4568 section 6.1 and
exercises DTLS-SRTP.

## Licence

Apache-2.0. See `LICENSE`.
