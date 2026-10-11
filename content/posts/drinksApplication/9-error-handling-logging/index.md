---
title: Entry 9 - Error Handling and Logging
date: 2026-10-06
showAuthor: true
series: ["Drinks Application"]
series_order: 9
---

A central part of our third-semester API is to handle errors accurately and throw the corresponding exceptions. We have learned how important it is to log critical moments in the code, so that errors never go unnoticed.

## A central exception class

 At the moment, all my API's business errors are thrown as `ApiException`, which carries a status code and a message and logs itself when it is created:

```java
public class ApiException extends RuntimeException {
    private int code;
    private static final Logger logger = LoggerFactory.getLogger(ApiException.class);

    public ApiException(int code, String msg){
        super(msg);
        this.code = code;
        logger.error("ApiException (code={}): {}", code, msg);
    }

    public ApiException (HttpStatus status, String msg) {
        this(status.getCode(), msg);
    }

    public int getCode(){
        return code;
    }
}
```

Because the logging sits in the constructor itself, every `throw new ApiException(...)` is automatically logged in one central place, without the services having to log themselves.

## Named status codes instead of magic numbers

I first wrote `throw new ApiException(404, ...)`, but it is hard to remember what a number means. I considered writing my own enum, but chose to import Javalin's `io.javalin.http.HttpStatus`, since the project is going to end up as a Javalin API and the enum is therefore already the right one. The extra constructor takes an `HttpStatus`, and `getCode()` still returns an `int`. The compiler now catches typos (`CONFOLICT`), which it never could with a bare number.

## What should be logged?

I deliberately do not log cases where a field is simply `null`, for example when a cocktail has no liqueur. That is expected data, not an error, and logging it would drown out the real problems in noise. The log levels follow this rule of thumb:

- `ERROR`: something failed that should not have (covered by `ApiException` and the transaction rollback in `AbstractDAO`)
- `WARN`: something unusual, but the app can continue
- `INFO`: important normal events (for example a new user created)
- `DEBUG`: detailed flow for troubleshooting

I never log `rawPassword` or `passwordHash`, not even on errors. Passwords are sensitive data, and logging them could compromise the users' security.

## Where to go from here

Error and exception handling is a topic that will be in constant development throughout the API's entire build process. Exceptions can happen in multiple places throughout the backend layers, and as the project grows, so does the risk of encountering one. Most of the Javalin and routing layer has not been built yet, and that is probably where most of the exception handling and logging will be.

I therefore expect this entry to grow considerably, with a lot more information and examples of correct exception handling.
