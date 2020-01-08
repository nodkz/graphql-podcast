# ApolloClient3

## Fine Tuning Apollo Client Caching for Your Data Graph

30.10.2019
Ben Newman, Apollo Client Architect at Apollo
Video: https://www.youtube.com/watch?v=n_j8QckQN5I
Slides: https://slides.com/benjamn/fine-tuning-apollo-client-caching

## What's new in Apollo Client 3.0

- Single `@apollo/client` package
- Cache eviction and garbage collection
- Unified declarative configuration API
- In-depth pagination example
- Undeflying technologies (time permitting)

### `@apollo/client` package for everything

- apollo-client
- apollo-utilities
- apollo-cache
- apollo-cache-inmemory
- apollo-link
- apollo-link-http
- @apollo/react-hooks
- graphql-tag

```ts
import React from "react";
import { render } from "react-dom";

import {
  ApolloClient,
  InMemoryCache,
  HttpLink,
  gql,
  useQuery,
  ApolloProvider,
} from "@apollo/client";

const client = new ApolloClient({
  link: new HttpLink(...),
  cache: new InMemoryCache(...),
});

const QUERY = gql`query { ... }`;

function App() {
  const { loading, data } = useQuery(QUERY);
  return ...;
}

render(
  <ApolloProvider client={client}>
    <App />
  </ApolloProvider>,
  document.getElementById("root"),
);
```

### Cache eviction and garbage collection

- Easily the most important missing features in Apollo Client
- Tracing garbage collection, as opposed to reference counting
- Important IDs can be explicitly retained to protect against collection
- Inspiration taken from [Hermes](https://github.com/convoyinc/apollo-cache-hermes/)

#### Hermes: GraphQL Client Cache Performance

Хорошая простая статья про то, как устроена нормализация кеша и какие принципы лежат под капотом и что все это стоит 
<https://github.com/convoyinc/apollo-cache-hermes/blob/master/docs/Motivation.md>

<https://github.com/convoyinc/apollo-cache-hermes/blob/master/docs/Design%20Exploration.md#entities>:

- One interesting observation of the existing implementations is that a fair bit of cost (both in CPU and memory) comes from having to represent object references as pointers in their normalized (flat) format. Additionally, they perform this reference <-> pointer translation for all nodes in the graph.
- If that translation can be skipped for some or most of the nodes, there's potential for some large improvements in performance. A pretty clear strategy for this is to only flatten the nodes that are directly referenced throughout the application. E.g. anything with an id property, under Apollo's default configuration.
- In other words, instead of indexing every node in the graph, the cache can index business entities. Entities would be free to contain nested objects (node), where the normalization process becomes a simple object merge, and retrieval requires minimal work (entity references).

### Garbage Collection

Вызываем запрос, и потом через некоторое время еще раз:

```js
const query = gql`
  query {
    favoriteBook {
      title
      isbn
      author {
        name
      }
    }
  }
`;
```

Записываем в первый ответ, а потом записываем второй:

```js
cache.writeQuery({
  query,
  data: {
    favoriteBook: {
      __typename: 'Book',
      isbn: '9781451673319',
      title: 'Fahrenheit 451',
      author: {
        __typename: 'Author',
        name: 'Ray Bradbury',
      }
    },
  },
});

cache.writeQuery({
  query,
  data: {
    favoriteBook: {
      __typename: 'Book',
      isbn: '0312429215',
      title: '2666',
      author: {
        __typename: 'Author',
        name: 'Roberto Bolaño',
      },
    },
  },
});
```

Что у нас записывается в кэш:

```js
expect(cache.extract()).toEqual({
  ROOT_QUERY: {
    __typename: "Query",
    book: { __ref: "Book:0312429215" },
  },

  "Book:9781451673319": {
    __typename: "Book",
    title: "Fahrenheit 451",
    author: {
      __ref: 'Author:Ray Bradbury',
    },
  },

  "Author:Ray Bradbury": {
    __typename: "Author",
    name: "Ray Bradbury",
  },
  "Book:0312429215": {
    __typename: "Book",
    title: "2666",
    author: {
      __ref: "Author:Roberto Bolaño"
    },
  },

  "Author:Roberto Bolaño": {
    __typename: "Author",
    name: "Roberto Bolaño",
  },
});
```

- The same query can no longer read Fahrenheit 451 or Ray Bradbury
- In fact, the old data are now unreachable  unless we know their IDs
- What happens when we call cache.gc()?
  - Начиная с ROOT_QUERY бежит вглудь собирая `__ref` в какой-нить Set. Вторым проходом пробегается по всем ключам в нормализованном кэше и удаляет те, которых нет в Set'е.
  - `cache.retain(id)` защитит запись от удаления из GC
  - `cache.release(id)` просто снимет отметку защиты от удаления в GC
  - `cache.evict(id)` сразу удалит запись из кэша (даже если запись защищена (`retain`))

### Declarative cache configuration API

The insight of declarative programming: declare your intentions as simply as you can, and then trust the computer to find the best way of satisfying them.

#### Fragment matching and `possibleTypes`

```graphql
query {
  all_characters {
    ... on Character {
      name
    }
    ... on Jedi {
      side
    }
    ... on Droid {
      model
    }
  }
}
```

- Query fragments can have type conditions
- Interfaces can be extended by subtypes
- Union types have member types
- Also possible: sub-subtypes, unions of interface types, unions of unions
- Apollo Client knows nothing about these relationships unless told about them

Чтобы дать эту информацию, раньше мы делали так:

```graphql
query {
  __schema {
    types {
      kind
      name
      possibleTypes {
        name
      }
    }
  }
}
```

И результат отправляли в `introspectionQueryResultData`:

```js
import {
  InMemoryCache,
  IntrospectionFragmentMatcher,
} from 'apollo-cache-inmemory';

import introspectionQueryResultData from './fragmentTypes.json';

const cache = new InMemoryCache({
  fragmentMatcher: new IntrospectionFragmentMatcher({
    introspectionQueryResultData,
  }),
});
```

Но с таким подходом в код бандлилось кучу ненужной информации:

```json
{
  "data": {
    "__schema": {
      "types": [
        {
          "kind": "OBJECT",
          "name": "AddToFilmPlanetsPayload",
          "possibleTypes": null
        },
        {
          "kind": "OBJECT",
          "name": "AddToFilmSpeciesPayload",
          "possibleTypes": null
        },
        {
          "kind": "OBJECT",
          "name": "AddToFilmStarshipsPayload",
          "possibleTypes": null
        },
        {
          "kind": "INTERFACE",
          "name": "Node",
          "possibleTypes": [
            {
              "name": "Asset"
            },
            {
              "name": "Film"
            },
          ]
        },
        //...other 100500 types with "possibleTypes": null
      ]
    }
  }
}
```

Apollo Client 3.0 allows you to provide just the necessary information:

```js
const cache = new InMemoryCache({
  possibleTypes: {
    Character: ["Jedi", "Droid"],
    Test: ["PassingTest", "FailingTest", "SkippedTest"],
    Snake: ["Viper", "Python", "Asp"],
    Python: ["BallPython", "ReticulatedPython"],
  },
});
```

Можно генерить [автоматически](https://github.com/apollographql/apollo-client/blob/release-3.0/docs/source/data/fragments.md#generating-possibletypes-automatically)

#### Type policies

- An object passed to the InMemoryCache constructor via the typePolicies option
- Each key corresponds to a __typename, and each value is a TypePolicy object that provides configuration for that type
- Configuration is optional, as most types can get by with default behavior

The Apollo Client 2.x way:

```js
import {
  InMemoryCache,
  defaultDataIdFromObject,
} from 'apollo-cache-inmemory';

const cache = new InMemoryCache({
  dataIdFromObject(object) {
    switch (object.__typename) {
      case 'Product': return `Product:${object.upc}`;
      case 'Person': return `Person:${object.name}:${object.email}`;
      case 'Book': return `Book:${object.title}:${object.author.name}`;
      // By default, the ID will include the __typename and the value of the id or _id field
      default: return defaultDataIdFromObject(object);
    }
  },
});
```

The Apollo Client 3.0 way:

```js
import { InMemoryCache } from '@apollo/client';

const cache = new InMemoryCache({
  typePolicies: {
    Product: {
      keyFields: ["upc"],
    },
    Person: {
      keyFields: ["name", "email"],
    },
    Book: {
      keyFields: ["title", "author", ["name"]],
    },
  },
});
```

Behind the scenes, a typical Book ID might look like
`Book:{"title":"Fahrenheit 451","author":{"name":"Ray Bradbury"}}`

Not all data needs normalization!

```js
const cache = new InMemoryCache({
  typePolicies: {
    SearchQueryWithResults: {
      // If we want the search results to be normalized and saved,
      // we might use the query string to identify them in the cache.
      keyFields: ["query"],
    },

    SearchResult: {
      // However, the individual search result objects might not
      // have meaningful independent identities, even though they
      // have a __typename. We can store them directly within the
      // SearchQueryWithResults object by disabling normalization:
      keyFields: false,
    },
  },
});
```

#### Field policies

In GraphQL fields can receive arguments, so a field’s value cannot always be uniquely identified by its name. In other words, the cache may need to store multiple distinct values for a single field, depending on its arguments.

```graphql
query Feed($type: FeedType!, $offset: Int, $limit: Int) {
  feed(type: $type, offset: $offset, limit: $limit) {
    ...FeedEntry
  }
}
```

By default, the cache stores separate values for each unique combination of arguments, since it has no knowledge of what the arguments mean, or which ones might be important.

Since Apollo Client 1.6, the recommended solution has been to include a @connection directive in any query that requests the feed field, to specify which arguments are important:

```graphql
query Feed($type: FeedType!, $offset: Int, $limit: Int) {
  feed(type: $type, offset: $offset, limit: $limit) @connection(
    key: "feed",
    filter: ["type"]
  ) {
    ...FeedEntry
  }
}
```

This solves the field identity problem, BUT it’s repetitive! Nothing stops different feed queries from using different @connection directives, or neglecting to include a @connection directive.

Apollo Client 3.0 allows specifying key arguments in one place:

```js
const cache = new InMemoryCache({
  typePolicies: {
    Query: {
      fields: {
        feed: {
          // This configuration means feed data will be stored according to its type, ignoring any other arguments like offset and limit
          keyArgs: ["type"],
        },
      },
    },
  },
});
```

Once you provide this information to the cache, you never have to repeat it anywhere else, as it will be uniformly applied to every query that asks for the feed field within a Query type.

```js
const cache = new InMemoryCache({
  typePolicies: {
    Query: {
      fields: {
        feed: {
          keyArgs: ["type"],
          // When you exclude arguments from the field’s identity (eg `limit`, `offset`), you can still use those arguments to implement a custom read function:
          read(feedData, { args }) {
            return feedData.slice(args.offset, args.offset + args.limit);
          },
          // A custom merge function controls how new data should be combined with existing data
          merge(existingData, incomingData, { args }) {
            return mergeFeedData(existingData, incomingData, args);
          },
        },
      },
    },
  },
});
```

### Pagination revisited

- Pagination is the pattern of requesting large lists of data in multiple smaller “pages”
- Currently achieved through a combination of `@connection`, `fetchMore`, and `updateQuery`
- Apollo Client 3.0 has a new solution for this important but tricky pattern.

Что было:

```js
<Query query={BOOKS_QUERY} variables={{
  offset: 0,
  limit: 10,
}} fetchPolicy="cache-and-network">
  {({ data, fetchMore }) => (
    <Library
      entries={data.books || []}
      onLoadMore={() =>
        fetchMore({
          variables: {
            offset: data.books.length
          },
          // This code needs to be repeated anywhere books are consumed in the application
          updateQuery(prev, { fetchMoreResult }) {
            if (!fetchMoreResult) return prev;
            return Object.assign({}, prev, {
              feed: [...prev.books, ...fetchMoreResult.books]
            });
          }
        })
      }
    />
  )}
</Query>
```

With Apollo Client 3.0, you can delete `updateQuery`s code, and implement a custom merge function instead in Field policies.

```js
new InMemoryCache({
  typePolicies: {
    Query: {
      fields: {
        books: {
          // First: let the cache know you want only one copy of the books field within the Query object
          keyArgs: false, // Take full control over this field

          // Now define what happens whenever the cache receives additional books
          merge(existing: Book[], incoming: Book[], { args }) {
            const merged = existing ? existing.slice(0) : [];
            for (let i = args.offset; i < args.offset + args.limit; ++i) {
              merged[i] = incoming[i - args.offset];
            }
            return merged;
          },

          // Provide a complementary read function to specify what happens when we ask for an arbitrary range of books we already have
          read(existing: Book[], { args }) {
            // Reading needs to work when there are no existing books, by returning undefined
            return existing && existing.slice(
              args.offset,
              args.offset + args.limit,
            );
          },
        }
      }
    }
  }
})
```

И тот код, который мы написали мы можем вынести в функцию помогайку `offsetLimitPaginatedField`:

```js
new InMemoryCache({
  typePolicies: {
    Query: {
      fields: {
        books: offsetLimitPaginatedField<Book>(),
      }
    }
  }
})
```

Pagination policies can be tricky to get right, but the same approach works no matter how fancy you get: cursors, deduplication, sorting, merging, et cetera…

```js
export function offsetLimitPaginatedField<T>() {
  return {
    keyArgs: false,
    merge(existing: T[] | undefined, incoming: T[], { args }) {
      const merged = existing ? existing.slice(0) : [];
      for (let i = args.offset; i < args.offset + args.limit; ++i) {
        merged[i] = incoming[i - args.offset];
      }
      return merged;
    },
    read(existing: T[] | undefined, { args }) {
      return existing && existing.slice(
        args.offset,
        args.offset + args.limit,
      );
    },
  };
}
```

#### One more thing!

```js
import { offsetLimitPaginatedField } from "./helpers/pagination";

new InMemoryCache({
  typePolicies: {
    Query: {
      fields: {
        books: offsetLimitPaginatedField<Book>(),
        // 1. What if we also have a Query field for reading individual books by their ISBN numbers?

        // 2. A read function can intercept this field and return an existing reference from the cache
        book: {
          read(existing, { args, toReference }) {
            return existing || toReference({
              __typename: 'Book',
              isbn: args.isbn,
            });
          },
        },
      }
    }
  }
});
```

Similar to the old `cacheRedirects` API, which has been removed in Apollo Client 3.0.

## How to try the beta

- `npm install @apollo/client@beta`
- Start using `@apollo/client` exports rather than the Apollo Client 2.0 packages
- When you're ready:
  - `npm remove apollo-client apollo-utilities apollo-cache apollo-cache-inmemory apollo-link apollo-link-http react-apollo @apollo/react graphql-tag`
- Follow the
  - [Release 3.0 pull request](https://github.com/apollographql/apollo-client/pull/5116)
  - [Apollo Client Roadmap](https://github.com/apollographql/apollo-client/blob/master/ROADMAP.md)
- Documentation (can be found on the release-3.0 branch)
  - [Cache configuration](https://github.com/apollographql/apollo-client/blob/release-3.0/docs/source/caching/cache-configuration.md)
  - [Fragment matching](https://github.com/apollographql/apollo-client/blob/release-3.0/docs/source/data/fragments.md)

## Deprecation cheat sheet

| Existing feature  | Status       | Replacement          |
|-------------------|--------------|----------------------|
| fragmentMatcher   | removed      | possibleTypes        |
| dataIdFromObject  | deprecated   | keyFields            |
| @connection       | deprecated   | keyArgs              |
| cacheRedirects    | removed      | field read function  |
| updateQuery       | avoidable    | field merge function |
| resetStore        | avoidable    | GC, eviction         |
| local state       | avoidable*   | field read function  |
| separate packages | consolidated | @apollo/client       |

