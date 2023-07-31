---
lastmod: 2023-05-24
old_version: true
date: 2021-11-03
linktitle: GraphQL
title: GraphQL gateway
weight: 150
images:
- /images/documentation/graphql/krakend-graphql.png
- /images/documentation/graphql/krakend-graphql-proxy.png
- /images/documentation/graphql/krakend-rest-to-graphql-transformation.png
menu:
  community_v2.3:
    parent: "050 Backends Configuration"
meta:
  noop_incompatible: true
  since: 2.0
  source: https://github.com/luraproject/lura
  namespace:
  - backend/graphql
  scope:
  - backend
---

The **GraphQL integration** allows you to work in two different modes:

1. Apply gateway functionality in the middle of a GraphQL client and its GraphQL servers (just proxy)
2. Convert REST endpoints to GraphQL calls (adapter/transformer).

KrakenD offers a simple yet powerful way of consuming GraphQL content from your distributed graphs. The main benefits of using KrakenD as a **GraphQL Gateway** are:

- Simple **GraphQL Federation**: chop your monolithic GraphQL server into different services and aggregate them in the gateway.
- User validation: Handle the authorization in the gateway before adding any load to your GraphQL servers.
- Protect and secure internal GraphQL endpoints
- Rate limit GraphQL usage
- Prepare aggregated data for external caching
- Hide complexity to the clients by providing a REST interface

## REST to GraphQL transformation
In this scenario, **the end-user consumes traditional REST content**, without even knowing that there is a GraphQL server behind:

![GraphQL](/images/documentation/graphql/krakend-rest-to-graphql-transformation.png)

**KrakenD can use the variables in the body or in the endpoint URL** to generate the final GraphQL query that will be sent to the GraphQL server. The query is loaded from an external file or declared inline in the configuration and contains any variables needing replacement with the user input.

**KrakenD acts as the GraphQL client**, negotiating with the GraphQL server the content and hiding its complexity to the end-user. The end-user consumes REST content and retrieves the data in JSON, XML, RSS, or any other format supported by KrakenD.

The configuration to consume GraphQL content from your GrapQL graphs could look like this:

{{< highlight json "hl_lines=7-16">}}
{
    "endpoint": "/marketing/{user_id}",
    "method": "POST",
    "backend": [
        {
            "timeout": "4100ms",
            "url_pattern": "/graphql?timeout=4s",
            "extra_config": {
                "backend/graphql": {
                    "type":  "mutation",
                    "query_path": "./graphql/mutations/marketing.graphql",
                    "variables": {
                        "user":"{user_id}",
                        "other_static_variables": {
                            "foo": false,
                            "bar": true
                        }
                    },
                    "operationName": "addMktPreferencesForUser"
                }
            }
        }
    ]
}
{{< /highlight >}}

The configuration for the namespace `backend/graphql` has the following structure:

{{< schema version="v2.3" data="backend/graphql.json" >}}

When using an inline `query` (as opposed to using a file from the `query_path`, which does this job automatically), make sure to use escaping when needed. Examples:
    - `"query": "{ \n find_follower(func: uid(\"0x3\")) {\n    name \n    }\n  }"`.
    - `"query": "{ q(func: uid(1)) { uid } }"`

The combination of `type` and the endpoint/backend `method` has the following behavior:

- `GET`: The query to the GQL server uses an **autogenerated query string** and can contain variables from the URL parameters **OR** the request body.
    - `method=GET` + `type=query`: Generates a query string using any `{variables}` in the endpoint, but you don't have any data in a possible body.
    - `method=GET` + `type=mutation`: Generates a query string including any variables in the body of the REST call (if present), but you cannot have `{variables}` from the URL
- `POST`: The query to the GQL server uses an **autogenerated body** containing all the variables of the URL parameters **OR** the request body. When the user and the KrakenD configuration define the same variables (collision), the user variables take preference.
    - `method=POST` + `type=query`: Generates a body using any `{variables}` in the endpoint, but it does not use the user's body to form the new body.
    - `method=POST` + `type=mutation`: Generates a body including any variables in the REST body plus the ones in the configuration, but you cannot have `{variables}` from the URL


Summarizing in a table:

|  | method=`POST` | method=`GET` |
|---|---|---|
| Type=`query` | Query string from user body | Query string from URL {params} |
| Type=`mutation` | Body from user body | Body from URL {params} |

{{< note title="Automatic content-type setting" type="tip" >}}
Since KrakenD CE v2.3.3 the content-type header the GraphQL receives is `application/json`, and is not longer needed to pass it from the client.
{{< /note >}}


## Examples of GraphQL request generation

### POST + mutation
Suppose the end-user makes the following request to KrakenD, which contains a body with a JSON containing review information to `/review/{id_show}`:

{{< terminal title="Query to graphql endpoint" >}}
curl -XPOST -d '{ "review": { "stars": 5, "commentary": "This is a great movie!" } }' http://krakend/review/1500
{{< /terminal >}}

The mutation is stored in an external file called `review.graphql` and has the following content:

```graphql
mutation CreateReviewForEpisode($ep: Episode!, $review: ReviewInput!) {
  createReview(episode: $ep, review: $review) {
    stars
    commentary
  }
}
```

The configuration of KrakenD for this example is as follows:

```json
{
    "endpoint": "/review/{id_show}",
    "method": "POST",
    "backend": [
        {
            "timeout": "4100ms",
            "url_pattern": "/graphql?timeout=4s",
            "extra_config": {
                "backend/graphql": {
                    "type":  "mutation",
                    "query_path": "./graphql/mutations/review.graphql",
                    "variables": {
                        "review": {
                            "stars": 3,
                            "commentary": "meh"
                        },
                        "ep": "JEDI",
                        "id_show": "{id_show}"
                    },
                    "operationName": "CreateReviewForEpisode"
                }
            }
        }
    ]
}
```

With the example and the configuration of KrakenD above, when the user sends a body, it will be sent as it is to the backend. However, if the user does not include any of the `variables` in the body, it will add them to the final request to the backend. So, with this example, any `review` will receive 3 stars and a "meh" comment if the end-user does not pass it.

With the cURL request in the example above, the backend receives the following JSON payload:

```json
{
    "query": "mutation CreateReviewForEpisode($ep: Episode!, $review: ReviewInput!) {\ncreateReview episode: $ep, review: $review) {\nstars\ncommentary\n}\n}\n",
    "variables": {
        "ep": "JEDI",
        "review": {
            "stars": 5,
            "commentary": "This is a great movie!"
        },
        "id_show": "{id_show}"
    },
    "operationName": "CreateReviewForEpisode"
}
```

- The `query` contains the content of the external file defining the GraphQL you want to execute.
- The `variables` section contains the following:
    - The variable `id_show` does not replace the value of `{id_show}`, as it is a POST + mutation
    - The `ep` field is passed as it is in the configuration because the user did not pass it.
    - The `review` variable is used because the POST has data, and is included in the final body, which is also passed to the GraphQL server.

### GET + mutation
In case the method is a GET, instead of a body. The configuration we will use is:
```json
{
    "endpoint": "/review/{stars}",
    "method": "GET",
    "backend": [
        {
            "timeout": "4100ms",
            "url_pattern": "/graphql?timeout=4s",
            "extra_config": {
                "backend/graphql": {
                    "type":  "mutation",
                    "query_path": "./graphql/mutations/review.graphql",
                    "variables": {
                        "review": {
                            "stars": "{stars}"
                        },
                        "ep": "JEDI",
                        "id_show": "1500"
                    },
                    "operationName": "CreateReviewForEpisode"
                }
            }
        }
    ]
}
```

And we call the endpoint like this:

{{< terminal title="Query to graphql endpoint" >}}
curl -G http://krakend/review/5
{{< /terminal >}}

In this case the GraphQL server receives an URL-encoded query with all the variables, where:

- The variable `{stars}` is replaced by its value `5` as passed in the URL
- The `review`, and `ep` fields are passed as they are in the configuration.

### POST + Query
In this example, we want to do a query instead of a mutation, and we have a `query_path` file with the following content:

```graphql
query Hero($episode: Episode, $withFriends: Boolean!) {
  hero(episode: $episode) {
    name
    friends @include(if: $withFriends) {
      name
    }
  }
}
```

And a KrakenD configuration like this:

```json
{
    "endpoint": "/hero",
    "method": "POST",
    "backend": [
        {
            "timeout": "4100ms",
            "url_pattern": "/graphql?timeout=4s",
            "extra_config": {
                "backend/graphql": {
                    "type":  "query",
                    "query_path": "./graphql/mutations/hero.graphql",
                    "variables": {
                        "episode": "unknown",
                        "withFriends": false
                    },
                    "operationName": "Hero"
                }
            }
        }
    ]
}
```

{{< terminal title="Query to graphql endpoint" >}}
curl -XPOST -d '{"episode": "JEDI"}' http://krakend/hero
{{< /terminal >}}

The GraphQL receives a body with the following content:

```json
{
    "query": "query Hero($episode: Episode, $withFriends: Boolean!) {\n  hero(episode: $episode) {\n    name\n    friends @include(if: $withFriends) {\n      name\n    }\n  }\n}",
    "variables": {
        "episode": "JEDI",
        "withFriends": false
    },
    "operationName": "Hero"
}
```
- The `episode` variable is taken from the POST as it was passed by the user
- The `withFriends` variable was not passed, so it's taken from the configuration.

### GET + Query
The final example of GET + query.

We have the following query in the `query_path` contents:

```graphql
query Hero($episode: Episode, $withFriends: Boolean!) {
  hero(episode: $episode) {
    name
    friends @include(if: $withFriends) {
      name
    }
  }
}
```

On KrakenD configuration:

```json
{
    "endpoint": "/hero/{episode}",
    "method": "GET",
    "backend": [
        {
            "timeout": "4100ms",
            "url_pattern": "/graphql?timeout=4s",
            "extra_config": {
                "backend/graphql": {
                    "type":  "query",
                    "query_path": "./graphql/mutations/hero.graphql",
                    "variables": {
                        "episode": "{episode}",
                        "withFriends": false
                    },
                    "operationName": "Hero"
                }
            }
        }
    ]
}
```

{{< terminal title="Query to graphql endpoint" >}}
curl -XGET http://krakend/hero/JEDI
{{< /terminal >}}

In this case the GraphQL server receives an URL-encoded query with all the variables, where:

- The variable `{episode}` is replaced by its value `JEDI`
- The `withFriends` field is passed as it is in the configuration.

## GraphQL gateway as a proxy
In this approach, KrakenD gets in the middle to validate or rate limit requests, but the request is forwarded to the GraphQL servers, who receive the original GraphQL query from the end user.

![Graphql](/images/documentation/graphql/krakend-graphql-proxy.png)

When working in this mode, all you need to do is to configure the GraphQL endpoint, and add as the backend your GraphQL. An example:

```json
{
    "endpoint": "/graphql",
    "method": "POST",
    "input_query_strings":[
        "query",
        "operationName",
        "variables"
    ],
    "backend": [
        {
            "timeout": "4100ms",
            "host": ["http://your-graphql.server:4000"],
            "url_pattern": "/graphql?timeout=4s"
        }
    ]
}
```

The previous example uses a set of recognized query strings to pass to the GraphQL server. You can also use `"input_query_strings":["*"]` to forward any query string. The exact configuration works with a `POST` method.

As the configuration above is not using `no-op`, you can take the opportunity to connect to more servers in the same endpoint by adding additional backend objects in the configuration.

In addition, if you'd like to use your GraphQL from a browser, like **Apollo Studio** you will need to add two additional components in your configuration:

- Accept the `OPTIONS` method adding the [flag `auto_options`](/docs/v2.3/service-settings/router-options/#auto_options)
- Add [CORS](/docs/v2.3/service-settings/cors/)

## GraphQL Federation
KrakenD's principles are working with simultaneous aggregation of data. In that sense, consuming multiple subgraphs (or back-end services) comes naturally. However, using the REST to GraphQL capabilities, you can federate data using a simple strategy: define the subgraphs in the configuration instead of moving this responsibility to the consumer.

It is a simplistic approach but still very powerful, as you can define templates with queries and let krakend aggregate the responses.

Create rest endpoints with fixed graphs you'd like to consume in the configuration. Then, in each back-end query (subgraph), you decide what transformation rules to apply, the validation, rate-limiting, etc., and even connect your endpoints with other services like queues.

The following example is a REST endpoint consuming data from 2 different subgraphs in parallel. Of course, you could add here any other KrakenD components you could need:

```json
{
    "endpoint": "/user-data/{id_user}",
    "backend": [
        {
            "timeout": "3100ms",
            "url_pattern": "/graphql?timeout=3s",
            "group": "user",
            "method": "GET",
            "host": ["http://user-graph:4000"],
            "extra_config": {
                "backend/graphql": {
                    "type":  "query",
                    "query_path": "./graphql/queries/user.graphql",
                    "variables": {
                        "user":"{user_id}"
                    },
                    "operationName": "getUserData"
                }
            }
        },
        {
            "timeout": "2100ms",
            "url_pattern": "/graphql?timeout=2s",
            "group": "user_metadata",
            "method": "GET",
            "host": ["http://metadata:4000"],
            "extra_config": {
                "backend/graphql": {
                    "type":  "query",
                    "query_path": "./graphql/queries/user_metadata.graphql",
                    "variables": {
                        "user":"{user_id}"
                    },
                    "operationName": "getUserMetadata"
                }
            }
        }
    ]
}
```