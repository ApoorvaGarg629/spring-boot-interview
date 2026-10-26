# 04. ResponseEntity, HTTP Status Codes & DTOs

## 1. Overview

In Spring MVC, a controller can return different types of responses depending on whether the application renders a view or exposes a REST API.

For REST APIs, the response commonly contains:

- HTTP status code
- Response headers
- Response body
- A DTO representing the returned data

Important concepts:

- `ResponseEntity<T>`: Controls the HTTP status, headers, and body.
- `HttpStatus`: Provides predefined HTTP status codes.
- `@ResponseStatus`: Declares a response status for a handler method or exception.
- DTO (Data Transfer Object): Defines the data exchanged between API clients and the application.
- `HttpMessageConverter`: Converts Java objects into HTTP response bodies, commonly JSON.

**Interview one-liner:** `ResponseEntity` provides control over the complete HTTP response, while DTOs define the structure of data exchanged through the API.

---

## 2. What Is `ResponseEntity`?

`ResponseEntity<T>` is a Spring class that represents an HTTP response, including its status code, headers, and body.

It is commonly used in REST controllers when the response status or headers need to be controlled explicitly.

### Basic Example

~~~java
@RestController
@RequestMapping("/api/products")
public class ProductController {

    @GetMapping("/{id}")
    public ResponseEntity<String> getProduct(
            @PathVariable Long id) {

        return ResponseEntity.ok("Product ID: " + id);
    }
}
~~~

`ResponseEntity.ok()` returns a response with:

- Status: `200 OK`
- Body: `"Product ID: 101"` for `/api/products/101`

### Returning a Java Object

~~~java
@GetMapping("/{id}")
public ResponseEntity<ProductResponse> getProduct(
        @PathVariable Long id) {

    ProductResponse product = productService.getById(id);

    return ResponseEntity.ok(product);
}
~~~

If JSON is negotiated and an appropriate JSON message converter is available, Spring serializes the `ProductResponse` object into JSON.

### Why Use `ResponseEntity`?

- Return different status codes depending on the result.
- Set custom response headers.
- Return a response body or an empty response.
- Make the API's HTTP behavior explicit.
- Handle success, not-found, and conflict scenarios clearly.

**Interview one-liner:** `ResponseEntity<T>` represents an HTTP response with a status, headers, and an optional body.

---

## 3. Common Ways to Build a `ResponseEntity`

### `ResponseEntity.ok()`

Returns `200 OK`.

~~~java
return ResponseEntity.ok(product);
~~~

### `ResponseEntity.status()`

Sets an explicit status.

~~~java
return ResponseEntity.status(HttpStatus.CREATED)
        .body(product);
~~~

### `ResponseEntity.created()`

Creates a `201 Created` response and sets the `Location` header.

~~~java
URI location = URI.create("/api/products/" + product.getId());

return ResponseEntity.created(location)
        .body(product);
~~~

For production code, construct the location URI using an appropriate URI builder when paths or values require safe encoding.

### `ResponseEntity.noContent()`

Returns `204 No Content` without a response body.

~~~java
return ResponseEntity.noContent().build();
~~~

### `ResponseEntity.notFound()`

Returns `404 Not Found`.

~~~java
return ResponseEntity.notFound().build();
~~~

### `ResponseEntity.badRequest()`

Returns `400 Bad Request`.

~~~java
return ResponseEntity.badRequest()
        .body("Invalid product request");
~~~

### `ResponseEntity.accepted()`

Returns `202 Accepted`.

~~~java
return ResponseEntity.accepted()
        .body("Request accepted for processing");
~~~

Use `202 Accepted` when processing has been accepted but is not yet complete.

### Quick Reference

| Method | Status |
|---|---|
| `ok(body)` | 200 OK |
| `status(HttpStatus.CREATED).body(body)` | 201 Created |
| `created(uri).body(body)` | 201 Created |
| `noContent().build()` | 204 No Content |
| `notFound().build()` | 404 Not Found |
| `badRequest().body(body)` | 400 Bad Request |
| `accepted().body(body)` | 202 Accepted |

---

## 4. Setting Response Headers

`ResponseEntity` allows custom headers to be added to the response.

### Example

~~~java
@GetMapping("/{id}")
public ResponseEntity<ProductResponse> getProduct(
        @PathVariable Long id) {

    ProductResponse product = productService.getById(id);

    return ResponseEntity.ok()
            .header("X-Application", "Product-Service")
            .body(product);
}
~~~

The response contains:

- Status: `200 OK`
- Header: `X-Application: Product-Service`
- Body: The product representation

### Using `HttpHeaders`

~~~java
@GetMapping("/download")
public ResponseEntity<byte[]> downloadFile() {

    byte[] fileContent = fileService.getFile();

    HttpHeaders headers = new HttpHeaders();
    headers.setContentType(MediaType.APPLICATION_OCTET_STREAM);
    headers.setContentDisposition(
            ContentDisposition.attachment()
                    .filename("report.pdf")
                    .build());

    return new ResponseEntity<>(
            fileContent,
            headers,
            HttpStatus.OK);
}
~~~

This example illustrates setting headers for a file download. The actual content type should match the file being returned.

### Common Response Headers

| Header | Purpose |
|---|---|
| `Content-Type` | Describes the response body's media type |
| `Location` | Identifies a resource, commonly after creation |
| `Cache-Control` | Controls caching behavior |
| `ETag` | Provides a representation validator |
| `Content-Disposition` | Suggests inline display or attachment download |

**Interview trap:** `ResponseEntity` does not automatically configure caching, authentication, or other security policies. Those behaviors must be implemented through appropriate application and framework configuration.

---

## 5. What Are HTTP Status Codes?

HTTP status codes indicate the result of processing an HTTP request.

They are divided into five categories.

| Category | Range | Meaning |
|---|---|---|
| 1xx | 100–199 | Informational |
| 2xx | 200–299 | Success |
| 3xx | 300–399 | Redirection |
| 4xx | 400–499 | Client error |
| 5xx | 500–599 | Server error |

For REST API interviews, focus especially on commonly used 2xx, 4xx, and 5xx responses.

---

## 6. Important 2xx Success Status Codes

### `200 OK`

The request succeeded.

Common uses:

- Fetching a resource
- Returning search results
- Returning a successfully processed request with a representation

~~~java
@GetMapping("/{id}")
public ResponseEntity<ProductResponse> getProduct(
        @PathVariable Long id) {

    return ResponseEntity.ok(productService.getById(id));
}
~~~

### `201 Created`

A request successfully created a resource.

~~~java
@PostMapping
public ResponseEntity<ProductResponse> create(
        @RequestBody ProductRequest request) {

    ProductResponse created = productService.create(request);

    URI location = URI.create("/api/products/" + created.getId());

    return ResponseEntity.created(location).body(created);
}
~~~

A `201 Created` response commonly includes a `Location` header identifying the created resource.

### `202 Accepted`

The request has been accepted for processing, but processing has not necessarily completed.

Typical use case: accepting a long-running report-generation request for asynchronous processing.

### `204 No Content`

The request succeeded, and the response has no body.

~~~java
@DeleteMapping("/{id}")
public ResponseEntity<Void> deleteProduct(
        @PathVariable Long id) {

    productService.delete(id);

    return ResponseEntity.noContent().build();
}
~~~

**Interview trap:** `204 No Content` should not include a response body. Use `200 OK` if the API needs to return a result representation.

---

## 7. Important 4xx Client Error Status Codes

### `400 Bad Request`

The request cannot be processed because its syntax, parameters, or supplied data are invalid.

Typical causes:

- Malformed JSON
- Failed type conversion
- Missing required parameters
- Validation failures

### `401 Unauthorized`

The request lacks valid authentication credentials.

Despite the name, this status is associated with authentication, not authorization.

### `403 Forbidden`

The server understood the request but refuses to fulfill it.

The client may be authenticated but lack the required permission. Applications may also use 403 for other policy-based access denials.

### `404 Not Found`

The requested resource or matching handler was not found.

### `405 Method Not Allowed`

The request path is recognized, but the HTTP method is not supported for that resource.

Example: sending `POST` to an endpoint that only supports `GET`.

### `406 Not Acceptable`

The server cannot provide a response representation acceptable under the request's content-negotiation requirements.

### `409 Conflict`

The request conflicts with the current state of the resource.

Examples:

- Duplicate unique business identifier
- Attempt to create an entity that already exists
- Version conflict during an update

### `415 Unsupported Media Type`

The server refuses the request because the supplied request content type is unsupported.

Example: sending XML to an endpoint that only accepts JSON.

### `422 Unprocessable Content`

The content type and syntax may be understood, but the server cannot process the request's instructions or content.

Applications sometimes use this status for semantic validation errors. Many Spring APIs instead use `400 Bad Request` for validation failures, so follow a consistent API convention.

### `429 Too Many Requests`

The client has sent too many requests within a specified period.

Often used for rate limiting.

---

## 8. Important 5xx Server Error Status Codes

### `500 Internal Server Error`

An unexpected server-side failure occurred.

Typical causes include an unhandled application exception or an unexpected dependency failure that is not mapped to a more specific status.

### `502 Bad Gateway`

A gateway or proxy received an invalid response from an upstream server.

### `503 Service Unavailable`

The service is temporarily unable to handle the request.

Possible causes include temporary overload, maintenance, or an unavailable dependency.

### `504 Gateway Timeout`

A gateway or proxy did not receive a timely response from an upstream server.

### Quick Status Code Revision

| Status | Interview meaning |
|---|---|
| 200 | Successful request |
| 201 | Resource created |
| 202 | Accepted for processing |
| 204 | Success without response body |
| 400 | Invalid request |
| 401 | Authentication required or invalid |
| 403 | Access forbidden |
| 404 | Resource or handler not found |
| 405 | HTTP method unsupported |
| 406 | No acceptable response representation |
| 409 | Conflict with current state |
| 415 | Unsupported request media type |
| 422 | Unprocessable content |
| 429 | Too many requests |
| 500 | Unexpected server error |
| 502 | Invalid upstream response |
| 503 | Service unavailable |
| 504 | Upstream timeout |

**Interview trap:** Use status codes consistently. Do not return `200 OK` for every outcome, including failures.

---

## 9. `ResponseEntity` vs `@ResponseStatus`

Both can be used to control HTTP status codes, but their usage differs.

### `@ResponseStatus`

Declares a status for a controller method or an exception class.

~~~java
@PostMapping
@ResponseStatus(HttpStatus.CREATED)
public ProductResponse create(
        @RequestBody ProductRequest request) {

    return productService.create(request);
}
~~~

The method returns a `ProductResponse` body with the declared success status.

### `ResponseEntity`

Allows the status, headers, and body to be selected programmatically.

~~~java
@PostMapping
public ResponseEntity<ProductResponse> create(
        @RequestBody ProductRequest request) {

    ProductResponse created = productService.create(request);

    return ResponseEntity.status(HttpStatus.CREATED)
            .body(created);
}
~~~

### Difference

| Feature | `@ResponseStatus` | `ResponseEntity` |
|---|---|---|
| Declare status | Yes | Yes |
| Set headers dynamically | Not by itself | Yes |
| Select status based on runtime result | Limited | Yes |
| Return a body | Yes, through normal return-value handling | Yes |
| Common use | Fixed status | Conditional or customized responses |

**Interview one-liner:** Use `@ResponseStatus` for a fixed status and `ResponseEntity` when response status, headers, or body need programmatic control.

---

## 10. What Is a DTO?

DTO stands for **Data Transfer Object**.

A DTO is an object designed to carry data between application layers or across a process boundary, such as an HTTP API.

In Spring Boot REST APIs, DTOs commonly represent request and response payloads.

Examples:

- `ProductRequest`
- `ProductResponse`
- `UserRegistrationRequest`
- `OrderResponse`

### Why Use DTOs?

- Separate the public API contract from database entities.
- Expose only the fields required by the client.
- Avoid accidentally exposing sensitive fields.
- Validate input independently of persistence rules.
- Support different request and response structures.
- Reduce coupling between the API and database schema.
- Avoid serialization problems involving entity relationships.
- Keep API versioning and compatibility easier to manage.

### Example Without DTOs

~~~java
@GetMapping("/{id}")
public Product getProduct(@PathVariable Long id) {
    return productRepository.findById(id).orElseThrow();
}
~~~

If `Product` is a JPA entity, returning it directly may expose internal fields or cause serialization issues involving relationships.

### Example With DTOs

~~~java
@GetMapping("/{id}")
public ProductResponse getProduct(@PathVariable Long id) {
    return productService.getById(id);
}
~~~

The response exposes only the intended API fields.

**Interview one-liner:** A DTO defines the data structure exchanged between components or systems without requiring the internal domain or persistence model to match the external contract.

---

## 11. Entity vs DTO

An entity and a DTO serve different purposes.

### Entity

A JPA entity represents persisted domain data and participates in ORM mapping.

~~~java
@Entity
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    private BigDecimal price;

    private String internalCode;

    // Getters and setters
}
~~~

The entity may contain persistence-specific annotations and internal fields.

### DTO

A response DTO defines the fields exposed by the API.

~~~java
public record ProductResponse(
        Long id,
        String name,
        BigDecimal price) {
}
~~~

The response does not expose `internalCode`.

### Comparison

| Feature | Entity | DTO |
|---|---|---|
| Primary purpose | Represent persisted/domain data | Transfer data |
| Persistence annotations | Commonly present | Usually absent |
| Database schema coupling | Often higher | Lower |
| API contract | Not necessarily suitable | Explicitly designed for the API |
| Sensitive fields | May exist internally | Expose only if required |
| Validation | Persistence/domain constraints may apply | API input constraints can be defined |
| Serialization | Can expose entity relationships | Usually easier to control |

**Interview trap:** A DTO is not automatically secure merely because it is a DTO. The developer must still decide which fields to include and validate incoming values.

---

## 12. Request DTO vs Response DTO

Request and response DTOs often represent different data.

### Request DTO

Contains fields that a client is allowed to submit.

~~~java
public record ProductRequest(
        @NotBlank String name,
        @NotNull @Positive BigDecimal price) {
}
~~~

### Response DTO

Contains fields returned to the client.

~~~java
public record ProductResponse(
        Long id,
        String name,
        BigDecimal price) {
}
~~~

### Why Separate Them?

A client should not normally decide database-generated identifiers or internal system fields during resource creation.

A request may contain:

- `name`
- `price`

A response may contain:

- `id`
- `name`
- `price`

Different operations may require different DTOs. For example, an update request might contain only fields that are allowed to change.

**Interview one-liner:** Request DTOs define accepted input; response DTOs define the data the API exposes to clients.

---

## 13. Mapping Between Entity and DTO

Mapping converts an entity into a response DTO or a request DTO into an entity/domain object.

### Manual Mapping

~~~java
public ProductResponse toResponse(Product product) {
    return new ProductResponse(
            product.getId(),
            product.getName(),
            product.getPrice());
}
~~~

Manual mapping is straightforward and makes the mapping explicit.

### Service Example

~~~java
@Service
public class ProductService {

    private final ProductRepository productRepository;

    public ProductService(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }

    public ProductResponse getById(Long id) {
        Product product = productRepository.findById(id)
                .orElseThrow(() ->
                        new ProductNotFoundException(id));

        return new ProductResponse(
                product.getId(),
                product.getName(),
                product.getPrice());
    }
}
~~~

This example assumes `ProductNotFoundException` is an application-defined exception.

### Mapping Request to Entity

~~~java
public Product toEntity(ProductRequest request) {
    Product product = new Product();
    product.setName(request.name());
    product.setPrice(request.price());
    return product;
}
~~~

The server controls fields such as the database ID and internal properties instead of blindly copying all client-supplied values.

### Mapping Libraries

For larger applications, mapping libraries such as MapStruct can reduce repetitive mapping code.

Manual mapping is still useful for small DTOs and for mappings that involve business rules.

**Interview trap:** DTO mapping is not the same as JSON serialization. Mapping transforms one Java object into another; serialization converts an object into a representation such as JSON.

---

## 14. Complete Example: Create a Product

This example demonstrates a request DTO, validation, service logic, a response DTO, and `ResponseEntity`.

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

### Response DTO

~~~java
public record ProductResponse(
        Long id,
        String name,
        BigDecimal price) {
}
~~~

### Controller

~~~java
@RestController
@RequestMapping("/api/products")
public class ProductController {

    private final ProductService productService;

    public ProductController(ProductService productService) {
        this.productService = productService;
    }

    @PostMapping
    public ResponseEntity<ProductResponse> create(
            @Valid @RequestBody ProductRequest request) {

        ProductResponse created = productService.create(request);

        URI location = URI.create("/api/products/" + created.id());

        return ResponseEntity.created(location).body(created);
    }

    @GetMapping("/{id}")
    public ResponseEntity<ProductResponse> getById(
            @PathVariable Long id) {

        return ResponseEntity.ok(productService.getById(id));
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        productService.delete(id);
        return ResponseEntity.noContent().build();
    }
}
~~~

### Expected Create Request

~~~http
POST /api/products
Content-Type: application/json
~~~

~~~json
{
  "name": "Keyboard",
  "price": 2500
}
~~~

### Expected Response

Status:

`201 Created`

Headers include:

`Location: /api/products/101`

Body:

~~~json
{
  "id": 101,
  "name": "Keyboard",
  "price": 2500
}
~~~

The ID and response values above are illustrative.

### Request Flow

1. The client sends JSON.
2. Spring maps the request to the controller.
3. `@RequestBody` deserializes JSON into `ProductRequest`.
4. `@Valid` triggers validation.
5. The controller delegates to `ProductService`.
6. The service applies business logic and persists the product.
7. The service maps the saved entity to `ProductResponse`.
8. The controller returns `ResponseEntity.created(...)`.
9. Spring writes the status, headers, and serialized body to the HTTP response.

---

## 15. Handling Not-Found Scenarios

A REST API should return an appropriate status when a requested resource does not exist.

### Using `ResponseEntity`

~~~java
@GetMapping("/{id}")
public ResponseEntity<ProductResponse> getById(
        @PathVariable Long id) {

    return productService.findById(id)
            .map(ResponseEntity::ok)
            .orElseGet(() -> ResponseEntity.notFound().build());
}
~~~

This assumes the service returns `Optional<ProductResponse>`.

### Using a Custom Exception

~~~java
public class ProductNotFoundException extends RuntimeException {

    public ProductNotFoundException(Long id) {
        super("Product not found: " + id);
    }
}
~~~

The service can throw this exception when the product is missing.

A global exception handler can convert it to `404 Not Found`.

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

Example error DTO:

~~~java
public record ApiError(
        String code,
        String message) {
}
~~~

This keeps error response formatting consistent across endpoints.

See: [01. Exception Handling](../06-Exception-Handling/01-Exception-Handling.md)

---

## 16. Designing Consistent Error Responses

An API should provide predictable error responses instead of exposing raw exception details.

### Example Error DTO

~~~java
public record ApiError(
        String code,
        String message) {
}
~~~

Possible JSON response:

~~~json
{
  "code": "PRODUCT_NOT_FOUND",
  "message": "Product not found: 101"
}
~~~

### Better Error Response Fields

Depending on the API's needs, an error representation may contain:

- `code`: Stable application-specific error code.
- `message`: Human-readable description.
- `timestamp`: Time of the error.
- `path`: Request path.
- `fieldErrors`: Validation errors associated with individual fields.
- `traceId`: Correlation identifier for support and troubleshooting.

Do not expose stack traces, database credentials, internal SQL, or other sensitive implementation details in public responses.

### Status Code vs Error Code

HTTP status codes describe the general outcome of the request.

Application error codes identify specific business or application errors.

For example:

- HTTP status: `404`
- Application code: `PRODUCT_NOT_FOUND`

Several application errors may share the same HTTP status while retaining distinct application codes.

---

## 17. DTO Validation and Error Handling

Request DTOs can define validation constraints, and the controller can trigger them using `@Valid`.

~~~java
public record ProductRequest(
        @NotBlank String name,
        @NotNull @Positive BigDecimal price) {
}
~~~

Controller:

~~~java
@PostMapping
public ResponseEntity<ProductResponse> create(
        @Valid @RequestBody ProductRequest request) {

    ProductResponse created = productService.create(request);

    return ResponseEntity.status(HttpStatus.CREATED).body(created);
}
~~~

If validation fails, Spring MVC typically raises `MethodArgumentNotValidException` for an invalid request body.

A global exception handler can return a consistent response structure.

~~~java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiError> handleValidation(
            MethodArgumentNotValidException ex) {

        ApiError error = new ApiError(
                "VALIDATION_FAILED",
                "One or more request fields are invalid");

        return ResponseEntity.badRequest().body(error);
    }
}
~~~

This is a simplified example. A production handler can extract field-level errors and return them in a dedicated list.

**Interview trap:** `@Valid` triggers validation; it does not define the response format. Exception handling determines how validation failures are represented to the client.

---

## 18. `ResponseEntity<T>` and Generic Types

`ResponseEntity<T>` uses a generic type parameter to represent the response body type.

Examples:

~~~java
ResponseEntity<ProductResponse>
ResponseEntity<List<ProductResponse>>
ResponseEntity<String>
ResponseEntity<Void>
~~~

Meaning:

- `ResponseEntity<ProductResponse>`: Response body is a product DTO.
- `ResponseEntity<List<ProductResponse>>`: Response body is a list of product DTOs.
- `ResponseEntity<String>`: Response body is text.
- `ResponseEntity<Void>`: No meaningful response body is returned.

Example:

~~~java
@GetMapping
public ResponseEntity<List<ProductResponse>> getAll() {
    return ResponseEntity.ok(productService.findAll());
}
~~~

The generic type provides compile-time type safety and makes the intended response body explicit.

**Interview trap:** `ResponseEntity<T>` does not itself serialize the body. Spring MVC's return-value handling and message converters produce the HTTP representation.

---

## 19. `ResponseEntity` vs Returning an Object Directly

### Direct Return

~~~java
@GetMapping("/{id}")
public ProductResponse getById(@PathVariable Long id) {
    return productService.getById(id);
}
~~~

With `@RestController`, Spring normally serializes the returned DTO and uses `200 OK` when the method completes successfully, unless the response handling is otherwise configured.

### `ResponseEntity`

~~~java
@GetMapping("/{id}")
public ResponseEntity<ProductResponse> getById(
        @PathVariable Long id) {

    ProductResponse product = productService.getById(id);

    return ResponseEntity.ok(product);
}
~~~

### Comparison

| Feature | Direct DTO return | `ResponseEntity<T>` |
|---|---|---|
| Return response body | Yes | Yes |
| Set explicit status | Requires another mechanism | Yes |
| Set headers | Requires another mechanism | Yes |
| Simple successful response | Concise | Slightly more explicit |
| Conditional response status | Less convenient | Convenient |

Use direct DTO returns for straightforward responses with default status and headers. Use `ResponseEntity` when explicit HTTP response control improves clarity.

---

## 20. HTTP Status Code Selection Scenarios

| Scenario | Typical status |
|---|---|
| Fetch a product successfully | 200 OK |
| Create a product successfully | 201 Created |
| Accept an asynchronous job | 202 Accepted |
| Delete a product with no body | 204 No Content |
| Malformed JSON request | 400 Bad Request |
| Missing or invalid authentication | 401 Unauthorized |
| Authenticated user lacks permission | 403 Forbidden |
| Product does not exist | 404 Not Found |
| Unsupported HTTP method | 405 Method Not Allowed |
| Duplicate product conflicts with a business rule | 409 Conflict |
| Unsupported request content type | 415 Unsupported Media Type |
| Rate limit exceeded | 429 Too Many Requests |
| Unexpected application error | 500 Internal Server Error |
| Temporary service outage | 503 Service Unavailable |

The appropriate response depends on the API contract and the actual failure. For example, a duplicate resource might be represented as `409 Conflict` or handled idempotently depending on the operation's semantics.

---

## 21. Frequently Asked Interview Questions

### Q1. What is `ResponseEntity`?

`ResponseEntity<T>` represents an HTTP response with a status code, headers, and an optional body.

### Q2. What is the difference between `ResponseEntity` and `@ResponseStatus`?

`@ResponseStatus` declares a status, whereas `ResponseEntity` allows programmatic control over status, headers, and body.

### Q3. What is the difference between 401 and 403?

401 indicates missing or invalid authentication credentials; 403 indicates that the server refuses access, commonly because the client lacks permission.

### Q4. What is the difference between 200 and 201?

200 indicates successful processing. 201 indicates that a resource was created.

### Q5. When should 204 No Content be returned?

When the request succeeds and the API intentionally returns no response body.

### Q6. What is a DTO?

A DTO is an object designed to carry data between components or across a system boundary.

### Q7. Why should DTOs be used instead of entities in REST APIs?

DTOs isolate the API contract from persistence models, control exposed fields, reduce coupling, and make request validation and response design easier.

### Q8. What is the difference between a request DTO and a response DTO?

A request DTO defines accepted input; a response DTO defines the data exposed to the client.

### Q9. What is the difference between entity-to-DTO mapping and serialization?

Mapping transforms one Java object into another. Serialization converts an object into a representation such as JSON.

### Q10. Can `ResponseEntity` return an empty response?

Yes. For example, `ResponseEntity.noContent().build()` returns `204 No Content` without a body.

### Q11. Why is the `Location` header useful with 201 Created?

It identifies the newly created resource so the client can retrieve or reference it.

### Q12. Does returning `ResponseEntity` automatically convert an object into JSON?

No. Spring MVC uses an appropriate message converter to serialize the response body.

### Q13. What happens when validation fails on a request DTO?

Spring MVC typically raises a validation exception, and the configured exception-handling mechanism determines the HTTP response.

### Q14. Why should error responses be consistent?

Consistent error formats simplify client-side handling, monitoring, debugging, and API integration.

### Q15. Should a controller always return `ResponseEntity`?

No. A controller can return a DTO directly when the default status and headers are sufficient. Use `ResponseEntity` when explicit control is useful.

### Q16. What is the difference between 400 and 422?

400 broadly indicates a bad request. 422 indicates that the content is understood but cannot be processed semantically. The exact convention depends on the API design.

### Q17. What is the difference between 404 and 409?

404 means the requested resource or handler was not found. 409 means the request conflicts with the current resource state.

### Q18. Can a DTO contain validation annotations?

Yes. Request DTOs commonly use Bean Validation constraints to define acceptable input.

---

## 22. Common Mistakes and Interview Traps

- Returning `200 OK` for every outcome, including failures.
- Returning `201 Created` when no resource was actually created and the operation does not otherwise justify that status.
- Returning a body with `204 No Content`.
- Confusing 401 authentication failures with 403 access denials.
- Confusing 404 Not Found with 409 Conflict.
- Returning JPA entities directly without considering API boundaries and serialization.
- Exposing passwords, internal identifiers, or sensitive fields in response DTOs.
- Assuming a DTO is automatically secure.
- Binding client input directly to all fields of a persistence entity.
- Using the same DTO for every request and response without considering different contracts.
- Confusing DTO mapping with JSON serialization.
- Assuming `ResponseEntity` performs business logic or persistence.
- Assuming `@Valid` automatically creates a consistent error body.
- Returning stack traces or internal exception details to API clients.
- Forgetting to set a useful `Location` header when appropriate after resource creation.
- Choosing HTTP status codes inconsistently across endpoints.

---

## 23. Quick Revision

| Concept | Remember |
|---|---|
| `ResponseEntity<T>` | Controls status, headers, and body |
| `ResponseEntity.ok()` | 200 OK |
| `ResponseEntity.created(uri)` | 201 Created with Location |
| `ResponseEntity.accepted()` | 202 Accepted |
| `ResponseEntity.noContent()` | 204 No Content |
| `ResponseEntity.notFound()` | 404 Not Found |
| `@ResponseStatus` | Declares a status for a handler or exception |
| DTO | Data Transfer Object |
| Request DTO | Defines accepted input |
| Response DTO | Defines output data |
| Entity | Represents persisted/domain data |
| Mapping | Converts one Java object into another |
| Serialization | Converts an object to JSON or another representation |
| 400 | Invalid request |
| 401 | Authentication failure or missing credentials |
| 403 | Access forbidden |
| 404 | Resource or handler not found |
| 409 | State conflict |
| 415 | Unsupported request media type |
| 500 | Unexpected server error |

### One-Line Interview Answers

- **`ResponseEntity`:** Represents an HTTP response containing status, headers, and an optional body.
- **DTO:** Defines a data contract for transferring information between components or systems.
- **Entity vs DTO:** An entity models persisted/domain data; a DTO models transferred data.
- **Request DTO:** Defines the input accepted by an API operation.
- **Response DTO:** Defines the data exposed by an API response.
- **201 Created:** Indicates that a resource was created successfully.
- **204 No Content:** Indicates success without a response body.
- **401 Unauthorized:** Indicates missing or invalid authentication credentials.
- **403 Forbidden:** Indicates that the server refuses access.
- **409 Conflict:** Indicates a conflict with the current resource state.
- **`@ResponseStatus`:** Declares an HTTP status for a handler or exception.
- **`Location` header:** Identifies a resource, commonly the one created by a request.
- **DTO mapping:** Transforms one Java object into another.
- **Serialization:** Converts a Java object into a response representation such as JSON.

---

## 24. Related Topics

- [01. Spring MVC Architecture & Request Flow](01-Spring-MVC-Architecture.md)
- [02. Controllers & Request Mapping](02-Controllers-and-Mappings.md)
- [03. Path Variables, Request Params, Headers & Request Body](03-Request-Parameters-and-Body.md)
- [05. Validation & Jackson](05-Validation-and-Jackson.md)
- [06. REST API Design & Clients](06-REST-API-Design-and-Clients.md)
- [01. Exception Handling](../06-Exception-Handling/01-Exception-Handling.md)
- [Spring Data JPA & Hibernate](../07-Spring-Data-JPA/01-Spring-Data-JPA-and-Hibernate-Overview.md)