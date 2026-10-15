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
