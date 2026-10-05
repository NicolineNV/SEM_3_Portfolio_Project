---
title: Entry 6 - The DAO Layer
date: 2026-10-05
showAuthor: true
series: ["Drinks Application"]
series_order: 6
---

The DAO layer is responsible for all communication with the database. I have made some of the project's most important design decisions here.

## Generic CRUD in one place

With 14 entities it would be wasteful to write create/read/update/delete 14 times. Instead, `IDAO<T, ID>` defines the contract and `AbstractDAO<T, ID>` implements it generically:

```java
public interface IDAO <T, ID> {
    T create(T entity);
    Optional<T> getById(ID id);
    List<T> getAll();
    T update(T entity);
    void delete(ID id);
}
```

There are two generic types because not all entities have the same kind of key. Most use `Long`, but join entities such as `SpiritFlavorTag` and `CocktailMenu` use their `@EmbeddedId` class. A concrete DAO therefore becomes very short:

```java
public class SpiritDAO extends AbstractDAO<Spirit, Long> {
    public SpiritDAO(){
        super(Spirit.class);
    }
}
```

## Transactions and EntityManager handling

Each method opens its own `EntityManager` through try-with-resources, so it is always closed. All transaction logic is collected in one method, so begin/commit/rollback is not repeated:

```java
protected T executeInTransaction(Function<EntityManager, T> action) {
        try (EntityManager em = emf.createEntityManager()) {
            EntityTransaction emTransaction = em.getTransaction();
            try {
                emTransaction.begin();
                T result = action.apply(em);
                emTransaction.commit();
                return result;
            } catch (RuntimeException e){
                if (emTransaction.isActive()) {
                    emTransaction.rollback();
                }
                logger.error("Transaction failed, rolled back: {}", e.getMessage(), e);
                throw e;
            }
        }
    } 
```

Only write operations (`create`, `update`, `delete`) run in a transaction. Reading needs none. Errors roll back and are logged before being rethrown.

## Lazy loading and `JOIN FETCH`

This is the most central technical problem I encountered in the DAO layer. Because the relations are `LAZY`, and the `EntityManager` is closed when the DAO method returns, accessing a lazy relation afterwards throws a `LazyInitializationException`. I learned that this typically happens when a service maps an entity to a DTO.

There are a few different ways to fix this problem, and two are very obvious "root fixes" that I deliberately rejected:

- **Making everything `EAGER`.** It does not remove the database calls, only their visibility and it applies globally, even where the relations are not used. I learned that this is called the N+1 problem. If my databse were small, maybe this would not be such a grave problem. Tho since I know, that I will seed my databse with over 600 cocktail recipes, and that each cocktail has 6 or more relations, then collecting just a list with all cocktails could be 6 times as many calls or maybe even more. Even though I only need some of the information.
- **Open Session in View** (keep the `EntityManager` open for the whole request). It hides when database calls happen and can hold connections open unnecessarily long. I does not fix the N+1 problem, only hides it. Of course it will not genereate any errors or exceptions, but it will make the site extreamly more slow. Another reason for not using this option, is that it will compromise the layers seperation of responsibilities, that I talked about in entry 3. With a open `EntityManager` controllers and JSON-mappers could have access to database calls and all the data would no longer be aggregated in the DAO layer.

Instead I use explicit `JOIN FETCH` per use case (wich is also the most recommended practice), so each query fetches exactly what it needs and nothing more. This way I can ensure that only the nessecary data will be fetched and I can also avoid any `LazyInitializationException`. To avoid repetition, the join part is collected in one constant, while the `WHERE` part is deliberately different for each method:

```java
public class CocktailDAO extends AbstractDAO<Cocktail, Long> {

    private static final String COCKTAIL_WITH_RELATIONS =
                "SELECT c FROM Cocktail c " +
                "LEFT JOIN FETCH c.cocktailFamily " + // LEFT JOIN FETCH because can be null
                "LEFT JOIN FETCH c.spirit " +
                "LEFT JOIN FETCH c.liqueur " +
                "LEFT JOIN FETCH c.mixer " +
                "LEFT JOIN FETCH c.syrup " +
                "LEFT JOIN FETCH c.garnish ";

        public Optional<Cocktail> getByIdWithDetails(Long id) {
            try (EntityManager em = emf.createEntityManager()) {

                String jpql = COCKTAIL_WITH_RELATIONS + "WHERE c.id = :id";

                return em.createQuery(jpql, Cocktail.class)
                        .setParameter("id", id)
                        .getResultStream()
                        .findFirst();
            }
        }

        // Rest of class logic...
}
```

`LEFT JOIN FETCH` is used because the relations are nullable. A plain `JOIN FETCH` would exclude cocktails that are missing, for example, a liqueur or a mixer in The Old Fashioned recipe. All six relations are `@ManyToOne` (single values), so they can be joined at the same time without problems.

There is one important limitation to notice: you cannot `JOIN FETCH` several `List` collections in the same query (it throws a `MultipleBagFetchException`). Therefore the ingredients' `flavorTags` lists are not fetched in that same query, but are accessed while the `EntityManager` is still open (I will talk more about that in a later entry).

## Specialized queries

Some user stories required more than simple `CRUD` methods. For example login looks up by username. I chose this because `username` is the natural key. Before a user logs in, only username and password are entered into the system. The user does not know what their id is, because id is a autogenereted number (`BIGSERIAL`), they would never see this number and can therefore not inform the system about this. This requires `username`is `UNIQUE` so that the post only touches one row. Other than that ***"show my favorites"*** for example also required more than a simple `CRUD` method. In `CocktailMenuDAO` the method `getFavoritesByUser` looks up by user id in a composite-key table and therefore also needs a little more elaborate method:

```java
public class UserDAO extends AbstractDAO<User, Long> {

    public UserDAO() {
        super(User.class);
    }

    public Optional<User> findByUsername(String username) {
        try (EntityManager em = emf.createEntityManager()) {
            String jpql = "SELECT u FROM User u WHERE u.username = :username";
            List<User> results = em.createQuery(jpql, User.class)
                    .setParameter("username", username)
                    .getResultList();
            return results.stream().findFirst();
        }
    }
}
```

Important to note is that JPQL uses **entity and field names** (`User`, `u.username`), not table and column names (`users`, the `username` column). Hibernate translates to SQL through `@Table`/`@Column`, so a table can be renamed without touching queries.
