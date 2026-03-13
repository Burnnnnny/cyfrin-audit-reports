**Lead Auditors**

[Farouk](https://x.com/Ubermensh3dot0)

[qpzm](https://x.com/qpzmly)

**Assisting Auditors**

[Alexzoid](https://x.com/alexzoid) (Formal Verification)

---

# 발견 사항 (Findings)
## 중간 위험 (Medium Risk)


### 높은 이용률 상황에서 `Venus._disableUnderlying`가 실패함

**설명:** [Sherlock M-1](https://github.com/PredictDotFun/prediction-market/blob/audit/qpzm/audit/others/2025.11.26%20-%20Final%20-%20Predict.fun%20Collaborative%20Audit%20Report%201764175259.pdf) 수정(`ef0712b`)으로, 높은 이용률 때문에 출금이 막힐 때 Venus 통합을 비활성화하기 위한 긴급 메커니즘 `_disableUnderlying`이 추가되었습니다. 그러나 이 함수 자체가 비활성화 전에 모든 자금을 출금하기 위해 `redeemUnderlying`를 호출합니다.

```solidity
// Venus.sol:259-263
if (redeemAmount > 0) {
    uint256 err = IVToken(vToken).redeemUnderlying(redeemAmount);
    if (err != 0) {
        revert VTokenCallFailed(err);  // ← Reverts during high utilization
    }
}
```

Venus 풀의 이용률이 높으면 `redeemUnderlying`는 에러 코드 13(`TOKEN_INSUFFICIENT_CASH`)과 함께 실패하고, 그 결과 함수 전체가 revert됩니다.

순환적인 문제는 다음과 같습니다.
```
[원래 문제]   높은 이용률 -> redeemUnderlying 실패 -> merge/redeem 동작 불가
[수정안]      긴급 메커니즘으로 _disableUnderlying 추가
[실제 동작]   높은 이용률 -> _disableUnderlying가 redeemUnderlying 호출 -> 실패 -> 비활성화 불가
```

**영향:** 이 긴급 비활성화 메커니즘은 바로 자신이 해결하려던 긴급 상황에서 사용할 수 없습니다.
모든 출금 작업이 막히며, 관리자가 탈출할 수 있는 동작하는 수단도 존재하지 않습니다. `softDisableUnderlying` 함수가 있으면 다음과 같은 점에서 도움이 됩니다.

1. 즉시 플래그 전환: 출금을 시도하지 않고 `underlyingIsEnabled = false`만 설정합니다.
2. 새 split은 underlying을 누적: 플래그가 꺼져 있으면 `splitPosition`은 자산을 Venus에 예치하지 않고 컨트랙트에 보관합니다.
3. merge 재개 가능: 플래그가 `false`일 때 merge는 컨트랙트의 underlying 잔액에서 직접 전송합니다(else branch).
4. Venus 내 자금은 안전하게 유지: 기존 vToken 포지션은 이용률이 낮아질 때까지 그대로 유지되고, 이후 관리자가 `disableUnderlying`를 통해 출금할 수 있습니다.

**개념 증명 (Proof of Concept):** 파일: [`test/foundry/poc/M-1_DisableUnderlyingHighUtilization.t.sol`](https://github.com/PredictDotFun/prediction-market/blob/audit/qpzm/test/foundry/poc/M-1_DisableUnderlyingHighUtilization.t.sol)

```bash
forge test --match-contract DisableUnderlyingHighUtilizationPOC -vvv
```

| 테스트 | 결과 |
|------|--------|
| `test_disableUnderlying_failsDuringHighUtilization` | PASS (revert 확인) |
| `test_fullScenario_emergencyDisableFails` | PASS |

**권장 완화 조치:** 출금을 시도하지 않고 플래그만 전환하는 `softDisableUnderlying` 함수를 추가하십시오.

```solidity
event UnderlyingTokenSoftDisabled(address indexed underlying);

/**
 * @notice Soft disables Venus without withdrawal. Use during high utilization.
 * @param underlying The underlying token
 */
function _softDisableUnderlying(address underlying) internal {
    if (!underlyingIsEnabled[underlying]) {
        revert UnderlyingTokenAlreadyDisabled();
    }
    underlyingIsEnabled[underlying] = false;
    // No withdrawal - funds remain in Venus until utilization drops
    emit UnderlyingTokenSoftDisabled(underlying);
}
```

soft disable 이후에는 플래그가 이미 `false`이므로 `disableUnderlying`는 revert하게 됩니다. 이용률이 떨어지면 관리자는 `enableUnderlying`를 호출해 Venus 통합을 재개하거나, 남아 있는 vToken을 인출하는 `withdrawRemainingVTokens` 함수를 별도로 추가할 수 있습니다. 즉 2단계 접근이 됩니다.

1. 높은 이용률 위기 시: `softDisableUnderlying` 호출
2. 이용률 하락 후: `enableUnderlying` 또는 `withdrawRemainingVTokens` 호출

**부분 유동성 안전성 분석:**

`softDisableUnderlying` 접근은 회계를 올바르게 유지합니다. 지급 가능성 불변식은 모든 단계에서 유지됩니다.

```
totalAssets >= totalYieldBearingConditionalTokenSupply

where:
  totalAssets = underlying.balanceOf(contract) + vToken.balanceOfUnderlying(contract)
  totalYieldBearingConditionalTokenSupply = sum of all YieldBearingConditionalToken balances for this underlying
```

| 단계 | 동작 | Underlying | vToken | YBCT 공급량 | 불변식 |
|------|--------|------------|--------|-------------|-----------|
| 1 | 위기 전 (flag=true) | 0 | 1000 | 1000 | 1000 ≥ 1000 |
| 2 | `softDisableUnderlying` (flag=false) | 0 | 1000 | 1000 | 1000 ≥ 1000 |
| 3 | 새 사용자가 200 split (컨트랙트에 유지) | 200 | 1000 | 1200 | 1200 ≥ 1200 |
| 4 | 임의 사용자가 200 merge (underlying에서 지급) | 0 | 1000 | 1000 | 1000 ≥ 1000 |
| 5 | 이용률 하락 후 완전 비활성화 | 1000 | 0 | 1000 | 1000 ≥ 1000 |

자금은 fungible합니다. 프로토콜은 어떤 사용자가 언제 예치했는지 추적할 필요가 없습니다. "전부 아니면 전무(all-or-nothing)" 접근과 "부분 유동성" 접근의 차이는 다음과 같습니다.

| 접근 방식 | 위기 중 유동 자금 | 출금 가능 여부 |
|----------|-------------------|----------------|
| 현재 방식 (all-or-nothing) | 0 | 0 |
| `softDisableUnderlying` 도입 시 | 신규 예치분 | 가능 (선착순) |

부분 유동성이 더 낫습니다. 아무도 출금하지 못하는 것보다는 적어도 일부 사용자가 출금할 수 있기 때문입니다.

**Predict.fun:** 인지함. 설계 변경 폭이 꽤 크고, 재배포까지 정당화할 정도는 아닐 가능성이 높다고 판단합니다.

\clearpage
## 낮은 위험 (Low Risk)


### VToken 가치 하락이 수익 계산에서 언더플로우를 유발함

**설명:** `Venus` 컨트랙트는 `depositedAmount[underlying]`에 예치된 스테이블코인 금액을 추적하지만, [`Venus::splitPrincipalAndYield`](https://github.com/PredictDotFun/prediction-market/blob/main/contracts/YieldBearing/Venus.sol#L118)의 수익 계산은 VToken 환율이 오르기만 한다고 가정합니다. 그러나 bad debt, 익스플로잇, 기타 Venus 프로토콜 손실로 인해 VToken 가치가 떨어지면 계산이 언더플로우를 일으킵니다.

```solidity
// Venus.sol:118-123
function splitPrincipalAndYield(address underlying) public returns (uint256 principal, uint256 yield) {
    address vToken = _getVToken(underlying);
    uint256 exchangeRateCurrent = IVToken(vToken).exchangeRateCurrent();
    (, principal) = divScalarByExpTruncate(depositedAmount[underlying], Exp({mantissa: exchangeRateCurrent}));
    yield = IVToken(vToken).balanceOf(address(this)) - principal;  // ← Underflows when exchange rate drops
}
```

`principal` 계산은 예치된 underlying 금액을 vToken 단위로 변환합니다.
```
principal = depositedAmount / exchangeRate
```

환율이 떨어지면 `principal`은 증가합니다. 즉, 동일한 underlying 금액을 표현하는 데 더 많은 vToken이 필요해집니다. 만약 `principal > vToken.balanceOf(this)`가 되면 수익 계산이 언더플로우됩니다.

예시 시나리오:
1. 사용자가 환율 1.0e18에서 1000 USDC를 예치하고 약 1000 vToken을 받음
2. `depositedAmount = 1000e18`
3. Venus에서 bad debt가 발생해 환율이 0.8e18로 하락
4. `principal = 1000e18 / 0.8e18 = 1250` vTokens
5. 실제 vToken 잔액은 약 1000
6. `yield = 1000 - 1250` → **UNDERFLOW**

유사한 문제는 `_disableUnderlying`에도 영향을 줍니다.

```solidity
// Venus.sol:253-264
uint256 redeemAmount = IVToken(vToken).balanceOfUnderlying(address(this));  // e.g., 800e18 (after bad debt)
uint256 originalDepositedAmount = depositedAmount[underlying];              // e.g., 1000e18
if (redeemAmount < originalDepositedAmount) {  // 800 < 1000 → true
    redeemAmount = originalDepositedAmount;    // redeemAmount = 1000e18, but only 800e18 available!
}

if (redeemAmount > 0) {
    uint256 err = IVToken(vToken).redeemUnderlying(redeemAmount);  // redeemUnderlying(1000e18) fails
    if (err != 0) {
        revert VTokenCallFailed(err);
    }
}
```

`balanceOfUnderlying < originalDepositedAmount`인 경우(예: bad debt로 인해 800 < 1000), 코드는 `redeemAmount = 1000e18`로 강제합니다. 하지만 vToken 가치가 실제로는 800e18밖에 안 되므로 Venus는 이를 이행할 수 없고, 에러 코드를 반환하면서 `VTokenCallFailed`로 revert됩니다.

**영향:** Venus 프로토콜에서 손실 이벤트(bad debt, 익스플로잇, 이자율 계산 오류 등)가 발생하면:

| 함수 | 영향 |
|----------|--------|
| `splitPrincipalAndYield` | 언더플로우로 revert |
| `_claimYield` | revert (`splitPrincipalAndYield` 호출) |
| `_disableUnderlying` | `VTokenCallFailed`로 revert (실제 보유량보다 많이 인출 시도) |

프로토콜은 교착 상태에 빠집니다. 관리자는 Venus 통합을 비활성화할 수 없고, 수익 관련 작업은 계속 revert됩니다. 즉 다음과 같은 상황이 됩니다.
- underlying 가치는 이미 하락해 있음
- 관리자가 이 상태에서 복구할 방법이 없음
- 모든 수익 관련 작업이 무기한 차단됨

**개념 증명 (Proof of Concept):** 파일: [`test/foundry/poc/M-2_VTokenValueDecrease.t.sol`](https://github.com/PredictDotFun/prediction-market/blob/audit/qpzm/test/foundry/poc/M-2_VTokenValueDecrease.t.sol)

```bash
forge test --match-contract VTokenValueDecreasePOC -vvv
```

| 테스트 | 결과 |
|------|--------|
| `test_disableUnderlying_revertsWhenExchangeRateDrops` | PASS (`VTokenCallFailed(9)`) |
| `test_splitPrincipalAndYield_underflowsWhenExchangeRateDrops` | PASS (산술 언더플로우) |
| `test_claimYield_revertsWhenExchangeRateDrops` | PASS (`splitPrincipalAndYield` 호출) |

POC의 핵심 시나리오:
```solidity
function test_disableUnderlying_revertsWhenExchangeRateDrops() public {
    // Simulate Venus bad debt: exchange rate drops to 0.8e18 (20% loss)
    vToken.setExchangeRate(0.8e18);

    // Now:
    // - vToken balance: 1000e18 vTokens
    // - balanceOfUnderlying: 1000e18 * 0.8e18 / 1e18 = 800e18 (worth only 800 now)
    // - depositedAmount: 1000e18 (original deposit)
    //
    // _disableUnderlying logic:
    // 1. redeemAmount = balanceOfUnderlying = 800e18
    // 2. originalDepositedAmount = 1000e18
    // 3. if (800 < 1000) -> redeemAmount = 1000e18  <-- tries to redeem more!
    // 4. redeemUnderlying(1000e18) fails because only 800e18 worth available

    vm.expectRevert(abi.encodeWithSelector(Venus.VTokenCallFailed.selector, 9));
    venus.disableUnderlying(address(underlying), yieldRecipient);
}
```

**권장 완화 조치:** 최소 수정으로, Venus 가치가 하락해도 revert하지 않도록 만드십시오.
1. `splitPrincipalAndYield`에서 `vTokensForPrincipal >= vTokenBalance`이면 `(vTokenBalance, 0)`을 반환합니다.
2. `_disableUnderlying`에서는 `originalDepositedAmount`를 강제하지 말고 실제 인출 가능한 양만 인출합니다.

참고로 이 최소 수정은 손실 시나리오에서 여전히 선착순 출금을 허용합니다. 보다 공정한 손실 분배를 원한다면, 사용자 상환 함수(`mergePositions`, `redeemPositions`)가 health ratio를 검사해야 합니다.
- `healthRatio >= 1`: 원금 전액 상환(손실 없음)
- `healthRatio < 1`: `amount * healthRatio`만 상환(비례 손실)

**Predict.fun:** 본 발견 사항을 인지했습니다. Venus 프로토콜의 환율 메커니즘을 조사한 결과는 다음과 같습니다.

현재 Venus 동작:
- 환율 공식: `(totalCash + totalBorrows - totalReserves) / totalSupply`
- Venus는 기본적으로 socialized debt를 구현하지 않음
- `liquidateBorrowFresh`를 통한 청산은 전체 부채 상환을 요구하므로, bad debt를 단순히 상각하는 메커니즘이 없음
- 정상 동작에서는 환율이 감소하지 않아야 함

환율이 감소할 수 있는 경우:

| 시나리오 | 가능성 | Venus 기본 동작 여부 |
|----------|------------|----------------|
| 정상적인 출금/상환 | 해당 없음 | 환율 변화 없음 |
| Socialized debt | 매우 낮음 | **아님** - 구현되어 있지 않음 |
| Venus 익스플로잇 | 낮음 | 외부 리스크 |


**Cyfrin:** Venus vToken은 **업그레이드 가능한 프록시 컨트랙트**입니다.
- 프록시: `0x95c78222B3D6e262426483D42CfA53685A67Ab9D`
- 구현체: `0x33d17f1e6107cd4d711b56eb0094bf39a471a8b5`

Venus 거버넌스는 향후 socialized debt를 추가하거나 환율 공식을 변경하도록 업그레이드할 수 있습니다. 이는 현재 제어 범위를 벗어나지만, 권장 완화 조치는 다음에 대한 defense-in-depth를 제공합니다.
1. Venus 익스플로잇 시나리오
2. 향후 Venus 업그레이드로 debt socialization이 도입되는 경우
3. 환율을 감소시킬 수 있는 기타 예기치 못한 메커니즘

이러한 엣지 케이스를 우아하게 처리할 수 있는 수정이 바람직합니다.


\clearpage
## 정보 (Informational)


### `WhitelistedERC1155`의 OR 로직으로 인한 전송 화이트리스트 우회

**설명:** [`_isTransferAllowed()`](https://github.com/PredictDotFun/prediction-market/blob/main/contracts/ConditionalTokens/WhitelistedERC1155.sol#L90-L100) 함수는 중첩된 IF 문을 사용해 AND가 아니라 OR 로직을 구현합니다. 즉 `msg.sender`, `from`, `to` 중 하나라도 화이트리스트에 있으면 전송이 허용되며, 모든 당사자가 화이트리스트에 있어야 하는 구조가 아닙니다.

```solidity
function _isTransferAllowed(address from, address to) internal view {
    if (transferControlEnabled) {
        if (!isTransferAllowed[msg.sender]) {
            if (!isTransferAllowed[from]) {
                if (!isTransferAllowed[to]) {
                    revert NotAllowedToTransfer();  // Only reverts if ALL are false
                }
            }
        }
    }
}
```

**영향:** 의도가 신뢰되지 않은 당사자 간 전송을 제한하는 것이라면, 기대보다 화이트리스트 보호가 약해집니다.

**권장 완화 조치:** 의도가 모든 당사자가 화이트리스트에 있어야 한다는 것이라면 다음과 같이 구현하십시오.

```solidity
function _isTransferAllowed(address from, address to) internal view {
    if (transferControlEnabled) {
        if (!isTransferAllowed[msg.sender] || !isTransferAllowed[from] || !isTransferAllowed[to]) {
            revert NotAllowedToTransfer();
        }
    }
}
```

Ethena의 StakedUSDe에도 유사한 화이트리스트 패턴이 있지만, 그 경우는 의도를 명확히 드러내는 OR 조건을 명시적으로 사용합니다.

[StakedUSDe.sol#L231-L236](https://github.com/ethena-labs/bbp-public-assets/blob/f3e56d5f06bfef82367d5d5b561398e91d5bebc1/contracts/contracts/StakedUSDe.sol#L231-L236)

```solidity
function _beforeTokenTransfer(address from, address to, uint256) internal virtual override {
    if (hasRole(FULL_RESTRICTED_STAKER_ROLE, from) || hasRole(FULL_RESTRICTED_STAKER_ROLE, to)) {
        revert OperationNotAllowed();
    }
    // ...
}
```

**Predict.fun:** 인지함. `msg.sender`, `from`, `to` 중 하나만 화이트리스트에 있어도 전송을 허용하는 것이 의도된 동작입니다.



### BLS12-381 상수 시간 hash-to-curve로 업그레이드를 고려할 것

**설명:** [`CTHelpers::getCollectionId`](https://github.com/PredictDotFun/prediction-market/blob/main/contracts/ConditionalTokens/CTHelpers.sol#L393)는 원래 Gnosis Conditional Tokens(2019)에서 가져온 try-and-increment 알고리즘을 사용합니다.

```solidity
function getCollectionId(
    bytes32 parentCollectionId,
    bytes32 conditionId,
    uint indexSet
) internal view returns (bytes32) {
    uint x1 = uint(keccak256(abi.encodePacked(conditionId, indexSet)));
    bool odd = x1 >> 255 != 0;
    uint y1;
    uint yy;
    do {
        x1 = addmod(x1, 1, P);
        yy = addmod(mulmod(x1, mulmod(x1, x1, P), P), B, P);
        y1 = sqrt(yy);
    } while (mulmod(y1, y1, P) != yy); // @audit Variable iterations
    // ...
}
```

이 접근은 2025년 3월 Pascal 업그레이드 이후 이용 가능한 최신 대안과 비교하면 몇 가지 한계가 있습니다.

**비교:**

| 항목 | 현재 방식 (BN254 Try-and-Increment) | BLS12-381 (`MAP_FP_TO_G1` 프리컴파일) |
|--------|-----------------------------------|-------------------------------------|
| **알고리즘** | x가 이차 잉여가 될 때까지 증가 | Simplified SWU (상수 시간 매핑) |
| **시간 복잡도** | 가변적 (~50% 확률로 매 반복 성공) | 상수 시간 (단일 프리컴파일 호출) |
| **가스 비용** | 보통 ~2,000-5,000, 더 커질 수 있음 | ~5,500 (고정) |
| **보안 수준** | 약 100비트 | 약 128비트 |
| **타이밍 사이드채널** | 취약함 (반복 횟수가 정보 노출) | 방어 가능 (상수 시간) |
| **프리컴파일** | `ecAdd` (0x06), `ecMul` (0x07) | `MAP_FP_TO_G1` (0x12) |
| **출력 크기** | 32 bytes (압축) | 48 bytes (압축) |
| **지원 환경** | 모든 EVM 체인 | Pectra+ 체인 (Ethereum 2025년 5월, BNB Pascal 2025년 3월) |

**영향:** 현재 구현의 직접적인 취약점은 아닙니다. 우리의 [루프 분석](../CTHelpers-getCollectionId-loop-analysis.md)에 따르면 최악의 경우도 제한적입니다(1600만 개 이상 샘플에서 최대 26회 반복, 약 85k gas). 그러나 try-and-increment 방식은 다음과 같은 특징이 있습니다.

1. **가변 가스 비용:** 최악의 반복 수가 일반적인 가스 추정보다 커질 수 있습니다(평균 약 20k, 최악 약 85k).
2. **타이밍 사이드채널:** 반복 횟수가 입력 해시에 대한 정보를 노출합니다.
3. **낮은 보안 여유:** BN254는 약 100비트, BLS12-381은 약 128비트입니다.

**권장 완화 조치:** 향후 업그레이드에서는 EIP-2537로 도입된 `MAP_FP_TO_G1` 프리컴파일(address 0x10)을 활용해 BLS12-381로 전환하는 방안을 고려하십시오. 이 방식은 다음 장점이 있습니다.

- 상수 시간 실행(타이밍 사이드채널 없음)
- 고정 가스 비용(예측 가능)
- 더 높은 보안 수준(128비트)

```solidity
// BLS12-381 precompile addresses (EIP-2537)
address constant BLS12_G1ADD = address(0x0b);        // 375 gas
address constant BLS12_MAP_FP_TO_G1 = address(0x10); // 5500 gas (constant-time SWU)

function hashToG1(bytes32 conditionId, uint256 indexSet) internal view returns (bytes memory) {
    // 1. Hash to 64-byte field element
    bytes memory fieldElement = new bytes(64);
    bytes32 hash = keccak256(abi.encodePacked(conditionId, indexSet));
    assembly {
        mstore(add(fieldElement, 0x40), hash) // Store hash in lower 32 bytes
    }

    // 2. Single precompile call - constant time, no iterations
    (bool success, bytes memory point) = BLS12_MAP_FP_TO_G1.staticcall(fieldElement);
    require(success, "MAP_FP_TO_G1 failed");

    return point; // 128 bytes (uncompressed G1 point)
}
```

참고: 이 전환은 `collectionId` 형식을 32바이트에서 48바이트(압축된 G1 포인트)로 바꾸게 되므로, 저장 구조와 기존 통합에도 영향을 줍니다.

**참고 자료:**
- [EIP-2537: BLS12-381 curve operations](https://eips.ethereum.org/EIPS/eip-2537)
- [Gnosis Conditional Tokens (2019)](https://github.com/gnosis/conditional-tokens-contracts)
- [hash_to_curve_comparison.md](https://github.com/PredictDotFun/prediction-market/blob/audit/qpzm/audit/hash_to_curve_comparison.md)
- [CTHelpers-getCollectionId-loop-analysis.md](../CTHelpers-getCollectionId-loop-analysis.md) - 최악 루프 반복 분석

**Predict.fun:** 인지함. 이 코드는 원래 Gnosis Conditional tokens에서 가져온 것이므로 현재 상태를 유지할 예정입니다. 변경 시 업스트림 코드와 다른 보안 보장을 갖게 될 수 있습니다.


\clearpage
