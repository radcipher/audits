## Findings Summary

| ID                                                                                                                    | Description | Severity |
|-----------------------------------------------------------------------------------------------------------------------| - |----------|
| [H-01](#[l-03-not-emitting-events-for-important-state-changes](https://codehawks.cyfrin.io/c/2024-09-stakelink/s/97)) |Inconsistent Vesting Release Rate Calculation in `SecondSwap_StepVesting::transferVesting` Function| High     |

# [H-01] Inconsistent Vesting Release Rate Calculation in `SecondSwap_StepVesting::transferVesting` Function

## Vulnerability Details
The SecondSwap_StepVesting::transferVesting function incorrectly recalculates the releaseRate when a grantor transfers their vesting to a beneficiary. As a result, the grantor receive more tokens than they are entitled to, leading to over-distribution of tokens. This issue arises because the new releaseRate is calculated based on the total remaining amount divided by the total number of steps, without properly accounting for the steps already claimed by the grantor. This flaw can lead to a situation where the total amount claimed by the grantor exceeds the amount they should be able to claim.
```solidity
function transferVesting(address _grantor, address _beneficiary, uint256 _amount) external {
        require(
            msg.sender == tokenIssuer || msg.sender == manager || msg.sender == vestingDeployer,
            "SS_StepVesting: unauthorized"
        );
        require(_beneficiary != address(0), "SS_StepVesting: beneficiary is zero");
        require(_amount > 0, "SS_StepVesting: amount is zero");
        Vesting storage grantorVesting = _vestings[_grantor];
        require(
            grantorVesting.totalAmount - grantorVesting.amountClaimed >= _amount,
            "SS_StepVesting: insufficient balance"
        ); // 3.8. Claimed amount not checked in transferVesting function

        grantorVesting.totalAmount -= _amount;
@>      grantorVesting.releaseRate = grantorVesting.totalAmount / numOfSteps;

        _createVesting(_beneficiary, _amount, grantorVesting.stepsClaimed, true);

        emit VestingTransferred(_grantor, _beneficiary, _amount);
    }
```

## PoC
The SecondSwap_StepVesting::transferVesting function is designed to allow a grantor to transfer their vesting to another beneficiary. When this function is called, the releaseRate for the grantor is recalculated as the total remaining amount of tokens divided by the total number of steps (which is a fixed value, numOfSteps). However, this calculation fails to take into account the steps that the grantor has already claimed.

Scenario: Potential Over-Claiming of Tokens

Consider the following example:

Initial Vesting Setup:

Total Vesting Amount: 1000 tokens
Number of Vesting Steps: 10 steps
Step Duration: 1 month
Release Rate: 100 tokens per step

Alice's Vesting State Before Transfer:

Total Amount: 1000 tokens
Steps Claimed: 3 steps
Amount Claimed: 300 tokens (3 steps * 100 tokens per step)
Remaining Amount: 700 tokens (1000 total - 300 claimed)

Alice Transfers 400 Tokens to Bob:

Remaining Total Amount for Alice: 1000 - 400 = 600 tokens
New Release Rate: 600 / 10 = 60 tokens per step (incorrect, since it should be recalculated based on remaining steps and remaining amount)

Bob's Vesting State:

Total Amount: 400 tokens
Release Rate: 400 / 10 = 40 tokens per step
Steps Claimed: 3 steps (inherited from Alice)

Claiming Process for Alice:

Alice has already claimed 300 tokens.
Alice's remaining claimable amount for the next 7 steps: 7 steps * 60 tokens per step = 420 tokens.
Total Amount Claimed by Alice: 300 (already claimed) + 420 (remaining steps) = 720 tokens.

The above scenario is not possible due to the below code if he is claiming after endTime

https://github.com/code-423n4/2024-12-secondswap/blob/main/contracts/SecondSwap_StepVesting.sol#L178-L183

but if he just claims in the last step for just before endTime Below scenario is possible where he would be able to claim more tokens.

Consider even if alice calls SecondSwap_StepVesting::claim when 10th step is going on he would receive:

Because function SecondSwap_StepVesting::claim calls SecondSwap_StepVesting::claimable where this if condition is not valid vesting.stepsClaimed + claimableSteps >= numOfSteps and claimableAmount is given by
```solidity
claimableAmount = vesting.releaseRate * claimableSteps;
```
Alice's remaining claimable amount for the next 6 steps: 6 steps * 60 tokens per step = 360 tokens
Total Amount Claimed by Alice: 300 (already claimed) + 360 (remaining steps) = 660 tokens.


## Impact
Over-Claiming of Tokens: When a user sells their vested tokens using the transferVesting function, they can end up receiving more tokens than they should. This occurs due to the incorrect recalculation of the releaseRate, allowing the user to over-claim tokens from their remaining vesting steps.

## Recommended mitigation steps
```solidity

function transferVesting(address _grantor, address _beneficiary, uint256 _amount) external {
        require(
            msg.sender == tokenIssuer || msg.sender == manager || msg.sender == vestingDeployer,
            "SS_StepVesting: unauthorized"
        );
        require(_beneficiary != address(0), "SS_StepVesting: beneficiary is zero");
        require(_amount > 0, "SS_StepVesting: amount is zero");
        Vesting storage grantorVesting = _vestings[_grantor];
        require(
            grantorVesting.totalAmount - grantorVesting.amountClaimed >= _amount,
            "SS_StepVesting: insufficient balance"
        ); // 3.8. Claimed amount not checked in transferVesting function

        grantorVesting.totalAmount -= _amount;
-       grantorVesting.releaseRate = grantorVesting.totalAmount / numOfSteps;
       
+       uint256 remainingSteps = numOfSteps - grantorVesting.stepsClaimed;
+       grantorVesting.releaseRate = (grantorVesting.totalAmount - grantorVesting.amountClaimed) / remainingSteps;

        _createVesting(_beneficiary, _amount, grantorVesting.stepsClaimed, true);

        emit VestingTransferred(_grantor, _beneficiary, _amount);
    }
```