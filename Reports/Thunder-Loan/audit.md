# 🔍 Audit Finding — Thunder Loan

- **Project:** [Thunder Loan – Cyfrin Audit (Nov 2023)](https://github.com/Cyfrin/2023-11-Thunder-Loan/)
- **Description:** Thunder Loan is a flash loan protocol reviewed during the Cyfrin November 2023 audit. The protocol allows users to execute flash loans across various asset pools.

---

## 🟥 High Severity Finding — Storage Layout Incompatibility

| Item             | Details                                               |
| ---------------- | ----------------------------------------------------- |
| **Severity**     | High                                                  |
| **Category**     | Storage Layout Collision                              |
| **Impact**       | Flash loan fee jumps from 0.3% to 100% after upgrade  |
| **Root Cause**   | Storage slot mismatch between V1 and V2               |
| **PoC Included** | ✅ Yes                                                |
| **Fix**          | Preserve deprecated slot and migrate value in upgrade |

---

The issue lies in the storage layout incompatibility between V1 and V2 contracts. In V1, the storage slots are:

- s_tokenToAssetToken (slot 0 - mapping doesn't use storage)
- s_feePrecision (slot 1)
- s_flashLoanFee (slot 2)

In V2, you've removed s_feePrecision and moved s_flashLoanFee to slot 1. After upgrade, when V2 accesses s_flashLoanFee, it will incorrectly read the value from slot 1 (previously s_feePrecision in V1) instead of slot 2.

---

Proof of Concept & Unit Test:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {ThunderLoan} from "./ThunderLoan.sol";
import {ThunderLoanUpgraded} from "./ThunderLoanUpgraded.sol";
import {MockPoolFactory} from "./MockPoolFactory.sol";

contract StorageLayoutPoCTest is Test {
    ThunderLoan internal implementationV1;
    ThunderLoanUpgraded internal implementationV2;
    ERC1967Proxy internal proxy;
    MockPoolFactory mockPoolFactory;

    function setUp() public {
        implementationV1 = new ThunderLoan();
        proxy = new ERC1967Proxy(address(implementationV1), "");
        mockPoolFactory = new MockPoolFactory();
        ThunderLoan(address(proxy)).initialize(address(mockPoolFactory));
    }

    function test_StorageCollisionAfterUpgrade() public {
        // Pre-upgrade state check
        assertEq(ThunderLoan(address(proxy)).getFeePrecision(), 1e18);
        assertEq(ThunderLoan(address(proxy)).getFee(), 3e15); // 0.3% fee

        // Deploy and upgrade to V2
        implementationV2 = new ThunderLoanUpgraded();
        vm.startPrank(ThunderLoan(address(proxy)).owner());
        ThunderLoan(address(proxy)).upgradeTo(address(implementationV2));
        vm.stopPrank();

        // Post-upgrade state check
        ThunderLoanUpgraded upgraded = ThunderLoanUpgraded(address(proxy));

        // V2 will read WRONG slot for flash loan fee
        assertEq(upgraded.getFee(), 1e18); // Now incorrectly returns 100% fee
        assertEq(upgraded.getFeePrecision(), 1e18); // Constant still works
    }
}
```

### 🧠 Key Explanations

#### Pre-upgrade State

| Function            | Expected Value | Storage Slot |
| ------------------- | -------------- | ------------ |
| `getFee()`          | `3e15` (0.3%)  | Slot 2       |
| `getFeePrecision()` | `1e18`         | Slot 1       |

#### Post-upgrade Behavior

| Function            | Actual Value After Upgrade | Problem                          | Storage Slot   |
| ------------------- | -------------------------- | -------------------------------- | -------------- |
| `getFee()`          | `1e18` (100%)              | Reads old `s_feePrecision` value | Slot 1 (wrong) |
| `getFeePrecision()` | `1e18`                     | ✅ Correct (constant unaffected) | N/A (constant) |

---

### 💥 Critical Impact

After upgrade, the protocol reads `1e18` (100%) as the flash loan fee instead of the intended `3e15` (0.3%).  
This breaks the core functionality of the flash loan system and can lead to **user fund loss** or a **total protocol freeze**.

---

### 🛠️ Recommended Fix

#### 1. Preserve Storage Layout

Keep the old variable in place but unused, to prevent storage collision:

```solidity
// In ThunderLoanUpgraded.sol
uint256 private deprecated_s_feePrecision; // Keep same slot as V1 (slot 1)
uint256 private s_flashLoanFee;            // Will remain in slot 2
uint256 public constant FEE_PRECISION = 1e18;
```
