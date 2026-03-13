**Lead Auditors**

[Immeas](https://x.com/0ximmeas)

[MrPotatoMagic](https://x.com/MrPotatoMagic)

---

# 발견 사항 (Findings)
## 중간 위험 (Medium Risk)


### `AtomicBatcher`가 placeholder ERC-7201 namespace를 사용함

**설명:** `AtomicBatcher`는 nonce storage slot을 아직 placeholder인 ERC-7201 namespace 상수로부터 파생합니다.

```solidity
/// @notice ERC-7201 namespace for nonce storage
string private constant _NAMESPACE = "<namespace>";
```

이 값이 프로젝트별로 유일한 namespace로 교체되지 않으면, 동일한 placeholder를 재사용하는 다른 컨트랙트/도구가 같은 storage slot에 쓰게 될 수 있습니다.

**영향:** 같은 placeholder namespace를 사용하는 다른 코드와 nonce storage가 충돌할 수 있습니다. 그 결과 replay protection이 깨지거나(예상치 못한 nonce 변경), 실행 실패가 발생하거나, 공유 저장소 환경(예: EIP-7702 스타일로 EOA storage를 쓰는 실행)에서 애플리케이션 간 간섭이 생길 수 있습니다.

**권장 완화 조치:** `"<namespace>"`를 고유하고 안정적인 식별자(예: `"accountable.atomicbatcher.nonce.v1"`)로 바꾸고, 업그레이드나 배포 전반에서 immutable하게 취급하십시오.

**Accountable:** 커밋 [`2247cec`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/2247cec53a91ccc8f2d47d6d976d99308d676b85)에서 수정됨

**Cyfrin:** 확인함. namespace는 이제 `accountable.atomicbatcher.nonce.v1`입니다.


### NAV가 오래된 상태에서도 `AccountableYield::accrueAndProcess`를 통해 즉시 출금 가능

**설명:** NAV가 오래되면 `AccountableYield::onRequestRedeem`는 "즉시 처리"를 비활성화하고 요청을 큐에 강제로 넣습니다.

```solidity
canFulfill = liquidity >= assets && !_navIsStale();
```

하지만 `AccountableYield::accrueAndProcess`는 누구나 호출할 수 있고 `whenNotStale`로 보호되지 않습니다. 이 함수는 출금 큐를 즉시 처리합니다.

```solidity
function accrueAndProcess() external ... {
    _accrueFees();
    usedAssets = _processAvailableWithdrawals();
    _updateDelinquentStatus();
}
```

그 결과 사용자는 redeem 요청을 큐에 넣은 뒤, 곧바로 `accrueAndProcess()`를 호출해(심지어 router/multicall을 통해 같은 트랜잭션 안에서도 가능) NAV가 stale한 상태에서도 요청을 처리하게 만들 수 있습니다.

**영향:** "NAV가 오래되면 즉시 출금할 수 없다"는 의도된 보호가 무력화됩니다. NAV가 업데이트되지 못한 동안에도 마지막으로 알려진(오래된) NAV 기반 가격으로 출금이 처리될 수 있으며, 이는 경제적으로 부정확할 수 있습니다.

**권장 완화 조치:** NAV가 stale하면 큐 처리도 막으십시오. 예를 들어 `accrueAndProcess()`와 `_processAvailableWithdrawals()`를 유발하는 다른 public 진입점(`AccountableYield::repay` 등)에 `whenNotStale`를 추가하십시오.


**Accountable:** 커밋 [`ddcbfa5`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/ddcbfa5faac90cc6e6ff3c1f1a0e754951363ba1)에서 수정됨

**Cyfrin:** 확인함. `repay`와 `accrueAndProcess` 모두 이제 `whenNotStale` 수정자를 가집니다.


### `AccountableYield::repay`와 `publishRate`의 트랜잭션 순서에 따라 상환 회계가 되돌려질 수 있음

**설명:** `AccountableYield::repay`는 상환 금액을 기준으로 `deployedAssets`를 줄입니다.

```solidity
uint256 deployed = deployedAssets;
deployedAssets -= Math.min(remaining, deployed);
```

하지만 이후 `publishRate(uint256 newDeployedValue)`는 DVN이 보고한 값으로 `deployedAssets`를 덮어씁니다.

```solidity
uint256 oldValue = deployedAssets;
deployedAssets = newDeployedValue;
```

이 둘은 서로 독립된 트랜잭션이기 때문에 실행 순서가 중요합니다. 먼저 `repay()`가 실행되어 `deployedAssets`를 줄인 뒤, 나중에 `publishRate()`가 상환 이전 NAV를 반영한 값을 가져오면, borrower 상환의 회계 효과가 사실상 "되돌려진" 것처럼 `deployedAssets`가 다시 커질 수 있습니다.

**영향:** 트랜잭션 순서에 따라 결과가 materially 달라질 수 있습니다. 네트워크 혼잡 상황에서 DVN 업데이트가 borrower 상환의 회계 효과를 실질적으로 무효화할 수 있으며, 그 결과 보고되는 NAV/공유 가격, 수수료 누적, 대출이 완전히 상환 상태에 도달할 수 있는지 여부까지 영향을 받을 수 있습니다. 이 리스크는 NAV grace period가 짧게 설정될수록(기본 24시간이지만 1시간까지도 낮출 수 있음) 더 커집니다. 또한 DVNPublisher의 async publish/execute 흐름 때문에 제안 시점과 온체인 실행 시점 사이에 본질적인 지연이 있어, 그 사이에 상환이 끼어들 가능성도 높습니다.

**권장 완화 조치:** `DVNPublisher.PublishRequest`는 이미 `timestamp`를 포함하므로, 이를 `AccountableYield.publishRate`까지 전달하십시오(예: `publishRate(uint256 value, uint256 measuredAt)`). 그리고 `lastNavMeasuredAt`를 저장해 `measuredAt <= lastNavMeasuredAt` 또는 `measuredAt < lastRepayTime`인 업데이트는 거부/무시하도록 하여, 오래된 스냅샷이 최신 상환 회계를 덮어쓰지 못하게 하십시오.


**Accountable:** 커밋 [`5b6498a`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/5b6498af30c542f93118e5d05206b70aeeb3b17f), [`6756c97`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/6756c97db4aa2595045576bada90ba0705bb2f03), [`06b6c4c`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/06b6c4c0238f8241a440861a578f1e6a701ff4b8)에서 수정됨

**Cyfrin:** 확인함. 이제 코드는 측정 시점 timestamp를 비교하고, 값이 두 개의 중간 측정값 평균인 경우에는 더 오래된 timestamp를 사용합니다.


### `AccountableOpenTerm`에서 나중 batch 요청을 취소하면 더 이른 출금이 지연될 수 있음

**설명:** Vault는 `AccountableAsyncRedeemVault::cancelRedeemRequest(controller, receiver)`를 통해 큐에 들어간 redeem 요청을 취소할 수 있게 합니다. 이 함수는 해당 controller의 queued shares를 제거합니다.

batch 메타데이터를 Vault 큐와 동기화하기 위해, Vault는 전략 훅 `AccountableOpenTerm::onCancelRedeemRequest(...)`를 호출합니다. 그런데 이 전략은 취소된 요청이 실제로 어느 batch에 속했는지는 모른 채, 가장 오래된 `pendingBatch`부터 시작해 앞으로 이동하면서 batch 총량을 줄입니다.

```solidity
uint256 batch = pendingBatch;
...
while (shares > 0 && batch <= maxBatch && maxIter > 0) {
    uint256 batchShares = _withdrawalBatches[batch].totalShares;
    if (batchShares >= shares) {
        _withdrawalBatches[batch].totalShares -= shares;
        break;
    } else {
        _withdrawalBatches[batch].totalShares = 0;
        shares -= batchShares;
        batch++;
    }
    --maxIter;
}
```

만약 사용자가 더 나중 batch에서 생성한 요청을 취소하면, 이 로직은 그 취소 shares를 가장 이른 batch의 `totalShares`에서 차감합니다. 그 결과 `pendingBatch.totalShares`가, 실제로 FIFO 큐 맨 앞에 남아 있는 shares보다 더 작아질 수 있습니다.

처리 시 전략은 `min(queueMaxShares, batch.totalShares)`만큼만 처리하고, 다음(아직 만료되지 않은) batch에 도달하면 중단합니다.

**영향:** 이미 만료되어 처리 가능해야 하는 더 이른 batch의 출금이, 유동성이 충분해도 인위적으로 지연될 수 있습니다. 전략이 `pendingBatch.totalShares`를 과소평가한 채 미래 batch로 넘어가 버리기 때문입니다. 이로 인해 lender의 출금 liveness가 저하되고, 예상치 못한 대기 시간이 발생할 수 있습니다.

**개념 증명 (Proof of Concept):** 다음 테스트를 `test/strategies/AccountableOpenTermBatch.t.sol`에 추가하십시오.
```solidity
function test_cancelFromFutureBatch_canDelayEarlierBatchProcessing() public {
    _setupLoanWithBatches(4 days, 7 days);

    // Three lenders deposit, borrower borrows everything (no immediate liquidity).
    vm.prank(alice);
    usdcOpenTermVault.deposit(USDC_AMOUNT, alice, alice);
    vm.prank(bob);
    usdcOpenTermVault.deposit(USDC_AMOUNT, bob, bob);
    vm.prank(charlie);
    usdcOpenTermVault.deposit(USDC_AMOUNT, charlie, charlie);

    vm.prank(borrower);
    usdcOpenTermLoan.borrow(USDC_AMOUNT * 3);

    // Batch 0: Alice + Bob queue withdrawals in the first interval.
    uint256 aliceB0 = USDC_AMOUNT / 2;
    uint256 bobB0 = USDC_AMOUNT / 3;
    vm.prank(alice);
    usdcOpenTermVault.requestRedeem(aliceB0, alice, alice);
    vm.prank(bob);
    usdcOpenTermVault.requestRedeem(bobB0, bob, bob);

    WithdrawalBatch memory b0Before = usdcOpenTermLoan.withdrawalBatches(0);
    assertEq(b0Before.totalShares, aliceB0 + bobB0, "batch0 tracks Alice+Bob");

    // Move to next interval -> Batch 1 is created by Charlie.
    vm.warp(block.timestamp + 7 days);
    uint256 charlieB1 = USDC_AMOUNT / 5;
    vm.prank(charlie);
    usdcOpenTermVault.requestRedeem(charlieB1, charlie, charlie);

    assertEq(usdcOpenTermLoan.currentBatch(), 1, "batch1 created");

    WithdrawalBatch memory b1Before = usdcOpenTermLoan.withdrawalBatches(1);
    assertEq(b1Before.totalShares, charlieB1, "batch1 tracks Charlie");

    // Charlie cancels. Strategy reduces starting from pendingBatch (0),
    // even though Charlie's request was created in batch 1.
    vm.prank(charlie);
    usdcOpenTermVault.cancelRedeemRequest(charlie, charlie);

    WithdrawalBatch memory b0AfterCancel = usdcOpenTermLoan.withdrawalBatches(0);
    WithdrawalBatch memory b1AfterCancel = usdcOpenTermLoan.withdrawalBatches(1);

    // NOTE: This shows the core accounting problem: batch0 shrinks (even though Alice+Bob are still queued),
    // and batch1 stays unchanged (even though Charlie is no longer queued).
    assertEq(
        b0AfterCancel.totalShares,
        (aliceB0 + bobB0) - charlieB1,
        "batch0 reduced by Charlie cancel (mis-attributed)"
    );
    assertEq(b1AfterCancel.totalShares, charlieB1, "batch1 unchanged (stale metadata)");

    // Queue now contains only Alice+Bob.
    assertEq(usdcOpenTermVault.totalQueuedShares(), aliceB0 + bobB0, "queue excludes cancelled Charlie");

    // We are already past batch0 expiry (4d) and before batch1 expiry (7d+4d).
    assertGe(block.timestamp, b0Before.expiry, "past batch0 expiry");
    assertLt(block.timestamp, b1Before.expiry, "before batch1 expiry");

    // Borrower repays enough liquidity to process ALL queued shares (Alice+Bob).
    // Due to understated batch0.totalShares, processing only does (alice+bob-charlie) shares and then stops at batch1.
    usdc.mint(borrower, USDC_AMOUNT * 3);
    vm.startPrank(borrower);
    usdc.approve(address(usdcOpenTermLoan), type(uint256).max);
    usdcOpenTermLoan.repay(USDC_AMOUNT * 3);
    vm.stopPrank();

    // Remaining queue shares == the "missing" amount (charlieB1), even though Charlie cancelled.
    // These are actually part of Alice/Bob's earlier requests that got pushed into the next batch window.
    assertEq(
        usdcOpenTermVault.totalQueuedShares(),
        charlieB1,
        "earlier requests rolled into next batch window (delayed until batch1 expiry)"
    );
    assertEq(usdcOpenTermLoan.pendingBatch(), 1, "pendingBatch advanced to batch1 and now blocks further processing");

    // Alice requested 500e11 shares and should be fully claimable:
    uint256 aliceClaimableShares = usdcOpenTermVault.maxRedeem(alice);
    assertEq(aliceClaimableShares, 500_000_000_000, "Alice fully claimable in batch0");

    // Bob requested 333333333333 shares, but only part of it was processed due to the bug.
    // From the trace: bob got RedeemClaimable(..., shares: 133333333333)
    uint256 bobClaimableShares = usdcOpenTermVault.maxRedeem(bob);
    assertEq(bobClaimableShares, 133_333_333_333, "Bob only partially claimable in batch0 (bug)");

    // The remainder should still be queued (333333333333 - 133333333333 = 200000000000)
    assertEq(usdcOpenTermVault.totalQueuedShares(), 200_000_000_000, "Remaining Bob shares still queued");

    // Demonstrate users can redeem what is currently claimable:
    vm.prank(alice);
    usdcOpenTermVault.redeem(aliceClaimableShares, alice, alice);

    vm.prank(bob);
    usdcOpenTermVault.redeem(bobClaimableShares, bob, bob);

    // After redeeming claimable amounts, queue should still contain Bob's remainder
    assertEq(usdcOpenTermVault.totalQueuedShares(), 200_000_000_000, "Bob remainder still queued after partial redeem");

    // Remaining shares can't become claimable until batch1 expires.
    // Warp past batch1 expiry and trigger processing again.
    WithdrawalBatch memory batch1 = usdcOpenTermLoan.withdrawalBatches(1);
    vm.warp(batch1.expiry + 1);
    usdcOpenTermLoan.processAvailableWithdrawals();

    // Now Bob's remainder should become claimable:
    uint256 bobClaimableAfter = usdcOpenTermVault.maxRedeem(bob);
    assertEq(bobClaimableAfter, 200_000_000_000, "Bob remainder becomes claimable only after batch1 expiry");

    // And Bob can finally redeem the rest:
    vm.prank(bob);
    usdcOpenTermVault.redeem(bobClaimableAfter, bob, bob);

    assertEq(usdcOpenTermVault.totalQueuedShares(), 0, "Queue fully drained after delayed processing");
}
```

**권장 완화 조치:** 취소가 정확한 batch에서 차감되도록 하십시오. queue에 들어갈 때 각 redeem request(또는 controller의 pending request)가 어느 batch에 속하는지 추적하고, 취소 시 그 batch에서 차감하도록 구현하십시오.

**Accountable:** 커밋 [`ec9ec5e`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/ec9ec5e489c4b02c4b5cee6debced79bcf7c3e3b)에서 수정됨

**Cyfrin:** 확인함. 이제 per-controller batch tracking을 통해 올바른 batch에서 shares가 차감됩니다.


### `YieldStrategyFactory.createYieldStrategy`에 필요한 modifier가 빠져 검증되지 않은 전략 배포가 가능함

**설명:** `createYieldStrategy` 함수는 `AccountableYield` 전략을 permissionless하게 배포할 수 있게 합니다. 하지만 배포 과정에서 다음을 검증하지 않습니다.
1. `whenNotPaused`를 통한 paused 상태 확인
2. signer가 설정된 경우 `onlyVerified`를 통한 인증 데이터 검증
3. `onlyWhitelistedAsset`를 통한 asset 화이트리스트 검증

**개념 증명 (Proof of Concept):** 아래처럼 다른 전략 factory인 `OpenTermFactory`, `FixedTermFactory`는 이러한 검증을 이미 적용합니다.

[`YieldStrategyFactory.sol`](https://github.com/Accountable-Protocol/credit-vaults-internal/blob/main2/src/factory/YieldStrategyFactory.sol)
```solidity
function createYieldStrategy(YieldFactoryParams memory params)
        external
        returns (address strategyProxy, address vault)
    {
```

[`OpenTermFactory.sol`](https://github.com/Accountable-Protocol/credit-vaults-internal/blob/main2/src/factory/OpenTermFactory.sol)
```solidity
function createOpenTermLoan(OpenTermFactoryParams memory params)
        external
        whenNotPaused
        onlyVerified
        onlyWhitelistedAsset(params.asset)
        returns (address strategyProxy, address vault)
    {
```

[`FixedTermFactory.sol`](https://github.com/Accountable-Protocol/credit-vaults-internal/blob/main2/src/factory/FixedTermFactory.sol)
```solidity
function createFixedTermLoan(FixedTermFactoryParams memory params)
        external
        whenNotPaused
        onlyVerified
        onlyWhitelistedAsset(params.asset)
        returns (address strategyProxy, address vault)
    {
```

**권장 완화 조치:** `createYieldStrategy` 함수에 `whenNotPaused`, `onlyVerified`, `onlyWhitelistedAsset` modifier를 적용하는 것을 고려하십시오.

**Accountable:** 커밋 [`f8d4a3f`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/f8d4a3fbb19ea995cc27a7a8fb4a9896129247e7)에서 수정됨

**Cyfrin:** 확인함. modifier가 적용되었습니다.

\clearpage
## 낮은 위험 (Low Risk)


### `AccountableOpenTerm`의 rate publish/rollback은 연체 상태를 갱신하지 않음

**설명:** `AccountableOpenTerm`에서 새 rate 업데이트 경로(`publishRate()` / `rollbackRate()`)는 `_accrueInterest()`를 호출한 뒤 핵심 경제 변수(`interestRate`, scale-factor 관련 상태, accrual timestamps 등)를 갱신합니다. 하지만 이 함수들은 대출의 경제 상태를 바꾼 뒤 `_updateDelinquentStatus()`를 호출하지 않습니다.

**영향:** 이후 다른 상호작용이 `_updateDelinquentStatus()`를 호출하기 전까지 연체 상태가 오래된 값으로 남을 수 있습니다. 이는 연체 추적과 벌금 적용 시점에 일시적인 불일치를 일으키고, rate 변경 직후의 연체 상태를 신뢰하는 모니터링/자동화에도 영향을 줄 수 있습니다.

**권장 완화 조치:** `publishRate()`와 `rollbackRate()` 끝에서 모두 `_updateDelinquentStatus()`를 호출하여, 연체 상태가 항상 최신 rate/accrual 상태를 반영하게 하십시오.

**Accountable:** 커밋 [`39c60c7`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/39c60c7c2ceaf4a7de87013aeb27acabbff088b5), [`f350a8d`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/f350a8dc1769f9401b4d1ef62d3748540545ca4d)에서 수정됨.

**Cyfrin:** 확인함. `_updateDelinquentStatus()`가 이제 `publishRate()`와 `rollbackRate()` 양쪽 모두에서 호출됩니다.


### 수수료 구조 업데이트가 대출 종료 후에도 accrual을 유발할 수 있음

**설명:** `AccountableYield`와 `AccountableOpenTerm` 모두에서 `onFeeStructureChange()`는 이자를 누적하기 전에 단지 `if (_loan.startTime != 0)`만 검사합니다.

```solidity
if (_loan.startTime != 0) {
    _accrueFees();     // AccountableYield
    // or _accrueInterest(); // AccountableOpenTerm
}
```

`startTime`은 한 번 설정되면 대출이 `Repaid`되거나 default 상태가 되더라도 리셋되지 않으므로, 대출이 더 이상 진행 중이 아니어도 fee-structure 변경이 accrual 로직을 다시 실행시킬 수 있습니다.

**영향:** `AccountableYield`에서는 management fee가 시간 기반이므로, 상환 후 오랫동안 accrual이 없다가 이후 fee-structure 업데이트가 발생하면 마지막 수수료 누적 이후 경과 시간을 한 번에 catch-up하며 많은 fee shares를 민팅할 수 있어, 홀더가 예상치 못하게 희석될 수 있습니다. `AccountableOpenTerm`도 마찬가지로 이자 회계를 갱신하거나(혹은 진행 중인 대출을 전제로 한 accrual 때문에 revert할 가능성도 있습니다). 다만 이는 대출 종료 후에 프로토콜이 fee structure를 갱신해야만 발생하므로 가능성은 낮습니다.

**권장 완화 조치:** `startTime != 0` 대신 loan state를 기준으로 `onFeeStructureChange()`를 제한하십시오. 예를 들어 `loanState == Ongoing*`일 때만 `_accrueFees()` / `_accrueInterest()`를 호출하거나, 다른 호출들처럼 `_requireLoanOngoing()`를 사용할 수 있습니다.

**Accountable:** 커밋 [`5f8fd3`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/5f8fd343287101ae69b6bc767e9459a9543526ab)에서 수정됨

**Cyfrin:** 확인함. 검사는 이제 `loanState == LoanState.OngoingDynamic`으로 바뀌었습니다.


### `AccountableOpenTerm` 훅의 연체 상태 갱신은 queue 반영 전 상태를 기준으로 동작함

**설명:** Vault는 전략 훅(`AccountableStrategy::onRequestRedeem`, `onDeposit`, `onMint` 등)을 먼저 호출한 뒤에야, 그 동작이 영향을 주는 Vault 상태(queued redeem의 queue totals, deposit/mint의 `totalAssets`/liquidity)를 갱신합니다.

그런데 `AccountableOpenTerm`은 이 훅 내부에서 연체 상태를 갱신합니다.

그 결과 Vault 쪽 값(`totalAssets`, `reservedLiquidity`, `totalQueuedShares` 등으로부터 계산되는 queued shares / available liquidity)에 의존하는 연체 계산은 action이 반영되기 전 스냅샷을 기준으로 이뤄질 수 있습니다.

* `AccountableOpenTerm::onRequestRedeem`: queued(즉시 처리되지 않는) 요청의 경우, 요청은 훅이 끝난 뒤에야 실제로 큐에 들어가므로, 새 queued shares가 반영되기 전에 연체 여부가 계산됩니다.
* `AccountableOpenTerm::onDeposit` / `onMint`: 훅은 Vault가 자산을 수령하거나 totals를 갱신하기 전에 실행되므로, 새로 들어온 유동성이 반영되기 전에 연체 여부가 계산됩니다.

**영향:** 연체 상태가 한 interaction 정도 지연될 수 있습니다(혹은 다른 상태 갱신이 일어날 때까지). 예를 들어:

* queued redeem가 즉시 대출을 delinquent로 표시하지 못하거나,
* 반대로 유동성을 회복시키는 deposit/mint가 즉시 delinquency를 해제하지 못할 수 있습니다.

이는 주로 correctness/timing 문제이며, 같은 트랜잭션 내에서까지 정확한 연체 판정이 필요하지 않다면 영향은 제한적입니다.

**권장 완화 조치:** 연체 계산이 추가/제거된 shares와 assets를 반영할 수 있도록, 해당 변화량을 계산 함수에 전달하는 방식을 고려하십시오. 또는 `Vault::cancelRedeemRequest`처럼 전략 훅 `strategy.updateLateStatus()`를 동작 끝에서 호출하는 패턴을 사용할 수 있습니다.

**Accountable:** 커밋 [`5f815ee`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/5f815ee49e6f88585befd3061abf2bc081ae3d8c)에서 수정됨

**Cyfrin:** 확인함. 전략 훅 내부에서의 delinquency update는 제거되었고, 이제 각 vault 함수가 마지막에 `strategy.updateLateStatus`를 호출합니다.


### `AccountableOpenTerm.loan.withdrawalPeriod`를 `0`에서 늘리면 출금이 막힐 수 있음

**설명:** `LoanTerms.withdrawalPeriod == 0`일 때 `AccountableOpenTerm::_createOrAddWithdrawalBatch`는 즉시 return하며 어떤 batch 메타데이터도 생성하거나 갱신하지 않습니다.

하지만 이 상태에서도 사용자는 queue에 들어갈 수 있습니다(예: 유동성 부족으로 `requestRedeem()`이 즉시 처리되지 않는 경우). 이후 terms가 업데이트되어 `withdrawalPeriod`가 non-zero가 되면, `_processAvailableWithdrawals()`는 "batch mode"로 전환되어 `WithdrawalBatch.totalShares`를 기준으로 shares를 처리합니다. 이때 `withdrawalPeriod == 0` 동안 쌓인 queued shares에는 대응하는 batch totals가 없을 수 있습니다.

**영향:** `withdrawalPeriod == 0`일 때 생성된 queued withdrawal은, 이후 `withdrawalPeriod > 0`로 전환되면 처리용 batch 메타데이터가 없어서 막히거나 막힌 것처럼 보일 수 있습니다. 그 결과 출금 지연이 발생하고, 운영 측에서 terms를 다시 조정해야 하는 상황이 생길 수 있습니다.

**권장 완화 조치:**
queued shares가 남아 있는 상태에서는 `withdrawalPeriod`를 증가시키지 못하게 하거나,

아니면 `withdrawalPeriod == 0`에서 `withdrawalPeriod > 0`로 전환될 때, queued withdrawals를 batch 메타데이터로 materialize하도록 하십시오. 예를 들면:

* `acceptTerms()`(또는 terms activation 경로)에서, *새* `withdrawalPeriod > 0`이고 `totalQueuedShares() > 0`이면, 현재 queued share 총량으로 `totalShares = totalQueuedShares()`인 현재 batch를 **초기화/시드**합니다. `startTime/expiry` 정렬도 적절히 맞춰야 합니다.
* 또는 `_processAvailableWithdrawals()`를 조정해, `withdrawalPeriod > 0`인데 queue가 비어 있지 않음에도 batch 메타데이터가 비어 있거나 누락된 경우:

  * 한 번은 "zero-period" 처리 경로로 fallback하거나,
  * 처리 전에 현재 queued amount를 반영하는 batch를 자동 생성하도록 할 수 있습니다.

**Accountable:** 커밋 [`10396d4`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/10396d49bbdd747cf76e961091a2c869b1194a27)에서 수정됨

**Cyfrin:** 확인함. 이제 queued shares가 남아 있는 상태에서 withdrawal period를 0에서 증가시키려 하면 `acceptTerms`가 revert합니다.


### `execute` 함수는 요청의 나이와 무관하게 seenSigner 값을 덮어씀

**설명:** `publish` 함수는 인증된 signer가 현재 처리 중인 batch에 대해 요청을 publish하도록 허용합니다. 설계상 signer는 한 batch 안에서 여러 요청을 publish할 수 있고, 각 요청은 서로 다른 timestamp를 가질 수 있습니다.

승인된 executor가 `execute`를 호출하면, 아래 루프는 signer의 첫 번째 요청 값을 두 번째 요청 값으로 덮어씁니다(요청이 2개라고 가정할 때). 그러나 두 번째 요청의 `request.timestamp`가 첫 번째 요청보다 더 오래된 값일 수도 있습니다. 이 경우 함수는 최신 값이 아니라 상대적으로 오래된 요청 값을 사용하게 되어, publish rate가 약간 부정확해질 수 있습니다.
```solidity
          for (uint256 j = 0; j < uniqueSigners; ++j) {
                if (requests[i].signer == seenSigners[j]) {
                    // Update to latest value from this signer
                    values[j] = requests[i].value;
                    isDuplicate = true;
                    break;
                }
            }
```

**개념 증명 (Proof of Concept):** 간단한 예를 들면:
 - Alice가 두 요청 R1, R2를 제출
 - R1과 R2의 timestamp는 각각 20, 10
 - execute 중 R1의 값이 R2의 값으로 덮어써지는데, R2는 더 오래된 요청임

**권장 완화 조치:** signer가 한 batch에서 여러 요청을 가질 수 있다면, 더 최신 timestamp인 경우에만 값을 덮어쓰도록 하십시오.

**Accountable:** 커밋 [`141ca3b`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/141ca3b7025ddb9316eb55b0399ed9999daf60aa)에서 수정됨

**Cyfrin:** 확인함. 이제 timestamp가 더 최신일 때만 값을 갱신합니다.


### penalty accrual 전에 `lastTotalAssets`가 갱신되어 오래된 값이 저장됨

**설명:** `AccountableYield` 전략 전반에서 `lastTotalAssets` 변수는 `_totalAssets` 함수가 반환한 값을 저장합니다. 그런데 `_totalAssets`는 `accruedPenalties`를 포함하므로, `lastTotalAssets`를 갱신하기 전에는 penalties가 먼저 누적되어야 합니다.
```solidity
/// @dev Total assets managed = vault assets + deployed assets + accrued penalties
    function _totalAssets(address vault_) internal view returns (uint256) {
        return IAccountableVault(vault_).totalAssets() + deployedAssets + accruedPenalties;
    }
```

하지만 `AccountableYield` 전반에서 `lastTotalAssets`를 갱신할 때는 penalties를 먼저 누적하지 않습니다.

[`AccountableYield.borrow`](https://github.com/Accountable-Protocol/credit-vaults-internal/blob/ba1c7754f891dd6d28a4b47d1989c8b03073abe2/src/strategies/AccountableYield.sol#L282)

[`AccountableYield.onDeposit`](https://github.com/Accountable-Protocol/credit-vaults-internal/blob/ba1c7754f891dd6d28a4b47d1989c8b03073abe2/src/strategies/AccountableYield.sol#L367)

[`AccountableYield.onMint`](https://github.com/Accountable-Protocol/credit-vaults-internal/blob/ba1c7754f891dd6d28a4b47d1989c8b03073abe2/src/strategies/AccountableYield.sol#L392)

[`AccountableYield._accrueFees/_accruedFeeShares`](https://github.com/Accountable-Protocol/credit-vaults-internal/blob/ba1c7754f891dd6d28a4b47d1989c8b03073abe2/src/strategies/AccountableYield.sol#L463)

비슷하게 `_totalAssets` 값을 penalties 누적 없이 직접 읽는 경우도 존재합니다.

[`AccountableYield._accruedFeeShares`](https://github.com/Accountable-Protocol/credit-vaults-internal/blob/ba1c7754f891dd6d28a4b47d1989c8b03073abe2/src/strategies/AccountableYield.sol#L515)  -  이 경우 `_accruedFeeShares`는 이 `newTotalAssets` 값을 `_sharePrice`에 반환하므로 stale share price까지 유발합니다.

[`AccountableYield._calculateRequiredLiquidity`](https://github.com/Accountable-Protocol/credit-vaults-internal/blob/ba1c7754f891dd6d28a4b47d1989c8b03073abe2/src/strategies/AccountableYield.sol#L620)


**권장 완화 조치:** `lastTotalAssets`를 갱신하기 전뿐 아니라 `_totalAssets` 반환값을 직접 읽기 전에도 penalties를 먼저 누적하는 것을 고려하십시오.

**Accountable:** 커밋 [`97f8b1a`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/97f8b1aed9e52e67c788892e97346d9b83bcacb1)에서 수정됨

**Cyfrin:** 확인함. 위에 언급한 경우들에서는 이제 penalties가 먼저 누적됩니다.

\clearpage
## 정보 (Informational)


### `AccountableOpenTerm`의 수동 이자율 제안 경로에는 상한이 없음

**설명:** 수동 이자율 경로는 상한 검증 없이 이자율을 제안/큐잉할 수 있습니다. 반면 DVN publishing 경로는 rate를 적용할 때 상한을 검증합니다.

* DVN 흐름: `AccountableOpenTerm::publishRate(uint256 newRate)`는 적용 전에 `newRate`를 `MAX_PUBLISH_RATE`와 비교합니다.
* 수동 흐름: `AccountableOpenTerm::proposeInterestRate(...)`는 pending rate를 큐에 넣고, `approveInterestRateChange()`가 이를 적용하지만, 큐에 들어가는 rate 자체에는 상한이 없습니다.

**권장 완화 조치:** 동일한 상한 검사를 `proposeInterestRate(...)`에 추가하는 것을 고려하십시오(권장). 그래야 비정상적으로 큰 이자율이 애초에 큐에 들어가지 못합니다.

**Accountable:** 커밋 [`4a737a6`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/4a737a63a92cff754a0e712ce5f8124f601829c1)에서 수정됨

**Cyfrin:** 확인함.


### `AccountableYield::setNavGracePeriod`는 잘못된 입력에 대해 `Unauthorized` 에러를 사용함

**설명:** `AccountableYield.setNavGracePeriod()`는 `period < MIN_NAV_GRACE_PERIOD`일 때 `Unauthorized()`로 revert하지만, 이는 권한 문제가 아니라 입력값 검증 실패입니다.

전용 에러(예: `InvalidNavGracePeriod()`)를 사용하거나 일반적인 입력 검증 에러를 재사용하는 것을 고려하십시오.

**Accountable:** 커밋 [`d051169`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/d05116991e253ad2e348604c3ca50770ee554607)에서 수정됨

**Cyfrin:** 확인함.


### `DVNPublisher::publish`는 최대 허용 업데이트 연령을 강제하지 않음

**설명:** `DVNPublisher::publish`는 `request.timestamp`가 미래가 아닌지만 검증하고, 이미 stale한 요청(제출 시점에 이미 너무 오래된 요청)에 대해서는 거부하지 않습니다. stale 처리 자체는 나중 `DVNPublisher::execute`에서만 이뤄집니다.

예를 들어 `require(request.timestamp + maxStaleness >= block.timestamp)`와 같은 체크를 `publish()`에 추가해, 제출 시점부터 이미 stale한 요청은 거부하고 `_pendingRequests`를 불필요하게 쌓지 않도록 하는 것을 고려하십시오.

**Accountable:** 커밋 [`5afc5fd`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/5afc5fd48b7ee0e2649c46e6c1dd261f9e00de49)에서 수정됨

**Cyfrin:** 확인함.


### 잘못된 batch ID에 대해 `publishedDataByBatchId`가 revert하는 것을 고려할 것

**설명:** `publishedDataByBatchId`는 published data를 반환하지만, `id`가 `currentBatchId`보다 작은지 확인하지 않습니다. 그 결과 invalid ID에 대해서도 빈 `PublishedData` 구조체를 반환하게 됩니다. 즉시 위험은 없지만, 향후 통합체 문제를 줄이려면 잘못된 ID 값은 거부하는 것이 더 안전합니다.

```solidity
function publishedDataByBatchId(uint256 id) external view returns (PublishedData memory) {
        return _publishedData[id];
    }
```

**권장 완화 조치:** `id`가 `currentBatchId` 이상이면 revert하도록 하는 것을 고려하십시오.

**Accountable:** 커밋 [`d721846`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/d721846475486afd796fd6fa159e95143e7d4b98)에서 수정됨

**Cyfrin:** 확인함.

\clearpage
## 가스 최적화 (Gas Optimization)


### `AtomicBatcher::_getNonceSlot`에서 `baseSlot`을 미리 계산해 둘 것

**설명:** `AtomicBatcher::_getNonceSlot`는 호출될 때마다 ERC-7201 namespace base slot을 런타임에 계산합니다.
```solidity
function _getNonceSlot(address account) private pure returns (bytes32) {
    bytes32 namespaceHash = keccak256(bytes(_NAMESPACE));
    bytes32 baseSlot = keccak256(abi.encode(uint256(namespaceHash) - 1)) & ~bytes32(uint256(0xff)); // @audit can be pre-computed
    return keccak256(abi.encode(account, uint256(baseSlot)));
}
```
namespace는 상수이므로, 파생된 `baseSlot`도 미리 계산해 `bytes32` 상수로 저장할 수 있습니다.
```solidity
// keccak256(abi.encode(uint256(keccak256("accountable.atomicbatcher.nonce.v1")) - 1)) & ~bytes32(uint256(0xff))
bytes32 private constant NONCE_BASE_SLOT = 0xa68386067ee8ee669468449acf0ad3e2ae0d09e4d99f78eaa329c6681c06b900;
```

**Accountable:** 커밋 [`fce94d2`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/fce94d2c67b3937ea1106f66138dbff7118227d8)에서 수정됨

**Cyfrin:** 확인함.


### `AtomicBatcher::_hashCallArray`에서 `callTypeHash`를 미리 계산해 둘 것

**설명:** `AtomicBatcher::_hashCallArray`는 호출될 때마다 call type hash(즉 call type string/type description의 `keccak256`)를 다시 계산합니다.
```solidity
function _hashCallArray(Call[] calldata calls) private pure returns (bytes32) {
    bytes32 callTypeHash = keccak256("Call(address target,uint256 value,bytes data)");
```
이 값은 상수이므로 한 번 미리 계산해 `bytes32` 상수로 둘 수 있습니다.

예를 들어 `bytes32 private constant _CALL_TYPEHASH = keccak256("...");`를 정의하고, `_hashCallArray()`에서 매번 계산하는 대신 `_CALL_TYPEHASH`를 사용하십시오.

**Accountable:** 커밋 [`6a63afe`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/6a63afee9fca88312fad4c208797b4b27b2f9b28)에서 수정됨

**Cyfrin:** 확인함.


### `AccountableOpenTerm::_mintFeeShares`에서 `IFeeManager`를 다시 만들지 말고 `fm`을 재사용할 것

**설명:** `AccountableOpenTerm::_mintFeeShares`에서는 treasury 주소를 가져오기 위해 `feeManager`를 다시 캐스팅합니다.

```solidity
address treasury_ = IFeeManager(feeManager).treasury();
```

하지만 `IFeeManager fm` 인터페이스는 이미 함수 인자로 전달되므로, 이 추가 cast/load는 불필요합니다.

```solidity
address treasury_ = fm.treasury();
```

**Accountable:** 커밋 [`5e13285`](http://github.com/Accountable-Protocol/credit-vaults-internal/commit/5e132854c6ce3230350daf12feea77bb4a7e8586)에서 수정됨

**Cyfrin:** 확인함.


### `_accrueFeeShares`에서 debt를 다시 계산하지 말고 `aum_`을 재사용할 것

**설명:** `AccountableOpenTerm::_accrueFeeShares`에서는 debt를 다음과 같이 다시 계산합니다.

```solidity
uint256 debt = _loan.outstandingPrincipal.mulDiv(scaleFactor_, PRECISION);
```

하지만 같은 debt/AUM 값은 `_accrueInterest()`에서 이미 계산되어 `aum_`으로 전달됩니다. 따라서 이 곱셈/나눗셈은 중복이며, `scaleFactor_` 파라미터도 불필요해질 수 있습니다.

예를 들어 `_accrueFeeShares`에서 `uint256 debt = aum_;`처럼 `aum_`을 직접 쓰고, 함수 시그니처와 호출부에서 `scaleFactor_` 파라미터를 제거하면 가스를 절약하고 코드도 단순화할 수 있습니다.

**Accountable:** 커밋 [`f350a8d`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/f350a8dc1769f9401b4d1ef62d3748540545ca4d)에서 수정됨

**Cyfrin:** 확인함.


### `AccountableYield::repay`에서 `remaining > 0`일 때만 `deployedAssets`를 갱신할 것

**설명:** `AccountableYield::repay`는 `deployedAssets`를 읽고 조건 없이 줄입니다.

```solidity
uint256 deployed = deployedAssets;
deployedAssets -= Math.min(remaining, deployed);
```

하지만 `remaining == 0`이면 `Math.min(0, deployed) == 0`이므로 아무 효과가 없습니다. 실제로 바로 아래에서 outstanding principal reduction도 `remaining > 0`일 때만 수행하고 있습니다.
```solidity
// Reduce outstanding principal
if (remaining > 0) {
     uint256 outstanding = _loan.outstandingPrincipal;
    _loan.outstandingPrincipal = outstanding > remaining ? outstanding - remaining : 0;
}
```

따라서 불필요한 storage read/write를 피하려면, `deployedAssets` 감소도 같은 `if (remaining > 0)` 블록 안으로 옮기는 것을 고려하십시오.
```solidity
// Reduce deployedAssets and outstanding principal
if (remaining > 0) {
    // Assets are moving from external → vault
    uint256 deployed = deployedAssets;
    deployedAssets -= Math.min(remaining, deployed);

     uint256 outstanding = _loan.outstandingPrincipal;
    _loan.outstandingPrincipal = outstanding > remaining ? outstanding - remaining : 0;
}
```

**Accountable:** 커밋 [`eec49ac`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/eec49ac44951a81228d0cd759b429f0ba13c5772)에서 수정됨

**Cyfrin:** 확인함.


### `createYieldStrategy`의 중복 zero address 검사를 제거하는 것을 고려할 것

**설명:** `createYieldStrategy`는 `Create2` 라이브러리의 `deploy` 함수를 사용해 `AccountableYield` 전략 인스턴스를 배포합니다. 배포 후에는 `strategyProxy`가 `address(0)`이 아닌지도 다시 확인합니다.
```solidity
strategyProxy = Create2.deploy(0, params.salt, strategyProxyBytecode);
if (strategyProxy == address(0)) revert FailedDeployment(ZERO_LOAN_PROXY_ADDRESS);
```

하지만 이는 필요하지 않습니다. `deploy` 함수 자체가 이미 이 상황을 검사하고 더 이른 시점에 revert하기 때문입니다.

```solidity
function deploy(uint256 amount, bytes32 salt, bytes memory bytecode) internal returns (address addr) {
        if (address(this).balance < amount) {
            revert Create2InsufficientBalance(address(this).balance, amount);
        }
        if (bytecode.length == 0) {
            revert Create2EmptyBytecode();
        }
        /// @solidity memory-safe-assembly
        assembly {
            addr := create2(amount, add(bytecode, 0x20), mload(bytecode), salt)
        }
        if (addr == address(0)) {
            revert Create2FailedDeployment();
        }
    }
```

**권장 완화 조치:** zero address 검사를 제거하는 것을 고려하십시오.

```diff
- if (strategyProxy == address(0)) revert FailedDeployment(ZERO_LOAN_PROXY_ADDRESS);
```

**Accountable:** 커밋 [`ec3b6b9`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/ec3b6b9d9e8c4bcaae7b913c1f00a2c6a11a4636)에서 수정됨

**Cyfrin:** 확인함. 이 최적화는 Open- 및 FixedTerm factory에도 함께 적용되었습니다.


### 이벤트를 더 일찍 발생시켜 가스를 절약하는 것을 고려할 것

**설명:** `DVNPublisherFactory.setImplementation`은 불필요한 메모리 변수 생성과 접근을 피하기 위해, `implementation` 상태 변경 전에 `ImplementationSet` 이벤트를 먼저 발생시킬 수 있습니다.

[`DVNPublisherFactory.sol#L34`](https://github.com/Accountable-Protocol/credit-vaults-internal/blob/ba1c7754f891dd6d28a4b47d1989c8b03073abe2/src/factory/DVNPublisherFactory.sol#L34)
```solidity
function setImplementation(address implementation_) external onlyOwner {
        address oldImplementation = implementation;
        implementation = implementation_;
        emit ImplementationSet(oldImplementation, implementation_);
    }
```

비슷한 사례는 `DVNPublisher`와 `AccountableYield`에도 있습니다.

[`DVNPublisher.sol#L125`](https://github.com/Accountable-Protocol/credit-vaults-internal/blob/ba1c7754f891dd6d28a4b47d1989c8b03073abe2/src/publisher/DVNPublisher.sol#L125)
```solidity
function setThreshold(uint256 threshold_) external onlyManager {
        if (threshold_ > MAX_THRESHOLD || threshold_ == 0) revert InvalidThreshold();

        uint256 oldThreshold = threshold;
        threshold = threshold_;

        emit ThresholdSet(oldThreshold, threshold_);
    }
```

[`DVNPublisher.sol#L153`](https://github.com/Accountable-Protocol/credit-vaults-internal/blob/ba1c7754f891dd6d28a4b47d1989c8b03073abe2/src/publisher/DVNPublisher.sol#L153)

```solidity
function setMaxStaleness(uint256 maxStaleness_) external onlyManager {
        if (maxStaleness_ == 0) revert ZeroValue();

        uint256 oldMaxStaleness = maxStaleness;
        maxStaleness = maxStaleness_;

        emit MaxStalenessSet(oldMaxStaleness, maxStaleness_);
    }
```

[`DVNPublisher.sol#L163`](https://github.com/Accountable-Protocol/credit-vaults-internal/blob/ba1c7754f891dd6d28a4b47d1989c8b03073abe2/src/publisher/DVNPublisher.sol#L163)
```solidity
    /// @inheritdoc IDVNPublisher
    function setMaxDeviation(uint256 maxDeviation_) external onlyManager {
        uint256 oldMaxDeviation = maxDeviation;
        maxDeviation = maxDeviation_;

        emit MaxDeviationSet(oldMaxDeviation, maxDeviation_);
    }
```

[`AccountableYield.sol#L226`](https://github.com/Accountable-Protocol/credit-vaults-internal/blob/ba1c7754f891dd6d28a4b47d1989c8b03073abe2/src/strategies/AccountableYield.sol#L226)
```solidity
function setNavGracePeriod(uint256 period) external onlyManager {
        if (period < MIN_NAV_GRACE_PERIOD) revert Unauthorized();

        uint256 oldPeriod = navGracePeriod;
        navGracePeriod = period;

        emit NavGracePeriodSet(oldPeriod, period);
    }
```

**권장 완화 조치:** 상태 업데이트 전에 먼저 이벤트를 발생시키는 것을 고려하십시오.

**Accountable:** 커밋 [`1a7ce24`](https://github.com/Accountable-Protocol/credit-vaults-internal/commit/1a7ce24cd0a84ed56386c8061778480640ab8364)에서 수정됨

**Cyfrin:** 확인함.

\clearpage
