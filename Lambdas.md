# Lambda Expressions in Java: A Chill Beginner's Guide ☕

## So… What Are Those Arrow Things?

When you start using streams in Java, you'll quickly notice these little arrow-functions popping up everywhere.

```java
x -> x * 2
name -> name.toUpperCase()
```

At first they look strange. Are they methods? Are they magic? 

These are called **lambda expressions**, and once you get the hang of them, they're pretty cool.

## What Is a Lambda (Really)?

A lambda is just a small, throwaway function that you write right where you need it.

Instead of:

- creating a method,
- giving it a name,
- scrolling up and down to find it later…

…you just write the logic inline and move on with your life.

Think of it like this:

- **Regular methods** are like a coffee machine - they can be reused, customised, and are worth investing in.
- **Lambdas** one and done, simple and disposable

If it's quick and obvious, you don't need the cookbook.

## The Shape of a Lambda

Every lambda follows the same basic pattern:

```
(parameters) -> { body }
```

That's it. Just two parts:

- **Parameters** – what comes in
- **Body** – what you do with them

And here's what lambdas **don't** have:

❌**No name** – They're anonymous, so you don't give them a name like `calculateTotal` or `isValid`
❌ **No return type** – Java is smart enough to figure out what type you're returning by looking at your code
 **❌No explicit parameter types** – You don't need to say the parameters are of type String, Integer, etc. Java infers it!

Java is doing a lot of the thinking for you here.

## Parentheses: The Simple Rules

### No Parameters

If your lambda takes nothing in, use empty parentheses:

```java
() -> System.out.println("Hello!")
() -> 42
() -> new ArrayList<>()
```

### One Parameter

With just one parameter, you can skip the parentheses:

```java
x -> x * 2
name -> name.toUpperCase()
n -> n % 2 == 0
```

You _can_ keep them if you want, but most people skip them

```java
(x) -> x * 2 //works, but why bother?
```
### Two or More Parameters

Multiple parameters = parentheses required:

```java
(a, b) -> a + b
(x, y) -> x * y
(firstName, lastName) -> firstName + " " + lastName
```

## "But Where's the Return Type?"

You never write it — Java figures it out from context:

```java
x -> x > 10                // returns boolean
x -> x * 2                 // returns int
name -> name.toUpperCase() // returns String
```

If it looks obvious to you, it probably looks obvious to the compiler too.

## Parameter Types? Also Optional

You don't need to say what type the parameters are either:

```java
// No need to write (Integer x)
numbers.stream().filter(x -> x > 10)

// No need to write (String name)
names.stream().map(name -> name.toUpperCase())

// No need to write (String a, String b)
names.stream().sorted((a, b) -> a.compareTo(b))
```

Java looks at the stream and goes:

_"Ah, I know what type this must be."_

And it just works.

## Lambdas Live Right Where They're Used

One of the nicest things about lambdas is that they're written inline, exactly where they apply:

```java
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);

numbers.stream()
    .filter(n -> n % 2 == 0)
    .map(n -> n * 2)
    .forEach(n -> System.out.println(n));
```

No jumping around your file.  
No hunting for helper methods.  
Everything is right there.

## Lambdas You'll See All the Time

Here are some very common patterns:

```java
// Filtering
words.stream().filter(w -> w.length() > 5)
numbers.stream().filter(n -> n % 2 == 0)

// Mapping (transforming)
names.stream().map(name -> name.toLowerCase())
numbers.stream().map(n -> n * n)

// Sorting
people.stream().sorted((p1, p2) -> p1.getAge() - p2.getAge())

// Doing something with each item
names.stream().forEach(name -> System.out.println(name))
```

## Lambdas vs. Regular Methods

So what's the difference between a lambda and a regular method?

| Lambda            | Regular Method    |
| ----------------- | ----------------- |
| No name           | Has a name        |
| Written inline    | Defined elsewhere |
| Usually used once | Reusable          |
| Short and simple  | Can be complex    |
| No types written  | Types required    |

Same logic, two styles:

**Lambda version:**

```java
numbers.stream().filter(n -> n > 10 && n < 100)
```

**Method version:**

```java
public boolean isBetweenTenAndHundred(int n) {
    return n > 10 && n < 100;
}

numbers.stream().filter(this::isBetweenTenAndHundred);
```

Both are valid. One is just more compact.

## When Not to Use a Lambda

Lambdas are great — but they're not always the right tool.

### When the Logic Gets Messy

Remember: lambdas in `filter()` return true (keep it) or false (ditch it). When that logic gets complex with multiple lines, write a method in your class instead.

Let's say you're running a cheese shop and need to filter your inventory:

```java
// This is... a lot
cheeses.stream()
    .filter(cheese -> {
        int age = cheese.getAgeInMonths();
        double smell = cheese.getSmellLevel();
        boolean hasMold = cheese.hasMold();
        return age >= 6 && smell < 8 && !hasMold;
    });
```

Instead, give it a name:

```java
public boolean isPremiumCheese(Cheese cheese) {
    int age = cheese.getAgeInMonths();
    double smell = cheese.getSmellLevel();
    boolean hasMold = cheese.hasMold();
    return age >= 6 && smell < 8 && !hasMold;
}

cheeses.stream().filter(this::isPremiumCheese);
```

Cleaner. Clearer. Easier to read.

The method returns true to keep the good cheese, false to ditch the funky stuff.

### When You Need It More Than Once

If you're checking the same cheese criteria everywhere:

```java
// Don't repeat this lambda all over the place:
inventory.stream().filter(c -> c.getPrice() < 20 && c.getType().equals("Cheddar"))
shoppingCart.stream().filter(c -> c.getPrice() < 20 && c.getType().equals("Cheddar"))
specialOffers.stream().filter(c -> c.getPrice() < 20 && c.getType().equals("Cheddar"))
```

Write it once:

```java
public boolean isAffordableCheddar(Cheese c) {
    return c.getPrice() < 20 && c.getType().equals("Cheddar");
}
```

Use it everywhere:

```java
inventory.stream().filter(this::isAffordableCheddar)
shoppingCart.stream().filter(this::isAffordableCheddar)
specialOffers.stream().filter(this::isAffordableCheddar)
```

### When a Name Explains It Better

Compare these:

```java
cheeses.stream()
    .filter(c -> c.getAgeInMonths() > 12 && c.getOrigin().equals("France"));
```

vs.

```java
cheeses.stream().filter(this::isFancyAgedFrenchCheese);
```

The second one reads like English — and that matters.

## Method References (That :: Thing)

You can plug existing methods into streams using **method references**:

```java
names.stream().map(String::toUpperCase);
numbers.stream().filter(this::isEven);
```

This is just a shorter way of saying:

_"Use this method here."_

Nothing spooky going on.

## The Simple Rule of Thumb 👍

✅ Use a **lambda** if it's short, simple, and used once

✅ Use a **method** if it's complex, reused, or needs a good name

If your lambda is longer than a few lines or hard to read — promote it to a method.

## Final Thoughts

Lambdas are just tiny, quick functions you write exactly where you need them. They're perfect for simple operations in streams. The compiler handles all the type inference, so you can keep your code clean and concise.

Just remember: if it's simple and one-off, lambda it. If it's complex or reused, method it