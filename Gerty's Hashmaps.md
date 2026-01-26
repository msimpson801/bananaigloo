# How HashMaps Work: The Neighborhood Metaphor

HashMaps store key-value pairs and let you look them up incredibly fast. Instead of searching through items one by one, they use **hashing** to jump straight to what you need.

Here's how.

---

## The Search Problem

Imagine a neighborhood where everyone lives in a house with a unique address.

You're looking for your old pal Gertrude because you need her nickname for your autobiography.

**The problem:** You don't know where she lives. You only know she's somewhere in this neighborhood.

So you start knocking:

🚶 House #1: "Are you Gertrude?"  
"Nope, I'm Cuthbert."

🚶 House #2: "Are you Gertrude?"  
"No, I'm Balthazar."

🚶 House #3: "Are you Gertrude?"  
"Absolutely not. I'm Shirley."

🚶 House #4: "Are you Gertrude?"  
"Nah mate, I'm Tallulah."

If there are:

- 10 houses → annoying
- 1,000 houses → exhausting
- 1,000,000 houses → you die of old age before finding her

This is how **lists** work. They check every item, one at a time.

In computer science terms, this is **O(n) time complexity**—the more items you have, the longer it takes to find anything.

---

## The Hash Solution

Now imagine you have a magic address book.

You say **"Gertrude"** and it instantly tells you: **"House #90210."**

You walk straight there. Knock once. Done.

No wandering. No small talk with Balthazar. No existential crisis when Tallulah tries to sell you essential oils.

This is what a **HashMap** does.

### How It Works

When you store data in a HashMap, it runs your key through a **hash function** that converts it into a number:

```
"Gertrude" → hash function → 90210
```

That number becomes the "address" where the value gets stored.

Here's what it looks like in Java:

java

```java
HashMap<String, String> nicknames = new HashMap<>();

// Store data: Java hashes each name to get an address
nicknames.put("Gertrude", "G-Unit");
nicknames.put("Balthazar", "B-zar");
nicknames.put("Tallulah", "Loo-Lah");
nicknames.put("Shirley", "Shironimo");
nicknames.put("Cuthbert", "Berty big balls");

// Look up data: Java hashes "Gertrude" and goes straight to the address
String nickname = nicknames.get("Gertrude");  // Returns "G-Unit"
```

When you call `get("Gertrude")`, Java:

1. Hashes "Gertrude" → gets 90210
2. Goes directly to address 90210
3. Returns "G-Unit"

This is **O(1) time complexity**—it takes the same amount of time whether you have 10 items or 10 million items.

**That's the power of a HashMap.**

You don't write the hash function yourself. Every object in Java has a built-in `hashCode()` method that Java uses automatically.

---

## The Collision Exception

Here's the catch: hashing isn't perfect.

Sometimes two different keys produce the same hash. This is called a **hash collision**.

### The Housemates Problem

You walk up to house #90210, expecting to find just Gertrude.

But when you knock, Prudence answers the door.

Turns out Gertrude and Prudence are **housemates**—they both hashed to the same address.

So you ask: "Are you Gertrude?"

Prudence says no. So you check the only other person in the house—and there's Gertrude.

Even with this extra step, it's **still way faster** than knocking on every house in the neighborhood.

### How Java Handles It

When a collision happens, Java:

1. Goes to the correct address (the hash)
2. Checks which key at that address is the right one

Java handles this automatically behind the scenes. In most cases, you don't need to worry about collisions at all.

---

## Why This Matters

**HashMaps:**

- O(1) lookup time on average
- Fast with 10 items
- Fast with 10 million items
- Speed doesn't grow as data grows

**Lists:**

- O(n) lookup time
- Get slower and slower as data grows
- Have to check every item

That's why HashMaps are the go-to choice when you need fast lookup