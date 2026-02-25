# Spring Data JPA Specifications: A Beginner's Guide to Dynamic Queries

## The Problem: Query Methods Get Out of Hand Fast

When you start using Spring Data JPA, repository query methods feel like magic.

You define an interface, write a method name, and Spring figures out the SQL for you:

```java
List<Country> findByContinent(String continent);
```

Clean. Simple. Brilliant.

But then your requirements get a bit more complex. A user wants to filter countries by continent _and_ whether they're landlocked. No problem:

```java
List<Country> findByContinentAndLandlocked(String continent, boolean landlocked);
```

Then someone wants to also filter by minimum population:

```java
List<Country> findByContinentAndLandlockedAndPopulationGreaterThan(
    String continent, boolean landlocked, long population
);
```

This is already getting unwieldy. And here's the real killer — what if the filters are **optional**? Maybe a user wants to filter by continent only. Or by landlocked status only. Or by all three. Or none.

You'd need a separate method for every possible combination:

```java
List<Country> findByContinent(String continent);
List<Country> findByLandlocked(boolean landlocked);
List<Country> findByContinentAndLandlocked(String continent, boolean landlocked);
List<Country> findByContinentAndPopulationGreaterThan(String continent, long population);
List<Country> findByContinentAndLandlockedAndPopulationGreaterThan(...);
// ... this never ends
```

This is the problem Specifications solve.

---

## The Solution: Reusable Query Building Blocks

Instead of writing one giant method per use case, the **Specification API** lets you write small, focused query pieces — and then **combine them like Lego bricks**.

One piece for "filter by continent". One piece for "filter by landlocked". One piece for "filter by population". Then snap them together however you need.

```
inContinent("Asia") + isLandlocked() + hasPopulationGreaterThan(1_000_000)
```

Sometimes you use all three. Sometimes just one. The pieces don't care — they just do their job.

---

## Setting Things Up

### 1. Your Entity

Let's say you have a `Country` entity:

```java
@Entity
public class Country {

    @Id
    @GeneratedValue
    private Long id;

    private String name;
    private String continent;
    private boolean landlocked;
    private long population;

    // getters and setters...
}
```

### 2. Extend JpaSpecificationExecutor

Your repository needs to extend `JpaSpecificationExecutor`. This is what unlocks Specification support:

```java
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.JpaSpecificationExecutor;

public interface CountryRepository
        extends JpaRepository<Country, Long>,
                JpaSpecificationExecutor<Country> {
}
```

You don't need to add any methods — `JpaSpecificationExecutor` already gives you `findAll(Specification<T> spec)` for free.

---

## Writing Your First Specification

A `Specification<T>` is just a functional interface with one method:

```java
Predicate toPredicate(Root<T> root, CriteriaQuery<?> query, CriteriaBuilder criteriaBuilder);
```

That sounds intimidating. Let's break down what each part means before writing any code.

|Parameter|What it represents|
|---|---|
|`Root<T> root`|Your entity — the "table" you're querying. Use it to refer to columns.|
|`CriteriaQuery<?> query`|The query being built. Mostly used for advanced scenarios.|
|`CriteriaBuilder cb`|A factory for building conditions (`equal`, `greaterThan`, `like`, etc.)|

Think of it this way: `root` is your table, `cb` is your toolbox of conditions.

### A Specification for Continent

```java
import org.springframework.data.jpa.domain.Specification;

public class CountrySpecifications {

    public static Specification<Country> inContinent(String continent) {
        return (root, query, cb) ->
            cb.equal(root.get("continent"), continent);
    }
}
```

That's it. This Specification says: _"give me countries where the continent column equals this value"_.

### A Specification for Landlocked

```java
public static Specification<Country> isLandlocked() {
    return (root, query, cb) ->
        cb.equal(root.get("landlocked"), true);
}
```

### A Specification for Minimum Population

```java
public static Specification<Country> hasPopulationGreaterThan(long minPopulation) {
    return (root, query, cb) ->
        cb.greaterThan(root.get("population"), minPopulation);
}
```

---

## Combining Specifications

Now for the fun part. Spring gives you `.and()`, `.or()`, and `.not()` to combine Specifications.

```java
Specification<Country> spec = CountrySpecifications.inContinent("Asia")
    .and(CountrySpecifications.isLandlocked())
    .and(CountrySpecifications.hasPopulationGreaterThan(5_000_000));

List<Country> results = countryRepository.findAll(spec);
```

Spring Data JPA handles the SQL translation. You just describe _what_ you want.

---

## The Lego Magic: Optional Filters

Here's where Specifications really shine. You can build them up conditionally based on what the user actually asked for:

```java
public List<Country> searchCountries(String continent, Boolean landlocked, Long minPopulation) {

    Specification<Country> spec = Specification.where(null); // start with "no filter"

    if (continent != null) {
        spec = spec.and(CountrySpecifications.inContinent(continent));
    }

    if (landlocked != null) {
        spec = spec.and(CountrySpecifications.isLandlocked());
    }

    if (minPopulation != null) {
        spec = spec.and(CountrySpecifications.hasPopulationGreaterThan(minPopulation));
    }

    return countryRepository.findAll(spec);
}
```

Now compare this to the alternative — you'd need a separate repository method for every combination. With Specifications, you have **three reusable pieces** that work in any configuration.

---

## The Full Picture

Here's everything together in one place:

```java
// CountrySpecifications.java
import org.springframework.data.jpa.domain.Specification;

public class CountrySpecifications {

    public static Specification<Country> inContinent(String continent) {
        return (root, query, cb) ->
            cb.equal(root.get("continent"), continent);
    }

    public static Specification<Country> isLandlocked() {
        return (root, query, cb) ->
            cb.equal(root.get("landlocked"), true);
    }

    public static Specification<Country> hasPopulationGreaterThan(long minPopulation) {
        return (root, query, cb) ->
            cb.greaterThan(root.get("population"), minPopulation);
    }
}
```

```java
// CountryRepository.java
public interface CountryRepository
        extends JpaRepository<Country, Long>,
                JpaSpecificationExecutor<Country> {
}
```

```java
// CountryService.java
public List<Country> searchCountries(String continent, Boolean landlocked, Long minPopulation) {

    Specification<Country> spec = Specification.where(null);

    if (continent != null) {
        spec = spec.and(CountrySpecifications.inContinent(continent));
    }
    if (landlocked != null) {
        spec = spec.and(CountrySpecifications.isLandlocked());
    }
    if (minPopulation != null) {
        spec = spec.and(CountrySpecifications.hasPopulationGreaterThan(minPopulation));
    }

    return countryRepository.findAll(spec);
}
```

---

## Quick Reference: Useful CriteriaBuilder Methods

|Method|What it does|Example|
|---|---|---|
|`cb.equal(root.get("field"), value)`|Equals check|continent = "Asia"|
|`cb.notEqual(root.get("field"), value)`|Not equals|continent != "Europe"|
|`cb.greaterThan(root.get("field"), value)`|Greater than|population > 1000000|
|`cb.lessThan(root.get("field"), value)`|Less than|population < 500000|
|`cb.like(root.get("field"), "%value%")`|Pattern match (SQL LIKE)|name LIKE '%stan%'|
|`cb.isTrue(root.get("field"))`|Boolean true check|landlocked = true|
|`cb.isNull(root.get("field"))`|Null check|capital IS NULL|

---

## When Should You Use Specifications?

Specifications are a great fit when:

- Filters are **optional** — users may or may not apply each one
- You have **more than two or three filter combinations** to support
- You want to **reuse filter logic** across different queries
- Your search form has several toggles, dropdowns, or inputs that all feed into one query

They're probably overkill for simple, fixed queries that never change. A `findByContinent(String continent)` method is still perfectly fine on its own.

---

## Summary

|Approach|Good for|
|---|---|
|Query methods (`findByX`)|Simple, fixed queries|
|`@Query` with JPQL/SQL|Complex but fixed queries|
|**Specifications**|**Dynamic queries with optional filters**|

The Specification API solves a real problem: it stops your repository from becoming a graveyard of method names, and gives you composable, readable building blocks instead. Write each filter once. Combine freely. Add new filters without touching the old ones.

That's the Lego principle — and it's a genuinely useful tool to have in your Spring toolkit.