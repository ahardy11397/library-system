# library-system: A Memelang Graduate-Level Demo

`library-system` is a complete library management CLI written entirely in [Memelang](https://github.com/aaronhardy/meme). It demonstrates typed records, persistent JSON serialization with null-safe `Optional<T>` handling, first-class `Set<String>` collections, in-language UTC calendar calculations, and cross-target parity across all six Memelang compilation backends (**Python**, **Rust**, **C++20**, **Go**, **C11**, and **TypeScript**).

---

## Architecture & Modules

The application is structured into thin, cohesive modules:

- **`main.meme`**: Thin CLI entry point. Routes command-line arguments (`argument count`, `argument <n>`), performs usage validation, and writes errors to `/dev/stderr` before exiting with status code `1`.
- **`catalog.meme`**: Book record definition, JSON persistence (`read json records Book ... or else nothing`), lookup by ISBN, title-based sorting, and catalog constraints.
- **`patron.meme`**: Patron record definition, JSON persistence, fine balance tracking, integer-cents money arithmetic, and exact dollar rendering.
- **`circulation.meme`**: Active loans, hold queues (`Set<String>` membership), checkout validation with four distinct failure modes, overdue fine computation, renewal logic, and FIFO hold queue priority.
- **`calendar.meme`**: Pure UTC civil calendar arithmetic derived from POSIX epoch seconds (leap year computation, days per month, and ISO date formatting `YYYY-MM-DD`).

---

## CLI Usage

The program compiles into a native executable via `meme build main.meme -o library_bin` or can be executed directly using `meme run main.meme`:

```bash
# Add books to the catalog
./library_bin add-book <isbn> <title> <author> <copies>

# Register patrons
./library_bin add-patron <patron-name>

# Check out a book (specifying loan duration in days)
./library_bin checkout <isbn> <patron> <days-until-due>

# Return a book (clears loan and applies any overdue fine to patron balance)
./library_bin return <isbn> <patron>

# Renew a book (extends due date by 7 days; blocked if holds exist)
./library_bin renew <isbn> <patron>

# Place a hold on a book (first-come, first-served queue)
./library_bin hold <isbn> <patron>

# Inspect patron fines (10 cents/day overdue, capped at $5.00)
./library_bin fines <patron>

# List catalog books sorted by title with real-time copy availability
./library_bin catalog

# View patron details, active loans, due dates, and running fine total
./library_bin patron <patron>
```

---

## Business Rules

1. **Checkout Failure Modes**:
   - `unknown book`: ISBN not found in the catalog.
   - `unknown patron`: Patron not registered.
   - `patron already has that ISBN out`: Patron cannot borrow multiple copies of the same book simultaneously.
   - `no copies left`: All physical copies are either checked out or reserved for patrons ahead in the hold queue.
2. **Hold Queue & Ordering**:
   - Patrons can place holds on books. Duplicate holds by the same patron on the same ISBN are rejected.
   - When a copy becomes available, the first patron in the hold queue (FIFO) has exclusive reservation priority. Other patrons cannot check out the reserved copy.
   - When the reserved patron checks out the book, their hold is automatically cleared from the queue.
3. **Renewals**:
   - A loan can be renewed to extend its due date by 7 days (604,800 seconds).
   - Renewals are strictly blocked if any holds exist for that ISBN.
4. **Fines & Money Handling**:
   - Money is tracked strictly in integer cents (`Int`) to prevent floating-point inaccuracies.
   - Overdue fine rate: **10 cents per day** overdue.
   - Fine cap: **500 cents ($5.00 maximum)** per loan.
   - Applied to the patron's running account balance upon return.
5. **Catalog Integrity**:
   - Books with 0 copies cannot be removed from the catalog while active loans remain open.

---

## Data Persistence

All state is persisted in standard JSON format in the `data/` directory (created automatically on demand):
- `data/books.json`: `List<Book>`
- `data/patrons.json`: `List<Patron>`
- `data/loans.json`: `List<Loan>`
- `data/holds.json`: `List<Hold>`

Missing files or malformed/corrupted files are handled cleanly through Memelang's `Optional<T>` record readers (`read json records ... or else nothing`), falling back to empty lists without crashing.

---

## Running the Test Suite

The test suite in `tests/meme/` covers catalog operations, patron records, checkout failure modes, return with fines, renewals, hold queue ordering, file error tolerance, and calendar math:

```bash
# Run tests against Python backend (default)
uv run --project /mnt/ssd/projects/meme meme test tests/meme

# Run tests across all native backends
uv run --project /mnt/ssd/projects/meme meme test tests/meme --target rust
uv run --project /mnt/ssd/projects/meme meme test tests/meme --target cpp
uv run --project /mnt/ssd/projects/meme meme test tests/meme --target go
uv run --project /mnt/ssd/projects/meme meme test tests/meme --target c
uv run --project /mnt/ssd/projects/meme meme test tests/meme --target ts
```

## Native Binary Build & Demo

To build a standalone native executable using the C11 backend:

```bash
uv run --project /mnt/ssd/projects/meme meme build main.meme -o library_bin
./library_bin
```
