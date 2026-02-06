# Bean There, Done That: A Silly Guide to @Configuration and @Bean 🫘

## Part 1: "Spring, please bake me some beans"

Think of `@Configuration` as a recipe book, and `@Bean` as individual recipes.

### First, let's create our bean class:

java

```java
public class Bean {
    private String name;
    private String color;
    private String origin;
    
    public Bean(String name, String color, String origin) {
        this.name = name;
        this.color = color;
        this.origin = origin;
    }
    
    public void describe() {
        System.out.println("🫘 " + name + " | Color: " + color + " | Origin: " + origin);
    }
    
    // getters
    public String getName() { return name; }
    public String getColor() { return color; }
    public String getOrigin() { return origin; }
}
```

### Now the Bean Kitchen! 👨‍🍳

java

```java
@Configuration
public class BeanKitchen {

    @Bean
    public Bean bakedBean() {
        return new Bean("Baked Bean", "Orange", "England");
    }

    @Bean
    public Bean kidneyBean() {
        return new Bean("Kidney Bean", "Dark Red", "Peru");
    }

    @Bean
    public Bean pintoBean() {
        return new Bean("Pinto Bean", "Speckled Brown", "Mexico");
    }

    @Bean
    public Bean blackBean() {
        return new Bean("Black Bean", "Black", "Latin America");
    }

    @Bean
    public Bean chickpea() {
        return new Bean("Chickpea", "Beige", "Middle East");
    }
}
```

**What did we just do?**

- Told Spring: "Hey, here's a class full of stuff you should create for me"
- Each `@Bean` method = Spring makes this object once and keeps it around
- Spring now owns these beans. You didn't `new` them. You just… trusted the kitchen.

---

## Part 2: "Fill the bathtub with beans" 🛁🫘

Now let's use those beans somewhere else.

Spring has a fun trick: **If you ask for a `List<Bean>`, Spring will give you ALL the Bean instances it created.**

java

````java
@Component
public class BathTub {

    private final List<Bean> bathTub;

    public BathTub(List<Bean> bathTub) {
        this.bathTub = bathTub;
    }

    public void soak() {
        System.out.println("🛁 Bathtub contains " + bathTub.size() + " beans:");
        bathTub.forEach(Bean::describe);
    }
}
````

Run this and Spring goes:

> "Oh, you want a `List<Bean>`?  
> Cool, here are all the Bean beans I already made."

**Your bathtub is now full of:**
```
🛁 Bathtub contains 5 beans:
🫘 Baked Bean | Color: Orange | Origin: England
🫘 Kidney Bean | Color: Dark Red | Origin: Peru
🫘 Pinto Bean | Color: Speckled Brown | Origin: Mexico
🫘 Black Bean | Color: Black | Origin: Latin America
🫘 Chickpea | Color: Beige | Origin: Middle East
````

**Is this useful?** ❌ No  
**Is it memorable?** ✅ Absolutely

---

## Part 3: "Okay okay... show me something slightly useful"

Let's read static JSON from a file using Jackson's `ObjectMapper`.

### 1️⃣ Create a silly JSON file

`src/main/resources/beans.json`

json

```json
{
  "beanType": "Magic Bean",
  "powerLevel": 9000,
  "special": true
}
```

### 2️⃣ Create a POJO

java

```java
public class MagicBean {
    private String beanType;
    private int powerLevel;
    private boolean special;

    // getters & setters (or use @Data from Lombok)
}
```

### 3️⃣ Configure a bean that reads the JSON

java

```java
@Configuration
public class JsonBeanConfig {

    @Bean
    public MagicBean magicBean(ObjectMapper objectMapper) throws Exception {
        return objectMapper.readValue(
                new ClassPathResource("beans.json").getInputStream(),
                MagicBean.class
        );
    }
}
```

**Spring already provides:**

- `ObjectMapper` ✅
- Resource loading ✅

So you just plug them together like LEGO 🧱

### Use it anywhere:

java

````java
@Component
public class BeanInspector {

    public BeanInspector(MagicBean magicBean) {
        System.out.println("Bean Type: " + magicBean.getBeanType());
        System.out.println("Power Level: " + magicBean.getPowerLevel());
        System.out.println("Is Special: " + magicBean.isSpecial());
    }
}
```

**Output:**
```
Bean Type: Magic Bean
Power Level: 9000
Is Special: true
````

Boom. Static data, zero effort at runtime.

---

## Part 4: "Real world-ish: different WebClients" 🌐

This is where `@Configuration` actually shines. Let's create WebClients for some... _interesting_ APIs.

java

```java
@Configuration
public class WebClientConfig {

    @Bean
    public WebClient footCheeseClient() {
        return WebClient.builder()
                .baseUrl("https://api.footcheese.io")
                .defaultHeader("User-Agent", "StinkyApp/1.0")
                .build();
    }

    @Bean
    public WebClient candyFlossBeardClient() {
        return WebClient.builder()
                .baseUrl("https://api.candyflossbeard.com")
                .defaultHeader("Accept", "application/json")
                .build();
    }

    @Bean
    public WebClient sockPuppetClient() {
        return WebClient.builder()
                .baseUrl("https://api.sockpuppet.net")
                .build();
    }
}
```

### Now use them in different services - Spring knows which is which!

java

```java
@Service
public class FootCheeseService {

    private final WebClient footCheeseClient;

    // Spring sees the parameter name matches the bean method name
    // and injects the right one!
    public FootCheeseService(WebClient footCheeseClient) {
        this.footCheeseClient = footCheeseClient;
    }

    public String getCheeseLevel() {
        return footCheeseClient.get()
            .uri("/smell-intensity")
            .retrieve()
            .bodyToMono(String.class)
            .block();
    }
}
```

java

```java
@Service
public class BeardService {

    private final WebClient candyFlossBeardClient;

    // Different parameter name = different WebClient injected!
    public BeardService(WebClient candyFlossBeardClient) {
        this.candyFlossBeardClient = candyFlossBeardClient;
    }

    public String getBeardSweetness() {
        return candyFlossBeardClient.get()
            .uri("/sweetness-rating")
            .retrieve()
            .bodyToMono(String.class)
            .block();
    }
}
```

**The magic:** Spring matches the parameter name (`footCheeseClient`) to the `@Bean` method name (`footCheeseClient()`). Each service gets its own pre-configured WebClient with the right base URL already set up!

### Alternative: Use @Qualifier when parameter names aren't enough

java

```java
@Service
public class MultiApiService {

    private final WebClient footClient;
    private final WebClient beardClient;

    public MultiApiService(
            @Qualifier("footCheeseClient") WebClient footClient,
            @Qualifier("candyFlossBeardClient") WebClient beardClient) {
        this.footClient = footClient;
        this.beardClient = beardClient;
    }
}
```

**Same type (`WebClient`), different personalities, Spring just knows.**

---

## Part 5: "Stop hardcoding stuff" (@ConfigurationProperties) 📦

This is where Spring goes full responsible adult.

### 1️⃣ YAML config (basic, calm, not scary)

`application.yml`

yaml

```yaml
app:
  bathtub:
    size: 5
    material: "porcelain with rubber duckies"
    temperature: "lukewarm"
    features:
      - "Jacuzzi jets"
      - "Bean holder"
      - "Self-cleaning"
      - "Disco lights"
```

> **Real world note:** In actual apps, this is where you'd put boring stuff like:
> 
> - Database URLs (prod vs dev)
> - API endpoints that change per environment
> - Feature flags
> - Timeouts and retry configs
> 
> But bathtubs are way more fun. 🛁

### 2️⃣ Configuration properties class

java

```java
@Configuration
@ConfigurationProperties(prefix = "app.bathtub")
public class BathTubProperties {

    private int size;
    private String material;
    private String temperature;
    private List<String> features;

    // getters & setters (Spring needs them!)
}
```

### 3️⃣ Use it anywhere

java

````java
@Component
public class FancyBathTub {

    public FancyBathTub(BathTubProperties props) {
        System.out.println("🛁 Bathtub Configuration:");
        System.out.println("   Size: " + props.getSize() + " beans");
        System.out.println("   Material: " + props.getMaterial());
        System.out.println("   Temperature: " + props.getTemperature());
        System.out.println("   Features:");
        props.getFeatures().forEach(f -> System.out.println("     - " + f));
    }
}
```

**Output:**
```
🛁 Bathtub Configuration:
   Size: 5 beans
   Material: porcelain with rubber duckies
   Temperature: lukewarm
   Features:
     - Jacuzzi jets
     - Bean holder
     - Self-cleaning
     - Disco lights
````

No `@Value`, no string soup, no sadness. ✨

You can swap out your entire config by just changing the YAML file. Perfect for different environments (dev/staging/prod) without touching code.