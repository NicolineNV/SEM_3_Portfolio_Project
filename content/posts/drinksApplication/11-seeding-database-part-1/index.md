---
title: Entry 11 - Seeding the Database from TheCocktailDB API - Part 1
date: 2026-10-14
showAuthor: true
series: ["Drinks Application"]
series_order: 11
---

# Seeding the Database from TheCocktailDB - Part 1

**Link to the external API: [TheCocktailDB](https://www.thecocktaildb.com/).**

This post documents how the database was filled with real cocktail recipes: where the data comes from, how it is transformed to fit my data model, and why the process is built the way it is. It is the part of the project that took the most effort to get right, so I will describe the decisions in detail, including the alternatives I rejected and the bugs I hit. 

I chose to tackle this part of the project as late as I did because I believed it was important to ensure that my database, entities, DAO layer and most of the service layer were in place before I started the data seeding. I stand by this decision, but looking back I should have examined the data more before building so much of the project architecture. I believe that would have saved me from many of the problems I encountered, for example the fact that 204 of the fetched drinks were rejected, many of them because they had more than one ingredient of the same type. So this is a lesson I will take with me in the future.

Because this part is as complicated and big as it is, I have chosen to split the documentation up into two parts, to avoid having one huge entry.

## The problem

My application recommends a cocktail based on a quiz of sorts. To do that, the database needs more than just a recipe name. Every cocktail points to **one** spirit, liqueur, mixer and syrup (all optional) and an optional garnish. Every spirit, liqueur, mixer and syrup carries **flavor intensities** from 1 to 5 for five dimensions: sweet, sour, bitter, salty and spicy. The quiz sums these intensities per cocktail and compares the result with what the user answered. As I have covered multiple times throughout this series, this is the core idea of my project. It is what distinguishes this project from so many other drink recipe applications.

TheCocktailDB is a free public API with 637 recipes (as stated on their website), which is why I chose this API in the first place. After all, it should be perfect for my project, and it mostly is, but its data does not fit my model in three major ways:

1. **Free-text ingredients.** A drink has up to 15 ingredient fields (`strIngredient1` … `strIngredient15`) with free text such as `Light rum`, `Bacardi Limon`, `Kahlua` and `Fresh lime juice`. The same kind of ingredient appears under many different names.
2. **No categories.** The API does not say whether an ingredient is a spirit, a liqueur, a mixer or a syrup. My schema needs exactly that. Of course this is a structural choice I made, and I knew from the start that the API would not necessarily have these categories, so I was prepared for this.
3. **No flavor data at all.** Intensities do not exist in the API. They had to be added by me (with help from an AI assistant, see Part 2). This was another problem I knew I would have to tackle.

So seeding was not a simple copy, unfortunately. It is a **transformation** from the API's loose data into my strict relational model. The rest of this post is about how that transformation was made possible.

## Alternatives I considered

| Approach | Why I did not choose it |
|---|---|
| **Call the API live when the app runs** | The app would depend on an external service being up and fast. The quiz needs flavor profiles, which the API does not have, so I would still need my own data. The queries in the quiz (filter by family and spirit, then score) are also much easier against my own tables than against an API. |
| **Write all recipes by hand as an SQL script** | Full control, but far too slow for hundreds of recipes. It would also hide how the data was produced, which is not ideal for documentation. |
| **Import everything first, then assign intensities afterwards** | This was my first idea, and what I thought I was going to do. The problem with this method is that I would import drinks that break my one-ingredient-per-type rule, and then have to repair them manually in the database. I preferred to decide the rules *before* importing so the database never contains data that violates my model. Of course I could have made the rules fit the data better, but then I would have been forced to change a lot of the structure I had already built. |

## What I chose

| Approach | Pros and cons |
|---|---|
| **Fetch once, classify with my own files, then import with a Java program** | This way was heavier and required many extra steps to transform the data, unfortunately. However, the positive side of this process is that it is repeatable, and every step can be inspected: the raw JSON is on disk, the classification is in readable CSV files, and rejected drinks are written to a file instead of disappearing. I can easily go through the rejected data and make the necessary changes to the system so that it can incorporate the rejected data later. |

The key idea in the chosen approach is that **the seed is a one-time offline process, not a runtime feature.** The finished application only ever talks to its own database. If the data in TheCocktailDB changes, it has no effect on my database, and if I (as admin) need to add or change data, I can do that without major consequences for the seeded data. This holds as long as I do not rerun `schema.sql`, because that would delete **all** data from the database, but that is a whole other problem in itself.

## The pipeline at a glance

```
TheCocktailDB API
      |  
      |  (1) CocktailDBFetcher  - 26 HTTP requests, in parallel (threads)
      |  
      v
database/seed/raw/a.json ... z.json <- raw cache, never edited by hand
      |
      |  (2) IngredientAnalyser - counts every distinct ingredient name
      |  
      v
database/seed/ingredient-candidates.csv <- 293 distinct names
      |
      |  (3) Classify them into two CSV files (drafted with AI assistance and reviewed by me)
      |
      v
database/seed/ingredient-aliases.csv <- raw name -> canonical name
database/seed/canonical-ingredients.csv <- canonical name -> type + 5 intensities
      |
      |  (4) CocktailImporter - applies the rules, writes to PostgreSQL
      |
      v
PostgreSQL (222 cocktails) and database/seed/skipped-cocktails.json (204 drinks)
```

Each stage saves its result as a **file on disk**, and the next stage starts by reading that file. The alternative would be to keep everything in memory and pass Java objects from one stage to the next, but then the data would be lost every time the program stops. That is deliberate. Because the files stay on disk, I can rerun stage 4 as many times as I like without calling the API again, and if something goes wrong I can open the file between two stages and see exactly what went in.

All files live under `database/seed/`, next to `schema.sql`, because they are data and not source code. The Java classes live in the package `app.seed`.

## Stage 1: Fetching the data (`CocktailDBFetcher`)

### What it does

TheCocktailDB does not have an endpoint that returns all drinks at once, so my first challenge was to import as many cocktails as possible using the other endpoints available. The one I found most useful was searching by first letter (`search.php?f=a`). The fetcher requests all 26 letters and saves each response unchanged as raw JSON in `database/seed/raw/<letter>.json`.

### Why by letter

I needed a way to get *all* recipes, not one by name. The letter endpoint returns the full recipe (ingredients, measures, instructions, category) in one call, so 26 requests were meant to cover the whole database. The alternative (listing by category and then looking up every drink by id) would need hundreds of requests, but I was prepared to use that method if searching by letter did not work.

### Why threads

In this third semester we have learned a lot of new, useful techniques and tools for developing our API, and one of them is concurrency (threading). I therefore strive to incorporate each tool at least once, where it makes sense, and this is a place where threads make real sense. Each request spends almost all of its time **waiting for the network**, not computing, so running several at once speeds up the process considerably.

The implementation uses:
- `HttpClient` from the Java standard library, so no extra dependency
- an `ExecutorService` with a **fixed pool of 4 threads**
- one task per letter, which returns a `Future`, and the main thread collects the results

The core of it looks like this:

```java
// Inside fetchAll()
ExecutorService executor = Executors.newFixedThreadPool(THREAD_COUNT);
try {
    List<Future<String>> futures = new ArrayList<>();
    for (char letter = 'a'; letter <= 'z'; letter++) {
        final char current = letter;
        futures.add(executor.submit(() -> fetchLetter(current)));
    }

    for (Future<String> future : futures) {
        try {
            logger.info(future.get());
        } catch (ExecutionException e) {
            logger.error("Fetch failed: {}", e.getCause().getMessage(), e.getCause());
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            logger.error("Fetching was interrupted");
            return;
        }
    }
} finally {
    executor.shutdown();
}
// The rest of the code
```

A `Future` is a receipt for work that is running on another thread and may not be finished yet. `executor.submit(...)` returns it immediately, and `future.get()` waits until the task is done and gives me its result, or throws an `ExecutionException` if the task failed.

**Why a `Future` and not a plain `Thread`?** A `Thread` runs a `Runnable`, and `run()` returns `void`, so it cannot hand back a result, and an exception inside it disappears into that thread instead of reaching the code that started it. With a `Future` I get both the result (here a status line such as `m: 25 drinks saved`) and the failure, and an error for one letter does not stop the other 25. Other options exist, such as `CompletableFuture` or parallel streams, but they are more advanced, and parallel streams do not let me choose the number of threads, which I wanted for a public API. `ExecutorService` with `Future` is the most direct way to say "do these independent jobs in parallel, then collect the answers".

I chose 4 threads because it is enough to give a clear speedup while staying polite to a free public API. Firing 26 requests at once would be a bit over the top and not necessary at all.

### Why not threads in the database layer

I also considered using threads when saving to the database, and decided against it. A JPA `EntityManager` is **not thread-safe**; sharing one between threads gives unpredictable errors. Each thread would need its own `EntityManager` and transaction, and the unique names (a spirit that appears in many drinks) would cause races between threads that both try to create the same row. The network part is slow and independent, so it benefits from threads. The database part is fast and has shared state, so it does not. The import takes about three seconds single-threaded, so there was nothing to gain.

### Caching and repeatability
- If a letter's file already exists, the fetcher skips it.
- It writes a file **only after the response has been parsed successfully**, so a broken or half-finished response never ends up in the cache.

That makes the fetch safe to rerun after a failure, and it means I normally fetch only once.

### Result

The fetch returned 426 drinks in total. Letters `u` and `x` returned nothing. Nine letters returned exactly 25 drinks, which indicates a cap on the API's side. However, this is not something I know for sure. It is only interesting to note that the letters that returned 25 drinks did so every time I fetched from the API, and no letter ever returned more than 25 drinks. The API says it holds 637 drinks in total, but only 426 drinks were fetched, so a lot of the recipes never reach my program. This is of course something to look into later, if time allows.

## Stage 2: Finding out what the data contains (`IngredientAnalyser`)

Before I could decide how to classify the ingredients, I needed to know what was there. `IngredientAnalyser` reads all the raw JSON files, **normalizes** every ingredient name (trim, lowercase, collapse repeated spaces) and counts how often each one occurs. The result is written to `ingredient-candidates.csv`.

The core of the counting is these few lines:

```java
String name = normalize(node.asText());          // "  Light  Rum " -> "light rum"
if (!name.isEmpty()) {
    counts.merge(name, 1, Integer::sum);         // first time: 1, afterwards: +1
}
```

The output file has two columns, `ingredient,count`, sorted with the most common ingredient first.

The result was **293 distinct ingredient names** across the 426 drinks. The number of ingredient *usages* is the sum of the `count` column in `ingredient-candidates.csv`, which is **1,672** (each time a drink mentions an ingredient, it counts once). Without normalization, `Vodka`, `vodka` and `Vodka ` would have counted as three different ingredients.

The reason this step exists is that it turns a vague task (***"classify the ingredients"***) into a finite, measurable one (***"classify these 293 names, starting with the most frequent"***). Because the file is sorted by count, the most common ingredients come first, so I could get good coverage with limited effort. That matters because I have limited time on this project.

## More information in the next entry

That was it for this part of the seeding of my database. I will continue the documentation and explanation in the next entry.
