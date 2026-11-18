---
title: Entry 15 - Preparing for production and the CI/CD Pipeline
date: 2026-10-26
showAuthor: true
series: ["Backend-REST-API"]
series_order: 15
---

## Preparing the application for production

### Configuration through environment variables

My `HibernateConfig` already supports two modes, and it is chosen by whether the environment variable `DEPLOYED` exists:

- **Local:** the database name, user and password are read from `config.properties`, which is in `.gitignore`. The schema mode comes from the same file, so I can change it during development.
- **Deployed:** the connection string, database name, user and password are read from the environment variables `CONNECTION_STR`, `DB_NAME`, `DB_USERNAME` and `DB_PASSWORD`, and `hibernate.hbm2ddl.auto` is always `validate`.

This is the same idea as the teacher's tutorial, so no change was needed. Two reasons for it:

1. Passwords must never be in the code or in GitHub. The environment is the standard place for secrets in containers.
2. In production, Hibernate must never change the schema. The database was created from `schema.sql` and filled by the seeding, so `validate` only checks that the entities match it and fails loudly if they do not.

### Packaging as a fat jar

A container has no local Maven repository, so the application has to be a single file with all its dependencies. The `maven-shade-plugin` builds `target/app.jar` during `mvn package`. The important parts of the configuration:

- **`ManifestResourceTransformer`** with the main class, so `java -jar app.jar` knows where to start.
- **Excluding `META-INF/*.SF`, `*.DSA` and `*.RSA`.** Some libraries are signed, and the signature no longer matches once files are merged into one jar, so Java would refuse to start it.
- **`ServicesResourceTransformer`.** Libraries such as Hibernate and Jackson find parts of themselves through files in `META-INF/services`. If two libraries have a file with the same name, one overwrites the other and the application breaks at runtime, not at build time. This transformer merges them instead. 

### The Dockerfile

```dockerfile
FROM eclipse-temurin:25-jre
COPY target/app.jar /app.jar
EXPOSE 7070
CMD ["java", "-jar", "/app.jar"]
```

The tutorial used a Java 17 base image. My project is built for Java 25, and a Java 17 runtime cannot run those class files, so the image must also be Java 25. The server log later confirmed `jvm 25.0.4.1+1-LTS`. The Dockerfile only copies a jar that Maven has already built, which keeps the image small.

### Challenge: `mvn: command not found`

When testing the jar in the terminal (`mvn clean package`, then `java -jar target/app.jar`), my Mac had no Maven. IntelliJ has its own bundled Maven, which the terminal cannot see. I installed Maven with Homebrew (`brew install maven`). Homebrew also installed Java 27 as a dependency of Maven. (GitHub Actions uses its own Maven, so the pipeline does not depend on this.)

### Challenge: the build failed in Maven but worked in IntelliJ

The first Maven build failed with about fifty `cannot find symbol` errors, such as `getName()`, `getSpiritId()` and `getCocktail()`, and `CocktailCandidate` "required: no arguments". Every missing method was one that **Lombok** normally generates, so Lombok was not running.

The cause is a change in Java 23: the compiler no longer runs annotation processors automatically just because they are on the classpath. IntelliJ switches annotation processing on by itself, so it had worked there the whole time. Maven needs to be told explicitly. The fix was to configure `maven-compiler-plugin` with Lombok as an annotation processor, and `release` set to 25:

```xml
<configuration>
    <release>25</release>
    <annotationProcessorPaths>
        <path>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.42</version>
        </path>
    </annotationProcessorPaths>
</configuration>
```

Using `release` also removed a warning about `-source`/`-target` without system modules. This problem would also have failed the GitHub Actions build, so finding it locally saved time.

### `.gitignore`

I reviewed the file to make sure build output and secrets never reach GitHub. `target/` and `config.properties` were already ignored. I added `.env` (the file the server uses for secrets) and `logs/` (logback writes to `logs/cocktailApp.log`), and removed the many rules for tools I do not use. One more point to remember: `.gitignore` does not affect files that were already committed, so `git ls-files` is the way to check that nothing sensitive is tracked.

## CI/CD with GitHub Actions and Docker Hub

### The idea

A CI/CD pipeline means I never have to build or upload anything by hand. Pushing to `main` triggers this chain:

1. GitHub starts an empty Ubuntu machine.
2. It checks out the code and installs Java 25.
3. `mvn package` compiles, **runs all the tests (including the Testcontainers tests)** and builds `app.jar`.
4. Only if every test passes, it logs in to Docker Hub, builds the image from the `Dockerfile` and pushes it.

Because the image is only published when the tests pass, a broken commit cannot reach the server. That is the real value of the pipeline.

### The workflow file

The file lives in `.github/workflows/` (the folder name must match exactly, with the leading dot). It uses the official actions for checkout, Java setup, Docker login and Docker build-and-push, and runs on pushes to `main`. The image is tagged `<username>/cocktail_api:latest`.

The teacher's tutorial was the starting point. The differences I had to make:

| Tutorial (Java 17 course) | My project |
|---|---|
| Java 17 | Java 25 |
| Corretto 17 base image | A Java 25 base image |
| Image name `hotel_api` | `cocktail_api` |
| Newer versions of the actions than the tutorial | Updated to current versions |

### Secrets

The workflow needs to log in to Docker Hub, but it is stored in a public-facing repository, so credentials cannot be written in it. They are stored as encrypted GitHub secrets (Settings, Secrets and variables, Actions):

- `DOCKERHUB_USERNAME`
- `DOCKERHUB_TOKEN`: an access token created in Docker Hub with read and write permissions. I used a token instead of my password because a token can be limited and revoked without changing the real password.

### Challenge: the "Get started with GitHub Actions" page

When I first opened the Actions tab, GitHub showed a page with suggested templates (such as "Java with Maven") and no run history. I was unsure if I was in the right place. This is the normal page for a repository without any workflow. None of the templates fit, since they have no Java 25, no Docker Hub and no Testcontainers. Instead I created my own workflow file in the repository, and the Actions tab then showed my runs.

### Result

The first run was green (about one minute), and the image `cocktail_api` appeared on Docker Hub as a public repository.


