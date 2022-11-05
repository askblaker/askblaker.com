---
title: Graphql library testing
date: 2022-11-04T12:00:00+00:00
tags:
  - graphql
  - rust
  - go
  - python
  - node
  - ariadne
  - strawberry
  - async-graphql
  - gqlgen
  - mercurius
---

## Testing Graphql libraries in different languages

# Graphql library testing

Testing DX and "performance" of different languages and libraries.

For the latest update see the [repo](https://github.com/askblaker/graphql-test)

| Language  | Library                                                         | Build/Install (s) | Hot reload (s) |  RPS |  kbps | 99% (ms) fake | 99% (ms) store |
| :-------- | :-------------------------------------------------------------- | ----------------: | -------------: | ---: | ----: | ------------: | -------------: |
| Python    | [Ariadne](https://ariadnegraphql.org/)                          |                 7 |              1 |   60 |  2400 |           190 |            285 |
| Python    | [Strawberry](https://strawberry.rocks/)                         |                 7 |              1 |   50 |  1800 |           340 |            340 |
| Rust      | [async-graphql](https://github.com/async-graphql/async-graphql) |                55 |              4 | 1200 | 45000 |             4 |             80 |
| Go        | [gqlgen](https://github.com/99designs/gqlgen)                   |               120 |              1 |  640 | 25600 |            42 |             34 |
| Javscript | [mercurius](https://mercurius.dev/#/)                           |                 3 |              1 |  305 | 12800 |            76 |             44 |

Disclaimer: This is probably all incorrect. Do not use for anything.

# Graphql Schema

```graphql
type Query {
  getString: String
  getStaticNumber: Int
  getPersons: [Person]!
  getFakePersons: [Person]!
}

type Mutation {
  addPerson(input: PersonInput!): Person
  itsTrue(number: Int): Boolean
}

type Person {
  name: String
  number: Int
  id: String
}

input PersonInput {
  name: String!
  number: Int!
}
```
