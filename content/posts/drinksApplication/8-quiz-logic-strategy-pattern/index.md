---
title: Entry 8 - The Quiz Logic and the Strategy Pattern
date: 2026-10-05
showAuthor: true
series: ["Drinks Application"]
series_order: 8
---

This is the application's most complex and academically interesting part: how the user's answers become a recommended recipe. A large part of the application's core logic and purpose lives here, and it will be the heart of the finished product once both the backend and the frontend are complete.

I expect this layer to evolve and change a lot throughout the build process, especially if there is time for some of the *"nice-to-have"* user stories.

## Overview of the flow

At the moment, within my current scope, this is what the quiz flow looks like:

1. **Drink family** is a *hard filter*: only cocktails in the chosen family are considered. The filtering happens in the database.
2. **Spirit** is an *optional hard filter*: if the user picks a spirit, the candidates are filtered on it. If they pick none (null), all spirits pass through.
3. **Taste**: the user gives a score from 1 to 5 for each of the five flavor dimensions (**Sweet**, **Sour**, **Bitter**, **Salty**, **Spicy**).
4. The candidates are scored against the user's answers, and the one with the smallest distance wins.

## Candidates and filtering

Both the filters and the candidate selection happen in a single DAO call. The spirit filter is only added if the user has chosen a spirit, which is why the query is built with a `StringBuilder`. A `String` is immutable, so every `+` creates a new copy, while a `StringBuilder` is modified in place. I could have solved this in another way, but the real advantage of `StringBuilder` here is that the query is built up step by step, depending on a condition:

```java
public class CocktailDAO extends AbstractDAO<Cocktail, Long> {

    // Variable, constructor, getByIdWithDetails() and getAllWithDetails() methods
    
    public List<CocktailCandidate> getCandidatesForQuiz(Long familyId, Long spiritId) {
            try (EntityManager em = emf.createEntityManager()) {

                StringBuilder jpql = new StringBuilder(COCKTAIL_WITH_RELATIONS) // StringBuilder because there is a conditional structure
                    .append("WHERE c.cocktailFamily.id = :familyId");

                if (spiritId != null) {
                    jpql.append(" AND c.spirit.id = :spiritId"); // Conditional = only append if user chooses a spirit type
                }

                var query = em.createQuery(jpql.toString(), Cocktail.class)
                    .setParameter("familyId", familyId);

                if (spiritId != null) {
                    query.setParameter("spiritId", spiritId);
                }

                List<Cocktail> cocktails = query.getResultList();

                List<CocktailCandidate> candidates = new ArrayList<>();
                for (Cocktail cocktail : cocktails) {
                    candidates.add(new CocktailCandidate
                            (cocktail, buildFlavorProfile(cocktail)));
                }

                return candidates;
            }
    }

    // buildFlavorProfile() and rankFlavorProfile() methods
}
```

`buildFlavorProfile` runs inside the same `try` block, while the `EntityManager` is still open. The query above fetches each cocktail's spirit, liqueur, mixer and syrup, but each of those has a `flavorTags` list that is `LAZY` and therefore not loaded yet. `buildFlavorProfile` needs exactly those lists to add up the intensities, and the first time one of them is accessed, Hibernate sends an extra query to fetch it. That only works while the `EntityManager` is open. Had I called the method from the service layer instead, it would have thrown a `LazyInitializationException`.

The obvious solution would be to `JOIN FETCH` the lists in the same query, like the other relations, but Hibernate does not allow fetching several `List` collections at once (`MultipleBagFetchException`). Instead, the lists are loaded one at a time through lazy loading, which means one small extra query per ingredient.

This is the N+1 pattern I rejected in Entry 6, but here it is a deliberate trade-off. The query is already filtered by family, and possibly by spirit, so it only covers a small subset of the cocktails. Hibernate also reuses the same object within one `EntityManager`, so if ten cocktails use gin, its flavor tags are only fetched once - the number of extra queries follows the number of distinct ingredients, not the number of cocktails. And unlike `EAGER`, which would trigger extra queries everywhere a cocktail is read, this happens in one place, in one method, where it is visible in the code.

## A cocktail's flavor profile

A cocktail's profile is calculated from its ingredients in two steps. First, the intensity per flavor dimension is **summed** across all ingredients:

```java
private Map<Long, Integer> buildFlavorProfile(Cocktail cocktail) {
        Map <Long, Integer> sums = new HashMap<>();

        if (cocktail.getSpirit() != null) {
            for (SpiritFlavorTag tag : cocktail.getSpirit().getFlavorTags()) {
                sums.merge(tag.getFlavorTag().getId(), tag.getIntensity(), Integer::sum);
            }
        }

        // likewise for liqueur, mixer and syrup

        return rankFlavorProfile(sums);
}
```

`Map.merge(key, value, Integer::sum)` inserts the value if the key is new, and otherwise adds it to the existing one. I chose **sum** over average or maximum because a strong sour ingredient could otherwise be diluted away (average) or hide all the other ingredients (maximum). With the sum, I get a truer picture of a cocktail's flavor profile, because it reflects how much each dimension takes up in the drink overall.

Next, the dimensions are **ranked** by sum and given the scores 5, 4, 3, 2, 1:

```java
private Map<Long, Integer> rankFlavorProfile(Map<Long, Integer> sums){
    
    if (sums.isEmpty()) {
        return sums;
    }

    List<Map.Entry<Long, Integer>> sortedEntries = new ArrayList<>(sums.entrySet());
    sortedEntries.sort((a, b) -> {
        int compareBySum = b.getValue() - a.getValue(); // Highest sum first

        if(compareBySum != 0) {
            return compareBySum;
        }
        return a.getKey().compareTo(b.getKey()); // deterministic - if two sums are equal
    });

    Map<Long, Integer> ranked = new HashMap<>();
    int score = sortedEntries.size();

    for (Map.Entry<Long, Integer> entry : sortedEntries) {
        ranked.put(entry.getKey(), score);
        score--;
    }

    return ranked;
}
```

**A worked example:** 

    Three ingredients give the sums: 
    Sweet 14, Bitter 10, Sour 9, Salty 8 and Spicy 4. 

    After ranking, the profile becomes: 
    Sweet 5, Bitter 4, Sour 3, Salty 2, Spicy 1. 

Keeping all five dimensions (rather than only the dominant one) means the distance to the user's answers can be calculated meaningfully. Ties are resolved by `flavorTagId`, because a `HashMap` otherwise gives no guaranteed order and the result could vary between runs.

## The Strategy pattern for matching

So now I have all these methods to help compare the cocktails' profiles to the user's answers, but none of them actually does the comparing. That is because the comparison itself is isolated behind an interface in the `app/strategies` package:

```java
public interface ScoringStrategy {
    double calculateDistance(Map<Long, Integer> userAnswers, Map<Long, Integer> cocktailProfile);
}

public class EuclideanDistanceStrategy implements ScoringStrategy {

    @Override
    public double calculateDistance(Map<Long, Integer> userAnswers, Map<Long, Integer> cocktailProfile) {
        Set<Long> allFlavorTagIds = new HashSet<>();
        allFlavorTagIds.addAll(userAnswers.keySet());
        allFlavorTagIds.addAll(cocktailProfile.keySet());

        double sumOfSquares = 0;
        for (Long tagId : allFlavorTagIds){
            int userValue = userAnswers.getOrDefault(tagId, 0);
            int cocktailValue = cocktailProfile.getOrDefault(tagId, 0);
            int difference = userValue - cocktailValue;
            sumOfSquares += (double) difference * difference;
        }
        return Math.sqrt(sumOfSquares);
    }
}
```

**Euclidean distance** is the Pythagorean theorem generalized to more dimensions (here five): the square root of the sum of the squared differences. This means that a shorter distance is a better match, and 0 is a perfect match.

**An example**: 

    The user answers is: 
    Sweet 5, Sour 1, Bitter 1, Salty 1, Spicy 1.
    
    The cocktail has the profile: 
    Sweet 5, Sour 3, Bitter 4, Salty 2, Spicy 1. 
    
    The differences are: 
    0, 2, 3, 1, 0 
    
    So the distance is: 
    √(0+4+9+1+0) = √14 ≈ 3.74.

It is important to note that a cocktail always has all five flavor dimensions, because its ingredients together cover every flavor tag in my seed data. The ranking therefore always gives the scores 5 to 1. `cocktailProfile.getOrDefault(tagId, 0)` is only a safety net in case a dimension should ever be missing.

The point of the **Strategy pattern** is that `QuizService` only knows the interface. The strategy is injected from outside, so the algorithm can be swapped, if ever needed, without changing the service:

```java
public class QuizService {

    private final CocktailDAO cocktailDAO;
    private final ScoringStrategy scoringStrategy;

    public QuizService (ScoringStrategy scoringStrategy) {
        this.cocktailDAO = new CocktailDAO();
        this.scoringStrategy = scoringStrategy;
    }

    public CocktailDTO findBestMatch (QuizAnswerDTO answers) {
        List<CocktailCandidate> candidates = cocktailDAO.getCandidatesForQuiz(
                answers.getCocktailFamilyId(), answers.getSpiritId());

        if (candidates.isEmpty()) {
            throw new ApiException(HttpStatus.NOT_FOUND,
                    "No recipes match the chosen cocktail family/spirit type");
        }

        CocktailCandidate bestMatch = null;
        double bestDistance = Double.MAX_VALUE;

        for (CocktailCandidate candidate : candidates) {
            double distance = scoringStrategy.calculateDistance(
                    answers.getTasteAnswers(), candidate.getFlavorProfile());

            if (distance < bestDistance) {
                bestDistance = distance;
                bestMatch = candidate;
            }
        }

        return CocktailDTO.fromEntity(bestMatch.getCocktail());
    }
}
```

The instantiation happens in one place: `new QuizService(new EuclideanDistanceStrategy())`.
