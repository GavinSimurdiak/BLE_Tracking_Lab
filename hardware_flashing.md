# Hardware Flashing & Packet Verification

With a keypair generated and firmware produced (see [setup_and_key_generation.md](setup_and_key_generation.md)), the next stage was getting that firmware running on real BLE hardware and confirming, at the packet level, that it was advertising correctly.

## Microcontroller Deployment (Espruino / Puck.js)

The target hardware was a **Puck.js**, a small Nordic nRF52-based Bluetooth Low Energy microcontroller that runs **Espruino**, a JavaScript interpreter built for embedded devices. Espruino was a good fit for this project since it let the generated advertisement payload be deployed as plain JavaScript rather than requiring a full native firmware build/flash toolchain.

Deployment workflow, using the browser-based **Espruino Web IDE**:

1. Powered the Puck.js with a coin-cell battery (CR2016, positive side matched to the marked terminal).
2. Connected to the device over **Web Bluetooth** directly from the browser — no drivers or native app needed. Since Puck.js devices advertise generically, multiple candidate devices can show up in the pairing list; the one with the strongest signal (or one confirmed with the physical LED check below) is the correct target.
3. Opened the generated advertisement script in the IDE and uploaded it to the device's **RAM** (rather than flash), so the payload could be iterated on quickly during testing.
4. Disconnected from the device after upload — the uploaded program only starts executing once the Web Bluetooth connection is torn down.

### Confirming device identity with the onboard LED

One addition beyond the base lab instructions: before uploading the final advertisement firmware, I used a short Espruino snippet to flip on the Puck.js's onboard LED (`LED1.set()`) as soon as I connected. Since several Puck.js devices can be broadcasting nearby at once (e.g., in a classroom setting), this gave an immediate, physical confirmation that the device paired in the Web IDE was actually the one in my hand, rather than relying solely on signal strength or trial and error. Once confirmed, I turned the LED back off (`LED1.reset()`) before flashing and deploying the real advertisement code — the finished "tag" has no visible indicators, consistent with how an actual lost-item tracker behaves.

## Packet Capture & Verification (nRF Sniffer + Wireshark)

To confirm the Puck.js was actually broadcasting a well-formed BLE advertisement (and not just trust the Espruino IDE's console output), I verified it over the air:

1. Connected an nRF52840 USB sniffer dongle and opened Wireshark.
2. Selected the **nRF Sniffer for Bluetooth LE** capture interface.
3. Filtered the capture down to just the device under test using its advertising (MAC) address:

   ```
   btle.advertising_address == <device MAC address>
   ```

4. Confirmed a steady stream of BLE advertising packets at the expected ~2 second interval, and inspected the **PDU type** of the captured packets to confirm the advertisement was structured as a connectionless, non-scannable advertisement — consistent with how a real offline-finding beacon advertises (it only needs to be *seen*, never connected to).

This step matters beyond just "does it work" — it's what let me independently validate the payload at the radio/link layer rather than trusting higher-level tooling, which is the same technique used to reverse-engineer or audit any BLE device's advertising behavior.

Next step: [report_fetching_location_recovery.md](report_fetching_location_recovery.md) covers how the beacon's location reports were retrieved and decrypted once real devices started picking it up.
