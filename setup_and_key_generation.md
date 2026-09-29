# Setup & Key Generation

This stage covers standing up the local environment and generating the cryptographic material the rest of the project depends on. Everything here happens **entirely on the local machine** — no key material is ever transmitted, and no offline-finding report can be decrypted by anything other than the private key generated in this step.

## Environment Setup

The project runs from a Python virtual environment to keep dependencies isolated:

```bash
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt
```

Core dependencies included Python's `cryptography` library for elliptic-curve operations and `sqlite3` for local storage of decrypted reports. BLE capture/analysis was done from a Linux (Kali) environment with Wireshark and the nRF Sniffer for Bluetooth LE plugin installed, since Web Bluetooth flashing (via the Espruino Web IDE) needed to happen from the host OS rather than the VM.

## Elliptic-Curve Keypair Generation

The core of Apple's Find My offline-finding protocol is an elliptic-curve keypair per accessory:

- A **private key** is generated locally and never leaves the machine that created it. Losing this key permanently forfeits the ability to decrypt any future location reports for that device — there is no recovery path.
- The corresponding **public key** is what gets broadcast (in part) over BLE. Apple's infrastructure — and any nearby "finder" device — only ever sees this public key, never the private key.

This is the property that makes the whole system trackable-by-design for research purposes: Apple's Find My network has no way to distinguish a genuine AirTag's public key from a public key generated independently via a standard crypto library. The network isn't validating a manufacturer signature or attestation on the key — it just relays reports for whatever key it's asked about.

Generating the keypair produces three artifacts:

- `private_key.pem` — the EC private key (kept local, treated as a secret)
- `public_key.pem` — the corresponding EC public key
- A generated firmware payload containing the **x-coordinate of the public key**, formatted for the BLE advertisement (see [hardware_flashing.md](hardware_flashing.md))

## Hashing the Advertisement Key

Only part of the public key (its x-coordinate) is ever broadcast over BLE — this keeps the advertisement payload small enough to fit Bluetooth's advertising packet size limits. Devices that later want to *query* for reports don't send the raw public key either; instead, they compute:

```
hashed_adv_key = base64( SHA256( x_coordinate_of_public_key ) )
```

This hashed value acts purely as a **lookup index** — it's what gets sent to request "have any reports been filed for this key?" without revealing the key itself outside of what was already broadcast publicly over the air. It's the same identifier a "finder" device implicitly targets when it encrypts a location report against the broadcast public key, so the two sides of the protocol (the beacon and the report retrieval) stay linked without ever exposing anything Apple's infrastructure could use to determine the tag's actual location.

Next step: [hardware_flashing.md](hardware_flashing.md) covers flashing the generated firmware to the microcontroller and confirming the beacon is broadcasting correctly.
