# Mediator

## General Types

```solidity
struct SafeProtocolAction {
    address to,
    uint256 value,
    bytes data
}
```

Safe transaction (invoked via `call`)

```solidity
struct SafeTransaction {
    SafeProtocolAction[] actions,
    uint256 nonce,
    bytes32 metaHash
}
```

Safe root access (invoked via `delegatecall`)

```solidity
struct SafeRootAccess {
    SafeProtocolAction action,
    uint256 nonce,
    bytes32 metaHash
}
```

Both `SafeTransaction` and `SafeRootAccess` MUST have a unique id, which is the EIP-712 hash of the object.

TODO: should the Safe be part of these structs. How to better specify the hashing here.

## Interface

```solidity
interface ISafeProtocolMediator {
    function executeTransaction(Safe safe, SafeTransaction tx) external view returns (bool success, bytes memory data);
    function executeRootAccess(Safe safe, SafeRootAccess rootAccess) external view returns (bool success, bytes memory data);
}
```

TODO: How to handle return data for `executeTransaction` (as it is an array of actions)
- return data of last call
- return an array of data byes
- return no bytes on success

## Flow Charts

## Automatic Enforcements