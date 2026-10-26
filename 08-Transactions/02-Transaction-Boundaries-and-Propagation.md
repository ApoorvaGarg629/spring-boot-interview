# 02. Transaction Boundaries & Propagation

## 1. What Is a Transaction Boundary?

A **transaction boundary** defines where a transaction begins and where it ends.

Within a transaction, multiple database operations are treated as one logical unit of work.

- If all operations succeed, the transaction commits.
- If a qualifying failure occurs, the transaction rolls back according to its rollback rules.

In Spring, transaction boundaries are commonly defined using `@Transactional` or programmatically using `TransactionTemplate`.

~~~java
@Service
public class OrderService {

    @Transactional
    public void placeOrder() {
        orderRepository.save(new Order());
        paymentRepository.save(new Payment());
    }
}
~~~

Both repository operations participate in the same transaction when they use the same transaction manager and transactional resource.

If a qualifying exception causes rollback, both operations are rolled back.

**Interview one-liner:** A transaction boundary defines the scope within which database operations participate in a single transaction.

---

## 2. Where Should Transaction Boundaries Be Defined?

Transaction boundaries should generally be defined at the **service layer**, because a service method represents a business operation that may involve multiple repositories.

### Recommended approach

~~~java
@Service
public class TransferService {

    private final AccountRepository accountRepository;
    private final TransactionRepository transactionRepository;

    public TransferService(
            AccountRepository accountRepository,
            TransactionRepository transactionRepository) {
        this.accountRepository = accountRepository;
        this.transactionRepository = transactionRepository;
    }

    @Transactional
    public void transfer(Long fromId, Long toId, BigDecimal amount) {
        Account from = accountRepository.findById(fromId).orElseThrow();
        Account to = accountRepository.findById(toId).orElseThrow();

        from.debit(amount);
        to.credit(amount);

        transactionRepository.save(
                new Transaction(fromId, toId, amount)
        );
    }
}
~~~

The transfer involves multiple database changes that should succeed or fail together.

### Why not define transactions only in repositories?

- A repository usually handles an individual persistence operation.
- A business operation may call several repositories.
- A service-level transaction can coordinate those operations into one unit of work.

Spring Data repository methods may already have transactional behavior, but the service layer is generally the right place to define the business transaction boundary.

**Interview one-liner:** Define transaction boundaries at the service layer to keep all database operations belonging to one business operation atomic.

---

## 3. Transaction Boundary: Method vs Class Level

`@Transactional` can be applied at the class or method level.

### Method-level annotation

~~~java
@Service
public class PaymentService {

    @Transactional
    public void processPayment() {
        // Transactional business operation
    }

    public void generateReport() {
        // No transaction declared by this annotation
    }
}
~~~

Only the annotated method declares the transaction configuration.

### Class-level annotation

~~~java
@Service
@Transactional
public class PaymentService {

    public void processPayment() {
        // Transactional
    }

    public void refundPayment() {
        // Transactional
    }

    @Transactional(readOnly = true)
    public Payment getPayment(Long id) {
        return paymentRepository.findById(id).orElseThrow();
    }
}
~~~

All eligible public methods inherit the class-level transaction configuration unless a method-level annotation overrides it.

### Important points

- Method-level configuration takes precedence over class-level configuration for that method.
- An annotation does not guarantee that a transaction will start; proxy and invocation rules still apply.
- In Spring's default proxy mode, self-invocation does not trigger a new transaction interceptor.
- Prefer explicit method-level annotations when they make important business transaction boundaries clearer.

**Interview question: Which takes precedence, class-level or method-level `@Transactional`?**

Method-level transaction attributes take precedence over class-level defaults for that method.

---

## 4. What Is Transaction Propagation?

**Transaction propagation** determines how a method behaves when it is called within an existing transaction.

For example, suppose `OrderService` calls `PaymentService`, and both methods are annotated with `@Transactional`.

Propagation determines whether the payment operation:
- Joins the existing order transaction.
- Creates an independent transaction.
- Runs without a transaction.
- Requires a transaction to already exist.

Spring provides seven standard propagation modes:

1. `REQUIRED`
2. `REQUIRES_NEW`
3. `SUPPORTS`
4. `NOT_SUPPORTED`
5. `MANDATORY`
6. `NEVER`
7. `NESTED`

The default propagation mode is **`Propagation.REQUIRED`**.

~~~java
@Transactional(propagation = Propagation.REQUIRED)
public void processOrder() {
    // Transactional work
}
~~~

The annotation above explicitly declares the default behavior.

---

## 5. Propagation.REQUIRED

**Definition:** Join the existing transaction if one exists; otherwise, create a new transaction.

~~~java
@Service
public class OrderService {

    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }

    @Transactional
    public void placeOrder() {
        // Transaction T1 begins
        saveOrder();

        paymentService.processPayment();
    }
}

@Service
public class PaymentService {

    @Transactional(propagation = Propagation.REQUIRED)
    public void processPayment() {
        // Joins T1 when called through the Spring proxy
        savePayment();
    }
}
~~~

### Case 1: No existing transaction

`OrderService.placeOrder()` starts transaction T1.

### Case 2: An existing transaction

`PaymentService.processPayment()` joins T1.

### Result

Both operations normally commit or roll back together.

If an inner operation marks the shared transaction rollback-only, the outer transaction cannot successfully commit. The outer method may receive `UnexpectedRollbackException` when it attempts to commit.

**Interview one-liner:** `REQUIRED` joins the current transaction or creates one if none exists.

**Use case:** Most standard business operations, such as placing an order and saving its payment record.

---

## 6. Propagation.REQUIRES_NEW

**Definition:** Always execute within a new, independent physical transaction, suspending the existing transaction if one exists.

~~~java
@Service
public class OrderService {

    private final AuditService auditService;

    public OrderService(AuditService auditService) {
        this.auditService = auditService;
    }

    @Transactional
    public void placeOrder() {
        saveOrder();

        auditService.recordAttempt();

        throw new RuntimeException("Order failed");
    }
}

@Service
public class AuditService {

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void recordAttempt() {
        auditRepository.save(new AuditRecord("Order attempted"));
    }
}
~~~

### Execution flow

1. `placeOrder()` starts transaction T1.
2. The order is saved within T1.
3. `recordAttempt()` suspends T1 and starts transaction T2.
4. T2 commits independently when `recordAttempt()` completes successfully.
5. T1 resumes.
6. The runtime exception causes T1 to roll back under the default rollback rules.
7. The audit record remains committed if T2 committed successfully.

### Important points

- The inner transaction is independent of the outer transaction.
- Rolling back the outer transaction does not roll back an already committed inner transaction.
- If the inner transaction fails and rolls back, the outer transaction is not automatically guaranteed to roll back; the propagated exception and outer rollback rules determine what happens.
- The suspended outer transaction may still hold database locks or connections.
- `REQUIRES_NEW` can require an additional database connection and can contribute to connection-pool exhaustion under load.

**Interview one-liner:** `REQUIRES_NEW` suspends the current transaction and starts an independent transaction.

**Use cases:**
- Audit records that must survive an outer transaction rollback.
- Independent status updates.
- Separate persistence operations that have deliberately independent commit requirements.

**Important:** Do not use `REQUIRES_NEW` casually. Independent commits can leave data inconsistent with the outer business operation.

---

## 7. REQUIRED vs REQUIRES_NEW

| Feature | `REQUIRED` | `REQUIRES_NEW` |
|---|---|---|
| Existing transaction | Joins it | Suspends it and starts another |
| New transaction | Only if none exists | Always |
| Commit behavior | Shared transaction commits once | Inner transaction commits independently |
| Outer rollback | Inner work normally rolls back with outer work | Already committed inner work remains committed |
| Connection usage | Usually shares the current transaction's resources | May need an additional connection |
| Typical use | Main business operation | Independent audit or status operation |

### Common interview scenario

**Question:** An outer method saves an order and calls an inner method annotated with `REQUIRES_NEW`. The inner method commits, but the outer method fails. What happens?

**Answer:** The outer transaction rolls back, but the inner transaction's committed changes remain.

---

## 8. Propagation.SUPPORTS

**Definition:** Join the existing transaction if one exists; otherwise, execute without a transaction.

~~~java
@Transactional(propagation = Propagation.SUPPORTS)
public Order findOrder(Long id) {
    return orderRepository.findById(id).orElseThrow();
}
~~~

### Behavior

- If called within a transaction, the method participates in it.
- If no transaction exists, Spring does not start a new one solely because of `SUPPORTS`.

The exact resource behavior can depend on the transaction manager and synchronization settings.

**Use case:** An operation that can work either within an existing transaction or without one.

**Interview one-liner:** `SUPPORTS` uses an existing transaction if available; otherwise, it runs without starting one.

---

## 9. Propagation.NOT_SUPPORTED

**Definition:** Execute without a transaction, suspending the current transaction if one exists.

~~~java
@Transactional(propagation = Propagation.NOT_SUPPORTED)
public void generateExternalReport() {
    // Executes without the current transaction
}
~~~

### Behavior

- If an existing transaction is present, Spring suspends it.
- The method executes without an active Spring-managed transaction.
- The suspended transaction resumes after the method finishes.

**Use case:** A lengthy operation that should not execute inside the caller's database transaction.

**Important:** Suspending a transaction does not necessarily release locks or database resources held by that transaction.

**Interview one-liner:** `NOT_SUPPORTED` suspends an existing transaction and executes the method without a transaction.

---

## 10. Propagation.MANDATORY

**Definition:** Execute within an existing transaction; throw an exception if no transaction exists.

~~~java
@Transactional(propagation = Propagation.MANDATORY)
public void updateAccountBalance() {
    // Must be called within an existing transaction
}
~~~

### Behavior

- Existing transaction: joins it.
- No existing transaction: throws `IllegalTransactionStateException` when the transaction interceptor processes the call.

**Use case:** A method that must only be invoked as part of a larger atomic business operation.

**Interview one-liner:** `MANDATORY` requires an existing transaction and fails if none exists.

---

## 11. Propagation.NEVER

**Definition:** Execute without a transaction; throw an exception if a transaction already exists.

~~~java
@Transactional(propagation = Propagation.NEVER)
public void performNonTransactionalOperation() {
    // Must not run within an existing transaction
}
~~~

### Behavior

- No existing transaction: executes without starting one.
- Existing transaction: throws `IllegalTransactionStateException`.

**Use case:** An operation that explicitly must not run within a transaction.

**Interview one-liner:** `NEVER` prohibits execution within an existing transaction.

---

## 12. Propagation.NESTED

**Definition:** Execute within a nested transaction scope, typically implemented using a database savepoint inside the existing physical transaction.

~~~java
@Transactional
public void processOrder() {
    saveOrder();

    try {
        paymentService.processPayment();
    } catch (RuntimeException ex) {
        // Handle nested operation failure
    }

    saveAdditionalOrderDetails();
}

@Transactional(propagation = Propagation.NESTED)
public void processPayment() {
    savePayment();
}
~~~

### Typical behavior when a savepoint is supported

1. The outer transaction starts.
2. The nested operation establishes a savepoint.
3. If the nested operation fails and the failure is handled appropriately, its changes can be rolled back to the savepoint.
4. The outer transaction may continue.
5. The outer transaction ultimately commits or rolls back as a whole.

### Important points

- `NESTED` is not the same as `REQUIRES_NEW`.
- A nested scope typically shares the outer physical transaction and uses a savepoint.
- Rolling back the outer transaction also rolls back the nested changes.
- Savepoint support depends on the transaction manager and underlying resource.
- `JpaTransactionManager` does not provide nested transaction support for JPA entity state by default. JDBC savepoint behavior does not automatically restore the in-memory state of managed JPA entities.

**Interview one-liner:** `NESTED` typically uses a savepoint within the existing transaction, allowing partial rollback while keeping the outer transaction active.

---

## 13. REQUIRED vs REQUIRES_NEW vs NESTED

| Feature | `REQUIRED` | `REQUIRES_NEW` | `NESTED` |
|---|---|---|---|
| Existing transaction | Joins it | Suspends it | Uses a nested scope |
| Physical transaction | Shared | Independent inner transaction | Usually shared |
| Savepoint | No separate savepoint by default | Not required | Typically used |
| Inner rollback | Can mark the shared transaction rollback-only | Rolls back the independent inner transaction | Can roll back to the savepoint |
| Outer rollback | Rolls back shared work | Does not undo a committed inner transaction | Rolls back all work in the outer transaction |
| Support considerations | Broadly supported | Depends on transaction manager capabilities | Requires savepoint support and compatible transaction manager |

### Easy way to remember

- `REQUIRED`: Same transaction.
- `REQUIRES_NEW`: New independent transaction.
- `NESTED`: Same physical transaction with a savepoint, where supported.

---

## 14. Propagation Modes Quick Comparison

| Propagation | Existing transaction? | No existing transaction? |
|---|---|---|
| `REQUIRED` | Join | Start a new transaction |
| `REQUIRES_NEW` | Suspend and start a new transaction | Start a new transaction |
| `SUPPORTS` | Join | Run without starting a transaction |
| `NOT_SUPPORTED` | Suspend and run without a transaction | Run without a transaction |
| `MANDATORY` | Join | Throw an exception |
| `NEVER` | Throw an exception | Run without a transaction |
| `NESTED` | Create a nested scope, typically using a savepoint | Start a transaction when supported |

For `NESTED`, exact behavior depends on the transaction manager. When no transaction exists, it generally behaves like a transaction boundary for the operation if supported.

---

## 15. How Propagation Works with Spring Proxies

Spring's declarative transaction management usually works through a proxy.

When a Spring-managed bean calls a transactional method through the proxy, the transaction interceptor applies the method's transaction attributes, including propagation.

### Example: Separate beans

~~~java
@Service
public class CheckoutService {

    private final PaymentService paymentService;

    public CheckoutService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }

    @Transactional
    public void checkout() {
        paymentService.charge();
    }
}

@Service
public class PaymentService {

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void charge() {
        // New transaction when invoked through the Spring proxy
    }
}
~~~

The call from `CheckoutService` to the injected `PaymentService` normally passes through the proxy, allowing `REQUIRES_NEW` to take effect.

### Self-invocation problem

~~~java
@Service
public class ReportService {

    @Transactional
    public void generateReport() {
        saveReport();
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveReport() {
        // Self-invocation does not trigger a new transaction interceptor
    }
}
~~~

Because `generateReport()` calls `saveReport()` directly on the same object, the call bypasses the proxy in Spring's default proxy mode.

Consequently, the `REQUIRES_NEW` configuration on `saveReport()` is not applied through that self-invocation.

### How to address it

- Move the independently transactional operation to a separate Spring bean.
- Restructure the application so the call passes through the appropriate Spring proxy.
- Use AspectJ-based transaction weaving when its capabilities and complexity are justified.

**Interview one-liner:** In Spring's default proxy mode, transactional propagation is applied when a method is invoked through the transaction proxy, not during ordinary self-invocation.

---

## 16. Rollback Rules and Propagation

Propagation determines how transactions are joined or created. Rollback rules determine which failures cause a transaction to roll back.

These are related but separate concepts.

### Default rollback behavior

By default, Spring rolls back on:
- `RuntimeException`
- `Error`

By default, Spring does not automatically roll back on checked exceptions.

~~~java
@Transactional
public void processOrder() throws IOException {
    orderRepository.save(new Order());

    throw new IOException("File processing failed");
}
~~~

The checked exception does not trigger rollback by default. If it escapes the transactional method, the transaction may commit despite the exception.

### Roll back on a checked exception

~~~java
@Transactional(rollbackFor = IOException.class)
public void processOrder() throws IOException {
    orderRepository.save(new Order());

    throw new IOException("File processing failed");
}
~~~

### Important propagation interaction

Suppose an inner method uses `REQUIRED` and throws a qualifying exception that marks the shared transaction rollback-only. Even if the outer method catches the exception, the shared transaction may still be unable to commit.

If the outer method attempts to commit a transaction already marked rollback-only, Spring may throw `UnexpectedRollbackException`.

With `REQUIRES_NEW`, the inner transaction has its own rollback outcome, separate from the outer transaction.

**Interview one-liner:** Propagation controls transaction participation; rollback rules determine whether a transaction is rolled back for a particular failure.

---

## 17. Common Transaction Propagation Scenarios

### Scenario 1: Both methods use REQUIRED

- Outer method starts T1.
- Inner method joins T1.
- Inner failure marks T1 rollback-only when the applicable rollback rules trigger.
- The transaction's final outcome is shared.

**Result:** Both operations normally commit or roll back together.

### Scenario 2: Outer REQUIRED, inner REQUIRES_NEW

- Outer method starts T1.
- Inner method suspends T1 and starts T2.
- T2 commits.
- T1 resumes and later rolls back.

**Result:** Inner committed changes remain, even though outer changes roll back.

### Scenario 3: Outer REQUIRED, inner SUPPORTS

- Outer method starts T1.
- Inner method joins T1.

**Result:** The inner method participates in T1 because a transaction already exists.

### Scenario 4: Outer REQUIRED, inner NOT_SUPPORTED

- Outer method starts T1.
- Inner method suspends T1.
- Inner method executes without a transaction.
- T1 resumes after the inner method returns.

**Result:** The inner method is not part of T1, but suspending T1 does not guarantee that its database locks or resources are released.

### Scenario 5: Outer REQUIRED, inner MANDATORY

- Outer method starts T1.
- Inner method joins T1.

**Result:** The call succeeds with respect to propagation because a transaction exists.

### Scenario 6: Outer REQUIRED, inner NEVER

- Outer method starts T1.
- Inner method is invoked while T1 exists.

**Result:** Spring throws `IllegalTransactionStateException` when the invocation passes through the transactional interceptor.

### Scenario 7: Outer REQUIRED, inner NESTED

- Outer method starts T1.
- Inner method creates a savepoint when supported.
- Inner work rolls back to the savepoint if the nested rollback is triggered.
- The outer transaction can continue if the failure is handled and the transaction remains usable.

**Result:** Partial rollback is possible without creating an independent physical transaction.

---

## 18. Common Interview Questions

### Q1. What is transaction propagation?

It defines how a method behaves when a transaction already exists, such as joining it, suspending it, or requiring one to be present.

### Q2. What is the default propagation mode in Spring?

`Propagation.REQUIRED`.

### Q3. What is the difference between REQUIRED and REQUIRES_NEW?

`REQUIRED` joins an existing transaction, whereas `REQUIRES_NEW` suspends it and starts an independent transaction.

### Q4. What is the difference between REQUIRES_NEW and NESTED?

`REQUIRES_NEW` uses an independent physical transaction, whereas `NESTED` typically uses a savepoint within the existing physical transaction.

### Q5. Does REQUIRES_NEW commit independently?

Yes. Its transaction can commit independently of the suspended outer transaction.

### Q6. If the outer transaction rolls back, does the REQUIRES_NEW transaction roll back?

No. Changes committed by the independent inner transaction remain committed.

### Q7. Can REQUIRED cause UnexpectedRollbackException?

Yes. For example, an inner method can mark the shared transaction rollback-only, after which the outer method attempts to commit it.

### Q8. What does SUPPORTS do?

It joins an existing transaction if available; otherwise, it does not start one.

### Q9. What is the difference between SUPPORTS and REQUIRED?

`SUPPORTS` does not create a transaction when none exists; `REQUIRED` does.

### Q10. What does NOT_SUPPORTED do?

It suspends an existing transaction and runs without a transaction.

### Q11. What does MANDATORY do?

It requires an existing transaction and throws an exception if none exists.

### Q12. What does NEVER do?

It throws an exception if a transaction exists and otherwise runs without starting one.

### Q13. What is NESTED propagation used for?

It allows a nested operation to roll back to a savepoint while potentially allowing the outer transaction to continue.

### Q14. Does NESTED always work with JPA?

No. Support depends on the transaction manager and underlying resource. JPA entity state is not automatically restored merely because a JDBC savepoint is rolled back.

### Q15. Does self-invocation apply REQUIRES_NEW?

Not in Spring's default proxy-based transaction mode, because the internal call bypasses the proxy.

### Q16. Can a private method reliably establish a transaction through the default proxy?

No. Private methods cannot be intercepted as transactional entry points by Spring's standard proxy-based approach.

### Q17. Can an inner REQUIRED method roll back the outer transaction?

Yes. Since both participate in the same transaction, a qualifying failure can mark the shared transaction rollback-only.

### Q18. Does catching an exception always prevent rollback?

No. The transaction may already be marked rollback-only, and rollback rules may also be configured independently of the exception handling in the caller.

### Q19. Does propagation guarantee that all database resources participate in one transaction?

No. Atomic coordination depends on the transaction manager, resource configuration, and whether operations use the same transactional resource. Multiple independent resources may require distributed transaction coordination.

### Q20. Is propagation the same as isolation?

No. Propagation determines how transactions are joined or created; isolation controls how concurrent transactions can observe each other's data.

---

## 19. Common Interview Traps

- `REQUIRED` is the default propagation mode.
- `REQUIRES_NEW` does not mean nested transaction; it creates an independent transaction.
- `NESTED` is typically savepoint-based and requires appropriate support.
- An inner `REQUIRED` method shares the outer transaction's outcome.
- An inner transaction marked rollback-only may cause `UnexpectedRollbackException` when the outer method attempts to commit.
- Catching an exception does not necessarily clear a rollback-only marker.
- `SUPPORTS` does not create a transaction when none exists.
- `NOT_SUPPORTED` suspends the transaction but does not guarantee that locks are released.
- `MANDATORY` requires a transaction.
- `NEVER` prohibits an existing transaction.
- Transaction annotations may not take effect during self-invocation.
- A transaction boundary does not automatically include remote HTTP calls, message brokers, or other independent systems.
- An independently committed inner transaction can leave data behind if the outer operation fails.
- Propagation settings are applied by transaction infrastructure; merely placing an annotation on a method does not guarantee that the annotation is intercepted.

---

## 20. Best Practices

- Define transaction boundaries around business operations, usually at the service layer.
- Use `REQUIRED` for operations that should succeed or fail together.
- Use `REQUIRES_NEW` only when independent commit or rollback behavior is explicitly required.
- Use `NESTED` only after verifying transaction-manager and savepoint support.
- Keep transactions short to reduce lock duration and resource consumption.
- Avoid long-running external API calls inside database transactions where possible.
- Understand rollback rules before catching exceptions inside transactional methods.
- Avoid self-invocation when different propagation behavior is required.
- Test rollback behavior against the actual database and transaction manager.
- Do not assume a database transaction makes distributed operations atomic.
- For reliable messaging or event publication, consider patterns such as the transactional outbox instead of assuming a database transaction can atomically commit a remote message broker operation.

---

## 21. Quick Revision

- **Transaction boundary:** Defines the scope of a transaction.
- **Propagation:** Defines how a method interacts with an existing transaction.
- **REQUIRED:** Join existing or create new.
- **REQUIRES_NEW:** Suspend existing and create independent transaction.
- **SUPPORTS:** Join existing or run without starting one.
- **NOT_SUPPORTED:** Suspend existing and run without a transaction.
- **MANDATORY:** Require an existing transaction.
- **NEVER:** Fail if a transaction exists.
- **NESTED:** Use a nested scope, typically with a savepoint.
- **Default propagation:** `REQUIRED`.
- **Self-invocation:** Bypasses the transaction proxy in default proxy mode.
- **UnexpectedRollbackException:** Can occur when an outer transaction tries to commit after being marked rollback-only.
- **Propagation vs isolation:** Transaction participation vs concurrent data visibility.
- **Service layer:** Usually the best place to define business transaction boundaries.

---

## 22. Related Topics

- [01. Transaction Management & @Transactional](01-Transactional-and-Transaction-Management.md)
- [03. Transaction Isolation & Concurrency Issues](03-Transaction-Isolation-and-Concurrency.md)
- [04. Spring Data JPA & Hibernate](../07-Spring-Data-JPA-and-Hibernate/01-Spring-Data-JPA-and-Hibernate.md)
- [05. Exception Handling](../06-Exception-Handling/01-Exception-Handling.md)
- [06. Spring Core: IoC, DI & Proxies](../01-Spring-Core/01-IoC-and-Dependency-Injection.md)