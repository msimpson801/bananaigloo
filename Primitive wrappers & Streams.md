# Primitive Wrappers & Streams in Java

### A Beginner's Guide — with Fruit 🍌

---

## Part 1: Primitives vs Objects — What's the difference?

Java has two kinds of values living in your program at any time.

**Primitives** are raw, bare-bones values baked directly into memory. They are not objects. They have no methods. They just _are_ a number, or a letter, or a true/false.

```java
int quantity = 12;       // just the number 12, sitting in memory
double price = 1.99;     // just the decimal 1.99
boolean ripe = true;     // just a true/false bit
char grade = 'A';        // just a single character
```

**Objects** are instances of a class. They live on the heap, they have methods you can call, and crucially — they can be `null`.

```java
String name = "Mango";   // an object, has methods like .length(), .toUpperCase()
Fruit fruit = new Fruit(); // your own object
```

The full list of Java primitives and their object counterparts:

|Primitive|Object Wrapper|Example value|
|---|---|---|
|`int`|`Integer`|42|
|`long`|`Long`|9999999999L|
|`double`|`Double`|3.14|
|`float`|`Float`|3.14f|
|`boolean`|`Boolean`|true|
|`char`|`Character`|'A'|
|`byte`|`Byte`|127|
|`short`|`Short`|32767|

---

## Part 2: Boxing and Unboxing

Converting between a primitive and its wrapper object has a name.

- **Boxing** = wrapping a primitive _into_ its object form
- **Unboxing** = pulling the primitive _out_ of its object wrapper

```java
// Boxing — primitive → object
int raw = 42;
Integer boxed = Integer.valueOf(42);  // explicit boxing
Integer alsoBoxed = 42;              // autoboxing — Java does it for you

// Unboxing — object → primitive
Integer wrapped = Integer.valueOf(42);
int unwrapped = wrapped.intValue();   // explicit unboxing
int alsoUnwrapped = wrapped;         // auto-unboxing — Java does it for you
```

The key word there is **auto**. Java silently boxes and unboxes values for you in many situations, without you writing a single line to make it happen.

---

## Part 3: Where Java is secretly boxing for you

This is where it gets interesting. Java does autoboxing quietly all over the place. Here are the most common situations where it happens without you really noticing.

### Putting primitives into a List

```java
List<Integer> numbers = new ArrayList<>();

numbers.add(1);   // you're passing an int
numbers.add(2);   // Java silently boxes 1, 2, 3 into Integer objects
numbers.add(3);   // before storing them in the list

// Why? Because List<T> requires T to be an Object.
// int is not an Object. Integer is.
// You wrote List<int>... wait, you can't. That won't even compile.
// You must write List<Integer>.
```

The moment you call `numbers.add(1)`, Java intercepts your `int 1` and wraps it into `new Integer(1)` before putting it in the list. You never see this happen.

### Getting a primitive back out

```java
List<Integer> numbers = List.of(10, 20, 30);

int first = numbers.get(0);   // numbers.get(0) returns an Integer
                               // Java auto-unboxes it into int for you
```

### Arithmetic with wrapper objects

```java
Integer a = 10;
Integer b = 20;
int sum = a + b;  // Java unboxes both Integer objects,
                  // adds the raw ints, gives you an int back
```

### Comparisons — the sneaky one

```java
Integer x = 200;
Integer y = 200;

System.out.println(x == y);      // false ⚠️  (comparing object references!)
System.out.println(x.equals(y)); // true  ✅  (comparing actual values)
```

This one catches everyone. When you use `==` on Integer objects, Java compares whether they are the _same object in memory_, not whether they hold the _same value_. Always use `.equals()` when comparing wrapper objects.

> Side note: for `Integer` values between -128 and 127, Java caches the objects and `==` _happens_ to work. This makes it even more confusing, because your code appears to work fine until you hit a value outside that range.

### Ternary operator — the really sneaky one

```java
Integer result = someCondition ? 1 : null;
// If someCondition is false, result is null.
// Then if you try to unbox it...

int value = result;  // NullPointerException 💥
```

Unboxing `null` always throws a `NullPointerException`. This is one of the most unexpected runtime crashes beginners hit, because the code looks perfectly fine.

### Method calls that expect primitives

```java
public static void print(int n) {
    System.out.println(n);
}

Integer boxed = 42;
print(boxed);   // Java auto-unboxes boxed → 42 before passing it in
```

### Returning from a method

```java
public Integer getQuantity() {
    return 12;   // you wrote 12, Java boxes it into Integer(12)
}
```

---

## Part 4: Why does this matter? The cost of boxing

Every time Java boxes a primitive into a wrapper object, it allocates a new object on the heap. That object takes memory, and the garbage collector eventually has to clean it up.

For a one-off value this is invisible. But if you're looping over a million items and boxing each one, it becomes a real performance issue.

```java
// This boxes 1,000,000 ints into Integer objects — expensive
List<Integer> millionNumbers = new ArrayList<>();
for (int i = 0; i < 1_000_000; i++) {
    millionNumbers.add(i);  // boxing on every iteration
}
```

This is exactly why primitive streams exist in Java — but we'll get to that in a moment.

---

## Part 5: Our Fruit class

Here's the class we'll use throughout the streams section.

```java
public class Fruit {
    public String name;
    public String type;    // "tropical" or "temperate"
    public int quantity;   // HOW MANY we have in stock

    public Fruit(String name, String type, int quantity) {
        this.name = name;
        this.type = type;
        this.quantity = quantity;
    }
}
```

And our fruit inventory:

```java
List<Fruit> fruits = List.of(
    new Fruit("Mango",     "tropical",  12),
    new Fruit("Pineapple", "tropical",   8),
    new Fruit("Banana",    "tropical",  20),
    new Fruit("Apple",     "temperate", 15),
    new Fruit("Pear",      "temperate",  7)
);
```

**Goal:** filter for tropical fruits only, then get the total quantity.

---

## Part 6: Streams — what they are, briefly

A stream is a pipeline that lets you describe _what_ you want to do with a collection, rather than writing out the loop mechanics yourself.

```java
// The loop way
int total = 0;
for (Fruit f : fruits) {
    if (f.type.equals("tropical")) {
        total += f.quantity;
    }
}

// The stream way
int total = fruits.stream()
    .filter(f -> f.type.equals("tropical"))
    .mapToInt(f -> f.quantity)
    .sum();
```

Both do the same thing. The stream version is more expressive once you know the vocabulary.

A stream has three parts:

1. **Source** — `fruits.stream()` — where the data comes from
2. **Intermediate operations** — `filter(...)`, `map(...)` — transform the data, can be chained
3. **Terminal operation** — `.sum()`, `.collect()`, `.count()` — produces the result and ends the pipeline

---

## Part 7: The problem — why .map() isn't enough

Now here's the question you're probably asking: _quantity is an int. I can see it right there in the class. Why can't I just do this?_

```java
fruits.stream()
    .filter(f -> f.type.equals("tropical"))
    .map(f -> f.quantity)   // this seems right...
    .sum();                  // ❌ COMPILE ERROR: cannot find symbol sum()
```

This doesn't compile. Here's why.

`stream()` creates a `Stream<Fruit>`. Every element is a `Fruit` object.

When you call `.map(f -> f.quantity)`, you're telling Java: _transform each Fruit into its quantity_. Java does that — but the result has to live inside a stream. And `Stream<T>` requires `T` to be an **object**, not a primitive.

So Java silently **boxes** your `int` quantity into an `Integer` object. The result is a `Stream<Integer>`.

```java
.map(f -> f.quantity)
// Java sees: map(Fruit -> int)
// int can't go in Stream<T> directly
// Java boxes it: map(Fruit -> Integer)
// Result: Stream<Integer>
```

Now you have a `Stream<Integer>`. And here's the painful part: `Stream<Integer>` has no `.sum()` method. No `.average()`. No `.min()` or `.max()` either.

Why not? Because those methods don't make sense on _any_ `Stream<T>`. What would it mean to sum a `Stream<Fruit>`? Or a `Stream<String>`? Java doesn't add math methods to the general-purpose stream type.

---

## Part 8: The solution — mapToInt()

Instead of `.map()`, use `.mapToInt()`. This produces a dedicated `IntStream` — a stream designed specifically for `int` values — and _that_ has all the numeric methods.

```java
fruits.stream()
    .filter(f -> f.type.equals("tropical"))
    .mapToInt(f -> f.quantity)   // ✅ produces an IntStream, no boxing
    .sum();                       // ✅ works! returns 40
```

The difference under the hood:

```
.map(f -> f.quantity)      →  Stream<Integer>  (boxing happens, no math methods)
.mapToInt(f -> f.quantity) →  IntStream        (no boxing, math methods available)
```

`mapToInt()` does _not_ box. It tells Java: I want a stream of raw `int` values. Java keeps them as primitives the whole way through, which is both faster and gives you access to the numeric terminal operations.

---

## Part 9: What IntStream gives you

Once you're in `IntStream`, you have:

```java
IntStream is = fruits.stream()
    .filter(f -> f.type.equals("tropical"))
    .mapToInt(f -> f.quantity);

is.sum();              // 40 — total of all quantities
is.count();            // 3L — how many elements (returns a long)
is.average();          // OptionalDouble containing 13.333...
is.min();              // OptionalInt containing 8
is.max();              // OptionalInt containing 20
is.summaryStatistics(); // all of the above in one object
```

### Why Optional?

`.min()`, `.max()`, and `.average()` return `OptionalInt` or `OptionalDouble` instead of a raw value. That's because your stream might be empty — if no fruits match the filter, there's no meaningful min or max. An `Optional` forces you to handle that case.

```java
OptionalInt max = fruits.stream()
    .filter(f -> f.type.equals("tropical"))
    .mapToInt(f -> f.quantity)
    .max();

// Safe way to use it
int maxQty = max.orElse(0);   // use 0 if stream was empty
```

### Going back to objects: .boxed()

If you need to collect your `IntStream` into a `List`, you have to go back to `Stream<Integer>` first. Use `.boxed()`.

```java
List<Integer> quantities = fruits.stream()
    .filter(f -> f.type.equals("tropical"))
    .mapToInt(f -> f.quantity)
    .boxed()                          // IntStream → Stream<Integer>
    .collect(Collectors.toList());    // [12, 8, 20]
```

---

## Part 10: The three primitive streams

There are three primitive stream types in Java, one for each common numeric type:

|Stream type|For primitive|Has|
|---|---|---|
|`IntStream`|`int`|`.sum()`, `.average()`, `.min()`, `.max()`|
|`LongStream`|`long`|Same|
|`DoubleStream`|`double`|Same|

You get to them with:

```java
.mapToInt(f -> f.someIntField)
.mapToLong(f -> f.someLongField)
.mapToDouble(f -> f.someDoubleField)
```

---

## Summary

Here's the whole story in one place.

**Why do wrappers exist?**  
Because Java generics and collections only work with objects. Primitives are not objects. The wrapper classes (`Integer`, `Double`, etc.) are the object versions of each primitive.

**What is autoboxing?**  
Java automatically converts between primitives and their wrappers when it needs to. Passing `int 5` to something expecting an `Integer`? Java boxes it for you. Getting an `Integer` from a list and assigning it to an `int`? Java unboxes it for you. This happens silently.

**Where does this show up day-to-day?**  
Anywhere you use generics: `List<Integer>`, `Optional<Integer>`, `Map<String, Integer>`. Every `.add(5)` on a list is boxing. Every `int x = list.get(0)` is unboxing.

**Why does this affect streams?**  
`Stream<T>` needs `T` to be an object. So `.map(f -> f.quantity)` boxes your `int` into `Integer`, giving you `Stream<Integer>`. That object stream has no numeric methods.

**What's the fix?**  
Use `.mapToInt()` instead. It gives you an `IntStream` — a primitive stream that skips boxing entirely and comes with `.sum()`, `.average()`, `.min()`, `.max()`, and `.summaryStatistics()` built in.

```java
// The complete pattern
int totalTropicalQuantity = fruits.stream()
    .filter(f -> f.type.equals("tropical"))  // Stream<Fruit>
    .mapToInt(f -> f.quantity)               // IntStream (no boxing)
    .sum();                                   // 40
```