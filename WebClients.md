# Beginner's Guide to Web Clients in Spring Boot

When you build an API, sometimes it needs to talk to _other_ APIs. Maybe you need to fetch data from a weather API, or call a microservice inside your own system. That's where Web Clients come in – they enable your API to make HTTP requests to these external services.

## Getting Started with WebClient

### Add the WebClient Dependency

Before you can use WebClient, make sure your project includes the Spring WebFlux dependency. This gives you access to `WebClient`.

**For Gradle** (add this to your `build.gradle`):

gradle

```gradle
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-webflux'
}
```

---
### Creating a WebClient

Use the `WebClient.builder()` method to create a new instance of WebClient. From there, you can configure it with various options like base URL, default headers, and more.

Here's an example:

java

```java
WebClient webClient = WebClient.builder()
        .baseUrl("https://lovelycheeseAPI.com")
        .defaultHeader("Accept", "application/json")
        .build();
```

**What each part does:**

- **`baseUrl()`** - Sets the starting part of all your API calls. You won't need to type the full URL every time
- **`defaultHeader()`** - Adds headers that will be included in every request (like telling the API you want JSON responses)
- **`build()`** - Finishes creating your WebClient so you can use it

---
## Making Your First GET Request

Now that you have a WebClient, you can use it to fetch data. Here's how to get a single cheese:

java

```java
Cheese cheddar = webClient.get()
        .uri("/api/cheeses/cheddar")
        .retrieve()
        .bodyToMono(Cheese.class)
        .block();
```

**Breaking it down step by step:**

- **`get()`** - You're making a GET request (fetching data, not sending it)
- **`uri("/api/cheeses/cheddar")`** - The path to the specific resource. Since you already set the base URL, this gets added to it
- **`retrieve()`** - Execute the request and get the response back from the API
- **`bodyToMono(Cheese.class)`** - Take the JSON response and convert it into a `Cheese` object
- **`block()`** - Stop and wait right here until the response arrives
---
## Understanding Mono vs Flux

When calling APIs with WebClient, you need to handle responses differently depending on whether you're getting back one object or multiple objects.

### Getting a Single Object with `bodyToMono()`

When an API returns just one object, use `bodyToMono()`:

java

```java
Cheese cheese = webClient.get()
        .uri("/api/cheeses/cheddar")
        .retrieve()
        .bodyToMono(Cheese.class)
        .block();
```

Think of `Mono` as a container that will eventually hold one item (or be empty if something goes wrong).

### Getting Multiple Objects with `bodyToFlux()`

When an API returns a list or collection of objects, use `bodyToFlux()`:

java

```java
webClient.get()
        .uri("/api/cheeses")
        .retrieve()
        .bodyToFlux(Cheese.class)
        .doOnNext(cheese -> System.out.println(cheese.getName()))
        .blockLast();
```

**What's happening here:**

- **`bodyToFlux(Cheese.class)`** - Convert the response into a `Flux`, which is a stream that can emit multiple objects
- **`doOnNext()`** - Do something with each cheese as it arrives (like printing its name)
- **`blockLast()`** - Wait for all items to arrive

Think of `Flux` as a stream or pipeline where objects flow through one by one.

### Collecting Items into a List

Often you just want to get all the items as a regular Java `List`. Here's how:

java

```java
List<Cheese> allCheeses = webClient.get()
        .uri("/api/cheeses")
        .retrieve()
        .bodyToFlux(Cheese.class)
        .collectList()
        .block();
```

The `collectList()` method gathers all the items from the `Flux` into a single `List`, which you can then use normally in your code.

---
## Managing Multiple WebClients

When building a larger application, you'll often need to communicate with multiple different APIs. For example, you might need to call a cheese API, a wine API, and a cracker API all from the same application. The best way to manage multiple WebClients is to create a configuration class where you define each one as a bean.

### Creating a Configuration Class

Use `@Configuration` to create a class that sets up all your WebClients in one place:

java

```java
@Configuration
public class WebClientConfig {

    @Bean
    public WebClient cheeseWebClient() {
        return WebClient.builder()
                .baseUrl("https://api.cheese.com")
                .defaultHeader("Accept", "application/json")
                .build();
    }

    @Bean
    public WebClient wineWebClient() {
        return WebClient.builder()
                .baseUrl("https://api.wine.com")
                .defaultHeader("Accept", "application/json")
                .build();
    }
}
```

**Why do this?** By defining each WebClient as a `@Bean`, Spring manages them for you. You can inject them wherever needed, and all your API configurations are in one easy-to-find place.

### Injecting WebClients into Services

Once you've created your configuration class, you can inject specific WebClients into your services using constructor injection:

java

```java
@Service
public class CheeseService {

    private final WebClient cheeseWebClient;

    public CheeseService(WebClient cheeseWebClient) {
        this.cheeseWebClient = cheeseWebClient;
    }

    public Cheese getCheese(String id) {
        return cheeseWebClient.get()
                .uri("/cheeses/{id}", id)
                .retrieve()
                .bodyToMono(Cheese.class)
                .block();
    }
}
```

Spring automatically knows which WebClient to inject based on the bean name.

Now that we know how to create and configure WebClients, and how to make GET requests, let's look at how we might do a POST request and how it differs from a simple GET request.

---
## Making our first POST request 

When making a POST request, we're **sending information to an API** to create or update data. Unlike GET requests, which retrieve data, POST requests include a payload of data that we want to send to the server. To do this, we need to specify the request body, which contains the data we want to send. In WebClient, we can use the `bodyValue()` method to specify the request body.

Here's an example of a POST request:

Java

```java
Cheese cheese = new Cheese("Cheddar", "Strong");

webClient.post()
        .uri("/api/cheeses")
        .contentType(MediaType.APPLICATION_JSON)
        .bodyValue(cheese)
        .retrieve()
        .bodyToMono(String.class)
        .block();
```


In this example, we're creating a new `Cheese` object and passing it to the `bodyValue()` method. We're also specifying the `Content-Type` header as `application/json`, which tells the server that the request body contains JSON data.

The main differences between a POST request and a GET request are:

- We use the `post()` method instead of `get()`

- We specify the request body using the `bodyValue()` method

- We typically specify the `Content-Type` header to indicate the format of the request body

---
## Handling Errors in Spring WebClient (Simple Guide)

When you call an API using `WebClient`, things can go wrong:

- The client sends a bad request (4xx)

- The server fails (5xx)

- Something else breaks (network, timeout, etc.)

WebClient gives you **three common ways** to deal with these problems.

![1️⃣](https://static.xx.fbcdn.net/images/emoji.php/v9/t7a/1/16/31_20e3.png) `onStatus` – Handle HTTP status codes (4xx / 5xx)

Use this **when you care about the HTTP response code**.

- Best for: turning HTTP errors into custom exceptions

- Runs **before** the body is read

Java

```java
webClient.get()
    .uri("/api/cheeses")
    .retrieve()
    .onStatus(HttpStatus::is4xxClientError, response ->
        Mono.error(new RuntimeException("Client error"))
    )
    .onStatus(HttpStatus::is5xxServerError, response ->
        Mono.error(new RuntimeException("Server error"))
    )
    .bodyToMono(String.class)
    .block();
```

**Think of it as:**  
“If the server responds with a bad status, fail in a controlled way.”

![2️⃣](https://static.xx.fbcdn.net/images/emoji.php/v9/t99/1/16/32_20e3.png) `doOnError` – Log or react to an error (no recovery)

Use this **when you just want to observe the error**, not fix it.

- Best for: logging, metrics, debugging

- Does **not** stop or recover from the error

Java

```java
webClient.get()
    .uri("/api/cheeses")
    .retrieve()
    .bodyToMono(String.class)
    .doOnError(ex ->
        System.out.println("Error occurred: " + ex.getMessage())
    )
    .block();
```

**Think of it as:**  
“If something fails, tell me—but still fail.”

![3️⃣](https://static.xx.fbcdn.net/images/emoji.php/v9/tb8/1/16/33_20e3.png) `onErrorResume` – Recover with a fallback

Use this **when you want the app to keep working** even if the call fails.

- Best for: default values, backup behavior

- Replaces the error with another `Mono`

Java

```java
webClient.get()
    .uri("/api/cheeses")
    .retrieve()
    .bodyToMono(String.class)
    .onErrorResume(ex -> Mono.just("Fallback value"))
    .block();
```

**Think of it as:**  
“If it fails, return something safe instead."


