# Unsafe Array Length Truncation in Strategy Shipping

**Protocol:** 1inch Aqua  
**Target:** 1inch/aqua  
**Vulnerable Address:** 0x499943E74FB0cE105688beeE8Ef2ABec5D936d31  
**Classification:** Informational  
**Status:** Acknowledged by Protocol  
**Audit Date:** February 18, 2026

---

## Table of Contents

1. [Vulnerability Details](#vulnerability-details)
   - [Bug Description](#bug-description)
   - [Logical Failure](#logical-failure)
2. [Impact & Risk Assessment](#impact--risk-assessment)
3. [Proof of Concept](#proof-of-concept)
4. [Recommendation](#recommendation)
5. [References](#references)

---

## Vulnerability Details

### Bug Description

The `ship` function in the Aqua contract (inherited by AquaRouter) performs an **unsafe cast** of the tokens array length from `uint256` to `uint8`.

#### Vulnerable Code Pattern

```solidity
uint8 tokensCount = tokens.length.toUint8();
```

#### The Problem

The implementation fails to validate that `tokens.length` is within the maximum range of a `uint8` (0–255). 

**What happens with 256+ tokens:**
- If a user provides an array with 256 or more tokens, the `tokensCount` will **wrap around**
- Example: 256 becomes 0, 257 becomes 1, 512 becomes 0, etc.
- This leads to an **incorrect state** being stored in the `_balances` mapping

#### Integer Overflow/Truncation Example

```solidity
// Examples of what happens with toUint8() conversion:
tokens.length = 255  →  tokensCount = 255  ✓ (valid)
tokens.length = 256  →  tokensCount = 0    ✗ (wraps to 0)
tokens.length = 257  →  tokensCount = 1    ✗ (wraps to 1)
tokens.length = 512  →  tokensCount = 0    ✗ (wraps to 0)
tokens.length = 1000 →  tokensCount = 232  ✗ (1000 % 256 = 232)
```

### Logical Failure

The `dock` function requires a **strict match** between the stored state and the provided array:

```solidity
require(balance.tokensCount == tokens.length, "Invalid tokens count");
```

#### Why This Creates a Trap

1. User calls `ship()` with 256 tokens
2. `tokensCount` is truncated to 0 and stored
3. User later calls `dock()` with the same 256 tokens
4. Check fails: `0 != 256` → "Invalid tokens count" revert
5. **Funds are permanently locked**

**Timeline:**
```
Time T0: ship() called with 256 tokens
         Stored: tokensCount = 0 (truncated from 256)

Time T1: dock() called with same 256 tokens
         Comparison: 0 == 256? NO
         Result: REVERT - "Invalid tokens count"
         
Outcome: Funds trapped forever
```

---

## Impact & Risk Assessment

### Impact: Permanent Denial of Service (DoS)

**Severity: HIGH** (despite "Informational" classification by protocol)

Any strategy shipped with more than 255 tokens becomes **irrecoverable**:
- The `dock` function requires the provided `tokens.length` to strictly equal the stored `balance.tokensCount`
- A truncated value causes all future retrieval attempts to **revert permanently**
- Associated funds are **trapped indefinitely** with no recovery path

### Fund Loss Scenario

```
Step 1: Ship with 256 tokens
        - Store: tokensCount = 0
        - Lock funds

Step 2: Attempt to Dock
        - Check: 0 == 256?
        - Result: REVERT
        
Step 3: Repeat Attempts
        - Each attempt fails the same way
        - No way to bypass or correct the stored value
        - Funds remain locked permanently
```

### Risk Breakdown

| Aspect | Level | Notes |
|--------|-------|-------|
| **Likelihood** | Low/Medium | Typical strategies use few tokens; protocol lacks enforcement against over-extension |
| **Impact** | High | Permanent loss of user funds + accounting state corruption |
| **Exploitability** | Low | Not intentionally exploitable, but easily triggered by accident |
| **Classification** | Informational | Per protocol designation, but high actual risk |
| **Business Impact** | Critical | User funds lock-up and trust loss |

### Root Cause Analysis

The root cause is the **implicit assumption** that:
1. Token arrays will never exceed 255 elements (undocumented assumption)
2. Type conversion from `uint256` to `uint8` is safe without validation
3. No maximum limit is enforced at the contract level

In Solidity 0.8.x, unchecked arithmetic is reverted in most cases, but **explicit type conversions** (like `.toUint8()`) may silently truncate without warnings, leading to this vulnerability.

---

## Proof of Concept

### PoC Walkthrough

#### Step 1: Deployment
An actor (malicious or accidental) calls `ship()` with a tokens array of length **256 or more**.

```solidity
address[] memory tokens = new address[](256);
// Populate tokens array with 256 different token addresses
for (uint i = 0; i < 256; i++) {
    tokens[i] = address(uint160(i + 1));
}

// Call ship with 256 tokens
bytes32 strategyHash = aquaRouter.ship(strategy, tokens);
```

#### Step 2: Truncation
The contract executes:
```solidity
uint8 tokensCount = tokens.length.toUint8();
// tokens.length = 256
// tokensCount = 256 % 256 = 0
```

#### Step 3: Storage
The contract stores the truncated value:
```solidity
balance.tokensCount = 0;  // Stored as 0, not 256!
_balances[strategyHash] = balance;
```

#### Step 4: Recovery Attempt
The user attempts to recover their funds by calling `dock()`:

```solidity
aquaRouter.dock(strategyHash, tokens);  // Providing the same 256 tokens
```

#### Step 5: Failure
The contract checks:
```solidity
require(balance.tokensCount == tokens.length, "Invalid tokens count");
// Compares: 0 == 256?
// Result: FALSE → REVERT with "Invalid tokens count"
```

#### Step 6: Finality
The transaction reverts. The tokens are **trapped indefinitely** because:
- The stored `tokensCount` is incorrect (0 instead of 256)
- No function exists to update or correct it
- Every `dock()` attempt with the correct token array will fail
- No alternative recovery mechanism exists

### PoC Code Example

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import "../src/AquaRouter.sol";

contract UnsafeArrayTruncationTest is Test {
    AquaRouter aquaRouter;
    
    function setUp() public {
        // Deploy AquaRouter
        aquaRouter = new AquaRouter();
    }
    
    function testArrayTruncationDoS() public {
        // Create 256 token addresses
        address[] memory tokens = new address[](256);
        for (uint i = 0; i < 256; i++) {
            tokens[i] = address(uint160(i + 1));
        }
        
        // Create a strategy
        AquaRouter.Strategy memory strategy = AquaRouter.Strategy({
            // ... strategy parameters
        });
        
        // Ship the strategy with 256 tokens
        bytes32 strategyHash = aquaRouter.ship(strategy, tokens);
        
        // Verify: tokensCount was truncated to 0
        // (Cannot directly check due to privacy, but evident from next step)
        
        // Attempt to dock with the same 256 tokens
        vm.expectRevert("Invalid tokens count");
        aquaRouter.dock(strategyHash, tokens);
        
        // Funds are now permanently trapped!
    }
}
```

### Real-World Attack Scenario

1. **Accidental:** A user with 256 tokens in their strategy accidentally ships all of them
2. **Consequence:** Funds lock immediately
3. **Discovery:** User realizes they cannot retrieve funds when calling `dock()`
4. **Resolution:** No solution exists except contract redeploy or manual intervention

---

## Recommendation

### Fix: Implement Strict Boundary Check

Add an explicit validation before the type conversion to ensure the array length does not exceed the capacity of a `uint8`.

#### Recommended Implementation

```solidity
function ship(
    AquaRouter.Strategy calldata strategy,
    address[] calldata tokens
) external returns(bytes32 strategyHash) {
    // ✓ ADD STRICT BOUNDARY CHECK
    require(
        tokens.length <= type(uint8).max,
        "Maximum tokens exceeded (max 255)"
    );
    
    strategyHash = keccak256(abi.encode(strategy));
    uint8 tokensCount = uint8(tokens.length);  // Safe: already validated
    
    // Store the strategy
    _balances[strategyHash].tokensCount = tokensCount;
    _strategies[strategyHash] = strategy;
    
    // ... rest of logic
    
    return strategyHash;
}
```

#### Alternative: Using Custom Error

```solidity
// Define custom error for better gas efficiency
error TokenCountExceeded(uint256 provided, uint256 maximum);

function ship(
    AquaRouter.Strategy calldata strategy,
    address[] calldata tokens
) external returns(bytes32 strategyHash) {
    // Validate token count doesn't exceed uint8 max
    if (tokens.length > type(uint8).max) {
        revert TokenCountExceeded(tokens.length, type(uint8).max);
    }
    
    strategyHash = keccak256(abi.encode(strategy));
    uint8 tokensCount = uint8(tokens.length);
    
    // ... rest of implementation
}
```

#### Enhanced Version with Documentation

```solidity
/**
 * @notice Ships a strategy with associated tokens
 * @dev Maximum 255 tokens per strategy due to uint8 storage constraint
 * @param strategy The strategy configuration
 * @param tokens Array of token addresses (max 255)
 * @return strategyHash Unique identifier for the strategy
 * @custom:throws TokenCountExceeded If tokens array exceeds 255 elements
 */
function ship(
    AquaRouter.Strategy calldata strategy,
    address[] calldata tokens
) external returns(bytes32 strategyHash) {
    // Enforce token count limit
    uint256 tokenCount = tokens.length;
    if (tokenCount > 255) {
        revert TokenCountExceeded(tokenCount, 255);
    }
    
    strategyHash = keccak256(abi.encode(strategy));
    
    // Safe cast: length is validated to be <= 255
    uint8 tokensCount = uint8(tokenCount);
    
    // Store balance information
    _balances[strategyHash] = StrategyBalance({
        tokensCount: tokensCount,
        // ... other fields
    });
    
    _strategies[strategyHash] = strategy;
    
    // Emit event for transparency
    emit StrategyShipped(strategyHash, tokenCount);
    
    return strategyHash;
}
```

### Additional Safeguards

#### 1. Input Validation at Contract Level

```solidity
// Add constant for maximum tokens
uint8 public constant MAX_TOKENS = type(uint8).max;  // 255

modifier validTokenCount(address[] calldata tokens) {
    require(tokens.length <= MAX_TOKENS, "Token count exceeds maximum");
    _;
}

function ship(
    AquaRouter.Strategy calldata strategy,
    address[] calldata tokens
) external validTokenCount(tokens) returns(bytes32 strategyHash) {
    // ... rest of implementation
}
```

#### 2. Documentation and Limits

```solidity
/**
 * Maximum number of tokens in a single strategy.
 * Limited to 255 due to uint8 storage optimization.
 */
uint8 public constant MAX_TOKENS_PER_STRATEGY = 255;
```

#### 3. Automated Testing

```solidity
function testBoundaryConditions() public {
    address[] memory tokens = new address[](255);
    // Should succeed
    aquaRouter.ship(strategy, tokens);
    
    address[] memory tokensOverLimit = new address[](256);
    // Should revert
    vm.expectRevert("Token count exceeds maximum");
    aquaRouter.ship(strategy, tokensOverLimit);
}
```

---

## References

### Solidity Documentation
- [Solidity: Checked vs Unchecked Arithmetic](https://docs.soliditylang.org/en/v0.8.0/control-structures.html#checked-or-unchecked-arithmetic)
- [Type Conversion and Truncation](https://docs.soliditylang.org/en/v0.8.0/types.html#explicit-conversions)
- [Integer Types and Overflow](https://docs.soliditylang.org/en/v0.8.0/types.html#integers)

### Vulnerability Context
- **Scan Report:** "Incorrect handling of tokens array length exceeding uint8 limit in ship function"
- **File:** `src/Aqua.sol`, lines 91–105
- **Pattern:** Unsafe type conversion without bounds checking
- **Similar Issues:** OpenZeppelin SafeCast library design rationale

### Related Reading
- [Common Smart Contract Vulnerabilities - Integer Overflow/Underflow](https://consensys.net/smart-contracts-best-practices/)
- [Solidity Best Practices - Input Validation](https://soliditylang.org/blog/)

### Audit Trail
- **Identified by:** Security Audit Team
- **Protocol Status:** Acknowledged
- **Classification:** Informational (High actual impact)
- **Resolution:** Awaiting implementation of recommended fix

---

## Summary

### Vulnerability Chain

```
Unsafe Type Conversion
    ↓
Array Length Truncation (256 → 0)
    ↓
Incorrect State Storage
    ↓
Dock Function Comparison Fails (0 ≠ 256)
    ↓
Permanent Fund Lock
    ↓
Denial of Service
```

### Key Takeaways

| Aspect | Details |
|--------|---------|
| **Root Cause** | Missing length validation before uint256 → uint8 conversion |
| **Trigger** | Any `ship()` call with ≥256 tokens |
| **Impact** | Permanent fund lock, impossible to recover |
| **Likelihood** | Low/Medium (accidental, not enforced) |
| **Fix Complexity** | Simple: Add 1 require statement |
| **Recommended Action** | Implement boundary check immediately |

### Implementation Checklist

- [ ] Add `require(tokens.length <= 255, "...")` before `toUint8()`
- [ ] Add unit tests for boundary conditions (255 and 256 tokens)
- [ ] Update function documentation with token count limits
- [ ] Consider using custom errors for better gas efficiency
- [ ] Add integration tests for the full ship/dock workflow
- [ ] Deploy fix to production
- [ ] Monitor for any affected strategies from the vulnerability window

---

## Document Control

- **Created:** February 18, 2026
- **Document:** unsafe-array-truncation.md
- **Protocol:** 1inch Aqua
- **Version:** 1.0
- **Status:** Documented and Recommended for Implementation

For questions regarding this finding, contact the security audit team.