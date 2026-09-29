# Report Fetching, Decryption & Anti-Tracking Evaluation

Once the beacon (see [hardware_flashing.md](hardware_flashing.md)) was confirmed to be advertising correctly, the final stage was letting real nearby devices pick it up, retrieving the resulting location reports, decrypting them locally, and separately evaluating the platform-level anti-stalking protections that are supposed to catch exactly this kind of device.

## How Secure Location Retrieval Works

Retrieving a real AirTag's (or in this case, the Puck.js beacon's) location reports normally requires an authenticated Apple account and direct access to Apple's Find My network. Rather than using personal account credentials directly against Apple's infrastructure, report retrieval was routed through an intermediary relay used for the course:

1. The relay is given only the **hashed advertisement key** (see [setup_and_key_generation.md](setup_and_key_generation.md)) — never the private key, and never the raw public key beyond what was already broadcast over BLE.
2. It queries the upstream Find My network for any reports filed under that hash and returns them **still encrypted**.
3. Because both the relay and the upstream network only ever see ciphertext, neither is in a position to determine the beacon's actual location — only the machine holding the matching private key can do that.

This mirrors the privacy design goal of the whole system: the party doing the fetching (whether that's Apple's own infrastructure or an intermediary) is architecturally incapable of learning device locations, because it never has access to the decryption key.

## Asymmetric Decryption

Each retrieved report is opaque ciphertext until decrypted locally:

- A fetch/decrypt script pulls the encrypted reports for the beacon's hashed key.
- Each report is decrypted **locally**, using the EC private key generated at the very start of the project, which never left the local machine.
- Successfully decrypting a report recovers the finder device's self-reported **latitude, longitude, accuracy, and timestamp** for the moment it observed the beacon.
- Decrypted results are persisted to a local SQLite database (`tracks.db`), which can then be queried for a clean, de-duplicated location history:

  ```sql
  SELECT DISTINCT timestamp, latitude, longitude
  FROM reports
  ORDER BY timestamp;
  ```

- That output can be exported to CSV or KML for plotting the recovered movement history on a map.

The practical takeaway: possession of the private key is the *only* thing standing between "meaningless ciphertext floating through someone else's infrastructure" and "a full location history." Nothing about the retrieval path itself — not the relay, not the upstream network — needs to be trusted with plaintext at any point.

## Anti-Tracking Alert Evaluation

The final part of the exercise flipped the perspective from *building* a tracker to *evaluating the defenses against one*. The beacon's status byte was set so that it identifies itself as a lost-item tracker (the same way a real AirTag would), and I carried it with me during normal day-to-day movement to observe how Apple's and Google's built-in **unwanted-tracker detection** behaved:

- Both ecosystems are designed to alert a user if an unrecognized item tracker appears to be moving with them over an extended period, specifically to catch covert stalking use cases like the ones described in the disclaimer in [README.md](README.md).
- I recorded how long it took from carrying the beacon until receiving an alert (or noted if no alert appeared within a full day of carrying it).

This is a useful real-world signal for how effective (or how delayed) these protections actually are in practice — informative both as a privacy feature evaluation and as a reminder of the detection window an actual unwanted tracker would have before being flagged.
