# NOUS OS

> **Rozumny system operacyjny dla telefonu, z wpisaną integralnością.**
> System operacyjny, który nie zdradza swojego właściciela — ale też nie
> zdradza nawet samego siebie.

[![Status](https://img.shields.io/badge/status-prototype-yellow)](#status)
[![Checkpoints](https://img.shields.io/badge/checkpoints-27%20green-green)](CHANGELOG.md)
[![Showcase](https://img.shields.io/badge/repo-showcase%20(no%20source)-lightgrey)](#kod-źródłowy)
[![License](https://img.shields.io/badge/license-all%20rights%20reserved-red)](COPYRIGHT.md)
[![Language](https://img.shields.io/badge/code-C11-orange)](#stos-technologiczny)
[![Target](https://img.shields.io/badge/target-ARM64%20%2F%20Fairphone%20%2F%20OnePlus%206-purple)](ROADMAP.md)

🇵🇱 polska wersja (główna) · [🇬🇧 English](README.en.md)

> **To repozytorium to publiczna prezentacja projektu.** Zawiera dokumentację,
> specyfikacje i zrzuty ekranu. **Kod źródłowy pozostaje prywatny** — patrz
> [Kod źródłowy](#kod-źródłowy).

---

## Co to jest

**NOUS OS** to mój autorski projekt systemu operacyjnego dla telefonu,
zaprojektowany od podstaw z dwoma niezwykłymi założeniami:

1. **System jest agentem moralnym** — ma wpisane zasady etyczne, które
   egzekwuje także wobec swojego właściciela (Lista Godności w TEE ROM,
   harm_score thresholds, fail-closed Ethics Gate).
2. **System wspiera osoby z ADHD** bez szpiegowania ich — wszystkie
   funkcje "wellbeing" są **izolowane od sieci na poziomie kompilacji**
   (próba dodania `<sys/socket.h>` do modułu wellbeing = błąd kompilacji,
   nie ostrzeżenie runtime).

NOUS to greckie słowo νοῦς — **umysł, intelekt, rozum**. Tłumaczy się
to wprost: **system rozumny**. Filozofia projektu opisana w
[MANIFESTO.md](MANIFESTO.md).

---

## Co działa dziś

**27 zielonych checkpointów** w QEMU aarch64 — pełna lista w
[CHANGELOG.md](CHANGELOG.md). Najważniejsze:

### Pełny stos systemu
- **Boot pipeline** — Docker reproducible build, cross-compile do ARM64,
  Alpine Linux mini-rootfs jako squashfs
- **Display server** — 1-bit Atkinson dithering, DRM/KMS bezpośrednio,
  polska typografia 8×8 pikseli, geometryczny font wektorowy dla UI
- **Touch shell** — launcher z 6 aplikacjami, nawigacja, motyw
  jasny/ciemny, dialer 1-9*0#, kontakty, wiadomości, ustawienia
- **Input** — klawiatura + dotyk (virtio-tablet), modality-agnostic
  dispatch
- **Intent Bus** — 9-stopniowa walidacja każdej decyzji (parsing,
  filtr prywatności, bramka etyczna z hardcap'em 50ms, audit ledger,
  klasyfikacja, dispatcher)
- **Audit ledger** — write-only Merkle chain, każda decyzja
  zhashowana, modyfikacja = abort

### Funkcje dla ADHD (Chronos)
- **5-poziomowa eskalacja przypomnień** — od cichego SILENT_ICON przez
  TOAST w rogu, BANNER na środku, po pełnoekranowy ALERT (inverted
  kolory) z badge dla SOUND / VOICE / VIBRATION
- **Time Buffer warning** — system zauważa gdy twoje oszacowanie
  ("kardiolog za 20 min") nie pasuje do historii ("zwykle zajmuje 45")
- **HEALTH cancel → Human Review** — odwołanie wizyty u kardiologa
  zawsze przechodzi przez double-check z powodem
- **Missed reminders ring** — przegapione przypomnienia trafiają na
  retrospektywną listę (zaagregowaną per task, decay'owaną w czasie)
- **Subsydiarność** — system najpierw sugeruje, nigdy nie wymusza

### Wellbeing module
- **Monitor stresu** — śledzi jak często system blokuje twoje
  intencje (denial pressure), nie szpieguje twojej aktywności
- **Stress decay** — po godzinie bez nowych blokad, "pamięć stresu"
  spada do ~5%. System nie trzyma urazy.
- **sec_ctx filter** — rozmowy przychodzące i SMS od innych ludzi
  **NIE liczą się jako twój stres** (incoming pressure ≠ user pressure)
- **§7 Kodeks** — moduł wellbeing kompiluje się z `-DNOUS_NO_NETWORK=1`,
  tripwire na `AF_INET`. Próba dodania networking'u = błąd kompilacji.

### Telefonia (mockowana w QEMU, prawdziwa po pmOS bridge)
- `mock_qrtr` symuluje QMI frame z modemu (incoming call, SMS)
- RIL daemon parsuje QMI z hostile-black-box length validation
- Seccomp-bpf filter na 12 syscalli (rest = SIGKILL)
- **CP22**: bridge ModemManager→NOUS dla pmOS na OnePlus 6

### Test infrastructure
- **Autonomous harness** (CP21): build → boot → wstrzykiwanie poleceń
  serialem → screenshot capture → asercje na logach
- **19/19 scenariuszy zielonych** w pełnym test suite

---

## Galeria (zrzuty z QEMU)

| | | |
|---|---|---|
| ![Home](docs/screenshots/shell_home.png) | ![Home dark](docs/screenshots/shell_home_dark.png) | ![Dialer](docs/screenshots/shell_dialer.png) |
| Launcher (motyw jasny) | Launcher (motyw ciemny) | Dialer |
| ![Contacts](docs/screenshots/shell_contacts.png) | ![Settings](docs/screenshots/shell_settings.png) | ![Keyboard](docs/screenshots/shell_keyboard.png) |
| Kontakty | Ustawienia | Klawiatura ekranowa |
| ![Incoming call](docs/screenshots/card_incoming_call.png) | ![Time buffer](docs/screenshots/card_time_buffer.png) | ![Wellbeing](docs/screenshots/card_wellbeing_alert.png) |
| Połączenie przychodzące | Time Buffer warning | Wellbeing alert |
| ![Missed list](docs/screenshots/card_missed_list.png) | ![Idle](docs/screenshots/idle_screen.png) | ![Guardian](docs/screenshots/guardian_nudge.png) |
| Lista przegapionych | Ekran spoczynku | Strażnik uwagi |

Wszystkie zrzuty z QEMU 540×960 (portrait, jak na prawdziwym telefonie).
Czysty 1-bit Atkinson, polskie znaki w typografii 8×8 + geometryczny
font wektorowy w shell UI.

---

## Architektura

System składa się z modułów w trzech warstwach (ring model):

```
┌─────────────────────────────────────────────────────────────┐
│ Ring-3 (userspace processes)                                 │
│ ├─ nous_display_server  ─ rendering, card queue, dispatch    │
│ ├─ nous_input_daemon    ─ keyboard + touch multiplex         │
│ └─ mock_chronos, mock_voice, mock_wellbeing, mock_qrtr       │
├─────────────────────────────────────────────────────────────┤
│ Ring-2 (Intent Bus daemon)                                   │
│ └─ nous_ril_daemon ─ modem hostile-black-box + intent_submit │
│                       ↓ 9 steps                              │
│                       parse → PFS → offline → ethics →       │
│                       classify → ledger → wellbeing →        │
│                       HAL dispatch                           │
├─────────────────────────────────────────────────────────────┤
│ Ring-1 (NeuroKernel, planned)                                │
│ └─ Reasoning, grounding, autonomy score                      │
├─────────────────────────────────────────────────────────────┤
│ Ring-0 (kernel)                                              │
│ └─ Linux + virtio drivers                                    │
└─────────────────────────────────────────────────────────────┘

         ┌─ TEE ROM (sealed) ─┐
         │ Ethics Gate        │  ← hard 50ms deadline, fail-closed
         │ DIGNITY list       │  ← read-only after boot
         │ Pubkeys (PCR 7)    │  ← measured by ACM (Boot Guard)
         └────────────────────┘
```

Specyfikacje projektowe (IntentBus, Żelazny Kodeks Etyczny, ARM64):
folder [`docs/specs/`](docs/specs/).

---

## Stos technologiczny

- **Język**: C11 z `-Wall -Wextra -Wpedantic -Wconversion -Wshadow
  -Wstrict-prototypes -Wmissing-prototypes -fstack-protector-strong`
- **Build**: GNU Make + Docker (reproducible build, pinned Debian
  bookworm, deterministic SOURCE_DATE_EPOCH)
- **Cross-compile**: aarch64-linux-gnu-gcc 12.2.0
- **Wirtualizacja**: QEMU `-M virt -cpu cortex-a57`, virtio-gpu-pci
  + virtio-keyboard-pci + virtio-tablet-pci
- **Dystrybucja**: Alpine Linux 3.19.1 mini-rootfs jako squashfs
- **Display**: DRM/KMS bezpośrednio (bez libdrm), Atkinson dither,
  AF_UNIX SOCK_DGRAM IPC z CRC32 frame validation
- **Audit**: SHA-256 chain (placeholder, docelowo Ed25519 via libsodium)
- **Testing**: Python 3 harness z QMP touch injection + screenshot
  diff (CP21)
- **Docelowy hardware**: Fairphone 4/5 lub OnePlus 6 (pmOS support)

---

## Plany

Pełny plan w [ROADMAP.md](ROADMAP.md) — trzy horyzonty: demo
architektonicznie kompletne (tygodnie), bring-up na fizycznym telefonie
(miesiące), wersja produkcyjna (lata). Katalog aplikacji shell w
[APPS.md](APPS.md), aktualny stan operacyjny w [STATUS.md](STATUS.md).

---

## Kod źródłowy

Kod źródłowy NOUS OS (C11, ~180 plików) jest **prywatny**. To repozytorium
służy prezentacji projektu — architektury, decyzji projektowych i UI.

W sprawie dostępu do kodu, współpracy lub rozmowy o projekcie:
**mielko@onet.pl**.

---

## Dokumentacja

| Plik | Co zawiera |
|---|---|
| [MANIFESTO.md](MANIFESTO.md) | Filozofia projektu — czemu w ogóle robimy NOUS, nie kolejnego Androida. **Zacznij tutaj.** |
| [CHANGELOG.md](CHANGELOG.md) | Historia 27 checkpointów, każdy jednolinijkowo. |
| [ROADMAP.md](ROADMAP.md) | Co dziś, co potem, trzy horyzonty. |
| [APPS.md](APPS.md) | Plan aplikacji shell — co już jest, co dochodzi. |
| [STATUS.md](STATUS.md) | Aktualny stan operacyjny — co kliknięcie robi co. |
| [docs/specs/](docs/specs/) | Specyfikacje: IntentBus, Żelazny Kodeks Etyczny, NOUS na ARM64. |

---

## Status

**Prototype / research artifact**. Nie jest to codzienny telefon —
nie zadzwonisz, nie odbierzesz SMS, nic nie przeżyje rebootu (tmpfs).
To kompletny proof-of-concept architektury.

---

## Filozofia w jednym zdaniu

> "Telefon, który **nie zadzwoni** pod numer z listy DIGNITY,
> **nawet gdy ja sam tego zażądam w słabości**."

To nie jest pre-marketing. To jest architektura. Patrz
[MANIFESTO.md](MANIFESTO.md).

---

## Licencja

**© 2025–2026 Artur Mielko. Wszelkie prawa zastrzeżone.** Patrz
[COPYRIGHT.md](COPYRIGHT.md).

To repozytorium udostępnia dokumentację i materiały wizualne do wglądu.
Kod źródłowy nie jest publikowany, a treści nie wolno redystrybuować bez
zgody autora.

---

## Autor

**Artur Mielko** — Polska, 2025–2026.

Projekt prowadzony po godzinach jako research artifact i przyszły
codzienny telefon autora. Otwarty na rozmowy o systems programming,
embedded Linux, ADHD-friendly UX, i o tym czy świętość daje się
skompilować w C11.

Kontakt: **mielko@onet.pl** · GitHub: [@kaszanisko](https://github.com/kaszanisko)
