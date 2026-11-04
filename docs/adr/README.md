# Architecture Decision Records: HandiExceller (Br'eye, Marble)

An Architecture Decision Record (ADR) is a short note that captures one important technical decision: the context at the time, what was decided, the alternatives, and the consequences.

These ADRs cover the main architecture decisions of HandiExceller's products, designed by Melchior Vila, the CTO (May 2021 – October 2026):

- **Br'eye**: a braille-learning device (Raspberry Pi, and Raspberry Pi CM4 + RP2040 for v2) that reads tactile dominoes and answers with audio. I started from an intern's non-Python prototype and redesigned it entirely.
- **Marble**: a Windows application for braille and accessible computing, usable with NVDA and JAWS, which I started. It plays the same content with the same code as Br'eye.

The ADRs describe design decisions and why they work, not the implementation work around them.

| # | Title | Decision in one line |
|---|---|---|
| [0001](0001-creative-user-generated-content.md) | Creative: user-generated content | Teachers build their own lessons and games as synced JSON content, and learner activity is recorded from day one. |
| [0002](0002-multi-language-by-design.md) | Multi-language by design | No user-facing string in code: enum keys, CSV tables, DeepL and Azure TTS tooling, per-language audio submodules. |
| [0003](0003-observer-for-language-and-settings.md) | Observer for language and settings | A singleton Observable notifies every component when the language, a setting or the UI state changes. |
| [0004](0004-offline-licence-cache.md) | Offline licence cache | Machine-bound keys with a hash-signed local cache that works offline and rejects tampering and clock changes. |
| [0005](0005-brey-v2-graceful-shutdown.md) | Br'eye v2 graceful shutdown | The RP2040 owns power through a state machine and a UART handshake with cause codes. |
| [0006](0006-rp2040-firmware-update-from-cm4.md) | RP2040 firmware update from the CM4 | Firmware ships with the app and is flashed over SWD on version mismatch, with a blind-flash fallback. |
| [0007](0007-brey-updates-from-git-diff.md) | Br'eye updates from git diffs | Versions are commits; the server sends changed files over WebSocket into a staging folder, checked and applied only when complete. |
| [0008](0008-marble-self-update-delta-zips.md) | Marble self-update | File-hash manifests, server-built delta zips per version and language set, SHA-256 check and a separate updater. |

Format: lightweight [MADR](https://adr.github.io/madr/) (Context, Decision, Alternatives, Consequences, Validation, References). Paths in References are relative to the private repositories they come from.
