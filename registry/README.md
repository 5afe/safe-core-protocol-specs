# Registry

The `Registry` serves a critical function in the Safe{Core} Protocol by keeping track of all `Modules` and ensuring they comply with the prescribed `rules`.


To fulfill this role, the Registry MUST define the `rules` that Modules should adhere to. Various mechanisms, such as **audits** and **bug bounties**, can be employed to verify and enforce these rules. The `Manager` interacts with the `Registry` to confirm that only approved Modules are allowed.

To facilitate these operations, the `Registry` MUST implement the following interface:

```solidity
interface SafeProtocolRegistry {
    /**
     * @param module Address of the module that should be checked
     * @param data bytes32 providing more information about the module. The type of this parameter is bytes32 to provide the flexibility to the developers to interpret the value in the registry. For example, it can be moduleType and registry would then check if given address can be used as that type of module.
     * @return listedAt MUST return the block number when the module was listed in the registry (or 0 if not listed)
     * @return flaggedAt MUST return the block number when the module was listed in the flagged as faulty (or 0 if not flagged)
     */
    function check(address module, bytes32 data) external view returns (uint64 listedAt, uint64 flaggedAt);
}
```

## Hyperstructures and Authorities

For a detailed discussion on Hyperstructures and Authorities and their role in the protocol, refer to this [article](https://mirror.xyz/konradkopp.eth/7Q3TrMFgx2VbZRKa7UEaisIMjimpMABiqGYo00T9egA).

## Automation

Current protocol requirements necessitate users to manually monitor their plugins and deactivate them if problems occur. Implementing a more automated approach enhances user protection. However, this needs to be balanced with a system that prevents parts of the protocol from being maliciously deactivated. Here, we need to evaluate mechanisms such as local overwrites, temporary deactivation, and stake-based systems.

A significant challenge with registries lies in their integration with relevant tech stack components. Implementing a push model, where information is actively sent to the necessary parts of the system, could offer significant advantages over a pull model. Though this model is challenging to implement, it's possible to incentivize a "semi-push" approach where external parties are responsible for data propagation and are rewarded for their efforts.
