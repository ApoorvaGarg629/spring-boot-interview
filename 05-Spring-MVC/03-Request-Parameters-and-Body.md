# 03. Path Variables, Request Params, Headers & Request Body

## 1. Overview of Request Data Binding

Spring MVC provides annotations to extract data from different parts of an HTTP request and bind it to controller method parameters.

| Annotation | Data source | Typical use |
|---|---|---|
| `@PathVariable` | URL path | Identify a specific resource |
| `@RequestParam` | Request parameters | Filtering, pagination, sorting |
| `@RequestHeader` | HTTP headers | Read metadata sent by the client |
| `@CookieValue` | Cookies | Read cookie values |
| `@RequestBody` | HTTP request body | Bind JSON/XML to a Java object |
| `@ModelAttribute` | Request parameters and form data | Bind form submissions to an object |
| `@RequestPart` | Individual multipart part | Upload files and process multipart requests |

Example:

~~~java
@RestController
@RequestMapping("/api/products")
public class ProductController {

    @GetMapping("/{id}")
    public String getProduct(
            @PathVariable Long id,
            @RequestParam(defaultValue = "false") boolean details,
            @RequestHeader(value = "X-Request-Id", required = false)
            String requestId) {

        return "Product ID: " + id;
    }
}
~~~

For a request such as:

`GET /api/products/101?details=true`

Spring extracts `id = 101` and `details = true`. If the client sends `X-Request-Id`, Spring can bind it to `requestId`.

**Interview one-liner:** Spring MVC uses argument annotations to bind URL values, request parameters, headers, cookies, and request bodies to controller method arguments.

---

## 2. `@PathVariable`

`@PathVariable` extracts values embedded in the URL path.

### Basic Example

~~~java
@GetMapping("/products/{id}")
public ProductResponse getProduct(@PathVariable Long id) {
    return productService.getById(id);
}
~~~

Request:

`GET /products/101`

Here, Spring extracts `101` from the path and converts it to `Long`.

### Multiple Path Variables

~~~java
@GetMapping("/users/{userId}/orders/{orderId}")
public OrderResponse getOrder(
        @PathVariable Long userId,
        @PathVariable Long orderId) {

    return orderService.getOrder(userId, orderId);
}
~~~

Request:

`GET /users/10/orders/500`

Values:

- `userId = 10`
- `orderId = 500`

### Explicit Variable Name

~~~java
@GetMapping("/products/{productId}")
public ProductResponse getProduct(
        @PathVariable("productId") Long id) {

    return productService.getById(id);
}
~~~

The URL variable is named `productId`, while the Java method parameter is named `id`.

Using an explicit name avoids depending on Java parameter-name metadata.

### Optional Path Variables

Path variables are required by default because they form part of the matched URL.

If a value is optional, define separate mappings or redesign the URL.

~~~java
@GetMapping({"/products", "/products/{id}"})
public Object getProducts(
        @PathVariable(required = false) Long id) {

    if (id == null) {
        return productService.findAll();
    }

    return productService.getById(id);
}
~~~

This pattern can work, but separate handler methods are often clearer:

~~~java
@GetMapping("/products")
public List<ProductResponse> getProducts() {
    return productService.findAll();
}

@GetMapping("/products/{id}")
public ProductResponse getProduct(@PathVariable Long id) {
    return productService.getById(id);
}
~~~

**Interview trap:** Do not use an optional path variable as a substitute for designing clear resource URLs. Separate mappings are generally easier to maintain.

### Type Conversion

Spring automatically converts compatible path values into target Java types.

~~~java
@GetMapping("/products/{id}")
public ProductResponse getProduct(@PathVariable Long id) {
    return productService.getById(id);
}
~~~

- `/products/101`: Conversion succeeds.
- `/products/abc`: Conversion to `Long` fails, typically resulting in a 400 Bad Request response.

### When to Use `@PathVariable`

Use it when the value identifies a resource or a particular element in a resource hierarchy.

Examples:

- `/users/10`
- `/orders/500`
- `/departments/3/employees/25`

---

## 3. `@RequestParam`

`@RequestParam` binds request parameters to controller method arguments.

It is commonly used for query parameters and form parameters.

### Basic Example

~~~java
@GetMapping("/products")
public List<ProductResponse> getProducts(
        @RequestParam String category) {

    return productService.findByCategory(category);
}
~~~

Request:

`GET /products?category=electronics`

Value:

`category = "electronics"`

### Multiple Request Parameters

~~~java
@GetMapping("/products")
public List<ProductResponse> getProducts(
        @RequestParam String category,
        @RequestParam int page,
        @RequestParam int size) {

    return productService.findProducts(category, page, size);
}
~~~

Request:

`GET /products?category=electronics&page=0&size=20`

### Required Parameters

`@RequestParam` is required by default.

~~~java
@GetMapping("/products")
public List<ProductResponse> getProducts(
        @RequestParam String category) {

    return productService.findByCategory(category);
}
~~~

If `category` is missing, Spring typically raises a missing-request-parameter exception, resulting in a 400 response unless customized.

### Optional Parameters

Use `required = false` when the parameter may be absent.

~~~java
@GetMapping("/products")
public List<ProductResponse> getProducts(
        @RequestParam(required = false) String category) {

    return productService.search(category);
}
~~~

When the parameter is absent, `category` can be `null`.

Use a wrapper type such as `Integer` for optional numeric values rather than a primitive `int`.

### Default Values

~~~java
@GetMapping("/products")
public List<ProductResponse> getProducts(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "10") int size) {

    return productService.findPage(page, size);
}
~~~

Request:

`GET /products`

Values:

- `page = 0`
- `size = 10`

A `defaultValue` also makes the parameter optional and is used when the parameter is missing or empty.

### Renaming a Request Parameter

~~~java
@GetMapping("/products")
public ProductResponse getProduct(
        @RequestParam("product_id") Long id) {

    return productService.getById(id);
}
~~~

Request:

`GET /products?product_id=101`

### Binding Multiple Values

~~~java
@GetMapping("/products")
public List<ProductResponse> getProducts(
        @RequestParam List<String> category) {

    return productService.findByCategories(category);
}
~~~

Request:

`GET /products?category=books&category=electronics`

Spring can bind repeated parameters to a collection.

Comma-separated values can also be converted to collections in supported conversion scenarios, for example:

`GET /products?category=books,electronics`

### Map-Based Binding

~~~java
@GetMapping("/search")
public Map<String, String> search(
        @RequestParam Map<String, String> params) {

    return params;
}
~~~

This captures request parameters in a map. For repeated parameter names or more advanced multi-value handling, use `MultiValueMap<String, String>`.

### `params` in `@RequestMapping` vs `@RequestParam`

These are different concepts.

`@RequestParam` extracts a parameter value:

~~~java
@GetMapping("/products")
public List<ProductResponse> getProducts(
        @RequestParam String category) {

    return productService.findByCategory(category);
}
~~~

The `params` mapping condition selects a handler based on parameter presence or value:

~~~java
@GetMapping(value = "/products", params = "active=true")
public List<ProductResponse> getActiveProducts() {
    return productService.findActive();
}
~~~

**Interview one-liner:** `@RequestParam` binds a request parameter to a method argument, while the `params` attribute of a mapping annotation uses parameter conditions to select a handler.

---

## 4. `@PathVariable` vs `@RequestParam`

| Feature | `@PathVariable` | `@RequestParam` |
|---|---|---|
| Source | URL path segment | Request parameter |
| Example | `/products/101` | `/products?id=101` |
| Typical purpose | Identify a resource | Filter, sort, paginate |
| Required by default | Yes | Yes |
| Optional handling | Usually use separate mappings | `required = false` or `defaultValue` |
| Common HTTP usage | Resource-oriented paths | Query strings and form parameters |

### Example

~~~java
@GetMapping("/products/{id}")
public ProductResponse getProduct(@PathVariable Long id) {
    return productService.getById(id);
}

@GetMapping("/products")
public List<ProductResponse> getProducts(
        @RequestParam(required = false) String category,
        @RequestParam(defaultValue = "0") int page) {

    return productService.search(category, page);
}
~~~

Use `/products/101` to identify a product and `/products?category=books&page=0` to search or filter products.

**Interview answer:** Path variables commonly identify resources; request parameters commonly provide optional filters, pagination, sorting, and other query criteria. This is a design convention, not an absolute technical restriction.

---

## 5. `@RequestHeader`

`@RequestHeader` extracts a value from an HTTP request header.

Headers typically contain request metadata, authentication information, content negotiation preferences, and tracing identifiers.

### Basic Example

~~~java
@GetMapping("/profile")
public String getProfile(
        @RequestHeader("X-Request-Id") String requestId) {

    return "Request ID: " + requestId;
}
~~~

Request header:

`X-Request-Id: abc-123`

### Optional Header

~~~java
@GetMapping("/profile")
public String getProfile(
        @RequestHeader(
            value = "X-Request-Id",
            required = false
        ) String requestId) {

    return requestId == null ? "No request ID" : requestId;
}
~~~

### Default Header Value

~~~java
@GetMapping("/profile")
public String getProfile(
        @RequestHeader(
            value = "X-Client",
            defaultValue = "web"
        ) String client) {

    return "Client: " + client;
}
~~~

If the header is missing or empty, the default value is used.

### Read All Headers

~~~java
@GetMapping("/headers")
public Map<String, String> getHeaders(
        @RequestHeader Map<String, String> headers) {

    return headers;
}
~~~

For headers that may have multiple values, use `HttpHeaders` or `MultiValueMap<String, String>` where appropriate.

### Common Headers

| Header | Purpose |
|---|---|
| `Authorization` | Carries credentials or a bearer token |
| `Accept` | Indicates acceptable response media types |
| `Content-Type` | Describes the request body's media type |
| `Accept-Language` | Indicates language preferences |
| `User-Agent` | Identifies the client software |
| `X-Request-Id` | Custom request correlation identifier |

Header names are case-insensitive according to HTTP semantics.

**Interview trap:** Reading an `Authorization` header does not authenticate or authorize a request by itself. Use Spring Security or an appropriate security mechanism to validate credentials and enforce access rules.

---

## 6. `@CookieValue`

`@CookieValue` extracts a cookie value from the HTTP request.

### Basic Example

~~~java
@GetMapping("/preferences")
public String getPreferences(
        @CookieValue("theme") String theme) {

    return "Theme: " + theme;
}
~~~

If the request contains a cookie named `theme`, Spring binds its value to the method argument.

### Optional Cookie

~~~java
@GetMapping("/preferences")
public String getPreferences(
        @CookieValue(
            value = "theme",
            defaultValue = "light"
        ) String theme) {

    return "Theme: " + theme;
}
~~~

If the cookie is absent or empty, the default value is used.

### Common Uses

- User preferences
- Session identifiers
- Client-side state
- Application-specific tracking

**Security note:** Do not trust a cookie simply because it came from the client. Sensitive cookies should be protected appropriately, and session/authentication handling should follow secure configuration practices.

---

## 7. `@RequestBody`

`@RequestBody` binds the HTTP request body to a Java object.

For JSON requests, Spring MVC typically uses a Jackson-based `HttpMessageConverter` to deserialize JSON into a Java object.

### Basic Example

~~~java
@PostMapping("/products")
public ProductResponse createProduct(
        @RequestBody ProductRequest request) {

    return productService.create(request);
}
~~~

Request:

~~~http
POST /products
Content-Type: application/json
~~~

JSON body:

~~~json
{
  "name": "Keyboard",
  "price": 2500
}
~~~

Spring deserializes the JSON into a `ProductRequest` instance and passes it to the controller method.

### DTO Example

~~~java
public class ProductRequest {

    private String name;
    private BigDecimal price;

    public ProductRequest() {
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public BigDecimal getPrice() {
        return price;
    }

    public void setPrice(BigDecimal price) {
        this.price = price;
    }
}
~~~

The exact requirements depend on the configured JSON library, object construction strategy, and DTO design.

### How `@RequestBody` Works Internally

1. The client sends a request containing a body.
2. `DispatcherServlet` identifies the matching controller method.
3. Spring resolves the method arguments.
4. `RequestResponseBodyMethodProcessor` handles the `@RequestBody` argument.
5. An appropriate `HttpMessageConverter` reads the body.
6. The converter deserializes the content into the requested Java type.
7. Spring invokes the controller method with the bound object.

### `Content-Type` Matters

For JSON requests, the client normally sends:

`Content-Type: application/json`

If the request body uses an unsupported media type, Spring can return **415 Unsupported Media Type**.

If the body is malformed or cannot be deserialized into the target type, the request typically results in **400 Bad Request**, unless error handling is customized.

### Required vs Optional Body

By default, `@RequestBody` is required.

~~~java
@PostMapping("/products")
public ProductResponse create(
        @RequestBody(required = false) ProductRequest request) {

    if (request == null) {
        throw new IllegalArgumentException("Request body is required");
    }

    return productService.create(request);
}
~~~

`required = false` allows a missing body, but the application must decide what to do when the resulting argument is `null`.

For ordinary creation endpoints, a required request body is usually more appropriate.

### Common Mistakes

- Sending JSON without the correct `Content-Type`.
- Using a request DTO with incompatible property types.
- Sending malformed JSON.
- Assuming `@RequestBody` automatically performs validation.
- Exposing persistence entities directly instead of using API DTOs.
- Assuming the controller must manually parse the JSON string.

**Interview one-liner:** `@RequestBody` delegates request-body conversion to an appropriate `HttpMessageConverter`, such as Jackson for JSON.

---

## 8. `@RequestBody` vs `@RequestParam`

| Feature | `@RequestBody` | `@RequestParam` |
|---|---|---|
| Data source | HTTP request body | Request parameters |
| Typical format | JSON, XML, or another supported body format | Query parameters or form parameters |
| Conversion | `HttpMessageConverter` | Spring type conversion and binding |
| Common use | Create/update API data | Filters, pagination, sorting |
| Example | `{"name":"Keyboard"}` | `?page=0&size=10` |

### Example

~~~java
@PostMapping("/products")
public ProductResponse create(
        @RequestBody ProductRequest request,
        @RequestParam(defaultValue = "false") boolean notify) {

    return productService.create(request, notify);
}
~~~

Request:

`POST /products?notify=true`

Body:

~~~json
{
  "name": "Keyboard",
  "price": 2500
}
~~~

The body binds to `ProductRequest`, while `notify` binds from the request parameter.

---

## 9. `@ModelAttribute`

`@ModelAttribute` binds request parameters and supported request values to an object. It is commonly used for traditional MVC form handling.

### Example

~~~java
public class ProductForm {

    private String name;
    private BigDecimal price;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public BigDecimal getPrice() {
        return price;
    }

    public void setPrice(BigDecimal price) {
        this.price = price;
    }
}
~~~

Controller:

~~~java
@Controller
@RequestMapping("/products")
public class ProductFormController {

    @PostMapping("/create")
    public String create(
            @ModelAttribute ProductForm form) {

        productService.createFromForm(form);
        return "redirect:/products";
    }
}
~~~

Form parameters such as `name=Keyboard&price=2500` can be bound to the object.

### `@ModelAttribute` vs `@RequestBody`

| Feature | `@ModelAttribute` | `@RequestBody` |
|---|---|---|
| Typical input | Form fields and request parameters | JSON/XML request body |
| Binding mechanism | Web data binding | `HttpMessageConverter` |
| Common use | Server-rendered MVC forms | REST APIs |
| Model integration | Can expose object in MVC model | Does not automatically add object to model |

### Security Consideration: Mass Assignment

Spring data binding can populate writable properties of an object. Binding client input directly to a persistence entity can expose fields that should not be modified by the client.

Prefer dedicated form objects or DTOs containing only the fields the client is allowed to set. Where necessary, restrict bindable fields and validate input.

**Interview trap:** `@ModelAttribute` and `@RequestBody` are not interchangeable; they use different binding mechanisms and are designed for different request formats.

---

## 10. `@RequestPart` and Multipart Requests

`@RequestPart` binds a specific part of a multipart request. It is useful when an endpoint accepts files alongside structured data.

### Example: File Upload

~~~java
@PostMapping(
    value = "/documents",
    consumes = MediaType.MULTIPART_FORM_DATA_VALUE
)
public String uploadDocument(
        @RequestPart("file") MultipartFile file) {

    return documentService.store(file);
}
~~~

The client submits a multipart request containing a part named `file`.

### File Plus JSON Metadata

~~~java
@PostMapping(
    value = "/documents",
    consumes = MediaType.MULTIPART_FORM_DATA_VALUE
)
public DocumentResponse uploadDocument(
        @RequestPart("file") MultipartFile file,
        @RequestPart("metadata") DocumentMetadata metadata) {

    return documentService.store(file, metadata);
}
~~~

The multipart request contains:

- `file`: The uploaded document.
- `metadata`: JSON metadata converted into a `DocumentMetadata` object.

The metadata part must use an appropriate media type, such as `application/json`, for JSON conversion.

### `@RequestPart` vs `@RequestParam`

- `@RequestPart` is designed for individual multipart parts and can use message converters for structured parts.
- `@RequestParam` can bind request parameters and multipart form fields, including files in supported scenarios.
- Use `@RequestPart` when a multipart part needs structured deserialization, such as JSON metadata.

**Interview trap:** File uploads require appropriate multipart configuration and size limits. Never assume every uploaded file is safe based only on its filename or supplied content type.

---

## 11. Type Conversion and Data Binding

Spring MVC automatically converts supported incoming values into the declared Java types.

Examples:

~~~java
@GetMapping("/products/{id}")
public ProductResponse getProduct(@PathVariable Long id) {
    return productService.getById(id);
}

@GetMapping("/search")
public ProductResponse search(
        @RequestParam int page,
        @RequestParam boolean active) {

    return productService.search(page, active);
}
~~~

Spring converts compatible strings into `Long`, `int`, and `boolean`.

### Common Conversion Errors

| Input | Target type | Typical result |
|---|---|---|
| `"101"` | `Long` | Success |
| `"abc"` | `Long` | Conversion failure |
| `"20"` | `int` | Success |
| `"twenty"` | `int` | Conversion failure |
| Missing required parameter | Required argument | Missing-parameter exception |

These failures typically produce a 400 response unless custom exception handling changes the response.

### Custom Conversion

Spring MVC supports conversion through its conversion service and custom converters or formatters.

For example, an application can define how a string should be converted into a custom domain type.

Use custom conversion when the value has a consistent representation across endpoints. For more complex business validation, validate the bound object or delegate validation to the appropriate application layer.

---

## 12. Request Validation with `@Valid`

Binding converts request data into Java values. Validation checks whether the resulting values satisfy declared constraints.

These are different operations.

### DTO with Validation Constraints

~~~java
public class ProductRequest {

    @NotBlank(message = "Name is required")
    private String name;

    @NotNull(message = "Price is required")
    @Positive(message = "Price must be positive")
    private BigDecimal price;

    public ProductRequest() {
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public BigDecimal getPrice() {
        return price;
    }

    public void setPrice(BigDecimal price) {
        this.price = price;
    }
}
~~~

### Validate the Request Body

~~~java
@PostMapping("/products")
public ProductResponse create(
        @Valid @RequestBody ProductRequest request) {

    return productService.create(request);
}
~~~

When validation constraints fail, Spring MVC typically raises `MethodArgumentNotValidException` for this request-body validation scenario. The response normally becomes 400 unless customized.

### Common Validation Annotations

| Annotation | Purpose |
|---|---|
| `@NotNull` | Value must not be null |
| `@NotBlank` | String must not be null and must contain a non-whitespace character |
| `@NotEmpty` | String, collection, map, or array must not be empty; null is invalid |
| `@Size` | Restricts supported value size |
| `@Min` | Numeric value must be at least the specified minimum |
| `@Max` | Numeric value must not exceed the specified maximum |
| `@Positive` | Numeric value must be greater than zero |
| `@Email` | Checks email-like format |

The validation provider and the appropriate validation dependency must be available for these constraints to execute.

For nested objects, use `@Valid` on the nested property as needed.

**Interview one-liner:** Binding converts incoming request data into Java values; validation checks those values against declared constraints.

See: [05. Validation & Jackson](05-Validation-and-Jackson.md)

---

## 13. How Request Arguments Are Resolved Internally

Spring MVC does not use one universal mechanism for every controller parameter.

The `HandlerMethodArgumentResolver` infrastructure resolves method arguments based on their types and annotations.

Examples:

| Argument | Typical resolver or mechanism |
|---|---|
| `@PathVariable` | `PathVariableMethodArgumentResolver` |
| `@RequestParam` | `RequestParamMethodArgumentResolver` |
| `@RequestHeader` | `RequestHeaderMethodArgumentResolver` |
| `@CookieValue` | `ServletCookieValueMethodArgumentResolver` |
| `@RequestBody` | `RequestResponseBodyMethodProcessor` |
| `@ModelAttribute` | `ModelAttributeMethodProcessor` |

Exact resolver selection depends on the declared parameter, configuration, and Spring version.

### Simplified Request Flow

1. The client sends an HTTP request.
2. `DispatcherServlet` receives the request.
3. `RequestMappingHandlerMapping` identifies the handler method.
4. A suitable `HandlerAdapter` invokes the method.
5. Argument resolvers extract and convert request values.
6. An appropriate message converter reads `@RequestBody` content.
7. Spring invokes the controller method.
8. The controller delegates to the service layer.
9. The return value is processed to produce the HTTP response.

**Interview trap:** `@RequestParam` and `@RequestBody` are not resolved by the same mechanism. Request parameters use argument resolution and conversion; request bodies use message converters.

---

## 14. Common HTTP Errors in Request Binding

| Status | Typical cause |
|---|---|
| 400 Bad Request | Missing required parameter, failed type conversion, malformed body, or validation failure |
| 404 Not Found | No matching handler or resource found |
| 405 Method Not Allowed | Request path exists, but the HTTP method is unsupported |
| 406 Not Acceptable | No compatible response representation |
| 413 Content Too Large | Request exceeds a configured content-size limit, where the server reports this status |
| 415 Unsupported Media Type | Request content type is unsupported |
| 500 Internal Server Error | Unexpected unhandled server-side failure |

The actual status and response body depend on exception handling and server configuration.

### Example Scenarios

- `/products/abc` where `id` must be a `Long`: usually 400.
- Missing required `category` request parameter: usually 400.
- Malformed JSON body: usually 400.
- JSON sent to an endpoint that only accepts another media type: potentially 415.
- Upload larger than configured multipart limits: commonly results in a request-size error, often 413.

**Interview trap:** A 400 response can result from binding or validation failures even when the URL mapping itself is valid.

---

## 15. Frequently Asked Interview Questions

### Q1. What is the difference between `@PathVariable` and `@RequestParam`?

`@PathVariable` extracts values from URL path segments, whereas `@RequestParam` extracts request parameters such as query parameters.

### Q2. Is `@RequestParam` required by default?

Yes. It is required by default unless `required = false` or a `defaultValue` is supplied.

### Q3. What happens if a required request parameter is missing?

Spring MVC typically raises a missing-request-parameter exception, resulting in a 400 response unless customized.

### Q4. What happens if a path variable cannot be converted to the target type?

Spring MVC typically raises a type-conversion exception, which normally results in a 400 response.

### Q5. What is the difference between `@RequestBody` and `@ModelAttribute`?

`@RequestBody` uses an `HttpMessageConverter` to deserialize the request body. `@ModelAttribute` binds request parameters and form values to an object.

### Q6. How does Spring convert JSON into a Java object?

An appropriate `HttpMessageConverter`, commonly backed by Jackson, deserializes JSON into the target Java type.

### Q7. Does `@RequestBody` automatically validate data?

No. Use `@Valid` or an appropriate validation mechanism to trigger constraint validation.

### Q8. What is the difference between `Content-Type` and `Accept`?

`Content-Type` describes the representation sent in the request body. `Accept` describes the response media types the client can accept.

### Q9. What is the difference between `consumes` and `produces`?

`consumes` restricts request media types, whereas `produces` specifies supported response media types.

### Q10. How can a request parameter be optional?

Use `@RequestParam(required = false)` or provide a `defaultValue`.

### Q11. Can a request parameter have multiple values?

Yes. Bind repeated parameter values to a collection or a `MultiValueMap`.

### Q12. What is `@RequestHeader` used for?

It extracts HTTP header values, such as custom correlation IDs or client metadata.

### Q13. What is `@CookieValue` used for?

It extracts a cookie value from the incoming HTTP request.

### Q14. What is `@RequestPart` used for?

It binds an individual multipart request part, including structured parts such as JSON metadata.

### Q15. Why should DTOs be used for request bodies?

DTOs define the API's input contract, reduce unwanted coupling to persistence entities, and help control which fields clients can submit.

### Q16. What is a `HandlerMethodArgumentResolver`?

It is Spring MVC infrastructure that resolves controller method arguments from the current request and application context.

### Q17. What is the difference between binding and validation?

Binding extracts and converts incoming data into Java values; validation checks those values against constraints.

### Q18. Can Spring bind a query parameter to an `int`?

Yes, when the supplied value can be converted to an integer. Invalid values normally result in a 400 response.

---

## 16. Common Mistakes and Interview Traps

- Using `@PathVariable` when the value is actually a query parameter.
- Forgetting that `@RequestParam` is required by default.
- Using a primitive type for an optional numeric parameter.
- Assuming `@RequestBody` parses JSON without an appropriate message converter.
- Forgetting to send the correct request `Content-Type`.
- Confusing `Content-Type` with `Accept`.
- Confusing `consumes` with `produces`.
- Assuming `@RequestBody` automatically triggers validation.
- Binding incoming request data directly to a persistence entity without considering mass assignment.
- Assuming `@RequestParam` is limited to query strings in every context.
- Treating a missing body, malformed body, and failed validation as the same error.
- Assuming `@RequestHeader` validates authentication credentials.
- Trusting uploaded filenames or client-supplied content types.
- Assuming every binding exception must return exactly the same error response regardless of custom exception handling.
- Putting business rules inside custom converters instead of the appropriate application layer.

---

## 17. Quick Revision

| Concept | Remember |
|---|---|
| `@PathVariable` | Extracts values from URL path segments |
| `@RequestParam` | Binds request parameters |
| `@RequestHeader` | Binds HTTP header values |
| `@CookieValue` | Binds cookie values |
| `@RequestBody` | Deserializes request-body content |
| `@ModelAttribute` | Binds form fields and request values to an object |
| `@RequestPart` | Binds a multipart request part |
| `required = false` | Allows a value to be absent |
| `defaultValue` | Supplies a default and makes the parameter optional |
| `HttpMessageConverter` | Reads request bodies and writes response bodies |
| `HandlerMethodArgumentResolver` | Resolves controller method arguments |
| `@Valid` | Triggers validation of a supported bound object |
| 400 | Common for binding, conversion, or validation failures |
| 415 | Unsupported request media type |
| `Content-Type` | Describes the request body's media type |
| `Accept` | Declares acceptable response media types |

### One-Line Interview Answers

- **`@PathVariable`:** Extracts a value from the URL path.
- **`@RequestParam`:** Binds a request parameter to a controller method argument.
- **`@RequestHeader`:** Extracts a value from an HTTP request header.
- **`@CookieValue`:** Extracts a cookie value from an HTTP request.
- **`@RequestBody`:** Uses an appropriate message converter to deserialize the request body into a Java object.
- **`@ModelAttribute`:** Binds request parameters and form values to an object.
- **`@RequestPart`:** Binds an individual part of a multipart request.
- **Binding:** Converts incoming request data into Java values.
- **Validation:** Checks bound values against declared constraints.
- **`HandlerMethodArgumentResolver`:** Resolves controller method arguments.
- **`HttpMessageConverter`:** Converts HTTP message bodies to and from Java representations.
- **DTO:** Defines a controlled data contract between the client and application.
- **400 Bad Request:** Commonly indicates malformed input, missing required parameters, conversion failure, or validation failure.
- **415 Unsupported Media Type:** Indicates that the request media type is not supported by the selected endpoint.

---

## 18. Related Topics

- [01. Spring MVC Architecture & Request Flow](01-Spring-MVC-Architecture.md)
- [02. Controllers & Request Mapping](02-Controllers-and-Mappings.md)
- [04. Responses & DTOs](04-Responses-and-DTOs.md)
- [05. Validation & Jackson](05-Validation-and-Jackson.md)
- [06. REST API Design & Clients](06-REST-API-Design-and-Clients.md)
- [01. Exception Handling](../06-Exception-Handling/01-Exception-Handling.md)