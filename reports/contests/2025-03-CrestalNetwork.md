## Findings Summary

| ID                                                                                                                    | Description | Severity |
|-----------------------------------------------------------------------------------------------------------------------| - |----------|
| [H-01]() |Anyone who is approving BlueprintV5 contract to spend ERC20 can get drained because Payment::payWithERC20 | High     |

# [H-01] Anyone who is approving BlueprintV5 contract to spend ERC20 can get drained because Payment::payWithERC20

## Vulnerability Details
payWithERC20 is supposed to be used inside BlueprintV5 contract to handle payment. But this function also can be used to drain anyone who is interact with BlueprintV5 and using it to approve payment token when creating an agent.
```solidity
@>    function payWithERC20(address erc20TokenAddress, uint256 amount, address fromAddress, address toAddress) public {  
        // check from and to address  
        require(fromAddress != toAddress, "Cannot transfer to self address");  
        require(toAddress != address(0), "Invalid to address");  
        require(amount > 0, "Amount must be greater than 0");  
        IERC20 token = IERC20(erc20TokenAddress);  
        token.safeTransferFrom(fromAddress, toAddress, amount);  
    }  
```
## Impact
user/victim who interacted would lose their funds drained by attacker

## Recommendations
Make the Payment::payWithERC20 internal