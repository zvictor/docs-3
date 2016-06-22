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

Apollo Client supports transparently batching multiple queries in a single roundtrip when they are fired at times that are close to one another, i.e. are fired within the same 10 millisecond tick of the batcher. This means, for example, that when your UI loads, there'll be a single HTTP roundtrip fired rather than one for each query. In order to support all GraphQL servers (including those that do not implement transport-level batching), Apollo implements query batching by merging together queries that are fired within the same tick of the batcher.

To turn on query batching, you must pass `shouldBatch: true` as one of the key-value pairs to the `ApolloClient` constructor. In addition, you must set your `ApolloClient` instance's network interface to an instance that implements `BatchedNetworkInterface`. Such a `BatchedNetworkInterface` can be constructed from a standard `NetworkInterface` (i.e. one that does not support batching) using the method `addQueryMerging` from `apollo-client/networkInterface`, as described below.


<h3 id="QueryMerging">Query merging</h3>

Apollo can provide batching functionality over a standard `NetworkInterface` implementation that does not typically support batching. It does this by merging queries together under one root query and then unpacking the merged result returned by the server. To see an example of query merging within Apollo, consider the following GraphQL queries:

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
{
  "data": {
    "person": {
      "name": "John Smith"
    }
  }
}
```

This means that your client code and server implementation can remain completely oblivious to the batching that Apollo performs.

- `addQueryMerging(networkInterface: NetworkInterface): BatchedNetworkInterface` This function takes an arbitrary `NetworkInterface` implementation and returns a `BatchedNetworkInterface` that batches queries together using query merging when `batchQuery` is called on it. The function can be imported from `apollo-client/networkInterface`.


<h3 id="BatchedNetworkInterface">interface BatchedNetworkInterface</h3>

This is the interface that an object should implement so that it can be used by Apollo Client to fetch batched queries.

- `batchQuery(request: GraphQLRequest[]): Promise<GraphQLResult[]>` This function on a batched network interface that takes an array of GraphQL request objects, submits a batched request that represents each of the requests and returns a promise. This promise is resolved with the results of each of the GraphQL requests. The promise should be rejected in case of a network error.

<h3 id="BatchingExample">Batching example</h3>

To set up batching on Apollo client, we just have to create the right network interface and pass in an option to the `ApolloClient` constructor that tells it to start batching queries. In code, this looks like:

```javascript
import ApolloClient, { createNetworkInterface } from 'apollo-client';
import { addQueryMerging } from 'apollo-client/networkInterface';

const networkInterface = addQueryMerging(createNetworkInterface('/graphql', {
  credentials: 'same-origin',
}));

const apolloClient = new ApolloClient({
  networkInterface: networkInterface,
  // ... other options ...
});

// These two queries happen in quick suggestion, possibly in totally different
// places within your UI.
apolloClient.query( { query: firstQuery })
apolloClient.query( { query: secondQuery })

// You don't have to do anything special - Apollo will now take care of batching the
// queries that were fired above.
```
