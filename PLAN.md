# PLAN — library-system: a Memelang graduate-level demo

Build a complete library management CLI entirely in Memelang. This is a
beta demonstration program: it must look like what a CS graduate would
build solo for a real small library.

Language docs: /mnt/ssd/projects/meme/docs/WRITING_GUIDE.md and
docs/TUTORIAL.md (read both first). Compiler commands run via
`uv run --project /mnt/ssd/projects/meme meme ...` from this repo.

## Ground rules (non-negotiable)

- Work ONLY in this repo. NO git commits — leave changes for review.
- 100% Memelang for program logic; no other languages.
- Money/fines in integer cents; floats only at print time via
  `format <x> with 2 decimals`.
- All persistence as JSON files in `data/` (created on demand) using the
  Optional record readers/writers; corrupt/missing file = clean Optional
  handling, never a traceback.
- Determinism: tests never assert exact wall-clock values (use fixed epoch
  values in test data; the clock is used only for "loaned at" timestamps).
- Every construct must parse with `--provider heuristic`.
- Run all test suites in the FOREGROUND only.

## Domain and CLI (argv-routed thin main + logic modules; unknown command →
usage + exit 1; every error = one stderr line + exit 1)

    library add-book <isbn> <title> <author> <copies>
    library add-patron <patron-name>
    library checkout <isbn> <patron> <days-until-due>
    library return <isbn> <patron>
    library renew <isbn> <patron>            # blocked if holds exist
    library hold <isbn> <patron>             # Set<String> of patrons per ISBN
    library fines <patron>                   # overdue fine: 10 cents/day, cap $5.00
    library catalog                          # books sorted by title, availability
    library patron <patron>                  # patron's loans + running fine total

Business rules: checkout fails on no copies left / unknown book / unknown
patron / patron already has that ISBN out; return clears the loan and
applies any fine to the patron's balance; renew extends due date 7 days but
is blocked when a hold queue is non-empty; a book with 0 copies cannot
be deleted from catalog while loans are open.

Suggested data model: `Book`, `Patron`, `Loan` records; `Map<String, Book>`
by ISBN; `Set<String>` hold queues per ISBN (a `Map<String, Set<String>>`
or per-ISBN list — your call, document it); loans as `List<Loan>`.
Month/day math for due-date display from epoch seconds (UTC, in-language).

## Tests — tests/meme/*.meme, `meme test`

Cover at minimum: add-book/add-patron happy paths and duplicate rejection;
checkout happy path + all four failure modes; return with fine (fixed
epochs); renew blocked by hold; hold queue ordering (first patron gets the
next free copy); fines cap at 500 cents; catalog sort order; corrupt data
file tolerance (Optional reader → clean error). Use `assert` in `test`
blocks; per-test scratch files, cleaned up.

## Acceptance ("done means")

1. `uv run --project /mnt/ssd/projects/meme meme test` — 0 failed (python).
2. Same with `--target rust`, `cpp`, `go`, `c`, `ts` — ALL SIX green.
3. `meme build main.meme -o library_bin` — run an end-to-end demo (2 books,
   2 patrons, checkout, hold, renew-blocked, return with fine, fines
   report) and include the transcript in your final summary.
4. README.md: CLI usage, data files, business rules, test instructions.
5. Final message: files, test count per target, demo transcript,
   deviations (if any) with reasons.
