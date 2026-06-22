# Database & JPA — Interview Questions

> Phase added: Phase 0
> Real project reference: H2 in-memory locally, PostgreSQL planned for AWS deployment

---

## Q1: What is JPA and how does it relate to Hibernate?

JPA (Java Persistence API) is a specification — a set of interfaces and annotations that define 
how Java objects should be mapped to relational database tables. It defines things like 
`@Entity`, `@Table`, `@Column`, `@OneToMany`, `EntityManager`.

Hibernate is the most popular implementation of the JPA specification. It provides the actual 
SQL generation, caching, and database interaction logic.

Spring Data JPA is a layer on top that makes JPA even easier — by extending `JpaRepository` 
you get CRUD operations for free.

**Hierarchy:** JPA (spec) → Hibernate (impl) → Spring Data JPA (abstraction)

---

## Q2: What is the difference between H2 and PostgreSQL?

| Feature | H2 | PostgreSQL |
|---|---|---|
| Type | In-memory / embedded | Standalone server |
| Installation | None — runs inside JVM | Requires installation |
| Data persistence | Lost on app stop | Persisted to disk |
| Use case | Local development, testing | Production |
| Performance | Fast for small data | Optimized for large scale |

In PulseBoard: H2 locally (no Docker, no installation needed), PostgreSQL on AWS RDS in 
production. The switch is just a config change — same JPA code works with both.

---

## Q3: What is `ddl-auto` in Spring JPA and what are the options?

`spring.jpa.hibernate.ddl-auto` controls how Hibernate manages the database schema:

| Value | Behavior | Use when |
|---|---|---|
| `create` | Drops and recreates schema on startup | Fresh start every time |
| `create-drop` | Creates on startup, drops on shutdown | Testing |
| `update` | Updates schema to match entities | Development |
| `validate` | Validates schema matches entities, fails if not | Production (with migrations) |
| `none` | Does nothing | Production (use Flyway/Liquibase) |

In PulseBoard locally I use `create-drop` — schema is fresh on every startup, matching the 
H2 in-memory approach. In production I'll use `validate` with Flyway migrations.

---

## Q4: What is the N+1 query problem and how do you solve it?

N+1 happens when loading a parent entity and then executing a separate query for each child.

```java
// Loads all teams (1 query)
List<Team> teams = teamRepository.findAll();
// Then for each team, loads members separately (N queries)
teams.forEach(team -> team.getMembers().size());
Result: 1 + N database queries instead of 1.

Solutions:

@EntityGraph — fetch associations eagerly in one query
JOIN FETCH in JPQL — SELECT t FROM Team t JOIN FETCH t.members
@BatchSize — loads associations in batches
In PulseBoard when building the team dashboard, loading team members alongside team data uses
JOIN FETCH to avoid N+1 on the manager dashboard query.

Q5: What is the difference between @OneToMany, @ManyToOne, @ManyToMany?
These define relationships between entities:

@ManyToOne — many records point to one record. In PulseBoard: many WeeklyUpdates belong
to one User. The foreign key is on the WeeklyUpdate table.
@OneToMany — one record has many records. In PulseBoard: one Team has many Members.
Usually mapped with mappedBy on the non-owning side.
@ManyToMany — many records relate to many records. In PulseBoard: a User can be a member
of many Teams and a Team has many Users. Requires a join table.
@OneToOne — one record relates to exactly one other record.
Q6: What is Lazy vs Eager loading in JPA?
Lazy loading — associated entities are loaded only when accessed. Default for
@OneToMany and @ManyToMany. Efficient but can cause LazyInitializationException if
accessed outside a transaction.
Eager loading — associated entities are loaded immediately with the parent. Default for
@ManyToOne and @OneToOne. Can cause performance issues if associations are large.
Best practice: Use lazy loading by default, use JOIN FETCH or @EntityGraph when you
know you need the association.

Q7: What is a database transaction and how does Spring manage it?
A transaction is a unit of work that is either fully committed or fully rolled back — ACID
properties (Atomicity, Consistency, Isolation, Durability).

Spring manages transactions with @Transactional. When you annotate a service method with
@Transactional, Spring wraps it in a transaction — if the method completes successfully,
changes are committed; if an exception is thrown, changes are rolled back.

@Transactional
public void submitWeeklyUpdate(UpdateRequest request) {
    weeklyUpdateRepository.save(update);      // both save
    targetRepository.saveAll(targets);         // or both rollback
}
In PulseBoard, the weekly update submission is transactional — saving the update and its
targets happens atomically.


