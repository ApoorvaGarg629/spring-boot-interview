# IoC & Dependency Injection

## What is IoC?

**IoC (Inversion of Control)** means transferring the responsibility of creating, configuring, and managing objects from the application code to the **Spring IoC Container**.

Without Spring, the developer explicitly creates dependencies using `new`.

~~~java
class OrderService {
    private PaymentService paymentService = new PaymentService();
}
~~~

Here, `OrderService` controls the creation of `PaymentService`.

With Spring, the container creates and injects the dependency.

~~~java
@Service
class OrderService {

    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
~~~

Spring creates `PaymentService`, creates `OrderService`, and injects `PaymentService` into it.

### Interview Definition

> **IoC is a design principle where the control of object creation, configuration, and dependency management is transferred from application code to a container such as the Spring IoC Container.**

---

## Why is it called Inversion of Control?

Normally:

~~~text
Application → creates → Dependency
~~~

With Spring:

~~~text
Spring Container → creates → Dependency
                  → injects → Dependency into Application
~~~

The control over object creation is therefore **inverted**.

---

# What is Dependency Injection?

**Dependency Injection (DI)** is the mechanism through which an object's required dependencies are provided by an external source instead of the object creating them itself.

Example:

~~~java
@Service
class OrderService {

    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
~~~

`OrderService` depends on `PaymentService`.

Instead of doing:

~~~java
PaymentService paymentService = new PaymentService();
~~~

Spring injects it through the constructor.

### Interview Definition

> **Dependency Injection is a design pattern/mechanism where an object's dependencies are supplied from outside rather than being created by the object itself.**

---

# IoC vs Dependency Injection

| IoC | Dependency Injection |
|---|---|
| Design principle | Mechanism/pattern used to implement IoC |
| Broad concept | Specific way of providing dependencies |
| Transfers control to container | Injects dependencies into objects |
| Spring Container implements IoC | Constructor, setter, and field injection implement DI |

### Interview One-Liner

> **IoC is the principle of giving control to the Spring container, while DI is the mechanism Spring uses to provide dependencies to objects.**

---

# What is a Dependency?

A dependency is an object that another object requires to perform its work.

Example:

~~~java
@Service
class OrderService {

    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
~~~

Here:

- `OrderService` is the dependent object.
- `PaymentService` is its dependency.
- Spring injects `PaymentService` into `OrderService`.

---

# Why do we use Dependency Injection?

DI provides:

- Loose coupling
- Better testability
- Easier maintenance
- Easier replacement of implementations
- Better separation of responsibilities
- Better adherence to SOLID principles
- Easier configuration
- Easier mocking during unit testing

---

# Tight Coupling vs Loose Coupling

## Tight Coupling

~~~java
class OrderService {

    private PaymentService paymentService = new PaymentService();
}
~~~

`OrderService` directly creates `PaymentService`.

Problems:

- Difficult to replace implementation
- Difficult to unit test
- Strong dependency on concrete class
- Changes can affect multiple classes

---

## Loose Coupling

~~~java
interface PaymentService {
    void pay();
}
~~~

~~~java
@Service
class OrderService {

    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
~~~

Now `OrderService` depends on an abstraction rather than creating a concrete implementation.

---

# Spring IoC Container

The **Spring IoC Container** is responsible for:

- Creating objects
- Managing objects
- Configuring objects
- Injecting dependencies
- Managing bean lifecycle
- Managing bean scopes
- Applying Spring features such as AOP proxies

The objects managed by the Spring container are called **Spring Beans**.

---

# Main Spring IoC Containers

The two important container interfaces are:

- `BeanFactory`
- `ApplicationContext`

## BeanFactory

Basic IoC container.

Provides:

- Bean creation
- Dependency injection
- Basic lifecycle management

## ApplicationContext

A more advanced container built on top of `BeanFactory`.

Provides:

- Dependency Injection
- Bean lifecycle
- Events
- Internationalization
- Resource loading
- Environment/profile support
- Integration with AOP
- Automatic detection of components

### Interview Question

**BeanFactory vs ApplicationContext?**

### Answer

> `BeanFactory` provides basic IoC functionality, while `ApplicationContext` extends it with enterprise-level features such as events, internationalization, resource loading, environment support, and easier integration with Spring features.

In Spring Boot applications, `ApplicationContext` is normally used.

---

# Types of Dependency Injection

Spring mainly supports:

- Constructor Injection
- Setter Injection
- Field Injection

---

# Constructor Injection

Dependencies are provided through the constructor.

~~~java
@Service
class OrderService {

    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
~~~

### Advantages

- Recommended approach
- Dependencies can be `final`
- Object can be immutable
- Required dependencies are guaranteed during construction
- Easy unit testing
- Makes dependencies explicit
- Prevents partially initialized objects

### Interview One-Liner

> **Constructor injection is preferred because required dependencies are explicit, can be final, and the object cannot be created without them.**

---

# Setter Injection

Dependencies are provided through setter methods.

~~~java
@Service
class OrderService {

    private PaymentService paymentService;

    @Autowired
    public void setPaymentService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
~~~

Useful when:

- Dependency is optional
- Dependency may change after object creation
- Setter-based configuration is appropriate

### Disadvantages

- Object can exist without the dependency
- Dependency is not immutable
- Easier to create partially initialized objects

---

# Field Injection

Dependency is injected directly into a field.

~~~java
@Service
class OrderService {

    @Autowired
    private PaymentService paymentService;
}
~~~

### Problems

- Dependencies are hidden
- Difficult to unit test without Spring
- Cannot easily make dependency `final`
- Encourages mutable fields
- Can result in partially initialized objects
- Makes class dependencies less explicit

### Interview Question

**Why is constructor injection preferred over field injection?**

### Answer

> Constructor injection makes dependencies explicit, supports immutability, improves testability, and ensures required dependencies are available when the object is created.

---

# Is @Autowired Required on a Constructor?

If a class has **only one constructor**, `@Autowired` is not required.

~~~java
@Service
class OrderService {

    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
~~~

Spring automatically uses the constructor.

---

# Multiple Constructors

If a class has multiple constructors, Spring needs to know which constructor should be used.

~~~java
@Service
class OrderService {

    private final PaymentService paymentService;

    public OrderService() {
        this.paymentService = null;
    }

    @Autowired
    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
~~~

`@Autowired` tells Spring which constructor to use.

---

# Dependency Injection Through Interfaces

A recommended approach is to depend on an interface.

~~~java
interface PaymentService {
    void pay();
}
~~~

Implementation:

~~~java
@Service
class CreditCardPaymentService implements PaymentService {

    @Override
    public void pay() {
        System.out.println("Credit card payment");
    }
}
~~~

Consumer:

~~~java
@Service
class OrderService {

    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
~~~

This gives loose coupling.

---

# Multiple Implementations of an Interface

Suppose we have:

~~~java
interface PaymentService {
    void pay();
}
~~~

~~~java
@Service
class CreditCardPaymentService implements PaymentService {
}
~~~

~~~java
@Service
class UpiPaymentService implements PaymentService {
}
~~~

Now Spring sees two beans implementing `PaymentService`.

If we inject:

~~~java
public OrderService(PaymentService paymentService) {
    this.paymentService = paymentService;
}
~~~

Spring does not know which implementation to choose.

This can result in:

~~~text
NoUniqueBeanDefinitionException
~~~

---

# Resolving Multiple Implementations with @Primary

Mark one implementation as the default.

~~~java
@Primary
@Service
class CreditCardPaymentService implements PaymentService {
}
~~~

Now when Spring requires a `PaymentService`, it selects the `@Primary` bean unless another bean is explicitly selected.

### Interview One-Liner

> `@Primary` defines the default bean when multiple beans of the same type are available.

---

# Resolving Multiple Implementations with @Qualifier

Use `@Qualifier` when you want a specific implementation.

~~~java
@Service
class OrderService {

    private final PaymentService paymentService;

    public OrderService(
            @Qualifier("upiPaymentService") PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
~~~

Implementation:

~~~java
@Service
class UpiPaymentService implements PaymentService {
}
~~~

Default bean name is generally the class name with the first letter converted to lowercase.

Example:

~~~text
UpiPaymentService
       ↓
upiPaymentService
~~~

### Interview One-Liner

> `@Qualifier` explicitly tells Spring which bean to inject when multiple candidates exist.

---

# @Primary vs @Qualifier

| @Primary | @Qualifier |
|---|---|
| Defines default bean | Selects a specific bean |
| Used when no specific bean is selected | Used for explicit selection |
| Can be used as fallback | More precise |

If both are present, an explicit `@Qualifier` takes precedence.

---

# Injecting Multiple Implementations

Spring can inject all implementations into a collection.

## List

~~~java
@Service
class PaymentProcessor {

    private final List<PaymentService> paymentServices;

    public PaymentProcessor(List<PaymentService> paymentServices) {
        this.paymentServices = paymentServices;
    }
}
~~~

## Set

~~~java
public PaymentProcessor(Set<PaymentService> paymentServices) {
    this.paymentServices = paymentServices;
}
~~~

## Map

~~~java
public PaymentProcessor(Map<String, PaymentService> paymentServices) {
    this.paymentServices = paymentServices;
}
~~~

For a `Map`, bean names are generally used as keys.

---

# How @Autowired Works

When Spring sees:

~~~java
@Autowired
private PaymentService paymentService;
~~~

or constructor injection:

~~~java
public OrderService(PaymentService paymentService) {
}
~~~

Spring performs dependency resolution approximately as follows:

1. Determine the required dependency type.
2. Search the application context for matching beans.
3. If exactly one candidate exists, inject it.
4. If multiple candidates exist, consider `@Primary`.
5. If `@Qualifier` is present, use the qualified bean.
6. If no suitable bean exists, application startup can fail.

---

# Required vs Optional Dependencies

By default, Spring expects dependencies to be available.

If a required dependency cannot be resolved, the application generally fails during startup.

For optional dependencies, alternatives include:

~~~java
Optional<T>

public OrderService(Optional<PaymentService> paymentService) {
    this.paymentService = paymentService;
}
~~~

~~~java
ObjectProvider<T>

public OrderService(ObjectProvider<PaymentService> paymentService) {
    this.paymentService = paymentService;
}
~~~

`ObjectProvider` is useful when you need lazy or optional access to a dependency.

---

# Dependency Lookup vs Dependency Injection

## Dependency Injection

The container provides the dependency.

~~~java
public OrderService(PaymentService paymentService) {
    this.paymentService = paymentService;
}
~~~

## Dependency Lookup

The application asks the container for the dependency.

Conceptually:

~~~java
PaymentService paymentService =
        applicationContext.getBean(PaymentService.class);
~~~

### Difference

> **DI pushes dependencies into the object, while dependency lookup requires the object/application to ask the container for the dependency.**

DI is generally preferred because it reduces coupling to the Spring container.

---

# Dependency Injection vs Dependency Lookup

| Dependency Injection | Dependency Lookup |
|---|---|
| Container provides dependency | Application requests dependency |
| Less coupled to Spring | More coupled to Spring |
| Better testability | More difficult to test |
| Preferred approach | Used in specific cases |

---

# Responsibilities of the IoC Container

The Spring container manages the complete lifecycle of Spring beans.

Important responsibilities:

- Bean creation
- Dependency resolution
- Dependency injection
- Bean initialization
- Bean destruction
- Bean scope management
- Configuration
- Lifecycle callbacks
- Proxy creation
- AOP integration

---

# Simplified Spring IoC Startup Flow

A simplified flow is:

~~~text
Spring Boot starts
        ↓
ApplicationContext is created
        ↓
Component scanning/configuration occurs
        ↓
Bean definitions are registered
        ↓
Dependencies are resolved
        ↓
Beans are instantiated
        ↓
Dependencies are injected
        ↓
Bean initialization occurs
        ↓
Application is ready
~~~

This is a simplified interview-level representation, not the exact internal implementation sequence.

---

# What is a BeanDefinition?

A `BeanDefinition` is metadata that tells Spring how a bean should be created and managed.

It can contain information such as:

- Bean class
- Scope
- Dependencies
- Lazy initialization
- Initialization method
- Destruction method
- Other configuration metadata

Spring uses this metadata when creating and managing beans.

---

# How Does Spring Know Which Classes Are Beans?

Common ways include:

## Component Scanning

~~~java
@Component
class PaymentService {
}
~~~

Spring detects the class during component scanning.

Common stereotype annotations include:

~~~text
@Component
@Service
@Repository
@Controller
@RestController
~~~

These are commonly detected through component scanning.

---

# @Component

Generic Spring-managed component.

~~~java
@Component
class PaymentService {
}
~~~

---

# @Service

Typically used for service/business logic.

~~~java
@Service
class OrderService {
}
~~~

`@Service` is a specialized form of `@Component`.

---

# @Repository

Typically used for persistence/data-access components.

~~~java
@Repository
class OrderRepository {
}
~~~

`@Repository` is also a specialized form of `@Component`.

It additionally participates in Spring's exception translation mechanism for persistence exceptions.

---

# @Controller

Used for Spring MVC controllers.

~~~java
@Controller
class OrderController {
}
~~~

---

# @RestController

Used commonly for REST APIs.

Conceptually combines:

~~~java
@Controller
@ResponseBody
~~~

---

# @Bean

A bean can also be explicitly registered using a configuration class.

~~~java
@Configuration
class AppConfig {

    @Bean
    PaymentService paymentService() {
        return new PaymentService();
    }
}
~~~

Here, Spring manages the returned object as a bean.

`@Bean` is especially useful when:

- You need to configure a third-party class
- You cannot modify the class to add `@Component`
- Bean creation requires custom configuration

---

# @Component vs @Bean

| @Component | @Bean |
|---|---|
| Usually placed on class | Placed on method |
| Detected through component scanning | Explicitly declared |
| Good for application classes | Good for custom/third-party configuration |
| Spring creates the component | Method provides the object |

---

# Dependency Injection with @Bean

~~~java
@Configuration
class AppConfig {

    @Bean
    PaymentService paymentService() {
        return new PaymentService();
    }

    @Bean
    OrderService orderService(PaymentService paymentService) {
        return new OrderService(paymentService);
    }
}
~~~

Spring resolves `PaymentService` and supplies it to the `orderService` bean method.

---

# Interface-Based DI and Dependency Inversion Principle

Consider:

~~~java
interface NotificationService {
    void send(String message);
}
~~~

Implementation:

~~~java
@Service
class EmailNotificationService implements NotificationService {
}
~~~

Consumer:

~~~java
@Service
class OrderService {

    private final NotificationService notificationService;

    public OrderService(NotificationService notificationService) {
        this.notificationService = notificationService;
    }
}
~~~

`OrderService` depends on the abstraction rather than the concrete implementation.

This supports the **Dependency Inversion Principle (DIP)**.

### DIP Interview Definition

> High-level modules should not depend directly on low-level concrete implementations; both should depend on abstractions.

Spring DI makes implementing this principle easier.

---

# DI and SOLID

Dependency Injection is strongly related to the **Dependency Inversion Principle**.

DI also helps with:

- Single Responsibility
- Open/Closed Principle
- Testability
- Loose coupling

However:

> **DI itself is not SOLID; it is a technique that helps implement the Dependency Inversion Principle and achieve loose coupling.**

---

# Circular Dependency

A circular dependency occurs when two or more beans depend on each other.

Example:

~~~text
A → B
B → A
~~~

Example:

~~~java
@Service
class A {

    private final B b;

    A(B b) {
        this.b = b;
    }
}
~~~

~~~java
@Service
class B {

    private final A a;

    B(A a) {
        this.a = a;
    }
}
~~~

This creates a circular dependency.

With constructor injection, Spring cannot create either bean because each requires the other during construction.

---

# How to Resolve Circular Dependency?

The preferred solution is usually to **redesign the classes**.

Possible approaches:

- Remove unnecessary dependency
- Extract shared functionality into another service
- Change the dependency direction
- Introduce an appropriate abstraction
- Use events where appropriate

`@Lazy` can sometimes break the immediate creation cycle, but it should generally be treated as a workaround rather than the first design choice.

---

# @Lazy

`@Lazy` tells Spring to initialize a bean lazily instead of eagerly initializing it during application startup.

Example:

~~~java
@Lazy
@Service
class PaymentService {
}
~~~

Or:

~~~java
public OrderService(@Lazy PaymentService paymentService) {
    this.paymentService = paymentService;
}
~~~

Useful when:

- Bean initialization is expensive
- Bean is rarely needed
- You need to defer creation
- Certain dependency cycles need to be broken

### Interview One-Liner

> `@Lazy` delays bean initialization until the bean is actually needed.

---

# DI and Bean Scope

Dependency injection also interacts with bean scopes.

Common scopes:

- Singleton
- Prototype
- Request
- Session
- Application
- WebSocket

The default Spring bean scope is **singleton**.

---

# Singleton Dependency Injected into Singleton

If both are singleton beans, one shared dependency instance is normally injected.

~~~text
OrderService
     ↓
PaymentService
     ↓
same singleton instance
~~~

---

# Prototype Dependency Injected into Singleton

Consider:

~~~java
@Service
class OrderService {

    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
~~~

If `PaymentService` is prototype-scoped but `OrderService` is singleton-scoped, the prototype dependency is normally created when the singleton is created.

The singleton does not automatically receive a new prototype instance every time its method is called.

This is an important interview scenario.

---

# ObjectProvider for Prototype Dependencies

`ObjectProvider` can be used when a new dependency instance is needed on demand.

~~~java
@Service
class OrderService {

    private final ObjectProvider<PaymentService> paymentServiceProvider;

    public OrderService(ObjectProvider<PaymentService> paymentServiceProvider) {
        this.paymentServiceProvider = paymentServiceProvider;
    }

    public void process() {
        PaymentService paymentService =
                paymentServiceProvider.getObject();

        paymentService.process();
    }
}
~~~

This allows Spring to obtain an appropriate instance when requested.

---

# Dependency Injection with Generics

Spring can use generic type information during dependency resolution.

Example:

~~~java
interface Processor<T> {
    void process(T value);
}
~~~

~~~java
@Component
class StringProcessor implements Processor<String> {

    @Override
    public void process(String value) {
    }
}
~~~

~~~java
@Component
class IntegerProcessor implements Processor<Integer> {

    @Override
    public void process(Integer value) {
    }
}
~~~

Spring can use generic type information to distinguish suitable candidates in supported injection scenarios.

---

# Proxies and Dependency Injection

Spring often injects a **proxy** instead of the original target object when framework features require interception.

Examples:

- `@Transactional`
- `@Async`
- AOP
- Method-level security
- Caching

For example:

~~~java
@Service
class PaymentService {

    @Transactional
    public void processPayment() {
    }
}
~~~

Spring may create a proxy around `PaymentService`.

Conceptually:

~~~text
Caller
   ↓
Spring Proxy
   ↓
Transaction handling
   ↓
Actual PaymentService
~~~

This allows Spring to execute additional behavior around method calls.

---

# Does Spring Use Reflection for Dependency Injection?

Spring heavily uses reflection and other runtime mechanisms to inspect classes, constructors, fields, methods, annotations, and metadata.

For example, Spring can discover:

~~~java
@Autowired
private PaymentService paymentService;
~~~

and resolve and inject the dependency.

However, modern Spring also uses optimized infrastructure, generated code, proxies, and other mechanisms depending on the feature and runtime configuration.

### Interview Answer

> **Spring uses reflection and metadata processing extensively to discover and inject dependencies, while also using proxies and other infrastructure mechanisms for framework features.**

---

# What Happens if a Dependency Is Missing?

Suppose:

~~~java
@Service
class OrderService {

    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
~~~

But there is no suitable `PaymentService` bean.

Spring cannot resolve the dependency.

The application generally fails during startup with an error such as:

~~~text
NoSuchBeanDefinitionException
~~~

or another dependency-resolution-related exception depending on the situation.

### Interview One-Liner

> If a required dependency cannot be resolved, Spring generally fails application startup rather than creating an incomplete bean.

---

# Common DI Interview Scenarios

## Scenario: Two implementations of the same interface

Problem:

~~~text
PaymentService
    ├── CreditCardPaymentService
    └── UpiPaymentService
~~~

Solution:

- `@Primary`
- `@Qualifier`
- Inject a collection of implementations

---

## Scenario: Dependency is optional

Use:

- `Optional<T>`
- `ObjectProvider<T>`
- Appropriate nullable/configuration design

---

## Scenario: Dependency should be immutable

Use constructor injection and `final` fields.

~~~java
private final PaymentService paymentService;
~~~

---

## Scenario: Object requires many dependencies

Constructor injection makes the large dependency list visible.

A very large constructor can also indicate that the class has too many responsibilities.

Possible solution:

- Split responsibilities
- Extract services
- Apply Single Responsibility Principle

Do not solve a large dependency list merely by switching to field injection.

---

## Scenario: Two beans have the same type

Use:

- `@Qualifier`
- `@Primary`

---

## Scenario: Need all implementations

Inject:

~~~java
List<PaymentService>
~~~

or:

~~~java
Set<PaymentService>
~~~

or:

~~~java
Map<String, PaymentService>
~~~

---

## Scenario: Need a new prototype object for every operation

Use an appropriate lookup mechanism such as:

~~~java
ObjectProvider<T>
~~~

rather than directly injecting the prototype into a singleton and expecting a new instance on every method call.

---

## Scenario: Circular dependency exists

First consider redesigning the dependency relationship.

Possible solutions:

- Remove unnecessary dependency
- Extract common functionality
- Change dependency direction
- Use events
- Use `@Lazy` only when appropriate

---

# Important Interview Questions

## What is IoC?

> IoC is the principle of transferring object creation and dependency management from application code to a container such as the Spring IoC Container.

---

## What is Dependency Injection?

> DI is the mechanism of providing an object's dependencies from outside instead of the object creating them itself.

---

## Difference between IoC and DI?

> IoC is the broader principle of transferring control to the container, while DI is a mechanism used to implement IoC.

---

## What are the types of DI?

> Constructor injection, setter injection, and field injection.

---

## Which type of DI is preferred?

> Constructor injection is generally preferred for required dependencies.

---

## Why is constructor injection preferred?

> It makes dependencies explicit, supports immutability, improves testability, and ensures required dependencies are available during object creation.

---

## Why is field injection generally discouraged?

> It hides dependencies, makes unit testing harder, prevents easy immutability, and allows partially initialized objects.

---

## Is @Autowired required on a single constructor?

> No. If the class has a single constructor, Spring can use it automatically.

---

## What happens if multiple beans match a dependency?

> Spring can use `@Primary` or `@Qualifier` to resolve the ambiguity; otherwise dependency resolution can fail.

---

## @Primary vs @Qualifier?

> `@Primary` specifies the default candidate, while `@Qualifier` explicitly selects a particular bean.

---

## Can Spring inject multiple implementations?

> Yes. Spring can inject implementations through `List`, `Set`, or `Map`.

---

## What is BeanFactory?

> `BeanFactory` is the basic Spring IoC container providing bean creation and dependency management.

---

## What is ApplicationContext?

> `ApplicationContext` is an advanced IoC container that extends BeanFactory functionality with features such as events, resources, environment support, and integration with Spring infrastructure.

---

## What is a Spring Bean?

> A Spring Bean is an object that is created, configured, and managed by the Spring IoC container.

---

## How does Spring create beans?

> Spring reads bean metadata, creates bean instances, resolves dependencies, injects them, performs initialization, and manages the bean lifecycle.

---

## What is @Bean?

> `@Bean` explicitly registers the object returned by a method as a Spring-managed bean.

---

## @Component vs @Bean?

> `@Component` is detected through component scanning, while `@Bean` explicitly declares a bean through a configuration method.

---

## What is circular dependency?

> A circular dependency occurs when beans directly or indirectly depend on each other, creating a dependency cycle.

---

## How can circular dependency be resolved?

> Prefer redesigning the dependency relationship; `@Lazy` can sometimes be used to defer dependency creation when appropriate.

---

## What is @Lazy?

> `@Lazy` delays bean initialization until the bean is actually required.

---

## What is loose coupling?

> Loose coupling means classes depend on abstractions rather than tightly depending on specific concrete implementations.

---

## How does Spring help achieve loose coupling?

> Spring injects dependencies from outside the class, allowing implementations to be changed without modifying the dependent class.

---

## What is Dependency Inversion Principle?

> High-level modules should depend on abstractions rather than directly depending on low-level concrete implementations.

---

# Quick Revision

~~~text
IoC
↓
Spring controls object creation and dependency management

DI
↓
Spring provides dependencies to objects

IoC Container
↓
Creates + configures + injects + manages beans

Bean
↓
Object managed by Spring

Constructor Injection
↓
Preferred for required dependencies

Setter Injection
↓
Useful for optional/changeable dependencies

Field Injection
↓
Generally discouraged

Multiple Implementations
↓
@Primary / @Qualifier / List / Set / Map

Optional Dependency
↓
Optional<T> / ObjectProvider<T>

Circular Dependency
↓
Prefer redesign / @Lazy when appropriate

Loose Coupling
↓
Depend on abstractions instead of concrete classes

DIP
↓
High-level code should depend on abstractions
~~~

# Interview One-Liners

- **IoC:** Spring takes control of object creation and dependency management.
- **DI:** Dependencies are supplied from outside instead of being created by the dependent object.
- **IoC vs DI:** IoC is the principle; DI is a mechanism for implementing it.
- **Spring Bean:** An object managed by the Spring IoC container.
- **Constructor Injection:** Preferred because dependencies are explicit, required, and can be immutable.
- **Field Injection:** Generally discouraged because it hides dependencies and hurts testability.
- **@Primary:** Selects the default bean among multiple candidates.
- **@Qualifier:** Explicitly selects a specific bean.
- **ApplicationContext:** Advanced Spring IoC container used in typical Spring Boot applications.
- **@Bean:** Explicitly registers a method's returned object as a Spring bean.
- **Circular Dependency:** A dependency cycle between beans.
- **@Lazy:** Delays bean initialization.
- **Loose Coupling:** Depend on abstractions rather than concrete implementations.
- **DIP:** High-level modules should depend on abstractions rather than low-level implementations.