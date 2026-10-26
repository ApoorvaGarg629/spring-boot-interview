# Spring MVC Architecture & Request Flow

> **Interview Focus:** Spring MVC architecture, Front Controller pattern, DispatcherServlet, HandlerMapping, HandlerAdapter, controllers, argument resolvers, HttpMessageConverters, ViewResolver, request processing lifecycle, Servlet Filter vs HandlerInterceptor, exception handling, and common interview scenarios.

---

## 1. What is Spring MVC?

**Spring MVC** is a web framework built on the Servlet API that follows the Model-View-Controller design pattern to build web applications and REST APIs.

It separates responsibilities into three main components:

- **Model:** Represents the data required by the application or view.
- **View:** Presents the response to the user, such as an HTML page.
- **Controller:** Handles incoming requests, invokes business logic, and determines the response.

Spring MVC is commonly used to build REST APIs with Spring Boot.

### Example

~~~java
@RestController
class UserController {

    @GetMapping("/users")
    public List<String> getUsers() {
        return List.of("Alice", "Bob");
    }
}
~~~

When a client sends a GET request to `/users`, Spring MVC routes the request to the appropriate controller method and returns the response.

### One-liner

> **Spring MVC is a Servlet-based web framework that processes HTTP requests through a centralized DispatcherServlet and routes them to appropriate handlers.**

---

## 2. What is the MVC Design Pattern?

MVC stands for Model-View-Controller.

### Model

Represents application data and, depending on the design, the state needed by the view.

Examples:

- DTOs
- Domain objects
- Data passed to a template

### View

Responsible for presenting data, usually as HTML.

Examples:

- Thymeleaf templates
- JSP pages

### Controller

Handles the request, calls the required services, and returns a view name or response body.

### Traditional MVC Flow

~~~text
Client
  |
  v
Controller
  |
  v
Service / Business Logic
  |
  v
Model Data
  |
  v
View
  |
  v
HTTP Response
~~~

For REST APIs, the controller usually returns data such as JSON rather than rendering a server-side HTML view.

---

## 3. Spring MVC Architecture

The central component of Spring MVC is the **DispatcherServlet**.

It follows the Front Controller design pattern.

~~~text
                    Client
                      |
                      v
               HTTP Request
                      |
                      v
              Servlet Container
              (e.g. Tomcat)
                      |
                      v
              DispatcherServlet
                      |
                      v
                HandlerMapping
                      |
                      v
                HandlerAdapter
                      |
                      v
                  Controller
                      |
                      v
                 Service Layer
                      |
                      v
               Repository Layer
                      |
                      v
                   Database
                      |
                      v
               Controller Result
                      |
             +--------+--------+
             |                 |
             v                 v
       REST Response       View Name
             |                 |
             v                 v
    HttpMessageConverter   ViewResolver
             |                 |
             v                 v
       JSON / XML          HTML / View
             |                 |
             +--------+--------+
                      |
                      v
                HTTP Response
                      |
                      v
                    Client
~~~

The exact processing path depends on whether the request returns a response body or a rendered view.

---

## 4. What is the Front Controller Pattern?

The **Front Controller pattern** provides one central entry point for incoming requests.

In Spring MVC, `DispatcherServlet` acts as the Front Controller.

Instead of each controller handling requests directly from the Servlet container, requests pass through the DispatcherServlet first.

### Responsibilities

- Receives incoming requests
- Coordinates request processing
- Finds the appropriate handler
- Invokes the handler through a HandlerAdapter
- Processes the result
- Coordinates view resolution or response-body conversion
- Delegates exception handling to configured resolvers

### Benefits

- Centralized request processing
- Consistent request handling
- Integration with interceptors
- Common exception handling
- Easier integration with Spring MVC infrastructure

### Interview one-liner

> **DispatcherServlet is Spring MVC's Front Controller that centrally coordinates request mapping, handler execution, response processing, and exception resolution.**

---

## 5. What is DispatcherServlet?

`DispatcherServlet` is a Servlet provided by Spring MVC that receives incoming HTTP requests and coordinates their processing.

In Spring Boot, it is normally configured automatically when the web application starts.

Conceptually:

~~~text
Incoming HTTP Request
         |
         v
  DispatcherServlet
         |
         +---- HandlerMapping
         |
         +---- HandlerAdapter
         |
         +---- Controller
         |
         +---- ExceptionResolver
         |
         +---- ViewResolver
         |
         +---- HttpMessageConverter
         |
         v
    HTTP Response
~~~

### Important points

- It is a Servlet, not a controller.
- It delegates actual business processing to controller methods and the service layer.
- It does not directly execute every controller method itself; it uses handler infrastructure.
- It coordinates multiple Spring MVC components.
- It can handle both REST API requests and traditional view-based requests.

---

## 6. How is DispatcherServlet Registered in Spring Boot?

In a Spring Boot Servlet web application, the framework normally configures and registers the DispatcherServlet automatically.

The application typically starts with:

~~~java
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
~~~

Spring Boot's web auto-configuration creates the necessary MVC infrastructure.

You generally do not need to manually register the DispatcherServlet for a standard Spring Boot MVC application.

### Interview question

**Q: Do we need to configure DispatcherServlet manually in Spring Boot?**

> No. Spring Boot normally configures and registers it automatically for a Servlet-based web application.

---

## 7. What Happens When a Request Arrives?

Consider this endpoint:

~~~java
@RestController
@RequestMapping("/users")
class UserController {

    @GetMapping("/{id}")
    public String getUser(@PathVariable Long id) {
        return "User " + id;
    }
}
~~~

A client sends:

~~~http
GET /users/10
~~~

Spring MVC processes the request through several components.

### Step 1: Servlet Container Receives the Request

The embedded server, such as Tomcat, receives the HTTP request.

The Servlet container manages the Servlet execution environment.

### Step 2: DispatcherServlet Receives the Request

The request reaches the DispatcherServlet through its registered URL mapping.

### Step 3: HandlerMapping Finds the Handler

Spring identifies the controller method that matches:

- HTTP method: `GET`
- URL pattern: `/users/{id}`

### Step 4: HandlerAdapter Invokes the Handler

The appropriate HandlerAdapter prepares the arguments and invokes the controller method.

### Step 5: Controller Executes

The controller receives `id = 10` and returns `"User 10"`.

### Step 6: Response Is Processed

Because the controller uses `@RestController`, the return value is written to the HTTP response body.

A suitable HttpMessageConverter handles the conversion.

### Step 7: Response Is Returned

The client receives the HTTP response.

~~~http
HTTP/1.1 200 OK
Content-Type: text/plain

User 10
~~~

---

## 8. Complete Spring MVC Request Lifecycle

The following is the conceptual lifecycle for a typical Spring MVC request.

~~~text
1. Client sends HTTP request
              |
              v
2. Servlet container receives request
              |
              v
3. Servlet Filters execute
              |
              v
4. DispatcherServlet receives request
              |
              v
5. HandlerMapping finds handler
              |
              v
6. HandlerInterceptor.preHandle()
              |
              v
7. HandlerAdapter invokes controller
              |
              v
8. Resolve method arguments
              |
              v
9. Execute controller method
              |
              v
10. Process return value
              |
              v
11. Render view OR write response body
              |
              v
12. HandlerInterceptor.postHandle()
    (for a normal MVC handler flow)
              |
              v
13. Render view if applicable
              |
              v
14. HandlerInterceptor.afterCompletion()
              |
              v
15. Response returns through Filters
              |
              v
16. Client receives HTTP response
~~~

> This is a simplified flow for a typical synchronous request. Exception paths, asynchronous processing, direct response-body handling, and Servlet container behavior can change the exact callback sequence.

---

## 9. What is HandlerMapping?

`HandlerMapping` identifies which handler should process an incoming request.

For annotation-based controllers, it matches request information against mappings such as:

- `@RequestMapping`
- `@GetMapping`
- `@PostMapping`
- `@PutMapping`
- `@DeleteMapping`
- `@PatchMapping`

Example:

~~~java
@RestController
@RequestMapping("/products")
class ProductController {

    @GetMapping("/{id}")
    public String getProduct(@PathVariable Long id) {
        return "Product " + id;
    }
}
~~~

For:

~~~http
GET /products/5
~~~

HandlerMapping identifies the matching handler method.

### Important point

HandlerMapping finds the handler; it does not normally execute the controller method itself.

### One-liner

> **HandlerMapping maps an incoming request to the appropriate handler, usually a controller method.**

---

## 10. What is HandlerAdapter?

`HandlerAdapter` invokes a handler selected by HandlerMapping.

It provides an abstraction so DispatcherServlet does not need to know the details of how every handler type is invoked.

For annotation-based controllers, Spring MVC commonly uses `RequestMappingHandlerAdapter`.

It supports functionality such as:

- Resolving controller method arguments
- Invoking controller methods
- Processing return values
- Applying method argument and return-value infrastructure

### HandlerMapping vs HandlerAdapter

| HandlerMapping | HandlerAdapter |
|---|---|
| Finds the appropriate handler | Invokes the selected handler |
| Determines which handler matches | Knows how to execute the handler |
| Request-to-handler mapping | Handler execution |

### One-liner

> **HandlerMapping finds the handler, while HandlerAdapter invokes it.**

---

## 11. What is a Controller?

A controller handles incoming web requests and coordinates the response.

Common annotations:

- `@Controller`
- `@RestController`

Example:

~~~java
@Controller
class PageController {

    @GetMapping("/home")
    public String home() {
        return "home";
    }
}
~~~

This usually returns the view name `home`.

Another example:

~~~java
@RestController
class HealthController {

    @GetMapping("/health")
    public String health() {
        return "UP";
    }
}
~~~

This returns `UP` as the response body.

### Best practice

Keep controllers thin.

They should normally:

- Accept and validate request input
- Call service methods
- Map results to response DTOs
- Return an appropriate response

Business rules generally belong in the service or domain layer rather than in controllers.

---

## 12. `@Controller` vs `@RestController`

### `@Controller`

Used mainly for MVC controllers that return view names.

~~~java
@Controller
class UserPageController {

    @GetMapping("/users")
    public String users(Model model) {
        model.addAttribute("name", "Alice");
        return "users";
    }
}
~~~

Here, `"users"` is normally interpreted as a view name.

### `@RestController`

Used mainly for REST APIs.

~~~java
@RestController
class UserApiController {

    @GetMapping("/users")
    public List<String> users() {
        return List.of("Alice", "Bob");
    }
}
~~~

The returned list is serialized into the response body, typically as JSON.

### Internal relationship

`@RestController` is a composed annotation equivalent in purpose to:

~~~java
@Controller
@ResponseBody
~~~

`@ResponseBody` tells Spring MVC to write the return value to the HTTP response body rather than treating it as a view name.

### Interview one-liner

> **`@Controller` commonly returns views, while `@RestController` combines `@Controller` and `@ResponseBody` so return values are written to the response body.**

---

## 13. What Is ModelAndView?

`ModelAndView` represents a model and a view together.

Example:

~~~java
@Controller
class ProductPageController {

    @GetMapping("/product")
    public ModelAndView product() {

        ModelAndView mav = new ModelAndView("product");

        mav.addObject("name", "Laptop");

        return mav;
    }
}
~~~

Here:

- `"product"` is the view name.
- `"name"` is a model attribute.
- The ViewResolver locates the corresponding view.

`ModelAndView` is useful in traditional server-rendered MVC applications.

REST APIs commonly return DTOs or `ResponseEntity` instead.

---

## 14. What Is a Model?

In traditional Spring MVC, `Model` carries attributes from the controller to the view.

Example:

~~~java
@Controller
class UserPageController {

    @GetMapping("/profile")
    public String profile(Model model) {

        model.addAttribute("username", "Alice");

        return "profile";
    }
}
~~~

The view can access the `username` attribute.

A Model is not the same as a database entity or a Model class in every architectural design. It is specifically the container for view-related attributes in this MVC context.

---

## 15. What Is ViewResolver?

`ViewResolver` resolves a logical view name to a concrete view implementation.

Example:

~~~java
@Controller
class HomeController {

    @GetMapping("/home")
    public String home() {
        return "home";
    }
}
~~~

The controller returns:

~~~text
home
~~~

A configured ViewResolver determines which view corresponds to that name.

For example, a template engine may resolve it to:

~~~text
templates/home.html
~~~

The exact location depends on the view technology and configuration.

### ViewResolver is mainly relevant to

- Thymeleaf
- JSP
- Other server-side view technologies

A REST controller returning JSON generally does not require ViewResolver to process the response body.

### One-liner

> **ViewResolver converts a logical view name returned by a controller into an actual view that can render the response.**

---

## 16. What Is HttpMessageConverter?

`HttpMessageConverter` reads and writes HTTP request and response bodies.

It converts between Java objects and HTTP representations such as JSON or text.

### Example: Java Object to JSON

~~~java
@RestController
class UserController {

    @GetMapping("/user")
    public User getUser() {
        return new User(1L, "Alice");
    }
}
~~~

Suppose:

~~~java
public record User(Long id, String name) {
}
~~~

A suitable converter, commonly backed by Jackson for JSON in a typical Spring Boot setup, serializes the object into:

~~~json
{
  "id": 1,
  "name": "Alice"
}
~~~

### Example: JSON to Java Object

~~~java
@PostMapping("/users")
public User createUser(@RequestBody User user) {
    return user;
}
~~~

Spring reads the JSON request body and converts it into a `User` object.

### Common converters

- `MappingJackson2HttpMessageConverter` for JSON in typical Jackson 2 setups
- `StringHttpMessageConverter` for text
- Byte-array converters for binary content
- Other converters depending on the application's dependencies and configuration

The exact available converters depend on the Spring version and application configuration.

### One-liner

> **HttpMessageConverter converts HTTP request bodies into Java objects and Java return values into HTTP response bodies.**

---

## 17. ViewResolver vs HttpMessageConverter

| ViewResolver | HttpMessageConverter |
|---|---|
| Resolves logical view names | Converts request/response bodies |
| Used for server-rendered views | Commonly used for REST APIs |
| Works with a View implementation | Works with Java objects and HTTP representations |
| Example: Thymeleaf template | Example: Java object to JSON |

### Interview question

**Q: Does a REST controller use ViewResolver to return JSON?**

> Generally, no. `@RestController` writes the return value to the response body, and a suitable HttpMessageConverter handles serialization.

---

## 18. Request Mapping Annotations

Spring MVC provides annotations for mapping requests.

| Annotation | Typical HTTP method |
|---|---|
| `@GetMapping` | GET |
| `@PostMapping` | POST |
| `@PutMapping` | PUT |
| `@PatchMapping` | PATCH |
| `@DeleteMapping` | DELETE |
| `@RequestMapping` | Multiple methods or general mappings |

Example:

~~~java
@RestController
@RequestMapping("/orders")
class OrderController {

    @GetMapping
    public List<String> getOrders() {
        return List.of("Order-1", "Order-2");
    }

    @PostMapping
    public String createOrder() {
        return "Created";
    }

    @GetMapping("/{id}")
    public String getOrder(@PathVariable Long id) {
        return "Order " + id;
    }

    @PutMapping("/{id}")
    public String updateOrder(@PathVariable Long id) {
        return "Updated " + id;
    }

    @DeleteMapping("/{id}")
    public String deleteOrder(@PathVariable Long id) {
        return "Deleted " + id;
    }
}
~~~

A request mapping can match more than the path. It may also consider HTTP method, parameters, headers, and content types.

Detailed request-mapping syntax is covered in:

[02. Controllers & Request Mapping](02-Controllers-and-Mappings.md)

---

## 19. How Are Controller Method Arguments Resolved?

Spring MVC supports many method parameter types.

Examples:

- `@PathVariable`
- `@RequestParam`
- `@RequestHeader`
- `@RequestBody`
- `HttpServletRequest`
- `HttpServletResponse`
- `Model`
- `Principal`
- `Authentication`
- `Errors` or `BindingResult` in supported validation flows

Spring MVC uses argument resolver infrastructure to prepare method arguments before invoking the controller.

Example:

~~~java
@GetMapping("/users/{id}")
public User getUser(
        @PathVariable Long id,
        @RequestParam(defaultValue = "false") boolean includeDetails,
        @RequestHeader("X-Request-Id") String requestId) {

    return new User(id, "Alice");
}
~~~

The argument values are resolved from the request and supplied to the controller method.

Detailed parameter handling is covered in:

[03. Path Variables, Request Params, Headers & Request Body](03-Request-Parameters-and-Body.md)

---

## 20. What Is HandlerMethodArgumentResolver?

`HandlerMethodArgumentResolver` is an extension point used by Spring MVC to resolve a controller method parameter.

For example:

~~~java
@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) {
    return new User(id, "Alice");
}
~~~

Spring must determine how to obtain the `id` argument from the incoming request.

The corresponding argument-resolution infrastructure performs this work.

Custom argument resolvers can be created when an application needs to populate custom controller method parameters.

### One-liner

> **HandlerMethodArgumentResolver determines how a controller method parameter is obtained from the request or Spring MVC context.**

---

## 21. What Is HandlerMethodReturnValueHandler?

`HandlerMethodReturnValueHandler` processes the value returned by a controller method.

Different return types need different processing.

Examples:

- A String representing a view name
- `ModelAndView`
- A DTO returned by a REST controller
- `ResponseEntity`
- An asynchronous return type

For a REST response, the return-value handling infrastructure works with HttpMessageConverter to write the response body.

### One-liner

> **HandlerMethodReturnValueHandler determines how a controller method's return value should be processed.**

---

## 22. What Is a Servlet Filter?

A Servlet Filter operates at the Servlet level and can intercept requests and responses before or after they reach the DispatcherServlet.

Common uses:

- Authentication and security
- Request logging
- Correlation IDs
- CORS handling
- Request/response wrapping
- Compression

A filter can decide whether to continue the filter chain.

Conceptually:

~~~text
Client
  |
  v
Filter 1
  |
  v
Filter 2
  |
  v
DispatcherServlet
  |
  v
Controller
~~~

A request may pass through several filters.

Filters are part of the Servlet infrastructure rather than being specific to Spring MVC controller mappings.

---

## 23. What Is HandlerInterceptor?

`HandlerInterceptor` is a Spring MVC extension point that intercepts requests around handler execution.

It provides three principal callbacks:

### `preHandle()`

Called before the handler executes.

It can allow processing to continue or stop it.

### `postHandle()`

Called after the handler returns successfully, before view rendering in the normal synchronous MVC flow.

### `afterCompletion()`

Called after request processing and view rendering have completed, including typical exception paths handled by the MVC framework.

Example:

~~~java
@Component
class RequestInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(
            HttpServletRequest request,
            HttpServletResponse response,
            Object handler) {

        System.out.println("Before controller");

        return true;
    }

    @Override
    public void postHandle(
            HttpServletRequest request,
            HttpServletResponse response,
            Object handler,
            ModelAndView modelAndView) {

        System.out.println("After controller");
    }

    @Override
    public void afterCompletion(
            HttpServletRequest request,
            HttpServletResponse response,
            Object handler,
            Exception ex) {

        System.out.println("Request completed");
    }
}
~~~

Register an interceptor through MVC configuration when it needs to be applied to selected paths.

---

## 24. Filter vs HandlerInterceptor

| Filter | HandlerInterceptor |
|---|---|
| Servlet-level component | Spring MVC component |
| Can run before DispatcherServlet | Runs within Spring MVC handler processing |
| Can wrap or replace request/response objects | Works with handler-oriented processing |
| Can apply even when no controller handler is selected, depending on mapping | Associated with MVC handler execution |
| Commonly used for security and low-level request processing | Commonly used for MVC-specific cross-cutting behavior |
| Uses `FilterChain` | Uses `preHandle`, `postHandle`, `afterCompletion` |

### Interview one-liner

> **A Filter intercepts requests at the Servlet level, whereas a HandlerInterceptor operates inside Spring MVC around the execution of a selected handler.**

### Important security point

Spring Security commonly integrates with the Servlet Filter chain. A HandlerInterceptor is not a replacement for Spring Security.

---

## 25. Where Does AOP Fit into Spring MVC?

AOP provides cross-cutting behavior around method execution.

Examples:

- Logging
- Transaction management
- Method security
- Performance measurement
- Auditing

Conceptually:

~~~text
Controller
    |
    v
Service Proxy
    |
    +---- Transaction logic
    |
    +---- Logging logic
    |
    v
Service Method
~~~

AOP is not a replacement for DispatcherServlet.

The two operate at different levels:

- DispatcherServlet coordinates web request processing.
- AOP can intercept method invocations on Spring-managed beans.

---

## 26. Exception Handling in Spring MVC

Exceptions can occur during:

- Request mapping
- Argument conversion
- Validation
- Controller execution
- Service execution
- Response serialization

Spring MVC provides exception-resolution infrastructure to process exceptions.

Important components and annotations include:

- `HandlerExceptionResolver`
- `@ExceptionHandler`
- `@ControllerAdvice`
- `@RestControllerAdvice`

Example:

~~~java
@RestControllerAdvice
class GlobalExceptionHandler {

    @ExceptionHandler(IllegalArgumentException.class)
    public ResponseEntity<String> handleIllegalArgument(
            IllegalArgumentException ex) {

        return ResponseEntity
                .badRequest()
                .body(ex.getMessage());
    }
}
~~~

This handles matching exceptions from controller request processing and returns a suitable HTTP response.

The exact response behavior depends on the exception, resolver configuration, and application setup.

Detailed exception handling is covered in:

[01. Exception Handling in Spring Boot](../06-Exception-Handling/01-Exception-Handling.md)

---

## 27. What Happens When No Handler Matches?

Suppose the client requests:

~~~http
GET /unknown-path
~~~

and no configured handler matches the request.

Spring MVC may report a no-handler condition, which is commonly returned as HTTP 404.

The exact behavior can depend on Spring Boot and Spring MVC configuration, including whether the application has explicitly mapped a resource handler or configured exception handling.

### Interview answer

> **If no suitable handler or resource mapping matches a request, the application typically returns HTTP 404 Not Found.**

---

## 28. What Happens When the HTTP Method Is Incorrect?

Suppose the controller has:

~~~java
@GetMapping("/users")
public List<String> getUsers() {
    return List.of("Alice");
}
~~~

But the client sends:

~~~http
POST /users
~~~

The path exists, but the request method does not match the mapping.

Spring MVC typically responds with:

~~~http
405 Method Not Allowed
~~~

### Difference

- **404 Not Found:** No matching resource or handler is found.
- **405 Method Not Allowed:** The resource path is known, but the HTTP method is not supported for it.

Exact behavior can vary with resource handling and configuration.

---

## 29. What Happens When Request JSON Cannot Be Converted?

Consider:

~~~java
@PostMapping("/users")
public User createUser(@RequestBody User user) {
    return user;
}
~~~

If the client sends invalid JSON or a value that cannot be converted to the required Java type, request-body conversion may fail before the controller method executes.

Spring MVC can raise an exception such as:

`HttpMessageNotReadableException`

The response is commonly HTTP 400 Bad Request when handled by the standard exception-resolution infrastructure.

### Interview answer

> **Invalid JSON or an unreadable request body can cause message conversion to fail before the controller method is invoked.**

---

## 30. What Happens When No Suitable Message Converter Exists?

Suppose the controller returns a type that cannot be written for the requested response media type.

Spring MVC may fail to find a suitable HttpMessageConverter.

This can result in an `HttpMediaTypeNotAcceptableException` or `HttpMessageNotWritableException`, depending on the failure.

Typical related HTTP responses include:

- 406 Not Acceptable when an acceptable response representation cannot be produced
- 500 Internal Server Error for some response-writing failures

The exact exception and status depend on where the failure occurs and the available converters.

---

## 31. Content Negotiation

Content negotiation determines which representation should be used for a request or response.

For example, a client may send:

~~~http
Accept: application/json
~~~

or:

~~~http
Accept: application/xml
~~~

Spring MVC uses request metadata, configured content-negotiation rules, and available converters to determine an appropriate response representation.

If no acceptable representation can be produced, the request may result in HTTP 406 Not Acceptable.

Content negotiation is especially relevant to REST APIs supporting multiple media types.

---

## 32. Request Body vs Response Body

### `@RequestBody`

Reads the HTTP request body and converts it into a Java object.

~~~java
@PostMapping("/users")
public User createUser(@RequestBody User user) {
    return user;
}
~~~

### `@ResponseBody`

Writes the method return value to the HTTP response body.

~~~java
@GetMapping("/status")
@ResponseBody
public String status() {
    return "UP";
}
~~~

### `@RestController`

Combines controller behavior with response-body behavior.

~~~java
@RestController
class StatusController {

    @GetMapping("/status")
    public String status() {
        return "UP";
    }
}
~~~

---

## 33. What Is ResponseEntity?

`ResponseEntity<T>` represents an HTTP response, including its:

- Body
- Status code
- Headers

Example:

~~~java
@GetMapping("/users/{id}")
public ResponseEntity<User> getUser(@PathVariable Long id) {

    User user = new User(id, "Alice");

    return ResponseEntity.ok(user);
}
~~~

To return a different status:

~~~java
return ResponseEntity
        .status(HttpStatus.CREATED)
        .body(user);
~~~

For a response without a body:

~~~java
return ResponseEntity.noContent().build();
~~~

`ResponseEntity` is commonly used in REST APIs when the controller needs explicit control over HTTP status and headers.

---

## 34. How Is Validation Integrated into Spring MVC?

Spring MVC can validate request data before executing business logic.

Example:

~~~java
public record CreateUserRequest(
        @NotBlank String name,
        @Email String email
) {
}
~~~

Controller:

~~~java
@PostMapping("/users")
public ResponseEntity<String> createUser(
        @Valid @RequestBody CreateUserRequest request) {

    return ResponseEntity
            .status(HttpStatus.CREATED)
            .body("User created");
}
~~~

The validation setup requires the appropriate validation implementation on the classpath.

When request validation fails, Spring MVC can raise a validation exception and return a client-error response through its exception-handling infrastructure.

### Interview point

Validation should normally happen at the request boundary, while business rules should also be enforced where necessary in the service or domain layer.

Detailed validation is covered in:

[05. Validation, Jackson & Serialization](05-Validation-and-Jackson.md)

---

## 35. How Does Spring MVC Work with the Service Layer?

A typical layered architecture is:

~~~text
HTTP Request
     |
     v
Controller
     |
     v
Service
     |
     v
Repository
     |
     v
Database
~~~

### Controller

Handles HTTP-specific concerns.

### Service

Implements business logic and coordinates operations.

### Repository

Handles persistence and database interaction.

### Why separate the layers?

- Better separation of concerns
- Easier unit testing
- Reusable business logic
- Cleaner controllers
- Easier maintenance

### Interview one-liner

> **The controller handles HTTP concerns, the service handles business logic, and the repository handles data access.**

---

## 36. Synchronous vs Asynchronous Request Processing

### Synchronous Request

In a typical synchronous request, the Servlet request-processing thread handles the request and remains involved until processing completes.

~~~text
Request
   |
   v
Servlet Thread
   |
   v
Controller -> Service -> Response
~~~

### Asynchronous Request

Spring MVC supports asynchronous request processing using suitable return types and Servlet asynchronous processing.

Examples include:

- `Callable<T>`
- `DeferredResult<T>`
- `WebAsyncTask<T>`

Conceptually:

~~~text
Request
   |
   v
Servlet Thread
   |
   v
Start Async Processing
   |
   v
Servlet Thread Released
   |
   v
Async Work Completes
   |
   v
Response Dispatched / Completed
~~~

The original Servlet thread can be released while the asynchronous result is being computed.

This does not automatically mean the application creates one new thread per request. Execution depends on the configured executor and processing model.

---

## 37. What Is the Difference Between Spring MVC and WebFlux?

| Spring MVC | Spring WebFlux |
|---|---|
| Servlet-based web framework | Reactive web framework |
| Commonly uses a thread-per-request execution model | Supports non-blocking reactive processing |
| Works well with traditional blocking libraries | Useful for non-blocking reactive pipelines |
| Uses Servlet API | Can run on Servlet containers or reactive servers |
| Supports asynchronous request processing | Designed around Reactive Streams |
| Uses familiar MVC controller model | Supports annotated controllers and functional endpoints |

### Important point

Spring MVC can support asynchronous processing, but it is not automatically a fully non-blocking reactive system.

A blocking database call in a Spring MVC application generally blocks the thread executing that call.

### Interview one-liner

> **Spring MVC is Servlet-based and commonly uses synchronous request handling, while WebFlux is designed for reactive, non-blocking processing.**

---

## 38. Spring MVC vs Spring Boot

| Spring MVC | Spring Boot |
|---|---|
| Web framework | Application framework built around Spring |
| Provides HTTP request handling and MVC infrastructure | Simplifies application setup and configuration |
| Provides DispatcherServlet, controllers, mappings, and related components | Provides auto-configuration, starters, and embedded server integration |
| Can be used without Spring Boot | Commonly used to create Spring MVC applications |

### Interview one-liner

> **Spring MVC handles web requests, while Spring Boot simplifies the configuration and deployment of applications that use Spring MVC.**

---

## 39. Spring MVC Request Flow: Complete Example

### DTO

~~~java
public record UserResponse(Long id, String name) {
}
~~~

### Service

~~~java
@Service
class UserService {

    public UserResponse findUser(Long id) {
        return new UserResponse(id, "Alice");
    }
}
~~~

### Controller

~~~java
@RestController
@RequestMapping("/api/users")
class UserController {

    private final UserService userService;

    UserController(UserService userService) {
        this.userService = userService;
    }

    @GetMapping("/{id}")
    public ResponseEntity<UserResponse> getUser(
            @PathVariable Long id) {

        UserResponse user = userService.findUser(id);

        return ResponseEntity.ok(user);
    }
}
~~~

### Request

~~~http
GET /api/users/10
Accept: application/json
~~~

### Processing flow

~~~text
1. Client sends GET /api/users/10
2. Servlet container receives the request
3. Filters execute
4. DispatcherServlet receives the request
5. HandlerMapping finds UserController.getUser()
6. HandlerAdapter resolves method arguments
7. @PathVariable resolves id = 10
8. Controller invokes UserService
9. Service returns UserResponse
10. Controller returns ResponseEntity
11. Return-value handling processes the response
12. HttpMessageConverter serializes UserResponse
13. HTTP status and headers are written
14. Client receives JSON response
~~~

### Response

~~~http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": 10,
  "name": "Alice"
}
~~~

This example illustrates the common Spring MVC REST request flow.

---

## 40. Common Interview Questions

### Q1. What is Spring MVC?

> A Servlet-based web framework that follows the MVC pattern and uses DispatcherServlet to coordinate HTTP request processing.

### Q2. What is DispatcherServlet?

> The Front Controller of Spring MVC that coordinates request mapping, handler execution, result processing, and exception resolution.

### Q3. What is the Front Controller pattern?

> A design pattern in which requests pass through a centralized controller that coordinates request processing.

### Q4. What is HandlerMapping?

> It finds the handler that matches an incoming request.

### Q5. What is HandlerAdapter?

> It invokes the selected handler and coordinates method argument and return-value processing.

### Q6. HandlerMapping vs HandlerAdapter?

> HandlerMapping finds the handler; HandlerAdapter invokes it.

### Q7. What is ViewResolver?

> It resolves a logical view name to a concrete view implementation.

### Q8. What is HttpMessageConverter?

> It converts HTTP request bodies into Java objects and Java return values into HTTP response bodies.

### Q9. Does `@RestController` use ViewResolver to return JSON?

> Generally, no. It writes return values to the response body using suitable message converters.

### Q10. `@Controller` vs `@RestController`?

> `@Controller` commonly supports view-based rendering, while `@RestController` also applies response-body behavior.

### Q11. What is ModelAndView?

> An object that carries both model data and a logical view name.

### Q12. What is a Servlet Filter?

> A Servlet-level component that intercepts requests and responses before or after they reach the target Servlet.

### Q13. What is HandlerInterceptor?

> A Spring MVC component that intercepts processing around the execution of a selected handler.

### Q14. Filter vs HandlerInterceptor?

> A Filter works at the Servlet level; a HandlerInterceptor works within Spring MVC's handler processing.

### Q15. What happens when no handler matches?

> The request typically results in HTTP 404 Not Found.

### Q16. What happens when the HTTP method does not match?

> Spring MVC typically returns HTTP 405 Method Not Allowed when the path exists but does not support that method.

### Q17. What happens when JSON conversion fails?

> Request-body conversion can fail before the controller method executes, commonly resulting in HTTP 400 Bad Request.

### Q18. What is `@RequestBody`?

> It binds the HTTP request body to a Java object through message conversion.

### Q19. What is `@ResponseBody`?

> It writes a method's return value to the HTTP response body.

### Q20. What is ResponseEntity?

> It represents an HTTP response with a status, headers, and optional body.

### Q21. How does Spring MVC handle exceptions?

> Through exception-resolution infrastructure, including HandlerExceptionResolver, `@ExceptionHandler`, and controller advice.

### Q22. What is content negotiation?

> The process of selecting an appropriate request or response representation based on media types and configuration.

### Q23. Does Spring MVC support asynchronous processing?

> Yes. It supports asynchronous request processing through mechanisms such as Callable, DeferredResult, and WebAsyncTask.

### Q24. Spring MVC vs WebFlux?

> Spring MVC is Servlet-based, while WebFlux supports reactive, non-blocking processing.

### Q25. Spring MVC vs Spring Boot?

> Spring MVC provides web request handling; Spring Boot simplifies application configuration and deployment.

---

## 41. Common Interview Traps

### Trap 1: DispatcherServlet directly executes business logic

**Incorrect.**

DispatcherServlet coordinates the request. The controller and service layer execute application logic.

### Trap 2: HandlerMapping invokes the controller

**Incorrect.**

HandlerMapping identifies the handler; HandlerAdapter invokes it.

### Trap 3: Every controller return value goes through ViewResolver

**Incorrect.**

REST response bodies are typically processed using HttpMessageConverter.

### Trap 4: `@RestController` automatically means JSON

**Not always.**

It writes the return value to the response body. The actual representation depends on the return type, content negotiation, available converters, and configuration.

### Trap 5: Filters and interceptors are the same

**Incorrect.**

Filters operate at the Servlet level; interceptors operate within Spring MVC around handler execution.

### Trap 6: Spring MVC is always blocking

**Incorrect.**

Spring MVC supports asynchronous request processing. However, asynchronous processing does not automatically make blocking operations non-blocking.

### Trap 7: A 404 and a 405 mean the same thing

**Incorrect.**

404 indicates no matching resource or handler was found. 405 indicates the resource path is known but the HTTP method is unsupported.

### Trap 8: Every exception is handled by the controller

**Incorrect.**

Exceptions can be resolved by Spring MVC's exception infrastructure, controller advice, or other configured mechanisms.

---

## 42. Quick Revision

### Spring MVC

> Servlet-based framework for handling HTTP requests using the MVC pattern.

### DispatcherServlet

> Front Controller that coordinates request processing.

### HandlerMapping

> Finds the matching handler.

### HandlerAdapter

> Invokes the selected handler.

### Controller

> Handles HTTP-specific input and response coordination.

### Service

> Contains or coordinates business logic.

### Repository

> Handles data access.

### ViewResolver

> Resolves a logical view name to a concrete view.

### HttpMessageConverter

> Converts Java objects to and from HTTP request/response bodies.

### `@Controller`

> Commonly used for view-based MVC controllers.

### `@RestController`

> Controller with response-body behavior.

### Model

> Carries attributes for a view.

### ModelAndView

> Carries both model data and a view name.

### Filter

> Intercepts requests at the Servlet level.

### HandlerInterceptor

> Intercepts Spring MVC handler execution.

### HandlerExceptionResolver

> Participates in resolving exceptions during MVC processing.

### `@RequestBody`

> Reads and converts the HTTP request body.

### `@ResponseBody`

> Writes the return value to the HTTP response body.

### ResponseEntity

> Represents status, headers, and body of an HTTP response.

### Content Negotiation

> Selects a suitable representation based on media types and configuration.

---

## 43. One-Liners to Remember

- **Spring MVC:** Servlet-based web framework for HTTP request handling.
- **Front Controller:** Central entry point for requests.
- **DispatcherServlet:** Coordinates Spring MVC request processing.
- **HandlerMapping:** Finds the handler.
- **HandlerAdapter:** Invokes the handler.
- **Controller:** Handles HTTP input and response coordination.
- **Service:** Contains business logic.
- **Repository:** Handles persistence.
- **ViewResolver:** Resolves view names.
- **HttpMessageConverter:** Converts HTTP bodies and Java objects.
- **`@Controller`:** Commonly returns view names.
- **`@RestController`:** Writes return values to response bodies.
- **Model:** Carries view attributes.
- **ModelAndView:** Carries a view and its model.
- **Filter:** Operates at the Servlet level.
- **HandlerInterceptor:** Operates around Spring MVC handler execution.
- **`@RequestBody`:** Converts request body into a Java object.
- **`@ResponseBody`:** Writes the return value into the response body.
- **ResponseEntity:** Controls response status, headers, and body.
- **404:** Resource or handler not found.
- **405:** HTTP method not supported for the resource.
- **400:** Common response for unreadable or invalid request input.
- **406:** No acceptable response representation can be produced.
- **Spring MVC vs WebFlux:** Servlet-based MVC vs reactive, non-blocking processing.
- **Spring MVC vs Spring Boot:** Web request handling vs application configuration and deployment support.

---

## 44. Related Topics

- [02. Controllers & Request Mapping](02-Controllers-and-Mappings.md)
- [03. Path Variables, Request Params, Headers & Request Body](03-Request-Parameters-and-Body.md)
- [04. ResponseEntity, HTTP Status Codes & DTOs](04-Responses-and-DTOs.md)
- [05. Validation, Jackson & Serialization](05-Validation-and-Jackson.md)
- [06. REST API Design, Versioning & HTTP Clients](06-REST-API-Design-and-Clients.md)

---

## 45. Final Interview Summary

If asked to explain the Spring MVC request flow in an interview:

> **Spring MVC uses DispatcherServlet as its Front Controller. The Servlet container receives an HTTP request and dispatches it to DispatcherServlet, which uses HandlerMapping to identify the appropriate handler and HandlerAdapter to invoke it. Spring resolves method arguments, executes the controller, and processes its return value. For REST responses, HttpMessageConverter typically serializes the returned object into the requested representation; for server-rendered pages, ViewResolver resolves the view name and the view renders the response. Spring MVC also integrates filters, interceptors, argument resolvers, validation, and exception-handling infrastructure throughout request processing.**