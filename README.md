# Example of Kotlin Spring Boot GraphQL Server with Spring Security authentication and authorization along with Kotlin GraphQL DSL Client generated by means of [Kobby Gradle Plugin](https://github.com/ermadmi78/kobby) from GraphQL Schema

1. See GraphQL schema [here](https://github.com/ermadmi78/kobby-gradle-example/blob/main/cinema-api/src/main/resources/io/github/ermadmi78/kobby/cinema/api/cinema.graphqls)
1. Start server: `./gradlew :cinema-server:bootRun`
1. Try to execute GraphQL queries in [console](http://localhost:8080/graphiql) (for example `query { countries { id name } }`)
1. Start client: `./gradlew :cinema-kotlin-client:bootRun`
1. See queries, generated by Kobby DSL in client output
1. See client source code [here](https://github.com/ermadmi78/kobby-gradle-example/blob/main/cinema-kotlin-client/src/main/kotlin/io/github/ermadmi78/kobby/cinema/kotlin/client/application.kt)
1. Just try to write your own query by means of Kobby DSL!

# Spring Security authorization support

See example of GraphQL resolvers authorization [here](https://github.com/ermadmi78/kobby-gradle-example/blob/main/cinema-server/src/main/kotlin/io/github/ermadmi78/kobby/cinema/server/resolvers/QueryResolver.kt)

See example of GraphQL server API tests [here](https://github.com/ermadmi78/kobby-gradle-example/blob/main/cinema-server/src/test/kotlin/io/github/ermadmi78/kobby/cinema/server/CinemaServerTest.kt)

**Coroutine based resolver authorization example:**

```kotlin
suspend fun country(id: Long): CountryDto? = hasAnyRole("USER", "ADMIN") {
    println("Query country by user [${authentication.name}] in thread [${Thread.currentThread().name}]")
    dslContext.selectFrom(COUNTRY)
        .where(COUNTRY.ID.eq(id))
        .fetchAny { it.toDto() }
}
```

**Mono based resolver authorization example:**

```kotlin
@PreAuthorize("hasRole('USER') or hasRole('ADMIN')")
fun film(id: Long): Mono<FilmDto?> = mono(resolverDispatcher) {
    val authentication = getAuthentication()!!
    println("Query film by user [${authentication.name}] in thread [${Thread.currentThread().name}]")

    dslContext.selectFrom(FILM)
        .where(FILM.ID.eq(id))
        .fetchAny { it.toDto() }
}
```

**Flux based resolver authorization example:**

```kotlin
@PreAuthorize("hasRole('USER') or hasRole('ADMIN')")
fun countries(
    name: String?,
    limit: Int,
    offset: Int
): Flux<CountryDto> = flux(resolverDispatcher) {
    val authentication = getAuthentication()!!
    println("Query countries by user [${authentication.name}] in thread [${Thread.currentThread().name}]")

    var condition: Condition = trueCondition()

    if (!name.isNullOrBlank()) {
        condition = condition.and(COUNTRY.NAME.containsIgnoreCase(name.trim()))
    }

    dslContext.selectFrom(COUNTRY)
        .where(condition)
        .limit(offset.prepare(), limit.prepare())
        .fetch { it.toDto() }
        .forEach {
            send(it)
        }
}
```

# Kotlin GraphQL Client DSL support

GraphQL Client DSL in this example is generated by means of [Kobby Gradle Plugin](https://github.com/ermadmi78/kobby) from [GraphQL Schema](https://github.com/ermadmi78/kobby-gradle-example/blob/main/cinema-api/src/main/resources/io/github/ermadmi78/kobby/cinema/api/cinema.graphqls).
See source code [here](https://github.com/ermadmi78/kobby-gradle-example/blob/main/cinema-kotlin-client/src/main/kotlin/io/github/ermadmi78/kobby/cinema/kotlin/client/application.kt). 

## Simple GraphQL query example

**GraphQL query:**

```graphql
{
  country(id: 1) {
    id
    name
  }
}
```

**Kotlin DSL query:**

```kotlin
context.query {
    country(1) {
        // id is primary key (see @primaryKey directive in schema)
        // name is default (see @default directive in schema)
    }
}
```

## Complex GraphQL query example

**GraphQL query:**

```graphql
{
  country(id: 7) {
    id
    name
    films(title: "d") {
      id
      title
      genre
      countryId
      actors(limit: -1) {
        id
        firstName
        lastName
        birthday
        gender
        countryId
        country {
          id
          name
        }
      }
    }
    actors(firstName: "d") {
      id
      fields(keys: ["birthday", "gender"])
      firstName
      lastName
      birthday
      gender
      countryId
      films(limit: -1) {
        id
        title
        countryId
      }
    }
  }
}
```

**Kotlin DSL query:**

```kotlin
context.query {
    country(7) {
        // id is primary key
        // name is default
        films {
            title = "d" // title is selection argument (see @selection directive in schema)

            // id is primary key
            // title is default
            genre()
            // countryId is required (see @required directive in schema)
            actors {
                limit = -1 // limit is selection argument (see @selection directive in schema)

                // id is primary key
                // firstName is default
                // lastName is default
                // birthday is required (see @required directive in schema)
                gender()
                // countryId is primary key
                country {
                    // id is primary key
                    // name is default
                }
            }
        }
        actors {
            firstName = "d" // firstName is selection argument (see @selection directive in schema)

            // id is primary key
            fields {
                keys = listOf(
                    "birthday",
                    "gender"
                ) // keys is selection argument (see @selection directive in schema)
            }
            // firstName is default
            // lastName is default
            // birthday is required
            gender()
            // countryId is primary key
            films {
                limit = -1 // limit is selection argument (see @selection directive in schema)

                // id is primary key
                // title is default
                // countryId is required
            }
        }
    }
}
```

## GraphQL unions query example

**GraphQL query:**

```graphql
{
  country(id: 17) {
    id
    native {
      __typename
      ... on Film {
        id
        title
        genre
        countryId
      }
      ... on Actor {
        id
        firstName
        lastName
        birthday
        gender
        countryId
        country {
          id
          name
        }
      }
    }
  }
}
```

**Kotlin DSL query:**

```kotlin
context.query {
    country(17) {
        __minimize() // switch off defaults to minimize query
        // id is primary key
        native {
            // __typename generated by Kobby
            __onFilm {
                // id is primary key
                // title is default
                genre()
                // countryId is required
            }
            __onActor {
                // id is primary key
                // firstName is default
                // lastName is default
                // birthday is required
                gender()
                // countryId is primary key
                country {
                    // id is primary key
                    // name is default
                }
            }
        }
    }
}
```

## Customized API

You can customize the generated GraphQL DSL by means of
Kotlin [extension functions](https://kotlinlang.org/docs/extensions.html). Note that all generated entities implements
DSL Context interface. So each entity is an entry point for executing GraphQL queries.

**First, let extend our DSL Context (source code
see [here](https://github.com/ermadmi78/kobby-gradle-example/blob/main/cinema-api/src/main/kotlin/io/github/ermadmi78/kobby/cinema/api/kobby/kotlin/_CinemaContext.kt)):**

```kotlin
suspend fun CinemaContext.findCountry(
    id: Long, 
    __projection: CountryProjection.() -> Unit = {}
): Country? =
    query {
        country(id, __projection)
    }.country

suspend fun CinemaContext.fetchCountry(
    id: Long, 
    __projection: CountryProjection.() -> Unit = {}
): Country = 
    findCountry(id, __projection)!!
```

**Second, let extend our Country entity (source code
see [here](https://github.com/ermadmi78/kobby-gradle-example/blob/main/cinema-api/src/main/kotlin/io/github/ermadmi78/kobby/cinema/api/kobby/kotlin/entity/_Country.kt)):**

```kotlin
suspend fun Country.refresh(
    __projection: (CountryProjection.() -> Unit)? = null
): Country = query {
    country(id) {
        __projection?.invoke(this) ?: __withCurrentProjection()
    }
}.country!!

suspend fun Country.findFilm(
    id: Long, 
    __projection: FilmProjection.() -> Unit = {}
): Film? = refresh {
    __minimize() // switch off all default fields to minimize GraphQL response
    film(id, __projection)
}.film
```

**Ok, we are ready to use our customized API:**

```kotlin
 // Fetch country by id
val country = context.fetchCountry(7)
println("Country: id=${country.id} name='${country.name}'")

//Find all country films
val films = country.findFilms {
    limit = -1
    genre()
}

films.forEach {
    println("Film: id=${it.id}, title='${it.title}' genre=${it.genre}")
}
```

**This Kotlin DSL code will produce two GraphQL queries:**

```graphql
{
  country(id: 7) {
    id
    name
  }
}
```

```graphql
{
  country(id: 7) {
    id
    films(limit: -1) {
      id
      title
      genre
      countryId
    }
  }
}
```

**More sophisticated example of API customization see
in [API tests](https://github.com/ermadmi78/kobby-gradle-example/blob/main/cinema-server/src/test/kotlin/io/github/ermadmi78/kobby/cinema/server/CinemaServerTest.kt)
.**

The tests `createCountryWithFilmAndActorsByMeansOfGeneratedAPI`
and `createCountryWithFilmAndActorsByMeansOfCustomizedAPI`
implements the same scenario by means of native generated API and by means of customized API. You can compare these two
test cases to see that the customized API significantly improves the readability of your code.
