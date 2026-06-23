# Changelog

All notable changes to NOUS OS are documented here. Each entry references
a "checkpoint" (CP) — a green-tested architectural milestone with
invariants locked in `CLAUDE.md`.

The format is loosely based on [Keep a Changelog](https://keepachangelog.com/),
adapted for a research project where checkpoints are the unit of progress
rather than semver versions.

---

## [Unreleased]

Work in progress — see [ROADMAP.md](ROADMAP.md) for direction.

---

## [CP27] — Touch shell with full navigation framework

Interactive touch-driven phone shell. Dialer, contacts, messages,
settings — all reachable by finger taps in the QEMU 540×960 portrait
window.

- Navigation framework with screen stack + back-chevron
- Dialer with 1-9, *, 0, # keypad; number entry + delete
- Contacts list with tap-to-call
- Messages screen (Vosk dictation demo + send skeleton)
- Settings: theme toggle, Do Not Disturb, Dignity Override
- Reminders / Wellbeing hubs link to Chronos / Wellbeing cards
- Universal back-chevron "‹" in top-left of every screen

## [CP26] — Geometric vector font + light/dark themes

Custom vector font replacing 8×8 bitmap for shell UI (cards retain
8×8). Two themes selectable at runtime.

- Geometric vector font in `.rodata` (still tampering-detectable
  via binary hash)
- Light theme (default): pure white bg, pure black fg
- Dark theme: pure black bg, pure white fg
- Theme toggle by tapping clock on home screen
- Atkinson dither still converges to zero shimmer on pure extremes

## [CP25] — Interactive touch shell with tiles

Home launcher with 6 application tiles. Hit-testing wired to display
state machine.

- 6 application tiles on home screen
- Hit-testing per tile region
- Tap → screen transition
- Status bar with clock + battery + signal placeholders

## [CP23-CP24] — Idle screen + icon tier

Clock-only idle screen for the lock state. Geometric icon set.

- Idle screen with large clock (HH:MM)
- Date below clock
- Icon tier: 6 application icons designed in 8×8 grid expanded to
  64×64 vector
- `NOUS_IDLE=1` debug mode

## [CP22] — ModemManager → NOUS bridge

Bridge layer between PostmarketOS's ModemManager (real telephony stack)
and NOUS Intent Bus. Foundation for actual phone calls on physical
OnePlus 6 hardware.

- D-Bus client polling ModemManager for incoming calls/SMS
- Translation to NOUS_INTENT_COMM packets
- Pass-through to nous_intent_submit (full 9-step validation)
- Outgoing calls/SMS via ModemManager API
- OnePlus 6 hardware audit completed (modem chipset compatibility
  verified)

## [CP21] — Autonomous test harness

Headless QEMU + screenshot capture + log assertions. CI-style
verification without manual clicking.

- `tests/system/harness.py` orchestrates build → boot → test → screenshot
- Touch injection via QEMU QMP protocol
- Screenshot capture for visual regression
- Log assertion framework
- Agent reviews screenshots to verify visual correctness
- 19/19 scenarios green

## [CP20b] — Touch / pointer input (infrastructure)

Modality-agnostic input. Keyboard D and pointer click both emit
`NOUS_ACTION_DISMISS`. `virtio-tablet-pci` added to QEMU.

- Dual-fd `poll(2)` multiplex over `event0` (keyboard) + `event1`
  (pointer/touch)
- `fd_classify()` recognizes keyboard (KEY_A) vs pointer (BTN_LEFT
  or BTN_TOUCH)
- `KEY_MAP[]` extended with `BTN_LEFT` and `BTN_TOUCH` → `DISMISS`
- Soft-fail per slot: device disconnect doesn't kill daemon
- Banner: "click/tap (dismiss)" added to help

## [CP20a] — Cross-process wellbeing snapshot via persist file

`monitor/wellbeing.c` flushes `g_state` to `/tmp/nous-wellbeing.txt`
on every `update()` / `decay()`. `mock_wellbeing --snapshot` reads
the file — cross-process view of stress accumulated on the live bus.

- `wb_persist_state()` helper, single-line key=value format
- Called after stress_index recalculation
- `mock_wellbeing --snapshot` mode (read-only)
- Empirical confirmation: 5× incoming calls (SYSTEM ctx) do NOT
  create the file — filter cleanly cuts the side effect

## [CP19b] — Multi-line missed list rendering

`mock_chronos --missed` shows full list of missed reminders as
left-aligned column (up to ~5 entries visible), not just "latest"
with count.

- `nous_text_draw` supports `\n` (line break + cy advance)
- `render_missed_list()` dedicated layout in `display/notify.c`
- `cmd_missed` packs entries into line2 with `\n` separator
- Polish header "PRZEGAPIONE" (UI side stays Polish)
- Truncation hint in line3 ("+N more")

## [CP19a] — Wellbeing decay + sec_ctx filter

Two fundamental improvements:

1. **sec_ctx filter**: incoming calls / SMS (sec_ctx=SYSTEM) do
   NOT increment stress. Only user-initiated intents (USER_LOCAL,
   USER_REMOTE) pump counters. Wellbeing tracks pressure FROM the
   user, not pressure ON them.

2. **Decay over time**: `nous_wellbeing_decay(elapsed_seconds)`
   multiplies stress by 0.95 per minute. After 60min without new
   denies, stress relaxes to ~5% of peak. Wellbeing is forgetful
   by design.

## [CP18] — Wellbeing module surfaces system stress

First time `monitor/wellbeing.c` materializes its voice on screen.
`mock_wellbeing --simulate-denies N` triggers escalation gradient.

- `NOUS_NOTIFY_WELLBEING_NUDGE = 14` kind
- 4-tier escalation mapped from `stress_index`:
  SILENT / MINI / BANNER / ALERT
- §7 Kodeks invariant preserved: `wellbeing.o` compiled with
  `-DNOUS_NO_NETWORK=1` + tripwire on `AF_INET` / `SOCK_STREAM`
- Compute-only fire-and-forget pattern (like Chronos reminders)

## [CP17] — Clear missed history (control message)

`mock_chronos --missed-clear` zeroes display_server's in-memory
`g_missed[]` ring AND unlinks `/tmp/nous-missed.txt`. Closes the
CP16 limitation that `rm` of the persistence file only erased the
representation while leaving BSS state untouched.

- `NOUS_NOTIFY_MISSED_CLEAR = 13` (second control message kind
  after CP15 DISMISS_NOW)
- Drain-loop handles CLEAR before CP8 INFO filter
- Idempotent: second clear on empty ring = no-op
- Empirical proof memory actually zeroed: fresh missed run after
  clear yields count=1, not 3+ from accumulated state

## [CP16] — Per-task aggregation + scheduler priority send-order fix

Two improvements:

1. **Aggregation**: 5× missed "spacer" → ONE line in
   `/tmp/nous-missed.txt` with count=5, not 5 identical lines
2. **Race fix**: scheduler tick sends cards in priority DESCENDING
   order. Eliminates race where low-prio card was briefly promoted
   to current_slot before high-prio sibling arrived in the next
   poll cycle

- File format change: 5 fields (task | first_ns | count | last_ns |
  priority), was 3 fields pre-CP16
- Two-pass tick in `cmd_schedule_run`: compute levels → send in
  DESC order
- `missed_find()` linear scan for aggregation lookup
- Per-task `last_count` displayed as "task xN" in card

## [CP15] — User-initiated dismiss action

User can close active card NOW with D key (or future LMB / touch tap
in CP20b). Closes BEFORE TTL expires, but does NOT register as
missed — conscious close ≠ silent miss.

- `NOUS_ACTION_DISMISS = 5` in closed action catalog
- `KEY_D` added to `KEY_MAP[]`
- `NOUS_NOTIFY_DISMISS_NOW = 12` control message kind
- Drain-loop in display server handles DISMISS_NOW: evict current_slot,
  flag `was_displayed=true`, skip render
- No confirmation card after dismiss (would defeat the purpose)

## [CP14] — Missed reminders ring buffer + drain-loop fix

First retrospective UI: cards expiring "in shadow" of higher-priority
ones leave a trace in `/tmp/nous-missed.txt`. `mock_chronos --missed`
shows summary card.

- `was_displayed` flag in `card_slot_t`
- Ring buffer (10 entries) with file persistence
- `expire_pass` detects silent miss → `missed_add()`
- **Drain-loop fix** in poll handler: `for (;;) recv(MSG_DONTWAIT)`
  drains all pending dgrams in one wake-up before find_best_slot
  runs — prevents transient promotion race
- `NOUS_NOTIFY_MISSED_LIST = 11` kind
- `mock_chronos --missed` retrospective card

## [CP13] — Display card priority queue + INFO/REMINDER split

Display server refactored from single-card model ("newer wins") to
priority queue (8 slots). Latent bug discovered: scheduler BANNER
used `NOUS_NOTIFY_INFO`, which CP8 filter dropped when queue empty.
Resolved by introducing `NOUS_NOTIFY_REMINDER = 10` as separate
semantic kind.

## [CP12] — Voice → Scheduler wire-up

`mock_voice` after `NOUS_OK` automatically calls `mock_chronos
--schedule-add` via fork+execlp (no shell). Voice-created tasks
persist to `/tmp/nous-schedule.txt`.

- Stage 9 in voice pipeline (post-wipe, post-display)
- `task_name_safe()` validation: rejects `|`, `\n`, `\r`, ctrl chars
- Default weight ROUTINE for voice tasks
- Defense-in-depth: execlp guards shell injection, name validation
  guards file-format injection

## [CP11] — Persistent CALENDAR store

Multi-task schedule with `/tmp/nous-schedule.txt` persistence. 5
new subcommands: `--schedule-add`, `--schedule-list`,
`--schedule-remove`, `--schedule-clear`, `--schedule-run`.

- `sched_entry_t` with in-memory table (SCHED_MAX=16)
- Pipe-delimited file format
- Multi-event tick loop with per-task `last_level` tracking
- 5-minute grace period past start time

## [CP10] — Voice pipeline stub (F-c)

First input modality beyond keyboard and modem. `mock_voice "..."`
exercises full 8-stage pipeline.

- 8 stages: capture → diarize → audio_pfs → asr → intent → submit
  → display → secure_wipe
- §20 Kodeks: linear stage order, third voice stripped before
  transcription
- §21 Kodeks: secure_wipe of audio buffer on every exit path
- Polish keyword parser (verbs + time phrases + task names)
- `NOUS_NOTIFY_TASK_ADDED = 9` kind

## [CP9] — Autonomous reminder tick loop

System acts WITHOUT user trigger. `mock_chronos --schedule
<task> --start-in <s>` runs tick loop that auto-recomputes
intervention_level over time.

- Foreground or background (`&`) mode
- Level-change detection prevents spam
- Naturally produces SILENT → TOAST → BANNER → ALERT escalation
  as time-to-event approaches

## [CP8] — Active-card state machine + HEALTH cancel two-stage

Display server filters stray INFO cards (when no preceding non-INFO
alert). `mock_chronos --cancel <task>` → PENDING_REVIEW;
`--cancel-confirm <task> <reason>` → finalize. HEALTH cancel always
requires human review (§17 Kodeks).

## [CP7] — Polish diacritics (Latin-2 UTF-8)

Full Polish character set renders correctly in 8×8 font.

- 18 Polish glyphs in `.rodata` (144 bytes)
- UTF-8 2-byte decoder in `display/text.c`
- `nous_text_width_px` codepoint-aware (not byte-aware)
- Graceful fallback for unsupported codepoints (zero glyph, no crash)

## [CP6] — Intervention Escalation (F-b)

Full visual demonstration of subsidiarity. 6 Chronos levels mapped
to 3 visual variants: SILENT_ICON → no card; TOAST → MINI corner;
BANNER → INFO white; VIBRATION/SOUND/VOICE → ALERT inverted.

## [CP5] — Chronos → Display (F-a Time Buffer warning)

First moment in NOUS history when Chronos speaks to the user.
`mock_chronos kardiolog 23 45` triggers Time Buffer warning when
user estimate << historic duration.

- HAL dispatch routing by `intent_class`
- `mock_chronos` CLI tool linking libnous directly
- `NOUS_NOTIFY_TIME_BUFFER = 5` kind
- C/P keypresses for confirm/postpone

## [CP4] — Input path: full user-action loop

Closed decision loop: modem → display → user keypress → ACTION
intent → display update.

- Keyboard via virtio_input (virtio-keyboard-pci)
- Closed action catalog (4 keys, 4 actions)
- `NOUS_CTX_USER_LOCAL` distinguishes human-initiated from
  daemon-initiated intents

## [CP3] — Intent Bus → Display end-to-end

Full pipeline: `mock_qrtr` → RIL daemon → Intent Bus 9 steps →
display sink → notification card on screen.

- 8×8 bitmap font in `.rodata` (1024 bytes, signature aesthetic)
- 3-line notification card layout (CALL, SMS, INFO)
- AF_UNIX SOCK_DGRAM wire format with CRC32
- Display server poll loop (single-threaded, no mutex)

## [CP2] — QEMU graphical pipeline (virtio-gpu → SDL → dither)

Visual output works. virtio-gpu-pci in guest, fbcon unbinding, DRM
raw ioctl path, Atkinson dither + bit-expand.

## [CP1] — QEMU aarch64 end-to-end pipeline (green)

Cross-compiled NOUS boots in QEMU ARM64 under WSL2. RIL core accepts
mock event, propagates through Ring-1 architecture, audits in
ledger. **Hard rollback point.**

---

For each checkpoint, the corresponding section in
[CLAUDE.md](CLAUDE.md) documents the architectural invariants that
must hold for the checkpoint to remain green.
