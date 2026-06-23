# NOUS OS — stan projektu (snapshot)

Autor: Artur Mielko · Ostatnia aktualizacja: kamień milowy CP27.

## Gdzie jesteśmy w jednym zdaniu

NOUS OS to **w pełni nawigowalny dotykiem telefon** działający w
symulatorze (QEMU, pionowy ekran 540×960), z ascetycznym interfejsem
1-bit, geometryczną czcionką, motywem jasny/ciemny i kompletem
ekranów-aplikacji — przetestowany automatycznie (**19/19 scenariuszy
zielonych**). Docelowo trafi jako warstwa na postmarketOS / OnePlus 6.

## Co działa — wszystko palcem, bez terminala

| Ekran / funkcja | Co robisz |
|---|---|
| **Ekran domowy (launcher)** | 6 aplikacji z ikonami; dotknięcie zegara → motyw jasny/ciemny |
| **Telefon** | klawiatura 1-9 * 0 #, wpis numeru, dzwonienie, kasowanie |
| **Kontakty** | lista; dotknięcie = połączenie wychodzące |
| **Wiadomości** | dyktowanie (Vosk) / wyślij SMS (szkielet) |
| **Ustawienia** | motyw, Nie-Przeszkadzać, Dignity Override |
| **Przypomnienia / Samopoczucie** | huby do kart Chronos / Wellbeing |
| **Połączenie / SMS / alert przychodzący** | karta wskakuje nad każdy ekran; ODBIERZ/ODRZUĆ lub dotknięcie = zamknij |
| **Powrót** | chevron „‹" w lewym górnym rogu każdego ekranu |

## Postęp wg planu wdrożenia (APPS.md)

| # | Pozycja | Status |
|---|---|---|
| 1 | Nawigacja ekranów + Ustawienia | ✅ |
| 2 | Telefon (dialer) | ✅ |
| 3 | Kontakty | ✅ |
| 4 | Wiadomości + klawiatura ekranowa | ✅ QWERTY + wyślij (Vosk-dyktowanie demo) |
| 5 | Strażnik uwagi (anty-prokrastynacja) | ✅ silnik regułowy + seam klasyfikatora (real SLM na urządzeniu) |
| 6 | Mapy + GPS | ⬜ wymaga sprzętu |
| 7 | Mosty: WhatsApp / Telegram / Signal | ⬜ wymaga serwera + sprzętu |

## Jak uruchomić symulator (WSL2 / Linux)

```bash
cd "/mnt/c/Users/amielko/Desktop/Nous OS/nous-os"
./scripts/run_phone.sh          # pionowe okno telefonu 540×960
```
Kliknij okno (focus), potem klikaj kafelki myszą (= dotyk). Wyjście: `Ctrl-A`, potem `x`.

Tryby debug: `NOUS_SPLASH=checker` (test ditheringu), `NOUS_IDLE=1`
(goły zegar), `NOUS_THEME=dark` (start w ciemnym).

## Testy automatyczne (bez klikania ręką)

```bash
NOUS_TEST_XRES=540 NOUS_TEST_YRES=960 python3 tests/system/harness.py --suite full
```
Headless QEMU → komendy serialem → asercje na logach → **zrzuty ekranu,
które agent ogląda** → wstrzykiwanie dotyku/klawiszy przez QMP.
Pojedynczy scenariusz: `--only <fragment-nazwy>`.

## Galeria

W `docs/screenshots/`: `shell_home.png`, `shell_home_dark.png`,
`shell_dialer.png`, `shell_settings.png`, `shell_contacts.png`,
`shell_call_card.png`, `card_incoming_call.png`, `card_wellbeing_alert.png`,
`card_missed_list.png`, `card_time_buffer.png`, `idle_screen.png`.

## Historia checkpointów (skrót; szczegóły w CLAUDE.md)

- **CP1–CP13** — pipeline QEMU→ekran, Intent Bus, Chronos, voice stub,
  polskie znaki, kolejka kart z priorytetami.
- **CP14–CP17** — przegapione przypomnienia (ring + plik), dismiss (D),
  agregacja, czyszczenie historii.
- **CP18–CP20a** — Wellbeing: nudge, decay, filtr sec_ctx (rozmowy
  przychodzące NIE liczą się jako stres), snapshot.
- **CP20b** — dotyk/wskaźnik (virtio-tablet).
- **CP21** — autonomiczny harness testowy (build→boot→test→screenshot).
- **CP22** — mostek ModemManager→NOUS (telefonia na pmOS), audyt obrazu OP6.
- **CP23–CP24** — ekran spoczynku z zegarem, ikony, tier geometryczny.
- **CP25** — interaktywna powłoka dotykowa (kafelki + hit-testing).
- **CP26** — wektorowy font geometryczny + motywy light/dark.
- **CP27** — framework nawigacji + dialer/kontakty/wiadomości/ustawienia.

## Uczciwe ograniczenia

- **WiFi i realne połączenia/SMS NIE działają w symulatorze** — QEMU nie
  ma radia ani modemu. Pojawią się dopiero na fizycznym OnePlus 6 + pmOS.
- W symulatorze dialer/kontakty pokazują **karty demonstracyjne**; na
  urządzeniu te same dotknięcia pójdą przez ModemManager (mostek CP22).
- **WhatsApp**: brak natywnego klienta Linux — rekomendowany most Matrix
  na własnym serwerze (szczegóły w APPS.md).
- To **prototyp/warstwa**, nie samodzielny pełny OS telefonu — podstawy
  (kernel, modem, kompozytor) bierzemy z postmarketOS.

## Następny krok

Pozycje 1-3 i 5 (silnik) zrobione w symulatorze. Zostaje: #4 pełna
klawiatura ekranowa / Vosk do SMS, mini-SLM jako klasyfikator strażnika,
oraz #6-7 (mapy/GPS, mosty komunikatorów) — które wymagają fizycznego
OnePlus 6 + postmarketOS. Kolejny naturalny krok bez sprzętu: #4 albo
mini-SLM strażnika.
