# Changelog

All notable changes to srtp-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0

The first release.

### Added

- **`srtp`**, `protect_rtp`, `unprotect_rtp`, `protect_rtcp` and
  `unprotect_rtcp` over a `Vec[u8; 1500]` held by the caller, the packet
  forms of RFC 3711 sections 3.1 and 3.4 and RFC 7714 sections 8 and 9, the
  length of an RTP header, and the master key identifier of a packet.
- **`srtpctx`**, the six protection profiles (AES_CM_128_HMAC_SHA1_80 and
  `_32`, AES_CM_256_HMAC_SHA1_80 and `_32`, AEAD_AES_128_GCM and
  AEAD_AES_256_GCM), the context with its session keys for SRTP and SRTCP,
  the key derivation rate, the master key identifier, the SRTCP index with
  `rtcp_index` and `set_rtcp_index`, and the initialisation vectors of the
  counter mode and of AES-GCM.
- **`srtpkdf`**, the key derivation function of RFC 3711 section 4.3 for
  AES-128 and, as in RFC 6188, for AES-256.
- **`srtpaes`** and **`srtpgcm`**, AES counter mode and AES-GCM over a packet
  buffer, on the block cipher and the GHASH multiplication of crypto-nv.
- **`srtpmac`**, HMAC-SHA1 one byte at a time over a packet, and a tag
  comparison that reads every byte.
- **`srtpindex`**, the index estimate of RFC 3711 section 3.3.1 and the
  replay window of 64 over a 48-bit or a 31-bit index.
- **`srtpdtls`**, the sending and the receiving context of a DTLS-SRTP
  handshake from the exported bytes (RFC 5764 section 4.2).
- **`srtpsdes`**, the SDES crypto attribute of RFC 4568 read and written, and
  a context set up from it.
- **`tests/embedded_probe.nv`**, a firmware program for a device that streams
  media, built for `--target=nrf52-qemu`. It prints `PASS: srtp-embedded`
  under QEMU. The image is 85,272 bytes of flash and 520 bytes of static RAM.
- Eight test suites. The vectors of RFC 3711 appendix B, RFC 6188 section 7,
  RFC 7714 sections 16 and 17, RFC 2202 and RFC 4568 section 6.1 are read and
  built byte for byte, and whole packets of all six profiles are compared
  with libsrtp 2.5.

### Not included

- AES-f8 mode (RFC 3711 section 4.1.2), the NULL cipher, unauthenticated
  SRTP and AES-192.
- A DTLS handshake. The exported bytes are the caller's.
- Several keys in one SDES attribute, FEC parameters and a master key
  identifier longer than 8 bytes.
- Header extension encryption (RFC 6904) and replay windows other than 64.

### Known

- The 12-byte master salt of the AES-GCM profiles is not placed in the key
  derivation input by RFC 7714, and this release follows libsrtp: the salt is
  the first 12 bytes of a 14-byte salt that ends in two zero bytes. The
  packets of the tests are libsrtp's.
- The first SRTCP packet carries the index 0, as RFC 3711 section 3.4 reads.
  libsrtp sends 1, and the comparison tests set the index to 1 first.
