# grav-db

Databases for [Grav](https://github.com/venomous-maker/grav): connections,
prepared statements with `?` placeholders, results, transactions with
savepoints, and a connection pool for threads. The drivers are **SQLite** and
**MySQL/MariaDB**, with **PostgreSQL** next, behind the same API, so code
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

For a MySQL or MariaDB server, open a URL instead; the rest is the same:

```grav
let db = MySql.open("mysql://app:secret@localhost/shop")
```

SQLite needs the system's `libsqlite3`, and MySQL a client library
(`libmariadb`, as in Debian's `libmariadb3`, or `libmysqlclient`). Each is
loaded when the first connection is made; `Sqlite.available()` and
`MySql.available()` say whether it could be.

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
`string` (TEXT), `Bytes` (BLOB) and `null`. MySQL values come out as `long`
(the integer types, YEAR and BIT; a `BIGINT UNSIGNED` above the `long` range
as a `ulong`), `float` (FLOAT, DOUBLE), `Bytes` (BINARY, VARBINARY and the
BLOBs), `null`, and `string` for the rest: text, DECIMAL (exact, so it is not
rounded through a float; `getFloat` converts it, `getInt` a whole one) and dates and
times.

**Writing.** `execute(sql, args)` returns the rows changed, `insert(sql, args)`
the new row's id, `exec(sql)` runs a script of statements without arguments.

**Prepared statements.** `prepare(sql)` parses once; `run(args)`,
`query(args)`, `queryOne(args)` and `each(args, f)` reuse it; `close()` frees
it. Connections also keep the statements they run prepared, so repeating the
same SQL through `query` or `execute` costs no parsing. A MySQL statement finds
its `?`s once (not those in quotes, backticks or comments) and writes each
run's arguments into the SQL as escaped literals: strings through the server's
own escaping for the connection's character set, `Bytes` as hex. A query is
one round trip, and the server keeps nothing between runs. MySQL reads a
query's rows whole before `each()` hands them out, so another query can run
inside it.

**Transactions.** `transaction(fn)` commits when the function returns and rolls
back when it throws (the exception goes on). A transaction inside another is a
savepoint: only its own changes are undone. `begin()`, `commit()`,
`rollback()` and `inTransaction()` do it by hand.

**Errors** are `DbError`s with the database's message, its `code` (SQLite's
extended code: 2067 for UNIQUE, 787 for a foreign key, 5 for "database is
locked"; MySQL's error number: 1062 for a duplicate key, 1452 for a foreign
key, 1213 for a deadlock, 2002 and 2003 for a server it cannot reach) and the
`sql`. Using a closed connection is a `StateError`.

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
part), and each row is filled in C. SQLite's `each()` streams rows in constant
memory.

Against MariaDB 11.8 on the same machine (over its Unix socket, InnoDB),
compared with Python's `MySQLdb` (mysqlclient, written in C), best of three:

| | grav-db | Python |
|---|---|---|
| insert 100,000 rows in a transaction, one at a time | 11.6 s | 13.8 s |
| read 100,000 rows of 4 columns | 0.11 s | 0.15 s |
| 20,000 lookups by primary key | 1.6 s | 2.0 s |
| peak memory, holding all 100,000 rows | 35 MiB | 47 MiB |

Single-row inserts and lookups wait on the server, one round trip each, for
both clients.

## Tests

```bash
grav test tests/sqlite.grav
GRAV_DB_MYSQL='mysql://user:password@localhost/grav_test' grav test tests/mysql.grav
```

They cover values in and out, NULLs and blobs, errors, transactions and
savepoints, the statement cache, streaming, a pool shared by threads, and rows
into objects through reflection. The MySQL tests make and drop their own
`gdb_*` tables in the database the URL names. Without `GRAV_DB_MYSQL`, only
the tests that need no server run: URLs and connection errors.

## License

MIT: see [LICENSE](LICENSE).
