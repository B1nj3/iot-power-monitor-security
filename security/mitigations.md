# Mitigations — IoT Remote Power-Status Indicator

**Author:** Ajibade Banji Segun
**Companion doc:** [`threat-model.md`](threat-model.md)
**Status:** In progress

This document tracks each identified vulnerability, the mitigation applied, and its current implementation status.

---

## Mitigation 1: Protect Device Credentials from Source-Code Exposure

**Vulnerability:** Plaintext credentials in `arduino_secrets.h`
**STRIDE category:** Information Disclosure
**Risk:** High
**Status:** ✅ Implemented

### The Problem

The Arduino IoT Cloud connection requires a Device ID and a Secret Key. By default, the Arduino Cloud sketch generator puts both into a file called `arduino_secrets.h`:

```cpp
#define SECRET_SSID "Wokwi-GUEST"
#define SECRET_OPTIONAL_PASS ""
#define SECRET_DEVICE_KEY "abcd1234-real-key-here"
```

This file is included in the sketch and compiled into the firmware. Two attack vectors follow:

1. **Repository exposure:** If `arduino_secrets.h` is accidentally committed to a public git repo, anyone can read the key and impersonate the device — publishing false `powerStatus` values to the owner's dashboard.
2. **Firmware extraction:** With physical access to a real ESP32, an attacker can dump the flash contents and recover the key even if the source was never published.

### The Mitigation

**Two-layer defence:**

**Layer 1 — Never commit the real secrets file to git.**

- `arduino_secrets.h` is added to [`.gitignore`](../.gitignore) so it cannot be accidentally staged.
- The repo instead contains [`arduino_secrets.h.example`](../firmware/arduino_secrets.h.example) — a template with placeholder values.
- Developers copy the example to `arduino_secrets.h` locally and fill in their real keys.

**Layer 2 (planned for real hardware) — Store secrets in ESP32 encrypted NVS.**

On real hardware (not Wokwi), the ESP32's Non-Volatile Storage supports flash encryption. Keys are written once during provisioning and never appear in source or in cleartext flash:

```cpp
#include <Preferences.h>

Preferences prefs;
prefs.begin("secure", false);
String deviceKey = prefs.getString("device_key", "");
```

Combined with the ESP32's Secure Boot and Flash Encryption features, this prevents key recovery even by an attacker with physical access.

### Verification

- Confirm `arduino_secrets.h` appears in `.gitignore` ✅
- Confirm the file is *not* visible in the repo file list ✅
- Confirm `arduino_secrets.h.example` contains only placeholder values ✅

---

## Mitigation 2: Enforce Cross-Check on Sensor Readings

**Vulnerability:** No integrity check on sensor state before cloud publish
**STRIDE category:** Tampering
**Risk:** Medium
**Status:** 🟡 In progress

### The Problem

The current sketch reads one analog value from one LDR and publishes it as truth:

```cpp
int sensorValue = analogRead(sensorPin);
bool deviceOn = (sensorValue > threshold);
powerStatus = deviceOn;
```

This is trivially fooled:

- Covering the LDR with tape → reports "off" when the appliance is actually running
- Shining a torch on the LDR → reports "on" when the appliance is actually off
- A single noisy reading at the wrong moment → false state change

### The Mitigation

**Debouncing + median filtering.** Instead of trusting a single reading, take multiple samples over a short window and require sustained agreement before flipping the reported state.

```cpp
const int SAMPLE_COUNT = 5;
const int STATE_CHANGE_THRESHOLD = 4;   // 4 of 5 samples must agree
int recentSamples[SAMPLE_COUNT];
int sampleIndex = 0;

void loop() {
  ArduinoCloud.update();

  recentSamples[sampleIndex] = analogRead(sensorPin);
  sampleIndex = (sampleIndex + 1) % SAMPLE_COUNT;

  int aboveThreshold = 0;
  for (int i = 0; i < SAMPLE_COUNT; i++) {
    if (recentSamples[i] > threshold) aboveThreshold++;
  }

  bool manualTest = digitalRead(buttonPin) == LOW;
  bool deviceOn = (aboveThreshold >= STATE_CHANGE_THRESHOLD) || manualTest;

  powerStatus = deviceOn;
  digitalWrite(ledPin, deviceOn ? HIGH : LOW);
  delay(100);
}
```

This eliminates transient noise and requires an attacker to sustain the tampering (holding a light on the sensor for several seconds) rather than a single flicker.

**Future work:** Add a secondary sensor (a current-clamp or voltage divider on the appliance's power lead) and only report "on" when *both* the LDR *and* the current sensor agree. This is a real cross-check and makes single-sensor spoofing ineffective.

### Verification

- In Wokwi: cover the LDR briefly (< 500 ms) and confirm the dashboard does not flip
- Cover the LDR for 3+ seconds and confirm the dashboard does flip
- Document both tests in [`attack-notes.md`](attack-notes.md)

---

## Mitigation 3: Signed Over-the-Air (OTA) Firmware Updates

**Vulnerability:** Unsigned OTA updates could allow malicious firmware to be pushed to the device
**STRIDE category:** Tampering, Elevation of Privilege
**Risk:** High (if OTA is enabled)
**Status:** 📋 Documented (not yet implemented)

### The Problem

The current project does not enable OTA updates, so this is a theoretical threat for now. But any real-world deployment will need remote updates — walking to every deployed sensor to reflash it is not viable. As soon as OTA is enabled, an unsigned update mechanism becomes the single highest-risk feature on the device: an attacker who can reach the update endpoint can replace the firmware entirely.

### The Mitigation

**Require every OTA payload to be cryptographically signed with a private key held by the developer, and verified by the device against a public key baked into firmware before flashing.**

Standard flow:

1. Developer builds new firmware image
2. Developer signs the image with their private RSA or ECDSA key → produces a signature file
3. Signed image is uploaded to the update server (Arduino Cloud, or self-hosted)
4. Device downloads the image and its signature
5. Device verifies the signature using its embedded public key
6. **Only if verification succeeds**, the device writes the new firmware to flash and reboots into it

The ESP-IDF framework supports this natively via `esp_secure_boot` and `esp_https_ota` with signature verification enabled in `menuconfig`. Reference: Espressif's Secure Boot v2 documentation.

**Why this matters:** without signature verification, an attacker who can spoof the update server (see [threat-model.md](threat-model.md) — Spoofing, rogue access point) can push any firmware they like, turning the sensor into a botnet node or a pivot point into the local network.

### Verification (when implemented)

- Attempt to flash an unsigned firmware image → device should reject it
- Attempt to flash an image signed with the wrong key → device should reject it
- Flash a correctly signed image → device should accept and boot it

---

## Summary Table

| # | Vulnerability | Mitigation | Status |
|---|---|---|---|
| 1 | Plaintext credentials in source | `.gitignore` + `.example` template; NVS encryption planned | ✅ Implemented |
| 2 | Single-sample sensor readings | Sample buffer + majority-vote filter | 🟡 In progress |
| 3 | Unsigned OTA updates | Cryptographic signature verification | 📋 Documented |

## Next Mitigations to Address

From [`threat-model.md`](threat-model.md), the following remain open:

- MQTT/TLS certificate pinning to prevent MitM on public networks
- Rate limiting on cloud publishes to prevent quota exhaustion
- WiFi credential
