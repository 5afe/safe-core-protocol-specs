# Metadata

To provide more context and information about different parts and their interactions Safe{Core} Protocol provides a way to attach meta information to contracts and interactions. The metadata is represented as `metadataHash` fields in the different parts of the protocol and the actual content that is represented by these hashes can be stored in different locations (depending on the use case).

To make sure that this metadata can be properly parsed and handled it is important to define a standard for it.

## Metadata Provider

To retrieve the metadata based on a `metadataHash` a `MetadataProvider` is required. There can be different types of providers. These providers are identified by a `type` and a `locations` (TODO: maybe a different name would be better, i.e. `connection`).

A non-exhaustive list of types are:
- `ipfs` - the `location` MUST be empty (the hash is the ipfs indentifier)
- `url` - the `location` MUST be a pattern where the `metadataHash` can be substituted
- `contract` - the `location` MUST be a contract address that implements the `MetadataProvider` contract interface


### Metadata Provider Interface

Hashing `metadata` with `keccak256` MUST result in `metadataHash`.

```solidity
interface MetadataProvider {
    retrieveMetadata(bytes32 metadataHash) external view returns (bytes memory metadata);
}
```

## Metadata Format

The metadata format should be flexible to enable different use cases. There are different criteria that should be considered:
- Semi-human-readable (i.e. strings in a structured data set)
- Off-chain machine readable (i.e. data parsable by scripts)
- On-chain machine verifiable (i.e. contracts should be able to regenerate the hash)

Considering these criteria a good initial standard to follow is [EIP-712](https://eips.ethereum.org/EIPS/eip-712).

### EIP-712 Metadata

[EIP-712](https://eips.ethereum.org/EIPS/eip-712) was designed for signing data and verifying it on-chain. While this is not the use case that is being covered here, it is possible to use this standard to create an easy way to verify these information on chain. Furthermore a lot of tooling around this standard already exists and further initiatives are pushed forward (i.e. to improve readability for humans of the typed data).

- Domain is used to provide more context on what metadata is represented here

TODO: we abuse the signing domain for context information, is there a better way?

Note: `name` and `version` of the domain are used to identify the metadata type as the type hash is not easy to use with a simple query interface (i.e. `MetadataProvider`) due to nested hashing. An alternative would be that the domain object contains details about the part/context the metadata information is related to (i.e. a plugin) and add type and version to the metadata object. 

```solidity
struct EIP712Domain {
    string name, // MUST be type name of the metadata (i.e. TransactionMetadata)
    string version, // MUST be version of the metadata (i.e. 1.0)
    uint256 chainId, // MUST be the chain where the integration related to this metadata resides
    address verifyingContract, // MUST be integration address related to this metadata (i.e. plugin address)
}
```

- Types

```solidity
struct ModuleMetadata {
    string name,
    string description,
    string version
}
```

```solidity
struct TransactionMetadata {
    string abi
}
```

TODO: this requires more explanations and examples