---
title: "Fun DSLs in Kotlin - Builders and Receiver Types"
date: 2021-12-22T13:53:06Z
draft: false
showToc: true
TocOpen: false
tags: ["dsl", "kotlin"]
ShowPostNavLinks: true
---

We use DSLs everywhere. Every language has different patterns, so today I wanted to
share a fun Kotlin pattern that you can use for builders in your embedded DSL.

# A Convoluted Example

Let's say that we are building a scraper today to help us scrape all the passwords off
an unprotected admin page in a website. The admins have access to a lot of JSON endpoints
that look something like:

```jsonc
{
  // ...
  "passwords": [
    {
      "username": "john.doe",
      "password": "hunter2"
    }
    // ...
  ]
}
```

And ideally we wanted to scrape a whole lot of files and URLs fast, and so we came up with
the following implementation:

```kt
data class Credentials(
    val username: String,
    val password: String,
)

class JsonScraper(
    private val query: String, // a query string like ".foo.bar.passwords"
    private val urls: List<String>,
    private val files: List<String>,
) {
    fun scrape() : List<Credentials> {
        // this is magically implemented to scrape everything
        return files.map { scrapeFile(it) } + urls.map { scrapeUrls(it) }
    }
}
```

However, actually writing a quick script or program using this API is not as fun
as we had hoped. After all, we need to get the files and URLs, build the lists on
our own, and pass them into the constructor like this:

```kt
val scraper = JsonScraper(
    query = "...",
    urls = listOf(url1, url2, ..., urlN),
    files = listOf(file1, file2, ..., fileN),
)
```

For small, relatively homogeneous classes without big constructors this might be
good enough (and in those cases: great! YAGNI), but in some others we might want a
more straightforward way of managing complex object constructors.

# In comes the Builder

_Note: if you are familiar with the Builder pattern you can skip this section_

I am not the biggest fan of design patterns, but a relatively simple one that can be
used often is the Builder pattern. If we go and consult a reference page about it,
for example [this one](https://refactoring.guru/design-patterns/builder), we can see
that the intent and problem statements pretty much match our own.

We introduce a new object, called a builder, that lets us build our more complex
object step by step. Let's try to implement it in the most traditional, Java-y
way first.

```kt
class Builder(
    private val query: String
) {
    private val urls = mutableListOf<String>()
    private val files = mutableListOf<String>()

    fun withUrl(url: String) : Builder {
        urls.add(url)
        return this
    }

    fun withFile(file: File) : Builder {
        files.add(file)
        return this
    }

    fun build() = JsonScraper(query, urls, files)
}
```

So what is the advantage of this kind of pattern? We can, for example, take only the core
parameters in the constructor and safely build up the optional components afterwards. So,
in a case where we are only adding files, instead of having to explicitly set and empty
list like:

```kt
val scraper = JsonScraper(
    query = query,
    files = listOf(...),
    urls = emptyList(),
)
```

We can just ignore the URL option all together:

```
val scraper = Builder(query)
    .withFile(file1)
    .withFile(file2)
    //...
    .build()
```

This construction is also more incremental in nature, so we can apply it in cases where
we receive a stream of options (`urls.forEach { builder.withUrl(it) }`) or when we are
receiving commands from a terminal UI.

# Improving the Pattern with Lambdas

This can already work well in many cases, but there is a use case where this pattern
might not work well enough as is.

Let's imagine that we have a smart part of our code that preconfigures the builder
with certain parameters, but afterwards still wants to yield back the control to the
user to set certain parameters manually. In that case, we end up in the awkward situation
of having a half-done builder sent into our code, which we have to finish, build, and
send back into the code. Something like:

```kt
val builder = AiAssistedScraper.configBuilder() // already presets parameters
    .manualAdjustmentX()
    .manualAdjustmentY()
val scraperConfig = builder.build()
val scraper = AiAssitedScraper(config)
```

This involves a lot of boilerplate code on the user's end. Luckily, we can use first-class
functions to simplify this API. What if we just pass in a function that modifies the builder
at will, and the second-order function takes care of the instantiation, preconfiguring and
building for us?

```
fun AiAssistedScraper.buildScraper(
    manualAdjustments: (ConfigBuilder) -> Unit
) : AiAssitedScraper {
    val builder = ConfigBuilder(/* preset parameters */)
    manualAdjustments.invoke(builder)
    return scraper(builder.build())
}
```

Now the user only needs to focus on the manual adjustments:

```
val scraper = AiAssistedScraper.buildScraper {
    builder -> builder
        .manualAdjustmentX()
        .manualAdjustmentY()
}
```

**Much simpler!**

This pattern is used extensively in the wild. For example, for building database migrations on Ruby on Rails:

```rb
# From https://guides.rubyonrails.org/getting_started.html#database-migrations
class CreateArticles < ActiveRecord::Migration[7.0]
  def change
    create_table :articles do |t|
      t.string :title
      t.text :body

      t.timestamps
    end
  end
end
```

Notice how the name of the table is passed into the function as an initial argument,
and how afterwards the other parameters are set using an inline function (in Ruby they call
code blocks).

# The Cherry on Top: Receiver Types

Kotlin has one more surprise for us: [receiver types](https://kotlinlang.org/docs/lambdas.html#function-literals-with-receiver).
These are more properly called, in the language's parlance, _function literals with receiver_, and work as follows:

1.  You declare the function as having, e.g., the type `A.(X, Y) -> B`
2.  You write your lambdas in the client code: they take two arguments of type X and Y, and return type B. Moreover, there is
    an implicit environmental variable of type X that can be accessed using just a bare function call. So, if X is a string,
    you will have access to the `length` field just by just using the unqualified identifier (like a variable).
3.  You call the code in the service with `function.invoke(a, x, y)` or, like an extension method, with `a.function(x,y)`

This can make using the API even easier, as we don't need to include an explicit builder parameter in our lambda.

So how does it work in practice? Let's just build on our code from before and write a simple function:

```kt
fun buildScraper(query: String, buildAction: Builder.() -> Unit): JsonScraper {
    val builder = Builder(query)
    builder.buildAction()
    return builder.build()
}

// we can now easily build our scraper!
val scraper = buildScraper(".secret.passwords") {
    withFile("./torrentedPasswords.json")
    withUrl("https://example.com/_secret/credentials.json")
    withFile("/mnt/flashdrive/stolen_passowrds.json")
}
```

And that is it! I hope you find this pattern useful in your code adventures. This is my first blog ever,
so if you have any suggestions let me know.

_Note: You can use any code sample here, except the one from the Ruby on Rails tutorial, for any purpose and without_
_restrictions whatsoever, provided you agree that I am not liable and there is no warranty whatsoever to any_
_effects it might cause. The RoR sample is a work of its respective authors, and licensed under the_
_[CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/) license._
