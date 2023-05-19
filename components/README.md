# Components

[General Types](../mediator/README.md#general-types)

## Fallback method/ Function handler

Non-static version (invoked via `call`)

```solidity
interface ISafeProtocolFallbackMethod {
    function handle(Safe safe, address sender, uint256 value, bytes calldata data) external returns (bytes memory result);
}
```

static version (invoked via `staticcall`)

```solidity
interface ISafeProtocolStaticFallbackMethod {
    function handle(Safe safe, address sender, uint256 value, bytes calldata data) external view returns (bytes memory result);
}
```

Kudos to @mfw78

## Guards/ Hooks

Guards can check any interaction done with an `Account` via the `Mediator`, and also check direct (some) direct interactions on the `Account`(i.e. via the `execTransaction` flow).

```solidity
interface ISafeProtocolGuard {
    function preCheck(Safe safe, SafeTransaction tx, uint256 executionType, bytes calldata executionMeta) external returns (bytes memory preCheckData);

    function preCheckRootAccess(Safe safe, SafeRootAccess rootAccess, uint256 executionType, bytes calldata executionMeta) external returns (bytes memory preCheckData) external returns (bytes memory preCheckData);

    function postCheck(Safe safe, bool success, bytes calldata preCheckData) external;
}
```

Execution types:
- Multisignature Flow
- Module Flow

TODO: provide more details on execution type and execution meta

## Modules/ Plug-Ins

Modules can trigger transactions on an `Account` via the `Mediator`.

```solidity
interface ISafeProtocolModule {
    function name() external view returns (string memory name);

    function version() external view returns (string memory version);

    function metaProvider() external view returns (uint256 type, bytes memory location);

    function requiresRootAccess() external view returns (bool requiresRootAccess);
}
```

### Module Interface Extensions

- Module Fee Payment Facilitator (something like In-App Payments)
- Gas Fee Payment Authorizer (allow 4337 and other relay usages with modules)

## Signature verifiers (ERC-1271)

```solidity
interface ISafeProtocolSignatureVerifier {
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