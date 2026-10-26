# 01. @Transactional & Transaction Management

## 1. What Is a Transaction?

A **transaction** is a group of database operations treated as a single logical unit of work.

A transaction should either complete successfully or be rolled back when a failure requires the changes to be undone.

### Example

Suppose we are transferring ₹1,000 from Account A to Account B.

The transfer requires two operations:

1. Deduct ₹1,000 from Account A.
2. Add ₹1,000 to Account B.

If the deduction succeeds but the addition fails, the deduction must be rolled back to prevent money from being lost.

### SQL Example

~~~sql
BEGIN;

UPDATE accounts
SET balance = balance - 1000
WHERE id = 1;

UPDATE accounts
SET balance = balance + 1000
WHERE id = 2;

COMMIT;
~~~

If an operation fails, the transaction can be rolled back instead of committed.

**Interview one-liner:** A transaction ensures that a group of operations is completed as a single logical unit, preserving data consistency.

---

## 2. What Is Transaction Management in Spring?

Spring transaction management provides a consistent way to manage transactions across application operations.

It supports:
- Declarative transaction management using `@Transactional`.
- Programmatic transaction management using `TransactionTemplate` or `PlatformTransactionManager`.
- Transaction propagation.
- Isolation levels.
- Rollback rules.
- Integration with JDBC, JPA, Hibernate, and supported transaction managers.

### Types of Transaction Management

| Type | Description |
|---|---|
| Declarative | Transaction behavior is declared using annotations or configuration. |
| Programmatic | Application code explicitly controls transaction operations. |

### Declarative Example

~~~java
@Service
public class EmployeeService {

    @Transactional
    public void updateEmployee() {
        // Database operations
    }
}
~~~

Spring manages the transaction around the method when the invocation passes through the configured transactional infrastructure.

The developer does not need to write explicit begin, commit, and rollback code for the typical declarative use case.

---

## 3. What Is `@Transactional`?

`@Transactional` is a Spring annotation used to declare transaction boundaries and transaction attributes.

It can be applied to:
- Methods.
- Classes.

### Example

~~~java
@Service
public class EmployeeService {

    private final EmployeeRepository employeeRepository;

    public EmployeeService(EmployeeRepository employeeRepository) {
        this.employeeRepository = employeeRepository;
    }

    @Transactional
    public void updateSalary(Long id, double salary) {
        Employee employee = employeeRepository.findById(id)
            .orElseThrow();

        employee.setSalary(salary);
    }
}
~~~

When invoked through Spring's transactional proxy, the method executes within a transaction according to the configured transaction manager and transaction attributes.

If the entity is managed, dirty checking can persist the salary change without an explicit `save()` call.

### What Happens Internally?

A simplified flow:

1. A caller invokes a Spring-managed service method.
2. The call passes through a transactional proxy or equivalent transaction infrastructure.
3. Spring checks the transaction attributes.
4. Spring starts a new transaction or joins an existing transaction according to the propagation setting.
5. The target method executes.
6. Spring commits the transaction if execution completes successfully.
7. Spring rolls back when an applicable rollback rule is triggered by an exception.

**Important:** `@Transactional` does not itself execute SQL or guarantee that a transaction will work with every arbitrary method invocation. It relies on Spring transaction infrastructure and a compatible transaction manager.

---

## 4. Where Should We Use `@Transactional`?

In a typical layered Spring Boot application, transaction boundaries are commonly placed in the **service layer**.

### Example Architecture

~~~text
Controller
    |
    v
Service (@Transactional)
    |
    v
Repository
    |
    v
Database
~~~

### Why the Service Layer?

A business operation may involve multiple repository calls.

For example, transferring money may require:
- Reading Account A.
- Reading Account B.
- Updating Account A.
- Updating Account B.

The service layer can coordinate these operations within one transaction.

~~~java
@Service
public class AccountService {

    private final AccountRepository accountRepository;

    public AccountService(AccountRepository accountRepository) {
        this.accountRepository = accountRepository;
    }

    @Transactional
    public void transfer(Long fromId, Long toId, double amount) {
        Account from = accountRepository.findById(fromId)
            .orElseThrow();

        Account to = accountRepository.findById(toId)
            .orElseThrow();

        from.setBalance(from.getBalance() - amount);
        to.setBalance(to.getBalance() + amount);
    }
}
~~~

Assuming both entities are managed in the same transaction, dirty checking can persist the changes together.

**Interview point:** Place transaction boundaries around business operations rather than automatically adding `@Transactional` to every method.

---

## 5. Class-Level vs Method-Level `@Transactional`

`@Transactional` can be applied to a class or a method.

### Class-Level

~~~java
@Service
@Transactional
public class EmployeeService {

    public void createEmployee() {
    }

    public void updateEmployee() {
    }
}
~~~

The class-level annotation provides default transaction attributes for eligible methods.

### Method-Level

~~~java
@Service
@Transactional
public class EmployeeService {

    public void createEmployee() {
    }

    @Transactional(readOnly = true)
    public Employee getEmployee(Long id) {
        return employeeRepository.findById(id)
            .orElseThrow();
    }
}
~~~

The method-level annotation takes precedence over the class-level transaction attributes for that method.

### Important Points

- A class-level annotation acts as a default for eligible methods.
- A method-level annotation can override class-level attributes.
- In proxy-based transaction management, only calls intercepted by the transactional infrastructure receive the declared behavior.
- For reliable proxy-based behavior, public service methods are the common convention.

**Interview one-liner:** Class-level `@Transactional` provides defaults for eligible methods, while method-level configuration can override those defaults.

---

## 6. ACID Properties

Transactions are commonly explained through the ACID properties.

| Property | Meaning |
|---|---|
| Atomicity | All operations in the transaction succeed together or are rolled back together. |
| Consistency | The transaction preserves defined data integrity constraints and application invariants. |
| Isolation | Concurrent transactions are controlled according to the selected isolation level. |
| Durability | Committed changes persist according to the database's durability guarantees. |

### Example: Bank Transfer

- **Atomicity:** Both account updates succeed together or are rolled back.
- **Consistency:** Business rules, such as valid balances, remain satisfied.
- **Isolation:** Concurrent transfers do not produce invalid results.
- **Durability:** Once committed, the transfer remains recorded according to database guarantees.

**Interview trap:** ACID is a set of transaction properties, not four annotations that must be added to a Spring service.

---

## 7. Default Transaction Behavior in Spring

When using Spring's default declarative transaction settings, the key defaults are:

| Attribute | Default |
|---|---|
| Propagation | `Propagation.REQUIRED` |
| Isolation | `Isolation.DEFAULT` |
| Read-only | `false` |
| Timeout | Transaction manager or underlying system default |
| Rollback for unchecked exceptions | Yes, by default |
| Rollback for `Error` | Yes, by default |
| Rollback for checked exceptions | No, by default |

These are Spring's default transaction attributes. Actual behavior also depends on the transaction manager, resource, and database.

### Example

~~~java
@Transactional
public void processPayment() {
    // Transactional work
}
~~~

Unless configured otherwise, the method uses `REQUIRED` propagation, the default isolation level, and Spring's standard rollback rules.

**Important:** `Isolation.DEFAULT` means Spring uses the underlying transaction system's default isolation level; it does not mean `READ_COMMITTED` on every database.

---

## 8. Rollback Behavior

One of the most important `@Transactional` interview topics is rollback behavior.

### Default Rollback Rules

Spring rolls back by default when a transaction encounters:
- An unchecked exception extending `RuntimeException`.
- An `Error`.

Spring does not automatically roll back by default for every checked exception.

### Example: Runtime Exception

~~~java
@Transactional
public void createEmployee() {
    employeeRepository.save(new Employee());

    throw new IllegalStateException("Operation failed");
}
~~~

Under the default rollback rules, the unchecked exception triggers rollback when it escapes the transactional boundary.

### Example: Checked Exception

~~~java
@Transactional
public void createEmployee() throws IOException {
    employeeRepository.save(new Employee());

    throw new IOException("File operation failed");
}
~~~

A checked exception such as `IOException` does not trigger rollback by default merely because it escapes the method.

The transaction may therefore commit database changes if no other rollback condition applies.

### Configure Rollback for a Checked Exception

~~~java
@Transactional(rollbackFor = IOException.class)
public void createEmployee() throws IOException {
    employeeRepository.save(new Employee());

    throw new IOException("Operation failed");
}
~~~

### Configure No Rollback for an Exception

~~~java
@Transactional(noRollbackFor = IllegalArgumentException.class)
public void process() {
    // Work that should not be rolled back for this exception
}
~~~

This changes the rollback rule for the specified exception type, subject to the transaction's overall outcome.

### Important Caveat

Rollback rules apply to exceptions observed by the transaction interceptor. If an exception is caught and suppressed inside the transactional method, Spring may see a normal return and commit the transaction unless it has otherwise been marked rollback-only.

**Interview one-liner:** By default, Spring rolls back for unchecked exceptions and errors, but not for checked exceptions.

---

## 9. `rollbackFor` vs `noRollbackFor`

| Attribute | Purpose |
|---|---|
| `rollbackFor` | Specifies exception types that should trigger rollback. |
| `noRollbackFor` | Specifies exception types that should not trigger rollback. |

### Example

~~~java
@Transactional(
    rollbackFor = Exception.class,
    noRollbackFor = IllegalArgumentException.class
)
public void process() throws Exception {
    // Transactional work
}
~~~

Here, `Exception` is configured to trigger rollback, except that `IllegalArgumentException` is explicitly configured not to trigger rollback.

Spring evaluates the most specific applicable rollback rule when multiple rules match.

**Interview trap:** `rollbackFor = Exception.class` is not the default behavior. It explicitly broadens rollback behavior to include checked exceptions.

---

## 10. What Happens If an Exception Is Caught Inside the Method?

Consider this example:

~~~java
@Transactional
public void process() {
    try {
        employeeRepository.save(new Employee());

        throw new RuntimeException("Failure");
    } catch (RuntimeException ex) {
        System.out.println("Exception handled");
    }
}
~~~

Because the exception is caught and does not escape the method, the transactional interceptor normally sees a successful return.

If the transaction has not already been marked rollback-only and no other failure occurs, Spring can commit the transaction.

### How to Ensure Rollback?

Option 1: Let the exception escape the transactional method.

~~~java
@Transactional
public void process() {
    employeeRepository.save(new Employee());

    throw new RuntimeException("Failure");
}
~~~

Option 2: Mark the current transaction rollback-only when appropriate.

~~~java
import org.springframework.transaction.interceptor.TransactionAspectSupport;

@Transactional
public void process() {
    try {
        // Database operations
    } catch (RuntimeException ex) {
        TransactionAspectSupport.currentTransactionStatus()
            .setRollbackOnly();

        throw ex;
    }
}
~~~

Use programmatic rollback-only marking deliberately; it couples the method to Spring's transaction infrastructure.

**Interview point:** Catching an exception does not automatically cause rollback. What matters is whether the transaction is marked rollback-only or an applicable rollback-triggering failure reaches the transaction boundary.

---

## 11. What Is Transaction Propagation?

**Propagation** determines how a transactional method behaves when it is invoked while a transaction already exists.

Spring provides several propagation modes through `Propagation`.

### Propagation Types

| Propagation | Behavior |
|---|---|
| `REQUIRED` | Joins the existing transaction or creates a new one if none exists. |
| `REQUIRES_NEW` | Suspends the existing transaction, if supported, and starts a new transaction. |
| `SUPPORTS` | Executes within an existing transaction if one exists; otherwise executes without one. |
| `NOT_SUPPORTED` | Suspends an existing transaction, if supported, and executes without a transaction. |
| `MANDATORY` | Requires an existing transaction; throws an exception if none exists. |
| `NEVER` | Requires that no transaction exists; throws an exception if one does. |
| `NESTED` | Executes within a nested transaction scope when supported, often using a savepoint. |

### Default: `REQUIRED`

~~~java
@Transactional
public void createOrder() {
    // Transaction A
    paymentService.processPayment();
}
~~~

If `processPayment()` is also transactional with `REQUIRED` propagation and is invoked through the appropriate transactional infrastructure, it normally participates in the existing transaction.

Both operations share the same physical transaction in the usual single-transaction-manager scenario.

### `REQUIRES_NEW`

~~~java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void saveAuditLog() {
    // Independent transaction
}
~~~

If called while another transaction exists, Spring suspends that transaction and starts an independent transaction when the transaction manager supports this behavior.

The inner transaction can commit even if the outer transaction later rolls back.

### Important Resource Consideration

`REQUIRES_NEW` may require an additional database connection while the outer transaction's connection remains held. Under heavy concurrency, this can exhaust a connection pool if it is not sized appropriately.

### `NESTED`

~~~java
@Transactional(propagation = Propagation.NESTED)
public void performNestedOperation() {
    // Nested transaction scope, if supported
}
~~~

Nested transactions often use database savepoints within one physical transaction.

Rolling back to a savepoint does not create an independently committed transaction.

Support depends on the transaction manager and underlying resource.

---

## 12. `REQUIRED` vs `REQUIRES_NEW` vs `NESTED`

| Feature | `REQUIRED` | `REQUIRES_NEW` | `NESTED` |
|---|---|---|---|
| Existing transaction | Joins it | Suspends it and starts another | Uses a nested scope when supported |
| Physical transaction | Usually shared | Independent transaction | Usually the same physical transaction |
| Independent commit | No | Yes | No |
| Savepoint-based rollback | Not inherently | Not inherently | Commonly used |
| Common use case | One atomic business operation | Independent audit or logging operation | Partial rollback within a larger transaction |

### Interview Scenario

Suppose an order operation calls a payment operation.

- `REQUIRED`: order and payment normally share one transaction.
- `REQUIRES_NEW`: payment may commit independently of the order transaction.
- `NESTED`: a supported savepoint can allow rollback of the nested work while retaining earlier outer work.

**Important:** Independent database transactions do not guarantee that external side effects, such as sending emails or calling remote services, can be rolled back.

---

## 13. What Is Transaction Isolation?

Isolation determines how concurrent transactions interact with one another.

Spring exposes isolation settings through `Isolation`.

### Isolation Levels

| Isolation | General Meaning |
|---|---|
| `DEFAULT` | Uses the database or transaction system's default isolation. |
| `READ_UNCOMMITTED` | May allow reading uncommitted changes, depending on the database. |
| `READ_COMMITTED` | Prevents dirty reads. |
| `REPEATABLE_READ` | Prevents dirty reads and non-repeatable reads under the standard isolation model. |
| `SERIALIZABLE` | Provides the strongest standard isolation model, subject to implementation and possible concurrency failures. |

Actual behavior differs between database engines and their concurrency-control implementations.

### Common Concurrency Problems

**Dirty Read:** A transaction reads data written by another transaction that has not committed.

**Non-Repeatable Read:** A transaction reads the same row twice and sees different committed values.

**Phantom Read:** Repeating a range query returns a different set of rows because another transaction inserted or deleted matching rows.

### Example

~~~java
@Transactional(isolation = Isolation.READ_COMMITTED)
public void updateEmployee() {
    // Database operations
}
~~~

This requests `READ_COMMITTED` isolation for the transaction when the transaction manager and database support it.

### Important Points

- Higher isolation can reduce concurrency and increase contention.
- Isolation behavior depends on the database.
- An isolation setting may only apply when Spring starts a new transaction.
- Joining an existing transaction generally means inheriting that transaction's actual isolation characteristics.
- Serializable isolation can still result in serialization failures that require retry handling.

**Interview one-liner:** Isolation controls the visibility and interaction of concurrent transactions to help prevent inconsistent reads and updates.

---

## 14. What Is a Read-Only Transaction?

`readOnly = true` declares that a transaction is intended for read operations.

~~~java
@Transactional(readOnly = true)
public Employee getEmployee(Long id) {
    return employeeRepository.findById(id)
        .orElseThrow();
}
~~~

### Benefits

Depending on the transaction manager and persistence provider, read-only transactions can:
- Communicate read-only intent to the database.
- Enable provider-specific optimizations.
- Reduce some persistence-context work, such as dirty checking in supported Hibernate configurations.
- Improve clarity about the method's intended responsibility.

### Important Caveat

`readOnly = true` is generally a hint, not a universal guarantee that writes are impossible.

Actual enforcement and optimization depend on the transaction manager, database, driver, and JPA provider.

**Interview trap:** Do not claim that `readOnly = true` always blocks INSERT, UPDATE, and DELETE operations.

---

## 15. What Is Transaction Timeout?

A transaction timeout specifies the maximum allowed time for a transaction, where supported by the transaction manager and resource.

~~~java
@Transactional(timeout = 10)
public void processOrder() {
    // Transactional work
}
~~~

The timeout is specified in seconds.

### Important Points

- It can help prevent excessively long-running transactions.
- Enforcement depends on the transaction manager and underlying resource.
- It does not necessarily interrupt arbitrary Java computation immediately.
- Long-running transactions can hold database connections and locks.
- External network calls inside transactions can extend transaction duration unnecessarily.

**Interview point:** Keep transactions short and avoid performing slow remote calls while holding database transactions open unless the business requirements justify it.

---

## 16. How Does `@Transactional` Work Internally?

Spring's common declarative transaction mechanism is based on AOP proxies and transaction interceptors.

### Simplified Internal Flow

~~~text
Caller
   |
   v
Spring Transaction Proxy
   |
   v
Transaction Interceptor
   |
   v
Transaction Manager
   |
   v
Target Service Method
   |
   v
Commit or Rollback
~~~

### Main Components

**1. Transactional Metadata**

Spring reads attributes such as propagation, isolation, timeout, and rollback rules from `@Transactional`.

**2. Proxy**

The proxy intercepts eligible calls and applies transaction behavior.

**3. Transaction Interceptor**

The interceptor coordinates the transaction around method execution.

**4. Transaction Manager**

The transaction manager begins, commits, or rolls back transactions and coordinates with the relevant resource.

**5. Target Method**

The actual business logic executes within the configured transactional boundary.

### Common Transaction Managers

- `PlatformTransactionManager`: Core Spring abstraction for imperative transaction management.
- `JpaTransactionManager`: Manages JPA transactions for a configured `EntityManagerFactory`.
- `DataSourceTransactionManager`: Manages transactions for a JDBC `DataSource`.
- `JtaTransactionManager`: Integrates with JTA transactions where configured.
- `ReactiveTransactionManager`: Supports reactive transaction management.

The appropriate transaction manager depends on the application and resource model.

---

## 17. What Is Self-Invocation in `@Transactional`?

Self-invocation occurs when a method in a Spring bean calls another method on the same object using `this` or an implicit `this`.

### Example

~~~java
@Service
public class EmployeeService {

    public void process() {
        updateEmployee();
    }

    @Transactional
    public void updateEmployee() {
        // Database operation
    }
}
~~~

When `process()` calls `updateEmployee()` directly on the same object, the call does not pass through the Spring proxy in the usual proxy-based setup.

As a result, the `@Transactional` annotation on `updateEmployee()` may not be applied as expected.

### Why?

Proxy-based transaction management intercepts calls that pass through the proxy. A direct call from one method to another on the target object bypasses that proxy.

### Common Solutions

1. Move the transactional method to a separate Spring-managed service.
2. Call the method through another Spring bean.
3. Use an appropriate AspectJ-based transaction configuration when proxy interception is insufficient.

**Interview one-liner:** Self-invocation bypasses the Spring proxy, so the called method's transactional advice is not applied by default in proxy-based mode.

---

## 18. Which Methods Support `@Transactional`?

In Spring's common proxy-based setup, public methods are the standard recommendation for transactional service methods.

Spring versions and proxy mechanisms have different capabilities for non-public methods, so do not assume every method visibility behaves identically.

### Best Practice

~~~java
@Service
public class EmployeeService {

    @Transactional
    public void updateEmployee(Long id) {
        // Business operation
    }
}
~~~

### Important Points

- The bean must be managed by Spring.
- The invocation must be intercepted by the transaction infrastructure.
- Proxy type and method visibility can affect interception.
- Private methods cannot be overridden by class-based proxies and are not a reliable place to declare transactional boundaries.
- Self-invocation is a common reason transaction behavior appears not to work.

**Interview point:** Put `@Transactional` on externally invoked service methods and ensure the call passes through Spring's transaction infrastructure.

---

## 19. What Happens When a Transactional Method Calls Another Transactional Method?

The result depends on the propagation setting and whether the second call passes through the relevant transaction infrastructure.

### Example

~~~java
@Service
public class OrderService {

    @Transactional
    public void createOrder() {
        saveOrder();
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveOrder() {
        // Database operation
    }
}
~~~

Because `saveOrder()` is called directly from another method in the same object, its `REQUIRES_NEW` annotation is not applied by the proxy-based mechanism in this example.

The method therefore executes within the transaction context established by the outer call, subject to the underlying resource behavior.

### Correct Design

Move the independent transactional operation into a separate Spring-managed bean and call that bean through its proxy.

**Interview trap:** Merely declaring `REQUIRES_NEW` on a method does not guarantee a new transaction if the invocation bypasses transactional interception.

---

## 20. What Is `TransactionTemplate`?

`TransactionTemplate` is Spring's programmatic transaction management abstraction.

It executes a callback within a transaction and handles the transaction lifecycle.

### Example

~~~java
@Service
public class EmployeeService {

    private final TransactionTemplate transactionTemplate;
    private final EmployeeRepository employeeRepository;

    public EmployeeService(
            TransactionTemplate transactionTemplate,
            EmployeeRepository employeeRepository) {
        this.transactionTemplate = transactionTemplate;
        this.employeeRepository = employeeRepository;
    }

    public void createEmployee() {
        transactionTemplate.executeWithoutResult(status -> {
            employeeRepository.save(new Employee());
        });
    }
}
~~~

### When Is It Useful?

- When transaction boundaries need to be controlled dynamically.
- When only a small portion of a method needs transactional behavior.
- When explicit programmatic control improves clarity.
- When avoiding annotation-based proxy limitations is useful.

### Declarative vs Programmatic

| Declarative | Programmatic |
|---|---|
| Uses `@Transactional` and transaction configuration. | Uses `TransactionTemplate` or transaction-manager APIs. |
| Less boilerplate. | More explicit control. |
| Best for common service-layer transactions. | Useful for dynamic or specialized transaction boundaries. |
| Relies on configured interception for annotation-driven behavior. | Explicitly invokes transaction operations through the template or manager. |

---

## 21. What Is `PlatformTransactionManager`?

`PlatformTransactionManager` is Spring's core abstraction for imperative transaction management.

Its main operations are:
- `getTransaction()`
- `commit()`
- `rollback()`

### Simplified Example

~~~java
TransactionStatus status =
    transactionManager.getTransaction(
        new DefaultTransactionDefinition()
    );

try {
    // Database operations

    transactionManager.commit(status);
} catch (RuntimeException ex) {
    transactionManager.rollback(status);
    throw ex;
}
~~~

This illustrates the concept; production code must handle transaction status, rollback failures, checked exceptions, and existing transaction semantics carefully.

**Interview point:** `PlatformTransactionManager` is the abstraction, while implementations such as `JpaTransactionManager` and `DataSourceTransactionManager` provide resource-specific transaction behavior.

---

## 22. What Happens When Multiple Repositories Are Called in One Transaction?

Suppose a service calls two repository methods.

~~~java
@Transactional
public void createOrder() {
    orderRepository.save(new Order());
    inventoryRepository.reduceStock();
}
~~~

If both repositories participate in the same transaction managed by the same compatible transaction manager and resource, their database changes can be committed or rolled back together.

### Important Conditions

- Both operations must participate in the same transaction.
- The resources must be coordinated by the transaction manager.
- Exceptions must trigger rollback or the transaction must be marked rollback-only.
- An independently committed transaction or external side effect is not automatically undone by the outer transaction.

**Interview trap:** `@Transactional` does not automatically create one atomic transaction across unrelated databases or arbitrary external services.

---

## 23. Transaction Management with JPA and Hibernate

In a typical Spring Boot JPA application, Spring coordinates transaction boundaries while Hibernate manages ORM operations through the JPA infrastructure.

### Example

~~~java
@Service
public class EmployeeService {

    private final EmployeeRepository employeeRepository;

    public EmployeeService(EmployeeRepository employeeRepository) {
        this.employeeRepository = employeeRepository;
    }

    @Transactional
    public void updateEmployee(Long id) {
        Employee employee = employeeRepository.findById(id)
            .orElseThrow();

        employee.setSalary(90000);
    }
}
~~~

### What Happens?

1. Spring starts or joins a transaction.
2. The repository retrieves the employee.
3. The entity becomes managed in the persistence context.
4. The salary changes in memory.
5. Hibernate detects the change through dirty checking.
6. SQL is issued when the persistence context is flushed.
7. The transaction commits if no rollback condition applies.

### Important Points

- A managed entity does not generally require an explicit repository `save()` after modification.
- SQL execution may be delayed until flush.
- A transaction can be rolled back even if SQL has already been issued, provided the database transaction is still active and the resource supports rollback.
- Detached entity changes are not automatically tracked by the persistence context.

---

## 24. Checked Exceptions and Rollback: Important Scenario

Consider a service method that updates an employee and then throws a checked exception.

~~~java
@Transactional
public void updateEmployee(Long id) throws Exception {
    Employee employee = employeeRepository.findById(id)
        .orElseThrow();

    employee.setSalary(95000);

    throw new Exception("Processing failed");
}
~~~

Under Spring's default rollback rules, a checked `Exception` does not automatically trigger rollback.

The transaction may commit the salary change despite the exception escaping the method.

### Corrected Version

~~~java
@Transactional(rollbackFor = Exception.class)
public void updateEmployee(Long id) throws Exception {
    Employee employee = employeeRepository.findById(id)
        .orElseThrow();

    employee.setSalary(95000);

    throw new Exception("Processing failed");
}
~~~

Now the configured rollback rule includes checked exceptions of the specified type.

**Interview point:** Always check whether the exception type matches Spring's rollback rules.

---

## 25. Common Transaction Problems

### 1. Transaction Does Not Roll Back

Possible reasons:
- A checked exception was thrown.
- The exception was caught and suppressed.
- The call bypassed the Spring proxy.
- The code executed outside a transaction.
- The transaction was already committed.
- The operation occurred outside the resource coordinated by the transaction manager.

### 2. `LazyInitializationException`

A lazy relationship is accessed after its persistence context is no longer available.

Possible solutions:
- Fetch required associations within the service transaction.
- Use fetch joins or entity graphs.
- Return DTOs built from the required data.
- Avoid relying on an open persistence context during response serialization.

### 3. Unexpected Transaction Commit

Possible reasons:
- An exception was caught inside the method.
- The exception did not match a rollback rule.
- An inner operation committed using an independent transaction.
- The database operation was not part of the transaction being managed.

### 4. Transaction Is Marked Rollback-Only

A participating operation may mark the shared transaction rollback-only after a failure.

Even if the outer method catches the original exception and returns normally, the transaction may still roll back. In some cases, Spring raises `UnexpectedRollbackException` when the outer transaction attempts to commit.

### 5. Long-Running Transactions

Long-running transactions can:
- Hold database connections.
- Retain locks.
- Increase contention.
- Increase the risk of timeouts.
- Reduce connection-pool availability.

Keep transactions focused on database work whenever possible.

---

## 26. Best Practices

- Place transaction boundaries around business operations, usually in the service layer.
- Keep transactions short.
- Understand default rollback rules.
- Use `rollbackFor` when checked exceptions must trigger rollback.
- Avoid catching and suppressing exceptions without considering transaction state.
- Understand propagation before using `REQUIRES_NEW` or `NESTED`.
- Avoid unnecessary remote calls inside database transactions.
- Use read-only transactions for read operations where appropriate.
- Avoid assuming `readOnly = true` universally prevents writes.
- Avoid self-invocation for methods that depend on transactional proxy behavior.
- Do not assume multiple databases or external services share one transaction automatically.
- Use database constraints and appropriate locking or concurrency controls for critical business invariants.
- Use retry logic only for appropriate transient failures and ensure the retried operation is safe.
- Monitor long-running transactions, connection pools, and database lock contention.
- Use an outbox or another reliable messaging pattern when database changes must be coordinated with asynchronous events.

---

## 27. Common Interview Questions and Answers

### Q1. What is `@Transactional`?

It declares transaction behavior for a method or class, allowing Spring to manage transaction boundaries through configured transaction infrastructure.

### Q2. Where should `@Transactional` generally be placed?

On service-layer methods that represent business operations involving database changes.

### Q3. What is the default propagation?

`Propagation.REQUIRED`.

### Q4. What is the default isolation level?

`Isolation.DEFAULT`, which uses the underlying transaction system's default.

### Q5. Does Spring roll back for checked exceptions by default?

No. By default, rollback occurs for unchecked exceptions and `Error`, but not checked exceptions.

### Q6. How do you roll back for checked exceptions?

Configure `rollbackFor`, such as `@Transactional(rollbackFor = Exception.class)`.

### Q7. What is propagation?

It determines how a transactional method behaves when a transaction already exists.

### Q8. What is the difference between `REQUIRED` and `REQUIRES_NEW`?

`REQUIRED` joins an existing transaction or starts one. `REQUIRES_NEW` suspends an existing transaction and starts an independent transaction when supported.

### Q9. What is the difference between `REQUIRES_NEW` and `NESTED`?

`REQUIRES_NEW` uses an independent transaction; `NESTED` commonly uses a savepoint within the existing physical transaction when supported.

### Q10. What is transaction isolation?

It controls how concurrent transactions interact and what data changes they can observe.

### Q11. Does `readOnly = true` prevent all writes?

No. It is generally a hint whose enforcement depends on the transaction manager, provider, database, and configuration.

### Q12. What is self-invocation?

It is a direct call from one method to another method on the same object, bypassing the proxy in the usual proxy-based transaction setup.

### Q13. Why might `@Transactional` not work?

Common reasons include self-invocation, a bean not managed by Spring, a call that bypasses the proxy, or missing transaction infrastructure.

### Q14. Does flushing commit a transaction?

No. Flushing synchronizes pending persistence changes with the database; commit completes the transaction.

### Q15. What is `PlatformTransactionManager`?

It is Spring's core abstraction for imperative transaction management.

### Q16. What is `TransactionTemplate`?

It provides programmatic transaction management using a callback.

### Q17. Can two repository calls share one transaction?

Yes, when both participate in the same transaction and are coordinated by a compatible transaction manager and resource.

### Q18. What is `UnexpectedRollbackException`?

It can occur when an outer transaction attempts to commit even though the transaction has already been marked rollback-only.

### Q19. Does `@Transactional` automatically make external API calls transactional?

No. A database transaction does not automatically roll back remote API calls, email delivery, or other external side effects.

### Q20. Why keep transactions short?

To reduce connection usage, lock duration, contention, and timeout risk.

---

## 28. Common Interview Traps

- **Trap:** `@Transactional` always rolls back for every exception.
  - **Correct:** Default rollback rules apply to unchecked exceptions and errors, not all checked exceptions.

- **Trap:** `REQUIRED` always creates a new transaction.
  - **Correct:** It joins an existing transaction when one is available.

- **Trap:** `REQUIRES_NEW` joins the outer transaction.
  - **Correct:** It suspends the outer transaction and starts an independent transaction when supported.

- **Trap:** `NESTED` is identical to `REQUIRES_NEW`.
  - **Correct:** Nested behavior commonly uses savepoints within one physical transaction.

- **Trap:** `flush()` commits the transaction.
  - **Correct:** Flush synchronizes pending changes; commit completes the transaction.

- **Trap:** Catching an exception automatically rolls back.
  - **Correct:** A caught exception may lead to commit unless the transaction is marked rollback-only or another rollback condition applies.

- **Trap:** Every method annotated with `@Transactional` is always intercepted.
  - **Correct:** Proxy-based behavior depends on the invocation passing through the proxy.

- **Trap:** `readOnly = true` makes writes impossible in every database.
  - **Correct:** It is generally a hint, and enforcement varies.

- **Trap:** An outer transaction can roll back an already committed `REQUIRES_NEW` transaction.
  - **Correct:** An independently committed transaction is not rolled back by the outer transaction.

- **Trap:** `@Transactional` guarantees atomicity across multiple databases and external services.
  - **Correct:** Atomicity depends on the resources and transaction coordination involved.

---

## 29. Quick Revision

- **Transaction:** A group of operations treated as one logical unit.
- **`@Transactional`:** Declares transaction boundaries and attributes.
- **Default Propagation:** `REQUIRED`.
- **Default Isolation:** `DEFAULT`.
- **Default Rollback:** Unchecked exceptions and `Error`.
- **`rollbackFor`:** Configures exceptions that should trigger rollback.
- **`noRollbackFor`:** Configures exceptions that should not trigger rollback.
- **`REQUIRED`:** Join or create a transaction.
- **`REQUIRES_NEW`:** Suspend the existing transaction and create an independent one.
- **`NESTED`:** Nested scope, commonly implemented using a savepoint when supported.
- **Isolation:** Controls concurrent transaction visibility and interaction.
- **Read-Only:** Declares read-only intent and may enable optimizations.
- **Self-Invocation:** Bypasses proxy-based transaction interception.
- **`PlatformTransactionManager`:** Core imperative transaction abstraction.
- **`TransactionTemplate`:** Programmatic transaction management.
- **Flush:** Synchronizes pending persistence changes.
- **Commit:** Completes the transaction.
- **Service Layer:** Common location for business transaction boundaries.
- **Short Transactions:** Reduce lock contention and connection-pool pressure.

### One-Line Interview Summary

Spring's `@Transactional` manages transaction boundaries through transaction infrastructure, with propagation, isolation, and rollback rules controlling how database operations behave.

---

## 30. Related Topics

- [01. JPA, Hibernate, Spring Data JPA & Entities](../07-Spring-Data-JPA/01-JPA-Hibernate-and-Entities.md)
- [02. Entity Relationships & Mapping](../07-Spring-Data-JPA/02-Entity-Relationships-and-Mapping.md)
- [03. Repository Methods, JPQL & Native Queries](../07-Spring-Data-JPA/03-Repositories-and-Queries.md)
- [04. Lazy Loading, Fetching & N+1 Problem](../07-Spring-Data-JPA/04-Lazy-Loading-and-N-plus-one.md)
- [05. Spring Core: IoC & Dependency Injection](../01-Spring-Core/01-IoC-and-Dependency-Injection.md)
- [06. Spring MVC Architecture & Request Flow](../05-Spring-MVC/01-Spring-MVC-Architecture.md)