# SQL Blame privacy statement

SQL Blame is a plugin for IntelliJ IDEA by Robert Weikert (contact: rw.devtools@gmail.com). This statement says what
the plugin does with data. It runs entirely on your computer; there is no SQL Blame server.

## What the plugin handles

While you run, debug or test an application, a small Java agent inside that application records the SQL statements
it executes: the SQL text, the parameter values, the execution time, the number of rows, the calling code location,
the thread and, when present, the Spring transaction and Spring Batch job and step. It also records the execution
plans of the queries and the JDBC URL and user of the connections.

## Where it goes

- From the application to the IDE that started or attached it: over a socket on `localhost` protected by a
  per-session token. An application attached remotely (through its debug port) sends the same data over your
  network to your IDE, protected by the same kind of token but not encrypted.
- In the IDE's memory (by default the newest 50 000 executions per project).
- In the run history in the IDE's system directory on your computer, never in your project (at most 100 runs and
  1 024 MB per project by default). You can switch it off or clear it in Settings | Tools | SQL Blame.
- In a `.sqlblame` file only when you export a run yourself. The file is obfuscated, not encrypted against a
  determined reader; send it only to people allowed to see the data in it.
- The database login an application used (JDBC URL, user, password, driver) is kept only in the IDE's password
  storage, to explain queries again after the application ended. You can switch this off or forget the logins in
  Settings | Tools | SQL Blame | Explain plans.

**Mask parameter values** (Settings | Tools | SQL Blame) replaces every parameter value with `***` inside your
application, so the values never reach the IDE.

## What leaves your computer

- Nothing is sent to the author of SQL Blame. There is no telemetry and no analytics.
- To show execution plans the plugin sends EXPLAIN statements to your own application's database.
- Licensing is handled by JetBrains through the IDE's JetBrains Marketplace licensing, under JetBrains' privacy
  policy.
- When SQL Blame causes an error in the IDE, the IDE's error dialog can send the error's stack trace to the plugin's
  page on JetBrains Marketplace, where the author can read it. Nothing is sent unless you choose to send it.
- Issues you open in this repository and e-mails you send are public or read by the author, respectively; do not
  include data that belongs to your employer or your clients.

## Questions

Write to rw.devtools@gmail.com.
