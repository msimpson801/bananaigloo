# Collectors: The "Where Should This Stuff Go?" Department 📦

Streams are great at processing data, but eventually you need to put the results somewhere. That's where Collectors come in.

Think of a collector as answering this question:

**"Cool… but what do you want me to END UP with?"**

A list? A set? A map? A map of lists? A map of maps of existential dread?

---

## Meet the Pets 🐾

Here's our data to work with:

```json
[
  { "name": "Alfred", "nickname": "Fuzzybutt", "species": "Cat", "swimmer": false, "age": 6 },
  { "name": "Michelle", "nickname": "Shelly the turtle", "species": "Turtle", "swimmer": true, "age": 3 },
  { "name": "Wilfred", "nickname": "Willy", "species": "Dog", "swimmer": true, "age": 8 },
  { "name": "Terrence", "nickname": "Terrible Terry The Tortoise", "species": "Tortoise", "swimmer": false, "age": 100 },
  { "name": "David", "nickname": "Dave the duck", "species": "Duck", "swimmer": true, "age": 4 },
  { "name": "Suzy", "nickname": "Suzy Swims", "species": "Goldfish", "swimmer": true, "age": 5 },
  { "name": "Maxamillion", "nickname": "Max", "species": "Dog", "swimmer": false, "age": 7 },
  { "name": "Barnabus", "nickname": "Barney", "species": "Dog", "swimmer": true, "age": 1 }
]
```

Quick overview:

- **8 pets total**
- **3 dogs** (Willy, Max, Barney)
- **5 swimmers** (Shelly, Willy, Dave, Suzy, Barney)
- **Age range:** 1 to 100 years (looking at you, Terry)

---

## 1️⃣ `toList()` – The "Just Gimme a List" Collector

Not shown in your code, but worth mentioning because it's the gateway drug.

java

````java
List<Pet> swimmers = pets.stream()
    .filter(Pet::isSwimmer)
    .collect(Collectors.toList());
```

**What you get:**
```
[Shelly the turtle, Willy, Dave the duck, Suzy Swims, Barney]
````

**Key points:**

- Keeps order (first in, first out)
- Allows duplicates (if you somehow have two identical pets, they're both invited)
- Makes zero promises beyond "here's a List"

---

## 2️⃣ `toSet()` – Because Duplicates Are Rude 🚫

Sometimes your list has repeats and you're like:

"I said ONE 'Dog'. Why do I have three?"

That's when you reach for a `Set`.

java

````java
Set<String> species = pets.stream()
    .map(Pet::getSpecies)
    .collect(Collectors.toSet());
```

**What you get:**
```
[Dog, Cat, Turtle, Tortoise, Duck, Goldfish]
````

Notice: Even though you have **THREE dogs** in the original list, "Dog" only appears once.

**Why a set?**

- Automatically removes duplicates 
- No guarantees on order
- Great when uniqueness matters more than anything else

**Visual:** Imagine a bouncer at a club with a clipboard. "Sorry, I already let 'Dog' in. You three other dogs? Out."

📌 **Important:** If your objects don't implement `equals()` and `hashCode()`, your "unique" set may quietly betray you. Two pets that LOOK different to you might look identical to Java.

---

## 3️⃣ `groupingBy()` – Herding Cats (and Dogs, and Ducks…) 🐕🐈🦆

This is where collectors start feeling powerful.

java

````java
Map<String, List<Pet>> petTypes = pets.stream()
    .collect(Collectors.groupingBy(Pet::getSpecies));
```

**What this does:**
- Groups pets by species
- Each key maps to a list of pets

**Result (conceptually):**
```
Dog     → [Willy, Max, Barney]
Cat     → [Fuzzybutt]
Turtle  → [Shelly the turtle]
Tortoise → [Terrible Terry The Tortoise]
Duck    → [Dave the duck]
Goldfish → [Suzy Swims]
````

**Visual:** Think of it like organizing your music library. All the rock songs go in one playlist, all the jazz in another, all the guilty pleasure 90s pop in a third (that you'll never admit exists).

**Key things to remember:**

- Keys come from your classifier function (in this case, `getSpecies()`)
- Values are lists by default (all the pets that match that species)
- Keys must be unique (maps demand it)
- Each bucket can have multiple items

This is your go-to when you think: **"I want piles of things, sorted by some rule."**

---

## 4️⃣ `partitioningBy()` – Two Buckets. No More. No Less. 🪣🪣

Partitioning is just grouping's very opinionated cousin who only sees the world in black and white.

java

````java
Map<Boolean, List<Pet>> swimmersAndNonSwimmers = pets.stream()
    .collect(Collectors.partitioningBy(Pet::isSwimmer));
```

**What you ALWAYS get:**
```
true  → [Shelly the turtle, Willy, Dave the duck, Suzy Swims, Barney]
false → [Fuzzybutt, Terrible Terry The Tortoise, Max]
````

Note: **Exactly two keys**: `true` and `false`. Every single time. No exceptions.

**The difference from groupingBy:**

|groupingBy|partitioningBy|
|---|---|
|Can create any number of groups|Always creates exactly 2 groups|
|Keys can be any type (String, Integer, etc.)|Keys are ALWAYS Boolean (true/false)|
|Empty groups don't appear|Both buckets always exist, even if empty|


**Perfect when the question is binary:**

- Swims / doesn't swim
- Good / evil
- Pineapple on pizza / no pineapple on pizza

---

## 5️⃣ `toMap()` – Maps Are Powerful and Will Hurt You If You're Careless ⚠️

java

````java
Map<String, String> nicknameLookup = pets.stream()
    .collect(Collectors.toMap(Pet::getName, Pet::getNickname));
```

**This builds a lookup table:**
```
"Alfred"      → "Fuzzybutt"
"Michelle"    → "Shelly the turtle"
"Wilfred"     → "Willy"
"Terrence"    → "Terrible Terry The Tortoise"
"David"       → "Dave the duck"
"Suzy"        → "Suzy Swims"
"Maxamillion" → "Max"
"Barnabus"    → "Barney"
````

Now when someone asks "What's Alfred's nickname?" you can look it lightning quick.

### ⚠️ VERY IMPORTANT MAP RULE:

**Keys must be unique or Java will panic.**

If two pets share the same name (say you have two pets both named "Max"), Java throws an `IllegalStateException` and storms off like a toddler.

**Safer version with a merge function:**

java

```java
Map<String, String> nicknameLookup = pets.stream()
    .collect(Collectors.toMap(
        Pet::getName,
        Pet::getNickname,
        (nickname1, nickname2) -> nickname1  // "first one wins"
    ));
```

This says: "If there's a collision, keep the first nickname and ignore the second."

**Use `toMap` when:**

- You want fast lookups ("What's this pet's nickname?")
- You are absolutely sure about key uniqueness —**or**—
- You've provided a merge strategy for conflicts

---

## 6️⃣ `minBy()` and `maxBy()` – Finding the Extremes 🏆

Who is the **youngest** pet?

java

```java
Optional<Pet> youngest = pets.stream()
    .collect(Collectors.minBy(Comparator.comparingInt(Pet::getAge)));
```

**Result:** `Optional[Pet(name=Barnabus, age=1)]` - Baby Barney wins!

Who is the **oldest** (spoiler: it's the tortoise):

java

````java
Optional<Pet> oldest = pets.stream()
    .collect(Collectors.maxBy(Comparator.comparingInt(Pet::getAge)));
````

**Result:** `Optional[Pet(name=Terrence, age=100)]` - Terrible Terry has seen things.

**Key points:**
- Result is an `Optional` (because what if the stream is empty?)
- `minBy()` finds the smallest value according to your comparator
- `maxBy()` finds the largest value
- You provide a `Comparator` to tell Java what "smallest" or "largest" means
---

## 7️⃣ `collectingAndThen()` – "Collect, Then Add a Bow on Top" 🎁

This collector is like getting a gift wrapped. First they put your item in the box (collect), then they wrap it with fancy paper (transform).

**Example:** Find the average age of all pets, then format it nicely as a message.

java

```java
String averageAgeMessage = pets.stream()
    .collect(Collectors.collectingAndThen(
        Collectors.averagingInt(Pet::getAge),
        avg -> String.format("The average pet age is %.1f years", avg)
    ));
```

**What happens:**

1. **First:** `averagingInt()` calculates the average age → `16.75` (a raw number)
2. **Then:** The finishing function wraps it in a nice sentence → `"The average pet age is 16.8 years"`

**Result:** A formatted string ready to display, not just a boring decimal number.

**Another example:** Get all the unique species and count them in one go:

java

```java
String report = pets.stream()
    .map(Pet::getSpecies)
    .collect(Collectors.collectingAndThen(
        Collectors.toSet(),
        set -> "We have " + set.size() + " different types of pets: " + set
    ));
```

**Result:** `"We have 6 different types of pets: [Dog, Cat, Turtle, Tortoise, Duck, Goldfish]"`

**Why use this?**

- You want to collect something AND immediately do something extra with the result
- Saves you from doing two separate operations
- Perfect for "get me the data, then format/count/wrap it"

**Visual:** It's like ordering a sandwich. They make the sandwich (collect the ingredients), then they cut it in half and wrap it in paper (the finishing touch). You get the final product, ready to eat, all in one transaction.

---

## Quick Reference Card 🃏

|Collector|What You Get|When To Use|
|---|---|---|
|`toList()`|Ordered list with duplicates|"Give me everything"|
|`toSet()`|Unordered unique values|"No repeats, please"|
|`groupingBy()`|Map with multiple groups|"Sort into categories"|
|`partitioningBy()`|Map with exactly 2 groups (true/false)|"Binary choice"|
|`toMap()`|Key-value lookup table|"Fast lookups by key"|
|`minBy()` / `maxBy()`|Optional of smallest/largest|"Find the extreme"|
|`collectingAndThen()`|Collected + transformed|"Collect, then polish it up"|
