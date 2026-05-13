# MKOSZ Dashboard — Project Documentation (kozgazkosar.hu)

> Last updated: 2026-05-13. Ez a fájl teljes kontextust ad a `mkosz-dashboard` repóhoz.

## Mi ez a projekt?

A **kozgazkosar.hu** weboldal generátora. HTML-eket állít elő a központi `mkosz-stats/mkosz_stats.sqlite` adatbázisból, és GitHub Pages-en hostolja.

**Tulajdonos**: KÖZGÁZ SC ÉS DSK kosárlabda klub.

## Repo struktúra (külön repók 2026-04 óta)

```
mkosz-dashboard/         ← EZ A REPO (HTML gen + Pages)
mkosz-stats/             ← Központi DB: mkosz_stats.sqlite
mkosz-scoresheet/        ← PDF extractor → scoresheet.sqlite
mkosz-play-by-play/      ← PBP scraper → pbp.sqlite
mkosz-scout/             ← PDF scout riport (külön projekt)
mkosz-scout-web/         ← Interaktív scout dashboard (külön projekt)
```

## Repo & hosting
- **GitHub**: `pozsikdani/mkosz-dashboard`
- **Pages**: `www.kozgazkosar.hu` (CNAME)
- **Adatforrás**: `STATS_DB_PATH` env → `../mkosz-stats/mkosz_stats.sqlite`
- **gh CLI**: `/opt/homebrew/bin/gh`
- **Nyelv**: Python 3 + HTML/JS (Chart.js)

## Daily CI Pipeline (UTC)
| Időpont | Mit | Hol |
|---|---|---|
| 06:00 | Scoresheet PDF letöltés + extract | mkosz-scoresheet |
| 06:00 | PBP scrape | mkosz-play-by-play |
| 06:00 | Edzéslátogatás (Google Sheets → HTML in-place) | mkosz-dashboard/update-attendance.yml |
| 06:30 | DB import: scoresheet + pbp + web → mkosz_stats.sqlite | mkosz-stats/daily-import.yml |
| 07:00 | Dashboard regen + Pages deploy | mkosz-dashboard/daily-update.yml |

**Manuális trigger** (ha a CI nem találja meg az új meccset):
```bash
gh workflow run "Napi scoresheet feldolgozás" --repo pozsikdani/mkosz-scoresheet
# várj ~5 perc
gh workflow run "Napi import" --repo pozsikdani/mkosz-stats   # ~12-13 perc!
# várj
gh workflow run "Napi dashboard frissítés" --repo pozsikdani/mkosz-dashboard
```

## Csapatok (TEAMS dict)
| Kulcs | comp_code | mkosz_extra_comps | output | szín |
|---|---|---|---|---|
| `kozgaz-b` | hun3k | hun3_plya | dashboards/ | `#C41E3A` piros |
| `kozgaz-a` | hun3kob | hun3_plya | dashboards-a/ | `#e17055` narancs |
| `kozgaz-noi` | whun_bud_na | — | dashboards-noi/ | `#6c5ce7` lila |
| `leftoverz` | hun_bud_rkfb | — | leftoverz/ | `#fdcb6e` sárga |
| `kozgaz-mefob` | whun_univn | — | dashboards-mefob/ | `#00cec9` teal |
| `kozgaz-mefob-ferfi` | hun_univn | — | dashboards-mefob-ferfi/ | `#a0a0b0` szürke |

A `mkosz_extra_comps` listája extra `comp_code`-okat ad a stat query-khez (pl. NB2 alapszakasz + rájátszás összevontan).

## Generálás
```bash
python3 generate_dashboards.py site             # teljes site (mind a 6 csapat + főoldal)
python3 generate_dashboards.py kozgaz-a         # csak Közgáz A
python3 generate_dashboards.py kozgaz-b
python3 update_attendance.py                    # csak edzéslátogatás (no SQLite)
```

## Site struktúra (kozgazkosar.hu)
```
/                              — Klub főoldal (hero, csapat-kártyák, MECCSEK, MECCSNAPTÁR)
/dashboards/                   — Közgáz B (Öregek NB2)
  index.html                   — Játékos grid mezszámokkal
  csapat.html                  — Csapat dashboard (3 view toggle: All/Home/Away)
  naptar.html                  — Meccsnaptár (havi nézet)
  {player-slug}.html × 18      — Egyéni játékos dashboard
  meccs/{gamecode}.html × 20   — Meccs-szintű oldalak (teljes szezon)
/dashboards-a/                 — Közgáz A (Fiatalok NB2)
  (ugyanaz a struktúra mint dashboards/)
  meccs/{gamecode}.html × 21   — Meccs-szintű oldalak (teljes szezon)
/dashboards-noi/               — Női A
/dashboards-mefob/             — MEFOB Női
/dashboards-mefob-ferfi/       — MEFOB Férfi
/leftoverz/                    — Leftoverz (Budapest Reg. Kiemelt)
```

## Navigation
- `NAV_TEAMS`: Öregek NB2 / Fiatalok NB2 / Női / Leftoverz / MEFOB Női / MEFOB Férfi
- `_nav_html(active_key, depth)` — depth=0 root, 1=team subdir, 2=match subpage
- Favicon: `/kozgaz_logo.png` (root-abszolút), minden head-ben

## Színrendszer (szemantikus, 2026-04 óta)
- **Pozitív** = `--green` `#00b894` (Csúcs pontszám, Csapat részesedés, Erősségek, top scorer kiemelés, Közgáz vonal a chart-okon)
- **Negatív** = `--red` `#e17055` (Fault/meccs, Fejlesztendő területek, T/U badge, vereség)
- **Semleges** = `--text-dim` `#8b8da0` és árnyalatok (`#c0c2ce` világos, `#5a5c6a` sötét, kategorikus adatok)
- **Klub piros** = `--accent` `#C41E3A` — csak branding (hero gradient, header glow, nav logo)

A dobástípus-eloszlás (3FG/2FG/FT) három szürke árnyalat. A pontszerzés-megoszlás (2PT/3PT/FT) lila/narancs/szürkéskék.

## H/V → vs/@ (2026-04)
A hazai/vendég badge minden helyen **vs** (hazai) és **@** (vendég), fehér színnel, badge nélkül.

## Mobile responsive
- 600px alatt: kompaktabb táblázatok, scrollozható game-log, mobil-barát match-row layout
- Táblázatok `.game-log-wrap` overflow-x:auto wrapperben

## Calendar / Scraping
- `scrape_schedule(cfg)` — országos: `mkosz.hu/bajnoksag-musor/{season}/{comp}/phase/0/csapat/{team_id}`
- `scrape_schedule_county(cfg)` — megyei: `megye.hunbasket.hu/{county}/bajnoksag-musor/...`
- `CALENDAR_CSS` + `_build_calendar_grid()` — közös calendar generator (homepage + per-team naptar.html)
- Hungarian dátum: `HU_MONTHS_PARSE` dict
- Rövid nevek: `CALENDAR_SHORT` dict + `calendar_short_name()`
- Lejátszott: zöld W / piros L; jövőbeli: szürke
- Fallback: `get_calendar_data_db()` ha scraping fail

## Edzéslátogatás (Közgáz B only)
- Google Sheets CSV fetch a `fetch_training_attendance()`-ben
- Spreadsheet: `1CY9OV_JY4C5uzTcA621zs0-rd5gPO7ELau5yvrSsA-0`, gid `1405052111`
- `ATTENDANCE_NAME_MAP` dict mappel becenév → DB név
- Megjelenítés: teal "Edzés" stat headerben + 🏋️ row az index oldalon
- Automatikus napi frissítés `.github/workflows/update-attendance.yml`

## NB2 Rájátszás (hun3_plya)
- TEAMS dict `mkosz_extra_comps: ["hun3_plya"]` Közgáz A/B-nél
- `_comp_list(cfg)` helper: visszaad `[comp_code, *mkosz_extra_comps]`-ot
- Stat query-k mind `comp_code IN ({_cph})` placeholder list-tel
- "2025/26 alapszakasz" → "2025/26 szezon" feliratok
- Pipeline: `mkosz-scoresheet/ci_update.py` SCORESHEET_COMPS bővítve `hun3_plya`-val; `mkosz-stats/config.py` COMPETITIONS-be bevezetve

## Web fallback képes PDF-ekhez (2026-04)
- Ha az `extract_scoresheet.py` 0 karaktert talál egy PDF-ben (raszterkép), a `scrape_match_web.py` lekéri a meccs adatait a `megye.hunbasket.hu`-ról
- Match-id: `WEB-{pdf_id}`
- Kinyer: csapatok, eredmény, dátum, **player box score** (név, license, pont, 2FG, 3FG, FT), **negyedenkénti bontás**
- Csak megyei (`hun_bud_*`, `whun_bud_*`) bajnokságokra
- ~45 korábban hiányzó meccs visszanyerve (Leftoverz, MEFOB Női, stb.)

## Foul-category parser fix (2026-05)
- T/U/B/C/D személyes faultkategória detekció javítva `extract_scoresheet.py`-ban
- Két fix: (1) row y-tartomány +12px lefelé (subscript U/T), (2) fallback main_letters-ből ha minden char ugyanaz a méret
- Force re-extract `hun3_plya` szezonra futtatva → mostantól T/U badge-ek megjelennek a player dashboard és meccs-oldal foul oszlopában

## VACUUM in mkosz-stats CI (2026-04)
- A `mkosz_stats.sqlite` 100MB GitHub limitet túllépett, a daily-import 12 napig sikertelen volt
- Fix: a `daily-import.yml`-be VACUUM lépés import után, push előtt
- Méret most ~56MB, stabil

## Match-level page (2026-05)
- **Közgáz A** (21 meccs) és **Közgáz B** (20 meccs) összes lejátszott meccsére generálódik
- URL séma: `{out_dir}/meccs/{gamecode}.html` (pl. `dashboards-a/meccs/hun3kob_125843.html`)
- Link: a `csapat.html` MECCSEK táblázat minden sora kattintható (JS `onclick`, `MATCH_URLS` dict)

### Új query helper: `get_match_details(conn, cfg, tp, gamecode)`
Visszaad egy dict-et: match info, quarters, **progression** (scoring_events made=1, minute, player, shot, points), kg_players + opp_players + opp_total, **timeouts**, **team_fouls** (personal_fouls-ból aggregálva), **foul_events** (jersey + quarter + minute).

### Új HTML generator: `generate_match_page(details, cfg, team_key)`
Strukúra:
1. **Hero**: scoreline + GYŐZELEM/VERESÉG badge + dátum + helyszín + +különbség
2. **Eredmény alakulása** (step chart):
   - Chart.js line `stepped:'after'`, két csapat
   - Custom plugin `quarterMarkerPlugin` — Q1/Q2/Q3 végén függőleges szaggatott vonal + szürke badge `Q1 vége · 15–15`
   - Q1-Q4 markers az x-tengelyen
   - Tooltip: `Q3 · Közgáz A · BÉRES MÁRTON (2p kétpontos)` az event_info dict-ből
   - Custom HTML legend (`chart-legend` + `legend-item`) — kattintással toggle, default Chart.js legend kikapcsolva
   - Sorrend: **hazai elöl, vendég hátul** (egyezik a quarter table és hero sorrenddel)
   - Quarter breakdown table közvetlenül a chart alatt (Bkg/Közgáz × Q1-Q4 + Σ)
3. **Érdekességek & Fun facts** (vs-grid):
   - 7 vs-card (összehasonlító): Legnagyobb vezetés, Leghosszabb run, Negyedek megnyerve, Faultok csapatonként, Időkérések, Záró 5 perc, Kipontozott (csak ha legalább egyiknek van)
   - 1 sb-card (kombinált): Kezdő / Pad pontok (4 érték egy kártyán)
   - 1 single-stat: Vezetés-váltás
4. **Negyed MVP-k**: top scorer per quarter mindkét csapatban (4 kis kártya)
5. **Pontszerzés megoszlása**: 2 stacked bar (lila/narancs/kékesszürke) — 2PT/3PT/FT % csapatonként
6. **Vezetési idő** (becslés): stacked bar Bkg/Közgáz/döntetlen, perc-alapú interpolációval (~5-15% pontatlanság)
7. **Hibák alakulása**: Közgáz játékosok foul-timeline-ja, Q1-Q4 marker-ekkel, hover-tooltip
8. **Közgáz játékosok** (box score):
   - Új oszlop: starter mark (`×` zöld a kezdő 5-nél)
   - Új oszlop: T/U badge a PF után (technikai/sportszerűtlen)
   - **Összesen sor** alul: pont/FG/FT(%)/PF/T/U összegek

### Kattinthatóság (csapat → meccs)
- `get_team_stats()` `game_log` query visszaadja a `gamecode`-ot is
- `_stats_to_js()` belekerül `"gamecode": gc`
- `generate_team_dashboard()` `match_urls` paraméter — JSON-ként injektálva `MATCH_URLS` változóként
- `renderGameLog()` JS — ha `MATCH_URLS[g.gamecode]` létezik, `tr.onclick = ...`, `cursor:pointer`, hover-piros háttér

### Kiterjesztési pontok (a jövőre)
1. ~~Minden lejátszott meccsre Közgáz A-nál~~ ✅ Kész (2026-05-13)
2. ~~Másik csapatokra is~~ ✅ Közgáz B is kész (2026-05-13)
3. **Játékos dashboard "Meccsenként részletezve"** szintén linkelhetővé
4. **Naptár** cellákról is link a meccs-oldalra
5. **Pace, eFG%, possessions** statok hozzáadása (FGA elérhetőség kérdéses, scoring_events-ből kinyerhető)
6. **További csapatok** (női, MEFOB, Leftoverz) — scoring_events adat szükséges hozzá

## generate_dashboards.py architektúra (kb. 4700 sor)
```
TEAMS dict                                  — 6 csapat config (comp_code, color, mkosz_extra_comps)
LEAGUES dict                                — 3 liga (nb2, budapesti, mefob) színek
NAV_TEAMS / _nav_html(depth)                — navigáció
NAV_CSS                                     — nav CSS

# Query függvények (mind comp_list IN (?,?,...) placeholder-ekkel)
_team_like(conn, cfg)                       — broad/specific LIKE pattern
get_roster(conn, cfg, tp)
get_game_log(conn, cfg, tp, player_id)
get_quarter_stats / _pbp
get_opponent_ppg / _pbp
get_tech_unsport
get_team_stats / _pbp                       — 9 SQL query, hv_filter támogatás
get_calendar_data_db                        — fallback ha scraping fail
get_match_details(conn, cfg, tp, gamecode)  — meccs-oldal data

# Generator függvények
generate_html()                             — egyéni játékos dashboard
generate_team_dashboard()                   — csapat dashboard (3 view toggle, match_urls)
generate_match_page(details, cfg, team_key) — meccs-oldal
generate_calendar(matches, cfg)             — havi naptár
generate_index(generated, cfg)              — csapat áttekintő (player grid)
generate_homepage(team_summaries)           — klub főoldal
generate_team(team_key)                     — orchestrate (player×N + team + match + calendar + index)
generate_site()                             — minden csapat + homepage

# Helper
_comp_list(cfg)                             — [comp_code, *mkosz_extra_comps]
_team_color_cfg(hex)                        — RGB → {color, bg, border}
_stats_to_js(d)                             — Python stats dict → JSON-serializable
_calc_lead_times(progression)               — vezetési-idő becslés
_calc_dist(fg2, fg3, ft)                    — pontmegoszlás
CALENDAR_CSS + _build_calendar_grid()       — közös calendar HTML+JS
shorten_opponent(name)                      — name → rövid forma
```

## Útmutató új meccs-szintű feature-ökre
1. **Adat**: bővítsd a `get_match_details()` SQL-jeit (új tábla / mező → új dict-kulcs)
2. **Számítás**: helper függvény közvetlenül a `generate_match_page()` elején (pl. `_calc_lead_times`, `_compute_fun_facts`)
3. **HTML**: új card a megfelelő helyre a meccs-oldal layout-jában
4. **CSS**: új class a `style` blokkban (a `generate_match_page()` saját stílusokkal)
5. **Test**: `python3 generate_dashboards.py kozgaz-a` és a `dashboards-a/meccs/{gc}.html` ellenőrzése

## Ismert korlátok / TODO
1. **Web fallback nem fedi az országos képes PDF-eket** (`hun3*`, csak `*_bud_*`)
2. **Lead time becslés** — `scoring_events.minute` mező csak ~38%-ban kitöltött, forward-fill interpolációval
3. ~~Meccs-oldal csak Közgáz A utolsó meccsére~~ ✅ Közgáz A (21) + B (20) összes meccsre kész
4. **EBH-Salgótarján alias** — szponzor-váltás mid-season, alias map nincs (low priority)
5. **README.md elavult** — a README csak az alapokat tartalmazza

## Git workflow
```bash
# Csak kód módosítás (a CI majd a HTML-eket regenerálja):
git add generate_dashboards.py
git commit -m "feat: ..."
git push

# Kód + helyileg regenerált HTML (pl. ha akarod élesben látni azonnal):
python3 generate_dashboards.py kozgaz-a
git add generate_dashboards.py dashboards-a/
git commit -m "..."
git push

# Ha a remote előrement (CI gyors volt):
git pull --rebase
# konfliktus esetén: git checkout --ours <file>
git push
```

## Aktuális állapot (2026-05-13)
- ✅ Match-level page: Közgáz A (21 meccs) + Közgáz B (20 meccs) — teljes szezon lefedve
- ✅ 9 csempés "Érdekességek & Fun facts" szekció
- ✅ Step chart custom plugin Q-marker badge-ekkel
- ✅ Player box score: starter ×, T/U, totals
- ✅ Web fallback teljes körű (match info + box score + quarters)
- ✅ Foul-category parser fix → T/U badge megjelenik mindenhol
- ✅ Szemantikus színrendszer kiterjesztve
- ✅ NB2 playoff (hun3_plya) merge alapszakaszba
- ✅ Mobile responsive
- ✅ vs/@ egységesen
- ✅ Favicon mindenhol
- ✅ VACUUM a daily-import CI-ban
