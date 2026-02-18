
---
created: 2026-01-15 08:21:43
scanned: 2026-01-15 08:24:47

---

# MawariVesting and Token Smart Contracts


## Incorrect token accounting when using fee-on-transfer tokens

### Severity

HIGH

### Description

The contract adds the full `amount` to `totalAllocated` when a vesting is added, but if the token uses a fee-on-transfer mechanism, the actual received amount is less than `amount`. This leads to an overestimation of `totalAllocated`, causing incorrect free balance calculations and potential inability to withdraw tokens or fulfill vesting claims.

### Recommendation

Measure the actual token balance change after the transfer and use that value for `totalAmount` and `totalAllocated`. For example, record the contract's token balance before and after the transfer, then calculate the difference.

### Affected Files


- `MawariVesting.sol`, line 183 to line 201

 ```solidity
        token.safeTransferFrom(msg.sender, address(this), amount);
        emit Deposited(amount);
    }

    /// @notice Allows owner to withdraw tokens from the contract
    /// @param to Address to receive the withdrawn tokens
    /// @param amount Amount of tokens to withdraw
    /// @dev Ensures withdrawal doesn't reduce balance below totalAllocated to protect vesting commitments
    function withdraw(address to, uint256 amount) external onlyOwner {
        if (to == address(0)) revert InvalidAddress();
        if (amount == 0) revert InvalidAmount();

        uint256 freeBalance = getFreeBalance();

        if (amount > freeBalance) {
            revert InsufficientFreeBalance();
        }

        token.safeTransfer(to, amount);
 ```


## Deactivating a vesting schedule does not reduce totalAllocated, leading to locked tokens

### Severity

HIGH

### Description

When a vesting schedule is deactivated (set to inactive), the remaining allocated tokens (totalAmount - claimedAmount) are not subtracted from `totalAllocated`. This causes the contract to consider these tokens as still allocated, preventing the owner from withdrawing them even though they are no longer claimable. This results in permanently locked funds.

### Recommendation

When deactivating a vesting, subtract the remaining allocation (totalAmount - claimedAmount) from `totalAllocated`. When reactivating, add it back if necessary.

### Affected Files


- `MawariVesting.sol`, line 338 to line 370

 ```solidity
    function setVestingState(bytes32 roleId, bool active) external onlyOwner {
        if (!vestings[roleId].initialized) revert VestingNotInitialized();
        if (vestings[roleId].active == active) revert VestingStateAlreadySet();

        VestingSchedule storage schedule = vestings[roleId];

        if (active) {
            // Reactivating: accumulate paused time if it was paused
            // Only count paused time that occurs after the cliff period
            if (schedule.pausedAt > 0) {
                uint256 cliffEndTime = schedule.startTime +
                    (schedule.cliffDurationInMonths * SECONDS_IN_MONTH);
                // Only accumulate paused time from max(pausedAt, cliffEndTime) to now
                uint256 pauseStart = schedule.pausedAt > cliffEndTime
                    ? schedule.pausedAt
                    : cliffEndTime;
                if (block.timestamp > pauseStart) {
                    unchecked {
                        schedule.totalPausedTime +=
                            block.timestamp -
                            pauseStart;
                    }
                }
                schedule.pausedAt = 0;
            }
        } else {
            // Pausing: record the pause timestamp
            schedule.pausedAt = block.timestamp;
        }

        schedule.active = active;
        emit VestingStateChanged(roleId, active);
    }
 ```


## Incorrect usage of custom error in require statements leading to compilation failure

### Severity

HIGH

### Description

The contract uses custom errors (`MaxSupplyExceeded`) as the second argument in `require` statements. However, Solidity's `require` function expects a string message, not a custom error type. This will cause the contract to fail compilation, making it impossible to deploy. The impact is critical as the contract cannot be executed without fixing this issue.

### Recommendation

Replace the `require` statements with explicit checks using `if` conditions followed by `revert` with the custom error. For example:
```solidity
if (initialSupply > MAX_SUPPLY) revert MaxSupplyExceeded();
```

### Affected Files


- `Token.sol`, line 36 to line 36

 ```solidity
        require(initialSupply <= MAX_SUPPLY, MaxSupplyExceeded());
 ```


- `Token.sol`, line 47 to line 47

 ```solidity
        require(totalSupply() + amount <= MAX_SUPPLY, MaxSupplyExceeded());
 ```


- `Token.sol`

 ```solidity

 ```


## Incorrect reduction of totalAllocated during token claims leading to premature fund withdrawal

### Severity

HIGH

### Description

The `_releaseVesting` function incorrectly subtracts the claimed token amount from `totalAllocated`. Since `totalAllocated` represents the total tokens reserved for all vesting schedules, reducing it during claims allows the owner to withdraw these funds prematurely. This can lead to insufficient contract balance when beneficiaries attempt to claim their remaining vested tokens, resulting in failed transactions and loss of funds.

### Recommendation

Remove the line `totalAllocated -= amount;` from the `_releaseVesting` function. The `totalAllocated` should remain unchanged during claims as it represents the initial total allocation, not the remaining balance.

### Affected Files


- `MawariVesting.sol`, line 264 to line 272

 ```solidity
        unchecked {
            totalAllocated += amount;
        }

        roleToAddress[roleId] = beneficiary;
        roleToRoleId[role] = roleId;

        emit VestingAdded(
            roleId,
 ```


## Incorrect usage of custom error in require statements leading to compilation failure

### Severity

HIGH

### Description

The contract incorrectly uses custom error types (MaxSupplyExceeded) as the message parameter in require statements. In Solidity, the require function expects a string message, not a custom error type. This syntax error will cause the contract to fail compilation entirely, preventing deployment. If somehow deployed (e.g., using non-standard compiler settings), it would not behave as intended since the custom error would not be properly triggered.

### Recommendation

Replace the require statements with direct error reverts using the custom error. Change:
`require(condition, MaxSupplyExceeded());` 
to 
`if (!condition) revert MaxSupplyExceeded();`

### Affected Files


- `Token.sol`, line 36 to line 38

 ```solidity
        require(initialSupply <= MAX_SUPPLY, MaxSupplyExceeded());
        _mint(msg.sender, initialSupply);
    }
 ```

