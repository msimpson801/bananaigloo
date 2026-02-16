# A Beginner's Guide to Advanced Collectors & Downstream Collectors 🍄

## What Are Collectors Anyway?

Java has these things called **collectors** which we can use to gather up the results from a stream and put them into a useful format. Think of them as different ways to organize your data at the end of a stream pipeline.

You're probably familiar with some of these collectors:

java

```java
List<String> names = characters.stream()
    .map(MarioCharacter::getName)
    .collect(Collectors.toList());  // Collect into a list

Set<String> colors = characters.stream()
    .map(MarioCharacter::getColour)
    .collect(Collectors.toSet());   // Collect into a set
```

So far so good? Great! Now let's look at some more powerful collectors that can do really cool things with our Mario characters.

---

## Two Types of Collectors 🎯

Here's the key thing to understand: **collectors fall into two categories:**

### **Standalone Collectors** (Can be used on their own)

These work directly with `.collect()`:

- `groupingBy()` - organize into groups
- `partitioningBy()` - split into true/false
- `teeing()` - apply two collectors at once
- `summarizingInt()` - get all statistics

### **Downstream Collectors** (Work inside other collectors)

These are like helper collectors that work _inside_ standalone collectors:

- `mapping()` - transform elements in each group
- `filtering()` - filter elements in each group
- `flatMapping()` - flatten collections in each group
- `collectingAndThen()` - transform the final result

**Some collectors can be both!** Like `reducing()`, `counting()`, `summingInt()`, `toList()`, `toSet()`.

Let's see them in action!

---

## Part 1: Standalone Collectors

These work directly in your `.collect()` call.

### 1. `groupingBy()` - Organize Into Buckets 📦

Group characters by their home world:

java

```java
Map<String, List<MarioCharacter>> byHomeWorld = characters.stream()
    .collect(Collectors.groupingBy(MarioCharacter::getHomeWorld));

System.out.println(byHomeWorld.get("Mushroom Kingdom"));
// Output: [Mario, Luigi, Peach, Goomba, Toad]
```

**What we got:** A Map where keys are home worlds and values are lists of characters.

You can also group by calculated values:

java

```java
Map<String, List<MarioCharacter>> byWealthLevel = characters.stream()
    .collect(Collectors.groupingBy(c -> 
        c.getNoOfCoins() > 100 ? "Rich" : "Poor"
    ));
```

---

### 2. `partitioningBy()` - Heroes vs Villains 🦸‍♂️🦹‍♂️

Split into exactly two groups based on true/false:

java

```java
Map<Boolean, List<MarioCharacter>> goodVsBad = characters.stream()
    .collect(Collectors.partitioningBy(MarioCharacter::isBadGuys));

List<MarioCharacter> heroes = goodVsBad.get(false);
List<MarioCharacter> villains = goodVsBad.get(true);

System.out.println("Heroes: " + heroes.size());    // 5
System.out.println("Villains: " + villains.size()); // 3
```

**Why use this over `groupingBy()`?** You're guaranteed to get both keys (true and false), even if one group is empty!

---

### 3. `teeing()` - Two Collectors, One Stream 🐦🐦

Calculate two different things from the same stream:

java

```java
record GameStats(int totalCoins, double avgLives) {}

GameStats stats = characters.stream()
    .collect(Collectors.teeing(
        Collectors.summingInt(MarioCharacter::getNoOfCoins),     // First collector
        Collectors.averagingInt(MarioCharacter::getNoOfLives),   // Second collector
        GameStats::new                                           // How to combine
    ));

System.out.println("Total coins: " + stats.totalCoins());  // 990
System.out.println("Average lives: " + stats.avgLives());   // 3.75
```

**Super efficient!** One pass through the stream, two results combined.

---

### 4. `summarizingInt()` - All Stats At Once 📊

Get count, sum, average, min, and max in one go:

java

```java
IntSummaryStatistics coinStats = characters.stream()
    .collect(Collectors.summarizingInt(MarioCharacter::getNoOfCoins));

System.out.println("Total: " + coinStats.getSum());        // 990
System.out.println("Average: " + coinStats.getAverage());  // 123.75
System.out.println("Max: " + coinStats.getMax());          // 500
System.out.println("Min: " + coinStats.getMin());          // 0
System.out.println("Count: " + coinStats.getCount());      // 8
```

One collector, five statistics! Also available: `summarizingLong()` and `summarizingDouble()`.

---

## Part 2: Downstream Collectors

These **must** be used inside other collectors - they're the "second argument" helpers!

### 5. `mapping()` - Transform While Collecting 🎨

Group by world, but only keep the names (not full objects):

java

```java
// ❌ This won't work on its own:
// characters.stream().collect(Collectors.mapping(...))

// ✅ Use it as a downstream collector:
Map<String, List<String>> namesByWorld = characters.stream()
    .collect(Collectors.groupingBy(
        MarioCharacter::getHomeWorld,        // Primary: group by this
        Collectors.mapping(                  // Downstream: transform each item
            MarioCharacter::getName,
            Collectors.toList()
        )
    ));

System.out.println(namesByWorld.get("Mushroom Kingdom"));
// Output: [Mario, Luigi, Peach, Goomba, Toad]
```

**What happened?**

1. Group characters by home world
2. For each character in a group, extract just the name
3. Collect those names into a list

---

### 6. `filtering()` - Filter Within Groups 🧹

Group by world, but only include rich characters (50+ coins):

java

```java
Map<String, List<MarioCharacter>> richByWorld = characters.stream()
    .collect(Collectors.groupingBy(
        MarioCharacter::getHomeWorld,        // Primary: group by world
        Collectors.filtering(                // Downstream: filter within each group
            c -> c.getNoOfCoins() >= 50,
            Collectors.toList()
        )
    ));
```

**Why not just filter before grouping?**

java

```java
// Filtering BEFORE - some worlds might disappear completely
Map<String, List<MarioCharacter>> option1 = characters.stream()
    .filter(c -> c.getNoOfCoins() >= 50)
    .collect(Collectors.groupingBy(MarioCharacter::getHomeWorld));

// Filtering AFTER - all worlds appear, some with empty lists
Map<String, List<MarioCharacter>> option2 = characters.stream()
    .collect(Collectors.groupingBy(
        MarioCharacter::getHomeWorld,
        Collectors.filtering(c -> c.getNoOfCoins() >= 50, Collectors.toList())
    ));
```

With downstream `filtering()`, you keep all groups in your map!

---

### 7. `flatMapping()` - Unpack Nested Collections 🎯

When each element has a collection inside it, flatten them out:

java

```java
// Imagine each character has getAbilities() returning List<String>
Map<String, List<String>> abilitiesByColor = characters.stream()
    .collect(Collectors.groupingBy(
        MarioCharacter::getColour,           // Primary: group by color
        Collectors.flatMapping(              // Downstream: flatten abilities
            c -> c.getAbilities().stream(),  // Get each character's abilities
            Collectors.toList()              // Collect into flat list
        )
    ));
```

**Without flatMapping:** You'd get `Map<String, List<List<String>>>` (lists of lists!)  
**With flatMapping:** You get `Map<String, List<String>>` (nice flat lists!)

---

### 8. `collectingAndThen()` - Transform the Final Result ✨

Collect something, then do one final transformation:

java

```java
// Find the richest character's name
String richest = characters.stream()
    .collect(Collectors.collectingAndThen(
        Collectors.maxBy(Comparator.comparing(MarioCharacter::getNoOfCoins)),  // Collect
        opt -> opt.map(MarioCharacter::getName).orElse("Nobody")               // Transform
    ));

System.out.println(richest); // "Bowser"
```

**Used as downstream collector:**

java

```java
// Group by world, find richest character name in each world
Map<String, String> richestPerWorld = characters.stream()
    .collect(Collectors.groupingBy(
        MarioCharacter::getHomeWorld,
        Collectors.collectingAndThen(
            Collectors.maxBy(Comparator.comparing(MarioCharacter::getNoOfCoins)),
            opt -> opt.map(MarioCharacter::getName).orElse("None")
        )
    ));
```

---

## Part 3: Flexible Collectors (Work Both Ways!)

Some collectors can be used standalone OR as downstream collectors.

### 9. `reducing()` - Custom Calculations 🔧

**Standalone - total wealth across all characters:**

java

```java
int totalWealth = characters.stream()
    .collect(Collectors.reducing(
        0,                                        // Starting value
        c -> c.getNoOfLives() * c.getNoOfCoins(), // Transform each
        Integer::sum                              // Combine them
    ));

System.out.println("Total wealth: " + totalWealth);
```

**Downstream - total wealth per home world:**

java

```java
Map<String, Integer> wealthByWorld = characters.stream()
    .collect(Collectors.groupingBy(
        MarioCharacter::getHomeWorld,
        Collectors.reducing(
            0,
            c -> c.getNoOfLives() * c.getNoOfCoins(),
            Integer::sum
        )
    ));
```

Perfect for custom aggregations!

---

## Part 4: Combining Collectors - The Real Magic! 🪄

You can nest downstream collectors to create powerful queries:

### Example 1: Filter, then Transform

Group by world, keep only rich characters, extract their names:

java

```java
Map<String, List<String>> richNamesByWorld = characters.stream()
    .collect(Collectors.groupingBy(
        MarioCharacter::getHomeWorld,                    // Group by world
        Collectors.filtering(                            // Filter within groups
            c -> c.getNoOfCoins() >= 50,
            Collectors.mapping(                          // Transform filtered results
                MarioCharacter::getName,
                Collectors.toList()                      // Collect names
            )
        )
    ));
```

**Reading tip:** Work from outside to inside:

1. Group by home world
2. Within each group, filter for 50+ coins
3. From those, extract the name
4. Collect names into a list

---

### Example 2: Group, Partition, Count

Group by world, then partition each world into heroes/villains with counts:

java

```java
Map<String, Map<Boolean, Long>> worldHeroVillainCounts = characters.stream()
    .collect(Collectors.groupingBy(
        MarioCharacter::getHomeWorld,
        Collectors.partitioningBy(
            MarioCharacter::isBadGuys,
            Collectors.counting()
        )
    ));

System.out.println(worldHeroVillainCounts.get("Mushroom Kingdom"));
// Output: {false=4, true=1}  (4 heroes, 1 villain)
```

---

### Example 3: Multiple Downstream Operations

Group by color, get average coins AND count:

java

```java
record ColorStats(double avgCoins, long count) {}

Map<String, ColorStats> statsByColor = characters.stream()
    .collect(Collectors.groupingBy(
        MarioCharacter::getColour,
        Collectors.teeing(
            Collectors.averagingInt(MarioCharacter::getNoOfCoins),
            Collectors.counting(),
            ColorStats::new
        )
    ));
```

---

## Quick Reference Table 📝

|Collector|Standalone?|Downstream?|What It Does|
|---|---|---|---|
|`groupingBy()`|✅|❌|Group elements into a Map|
|`partitioningBy()`|✅|❌|Split into true/false groups|
|`teeing()`|✅|❌|Apply two collectors at once|
|`summarizingInt()`|✅|✅|Get all statistics|
|`mapping()`|❌|✅|Transform elements in groups|
|`filtering()`|❌|✅|Filter within groups|
|`flatMapping()`|❌|✅|Flatten nested collections|
|`collectingAndThen()`|✅|✅|Transform final result|
|`reducing()`|✅|✅|Custom aggregation|
|`counting()`|✅|✅|Count elements|
|`summingInt()`|✅|✅|Sum values|
|`toList()`, `toSet()`|✅|✅|Basic collections|

---

## The Common Patterns 🎨

**Pattern 1: Group and Transform**

java

```java
Collectors.groupingBy(
    classifier,                    // What to group by
    Collectors.mapping(...)        // How to transform each group
)
```

**Pattern 2: Group and Filter**

java

```java
Collectors.groupingBy(
    classifier,
    Collectors.filtering(...)      // Keep only some items per group
)
```

**Pattern 3: Group and Aggregate**

java

```java
Collectors.groupingBy(
    classifier,
    Collectors.counting()          // Or summingInt(), averagingInt(), etc.
)
```

**Pattern 4: Group, Filter, Transform**

java

```java
Collectors.groupingBy(
    classifier,
    Collectors.filtering(
        predicate,
        Collectors.mapping(...)    // Nest as deep as needed!
    )
)
```