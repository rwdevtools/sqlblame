# SQL Blame rules

SQL Blame checks every captured query against the rules below and marks the queries that break one (the Flags
column, "Only flagged", the issues of a query's detail). The rules run in the IDE on what the agent captured. Nothing
is sent anywhere and the application is not slowed down by them.

Turn rules on or off and change their thresholds in **Settings | Tools | SQL Blame | Rules**. A rule that cannot be
sure stays silent: SQL Blame prefers a missed issue over a false alarm.

**Scope.** Several rules count executions "in one transaction". With the current agent the transaction is exact: the
agent sees `commit`, `rollback` and the auto-commit mode of every connection, so two transactions are never mixed up,
also when the same method runs twice in a row. Statements in auto-commit mode are each their own transaction. For the
counting rules they form a *burst*: consecutive statements on one thread with at most one second between them. A
statement of a transaction on that thread ends the burst.
Queries from older agents fall back to an approximation (same thread, same Spring transaction, same connection).

**Transaction rules.** SB011-SB022 judge whole transactions: who opened them, how they ended and why, Hibernate's
flushes, work handed to other threads. They need the transaction history (Settings | Tools | SQL Blame, **Record
transaction history**, on by default) and are off while it is off. SB018 and SB021 also need its Hibernate part. What
the history shows besides the rules is on [transactions.md](transactions.md).

Each rule below has the same parts as its page in the IDE (**SQL Blame Rules** in the tool window's gear menu, or a
click on a Flags icon): why it is an issue, how to fix it per stack, how SQL Blame detects it, and resources.

## Related SonarQube rules

SonarQube checks source code, SQL Blame checks the statements that actually ran. Where a SonarQube rule covers the
same problem, the table names it and says what SQL Blame does differently. The keys are SonarQube's rule keys, which
the Rules page of SonarQube Server or SonarQube Cloud finds; the public site rules.sonarsource.com that used to show
them is no longer online.

| Rule | Related SonarQube rules | What SQL Blame does differently |
|---|---|---|
| SB002 Unbatched writes | `java:S6912` (batch processing in JDBC, a `Statement` executed in a loop) | Also catches `PreparedStatement` loops and the writes an ORM sends one by one, and needs the threshold of executions with different values. |
| SB005 Large result set | `java:S6905`, `plsql:S1499`: `SELECT *` | Counts the rows read, not the columns selected. A `SELECT *` alone is not marked. |
| SB006 UPDATE / DELETE without WHERE | `plsql:S1590`, `tsql:S1590` | Checks the executed SQL, including SQL built at run time and framework bulk statements, and exempts SQL Server temporary tables as the T-SQL rule does. |
| SB013 @Transactional did not apply | `java:S2229` (same-class call of a method with incompatible `@Transactional` values), `java:S2230` (`@Transactional` on a non-public method) | Sees the calls that bypassed the proxy at run time, also final and static methods and beans Spring did not proxy, and marks the statements that ran without the intended transaction. Its editor inspection reports calls on `this` in Java and Kotlin. |

The other rules have no counterpart there: they depend on what only exists at run time (executions, parameter values,
transactions, errors).

<!-- Generated from src/main/resources/rules/SBxxx.md by scripts/rules-doc.sh: edit those files, not the text below. -->

| Id | Rule | Severity | Type |
|---|---|---|---|
| SB001 | N+1 select | Major | Performance |
| SB002 | Unbatched writes | Minor | Performance |
| SB003 | Repeated identical query | Minor | Performance |
| SB004 | Slow query | Major | Performance |
| SB005 | Large result set | Minor | Performance |
| SB006 | UPDATE / DELETE without WHERE | Critical | Data safety |
| SB007 | Leading-wildcard LIKE | Minor | Performance |
| SB008 | Large IN list | Minor | Performance |
| SB009 | Lock wait / deadlock | Critical | Reliability |
| SB010 | SQL error | Major | Reliability |
| SB011 | Unexpected rollback | Critical | Data safety |
| SB012 | Write outside the transaction | Major | Data safety |
| SB013 | @Transactional did not apply | Major | Reliability |
| SB014 | Long transaction | Major | Performance |
| SB015 | Write in a read-only transaction | Minor | Reliability |
| SB016 | REQUIRES_NEW in a loop | Minor | Performance |
| SB017 | Committed despite an exception | Major | Data safety |
| SB018 | Lazy loading after the session closed | Major | Reliability |
| SB019 | Query outside the transaction (Open Session in View) | Minor | Performance |
| SB020 | Write after commit | Major | Data safety |
| SB021 | Unique violation from the flush order | Major | Reliability |
| SB022 | Write in a task started inside a transaction | Major | Data safety |

<!-- rule SB001 -->
## SB001 N+1 select

Severity: Major. Type: Performance. Tags: `n+1`, `jpa`, `mybatis`, `jooq`. Threshold: 5 executions (default, set in Settings | Tools | SQL Blame | Rules).

### Why is this an issue?

The application loads a list with one query and then runs one more query for every row of it: 1 + N round trips
where one or two would do. Each round trip pays for network latency, parsing and planning. With 200 rows that is 201
queries. The cost grows with the data, so the problem often stays hidden in tests with a handful of rows and shows up
in production.

### How to fix it

Load what the code needs together with the list: in the same query (a join) or in one more query for all rows at once
(an `IN` list or a batch fetch).

#### JPA / Hibernate

Noncompliant: a lazy collection touched in a loop runs one SELECT per author.

```java
List<Author> authors = em.createQuery("select a from Author a", Author.class).getResultList();
for (Author author : authors) {
    author.getBooks().size();
}
```

Compliant: fetch the association with the list, or let Hibernate load it in batches.

```java
em.createQuery("select distinct a from Author a join fetch a.books", Author.class).getResultList();

@BatchSize(size = 50) // on the association: one query per 50 authors
@OneToMany(mappedBy = "author")
private List<Book> books;
```

#### Spring Data

Noncompliant: one `findById` per id.

```java
ids.forEach(id -> repository.findById(id));
```

Compliant: one query for all ids, or an entity graph on the repository method.

```java
repository.findAllById(ids);

@EntityGraph(attributePaths = "books")
List<Author> findAll();
```

#### MyBatis

Noncompliant: a nested select runs once per parent row.

```xml
<collection property="books" column="id" select="selectBooksByAuthor"/>
```

Compliant: a nested result map over one join.

```xml
<collection property="books" ofType="Book" resultMap="bookResult" columnPrefix="book_"/>
```

#### jOOQ

Noncompliant: one `fetch` per parent row.

```java
for (AuthorRecord author : ctx.selectFrom(AUTHOR).fetch()) {
    ctx.selectFrom(BOOK).where(BOOK.AUTHOR_ID.eq(author.getId())).fetch();
}
```

Compliant: the children in the same query with `multiset`.

```java
ctx.select(AUTHOR.NAME,
        multiset(select(BOOK.TITLE).from(BOOK).where(BOOK.AUTHOR_ID.eq(AUTHOR.ID))))
    .from(AUTHOR)
    .fetch();
```

### How SQL Blame detects it

SQL Blame marks an N+1 select when all of these hold:

- the statement is a SELECT (also `WITH ... SELECT`) with parameters (placeholders or literals)
- the same normalized SQL from the same line of application code ran at least 5 executions one by one in one
  transaction
- at least two of those executions had different parameter values

All of those executions are marked.

Code analyzers look for a query call inside a loop in the source. SQL Blame works from the executions, so it also
finds the loops the ORM runs for you (lazy loading, nested selects) and stays silent when a loop runs only once.

Limits: outside an exact transaction (auto-commit bursts, older agents) the rule needs twice the threshold. Executions
without application code in the call path are not counted. With masked parameter values the values cannot be
compared, so the rule stays silent. The same values repeated are SB003, not SB001. Batches never count.

### Resources

- [Hibernate ORM User Guide: Fetching](https://docs.hibernate.org/orm/6.6/userguide/html_single/Hibernate_User_Guide.html#fetching)
- [Spring Data JPA: Query methods, entity graphs](https://docs.spring.io/spring-data/jpa/reference/jpa/query-methods.html)
- [MyBatis: Mapper XML files, result maps](https://mybatis.org/mybatis-3/sqlmap-xml.html)
- [jOOQ manual: MULTISET value constructor](https://www.jooq.org/doc/latest/manual/sql-building/column-expressions/multiset-value-constructor/)

<!-- rule SB002 -->
## SB002 Unbatched writes

Severity: Minor. Type: Performance. Tags: `batch`, `jdbc`, `jpa`, `spring-batch`. Threshold: 10 executions (default, set in Settings | Tools | SQL Blame | Rules).

### Why is this an issue?

Every INSERT, UPDATE or DELETE sent on its own is a separate round trip. A JDBC batch sends many rows in one round trip
and lets the database apply them together. Importing 10 000 rows one statement at a time is typically 10 to 100 times
slower than in batches of a few hundred.

### How to fix it

Send the rows as one JDBC batch through the framework's batch support instead of calling the single-row method in a
loop.

#### JPA / Hibernate

Noncompliant: `IDENTITY` ids turn insert batching off, because Hibernate must run each INSERT to learn the id.

```java
@Id @GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;
```

Compliant: sequence ids with an allocation size, plus a batch size with ordered inserts and updates.

```java
@Id @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "book_seq")
@SequenceGenerator(name = "book_seq", allocationSize = 50)
private Long id;

// application.properties
// spring.jpa.properties.hibernate.jdbc.batch_size=50
// spring.jpa.properties.hibernate.order_inserts=true
// spring.jpa.properties.hibernate.order_updates=true
```

#### Spring Data

Noncompliant: one `save` per item.

```java
items.forEach(repository::save);
```

Compliant: `saveAll`, with the Hibernate batch size above.

```java
repository.saveAll(items);
```

#### JdbcTemplate

Noncompliant: one `update` per row.

```java
for (Book book : books) {
    jdbc.update("insert into book (id, title) values (?, ?)", book.id(), book.title());
}
```

Compliant: `batchUpdate` with a batch size.

```java
jdbc.batchUpdate("insert into book (id, title) values (?, ?)", books, 500,
    (ps, book) -> { ps.setLong(1, book.id()); ps.setString(2, book.title()); });
```

#### MyBatis

Noncompliant: one mapper call per row in a default (simple) session.

```java
books.forEach(bookMapper::insert);
```

Compliant: a batch session, which sends the statements on `flushStatements` or `commit`.

```java
try (SqlSession session = sqlSessionFactory.openSession(ExecutorType.BATCH)) {
    BookMapper mapper = session.getMapper(BookMapper.class);
    books.forEach(mapper::insert);
    session.commit();
}
```

#### jOOQ

Noncompliant: one `execute` per record.

```java
records.forEach(record -> ctx.insertInto(BOOK).set(record).execute());
```

Compliant: one batch for all records.

```java
ctx.batchInsert(records).execute();
```

#### Spring Batch

Noncompliant: an `ItemWriter` that writes the chunk item by item.

```java
ItemWriter<Book> writer = chunk -> chunk.forEach(book -> jdbc.update(INSERT_BOOK, book.id(), book.title()));
```

Compliant: a batching writer sends one JDBC batch per chunk.

```java
new JdbcBatchItemWriterBuilder<Book>()
    .dataSource(dataSource)
    .sql("insert into book (id, title) values (:id, :title)")
    .beanMapped()
    .build();
```

### How SQL Blame detects it

SQL Blame marks unbatched writes when all of these hold:

- the statement is an INSERT, UPDATE, DELETE or MERGE
- the same normalized SQL from the same line of application code ran at least 10 executions as single executions (not
  `executeBatch`) in one transaction or burst
- at least two of those executions had different parameter values

All of those executions are marked.

SonarQube's S6912 flags a `Statement` executed in a loop. SQL Blame sees the executions, so it also catches the
writes an ORM sends one by one (`IDENTITY` ids, no batch size) and loops over a `PreparedStatement`, and it ignores
loops that run once.

Limits: executions without application code in the call path are not counted, and different statements never add up.
The same write with the same values every time (a set-based `INSERT ... SELECT` run once per test, a retry) is not a
loop over rows and is not marked. With masked parameter values the values cannot be compared, so the rule stays
silent.

### Resources

- [Hibernate ORM User Guide: Batching](https://docs.hibernate.org/orm/6.6/userguide/html_single/Hibernate_User_Guide.html#batch)
- [Spring Framework: JDBC batch operations](https://docs.spring.io/spring-framework/reference/data-access/jdbc/advanced.html)
- [MyBatis: Java API, SqlSessionFactory](https://mybatis.org/mybatis-3/java-api.html)
- [jOOQ manual: Batch execution](https://www.jooq.org/doc/latest/manual/sql-execution/batch-execution/)
- [Spring Batch: Database item readers and writers](https://docs.spring.io/spring-batch/reference/readers-and-writers/database.html)
- Related rules: SonarQube rule `java:S6912` (batch processing in JDBC), which
  flags a `java.sql.Statement` executed in a loop.

<!-- rule SB003 -->
## SB003 Repeated identical query

Severity: Minor. Type: Performance. Tags: `transaction`, `jpa`, `cache`. No threshold.

### Why is this an issue?

The same SELECT with the same values ran again in the same transaction, with no write in between. Inside the
transaction it returns what the application already had, so the second round trip is wasted work. It often points to
two layers that load the same data without knowing about each other.

### How to fix it

Keep the first result: pass it down instead of loading it again, or cache it for the transaction or longer.

#### JPA / Hibernate

Noncompliant: a query by id runs SQL every time, even for an entity that is already loaded.

```java
Author a = em.createQuery("select a from Author a where a.id = :id", Author.class)
    .setParameter("id", id).getSingleResult();
Author b = em.createQuery("select a from Author a where a.id = :id", Author.class)
    .setParameter("id", id).getSingleResult();
```

Compliant: `find` returns the entity from the persistence context without SQL the second time.

```java
Author a = em.find(Author.class, id);
Author b = em.find(Author.class, id); // no SQL
```

#### Spring

Noncompliant: two services load the same reference data in one request.

```java
Country country = countryRepository.findByCode(code); // in OrderService and again in InvoiceService
```

Compliant: load it once and pass it on, or cache data that rarely changes.

```java
@Cacheable("countries")
public Country country(String code) {
    return countryRepository.findByCode(code);
}
```

#### JdbcTemplate

Noncompliant: the same lookup inside a loop over items that share a value.

```java
for (Order order : orders) {
    String name = jdbc.queryForObject("select name from customer where id = ?", String.class, customerId);
}
```

Compliant: query once, before the loop.

```java
String name = jdbc.queryForObject("select name from customer where id = ?", String.class, customerId);
for (Order order : orders) { /* use name */ }
```

### How SQL Blame detects it

SQL Blame marks a repeated query when all of these hold:

- the statement is a SELECT
- the same SQL text with the same parameter values ran at least twice in one transaction
- no INSERT, UPDATE, DELETE, MERGE or other non-SELECT statement ran in that transaction between them

All of those executions are marked.

Limits: only inside a transaction. In auto-commit mode each statement may see the changes of other sessions, so a
repeat can be intended (polling, for example). SELECTs whose result changes by itself (`nextval`, `NEXT VALUE FOR`,
`random()`, `now()`, `uuid()`, advisory locks and similar) are left out. Masked, truncated or uncaptured values,
batches and failed executions are not compared.

### Resources

- [Hibernate ORM User Guide: Persistence context](https://docs.hibernate.org/orm/6.6/userguide/html_single/Hibernate_User_Guide.html#pc)
- [Spring Framework: Cache abstraction](https://docs.spring.io/spring-framework/reference/integration/cache.html)

<!-- rule SB004 -->
## SB004 Slow query

Severity: Major. Type: Performance. Tags: `index`, `sql`. Threshold: 100 ms (default, set in Settings | Tools | SQL Blame | Rules).

### Why is this an issue?

A statement that takes long holds a database connection, and often locks, for all that time, and adds directly to the
response time of the code waiting for it. Under load, slow statements exhaust the connection pool and other requests
queue behind them.

### How to fix it

Open the query in an SQL console (Open in SQL Scratch) and look at its plan: `EXPLAIN ANALYZE` on PostgreSQL,
`EXPLAIN ANALYZE` or `EXPLAIN` on MySQL, the actual execution plan on SQL Server. Typical causes are a missing index on
a filtered or joined column, a function on an indexed column, a large sort without an index, or too much data
(see SB005).

#### SQL

Noncompliant: a function on the indexed column stops the database from using the index.

```sql
select * from customer where lower(email) = ?
```

Compliant: compare the column itself, or index the expression.

```sql
create index customer_email_lower on customer (lower(email));
select * from customer where lower(email) = ?
```

#### JPA / Hibernate

Noncompliant: sorting a large table by a column without an index.

```java
em.createQuery("select i from Invoice i order by i.createdAt desc", Invoice.class).setMaxResults(20).getResultList();
```

Compliant: an index matching the order, declared on the entity (or in the migration).

```java
@Entity
@Table(indexes = @Index(name = "invoice_created_at", columnList = "created_at"))
class Invoice { /* ... */ }
```

### How SQL Blame detects it

The execution took at least 100 ms, measured around the JDBC call in the application: database time plus network
time. For a batch the whole batch is measured, and the issue names the number of statements in it.

Execution time is not visible in the source. SQL Blame measures every execution, so the result depends on the data and
the database of the run: the same query can be fast in a test and slow in production.

Limits: the time includes the network and the JDBC driver, and the first execution of a statement can be slower
(connection setup, plan caching). Rows fetched lazily after the call returns are not part of the measured time.

### Resources

- [PostgreSQL: Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)
- [MySQL: Optimizing queries with EXPLAIN](https://dev.mysql.com/doc/refman/8.4/en/using-explain.html)
- [PostgreSQL: Indexes on expressions](https://www.postgresql.org/docs/current/indexes-expressional.html)

<!-- rule SB005 -->
## SB005 Large result set

Severity: Minor. Type: Performance. Tags: `paging`, `memory`, `sql`. Threshold: 1,000 rows (default, set in Settings | Tools | SQL Blame | Rules).

### Why is this an issue?

Every row read is sent over the network, turned into objects and kept in memory. Reading thousands of rows to show a
page of twenty, or to count or sum them in Java, wastes database, network and heap. Large results also grow with the
data, so a query that is fine today can run the application out of memory later.

### How to fix it

Filter and aggregate in the database (`WHERE`, `COUNT`, `SUM`), page the result, or stream it with a fetch size when
every row is needed.

#### JPA / Hibernate

Noncompliant: all rows are loaded to count them in Java.

```java
int open = em.createQuery("select i from Invoice i where i.open = true", Invoice.class).getResultList().size();
```

Compliant: count in the database.

```java
long open = em.createQuery("select count(i) from Invoice i where i.open = true", Long.class).getSingleResult();
```

#### Spring Data

Noncompliant: `findAll` to show one page.

```java
List<Invoice> invoices = repository.findAll();
```

Compliant: a `Pageable`, or a `Slice` when the total count is not needed.

```java
Page<Invoice> page = repository.findAll(PageRequest.of(0, 20, Sort.by("createdAt").descending()));
```

#### JdbcTemplate

Noncompliant: the whole table in one list.

```java
List<Row> rows = jdbc.query("select * from event", rowMapper);
```

Compliant: a fetch size and row-by-row processing when every row is needed.

```java
jdbc.setFetchSize(500);
jdbc.query("select * from event", (RowCallbackHandler) rs -> process(rs));
```

#### Spring Batch

Noncompliant: a reader that loads the whole table into a list.

```java
new ListItemReader<>(jdbc.query("select * from event", rowMapper));
```

Compliant: a paging or cursor reader.

```java
new JdbcPagingItemReaderBuilder<Event>()
    .name("events").dataSource(dataSource).pageSize(500)
    .selectClause("select *").fromClause("from event").sortKeys(Map.of("id", Order.ASCENDING))
    .rowMapper(rowMapper).build();
```

### How SQL Blame detects it

The application read at least 1,000 rows through the result set of the execution. The issue adds "without WHERE or
LIMIT" when the SELECT has neither at its top level.

From the SQL alone the size of a result can only be guessed (`SELECT *`, a missing `WHERE`). SQL Blame counts the
rows read, so it also finds large results of filtered queries and stays silent for small tables.

Limits: only rows read through `ResultSet#next` count. When the agent reports no row count, the rule is silent. The
rule counts rows, not columns: a `SELECT *` with few rows is not marked.

### Resources

- [Spring Data JPA: Paging and sorting](https://docs.spring.io/spring-data/jpa/reference/repositories/query-methods-details.html#repositories.special-parameters)
- [PostgreSQL: LIMIT and OFFSET](https://www.postgresql.org/docs/current/queries-limit.html)
- [Spring Batch: Database item readers and writers](https://docs.spring.io/spring-batch/reference/readers-and-writers/database.html)
- Related rules: SonarQube rules `java:S6905` (SQL queries should retrieve only
  the necessary fields) and `plsql:S1499` for PL/SQL (`SELECT *`) look at the columns, SB005 looks at the rows.

<!-- rule SB006 -->
## SB006 UPDATE / DELETE without WHERE

Severity: Critical. Type: Data safety. Tags: `sql`, `data-loss`. No threshold.

### Why is this an issue?

An UPDATE or DELETE without a WHERE clause changes every row of the table. In application code that is almost always a
bug, such as a lost condition or a call to the wrong method, and it can destroy data that a rollback in the test does
not bring back in production.

### How to fix it

Add the condition. If changing or removing every row is intended, make that visible in the code.

#### SQL

Noncompliant: every account is blocked.

```sql
update account set blocked = true
```

Compliant: only the account meant.

```sql
update account set blocked = true where id = ?
```

#### Spring Data

Noncompliant: a modifying query that lost its condition.

```java
@Modifying
@Query("update Account a set a.blocked = true")
int block(@Param("id") long id);
```

Compliant: the condition uses the parameter.

```java
@Modifying
@Query("update Account a set a.blocked = true where a.id = :id")
int block(@Param("id") long id);
```

#### JdbcTemplate

Noncompliant: a cleanup that deletes everything instead of the expired rows.

```java
jdbc.update("delete from session");
```

Compliant: the condition, or a named method when emptying the table is the point.

```java
jdbc.update("delete from session where expires_at < ?", now);
```

To empty a table on purpose, `TRUNCATE` states the intent, but it commits implicitly on MySQL and Oracle.

### How SQL Blame detects it

An UPDATE or DELETE (also after `WITH ...`) has no `WHERE` at the level of the statement itself. A WHERE inside a subquery does not count. Every statement of a `Statement` batch is checked.

Source checks see the SQL written in the code. SQL Blame checks the SQL that ran, so it also sees statements built at
run time and the SQL that frameworks generate: Spring Data's `deleteAllInBatch` runs `delete from <table>` and is
marked too.

Limits: `TRUNCATE` is not marked, nor `WHERE CURRENT OF`, nor an UPDATE or DELETE with a `JOIN` at the top level
(MySQL and SQL Server syntax, where the join restricts the rows), nor a SQL Server temporary table (`#name`). Only the
first statement of a text with several statements is checked.

### Resources

- [PostgreSQL: DELETE](https://www.postgresql.org/docs/current/sql-delete.html)
- [MySQL: TRUNCATE TABLE](https://dev.mysql.com/doc/refman/8.4/en/truncate-table.html)
- Related rules: SonarQube rule S1590, UPDATE and DELETE statements should contain WHERE clauses
  (`plsql:S1590`, `tsql:S1590`).
  The T-SQL version exempts temporary tables, as SB006 does.

<!-- rule SB007 -->
## SB007 Leading-wildcard LIKE

Severity: Minor. Type: Performance. Tags: `index`, `sql`. No threshold.

### Why is this an issue?

A B-tree index can only narrow a `LIKE` search by the pattern's leading characters. A pattern that starts with `%` or
`_` gives the index nothing to start from, so the database reads every row (or every index entry) of the table. The
query gets slower as the table grows.

### How to fix it

Search by prefix where that is enough. For "contains" searches use the database's text search: a trigram index
(`pg_trgm` with a GIN index) or full-text search on PostgreSQL, a FULLTEXT index on MySQL, full-text search on SQL
Server, or a search engine.

#### SQL

Noncompliant: a "contains" search.

```sql
select * from book where title like '%' || ? || '%'
```

Compliant: a prefix search, which can use an index on `title`.

```sql
select * from book where title like ? || '%'
```

#### JPA / Spring Data

Noncompliant: `Containing` and `EndingWith` derive patterns that start with `%`.

```java
List<Book> findByTitleContaining(String part);
```

Compliant: `StartingWith`, or a native query against a text search index.

```java
List<Book> findByTitleStartingWith(String prefix);
```

#### PostgreSQL

Compliant when a "contains" search is required: a trigram index serves `LIKE '%...%'`.

```sql
create extension if not exists pg_trgm;
create index book_title_trgm on book using gin (title gin_trgm_ops);
```

### How SQL Blame detects it

The right operand of a `LIKE` or `ILIKE` is a literal that starts with `%` or `_`, or a parameter whose bound value
starts with `%` or `_`. SQL Blame looks inside `lower(...)`, `upper(...)`, `concat(...)` and `cast(...)`, and at the
first part of `'%' || ...`.

In the source a pattern is visible only when it is a literal. SQL Blame sees the bound values, so it also finds
patterns built in Java and the patterns that Spring Data derives.

Limits: masked values are not checked. A pattern built in a way SQL Blame does not follow (another function, a
column) is not checked. A trigram index makes such a search fast, but the rule still marks it.

### Resources

- [PostgreSQL: pg_trgm](https://www.postgresql.org/docs/current/pgtrgm.html)
- [PostgreSQL: Index types](https://www.postgresql.org/docs/current/indexes-types.html)
- [MySQL: Full-text search functions](https://dev.mysql.com/doc/refman/8.4/en/fulltext-search.html)

<!-- rule SB008 -->
## SB008 Large IN list

Severity: Minor. Type: Performance. Tags: `sql`, `jpa`, `statement-cache`. Threshold: 100 elements (default, set in Settings | Tools | SQL Blame | Rules).

### Why is this an issue?

A long `IN (...)` list makes a large statement that the database parses and plans every time. When the number of
elements changes from call to call, every length is a different statement, so the statement caches of the driver and
the database fill up with copies. Some databases also have hard limits: Oracle allows 1 000 expressions in a list, and
SQL Server rejects a request with more than 2 100 parameters (error 8003).

### How to fix it

Pass the values in one parameter, join against them, or select them with a subquery.

#### SQL (PostgreSQL)

Noncompliant: one placeholder per id.

```sql
select * from book where id in (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ...)
```

Compliant: one array parameter.

```sql
select * from book where id = any(?)
```

#### JdbcTemplate

Noncompliant: a named parameter expanded into one placeholder per element.

```java
named.query("select * from book where id in (:ids)", Map.of("ids", ids), rowMapper);
```

Compliant: the array as one parameter (PostgreSQL).

```java
jdbc.query("select * from book where id = any(?)", rowMapper, (Object) ids.toArray(new Long[0]));
```

#### JPA / Hibernate

Noncompliant: a list parameter with a different length every time.

```java
em.createQuery("select b from Book b where b.id in :ids", Book.class).setParameter("ids", ids);
```

Compliant: a subquery or join that selects the rows in the database. For lists that must stay, padding keeps the
statement cache small.

```java
// application.properties
// spring.jpa.properties.hibernate.query.in_clause_parameter_padding=true
```

### How SQL Blame detects it

An `IN (...)` list of values in the executed SQL has at least 100 elements. `IN (subquery)` does not count.

In the source the list is one parameter (`in (:ids)`). SQL Blame sees the executed SQL after the framework expanded
it, so it knows how long the list was.

Limits: only value lists in the SQL text count, and an array parameter is one value.

### Resources

- [Oracle: ORA-01795, maximum number of expressions in a list](https://docs.oracle.com/en/error-help/db/ora-01795/)
- [SQL Server: Maximum capacity specifications](https://learn.microsoft.com/en-us/sql/sql-server/maximum-capacity-specifications-for-sql-server)
- [PostgreSQL: Row and array comparisons](https://www.postgresql.org/docs/current/functions-comparisons.html)
- [Hibernate ORM: in_clause_parameter_padding](https://docs.hibernate.org/orm/6.6/userguide/html_single/Hibernate_User_Guide.html#settings-query)

<!-- rule SB009 -->
## SB009 Lock wait / deadlock

Severity: Critical. Type: Reliability. Tags: `locking`, `transaction`, `concurrency`. No threshold.

### Why is this an issue?

The statement failed because of a conflict with another transaction: a deadlock (the database rolled back one of the
two), a lock wait that timed out, or a serialization failure. Such errors depend on timing. They show up under load
and are hard to reproduce, so a single one in a test run is worth a look.

### How to fix it

Keep transactions short, touch rows and tables in the same order everywhere, and keep user interaction and remote
calls out of transactions. Retry the whole transaction on deadlocks and serialization failures, and lock explicitly
(`SELECT ... FOR UPDATE`) where an update depends on what was read.

#### Spring

Noncompliant: a remote call inside the transaction keeps the row locks for its whole duration.

```java
@Transactional
public void pay(long orderId) {
    Order order = orders.lockById(orderId);
    paymentClient.charge(order); // remote call while holding the locks
    order.markPaid();
}
```

Compliant: the remote call outside, a short transaction, retried on a conflict.

```java
public void pay(long orderId) {
    Charge charge = paymentClient.charge(orders.findById(orderId).orElseThrow());
    retry.execute(() -> tx.executeWithoutResult(status -> orders.lockById(orderId).markPaid(charge)));
}
```

#### SQL

Noncompliant: two transactions update the same rows in opposite order.

```sql
-- transaction 1                              -- transaction 2
update account set balance = balance - 10     update account set balance = balance - 5
  where id = 1;                                 where id = 2;
update account set balance = balance + 10     update account set balance = balance + 5
  where id = 2;                                 where id = 1;
```

Compliant: every transaction locks the rows in the same order (here by id) before it changes them.

```sql
select id from account where id in (1, 2) order by id for update;
```

### How SQL Blame detects it

The execution failed with one of these codes:

| Database | SQLState / vendor code | Meaning |
|---|---|---|
| PostgreSQL | 40P01 | deadlock_detected |
| PostgreSQL | 55P03 | lock_not_available (`NOWAIT`, `lock_timeout`) |
| any (SQL standard) | 40001 | serialization failure (PostgreSQL, MySQL / MariaDB deadlock, SQL Server deadlock, H2 deadlock, DB2) |
| MySQL / MariaDB | 1213 | ER_LOCK_DEADLOCK |
| MySQL / MariaDB | 1205 | ER_LOCK_WAIT_TIMEOUT |
| Oracle | ORA-00060 | deadlock detected while waiting for resource |
| Oracle | ORA-00054 | resource busy and acquire with NOWAIT specified or timeout expired |
| Oracle | ORA-30006 | resource busy; acquire with WAIT timeout expired |
| SQL Server | 1205 | transaction was deadlocked and chosen as the deadlock victim |
| SQL Server | 1222 | lock request time out period exceeded |
| H2 | error code 50200 | LOCK_TIMEOUT_1 (H2 deadlocks, DEADLOCK_1, report 40001) |

Vendor codes are only trusted together with the driver they belong to (exception class `com.mysql.` or
`org.mariadb.`, `com.microsoft.sqlserver.` or `net.sourceforge.jtds.`, `org.h2.`, or the `ORA-nnnnn` text of an Oracle
message), because the same number means something else elsewhere: MySQL 1222 is a column count error.

Lock conflicts cannot be found in the source: they happen only when transactions overlap.

Limits: a MariaDB lock wait timeout reported as SQLState HY000 by a driver that throws plain `java.sql` exceptions is
not recognized and is reported as SB010.

### Resources

- [PostgreSQL: Error codes (appendix A)](https://www.postgresql.org/docs/current/errcodes-appendix.html)
- [PostgreSQL: Explicit locking, deadlocks](https://www.postgresql.org/docs/current/explicit-locking.html)
- [MySQL: Server error message reference](https://dev.mysql.com/doc/mysql-errors/8.4/en/server-error-reference.html)
- [MySQL: Deadlocks in InnoDB](https://dev.mysql.com/doc/refman/8.4/en/innodb-deadlocks.html)
- [Oracle: ORA-00060](https://docs.oracle.com/en/error-help/db/ora-00060/)
- [SQL Server: Database engine errors](https://learn.microsoft.com/en-us/sql/relational-databases/errors-events/database-engine-events-and-errors-1000-to-1999)
- [H2: ErrorCode](https://h2database.com/javadoc/org/h2/api/ErrorCode.html)

<!-- rule SB010 -->
## SB010 SQL error

Severity: Major. Type: Reliability. Tags: `error`, `transaction`. No threshold.

### Why is this an issue?

The statement failed. Even when the application catches the exception, a failed statement usually spoils the
transaction it ran in: PostgreSQL rejects every further statement until the rollback, and Spring marks the transaction
rollback-only. An error that is caught and logged can hide data that was never written.

### How to fix it

Read the error in the detail. The SQLState class says where to look: class 23 (constraint violation) usually means a
missing check before the write or a race between two writers. Class 42 means a typo or a schema that does not match the code, 57014 is a statement timeout or cancel, and class 08 is a lost connection.

#### SQL (PostgreSQL)

Noncompliant: check, then insert. A concurrent writer can insert the same key between the two statements (SQLState
23505).

```sql
select count(*) from tag where name = ?;
insert into tag (name) values (?);
```

Compliant: one statement that handles the conflict.

```sql
insert into tag (name) values (?) on conflict (name) do nothing;
```

#### Spring

Noncompliant: an exception swallowed inside the transaction, which is already broken.

```java
@Transactional
public void importAll(List<Tag> tags) {
    for (Tag tag : tags) {
        try { tagRepository.saveAndFlush(tag); } catch (DataIntegrityViolationException ignored) { }
    }
}
```

Compliant: avoid the error in SQL (above), or give each item its own transaction.

```java
for (Tag tag : tags) {
    try { tx.executeWithoutResult(status -> tagRepository.saveAndFlush(tag)); }
    catch (DataIntegrityViolationException duplicate) { log.debug("tag exists: {}", tag.getName()); }
}
```

### How SQL Blame detects it

The execution threw an exception that is not a lock conflict (SB009). The issue names the class of the error from its
SQLState: integrity constraint violation (23), syntax error or undefined object (42), statement canceled or timed out
(57014), connection error (08), data exception (22), transaction rolled back (40), invalid transaction state (25),
feature not supported (0A), otherwise the exception's class name.

Whether a statement fails is decided at run time. SQL Blame records every failed execution, also those the
application catches and hides.

### Resources

- [PostgreSQL: Error codes (appendix A)](https://www.postgresql.org/docs/current/errcodes-appendix.html)
- [PostgreSQL: INSERT, ON CONFLICT](https://www.postgresql.org/docs/current/sql-insert.html)
- [Spring Framework: Rolling back a declarative transaction](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/rolling-back.html)

<!-- rule SB011 -->
## SB011 Unexpected rollback

Severity: Critical. Type: Data safety. Tags: `transaction`, `spring`, `rollback`. No threshold.

### Why is this an issue?

A method that joined the transaction (Spring's default propagation `REQUIRED`) failed, and Spring marked the whole
transaction rollback-only. The caller caught the exception and carried on, so the code at the top expected a commit.
At the end Spring rolled everything back instead and threw `UnexpectedRollbackException`. Every write of the
transaction is lost, including the writes the caller made after catching the exception.

### How to fix it

Decide which work belongs together. If the inner work may fail without spoiling the rest, give it a transaction of
its own; if it may not, let the exception reach the top instead of catching it.

#### Spring

Noncompliant: the caller catches the exception of a joined method and expects its own writes to commit.

```java
@Transactional
public void importOrders(List<Order> orders) {
    for (Order order : orders) {
        try {
            orderService.save(order); // @Transactional, joins this transaction
        } catch (OrderValidationException invalid) {
            log.warn("skipped {}", order.getId()); // the transaction is already rollback-only
        }
    }
}
```

Compliant: each order in its own transaction, so one failure does not mark the others.

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void save(Order order) { ... }
```

Compliant as well: validate before writing, so nothing fails inside the transaction.

```java
List<Order> valid = orders.stream().filter(validator::isValid).toList();
valid.forEach(orderService::save);
```

#### TransactionTemplate

Noncompliant: an inner template joins and marks the outer transaction.

```java
outer.executeWithoutResult(status -> {
    inner.executeWithoutResult(s -> s.setRollbackOnly()); // marks the shared transaction
    repository.save(entity);                              // rolled back with it
});
```

Compliant: the inner template starts its own transaction.

```java
inner.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRES_NEW);
```

### How SQL Blame detects it

SQL Blame marks every statement of a transaction when all of these hold:

- the scope that began the transaction ended with a commit call; for CDI `@Transactional`, the method that began it
  returned without an exception that rolls back (a CDI interceptor rolls a marked transaction back itself, without
  trying the commit and without telling the caller)
- the transaction was marked rollback-only and rolled back instead of committing
- the mark came from a participant: a joined scope that rolled back, or `setRollbackOnly()` called in an inner scope,
  or the commit threw `UnexpectedRollbackException`

The issue says who marked the transaction, with which exception and where.

Not marked: a rollback the code asked for, a transaction marked rollback-only in its own outermost scope (for
example `status.setRollbackOnly()` in a `TransactionTemplate` callback, which rolls back quietly; Spring Batch 6
does the same with a chunk whose write failed), and test transactions rolled back after the test.

Limits: needs the transaction history (Settings | Tools | SQL Blame). Statements beyond the 10 000 kept per
transaction are not marked.

### Resources

- [Spring Framework: Transaction propagation](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/tx-propagation.html)
- [Spring Framework API: UnexpectedRollbackException](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/UnexpectedRollbackException.html)

<!-- rule SB012 -->
## SB012 Write outside the transaction

Severity: Major. Type: Data safety. Tags: `transaction`, `spring`, `spring-batch`. No threshold.

### Why is this an issue?

A Spring transaction or a Spring Batch chunk was active on the thread, but the write ran on a connection in
auto-commit mode. The database committed it at once. When the transaction rolls back, this write stays, and a chunk
that is retried writes it again. The usual cause is a transaction manager that manages another `DataSource` than the
one the code writes to, or a step configured with `ResourcelessTransactionManager`.

### How to fix it

Use the transaction manager of the `DataSource` that the code writes to, and let every writer get its connection
through Spring (`DataSourceUtils`, `JdbcTemplate`, JPA) so it joins the transaction.

#### Spring Batch

Noncompliant: the step's transaction manager does not manage the writer's database.

```java
@Bean
Step importStep(JobRepository jobRepository, JdbcBatchItemWriter<Order> writer) {
    return new StepBuilder("import", jobRepository)
        .<Order, Order>chunk(100, new ResourcelessTransactionManager())
        .reader(reader()).writer(writer)
        .build();
}
```

Compliant: the chunk transaction is the writer's database transaction.

```java
.<Order, Order>chunk(100, new DataSourceTransactionManager(ordersDataSource))
```

#### Spring

Noncompliant: a connection taken from the `DataSource` directly, outside Spring's transaction.

```java
@Transactional
public void archive(long id) {
    try (Connection c = dataSource.getConnection(); PreparedStatement ps = c.prepareStatement(SQL)) {
        ps.setLong(1, id);
        ps.executeUpdate(); // auto-commit: committed whatever happens next
    }
}
```

Compliant: `JdbcTemplate` uses the transaction's connection.

```java
@Transactional
public void archive(long id) {
    jdbcTemplate.update(SQL, id);
}
```

### How SQL Blame detects it

SQL Blame marks an INSERT, UPDATE, DELETE or MERGE when all of these hold:

- the connection was in auto-commit mode when the statement ran
- a scope with a transaction was the innermost scope on the thread: Spring's, a Spring Batch chunk, or a JTA
  transaction (standalone, begun by a CDI `@Transactional` interceptor, or through Spring's `JtaTransactionManager`)
- the statement did not run in an after-commit or after-completion callback (rule SB020 covers those); a JTA
  synchronization's `beforeCompletion` still runs inside the transaction

Not marked: statements of Spring Batch's job repository on its metadata tables, writes inside a scope without a
transaction (`NOT_SUPPORTED`, `SUPPORTS` without an outer transaction), and SELECTs.

Limits: needs the transaction history and the auto-commit mode reported by the driver. Some XA drivers report
auto-commit also while their connection takes part in a JTA transaction (PostgreSQL's), so a statement on an XA
connection in auto-commit counts as inside the transaction there: a forgotten enlistment of an XA connection is not
found. In runs recorded by an earlier version JTA transactions are not judged.

### Resources

- [Spring Batch: Configuring a step](https://docs.spring.io/spring-batch/reference/step/chunk-oriented-processing/configuring.html)
- [Spring Framework: Synchronizing resources with transactions](https://docs.spring.io/spring-framework/reference/data-access/transaction/tx-resource-synchronization.html)

<!-- rule SB013 -->
## SB013 @Transactional did not apply

Severity: Major. Type: Reliability. Tags: `transaction`, `spring`, `proxy`. No threshold.

### Why is this an issue?

Spring applies `@Transactional` through a proxy around the bean. A call that does not go through the proxy runs the
method without the transaction it asks for: a call on `this` from another method of the same class, a private,
final or static method, or a bean Spring did not proxy. The method then runs without a transaction, or silently
joins the caller's transaction with the caller's settings, so `REQUIRES_NEW`, `readOnly` or `rollbackFor` on it have
no effect.

### How to fix it

Call the method through the proxy: move it to another bean, or inject the bean into itself. Make the method public
and not final.

#### Spring

Noncompliant: self-invocation; the audit entry joins the outer transaction and rolls back with it.

```java
@Service
public class OrderService {
    @Transactional
    public void place(Order order) {
        orders.save(order);
        audit(order); // a call on this: no proxy, no new transaction
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void audit(Order order) { auditLog.save(new AuditEntry(order)); }
}
```

Compliant: the method lives in another bean and is called through its proxy.

```java
@Service
public class AuditService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void audit(Order order) { auditLog.save(new AuditEntry(order)); }
}
```

Noncompliant: a private method; Spring's proxy cannot intercept it.

```java
@Transactional
private void saveAll(List<Order> orders) { orders.forEach(this.orders::save); }
```

Compliant: a public method of a Spring bean, called from another bean.

```java
@Transactional
public void saveAll(List<Order> orders) { orders.forEach(this.orders::save); }
```

### How SQL Blame detects it

At run time the agent checks every method of the application's packages annotated `@Transactional` (Spring's,
`jakarta.transaction` or `javax.transaction`, on the method or its class) when it starts. If neither Spring's
transaction interceptor nor a CDI transaction interceptor announced that call, the proxy was bypassed. SQL Blame
marks the statements that ran inside the method during that call (the method is on their call stack and no later
call of it went through the proxy) and lists each bypass in the Transactions tab, also when the method ran no
statement.

In the editor, an inspection reports a call on `this` of a `@Transactional` method whose settings differ from the
caller's, in Java and Kotlin, and says when the called method is final, so that no call reaches it through a
class-based proxy (in Kotlin: not `open` and not opened by the kotlin-spring plugin). It is shown only where JetBrains'
own Spring inspection for this is not enabled.

Not marked: methods called through reflection (test runners, frameworks), and methods outside the application's
packages. The agent reports at most 20 bypasses per method.

Limits: needs the transaction history and Spring's transaction interceptor or a CDI one. A method deeper than the 32
application frames kept per statement is not found on its stack.

### Resources

- [Spring Framework: Using @Transactional (proxy mode)](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html)
- [Spring Framework: Understanding AOP proxies](https://docs.spring.io/spring-framework/reference/core/aop/proxying.html)

<!-- rule SB014 -->
## SB014 Long transaction

Severity: Major. Type: Performance. Tags: `transaction`, `locking`. Threshold: 5 s (default, set in Settings | Tools | SQL Blame | Rules).

### Why is this an issue?

The transaction stayed open for a long time. It held a connection from the pool all that time, and every row it
wrote stayed locked until the end, so other transactions that touch those rows wait. A write early in a long
transaction is the worst case: the lock is taken at once and held through all the processing that follows.

### How to fix it

Keep transactions short: read and compute outside the transaction, then write in one short transaction. Split long
batches into chunks with their own commits, and never wait for a remote call or a user inside a transaction.

#### Spring

Noncompliant: a remote call inside the transaction, after the first write.

```java
@Transactional
public void confirm(long orderId) {
    Order order = orders.findById(orderId).orElseThrow();
    order.setStatus(Status.CONFIRMED);   // flushed later, but the row is locked once it is written
    paymentClient.capture(order);        // seconds over the network
    mailer.sendConfirmation(order);
}
```

Compliant: the slow work runs outside, the transaction only writes.

```java
public void confirm(long orderId) {
    Order order = orderReader.load(orderId);
    paymentClient.capture(order);
    orderWriter.markConfirmed(orderId); // @Transactional, short
    mailer.sendConfirmation(order);
}
```

#### Spring Batch

Noncompliant: one chunk for the whole input.

```java
.<Row, Row>chunk(1_000_000, transactionManager)
```

Compliant: a commit every few hundred items.

```java
.<Row, Row>chunk(500, transactionManager)
```

### How SQL Blame detects it

SQL Blame marks every statement of a transaction that lasted at least 5 s, from the start of the scope that
began it to its commit or rollback; for a connection's own transaction (plain JDBC, MyBatis or jOOQ without Spring)
from its first statement to `commit()` or `rollback()`. When the transaction wrote, the issue says how long the rows
of its first write stayed locked.

Not marked: test transactions (the test's own transaction lasts as long as the test), scopes without a transaction
and scopes that joined another transaction (the outermost one is measured), and transactions that started before
SQL Blame was attached.

Limits: needs the transaction history. The duration includes everything the application did inside the
transaction, not only database time.

### Resources

- [PostgreSQL: Explicit locking](https://www.postgresql.org/docs/current/explicit-locking.html)
- [HikariCP: About pool sizing](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing)

<!-- rule SB015 -->
## SB015 Write in a read-only transaction

Severity: Minor. Type: Reliability. Tags: `transaction`, `spring`, `read-only`. No threshold.

### Why is this an issue?

The write ran in a transaction declared `readOnly = true`. Depending on the database and the driver the write fails
(PostgreSQL rejects it in a read-only transaction), is silently skipped (Hibernate does not flush a read-only
session), or goes through while a replica router sends it to a read-only database. Either the declaration is wrong or
the write is in the wrong place.

### How to fix it

Move the write to a method with a read-write transaction, or remove `readOnly` from a method that writes.

#### Spring

Noncompliant: a read-only method that also records the visit.

```java
@Transactional(readOnly = true)
public Product show(long id) {
    Product product = products.findById(id).orElseThrow();
    visits.save(new Visit(product)); // a write in a read-only transaction
    return product;
}
```

Compliant: the write in its own read-write transaction.

```java
@Transactional(readOnly = true)
public Product show(long id) { return products.findById(id).orElseThrow(); }

@Transactional
public void recordVisit(long id) { visits.save(new Visit(products.getReferenceById(id))); }
```

#### JdbcTemplate

Noncompliant: an update in a read-only `TransactionTemplate`.

```java
readOnlyTemplate.executeWithoutResult(status -> jdbc.update("update product set views = views + 1 where id = ?", id));
```

Compliant: a template without `readOnly` for writes.

```java
writeTemplate.executeWithoutResult(status -> jdbc.update("update product set views = views + 1 where id = ?", id));
```

### How SQL Blame detects it

SQL Blame marks an INSERT, UPDATE, DELETE or MERGE that ran in a transaction one of whose scopes (from the one that
began it down to the statement's own) is declared read-only. The issue names that scope.

Not marked: writes in auto-commit (rule SB012), statements of Spring Batch's job repository, and SELECTs.

Limits: needs the transaction history. A failed write is marked too: the failure is often the read-only flag at work.
JTA and CDI `@Transactional` declare no read-only flag, so their transactions are never read-only here.

### Resources

- [Spring Framework: @Transactional settings](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html#transaction-declarative-attransactional-settings)
- [PostgreSQL: SET TRANSACTION](https://www.postgresql.org/docs/current/sql-set-transaction.html)

<!-- rule SB016 -->
## SB016 REQUIRES_NEW in a loop

Severity: Minor. Type: Performance. Tags: `transaction`, `spring`, `connection-pool`. Threshold: 10 transactions (default, set in Settings | Tools | SQL Blame | Rules).

### Why is this an issue?

`REQUIRES_NEW` suspends the outer transaction and starts a new one on a second connection. In a loop inside a
transaction, every iteration takes a connection while the outer one stays checked out, commits on its own, and
cannot see what the outer transaction has written but not yet committed. Under load the pool runs out of
connections, and an inner transaction that touches a row the outer one wrote waits on that row's lock.

### How to fix it

If the items can commit together, do the inner work in one new transaction for the whole loop. If each item needs
its own commit, move the loop out of the outer transaction, so only one connection is checked out at a time.

#### Spring

Noncompliant: a new transaction per item inside an outer transaction.

```java
@Transactional
public void importAll(List<Item> items) {
    for (Item item : items) {
        logService.record(item); // @Transactional(propagation = REQUIRES_NEW)
    }
}
```

Compliant: one inner transaction for all entries.

```java
@Transactional
public void importAll(List<Item> items) {
    logService.recordAll(items); // REQUIRES_NEW once
}
```

Compliant as well: no outer transaction, one commit per item.

```java
public void importAll(List<Item> items) {
    items.forEach(itemService::importOne); // @Transactional each
}
```

### How SQL Blame detects it

SQL Blame counts the scopes that began a new transaction with propagation `REQUIRES_NEW` while an outer transaction
was suspended, per outer transaction and per place: the `@Transactional` method, or the line of the
`TransactionTemplate` or manager call. From 10 transactions of them, the statements of all those inner transactions are
marked, also the ones that come later.

Not marked: `REQUIRES_NEW` without an outer transaction, fewer inner transactions than the threshold, and inner
transactions from different places.

Limits: needs the transaction history. Inner transactions that ran no statement are counted but have nothing to mark.

### Resources

- [Spring Framework: Transaction propagation, REQUIRES_NEW](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/tx-propagation.html#tx-propagation-requires_new)
- [HikariCP: About pool sizing](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing)

<!-- rule SB017 -->
## SB017 Committed despite an exception

Severity: Major. Type: Data safety. Tags: `transaction`, `spring`, `exception`. No threshold.

### Why is this an issue?

An exception left a `@Transactional` method, yet the transaction committed. By default Spring rolls back only for
unchecked exceptions (`RuntimeException` and `Error`); a checked exception, or one listed in `noRollbackFor`, lets
the commit go ahead. The caller sees a failure while the half-done work stays in the database.

### How to fix it

Say which exceptions roll back. Add the checked exception to `rollbackFor`, or, from Spring Framework 6.2, roll back
for every exception with `@EnableTransactionManagement(rollbackOn = RollbackOn.ALL_EXCEPTIONS)`.

#### Spring

Noncompliant: a checked exception after the first write; the order is committed without its lines.

```java
@Transactional
public void place(Order order) throws StockException {
    orders.save(order);
    stock.reserve(order.lines()); // throws StockException (checked)
}
```

Compliant: the rule names the exception.

```java
@Transactional(rollbackFor = StockException.class)
public void place(Order order) throws StockException { ... }
```

Noncompliant: `noRollbackFor` on an exception that leaves half the work.

```java
@Transactional(noRollbackFor = PaymentDeclinedException.class)
public void checkout(Cart cart) { ... }
```

Compliant: catch the exception where the remaining work is consistent, and roll back otherwise.

```java
@Transactional
public void checkout(Cart cart) {
    try { payments.charge(cart); }
    catch (PaymentDeclinedException declined) { cart.markDeclined(); return; }
    orders.create(cart);
}
```

### How SQL Blame detects it

SQL Blame marks every statement of a transaction when an exception left the transactional method that began it,
the method's rollback rules decided not to roll back for it (the default for a checked exception, or a
`noRollbackFor` rule; with CDI `@Transactional` a `dontRollbackOn` rule), and the transaction then committed. The issue names the exception, the method and the rule
that decided.

Not marked: exceptions caught inside the method, including a participant's exception that the outer method caught
(the outer method then ended normally), exceptions that rolled the transaction back, and transactions that rolled
back for another reason.

Limits: needs the transaction history and Spring's transaction interceptor (or the AspectJ aspect) or a CDI
`@Transactional` interceptor (`jakarta.transaction` or `javax.transaction`).

### Resources

- [Spring Framework: Rolling back a declarative transaction](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/rolling-back.html)
- [Spring Framework API: RuleBasedTransactionAttribute](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/interceptor/RuleBasedTransactionAttribute.html)

<!-- rule SB018 -->
## SB018 Lazy loading after the session closed

Severity: Major. Type: Reliability. Tags: `transaction`, `jpa`, `lazy-loading`. No threshold.

### Why is this an issue?

The query loaded an entity whose association is lazy. The transaction ended, the persistence context closed, and
later code touched the association: Hibernate threw `LazyInitializationException` ("could not initialize proxy - no
Session"). The data was never loaded, and the request or job failed far from the place that caused it.

### How to fix it

Load what the caller needs while the transaction is open: fetch the association with the query, keep the access
inside the transaction, or return a DTO with exactly the needed data. The transaction's tab, opened with Transaction in the
query detail, shows these options with a snippet built from the run.

#### JPA / Hibernate

Noncompliant: the items are read after the service method returned.

```java
Order order = orderService.find(id); // @Transactional, ends here
order.getItems().size();             // LazyInitializationException
```

Compliant: fetch the association in the query.

```java
@Query("select o from Order o join fetch o.items where o.id = :id")
Optional<Order> findWithItems(long id);
```

#### Spring Data JPA

Compliant: an entity graph on the repository method.

```java
@EntityGraph(attributePaths = "items")
Optional<Order> findById(Long id);
```

#### Spring

Compliant: the access inside the transaction, or a DTO built there.

```java
@Transactional(readOnly = true)
public OrderView view(long id) {
    Order order = orders.findById(id).orElseThrow();
    return new OrderView(order.getId(), order.getItems().size());
}
```

### How SQL Blame detects it

The agent records every `LazyInitializationException` when it is created: the entity or collection role and the line
of the access. SQL Blame takes the last transaction that ended on the same thread before it and marks the SELECT of
that transaction that read the owner entity's table: the tables come from Hibernate's own mapping, which the agent
reads once per session factory, and of several such SELECTs the one with the owner's id among its parameters wins,
else the last one. Every exception is also listed in the Transactions tab, with or without a marked query.

Not marked: a query when no SELECT of that transaction reads the table; the exception is still listed. When the
agent has no mapping yet (after an attach, until Hibernate runs its first insert, update or delete), the table name
is guessed from the class name in snake case, singular or plural.

Limits: needs the transaction history and Hibernate 5.6 to 7. The agent records at most 1 000 of these exceptions per
JVM.

### Resources

- [Hibernate ORM User Guide: Fetching](https://docs.hibernate.org/orm/6.6/userguide/html_single/Hibernate_User_Guide.html#fetching)
- [Spring Data JPA: Configuring fetch and load graphs](https://docs.spring.io/spring-data/jpa/reference/jpa/query-methods.html#jpa.entity-graph)

<!-- rule SB019 -->
## SB019 Query outside the transaction (Open Session in View)

Severity: Minor. Type: Performance. Tags: `transaction`, `jpa`, `open-in-view`. No threshold.

### Why is this an issue?

Spring Boot enables Open Session in View (`spring.jpa.open-in-view=true`) by default and logs a warning about it. The
persistence context stays open for the whole web request, so lazy associations still load after the service
transaction ended: in the controller, the view or the JSON serializer. Each of these loads is a query outside any
transaction, often one per row (N+1), and it holds a connection until the response is written.

### How to fix it

Turn Open Session in View off and load what the response needs inside the service transaction: fetch joins, entity
graphs or DTO projections.

#### Spring Boot

Noncompliant: the default.

```properties
# spring.jpa.open-in-view not set: true
```

Compliant: off, so a missing fetch fails in development instead of querying silently.

```properties
spring.jpa.open-in-view=false
```

#### JPA / Hibernate

Noncompliant: the controller serializes a lazy collection after the service returned.

```java
@GetMapping("/orders/{id}")
Order order(@PathVariable long id) {
    return orderService.find(id); // items load during JSON serialization
}
```

Compliant: the service returns what the response needs.

```java
@GetMapping("/orders/{id}")
OrderView order(@PathVariable long id) {
    return orderService.view(id); // DTO built inside the transaction
}
```

### How SQL Blame detects it

SQL Blame marks a SELECT when all of these hold:

- it ran inside a web request kept open by Open Session in View (Spring's `OpenEntityManagerInViewInterceptor` or
  `OpenEntityManagerInViewFilter`, or the `OpenSessionInView` interceptor and filter of `orm.hibernate5`)
- no transaction was active on the thread
- a transaction that started in the same request had already ended

The issue names the request and the transaction that ended before.

Not marked: queries of the request before its first transaction (a security filter loading the user, for example),
queries in requests without Open Session in View, and writes.

Limits: needs the transaction history. Only the transactions started directly in the request are compared (the
last 100).

### Resources

- [Spring Boot: Open EntityManager in View](https://docs.spring.io/spring-boot/reference/data/sql.html#data.sql.jpa-and-spring-data.open-entity-manager-in-view)
- [Spring Framework API: OpenEntityManagerInViewInterceptor](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/orm/jpa/support/OpenEntityManagerInViewInterceptor.html)

<!-- rule SB020 -->
## SB020 Write after commit

Severity: Major. Type: Data safety. Tags: `transaction`, `spring`, `events`. No threshold.

### Why is this an issue?

`afterCommit` and `afterCompletion` callbacks, such as a `@TransactionalEventListener` (default phase
`AFTER_COMMIT`), run after the transaction has committed or rolled back, while its connection is still bound to the
thread. A write there joins a transaction that is already over: nothing commits it as part of that transaction. With
JPA the change is never flushed; with plain JDBC it depends on how the connection is reset afterwards. Either way the
write is not where the code expects it.

### How to fix it

Give the write a transaction of its own: `REQUIRES_NEW` on the method the listener calls (or on the listener
itself), or write in `BEFORE_COMMIT` when the write belongs to the original transaction.

#### Spring

Noncompliant: the listener saves after the commit without a transaction of its own.

```java
@TransactionalEventListener
public void on(OrderPlaced event) {
    notifications.save(new Notification(event.orderId())); // not committed
}
```

Compliant: a new transaction for the write.

```java
@TransactionalEventListener
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void on(OrderPlaced event) {
    notifications.save(new Notification(event.orderId()));
}
```

Compliant as well: the write is part of the original transaction.

```java
@TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)
public void on(OrderPlaced event) { ... }
```

### How SQL Blame detects it

SQL Blame marks an INSERT, UPDATE, DELETE or MERGE that ran in a Spring `afterCommit` or `afterCompletion` callback
or in the `afterCompletion` of a JTA synchronization (`registerSynchronization`, also interposed ones), and still
ran in the finished transaction. The agent notes in which callback a scope began, so a statement in a transaction the
callback began itself (`REQUIRES_NEW`) or after it suspended the finished one (`NOT_SUPPORTED`) is told apart from a
statement in the callback of a `REQUIRES_NEW` transaction that began earlier and has just ended. The issue names the
phase and the transaction whose callback ran it, and says what became of the write. With auto-commit off, nothing
commits it. With auto-commit on, it commits on its own outside any transaction, and nothing rolls it back if later
work fails. Spring Framework 7 with Hibernate 7 gives the second case: the connection is already back in auto-commit
when the callbacks run.

Not marked: writes in a transaction the callback began itself or after it suspended the finished one, writes in a JTA
synchronization's `beforeCompletion` (still inside the transaction), and writes on a connection whose auto-commit
mode is not known.

Limits: needs the transaction history.

### Resources

- [Spring Framework: Transaction-bound events](https://docs.spring.io/spring-framework/reference/data-access/transaction/event.html)
- [Spring Framework API: TransactionSynchronization#afterCommit](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/support/TransactionSynchronization.html#afterCommit())

<!-- rule SB021 -->
## SB021 Unique violation from the flush order

Severity: Major. Type: Reliability. Tags: `transaction`, `jpa`, `flush`. No threshold.

### Why is this an issue?

Hibernate does not run the statements of a flush in the order the code called `persist` and `remove`. It runs
inserts first, then updates and collection changes, and deletes last. Code that deletes a row and inserts a new one
with the same unique key in the same transaction therefore fails with a unique violation: the insert reaches the
database while the old row is still there.

### How to fix it

Flush the delete before the insert, or update the existing row instead of replacing it.

#### JPA / Hibernate

Noncompliant: replace a row with the same unique code in one flush.

```java
em.remove(oldTag);                 // code = 'sale'
em.persist(new Tag("sale"));       // INSERT runs before the DELETE: unique violation
```

Compliant: send the delete first.

```java
em.remove(oldTag);
em.flush();                        // DELETE now
em.persist(new Tag("sale"));
```

Compliant as well: change the existing entity.

```java
oldTag.setLabel(newLabel);         // one UPDATE, no conflict
```

#### Spring Data JPA

Noncompliant: delete and save in one transaction.

```java
tags.delete(oldTag);
tags.save(new Tag("sale"));
```

Compliant: flush after the delete.

```java
tags.delete(oldTag);
tags.flush();
tags.save(new Tag("sale"));
```

### How SQL Blame detects it

SQL Blame marks a statement when all of these hold:

- it failed with a unique or primary key violation (SQLState 23505, MySQL / MariaDB 1062, SQL Server 2627 / 2601,
  Oracle ORA-00001, SQLite UNIQUE constraint failed)
- it ran an entity insert or update of a Hibernate flush
- the plan of that flush, read from Hibernate's action queue when the flush started, still had deletes on the same
  table (by table name, or by entity when Hibernate did not tell the tables)

This is no guess from the SQL order: the flush plan shows the waiting delete.

Not marked: other constraint violations, and unique violations without a waiting delete on that table (a real
duplicate).

Limits: needs the transaction history and Hibernate 5.6 to 7.

### Resources

- [Hibernate ORM User Guide: Flush operation order](https://docs.hibernate.org/orm/6.6/userguide/html_single/Hibernate_User_Guide.html#flushing-order)
- [PostgreSQL: Error codes (appendix A)](https://www.postgresql.org/docs/current/errcodes-appendix.html)

<!-- rule SB022 -->
## SB022 Write in a task started inside a transaction

Severity: Major. Type: Data safety. Tags: `transaction`, `spring`, `async`. No threshold.

### Why is this an issue?

An `@Async` method or a task given to a Spring executor runs on another thread. Transactions are bound to a thread,
so the task does not take part in the transaction it was started in: its writes commit on their own (or in a
transaction of their own), stay when the starting transaction rolls back, and cannot see that transaction's data
until it commits. A task that starts before the commit may also read rows that are about to change.

### How to fix it

Start the task after the commit, and give it the data it needs instead of letting it read uncommitted rows.

#### Spring

Noncompliant: the task is handed over inside the transaction.

```java
@Transactional
public void place(Order order) {
    orders.save(order);
    mailer.sendConfirmationAsync(order.getId()); // @Async: writes outside this transaction
}
```

Compliant: publish an event and handle it after the commit.

```java
@Transactional
public void place(Order order) {
    orders.save(order);
    events.publishEvent(new OrderPlaced(order.getId()));
}

@Async
@TransactionalEventListener
public void on(OrderPlaced placed) { mailer.sendConfirmation(placed.orderId()); }
```

#### Task executor

Noncompliant: a task submitted inside a transaction writes on its own.

```java
@Transactional
public void close(long accountId) {
    accounts.markClosed(accountId);
    executor.execute(() -> statements.archive(accountId));
}
```

Compliant: the task runs once the transaction has committed.

```java
TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronization() {
    @Override public void afterCommit() { executor.execute(() -> statements.archive(accountId)); }
});
```

### How SQL Blame detects it

SQL Blame marks an INSERT, UPDATE, DELETE or MERGE that ran in a task handed over from another thread (an `@Async`
method, or a task given to a Spring task executor or scheduler) when the scope that handed it over was in a
transaction that had not ended when the task started. The issue names the starting thread and its transaction.

Not marked: tasks started outside a transaction or after it ended, writes in a transaction the task began itself
(an `@Async` method that is also `@Transactional`: a separate unit of work by design), statements of Spring Batch's
job repository, and reads.

Limits: needs the transaction history. Threads started without a Spring executor (`new Thread`,
`CompletableFuture` on the common pool) are not linked to the starting transaction and stay unmarked. So are tasks
given directly to a Spring executor that runs them without Spring code on the worker thread:
`SimpleAsyncTaskExecutor` without a concurrency limit, `ConcurrentTaskExecutor` over a plain JDK executor and
`ThreadPoolTaskExecutor` before Spring 6.1. `@Async` methods are linked on every executor.

### Resources

- [Spring Framework: The @Async annotation](https://docs.spring.io/spring-framework/reference/integration/scheduling.html#scheduling-annotation-support-async)
- [Spring Framework: Transaction-bound events](https://docs.spring.io/spring-framework/reference/data-access/transaction/event.html)
