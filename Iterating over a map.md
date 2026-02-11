# 🎤 The Fun Guide to Iterating Over Maps in Java

_Learn Java Maps with Hip-Hop Legends!_

---

## 🗺️ What is a Map Anyway?

A **Map** stores data as **key-value pairs**. Each unique key maps to exactly one value.

Think of it like this: if you know a rapper's stage name (the key), you can look up their real name (the value).

In Java, the most commonly used implementation is `HashMap`.

---

## 🎯 Setting Up Our Map

Let's create a map of legendary rappers and their real names:

java

```java
import java.util.HashMap;
import java.util.Map;

Map<String, String> rappers = new HashMap<>();
rappers.put("Eminem", "Marshall Mathers");
rappers.put("Snoop Dogg", "Calvin Broadus");
rappers.put("Jay-Z", "Shawn Carter");
rappers.put("Dr. Dre", "Andre Young");
rappers.put("Ice Cube", "O'Shea Jackson");
```

### Visual Representation:

```
┌─────────────┐          ┌─────────────────┐
│   Eminem    │  ──────→ │ Marshall Mathers│
└─────────────┘          └─────────────────┘

┌─────────────┐          ┌─────────────────┐
│ Snoop Dogg  │  ──────→ │ Calvin Broadus  │
└─────────────┘          └─────────────────┘

┌─────────────┐          ┌─────────────────┐
│    Jay-Z    │  ──────→ │  Shawn Carter   │
└─────────────┘          └─────────────────┘

┌─────────────┐          ┌─────────────────┐
│   Dr. Dre   │  ──────→ │  Andre Young    │
└─────────────┘          └─────────────────┘

┌─────────────┐          ┌─────────────────┐
│  Ice Cube   │  ──────→ │ O'Shea Jackson  │
└─────────────┘          └─────────────────┘
```

---
# 🔁 Ways to Iterate Over a Map

There are multiple ways to iterate over a `Map` in Java. Let’s break them down from basic to advanced.

---
## 🔑 Method 1: Getting Just the Keys

### 🎫 Using `keySet()`

Want to see all the stage names? Use `keySet()` to get just the keys!

java

```java
for (String stageName : rappers.keySet()) {
    System.out.println(stageName);
}
```

**📤 Output:**

```
Eminem
Snoop Dogg
Jay-Z
Dr. Dre
Ice Cube
```

> ⚠️ **Remember:** This only gives you the keys. You don't get the values unless you do an extra lookup!

---

## 🔍 Method 2: Keys + Lookups

### 🔎 Using `keySet()` with `get()`

You can iterate over keys and then look up each value using `get()`:

java

```java
for (String stageName : rappers.keySet()) {
    String realName = rappers.get(stageName);
    System.out.println(stageName + " is really " + realName);
}
```

**📤 Output:**

```
Eminem is really Marshall Mathers
Snoop Dogg is really Calvin Broadus
Jay-Z is really Shawn Carter
Dr. Dre is really Andre Young
Ice Cube is really O'Shea Jackson
```

**✅ Pros:**

- Simple to understand
- Easy for beginners

**❌ Cons:**

- Less efficient - does two operations per entry (get key, then lookup value)
- More code than necessary

👉 Prefer `entrySet()` when you need both key and value.

---

## 💎 Method 3: Getting Just the Values

### 📝 Using `values()`

Only care about the real names? Use `values()`:

java

```java
for (String realName : rappers.values()) {
    System.out.println(realName);
}
```

**📤 Output:**

```
Marshall Mathers
Calvin Broadus
Shawn Carter
Andre Young
O'Shea Jackson
```

> ⚠️ **Remember:** This gives you only the values. You lose access to the keys entirely!

---

## ⭐ Method 4: The Best Way - entrySet()

### 🏆 Using `entrySet()` - The Champion Method!

This is the most common and efficient way to iterate when you need **both keys and values**.

java

```java
for (Map.Entry<String, String> entry : rappers.entrySet()) {
    String stageName = entry.getKey();
    String realName = entry.getValue();
    System.out.println(stageName + " = " + realName);
}
```

**📤 Output:**

```
Eminem = Marshall Mathers
Snoop Dogg = Calvin Broadus
Jay-Z = Shawn Carter
Dr. Dre = Andre Young
Ice Cube = O'Shea Jackson
```

> 💡 **Pro Tip:** Notice the `=` sign! When you print a `Map.Entry` object directly, Java automatically formats it as `key=value`. This is built into the Entry class!

### You can also just print the entry directly:

java

```java
for (Map.Entry<String, String> entry : rappers.entrySet()) {
    System.out.println(entry);  // Automatically shows key=value
}
```

**Same output!**

### 🎯 Why is this better?

Using `entrySet()` is more efficient because:

1. You get the key and value together in one operation
2. No extra lookup needed (unlike using `keySet()` with `get()`)
3. Cleaner, more readable code
4. Better performance, especially for large maps

---

## 🚀 Method 5: Modern Java - forEach()

### ✨ Using `forEach()` with Lambda (Java 8+)

The cleanest, most modern syntax! Perfect for simple operations:

java

```java
rappers.forEach((stageName, realName) -> {
    System.out.println(stageName + " = " + realName);
});
```

**📤 Output:**

```
Eminem = Marshall Mathers
Snoop Dogg = Calvin Broadus
Jay-Z = Shawn Carter
Dr. Dre = Andre Young
Ice Cube = O'Shea Jackson
```

Even shorter for simple operations:

java

```java
rappers.forEach((stage, real) -> 
    System.out.println(stage + " = " + real));
```

> 💡 **Pro Tip:** This is the preferred method in modern Java! It's concise, readable, and efficient.

---

## 🌊 Method 6: Streams (Advanced)

### 🎨 Using Streams for Powerful Operations

Streams let you filter, transform, and process map data in elegant ways!

#### Example 1: Filter rappers whose real names start with 'C'

java

```java
rappers.entrySet().stream()
    .filter(entry -> entry.getValue().startsWith("C"))
    .forEach(entry -> 
        System.out.println(entry.getKey() + " = " + entry.getValue()));
```

**📤 Output:**

```
Snoop Dogg = Calvin Broadus
```

#### Example 2: Convert all stage names to uppercase

java

```java
rappers.entrySet().stream()
    .forEach(entry -> 
        System.out.println(entry.getKey().toUpperCase() + 
                          " = " + entry.getValue()));
```

**📤 Output:**

```
EMINEM = Marshall Mathers
SNOOP DOGG = Calvin Broadus
JAY-Z = Shawn Carter
DR. DRE = Andre Young
ICE CUBE = O'Shea Jackson
```

---

## 📊 Quick Reference Table

|Method|What You Get|Best Use Case|Efficiency|
|---|---|---|---|
|`keySet()`|Just the keys|When you only need keys|⭐⭐⭐|
|`keySet() + get()`|Keys and values (separately)|Quick prototypes|⭐⭐|
|`values()`|Just the values|When you only need values|⭐⭐⭐|
|`entrySet()`|Key-value pairs together|When you need both (most common)|⭐⭐⭐⭐⭐|
|`forEach()`|Key-value pairs|Modern, clean syntax|⭐⭐⭐⭐⭐|
|`stream()`|Transformed/filtered data|Complex operations|⭐⭐⭐⭐|

---

## 🎓 Key Takeaways

> 💡 **Pro Tip:** When you need both keys and values, use `entrySet()` or `forEach()` instead of `keySet()` with `get()` for better performance!

> 💡 **Pro Tip:** The `=` format you see when printing entries (`Eminem=Marshall Mathers`) is automatic! It's how Java's `Map.Entry` class represents itself as a string.

> 💡 **Pro Tip:** Use streams when you need to filter, transform, or perform complex operations on your map data. They're powerful but might be overkill for simple iterations.

> ⚠️ **Remember:** Maps don't guarantee order (unless you use `LinkedHashMap` or `TreeMap`). Don't rely on items appearing in the order you added them!
