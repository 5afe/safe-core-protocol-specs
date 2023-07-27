# Integrations

Integrations extend the functionality of Accounts in different ways. Initial integrations are `plugins`, `hooks`, `function handlers` and `signature verifiers`, but additional integrations can be added to the Safe{Core} Protocol at a later point.

[General Types](../manager/README.md#general-types)

## Plugins

Plugins allow to add any arbitrary logic to an account such as recovery mechanisms, session keys, and automations.

Plugins can trigger transactions on an `Account` via the `Manager`.

```solidity
interface ISafeProtocolPlugIn {
    function name() external view returns (string memory name);

    function version() external view returns (string memory version);

    function metaProvider() external view returns (uint256 type, bytes memory location);

    function requiresRootAccess() external view returns (bool requiresRootAccess);
}
```

### Plugin Interface Extensions

- TBD: Gas Fee Payment Authorizer (allow 4337 and other relaying methods with plugins)

## Hooks

Hooks add additional logic at certain points of the transaction lifecycle. Hooks enable various forms of security protections such as allow- and deny-lists, MEV-protections, risk-assessments, and more. The Safe{Core} Protocol currently recognizes the following types of hooks:
- `preCheck` / `preCheckRootAccess` verify custom conditions using the state before a transaction is executed
- `postCheck` verify custom conditions at the end of a transaction and reverts 

Hooks can check any interaction done with an `Account` via the `Manager`, and also check direct (some) direct interactions on the `Account`(i.e. via the `execTransaction` flow).

```solidity
interface ISafeProtocolHooks {
    function preCheck(Safe safe, SafeTransaction tx, uint256 executionType, bytes calldata executionMeta) external returns (bytes memory preCheckData);

    function preCheckRootAccess(Safe safe, SafeRootAccess rootAccess, uint256 executionType, bytes calldata executionMeta) external returns (bytes memory preCheckData);

    function postCheck(Safe safe, bool success, bytes calldata preCheckData) external;
}
```

Execution types:
- Multisignature Flow
- Plugin Flow

TODO: provide more details on execution type and execution metadata

## Function handler

Non-static version (invoked via `call`)

```solidity
interface ISafeProtocolFunctionHandler {
    function handle(Safe safe, address sender, uint256 value, bytes calldata data) external returns (bytes memory result);
}
```

    function metadataProvider() external view returns (uint256 type, bytes memory location);

```solidity
interface ISafeProtocolStaticFunctionHandler {
    function handle(Safe safe, address sender, uint256 value, bytes calldata data) external view returns (bytes memory result);
}
```

Kudos to @mfw78

## Signature verifiers

Signature Verifiers leverage the underutilized ERC-1271 standard, enabling customizable logic for signature verification. Signature Verifiers incorporate off-chain functions, increasing the versatility of smart accounts. An example of this approach is [CowSwap’s Time-Weighted Average Price (TWAP)](https://github.com/rndlabs/cowswap-twap-orders) integration.

References:
- https://github.com/zerodevapp/kernel/blob/main/src/validator/IValidator.sol


- EIP-712 based Signature Verifier

```solidity
interface ISafeProtocol712SignatureVerifier {
    /**
     * @dev If called by `SignatureVerifierMuxer`, the following has already been checked:
     *      _hash = h(abi.encodePacked("\x19\x01", domainSeparator, h(typeHash || encodeData)));
     * @param safe The Safe that has delegated the signature verification
     * @param sender The address that originally called the Safe's `isValidSignature` method
     * @param _hash The EIP-712 hash whose signature will be verified
     * @param domainSeparator The EIP-712 domainSeparator
     * @param typeHash The EIP-712 typeHash
     * @param encodeData The EIP-712 encoded data
     * @param payload An arbitrary payload that can be used to pass additional data to the verifier
     * @return magic The magic value that should be returned if the signature is valid (0x1626ba7e)
     */
    function isValidSignature(
        Safe safe,
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
