# SQL Blame

**Git blame for your SQL.** An IntelliJ IDEA plugin that shows every SQL query your application executes while you
run, debug or test it: with its real parameters, its execution time and row count, and the line of your code that
executed it. Works for any JDBC stack (JPA / Hibernate, MyBatis, JdbcTemplate, jOOQ, plain JDBC), with no changes to
your project and no server.

- **Real SQL, copy-ready.** Every execution with its parameters inlined and typed quoting, ready to paste into a
  database console.
- **Who ran it.** The call path of every query, with your own code in bold; one click opens the line that executed it.
- **Grouped.** The same query with different parameters is one row with count, total and maximum time, so the
  expensive and the chatty queries stand out.
- **Context.** Spring transaction, Spring Batch job and step, and the thread, when present.
- **Execution plans.** The plan of every query as a tree, a diagram or the database's own text, captured during the
  run, so it is still there when a Testcontainers database is gone; Explain Analyze on demand
  ([docs/explain.md](docs/explain.md)).
- **Any JDBC stack, any run.** Run, Debug and test runs from the IDE, runs delegated to Gradle and Maven goal runs
  (`test`, `verify`, `spring-boot:run`). Applications on Java 8 or later.
- **Already running applications.** **Attach SQL Blame...** loads the agent into a JVM started outside the IDE,
  without a restart: on your computer directly, in a container or on another machine through its debug port
  ([docs/attach.md](docs/attach.md)).
- **Local only.** Nothing leaves your machine; parameter values can be masked.

## Requirements

- IntelliJ IDEA 2026.2 or later, including the free IntelliJ IDEA (no subscription needed).
- An application that talks to its database through JDBC, running on Java 8 or later on your machine, or elsewhere
  with a JDWP debug port you can reach.

## Getting started

1. Install SQL Blame from the JetBrains Marketplace (Settings | Plugins | Marketplace) or from a zip
   (Settings | Plugins | gear icon | Install Plugin from Disk).
2. Run, debug or test your application as usual.
3. Open the **SQL Blame** tool window (bottom). The **Queries** tab lists the queries of the run, grouped, most
   expensive first.
4. Select a query to see the SQL with its parameters, its executions and the call path. **Copy SQL** copies the
   selected execution with its parameters inlined; **Go to Caller** (or a double-click on a frame) opens your code.
5. **Explain Plan** in the SQL toolbar of a query's detail tab shows its execution plan.

The first captured run shows a notification once, with links to turn capture off and to the settings.

In the tool window you can pick one run or all runs, filter by SQL text and minimum duration, pause recording and
clear the list. The **Processes** tab shows which JVMs have the agent attached.

For an application that is already running (started in a terminal, a local Tomcat, a container), use **Attach SQL
Blame...** in the Run menu or the tool window toolbar and pick it from the list, or enter its host and debug port on
the Remote tab. Details: [docs/attach.md](docs/attach.md).

## How it works

The plugin adds a small Java agent to the JVMs started from the IDE. The agent wraps JDBC connections inside your
application, records each statement with its parameters and the calling stack frames, and sends them over a local
socket to the IDE. It also explains the first execution of each query and sends the plan along. **Attach SQL Blame...**
loads the same agent into a running JVM, through the JDK's Attach API or a JDWP debug port. Nothing in your project
changes: no dependency, no configuration, no build script edit.

Runs without a JDBC driver on the class path (`META-INF/services/java.sql.Driver`) do not get the agent at all, so
they pay nothing.

## Turning it off

- **One click:** the **Capture SQL** check box in the SQL Blame tool window toolbar. While it is off, runs start
  without the agent.
- **Settings | Tools | SQL Blame:** the same switch, plus masking, the number of executions kept and the
  application packages.
- **Per run configuration:** the SQL Blame option in the run configuration editor: Default (follow Capture SQL),
  Always or Never.
- **Run with SQL Blame** (Run menu): while capture is off, runs the selected configuration once with the agent.

## Cost of the agent

The agent adds work to your application, and we would rather tell you how much than let you find out. Measured on
2026-10-25 on a laptop CPU (AMD Ryzen 7 260) with the IDE receiving every query; your numbers depend on the machine,
the JVM and how fast your database answers.

- **Per query:** about 40-50 microseconds, mostly for recording which line of your code ran the query. Against a
  database in Docker (1-1.5 ms per query) that is a few percent; against a very fast local database it is more
  noticeable.
- **Per JVM start:** about 0.2-0.25 s for the agent to start, plus about 0.1-0.3 s while a Spring Boot application
  loads its classes: roughly +0.3-0.5 s per test JVM in total. About 2 MiB more retained heap and 8-10 MiB more
  metaspace.
- **Typical effect on a test suite:** with Postgres in Docker / Testcontainers, test time grows by about 7-10%; with a
  very fast local database, by about 10-20%. Runs that execute little SQL mostly pay the startup cost.
- **Runs without a database:** a run without a JDBC driver on its class path does not get the agent. Where the plugin
  cannot see the class path in advance (runs delegated to Gradle), the agent only checks it and stops: about
  10-45 ms per JVM start.
- **When it is off:** the agent is not attached at all. When no IDE is listening to a run, the agent records nothing.
- **Execution plans:** the EXPLAIN of each query's first execution added 0.1-1.8% to test time on top of the agent
  (measured 2026-11-01, same machine); at most 2 000 plans and 5 s of EXPLAIN time per JVM. Switch it off in
  Settings | Tools | SQL Blame | Explain plans.
- **Attached to a running application:** the same cost per query as an agent started with the JVM (within 1.6%, 3.4%
  when the events travel over the debug port; measured 2026-11-03). Loading takes about 0.5 s locally and about 1 s
  remotely; the first 10 s after it run 2-5% slower while the JVM compiles the agent's code.

If the IDE cannot keep up, the agent drops events instead of slowing your application down; the Processes tab then
says how many were dropped.

## Privacy

- Everything stays on your machine. The agent sends events only to the IDE that started the run, over a socket on
  `localhost` protected by a per-session token. An application attached remotely sends them over your network to
  your IDE, protected by the same kind of token but not encrypted, and the IDE sends the agent jar to the
  application through its debug port.
- No telemetry, no analytics, no server of ours. SQL Blame makes no network calls of its own except the EXPLAIN
  statements it sends to your application's database; licensing goes through the IDE's own JetBrains Marketplace
  licensing.
- Error reports: when SQL Blame causes an error in the IDE, the IDE's error dialog can send its stack trace to the
  plugin's page on JetBrains Marketplace. Nothing is sent unless you choose to send it there.
- **Mask parameter values** (Settings | Tools | SQL Blame) replaces every value with `***` inside your application
  for runs started afterwards, and in the tool window at once. Copied SQL then contains `'***'` instead of values.
- Captured queries live in the IDE's memory (by default the newest 50 000 executions per project). Finished runs are
  also saved in a run history in the IDE's system directory, never in your project (at most 100 runs and 1 024 MB
  per project by default), so they survive closing the IDE. Switch it off or clear it in Settings | Tools | SQL
  Blame, Run history ([FAQ](docs/faq.md#where-is-the-run-history-kept)).
- Execution plans are kept with the run, in the history and in exported files. To explain a query again after the
  application ended, SQL Blame keeps the database login the application used (JDBC URL, user, password, driver) in
  the IDE's password storage only, never in files or logs; turn it off or forget the logins in Settings | Tools |
  SQL Blame | Explain plans ([FAQ](docs/faq.md#which-login-does-explain-plan-use-after-the-run)).

## Limitations

Remote JVMs without a debug port, reactive drivers (R2DBC), JDBC drivers in isolated class loaders, Gradle tasks
that run tests only as a dependency and some Maven setups are not captured. After an attach, connections the
application opened earlier and still holds are not captured either. Details and workarounds: [FAQ](docs/faq.md) and
[docs/attach.md](docs/attach.md).

## License and price

SQL Blame is a paid plugin with a 30-day free trial, sold through the JetBrains Marketplace. Beta builds need no
license and have no expiry date.

## Support

- Questions, bugs and ideas: [issues](https://github.com/rwdevtools/sqlblame/issues) (public; do not post data that
  belongs to your employer or clients), or e-mail rw.devtools@gmail.com.
- Errors in the IDE caused by SQL Blame can be sent from the IDE's error dialog; they reach the author through
  JetBrains Marketplace.
- Security problems: see [SECURITY.md](SECURITY.md).

## Legal

[End user license agreement](EULA.md) · [Privacy statement](PRIVACY.md) · [Changes](CHANGELOG.md)

The source code of SQL Blame is not public; this repository holds its documentation and issue tracker.
