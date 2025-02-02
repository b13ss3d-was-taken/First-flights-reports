# Pieces Protocol - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Potential front-running in `buyOrder` function allows attackers to preempt legitimate buyers](#H-01)

- ## Low Risk Findings
    - ### [L-01. Excess ETH received in `buyOrder` function has no withdrawal mechanism, leading to permanent loss of funds](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #32

### Dates: Jan 16th, 2025 - Jan 23rd, 2025

[See more contest details here](https://codehawks.cyfrin.io/c/2025-01-pieces-protocol)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 1
- Medium: 0
- Low: 1


# High Risk Findings

## <a id='H-01'></a>H-01. Potential front-running in `buyOrder` function allows attackers to preempt legitimate buyers            



## Summary

Anyone can monitor the `buyOrder` transactions and place their own order with the same details or even a lower `msg.value`, but with a higher gas price, to ensure their transaction gets mined first. This allows the attacker to front-run the legitimate buyer, becoming the purchaser of the token instead. As a result, the intended buyer loses the opportunity to acquire the token, and the attacker gains unfair access to it.&#x20;

## Impact

The attacker can buy a specific token before the legitimate buyer's transaction is processed, effectively stealing the opportunity to purchase the asset. This not only causes financial loss for the buyer but also undermines the fairness and reliability of the platform.

## Tools Used

* Manual review.

## Recommendations

* You can consider implementing a two-step process where people interested in buying a specific token first have to join a 'waitlist' with their offer, and the seller has to choose which offer to accept.
* Also you can think of using a private mempool

    


# Low Risk Findings

## <a id='L-01'></a>L-01. Excess ETH received in `buyOrder` function has no withdrawal mechanism, leading to permanent loss of funds            



## Summary

Excess ETH sent during the buyOrder function is not refunded to the buyer, leading to permanent loss of funds.

## Vulnerability Details

In the `buyOrder` function, the contract checks if the `msg.value` is greater than or equal to the required amount (`order.price + sellerFee`). However, if the buyer sends more ETH than required, the excess ETH remains stuck in the contract. There is no mechanism to refund the excess ETH to the buyer.

## Impact

Users who send more ETH than necessary will lose their funds permanently or ETH sent accidentally (e.g., via `selfdestruct` or a direct transfer) cannot be recovered.\
This can lead to significant financial losses, especially for high-value transactions.

The contract will accumulate unnecessary ETH over time, which could be exploited or cause operational issues.

## Tools Used

* Manual code review.

## Recommendations

1. Calculate the excess ETH (`msg.value - totalRequired`) and refund it to the buyer using `transfer` or `call`.

   ```Solidity
   if (msg.value > order.price + sellerFee) {
       uint256 excess = msg.value - (order.price + sellerFee);
       payable(msg.sender).call{value: excess}(""); 
   }
   ```
2. Ensure the contract only accepts the exact amount of ETH required for the transaction, or explicitly handle excess ETH.
3. Add a function to allow the owner to withdraw any accidentally locked ETH, ensuring transparency and control over the contract's balance.

   ```Solidity
   function withdraw() external onlyOwner {
     payable(owner).call{value: address(this).balance}("");
   }
   ```



