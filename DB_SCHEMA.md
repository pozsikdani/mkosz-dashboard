# Adatbázis séma — `mkosz_stats.sqlite`

> Utolsó frissítés: 2026-05-13. Ez a dokumentum a `kozgazkosar.hu` weblap szempontjából írja le a központi adatbázist.

## Áttekintés

A `mkosz_stats.sqlite` adatbázis a **központi adatforrás** a `kozgazkosar.hu` weblap számára. A `mkosz-stats` repó tárolja és tartja karban; a `mkosz-dashboard/generate_dashboards.py` csak olvas belőle (`STATS_DB_PATH=../mkosz-stats/mkosz_stats.sqlite`).

**Két párhuzamos adatfolyam tölti fel:**

1. **Scoresheet ág** — a `mkosz-scoresheet` projekt letölti az MKOSZ hivatalos PDF jegyzőkönyveit, kinyeri a box score-t, negyedeket, faultokat. Megyei (képes) PDF-eknél web fallback van. Forrás: `source='scoresheet'` a `player_game_stats`-ben.
2. **Play-by-play (PBP) ág** — a `mkosz-play-by-play` projekt scrape-eli az MKOSZ event listáját (lövés, asszist, lepattanó, csere, fault). Forrás: `source='pbp'` / `'merged'`.

Az importot a `mkosz-stats/daily-import.yml` CI futtatja minden nap 06:30 UTC-kor.

**A séma egy klasszikus star schema**: minden tény-tábla a `matches.gamecode`-on lóg. Ezért a `gamecode` szinte minden query JOIN-kulcsa.

## Tábla-térkép (használati színkód)

- 🟢 **HASZNÁLT** — közvetlenül lekérdezi a `generate_dashboards.py`
- ⚪ **NEM HASZNÁLT** — a weblap nem joinol rá (más projekt használhatja, pl. mkosz-scout)
- 🔴 **ÜRES** — fizikailag nincs benne adat (helyettesítő logika a kódban)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    KERET-TÁBLÁK                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ⚪ seasons (1)                  ⚪ competitions (17)                        │
│   ┌──────────────┐                ┌───────────────────────────┐             │
│   │ season_code  │◄───────────────│ comp_code + season (PK)   │             │
│   │ label        │                │ comp_name, level, gender  │             │
│   └──────────────┘                └───────────────────────────┘             │
│   nem kérjük le —                 nem kérjük le — a TEAMS dict-be           │
│   konstansok a kódban             hard-coded a comp_code+szezon             │
│                                                                             │
│   🔴 teams (0!)    🔴 team_aliases (0!)    ⚪ players (453)                   │
│   ┌──────────┐     ┌──────────────┐        ┌─────────────────────┐          │
│   │ üres     │     │ üres         │        │ playercode, license │          │
│   └──────────┘     └──────────────┘        │ canonical_name, ... │          │
│                                            └─────────────────────┘          │
│   csapatok névként                         a generátor NEM joinol           │
│   élnek matches-ben                        a players-re                     │
│                                                                             │
│   ⚪ player_names (845)            🔴 rosters (0!)                           │
│      név-variánsok                    üres — a roster a player_game_       │
│      (nem használjuk)                 stats-ből származik                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                    MECCS — A WEBLAP MOTORJA                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   🟢🟢🟢  matches (1329)  ←── a generátor szíve                              │
│   ┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓                         │
│   ┃ gamecode (PK)                                ┃                          │
│   ┃ comp_code + season       (← WHERE szűrő)     ┃                          │
│   ┃ team_a_name / team_b_name (← LIKE pattern)   ┃ ← csapatfelismerés:      │
│   ┃ score_a / score_b                            ┃   _team_like()           │
│   ┃ match_date, venue                            ┃                          │
│   ┃ has_pbp                  (← ág-választás)    ┃                          │
│   ┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛                         │
│                          ▲                                                  │
└──────────────────────────┼──────────────────────────────────────────────────┘
                           │ gamecode FK
   ┌───────────────────────┼─────────────────────────────────────────────┐
   │                       │                                             │
   ▼                       ▼                                             ▼
🟢🟢🟢                  🟢                                              ⚪
┏━━━━━━━━━━━━━━━━━━━━┓ ┏━━━━━━━━━━━━━━━━━━━━━┓                ┌──────────────────┐
┃ player_game_stats  ┃ ┃ quarter_scores      ┃                │ shots (46k)      │
┃ (32 870 sor)       ┃ ┃ (5842)              ┃                │ shotchart pontok │
┃ ────────────────   ┃ ┃ Q1..OT × A/B        ┃                │                  │
┃ MINDEN player-stat ┃ ┃                     ┃                │ nem használja a  │
┃ innen jön:         ┃ ┃ neg-bontás meccs-   ┃                │ weblap (csak a   │
┃ • box score        ┃ ┃ oldalon + szezon-   ┃                │ mkosz-scout PDF) │
┃ • starter ×        ┃ ┃ aggregátum          ┃                └──────────────────┘
┃ • T/U badge        ┃ ┗━━━━━━━━━━━━━━━━━━━━━┛
┃ • PPG, FG%, FT%    ┃
┃ • assists/tov/+-   ┃
┃   (ha PBP megvan)  ┃
┗━━━━━━━━━━━━━━━━━━━━┛

   ┌──────────────────────┬──────────────────────┬─────────────────────────┐
   ▼                      ▼                      ▼                         ▼
🟢🟢                    🟢                      ⚪                         🟢
┏━━━━━━━━━━━━━━━━━━┓ ┏━━━━━━━━━━━━━━━━━━┓ ┌──────────────────┐ ┏━━━━━━━━━━━━━━━━━━━┓
┃ scoring_events   ┃ ┃ pbp_events       ┃ │ substitutions    │ ┃ personal_fouls    ┃
┃ (103 823)        ┃ ┃ (225 221)        ┃ │ (23 455)         │ ┃ (33 307)          ┃
┃ ─────────────    ┃ ┃ ─────────────    ┃ │ ─────────────    │ ┃ ─────────────     ┃
┃ MECCS-OLDAL:     ┃ ┃ csak ahol        ┃ │ NEM használja    │ ┃ • foul timeline   ┃
┃ • step chart     ┃ ┃ has_pbp=1:       ┃ │ a weblap         │ ┃ • T/U badge       ┃
┃ • progression    ┃ ┃ • opp PPG        ┃ │ (mkosz-scout     │ ┃ • team_fouls      ┃
┃ • Q-MVP-k        ┃ ┃ • quarter stats  ┃ │  használja:      │ ┃ • foul_category   ┃
┃ • point-split    ┃ ┃ • PBP-alapú      ┃ │  rotation,       │ ┃   parser fix      ┃
┃ • leghosszabb    ┃ ┃   team-stats     ┃ │  lineup)         │ ┃                   ┃
┃   run            ┃ ┃                  ┃ │                  │ ┃                   ┃
┃ • záró 5 perc    ┃ ┃ NB2-ben gyakran  ┃ │                  │ ┃                   ┃
┃ • vezetés-váltás ┃ ┃ üres → fallback  ┃ │                  │ ┃                   ┃
┗━━━━━━━━━━━━━━━━━━┛ ┗━━━━━━━━━━━━━━━━━━┛ └──────────────────┘ ┗━━━━━━━━━━━━━━━━━━━┛

                          🟢
                          ┏━━━━━━━━━━━━━━━━━━┓
                          ┃ timeouts (7726)  ┃
                          ┃ ─────────────    ┃
                          ┃ Fun fact csempe: ┃
                          ┃ időkérések száma ┃
                          ┗━━━━━━━━━━━━━━━━━━┛
```

## Sor-számok (2026-05-13)

| Tábla | Sorok | Állapot |
|---|---:|---|
| `seasons` | 1 | ⚪ nem használt |
| `competitions` | 17 | ⚪ nem használt |
| `teams` | **0** | 🔴 üres |
| `team_aliases` | **0** | 🔴 üres |
| `players` | 453 | ⚪ nem használt |
| `player_names` | 845 | ⚪ nem használt |
| `rosters` | **0** | 🔴 üres |
| `matches` | **1 329** | 🟢 HASZNÁLT |
| `player_game_stats` | **32 870** | 🟢 HASZNÁLT |
| `scoring_events` | **103 823** | 🟢 HASZNÁLT |
| `pbp_events` | **225 221** | 🟢 HASZNÁLT (`has_pbp=1` esetén) |
| `substitutions` | 23 455 | ⚪ nem használt |
| `timeouts` | **7 726** | 🟢 HASZNÁLT |
| `personal_fouls` | **33 307** | 🟢 HASZNÁLT |
| `shots` | 46 790 | ⚪ nem használt |
| `quarter_scores` | **5 842** | 🟢 HASZNÁLT |

**Weblap használ: 7 tábla a 16-ból** (~44%).

## Használati táblázat

| Tábla | Weblap? | Hol használjuk |
|---|---|---|
| `matches` | ✅ | minden — comp+szezon szűrés, score, dátum, venue, has_pbp ág-választás |
| `player_game_stats` | ✅ | box score, szezon-aggregátum, starter ×, T/U, PPG, FG%, FT%, assist/tov/+- |
| `scoring_events` | ✅ | meccs-oldal step chart, progression, Q-MVP-k, point-split, leghosszabb run, záró 5 perc, vezetés-váltás |
| `pbp_events` | ✅ (csak `has_pbp=1`) | opp PPG, PBP-alapú team_stats, quarter stats |
| `personal_fouls` | ✅ | foul timeline, T/U badge, team_fouls aggregátum |
| `timeouts` | ✅ | meccs-oldal "Időkérések" csempe |
| `quarter_scores` | ✅ | meccs-oldal Q1-Q4 táblázat |
| `players` / `player_names` | ❌ | a `player_name` közvetlenül a `player_game_stats`-ből jön |
| `competitions` / `seasons` | ❌ | konstansok a `TEAMS` dict-ben (`generate_dashboards.py`) |
| `substitutions` | ❌ | csak a `mkosz-scout` használja (rotation/lineup) |
| `shots` | ❌ | csak a `mkosz-scout` PDF (shotchart) |
| `teams` / `team_aliases` / `rosters` | ❌ | üresek — helyettesítő logika a kódban |

## Mezőszintű részletek a 7 használt tábláról

### `matches` (a központi entitás)

| Mező | Típus | Mire használjuk |
|---|---|---|
| `gamecode` | TEXT PK | Minden join kulcsa; URL séma: `meccs/{gamecode}.html` |
| `comp_code` + `season` | TEXT | WHERE szűrő: `comp_code IN (cfg.comp_code, *mkosz_extra_comps)` |
| `round_name` | TEXT | Naptár címke |
| `match_date`, `match_time` | TEXT | Hero, naptár, game log |
| `venue` | TEXT | Meccs-oldal hero |
| `team_a_name`, `team_b_name` | TEXT | **Csapatfelismerés** `_team_like()` LIKE pattern-nel (mert `teams` üres) |
| `team_a_id`, `team_b_id` | INT | Gyakran NULL, nem támaszkodunk rá |
| `score_a`, `score_b` | INT | Eredmény, GYŐZELEM/VERESÉG badge, +/- különbség |
| `quarter_scores` | TEXT | Régi mező, ma a `quarter_scores` táblát használjuk helyette |
| `has_scoresheet`, `has_pbp`, `has_shotchart` | INT | **Ág-választás**: `has_pbp=1` esetén PBP-alapú stat, különben scoresheet only |
| `scoresheet_match_id` | TEXT | `WEB-{pdf_id}` jelöli a web fallback meccseket |

### `player_game_stats` (egy sor = egy játékos × egy meccs)

| Mező | Forrás | Megjegyzés |
|---|---|---|
| `gamecode`, `team` (A/B), `player_name` | scoresheet | PK |
| `playercode` | scoresheet/pbp | Lehet NULL — a `player_name` az elsődleges azonosító a weblapnál |
| `license_number`, `jersey_number` | scoresheet | Mezszám a player gridhez + foul timeline-hoz |
| `is_starter`, `entry_quarter` | scoresheet | Meccs-oldal `×` jel a kezdő 5-höz; NB2 starter detection |
| `points`, `fg2_made/attempted`, `fg3_made/attempted`, `ft_made/attempted` | scoresheet | Box score |
| `personal_fouls` | scoresheet | PF oszlop a box score-ban |
| `minutes`, `oreb`, `dreb`, `assists`, `steals`, `turnovers`, `blocks`, `fouls_drawn`, `plus_minus` | PBP | Csak ha `has_pbp=1`; különben NULL |
| `val`, `ts_pct`, `efg_pct`, `game_score`, `usg_pct`, `ast_to`, `tov_pct` | computed | Származtatott |
| `source` | meta | `'scoresheet'` / `'pbp'` / `'merged'` |

### `scoring_events` (event-by-event scoring)

| Mező | Megjegyzés |
|---|---|
| `gamecode`, `event_seq` | UNIQUE — sorrendet az `event_seq` adja |
| `quarter` (INT) | 1-4, OT=5+ |
| `minute` (TEXT) | **CSAK ~38%-ban kitöltött** — forward-fill kell a meccs-oldal logikájában (utolsó ismert perc öröklődik a következő eventre) |
| `team` (A/B) | A/B csapat |
| `playercode`, `license_number`, `jersey_number` | Lövő azonosító (jersey a backup) |
| `points` | 1, 2, vagy 3 |
| `shot_type` | `'2FG'`, `'3FG'`, `'FT'`, `'MULTI'` |
| `made` (0/1) | Kosár vagy hibás (HIBÁS FT is bekerül) |
| `score_a`, `score_b` | Pillanatnyi állás → step chart Y-érték |

### `pbp_events` (full play-by-play)

| Mező | Megjegyzés |
|---|---|
| `gamecode`, `event_seq` | UNIQUE |
| `quarter`, `minute` (INT) | A `minute` itt rendesen kitöltött (PBP forrás) |
| `team` (A/B) | |
| `playercode`, `player_name` | |
| `event_type` | pl. lövés, asszist, lepattanó, lopás, blokk, fault, sub, timeout |
| `event_raw` | Eredeti MKOSZ string |
| `is_scoring`, `points` | Pontszerző eventek megjelölése |
| `score_a`, `score_b` | Pillanatnyi állás |

**Fontos**: NB2-ben (`hun3*`) gyakran nincs PBP → a `has_pbp` flag 0, és a weblap a scoresheet ágra fallbackel.

### `personal_fouls` (faultkategóriás)

| Mező | Megjegyzés |
|---|---|
| `gamecode`, `team`, `playercode`, `jersey_number` | |
| `foul_number` | Sorrend a meccsen belül játékosra |
| `quarter`, `minute` | Foul timeline pozíciójához |
| `foul_type` | `'defensive'` (default) / `'offensive'` |
| `foul_category` | **T** (technikai), **U** (sportszerűtlen), **B/C/D** (kombinált) — a 2026-05-i parser-fix után pontos |
| `free_throws` | A faultból járó FT-k száma |
| `offsetting` | Ha kiütik egymást |

### `timeouts`

| Mező | Megjegyzés |
|---|---|
| `gamecode`, `quarter`, `minute`, `team` | |
| `source` | `'scoresheet'` / `'pbp'` |

Egy sor = egy időkérés. Meccs-oldal Fun fact csempén csapatonként összesítjük.

### `quarter_scores`

| Mező | Megjegyzés |
|---|---|
| `gamecode`, `quarter` (TEXT: `'1'`-`'4'`, `'OT1'`...) | PK |
| `score_a`, `score_b` | Negyed-pont (NEM kumulált — épp annak a negyednek az értéke) |

## Három adatszint — liga szerint hierarchikusan bővülő

Az MKOSZ liga-szintenként más-más adatforrást ad. Minden új feature tervezésekor előbb el kell dönteni, hogy melyik szinttől felfelé működik. Magasabb szint mindig tartalmazza az alacsonyabbak adatait.

| Szint | Liga / comp_code | Jegyzőkönyv | PBP | Shotchart | Új mit kapunk |
|---|---|:-:|:-:|:-:|---|
| **1. Jegyzőkönyv-szint** | `hun3*` (NB2, Közgáz A/B), `hun_bud_*` / `whun_bud_*` (Bp. megyei, Női + Leftoverz) | ✅ | ❌ | ❌ | alap: pts/FGM/FTM/FTA/PF, scoring_events (csak `made=1`!), quarter_scores, faultok (T/U/B/C/D), timeouts. Bp. megyei képes PDF-eknél web fallback (`WEB-*` match-id) — ott csak box score + quarter, scoring_events nélkül. |
| **2. + PBP** | `hun_univ*`, `whun_univ*` (MEFOB, Közgáz MEFOB ffi/női) | ✅ | ✅ | ❌ | + FGA/3PA → eFG%/TS%/PPS, assists, tov, reb (oreb/dreb), steals, blocks, fouls_drawn, plus_minus, percek, substitutions, possessions/Pace, OffRtg/DefRtg |
| **3. + Shotchart** | `hun2a`, `hun2b` (NB1B, Közgáz nem játszik itt) | ✅ | ✅ | ✅ | + `shots` tábla (x/y koordináták, zone) → lövéstérkép, hot zones |

**Adatforrás per tier:**
- **Jegyzőkönyv-szint** adatforrásai: a hivatalos PDF (MKOSZ vagy megyei verzió) + mkosz.hu / megye.hunbasket.hu (standings, schedule scraping). NB2 és Bp. megyei adat-szempontból azonos — csak a beszerző URL és a PDF-elrendezés más.
- **+PBP** plusza: az MKOSZ event-list HTML scrape (`mkosz.hu/merkozes-esemenylista/`).
- **+Shotchart** plusza: `mkosz.hu/ajax/film.php` shotchart API.

**Mérési pont a megyei web fallback hatásáról (2026-05-13):** whun_bud_na 131/173 (76%), hun_bud_rkfb 100/119 (84%) meccs rendelkezik scoring_events-szel. A többi képes PDF → web fallback → csak box score.

**Forrás tier-ekenként:**

| Forrás | Tier 1 (jegyzőkönyv) | Tier 2 (+ PBP) | Tier 3 (+ shotchart) | (web fallback*) |
|---|:-:|:-:|:-:|:-:|
| `matches` | ✅ | ✅ | ✅ | ✅ |
| `player_game_stats` PTS/FGM/FTM/FTA/PF | ✅ | ✅ | ✅ | ✅ |
| `player_game_stats` FGA/3PA | ❌ | ✅ | ✅ | ❌ |
| `player_game_stats` ast/tov/reb/stl/blk/min/+- | ❌ | ✅ | ✅ | ❌ |
| `scoring_events` (csak made=1!) | ✅ | ✅ | ✅ | ❌ |
| `quarter_scores` | ✅ | ✅ | ✅ | ✅ |
| `personal_fouls` | ✅ | ✅ | ✅ | részleges |
| `timeouts` | ✅ | ✅ | ✅ | ❌ |
| `pbp_events` | ❌ | ✅ | ✅ | ❌ |
| `substitutions` | ❌ | ✅ | ✅ | ❌ |
| `shots` (shotchart) | ❌ | ❌ | ✅ | ❌ |

\* Web fallback: csak Bp. megyei képes PDF-eknél (`WEB-*` match-id) — egyébként a megyei is a teljes tier 1-et megkapja.

**A 6 Közgáz csapat adatszintje:**

| Csapat | Liga | Tier | Mire futtatható feature |
|---|---|---|---|
| Közgáz B | NB2 (`hun3k` + `hun3_plya`) | **1. Jegyzőkönyv** | tier 1 mutatók |
| Közgáz A | NB2 (`hun3kob` + `hun3_plya`) | **1. Jegyzőkönyv** | tier 1 mutatók |
| Közgáz Női | Bp. megyei (`whun_bud_na`) | **1. Jegyzőkönyv** | tier 1 (de 173-ból 42 web-fallback meccsen scoring_events nincs) |
| Leftoverz | Bp. megyei (`hun_bud_rkfb`) | **1. Jegyzőkönyv** | tier 1 (de 119-ből 19 web-fallback meccsen scoring_events nincs) |
| Közgáz MEFOB Női | MEFOB (`whun_univn`) | **2. + PBP** | + eFG%/TS%, advanced stats, Pace |
| Közgáz MEFOB Férfi | MEFOB (`hun_univn`) | **2. + PBP** | mint fent |

**Fontos**: a `scoring_events` még a magasabb tier-ekben is csak a beesett kosarakat tartalmazza — a `made=0` *kizárólag FT*-re van rögzítve (a parser így működik, mert a jegyzőkönyv csak az FT-kísérleteket lajstromozza explicit). Ezért az **FGA / 3PA** csakis a `player_game_stats.fg2_attempted` + `fg3_attempted` oszlopokból jön, ami **NB2-ben NULL**.

## Üres / nem használt táblák — miért és milyen pótlással?

### `teams` (0) + `team_aliases` (0) — üresek
A csapatok **szöveges nevekként** élnek a `matches.team_a_name` / `team_b_name` oszlopokban. A generátor a `_team_like(conn, cfg)` helperben épít LIKE pattern-eket (`'%KÖZGÁZ%'`, `'%kozgazdasagi%'`, stb.), és így találja meg egy csapat meccseit. Ez törékeny (lásd EBH-Salgótarján szponzorváltás), de a `teams` tábla nincs feltöltve.

### `rosters` (0) — üres
A "csapat keret" a `player_game_stats`-ből származik: aki játszott egy meccsen az A vagy B oldalon, az a roster tagja. A `get_roster(conn, cfg, tp)` ezt aggregálja jersey_number + player_name szerint.

### `players` / `player_names` — léteznek, de nem használjuk
A `player_name` mező közvetlenül a `player_game_stats`-ben van (a scoresheet-en szereplő név), így a generátor nem joinol a `players`-re. A `playercode` mező NULL is lehet régi importoknál, ezért megbízhatatlan elsődleges kulcs.

### `competitions` / `seasons` — léteznek, de nem használjuk
A `TEAMS` dict a `generate_dashboards.py`-ban hard-coded `comp_code` + `season` értékeket tartalmaz, ezért nem kell ezeket a táblákat lekérdezni futás közben.

### `substitutions` / `shots` — nem weblap-feature
A `mkosz-scout` (PDF scout riport) és a `mkosz-scout-web` projekt használja: rotation patterns, lineup net rating, shot chart. A weblap eddig nem hozott be ilyen feature-t.

## Hivatkozások

- Teljes projektkontextus: `CLAUDE.md` (ugyanezen mappában)
- Mezőszintű részletek (mkosz-stats oldalról): `mkosz-stats/README.md` + `mkosz-stats/CHANGELOG.md`
- DB létrehozó SQL: `mkosz-stats/mkosz_stats/schema.py` (vagy `sqlite3 mkosz_stats.sqlite ".schema"`)
