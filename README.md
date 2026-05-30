# Jakie praktyki korelują z jakością PR-ów tworzonych przez agentów?

Analiza zbioru **AIDev-pop** (~33,6 tys. PR-ów napisanych przez agentów AI:
OpenAI Codex, Copilot, Devin, Cursor, Claude Code) odpowiadająca na pytanie:
*które praktyki (rozmiar PR-a, typ zadania, granularność commitów) wiążą się z
jakością PR-ów agentowych?*

Cała analiza jest w [analysis.ipynb](analysis.ipynb), a wyniki liczbowe w
[analysis_outputs/](analysis_outputs/).

## Co badamy

**Praktyki (zmienne niezależne):**
- **Rozmiar PR-a** – dodane / usunięte / zmienione linie, liczba plików
- **Granularność commitów** – liczba commitów, średni i maks. rozmiar commita
- **Typ zadania** – kategoria Conventional Commit (`feat`, `fix`, `docs`, …)
- **Opis** – długość tytułu i treści, powiązanie z issue

**Wskaźniki jakości (zmienne zależne):**
- czy PR został zmergowany,
- czas do zmergowania,
- liczba komentarzy inline w review,
- liczba żądanych zmian (*changes requested*).

## Proces i metody

Każda metoda pokazuje **co innego**, dlatego stosujemy ich kilka:

| Metoda | Wynik | Co mówi |
|---|---|---|
| Korelacja **Spearmana** | ρ | surowy, jednowymiarowy związek monotoniczny (bez kontroli) |
| Regresja **logistyczna** | iloraz szans (OR) | szansa zmergowania, z kontrolą pozostałych zmiennych |
| Model **Coxa** (PH) | hazard ratio (HR) | *szybkość* mergowania (HR>1 = szybciej); obsługuje cenzurowanie wciąż otwartych PR-ów |
| **Ujemna dwumianowa** (NB) | IRR | liczniki (komentarze, żądane zmiany) — radzi sobie z naddyspersją |

Kluczowe decyzje w przygotowaniu danych:
- **Logarytmowanie (`log1p`) skośnych predyktorów** (linie, pliki, commity,
  długości, gwiazdki repo). Rozkłady mają **ciężkie ogony** — pojedyncze
  ogromne PR-y zdominowałyby modele i zaburzyły współczynniki; po transformacji
  efekty są stabilne i interpretowalne.
- **Kontrole**: agent, `log(gwiazdki repo)`, język repozytorium — żeby efekty
  praktyk nie myliły się z tym, *kto* i *gdzie* tworzył PR.
- **Błędy standardowe klastrowane na poziomie repozytorium** (PR-y z jednego
  repo nie są niezależne).
- Cox uwzględnia **cenzurowanie** (otwarte PR-y to brak zdarzenia, nie „szybki
  merge”); rzadkie typy zadań i języki łączone są w „other”.

## Najważniejsze wyniki

**1. Granularność commitów to najsilniejszy i najbardziej dwuznaczny sygnał.**
Po kontroli pozostałych zmiennych więcej commitów (`log_n_commits`):
- **zwiększa szansę mergowania** (OR ≈ 1,67, p < 0,001),
- ale ściąga **znacznie więcej uwagi w review** — komentarze inline IRR ≈ 3,7
  (p < 0,001), żądane zmiany IRR ≈ 2,8 (p < 0,001),
- i **wydłuża czas do mergowania** (HR ≈ 0,76 → mniej commitów = szybszy merge).

Co ciekawe, surowa korelacja jest *odwrotna* (ρ = −0,23 z mergowaniem): w danych
więcej commitów towarzyszy gorszym PR-om, ale gdy odfiltrujemy rozmiar i agenta,
sam fakt iterowania commitami pomaga PR-owi przejść — dobra ilustracja, **po co
modele wielowymiarowe**.

**2. Większy PR = więcej oporu przy ocenie, niższa akceptacja.** Więcej dodanych linii
obniża szansę mergowania (OR ≈ 0,79) i spowalnia merge (HR ≈ 0,91), a podnosi
liczbę komentarzy (IRR ≈ 1,31) i żądanych zmian (IRR ≈ 1,38). Rozłożenie zmian na
**więcej plików** za to *zmniejsza* liczbę żądanych zmian (IRR ≈ 0,82, p < 0,001).

**3. Typ zadania ma realne znaczenie dla akceptacji.** Wskaźnik mergowania waha
się od **0,84 dla `docs`** do 0,55 (`perf`) i 0,32 (`other`); `fix`/`feat`
~0,65–0,71. W modelu `docs` ma najwyższą szansę mergowania, a `perf`, `fix` i
`feat` niższą (względem `build`).
PR-y `docs` i `chore` ściągają przy tym więcej komentarzy inline (IRR ≈ 1,8 i 1,6).

**4. Powiązanie z issue nie zwiększa akceptacji.** Wbrew intuicji nie wpływa na
szansę mergowania (OR ≈ 1,01, nieistotne), za to wiąże się z *wolniejszym*
mergowaniem (HR ≈ 0,79) i większą liczbą komentarzy inline (IRR ≈ 1,42) — to
raczej znacznik trudniejszych, bardziej dyskutowanych zadań.

**5. Najwięcej różnic jest między agentami** (dlatego są kontrolowane w modelach).
Codex: 83% mergowanych (n ≈ 21,8 tys., mediana 1 commit, 74 zmienione linie);
Copilot tylko 43% i generuje **wielokrotnie więcej żądanych zmian**
(IRR ≈ 7,2 vs baza), Devin podobnie (IRR ≈ 3,3). Mniejsze, prostsze PR-y idą w
parze z lepszymi wynikami.

> **Wniosek:** **Typ zadania, rozmiar zmiany i granularność commitów** wszystkie mają
> wkład w jakość PRa. 
> Niska Cramér's V typu zadania (0,14, formalnie „mała") nie oznacza więc,
> że typ jest nieistotny — to artefakt miary przy wyniku binarnym,
> w którym większość PR-ów to `feat`/`fix` blisko średniej; w
> regresji efekt `docs` vs `perf` jest realny i istotny. Praktycznie: mniejsze
> PR-y mergują się częściej i szybciej; bardziej rozdrobnione (granularne) commity zwiększają szansę
> akceptacji kosztem dłuższego review; `docs` są akceptowane najchętniej,
> `fix`/`perf` najrzadziej. Powiązanie z issue nie jest samo w sobie gwarancją
> jakości.

## Struktura wyników

- `logit_merged.csv` – regresja logistyczna (P(merge))
- `cox_time_to_merge.csv` – model Coxa (czas do mergowania)
- `negbin_inline_cmts_glm.csv` – NB dla komentarzy inline
- `negbin_changes_requested.csv` – NB dla żądanych zmian
- `bivariate_spearman.csv` – korelacje Spearmana
- `summary_by_agent.csv`, `merge_rate_by_task_type.csv` – statystyki opisowe
- `figs/` – wykresy
