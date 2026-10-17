---
title: Entry 12 - Seeding the Database from TheCocktailDB API - Part 2
date: 2026-10-16
showAuthor: true
series: ["Drinks Application"]
series_order: 12
---

# Seeding the Database from TheCocktailDB - Part 2

**Link to the external API: [TheCocktailDB](https://www.thecocktaildb.com/).**

This is the second part of my documentation for the process of seeding my database with data from TheCocktailDB API. In my last entry, I covered the overall pipeline, some of the problems with the API and its data compared to my model, the classes `CocktailDBFetcher` and `IngredientAnalyser`, and why I chose this procedure instead of other alternatives.

In this part I will cover the classification files, the importer, and the bugs I ran into along the way.

## Stage 3: The classification files

This is the heart of the design, and the part where the reasoning is easiest to forget. It took me some time to get to this process, and I had to lean on the help of an AI to get through this point. I will therefore explain it in detail, and point out the places where the AI did the majority of the work.

For the classification, I have two files, not one:

### `ingredient-aliases.csv` (raw name → canonical name)

```
// An example of a few rows in the file

raw,canonical

light rum,White Rum
kahlua,Coffee Liqueur
absolut vodka,Vodka
fresh lime juice,Lime Juice
// ....
```

This file answers the question: *"The API wrote this. So what do I call it?"* It maps many raw names to one canonical name, so I do not end up with three different ways to write *"Lime Juice"*. The file has 293 rows, one per distinct raw name.

### `canonical-ingredients.csv` (canonical name → type and intensities)

```
// An example of a few rows in the file

canonical,type,sweet,sour,bitter,salty,spicy

White Rum,SPIRIT,2,1,1,1,1
Coffee Liqueur,LIQUEUR,4,1,3,1,1
Lime Juice,MIXER,1,5,1,1,1
Lime,GARNISH,,,,,
Ice,IGNORE,,,,,
// ....
```

This file answers: *"What is this ingredient in my model?"* It has 132 rows, where each canonical ingredient has a **type** (`SPIRIT`, `LIQUEUR`, `MIXER`, `SYRUP`, `GARNISH` or `IGNORE`) and, for the four types that carry flavor, five intensities from 1 to 5.

### Why two files instead of one

The answer is that the relationship is **many-to-one**. Kahlua, Tia Maria and coffee brandy are three different *names* for roughly one thing in my model. If I had put the intensities on the raw names, I would have repeated the same five numbers many times and risked them drifting apart when I changed one. With two files, I make sure that each flavor value is stored **once**, on the canonical ingredient, and the aliases file only handles naming. This is the same idea as database normalization, just applied to the seed data on a file level.

It is also important to note that this is just my way of doing it. I could have chosen to value each raw-named ingredient as its own *"entity"*. After all, some brands of gin taste sweeter and some more bitter than others, and you could therefore argue that treating them as roughly one thing is not true to reality. However, I chose this *restriction* of my model to keep a tighter and more manageable scope for myself. After all, this is *"only"* an educational project.

### The special values

- **`IGNORE`** marks ingredients that should not affect the model at all: ice, water, egg white, salt, bitters and similar. They do not change which spirit or mixer a drink is built on.
- **`GARNISH`** ingredients become the optional garnish. They have no intensities because garnish has no flavor profile in my schema.
- **`UNCLASSIFIED`** appears only in the aliases file, for the 30 names that were deliberately left unclassified (for example `advocaat` and `cream of coconut`). A drink that uses one of these is skipped.

### An honest limitation: the files and intensities are AI-generated 

As I stated earlier, I received help from an **AI assistant** for some of this process. The biggest part was the creation of the two classification files. To save time and avoid unnecessary effort and hours, I chose to let an AI make them for me at my request. I of course told it how to categorize the ingredients and how the files should classify the data, but I did not write the files myself. I did, however, review the result, but overall I chose to stick with what the AI wrote for now.

The intensity numbers (sweet, sour, bitter, salty, spicy, 1 to 5) were also **produced by an AI assistant at my request**, based on general knowledge of how these ingredients usually taste. They are **not measured data**, they were not taken from a taste database, and they are not the result of a taste test. They are therefore subjective and probably not entirely right: one person may find gin more bitter than the file says, another may find a liqueur less sweet.

That is acceptable for this application, given the circumstances: after all, this is only an educational project. However, it must be stated clearly that the quiz is not scientifically accurate because of these estimates. It ranks cocktails *relative to each other*, so what matters most is that the numbers are *consistent* (sweet liqueurs score higher on sweet than dry spirits) and not that each individual number is perfect. And because all intensities live in one file, `canonical-ingredients.csv`, they can be corrected in one place without changing any code.

One thing to be aware of is that many of the choices in the alias file are rough simplifications. When a raw name had no exact match among my canonical ingredients, it was mapped to the closest one, even though they do not taste the same. For example, `aperol` is mapped to `Campari`, although Aperol is noticeably sweeter and less bitter, and `mezcal` is mapped to `Tequila`, which ignores its smoky character. In the same way, `pomegranate juice` becomes `Cranberry Juice`, `strawberry schnapps` becomes `Peach Schnapps` and `coconut milk` becomes `Cream`.

Each of these mappings is a trade-off between keeping the list of canonical ingredients short and keeping each ingredient true to reality. I chose a short list to keep the scope manageable, but it means that some drinks will have a flavor profile that is slightly different from how the real drink tastes. Because the mappings live in one file, they can be refined later by adding a new canonical ingredient and pointing the alias to it.

## Stage 4: The import (`CocktailImporter`)

`CocktailImporter` reads the raw JSON and both CSV files, and for each drink decides: save it, or skip it and say why.

### Step by step for one drink

1. Go through `strIngredient1` to `strIngredient15`. Ignore empty ones.
2. Add each ingredient and its measure to a text list (this ends up in the cocktail's `description`, because my schema has no separate measure columns).
3. Normalize the name, look it up in the aliases file, then look up the canonical entry. If either lookup fails or the alias is `UNCLASSIFIED`, **skip the drink** with reason `UNCLASSIFIED_INGREDIENT`.
4. Depending on the type:
   - `IGNORE`: do nothing.
   - `GARNISH`: remember just the first one (because my model only allows one garnish per drink).
   - anything else: remember it for that type. If the drink already has a ***different*** ingredient of the same type, **skip it** with reason `TOO_MANY_OF_TYPE`.
5. If the drink has no spirit, liqueur, mixer or syrup after this, **skip it** with reason `NO_INGREDIENTS_AFTER_IGNORE`.
6. Otherwise, find or create the family, spirit, liqueur, mixer, syrup and garnish, create the cocktail, and save it.

The code that enforces the one-per-type rule is this `switch`:

```java
private Optional<String> classifyAndSave(EntityManager em, JsonNode drink, String name) {

    // Variables and additional code

    switch (c.type()) {
        case "IGNORE" -> { }                                            // ice, water, bitters ...
        case "GARNISH" -> { if (garnish == null) garnish = c.name(); }  // keep the first garnish
        default -> {
            String existing = byType.putIfAbsent(c.type(), c.name());   // type -> ingredient
            if (existing != null && !existing.equals(c.name())) {
                return skip(name, "TOO_MANY_OF_TYPE", c.type() + ": " + existing + " + " + c.name());
            }
        }
    }

    // Building Cocktail and catching exceptions
}
```

`byType` is a `Map<String, String>` from type to ingredient (for example `SPIRIT -> Vodka`). `putIfAbsent` stores the ingredient if the type is free, and returns the existing one if the type is already taken. A second, *different* ingredient of the same type means the drink does not fit.

One subtle rule: the same canonical ingredient appearing twice is **not** "too many". `Sugar` and `Powdered sugar` both become `Sugar`, so that is fine. Only two *different* ingredients of the same type break the model.

### Why drinks are skipped instead of forced to fit

At the start of this project, when I built my model, it did not occur to me that some recipes would have more than one ingredient of the same type. To incorporate this now, it would be necessary to revisit my model and maybe even reorganize it completely. My model only allows one spirit, one liqueur, one mixer and one syrup per cocktail. Many real cocktails have two or more spirits (for example a Long Island Iced Tea). I could have picked one arbitrarily, but then the stored recipe would be wrong, and the flavor profile would be wrong with it. I preferred a smaller, correct data set over a larger, distorted one. This was a conscious trade-off, and it matches the decision to skip non-fitting recipes rather than remodel the schema (which would have grown the scope).

### Skipped drinks are data, not errors

A skipped drink is an expected outcome, not something that went wrong, so it is **not an exception**. Each skip is recorded as a small object (`name`, `reason`, `detail`) and the whole list is written to `database/seed/skipped-cocktails.json` at the end. Real errors (a file that cannot be read, a database failure) are exceptions. This keeps the two situations separate: the first is part of the normal result, the second is a failure.

The skipped drinks are stored like this:

```JSON
[{
  "name" : "AT&T",
  "reason" : "TOO_MANY_OF_TYPE",
  "detail" : "SPIRIT: Vodka + Gin"
}, {
  "name" : "Dirty Martini",
  "reason" : "UNCLASSIFIED_INGREDIENT",
  "detail" : "Olive Brine"
}, {
  "name" : "Pineapple Gingerale Smoothie",
  "reason" : "NO_INGREDIENTS_AFTER_IGNORE",
  "detail" : ""
}]
```

This is how many drinks fall under each skip reason:

```java
3 "reason" : "NO_INGREDIENTS_AFTER_IGNORE"
177 "reason" : "TOO_MANY_OF_TYPE"
24 "reason" : "UNCLASSIFIED_INGREDIENT"
```

### Find-or-create

Ingredient names are `UNIQUE` in the database, and the same ingredient is used by many cocktails. So for every spirit, liqueur, mixer, syrup, garnish, family and flavor tag, the importer first **looks up by name** and only creates the row if it does not exist. When it creates a spirit, liqueur, mixer or syrup, it also attaches the five flavor tags with the intensities from the canonical file. The `FlavorTag` rows (`Sweet`, `Sour`, `Bitter`, `Salty`, `Spicy`) are created the same way, the first time they are needed.

The generic helper that does this for every entity type:

```java
private <T> T findOrCreate(EntityManager em, Class<T> type, String name, Supplier<T> creator) {
    List<T> found = em.createQuery(
            "SELECT e FROM " + type.getSimpleName() + " e WHERE e.name = :n", type)
            .setParameter("n", name).getResultList();

    if (!found.isEmpty()) return found.get(0);    // already in the database: reuse it
    T created = creator.get();                    // otherwise build a new one ...
    em.persist(created);                          // ... and save it
    return created;
}
```

and a typical call, which creates a spirit with its flavor tags only if it does not exist yet:

```java
if (byType.containsKey("SPIRIT"))
        cocktail.setSpirit(findOrCreate(em, Spirit.class, byType.get("SPIRIT"),
                () -> withTags(new Spirit(byType.get("SPIRIT"), null), em, "SPIRIT")));
```

The tag names are the **English column names from the CSV**, and the frontend can translate them to Danish later on. That way, the database holds stable identifiers and changing a display word never requires a data change.

### One transaction per drink

Each drink is saved in its own transaction. If one drink fails, the drinks before it stay saved, and only that drink is rolled back. This also means the importer can be rerun safely.

### Rerunning (idempotence)

Before saving, the importer checks whether a cocktail with that name already exists. If it does, it is counted as "already existed" and left untouched. The consequence is important: **if I change the classification files, I must reset the database** before importing again, otherwise the old rows remain. I learned this the hard way (described in more detail later in this entry).

### Families

I did not have my own cocktail families yet, so the family name comes from TheCocktailDB's `strCategory` (for example `Cocktail`, `Shot`, `Ordinary Drink`). Drinks marked `Non alcoholic` go into a family called `Mocktails`. Families are created through find-or-create, so I can rename or merge them in the database later without touching the importer.

### Why the importer uses the `EntityManager` directly instead of my DAOs

My DAOs open a short-lived `EntityManager` for each operation. The importer needs **one** `EntityManager` shared across the whole run, so that a spirit created for drink 1 is a *managed* object that drink 2 can reuse, and so it can control one transaction per drink. It also does not go through `CocktailService`, because that service checks things like "is the caller an admin" and validates ids, which makes sense for an HTTP request but not for a trusted offline tool.

The trade-off is real and I want to be honest about it: the importer **bypasses the DAO and service layers** and duplicates the rule "a cocktail needs at least one ingredient". I accept that because the seed is a one-off tool that lives in its own package (`app.seed`) and is not part of the running API.

## What went wrong along the way (and what it taught me)

These are worth including because each one shows how I went about debugging my code.

| Problem | Cause | Lesson |
|---|---|---|
| `skipped-cocktails.json` stayed empty although 204 drinks were skipped | My `run()` method was missing the call to `writeSkipped()` and the final log line. The log showed no `Import done` line, which was the clue. | Read the log for what is *missing*, not only for errors. |
| Mocktails ended up in the wrong family | A typo: the code compared with `"Non alcoholics"`, but the API says `"Non alcoholic"`. | Existing rows are never updated, so after fixing the code I had to reset the database. |
| The importer said `222 already existed` after I reran `schema.sql` | The IntelliJ data source pointed at a different database (`postgres`) than the application used. | Always check *which database* a script runs against. |
| `wrong column type [intensity] ... found SMALLINT, expecting INTEGER` | The schema used `SMALLINT`, but the entity field is a Java `int`. | This is exactly what `hbm2ddl.auto=validate` is for: it catches schema/entity mismatches early. I changed the schema to `INTEGER`. |
| `new Garnish(name, null)` did not compile | `Garnish` used `@AllArgsConstructor`, which includes the `id`. | Lombok-generated constructors depend on the field list; I removed `@AllArgsConstructor` from `Garnish` and used the no-arg constructor and setters instead. |

## Limitations and what I would do differently

- **About 48% of the fetched drinks (204 of 426) were skipped** because they have more than one ingredient of the same type, or an ingredient I did not classify. The data model is simpler than reality, and I chose to keep it that way for a more manageable scope.
- **The intensities are AI-generated, subjective estimates**. If I had more time, I would choose to set the flavor intensities myself or, even better, run some sort of survey to gather results as close to reality as possible.
- **Only the first garnish is kept**, and measures live as plain text in `description`, so the app cannot scale a recipe or search by quantity.
- **File paths are relative** (`database/seed/...`), so the importer must be run from the project root.
- **No incremental update.** Changing a classification means resetting the database. A more advanced importer would be able to update existing rows, and this would also be more desirable if the app were ever to go into production.
- **Possible API limit** of 25 drinks per letter. 
- **Data source terms.** The free TheCocktailDB key is meant for development and learning. If the app were ever published commercially, the terms and attribution would need to be checked.

If I had more time, I would add more aliases to reduce skips, support a second spirit per cocktail, and turn the importer into a repeatable command with a "reset and seed" option.
