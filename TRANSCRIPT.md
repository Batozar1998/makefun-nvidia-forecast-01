# Build transcript — polymarket-nvidia-ceo-forecast-01 entry

Agent: Muse (personal AI agent of Batozar / batozar98), with a subagent
building the entry.
Date: 2026-11-13 (WIB)
Bounty: make.fun "Forecast Polymarket's next NVIDIA CEO prices minute by
minute, 4 hours ahead" (polymarket-nvidia-ceo-forecast-01), $250, 100
places paid. Deadline Sat 2026-11-14 04:00 WIB (2026-11-13T21:00:00Z).

## Session log

1. User had already entered the make.fun joke bounty (ai-joke-03,
   submitted and accepted 2026-11-13 19:35 WIB) and chose this bounty
   next ("Ikut NVIDIA forecast ($250)") after the Nasdaq forecast bounty
   was found closed.
2. Agent read the competition spec live (GET
   /api/competitions/polymarket-nvidia-ceo-forecast-01) and the kit
   (starter/main.ts, input/p00.in–p05.in, harness/):
   program `python3 /bot/main.py`, run once per case, no network, ≤8s,
   1 CPU, 1024 MB; input `/input/<id>.in` (schema
   makefun/polymarket-case/v1, 24 markets × 72h of 1-minute Yes prices);
   output `/output/<id>.out` = exactly
   `{"forecasts": [{"id": <market id>, "q": [[5 quantiles] x 240]}]}`;
   scoring = mean pinball loss (×100) vs realised prices; a prize needs
   a loss strictly below the best house entry's loss on the final.
   House baseline: q = x + z_τ · 0.0025 · sqrt(h); published practice
   loss 0.363589.
3. Agent built a local scorer (`score_local.py`): fetched all 24
   markets' real Polymarket price history (2026-11-12T06:00Z to
   2026-11-13T06:00Z) from the public CLOB API, built a per-minute
   price grid, and validated it against the practice inputs (72,535
   points, 0 mismatches). The scorer reproduces the house baseline's
   practice loss as 0.3718 (per-case mean) / 0.3623 (item-weighted)
   vs the published 0.363589 — close enough to trust for comparisons.
4. First forecaster (v1): pure empirical change quantiles over a
   horizon grid, median = empirical median change. Practice loss
   0.383 — WORSE than the baseline. Diagnosis: the long-horizon
   empirical distribution conflates the 72h trend with 4h-ahead
   uncertainty (e.g. Colette Kress fell 0.485 → 0.17 over the window),
   and rare huge jumps inflate the tails, so the bands were too wide.
5. Experiments on the 6 practice cases (same scorer):
   - median = checkpoint price + normal quantiles with per-market
     σ = mean|1-min change| / 0.7979: 0.3615
   - σ from 60-min changes: 0.4955 (trend contamination — rejected)
   - empirical quantiles, changes winsorized ±0.02, median = x: 0.3405
   - same, last 24h only: 0.3395
   - winsor ±0.05: 0.3329; winsor ±0.01: 0.3500
   - 50/50 blend of winsorized-empirical and normal (σ from 1-min
     changes): 0.3221 — best, beats the baseline on all six cases
   - blend weights 25/75 and 75/25: 0.3387 and 0.3209 (the 75% figure
     is within noise of 50/50, so the principled 50/50 was kept rather
     than chasing a razor-thin optimum)
   - a σ floor of 0.0008: 0.3233 (slightly worse; dropped — the
     empirical component already gives quiet markets some tail width)
6. Final `main.py` implements the 50/50 blend (winsor ±0.02, median =
   checkpoint price, fallback to the house baseline for markets with
   fewer than 10 prices — the threshold was checked: 2/10/15 all give
   the same practice score, 30 is worse because three young markets in
   p03 have only 19–20 prices).
7. Validation of the final program:
   - practice loss 0.3221 per-case mean, 0.3121 item-weighted
     (baseline 0.3718 / 0.363589 published); beats the baseline on all
     six cases (p00 0.482 vs 0.536, p01 0.358 vs 0.380, p02 0.385 vs
     0.408, p03 0.307 vs 0.369, p04 0.239 vs 0.299, p05 0.162 vs 0.238)
   - deterministic: two runs produce byte-identical outputs
   - ~0.3s per case (budget 8s); output ≤ 234 KB per case (limit
     1,048,576 bytes)
   - format checks pass on all six practice outputs: exactly one key
     `forecasts`, every market has 240 rows × 5 quantiles, all finite,
     in [0, 1], non-decreasing per row
8. Added MIT LICENSE (with SPDX line), README.md, and this transcript.
   Local git repo initialised, committed as Batozar1998. Bundled the
   sessions and ran `check` — 0 failed. NOT submitted; the GitHub repo
   and the final submit are the parent agent's step.

## Honest caveats

- Practice is not the final: the final's 12 checkpoints run on new
  data (2026-11-13T21:00Z to 2026-11-15T17:00Z). A ~14% practice margin
  over the house baseline is a good sign, not a guarantee — and the
  house *model* forecast (per-market σ + drift from an AI call) is the
  other bar; its practice loss is not published, so it could not be
  compared directly.
- The forecaster adapts per market from that market's own recent
  prices. If a market jumps on news in the final window, no
  backward-looking forecaster can foresee it; the bands are built to
  absorb jumps, not predict them.
