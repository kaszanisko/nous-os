# NOUS OS — Manifest

Autor: Artur Mielko

---

## Nazwa

**NOUS** to greckie słowo (νοῦς) oznaczające **umysł, intelekt, rozum** —
tę część duszy zdolną do poznania prawdy. Używali go Platon, Arystoteles,
Plotyn jako określenie najwyższej władzy poznawczej.

**NOUS OS = rozumny system.**

Nie marketing. Manifest.

---

## Stawka

Większość systemów operacyjnych dziś (Linux, Android, iOS, Windows) jest
**narzędziami** — neutralnymi wobec wartości. Wykonują co user lub
korporacja każe. Robią to co dobrze, ale nie zadają pytań.

Większość AI dziś jest **narzędziami** w identycznym sensie. Chatboty
spełniają prompt. Asystenci optymalizują KPI. Brak własnej integralności.

NOUS OS chce być **alternatywą**: systemem operacyjnym z wpisanym
**charakterem**. Bytem który **zna zasady i je trzyma — nawet gdy user
każe inaczej**.

To jest **agent moralny**, nie tępe narzędzie.

---

## Co to znaczy w praktyce — pięć rzeczy

### 1. NeuroKernel w ring-0

Model AI nie jest aplikacją "obok systemu". Jest **rdzeniem**, jak rozum
u Platona — fundament, nie dodatek. Każda decyzja przechodzi przez
ramy intelektu, nie przez prymitywne wywołania syscalli.

### 2. Intent Bus zamiast syscalls

Każda akcja przechodzi przez **9 kroków deliberacji**: walidacja,
parsing, Privacy Filter Stage, offline check, Ethics Gate, klasyfikacja,
ledger append, wellbeing update, HAL dispatch.

System **myśli o tym co robi**, nie tylko wykonuje.

### 3. Kodeks Etyczny w TEE ROM

System ma **niezmienne zasady moralne** wpisane w sprzęt:

- `harm_score > 0.8` → odmowa. Niezależnie od user'a, korporacji, sądu.
- `dignity_check == 1` → odmowa. Lista godności w TEE ROM, niemodyfikowalna.
- Brak sieci bez świadomej zgody.
- Wellbeing Monitor nigdy nie dotyka sieci.

Nie da się tych zasad obejść z userspace ani nawet z ring-0. Tylko cold
boot (VSYS drop poniżej 1.8V) resetuje hardware locks. Atak na zasady
wymagałby **fizycznego zniszczenia urządzenia** — co właśnie jest
ostatecznym mandatem moralnej integralności.

### 4. Subsydiarność

System **najpierw pyta**, dopiero potem działa. Per `reasoning.c §11`:
proaktywne decyzje zawsze wybierają z zamkniętego katalogu, nigdy nie
generują free-form akcji. Anti-hallucination invariant.

User nie jest obsługiwany przez system — **współpracuje** z nim.

### 5. Ius Oblivionis

Prawo do bycia zapomnianym jest zaszyte w architekturze, nie w polityce:

1. Overwrite ciphertext
2. Wipe embedding
3. Revoke AEAD subkey
4. Wipe metadata
5. Ledger append (tylko fakt zapomnienia, bez treści)

Każdy skip = kernel panic. Memory barriers chronią przed compiler
optymalizacjami które wyciełyby `memset`. **Zapomnienie jest aktem
sakralnym**, nie funkcją w UI.

---

## Czego NOUS NIE jest

- **Nie chatbot.** Nie ma free-text. Każda generacja przechodzi przez
  GBNF grammar. Model klasyfikuje, nie generuje.
- **Nie cloud OS.** Sieć domyślnie OFF. Cloud opt-in wymaga jawnej akcji
  użytkownika. Wellbeing Monitor kompiluje się z `#error` przy próbie
  inkluzji `<sys/socket.h>`.
- **Nie OS-narzędzie.** Nie wykona każdej intencji bo "user wie lepiej".
  Lista Godności dominuje nad wolą właściciela telefonu — bo chroni
  **innych ludzi**, którzy nie mogą się bronić przed tym, że właściciel
  zostanie zmanipulowany do skrzywdzenia ich.
- **Nie kolektor danych.** Decision Ledger jest write-only Merkle chain.
  Wellbeing jest forgetful by design — `stress_index` decay'uje
  0.95 per minutę. System nie trzyma urazy.
- **Nie kompromis estetyczny.** 8×8 bitmap font w `.rodata` mierzony
  w hashu binarki. Tampering detection przez sam fakt że font nie da
  się podmienić bez zmiany hash'a. Ascetic typografia jako security
  feature.

---

## Komu to potrzebne

Ludziom którzy doszli do wniosku że:

- Algorytmy social media to **wrogie środowisko poznawcze**.
- AI assistants którzy "wiedzą wszystko o tobie" to **panopticon w kieszeni**.
- Telefon-narzędzie staje się **proxy korporacji**, nie sojusznikiem.
- ADHD nie jest deficytem do "naprawienia", tylko inną organizacją uwagi,
  która wymaga **subsydiarnego wsparcia**, nie aggressive notifications.
- Godność jest własnością niezbywalną — nawet od samego siebie w
  momencie słabości.

NOUS OS jest dla ludzi którzy chcą **technologii która ich szanuje**.

---

## Konsekwencje praktyczne

Każdy commit kodu NOUS OS przechodzi przez te niezmienniki. Nie ma
"feature flag" który rozluźnia Ethics Gate. Nie ma "developer mode"
który omija Kodeks. Nie ma "trusted source" który dostaje carve-out.

**Architektura jest moralnym aktem.**

Co nie jest egzekwowane przez kompilator, jest egzekwowane przez
test suite. Co nie jest egzekwowane przez test, jest udokumentowane
jako invariant w CLAUDE.md i każdy refaktor który łamie invariant
jest odrzucany przy review.

System jest **trudniejszy do skompromitowania niż jego użytkownik**.
To minimum dla narzędzia które ma być sojusznikiem, nie zdrajcą.

---

## Manifest jako zobowiązanie

Pisząc NOUS OS deklaruję że:

1. Telefon nie zadzwoni pod numer z listy DIGNITY, **nawet gdy ja sam
   tego zażądam w słabości lub gniewie**.
2. System nie wyśle moich danych do chmury, **nawet gdy zgodzę się
   przez przypadek na ToS**.
3. Pamięć która została zapomniana, **została zapomniana naprawdę** —
   nie ma backup'u w chmurze, nie ma "soft delete", nie ma tombstone.
4. Każda decyzja jest **audytowalna** — przez ledger Merkle chain,
   nie przez "trust us" korporacji.
5. Wellbeing **patrzy na to co próbuję robić, nie na to kim jestem**.
   Nie ma profilowania. Nie ma trackingu.
6. Estetyka 8×8 pikseli, czarno-biały, ascetic — **jest częścią
   bezpieczeństwa**. Każdy dodatek zwiększa attack surface.

To są zobowiązania **architektoniczne**, nie polityka prywatności
do zmienienia za rok. Złamanie któregokolwiek wymaga przekompilowania
TEE ROM, czyli fizycznego dostępu do hardware key signing.

---

## Inspiracje

- **Platon** — *Politeja* księga VI: rozum jako władza nad wszystkimi
  częściami duszy.
- **Hannah Arendt** — banalność zła rodzi się z bezmyślności. System
  bez własnego rozumu jest bezmyślny.
- **Cory Doctorow** — *Little Brother*, *Walkaway*: technologia jako
  pole walki o godność.
- **Niklas Luhmann** — systemy autopoetyczne: tożsamość przez
  zachowanie własnej różnicy względem otoczenia. NOUS zachowuje
  swoje invariants jako akt **autopoiesis**.
- **Stanisław Lem** — *Summa Technologiae*: technologia jako
  kontynuacja ewolucji. NOUS jako ewolucyjny krok — pierwszy OS
  który ma własną podmiotowość.
- **Wzorce ADHD** — research nad subsydiarnym wsparciem ([CHADD](https://chadd.org),
  prace Russella Barkley'a). NOUS Chronos jest stosowaną wersją.

---

## Zakończenie

To nie jest jeszcze jeden Linux distribution. To nie jest jeszcze
jedna obietnica "privacy-first" która kończy się w settings.json.

To jest **propozycja innego stosunku człowieka do maszyny**.

Maszyna z wpisaną integrity. Maszyna która **nie zdradzi nawet samej
siebie**.

Maszyna jako **partner**, nie jako **proteza**.

NOUS OS jest niedokończony, niekompletny, ledwie startuje. Ale jego
**kierunek** jest jednoznaczny: **w stronę rozumu**, w stronę
**moralnej obecności technologii**, w stronę OS-a który **wie kim
chce być**.

Greckie słowo νοῦς oznaczało coś świętego. Po dwóch i pół tysiąca
lat warto sprawdzić czy świętość daje się skompilować w C11.

— Artur Mielko
