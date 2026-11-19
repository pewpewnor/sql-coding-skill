---
name: sql-coding
description: Write, review, and refactor SQL queries and statements, including standalone SQL and SQL embedded in Zig strings. Use only when the task deals with SQL itself, not for database-related host code without SQL changes or review.
---

# SQL coding

## Scope

Apply this skill only when writing, revising, inspecting, or reviewing SQL queries
or statements, including DDL, migrations, and SQL embedded in Zig or other
programming-language strings. Database-related Zig code alone does not trigger
this skill when no SQL statement is being worked on.

This skill covers **SQL code only**: formatting, naming, readability, correctness, and safety. Do not prescribe project structure, file organization, application architecture, or formatting for non-SQL code.

## Guiding principles

1. **Preserve semantics.** Never change what a query does just to make it look consistent.
2. **Respect the dialect.** Use syntax supported by the target database and version; do not assume SQLite-specific syntax is portable.
3. **Be compact without being cramped.** Keep short clauses and related items together; break lines at meaningful boundaries.
4. **Keep formatting consistent.** The SQL inside source-code strings must look like standalone SQL, ignoring only the host language's necessary outer indentation.
5. **Favor safe, explicit SQL.** Validate references, use parameters for external values, and scope data changes deliberately.

## General formatting

- Write SQL **keywords in uppercase**: `CREATE TABLE`, `PRIMARY KEY`, `SELECT`, `FROM`, `JOIN`, `WHERE`, `INSERT INTO`, `VALUES`, `UPDATE`, `SET`, `DELETE FROM`, `ORDER BY`.
- Prefer lowercase `snake_case` for identifiers. Preserve existing names when changing them would be incompatible.
- SQL function names may be lowercase (`length`, `trim`, `count`, `max`); do not change their case merely for stylistic reasons.
- Use **four spaces per indentation level**. Never use tabs for SQL indentation.
- Begin major clauses on new lines, aligned at the same statement level.
- **Keep short, simple content on its clause line**: `SELECT name`, `WHERE id = 1`, `ORDER BY score DESC`.
- **Group short related items on a line** rather than placing every column or value on a separate line: `id, name`.
- If a line becomes hard to scan, break it at a logical boundary and indent continuations by four spaces.
- Use one space after commas and around binary operators. Do not align assignments using extra padding.
- Use single quotes for string literals: `'admin'`. Quote identifiers only as required by the dialect.
- Finish complete standalone statements with `;`. In embedded SQL, include `;` if accepted by the driver/API; otherwise omit it consistently.
- Prefer the standard `=` operator for portable SQL. If editing existing dialect-specific SQL that uses `==` (as SQLite allows), **do not change it solely for style**.
- Avoid unnecessary blank lines within a statement; separate completed statements with one blank line.

## CREATE TABLE and constraints

Put each column definition on its own line, followed by a comma except for the last definition. Keep the name and type together; place substantial column constraints on **indented continuation lines**. This is the preferred style, even when the constraint could fit on the column line.

```sql
CREATE TABLE players (
    id INTEGER
        PRIMARY KEY,
    name TEXT NOT NULL
        CHECK (length(trim(name)) > 0)
) STRICT;
```

- `NOT NULL` may stay on the type line.
- Put `PRIMARY KEY`, `REFERENCES`, `CHECK`, `UNIQUE`, and other substantial constraints on the next line at one extra indentation level when they belong to a column.
- Keep a short constraint and its condition or action together: `CHECK (score >= 0)` and `REFERENCES players(id) ON DELETE CASCADE`.
- Use `CHECK (expression)` consistently with a space before `(`.
- Put table-level constraints on their own lines alongside column definitions; indent their continuation lines as needed.
- Use dialect-specific types and modifiers (such as SQLite `STRICT`) only when appropriate.
- Verify every referenced table, column, and constraint. Formatting must not conceal a broken foreign key.

## INSERT

Put the target columns inside parentheses on separate lines from `INSERT INTO` and `VALUES`. **Group short column names and their corresponding values together**; split into multiple lines only when needed for readability.

```sql
INSERT INTO players (
    id, name
)
VALUES (
    1, 'admin'
);
```

Always list insertion columns explicitly unless there is a strong reason not to. Keep the value order matched to the column order.

## SELECT, JOIN, and WHERE

Keep a short `SELECT` list on the same line as `SELECT`. Put `FROM`, each `JOIN`, and `WHERE` on separate lines. **Keep a simple `JOIN ... ON ...` on one line**, rather than automatically putting `ON` on a new line.

```sql
SELECT players.name
FROM players
JOIN score_history ON score_history.player_id = players.id
WHERE score_history.level = 1;
```

- Use table names directly for short queries; aliases are optional when they genuinely improve readability.
- Qualify ambiguous columns, and keep qualification consistent in joins when it aids clarity.
- Prefer explicit `JOIN` syntax over implicit comma joins.
- Keep single predicates on the `WHERE` line; place additional conditions on indented continuation lines starting with `AND` or `OR`.
- Parenthesize mixed `AND` and `OR` logic to make precedence clear.
- Take care not to change outer joins accidentally by moving join conditions from `ON` into `WHERE`.
- Avoid `SELECT *` in queries that need a stable, well-defined result shape.
- Add a deterministic tie-breaker when ordered results must remain stable.

## UPDATE

Keep `UPDATE` on one line with the table name. Put `SET` on its own line; indent one assignment per line beneath it. Keep a short `WHERE` condition on the same line.

```sql
UPDATE score_history
SET
    score = 2000,
    level = 2
WHERE id = 1;
```

An unfiltered `UPDATE` should be intentional and clearly justified, not an accidental omission of `WHERE`.

## DELETE

Keep `DELETE FROM` and the table on one line; use a separate `WHERE` line for targeted deletions.

```sql
DELETE FROM players
WHERE id = 1;
```

An unfiltered `DELETE` should be an explicit intentional operation.

## Other SQL constructs

Maintain the same compact clause-first approach across `GROUP BY`, `HAVING`, `ORDER BY`, `LIMIT`, `RETURNING`, CTEs, subqueries, `CASE`, and dialect-specific upserts.

- Indent subqueries one level within their containing parentheses.
- Align closing parentheses with the construct that opened them.
- Break complex `CASE` expressions across `WHEN` / `THEN` / `ELSE` / `END` lines.
- Place set operators such as `UNION ALL` on their own lines between query blocks.
- Do not add dialect-specific syntax unless the target database supports it.

## SQL embedded in programming languages

**Embedded SQL must use the same structure as standalone SQL.** Only the host language's string delimiter, parameter-binding syntax, and outer indentation may differ. Do not turn SQL into one long line or alter its internal layout just because it is inside code.

### Zig

Use Zig multiline string literals for SQL with multiple clauses. Each SQL line
starts with `\\`; SQL indentation follows those markers. Preserve Zig's outer
indentation and format the SQL content as if it were a standalone statement.
Do not collapse a query into a single quoted string or concatenate fragments
solely to avoid multiline literals. A short single-clause statement may remain
a normal Zig string when it already follows the SQL layout rules.

For example, with a driver that accepts positional placeholders:

```zig
const query =
    \\SELECT id, name
    \\FROM accounts
    \\WHERE id = ?
;
```

Bind the value through the driver's parameter API. Do not use `std.fmt` or concatenation to insert external values into SQL. SQL loaded with `@embedFile` follows the same rules in its source `.sql` file. Match placeholder order and SQL terminator handling to the driver.

- Ignore only the common outer indentation when comparing embedded SQL to standalone SQL.
- Use the database driver's supported placeholders (`?`, `:name`, `$1`, etc.) and matching bindings.
- Never interpolate untrusted values directly into SQL text.
- When identifiers must be dynamic, validate against an allowlist; parameter placeholders are for values, not SQL identifiers.
- If a query is composed from SQL fragments, ensure the assembled SQL still follows these layout rules.

## Correctness and safety checks

- Confirm table names, column references, foreign keys, and aliases actually exist.
- Validate the SQL against its intended dialect and version; for example, `STRICT` and `==` are SQLite-specific syntax choices, not universal SQL conventions.
- Use `IS NULL` / `IS NOT NULL` for `NULL` tests.
- Ensure joins and grouping produce the intended number of rows.
- Parameterize external values; do not construct SQL with untrusted string interpolation.
- Verify that `UPDATE` and `DELETE` target the intended rows.
- Avoid changing result order, cardinality, constraints, or data types merely to satisfy formatting rules.
- Keep comments concise and focused on **why** a SQL decision was made.

## Review checklist

- [ ] SQL keywords, indentation, spacing, and line breaks follow this compact style.
- [ ] DDL constraints use indented continuation lines.
- [ ] `INSERT` column lists and values use compact, corresponding groups.
- [ ] Simple `SELECT`, `JOIN ... ON`, and `WHERE` clauses stay on one line where readable.
- [ ] `UPDATE SET` assignments are vertically listed; `DELETE` filters are clearly scoped.
- [ ] Embedded SQL uses the same layout as standalone SQL.
- [ ] The statement is valid in the chosen dialect and preserves intended semantics.
- [ ] Any external values are safely parameterized.
