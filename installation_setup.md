## Setup
### Pre-requisites
Please refer to the [set up](https://docs.corda.net/getting-set-up.html) instructions.

### Build step
To build jar use command `./gradelw jar` .This will generate the following files

| File | Description                                                     |
|-----------|-----------------------------------------------------------------|
| `did-contracts-1.0-SNAPSHOT.jar` | contains state and contracts                  |
| `did-flows-1.0-SNAPSHOT.jar` |  contains flows for read                 |
| `did-witness-flows-1.0-SNAPSHOT.jar`| contains flows for create,update,delete




### Running the nodes
The nodes need to have a config file with list of witness nodes. Example configuration shown below
```bash
nodes = [
"O=PartyB,L=New York,C=US"
]

notary = "O=Notary,L=London,C=GB"
```

## Developers can perform CRUD operations in two ways:
* REST APIs
* Flows

### Corda DID REST APIs ([`did-api`](did-api))

The DID API is the server component to be deployed by consortium member nodes in conjunction with the CorDapp.
It provides Method specific APIs to _create_, _read_, _update_ or _delete_ DID documents.

The Corda DID method achieves proof-of-ownership of a *document* by requiring proof-of-ownership of the *keys* contained in the document.

To implement that, any DID document must be be wrapped in an envelope.
This envelope must contain signatures by all private keys associated with the public keys contained in the documents.

![Corda DID API](did_envelope.svg)

Envelopes that do not contain signatures for all public keys will be rejected.
Envelopes using unsupported cryptographic suites or unsupported serialisation mechanisms will be rejected.
In the current implementation there are severe restrictions on which suites and serialisation mechanisms can be used (see _Caveats_ below).

#### API Format

##### Instruction

Instruction data tells the API what to do with the document received.
It also contains proof of ownership of keys.
The instruction data is to be formatted according to the following schema:

```json
{
  "definitions": {},
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": [
    "action",
    "signatures"
  ],
  "properties": {
    "action": {
      "$id": "#/properties/action",
      "type": "string",
      "enum": [
        "create",
        "read",
        "update",
        "delete"
      ]
    },
    "signatures": {
      "$id": "#/properties/signatures",
      "type": "array",
      "items": {
        "$id": "#/properties/signatures/items",
        "type": "object",
        "required": [
          "id",
          "type",
          "signatureBase58"
        ],
        "properties": {
          "id": {
            "$id": "#/properties/signatures/items/properties/id",
            "type": "string",
            "description": "The ID of the public key that is part of the key pair signing the document.",
            "examples": [
              "did:corda:testnet:3df6b0a1-6b02-4053-8900-8c36b6d35fa1#keys-1",
              "did:corda:tcn:3df6b0a1-6b02-4053-8900-8c36b6d35fa1#keys-2"
            ],
            "pattern": "^did:corda:(testnet|tcn-uat|tcn):[0-9a-f]{8}\\b-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-\\b[0-9a-f]{12}#.+$"
          },
          "type": {
            "$id": "#/properties/signatures/items/properties/type",
            "type": "string",
            "description": "The cryptographic suite this key has been generated with. More formats (RsaSignature2018, EdDsaSASignatureSecp256k1) to follow.",
            "enum": [
              "Ed25519Signature2018"
            ]
          },
          "signatureBase58": {
            "$id": "#/properties/signatures/items/properties/signatureBase58",
            "type": "string",
            "description": "The binary signature in Base58 representation. More formats to follow.",
            "examples": [
              "54CnhKVqE63rMAeM1b8CyQjL4c8teS1DoyTfZnKXRvEEGWK81YA6BAgQHRah4z1VV4aJpd2iRHCrPoNTxGXBBoFw"
            ]
          }
        }
      }
    }
  }
}
```

i.e.:

```json
{
  "action": "create",
  "signatures": [
    {
      "id": "did:corda:tcn:d51924e1-66bb-4971-ab62-ec4910a1fb98#keys-1",
      "type": "Ed25519Signature2018",
      "signatureBase58": "54CnhKVqE63rMAeM1b8CyQjL4c8teS1DoyTfZnKXRvEEGWK81YA6BAgQHRah4z1VV4aJpd2iRHCrPoNTxGXBBoFw"
    }
  ]
}
```

##### Document

The format of the document follows the [Data Model and Syntaxes for Decentralized Identifiers Draft Community Group Report 06 February 2019](https://w3c-ccg.github.io/did-spec) in JSON-LD.

#### Methods

Envelopes are implemented as `multipart/form-data` HTTP requests with two parts:

| Key           | Value            |
|---------------|------------------|
| `instruction` | Instruction JSON |
| `document`    | DID JSON         |

This format is chosen to circumvent issues with canonical document representation for hashing.

##### Create (`PUT {did}`)

This is used to create a new DID.
Proof of ownership of the document has to be presented in the envelope.

Payload includes:
- The document consisting of the encoded public key,type of public key,controller of public key.
- The instruction consisting of action to perform (create), encoded signature on the document and type of signature.

Instruction:

```json
{
  "action": "create",
  "signatures": [
	{
	  "id": "did:corda:tcn:a609bcc0-a3a8-11e9-b949-fb002eb572a5#keys-1",
	  "type": "Ed25519Signature2018",
	  "signatureBase58": "2M12aBn5ijmmUyHtTf56NTJsUEUbpbqbAgpsvxsfMa2KrL5MR5rGb4dP37QoyRWp94kqreDMV9P4K3QHfE67ypTD"
	}
  ]
}
```

Document:

```json
{
  "@context": "https://w3id.org/did/v1",
  "id": "did:corda:tcn:a609bcc0-a3a8-11e9-b949-fb002eb572a5",
  "publicKey": [
	{
	  "id": "did:corda:tcn:a609bcc0-a3a8-11e9-b949-fb002eb572a5#keys-1",
	  "type": "Ed25519VerificationKey2018",
	  "controller": "did:corda:tcn:a609bcc0-a3a8-11e9-b949-fb002eb572a5",
	  "publicKeyBase58": "GfHq2tTVk9z4eXgyNRg7ikjUaaP1fuE4Ted3d6eBaYSTxq9iokAwcd16hu8v"
	}
  ]
}
```

HTTP Request:


```bash
curl -X PUT \
http://example.org/did:corda:tcn:a609bcc0-a3a8-11e9-b949-fb002eb572a5 \
  -H 'content-type: multipart/form-data' \
  -F instruction='{
  "action": "create",
  "signatures": [
	{
	  "id": "did:corda:tcn:a609bcc0-a3a8-11e9-b949-fb002eb572a5#keys-1",
	  "type": "Ed25519Signature2018",
	  "signatureBase58": "2M12aBn5ijmmUyHtTf56NTJsUEUbpbqbAgpsvxsfMa2KrL5MR5rGb4dP37QoyRWp94kqreDMV9P4K3QHfE67ypTD"
	}
  ]
}' \
  -F document'={
  "@context": "https://w3id.org/did/v1",
  "id": "did:corda:tcn:a609bcc0-a3a8-11e9-b949-fb002eb572a5",
  "created":"2019-07-11T10:27:27.326Z",
  "publicKey": [
	{
	  "id": "did:corda:tcn:a609bcc0-a3a8-11e9-b949-fb002eb572a5#keys-1",
	  "type": "Ed25519VerificationKey2018",
	  "controller": "did:corda:tcn:a609bcc0-a3a8-11e9-b949-fb002eb572a5",
	  "publicKeyBase58": "GfHq2tTVk9z4eXgyNRg7ikjUaaP1fuE4Ted3d6eBaYSTxq9iokAwcd16hu8v"
	}
  ]
}'
```

Response:

 - The API will respond with status `200` for a request with a well-formed instruction *and* a well-formed document *and* valid signature(s) *and* an unused ID.
 - The API will respond with status `400` for a request with a deformed instruction *or* a deformed document *or* at least one invalid signature.
 - The API will respond with status `409` for a request with an ID that is already taken.

##### Read (`GET {did}`)

A simple `GET` request specifying the id as fragment is used to retrieve a DID document.The DID document contains a list of public keys, the type of the public key,information about encodings used on those public keys and the controller of each public key.

HTTP Request:

```bash
curl -X GET http://example.org/did:corda:tcn:a609bcc0-a3a8-11e9-b949-fb002eb572a5

```
Response:
``` bash
{
"@context":"https://w3id.org/did/v1",
"id":"did:corda:tcn:a609bcc0-a3a8-11e9-b949-fb002eb572a5",
"created":"2019-07-11T10:27:27.326Z",
"publicKey":[
        {
            "id":"did:corda:tcn:a609bcc0-a3a8-11e9-b949-fb002eb572a5#keys-1",
            "type":"Ed25519VerificationKey2018",
            "controller":"did:corda:tcn:a609bcc0-a3a8-11e9-b949-fb002eb572a5",
            "publicKeyBase58":"GfHq2tTVk9z4eXgyNRg7ikjUaaP1fuE4Ted3d6eBaYSTxq9iokAwcd16hu8v"
        }
    ]
}
```


 - The API will respond with status `200` for a request with a known ID.
 - The API will respond with status `404` for a request with an unknown ID.
 - The API will respond with status `400` for a request with an ID with incorrect format.

##### Update (`POST {did}`)

Updates use the optional [created](https://w3c-ccg.github.io/did-spec/#created-optional) and [updated](https://w3c-ccg.github.io/did-spec/#updated-optional) concepts to mitigate replay attacks.
This means an update will only be successful if the `updated` field *in the DID document* is set to an instant that is later than the instant previously saved with that field.
Should no previous update be recorded, the update will only be successful if the `created` field *in the document* is set to an instant that is later than the instant provided with the update.

The calculation of the current time is done by the DID owner without verification of its accuracy by the consortium.
This is appropriate since this field is only used to determine a before/after relationship.
Consumers of the DID document need to take into account that this value is potentially inaccurate.

Payload includes:
- The document consisting of the new encoded public key,type of public key,controller of public key.
- The instruction consisting of action to perform (update), encoded signature(s) on this document using all private keys(including the one being added) assosiated with the public keys in the document and type of signature.

HTTP request:

```bash
curl -X POST \
http://example.org/did:corda:tcn:a609bcc0-a3a8-11e9-b949-fb002eb572a5 \
  -H 'content-type: multipart/form-data' \
  -F instruction='{
  "action": "update",
  "signatures": [
    {
        "id":"did:corda:tcn:a609bcc0-a3a8-11e9-b949-fb002eb572a5#keys-1",
        "type":"Ed25519Signature2018",
        "signatureBase58":"57HQXkem7pXpfHnP3DPTyLqSQB9NuZNj7V4hS61kbkQA28hCuYtSmFQCABj8HBX2AmDss13iDkNY2H3zqRZsYnD4"
    },
    {
        "id":"did:corda:tcn:a609bcc0-a3a8-11e9-b949-fb002eb572a5#keys-2",
        "type":"Ed25519Signature2018",
        "signatureBase58":"26kkhZbQLSNvEKbPvx18GRfSoVMu2bDXutvnWcQQyrGxqz5VKijkFV2GohbkbafPa2WqVad7wnyLwx1zxjvVfvSa"
    }
  ]
}' \
  -F document'={
  "@context": "https://w3id.org/did/v1",
  "id": "did:corda:tcn:a609bcc0-a3a8-11e9-b949-fb002eb572a5",
  "created":"2019-07-11T10:27:27.326Z",
  "updated":"2019-07-11T10:29:15.116Z",
  "publicKey": [
	{
	  "id": "did:corda:tcn:a609bcc0-a3a8-11e9-b949-fb002eb572a5#keys-2",
	  "type": "Ed25519VerificationKey2018",
	  "controller": "did:corda:tcn:a609bcc0-a3a8-11e9-b949-fb002eb572a5",
	  "publicKeyBase58": "GfHq2tTVk9z4eXgyHhSTmTRf4NFuTv7afqFroA8QQFXKm9fJcBtMRctowK33"
	}
  ]
}'
```

 - The API will respond with status `200` if update is successful.
 - The API will respond with status `404` for a request with an unknown ID.
 - The API will respond with status `400` for other cases of incorrect payload(mismatched signatures,malformed document,instruction etc.).
 
 

##### Delete (`DELETE {did}`)

This method is used to disable the identity on the ledger.Once deleted the identity cannot be used again.Delete accepts only instruction as payload , the instruction contains signature for the public on the latest DID document from the ledger.

Payload includes:
- The instruction consisting of action to perform (delete), encoded signature on the latest DID document on the ledger using all private keys assosiated with public keys present in the document and type of the signature.

HTTP request:

```bash
curl -X DELETE \
http://example.org/did:corda:tcn:a609bcc0-a3a8-11e9-b949-fb002eb572a5 \
  -H 'content-type: multipart/form-data' \
  -F instruction='{
  "action": "delete",
  "signatures": [
	{
	 "id":"did:corda:tcn:a609bcc0-a3a8-11e9-b949-fb002eb572a5#keys-2",
	 "type":"Ed25519Signature2018",
	 "signatureBase58":"26kkhZbQLSNvEKbPvx18GRfSoVMu2bDXutvnWcQQyrGxqz5VKijkFV2GohbkbafPa2WqVad7wnyLwx1zxjvVfvSa"
	}
  ]
}'
```


 - The API will respond with status `200` if delete is successful.
 - The API will respond with status `404` for a request with an unknown ID.
 - The API will respond with status `400` for other cases of incorrect payload(mismatched signatures,malformed instruction etc.).

### Corda DID Flows ([`did-witness-flows`](did-witness-flows))
The DID Flows is the CorDapp component to be deployed by consortium member nodes. It provides Method specific flows to create, read, update or delete DID documents.
The DID flows can be invoked from RPC client or from another flows.

##### Create (`CreateDidFlow`)
This is used to create a new DID.
Proof of ownership of the document has to be presented in the envelope as outlined in the API format.

* invoke CreateDidFlow via RPC:
```rpc.startFlowDynamic(CreateDidFlow::class.java, envelope)```

* invoke CreateDidFlow from another flow:  
```subFlow(CreateDidFlow(envelope))```

where envelope is an instance of type `DidEnvelope`

##### Read (`FetchDidDocumentFlow`)
This is used to fetch a did document from node's local vault. It returns an instance of type `DidDocument`

* invoke FetchDidDocumentFlow via RPC:
```rpc.startFlowDynamic(FetchDidDocumentFlow::class.java, linearId)```

* invoke FetchDidDocumentFlow from another flow:  
```subFlow(FetchDidDocumentFlow(linearId))```

where linearId is an instance of type `UniqueIdentifier` and it is the UUID part of the did.

There might be a case where a node which is not part of the DID Business Network may request DID document from one of the DID consortium nodes.
In such situations, nodes can invoke `FetchDidDocumentFromRegistryNodeFlow` defined in ([`did-flows`](did-flows)) module.

* invoke FetchDidFetchDidDocumentFromRegistryNodeFlowDocumentFlow via RPC:
```rpc.startFlowDynamic(FetchDidDocumentFromRegistryNodeFlow::class.java, didRegistryNode, linearId)```

* invoke FetchDidDocumentFromRegistryNodeFlow from another flow:  
```subFlow(FetchDidDocumentFromRegistryNodeFlow(didRegistryNode, linearId))```

where linearId is an instance of type `UniqueIdentifier` and it is the UUID part of the did && didRegistryNode is an instance of type `Party` representing the did consortium node. 

##### Update (`UpdateDidFlow`)
This is used to update an existing DID.

* invoke UpdateDidFlow via RPC:
```rpc.startFlowDynamic(UpdateDidFlow::class.java, envelope)```

* invoke UpdateDidFlow from another flow:  
```subFlow(UpdateDidFlow(envelope))```

where envelope is an instance of type `DidEnvelope`

##### Delete (`DeleteDidFlow`)
This is used to disable an existing DID. Delete operation introduces no changes to the DidDocument. It only updates the DidState with status `Deleted` 

* invoke DeleteDidFlow via RPC:    
```rpc.startFlowDynamic(DeleteDidFlow::class.java, instruction, did)```

* invoke DeleteDidFlow from another flow:  
```subFlow(DeleteDidFlow(instruction, did))```

where instruction is the instruction JSON object (in string form) containing signatures of did-owner on the did-document to be deactivated
&& did is the did to be deleted.