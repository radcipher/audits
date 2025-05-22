## Findings Summary

| ID                                                                                                                    | Description | Severity |
|-----------------------------------------------------------------------------------------------------------------------| - |----------|
| [H-01](#[l-03-not-emitting-events-for-important-state-changes](https://codehawks.cyfrin.io/c/2024-09-stakelink/s/97)) | Potential Fund Locking and Integer Underflow in `MergeTgt::withdrawRemainingTitn` Due to Unrestricted Growth of `totalTitnClaimable` | High     |

# [H-01] Potential Fund Locking and Integer Underflow in `MergeTgt::withdrawRemainingTitn` Due to Unrestricted Growth of `totalTitnClaimable`

## Vulnerability Details
The MergeTgt::withdrawRemainingTitn function contains a potential miscalculation due to incorrect assumptions about totalTitnClaimable. The function computes unclaimedTitn as follows:
```solidity
uint256 unclaimedTitn = remainingTitnAfter1Year - initialTotalClaimable;
```
Since initialTotalClaimable is assigned from `totalTitnClaimable`, but `totalTitnClaimable` can exceed the actual TITN balance, this can lead to unexpected reverts due to Solidity's built-in underflow protection in version ^0.8.9.

## Impact
Unexpected reverts: If `totalTitnClaimable` exceeds remainingTitnAfter1Year, the subtraction underflows and causes a revert.
Potential denial of service: If `totalTitnClaimable` is inflated beyond actual TITN reserves, the contract may become unusable for withdrawals(Unless owner deposits another TITN_ARB amount of TITN tokens. But the sponser stated in discord that deposit will be done just once by the owner).

## PoC
1. MergeTgt::withdrawRemainingTitn Fails Due to Underflow
```solidity
function withdrawRemainingTitn() external nonReentrant {
        require(launchTime > 0, "Launch time not set");

        if (block.timestamp - launchTime < 360 days) {
            revert TooEarlyToClaimRemainingTitn();
        }

        uint256 currentRemainingTitn = titn.balanceOf(address(this));

        if (remainingTitnAfter1Year == 0) {
            // Initialize remainingTitnAfter1Year to the current balance of TITN
            remainingTitnAfter1Year = currentRemainingTitn;

            // Capture the total claimable TITN at the time of the first claim
            initialTotalClaimable = totalTitnClaimable;
        }

        uint256 claimableTitn = claimableTitnPerUser[msg.sender];
        require(claimableTitn > 0, "No claimable TITN");

        // Calculate proportional remaining TITN for the user
@>        uint256 unclaimedTitn = remainingTitnAfter1Year - initialTotalClaimable;
        uint256 userProportionalShare = (claimableTitn * unclaimedTitn) / initialTotalClaimable;

        uint256 titnOut = claimableTitn + userProportionalShare;
....
```
`initialTotalClaimable` = `totalTitnClaimable`, but `totalTitnClaimable` can be greater than remainingTitnAfter1Year.
Since Solidity ^0.8.9 enforces safe math, this subtraction will revert instead of wrapping.
2. MergeTgt::onTokenTransfer Allows More TGT Than Expected
```solidity
function onTokenTransfer(address from, uint256 amount, bytes calldata extraData) external nonReentrant {
        if (msg.sender != address(tgt)) {
            revert InvalidTokenReceived();
        }
        if (lockedStatus == LockedStatus.Locked || launchTime == 0) {
            revert MergeLocked();
        }
        if (amount == 0) {
            revert ZeroAmount();
        }
        if (block.timestamp - launchTime > 360 days) {
            revert MergeEnded();
        }

        // tgt in, titn out

@>        uint256 titnOut = quoteTitn(amount);
@>        claimableTitnPerUser[from] += titnOut;
@>        totalTitnClaimable += titnOut;

        emit ClaimableTitnUpdated(from, titnOut);
    }
```
There is no upper limit to `totalTitnClaimable`.The expected TGT amount is defined by `TGT_TO_EXCHANGE`, but the contract does not enforce this limit, allowing users to deposit more TGT than intended, which leads to an inflated totalTitnClaimable.
Users can deposit excessive TGT, causing `totalTitnClaimable` to grow beyond the available TITN balance.
This means at withdrawal, initialTotalClaimable > remainingTitnAfter1Year, triggering underflow and revert.

## Recommendations
```solidity
function onTokenTransfer(address from, uint256 amount, bytes calldata extraData) external nonReentrant {
    if (msg.sender != address(tgt)) {
        revert InvalidTokenReceived();
    }
    if (lockedStatus == LockedStatus.Locked || launchTime == 0) {
        revert MergeLocked();
    }
    if (amount == 0) {
        revert ZeroAmount();
    }
    if (block.timestamp - launchTime > 360 days) {
        revert MergeEnded();
    }

    // tgt in, titn out

    uint256 titnOut = quoteTitn(amount);
    +      require(totalTitnClaimable + titnOut <= titn.balanceOf(address(this)), "Exceeds TITN reserves");
    claimableTitnPerUser[from] += titnOut;
    totalTitnClaimable += titnOut;

    emit ClaimableTitnUpdated(from, titnOut);
}
```
Ensures totalTitnClaimable does not exceed actual TITN balance.
Prevents inflation of totalTitnClaimable beyond available funds which is 173.7 Million TITN.
