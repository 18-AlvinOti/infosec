
---
created: 2025-09-17 01:27:35
scanned: 2025-09-17 22:45:36

---

# Snowfork snowbridge selected contracts scan


## Revert in computeNumRequiredSignatures due to log2(0) when signature is first used

### Severity

HIGH

### Description

The `computeNumRequiredSignatures` function uses `Math.log2(signatureUsageCount)` which will revert when `signatureUsageCount` is 0. Since validators' `usageCounters` are initialized to 0, the first use of a validator's signature in `submitInitial` will cause a transaction revert, preventing new validators from being used and disrupting the submission process.

### Recommendation

Adjust the formula to handle `signatureUsageCount = 0` by adding 1 before taking the logarithm. For example, use `Math.log2(signatureUsageCount + 1)` to avoid log2(0).

### Affected Files


- `contracts/src/BeefyClient.sol`, line 519 to line 521

 ```solidity
        numRequiredSignatures +=
            1 +
            (2 * Math.log2(signatureUsageCount, Math.Rounding.Ceil));
 ```


## Missing validation for next authority set length in MMRLeaf

### Severity

HIGH

### Description

When processing an MMRLeaf for the next validator set, the contract does not check that `leaf.nextAuthoritySetLen` is greater than zero. A malicious leaf with `nextAuthoritySetLen = 0` would cause the next validator set to have a length of 0, leading to underflows and invalid operations in future computations (e.g., quorum calculation).

### Recommendation

Add a check to ensure `leaf.nextAuthoritySetLen` is greater than zero. For example, include `require(leaf.nextAuthoritySetLen > 0, "Invalid next authority set length");` in the validation block.

### Affected Files


- `contracts/src/BeefyClient.sol`, line 399 to line 401

 ```solidity
            if (leaf.nextAuthoritySetID != nextValidatorSet.id + 1) {
                revert InvalidMMRLeaf();
            }
 ```


## Use of transient storage for reentrancy guard may lead to failure on networks without EIP-1153 support.

### Severity

HIGH

### Description

The nonreentrant modifier uses transient storage (tstore/tload), which is part of EIP-1153. If the contract is deployed on a network that hasn't implemented this EIP (like Ethereum before Cancun), the tstore and tload opcodes will be invalid, causing the modified functions to revert. This would disable the reentrancy protection, allowing reentrant calls and potential exploits such as reentrancy attacks.

### Recommendation

Replace the transient storage with a traditional reentrancy guard using a regular state variable. For example, use a boolean in storage that is checked and set during function execution.

### Affected Files


- `contracts/src/Gateway.sol`, line 71 to line 86

 ```solidity
    modifier nonreentrant() {
        assembly {
            if tload(0) {
                revert(0, 0)
            }

            // Set the flag to mark the function is currently executing.
            tstore(0, 1)
        }
        _;
        // Unlocks the guard, making the pattern composable.
        // After the function exits, it can be called again, even in the same transaction.
        assembly {
            tstore(0, 0)
        }
    }
 ```


## Nonce marked as used before successful verification, leading to denial of service.

### Severity

HIGH

### Description

In the v2_submit function, the nonce is set as used before verifying the commitment. If the commitment verification fails, the nonce is still marked as used. This means that a valid message with the same nonce cannot be processed later, causing a denial of service. Attackers can exploit this by submitting invalid messages with valid nonces to block legitimate messages.

### Recommendation

Move the nonce check and set operations after the commitment verification. Only mark the nonce as used after the message has been successfully verified.

### Affected Files


- `contracts/src/Gateway.sol`, line 467 to line 478

 ```solidity
        if ($.inboundNonce.get(message.nonce)) {
            revert IGatewayBase.InvalidNonce();
        }

        bytes32 leafHash = keccak256(abi.encode(message));

        $.inboundNonce.set(message.nonce);

        // Produce the commitment (message root) by applying the leaf proof to the message leaf
        bytes32 commitment = MerkleProof.processProof(leafProof, leafHash);

        // Verify that the commitment is included in a parachain header finalized by BEEFY.
 ```


## Revert in computeNumRequiredSignatures due to log2(0) when signatureUsageCount is zero

### Severity

HIGH

### Description

The `computeNumRequiredSignatures` function uses `Math.log2(signatureUsageCount)` which will revert when `signatureUsageCount` is zero. This occurs when a validator's signature is used for the first time, causing `submitInitial` to fail. This prevents relayers from initiating new commitments for validators with zero prior signature usage, severely impacting the contract's functionality.

### Recommendation

Adjust the formula to handle `signatureUsageCount = 0` by using `signatureUsageCount + 1` inside the log2 function. For example, replace `signatureUsageCount` with `signatureUsageCount + 1` to avoid taking the logarithm of zero.

### Affected Files


- `contracts/src/BeefyClient.sol`, line 519 to line 521

 ```solidity
        numRequiredSignatures +=
            1 +
            (2 * Math.log2(signatureUsageCount, Math.Rounding.Ceil));
 ```


## Reentrancy guard uses invalid opcodes, providing no protection

### Severity

HIGH

### Description

The nonReentrant modifier uses TLOAD and TSTORE opcodes, which are part of EIP-1153 and not available on Ethereum mainnet. This causes the reentrancy guard to fail, allowing reentrant calls to functions marked as nonreentrant. Attackers can re-enter the contract during critical operations, leading to potential fund theft or state corruption.

### Recommendation

Replace TLOAD and TSTORE with regular storage variables for the reentrancy guard. For example, use a uint256 storage variable to track the reentrancy status.

### Affected Files


- `contracts/src/Gateway.sol`, line 71 to line 86

 ```solidity
    modifier nonreentrant() {
        assembly {
            if tload(0) {
                revert(0, 0)
            }

            // Set the flag to mark the function is currently executing.
            tstore(0, 1)
        }
        _;
        // Unlocks the guard, making the pattern composable.
        // After the function exits, it can be called again, even in the same transaction.
        assembly {
            tstore(0, 0)
        }
    }
 ```


## Unauthorized agent creation allows anyone to register agents

### Severity

HIGH

### Description

The v2_createAgent function is callable by any user, allowing them to create arbitrary agent IDs. This can lead to unauthorized agents being registered, potentially enabling malicious actors to spoof legitimate agents and execute privileged operations.

### Recommendation

Restrict access to the v2_createAgent function, ensuring only authorized entities (e.g., the gateway itself via verified messages) can create agents. Add an access control modifier such as onlySelf.

### Affected Files


- `contracts/src/Gateway.sol`, line 535 to line 537

 ```solidity
    function v2_createAgent(bytes32 id) external {
        CallsV2.createAgent(id);
    }
 ```


## Initialize function is publicly accessible leading to unauthorized initialization

### Severity

HIGH

### Description

The initialize function can be called by any user, allowing them to initialize the contract's storage. If the contract is used as a proxy implementation, an attacker could initialize it with malicious data, compromising the contract's configuration.

### Recommendation

Override the initialize function in the contract to include an initializer modifier that ensures it can only be called once and by the authorized deployer. For example, use OpenZeppelin's Initializable library and protect the function with an onlyInitializing modifier.

### Affected Files


- `contracts/src/Gateway.sol`, line 694 to line 696

 ```solidity
    function initialize(bytes calldata data) external virtual {
        Initializer.initialize(data);
    }
 ```


## Out-of-order message processing due to non-sequential nonce tracking

### Severity

MEDIUM

### Description

The v2_submit function uses a bitmap to track nonces, allowing messages to be processed out of order. This can lead to replay attacks or state inconsistencies if later messages are processed before earlier ones, violating the expected message sequencing.

### Recommendation

Replace the bitmap-based nonce tracking with a sequential nonce check, ensuring that messages are processed in the correct order. Track the highest processed nonce and require each new nonce to be exactly one greater than the previous.

### Affected Files


- `contracts/src/Gateway.sol`, line 467 to line 473

 ```solidity
        if ($.inboundNonce.get(message.nonce)) {
            revert IGatewayBase.InvalidNonce();
        }

        bytes32 leafHash = keccak256(abi.encode(message));

        $.inboundNonce.set(message.nonce);
 ```

