# CineMind AI — Phase 2: Recommendation Engine

## What's in this phase
- `app/ml/feature_engineering.py` — content-based feature matrix: weighted TF-IDF (overview+tagline+keywords) + multi-hot genres + multi-hot director/top-cast + one-hot language, each block L2-normalized before weighting so no block dominates by column count alone.
- `app/ml/content_based.py` — top-K similarity neighbors per movie (not a full N×N matrix — that's the real production pattern once the catalog is thousands of movies, not 20).
- `app/ml/collaborative.py` — mean-centered truncated SVD over the ratings matrix. Genuine matrix factorization, not a lookup table.
- `app/ml/hybrid.py` — query-vector builder (used for cold-start profiles from onboarding preferences), popularity prior, and hidden-gem thresholds **derived from the actual catalog distribution**, not hard-coded.
- `app/ml/recommender.py` — the `HybridRecommender`: `final_score = α·content + β·cf + γ·popularity`, with weights dynamically zeroed out (not just left at defaults) when a signal genuinely isn't available (new user, new movie), and explanations generated from whichever term actually won — not a canned string bank.
- `app/ml/evaluation.py` + `scripts/evaluate_recommender.py` — Precision/Recall/F1/NDCG/MAP@K, Coverage, Diversity, RMSE/MAE, comparing Content-only vs CF-only vs Hybrid on a leakage-free 80/20 per-user split.
- `app/ml/artifact_io.py` + `scripts/train_recommender.py` — the offline/online boundary: training happens once in a batch script, the artifacts are pickled to `ml_artifacts/`, and would-be API requests just load them (no retraining per request).

## Quickstart
```bash
cd backend
# (assumes Phase 1 already ran: alembic upgrade head + seed_sample_data)
python -m scripts.train_recommender
python -m scripts.evaluate_recommender
pytest tests/test_recommender.py -v
```

## Verified in this sandbox (against the 20-movie / 60-user Phase 1 sample data)

**Training runs end-to-end:**
```
[1/4] 20 movies, 306 feature dims
[2/4] top-20 content similarity neighbors
[3/4] SVD, n_factors=10, 60 users x 20 movies
[4/4] hidden-gem thresholds from catalog: min_vote_count=9375, popularity_cutoff=53.00
```

**Personalized recommendations for a real user** (`ml_user_1`, who rated Tenet 5.0, Meg 2 & Gangubai 4.0, Interstellar 3.5 — a sci-fi/action-leaning profile):
```
0.519 Inception          | collaborative | Users with similar taste to yours rated this highly.
0.516 Pushpa: The Rise    | collaborative | Users with similar taste to yours rated this highly.
0.516 Batman Begins       | collaborative | Users with similar taste to yours rated this highly.
```

**Cold-start user (no ratings, no preferences)** — correctly falls back to popularity, not an error or a random list:
```
0.888 Inception       | popularity | Popular and highly rated (8.4/10) — a solid pick while we learn your taste.
0.879 The Dark Knight  | popularity | Popular and highly rated (8.5/10) — a solid pick while we learn your taste.
```

**Similar movies to Inception** (pure content-based) correctly surfaces other Nolan sci-fi/action films above unrelated genres — verified explicitly in `tests/test_recommender.py::test_nolan_movies_are_more_similar_to_each_other_than_to_unrelated_genre`.

**Hidden gems** on this catalog: *Your Name.*, *Parasite*, *Mad Max: Fury Road* — high rating, sufficient votes, below-median popularity, computed from real thresholds not guessed constants.

**Offline evaluation** (Content-only vs CF-only vs Hybrid, 80/20 per-user split, 60 test users):

| variant | P@5 | R@5 | F1@5 | NDCG@5 | MAP@5 | Coverage@5 | Diversity@5 | RMSE | MAE |
|---|---|---|---|---|---|---|---|---|---|
| content_based | 0.140 | 0.471 | 0.211 | 0.315 | 0.229 | 0.850 | 0.887 | – | – |
| collaborative | 0.170 | 0.554 | 0.254 | 0.386 | 0.304 | 1.000 | 0.928 | 0.915 | 0.757 |
| hybrid | 0.170 | 0.592 | 0.258 | 0.366 | 0.263 | 0.900 | 0.892 | 0.915 | 0.757 |

**Honest read of these numbers, not a cherry-picked one:** collaborative and hybrid beat content-based here, which makes sense given how the sample data was generated — each synthetic user has a genre bias baked directly into their ratings (see Phase 1's `generate_sample_dataset.py`), which is exactly the kind of signal CF is built to pick up, while the content model has to infer taste indirectly through text/genre similarity. RMSE ≈0.92 on a 0.5–5 scale with only 20 movies and 10 latent factors is a reasonable, unremarkable number — not evidence of a great model, just evidence the pipeline is computing something real. **This is not the real evaluation** — it's proof the metric code and split logic are correct. Re-run `evaluate_recommender.py` after ingesting real TMDB/MovieLens data (Phase 1 `PHASE1_README.md`) for numbers worth reporting in the final project write-up.

## Design decisions worth flagging
- **Leakage avoidance**: `evaluate_recommender.py` deliberately does NOT reuse the live `HybridRecommender` against the real DB, because that class reads `Rating` rows straight from the database — using it directly would leak test-set ratings into both the user profile and the CF model. The evaluation script builds train-only versions of both from the split DataFrame instead.
- **Top-K, not full similarity matrix**: `content_based.py` stores only the top-K neighbors per movie. Fine to materialize a dense N×N matrix at 20 movies; at real catalog scale (10k+) that step should move to `sklearn.neighbors.NearestNeighbors(metric="cosine")` — flagged directly in the code comment, not silently left as a scaling landmine.
- **Weights are dynamically renormalized**, not just defaulted, when a component is unavailable — a brand-new movie with zero ratings doesn't get a phantom `β·0` dragging its score down against movies that do have CF signal; the remaining active weights are renormalized so scores stay comparable.

## Next step
Phase 3 (Database refinement) is already effectively done as part of Phase 1/2 — the schema, migrations, and artifact tables are in place. Say **"proceed with Phase 4"** to move to the FastAPI backend: wiring these ML modules and Phase 1 models into the actual REST API (auth, movies, search, recommendations, watchlist, history, analytics endpoints) from the architecture doc.
