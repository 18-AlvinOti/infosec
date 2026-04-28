
---
created: 2025-09-08 17:00:53
scanned: 2025-09-08 17:10:38

---

# Rescan 1inch Limit Order Protocol Selected Contracts


## Incorrect mask for threshold amount leading to truncated values

### Severity

HIGH

### Description

The `threshold` function uses a mask (`_AMOUNT_MASK`) that only covers 160 bits instead of the intended 185 bits. This results in truncation of the threshold value, potentially allowing takers to spend more than the intended maximum. The incorrect mask leads to loss of the upper 25 bits of the threshold, causing critical miscalculations in order parameters.

### Recommendation

Adjust `_AMOUNT_MASK` to cover the correct 185 bits. For example, use `0x01ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff` to represent a 185-bit mask, ensuring all relevant bits are included in the threshold calculation.

### Affected Files


- `contracts/libraries/TakerTraitsLib.sol`, line 33 to line 34

 ```solidity
    uint256 private constant _AMOUNT_MASK =
        0x000000000000000000ffffffffffffffffffffffffffffffffffffffffffffff;
 ```


## Order execution functions are not paused when the contract is paused, allowing order processing to continue despite the pause.

### Severity

HIGH

### Description

The contract inherits from `Pausable` and `OrderMixin`, but the `OrderMixin` functions (e.g., order filling) are not protected by the `whenNotPaused` modifier. When the owner calls `pause()`, the intended behavior is to halt all trading functionality. However, since the `OrderMixin` functions do not check the paused state, users can continue to execute orders even when the pause is active. This defeats the purpose of the pause feature, potentially allowing unwanted trades during emergencies or maintenance.

### Recommendation

Apply the `whenNotPaused` modifier to all external functions in `OrderMixin` that handle order creation, filling, or cancellation. This ensures these functions are disabled when the contract is paused. If `OrderMixin` is designed to be pausable, the modifier should be added in its function definitions or enforced via overrides in `LimitOrderProtocol`.

### Affected Files


- `contracts/LimitOrderProtocol.sol`, line 31 to line 38

 ```solidity
contract LimitOrderProtocol is
    EIP712("1inch Limit Order Protocol", "4"),
    Ownable,
    Pausable,
    OrderMixin
{
    // solhint-disable-next-line no-empty-blocks
    constructor(IWETH _weth) OrderMixin(_weth) Ownable(msg.sender) {}
 ```


## Incorrect calculation of remaining invalidator leading to wrong remaining amounts

### Severity

HIGH

### Description

The `remains` function incorrectly calculates the invalidator by taking the inverse of `(remainingMakingAmount - makingAmount)` instead of the inverse of `remainingMakingAmount`. This results in incorrect remaining amounts being stored, causing invalid order invalidations. This can allow orders to be improperly filled or rejected, leading to financial loss and disruption of intended protocol behavior.

### Recommendation

Correct the calculation by inverting `remainingMakingAmount` directly. Replace `~(remainingMakingAmount - makingAmount)` with `~remainingMakingAmount` in the `remains` function.

### Affected Files


- `contracts/libraries/RemainingInvalidatorLib.sol`, line 73 to line 83

 ```solidity
    function remains(
        uint256 remainingMakingAmount,
        uint256 makingAmount
    ) internal pure returns (RemainingInvalidator) {
        unchecked {
            return
                RemainingInvalidator.wrap(
                    ~(remainingMakingAmount - makingAmount)
                );
        }
    }
 ```

