# 01. Exception Handling in Spring Boot

## 1. What Is Exception Handling in Spring Boot?

Exception handling is the process of detecting, handling, and responding to errors that occur during application execution.

In Spring Boot REST APIs, exception handling helps return meaningful HTTP status codes and consistent error responses instead of exposing internal implementation details.

Spring provides multiple mechanisms for exception handling:

- `try-catch`: Handles exceptions locally inside a method.
- `@ExceptionHandler`: Handles specific exceptions in a controller or advice class.
- `@ControllerAdvice`: Provides exception handling across multiple controllers.
- `@RestControllerAdvice`: Provides global exception handling for REST APIs.
- `ResponseEntity`: Controls the HTTP status, headers, and response body.
- `ResponseStatusException`: Allows an exception to carry an HTTP status and reason.
- `@ResponseStatus`: Associates an HTTP status with an exception class or handler method.

**Interview one-liner:** Spring Boot exception handling allows applications to convert runtime failures into meaningful, consistent HTTP responses.

---

## 2. Types of Exceptions in Spring Applications

### Checked Exceptions

Checked exceptions are checked by the compiler. A method must catch them or declare them using `throws`, unless the exception is handled at an appropriate enclosing level.

Examples:

- `IOException`
- `SQLException` (when using APIs that expose it directly)

~~~java
public void readFile() throws IOException {
    Files.readString(Path.of("data.txt"));
}
~~~

### Unchecked Exceptions

Unchecked exceptions extend `RuntimeException` and do not need to be declared or caught by the compiler.

Examples:

- `NullPointerException`
- `IllegalArgumentException`
- `IllegalStateException`
- Custom business exceptions extending `RuntimeException`

~~~java
public Product findProduct(Long id) {
    if (id == null) {
        throw new IllegalArgumentException("Product ID is required");
    }

    return productRepository.findById(id).orElseThrow();
}
~~~

### Errors

`Error` represents serious JVM or system-level problems, such as `OutOfMemoryError` or `StackOverflowError`.

These are generally not handled as ordinary application exceptions.

### Comparison

| Type | Compiler requires handling? | Examples |
|---|---|---|
| Checked exception | Yes, through catch or declaration | `IOException` |
| Unchecked exception | No | `IllegalArgumentException` |
| Error | No | `OutOfMemoryError` |

**Interview trap:** Spring does not require every exception to be caught inside a service or controller. Exceptions can propagate to Spring MVC's exception-resolution mechanism.

---

## 3. Why Is Exception Handling Important?

Without proper exception handling, an API may return inconsistent error responses or expose unnecessary internal information.

Exception handling helps with:

- Returning appropriate HTTP status codes.
- Providing consistent error response structures.
- Separating business logic from error-response formatting.
- Preventing sensitive implementation details from leaking to clients.
- Logging unexpected failures.
- Making API errors easier for clients to process.
- Improving maintainability and debugging.

### Poor Approach

~~~java
@GetMapping("/{id}")
public Product getProduct(@PathVariable Long id) {
    try {
        return productService.getById(id);
    } catch (Exception ex) {
        return null;
    }
}
~~~

Problems:

- Catches every exception without distinguishing its cause.
- Silently hides failures.
- Returns an unclear response.
- Makes debugging difficult.
- Can cause a successful-looking HTTP response with a null body, depending on response handling.

### Better Approach

Let meaningful exceptions propagate and handle them centrally where appropriate.

~~~java
@GetMapping("/{id}")
public ProductResponse getProduct(@PathVariable Long id) {
    return productService.getById(id);
}
~~~

The service can throw a specific exception when the product is missing, and a global exception handler can translate it into a suitable HTTP response.

---

## 4. Local Exception Handling with `try-catch`

A `try-catch` block handles an exception near the code that may fail.

~~~java
public int divide(int a, int b) {
    try {
        return a / b;
    } catch (ArithmeticException ex) {
        return 0;
    }
}
~~~

Here, division by zero is handled locally.

However, returning `0` is only appropriate if the application's business requirements define that behavior. Otherwise, it may hide a real error.

### When to Use `try-catch`

Use local exception handling when:

- The current method can meaningfully recover.
- A resource needs to be cleaned up.
- An exception must be translated at a specific boundary.
- The method can provide a valid fallback according to business requirements.

Avoid catching an exception simply to log it and rethrow the same exception without adding useful context or handling.

### Try-With-Resources

Use try-with-resources for resources implementing `AutoCloseable`.

~~~java
public String readFile(Path path) throws IOException {
    try (BufferedReader reader = Files.newBufferedReader(path)) {
        return reader.readLine();
    }
}
~~~

The resource is closed automatically when the block exits, including when an exception occurs.

**Interview one-liner:** Use `try-catch` for local recovery; use centralized exception handling to standardize application-level error responses.

---

## 5. What Is `@ExceptionHandler`?

`@ExceptionHandler` marks a method that handles one or more specified exception types.

It can be declared inside a controller for local exception handling or inside a controller advice class for shared handling.

### Controller-Level Example

~~~java
@RestController
@RequestMapping("/api/products")
public class ProductController {

    @GetMapping("/{id}")
    public ProductResponse getProduct(@PathVariable Long id) {
        return productService.getById(id);
    }

    @ExceptionHandler(ProductNotFoundException.class)
    public ResponseEntity<ApiError> handleProductNotFound(
            ProductNotFoundException ex) {

        ApiError error = new ApiError(
                "PRODUCT_NOT_FOUND",
                ex.getMessage());

        return ResponseEntity.status(HttpStatus.NOT_FOUND)
                .body(error);
    }
}
~~~

The handler applies to exceptions raised while processing requests handled by this controller, subject to Spring MVC's exception-resolution rules.

### Custom Exception

~~~java
public class ProductNotFoundException extends RuntimeException {

    public ProductNotFoundException(Long id) {
        super("Product not found with ID: " + id);
    }
}
~~~

### Error DTO

~~~java
public record ApiError(
        String code,
        String message) {
}
~~~

When `ProductNotFoundException` reaches Spring MVC's exception handling, the handler returns:

- HTTP status: `404 Not Found`
- Error code: `PRODUCT_NOT_FOUND`
- Message: A description of the missing product

### Important Points

- `@ExceptionHandler` can handle specific exception types.
- The handler can return an object, a response body, or a `ResponseEntity`.
- The handler can inspect the exception to create a useful response.
- A controller-level handler is not automatically global.

**Interview one-liner:** `@ExceptionHandler` maps specified exceptions to handler methods that produce an appropriate response.

---

## 6. What Is `@ControllerAdvice`?

`@ControllerAdvice` allows cross-cutting controller behavior to be shared across multiple controllers.

It can be used for:

- Global exception handling.
- Shared model attributes.
- Shared data-binding configuration.

For REST APIs, it is commonly used with `@ExceptionHandler`.

### Example

~~~java
@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ProductNotFoundException.class)
    public ResponseEntity<ApiError> handleProductNotFound(
            ProductNotFoundException ex) {

        ApiError error = new ApiError(
                "PRODUCT_NOT_FOUND",
                ex.getMessage());

        return ResponseEntity.status(HttpStatus.NOT_FOUND)
                .body(error);
    }
}
~~~

This advice can handle the specified exception across controllers within its applicable scope.

### Why Use `@ControllerAdvice`?

Without a shared handler, several controllers may repeat the same exception-to-response logic.

A controller advice centralizes that logic and promotes consistent error responses.

### Important Note

`@ControllerAdvice` is not limited to exception handling. It can also provide shared model attributes and binding behavior.

---

## 7. What Is `@RestControllerAdvice`?

`@RestControllerAdvice` is a composed annotation that combines:

- `@ControllerAdvice`
- `@ResponseBody`

It is designed for controller-wide behavior where returned values should be written to the HTTP response body.

### Example

~~~java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ProductNotFoundException.class)
    public ResponseEntity<ApiError> handleProductNotFound(
            ProductNotFoundException ex) {

        ApiError error = new ApiError(
                "PRODUCT_NOT_FOUND",
                ex.getMessage());

        return ResponseEntity.status(HttpStatus.NOT_FOUND)
                .body(error);
    }
}
~~~

### `@ControllerAdvice` vs `@RestControllerAdvice`

| Feature | `@ControllerAdvice` | `@RestControllerAdvice` |
|---|---|---|
| Purpose | Shared controller behavior | Shared controller behavior with response-body semantics |
| Default return handling | Depends on handler and return-value annotations | Returned objects are treated as response bodies |
| Common use | MVC views and shared controller logic | REST API exception handling |

**Interview one-liner:** `@RestControllerAdvice` combines `@ControllerAdvice` and `@ResponseBody`, making it convenient for global REST API exception handling.

---

## 8. Global Exception Handling: Complete Example

A common Spring Boot design uses custom exceptions, an error DTO, and a global exception handler.

### Step 1: Define a Custom Exception

~~~java
public class ProductNotFoundException extends RuntimeException {

    public ProductNotFoundException(Long id) {
        super("Product not found with ID: " + id);
    }
}
~~~

### Step 2: Define an Error DTO

~~~java
public record ApiError(
        String code,
        String message) {
}
~~~

### Step 3: Throw the Exception from the Service

~~~java
@Service
public class ProductService {

    private final ProductRepository productRepository;

    public ProductService(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }

    public ProductResponse getById(Long id) {
        Product product = productRepository.findById(id)
                .orElseThrow(() -> new ProductNotFoundException(id));

        return new ProductResponse(
                product.getId(),
                product.getName(),
                product.getPrice());
    }
}
~~~

This assumes `Product`, `ProductRepository`, and `ProductResponse` exist in the application.

### Step 4: Define the Global Exception Handler

~~~java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ProductNotFoundException.class)
    public ResponseEntity<ApiError> handleProductNotFound(
            ProductNotFoundException ex) {

        ApiError error = new ApiError(
                "PRODUCT_NOT_FOUND",
                ex.getMessage());

        return ResponseEntity.status(HttpStatus.NOT_FOUND)
                .body(error);
    }
}
~~~

### Step 5: Controller

~~~java
@RestController
@RequestMapping("/api/products")
public class ProductController {

    private final ProductService productService;

    public ProductController(ProductService productService) {
        this.productService = productService;
    }

    @GetMapping("/{id}")
    public ProductResponse getProduct(@PathVariable Long id) {
        return productService.getById(id);
    }
}
~~~

### Execution Flow

1. The client requests `/api/products/101`.
2. The controller calls `productService.getById(101)`.
3. The service searches the repository.
4. If the product is missing, the service throws `ProductNotFoundException`.
5. The exception propagates through the service and controller invocation.
6. Spring MVC's exception-resolution mechanism locates the applicable exception handler.
7. `GlobalExceptionHandler` creates an `ApiError`.
8. The client receives `404 Not Found` with a structured error body.

Example response:

~~~json
{
  "code": "PRODUCT_NOT_FOUND",
  "message": "Product not found with ID: 101"
}
~~~

**Interview one-liner:** The service throws a meaningful exception, and the global exception handler translates it into a consistent HTTP response.

---

## 9. Handling Multiple Exceptions

A global exception handler can contain multiple `@ExceptionHandler` methods.

~~~java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ProductNotFoundException.class)
    public ResponseEntity<ApiError> handleNotFound(
            ProductNotFoundException ex) {

        return ResponseEntity.status(HttpStatus.NOT_FOUND)
                .body(new ApiError(
                        "PRODUCT_NOT_FOUND",
                        ex.getMessage()));
    }

    @ExceptionHandler(IllegalArgumentException.class)
    public ResponseEntity<ApiError> handleBadRequest(
            IllegalArgumentException ex) {

        return ResponseEntity.badRequest()
                .body(new ApiError(
                        "INVALID_ARGUMENT",
                        ex.getMessage()));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiError> handleUnexpected(
            Exception ex) {

        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(new ApiError(
                        "INTERNAL_ERROR",
                        "An unexpected error occurred"));
    }
}
~~~

### Important Rules

- Prefer specific handlers for known application exceptions.
- Use a generic handler as a final fallback, not as a replacement for meaningful exception classification.
- Avoid returning raw internal exception messages for unexpected server errors.
- Log unexpected failures with sufficient diagnostic information.
- Avoid duplicate or ambiguous exception-handling methods.

Spring chooses a suitable handler based on the exception type and the applicable exception-resolution rules.

**Interview trap:** Do not assume a generic `Exception` handler should always return `400 Bad Request`. Unexpected application failures usually require a server-error status such as `500 Internal Server Error`.

---

## 10. Custom Exceptions in Spring Boot

Custom exceptions represent application-specific failures.

They make error handling clearer than using generic exceptions for every situation.

### Example

~~~java
public class InsufficientStockException extends RuntimeException {

    public InsufficientStockException(String message) {
        super(message);
    }
}
~~~

Service:

~~~java
public void placeOrder(int requestedQuantity, int availableStock) {
    if (requestedQuantity > availableStock) {
        throw new InsufficientStockException(
                "Insufficient stock for requested quantity");
    }
}
~~~

Global handler:

~~~java
@ExceptionHandler(InsufficientStockException.class)
public ResponseEntity<ApiError> handleInsufficientStock(
        InsufficientStockException ex) {

    return ResponseEntity.status(HttpStatus.CONFLICT)
            .body(new ApiError(
                    "INSUFFICIENT_STOCK",
                    ex.getMessage()));
}
~~~

A stock conflict can reasonably be represented as `409 Conflict`, depending on the API contract.

### Benefits of Custom Exceptions

- Express business failures clearly.
- Avoid mixing unrelated errors under generic exception types.
- Simplify exception-to-status mapping.
- Improve readability and testing.
- Allow consistent error codes and messages.

### Checked vs Unchecked Custom Exceptions

A custom exception can extend either `Exception` or `RuntimeException`.

~~~java
public class BusinessException extends RuntimeException {
    public BusinessException(String message) {
        super(message);
    }
}
~~~

Unchecked custom exceptions are common for business failures in Spring applications because they do not require every caller to declare or catch them.

Choose checked exceptions when the application's design benefits from requiring callers to handle or declare the condition.

**Interview trap:** Spring does not require all custom exceptions to extend `RuntimeException`. The choice depends on the application's error-handling design.

---

## 11. `ResponseStatusException`

`ResponseStatusException` allows an application to associate an exception with an HTTP status and a reason.

### Example

~~~java
@GetMapping("/{id}")
public ProductResponse getProduct(@PathVariable Long id) {

    return productService.findById(id)
            .orElseThrow(() ->
                    new ResponseStatusException(
                            HttpStatus.NOT_FOUND,
                            "Product not found"));
}
~~~

This can be useful for straightforward cases where a custom exception class would be unnecessary.

### Advantages

- Quick to use.
- Carries an HTTP status.
- Avoids creating a separate exception class for every simple case.

### Limitations

- Can couple application logic to HTTP-specific behavior.
- May scatter status-selection logic across services and controllers.
- Does not automatically create the application's preferred structured error DTO.

### When to Use It

Use it for simple, localized HTTP errors. Prefer custom domain/application exceptions plus centralized handling when the application needs consistent error codes, reusable business exceptions, or transport-independent service logic.

**Interview one-liner:** `ResponseStatusException` lets code raise an exception with an associated HTTP status and reason.

---

## 12. `@ResponseStatus`

`@ResponseStatus` associates an HTTP status with a handler method or an exception class.

### On a Controller Method

~~~java
@PostMapping
@ResponseStatus(HttpStatus.CREATED)
public ProductResponse create(
        @RequestBody ProductRequest request) {

    return productService.create(request);
}
~~~

A successful invocation returns `201 Created`.

### On a Custom Exception

~~~java
@ResponseStatus(HttpStatus.NOT_FOUND)
public class ProductNotFoundException extends RuntimeException {

    public ProductNotFoundException(Long id) {
        super("Product not found with ID: " + id);
    }
}
~~~

When this exception is resolved through Spring MVC's response-status handling, it is associated with `404 Not Found`.

### `@ResponseStatus` vs `ResponseStatusException`

| Feature | `@ResponseStatus` | `ResponseStatusException` |
|---|---|---|
| Form | Annotation | Exception class |
| Status selection | Declared statically | Supplied when creating the exception |
| Dynamic status per failure | Not directly | Yes |
| Common use | Fixed handler or exception status | Conditional or localized status |
| Structured custom error DTO | Usually needs additional handling | Usually needs additional handling |

**Interview trap:** `@ResponseStatus` does not replace a global exception handler when the API requires a custom, consistent JSON error structure.

---

## 13. Handling Validation Exceptions

Spring MVC supports Bean Validation through annotations such as `@Valid`, `@NotBlank`, and `@Positive`.

### Request DTO

~~~java
public record ProductRequest(
        @NotBlank(message = "Name is required")
        String name,

        @NotNull(message = "Price is required")
        @Positive(message = "Price must be positive")
        BigDecimal price) {
}
~~~

### Controller

~~~java
@PostMapping
public ProductResponse create(
        @Valid @RequestBody ProductRequest request) {

    return productService.create(request);
}
~~~

When request-body validation fails, Spring MVC typically raises `MethodArgumentNotValidException`.

### Global Handler

~~~java
@ExceptionHandler(MethodArgumentNotValidException.class)
public ResponseEntity<ApiError> handleValidation(
        MethodArgumentNotValidException ex) {

    return ResponseEntity.badRequest()
            .body(new ApiError(
                    "VALIDATION_FAILED",
                    "One or more request fields are invalid"));
}
~~~

This is a simplified handler. A production API may return individual field errors.

### Example Field-Level Error DTOs

~~~java
public record FieldErrorDetail(
        String field,
        String message) {
}

public record ValidationErrorResponse(
        String code,
        List<FieldErrorDetail> errors) {
}
~~~

The handler can collect binding errors and convert them into `FieldErrorDetail` objects.

### Important Exception Types

| Exception | Typical scenario |
|---|---|
| `MethodArgumentNotValidException` | Request-body object fails validation |
| `HandlerMethodValidationException` | Supported controller method validation fails in modern Spring MVC |
| `ConstraintViolationException` | Bean Validation constraints fail in applicable service or method-validation scenarios |
| `MethodArgumentTypeMismatchException` | Request value cannot be converted to the expected type |
| `MissingServletRequestParameterException` | Required request parameter is missing |
| `HttpMessageNotReadableException` | Request body cannot be read or deserialized |

Exact exception types depend on how validation is triggered and the Spring Framework version.

**Interview trap:** Do not assume every validation failure raises `ConstraintViolationException`. The exception depends on the validation mechanism and where validation occurs.

---

## 14. Handling Malformed JSON and Type Conversion Errors

Not all bad requests are validation failures.

### Malformed JSON

A request body containing invalid JSON can raise `HttpMessageNotReadableException`.

Example invalid JSON:

~~~json
{
  "name": "Keyboard",
  "price":
}
~~~

A handler can return a consistent response:

~~~java
@ExceptionHandler(HttpMessageNotReadableException.class)
public ResponseEntity<ApiError> handleUnreadableBody(
        HttpMessageNotReadableException ex) {

    return ResponseEntity.badRequest()
            .body(new ApiError(
                    "INVALID_REQUEST_BODY",
                    "Request body is malformed or unreadable"));
}
~~~

### Type Conversion Failure

A request such as `/api/products/abc` may fail when Spring attempts to convert `abc` to a `Long`.

~~~java
@ExceptionHandler(MethodArgumentTypeMismatchException.class)
public ResponseEntity<ApiError> handleTypeMismatch(
        MethodArgumentTypeMismatchException ex) {

    return ResponseEntity.badRequest()
            .body(new ApiError(
                    "INVALID_PARAMETER",
                    "A request parameter has an invalid value"));
}
~~~

### Why Separate Handlers?

Malformed bodies, invalid field values, missing resources, and unexpected server errors represent different conditions.

Separate handlers can provide appropriate status codes, stable error codes, and clearer messages.

**Security note:** Avoid including raw exception details in client responses when they might expose implementation information.

---

## 15. Exception Handling in the Request Lifecycle

A simplified Spring MVC request flow is:

1. The client sends an HTTP request.
2. `DispatcherServlet` receives it.
3. Spring finds a matching handler method.
4. Spring resolves controller method arguments.
5. Spring invokes the controller method.
6. The controller calls the service layer.
7. An exception occurs in the controller or a downstream call.
8. The exception propagates to Spring MVC's exception-resolution mechanism.
9. An appropriate `HandlerExceptionResolver` processes the exception.
10. The handler creates a response status and body.
11. Spring writes the HTTP response.

### Where Can an Exception Occur?

- During request mapping.
- During request-body deserialization.
- During argument conversion.
- During validation.
- Inside the controller.
- Inside the service.
- During repository or database operations.
- During response serialization.

### Important Distinction

Exceptions raised before a controller method is invoked may still be handled by Spring MVC's exception-resolution infrastructure, but not every exception raised anywhere in the application is automatically handled by a controller advice.

For example, exceptions raised in servlet filters occur outside the normal controller invocation path and may need filter-level handling or Spring Security's exception-handling components.

**Interview one-liner:** Spring MVC uses exception-resolution infrastructure to locate an applicable handler and turn supported request-processing exceptions into HTTP responses.

---

## 16. How Does Spring MVC Resolve Exceptions?

Spring MVC uses the `HandlerExceptionResolver` mechanism to process exceptions raised during request handling.

Important components include:

### `ExceptionHandlerExceptionResolver`

Resolves exceptions using methods annotated with `@ExceptionHandler`, including applicable methods in controller advice classes.

### `ResponseStatusExceptionResolver`

Handles exceptions associated with response statuses, including `@ResponseStatus` and `ResponseStatusException`.

### `DefaultHandlerExceptionResolver`

Handles several standard Spring MVC exceptions and maps them to appropriate HTTP status codes.

These resolvers are part of the normal Spring MVC exception-resolution infrastructure.

### Typical Resolution Order

Spring MVC generally consults its configured exception resolvers in order. In standard configurations, annotation-based exception handling is commonly processed before response-status handling and the default resolver.

The precise behavior depends on resolver configuration and the application.

**Interview trap:** `@RestControllerAdvice` is not a replacement for the entire Spring MVC exception-resolution system. It participates in that system through exception-handler resolution.

---

## 17. Local vs Global Exception Handling

| Feature | Controller-level `@ExceptionHandler` | Global `@RestControllerAdvice` |
|---|---|---|
| Scope | Applicable controller | Applicable set of controllers |
| Common use | Controller-specific failures | Shared application error handling |
| Reusability | Limited | High |
| Centralized response format | Possible locally | Easier across the API |
| Priority | Local handler can take precedence for its controller | Used when applicable local handling does not resolve the exception |

### Recommended Approach

- Use local handlers when a controller has genuinely controller-specific behavior.
- Use global advice for shared exceptions and standardized REST error responses.
- Avoid duplicating the same exception mapping in multiple controllers.
- Use advice selectors when an application needs different policies for different controller groups.

### Restricting Advice Scope

Spring allows controller advice to be restricted using selectors such as package, annotation, or assignable type.

~~~java
@RestControllerAdvice(basePackages = "com.example.products")
public class ProductExceptionHandler {
    // Exception handlers for applicable controllers
}
~~~

This can help separate error-handling policies across application modules.

---

## 18. Exception Handling and HTTP Status Codes

Choose a status code that reflects the failure.

| Exception or scenario | Typical HTTP status |
|---|---|
| Invalid request body | 400 Bad Request |
| Missing required parameter | 400 Bad Request |
| Type conversion failure | 400 Bad Request |
| Request validation failure | 400 Bad Request |
| Missing resource | 404 Not Found |
| Unsupported HTTP method | 405 Method Not Allowed |
| Unsupported request content type | 415 Unsupported Media Type |
| Business-state conflict | 409 Conflict |
| Missing or invalid authentication | 401 Unauthorized |
| Access denied | 403 Forbidden |
| Unexpected server exception | 500 Internal Server Error |
| Temporarily unavailable service | 503 Service Unavailable |

These are common conventions, not universal mappings for every application. Configure and document the API's error contract consistently.

### Example: Business Conflict

~~~java
@ExceptionHandler(InsufficientStockException.class)
public ResponseEntity<ApiError> handleInsufficientStock(
        InsufficientStockException ex) {

    return ResponseEntity.status(HttpStatus.CONFLICT)
            .body(new ApiError(
                    "INSUFFICIENT_STOCK",
                    ex.getMessage()));
}
~~~

### Example: Unexpected Failure

~~~java
@ExceptionHandler(Exception.class)
public ResponseEntity<ApiError> handleUnexpected(Exception ex) {

    // Log the exception using the application's logging framework.

    return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(new ApiError(
                    "INTERNAL_ERROR",
                    "An unexpected error occurred"));
}
~~~

The generic handler should not expose the raw exception message to the client.

---

## 19. Logging Exceptions Properly

Exception handling and exception logging serve different purposes.

- Exception handling determines how the application responds.
- Exception logging records diagnostic information for developers and operators.

### Example

~~~java
private static final Logger log =
        LoggerFactory.getLogger(GlobalExceptionHandler.class);

@ExceptionHandler(Exception.class)
public ResponseEntity<ApiError> handleUnexpected(Exception ex) {

    log.error("Unexpected error while processing request", ex);

    return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(new ApiError(
                    "INTERNAL_ERROR",
                    "An unexpected error occurred"));
}
~~~

Passing the exception as the logging argument preserves its stack trace in the log output for supported logging frameworks.

### Logging Guidelines

- Log unexpected server errors with useful context.
- Avoid logging the same exception at every layer without a reason.
- Avoid logging passwords, access tokens, or sensitive request data.
- Use appropriate log levels.
- Include a correlation or trace ID where available.
- Do not expose stack traces to clients.

### Common Mistake

~~~java
catch (Exception ex) {
    log.error("Error occurred");
    throw ex;
}
~~~

This logs a generic message without attaching the exception, so the stack trace is not recorded by that logging call.

If the exception is rethrown, another layer may log it, but logging should be intentional and should not create noisy duplicates.

---

## 20. Spring Boot Error Handling: `ErrorController` and `/error`

Spring Boot provides default error handling for servlet-based web applications.

The default error infrastructure commonly routes unresolved servlet errors to `/error`, with `BasicErrorController` providing default error responses.

### Important Components

- `ErrorController`: An interface used for error-controller customization.
- `BasicErrorController`: Spring Boot's default MVC error controller.
- `/error`: The conventional error path used by Spring Boot's default error handling.

### Does `@RestControllerAdvice` Replace `/error`?

No.

`@RestControllerAdvice` handles applicable exceptions through Spring MVC's exception-resolution mechanism.

The `/error` infrastructure provides a fallback path for servlet errors that reach the container's error handling.

They serve related but different purposes.

### When to Customize `/error`

Customize the default error endpoint when the application needs a different fallback error response for servlet errors not handled by its usual exception handlers.

For most REST APIs, start with a global `@RestControllerAdvice` for application exceptions and common MVC errors. Customize the fallback infrastructure only when the API requirements justify it.

**Interview trap:** Do not assume every error automatically passes through `@RestControllerAdvice`. Servlet filters, container error dispatches, security failures, and infrastructure-level errors may follow different paths.

---

## 21. Exception Handling with Spring Security

Spring Security has its own mechanisms for authentication and authorization failures.

Important components:

- `AuthenticationEntryPoint`: Handles requests that require authentication or contain invalid authentication credentials.
- `AccessDeniedHandler`: Handles access-denied situations, commonly when an authenticated user lacks sufficient authority.

### Why Is This Different?

A request can be rejected before it reaches a controller.

For example:

1. A request enters the servlet filter chain.
2. Spring Security checks authentication and authorization.
3. Security rejects the request.
4. The controller method is never invoked.

In this scenario, a controller-level `@ExceptionHandler` or ordinary controller advice may not handle the failure.

### Common Status Codes

- `401 Unauthorized`: Authentication is required or credentials are invalid.
- `403 Forbidden`: Access is denied.

Exact behavior depends on the security configuration and authentication mechanism.

**Interview one-liner:** Spring MVC advice handles applicable controller exceptions, while Spring Security uses components such as `AuthenticationEntryPoint` and `AccessDeniedHandler` for security failures.

---

## 22. Exception Handling Best Practices

- Create meaningful custom exceptions for important business failures.
- Prefer specific exception handlers over a single generic handler for every error.
- Centralize shared REST error formatting using `@RestControllerAdvice`.
- Return appropriate HTTP status codes.
- Use stable application error codes.
- Avoid exposing stack traces and internal implementation details.
- Log unexpected exceptions with diagnostic context.
- Avoid logging sensitive data.
- Validate request DTOs with Bean Validation.
- Keep controllers focused on HTTP concerns.
- Keep business exception decisions close to the relevant business logic.
- Avoid silently swallowing exceptions.
- Avoid catching `Exception` when only one specific exception is expected.
- Avoid returning `200 OK` for failed operations.
- Keep the error response structure consistent across endpoints.
- Document status codes and error payloads in API documentation.
- Test both expected failures and unexpected exceptions.
- Use separate security handling for authentication and authorization failures.

---

## 23. Frequently Asked Interview Questions

### Q1. What is exception handling in Spring Boot?

It is the process of handling application and request-processing failures and translating them into appropriate responses, commonly HTTP responses for REST APIs.

### Q2. What is `@ExceptionHandler`?

It identifies a method that handles specified exception types within a controller or applicable controller advice.

### Q3. What is `@ControllerAdvice`?

It provides shared controller behavior across multiple controllers, including exception handling, model attributes, and binding configuration.

### Q4. What is `@RestControllerAdvice`?

It combines `@ControllerAdvice` and `@ResponseBody`, making it convenient for global REST API exception handling.

### Q5. What is the difference between `@ControllerAdvice` and `@RestControllerAdvice`?

`@RestControllerAdvice` adds response-body semantics to the shared controller behavior, whereas `@ControllerAdvice` does not apply that behavior by itself.

### Q6. What is the difference between local and global exception handling?

Local exception handlers apply to a controller; global advice handles exceptions across its applicable controller scope.

### Q7. What is a custom exception?

It is an application-defined exception that represents a meaningful business or application failure.

### Q8. What is `ResponseStatusException`?

It is a Spring exception that can carry an HTTP status and reason.

### Q9. What is the difference between `@ResponseStatus` and `ResponseStatusException`?

`@ResponseStatus` declares a fixed status through an annotation; `ResponseStatusException` allows a status to be supplied when the exception is created.

### Q10. How do you handle a resource-not-found exception?

Throw a meaningful custom exception and map it to `404 Not Found` using `@ExceptionHandler` or an appropriate response-status mechanism.

### Q11. How do you handle validation failures?

Trigger validation with `@Valid` or another supported mechanism and use exception handling to return a consistent error response.

### Q12. Which exception is commonly raised when a JSON request body fails validation?

`MethodArgumentNotValidException` is common for `@Valid @RequestBody` validation failures. Other validation mechanisms can produce different exception types.

### Q13. Which exception is commonly raised for malformed JSON?

`HttpMessageNotReadableException` is commonly raised when Spring MVC cannot read or deserialize a request body.

### Q14. What is `HandlerExceptionResolver`?

It is Spring MVC infrastructure that attempts to resolve exceptions raised during request processing.

### Q15. What is `ExceptionHandlerExceptionResolver`?

It resolves exceptions using applicable `@ExceptionHandler` methods.

### Q16. Does `@RestControllerAdvice` handle exceptions raised in servlet filters?

Not necessarily. Filters execute outside the normal controller invocation path, so filter-level or security-specific handling may be required.

### Q17. Does Spring Boot provide default error handling?

Yes. In servlet-based web applications, Spring Boot provides default error infrastructure that commonly uses `/error` and `BasicErrorController`.

### Q18. What is the difference between exception handling and logging?

Exception handling determines the response or recovery behavior; logging records information for diagnosis and monitoring.

### Q19. Should every exception return 500?

No. Known conditions should map to appropriate statuses, such as 400 for invalid input and 404 for a missing resource. Unexpected failures commonly map to 500.

### Q20. Why should raw exception messages not always be returned to clients?

They may reveal implementation details, sensitive data, internal paths, database information, or other details that should remain private.

---

## 24. Common Interview Traps

- Assuming `@ExceptionHandler` is automatically global.
- Assuming `@ControllerAdvice` and `@RestControllerAdvice` are identical.
- Assuming every exception must be caught inside the service.
- Catching every exception and returning `null`.
- Returning `200 OK` for failed operations.
- Mapping every exception to `400 Bad Request`.
- Mapping every exception to `500 Internal Server Error`.
- Returning raw exception messages for unexpected failures.
- Assuming `@Valid` itself formats the error response.
- Assuming every validation failure raises the same exception type.
- Assuming controller advice handles every exception in the entire application.
- Assuming Spring Security failures always reach the controller.
- Assuming `@ResponseStatus` creates a custom JSON error structure automatically.
- Confusing logging with handling.
- Logging an exception without attaching the throwable when a stack trace is needed.
- Duplicating identical exception handling logic in every controller.
- Assuming `/error` and `@RestControllerAdvice` serve exactly the same purpose.
- Assuming custom exceptions must always extend `RuntimeException`.

---

## 25. Quick Revision

| Concept | Remember |
|---|---|
| `try-catch` | Local exception handling |
| `@ExceptionHandler` | Handles specified exceptions |
| `@ControllerAdvice` | Shared controller behavior |
| `@RestControllerAdvice` | Global REST-oriented controller advice |
| Custom exception | Represents a meaningful application failure |
| `ResponseStatusException` | Exception carrying an HTTP status and reason |
| `@ResponseStatus` | Declares a status for a handler or exception |
| `ResponseEntity` | Controls response status, headers, and body |
| `HandlerExceptionResolver` | Spring MVC exception-resolution infrastructure |
| `ExceptionHandlerExceptionResolver` | Resolves exceptions using `@ExceptionHandler` methods |
| `DefaultHandlerExceptionResolver` | Handles standard Spring MVC exceptions |
| `BasicErrorController` | Spring Boot's default MVC error controller |
| `MethodArgumentNotValidException` | Common request-body validation exception |
| `HttpMessageNotReadableException` | Common unreadable or malformed request-body exception |
| `AuthenticationEntryPoint` | Handles authentication entry failures |
| `AccessDeniedHandler` | Handles access-denied failures |
| 400 | Invalid request |
| 401 | Authentication required or invalid |
| 403 | Access forbidden |
| 404 | Resource or handler not found |
| 409 | State conflict |
| 500 | Unexpected server failure |

### One-Line Interview Answers

- **Exception handling:** Converts failures into controlled application behavior or meaningful HTTP responses.
- **`@ExceptionHandler`:** Maps specific exceptions to handler methods.
- **`@ControllerAdvice`:** Shares controller behavior across applicable controllers.
- **`@RestControllerAdvice`:** Provides shared controller advice with response-body semantics.
- **Custom exception:** Represents a meaningful application-specific failure.
- **`ResponseStatusException`:** Associates an exception with an HTTP status and reason.
- **`@ResponseStatus`:** Declares an HTTP status for a handler or exception.
- **`HandlerExceptionResolver`:** Resolves exceptions during Spring MVC request processing.
- **Global exception handling:** Centralizes shared exception-to-response logic.
- **Validation handling:** Converts validation failures into consistent error responses.
- **`BasicErrorController`:** Provides Spring Boot's default MVC error endpoint behavior.
- **Security exception handling:** Uses Spring Security components to handle authentication and access-denied failures.

---

## 26. Related Topics

- [01. Spring MVC Architecture & Request Flow](../05-Spring-MVC/01-Spring-MVC-Architecture.md)
- [02. Controllers & Request Mapping](../05-Spring-MVC/02-Controllers-and-Mappings.md)
- [03. Path Variables, Request Params, Headers & Request Body](../05-Spring-MVC/03-Request-Parameters-and-Body.md)
- [04. ResponseEntity, HTTP Status Codes & DTOs](../05-Spring-MVC/04-Responses-and-DTOs.md)
- [05. Validation & Jackson](../05-Spring-MVC/05-Validation-and-Jackson.md)
- [06. REST API Design & Clients](../05-Spring-MVC/06-REST-API-Design-and-Clients.md)
- [Spring Security](../09-Spring-Security/01-Spring-Security-Fundamentals.md)