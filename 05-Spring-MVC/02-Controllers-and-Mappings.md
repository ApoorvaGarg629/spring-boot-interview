# 02. Controllers & Request Mapping

## 1. What Is a Controller in Spring MVC?

A **Controller** handles incoming HTTP requests, invokes the required business logic through the service layer, and returns a response.

- `@Controller`: Used primarily for MVC controllers that return views.
- `@RestController`: Used primarily for REST APIs that return response data directly.
- `@RequestMapping`: Maps HTTP requests to controller classes or methods.
- `@GetMapping`, `@PostMapping`, `@PutMapping`, `@PatchMapping`, `@DeleteMapping`: Specialized mappings for HTTP methods.

**Interview one-liner:** A controller receives HTTP requests, delegates processing to the service layer, and returns an appropriate view or HTTP response.

---

## 2. `@Controller` vs `@RestController`

### `@Controller`

Used primarily in traditional Spring MVC applications that render views such as Thymeleaf templates.

~~~java
@Controller
public class HomeController {

    @GetMapping("/home")
    public String home(Model model) {
        model.addAttribute("message", "Welcome");
        return "home";
    }
}
~~~

Here, `"home"` is interpreted as a view name and resolved by a `ViewResolver`.

To return a response body directly from a `@Controller` method, use `@ResponseBody`.

~~~java
@Controller
public class GreetingController {

    @GetMapping("/greeting")
    @ResponseBody
    public String greeting() {
        return "Hello Spring";
    }
}
~~~

### `@RestController`

Used primarily for REST APIs where methods return data such as JSON, XML, strings, or other supported response formats.

~~~java
@RestController
public class GreetingController {

    @GetMapping("/greeting")
    public String greeting() {
        return "Hello Spring";
    }
}
~~~

The returned string is written to the HTTP response body instead of being interpreted as a view name.

### Internal Working

`@RestController` is a composed annotation equivalent to:

~~~java
@Controller
@ResponseBody
public @interface RestController {
}
~~~

The `@ResponseBody` behavior applies to the controller's request-handling methods.

### Difference

| Feature | `@Controller` | `@RestController` |
|---|---|---|
| Primary purpose | MVC views | REST APIs |
| Default return behavior | Usually resolves a view | Writes data to response body |
| `@ResponseBody` required for direct response data | Usually yes | No |
| Common use case | Server-rendered pages | JSON APIs |

**Interview trap:** Returning `"home"` from a `@Controller` method normally means a view name; returning `"home"` from a `@RestController` method means response-body text.

---

## 3. What Is `@RequestMapping`?

`@RequestMapping` maps incoming HTTP requests to a controller class or handler method.

It can match requests using:

- URL path
- HTTP method
- Request parameters
- Request headers
- `Content-Type` through `consumes`
- Acceptable response media types through `produces`

### Class-Level Mapping

Defines a common URL prefix or shared request conditions.

~~~java
@RestController
@RequestMapping("/api/products")
public class ProductController {

    @GetMapping
    public String getAllProducts() {
        return "All products";
    }

    @GetMapping("/{id}")
    public String getProductById(@PathVariable Long id) {
        return "Product " + id;
    }
}
~~~

The endpoints are:

- `GET /api/products`
- `GET /api/products/10`

The class-level mapping is combined with the method-level mapping.

### Method-Level Mapping

Defines the endpoint and request conditions for an individual handler method.

~~~java
@GetMapping("/{id}")
public Product getProduct(@PathVariable Long id) {
    return productService.findById(id);
}
~~~

### Mapping with HTTP Method

~~~java
@RequestMapping(
    path = "/products",
    method = RequestMethod.GET
)
public List<Product> getProducts() {
    return productService.findAll();
}
~~~

The specialized equivalent is:

~~~java
@GetMapping("/products")
public List<Product> getProducts() {
    return productService.findAll();
}
~~~

**Interview one-liner:** `@RequestMapping` defines request-matching conditions at class or method level, while specialized annotations provide concise mappings for individual HTTP methods.

---

## 4. Specialized Request Mapping Annotations

| Annotation | HTTP method | Typical purpose |
|---|---|---|
| `@GetMapping` | GET | Retrieve data |
| `@PostMapping` | POST | Create a resource or trigger processing |
| `@PutMapping` | PUT | Replace a resource representation |
| `@PatchMapping` | PATCH | Partially update a resource |
| `@DeleteMapping` | DELETE | Delete a resource |

Example:

~~~java
@RestController
@RequestMapping("/api/products")
public class ProductController {

    @GetMapping
    public List<Product> getAll() {
        return productService.findAll();
    }

    @GetMapping("/{id}")
    public Product getById(@PathVariable Long id) {
        return productService.findById(id);
    }

    @PostMapping
    public Product create(@RequestBody ProductRequest request) {
        return productService.create(request);
    }

    @PutMapping("/{id}")
    public Product replace(
            @PathVariable Long id,
            @RequestBody ProductRequest request) {
        return productService.replace(id, request);
    }

    @PatchMapping("/{id}")
    public Product updatePartially(
            @PathVariable Long id,
            @RequestBody ProductRequest request) {
        return productService.updatePartially(id, request);
    }

    @DeleteMapping("/{id}")
    public void delete(@PathVariable Long id) {
        productService.delete(id);
    }
}
~~~

`Product`, `ProductRequest`, and `productService` represent application-specific classes and dependencies.

### Important HTTP Semantics

- GET should be safe and should not intentionally change application state.
- PUT is generally used for complete replacement and is idempotent.
- PATCH is used for partial updates; idempotency depends on the patch operation.
- POST is commonly used for creation or processing and is not inherently idempotent.
- DELETE is defined as idempotent in HTTP semantics, even if repeated requests produce different response codes.

**Interview trap:** The HTTP method should reflect the operation's intended semantics. The annotations themselves do not automatically implement CRUD logic or enforce correct update behavior.

---

## 5. Mapping Attributes of `@RequestMapping`

### `value` and `path`

Both specify URL paths and are aliases for each other.

~~~java
@GetMapping(value = "/products")
public List<Product> getProducts() {
    return productService.findAll();
}
~~~

Equivalent:

~~~java
@GetMapping(path = "/products")
public List<Product> getProducts() {
    return productService.findAll();
}
~~~

Multiple paths can be specified.

~~~java
@GetMapping({"/products", "/items"})
public List<Product> getProducts() {
    return productService.findAll();
}
~~~

Both paths map to the same handler method.

### `method`

Restricts the mapping to particular HTTP methods.

~~~java
@RequestMapping(
    path = "/products",
    method = RequestMethod.POST
)
public Product create(@RequestBody ProductRequest request) {
    return productService.create(request);
}
~~~

### `params`

Restricts a mapping based on query parameters.

~~~java
@GetMapping(value = "/products", params = "active=true")
public List<Product> getActiveProducts() {
    return productService.findActive();
}
~~~

Matches a request such as:

`GET /products?active=true`

Other examples:

~~~java
@GetMapping(value = "/products", params = "category")
public List<Product> getByCategory(
        @RequestParam String category) {
    return productService.findByCategory(category);
}

@GetMapping(value = "/products", params = "!archived")
public List<Product> getNonArchivedProducts() {
    return productService.findNonArchived();
}
~~~

- `params = "category"`: Parameter must be present.
- `params = "!archived"`: Parameter must be absent.
- `params = "active=true"`: Parameter must have the specified value.
- `params = "active!=true"`: Parameter must not have the specified value.

Use these conditions when different handler methods need to be selected based on query parameters.

### `headers`

Restricts a mapping based on request headers.

~~~java
@GetMapping(
    value = "/products",
    headers = "X-Client=mobile"
)
public List<Product> getMobileProducts() {
    return productService.findAll();
}
~~~

Matches a request containing:

`X-Client: mobile`

Header conditions can be useful for custom request-routing requirements. For content negotiation, `produces` is generally more expressive than manually matching the `Accept` header.

### `consumes`

Restricts the request's `Content-Type`, describing the media type of the incoming request body.

~~~java
@PostMapping(
    value = "/products",
    consumes = "application/json"
)
public Product create(@RequestBody ProductRequest request) {
    return productService.create(request);
}
~~~

This mapping expects JSON request content.

If the request's content type does not match, Spring MVC can return **415 Unsupported Media Type**, depending on the request and configured handlers.

### `produces`

Restricts the response media types the handler can produce.

~~~java
@GetMapping(
    value = "/products",
    produces = "application/json"
)
public List<Product> getProducts() {
    return productService.findAll();
}
~~~

Spring uses the `Accept` header and content-negotiation configuration to determine whether the requested response format is supported.

If no compatible representation is available, the request can result in **406 Not Acceptable**.

### Quick Comparison

| Attribute | Matches |
|---|---|
| `path` / `value` | URL path |
| `method` | HTTP method |
| `params` | Query parameters |
| `headers` | Request headers |
| `consumes` | Incoming request media type |
| `produces` | Acceptable response media type |

**Interview one-liner:** `consumes` validates the request content type, whereas `produces` describes the response media types supported by the handler.

---

## 6. Path Variables vs Request Parameters

Both are used to extract values from an HTTP request, but they serve different purposes.

### `@PathVariable`

Extracts a value from the URL path.

~~~java
@GetMapping("/products/{id}")
public Product getProduct(@PathVariable Long id) {
    return productService.findById(id);
}
~~~

Request:

`GET /products/101`

Here, `id = 101`.

### `@RequestParam`

Extracts query parameters or, when configured, form parameters.

~~~java
@GetMapping("/products")
public List<Product> getProducts(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "10") int size) {
    return productService.findPage(page, size);
}
~~~

Request:

`GET /products?page=2&size=20`

Here, `page = 2` and `size = 20`.

### Difference

| Feature | `@PathVariable` | `@RequestParam` |
|---|---|---|
| Source | URL path segment | Request parameter |
| Example | `/products/101` | `/products?id=101` |
| Common use | Identify a resource | Filtering, pagination, sorting |
| Required by default | Yes, unless configured otherwise | Yes, unless optional/default configured |

### Optional Request Parameters

~~~java
@GetMapping("/products")
public List<Product> search(
        @RequestParam(required = false) String category) {
    return productService.search(category);
}
~~~

Alternatively, provide a default value:

~~~java
@RequestParam(defaultValue = "all") String category
~~~

A `defaultValue` makes the parameter optional and applies the default when it is missing or empty.

**Interview trap:** `@RequestParam` does not mean query string only in every context; Spring MVC can also bind request parameters from form data.

---

## 7. `@RequestBody` and `@ModelAttribute`

### `@RequestBody`

Binds the HTTP request body to a Java object, typically by using an `HttpMessageConverter` such as Jackson for JSON.

~~~java
@PostMapping("/products")
public Product create(@RequestBody ProductRequest request) {
    return productService.create(request);
}
~~~

Example JSON request:

~~~json
{
  "name": "Keyboard",
  "price": 2500
}
~~~

Spring reads the body and converts it into a `ProductRequest` object.

### `@ModelAttribute`

Binds request parameters, form fields, and supported request values to an object. It can also expose the object to the MVC model.

~~~java
@PostMapping("/products/form")
public String createFromForm(
        @ModelAttribute ProductForm form) {
    productService.createFromForm(form);
    return "redirect:/products";
}
~~~

Typical use cases include HTML form submissions and MVC form handling.

### Difference

| Feature | `@RequestBody` | `@ModelAttribute` |
|---|---|---|
| Typical input | JSON or XML request body | Form fields and request parameters |
| Binding mechanism | `HttpMessageConverter` | Web data binding |
| Common use | REST API | MVC form submission |
| Typical output context | REST response | View/model workflow |

**Interview trap:** `@RequestBody` does not parse JSON by itself. A suitable message converter, commonly Jackson-based, performs the deserialization.

For complex objects, use DTOs rather than exposing persistence entities directly as API contracts.

---

## 8. `@RequestHeader` and `@CookieValue`

### `@RequestHeader`

Extracts a value from an HTTP request header.

~~~java
@GetMapping("/profile")
public String getProfile(
        @RequestHeader("X-Request-Id") String requestId) {
    return "Request ID: " + requestId;
}
~~~

Make a header optional when appropriate:

~~~java
@RequestHeader(
    value = "X-Request-Id",
    required = false
) String requestId
~~~

### `@CookieValue`

Extracts a cookie value from the request.

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

**Interview one-liner:** `@RequestHeader` binds HTTP headers, while `@CookieValue` binds cookies sent by the client.

---

## 9. Mapping Multiple Values and Regular Expressions

### Multiple Paths

~~~java
@GetMapping({"/products", "/items"})
public List<Product> getProducts() {
    return productService.findAll();
}
~~~

Both paths invoke the same method.

### Path Variables with Regular Expressions

~~~java
@GetMapping("/products/{id:\\d+}")
public Product getProduct(@PathVariable Long id) {
    return productService.findById(id);
}
~~~

The regular expression restricts the path variable to digits.

This can help distinguish overlapping routes, but prefer clear endpoint designs and type conversion where possible.

### Wildcard Patterns

Spring MVC supports path pattern matching, including patterns such as:

- `*`: Matches characters within a path segment.
- `**`: Matches multiple path segments, subject to the path-pattern syntax and placement rules.
- `{id}`: Captures a path segment.
- `{id:\\d+}`: Captures a path segment constrained by a regular expression.

Exact matching behavior depends on the configured path-pattern strategy and Spring version.

**Interview trap:** Do not assume all Ant-style pattern behavior is identical to Spring's modern `PathPattern` matching.

---

## 10. What Happens When Multiple Mappings Match?

Spring MVC uses `RequestMappingHandlerMapping` and related mapping infrastructure to identify suitable handlers.

When multiple mappings are eligible, Spring evaluates their matching conditions and selects the most specific match according to its mapping rules.

For example:

~~~java
@GetMapping("/products/{id}")
public Product getById(@PathVariable Long id) {
    return productService.findById(id);
}

@GetMapping("/products/search")
public List<Product> search() {
    return productService.search();
}
~~~

A request to `/products/search` should match the literal `search` route rather than treating `search` as the `id` path variable, because the literal route is more specific.

### Ambiguous Mapping

An ambiguous mapping occurs when Spring cannot uniquely register or select a handler for the same mapping conditions.

Example:

~~~java
@GetMapping("/products")
public List<Product> getProducts() {
    return productService.findAll();
}

// Invalid when registered alongside the same mapping:
@GetMapping("/products")
public List<Product> getAllProducts() {
    return productService.findAll();
}
~~~

Spring generally detects duplicate handler mappings during application startup and throws an ambiguous mapping exception.

### Common Causes

- Duplicate path and HTTP method mappings.
- Two controller methods with effectively identical mapping conditions.
- Overlapping mapping declarations that cannot be distinguished appropriately.

**Interview trap:** Java method names do not distinguish request mappings. The mapping conditions must distinguish the endpoints.

---

## 11. Required vs Optional Parameters and Type Conversion

Spring MVC converts supported request values into the target Java types.

~~~java
@GetMapping("/products/{id}")
public Product getProduct(@PathVariable Long id) {
    return productService.findById(id);
}
~~~

For `/products/101`, Spring converts `"101"` into `Long`.

For `/products/abc`, conversion fails and typically results in a **400 Bad Request** response.

### Default Values

~~~java
@GetMapping("/products")
public List<Product> getProducts(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "10") int size) {
    return productService.findPage(page, size);
}
~~~

### Optional Values

~~~java
@GetMapping("/products")
public List<Product> getProducts(
        @RequestParam(required = false) Integer categoryId) {
    return productService.findByCategoryId(categoryId);
}
~~~

Use a wrapper type such as `Integer` when the value may be absent. A primitive such as `int` cannot represent `null`.

**Interview trap:** `required = false` means the parameter can be absent; it does not mean every supplied value is valid.

---

## 12. `@RequestMapping` Inheritance and Composition

### Class-Level and Method-Level Mappings

~~~java
@RestController
@RequestMapping("/api/v1")
public class OrderController {

    @GetMapping("/orders")
    public List<Order> getOrders() {
        return orderService.findAll();
    }
}
~~~

The resulting route is:

`GET /api/v1/orders`

Class-level and method-level conditions combine to form the effective mapping.

### Composed Annotations

Annotations such as `@GetMapping` and `@PostMapping` are composed shortcuts built on `@RequestMapping`.

They improve readability and reduce repeated configuration.

Prefer a specialized annotation when the endpoint uses one standard HTTP method.

---

## 13. Controller Layer vs Service Layer

A controller should handle HTTP concerns, not business rules.

### Controller Responsibilities

- Map HTTP requests to handler methods.
- Bind path variables, query parameters, headers, and bodies.
- Apply request validation.
- Delegate work to services.
- Return suitable HTTP responses.

### Service Responsibilities

- Implement business rules.
- Coordinate domain operations.
- Define transactional boundaries where appropriate.
- Call repositories or other services.

### Repository Responsibilities

- Perform data-access operations.
- Execute persistence queries.
- Interact with the database through the selected persistence technology.

Example:

~~~java
@RestController
@RequestMapping("/api/products")
public class ProductController {

    private final ProductService productService;

    public ProductController(ProductService productService) {
        this.productService = productService;
    }

    @GetMapping("/{id}")
    public ProductResponse getById(@PathVariable Long id) {
        return productService.getById(id);
    }
}
~~~

The controller delegates to the service instead of implementing database or business logic directly.

**Interview one-liner:** The controller handles HTTP concerns, the service handles business logic, and the repository handles data access.

---

## 14. Controller Constructor Injection

Constructor injection is the recommended approach for required dependencies.

~~~java
@RestController
@RequestMapping("/api/products")
public class ProductController {

    private final ProductService productService;

    public ProductController(ProductService productService) {
        this.productService = productService;
    }
}
~~~

Benefits:

- Required dependencies are explicit.
- Fields can be `final`.
- The object cannot be constructed without the required dependency.
- Unit testing is easier because dependencies can be passed directly.
- The class is easier to reason about.

With a single constructor, Spring can use it for dependency injection without requiring `@Autowired`.

**Interview trap:** Constructor injection does not automatically make the injected dependency immutable internally. A `final` field prevents reassignment of the reference, not mutation of the referenced object.

---

## 15. Common HTTP Errors Related to Request Mapping

| Status | Meaning | Typical cause |
|---|---|---|
| 400 Bad Request | Request cannot be processed as supplied | Invalid type conversion or malformed input |
| 404 Not Found | No suitable resource or handler was found | Unknown path or application-specific missing resource |
| 405 Method Not Allowed | HTTP method is unsupported for the matched resource | Sending POST to a GET-only endpoint |
| 406 Not Acceptable | No compatible response representation | Incompatible `Accept` header |
| 415 Unsupported Media Type | Request media type is unsupported | Sending XML to a JSON-only endpoint |
| 500 Internal Server Error | Unexpected server-side failure | Unhandled application error |

Exact responses depend on the configured exception handlers, servlet configuration, and application behavior.

### Example: 405 Method Not Allowed

If the controller defines:

~~~java
@GetMapping("/products")
public List<Product> getProducts() {
    return productService.findAll();
}
~~~

A POST request to `/products` does not match this GET-only handler. Spring MVC may return 405 when it can determine that the path exists but the HTTP method is unsupported.

**Interview trap:** A 404 and a 405 are not the same. A 404 indicates that no matching handler/resource was found; a 405 indicates that the resource or path is recognized but the HTTP method is not supported.

---

## 16. `@ResponseStatus` and `ResponseEntity`

### `@ResponseStatus`

Specifies an HTTP status for a handler method or exception class.

~~~java
@PostMapping("/products")
@ResponseStatus(HttpStatus.CREATED)
public Product create(@RequestBody ProductRequest request) {
    return productService.create(request);
}
~~~

This declares a `201 Created` response for the successful handler execution.

### `ResponseEntity`

Provides control over the HTTP status, headers, and response body.

~~~java
@PostMapping("/products")
public ResponseEntity<ProductResponse> create(
        @RequestBody ProductRequest request) {

    ProductResponse created = productService.create(request);

    return ResponseEntity
            .status(HttpStatus.CREATED)
            .body(created);
}
~~~

Use `ResponseEntity` when the response status or headers need to vary depending on the outcome.

Example:

~~~java
@GetMapping("/products/{id}")
public ResponseEntity<ProductResponse> getById(
        @PathVariable Long id) {

    return productService.findResponseById(id)
            .map(ResponseEntity::ok)
            .orElseGet(() -> ResponseEntity.notFound().build());
}
~~~

Here, `findResponseById` is assumed to return an `Optional<ProductResponse>`.

### Difference

- `@ResponseStatus`: Declares a status for a handler or exception.
- `ResponseEntity`: Programmatically controls status, headers, and body for each response.

---

## 17. Testing Controller Mappings

Spring Boot supports controller tests using MockMvc.

Example:

~~~java
@WebMvcTest(ProductController.class)
class ProductControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockitoBean
    private ProductService productService;

    @Test
    void shouldReturnProduct() throws Exception {
        ProductResponse response =
                new ProductResponse(1L, "Keyboard");

        given(productService.getById(1L)).willReturn(response);

        mockMvc.perform(get("/api/products/1"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.id").value(1))
                .andExpect(jsonPath("$.name").value("Keyboard"));
    }
}
~~~

This example assumes:

- `ProductResponse` has a constructor accepting `id` and `name`.
- `ProductController` exposes `GET /api/products/{id}`.
- `productService.getById` returns the expected DTO.
- The project uses Spring Boot 3.4+ or Spring Boot 4, where `@MockitoBean` is available.
- Static imports for MockMvc, Mockito BDD methods, `status()`, and `jsonPath()` are present.

For older Spring Boot versions, `@MockBean` is commonly used instead of `@MockitoBean`.

### What Does `@WebMvcTest` Do?

- Loads a focused MVC test context.
- Tests request mapping, argument binding, serialization, validation, and MVC exception handling.
- Usually excludes ordinary service and repository beans from the slice.
- Allows service dependencies to be mocked.

**Interview one-liner:** `@WebMvcTest` tests the web layer in isolation, while `@SpringBootTest` is used for broader application-context integration testing.

---

## 18. Frequently Asked Interview Questions

### Q1. What is the difference between `@Controller` and `@RestController`?

`@Controller` is commonly used for view-based MVC, while `@RestController` combines `@Controller` and `@ResponseBody` so handler return values are written to the response body.

### Q2. What is the difference between `@RequestMapping` and `@GetMapping`?

`@RequestMapping` supports general mapping conditions and all HTTP methods. `@GetMapping` is a composed annotation specifically for GET requests.

### Q3. Can `@RequestMapping` be used at both class and method level?

Yes. The class-level mapping supplies common conditions, and the method-level mapping narrows or extends the effective mapping.

### Q4. What is the difference between `path` and `value`?

They are aliases for the URL path attribute in `@RequestMapping` and its composed annotations.

### Q5. What is the difference between `consumes` and `produces`?

`consumes` matches the incoming request content type. `produces` specifies the response media types that the handler can produce.

### Q6. What is the difference between `@PathVariable` and `@RequestParam`?

`@PathVariable` extracts values from URL path segments, while `@RequestParam` extracts request parameters such as query parameters.

### Q7. What happens if a required request parameter is missing?

Spring MVC typically raises a missing-request-parameter exception, which normally results in a 400 response unless customized.

### Q8. What happens if a path variable cannot be converted to the target type?

Spring MVC typically raises a type-conversion exception, often resulting in a 400 response.

### Q9. Can two controller methods use the same URL?

Yes, if their mapping conditions distinguish them, such as different HTTP methods or suitable parameter/header conditions. Indistinguishable mappings can cause an ambiguity error.

### Q10. How does Spring MVC find a controller method?

`RequestMappingHandlerMapping` identifies a matching handler method. A suitable `HandlerAdapter` then invokes it as part of the DispatcherServlet request-processing flow.

### Q11. Can a controller return a Java object directly?

Yes. With `@RestController` or `@ResponseBody`, Spring uses an appropriate `HttpMessageConverter` to serialize the object, commonly into JSON.

### Q12. What is the purpose of `@ResponseStatus`?

It declares an HTTP status for a handler method or exception class.

### Q13. When should `ResponseEntity` be used?

Use it when response status codes, headers, or bodies must be controlled programmatically.

### Q14. Should controllers contain business logic?

Controllers should primarily handle HTTP-specific concerns and delegate business operations to services.

### Q15. Does `@GetMapping` automatically make an operation read-only?

No. It maps GET requests, but the application must still implement the intended safe HTTP semantics.

---

## 19. Common Mistakes and Interview Traps

- Returning a view name from `@RestController` expecting a template to render.
- Returning a string from `@Controller` without `@ResponseBody` and expecting it to be plain response text.
- Confusing query parameters with path variables.
- Reversing `consumes` and `produces`.
- Creating duplicate mappings with identical conditions.
- Using primitive types for optional request values that may be absent.
- Assuming `@RequestBody` and `@ModelAttribute` perform the same binding.
- Putting database queries and business rules directly in controllers.
- Assuming a missing handler always produces a 404 in every configuration.
- Assuming a 405 and 404 are interchangeable.
- Assuming mapping annotations automatically validate business rules.
- Assuming every controller must return `ResponseEntity`.
- Assuming a `@RestController` automatically handles authentication and authorization.
- Assuming `@RequestMapping` by itself implements the endpoint's business behavior.

---

## 20. Quick Revision

| Concept | Remember |
|---|---|
| `@Controller` | Commonly returns views |
| `@RestController` | Returns response-body data |
| `@RequestMapping` | General request mapping |
| `@GetMapping` | GET endpoint |
| `@PostMapping` | POST endpoint |
| `@PutMapping` | PUT endpoint |
| `@PatchMapping` | PATCH endpoint |
| `@DeleteMapping` | DELETE endpoint |
| `@PathVariable` | Extracts a path segment |
| `@RequestParam` | Extracts request parameters |
| `@RequestBody` | Deserializes the request body |
| `@ModelAttribute` | Binds request values to an object |
| `@RequestHeader` | Extracts a request header |
| `@CookieValue` | Extracts a cookie |
| `consumes` | Matches request content type |
| `produces` | Declares supported response media types |
| `@ResponseStatus` | Declares an HTTP status |
| `ResponseEntity` | Controls status, headers, and body |
| `@WebMvcTest` | Focused MVC testing |

### One-Line Interview Answers

- **Controller:** Maps incoming HTTP requests to application operations and returns a view or response.
- **`@RestController`:** Combines `@Controller` and `@ResponseBody`.
- **`@RequestMapping`:** Defines URL and request-matching conditions for handler methods.
- **`@GetMapping`:** Maps GET requests to a handler method.
- **`@PathVariable`:** Extracts values from the URL path.
- **`@RequestParam`:** Binds request parameters to method arguments.
- **`@RequestBody`:** Converts the request body into a Java object using an appropriate message converter.
- **`consumes`:** Restricts the request media type.
- **`produces`:** Declares the response media types supported by a handler.
- **`ResponseEntity`:** Allows explicit control over HTTP status, headers, and body.
- **Ambiguous mapping:** Occurs when Spring cannot uniquely distinguish handler mappings.
- **Controller responsibility:** Handle HTTP concerns and delegate business logic to the service layer.

---

## 21. Related Topics

- [01. Spring MVC Architecture & Request Flow](01-Spring-MVC-Architecture.md)
- [03. Request Parameters & Request Body](03-Request-Parameters-and-Body.md)
- [04. Responses & DTOs](04-Responses-and-DTOs.md)
- [05. Validation & Jackson](05-Validation-and-Jackson.md)
- [06. REST API Design & Clients](06-REST-API-Design-and-Clients.md)
- [01. Exception Handling](../06-Exception-Handling/01-Exception-Handling.md)