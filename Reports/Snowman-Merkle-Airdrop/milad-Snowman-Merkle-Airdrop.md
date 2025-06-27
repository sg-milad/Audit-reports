# Snowman Merkle Airdrop - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Unrestricted NFT Minting in Snowman Contract](#H-01)




# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #42

### Dates: Jun 12th, 2025 - Jun 19th, 2025

[See more contest details here](https://codehawks.cyfrin.io/c/2025-06-snowman-merkle-airdrop)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 1
- Medium: 0
- Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. Unrestricted NFT Minting in Snowman Contract            



## Description

The `Snowman` contract is designed to issue NFTs to users who have staked `Snow` tokens through the `SnowmanAirdrop` contract. The expected behavior is that only users who participate in the staking mechanism receive `Snowman` NFTs as a form of reward.

However, the `mintSnowman()` function in `Snowman.sol` is declared `external` and **lacks any access control**. As a result, **any address** can call this function and mint an arbitrary number of NFTs to any recipient. This breaks the intended tokenomics and reward logic by allowing free and unlimited minting of NFTs without staking `Snow` tokens.

```solidity
function mintSnowman(address receiver, uint256 amount) external {
    for (uint256 i = 0; i < amount; i++) {
@>      _safeMint(receiver, s_TokenCounter);
        emit SnowmanMinted(receiver, s_TokenCounter);
        s_TokenCounter++;
    }
}
```

## Risk

**Likelihood**:

* The vulnerable function is declared external, making it callable by any externally owned account or contract.

* There is no access control modifier (onlyOwner, onlyAirdrop, etc.), so no checks prevent unauthorized callers.

**Impact**:

Unlimited Minting: Any actor can mint arbitrary numbers of NFTs, inflating supply and undermining scarcity.

Broken Tokenomics: The staking-reward model fails since NFTs can be obtained without staking Snow tokens.

Airdrop Fairness Undermined: Legitimate stakers lose confidence if anyone can mint freely.

Protocol Integrity Damage: Reputation and trust suffer; integrations expecting correct supply may malfunction.

## Proof of Concept

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Test} from "forge-std/Test.sol";
import {Snowman} from "../src/Snowman.sol";

contract PoCTest_SnowmanAccessControl is Test {
    Snowman nft;

    address attacker = makeAddr("attacker");

    function setUp() public {
        // Deploy with a placeholder SVG URI
        nft = new Snowman("ipfs://snowman-uri");
    }

    function testAnyoneCanMintNFT() public {
        // Attacker mints 3 NFTs to themselves without staking
        vm.prank(attacker);
        nft.mintSnowman(attacker, 3);

        // Verify that tokens were minted to attacker
        assertEq(nft.ownerOf(0), attacker);
        assertEq(nft.ownerOf(1), attacker);
        assertEq(nft.ownerOf(2), attacker);
        assertEq(nft.balanceOf(attacker), 3);
        assertEq(nft.getTokenCounter(), 3);
    }
}
```

## Recommended Mitigation

Restrict mintSnowman() so only the authorized SnowmanAirdrop contract (or another allowed address) can call it:

```diff
- // Original: unrestricted mint function
- function mintSnowman(address receiver, uint256 amount) external {
+ // Mitigated: onlyAirdrop modifier restricts access
+ function mintSnowman(address receiver, uint256 amount) external onlyAirdrop {
     for (uint256 i = 0; i < amount; i++) {
         _safeMint(receiver, s_TokenCounter);
         emit SnowmanMinted(receiver, s_TokenCounter);
         s_TokenCounter++;
     }
 }

+ // Add an immutable reference to the authorized airdrop contract
+ address private immutable i_airdrop;
+
+ modifier onlyAirdrop() {
+     if (msg.sender != i_airdrop) revert SM__NotAllowed();
+     _;
+ }
+
+ // Update constructor signature to accept the airdrop address
+ constructor(string memory _SnowmanSvgUri, address _airdrop) ERC721("Snowman Airdrop", "SNOWMAN") {
+     s_SnowmanSvgUri = _SnowmanSvgUri;
+     i_airdrop = _airdrop;
+ }

```

* Ensure that SnowmanAirdrop is deployed first (or its address is known) and passed into the Snowman constructor.

* In the SnowmanAirdrop contract, implement the staking checks and Merkle-based validation before calling mintSnowman().

* Optionally, add further sanity checks (e.g., max amount per stake, event logging) as needed for governance or monitoring.

    





