# 🚀 Your First Spring Boot GET Request: A Beginner's Adventure!

_Welcome to the world of APIs! If you've ever wondered how apps talk to servers, you're in the right place. Let's make this fun!_

---

## 🤔 What Even IS a GET Request?

Imagine you walk into a coffee shop and say: "Can I get a cappuccino, please?"

That's basically a GET request! You're asking for something, and you expect to get it back.

In the web world:

|You Say|Web Equivalent|
|---|---|
|**You**|Your browser or app|
|**Coffee shop**|The server|
|**"Can I get a cappuccino?"**|A GET request|
|**Your cappuccino**|The data you get back|

Easy, right? 😄

---

## 🎯 Let's Build Something!

We're going to build a simple API that returns information about countries. No fancy database setup, no complicated stuff. Just pure, fun coding!

---

## 🏁 Step 1: The "Hello World" Moment

Every coder's journey starts here. Let's create the simplest GET request ever:

```java
@RestController
@RequestMapping("/api")
public class CountryController {

    @GetMapping("/helloworld")
    public String helloWorld() {
        return "Hello World";
    }
}
```

### 🔍 What's happening here?

| Annotation                   | What It Does                                        |
| ---------------------------- | --------------------------------------------------- |
| `@RestController`            | "Hey Spring, this class handles web requests!"      |
| `@RequestMapping("/api")`    | "All my URLs start with /api"                       |
| `@GetMapping("/helloworld")` | "When someone visits /helloworld, run this method!" |

### ✅ Try it out:

Start your app and go to:

```
http://localhost:8081/api/helloworld
```

🎊 **BOOM!** You'll see "Hello World" in your browser!

You just made your first API endpoint! How cool is that?! 🎉

---

## 🌍 Step 2: Let's Get Real Data

Okay, strings are boring. Let's return actual objects!

### 📝 First, create a simple Country class:

```java
public class Country {
    private String name;
    private String capital;
    private String continent;
    private int populationMillions;
    
    // Constructor, getters, setters... (the usual Java stuff)
}
```

### 🔄 Now, let's return a LIST of countries:

```java
@GetMapping("/countries/getall")
public List<Country> getAllCountries() {
    return countries;  // Imagine we have a list of countries
}
```

Visit:

```
http://localhost:8081/api/countries/getall
```

### 📊 You'll see:

```json
[
  {
    "name": "France",
    "capital": "Paris",
    "continent": "Europe",
    "populationMillions": 67
  },
  {
    "name": "Japan",
    "capital": "Tokyo",
    "continent": "Asia",
    "populationMillions": 125
  }
]
```

> **🪄 Magic Alert!** Spring Boot automatically converts your Java objects to JSON. You didn't have to write ANY conversion code. That's the Spring Boot magic! ✨

---

## 🎯 Step 3: Path Parameters - Fill in the Blank!

What if you want info about just ONE specific country?

Here's where **path parameters** come in. Think of them as fill-in-the-blanks in your URL:

```java
@GetMapping("/countries/{countryName}")
public Country getCountryByName(@PathVariable String countryName) {
    return countries.stream()
        .filter(c -> c.getName().equalsIgnoreCase(countryName))
        .findFirst()
        .orElse(null);
}
```

### 🔍 Let's break this down:

```
1. countries.stream()         → Start looking through all countries
2. .filter(c -> ...)          → Keep only countries that match the name
3. .findFirst()               → Grab the first match
4. .orElse(null)              → If nothing found, return null
```

### 🌐 Now you can do this:

| URL                                          | Result                          |
| -------------------------------------------- | ------------------------------- |
| `http://localhost:8081/api/countries/France` | Returns France data ✅           |
| `http://localhost:8081/api/countries/Japan`  | Returns Japan data ✅            |
| `http://localhost:8081/api/countries/Narnia` | Returns null (nothing found!) ❌ |

### 📝 The URL template:

```
Template:     /countries/{countryName}
You type:     /countries/France
Code receives: countryName = "France"
```


> **💡 Note:** Later we'll learn about `ResponseEntity` which lets you return proper HTTP status codes like 404 for "not found" - but for now, we're keeping it simple!

---

## 🔍 Step 4: Query Parameters - The Filter Masters

Path parameters are cool, but what if you want to FILTER your results?

Enter: **Query Parameters!** They're the things after the `?` in URLs.

```java
@GetMapping("/countries/filter")
public List<Country> getCountriesByContinent(@RequestParam String continent) {
    return countries.stream()
        .filter(c -> c.getContinent().equalsIgnoreCase(continent))
        .collect(Collectors.toList());
}
```

### 🌐 Try these URLs:

- `http://localhost:8081/api/countries/filter?continent=Europe`
- `http://localhost:8081/api/countries/filter?continent=Asia`
- `http://localhost:8081/api/countries/filter?continent=Africa`

### 🍕 Real-World Example:

Think about ordering pizza online:

```
Base:         pizzaplace.com/order
With filters: pizzaplace.com/order?size=large&crust=thin&toppings=pepperoni
```

Same concept! The `?` starts the filters, and `&` connects multiple filters.

---

## 🎨 Step 5: Optional Parameters - Choose Your Own Adventure

Sometimes you want filters to be optional. Like, you CAN filter, but you don't HAVE to.

```java
@GetMapping("/countries/search")
public List<Country> searchCountries(
    @RequestParam String continent,                          // Required!
    @RequestParam(required = false) Integer minPopulation) {  // Optional!
    
    return countries.stream()
        .filter(c -> c.getContinent().equalsIgnoreCase(continent))
        .filter(c -> minPopulation == null || c.getPopulation() >= minPopulation)
        .collect(Collectors.toList());
}
```

### ✅ Now you can do these:

|Query|Works?|Why|
|---|---|---|
|`?continent=Europe`|✅|Just filter by continent (required!)|
|`?continent=Europe&minPopulation=50`|✅|Add optional population filter|
|`?continent=Asia&minPopulation=100`|✅|Different continent with population|
|`?minPopulation=100`|❌|Won't work! Continent is required|
|No parameters at all|❌|Won't work! Must provide continent|

### 🔑 The key difference:

|Parameter Type|Code|Required?|
|---|---|---|
|**REQUIRED**|`@RequestParam String continent`|Yes (no `required=false`)|
|**OPTIONAL**|`@RequestParam(required = false) Integer minPopulation`|No (has `required=false`)|

> **💡 Why do this?** We want users to at least specify a continent so they don't accidentally query everything. The population filter is just an extra refinement if they want it!

---

## 🎁 Step 6: Default Values - Making Life Easy

Want to make things even easier for your users? Give them sensible defaults!

```java
@GetMapping("/countries/sorted-simple")
public List<Country> getSortedCountriesSimple(
    @RequestParam(defaultValue = "asc") String order) {
    
    List<Country> sorted = new ArrayList<>(countries);
    
    if (order.equalsIgnoreCase("desc")) {
        sorted.sort((a, b) -> b.getName().compareTo(a.getName()));
    } else {
        sorted.sort((a, b) -> a.getName().compareTo(b.getName()));
    }
    
    return sorted;
}
```

### 🌐 What happens:

|URL|Result|
|---|---|
|`/countries/sorted-simple`|Gets **ascending** order (A to Z) - the default!|
|`/countries/sorted-simple?order=desc`|Gets **descending** order (Z to A)|
|`/countries/sorted-simple?order=asc`|Gets **ascending** order explicitly|

Users get good defaults without having to type everything! 🎯

### 📊 Example results:

|Sort Order|Countries|
|---|---|
|**Default (asc)**|Argentina, Brazil, Canada, China, Egypt...|
|**With desc**|United States, United Kingdom, Spain, South Korea...|

---

## 🎪 Step 7: Mixing It All Together!

Here's where it gets FUN. You can combine path parameters AND query parameters!

```java
@GetMapping("/countries/{continent}/details")
public List<String> getCountryDetails(
    @PathVariable String continent,
    @RequestParam(defaultValue = "false") boolean includeCapital) {
    
    return countries.stream()
        .filter(c -> c.getContinent().equalsIgnoreCase(continent))
        .map(c -> includeCapital ? 
            c.getName() + " (Capital: " + c.getCapital() + ")" : 
            c.getName())
        .collect(Collectors.toList());
}
```

### 🌐 Try these:

**URL:** `/countries/Europe/details?includeCapital=true`

```json
[
  "France (Capital: Paris)",
  "Germany (Capital: Berlin)",
  "Spain (Capital: Madrid)"
]
```

**URL:** `/countries/Asia/details?includeCapital=false`

```json
[
  "Japan",
  "China",
  "India"
]
```


---

## 🚀 Level Up: More Advanced Stuff!

Now that you've got the basics down, let's explore some more powerful features. Don't worry - we'll keep it fun!

---

## 🚦 Advanced Topic: Response Entity - Taking Control!

So far, when we return `null`, the browser just gets an empty response. But what if we want to be more specific? Like saying "Hey, I looked everywhere but couldn't find that country!"

Enter **ResponseEntity** - it lets you control BOTH the data AND the HTTP status code:

```java
@GetMapping("/countries/{countryName}/status")
public ResponseEntity<Country> getCountryWithStatus(@PathVariable String countryName) {
    Country country = countries.stream()
        .filter(c -> c.getName().equalsIgnoreCase(countryName))
        .findFirst()
        .orElse(null);
    
    if (country != null) {
        return ResponseEntity.ok(country);  // 200 OK
    } else {
        return ResponseEntity.notFound().build();  // 404 Not Found
    }
}
```

### 🔍 What's happening?

|Code|HTTP Status|Meaning|
|---|---|---|
|`ResponseEntity.ok(country)`|200|Returns the country (Success!) ✅|
|`ResponseEntity.notFound().build()`|404|Returns nothing (Not Found!) ❌|

### 🌐 Try it:

|URL|Status Code|Result|
|---|---|---|
|`/countries/France/status`|200 OK|France data returned|
|`/countries/Narnia/status`|404 Not Found|Nothing returned|

### 📊 Common Status Codes:

|Code|Name|Meaning|
|---|---|---|
|**200**|OK|Everything worked! ✅|
|**404**|Not Found|Couldn't find what you asked for 🤷|
|**400**|Bad Request|You asked for something wrong 🚫|
|**500**|Server Error|Oops, we messed up! 😱|

> **💡 Why is this useful?** Your frontend/client can check the status code and show appropriate messages like "Country not found!" instead of just showing nothing.

---

## 🎨 Advanced Topic: Optional Parameters & Validation

Earlier we learned about `required = false` for optional parameters. Now let's see how to validate them!

```java
@GetMapping("/countries/search")
public List<Country> searchCountries(
    @RequestParam(required = false) String continent,
    @RequestParam(required = false) Integer minPopulation,
    @RequestParam(required = false) Integer maxPopulation) {
    
    // Validate: maxPopulation should be greater than minPopulation
    if (minPopulation != null && maxPopulation != null && minPopulation > maxPopulation) {
        throw new IllegalArgumentException("minPopulation can't be greater than maxPopulation!");
    }
    
    return countries.stream()
        .filter(c -> continent == null || c.getContinent().equalsIgnoreCase(continent))
        .filter(c -> minPopulation == null || c.getPopulationMillions() >= minPopulation)
        .filter(c -> maxPopulation == null || c.getPopulationMillions() <= maxPopulation)
        .collect(Collectors.toList());
}
```

### ⚠️ What if someone does this?

```
/countries/search?minPopulation=200&maxPopulation=50
```

The code catches it and throws an error! 🛑

### ✅ Better approach with ResponseEntity:

```java
@GetMapping("/countries/search-validated")
public ResponseEntity<?> searchCountriesValidated(
    @RequestParam(required = false) String continent,
    @RequestParam(required = false) Integer minPopulation,
    @RequestParam(required = false) Integer maxPopulation) {
    
    // Validate parameters
    if (minPopulation != null && maxPopulation != null && minPopulation > maxPopulation) {
        return ResponseEntity
            .badRequest()
            .body("Error: minPopulation can't be greater than maxPopulation!");
    }
    
    List<Country> results = countries.stream()
        .filter(c -> continent == null || c.getContinent().equalsIgnoreCase(continent))
        .filter(c -> minPopulation == null || c.getPopulationMillions() >= minPopulation)
        .filter(c -> maxPopulation == null || c.getPopulationMillions() <= maxPopulation)
        .collect(Collectors.toList());
    
    return ResponseEntity.ok(results);
}
```

Now instead of crashing, you get a nice 400 Bad Request with a helpful error message! 🎯

---

## 📨 Advanced Topic: Request Headers - Secret Messages!

Headers are like secret notes attached to your request. They carry extra information that isn't in the URL.

```java
@GetMapping("/countries/sorted")
public List<Country> getSortedCountries(
    @RequestHeader(value = "Sort-Order", defaultValue = "asc") String sortOrder) {
    
    List<Country> sorted = new ArrayList<>(countries);
    
    if (sortOrder.equalsIgnoreCase("desc")) {
        sorted.sort((a, b) -> b.getName().compareTo(a.getName()));
    } else {
        sorted.sort((a, b) -> a.getName().compareTo(b.getName()));
    }
    
    return sorted;
}
```

### 💻 Test it with cURL:

```bash
# Ascending order (default)
curl http://localhost:8081/api/countries/sorted

# Descending order
curl -H "Sort-Order: desc" http://localhost:8081/api/countries/sorted
```

### 📋 What are headers used for in real life?

|Header|Purpose|
|---|---|
|`Authorization: Bearer token123`|Your login token 🔐|
|`Content-Type: application/json`|Telling the server what format you're sending 📄|
|`Accept: application/json`|Telling the server what format you want back 📥|
|`User-Agent: Mozilla/5.0...`|What browser you're using 🌐|

### 🔄 Multiple Headers Example:

```java
@GetMapping("/countries/custom")
public ResponseEntity<List<Country>> getCustomCountries(
    @RequestHeader(value = "Sort-Order", defaultValue = "asc") String sortOrder,
    @RequestHeader(value = "Include-Population", defaultValue = "true") boolean includePopulation,
    @RequestParam(required = false) String continent) {
    
    // Your logic here...
    return ResponseEntity.ok(results);
}
```

---

## 🧪 Testing Your API

### Option 1: Use Your Browser! 🌐

Easiest way! Just type the URL:

```
http://localhost:8081/api/countries/getall
```

**⚠️ Limitation:** Browsers only do GET requests, and they don't make it easy to add headers or complex parameters.

---

### Option 2: Use cURL (Terminal) 💻

```bash
# Simple GET
curl http://localhost:8081/api/helloworld

# With query parameters
curl "http://localhost:8081/api/countries/filter?continent=Europe"

# With custom headers
curl -H "Sort-Order: desc" http://localhost:8081/api/countries/sorted
```

---

### Option 3: Use Postman 📮

Postman is like a GUI for APIs:

|Step|Action|
|---|---|
|1️⃣|Download Postman (it's free!)|
|2️⃣|Create a new request|
|3️⃣|Select GET|
|4️⃣|Type your URL|
|5️⃣|Add parameters in the "Params" tab|
|6️⃣|Click "Send"|
|7️⃣|See the results!|

---

## 🎓 Quick Reference Cheat Sheet

|What You Want|Annotation|Example URL|When to Use|
|---|---|---|---|
|Basic endpoint|`@GetMapping("/hello")`|`/hello`|Simple responses|
|Get by ID/name|`@PathVariable`|`/users/john`|Specific items|
|Filter results|`@RequestParam`|`/users?age=25`|Filtering/searching|
|Optional filter|`required=false`|`/users?age=25`|Optional filtering|
|Default values|`defaultValue="10"`|`/users?limit=10`|User convenience|
|Custom status|`ResponseEntity`|Any|Control HTTP codes|
|Read headers|`@RequestHeader`|Any|Auth, metadata|

---

## 💡 Common Mistakes (And How to Avoid Them!)

### ❌ Mistake #1: Forgetting the Question Mark

|Wrong|Right|
|---|---|
|`/countries/filtercontinent=Europe`|`/countries/filter?continent=Europe`|

Query parameters ALWAYS start with `?`!

---

### ❌ Mistake #2: Port Confusion

If you see "Connection refused," check:

- ✅ Is your app running?
- ✅ Are you using the right port? (8081 vs 8080)
- ✅ Did you set `server.port=8081` in application.properties?

---

### ❌ Mistake #3: Case Sensitivity

|These Are Different|
|---|
|`/Countries`|
|`/countries`|

URLs are case-sensitive! Always use the exact casing from your `@GetMapping`.

---

### ❌ Mistake #4: Missing @RestController

```java
// ❌ Won't work
public class CountryController { ... }

// ✅ Will work
@RestController
public class CountryController { ... }
```

Without `@RestController`, Spring won't know your class handles web requests!

---

## 🎯 Real-World Example: Putting It All Together!

Let's create one endpoint that uses most of what we learned (but keeps it simple!):

```java
@GetMapping("/continents/{continent}/countries/search")
public List<Country> advancedSearch(
    @PathVariable String continent,
    @RequestParam(required = false) Integer minPopulation,
    @RequestParam(required = false) Integer maxPopulation,
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "10") int size) {
    
    // Filter by continent and population
    List<Country> results = countries.stream()
        .filter(c -> c.getContinent().equalsIgnoreCase(continent))
        .filter(c -> minPopulation == null || c.getPopulationMillions() >= minPopulation)
        .filter(c -> maxPopulation == null || c.getPopulationMillions() <= maxPopulation)
        .collect(Collectors.toList());
    
    // Sort alphabetically
    results.sort((a, b) -> a.getName().compareTo(b.getName()));
    
    // Paginate
    int start = page * size;
    int end = Math.min(start + size, results.size());
    
    if (start >= results.size()) {
        return new ArrayList<>();  // Empty list if page is too high
    }
    
    return results.subList(start, end);
}
```

### 🌐 Example usage:

```
http://localhost:8081/api/continents/Europe/countries/search?minPopulation=50&page=0&size=5
```

### ✅ This endpoint:

- ✅ Uses a path parameter (continent)
- ✅ Uses multiple optional query parameters (population filters)
- ✅ Uses default values (page, size)
- ✅ Filters, sorts, and paginates results
- ✅ Returns an empty list if nothing found

**You just built a professional-grade API endpoint!** 🎉

---

## 🎊 Congratulations!

You just went from zero to GET request hero! 🦸

### 💡 Remember:

- ✅ Start simple (Hello World)
- ✅ Add complexity gradually (path params, then query params)
- ✅ Test as you go (use browser, cURL, or Postman)
- ✅ Don't be afraid to experiment!

**The best way to learn is by doing.** So open up your IDE and start coding! Break things, fix things, and most importantly - have FUN! 🎉

---

## 📚 Quick Glossary

|Term|Definition|
|---|---|
|**API**|Application Programming Interface. How apps talk to each other.|
|**GET Request**|Asking a server for data (like ordering from a menu).|
|**Endpoint**|A specific URL that does something (like `/countries/getall`).|
|**Path Parameter**|A variable in the URL path (`/users/{id}`).|
|**Query Parameter**|Filters after the `?` (`?continent=Europe`).|
|**JSON**|JavaScript Object Notation. How data is formatted and sent.|
|**Status Code**|A number telling you what happened (200=success, 404=not found).|
|**Controller**|A class that handles web requests in Spring Boot.|
