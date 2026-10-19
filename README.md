# iyi-pg

A native PostgreSQL driver for [iyi](https://iyi.dev): a port of
[crystal-pg](https://github.com/will/crystal-pg) 0.31.0, written in iyi with no
Crystal code and no Crystal standard library underneath. It is a driver for
[iyi-db](https://github.com/iyilang/db), the common database interface, and
speaks the PostgreSQL wire protocol (version 3) itself.

It keeps crystal-pg's shape - `postgres://` URIs, binary results decoded by
type OID, arrays, `numeric`, `interval`, the geometric types, LISTEN/NOTIFY,
COPY, notices, SCRAM-SHA-256 and md5 authentication, TLS, logical
replication - and changes what iyi's rules require: errors are values, types
are closed, and tasks are structured.

## Requirements

iyi 0.18.0 or later, and iyi-db 0.2.0 (required by this package's `iyi.mod`).
Tested against PostgreSQL 16.

## Installation

Add it to a project under the short name `pg`, and iyi-db under `db`, whose
`Db.open` a program calls:

```sh
iyi get github.com/iyilang/pg --as pg
iyi get github.com/iyilang/db --as db
```

Files then `import pg` for the root module - `Pg.connect`,
`Pg.connect_listen`, `Pg.connect_replication`, `Pg::VERSION` - and its other
modules as `pg/pg/...`: the short name, then the module's path in the
package. Importing `pg` (or `pg/pg/core`) registers the driver for
`postgres://` and `postgresql://`. A package required only by another is
not named in a program's own imports, so `import db` needs the second line.

## Usage

```iyi
module app

import db
import db/db/errors::{DBError}
import pg

def contacts : Nil | DBError
  Db.open("postgres://user:pass@127.0.0.1/mydb") do |db|
    db.exec("create table contacts (name text, age integer)")!
    db.exec("insert into contacts values ($1, $2)", "John Doe", 30)!

    puts db.scalar("select max(age) from contacts")! # => 30

    db.query("select name, age from contacts order by age desc") do |rs|
      rs.each do
        puts "#{rs.read(String)!} (#{rs.read(Int32)!})"
      end!
    end!
  end
end

case contacts
in Nil     then puts "done"
in DBError then puts "failed: #{it.message}"
end
```

The URI's query takes iyi-db's pool options (`max_pool_size`, ...) and the
connection's: `application_name`, `sslmode`, `sslrootcert`, `sslcert`,
`sslkey`, `ssl_min_protocol_version`, `channel_binding`, `auth_methods`,
`replication`, and `host` for a Unix socket directory. Missing parts come
from `PGHOST`, `PGPORT`, `PGDATABASE`, `PGUSER`, `PGPASSWORD`, `PGAPPNAME`
and the password file (`~/.pgpass`, `%APPDATA%\postgresql\pgpass.conf` on
Windows, or `PGPASSFILE`), as libpq takes them; the user is the account the
program runs as, and the database the user's name. A host is a name,
resolved by iyi's own resolver, or an address. A parameter this driver does
not know is logged as ignored (iyi-db's pool options are known).

A server error is a `PQError` (module `pg/pq/errors`) whose `fields` hold
what the server sent: `err.field_message(:code)`, `:detail`, `:hint`, ...

### TLS

Over TCP, a session is encrypted with iyi's `std/tls` as `sslmode` says,
libpq's way: `disable` never; `prefer`, the default, when the server takes
TLS and its certificate is trusted, unencrypted otherwise; `allow`
unencrypted, and over TLS when the server refuses that; `require`,
`verify-ca` and `verify-full` always. A Unix socket is never encrypted.

```iyi
Db.open("postgres://user:pass@db.example.com/mydb?sslmode=verify-full&sslrootcert=/etc/ssl/pg-root.crt")
```

`std/tls` checks every server's certificate - its chain to a root, and the
host name - and cannot be told not to, so this driver checks it whatever the
`sslmode`, where libpq checks nothing for `require` and `prefer` and no host
name for `verify-ca`. The roots are the ones in `sslrootcert`, or in
`~/.postgresql/root.crt` (`%APPDATA%\postgresql\root.crt` on Windows); for
`require` and `prefer` without either, and for `sslrootcert=system` (which
means `verify-full` unless an `sslmode` is given), the system's.
`verify-ca` and `verify-full` without a root file are refused, as libpq
refuses them. A server whose certificate is self-signed needs its
certificate, or its root, in `sslrootcert`; with `prefer` it is otherwise
reached unencrypted, after a handshake that fails.

A server that asks for a client certificate is sent the one in `sslcert`
and `sslkey` (`~/.postgresql/postgresql.crt` and `postgresql.key`): PEM, the
key Ed25519, P-256 or RSA, unencrypted. TLS 1.3 is spoken, and 1.2 with a
server that has nothing newer, unless `ssl_min_protocol_version=TLSv1.3`.

Over TLS, SCRAM-SHA-256-PLUS binds the password exchange to the channel
(`tls-server-end-point`) when the server offers it, so a man in the middle
who ends TLS on both sides cannot relay it. `channel_binding=require`
refuses a session without it - a server that asks for md5, a password, or
none - and `channel_binding=disable` never binds.

In iyi 0.18.0, a handshake run as a task in a debug build can overrun the
task's 256 KiB stack - `std/ecdsa`'s check of one signature takes about
230 KB of it in a debug build - and the program dies of a memory fault or
reads a certificate as untrusted
([iyi#130](https://github.com/iyilang/iyi/issues/130)). A `--release`
build, or the handshake on the main fiber, is not affected.

### Listen/Notify

There are two ways to listen for notifications. For docs on `NOTIFY`, please
read <https://www.postgresql.org/docs/current/static/sql-notify.html>.

1. Any connection can be given a callback to run on notifications. However they
   are only received when other traffic is going on.
2. A special listen-only connection can be established for instant notification
   processing with `Pg.connect_listen`. Its read loop runs as a task of
   the group passed to it, and ends when the connection is closed.

```iyi
group do |g|
  conn = Pg.connect_listen("postgres:///", ["a", "b"], g) do |n|
    puts "    got: #{n.payload} on #{n.channel}"
  end.or_panic
  # ...
  conn.close
end
```

Without a group, `Pg.connect_listen(url, channels) { |n| ... }` reads in
place until the connection is closed. `examples/listen_notify.iyi` is the
whole program.

### Arrays

The array types (bool, "char", int2, int4, int8, float4, float8, text,
numeric, timestamp, timestamptz, date, uuid) read as nested `Array(DBValue)`,
or as exactly the type asked for:

```iyi
db.query_one("select ARRAY[1, null, 3]", as: Array(Int32 | Nil))! # => [1, nil, 3]
db.query_one("select '{hello, world}'::text[]", as: Array(String))! # => ["hello", "world"]
```

An array argument is sent in the array text format.

### Notices and errors

```iyi
Db.connect("postgres:///") do |cnn|
  pg = cnn.as(PGConnection) # import pg/pg/core::{PGConnection}
  pg.on_notice { |x| puts "pgSQL #{x}" }
  pg.exec("SELECT foo('yes')")!            # => pgSQL NOTICE:  Glad we agree!
  case err = pg.exec("SELECT foo('no')")
  in PQError    then puts err.message      # => You know nothing John Snow!
  in DBError    then puts "other: #{err.message}"
  in ExecResult then nil
  end
end
```

### COPY

```iyi
writer = pg.exec_copy("COPY table FROM STDIN")!
writer << data
writer.close!
data = pg.exec_copy("COPY table TO STDOUT")!.gets_to_end!
```

### Logical replication

A `Handler` subclass receives the WAL messages of a publication through a
replication slot (`pgoutput`, protocol version 1, binary tuples). The
connection sends standby status updates every `keepalive_interval` and
reconnects, resuming at the last flushed position, when it drops.

```iyi
import pg/pg/replication::{Handler, Connection}
import pg/pg/replication/messages::*

class MyHandler < Handler
  def received(data : XLogData, connection : Connection, &) : Nil
    yield
    connection.last_wal_byte_flushed = data.wal_end
    connection.last_wal_byte_applied = data.wal_end if data.message.is_a?(Commit)
  end

  def received(msg : Insert) : Nil
    puts "insert into #{msg.oid}"
  end
end

group do |g|
  conn = Pg.connect_replication(url, handler: MyHandler.new,
    publication_name: "my_publication", slot_name: "my_slot", group: g).or_panic
  # ...
  conn.close
end
```

The server needs `wal_level=logical`. Without `group:`,
`connect_replication` streams in place until the connection is closed.

## Supported datatypes

| PostgreSQL | read as |
|---|---|
| text, varchar, bpchar, name, xml, unknown | `String` |
| "char" | `Char` |
| boolean | `Bool` |
| int2, int4, int8, oid, regtype, pg_lsn | `Int16`, `Int32`, `Int64`, `UInt32`, `Int32`, `UInt64` |
| float4, float8 | `Float32`, `Float64` |
| timestamptz, timestamp, date | `Time` (UTC) |
| json, jsonb | `JSONValue`; `read(JSON::Any)` parses it |
| uuid | `UUIDValue`; `read(UUID)` answers the `std/uuid` value |
| bytea, and any OID without a decoder | `Bytes` |
| numeric | `Numeric` (1) |
| interval | `Interval` (2) |
| point, line, lseg, box, path, polygon, circle | `Point`, `Line`, `LineSegment`, `Box`, `Path`, `Polygon`, `Circle` |
| record, and registered composite types | `Record` (3) |
| the arrays above | `Array(DBValue)` or a typed `Array(T)` |

1: `Numeric` keeps every digit, and reads NaN and (PostgreSQL 14+)
`Infinity` and `-Infinity`. `to_f` is approximate; `to_big_d` and `to_big_r`
(and `read(BigDecimal)`, `read(BigRational)`, which refuse NaN and the
infinities) are exact. crystal-pg needed LibGMP and an extra require for
those; iyi's `std/big` needs neither.

2: A PostgreSQL interval does not map onto one iyi type. `Interval#to_span`
answers a `Span` (with `approx_months:` for intervals with months), and
`to_month_span` a `MonthSpan`.

3: A composite value reads as a `Record` of its fields, each decoded by its
own type. A named composite type has its own OID; register it with
`Decoders.register_decoder RecordDecoder.new([oid])`.

Values bound as arguments are iyi-db's `DBValue`s: the scalars, `Time` (to
the microsecond, PostgreSQL's precision), `Span`, `Bytes`, arrays, and this
driver's value classes (`Numeric`, `Interval`, the geometric types,
`UUIDValue`, `JSONValue`, `Record`). A statement takes at most 65535 of
them.

## Authentication methods

By default this driver will accept `scram-sha-256-plus`, `scram-sha-256` and
`md5`, as well as `trust`. However `cleartext` is disabled by default. You can
control exactly which auth methods the client will accept by passing in a
comma separated list to the `auth_methods` parameter, for example

```iyi
Db.open("postgres://example.com/dbname?auth_methods=cleartext,md5,scram-sha-256")
```

**DO NOT TURN `cleartext` ON UNLESS YOU ABSOLUTELY NEED IT!** Merely by having
this option enabled exposes a postgres client to downgrade man-in-the-middle
attacks, even if the server is configured to not support cleartext. Even if you
use TLS, you are not safe unless you are fully verifying the server's cert, as
the attacker can terminate TLS and re-negotiate a connection with the server.

```
client                     attacker                     server
----------------------------------------------------------------------------
I want to connect \
                   \->  intercepts, forwards
                        I want to connect \
                                           \----->  receives connection request

                                                  / I support scram and/or md5 only
                        intercepts, sends      <-/
                     /  I only support cleartext
receives attacker <-/
claiming server
only supports cleartext
sends password because
cleartext enabled \
                   \->  receives clear password,
                        negotiates scram/md5
                        with real server      \
                                               \--> accepts scram/md5 auth

```

It is a mistake for any driver to support cleartext by default, and it's a
mistake that postgres continues to have this as an option at all.

## Modules

| module | what it holds |
|---|---|
| `pg` (the root module) | `Pg.connect`, `connect_listen`, `connect_replication`, `ListenConnection`, `VERSION` |
| `pg/core` | `PGDriver`, `PGConnection`, `PGStatement`, `PGResultSet`, `PGConnectionBuilder`; registers the driver |
| `pg/decoder` | `Decoder` and one per type, `Decoders` (`from_oid`, `register_decoder`), `UUIDValue`, `JSONValue` |
| `pg/numeric`, `pg/interval`, `pg/geo`, `pg/record` | the value classes |
| `pg/escape` | `EscapeHelper`: `escape_identifier`, `escape_literal` |
| `pg/copy_result` | `CopyResult` |
| `pg/replication` | logical replication: `Connection`, `Handler` |
| `pg/replication/messages` | the replication frames and WAL messages: `XLogData`, `Begin`, `Insert`, ... |
| `pg/errors` | `PGError`, `RuntimeError` |
| `pg/pq/connection` | `PQConnection`, the protocol session |
| `pg/pq/conninfo`, `pg/pq/pgpass` | `ConnInfo`, `PgPass` |
| `pg/pq/ssl` | the `TlsConfig` a `ConnInfo` asks for: roots, client certificate; the channel binding's certificate hash |
| `pg/pq/frame`, `pg/pq/param`, `pg/pq/stream`, `pg/pq/scram` | messages, parameter encoding, the buffered socket or TLS session, SCRAM |
| `pg/pq/errors` | `ConnectionError`, `StreamError`, `PQError`, `ErrorField` |

## What is different from crystal-pg

| crystal-pg | iyi-pg | why |
|---|---|---|
| raises `PQ::PQError`, `PQ::ConnectionError`, `PG::RuntimeError`, `IO::Error` | answers them as `DBError`s; a socket failure is `StreamError`, which a statement answers as iyi-db's `ConnectionLost` | errors are values (SPEC.md III.1) |
| `PG::Connection`, `PG::Statement`, `PG::ResultSet`, `PG::Driver` | `PGConnection`, `PGStatement`, `PGResultSet`, `PGDriver` in one module | the module imports iyi-db's `Connection`, ... by those names, and the four refer to one another (R-1) |
| `PQ::ConnInfo.new(uri)`, `.new(hash)` raise | `ConnInfo.from_uri`, `.from_hash`, `.from_conninfo_string` answer `ConnInfo \| DBError`; `.new(...)` takes typed parts | a constructor has no error to answer |
| `rs.read` answers `UUID`, `JSON::PullParser` | `UUIDValue`, `JSONValue`; `read(UUID)`, `read(JSON::Any)` | `DBValue` is closed (iyi-db) |
| `PG.connect_listen(url, *channels, blocking: false)` spawns a fiber | `connect_listen(url, channels, group)` spawns in the caller's group; without a group it reads in place | a task cannot outlive its group (III.4.1) |
| `timestamptz` read in the session's time zone; `time_zone`, `clear_time_zone_cache` | `Time` in UTC, the same instant | iyi's `Time` has no zones |
| `Time::MonthSpan` | `MonthSpan` | `std/time` has none |
| `PG::Numeric#to_big_d`/`#to_big_r` behind `require "pg_ext/..."` and LibGMP | always there | `std/big` is pure iyi |
| `PG::Record(*T)`, which no decoder produced | `Record`, decoded for `record` and registered composite OIDs | so it can be read at all |
| `CopyResult < IO` | `CopyResult` with `read`, `read_char`, `gets_to_end`, `write`, `<<`, `close` answering errors | `IyiIO` cannot answer one |
| enums bound as their value | pass `.value` | an enum is not a `DBValue` |
| `sslmode=require` and `prefer` encrypt without checking the certificate; `verify-ca` checks no host name | every certificate is checked, chain and host name: against `sslrootcert` or `root.crt`, else the system's roots | `std/tls` has no way to turn the checks off |

## Development

```sh
DATABASE_URL=postgres://postgres:password@127.0.0.1/iyi_pg_test iyi test pg
```

The tests are crystal-pg's specs, ported, beside the modules they test, and
iyi-db's shared driver specs (`pg/driver_specs_test.iyi`). They read
`DATABASE_URL`, `postgres:///` by default. The replication tests that need
`wal_level=logical` are skipped, with a note, on a server without it, and
the md5 and cleartext authentication tests, which need matching `pg_hba.conf`
lines, run only with `RUN_AUTH_SPECS` set. The TLS tests
(`pg/pq/ssl_test.iyi`) run when `SSL_ROOT_CERT` names the root of the
server's certificate for 127.0.0.1, and its client certificate test with
`RUN_AUTH_SPECS`, `SSL_CLIENT_CERT` and `SSL_CLIENT_KEY` as well; the file
says which `pg_hba.conf` lines they need. Run the rest against a server
without TLS: their examples run as tasks, and connect with the default
`sslmode=prefer` (see TLS above). `examples/` holds crystal-pg's examples,
ported.

## Contributors

crystal-pg is the work of [Will Leinweber](https://github.com/will) and its
[contributors](https://github.com/will/crystal-pg/graphs/contributors); this
port keeps its licence (LICENSE).
