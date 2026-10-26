# 01. Auto-Configuration & @EnableAutoConfiguration

## 1. What Is Auto-Configuration?

**Spring Boot Auto-Configuration** automatically configures Spring beans and application components based on the libraries available on the classpath, existing beans, and application properties.

It reduces the amount of manual configuration required to build a Spring application.

For example, when Spring Boot detects Spring MVC on the classpath, it can configure components required for handling HTTP requests.

### Without Auto-Configuration

We may need to configure several components manually, such as:
- DispatcherServlet
- Request mappings infrastructure
- HTTP message converters
- JSON serialization support

### With Auto-Configuration

Spring Boot detects the relevant dependencies and configures suitable defaults automatically.

~~~java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
~~~

With the appropriate web starter on the classpath, Spring Boot configures much of the web infrastructure automatically.

**Interview point:** Auto-configuration provides sensible defaults based on the application's classpath, existing beans, and configuration properties.

---

## 2. What Is `@EnableAutoConfiguration`?

`@EnableAutoConfiguration` tells Spring Boot to enable its auto-configuration mechanism.

It imports configuration classes that can register beans based on specific conditions.

~~~java
import org.springframework.boot.autoconfigure.EnableAutoConfiguration;
import org.springframework.context.annotation.Configuration;

@Configuration
@EnableAutoConfiguration
public class AppConfig {
}
~~~

This allows Spring Boot to configure suitable components based on the application's dependencies and environment.

### How Does It Work?

At a high level:

1. Spring Boot starts creating the application context.
2. `@EnableAutoConfiguration` activates auto-configuration.
3. Spring Boot discovers candidate auto-configuration classes.
4. Conditional annotations determine which configurations apply.
5. Applicable configurations register beans in the application context.
6. The application can use these beans through dependency injection.

**Important:** Auto-configuration classes are candidates, not guaranteed configurations. Their conditions must match before their bean definitions are applied.

---

## 3. Relationship Between `@SpringBootApplication` and `@EnableAutoConfiguration`

`@SpringBootApplication` is a composed annotation that combines three important annotations:

~~~java
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan
~~~

Example:

~~~java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
~~~

### Purpose of Each Annotation

| Annotation | Purpose |
|---|---|
| `@SpringBootConfiguration` | Identifies the class as a Spring Boot configuration class. It is a specialized form of `@Configuration`. |
| `@EnableAutoConfiguration` | Enables Spring Boot's auto-configuration mechanism. |
| `@ComponentScan` | Scans packages for components such as `@Component`, `@Service`, `@Repository`, and `@Controller`. |

### Interview Question: Is `@EnableAutoConfiguration` the Same as `@ComponentScan`?

No.

- `@ComponentScan` discovers application components in scanned packages.
- `@EnableAutoConfiguration` imports configuration candidates that conditionally configure infrastructure and other beans.

For example, your own `UserService` may be discovered through component scanning, while a suitable `DataSource` may be configured through auto-configuration.

**Interview one-liner:** `@ComponentScan` discovers application components, whereas `@EnableAutoConfiguration` configures application infrastructure based on classpath and environment conditions.

---

## 4. How Does Spring Boot Discover Auto-Configuration Classes?

Modern Spring Boot versions, including Spring Boot 3.x and 4.x, use the `AutoConfiguration.imports` mechanism to list auto-configuration candidates.

The file is located at:

~~~text
META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
~~~

It contains fully qualified class names of auto-configuration classes, typically one per line.

Example:

~~~text
com.example.autoconfigure.FeatureAutoConfiguration
com.example.autoconfigure.ClientAutoConfiguration
~~~

These are illustrative class names, not built-in Spring Boot classes.

Spring Boot discovers candidate configurations through this mechanism and evaluates their conditions.

### What About `spring.factories`?

Older Spring Boot versions used `META-INF/spring.factories` to register auto-configuration candidates.

For modern Spring Boot versions:

- Auto-configuration candidates are listed in `AutoConfiguration.imports`.
- `spring.factories` is still used for certain other extension mechanisms.
- Do not assume that modern auto-configuration candidates are registered through the old `EnableAutoConfiguration` entry in `spring.factories`.

**Interview trap:** If asked how Spring Boot 3 discovers auto-configuration classes, mention `AutoConfiguration.imports`, not the legacy `spring.factories` registration mechanism.

---

## 5. How Does Spring Boot Decide Which Configuration to Apply?

Spring Boot uses **conditional annotations** to determine whether an auto-configuration class or bean definition should be applied.

These conditions prevent unnecessary configuration and allow applications to customize defaults.

### Important Conditional Annotations

| Annotation | Purpose |
|---|---|
| `@ConditionalOnClass` | Applies when a specified class is present on the classpath. |
| `@ConditionalOnMissingClass` | Applies when a specified class is absent from the classpath. |
| `@ConditionalOnBean` | Applies when a specified bean exists. |
| `@ConditionalOnMissingBean` | Applies when a specified bean does not exist. |
| `@ConditionalOnProperty` | Applies when a property matches the configured condition. |
| `@ConditionalOnResource` | Applies when a specified resource is available. |
| `@ConditionalOnWebApplication` | Applies when the application is of the required web-application type. |
| `@ConditionalOnNotWebApplication` | Applies when the application is not a web application. |
| `@ConditionalOnExpression` | Applies when a SpEL expression evaluates to `true`. |
| `@Conditional` | Applies a custom condition implementation. |

### Example: `@ConditionalOnClass`

~~~java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(name = "com.example.SomeLibrary")
public class FeatureAutoConfiguration {
}
~~~

The configuration is eligible only when the specified class is available.

Using the `name` attribute allows the condition to check a class without directly referencing it in the annotation's class literal.

### Example: `@ConditionalOnProperty`

~~~java
@Configuration(proxyBeanMethods = false)
@ConditionalOnProperty(
    prefix = "feature",
    name = "enabled",
    havingValue = "true"
)
public class FeatureAutoConfiguration {
}
~~~

Configuration:

~~~properties
feature.enabled=true
~~~

When the property has the required value, the condition matches.

**Important:** Unless configured otherwise, `@ConditionalOnProperty` generally matches when the property is present and not equal to `false`. Setting `havingValue = "true"` explicitly requires the value to be `"true"`.

### Example: `@ConditionalOnMissingBean`

~~~java
@Configuration(proxyBeanMethods = false)
public class GreetingAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean(GreetingService.class)
    public GreetingService greetingService() {
        return new GreetingService();
    }
}
~~~

If the application context already contains a matching `GreetingService` bean, this bean definition is not applied.

This is a key mechanism that makes auto-configuration customizable.

---

## 6. How Can a Developer Override Auto-Configuration?

Spring Boot generally follows a **back-off mechanism**: an auto-configuration backs off when the relevant condition is no longer satisfied.

For example, a library may provide a default service only when the application has not defined its own service.

### Example

~~~java
public interface GreetingService {
    String greet();
}
~~~

Default implementation:

~~~java
public class DefaultGreetingService implements GreetingService {

    @Override
    public String greet() {
        return "Hello from default service";
    }
}
~~~

Auto-configuration:

~~~java
@AutoConfiguration
public class GreetingAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean(GreetingService.class)
    public GreetingService greetingService() {
        return new DefaultGreetingService();
    }
}
~~~

Application-defined implementation:

~~~java
@Service
public class CustomGreetingService implements GreetingService {

    @Override
    public String greet() {
        return "Hello from custom service";
    }
}
~~~

Because the application provides a `GreetingService`, the default bean definition backs off, assuming the custom bean is registered in the same application context and the conditions are evaluated as expected.

### Common Ways to Customize Auto-Configuration

1. Define your own bean when the relevant auto-configuration uses `@ConditionalOnMissingBean`.
2. Change the relevant configuration property.
3. Add or remove dependencies that affect classpath conditions.
4. Exclude a specific auto-configuration when it is not needed.
5. Use a custom configuration class to provide the required behavior.

**Important:** Defining a custom bean does not universally replace every auto-configured bean. The outcome depends on the specific auto-configuration conditions and bean definitions.

---

## 7. How to Exclude Auto-Configuration

Sometimes, Spring Boot configures a component that the application does not need.

You can exclude a specific auto-configuration class.

### Approach 1: Using `exclude`

~~~java
@SpringBootApplication(
    exclude = DataSourceAutoConfiguration.class
)
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
~~~

Import:

~~~java
import org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration;
~~~

This prevents that auto-configuration class from being applied.

**Use case:** An application intentionally does not want Spring Boot to configure a JDBC data source.

**Warning:** Excluding an auto-configuration can cause startup failures if other components depend on the beans it normally provides.

### Approach 2: Using `excludeName`

~~~java
@SpringBootApplication(
    excludeName = {
        "org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration"
    }
)
public class Application {
}
~~~

`excludeName` accepts fully qualified class names as strings.

It is useful when you do not want to directly reference the excluded class at compile time.

### Approach 3: Using `spring.autoconfigure.exclude`

~~~properties
spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
~~~

Multiple exclusions can be configured as a comma-separated list.

**Interview point:** Exclusion disables a particular auto-configuration class; it does not disable every form of configuration related to that technology.

---

## 8. What Happens When Multiple Auto-Configurations Are Applicable?

Multiple auto-configuration classes may be candidates in the same application.

Spring Boot evaluates their conditions and uses configuration ordering where necessary.

### Ordering Annotations

| Annotation | Purpose |
|---|---|
| `@AutoConfigureBefore` | Requests that an auto-configuration be processed before another specified configuration. |
| `@AutoConfigureAfter` | Requests that an auto-configuration be processed after another specified configuration. |
| `@AutoConfigureOrder` | Specifies an order for auto-configuration processing. |

Example:

~~~java
@AutoConfiguration
@AutoConfigureAfter(DataSourceAutoConfiguration.class)
public class MyJpaAutoConfiguration {
}
~~~

This indicates that `MyJpaAutoConfiguration` should be processed after `DataSourceAutoConfiguration`.

### Important Distinction

Auto-configuration ordering is not a universal guarantee that every bean from one configuration is instantiated before every bean from another.

It primarily controls the ordering of auto-configuration processing and can influence conditional bean-definition evaluation.

Also, do not confuse these annotations with:

- `@Order`, which is commonly used to order certain components and collections.
- `@DependsOn`, which declares bean initialization dependencies.
- `@Primary`, which helps resolve injection ambiguity between multiple candidates.

---

## 9. What Is `@AutoConfiguration`?

`@AutoConfiguration` is an annotation for declaring a Spring Boot auto-configuration class.

It is available in modern Spring Boot versions.

Example:

~~~java
@AutoConfiguration
@ConditionalOnClass(SomeLibrary.class)
public class SomeLibraryAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public SomeLibraryService someLibraryService() {
        return new SomeLibraryService();
    }
}
~~~

This example is illustrative; `SomeLibrary` and `SomeLibraryService` represent application-specific classes.

### `@Configuration` vs `@AutoConfiguration`

| `@Configuration` | `@AutoConfiguration` |
|---|---|
| Declares a source of bean definitions. | Declares a Spring Boot auto-configuration class. |
| Commonly used in application configuration. | Commonly used in auto-configuration modules and libraries. |
| Can be processed through component scanning or imports. | Designed for Spring Boot's auto-configuration mechanism. |
| Does not itself make a class an auto-configuration candidate. | Supports auto-configuration ordering metadata and is intended for auto-configuration candidates. |

**Interview point:** Annotating a class with `@AutoConfiguration` alone does not guarantee discovery. It must also be registered through the appropriate auto-configuration discovery mechanism.

---

## 10. What Is `@Import` and How Is It Related?

Spring's `@Import` annotation allows one configuration class to import another configuration class or supported import selector/registrar.

~~~java
@Configuration
@Import(FeatureConfiguration.class)
public class AppConfiguration {
}
~~~

Spring Boot's `@EnableAutoConfiguration` uses an import mechanism internally to activate auto-configuration discovery.

### Important Internal Concept

The `@EnableAutoConfiguration` mechanism uses `@Import` with `AutoConfigurationImportSelector`.

At a high level, the selector helps identify auto-configuration candidates and applies relevant filtering and exclusion rules.

**Interview one-liner:** `@Import` is a general Spring mechanism for importing configuration, while `@EnableAutoConfiguration` uses Spring Boot's import selector to discover and apply eligible auto-configuration candidates.

---

## 11. How to Debug Auto-Configuration

Sometimes an expected bean is missing, or an auto-configuration is unexpectedly applied.

Spring Boot provides ways to inspect the conditions.

### Approach 1: Enable the Debug Report

Run the application with:

~~~bash
java -jar app.jar --debug
~~~

Or configure:

~~~properties
debug=true
~~~

The condition evaluation report helps show which auto-configurations matched and which did not.

### Approach 2: Use the Actuator Conditions Endpoint

If Spring Boot Actuator is included and the endpoint is appropriately exposed and secured, the conditions endpoint can show auto-configuration evaluation details.

~~~properties
management.endpoints.web.exposure.include=conditions
~~~

Endpoint:

~~~text
/actuator/conditions
~~~

**Security note:** Do not expose diagnostic or management endpoints publicly without an appropriate security configuration.

### Common Reasons an Auto-Configuration Does Not Apply

- A required dependency is missing.
- A required class is absent from the classpath.
- A required property has not been configured.
- A required bean does not exist.
- A matching bean already exists, causing `@ConditionalOnMissingBean` to fail.
- The auto-configuration has been explicitly excluded.
- The application type does not match the condition.
- A custom condition evaluates to `false`.

---

## 12. Auto-Configuration vs Component Scanning vs Explicit Configuration

| Feature | Auto-Configuration | Component Scanning | Explicit Configuration |
|---|---|---|---|
| Main purpose | Provides conditional defaults | Discovers application components | Registers explicitly declared configuration and beans |
| Typical mechanism | `@EnableAutoConfiguration` | `@ComponentScan` | `@Configuration` and `@Bean` |
| Main input | Classpath, beans, properties, conditions | Package structure and component annotations | Developer-defined configuration |
| Can be customized? | Yes, through properties, conditions, beans, and exclusions | Yes, through scan configuration and filters | Yes, directly |
| Example | Default web or data infrastructure | Discovering `UserService` | Declaring a custom `PaymentClient` bean |

These mechanisms work together in a typical Spring Boot application.

---

## 13. Common Interview Questions and Answers

### Q1. What is Spring Boot auto-configuration?

It automatically configures suitable Spring beans based on classpath dependencies, existing beans, properties, and conditional rules.

### Q2. What is the purpose of `@EnableAutoConfiguration`?

It enables Spring Boot's mechanism for discovering and applying eligible auto-configuration classes.

### Q3. What does `@SpringBootApplication` contain?

It combines `@SpringBootConfiguration`, `@EnableAutoConfiguration`, and `@ComponentScan`.

### Q4. Does Spring Boot configure every available component?

No. It applies configurations only when their conditions match.

### Q5. What is the purpose of `@ConditionalOnMissingBean`?

It allows a default bean definition to be applied only when a matching bean is not already available.

### Q6. How do you disable a particular auto-configuration?

Use `exclude`, `excludeName`, or the `spring.autoconfigure.exclude` property.

### Q7. What is the difference between `@ConditionalOnClass` and `@ConditionalOnBean`?

`@ConditionalOnClass` checks for a class on the classpath, while `@ConditionalOnBean` checks for a bean in the application context.

### Q8. What is the difference between `@ConditionalOnProperty` and `@ConditionalOnMissingBean`?

`@ConditionalOnProperty` evaluates a configuration property, while `@ConditionalOnMissingBean` checks whether a matching bean is present.

### Q9. How does Spring Boot discover auto-configuration classes in Spring Boot 3?

It discovers candidates through `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`.

### Q10. Is `@AutoConfiguration` sufficient to register a class as an auto-configuration candidate?

No. The class must also be registered through the appropriate discovery mechanism.

### Q11. What is the back-off mechanism?

It is the behavior in which an auto-configuration avoids registering a default bean when a relevant condition, often `@ConditionalOnMissingBean`, is no longer satisfied.

### Q12. How can you find out why an auto-configuration did not apply?

Enable the debug condition evaluation report using `--debug` or `debug=true`; the Actuator conditions endpoint is another option when available and appropriately configured.

### Q13. Does auto-configuration eliminate the need for manual configuration?

No. It reduces boilerplate and supplies defaults, but applications can still define custom beans and configuration.

### Q14. What is the difference between `@Import` and `@ComponentScan`?

`@Import` explicitly imports configuration or supported import mechanisms, while `@ComponentScan` searches configured packages for component classes.

### Q15. Does defining a custom bean always override an auto-configured bean?

No. It depends on how the auto-configuration is written, the relevant conditions, bean types, qualifiers, and bean registration behavior.

---

## 14. Common Interview Traps

- **Trap:** `@SpringBootApplication` and `@EnableAutoConfiguration` are identical.
  - **Correct:** `@SpringBootApplication` includes `@EnableAutoConfiguration` along with two other annotations.

- **Trap:** Auto-configuration and component scanning do the same thing.
  - **Correct:** They serve different purposes and work together.

- **Trap:** Every auto-configuration candidate is applied.
  - **Correct:** Its conditions must match.

- **Trap:** `@ConditionalOnMissingBean` checks whether a class exists on the classpath.
  - **Correct:** It checks for matching beans; `@ConditionalOnClass` checks the classpath.

- **Trap:** All auto-configuration is registered through `spring.factories` in modern Spring Boot.
  - **Correct:** Auto-configuration candidates use `AutoConfiguration.imports` in modern Spring Boot.

- **Trap:** `@AutoConfiguration` automatically makes any class discoverable.
  - **Correct:** The class must be registered through the auto-configuration discovery mechanism.

- **Trap:** Excluding `DataSourceAutoConfiguration` disables every database-related feature.
  - **Correct:** It excludes that configuration class; other database-related configurations may still be present.

- **Trap:** Auto-configuration order guarantees the complete initialization order of every bean.
  - **Correct:** Configuration ordering is not the same as bean dependency ordering.

- **Trap:** Auto-configuration means developers cannot override defaults.
  - **Correct:** Custom beans and properties can customize or replace supported defaults.

---

## 15. Quick Revision

- **Auto-Configuration:** Automatically provides configuration based on classpath, beans, properties, and conditions.
- **`@EnableAutoConfiguration`:** Enables the auto-configuration mechanism.
- **`@SpringBootApplication`:** Combines `@SpringBootConfiguration`, `@EnableAutoConfiguration`, and `@ComponentScan`.
- **`@ConditionalOnClass`:** Checks whether a class is available on the classpath.
- **`@ConditionalOnMissingBean`:** Allows a default bean to back off when a matching bean exists.
- **`@ConditionalOnProperty`:** Applies configuration based on property values.
- **`@AutoConfiguration`:** Declares a Spring Boot auto-configuration class.
- **`AutoConfiguration.imports`:** Registers auto-configuration candidates in modern Spring Boot.
- **`@Import`:** Imports configuration or supported import mechanisms.
- **Exclusions:** Disable specific auto-configuration classes.
- **Debugging:** Use `--debug`, `debug=true`, or the Actuator conditions endpoint.
- **Back-off:** Allows application-defined configuration to take precedence when the relevant conditions support it.

### One-Line Interview Summary

Spring Boot auto-configuration uses discovered configuration candidates and conditional rules to provide sensible defaults while allowing developers to customize or exclude those defaults.

---

## 16. Related Topics

- [02. Custom Auto-Configuration & Conditional Annotations](02-Custom-Auto-Configuration-and-Conditional-Annotations.md)
- [03. Auto-Configuration Internals & Debugging](03-Auto-Configuration-Internals-and-Debugging.md)
- [04. Spring Boot Starters & Dependencies](../02-Spring-Boot-Basics/03-Starters-and-SpringBootApplication.md)
- [05. Spring Boot Overview](../02-Spring-Boot-Basics/01-Spring-Boot-Overview.md)
- [06. Configuration & Profiles](../04-Configuration-and-Profiles/01-Externalized-Configuration-and-Profiles.md)