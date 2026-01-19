# 🐢 From Regular Turtles to Ninja Legends: Understanding `super` in Java

Every turtle in Java knows a few basic things about itself.

It has a **name**.  
It has an **age**.  
And if you ask it who it is, it can introduce itself.

That's just turtle life.

But some turtles… mutate.

In Java, that mutation is called **inheritance**—and the keyword that makes it all work is **`super`**.

Think of `super` as how a child class talks to its parent:

> _“I’ll keep what works… change what doesn’t… and add some brand-new ninja skills.”_

Let’s see how that plays out.

## 🐢 The Parent Class: A Plain Old Turtle

Our parent class defines what every turtle can do.

```java
public class Turtle {
    protected String name;
    protected int age;

    public Turtle(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public void introduce() {
        System.out.println("Hi, I'm " + name + " and I'm " + age + " years old.");
    }

    public void move() {
        System.out.println(name + " slowly crawls forward...");
    }
}
```

Nothing fancy.  
No flips.  
No pizza.

Just a turtle being a turtle.

## 🥷 The Child Class: Same Turtle, More Attitude

Now let's create a `SuperNinjaTurtle`.

Ninja turtles are still turtles, so they have a name and age and can introduce themselves. But they also wear bandanas, carry weapons, have catchphrases, and absolutely do not crawl slowly.

```java
public class SuperNinjaTurtle extends Turtle {
    private String bandanaColor;
    private String weapon;
    private String catchphrase;

    public SuperNinjaTurtle(String name, int age,
                            String bandanaColor,
                            String weapon,
                            String catchphrase) {
        super(name, age);   // Let the parent set up name & age
        this.bandanaColor = bandanaColor;
        this.weapon = weapon;
        this.catchphrase = catchphrase;
    }
```

### 🧱 Constructor Rule of Thumb

The parent already knows how to create a turtle. So we let it handle the basics first — then we add the ninja gear.

## 1️⃣ Calling the Parent Constructor with super()

When we create a ninja turtle, we don't want to rewrite all the name and age setup code. The parent class already does that perfectly. So we call `super(name, age)` to say "Hey parent, you handle your properties, and I'll handle mine."

This has to be the first line in the constructor. Java wants the parent fully set up before the child adds anything extra.

## 2️⃣ Taking Parent Behavior… and Putting a Spin on It

Every turtle can introduce itself. A ninja turtle just has more to say.

```java
    @Override
    public void introduce() {
        super.introduce(); // Keep the turtle intro
        System.out.println(
            "I wear a " + bandanaColor + " bandana and wield " + weapon + "!"
        );
        System.out.println(catchphrase);
    }
```

Here’s what’s happening:

- We **reuse** the parent’s `introduce()` method
    
- Then we **extend** it with ninja details
    

This is the perfect use of `super.method()`:

> _“Do the normal turtle thing… then let me finish.”_

## 3️⃣ Completely Replacing Parent Behavior

Regular turtles move slowly. Ninja turtles? Absolutely not.

```java
    @Override
    public void move() {
        System.out.println(name + " performs a triple backflip while sprinting!");
    }
```

Notice what's missing — no `super.move()`. We're not improving the crawl. We're throwing it away.

Sometimes inheritance isn't about building on behavior. It's about saying: _"Thanks, but I've evolved."_

## 4️⃣ Doing Something the Parent Never Could

Plain turtles don't eat pizza. Ninja turtles… live on it.

```java
    public void eatPizza() {
        System.out.println(name + " devours a slice of pizza. " + catchphrase);
    }
}
```

This method doesn't override anything, doesn't use super, and exists only in the child class. Inheritance doesn't mean you're limited to what the parent gives you. It just gives you a starting point.

## 5️⃣ Accessing Parent Variables with `super`

So far, we’ve used `super` with constructors and methods.  
But `super` can also be used with **variables**.

Most of the time, you won’t need it. Since `name` is marked `protected`, the child class can already access it directly.

You use `super.variable` when you want to be **explicit** about accessing the parent’s version of a variable.

Here’s an example inside the child class:

`public void revealIdentity() {     System.out.println(         "My real name is " + super.name +         ", but my ninja name is " + ninjaName + "."     ); }`

Using `super.name` clearly communicates intent:

> _“I’m referring to the original turtle name, not the ninja identity.”_

### Design Tip 🧠

In clean object-oriented design, it’s usually better to give child variables **distinct names** (like `ninjaName`) rather than reuse the same one.

But understanding `super.variable` is still important because:

- you’ll see it in real-world and legacy code
    
- it explains how inheritance scopes work
    
- it mirrors how `super.method()` behaves


## 🎬 See It in Action

```java
public class TurtlePower {
  public static void main(String[] args) {
        SuperNinjaTurtle leo = new SuperNinjaTurtle(
            "Leonardo",
            "The Blue Shadow",
            15,
            "blue",
            "twin katanas",
            "Turtle Power!"
        );

        leo.introduce();
        System.out.println();

        leo.move();
        System.out.println();

        leo.eatPizza();
        System.out.println();

        leo.revealIdentity();
}
```

**Output:**

```
Hi, I'm Leonardo and I'm 15 years old.
But in the shadows, they call me The Blue Shadow.
I wear a blue bandana and wield twin katanas!
Turtle Power!

The Blue Shadow performs a triple backflip while sprinting!

The Blue Shadow devours a slice of pizza. Turtle Power!

My real name is Leonardo, but my ninja name is The Blue Shadow.
```

## 🧠 The Big Picture

With inheritance, you get four main ways to use **super**:

| What you want            | How you do it                 |
| ------------------------ | ----------------------------- |
| Initialize parent state  | Call `super()` in constructor |
| Build on parent behavior | Call `super.method()`         |
| Access parent variables  | Use `super.variable`          |
| Replace parent behavior  | Override without super        |

And remember: you can always add completely new behavior that the parent never had.

For constructors, let the parent set up shared state, then let the child add its own personality.
## 🐢 Final Thoughts

Every ninja turtle is still a turtle. They just introduce themselves with more swagger, move with more flair, and eat way more pizza.

That's what super is really about: respecting where you came from… while leveling up.

**Cowabunga.** 🍕