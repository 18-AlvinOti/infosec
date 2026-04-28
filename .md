
---
created: 2025-11-06 03:29:38
scanned: 2025-11-06 03:33:48

---

# Scan of vechain thor selected files 2


## Missing access control in sponsor and unsponsor functions allows any user to modify sponsorship for any account.

### Severity

HIGH

### Description

The `sponsor` and `unsponsor` functions in the Prototype contract are publicly accessible without proper access controls. Any user can call these functions with any `_self` address, potentially adding or removing themselves as a sponsor for any account. If the underlying `PrototypeNative` functions (`native_sponsor` and `native_unsponsor`) do not enforce additional access checks, this allows unauthorized users to manipulate sponsorship status, leading to privilege escalation. Attackers could exploit this to become sponsors for accounts they do not control, affecting the account's sponsorship logic and potentially leading to financial impacts depending on how sponsors are utilized.

### Recommendation

Add access control modifiers to `sponsor` and `unsponsor` functions to ensure only authorized entities (e.g., the account itself, its master, or designated authorities) can modify sponsorships. For example, use the `onlySelfOrMaster` modifier similar to other privileged functions.

### Affected Files


- `builtin/gen/prototype.sol`, line 99 to line 111

 ```solidity
    function sponsor(address _self) public {
        require(
            PrototypeNative(this).native_sponsor(_self, msg.sender),
            "builtin: already sponsored"
        );
    }

    function unsponsor(address _self) public {
        require(
            PrototypeNative(this).native_unsponsor(_self, msg.sender),
            "builtin: not sponsored"
        );
    }
 ```


## Incorrect accounting of effectiveVET in delegation functions leading to underflow

### Severity

HIGH

### Description

The `addDelegation` function increases `effectiveVET` by `msg.value`, but the actual stake stored is `(msg.value * multiplier) / 100`. When withdrawing via `withdrawDelegation`, `effectiveVET` is decreased by the stake amount, which can be higher than the original `msg.value` if the multiplier exceeds 100. This discrepancy causes an underflow in `effectiveVET` during withdrawal, leading to transaction failures and incorrect balance tracking, potentially making withdrawals impossible and destabilizing the contract's financial integrity.

### Recommendation

Adjust `effectiveVET` in `addDelegation` to account for the multiplied stake (`msg.value * multiplier / 100`). Update the `checkStake` modifier to validate the multiplied value, ensuring it's a multiple of 1e18. This ensures accurate tracking and prevents underflow during withdrawals.

### Affected Files


- `builtin/gen/staker.sol`, line 138 to line 170

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

    /**
     * @dev exitDelegation signals the intent to exit a delegation position at the end of the staking period.
     * Funds are available once the current staking period ends.
     */
    function signalDelegationExit(
        uint256 delegationID
    ) external onlyDelegatorContract delegatorNotPaused {
        StakerNative(address(this)).native_signalDelegationExit(delegationID);
        emit DelegationSignaledExit(delegationID);
    }

    /**
 ```


## Delegation stake validation bypass allows non-integer VET amounts

### Severity

HIGH

### Description

The `checkStake` modifier validates `msg.value` as a multiple of 1e18 but doesn't consider the multiplier applied in `addDelegation`. This allows the actual stake (`msg.value * multiplier / 100`) to be a non-integer VET amount, violating the contract's stake requirements. This can lead to fractional VET stakes, causing errors in downstream logic and inconsistent state in the StakerNative contract.

### Recommendation

Modify the `checkStake` check in `addDelegation` to validate `(msg.value * multiplier) % 100 == 0` and ensure `(msg.value * multiplier / 100)` is a multiple of 1e18. This enforces valid stake amounts post-multiplier application.

### Affected Files


- `builtin/gen/staker.sol`, line 371 to line 400

 ```solidity
    modifier checkStake(uint256 amount) {
        require(amount > 0, "staker: stake is empty");
        require(amount % 1e18 == 0, "staker: stake is not multiple of 1VET");
        require(
            amount <= 100_000_000_000 * 1e18,
            "staker: stake is above max supply"
        );
        _;
    }

    modifier stakerNotPaused() {
        uint256 switches = StakerNative(address(this))
            .native_getControlSwitches();
        require(
            (switches & STAKER_PAUSED_BIT) == 0,
            "staker: staker is paused"
        );
        _;
    }

    modifier delegatorNotPaused() {
        uint256 switches = StakerNative(address(this))
            .native_getControlSwitches();
        require(
            (switches & STAKER_PAUSED_BIT) == 0,
            "staker: staker is paused"
        );
        require(
            (switches & DELEGATOR_PAUSED_BIT) == 0,
            "staker: delegator is paused"
 ```


## Lack of access control in sponsor and unsponsor functions allows any user to modify sponsorship status

### Severity

HIGH

### Description

The `sponsor` and `unsponsor` functions in the Prototype contract are publicly accessible without any access control checks. This allows any user to call these functions and add or remove themselves as a sponsor for any contract address (`_self`). If the underlying `PrototypeNative` implementation permits these changes without additional restrictions, this could lead to unauthorized sponsors gaining privileges (e.g., resource allocation, fee sponsorship), potentially leading to abuse of contract resources or disruption of intended functionality.

### Recommendation

Implement access control in the `sponsor` and `unsponsor` functions to ensure only authorized entities (e.g., the contract's master or the contract itself) can modify sponsorship status. For example, add the `onlySelfOrMaster(_self)` modifier to these functions to align with other privileged operations.

### Affected Files


- `builtin/gen/prototype.sol`, line 99 to line 111

 ```solidity
    function sponsor(address _self) public {
        require(
            PrototypeNative(this).native_sponsor(_self, msg.sender),
            "builtin: already sponsored"
        );
    }

    function unsponsor(address _self) public {
        require(
            PrototypeNative(this).native_unsponsor(_self, msg.sender),
            "builtin: not sponsored"
        );
    }
 ```


## Incorrect target address for StakerNative function calls leading to failed transactions

### Severity

HIGH

### Description

The Staker contract incorrectly uses `address(this)` as the target for all calls to the StakerNative interface functions. Since the Staker contract does not implement these functions, any transaction invoking functions like `addValidation`, `increaseStake`, etc., will revert, rendering the contract non-functional. This is a critical issue as it prevents the contract from performing its core operations.

### Recommendation

Deploy a separate contract that implements the StakerNative interface and update the Staker contract to reference the correct StakerNative contract address. Replace `address(this)` with the actual StakerNative contract address, possibly via a state variable initialized during construction.

### Affected Files


- `builtin/gen/staker.sol`, line 56 to line 61

 ```solidity
        StakerNative(address(this)).native_addValidation(
            validator,
            msg.sender,
            period,
            msg.value
        );
 ```


## Potential reentrancy in withdraw functions due to external call after state changes

### Severity

MEDIUM

### Description

The `withdrawStake` and `withdrawDelegation` functions perform an external call to `msg.sender` after updating state variables and interacting with the StakerNative contract. If the recipient is a malicious contract, it could reenter the Staker contract, potentially leading to inconsistent state or double-withdrawals if the StakerNative's state isn't properly secured against reentrancy.

### Recommendation

Apply the checks-effects-interactions pattern strictly by performing the external transfer before any state changes or use a reentrancy guard modifier to prevent reentrant calls during the withdrawal process.

### Affected Files


- `builtin/gen/staker.sol`

 ```solidity

 ```


## Missing access control on critical validator management functions

### Severity

HIGH

### Description

Functions like `setBeneficiary`, `decreaseStake`, and `signalExit` rely on the StakerNative contract to validate the caller's permissions. If the StakerNative contract does not properly check that the caller (passed as `msg.sender` from Staker) is authorized (e.g., the endorser for the validator), attackers could arbitrarily modify beneficiary addresses, decrease stakes, or trigger exits for any validator.

### Recommendation

Ensure the StakerNative contract rigorously validates that the provided `endorser` (passed as `msg.sender` from Staker) has the authority to modify the validator's details. The Staker contract should not assume the native functions enforce these checks without verification.

### Affected Files


- `builtin/gen/staker.sol`

 ```solidity

 ```

