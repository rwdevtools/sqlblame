# SQL Blame rules

SQL Blame checks every captured query against the rules below and marks the queries that break one (the Flags
column, "Only flagged", the issues of a query's detail). The rules run in the IDE on what the agent captured; nothing
is sent anywhere and the application is not slowed down by them.

Turn rules on or off and change their thresholds in **Settings | Tools | SQL Blame | Rules**. A rule that cannot be
sure stays silent: SQL Blame prefers a missed issue over a false alarm.

**Scope.** Several rules count executions "in one transaction". With the current agent the transaction is exact: the
agent sees `commit`, `rollback` and the auto-commit mode of every connection, so two transactions are never mixed up,
also when the same method runs twice in a row. Statements in auto-commit mode are each their own transaction. For the
counting rules they form a *burst*: consecutive statements on one thread with at most one second between them. A
statement of a transaction on that thread ends the burst.
Queries from older agents fall back to an approximation (same thread, same Spring transaction, same connection).

Each rule below has the same parts as its page in the IDE (**SQL Blame Rules** in the tool window's gear menu, or a
click on a Flags icon): why it is an issue, how to fix it per stack, how SQL Blame detects it, and resources.

## Related SonarQube rules

SonarQube checks source code; SQL Blame checks the statements that actually ran. Where a SonarQube rule covers the
same problem, the table names it and says what SQL Blame does differently. Rule numbers refer to
[rules.sonarsource.com](https://rules.sonarsource.com).

| Rule | Related SonarQube rules | What SQL Blame does differently |
|---|---|---|
| SB002 Unbatched writes | [S6912](https://rules.sonarsource.com/java/RSPEC-6912/) (Java: batch processing in JDBC, a `Statement` executed in a loop) | Also catches `PreparedStatement` loops and the writes an ORM sends one by one; needs the threshold of executions with different values. |
| SB005 Large result set | [S6905](https://rules.sonarsource.com/java/RSPEC-6905/) (Java), [S1499](https://rules.sonarsource.com/plsql/RSPEC-1499/) (PL/SQL): `SELECT *` | Counts the rows read, not the columns selected; a `SELECT *` alone is not marked. |
| SB006 UPDATE / DELETE without WHERE | S1590 ([PL/SQL](https://rules.sonarsource.com/plsql/RSPEC-1590/), [T-SQL](https://rules.sonarsource.com/tsql/RSPEC-1590/)) | Checks the executed SQL, including SQL built at run time and framework bulk statements; exempts SQL Server temporary tables as the T-SQL rule does. |

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

<!-- rule SB001 -->
## SB001 N+1 select

Severity: Major. Type: Performance. Tags: `n+1`, `jpa`, `mybatis`, `jooq`. Threshold: 5 executions (default; Settings | Tools | SQL Blame | Rules).

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

Severity: Minor. Type: Performance. Tags: `batch`, `jdbc`, `jpa`, `spring-batch`. Threshold: 10 executions (default; Settings | Tools | SQL Blame | Rules).

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
- Related rules: SonarQube [S6912](https://rules.sonarsource.com/java/RSPEC-6912/) (batch processing in JDBC), which
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

Severity: Major. Type: Performance. Tags: `index`, `sql`. Threshold: 100 ms (default; Settings | Tools | SQL Blame | Rules).

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

Severity: Minor. Type: Performance. Tags: `paging`, `memory`, `sql`. Threshold: 1,000 rows (default; Settings | Tools | SQL Blame | Rules).

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
- Related rules: SonarQube [S6905](https://rules.sonarsource.com/java/RSPEC-6905/) (SQL queries should retrieve only
  the necessary fields) and [S1499](https://rules.sonarsource.com/plsql/RSPEC-1499/) for PL/SQL (`SELECT *`) look at the columns; SB005 looks at the rows.

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

An UPDATE or DELETE (also after `WITH ...`) has no `WHERE` at the level of the statement itself; a WHERE inside a
subquery does not count. Every statement of a `Statement` batch is checked.

Source checks see the SQL written in the code. SQL Blame checks the SQL that ran, so it also sees statements built at
run time and the SQL that frameworks generate: Spring Data's `deleteAllInBatch` runs `delete from <table>` and is
marked too.

Limits: `TRUNCATE` is not marked, nor `WHERE CURRENT OF`, nor an UPDATE or DELETE with a `JOIN` at the top level
(MySQL and SQL Server syntax, where the join restricts the rows), nor a SQL Server temporary table (`#name`). Only the
first statement of a text with several statements is checked.

### Resources

- [PostgreSQL: DELETE](https://www.postgresql.org/docs/current/sql-delete.html)
- [MySQL: TRUNCATE TABLE](https://dev.mysql.com/doc/refman/8.4/en/truncate-table.html)
- Related rules: SonarQube S1590, UPDATE and DELETE statements should contain WHERE clauses
  ([PL/SQL](https://rules.sonarsource.com/plsql/RSPEC-1590/), [T-SQL](https://rules.sonarsource.com/tsql/RSPEC-1590/)).
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

Severity: Minor. Type: Performance. Tags: `sql`, `jpa`, `statement-cache`. Threshold: 100 elements (default; Settings | Tools | SQL Blame | Rules).

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

Compliant: a subquery or join that selects the rows in the database; for lists that must stay, padding keeps the
statement cache small.

```java
// application.properties
// spring.jpa.properties.hibernate.query.in_clause_parameter_padding=true
```

### How SQL Blame detects it

An `IN (...)` list of values in the executed SQL has at least 100 elements. `IN (subquery)` does not count.

In the source the list is one parameter (`in (:ids)`). SQL Blame sees the executed SQL after the framework expanded
it, so it knows how long the list was.

Limits: only value lists in the SQL text count; an array parameter is one value.

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
missing check before the write or a race between two writers; class 42 means a typo or a schema that does not match
the code; 57014 is a statement timeout or cancel; class 08 is a lost connection.

#### SQL (PostgreSQL)

Noncompliant: check, then insert; a concurrent writer can insert the same key between the two statements (SQLState
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

Noncompliant: an exception swallowed inside the transaction; the transaction is already broken.

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
feature not supported (0A); otherwise the exception's class name.

Whether a statement fails is decided at run time. SQL Blame records every failed execution, also those the
application catches and hides.

### Resources

- [PostgreSQL: Error codes (appendix A)](https://www.postgresql.org/docs/current/errcodes-appendix.html)
- [PostgreSQL: INSERT, ON CONFLICT](https://www.postgresql.org/docs/current/sql-insert.html)
- [Spring Framework: Rolling back a declarative transaction](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/rolling-back.html)
