---
title: Entry 5 - Entities and JPA Mapping
date: 2026-10-04
showAuthor: true
series: ["Drinks Application"]
series_order: 5
---

This entry covers how the schema became Java classes, and the design choices that deserve a closer explanation.

## Basic entity and business key

All entities follow the same pattern. The example here is `Garnish`:

```java
@Entity
@Table(name ="garnish")
public class Garnish {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true, length = 100)
    private String name;

    @Column(columnDefinition = "TEXT")
    private String description;

    // Constructors, getters and setters ...

    @Override
    public boolean equals(Object o){
        if (this == o) return true;
        if (!(o instanceof Garnish that)) return false;
        return name != null && name.equals(that.name);
    }
    @Override
    public int hashCode(){
        return name != null ? name.hashCode() : 0;
    }
}
```

**Why do `equals`/`hashCode` use `name` and not `id`?** `id` is `null` until Hibernate has saved the object. If I used `id`, two newly created, not yet saved objects would both have `id = null` and would wrongly be considered equal. `name` has a `UNIQUE` constraint in the database and is therefore the real business key, whether or not the object has been saved. Without an overridden `equals`, two objects representing the same database row would be treated as different in a `Set` and in `contains()`.

## Enums and automatic timestamps

`User` shows two important patterns:

```java
@Enumerated(EnumType.STRING)
@Column(nullable = false, length = 10)
private Role role = Role.USER;

@Column(name = "created_at", nullable = false, updatable = false)
private LocalDateTime createdAt;

@PrePersist
protected void onCreate() {
    this.createdAt = LocalDateTime.now();
}
```

I have chosen to use `EnumType.STRING`, and not `ORDINAL`. `ORDINAL` stores the enum as a number (0, 1, ...), and if values are later reordered or inserted, the meaning of every existing row silently changes. `@PrePersist` sets the timestamp automatically on first save, so it can never be forgotten.

## Composite keys and join entities with extra data

The flavor profile join tables have an extra column (`intensity`), which a plain `@ManyToMany` cannot hold. The solution was to turn the link into its own entity with a composite key like this:

```java
@Embeddable
public class SpiritFlavorTagId implements Serializable{

    private Long spiritId;
    private Long flavorTagId;

    // Constructors, equals and hashCode (both fields)
}

@Entity
@Table(name = "spirit_flavor_tag")
public class SpiritFlavorTag {

    @EmbeddedId
    private SpiritFlavorTagId id = new SpiritFlavorTagId();

    @ManyToOne(fetch = FetchType.LAZY)
    @MapsId("spiritId")
    @JoinColumn(name = "spirit_id")
    private Spirit spirit;

    @ManyToOne(fetch = FetchType.LAZY)
    @MapsId("flavorTagId")
    @JoinColumn(name = "flavor_tag_id")
    private FlavorTag flavorTag;

    @Column(nullable = false)
    private Integer intensity;
}
```

`@MapsId` is the key: it tells Hibernate that the relation's id is also part of the composite primary key. That way `spirit_id` and `flavor_tag_id` are both foreign keys and the primary key, exactly as I have it in my SQL script. The same pattern is used for `LiqueurFlavorTag`, `MixerFlavorTag`, `SyrupFlavorTag` and `CocktailMenu` (the user's saved favorites).

On the parent side, a helper method and `cascade` make it very easy to work with:

```java
@Entity
@Table(name ="spirit")
public class Spirit {

    // Other variables, constructors, getters, setters

    @OneToMany(mappedBy = "spirit", cascade = CascadeType.ALL, orphanRemoval = true)
        private List<SpiritFlavorTag> flavorTags = new ArrayList<>();

    public void addFlavorTag (FlavorTag tag, int intensity){
            flavorTags.add(new SpiritFlavorTag(this, tag, intensity));
        }

    // Equals and hashCode    
}
```

## Cocktail: the optional relations and lazy loading

`Cocktail` ties everything together. Its the biggest class and represents the complete cocktail recipe. Five of the relations are optional, and all are `LAZY`:

```java
@Entity
@Table(name = "cocktail")
public class Cocktail {

    // Other variables

    @ManyToOne(optional = false, fetch = FetchType.LAZY)
    @JoinColumn(name = "cocktail_family_id", nullable = false)
    private CocktailFamily cocktailFamily;

    @ManyToOne(optional = true, fetch = FetchType.LAZY)
    @JoinColumn(name = "spirit_id", nullable = true)
    private Spirit spirit;

    // Likewise for liqueur, mixer, syrup and garnish

    // Constructors, getters, setters, equals and hashCode
}
```
The default for `@ManyToOne` is `EAGER`, which would load every relation each time a cocktail is read, even where they are not used at all. With `LAZY` I only load what I ask for. The price is that lazy loading has to be handled deliberately, which is something I will cover in the next entry.

## Registration and validation of the schema

Entities are registered explicitly in `EntityRegistry` with `addAnnotatedClass(...)`. The `Role` enum and the `@Embeddable` id classes do not need to be registered, since they are discovered through the entities that use them.

`hbm2ddl.auto` is used differently throughout the project. While developing the entities I used `create` to quickly see whether the mappings were valid. Afterwards I ran `schema.sql` manually and switched to `validate`:

```java
// HibernateConfig: locally the mode is controlled via config.properties, deployed is always validate
if (System.getenv("DEPLOYED") != null) {
            props.put("hibernate.hbm2ddl.auto", "validate");
            setDeployedProperties(props);
        } else {
            String mode = Utils.getPropertyValue("HBM2DDL_MODE", "config.properties");
            props.put("hibernate.hbm2ddl.auto", mode);
            setDevProperties(props);
        }
    return props;
```

`validate` is the only mode that proves my entities match my hand-written schema (column names, nullability and types). Hibernate changes nothing, but fails at startup if something does not line up. I ruled out `update` because Hibernate's schema guesses are imprecise, and it never removes obsolete columns.
