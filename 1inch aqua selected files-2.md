
---
created: 2025-11-19 01:13:48


---

#  1inch aqua selected files


## Potential reentrancy via Multicall due to lack of reentrancy guards

### Severity

HIGH

### Description

The AquaRouter contract inherits from Multicall, enabling batch function calls. If functions in Aqua or Simulator perform external calls without reentrancy protection, attackers could exploit the multicall function to re-enter the contract, potentially manipulating state or draining funds. This is critical if external interactions in parent contracts are not safeguarded.

### Recommendation

Implement reentrancy guards (e.g., OpenZeppelin's ReentrancyGuard) on all external-facing functions in Aqua and Simulator, especially those interacting with untrusted contracts. Additionally, apply a reentrancy guard to the multicall function itself.

### Affected Files


- `src/AquaRouter.sol`, line 11 to line 11

 ```solidity
contract AquaRouter is Aqua, Simulator, Multicall {}
 ```


## Inheritance order may cause unintended function overrides

### Severity

MEDIUM

### Description

The linearization of AquaRouter's inheritance (Aqua → Simulator → Multicall) might lead to unexpected function overrides if parent contracts share function names. For example, if Aqua and Simulator both implement a critical function, the Simulator's version could override Aqua's, bypassing intended logic and causing vulnerabilities.

### Recommendation

Explicitly override conflicting functions using the `override` keyword and audit parent contracts to ensure intended behavior. Use distinct function names or adjust inheritance order if necessary.

### Affected Files


- `src/AquaRouter.sol`, line 11 to line 11

 ```solidity
contract AquaRouter is Aqua, Simulator, Multicall {}
 ```


## Reentrancy lock remains locked if the function reverts, leading to denial of service

### Severity

HIGH

### Description

The `nonReentrantStrategy` modifier locks a `strategyHash` but does not unlock it if the function execution reverts. This leaves the `strategyHash` permanently locked in the `_reentrancyLocks` mapping, preventing any future calls to functions using the same `strategyHash`. This results in a denial of service (DoS) for those strategies, as subsequent transactions will fail due to the lock remaining active.

### Recommendation

Restructure the locking mechanism to ensure the unlock operation occurs even if the function reverts. This can be achieved by using an internal function` that wraps the business logic, allowing the modifier to handle the lock and unlock operations safely. For example:

```solidity
modifier nonReentrantStrategy(bytes32 strategyHash) {
    _reentrancyLocks[strategyHash].lock();
    _nonReentrantStrategyExecution(strategyHash);
    _reentrancyLocks[strategyHash].unlock();
}

function _nonReentrantStrategyExecution(bytes32 strategyHash) private {
    _;
}
```

This approach ensures the `unlock` is called after the private function execution, even if it reverts. However, note that Solidity's modifier semantics may still pose challenges, and alternative designs (e.g., using a transient lock flag reset on each call) should be considered to avoid persistent state corruption.

### Affected Files


- `src/AquaApp.sol`, line 36 to line 40

 ```solidity
    modifier nonReentrantStrategy(bytes32 strategyHash) {
        _reentrancyLocks[strategyHash].lock();
        _;
        _reentrancyLocks[strategyHash].unlock();
    }
 ```


- `src/AquaApp.sol`, line 36 to line 44

 ```solidity
    modifier nonReentrantStrategy(bytes32 strategyHash) {
        _reentrancyLocks[strategyHash].lock();
        _;
        _reentrancyLocks[strategyHash].unlock();
    }

    constructor(IAqua aqua) {
        AQUA = aqua;
    }
 ```


## Missing check for equal lengths of tokens and amounts arrays in ship function

### Severity

HIGH

### Description

The `ship` function does not validate that the `tokens` and `amounts` arrays have the same length. If these arrays have different lengths, it can lead to out-of-bounds access when the `amounts` array is shorter than `tokens`, causing the transaction to revert. This results in failed transactions and potential loss of gas fees.

### Recommendation

Add `require(tokens.length == amounts.length, "Mismatched array lengths");` at the beginning of the `ship` function.

### Affected Files


- `src/Aqua.sol`, line 91 to line 116

 ```solidity
    function ship(
        address app,
        bytes calldata strategy,
        address[] calldata tokens,
        uint256[] calldata amounts
    ) external returns (bytes32 strategyHash) {
        strategyHash = keccak256(strategy);
        uint8 tokensCount = tokens.length.toUint8();
        require(
            tokensCount != _DOCKED,
            MaxNumberOfTokensExceeded(tokensCount, _DOCKED)
        );

        emit Shipped(msg.sender, app, strategyHash, strategy);
        for (uint256 i = 0; i < tokens.length; i++) {
            Balance storage balance = _balances[msg.sender][app][strategyHash][
                tokens[i]
            ];
            require(
                balance.tokensCount == 0,
                StrategiesMustBeImmutable(app, strategyHash)
            );
            balance.store(amounts[i].toUint248(), tokensCount);
            emit Pushed(msg.sender, app, strategyHash, tokens[i], amounts[i]);
        }
    }
 ```


## Missing balance check in pull function leading to underflow

### Severity

HIGH

### Description

The `pull` function lacks a check to ensure the requested `amount` does not exceed the available balance. If `amount` exceeds `prevBalance`, an underflow occurs during subtraction, reverting the transaction. This can be exploited to disrupt contract operations and deny service.

### Recommendation

Insert `require(amount <= prevBalance, "Insufficient balance");` after loading `prevBalance`.

### Affected Files


- `src/Aqua.sol`, line 136 to line 151

 ```solidity
    function pull(
        address maker,
        bytes32 strategyHash,
        address token,
        uint256 amount,
        address to
    ) external {
        Balance storage balance = _balances[maker][msg.sender][strategyHash][
            token
        ];
        (uint248 prevBalance, uint8 tokensCount) = balance.load();
        balance.store(prevBalance - amount.toUint248(), tokensCount);

        IERC20(token).safeTransferFrom(maker, to, amount);
        emit Pulled(maker, msg.sender, strategyHash, token, amount);
    }
 ```


## Incorrect handling of tokens array length exceeding uint8 limit in ship function

### Severity

MEDIUM

### Description

The `ship` function converts `tokens.length` to `uint8`, which truncates values over 255. This leads to incorrect `tokensCount` storage, making docking impossible and funds irrecoverable if the array exceeds 255 elements.

### Recommendation

Add `require(tokens.length <= type(uint8).max, "Too many tokens");` to enforce a maximum of 255 tokens per strategy.

### Affected Files


- `src/Aqua.sol`, line 91 to line 105

 ```solidity
    function ship(
        address app,
        bytes calldata strategy,
        address[] calldata tokens,
        uint256[] calldata amounts
    ) external returns (bytes32 strategyHash) {
        strategyHash = keccak256(strategy);
        uint8 tokensCount = tokens.length.toUint8();
        require(
            tokensCount != _DOCKED,
            MaxNumberOfTokensExceeded(tokensCount, _DOCKED)
        );

        emit Shipped(msg.sender, app, strategyHash, strategy);
        for (uint256 i = 0; i < tokens.length; i++) {
 ```


## Potential AMM price oracle manipulation via batched multicall transactions

### Severity

HIGH

### Description

The AquaRouter inherits Multicall functionality, allowing multiple function calls in a single transaction. If Aqua or Simulator contain functions that interact with AMM pools (e.g., swaps, liquidity changes), an attacker could manipulate the pool's price oracle within one transaction. For example, an attacker could perform a series of swaps and liquidity operations in a single multicall to artificially move the AMM's price before executing a trade at the manipulated rate, leading to incorrect pricing and potential fund loss.

### Recommendation

1) Implement transaction-level locks to prevent multiple state-changing AMM interactions in a single transaction. 2) Use time-weighted average price (TWAP) oracles instead of spot prices. 3) Add slippage protections and validate price changes between consecutive operations in batched calls.

### Affected Files


- `src/AquaRouter.sol`, line 11 to line 11

 ```solidity
contract AquaRouter is Aqua, Simulator, Multicall {}
 ```


## Missing check for equal array lengths in `ship` function leading to potential out-of-bounds access

### Severity

HIGH

### Description

The `ship` function does not validate that the `tokens` and `amounts` arrays have the same length. If these arrays are of different lengths, accessing `amounts[i]` in the loop will cause an out-of-bounds error, leading to transaction reverts. This can disrupt the expected functionality of the contract, causing failed transactions and potential loss of user funds if not properly handled.

### Recommendation

Add a require statement to ensure `tokens.length == amounts.length` at the beginning of the `ship` function to prevent mismatched array lengths.

### Affected Files


- `src/Aqua.sol`, line 91 to line 116

 ```solidity
    function ship(
        address app,
        bytes calldata strategy,
        address[] calldata tokens,
        uint256[] calldata amounts
    ) external returns (bytes32 strategyHash) {
        strategyHash = keccak256(strategy);
        uint8 tokensCount = tokens.length.toUint8();
        require(
            tokensCount != _DOCKED,
            MaxNumberOfTokensExceeded(tokensCount, _DOCKED)
        );

        emit Shipped(msg.sender, app, strategyHash, strategy);
        for (uint256 i = 0; i < tokens.length; i++) {
            Balance storage balance = _balances[msg.sender][app][strategyHash][
                tokens[i]
            ];
            require(
                balance.tokensCount == 0,
                StrategiesMustBeImmutable(app, strategyHash)
            );
            balance.store(amounts[i].toUint248(), tokensCount);
            emit Pushed(msg.sender, app, strategyHash, tokens[i], amounts[i]);
        }
    }
 ```

