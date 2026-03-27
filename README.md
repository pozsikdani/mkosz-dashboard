# mkosz-dashboard

Interactive basketball dashboards for Kozgaz SC, hosted at **[www.kozgazkosar.hu](https://www.kozgazkosar.hu)**.

Generates HTML dashboards from SQLite databases produced by [mkosz-scoresheet](https://github.com/pozsikdani/mkosz-scoresheet) and [mkosz-play-by-play](https://github.com/pozsikdani/mkosz-play-by-play).

## Teams

| Folder | Team | League |
|---|---|---|
| `dashboards/` | Kozgaz B | NB2 Kelet |
| `dashboards-a/` | Kozgaz A | NB2 Kozep B |
| `dashboards-noi/` | Kozgaz Women | Noi A (Budapest) |
| `dashboards-mefob-ferfi/` | MEFOB Men | University West |
| `dashboards-mefob/` | MEFOB Women | University West |
| `leftoverz/` | Leftoverz | Regional (Budapest) |

## Usage

```bash
pip install PyMuPDF requests beautifulsoup4

# Generate all dashboards (needs nb2_full.sqlite + pbp.sqlite)
NB2_DB_PATH=../mkosz-scoresheet/nb2_full.sqlite \
PBP_DB_PATH=../mkosz-play-by-play/pbp.sqlite \
python3 generate_dashboards.py site

# Generate single team
python3 generate_dashboards.py kozgaz-b

# Local preview
python3 -m http.server 8765
```

## Automation

- **daily-update.yml** — 6:17 UTC: clones scoresheet + PBP repos, generates dashboards, commits
- **update-attendance.yml** — 6:00 UTC: fetches training attendance from Google Sheets, updates HTML
