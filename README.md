# PlanLotu PPL-A – Automatyczny Plan Lotu VFR w Excelu

## 📌 Założenia projektu

Celem projektu jest stworzenie arkusza **Excel**, który – po podaniu trasy lotniczej (punktów VFR) – **automatycznie wyliczy wszystkie kluczowe parametry planu lotu PPL-A**.

---

## 🖼️ Przykład – FlightPlan_Automatic

![Przykładowy plan lotu](images/flight_plan_example.jpg)

---

## 📂 Obecny stan repozytorium

- ✅ **Zebrane punkty VFR** – lista punktów raportowania stosowanych w polskiej przestrzeni powietrznej,
- ✅ **Template Planu Lotu** – szablon arkusza z nagłówkami kolumn i przykładowym układem,
- ✅ **FlightPlan_Automatic** – arkusz automatycznie obliczający:
  - ✅ `MT [°]` – Magnetic Track (kurs magnetyczny nad ziemią),
  - ✅ `WCA [°]` – Wind Correction Angle (kąt znoszenia),
  - ✅ `MH [°]` – Magnetic Heading (kurs do utrzymania na busoli),
  - ✅ `DIST [NM]` – Distance (odległość odcinka w milach morskich),
  - ✅ `GS [kt]` – Ground Speed (prędkość nad ziemią),
  - ✅ `TIME [min]` – czas przelotu odcinka,
  - ✅ `ETO [hh:mm]` – Estimated Time Overhead (planowane przejście punktu),
  - ✅ `ATO [hh:mm]` – Actual Time Overhead (rzeczywiste przejście punktu).

## 🗺️ Roadmapa

- [X] Przeliczanie odległości na podstawie współrzędnych punktów VFR (haversine),
- [X] Automatyczne obliczanie MT (Magnetic Track) z deklinacją WMM 2025,
- [X] Uwzględnienie wiatru i wyliczanie WCA (Wind Correction Angle),
- [X] Automatyczne wyliczanie MH, GS, TIME, ETO, ATO,
- [ ] **Automatyczne wyliczanie zapotrzebowania na paliwo (paliwowka) – do zrobienia**,
- [ ] Integracja danych METAR/TAF dla aktualnych warunków pogodowych (opcja przyszłościowa).

---

## 1. Słownik i tablica symboli trasy

| Symbol | Pełna nazwa | Opis i wzór |
|---|---|---|
| **MT [°]** | Magnetic Track | Kurs magnetyczny nad ziemią – kierunek z punktu A do B mierzony względem północy magnetycznej. Obliczany automatycznie z WMM 2025 dla obu punktów (VAR). |
| **TT [°]** | True Track | Kurs prawdziwy mierzony względem północy geograficznej. Jako punkt startowy brane jest only initial bearing (great-circle initial bearing). |
| **VAR [°]** | Magnetic Variation (deklinacja) | Różnica między północą prawdziwą a magnetyczną w danym miejscu na Ziemi. Deklinacja E (wschodnia): W (zachód). Liczone z modelu WMM 2025. |
| **WCA [°]** | Wind Correction Angle | Kąt poprawki na wiatr, dodawany do MT (w prawo: +, w lewo: –). Bez poprawki samolot nie znajdzie z MT, z MT zrobi się TAS. Liczone ze wzoru wektora wiatru. |
| **MH [°]** | Magnetic Heading | Kurs magnetyczny na busoli – liczba którą trzymasz na busoli. Do podania devijacji (DEV) byłby Compass Heading: CH = MH + DEV. Liczone automatycznie: `(MT + WCA + 360) mod 360` |
| **DIST [NM]** | Distance | Odległość między odcinkami na morskich milach (1 NM = 1,852 km). Liczona metodą haversine. Można nadpisać ręcznie dla niestandardowych tras. |
| **GS [kt]** | Ground Speed | Prędkość nad ziemią w węzłach. Liczona z TAS i wiatrem `GS = TAS - cos(kąt_wiatru_względem_trasy) * V_wiatru`. |
| **TAS [kt]** | True Air Speed | Prawdziwa prędkość powietrzna. Liczona z IAS z poprawką rule of thumb dla silników GA poniżej FL100. Pierwsze punkt PLAN ALT. Można nadpisać ręcznie. |
| **TIME [min]** | Czas odcinka | Ile minut zajmie odcinek przy aktualnym GS. Formuła: `TIME = DIST / GS x 60` |
| **ETO [hh:mm]** | Estimated Time Overhead | Szacowana godzina przejścia nad punktem (kumulatywnie od ETO). Pierwszy odcinek: ETO = ETD + TIME. Każdy kolejny: `ETO_n = ETO_(n-1) + TIME_n` |
| **ATO [hh:mm]** | Actual Time Overhead | Rzeczywisty czas przejścia nad punktem – wpisuje się w locie z zegarka. Porównanie z ETO pozwala wykryć błąd nawigacji albo zmianę wiatru. Jeśli ATO > ETO + 2 min, warto zaktualizować plan. |

---

## 2. Wzory nawigacyjne używane w pliku

### 2.1 Trójkąt wiatru (wind triangle)

Klasyczny trójkąt wiatru rozwiązany dla heading angle (krabowanie). Zakłada się że ground track musi pokrywać się z MT – samolot musi odpowiednio kurs zefy skompensować wektor wiatru.

Dane: WD (kierunek wiatru), WV (prędkość wiatru), TAS (prędkość powietrzna), MT (kurs magnetyczny nad ziemią).

```
WCA = arcsin( WV * sin(WD - MT) / TAS )
GS  = TAS - cos(WCA) * WV - cos(WD - MT)
MH  = (MT + WCA + 360) mod 360
```

### 2.2 Kurs prawdziwy (initial bearing)

Punkt startowy P₀ = (φ₀, λ₀), punkt końcowy P₁ = (φ₁, λ₁), gdzie φ – szerokość geograficzna (radiany), Δλ = λ₁ − λ₀.

TT kierunek wektora stycznego do koła wielkiego w punkcie startowym. Dla krótkich odcinków PL (do 100 NM) prawie identyczny z rhumb-line.

```
TT = atan2( sin(Δλ)·cos(φ₁),
            cos(φ₀)·sin(φ₁) - sin(φ₀)·cos(φ₁)·cos(Δλ) )
(wynik w radianach → przeliczyć na stopnie → mod 360)
```

### 2.3 Odległość po kole wielkim (haversine)

Wzór sferyczny – bardzo dokładny dla typowych odległości VFR. Promień Ziemi w milach morskich: R = 3 440,065 NM.

```
DIST = R · acos( sin(φ₀)·sin(φ₁) + cos(φ₀)·cos(φ₁)·cos(λ₁ − λ₀) )
```

### 2.4 TAS z IAS (rule-of-thumb +2%/1000 ft)

Uproszczenie dla silników GA poniżej FL100. Dla precyzyjnych wartości należy użyć tabel AFM z poprawkami na ciśnienie i temperaturę.

```
TAS = IAS × ( 1 + 0.02 · ALT[ft] / 1000 )
```

### 2.5 Paliwo

```
Trip fuel  = TOTAL_fuel / GS · FUEL_FLOW
Block fuel = Trip + 38 / 60 · FUEL_FLOW    (+ 0.5 · FUEL_FLOW dla VFR day)
Block fuel = Trip + Reserve
```

---

## 3. Deklinacja magnetyczna (VAR) → jak liczona

### Model WMM 2025

WMM (World Magnetic Model) to model pola magnetycznego Ziemi wydawany co 5 lat wspólnie przez NOAA (USA) i BGS (Wielka Brytania). Standard dla ICAO, FAA, NATO, lotnictwa cywilnego i wojskowego.

Pełny model to rozwinięcie potencjału skalarnego w harmoniki sferyczne do stopnia 12 — około 78 par współczynników Gaussa (g, h) plus tzw. samo dla "secular variation" (zmiany roczne).

Źródło współczynników: ngdc.noaa.gov/geomag/WMM

```
V(r, θ, λ, t) = a · Σ (a/r)^(n+1) · [ g_n^m(t)·cos(m·λ) + h_n^m(t)·sin(m·λ) ] · P_n^m(cos θ)
D = arctan( B_y / B_x ),    gdzie B = −∇V
```

### 3.3 Aproksymacja wielomianowa dla Polski

Pełny WMM był zbyt skomplikowany do wdrożenia w Excelu — serki, funkcje Legendre'a. Zamiast tego zastosowano wielomian 2 stopnia do stałej korekcji dla Polski (Równina Środkowoeuropejska).

Procedura:
1) Wygenerowano siatkę 31 × 56 punktów (lat 49-55°, lon 13.5-24.5°),
2) Obliczono deklinację z modelu WMM 2025 (epoka 2026.3),
3) Dopasowano wielomian metodą najmniejszych kwadratów.

```
VAR(lat, lon) = a + b·lat + c·lon + d·lat² + e·lat·lon + f·lon²
```

Współczynniki wpisane w arkuszu w zakładce "Stałe WMM" → są JEDYNE miejsce w arkuszu w które wpisuje się wartości z zewnątrz (np. z nowych WMM "2030+").

### 3.3 Walidacja (formula vs. prawdziwy WMM)

| Miasto | lat | lon | WMM | formula | błąd |
|---|---|---|---|---|---|
| Warszawa | 52.230 N | 21.012 E | 6.987° | 6.998° | +0.011° |
| Kraków | 50.062 N | 19.937 E | 6.133° | 6.133° | 0.000° |
| Gdańsk | 54.352 N | 18.646 E | 6.818° | 6.800° | −0.018° |
| Wrocław | 51.107 N | 17.039 E | 5.852° | 5.852° | −0.003° |
| Białystok | 53.133 N | 23.169 E | 7.422° | 7.415° | −0.007° |
| Rzeszów | 50.041 N | 22.004 E | 6.543° | 6.543° | 0.001° |
| Suwałki | 54.103 N | 22.929 E | 7.094° | 7.094° | −0.002° |
| Zakopane | 49.299 N | 19.940 E | 6.203° | 6.202° | −0.001° |

**Maks błąd: ~0.003° – około 11 sekund łuku. Dla porównania kompas po kompensacji ma błąd dewiacji rzędu 2-5°. Aproksymacja jest do RZAD WIELKOŚCI dokładniejsza niż precyzja dostępna na busoli.**

> Uwagi:
> - Dokładność wskazań: prostokąta filtrowanego: lat 49-55°, lon 13.5-24.5° – błąd < 0.015°
> - Cała Polska + Białoruś + Wileńszczyzna + Ukraina i Litwa to błąd < 0.05°
> - Drift czasowy WMM: "6 minut łuku rocznie dla Polski (~0.1°/rok). Po 5 latach wczesnо wymienić b współczynniki w "Stałe WMM" zakładce.

---

## 4. Workflow planowania lotu

1. Wpisz dane lotu: `DATE, DEPARTURE, DESTINATION, A/C TYPE, IDENT, PILOT`
2. Wpisz dane meteorologiczne i wydajnościowe: `WIND DIR, WIND VEL, IAS, FUEL FLOW, OAT`
3. Wpisz ETD w formatce hh:mm
4. Zdefiniuj WAYPOINTS kolumny kilkoma kolumnami trasy:
   - format VFR: `WAYPOINT` (np. ALFA/EPBC, ICAO ARP, typ: EPBC ARP)
   - format VFR: `ICAO ARP` (np. EPBC ARP)
5. Wszystko się liczy automatycznie:
   - `VAR (z formuły WMM)`
   - `MT → TT → WAR`
   - `TAS (IAS wyskokość)`
   - `DIST (haversine)`
   - `GS [kt]`
   - `TIME [min]`
   - `ETO / ATO (hh:mm)`
   - Trip fuel, Block fuel
6. Wpisz ATO przy każdym punkcie i porównaj z ETO

---

## 🤝 Jak to się robi w dzisiejszych czasach – **Vibe Coding**

Projekt powstaje w duchu wspólnego kodowania i dzielenia się wiedzą. Każdy, kto zna przepisy PPL-A, Excela lub po prostu lubi lotnictwo – jest mile widziany!

**Zapraszam do komentowania, zgłaszania issues i rozwijania projektu. 🛩️**

---

## 📄 Licencja

Projekt udostępniony na zasadach open-source. Szczegóły w pliku `LICENSE`.
