# review-testing-PR reviewer notes

## Architecture

Daylist is a small server-rendered PHP task manager with no framework, database, Composer dependencies, or JavaScript. Persistent tasks are stored in `data/tasks.json`; request handling, validation, sessions, CSRF checks, and storage are intended to live in `app.php`, while `public/index.php` renders the HTML interface. `tools/chaos.py` is a standalone test utility that injects malformed JSON records and restores a backup.

## Conventions

- Run the app with PHP 8+ and serve only `public/`; `README.md` explicitly requires `php -S localhost:8000 -t public` and keeping `app.php` and `data` outside the document root.
- Treat persisted JSON as untrusted input. `app.php` filters records by required types and constraints (`id`, nonblank `title` ≤200 bytes, and boolean `done`) before rendering or mutation.
- Use one exclusive file lock around reads and writes. `taskStore()` in `app.php` opens `tasks.json` with `c+`, takes `LOCK_EX`, rewinds, writes JSON, truncates, flushes, and unlocks in `finally`.
- Mutating requests use POST, an allowlisted `action` (`add`, `edit`, `toggle`, `delete`), CSRF validation, and a 303 redirect after success (`app.php`).
- Escape all user-controlled output with the shared `escape()` helper. Examples include task titles, IDs, error messages, status messages, and accessible labels in `public/index.php`.
- Keep the UI progressively enhanced and server-rendered: task edits, toggles, deletes, filters, and confirmation use ordinary forms, links, and `<details>` elements (`public/index.php`).
- The Python chaos tool must not run concurrently with PHP. It uses an exclusive backup (`open("xb")`), atomic replacement via `os.replace`, and requires restore before another injection (`tools/chaos.py`).

## Intentional non-standard choices

- Local JSON storage and a shared task list for every visitor are deliberate learning-project constraints, not a substitute for a production database or account model (`README.md`).
- No JavaScript is used; delete confirmation is implemented with native `<details>` and a second POST form (`public/index.php`).
- Malformed persisted records are discarded rather than repaired or surfaced individually; this supports the documented chaos test behavior (`app.php`, `README.md`).

## Watch out for

- Treat the sampled `app.php` as currently broken until corrected: `b2hex`, the `escape()` declaration/body, the malformed `$filter` expression, and the `formFields()` return type/signature are invalid or inconsistent PHP.
- `public/index.php` requires `bootstrap.php`, but the supplied files define `app.php` instead; flag missing or incorrectly wired bootstrap behavior.
- Preserve byte-length validation consistency: both request validation and persisted-record validation use `strlen`, while user-facing text says “bytes” (`app.php`).
- Do not bypass `taskStore()` for task reads or writes; doing so can reintroduce lost updates or expose malformed records.
- Ensure every rendered dynamic value remains escaped, including filter parameters and task IDs used in HTML attributes.