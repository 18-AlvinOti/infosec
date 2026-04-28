
---
created: 2026-01-02 13:01:48
scanned: 2026-01-09 09:28:31

---

# SNowbunny


## Fallback function allows Ether to be sent to the proxy, leading to locked funds

### Severity

HIGH

### Description

The fallback function in the GatewayProxy is marked as `payable`, which allows users to send Ether along with function calls. However, the contract's receive function reverts on direct Ether transfers. If a user sends Ether via a function call (intentionally or accidentally), the Ether will be stored in the proxy contract but cannot be recovered. Since the proxy does not implement any mechanism to handle or withdraw these funds, they become permanently locked, leading to loss of user funds.

### Recommendation

Remove the `payable` modifier from the fallback function to prevent Ether from being sent to the contract via function calls. This ensures that any transaction sending Ether to the proxy (with or without data) will revert, aligning with the contract's intention to reject native currency.

### Affected Files


- `src/GatewayProxy.sol`, line 29 to line 50

 ```solidity
    fallback() external payable {
        address implementation = ERC1967.load();
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(
                gas(),
                implementation,
                0,
                calldatasize(),
                0,
                0
            )
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 {
                revert(0, returndatasize())
            }
            default {
                return(0, returndatasize())
            }
        }
    }
 ```

