# SQL Blame FAQ

## Which runs are captured?

Runs started from IntelliJ IDEA on your machine whose class path contains a JDBC driver:

- Run, Debug and test runs that use the IDE runner (Application, JUnit, TestNG, Spring Boot run configurations),
  including "Run with Coverage".
- Runs delegated to Gradle (the default for Gradle projects): tests from the gutter or the Gradle tool window, and
  Application runs.
- Maven goal runs: `test` (surefire), `verify` (surefire and failsafe), `spring-boot:run`, and tests delegated to
  Maven ("Delegate IDE build/run actions to Maven").

Each JVM gets exactly one agent, and the **Processes** tab of the SQL Blame tool window lists them.

Applications that are already running, started outside the IDE, can be added with **Attach SQL Blame...** (next
question).

## Can I capture an application that is already running?

Yes, with **Attach SQL Blame...** in the Run menu or the SQL Blame tool window, without a restart or a flag. An
application on your computer under your own user is attached directly and shows up as `[attached]`. One in a
container, on another machine or behind `kubectl port-forward` needs a JDWP debug port
(`-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005`) and shows up as `[remote]`. The numbers
match a run started from the IDE. Queries on a connection the application opened before the attach and still holds are
not shown. With a connection pool, every connection borrowed after the attach is captured.

On Java 21 and later the JVM prints a warning when SQL Blame loads. A JVM started with
`-XX:-EnableDynamicAgentLoading` refuses the load, and SQL Blame asks you to start it with
`-XX:+EnableDynamicAgentLoading`. A remote application on a Java 8 JRE cannot be attached. The details, how the
events travel and what each message means are on [attach.md](attach.md).

## Which Java versions and frameworks are supported?

The application can run on Java 8 or later. SQL Blame works at the JDBC level, so JPA / Hibernate, MyBatis, Spring
JdbcTemplate, jOOQ, QueryDSL and plain JDBC are all captured, with HikariCP, Commons DBCP2, Tomcat JDBC and c3p0
pools. It runs next to JaCoCo, IntelliJ's coverage and debugger, the OpenTelemetry Java agent, p6spy,
datasource-proxy and Spring Boot devtools restarts.

Spring transaction names and Spring Batch job and step names are shown when those libraries are present. SQL Blame reads
them without depending on them.

## What is not captured?

- An application on another machine, in Docker, in WSL or on a remote run target, unless it has a JDWP debug port
  you can reach ([attach.md](attach.md)).
- Queries through reactive drivers (R2DBC). SQL Blame captures JDBC only.
- JDBC drivers in isolated class loaders that do not delegate to the application class loader (JBoss Modules, OSGi
  containers). The application works as usual, but its queries do not appear.
- Tests that a Gradle task runs only as a dependency of the task you started, e.g. `build` or `check` running
  `test`. Run the test task, or the test from the gutter, itself.
- Short JVMs started without the IDE, e.g. `./gradlew test` or `mvn test` in a terminal. They usually end before there
  is time to attach, but a long-running application started outside the IDE can be attached with **Attach SQL
  Blame...**.
- Queries on a connection the application opened before an attach and still holds.

## What are the Maven specifics?

- The agent is added to the JVMs Maven forks, not to Maven itself: surefire and failsafe test JVMs and the
  `spring-boot:run` application (`run`, `start` and `test-run` goals).
- Your own `argLine` / `jvmArguments` keep working, including JaCoCo's `prepare-agent` and surefire's `@{argLine}`,
  and the agent is appended last.
- Not captured: tests that run inside the Maven JVM (`forkCount=0`), other plugins that start JVMs (`exec:java`,
  `exec:exec`, Quarkus, Liquibase), Application run configurations delegated to Maven, and the classic Maven launcher
  (registry key `maven.use.scripts` switched off). Maven 4 is not tested yet.

## How do I switch it off?

Click **Capture SQL** in the SQL Blame tool window toolbar: runs started afterwards do not get the agent. The same
switch is in Settings | Tools | SQL Blame. Per run configuration, the SQL Blame option can be set to Always or Never.
While capture is off, **Run with SQL Blame** in the Run menu runs one configuration with the agent.

A run that is already running keeps its agent until it ends.

## How much does it slow my application down?

About 40-50 microseconds per query and about 0.3-0.5 s per JVM start. Test suites against Postgres in Docker take
about 7-10% longer, against a very fast local database about 10-20%. Runs without a JDBC driver do not get the
agent. The full figures, with the machine they were measured on, are in the README section
[Cost of the agent](../README.md#cost-of-the-agent).

## The query list says events were dropped

The agent never blocks your application: when the IDE cannot read events fast enough, it drops them and counts how
many. The Processes tab shows the count, and the query list of that run is incomplete. This happens only with very
high query rates.

## A query shows no application frame in its call path

SQL Blame decides which frames are "your code" from the source roots of your project. If your code is in a library
module or the guess is wrong, set the packages in Settings | Tools | SQL Blame | Application packages
(comma separated, e.g. `com.example.shop`).

## What do the markers in the Flags column mean?

SQL Blame checks the captured queries against ten rules in the IDE, and the agent does no extra work for it. Each rule,
why it matters, how to fix it and exactly when it fires is described in [rules.md](rules.md):

| Id | Rule | Severity |
|---|---|---|
| SB001 | N+1 select: the same SELECT with different values 5 or more times from one line of code in one transaction | Major |
| SB002 | Unbatched writes: the same INSERT / UPDATE / DELETE 10 or more times one by one instead of as a batch | Minor |
| SB003 | Repeated identical query: the same SELECT with the same values twice in one transaction, no write between | Minor |
| SB004 | Slow query: 100 ms or longer | Major |
| SB005 | Large result set: 1 000 or more rows read | Minor |
| SB006 | UPDATE / DELETE without WHERE | Critical |
| SB007 | Leading-wildcard LIKE (`'%...'`) | Minor |
| SB008 | Large IN list: 100 or more elements | Minor |
| SB009 | Lock wait / deadlock | Critical |
| SB010 | SQL error | Major |

Each rule can be turned off and the thresholds changed in Settings | Tools | SQL Blame | Rules. The agent reports the
real database transactions (commit, rollback, auto-commit), so the rules count per transaction, also outside Spring.
A rule that cannot be sure stays silent: with masked parameter values N+1 and repeated queries are not detected.
"Only flagged" in the filter bar shows only flagged queries (the filter icon next to it chooses the rules). Hover over
an icon in the Flags column for that rule, click it for the full rule description. The Issues tab of a query's detail
lists its issues with the executions that caused them. All rule pages are also under "SQL Blame Rules" in the tool
window's gear menu.

## Can I see a query's execution plan?

Yes: **Explain Plan** in the SQL toolbar of a query's detail tab opens the plan as a tree, a diagram or the
database's own text, with hints such as full scans of large tables or sorts that spill to disk. You do not set up a
data source. During the run the agent explains the first execution of every query, so the plan is there after the
application ended. **Re-run** explains it again and **Explain Analyze** runs it (SELECT only, rolled back) to show
real row counts and times. Supported: PostgreSQL, MySQL, MariaDB, Oracle, SQL Server, H2, HSQLDB and SQLite. Db2 is
untested. The details are on [explain.md](explain.md).

## Why is there no plan for my query?

The plan tab says why. The usual reasons:

- During the run the agent explains only the first execution of each query, and never batches (`executeBatch`) or
  stored procedure calls. It stops at 2 000 plans or 5 s of EXPLAIN time per JVM, and does nothing while "Capture the
  plan of each query's first execution during runs" is off (Settings | Tools | SQL Blame | Explain plans). Explain
  Plan then asks for a plan on demand.
- Only `SELECT`, `INSERT`, `UPDATE`, `DELETE` and `MERGE` have a plan. SQL with more than one statement is refused.
- Derby and databases SQL Blame does not know get a message instead. Use **Open in SQL Scratch** and the IDE's own
  Explain Plan with a data source of yours. Db2 plans need its explain tables, which SQL Blame never creates.
- With **Mask parameter values** on, only PostgreSQL 12 and later (a generic plan), Oracle, HSQLDB and Db2 can
  explain the query.
- The database version is older than the oldest one Explain Plan supports (see the next question).

## Does Explain Plan work with PostgreSQL 9 or other old versions?

Yes, from PostgreSQL 9.0 on. The oldest versions are MySQL 5.6.5 (Explain Analyze from 8.0.18), MariaDB 10.1.2,
Oracle 10g Release 2, SQL Server 2005 and SQLite 3.24.0. Older servers get a message instead of a plan. Capturing
queries works with every version. The full table, with the tested versions and the vendor sources, is in
[explain.md](explain.md#supported-versions).

## Does Explain Plan work with Testcontainers?

Yes. The plan is taken while the container is running, so it stays after the tests removed it. PostgreSQL, MySQL,
MariaDB and H2 plans are taken on the test's own connection and see rows it has not committed yet. Oracle and
SQL Server plans are taken on a separate connection and see only committed data. Once the container is gone,
**Re-run** and **Explain Analyze** cannot reach it: the plan tab says so and keeps the captured plan.

## Which login does Explain Plan use after the run?

The one the application used. The agent sends the IDE the JDBC URL, user, password, driver class and driver jars of
each database the application connected to. The IDE keeps them in its password storage, never in run files,
exports or logs, and the Explain Worker (a small Java process started by the plugin with the application's JDBC
driver) connects with them to explain a query after the application ended. A short notification says so
the first time. Its **Don't keep logins** link turns this off and forgets the kept logins, as does **Reuse
application logins** in Settings | Tools | SQL Blame | Explain plans. **Forget Stored Logins** and **Clear History**
forget them and leave the reuse on.

Logins without a password, such as IAM tokens, Kerberos or Azure AD, cannot be replayed and are not kept. Neither
are connections the application gets from a driver's own `DataSource` class without calling the driver. For those
databases, add a connection yourself (next question), or use the plans captured during the run.

## Can I explain a query on a database of my choice?

Yes, with **Connect to Database...**: in the plan tab's message when Re-run cannot reach the database, in the plan
tab's **Database Connection** menu, or in Settings | Tools | SQL Blame | Explain plans. Enter a name, the JDBC URL,
user, password and the driver's jar files. From a plan tab the dialog is already filled in with what the agent saw.
**Test Connection** checks it through the Explain Worker. The connection can point to another database than the
run's, for example a local copy with real data, so the plan header names it ("Explained via <name>") and such plans
are not saved with the run. The login goes to the IDE's password storage like the automatic ones. **Forget Stored
Logins** and **Clear History** leave it there, and you remove it in the settings list.

## Does any data leave my machine?

No. Events go from your application to the IDE over a `localhost` socket protected by a per-session token. SQL Blame
has no telemetry and makes no network calls of its own except the EXPLAIN statements described below. Licensing uses the
IDE's JetBrains Marketplace licensing. When SQL Blame causes an error in the IDE, you can send the error's
stack trace from the IDE's error dialog to the plugin's page on JetBrains Marketplace, and nothing is sent unless you
do.
Captured queries are kept in the IDE's memory (by default the newest 50 000 executions per project) and written to
disk on your machine in two cases: the run history saves every finished run (see "Where is the run history kept?"),
and an export writes the file you choose (see "How do I share a run or keep it for later?"). Both hold the
execution plans of the run too.

For Explain Plan the agent, or after the run the Explain Worker, runs EXPLAIN against your own database, on the
connection your application already uses, on a new one from its JDBC driver, or on a connection you added with
**Connect to Database...**. Nothing else is contacted. The database logins kept for explaining queries
after a run are stored only in the IDE's password storage (see "Which login does Explain Plan use after the run?").

An application attached remotely with **Attach SQL Blame...** sends its events to the IDE over your network,
unencrypted but protected by the run's token, and the IDE sends the agent jar to the application through its debug
port. Nothing else
travels, and nothing goes to a server of ours.

To keep parameter values out of the IDE, turn on **Mask parameter values** in Settings | Tools | SQL Blame: values
are replaced with `***` inside the application for runs started afterwards, and hidden in the tool window at once.

## How do I share a run or keep it for later?

Select the run on the Processes tab and choose **Export Run...** (also next to the run selector on the Queries tab),
or **Export All Runs...** for every listed run. SQL Blame writes a `.sqlblame` file with the captured executions of
the run: SQL, parameter values, timings, row counts, errors, call paths, transaction and job / step context and
execution plans, plus the run name, the JVM command, the Java version and, per database, a hash of its JDBC URL and
user (never the password). Nothing else from the IDE or the project goes into the file.
**Import Run...** on the Processes tab opens such a file, and its runs appear as `[imported] <name> - <date>`,
read-only,
with the rules computed by your current settings. Go to Caller works when the classes exist in the open project.
Import needs a license or an active trial, like capturing. Export always works.

The export dialog has **Mask parameter values**, which leaves every value out of the file, and the plans of
executions that had values (a plan can contain them). SQL text and error
messages are written as captured, and they can contain values too.

The file is compressed and scrambled so it cannot be read in a text editor, and a damaged or edited file is
rejected. This is not encryption against a determined reader: anyone with the SQL Blame plugin can decode it. Treat
the file like the data it contains and send it only to people allowed to see that data.

## Where is the run history kept?

When a run with captured queries ends, SQL Blame saves it in the project's run history, so the run is still there
after you close the IDE. The next time you open the project, the Processes tab lists these runs as `[history] <name> -
<date>`, and a run's queries are read from disk when you first show it. Runs still going when you
close the project are saved too.

The history lives in the IDE's system directory under `sqlblame/history/`, one folder per project. Nothing is
written into your project, so nothing ends up in version control. Each saved run holds what an exported run file
holds (SQL, parameter values, timings, row counts, errors, call paths, transaction and job / step context, execution
plans, run name, JVM command) in the same scrambled format. Parameter values are masked if **Mask parameter values** was
on when the
run was saved.

By default a project keeps at most 100 runs and 1 024 MB, and the oldest runs go first. **Keep in History** in the
Processes context menu protects a run from the run limit, but not from the disk limit.

To change this, open Settings | Tools | SQL Blame, section Run history. There you can switch off **Save finished runs
in the run history**, change both limits, see the space used and delete the history of every project with **Clear
History of All Projects...**. **Clear History...** in the Processes context menu deletes the current project's
history only. Switching the history off stops saving and listing runs, but the files already saved stay until you
clear them.

Opening a history run needs a license or an active trial, like capturing.

## Is SQL highlighted?

When the bundled "Database Tools and SQL" plugin is available and enabled, the SQL in the detail view is highlighted.
Otherwise it is shown as plain text. Everything else works the same.

## Do I need a license?

SQL Blame is a paid plugin with a 30-day free trial through the JetBrains Marketplace. Without a license or an
active trial, new runs start without the agent, but queries already captured stay visible. Beta builds need no license.
