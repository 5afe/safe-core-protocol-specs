# Modules

Modules extend the functionality of Accounts in different ways. Initial modules are `plugins`, `hooks`, `function handlers` and `signature validators`, but additional modules can be added to the Safe{Core} Protocol at a later point.

[General Types](../manager/README.md#general-types)

## Plugins

Plugins allow to add any arbitrary logic to an account such as recovery mechanisms, session keys, and automations.

Plugins can trigger transactions on an `Account` via the `Manager`.

```solidity
interface ISafeProtocolPlugIn {
    function name() external view returns (string memory name);

    function version() external view returns (string memory version);

    function metaProvider() external view returns (uint256 type, bytes memory location);

    function requiresPermissions() external view returns (uint8 permissions);
}
```

### Plugin permissions

The table below elaborates the permission types that a plugin can have. For each transaction, the `Manager` will check if the plugin has the required permission. Each permission should be granted by the account explicitly. There is no hierarchy or precedence order for the permissions.

| **Permission name**  | **Value** | **Description**                                                                                                                                                                                                                                                                                  |
|----------------------|-----------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| EXECUTE_CALL         | `1`       | Plugin can invoke `CALL` transactions through an account but, value of `to` cannot be the account itself.                                                                                                                                                                                        |
| CALL_TO_SELF         | `2`       | Plugin can invoke `CALL` transactions through an account but, value of `to` can only be the account itself. This permission is useful in cases where a plugin needs to modify the state of the account. For example, swapping owner of the account with a new owner during the recovery process. |
| EXECUTE_DELEGATECALL | `4`       | Plugin can invoke `DELEGATECALL` transactions through the account with no restriction on parameter values                                                                                                                                                                                        |

Inspired from [EIP-6617](https://eips.ethereum.org/EIPS/eip-6617), must `requiresPermissions()` returns a `uint8` that represents bit-based permissions. The manager interprets the returned value as a bit-based permission and checks if the Plugin has the required permission.

| Permission                                         | Bit Representation | uint8 Value |
|----------------------------------------------------|--------------------|-------------|
| EXECUTE_CALL                                       | 00000001           | 1           |
| CALL_TO_SELF                                       | 00000010           | 2           |
| EXECUTE_DELEGATECALL                               | 00000100           | 4           |
| EXECUTE_CALL + CALL_TO_SELF                        | 00000011           | 3           |
| EXECUTE_CALL + EXECUTE_DELEGATECALL                | 00000101           | 5           |
| CALL_TO_SELF + EXECUTE_DELEGATECALL                | 00000110           | 6           |
| EXECUTE_CALL + CALL_TO_SELF + EXECUTE_DELEGATECALL | 00000111           | 7           |

### Plugin Interface Extensions

- TBD: Gas Fee Payment Authorizer (allow 4337 and other relaying methods with plugins)

## Hooks

Hooks add additional logic at certain points of the transaction lifecycle. Hooks enable various forms of security protections such as allow- and deny-lists, MEV-protections, risk-assessments, and more. The Safe{Core} Protocol currently recognizes the following types of hooks:
- `preCheck` / `preCheckRootAccess` verify custom conditions using the state before a transaction is executed
- `postCheck` verify custom conditions at the end of a transaction and reverts 

Hooks can check any interaction done with an `Account` via the `Manager`, and also check direct (some) direct interactions on the `Account`(i.e. via the `execTransaction` flow).

```solidity
interface ISafeProtocolHooks {
    function preCheck(address account, SafeTransaction tx, uint8 executionType, bytes calldata executionMeta) external returns (bytes memory preCheckData);

    function preCheckRootAccess(address account, SafeRootAccess rootAccess, uint8 executionType, bytes calldata executionMeta) external returns (bytes memory preCheckData);

    function postCheck(address account, bool success, bytes calldata preCheckData) external;
}
```

### Parameter `executionMeta` value

| Execution Type                                                                                                                                                                                 | Value                                                                                                                                                                                        |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Multisignature Flow (for accounts that use [Guard interface](https://github.com/safe-global/safe-contracts/blob/8ffae95faa815acf86ec8b50021ebe9f96abde10/contracts/base/GuardManager.sol#L10)) | Encoded data created from parameter values received from `checkTransaction(...)` i.e. `abi.encode(to, value, data, gas, baseGas, gasPrice, gasToken, refundReceiver, signatures, msgSender)` |
| Plugin Flow                                                                                                                                                                                    | Encoded address of the Plugin i.e. `abi.encode(pluginAddress)`                                                                                                                               |

### Parameter `executionType` value

| Execution Type      | Value |
|---------------------|-------|
| Multisignature Flow | `0`   |
| Plugin Flow         | `1`   |

TODO: provide more details on execution metadata

## Function handler

Non-static version (invoked via `call`)

```solidity
interface ISafeProtocolFunctionHandler {
    function handle(address account, address sender, uint256 value, bytes calldata data) external returns (bytes memory result);
}
```

    function metadataProvider() external view returns (uint256 type, bytes memory location);

```solidity
interface ISafeProtocolStaticFunctionHandler {
    function handle(address account, address sender, uint256 value, bytes calldata data) external view returns (bytes memory result);
}
```

Kudos to @mfw78

## Signature validators

There are continuous efforts to expand the types of signatures supported by the EVM beyond the currently predominant secp256k1 elliptic curve. For example, a signature scheme gaining popularity is based on the secp256r1 elliptic curve (see EIP-7212). Signature Validators allow accounts to support new standards and enable use-cases such as Passkeys-enabled smart accounts, BLS/Schnorr or quantum-secure signatures.

References:
- https://github.com/zerodevapp/kernel/blob/main/src/validator/IValidator.sol


- EIP-712 based Signature Validator

```solidity
interface ISafeProtocol712SignatureValidator {
    /**
     * @param safe The Safe that has delegated the signature verification
     * @param sender The address that originally called the Safe's `isValidSignature` method
     * @param _hash The EIP-712 hash whose signature will be verified
     * @param domainSeparator The EIP-712 domainSeparator
     * @param typeHash The EIP-712 typeHash
     * @param encodeData The EIP-712 encoded data
     * @param payload An arbitrary payload that can be used to pass additional data to the validator
     * @return magic The magic value that should be returned if the signature is valid (0x1626ba7e)
     */
    function isValidSignature(
        address account,
        address sender,
        bytes32 _hash,
        bytes32 domainSeparator,
        bytes32 typeHash,
        bytes calldata encodeData,
        bytes calldata payload
    ) external view returns (bytes4 magic);
}
```

Kudos to @mfw78
