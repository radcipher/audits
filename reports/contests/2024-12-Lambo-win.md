## Findings Summary

| ID                                                                                                                    | Description | Severity |
|-----------------------------------------------------------------------------------------------------------------------| - |----------|
| [H-01](#[l-03-not-emitting-events-for-important-state-changes](https://codehawks.cyfrin.io/c/2024-09-stakelink/s/97)) | Incorrect Usage of `msg.value` in `VirtualToken::cashIn` | High     |
| [M-01](#[l-03-not-emitting-events-for-important-state-changes](https://codehawks.cyfrin.io/c/2024-09-stakelink/s/97)) | Lack of Mechanism to Withdraw Native Ether in `LamboRebalanceOnUniwap`| Medium   |

# [H-01] Incorrect Usage of `msg.value` in `VirtualToken::cashIn`

## Vulnerability Details
In the VirtualToken::cashIn function, the msg.value is used for minting tokens even when underlyingToken is not the native token, which is incorrect. It should use the amount parameter instead.


## PoC
For example, if the contract is interacting with a non-native ERC20 token and a user calls cashIn(100), but sends 0 ETH (msg.value), the contract will still mint 100 tokens to the user, even though the actual asset transfer didn't occur.

This mismatch can cause discrepancies in the amount of tokens minted and the actual value transferred by the user.

## Recommendations
To fix this, the msg.value check should only be used when the underlyingToken is the native token, and the amount parameter should be used to mint tokens for non-native transfers.
```solidity
function cashIn(uint256 amount) external payable onlyWhiteListed {
        if (underlyingToken == LaunchPadUtils.NATIVE_TOKEN) {
            require(msg.value == amount, "Invalid ETH amount");
+           _mint(msg.sender, msg.value);
        } else {
            _transferAssetFromUser(amount);
+           _mint(msg.sender, amount);
        }
-        _mint(msg.sender, msg.value);
        emit CashIn(msg.sender, msg.value);
    }
```

# [M-01] Lack of Mechanism to Withdraw Native Ether in `LamboRebalanceOnUniwap`

## Vulnerability Details
The LamboRebalanceOnUniwap contract has multiple payable functions and a receive() function to accept native Ether (ETH). However, it lacks a mechanism to withdraw the accumulated native Ether, potentially leading to the contract holding Ether indefinitely. This could result in a loss of funds and limited functionality for the contract owner.In LamboRebalanceOnUniwap::extractProfit there is no functionality withdraw native ether.

## PoC
Add the below testcase to RebalanceTest.sol and run it due to receive function contract is able to receive ether but does not have any functionality to witdraw it.
```solidity
function test_loss_of_funds() public {
        
        address sender = address(1); // or any other address you control
        vm.deal(sender, 10 ether);
        vm.prank(sender);
        (bool success, ) = address(lamboRebalance).call{value: 10 ether}("");
        require(success, "Transfer failed");

        
        console.log("balance:",address(lamboRebalance).balance);
        
    }
```

## Recommended mitigation steps
Implement a function to withdraw native Ether from the contract. The withdrawEther function, as shown in the code example, allows the contract owner to transfer the Ether balance to a specified address.

```solidity
function withdrawEther(address payable to) external onlyOwner {
    uint256 balance = address(this).balance;
    require(balance > 0, "No Ether available");
    to.transfer(balance);
}
```