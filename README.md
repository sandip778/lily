# lily-ledger

This is a public, timestamped record of a NEPSE stock-prediction model's daily outputs. It lets anyone check
that the published track record was not edited after the fact.

## How it works

1. **Same evening.** After the model runs on a trading day, every prediction for that day goes into one
   **canonical file**, and the file's SHA-256 hash (the *commitment*) is added to `ledger.json`. The commit is
   pushed here before the next NEPSE session opens (11:00 Nepal time). GitHub's commit history is the
   independent timestamp.
2. **While the holding period runs** (10 trading days plus a buffer), the file itself stays private. Only its
   hash is public.
3. **After the outcome is known**, the file is published in `revealed/<pred_date>.csv`. Anyone can hash it and
   compare the result with the commitment made weeks earlier.

Each file starts with a random salt line. On a day with few predictions, the salt stops anyone from guessing
the contents and checking guesses against the hash.

## `ledger.json` fields

| field | meaning |
|---|---|
| `pred_date` | trading day the predictions were made for (after that day's close) |
| `n_predictions` | number of rows in the canonical file |
| `commitment` | SHA-256 of the canonical file, lowercase hex |
| `committed_at_utc` | when the commitment was recorded (UTC) |
| `model_version` | model version that produced the predictions |
| `reveal_date` | earliest planned reveal date; the file is revealed only once every outcome has settled |
| `late` | `true` if recorded after the next session had opened; such days never count as verified |

## Canonical file format

- UTF-8, `\n` line endings, final newline.
- Line 1: `salt:` followed by 64 lowercase hex characters.
- Line 2: `Symbol,Pred_Date,Expected_Alpha,Alpha_Signal,Stock_Condition,Regime,Model_Version`.
- Then one row per prediction, sorted by Symbol. Dates are `YYYY-MM-DD`; Expected_Alpha has exactly 6 decimals.
- A field containing a comma is wrapped in double quotes, e.g. `"overbought, at 52W high"`.

## Verify a revealed day

Windows PowerShell:

```powershell
(Get-FileHash -Algorithm SHA256 .\revealed\2026-11-04.csv).Hash.ToLower()
```

macOS / Linux:

```sh
sha256sum revealed/2026-11-04.csv
```

The result must equal that day's `commitment` in `ledger.json`. Then check in this repository's commit
history that the commitment was pushed before the next session opened.

Do not open and re-save a revealed file in Excel or any editor before hashing it. Any change, even to line
endings, changes the hash.

## Disclaimer

This repository publishes the historical record of a statistical model for research and educational
purposes. It is not investment advice or a recommendation to buy or sell any security.
