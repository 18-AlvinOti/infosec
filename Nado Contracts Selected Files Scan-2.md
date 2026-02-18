
---
created: 2025-12-13 02:09:16
scanned: 2025-12-13 02:14:34

---

# Nado Contracts Selected Files Scan


## Incorrect fee calculation for taker sell orders leading to negative fees

### Severity

HIGH

### Description

When a taker order is a sell (negative `matchQuote`), the fee calculation incorrectly inverts the minimum fee, leading to negative fees (rebates) instead of positive fees. This allows takers to pay less than required or even receive rebates, reducing protocol revenue and enabling potential exploitation.

### Recommendation

Remove the sign inversion for `meteredQuote` when `matchQuote` is negative. Instead, apply the minimum fee as a positive value based on the direction of the trade.

### Affected Files


- `core/contracts/OffchainExchange.sol`, line 449 to line 469

 ```solidity
        if (taker) {
            // flat minimum fee
            if (alreadyMatched == 0) {
                meteredQuote += market.minSize;
                if (matchQuote < 0) {
                    meteredQuote = -meteredQuote;
                }
            }

            // exclude the portion on [0, self.min_size) for match_quote and
            // add to metered_quote
            // fee is only applied on [minSize, quote_amount)
            int128 feeApplied = MathHelper.abs(alreadyMatched + matchQuote) -
                market.minSize;
            feeApplied = MathHelper.min(feeApplied, matchQuote.abs());
            if (feeApplied > 0) {
                if (matchQuote < 0) {
                    feeApplied = -feeApplied;
                }
                meteredQuote += feeApplied;
            }
 ```


## Inverted price check in order matching logic

### Severity

HIGH

### Description

The price validation for matching orders uses inverted comparisons, allowing orders to match when maker prices are worse than taker prices. This leads to incorrect trade executions where users get worse prices than expected, resulting in financial losses.

### Recommendation

Reverse the comparison operators. For maker selling (amount > 0), require `maker.priceX18 <= taker.priceX18`. For maker buying (amount < 0), require `maker.priceX18 >= taker.priceX18`.

### Affected Files


- `core/contracts/OffchainExchange.sol`, line 682 to line 692

 ```solidity
        if (maker.order.amount > 0) {
            require(
                maker.order.priceX18 >= taker.order.priceX18,
                ERR_ORDERS_CANNOT_BE_MATCHED
            );
        } else {
            require(
                maker.order.priceX18 <= taker.order.priceX18,
                ERR_ORDERS_CANNOT_BE_MATCHED
            );
        }
 ```


## Debt transfer to parent subaccount without health check

### Severity

MEDIUM

### Description

When closing an isolated subaccount, any remaining debt (negative vQuoteBalance) is transferred to the parent without checking if the parent can sustain it. This can make the parent account undercollateralized, potentially leading to bad debt in the system.

### Recommendation

Add a health check for the parent subaccount after transferring balances to ensure it remains properly collateralized. Revert if the transfer would make the parent unhealthy.

### Affected Files


- `core/contracts/OffchainExchange.sol`, line 162 to line 175

 ```solidity
            if (balance.vQuoteBalance != 0) {
                perpEngine.updateBalance(
                    productId,
                    subaccount,
                    0,
                    -balance.vQuoteBalance
                );
                perpEngine.updateBalance(
                    productId,
                    parent,
                    0,
                    balance.vQuoteBalance
                );
            }
 ```


## Undeclared variable 'productIds' leads to compilation errors or incorrect product ID references.

### Severity

HIGH

### Description

The `updateStates` function references `productIds[i]`, but `productIds` is not declared in the contract. This results in a compilation error or references to an undefined array, causing incorrect product state updates or runtime errors. This can lead to manipulating the wrong product's state, affecting funding calculations and balances.

### Recommendation

Declare the `productIds` array in the contract or pass it as a function parameter. Ensure the array length matches `avgPriceDiffs.length` to prevent out-of-bounds access.

### Affected Files


- `core/contracts/PerpEngineState.sol`, line 103 to line 106

 ```solidity
        for (uint32 i = 0; i < avgPriceDiffs.length; i++) {
            uint32 productId = productIds[i];
            State memory state = states[productId];
            if (state.openInterest == 0) {
 ```


## Undeclared 'SECONDS_PER_DAY' causes compilation error.

### Severity

HIGH

### Description

The `require` statement uses `SECONDS_PER_DAY`, which is not defined in the provided code. This prevents the contract from compiling, rendering it non-functional and halting all operations dependent on state updates.

### Recommendation

Define `SECONDS_PER_DAY` as a constant (e.g., `86400`) in the contract or import it from the correct source to resolve the compilation issue.

### Affected Files


- `core/contracts/PerpEngineState.sol`, line 109 to line 109

 ```solidity
            require(dt < 7 * SECONDS_PER_DAY, ERR_INVALID_TIME);
 ```


## Missing initialization access control allowing re-initialization

### Severity

HIGH

### Description

The `initialize` function in the WithdrawPool contract is declared as `external` without any access control modifiers. If the parent `BaseWithdrawPool` contract's `_initialize` function does not include initialization guards (e.g., OpenZeppelin's `initializer` modifier), an attacker could call `initialize` after deployment to reset the `clearinghouse` and `verifier` addresses, leading to privilege escalation and potential control over the contract.

### Recommendation

Ensure the `_initialize` function in `BaseWithdrawPool` uses the `initializer` modifier from OpenZeppelin's upgradeable contracts framework to prevent re-initialization. Additionally, restrict the `initialize` function with the `onlyInitializing` modifier to enforce proper initialization flow.

### Affected Files


- `core/contracts/WithdrawPool.sol`, line 16 to line 18

 ```solidity
    function initialize(address _clearinghouse, address _verifier) external {
        _initialize(_clearinghouse, _verifier);
    }
 ```


## Missing check for reduce-only orders when reusing existing isolated subaccounts

### Severity

HIGH

### Description

When a reduce-only order is placed using an existing isolated subaccount, the contract does not validate if the order actually reduces the position. This allows an attacker to increase their position in the same direction as the existing subaccount's position, violating the reduce-only constraint. This can lead to unintended positions and potential liquidation risks.

### Recommendation

Add a check for reduce-only orders when reusing existing isolated subaccounts to ensure the order's direction reduces the current position. Verify that the order's amount is opposite to the existing position's direction.

### Affected Files


- `core/contracts/OffchainExchange.sol`, line 925 to line 951

 ```solidity
        if (newIsolatedSubaccount == bytes32(0)) {
            require(
                !_isReduceOnly(txn.order.appendix),
                "Reduce-only order cannot create isolated subaccount"
            );
            require(
                mask != (1 << MAX_ISOLATED_SUBACCOUNTS_PER_ADDRESS) - 1,
                "Too many isolated subaccounts"
            );
            uint8 id = 0;
            while (mask & 1 != 0) {
                mask >>= 1;
                id += 1;
            }

            // |  address | reserved | productId |   id   |  'iso'  |
            // | 20 bytes |  6 bytes |  2 bytes  | 1 byte | 3 bytes |
            newIsolatedSubaccount = bytes32(
                (uint256(uint160(senderAddress)) << 96) |
                    (uint256(txn.productId) << 32) |
                    (uint256(id) << 24) |
                    6910831
            );
            isolatedSubaccountsMask[senderAddress] |= 1 << id;
            parentSubaccounts[newIsolatedSubaccount] = txn.order.sender;
            isolatedSubaccounts[txn.order.sender][id] = newIsolatedSubaccount;
        }
 ```


## Unbounded growth of custom fee addresses array leading to gas inefficiency

### Severity

MEDIUM

### Description

The `customFeeAddresses` array is never pruned when a user's fee tier is reset to zero. This causes the array to grow indefinitely, leading to potential out-of-gas errors when accessing the array and increased gas costs for transactions that process this array.

### Recommendation

Implement a mechanism to remove users from `customFeeAddresses` when their fee tier is set to zero, or use a mapping to track active custom fee tiers instead of an array. Alternatively, manage the array with add/remove functions to prevent unbounded growth.

### Affected Files


- `core/contracts/OffchainExchange.sol`, line 844 to line 852

 ```solidity
    function updateFeeTier(address user, uint32 newTier) external {
        require(msg.sender == address(clearinghouse), ERR_UNAUTHORIZED);
        if (newTier != 0 && !addressTouched[user]) {
            addressTouched[user] = true;
            customFeeAddresses.push(user);
        }
        feeTiers[user] = newTier;
        emit FeeTierUpdate(user, newTier);
    }
 ```


## Incorrect funding rate calculation by updating both long and short cumulative funding

### Severity

HIGH

### Description

The `updateStates` function incorrectly adds the same `paymentAmount` to both `cumulativeFundingLongX18` and `cumulativeFundingShortX18`. This results in both long and short positions being charged the funding payment, which is incorrect. Funding payments should transfer from one side (e.g., longs) to the other (shorts). This bug causes all positions to lose value, leading to incorrect accounting and loss of funds for users.

### Recommendation

Adjust the funding calculation so that only one side is updated. For example, if the funding rate is positive, longs pay shorts, so `cumulativeFundingLongX18` should increase by `paymentAmount` and `cumulativeFundingShortX18` should remain unchanged (or vice versa depending on the intended direction).

### Affected Files


- `core/contracts/PerpEngineState.sol`, line 126 to line 127

 ```solidity
                state.cumulativeFundingLongX18 += paymentAmount;
                state.cumulativeFundingShortX18 += paymentAmount;
 ```

