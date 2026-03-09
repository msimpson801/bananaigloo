# Debugging WebClient in Spring Boot: The Complete Beginner's Guide

WebClient is powerful. It's also surprisingly hard to debug. When something goes wrong — and it will — you're left staring at your code wondering: did the request even go out? What URL did it actually hit? What did the server say back?

This guide walks through everything you need to answer those questions, step by step, using a real example the whole way through.

---

## The scenario

Stuart loves cheese. Your Spring Boot app calls an external cheese API on his behalf. When Stuart asks for brie, your app makes an HTTP GET request to `cheese.com/get-cheeses/brie` and gets back a cheese object.

Here's the `Cheese` class:

```java
public class Cheese {
    private String name;
    private String countryOfOrigin;

    // getters and setters
}
```

Here's the `WebClient` bean:

```java
@Configuration
public class WebClientConfig {

    @Bean
    public WebClient webClient() {
        return WebClient.builder()
            .baseUrl("https://cheese.com")
            .build();
    }
}
```

And here's the service that makes the call:

```java
@Service
public class CheeseService {

    private final WebClient webClient;

    public CheeseService(WebClient webClient) {
        this.webClient = webClient;
    }

    public Mono<Cheese> getCheese(String cheeseType) {
        return webClient.get()
            .uri("/get-cheeses/{cheeseType}", cheeseType)
            .retrieve()
            .bodyToMono(Cheese.class);
    }
}
```

This works. Stuart gets his brie. But one day Stuart stops getting his cheese and you have absolutely no idea why. The code looks fine. Nothing is obviously broken. You need to start spying on what's actually happening.

---

## The three things you need to know

When debugging a WebClient call, you're almost always trying to answer one of these three questions:

1. **What request am I sending?** — URL, HTTP method, headers
2. **What response did I get back?** — status code, response headers
3. **What was in the response body?** — the actual data that came back

Each question has its own tool. Let's go through them.

---

## Part 1: Seeing the request

### The nuclear option: turn on Netty logging

The absolute fastest way to see what's going on requires zero code changes. WebClient is built on top of a library called Reactor Netty, which manages the actual HTTP connection. You can tell Netty to log every byte it sends and receives by adding two lines to `application.properties`:

```properties
logging.level.reactor.netty.http.client=DEBUG
logging.level.io.netty.handler.logging=DEBUG
```

Restart your app and your console will show the raw HTTP traffic — method, URL, headers, body — for every request your WebClient makes.

> ⚠️ **Development only — never deploy this.** Wire-level logging prints everything in plain text, including `Authorization` headers, API keys, passwords, and any sensitive data in request or response bodies. Use it locally to get your answer, then remove both lines before you commit or deploy. It is also extremely noisy — every single byte gets logged, which makes production logs unreadable.

### The proper way: Exchange Filter Functions

A much better approach for ongoing debugging is an `ExchangeFilterFunction`. This is a piece of code you attach to your `WebClient` bean that automatically intercepts every request and every response that bean makes. You write it once and forget about it.

Think of it like a CCTV camera on a door. Everything going out gets filmed, everything coming in gets filmed, and you don't have to do anything extra each time.

You attach it to the `WebClient` builder:

```java
@Configuration
public class WebClientConfig {

    @Bean
    public WebClient webClient() {
        return WebClient.builder()
            .baseUrl("https://cheese.com")
            .filter(logRequest())
            .build();
    }

    private ExchangeFilterFunction logRequest() {
        return ExchangeFilterFunction.ofRequestProcessor(request -> {

            log.info("Request type   : {}", request.method());
            log.info("Request URL    : {}", request.url());

            request.headers().forEach((name, values) ->
                log.info("Request header : {} = {}", name, values)
            );

            return Mono.just(request);
        });
    }
}
```

Now when Stuart asks for brie, the console shows:

```
INFO  Request type   : GET
INFO  Request URL    : https://cheese.com/get-cheeses/brie
INFO  Request header : Accept = [application/json]
```

Two things worth explaining here.

**`ExchangeFilterFunction.ofRequestProcessor`** is how you tell Spring: "intercept the request before it goes out." Inside the lambda you receive a `ClientRequest` object — the full outgoing request. You can read the method, the URL, the headers, anything you like.

**`Mono.just(request)`** at the end is how you hand the request back to the pipeline. WebClient is reactive, which means everything flows through as a `Mono` — a container that holds one value. You've finished looking at the request, and now you need to give it back so the rest of the pipeline can continue. `Mono.just(request)` simply says: "here is this value, wrapped in a Mono, please carry on." You haven't changed the request at all — you've just observed it and passed it through.

---

## Part 2: Seeing the response status

Now you can see what you're sending. But what came back?

The response has its own filter function: `ExchangeFilterFunction.ofResponseProcessor`. It works exactly the same way as the request version, except it intercepts the response after it arrives, before your code does anything with it.

Add it alongside the request filter:

```java
@Bean
public WebClient webClient() {
    return WebClient.builder()
        .baseUrl("https://cheese.com")
        .filter(logRequest())
        .filter(logResponse())
        .build();
}

private ExchangeFilterFunction logResponse() {
    return ExchangeFilterFunction.ofResponseProcessor(response -> {

        log.info("Response status : {}", response.statusCode());

        return Mono.just(response);
    });
}
```

Just like with the request, `Mono.just(response)` at the end says: "I've finished looking, here is the response back, please continue."

Now the console shows both sides of the conversation:

```
INFO  Request type    : GET
INFO  Request URL     : https://cheese.com/get-cheeses/brie
INFO  Request header  : Accept = [application/json]
INFO  Response status : 200 OK
```

This alone is enormously useful. If Stuart isn't getting his cheese and you see `Response status : 401 UNAUTHORIZED`, you immediately know it's an authentication problem. If you see `Response status : 404 NOT_FOUND`, you know the URL is wrong. You haven't touched the service code at all — just attached two filters to the bean.

### The one thing filter functions can't do

Reading the response body inside a filter function is awkward. The response body is a stream of bytes — and a stream can only be read once. If the filter reads it, there's nothing left for your actual code to deserialize into a `Cheese`. You'd have to reconstruct the stream and put it back, which is messy.

For the response body, there's a better tool.

---

## Part 3: Seeing the response body

### First, let's talk about `retrieve()`

Up until now, Stuart's cheese call has used `retrieve()`:

```java
return webClient.get()
    .uri("/get-cheeses/{cheeseType}", cheeseType)
    .retrieve()
    .bodyToMono(Cheese.class);
```

`retrieve()` is the easy button. You hand control to Spring and it handles the response for you — it checks the status code, reads the body, and gives you a `Cheese`. You don't see any of it happening.

That's fine in normal operation. But when you're debugging, that invisibility is the problem.

### Swapping `retrieve()` for `exchangeToMono()`

`exchangeToMono()` is the manual version of `retrieve()`. Instead of Spring handling the response invisibly, Spring hands the raw response directly to you and says: "here you go — you deal with it."

You receive a `ClientResponse` object — the full, unprocessed HTTP response. Status code, headers, body — all there, all yours to inspect. Then, when you're done looking, you read the body yourself by calling `response.bodyToMono(Cheese.class)`.

Here's Stuart's call rewritten:

```java
public Mono<Cheese> getCheese(String cheeseType) {
    return webClient.get()
        .uri("/get-cheeses/{cheeseType}", cheeseType)
        .exchangeToMono(response -> {
            log.info("Status: {}", response.statusCode());
            return response.bodyToMono(Cheese.class);
        });
}
```

**There is one rule you must never break:** inside `exchangeToMono`, you must always call `bodyToMono` or `bodyToFlux`. Always. Even if you don't need the body. If you don't read it, Spring holds the HTTP connection open waiting, never returns it to the connection pool, and eventually your app runs out of connections entirely. Always read the body.

### What is `bodyToMono`?

`bodyToMono(Cheese.class)` tells Spring: "take the raw bytes of the response body, run them through Jackson to turn the JSON into a `Cheese` object, and give me a `Mono` that will contain that `Cheese` when it's ready."

The `Mono` part just means it's a single value arriving asynchronously — one `Cheese`, eventually.

### Now use `doOnNext` to peek at the body

Once `bodyToMono` has done its job and produced a `Cheese`, you can use `doOnNext` to look at it before it continues up the chain.

`doOnNext` means: "when a value arrives, run this code as a side effect, then let the value carry on unchanged." It's a peek, not an interception:

```java
public Mono<Cheese> getCheese(String cheeseType) {
    return webClient.get()
        .uri("/get-cheeses/{cheeseType}", cheeseType)
        .exchangeToMono(response -> {
            log.info("Status : {}", response.statusCode());
            return response.bodyToMono(Cheese.class);
        })
        .doOnNext(cheese ->
            log.info("Body   : {} from {}", cheese.getName(), cheese.getCountryOfOrigin())
        );
}
```

Console output when Stuart fetches brie:

```
INFO  Status : 200 OK
INFO  Body   : Brie from France
```

You can now see the actual deserialized Java object — not raw JSON, not bytes, but the real `Cheese` that your code is working with.

### Catching failures with `doOnError`

What about when things go completely wrong? Network timeout, server down, DNS failure? `doOnError` fires whenever an error occurs anywhere in the pipeline. Importantly, it does not swallow the error — it observes it and lets it keep propagating so your application can still handle it properly:

```java
public Mono<Cheese> getCheese(String cheeseType) {
    return webClient.get()
        .uri("/get-cheeses/{cheeseType}", cheeseType)
        .exchangeToMono(response -> {
            log.info("Status : {}", response.statusCode());
            return response.bodyToMono(Cheese.class);
        })
        .doOnNext(cheese ->
            log.info("Body   : {} from {}", cheese.getName(), cheese.getCountryOfOrigin())
        )
        .doOnError(error ->
            log.error("Failed : {}", error.getMessage())
        );
}
```

Now instead of silent failures you see things like:

```
ERROR Failed : Connection refused: cheese.com/1.2.3.4:443
```

Or if the server responded with an error status:

```
ERROR Failed : 503 Service Unavailable from GET https://cheese.com/get-cheeses/brie
```

---

## The complete setup

Here is everything working together. Two filter functions on the bean handle all requests and responses globally. `exchangeToMono`, `doOnNext`, and `doOnError` give you per-call visibility into the body and any failures.

```java
@Configuration
public class WebClientConfig {

    @Bean
    public WebClient webClient() {
        return WebClient.builder()
            .baseUrl("https://cheese.com")
            .filter(logRequest())
            .filter(logResponse())
            .build();
    }

    private ExchangeFilterFunction logRequest() {
        return ExchangeFilterFunction.ofRequestProcessor(request -> {
            log.info("➡️  {} {}", request.method(), request.url());
            request.headers().forEach((name, values) ->
                log.info("   Header : {} = {}", name, values)
            );
            return Mono.just(request);
        });
    }

    private ExchangeFilterFunction logResponse() {
        return ExchangeFilterFunction.ofResponseProcessor(response -> {
            log.info("⬅️  Status : {}", response.statusCode());
            return Mono.just(response);
        });
    }
}
```

```java
@Service
public class CheeseService {

    private final WebClient webClient;

    public CheeseService(WebClient webClient) {
        this.webClient = webClient;
    }

    public Mono<Cheese> getCheese(String cheeseType) {
        return webClient.get()
            .uri("/get-cheeses/{cheeseType}", cheeseType)
            .exchangeToMono(response -> response.bodyToMono(Cheese.class))
            .doOnNext(cheese ->
                log.info("🧀 {} from {}", cheese.getName(), cheese.getCountryOfOrigin())
            )
            .doOnError(error ->
                log.error("💥 {}", error.getMessage())
            );
    }
}
```

When Stuart fetches brie and everything is working:

```
INFO  ➡️  GET https://cheese.com/get-cheeses/brie
INFO     Header : Accept = [application/json]
INFO  ⬅️  Status : 200 OK
INFO  🧀 Brie from France
```

When the cheese API is having a bad day:

```
INFO  ➡️  GET https://cheese.com/get-cheeses/brie
INFO     Header : Accept = [application/json]
INFO  ⬅️  Status : 503 SERVICE_UNAVAILABLE
ERROR 💥 503 Service Unavailable from GET https://cheese.com/get-cheeses/brie
```

---

## Summary: which tool for which question

|Question|Tool|Where you set it up|
|---|---|---|
|What URL and method am I sending?|`ExchangeFilterFunction.ofRequestProcessor`|On the `WebClient` bean — runs for every call|
|What headers am I sending?|`ExchangeFilterFunction.ofRequestProcessor`|On the `WebClient` bean — runs for every call|
|What status code came back?|`ExchangeFilterFunction.ofResponseProcessor`|On the `WebClient` bean — runs for every call|
|What was in the response body?|`exchangeToMono` + `doOnNext`|On the individual call in your service|
|Did something blow up?|`doOnError`|On the individual call in your service|
|Everything at once, right now, locally|`logging.level.reactor.netty.http.client=DEBUG` in `application.properties`|**Remove before deploying — logs tokens and passwords**|