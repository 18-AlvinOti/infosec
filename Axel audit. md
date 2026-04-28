
---
created: 2025-12-23 04:11:06
scanned: 2026-01-06 17:44:30

---

# Axel 1st scan


## Incorrect use of onlySelf modifier prevents command execution

### Severity

HIGH

### Description

The functions `deployToken`, `mintToken`, `burnToken`, `transferOwnership`, and `transferOperatorship` in `AxelarGatewayMultisig` are decorated with the `onlySelf` modifier. However, when these functions are called via the `execute` function using `address(this).call()`, the `msg.sender` is the original transaction sender, not the contract itself. This causes the `onlySelf` check to fail, preventing any commands from being successfully executed. This renders the gateway non-functional as all critical operations are blocked.

### Recommendation

Remove the `onlySelf` modifier from the affected functions. The `execute` function already performs necessary access control via signature validation, making the modifier redundant. Ensure proper access control is maintained through the existing signature checks.

### Affected Files


- `AxelarGatewayProxyMultisig.sol`, line 1713 to line 1731

 ```solidity
    function deployToken(bytes calldata params) external onlySelf {
        (
            string memory name,
            string memory symbol,
            uint8 decimals,
            uint256 cap
        ) = abi.decode(params, (string, string, uint8, uint256));

        _deployToken(name, symbol, decimals, cap);
    }

    function mintToken(bytes calldata params) external onlySelf {
        (string memory symbol, address account, uint256 amount) = abi.decode(
            params,
            (string, address, uint256)
        );

        _mintToken(symbol, account, amount);
    }
 ```


## Incorrect epoch retention check allows old signers to remain valid longer than intended

### Severity

HIGH

### Description

The `_areValidRecentOwners` and `_areValidRecentOperators` functions use `OLD_KEY_RETENTION + 1` when calculating the number of recent epochs to check. This results in checking one more epoch than intended (17 instead of 16). Attackers could exploit this by using old, possibly compromised keys from epochs beyond the intended retention period to execute commands, leading to unauthorized actions like token minting or ownership transfers.

### Recommendation

Replace `OLD_KEY_RETENTION + 1` with `OLD_KEY_RETENTION` to correctly check the intended number of retained epochs.

### Affected Files


- `AxelarGatewayMultisig.sol`, line 1366 to line 1366

 ```solidity
        uint256 recentEpochs = OLD_KEY_RETENTION + uint256(1);
 ```


## Missing initialization guard allows repeated setup calls

### Severity

HIGH

### Description

The `setup` function lacks an initialization guard, allowing it to be called multiple times via upgrades. Malicious or compromised admins can repeatedly call `upgrade` with a new implementation that resets admin/owner/operator roles, enabling complete control takeover and bypassing intended multisig protections.

### Recommendation

Add a boolean flag (e.g., `initialized`) checked and set on first execution to prevent reinitialization of the contract's critical parameters.

### Affected Files


- `AxelarGatewayMultisig.sol`, line 1740 to line 1741

 ```solidity
    function setup(bytes calldata params) external override {
        // Prevent setup from being called on a non-proxy (the implementation).
 ```


- `AxelarGatewayProxyMultisig.sol`, line 1785 to line 1825

 ```solidity
    function setup(bytes calldata params) external override {
        // Prevent setup from being called on a non-proxy (the implementation).
        require(implementation() != address(0), "NOT_PROXY");

        (
            address[] memory adminAddresses,
            uint256 adminThreshold,
            address[] memory ownerAddresses,
            uint256 ownerThreshold,
            address[] memory operatorAddresses,
            uint256 operatorThreshold
        ) = abi.decode(
                params,
                (address[], uint256, address[], uint256, address[], uint256)
            );

        uint256 adminEpoch = _adminEpoch() + uint256(1);
        _setAdminEpoch(adminEpoch);
        _setAdmins(adminEpoch, adminAddresses, adminThreshold);

        uint256 ownerEpoch = _ownerEpoch() + uint256(1);
        _setOwnerEpoch(ownerEpoch);
        _setOwners(ownerEpoch, ownerAddresses, ownerThreshold);

        uint256 operatorEpoch = _operatorEpoch() + uint256(1);
        _setOperatorEpoch(operatorEpoch);
        _setOperators(operatorEpoch, operatorAddresses, operatorThreshold);

        emit OwnershipTransferred(
            new address[](uint256(0)),
            uint256(0),
            ownerAddresses,
            ownerThreshold
        );
        emit OperatorshipTransferred(
            new address[](uint256(0)),
            uint256(0),
            operatorAddresses,
            operatorThreshold
        );
    }
 ```


## Lack of commandId uniqueness check allows duplicate commandIds in a single execute call, leading to skipped commands.

### Severity

MEDIUM

### Description

The execute function processes an array of commandIds without checking for duplicates within the same input. If multiple commands share the same commandId, only the first occurrence is executed, and subsequent ones are skipped. This can result in commands not being processed as intended, potentially disrupting contract operations.

### Recommendation

Add a check to ensure all commandIds in the input array are unique. This can be done by tracking processed commandIds within the loop and reverting if duplicates are found in the same batch.

### Affected Files


- `AxelarGatewayMultisig.sol`, line 1828 to line 1832

 ```solidity
        for (uint256 i; i < commandsLength; i++) {
            bytes32 commandId = commandIds[i];

            if (isCommandExecuted(commandId))
                continue; /* Ignore if duplicate commandId received */
 ```


## Previous owners can execute privileged commands after ownership transfer due to retained epoch validity.

### Severity

HIGH

### Description

The contract allows owners from recent epochs (up to 16 epochs old) to execute commands like deployToken, mintToken, and burnToken. If ownership is transferred but old owner keys are compromised, attackers can misuse these keys to execute sensitive operations even after they should be invalidated.

### Recommendation

Reduce the retention period for old owner epochs or implement a mechanism to explicitly invalidate previous epochs upon ownership transfer to prevent outdated owners from retaining privileges.

### Affected Files


- `AxelarGatewayMultisig.sol`, line 1362 to line 1376

 ```solidity
    function _areValidRecentOwners(
        address[] memory accounts
    ) internal view returns (bool) {
        uint256 ownerEpoch = _ownerEpoch();
        uint256 recentEpochs = OLD_KEY_RETENTION + uint256(1);
        uint256 lowerBoundOwnerEpoch = ownerEpoch > recentEpochs
            ? ownerEpoch - recentEpochs
            : uint256(0);

        while (ownerEpoch > lowerBoundOwnerEpoch) {
            if (_areValidOwnersInEpoch(ownerEpoch--, accounts)) return true;
        }

        return false;
    }
 ```

