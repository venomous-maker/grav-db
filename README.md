# grav-db

Databases for [Grav](https://github.com/venomous-maker/grav): connections,
prepared statements with `?` placeholders, results, transactions with
savepoints, batched inserts, and a connection pool for threads. The drivers
are **SQLite**, **MySQL/MariaDB** and **PostgreSQL**, behind the same API, so
code written against one moves to the others. It is the base for an ORM: rows fill
objects through Grav's reflection (`reflect(User).fields()`).

## Install

grav-db needs a Grav compiler with `Decimal`, microsecond `DateTime`s and
reflection that creates objects (`ClassInfo.create`), from November 2026 on.

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

For a server, open a URL instead; the rest is the same:

```grav
let my = MySql.open("mysql://app:secret@localhost/shop")
let pg = Postgres.open("postgresql://app:secret@localhost/shop")
```

SQLite needs the system's `libsqlite3`, MySQL a client library (`libmariadb`,
as in Debian's `libmariadb3`, or `libmysqlclient`) and PostgreSQL `libpq`
(`libpq5`). Each is loaded when the first connection is made;
`Sqlite.available()`, `MySql.available()` and `Postgres.available()` say
whether it could be.

## The API

**Opening.** `Sqlite.open(path)` (created if missing; `":memory:"` for a
private in-memory database), `Sqlite.openReadOnly(path)`, `Sqlite.version()`.
A connection waits up to 5 s for another connection's lock before failing
(`setBusyTimeout(ms)`); for several writers on one file, use WAL mode
(`PRAGMA journal_mode=WAL`) and a `Pool`.

**Connecting to MySQL or MariaDB.** `MySql.open(url)` takes
`mysql://user:password@host:port/database` (`mariadb://` too; the port is 3306
unless given), with `?socket=/run/mysqld/mysqld.sock` and `?timeout=seconds`.
Escape any of `:@/?#` in the password as `%XX`. `localhost` means the local
Unix socket. `MySql.connect(options)` takes a `MySqlOptions` (`host`, `port`,
`user`, `password`, `database`, `socket`, `connectTimeout`) instead. A
connection talks utf8mb4, has `ping()`, `serverVersion()` and `changes()`, and
refuses `LOAD DATA LOCAL`.

**Connecting to PostgreSQL.** `Postgres.open(url)` takes
`postgresql://user:password@host:5432/database` (`postgres://` too), with
libpq's options in the query (`?sslmode=require`, `?connect_timeout=5`), or
libpq's `host=... dbname=...` form. It waits 10 s at most for the server unless
told otherwise, and talks UTF-8. A connection has `ping()`, `serverVersion()`
and `changes()`; the server's NOTICEs are not printed.

**Arguments** go in a list, one per `?`: numbers, bools, strings, `Bytes`,
`DateTime`s, `Decimal`s, `JsonValue`s, `null` and `T?` values (an enum goes in
as its number; a `DateTime` as its local time, `2024-02-28 09:30:15.250000`,
with its zone for PostgreSQL so a `timestamptz` gets the exact moment; a
`Decimal` as its digits, so it compares exactly; a `JsonValue` as its JSON
text). The count must match
the placeholders (`DbError` otherwise). PostgreSQL also takes `$1`, `$2`, ...
written out (not mixed with `?`), and `??` for a literal `?` (jsonb's `?|`);
it sends the arguments apart from the SQL, untyped so the server infers each
from where it goes (`Bytes` as bytea), so `SELECT ?` gives text back: write
`?::int8` when there is nothing to infer from.

**Reading.**

| Call | Gives |
|------|-------|
| `query(sql, args)` | a `ResultSet` of every row (`for (row in ...)`, `size()`, `get(i)`, `first()`) |
| `queryOne(sql, args)` | the first row or `null` (reads no further) |
| `queryValue(sql, args)` | the first column of the first row, or `null` |
| `each(sql, args, fn(row))` | each row as it is read, nothing kept (for big results) |

A `Row` gives `get(name)` (an `any`; `null` for SQL NULL), `at(i)`,
`isNull(name)`, `getString`, `getInt` (a `long`), `getFloat`, `getBool`,
`getBytes`, `getDateTime`, `getDecimal`, `getJson`, `has(name)`, `columns()`
and `toMap()`. A column that is not there is a `ValueError`.

**Dates and times** come out as Grav's `DateTime`, to the microsecond: MySQL's
DATE, DATETIME and TIMESTAMP, PostgreSQL's date, timestamp and timestamptz,
and SQLite's text in a column declared DATE, DATETIME or TIMESTAMP. A date is
its local midnight, a time without a zone is local time, and a timestamptz is
the exact moment whatever the session's zone. What a `DateTime` cannot hold
stays a string: MySQL's `0000-00-00`, PostgreSQL's `infinity` and BC dates, and
times of day (TIME) and intervals, which are not moments. `getDateTime(name)`
also parses a date held as text (`"2024-02-28 09:30"`).

**Exact decimals** come out as Grav's `Decimal` (up to 38 digits, keeping the
column's scale: `19.90` stays `19.90`): MySQL's DECIMAL, PostgreSQL's numeric,
and what SQLite holds in a column declared DECIMAL or NUMERIC. A number a
`Decimal` cannot hold (PostgreSQL's NaN, or more than 38 digits) stays a
string. `getDecimal(name)` also converts whole numbers, floats and numeric
text; `getFloat` gives the nearest float, and `getInt` the value when it is a
whole number (a `ValueError` for `19.99`).

**JSON** comes out parsed, as a `JsonValue`: PostgreSQL's json and jsonb,
MySQL's JSON, MariaDB's JSON (a LONGTEXT the server marks as JSON), and what
SQLite holds in a column declared JSON. `getJson(name)` also parses text.

SQLite values come out as `long` (INTEGER), `float` (REAL), `string` (TEXT),
`Bytes` (BLOB), `DateTime`, `Decimal` and `JsonValue` (by the column's declared
type, see above) and `null`. MySQL values come out as `long` (the integer
types, YEAR and BIT; a `BIGINT UNSIGNED` above the `long` range as a `ulong`),
`float` (FLOAT, DOUBLE), `Bytes` (BINARY, VARBINARY and the BLOBs),
`DateTime`, `Decimal` (DECIMAL), `JsonValue` (JSON), `null`, and `string` for
the rest (text, TIME, ENUM, SET). PostgreSQL values come out as `long`
(smallint, integer, bigint, oid), `float` (real, double precision, including
NaN and infinities), `bool` (boolean), `Bytes` (bytea), `DateTime`, `Decimal`
(numeric), `JsonValue` (json, jsonb), `null`, and `string` for the rest (text,
time, interval, uuid, arrays in their text form).

**Into objects.** Grav's reflection fills a model from a row:
`reflect(User).create()` makes one and `field.set(user, row.get(col))` stores
each value, converting where it fits (a DECIMAL's `"42"` into an `int`, MySQL's
`1` into a `bool`), so a column's type need not match the field's exactly. Inside
a generic `Repository<T>`, `reflect(T)` is the model class.

**Writing.** `execute(sql, args)` returns the rows changed, `insert(sql, args)`
the new row's id, `exec(sql)` runs a script of statements without arguments.
PostgreSQL has no last insert id: `insert` returns the first column of the
first row when the SQL has a `RETURNING` clause (`... RETURNING id`, one round
trip), else `lastval()` (the last value this session took from a sequence: a
SERIAL or IDENTITY column).

**Many rows.** `executeMany(sql, rows)` runs `sql` once for each list of
arguments in `rows`, all in one transaction (a savepoint inside one), so
either every row goes in or none does; it returns the rows changed. MySQL
sends an INSERT whose `?`s are in the one tuple after VALUES as multi-row
INSERTs of up to 512 KiB each; PostgreSQL pipelines the runs, 256 to a round
trip; SQLite runs its prepared statement in a loop.

**Prepared statements.** `prepare(sql)` parses once; `run(args)`,
`query(args)`, `queryOne(args)` and `each(args, f)` reuse it; `close()` frees
it. Connections also keep the statements they run prepared, so repeating the
same SQL through `query` or `execute` costs no parsing (a call with the very
same string, such as a literal, finds its statement without hashing the SQL).
A PostgreSQL statement is prepared on the server when it first runs, and
deallocated there when it is closed or drops out of the cache. A MySQL statement finds
its `?`s once (not those in quotes, backticks or comments) and writes each
run's arguments into the SQL as escaped literals: strings through the server's
own escaping for the connection's character set, `Bytes` as hex. A query is
one round trip, and the server keeps nothing between runs. MySQL and
PostgreSQL read a query's rows whole before `each()` hands them out, so
another query can run inside it.

**Transactions.** `transaction(fn)` commits when the function returns and rolls
back when it throws (the exception goes on). A transaction inside another is a
savepoint: only its own changes are undone. `begin()`, `commit()`,
`rollback()` and `inTransaction()` do it by hand.

**Errors** are `DbError`s with the database's message, its `code` (SQLite's
extended code: 2067 for UNIQUE, 787 for a foreign key, 5 for "database is
locked"; MySQL's error number: 1062 for a duplicate key, 1452 for a foreign
key, 1213 for a deadlock, 2002 and 2003 for a server it cannot reach; 0 from
PostgreSQL), its SQLSTATE in `state` when the database gives one ("23505" for
a unique violation in PostgreSQL, "23000" in MySQL; "" from SQLite) and the
`sql`. Using a closed connection is a `StateError`.

**Threads.** A connection is for one thread at a time. `new Pool(max, factory)`
lends connections: `pool.with(fn(c) { ... })` takes one (opening it if none is
free, waiting while `max` are out) and gives it back afterwards, rolling back a
transaction left open. `acquire()` / `release(c)` do the same by hand.

## Speed and memory

On a 4-core laptop (i5-8350U), against Python's fastest C-based clients doing
the same work, best of three. The machine was busy with other builds while
these ran, so the times are high for both; client CPU time (what each client
spends itself, server time aside) is the steadier comparison.

SQLite (Python's `sqlite3`), an in-memory database:

| | grav-db | Python |
|---|---|---|
| insert 100,000 rows one at a time, in a transaction | 0.27 s | 0.98 s |
| `executeMany` of 100,000 rows | 0.25 s | 0.34 s |
| read 100,000 rows of 4 columns | 0.08 s | 0.41 s |
| 20,000 lookups by primary key | 0.07 s | 0.13 s |
| client CPU, all of it | 0.80 s | 1.64 s |

MariaDB 11.8 (Python's `MySQLdb`, mysqlclient), over its Unix socket, InnoDB:

| | grav-db | Python |
|---|---|---|
| `executeMany` of 100,000 rows | 1.15 s | 2.14 s |
| read 100,000 rows of 4 columns | 0.17 s | 0.16 s |
| 20,000 lookups by primary key | 1.9 s | 2.3 s |
| 20,000 single inserts in a transaction | 2.2 s | 2.5 s |
| client CPU, all of it | 1.36 s | 3.46 s |

PostgreSQL 17 (Python's `psycopg` 3, which pipelines its `executemany`), over
its Unix socket:

| | grav-db | Python |
|---|---|---|
| `executeMany` of 100,000 rows | 1.66 s | 2.30 s |
| read 100,000 rows of 4 columns | 0.09 s | 0.12 s |
| 20,000 lookups by primary key | 1.25 s | 3.87 s |
| 20,000 single inserts in a transaction | 1.39 s | 4.44 s |
| client CPU, all of it | 1.55 s | 9.32 s |

grav-db prepares each statement on the server once and sends only its
arguments after that; psycopg sends the SQL with each call.

Memory, reading 100,000 rows of 4 columns and holding them (peak resident
size; in brackets, the same program that only opens the connection):

| | grav-db | Python |
|---|---|---|
| SQLite | 25 MiB (12) | 39 MiB (17) |
| MariaDB | 26 MiB (12) | 47 MiB (18) |

so the rows take 13–14 MiB in grav-db and 22–29 MiB in Python.

A result keeps its values in one array, row after row (a `Row` is a view of its
part), each row is filled in C, and arguments are bound in C. SQLite's
`each()` streams rows in constant memory; MySQL reads rows straight off the
connection into the result. Single-row statements against a server wait on
it, one round trip each, for any client: `executeMany` is the way to go
faster.

## Tests

```bash
grav test tests/sqlite.grav
GRAV_DB_MYSQL='mysql://user:password@localhost/grav_test' grav test tests/mysql.grav
GRAV_DB_POSTGRES='postgresql://user:password@localhost/grav_test' grav test tests/postgres.grav
```

They cover values in and out, NULLs and blobs, errors, transactions and
savepoints, the statement cache, streaming, a pool shared by threads, and rows
into objects through reflection, and `executeMany`. The server tests make and
drop their own `gdb_*` tables in the database the URL names. Without
`GRAV_DB_MYSQL` or `GRAV_DB_POSTGRES`, only the tests that need no server run
(URLs and connection errors). A throwaway PostgreSQL for them needs no setup
beyond the server's own programs:

```bash
initdb -D /tmp/pg -U grav --auth=trust && pg_ctl -D /tmp/pg -o "-k /tmp -p 54329" -w start
createdb -h /tmp -p 54329 -U grav grav_test
GRAV_DB_POSTGRES='postgresql://grav@/grav_test?host=/tmp&port=54329' grav test tests/postgres.grav
```

## License

MIT: see [LICENSE](LICENSE).
