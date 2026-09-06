# Browser Fingerprint Surfaces

A working map of the surfaces a Chromium build leaks identity through, organised the
way a patch set actually has to be organised.

These are notes from building and maintaining a custom Chromium for browser-profile
isolation. The repository documents **what leaks and how it is probed** — it is a
reference for people working on anti-fraud detection, privacy engineering, and
browser automation. It contains no patches.

---

## Why layers, not a checklist

Most writing on this topic is a flat list of "things to spoof". That framing produces
brittle builds, because the surfaces are not independent — they constrain each other.

A patch set that survives Chromium rebases needs to be ordered by *how deep in the
browser the change sits*, so that a rebase conflict in one layer does not cascade:

| Layer | Concern | Rebase risk |
| --- | --- | --- |
| 0 · De-telemetry | Strip upstream phone-home and histogram reporting | Low |
| 1 · Branding | Product name, icons, version strings | Low |
| 2 · Module | Config plumbing, IPC surface, crypto, seeding | Medium |
| 3 · Automation | Remove automation and DevTools tells | High |
| 3 · Fingerprint | JS-visible surfaces | High |
| 3 · Network | TLS and HTTP/2 stack behaviour | High |

Layers 0–2 are infrastructure. Layer 3 is where detection actually happens, and it
splits three ways because the three sub-layers are probed by completely different
techniques.

---

## The coherence problem

**This is the part most implementations get wrong.**

Every surface below must agree with every other surface. A profile claiming Windows
in its User-Agent, exposing macOS-only fonts, and negotiating a TLS handshake whose
cipher ordering matches Linux Chrome is *more* identifiable than a profile that
spoofs nothing at all — because inconsistency is itself a rare, high-entropy signal.

Practical consequence: a fingerprint layer needs a **coherence validator** that runs
over the whole generated profile and rejects impossible combinations, not a per-surface
patch that each independently returns a plausible-looking value.

The three combinations that most often break coherence in practice:

- `navigator.platform` / UA-CH `platform` / font set / WebGL renderer string
- `Intl.DateTimeFormat().resolvedOptions().timeZone` / `Date.getTimezoneOffset()` /
  geolocation / the IP's actual timezone
- `navigator.languages` / the `Accept-Language` request header

A second-order rule: **noise must be deterministic per profile.** Canvas noise that is
re-randomised on every read is trivially detected by hashing the same canvas twice in
one session. Seed the noise from a per-profile key and derive it deterministically.

---

## Layer 3a — Automation tells

Detection here is boolean: one hit and the session is flagged. No entropy analysis needed.

| Surface | What leaks | How it is probed |
| --- | --- | --- |
| `navigator.webdriver` | WebDriver-controlled session | Direct property read |
| Automation infobar | "Chrome is being controlled by automated software" | Window dimension deltas |
| `chrome.runtime` id | Extension-host marker present in automation builds | Property presence |
| CDP `Runtime.enable` | Attaching the debugger mutates JS error stack behaviour | Error stack inspection, console timing |
| Headless UA | `HeadlessChrome` token in the User-Agent string | String match |
| DevTools endpoint | Open debugging port reachable from page context | Local fetch probe |
| Outer dimensions | `outerWidth`/`outerHeight` of 0 in headless | Arithmetic against `innerWidth` |

---

## Layer 3b — Fingerprint surfaces

Detection here is statistical: each surface contributes entropy, and the combination
identifies. Individually harmless, jointly unique.

| Surface | What leaks | How it is probed |
| --- | --- | --- |
| Canvas | GPU and font rasterisation differences | Hash of `toDataURL` / `getImageData` |
| OffscreenCanvas | The same raster path, via a second API | `convertToBlob` hash — often forgotten |
| AudioContext | DSP pipeline float rounding | `OfflineAudioContext` → oscillator → compressor → hash |
| ClientRects | Sub-pixel layout geometry | `getBoundingClientRect` values |
| WebGL | GPU vendor and renderer strings | `WEBGL_debug_renderer_info` unmasked params |
| WebGPU | Adapter info — a newer parallel to WebGL | `requestAdapter().info` |
| Fonts | Installed font set, strongly implies OS | `measureText` width probing, `FontFace` checks |
| Timezone | Host timezone | `Intl.DateTimeFormat` + `getTimezoneOffset` |
| Hardware | CPU core count, RAM class | `hardwareConcurrency`, `deviceMemory` |
| Screen | Resolution, available area, colour depth | `screen.*` properties |
| UA-CH | High-entropy client hints | `userAgentData.getHighEntropyValues()` |
| Languages | Locale preference ordering | `navigator.languages` vs `Accept-Language` |
| Battery | Charge level and timings, a slow-moving quasi-identifier | `getBattery()` |

Timezone deserves a note: it is enforced in ICU, not in JavaScript. Patching only the
JS-visible getter leaves `Intl` disagreeing with `Date`, which is a clean detection.

---

## Layer 3c — Network surfaces

Invisible to JavaScript entirely. This layer is where most anti-detect work stops, and
therefore where detection increasingly starts — a server can fingerprint the connection
before a single byte of page script runs.

| Surface | What leaks | Known as |
| --- | --- | --- |
| TLS cipher suites | Offered list and its ordering | JA3 / JA4 |
| TLS supported groups | Curve list and ordering | JA3 / JA4 |
| TLS extensions | Which extensions, in what order, GREASE placement | JA3 / JA4 |
| ALPN | Protocol offer list | JA3 / JA4 |
| HTTP/2 SETTINGS | Frame values and their send order | Akamai H2 fingerprint |
| HTTP/2 pseudo-headers | `:method :authority :scheme :path` ordering | Akamai H2 fingerprint |
| WebRTC ICE | Local network addresses via STUN candidates | Direct IP leak |

The coherence rule applies here too, and bites hardest: a JA4 hash saying Chrome 131 on
Windows, paired with a JS layer saying Chrome 120 on macOS, is a contradiction no real
client produces.

---

## Reading list

- [BrowserLeaks](https://browserleaks.com/) — live probes for most Layer 3b surfaces
- [Am I Unique](https://amiunique.org/) — entropy of a given combination
- [JA4+](https://github.com/FoxIO-LLC/ja4) — the successor scheme to JA3
- [Chromium source](https://source.chromium.org/) — the only authority on what a surface actually does

---

## Scope

This is a defensive and research reference. It maps surfaces and describes how they are
detected; it does not distribute patches, evasion code, or a build.

