---
title: Entry 7 - DTOs and the Service Layer
date: 2026-10-05
showAuthor: true
series: ["Drinks Application"]
series_order: 7
---

The service layer contains the business logic and is the only layer that talks to both DAOs and DTOs.

## Why DTOs?

Why do I even use DTOs when I could just use the entities I have already created? Well, that is because entities never leave the service layer, and there are three very important reasons for that:

1. **Security.** `User` contains `passwordHash`. If an entity is sent out directly (for example as JSON), the hash leaks to the frontend, which is less than ideal. A `UserDTO` simply does not have the field, so this issue cannot occur.
2. **Lazy loading.** I have talked about lazy loading a lot at this point, and that is because it is probably the most annoying and most frequent problem I have encountered throughout the build process. DTOs are ideal for avoiding this problem because they are built from data that has already been fetched, so a `LazyInitializationException` cannot occur in a later layer.
3. **Control.** With DTOs, I can deliberately choose what is exposed, and I can change the data model without changing how the API looks from the outside.

I distinguish between **response DTOs** (what is sent out, for example `CocktailDTO`) and **request DTOs** (what is received, for example `CreateCocktailDTO`), since they have different fields and purposes. Lombok removes boilerplate, and mapping from an entity is done manually in a static factory method:

```java
@Getter
@AllArgsConstructor
@NoArgsConstructor
public class CocktailDTO {

    private Long id;
    private String name;
    private String description;
    private String instructions;
    private String cocktailFamily;
    private String spirit;
    private String liqueur;
    private String mixer;
    private String syrup;
    private String garnish;

    public static CocktailDTO fromEntity (Cocktail c) {
        return new CocktailDTO (
                c.getId(),
                c.getName(),
                c.getDescription(),
                c.getInstructions(),
                c.getCocktailFamily() != null ? c.getCocktailFamily().getName() : null,
                c.getSpirit() != null ? c.getSpirit().getName() : null,
                c.getLiqueur() != null ? c.getLiqueur().getName() : null,
                c.getMixer() != null ? c.getMixer().getName() : null,
                c.getSyrup() != null ? c.getSyrup().getName() : null,
                c.getGarnish() != null ? c.getGarnish().getName() : null
        );
    }
}
```

The `null` checks are necessary because the ingredients are optional. The DTO only contains the name of each relation, not the whole nested object, because the client only needs that much to display the recipe.

## `UserService` and password security

Passwords are never stored in plain text. I use BCrypt, which is built for passwords: it is deliberately slow (making brute force impractical) and builds a unique random salt into every hash, so two identical passwords get different hashes. We have not covered the topic of security yet (we will be introduced to it later this week), so my approach to password security might change as development progresses. For now, however, I believe BCrypt works well for my API, though this decision is based on my very limited knowledge of security.

```java
public class UserService {

    // Variables and constructor

    public UserDTO register(String username, String email, String rawPassword) {
            if (userDAO.findByUsername(username).isPresent()) {
                throw new ApiException(HttpStatus.CONFLICT, "Username already in use: " + username);
            }

            String hashedPassword = BCrypt.hashpw(rawPassword, BCrypt.gensalt());
            User savedUser = userDAO.create(new User(username, email, hashedPassword));
            logger.info("New user created: {}", username);
            return UserDTO.fromEntity(savedUser);
        }

        public UserDTO login(String username, String rawPassword) {
            User user = userDAO.findByUsername(username)
                    .orElseThrow(() -> new ApiException(HttpStatus.UNAUTHORIZED, "Wrong username or password"));

            if (!BCrypt.checkpw(rawPassword, user.getPasswordHash())) {
                throw new ApiException(HttpStatus.UNAUTHORIZED, "Wrong username or password"); // Same message no matter which of the two is wrong
            }
            return UserDTO.fromEntity(user);
        }

        // IsAdmin boolean method
}
```

The hash string contains the algorithm, the cost factor and the salt, which is why `checkpw` can recompute and compare without a separately stored salt. This is also why I never hash the entered password again and compare the two hashes with `equals`: the salts would be different. Login gives the **same error message** whether the username does not exist or the password is wrong, so an attacker cannot tell which usernames exist.

## `CocktailMenuService`: existence checks and duplicates

The save-favorite logic validates before it writes. It checks that the link does not already exist (`409`, `HttpStatus.CONFLICT`) and that both the user and the cocktail exist (`404`, `HttpStatus.NOT_FOUND`). This gives clear errors instead of cryptic database errors.

```java
public class CocktailMenuService {

    // Variables 

    public CocktailDTO saveFavorite (Long userId, Long cocktailId) {
        CocktailMenuId id = new CocktailMenuId(userId, cocktailId);
        if (cocktailMenuDAO.getById(id).isPresent()) {
            throw new ApiException(HttpStatus.CONFLICT, "The cocktail is already saved as a favorite");
        }
        User user = userDAO.getById(userId)
                .orElseThrow(() -> new ApiException(HttpStatus.NOT_FOUND, "User does not exist: " + userId));
        Cocktail cocktail = cocktailDAO.getByIdWithDetails(cocktailId)
                .orElseThrow(() -> new ApiException(HttpStatus.NOT_FOUND, "Cocktail does not exist: " + cocktailId));

        cocktailMenuDAO.create(new CocktailMenu(user, cocktail));
        return CocktailDTO.fromEntity(cocktail);
    }

    // Get and remove methods

}
```

The full `User` and `Cocktail` objects are fetched for two reasons. The lookups double as the existence checks, and the `CocktailMenu` entity needs real entity references: its `user` and `cocktail` fields are annotated with `@MapsId` (just like the `SpiritFlavorTag` example in Entry 5), so Hibernate derives the composite key from them when the row is saved.

## `CocktailService`: validation and role check

The business rules that I deliberately kept out of the database live here:

```java
public class CocktailService {

    // Variables, constructor, getById() and getAll() methods

    // Admin method - for admin to create new cocktail in database
    public CocktailDTO createCocktail(CreateCocktailDTO request, UserDTO admin) {

        if (!userService.isAdmin(admin)) {
            throw new ApiException(HttpStatus.FORBIDDEN, "Only admins can create new cocktails");
        }
        if (request.getSpiritId() == null && request.getLiqueurId() == null
                && request.getMixerId() == null && request.getSyrupId() == null) {
            throw new ApiException(HttpStatus.BAD_REQUEST,
                    "A cocktail must have at least one ingredient (spirit, liqueur, mixer or syrup)");
        }

        // ... looks up the family and the chosen ingredients, then creates and saves the cocktail
    }
}
```

The role check reuses `userService.isAdmin(...)`, so the rule for "what is an admin" exists in one place. The service takes a `UserDTO` and does not look up the user itself. It relies on the caller (the upcoming Javalin layer) having already determined who is logged in.


