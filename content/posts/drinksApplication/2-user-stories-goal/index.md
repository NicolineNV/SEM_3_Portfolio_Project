---
title: Entry 2 - Visualize the goal - User stories
date: 2026-09-01
showAuthor: true
series: ["Drinks Application"]
series_order: 2
---

This is my 3rd semester portfolio project, so the goal is very clear:

In the first 1-10 weeks, I will build a RESTful API backend in Java.
From week 11-14, I will build a React frontend.
After all 14 weeks, I will have a fully functioning backend and frontend application.

## User Stories

To really understand where I want this application to go, one of the first things I did was to create the user stories and acceptance criteria. Without that, I knew I could too quickly get distracted by one part of the application and completely forget the bigger picture. I also knew that the product I envisioned was way more than the MVP.

To make the MVP possible, I have to first and foremost focus on the most important user stories, so for that reason, I have divided them into 2 categories: those needed for MVP, and those that are just "nice-to-have".

When I was writing my user stories, I asked myself a question: "Would it not make sense to have multiple user roles?". After all, when the app is done, an admin is needed to maintain the application and make sure that it keeps up with user demand. Therefore, I have added an admin flow to my user stories as well.

### User stories for MVP

1. **As a User**, I want to create an account and log in, so that my saved recipes and preferences are tied to my profile.
    - _Criteria: create account, log in, and see saved data._

2. **As a User**, I want to save the recipes that I like on my profile, so that I can easily find them again.
    - _Criteria: save, revisit saved recipes._

3. **As a User**, I want to answer a series of questions, so that the app can find the recipe that best suits my mood.
    - _Criteria: guide the user through the quiz flow, and use their answers to recommend a matching recipe._

4. **As a User**, I want to choose the kind of "drink family" that I want, so that the app knows what kind of recipe to recommend.
    - _Criteria: different categories, also known as "drink families," each with their own criteria._

5. **As a User**, I want to describe the taste of the drink I want, so that the recipe I get matches my preferred taste.
    - _Criteria: 5 ajustable scales from 1-5, to pinpoint preferred taste profile (Sweet, Sour, Bitter, Salt, Spicy)._

6. **As a User**, I want to choose what kind of spirit goes into my drink, so that the app can find the kind of drink I want.
    - _Criteria: able to filter through types of spirits to narrow down recipes._

7. **As a User** who has answered all the questions, I want the recommended recipe to be displayed, so that I can follow it and make my drink.
    - _Criteria: display final recipe based on the user's answers._

8. **As an Admin**, I want to log in to my account and have access to admin-only features, so that I can maintain and manage the application.
    - _Criteria: check account role at login; admin accounts get access to admin-only features._

9. **As an Admin**, I want to add new drink recipes to the database, so that the app keeps up with drinking trends and user demand.
    - _Criteria: able to create and save recipes in the application's own database._


### User stories for the "nice-to-have"

10. **As a User**, I want the app to give me recommendations for other drinks, so that I can explore more drinks I might like.
    - _Criteria: tailored recommendations based on previously saved recipes._

11. **As a User**, I want to share my liked drink recipes with other users, so that my friends and family can try the same drink as me.
    - _Criteria: able to share between profiles._

12. **As a User**, I want to choose how strong or light the drink should be, so that the drink better matches my drinking style.
    - _Criteria: able to manipulate final recipe amounts._

13. **As a User**, I want to filter recipes by ingredients I already have versus ones I'd need to buy, so that I can still get recipes even if I'm not able to go to the store.
    - _Criteria: recommended recipes depend on filtering between ingredients already owned or not._

14. **As a User**, I want to garnish my drink as I prefer, so that I get a drink that suits the mood and setting.
    - _Criteria: able to apply garnish to the final recipe as an extra category, not necessarily included in the recipe by default._

15. **As a User**, I want to rate the recipe by giving it 1-5 stars, so that the app better knows which kind of drinks I like.
    - _Criteria: able to submit a 1-5 star rating per recipe, used to refine future recommendations._

16. **As a User**, I want to browse all recipes without going through the quiz, so that I can explore freely when I already know what I want.
    - _Criteria: display all recipes without requiring the user to answer questions._

17. **As an Admin**, I want to view and search for user profiles, so that I can manage and help users when needed.
    - _Criteria: display all users, search through user accounts by ID, phone number or e-mail._

18. **As an Admin**, I want to see which drinks are trending and their ratings, so that I can examine user drinking trends.
    - _Criteria: see top 10 most popular drinks by users' average rating._