# Cryptick Token

<div align="center">

[![Release Suite](https://github.com/alphaseneca/cryptick-token/actions/workflows/release.yml/badge.svg)](https://github.com/alphaseneca/cryptick-token/actions/workflows/release.yml)
[![Pages Deployment](https://github.com/alphaseneca/cryptick-token/actions/workflows/pages.yml/badge.svg)](https://github.com/alphaseneca/cryptick-token/actions/workflows/pages.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![FIDO2 / WebAuthn Level 3](https://img.shields.io/badge/FIDO2-CTAP2.1-success.svg)](https://fidoalliance.org/)
[![NIST SP 800-73-5](https://img.shields.io/badge/NIST-PIV%20SmartCard-informational.svg)](https://csrc.nist.gov/publications/detail/sp/800-73/5/final)

### The 100% Open-Source Biometric Hardware Security Key & Cryptographic Hardware Enclave

[**Official Website & Product Showcase**](https://alphaseneca.github.io/cryptick-token/) • [**Try Interactive Simulator**](https://alphaseneca.github.io/cryptick-token/#simulator) • [**Hardware Datasheet**](https://alphaseneca.github.io/cryptick-token/#specs) • [**Schematic PDF**](assets/cryptick-token-pcb-v0.1.0-schematic.pdf)

</div>

---

## Overview

**Cryptick Token** is an open-source hardware security key engineered to replace passwords, SMS 2FA codes, and blind-touch security tokens forever. Built on the Espressif ESP32-S3 dual-core microcontroller with dedicated hardware cryptographic acceleration, Cryptick combines:

- **What You See Is What You Sign (WYSIWYS)**: 0.96" high-contrast OLED display directly shows the verified relying party domain and username before signing, preventing phishing attacks.
- **Biometric Optical Enclave**: R558S capacitive/optical fingerprint sensor (508 DPI, 120 slots) keeps biometric templates completely offline in isolated silicon.
- **Dual-Mode Connectivity**: Plug in directly via native **USB OTG** or authenticate wirelessly using **Bluetooth Low Energy (BLE 5.0)** with dual-GATT architecture.
- **Full Protocol Suite**: FIDO2 CTAP2.1 Passkeys, NIST SP 800-73-5 PIV SmartCard, RFC 6238 OATH-TOTP standalone authenticator, and native OpenSSH hardware keys.
- **1-Click Computer Login**: Instant macOS SmartCard pairing (`sc_auth` / CryptoTokenKit) and Linux PAM login.

---

## Ecosystem Architecture & Submodules

This repository serves as the central consolidator and product home for the entire Cryptick Token ecosystem. Each component is tracked as a Git submodule:

| Submodule / Component | Repository | Description | Tech Stack |
| :--- | :--- | :--- | :--- |
| **Hardware & PCB** | [`cryptick-token-pcb`](https://github.com/alphaseneca/cryptick-token-pcb) | 2-layer PCB design, schematics, BOM, and JLCPCB manufacturing release | KiCad 8.x, KiForge |
| **Firmware & Core** | [`cryptick-token-code`](https://github.com/alphaseneca/cryptick-token-code) | ESP32-S3 firmware, CTAP2.1, PIV CCID applet, BLE GATT, and LittleFS enclave | C++ / PlatformIO |
| **Desktop Daemon & CLI** | [`cryptick-token-daemon`](https://github.com/alphaseneca/cryptick-token-daemon) | Background daemon, `cryptick` CLI, macOS SmartCard / Linux PAM, OpenSSH middleware | Rust (Cargo) |
| **Browser Extension** | [`cryptick-token-extension`](https://github.com/alphaseneca/cryptick-token-extension) | Manifest V3 WebExtension passkey bridge and live 30s TOTP companion | JavaScript / MV3 |

```
                                  ┌───────────────────────────────┐
                                  │         cryptick-token        │
                                  │   (Consolidated Main Repo)    │
                                  └───────────────┬───────────────┘
                     ┌────────────────────────────┼────────────────────────────┐
                     ▼                            ▼                            ▼
       ┌──────────────────────────┐ ┌──────────────────────────┐ ┌──────────────────────────┐
       │    cryptick-token-pcb    │ │   cryptick-token-code    │ │  cryptick-token-daemon   │
       │  (KiCad Hardware & BOM)  │ │  (ESP32-S3 Firmware/C++) │ │ (Rust Host Daemon & CLI) │
       └──────────────────────────┘ └──────────────────────────┘ └─────────────┬────────────┘
                                                                               │
                                                                 ┌─────────────▼────────────┐
                                                                 │ cryptick-token-extension │
                                                                 │  (WebExtension Bridge)   │
                                                                 └──────────────────────────┘
```

---

## Cloning the Consolidated Monorepo

To clone this repository along with all child submodules in one command:

```bash
# Clone recursively
git clone --recurse-submodules https://github.com/alphaseneca/cryptick-token.git
cd cryptick-token

# If you cloned without --recurse-submodules, initialize them:
git submodule update --init --recursive
```

To update all submodules to their latest remote commits:

```bash
git submodule update --remote --merge
```

---

## Cryptick Token vs. The Competition

| Feature | Cryptick Token | Standard USB/NFC Key | Biometric USB Key | Open-Source USB Key |
| :--- | :---: | :---: | :---: | :---: |
| **Physical Display (WYSIWYS)** | **0.96" OLED Screen** | None (Blind Touch) | None (Blind Touch) | None |
| **Biometric Fingerprint Sensor** | **508 DPI Enclave** | None | Capacitive Area | None |
| **Wireless Bluetooth (BLE 5.0)** | **Dual-GATT (iOS/Android/Laptops)** | NFC Only | None (Wired Only) | NFC Only |
| **Wired Connection** | **Native USB OTG** | USB | USB | USB |
| **100% Open Hardware & Code** | **Full KiCad & C++/Rust** | Proprietary Closed | Proprietary Closed | Open Source |
| **Standalone TOTP 2FA Display** | **Direct on Screen** | Requires App | Not Supported | Not Supported |
| **1-Click Computer Login** | **macOS & Linux PAM** | Complex Setup | Limited | Limited |
| **Rechargeable Battery** | **Integrated LiPo (BQ25185)** | No Battery | No Battery | No Battery |
| **Starting Price** | **$49 DevKit / $69 Pro** | $55 | $90+ | $50 |

---

## Automated Tag Release Workflows

Each repository in the Cryptick Token family contains an automated GitHub Actions release workflow triggered on tag pushes (`v*`):

- **`cryptick-token`**: Bundles documentation, website distributions, and schematics.
- **`cryptick-token-pcb`**: KiForge builds Gerbers, BOM, drill, and pick-and-place files.
- **`cryptick-token-code`**: PlatformIO compiles production firmware (`.bin`), bootloaders, and flash scripts.
- **`cryptick-token-daemon`**: Cargo cross-compiles release binaries for macOS (arm64), Linux (x86_64), and Windows.
- **`cryptick-token-extension`**: Packages Manifest V3 extension archives for Chrome, Brave, and Firefox.

To trigger a release across all components:

```bash
git tag v0.1.0
git push origin v0.1.0
```

---

## License

All Cryptick Token source code and documentation are released under the [MIT License](LICENSE).
Hardware designs are released under CERN-OHL-S v2 / Open Source Hardware guidelines.