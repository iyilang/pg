## Unreleased

A statement cache, and the driver's module state made safe to reach from
every worker.

* Each connection keeps the statements it runs prepared on the server
  ([#1](https://github.com/iyilang/pg/issues/1)). A query's first execution
  on a connection parses it under a name and describes it, in the same
  round trip as the execution; every later one sends only Bind, Execute and
  Sync, where each sent Parse, Bind, Describe, Execute and Sync before. The
  cache is an LRU of `statement_cache_capacity` statements (512, pgx's
  default), a Close for each one given up going with the next exchange.
  `statement_cache_capacity=0`, or `prepared_statements_cache=false`, turns
  it off; unprepared statements are parsed every time, as before.
  - A kept statement the server refuses to bind as out of date - 26000
    (dropped), 0A000 (`cached plan must not change result type`), class 42,
    or class 22 for one with parameters - is prepared anew and the query run
    again, once, outside a transaction block; inside one the error is
    answered. `DISCARD ALL` and `DEALLOCATE ALL` empty the cache, and DDL
    on the same connection (`CREATE`, `ALTER`, `DROP`) gives up every kept
    statement: one keeps the parameter types inferred when it was prepared,
    and the suite's `timestamptz` read back three hours off on a server in
    Europe/Istanbul when its INSERT kept the `timestamp` parameter of the
    table dropped before.
  - `PQConnection#statement_cache`, `#parse_count` (the Parse messages a
    session sent) and `#transaction_status` (the last ReadyForQuery's).
  - Measured on macOS 26.6, Apple M2 Pro, 10 cores (the user's machine),
    against a throwaway PostgreSQL 14.19 there, with clients built by iyi
    master befaa8bc4. A SELECT of four columns by primary key, 1000 times
    on one connection, sent per query: before, a Parse, a Bind, a Describe
    portal, an Execute and a Sync; after, a Bind, an Execute and a Sync, and
    one Parse and one Describe statement in all. The same SELECT in a loop,
    2 s warm-up then 10 s measured, three runs each, alternating:

    | | queries/s (best of 3) | runs | postgres CPU per query |
    |---|---:|---|---:|
    | 1 connection, before | 15,550 | 11,734 / 14,393 / 15,550 | 23.8-27.2 us |
    | 1 connection, after | 21,739 | 21,739 / 18,728 / 18,945 | 13.7-15.0 us |
    | 10 connections, before | 52,610 | 52,130 / 52,610 / 51,335 | 35.7-37.0 us |
    | 10 connections, after | 72,543 | 69,769 / 72,242 / 72,543 | 21.8-22.9 us |

    Client and server shared the machine, which was at 100 % in every run;
    the client used 659-862 % of it, its collector's helpers spinning as
    they do on darwin at that iyi master.
* The module state a connection reaches is `Share`, so a program that opens
  and queries connections in parallel tasks gets no #148 reach warning from
  this driver (iyilang/iyi#148). `@[SafeReach]` is not used: it is in no
  released iyi (not 0.18.0 to 0.19.1), and this driver still builds with
  iyi 0.18.0. `iyi check` (iyi master befaa8bc4) of a program that opens
  connections and queries them in `spawn_parallel` tasks warned 12 times,
  9 of them about this driver's constants; it warns 3 times now, all about
  iyi-db's (`DRIVERS`, `AROUND_HOOKS`, `LOG`).
  - `ConnInfo`'s tables (`SSL_MODES`, `KNOWN_KEYS`, `ENV_DEFAULTS`, ...)
    and `RangeValue::QUOTED` are `List`s.
  - `LOG` is a `SharedLog`: `Log.for("pg")` behind a `Mutex`, std's `Logger`
    not being `Share`. Set its level or backend on `Log.for("pg")`.
  - The decoder registry is a `DecoderRegistry` behind a `Mutex`, and a
    connection reads its columns through a `DecoderTable` it holds - the
    OIDs sorted in a `List`, their decoders in another - taken again only
    after a registration, so reading a column takes no lock. A decoder is
    `Share`: `ArrayDecoder`, `RecordDecoder`, `RangeDecoder` and
    `MultirangeDecoder` keep their OIDs in a `List`, and `oids` answers a
    copy. A decoder of your own sets its fields in `initialize` only, of
    `Share` types.

## v0.3.1

On iyi-db 0.3.0, for its pool and closing fixes: `Database#close` closes
every connection, a closed pool refuses checkouts, a waiting checkout keeps
to `checkout_timeout`, and an error closing what a query opened is
answered. Nothing in this driver changes.

## v0.3.0

Types, ranges and connection options crystal-pg does not have.

* More types read: `time` as `Span`, `timetz` as `TimeTz`, `inet` and `cidr`
  as `Inet` (over `std/netip`), `macaddr`/`macaddr8` and `bit`/`varbit` as
  `String`, `money` as `Int64` in the minor unit. They read as `Bytes`
  before. `TimeTz` and `Inet` bind as arguments.
* The arrays of every type the driver reads: `varchar[]` (so
  `array_agg(varchar_col)`), `bpchar[]`, `name[]`, `xml[]`, `bytea[]`,
  `oid[]`, `json[]`/`jsonb[]`, `timestamptz[]`, `interval[]`, the geometric
  arrays, `record[]` and the arrays of the new types. They read as `Bytes`
  before. Typed reads (`Array(String)`, `Array(Interval)`, `Array(Bytes)`,
  ...) cover them.
* Range types (`pg/range`): the six built-in ranges read as `RangeValue`,
  their multiranges (PostgreSQL 14+) as `MultirangeValue`, and their arrays;
  both bind. `RangeDecoder` and `MultirangeDecoder` register a range type of
  one's own.
* `connect_timeout` (and `PGCONNECT_TIMEOUT`) bounds each connection
  attempt: the TCP connect, the TLS handshake, and startup and
  authentication. On Windows std/socket's connect ignores its timeout
  ([iyi#131](https://github.com/iyilang/iyi/issues/131)), so the connect is
  cancelled at the deadline.
* `options` (and `PGOPTIONS`) is sent as the session's command-line options.
* `PGSSLMODE`, `PGSSLROOTCERT`, `PGSSLCERT`, `PGSSLKEY`, `PGCHANNELBINDING`
  and `PGSSLMINPROTOCOLVERSION` are read, a parameter given winning; a value
  they cannot take is an `ArgumentError`.
* A `key=value` string takes single-quoted values with `\'` and `\\`, and
  spaces around `=`, as libpq does.
* `PGConnection#cancel` cancels the running statement from another task (a
  CancelRequest, unencrypted as libpq's before 17); `#backend_pid` is the
  server process's id.

## v0.2.1

Fixes from a comparison with crystal-pg, each with a test.

* The SCRAM client nonce comes from `Random::Secure`, as crystal-pg's does;
  it came from the seeded, guessable `Random`.
* `numeric` `Infinity` and `-Infinity` (PostgreSQL 14+) read as a `Numeric`
  (`infinite?`, `to_s`, `to_f`) where they panicked; `read(BigDecimal)` and
  `read(BigRational)` refuse them and NaN, which they read as zero.
* A `Time` argument is sent to the microsecond, cut: the server rounded the
  nine digits sent, and `23:59:59.999999999` became the next day
  (crystal-pg #252).
* COPY FROM STDIN sends as it goes, a buffer at a time, where it held the
  whole input until `close`. A write buffer grown for a large message is
  let go once it is sent.
* Two tasks sending on one connection - a replication keepalive beside the
  run loop's - send each message once and whole; a send that waited on the
  socket could have its bytes sent again by the other.
* A `Record` argument is written with its fields quoted, so commas,
  parentheses, quotes, backslashes and empty strings come through; `Bytes`
  and `Time` fields in their text forms.
* More than 65535 arguments, the protocol's limit, are refused; the count
  was sent cut to 16 bits.
* A malformed message from the server - a length under four, a body shorter
  than its fields, a negative length - is a `StreamError` (the connection
  lost), not a panic. So is a decoder that reads past its column.
* An unreadable password file is passed over with a warning, not a panic; on
  Windows the file is libpq's `%APPDATA%\postgresql\pgpass.conf`.
* A `PGPORT` that is not a number is an `ArgumentError`, not port 5432. The
  default user is the account the program runs as (`std/user`), not
  `$USER` or `"postgres"`, and the default database is the user's name,
  `PGUSER`'s included, as libpq has them. A connection parameter this driver
  ignores is logged; `ConnInfo.new` refuses an unknown `sslmode`.
* `version` reads a pre-release server's (`16beta1`, `17devel`) by its
  leading digits, where it answered 0.

## v0.2.0

On iyi 0.18.0, whose `std/tls` and resolver this release is built on; iyi
0.17.0 is no longer enough. On iyi-db 0.2.0.

* The entry point is the package's root module, as iyi-db 0.2.0's is:
  `import pg` and `Pg.connect`, `Pg.connect_listen`, `Pg.connect_replication`,
  `Pg::ListenConnection` and `Pg::VERSION`, where they were `import pg/pg/pg`
  and `Pg::Pg.connect`, ... The other modules are where they were. With
  iyi-db 0.2.0, `Db.open` and `Db.connect` (`import db`) are what were
  `Db::Db.open` and `Db::Db.connect`.
* TLS, over `std/tls`: `sslmode` `prefer` (the default) and `allow` negotiate
  it as libpq does - `prefer` goes on unencrypted when the server declines or
  the handshake fails, `allow` tries TLS when the server refuses an
  unencrypted session - and `require`, `verify-ca` and `verify-full` insist
  on it. A Unix socket is never encrypted. TLS 1.3, and 1.2 unless
  `ssl_min_protocol_version=TLSv1.3`.
* Every server certificate is checked, chain and host name, whatever the
  `sslmode`: `std/tls` cannot be told not to. The roots are `sslrootcert`'s,
  `~/.postgresql/root.crt`'s (`%APPDATA%\postgresql\root.crt` on Windows),
  or, for `require` and `prefer` without either and for
  `sslrootcert=system`, the system's. `sslrootcert=system` means
  `verify-full` unless an `sslmode` is given; `verify-ca` and `verify-full`
  without a root file are refused.
* Client certificates: `sslcert` and `sslkey`, or
  `~/.postgresql/postgresql.crt` and `postgresql.key`, sent to a server that
  asks.
* SCRAM-SHA-256-PLUS (`tls-server-end-point`) when the server offers it over
  TLS, and `channel_binding` (`prefer`, `require`, `disable`).
* Host names resolve: `std/socket` connects to a name through iyi's resolver.

### Known issue

* In a debug build of iyi 0.18.0 a TLS handshake run as a task can overrun
  the task's 256 KiB stack (`std/ecdsa` takes about 230 KB of it for one
  signature) and end in a memory fault or a certificate read as untrusted
  ([iyi#130](https://github.com/iyilang/iyi/issues/130)). A `--release`
  build, or the handshake on the main fiber, is not affected.

## v0.1.0

The first release of iyi-pg: crystal-pg 0.31.0 ported to iyi, in iyi, on
iyi-db 0.1.1, with no Crystal code left.

* `postgres://` and `postgresql://` URIs, a libpq `key=value` string, the
  `PG*` environment variables and the password file describe a connection
  (`ConnInfo`).
* The wire protocol is written here over `std/socket`: startup,
  SCRAM-SHA-256 (HMAC and PBKDF2 over `std/digest`), md5 and cleartext
  authentication with `auth_methods`, the extended query protocol with binary
  results, the simple query protocol (`exec_all`), COPY (`exec_copy`),
  notices, and LISTEN/NOTIFY (`Pg::Pg.connect_listen`).
* Decoders for every type crystal-pg reads, arrays as nested
  `Array(DBValue)` or a typed `Array(T)`, and `Decoders.register_decoder`.
* `Numeric#to_big_d` and `#to_big_r`, and `read(BigDecimal)`,
  `read(BigRational)`, without crystal-pg's `pg_ext` requires.
* `Record` reads `record` columns and registered composite types; crystal-pg's
  `PG::Record` had no decoder behind it.
* Logical replication (`Pg::Pg.connect_replication`, `Handler`, the
  `pgoutput` messages), with keepalives and reconnection as tasks of a group.
  A `'t'` (text) tuple value is read as the protocol sends it, a length and
  the bytes, where crystal-pg read a NUL-terminated string.

### What iyi's rules changed

* Errors are values: `PQError`, `ConnectionError`, `StreamError` (a socket
  failure, answered by a statement as iyi-db's `ConnectionLost`) and
  `RuntimeError` are `DBError`s.
* The driver classes are `PGDriver`, `PGConnection`, `PGStatement`,
  `PGResultSet` in `pg/core`.
* `uuid` and `json` read as `UUIDValue` and `JSONValue` (`DBValue` is closed);
  `read(UUID)` and `read(JSON::Any)` answer the std values.
* `timestamptz` reads in UTC; `time_zone` and `clear_time_zone_cache` are
  gone with the zones.
* `connect_listen` and `connect_replication` run their loops as tasks of the
  caller's group, or in place without one.
* `Interval#to_span` answers `ArgumentError` for months it is not told how to
  count; `to_month_span` answers `MonthSpan`. crystal-pg's `to_spans` called
  methods that did not exist and is not ported.

### Not supported yet

* TLS: `sslmode=require`, `verify-ca`, `verify-full`, client certificates and
  SCRAM-SHA-256-PLUS are refused with `ConnectionError`; `prefer` and `allow`
  connect unencrypted. iyi has no TLS library.
* Host names other than `localhost`: `std/socket` resolves no names.

crystal-pg's history before the port is in its
[CHANGELOG](https://github.com/will/crystal-pg/blob/master/CHANGELOG).
