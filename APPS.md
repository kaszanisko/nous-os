# NOUS OS — manifest aplikacji preinstalowanych (propozycja przed wdrożeniem)

Autor: Artur Mielko. Cel: zdefiniować, co **fabrycznie** jest na urządzeniu
(OnePlus 6 / postmarketOS + warstwa NOUS), zanim zrobimy obraz produkcyjny.

## Zasada doboru — to jest oświadczenie wartości, nie lista funkcji

NOUS jest systemem dla osób z ADHD, zaprojektowanym **przeciw** pętli
dopaminowej. Dlatego dobór aplikacji rządzi się regułą odwrotną niż w
zwykłym telefonie: **domyślnie jest tego MAŁO, a każda pozycja musi
zarobić na obecność.** Nie ma sklepu z apkami, nie ma feedów, nie ma
nieskończonego scrolla. To **feature**, nie brak.

Trzy filtry, każdą pozycję musi przejść:
1. **Czy służy intencji użytkownika, czy ją porywa?** (kalendarz służy;
   feed porywa)
2. **Czy da się zrobić ascetycznie** (1-bit, karta, bez animacji-nagród)?
3. **Czy respektuje Kodeks** (§4 sieć-off-by-default, §7 izolacja,
   PFS, brak telemetrii)?

---

## TIER 1 — NOUS-native (budujemy my, UI kart/kafelków)

Kompilowane do obrazu, ascetyczne, sterowane dotykiem. Status: co już jest.

| App | Funkcja | Status |
|---|---|---|
| **Telefon (dialer)** | klawiatura numeryczna → połączenie wychodzące; przychodzące jako karta | do zbudowania |
| **Kontakty** | lista „ulubieni najpierw", mała, bez scrolla-nałogu | do zbudowania |
| **Wiadomości** | SMS + (przez most) WhatsApp/Telegram/Signal w jednej karcie; pisanie **głosem (Vosk)** lub klawiaturą ekranową | do zbudowania |
| **Chronos** | przypomnienia z 5-poziomową eskalacją, time-buffer, lista przegapionych | ✅ CP5-CP19 |
| **Samopoczucie** | monitor presji systemu, decay, nudge | ✅ CP18-CP20a |
| **Strażnik uwagi** | lokalny mini-SLM: anty-prokrastynacja, „wróć do tego co zaplanowałeś" | Faza F (zaprojektowane) |
| **Ustawienia** | motyw, jasność, głośność, profil dźwięku, Nie-Przeszkadzać, Dignity Override | do zbudowania |
| **Notatki** | głosowe, minimalne | do zbudowania |
| **Mapy** | Pure Maps + offline OSM (nawigacja, pozycja z GPS/dongla) | adopcja + integracja |

## TIER 2 — adopcja ze stosu pmOS (istnieją, my themujemy/spinamy)

Nie piszemy od zera — bierzemy gotowe, ascetyzujemy wygląd, spinamy z
Intent Bus tam gdzie ma sens.

| App | Po co | Uwaga |
|---|---|---|
| **ModemManager + Calls** | backend telefonii (głos 2G/CSFB, połączenia) | rdzeń — mostek CP22 już gada z MM |
| **Chatty** | backend SMS + klient Matrix (brama do WhatsApp) | albo nasze „Wiadomości" na wierzchu |
| **Pure Maps + GeoClue** | mapy + GPS | GNSS na OP6 do weryfikacji (Faza C) — fallback dongle USB |
| **Foliate / czytnik** | e-booki, PDF — „głębokie skupienie", anty-feed | opcjonalnie |
| **Portfolio** | menedżer plików | minimalny |
| **Zegar / Budzik / Kalkulator** | podstawy | lekkie |

## TIER 3 — komunikatory (wymagają mostów / pracy)

| App | Droga | Rekomendacja |
|---|---|---|
| **WhatsApp** | most mautrix-whatsapp na serwerze użytkownika → klient Matrix | **rekomendowane** (Twój serwer, jeden UI wiadomości) |
| **Telegram** | klient natywny Linux (działa) lub most mautrix-telegram | natywny na start, most później |
| **Signal** | Axolotl / Flare (klienci pmOS) lub most | natywny |
| **E-mail** | Geary / nasz minimalny | opcjonalnie, z PFS na załącznikach |

## ŚWIADOMIE WYKLUCZONE — to jest punkt, nie luka

- **Sklep z aplikacjami / market** — nie. Obraz jest kurowany; instalacja
  = świadoma decyzja przez `apk`, nie one-tap impuls.
- **Feedy społecznościowe** (FB/IG/TikTok/X/YouTube-app) — nie. Cały
  thesis anty-dopaminowy. Jeśli user chce, robi to w przeglądarce, a
  **Strażnik uwagi to widzi** i interweniuje.
- **Gry-zżeracze czasu, apki z reklamami, cokolwiek z nagrodą zmienną** — nie.
- **Przeglądarka** — TAK, ale **bramkowana**: jedna karta (WEB_VIEW_CAP=1,
  §64), content filter (§67), i jest jedynym wektorem dopaminy, który
  Strażnik uwagi aktywnie obserwuje.

---

## Co znaczy „preinstalowane" technicznie

Na pmOS = pakiety `apk` wpieczone w rootfs przy budowie obrazu +
nasze binarki NOUS skompilowane do obrazu. Czyli ten manifest to
dosłownie: lista `apk add` (Tier 2/3) + nasze artefakty (Tier 1) +
skrypty konfiguracji mostów (WhatsApp/Matrix).

## Rekomendowana kolejność budowy (Tier 1)

Każdy krok testowany autonomicznie w symulatorze (harness), zanim
trafi na sprzęt. Kolejność dobrana tak, by **najpierw zbudować
framework, który odblokowuje resztę**:

1. **Nawigacja ekranów + Ustawienia** — framework „home → ekran → powrót"
   (potrzebny KAŻDEMU kolejnemu ekranowi) + hub ustawień (motyw,
   DND, Dignity). *Najwyższa dźwignia.*
2. **Telefon (dialer)** — klawiatura numeryczna, north-star „dzwonić".
3. **Kontakty** — lista + integracja z dialerem i wiadomościami.
4. **Wiadomości + klawiatura ekranowa / dyktowanie** — SMS na start.
5. **Strażnik uwagi (Faza F)** — reguły + mini-SLM.
6. **Mapy** — Pure Maps + weryfikacja GPS (wymaga sprzętu).
7. **Mosty komunikatorów** (WhatsApp/Telegram/Signal) — wymaga
   serwera + sprzętu (Faza D/po zakupie OP6).

Pozycje 1-5 są w pełni testowalne w symulatorze JUŻ. 6-7 potrzebują
fizycznego OnePlusa.
