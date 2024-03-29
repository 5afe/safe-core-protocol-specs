# Modules

Modules extend the functionality of Accounts in different ways. Initial modules are `plugins`, `hooks`, `function handlers` and `signature validators`, but additional modules can be added to the Safe{Core} Protocol at a later point.

[General Types](../manager/README.md#general-types)

## Module Types

Each module is assigned a value that represents the type of module it is. The value is a power of 2, which permits bitwise operations and efficient storage of values. The table below lists the module types and their corresponding values. A contract can be used as multiple module types.

| Module type               | Value |
|---------------------------|-------|
| Plugin                    | 1     |
| Function Handler          | 2     |
| Hooks                     | 4     |
| Signature Validator Hooks | 8     |
| Signature Validator       | 16    |

## Plugins

Plugins allow to add any arbitrary logic to an account such as recovery mechanisms, session keys, and automations.

Plugins can trigger transactions on an `Account` via the `Manager`.

```solidity
interface ISafeProtocolPlugin {
    function name() external view returns (string memory name);

    function version() external view returns (string memory version);

    function metaProvider() external view returns (uint256 type, bytes memory location);

    function requiresPermissions() external view returns (uint8 permissions);
}
```

### Plugin Permissions

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

### Hooks for Transaction Execution

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

#### Parameter `executionMeta` value

| Execution Type                                                                                                                                                                                 | Value                                                                                                                                                                                        |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Multisignature Flow (for accounts that use [Guard interface](https://github.com/safe-global/safe-contracts/blob/8ffae95faa815acf86ec8b50021ebe9f96abde10/contracts/base/GuardManager.sol#L10)) | Encoded data created from parameter values received from `checkTransaction(...)` i.e. `abi.encode(to, value, data, gas, baseGas, gasPrice, gasToken, refundReceiver, signatures, msgSender)` |
| Plugin Flow                                                                                                                                                                                    | Encoded address of the Plugin i.e. `abi.encode(pluginAddress)`                                                                                                                               |

#### Parameter `executionType` Value

| Execution Type      | Value |
|---------------------|-------|
| Multisignature Flow | `0`   |
| Plugin Flow         | `1`   |

TODO: provide more details on execution metadata

## Function Handler
d
Non-static version (invoked via `call`):

```solidity
interface ISafeProtocolFunctionHandler {
    function handle(address account, address sender, uint256 value, bytes calldata data) external returns (bytes memory result);
}
```

Static version (invoked via `staticcall`):

```solidity
interface ISafeProtocolStaticFunctionHandler {
    function handle(address account, address sender, uint256 value, bytes calldata data) external view returns (bytes memory result);
}
```

Function handlers, once installed are executed without checking the registry.

Kudos to @mfw78

## Signature Validators

Signature validators enable modules to extend an account's [ERC-1271](https://eips.ethereum.org/EIPS/eip-1271) `isValidSignature` implementation allowing for custom signature validation rules.
Inspired from [EIP-712](https://eips.ethereum.org/EIPS/eip-712) which specifies standard for typed structured data hashing and signing, a signature validator is expected to validate a signed message for a specific domain. To do so, a `SignatureValidatorManager` contract acts as storage for maintaining enabled validators (approved by the registry) per domain per account or use a default signature validation scheme.

Apart from validating EIP-712 typed signed message an account might also be required to support other arbitrary signed messages for interaction with projects following other message hashing and signing formats. Other signing standards can be supported using default signature validation scheme which depends on the account implementation.

### Enabling Signature Validator

For enabling a domain specific EIP-712 signature validator, first a signature validator must be enabled by the account which is outlined by the below sequence diagram.

```mermaid
sequenceDiagram
    participant Account
    participant SignatureValidatorManager
    participant RegistryContract

    Account->>SignatureValidatorManager: Set SignatureValidatorContract for a specific domain.
        SignatureValidatorManager->>RegistryContract: Check if SignatureValidatorContract is listed and not flagged
        RegistryContract-->>SignatureValidatorManager: Return result
    SignatureValidatorManager->>SignatureValidatorManager: Check if SignatureValidatorContract implements ISafeProtocolSignatureValidator interface
    SignatureValidatorManager-->>Account: Ok
```

The above sequence diagram only covers a flow when the transaction executes successfully.
The possible cases for a transaction to revert are:
- Signature validator contract is not listed in registry
- Signature validator contract is flagged in registry
- Transaction ran out of gas

### Validating Account Signature

After a enabling a signature validator and setting `SignatureValidatorManager` as function handler in `SafeProtocolManager`, external entities can request validating account signatures. The diagram below illustrates the sequence of calls that are made when an account signature is to be validated. When a signature for an account is to be validated by an external entity, here referred as `ExternalContract`, the `ExternalContract` contract calls the `isValidSignature(bytes32,bytes)` function supported by the account. Account further calls `SignatureValidatorManager`. `SignatureValidatorManager` decides to use default validation scheme or domain specific validator based on the contents of the received data. If domain specific signature validator is to be used and the signature validator contract is set for the account, listed and not flagged, the `SignatureValidatorManager` calls the `SignatureValidatorContract` to check if the signature is valid. Additionally, hooks for signature validators help to provide a way to add additional checks before and after the calling a signature validation function.

```mermaid
sequenceDiagram
    participant ExternalContract
    participant Account
    participant SafeProtocolManager
    participant SignatureValidatorManager
    participant RegistryContract
    participant SignatureValidatorContract
    participant SignatureValidatorHooks

    ExternalContract->>Account: Check if signature is valid for the account (Call isValidSignature(bytes32,bytes))
    Account->>SafeProtocolManager: SafeProtocolManager enabled as fallback handler for account
    SafeProtocolManager->>SignatureValidatorManager: Call handle(...) function
    SignatureValidatorManager->>SignatureValidatorManager: Decode data and extract 4 bytes signature selector
    alt selector indicating to use domain specific validator
        SignatureValidatorManager->>SignatureValidatorManager: Decode data and extract domain, hashStruct and signatures
        SignatureValidatorManager->>SignatureValidatorManager: Validate the hash and load Signature Validator for the domain
        SignatureValidatorManager->>RegistryContract: Check if Signature Validator contract is listed and not flagged
        RegistryContract-->>SignatureValidatorManager: Return result
        SignatureValidatorManager->>SignatureValidatorHooks: Execute preValidationHook(...) if enabled
        SignatureValidatorHooks-->>SignatureValidatorManager: Return result

        SignatureValidatorManager->>SignatureValidatorContract: Check for signature validity (call isValidSignature(...))
        SignatureValidatorContract-->>SignatureValidatorManager: Return signature validation result

    else use default validation flow
        SignatureValidatorManager->>SignatureValidatorManager: Decode data and extract hash and signatures
        SignatureValidatorManager->>SignatureValidatorHooks: Execute preValidationHook(...) if enabled
        SignatureValidatorManager->>Account: Check if given signature is valid. (call should be other than account's isValidSignature(...) function)
        Account-->>SignatureValidatorManager: Ok
    end
    SignatureValidatorManager->>SignatureValidatorHooks: Execute postValidationHook(...) if enabled
    SignatureValidatorManager-->>ExternalContract: Return signature validation result
    Account-->>ExternalContract: Return signature validation result
```

The above sequence diagram only covers a flow when the transaction executes successfully.
The possible cases for a transaction to revert are:
- No Signature validator contract is set for the domain
- Invalid message hash 
- Signature validator contract is not listed in registry
- Signature validator contract is flagged in registry
- Decoding of data failed
- If signature validator hook is enabled, call to `preValidationHook(...)` reverted
- Call to signature validator contract reverted
- Default signature validation failed
- If signature validator hook is enabled, call to `postValidationHook(...)` reverted
- Transaction ran out of gas

The information required to route the signature validation to the correct validator has to be passed to the manager. As the validation flow is based on ERC-1271, it is necessary to encode this information into the parameters of this standard. ERC-1271 defines an interface to verify signatures based on contract logic. For this, `isValidSignature(bytes32,bytes)` function has to be called where the first parameter i.e., `bytes32` is the hash of the message that was signed, and the second parameter i.e., `bytes` are the signatures (which are arbitrary bytes).

It is important to understand the signing and verification flow related to ERC-1271. Any dapp can make use of `eth_sign` or `eth_signTypedData` to retrieve a signature from a connected wallet. To verify this signature for an EOA the dapp will calculate the hash of the message (either ERC-191 or ERC-712 based) and recover the signer address from the signature using the ECDSA based recovery. For smart contracts the flow is similar, but instead of using the ECDSA based recovery method to validate the signer, the dapp calls `isValidSignature` on the smart contract with the hash and the signature.

This allows us to add additional logic when validating signatures, such as differentiating the validation logic based on the message. The challenge is that it is impossible to retrieve the message that was signed from the hash. To work around it we can encode the message into the signature and then when the smart contract is called this message is extracted, hashed and validated against the hash. This is outlined in the following example code:

```solidity
function isValidSignature(bytes32 hash, bytes memory payload) external view returns (bytes4) {
  (bytes memory signature, bytes memory message) = abi.decode(payload, (bytes,bytes));
  require(hash == keccak256(message), "Unexpected message/hash combination");
  // more validation logic
}
```

Using this approach it is also possible to encode ERC-712 based messages in the signature (for signatures requested via `eth_signTypedData`). With this we can route the signature validation based on the domain of the ERC-712 message. The payload contain the signature, the domainHash and the hashStruct. Only minor adjustments are required to the previous code:

```solidity
mapping(bytes32 => ISignatureValidator) validatorForDomain;
function isValidSignature(bytes32 hash, bytes memory payload) external view returns (bytes4) {
  (bytes32 domainHash, bytes32 hashStruct, bytes memory signature) = abi.decode(payload, (bytes32,bytes32,bytes));
  require(hash, keccak256(0x19, 0x01, domainHash, hashStruct), "Could not validate data");
  ISignatureValidator validator = validatorForDomain[domainHash];
  return validator.isValidSignature(msg.sender /* account */, _msgSender(), domainHash, hashStruct, signature);
}
```

To distinguish signatures that follow this format from "default" signatures the signature is prepended with a special bytes4 identifier. When the dapp requests the signature the wallet can then create the signature in the required format for the correct signature routing.

The encoded data to be received by the signature validator manager as `bytes` parameter of `isValidSignature(bytes32,bytes)` is expected to be encoded as follows when domain specific signature validator is to be used:

```solidity
function encodeData(bytes32 domain, bytes32 hashStruct, bytes calldata signatures) public pure returns (bytes memory) {
        bytes memory data = abi.encode(domain, hashStruct, signatures);
        bytes4 selector = "Account712Signature(bytes32,bytes32,bytes)";
        return abi.encodeWithSelector(selector, data);
}
```

In the default signature validation flow, the wallet is expected to generate a hash of the message i.e., `messageHash` for signing and submit it for validation as follows:

```
    ACCOUNT_MSG_TYPEHASH = keccak256("SafeMessage(bytes message)");
    DOMAIN_SEPARATOR_TYPEHASH = keccak256("EIP712Domain(uint256 chainId,address verifyingContract"));

    domainSeparator = keccak256(DOMAIN_SEPARATOR_TYPEHASH || chainId || accountAddress);
    dataHash = keccak256("encoded-bytes-data-to-be-signed");

    hashStruct = keccak256(DOMAIN_SEPARATOR_TYPEHASH || dataHash);
    messageHash = keccak256(0x19 || 0x01 || domainSeparator || dataHash);

    signatures = sign(messageHash);
    result = account.isValidSignature(dataHash, signatures);
```

In the above code, the account's domain separator is included in the message whose hash is signed. By doing so, the account only approves relevant signatures and not any other arbitrary signed messages. The domain separator is expected to include the account address and chainId to avoid cross-chain replay attacks. The actual value domain separator depends on the account implementation.

### Signature Validator Interface

A signature validator must implement the following interface.

```solidity
interface ISafeProtocolSignatureValidator {
    /**
     * @param safe The Safe that has delegated the signature verification
     * @param sender The address that originally called the Safe's `isValidSignature` method
     * @param messageHash The EIP-712 hash whose signature will be verified
     * @param domainSeparator The EIP-712 domainSeparator
     * @param hashStruct Hash generated from typeHash and encodeData
     * @param signature The signature to be verified
     * @return magic The magic value that should be returned if the signature is valid (0x1626ba7e)
     */
    function isValidSignature(
        address account,
        address sender,
        bytes32 messageHash,
        bytes32 domainSeparator,
        bytes32 hashStruct
        bytes calldata signatures
    ) external view returns (bytes4 magic);
}
```

### Hooks for Signature Validator

Signature validation is a critical part of smart contract accounts. A flawed implementation of validation logic can lead to approving a transaction with an invalid signature. Signature validator hooks can be used to add safety net to the signature validation process and hold the power to block a signature validator from approving unauthorized actions. Hooks can be enabled for all validators before and after execution of the validation function. This can be done by adding contract implementing `ISignatureValidatorHook` interface to the `SignatureValidatorManager` by the account. The hooks are not specific to a domain and are executed for all signature validations. The hooks function(s) should revert on failed checks.

```solidity
interface ISignatureValidatorHooks {
    /**
     * @param account Address of the account for which signature is being validated
     * @param validator Address of the validator contract to be used for signature validation. This address will be account address in case of default signature validation flow is used.
     * @param payload The payload provided for the validation
     * @return result bytes containing the result
     */
    function preValidationHook(address account, address validator, bytes calldata payload) external returns (bytes memory result);

    /**
     * @param account Address of the account for which signature is being validated
     * @param preValidationData Data returned by preValidationHook
     * @return result bytes containing the result
     */
    function postValidationHook(address account, bytes calldata preValidationData) external returns (bytes memory result);
}
```

### Signature Validator Manager

A signature validator manager must implement the following interface.

```solidity
interface ISafeProtocolSignatureValidatorManager {
    /**
     * @param domainSeparator bytes32 containing the domain for which Signature Validator contract should be used
     * @param signatureValidatorContract Address of the Signature Validator Contract implementing ISafeProtocolSignatureValidator interface
     */
    function setSignatureValidator(bytes32 domainSeparator, address signatureValidatorContract) external;

    /**
     * @param signatureValidatorHooksContract Address of the contract to be used as Hooks for Signature Validator implementing ISignatureValidatorHook interface
     */
    function setSignatureValidatorHooks(address signatureValidatorHooksContract) external;
}
```

### Security Considerations

A malicious actor can change the routing of the signature validation flow by adding/removing the 4 bytes of the signature selector. Under such cases, the signature validation should fail and revert the transaction.

Suppose a malicious actor removes the 4 bytes of the signature selector from a payload meant for domain-specific validation flow and submits the modified payload for validation such that the validator manager uses default signature validation flow. In such a case, as the account expects `messageHash` to be generated by including the domain separator defined by the signature validator manager, the account is expected to revert the call as the hash used for signing differs from the hash used for recovery.

Similarly, if a malicious actor adds 4 bytes of the signature selector to a payload meant for default signature validation flow, submits the modified payload for validation such that the validator manager uses domain-specific signature validation flow. Even if permitted by the registry and the account has set a signature validator that matches the signature validator's domain separator, the validation would only pass on approval of the signature validator contract. So here, the flow is secure as long as the signature validator contract is not malicious.

Kudos to @mfw78

References:
- https://github.com/zerodevapp/kernel/blob/main/src/interfaces/IValidator.sol
- https://eips.ethereum.org/EIPS/eip-6900
