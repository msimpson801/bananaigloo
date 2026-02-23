# A Beginner's Guide to SQL Joins (Winter Olympics Edition)

Picture this. It's the Winter Olympics. Snow is falling, the crowd is roaring, Bjorn Bjornsson is hurtling down a ski jump at speeds that would make most of us cry. And somewhere in a server room, a database is quietly keeping track of it all.

That database has tables. Those tables need to talk to each other. And that's exactly what we're going to learn today.

---

## The Country Table

This is a table of every country in the world. All of them. The skiing powerhouses, the tropical paradises, the tiny island nations that have never seen a snowflake. They all get a row.

|country_id|country_name|continent|
|---|---|---|
|NOR|Norway|Europe|
|SWE|Sweden|Europe|
|CAN|Canada|North America|
|CRI|Costa Rica|Central America|
|VEN|Venezuela|South America|

That `country_id` column is the important one. NOR means Norway, CAN means Canada — short, unique, and unambiguous. Every country gets its own code. Remember these codes, because they're about to show up somewhere else.

Costa Rica and Venezuela are perfectly legitimate entries. They exist, they have country codes, everything is in order. They just happen to be warm, sunny places where nobody has ever strapped on a pair of skis and thought "yes, this is my destiny." That detail is going to matter a lot in a moment.

---

## The Athlete Table

Here are our competitors. A fine collection of winter sporting talent.

|athlete_id|athlete_name|sport|country_id|
|---|---|---|---|
|ATH001|Bjorn Bjornsson|Ski Jump|NOR|
|ATH002|Astrid Snowsdottir|Biathlon|NOR|
|ATH003|Lars Skimaster|Cross-Country Skiing|SWE|
|ATH004|Chad Canuck|Ice Hockey|CAN|
|ATH005|Maple McFrosty|Speed Skating|CAN|
|ATH006|Engelbert Humperdink|Freestyle Skiing|XYZ|

Now look at that `country_id` column on the right. Bjorn and Astrid have `NOR` — that points straight back to Norway in the country table. Lars has `SWE`. Chad and Maple have `CAN`.

And then there's Engelbert. Bless him. He's put down `XYZ` as his country. There is no XYZ anywhere in the country table. He's pointing at a country that simply does not exist. Whether this was a typo, a joke, or Engelbert being Engelbert, we may never know. But it's going to cause some very interesting behaviour when we start joining tables.

So here's where we stand. Three awkward situations that our joins are going to have to deal with:

- **Costa Rica and Venezuela** are in the country table, but no athletes have those country codes. No matches waiting for them.
- **Engelbert** is in the athlete table, but his country code XYZ matches nothing in the country table. He's pointing at thin air.

---

## How the Tables Connect

Here's the core idea before we get into join types. That `country_id` column in the athlete table is the bridge between the two tables. When you run a join, SQL walks across that bridge — it looks at each athlete's `country_id`, finds the row in the country table with the matching code, and stitches the two rows together into one.

```
athlete.country_id  ──────►  country.country_id
      NOR           ──────►  Norway ✅
      NOR           ──────►  Norway ✅
      SWE           ──────►  Sweden ✅
      CAN           ──────►  Canada ✅
      CAN           ──────►  Canada ✅
      XYZ           ──────►  ??? doesn't exist ❌
```

And from the country side, CRI and VEN are sitting there with no athletes pointing at them at all. Now let's see what each join does with this situation.

---

## INNER JOIN — Only Perfect Matches

sql

```sql
SELECT country.country_name, athlete.athlete_name, athlete.sport
FROM athlete
INNER JOIN country ON athlete.country_id = country.country_id;
```

Inner join is ruthless. It only returns rows where both sides have a match. Walk across the bridge — if there's something on the other side, great. If not, the whole row gets thrown out without a second thought.

|country_name|athlete_name|sport|
|---|---|---|
|Norway|Bjorn Bjornsson|Ski Jump|
|Norway|Astrid Snowsdottir|Biathlon|
|Sweden|Lars Skimaster|Cross-Country Skiing|
|Canada|Chad Canuck|Ice Hockey|
|Canada|Maple McFrosty|Speed Skating|

**Costa Rica and Venezuela?** Gone. Nothing in the athlete table points at CRI or VEN, so they never appear.

**Engelbert?** Gone. His XYZ matches nothing in the country table, so he gets dropped entirely.

Clean, tidy results — but you've lost information. Looking at this output alone, you'd have no idea that Costa Rica and Venezuela were even in the database. And you'd have no idea that Engelbert is lurking somewhere with a country code that doesn't exist.

---

## LEFT JOIN — Every Country, Matched or Not

sql

```sql
SELECT country.country_name, athlete.athlete_name, athlete.sport
FROM country
LEFT JOIN athlete ON athlete.country_id = country.country_id;
```

Left join says: _whatever table I wrote first — the left table — keep every single row from it, no matter what._ Here the country table is first. So every country in our database comes back. The athlete table fills in where it can, and where it can't, it just writes NULL.

|country_name|athlete_name|sport|
|---|---|---|
|Norway|Bjorn Bjornsson|Ski Jump|
|Norway|Astrid Snowsdottir|Biathlon|
|Sweden|Lars Skimaster|Cross-Country Skiing|
|Canada|Chad Canuck|Ice Hockey|
|Canada|Maple McFrosty|Speed Skating|
|Costa Rica|NULL|NULL|
|Venezuela|NULL|NULL|

Costa Rica and Venezuela are back, sitting at the bottom with their NULLs. And those NULLs are genuinely useful. If you're building a report on Winter Olympic participation by country, you'd want to know which countries exist in the world but sent nobody. That's real information, not an error.

**Engelbert?** Still gone. He lives in the athlete table, which is on the right side here. Left join only protects the left table. His XYZ has no match, and the left join has no interest in saving him.

---

## RIGHT JOIN — Every Athlete, Matched or Not

sql

```sql
SELECT country.country_name, athlete.athlete_name, athlete.sport
FROM country
RIGHT JOIN athlete ON athlete.country_id = country.country_id;
```

Flip it around. Now the athlete table is the protected one — every athlete comes back no matter what. The country table fills in where it can.

|country_name|athlete_name|sport|
|---|---|---|
|Norway|Bjorn Bjornsson|Ski Jump|
|Norway|Astrid Snowsdottir|Biathlon|
|Sweden|Lars Skimaster|Cross-Country Skiing|
|Canada|Chad Canuck|Ice Hockey|
|Canada|Maple McFrosty|Speed Skating|
|NULL|Engelbert Humperdink|Freestyle Skiing|

Hello Engelbert! He's back. His XYZ still matches nothing, so the country columns are NULL — but the right join kept him in the results anyway. Those NULLs are a flashing warning light saying _"this athlete has bad data, someone needs to fix this."_

**Costa Rica and Venezuela?** Gone again. They live in the country table, which is on the left side here, and the right join doesn't protect the left.

---

## FULL JOIN — Nobody Gets Left Behind

sql

```sql
SELECT country.country_name, athlete.athlete_name, athlete.sport
FROM country
FULL JOIN athlete ON athlete.country_id = country.country_id;
```

Full join is the most generous option of all. Every row from both tables comes back. Matches get joined together nicely. Anything without a match gets NULLs on the missing side. Nobody gets left out in the cold — which, given this is the Winter Olympics, is really saying something.

|country_name|athlete_name|sport|
|---|---|---|
|Norway|Bjorn Bjornsson|Ski Jump|
|Norway|Astrid Snowsdottir|Biathlon|
|Sweden|Lars Skimaster|Cross-Country Skiing|
|Canada|Chad Canuck|Ice Hockey|
|Canada|Maple McFrosty|Speed Skating|
|Costa Rica|NULL|NULL|
|Venezuela|NULL|NULL|
|NULL|Engelbert Humperdink|Freestyle Skiing|

Everyone is accounted for. The matched athletes have their countries. Costa Rica and Venezuela are there with their lonely NULLs. And Engelbert is at the bottom, his XYZ exposed, NULLs blaring like a klaxon.

This is the join you reach for when you want a full audit — not just the clean matching data, but every gap and problem hiding in either table.

---

## The Scorecard

Here's how each join handled our three awkward situations:

|                | Norway, Sweden, Canada athletes | Costa Rica & Venezuela | Engelbert (XYZ) |
| -------------- | ------------------------------- | ---------------------- | --------------- |
| **INNER JOIN** | ✅ Included                      | ❌ Dropped              | ❌ Dropped       |
| **LEFT JOIN**  | ✅ Included                      | ✅ NULL rows            | ❌ Dropped       |
| **RIGHT JOIN** | ✅ Included                      | ❌ Dropped              | ✅ NULL rows     |
| **FULL JOIN**  | ✅ Included                      | ✅ NULL rows            | ✅ NULL rows     |

## The One Sentence Version

**INNER JOIN** — only show me rows where both tables have a match.

**LEFT JOIN** — show me everything from the first table, and match the second table where possible.

**RIGHT JOIN** — show me everything from the second table, and match the first table where possible.

**FULL JOIN** — show me absolutely everything from both tables, matches and all.