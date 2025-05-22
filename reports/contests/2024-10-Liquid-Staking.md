## Findings Summary

| ID | Description | Severity |
| - | - | - |
| [L-01](#[l-03-not-emitting-events-for-important-state-changes](https://codehawks.cyfrin.io/c/2024-09-stakelink/s/97)) | Incorrect Value Emitted in `Withdraw` Events | Low |

# [L-01] Wrong value emitted in Withdraw event


## Vulnerability Details
Below is the event in `PriorityPool` contract where `amount` should be equal to the total amount which the user has withdrawn.

event Withdraw(address indexed account, uint256 amount);

Below is the function responsible for withdrawing the staked amount.


```solidity

function withdraw(
        uint256 _amountToWithdraw,
        uint256 _amount,
        uint256 _sharesAmount,
        bytes32[] calldata _merkleProof,
        bool _shouldUnqueue,
        bool _shouldQueueWithdrawal
    ) external {
        if (_amountToWithdraw == 0) revert InvalidAmount();

        uint256 toWithdraw = _amountToWithdraw;
        address account = msg.sender;

        // attempt to unqueue tokens before withdrawing if flag is set
        if (_shouldUnqueue == true) {
            _requireNotPaused();

            if (_merkleProof.length != 0) {
                bytes32 node = keccak256(
                    bytes.concat(keccak256(abi.encode(account, _amount, _sharesAmount)))
                );
                if (!MerkleProofUpgradeable.verify(_merkleProof, merkleRoot, node))
                    revert InvalidProof();
            } else if (accountIndexes[account] < merkleTreeSize) {
                revert InvalidProof();
            }

            uint256 queuedTokens = getQueuedTokens(account, _amount);
            uint256 canUnqueue = queuedTokens <= totalQueued ? queuedTokens : totalQueued;
            uint256 amountToUnqueue = toWithdraw <= canUnqueue ? toWithdraw : canUnqueue;

            if (amountToUnqueue != 0) {
                accountQueuedTokens[account] -= amountToUnqueue;
                totalQueued -= amountToUnqueue;
                toWithdraw -= amountToUnqueue;
                emit UnqueueTokens(account, amountToUnqueue);
            }
        }

        // attempt to withdraw if tokens remain after unqueueing
        if (toWithdraw != 0) {
            IERC20Upgradeable(address(stakingPool)).safeTransferFrom(
                account,
                address(this),
                toWithdraw
            );
            toWithdraw = _withdraw(account, toWithdraw, _shouldQueueWithdrawal);
        }

        token.safeTransfer(account, _amountToWithdraw - toWithdraw);
    }

```
```solidity

function _withdraw(
        address _account,
        uint256 _amount,
        bool _shouldQueueWithdrawal
    ) internal returns (uint256) {
        if (poolStatus == PoolStatus.CLOSED) revert WithdrawalsDisabled();

        uint256 toWithdraw = _amount;

        if (totalQueued != 0) {
            uint256 toWithdrawFromQueue = toWithdraw <= totalQueued ? toWithdraw : totalQueued;

            totalQueued -= toWithdrawFromQueue;
            depositsSinceLastUpdate += toWithdrawFromQueue;
            sharesSinceLastUpdate += stakingPool.getSharesByStake(toWithdrawFromQueue);
            toWithdraw -= toWithdrawFromQueue;
        }

        if (toWithdraw != 0) {
            if (!_shouldQueueWithdrawal) revert InsufficientLiquidity();
            withdrawalPool.queueWithdrawal(_account, toWithdraw);
        }

        emit Withdraw(_account, _amount - toWithdraw);
        return toWithdraw;
    }

```

## Impact
Frontend or other off-chain services may display incorrect values, potentially misleading users.

## Poc

Consider a user with currently 2 tokens in a queue who wants to withdraw a total of 20 tokens. If the user sets `_shouldUnqueue` and `_shouldQueueWithdrawal` to true, the following happens:

1. In the `PriorityPool::withdraw` function:

   toWithdraw is initially set to 20.
   Since \_shouldUnqueue is true, toWithdraw becomes 18 (20 - 2). As it first withdraws from the tokens which it had transferred to queue.

2. The `PriorityPool::withdraw` function then calls `PriorityPool::_withdraw` with \_amount set as 18 which inturn sets `toWithdraw` as 18 in `PriorityPool::_withdraw`:

Assuming now total queued tokens are 5, `_withdraw` function reduces toWithdraw to 13 (18 - 5).
The Withdraw event is emitted with an amount of 5 (18 - 13), even though the total withdrawn amount is actually 7 (2 + 5).Where 2     is the tokens amount withdrawn in `PriorityPool::withdraw` function


## Tools Used
Manual

## Recommendations
Instead of emiiting the event in `PriorityPool::_withdraw` do it in `PriorityPool::withdraw` as shown.
```solidity
               function withdraw(
        uint256 _amountToWithdraw,
        uint256 _amount,
        uint256 _sharesAmount,
        bytes32[] calldata _merkleProof,
        bool _shouldUnqueue,
        bool _shouldQueueWithdrawal
    ) external {
        if (_amountToWithdraw == 0) revert InvalidAmount();

        uint256 toWithdraw = _amountToWithdraw;
        address account = msg.sender;

        // attempt to unqueue tokens before withdrawing if flag is set
        if (_shouldUnqueue == true) {
            _requireNotPaused();

            if (_merkleProof.length != 0) {
                bytes32 node = keccak256(
                    bytes.concat(keccak256(abi.encode(account, _amount, _sharesAmount)))
                );
                if (!MerkleProofUpgradeable.verify(_merkleProof, merkleRoot, node))
                    revert InvalidProof();
            } else if (accountIndexes[account] < merkleTreeSize) {
                revert InvalidProof();
            }

            uint256 queuedTokens = getQueuedTokens(account, _amount);
            uint256 canUnqueue = queuedTokens <= totalQueued ? queuedTokens : totalQueued;
            uint256 amountToUnqueue = toWithdraw <= canUnqueue ? toWithdraw : canUnqueue;

            if (amountToUnqueue != 0) {
                accountQueuedTokens[account] -= amountToUnqueue;
                totalQueued -= amountToUnqueue;
                toWithdraw -= amountToUnqueue;
                emit UnqueueTokens(account, amountToUnqueue);
            }
        }

        // attempt to withdraw if tokens remain after unqueueing
        if (toWithdraw != 0) {
            IERC20Upgradeable(address(stakingPool)).safeTransferFrom(
                account,
                address(this),
                toWithdraw
            );
            toWithdraw = _withdraw(account, toWithdraw, _shouldQueueWithdrawal);
        }

        token.safeTransfer(account, _amountToWithdraw - toWithdraw);
+      emit Withdraw(_account, _amountToWithdraw - toWithdraw);
    }

    function _withdraw(
        address _account,
        uint256 _amount,
        bool _shouldQueueWithdrawal
    ) internal returns (uint256) {
        if (poolStatus == PoolStatus.CLOSED) revert WithdrawalsDisabled();

        uint256 toWithdraw = _amount;

        if (totalQueued != 0) {
            uint256 toWithdrawFromQueue = toWithdraw <= totalQueued ? toWithdraw : totalQueued;

            totalQueued -= toWithdrawFromQueue;
            depositsSinceLastUpdate += toWithdrawFromQueue;
            sharesSinceLastUpdate += stakingPool.getSharesByStake(toWithdrawFromQueue);
            toWithdraw -= toWithdrawFromQueue;
        }

        if (toWithdraw != 0) {
            if (!_shouldQueueWithdrawal) revert InsufficientLiquidity();
            withdrawalPool.queueWithdrawal(_account, toWithdraw);
        }

-      emit Withdraw(_account, _amount - toWithdraw);
        return toWithdraw;
    }
```
