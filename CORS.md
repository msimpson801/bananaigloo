# Why Your Frontend Can’t Call the API: A Story Featuring Everyone’s Least Favourite Corr, Jim Corr

**Or: The Tale of the Overprotective Browser (Featuring The Corrs, U2, and The Cranberries)**

## The Mystery That Haunts Every Developer

You've built your Spring Boot API. It's beautiful. It works. Your music endpoint serves up band info like a dream.

You test it in Postman.

✅ `GET /api/music/band-info/thecorrs` — perfect JSON  
✅ `GET /api/music/band-info/u2` — amazing 
✅ `GET /api/music/band-info/thecranberries` — works flawlessly

Then you fire up your React app and try the exact same requests—

💥 **ERROR** 
💥 **Blocked by CORS policy** 
💥 **Red text everywhere**

Suddenly you’re googling _“why does Postman work but browser hates me”_.

But here’s the weird part…

👉 **Postman still works perfectly when you hit your API.**

What is going on?!

## What Is This CORS Thing?

**Spoiler: It has absolutely nothing to do with Jim Corr or the Corr family.**

The problem isn't which band you're requesting or whether you spelled "thecranberries" correctly.

**It's about CORS** — Cross-Origin Resource Sharing — and it's blocking every single request from your React app.

Here's the thing: **Postman doesn't care about CORS.**

Postman is pure chaos energy. It just sends HTTP requests. No questions. No rules. No judgment. Whether it's The Corrs or U2, Postman says "grand, off you go."

Your **browser**?

Your browser is a nightclub bouncer with serious trust issues.

When your React app (living at `http://localhost:3000`) tries to call your Spring Boot API (chilling at `http://localhost:8080`), the browser goes:

🚨  
**"WOAH WOAH WOAH.**  
**Different origin.**  
**Different port.**  
**I don't know you like that."**  
🚨

And it shuts down ALL your requests:

- The Corrs? Blocked.
- U2? Blocked.
- The Cranberries? You guessed it—blocked.

CORS is a browser security feature that exists to stop random websites from quietly stealing data on your behalf. Noble? Yes. Important? Absolutely. Extremely annoying when it blocks your entire Irish music API? Also yes.
## Why Every Band Gets Blocked (But Postman Works Fine)

Here's what's happening:

**In Postman:**

```
Request: GET http://localhost:8080/api/music/band-info/thecorrs
Response: 200 OK ✅

Request: GET http://localhost:8080/api/music/band-info/u2
Response: 200 OK ✅

Request: GET http://localhost:8080/api/music/band-info/thecranberries
Response: 200 OK ✅
```

**In Your React App:**

```
Request: GET http://localhost:8080/api/music/band-info/thecorrs
Browser: "Nope. Different origin. BLOCKED." ❌

Request: GET http://localhost:8080/api/music/band-info/u2
Browser: "Still nope. BLOCKED." ❌

Request: GET http://localhost:8080/api/music/band-info/thecranberries
Browser: "Are you not listening? BLOCKED." ❌
```

The problem isn't **what** you're requesting.

The problem is **where** you're requesting it from.

## Why Postman Gets a Free Pass

**Postman doesn't run in a browser.**

Which means:

- No Same-Origin Policy
- No CORS checks
- No bouncer at the door
- 

Postman is basically saying: _"I understand the risks. Send the requests."_

## Why Your Browser Does Not Trust You

Browsers enforce CORS because otherwise:

- Any dodgy website could try to call sensitive APIs
- Cookies and auth tokens would be sent automatically
- Chaos would reign across the internet

So the browser treats **every** cross-origin request as suspicious.

It doesn't matter if you're requesting:

- The Corrs
- U2
- The Cranberries
- Thin Lizzy
- Literally any endpoint

If it's a different origin, it's blocked. Full stop.

## The Fix: Teaching Your API Some Manners

You need to tell your Spring Boot API:

_"Hey. That frontend at localhost:3000?  
It's sound.  
Let it fetch the  bands."_

### Option 1: The Quick Fix (aka "Just Let Me Demo This")

Add `@CrossOrigin` to your controller:

java

```java
@RestController
@RequestMapping("/api/music")
@CrossOrigin(origins = "http://localhost:3000") // Approved guest
public class MusicController {

    @GetMapping("/band-info/{bandName}")
    public BandInfo getBandInfo(@PathVariable String bandName) {
        // This will now work for thecorrs, u2, thecranberries, etc.
        return musicService.getBandInfo(bandName);
    }

}
```

**Pros:**

- Takes 10 seconds
- Instantly works for ALL your endpoints

**Cons:**

- You'll copy-paste this on every controller
- Becomes messy fast
- Future You will sigh heavily

### Option 2: The "I Know What I'm Doing" Global Config

Do it once. Fix it for every band. Every endpoint. Forever.

java

```java
@Configuration
public class CorsConfig {

    @Bean
    public WebMvcConfigurer corsConfigurer() {
        return new WebMvcConfigurer() {
            @Override
            public void addCorsMappings(CorsRegistry registry) {
                registry.addMapping("/**")  // ALL endpoints - every band, every route
                        .allowedOrigins("http://localhost:3000")
                        .allowedMethods("GET", "POST", "PUT", "DELETE", "OPTIONS")
                        .allowedHeaders("*")
                        .allowCredentials(true);
            }
        };
    }
}
```

**Pros:**

- One config file fixes EVERYTHING
- Any future controllers you add to your music API will work
- Clean, professional, scalable

**Cons:**

- Requires creating a new file  

## Words of wisdom

### 1. Don't Use `allowedOrigins("*")` in Production

Yes, it works.

It also tells the internet: _"Anyone can access my entire Irish music database."_

That's grand for testing.  
Less grand when your boss finds out.

### 2. Read the Console Error

That angry red message is actually useful.

Look for:

- `blocked by CORS policy`
- `Access-Control-Allow-Origin`
- `preflight request`

Your browser is leaving clues. Follow them.

### 3. It's NOT the Band's Fault

If The Corrs are blocked, U2 is blocked, and The Cranberries are blocked...

**It's not about the bands.**

It's about the origin.

Stop trying different endpoints and fix the CORS config once.

## The Reality Check

CORS isn't broken.  
Your API isn't broken.  
The Corrs didn't do anything wrong.  
Jim Corr is innocent (at least of this).  
**You are not bad at your job.**

Your browser is doing its job — protecting users from sketchy cross-origin requests.

Postman just doesn't live in that world.

## Try It Again

After adding CORS config, **restart Spring Boot** and test all three bands from React:

javascript

```javascript
// Test The Corrs
fetch('http://localhost:8080/api/music/band-info/thecorrs')
  .then(res => res.json())
  .then(band => console.log('The Corrs:', band));

// Test U2
fetch('http://localhost:8080/api/music/band-info/u2')
  .then(res => res.json())
  .then(band => console.log('U2:', band));

// Test The Cranberries
fetch('http://localhost:8080/api/music/band-info/thecranberries')
  .then(res => res.json())
  .then(band => console.log('The Cranberries:', band));
```

If all three work:  
🎉 **You've solved CORS for your entire API.**

If they don't:  
😅 **Did you restart Spring Boot? Seriously.**
