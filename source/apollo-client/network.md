---
title: Network layer
order: 141
description: How to point your Apollo client to a different GraphQL server, or use a totally different protocol.
---

<h2 id="custom-network-interface">Custom network interface</h2>

You can define a custom network interface and pass it to the Apollo Client to send your queries in a different way. You might want to do this for a variety of reasons:

1. You want a custom transport that sends queries over Websockets instead of HTTP
2. You want to modify the query or variables before they are sent
3. You want to run your app against a mocked client-side schema and never send any network requests at all

All you need to do is create a `NetworkInterface` and pass it to the `ApolloClient` constructor.

<h3 id="NetworkInterface">interface NetworkInterface</h3>

This is the interface that an object should implement so that it can be used by the Apollo Client to make queries.

- `query(request: GraphQLRequest): Promise<GraphQLResult>` This function on your network interface is pretty self-explanatory - it takes a GraphQL request object, and should return a promise for a GraphQL result. The promise should be rejected in the case of a network error.

<h3 id="GraphQLRequest">interface GraphQLRequest</h3>

Represents a request passed to the network interface. Has the following properties:

- `query: string` The query to send to the server.
- `variables: Object` The variables to send with the query.
- `debugName: string` An optional parameter that will be included in error messages about this query. XXX do we need this?

<h3 id="GraphQLResult">interface GraphQLResult</h3>

This represents a result that comes back from the GraphQL server.

- `data: any` This is the actual data returned by the server.
- `errors: Array` This is an array of errors returned by the server.

<h2 id="query-batching">Query batching</h2>

The Apollo client supports transparently batching multiple queries in a single roundtrip when they are fired at times that are close to one another. This means, for example, that when your UI loads, there'll be a single HTTP roundtrip fired rather than one for each query. There are two ways to get batching support: use a GraphQL server implementation that supports query batching over the network transport such as Apollo server or add query batching to an existing `NetworkInterface` using query merging.

The preferred approach is to use a modern GraphQL server such as Apollo server that implements query batching over the transport. Since this batching approach leaves your query documents exactly the way they are sent from the client (i.e. multiple queries are not merged into a single query), debugging, logging, etc. are much more pleasant. If using such a GraphQL implementation is not possible, Apollo also provides query merging as a batching mechanism that is supported on all GraphQL servers.

<h3 id="BatchedNetworkInterface">interface BatchedNetworkInterface</h3>

This is the interface that an object should implement so that it can be used by Apollo Client to fetch batched queries.

- `batchQuery(request: GraphQLRequest[]): Promise<GraphQLResult[]>` This function on a batched network interface takes an array of GraphQL request objects, submits a batched request that represents each of the requests and returns a promise that is resolved with the results of each of the GraphQL requests. The promise should be rejected in case of a network error.

<h3 id="TransportBatching">Batching over the HTTP transport</h3>

Apollo client provides an implementation of query batching over the HTTP transport which will work with any GraphQL server that supports it. Single (i.e. non-batched) queries are sent to the server as the following JSON:

```
{
    query: "< query document as a string >",
    variables: "variable object ",
}
```

When several queries are batched together, the following is sent to the server:

```
{
    queries: [
        <query document 0 as string>,
        <query document 1 as string>,
        ...
    ],
    variables: [
        <variables for document 0>,
        <variables for document 1>,
        ...
    ],
}
```

<h3 id="QueryMerging">Query merging</h3>

Apollo can provide batching functionality over a standard `NetworkInterface` implementation that does not typically support batching by merging queries together under one root query and then unpacking the merged result returned by the server. To see an example of query merging within Apollo, consider the following GraphQL queries:

```graphql
query someAuthor {
    author {
        firstName
        lastName
    }
}
```

```graphql
query somePerson {
    person {
        name
    }
}
```

Assume these two queries are fired at roughly the same time (i.e. within the same tick of the batcher). Then, Apollo will internally merge these two queries into the following:

```graphql
query __composed {
    ___someAuthor___requestIndex_0__fieldIndex_0: author {
        firstName
        lastName
    }

    ___somePerson___requestIndex_1__fieldIndex_0: person {
        name
    }
}
```

Once the results are returned for this query, Apollo will take care of unpacking the merged result and present your code with data that looks like the following:

```json
{
    "data": {
        "author": {
            "firstName": "John",
            "lastName": "Smith"
        }
    }
}
```

```json
    "data": {
        "person": {
            "name": "John Smith"
        }
    }
```

This means that your client code and server implementation can remain completely oblivious to the batching that Apollo performs.

- `addQueryMerging(networkInterface: NetworkInterface): BatchedNetworkInterface` This function takes an arbitrary `NetworkInterface` implementation and returns a `BatchedNetworkInterface` that batches queries together using query merging when `batchQuery` is called on it.
