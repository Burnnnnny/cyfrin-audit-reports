**Lead Auditors**

[Immeas](https://twitter.com/0ximmeas)

[Chinmay](https://x.com/dev_chinmayf)

**Assisting Auditors**

[Alexzoid](https://x.com/alexzoid_eth) (Formal Verification)

---

# 발견 사항 (Findings)
## 치명적 위험 (Critical Risk)


### 상환(redeem) 요청을 취소하면 출금 대기열이 영구적으로 차단됨

**설명:** 현재 헤드 항목(`_queue.nextRequestId`)이 완전히 제거되면(예: `shares`를 0으로 만들고 `controller`를 지우는 취소) `nextRequestId`를 진행시키지 않고 `AccountableWithdrawalQueue`가 헤드에서 교착 상태(deadlock)에 빠질 수 있습니다.

[`AccountableWithdrawalQueue::_processUpToShares`](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/vault/queue/AccountableWithdrawalQueue.sol#L153-L156) 및 [`AccountableWithdrawalQueue::_processUpToRequestId`](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/vault/queue/AccountableWithdrawalQueue.sol#L193-L196)에서 루프는 `nextRequestId`를 증가시키기 전에 `if (shares_ == 0) break;`를 확인합니다.
```solidity
(uint256 shares_, uint256 assets_, bool processed_) =
    _processRequest(request_, liquidity, maxShares_, precision_);

if (shares_ == 0) break;
```

헤드가 빈 항목(`controller == address(0)`)인 경우 [`AccountableWithdrawalQueue::_processRequest`](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/vault/queue/AccountableWithdrawalQueue.sol#L219)는 `(0, 0, true)`를 반환하고, `shares_ == 0`이므로 루프가 중단됩니다.
```solidity
if (request.controller == address(0)) return (0, 0, true);
```
헤드가 절대 진행되지 않으므로 후속 처리 또는 미리 보기 호출은 영원히 동일한 빈 헤드에 갇히게 됩니다.

이것은 현재 요청이 헤드에 있는 사용자가 처리 시점에 헤드 항목이 완전히 삭제되도록(예: 즉시 취소 이행) 먼지(dust) 양(1 wei라도)을 취소함으로써 트리거될 수 있습니다([`AccountableWithdrawalQueue::_delete`](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/vault/queue/AccountableWithdrawalQueue.sol#L130-L134)).
```solidity
/// @dev 대기열에서 출금 요청 및 해당 컨트롤러 삭제
function _delete(address controller, uint128 requestId) private {
    delete _queue.requests[requestId];
    delete _requestIds[controller];
}
```
헤드가 빈 슬롯이 되고 포인터가 움직이지 않으면 전체 대기열이 차단(brick)됩니다.


**영향:** 대기열이 영구적으로 멈추고 후속 사용자는 출금할 수 없게 됩니다.

**개념 증명 (Proof of Concept):** `test/vault/AccountableWithdrawalQueue.t.sol`에 다음 테스트를 추가하십시오.
```solidity
function testHeadDeletionDeadlocksQueue() public {
    // Setup: deposits are instant, redemptions are queued, cancel is instantly fulfilled
    strategy.setInstantFulfillDeposit(true);
    strategy.setInstantFulfillRedeem(false);
    strategy.setInstantFulfillCancelRedeem(true);

    // Seed vault with liquidity and create first (head) request by Alice
    // This helper deposits for Alice and Bob at 1e36 price.
    _setupInitialDeposits(1e36, DEPOSIT_AMOUNT);

    // 1) Alice creates a redeem request -> head of queue (requestId = 1)
    uint256 aliceSharesToQueue = 1;
    vm.prank(alice);
    uint256 headId = vault.requestRedeem(aliceSharesToQueue, alice, alice);
    assertEq(headId, 1, "first request should be head (id = 1)");

    // 2) Alice cancels; cancel is fulfilled instantly by the strategy.
    //    This fully removes the head request entry (controller becomes address(0)),
    //    but _queue.nextRequestId is NOT advanced by the implementation.
    vm.prank(alice);
    vault.cancelRedeemRequest(headId, alice);

    // Sanity: queue indices should still point at the deleted head
    (uint128 nextRequestId, uint128 lastRequestId) = vault.queue();
    assertEq(nextRequestId, 1, "nextRequestId remains stuck at deleted head");
    assertGe(lastRequestId, 1, "there is at least one request in the queue history");

    // 3) Charlie makes a NEW redeem request -> tail (requestId = 2).
    //    This request is perfectly processable with existing liquidity.
    token.mint(charlie, 1000e6);
    vm.prank(charlie);
    token.approve(address(vault), 1000e6);
    vm.prank(charlie);
    vault.deposit(1000e6, charlie);

    uint256 charlieShares = vault.balanceOf(charlie) / 2;
    vm.prank(charlie);
    uint256 tailId = vault.requestRedeem(charlieShares, charlie, charlie);
    assertEq(tailId, 2, "second request should be tail (id = 2)");

    // Check queue bounds reflect head(=1, deleted) and tail(=2, valid)
    (nextRequestId, lastRequestId) = vault.queue();
    assertEq(nextRequestId, 1, "still pointing at deleted head");
    assertEq(lastRequestId, 2, "tail id should be 2");

    // 4) Attempt to process. BUG: _processUpToShares reads head slot (controller==0),
    //    inner _processRequest returns (0,0,true), outer loop sees shares_==0 and BREAKS
    //    BEFORE ++nextRequestId, so NOTHING gets processed and the queue is permanently stuck.
    uint256 assetsBefore = vault.totalAssets();
    uint256 used = vault.processUpToShares(type(uint256).max);
    assertEq(used, 0, "deadlock: processing does nothing while a valid tail exists");

    (uint256 _shares, uint256 _assets) = vault.processUpToRequestId(2);
    assertEq(_shares, 0, "deadlock: processing does nothing while a valid tail exists");
    assertEq(_assets, 0, "deadlock: processing does nothing while a valid tail exists");

    // 5) Verify tail wasn't progressed at all
    assertEq(vault.claimableRedeemRequest(0, charlie), 0, "tail remains unclaimable");
    assertEq(vault.pendingRedeemRequest(0, charlie), charlieShares, "tail remains fully pending");
    assertEq(vault.totalAssets(), assetsBefore, "no reserves changed due to deadlock");
    (nextRequestId, lastRequestId) = vault.queue();
    assertEq(nextRequestId, 1, "nextRequestId is still stuck at deleted head");
}
```

**권장 완화 조치:** 처리된 경우 카운터를 증가시키고 break 대신 `continue`를 사용하는 것을 고려하십시오.
```solidity
if (shares_ == 0) {
    if (processed_) {
        ++nextRequestId;
        continue;
    }
    break;
}
```

**Accountable:** 커밋 [`2df3becf`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/2df3becf29e20d0d1707eb0567b51fe103f606ed) 및 [`b432631`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/b432631089e400742b1e584976e26e9e7ae8da85)에서 수정됨.

**Cyfrin:** 확인함. shares가 0이라도 요청이 처리되었으면 카운터가 증가합니다.


### `AccountableAsyncRedeemVault::fulfillCancelRedeemRequest`가 요청 데이터의 동기화를 해제하여 대기열 처리에 영구적인 DOS 유발 가능

**설명:** `fulfillCancelRedeemRequest()` 함수는 먼저 입력 `requestID`로 상환 요청 취소를 완료한 다음 `_reduce()`를 호출하여 요청 상태와 `totalQueuedShares`를 업데이트합니다.

```solidity
    function fulfillCancelRedeemRequest(address controller) public onlyOperatorOrStrategy {
        _fulfillCancelRedeemRequest(_requestIds[controller], controller);
        _reduce(controller, _vaultStates[controller].pendingRedeemRequest);
    }
```

문제는 `_reduce()` 호출에서 `_vaultStates[controller].pendingRedeemRequest`의 현재 값을 사용하고 있지만, 이는 `_fulfillCancelRedeemRequest()`에서 0으로 설정되었다는 것입니다.

즉, 여기서 `_reduce()`는 항상 0 지분(shares)으로 호출되며, shares 입력이 0일 때 되돌리지 않습니다. 그러나 요청 구조체와 `totalQueuedShares` 값을 손상시킵니다.

요청은 실제 지분 값과 함께 여전히 존재하며 대기열의 일반적인 일괄 처리에서 문제를 일으킵니다.

결과적인 영향의 한 가지 예는 다음과 같습니다.
1. 사용자 X가 100 shares에 대한 상환 요청을 합니다.
2. 사용자 X가 이 상환 요청을 취소합니다.
3. 그의 요청은 즉시 이행되지 않습니다(전략에 따라 다름).
4. 운영자가 `fulfillCancelRedeemRequest()`를 호출하여 이 취소를 처리합니다.
5. 호출이 제대로 진행됩니다. 결과적으로 [state.pendingRedeemRequest = 0]이지만 요청 상태에는 여전히 request.shares == 100 및 기타 값이 있습니다. 또한 `_queue.nextRequestID`는 변경되지 않습니다.
6. 이제 `processUpToShares()`를 통해 일괄 처리가 진행되면 사용자 X의 requestID도 처리되도록 보장되지만(nextRequestID에서 lastRequestID까지 대기열에 여전히 있음), 그렇게 되면 `state.pendingRedeemRequest`가 5단계에서 0으로 설정되었기 때문에 `_processUptoShares()` => `_fulfillRedeemRequest()`에서 되돌려집니다(revert).

```solidity
    function _fulfillRedeemRequest(uint128 requestId, address controller, uint256 shares, uint256 price)
        internal
        override
    {
        VaultState storage state = _vaultStates[controller];
        if (state.pendingRedeemRequest == 0) revert NoRedeemRequest();
        if (state.pendingRedeemRequest < shares) revert InsufficientAmount();
        if (state.pendingCancelRedeemRequest) revert RedeemRequestWasCancelled();
```


**영향:** 이 함수가 호출되면 requestID 데이터별로 저장된 값과 컨트롤러의 vaultState 간에 영구적인 비동기화가 발생하여 대기열 처리를 여러 방식으로 방해합니다.

여기서 소개된 예는 대기열 처리를 영구적으로 차단하는 치명적인 DOS입니다. 이는 비동기 취소 처리를 제공하는 전략에서 발생하지만 볼트가 이 동작과 호환될 것으로 예상되므로 이를 수정하는 것이 중요합니다.


**권장 완화 조치:**
```solidity
    function fulfillCancelRedeemRequest(address controller) public onlyOperatorOrStrategy {

+++        uint256 pendingShares = state.pendingRedeemRequest;
               _fulfillCancelRedeemRequest(_requestIds[controller], controller);
---          _reduce(controller, _vaultStates[controller].pendingRedeemRequest);
+++        _reduce(controller, pendingShares);
    }
```

**Accountable:** 커밋 [`84946dd`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/84946dd49dd70f9f5dfe40184beb52b734362701)에서 수정됨.

**Cyfrin:** 확인함. `pendingShares`가 이제 fulfill 전에 캐시되고 `_reduce`에 인수로 전달됩니다.


### 비동기 취소가 허용되는 경우 대기열 처리 시 치명적인 DOS 발생

**설명:** `cancelRedeemRequest()` 함수를 사용하여 대기열 처리(즉, `processUpToShares()` 및 `processUpToRequestID()`)를 되돌릴(DOS) 수 있습니다.

공격 경로는 다음과 같습니다.
- `cancelRedeemRequest()`가 `state.pendingCancelRedeemRequest = true`를 표시합니다.
- 관련 전략이 비동기 취소를 지원할 수 있으므로 이 취소가 즉시 이행되지 않는다고 가정합니다.


```solidity
    function cancelRedeemRequest(uint256 requestId, address controller) public onlyAuth {
        _checkController(controller);
        VaultState storage state = _vaultStates[controller];
        if (state.pendingRedeemRequest == 0) revert NoPendingRedeemRequest();
        if (state.pendingCancelRedeemRequest) revert CancelRedeemRequestPending();

        state.pendingCancelRedeemRequest = true;

        bool canCancel = strategy.onCancelRedeemRequest(address(this), controller); // @audit strategy can choose to return false here, thus mandating async cancellations.
        if (canCancel) {
            uint256 pendingShares = state.pendingRedeemRequest;

            _fulfillCancelRedeemRequest(uint128(requestId), controller);
            _reduce(controller, pendingShares);
        }
        emit CancelRedeemRequest(controller, requestId, msg.sender);
    }
```



- 이 단계에서 요청 상태의 지분 "감소(reducing)"도 건너뜁니다. `_reduce()`는 취소가 `fulfillCancelRedeemRequest()`를 통해 이행될 때만 호출되기 때문입니다.
- 나중에 `processUpToShares()`가 호출되면 `_processRequest()`는 일반 요청 데이터를 반환합니다(취소 로직에서 request.shares가 감소되지 않았으므로 "0 값"을 반환하지 않음) => 따라서 루프를 중단하거나 nextRequestID로 계속하지 않습니다.
- `_fulfillRedeemRequest()`를 호출하고 pendingCancelRedeemRequest = true로 인해 되돌립니다.

```solidity
    function _fulfillRedeemRequest(uint128 requestId, address controller, uint256 shares, uint256 price)
        internal
        override
    {
        VaultState storage state = _vaultStates[controller];
        if (state.pendingRedeemRequest == 0) revert NoRedeemRequest();
        if (state.pendingRedeemRequest < shares) revert InsufficientAmount();
        if (state.pendingCancelRedeemRequest) revert RedeemRequestWasCancelled();  // @audit
```

즉, 단일 비동기 취소(처리를 위해 대기 중인)라도 대기열 처리를 DOS할 수 있습니다.


**영향:** 전략 컨트랙트가 비동기 취소를 허용하는 경우, 정상적인 운영 중이나 공격자가 프로세스 호출을 프론트러닝하여 대기열 처리를 반복적으로 DOS할 수 있습니다.


**권장 완화 조치:** 시스템에서 비동기 취소 지원을 제거하여 이러한 종류의 공격을 방지하는 것을 고려하십시오.

**Accountable:** 커밋 [`2eeb273`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/2eeb2736eb5ba8dafa2c9f2f458b31fd8eb2d6bf)에서 수정됨.

**Cyfrin:** 확인함. 상환 요청의 비동기 취소가 이제 제거되었습니다.


### 부분 상환(Partial redemptions)을 사용하여 자산을 훔칠 수 있음

**설명:** 상환 요청이 부분적으로 채워질 때 요청 상태가 제대로 처리되지 않아 요청의 나머지 부분에 대해 부풀려진 상환 가격이 발생합니다.

- 새로운 상환이 기존 requestID로 푸시되면 업데이트된 `totalValue`와 업데이트된 `request.shares`를 사용하여 평균 상환 가격이 계산됩니다. 그런 다음 이것은 `request.sharePrice`로 저장됩니다(해당 지분에 대한 자산을 계산하는 데 사용됨).

```solidity
        } else { // if controller had an existing active requestID
            requestId = requestId_;

            WithdrawalRequest storage request = _queue.requests[requestId_];

            request.shares += shares;

            if (processingMode == ProcessingMode.RequestPrice) {
                request.totalValue += shares.mulDiv(sharePrice, _precision);
                request.sharePrice = request.totalValue.mulDiv(_precision, request.shares); // the average sharePrice is being calculated here.
            } // the whole request will have a single price, averaged recursively as new redeem requests come up.

        totalQueuedShares += shares;
    }
```


- 요청이 완전히 이행되거나 완전히 취소될 때는 요청 데이터가 삭제되므로 잘 작동합니다. 그러나 문제는 요청이 부분적으로 채워질 때 `request.shares`는 감소하지만 `totalValue`는 절대 감소하지 않는다는 것입니다.


```solidity
    function _reduce(address controller, uint256 shares) internal returns (uint256 remainingShares) {
        uint128 requestId = _requestIds[controller];
        if (requestId == 0) revert NoQueueRequest();

        uint256 currentShares = _queue.requests[requestId].shares;
        if (shares > currentShares || currentShares == 0) revert InsufficientShares();

        remainingShares = currentShares - shares;
        totalQueuedShares -= shares;

        if (remainingShares == 0) {
            _delete(controller, requestId);
        } else {
            _queue.requests[requestId].shares = remainingShares;
        } // @audit the totalValue is not updated here.
    }
```


공격 경로는 다음과 같습니다.
- 사용자는 sharePrice == 2일 때 100 shares에 대한 상환 요청을 합니다. 저장된 요청 데이터는 => {request.totalValue = 200, request.sharePrice = 2, request.shares = 100}.
- 이 요청은 부분적으로, 즉 50 shares만큼 이행됩니다. 결과 상태 => {request.totalValue = 200, request.sharePrice = 2, request.shares = 50}. 사용자는 100 자산을 얻었습니다.
- 사용자는 동일한 컨트롤러 주소에 대해 100 shares로 또 다른 상환 요청을 하여 동일한 requestID 데이터가 수정됩니다. 새로운 sharePrice는 부풀려진 "request.totalValue"와 일반적인 request.shares를 사용하여 계산됩니다. 계산에 따르면 결과 상태 => {request.totalValue = 400, request.shares = 150, and request.sharePrice = 2.66}
- 이 요청이 완전히 채워진다고 가정합니다. 사용자는 이제 400 자산을 얻습니다.

사용자는 sharePrice가 2에 불과했지만 200 shares를 상환하여 총 500 자산을 얻었습니다. 이는 계산에서 상환 가격을 계산하기 위해 부풀려진 request.totalValue 값을 사용하기 때문입니다.

- 이 request.sharePrice는 `_fulfillRedeemRequest()` 흐름에서 컨트롤러에게 빚진 자산을 계산할 때 사용됩니다.

즉, 부풀려진 양의 자산이 VaultState.maxWithdraw에 추가되어 컨트롤러가 실제 sharePrice를 사용했을 때보다 더 많은 자산을 청구할 수 있습니다.

참고: 부분 상환은 `fulfillRedeemRequest()`가 요청 지분의 일부와 함께 호출될 때 가능하며, `processUptoShares()`가 사용되고 maxShares/liquidityShares로 블록을 눌러 특정 요청이 완전히 처리되지 않을 때도 가능합니다.

**영향:** 볼트가 processingMode == RequestPrice로 구성된 경우, 상환 요청이 부분적으로 이행되면 공격자가 자산을 쉽게 훔칠 수 있습니다.

이 문제는 processingMode == RequestPrice일 때만 존재합니다. 그때만 request.sharePrice 값이 빚진 자산을 계산하는 데 사용되기 때문입니다.

**권장 완화 조치:** 시스템을 단순화하기 위해 processingMode 로직을 완전히 제거하거나, `_reduce()` 함수의 일부로 `request.totalValue`에서 상환된 자산을 줄이는 것을 고려하십시오.

**Accountable:** 커밋 [`4e5eef5`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/4e5eef57464d548ec09048eae27b6fcc1489a5c3)에서 수정됨.

**Cyfrin:** 확인함. `processingMode`가 제거되었으며 `totalValue`도 제거되었습니다.

\clearpage
## 높은 위험 (High Risk)


### `AccountableAsyncRedeemVault::fulfillRedeemRequest`가 processingMode를 무시하고 상환 요청을 완료하기 위해 currentPrice를 직접 사용함

**설명:** `requestRedeem` 함수를 사용하여 상환 요청을 하면 출금 대기열에 새 요청 구조체가 푸시됩니다. 볼트의 processingMode가 `== RequestPrice`로 구성된 경우, 당시의 현재 `sharePrice`가 나중에 요청이 처리될 때 사용하기 위해 "request.sharePrice"로 저장됩니다.

`AccountableWithdrawalQueue`의 모든 함수는 이 가격을 존중하며 사용자가 받는 자산은 이 저장된 sharePrice에 따라 달라집니다(processingMode == RequestPrice인 경우).

그러나 `AccountableAsyncRedeemVault`에는 처리 모드를 무시하고 현재 `sharePrice`를 사용하는 하나의 함수가 있습니다.

```solidity
    function fulfillRedeemRequest(address controller, uint256 shares) public onlyOperatorOrStrategy {
        _fulfillRedeemRequest(_requestIds[controller], controller, shares, sharePrice());
        _reduce(controller, shares);
    }
```

여기서 `sharePrice`는 지분의 현재 가격을 가져오지만, 요청 시간 이후 `sharePrice`가 변경된 경우 `processingMode == RequestPrice`일 때 발생해서는 안 되는 처리 지연으로 인해 사용자가 더 적은 양의 자산을 얻을 수 있으므로 사용자에게 불리할 수 있습니다.

**영향:** `processingMode == RequestPrice`로 구성된 볼트의 경우, `fulfillRedeemRequest` 함수는 상환 요청 시 저장된 가격이 사용자가 받는 자산을 계산하는 데 사용될 것이라는 보장을 깨뜨리며, 이는 어떤 이유로 sharePrice가 감소한 경우 불리할 수 있습니다.

**권장 완화 조치:**
```solidity
    function fulfillRedeemRequest(address controller, uint256 shares) public onlyOperatorOrStrategy {
+++         uint256 price;
+++         if (processingMode == ProcessingMode.CurrentPrice)
+++              price = sharePrice();
+++       }
+++         else {
+++              uint128 requestId = _requestIds[controller];
+++              price = _queue.requests[requestId].sharePrice;
+++      }

               _fulfillRedeemRequest(_requestIds[controller], controller, shares, price);
               _reduce(controller, shares);
           }
```

**Accountable:** 커밋 [`4e5eef5`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/4e5eef57464d548ec09048eae27b6fcc1489a5c3)에서 `processingMode`가 제거되어 해당 사항 없음.


### `AccountableOpenTerm` 대출 원금이 0이 되면 이자를 상환할 수 없음

**설명:** `AccountableOpenTerm`에서 이자는 `_scaleFactor`를 통해 가상으로 발생하지만 해당 이자를 지불/실현할 메커니즘이 없습니다. 유일한 자금 조달 경로는 `borrow()`, `supply()` 및 `repay()`입니다. `supply()`와 `repay()` 모두 먼저 인출을 서비스한 다음 `_loan.outstandingPrincipal`을 줄입니다. 원금이 0에 도달하면 `repay()`는 `loanState = Repaid`로 설정합니다. `Repaid` 상태에서는 `_requireLoanOngoing()`이 추가 `supply()/repay()`를 차단하고, `_sharePrice()`는 `assetShareRatio()`로 전환됩니다(발생한 `_scaleFactor` 무시). 결과적으로 발생한 이자는 지불할 수 없게 되며 LP(또는 수수료 수령자)에게 전달되지 않습니다.

**영향:** 차용인이 시간이 지난 후 원금을 상환하면 대출이 `Repaid`로 전환되고 발생한 모든 이자가 사실상 면제됩니다. LP는 이자 없이 원금을 돌려받습니다. 차용인은 이자를 실현하기 전에 원금을 상환함으로써 항상 이자를 피할 수 있습니다. 이는 악의적인 차용인이 의도적으로 수행하거나 지불이 원금을 먼저 감소시키기 때문에 의도치 않게 발생할 수도 있습니다. 따라서 모든 이자는 마지막 지불과 함께 상환되어야 합니다.

**개념 증명 (Proof of Concept):** `AccountableOpenTerm.t.sol`에 다음 테스트를 추가하십시오.
```solidity
function test_openTerm_repay_principal_only_setsRepaid_no_interest_paid() public {
    vm.warp(1739893670);

    // Setup borrower & terms
    vm.prank(manager);
    usdcLoan.setPendingBorrower(borrower);
    vm.prank(borrower);
    usdcLoan.acceptBorrowerRole();

    LoanTerms memory terms = LoanTerms({
        minDeposit: 0,
        minRedeem: 0,
        maxCapacity: USDC_AMOUNT,
        minCapacity: USDC_AMOUNT / 2,
        interestRate: 150_000,           // 15% APR so scale factor grows visibly
        interestInterval: 30 days,
        duration: 0,
        depositPeriod: 2 days,
        acceptGracePeriod: 0,
        lateInterestGracePeriod: 0,
        lateInterestPenalty: 0,
        withdrawalPeriod: 0
    });
    vm.prank(manager);
    usdcLoan.setTerms(terms);
    vm.prank(borrower);
    usdcLoan.acceptTerms();

    // Single LP deposits during deposit period → 1:1 shares at PRECISION
    vm.prank(alice);
    usdcVault.deposit(USDC_AMOUNT, alice, alice);
    assertEq(usdcVault.totalAssets(), USDC_AMOUNT, "vault funded");

    // Borrower draws full principal → vault drained
    vm.prank(borrower);
    usdcLoan.borrow(USDC_AMOUNT);
    assertEq(usdcVault.totalAssets(), 0, "all assets borrowed");
    assertEq(usdcLoan.loan().outstandingPrincipal, USDC_AMOUNT, "principal outstanding");

    // Time passes → interest accrues virtually (scale factor > PRECISION)
    vm.warp(block.timestamp + 180 days);
    uint256 sfBefore = usdcLoan.accrueInterest();
    assertGt(sfBefore, 1e36, "scale factor increased (virtual interest)");

    // Borrower repays EXACTLY principal (no extra for interest)
    usdc.mint(borrower, USDC_AMOUNT);
    vm.startPrank(borrower);
    usdc.approve(address(usdcVault), type(uint256).max);
    usdcLoan.repay(USDC_AMOUNT);
    vm.stopPrank();

    // Loan marked repaid even though totalAssets < totalShareValue at sfBefore
    assertEq(uint8(usdcLoan.loanState()), uint8(LoanState.Repaid), "loan flipped to Repaid");

    // After Repaid, share price uses assetShareRatio (actual assets), not the higher scale factor.
    // With one LP and totalAssets == totalSupply, ratio == PRECISION → no interest realized.
    uint256 spAfter = usdcLoan.sharePrice(address(usdcVault));
    assertEq(spAfter, 1e36, "share price fell back to assetShareRatio (no interest paid)");

    // Sanity: vault now only holds repaid principal
    assertEq(usdcVault.totalAssets(), USDC_AMOUNT, "vault holds only principal after repay");
    assertEq(usdcVault.totalSupply(), USDC_AMOUNT, "shares unchanged");

    // Now borrower cannot "pay the interest" anymore
    vm.prank(borrower);
    vm.expectRevert(); // blocked by _requireLoanOngoing()
    usdcLoan.supply(1e6);
}
```

**권장 완화 조치:** 원금/이자를 별도로 추적하는 대신 차용인 부채를 부채 지분(debt shares)으로 모델링하는 것을 고려하십시오.

`borrow(assets)` 시 `accrue()` 후 `debtShares = ceil(assets * PRECISION / price)`를 발행합니다. 여기서 `price = scaleFactor`. 그러면 부채는 `debtShares * price / PRECISION`과 같습니다. 이자 발생은 지분이 아니라 `price`만 이동시킵니다.

`repay(assets)` 시 `accrue()` 후 `sharesToBurn = floor(assets * PRECISION / price)`를 소각합니다(잔액으로 제한됨). `debtShares == 0`이면 대출이 상환된 것입니다.

이렇게 하면 현재 가격으로 이자를 항상 상환할 수 있고, "원금이 0이 되는" 막다른 골목을 방지하며, 부분/빈번한 상환을 깔끔하게 지원합니다. 프로토콜/설립 수수료가 적용되는 경우, 수수료 회계가 올바르게 유지되도록 초과 환불 전에 각 발생/정산 단계에서 수수료를 가져가십시오.

**Accountable:** 커밋 [`fce6961`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/fce6961c71269739ec35da60131eaf63e66e1726) 및 [`8e53eba`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/8e53eba7340f223f86c9c392f50b8b2d885fdd39)에서 수정됨.

**Cyfrin:** 확인함. 부채는 지분을 사용하여 추적되지 않습니다.

\clearpage
## 중간 위험 (Medium Risk)


### 볼트 지분 토큰에 대한 전송 제한을 완전히 우회 가능

**설명:** `AccountableVault.sol`(`AccountableAsyncRedeemVault`가 상속함)에는 `_checkTransfer()` 함수에 적용된 특정 전송 제한(KYC, 보낸 사람 주소가 스로틀 타임스탬프의 적용을 받는 경우)이 있습니다.

이러한 제한은 지분 보유자가 보유 지분을 이동하려고 할 때 `transfer()`/`transferFrom()` 함수(ERC20에서 상속됨)에 적용됩니다.

이러한 제한은 내부 `_transfer()` 함수가 사용될 때는 적용되지 않는데, 이는 지분 토큰이 예치 및 상환에만 이동될 것이므로 대부분의 경우 문제가 없습니다.

그러나 사용자가 `cancelRedeemRequest()` 기능을 사용하여 이러한 모든 제한을 완전히 우회하고 지분 토큰을 다른 주소로 이동할 수 있는 한 가지 경우가 있습니다.

방법은 다음과 같습니다.

- 컨트롤러가 볼트에 예치금이 있다고 가정합니다.
- 컨트롤러가 상환 요청을 합니다.
- 컨트롤러가 즉시 상환 요청을 취소합니다.
- 컨트롤러가 `claimCancelRedeemRequest()`를 호출하여 지분 토큰이 "receiver" 주소로 전송됩니다.

```solidity
   function claimCancelRedeemRequest(uint256 requestId, address receiver, address controller)
        public
        onlyAuth
        returns (uint256 shares)
    {
        _checkController(controller);
        VaultState storage state = _vaultStates[controller];
        shares = state.claimableCancelRedeemRequest;
        if (shares == 0) revert ZeroAmount();

        strategy.onClaimCancelRedeemRequest(address(this), controller);

        state.claimableCancelRedeemRequest = 0;

        _transfer(address(this), receiver, shares); // @audit bypasses all transfer restrictions.

        emit CancelRedeemClaim(receiver, controller, requestId, msg.sender, shares);
    }
```

이 전송 단계에서는 내부 `_transfer()` 함수가 사용되어 AccountableVault 로직에 따라 적용되는 모든 전송 제한을 건너뜁니다.

**영향:** `claimCancelRedeemRequest()`를 호출할 때 이 "receiver" 주소 입력은 컨트롤러의 선택이며 `_checkTransfer()`가 우회되므로 확인이 없습니다. 이를 통해 "to" 주소가 KYC되지 않았거나 "from" 주소에서 시작된 전송이 쿨다운 시간과 함께 작동해야 하는 경우에도 지분을 전송할 수 있습니다.

이런 식으로 컨트롤러는 전송 제한을 우회하여 볼트 지분을 임의의 수신자 주소로 이동할 수 있습니다.

**권장 완화 조치:** `claimCancelRedeemRequest()`에서 수신자 주소 로직을 제거하고 취소된 지분을 다시 컨트롤러 주소로 전송하십시오. 컨트롤러는 이미 KYC된 것으로 예상되고 지분이 원래 보유자에게 돌아가는 경우 쿨다운 확인이 필요하지 않으므로 문제가 해결됩니다.

**Accountable:** 커밋 [`2eeb273`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/2eeb2736eb5ba8dafa2c9f2f458b31fd8eb2d6bf)에서 수정됨.

**Cyfrin:** 확인함. `reciever`가 이제 KYC에 대해 확인됩니다.


### `AccountableVault::_checkTransfer`에서 `transferWhitelist` 확인 누락

**설명:** AccountableVault.sol은 "transferWhitelist" 기능을 사용하여 볼트 지분 전송이 허용되어야 하는 주소를 선택하고 `_checkTransfer()`에서 확인된 다른 제한을 재정의합니다.

`transfer()` 및 `transferFrom()` 함수 모두 내부적으로 `_checkTransfer()`를 호출하지만 모든 전송 흐름에서 "transferWhitelist" 확인이 누락되었습니다.

**영향:** transferWhitelist 기능이 작동하지 않으므로 주소가 허용 목록에 있는지 여부는 차이가 없습니다.

```solidity
    /// @notice Mapping of addresses that can override transfer restrictions
    mapping(address => bool) public transferWhitelist;
```

위의 주석은 "전송 제한을 재정의할 수 있는 주소 매핑"이라고 되어 있지만 transferWhitelist가 확인되지 않으므로 사실이 아닙니다.

또한 현재 두 전략 컨트랙트 모두에서 vault.setTransferWhitelist()를 호출하는 메서드가 누락되어 있으므로 수정할 때 주의하십시오.

**권장 완화 조치:**
```solidity
    function _checkTransfer(uint256 amount, address from, address to) private {

+++      if(transferWhitelist[from] && transferWhitelist[to]) return;

        if (amount == 0) revert ZeroAmount();
        if (!transferableShares) revert SharesNotTransferable();
        if (!isVerified(to, msg.data)) revert Unauthorized();
        if (throttledTransfers[from] > block.timestamp) revert TransferCooldown();
    }
```

또한 해당 전략의 컨텍스트에서 필요한 경우 AccountableFixedTerm 및 AccountableOpenTerm 전략 컨트랙트에 메서드(vault.setTransferWhitelist()를 호출하는 메서드)를 추가하는 것을 고려하십시오.

**Accountable:** 커밋 [`6a81e38`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/6a81e389513ad690216fc8c037ec69513f3121c7)에서 화이트리스트 제거됨.

**Cyfrin:** 확인함. 화이트리스트가 제거되었습니다.


### `AccountableAsyncRedeemVault`가 화이트리스트에 없거나 KYC되지 않은 주소에 대한 예치 허용

**설명:** `AccountableAsyncRedeemVault`의 거의 모든 함수는 `onlyAuth()` 수정자를 사용하여 호출자가 KYC되었거나 화이트리스트에 있는지 확인합니다(볼트 자체 정책에 따라).

이 로직은 AccessBase.sol의 `isVerified()` 함수에서 볼 수 있습니다.

`AccountableAsyncRedeemVault::onlyAuth` 수정자는 다음과 같습니다.

```solidity
    modifier onlyAuth() {
        if (!isVerified(msg.sender, msg.data)) revert Unauthorized();
        _;
    }

```

이것은 `msg.sender`를 확인할 "Account" 주소로 전달하지만 이러한 확인이 작동하지 않습니다.

여기서 `deposit()` 함수를 보면 `msg.sender`는 예치가 수행될 실제 계정 주소가 아니라 "receiver" 주소가 실제 계정입니다. "Receiver" 주소는 지분을 받지만 화이트리스트에 있거나 KYC되었는지 확인되지 않습니다.

```solidity
    function deposit(uint256 assets, address receiver, address controller) public onlyAuth returns (uint256 shares) {
        _checkController(controller);
        if (assets == 0) revert ZeroAmount();
        if (assets > maxDeposit(controller)) revert ExceedsMaxDeposit();

        uint256 price = strategy.onDeposit(address(this), assets, receiver, controller);
        shares = _convertToShares(assets, price, Math.Rounding.Floor);

        _mint(receiver, shares);
        _deposit(controller, assets);

```

즉, KYC된 사용자가 `deposit()`을 호출하고 임의의 "receiver" 주소(`setOperator()`를 사용하여 KYC된 사용자를 운영자로 설정했으며 입력 매개변수에 대해 `controller == receiver`를 사용할 수 있는 주소)에 대해 새 지분 토큰을 발행할 수 있습니다. 이 "receiver"는 볼트 지분을 보유하고 운영자를 통해 상환하는 등 볼트에 참여할 수 있습니다.

**영향:** KYC/화이트리스트 구성은 KYC된 주소가 KYC되지 않은 주소에 대해 지분을 발행하는 것을 방지하지 못합니다.

`onlyAuth()`는 msg.sender만 확인하고 포지션을 보유한 다른 주소는 확인하지 않기 때문에 볼트의 다른 메서드에 대한 액세스 제어에도 유사한 문제가 존재할 수 있습니다.

**권장 완화 조치:** KYC된/화이트리스트된 사용자에게 부여하려는 권한이 무엇인지 문서화하는 것을 고려하십시오. KYC되지 않은 다른 주소에 대해 포지션을 열도록 허용해서는 안 되는 경우, 실제 수신자/컨트롤러 주소에 대해 인증 확인을 수행해야 합니다.

**Accountable:** 커밋 [`c804a31`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/c804a31d3e5b161065b775fa57f3590be3581e5a) 및 [`2eeb273`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/2eeb2736eb5ba8dafa2c9f2f458b31fd8eb2d6bf)에서 수정됨.

**Cyfrin:** 확인함. `reciever`와 `controller` 모두 호출 전반에 걸쳐 KYC된 것으로 확인됩니다.


### 투자 관리자(InvestmentManager)가 `AccountableFixedTerm::coverDefault`를 사용하여 누구의 토큰 승인이든 오용 가능

**설명:** `AccountableFixedTerm::coverDefault`는 대출의 투자 관리자가 시스템에 추가 자산을 추가할 수 있도록 합니다.

```solidity
    function coverDefault(uint256 assets, address provider) external onlySafetyModuleOrManager whenNotPaused {
        _requireLoanInDefault();

        loanState = LoanState.InDefaultClaims;

        IAccountableVault(vault).lockAssets(assets, provider);

        emit DefaultCovered(safetyModule, provider, assets);
    }
```

그리고 `lockAssets()`는 입력 "provider" 주소에서 자산을 가져와 볼트로 전송합니다.

즉, 자산 토큰 잔액이 있고 볼트 컨트랙트를 승인한(과거의 잠재적 보류 승인) 사용자 주소는 여기서 자금을 잃을 위험이 있습니다.

관리자는 허가 없이 임의의 제공자 주소에서 자금을 인출할 수 있으며, "제공자"는 아무것도 돌려받지 못하고 승인된 자금을 잃게 됩니다.

**영향:** 사용자 => 볼트 컨트랙트의 보류 중인 자산 승인이 대출 채무 불이행을 충당하는 데 오용될 수 있습니다.

동일한 문제가 AccountableOpenTerm에도 존재합니다.

**권장 완화 조치:** `coverDefault()`에서 "provider" 주소 로직을 제거하고 단순히 `msg.sender`에서 자산을 인출하는 것을 고려하십시오.

**Accountable:** 커밋 [`014d7fb`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/014d7fb6f11766fada9054a736a264cf1d95c9f6)에서 수정됨.

**Cyfrin:** 확인함. `provider`가 제거되었습니다.


### 수동/즉시 `fulfillRedeemRequest`가 유동성을 예약하지 않음

**설명:** `AccountableAsyncRedeemVault`는 대기열을 통해 처리할 때만(`AccountableWithdrawalQueue::processUpToShares` / `AccountableWithdrawalQueue::processUpToRequestId`) 예약된 유동성을 고려합니다.
그러나 수동 이행 경로(`fulfillRedeemRequest`)와 `requestRedeem`의 즉시 분기는 `reservedLiquidity`를 증가시키지 않고 지분을 청구 가능으로 표시합니다.

이러한 경로가 혼합되면 여러 이행이 각각 독립적으로 "유동성" 확인을 통과할 수 있으므로(초기 이행에 의해 예약된 것이 없기 때문에) 다음과 같은 상태가 생성됩니다.

```
sum(사용자 간 청구 가능 자산) > vault.totalAssets() - reservedLiquidity
```

잠재적으로 청구 가능한 자산이 사용 가능한 유동성보다 커질 수 있습니다.

**영향:** 볼트는 사용 가능한 자산보다 더 많은 청구 가능한 상환을 갖게 되어, 나중의 출금이 되돌려지고(통합 로직에 따라 다름) 사용자 간의 공정성 및 회계 문제가 발생할 수 있습니다.

**개념 증명 (Proof of Concept):** `AccountableWithdrawalQueue.t.sol`에 다음 테스트를 추가하십시오.
```solidity
function test_manualFulfill_vsQueuedFulfill_mismatch() public {
    // Setup: price = 1e36, deposits for Alice & Bob
    _setupInitialDeposits(1e36, DEPOSIT_AMOUNT);

    uint256 aliceHalf = vault.balanceOf(alice) / 2;
    uint256 bobHalf   = vault.balanceOf(bob)   / 2;

    // === (A) Queue Bob first and reserve via processor ===
    vm.prank(bob);
    uint256 bobReqId = vault.requestRedeem(bobHalf, bob, bob);
    assertEq(bobReqId, 1, "Bob should be the head of the queue");

    // Processor path reserves liquidity for Bob
    uint256 price = strategy.sharePrice(address(vault)); // 1e36
    uint256 expectedBobAssets = (bobHalf * price) / 1e36;
    uint256 used = vault.processUpToShares(bobHalf);
    assertEq(used, expectedBobAssets, "queued fulfill reserves exact assets for Bob");

    // Sanity: reservedLiquidity == Bob's claimable assets
    uint256 reservedBefore = vault.reservedLiquidity();
    assertEq(reservedBefore, expectedBobAssets, "only Bob's queued path bumped reservedLiquidity");

    // === (B) Now manually fulfill Alice (no reservation bump) ===
    vm.prank(alice);
    uint256 aliceReqId = vault.requestRedeem(aliceHalf, alice, alice);
    assertEq(aliceReqId, 2, "Alice should be behind Bob in the queue");

    // Manual fulfill creates claimables but doesn't increase reservedLiquidity
    strategy.fulfillRedeemRequest(0, address(vault), alice, aliceHalf);

    // Compute claimables in assets
    uint256 aliceClaimableShares = vault.claimableRedeemRequest(0, alice);
    uint256 bobClaimableShares   = vault.claimableRedeemRequest(0, bob);
    assertEq(aliceClaimableShares, aliceHalf, "Alice claimable shares set by manual fulfill");
    assertEq(bobClaimableShares,   bobHalf,   "Bob claimable shares set by queued processor");

    uint256 aliceClaimableAssets = (aliceClaimableShares * price) / 1e36;
    uint256 bobClaimableAssets   = (bobClaimableShares   * price) / 1e36;
    uint256 totalClaimables      = aliceClaimableAssets + bobClaimableAssets;

    // Mismatch: claimables exceed reservedLiquidity because Alice's path didn't reserve
    assertGt(totalClaimables, reservedBefore, "claimables > reservedLiquidity (oversubscription)");

    // === (C) Bob withdraws his reserved claim → consumes all reservation ===
    uint256 bobMax = vault.maxWithdraw(bob);
    assertEq(bobMax, bobClaimableAssets, "Bob can withdraw exactly his reserved amount");

    uint256 vaultAssetsBefore = vault.totalAssets();
    vm.prank(bob);
    vault.withdraw(bobMax, bob, bob);

    // After paying Bob, reservation is zero, but Alice still has claimables (unreserved)
    uint256 reservedAfter = vault.reservedLiquidity();
    assertEq(reservedAfter, 0, "all reserved liquidity consumed by Bob's withdrawal");

    uint256 aliceClaimableShares2 = vault.claimableRedeemRequest(0, alice);
    uint256 aliceClaimableAssets2 = (aliceClaimableShares2 * price) / 1e36;
    assertEq(aliceClaimableShares2, aliceHalf, "Alice still has claimables (manual path)");
    assertGt(aliceClaimableAssets2, reservedAfter, "manual claimables remain with zero reservation");

    // Optional sanity: vault asset balance decreased by Bob's withdrawal only
    uint256 vaultAssetsAfter = vault.totalAssets();
    assertEq(vaultAssetsBefore - vaultAssetsAfter, bobMax, "vault paid only the reserved portion");
}
```

**권장 완화 조치:** `_fulfillRedeemRequest`를 예약 회계의 단일 진실 소스로 만드는 것을 고려하십시오.

1. `reservedLiquidity` 증가를 `_fulfillRedeemRequest`로 이동합니다.
2. `processUpToShares` / `processUpToRequestId`에서 `reservedLiquidity` 증가를 제거합니다(이중 계산 방지).

**Accountable:** 커밋 [`c3a7cbf`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/c3a7cbf0275a758a5d816c9f3298bc95d788db4f)에서 수정됨.

**Cyfrin:** 확인함. 권장 완화 조치가 구현되었습니다. `reservedLiquidity`는 `_fulfillRedeemRequest`에서 추적되고 "process" 함수에서 제거됩니다.


### 지분 소각 메커니즘으로 인해 `AccountableFixedTerm::claimInterest` 예측 불가

**설명:** [`AccountableFixedTerm::claimInterest`](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/strategies/AccountableFixedTerm.sol#L215-L217)를 사용하면 대출 기관은 볼트 지분을 소각하고 자산을 수령하여 이미 지급된 이자의 지분을 상환할 수 있습니다. 소각은 지금까지 실제로 펀딩된 이자가 아니라 전체 기간 최대 순수익(대출 수락 시 고정됨)을 기반으로 하는 제수를 사용합니다.

```solidity
uint256 maxNetYield = PRECISION + _interestParams.netReturn;
claimedInterest = shares.mulDiv(claimableInterest, totalShares, Math.Rounding.Floor);
uint256 usedShares = claimedInterest.mulDiv(PRECISION, maxNetYield, Math.Rounding.Ceil);
```

`netReturn`은 낙관적인 기간 종료 수치이므로 초기 청구자는 청구 단위당 더 적은 지분을 소각하여 `totalSupply`를 줄이고 나중의 결과를 주문 및 타이밍에 의존하게 만듭니다. 이는 예측할 수 없는 사용자별 결과를 산출하고 초기 청구자에게 체계적인 이점을 제공하며, 특히 나중에 대출 실적이 저조하거나 채무 불이행이 발생하는 경우 초기 청구가 낙관적인 비율로 결정되고 후기 청구자가 부족분을 부담하게 되므로 해롭습니다.

대출이 채무 불이행 없이 완료되고 모두가 결국 청구하면, 동일 지분 대출 기관은 동일한 총 이자에 수렴합니다.

**영향:** * 예측할 수 없는 사용자 지급 / MEV: 두 명의 동일한 대출 기관이 순전히 청구 순서 때문에 다른 금액을 청구할 수 있습니다. 봇은 `pay()` 직후에 청구하여 수익을 개선할 수 있습니다.
* 비대칭 채무 불이행 위험: 만기 전에 대출이 채무 불이행되는 경우, 초기 청구자는 잠재적인 전체 기간 순수익을 사용하여 계산된 현금 흐름을 이미 추출했습니다. 후기/미청구자는 남은 청구 가능 이자/회수금이 적어 불공정한 "조기 청구" 최적화를 생성하고 협력적인 사용자의 손실을 악화시킵니다.
* UX / 평판 위험: "청구"를 누르는 사용자는 금액을 결정적으로 알 수 없습니다. 결과는 동일한 간격 내에서 프론트런될 수 있습니다.

**개념 증명 (Proof of Concept):** `AccountableFixedTerm.t.sol`에 다음 테스트를 추가하십시오.
```solidity
function test_earlyClaimerAdvantage_dueToMaxNetReturnBurn_usdc() public {
    vm.warp(1739893670);

    // Setup borrower/terms identical to other tests
    vm.prank(manager);
    usdcLoan.setPendingBorrower(borrower);

    vm.prank(borrower);
    usdcLoan.acceptBorrowerRole();

    vm.prank(manager);
    usdcLoan.setTerms(
        LoanTerms({
            minDeposit: 0,
            minRedeem: 0,
            maxCapacity: USDC_AMOUNT,
            minCapacity: USDC_AMOUNT / 2,
            interestRate: 1e5,
            interestInterval: 30 days,
            duration: 360 days,
            lateInterestGracePeriod: 2 days,
            depositPeriod: 2 days,
            acceptGracePeriod: 0,
            lateInterestPenalty: 5e2,
            withdrawalPeriod: 0
        })
    );

    // Equal deposits for Alice & Bob
    uint256 userDeposit = USDC_AMOUNT / 2;

    uint256 aliceBalanceBefore = usdc.balanceOf(alice);
    uint256 bobBalanceBefore   = usdc.balanceOf(bob);

    vm.prank(alice);
    usdcVault.deposit(userDeposit, alice, alice);

    vm.prank(bob);
    usdcVault.deposit(userDeposit, bob, bob);

    // Sanity: equal initial shares
    assertEq(usdcVault.balanceOf(alice), userDeposit, "alice initial shares");
    assertEq(usdcVault.balanceOf(bob),   userDeposit, "bob initial shares");

    // Accept loan
    vm.warp(block.timestamp + 3 days);
    vm.prank(borrower);
    usdcLoan.acceptLoanLocked();

    // Fund borrower to pay interest and approve
    usdc.mint(borrower, 2_000_000e6);
    vm.prank(borrower);
    usdc.approve(address(usdcLoan), 2_000_000e6);

    uint256 aliceMidClaim;
    uint256 aliceEndClaim;
    uint256 bobEndClaim;

    // Pay month by month; Alice claims once in the middle, Bob waits
    for (uint8 i = 1; i <= 12; i++) {
        uint256 nextDueDate = usdcLoan.loan().startTime + (i * usdcLoan.loan().interestInterval);
        vm.warp(nextDueDate + 1 days);

        // Borrower pays owed interest for this interval
        vm.startPrank(borrower);
        uint256 owed = _interestOwed(usdcLoan);
        usdcLoan.pay(owed);
        vm.stopPrank();

        // Alice claims right after month 6 payment
        if (i == 6) {
            vm.prank(alice);
            aliceMidClaim = usdcLoan.claimInterest();
            assertGt(aliceMidClaim, 0, "alice mid-term claim > 0");
        }
    }

    // After last payment, both can claim
    vm.prank(alice);
    aliceEndClaim += usdcLoan.claimInterest();

    vm.prank(bob);
    bobEndClaim += usdcLoan.claimInterest();

    uint256 aliceTotal = aliceMidClaim + aliceEndClaim;
    uint256 bobTotal   = bobEndClaim;

    // Alice has gotten more than Bob by claiming early
    assertGt(aliceTotal, bobTotal, "Alice (mid+end) should claim more than Bob (end only)");

    // repay & clean-up
    vm.prank(borrower);
    usdcLoan.repay(0);

    // Ensure both still redeem principal back pro-rata after interest claims
    uint256 sharesAlice = usdcVault.balanceOf(alice);
    uint256 sharesBob   = usdcVault.balanceOf(bob);

    vm.prank(alice);
    usdcVault.requestRedeem(sharesAlice, alice, alice);
    vm.prank(bob);
    usdcVault.requestRedeem(sharesBob, bob, bob);

    vm.startPrank(alice);
    uint256 maxWithdrawAlice = usdcVault.maxWithdraw(alice);
    usdcVault.withdraw(maxWithdrawAlice, alice, alice);
    vm.stopPrank();

    vm.startPrank(bob);
    uint256 maxWithdrawBob = usdcVault.maxWithdraw(bob);
    usdcVault.withdraw(maxWithdrawBob, bob, bob);
    vm.stopPrank();

    assertEq(usdcVault.balanceOf(alice), 0, "alice no shares");
    assertEq(usdcVault.balanceOf(bob),   0, "bob no shares");

    uint256 aliceBalanceAfter  = usdc.balanceOf(alice);
    uint256 bobBalanceAfter    = usdc.balanceOf(bob);

    uint256 aliceGain = aliceBalanceAfter - aliceBalanceBefore;
    uint256 bobGain   = bobBalanceAfter   - bobBalanceBefore;

    // Alice and Bob has gained the same in the end
    assertEq(aliceGain, bobGain, "alice and bob gained the same");
}
```

**권장 완화 조치:** 지분 소각을 누적기("지분당 보상") 모델로 대체하는 것을 고려하십시오. 실제 순이자(수수료 후)가 지급될 때만 `netInterest / totalShares`만큼 증가하는 고정밀 `accInterestPerShare`를 유지하십시오. 각 대출 기관은 이 누적기의 체크포인트를 추적하고, 청구 시 `(accCurrent − checkpoint) × shares`를 받고 체크포인트를 업데이트합니다.
대출 중간에 전송/발행/소각이 허용되는 경우, 먼저 현재 누적기에서 당사자의 보류 중인 이자를 정산한 다음 체크포인트를 조정하십시오.
```solidity
uint256 accInterestPerShare;
mapping(address user => uint256 index) userIndex;
mapping(address user => uint256 interest) pendingInterest;

function onTransfer(address from, address to, uint256 amount) external onlyVault nonReentrant {

    // Settle sender’s pending interest (if not mint)
    if (from != address(0)) {
        _settleAccount(from);
        userIndex[from] = accInterestPerShare;
    }

    // Settle receiver’s pending interest (if not burn)
    if (to != address(0)) {
        _settleAccount(to);
        userIndex[to] = accInterestPerShare;
    }

}

/// Internal: settle one account’s pending interest using current accumulator
function _settleAccount(address user) internal {
    uint256 shares = vault.balanceOf(user);
    uint256 idx = userIndex[user];

    if (shares == 0) {
        userIndex[user] = accInterestPerShare;
        return;
    }

    uint256 delta  = accInterestPerShare - idx;
    if (delta == 0) return;

    pendingInterest[user] += (shares * delta) / PRECISION;
    userIndex[user] = accInterestPerShare;
}
```

이렇게 하면 지급이 결정론적이고 호출 순서와 무관하게 되며, 실제로 받은 이자만 분배하고(미래 수익률 "사전 청구" 없음), 부분 지급이나 채무 불이행 시 공정하게 유지하면서 소각 없이 가격 불변성을 보존합니다.

**Accountable:** 커밋 [`19a50c8`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/19a50c8e1275545ae3e461233f4699cb681ec731) 및 [`fd74c1d`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/fd74c1d62d4b1ae8cac03f501fd398e1a6854545)에서 수정됨.

**Cyfrin:** 확인함. 이자 발생 시스템이 사용되며 볼트는 이제 전송을 위해 전략의 `onTransfer` 후크를 호출합니다.


### `AccountableOpenTerm` 대출에서 수수료가 공제되지 않음

**설명:** `AccountableOpenTerm`에서 `interestData()`는 0이 아닌 `performanceFee` 및 `establishmentFee`를 반환하지만 수수료를 부과하는 경로가 없습니다. `_accrueInterest()`는 기본 이자에 대한 `_scaleFactor`만 업데이트하며 `supply` 또는 `repay` 중 어느 것도 `FeeManager`를 호출하지 않습니다(FixedTerm의 `collect`와 달리). 결과적으로 수수료가 청구되지 않습니다.

**영향:** 프로토콜/관리자 수수료가 사실상 전혀 징수되지 않습니다.

**권장 완화 조치:** `supply()`/`repay()`에서 다른 상태 변경 전에 수수료를 부과하는 것을 고려하십시오. 경과 기간에 대한 수수료를 계산하고 `FeeManager`로 이체한 다음 진행하십시오.

**Accountable:** 커밋 [`fce6961`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/fce6961c71269739ec35da60131eaf63e66e1726) 및 [`8e53eba`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/8e53eba7340f223f86c9c392f50b8b2d885fdd39)에서 수정됨.

**Cyfrin:** 확인함. `performanceFee` 및 `establishmentFee`는 이제 오픈 기간 대출에 대해 공제됩니다.


### OpenTerm 대출의 차용인이 사실상 영원히 연체 상태를 유지할 수 있음

**설명:** 볼트 준비금이 `_calculateRequiredLiquidity()` 미만으로 떨어지면 연체(Delinquency)가 플래그 지정되어 `delinquencyStartTime`이 설정됩니다. 연체료는 유예 기간이 경과한 후에만 발생합니다. 유예 기간 만료 전에 차용인이 잠시 유동성을 복원하면(예: `supply()`/`repay()`) 연체가 해제되고 `delinquencyStartTime`이 0으로 재설정됩니다. 차용인은 즉시 다시 `borrow()`하여 준비금을 임계값으로 떨어뜨릴 수 있으며, 이자가 다시 발생하는 다음 블록에서 새로운 유예 기간을 시작할 수 있습니다. 이 "펄스"는 한 블록 내에서도 연속으로 실행될 수 있으므로 차용인은 벌금을 전혀 물지 않고 사실상 무기한 연체 상태를 유지할 수 있습니다.

**영향:** 차용인은 대출 기관을 과소 준비 상태로 유지하면서 연체료를 피할 수 있어 대출 기관 보호가 저하됩니다.

**권장 완화 조치:** 대출이 연체되는 즉시 벌금이 부과되도록 유예 기간을 완전히 제거하는 것을 고려하십시오. 이렇게 하면 복잡성이 줄어들고 다른 많은 대출 프로토콜의 작동 방식과 일치하게 됩니다.

또는 유예 기간을 누적으로 재설계하는 것을 고려하십시오. 즉, 1년 대출에는 차용인이 사용할 수 있는 누적 1주 유예 기간이 있습니다.

**Accountable:** 인지함. 벌금 유예 기간이나 벌금 활성화 여부에 관해 대출을 관리하는 방법에 대한 실행 가능한 경로가 없습니다. 상당한 유예 기간을 갖는 것은 의도적인 것이며 대체 수단으로 관리자는 항상 채무 불이행을 시작할 수 있습니다. 대부분의 사용 사례에서 차용/상환 조치는 그리 자주 발생하지 않으며 이러한 주체가 오프체인에서도 다른 곳에 자금을 배치한다는 점을 감안할 때, 그러한 조치를 취하는 것은 평판 비용이 따를 수 있습니다.


### 채무 불이행 시 출금 대기열 `RequestPrice`가 프론트런 될 수 있음

**설명:** `AccountableWithdrawalQueue`에서 `processingMode == ProcessingMode.RequestPrice`일 때, 상환 요청의 가치는 요청 시점의 주가로 고정됩니다. 요청은 나중에 잠재적으로 매우 다른 가격으로 처리됩니다.

**영향:** * 정상 운영: 이자가 발생함에 따라 가격이 일반적으로 상승하므로 요청자가 불리한 경우가 많습니다. 요청 시점에 고정하면 후속 이익을 포기하게 됩니다.
* 채무 불이행: 요청자는 연체/채무 불이행 직전에 인출을 제출하여 채무 불이행을 프론트런하고 채무 불이행 전의 더 높은 가격을 유지하여 유동성을 고갈시키고 손실을 나머지 LP에게 떠넘길 수 있습니다. 이는 공정성이 가장 중요한 시점에 손실 사회화를 악화시킵니다.

**권장 완화 조치:** 상환 가치가 항상 처리 시점에 결정되도록 `ProcessingMode.RequestPrice`(및 `AccountableWithdrawalQueue.processingMode` 전체)를 제거하는 것을 고려하십시오. 또는 상환 요청을 무효화할 큰 가격 변동에 대한 안전 장치를 구현하십시오.

**Accontable:**
커밋 [`4e5eef5`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/4e5eef57464d548ec09048eae27b6fcc1489a5c3)에서 수정됨.

**Cyfrin:** 확인함. `processingMode`가 제거되었으며 전체적으로 현재 가격이 사용됩니다.


### `AccountableFixedTerm::pay`의 자동 인출(Auto-draw)로 제3자가 원치 않는 차입 강제 가능

[`AccountableFixedTerm::pay`](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/strategies/AccountableFixedTerm.sol#L354-L357)에서 양의 `_loan.drawableFunds`는 만기 이자/수수료를 전송하기 전에 `_updateAndRelease(drawableFunds)`를 통해 자동으로 인출됩니다.
```solidity
uint256 drawableFunds = _loan.drawableFunds;
if (drawableFunds > 0) {
    _updateAndRelease(drawableFunds);
}
```

사용자가 볼트에 예치/발행할 때 `_loan.drawableFunds`가 증가하므로 제3자는 차용인이 `pay`를 호출하기 직전에 예치할 수 있습니다. 이로 인해 `pay`는 새로운 유동성만큼 `_loan.outstandingPrincipal`을 증가시키고 해당 추가 원금에 대한 남은 기간 이자를 추가하는 동시에 차용인의 동의 없이 차용인에게 자산을 릴리스합니다.

**영향:** 차용인은 원금 규모에 대한 재량권을 상실합니다. `pay`를 호출하면 부채(원금 + 미래 이자)가 예기치 않게 증가할 수 있습니다. 공격자가 각 지불 기간 전에 볼트를 "채워" 반복적으로 인출을 강제하고 미래의 이자 지불을 늘릴 수 있으므로 그리핑/경제적 DoS가 가능합니다.

**권장 완화 조치:** `AccountableFixedTerm::pay`에서 자동 인출을 제거하는 것을 고려하십시오. 대출 증가는 이자 지불 중 암시적으로 발생하는 것이 아니라 명시적인 차용인 조치(예: `draw(uint256)`)를 통해서만 발생해야 합니다.

**Accountable:** 커밋 [`03f871b`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/03f871bfc7baff5fe5f9dfbd8a0ef74e99619e78)에서 수정됨.

**Cyfrin:** 확인함. `pay`에서 "자동 인출"이 제거되었습니다.


### 잦은 `AccountableOpenTerm::accrueInterest` 호출로 이자 발생 감소

**설명:** [`AccountableOpenTerm::_linearInterest`](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/strategies/AccountableOpenTerm.sol#L602-L604)에서 이자 발생은 정수 수학 `_linearInterest(rate, dt) = rate * dt / DAYS_360_SECONDS`를 사용합니다.
```solidity
function _linearInterest(uint256 interestRate, uint256 timeDelta) internal pure returns (uint256) {
    return interestRate.mulDiv(timeDelta, DAYS_360_SECONDS);
}
```

작은 `timeDelta`의 경우 이는 종종 0으로 반올림됩니다. 그러나 `accrueInterest()`는 계산된 증분이 0인 경우에도 여전히 `_accruedAt = block.timestamp`를 설정합니다. 따라서 짧은 간격으로 반복 호출하면 경과 시간을 많은 0 증분 조각으로 버려 동일한 벽시계 기간 동안 단일 발생보다 지속적으로 낮은 `_scaleFactor`를 생성합니다.

예를 들어, 15% APY(150_000)의 경우 207초(약 4분)마다 한 번씩 호출해야 합니다.
```
360 days / 150_000 = 31104000 / 150_000 = 207
```

**영향:** 모든 행위자는 짧은 간격으로 `accrueInterest()`를 반복적으로 호출하여 이자 성장을 억제할 수 있습니다. 시간이 지남에 따라 이는 LP에게 실질적으로 적게 지불하고(주가 하락 / 차용인이 빚진 자산 감소) 이자와 관련된 프로토콜 수수료 기준을 줄입니다. 효과는 호출 케이던스 및 APR에 따라 복합적으로 작용하여 특권 액세스 없이도 측정 가능한 손실을 초래합니다.

**개념 증명 (Proof of Concept):** `AccountableOpenTerm.t.sol`에 다음 테스트를 추가하십시오.
```solidity
function test_interest_rounding_from_frequent_accrue_calls() public {
    vm.warp(1739893670);

    vm.prank(manager);
    usdcLoan.setPendingBorrower(borrower);
    vm.prank(borrower);
    usdcLoan.acceptBorrowerRole();

    // Use a common APR (15%) and short interval; depositPeriod = 0 to keep price logic simple.
    LoanTerms memory terms = LoanTerms({
        minDeposit: 0,
        minRedeem: 0,
        maxCapacity: USDC_AMOUNT,
        minCapacity: USDC_AMOUNT / 2,
        interestRate: 150_000,        // 15% APR in bps units
        interestInterval: 30 days,
        duration: 0,
        depositPeriod: 0,
        acceptGracePeriod: 0,
        lateInterestGracePeriod: 0,
        lateInterestPenalty: 0,
        withdrawalPeriod: 0
    });
    vm.prank(manager);
    usdcLoan.setTerms(terms);
    vm.prank(borrower);
    usdcLoan.acceptTerms();

    // Provide principal so interest accrues on outstanding assets.
    vm.prank(alice);
    usdcVault.deposit(USDC_AMOUNT, alice, alice);

    // Snapshot the baseline state just after start.
    uint256 snap = vm.snapshot();

    // ------------------------------------------------------------
    // Scenario A: "Spam accrual" — call accrueInterest() every 12s for 1 hour.
    // Each 12s step yields baseRate = rate * 12 / 360d ≈ 0 (integer), but _accruedAt is reset,
    // so we lose that fractional time forever.
    // ------------------------------------------------------------
    uint256 step = 180;          // 3 minutes
    uint256 total = 3600;        // 1 hour
    uint256 n = total / step;    // 300 iterations

    for (uint256 i = 0; i < n; i++) {
        vm.warp(block.timestamp + step);
        usdcLoan.accrueInterest(); // returns new scale but we just trigger the reset
    }

    // Capture the resulting scale factor after the spammy accrual pattern
    uint256 sfSpam = usdcLoan.accrueInterest(); // one more call just to read the value

    // ------------------------------------------------------------
    // Scenario B: Single accrual after the same total wall-clock time.
    // ------------------------------------------------------------
    vm.revertTo(snap);
    vm.warp(block.timestamp + total);
    uint256 sfClean = usdcLoan.accrueInterest();

    // Expect the spammed path to have strictly lower scale factor than the clean path.
    assertLt(sfSpam, sfClean, "frequent zero-delta accrual bleeds interest vs single accrual");

    // Anything more often than 207 in this case will result in no interest growth at all.
    assertEq(sfSpam, 1e36, "frequent accruals yield no interest growth");
}
```

**권장 완화 조치:** 이자율을 추적하기 위해 더 높은 정밀도를 사용하는 것을 고려하십시오. 예: 1e18 또는 1e36.

**Accountable:** 커밋 [`29c3f72`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/29c3f72ddcb73474918dc3e74a52a2dd3c247bb5)에서 수정됨.

**Cyfrin:** 확인함. `_linearInterest`는 이제 `PRECISION`으로 확장됩니다.


### `withdraw()`에서 유효하지 않은 `maxWithdraw()` 확인

**설명:** 볼트가 `maxWithdraw(controller/owner)` 대신 `maxWithdraw(receiver)`를 잘못 확인합니다.

**영향:**
- 소유자의 한도 대신 수신자의 한도를 악용하여 승인되지 않은 출금을 허용합니다.
- `withdraw()`에서 DDoS

**개념 증명 (Proof of Concept):** ❌ 위반됨: https://prover.certora.com/output/52567/ef88bd2d76b74cafb175f8d026e484b3/?anonymousKey=599db11fbc5df1632ff4006c69a03f836b23fa6c

```solidity
// MUST NOT be higher than the actual maximum that would be accepted
rule eip4626_maxWithdrawNoHigherThanActual(env e, uint256 assets, address receiver, address owner) {

    setup(e);

    storage init = lastStorage;

    mathint limit = maxWithdraw(e, owner) at init;

    withdraw@withrevert(e, assets, receiver, owner) at init;
    bool reverted = lastReverted;

    // Withdrawals above the limit must revert
    assert(assets > limit => reverted, "Withdraw above limit MUST revert");
}
```

✅ 수정 후 확인됨: https://prover.certora.com/output/52567/8e7cfdf612d64a4cb7e5d9d9d939968e/?anonymousKey=a961467ded443bd1cab3718ca882be71f38887e9

**권장 완화 조치:**
```diff
diff --git a/credit-vaults-internal/src/vault/AccountableAsyncRedeemVault.sol b/credit-vaults-internal/src/vault/AccountableAsyncRedeemVault.sol
index a64f47c..c8824bb 100644
--- a/credit-vaults-internal/src/vault/AccountableAsyncRedeemVault.sol
+++ b/credit-vaults-internal/src/vault/AccountableAsyncRedeemVault.sol
@@ -173,7 +173,7 @@ contract AccountableAsyncRedeemVault is IAccountableAsyncRedeemVault, Accountabl
     function withdraw(uint256 assets, address receiver, address controller) public onlyAuth returns (uint256 shares) {
         _checkController(controller);
         if (assets == 0) revert ZeroAmount();
-        if (assets > maxWithdraw(receiver)) revert ExceedsMaxRedeem();
+        if (assets > maxWithdraw(controller)) revert ExceedsMaxRedeem(); // @certora FIX for eip4626_maxWithdrawNoHigherThanActual (receiver -> controller)

         VaultState storage state = _vaultStates[controller];
         shares = _convertToShares(assets, state.withdrawPrice, Math.Rounding.Floor);
```

**Accountable:** 커밋 [`6dc92b0`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/6dc92b0b09e8d2e2fc01b94d41902bbf4f5fc293)에서 수정됨.

**Cyfrin:** 확인함. `controller`가 이제 `maxWithdraw`에 전달됩니다.

\clearpage
## 낮은 위험 (Low Risk)


### 상속되는 업그레이드 가능한 컨트랙트는 스토리지 충돌을 방지하기 위해 ERC7201 네임스페이스 스토리지 레이아웃 또는 스토리지 갭을 사용해야 함

**설명:** 프로토콜에는 다른 컨트랙트가 상속하는 업그레이드 가능한 컨트랙트가 있습니다. 이러한 컨트랙트는 다음 중 하나를 사용해야 합니다.
* [ERC7201](https://eips.ethereum.org/EIPS/eip-7201) 네임스페이스 스토리지 레이아웃 - [예시](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/master/contracts/access/AccessControlUpgradeable.sol#L60-L72)
* 스토리지 갭 (이것은 [오래되었으며 더 이상 선호되지 않는](https://blog.openzeppelin.com/introducing-openzeppelin-contracts-5.0#Namespaced) 방법이지만)

이상적인 완화책은 모든 업그레이드 가능한 컨트랙트가 ERC7201 네임스페이스 스토리지 레이아웃을 사용하는 것입니다.

위의 두 가지 기술 중 하나를 사용하지 않으면 업그레이드 중에 스토리지 충돌이 발생할 수 있습니다.

**Accountable:** 커밋 [`8422762`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/842276202616ce58cdf7d766c2792fc3752157ba)에서 수정됨.

**Cyfrin:** 확인함. 네임스페이스 스토리지가 이제 `AccountableStrategy`에서 사용됩니다.


### `AccountableAsyncRedeemVault::requestRedeem`에서 컨트롤러 검증 누락으로 인해 영(zero) 주소 상태 허용

**설명:** `requestRedeem()` 함수는 `_checkController(controller)` 검증을 호출하지 않아 영 주소가 볼트 상태를 축적할 수 있게 합니다.

**영향:** `zeroControllerEmptyState` 위반.

**개념 증명 (Proof of Concept):** ❌ 위반됨: https://prover.certora.com/output/52567/acc42433123e4b289c0f84e69fa52a44/?anonymousKey=e60b3d66b5574868073bfde4218b385aa2fe5f2a

```solidity
// Zero address must have empty state for all vault fields
invariant zeroControllerEmptyState(env e)
    ghostVaultStatesMaxMint256[0] == 0 &&
    ghostVaultStatesMaxWithdraw256[0] == 0 &&
    ghostVaultStatesDepositAssets256[0] == 0 &&
    ghostVaultStatesRedeemShares256[0] == 0 &&
    ghostVaultStatesDepositPrice256[0] == 0 &&
    ghostVaultStatesMintPrice256[0] == 0 &&
    ghostVaultStatesRedeemPrice256[0] == 0 &&
    ghostVaultStatesWithdrawPrice256[0] == 0 &&
    ghostVaultStatesPendingDepositRequest256[0] == 0 &&
    ghostVaultStatesPendingRedeemRequest256[0] == 0 &&
    ghostVaultStatesClaimableCancelDepositRequest256[0] == 0 &&
    ghostVaultStatesClaimableCancelRedeemRequest256[0] == 0 &&
    !ghostVaultStatesPendingCancelDepositRequest[0] &&
    !ghostVaultStatesPendingCancelRedeemRequest[0] &&
    ghostRequestIds128[0] == 0
filtered { f -> !EXCLUDED_FUNCTION(f) } { preserved with (env eFunc) { SETUP(e, eFunc); } }
```

✅ 수정 후 확인됨: https://prover.certora.com/output/52567/f385fd34e82c4635bd410279e4da2c97/?anonymousKey=82309551a07845692bfabb2164179224523f87ba

**권장 완화 조치:**
```diff
diff --git a/credit-vaults-internal/src/vault/AccountableAsyncRedeemVault.sol b/credit-vaults-internal/src/vault/AccountableAsyncRedeemVault.sol
index 4cd0a3e..a64f47c 100644
--- a/credit-vaults-internal/src/vault/AccountableAsyncRedeemVault.sol
+++ b/credit-vaults-internal/src/vault/AccountableAsyncRedeemVault.sol
@@ -113,6 +113,9 @@ contract AccountableAsyncRedeemVault is IAccountableAsyncRedeemVault, Accountabl
         onlyAuth
         returns (uint256 requestId)
     {
+        // @certora FIX for zeroControllerEmptyState
+        _checkController(controller);
+
         _checkOperator(owner);
         _checkShares(owner, shares);
```

**Accountable:** 커밋 [`e90d3de`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/e90d3de5c133c73f0e783d552bb4e256400a547c)에서 수정됨.

**Cyfrin:** 확인함. `checkController`가 함수에 수정자로 추가되었습니다.


### 예약된 자산이 볼트에서 추출될 수 있음

**설명:** 일부 전략 함수는 해당 자산이 `reservedLiquidity`의 일부인지 확인하지 않고 자산을 릴리스할 수 있습니다. `AccountableFixedTerm._loan.drawableFunds`는 대기열 `reservedLiquidity`와 동기화되는지 확인되지 않습니다. 따라서 차용인은 실수로 해야 하는 것보다 더 많은 자금을 빌릴 수 있습니다.

**영향:** 인출을 존중하는 데 필요한 자금을 릴리스하여 볼트가 지급 불능 상태가 될 수 있습니다.

**개념 증명 (Proof of Concept):** `FixedTerm.acceptLoanLocked(), FixedTerm.borrow(), FixedTerm.pay(), FixedTerm.acceptLoanDynamic(), FixedTerm.claimInterest()`에서 위반됨: https://prover.certora.com/output/52567/edb399a43d1849a9b22f027e66b17924/?anonymousKey=3dcf62dfa004381083966b3639b6a485fa2e9501

```solidity
// Reserved liquidity must not exceed total assets
invariant reservedLiquidityBacked(env e)
    ghostReservedLiquidity256 <= ghostTotalAssets256
```

**권장 완화 조치:** `reservedLiquidity`가 출금 대기열에서 증가하면 이를 FixedTerm 전략에 동기화해야 합니다.

**Accountable:** 커밋 [`979c0e`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/979c0ebe4bd5860fe9b2e446f9fac2ae3919b39c)에서 수정됨.

불변량을 충족하고 FixedTerm 대출에서 상환을 허용할 수 있는 향후 업그레이드를 방지하기 위해 문제가 해결되었지만 현재로서는 `reservedLiquidity`를 `drawableFunds`와 동기화되지 않도록 증가시킬 수 있는 방법은 없습니다.

`Repaid` 상태에서는 `_requireLoanOngoing`으로 인해 대출 후 차입이 발생할 수 없으므로 `reservedLiquidity`를 증가시키는 모든 상환에는 예치/차입이 모두 차단된 상태가 필요합니다.

**Cyfrin:** 확인함. `reservedLiquidity`는 이제 FixedTerm에서 확인됩니다.


### `Authorizable::_verify`는 EIP-712 유형 구조화 데이터 해싱을 사용해야 함

**설명:** [`Authorizable::_verify`](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/access/Authorizable.sol#L46-L78)는 `chainId`를 포함하는 임시 페이로드에 서명하지만 흐름은 [EIP-712](https://eips.ethereum.org/EIPS/eip-712) 유형 데이터 호환이 아닙니다. 이는 지갑 UX/가시성 및 상호 운용성을 제한하고 도메인 데이터(`chainId`)와 메시지 데이터를 혼합합니다.

**영향:** 사용자는 모호한 서명 프롬프트에 더 취약합니다. 생태계 호환성 약화; 더 어려운 감사/업그레이드; 컨트랙트 또는 체인 간 인코딩/패킹 실수 및 재생 버그 위험이 더 높습니다.

**권장 완화 조치:**
EIP-712를 채택하고 `chainId`를 도메인 구분자로 이동합니다(구조체에서 제거). 메시지의 기존 의도를 유지하십시오.

* **도메인:** `{ name: "Authorizable", version: "1", chainId, verifyingContract: address(this) }`.
* **유형화된 구조체 (내부에 chainId 없음):**

  ```solidity
  struct TxAuthData {
      bytes   functionCallData;   // selector + encoded args
      address contractAddress;    // target contract (can be redundant with domain; decide and document)
      address account;            // controller / signer subject
      uint256 nonce;              // per-account nonce
      uint256 blockExpiration;    // deadline
  }

  bytes32 constant TXAUTH_TYPEHASH = keccak256(
      "TxAuthData(bytes functionCallData,address contractAddress,address account,uint256 nonce,uint256 blockExpiration)"
  );
  ```
* **해싱 및 확인 (OZ EIP712 + SignatureChecker 사용):**

  ```solidity
  bytes32 structHash = keccak256(abi.encode(
      TXAUTH_TYPEHASH,
      keccak256(txAuth.functionCallData), // hash dynamic bytes
      txAuth.contractAddress,
      txAuth.account,
      txAuth.nonce,
      txAuth.blockExpiration
  ));
  bytes32 digest = _hashTypedDataV4(structHash);
  require(
      SignatureChecker.isValidSignatureNow(signer, digest, signature),
      "INVALID_SIGNATURE"
  );
  ```
**Accountable:** 커밋 [`70cd486`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/70cd4863e3bef0f80f2eeb012c79a801c099fc7e)에서 수정됨.

**Cyfrin:** 확인함. EIP-712 유형 데이터가 이제 서명에 사용됩니다.


### 배포 스크립트에 암호화되지 않은 개인 키 필요

**설명:** 배포 스크립트 [`FactoryScript.s.sol`](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/script/FactoryScript.s.sol) 및 [`FeeManagerScript.s.sol`](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/script/FeeManagerScript.s.sol)은 개인 키를 환경 변수로 일반 텍스트로 저장해야 합니다.

```solidity
uint256 deployerPk = vm.envUint("DEPLOYER_TESTNET_PK");
```

개인 키를 일반 텍스트로 저장하는 것은 버전 관리, 잘못 구성된 백업 또는 손상된 개발자 머신을 통한 우발적인 노출 가능성을 높이므로 운영 보안 위험을 나타냅니다.

더 안전한 접근 방식은 암호화된 키 저장을 허용하는 Foundry의 [지갑 관리 기능](https://getfoundry.sh/forge/reference/script/)을 사용하는 것입니다. 예를 들어, [`cast`](https://getfoundry.sh/cast/reference/wallet/import/)를 사용하여 개인 키를 로컬 키 저장소로 가져올 수 있습니다.

```bash
cast wallet import deployerKey --interactive
```

그런 다음 배포 중에 이 키를 안전하게 참조할 수 있습니다.

```bash
forge script script/Deploy.s.sol:DeployScript \
    --rpc-url "$RPC_URL" \
    --broadcast \
    --account deployerKey \
    --sender <address associated with deployerKey> \
    -vvv
```
그리고 `vm.startBroadcast()`만 사용합니다.
```solidity
vm.startBroadcast();

...

vm.stopBroadcast();
```

추가 지침은 Patrick의 [이 설명 비디오](https://www.youtube.com/watch?v=VQe7cIpaE54)를 참조하십시오.

**Accountable:** 커밋 [`79d8cfd`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/79d8cfd8dee652adb0964ade05280a745cedb3b3)에서 수정됨.

**Cyfrin:** 확인함. 배포 스크립트는 이제 일반 텍스트로 개인 키를 요구하지 않습니다.

\clearpage
## 정보 (Informational)


### 우발적인 소유권 및 관리자 포기 방지

**설명:** 상속된 `renounceOwnership()`은 마지막 권한자가 자신을 제거하여 컨트랙트를 영구적으로 소유자 없는 상태 또는 관리자 없는 상태로 만들어 중요한 기능을 차단할 수 있도록 허용합니다.

`TokenAirdrop`에서 `renounceOwnership()`을 재정의하여 항상 되돌리도록(revert) 하는 것을 고려하십시오.

**Accountable:** 커밋 [`be75091`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/be7509165d468fd19bf48bab3fa87e565412a5b6)에서 수정됨.

**Cyfrin:** 확인함.


### 일관되게 `Ownable2Step` 사용 고려

**설명:** 현재 일부 컨트랙트는 Ownable만 사용합니다. 우발적인 소유권 손실을 방지하기 위해 모든 컨트랙트가 Ownable2Step을 사용하도록 고려하십시오.


**Accontable:**
커밋 [`be75091`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/be7509165d468fd19bf48bab3fa87e565412a5b6)에서 수정됨.

**Cyfrin:** 확인함.


### 최소 예치 금액 강제 고려

볼트/전략은 임의의 소액 예금(최소 1 wei)을 허용합니다. 기능적으로는 정확하지만 먼지(dust) 예금은 합법적인 사용자에게는 사실상 쓸모가 없으며(예금 호출 가스가 종종 예치된 가치를 초과하므로) 공격자가 반올림 엣지 케이스를 악용하는 데 남용될 수 있습니다.

블랙햇의 가능한 공격 벡터를 제거하려면 `minimumDepositAmount`를 강제하는 것을 고려하십시오.

**Accountable:** [`b9edb2b`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/b9edb2bf071db803fbd16460411688207aedc85d)에서 수정됨.

**Cyfrin:** 확인함.


### ERC7540 사양 위반

**설명:** `AccountableAsyncRedeemVault`에 대해 ERC7540 사양과 몇 가지 편차가 발견되었습니다.

1. [ERC-7540](https://eips.ethereum.org/EIPS/eip-7540) 사양에 따르면:

> 소유자와 동일하지 않은 msg.sender에 대한 지분의 상환 요청 승인은 소유자의 지분에 대한 ERC-20 승인 또는 소유자가 msg.sender를 운영자로 승인한 경우에 올 수 있습니다.

`requestRedeem()`의 현재 구현은 운영자 승인(소유자에서 호출자로)만 지원합니다. 컨트랙트는 지분 승인을 위한 ERC-20 허용 경로를 구현하지 않아 요청 상환 기능을 운영자가 승인한 주소로만 제한합니다.

2. EIP에 따르면,

> 동일한 requestID를 가진 모든 요청은 동시에 Pending에서 Claimable 상태로 전환되어야 하며 동일한 환율을 받아야 합니다.

즉, 요청은 항상 전체적으로 처리되어야 합니다. 현재 볼트 구현은 부분 상환을 허용하며, 그것도 다른 주가로 허용합니다(processingMode == CurrentPrice인 경우).


**영향:** ERC7540 미준수.

**권장 완화 조치:** 볼트가 EIP를 완전히 준수하도록 의도되었는지 문서화하고, 그렇다면 그에 따라 구현을 변경하는 것을 고려하십시오.

**Accountable:** 100% 준수하려는 의도가 아니므로 이를 인정하며, 이를 문서화할 것입니다.


### `AccountableAsyncRedeemVault::cancelRedeemRequest` 흐름에서 잘못된 이벤트 방출 가능

**설명:** `cancelRedeemRequest()`는 "requestID"를 입력으로 받지만 사용되지 않으며 입력 컨트롤러 주소와 연관되어 있는지 검증되지 않습니다.

모든 취소 흐름(즉시/비동기)은 `_requestIds[controller]`에 저장된 컨트롤러 주소의 requestID로 작동하지만, 입력 requestID는 `CancelRedeemRequest()` 및 `CancelRedeemClaimable()` 이벤트의 이벤트 데이터에만 사용됩니다.

이것이 검증되지 않기 때문에 호출자는 임의의 requestID를 입력하고 이벤트에서 방출되도록 할 수 있습니다.

**영향:** 잘못된 이벤트 방출이 가능하며, 잠재적으로 프론트엔드 및 이 이벤트 데이터를 사용하는 다른 사람의 데이터 손상을 초래할 수 있습니다.

**권장 완화 조치:** `cancelRedeemRequest()` 함수 정의에서 "requestID" 매개변수를 제거하고 이벤트 방출 시 컨트롤러의 기존 requestID를 사용하기만 하면 됩니다.

**Accountable:** 커밋 [`aa64491`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/aa64491b1ebe68375793efbc961a323ea739f58c) 및 [`0675c3d`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/0675c3de2f4eae3456470f04a2241c8f60255088)에서 수정됨.

**Cyfrin:** 확인함. 컨트롤러의 상환 요청이 이제 사용됩니다.


### ERC20 0 금액 전송 거부

**설명:** `_checkTransfer` 함수는 0 금액 전송 시 되돌리며, 이는 0 값의 전송이 정상적인 전송으로 [처리되어야 함](https://eips.ethereum.org/EIPS/eip-20#transfer)을 의무화하는 ERC-20 표준을 위반합니다.

**영향:** `eip20_transferSupportZeroAmount` 및 `eip20_transferFromSupportZeroAmount` 위반.

**개념 증명 (Proof of Concept):** ❌ 위반됨: https://prover.certora.com/output/52567/9c9c3c73f4d64f9baf1284ced4f4a8f5/?anonymousKey=160f0b0d10e3f688f1981708e4aa3819e7023a80

```solidity
// EIP20-06: Verify transfer() handles zero amount transfers correctly
// EIP-20: "Transfers of 0 values MUST be treated as normal transfers and fire the Transfer event."
rule eip20_transferSupportZeroAmount(env e, address to, uint256 amount) {

    setup(e);

    // Perform transfer
    transfer(e, to, amount);

    // Zero amount transfers must succeed
    satisfy(amount == 0);
}

// EIP20-09: Verify transferFrom() handles zero amount transfers correctly
// EIP-20: "Transfers of 0 values MUST be treated as normal transfers and fire the Transfer event."
rule eip20_transferFromSupportZeroAmount(env e, address from, address to, uint256 amount) {

    setup(e);

    // Perform the transferFrom
    transferFrom(e, from, to, amount);

    // Zero amount transferFrom must succeed
    satisfy(amount == 0);
}
```

✅ 수정 후 확인됨: https://prover.certora.com/output/52567/0babf2c2b4da49ec87cc0ae00036b0e7/?anonymousKey=21b1c4b60901ee2fea0115aa8a1b0e621c04bfaa

**권장 완화 조치:**
```diff
diff --git a/credit-vaults-internal/src/vault/AccountableVault.sol b/credit-vaults-internal/src/vault/AccountableVault.sol
index 629b6d0..fb3676a 100644
--- a/credit-vaults-internal/src/vault/AccountableVault.sol
+++ b/credit-vaults-internal/src/vault/AccountableVault.sol
@@ -141,7 +141,8 @@ abstract contract AccountableVault is IAccountableVault, ERC20, AccessBase {

     /// @dev Checks transfer restrictions before executing the underlying transfer
     function _checkTransfer(uint256 amount, address from, address to) private {
-        if (amount == 0) revert ZeroAmount();
+        // @certora FIX for eip20_transferSupportZeroAmount and eip20_transferFromSupportZeroAmount
+        // if (amount == 0) revert ZeroAmount();
         if (!transferableShares) revert SharesNotTransferable();
         if (!isVerified(to, msg.data)) revert Unauthorized();
         if (throttledTransfers[from] > block.timestamp) revert TransferCooldown();
```

**Accountable:** 커밋 [`e90d3de`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/e90d3de5c133c73f0e783d552bb4e256400a547c)에서 수정됨.

**Cyfrin:** 확인함.


### `nonReentrant`가 첫 번째 수정자가 아님

**설명:** `FeeManager::withdrawProtocolFee`에서 `nonReentrant`는 첫 번째 수정자가 아닙니다. 다른 수정자의 재진입을 방지하려면 `nonReentrant` 수정자가 수정자 목록의 첫 번째 수정자여야 합니다. 일관된 재진입 보호를 위해 `nonReentrant`를 첫 번째에 두는 것을 고려하십시오.

**Accountable:** 커밋 [`c7f31b5`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/c7f31b51fc1bfb4fe96450a189751f6f72d8274d)에서 수정됨.

**Cyfrin:** 확인함.


### 사용되지 않는 오류

`https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/constants/Errors.sol`의 다음 오류는 사용되지 않습니다. 사용되지 않는 오류를 사용하거나 제거하는 것을 고려하십시오.


- [Line: 15](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/constants/Errors.sol#L15) `error InvalidVerifier();`

- [Line: 18](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/constants/Errors.sol#L18) `error InvalidExpiration();`

- [Line: 27](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/constants/Errors.sol#L27) `error UnauthorizedOwnerOrReceiver();`

- [Line: 30](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/constants/Errors.sol#L30) `error UnauthorizedController();`

- [Line: 45](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/constants/Errors.sol#L45) `error AccountAlreadyVerified();`

- [Line: 49](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/constants/Errors.sol#L49) `error NotAdminOrOperator(address account);`

- [Line: 52](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/constants/Errors.sol#L52) `error Paused(address account);`

- [Line: 59](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/constants/Errors.sol#L59) `error CancelDepositRequestPending();`

- [Line: 65](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/constants/Errors.sol#L65) `error DepositRequestWasCancelled();`

- [Line: 71](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/constants/Errors.sol#L71) `error ExceedsDepositLimit();`

- [Line: 89](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/constants/Errors.sol#L89) `error NoDepositRequest();`

- [Line: 95](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/constants/Errors.sol#L95) `error NoPendingDepositRequest();`

- [Line: 101](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/constants/Errors.sol#L101) `error NoCancelDepositRequest();`

- [Line: 113](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/constants/Errors.sol#L113) `error ProposalExpired();`

- [Line: 116](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/constants/Errors.sol#L116) `error NoPendingProposal();`

- [Line: 122](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/constants/Errors.sol#L122) `error AlreadyInQueue();`

- [Line: 141](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/constants/Errors.sol#L141) `error LoanAlreadyAccepted();`

- [Line: 153](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/constants/Errors.sol#L153) `error LoanInDefault();`

- [Line: 162](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/constants/Errors.sol#L162) `error LoanNotAcceptedByBorrower();`

- [Line: 168](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/constants/Errors.sol#L168) `error ZeroSharePrice();`

- [Line: 177](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/constants/Errors.sol#L177) `error LoanNotRepaid();`

- [Line: 186](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/constants/Errors.sol#L186) `error PaymentNotDue();`

- [Line: 192](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/constants/Errors.sol#L192) `error RequestDepositFailed();`

- [Line: 195](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/constants/Errors.sol#L195) `error RequestRedeemFailed();`

- [Line: 210](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/constants/Errors.sol#L210) `error BorrowerNotSet();`

- [Line: 213](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/constants/Errors.sol#L213) `error PriceOracleNotSet();`

- [Line: 216](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/constants/Errors.sol#L216) `error RewardsDistributorNotSet();`

**Accountable:** 커밋 [`18ce919`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/18ce919d1771bdb0951e3601c667e5608957122c)에서 오류가 제거됨.

**Cyfrin:** 확인함.


### 이벤트 없는 상태 변경

이 함수에는 상태 변수 변경이 있지만 이벤트가 방출되지 않습니다. 오프체인 인덱서가 변경 사항을 추적할 수 있도록 이벤트를 방출하는 것을 고려하십시오.

- [Line: 47](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/modules/GlobalRegistry.sol#L47)

	```solidity
	    function setSecurityAdmin(address securityAdmin_) external onlyOwner {
	```

- [Line: 53](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/modules/GlobalRegistry.sol#L53)

	```solidity
	    function setOperationsAdmin(address operationsAdmin_) external onlyOwner {
	```

- [Line: 59](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/modules/GlobalRegistry.sol#L59)

	```solidity
	    function setTreasury(address treasury_) external onlyOwner {
	```

- [Line: 65](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/modules/GlobalRegistry.sol#L65)

	```solidity
	    function setVaultFactory(address vaultFactory_) external onlyOwner {
	```

- [Line: 71](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/modules/GlobalRegistry.sol#L71)

	```solidity
	    function setRewardsFactory(address rewardsFactory_) external onlyOwner {
	```

**Accountable:** 커밋 [`13600f4`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/13600f460f9796f16d151d19dc4a1d5c35c1475d)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage
## 가스 최적화 (Gas Optimization)


### 스토리지 읽기 최적화

**설명:**
1. [`AccountableOpenTerm::_calculateRequiredLiquidity`](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/strategies/AccountableOpenTerm.sol#L642-L655): `vault` 및 `_scaleFactor`를 캐시할 수 있습니다. 또한 `_calculateRequiredLiquidity`가 `address vault_`를 매개변수로 취하도록 변경하는 것을 고려하십시오. 그러면 [`_isDelinquent`](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/strategies/AccountableOpenTerm.sol#L492-L497), [`_getAvailableLiquidity`](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/strategies/AccountableOpenTerm.sol#L658-L663), [`_borrowable`](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/strategies/AccountableOpenTerm.sol#L518-L523) 및 [`_validateLiquidityForTermChange`](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/strategies/AccountableOpenTerm.sol#L526-L533) 흐름에서도 `vault` 읽기를 캐시할 수 있습니다.

2. [`AccountableOpenTerm::_getAvailableLiquidityForProcessing`](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/strategies/AccountableOpenTerm.sol#L666-L671): `vault`를 캐시할 수 있습니다. 또한 위와 같이 `address vault_`를 매개변수로 추가한 다음 [`_processAvailableWithdrawals`](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/strategies/AccountableOpenTerm.sol#L536-L570)에서 캐시된 값을 사용하는 것을 고려하십시오.

3. [`AccountableOpenTerm::_penaltyFee`](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/strategies/AccountableOpenTerm.sol#L587-L599), [L595](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/strategies/AccountableOpenTerm.sol#L595)에서 캐시된 값 `gracePeriod`를 사용하십시오.

4. [`AccountableOpenTerm::supply`](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/strategies/AccountableOpenTerm.sol#L238-L244): `vault`를 캐시할 수 있습니다.

5. [`AccountableOpenTerm::repay`](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/strategies/AccountableOpenTerm.sol#L260-L272): `vault`를 캐시할 수 있습니다.

6. [`AccountableFixedTerm::_sharePrice`](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/strategies/AccountableFixedTerm.sol#L533-L540): `loanState`를 캐시할 수 있습니다.

7. [`AccountableStrategy::acceptBorrowerRole`](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/strategies/AccountableStrategy.sol#L181-L189): [L185](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/strategies/AccountableStrategy.sol#L185)에서 `pendingBorrower` 대신, [L188](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/strategies/AccountableStrategy.sol#L188)에서 `borrower` 대신 `msg.sender`를 사용하십시오.

8. [`AccountableStrategy::_requireLoanNotOngoing`](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/strategies/AccountableStrategy.sol#L474-L476): `loanState`를 캐시할 수 있습니다.

9. [`AccountableStrategy::_requireLoanOngoing`](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/strategies/AccountableStrategy.sol#L479-L481): `loanState`를 캐시할 수 있습니다.

10. [AccountableWithdrawalQueue::_push](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/vault/queue/AccountableWithdrawalQueue.sol#L82-L84): 생성 시 `_queue.nextRequestId`를 `1`로 설정하고 if를 제거하여 각 `_push`마다 읽기를 저장하십시오.

11. [`AccountableAsyncRedeemVault::redeem`](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/vault/AccountableAsyncRedeemVault.sol#L155-L156): `state.redeemPrice`를 캐시할 수 있습니다.

12. [`AccountableAsyncRedeemVault::withdraw`](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/vault/AccountableAsyncRedeemVault.sol#L176-L177): `state.withdrawPrice`를 캐시할 수 있습니다.

13. [`AccountableAsyncRedeemVault::_updateRedeemState`](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/vault/AccountableAsyncRedeemVault.sol#L197-L201): `state.maxWithdraw` 및 `state.redeemShares`를 캐시할 수 있습니다.

14. [`AccountableAsyncRedeemVault::_fulfillRedeemRequest`](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/vault/AccountableAsyncRedeemVault.sol#L305-L326): `state.pendingRedeemRequest`, `state.maxWithdraw` 및 `state.redeemShares`를 캐시할 수 있습니다.

15. [`AccountableAsyncRedeemVault::maxRedeem`](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/vault/AccountableAsyncRedeemVault.sol#L352-L356) 및 [AccountableAsyncRedeemVault::maxWithdraw](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/vault/AccountableAsyncRedeemVault.sol#L359-L363): 다음과 같이 다시 작성할 수 있습니다.
    ```solidity
    function maxWithdraw(address receiver) public view override returns (uint256 maxAssets) {
        VaultState storage state = _vaultStates[receiver];
        maxAssets = state.maxWithdraw;
        if (state.redeemShares == 0) return 0;
    }
    ```

16. [`Authorizable::_verify`](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/access/Authorizable.sol#L47-L75): `signer`를 캐시할 수 있습니다.

17. [`RewardsDistributorMerkle::acceptRoot`](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/rewards/RewardsDistributorMerkle.sol#L67-L68): `_pendingRoot.validAt`을 캐시할 수 있습니다.

18. [`RewardsDistributorMerkle::claim`](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/rewards/RewardsDistributorMerkle.sol#L100-L102): `claimed[account][asset])`을 캐시할 수 있습니다.

19. [`RewardsDistributorStrategy::claim`](https://github.com/Accountable-Protocol/audit-2025-09-accountable/blob/fc43546fe67183235c0725f6214ee2b876b1aac6/src/rewards/RewardsDistributorStrategy.sol#L40-L42): `claimed[account][asset])`을 캐시할 수 있습니다.


**Accountable:** 커밋 [`8e1cfa2`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/8e1cfa29de4dc4b0198e26e59acb56e7c929dbcf)에서 대부분 수정됨.

**Cyfrin:** 확인함.

\clearpage

