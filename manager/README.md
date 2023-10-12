# Manager

At the heart of the protocol is the `Manager`, ensuring adherence to the prescribed conduct and procedures set by the `Registry`. The `Manager` serves as an intermediary layer coordinating communication and interactions between `Accounts` and `Modules`.

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
    address safe,
    SafeProtocolAction[] actions,
    uint256 nonce,
    bytes32 metadataHash
}
```

Safe root access (invoked via `delegatecall`)

```solidity
struct SafeRootAccess {
    address safe,
    SafeProtocolAction action,
    uint256 nonce,
    bytes32 metadataHash
}
```

Both `SafeTransaction` and `SafeRootAccess` MUST have a unique id, which is the EIP-712 hash of the object.

## Interface

```solidity
interface ISafeProtocolManager {
    function executeTransaction(Safe safe, SafeTransaction tx) external view returns (bytes[] memory data);
    function executeRootAccess(Safe safe, SafeRootAccess rootAccess) external view returns (bytes memory data);
}
```

To handle return data for `executeTransaction` (as it is an array of actions)
- The size of `data` must equal the size of the actions array.
- Each element in `data` corresponds to the action that has been executed i.e. `data[i]` is the result of `action[i]`.
- If execution of action(s) fail, the transaction must revert.

## Uniqueness

As mentioned before it is required that both `SafeTransaction` and `SafeRootAccess` can be uniquely identified. An example where this is important is tooling related to indexing and querying information for the modules. For this purpose a `nonce` field is present in the structs, which allows to make the hash calculated for these is unique.

**Important:** It is the responsibility of the integration (i.e. plugin) to ensure that each of structs can be uniquely identified.

## Flow Charts

### Root Access Flow

```mermaid
sequenceDiagram
    participant P as Plug In
    participant M as Protocol Manager
    participant R as Registry
    participant H as Hooks
    participant A as Safe Account
    P->>M: `executeRootAccess`
    activate M
    alt
        M->>R: `check(address integration)`
        activate R
        R-->>M: `uint256 listedAt, uint256 flaggedAt`
        deactivate R
        opt if Hook is set
            M->>H: `preCheckRootAccess(Safe safe, SafeTransaction tx, uint256 executionType, bytes executionMeta)`
            activate H
            H-->>M: `bytes preCheckData`
        end
        M->>A: `execTransactionFromModuleReturnData`
        activate A
        A-->>M: `bool success, bytes memory returnData`
        deactivate A
        opt if not success
            M--XM: `revert`
        end
            opt if Hook is set
                M->>H: `postCheck(Safe safe, SafeTransaction tx, uint256 executionType, bytes preCheckDa)`
            end
            M-->>P: `bytes data`
        deactivate H
        deactivate M
    else on any revert
        M--XP: `revert`
    end
```

## Automatic Enforcements

TBD

## Upgradeability  

It is inevitable that more features will be added to Safe{Core} Protocol (e.g. new modules). As the Manager is the central part of this setup, it is important to consider a path for integrating these new features. Using an upgradeable proxy for the Manager would introduce unacceptable security concerns. Separating too much of the functionality into separate contract to allow reusability (i.e. the list of enabled integration) would increase the gas costs, and so is also not practical. A better pattern is to allow new versions of the Manager to load information from a previous version and thereby facilitate a migration.
