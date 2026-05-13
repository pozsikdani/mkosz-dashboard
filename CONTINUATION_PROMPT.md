# Folytatási prompt új session-höz

Másold be ezt a szöveget egy új Claude Code session elejére:

---

Folytatom a kozgazkosar.hu weblap fejlesztését. A projekt teljes dokumentációja a `/Users/danipozsik/Desktop/claudecode/mkosz-dashboard/CLAUDE.md`-ben van, kérlek olvasd el először.

**Gyors összefoglaló:**
- Repo: `/Users/danipozsik/Desktop/claudecode/mkosz-dashboard` → www.kozgazkosar.hu (GitHub Pages)
- Generátor: `generate_dashboards.py` (~4700 sor) — `STATS_DB_PATH`=`../mkosz-stats/mkosz_stats.sqlite`-ot olvas
- 6 csapatra generál: kozgaz-b, kozgaz-a, kozgaz-noi, leftoverz, kozgaz-mefob, kozgaz-mefob-ferfi
- Daily CI: 7:00 UTC → Pages deploy (kozgazkosar.hu)
- Színrendszer: zöld=pozitív, piros=negatív, szürke=semleges, klub piros=branding
- Hazai/Vendég: vs/@ jelölés

**Friss állapot (2026-05-13):**
A meccs-szintű oldal proof-of-concept készen áll a Közgáz A legutolsó meccsére (`dashboards-a/meccs/hun3_plya_133583.html`). A csapat dashboard MECCSEK táblázat utolsó sora kattintható.

A meccs-oldal tartalma:
1. Hero (scoreline + GYŐZELEM/VERESÉG badge)
2. Eredmény alakulása (step chart Q-marker badge-ekkel)
3. Érdekességek & Fun facts (9 csempe vs-grid-ben)
4. Negyed MVP-k
5. Pontszerzés megoszlása (stacked bar 2PT/3PT/FT)
6. Vezetési idő (becslés, stacked bar)
7. Hibák alakulása (foul timeline)
8. Közgáz játékosok (box score + starter × + T/U + Összesen)

**Lehetséges következő lépések** (a CLAUDE.md "Kiterjesztési pontok" szakasza):
1. Minden Közgáz A meccsre meccs-oldal (most csak az utolsó)
2. Más csapatokra is meccs-oldal (Közgáz B, Női, stb.)
3. Játékos dashboard "Meccsenként részletezve" → klikk-elhetővé tétel a meccs-oldalra
4. Naptár cellákból link a meccs-oldalra
5. Új fejlett statisztikák (eFG%, TS%, pace, possessions) — FGA elérhetőség kérdéses

**Munkamódszer:**
- A `mkosz-dashboard/CLAUDE.md` mindent dokumentál — adatfolyamat, függvénystruktúra, CI, kiterjesztési pontok
- Új meccs-szintű feature esetén: `get_match_details()` → `generate_match_page()` → új CSS osztály
- A CI minden este 7 UTC-kor regenerálja a HTML-eket. Csak a `generate_dashboards.py`-t commit-old, a HTML majd regenerálódik (kivéve ha azonnal akarod látni)
- Tesztelés: `python3 generate_dashboards.py kozgaz-a` és `dashboards-a/meccs/{gc}.html` ellenőrzése Chrome-mal
- Git: ha a CI gyors volt és a remote előrement, `git pull --rebase` után push; konfliktusnál `git checkout --ours <fájl>`

**Mit szeretnék most:** [ITT ÍRD LE AMIT TENNI AKARSZ]

---

## Megjegyzések:
- Az MCP Chrome-tools elérhetőek (`mcp__Claude_in_Chrome__*`) — élesben tudod ellenőrizni a változásokat
- A `gh` CLI elérhető a `/opt/homebrew/bin/gh` útvonalon
- Auto mode beilleszthető az új session elejére ha autonóm végrehajtást szeretnél
