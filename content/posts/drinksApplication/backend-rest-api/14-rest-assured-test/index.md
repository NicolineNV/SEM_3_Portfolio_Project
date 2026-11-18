---
title: Entry 14 - REST tests with REST-assured and Testcontainers
date: 2026-10-25
showAuthor: true
series: ["Backend-REST-API"]
series_order: 14
---

## REST tests with REST-assured and Testcontainers

### Approach

A REST test starts the whole application as a real HTTP server and sends real requests to it. It is different from a service test because it also verifies routing, JSON serialization, status codes and the exception handlers.

The test class `CocktailControllerTest` extends the same `IntegrationTestBase` as the other integration tests, so it already has an `EntityManagerFactory` that points to a fresh PostgreSQL container:

```java
@BeforeAll
static void startServer() {
    app = ApplicationConfig.startServer(0, emf);   // port 0 = Javalin picks a free port
    RestAssured.baseURI = "http://localhost";
    RestAssured.port = app.port();
    RestAssured.basePath = "/api/v1";
}
```

Choosing port `0` means the test never fails because a port is already in use, for example by the development server. Test data is created through the DAOs in a helper method, and the database is cleaned after every test.

REST-assured tests are written in a `given / when / then` style:

```java
given()
.when()
    .get("/cocktails/{id}", id)
.then()
    .statusCode(200)
    .body("name", equalTo("Old Fashioned"));
```

### The first three tests

| Test | Expected result |
|---|---|
| Get an existing cocktail | 200 and the correct `name` in the JSON |
| Get an unknown id | 404 and `status: 404` in the JSON body |
| Get `/cocktails/abc` | 400 |

### Library version

I added `io.rest-assured:rest-assured` (version 5.5.1) with test scope. It brings Groovy (4.0.22) as a dependency. I could not verify beforehand whether this version works on Java 25, because I found no clear statement either way. It was tested by running it. The tests ran without Groovy errors.

### Challenge: "Could not find a valid Docker environment"

After the tests had passed several times, a later run failed. The cause was not the test code. Testcontainers needs Docker to start PostgreSQL, and the log said:

```
UnixSocketClientProviderStrategy: ... NoSuchFileException (/var/run/docker.sock)
```

Docker Desktop was not running. The fix was to start Docker Desktop (with `docker ps` as a check). If the socket is still missing, the setting "Allow the default Docker socket to be used" in Docker Desktop's advanced settings must be turned on.

The log also showed a second error: a `NullPointerException` from `@AfterAll`, because `app` was never created when the container failed to start. It hid the real cause. I fixed it with a null check:

```java
@AfterAll
static void stopServer() {
    if (app != null) app.stop();
}
```

This also matters later: GitHub Actions' Linux runners have Docker installed, so the same tests run in the pipeline without extra setup.
