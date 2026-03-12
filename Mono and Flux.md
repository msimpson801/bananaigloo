# A Beginner's Guide to Reactive Programming, Mono, Flux and Spring WebClient

_No prior experience needed — just a fondness for cheese and coffee._

---

## Part 1: Start With Something Familiar

Before anything reactive, let's start with something concrete and simple.

You have a `Cheese` object. It exists right now, in memory. You can call methods on it immediately:

```java
Cheese cheese = new Cheese("Stilton", "England", 9);

cheese.getName();            // "Stilton"
cheese.getCountryOfOrigin(); // "England"
cheese.getPungency();        // 9
```

You can also have a `List<Cheese>` — multiple cheeses, all present and accounted for. You can loop through them, filter them, sort them:

```java
List<Cheese> cheeses = List.of(
    new Cheese("Brie", "France", 3),
    new Cheese("Stilton", "England", 9),
    new Cheese("Gouda", "Netherlands", 5)
);

cheeses.stream()
    .filter(c -> c.getPungency() > 4)
    .forEach(System.out::println);
```

These are **concrete, present things**. They exist right now. You can use them right now. Simple.

Hold onto this feeling — we're about to contrast it with something different.

---

## Part 2: What Is a Mono?

A `Mono<Cheese>` is not a cheese.

It is a **promise of a cheese**. A cheese that doesn't exist yet, but will at some point in the future. When it arrives, you've already written down what to do with it.

Think of your local cheesemonger. Normally they hand you the cheese directly — instant, done. But imagine a new kind of cheesemonger. This one says:

> _"I don't have your Stilton right now. It's coming in on Thursday's delivery. Leave me your number and I'll call you when it arrives."_

You don't have a cheese. But you have a **promise** of one. And you can write down instructions: _"When the cheese arrives, slice it, put it on a board, and serve it with crackers."_

That's a `Mono<Cheese>`. One item. In the future. With instructions attached.

```java
Mono<Cheese> futureCheese = cheeseShop.orderStilton();

futureCheese
    .doOnNext(cheese -> slice(cheese))
    .doOnNext(cheese -> serveWithCrackers(cheese))
    .subscribe();
```

**Key insight:** when you write `.doOnNext(cheese -> slice(cheese))`, you are not slicing anything right now. You are writing a note to your future self: _"when the cheese shows up, do this."_

Nothing happens until the cheese arrives.

---

## Part 3: What Is a Flux?

If `Mono` is a promise of **one** thing, `Flux` is a promise of **many** things — a stream of items arriving over time.

Same cheesemonger, different scenario:

> _"I've got a whole delivery of cheeses coming in. I'll call you with each one as they arrive off the lorry."_

```java
Flux<Cheese> cheeseDelivery = cheeseShop.getDelivery();

cheeseDelivery
    .filter(cheese -> cheese.getPungency() > 5)
    .doOnNext(cheese -> System.out.println("Strong one: " + cheese.getName()))
    .subscribe();
```

Again — nothing is happening yet when you write this. You're writing the recipe. The items will flow through your instructions one by one as they arrive.

||Concrete|Reactive|
|---|---|---|
|One item|`Cheese`|`Mono<Cheese>`|
|Many items|`List<Cheese>`|`Flux<Cheese>`|
|Available|Right now|In the future|
|Your code|Acts on it immediately|Describes what to do when it arrives|

---

## Part 4: Instructions for the Future, Not Right Now

This is the most important thing to understand, so it's worth dwelling on.

When you write a chain like this:

```java
Mono<Cheese> result = cheeseShop.orderStilton()
    .map(cheese -> cheese.getName().toUpperCase())
    .doOnNext(name -> System.out.println("Got cheese: " + name))
    .onErrorReturn("CHEDDAR");
```

You are not doing any of those things yet. The cheese hasn't arrived. You are writing a **to-do list for the future**.

Reading it naturally: _"When a cheese arrives, uppercase its name. Then print it. If anything goes wrong at any point, use CHEDDAR instead."_

That whole chain just sits there, waiting. It springs into life when something **subscribes** — which is the trigger that says _"ok, go, start the whole process."_

```java
result.subscribe(); // NOW it actually runs
```

In Spring WebFlux, the framework subscribes for you automatically — which is why you often won't see an explicit `.subscribe()` call in WebClient code. Spring pulls the trigger behind the scenes.

---

## Part 5: Why Does Any of This Exist? Meet Ed.

To understand _why_ reactive programming exists, we need to talk about Ed.

Ed is a CEO's personal assistant. Capable, reliable, and very busy. Every morning the CEO needs several things done:

- ☕ Fetch the morning coffee from the café downstairs
- 📋 Brief the 2pm meeting attendees
- 📅 Prepare the afternoon agenda
- 🍽️ Book a restaurant for the evening client dinner

Ed's **old way** of working — the blocking way:

```
1. Go downstairs to café
2. Order coffee
3. Stand at the counter and wait...
4. Wait...
5. Wait... (coffee takes 3 minutes)
6. Collect coffee, go back upstairs
7. Brief the meeting attendees
8. Prepare the agenda
9. Book the restaurant
```

Three minutes of Ed's time, completely wasted. Just standing there. And the CEO's briefings and agenda are delayed because of it.

In code, a **blocking** approach looks like this:

```java
Coffee coffee = cafe.orderCoffee();  // thread stops here and waits
ed.deliverToCEO(coffee);
ed.briefMeetingAttendees();
ed.prepareAgenda();
ed.bookRestaurant();
```

That `cafe.orderCoffee()` call freezes everything. Nothing else happens until it returns.

---

## Part 6: Ed Gets a Buzzer

Now the café gives Ed a buzzer. He orders the coffee, pockets the buzzer, and heads back upstairs. While he's waiting for it to go off, he's fully productive:

```
1. Go downstairs to café
2. Order coffee, get buzzer
3. Head back upstairs
4. Brief the meeting attendees      ← getting useful things done
5. Prepare the agenda               ← still going
6. Book the restaurant              ← nearly there
7. *BUZZ* — coffee's ready!
8. Nip back down, collect coffee
9. Deliver to CEO
```

Everything gets done. Nothing was wasted. The coffee still took 3 minutes — but Ed didn't stand there for 3 minutes.

The reactive version in code:

```java
Mono<Coffee> futureCoffee = cafe.orderCoffeeAsync();  // returns immediately, no waiting

futureCoffee
    .doOnNext(coffee -> ed.deliverToCEO(coffee))  // runs when coffee arrives
    .subscribe();

// These run straight away, without waiting for coffee:
ed.briefMeetingAttendees();
ed.prepareAgenda();
ed.bookRestaurant();
```

The logs now look like:

```
[09:00] Briefing meeting attendees...
[09:01] Preparing agenda...
[09:01] Booking restaurant...
[09:03] ☕ Coffee arrived! Delivering to CEO.
```

---

## Part 7: An Important Clarification

You might be thinking: _"So reactive programming means my code skips ahead and does other things while waiting?"_

Not quite — and this is worth getting clear on.

**Within your own chain of logic, everything still happens in order.** If you have business logic that depends on the cheese, it will still wait for the cheese. The sequence of your own code is unchanged.

What actually happens is subtler and happens **one level down**, at the thread level.

In a normal blocking web server, when your code makes an HTTP call and waits, the **thread** is frozen. Just sitting there. Doing nothing. In a busy server handling thousands of requests, you quickly run out of available threads — they're all frozen, all waiting.

In a reactive web server, when your code makes an HTTP call, the thread is **released back into a pool**. It goes off and handles a completely different request from a completely different user. When your response eventually comes back, a thread picks it up and continues where you left off.

**The win isn't for your individual request** — it still takes the same amount of time end-to-end. **The win is that your server can handle far more requests at once**, with far fewer threads.

Going back to Ed: he still does everything for the CEO in the right order. The coffee still gets delivered before the next coffee-dependent task. It's just that while waiting, he's free to help someone else — he's not standing frozen at a counter.

---

## Part 8: Spring WebClient

Now we get to the point. Spring's `WebClient` is built entirely on reactive programming. When it makes an HTTP call to another service, it doesn't block a thread waiting for the response — it works exactly like Ed's buzzer.

This is why when you look at WebClient code, you'll see `Mono` and `Flux` everywhere. It's not to make your life difficult — it's because every HTTP call is inherently a _future value_. You've made the request; the response hasn't arrived yet.

Here's a simple WebClient call:

```java
WebClient client = WebClient.create("https://api.example.com");

Mono<Cheese> cheeseMono = client.get()
    .uri("/cheese/stilton")
    .retrieve()
    .bodyToMono(Cheese.class);
```

Reading `.bodyToMono(Cheese.class)` naturally: _"When the HTTP response body arrives, deserialise it into a `Cheese` object and give it to me as a Mono."_

You're not getting a `Cheese` back. You're getting a `Mono<Cheese>` — a promise of one, with instructions attached.

If you're expecting a list of cheeses:

```java
Flux<Cheese> cheeses = client.get()
    .uri("/cheeses")
    .retrieve()
    .bodyToFlux(Cheese.class);
```

---

## Part 9: Handling Errors the Reactive Way

Here's something that catches everyone out at first.

In normal Java, you throw exceptions like this:

```java
throw new RuntimeException("Something went wrong!");
```

But in reactive code, **errors have to travel through the pipeline too**. You can't just throw a regular exception out of a `Mono` — it has to be wrapped up and sent down the same channel as everything else.

This is why you see `Mono.error()` in WebClient code:

```java
Mono<Cheese> cheeseMono = client.get()
    .uri("/cheese/stilton")
    .retrieve()
    .onStatus(
        status -> status.is5xxServerError(),
        response -> Mono.error(new CheeseShopException("The shop is on fire 🔥"))
        //  ✅ Mono.error() — travels through the pipeline
        //  ❌ throw new CheeseShopException() — this won't work properly here
    )
    .bodyToMono(Cheese.class);
```

Why? Think about it with the cheesemonger analogy. If the cheesemonger discovers the cheese is bad — that's a problem that happens **in the future**, at the moment of delivery. They don't phone your mum to complain. They call **you**, through the same channel you agreed on. The error travels through the same pipeline as the success.

`Mono.error()` is that call. It signals _"something went wrong at this point in the future"_, and any `.onErrorReturn()` or `.onErrorMap()` you've set up downstream will catch it.

---

## Part 10: A Full WebClient Example, Annotated

Let's put it all together. Here's a realistic WebClient method with everything explained:

```java
public Mono<Cheese> getCheese(String name) {
    return webClient.get()
        .uri("/cheese/" + name)

        // Trigger the request and inspect what comes back
        .retrieve()

        // If it's a 4xx, we probably asked for something wrong
        .onStatus(HttpStatusCode::is4xxClientError, response ->
            Mono.error(new CheeseNotFoundException("No cheese found: " + name))
        )

        // If it's a 5xx, the cheese shop itself is broken
        .onStatus(HttpStatusCode::is5xxServerError, response ->
            Mono.error(new CheeseShopDownException("Try again later"))
        )

        // When the body arrives, turn it into a Cheese object
        .bodyToMono(Cheese.class)

        // If anything above went wrong, log it
        .doOnError(e -> log.error("Failed to get cheese '{}': {}", name, e.getMessage()))

        // And return a safe fallback rather than propagating the error
        .onErrorReturn(new Cheese("Cheddar", "England", 5));
}
```

Reading this top to bottom as a plain sentence: _"Make a GET request for this cheese. If I get a 4xx back, signal a not-found error. If I get a 5xx, signal a shop-down error. Otherwise, when the body arrives, turn it into a Cheese. If anything at all went wrong, log it and return Cheddar as a safe fallback."_

None of this runs when the method is called. It returns a `Mono<Cheese>` — a set of instructions. It all runs when something subscribes, which Spring does automatically when this is returned from a controller.

---

## Part 11: Reading WebClient Code in the Wild

Now that you understand the mental model, here's how to approach WebClient code you haven't written yourself.

**Step 1: Find the `.bodyToMono()` or `.bodyToFlux()` call.** That tells you what type of object the HTTP response is being turned into.

**Step 2: Read the chain as a to-do list.** Each method in the chain is one instruction that will run when the response arrives — in order, top to bottom.

**Step 3: When you see `Mono.error()`...** Something went wrong in the future (a bad status code, a failed parse) and the error is being sent down the pipeline. Look for `.onError*` methods downstream to see how it's handled.

**Step 4: Remember nothing has happened yet.** The method returning a `Mono` is just handing you a recipe. The actual HTTP call hasn't been made. Spring makes it when it needs to.

---

## Quick Reference

|Term|What it means|
|---|---|
|`Mono<T>`|A promise of 0 or 1 item of type `T`, arriving in the future|
|`Flux<T>`|A promise of 0 to many items of type `T`, arriving over time|
|`.bodyToMono(Foo.class)`|"When the response body arrives, turn it into a `Foo`"|
|`.bodyToFlux(Foo.class)`|"When the response arrives, turn each item into a `Foo`"|
|`.map(x -> y)`|Transform the value when it arrives|
|`.doOnNext(x -> ...)`|Do something when a value arrives, without transforming it|
|`Mono.error(exception)`|Signal that something went wrong, send the error down the pipeline|
|`.onStatus(...)`|Inspect the HTTP status code and react to it|
|`.doOnError(e -> ...)`|Do something when an error arrives (e.g. logging)|
|`.onErrorReturn(fallback)`|If anything went wrong, use this value instead|
|`.subscribe()`|Pull the trigger — start the whole thing running|

---

## The Mental Model to Keep

Whenever you see `Mono` or `Flux` — whether in WebClient, in a service method, or in a controller — just say to yourself:

> _"This isn't a value. This is a set of instructions for when the value arrives."_

The cheese isn't here yet. But the cheesemonger has your number, the buzzer is in your pocket, and you've already written down exactly what you're going to do when it arrives. 🧀