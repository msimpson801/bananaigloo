# 🍊 Jackson & ObjectMapper in Spring Boot: A Beginner's Guide

> _You write Java. Your users get JSON. Your app calls APIs that return JSON. How does all this magic happen? Meet Jackson — Spring Boot's behind-the-scenes wizard._

---

## 🎩 The Magic Behind Your REST Endpoints

Let's say you have a simple controller:

```java
@RestController
@RequestMapping("/fruits")
public class FruitController {

    @GetMapping
    public List<Fruit> getFruits() {
        return List.of(
            new Fruit("Orange", "citrus", true),
            new Fruit("Lemon", "citrus", false),
            new Fruit("Grapefruit", "citrus", true),
            new Fruit("Mango", "tropical", true),
            new Fruit("Pineapple", "tropical", false)
        );
    }
}
```

Your `Fruit` class is a plain Java object:

```java
public class Fruit {
    private String name;
    private String category;
    private boolean inSeason;

    // constructor, getters, setters...
}
```

You hit `GET /fruits` in your browser or Postman and... you get this:

```json
[
  { "name": "Orange", "category": "citrus", "inSeason": true },
  { "name": "Lemon", "category": "citrus", "inSeason": false },
  { "name": "Grapefruit", "category": "citrus", "inSeason": true },
  { "name": "Mango", "category": "tropical", "inSeason": true },
  { "name": "Pineapple", "category": "tropical", "inSeason": false }
]
```

You didn't write any JSON serialisation code. So what happened? 🤔

**Jackson happened.** Spring Boot automatically includes Jackson and configures it to convert your Java objects to JSON (and back again) whenever you use `@RestController`. This process is called **serialisation** (Java → JSON) and **deserialisation** (JSON → Java).

---

## 🥦 The Reverse: Calling External APIs

Now flip it around. You're using `WebClient` to fetch vegetables from an external API:

```java
@Service
public class VegetableService {

    private final WebClient webClient = WebClient.create("https://api.veggieland.com");

    public List<Vegetable> fetchVegetables() {
        return webClient.get()
            .uri("/vegetables")
            .retrieve()
            .bodyToFlux(Vegetable.class)  // 🪄 JSON → Java
            .collectList()
            .block();
    }
}
```

When you hit that same endpoint in Postman, you see raw JSON:

```json
[
  { "name": "Broccoli", "colour": "green", "calories": 55 },
  { "name": "Carrot", "colour": "orange", "calories": 41 }
]
```

But in your Spring Boot app, `fetchVegetables()` returns a `List<Vegetable>` — real Java objects you can loop over, filter, and pass around. Jackson silently read the JSON and mapped each field to the matching field in your `Vegetable` class.

**Same magic, opposite direction.** ✨

---

## 🏷️ Useful Jackson Annotations

This is where things get really fun. Jackson gives you a toolkit of annotations to control exactly how your objects are serialised and deserialised.

### `@JsonProperty` — Rename an Awkward Field

Imagine you're consuming an external vegetable API, and the response looks like this:

```json
{ "veg_nm": "Carrot", "cal_per_100g": 41 }
```

Yikes. You don't want `vegNm` and `calPer100g` littering your Java code. Use `@JsonProperty` to map those ugly names to something sensible:

```java
public class Vegetable {

    @JsonProperty("veg_nm")
    private String name;

    @JsonProperty("cal_per_100g")
    private int caloriesPer100g;
}
```

Now your Java code uses clean `name` and `caloriesPer100g`, but Jackson knows to look for `veg_nm` and `cal_per_100g` when reading the JSON. 🎉

---

### `@JsonIgnore` — Keep Secrets Secret 🤫

Sometimes you have fields you simply don't want to expose. Maybe it's a password, a secret token, or some supremely embarrassing internal data your frontend has absolutely no business knowing about:

```java
public class User {

    private String username;
    private String email;

    @JsonIgnore
    private String superTopSecretPassword;  // 🙈 absolutely not going anywhere near the frontend

    @JsonIgnore
    private String mothersMaidenName;  // why do we even store this, honestly

    @JsonIgnore
    private int numberOfTimesTheyveCalledSupportThisWeek;  // they don't need to know we know
}
```

When this object is serialised to JSON, the shameful fields are completely omitted — the frontend only sees what you want it to see:

```json
{ "username": "alice", "email": "alice@example.com" }
```

Blissful ignorance. 😇

---

### `@JsonIgnoreProperties` — Ignore Fields You Don't Need

When consuming an external API, the response might have 30 fields but you only care about 3. The World Countries API probably returns everything — national anthem lyrics, timezone history, the entire Wikipedia article — but you only need `name` and `population`. Rather than cluttering your class, ignore the extras at the class level:

```java
@JsonIgnoreProperties(ignoreUnknown = true)
public class Country {
    private String name;
    private long population;
    // all those other 47 fields from the API? silently binned 🗑️
}
```

This is also a great safety net — if the external API adds new fields in the future, your app won't blow up.

---

### `@JsonAlias` — Accept Multiple Names for One Field

Sometimes an API is inconsistent, or you're merging data from multiple sources that name things slightly differently. `@JsonAlias` lets one Java field accept several possible JSON names:

```java
public class Fruit {

    @JsonAlias({"fruit_name", "fruitName", "title"})
    private String name;
}
```

Jackson will map any of those JSON keys to `name`. During serialisation, it will still use `name` (or whatever `@JsonProperty` says).

---

### `@JsonInclude` — Only Include Non-Null Fields

By default, Jackson includes fields even when they're `null`. This can bloat your JSON. Use `@JsonInclude` to skip nulls:

```java
@JsonInclude(JsonInclude.Include.NON_NULL)
public class Fruit {
    private String name;
    private String description;  // null? not included in the JSON output
    private Boolean inSeason;
}
```

---

### `@JsonFormat` — Control Date & Number Formatting

Dates are a notorious pain. `@JsonFormat` lets you dictate exactly how they appear:

```java
public class Landmark {

    @JsonFormat(shape = JsonFormat.Shape.STRING, pattern = "dd-MM-yyyy")
    private LocalDate dateOpened;
}
```

Instead of a cryptic timestamp number, you'll get `"dateOpened": "31-03-1889"`. Much friendlier — even Gustave Eiffel would approve.

---

### `@JsonSerialize` / `@JsonDeserialize` — Go Fully Custom

For truly bespoke logic, you can write your own serialiser or deserialiser:

```java
public class Fruit {

    @JsonSerialize(using = UpperCaseSerializer.class)
    private String name;  // always serialised as uppercase
}
```

This is an advanced move — most of the time the other annotations cover your needs.

---

## 🧪 Tests with Groovy, Spock & ObjectMapper

Here's where ObjectMapper becomes your best friend in tests.

Imagine you're testing a service that works with a `Country` object. Sounds simple enough — but a country has a `Capital` city, which has a list of `Landmark` objects, each with a `GeoCoordinate`, opening hours, an `Architect` with their own biography, and a list of `VisitorReview` objects each containing a `Reviewer` with their travel history...

You get the idea. Building all that out in Groovy is the stuff of nightmares:

```groovy
// 😩 This is what despair looks like
def architect = new Architect(name: "Gustave Eiffel", nationality: "French", 
                               born: 1832, famousQuote: "I should be jealous of the tower")
def landmark = new Landmark(name: "Eiffel Tower", yearBuilt: 1889,
                             architect: architect,
                             coordinates: new GeoCoordinate(lat: 48.8584, lon: 2.2945),
                             reviews: [
                                new VisitorReview(rating: 5, comment: "ooh la la",
                                                  reviewer: new Reviewer(name: "Bob", ...)),
                                // send help
                             ])
def capital = new Capital(name: "Paris", population: 2161000, landmarks: [landmark])
def country = new Country(name: "France", capital: capital, ...)
// I've forgotten what I was even testing
```

### The Easy Way: JSON Files + ObjectMapper

Instead, write your test data as JSON — it's much more natural for nested structures — stick it in a file, and let Jackson do the heavy lifting.

**Step 1:** Create `src/test/resources/fixtures/france.json`:

```json
{
  "name": "France",
  "continent": "Europe",
  "population": 68000000,
  "currency": "Euro",
  "officialLanguage": "French",
  "capital": {
    "name": "Paris",
    "population": 2161000,
    "nicknames": ["City of Light", "City of Love", "That place with the tower"],
    "landmarks": [
      {
        "name": "Eiffel Tower",
        "yearBuilt": 1889,
        "heightMetres": 330,
        "visitorsPer Year": 7000000,
        "coordinates": {
          "lat": 48.8584,
          "lon": 2.2945
        },
        "architect": {
          "name": "Gustave Eiffel",
          "nationality": "French",
          "born": 1832,
          "otherFamousWorks": ["Statue of Liberty internal structure", "Bordeaux bridge"]
        },
        "reviews": [
          {
            "rating": 5,
            "comment": "Worth every step of those 1,665 stairs. My legs disagree.",
            "reviewer": { "name": "Bob", "homecountry": "UK", "tripsThisYear": 4 }
          },
          {
            "rating": 3,
            "comment": "Very tall. Confirmed.",
            "reviewer": { "name": "Jan", "homeCountry": "Netherlands", "tripsThisYear": 12 }
          }
        ]
      },
      {
        "name": "The Louvre",
        "yearBuilt": 1793,
        "heightMetres": 21,
        "visitorsPerYear": 9000000,
        "coordinates": {
          "lat": 48.8606,
          "lon": 2.3376
        },
        "architect": {
          "name": "Pierre Lescot",
          "nationality": "French",
          "born": 1515,
          "otherFamousWorks": ["Various things that are also in France"]
        },
        "reviews": [
          {
            "rating": 4,
            "comment": "Saw the Mona Lisa. She is smaller than expected. Still cool.",
            "reviewer": { "name": "Alice", "homeCountry": "USA", "tripsThisYear": 2 }
          }
        ]
      }
    ]
  }
}
```

**Step 2:** Load it in your Spock test using `ObjectMapper`:

```groovy
import com.fasterxml.jackson.databind.ObjectMapper
import spock.lang.Specification

class CountryServiceSpec extends Specification {

    ObjectMapper objectMapper = new ObjectMapper()

    def "should calculate total landmark visitors for a country's capital"() {
        given: "France, loaded from a JSON fixture like a civilised person"
        def json = getClass().getResourceAsStream("/fixtures/france.json")
        def france = objectMapper.readValue(json, Country)

        when:
        def totalVisitors = countryService.totalCapitalLandmarkVisitors(france)

        then:
        totalVisitors == 16_000_000  // 7M Eiffel Tower + 9M Louvre
    }

    def "should find the oldest landmark in the capital"() {
        given:
        def json = getClass().getResourceAsStream("/fixtures/france.json")
        def france = objectMapper.readValue(json, Country)

        when:
        def oldest = countryService.oldestCapitalLandmark(france)

        then:
        oldest.name == "The Louvre"
        oldest.yearBuilt == 1793
    }
}
```

Clean, readable, and your test data lives in one easy-to-edit place. Want to test with a different country? Just add `germany.json`. Want an edge case with a capital that has no landmarks? Add `sad-country.json`. 🙌

---

## ⚙️ ObjectMapper Configuration Options

When you need more control over how Jackson behaves, you can configure the `ObjectMapper` directly. In Spring Boot, the cleanest way is to define a bean:

```java
@Configuration
public class JacksonConfig {

    @Bean
    public ObjectMapper objectMapper() {
        return new ObjectMapper()
            .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false)
            .configure(DeserializationFeature.FAIL_ON_NULL_FOR_PRIMITIVES, false)
            .configure(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, false)
            .setSerializationInclusion(JsonInclude.Include.NON_NULL)
            .registerModule(new JavaTimeModule());
    }
}
```

Here's what each option does:

|Configuration|What it does|
|---|---|
|`FAIL_ON_UNKNOWN_PROPERTIES, false`|Don't crash if the JSON has fields your class doesn't know about|
|`FAIL_ON_NULL_FOR_PRIMITIVES, false`|Don't crash if a `null` JSON value maps to an `int` or `boolean`|
|`WRITE_DATES_AS_TIMESTAMPS, false`|Serialise dates as readable strings, not epoch numbers|
|`NON_NULL` serialisation|Omit null fields from all JSON output|
|`JavaTimeModule`|Add support for `LocalDate`, `LocalDateTime`, `ZonedDateTime` etc.|

### In Tests

You can configure ObjectMapper inline for specific test needs:

```groovy
ObjectMapper objectMapper = new ObjectMapper()
    .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false)
    .registerModule(new JavaTimeModule())
```

This is especially handy when your fixture JSON was generated from a richer model than your test class — you don't want the test failing because of a field you don't care about.

---

## 🗺️ Quick Reference

|Annotation|Use it when...|
|---|---|
|`@JsonProperty("field_name")`|JSON field name differs from your Java field name|
|`@JsonAlias({"a", "b"})`|Multiple possible JSON keys should map to one field|
|`@JsonIgnore`|You never want this field in the JSON (serialise or deserialise)|
|`@JsonIgnoreProperties(ignoreUnknown = true)`|Silently ignore unexpected fields from an API|
|`@JsonInclude(NON_NULL)`|Omit null fields from serialised output|
|`@JsonFormat(pattern = "dd-MM-yyyy")`|Control date/number formatting|

---

## 🎓 Summary

- **Spring Boot + `@RestController`**: Jackson automatically serialises your Java objects to JSON responses and deserialises incoming JSON into Java objects.
- **WebClient**: Same story in reverse — Jackson maps the API's JSON response body into your Java types.
- **Annotations**: Give you fine-grained control over naming, visibility, formatting, and nulls without writing a single line of manual parsing code.
- **Tests**: Store complex test data as JSON files and use `objectMapper.readValue()` to load them into Java objects — far cleaner than building deeply nested objects by hand.
- **Configuration**: Tune Jackson globally via a `@Bean` or locally per `ObjectMapper` instance to handle edge cases gracefully.

Jackson is one of those libraries that does so much heavy lifting you barely notice it's there — until the day you need it to do something specific, and you discover just how powerful it really is. 🚀