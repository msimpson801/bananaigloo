# 🧀 REST Clients in Java — The Cheesy Way

> Spring 6 / Spring Boot 3 · Blocking Calls Only · Zero WebFlux Required

---

##  01 — The Setup: Your Cheese API

We're hitting a fictional **Cheese API** at `https://api.cheese.io/v1`. Two endpoints:

| Method   | Path              | Description         |
| -------- | ----------------- | ------------------- |
| `GET`    | `/cheeses`        | List all cheeses    |
| `GET`    | `/cheeses/{name}` | Get a single cheese |
| `DELETE` | `/cheeses/{name}` | Delete a cheese     |

Build a `RestClient` bean once, reuse everywhere. This is Spring 6's shiny synchronous HTTP client — no Reactor, no flux, no fuss.

```java
@Configuration
public class CheeseClientConfig {

    @Bean
    public RestClient cheeseRestClient(RestClient.Builder builder) {
        return builder
            .baseUrl("https://api.cheese.io/v1")
            .defaultHeader("Accept", "application/json")
            .build();
    }
}
```

> 💡 **Tip:** Spring Boot auto-configures a `RestClient.Builder` with sensible defaults (connection pooling, timeouts from your `application.yml`). Inject the builder, not a raw `RestClient.create()`.

---

## 02 — Your Domain Models

Two records — one for a single cheese, one wrapping the list response.

```java
// A single cheese
public record Cheese(
    Long   id,
    String name,
    String origin,
    String texture,
    double pungencyLevel,  // 0.0 = mild brie, 10.0 = limburger at 3am
    String description
) {}

// The list endpoint wraps results in a "cheeses" array
public record CheeseList(
    List<Cheese> cheeses
) {}
```

---

## 03 — Fetching Cheeses

### 🗂 All the cheeses

Use `ParameterizedTypeReference` when you need a generic type like `List<T>` — Jackson can't infer it at runtime otherwise. Or if the API wraps its list in an object, just deserialise that directly:

```java
@Service
public class CheeseService {

    private final RestClient restClient;

    public CheeseList listCheeses() {
        return restClient
            .get()
            .uri("/cheeses")
            .retrieve()
            .body(CheeseList.class);
    }
```

### 🔎 A single cheese by name

Path variables slot neatly into the URI template — no string concatenation needed.

```java
    public Cheese getCheese(String name) {
        return restClient
            .get()
            .uri("/cheeses/{name}", name)
            .retrieve()
            .body(Cheese.class);
    }
```

> 💡 **Tip:** `.retrieve()` is the happy path — it deserialises the body and throws `RestClientResponseException` on 4xx/5xx automatically. For more control, read on to § 04.

---

## 04 — `onStatus` — Sad Paths & Happy Paths

Sometimes the API cries. `.onStatus()` lets you intercept specific HTTP status codes and throw meaningful exceptions instead of letting Spring throw a generic one.

- ✅ **200 OK** — 🎉 brie, sweet and mild
- ❌ **404** — 😢 asked for foot cheese, nothing came back
- ❌ **5xx** — 😭 our local cheesemonger is down — could be the server crashed, ran out of memory, or Dave tripped over the power cable again

```java
    public Cheese getCheeseOrCry(String name) {
        return restClient
            .get()
            .uri("/cheeses/{name}", name)
            .retrieve()
            .onStatus(
                HttpStatusCode::is4xxClientError,       // predicate
                (request, response) -> {                // error handler
                    throw new CheeseNotFoundException(
                        "No cheese called '%s' 😢".formatted(name)
                    );
                }
            )
            .onStatus(
                HttpStatusCode::is5xxServerError,
                (request, response) -> {
                    throw new CheeseMongerDownException(
                        "The cheesemonger is unavailable 🧀💀"
                    );
                }
            )
            .body(Cheese.class);
    }
```

> ⚠️ **Watch out:** The error handler lambda **must throw or return** — it can't return null. The response body stream is still open inside the lambda, so read it if you want to include API error messages in your exception.

Want to include the error body from the API? Read it inside the handler:

```java
            .onStatus(
                status -> status.value() == 404,
                (request, response) -> {
                    // Read the error JSON from the API
                    String errorBody = new String(
                        response.getBody().readAllBytes(),
                        StandardCharsets.UTF_8
                    );
                    throw new CheeseNotFoundException("API said: " + errorBody);
                }
            )
```

---

## 05 — Bodyless Requests — Delete That Cheese

Some calls return nothing useful (204 No Content, or a bare 200). Use `.toBodilessEntity()` to get the `ResponseEntity` without touching the body — otherwise Spring will try to deserialise nothing and get upset.

```java
    public void deleteCheese(String name) {
        ResponseEntity<Void> response = restClient
            .delete()
            .uri("/cheeses/{name}", name)
            .retrieve()
            .onStatus(
                status -> status.value() == 404,
                (req, res) -> { throw new CheeseNotFoundException("Already gone!"); }
            )
            .toBodilessEntity();   // returns ResponseEntity<Void>

        if (response.getStatusCode().is2xxSuccessful()) {
            log.info("Cheese '{}' has been eliminated. Farewell.", name);
        }
    }
```

**Alternatives at a glance:**

|Method|Returns|Use when|
|---|---|---|
|`.toBodilessEntity()`|`ResponseEntity<Void>`|You want the headers/status|
|`.body(Void.class)`|`null`|You only care it didn't explode|
|`.retrieve()` with no terminal|_(nothing)_|❌ Don't — the call won't execute!|

---

## 06 — `exchange()` — Advanced Cheese Operations

`.exchange()` is the escape hatch. Instead of Spring processing the response, _you_ get the raw `ClientHttpResponse` and do whatever you want with it. Great for conditional logic based on status, reading headers, handling multiple possible response types, or streaming.

> ⚠️ **Warning:** With `.exchange()`, **you are responsible for consuming or closing the response body**. If you don't, you'll leak connections.

```java
    /**
     * Fetch a cheese — but if it's 404, return an empty Optional
     * instead of throwing. If it's anything else bad, still throw.
     */
    public Optional<Cheese> findCheese(String name) {
        return restClient
            .get()
            .uri("/cheeses/{name}", name)
            .exchange((request, response) -> {

                if (response.getStatusCode().value() == 404) {
                    return Optional.empty();  // foot cheese? never heard of it
                }

                if (response.getStatusCode().isError()) {
                    throw new CheeseApiException(
                        "Unexpected status: " + response.getStatusCode()
                    );
                }

                // All good — brie it is
                Cheese cheese = response.bodyTo(Cheese.class);
                return Optional.ofNullable(cheese);
            });
    }
```

You can also send custom **request headers** — useful when the API expects a username, tenant, or trace ID per call. Stuart wants brie:

```java
    public Cheese getCheeseForUser(String name, String username) {
        return restClient
            .get()
            .uri("/cheeses/{name}", name)
            .header("X-Cheese-User", username)   // "Stuart"
            .header("X-Request-ID", UUID.randomUUID().toString())
            .exchange((request, response) -> {

                if (response.getStatusCode().value() == 403) {
                    throw new CheeseAccessDeniedException(
                        username + " is not allowed near the brie 🚫"
                    );
                }

                if (response.getStatusCode().isError()) {
                    throw new CheeseApiException(
                        "Unexpected status: " + response.getStatusCode()
                    );
                }

                return response.bodyTo(Cheese.class);
            });
    }
```

```java
// Stuart wants brie
Cheese brie = cheeseService.getCheeseForUser("brie", "Stuart");
```

**When to use what:**

|Use `retrieve()` when…|Use `exchange()` when…|
|---|---|
|Happy path + simple error handling|Multiple possible response types|
|Throwing on 4xx/5xx is fine|You want `Optional` instead of exceptions|
|You just want the body|You need per-call request headers|
|Standard JSON deserialisation|Streaming or raw byte access|

---

## 07 — Errors Without Throwing Exceptions

`onStatus` is great when you _want_ to throw. But sometimes you don't — you just want a null or an empty `Optional` back and let the caller decide what to do.

The trick is to skip `retrieve()` and use `exchange()` instead. With `exchange()` you own the response — no automatic exception throwing, you just check the status yourself:

```java
public Cheese getCheeseOrNull(String name) {
    return restClient
        .get()
        .uri("/cheeses/{name}", name)
        .exchange((request, response) -> {
            if (response.getStatusCode().isError()) {
                return null;   // foot cheese? cheesemonger down? either way, null
            }
            return response.bodyTo(Cheese.class);
        });
}
```

Prefer an `Optional`? Same idea:

```java
public Optional<Cheese> findCheese(String name) {
    return restClient
        .get()
        .uri("/cheeses/{name}", name)
        .exchange((request, response) -> {
            if (response.getStatusCode().isError()) {
                return Optional.empty();
            }
            return Optional.ofNullable(response.bodyTo(Cheese.class));
        });
}
```

Or just a plain try/catch if you want to keep using `retrieve()` and handle it at the call site:

```java
try {
    return restClient.get()
        .uri("/cheeses/{name}", name)
        .retrieve()
        .body(Cheese.class);
} catch (RestClientResponseException ex) {
    log.warn("Cheese '{}' not available: {}", name, ex.getStatusCode());
    return null;
}
```

> ⚠️ **Warning:** With `exchange()`, **you are responsible for consuming or closing the response body**. If you just return null without reading the body, make sure to drain or close the stream to avoid leaking connections.

---

##  08 — Why Blocking is a easier to Debug in IntelliJ

This is _the_ hidden superpower of synchronous REST clients.

With `RestClient`, the entire HTTP call happens on **your thread**, synchronously. Set a breakpoint anywhere in the call chain and IntelliJ stops exactly there, with a normal, readable call stack.

**Step-by-step debugging wins:**

1. **Set a breakpoint inside your `onStatus` lambda** — IntelliJ stops right there, on your thread, with the full response object in scope. Hover over `response.getStatusCode()` and see the actual value.
    
2. **Step into `.body(Cheese.class)`** — watch Jackson walk through your JSON field by field. Instantly see if a field is null because it's missing from the response or because your field name doesn't match.
    
3. **The call stack is linear** — `YourService.getCheese()` → `RestClient.retrieve()` → `HttpClient.send()`. No reactor schedulers, no operator chains, no thread-hopping. What you see is what happened, in order.
    
4. **Use IntelliJ's HTTP Client** (`.http` files) to replay the exact same request against your local app — right-click the URI string in your code and choose "Open in HTTP Client".
    

> ⚠️ **Compare: WebFlux / Reactive** With `WebClient`, a breakpoint inside a `.map()` or `.flatMap()` shows a Reactor operator stack — not your code. The actual HTTP call may have happened on a Netty I/O thread, and the response processing on yet another. You're debugging a pipeline, not a sequence.

**Rule of thumb:** Use blocking `RestClient` for everything until you actually have a throughput problem that reactive solves. Premature reactification is the root of many debugging nightmares.

---

## Quick Reference Cheat Sheet 🧀

|What you want|How|
|---|---|
|Simple GET, typed body|`.retrieve().body(Cheese.class)`|
|GET list of things|`.retrieve().body(new ParameterizedTypeReference<>() {})`|
|Handle specific status code|`.retrieve().onStatus(pred, handler)`|
|No response body (DELETE/PUT)|`.retrieve().toBodilessEntity()`|
|Full response control|`.exchange((req, res) -> ...)`|
|Return null instead of throwing|`exchange()` + check `isError()` manually|
|Return `Optional.empty()` on error|`exchange()` + return `Optional.empty()`|
|Read response headers too|`.retrieve().toEntity(Cheese.class)`|
|Add query params|`.uri("/cheeses?page={p}", page)`|
|Per-call request headers|`.header("X-User", username)` before `.exchange()`|

---

_Built with Spring Framework 6.x · Spring Boot 3.x · Jackson 2.x · 100% real cheese enthusiasm_

_🧀 No cheeses were harmed in the making of this guide 🧀_# 🧀 REST Clients in Java — The Cheesy Way

> Spring 6 / Spring Boot 3 · Blocking Calls Only · Zero WebFlux Required

---

## § 01 — The Setup: Your Cheese API

We're hitting a fictional **Cheese API** at `https://api.cheese.io/v1`. Two endpoints:

|Method|Path|Description|
|---|---|---|
|`GET`|`/cheeses`|List all cheeses|
|`GET`|`/cheeses/{name}`|Get a single cheese|
|`DELETE`|`/cheeses/{name}`|Delete a cheese|

Build a `RestClient` bean once, reuse everywhere. This is Spring 6's shiny synchronous HTTP client — no Reactor, no flux, no fuss.

```java
@Configuration
public class CheeseClientConfig {

    @Bean
    public RestClient cheeseRestClient(RestClient.Builder builder) {
        return builder
            .baseUrl("https://api.cheese.io/v1")
            .defaultHeader("Accept", "application/json")
            .build();
    }
}
```

> 💡 **Tip:** Spring Boot auto-configures a `RestClient.Builder` with sensible defaults (connection pooling, timeouts from your `application.yml`). Inject the builder, not a raw `RestClient.create()`.

---

## § 02 — Your Domain Models

Two records — one for a single cheese, one wrapping the list response.

```java
// A single cheese
public record Cheese(
    Long   id,
    String name,
    String origin,
    String texture,
    double pungencyLevel,  // 0.0 = mild brie, 10.0 = limburger at 3am
    String description
) {}

// The list endpoint wraps results in a "cheeses" array
public record CheeseList(
    List<Cheese> cheeses
) {}
```

---

## § 03 — Fetching Cheeses

### 🗂 All the cheeses

Use `ParameterizedTypeReference` when you need a generic type like `List<T>` — Jackson can't infer it at runtime otherwise. Or if the API wraps its list in an object, just deserialise that directly:

```java
@Service
public class CheeseService {

    private final RestClient restClient;

    public CheeseList listCheeses() {
        return restClient
            .get()
            .uri("/cheeses")
            .retrieve()
            .body(CheeseList.class);
    }
```

### 🔎 A single cheese by name

Path variables slot neatly into the URI template — no string concatenation needed.

```java
    public Cheese getCheese(String name) {
        return restClient
            .get()
            .uri("/cheeses/{name}", name)
            .retrieve()
            .body(Cheese.class);
    }
```

> 💡 **Tip:** `.retrieve()` is the happy path — it deserialises the body and throws `RestClientResponseException` on 4xx/5xx automatically. For more control, read on to § 04.

---

## § 04 — `onStatus` — Sad Paths & Happy Paths

Sometimes the API cries. `.onStatus()` lets you intercept specific HTTP status codes and throw meaningful exceptions instead of letting Spring throw a generic one.

- ✅ **200 OK** — 🎉 brie, sweet and mild
- ❌ **404** — 😢 asked for foot cheese, nothing came back
- ❌ **5xx** — 😭 our local cheesemonger is down — could be the server crashed, ran out of memory, or Dave tripped over the power cable again

```java
    public Cheese getCheeseOrCry(String name) {
        return restClient
            .get()
            .uri("/cheeses/{name}", name)
            .retrieve()
            .onStatus(
                HttpStatusCode::is4xxClientError,       // predicate
                (request, response) -> {                // error handler
                    throw new CheeseNotFoundException(
                        "No cheese called '%s' 😢".formatted(name)
                    );
                }
            )
            .onStatus(
                HttpStatusCode::is5xxServerError,
                (request, response) -> {
                    throw new CheeseMongerDownException(
                        "The cheesemonger is unavailable 🧀💀"
                    );
                }
            )
            .body(Cheese.class);
    }
```

> ⚠️ **Watch out:** The error handler lambda **must throw or return** — it can't return null. The response body stream is still open inside the lambda, so read it if you want to include API error messages in your exception.

Want to include the error body from the API? Read it inside the handler:

```java
            .onStatus(
                status -> status.value() == 404,
                (request, response) -> {
                    // Read the error JSON from the API
                    String errorBody = new String(
                        response.getBody().readAllBytes(),
                        StandardCharsets.UTF_8
                    );
                    throw new CheeseNotFoundException("API said: " + errorBody);
                }
            )
```

---

## § 05 — Bodyless Requests — Delete That Cheese

Some calls return nothing useful (204 No Content, or a bare 200). Use `.toBodilessEntity()` to get the `ResponseEntity` without touching the body — otherwise Spring will try to deserialise nothing and get upset.

```java
    public void deleteCheese(String name) {
        ResponseEntity<Void> response = restClient
            .delete()
            .uri("/cheeses/{name}", name)
            .retrieve()
            .onStatus(
                status -> status.value() == 404,
                (req, res) -> { throw new CheeseNotFoundException("Already gone!"); }
            )
            .toBodilessEntity();   // returns ResponseEntity<Void>

        if (response.getStatusCode().is2xxSuccessful()) {
            log.info("Cheese '{}' has been eliminated. Farewell.", name);
        }
    }
```

**Alternatives at a glance:**

|Method|Returns|Use when|
|---|---|---|
|`.toBodilessEntity()`|`ResponseEntity<Void>`|You want the headers/status|
|`.body(Void.class)`|`null`|You only care it didn't explode|
|`.retrieve()` with no terminal|_(nothing)_|❌ Don't — the call won't execute!|

---

## § 06 — `exchange()` — Advanced Cheese Operations

`.exchange()` is the escape hatch. Instead of Spring processing the response, _you_ get the raw `ClientHttpResponse` and do whatever you want with it. Great for conditional logic based on status, reading headers, handling multiple possible response types, or streaming.

> ⚠️ **Warning:** With `.exchange()`, **you are responsible for consuming or closing the response body**. If you don't, you'll leak connections.

```java
    /**
     * Fetch a cheese — but if it's 404, return an empty Optional
     * instead of throwing. If it's anything else bad, still throw.
     */
    public Optional<Cheese> findCheese(String name) {
        return restClient
            .get()
            .uri("/cheeses/{name}", name)
            .exchange((request, response) -> {

                if (response.getStatusCode().value() == 404) {
                    return Optional.empty();  // foot cheese? never heard of it
                }

                if (response.getStatusCode().isError()) {
                    throw new CheeseApiException(
                        "Unexpected status: " + response.getStatusCode()
                    );
                }

                // All good — brie it is
                Cheese cheese = response.bodyTo(Cheese.class);
                return Optional.ofNullable(cheese);
            });
    }
```

You can also send custom **request headers** — useful when the API expects a username, tenant, or trace ID per call. Stuart wants brie:

```java
    public Cheese getCheeseForUser(String name, String username) {
        return restClient
            .get()
            .uri("/cheeses/{name}", name)
            .header("X-Cheese-User", username)   // "Stuart"
            .header("X-Request-ID", UUID.randomUUID().toString())
            .exchange((request, response) -> {

                if (response.getStatusCode().value() == 403) {
                    throw new CheeseAccessDeniedException(
                        username + " is not allowed near the brie 🚫"
                    );
                }

                if (response.getStatusCode().isError()) {
                    throw new CheeseApiException(
                        "Unexpected status: " + response.getStatusCode()
                    );
                }

                return response.bodyTo(Cheese.class);
            });
    }
```

```java
// Stuart wants brie
Cheese brie = cheeseService.getCheeseForUser("brie", "Stuart");
```

**When to use what:**

|Use `retrieve()` when…|Use `exchange()` when…|
|---|---|
|Happy path + simple error handling|Multiple possible response types|
|Throwing on 4xx/5xx is fine|You want `Optional` instead of exceptions|
|You just want the body|You need per-call request headers|
|Standard JSON deserialisation|Streaming or raw byte access|

---

## § 07 — Errors Without Throwing Exceptions

`onStatus` is great when you _want_ to throw. But sometimes you don't — you just want a null or an empty `Optional` back and let the caller decide what to do.

The trick is to skip `retrieve()` and use `exchange()` instead. With `exchange()` you own the response — no automatic exception throwing, you just check the status yourself:

```java
public Cheese getCheeseOrNull(String name) {
    return restClient
        .get()
        .uri("/cheeses/{name}", name)
        .exchange((request, response) -> {
            if (response.getStatusCode().isError()) {
                return null;   // foot cheese? cheesemonger down? either way, null
            }
            return response.bodyTo(Cheese.class);
        });
}
```

Prefer an `Optional`? Same idea:

```java
public Optional<Cheese> findCheese(String name) {
    return restClient
        .get()
        .uri("/cheeses/{name}", name)
        .exchange((request, response) -> {
            if (response.getStatusCode().isError()) {
                return Optional.empty();
            }
            return Optional.ofNullable(response.bodyTo(Cheese.class));
        });
}
```

Or just a plain try/catch if you want to keep using `retrieve()` and handle it at the call site:

```java
try {
    return restClient.get()
        .uri("/cheeses/{name}", name)
        .retrieve()
        .body(Cheese.class);
} catch (RestClientResponseException ex) {
    log.warn("Cheese '{}' not available: {}", name, ex.getStatusCode());
    return null;
}
```

> ⚠️ **Warning:** With `exchange()`, **you are responsible for consuming or closing the response body**. If you just return null without reading the body, make sure to drain or close the stream to avoid leaking connections.

---

## § 08 — Why Blocking is a Joy to Debug in IntelliJ

This is _the_ hidden superpower of synchronous REST clients.

With `RestClient`, the entire HTTP call happens on **your thread**, synchronously. Set a breakpoint anywhere in the call chain and IntelliJ stops exactly there, with a normal, readable call stack.

**Step-by-step debugging wins:**

1. **Set a breakpoint inside your `onStatus` lambda** — IntelliJ stops right there, on your thread, with the full response object in scope. Hover over `response.getStatusCode()` and see the actual value.
    
2. **Step into `.body(Cheese.class)`** — watch Jackson walk through your JSON field by field. Instantly see if a field is null because it's missing from the response or because your field name doesn't match.
    
3. **The call stack is linear** — `YourService.getCheese()` → `RestClient.retrieve()` → `HttpClient.send()`. No reactor schedulers, no operator chains, no thread-hopping. What you see is what happened, in order.
    
4. **Use IntelliJ's HTTP Client** (`.http` files) to replay the exact same request against your local app — right-click the URI string in your code and choose "Open in HTTP Client".
    

> ⚠️ **Compare: WebFlux / Reactive** With `WebClient`, a breakpoint inside a `.map()` or `.flatMap()` shows a Reactor operator stack — not your code. The actual HTTP call may have happened on a Netty I/O thread, and the response processing on yet another. You're debugging a pipeline, not a sequence.

**Rule of thumb:** Use blocking `RestClient` for everything until you actually have a throughput problem that reactive solves. Premature reactification is the root of many debugging nightmares.

---

## Quick Reference Cheat Sheet 🧀

|What you want|How|
|---|---|
|Simple GET, typed body|`.retrieve().body(Cheese.class)`|
|GET list of things|`.retrieve().body(new ParameterizedTypeReference<>() {})`|
|Handle specific status code|`.retrieve().onStatus(pred, handler)`|
|No response body (DELETE/PUT)|`.retrieve().toBodilessEntity()`|
|Full response control|`.exchange((req, res) -> ...)`|
|Return null instead of throwing|`exchange()` + check `isError()` manually|
|Return `Optional.empty()` on error|`exchange()` + return `Optional.empty()`|
|Read response headers too|`.retrieve().toEntity(Cheese.class)`|
|Add query params|`.uri("/cheeses?page={p}", page)`|
|Per-call request headers|`.header("X-User", username)` before `.exchange()`|

---

_Built with Spring Framework 6.x · Spring Boot 3.x · Jackson 2.x · 100% real cheese enthusiasm_

_🧀 No cheeses were harmed in the making of this guide 🧀_