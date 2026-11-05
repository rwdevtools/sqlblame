# Explain Plan

SQL Blame shows the execution plan of a captured query in the IDE, with no data source to set up. The agent takes
the plan while the application runs, so it is still there after the JVM ended and after a Testcontainers database
was removed. While the database still exists, the plan can be explained again, or measured with EXPLAIN ANALYZE.

## Opening a plan

Open a query's detail tab (double-click a group on the Queries tab) and click **Explain Plan** in the toolbar of the
SQL editor, next to Copy SQL and Open in SQL Scratch. A tab "Plan: <short SQL>" opens in the same row as the detail
tabs. It behaves like a detail tab: close, rename, drag, open in a window of its own and move back. A query has at
most one plan tab; opening it again from another execution makes that execution the one Re-run explains.

The header says which database and version produced the plan, whether it is an **Estimated plan** or an **Actual
plan (EXPLAIN ANALYZE)**, and where it comes from and when. Under it, one line names the query the plan belongs to:
its SQL (cut to fit, the whole SQL with values in the tooltip), the caller (click it to open the line of code) and
the execution number within the query's group. The arrow at the start of the line shows the SQL with values, wrapped,
up to eight lines with a scroll bar. Long values such as a 500-character string literal wrap here and in the details
panel instead of widening the tab.
The summary strip below shows total cost, estimated rows and planning time; actual plans add actual rows, execution
time and buffers. On the right of the strip, one badge per kind of problem found in the plan (see below); a click
selects the next step that has it.

## Where a plan comes from

| Source in the header | When |
|---|---|
| Captured during the run | The agent explained the first execution of the query's group right after it ran |
| Captured in an earlier run on the same database | The same query was explained in a run of the last 24 hours against the same database, so the agent skipped it (remembered until the IDE closes) |
| Explained by the running application | You clicked Re-run or Explain Analyze while the application was still running |
| Explained by the Explain Worker | You clicked Re-run or Explain Analyze after the application ended |

### During the run

The agent explains the **first execution of every group** (the same SQL with different values), once per JVM.

On PostgreSQL (also CockroachDB and YugabyteDB), MySQL, MariaDB and H2 it runs a plain `EXPLAIN` on the
application's own connection, in its transaction, right after the query. The plan sees the test's uncommitted rows
and temporary tables, and a plain EXPLAIN changes no data. If the transaction has already failed (PostgreSQL after an
error), no plan is taken.

Oracle, SQL Server, HSQLDB, SQLite and Db2 are different: EXPLAIN there changes session settings or writes
`PLAN_TABLE`, so the agent explains on a separate connection on its own thread, never on the application's thread.
That connection does not see data the application has not committed yet. A test that inserts rows and queries them
in one transaction gets a plan for the committed data only.

Only single executions are explained, not batches (`executeBatch`) or stored procedure calls. Statements SQL Blame
runs itself never appear in the query list.

One side effect on MySQL and MariaDB: the EXPLAIN runs between the query and the application's next statement, so
an application that reads `ROW_COUNT()` or `SHOW WARNINGS` right after its query sees the EXPLAIN's values instead,
once per group. Statements with `SQL_CALC_FOUND_ROWS` are explained on a separate connection, so `FOUND_ROWS()`
keeps working. If your code relies on `ROW_COUNT()`, switch plan capture off (see "Settings").

Capture is bounded per JVM: at most 2 000 plans and 5 s of EXPLAIN time, whichever comes first. If the plans taken on
the application's thread average more than 20 ms, the agent moves the rest of the run's plans to its own thread.
Measured on the author's test projects, plan capture adds 0.1% to 1.8% to test time.

### Re-run, during and after the run

**Re-run** asks for a new estimated plan, with a 15 s limit; the query is not run. While the application runs, its
agent explains the execution on a new read-only connection straight from the JDBC driver, outside the application's
pool, and rolls back at the end. Once the application has ended, the **Explain Worker** does the same. It is a small
Java process the plugin starts with the project's JDK (or the IDE's runtime without a project SDK) and the
application's JDBC driver jars on its class path, so the IDE never loads your driver. It ends after 5 minutes
without a request.

The worker connects with the login the application used (see "Logins" below). When the database is gone, for example
a Testcontainers container removed at the end of the tests, the worker cannot connect: the tab says so and keeps
showing the plan captured during the run. A database connection of your own can take over then (see "Connect to
Database..." below).

Every request answers in the bar right under the plan tab's toolbar: a progress line while it runs, then one sentence
when no new plan came back, with a **Settings** link where a setting is the cause and **Connect to Database...**
where a connection of your own can help. When a request is refused or cannot
be tried, a balloon at the clicked button says so too. Typical cases: no login is stored for the database, or the run
was recorded by a SQL Blame version without plans or with plan capture off (older history entries and imports). The
database's or the worker's own message sits behind **Details** when it is short, and always in `idea.log`
(Help | Show Log in Explorer / Finder).

## Explain Analyze

**Explain Analyze** runs the query to measure it and shows the actual plan: real row counts, times, loops and, on
PostgreSQL, buffers. Because it runs the query, SQL Blame only does it when you click it, and only for a `SELECT`
without `FOR UPDATE`, `FOR SHARE` or lock hints. It runs in a read-only transaction that is rolled back, with a time
limit of 30 s (Settings). The first time, a dialog asks for confirmation. **Cancel** stops a running request; closing
the tab cancels it too. It still costs what the query costs: it can take as long as it does in the application, and
the database does the same work.

## Views

The **Tree** view (the default) has one row per step with Operation, Object, Est. rows, Cost and % of cost (the
step's own cost over the plan's); actual plans add Actual rows, Actual time, Loops and Buffers. The costliest step is
bold, and actual rows 10 or more times off the estimate are marked. Expand All, Collapse All and Go to Costliest Step
are in the toolbar.

The **Diagram** draws each step as a card with an edge to the step that uses its rows, left to right or top down.
Thicker edges carry more rows, and a card's background gets warmer the larger its share of the cost. Ctrl+mouse
wheel zooms and dragging pans. A click on a card's handle hides its inputs; a minimap appears when the plan does not
fit. Export PNG... saves the picture.

**Text** shows the database's own plan text in a read-only editor, with the raw JSON, XML or output in a second tab.

The panel on the right shows the selected step's details: operation, object, costs, predicates, output columns,
every property the database reported, and the explanation of each problem found in it. The selection is shared by
the Tree and the Diagram. **Copy Plan** copies the plan as text.

**Go to Entity** (or a double-click on a step) opens the class that maps the step's table: a JPA entity found by
`@Table(name)`, entity name or class name, or a jOOQ generated table class. Names match without case and
underscores, the schema is ignored.

## Problems found in plans

These hints come from the plan, not from the captured executions, so they are separate from the query rules
SB001-SB010 ([rules.md](rules.md)). Some need an actual plan.

| Problem | When it is shown | Needs an actual plan |
|---|---|---|
| Full scan | A table scan (Seq Scan, `TABLE ACCESS FULL`, Clustered Index Scan, `type: ALL`) reads 10 000 or more rows and filters them | No |
| Wrong estimate | Actual rows are 10 or more times above or below the estimate | Yes |
| Spill to disk | A sort, hash or temporary table wrote to disk (`external merge Disk`, hash batches > 1, spill warnings) | Yes, except MySQL's temporary table with filesort |
| Many loops | The inner side of a nested loop ran 1 000 or more times | Yes |
| Filter discards rows | A filter throws away 90% or more of at least 1 000 rows read, or a whole index is read with no index condition | Yes, except the whole index read |
| Lookup per row | SQL Server Key / RID Lookup or Oracle `TABLE ACCESS BY INDEX ROWID` ran 1 000 or more times | Yes |

PostgreSQL and Oracle estimate rows after the filter, so their estimated plans show a full scan only when the filter
keeps many rows. SQLite reports no row counts, so its plans get no row-based hints.

## Databases

| Database | Estimated plan | Explain Analyze | Tested |
|---|---|---|---|
| PostgreSQL | `EXPLAIN (FORMAT JSON)` | `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)` | PostgreSQL 17 |
| CockroachDB, YugabyteDB | as PostgreSQL (CockroachDB as text, read on a best-effort basis) | as PostgreSQL | No |
| MySQL | `EXPLAIN FORMAT=JSON` | `EXPLAIN ANALYZE` (8.0.18 and later) | MySQL 8.4 |
| MariaDB | `EXPLAIN FORMAT=JSON` | `ANALYZE FORMAT=JSON` | MariaDB 11.4 |
| Oracle | `EXPLAIN PLAN` + `DBMS_XPLAN.DISPLAY` | `DBMS_XPLAN.DISPLAY_CURSOR` (needs `SELECT` on the `V$` views) | Oracle Free 23 |
| SQL Server | `SET SHOWPLAN_XML ON` | `SET STATISTICS XML ON` | SQL Server 2022 |
| H2 | `EXPLAIN` | `EXPLAIN ANALYZE` | H2 2.3 |
| HSQLDB | `EXPLAIN PLAN FOR` | - | HSQLDB 2.7 |
| SQLite | `EXPLAIN QUERY PLAN` | - | SQLite 3.54 |
| Db2 | `EXPLAIN PLAN FOR` | - | No (written from IBM's documentation) |

Db2 plans need the explain tables; SQL Blame never creates them (`SYSPROC.SYSINSTALLOBJECTS` does). Derby and other
databases get a message: open the query with **Open in SQL Scratch** and use the IDE's own Explain Plan with a data
source of yours.

Only `SELECT`, `INSERT`, `UPDATE`, `DELETE` and `MERGE` have a plan. SQL with more than one statement is refused.
On SQL Server the estimated plan is built with the parameter values written into the statement, because its driver
returns no plan for a prepared statement with parameters.

### Supported versions

Capturing queries does not depend on the database version: the agent works on the JDBC interfaces, so any database
whose JDBC driver runs on Java 8 or later is captured. Explain Plan sends the statements above, and each one exists
only from a certain version on. The table lists the oldest version that has them, according to the vendor's
documentation, and the versions SQL Blame is tested against.

| Database | Estimated plan from | Explain Analyze from | Tested | Source |
|---|---|---|---|---|
| PostgreSQL | 9.0 | 9.0 | 17 | [EXPLAIN in 9.0](https://www.postgresql.org/docs/9.0/sql-explain.html) (options in parentheses, `FORMAT JSON`, `BUFFERS`) |
| CockroachDB | 21.2 | 21.2 | No | [EXPLAIN in 21.2](https://www.cockroachlabs.com/docs/v21.2/explain) (the text format SQL Blame reads) |
| YugabyteDB | every YSQL version | every YSQL version | No | YSQL is built on PostgreSQL 11 or later |
| MySQL | 5.6.5 | 8.0.18 | 8.4 | [EXPLAIN in 5.6](https://dev.mysql.com/doc/refman/5.6/en/explain.html) (`FORMAT=JSON`), [EXPLAIN ANALYZE in 8.0](https://dev.mysql.com/doc/refman/8.0/en/explain.html#explain-analyze) |
| MariaDB | 10.1.2 | 10.1.2 | 11.4 | [EXPLAIN FORMAT=JSON](https://mariadb.com/kb/en/explain-format-json/), [ANALYZE FORMAT=JSON](https://mariadb.com/kb/en/analyze-format-json/) |
| Oracle | 10g Release 2 (10.2) | 10g Release 2 (10.2) | Free 23 | [DBMS_XPLAN in 10.2](https://docs.oracle.com/cd/B19306_01/appdev.102/b14258/d_xplan.htm) (`DISPLAY`, `DISPLAY_CURSOR`, `ALLSTATS LAST`); `PLAN_TABLE.OTHER_XML` exists from 10.2 |
| SQL Server | 2005 | 2005 | 2022 | [SET SHOWPLAN_XML](https://learn.microsoft.com/en-us/sql/t-sql/statements/set-showplan-xml-transact-sql), [SET STATISTICS XML](https://learn.microsoft.com/en-us/sql/t-sql/statements/set-statistics-xml-transact-sql) (both added in 2005) |
| H2 | 2.0 | 2.0 | 2.3.232 | Plan text of H2 2.x; 1.4 is not tested |
| HSQLDB | 2.0 | - | 2.7.4 | `EXPLAIN PLAN FOR` in the [HSQLDB 2 guide](https://hsqldb.org/doc/2.0/guide/) |
| SQLite | 3.24.0 | - | 3.54.1 | [EXPLAIN QUERY PLAN](https://www.sqlite.org/eqp.html) rows with `id` and `parent` since [3.24.0](https://www.sqlite.org/releaselog/3_24_0.html); the version is the one inside the `sqlite-jdbc` driver |
| Db2 for Linux, UNIX and Windows | 11.5 | - | No | [EXPLAIN](https://www.ibm.com/docs/en/db2/11.5?topic=statements-explain) and the explain tables of 11.5 |

When the driver reports a PostgreSQL, MySQL, MariaDB, Oracle, SQL Server or SQLite version older than the table's,
SQL Blame does not send the EXPLAIN and the plan tab says the version is too old. For the other databases it tries,
and an error shows up as a failed explain. CockroachDB and YugabyteDB report the PostgreSQL version they are
compatible with, not their own, so their version is not checked.

Some features need newer versions:

- Plans with masked parameter values (a generic plan): PostgreSQL 12 and later, which has `plan_cache_mode`
  ([12](https://www.postgresql.org/docs/12/runtime-config-query.html#GUC-PLAN-CACHE-MODE)). Not on YugabyteDB.
- The lock wait limit on PostgreSQL (`lock_timeout`) exists from 9.3; on 9.0-9.2 only the statement time limit
  applies.

## Logins

To explain a query after the application ended, the worker needs the database login. The agent sends the IDE the
JDBC URL, user, password, driver class and driver jar paths the application used to connect. The IDE keeps them in
its own password storage (Settings | Appearance & Behavior | System Settings | Passwords), one entry per URL and
user. They are never written to run files, the run history, exports or logs. The first time a login is kept, a
short notification ("SQL Blame can explain queries after the run") says so, with a **Settings** link and
**Don't keep logins**, which does the same as turning off the setting below.

**Settings | Tools | SQL Blame | Explain plans | Reuse application logins** turns this off and forgets every kept
login. **Forget Stored Logins** forgets them and leaves the setting on. **Clear History** forgets them too.

Logins without a password (IAM tokens, Kerberos, Azure AD) cannot be replayed and are not kept. Neither are the
details of connections the application gets from a driver's own `DataSource` class without a `Driver#connect` call.
For those, the plans captured during the run are all there is, unless you add a connection yourself (next section).

## Connect to Database...

When no kept login can reach the database, add a connection yourself: **Connect to Database...** in the plan tab's
message, in the plan tab's **Database Connection** menu (the database icon in its toolbar), or under **Database
connections for Explain Plan** in Settings | Tools | SQL Blame | Explain plans, where you can also edit, test and
remove connections. Typical cases: a run from the history recorded without plans, a login that was never kept, or a
Testcontainers database that is gone while a local copy with the same schema is still there.

The dialog asks for a name, the JDBC URL, user, password and the JDBC driver's jar files. Opened from a plan tab, it
is filled in with what the agent saw for that query's database: the URL, user and driver jars, and the password when
a login is kept. **Test Connection** connects through the Explain Worker and names the database it reached. The
connection is kept like the automatic logins: URL, user, password and driver jars in the IDE's password storage,
and only the name in the SQL Blame settings file. **Forget Stored Logins** and **Clear History** leave these
connections alone; remove them in the settings list.

A connection you save in a plan tab explains the query right away. It is also chosen for that query's database:
when Re-run finds no kept login for it, or the kept login can no longer connect, the worker uses that connection (the
most recently used one, if several are chosen for the same database). The plan tab's **Database Connection** menu
picks the connection for that tab directly, or goes back to the run's own database.

Because the connection may point to another database than the run's, the plan header names it ("Explained via
<name>"), and such plans are shown in the plan tab only: they are not saved with the run. The worker treats these
connections like the run's own logins: only EXPLAIN, read-only where the database allows it, rolled back, with the
same time limits, and Explain Analyze for SELECT only.

## Plans in run files and the history

Plans are saved with the run: in memory, in the run history and in exported `.sqlblame` files. With them goes a hash
of the JDBC URL and user of their database, never the URL, user or password themselves. An exported run opened
elsewhere shows its plans; Re-run there works only if that IDE has a login for the same database or a connection
chosen for it. With **Mask parameter values** on, saved runs and
exports leave out the plans of executions that had values, because a plan can contain them.

## Masked parameter values

With **Mask parameter values** on, the agent has no values to explain with. PostgreSQL 12 and later get a generic
plan instead (one the database makes without knowing the values); Oracle, HSQLDB and Db2 do not use values for an
estimated plan anyway. Other databases say that the values are masked.

## Settings

Settings | Tools | SQL Blame, section **Explain plans**:

| Setting | Default |
|---|---|
| Capture the plan of each query's first execution during runs | on (applies to runs started afterwards) |
| Plans per JVM | 2 000 |
| EXPLAIN time per JVM (s) | 5 |
| Time limit of Explain Analyze (s) | 30 |
| Ask before Explain Analyze runs a query | on; the first confirmation turns it off |
| Reuse application logins to explain queries after a run | on |
| Forget Stored Logins | button |
| Database connections for Explain Plan | none; add, edit, test and remove (saved at once) |

With capture off, nothing is explained during the run, and Re-run while the application runs is answered by the
worker, which needs a kept login.

Re-run and Explain Analyze need a license or an active trial, like capturing queries. Plans already kept stay
visible without one.
