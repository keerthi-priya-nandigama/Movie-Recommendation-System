# CineMind AI — Phase 1: Dataset & Database

## What's in this phase
- Full SQLAlchemy schema (25 tables) matching the architecture doc, with Alembic migrations.
- `scripts/ingest_tmdb.py` — real TMDB ingestion (metadata, cast, crew, genres, keywords, collections). **Run this yourself** — this sandbox can't reach `api.themoviedb.org`.
- `scripts/ingest_movielens.py` — real MovieLens ratings ingestion (the user×movie interaction matrix the CF model trains on). **Run this yourself** — this sandbox can't reach `grouplens.org`.
- `scripts/generate_sample_dataset.py` + `scripts/seed_sample_data.py` — a bundled 20-movie sample dataset, shaped exactly like real TMDB/MovieLens responses, that exercises the *exact same* `upsert_movie()` / `ingest_ratings()` code as the real scripts. This is how the pipeline was verified end-to-end in this sandbox (see `tests/test_ingestion.py` — 7/7 passing).

## Quickstart (sample data, no API key needed)
```bash
cd backend
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env

python -m alembic upgrade head
python -m scripts.generate_sample_dataset
python -m scripts.seed_sample_data

pytest tests/test_ingestion.py -v
```

## Running against real data
1. **TMDB**: get a free API key at https://www.themoviedb.org/settings/api, put it in `.env` as `TMDB_API_KEY`, then:
   ```bash
   python -m scripts.ingest_tmdb --pages 25      # ~500 popular movies
   python -m scripts.ingest_tmdb --movie-id 27205  # or a specific movie
   ```
2. **MovieLens**: download `ml-latest-small.zip` (dev) or `ml-25m.zip` (full eval) from https://grouplens.org/datasets/movielens/, unzip into `data/movielens/`, then:
   ```bash
   python -m scripts.ingest_movielens --dir data/movielens --max-ratings 200000
   ```
   Ratings only get inserted for movies already ingested from TMDB — ingest TMDB first (or ingest by the tmdb ids that appear in `links.csv`).

## What was verified in this sandbox
- Migrations apply cleanly (SQLite; same models work unchanged against Postgres by swapping `DATABASE_URL`).
- All 20 sample movies ingested with correct genres/cast/crew/keywords/countries/companies.
- Cross-movie entity dedup works: e.g. Leonardo DiCaprio is a single `people` row referenced by both *Inception* and *Titanic*; *The Dark Knight* and *Batman Begins* correctly group under one `collections` row.
- Re-running ingestion on the same movie does not create duplicates (idempotent upsert).
- 609 synthetic ratings from 60 users loaded and correctly linked, giving a real (if small) interaction matrix — genre-biased so a Phase 3 CF/content model has real signal to learn from, not noise.

## Known gap (by design, not an oversight)
The sample dataset's ratings are synthetic (generated with a per-user genre bias, see `generate_sample_dataset.py`) because this sandbox has no network path to the real MovieLens ratings file. The ingestion *code* is real and tested; the *data* running through it in this environment is a stand-in until you run `ingest_movielens.py` locally with the actual dataset.
