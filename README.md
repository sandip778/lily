# lily-ledger

This is a public, timestamped record of a NEPSE stock-prediction model's daily outputs. It lets anyone check
that the published track record was not edited after the fact.

## How it works

1. **Same evening.** After the model runs on a trading day, every prediction for that day goes into one
   **canonical file**, and the file's SHA-256 hash (the *commitment*) is added to `ledger.json`. The file is
   also stamped with **OpenTimestamps**, and the proof is saved as `proofs/<pred_date>.ots`. Both are pushed
   here before the next NEPSE session opens (11:00 Nepal time).
2. **While the holding period runs** (10 trading days plus a buffer), the file itself stays private. Only its
   hash is public.
3. **After the outcome is known**, the file is published in `revealed/<pred_date>.csv`. Anyone can hash it and
   compare the result with the commitment made weeks earlier.

## Why the `.ots` proofs matter: git dates are not proof

A git commit's date is written by whoever makes the commit, and it can be set to any value. History can also be
rewritten and force-pushed. So the dates in this repository's history, on their own, prove nothing about when a
prediction was made.

The proof of time is the **OpenTimestamps** file in `proofs/`. Each proof ties that day's commitment hash to a
Bitcoin block. Once the block is mined, nobody can produce a valid proof for a hash that did not exist before
that block, and that includes the author of this repository. A proof contains only the hash, never the
predictions, so it is published as soon as it is made.

A new proof is "pending" for the first few hours, until the calendar servers' Bitcoin transaction confirms.
It is then upgraded in place to a complete proof.

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

The result must equal that day's `commitment` in `ledger.json`.

Then check **when** it existed:

- **In a browser:** open https://opentimestamps.org, drop `revealed/<pred_date>.csv` and
  `proofs/<pred_date>.ots` on the page, and read the Bitcoin block time it attests.
- **On the command line:** `ots verify proofs/<pred_date>.ots -f revealed/<pred_date>.csv`, using any
  OpenTimestamps client.

The attested time must come before the next NEPSE session after `pred_date`. Note that the block time is an
upper bound; the stamp was made at or before it. A proof is checked against Bitcoin, not against this
repository, so it remains valid even if this repository were edited.

Even before a day is revealed, anyone can confirm what its proof commits to. `ots info proofs/<pred_date>.ots`
prints the file hash, which equals that day's `commitment`.

`test-stamp/` holds one stamp of a meaningless random file. It was made to test the setup and is not a prediction.

Do not open and re-save a revealed file in Excel or any editor before hashing it. Any change, even to line
endings, changes the hash.

## Disclaimer

This repository publishes the historical record of a statistical model for research and educational
purposes. It is not investment advice or a recommendation to buy or sell any security.
