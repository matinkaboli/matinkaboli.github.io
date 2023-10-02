---
layout: post
title: Multicall in Solidity
date: 2023-10-01 15:42
categories: solidity
---

In a recent project, we needed a feature called Multicall,
which lets you run several functions of a contract in just one transaction.
I knew about this because I'd seen it in Uniswap V3 Periphery code.
So, I basically copied their Multicall code,
made some tweaks for compatibility, and it worked like a charm.

Here’s the Multicall.sol file from Uniswap V3 Periphery:

```solidity
// SPDX-License-Identifier: GPL-2.0-or-later
pragma solidity =0.7.6;
pragma abicoder v2;

import '../interfaces/IMulticall.sol';

/// @title Multicall
/// @notice Enables calling multiple methods in a single call to the contract
abstract contract Multicall is IMulticall {
    /// @inheritdoc IMulticall
    function multicall(bytes[] calldata data) public payable override returns (bytes[] memory results) {
        results = new bytes[](data.length);
        for (uint256 i = 0; i < data.length; i++) {
            (bool success, bytes memory result) = address(this).delegatecall(data[i]);

            if (!success) {
                // Next 5 lines from https://ethereum.stackexchange.com/a/83577
                if (result.length < 68) revert();
                assembly {
                    result := add(result, 0x04)
                }
                revert(abi.decode(result, (string)));
            }

            results[i] = result;
        }
    }
}
```

The `multicall` function is quite simple.
It takes an array of `bytes` called `data` and processes each element in the array one after the other.
For each element, it uses `address(this).delegatecall(data[i])` to carry out the function call within the same contract.
It then verifies if each call was successful,
and if any of them weren't, it cancels the entire transaction using `revert`.

After integrating `Multicall` into the main contract,
I started to revamp the existing functions within the contract.
Instead of bundling multiple functions like `transferFrom`, `submit`, and `sweepToken` into one,
I separated them into three distinct functions: `transferFrom`, `submit`, and `sweepToken`.
Now, you can execute all three of these functions in a single transaction, thanks to `Multicall`.
This change introduced a plethora of new possibilities and enhanced the flexibility of our contract code.

When I started writing tests for each contract, everything went smoothly.

The primary distinction between our contract code and V3 Periphery was the use of
ETH to store a proxy fee in our contract each time a user invoked the multicall function.
In contrast, V3 Periphery does not employ ETH storage within its contracts.
Additionally, we incorporated a `withdrawAdmin` function to facilitate the
retrieval of the stored ETH within our contract.

It's worth noting that in V3 Periphery,
any remaining ETH in the contracts can be reclaimed by anyone through the `refundETH` function.
This ensures transparency and accessibility in handling any residual ETH.

### The Problem

This seemingly minor distinction gave rise to a critical vulnerability within our contract.
The persistence of `msg.value` and `msg.sender` inside each delegatecall turned out to be a significant issue.

Let's consider a scenario in which a user intends to submit 1 ETH to the Lido contract in exchange for stETH.
However, the contract currently holds 5 ETH as a proxy fee,
which can only be withdrawn using the `withdrawAdmin` function.

Here’s the `submitLido` function:

```solidity
    /**
     * @notice Transfers ETH to the Lido protocol to get ST_ETH
     * @return stethAmount Amount of ST_ETH token that is being transferred to the recipient
     */
    function submitLido()
        external
        payable
        returns (uint256 stethAmount)
    {
        // Get StETH by paying ETH
        stethAmount = steth.submit{value: msg.value}(msg.sender);
    }
```

The vulnerability arises when a user repeatedly calls the `submitLido` function six times,
with the same 1 ETH as the transaction value. Due to the persistence of `msg.value` in each delegatecall,
the caller effectively utilizes the 5 ETH stored within the contract to their advantage. Consequently,
when the caller subsequently invokes the `sweepToken` function to sweep stETH tokens from the proxy contract,
they end up with stETH tokens equivalent to 6 ETH in value.
This exploit takes advantage of the stored proxy fee,
allowing the user to amass a greater amount of stETH tokens than initially intended.

### The Solution

Indeed, this scenario may not occur frequently,
but it highlights a potential vulnerability in the contract.
Upon reviewing OpenZeppelin's `Multicall` and finding no immediate solution,
conducting further research was a prudent step.
It became evident that implementing a mechanism akin to OpenZeppelin's `ReentrancyGuard` contract
would be essential to safeguard against such exploits and ensure the contract's security.
This proactive approach helps fortify the contract and prevents unforeseen
vulnerabilities from being exploited in the future.

So this is the final `Multicall` contract having a guard against ETH reuse:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.18;

/**
 * @title Handles multicall function
 */
contract Multicall {
    uint256 private constant NOT_ENTERED = 1;
    uint256 private constant ENTERED = 2;
    uint256 private _status;

    /**
     * @notice Thrown when the nonETHReuse modifier is called twice in the multicall
     */
    error EtherReuseGuardCall();

    /**
     * @dev Prevents a caller from calling multiple functions that work with ETH in a transaction
     */
    modifier nonETHReuse() {
        _nonReuseBefore();
        _;
    }

    /**
     * @notice Sets status to NOT_ENTERED
     */
    constructor() payable {
        _status = NOT_ENTERED;
    }

    /**
     * @notice Multiple calls on proxy functions
     * @param _calldata An array of calldata that is called one by one
     */
    function multicall(bytes[] calldata _calldata) external payable {
        // Unlock ether locker just in case if it was locked before
        unlockETHReuse();

        // Loop through each calldata and execute them
        for (uint256 i = 0; i < _calldata.length;) {
            (bool success, bytes memory result) = address(this).delegatecall(_calldata[i]);

            // Check if the call was successful or not
            if (!success) {
                // Next 7 lines from https://ethereum.stackexchange.com/a/83577
                if (result.length < 68) revert();

                assembly {
                    result := add(result, 0x04)
                }

                revert(abi.decode(result, (string)));
            }

            // Increment variable i more efficiently
            unchecked {
                ++i;
            }
        }

        /*
         * To ensure proper execution, unlock reusability for future use.
         * In some cases, the caller might invoke a function with the 'nonETHReuse'
         * modifier directly, bypassing the 'unlockETHReuse' step at the beginning of the
         * multicall. This would render the function unusable if not unlocked here.
         */
        unlockETHReuse();
    }

    /**
     * @notice Unlocks the reentrancy
     * @dev Should be used before and after all the calls
     */
    function unlockETHReuse() internal {
        _status = NOT_ENTERED;
    }

    function _nonReuseBefore() private {
        // On the first call to nonETHReuse, _status will be NOT_ENTERED
        if (_status == ENTERED) {
            revert EtherReuseGuardCall();
        }

        // Any calls to nonETHReuse after this point will fail
        _status = ENTERED;
    }
}
```

To address the vulnerability,
a crucial step involves initializing the `_status` variable to `NOT_ENTERED` before
the execution of the `multicall`'s for loop.
This state indicates that no ETH function has been invoked yet.

Furthermore, the `submitLido` function should be equipped with the `nonETHReuse` modifier,
which verifies whether the `_status` is set to `ENTERED` or not. With this modifier in place,
any subsequent calls to `submitLido` will result in a revert,
as the `_status` is already marked as `ENTERED`.
This modification effectively prevents the exploitation of the
contract by disallowing multiple calls to `submitLido` when the status is already in use,
ensuring the contract's security.

Indeed, to mitigate the vulnerability effectively,
it's essential to consistently apply the `nonETHReuse`
modifier to functions that interact with `msg.value`.

This precautionary measure helps safeguard the contract's integrity and security.
Failing to apply this modifier to such functions could potentially lead to severe consequences,
emphasizing the critical importance of thorough and diligent
code review and testing to ensure the modifier is consistently used where necessary.

The introduction of the `nonETHReuse` modifier to protect against reentrancy vulnerabilities,
as described, is indeed a robust security measure.
However, this approach may limit the usability of functions outside the context of the `multicall`.
Since it's the `Multicall` contract that unlocks the `_status` before the actual call,
other functions may not be accessible unless invoked through the `multicall`.

While this restriction enhances security,
it's crucial to carefully consider the trade-offs between security and usability in your specific application.
You may want to assess whether the additional protection provided by
the modifier is worth the inconvenience of limited usability in certain scenarios.
Finding the right balance between security and functionality is
a key aspect of designing robust smart contracts.

### The downside

There is a trade-off with this approach: users will need to call the multicall for
every transaction associated with functions featuring the nonETHReuse modifier.
This necessity arises from the fact that after a direct call to these functions,
the `_status` is set to `ENTERED`, causing subsequent direct calls to revert.
It's important to note that checking the `_status` before executing a transaction
is not a practical solution, as it can be vulnerable to front-running,
where malicious actors exploit transaction order to their advantage.

## Conclusion

In conclusion, addressing vulnerabilities in smart contract development is a nuanced and evolving process.
While there may not be a one-size-fits-all solution,
comprehending the intricacies of a vulnerability opens the door to
innovative ideas tailored to the unique codebase and context of your project.

By recognizing and comprehending vulnerabilities,
you empower yourself to explore various strategies that may lead to even more effective
solutions than those initially conceived. The key is to remain proactive,
adaptable, and collaborative, leveraging your understanding of the vulnerability
as a springboard for developing robust and secure smart contracts.

Furthermore, I highly recommend checking out Samczsun's insightful article titled
"Two Rights Might Make a Wrong" at https://samczsun.com/two-rights-might-make-a-wrong/.
In this article, Samczsun shares their experiences grappling with a similar issue and
offers valuable insights into how they successfully addressed it.
Exploring their approach can provide additional perspective and guidance on handling
vulnerabilities in smart contract development, reinforcing the importance of sharing
knowledge and experiences within our community.

I'd like to express my heartfelt gratitude to [Pashov](https://twitter.com/pashovkrum/)
for their invaluable assistance in improving and refining this contract.
Their insights and contributions have played a
pivotal role in enhancing the contract's functionality and security.
