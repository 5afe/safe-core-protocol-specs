# 4337 Plugin

The Core Protocol allows 4337 support for accounts to be implemented as a function handler and a plugin.

## Paymaster



## User Operation Validation

User operations

## Execution

TBD.

# Rationale

- I see no reason that 4337 needs to be directly integrated into the protocol.
- The issue mentions that "an additional signature flow would be 4337", but I'm not sure I understand how this fits into the current signature specification for the protocol (which is very much tied to EIP-712 and ERC-1271).
  - In theory, you could implement 4337 in terms of `isValidSignature`; this would allow an `ISafeProtocolSignatureValidator` implementation to extend the signature validation behaviour of 4337 user operation validation, however this will be quite limiting. In particular, 4337 user operations will likely have a single domain, meaning that it will be an "all or nothing" specialized implementation unless you were to add an additional layer of indirection at the signature validator level, in which case, might as well add it at the 4337 plugin level and skip the signature manager altogether.
