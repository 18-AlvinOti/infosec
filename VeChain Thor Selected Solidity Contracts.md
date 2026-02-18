
---
created: 2025-10-22 07:48:31
scanned: 2025-11-02 00:56:14

---

# VeChain Thor Selected Solidity Contracts


## Staker contract calls unimplemented native functions leading to failed transactions

### Severity

HIGH

### Description

The Staker contract makes external calls to functions (e.g., native_addValidation, native_increaseStake) via the StakerNative interface, but these functions are not implemented in the provided contract. Since the Staker contract uses address(this) as the StakerNative address, it attempts to call these functions on itself, which do not exist. This results in all such transactions reverting, rendering the contract non-functional.

### Recommendation

Implement the native_* functions as per the StakerNative interface within the contract or ensure that the Staker contract is deployed alongside a contract that implements these functions and correctly references its address.

### Affected Files


- `builtin/gen/staker.sol`, line 51 to line 63

 ```solidity
    function addValidation(
        address validator,
        uint32 period
    ) external payable checkStake(msg.value) stakerNotPaused {
        effectiveVET += msg.value;
        StakerNative(address(this)).native_addValidation(
            validator,
            msg.sender,
            period,
            msg.value
        );
        emit ValidationQueued(validator, msg.sender, period, msg.value);
    }
 ```


## Missing access control in increaseStake allows unauthorized endorser changes

### Severity

HIGH

### Description

The increaseStake function passes msg.sender as the endorser when calling native_increaseStake. If the native function allows updating the endorser, any user can call this function for a validator and change the endorser, leading to unauthorized privilege escalation. The current code does not validate if the caller is the original endorser.

### Recommendation

Ensure that the native_increaseStake function validates the caller is the current endorser of the validator. Add a check in the Staker contract before invoking the native function to confirm the caller's authorization.

### Affected Files


- `builtin/gen/staker.sol`, line 68 to line 78

 ```solidity
    function increaseStake(
        address validator
    ) external payable checkStake(msg.value) stakerNotPaused {
        effectiveVET += msg.value;
        StakerNative(address(this)).native_increaseStake(
            validator,
            msg.sender,
            msg.value
        );
        emit StakeIncreased(validator, msg.value);
    }
 ```


## Missing validation on delegation multiplier allows zero multiplier

### Severity

MEDIUM

### Description

The addDelegation function accepts a uint8 multiplier without enforcing a minimum value. A multiplier of zero would result in a delegation with no weight, which is likely unintended. The comment specifies that 100 represents a 1x multiplier, but the code does not check if the multiplier is at least 100.

### Recommendation

Add a require statement to ensure the multiplier is at least 100 (e.g., `require(multiplier >= 100, "Multiplier too low")`).

### Affected Files


- `builtin/gen/staker.sol`, line 138 to line 157

 ```solidity
    function addDelegation(
        address validator,
        uint8 multiplier // (% of msg.value) 100 for x1, 200 for x2, etc. This enforces a maximum of 2.55x multiplier
    )
        external
        payable
        onlyDelegatorContract
        checkStake(msg.value)
        delegatorNotPaused
        returns (uint256 delegationID)
    {
        effectiveVET += msg.value;
        delegationID = StakerNative(address(this)).native_addDelegation(
            validator,
            msg.value,
            multiplier
        );
        emit DelegationAdded(validator, delegationID, msg.value, multiplier);
        return delegationID;
    }
 ```


## Initialization deadlock prevents adding the first approver or voting contract

### Severity

HIGH

### Description

The contract's `addApprover`, `revokeApprover`, `attachVotingContract`, and `detachVotingContract` functions can only be called by the contract itself. However, when the contract is deployed, there are no approvers (approverCount is 0) and no attached voting contracts. This creates a deadlock because proposing a new approver requires the proposer to be an existing approver or a voting contract, which cannot exist without first being added via these functions. As a result, the contract is uninitialized and cannot function as intended.

### Recommendation

Introduce an initialization function that allows a one-time setup of initial approvers or voting contracts during deployment, bypassing the usual proposal process. This function should be callable by a deployer or a specific address to bootstrap the system.

### Affected Files


- `builtin/gen/executor.sol`, line 126 to line 134

 ```solidity
    function addApprover(address _approver, bytes32 _identity) public {
        onlyThis();
        require(_approver != 0, "builtin: invalid approver");
        require(_identity != 0, "builtin: invalid identity");
        require(approvers[_approver].identity == 0, "builtin: approver exists");
        require(approverCount < 255, "builtin: too many approvers");

        approvers[_approver] = approver(_identity, true);
        approverCount++;
 ```


## Proposal ID collision due to insufficient uniqueness in ID generation

### Severity

MEDIUM

### Description

The `propose` function generates a proposal ID using `keccak256(abi.encodePacked(uint64(now), msg.sender))`. Since `uint64(now)` truncates the current timestamp to 64 bits, if the same proposer (`msg.sender`) submits multiple proposals within the same second (or same truncated timestamp value), the generated proposal ID will collide. This can lead to failed proposal submissions and potential denial of service for legitimate proposals.

### Recommendation

Use a nonce or a counter specific to each proposer to ensure uniqueness. Alternatively, include a more precise timestamp (e.g., `block.timestamp` without truncation) and other unique elements like `block.number` to reduce collision probability.

### Affected Files


- `builtin/gen/executor.sol`, line 58 to line 60

 ```solidity
        bytes32 proposalID = keccak256(
            abi.encodePacked(uint64(now), msg.sender)
        );
 ```


## Missing validation for period in addValidation allows invalid staking periods

### Severity

MEDIUM

### Description

The `addValidation` function does not validate that the `period` parameter is greater than zero. If the underlying `native_addValidation` function allows a period of zero, this could lead to validations with invalid staking periods, potentially causing unexpected behavior in the staking logic or allowing attackers to create invalid validations.

### Recommendation

Add a require statement to ensure `period` is greater than zero. For example: `require(period > 0, "Invalid period");`.

### Affected Files


- `builtin/gen/staker.sol`, line 51 to line 63

 ```solidity
    function addValidation(
        address validator,
        uint32 period
    ) external payable checkStake(msg.value) stakerNotPaused {
        effectiveVET += msg.value;
        StakerNative(address(this)).native_addValidation(
            validator,
            msg.sender,
            period,
            msg.value
        );
        emit ValidationQueued(validator, msg.sender, period, msg.value);
    }
 ```


## Incorrect totalSupply declaration leading to potential implementation errors

### Severity

HIGH

### Description

The contract declares `totalSupply` as a function, while the comments indicate it should be a public variable. This inconsistency can mislead developers into improper ERC20 implementations. Child contracts relying on the comment's guidance may fail to correctly override `totalSupply`, causing ERC20 non-compliance and unexpected behavior when interacting with other contracts that expect standard ERC20 methods.

### Recommendation

Replace the function declaration with a public variable. Uncomment `uint256 public totalSupply;` and remove the function declaration to ensure the automatic getter is used, aligning code with the intended design.

### Affected Files


- `builtin/gen/token.sol`, line 16 to line 16

 ```solidity
    function totalSupply() public constant returns (uint256 supply);
 ```

