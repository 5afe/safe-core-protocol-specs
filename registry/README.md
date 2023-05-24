# Registry

The `Registry` keeps track of all `Components` and if they comply to the `correct conduct and procedures`.

Therefore it also MUST define the `correct conduct and procedures`. Different mechanisms and tools then can be used to check and enforce these. Examples for this could be **Audits** and **Bug Bounties**. The `Mediator` makes use of the `Registry` to check that only interaction with `Components` that comply to the conduct and procedures are possible.

For this the registry MUST implement the following interface:

```solidity
interface SafeProtocolRegistry {
    /// @param component Address of the component that should be checked 
    /// @return listedAt MUST return the block number when the component was listed in the registry (or 0 if not listed)  
    /// @return flaggedAt MUST return the block number when the component was listed in the flagged as faulty (or 0 if not flagged)  
    function check(address component) external view returns (uint256 listedAt, uint256 flaggedAt);
}
```

## Hyperstructures and Authorities

- https://mirror.xyz/konradkopp.eth/7Q3TrMFgx2VbZRKa7UEaisIMjimpMABiqGYo00T9egA

## Automation

Currently it is required that users manually keep track of their modules and deactivate them in the case that issues arise. Having a more automated approach will be able to protect users better. This needs to be balances with a system that prevents that parts of the protocol are not just maliciously deactivated. Here the possibility of "local overwrites", "temporary deactivation" and "putting something at stake" need to be evaluated.

Another challenge with registries is to properly connect them to the relevent parts of the tech stack. Here a push model could provide quite some benefits over a pull model. This is quite challenging to implement, but using different incentives it is possible to motivate a "semi-push" approach where external parties are responsible for the propagation and get rewarded for this.