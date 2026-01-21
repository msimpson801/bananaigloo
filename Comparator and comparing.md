# Comparator vs Comparable: A Sock Drawer Story Java Will Never Forget

Spring has sprung.

The sun is shining. The birds are singing. It's time to do some spring cleaning. You decide today is the day you will finally sort out the sock drawer.

You open it.

It's chaos.

Socks are everywhere. Long socks. Short socks. Bright socks. That one pair of novelty edible socks you bought as a joke. One lonely widowed sock that lost its partner a long time ago, still holding out hope.

So before you start sorting your socks, you stop and ask a reasonable question:

**How do you even sort socks?**

By color?  
Black and white socks here, coloured socks there?

By pattern?  
Plain socks on one side, patterned chaos on the other?

By purpose?  
Everyday socks in front. The lucky socks hidden at the back for special occasions.

By number of holes?  
By emotional attachment?  
By which ones you're convinced are slightly cursed?

None of these choices are wrong.  
They're just different ways of ordering the same pile of socks, depending on what matters to you at that moment.

This is exactly the situation Java finds itself in.

## When Java Doesn't Have to Think

Some things are easy for Java.

Numbers know their place in the universe.  
1 comes before 2.  
2 comes before 10.

Strings also behave themselves. Java knows that "apple" comes before "banana". Alphabetical order just makes sense.

So when you write:

```java
Collections.sort(numbers);
Collections.sort(words);
```

Java doesn't hesitate. The rules are obvious. The items already know how to line up.

In other words, they have a **natural order**.

## When Java Opens the Sock Drawer

Now you introduce your own sock class:

```java
class Sock {
    String color;
    int size;
    String pattern;
}
```

You hand Java a list of Sock objects and say:

```java
Collections.sort(socks);
```

Java has no idea where to begin. It has all the same questions you had.

"Sort... by what exactly?"

Color?  
Size?  
Pattern?  
Vibes?

Java refuses to guess. It needs instructions.

And this is where Comparable and Comparator come in.

## Comparable: Socks That Know Where They Belong

Sometimes, there is one obvious way to sort something.

Imagine socks that come with a label sewn into them:

**"Sort me by size."**

No debate. No alternatives. A universally agreed upon rule that says when it comes to socks, size matters.

That's **Comparable**.

When a class implements Comparable, it's saying:

"If you ever compare two of us, this is the default way we should be ordered."

```java
class Sock implements Comparable<Sock> {
    String color;
    int size;
    String pattern;

    @Override
    public int compareTo(Sock otherSock) {
        return this.size - otherSock.size;
    }
}
```

Now Java doesn't have to ask questions.

```java
Collections.sort(socks);
```

It just works. Socks line up by size. Everyone understands the rules.

This comparison lives **inside the class**.  
It is the sock's **natural order**.

## Comparator: Deciding on the Fly

But some days, size isn't what you care about.

Today, you want to sort socks by color.  
Tomorrow, by size again.  
Next week, by which socks would survive longest in a post-apocalyptic wasteland.

You can't keep changing the label sewn into every sock. The socks would unionize. There would be consequences.

So instead, you bring in an **external rule**.

This is **Comparator**.

A Comparator doesn't change the socks.  
It just says:

"For this sort, right now, this is the rule."

```java
Comparator<Sock> byColor =
    (s1, s2) -> s1.color.compareTo(s2.color);

Collections.sort(socks, byColor);
```

Same socks.  
Different order.

Tomorrow, you can swap the rule:

```java
Collections.sort(socks, Comparator.comparing(s -> s.size).reversed());
```

No class changes. No drama. No sock revolution.

**Comparator is flexible.**  
**Comparator is temporary.**  
**Comparator is how you sort socks by color today and by pure luck tomorrow.**

## The Secret: They Do the Same Job

Here's the part most people miss:

**Comparable and Comparator do the exact same thing.**

Both answer one question:

"If I compare sock A to sock B, which one comes first?"

Both return a number:

- **Negative number** = A comes first
- **Zero** = they're equal
- **Positive number** = B comes first

Here's what that actually looks like:

```java
return 5 - 10;   // = -5 (negative) = "I go first"
return 10 - 10;  // = 0 (equals) = "we're twins"  
return 10 - 5;   // = 5 (positive) = "you go first"
```

Java doesn't care if you return -1 or -8472.  
Negative? First one wins.  
Positive? Second one wins.

That's it. That's the whole magic trick.

The only difference is **where that logic lives**.

- **Comparable**: the sock knows how to compare itself
- **Comparator**: someone else decides how to compare socks

Same question.  
Same math.  
Different source of truth.

## When Both Exist (And Who Wins)

You can use both at the same time.

```java
class Sock implements Comparable<Sock> {
    public int compareTo(Sock other) {
        return this.size - other.size;
    }
}

Collections.sort(socks);
// Uses Comparable (sorts by size)

Collections.sort(socks, byColor);
// Uses Comparator (sorts by color)
// Comparable is completely ignored
```

If a Comparator is provided, it **always wins**.

Always.

The Comparator walks into the room.  
Comparable looks up from its desk.  
Comparator says "I'm in charge now."  
Comparable nods and steps aside.

No argument. No debate. Comparator wins.

Comparable is the default.  
Comparator is the override.

## Choosing the Right Tool

**Use Comparable when:**

- There is one obvious, natural order
- Every instance should agree on it
- It would feel strange not to build it in
- Shoe sizes? Sort by number (that's just what they are)
- Playing cards? Sort by rank (it's in their DNA)
- Socks? Maybe sort by size (the natural order)

**Use Comparator when:**

- You need multiple ways to sort
- The order depends on context
- You don't own the class
- You want flexibility
- Or when it's Tuesday and you're feeling chaotic

## The End

That's it.

You now understand Comparable and Comparator.

They both compare things.  
They both return numbers.  
One lives inside. One lives outside.

Time to sort those socks.