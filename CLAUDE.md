# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this branch is

`master` is the **Delphi 12 / VCL, Windows-only** HeidiSQL (MySQL/MariaDB, MS SQL, PostgreSQL, SQLite, Interbase/Firebird client). It cannot be compiled on macOS or Linux, and there is no automated test suite for the app itself: verify changes by reading the code, and say so when something was not compiled. Non-Windows builds live on the **`lazarus` branch** of the official repo (`HeidiSQL/HeidiSQL`), see the last section.

## Building (Windows only)

- `php build.php` is the real build: it needs Delphi 12.3 at `C:\Program Files (x86)\Embarcadero\Studio\23.0`, madExcept, git, php and the Transifex `tx` client (`extra/internationalization/`). It compiles SynEdit and VirtualTreeView from `components/`, compiles the `res/*.rc` files, downloads and compiles the translations, patches the `.mo` files and the madExcept handler into the exe, and writes `out/heidisql.exe`. `out/heidisql.iss` is then loaded in Inno Setup for the installer.
- **`build.php` runs `git clean -dfx` first**, deleting every unversioned file in the working tree. Never run it with uncommitted or untracked work you want to keep.
- The version is `FILEVERSION` from `res/version.rc` with the last number replaced by `git log` commit count + 671 (earlier Subversion commits), so the build number is not stored in the repo.
- For IDE work: install the SynEdit and VirtualTree packages from `components/` first, run `build-res.bat` for the resource files, then open `packages/Delphi12.3/heidisql.dproj` (the one `build.php` uses; a `Delphi12.1` copy of the project also exists).

## Architecture

All application code is in `source/`. Forms come as `.pas` + `.dfm` pairs (about 36), and `main.pas` (`TMainForm`, ~16k lines) is the main window and by far the largest unit.

**Database layer, spread over three kinds of files:**
- `dbconnection.pas` holds the abstract `TDBConnection` and one subclass per driver: `TMySQLConnection` (libmysql/libmariadb), `TPgConnection` (libpq), `TSQLiteConnection`, `TAdoDBConnection` (MS SQL through ADO) and `TInterbaseConnection` (FireDAC). Each has a matching `TDBQuery` subclass (`TMySQLQuery`, `TPGQuery`, ...). Shared model classes (`TDBObject`, `TTableColumn`, `TTableKey`, `TForeignKey`, ...) and the per-database object cache (`TDatabaseCache`) are also there.
- `dbstructures.pas` defines `TNetType` (every connection type: TCP/IP, named pipe, SSH tunnel, RDS, local, ...), the query-id enum `TQueryId` and `TDbLib`, the base class that `LoadLibrary`s a client DLL and binds its procedures with `AssignProc`.
- `dbstructures.<engine>.pas` (mysql, postgresql, mssql, sqlite, interbase) holds that engine's constants, data types and its SQL provider. Engine-specific SQL is reached through `FSqlProvider.GetSql(qXxx)`, so a new per-engine statement means a new `TQueryId` value plus an implementation in every `dbstructures.<engine>.pas`.
- Client libraries are not linked: they are DLLs loaded at runtime from the exe folder (`out/`: `libmariadb-64.dll`, `libpq-*.dll`, `sqlite3-64.dll`, ...).

**Data files next to the exe (`out/`, tracked in git):** `functions-<engine>.ini` feed the SQL editor's function/keyword help and are produced by `extra/functions-ini-generator.php`; `Snippets/` holds `.sql` snippets; `heidisql.iss` is the installer script. The other PHP scripts in `extra/` are one-off generators (`update_pas_mysql_structures.php`, `keywords-generator.php`, `find-unused-translation-strings.php`).

**Translations:** user-visible strings go through `_('...')` (`gnugettext.pas`) or `f_('...%s...', [args])` (`apphelpers.pas`). `.po`/`.mo` files are gitignored and come from Transifex (`.tx/config` maps `out/locale/<lang>/LC_MESSAGES/default.po`), so never edit them by hand.

**Vendored code:** `components/synedit` and `components/virtualtreeview` are the editor and tree controls, with local changes, so treat them as part of the project. Other helpers sit in `source/` subfolders (`detours`, `sizegrip`, `vcl-styles-utils`).

## Conventions

- Commit messages look like `fix: ...` or `feat: ...`, and reference tickets as `See #1234`.
- Official contribution rules (`CONTRIBUTING.md`): pull requests are accepted **only for bug fixes**, and every PR must cite a ticket. If none exists, an issue has to be opened first with every field of the bug template filled in (description, HeidiSQL version and OS, database server version, reproduction recipe).
- `.gitattributes` normalizes line endings: `.pas`, `.dfm`, `.inc` use native endings, while `.iss`, `.dpk`, `.dproj` and `.groupproj` stay CRLF.

## Non-Windows builds (`lazarus` branch)

The Lazarus/FPC port has its own `heidisql.lpi`, `Makefile`, `create-macos-app.sh` and CI, and is not in this tree. Fetch it from the official repo (`git remote add upstream https://github.com/HeidiSQL/HeidiSQL.git && git fetch upstream lazarus`) and work in a separate worktree so this checkout stays on `master`. On macOS it needs:
- FPC 3.2.4rc1a and Lazarus 4.8 for the host CPU, with every patch from `extra/lazarus-patches/4.8/` applied to the Lazarus tree (Cocoa HiDPI, VirtualTreeView, SynEdit); without them it fails to build or renders wrongly.
- `make build-macos` with `lazbuild` on the `PATH`, producing `out/macos/heidisql`; then `./create-macos-app.sh --adhoc` to bundle the client dylibs into `heidisql.app/Contents/Frameworks` and sign ad-hoc. That script assumes Homebrew; on MacPorts a copy pointing at `/opt/local` is needed.
- The `.lpi` has no target CPU, so the same sources build for Apple Silicon and Intel. On FPC 3.2.4rc1 x86_64, generic containers keyed by small enum types can lose keys, which is why upstream code keys such dictionaries by `Ord(...)`.
