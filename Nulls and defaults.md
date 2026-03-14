# Nulls and defaults in Spring

> A friendly guide to sensible defaults in Java — from field initialisation all the way to Jackson deserialization

---

Let's start with three people. They all have a `Person` object, and one of the fields is a list of their favourite things.

|Person|Favourite things|
|---|---|
|Mark|travelling, spicy food|
|Edward|collecting coins, Formula One|
|Simon|`null` — Simon is a bit grumpy|

Mark and Edward are easy to work with. Simon? Not so much. If you try to iterate over Simon's favourite things without checking for null first, you'll get a `NullPointerException` faster than you can say "null check".

Let's fix that — and cover every place in a Spring Boot app where null can sneak back in.

---

## Part 1: defaults when using `new`

Here's a basic `Person` class with some Lombok annotations to save us writing boilerplate constructors ourselves.

```java
@NoArgsConstructor
@AllArgsConstructor
public class Person {
    private String name;
    private String hometown;
    private List<String> favouriteThings;
}
```

`@NoArgsConstructor` generates a constructor with no arguments — `new Person()`. `@AllArgsConstructor` generates one that takes every field — `new Person("Simon", "Leeds", someList)`. Lombok writes both so we don't have to.

`name` and `hometown` being null is probably fine — null means "we don't know yet," which is genuinely useful information. But a null list is almost never useful. You can iterate over an empty list; you cannot iterate over null.

The fix is to set a default value right on the field:

```java
@NoArgsConstructor
@AllArgsConstructor
public class Person {
    private String name;
    private String hometown;
    private List<String> favouriteThings = new ArrayList<>();
}
```

Now when Simon is created with `new Person()`, the no-args constructor runs and the field initialiser fires — Simon gets an empty list:

```java
Person simon = new Person();
System.out.println(simon.getFavouriteThings()); // []
```

No crash, no drama, no Simon-related incidents. ✅

> **Why does this work?** Field initialisers run as part of the constructor. When you call `new Person()`, Java runs the no-args constructor, which includes setting `favouriteThings = new ArrayList<>()`. The default is baked in before you even touch the object.

You can do this for any field — not just lists. You could default a status to `"ACTIVE"`, give everyone a starting score of zero, whatever makes sense.

---

## Part 2: adding a Lombok builder — and breaking everything

A lot of Spring Boot codebases use Lombok's `@Builder`. It's great — when your object has many fields, passing them all into a constructor gets messy fast. A builder lets you name each one clearly:

```java
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Person {
    private String name;
    private String hometown;
    private List<String> favouriteThings = new ArrayList<>();
}
```

Looks fine, right? Let's try it:

```java
// Using new — still works fine
Person mark = new Person();
System.out.println(mark.getFavouriteThings()); // []

// Using the builder
Person simon = Person.builder()
    .name("Simon")
    .build();
System.out.println(simon.getFavouriteThings()); // null 😬
```

Simon's list is **null again**. The builder bypasses the constructor entirely — it sets each field directly, and any field you don't explicitly set just stays as whatever Java's default is. For objects, that's null.

Your field initialiser (`= new ArrayList<>()`) is completely ignored when going through the builder.

The fix is one extra annotation:

```java
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Person {
    private String name;
    private String hometown;

    @Builder.Default
    private List<String> favouriteThings = new ArrayList<>();
}
```

Now let's try again:

```java
// Using new
Person mark = new Person();
System.out.println(mark.getFavouriteThings()); // []

// Using the builder
Person simon = Person.builder()
    .name("Simon")
    .build();
System.out.println(simon.getFavouriteThings()); // []
```

`@Builder.Default` tells Lombok to wire the initialiser into the builder's generated code so it runs even when you don't set the field. Both paths now give you an empty list. ✅

---

## Part 3: data from an external API

In Spring Boot you often receive data from external APIs. `RestClient` handles the HTTP call, and under the hood it uses **Jackson** to turn the JSON response into Java objects — a process called deserialization.

Jackson doesn't use your constructor or builder. It creates the object its own way and sets fields directly. Which means all your defaults? Potentially ignored again.

There are two distinct scenarios to understand:

### Scenario A — key missing entirely

```json
{ "name": "Simon" }
```

No `favouriteThings` key at all. Jackson never touches that field — your field initialiser already ran, so Simon gets an empty list. You're fine. ✅

### Scenario B — key present, value null

```json
{ "name": "Simon", "favouriteThings": null }
```

Jackson sees the key, reads null, and actively sets `favouriteThings = null` — **overwriting your default**. This is the one to watch out for. ❌

---

For Scenario B, you need to tell Jackson what to do when it encounters a null for that field. There are two options worth knowing:

**`Nulls.SKIP`** — "If the JSON value is null, ignore it completely and leave whatever the field already was."

- Relies on your field initialiser having set a value. The null is silently discarded.

**`Nulls.AS_EMPTY`** — "If the JSON value is null, convert it to the empty version of this type."

- For a `List` you get `[]`. For a `String`, `""`. Doesn't rely on any initialiser.

For lists, both give you the same result — an empty list. But `AS_EMPTY` is arguably more explicit: you're saying "make this empty" rather than "pretend you didn't see that." It also doesn't depend on remembering to write the field initialiser, which makes it a bit more robust.

Applied to just the field we care about:

```java
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Person {
    private String name;
    private String hometown;  // null is meaningful here, leave it alone

    @Builder.Default
    @JsonSetter(nulls = Nulls.AS_EMPTY)
    private List<String> favouriteThings = new ArrayList<>();
}
```

If the API sends `"hometown": null`, Jackson sets it to null — which is exactly right. Null means "we don't have this information." Only the list gets the special treatment.

---

## Part 4: applying the Jackson config globally

If you have many classes with list fields, annotating every single one gets tedious. You can configure Jackson once, globally, so that all `List` fields across your whole application get `AS_EMPTY` behaviour automatically.

The right way to do this in Spring Boot is with a `Jackson2ObjectMapperBuilderCustomizer` bean. Create a configuration class — something like `config/JacksonConfig.java`:

```java
@Configuration
public class JacksonConfig {

    @Bean
    public Jackson2ObjectMapperBuilderCustomizer jacksonCustomizer() {
        return builder -> builder.postConfigurer(mapper ->
            mapper.configOverride(List.class)
                  .setSetterInfo(JsonSetter.Value.forValueNulls(Nulls.AS_EMPTY))
        );
    }
}
```

This plugs into Spring Boot's existing Jackson setup rather than replacing it. The `postConfigurer` callback runs after Spring Boot has already configured the `ObjectMapper` with its own sensible defaults — things like date formatting and module registration — so you're adding to it, not clobbering it.

Spring Boot then shares this `ObjectMapper` across the whole application, including `RestClient`. You don't need to wire anything into `RestClient` manually — it picks it up automatically.

> ⚠️ **Why not just declare a raw `@Bean ObjectMapper`?**
> 
> You can, but it replaces Spring Boot's autoconfigured one entirely. That means you'd need to manually re-add date formatting, the Kotlin module if you're using it, `JavaTimeModule`, and anything else Spring Boot was setting up for you. The customizer approach is additive — it's almost always the better choice.

### Why scope it to `List.class`?

This is the important bit. The global config above only targets fields of type `List`. That's deliberate.

If you applied `AS_EMPTY` to all types globally, `"hometown": null` would become `""`. Now you can't tell the difference between "we don't have a hometown" and "the person has an empty hometown." You've lost meaningful data.

Scoped to `List.class` only, `"hometown": null` stays null and `"favouriteThings": null` becomes `[]`. Each type behaves correctly for what it represents.

Collections are the sweet spot for this kind of global default because a null list is almost never intentional — it virtually always means "empty" rather than "unknown." Strings, custom objects, and other types are far more likely to carry meaning when null, so leave those to field-level annotations.

---

## The full picture

|Technique|Works with `new`?|Works with builder?|JSON key missing?|JSON key is null?|
|---|---|---|---|---|
|Field initialiser only|✅|❌|✅|❌|
|`@Builder.Default`|✅|✅|✅|❌|
|`@JsonSetter SKIP`|—|—|✅|✅ keeps initialiser value|
|`@JsonSetter AS_EMPTY`|—|—|✅|✅ actively sets empty|
|Global config (`List.class`)|—|—|✅|✅ all lists, no annotations needed|

A useful rule of thumb to end on: things like `hometown` or `middleName` are genuinely nullable — null means "we don't know," and that's valuable. But collections should almost never be null. An empty list is always more useful than null, and the techniques above make sure it stays that way regardless of how your object was created.

Simon remains grumpy. But at least he won't crash your app. 🎉