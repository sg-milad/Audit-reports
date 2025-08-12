# OrderBook - Findings Report

- ## Low Risk Findings
  - ### [L-01. Integer Truncation in Fee Calculation Leads to Protocol Undercharging and Seller Overpayment](#L-01)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #43

### Dates: Jul 3rd, 2025 - Jul 10th, 2025

[See more contest details here](https://codehawks.cyfrin.io/c/2025-07-orderbook)

# <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 0
- Medium: 0
- Low: 1

# Low Risk Findings

## <a id='L-01'></a>L-01. Integer Truncation in Fee Calculation Leads to Protocol Undercharging and Seller Overpayment

## Description

The fee calculation uses integer division which truncates fractional values. For prices not divisible by 100, this results in:

- Protocol receiving less than the intended 3% fee

- Seller receiving marginally more than intended

- Small value leakage that compounds over many transactions

```Solidity
function buyOrder(uint256 _orderId) public {
    // ...
    uint256 protocolFee = (order.priceInUSDC * FEE) / PRECISION; // @> Truncation occurs here
    uint256 sellerReceives = order.priceInUSDC - protocolFee; // @> Seller benefits from truncation
    // ...
}
```

## Risk

**Likelihood**: High

- Occurs in every transaction where price % 100 ≠ 0

- Affects 99% of possible price points

- Magnified with high transaction volume

**Impact**: Medium

- Protocol loses expected revenue from fees

- Sellers receive unintended windfall gains

- Fee inaccuracy violates protocol specifications

- Value leakage compounds significantly over time

## Proof of Concept

```Solidity
function test_feeTruncationGivesDustToSeller() public {  //  run this in TestOrderBook.t.sol
        // Setup: Alice lists 0.000001 WBTC for 0.000001 USDC (smallest units)
        vm.startPrank(alice);
        wbtc.approve(address(book), 1); // 1 satoshi (0.00000001 WBTC)
        uint256 orderId = book.createSellOrder(address(wbtc), 1, 1, 1 days);
        vm.stopPrank();

        // Pre-balance check
        uint256 protocolFeesBefore = book.totalFees();

        // Execute buy order
        vm.startPrank(dan);
        usdc.approve(address(book), 1);
        book.buyOrder(orderId);
        vm.stopPrank();

        // Post-balance check: Protocol gets NO fees (0 instead of 0.03 base units)
        uint256 protocolFeesAfter = book.totalFees();
        console2.log("before", protocolFeesBefore); // 0
        console2.log("after", protocolFeesAfter); // 0
        assertEq(protocolFeesAfter - protocolFeesBefore, 0, "Protocol gets 0 fees");
        assertEq(usdc.balanceOf(alice), 1, "Seller keeps full amount");
    }
```

## Recommended Mitigation

Increase precision and implement rounding up:

```diff
- uint256 public constant FEE = 3; // 3%
- uint256 public constant PRECISION = 100;
+ uint256 public constant FEE = 300; // 3% with 10000 precision
+ uint256 public constant PRECISION = 10000; // 0.01% granularity

function buyOrder(uint256 _orderId) public {
    // ...
-   uint256 protocolFee = (order.priceInUSDC * FEE) / PRECISION;
+   uint256 protocolFee = (order.priceInUSDC * FEE + PRECISION - 1) / PRECISION;
    // ...
}
```
