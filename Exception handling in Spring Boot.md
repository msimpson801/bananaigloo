# A Beginner's Guide to Exception Handling in Spring Boot REST APIs

If you're building REST APIs with Spring Boot, you'll quickly run into a common question: _What should happen when something goes wrong?_ Maybe a user requests data that doesn't exist, or tries to create something that's already there. How do you handle these situations cleanly?

In this guide, we'll walk through exception handling step by step, starting from the simplest (but problematic) approach and evolving toward a professional, scalable solution. We'll use a simple Country API as our example throughout.

## The Starting Point: A Simple Country Service

Let's imagine we're building an API that provides information about countries. Here's our basic setup:

java

```java
@Service
public class CountryService {
    private final List<Country> countries = List.of(
        Country.builder().name("Italy").code("IT").population(59554023).build(),
        Country.builder().name("France").code("FR").population(65426179).build(),
        Country.builder().name("Japan").code("JP").population(125124989).build()
    );

    public Country getCountryByName(String name) {
        return countries.stream()
                .filter(c -> c.getName().equalsIgnoreCase(name))
                .findFirst()
                .orElse(null);
    }
}
```

And a simple controller to expose it:

java

```java
@RestController
@RequestMapping("/countries")
public class CountryController {
    private final CountryService countryService;

    @GetMapping("/getcountrybyname/{name}")
    public Country getCountryByName(@PathVariable String name) {
        return countryService.getCountryByName(name);
    }
}
```

## The Problem with Returning `null`

What happens when someone asks for a country that doesn't exist, like "Atlantis"?

With our current code, the service returns `null`, and Spring returns an HTTP 200 OK status with an empty response body. **This is misleading.** In REST API design, HTTP 200 means "success" – but there was no success here. The country wasn't found, yet we're telling the client everything worked fine.

This is confusing for anyone using your API and makes debugging harder.

## Step 1: Express Failure with Exceptions

Instead of returning `null`, let's be explicit about what went wrong. We'll create a custom exception:

java

```java
public class CountryNotFoundException extends RuntimeException {
    public CountryNotFoundException(String name) {
        super("Country not found: " + name);
    }
}
```

Now we update our service to throw this exception when a country isn't found:

java

```java
public Country getCountryByName(String name) {
    return countries.stream()
            .filter(c -> c.getName().equalsIgnoreCase(name))
            .findFirst()
            .orElseThrow(() -> new CountryNotFoundException(name));
}
```

**Progress made:** Our code now clearly communicates failure.

**New problem:** Spring returns HTTP 500 Internal Server Error. While this indicates something went wrong, a 500 status suggests a server malfunction, not a missing resource. We should return 404 Not Found instead.

## Step 2: Handling Exceptions in Controllers (The Tempting But Wrong Path)

A natural next step is to catch the exception in your controller:

java

```java
@GetMapping("/getcountrybyname/{name}")
public ResponseEntity<?> getCountryByName(@PathVariable String name) {
    try {
        return ResponseEntity.ok(countryService.getCountryByName(name));
    } catch (CountryNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
                             .body(ex.getMessage());
    }
}
```

This works! When someone requests a non-existent country, they get a proper 404 response.

**But here's the problem:** Imagine you have 20 endpoints. Each one might throw different exceptions. You'd need try-catch blocks everywhere, repeating the same error-handling logic over and over. Your controllers become bloated with repetitive code that has nothing to do with their actual purpose.

This approach doesn't scale. There must be a better way.

## Step 3: Global Exception Handling with `@RestControllerAdvice`

Spring Boot provides a powerful feature for handling exceptions globally across your entire application. Instead of catching exceptions in each controller, you create a single class that handles them all:

java

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(CountryNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public String handleCountryNotFound(CountryNotFoundException ex) {
        return ex.getMessage();
    }
}
```

Now your controller can go back to being simple:

java

```java
@GetMapping("/getcountrybyname/{name}")
public Country getCountryByName(@PathVariable String name) {
    return countryService.getCountryByName(name);
}
```

**What's happening here?** When `CountryNotFoundException` is thrown anywhere in your application, Spring automatically routes it to your `GlobalExceptionHandler`. The handler converts it into the appropriate HTTP response (404 Not Found) with the exception's message.

Your controller stays focused on what it should do: handling the request. The exception handler stays focused on what it should do: translating exceptions into proper HTTP responses.

## Step 4: Professional Error Responses with a Custom Error Object

So far, we've been returning simple string messages. While this works, professional APIs typically return structured error responses with consistent formatting. Let's create a proper error response object:

java

```java
@Data
@AllArgsConstructor
public class ApiErrorResponse {
    private int status;
    private String message;
}
```

Now we can return structured error information:

java

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(CountryNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ApiErrorResponse handleCountryNotFound(CountryNotFoundException ex) {
        return new ApiErrorResponse(
            HttpStatus.NOT_FOUND.value(),
            ex.getMessage()
        );
    }
}
```

When a country isn't found, the API now returns a clean JSON response:

json

```json
{
    "status": 404,
    "message": "Country not found: Atlantis"
}
```

This gives API consumers:

- **Consistent structure** across all error responses
- **HTTP status code** in the response body for easy parsing
- **Specific details** about what went wrong

## Step 5: Scaling Up – Adding More Exceptions

Let's say you add a feature to create new countries. What if someone tries to create a country that already exists? You'd want to return a 409 Conflict status.

Simply create a new exception:

java

```java
@ResponseStatus(HttpStatus.CONFLICT)
public class CountryAlreadyExistsException extends RuntimeException {
    public CountryAlreadyExistsException(String name) {
        super("Country already exists: " + name);
    }
}
```

Use it in your service:

java

```java
public void addCountry(Country country) {
    if (countryExists(country.getName())) {
        throw new CountryAlreadyExistsException(country.getName());
    }
    // proceed with adding country
}
```

Add one method to your global handler:

java

```java
@ExceptionHandler(CountryAlreadyExistsException.class)
@ResponseStatus(HttpStatus.CONFLICT)
public ApiErrorResponse handleCountryAlreadyExists(CountryAlreadyExistsException ex) {
    return new ApiErrorResponse(
        HttpStatus.CONFLICT.value(),
        ex.getMessage()
    );
}
```

That's it. No controller changes needed. The system automatically knows how to handle this new exception everywhere it might occur.

## The Final Architecture: Clean Separation of Concerns

Here's how the pieces fit together:

1. **Controller** receives the request and calls the service
2. **Service** contains your business logic and throws domain-specific exceptions when something goes wrong
3. **`@RestControllerAdvice`** catches exceptions globally and formats them into `ApiErrorResponse` objects
4. **`@ResponseStatus`** on each handler method defines what HTTP status code to return

This architecture gives you:

- **Clean controllers** that focus only on routing requests
- **Expressive services** that clearly communicate what went wrong
- **Centralized error handling** that ensures consistent behavior
- **Easy scalability** as you add new features and exception types

## Key Principles to Remember

**Don't return `null` for errors.** It hides what really happened and leads to confusing API responses.

**Don't handle exceptions in controllers.** You'll end up with repetitive code that's hard to maintain.

**Use `@RestControllerAdvice` for global handling.** It centralizes error handling and ensures consistent, structured error responses across your entire application.

**Return structured error objects.** Use a consistent `ApiErrorResponse` format rather than plain strings for professional, parseable error messages.

**Use `@ResponseStatus` on handler methods.** This clearly declares what HTTP status each exception type should produce.

**Think in terms of domain exceptions.** Create exceptions that represent what went wrong in your business logic, not just generic errors.