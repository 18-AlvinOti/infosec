
---
created: 2025-12-13 02:20:33
scanned: 2025-12-13 02:25:46

---

# Nado Contracts Reloaded Files Scan


## Failed slow mode transactions are marked as processed, leading to state inconsistencies.

### Severity

HIGH

### Description

In the `_executeSlowModeTransaction` function, the code deletes the slow mode transaction and increments the transaction index (`txUpTo`) before processing the transaction. If the transaction processing fails (except for out-of-gas errors), the transaction is still considered processed, leading to skipped transactions and potential loss of funds or incorrect state updates. This can result in failed transactions (e.g., deposits) not being retried, leaving user funds stuck or the protocol in an inconsistent state.

### Recommendation

Process the transaction first and only increment `txUpTo` and delete the transaction if it succeeds. Move the deletion and increment after successful execution to ensure atomicity.

### Affected Files


- `core/contracts/Endpoint.sol`, line 421 to line 441

 ```solidity
        SlowModeTx memory txn = slowModeTxs[_slowModeConfig.txUpTo];
        delete slowModeTxs[_slowModeConfig.txUpTo++];

        require(
            fromSequencer || (txn.executableAt <= block.timestamp),
            ERR_SLOW_TX_TOO_RECENT
        );

        if (block.chainid == 31337) {
            // for testing purposes, we don't fail silently when the chainId is hardhat's default.
            this.processSlowModeTransaction(txn.sender, txn.tx);
        } else {
            uint256 gasRemaining = gasleft();
            // solhint-disable-next-line no-empty-blocks
            try this.processSlowModeTransaction(txn.sender, txn.tx) {} catch {
                // we need to differentiate between a revert and an out of gas
                // the issue is that in evm every inner call only 63/64 of the
                // remaining gas in the outer frame is forwarded. as a result
                // the amount of gas left for execution is (63/64)**len(stack)
                // and you can get an out of gas while spending an arbitrarily
                // low amount of gas in the final frame. we use a heuristic
 ```


## DumpFees transaction does not collect fees from Perp products, leading to unclaimed funds.

### Severity

MEDIUM

### Description

The `DumpFees` transaction in the `processSlowModeTransaction` function only collects sequencer fees from Spot products, ignoring Perp products. This results in accumulated fees from Perp transactions remaining unclaimed in the contract, causing a loss of protocol revenue.

### Recommendation

Modify the DumpFees transaction to collect fees from both Spot and Perp products by including product IDs from the Perp engine in the fee collection loop.

### Affected Files


- `core/contracts/Endpoint.sol`, line 508 to line 515

 ```solidity
            uint32[] memory spotIds = spotEngine.getProductIds();
            int128[] memory fees = new int128[](spotIds.length);
            for (uint256 i = 0; i < spotIds.length; i++) {
                fees[i] = sequencerFee[spotIds[i]];
                sequencerFee[spotIds[i]] = 0;
            }
            requireSubaccount(X_ACCOUNT);
            clearinghouse.claimSequencerFees(fees);
 ```


## Owner can withdraw arbitrary amounts of tokens, not limited to collected fees

### Severity

HIGH

### Description

The `removeLiquidity` function allows the contract owner to withdraw any amount of tokens from the contract, regardless of the collected fees tracked in the `fees` mapping. This enables the owner to drain all tokens held by the contract, including those not intended as fees, leading to loss of user funds.

### Recommendation

Modify the `removeLiquidity` function to deduct the withdrawn amount from the `fees` mapping and ensure the amount does not exceed the collected fees for the product.

### Affected Files


- `core/contracts/BaseWithdrawPool.sol`, line 116 to line 122

 ```solidity
    function removeLiquidity(
        uint32 productId,
        uint128 amount,
        address sendTo
    ) external onlyOwner {
        handleWithdrawTransfer(getToken(productId), sendTo, amount);
    }
 ```


## Missing idx validation in submitWithdrawal allows minIdx rollback

### Severity

MEDIUM

### Description

The `submitWithdrawal` function does not validate that the provided `idx` is greater than the current `minIdx`. This allows the clearinghouse to potentially decrease `minIdx`, enabling fast withdrawals with lower indices than previously allowed, which could bypass intended sequencing controls.

### Recommendation

Add a `require(idx > minIdx, "Invalid index")` check in `submitWithdrawal` to ensure monotonic increase of `minIdx`.

### Affected Files


- `core/contracts/BaseWithdrawPool.sol`, line 81 to line 97

 ```solidity
    function submitWithdrawal(
        IERC20Base token,
        address sendTo,
        uint128 amount,
        uint64 idx
    ) public {
        require(msg.sender == clearinghouse);

        if (markedIdxs[idx]) {
            return;
        }
        markedIdxs[idx] = true;
        // set minIdx to most recent withdrawal submitted by sequencer
        minIdx = idx;

        handleWithdrawTransfer(token, sendTo, amount);
    }
 ```


## Infinite loop in `pow` function due to integer overflow of loop variable

### Severity

HIGH

### Description

The `pow` function uses a loop variable `i` of type `int128` that is multiplied by 2 in each iteration. When `y` is large enough (e.g., `y >= 2^127`), multiplying `i` by 2 causes an overflow, turning `i` into a negative value. This results in an infinite loop as the loop condition `i <= y` remains true indefinitely, leading to transaction reverts due to out-of-gas errors. This makes the function unusable for large exponents and causes denial of service.

### Recommendation

Replace the loop variable `i` with a `uint256` and iterate over each bit of `y` using bitwise operations to avoid overflow. For example, use a bit index from 0 to 127 and check each bit of `y` using a right shift. This approach ensures the loop terminates correctly without integer overflow risks.

### Affected Files


- `core/contracts/libraries/MathSD21x18.sol`, line 79 to line 91

 ```solidity
    function pow(int128 x, int128 y) internal pure returns (int128) {
        unchecked {
            require(y >= 0, ERR_OVERFLOW);
            int128 result = ONE_X18;
            for (int128 i = 1; i <= y; i *= 2) {
                if (i & y != 0) {
                    result = mul(result, x);
                }
                x = mul(x, x);
            }
            return result;
        }
    }
 ```


## Tokens transferred before slow mode transaction processing can lead to loss of funds

### Severity

HIGH

### Description

In the `depositCollateralWithReferral` function, tokens are transferred to the clearinghouse immediately when the user submits the transaction. If the subsequent slow mode transaction processing fails (e.g., due to invalid product ID), the tokens remain in the clearinghouse but are not credited to the user's subaccount, resulting in permanent loss of funds.

### Recommendation

Transfer tokens to the clearinghouse only after the slow mode transaction is successfully processed. Consider holding the tokens in escrow within the Endpoint contract until the transaction is confirmed.

### Affected Files


- `core/contracts/Endpoint.sol`, line 263 to line 303

 ```solidity
    function depositCollateralWithReferral(
        bytes32 subaccount,
        uint32 productId,
        uint128 amount,
        string memory
    ) public {
        require(!RiskHelper.isIsolatedSubaccount(subaccount), ERR_UNAUTHORIZED);

        address sender = address(bytes20(subaccount));

        // depositor / depositee need to be unsanctioned
        requireUnsanctioned(msg.sender);
        requireUnsanctioned(sender);

        if (!isValidDepositAmount(subaccount, productId, amount)) {
            // we cannot revert here, otherwise direct deposit could be blocked when there are
            // multiple assets awaiting credit but one of them is below the minimum deposit amount.
            // we can just skip the deposit and continue with the next asset.
            return;
        }

        handleDepositTransfer(
            IERC20Base(spotEngine.getToken(productId)),
            msg.sender,
            uint256(amount)
        );
        // copy from submitSlowModeTransaction
        SlowModeConfig memory _slowModeConfig = slowModeConfig;

        slowModeTxs[_slowModeConfig.txCount++] = SlowModeTx({
            executableAt: uint64(block.timestamp) + SLOW_MODE_TX_DELAY, // hardcoded to three days
            sender: sender,
            tx: abi.encodePacked(
                uint8(TransactionType.DepositCollateral),
                abi.encode(
                    DepositCollateral({
                        sender: subaccount,
                        productId: productId,
                        amount: amount
                    })
                )
 ```


## Fee charged before transfer may lead to loss of funds on failure

### Severity

HIGH

### Description

In the `TransferQuote` transaction processing, the fee is charged before executing the transfer. If the transfer fails (e.g., due to insufficient balance), the fee is not refunded, causing users to lose the fee without completing the intended operation.

### Recommendation

Perform the transfer first and charge the fee only after ensuring the transfer is successful. Use a checks-effects-interactions pattern to avoid premature state changes.

### Affected Files


- `core/contracts/Endpoint.sol`, line 709 to line 729

 ```solidity
        } else if (txType == TransactionType.TransferQuote) {
            SignedTransferQuote memory signedTx = abi.decode(
                transaction[1:],
                (SignedTransferQuote)
            );
            _recordSubaccount(signedTx.tx.recipient);
            validateSignature(
                signedTx.tx.sender,
                _hashTypedDataV4(computeDigest(txType, transaction[1:])),
                signedTx.signature
            );
            validateNonce(signedTx.tx.sender, signedTx.tx.nonce);
            if (
                RiskHelper.isIsolatedSubaccount(signedTx.tx.recipient) ||
                RiskHelper.isIsolatedSubaccount(signedTx.tx.sender)
            ) {
                chargeFee(signedTx.tx.sender, HEALTHCHECK_FEE / 10);
            } else {
                chargeFee(signedTx.tx.sender, HEALTHCHECK_FEE);
            }
            clearinghouse.transferQuote(signedTx.tx);
 ```


## Negative fee calculation allows increasing transfer amount, leading to loss of funds.

### Severity

HIGH

### Description

If `withdrawFeeX18` is negative, the calculated `fee` in `fastWithdrawalFeeAmount` becomes negative. This causes `transferAmount` to increase instead of decrease, allowing users to withdraw more funds than allowed, potentially draining the contract's balance.

### Recommendation

Add a check to ensure `feeX18` is non-negative (e.g., `require(feeX18 >= 0, "Invalid fee")`) and validate `withdrawFeeX18` in the engine configuration to prevent negative values.

### Affected Files


- `core/contracts/BaseWithdrawPool.sol`, line 99 to line 108

 ```solidity
    function fastWithdrawalFeeAmount(
        IERC20Base token,
        uint32 productId,
        uint128 amount
    ) public view returns (int128) {
        uint8 decimals = token.decimals();
        require(decimals <= MAX_DECIMALS);
        int256 multiplier = int256(10 ** (MAX_DECIMALS - uint8(decimals)));
        int128 amountX18 = int128(amount) * int128(multiplier);

 ```


## Potential integer overflow in minFeeX18 calculation.

### Severity

MEDIUM

### Description

Multiplying `withdrawFeeX18` by 5 without overflow checks can result in an integer overflow if `withdrawFeeX18` is sufficiently large. This may lead to incorrect fee calculations, causing undercharging or overcharging users.

### Recommendation

Use safe arithmetic library functions (e.g., `mul` from `MathSD21x18`) to handle multiplication and prevent overflows.

### Affected Files


- `core/contracts/BaseWithdrawPool.sol`, line 110 to line 110

 ```solidity
        int128 minFeeX18 = 5 * spotEngine().getConfig(productId).withdrawFeeX18;
 ```


## Fees can accumulate negative values, leading to incorrect accounting.

### Severity

MEDIUM

### Description

The `fees` mapping uses `int128`, allowing negative values if a negative `fee` is added. This disrupts fee tracking and may cause underflow in dependent logic expecting non-negative fees.

### Recommendation

Ensure `fee` is non-negative before updating `fees` (e.g., `require(fee >= 0, "Negative fee")`) and use `uint128` for the `fees` mapping to enforce non-negative values.

### Affected Files


- `core/contracts/BaseWithdrawPool.sol`, line 76 to line 76

 ```solidity
        fees[signedTx.tx.productId] += fee;
 ```

