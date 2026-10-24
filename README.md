# Box Score Lab: Fantasy Team Builder

Build a fantasy team from real NBA per-game stats. Six roster spots, a $110 salary cap, and the classic fantasy tension: stars and scrubs, or balanced value?

**v3 adds a live data pipeline:** the site no longer hard-codes its stats. It loads `data/players.json` on every visit, and a scheduled GitHub Action refreshes that file from stats.nba.com. See "How the data pipeline works" below.

## The game

- Roster: one PG, SG, SF, PF, and C, plus one FLEX (any position)
- Salary cap: $110. Every player has a salary set from his fantasy production
- Pick players from the pool, watch the cap, fill every slot, maximize projected fantasy points
- **Auto-build best team** runs an exhaustive search over every 6-player combination and finds the true optimum under the cap. It is not the six biggest names: at the time of writing the optimal team (297.6 projected points, exactly $110) leaves out the highest-scoring fantasy player entirely, because his salary buys more production spread across two slots
- **Copy my team** copies your six as text, ready to challenge a friend

## Scoring (site-defined, simplified fantasy points format)

Fantasy points per game are calculated from real per-game stats:

| Stat | Points |
| --- | --- |
| Points | 1 |
| Rebounds | 1.2 |
| Assists | 1.5 |
| Steals | 3 |
| Blocks | 3 |
| Turnovers | -1 |

Worked example: Nikola Jokić, 27.7 pts + 12.9 reb + 10.7 ast + 1.4 stl + 0.8 blk, minus 3.7 turnovers, equals 62.1 fantasy points per game, the highest in the pool. His $23 salary is the decision, not his production.

Position eligibility is site-defined and includes common secondary positions (for example Luka Dončić at PG/SG).

## What is in v2

- Fantasy team builder with live cap meter, roster slots, and a verdict on your projected total
- Player pool of 26 headline players, sortable by fantasy points, value (fantasy points per $), salary, or raw points, and searchable by player, team, or position
- Value column, because finding a $15 player who produces like a $20 player is the actual skill in fantasy
- Scoring explainer and glossary, in the spirit of Basketball-Reference

v1 (stats table, leader cards, head-to-head compare) is in the git history once this is committed to GitHub.

## How the data pipeline works

```
stats.nba.com  ->  scripts/update_stats.py (nba_api)  ->  data/players.json  ->  index.html
       ^                        ^
       |                        |
  official NBA stats     GitHub Action, daily 06:30 UTC
```

1. `scripts/update_stats.py` calls the `LeagueLeaders` per-game endpoint via the `nba_api` Python package, keeps rotation players (minutes and games-played filters that scale with how far the season has progressed), takes the top 30 scorers, resolves positions (a small override map for well-known multi-position players, the NBA's listed position for everyone else), and writes `data/players.json`.
2. `.github/workflows/update-stats.yml` runs that script on GitHub's servers every day, and commits `players.json` back to the repo only if it changed. Manual runs: repo Actions tab, "Update player stats", Run workflow. Change the season each October via the `SEASON` value in the workflow file.
3. The page fetches `data/players.json` on load and rebuilds the pool, salaries, and fantasy points from it. The header badge shows the season and the date the file was generated.

Failure design, because stats.nba.com is genuinely flaky from cloud IPs: the script retries each fetch, a failed run commits nothing, and if `players.json` cannot be loaded the page silently falls back to an embedded, hand-verified snapshot of the 2025-26 season (source: [Basketball-Reference](https://www.basketball-reference.com/leagues/NBA_2026_per_game.html), captured 24 Oct 2026). The shipped `players.json` is that same snapshot as a seed, until the Action's first successful run replaces it with the current season.

Fantasy points, salaries, and eligibility are this site's own layer on top of the real stats. Personal learning project. Not affiliated with Basketball-Reference, the NBA, or any fantasy platform.

## Run it

Open `index.html` in any browser, or:

```bash
python3 -m http.server 8000
```

## Put it on GitHub

For v3, use the terminal route (or GitHub Desktop), not the web upload page: the repo now contains a hidden `.github` folder that the website upload flow handles badly, and the Action only exists if that folder lands intact.

```bash
cd basketball-stats
git init
git add .
git commit -m "Box Score Lab v3: fantasy builder with live data pipeline"
gh repo create box-score-lab --public --source=. --push
```

Then:
1. Enable GitHub Pages (repo Settings: Pages: Deploy from branch `main`, folder `/root`).
2. Check the Actions tab: the "Update player stats" workflow should be listed. Run it once manually. If stats.nba.com blocks the runner that day, it fails without breaking anything; the daily schedule retries.
3. Open the Pages link: the header badge should read "Live data file: 2026-27 season, Updated (date)" once a run has succeeded, and "Snapshot mode" until then.

## Roadmap

- v2.1: A weekly matchup mode: your six vs a friend's six pasted in as text
- v2.2: Games-played risk flag (several stars in the pool played under 45 games in 2025-26, and the builder currently prices them as if they played 82)
- v3 idea: Season selector (2024-25 vs 2025-26) so prices move year to year

## Stack

Vanilla HTML, CSS, and JavaScript in one file, Argon-style dashboard cards. No framework, no build step, no API key.
