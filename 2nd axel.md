
---
created: 2026-01-06 17:52:22
scanned: 2026-01-06 18:19:52

---

# 2nd axel


## Owner cannot burn tokens from an account without allowance

### Severity

HIGH

### Description

The `burnFrom` function in `BurnableMintableCappedERC20` requires the owner to have an allowance from the account to burn tokens. This prevents the owner from burning tokens unless the account has explicitly approved the owner, which contradicts the intended privileged functionality. This renders the owner's burn capability ineffective unless prior approval exists, undermining the contract's design.

### Recommendation

Remove the allowance check in the `burnFrom` function to allow the owner to burn tokens without requiring an allowance. The function should directly burn tokens from the specified account.

### Affected Files


- `contract.sol`, line 834 to line 840

 ```solidity
    function burnFrom(address account, uint256 amount) external onlyOwner {
        uint256 _allowance = allowance[account][msg.sender];
        if (_allowance != type(uint256).max) {
            _approve(account, msg.sender, _allowance - amount);
        }
        _burn(account, amount);
    }
 ```


## Deposit address changes when owner is updated, causing loss of funds

### Severity

HIGH

### Description

The `depositAddress` function uses the current owner's address to compute the deposit address. If the owner changes, existing deposit addresses become invalid, making tokens sent to old addresses irretrievable. This can lead to permanent loss of funds as the new owner cannot burn tokens from addresses generated under the previous owner.

### Recommendation

Store the initial owner's address in an immutable variable during contract deployment and use it for deposit address computation to ensure consistency regardless of ownership changes.

### Affected Files


- `contract.sol`, line 805 to line 825

 ```solidity
    function depositAddress(bytes32 salt) public view returns (address) {
        /* Convert a hash which is bytes32 to an address which is 20-byte long
        according to https://docs.soliditylang.org/en/v0.8.1/control-structures.html?highlight=create2#salted-contract-creations-create2 */
        return
            address(
                uint160(
                    uint256(
                        keccak256(
                            abi.encodePacked(
                                bytes1(0xff),
                                owner,
                                salt,
                                keccak256(
                                    abi.encodePacked(
                                        type(DepositHandler).creationCode
                                    )
                                )
                            )
                        )
                    )
                )
 ```


## Cap check bypass when token cap is set to zero, allowing unlimited minting.

### Severity

HIGH

### Description

If the token is deployed with a cap of zero, the mint` function skips the cap check, allowing the owner to mint unlimited tokens. This violates the intended capped supply if the cap is mistakenly set to zero during deployment.

### Recommendation

Enforce that the cap must be greater than zero in the constructor, or modify the check to handle zero cap as uncapped explicitly without bypassing the totalSupply validation.

### Affected Files


- `contract.sol`, line 748 to line 756

 ```solidity
    function mint(address account, uint256 amount) external onlyOwner {
        uint256 capacity = cap;

        _mint(account, amount);

        if (capacity == 0) return;

        if (totalSupply > capacity) revert CapExceeded();
    }
 ```


## `burnFrom` function restricted to owner only, breaking ERC20 allowance semantics.

### Severity

HIGH

### Description

The `burnFrom` function is callable only by the owner, preventing approved spenders from burning tokens. This violates the ERC20 standard's allowance mechanism and disrupts integration with contracts expecting standard behavior.

### Recommendation

Remove the `onlyOwner` modifier and ensure the function checks the caller's allowance, allowing any approved spender to call `burnFrom`.

### Affected Files


- `contract.sol`, line 834 to line 840

 ```solidity
    function burnFrom(address account, uint256 amount) external onlyOwner {
        uint256 _allowance = allowance[account][msg.sender];
        if (_allowance != type(uint256).max) {
            _approve(account, msg.sender, _allowance - amount);
        }
        _burn(account, amount);
    }
 ```

