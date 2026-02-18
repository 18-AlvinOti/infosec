
---
created: 2025-09-08 17:09:14
scanned: 2025-09-08 17:19:56

---

# Valantis-stex selected struct Solidity files


## Potential truncation of token amount due to use of uint96 for amountToken1

### Severity

HIGH

### Description

The `LPWithdrawalRequest` struct uses `uint96` for `amountToken1`. If the actual amount exceeds the maximum value of uint96 (2^96 - 1 ≈ 7.9e28), it will be truncated, leading to incorrect accounting. This can result in loss of funds for users attempting to withdraw large amounts, as the stored value would be lower than intended. High risk because it directly affects core accounting logic.

### Recommendation

Change the data type of `amountToken1` from `uint96` to `uint256` to prevent truncation and ensure the full range of possible token amounts can be represented accurately.

### Affected Files


- `src/structs/WithdrawalModuleStructs.sol`, line 4 to line 8

 ```solidity
struct LPWithdrawalRequest {
    address recipient;
    uint96 amountToken1;
    uint256 cumulativeAmountToken1LPWithdrawalCheckpoint;
}
 ```

