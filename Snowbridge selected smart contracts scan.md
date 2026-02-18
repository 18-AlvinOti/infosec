
---
created: 2025-09-27 04:28:22
scanned: 2025-09-27 04:34:14

---

# Snowbridge selected smart contracts scan


## Nonce incremented before message processing in V1 submit function leading to failed messages blocking subsequent messages

### Severity

HIGH

### Description

In the `submitV1` function, the channel's inbound nonce is incremented before processing the message. If the message processing fails (caught in the try/catch blocks), the nonce is still incremented. This means that even if a message fails to process, the next message must have the next nonce, making it impossible to retry the failed message. This can lead to a situation where valid messages are rejected because their nonce is not the expected one, effectively blocking the message queue.

### Recommendation

Increment the nonce only after the message has been successfully processed. Move the `channel.inboundNonce++` after the message processing steps, ensuring that the nonce is only updated if the message is successful. Alternatively, use a temporary variable to track the expected nonce and update the state only upon successful processing.

### Affected Files


- `contracts/src/Gateway.sol`, line 157 to line 165

 ```solidity
        // Ensure this message is not being replayed
        if (message.nonce != channel.inboundNonce + 1) {
            revert IGatewayBase.InvalidNonce();
        }

        // Increment nonce for origin.
        // This also prevents the re-entrancy case in which a malicious party tries to re-enter by
        // calling `submitInbound` again with the same (message, leafProof, headerProof) arguments.
        channel.inboundNonce++;
 ```


## Potential Reinitialization of Contract via `initialize` Function

### Severity

MEDIUM

### Description

The `initialize` function is marked as `external` and `virtual`, allowing anyone to call it if the contract is not used behind a proxy. If the contract is deployed without a proxy (e.g., directly), an attacker could call this function to initialize or reinitialize the contract's state, potentially setting critical parameters maliciously. This could lead to a takeover or misconfiguration of the contract.

### Recommendation

Add a check to ensure that the function can only be called once, or only by a specific address (e.g., the proxy's admin). For example, include a check that the contract has not been initialized yet, or use a constructor to initialize the proxy's implementation. Alternatively, use the `initializer` modifier from OpenZeppelin to prevent reinitialization.

### Affected Files


- `contracts/src/Gateway.sol`, line 697 to line 699

 ```solidity
    function initialize(bytes calldata data) external virtual {
        Initializer.initialize(data);
    }
 ```


## Incorrect Gas Refund Calculation in V1 Submit Function

### Severity

MEDIUM

### Description

The `submitV1` function calculates the gas used as the sum of `v1_transactionBaseGas()` and the difference between `startGas` and `gasleft()`. However, `v1_transactionBaseGas()` is a static estimate of the base gas cost, which may not accurately reflect the actual gas used before `startGas` was measured. This can lead to overestimation of the gas used, resulting in overpayment to relayers. Conversely, if the actual base gas is higher than the estimate, relayers may be undercompensated, leading to economic inefficiencies.

### Recommendation

Re-evaluate the gas estimation method. Consider measuring the actual gas used by including all code within the metered section (i.e., moving the `startGas` measurement to the very beginning of the function). Alternatively, adjust the `v1_transactionBaseGas` calculation to more accurately reflect the fixed costs incurred before the metered code.

### Affected Files


- `contracts/src/Gateway.sol`, line 257 to line 258

 ```solidity
        uint256 gasUsed = v1_transactionBaseGas() + (startGas - gasleft());
        uint256 refund = gasUsed * Math.min(tx.gasprice, message.maxFeePerGas);
 ```


## Multiple runtime features can be enabled simultaneously, causing conflicting re-exports

### Severity

HIGH

### Description

The code uses mutually exclusive `#[cfg(feature)]` attributes to re-export different runtime implementations (polkadot/westend/paseo). However, Rust features are additive by default, meaning multiple features could be enabled simultaneously during compilation. This would result in multiple conflicting definitions of `RuntimeCall` and other re-exported items being exposed under the same name, leading to compilation failures or undefined runtime behavior if the contract is built with multiple features enabled. This is critical for blockchain contracts as it could completely prevent successful compilation/deployment, or cause runtime type mismatches if somehow deployed.

### Recommendation

Enforce mutual exclusivity of the runtime selection features. This can be done by:
1. Using a single non-boolean feature with mutually exclusive values in Cargo.toml (e.g. `runtime = ["polkadot", "westend", "paseo"]`)
2. Adding compile-time checks using `compile_error!` if multiple features are enabled
3. Using `cfg!(all(...))` guards to prevent ambiguous re-exports when multiple features are enabled

### Affected Files


- `control/preimage/src/relay_runtime.rs`, line 1 to line 14

 ```solidity
#[cfg(feature = "polkadot")]
pub use polkadot_runtime::runtime_types::polkadot_runtime::RuntimeCall;
#[cfg(feature = "polkadot")]
pub use polkadot_runtime::*;

#[cfg(feature = "westend")]
pub use westend_runtime::runtime_types::westend_runtime::RuntimeCall;
#[cfg(feature = "westend")]
pub use westend_runtime::*;

#[cfg(feature = "paseo")]
pub use paseo_runtime::runtime_types::paseo_runtime::RuntimeCall;
#[cfg(feature = "paseo")]
pub use paseo_runtime::*;
 ```


## Incorrect `valid_from` block calculation using fixed LAUNCH_BLOCK

### Severity

HIGH

### Description

The code calculates the `valid_from` block for treasury spends by adding a delay to a fixed `LAUNCH_BLOCK`. If the actual proposal is enacted after `LAUNCH_BLOCK`, the `valid_from` will be set to a block in the past, making the funds immediately spendable. This defeats the purpose of the delay, allowing recipients to claim funds earlier than intended.

### Recommendation

Calculate `valid_from` based on the current block number at the time the proposal is enacted, rather than a fixed `LAUNCH_BLOCK`. This can be done by passing the current block number as a parameter or fetching it from the runtime's block number storage.

### Affected Files


- `control/preimage/src/treasury_commands.rs`

 ```solidity

 ```


## Nonce incremented before message processing leading to stuck channels on failure

### Severity

HIGH

### Description

In the `submitV1` function, the channel's inbound nonce is incremented before processing the message. If message processing fails (e.g., due to a revert in the handler), the nonce is still incremented. This prevents the same message from being retried, as subsequent submissions must use the next nonce. This can lead to stuck channels where valid messages cannot be reprocessed after transient failures, potentially halting operations dependent on the channel's messages.

### Recommendation

Increment the nonce only after successful message processing. Move the nonce increment to after the message has been successfully handled, or use a temporary variable to track the expected nonce during processing without persisting the increment until success is confirmed.

### Affected Files


- `contracts/src/Gateway.sol`, line 157 to line 165

 ```solidity
        // Ensure this message is not being replayed
        if (message.nonce != channel.inboundNonce + 1) {
            revert IGatewayBase.InvalidNonce();
        }

        // Increment nonce for origin.
        // This also prevents the re-entrancy case in which a malicious party tries to re-enter by
        // calling `submitInbound` again with the same (message, leafProof, headerProof) arguments.
        channel.inboundNonce++;
 ```


## Unrestricted agent creation allowing privilege escalation

### Severity

HIGH

### Description

The `v2_createAgent` function is externally accessible without access control, allowing any user to create an agent with an arbitrary ID. This can lead to unauthorized agents being created, potentially overwriting existing ones or enabling malicious actors to control critical functions reserved for specific agents, compromising the system's security.

### Recommendation

Restrict access to the `v2_createAgent` function using an access control mechanism, ensuring only authorized entities (e.g., the gateway itself via verified messages) can create agents. Implement checks within `CallsV2.createAgent` to validate caller permissions or restrict invocation to specific trusted sources.

### Affected Files


- `contracts/src/Gateway.sol`, line 538 to line 540

 ```solidity
    function v2_createAgent(bytes32 id) external {
        CallsV2.createAgent(id);
    }
 ```


## Multiple runtime features can be enabled simultaneously, causing conflicting imports and compilation errors.

### Severity

HIGH

### Description

The code conditionally imports different runtime modules (polkadot, westend, paseo) based on feature flags. However, these features are not mutually exclusive. If multiple features are enabled during compilation, the same type names (e.g., `RuntimeCall`) and wildcard imports (`*`) from different runtimes will conflict, leading to compilation errors or undefined runtime behavior. This could prevent the contract from compiling correctly or cause unintended runtime logic if multiple runtimes are inadvertently included.

### Recommendation

Enforce mutual exclusivity of the runtime features. In `Cargo.toml`, mark the features as conflicting. Alternatively, modify the `cfg` attributes to check for exactly one active feature (e.g., `#[cfg(all(feature = "polkadot", not(any(feature = "westend", feature = "paseo"))))]`). This ensures only one runtime is imported, preventing identifier collisions.

### Affected Files


- `control/preimage/src/relay_runtime.rs`, line 1 to line 14

 ```solidity
#[cfg(feature = "polkadot")]
pub use polkadot_runtime::runtime_types::polkadot_runtime::RuntimeCall;
#[cfg(feature = "polkadot")]
pub use polkadot_runtime::*;

#[cfg(feature = "westend")]
pub use westend_runtime::runtime_types::westend_runtime::RuntimeCall;
#[cfg(feature = "westend")]
pub use westend_runtime::*;

#[cfg(feature = "paseo")]
pub use paseo_runtime::runtime_types::paseo_runtime::RuntimeCall;
#[cfg(feature = "paseo")]
pub use paseo_runtime::*;
 ```


## Incorrect construction of XCM Junctions leading to compilation errors

### Severity

HIGH

### Description

The code incorrectly uses array syntax to construct XCM v4 Junctions (X1, X2 variants). In Rust, enum variants like `Junctions::X2` expect individual elements as separate arguments, not an array. This results in compilation errors, preventing the contract from functioning entirely. For example, `Junctions::X2([Junction::PalletInstance(50), ...])` is invalid syntax as `X2` takes two separate `Junction` parameters, not an array.

### Recommendation

Replace array syntax with proper enum variant initialization. For `X1`, pass a single `Junction`; for `X2`, pass two separate `Junction` parameters. Example correction: `Junctions::X2(Junction::PalletInstance(50), Junction::GeneralIndex(1337))`.

### Affected Files


- `control/preimage/src/treasury_commands.rs`

 ```solidity

 ```

