# A Beginner's Guide to Controller Advice: Why Your Error Handling Needs a Hero

---

## What We're Building

We're going to build a city travel API step-by-step. With each step, we'll discover why exception handling gets messy—and how `@ControllerAdvice` solves it elegantly.

By the end, you'll understand:

- How exceptions work in Spring Boot
- Why Spring's default behavior isn't always helpful
- How exceptions bubble up through your application layers
- Why Controller Advice is the hero we need

Let's dive in!

---

## Step 1: Our Simple City API

We're building an API that returns information about cities. Here's our starting point:

### The City Model

java

```java
public class City {
    private String name;
    private int population;
    private List<String> landmarks;
    
    // constructors, getters, setters...
}
```

### The Service

java

```java
@Service
public class CityService {
    
    private List<City> cities = initializeCities(); // pretend we load these
    
    public City findCity(String cityName) {
        return cities.stream()
            .filter(city -> city.getName().equalsIgnoreCase(cityName))
            .findFirst()
            .orElse(null);  // Returns null if not found
    }
}
```

### The Controller

java

```java
@RestController
@RequestMapping("/api/cities")
@RequiredArgsConstructor
public class CityController {
    
    private final CityService cityService;
    
    @GetMapping("/{cityName}")
    public City getCity(@PathVariable String cityName) {
        return cityService.findCity(cityName);
    }
}
```

### Testing Our API

Try: `GET /api/cities/Paris`

**Response:**

json

````json
{
  "name": "Paris",
  "population": 2161000,
  "landmarks": ["Eiffel Tower", "Louvre", "Notre-Dame"]
}
````

Success! Life is good.

---

## Step 2: Houston, We Have a Problem

But what happens when someone asks for a city that doesn't exist?

Try: `GET /api/cities/Atlantis`

**Response:**
```
Status: 200 OK
Body: null
````

Wait, what? The city doesn't exist, but we're telling the user "200 OK, everything's fine!" and returning `null`.

This is like asking someone for directions and they just stare at you silently. Technically not an error response, but super unhelpful.

**The Problem:** Our service returns `null`, and the controller happily passes that `null` along as a successful response.

---

## Step 3: Let's Throw an Exception

Instead of returning `null`, we should signal that something went wrong. Let's throw an exception:

### Create a Custom Exception

java

```java
public class CityNotFoundException extends RuntimeException {
    public CityNotFoundException(String cityName) {
        super("City not found: " + cityName);
    }
}
```

### Update the Service

java

```java
@Service
public class CityService {
    
    private List<City> cities = initializeCities();
    
    public City findCity(String cityName) {
        return cities.stream()
            .filter(city -> city.getName().equalsIgnoreCase(cityName))
            .findFirst()
            .orElseThrow(() -> new CityNotFoundException(cityName));  // Throw exception!
    }
}
```

Now try: `GET /api/cities/Atlantis`

**Response:**

json

````json
{
  "timestamp": "2026-02-15T10:15:30.123+00:00",
  "status": 500,
  "error": "Internal Server Error",
  "path": "/api/cities/Atlantis",
  "message": "City not found: Atlantis"
}
````

We get an error response now... but it's a **500 Internal Server Error**. That's progress, but not quite right.

---

## Step 4: Understanding Spring's Default Exception Behavior

**What just happened?** Let's understand how Spring Boot handles exceptions by default.

### The Exception Journey
```
1. Service Layer:       throw new CityNotFoundException("Atlantis")
                                      ↓
2. Controller Layer:    [No try-catch, exception continues upward]
                                      ↓
3. Spring Framework:    [Default Exception Handler activated]
                                      ↓
4. User Receives:       500 Internal Server Error
````

### Why Does Spring Return 500?

Spring Boot has a **default exception handler** that catches all uncaught exceptions. When it sees an exception it doesn't recognize (like our `CityNotFoundException`), it plays it safe and assumes something broke internally.

**Spring's logic:** "I don't know what `CityNotFoundException` means. Better assume it's our fault and send 500."

### The Problem with 500

A **500 Internal Server Error** means "_we_ messed up" (server-side bug). But in this case:

- The user asked for a city that doesn't exist
- That's not _our_ fault—it's just a resource not found
- This should be a **404 Not Found** error

---

## Step 5: Handling the Exception in Our Controller

Let's catch the exception ourselves and return a proper 404:

java

```java
@Slf4j  // Lombok: auto-creates logger
@RestController
@RequestMapping("/api/cities")
@RequiredArgsConstructor
public class CityController {
    
    private final CityService cityService;
    
    @GetMapping("/{cityName}")
    public ResponseEntity<City> getCity(@PathVariable String cityName) {
        try {
            City city = cityService.findCity(cityName);
            return ResponseEntity.ok(city);
        } catch (CityNotFoundException e) {
            log.error("City not found: {}", cityName);
            return ResponseEntity.notFound().build();
        }
    }
}
```

**What's `@Slf4j`?** It's a Lombok annotation that automatically creates this for you:

java

````java
private static final Logger log = LoggerFactory.getLogger(CityController.class);
````

Now try: `GET /api/cities/Atlantis`

**Response:**
```
Status: 404 Not Found
Body: (empty)
```

Much better! We're returning the correct HTTP status code and logging the error for debugging.

### The New Exception Flow
```
Service Layer:          throw new CityNotFoundException("Atlantis")
                                      ↓
Controller Layer:       [Catches in try-catch block]
                                      ↓
                        log.error("City not found: Atlantis")
                                      ↓
                        return 404 Not Found
                                      ↓
User Receives:          404 Not Found
````

Great! Problem solved, right?

Not quite...

---

## Step 6: Our App Gets More Complex

Our travel app is getting popular! Users want to see the **top restaurants** for each city.

### Add Restaurants to City

java

```java
public class City {
    private String name;
    private int population;
    private List<String> landmarks;
    private List<Restaurant> topRestaurants;  // NEW!
    
    // constructors, getters, setters...
}
```

### The Restaurant Model

java

```java
public class Restaurant {
    private String name;
    private String cuisine;
    private double rating;
    
    // constructors, getters, setters...
}
```

### The Challenge

Restaurant data comes from an **external API**: `https://www.supercoolrestaurants.com/api/restaurants/{cityName}`

We need to call this API to get restaurant data.

---

## Step 7: Calling External APIs with WebClient

Let's create a service to fetch restaurant data:

### The Restaurant Service

java

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class RestaurantService {
    
    private final WebClient webClient;
    
    public List<Restaurant> getTopRestaurants(String cityName) {
        return webClient.get()
            .uri("/api/restaurants/{city}", cityName)
            .retrieve()
            .bodyToFlux(Restaurant.class)  // Deserialize JSON array
            .collectList()                 // Collect into List
            .block();                      // Wait for result
    }
}
```

**What's happening here?**

- `webClient.get()` → Make a GET request
- `.uri("/api/restaurants/{city}", cityName)` → Set the URL with path variable
- `.retrieve()` → Execute the request
- `.bodyToFlux(Restaurant.class)` → Convert JSON array to stream of Restaurant objects
- `.collectList()` → Gather into a `List<Restaurant>`
- `.block()` → Block and wait for the result (synchronous call)

### Update City Service

java

````java
@Slf4j
@Service
@RequiredArgsConstructor
public class CityService {
    
    private final RestaurantService restaurantService;
    private List<City> cities = initializeCities();
    
    public City findCity(String cityName) {
        City city = cities.stream()
            .filter(c -> c.getName().equalsIgnoreCase(cityName))
            .findFirst()
            .orElseThrow(() -> new CityNotFoundException(cityName));
        
        // Enrich city with restaurant data from external API
        List<Restaurant> restaurants = restaurantService.getTopRestaurants(cityName);
        city.setTopRestaurants(restaurants);
        
        return city;
    }
}
````

Now our API returns cities WITH their top restaurants. Awesome!

---

## Step 8: When External APIs Fail

But what happens when the restaurant API has problems?

### Possible Failures

**400 Bad Request:**
```
User asks for:  GET /api/cities/Paris
We call:        https://supercoolrestaurants.com/api/restaurants/Paris
They return:    400 Bad Request (maybe they don't recognize "Paris")
```

**500 Server Error:**
```
User asks for:  GET /api/cities/Tokyo
We call:        https://supercoolrestaurants.com/api/restaurants/Tokyo
They return:    500 Internal Server Error (their server is down)
````

**What happens to our user?**

If we don't handle these errors, WebClient throws a `WebClientResponseException`, which bubbles up through our service → controller → Spring's default handler → **500 error to user**.

But it's not _our_ fault their API is down! We need better error handling.

---

## Step 9: Handling External API Errors

Let's create custom exceptions and handle API failures properly:

### Create Custom Exceptions

java

```java
public class RestaurantApiBadRequestException extends RuntimeException {
    public RestaurantApiBadRequestException(String cityName) {
        super("Restaurant API rejected city name: " + cityName);
    }
}

public class RestaurantApiServerException extends RuntimeException {
    public RestaurantApiServerException() {
        super("Restaurant API is currently unavailable");
    }
}

public class RestaurantApiException extends RuntimeException {
    public RestaurantApiException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

### Update Restaurant Service with Error Handling

java

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class RestaurantService {
    
    private final WebClient webClient;
    
    public List<Restaurant> getTopRestaurants(String cityName) {
        try {
            return webClient.get()
                .uri("/api/restaurants/{city}", cityName)
                .retrieve()
                .onStatus(
                    HttpStatus::is4xxClientError, 
                    response -> {
                        log.error("Bad request to restaurant API for city: {}", cityName);
                        return Mono.error(new RestaurantApiBadRequestException(cityName));
                    }
                )
                .onStatus(
                    HttpStatus::is5xxServerError,
                    response -> {
                        log.error("Restaurant API server error for city: {}", cityName);
                        return Mono.error(new RestaurantApiServerException());
                    }
                )
                .bodyToFlux(Restaurant.class)
                .collectList()
                .block();
        } catch (WebClientResponseException e) {
            log.error("Unexpected error calling restaurant API", e);
            throw new RestaurantApiException("Failed to fetch restaurants", e);
        }
    }
}
```

**What's `.onStatus()` doing?**

- It intercepts HTTP error responses
- When restaurant API returns 400 → We throw `RestaurantApiBadRequestException`
- When restaurant API returns 500 → We throw `RestaurantApiServerException`

Now we have **4 different exceptions** that can be thrown:

1. `CityNotFoundException` (from our city service)
2. `RestaurantApiBadRequestException` (400 from external API)
3. `RestaurantApiServerException` (500 from external API)
4. `RestaurantApiException` (unexpected errors)

---

## Step 10: The Controller Becomes a Nightmare

Remember our clean controller? Now we need to catch ALL these exceptions:

java

```java
@Slf4j
@RestController
@RequestMapping("/api/cities")
@RequiredArgsConstructor
public class CityController {
    
    private final CityService cityService;
    
    @GetMapping("/{cityName}")
    public ResponseEntity<?> getCity(@PathVariable String cityName) {
        try {
            City city = cityService.findCity(cityName);
            return ResponseEntity.ok(city);
            
        } catch (CityNotFoundException e) {
            log.error("City not found: {}", cityName);
            return ResponseEntity
                .status(HttpStatus.NOT_FOUND)
                .body(new ErrorResponse("City not found"));
                
        } catch (RestaurantApiBadRequestException e) {
            log.error("Bad request to restaurant API: {}", e.getMessage());
            return ResponseEntity
                .status(HttpStatus.BAD_REQUEST)
                .body(new ErrorResponse("Invalid city for restaurant lookup"));
                
        } catch (RestaurantApiServerException e) {
            log.error("Restaurant API is down");
            return ResponseEntity
                .status(HttpStatus.SERVICE_UNAVAILABLE)
                .body(new ErrorResponse("Restaurant service temporarily unavailable"));
                
        } catch (RestaurantApiException e) {
            log.error("Unexpected restaurant API error", e);
            return ResponseEntity
                .status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(new ErrorResponse("Failed to load restaurant data"));
        }
    }
}
```

**Yikes!** Our controller went from 5 lines to 30+ lines. And this is just ONE endpoint!

### The ErrorResponse Class

java

```java
public class ErrorResponse {
    private String message;
    private String timestamp;
    
    public ErrorResponse(String message) {
        this.message = message;
        this.timestamp = LocalDateTime.now().toString();
    }
    
    // getters, setters...
}
```

---

## Step 11: The Real Problem Emerges

Imagine our app continues to grow:

### More Endpoints

java

```java
@RestController
@RequestMapping("/api/cities")
public class CityController {
    
    @GetMapping("/{cityName}")
    public ResponseEntity<?> getCity(String cityName) { ... }
    
    @GetMapping("/{cityName}/hotels")
    public ResponseEntity<?> getHotels(String cityName) { ... }  // Needs same error handling!
    
    @GetMapping("/{cityName}/attractions")
    public ResponseEntity<?> getAttractions(String cityName) { ... }  // Needs same error handling!
}
```

### More Controllers

java

````java
@RestController
@RequestMapping("/api/countries")
public class CountryController {
    // Needs to handle CityNotFoundException too!
    // And CountryNotFoundException!
    // And RestaurantApiException!
    // And HotelApiException! (new external API)
}

@RestController
@RequestMapping("/api/regions")
public class RegionController {
    // Needs ALL the same exception handling...
}
````

### The Maintenance Nightmare

Every controller and every endpoint needs the same massive try-catch blocks. If you:
- Add a new exception → Update EVERY controller
- Change error response format → Update EVERY controller
- Change logging strategy → Update EVERY controller

**This violates DRY (Don't Repeat Yourself) principle in the worst way possible.**

### How Exceptions Actually Flow

Remember, exceptions naturally bubble up:
```
Restaurant API Error
        ↓
RestaurantService (throws RestaurantApiServerException)
        ↓
CityService (doesn't catch it, passes it up)
        ↓
CityController (must catch ALL possible exceptions)
        ↓
User (gets response based on what controller caught)
```

We're forced to handle errors at the controller level, even though the controller's job should be routing requests, not error management.

---

## Step 12: Enter the Hero — Controller Advice

What if we could intercept exceptions as they bubble up, BEFORE they reach the controller?

What if there was ONE central place to handle ALL exceptions from ALL controllers?

**That's exactly what `@ControllerAdvice` does.**

### How Controller Advice Changes Everything
```
Restaurant API Error
        ↓
RestaurantService (throws RestaurantApiServerException)
        ↓
CityService (doesn't catch it, passes it up)
        ↓
CityController (doesn't catch it, passes it up)
        ↓
@ControllerAdvice (INTERCEPTS IT HERE!) ← Our hero!
        ↓
GlobalExceptionHandler.handleRestaurantServerError()
        ↓
User (gets 503 Service Unavailable)
````

---

## Step 13: Creating Our Global Exception Handler

Let's create a `@ControllerAdvice` class:

java

```java
@Slf4j
@ControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(CityNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleCityNotFound(CityNotFoundException e) {
        log.error("City not found: {}", e.getMessage());
        return ResponseEntity
            .status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse("City not found: " + e.getMessage()));
    }
    
    @ExceptionHandler(RestaurantApiBadRequestException.class)
    public ResponseEntity<ErrorResponse> handleRestaurantBadRequest(
            RestaurantApiBadRequestException e) {
        log.error("Bad request to restaurant API: {}", e.getMessage());
        return ResponseEntity
            .status(HttpStatus.BAD_REQUEST)
            .body(new ErrorResponse("Invalid city for restaurant lookup"));
    }
    
    @ExceptionHandler(RestaurantApiServerException.class)
    public ResponseEntity<ErrorResponse> handleRestaurantServerError(
            RestaurantApiServerException e) {
        log.error("Restaurant API server error");
        return ResponseEntity
            .status(HttpStatus.SERVICE_UNAVAILABLE)
            .body(new ErrorResponse("Restaurant service temporarily unavailable"));
    }
    
    @ExceptionHandler(RestaurantApiException.class)
    public ResponseEntity<ErrorResponse> handleRestaurantApiError(
            RestaurantApiException e) {
        log.error("Unexpected restaurant API error", e);
        return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(new ErrorResponse("Failed to load restaurant data"));
    }
    
    // Catch-all for any unexpected exceptions
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGenericException(Exception e) {
        log.error("Unexpected error occurred", e);
        return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(new ErrorResponse("An unexpected error occurred"));
    }
}
```

### Understanding the Annotations

**`@ControllerAdvice`**

- Tells Spring: "This class handles exceptions globally for ALL controllers"
- Think of it as a global safety net

**`@ExceptionHandler(CityNotFoundException.class)`**

- Tells Spring: "When a `CityNotFoundException` is thrown from ANY controller, route it to THIS method"
- You can handle multiple exception types in one method: `@ExceptionHandler({Exception1.class, Exception2.class})`

---

## Step 14: Our Controller Becomes Clean Again

Now we can remove ALL error handling from our controller:

java

````java
@Slf4j
@RestController
@RequestMapping("/api/cities")
@RequiredArgsConstructor
public class CityController {
    
    private final CityService cityService;
    
    @GetMapping("/{cityName}")
    public City getCity(@PathVariable String cityName) {
        return cityService.findCity(cityName);
    }
    
    @GetMapping("/{cityName}/hotels")
    public List<Hotel> getHotels(@PathVariable String cityName) {
        return hotelService.findHotels(cityName);
    }
    
    @GetMapping("/{cityName}/attractions")
    public List<Attraction> getAttractions(@PathVariable String cityName) {
        return attractionService.findAttractions(cityName);
    }
}
````

**Look at that beautiful, clean code!**

No try-catch blocks. No error handling logic. Just pure business logic.

All exceptions automatically bubble up to `GlobalExceptionHandler` and are handled consistently.

---

## Step 15: How It All Works Together

Let's trace a request that fails:

### User Request
```
GET /api/cities/UnknownCity
```

### The Journey
```
1. CityController.getCity("UnknownCity")
        ↓
2. CityService.findCity("UnknownCity")
        ↓
3. RestaurantService.getTopRestaurants("UnknownCity")
        ↓
4. External API returns 400 Bad Request
        ↓
5. RestaurantService throws RestaurantApiBadRequestException
        ↓
6. Exception bubbles up through CityService (not caught)
        ↓
7. Exception bubbles up through CityController (not caught)
        ↓
8. Spring's Exception Resolution Chain activates
        ↓
9. Spring checks: "Do I have an @ExceptionHandler for RestaurantApiBadRequestException?"
        ↓
10. YES! Routes to GlobalExceptionHandler.handleRestaurantBadRequest()
        ↓
11. Handler logs error and returns ResponseEntity with 400 status
        ↓
12. User receives: 400 Bad Request with error message
````

### Spring's Exception Resolution Order

When an exception occurs, Spring tries to handle it in this priority order:

1. **Controller-level `@ExceptionHandler`** (defined inside the controller itself)
2. **`@ControllerAdvice` with `@ExceptionHandler`** (our global handler)
3. **Spring's default exception resolvers** (the ones that return generic 500 errors)

Since we didn't catch the exception in the controller, it goes straight to our `@ControllerAdvice`.

---

## Step 16: The Benefits Become Clear

### Before Controller Advice ❌

java

```java
// CityController
@GetMapping("/{cityName}")
public ResponseEntity<?> getCity(String cityName) {
    try {
        return ResponseEntity.ok(cityService.findCity(cityName));
    } catch (CityNotFoundException e) { ... }
    catch (RestaurantApiBadRequestException e) { ... }
    catch (RestaurantApiServerException e) { ... }
    catch (RestaurantApiException e) { ... }
}

// CountryController - DUPLICATE ERROR HANDLING!
@GetMapping("/{countryName}")
public ResponseEntity<?> getCountry(String countryName) {
    try {
        return ResponseEntity.ok(countryService.findCountry(countryName));
    } catch (CountryNotFoundException e) { ... }
    catch (RestaurantApiBadRequestException e) { ... }  // Same!
    catch (RestaurantApiServerException e) { ... }      // Same!
    catch (RestaurantApiException e) { ... }            // Same!
}

// RegionController - MORE DUPLICATION!
@GetMapping("/{regionName}")
public ResponseEntity<?> getRegion(String regionName) {
    // ... yet another copy of the same error handling
}
```

### After Controller Advice ✅

java

```java
// CityController - CLEAN!
@GetMapping("/{cityName}")
public City getCity(@PathVariable String cityName) {
    return cityService.findCity(cityName);
}

// CountryController - CLEAN!
@GetMapping("/{countryName}")
public Country getCountry(@PathVariable String countryName) {
    return countryService.findCountry(countryName);
}

// RegionController - CLEAN!
@GetMapping("/{regionName}")
public Region getRegion(@PathVariable String regionName) {
    return regionService.findRegion(regionName);
}

// GlobalExceptionHandler - ONE PLACE FOR ALL ERROR HANDLING!
@ControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(RestaurantApiBadRequestException.class)
    public ResponseEntity<ErrorResponse> handleRestaurantBadRequest(...) {
        // Handles this exception from ALL controllers
    }
}
```

---

## Step 17: Real-World Benefits

### 1. **Consistency**

All endpoints return errors in the same format:

json

```json
{
  "message": "City not found: Atlantis",
  "timestamp": "2026-02-15T14:23:45"
}
```

No matter which controller or endpoint throws the error, the response structure is identical.

### 2. **Maintainability**

Need to change error responses? Update ONE class:

java

```java
@ControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(CityNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleCityNotFound(CityNotFoundException e) {
        // Change error format here
        return ResponseEntity
            .status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse(
                e.getMessage(),
                "CITY_NOT_FOUND",  // NEW: Error code
                LocalDateTime.now(),
                "Contact support if this persists"  // NEW: Help message
            ));
    }
}
```

This change applies to EVERY controller automatically.

### 3. **Monitoring and Logging**

Centralized logging makes debugging easier:

java

```java
@ExceptionHandler(RestaurantApiServerException.class)
public ResponseEntity<ErrorResponse> handleRestaurantServerError(
        RestaurantApiServerException e, 
        HttpServletRequest request) {
    
    log.error("Restaurant API failure - Endpoint: {} - User: {} - Time: {}", 
        request.getRequestURI(),
        request.getRemoteUser(),
        LocalDateTime.now());
    
    // Could also send alert to monitoring system
    alertingService.sendAlert("Restaurant API Down", e);
    
    return ResponseEntity
        .status(HttpStatus.SERVICE_UNAVAILABLE)
        .body(new ErrorResponse("Restaurant service temporarily unavailable"));
}
```

### 4. **Separation of Concerns**

Controllers focus on routing and business logic. Exception handling is separate.

java

```java
// Controller = Routing
@RestController
public class CityController {
    public City getCity(String name) { ... }
}

// Service = Business Logic  
@Service
public class CityService {
    public City findCity(String name) { ... }
}

// ControllerAdvice = Error Handling
@ControllerAdvice
public class GlobalExceptionHandler {
    public ResponseEntity<ErrorResponse> handleCityNotFound(...) { ... }
}
```

Clean architecture!

---

## Step 18: Common Patterns and Best Practices

### Pattern 1: Multiple Exceptions, Same Response

java

```java
@ExceptionHandler({
    CityNotFoundException.class,
    CountryNotFoundException.class,
    RegionNotFoundException.class
})
public ResponseEntity<ErrorResponse> handleNotFound(RuntimeException e) {
    log.error("Resource not found: {}", e.getMessage());
    return ResponseEntity
        .status(HttpStatus.NOT_FOUND)
        .body(new ErrorResponse(e.getMessage()));
}
```

### Pattern 2: Accessing Request Details

java

```java
@ExceptionHandler(CityNotFoundException.class)
public ResponseEntity<ErrorResponse> handleCityNotFound(
        CityNotFoundException e,
        HttpServletRequest request) {
    
    log.error("City not found - Path: {} - Query: {}", 
        request.getRequestURI(),
        request.getQueryString());
    
    return ResponseEntity
        .status(HttpStatus.NOT_FOUND)
        .body(new ErrorResponse("City not found"));
}
```

### Pattern 3: Validation Errors

java

```java
@ExceptionHandler(MethodArgumentNotValidException.class)
public ResponseEntity<ErrorResponse> handleValidationErrors(
        MethodArgumentNotValidException e) {
    
    List<String> errors = e.getBindingResult()
        .getFieldErrors()
        .stream()
        .map(error -> error.getField() + ": " + error.getDefaultMessage())
        .collect(Collectors.toList());
    
    log.error("Validation failed: {}", errors);
    
    return ResponseEntity
        .status(HttpStatus.BAD_REQUEST)
        .body(new ErrorResponse("Validation failed", errors));
}
```

### Pattern 4: Specific Controller Advice

You can limit `@ControllerAdvice` to specific packages or controllers:

java

```java
// Only handles exceptions from controllers in this package
@ControllerAdvice(basePackages = "com.example.api.cities")
public class CityExceptionHandler {
    // ...
}

// Only handles exceptions from these specific controllers
@ControllerAdvice(assignableTypes = {CityController.class, CountryController.class})
public class LocationExceptionHandler {
    // ...
}
```

---

## Recap: The Complete Picture

### What We Built

1. **Started simple**: Basic city API with null returns
2. **Introduced exceptions**: Threw `CityNotFoundException` instead of returning null
3. **Handled in controller**: Added try-catch blocks (got messy)
4. **Added complexity**: External restaurant API with multiple error scenarios
5. **Controller exploded**: Try-catch blocks became unmanageable
6. **Introduced Controller Advice**: Centralized ALL exception handling

### Key Concepts

**How Exceptions Work in Spring:**

- Exceptions thrown in services bubble up to controllers
- If not caught, they bubble up to Spring's exception resolution chain
- Spring's default behavior: return 500 for unknown exceptions
- Controller Advice intercepts exceptions BEFORE Spring's default handler

**Why Controller Advice Matters:**

- ✅ One place to handle ALL exceptions from ALL controllers
- ✅ Consistent error responses across your entire API
- ✅ Clean, readable controllers focused on business logic
- ✅ Easy to maintain and modify error handling
- ✅ Separation of concerns (routing vs error handling)
- ✅ DRY principle (Don't Repeat Yourself)

### The Magic Annotations

java

```java
@ControllerAdvice              // "I handle exceptions globally"
@ExceptionHandler(Exception.class)  // "I handle this exception type"
@Slf4j                         // "Give me a logger" (Lombok)
@RequiredArgsConstructor       // "Generate constructor for final fields" (Lombok)
```