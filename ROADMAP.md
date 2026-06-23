# NOUS OS — Roadmap

This document describes where NOUS OS **is today** and what it
**aims to become**. It is honest about the gap between the two.

The corresponding Polish summary lives at the bottom (Po polsku).

---

## Where we are today (June 2025)

NOUS OS is a **research-quality architectural prototype** running in
QEMU aarch64 emulation under WSL2 / Linux. **27 green checkpoints**
documented in [CLAUDE.md](CLAUDE.md), each with empirical tests and
locked architectural invariants.

### What's actually working

**Boot pipeline**
- Reproducible Docker build (Debian bookworm pinned, deterministic timestamps)
- Cross-compile to ARM64 (aarch64-linux-gnu-gcc)
- Alpine Linux mini-rootfs (3.19.1) packed as squashfs
- Custom `/sbin/init` mounts tmpfs, starts daemons, exec's `/bin/sh`
- Network restricted=on in QEMU (offline-first verified)

**Display stack**
- virtio-gpu-pci passes through to host (SDL backend on WSLg)
- DRM raw ioctl path (no libdrm dependency)
- 1-bit Atkinson dithering with persistent error-diffusion context
- 8×8 bitmap font in `.rodata` (1024 bytes, signature aesthetic)
- Polish diacritics via 18 extra glyphs (144 bytes) + UTF-8 2-byte decoder
- Vector geometric font for shell UI (CP26) with light/dark themes
- Priority queue (8 slots) for notification cards with FIFO tiebreaker
- Multi-line list rendering for missed-reminders card

**Input stack**
- virtio-keyboard-pci + virtio-tablet-pci (touch / mouse) multiplexed via poll(2)
- Closed action catalog (5 actions, 9 input codes including BTN_TOUCH)
- `NOUS_CTX_USER_LOCAL` vs `NOUS_CTX_SYSTEM` distinction
- Modality-agnostic dispatch (KEY_D and click both emit DISMISS)
- Touch shell with hit-tested tile launcher (CP25)

**Telephony stack (mocked)**
- `mock_qrtr` injects QMI frames over AF_UNIX
- RIL daemon parses QMI with hostile-black-box length validation
- Seccomp-bpf filter on 12 syscalls (rest = SIGKILL)
- Service classification by `msg_id` (VOICE / WMS / NAS)
- ModemManager bridge prototyped (CP22) for PostmarketOS integration

**Intent Bus (ring-2)**
- 9-step validation flow: parse → PFS → offline → ethics → classify
  → ledger → wellbeing → HAL dispatch
- Ethics Gate with hard 50ms deadline (fail-closed)
- Decision Ledger as write-only Merkle chain (SHA-256 placeholder)
- 4200–4500 byte packet budget enforced by `_Static_assert`

**Chronos (ADHD support)**
- 5-tier escalation: SILENT → TOAST → BANNER → VIBRATION → SOUND → VOICE
- Time Buffer warning: user-estimate vs historic-duration comparison
- HEALTH cancel → Human Review Queue (two-stage with reason)
- Persistent multi-task schedule (`/tmp/nous-schedule.txt`)
- Missed reminders ring buffer with per-task aggregation
- Per-task `last_level` anti-spam
- Race-fixed scheduler send order (priority DESCENDING)

**Wellbeing module**
- Tracks "denial pressure" as EMA `stress_index`
- 4-tier escalation (SILENT / MINI / BANNER / ALERT)
- Decay over time (0.95 per minute multiplier)
- `sec_ctx` filter: incoming calls / SMS (SYSTEM ctx) do NOT count
  toward stress — wellbeing tracks pressure FROM user, not ON them
- Cross-process snapshot via `/tmp/nous-wellbeing.txt`
- **§7 Kodeks**: compile-time isolation from network headers
  (build fails if `<sys/socket.h>` leaks in)

**Voice pipeline (stubbed)**
- 8 stages with `static` ASR (compile-time guarantee that
  third-party voices cannot reach transcription)
- Polish keyword parser as ASR placeholder
- Secure wipe of audio buffer on every exit path (3-pass with
  memory barriers per §13 Kodeks)
- Voice → Scheduler wire-up via fork+execlp (no shell injection)

**Touch shell (CP25-CP27)**
- 6-app launcher with tile hit-testing
- Navigation framework with back-chevron
- Dialer (1-9, *, 0, #)
- Contacts list with tap-to-call
- Messages with Vosk dictation demo
- Settings (theme, DND, Dignity Override)
- Idle screen with clock
- Light/dark themes (Atkinson stays zero-shimmer on pure extremes)

**Testing**
- Autonomous test harness (CP21): build → boot → command-injection
  via serial → screenshot capture → log assertions
- 19/19 scenarios green
- Screenshot review (agent looks at the visual output)

**Documentation**
- [CLAUDE.md](CLAUDE.md) — 135+ architectural invariants
- [MANIFESTO.md](MANIFESTO.md) — philosophical positioning
- [APPS.md](APPS.md) — application catalog
- [PORTFOLIO.md](PORTFOLIO.md) — CV / interview talking points

---

## What is NOT working

Honestly, so this is not misrepresented:

- **Real cellular**: no real modem driver; QEMU has no radio. Calls
  and SMS are mocked. The ModemManager bridge (CP22) is the planned
  integration path for real hardware.
- **Real ASR**: voice pipeline uses keyword parser, not Vosk/Whisper
  inference. Vosk integration prototyped in messages app (CP27).
- **Real audio**: no audio device wired. VIBRATION / SOUND / VOICE
  levels render as text badges, not actual sound.
- **Persistence across reboots**: `/tmp` is tmpfs. Tasks, missed
  history, wellbeing state — all lost on QEMU reboot.
- **Real bootloader**: build scripts exist (`build/scripts/sign_image.py`,
  `build/scripts/build_image.sh`) but not integrated end-to-end with
  QEMU yet. EFI / A/B partitions / Ed25519 verify are designed but
  not bring-up'd.
- **Real TEE**: Ethics Gate is a software stub returning mock
  verdicts. Real implementation requires TEE attestation +
  ROM-loaded harm classifier.
- **Real SLM inference**: model wrapper API exists but no actual
  model loaded. GBNF grammar is implemented; integration with a real
  on-device LLM (GGUF format) is future work.
- **App store / sandboxing**: no third-party apps, no bwrap setup
  for WPE WebKit.
- **Multi-language UI**: Polish + English mixed for now (UI Polish,
  audit/logs English). No localization framework.

---

## Three horizons

### Horizon 1 — Architecturally complete demo

**Goal**: prove the architecture lives end-to-end in QEMU. Suitable
for academic paper, conference talk, OSS release.

**Estimated effort**: 5-10 checkpoints, 2-3 weeks full-time.

**Concrete work**:
- **CP28+**: audio cue for ALERT (virtio-sound + ALSA + bell.wav)
- **CP29**: ledger missed events (audit/display crossing)
- **CP30**: real-time wellbeing daemon (periodic decay tick)
- **CP31**: snapshot test infrastructure with golden files in CI
- **CP32**: dignity override flow demo (vocal phrase + scope check)
- **CP33**: enriched MISSED_LIST card with per-task last-seen timestamp
- **CP34**: full Polish localization framework (extract strings,
  fallback to English)

**Outcome**: end-to-end demo showcasing the full architecture as
research artifact. Not a phone, but a complete proof-of-concept.

### Horizon 2 — Fairphone / OnePlus 6 bring-up

**Goal**: actual phone you can carry. Mocks replaced with real
hardware drivers via PostmarketOS plumbing.

**Estimated effort**: 15-25 checkpoints, 2-3 months full-time.

**Concrete work**:
- **CP35**: PostmarketOS base + NOUS as overlay (use their modem,
  audio, kernel drivers — replace their Phosh/Plasma Mobile shell
  with NOUS shell)
- **CP36**: Real ASR via Vosk on-device (~50MB model)
- **CP37**: WPE WebKit linked in (`build/Dockerfile.qemu` already
  has libegl-mesa-dev etc.)
- **CP38**: Persist partition (LUKS2 mount, /var/lib/nous/)
- **CP39**: Real TEE → libsodium Ed25519 + ROM keys
- **CP40**: Bootloader integration (EFI + signed manifest + A/B)
- **CP41**: Battery / PMIC integration
- **CP42**: Contacts database (CardDAV optional)
- **CP43**: SMS storage with privacy guarantees (encrypted at rest)
- **CP44**: WiFi via NetworkManager
- **CP45**: Bluetooth via BlueZ (subset: headset profile only)

**Outcome**: working phone, simplified, demonstrably runs as your
daily device. Some apps will be missing or rough, but **you can
call and text**.

### Horizon 3 — Production-shippable

**Goal**: phone you could give a non-technical user. Robust enough
for daily use across diverse scenarios.

**Estimated effort**: 50-100 checkpoints, 1-2 years.

**Concrete work** (representative):
- Full seccomp/SELinux hardening of every daemon
- OTA updates with rollback recovery
- Crash recovery / reboot persistence
- Multi-language UI (English, German, Spanish, French, Czech at
  minimum)
- Apps SDK + sandbox per-app
- E-ink overlay driver (Fairphone-specific)
- Per-user wellbeing calibration
- CalDAV sync
- Camera, microphone hardware bring-up
- Kill switch hardware integration (4× one-way locks)
- Audit ledger viewable through UI
- Battery curve calibration
- Repair / unbrick procedures
- Internationalization, accessibility (larger font, screen reader)
- User documentation
- Manufacturing test suite
- Long-haul reliability tests
- **Regulatory**: FCC/CE/PTCRB certification, GDPR audit, app store
  policy, supply chain security

**Outcome**: phone with retail-quality polish. Not in this project's
realistic scope as a solo effort — would require team + funding.

---

## Alternative paths to "I actually use it"

Building a production phone is a multi-year effort. For a solo
developer with limited time, three pragmatic shortcuts exist:

### Path A — PostmarketOS + NOUS overlay (3-6 months)

Take PostmarketOS as base (working modem, kernel, audio on
Fairphone 4/5 or OnePlus 6). Replace their shell with NOUS shell.

**Result**: real phone, real calls, real SMS, NOUS UX (Chronos,
Wellbeing, ascetic cards). Compromise: not everything is "yours"
(kernel, modem driver from PostmarketOS).

### Path B — Android companion (1-2 months)

Keep Android phone. Write Android app that bridges incoming
calls/SMS to NOUS server running on home Linux box (laptop, RPi).
NOUS provides ADHD UX, ethics gate, wellbeing tracking.

**Result**: works with hardware you already have. Privacy preserved
(server is at home, not in cloud).

### Path C — Chronos as standalone Android / desktop app (2-4 weeks)

Skip the OS layer entirely. Port Chronos + Wellbeing as standalone
Android app (Kotlin + JNI to `libnous.so`) or Linux desktop app
(GTK4 directly on top of existing `libnous.a`).

**Result**: useful daily ADHD assistant. 90% of the architectural
value lands in your pocket in a month.

---

## Po polsku — krótkie streszczenie

### Gdzie jesteśmy

NOUS OS to **prototyp architektoniczny działający w QEMU**, 27
zielonych checkpointów udokumentowanych w CLAUDE.md.

**Działa**: boot pipeline, display server (1-bit Atkinson +
polska typografia 8×8), klawiatura + dotyk, Chronos (5-poziomowa
eskalacja, missed reminders, persistent schedule), Wellbeing (decay,
sec_ctx filter, persist snapshot), voice stub, mockowana telefonia,
touch shell (launcher, dialer, kontakty, wiadomości, ustawienia),
autonomous test harness, ModemManager bridge dla pmOS.

**Nie działa**: prawdziwy modem (QEMU nie ma radia), real ASR (Vosk
prototyp w wiadomościach), real audio, persystencja cross-reboot,
real bootloader, real TEE, real SLM inference, app store.

### Trzy horyzonty

- **Horyzont 1** (2-3 tygodnie, 5-10 CP): architecturally complete
  demo — audio, dignity flow, golden tests, localization framework
- **Horyzont 2** (2-3 miesiące, 15-25 CP): Fairphone / OnePlus 6
  bring-up — pmOS jako baza, NOUS jako overlay
- **Horyzont 3** (1-2 lata): production-shippable — pełne
  utwardzenie, OTA, certyfikacja, multi-language, app store

### Skróty do codziennego użytku

- **Droga A** (3-6 mies): pmOS + NOUS overlay na fizycznym telefonie
- **Droga B** (1-2 mies): Android companion + NOUS server na RPi
  w domu
- **Droga C** (2-4 tyg): Chronos + Wellbeing jako standalone Android
  / Linux desktop app

Najszybsza droga do "rzeczywiście używam codziennie": **Droga C**.

---

## How to contribute

See [CONTRIBUTING.md](CONTRIBUTING.md). Short version: read
[MANIFESTO.md](MANIFESTO.md) first; if you disagree with the
philosophical positioning, fork rather than try to soften the
invariants. The Iron Code is what makes NOUS NOUS.

---

*Last updated: June 2025*
