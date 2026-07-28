# Threat Model — IoT Remote Power-Status Indicator

**Author:** Ajibade Banji Segun
**Method:** STRIDE (Spoofing, Tampering, Repudiation, Information Disclosure, Denial of Service, Elevation of Privilege)
**Status:** In progress

---

## 1. System Overview

The system consists of four components:

1. **Monitored appliance** — the physical device being observed (inverter, freezer, etc.)
2. **Sensor node** — ESP32 microcontroller with an LDR sensor and LED indicator
3. **Network** — WiFi connection (Wokwi-GUEST in simulation) and MQTT over TLS to the cloud
4. **Cloud & dashboard** — Arduino IoT Cloud stores the `powerStatus` variable and displays it to the user via web or mobile

See [`docs/1_block_diagram.png`](../docs/1_block_diagram.png) for the full architecture.

## 2. Assets

What we are trying to protect:

- **Integrity of the powerStatus signal** — the user must be able to trust the on/off reading
- **Device credentials** — the Device ID and Secret Key that authenticate the ESP32 to Arduino Cloud
- **Availability of the monitoring service** — the dashboard must reflect current state in real time
- **User privacy** — knowing when someone's appliances are on/off can reveal occupancy patterns

## 3. Trust Boundaries

- Between the appliance and the sensor (physical access to the sensor is possible)
- Between the sensor and the WiFi network (open or weak WiFi is a risk)
- Between the sensor and Arduino Cloud (MQTT/TLS boundary)
- Between the cloud and the user's browser/phone (authentication boundary)

## 4. STRIDE Analysis

### 🎭 Spoofing

| Threat | Description | Risk |
|---|---|---|
| Rogue device impersonating the sensor | If the Device ID and Secret Key are extracted from firmware, an attacker could publish false `powerStatus` values from anywhere in the world | **High** |
| Rogue access point | An attacker sets up an open WiFi network named `Wokwi-GUEST` to capture the ESP32's traffic | **Medium** |
| Phishing the user's Arduino Cloud login | Attacker gains access to the dashboard by stealing user credentials | **Medium** |

### 🔧 Tampering

| Threat | Description | Risk |
|---|---|---|
| Modifying `powerStatus` in transit | If TLS is not properly verified, a man-in-the-middle could flip the boolean | **Medium** |
| Physically tampering with the LDR | Covering the sensor with tape reports the appliance as off; shining a light reports it as on | **Low** (physical access required) |
| Modifying firmware via unauthorized OTA update | If OTA updates are not signed, malicious firmware could be pushed to the device | **High** |

### 📝 Repudiation

| Threat | Description | Risk |
|---|---|---|
| No audit log of state changes | If the sensor sends a false reading, there is no timestamped record to prove what was reported when | **Low** for this use case |

### 👁️ Information Disclosure

| Threat | Description | Risk |
|---|---|---|
| Plaintext credentials in `arduino_secrets.h` | Anyone with access to the source (GitHub, physical flash) can extract keys | **High** |
| Unencrypted WiFi traffic | Open networks expose MQTT payloads to anyone on the same network | **High** |
| Occupancy pattern leakage | `powerStatus` history reveals when a home is occupied | **Medium** |

### ⛔ Denial of Service

| Threat | Description | Risk |
|---|---|---|
| WiFi jamming | Simple radio jamming disconnects the ESP32 from the cloud | **Low** (requires proximity) |
| Flooding the cloud with junk updates | A compromised device could exhaust the user's Arduino Cloud quota | **Medium** |
| Power cutting the sensor itself | If someone unplugs the sensor, the dashboard shows stale data — this is exactly the failure mode the project is meant to prevent | **Medium** |

### ⬆️ Elevation of Privilege

| Threat | Description | Risk |
|---|---|---|
| Compromised sensor pivoting to other network devices | An attacker who takes over the ESP32 could scan the local network for other devices | **Medium** |
| Exploiting an ESP32 firmware vulnerability | Known vulnerabilities in the ESP32 WiFi stack could give an attacker remote code execution | **Medium** |

## 5. Mitigation Plan

Detailed implementation in [`mitigations.md`](mitigations.md). Summary:

| Vulnerability | Mitigation |
|---|---|
| Plaintext credentials in source | Move secrets to ESP32 encrypted NVS (non-volatile storage) |
| Secrets committed to git | `.gitignore` for `arduino_secrets.h`, provide `.example` template only |
| Unverified TLS | Enable certificate pinning to Arduino Cloud CA |
| Unsigned OTA updates | Require signed firmware images before flashing |
| Data integrity | Add HMAC signature to sensor readings before publish |
| Physical tampering | Add a secondary sensor (current draw, voltage) to cross-check LDR readings |

## 6. Out of Scope

- Physical security of the appliance itself
- Cloud provider (Arduino) internal security controls
- User endpoint device security (their phone or laptop)

## 7. References

- Microsoft STRIDE threat modeling framework
- OWASP IoT Top 10 (2018)
- ESP32 Technical Reference Manual — Secure Boot and Flash Encryption
- Arduino IoT Cloud documentation
