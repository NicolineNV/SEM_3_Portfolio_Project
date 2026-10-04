---
title: Entry 4 - Database and schema - important decisions
date: 2026-10-04
showAuthor: true
---

## Database

One of the first things I dug into was the relationships between the entities in my domain model, and how they would map to the tables in my database. The database has changed quite a bit from my original thinking when I drew up my first domain model. This is due to a number of challenges I ran into, and an even larger number of important decisions. This took my idea from a simple hand-drawn domain model to a detailed and well-thought-out database structure that satisfies `1NF`, `2NF` and `3NF`.

At the moment, my ERD diagram looks like this:

![ERD diagram](erd-diagram.png)

## Important decisions and reasoning

### Schema

I decided to build my `schema.sql` and database first before my Java entities. This means that the entities mirror the tables, and not the other way around. `schema.sql` is therefore "the truth", not Hibernate's auto-generated guesses. All though it can come in handy somethimes that Hibernate can generate tables automatically (`hbm2ddl.auto=create`) - it can also has its down sides like how its guesses about column types and constraints are imprecise. With a hand-written schema I control data types, `CHECK` constraints and foreign keys myself, and I can prove that the entities and the schema agree 

The script lives in `database/` in the project root and not in `src/main/resources`. It is run manually against PostgreSQL and is not part of the application's runtime, so it should not be packaged into the JAR.

### Flavor profiles: join tables with intensity

As I mentioned in the post about my idea, the user should be able to choose a cocktail based on a flavor preference. This is the most important aspect of my API - you could even go as far as to say it is part of the main purpose.

Every ingredient (Spirit, Liqueur, Mixer, Syrup) has an intensity from 1 to 5 for each flavor dimension (sweet, sour, bitter, salty, spicy). These flavor profiles are modeled as `FlavorTag` + **join tables** holding an `intensity` value for each ingredient type, such as `spirit_flavor_tag`. Since an ordinary `@ManyToMany` relationship cannot hold extra columns, the join tables have to be entities of their own with composite keys.

```SQL
CREATE TABLE flavor_tag (
    id   BIGSERIAL PRIMARY KEY,
    name VARCHAR(50) NOT NULL UNIQUE
);

CREATE TABLE spirit_flavor_tag (
    spirit_id     BIGINT  NOT NULL REFERENCES spirit(id)     ON DELETE CASCADE,
    flavor_tag_id BIGINT  NOT NULL REFERENCES flavor_tag(id) ON DELETE CASCADE,
    intensity     SMALLINT NOT NULL CHECK (intensity BETWEEN 1 AND 5),
    PRIMARY KEY (spirit_id, flavor_tag_id)
);
```

The primary key is composed of the two foreign keys, so an ingredient can only have one intensity per flavor dimension. The `CHECK` constraint guarantees at the database level that the intensity is between 1 and 5, so invalid data can never get in, regardless of which code writes to the table.

### `CocktailMenu` join table

`CocktailMenu` was chosen to be a join table between `User` and `Cocktail` called `user_favorite_cocktail`. Its main purpose is to hold the users' saved favorites, and also has an extra column, `saved_at`. This table covers a big part of user story 2.

```SQL
CREATE TABLE user_favorite_cocktail (
    user_id     BIGINT    NOT NULL REFERENCES users(id)    ON DELETE CASCADE,
    cocktail_id BIGINT    NOT NULL REFERENCES cocktail(id) ON DELETE CASCADE,
    saved_at    TIMESTAMP NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, cocktail_id)
);
```

### The main table `cocktail` - what defines it?

When I was building the tables and their relationships to each other, I asked myself an important question: **"What does a cocktail actually contain in my system?"**

My first draft made Spirit, Liqueur, Mixer and Syrup required on a cocktail. That did not hold up in reality: Cocktails such as the **Old Fashioned** only have a base alcohol **(Spirit)** and ice, so all the other category columns would end up being `NULL`. My thought was then to make **Liqueur**, **Mixer**, **Syrup** and **Garnish** nullable and keep only **Spirit** as `NOT NULL`, but that was not possible either, since some recipes are non-alcoholic.

So in the end, only `cocktail_family_id` is `NOT NULL` among the relationships, and all the others are nullable:

```SQL
CREATE TABLE cocktail (
    id                 BIGSERIAL PRIMARY KEY,
    name               VARCHAR(100) NOT NULL UNIQUE,
    description        TEXT,
    instructions       TEXT         NOT NULL,
    cocktail_family_id BIGINT       NOT NULL REFERENCES cocktail_family(id) ON DELETE RESTRICT,
    spirit_id          BIGINT       REFERENCES spirit(id)                   ON DELETE SET NULL,
    liqueur_id         BIGINT       REFERENCES liqueur(id)                  ON DELETE SET NULL,
    mixer_id           BIGINT       REFERENCES mixer(id)                    ON DELETE SET NULL,
    syrup_id           BIGINT       REFERENCES syrup(id)                    ON DELETE SET NULL,
    garnish_id         BIGINT       REFERENCES garnish(id)                  ON DELETE SET NULL,
    created_at         TIMESTAMP    NOT NULL DEFAULT now()
);
```

The rule ***"a cocktail must have at least one ingredient"*** is a **business rule** and is therefore enforced in the service layer, not as a database constraint. This keeps the schema flexible and puts the logic where it can be tested and produce a meaningful error message.

**Non-alcoholic** drinks are modeled as a `cocktail_family` **("Mocktails")** rather than a separate `is_alcoholic` flag. This is simpler and fits naturally with the quiz's first question about drink family.

### Foreign key strategy

The choice of `ON DELETE` behavior is deliberate and follows how "heavy" the relationship is:

- **`RESTRICT`** on `cocktail_family`: a family that is used by cocktails cannot be deleted by accident.
- **`SET NULL`** on ingredients and garnish: if an ingredient is deleted, the cocktail simply loses that ingredient, which is possible because the columns are nullable.
- **`CASCADE`** on favorites and flavor tags: they are dependent links that make no sense without their parent.

### Data from TheCocktailDB

**The database is seeded once** from TheCocktailDB, instead of the application making constant live calls to the API.

The reasoning is that the API does not contain flavor profiles, which, as I mentioned, are the core data of my system. The flavor profiles are necessary for the user to be able to filter the recipes by flavor.

In addition, the ingredients are just unstructured strings that would have to be normalized in my database anyway, and my admin user story (9) requires that my own database is the source of truth. That is the user story that lets an Admin add new recipes to the database.
