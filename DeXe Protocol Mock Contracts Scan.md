
---
created: 2025-09-11 00:06:21
scanned: 2025-09-11 00:10:03

---

# DeXe Protocol Mock Contracts Scan


## Missing access control on critical functions allowing unauthorized state changes and token transfers.

### Severity

HIGH

### Description

The `setTransferAmount` function allows any user to set the `amount20` state variable, which determines how many ERC20 tokens are transferred when `execute` is called. Additionally, the `execute` function can be called by any user, potentially transferring tokens from the `govAddress` if it has approved this contract. This allows unauthorized users to manipulate transfer amounts and trigger token transfers without proper authorization, leading to potential theft of funds.

### Recommendation

Add access control checks to ensure only authorized addresses (e.g., `govAddress`) can call `setTransferAmount` and `execute`. Use modifiers like `onlyGov` to restrict access to these functions.

### Affected Files


- `contracts/mock/gov/ExecutorTransferMock.sol`, line 21 to line 35

 ```solidity
    function setTransferAmount(uint256 _amount20) external {
        amount20 = _amount20;
    }

    function execute() external payable {
        address _govAddress = govAddress;

        if (amount20 > 0) {
            IERC20(mock20Address).transferFrom(
                _govAddress,
                address(this),
                amount20
            );
        }
    }
 ```


## Missing access control on critical `setPower` function allowing any user to modify internal state

### Severity

HIGH

### Description

The `setPower` function is external and lacks access control, allowing any user to modify the `_power` state variable. Although `_power` is not used in current logic, in a real implementation this could control critical voting power calculations. Unauthorized modification of privileged state variables violates access control principles and can lead to governance manipulation.

### Recommendation

Add the `onlyOwner` modifier to the `setPower` function to restrict access to the contract's owner.

### Affected Files


- `contracts/mock/gov/VotePowerMock.sol`, line 17 to line 19

 ```solidity
    function setPower(uint256 power) external {
        _power = power;
    }
 ```


## Unused `_power` state variable leads to incorrect vote transformation calculations.

### Severity

HIGH

### Description

The `transformVotes` and `transformVotesFull` functions return `votes * votes` instead of using the `_`power` state variable. This indicates a logical error where the configured power value is ignored, leading to incorrect vote power calculations. The contract's core functionality (vote power transformation) is fundamentally broken as a result.

### Recommendation

Modify the transformation functions to incorporate the `_power` variable (e.g., `return votes * _power;`) and ensure the `setPower` function properly updates the variable used in calculations.

### Affected Files


- `contracts/mock/gov/VotePowerMock.sol`, line 21 to line 36

 ```solidity
    function transformVotes(
        address,
        uint256 votes
    ) external pure override returns (uint256 resultingVotes) {
        return votes * votes;
    }

    function transformVotesFull(
        address,
        uint256 votes,
        uint256,
        uint256,
        uint256
    ) external pure override returns (uint256 resultingVotes) {
        return votes * votes;
    }
 ```

