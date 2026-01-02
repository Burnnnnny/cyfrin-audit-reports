**Lead Auditors**

[Immeas](https://twitter.com/0ximmeas)

[MrPotatoMagic](https://x.com/MrPotatoMagic)

---

# 발견 사항 (Findings)
## 중간 위험 (Medium Risk)


### 소유자가 볼트(Vault) 자체 공유 토큰을 구조(rescue)할 수 있음

**설명:** [`SherpaVault::rescueTokens`](https://github.com/hedgemonyxyz/sherpa-vault-smartcontracts/blob/50eb8ad6ee048a767f7ed2265404c59592c098b7/contracts/SherpaVault.sol#L730-L740)는 래퍼 토큰(wrapper token) 구조를 금지합니다:
```solidity
// CRITICAL: Cannot rescue the wrapper token (user funds)
// This protects deposited SherpaUSD from being withdrawn by owner
if (token == stableWrapper) revert CannotRescueWrapperToken();
```

하지만 볼트 자체의 공유 토큰(`token == address(vault)`)을 구조하는 것은 허용합니다. 대기 중인 입금에 대해 [새로 발행된 공유](https://github.com/hedgemonyxyz/sherpa-vault-smartcontracts/blob/50eb8ad6ee048a767f7ed2265404c59592c098b7/contracts/SherpaVault.sol#L543-L544)는 `address(this)`의 볼트 보관소에 보관되며 사용자 상환은 이 잔액에서 전송되므로:
```solidity
accountingSupply += mintShares;
_mint(address(this), mintShares);
```

소유자는 `rescueTokens`를 통해 보관된 공유를 외부로 전송하여, 사용자의 대기/상환 잔액을 뒷받침하는 볼트 보유 풀을 줄일 수 있습니다.

**영향:** 소유자(또는 탈취된 소유자 키)는 볼트가 보관하는 공유를 `address(this)`에서 이동시켜, 이를 통해 기본 예치금을 인출할 수 있습니다.

**권장 완화 조치:** 볼트 자체 공유 토큰의 구조를 금지하십시오:

```diff
- if (token == stableWrapper) revert CannotRescueWrapperToken();
+ if (token == stableWrapper || token == address(this)) revert CannotRescueWrapperToken();
```

**Sherpa:** 커밋 [`1a634e0`](https://github.com/hedgemonyxyz/sherpa-vault-smartcontracts/commit/1a634e0331968ea5a73f38a62ef824da9376ab52)에서 수정됨

**Cyfrin:** 확인함. 볼트 토큰도 이제 구조가 방지됩니다.


### 소유자가 동일 블록 자금 유출(drains)을 위해 관리자 호출을 연결(chain)할 수 있음

**설명:** 프로토콜의 관리자 제어를 통해 소유자는 단일 트랜잭션에서 볼트와 래퍼에 걸쳐 권한이 있는 호출을 연결할 수 있습니다:

* **볼트 경로:** `SherpaVault::setStableWrapper`를 호출하여 구조로부터 보호되는 토큰을 전환합니다. 그런 다음 즉시 `SherpaVault::rescueTokens`를 호출하여 이전 래퍼의 잔액을 볼트에서 인출합니다.
* **래퍼 운영자(Operator) 경로:** `SherpaUSD::setOperator`를 호출한 다음, (운영자로서) `SherpaUSD::transferAsset`을 사용하여 USDC를 래퍼 외부로 이동합니다.
* **래퍼 키퍼(Keeper) 경로:** `SherpaUSD::setKeeper`를 호출한 다음, `SherpaUSD::depositToVault`를 사용하여 승인을 남겨둔 사용자의 USDC를 가져와 키퍼에게 SherpaUSD를 발행하고, 위의 `transferAsset` 경로를 통해 가치를 추출합니다.

이 모든 것은 소유자 전용이며 기본 지연(delay)이 없으므로 동일한 블록에서 함께 실행될 수 있습니다.

**영향:** 코드 주석에서 소유자 권한 제한을 강조하더라도, 소유자(또는 탈취된 키)는 사용자 경고나 대응 시간 없이 즉시 보관소를 리디렉션하고 자금을 이동할 수 있습니다. 이는 명시된 의도와 실제 권한 사이에 신뢰 격차를 만듭니다.

**권장 완화 조치:** * `SherpaVault.setStableWrapper`, `SherpaVault.rescueTokens`, `SherpaUSD.setOperator`, `SherpaUSD.setKeeper`에 지연(최소 한 번의 인출 에포크)을 추가하고, `SherpaUSD.transferAsset` 지연을 고려하십시오.
* `SherpaVault.stableWrapper`, `SherpaUSD.keeper`를 불변(immutable)으로 만드십시오.
* 변경 사항이 적용되기 전에 사람들이 인출하거나 승인을 줄일 수 있도록 사용자 보호 지연이 있는 타임락(예: OpenZeppelin [`TimelockController`](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/governance/TimelockController.sol))을 사용하십시오.

**Sherpa:**
> 볼트 경로: SherpaVault::setStableWrapper를 호출하여 구조로부터 보호되는 토큰을 전환합니다. 그런 다음 즉시 SherpaVault::rescueTokens를 호출하여 이전 래퍼의 잔액을 볼트에서 인출합니다.
> 래퍼 키퍼 경로: SherpaUSD::setKeeper를 호출한 다음, SherpaUSD::depositToVault를 사용하여 승인을 남겨둔 사용자의 USDC를 가져와 키퍼에게 SherpaUSD를 발행하고, 위의 transferAsset 경로를 통해 가치를 추출합니다.

우리는 의사 불변(pseudo-immutable) `stableWrapper`와 `keeper`를 구현하고 있습니다. 둘 다 배포 중에 한 번 설정되며 시스템 초기화 후에는 변경할 수 없습니다. 이는 닭과 달걀의 배포 문제(볼트 생성자는 래퍼 주소가 필요하지만 볼트가 존재할 때까지 래퍼를 배포할 수 없음)를 해결하는 데 필요한 배포 유연성을 유지하면서 두 가지 공격 표면을 모두 제거합니다. 우리는 임시 래퍼 주소로 볼트를 배포한 다음 `setStableWrapper()`를 한 번 호출하여 실제 래퍼를 설정하고 영구적으로 잠그는 방식으로 이 문제를 해결합니다.

> 래퍼 운영자 경로: SherpaUSD::setOperator를 호출한 다음, (운영자로서) SherpaUSD::transferAsset을 사용하여 USDC를 래퍼 외부로 이동합니다.

`setOperator` 및 관련 관리자 기능에 대한 타임락/지연은 우리 볼트의 신뢰 모델과 아키텍처를 고려할 때 효과적이지 않을 것입니다. 운영자는 이미 전략 자금(온체인 및 오프체인 전략 위임을 위해 펀드 매니저에게 전송됨)을 수동으로 보관하고 있으며 마음대로 시스템을 일시 중지할 수 있습니다. 즉, 타임락 창 동안 인출을 단순히 일시 중지하여 타임락 지연을 우회할 수 있습니다. 운영자는 운영 유연성(인사 변경, 키 교체)을 위해 변경 가능해야 하므로 `keeper` 및 `setStableWrapper`처럼 불변으로 만들 수 없습니다. 소유자 역할은 운영자 선택을 제어하는 2-of-3 멀티시그이므로 중앙화는 가능한 한 완화됩니다.

커밋 [`15e2706`](https://github.com/hedgemonyxyz/sherpa-vault-smartcontracts/commit/15e270673d42e02f1e3a08bcba6d1ac61f14010d)에서 수정됨

**Cyfrin:** 확인함. `stableWrapper`와 `keeper` 모두 초기 할당 후 잠기게 되어 사실상 불변이 됩니다. 운영자 관련 우려 사항은 인지했습니다.



### 수익이 발생한 후에는 인출이 사실상 기본 체인에서만 발생할 수 있음

**설명:** 라운드 롤(round rolls) 동안 수익은 `SherpaVault::_adjustBalanceAndEmit`의 기본 체인에서만 실현됩니다. 이는 다른 체인에서 인출이 발생하는 경우 시스템을 문제 있는 상태로 둡니다.

다음 시나리오를 상상해 보십시오: 체인 A와 B에 각각 500개씩 총 1000개의 SherpaUSD 입금이 있고, A가 기본 체인입니다. 100 SherpaUSD가 A에 수익으로 추가됩니다. 잔액은 500 + 600이 되고, 글로벌 총액은 1100이 되어 주당 가격은 1.1이 됩니다.
총 주식의 절반을 가진 Alice가 체인 B에서 인출하기로 결정하여 550 SherpaUSD(USDC)를 받습니다. 체인 B에는 이 금액이 없으므로 프로토콜은 50 SherpaUSD를 A에서 B로 리밸런싱해야 합니다.

그들은 체인 A에서 `SherpaUSD::ownerBurn(50)`을 호출한 다음 체인 B에서 `SherpaUSD::ownerMint(50)`을 호출하여 이를 수행합니다. 이렇게 하면 두 체인 모두 `approvedTotalStakedAdjustment`와 `approvedAccountingAdjustment`에 50이 저장됩니다. 후자가 문제입니다.

운영자가 `SherpaVault::adjustTotalStaked`를 호출하면 SherpaUSD의 리밸런싱이 완료되고 Alice는 사실상 인출할 수 있습니다. 그러나 공유가 이동되지 않았으므로 `approvedAccountingAdjustment`의 상태를 지울 방법이 없습니다. `SherpaVault::adjustAccountingSupply`가 호출되면 공유가 이동되지 않았으므로 `accountingSupply`가 손상됩니다. 따라서 `consumeAccountingApproval`은 볼트에서만 지울 수 있으므로 `approvedAccountingAdjustment`의 상태는 사실상 영구적으로 손상됩니다.

게다가 체인 A에서 `SherpaVault::adjustAccountingSupply`가 호출되면 `accountingSupply`가 감소하고 `_unstake()` 함수의 `accountingSupply` 뺄셈이 체인 A에서 언더플로우되어 자금이 묶일(brick) 수 있습니다.


**영향:** 인출은 수익이 발생하는 즉시 기본 체인에서만 안전하게 발생할 수 있습니다. 보조 체인에서 수익을 인출하면 두 체인 모두에서 `SherpaUSD.approvedAccountingAdjustment` 또는 `SherpaVault.accountingSupply`가 손상됩니다.

**권장 완화 조치:** 분할 승인 모드를 고려하십시오. 명시적인 자산 전용 리밸런싱(`approvedAccountingAdjustment` 설정 없이 `approvedTotalStakedAdjustment` 설정)과 공유 동기화 모드(둘 다 설정)를 도입하십시오.

**Sherpa:** 커밋 [`34f2092`](https://github.com/hedgemonyxyz/sherpa-vault-smartcontracts/commit/34f2092f8f882005304c8f1a2ad311ed91d9161a)에서 수정됨

**Cyfrin:** 확인함. 자산만 리밸런싱하는 호출이 추가되었습니다.

\clearpage
## 낮은 위험 (Low Risk)


### 잘못 구성된 소수점 스케일이 볼트 회계를 왜곡할 수 있음

**설명:** 볼트의 수학은 래핑된 자산(USDC, 6 소수점) 및 운영자가 제공하는 `globalPricePerShare`와 동일한 소수점 스케일을 가정합니다. 배포 시 `vaultParams.decimals = 6`을 설정하고 래퍼가 USDC의 6 소수점을 강제하지만, 잘못된 구성은 변환을 왜곡합니다.

**영향:** 6 소수점 이상으로 볼트를 구성하면 잘못된 회계가 발생하고 리밸런싱에서 후속 되돌리기(revert)가 발생할 수 있습니다.

**권장 완화 조치:** `SherpaUSD`와 마찬가지로 볼트 소수점을 6으로 고정하는 것을 고려하십시오.

**Sherpa:** 커밋 [`1a634e0`](https://github.com/hedgemonyxyz/sherpa-vault-smartcontracts/commit/1a634e0331968ea5a73f38a62ef824da9376ab52)에서 수정됨

**Cyfrin:** 확인함. `_vaultParams.decimals`는 이제 생성자에서 6인지 확인됩니다.


### SherpaUSD는 전송 수수료(fee-on-transfer) 토큰과 함께 작동하지 않음

**설명:** SherpaUSD 컨트랙트는 전송 수수료 토큰과 올바르게 작동할 수 없습니다. 그러한 토큰의 예로는 USDT가 있으며, 주석에 따르면 지원될 것으로 예상됩니다. 참고: USDT에는 아직 수수료가 활성화되지 않았지만 향후 언제든지 활성화될 수 있습니다.

```solidity
        // CRITICAL: SherpaUSD only supports 6-decimal assets (USDC, USDT, etc.)
```

예를 들어:
 - 전송 수수료 토큰에 의해 2%의 수수료가 부과된다고 가정합니다.
 - 키퍼가 100e6 금액으로 depositToVault 함수를 호출합니다.
 - 100 SherpaUSD가 키퍼에게 발행됩니다.
 - 전송 시 부과되는 수수료로 인해 컨트랙트는 98 토큰만 받습니다.
 - 이것이 시간이 지남에 따라 쌓이면 나중에 인출하는 사람들은 토큰을 완전히 인출할 수 없거나 일부만 인출할 수 있어 손실을 입을 수 있습니다.
```solidity
function depositToVault(
    address from,
    uint256 amount
) external nonReentrant onlyKeeper {
    if (amount == 0) revert AmountMustBeGreaterThanZero();

    _mint(keeper, amount);
    depositAmountForEpoch += amount;

    emit DepositToVault(from, amount);

    IERC20(asset).safeTransferFrom(from, address(this), amount);
}
```

**권장 완화 조치:** 전송 수수료 토큰에 대한 지원 추가를 고려하십시오. 또는 그러한 토큰을 지원하지 않는 것을 고려하십시오.

**Sherpa:** 커밋 [`0b32641`](https://github.com/hedgemonyxyz/sherpa-vault-smartcontracts/commit/0b326416fc7312ee11b279964a947a10b642cc2d)에서 수정됨

**Cyfrin:** 확인함. 주석이 FOT 토큰(USDT 포함)이 지원되지 않음을 명시적으로 언급하도록 변경되었습니다.


### `SherpaUSD::ownerMint`/`ownerBurn`에서 직접 금액을 할당하면 totalStaked 및 accountingSupply에 대한 회계가 깨질 수 있음

**설명:** `SherpaUSD::ownerMint` 및 `ownerBurn` 함수는 `approvedTotalStakedAdjustment` 및 `approvedAccountingAdjustment` 매핑에 금액 매개변수를 직접 할당합니다. 그러나 이전 승인이 소비되기 전에 볼트에서 더 많은 토큰이 발행되거나 소각되면 올바르게 작동하지 않습니다.

예를 들어:
 - 운영자가 SherpaVault에 100 SherpaUSD를 발행합니다.
 - 이는 `approvedTotalStakedAdjustment` 및 `approvedAccountingAdjustment`를 각각 100 SherpaUSD로 추적합니다.
 - 운영자는 이전 승인이 소비되기 전에 200 토큰의 또 다른 발행을 수행합니다.
 - 이제 문제는 approvedTotalStakedAdjustment 및 approvedAccountingAdjustment가 300 SherpaUSD 대신 각각 200 SherpaUSD를 저장하도록 덮어쓰여진다는 것입니다.
 - 이것은 명백히 잘못된 것이며 이전 승인이 볼트에 의해 아직 소비되지 않았기 때문에 회계가 깨집니다.

```solidity
function ownerMint(address to, uint256 amount) external onlyOperator {
    _mint(to, amount);

    // Approve vault to adjust by this amount
    approvedTotalStakedAdjustment[to] = amount;
    approvedAccountingAdjustment[to] = amount;

    emit PermissionedMint(to, amount);
    emit RebalanceApprovalSet(to, amount, amount);
}

/**
 * @notice Operator-level burn for manual rebalancing across chains
 * @param from Address to burn from
 * @param amount Amount to burn
 * @dev Sets approval for vault to adjust totalStaked and accountingSupply
 */
function ownerBurn(address from, uint256 amount) external onlyOperator {
    _burn(from, amount);

    // Approve vault to adjust by this amount
    approvedTotalStakedAdjustment[from] = amount;
    approvedAccountingAdjustment[from] = amount;

    emit PermissionedBurn(from, amount);
    emit RebalanceApprovalSet(from, amount, amount);
}
```

**권장 완화 조치:** ownerMint 및 ownerBurn에서 직접 금액 할당을 각각 += 및 -= 연산자로 대체하는 것을 고려하십시오.

**Sherpa:** 커밋 [`1cd0018`](https://github.com/hedgemonyxyz/sherpa-vault-smartcontracts/commit/1cd00183932d086b0fd07d6c34cd3f5aacb2b359)에서 수정됨

**Cyfrin:** 확인함. 승인이 소비되었는지 강제하는 확인이 추가되었습니다. 이는 회계 손상을 방지합니다.

\clearpage
## 정보 (Informational)


### `SherpaVault::_rollInternal` 가격 계산 주석과 수학이 일치하지 않음

**설명:** 새 가격을 계산할 때 스크립트는 모든 체인의 모든 볼트를 쿼리한 다음 이를 `SherpaVault:: rollToNextRound`에 전달합니다. 이는 차례로 [`SherpaVault::_rollInternal`](https://github.com/hedgemonyxyz/sherpa-vault-smartcontracts/blob/50eb8ad6ee048a767f7ed2265404c59592c098b7/contracts/SherpaVault.sol#L518-L529)을 호출하며 여기서 새 가격이 계산됩니다:
```solidity
// Calculate global price using script-provided totals
// globalBalance must include pending deposits for correct price calculation
uint256 globalBalance = isYieldPositive
    ? globalTotalStaked + globalTotalPending + yield
    : globalTotalStaked + globalTotalPending - yield;

uint256 newPricePerShare = ShareMath.pricePerShare(
    globalShareSupply,
    globalBalance,
    globalTotalPending,
    _vaultParams.decimals
);
```
코드 주석에는 `globalBalance must include pending deposits`라고 명시되어 있지만 `globalBalance`는 [`ShareMath:pricePerShare`](https://github.com/hedgemonyxyz/sherpa-vault-smartcontracts/blob/50eb8ad6ee048a767f7ed2265404c59592c098b7/contracts/lib/ShareMath.sol#L66-L77)에 전달되며, 여기서 즉시 `pending` 금액을 뺍니다: (`(totalBalance - pending) / totalSupply`):
```solidity
function pricePerShare(
    uint256 totalSupply,   // @audit-info globalShareSupply
    uint256 totalBalance,  // @audit-info globalBalance
    uint256 pendingAmount, // @audit-info globalTotalPending
    uint256 decimals
) internal pure returns (uint256) {
    uint256 singleShare = 10 ** decimals;
    return
        totalSupply > 0
            ? (singleShare * (totalBalance - pendingAmount)) / totalSupply
            : singleShare;
}
```
`_rollInternal`의 주석은 적용된 수학과 일치하지 않으며 실제 가격 계산에는 `pendingAmount`가 포함되지 않습니다.

주석을 변경하거나 주석이 맞다면 수학을 변경하는 것을 고려하십시오.


**Sherpa:** 커밋 [`9dbaf27`](https://github.com/hedgemonyxyz/sherpa-vault-smartcontracts/commit/9dbaf277ec7b7349af682aa5d9f6a6ae78151db9)에서 수정됨

**Cyfrin:** 확인함. 주석이 잘못되었고 수정되지 않았습니다.


### `SherpaUSD::consumeTotalStakedApproval` 및 `SherpaUSD::consumeAccountingApproval`은 누구나 호출 가능

**설명:** 크로스체인 회계의 리밸런싱/정산 흐름에서, [`SherpaUSD::consumeTotalStakedApproval`](48a767f7ed2265404c59592c098b7/contracts/SherpaUSD.sol#L282-L291) 및 [`SherpaUSD::consumeAccountingApproval`](https://github.com/hedgemonyxyz/sherpa-vault-smartcontracts/blob/50eb8ad6ee048a767f7ed2265404c59592c098b7/contracts/SherpaUSD.sol#L293-L302)은 `SherpaUSD::ownerMint`/`SherpaUSD::ownerBurn`에 의해 설정된 일회성 승인에 대한 "소비/지우기" 단계 역할을 합니다. 이들은 해당 조정이 적용된 후 승인이 재사용되는 것을 방지하기 위해 `SherpaVault::adjustTotalStaked` 및 `SherpaVault::adjustAccountingSupply` 주변에서 호출됩니다.

두 함수 모두 외부에서 호출 가능하며 `vault` 매개변수를 받지만, 상태 변경은 `if (msg.sender != vault) revert OnlyVaultCanConsume();`에 의해 게이트되며 승인은 호출자 주소로 키가 지정됩니다:
```solidity
function consumeTotalStakedApproval(address vault) external {
    // @audit-issue anyone can call by passing their own address as `vault`
    if (msg.sender != vault) revert OnlyVaultCanConsume();
    approvedTotalStakedAdjustment[vault] = 0;
    emit TotalStakedApprovalConsumed(vault);
}
```
결과적으로 어떤 주소든 함수를 호출할 수 있지만, 호출은 볼트의 승인 항목이 아니라 자신의 승인 항목만 지울 수 있습니다. 동작은 이 설계에서 정확하고 악용 불가능합니다. 그러나 명시적인 `vault` 매개변수와 결합된 개방형 호출 가능 표면은 통합자와 리뷰어에게 혼란을 줄 수 있습니다.

`vault` 매개변수를 제거하고 `onlyKeeper` 수정자를 추가하여 실제 볼트만 호출할 수 있도록 하는 것을 고려하십시오. 이는 최소 권한의 원칙을 따르고 사용 가능한 공격 표면을 제한합니다.

**Sherpa:** 커밋 [`c33eb52`](https://github.com/hedgemonyxyz/sherpa-vault-smartcontracts/commit/c33eb5212430b6c4115be9f87950e47540f20522)에서 수정됨

**Cyfrin:** 확인함. 두 함수 모두 이제 `vault` 매개변수가 제거되었고 `onlyKeeper` 수정자가 추가되었습니다.


### `CCIPReceiver` 의존성이 필요하지 않음

**설명:** `SherpaVault`는 `CCIPReceiver`를 상속하지만, 프로토콜의 크로스체인 흐름은 임시 메시지 전달보다는 CCIP 소각/발행 토큰 풀을 사용합니다. EVM 체인에서 Chainlink의 [크로스체인 토큰 패턴](https://docs.chain.link/ccip/concepts/cross-chain-token/evm/tokens)은 토큰/볼트 컨트랙트에 `CCIPReceiver` 구현을 요구하지 않으며, `mint/burn` 스타일 후크를 통한 풀 권한 부여만 요구합니다. `CCIPReceiver`(및 `_ccipReceive` 스텁)를 유지하면 기능 제공 없이 바이트코드 크기, 배포 비용 및 표면적이 증가합니다.

상속 및 관련 코드를 제거하여 컨트랙트를 단순화하고 가스/바이트코드 공간을 줄이며 실제로 사용되지 않는 메시지 브리지 의존성을 암시하는 것을 피하는 것을 고려하십시오.

**Sherpa:** 커밋 [`59974b2`](https://github.com/hedgemonyxyz/sherpa-vault-smartcontracts/commit/59974b29c59e2cc5afce87bbfd87a625bc05a94b)에서 제거됨

**Cyfrin:** 확인함. `CCIPReceiver` 의존성이 이제 제거되었습니다.


### `SherpaVault::redeem` 명칭이 모호함

**설명:** `SherpaVault`는 ERC-4626과 인접한 용어를 사용하지만 의미는 다릅니다. ERC-4626에서 `redeem`은 공유를 소각하여 자산을 인출하는 것을 의미합니다. `SherpaVault`에서 `redeem`은 미상환 공유를 사용자 지갑으로 이동하여 이전 입금을 확정하는 것을 의미합니다. 이 명칭은 ERC-4626 동작을 가정하는 통합자와 도구를 오도할 수 있습니다.

혼란을 방지하기 위해 `redeem`을 `finalizeDeposit` / `claimShares`로 이름을 바꾸는 것을 고려하십시오.

**Sherpa:** 커밋 [`8e9ba92`](https://github.com/hedgemonyxyz/sherpa-vault-smartcontracts/commit/8e9ba923e8402b877e16cd1d9a89143acdafe855)에서 수정됨

**Cyfrin:** 확인함. 이제 `claimShares`가 사용됩니다.


### `minimumSupply` 확인으로 인해 일부 SherpaUSD는 절대 언스테이킹할 수 없음

**설명:** SherpaVault의 `SherpaVault::_unstake` 함수에는 총 스테이킹 자산이 minimumSupply보다 작지 않고 0보다 큰지 확인하는 검사가 포함되어 있습니다. 그러나 다른 사용자가 의도적으로 또는 의도치 않게 사용자가 영구적으로 언스테이킹하는 것을 차단할 수 있습니다.

```solidity
// Ensure vault maintains minimum supply (allow full exit to 0)
        if (totalStaked - wrappedTokensToWithdraw < vaultParams.minimumSupply &&
            totalStaked - wrappedTokensToWithdraw > 0) {
            revert MinimumSupplyNotMet();
        }
```

예를 들어:
 - `minimumSupply` = 1000 SherpaUSD라고 가정합니다.
 - Alice가 1000 SherpaUSD를 입금합니다.
 - 악의적인 Bob이 1 wei SherpaUSD를 입금합니다. `_stakeInternal` 함수의 `if (totalWithStakedAmount < _vaultParams.minimumSupply) revert MinimumSupplyNotMet();` 문이 총 스테이킹 공급량 + 대기 금액, 즉 totalWithStakedAmount를 확인하므로 이는 허용됩니다. 이제 1000 SherpaUSD + 1 wei SherpaUSD가 됩니다.
 - Alice는 이제 Bob이 인출을 완료할 때까지 시스템을 나갈 수 없습니다. 이는 `_unstake` 함수의 minimumSupply 확인 때문에 발생합니다.
 - Alice는 1 wei SherpaUSD만 인출할 수 있으며 나머지는 영구적으로 잠깁니다.

공유된 스크립트를 기반으로 하면 `minimumSupply`가 1 USD일 것으로 예상되므로 이 문제는 현재 위험을 초래하지 않습니다.

**권장 완화 조치:** 안전 조치로 다음 권장 사항 중 하나 또는 둘 다를 구현하는 것이 좋습니다:
1. `minimumSupply`를 구성 가능하게 유지하기 위한 설정자 함수를 구현하십시오.
2. 모든 사용자가 개별적으로 최소 공급량 이상을 입금하도록 확인을 추가하십시오.

**Sherpa:** 커밋 [`720c2c0`](https://github.com/hedgemonyxyz/sherpa-vault-smartcontracts/commit/720c2c053d4e22fcb73a7bda97e4282fc749f5f4)에서 수정됨

**Cyfrin:** 확인함. 최소 입금이 강제되었습니다. `minimumSupply`는 불변으로 남습니다.


### 기본 내림 대신 명시적 반올림 동작 구현 고려

**설명:** `ShareMath.sol`의 모든 함수는 현재 가장 가까운 정수로 내림합니다. 이는 SherpaVault의 특정 경우에 불리할 수 있습니다.

예를 들어, `ShareMath.pricePerShare` 함수는 정수 나눗셈을 사용하여 정밀도 손실을 유발합니다. 따라서 주당 가격을 약간 과소평가하게 됩니다.

```solidity
function pricePerShare(
    uint256 totalSupply,
    uint256 totalBalance,
    uint256 pendingAmount,
    uint256 decimals
) internal pure returns (uint256) {
    uint256 singleShare = 10 ** decimals;
    return
        totalSupply > 0
            ? (singleShare * (totalBalance - pendingAmount)) / totalSupply
            : singleShare;
}
```

**권장 완화 조치:** 각 경우에 왜 각각의 반올림 방향이 적절한지 논리적으로 설명하는 주석을 추가하는 것 외에도 명시적인 반올림을 수행하는 것이 좋습니다.

**Sherpa:** 커밋 [`61345a1`](https://github.com/hedgemonyxyz/sherpa-vault-smartcontracts/commit/61345a1967311167ec8fe4dba81bb2a21247ea50)에서 수정됨

**Cyfrin:** 확인함. 특정 반올림 방향에 대한 문서가 추가되었습니다.

\clearpage
## 가스 최적화 (Gas Optimization)


### 상태 업데이트 전 이벤트를 발생시켜 설정자(setters) 최적화

**설명:** `SherpaUSD::setKeeper`, `SherpaUSD::setOperator`, `SherpaUSD::setAutoTransfer` 및 `SherpaVault::setDepositsEnabled`, `SherpaVault::setStableWrapper` 함수는 이벤트 발생에 사용되는 이전 값을 저장하기 위해 불필요한 메모리 변수를 생성합니다. 그러나 이벤트를 먼저 발생시키면 이것이 필요하지 않습니다.

예를 들어, setKeeper 함수는 다음과 같은 방식으로 최적화할 수 있습니다:

```solidity
function setKeeper(address _keeper) external onlyOwner {
    if (_keeper == address(0)) revert AddressMustBeNonZero();
    emit KeeperSet(keeper, _keeper);
    keeper = _keeper;
}
```

**권장 완화 조치:** 이벤트를 먼저 발생시켜 메모리 변수를 제거하는 것을 고려하십시오.

**Sherpa:** 커밋 [`7e34a6b`](https://github.com/hedgemonyxyz/sherpa-vault-smartcontracts/commit/7e34a6b064b63d8f7a3f2c66c49e10adab0198b7)에서 수정됨

**Cyfrin:** 확인함.

\clearpage

