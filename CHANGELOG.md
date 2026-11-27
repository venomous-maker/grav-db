# Changelog

Every release of grav-db, newest first. Each is a `vX.Y.Z` tag with a GitHub
release carrying its section; the `grav` line in `grav.toml` says which
compiler versions it builds with.

## Unreleased

## 0.2.1 (2026-11-27)

### Fixes

- A MySQL, MariaDB or PostgreSQL connection the server dropped while it sat idle (MySQL's
  `wait_timeout`, 8 hours by default; a restart; a proxy's or firewall's idle limit;
  PostgreSQL's `idle_session_timeout`) reconnects on its next use instead of failing every
  query with "server has gone away". After `checkIdleMs` (30 s) without use, the next query
  pings the server first; a connection in a transaction never reconnects (the transaction
  would be lost without a word), and statements made before keep working (PostgreSQL ones
  are prepared again). `reconnects` counts how often it happened.

## 0.2.0 (2026-11-20)

### Requirements

- Needs grav 1.9.0 or later, and uses the 0.2.0 releases of the packages it
  depends on.

## 0.1.1 (2026-11-19)

### Performance

- `quote()` skips escaping a name that holds no quote character (nearly all).

## 0.1.0 (2026-11-19)

The first tagged release. Needs grav 1.7.0 or newer.

Databases for [Grav](https://github.com/venomous-maker/grav): connections, prepared statements with `?` placeholders, results, transactions with savepoints, batched inserts, and a connection pool for threads. The drivers are **SQLite**, **MySQL/MariaDB** and **PostgreSQL**, behind the same API, so code written against one moves to the others. It is the base for an ORM: rows fill objects through Grav's reflection (`reflect(User).fields()`).
