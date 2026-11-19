# Movie Title Matcher

A dbt + DuckDB pipeline that matches MovieLens movie titles to IMDb IDs, scored against MovieLens' own IMDb links as ground truth.

Matching titles across catalogs is harder than it looks. The two sources write the same film differently:

| MovieLens | IMDb |
| --- | --- |
| `Matrix, The (1999)` | `The Matrix` |
| `City of Lost Children, The (Cité des enfants perdus, La) (1995)` | `The City of Lost Children` / `La cité des enfants perdus` |
| `Auberge espagnole, L' (2002)` | `L'auberge espagnole` |
| `Jungle Book, The (1967)` | three films called `The Jungle Book` (1967, 1994, 2016) |

## Architecture

```
 SOURCES          STAGING                 INTERMEDIATE                   MARTS               EVALUATION
 MovieLens  ──▶  stg_movielens__movies ─┐
                 stg_movielens__links   │
                                        ├─▶ int_movielens__titles_parsed ─┐
 IMDb       ──▶  stg_imdb__titles ──────┤                                 ├─▶ int_title_match_candidates ─▶ fct_title_matches ─▶ eval_match_quality
                 stg_imdb__akas         └─▶ int_imdb__titles_keyed ───────┘
```

- **Staging:** renames and type casts only. MovieLens IMDb IDs are rebuilt into the `tt0114709` form.
- **Intermediate:** parses MovieLens titles (year, alternate title, trailing articles in 8 languages), normalizes both sides to a shared key, and generates candidate pairs by tier.
- **Marts:** keeps one best match per title, then enforces one title per IMDb ID so no film is mapped twice.
- **Evaluation:** precision, recall, and coverage per tier against ground truth.

### Matching tiers

| Tier | Rule | Status |
| --- | --- | --- |
| exact | Display title identical, same year | Done |
| normalized | Normalized title identical, year within ±1 | Done |
| alternate title | Match against IMDb alternate titles (`title.akas`) | Next |
| fuzzy | `jaro_winkler_similarity` above a threshold | Next |

## Testing

- **Data tests:** uniqueness and not-null on every ID, accepted values on match tier, referential integrity between sources, and one IMDb ID per title in the final mapping.
- **Unit tests:** dbt unit tests with mocked inputs check title parsing (articles, years, alternate titles) and the year window that separates remakes.
- **Quality gate:** `tests/assert_precision_above_threshold.sql` fails the build if overall precision drops below `min_precision` (default 0.95).

## CI

GitHub Actions runs `dbt build` on every pull request against small hand-made fixtures in `fixtures/` (real titles plus a few invented edge cases). CI never downloads IMDb's full files, which are large and licensed for personal, non-commercial use. Full runs happen locally.

## Run it

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
export DBT_PROFILES_DIR=.

# Quick check on fixtures (what CI runs)
python scripts/load_raw.py --fixtures
dbt build --target ci

# Full run on real data (downloads ~1 GB)
python scripts/load_raw.py
dbt build
```

See results:

```bash
python -c "import duckdb; print(duckdb.connect('data/movies.duckdb').sql('select * from eval_match_quality'))"
```

## Results

_Full-data results go here after the first real run._

## Data

- [MovieLens](https://grouplens.org/datasets/movielens/) `ml-latest-small`, GroupLens Research.
- [IMDb non-commercial datasets](https://developer.imdb.com/non-commercial-datasets/). Information courtesy of IMDb (https://www.imdb.com). Used with permission.
