---
title: Entry 10 - Testing - Unit and Integration
date: 2026-10-11
showAuthor: true
series: ["Drinks Application"]
series_order: 10
---

The course requires three kinds of tests: unit tests, integration tests and REST-assured tests. They answer different questions and can be written at different points in the project. This part documents the unit and integration tests written so far, the infrastructure behind them, the mistakes I made along the way, and what the tests deliberately do **not** cover.

## Testing strategy: test what is different, not what is identical

All 14 DAOs inherit the same CRUD methods from `AbstractDAO`. Writing one test class per DAO would test the same code 14 times. Instead I tested `AbstractDAO`'s CRUD once, through `UserDAO`, and only wrote targeted tests for classes where the behavior differs. I mainly focused on the places where the code is most critical and errors are most likely. My main focus was anything that touched my database. After all, the next step after testing was to seed the database with data from the external API, so I had to make sure that all CRUD methods and specialized queries worked before that.

### Critical classes that also needed testing

- **`CocktailDAO`:** five nullable relations, a shared `JOIN FETCH` query, and the quiz candidate query.
- **`CocktailMenuDAO`:** a composite key mapped with `@EmbeddedId` and `@MapsId`, which is a different mapping pattern from the simple entities.
- **`Spirit`:** cascading of flavor tags through a join entity (covered inside the `CocktailDAO` flavor-profile test).

Simple entities such as `Garnish` and `CocktailFamily` contain nothing but plain columns, so they carry little risk and are exercised indirectly as test data in the other tests. They needed no special attention, so I skipped over them fairly quickly.

## Unit tests

Unit tests exercise one class in isolation, with no database. `EuclideanDistanceStrategy` is a good fit because it is pure math and holds the whole matching logic. It is critical that this class calculates the correct distance, which makes it the obvious choice:

```java
public class EuclideanDistanceStrategyTest {

    private final EuclideanDistanceStrategy strategy = new EuclideanDistanceStrategy();

    @Test
    void identicalProfilesHaveZeroDistance() {
        Map<Long, Integer> profile = Map.of(1L, 5, 2L, 3);
        assertEquals(0.0, strategy.calculateDistance(profile, profile), 0.0001);
        // delta parameter needed because double-calculations can have microscopic differences in result
    }
    @Test
    void calculateKnownDistanceCorrectly() {
        Map<Long, Integer> userAnswers = Map.of(1L, 5, 2L, 1);
        Map<Long, Integer> cocktailProfile = Map.of(1L, 3, 2L, 1);
        
        // Difference: dim1 = 5-3=2, dim2 = 1-1=0 → √(2² + 0²) = √4 = 2.0
        assertEquals(2.0, strategy.calculateDistance(userAnswers, cocktailProfile), 0.0001);
    }
    @Test
    // Most important test - tests that a NullPointerException is not thrown if some dimensions are missing
    void missingDimensionCountsAsZero() {
        Map<Long, Integer> userAnswers = Map.of(1L, 5);
        Map<Long, Integer> cocktailProfile = Map.of(2L, 5);
        
        // dim1: 5-0=5, dim2: 0-5=-5 → √(25+25) = √50 ≈ 7.0711
        assertEquals(Math.sqrt(50), strategy.calculateDistance(userAnswers, cocktailProfile), 0.0001);
    }
}
```

The three tests increase in difficulty: 

- The first test is the simplest case: identical input gives 0. I compare `profile` to `profile` to simulate a user answer and a candidate with identical values. Two identical flavor profiles are a perfect match, and I wanted to make sure that a perfect match gives a distance of 0 as expected. `assertEquals` needs a delta (`0.0001`) because floating-point arithmetic can produce tiny rounding errors, so an exact comparison of two double values could fail even when the result is correct.

- The second test uses a known distance that I calculated by hand and checks that the code reaches the same result. This shows that the logic is correct and that I can trust the calculation.

- The last and most difficult test is an edge case that exercises the `getOrDefault` line. Without that line, a dimension missing from one of the maps would cause a `NullPointerException`. The user can choose not to rate every flavor dimension, so one or more dimensions might be missing. The test also covers a cocktail candidate that is missing dimensions. That will most likely not happen, because I have decided that all candidates will always have all five, but I wanted to be sure that the calculation would not fail if it did.

## Integration test infrastructure

### A real database in Docker

Integration tests run against a real PostgreSQL database started by **Testcontainers**. I chose it over mocks or an in-memory database because the JPA mappings are then tested against the same database engine as production. Of course, this means Docker must be running locally.

My project uses Testcontainers 2.x. The module is named `testcontainers-postgresql`, and the container class moved to a new package and is no longer generic:

```java
import org.testcontainers.postgresql.PostgreSQLContainer;
// Other imports

public final class TestHibernateConfig {

    // Variables, constructor and getEntityManagerFactory()

    private static void startContainer() {
        container = new PostgreSQLContainer("postgres:16")
                .withDatabaseName("test_db")
                .withUsername("test")
                .withPassword("test");
        container.start();
    }

    private static Properties buildProps() {
        Properties props = HibernateBaseProperties.createBase();
        props.put("hibernate.hbm2ddl.auto", "create-drop");
        props.put("hibernate.connection.url", container.getJdbcUrl());
        props.put("hibernate.connection.username", container.getUsername());
        props.put("hibernate.connection.password", container.getPassword());
        return props;
    }
}
```

The test configuration uses `create-drop`: Hibernate builds the schema from the entities for each test run. That is intentional, because the goal of these tests is to verify that entities and DAOs work together. Because of this, the `ON DELETE CASCADE` and `ON DELETE SET NULL` rules from my script do not exist in the test database, so I have deliberately not written a test that deletes a user and expects the favorite cocktails to disappear as well. Such a test would fail, not because of my code, but because the delete rules only exist in `schema.sql` and are not something Hibernate knows about. Whether the entities match `schema.sql` is verified separately with `hbm2ddl.auto=validate` (set in `config.properties`) against the real database, so it does not need an integration test.

### Making the DAOs testable

The first version of `AbstractDAO` always fetched `HibernateConfig.getEntityManagerFactory()`, which meant any test would have hit my development database. The tests exposed this architectural gap. The fix was constructor injection of the `EntityManagerFactory`, so a test can pass in its own:

```java
protected AbstractDAO(Class<T> entityClass) {
    this(entityClass, HibernateConfig.getEntityManagerFactory());
}

protected AbstractDAO(Class<T> entityClass, EntityManagerFactory emf) {
    this.entityClass = entityClass;
    this.emf = emf;
}
```

The production code is unchanged (the simple constructor works as before), but integration tests can now inject their own `EntityManagerFactory`:

```java
class UserDAOIntegrationTest extends IntegrationTestBase {

    private UserDAO userDAO;

    @BeforeEach
    void setup() {
        userDAO = new UserDAO(emf);
    }
}
```

### A shared base class

All intergrationtest extends an abstract class called `IntegrationTestBase`. This abstract class holds a `@BeforeAll` method that creates an `EntityManagerFactory` and an `@AfterEach` method that cleans the database:

```java
public abstract class IntegrationTestBase {

    protected static EntityManagerFactory emf;

    @BeforeAll
    static void initEmf() {
        emf = TestHibernateConfig.getEntityManagerFactory();
    }

    @AfterEach
    void cleanDatabase() {
        try (EntityManager em = emf.createEntityManager()) {
            em.getTransaction().begin();

            // Deleted in order, so foreign keys do not block (children before parents)
            em.createQuery("DELETE FROM CocktailMenu").executeUpdate();
            em.createQuery("DELETE FROM SpiritFlavorTag").executeUpdate();
            // ... remaining join entities, then Cocktail, ingredients, family, FlavorTag, User
            em.getTransaction().commit();
        }
    }
}
```

Rows are deleted children-first because foreign keys would otherwise block the deletes - the same reason `schema.sql` drops tables in a specific order.

**Lesson learned: `@AfterEach` vs `@AfterAll`.** My first run failed during test discovery because the cleanup method was annotated with `@AfterAll`. JUnit requires `@AfterAll` methods to be `static`, since they run once without a test instance. More importantly, `@AfterAll` would have been wrong even if made static: the database would only be cleaned once at the end, so data from one test would leak into the next and `UNIQUE` constraints would make later tests fail for reasons unrelated to the code. `@AfterEach`, on the other hand, guarantees that every test starts with an empty database, so data does not leak into later tests and `UNIQUE` constraints do not cause problems. `@BeforeAll` on the `EntityManagerFactory` setup is correct, though, and must be static, because I want to keep the same factory throughout the whole test class.

## What the integration tests cover

### `UserDAOIntegrationTest`: the shared CRUD logic

These tests establish that `AbstractDAO`'s transactions and mappings work. In `UserDAOIntegrationTest`, I test that it is possible to create a user, that I can retrieve the same user afterwards, and that deleting a user actually removes them from the database. I also test `findByUsername()`, both when the user exists and when they do not. In the latter case, the method has to return an empty `Optional` instead of throwing an exception, which it fortunately does. Writing them also showed me that my first version of the test class did not cover `update` at all, so I added a dedicated test for that later in the process:

```java
@Test
void updatePersistsChanges() {
    User saved = userDAO.create(new User("tempUser", "old@test.dk", "hashed_pw"));

    saved.setEmail("new@test.dk");
    userDAO.update(saved);

    Optional<User> found = userDAO.getById(saved.getId());
    assertTrue(found.isPresent());
    assertEquals("new@test.dk", found.get().getEmail());
}
```

This test matters because `update` uses `em.merge(...)`. After `create` returns, the `EntityManager` that saved the user is closed, so the returned object is *detached* and Hibernate no longer tracks changes to it. Calling a setter alone changes nothing in the database. `merge` copies the changes into a new `EntityManager`, and reading the user back with a fresh `getById` proves the change was persisted. 

### `CocktailDAOIntegrationTest`: optional relations, `JOIN FETCH` and quiz candidates

The most important test recreates the Old Fashioned case: a cocktail saved with only a spirit, and no liqueur, mixer or syrup:

```java
@Test // Test that cocktails without optional ingredients are still fetched by getByIdWithDetails()
void cocktailNoOptionalIngredients() {
    CocktailFamily family = createFamily("Classics");
    Spirit whiskey = createSpirit("Whiskey");
    Cocktail saved = createCocktail("Old Fashioned", family, whiskey);

    Optional<Cocktail> found = cocktailDAO.getByIdWithDetails(saved.getId());

    assertTrue(found.isPresent());
    Cocktail cocktail = found.get();

    // Works without exceptions because JOIN FETCH loaded the relations
    assertEquals("Whiskey", cocktail.getSpirit().getName());
    assertEquals("Classics", cocktail.getCocktailFamily().getName());
    assertNull(cocktail.getLiqueur());
    assertNull(cocktail.getMixer());
    assertNull(cocktail.getSyrup());
    assertNull(cocktail.getGarnish());
}
```

Calling `getSpirit().getName()` after the `EntityManager` has closed is itself the proof that `JOIN FETCH` worked. 

A second test saves a mocktail without a spirit and checks that `getAllWithDetails` still returns it, which proves the query uses `LEFT JOIN FETCH` rather than a plain `JOIN FETCH` that would silently drop such cocktails:

A third test demonstrates the problem that `JOIN FETCH` solves, by deliberately triggering the exception with the plain `getById`:

```java
@Test // Test that shows why using JOIN FETCH is necessary
void plainGetByIdLeavesRelationsLazy() {
    CocktailFamily family = createFamily("Classics");
    Cocktail saved = createCocktail("Old Fashioned", family, createSpirit("Whiskey"));

    Cocktail found = cocktailDAO.getById(saved.getId()).orElseThrow();

    // EntityManager is closed, and spirit was never fetched
    assertThrows(LazyInitializationException.class, () -> found.getSpirit().getName());
}
```

The remaining tests cover the quiz query. `candidatesFilteredByFamily` shows that the family filter is a hard filter: only cocktails from the chosen family are selected. `spiritFilterIsOptional` runs the same query twice, once with a spirit id (one cocktail remains) and once with `null` (both pass), which is exactly the ***"user has no spirit preference"*** case.

The flavor-profile test is the most demanding. It ensures that a cocktail's flavor dimensions are calculated correctly. The test takes a cocktail with a spirit and a syrup that have different intensities per dimension, and checks that summing and ranking them gives the expected result. Salty and Spicy are given the same sum on purpose, so the tie-breaker case is tested too:

```java
@Test // Checks that a candidate's flavor profile sums intensities and ranks dimensions
void candidateFlavorProfile () {

    // Important - flavor tags need to be created in this order!
    FlavorTag sweet = flavorTagDAO.create(new FlavorTag("Sweet"));
    FlavorTag sour = flavorTagDAO.create(new FlavorTag("Sour"));
    FlavorTag bitter = flavorTagDAO.create(new FlavorTag("Bitter"));
    FlavorTag salty = flavorTagDAO.create(new FlavorTag("Salty"));
    FlavorTag spicy = flavorTagDAO.create(new FlavorTag("Spicy"));

    // Creates Spirit and Syrup, creates CocktailFamily and Cocktail, makes List<CocktailCandidates>

    /* 
    Spirit: Sweet 1, Sour 2, Bitter 4, Salty 1, Spicy 1
    Syrup: Sweet 5, Sour 1, Bitter 1, Salty 1, Spicy 1

    Sums: Sweet 6, Sour 3, Bitter 5, Salty 2, Spicy 2

    Ranking: Sweet 5, Bitter 4, Sour 3, Salty 2, Spicy 1
    Salty and Spicy tie: the one with the lowest id wins. (Salty was created first = lowest id) 
    */

    assertEquals(5, profile.get(sweet.getId()));
    assertEquals(4, profile.get(bitter.getId()));
    assertEquals(3, profile.get(sour.getId()));
    assertEquals(2, profile.get(salty.getId()));
    assertEquals(1, profile.get(spicy.getId()));
}
```

The flavor tags are created in a fixed order, because ties are resolved on the lowest id, which makes the creation order part of the test's preconditions. This test also verifies that the cascade from `Spirit` to its `SpiritFlavorTag` join entities works.

### `CocktailMenuDAOIntegrationTest`: composite keys

`CocktailMenu` is mapped with `@EmbeddedId` and `@MapsId`, so its key is derived from two relations. The tests verify the following:

- **`savedFavoriteCompositeKey()`**: 
    - A saved favorite can be found again with a `CocktailMenuId`, and `@PrePersist` has set `savedAt`.

- **`getFavoritesReturnsCocktails()`**: 
    - `getFavoritesByUser()` returns cocktails with their relations fetched (accessing `getSpirit().getName()` would throw without the `JOIN FETCH` chain).

- **`favoritesAreSeparatedPerUser()`**: 
    - Favorites are separated per user, so one user is never able to see another user's favorites.

- **`deleteRemovesOnlyLink()`**: 
    - Deleting a favorite removes only the link, not the user or the cocktail.

- **`savingTheSameTwice()`**: 
    - Saving the same favorite twice is rejected by the composite primary key.

The last test also explains a design choice in the service layer: `CocktailMenuService.saveFavorite` checks for duplicates *before* saving, so the user receives a clean `409 Conflict` instead of a raw database error. 

## Mistakes along the way

Overall, the testing went smoothly. I only really encountered two problems, and both were reasonably easy to resolve:

- **Testcontainers version mismatch.** My first configuration used the 1.x package and class names, but the project uses 2.x. The container class had moved packages and lost its generic type parameter, so the configuration had to be rewritten.
- **`@AfterAll` instead of `@AfterEach`**, as I described above.

## Known limitations

I want to be explicit about what these tests do **not** prove:

- **Database-level `ON DELETE` rules.** Test tables are generated by Hibernate, not by `schema.sql`, so `ON DELETE CASCADE` and `SET NULL` do not exist in the test database. As mentioned, I chose not to write tests that delete a user and expect favorites to disappear, because they would fail for reasons unrelated to my code. Those rules are therefore not covered by any automatic test, and will be testest manually later after I have seeded my database with data from TheCocktailDB's API.

- **`getAll` and the simple entities** are not tested directly, since they use the same generic code as the tested paths. I expect that it will not be necessary to test all the other entities because of this, but of course I can never be completly sure that the code runs as it should without actually testing them. At the moment, however, I do not see the need for that.

- **`QuizService` as a whole.** The pieces are covered (candidate filtering and profile calculation in the DAO tests, the distance function in the unit tests), but an end-to-end test of filtering, profile building and strategy working together is still outstanding. I chose to postpone it in favor of seeding real data first. However, I do need to write a `QuizService` test before I can confidently use the class's methods.

## REST-assured

REST-assured tests send real HTTP requests to running endpoints, so they can only be written once the Javalin layer exists. I have not built that layer yet, and therefore it will only be added later in the build process.
