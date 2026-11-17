# TFQL - TraceForge Query Language

Status: baseline design (v0.1)

## 1. Goals

- Readable by anyone who has used a log tool: `level = "ERROR" AND service = "payment-service"`.
- Unambiguous precedence, explicit parentheses, helpful parse errors with a column position.
- Every operator either prunes (pushed into the planner) or filters (residual); nothing is magic.
- No evaluation of user strings as code; a query is only ever an AST walked by the evaluator.

## 2. Grammar (EBNF)

```ebnf
query       = expression EOF ;
expression  = or_expr ;
or_expr     = and_expr { "OR" and_expr } ;
and_expr    = not_expr { ("AND" | implicit_and) not_expr } ;
not_expr    = [ "NOT" ] primary ;
primary     = "(" expression ")" | predicate | bare_term ;
predicate   = field operator value
            | field "IN" "(" value { "," value } ")"
            | field "EXISTS"
            | field "BETWEEN" value "AND" value ;
operator    = "=" | "!=" | ">" | ">=" | "<" | "<=" | "~" | "!~" | ":" ;
field       = identifier { "." identifier } ;
value       = string | number | boolean | duration | null ;
bare_term   = string ;                 (* free text -> message substring match *)
identifier  = letter { letter | digit | "_" | "-" } ;
string      = '"' { char } '"' | "'" { char } "'" ;
number      = [ "-" ] digit { digit } [ "." digit { digit } ] ;
duration    = number ("s" | "m" | "h" | "d") ;
boolean     = "true" | "false" ;
```

`implicit_and` - two adjacent predicates with no operator are an AND (`level="ERROR" service="api"`).
Keywords are case-insensitive (`and`, `And`, `AND`); field names and values are case-sensitive,
except `level`, which is normalized to upper case on both sides.

## 3. Precedence

From tightest to loosest: parentheses → comparison → `NOT` → `AND` → `OR`.

`a = 1 OR b = 2 AND c = 3` parses as `a = 1 OR (b = 2 AND c = 3)`.
`NOT a = 1 AND b = 2` parses as `(NOT a = 1) AND b = 2`.

## 4. Operators

| Operator | Meaning | Applies to | Pushdown |
| --- | --- | --- | --- |
| `=` / `!=` | exact equality | string, number, bool | yes (summaries, bloom) |
| `>` `>=` `<` `<=` | ordered comparison | number, timestamp | yes for `timestamp`; numeric ranges in `.tfm` |
| `~` / `!~` | regex match / no match | string | no (residual only, with a compile cache and size cap) |
| `:` | case-insensitive substring | string | no |
| `IN (...)` | set membership | string, number | yes |
| `BETWEEN a AND b` | inclusive range | number, timestamp | yes for `timestamp` |
| `EXISTS` | field present and non-null | any | yes (attr key set) |

Bare strings match `message` by substring: `"connection refused"` ≡ `message : "connection refused"`.

## 5. Fields

Canonical record fields (`timestamp`, `level`, `service`, `host`, `environment`, `trace_id`,
`request_id`, `logger`, `thread`, `message`) are addressed directly. Anything else resolves against
`attributes` using the dotted key, so `http.status_code >= 500` and `attributes.http.status_code >= 500`
are equivalent. An unknown field is not an error: it is a predicate that matches nothing (and
`EXISTS` makes that explicit). `tenant` is reserved: a user predicate on it is rejected; the
executor injects the authenticated tenant itself.

## 6. Timestamps

Accepted value forms for `timestamp`:

- ISO-8601, optional timezone (naive is treated as UTC): `"2026-11-17T10:00:00"`, `"2026-11-17T10:00:00Z"`
- Epoch seconds / milliseconds / microseconds (disambiguated by magnitude)
- Relative: `now`, `now-15m`, `now-24h`, `now-7d`

`timestamp >= "2026-11-17T10:00:00" AND timestamp < now-5m` is the canonical window form. The planner
intersects every `timestamp` bound it can prove with the request's `from`/`to`; an unsatisfiable
intersection short-circuits to an empty result without touching disk.

## 7. Type coercion

- Comparing a string literal against a numeric attribute coerces the literal; failure makes the
  predicate false rather than erroring (logs are messy by nature).
- `level` comparisons use severity ordering, so `level >= "WARN"` means WARN, ERROR or FATAL.
- Booleans compare only to `true`/`false`.
- `null` is matched only by `= null`, `!= null` and `EXISTS`.

## 8. Errors

Parse and validation failures return HTTP 400 with:

```json
{ "error": "query_parse_error",
  "message": "expected a value after '=' but found ')'",
  "position": 24,
  "query": "level = \"ERROR\" AND service = )",
  "hint": "values must be quoted strings, numbers or booleans" }
```

Classes: unexpected token, unterminated string, unbalanced parenthesis, unknown operator,
invalid regex, invalid timestamp, reserved field, query too long / too many predicates
(DoS guard: `max_query_length`, `max_ast_nodes`, `max_regex_length`).

## 9. Execution model

```
tokens → AST → validate → plan → execute(streaming) → page
```

The planner splits the AST into `(interval, partition_filters, probes, residual)`. It is purely
functional and unit-tested in isolation: given an AST, assert the plan. The executor is a generator
over candidate segments → buckets → blocks → records, so memory is bounded by one decompressed block.

`explain` output:

```json
{ "interval": ["2026-11-17T09:00:00Z", "2026-11-17T10:00:00Z"],
  "partition_filters": { "service": ["payment-service"], "level": ["ERROR", "WARN"] },
  "probes": [], "residual": "http.status_code >= 500",
  "segments_considered": 12, "segments_pruned": 10,
  "blocks_read": 7, "records_scanned": 6800, "records_matched": 41,
  "duration_ms": 38 }
```

`records_scanned / records_matched` is the index-efficiency metric the benchmark suite asserts on.

## 10. Reserved for post-1.0

Aggregations (`| stats count() by service`), field projection, sorting by arbitrary fields, joins,
saved queries with parameters, and alerting predicates. The lexer reserves `|`, `stats`, `by` so the
syntax stays open.
