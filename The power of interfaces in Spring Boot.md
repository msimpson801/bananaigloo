# 🎸 The Power of Interfaces in Spring Boot

### _A step-by-step guide using the world's greatest music_

---

## Step 1 — Start Simple: A Controller and a Service

You're building a music app. The goal: hit an endpoint, get back a list of bands. Easy.

You create a `BandService` that returns some classic rock legends, and a `BandController` that exposes them at `GET /bands`.

```java
// BandService.java
@Service
public class BandService {

    public List<String> getBands() {
        return List.of(
            "Led Zeppelin",
            "The Rolling Stones",
            "Fleetwood Mac",
            "Deep Purple"
        );
    }
}
```

```java
// BandController.java
@RestController
public class BandController {

    private final BandService bandService;

    public BandController(BandService bandService) {
        this.bandService = bandService;
    }

    @GetMapping("/bands")
    public List<String> getBands() {
        return bandService.getBands();
    }
}
```

Hit `GET /bands` and you get:

```json
["Led Zeppelin", "The Rolling Stones", "Fleetwood Mac", "Deep Purple"]
```

🎉 Clean. Simple. The controller asks the service for bands, the service delivers. Life is good.

---

## Step 2 — The Plot Twist

Your app is a hit. But now comes **The New Requirement™**:

> _"In production we want classic rock. In our dev environment, we want jazz. Oh and our testers want 90s hip-hop. And please don't break anything."_

Your first instinct might be to add some `if` statements to `BandService`. Maybe check a config property. Maybe pass a `genre` flag. Before long, your clean little service looks like a Saturday night setlist that went badly wrong.

```java
// ❌ Please don't do this
public List<String> getBands(String genre) {
    if (genre.equals("jazz")) {
        return List.of("Miles Davis", "John Coltrane"...);
    } else if (genre.equals("hiphop")) {
        return List.of("Nas", "Wu-Tang Clan"...);
    } else {
        // ... and so on forever
    }
}
```

This is messy, hard to test, and a nightmare to extend. There's a much better way.

---

## Step 3 — Extract the Interface

An interface is a **contract**. It says: _"whoever implements me must have these methods."_ It doesn't care about the how — just the what.

Let's turn `BandService` from a concrete class into an interface:

```java
// BandService.java
public interface BandService {
    List<String> getBands();
}
```

That's it. Now we write separate implementations — one per genre:

```java
// ClassicRockBandService.java
@Service
public class ClassicRockBandService implements BandService {

    @Override
    public List<String> getBands() {
        return List.of(
            "Led Zeppelin",
            "The Rolling Stones",
            "Fleetwood Mac",
            "Deep Purple"
        );
    }
}
```

```java
// JazzBandService.java
@Service
public class JazzBandService implements BandService {

    @Override
    public List<String> getBands() {
        return List.of(
            "Miles Davis Quintet",
            "Dave Brubeck Quartet",
            "John Coltrane",
            "Thelonious Monk Trio"
        );
    }
}
```

```java
// HipHopBandService.java
@Service
public class HipHopBandService implements BandService {

    @Override
    public List<String> getBands() {
        return List.of(
            "Wu-Tang Clan",
            "Nas",
            "A Tribe Called Quest",
            "The Notorious B.I.G."
        );
    }
}
```

> **The controller stays completely untouched.** It still just asks for a `BandService`. It has no idea which one it's going to get — and that's the whole point.

---

## Step 4 — Spring's Crisis

Here's where Spring panics a little. You now have **three beans** that all implement `BandService`. When the controller's constructor asks for one, Spring has a meltdown:

```
***************************
APPLICATION FAILED TO START
***************************

Parameter 0 of constructor in BandController required a single bean,
but 3 were found:
  - classicRockBandService
  - jazzBandService
  - hipHopBandService
```

Spring needs guidance. It gives you three elegant ways to provide it.

---

## Step 5 — Solution One: `@Primary`

The simplest fix. Crown one implementation as the default. When Spring is unsure which bean to inject, it picks the one marked `@Primary`.

```java
// ClassicRockBandService.java
@Primary   // 👑 This one is the default
@Service
public class ClassicRockBandService implements BandService {

    @Override
    public List<String> getBands() {
        return List.of(
            "Led Zeppelin",
            "The Rolling Stones",
            "Fleetwood Mac",
            "Deep Purple"
        );
    }
}
```

Now when the controller asks for a `BandService`, Spring immediately reaches for `ClassicRockBandService`. No drama.

**Best for:** When you have one obvious "main" implementation and the others are exceptions.

---

## Step 6 — Solution Two: `@Qualifier`

Sometimes `@Primary` isn't enough — especially if you need **multiple implementations in the same class**. What if you want a `MixedTapeController` that returns both jazz _and_ hip-hop bands on different endpoints?

Give each bean a name with `@Qualifier`, then ask for it by name at the injection point:

```java
// JazzBandService.java
@Service
@Qualifier("jazz")
public class JazzBandService implements BandService {

    @Override
    public List<String> getBands() {
        return List.of(
            "Miles Davis Quintet",
            "Dave Brubeck Quartet",
            "John Coltrane",
            "Thelonious Monk Trio"
        );
    }
}
```

```java
// HipHopBandService.java
@Service
@Qualifier("hiphop")
public class HipHopBandService implements BandService {

    @Override
    public List<String> getBands() {
        return List.of(
            "Wu-Tang Clan",
            "Nas",
            "A Tribe Called Quest",
            "The Notorious B.I.G."
        );
    }
}
```

```java
// MixedTapeController.java
@RestController
public class MixedTapeController {

    private final BandService jazzService;
    private final BandService hipHopService;

    public MixedTapeController(
        @Qualifier("jazz")   BandService jazzService,
        @Qualifier("hiphop") BandService hipHopService
    ) {
        this.jazzService   = jazzService;
        this.hipHopService = hipHopService;
    }

    @GetMapping("/bands/jazz")
    public List<String> jazz() {
        return jazzService.getBands();
    }

    @GetMapping("/bands/hiphop")
    public List<String> hipHop() {
        return hipHopService.getBands();
    }
}
```

`GET /bands/jazz` → Miles Davis. `GET /bands/hiphop` → Wu-Tang. Perfect.

**Best for:** When you genuinely need multiple implementations in the same class, or want surgical control over exactly which bean goes where.

---

## Step 7 — Solution Three: `@Profile` ✨ (The Cool One)

This is the one that makes developers' eyes light up. Spring Profiles let you activate **different beans depending on which environment you're running in** — development, staging, production — with zero code changes.

Tag each service with the profile it belongs to:

```java
// ClassicRockBandService.java
@Service
@Profile("prod")   // 🚀 Production gets the classics
public class ClassicRockBandService implements BandService {

    @Override
    public List<String> getBands() {
        return List.of(
            "Led Zeppelin",
            "The Rolling Stones",
            "Fleetwood Mac",
            "Deep Purple"
        );
    }
}
```

```java
// JazzBandService.java
@Service
@Profile("dev")   // 🛠️ Developers get jazz
public class JazzBandService implements BandService {

    @Override
    public List<String> getBands() {
        return List.of(
            "Miles Davis Quintet",
            "Dave Brubeck Quartet",
            "John Coltrane",
            "Thelonious Monk Trio"
        );
    }
}
```

```java
// HipHopBandService.java
@Service
@Profile("test")   // 🧪 Testers get 90s hip-hop
public class HipHopBandService implements BandService {

    @Override
    public List<String> getBands() {
        return List.of(
            "Wu-Tang Clan",
            "Nas",
            "A Tribe Called Quest",
            "The Notorious B.I.G."
        );
    }
}
```

Activate the right profile in `application.properties`:

```properties
spring.profiles.active=dev
```

Or pass it as an argument at startup:

```bash
# Production
java -jar myapp.jar --spring.profiles.active=prod

# Test environment
java -jar myapp.jar --spring.profiles.active=test
```

|Profile|Active Bean|Returns|
|---|---|---|
|`dev`|`JazzBandService`|Miles Davis, John Coltrane...|
|`test`|`HipHopBandService`|Wu-Tang Clan, Nas...|
|`prod`|`ClassicRockBandService`|Led Zeppelin, Fleetwood Mac...|

**The controller never changes. Not one character.**

**Best for:** Environment-specific behaviour. Also brilliant for swapping real integrations for fake ones — real Spotify API in prod, stubbed responses in dev.

---

## The Big Picture

Here's what you built, step by step:

```
BandController
    │
    └── depends on ──▶ BandService  (interface — just a contract)
                            │
                ┌───────────┼───────────┐
                ▼           ▼           ▼
        ClassicRock      Jazz        HipHop
        BandService   BandService  BandService
        (@Profile     (@Profile    (@Profile
          "prod")       "dev")       "test")
```

The controller only knows about the interface. Spring decides which implementation to wire in — based on your annotations. You never touch the controller again.

### Quick Reference

|Annotation|What it does|Best for|
|---|---|---|
|`@Primary`|Marks one bean as the default when multiple match|One obvious main implementation|
|`@Qualifier`|Names a bean so it can be requested explicitly|Multiple implementations in one class|
|`@Profile`|Activates a bean only in specific environments|Swapping behaviour per environment|

---

> An interface is a promise. `@Primary`, `@Qualifier`, and `@Profile` are how Spring decides who keeps it.

Now go forth and refactor. Your future self — and your colleagues — will thank you. 🎸# 🎸 The Power of Interfaces in Spring Boot

### _A step-by-step guide using the world's greatest music_

---

## Step 1 — Start Simple: A Controller and a Service

You're building a music app. The goal: hit an endpoint, get back a list of bands. Easy.

You create a `BandService` that returns some classic rock legends, and a `BandController` that exposes them at `GET /bands`.

```java
// BandService.java
@Service
public class BandService {

    public List<String> getBands() {
        return List.of(
            "Led Zeppelin",
            "The Rolling Stones",
            "Fleetwood Mac",
            "Deep Purple"
        );
    }
}
```

```java
// BandController.java
@RestController
public class BandController {

    private final BandService bandService;

    public BandController(BandService bandService) {
        this.bandService = bandService;
    }

    @GetMapping("/bands")
    public List<String> getBands() {
        return bandService.getBands();
    }
}
```

Hit `GET /bands` and you get:

```json
["Led Zeppelin", "The Rolling Stones", "Fleetwood Mac", "Deep Purple"]
```

🎉 Clean. Simple. The controller asks the service for bands, the service delivers. Life is good.

---

## Step 2 — The Plot Twist

Your app is a hit. But now comes **The New Requirement™**:

> _"In production we want classic rock. In our dev environment, we want jazz. Oh and our testers want 90s hip-hop. And please don't break anything."_

Your first instinct might be to add some `if` statements to `BandService`. Maybe check a config property. Maybe pass a `genre` flag. Before long, your clean little service looks like a Saturday night setlist that went badly wrong.

```java
// ❌ Please don't do this
public List<String> getBands(String genre) {
    if (genre.equals("jazz")) {
        return List.of("Miles Davis", "John Coltrane"...);
    } else if (genre.equals("hiphop")) {
        return List.of("Nas", "Wu-Tang Clan"...);
    } else {
        // ... and so on forever
    }
}
```

This is messy, hard to test, and a nightmare to extend. There's a much better way.

---

## Step 3 — Extract the Interface

An interface is a **contract**. It says: _"whoever implements me must have these methods."_ It doesn't care about the how — just the what.

Let's turn `BandService` from a concrete class into an interface:

```java
// BandService.java
public interface BandService {
    List<String> getBands();
}
```

That's it. Now we write separate implementations — one per genre:

```java
// ClassicRockBandService.java
@Service
public class ClassicRockBandService implements BandService {

    @Override
    public List<String> getBands() {
        return List.of(
            "Led Zeppelin",
            "The Rolling Stones",
            "Fleetwood Mac",
            "Deep Purple"
        );
    }
}
```

```java
// JazzBandService.java
@Service
public class JazzBandService implements BandService {

    @Override
    public List<String> getBands() {
        return List.of(
            "Miles Davis Quintet",
            "Dave Brubeck Quartet",
            "John Coltrane",
            "Thelonious Monk Trio"
        );
    }
}
```

```java
// HipHopBandService.java
@Service
public class HipHopBandService implements BandService {

    @Override
    public List<String> getBands() {
        return List.of(
            "Wu-Tang Clan",
            "Nas",
            "A Tribe Called Quest",
            "The Notorious B.I.G."
        );
    }
}
```

> **The controller stays completely untouched.** It still just asks for a `BandService`. It has no idea which one it's going to get — and that's the whole point.

---

## Step 4 — Spring's Crisis

Here's where Spring panics a little. You now have **three beans** that all implement `BandService`. When the controller's constructor asks for one, Spring has a meltdown:

```
***************************
APPLICATION FAILED TO START
***************************

Parameter 0 of constructor in BandController required a single bean,
but 3 were found:
  - classicRockBandService
  - jazzBandService
  - hipHopBandService
```

Spring needs guidance. It gives you three elegant ways to provide it.

---

## Step 5 — Solution One: `@Primary`

The simplest fix. Crown one implementation as the default. When Spring is unsure which bean to inject, it picks the one marked `@Primary`.

```java
// ClassicRockBandService.java
@Primary   // 👑 This one is the default
@Service
public class ClassicRockBandService implements BandService {

    @Override
    public List<String> getBands() {
        return List.of(
            "Led Zeppelin",
            "The Rolling Stones",
            "Fleetwood Mac",
            "Deep Purple"
        );
    }
}
```

Now when the controller asks for a `BandService`, Spring immediately reaches for `ClassicRockBandService`. No drama.

**Best for:** When you have one obvious "main" implementation and the others are exceptions.

---

## Step 6 — Solution Two: `@Qualifier`

Sometimes `@Primary` isn't enough — especially if you need **multiple implementations in the same class**. What if you want a `MixedTapeController` that returns both jazz _and_ hip-hop bands on different endpoints?

Give each bean a name with `@Qualifier`, then ask for it by name at the injection point:

```java
// JazzBandService.java
@Service
@Qualifier("jazz")
public class JazzBandService implements BandService {

    @Override
    public List<String> getBands() {
        return List.of(
            "Miles Davis Quintet",
            "Dave Brubeck Quartet",
            "John Coltrane",
            "Thelonious Monk Trio"
        );
    }
}
```

```java
// HipHopBandService.java
@Service
@Qualifier("hiphop")
public class HipHopBandService implements BandService {

    @Override
    public List<String> getBands() {
        return List.of(
            "Wu-Tang Clan",
            "Nas",
            "A Tribe Called Quest",
            "The Notorious B.I.G."
        );
    }
}
```

```java
// MixedTapeController.java
@RestController
public class MixedTapeController {

    private final BandService jazzService;
    private final BandService hipHopService;

    public MixedTapeController(
        @Qualifier("jazz")   BandService jazzService,
        @Qualifier("hiphop") BandService hipHopService
    ) {
        this.jazzService   = jazzService;
        this.hipHopService = hipHopService;
    }

    @GetMapping("/bands/jazz")
    public List<String> jazz() {
        return jazzService.getBands();
    }

    @GetMapping("/bands/hiphop")
    public List<String> hipHop() {
        return hipHopService.getBands();
    }
}
```

`GET /bands/jazz` → Miles Davis. `GET /bands/hiphop` → Wu-Tang. Perfect.

**Best for:** When you genuinely need multiple implementations in the same class, or want surgical control over exactly which bean goes where.

---

## Step 7 — Solution Three: `@Profile` ✨ (The Cool One)

This is the one that makes developers' eyes light up. Spring Profiles let you activate **different beans depending on which environment you're running in** — development, staging, production — with zero code changes.

Tag each service with the profile it belongs to:

```java
// ClassicRockBandService.java
@Service
@Profile("prod")   // 🚀 Production gets the classics
public class ClassicRockBandService implements BandService {

    @Override
    public List<String> getBands() {
        return List.of(
            "Led Zeppelin",
            "The Rolling Stones",
            "Fleetwood Mac",
            "Deep Purple"
        );
    }
}
```

```java
// JazzBandService.java
@Service
@Profile("dev")   // 🛠️ Developers get jazz
public class JazzBandService implements BandService {

    @Override
    public List<String> getBands() {
        return List.of(
            "Miles Davis Quintet",
            "Dave Brubeck Quartet",
            "John Coltrane",
            "Thelonious Monk Trio"
        );
    }
}
```

```java
// HipHopBandService.java
@Service
@Profile("test")   // 🧪 Testers get 90s hip-hop
public class HipHopBandService implements BandService {

    @Override
    public List<String> getBands() {
        return List.of(
            "Wu-Tang Clan",
            "Nas",
            "A Tribe Called Quest",
            "The Notorious B.I.G."
        );
    }
}
```

Activate the right profile in `application.properties`:

```properties
spring.profiles.active=dev
```

Or pass it as an argument at startup:

```bash
# Production
java -jar myapp.jar --spring.profiles.active=prod

# Test environment
java -jar myapp.jar --spring.profiles.active=test
```

|Profile|Active Bean|Returns|
|---|---|---|
|`dev`|`JazzBandService`|Miles Davis, John Coltrane...|
|`test`|`HipHopBandService`|Wu-Tang Clan, Nas...|
|`prod`|`ClassicRockBandService`|Led Zeppelin, Fleetwood Mac...|

**The controller never changes. Not one character.**

**Best for:** Environment-specific behaviour. Also brilliant for swapping real integrations for fake ones — real Spotify API in prod, stubbed responses in dev.

---

## The Big Picture

Here's what you built, step by step:

```
BandController
    │
    └── depends on ──▶ BandService  (interface — just a contract)
                            │
                ┌───────────┼───────────┐
                ▼           ▼           ▼
        ClassicRock      Jazz        HipHop
        BandService   BandService  BandService
        (@Profile     (@Profile    (@Profile
          "prod")       "dev")       "test")
```

The controller only knows about the interface. Spring decides which implementation to wire in — based on your annotations. You never touch the controller again.

### Quick Reference

|Annotation|What it does|Best for|
|---|---|---|
|`@Primary`|Marks one bean as the default when multiple match|One obvious main implementation|
|`@Qualifier`|Names a bean so it can be requested explicitly|Multiple implementations in one class|
|`@Profile`|Activates a bean only in specific environments|Swapping behaviour per environment|

---

> An interface is a promise. `@Primary`, `@Qualifier`, and `@Profile` are how Spring decides who keeps it.

Now go forth and refactor. Your future self — and your colleagues — will thank you. 🎸