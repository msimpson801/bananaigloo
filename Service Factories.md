# Using a Service Factory in Spring Boot (Factory Method Pattern)

In Spring Boot applications, it’s common to have multiple services that perform similar actions but behave differently based on a specific “type.” A typical (but messy) solution is to use long `if-else` or `switch` blocks directly inside controllers.

A better approach is to use the **Factory Method Pattern**.

To understand this pattern  we’ll walk through an example that chooses between a `DogService` and a `CatService`.

---

## 1. Create a Common Interface

First, define a shared contract that all pet services must follow. This allows the rest of your application to interact with them in a consistent way, regardless of the concrete implementation.

```java
public interface PetService {     
String getGreeting();
}
```

Any service returned by the factory will implement this interface.

---

## 2. Implement the Services

Next, create concrete implementations for each pet type. Annotate them with `@Service` so Spring can manage them as beans.

```java
@Service  
public class DogService implements PetService{  
    @Override  
    public String getGreeting() {  
        return "I am dog, woof";  
    }  
}

@Service  
public class CatService implements PetService{  
    @Override  
    public String getGreeting() {  
        return "I am a cat, miaow";  
    }  
}
```


Each service contains only the logic specific to that pet.

---

## 3. Build the Factory

The factory is responsible for selecting the correct service at runtime.

Instead of using the `new` keyword (which would bypass Spring’s dependency injection), we inject the existing service beans into the factory.

```Java
@Component  
@RequiredArgsConstructor  
public class PetFactory {  
    private final DogService dogService;  
    private final CatService catService;  
  
    public PetService getService(String petType) {  
        switch (petType.toLowerCase()) {  
            case "dog":  
                return dogService;  
            case "cat":  
                return catService;  
            default:  
                throw new IllegalArgumentException("Unknown pet type: " + petType);  
        }  
    }  
}
```

### Key Points

- The factory returns the **interface type (`PetService`)**, not a concrete class
    
- The controller doesn’t need to know _how_ the decision is made
    
- Adding a new pet only requires:
    
    - A new service
        
    - One new case in the factory
        

---

## 4. Use the Factory in a Controller

Finally, inject the factory into your controller. The controller simply asks for a service and uses it—no conditionals needed.
```Java
@RestController  
@RequestMapping("/pets")  
@RequiredArgsConstructor  
public class PetController {  
    private final PetFactory petFactory;  
  
  
    @GetMapping("/{type}")  
    public String greetPet(@PathVariable String type) {  
        PetService service = petFactory.getService(type);  
        return service.getGreeting();  
    }  
}

```

Now, requests like:

- `GET /pets/dog`
    
- `GET /pets/cat`
    

Automatically use the correct service.