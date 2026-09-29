# Build Your Own AirTag — BLE Offline-Finding Research Lab

## Overview

This project is a hands-on implementation and security analysis of Apple's **Find My** offline-finding protocol, built from a low-cost BLE microcontroller (a Puck.js) instead of a real AirTag. The goal was to understand the cryptographic and networking design of "crowd-sourced" location tracking systems by building a functionally equivalent device, deploying it, and recovering its location history exactly the way a real AirTag owner would — without Apple, or anyone else, ever seeing the device's location in the clear.

At a high level, the project involved:

1. Generating an elliptic-curve keypair locally and deriving a BLE advertisement payload from the public key, mimicking the structure Apple devices use to recognize a "lost" Find My accessory.
2. Flashing that advertisement to a Puck.js microcontroller so it broadcasts like a real offline-finding beacon.
3. Verifying the advertisement over the air using a BLE sniffer and Wireshark.
4. Letting nearby Apple devices anonymously detect the beacon, encrypt their own location with the broadcast public key, and relay a report upstream.
5. Retrieving the resulting encrypted location reports and decrypting them locally with the private key, which never left the machine that generated it.
6. Separately, evaluating Apple/Google's **anti-stalking (unwanted tracker) detection** behavior by carrying the beacon and observing how long it took before a phone flagged it as a potential tracker.

This mirrors the research described in Heinrich et al., *Who Can Find My Devices?* (PETS 2021), and the open-source OpenHaystack / macless-haystack projects, which demonstrated that Apple's Find My network cannot distinguish a real AirTag's broadcast key from any other elliptic-curve public key — including one generated with a standard crypto library and flashed to arbitrary hardware.

## Tech Stack & Key Technologies

- **Embedded / Firmware:** Puck.js (Nordic nRF52-based BLE microcontroller), Espruino (JavaScript runtime for embedded devices), Espruino Web IDE (Web Bluetooth flashing)
- **Cryptography:** Elliptic-curve key generation, ECIES-style asymmetric encryption/decryption, SHA-256 hashing, base64 encoding, Python `cryptography` library
- **Networking / Protocol Analysis:** Bluetooth Low Energy (BLE) advertising packets, nRF52840 USB sniffer dongle, Wireshark with the nRF Sniffer for Bluetooth LE plugin
- **Backend / Data:** Python, SQLite (local encrypted-report storage and decrypted track history), CSV/KML export for mapping
- **Environment:** Python virtual environments (`venv`), pip-managed dependencies, Linux (Kali) tooling for BLE capture

## Architectural Overview

```
 ┌─────────────┐   BLE advertisement    ┌──────────────────┐
 │  Puck.js     │ ─────────────────────▶ │ Nearby Apple      │
 │ (public key  │   (public key bytes)   │ devices (finders) │
 │  broadcast)  │                        └─────────┬─────────┘
 └─────────────┘                                    │ encrypts its own
        ▲                                           │ location with the
        │ local keygen                              │ broadcast public key
        │ (private key never leaves                 ▼
        │  the owner's machine)             ┌──────────────────┐
        │                                   │ Apple Find My     │
 ┌─────────────┐                            │ network (reports  │
 │  Owner's     │   hashed public-key ID    │ stored, indexed   │
 │  laptop      │ ─────────────────────────▶│ by key hash)      │
 │ (keypair,    │                           └─────────┬─────────┘
 │  decryption) │   encrypted reports (still          │
 │              │◀──encrypted, never decryptable──────┘
 └─────────────┘   by the network or relay)
        │
        ▼ local ECIES decryption with private key
 ┌─────────────┐
 │  tracks.db   │  →  CSV / KML export of recovered location history
 └─────────────┘
```

The design keeps three parties intentionally blind to the plaintext location: the finding device only reports what it sees (never anything about itself), the network/relay only ever handles ciphertext, and only the holder of the private key — generated and kept entirely client-side — can ever recover a readable location.

## Key Technical Takeaways

- **Practical asymmetric cryptography:** implemented the full lifecycle of an EC keypair — generation, public-key hashing/derivation for use as a lookup index, and later asymmetric decryption of untrusted, network-delivered ciphertext.
- **Protocol-level understanding of crowd-sourced offline finding:** traced how Apple's Find My network achieves its stated privacy goals (the network never learns device locations, finders never expose their own identity, and key rotation would normally prevent long-term tracking of production AirTags).
- **BLE advertisement analysis:** captured and inspected raw BLE advertising PDUs with a hardware sniffer and Wireshark, filtering by advertising address to confirm the beacon's on-air behavior matched the expected Find My format.
- **Embedded firmware deployment:** flashed and iterated on JavaScript firmware for a Bluetooth microcontroller over Web Bluetooth, including using the Puck.js's onboard LED as a quick physical identity check — toggling it on to visually confirm the correct device was paired before flashing the final advertisement firmware, then switching it back off before deployment.
- **Security/privacy control evaluation:** measured real-world effectiveness of Apple's and Google's built-in unwanted-tracker alert systems by observing how long it took for a phone to flag the beacon as a possible tracking device.

## Legal / Ethical Disclaimer

This project was completed entirely within a supervised, university-run security course lab, under an explicit scope-of-authorization agreement:

- Only a device I built and owned myself was ever advertised or tracked.
- The device was never placed on another person, another person's belongings, or a vehicle that wasn't mine, and no one else's tag was tracked.
- The hardware used has no speaker and is **not compliant** with the industry unwanted-tracking standard used by commercial trackers — it exists solely as an instructional stand-in for a real AirTag.
- All key material, location reports, and identifying details in this repository have been generated for, and are limited to, this lab exercise.
