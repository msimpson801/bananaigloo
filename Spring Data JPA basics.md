# Query Your Database Without Writing SQL

### A fun guide to Spring Data JPA

Ever written 50 lines of SQL just to fetch 3 rows from a table?

Yeah. We’ve all been there.

Now imagine this instead:

You write a simple Java interface.  
You don’t write any SQL.  
You don’t implement anything.  
You just… run your app.

And it works.

That’s the magic of **Spring Data JPA**.

It lets you describe your database once using a Java class — and from that point on, you work with clean, simple Java objects instead of messy SQL strings and `ResultSet` gymnastics.

In this guide, we’re going to build something real using a **country** table and see just how far we can go without touching SQL at all.

Spoiler: pretty far 🚀

---

## Step 0 — The Database Table

To make this concrete, let’s picture the actual table sitting in our database.

Nothing fancy. Just five rows and a handful of columns:

|id|name|capital_city|is_landlocked|population|
|---|---|---|---|---|
|1|France|Paris|false|68,000,000|
|2|Switzerland|Bern|true|8,700,000|
|3|Brazil|Brasília|false|215,000,000|
|4|Bolivia|Sucre|true|12,000,000|
|5|Japan|Tokyo|false|125,000,000|

Each row represents one country.  
Each column represents a property of that country.

Now here’s the important part:

We’re going to describe this structure **once** using a Java class. After that, we stop thinking in terms of rows and columns and start thinking in terms of clean, simple Java objects.

No more `"SELECT * FROM country"`.  
Just `Country` objects in your code.

---

## Step 1 — The Entity: Teaching Java About Your Table

Now we turn that database table into something Java understands.

An **Entity** is just a regular Java class that represents a table.  
Each object of that class represents one row.

So:

- One `Country` object → one row in the `country` table
    
- One field → one column
    

We use annotations to tell Spring and JPA how everything maps together. Think of annotations as little sticky notes that say:

> “Hey Spring, this field is the primary key.”  
> “This class maps to a table.”  
> “This column cannot be null.”

Here’s what that looks like:

```java
@Entity                  // "Hey Spring, this class maps to a database table"
@Table(name = "country") // Explicitly name the table (defaults to class name)
@Data                    // Lombok: generates getters, setters, equals, hashCode, toString
@NoArgsConstructor       // Lombok: no-args constructor (JPA requires this)
@AllArgsConstructor      // Lombok: constructor with ALL fields
@Builder                 // Lombok: Country.builder().name("France").build()
public class Country {

    @Id                  // "This is the primary key"
    @GeneratedValue(strategy = GenerationType.IDENTITY) // Auto-increment
    private Long id;

    @Column(nullable = false)  // Adds a NOT NULL constraint
    private String name;

    private String capitalCity;  // Maps to: capital_city (camelCase → snake_case, automatic)

    private boolean isLandlocked;

    private long population;
}
```
### What just happened?

- `@Entity` tells JPA this class should be stored in the database.
    
- `@Table(name = "country")` links it to our table.
    
- `@Id` marks the primary key.
    
- `@GeneratedValue` lets the database generate IDs automatically.
    
- `@Column(nullable = false)` enforces a constraint at the database level.

We are also using **Lombok** annotations to eliminate the usual 80 lines of getters and setters, constructors etc

---

## Step 2 — The Repository: Your Free Database Remote Control

Now create an interface. Just an interface — no implementation class, no SQL, nothing.

```java
import org.springframework.data.jpa.repository.JpaRepository;

public interface CountryRepository extends JpaRepository<Country, Long> {
    // Yep. That's it for now. Seriously.
}
```

By extending `JpaRepository<Country, Long>` you're saying: _"this repository manages `Country` entities whose primary key is a `Long`."_ Spring sees this at startup and generates a full working implementation automatically — you never write it, you never see it, it just works.

Out of the box you now have all of this for free:

|Method|SQL|
|---|---|
|`findAll()`|`SELECT * FROM country`|
|`findById(1L)`|`SELECT * FROM country WHERE id = 1`|
|`save(country)`|`INSERT` or `UPDATE` (checks if id exists)|
|`deleteById(1L)`|`DELETE FROM country WHERE id = 1`|
|`count()`|`SELECT COUNT(*) FROM country`|
|`existsById(1L)`|`SELECT EXISTS(...)`|

---

## Step 3 — Using It: Inject and Go

Wire the repository into a service using Spring's constructor injection, and you're done.

```java
@Service
@RequiredArgsConstructor  // Lombok: builds a constructor for all final fields = injection
public class CountryService {

    private final CountryRepository countryRepository;

    public List<Country> getAllCountries() {
        return countryRepository.findAll();  // SELECT * FROM country — done!
    }

    public Country addCountry(Country country) {
        return countryRepository.save(country);  // INSERT INTO country...
    }
}
```

When you call `findAll()`, Spring has already wired up Hibernate to run `SELECT * FROM country`, map every row to a `Country` object, and hand you back a `List<Country>`. No `ResultSet`. No `PreparedStatement`. Gone.

---

## Step 4 — Derived Query Methods ✨ The Real Magic

Here's where it gets fun. You can add methods to your repository using **specific naming patterns**, and Spring will parse the method name and generate the SQL for you automatically.

```java
public interface CountryRepository extends JpaRepository<Country, Long> {

    // WHERE name = ?
    Optional<Country> findByName(String name);

    // WHERE is_landlocked = true
    List<Country> findByIsLandlockedTrue();

    // WHERE population > ?
    List<Country> findByPopulationGreaterThan(long population);

    // WHERE population BETWEEN ? AND ?
    List<Country> findByPopulationBetween(long min, long max);

    // WHERE name LIKE '%?%' (case insensitive)
    List<Country> findByNameContainingIgnoreCase(String keyword);

    // WHERE is_landlocked = ? AND population > ?
    List<Country> findByIsLandlockedAndPopulationGreaterThan(boolean landlocked, long pop);

    // ORDER BY population DESC
    List<Country> findAllByOrderByPopulationDesc();
}
```

Spring reads the method name and splits it on keywords like `By`, `And`, `Or`, `GreaterThan`, `Between`, `Containing` — matching each part to a field on your entity. No code required, just a well-named method.

> 🛡️ **Typo safety:** These method names are parsed against your entity fields _at startup_. Misspell a field name and the app refuses to start — catching the bug before any user ever hits it.

---

## Step 5 — Complex Queries: SQL's Back, On Your Terms

When your method name would be something like `findByCapitalCityNotNullAndIsLandlockedFalseAndPopulationGreaterThanOrderByNameAsc` (valid, but awful), you can use `@Query` to write it yourself:

```java
public interface CountryRepository extends JpaRepository<Country, Long> {

    // JPQL — uses Java class/field names, not table/column names
    @Query("SELECT c FROM Country c WHERE c.population > :minPop AND c.isLandlocked = false")
    List<Country> findLargeCoastalCountries(@Param("minPop") long minPopulation);

    // Native SQL — use nativeQuery = true for database-specific features
    @Query(
        value = "SELECT * FROM country WHERE capital_city IS NOT NULL ORDER BY population DESC LIMIT 10",
        nativeQuery = true
    )
    List<Country> findTop10PopulatedWithCapital();
}
```

JPQL (the default) uses your Java class and field names rather than raw table columns, so it stays database-agnostic. Native queries give you full SQL power when you need something specific to your database.

---

## What Spring Did Behind the Scenes

When your app boots up, a lot of invisible work happens so everything is ready the moment the first request arrives:

1. **Finds your repository interface** — Spring spots that `CountryRepository` extends `JpaRepository`.
2. **Builds a real implementation for you** — generates all the working code behind your interface so `findAll()`, `save()` and friends actually do something.
3. **Parses your custom method names** — `findByPopulationGreaterThan` gets broken apart and turned into a query automatically.
4. **Compiles your `@Query` annotations** — manually written queries are validated and prepared upfront, not at request time.
5. **Hibernate translates to your database** — it speaks MySQL, PostgreSQL, H2, and more. You write Java; Hibernate figures out the exact SQL dialect.

> **The deal:** Spring writes the boring code. Hibernate speaks to the database. You just describe what you want. Zero SQL for the common 80% of queries — and a clean escape hatch to raw SQL when you genuinely need it.

---

## Good to Know — Typo Safety in Action

When Spring parses your derived method names at startup, it checks every word against the fields on your entity. So if you accidentally wrote this:

```java
List<Country> findByNaame(String name); // typo — "Naame" doesn't exist on Country
```

The app refuses to start with:

```
No property 'naame' found for type 'Country'
```

Compare that to a typo buried in a raw SQL string — that would only explode when someone actually hits that endpoint, possibly in production. Spring catches it at the earliest possible moment.

---

## Good to Know — Use Optional for Single Results

When fetching a single country that might not exist, return `Optional<Country>` instead of `Country` — otherwise you risk a `NullPointerException` if nothing is found.

```java
Optional<Country> findByName(String name);
```

Then in your service, handle both cases cleanly:

```java
Optional<Country> result = countryRepository.findByName("France");

if (result.isPresent()) {
    System.out.println("Found: " + result.get().getCapitalCity()); // Paris
} else {
    System.out.println("Country not found");
}
```

`findById()` already returns `Optional` out of the box — your custom single-result methods should too.

---

## Good to Know — Pagination is Built In

If your country table had 195 rows, you wouldn't want to return them all in one go. `JpaRepository` supports pagination out of the box with `Pageable` — no extra setup needed.

```java
// Get page 0, 10 countries per page, sorted by name
Pageable pageable = PageRequest.of(0, 10, Sort.by("name"));
Page<Country> page = countryRepository.findAll(pageable);

page.getContent();        // the 10 countries on this page
page.getTotalElements();  // 195 — total countries in the database
page.getTotalPages();     // 20 — how many pages exist
page.hasNext();           // true
```

Your derived query methods can accept `Pageable` too:

```java
Page<Country> findByIsLandlockedFalse(Pageable pageable);
```

That's all you need to add paging to any query — pass in a `PageRequest`, get back a `Page`.

---

## Good to Know — Relationships (A Gentle Introduction)

Real databases have related tables. In our case, a country has many cities. You can model this directly on your entities using `@OneToMany` and `@ManyToOne`.

```java
@Entity
public class City {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private long population;

    @ManyToOne                        // Many cities belong to one country
    @JoinColumn(name = "country_id")  // The foreign key column in the city table
    private Country country;
}
```

And on the `Country` side:

```java
@OneToMany(mappedBy = "country")  // One country has many cities
private List<City> cities;
```

Now you can do things like `country.getCities()` and Spring handles the join query for you. Relationships are a big topic with their own gotchas — lazy loading in particular is worth reading up on when you get there — but this is the shape of it.

> 🗺️ **The big picture:** Entity relationships let you navigate your data as a graph of Java objects rather than writing joins by hand. Start simple, add relationships only when you need them.# Query Your Database Without Writing SQL
