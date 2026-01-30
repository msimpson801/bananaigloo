# The Weird and Wonderful World of Generic Methods

## The Problem: Why We Need Generic Methods

You know how every object in Java has a `toString()` method?  
It prints stuff out.

You can call `toString()` on **literally any object in Java**:

- a unicorn 🦄
    
- an odd sock 🧦
    
- a Hobnob biscuit 🍪
    
- an inflatable banana 🍌
    

Now imagine you have separate lists of each of these weird and wonderful things, and you want to print them all out.

We _could_ write a method that prints a list of odd socks.  
Then another for unicorns.  
Then another for Hobnobs.  
Then another for inflatable bananas.

And another.  
And another.  
And another.

Nightmare fuel.

java

```java
public void printOddSocks(List<OddSock> socks) {
    for (OddSock sock : socks) {
        System.out.println(sock);
    }
}

public void printUnicorns(List<Unicorn> unicorns) {
    for (Unicorn unicorn : unicorns) {
        System.out.println(unicorn);
    }
}

public void printHobnobs(List<Hobnob> biscuits) {
    for (Hobnob biscuit : biscuits) {
        System.out.println(biscuit);
    }
}

public void printInflatableBananas(List<InflatableBanana> bananas) {
    for (InflatableBanana banana : bananas) {
        System.out.println(banana);
    }
}
```

That's a lot of helper methods doing the exact same thing. What if we could just write ONE method and one method only?

## Enter the Thingymabob Printer 5000™

Behold: a **generic method**.

java

```java
public <T> void printThingymabobs(List<T> thingymabobs) {
    for (T thingymabob : thingymabobs) {
        System.out.println(thingymabob);
    }
}
```

It's going to take in a list of thingymabobs and print them to screen. Simple as that.

## Wait… What’s With the `T`?

`T` is just a **placeholder for a type** that will be decided later.

Most programmers use `T` because it’s short for _Type_, but you could technically call it anything:

- `T`
    
- `Thingymabob`
    
- `Flibberdoodle`
    
- `Whatchamacallit`
    


The key idea: **`T` represents a type that Java will figure out later**.

## One Method, Many Types

Now you can print _anything_:

java

```java
printThingymabobs(myOddSocks);    // T = OddSock
printThingymabobs(myUnicorns);     // T = Unicorn
printThingymabobs(myHobnobs);      // T = Hobnob
```


Same method.  
Different types.  
Zero copy-paste.

✨ Magic ✨
## Java Figures It Out For You (Type Inference Magic!)

Notice we didn't have to write:

java

```java
printThingymabobs<OddSock>(myOddSocks);  // You CAN do this, but why bother?
```

Java looks at the argument and goes:

> “Ah yes. Sock-shaped objects. I know what `T` is.”

This is called **type inference**, which is a fancy way of saying:

> “Java isn’t  a silly goose  — it can work it out.”

## Let's Decode That Method Signature

java

```java
public <T> void printThingymabobs(List<T> thingymabobs)
```

Breaking it down part by part:

**`<T>`** - "Oi! I'm a generic method! I work with types that haven't been decided yet!"

**`void`** - This particular method is void and doesn't return anything

**`printThingymabobs`** - The method name (you can call it whatever you like)

**`List<T>`** - I will accept a list of... whatever `T` ends up being

## ❗ Important: Where Does `<T>` Go?

This trips up literally every beginner, so pay attention! The `<T>` goes **before the return type**, not before the method name.

java

```java
// ✅ Correct
public <T> T getFirst(List<T> items)

// ❌ Wrong (won't compile and will make you sad)
public T <T> getFirst(List<T> items)
```

**Why this weird order?** Java needs to know about `T` before it sees `T` being used anywhere else in the signature. Think of it like introducing someone at a party: "Everyone, this is `T`" (the `<T>` bit), and then `T` can participate in the rest of the conversation (the return type and parameters).

Remember: `<T>` comes first, _always_. This little rule will save you from a lot of compiler-error misery.

## More Generic Method Goodness

Once you understand the basics, you can write all sorts of generic methods:

### The Swaperoo

Puts the first thing in the list last and the last thing first:

java

```java
public <T> void swaperoo(List<T> items) {
    if (items.size() < 2) return;
    T first = items.get(0);
    T last = items.get(items.size() - 1);
    items.set(0, last);
    items.set(items.size() - 1, first);
}
```

### The Random Thing Getter

Gets a random item from a list based on its size:

java

```java
public <T> T getRandomThing(List<T> items) {
    int randomIndex = new Random().nextInt(items.size());
    return items.get(randomIndex);
}
```

### The HooHah Counter

Counts the number of duplicates of a particular item. I give you a list of eggs, you tell me how many pickled eggs are in my list:

java

```java
public <T> int countTheNumberOfHooHahs(List<T> items, T targetHooHah) {
    int count = 0;
    for (T item : items) {
        if (item.equals(targetHooHah)) {
            count++;
        }
    }
    return count;
}
```

## Sometimes You Want _Almost_ Anything (Bounded Generics)

So far, our generic methods accept **anything at all**.

But sometimes you want limits.

For example: finding the biggest number in a list should work for `Integer`, `Double`, `Float`, etc.—but not for odd socks or inflatable bananas.

Enter **bounded generics**:

java

```java
public <T extends Number> T findBiggestNumber(List<T> numbers) {
    if (numbers.isEmpty()) return null;
    
    T biggest = numbers.get(0);
    for (T number : numbers) {
        if (number.doubleValue() > biggest.doubleValue()) {
            biggest = number;
        }
    }
    return biggest;
}
```

**What's that `<T extends Number>` doing?** It's saying "`T` can be any type, as long as it's a `Number` or something that extends `Number`." So you can use it with:

java

```java
findBiggestNumber(myIntegers);   // ✅ Works!
findBiggestNumber(myDoubles);    // ✅ Works!
findBiggestNumber(myOddSocks);   // ❌ Compiler says "Nope!"
```

This gives you the flexibility of generics while still having some safety rails. You're saying "I want this to work with lots of types, but they all need to have something in common."

## Quick Reference Guide

Here are the essential patterns you'll use:

### Generic method that returns nothing

java

```java
public <T> void doSomething(List<T> items) {
    // Do stuff with items
}
```

### Generic method that returns something (must be of same type, of course)

java

```java
public <T> T getSomething(List<T> items) {
    // Do stuff
    return items.get(0);
}
```

### Generic method with multiple parameters

java

```java
public <T> void doSomethingWithTwo(List<T> firstList, T singleItem) {
    firstList.add(singleItem);
}
```

### Generic method with multiple different types

java

```java
public <T, U> void mixAndMatch(List<T> firstList, List<U> secondList) {
    System.out.println("First list has " + firstList.size() + " items");
    System.out.println("Second list has " + secondList.size() + " items");
}
```

### Generic method with bounds

java

```java
public <T extends Number> void doNumberStuff(List<T> numbers) {
    // Can use Number methods like doubleValue(), intValue(), etc.
}
```

## Spot the Generic Methods in the Wild!

Java already has tons of generic methods built in. Let's decode a few so you can recognize them when you see them:

### Collections.sort()

java

```java
public static <T> void sort(List<T> list)
```

**Translation:** "I'm a generic method (see that `<T>`?) that sorts a list. Give me a list of anything—strings, numbers, unicorns—and I'll sort it. I don't return anything because I'm not taking your list and returning you a new sorted list. I'm taking your list and re-jigging it right there on the spot."

### Collections.max()

java

```java
public static <T> T max(Collection<T> collection)
```

**Translation:** "I'm a generic method that finds the biggest thing in a collection. Whatever type you give me (`T`), that's the type I'll give back. Give me numbers, I return a number. Give me strings, I return a string."

### Arrays.asList()

java

```java
public static <T> List<T> asList(T... items)
```

**Translation:** "I'm a generic method that takes however many items you throw at me (that's what the `...` means) and turns them into a list. Whatever type you give me is the type of list I create. So `Arrays.asList(1, 2, 3)` gives you a `List<Integer>`."

### Collections.swap()

java

```java
public static <T> void swap(List<T> list, int i, int j)
```

**Translation:** "I'm a generic method that swaps two items in a list. The `<T>` means I work with any type of list. I need three things: the list itself, and two positions (i and j) to swap. I don't return anything—I just do the swap."

Once you start looking for that `<T>` in method signatures, you'll spot generic methods everywhere. They're the reason you can use the same handy methods on lists of strings, lists of numbers, or lists of inflatable bananas!

## When Should I Use a Generic Method?

Not sure if you need a generic method? Here's a handy checklist:

### ✅ Use a generic method when:

- You're doing the same logic on different types (like our printer example—same printing logic, different objects)
- The method logic doesn't care what the objects actually are (it just needs to loop, sort, swap, count, etc.)
- You want compile-time safety instead of `Object` chaos (the compiler can catch mistakes before you run the code)
- You find yourself copy-pasting the same method for different types

### ❌ Don't use a generic method when:

- The logic depends on specific fields or methods that only certain classes have (like accessing a `name` field that not everything has)
- You're immediately casting everything anyway (that's a code smell—you're probably doing something wrong)
- You're only ever going to use it with one type (just use that type directly and keep it simple)

And that's it! You've now got the power to write one method that works with unicorns, odd socks, Hobnobs, inflatable bananas, and whatever other weird and wonderful things Java throws at you. Happy coding!