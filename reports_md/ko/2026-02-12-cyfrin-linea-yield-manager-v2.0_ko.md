**Lead Auditors**

[Dacian](https://x.com/DevDacian)

[Stalin](https://x.com/0xStalin)
**Assisting Auditors**

[ChainDefenders](https://x.com/DefendersAudits) ([1337web3](https://x.com/1337web3), [PeterSRWeb3](https://x.com/PeterSRWeb3))


---

# 발견 사항 (Findings)
## 높은 위험 (High Risk)


### 양의 수익과 충분한 `stVault` 유동성이 감지되었을 때 `lstLiabilities` 상환이 실행되면 `userFunds` 회계 무결성이 깨져 양의 수익 보고와 부채 및 의무 상환 핵심 기능이 손상됨

**설명:** 시스템은 양의 수익이 감지되고 `stVault`에 유동성이 있을 때 모든 부채와 의무를 상환하려고 시도합니다. 그러나 `YieldManager::reportYield` 실행 흐름에서 `lstLiabilities`를 상환해도 `userFunds`가 수정되지 않아, Linea 사용자에게 실제로 지급되어야 하는 자금이 정확히 반영되지 않습니다.

문제의 핵심은 [`userFunds`가 "Linea 사용자에게 지급되어야 하는 자금"](https://github.com/Consensys/linea-monorepo/issues/12#issuecomment-3494448425)을 의미하는데도, `LineaRollup`에 네이티브 잔액이 부족하면 사용자는 `stETH`(lstToken) 형태로 자금을 받을 수 있다는 점입니다. 따라서 `lstLiabilities`를 상환하면서도 `userFunds`를 줄이지 않으면, 실제 사용자에게 남은 지급 의무와 `userFunds`가 서로 어긋나게 됩니다.

```
- T0: `stVault`에 200 ETH가 들어 있고, 그 100%가 BeaconChain에 스테이킹되어 있습니다.
회계 상태:
  dashboard.totalValue() => 200 ETH
  stVault.availableBalance() => 0 ETH (모두 스테이킹됨)
  userFunds => 200 ETH
  lstLiabilityPrincipal => 0 ETH
  부채나 수수료 없음

    ↓

- T1: 사용자 출금 150 ETH를 처리하기 위해 LST가 민팅됩니다.
    ↓
누군가 호출: `LineaRollupYieldExtension::claimMessageWithProofAndWithdrawLST`
    ↓
호출: `YieldManager::withdrawLST`
    ↓
delegatecall: `LidoStVaultYieldProvider::withdrawLST`
    ↓
호출: `Dashboard::mintStETH` - 출금자에게 150 ETH 상당의 stETH를 민팅
    ↓
- VaultHub 기준 Vault의 liabilities가 증가하고, YieldProvider의 `lstLiabilityPrincipal`도 증가합니다.
    ↓
출금 처리와 stETH 민팅 이후의 회계 상태:
  dashboard.totalValue() => 200 ETH
  stVault.availableBalance() => 0 ETH (모두 스테이킹됨)
  userFunds => 200 ETH
  lstLiabilityPrincipal => 150 ETH
  부채나 수수료 없음

    ↓

- T2: `lstLiabilities`가 커졌기 때문에 이를 상환하기 위해 BeaconChain에서 출금을 요청합니다.
    ↓
누군가 호출: `YieldManager.unstakePermissionless(yieldProvider, payload)`
    ↓
delegatecall: `LidoStVaultYieldProvider.unstakePermissionless()`
    ↓
검증: `_validateUnstakePermissionlessRequest()` - Merkle proof로 beacon state를 확인
    ↓
제출: `_unstake() → Dashboard.requestWithdrawals() → EIP-7002 precompile`
    ↓
Beacon chain: validator exit 대기열에 들어감
    ↓
⏳ 2~7일 대기
    ↓
Beacon chain: ETH를 `stVault`로 자동 전송
    ↓
BeaconChain 출금 대기 기간이 끝나고 `stVault`가 스테이킹 ETH 200 ETH를 모두 돌려받은 뒤의 회계 상태:
  dashboard.totalValue() => 200 ETH
  stVault.availableBalance() => 200 ETH
  userFunds => 200 ETH
  lstLiabilityPrincipal => 150 ETH
  부채나 수수료 없음

    ↓

- T3: 수익을 보고합니다. 이전 단계들에서는 수익/이자를 반영하지 않았지만, 여기서는 최소 1 wei의 수익이 있다고 가정해 양의 수익 경로로 실행됩니다.
    ↓
호출: `YieldManager::reportYield`
    ↓
delegatecall: `LidoStVaultYieldProvider::reportYield`
    ↓
-> 양의 수익이 감지되어 내부적으로 `LidoStVaultYieldProvider._payMaximumPossibleLSTLiability()`를 호출
  -> `StakingVault` 잔액은 200 ETH이고 liabilities는 150 ETH이므로:
  ->    ↓
  호출: `Dashboard::rebalanceVaultWithShares` - 150 ETH 상당의 share를 재조정
    -> `VAULT_HUB.rebalance()`를 호출해 150 ETH 상당의 share를 재조정
      -> 회계를 갱신해 150 ETH 상당의 share 상환을 반영하고, 외부 호출로 `stVault::withdraw`를 실행
      ->   ↓
        -> `stVault::withdraw`를 150 ETH에 대해 호출
          -> 150 ETH가 Vault에서 VaultHub로 출금됨
      -> 이후 `VaultHub.rebalance()` 실행은 `stVault`에서 출금되기 전 지점으로 돌아가 계속되고, 방금 인출한 ETH를 Lido(CorePool/LidoV2)에 다시 예치하며 회계를 재조정
  -> 그 후 실행은 `Dashboard::rebalanceVaultWithShares` 호출 전의 `LidoStVaultYieldProvider::reportYield`로 복귀합니다. 이 예시에서는 의무나 수수료가 없으므로 YieldProvider 내부 처리가 끝나고 다시 `YieldManager::reportYield`로 돌아옵니다. 상환한 `lstLiabilities`가 새로 발생한 수익보다 더 크기 때문에 positive yield는 0입니다.
- YieldManager 입장에서는 양의 수익이 생성되지 않았으므로 `userFunds`, `yieldReportedCumulative`, `userFundsInYieldProvidersTotal`은 변경되지 않습니다.
    ↓
수익 보고 후 회계 상태 - 150 ETH의 `lstLiability`가 상환됨
  dashboard.totalValue() => 50 ETH
  stVault.availableBalance() => 50 ETH
  userFunds => 200 ETH
  lstLiabilityPrincipal => 0 ETH
  부채나 수수료 없음

    ↓

- T4: Vault에서 50 ETH를 출금 요청합니다.
    ↓
호출: `YieldManager::withdrawFromYieldProvider`
    ↓
delegatecall: `LidoStVaultYieldProvider::withdrawFromYieldProvider`
    ↓
호출: `Dashboard.withdraw()` - 요청된 50 ETH를 Vault에서 출금
    ↓
실행은 다시 `YieldManager::withdrawFromYieldProvider`로 돌아옵니다.
- 출금된 50 ETH만큼 `userFunds`와 `userFundsInYieldProvidersTotal`이 감소합니다.
    ↓
출금 처리 후 회계 상태:
  dashboard.totalValue() => 0 ETH
  stVault.availableBalance() => 0 ETH
  userFunds => 150 ETH
  lstLiabilityPrincipal => 0 ETH
  부채나 수수료 없음

최종 결과는 Vault가 비어 있는데도 `userFunds`가 150 ETH로 남는다는 것입니다. 겉으로 보면 Linea 사용자에게 150 ETH 부족분이 생긴 것처럼 보이지만, 실제로는 Linea 사용자들이 T1 시점에 이미 150 ETH 상당을 stETH 형태로 인출했고, 나머지 50 ETH는 T4 시점에 추가 출금을 처리하기 위해 `LineaMessage`로 보내졌습니다.
```

**영향:** `userFunds` 회계 변수의 무결성이 깨집니다. 주요 영향은 수익 보고 메커니즘 붕괴입니다. 실제 vault의 `totalValue`에 비해 `userFunds`가 부풀려져 양의 수익을 감지하지 못하게 되기 때문입니다. 부수적으로는 프로토콜이 양의 수익이 없다고 판단하므로, 그 양의 수익으로 시스템 의무를 지불하는 것도 더 이상 이뤄지지 않습니다.

**권장 완화 조치:** 회계 계층을 리팩터링해 출금 처리를 통합하고, stETH 기반 출금도 네이티브 ETH 출금과 동일하게 취급해야 합니다. 출금 확정 시 lst 발행 시점에 `userFunds`와 `userFundsInYieldProvidersTotal`을 선제적으로 감소시켜야 합니다.

**Linea:** 커밋 [40b3328](https://github.com/Consensys/linea-monorepo/tree/40b3328912d17a745182e58d84e21d65cdc31589) 기준으로 수정됨.

**Cyfrin:** 확인함. `userFunds`와 `lstLiabilities`의 의미가 재정의되었습니다. 이제 코드는 `userFunds`를 Linea 사용자에게 실제로 지급되어야 하는 자금으로 올바르게 추적하도록 리팩터링되었습니다.
- 사용자 출금을 처리하기 위해 stETH가 mint될 때마다, mint된 stETH의 ETH 가치만큼 `userFunds`에서 차감되어 그 자금이 더 이상 Linea 사용자에게 지급될 필요가 없음을 반영합니다.
- 또한 YieldManager는 이제 `lstPrincipal`만이 아니라 `lstLiabilities`(원금과 이자)를 가능한 한 많이 상환하려고 시도합니다.

lst가 원래 사용자 출금을 처리하기 위해 mint된 시점에 그 가치가 이미 `userFunds`에서 차감되므로, 이후 `lstLiabilities` 상환 시에는 해당 ETH를 다시 `userFunds`에서 차감할 필요가 없습니다.

\clearpage
## 중간 위험 (Medium Risk)


### `YieldManager::withdrawLST`가 오래된 `lstLiabilityPrincipal`을 사용해 음수 리베이스 시 일시적 DoS를 유발할 수 있음

**설명:** Lido의 `stETH`는 리베이스 토큰이며, 슬래싱 등으로 인해 양의 리베이스와 음의 리베이스가 모두 발생할 수 있습니다.

`LidoStVaultYieldProvider::withdrawLST`는:
* 입력으로 토큰 수량인 `_amount`를 받습니다.
* LidoV3의 `Dashboard::mintStETH`를 호출해 토큰 `_amount`만큼 mint를 시도합니다.
* 여기서 `STETH::getSharesByPooledEth`로 shares를 구하고, 실제로는 그 shares가 mint됩니다.
* 그런데 `$$.lstLiabilityPrincipal += _amount;`로 입력 토큰 수량 `_amount`를 그대로 부채로 저장합니다. 이 값은 이후 `stETH` 리베이스에 따라 증가하거나 감소할 수 있습니다.

코드의 다른 함수들은 `$$.lstLiabilityPrincipal`을 사용하기 전에 `_syncExternalLiabilitySettlement`를 호출해 동기화합니다.

하지만 "진입" 함수인 `YieldManager::withdrawLST`에서는 동기화가 일어나지 않습니다.
```solidity
YieldProviderStorage storage $$ = _getYieldProviderStorage(_yieldProvider);
// @audit `$$.lstLiabilityPrincipal` used without being sync'd
if ($$.lstLiabilityPrincipal + _amount > $$.userFunds) {
  revert LSTWithdrawalExceedsYieldProviderFunds();
}
```

이 시점의 `$$.lstLiabilityPrincipal`은 양/음 리베이스 여부에 따라 더 이상 정확하지 않을 수 있습니다.

**영향:** 양의 리베이스 상황에서는 Lido 시스템이 vault에 예치된 ETH와 담보화 파라미터로 overminting을 막기 때문에 실제 영향은 없습니다. 그러나 음의 리베이스에서는 다음과 같은 일시적 DoS가 발생할 수 있습니다.
```
State: lstLiabilityPrincipal = 100 ETH (stale)
Reality: Dashboard.liabilityShares worth 90 ETH (-10% slashing)
Request: withdrawLST(50 ETH) with userFunds = 140 ETH

Stale Check: 100 + 50 = 150 > 140 ✗ REVERTS
Real Check: 90 + 50 = 140 ≤ 140 ✓ Should pass

Result: LST withdrawal blocked during reserve deficit
```

이 일시적 DoS는 이후 어떤 작업이든 동기화를 트리거하는 연산이 한 번 실행되면 해소됩니다.

**권장 완화 조치:** `YieldManager::withdrawLST`에서 `$$.lstLiabilityPrincipal`을 사용하기 전에, 다른 모든 경로와 동일하게 먼저 동기화하십시오.

**Linea:** [PR 1703](https://github.com/Consensys/linea-monorepo/pull/1703)에 속한 커밋 [2b99f9bf](https://github.com/Consensys/linea-monorepo/pull/1703/commits/2b99f9bf08aaecf0c28cb399064b821f42340e32) 및 [5cbe6b5](https://github.com/Consensys/linea-monorepo/pull/1703/commits/5cbe6b5e71d9425f2c22f4ea9e33fc85c71936cd)에서 수정됨.

**Cyfrin:** 확인함. [`userFunds` 의미 변경 PR](https://github.com/Consensys/linea-monorepo/pull/1703)로 인해 `YieldManager::withdrawLST`에서 더 이상 `$$.lstLiabilityPrincipal`을 검사에 사용하지 않게 되었으므로 동기화가 필요 없어졌습니다.


### `YieldManager::_getTotalSystemBalance`에서 음수 수익을 전혀 반영하지 않아 일시적 DoS가 발생할 수 있음

**설명:** `YieldManager::reportYield`는 provider들로부터 `outstandingNegativeYield`를 받지만, 이를 이벤트로만 남기고 `$$.userFunds`는 줄이지 않습니다. 이는 의도된 동작입니다(`$$.userFunds`는 자산 추적기가 아니라 부채 원장임). 문제는 음수 수익이 다른 어디에도 반영되지 않아 `_getTotalSystemBalance`가 부풀려진 값을 반환한다는 점입니다.
```solidity
function _getTotalSystemBalance() internal view returns (...) {
    // @audit not adjusted for negative yield
    totalSystemBalance = L1_MESSAGE_SERVICE.balance +
                         address(this).balance +
                         $.userFundsInYieldProvidersTotal;
}
```

이 부풀려진 값은 이후 준비금 임계값 계산에 사용되어, 잘못된 운영 판단으로 이어집니다.

**영향:** 슬래싱 이후 `_getTotalSystemBalance`가 부풀려지면 준비금 임계값이 잘못 계산되어 다음과 같은 문제가 생깁니다.

**일시적 DoS(실제로는 가능해야 하는 작업이 잘못 차단됨):**
- `receiveFundsFromReserve` - 실제로 준비금이 충분해도 자금을 받을 수 없음
- `fundYieldProvider` - 자금을 다시 스테이킹할 수 없음
- `unpauseStaking` - 스테이킹을 재개할 수 없음

**잘못된 실행(실제로는 필요 없거나 허용되면 안 되는 작업이 허용됨):**
- `unstakePermissionless` - 준비금이 실제로 충분해도 언스테이킹 허용
- `replenishWithdrawalReserve` - 불필요한 준비금 보충 시도

**기타 영향:**
- `getTargetReserveDeficit`가 부풀려진 deficit를 반환해 downstream 계산에 영향을 줌
- `withdrawLST`의 체크가 더 보수적으로 동작할 수 있음(다만 Lido가 실제 과도한 출금을 막아 줌)

**예시:** 100 ETH 슬래싱 후 준비금 요구 비율이 10%일 때:
- 계산된 최소치: 100 ETH(부풀려진 1000 ETH 기준)
- 실제 최소치: 90 ETH(실제 900 ETH 기준)
- 준비금이 95 ETH라면: 수익이 회복될 때까지 작업이 잘못 차단됨

이 영향은 양의 수익이 누적 회계를 정상화할 때까지 일시적으로 지속됩니다.

**권장 완화 조치:** 개별 `_yieldProvider`의 `outstandingNegativeYield`를 `YieldProviderStorage`에 저장하고, 이를 다음 위치에서 사용하십시오.
* `YieldManager::withdrawLST`의 `LSTWithdrawalExceedsYieldProviderFunds` 체크
* `YieldManager::withdrawableValue`가 보다 정확한 값을 반환하도록

보다 어려운 지점은 `YieldManager::_getTotalSystemBalance`가 이상적으로는 모든 yield provider의 총 음수 수익을 반영해야 정확한 값을 줄 수 있다는 것입니다. 그러나 이를 온체인에서 효율적으로 구현하기는 어렵고, 실제로는 구현하지 않는 편이 더 안전할 수 있습니다.

**Linea:** 커밋 [4fd227](https://github.com/Consensys/linea-monorepo/commit/4fd227022a84bf8c8ae981980d3600dd5baecb42)에서 수정됨

**Cyfrin:** 확인함. 각 Yield Provider별 Outstanding Negative Yield를 저장하고, 이를 stETH mint를 통한 사용자 출금 처리 시 한도를 조이는 데 사용합니다. 총 시스템 잔액에는 음수 수익을 반영하지 않는데, 비동기적으로 부채와 의무를 상환할 수 있는 구조에서 오래된 데이터를 사용하면 오히려 DoS를 유발할 수 있기 때문입니다.


### `YieldManager::fundYieldProvider`와 `LidoStVaultYieldProvider::fundYieldProvider`가 `isStakingPaused` 및 `isOssificationInitiated`를 강제하지 않아 안전하지 않은 스테이킹을 허용함

**설명:** `YieldManager::fundYieldProvider`와 `LidoStVaultYieldProvider::fundYieldProvider`는 `isStakingPaused`와 `isOssificationInitiated` 플래그를 확인하지 않으므로, 중단되어야 하는 상황에서도 새 자금을 스테이킹할 수 있습니다. 프로토콜 [사양](https://hackmd.io/@kyzooroast/HkAKIXS6ex#Pausing-Beacon-Chain-Deposits)은 다음과 같이 말합니다. *"When ossification has been initiated or completed, liabilities are incurred, or withdrawal deficits exist, new validator deposits must be paused to protect user funds."*

Yield Provider는 다음 두 방법으로 pause될 수 있습니다.
* `YieldManager::pauseStaking`를 명시적으로 호출
* `YieldManager::withdrawLST`로 LST 부채가 발생하거나 `YieldManager::initiateOssification`로 ossification이 시작될 때 `_pauseStakingIfNotAlready`가 호출되어 해당 `_yieldProvider`의 `isStakingPaused = true` 설정

`LidoStVaultYieldProvider::fundYieldProvider`는 `isOssified` 플래그가 `true`일 경우 revert하지만, 이 함수와 `YieldManager::fundYieldProvider` 모두 `isStakingPaused`가 `true`인지에 대해서는 전혀 revert하지 않습니다.

**영향:** 명시적으로 스테이킹이 pause된 yield provider에도 새 자금을 스테이킹할 수 있어 안전하지 않은 스테이킹 작업이 가능해집니다.

**권장 완화 조치:** 이미 ossification 체크가 있는 `LidoStVaultYieldProvider::fundYieldProvider`에 `isStakingPaused`와 `isOssificationInitiated` 체크도 추가하십시오.

**Linea:** 커밋 [becfd756](https://github.com/Consensys/linea-monorepo/commit/becfd75633a2d46de9fa1d9b02554aef9be1cbde)에서 수정됨.

**Cyfrin:** 확인함. 이제 `LidoStVaultYieldProvider::fundYieldProvider`는 스테이킹이 pause되었거나, ossification이 시작되었거나 완료된 경우 revert합니다.


### 부채와 의무 정산에서 우선 상환 최적화가 부족해 시스템에 미지급 음수 수익이 누적됨

**설명:** `lstLiabilities`에 대한 이자와 운영 비용(의무)은 스테이킹된 ETH에서 발생한 수익으로 우선 정산되도록 설계되어 있습니다. 모든 채무(부채와 의무 포함)가 전부 정산된 뒤에만 남는 수익이 양의 수익으로 Linea(L2)에 보고되어야 합니다.

부채와 의무의 지급은 수익 보고와 함께 동작하도록 설계되어 있으며, 양의 수익 델타가 감지될 때만 트리거됩니다.

잘못된 우선순위 문제는 vault 잔액이 충분해야만 정산이 가능한 구조에서 비롯됩니다. vault에 자금이 없으면 부채 정산이 미뤄지고, 그 결과 미지급 음수 수익이 누적됩니다.

예를 들어 `stVault` 내 `ETH` 100%가 스테이킹되어 있는 상황을 보겠습니다.
```
- T0:
dashboard.totalValue = 100 ETH
userFunds = 100 ETH
stVault.availableBalance = 0 ETH
의무도 liabilities도 없음

- T1 - 1 ETH의 수익이 발생했고, 0.1 ETH의 의무도 생겼습니다.
dashboard.totalValue = 101 ETH
userFunds = 100 ETH
stVault.availableBalance = 0 ETH
obligations (fees/liabilities) = 0.1 ETH
- 이 시점에는 1 ETH의 양의 수익이 감지되지만, `stVault`에 잔액이 없어 실제 지급은 할 수 없습니다. 즉 출금 자체가 불가능합니다.
- 그 결과 이 1 ETH는 L2에 양의 수익으로 보고되고, 0.1 ETH의 의무는 그대로 남습니다.

- T2 - BeaconChain에서 1 ETH의 부분 출금이 완료되었고, 의무와 수익이 각각 0.1 ETH씩 추가로 반영됩니다.
dashboard.totalValue = 101.1 ETH
userFunds = 101 ETH
stVault.availableBalance = 1 ETH
obligations (fees/liabilities) = 0.2 ETH

- T3 - 수익을 보고하며, 추가로 1 ETH의 수익과 의무가 발생합니다.
dashboard.totalValue = 102.1 ETH
userFunds = 101 ETH
stVault.availableBalance = 1 ETH
obligations (fees/liabilities) = 1.1 ETH

이번에는 의무 1 ETH가 상환되지만, 0.1 ETH는 여전히 남습니다.

핵심 문제는 T1 시점에 실제로는 먼저 갚아야 할 미지급 의무가 있었는데도 1 ETH를 양의 수익으로 보고했다는 점입니다. 즉 의무 지급이 우선되어야 했지만, 실제 구현에서는 수익 분배가 먼저 일어났습니다.

```

**영향:** 시스템에 음수 수익이 누적되며 다음과 같은 결과를 낳습니다.

1. **스테이킹 ETH의 실효 수익 감소:** `stETH` 리베이스 비율이 누적 스테이킹 이자를 초과하면 실제 수익이 줄어듭니다. 부채 정산이 지연되면 `lstLiabilities` 이자가 계속 복리로 쌓여 부족분이 더 커지기 때문입니다.

2. **다른 문제의 악화:** 음수 수익이 적절한 추적 없이 누적되어, 관련 이슈에서 지적한 결함들을 더 심하게 만듭니다.

**권장 완화 조치:** 양의 수익이 존재하는지 판단하는 시점에, vault 잔액이 충분하다고 낙관적으로 가정하는 대신 의무와 부채를 함께 고려해야 합니다. 이렇게 하면 "양의 수익"은 vault에 잔액이 생길 때(예: beacon chain에서 생성된 수익 인출이 완료될 때) 부채/의무 상환 재원으로 예약될 수 있습니다.

핵심은 미지급 항목이 남아 있는 동안에는 양의 수익을 L2에 보고하지 말고, 그 수익을 먼저 채무 상환에 우선 배정해야 한다는 것입니다. 모든 채무가 정산된 뒤에만 생성된 수익을 L2에 보고해야 합니다.

코드 차원에서는 대략 다음과 같은 수정이 가능합니다.
- `ALL_OBLIGATIONS`는 liabilities, obligations, node fee를 포함한다고 가정
```diff
  function reportYield(
    address _yieldProvider
  ) external onlyDelegateCall returns (uint256 newReportedYield, uint256 outstandingNegativeYield) {
    ...
    uint256 lastUserFunds = $$.userFunds;
    uint256 totalVaultFunds = _getDashboard($$).totalValue();
    // Gross positive yield
-    if (totalVaultFunds > lastUserFunds) {
+    if (totalVaultFunds > lastUserFunds + ALL_OBLIGATIONS ) {
      ...
      // Gross negative yield
    } else {
      newReportedYield = 0;
      outstandingNegativeYield = lastUserFunds - totalVaultFunds;
    }
  }

```

**Linea:** 커밋 [0e46ee](https://github.com/Consensys/linea-monorepo/commit/0e46ee6efe6ed79526f2a2ed55c1ca82f7e0e663)에서 수정됨.

**Cyfrin:** 확인함. 이제 기초 `stVault`가 보유한 총 가치가 모든 부채, 의무, 수수료를 초과할 때만 양의 수익을 보고합니다. 양의 수익이 없으면 `outstandingNegativeYield`를 시스템에 누적하며, `reportYield`가 실행될 때마다 부채, 의무, 수수료 지급을 시도합니다.


### LST 출금과 상환이 `userFundsInYieldProvidersTotal`을 부풀려 이후 핵심 프로토콜 함수에 DoS를 유발함

**설명:** `YieldManager::userFundsInYieldProvidersTotal`은 yield provider에 예치된 사용자 자금의 총량을 저장하며, 사용자가 출금할 때 결국 상환해야 하는 부채입니다. 그런데 다음 상황에서:

* `YieldProvider::withdrawLST`를 통해 LST가 출금되면 사용자에게 상환해야 할 부채는 줄어드는데, 이를 반영해 `userFundsInYieldProvidersTotal`이나 `userFunds`를 줄이지 않습니다(관련 이슈 참고).

* `YieldManager::reportYield`가 호출되고 그 안에서 `LidoStVaultYieldProvider::_payMaximumPossibleLSTLiability`가 실행되어 LST 부채가 상환되더라도, `stVault`의 ETH가 LST 부채 상환에 사용되는데도 `userFundsInYieldProvidersTotal`은 줄어들지 않습니다.

**영향:** LST가 출금되고 LST 부채가 상환될수록 `YieldManager::userFundsInYieldProvidersTotal`은 시간이 지날수록 부풀려집니다. 그 결과:
* `YieldManager::_getTotalSystemBalance`가 총 시스템 잔액 계산에 `userFundsInYieldProvidersTotal`을 사용하므로 부풀려진 값을 반환합니다.
* `YieldManager::isWithdrawalReserveBelowMinimum`는 결국 `_getTotalSystemBalance`를 호출하는데, 그 반환값이 부풀려져 있으면 하드 최소값보다 퍼센트 기반 계산이 더 큰 한, 최소 준비금 요구량이 실제보다 높게 계산됩니다.

결과적으로 `isWithdrawalReserveBelowMinimum`가 실제로는 `false`여야 하는데도 `true`를 반환할 수 있습니다. 이는 부풀려진 `userFundsInYieldProvidersTotal`에 기반하기 때문이며, 이 함수를 호출하는 프로토콜 함수들이 불필요하게 revert하여 DoS가 발생합니다.

**권장 완화 조치:** LST를 출금하고 LST 부채를 상환할 때 `userFundsInYieldProvidersTotal`이 정확히 유지되어야 합니다. 가장 간단한 방법은 `YieldManager::withdrawLST`에서 `userFundsInYieldProvidersTotal`을 감소시키는 것입니다.

**Linea:** 커밋 [40b3328](https://github.com/Consensys/linea-monorepo/tree/40b3328912d17a745182e58d84e21d65cdc31589) 기준으로 수정됨.

**Cyfrin:** 확인함. 이제 `YieldManager::withdrawLST`는 해당 자금이 더 이상 Linea 사용자에게 지급되어야 하지 않음을 반영하기 위해 `userFundsInYieldProvidersTotal`에서도 차감하며, 동시에 Lido에 대한 부채가 생성되었음을 나타내기 위해 `lstLiabilityPrincipal`을 증가시킵니다.


### `YieldManager::unpauseStaking`가 오래된 `lstLiabilityPrincipal`을 사용해 외부 행위자의 LST 부채 상환 시 DoS를 유발함

**설명:** `YieldManager::unpauseStaking`에는 다음 검사 코드가 있습니다.
```solidity
if ($$.lstLiabilityPrincipal > 0) {
  revert UnpauseStakingForbiddenWithCurrentLSTLiability();
}
```

하지만 코드의 다른 위치와 달리, 사용 전에 `LidoStVaultYieldProvider::_syncExternalLiabilitySettlement`로 `$$.lstLiabilityPrincipal`을 동기화하지 않습니다. 즉 오래된 값을 사용합니다.

**영향:** LST 부채는 외부 주체가 정산할 수 있습니다. 외부 주체가 vault의 LST 부채를 정산한 뒤에도 `YieldManager::unpauseStaking`는 아직 부채가 남아 있다고 잘못 판단해 revert하며, 그 결과 서비스 거부가 발생합니다.

**권장 완화 조치:** `YieldManager::unpauseStaking`에서 `$$.lstLiabilityPrincipal`을 사용하기 전에 `LidoStVaultYieldProvider::_syncExternalLiabilitySettlement`로 먼저 동기화해야 합니다.

**Linea:** 커밋 [0c17a98](https://github.com/Consensys/linea-monorepo/commit/0c17a98aaca91a81d6fe081c23c125ad7bea5c01#diff-5859d4fa4970e34f899c9068e747565347ba390a46e54ead17419d5c81df83b4R882-R886)에서 수정됨.

**Cyfrin:** 확인함.


### 수익 보고 전에 ossification과 yield provider 제거가 먼저 일어나면 외부 LST 부채 정산분이 프로토콜에서 누락됨

**설명:** 외부 행위자가 LST 부채를 정산하면 일종의 windfall이 발생합니다(vault에는 `userFunds`보다 많은 ETH가 존재하게 됨). 하지만 프로토콜의 회계는 이를 즉시 반영하지 않습니다. 다음에 `YieldManager::reportYield`가 호출되면, 이 외부 LST 부채 정산으로 생긴 windfall이 인식되어 L2에 배분할 수익으로 보고됩니다.

그러나 `YieldManager::reportYield`가 호출되지 않은 상태에서 다음이 먼저 일어나면 이 windfall은 영구히 사라질 수 있습니다.
1. ossification이 시작되고 완료됨
2. `YieldManager::withdrawFromYieldProvider` 호출 후 `YieldManager::removeYieldProvider` 호출
* `withdrawFromYieldProvider`는 외부 정산을 감지하기 위해 `lstLiabilityPrincipal`을 올바르게 동기화하지만, 그 windfall을 `userFunds`에 반영하지는 않습니다.
* `YieldManager::removeYieldProvider`는 제거를 허용하기 전에 `userFunds == 0`만 검사하고 vault의 실제 잔액이 `userFunds`와 일치하는지는 확인하지 않습니다.

**영향:** L2 사용자에게 분배될 수 있었던 "windfall" 프로토콜 자산이 영구적으로 손실됩니다.
* 프로토콜의 내부 회계(`userFunds`, `userFundsInYieldProvidersTotal`)가 물리적인 현실(`vaultBalance`)과 영구적으로 분리되며, 이를 다시 맞출 수단이 없습니다.
* 이후 vault 소유권이 새로운 소유자에게 이전됩니다.

**권장 완화 조치:** 1) `YieldManager::removeYieldProvider`는 vault의 실제 가치도 추가로 검사하고 0이 아니면 revert해야 합니다.
```solidity
    uint256 actualVaultValue;
    if ($$.isOssified) {
        actualVaultValue = IStakingVault($$.ossifiedEntrypoint).availableBalance();
    } else {
        actualVaultValue = IDashboard($$.primaryEntrypoint).totalValue();
    }

    if (actualVaultValue > 0) {
        revert VaultHasUnreportedValue(actualVaultValue);
    }
```

2) 현재 `YieldManager::initiateOssification`는 `LidoStVaultYieldProvider::initiateOssification`를 호출하고, 여기서 `_payMaximumPossibleLSTLiability`가 실행됩니다. 외부 행위자가 LST 부채를 정산한 경우 `dashboard.liabilityShares()`는 0을 반환하므로 `if` 분기가 실행되지 않습니다.
```solidity
  function _payMaximumPossibleLSTLiability(
    YieldProviderStorage storage $$
  ) internal returns (uint256 liabilityPaidETH) {
    if ($$.isOssified) return 0;
    IDashboard dashboard = IDashboard($$.primaryEntrypoint);
    address vault = $$.ossifiedEntrypoint;
    uint256 rebalanceShares = Math256.min(
      dashboard.liabilityShares(), // @audit returns 0 when LST liability externally settled
      STETH.getSharesByPooledEth(IStakingVault(vault).availableBalance())
    );
    if (rebalanceShares > 0) {
      // @audit code never executes when LST liability externally settled
      //
      // Cheaper lookup for before-after compare than availableBalance()
      uint256 vaultBalanceBeforeRebalance = vault.balance;
      dashboard.rebalanceVaultWithShares(rebalanceShares);
      // Apply consistent accounting treatment that LST interest paid first, then LST principal
      _syncExternalLiabilitySettlement($$, dashboard.liabilityShares(), $$.lstLiabilityPrincipal);
      liabilityPaidETH = vaultBalanceBeforeRebalance - vault.balance;
    }
  }
```

가능한 수정안 중 하나는 `LidoStVaultYieldProvider::_payMaximumPossibleLSTLiability`가 항상 LST 부채를 동기화하도록 만드는 것입니다. 현재처럼 `dashboard.liabilityShares() == 0`일 때 동기화를 건너뛰는 전략은, 외부에서 LST 부채가 상환된 경우를 처리하지 못하므로 잘못된 것으로 보입니다.

**Linea:** `_payMaximumPossibleLSTLiability`가 항상 `_syncExternalLiabilitySettlement`를 호출하도록 커밋 [f45bdfc](https://github.com/Consensys/linea-monorepo/commit/f45bdfc74a90704f4f6320b1dc6683f2b1517338)에서 수정됨.

`YieldManager::removeYieldProvider`가 실제 잔액이 0이 아니면 revert하도록 하자는 제안은, ETH를 vault에 보내서 DoS를 유발할 수 있으므로 구현하지 않았습니다.

또한 `LidoStVaultYieldProvider:exitVendorContracts`가 vendor exit data가 없으면 revert하도록 하는 커밋 [55fe25a](https://github.com/Consensys/linea-monorepo/commit/55fe25aef79d9ec76b0cf8441933045b2bf86ee5)도 추가되었습니다.

**Cyfrin:** 확인함.

\clearpage
## 낮은 위험 (Low Risk)


### `L1MessageService` 잔액이 정확히 필요한 값과 같을 때 LST 출금을 revert하지 않음

**설명:** `LineaRollupYieldExtension::claimMessageWithProofAndWithdrawLST` 함수는 `L1MessageService` 잔액이 메시지 전달을 충족하기에 부족할 때만 yield provider에서 LST 토큰을 출금하도록 설계되었습니다. 문서에도 "`L1MessageService`가 메시지 전달을 충족할 만큼 충분한 잔액을 가지고 있으면 revert해야 한다"고 적혀 있습니다. 그러나 실제 잔액 검사는 less-than-or-equal-to(`<=`)가 아니라 strict less-than(`<`)를 사용합니다.

```solidity
if (_params.value < address(this).balance) {
  revert LSTWithdrawalRequiresDeficit();
}
```

따라서 `_params.value`가 `address(this).balance`와 정확히 같은 경우, 조건식은 false가 되어 함수가 계속 실행되고, 이미 충분한 잔액이 있음에도 yield provider에서 LST를 출금하게 됩니다.

**영향:** 컨트랙트 잔액이 청구 금액과 정확히 일치하는 경우에도 잘못해서 yield provider에서 LST를 출금하게 되며, 그 결과:

1. 이미 자금이 충분한데도 불필요한 LST 출금이 발생함

2. 호출자 가스가 낭비됨

3. "실제 잔액 부족이 있을 때만 LST 출금이 일어나야 한다"는 명시된 불변식이 깨짐

4. 수익 관리 시스템의 운영 효율이 떨어질 수 있음

**권장 완화 조치:** 비교 연산자를 `<`에서 `<=`로 바꿔, 잔액이 충분한 경우(정확히 같은 경우 포함)에도 revert되도록 하십시오.

```solidity
if (_params.value <= address(this).balance) {
  revert LSTWithdrawalRequiresDeficit();
}
```

이렇게 하면 실제 부족분이 있을 때(`_params.value > address(this).balance`)에만 LST 출금이 일어나게 됩니다.

**Linea:** 커밋 [9722a2a](https://github.com/Consensys/linea-monorepo/commit/9722a2a048c9e0c0f0dd8bf959a37516e7590b8d)에서 수정됨.

**Cyfrin:** 확인함.


### `LidoStVaultYieldProvider::reportYield`에서 `_payNodeOperatorFees`가 revert하면 수익 회계가 잘못됨

**설명:** `LidoStVaultYieldProvider::reportYield`가 `_payNodeOperatorFees`를 호출할 때, `_payNodeOperatorFees`가 revert하면 `nodeOperatorFees` 값은 사실상 `0`으로 취급됩니다.

그럼에도 상위 함수인 `YieldManager::reportYield`는 실패한 수수료 지급을 반영하지 않은 채 수익 처리를 계속합니다. 그 결과 수익이 잘못 보고되고, 운영자 수수료가 차감되지 않았음에도 `userFunds`는 **인위적으로 부풀려진 상태**가 됩니다.

 **영향:**

`_payNodeOperatorFees`가 revert하면 `$$.userFunds`가 과대계상되어 사용자 잔액 회계가 부정확해지고, 수익을 과대평가할 수 있습니다.
또한 운영자 수수료를 지불할 자금이 부족한 상황에서도 revert가 일어나지 않아, 일관성 없는 상태 갱신이 유지될 수 있습니다.

**권장 완화 조치:** `_payNodeOperatorFees`가 실패하더라도 원래 의도된 `nodeOperatorFees` 값을 보존하고, 이후 출금이나 수익 조정 같은 작업에서 이를 반영하십시오.

**Linea:** [PR 1703](https://github.com/Consensys/linea-monorepo/pull/1703/files#diff-c8d16d3a1aee9686a5fec0c0d8b96ea370258f3ecaa6b4faae091ccae151a9c2)에 속한 커밋 [0e46ee](https://github.com/Consensys/linea-monorepo/pull/1703/commits/0e46ee6efe6ed79526f2a2ed55c1ca82f7e0e663)에서 수정됨.

**Cyfrin:** 확인함. 이제 기초 `stVault`가 보유한 총 가치가 모든 부채, 의무, 수수료를 초과할 때만 양의 수익을 보고하며, `reportYield`가 실행될 때마다 부채, 의무, 수수료 지급을 시도합니다.


### `YieldManager::replenishWithdrawalReserve` 함수의 위험

**설명:** `replenishWithdrawalReserve` 함수는 사용 가능한 유동성을 이용해 출금 준비금을 목표 임계치까지 보충하도록 설계되었습니다. 주요 특징은 다음과 같습니다.

* **Permissionless Access:** 누구나 이 함수를 호출할 수 있습니다.
* **Yield Provider Selection:** 어떤 yield provider를 사용할지는 호출자가 지정합니다.
* **Reserve Logic:**

  1. 출금 준비금이 유효 최소 임계값 아래인지 확인합니다.
  2. 먼저 YieldManager 자신의 잔액으로 부족분을 메우려 시도합니다.
  3. 부족하면 지정된 yield provider에서 인출합니다.
  4. 그래도 deficit가 남아 있으면 추가 출금을 막기 위해 스테이킹을 pause합니다.

잠재적 위험:
함수가 임의의 `_yieldProvider` 선택을 허용하므로, 악의적이거나 부주의한 호출자가 부채가 더 많거나 유동성이 낮은 provider를 선택할 수 있습니다. 이는 다음을 초래할 수 있습니다.

* 비효율적인 자금 사용
* 선택된 provider가 출금을 감당하지 못해 보충 실패
* 불충분한 보충으로 인해 의도치 않은 스테이킹 pause

**영향:**
1. **유동성 위험:** 최적이 아닌 yield provider를 사용하면 중요한 유동성이 고갈되거나 준비금 임계값을 충족하지 못할 수 있습니다.

2. **운영상 위험:** 불필요한 스테이킹 pause가 발생할 수 있습니다.

**권장 완화 조치:** 위험을 줄이기 위해 다음을 고려하십시오.

1. **접근 제한:**

   * 이 함수를 신뢰된 역할(예: `ADMIN` 또는 `TREASURY_MANAGER`)만 호출할 수 있도록 제한합니다.

2. **Provider 선택 최적화:**

   * 사용자 입력에 의존하지 말고, 내부적으로 유동성 대비 부채 비율이 가장 유리한 yield provider를 선택합니다.
   * 또는 점수화 메커니즘을 도입해 가장 적합한 provider를 자동 선택합니다.

**Linea:** 비즈니스 요구사항상, 권한 있는 역할이 제때 행동하지 못할 경우 누구나 `L1MessageService` 잔액을 보충할 수 있어야 하므로 이 함수는 permissionless여야 한다는 입장입니다.


### `YieldManager`로 들어오는 새 예치금이 LST 부채를 상환하지 않아 프로토콜의 이자 부채가 더 커짐

**설명:** [기술 명세](https://hackmd.io/@kyzooroast/HkAKIXS6ex#LST-Liability-Accounting)는 다음과 같이 설명합니다.
> `liabilityPrincipal` is effectively an advance to the user on the staked reserve funds. Hence it is paid down from:
> * New deposits into the YieldManager
> * Withdrawals from a Yield Provider
> * Earned yield

마지막 두 항목은 구현되어 있지만, 첫 번째는 `YieldManager::receiveFundsFromReserve`와 `receive` 구현에서 볼 수 있듯 아직 구현되지 않았습니다.
```solidity
  function receiveFundsFromReserve() external payable {
    if (msg.sender != L1_MESSAGE_SERVICE) {
      revert SenderNotL1MessageService();
    }
    if (isWithdrawalReserveBelowMinimum()) {
      revert InsufficientWithdrawalReserve();
    }
    emit ReserveFundsReceived(msg.value);
  }

  receive() external payable {}
```

**영향:** 새 예치금으로 LST 부채를 상환하지 않으므로, 프로토콜이 더 큰 이자 부채를 누적하게 됩니다.

**권장 완화 조치:** 어려운 점은 LST 부채가 yield provider별로 관리되는 반면, 들어오는 ETH 예치금에는 yield provider 컨텍스트가 없다는 것입니다. 기존 인터페이스를 유지하면서 이를 온체인에서 구현하려면, LST 부채를 가진 yield provider 목록/집합을 추적하고 `receiveFundsFromReserve`가 이를 순회하며 가능한 한 많이 부채를 상환하도록 해야 합니다.

또는 사양에서 이 요구사항 자체를 제거하도록 수정하는 것도 한 방법입니다.

**Linea:** LST 부채는 양의 부채를 가진 yield provider에 자금이 공급될 때 상환되며, `YieldManager` 자체가 자금을 받는다고 바로 상환되지는 않습니다. 조금 덜 최적이긴 하지만 코드 단순성이 훨씬 좋아지므로, 이 점을 명세에 반영하겠다는 입장입니다.


### 모든 permissionless 출금 함수가 pause되면 사용자가 자금을 인출할 수 없어 프로토콜 명세와 충돌함

**설명:** 프로토콜 기술 명세는 다음과 같이 말합니다.
> users can [always eventually withdraw](https://hackmd.io/@kyzooroast/HkAKIXS6ex#Censorship-Resistant-User-Withdrawals) their funds; **no actor can permanently block or prevent users from withdrawing their assets**

그러나 모든 permissionless 출금 경로는 권한 있는 행위자에 의해 pause될 수 있습니다. `YieldManager::withdrawLST`, `replenishWithdrawalReserve`, `unstakePermissionless` 모두 `whenTypeAndGeneralNotPaused(PauseType.NATIVE_YIELD_PERMISSIONLESS_ACTIONS)` modifier를 사용합니다.

**영향:** 권한 있는 행위자가 `PauseType.NATIVE_YIELD_PERMISSIONLESS_ACTIONS`를 pause한 뒤 영구히 해제하지 않으면, 사용자는 자산을 영구적으로 인출하지 못할 수 있습니다. 이는 "어떤 행위자도 사용자의 출금을 영구히 막을 수 없다"는 명세를 위반합니다.

**권장 완화 조치:** 기술 명세를 따르려면 permissionless action을 pause할 수 없게 해야 합니다. 그렇지 않다면 명세의 해당 문구를 수정해야 합니다.

**Linea:** `L1MessageService::claimMessageWithProof` 함수 쪽 permissionless 출금에도 무기한 pause가 존재할 수 있다는 점을 들어, 현재 설계 취지를 유지하겠다는 입장입니다.


### 출금 준비금이 deficit 상태여도 beacon chain 예치가 계속 일어날 수 있어 안전 속성을 위반함

**설명:** 프로토콜 기술 명세는 [다음](https://hackmd.io/@kyzooroast/HkAKIXS6ex#Safety-Properties)을 요구합니다.
> Beacon Chain Deposit Restriction: Beacon chain deposits MUST be paused when:
>
> * Withdrawal reserve is in deficit, or
> * Outstanding stETH liabilities exist, or
> * Ossification has been initiated or completed

대체로는 구현되어 있지만, 다음과 같은 엣지 케이스가 존재합니다.

* 일반적인 L2→L1 출금 흐름으로 인해 출금 준비금이 자연스럽게 deficit 상태에 빠짐
* pause되지 않은 모든 yield operator에 대해 `YieldManager::_pauseStakingIfNotAlready`를 호출하는 작업이 트리거되지 않음
* 이전 자금 공급으로 인해 하나 이상의 Lido `stVault`에 이미 ETH가 존재함
* Lido의 DEPOSITOR 역할이 실제 예치를 수행함

**영향:** 출금 준비금이 deficit 상태일 때도 beacon chain 예치가 계속 일어나, 프로토콜의 안전 기준과 충돌합니다.

**권장 완화 조치:** 이를 완화하는 유일한 방법은 다음과 같습니다.
* 오프체인에서 출금 준비금 상태를 모니터링
* deficit 상태로 떨어지면 모든 yield provider에 대해 `YieldManager::pauseStaking` 호출
* deficit 상태가 해소되면 모든 yield provider에 대해 `YieldManager::unpauseStaking` 호출

하지만 이 완화책은 복잡도를 크게 늘리며, 그만한 가치가 있는지는 의문입니다. 이 시나리오를 허용하도록 프로토콜 안전 기준 자체를 조정하는 편이 나을 수 있습니다.

**Linea:** 이를 처리하기 위한 오프체인 Native Yield Automation Service를 둘 계획이라고 응답함.

\clearpage
## 정보성 (Informational)


### todo 주석 제거

**설명:** 다음 "todo" 주석을 제거하십시오.
```solidity
LineaRollup.sol
38:  // TODO - Add access control to proxy admin only
```

**Linea:** 커밋 [d8f57d5](https://github.com/Consensys/linea-monorepo/commit/d8f57d5d7f6abf043502a3e2e9c04b40b8a2df19)에서 수정됨.

**Cyfrin:** 확인함.


### `ErrorUtils::revertIfZeroAddress`를 일관되게 사용

**설명:** 일부 코드(예: `YieldProverBase::constructor`)는 입력 주소가 0 주소가 아닌지 검증하기 위해 `ErrorUtils::revertIfZeroAddress`를 사용합니다. 하지만 다른 부분은 이를 사용하지 않고 동일한 체크를 다시 구현하고 있습니다. 다음 위치에서도 일관되게 `ErrorUtils::revertIfZeroAddress`를 사용하십시오.
* `LineaRollup::initialize, reinitializeLineaRollupV7` - yield manager 검증
* `YieldManager::initialize` - `defaultAdmin` 검증

**Linea:** 커밋 [b4b8ef5](https://github.com/Consensys/linea-monorepo/commit/b4b8ef57cbb870d9601a3a6e1d5b725be91de8c8)에서 수정됨.

**Cyfrin:** 확인함.


### `LineaRollupYieldExtension`의 불필요한 `FUNDER_ROLE`

**설명:** `LineaRollupYieldExtension` 컨트랙트는 `FUNDER_ROLE` 상수를 정의하고 주석으로 "The role required to call fund()"라고 설명합니다. 그러나 실제 `fund` 함수에는 접근 제어가 없어 누구나 호출할 수 있습니다.

```solidity
function fund() external payable virtual {
  if (msg.value == 0) revert NoEthSent();
  emit FundingReceived(msg.value);
}
```

이 함수에는 `onlyRole(FUNDER_ROLE)` modifier가 없으므로, 이 역할 상수는 실질적으로 쓸모없고 개발자, 감사자, 사용자에게 오해를 줄 수 있습니다.

**영향:** `FUNDER_ROLE` 정의 자체가 곧바로 보안 취약점을 만드는 것은 아니지만, 문서와 실제 구현이 어긋나 있어 다음과 같은 혼란을 초래할 수 있습니다.

1. 코드베이스를 유지보수하는 개발자가 접근 제어가 있다고 잘못 가정할 수 있음

2. 이후 개발자가 `FUNDER_ROLE`을 부여/회수하면 `fund` 접근이 제어될 것이라 착각한 채 구현을 진행해 또 다른 실수를 만들 수 있음

컨트랙트의 다른 주석에 따르면 `fund` 함수는 "accept both permissionless donations and YieldManager withdrawals"를 의도적으로 지원하도록 설계된 것으로 보입니다. 그렇다면 접근 제어가 없는 현재 구현은 의도된 것이지만, `FUNDER_ROLE` 상수와 그 설명은 이 의도와 모순됩니다.

**권장 완화 조치:** `fund` 함수가 의도적으로 permissionless라면 사용되지 않는 `FUNDER_ROLE` 상수와 관련 주석을 제거하십시오. 이렇게 하면 문서와 구현 간 불일치를 없앨 수 있습니다.

반대로 `fund` 함수에 접근 제어가 필요하다면 `onlyRole(FUNDER_ROLE)` modifier를 추가할 수 있습니다. 다만 이는 permissionless donation을 허용하겠다는 현재 의도와 충돌합니다.

**Linea:** 커밋 [9876bc5](https://github.com/Consensys/linea-monorepo/commit/9876bc50dec4e614ad103d336361592a36a007bc)에서 수정됨.

**Cyfrin:** 확인함.


### `YieldManagerStorageLayout`의 storage location namespace root 불일치

**설명:** `YieldManagerStorageLayout`에서 `YieldManagerStorage`는 `@custom:storage-location erc7201:linea.storage.YieldManager`로 주석 처리되어 있지만, 컨트랙트가 실제로 사용하는 하드코딩 namespace root는 `"linea.storage.YieldManagerStorage"`라는 다른 식별자를 기준으로 문서화 및 선택되어 있습니다. ERC-7201에 따르면 annotation에 쓰인 namespace id는 storage root를 계산할 때 사용한 id와 정확히 같아야 하며, 그렇지 않으면 annotation이 실제 storage schema를 설명하지 못합니다.

**영향:** annotation을 읽고 `erc7201("linea.storage.YieldManager")`를 계산하는 오프체인 툴은 잘못된 storage root를 조회하게 되어, 상태 디코딩, 분석, 모니터링이 틀어질 수 있습니다. 표준 비준수는 감사와 업그레이드 과정에서 혼란을 초래하고, 다른 컴포넌트가 annotation에 적힌 id를 문자 그대로 따를 경우 충돌이나 불일치 위험도 높입니다.

**권장 완화 조치:** 다음 둘 중 하나를 선택하고, annotation, 주석, 상수를 완전히 일치시키십시오.

1. Option A(최소 변경): annotation을 slot의 문서화/계산된 id에 맞게 수정

 - 변경값: `@custom:storage-location erc7201:linea.storage.YieldManagerStorage`

2. Option B(현재 annotation 유지): `"linea.storage.YieldManager"`에 대한 ERC-7201 공식을 사용해 상수와 문서 주석을 다시 계산하고, `YieldManagerStorageLocation`도 그 값으로 갱신

- 공식: `keccak256(abi.encode(uint256(keccak256(bytes("linea.storage.YieldManager"))) - 1)) & ~bytes32(uint256(0xff))`

- 주석과 16진수 값이 이 id와 정확히 일치하도록 유지

**Linea:** 커밋 [f3e11c0](https://github.com/Consensys/linea-monorepo/commit/f3e11c0dc8a887eb3a377d2ec14d7366809fbeb8)에서 수정됨.

**Cyfrin:** 확인함.


### `lstLiabilityPrincipal` 동기화 시 이벤트 발행 고려

**설명:** `LidoStVaultYieldProvider::_syncExternalLiabilitySettlement`는 양/음의 `stETH` 리베이스로 인해 변할 수 있는 `$$.lstLiabilityPrincipal`을 동기화합니다.

하지만 이 함수는 `$$.lstLiabilityPrincipal`을 변경하면서도 아무 이벤트도 발생시키지 않습니다. 오프체인 추적과 감사 편의성을 위해 최소한 변화량(delta)을 담은 이벤트를 발행하는 것을 고려하십시오.

**Linea:** 커밋 [9f32daf](https://github.com/Consensys/linea-monorepo/commit/9f32daf25bf053bd099314bd1db974415a98c7fe)에서 수정됨.

**Cyfrin:** 확인함.


### 중복된 체크를 modifier로 리팩터링

**설명:** 다음 중복 체크를 modifier로 리팩터링하십시오.
* `YieldManager.sol`:
```solidity
// duplicated in `receiveFundsFromReserve` and `withdrawLST`
if (msg.sender != L1_MESSAGE_SERVICE) {
  revert SenderNotL1MessageService();
}

// duplicated in `receiveFundsFromReserve`, `fundYieldProvider`, `unpauseStaking`
if (isWithdrawalReserveBelowMinimum()) {
  revert InsufficientWithdrawalReserve();
}

// duplicated in `unstakePermissionless` and `replenishWithdrawalReserve`
if (!isWithdrawalReserveBelowMinimum()) {
  revert WithdrawalReserveNotInDeficit();
}

// duplicated in `initiateOssification` and `progressPendingOssification`
if ($$.isOssified) {
  revert AlreadyOssified();
}
```

**Linea:** 커밋 [e832420](https://github.com/Consensys/linea-monorepo/commit/e832420b45c5513d8d19adc74e0db4492917c2d50)에서 앞의 세 제안을 구현함.

**Cyfrin:** 확인함.


### LST 출금으로 시스템 잔액이 최소 준비금 요구치 아래로 떨어질 수 있음

**설명:** 핵심 프로토콜 [속성](https://hackmd.io/@kyzooroast/HkAKIXS6ex#Minimum-Withdrawal-Reserve) 중 하나는 다음과 같습니다. "YieldManager가 관여하는 모든 ETH 전송 작업은 출금 준비금이 최소 임계치 이상(≥)으로 유지되도록 해야 한다."

그러나 `LineaRollupYieldExtension::claimMessageWithProofAndWithdrawLST`와 `YieldManager::withdrawLST` 어느 쪽도 `YieldManager::isWithdrawalReserveBelowMinimum`를 호출해 LST 출금이 최소 준비금 요구를 깨지 않는지 확인하지 않습니다.

그리고 설령 이 호출이 있더라도, `YieldManager::_getTotalSystemBalance`는:
* LST 출금이 결국 시스템 밖으로 나가는 부채라는 점에도 불구하고 총 시스템 잔액 계산 시 미상환 LST 부채를 차감하지 않습니다.
* 별도 이슈에서 지적한 것처럼, LST 부채 상환 시 `userFundsInYieldProvidersTotal`이 손상되는 문제의 영향을 받습니다.

**영향:** LST 출금이 최소 준비금 요구를 위반해, 프로토콜이 요구량보다 적은 준비금만 보유하게 될 수 있습니다.

**권장 완화 조치:** * `YieldManager::withdrawLST`가 모든 작업을 마친 뒤 `isWithdrawalReserveBelowMinimum`를 호출해 LST 출금이 최소 준비금 요구를 깨지 않았는지 검증해야 합니다.
* `YieldManager::_getTotalSystemBalance`는 총 시스템 잔액 계산 시 미상환 LST 부채를 차감해야 합니다.
* LST 부채 상환 시 `userFundsInYieldProvidersTotal`이 손상되는 문제도 해결되어야 합니다(별도 이슈).

**Linea:** 기능 관점에서 `LineaRollupYieldExtension::claimMessageWithProofAndWithdrawLST` → `YieldManager::withdrawLST`는 준비금이 완전히 고갈되었을 때 사용자가 LST로라도 출금할 수 있게 하는 비상 경로라고 설명했습니다.

따라서 이 흐름은 "YieldManager가 관여하는 모든 ETH 전송 작업은 출금 준비금이 최소 임계치 이상이어야 한다"는 기술 명세의 예외로 의도된 것입니다.

아래 두 항목은 다른 이슈의 완화 조치로 해결될 예정이라고 덧붙였습니다.
> * `YieldManager::_getTotalSystemBalance` should deduct outstanding LST liabilities when calculating the total system balance
> * The corruption of `userFundsInYieldProvidersTotal` when LST liabilities are repaid should be fixed (separate issue).


### 사용자 원금이 permissionless하게 시스템 의무를 상환하는 데 사용될 수 있어 안전 속성을 위반함

**설명:** 프로토콜 기술 명세는 다음과 같이 말합니다.
> User principal (L1 deposits + L2 circulating ETH) MUST [NOT be used](https://hackmd.io/@kyzooroast/HkAKIXS6ex#Safety-Properties) for system obligations; all obligations MUST be settled exclusively from [unreported yield](https://hackmd.io/@kyzooroast/HkAKIXS6ex#Unreported-Yield) which ensures that users funds are [insulated](https://hackmd.io/@kyzooroast/HkAKIXS6ex#LST-Liability-Accounting) from ongoing system obligations

하지만 permissionless 행위자는 `VaultHub::settleLidoFees` 같은 LidoV3 함수를 프로토콜의 `stVault`에 대해 직접 호출할 수 있고, 이 과정에서 예치된 ETH가 시스템 의무 지불에 사용될 수 있습니다.

**영향:** 이 안전 속성을 위반하는 것 외에는, 프로토콜이 다음과 같이 동작하기 때문에 추가적인 직접 영향은 크지 않아 보입니다.
* 사용자에게 지급해야 하는 총 예치 ETH `userFunds`와 vault별 예치 ETH `userFundsInYieldProvidersTotal`을 모두 추적함
* 외부 부채 정산으로 발생한 ETH 유출을 음수 수익으로 취급하고, 이는 결국 스테이킹 수익으로 상쇄됨

**Linea:** 인지함.


### `YieldManager::fundYieldProvider`는 자금 공급 후 스테이킹을 자동으로 unpause하지 않음

**설명:** `YieldManager::fundYieldProvider`에서 자금을 yield provider로 다시 옮겨 잔액을 복구하거나 부채를 상환한 뒤에도, 모든 안전 조건(예: 충분한 출금 준비금, 정리된 LST 부채, 완료된 ossification)이 충족되었다면 스테이킹을 재개할 수 있음에도 **자동으로 unpause하지 않습니다**.

그 결과 yield provider가 완전히 정상화된 뒤에도 **pause 상태에 갇힌 채** 남을 수 있습니다. 이는 스테이킹 운영의 불필요한 다운타임을 만들고, 프로토콜 효율이나 수익 생성에도 악영향을 줄 수 있습니다.

**권장 완화 조치:** `fundYieldProvider()`(또는 내부 `_fundYieldProvider` 헬퍼)를 개선해, 자금 공급 후 provider가 조건을 충족하면 **자동으로 unpause를 시도**하도록 하십시오.

**Linea:** Linea 사용자 유동성을 수익 효율보다 우선하기 때문에, 준비금이 부족할 때는 빠르게 pause하고 재개는 신중하게 하겠다는 입장입니다. unpause 타이밍은 Native Yield Automation Service가 출금 준비금 상태를 보고 처리할 예정입니다.

\clearpage
## 가스 최적화 (Gas Optimization)


### 입력 관련 체크를 먼저 수행해 빠르게 실패하기

**설명:** 함수 호출이 잘못된 입력 때문에 revert될 예정이라면, 그 전에 다른 작업을 수행할 이유가 없습니다. 따라서 입력 관련 체크를 먼저 수행해 빠르게 실패하도록 하십시오.
* `LineaRollup::initialize, reinitializeLineaRollupV7` - 유효한 yield manager 주소 검사를 가장 먼저 수행. 코드 중복을 줄이기 위해 modifier를 만들어 두 함수에 적용하는 것도 고려하십시오.

**Linea:** 커밋 [b4b8ef5](https://github.com/Consensys/linea-monorepo/commit/b4b8ef57cbb870d9601a3a6e1d5b725be91de8c8)에서 수정됨.

**Cyfrin:** 확인함.


### `LidoStVaultYieldProvider::_syncExternalLiabilitySettlement`에서 `liabilityETH` 제거 리팩터링

**설명:** `LidoStVaultYieldProvider::_syncExternalLiabilitySettlement`의 로컬 변수 `liabilityETH`는 이름이 있는 반환 변수를 사용해 제거할 수 있습니다.
```solidity
  function _syncExternalLiabilitySettlement(
    YieldProviderStorage storage $$,
    uint256 _liabilityShares,
    uint256 _lstLiabilityPrincipalCached
  ) internal returns (uint256 lstLiabilityPrincipalSynced) {
    lstLiabilityPrincipalSynced = STETH.getPooledEthBySharesRoundUp(_liabilityShares);
    // If true, this means an external actor settled liabilities.
    if (lstLiabilityPrincipalSynced < _lstLiabilityPrincipalCached) {
      $$.lstLiabilityPrincipal = lstLiabilityPrincipalSynced;
    } else {
      lstLiabilityPrincipalSynced = _lstLiabilityPrincipalCached;
    }
  }
```

**Linea:** 커밋 [5b55950](https://github.com/Consensys/linea-monorepo/commit/5b5595062d4c4f37ceb8e23fc737e34239f7b232)에서 수정됨.

**Cyfrin:** 확인함.
