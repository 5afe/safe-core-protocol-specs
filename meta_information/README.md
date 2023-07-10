# Meta Information

To provide more context and information about different parts and their interactions the Safe protocol provides a way to attach meta information to contracts and interactions. The meta data is represented as `metaDataHash` fields in the different parts of the protocol and the actual content that is represented by these hashes can be stored in different locations (depending on the use case).

To make sure that this meta data can be properly parsed and handled it is important to define a standard for it.

## Meta Data Provider

To retrieve the meta data based on a `metaDataHash` a `MetaDataProvider` is required. There can be different types of providers. These providers are identified by a `type` and a `locations` (TODO: maybe a different name would be better, i.e. `connection`).

A non-exhaustive list of types are:
- `ipfs` - the `location` MUST be empty (the hash is the ipfs indentifier)
- `url` - the `location` MUST be a pattern where the metaDataHash can be substituted
- `contract` - the `location` MUST be a contract address that implements the `MetaDataProvider` contract interface


### Meta Data Provider Interface

Hashing `metaData` with `keccak256` MUST result in `metaDataHash`.

```solidity
interface MetaDataProvider {
    retrieveMetaData(bytes32 metaDataHash) external view returns (bytes memory metaData);
}
```

## Meta Data Format

The meta data format should be flexible to enable different use cases. There are different criteria that should be considered:
- Semi-human-readable (i.e. strings in a structured data set)
- Off-chain machine readable (i.e. data parsable by scripts)
- On-chain machine verifiable (i.e. contracts should be able to regenerate the hash)

Considering these criteria a good initial standard to follow is [EIP-712](https://eips.ethereum.org/EIPS/eip-712).

### EIP-712 Meta Data

[EIP-712](https://eips.ethereum.org/EIPS/eip-712) was designed for signing data and verifying it on-chain. While this is not the use case that is being covered here, it is possible to use this standard to create an easy way to verify these information on chain. Furthermore a lot of tooling around this standard already exists and further initiatives are pushed forward (i.e. to improve readability for humans of the typed data).

- Domain is used to provide more context on what meta data is represented here

TODO: we abuse the signing domain for context information, is there a better way?

Note: `name` and `version` of the domain are used to identify the meta data type as the type hash is not easy to use with a simple query interface (i.e. `MetaDataProvider`) due to nested hashing. An alternative would be that the domain object contains details about the part/context the meta information is related to (i.e. a module) and add type and version to the meta data object. 

```solidity
struct EIP712Domain {
    string name, // MUST be type name of the meta data (i.e. TransactionMetaData)
    string version, // MUST be version of the meta data (i.e. 1.0)
    uint256 chainId, // MUST be the chain where the component related to this meta data resides
    address verifyingContract, // MUST be component address related to this meta data (i.e. module address)
}
```

- Types

```solidity
struct ComponentMetaData {
    string name,
    string description,
    string version
}
```

```solidity
struct TransactionMetaData {
    string abi
}
```

TODO: this requires more explanations and examples