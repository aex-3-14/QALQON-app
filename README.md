# QALQON

QALQON is an Android security app that checks installed apps and APK files for known threats, warns about risky installation prompts, and optionally filters DNS for supported browsers and messengers.

APK scanning runs on the device and works offline with the bundled threat list. QALQON supports Uzbek Latin, Uzbek Cyrillic, Russian, and English.

**Current source:** QALQON 1.0 · Android 8.0+ · Kotlin · `uz.qalqon.app`

## How protection works

### APK scanning

QALQON can scan installed apps, a single APK selected with Android’s file picker, or APKs it can access in shared storage. Scanning shared storage requires the relevant Android file-access permission. System apps have a separate scan option.

1. **Calculate the fingerprint.** QALQON calculates the APK’s SHA-256 hash locally. It also checks readable split APKs for installed apps.
2. **Look up known threats.** The scanner compares the fingerprint with its bundled and downloaded hash lists. A match is reported as a known threat.
3. **Inspect eligible APKs.** QALQON can examine manifest permissions and selected DEX code patterns in APKs from unverified sources. A sideloaded APK that requests SMS-related permissions can receive a high-risk warning. Unreadable metadata or an unverified installation source can also trigger a review warning.
4. **Save the result locally.** QALQON stores findings and caches fingerprints in app-private storage. New app installs and updates can trigger a post-install scan. Background scans are scheduled roughly every 12 hours when enabled.

Apps attributed by Android to a recognized store are checked by hash, including in detailed scan mode. The store catalog includes Google Play, Galaxy Store, AppGallery, Xiaomi, OPPO/HeyTap, Vivo/iQOO, HONOR, Palm Store, Amazon Appstore, Meizu, and Lenovo package IDs. An app’s store source gives context for the scan. It cannot establish that the APK is safe.

**How to read a result:** A **Known threat** is an exact match to a listed hash. **Dangerous indicators** and **Suspicious** can reflect permission, code, source, or inspection warnings. The current app can display **Safe** for recognized-store entries. Read the result details to see whether QALQON found no known hash match or could not verify the APK. Modified APKs can have different hashes, and static inspection has limited coverage.

### Installation Guard

With Android Accessibility permission, QALQON watches for text patterns in recognized installer screens and can request Android’s Back action to dismiss a prompt. It can intervene on an ordinary sideload prompt, even when the APK is legitimate. The guard uses the installer interface and may miss prompts on some Android versions or devices. The on-device APK scanner runs separately.

### Website Protection

With Android VPN permission, QALQON creates a per-app, DNS-only VPN. It handles supported apps’ system UDP DNS queries and sends permitted, uncached requests to Quad9 over DNS-over-TLS, with DNS-over-HTTPS as a fallback. Ordinary website and messaging traffic uses the device’s regular connection.

- **Supported apps** include browsers such as Chrome, Firefox, Samsung Internet, Edge, Brave, and Opera, as well as Telegram clients, WhatsApp, Signal, Messenger, Viber, and Discord. A downloaded package list can add more apps. VPN membership is based on an exact Android package ID.
- **Only approved websites** checks intercepted requests against the downloaded domain list when available. A listed domain includes its subdomains. Built-in service-domain exceptions help messaging apps connect and also apply to browser DNS in the tunnel.
- **Coverage** depends on the app using DNS that reaches the tunnel. Custom encrypted DNS, cached addresses, and direct IP connections can fall outside filtering. Android’s Strict Private DNS mode stops the QALQON tunnel. With no saved website list, local approved-site enforcement is inactive and Quad9 filtering can still operate if reachable.

See [BrowserPackages.kt](app/src/main/java/uz/qalqon/app/vpn/BrowserPackages.kt), [MessengerPackages.kt](app/src/main/java/uz/qalqon/app/vpn/MessengerPackages.kt), and [OfficialAppStores.kt](app/src/main/java/uz/qalqon/app/security/OfficialAppStores.kt) for the built-in catalogs.

## Network connections and privacy

| Service | What QALQON sends or downloads |
| --- | --- |
| **GitHub Raw / ThreatDB** | Downloads metadata and compressed APK hash lists. Requests include normal connection metadata, an app User-Agent, and a cached ETag when available. |
| **GitHub Raw / Allowlist** | Downloads text lists of website domains and Android package IDs. QALQON uses the package list locally to select which installed apps enter the DNS tunnel. |
| **Quad9** | Receives permitted DNS queries from supported apps when Website Protection is on, plus a DNS reachability probe during startup. |
| **VirusTotal** | When you confirm **Check online**, QALQON opens a VirusTotal report URL in your browser. The URL contains the selected APK’s SHA-256 fingerprint. QALQON uses no VirusTotal API and uploads no APK. |
| **MalwareBazaar** | The publisher’s ThreatDB workflow uses MalwareBazaar as an upstream source. The Android app makes no direct MalwareBazaar requests. |

Normal scans send no APK files, app inventory, or scan results to remote services. History, fingerprints, and rotating diagnostics are kept on-device. Diagnostics can include package IDs and some denied domain names. The app requires no QALQON account.

See the [privacy policy draft](PRIVACY_POLICY_DRAFT.md), [Quad9 privacy policy](https://quad9.net/privacy/policy/), and [MalwareBazaar](https://bazaar.abuse.ch/).

## How threat data is updated

QALQON uses two independent sources of APK hashes.

- **Built-in database.** The bundled [APK hash file](app/src/main/assets/apk_hashes.txt) ships inside the app and is available on first launch without internet. QALQON 1.0 contains 124,685 unique hashes. This file can change with a new app build.
- **Downloaded database.** QALQON checks the publisher’s ThreatDB metadata over HTTPS, then can apply a compatible delta or retrieve a full compressed snapshot. It checks the advertised sizes and SHA-256 checksums before replacing the downloaded database. The bundled file remains available either way.

WorkManager schedules automatic checks roughly every 24 hours after initial setup when enabled. The project’s publishing workflow runs separately, so a successful check can find the same data. You can also check manually with **Update now**. The website and app lists have separate download and validation logic.

**QALQON 1.0 release note:** On 22 November 2026, the live metadata was reachable but the advertised snapshot and deltas returned HTTP 404. Offline scanning and the two allowlist downloads are independent of that outage. The clean-install full-snapshot path in this source also needs its staging directory created before writing the download.

See [QALQON-ThreatDB](https://github.com/aex-3-14/QALQON-ThreatDB) and [QALQON-Allowlist](https://github.com/aex-3-14/QALQON-Allowlist).

## Under the hood

| Component | Responsibility |
| --- | --- |
| [`security/`](app/src/main/java/uz/qalqon/app/security/) | APK hashing and static inspection, verdicts, scan workers, installer monitoring, and notifications |
| [`vpn/`](app/src/main/java/uz/qalqon/app/vpn/) | Per-app routing, DNS packet handling, local domain rules, response caching, and encrypted Quad9 transport |
| [`updates/`](app/src/main/java/uz/qalqon/app/updates/) | Threat metadata, delta staging, hash validation, local SQLite lookup index, and allowlist downloads |
| [`core/`](app/src/main/java/uz/qalqon/app/core/) | Preferences, scan history, diagnostics, and localized policy text |
| [`ui/`](app/src/main/java/uz/qalqon/app/ui/) | Dashboard, scan controls and results, permission setup, and settings |

## Build from source

Open the repository in Android Studio. This source uses JDK 17, Android SDK 35 to compile, and Gradle 8.9. Use a verified Gradle 8.9 installation to build a debug APK. The source archive omits the wrapper JAR, so its wrapper scripts require a restored, verified JAR.

```sh
gradle :app:assembleDebug
```

The resulting debug APK is under `app/build/outputs/apk/debug/`. Test the installer guard, storage access, and DNS filtering on physical Android devices because those features depend on Android permissions and device behavior.

## License and data attribution

QALQON source is distributed under the [PolyForm Noncommercial License 1.0.0](LICENSE.md).

- **Bundled hashes** are derived from Arvind Prasad and Sumit Kumar’s *Droidware Android Malware Dataset*, version 2, Cybersecurity Lab, GLA University. QALQON distributes a subset in SHA-256 text form. [Dataset and license](https://data.mendeley.com/datasets/x7sb4g2kkv/2) · [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)
- **Downloaded threat hashes** are published through QALQON-ThreatDB using MalwareBazaar metadata. [ThreatDB](https://github.com/aex-3-14/QALQON-ThreatDB) · [MalwareBazaar Community API](https://bazaar.abuse.ch/api/)
