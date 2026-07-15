# QIF to Wealthfolio CSV Converter

Converts Quicken `.qif` export files (Bank/Checking, Cash, CCard, and best-effort Investment
sections) into CSV files formatted for [Wealthfolio's](https://wealthfolio.app) CSV Import wizard.

## Features

- Parses Quicken QIF `!Type:Bank`, `!Type:Cash`, `!Type:CCard`, `!Type:Cat`, and `!Type:Invst` sections.
- Maps transactions to Wealthfolio `activityType` values (DEPOSIT, WITHDRAWAL, DIVIDEND,
  INTEREST, TAX, FEE, TRANSFER_IN, TRANSFER_OUT, BUY, SELL) using category info and
  payee/memo keyword hints.
- Normalizes all QIF date formats (including two-digit and space-padded years, e.g. `1/ 2' 0`)
  to ISO-8601 (`YYYY-MM-DD`).
- Captures Quicken check/reference numbers (`N` field) and writes them into a `comment` column
  alongside payee and memo, so they show up as notes in Wealthfolio.
- Logs any transaction blocks it could not confidently map to a `_parsing_warnings.txt` file
  instead of silently dropping them.
- Outputs UTF-8 CSV with decimal points (`.`), never commas, per Wealthfolio's format.

## Requirements

- Python 3.8 or later. No third-party dependencies — only the standard library is used.

## Usage

```bash
python3 qif_to_wealthfolio5.py --currency USD path/to/file1.qif path/to/file2.qif
```

- `--currency` — 3-letter ISO 4217 currency code applied to every transaction in the
  given file(s), e.g. `USD`, `EUR`, `CAD`.
- `--default-currency` — fallback currency used if `--currency` is omitted (defaults to `USD`).
- Multiple QIF files can be passed at once; each produces its own output CSV.

### Example

```bash
python3 qif_to_wealthfolio5.py --currency USD "Checking_Export_2026.QIF"
```

This produces:

- `Checking_Export_2026_wealthfolio.csv` — the Wealthfolio-ready CSV
- `Checking_Export_2026_parsing_warnings.txt` — only created if any blocks were skipped

## Output CSV format

```
date,symbol,quantity,activityType,unitPrice,currency,fee,amount,comment
2026-07-02,$CASH-USD,1,WITHDRAWAL,0.00,USD,0.00,1200.00,Check #1052 | Landlord LLC | July Rent
```

- Cash activities (`DEPOSIT`, `WITHDRAWAL`, `DIVIDEND`, `INTEREST`, `TAX`, `FEE`,
  `TRANSFER_IN`, `TRANSFER_OUT`) use the special symbol `$CASH-<CURRENCY>` (e.g. `$CASH-USD`),
  with `quantity` fixed at `1` and `unitPrice` fixed at `0.00`, per Wealthfolio's cash-import
  convention.
- Investment trades (`BUY`, `SELL`) use the actual security symbol, with real `quantity` and
  `unitPrice` values parsed from the QIF `Invst` block; `amount` is left blank for these rows
  since Wealthfolio auto-calculates it as quantity × unitPrice.
- The `comment` column combines the Quicken check number (labeled `Check #...` when numeric,
  or `Ref: ...` for non-numeric reference codes like `DEP`), payee, and memo.

## Mapping logic (brief)

1. **Transfers** — any transaction with "transfer" in its category/payee/memo, or categorized
   as "IRA Transfer", becomes `TRANSFER_IN` (positive amount) or `TRANSFER_OUT` (negative).
2. **Income** — transactions in an income category, or containing dividend/interest keywords
   (matched on word boundaries to avoid false positives like "internet"), map to `DIVIDEND`,
   `INTEREST`, `TAX` (refunds), `FEE`, or `DEPOSIT` as a fallback.
3. **Expense** — transactions in an expense category, or with a negative amount, map to `FEE`,
   `TAX`, or `WITHDRAWAL` as a fallback.
4. **Investment trades** — QIF action lines containing "buy"/"sell" map to `BUY`/`SELL`;
   dividend/interest actions in an Invst block map to `DIVIDEND`/`INTEREST`. Anything not
   confidently recognized is logged and skipped rather than guessed.

## Importing into Wealthfolio

1. Open Wealthfolio and go to **Accounts**, then select (or create) the account you want to
   import into — typically a cash/checking account for bank exports.
2. Go to **Activities → Import CSV** (or the CSV import option in the account view).
3. **Upload** — drop in the generated `_wealthfolio.csv` file. Wealthfolio shows a live preview
   of the parsed rows.
4. **Mapping** — align the CSV columns to Wealthfolio fields. Since the script already uses
   Wealthfolio's exact header names (`date`, `symbol`, `quantity`, `activityType`, `unitPrice`,
   `currency`, `fee`, `amount`, `comment`), the importer should auto-map every column. Confirm
   each `activityType` value is recognized (it will be, since the script only emits Wealthfolio's
   supported types). Save the mapping as a template if you plan to import more files into the
   same account.
5. **Review Assets** — for pure cash imports there are no new securities to confirm. If your
   file includes `BUY`/`SELL` rows, confirm the ticker symbols resolve correctly.
6. **Review Activities** — preview every row before import. Wealthfolio flags duplicates
   automatically (matched on date + symbol + quantity + unit price + amount + activity type),
   so re-importing the same file twice is safe.
7. **Import** — confirm the final summary of rows to import vs. skipped.

### Notes on transfers between two Wealthfolio accounts

The CSV importer only writes to the single account selected in step 1. If a transaction is a
transfer between two accounts you track separately in Wealthfolio, import the `TRANSFER_OUT`
row into the source account, then switch accounts and import the matching `TRANSFER_IN` row into
the destination account.

## Known limitations

- Investment (`!Type:Invst`) parsing is best-effort: only clear buy/sell/dividend/interest
  patterns are mapped. Anything else (stock splits, spin-offs, complex corporate actions) is
  logged to the warnings file and skipped — add these manually in Wealthfolio if needed.
- Category income/expense classification depends on the QIF file actually containing a
  `!Type:Cat` section with `I`/`E` markers. Files without one rely entirely on keyword hints.
