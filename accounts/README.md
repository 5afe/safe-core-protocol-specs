# Accounts

The Safe{Core} Protocol is designed to be **account agnostic**. This initial alpha version sets a focus on the 1.x versions of Safe Smart Accounts to expedite the development process and gather feedback. These learnings are the foundation upon which the protocol is opened up to other account implementations.

## Safe 1.4.0

[Source Code](https://github.com/safe-global/safe-contracts/tree/v1.4.0)

<img src="../_assets/accounts_safe_140.png" width=400/>

### Interface

```Solidity
interface ISafe {
    function execTransactionFromModuleReturnData(
        address to,
        uint256 value,
        bytes memory data,
        uint8 operation
    ) external returns (bool success, bytes memory returnData);
}
```
