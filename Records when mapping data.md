# Java Records: Because Life's Too Short for Boilerplate

## The Thing We Do a Million Times (But Never Talk About)

You know what you do constantly in any real application?

**Mapping.**

Taking one thing and turning it into another thing.

- Big thing→ smaller thing
- Messy thing → tidy thing

Not glamorous, but something we do time and time again.

And for the longest time, Java made this... unnecessarily painful.

## Picture This

Imagine you’re dealing with a **large object** coming from some external API.

It has _everything_.

Stuff you need.  
Stuff you don’t need.

java

```java
public class Country {
    private String name;
    private String officialName;
    private String capital;
    private long population;
    private double area;
    private String currency;
    private String timeZone;
    private String callingCode;
    private boolean drivesOnLeft;
    private String nationalAnthem;
    
    // getters
    // setters
    // constructors
    // toString
    // equals
    // hashCode
}
```

You didn't ask for this. You didn't want this. You just need to live with it.

But here's the thing: **your app only needs three fields.**

- Name
- Capital
- Population

That's it. That's the whole list.

So what do we do?

We **map**.

But before you can map, you need somewhere to map _to_.

You already have the **chonky boy** — that massive `Country` class with all its emotional baggage.

Now you need the **slim jim** — something light, focused, and actually useful.

Something that says:

> “I only care about name, capital, and population.  
> The rest can stay in chonky boy land.”

So you create `BasicCountryInfo`.

The question is: _how_ do you create it?

## The Old Way™ (AKA: Pain)

Pre–Java 16, this was the standard move. You create a new class with just the stuff you need:

java

```java
public class BasicCountryInfo {
    private String name;
    private String capital;
    private long population;

    public BasicCountryInfo(String name, String capital, long population) {
        this.name = name;
        this.capital = capital;
        this.population = population;
    }

    public String getName() {
        return name;
    }

    public String getCapital() {
        return capital;
    }

    public long getPopulation() {
        return population;
    }

    @Override
    public String toString() {
        return "BasicCountryInfo{name='" + name + "', capital='" + capital +
               "', population=" + population + "}";
    }
    
    // Also equals() and hashCode() if you're feeling ambitious
    // (You're not)
}
```

Look at this.

**Really look at it.**

You have three pieces of data. THREE.

And yet you wrote... _*checks notes*_ ...30+ lines of code.

That's a 10:1 boilerplate-to-meaning ratio.

This is not cool.

## Lombok Swoops In Like a Superhero

Okay, so maybe you discovered Lombok:

java

```java
@Data
@AllArgsConstructor
public class BasicCountryInfo {
    private String name;
    private String capital;
    private long population;
}
```

_Beautiful._

Everyone claps.

The code is clean, the boilerplate is gone

But wait.

Before you ride off into the sunset, ask yourself one question:

## The Question Nobody Asks

**Do I actually need this to be a _class_?**

Let's be honest:

- Will these fields ever change? → **Nope**
- Am I adding behavior and methods? → **Nope**
- Is this literally just a container for data? → **Yep**

If you're just holding data to pass it around...

If you're never going to call `.setPopulation()`...

If this object exists purely to carry information from Point A to Point B...

**Then you don't need a class.**

You need a **record**.

## Enter: Java Records 🎊

Records are Java finally looking at all of us and saying:

> "I'm sorry I made you write getters for 20 years. Here, let me fix that."

Here's that same `BasicCountryInfo` as a record:

java

```java
public record BasicCountryInfo(
    String name,
    String capital,
    long population
) {}
```

That's it.

I'm not hiding code. There's no "part 2" below.

**That. Is. It.**

No getters. No constructor. No toString(). No equals(). No hashCode().

Java does it all. Automatically. For free.

## What Do You Actually Get?

When you write a record, Java generates:

1. **A constructor** (with all the fields)
2. **Accessor methods** — but they're not `getName()`, just `name()`
3. **A `toString()`** that actually makes sense
4. **`equals()` and `hashCode()`** that actually work

Here's how you use it:

java

```java
BasicCountryInfo france = 
    new BasicCountryInfo("France", "Paris", 67_000_000);

System.out.println(france.name());       // France
System.out.println(france.capital());    // Paris
System.out.println(france.population()); // 67000000
```

Notice something missing?

👉 **No setters.**

Records are **immutable**. Once you create one, it never changes.

And before you panic — that's actually _good_.

## Why Immutable Is Perfect Here

Think about what you're doing when you map data:

java

```java
public BasicCountryInfo map(Country country) {
    return new BasicCountryInfo(
        country.getName(),
        country.getCapital(),
        country.getPopulation()
    );
}
```

You're extracting information. You're creating a snapshot.

Now ask yourself:

**After you create this `BasicCountryInfo`... do you ever change it?**

Do you ever do this?

java

```java
basicInfo.setPopulation(999_999);
```

**No. Never. That would be weird.**

You created it to:

- Display in a UI
- Serialize to JSON
- Pass to another service
- Store temporarily

You don't mutate it. You just _read_ it.

**This is exactly why records exist.**

## The Real-World Use Case (Streaming FTW)

Here's the pattern you see _constantly_ in real apps:

java

```java
// Step 1: Get big messy data from somewhere
List<Country> allCountries = apiClient.fetchAllCountries();

// Step 2: Map it to something clean
List<BasicCountryInfo> basicInfoList = allCountries.stream()
    .map(country -> new BasicCountryInfo(
        country.getName(),
        country.getCapital(),
        country.getPopulation()
    ))
    .toList();

// Step 3: Use the clean data
return basicInfoList; // Send to frontend, serialize, whatever
```

This is _everywhere_ in Java:

- API responses → DTOs
- Database entities → View models
- Complex objects → Simple summaries

And in every case:

1. ✅ Create the mapped object
2. ✅ Pass it around
3. ✅ Read from it
4. ❌ Never modify it

**That's the record sweet spot.**

## When Records Are Perfect

Use records when you're:

- **Mapping data** from one shape to another
- **Returning DTOs** from APIs
- **Transferring information** between layers
- **Creating value objects** that represent data, not behavior

Records make your intent crystal clear:

> "This is data. Pure data. It doesn't change. It doesn't do things. It just _is_."

## When Records Are... Not Perfect

Records have limits. And that's okay!

### ❌ You Can't Mutate Them

This won't work:

java

```java
info.population = 1_000_000; // Compiler says no
```

Need changing state? Use a class.

### ❌ They Can't Extend Other Classes

Records secretly extend `java.lang.Record`, so:

java

```java
public record MyRecord(...) extends SomeClass {} // Illegal!
```

### ❌ They're Not Great for Complex Behavior

You _can_ add methods:

java

```java
public record BasicCountryInfo(
    String name,
    String capital,
    long population
) {
    public boolean isLargeCountry() {
        return population > 100_000_000;
    }
    
    public String summary() {
        return name + " has " + population + " people";
    }
}
```

But if you start adding _lots_ of logic, ask yourself:

> "Am I still just holding data... or am I building behavior?"

If it's the latter — use a class.

## The Cheat Sheet

**Use a RECORD when:**

- Data is immutable
- It's just carrying information
- You're mapping/transferring stuff

**Use a CLASS when:**

- State changes over time
- Behavior and logic matter
- You need inheritance

**Use LOMBOK when:**

- You're stuck with classes
- You want less boilerplate
- Records aren't available (older Java)

## Final Thought

Records aren't here to replace classes.

They're here to replace **boring** classes.

The ones where you write:

- Fields
- Constructor
- Getters
- `toString()`
- `equals()` and `hashCode()`

...and literally nothing else.

If that's what you're doing, Java is gently tapping you on the shoulder:

> "Hey... this might be a record."

And honestly? Java doesn't do that often.

So when it does?

**Listen.**