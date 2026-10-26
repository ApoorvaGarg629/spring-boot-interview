# 01. JPA, Hibernate, Spring Data JPA & Entities

## 1. What Is JPA?

**JPA (Java Persistence API)** is a Java specification that defines how Java objects are mapped to relational database tables and how persistent data is managed.

In modern Java applications, JPA is commonly used to perform database operations using Java objects instead of writing SQL for every operation.

JPA defines:
- Entity mapping annotations.
- Entity lifecycle and persistence context concepts.
- Interfaces such as `EntityManager`.
- JPQL (Java Persistence Query Language).
- Relationships between entities.
- Persistence and transaction-related behavior.

**Important:** JPA is a specification, not an implementation.

### Why Do We Need JPA?

Without an ORM framework, developers often need to:
- Write SQL queries for database operations.
- Map query results to Java objects manually.
- Handle object-to-table mapping.
- Manage relationships between objects and database records.
- Handle persistence-related boilerplate code.

JPA defines a standard way to manage these tasks through object-relational mapping.

### Example

Suppose we have an `Employee` table:

| id | name | salary |
|---|---|---|
| 1 | Rahul | 50000 |
| 2 | Priya | 60000 |

With JPA, a database row can be represented as a Java object:

~~~java
Employee employee = entityManager.find(Employee.class, 1L);
~~~

The JPA provider retrieves the corresponding record and maps its data to an `Employee` object.

---

## 2. What Is ORM?

**ORM (Object-Relational Mapping)** is a technique for mapping Java objects to relational database tables.

It connects two different models:

- Object-oriented model: classes, objects, fields, and relationships.
- Relational model: tables, rows, columns, and foreign keys.

### Example

Java class:

~~~java
public class Employee {
    private Long id;
    private String name;
    private double salary;
}
~~~

Corresponding database table:

~~~sql
CREATE TABLE employee (
    id BIGINT PRIMARY KEY,
    name VARCHAR(100),
    salary DECIMAL(10, 2)
);
~~~

An ORM framework maps the Java class and its fields to the database table and columns.

### Benefits of ORM

- Reduces repetitive JDBC and result-mapping code.
- Maps Java objects to database records.
- Supports relationships between entities.
- Supports transaction-aware persistence operations.
- Provides object-oriented querying through JPQL.
- Can generate SQL for supported database operations.

### Important Limitation

ORM does not eliminate the need to understand SQL.

Developers still need to understand:
- Joins.
- Indexes.
- Query execution plans.
- Transactions and isolation.
- Lazy loading.
- N+1 query problems.
- Database-specific behavior and performance.

**Interview one-liner:** ORM maps object-oriented classes and relationships to relational tables and relationships.

---

## 3. What Is Hibernate?

**Hibernate** is an ORM framework and one of the most widely used implementations of the JPA specification.

It provides an implementation of JPA interfaces and adds additional ORM features.

Hibernate handles tasks such as:
- Mapping entities to database tables.
- Generating SQL.
- Managing entity state.
- Dirty checking.
- Lazy loading.
- First-level caching.
- Relationship management.
- Schema generation.
- Query execution through JPQL/HQL and other supported query mechanisms.

### Example

~~~java
Employee employee = entityManager.find(Employee.class, 1L);
~~~

The application uses the JPA `EntityManager` API, while Hibernate can act as the underlying provider that executes the database operations.

### JPA vs Hibernate

| JPA | Hibernate |
|---|---|
| A specification/API for persistence. | An ORM framework and JPA implementation. |
| Defines standard interfaces and annotations. | Implements JPA and provides additional features. |
| Promotes provider portability. | Provides provider-specific extensions. |
| Examples: `EntityManager`, `@Entity`, `@Id`. | Examples: Hibernate-specific APIs, HQL extensions, and additional configuration options. |

**Interview trap:** JPA and Hibernate are not interchangeable terms. JPA defines the standard; Hibernate is one implementation.

---

## 4. What Is Spring Data JPA?

**Spring Data JPA** is a Spring Data module that simplifies persistence by providing repository abstractions on top of JPA.

It reduces boilerplate code for common database operations.

Instead of implementing CRUD methods manually, developers can define repository interfaces.

### Without Spring Data JPA

Using JPA directly, developers may need to write code such as:

~~~java
@Repository
public class EmployeeDao {

    @PersistenceContext
    private EntityManager entityManager;

    public Employee findById(Long id) {
        return entityManager.find(Employee.class, id);
    }

    public void save(Employee employee) {
        entityManager.persist(employee);
    }
}
~~~

This example shows basic persistence operations. A real application also needs appropriate transaction boundaries and additional handling for updates and deletes.

### With Spring Data JPA

~~~java
public interface EmployeeRepository
        extends JpaRepository<Employee, Long> {
}
~~~

Spring Data JPA provides implementations for inherited repository methods at runtime.

Examples:

~~~java
employeeRepository.findById(1L);
employeeRepository.findAll();
employeeRepository.save(employee);
employeeRepository.deleteById(1L);
~~~

### What Does Spring Data JPA Provide?

- Repository interfaces.
- CRUD methods.
- Pagination and sorting.
- Derived query methods.
- JPQL and native query support.
- Integration with Spring transactions.
- Integration with Spring's dependency injection.
- Auditing support when configured.
- Specifications and other query composition mechanisms.

**Important:** Spring Data JPA does not replace JPA or Hibernate. It builds on JPA and delegates persistence operations to a JPA provider, such as Hibernate.

---

## 5. JPA vs Hibernate vs Spring Data JPA

This is one of the most frequently asked Spring Boot interview questions.

| Feature | JPA | Hibernate | Spring Data JPA |
|---|---|---|---|
| Type | Specification/API | ORM framework and JPA provider | Spring Data abstraction |
| Main responsibility | Defines persistence standards | Implements ORM and JPA behavior | Simplifies repository development |
| Entity mapping | Defines standard annotations | Implements mapping behavior | Uses JPA entity mappings |
| CRUD implementation | Defines persistence APIs | Executes persistence operations | Generates repository implementations |
| Query support | JPQL and Criteria API specifications | Implements JPA queries and provides HQL/extensions | Derived queries, `@Query`, repository abstractions |
| Example | `EntityManager` | `Session` and Hibernate internals | `JpaRepository` |

### Remember the Relationship

~~~text
Application
    |
    v
Spring Data JPA Repository
    |
    v
JPA APIs
    |
    v
Hibernate (JPA Provider)
    |
    v
JDBC
    |
    v
Database
~~~

This is a simplified conceptual flow. Spring Data JPA delegates to JPA infrastructure, the provider uses database connectivity infrastructure, and actual execution can involve additional Spring and Hibernate components.

**Interview one-liner:** Spring Data JPA simplifies repository development, JPA defines the persistence contract, and Hibernate commonly implements that contract.

---

## 6. What Is an Entity?

An **entity** is a Java class whose instances represent persistent data and are mapped to database records.

An entity is generally annotated with `@Entity`.

### Example

~~~java
import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.Table;

@Entity
@Table(name = "employees")
public class Employee {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    private double salary;

    public Employee() {
    }

    public Employee(String name, double salary) {
        this.name = name;
        this.salary = salary;
    }

    public Long getId() {
        return id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public double getSalary() {
        return salary;
    }

    public void setSalary(double salary) {
        this.salary = salary;
    }
}
~~~

### Important Annotations

| Annotation | Purpose |
|---|---|
| `@Entity` | Marks the class as a JPA entity. |
| `@Table` | Configures the database table mapping. |
| `@Id` | Specifies the primary key. |
| `@GeneratedValue` | Configures primary-key generation. |
| `@Column` | Configures a column mapping. |
| `@Transient` | Excludes a field or property from persistence mapping. |

### Important Rules for Entity Classes

A portable JPA entity should generally:
- Be a top-level class or a static nested class.
- Be non-final.
- Have a public or protected no-argument constructor.
- Have a primary key.
- Avoid final persistent fields and methods.
- Follow the access strategy defined by its mapping annotations.

A JPA provider may impose additional requirements depending on its implementation and configuration.

**Interview point:** An entity represents persistent data, while a DTO is designed to transfer data between application layers or across an API boundary.

---

## 7. `@Entity` vs `@Table`

`@Entity` identifies a class as a persistent entity.

`@Table` specifies table-level mapping details.

### Example

~~~java
@Entity
@Table(name = "employee_details")
public class Employee {

    @Id
    private Long id;

    private String name;
}
~~~

Here:
- `Employee` is the entity class.
- `employee_details` is the explicitly specified table name.
- `id` is the primary key.

### What Happens If `@Table` Is Not Used?

JPA uses the entity's default table mapping, subject to the provider's naming strategies and configuration.

You do not need to use `@Table` for every entity.

**Interview one-liner:** `@Entity` makes a class persistent, whereas `@Table` customizes the table to which it is mapped.

---

## 8. Primary Keys: `@Id` and `@GeneratedValue`

Every entity must have a primary key.

JPA uses `@Id` to identify the entity's primary-key field or property.

### Example

~~~java
@Id
private Long id;
~~~

The key may be assigned by the application or generated according to a configured strategy.

### `@GeneratedValue`

~~~java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;
~~~

Common generation strategies:

| Strategy | Description |
|---|---|
| `GenerationType.AUTO` | Lets the persistence provider choose an appropriate strategy. |
| `GenerationType.IDENTITY` | Uses a database identity column or equivalent mechanism. |
| `GenerationType.SEQUENCE` | Uses a database sequence. |
| `GenerationType.TABLE` | Uses a database table-based key-generation mechanism. |

### Important Differences

- `IDENTITY` commonly depends on database-generated identity values.
- `SEQUENCE` requires sequence support or a suitable provider strategy.
- `AUTO` allows the provider to select a strategy.
- `TABLE` is portable in concept but can involve additional database operations and contention.

The best strategy depends on the database and application requirements.

**Interview trap:** `@GeneratedValue` is not mandatory if the application assigns primary keys itself.

---

## 9. Field Access vs Property Access

JPA can access persistent attributes through fields or JavaBean properties.

The default access strategy is typically determined by where mapping annotations such as `@Id` are placed.

### Field Access

Annotations are placed on fields.

~~~java
@Entity
public class Employee {

    @Id
    private Long id;

    private String name;

    protected Employee() {
    }
}
~~~

With field access, the provider accesses persistent fields directly rather than relying on getters and setters for persistence access.

### Property Access

Annotations are placed on getter methods.

~~~java
@Entity
public class Employee {

    private Long id;
    private String name;

    protected Employee() {
    }

    @Id
    public Long getId() {
        return id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
~~~

With property access, the provider uses the mapped getter and setter properties.

### Key Difference

| Field Access | Property Access |
|---|---|
| Mapping annotations are generally placed on fields. | Mapping annotations are generally placed on getters. |
| Persistence access targets fields. | Persistence access targets properties. |
| Getters and setters are not required solely for JPA field access. | Proper JavaBean-style accessors are generally needed for mapped properties. |

**Important:** Avoid mixing annotations on fields and getters within one entity unless you intentionally configure the access strategy with `@Access`.

---

## 10. `@Column` and Column Mapping

`@Column` customizes the mapping of an entity attribute to a database column.

### Example

~~~java
@Column(
    name = "employee_name",
    nullable = false,
    length = 100
)
private String name;
~~~

Common attributes:

| Attribute | Purpose |
|---|---|
| `name` | Specifies the database column name. |
| `nullable` | Declares whether the column should be nullable in generated schema metadata. |
| `unique` | Declares a unique constraint in generated schema metadata. |
| `length` | Specifies the length for applicable column types. |
| `precision` | Specifies precision for applicable numeric columns. |
| `scale` | Specifies scale for applicable numeric columns. |
| `insertable` | Controls whether the column is included in generated INSERT statements. |
| `updatable` | Controls whether the column is included in generated UPDATE statements. |

### Example: Non-Updatable Column

~~~java
@Column(updatable = false)
private String createdBy;
~~~

This tells the provider not to include the column in generated UPDATE statements.

It does not independently enforce immutability across every application operation or database client.

**Interview trap:** `nullable = false` is mapping/schema metadata; it is not a replacement for application-level validation or a guaranteed runtime validation mechanism.

---

## 11. `@Transient` vs Java `transient`

These have different meanings.

### JPA `@Transient`

~~~java
@Transient
private String temporaryMessage;
~~~

The attribute is excluded from JPA persistence mapping.

### Java `transient`

~~~java
private transient String temporaryMessage;
~~~

The Java `transient` keyword affects Java's built-in serialization mechanism. Under JPA field access, it also excludes that field from persistent mapping under the JPA rules.

### Difference

| JPA `@Transient` | Java `transient` |
|---|---|
| A JPA annotation. | A Java language keyword. |
| Explicitly excludes an attribute from JPA persistence mapping. | Primarily affects Java serialization and also has JPA mapping implications for field access. |
| Can be used on a mapped property getter. | Applies to fields. |

**Interview point:** Use `@Transient` when you specifically want to express that a property is not persistent in JPA.

---

## 12. What Is `EntityManager`?

`EntityManager` is a standard JPA interface used to interact with the persistence context and manage entity operations.

It supports:
- Persisting new entities.
- Finding entities by primary key.
- Merging detached entity state.
- Removing managed entities.
- Executing JPQL and Criteria queries.
- Flushing pending changes.
- Managing entity state.

### Example

~~~java
@Repository
public class EmployeeDao {

    @PersistenceContext
    private EntityManager entityManager;

    public Employee findById(Long id) {
        return entityManager.find(Employee.class, id);
    }
}
~~~

`@PersistenceContext` injects a container-managed `EntityManager` according to the configured persistence infrastructure.

In a typical Spring Boot application, Spring manages the integration and transaction-aware behavior.

### Common `EntityManager` Methods

| Method | Purpose |
|---|---|
| `persist(entity)` | Makes a new entity managed. |
| `find(Entity.class, id)` | Finds an entity by primary key. |
| `merge(entity)` | Copies entity state into a managed instance and returns that instance. |
| `remove(entity)` | Marks a managed entity for removal. |
| `flush()` | Synchronizes pending persistence-context changes with the database. |
| `refresh(entity)` | Reloads entity state from the database. |
| `clear()` | Detaches all managed entities from the persistence context. |
| `detach(entity)` | Detaches a particular managed entity. |

---

## 13. What Is a Persistence Context?

A **persistence context** is a set of managed entity instances in which each persistent entity identity is associated with a managed object instance.

It tracks entity state and changes so that the persistence provider can synchronize changes with the database.

### Example

~~~java
@Transactional
public void updateEmployee(Long id) {
    Employee employee = entityManager.find(Employee.class, id);

    employee.setSalary(75000);
}
~~~

If `employee` is managed, the provider detects the changed salary through dirty checking and synchronizes it with the database when appropriate.

An explicit `save()` or `update()` call is not required for this managed entity change to be detected.

### Important Persistence Context Concepts

- Managed entities are tracked.
- Changes to managed entities can be detected automatically.
- The persistence context maintains entity identity.
- It helps implement first-level caching.
- Changes are synchronized with the database during flushing.
- It is commonly associated with a transaction-scoped unit of work in Spring applications.

**Interview one-liner:** The persistence context tracks managed entity instances and their changes so that the JPA provider can synchronize them with the database.

---

## 14. Entity Lifecycle States

JPA entities commonly move through four lifecycle states.

### 1. Transient (New)

The object has been created in Java but is not managed by a persistence context.

~~~java
Employee employee = new Employee("Rahul", 50000);
~~~

At this point, merely creating the object does not persist it.

### 2. Managed

The entity is associated with a persistence context.

~~~java
entityManager.persist(employee);
~~~

The provider tracks changes to the managed entity.

### 3. Detached

The entity was previously managed but is no longer associated with the current persistence context.

This can happen when:
- The persistence context is cleared.
- The entity is explicitly detached.
- The persistence context ends.

~~~java
entityManager.detach(employee);
~~~

Changes made to a detached entity are not automatically tracked by that persistence context.

### 4. Removed

A managed entity has been marked for deletion.

~~~java
entityManager.remove(employee);
~~~

The corresponding database deletion occurs when the persistence provider synchronizes the removal, typically during a flush.

### Lifecycle Summary

~~~text
Transient
    |
    | persist()
    v
Managed
    |
    | detach() / clear() / context ends
    v
Detached
    |
    | merge(detachedEntity)
    v
Managed copy

Managed
    |
    | remove()
    v
Removed
~~~

This is a simplified diagram. The exact state transitions depend on the operation and the entity's current state.

### Interview Trap: Does `merge()` Make the Original Object Managed?

No. `merge()` copies state into a managed instance and returns that managed instance. The original detached object remains detached.

Correct usage:

~~~java
Employee managedEmployee = entityManager.merge(detachedEmployee);
~~~

Do not assume `detachedEmployee` itself becomes managed.

---

## 15. What Is Dirty Checking?

**Dirty checking** is the mechanism through which a JPA provider detects changes to managed entities and synchronizes those changes with the database.

### Example

~~~java
@Transactional
public void increaseSalary(Long id) {
    Employee employee = entityManager.find(Employee.class, id);

    employee.setSalary(employee.getSalary() + 5000);
}
~~~

When the transaction is flushed or committed, the provider can detect the salary change and generate an UPDATE statement.

Conceptually:

~~~sql
UPDATE employees
SET salary = ?
WHERE id = ?;
~~~

The exact SQL and timing depend on the provider and configuration.

### Important Points

- Dirty checking applies to managed entities.
- It does not automatically track changes to detached objects.
- An explicit `save()` is generally unnecessary for a change to a managed entity.
- Flush may occur before transaction commit, including before certain queries.
- Changes can still be rolled back if the transaction is rolled back.

**Interview one-liner:** Dirty checking automatically detects modifications to managed entities and synchronizes those changes with the database.

---

## 16. What Is Flushing?

**Flushing** synchronizes pending persistence-context changes with the database.

It does not necessarily commit the transaction.

### Example

~~~java
@Transactional
public void createEmployee() {
    Employee employee = new Employee("Priya", 60000);

    entityManager.persist(employee);
    entityManager.flush();
}
~~~

`flush()` requests synchronization of pending changes to the database.

### Flush vs Commit

| Flush | Commit |
|---|---|
| Synchronizes pending persistence changes with the database. | Completes the transaction successfully. |
| Does not itself commit the transaction. | Makes the transaction's changes durable according to the database's transaction semantics. |
| May happen multiple times in a transaction. | Normally occurs once at successful transaction completion. |
| SQL may execute during flush. | Commit finalizes the transaction. |

**Interview trap:** An INSERT statement executing during flush does not mean the transaction has been committed.

---

## 17. First-Level Cache

The first-level cache is the persistence-context cache associated with an `EntityManager`.

It is part of JPA's persistence-context behavior.

### Example

~~~java
Employee first = entityManager.find(Employee.class, 1L);
Employee second = entityManager.find(Employee.class, 1L);
~~~

When both calls occur within the same persistence context and the entity remains managed, the provider can return the same managed object instance without executing a second database query.

### Important Points

- It is associated with the persistence context.
- It maintains identity for managed entities.
- It is not a shared application-wide cache.
- It generally ends with the persistence context.
- It can reduce repeated database access for the same managed entity.

**Interview trap:** First-level caching is not the same as Hibernate's optional second-level cache.

---

## 18. What Is Spring Data `JpaRepository`?

`JpaRepository` is a Spring Data JPA repository interface that combines repository abstractions for CRUD operations, pagination, sorting, and JPA-specific functionality.

### Example

~~~java
import org.springframework.data.jpa.repository.JpaRepository;

public interface EmployeeRepository
        extends JpaRepository<Employee, Long> {
}
~~~

Here:
- `Employee` is the entity type.
- `Long` is the entity's primary-key type.

Spring Data JPA provides a repository implementation at runtime.

### Common Methods

| Method | Purpose |
|---|---|
| `save(entity)` | Saves an entity using new/existing entity detection and persistence operations. |
| `findById(id)` | Finds an entity by ID and returns an `Optional`. |
| `findAll()` | Retrieves all matching entities. |
| `existsById(id)` | Checks whether an entity with the ID exists. |
| `deleteById(id)` | Deletes an entity by ID when found, according to repository behavior. |
| `count()` | Counts entities. |
| `findAll(Pageable)` | Retrieves a page of entities. |
| `findAll(Sort)` | Retrieves entities with sorting. |

### `JpaRepository` vs `CrudRepository`

| `CrudRepository` | `JpaRepository` |
|---|---|
| Provides basic CRUD operations. | Provides CRUD plus additional JPA-specific repository functionality. |
| Does not itself define the full JPA-specific API. | Adds capabilities such as flushing and batch-oriented repository methods. |
| More general repository abstraction. | Specifically designed for JPA-backed repositories. |

**Interview point:** Use `JpaRepository` when the application uses Spring Data JPA and needs its additional repository capabilities.

---

## 19. How Does `save()` Work in Spring Data JPA?

Spring Data JPA's `save()` generally decides whether to call `EntityManager.persist()` or `EntityManager.merge()` based on whether the entity is considered new.

Conceptually:

~~~java
if (entityIsNew) {
    entityManager.persist(entity);
    return entity;
} else {
    return entityManager.merge(entity);
}
~~~

This is simplified pseudocode, not the complete implementation. Spring Data JPA uses entity-newness detection, which can be customized.

### Important Points

- `save()` is not simply an SQL INSERT method.
- A new entity is typically persisted.
- An existing or non-new entity may be merged.
- The returned instance matters when merge semantics apply.
- `save()` does not necessarily execute SQL immediately.
- SQL execution may occur during flush or transaction completion.

### Does `save()` Always Insert?

No. It can lead to an INSERT or UPDATE depending on entity state, newness detection, identifier configuration, and provider behavior.

### Does `save()` Always Update?

No. New entities are typically inserted.

### Is `save()` Required After Updating a Managed Entity?

Generally, no. A managed entity's changes are tracked through dirty checking.

Example:

~~~java
@Transactional
public void updateEmployee(Long id) {
    Employee employee = employeeRepository.findById(id)
        .orElseThrow();

    employee.setSalary(80000);
}
~~~

The transaction's persistence context tracks the change. An additional `save(employee)` is generally unnecessary in this managed-entity scenario.

---

## 20. What Are Derived Query Methods?

Spring Data JPA can derive queries from repository method names.

### Example

~~~java
public interface EmployeeRepository
        extends JpaRepository<Employee, Long> {

    List<Employee> findByName(String name);

    List<Employee> findBySalaryGreaterThan(double salary);

    List<Employee> findByNameAndSalaryGreaterThan(
        String name,
        double salary
    );

    boolean existsByName(String name);

    long countBySalaryGreaterThan(double salary);

    void deleteByName(String name);
}
~~~

Spring Data parses supported method-name patterns and creates the appropriate query.

### Common Keywords

| Keyword | Meaning |
|---|---|
| `findBy` | Finds records matching criteria. |
| `existsBy` | Checks for matching records. |
| `countBy` | Counts matching records. |
| `deleteBy` | Deletes matching records. |
| `And` | Combines conditions using AND. |
| `Or` | Combines conditions using OR. |
| `GreaterThan` | Greater-than comparison. |
| `LessThan` | Less-than comparison. |
| `Between` | Range comparison. |
| `Containing` | Matches a containing string or collection element according to the property type. |
| `StartingWith` | Matches values beginning with a given value. |
| `EndingWith` | Matches values ending with a given value. |
| `OrderBy` | Specifies ordering in the method name. |
| `IgnoreCase` | Requests case-insensitive comparison for supported property types and providers. |

**Interview trap:** Derived query method names use Java entity property names, not necessarily database column names.

---

## 21. What Is `@Query`?

`@Query` lets developers declare a repository query explicitly.

### JPQL Example

~~~java
public interface EmployeeRepository
        extends JpaRepository<Employee, Long> {

    @Query("SELECT e FROM Employee e WHERE e.salary > :salary")
    List<Employee> findEmployeesWithSalaryGreaterThan(
        @Param("salary") double salary
    );
}
~~~

JPQL queries entity names and Java entity attributes rather than directly querying table and column names.

### Native SQL Example

~~~java
@Query(
    value = "SELECT * FROM employees WHERE salary > :salary",
    nativeQuery = true
)
List<Employee> findEmployeesUsingNativeSql(
    @Param("salary") double salary
);
~~~

Native queries use SQL understood by the target database and may be database-specific.

### JPQL vs Native SQL

| JPQL | Native SQL |
|---|---|
| Queries entities and entity attributes. | Queries database tables and columns. |
| More database-independent. | Can use database-specific features. |
| Uses object-oriented persistence semantics. | Uses SQL directly. |
| Example: `SELECT e FROM Employee e` | Example: `SELECT * FROM employees` |

**Interview one-liner:** JPQL operates on entity models, whereas native queries operate directly on database SQL structures.

---

## 22. What Is the Difference Between `findById()` and `getReferenceById()`?

Both repository methods provide access to an entity by its identifier, but their behavior differs.

### `findById()`

~~~java
Optional<Employee> employee = employeeRepository.findById(1L);
~~~

- Returns an `Optional<Employee>`.
- Usually performs a database lookup if the entity is not already available in the persistence context.
- Represents the possibility that the entity does not exist.

### `getReferenceById()`

~~~java
Employee employee = employeeRepository.getReferenceById(1L);
~~~

- Returns an entity reference, which may be a proxy or another provider-managed reference.
- May defer database access until the entity's state is accessed.
- If the row does not exist, failure may occur when the reference is initialized rather than at the method call.
- Accessing an uninitialized reference after its persistence context is closed can cause a lazy-initialization-related failure.

**Interview point:** Use `findById()` when you need to check whether the record exists. Use `getReferenceById()` when a reference is sufficient and its deferred-loading behavior is appropriate.

---

## 23. What Is Lazy Loading?

Lazy loading means that associated data is loaded when it is accessed rather than necessarily being fetched immediately with the owning entity.

Example:

~~~java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "department_id")
private Department department;
~~~

When the `Employee` is loaded, its associated `Department` may not be loaded until the association is accessed.

### Lazy vs Eager Loading

| Lazy Loading | Eager Loading |
|---|---|
| Associated data may be fetched when accessed. | Associated data is required to be fetched eagerly according to the mapping contract. |
| Can avoid loading unused relationships. | Can load data that the application does not need. |
| May result in additional queries. | Can increase the amount of data fetched. |
| Can cause lazy-initialization issues outside the persistence context. | Does not guarantee one SQL query; the provider may use multiple queries. |

### Default Fetch Types in JPA

| Relationship | Default Fetch Type |
|---|---|
| `@OneToOne` | EAGER |
| `@ManyToOne` | EAGER |
| `@OneToMany` | LAZY |
| `@ManyToMany` | LAZY |

**Important:** Prefer explicit fetch plans that fit the use case rather than relying on defaults.

---

## 24. What Is the N+1 Query Problem?

The N+1 query problem occurs when an application executes one query to fetch a group of entities and then additional queries to load associated data for each entity.

Example:
1. One query fetches 100 employees.
2. Each employee's department is loaded separately.
3. The application may execute one query for employees plus up to 100 department queries.

This can cause performance problems.

### Common Solutions

- Use `JOIN FETCH` in JPQL when appropriate.
- Use `@EntityGraph` to define a fetch plan.
- Use DTO projections when only specific fields are needed.
- Use batch fetching where supported and configured.
- Inspect generated SQL and query counts.
- Avoid indiscriminately changing every relationship to EAGER.

### Example: `JOIN FETCH`

~~~java
@Query("""
    SELECT e
    FROM Employee e
    JOIN FETCH e.department
    """)
List<Employee> findEmployeesWithDepartments();
~~~

This fetches employees together with their associated departments using a fetch join.

The precise SQL and behavior depend on the provider and mapping. Pagination with collection fetch joins requires particular care.

**Interview one-liner:** The N+1 problem occurs when loading N entities triggers additional queries for their relationships; fetch joins, entity graphs, projections, or batch fetching can help avoid it.

---

## 25. What Is the Difference Between an Entity and a DTO?

An entity represents persistent domain data and participates in ORM management.

A DTO (Data Transfer Object) carries data between layers or across an API boundary.

### Entity Example

~~~java
@Entity
public class Employee {

    @Id
    @GeneratedValue
    private Long id;

    private String name;
    private double salary;
}
~~~

### DTO Example

~~~java
public record EmployeeResponse(
    Long id,
    String name
) {
}
~~~

The DTO exposes only the fields required by the API.

### Entity vs DTO

| Entity | DTO |
|---|---|
| Represents persistent domain data. | Transfers data between layers or API boundaries. |
| Managed by the persistence context when attached. | Usually not managed by JPA. |
| Contains persistence mapping annotations. | Normally has no JPA mapping annotations. |
| May contain relationships and persistence-specific behavior. | Contains only the data required for a particular operation or response. |
| Often maps to a database table or view. | Does not need a corresponding table. |

### Why Avoid Returning Entities Directly from REST APIs?

- Can expose fields that should remain internal.
- Can create unwanted coupling between database structure and API contracts.
- Bidirectional relationships can cause JSON serialization recursion.
- Lazy relationships can trigger unexpected queries.
- Serialization can fail when lazy associations are accessed outside the persistence context.
- API response shape may change unintentionally when entity mappings change.

**Interview point:** Entities model persistence; DTOs model data exchanged across application boundaries.

---

## 26. What Is the Difference Between `javax.persistence` and `jakarta.persistence`?

The persistence API package changed from `javax.persistence` to `jakarta.persistence` as part of the Jakarta EE transition.

### Older Applications

~~~java
import javax.persistence.Entity;
import javax.persistence.Id;
~~~

### Modern Spring Boot Applications

~~~java
import jakarta.persistence.Entity;
import jakarta.persistence.Id;
~~~

For Spring Boot 3.x and later, use `jakarta.persistence` for JPA annotations and interfaces.

**Interview trap:** Do not mix `javax.persistence` and `jakarta.persistence` APIs in a modern Spring Boot 3+ application.

---

## 27. What Is Schema Generation?

JPA providers can generate or modify database schema structures based on entity mappings when schema-generation settings are enabled.

In Spring Boot, a common property is:

~~~properties
spring.jpa.hibernate.ddl-auto=update
~~~

Common Hibernate schema modes used by Spring Boot:

| Value | Purpose |
|---|---|
| `none` | Does not manage the schema through this Hibernate schema-management setting. |
| `validate` | Validates the schema against the entity mappings. |
| `update` | Attempts to update the schema to match the mappings. |
| `create` | Creates the schema, potentially dropping existing schema objects first. |
| `create-drop` | Creates the schema and drops it when the relevant session factory or application context shuts down. |

Exact behavior depends on the Hibernate version and configuration.

### Which Mode Should Be Used?

- Development: `update` can be convenient for experiments, though it is not a full migration solution.
- Testing: use an isolated test database or controlled schema setup.
- Production: prefer versioned migration tools such as Flyway or Liquibase and generally use `validate` or `none` for Hibernate schema management.

**Important:** `update` is not a substitute for database migration scripts. It does not provide a complete, reliable migration history or guarantee safe handling of every schema change.

---

## 28. What Is the Difference Between JDBC, JPA, Hibernate, and Spring Data JPA?

| Technology | Responsibility |
|---|---|
| JDBC | Provides low-level Java database connectivity and SQL execution APIs. |
| JPA | Defines a standard persistence API and ORM specification. |
| Hibernate | Implements ORM and JPA persistence behavior. |
| Spring Data JPA | Simplifies persistence access through Spring repository abstractions. |

### Conceptual Flow

~~~text
Service Layer
     |
     v
Repository Interface
     |
     v
Spring Data JPA
     |
     v
JPA / EntityManager
     |
     v
Hibernate Provider
     |
     v
JDBC
     |
     v
Database
~~~

The exact call path varies depending on the operation and configuration.

---

## 29. Common Interview Questions and Answers

### Q1. What is JPA?

JPA is a Java persistence specification that defines object-relational mapping and persistence APIs.

### Q2. Is JPA an implementation?

No. JPA is a specification; Hibernate is a commonly used implementation.

### Q3. What is Spring Data JPA?

It is a Spring Data module that simplifies JPA persistence through repository interfaces and generated implementations.

### Q4. What is an entity?

An entity is a Java class mapped to persistent data and managed through JPA.

### Q5. Why does an entity need a primary key?

JPA uses the primary key to identify an entity and maintain its persistent identity.

### Q6. What is a persistence context?

It is the set of managed entity instances tracked by the persistence provider.

### Q7. What is dirty checking?

It is the automatic detection of changes to managed entities so that the provider can synchronize them with the database.

### Q8. What is the difference between `persist()` and `merge()`?

`persist()` makes a new entity managed. `merge()` copies state into a managed instance and returns that instance.

### Q9. Does `merge()` attach the original detached object?

No. The returned instance is managed; the original detached instance remains detached.

### Q10. Does flushing commit a transaction?

No. Flushing synchronizes pending persistence changes with the database but does not itself commit the transaction.

### Q11. What is first-level caching?

It is the persistence-context cache associated with an `EntityManager`.

### Q12. What is the difference between lazy and eager loading?

Lazy loading may defer loading associated data until access; eager loading requires associated data to be fetched eagerly according to the mapping contract.

### Q13. What is the N+1 query problem?

It occurs when fetching a group of entities triggers additional queries to load related data for individual entities.

### Q14. What is the purpose of `JpaRepository`?

It provides CRUD, pagination, sorting, and additional JPA-oriented repository operations.

### Q15. Does `save()` always execute an INSERT?

No. It may result in an INSERT or an update-related operation depending on newness detection and entity state.

### Q16. Is `save()` required after modifying a managed entity?

Generally, no. Dirty checking detects modifications to managed entities within the persistence context.

### Q17. What is the difference between JPQL and SQL?

JPQL queries entity names and Java attributes, whereas SQL queries database tables and columns.

### Q18. Why use DTOs instead of returning entities from controllers?

DTOs provide controlled API responses, reduce coupling to persistence mappings, and help avoid serialization and lazy-loading problems.

### Q19. What is the difference between `findById()` and `getReferenceById()`?

`findById()` returns an `Optional` after attempting to find the entity. `getReferenceById()` returns a reference and may defer loading or missing-row detection.

### Q20. What is the difference between `@Entity` and `@Table`?

`@Entity` declares a persistent entity; `@Table` customizes its table mapping.

---

## 30. Common Interview Traps

- **Trap:** JPA is a framework.
  - **Correct:** JPA is a specification/API.

- **Trap:** Hibernate and Spring Data JPA are the same.
  - **Correct:** Hibernate implements ORM/JPA behavior; Spring Data JPA simplifies repository development.

- **Trap:** Calling `save()` always executes SQL immediately.
  - **Correct:** SQL may execute later during flushing or transaction completion.

- **Trap:** Every entity change requires `save()`.
  - **Correct:** Managed entity changes are tracked through dirty checking.

- **Trap:** `merge()` makes the original detached object managed.
  - **Correct:** It returns a managed instance containing the copied state.

- **Trap:** `flush()` commits a transaction.
  - **Correct:** Flush synchronizes changes; commit completes the transaction.

- **Trap:** EAGER fetching means a single SQL query.
  - **Correct:** EAGER fetching does not guarantee a single query.

- **Trap:** LAZY fetching always means one query per relationship.
  - **Correct:** Query behavior depends on access patterns, fetch plans, provider behavior, and batch settings.

- **Trap:** `@Column(nullable = false)` guarantees request validation.
  - **Correct:** Database constraints, schema generation, and application validation are separate concerns.

- **Trap:** `JpaRepository` replaces Hibernate.
  - **Correct:** It builds on JPA and uses a persistence provider.

- **Trap:** A DTO must map to a database table.
  - **Correct:** DTOs are data-transfer structures and do not require persistence mapping.

- **Trap:** `spring.jpa.hibernate.ddl-auto=update` is a complete production migration strategy.
  - **Correct:** Use versioned database migrations for controlled schema evolution.

---

## 31. Quick Revision

- **JPA:** Persistence specification and standard APIs.
- **Hibernate:** Common JPA implementation and ORM framework.
- **Spring Data JPA:** Simplifies database access using repository abstractions.
- **ORM:** Maps Java objects to relational database structures.
- **`@Entity`:** Marks a persistent entity.
- **`@Table`:** Customizes table mapping.
- **`@Id`:** Defines the primary key.
- **`@GeneratedValue`:** Configures key generation.
- **`EntityManager`:** Manages persistence operations and entity state.
- **Persistence Context:** Tracks managed entity instances.
- **Managed Entity:** An entity tracked by the current persistence context.
- **Dirty Checking:** Detects changes to managed entities.
- **Flush:** Synchronizes pending changes with the database.
- **Commit:** Completes the transaction.
- **First-Level Cache:** Persistence-context cache.
- **`JpaRepository`:** Provides CRUD and extended repository methods.
- **`save()`:** Persists or merges based on newness detection.
- **Lazy Loading:** Defers loading of associated data when possible.
- **N+1 Problem:** Additional queries are executed for related data.
- **JPQL:** Queries entity models.
- **DTO:** Transfers data without exposing persistence entities directly.
- **Schema Migrations:** Use tools such as Flyway or Liquibase for controlled database changes.

### One-Line Interview Summary

Spring Data JPA simplifies repository operations, JPA defines standard persistence behavior, and Hibernate commonly implements that behavior by mapping managed Java entities to relational database records.

---

## 32. Related Topics

- [02. Entity Relationships & Mapping](02-Entity-Relationships-and-Mapping.md)
- [03. Repository Methods, JPQL & Native Queries](03-Repositories-and-Queries.md)
- [04. Lazy Loading, Fetching & N+1 Problem](04-Lazy-Loading-and-N-plus-one.md)
- [05. Transactions & Persistence Context](../08-Transactions/01-Spring-Transactions-and-Propagation.md)
- [06. DTOs, Validation & REST Responses](../05-Spring-MVC/04-Responses-and-DTOs.md)
- [07. Exception Handling](../06-Exception-Handling/01-Exception-Handling.md)