# CineMind AI — Phase 4: FastAPI Backend

## What's in this phase
- `app/main.py` — the FastAPI app, CORS configured for the Vite dev server (`localhost:5173`).
- `app/core/security.py` — JWT issuance/verification (access + refresh tokens) and password hashing.
- `app/core/deps.py` — `get_current_user` / `get_current_user_optional` dependencies.
- `app/core/cache.py` — TTL cache wrapper for heavy read endpoints (swap for Redis later without touching call sites).
- `app/schemas/` — Pydantic request/response DTOs.
- `app/services/` — business logic (routers stay thin): `auth_service`, `search_service` (dynamic multi-filter query builder), `recommendation_service` (wraps the Phase 2 `HybridRecommender`), `activity_service` (ratings/watchlist/history + interaction logging), `analytics_service` (real-time taste profile computation).
- `app/routers/` — `auth`, `movies`, `search`, `people`, `discovery` (genres/languages/countries/collections/timeline/eras), `recommendations`, `activity` (ratings/watchlist/history), `analytics`.

## Quickstart
```bash
cd backend
pip install -r requirements.txt
# (assumes Phase 1 + 2 already ran: alembic upgrade head, seed_sample_data, train_recommender)

python -m uvicorn app.main:app --reload
```
Then open **http://127.0.0.1:8000/docs** for interactive Swagger UI — every endpoint below is listed there with a "Try it out" button.

## Verified in this sandbox
- **A real `uvicorn` process** (not just `TestClient`) booted, served `/health`, `/movies/1`, and `/docs` over actual HTTP.
- **42/42 automated tests pass** (`pytest tests/ -v`): 7 ingestion (Phase 1) + 14 ML (Phase 2) + 21 API (Phase 4).
- **Full manual flow**: register → login → search (multi-filter) → rate a movie → add to watchlist → get personalized recommendations with real explanations → view taste profile — all exercised end-to-end against the seeded sample data.

## Endpoints implemented

| Method | Path | Auth | Notes |
|---|---|---|---|
| POST | `/auth/register` | – | Returns access + refresh tokens |
| POST | `/auth/login` | – | |
| POST | `/auth/refresh` | – | |
| GET | `/auth/me` | ✓ | |
| GET | `/movies/{id}` | – | Full detail: cast, crew, genres, keywords, collection |
| GET | `/movies/{id}/similar` | – | Content-based, from Phase 2 |
| GET | `/search` | – | genre/actor/director/language/country/year/rating/keyword/collection, combinable |
| GET | `/genres`, `/genres/{id}/movies` | – | |
| GET | `/languages`, `/languages/{code}/movies` | – | |
| GET | `/countries`, `/countries/{code}/movies` | – | |
| GET | `/collections/{id}` | – | Franchise page, chronologically ordered |
| GET | `/timeline/{year}` | – | |
| GET | `/eras/{label}` | – | e.g. `/eras/2010-2019` |
| GET | `/people/{id}` | – | Actor/director profile + filmography |
| GET | `/recommendations/for-you` | ✓ | Hybrid engine, real explanations |
| GET | `/recommendations/because-you-liked/{movie_id}` | – | |
| GET | `/recommendations/hidden-gems` | – | |
| POST | `/ratings/{movie_id}` | ✓ | |
| GET/POST/DELETE | `/watchlist[/{movie_id}]` | ✓ | |
| GET/POST | `/history[/{movie_id}]` | ✓ | |
| GET | `/analytics/taste-profile` | ✓ | Computed live from ratings, not stored/static |
| POST | `/onboarding/preferences` | ✓ | Feeds cold-start recommendations |

## A real bug this caught
`MovieDetailOut` originally sorted cast by `x.billing_order or 999` — which silently treats billing_order `0` (the lead actor) as falsy and pushes them to the *back* of the cast list. `/movies/1` returned Leonardo DiCaprio last instead of first until this was caught by inspecting real API output, not just a passing test. Fixed to `x.billing_order if x.billing_order is not None else 999`; regression-tested in `tests/test_api.py::test_get_movie_detail`.

## Known simplifications (by design, flagged not hidden)
- **Recommendation artifact caching**: `recommendation_service.get_recommender()` caches the loaded ML artifacts (TF-IDF matrix, CF factors) in a module-level dict for the life of the process, and rebuilds a lightweight `HybridRecommender` wrapper per request around them (cheap — it's just object construction, no I/O). Call `reset_recommender_cache()` after re-running `train_recommender.py` so a long-running API process picks up new artifacts without a restart — not yet wired to a file-watcher or admin endpoint; that's a Phase 8 (Optimization) item.
- **Mood-based discovery, movie/actor/director comparison, and admin analytics** from the architecture doc are not yet built as endpoints — the underlying pieces exist (mood tables from Phase 1, `build_query_vector` from Phase 2) but the routers aren't written. Next logical addition if you want them before the frontend.

## Next step
Say **"proceed with Phase 5"** for the React frontend, which will consume exactly these endpoints.
