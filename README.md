# grav-db

Databases for [Grav](https://github.com/venomous-maker/grav): connections,
prepared statements with `?` placeholders, results, transactions with
savepoints, and a connection pool for threads. The first driver is **SQLite**;
**MySQL/MariaDB** and **PostgreSQL** come next, behind the same API, so code
written against one moves to the others. It is the base for an ORM: rows fill
objects through Grav's reflection (`reflect(User).fields()`).

## Install

From GitHub:

```bash
grav add github.com/venomous-maker/grav-db          # the newest tag, or the default branch
grav add github.com/venomous-maker/grav-db@v1.0.0   # a given tag
```

From a checkout on your machine (used as it is, so edits show at once):

```bash
grav add ../grav-db
```

which writes `grav-db = { path = "../grav-db" }` to your `grav.toml`. Then:

```grav
import "std"
import "grav-db"

fn main() {
    let db = Sqlite.open("app.db")
    db.exec("CREATE TABLE IF NOT EXISTS users (id INTEGER PRIMARY KEY, email TEXT UNIQUE, age INTEGER)")
    let id = db.insert("INSERT INTO users (email, age) VALUES (?, ?)", ["ada@example.com", 36])
    for (row in db.query("SELECT email, age FROM users WHERE age > ?", [18])) {
        print(row.getString("email") + " " + "${row.getInt("age")}")
    }
    db.transaction(fn() { db.execute("UPDATE users SET age = age + 1 WHERE id = ?", [id]) })
}
```

SQLite needs the system's `libsqlite3` (loaded when the first database is
opened; `Sqlite.available()` says whether it could be).

## The API

**Opening.** `Sqlite.open(path)` (created if missing; `":memory:"` for a
private in-memory database), `Sqlite.openReadOnly(path)`, `Sqlite.version()`.
A connection waits up to 5 s for another connection's lock before failing
(`setBusyTimeout(ms)`); for several writers on one file, use WAL mode
(`PRAGMA journal_mode=WAL`) and a `Pool`.

**Arguments** go in a list, one per `?`: numbers, bools, strings, `Bytes`,
`null` and `T?` values (an enum goes in as its number). The count must match
the placeholders (`DbError` otherwise).

**Reading.**

| Call | Gives |
|------|-------|
| `query(sql, args)` | a `ResultSet` of every row (`for (row in ...)`, `size()`, `get(i)`, `first()`) |
| `queryOne(sql, args)` | the first row or `null` (reads no further) |
| `queryValue(sql, args)` | the first column of the first row, or `null` |
| `each(sql, args, fn(row))` | each row as it is read, nothing kept (for big results) |

A `Row` gives `get(name)` (an `any`; `null` for SQL NULL), `at(i)`,
`isNull(name)`, `getString`, `getInt` (a `long`), `getFloat`, `getBool`,
`getBytes`, `has(name)`, `columns()` and `toMap()`. A column that is not there
is a `ValueError`. SQLite values come out as `long` (INTEGER), `float` (REAL),
`string` (TEXT), `Bytes` (BLOB) and `null`.

**Writing.** `execute(sql, args)` returns the rows changed, `insert(sql, args)`
the new row's id, `exec(sql)` runs a script of statements without arguments.

**Prepared statements.** `prepare(sql)` parses once; `run(args)`,
`query(args)`, `queryOne(args)` and `each(args, f)` reuse it; `close()` frees
it. Connections also keep the statements they run prepared, so repeating the
same SQL through `query` or `execute` costs no parsing.

**Transactions.** `transaction(fn)` commits when the function returns and rolls
back when it throws (the exception goes on). A transaction inside another is a
savepoint: only its own changes are undone. `begin()`, `commit()`,
`rollback()` and `inTransaction()` do it by hand.

**Errors** are `DbError`s with the database's message, its `code` (SQLite's
extended code: 2067 for UNIQUE, 787 for a foreign key, 5 for "database is
locked") and the `sql`. Using a closed connection is a `StateError`.

**Threads.** A connection is for one thread at a time. `new Pool(max, factory)`
lends connections: `pool.with(fn(c) { ... })` takes one (opening it if none is
free, waiting while `max` are out) and gives it back afterwards, rolling back a
transaction left open. `acquire()` / `release(c)` do the same by hand.

## Speed and memory

On a 4-core laptop (i5-8350U), against Python's `sqlite3` module (written in C)
on the same in-memory database:

| | grav-db | Python |
|---|---|---|
| insert 100,000 rows in a transaction | 0.10 s | 0.13 s |
| read 100,000 rows of 4 columns | 0.05 s | 0.12 s |
| 20,000 lookups by primary key | 0.03 s | 0.06 s |
| peak memory, holding all 100,000 rows | 26 MiB | 35 MiB |

A result keeps its values in one array, row after row (a `Row` is a view of its
part), and each row is filled in C. `each()` streams rows in constant memory.

## Tests

```bash
grav test tests/sqlite.grav
```

They cover values in and out, NULLs and blobs, errors, transactions and
savepoints, the statement cache, streaming, a pool shared by threads on a file,
and rows into objects through reflection.

## License

MIT: see [LICENSE](LICENSE).
