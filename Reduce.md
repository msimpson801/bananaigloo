# Java `reduce()` — A Beginner's Guide 🧮

So you've heard about `Stream.reduce()` and wondered what the fuss is about. Don't worry — by the end of this guide it'll click, and you'll actually _enjoy_ using it.

---

## 🤔 What Even Is `reduce()`?

Imagine you have a list of numbers and want to add them all together. One way is a plain old loop:

```java
int[] numbers = {1, 2, 3, 4, 5};
int total = 0;
for (int n : numbers) {
    total = total + n;
}
// total = 15
```

`reduce()` does the exact same thing, just in a more expressive, stream-friendly way. It **reduces** a whole list of values down to a **single result**.

---

## 🧱 The Building Blocks

Before looking at code, let's get the vocabulary straight. `reduce()` works with three concepts:

|Term|What it means|
|---|---|
|**Starting value**|Where we begin (e.g. `0` for addition)|
|**Accumulated value**|The running result so far|
|**Current element**|The item from the list we're processing right now|

---

## 👀 Visualising What reduce() Does

Let's add `[1, 2, 3, 4, 5]` together with a starting value of `0`.

```
Start:   acc = 0

Step 1:  acc(0)  + current(1)  → acc = 1
Step 2:  acc(1)  + current(2)  → acc = 3
Step 3:  acc(3)  + current(3)  → acc = 6
Step 4:  acc(6)  + current(4)  → acc = 10
Step 5:  acc(10) + current(5)  → acc = 15

Result: 15 ✅
```

Each step takes whatever we've built up so far (the **accumulated value**) and combines it with the **current element**, producing a new accumulated value. This repeats until the list is exhausted.

---

## 📝 The Code

Here's how that looks in Java:

```java
import java.util.List;

List<Integer> numbers = List.of(1, 2, 3, 4, 5);

int total = numbers.stream()
    .reduce(0, (acc, current) -> acc + current);

System.out.println(total); // 15
```

Let's break down `.reduce(0, (acc, current) -> acc + current)`:

- **`0`** → the starting value (also called the _identity_)
- **`acc`** → the accumulated value, starts at `0` and grows each step
- **`current`** → the current element from the list
- **`acc + current`** → what we want to combine them into

You can name `acc` and `current` anything you like — they're just lambda parameters:

```java
.reduce(0, (runningTotal, nextNumber) -> runningTotal + nextNumber)
// Does exactly the same thing!
```

---

## 🔢 More Number Examples

**Find the maximum value:**

```java
List<Integer> numbers = List.of(3, 7, 2, 9, 4);

int max = numbers.stream()
    .reduce(Integer.MIN_VALUE, (acc, current) -> acc > current ? acc : current);

System.out.println(max); // 9
```

**Multiply all numbers together:**

```java
List<Integer> numbers = List.of(1, 2, 3, 4, 5);

int product = numbers.stream()
    .reduce(1, (acc, current) -> acc * current);

System.out.println(product); // 120
```

> 💡 Notice the starting value changed to `1` here. Why? Because multiplying by `0` would wipe everything out!

---

## 🎨 Wait — It Doesn't Have to Return a Number!

Here's where `reduce()` gets really fun. You can return **anything** — a String, a List, even a custom object.

### Joining Strings

```java
List<String> words = List.of("Java", "is", "pretty", "cool");

String sentence = words.stream()
    .reduce("", (acc, current) -> acc + " " + current);

System.out.println(sentence.trim()); // "Java is pretty cool"
```

Visualised:

```
Start:   acc = ""

Step 1:  ""           + " Java"   → " Java"
Step 2:  " Java"      + " is"     → " Java is"
Step 3:  " Java is"   + " pretty" → " Java is pretty"
Step 4:  " Java is pretty" + " cool" → " Java is pretty cool"
```

### Working with Objects — Total Age

Say you have a list of `Person` objects and want the total age across everyone. First, a simple `Person` class:

```java
class Person {
    String name;
    int age;

    Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
}
```

And a list of people:

```java
List<Person> people = List.of(
    new Person("Alice", 30),
    new Person("Bob",   25),
    new Person("Carol", 35),
    new Person("Dave",  28)
);
```

To get the total age, we start from `0` and add each person's age as we go:

```java
int totalAge = people.stream()
    .reduce(0, (acc, person) -> acc + person.age);

System.out.println(totalAge); // 118
```

Visualised step by step:

```
Start:   acc = 0

Step 1:  acc(0)   + Alice's age(30) → acc = 30
Step 2:  acc(30)  + Bob's age(25)   → acc = 55
Step 3:  acc(55)  + Carol's age(35) → acc = 90
Step 4:  acc(90)  + Dave's age(28)  → acc = 118

Result: 118 ✅
```

---

## ❓ Why Do I Sometimes See Three Arguments in reduce()?

You may come across `reduce()` written with three arguments in the wild and wonder what's going on:

```java
people.stream()
    .reduce(
        0,
        (acc, person) -> acc + person.age,
        (a, b) -> a + b                     // ← what's this?
    );
```

That third argument is a **combiner**. It only matters when streams are run in parallel (split across multiple CPU cores). In that case, Java might process chunks of the list separately and then need to know how to combine those chunk results together at the end. The combiner tells it how.

For everyday, non-parallel streams it is completely ignored. You can think of it as Java asking _"just in case I split this work up, how should I stitch the pieces back together?"_ — and for something like addition, the answer is simply `(a, b) -> a + b`.

You'll mostly see it when the stream's input type and output type differ (like going from a `Person` to an `int`). Don't worry about mastering it right now — just know that when you spot that third argument, it's there to handle parallel processing, not to change what the reduce actually does in normal usage.

---

## 🗺️ The Mental Model

Whenever you see `reduce()`, think of it like a **snowball rolling downhill**:

```
⬇ Start with a small snowball (starting value)
⬇ Roll over 1st item → snowball gets bigger (new acc)
⬇ Roll over 2nd item → snowball gets bigger
⬇ Roll over 3rd item → snowball gets bigger
⬇ ... keep going until the hill ends
⬇ You have one big snowball at the bottom (the result) ⛄
```

The lambda `(acc, current) -> ...` is just the rule for **how the snowball grows** each time it picks something up.

---

## ✅ Quick Reference

```java
// Sum numbers
list.stream().reduce(0, (acc, n) -> acc + n);

// Product
list.stream().reduce(1, (acc, n) -> acc * n);

// Max
list.stream().reduce(Integer.MIN_VALUE, (acc, n) -> acc > n ? acc : n);

// Concatenate strings
list.stream().reduce("", (acc, s) -> acc + s);

// Total age from a list of Person objects
people.stream().reduce(0, (acc, person) -> acc + person.age);
```

---

## 🎉 You've Got This!

`reduce()` is one of those things that feels weird at first but becomes second nature quickly. The key insight is:

> **reduce() replaces a loop that builds up a result one element at a time.**

Once you see it that way, you'll start spotting opportunities to use it everywhere. Happy coding! ☕# Java `reduce()` — A Beginner's Guide 🧮

So you've heard about `Stream.reduce()` and wondered what the fuss is about. Don't worry — by the end of this guide it'll click, and you'll actually _enjoy_ using it.

---

## 🤔 What Even Is `reduce()`?

Imagine you have a list of numbers and want to add them all together. One way is a plain old loop:

```java
int[] numbers = {1, 2, 3, 4, 5};
int total = 0;
for (int n : numbers) {
    total = total + n;
}
// total = 15
```

`reduce()` does the exact same thing, just in a more expressive, stream-friendly way. It **reduces** a whole list of values down to a **single result**.

---

## 🧱 The Building Blocks

Before looking at code, let's get the vocabulary straight. `reduce()` works with three concepts:

|Term|What it means|
|---|---|
|**Starting value**|Where we begin (e.g. `0` for addition)|
|**Accumulated value**|The running result so far|
|**Current element**|The item from the list we're processing right now|

---

## 👀 Visualising What reduce() Does

Let's add `[1, 2, 3, 4, 5]` together with a starting value of `0`.

```
Start:   acc = 0

Step 1:  acc(0)  + current(1)  → acc = 1
Step 2:  acc(1)  + current(2)  → acc = 3
Step 3:  acc(3)  + current(3)  → acc = 6
Step 4:  acc(6)  + current(4)  → acc = 10
Step 5:  acc(10) + current(5)  → acc = 15

Result: 15 ✅
```

Each step takes whatever we've built up so far (the **accumulated value**) and combines it with the **current element**, producing a new accumulated value. This repeats until the list is exhausted.

---

## 📝 The Code

Here's how that looks in Java:

```java
import java.util.List;

List<Integer> numbers = List.of(1, 2, 3, 4, 5);

int total = numbers.stream()
    .reduce(0, (acc, current) -> acc + current);

System.out.println(total); // 15
```

Let's break down `.reduce(0, (acc, current) -> acc + current)`:

- **`0`** → the starting value (also called the _identity_)
- **`acc`** → the accumulated value, starts at `0` and grows each step
- **`current`** → the current element from the list
- **`acc + current`** → what we want to combine them into

You can name `acc` and `current` anything you like — they're just lambda parameters:

```java
.reduce(0, (runningTotal, nextNumber) -> runningTotal + nextNumber)
// Does exactly the same thing!
```

---

## 🔢 More Number Examples

**Find the maximum value:**

```java
List<Integer> numbers = List.of(3, 7, 2, 9, 4);

int max = numbers.stream()
    .reduce(Integer.MIN_VALUE, (acc, current) -> acc > current ? acc : current);

System.out.println(max); // 9
```

**Multiply all numbers together:**

```java
List<Integer> numbers = List.of(1, 2, 3, 4, 5);

int product = numbers.stream()
    .reduce(1, (acc, current) -> acc * current);

System.out.println(product); // 120
```

> 💡 Notice the starting value changed to `1` here. Why? Because multiplying by `0` would wipe everything out!

---

## 🎨 Wait — It Doesn't Have to Return a Number!

Here's where `reduce()` gets really fun. You can return **anything** — a String, a List, even a custom object.

### Joining Strings

```java
List<String> words = List.of("Java", "is", "pretty", "cool");

String sentence = words.stream()
    .reduce("", (acc, current) -> acc + " " + current);

System.out.println(sentence.trim()); // "Java is pretty cool"
```

Visualised:

```
Start:   acc = ""

Step 1:  ""           + " Java"   → " Java"
Step 2:  " Java"      + " is"     → " Java is"
Step 3:  " Java is"   + " pretty" → " Java is pretty"
Step 4:  " Java is pretty" + " cool" → " Java is pretty cool"
```

### Working with Objects — Total Age

Say you have a list of `Person` objects and want the total age across everyone. First, a simple `Person` class:

```java
class Person {
    String name;
    int age;

    Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
}
```

And a list of people:

```java
List<Person> people = List.of(
    new Person("Alice", 30),
    new Person("Bob",   25),
    new Person("Carol", 35),
    new Person("Dave",  28)
);
```

To get the total age, we start from `0` and add each person's age as we go:

```java
int totalAge = people.stream()
    .reduce(0, (acc, person) -> acc + person.age);

System.out.println(totalAge); // 118
```

Visualised step by step:

```
Start:   acc = 0

Step 1:  acc(0)   + Alice's age(30) → acc = 30
Step 2:  acc(30)  + Bob's age(25)   → acc = 55
Step 3:  acc(55)  + Carol's age(35) → acc = 90
Step 4:  acc(90)  + Dave's age(28)  → acc = 118

Result: 118 ✅
```

---

## ❓ Why Do I Sometimes See Three Arguments in reduce()?

You may come across `reduce()` written with three arguments in the wild and wonder what's going on:

```java
people.stream()
    .reduce(
        0,
        (acc, person) -> acc + person.age,
        (a, b) -> a + b                     // ← what's this?
    );
```

That third argument is a **combiner**. It only matters when streams are run in parallel (split across multiple CPU cores). In that case, Java might process chunks of the list separately and then need to know how to combine those chunk results together at the end. The combiner tells it how.

For everyday, non-parallel streams it is completely ignored. You can think of it as Java asking _"just in case I split this work up, how should I stitch the pieces back together?"_ — and for something like addition, the answer is simply `(a, b) -> a + b`.

You'll mostly see it when the stream's input type and output type differ (like going from a `Person` to an `int`). Don't worry about mastering it right now — just know that when you spot that third argument, it's there to handle parallel processing, not to change what the reduce actually does in normal usage.

---

## 🗺️ The Mental Model

Whenever you see `reduce()`, think of it like a **snowball rolling downhill**:

```
⬇ Start with a small snowball (starting value)
⬇ Roll over 1st item → snowball gets bigger (new acc)
⬇ Roll over 2nd item → snowball gets bigger
⬇ Roll over 3rd item → snowball gets bigger
⬇ ... keep going until the hill ends
⬇ You have one big snowball at the bottom (the result) ⛄
```

The lambda `(acc, current) -> ...` is just the rule for **how the snowball grows** each time it picks something up.

---

## ✅ Quick Reference

```java
// Sum numbers
list.stream().reduce(0, (acc, n) -> acc + n);

// Product
list.stream().reduce(1, (acc, n) -> acc * n);

// Max
list.stream().reduce(Integer.MIN_VALUE, (acc, n) -> acc > n ? acc : n);

// Concatenate strings
list.stream().reduce("", (acc, s) -> acc + s);

// Total age from a list of Person objects
people.stream().reduce(0, (acc, person) -> acc + person.age);
```

---

## 🎉 You've Got This!

`reduce()` is one of those things that feels weird at first but becomes second nature quickly. The key insight is:

> **reduce() replaces a loop that builds up a result one element at a time.**

Once you see it that way, you'll start spotting opportunities to use it everywhere. Happy coding! ☕