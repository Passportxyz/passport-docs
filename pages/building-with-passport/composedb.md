# ComposeDB

ComposeDB is a decentralized graph database built on top of Ceramic.
Whereas other databases typically store tables of information, a graph database stores relationships between nodes.
Instead of just storing keys and values, a graph database allows connectioons between values to exist too. 
This is very powerful for understanding, for example, user journeys, paths, histories and networks.

In Compose DB, the nodes of the graph are acounts or documents with a globally unique ID. The relationships between these nodes are known as "edges". You can query nodes and edges.

Being a product built on top of Ceramic, ComposeDB provides many benefits for web3 native applications including strong Ethereum integrations. There is also a rich, Graph-QL style API that helps developers to construct complex queries. These are very useful features for Gitcoin Passport.

Perhaps more importantly, ComposeDB allows us to create and publish data schemas that work across different tech-stacks, meaning we can deploy apps that work identically across multiple layer-1 blockchains and their layer-2's. Data on ComposeDB is updateable allowing, for example, Stamp expiry and reverification.

Read more in the [ComposeDB documentation](https://developers.ceramic.network/docs/composedb/getting-started).

## Passport on ComposeDB

The data used by Gitcoin Passport fits very well into the graph structure. There are two basic elements that Passport considers: user addresses and Stamps. These are nodes. The links between Stamps and users and edges. 

Let's say you want to know who owns a particular Stamp. You would identify the **node** for that Stamp and then query all the **edges** that connect the Stamp to user addresses.

> **You do not have to construct any of these queries as a Passport user! All this logic is handled by the Passport app.**

Nodes can have properties associated with them, such as `isDeleted` or `isRevoked` that clarify something about it. These can be used to filter out edges in the graph (e.g. a query can be used to identify all edges connecting a node to other nodes whose `isRevoked` value is `false`).

## Schema

Here's what a Stamp looks like in JSON format:

```json
{
  "type": [
    "VerifiableCredential", "GitcoinPassportStamp"
  ],
  "@context": [
    "https://www.w3.org/2018/credentials/v1", "https://credentials.passport.gitcoin.co/"
  ],
  "issuer": "...",
  "issuanceDate": "2023-10-23T13:32:52.935Z",
  "expirationDate": "2024-01-21T13:32:52.935Z",
  "credentialSubject": {
    "id": "did:pkh:eip155:1:0x0000000000000000000000000000000000000000",
    "hash": "v0.0.0:12121212121212121212121212121212121212",
    "provider": "MyProvider",
    "@context": [...]
  },
  "proof": {...}
}
```

and here is what it looks like to create this object in a ComposeDB database:

```bash
type GitcoinPassportStamp implements VerifiableCredential
  @createModel(
    accountRelation: LIST
    description: "A gitcoin passport stamp with a provider and hash"
  )
  @createIndex(fields: [{ path: "issuer" }])
  @createIndex(fields: [{ path: "issuanceDate" }])
  @createIndex(fields: [{ path: "expirationDate" }]) {
  _context: [String!]!
    @string(minLength: 1, maxLength: 1024)
    @list(maxLength: 1024)
  type: [String!]! @string(minLength: 1, maxLength: 1024) @list(maxLength: 1024)
  issuer: String! @string(minLength: 1, maxLength: 1024)
  issuanceDate: DateTime!
  expirationDate: DateTime
  credentialSubject: GitcoinPassportVcCredentialSubject!
  proof: GitcoinPassportVcProof!
}
```

The fields are as follows:
- `type`: Passport Stamps are always `VerifiableCredentials`
- `context`: an array of strings with lengths between 1 and 1024 bytes to store links to the specifications for verifiable credentials and the Passport credential.
- `issuer`: a string with length 1 to 1024 bytes to store the address issuing the credential.
- `issuanceDate`: A datetime object referencing the time the credential was created.
- `expirationDate`: A datetime object referencing the time the credential will expire.
- `credentialSubject`: An object with the following subfields:
  - `id`: A `did` unique identifier for this credential.
  - `hash`: A hash of this Stamp object
  - `provider`: A hash corresponding to the Stamp provider
  - `context`: A string of lenghth 1 - to 1024 bytes for storing additional information about the Stamp.
- `proof`: A hash demonstrating that the Stampo was really attested by a provider.




Here's an example of a query to retrieve 10 Passport Stamps. Users familiar with GraphQL will recognize the syntax.

```js
// Read some documents from compose
const m = `query MyQuery {
  gitcoinPassportStampIndex(
    last: 10
  ) {
    edges {
      node {
        credentialSubject {
          _id
          hash
          metaPointer
          provider
        }
        proof {
          ...GitcoinPassportStampGitcoinPassportVcProofFragment
        }
        expirationDate
        id
        issuanceDate
        issuer
      }
    }
  }
}
```
