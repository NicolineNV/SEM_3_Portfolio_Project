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

Only write operations (`create`, `update`, `delete`) run in a transaction. Reading needs none. On error, the transaction is rolled back and the exception is logged before being rethrown.

## Lazy loading and `JOIN FETCH`

This is the most central technical problem I encountered in the DAO layer. Because the relations are `LAZY`, and the `EntityManager` is closed when the DAO method returns, accessing a lazy relation afterwards throws a `LazyInitializationException`. I learned that this typically happens when a service maps an entity to a DTO.

There are a few ways to fix this problem. Two of them are obvious "root fixes" that I deliberately rejected:

- **Making everything `EAGER`.** It does not remove the database calls, only their visibility, and it applies globally, even where the relations are not used. I learned that this is called the **N+1 problem**. If my database were small, this might not be such a serious problem. However, since I know that I will seed my database with over 600 cocktail recipes, and that each cocktail has six relations, fetching a simple list of all cocktails could trigger up to six extra queries per cocktail, even though I only need some of the information.
- **Open Session in View** (keeping the `EntityManager` open for the whole request). It hides when database calls happen and can hold connections open for unnecessarily long. It does not fix the N+1 problem, it only hides it. Of course, it will not generate any errors or exceptions, but it will make the site much slower. Another reason for not using this option is that it would compromise the separation of responsibilities between the layers that I described in Entry 3. With an open `EntityManager`, controllers and JSON mappers could trigger database calls, and data access would no longer be confined to the DAO layer.

Instead, I use an explicit `JOIN FETCH` per use case (which is also the recommended practice), so each query fetches exactly what it needs and nothing more. This also means I avoid any `LazyInitializationException`. To avoid repetition, the join part is collected in one constant, while the `WHERE` part is deliberately different for each method:

```java
public class CocktailDAO extends AbstractDAO<Cocktail, Long> {

    private static final String COCKTAIL_WITH_RELATIONS =
                "SELECT c FROM Cocktail c " +
                "LEFT JOIN FETCH c.cocktailFamily " + 
                "LEFT JOIN FETCH c.spirit " + // LEFT JOIN FETCH because five of the six relations are nullable
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

`LEFT JOIN FETCH` is used because the relations are nullable. A plain `JOIN FETCH` would exclude cocktails that are missing one of them, for example the Old Fashioned, which has no liqueur or mixer. All six relations are `@ManyToOne` (single values), so they can be joined at the same time without problems.

There is one important limitation to note: you cannot `JOIN FETCH` several `List` collections in the same query (it throws a `MultipleBagFetchException`). Therefore, the ingredients' `flavorTags` lists are not fetched in the same query but are accessed while the `EntityManager` is still open (I will talk more about that in a later entry).

## Specialized queries

Some user stories required more than simple `CRUD` methods. For example, login looks a user up by username. I chose this because `username` is the natural key. When a user logs in, the only things they enter are their username and password. The user does not know their id, because it is an auto-generated number (`BIGSERIAL`). They never see it and therefore cannot give it to the system. This requires `username` to be `UNIQUE`, so that the lookup matches at most one row.

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

Note that JPQL uses **entity and field names** (`User`, `u.username`), not table and column names (`users`, the `username` column). Hibernate translates to SQL through `@Table`/`@Column`, so a table can be renamed without touching queries.

Another example is ***"show my favorites"***. In `CocktailMenuDAO`, the method `getFavoritesByUser` looks up by user id in a composite-key table and therefore needs a slightly more elaborate query.
