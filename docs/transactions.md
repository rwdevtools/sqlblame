# Transaction history

SQL Blame shows, next to every query, the transaction it ran in and what happened to that transaction: who opened it
and with which settings, which inner methods joined it or suspended it, when Hibernate sent the statement and why at
that moment, and whether the transaction committed or rolled back, and because of what. The **Transactions** tab
puts all transactions of a run on a timeline.

It works for Spring transactions (JPA, `DataSourceTransactionManager`, JTA, Spring Batch's
`ResourcelessTransactionManager`), Hibernate 5.6 to 7, Spring Batch 4 to 6, standalone JTA with Narayana or
Atomikos, and CDI `@Transactional` (`jakarta.transaction` or `javax.transaction`, as in Quarkus, Helidon or WildFly).
Without Spring (plain JDBC, MyBatis or jOOQ alone) every query still gets "committed" or "rolled back" from
the connection's own `commit()` and `rollback()`. Nothing changes in your project: the agent reads these frameworks
read-only, and nothing about the history leaves your machine.

The sections below follow the transaction questions developers ask most often, each with what SQL Blame shows for it.

## Where to look

Select a query, pick an execution in its detail and click **Transaction** next to Issues. It opens a tab of its own,
"Transaction: <name>", with the run's timeline zoomed on the transaction that execution ran in (its bar selected, the
other threads of the same time around it) and the details below: the outcome with time, duration, number of
statements and the connection, then one sentence on the cause. Below come the scope tree (propagation, isolation,
read-only, timeout, rollback rules and result per scope) and the transaction's event list with this query
highlighted. Clicking Transaction again for the same transaction, from any of its statements, brings that tab back
instead of opening a second one; the tab can be renamed, dragged and closed like a query tab. For a run recorded
without the transaction history the entry is disabled. The event list's Flags column shows the transaction rules
(SB011-SB022) that concern a line, one icon per rule with the finding as tooltip and the rule page on a click; the
same icons follow the header line, once per rule. Double-click a scope to open its code, or a statement to open it in
its query tab. A query that ran in auto-commit says "Auto-commit: this statement was its own transaction."

In the Queries tab, SB011-SB022 appear in the Flags column like the other rules, groups with rolled-back executions
get a **Rolled back** icon, and **Only rolled back** keeps only those. The Executions table shows the outcome as icon
and word in its Transaction column and greys out rolled-back executions.

The **Transactions** tab, pinned after Queries and Processes, draws the run as a timeline: one lane per thread, one
bar per transaction with its outcome, nested scopes below their parent, query ticks on the bar. Click a bar to see
that scope first: one line with what it did itself (for example "joined the transaction of @Transactional
OrderService.place and marked it rollback-only", or "its own transaction (REQUIRES_NEW)"), the scope selected in
the tree and its own lines in bold in the event list, below that the story of the transaction it ran in, as in a
transaction's own tab. Double-click a bar, or use **Show Queries of This Transaction** (the magnifier), to filter
Queries to that transaction. Ctrl+mouse wheel zooms, dragging pans, and the toolbar has **Fit**, **Actual Size**,
**Export PNG...** and the filters **Only
Rolled Back**, **Only Long**, **Only Flagged** and **Hide Spring Batch Metadata**. While a filter hides bars, "Filtered:
N of M" next to the filters gives the count; a run with more than 25,000 transactions says there how many older ones
were left out.

With two monitors, **Open Timeline in Separate Window** in the toolbar of the Transactions tab, or of a transaction's
own tab, moves the timeline with its toolbar into a window of its own, which keeps its size and position for the next
time. The details stay in the SQL Blame window and follow every click in the timeline window; if the tool window was
hidden, a click shows it without taking the keyboard focus. **Move Back to SQL Blame** in the window, the Move Back
link above the details, or closing the window puts the timeline back.

Findings without a statement of their own, a `@Transactional` method that bypassed the proxy (SB013) or a lazy load
after the session closed (SB018), sit on the bar of the transaction they belong to: a warning frame and the rule's icon
on the bar, the finding in its tooltip, and a flagged line in its event list. A bypass outside any transaction is the
rule's icon on its thread's lane; hover it for the finding, click it for the rule page.

## "Transaction marked as rollback-only": what caused it?

A method that joined the transaction threw, Spring marked the whole transaction rollback-only, the caller caught the
exception and carried on, and the commit at the end threw `UnexpectedRollbackException`. The transaction's tab names
the method that marked it, with the exception:

```text
Rolled back · 19:39:25.247 · 5 ms · 2 statements · connection 3, transaction 239
OrderValidationException: book 1 has 2 copies, 100 needed in OrderValidator.validate marked the transaction
rollback-only; the outer commit rolled it back and threw UnexpectedRollbackException.
```

The event list shows the `join` of `OrderValidator.validate`, the exception that left it, the `marked` line and the
final `rollback`. Rule [SB011](rules.md#sb011-unexpected-rollback) flags every statement of that transaction.

With CDI `@Transactional` the caller does not even get an exception: the interceptor rolls the marked transaction
back quietly when the outer method returns. The transaction's tab names the participant in the same way.

## `@Transactional` method called from the same class does nothing

The call went to `this`, not through Spring's proxy, so the annotation did not apply: the method ran without a
transaction, or silently inside the caller's, ignoring its own `REQUIRES_NEW`, `readOnly` or `rollbackFor`. The agent
checks every call of an annotated method in your packages at run time, so it also catches private, final and static
methods and beans Spring did not proxy. The event list shows "@Transactional ProxyBypassScenarios.recordAttempt ran
without the transaction proxy", and [SB013](rules.md#sb013-transactional-did-not-apply) names the caller's line. A
bypass that ran no statement still shows on the Transactions tab: on the bar of its transaction, or as the rule's icon on
its thread's lane when it ran outside one.

In the editor, the inspection "@Transactional method called on this" marks such calls in Java and Kotlin code when
the called method's settings differ from the caller's, and says so when the method is final (a Kotlin function that is
not `open` and whose class the kotlin-spring plugin does not open): then no call reaches it through a class-based
proxy. The inspection stays silent where JetBrains' own inspection for this ("@Transactional self-invocation
method calls" of the Spring plugin) is enabled, so each call is marked once.

## `LazyInitializationException: could not initialize proxy - no Session`

The agent records each `LazyInitializationException` when it is created: the entity or collection (`Author.books`)
and the line of the access. SQL Blame adds the query that loaded the entity, the transaction it ran in and when that
transaction ended. The loading query is the SELECT of that transaction that read the owner entity's table, taken from
Hibernate's own mapping, and among several the one with the owner's id. [SB018](rules.md#sb018-lazy-loading-after-the-session-closed) flags the loading query:

```text
SB018: Author.books was loaded here and accessed at LazyLoadingScenarios.countBooksAfterTransaction
(LazyLoadingScenarios.java:22) after @Transactional AuthorService.findAuthor ended at 19:39:24.839.
```

Below it are fix options, each with a snippet and **Copy**: `@EntityGraph(attributePaths = "books")` on the Spring
Data method, `JOIN FETCH` in the query, a transaction around the method that reads the data, and a DTO projection.
The `@EntityGraph` and `@Transactional` options also have **Apply...**, which shows the change as a diff and edits your
code only after you confirm, and only in your project's sources. When the loading line calls a method your repository
inherits from Spring Data, such as `findById`, Apply declares it again in your repository interface with the
annotation; Spring Data's own interfaces are never changed. A loop that touches the same association of many
entities shows one block with the number of accesses and one set of fix options, one block per association.

The loading query is inferred, not recorded. The table names come from Hibernate's mapping, which the agent reads once
per session factory. After an attach that mapping arrives with the first insert, update or delete, so a lazy load
before it falls back to the entity name in snake case (`Author` -> `author`). When no SELECT of the transaction read
the table, the exception still shows with its flag on the Transactions tab (on the bar of the transaction that ended last
on that thread, and in its event list), without a flagged query.

## What does `spring.jpa.open-in-view` do?

Spring Boot keeps the persistence context open for the whole web request, so lazy associations still load after the
service transaction ended, in the controller, the view or the JSON serializer. Each such load is a query outside any
transaction. SQL Blame shows the request as its own scope on the timeline ("Web request ... (Open Session in View)"
with the number of queries under it), and [SB019](rules.md#sb019-query-outside-the-transaction-open-session-in-view)
flags the SELECTs that ran after the request's transaction ended:

```text
SB019: Ran in request /authors/2/titles after @Transactional AuthorService.findAuthor ended at 19:39:25.210.
```

## Rollback on a `RuntimeException`, but not on a checked exception

By default Spring rolls back for unchecked exceptions and errors only. When a checked exception (or one listed in
`noRollbackFor`) leaves a `@Transactional` method, the transaction commits and the caller still sees the failure. The
event list says which rule decided:

```text
exception OutOfStockException: book 2 is reserved by someone else left CheckedExceptionScenarios.reserveCopy:
does not roll back (default rule: unchecked exceptions and errors roll back)
commit @Transactional CheckedExceptionScenarios.reserveCopy
```

[SB017](rules.md#sb017-committed-despite-an-exception) flags the transaction's statements. CDI `@Transactional` has
the same default, with `rollbackOn` and `dontRollbackOn` in place of `rollbackFor` and `noRollbackFor`.

## Writes in a `@TransactionalEventListener` are lost

An `AFTER_COMMIT` listener runs after the transaction is over. A write there is not part of it: with JPA the change is
never flushed, with plain JDBC it depends on the connection. Statements run from `afterCommit` and `afterCompletion`
callbacks carry the phase, and [SB020](rules.md#sb020-write-after-commit) flags writes there that did not start a
new transaction (`REQUIRES_NEW`), saying what became of them. The agent notes where each transaction began, so a
`REQUIRES_NEW` method called from the listener is clean, while a write in the callback of a `REQUIRES_NEW`
transaction that began earlier and has just committed is flagged. A JPA change that was never flushed produces no
statement, so SQL Blame can only show the listener's other statements with their phase. The same holds for the
`afterCompletion` of a JTA synchronization; its `beforeCompletion` still runs inside the transaction and is not
flagged.

## Insert after delete in the same transaction fails with a unique violation

Hibernate does not run a flush in the order your code called `remove` and `persist`: inserts and updates go first,
deletes last. Code that deletes a row and inserts one with the same unique key then fails. At the start of every flush
the agent reads Hibernate's pending actions, so the event list shows the flush plan, for example "flush before commit:
1 insert, 1 collection removal, 1 collection insert, 1 delete", and
[SB021](rules.md#sb021-unique-violation-from-the-flush-order) flags the failed insert:

```text
SB021: Unique violation in the insert phase of the flush, while 1 deletes on tx_label were still waiting:
Hibernate runs deletes last.
```

The waiting delete is read from Hibernate, not guessed from the SQL order.

## `@Transactional` tests: "rollback does not work", "data not visible in another thread"

A test's own transaction is labelled "Test transaction" and its cause says "rolled back after the test, as Spring does
unless the test has @Commit". Work handed to another thread through `@Async` or a Spring task executor or scheduler
shows on the timeline as a task linked to the transaction it was started in ("task from thread Test worker"). It runs
outside that transaction: its writes commit on their own and it cannot see the transaction's uncommitted rows.
[SB022](rules.md#sb022-write-in-a-task-started-inside-a-transaction) flags those writes.

Threads started with `new Thread` or `CompletableFuture` on the common pool show as separate lanes without a link.

## Spring Batch: chunk rollback, skip, "processor called twice"

Each chunk transaction is a scope named after its step and number, "Chunk 2 of step importLabels (Spring Batch)",
with a `chunk` line for its items, writes and skips. When a write fails, the chunk rolls back, and the cause names the
exception and the place that marked it. A fault-tolerant step then retries the items one per transaction to find the
bad one; these transactions are labelled "Skip scan of chunk 2 of step importLabels", which is why the processor and
the writer run again for the same items. At the end of a step the timeline adds a summary line with its commits,
rollbacks and skips.

Statements of Spring Batch's job repository (`BATCH_*` tables) are marked: **Hide Spring Batch Metadata** removes
transactions that only touch them, and rules SB012, SB015 and SB022 ignore them.

When a writer's statements ran in auto-commit while the chunk was active (a step with another transaction manager
than the writer's `DataSource`, or `ResourcelessTransactionManager`),
[SB012](rules.md#sb012-write-outside-the-transaction) flags them: they stay when the chunk rolls back.

## When does Hibernate send my SQL?

Every Hibernate statement says which flush sent it: the flush before commit, an auto flush before a query (with that
query's line), or an explicit `flush()` (with its line). An INSERT or DELETE also names the call that scheduled it:

```text
INSERT of Label #36, scheduled at FlushScenarios.createLabelThenQuery (FlushScenarios.java:35); sent at the auto
flush before a query at FlushScenarios.createLabelThenQuery (FlushScenarios.java:36).
```

An insert of an entity whose id the database generates is sent at once, and the statement says so. The scheduling
line is taken once per transaction source: when one `@Transactional` method persists the same entity class from two
lines in different transactions, both show the line seen first. A double-click on the event line of a flush before
commit opens the method that started the transaction.

## Why does Hibernate run this UPDATE?

No `save()` is needed: at the flush, Hibernate compares every loaded entity with its state at loading time and
updates the changed ones (dirty checking). The statement names the entity, its id and the changed properties:

```text
UPDATE of Label #1000002 because name changed (dirty checking); sent at the flush before commit.
```

Inserts, deletes and collection changes are named the same way ("collection insert of Label.aliases").

## The other rules

Three rules have no forum question of their own but come from the same history:

- [SB014](rules.md#sb014-long-transaction) Long transaction: open for 5 s or longer (threshold in Settings), with how
  long the rows of its first write stayed locked
- [SB015](rules.md#sb015-write-in-a-read-only-transaction) Write in a read-only transaction
- [SB016](rules.md#sb016-requires_new-in-a-loop) `REQUIRES_NEW` in a loop: 10 or more new transactions from one place
  inside one outer transaction (threshold in Settings), each on a second pool connection

Overlapping transactions on different threads that write to the same table are outlined on the timeline as lock
candidates. When a statement waited for a lock (SB009) and the other writer ran in the same JVM, its tooltip names it.

## Settings

Both switches are in Settings | Tools | SQL Blame and apply to runs started or attached afterwards.

**Record transaction history** is on by default. When it is off, the agent installs none of its transaction hooks and
the IDE looks as it did before the feature: no Transaction entry in the query detail, no Transactions tab, no outcome in Executions and
no rules SB011-SB022.

**Include Hibernate entity actions and flushes**, under it and also on by default, is the most expensive part on
Hibernate batch jobs. Turned off, it drops the entity actions, scheduling calls, flush causes, flush plans and
`LazyInitializationException` records, so SB018 and SB021 have nothing to report. Transaction scopes and outcomes stay.

Each rule is turned on or off, and the thresholds of SB014 and SB016 are changed, in Settings | Tools | SQL Blame |
Rules.

## Cost

Measured on 2026-11-18 on the machine of the README's [Cost of the agent](../README.md#cost-of-the-agent), on top of
the agent with the history off:

- In short test JVMs the full history added between 0 and 3.8 percentage points of test time: 1.4-2.1 points against
  Postgres in Docker, up to 3.8 against a fast local database, where the agent's time is a larger share of the total.
  Most of it is the warm-up of the agent's new code in JVMs that end after a few seconds: reading each transaction's
  attributes and preparing Spring's and Hibernate's classes.
- In a running Hibernate application, once that code is compiled: +1.5% with the agent started with the JVM and +3.0%
  after an attach (+0.8% / +1.3% on a JdbcTemplate workload).
- No event is dropped for it: every test JVM received every query.

## Attached applications

**Attach SQL Blame...** ([attach.md](attach.md)) records the history too: Spring's and Hibernate's classes are already
loaded and are changed in place like the JDBC classes. Transactions that were open when SQL Blame attached are shown as
"started before SQL Blame was attached". `@Transactional` classes of your packages that were loaded before the attach
are checked for proxy bypasses as well. Statements on a connection the pool opened before the attach are not captured,
so an entity action whose statement went over such a connection says so in the event list.

## Run files and privacy

Exported `.sqlblame` files and the run history carry the transaction history (file version 4). Runs from older files,
from an older agent or with the setting off say "This run was recorded without transaction details".

The history adds names of classes, methods, entities, properties, tables, transactions and exceptions, entity ids
and exception messages, cut to 300 characters. With **Mask parameter values** on, the agent leaves entity ids and
exception messages out, since they may contain values, and an export with masking leaves them out of the file.

## What is not covered

- Application servers with isolated class loaders, and R2DBC: SQL Blame does not capture their queries at all.
- Threads started without a Spring executor: shown without a link to the transaction they came from. `@Async`
  methods are linked on every executor. A task given to a Spring executor directly is linked when Spring's own code
  runs around it on the worker thread: `ThreadPoolTaskExecutor` from Spring 6.1 on, the task schedulers, and
  `SimpleAsyncTaskExecutor` with a concurrency limit, each also with the application's `TaskDecorator`. Not linked:
  `SimpleAsyncTaskExecutor` without a limit (Spring Boot's executor with virtual threads), `ConcurrentTaskExecutor`
  or `TaskExecutorAdapter` over a plain JDK executor, `ThreadPoolTaskExecutor` before Spring 6.1, and on Spring 5.3
  some `execute` / `submit` calls of the schedulers (their `schedule` methods are linked). A task scheduled outside a
  transaction, as `@Scheduled` methods are, has no transaction to link to. A repeating schedule
  (`scheduleAtFixedRate`, a cron trigger) started inside a transaction is linked for its first run only.
- JTA: a scope shows the timeout as its only attribute; JTA has no isolation level or read-only flag of its own, so
  rule SB015 does not apply. Rule SB012 does not find an XA connection that was not enlisted, only a write on a
  connection without XA. CDI `@Transactional` was tested with Narayana's interceptors in Weld SE (Weld 3 and 6), not
  in Quarkus or an application server.
- Transactions ended by a `COMMIT` statement in SQL or implicitly by the database.
- Tested with Spring Framework 5.3, 6.2 and 7.0, Hibernate 5.6, 6.6 and 7.4, Spring Batch 4, 5 and 6, on Java 8, 17,
  21 and 25.
