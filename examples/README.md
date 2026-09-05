# Sample output

Produced by an actual run, not hand-written:

```
python main.py --input invoices --output output/results.csv \
               --line-items-csv output/results_line_items.csv
```

```
          INFO                                     1 to process, 0 already in output/results.csv
[  1/1]    OK    invoices.pdf                      Acme Tech Solutions  ·  2026-10-12  ·  4,112.14 USD  ·  4 items

================================================================
  Run complete in 0m 1s
  Parsed        1   -> output/results.csv
  Skipped       0   (already processed)
  Failed        0
================================================================
```

- **`sample-results.csv`** — the default: one row per invoice, line items
  as JSON in a single column.
- **`sample-results-line-items.csv`** — the optional `--line-items-csv`
  long format: one row per line item, keyed by `source_file`.

Note that the line items sum to 3,789.99 against a stated total of
4,112.14. That gap is tax, and it is why the built-in cross-check only
flags an invoice when the line items exceed the total — a naive equality
check would mark almost every real invoice as suspect.
