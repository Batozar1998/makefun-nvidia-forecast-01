# Forecast Polymarket's next NVIDIA CEO prices — make.fun entry

Bounty: [polymarket-nvidia-ceo-forecast-01](https://make.fun/bounty/polymarket-nvidia-ceo-forecast-01)
— "Forecast Polymarket's next NVIDIA CEO prices minute by minute, 4 hours
ahead" ($250, 100 places paid).

## The task

For each of the 24 Polymarket "Next CEO of NVIDIA?" markets, forecast the
Yes price minute by minute for 4 hours (240 minutes) ahead, as five
quantiles (0.05, 0.25, 0.5, 0.75, 0.95) per minute. The program runs once
per case at each of 12 checkpoints (4h apart, 2026-11-13T21:00Z to
2026-11-15T17:00Z), in a sandbox with no network, ≤8 seconds, 1 CPU,
1024 MB. It reads `/input/<id>.in` (72h of 1-minute Yes prices per market)
and writes `/output/<id>.out`. Scoring is mean pinball loss against the
realised prices; a prize needs beating the best house entry's loss.

## The approach

`main.py` is a per-market statistical forecaster — no machine learning,
no network, standard library only:

- **Median = the checkpoint price.** These prices are (near-)martingales;
  the 72h median change is noise that does not persist 4h ahead.
- **Spread = a 50/50 blend of two estimators:**
  - *Empirical:* for horizons 1, 2, 4, 8, 16, 32, 64, 128, 240 minutes,
    the quantiles of the price change over that horizon, each change
    winsorized at ±0.02 (these prices move in steps and jump; a few huge
    jumps must not set the whole band), interpolated in sqrt(h). This
    captures each market's own step shape and fat tails.
  - *Normal:* q = z_τ · σ · sqrt(h), with σ the market's own per-minute
    volatility (mean |1-minute change| / 0.7979). This smooths and
    stabilises the bands where the empirical distribution is noisy.
- Bands are clipped to [0, 1], forced non-decreasing, rounded to 6
  decimals. Markets with fewer than 10 prices fall back to the house
  baseline formula (σ = 0.0025).

## Local practice score

Scored on the 6 practice cases (p00–p05) against real Polymarket prices
fetched from the public CLOB API (see `TRANSCRIPT.md` for the method):

| forecaster | mean pinball loss |
|---|---|
| house baseline (published practice: 0.364) | 0.372 |
| this entry | **0.322** (item-weighted 0.312) |

It beats the house baseline on all six practice cases. (Practice is no
guarantee for the final — the final's 12 checkpoints run on new data —
but a ~14% margin on practice is the honest signal we have.)

## How to run

```sh
python3 main.py
```

Reads `$MAKEFUN_INPUT_DIR` (default `/input`), writes
`$MAKEFUN_OUTPUT_DIR` (default `/output`): one `<id>.out` per `<id>.in`.
Deterministic (no randomness), ~0.3s per case on 1 CPU.

## Files

- `main.py` — the entry program (entrypoint)
- `TRANSCRIPT.md` — the agent session transcript that built it
- `LICENSE` — MIT
