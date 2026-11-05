<!-- Keep a Changelog guide -> https://keepachangelog.com -->

# SQL Blame Changelog

## [Unreleased]

First beta.

### Added
- SQL Blame tool window: the Queries tab groups the queries of a run by normalized SQL with count, total and maximum
  time. The detail shows the SQL of an execution with its parameters, the executions of the group, the call path
  (application frames in bold) and the Spring transaction, Spring Batch job and step and thread when present.
- Copy SQL: the selected execution with its parameters inlined and typed quoting (toolbar, context menu, copy
  shortcut).
- Go to Caller and navigation from any call path frame to the line of code.
- Query details open in closable, renamable tabs (close button, context menu, middle click, Close Tab), and a detail can
  move into a window of its own and back into the tool window.
- Query rules SB001-SB010 (`docs/rules.md`): N+1 select, unbatched writes, repeated identical query, slow query,
  large result set, UPDATE / DELETE without WHERE, leading-wildcard LIKE, large IN list, lock wait / deadlock and SQL
  error. Flags column, Only flagged filter, offending executions in the detail, and each rule on / off and its threshold
  in Settings.
- Exact transaction boundaries: the agent numbers the transactions of each connection (commit, rollback, auto-commit),
  so markers count per real database transaction, also for JdbcTemplate, MyBatis and plain JDBC outside Spring.
- Run selector (all runs or one run), SQL text and minimum-duration filter, Pause and Clear. The newest 50 000
  executions per project are kept.
- Queries toolbar in two rows: runs and their actions above, the filters of the collected data below.
- Processes tab: the JVMs with the agent attached, their Java version and dropped events.
- Attach to runs from the IDE runner, runs delegated to Gradle and Maven goal runs (surefire, failsafe,
  `spring-boot:run`). Runs without a JDBC driver on the class path never get the agent.
- Capture SQL switch in the tool window, per run configuration mode (Default / Always / Never) and Run with SQL Blame.
- Settings | Tools | SQL Blame: capture, parameter masking, executions kept, markers, application packages.
- Applications on Java 8 or later with JPA / Hibernate, MyBatis, JdbcTemplate, jOOQ, QueryDSL and plain JDBC.
- Own tool window icon and plugin logo.
- Run history that survives closing the IDE (at most 100 runs and 1 024 MB per project), Export Run / Import Run of
  `.sqlblame` files.
- Explain Plan (`docs/explain.md`): the agent captures the plan of each query's first execution during the run, so
  it is still shown after the JVM ended and a Testcontainers database is gone. A "Plan" tab next to the detail tabs
  shows it as Tree, Diagram or Text, with node details and a summary of cost, rows and times.
- Re-run and Explain Analyze (SELECT only, read-only, rolled back, confirmed once, 30 s limit, Cancel), answered by
  the running application or, after it ended, by an Explain Worker process with the application's JDBC driver.
- PostgreSQL, MySQL, MariaDB, Oracle, SQL Server, H2, HSQLDB and SQLite tested, CockroachDB, YugabyteDB and Db2
  untested.
- Supported database versions listed in `docs/explain.md` (PostgreSQL 9.0, MySQL 5.6.5, MariaDB 10.1.2, Oracle 10.2,
  SQL Server 2005 and SQLite 3.24.0 at the oldest). Older servers get a message instead of an EXPLAIN they reject.
- Problems found in plans: full scan of a large table, wrong estimate, spill to disk, many loops, filter discarding
  most rows, lookup per row.
- Go to Entity from a table step to its JPA entity or jOOQ table class.
- Plan header with the query it belongs to in one line (SQL, caller link, execution number), expandable to the SQL with
  values. Long literals wrap instead of widening the tab.
- Database logins for explaining after the run kept in the IDE's password storage only. Settings section "Explain
  plans" with capture budgets, the Explain Analyze limit, login reuse and Forget Stored Logins.
- A short one-time notification when SQL Blame first keeps a database login, with Settings and Don't keep logins.
- Connect to Database...: database connections of your own for Explain Plan (pre-filled from what the agent saw,
  Test Connection, kept in the password storage), used when the run's login is missing or no longer connects, or picked
  in the plan tab. Plans made through them name the connection and are not saved with the run.
- Plans saved in the run history and in exported `.sqlblame` files (file version 2, version 1 files still open).
- Attach SQL Blame... (Run menu, Queries and Processes toolbars, `docs/attach.md`): captures the SQL of an
  application already running outside the IDE, without a restart. Local JVMs of your user are attached through the
  Attach API (`[attached]`), JVMs in a container or on another machine through a JDWP debug port (`[remote]`).
- Remote events reach the IDE over the first transport that works: a connection back to the IDE, a port the agent
  listens on, JMX, or the debug port itself. The transport is shown in the Processes tooltip.
- Detach and Attach Again for attached runs. Plans, rules, export and the run history work on them as on any run.
- Plain messages for refused attaches: JEP 451 (`-XX:-EnableDynamicAgentLoading`, with Copy Flag), another OS user,
  an unreachable port, a debugger already connected, an idle application, a Java 8 JRE.
- Saved remote targets and Test Connection, one confirmation before the first remote attach, and Settings | Tools | SQL
  Blame | Attach.
- Run files version 3 with where a run was attached (versions 1 and 2 still open).
