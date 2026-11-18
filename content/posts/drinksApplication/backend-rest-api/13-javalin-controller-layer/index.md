---
title: Entry 13 - Building the Javalin layer
date: 2026-10-23
showAuthor: true
series: ["Backend-REST-API"]
series_order: 13
---

## Constructor injection all the way down

### The problem

My first version of the REST layer had a chain of objects that each created the next one themselves: the controller created a service, the service created a DAO, and the DAO opened its `EntityManager` from the global `HibernateConfig.getEntityManagerFactory()`. That works in production, but it makes the REST layer untestable: a REST test would talk to my real development database, not to the throwaway PostgreSQL container.

### The solution

The `EntityManagerFactory` is now passed in from the outside, through constructors, down the whole chain:

```
Main / test  ->  ApplicationConfig.startServer(port, emf)
                    -> Routes(emf)
                        -> CocktailController(emf)
                            -> CocktailService(emf)
                                -> CocktailDAO(emf)
```

- In production, `Main` passes `HibernateConfig.getEntityManagerFactory()`.
- In a REST test, the test passes the `EntityManagerFactory` that points to the Testcontainers database.

Services that used to have a no-argument constructor kept it, delegating to the real configuration, so nothing else in the project broke.

### Why this design

This is dependency injection by hand. There is no framework: the dependencies are visible in each constructor, which is enough for a project of this size. 


## The Javalin skeleton and error handling

### Structure

The REST layer follows the same layered architecture as the rest of the project:

- **`ApplicationConfig`** creates and starts the Javalin server, sets the base path and registers the exception handlers.
- **`Routes`** defines which URL goes to which controller method.
- **Controllers** (starting with `CocktailController`) read the request, call a service and write the response. They contain no business logic.
- **Services and DAOs** are the layers that already existed. Entities never leave the service layer: the controller only sees DTOs, so fields such as `passwordHash` cannot leak into a response.

The API is served under the base path **`/api/v1`**, which also means a later, incompatible version could live next to it. The first working endpoint is `GET /api/v1/cocktails/{id}`.

### Error handling

All errors are turned into JSON with the same shape, `{ "status": ..., "message": ... }`, by three handlers in `ApplicationConfig`:

| Exception | Response | Reason |
|---|---|---|
| `ApiException` (own exception with a status code) | The status code and message from the exception | The services decide what is wrong (for example 404 for an unknown id) |
| `ValidationException` (Javalin) | 400 "Invalid request" | A path parameter such as `/cocktails/abc` is the client's mistake, not the server's |
| Any other `Exception` | 500 "Internal server error" | The full exception is logged on the server, but the client gets no details |

### Challenge: invalid ids returned 500

`ctx.pathParamAsClass("id", Long.class).get()` throws a `ValidationException` when the id is not a number. Without a specific handler, my catch-all handler turned this into a 500, which wrongly says that the server failed. Adding a handler for `ValidationException` makes it a 400. A REST test (Entry 4) covers this case.

### Security decisions I kept to

- The catch-all handler never sends exception text to the client. Internal messages can reveal class names, SQL or file paths.
- The same error is returned for a wrong username and a wrong password, so an attacker cannot find out which usernames exist.
- `rawPassword` and `passwordHash` are never logged.
