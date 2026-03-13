**수석 감사자 (Lead Auditors)**

[0kage](https://twitter.com/0kage_eth)

[Giovanni Di Siena](https://twitter.com/giovannidisiena)

[Hans](https://twitter.com/hansfriese)

[Carlos](https://twitter.com/carlitox477)

**보조 감사자 (Assisting Auditors)**

[Alex Roan](https://twitter.com/alexroan)


---

# 발견 사항 (Findings)
## 중간 중요도 (Medium Risk)


### Chainlink 가격 및 L2 시퀀서 가동 시간 피드가 권장 검증 및 안전장치 없이 사용됨 (Chainlink price and L2 sequencer uptime feeds are not used with recommended validations and guardrails)

**설명:** [`ChainlinkPriceOracleV1`](https://github.com/feat/dolomite-margin/blob/e10f14320ece20d7492e8e68400333c5c7dec656/contracts/external/oracles/ChainlinkPriceOracleV1.sol)은 [`Storage::fetchPrice`](https://github.com/feat/dolomite-margin/blob/e10f14320ece20d7492e8e68400333c5c7dec656/contracts/protocol/lib/Storage.sol#L455-L456)에서 사용되는 [IPriceOracle](https://github.com/feat/dolomite-margin/blob/e10f14320ece20d7492e8e68400333c5c7dec656/contracts/external/oracles/ChainlinkPriceOracleV1.sol#L38)의 구현체입니다. 프로토콜은 현재 [가격이 0이 될 수 없음](https://github.com/feat/dolomite-margin/blob/e10f14320ece20d7492e8e68400333c5c7dec656/contracts/protocol/lib/Storage.sol#L457-L462)을 검증하지만, 진부함(staleness)과 라운드 불완전성(round incompleteness)에 대한 확인이 없어 잘못된 0이 아닌 가격을 사용할 수 있습니다. `ChainlinkPriceOracleV1`은 현재 이러한 추가 검증과 함께 권장되는 `IChainlinkAggregator::latestRoundData` 함수 대신 [더 이상 사용되지 않는(deprecated)](https://docs.chain.link/data-feeds/api-reference) `IChainlinkAggregator::latestAnswer` 함수를 사용합니다.

L2 시퀀서 다운타임 검증은 [`OperationImpl::_verifyFinalState`](https://github.com/feat/dolomite-margin/blob/e10f14320ece20d7492e8e68400333c5c7dec656/contracts/protocol/impl/OperationImpl.sol#L317) 및 [`LiquidateOrVaporizeImpl::liquidate`](https://github.com/feat/dolomite-margin/blob/e10f14320ece20d7492e8e68400333c5c7dec656/contracts/protocol/impl/LiquidateOrVaporizeImpl.sol#L64) 모두에서 [IOracleSentinel 인터페이스](https://github.com/feat/dolomite-margin/blob/e10f14320ece20d7492e8e68400333c5c7dec656/contracts/protocol/interfaces/IOracleSentinel.sol#L27-L28)를 준수하는 컨트랙트를 호출하여 처리됩니다. [`getFlag`](https://arbiscan.io/address/0x3c14e07edd0dc67442fa96f1ec6999c57e810a83#code)가 호출되는 컨트랙트는 단순히 시퀀서 가동 상태만 반환하고 다른 것은 반환하지 않습니다.

일정 기간의 다운타임 후 L2 시퀀서가 다시 온라인 상태가 되고 오라클이 가격을 업데이트하면, 다운타임 동안 발생한 모든 가격 변동이 한꺼번에 적용됩니다. 이러한 변동이 크면 대출자는 포지션을 구하려 서두르고 청산인은 대출자를 청산하려 서두르게 됩니다. 청산은 향후 Chainlink Automation에 의해 처리될 예정이므로 청산이 허용되지 않는 유예 기간(grace period)이 없으면 대출자는 대규모 청산을 겪을 가능성이 높습니다. L2 다운타임으로 인해 원하더라도 포지션에 조치를 취할 수 없었기 때문에 이는 대출자에게 불공평합니다.

**파급력:**
1. 진부함 및 라운드 불완전성 검증 부족으로 인해 잘못된 0이 아닌 가격이 사용될 수 있습니다.
2. 시퀀서 다운타임 유예 기간이 없으면 중간 기간 동안 큰 가격 편차가 있는 경우 시퀀서가 다시 가동되자마자 대출 포지션이 즉시 청산될 수 있습니다.

**권장 완화 방안:** Dolomite Margin 프로토콜은 Chainlink 데이터 피드에서 반환된 값을 올바르게 검증하고, L2 시퀀서 다운타임 후 청산이 재개되기 전에 대출자가 추가 담보를 예치할 수 있는 유예 기간을 제공해야 합니다.

**Dolomite:** [6a8ae06](https://github.com/dolomite-exchange/dolomite-margin/commit/6a8ae061fa84110db7b111512f705a6cd0a472bb) 커밋으로 수정되었습니다.

**Cyfrin:** 인정함.


### 관리자가 `AdminImpl::ownerWithdrawUnsupportedTokens`를 사용하여 이중 진입점 ERC-20 마켓을 고갈시킬 수 있음 (Admin can drain market of double-entrypoint ERC-20 using `AdminImpl::ownerWithdrawUnsupportedTokens`)

**설명:** `AdminImpl::ownerWithdrawUnsupportedTokens`는 소유자가 Dolomite Margin 주소로 잘못 전송된 지원되지 않는 ERC-20 토큰을 인출할 수 있도록 의도되었습니다. 이중 진입점(double-entrypoint) ERC-20 토큰이 Dolomite에 마켓으로 상장된 경우 관리자가 전체 토큰 잔액을 고갈시킬 수 있습니다.

이러한 토큰은 레거시 토큰이 로직을 새 토큰에 위임하므로 두 개의 별도 주소가 동일한 토큰과 상호 작용하는 데 사용된다는 점에서 문제가 됩니다. 이전 사례로는 [Compound에 통합되었을 때 취약점을 초래한 TUSD](https://blog.openzeppelin.com/compound-tusd-integration-issue-retrospective/)가 있습니다. 이는 특히 이러한 유형의 취약점을 쉽게 감지할 수 없으므로 담보 토큰을 신중하게 선택하는 것이 중요함을 강조합니다. 또한 업그레이드 가능한 담보 토큰이 향후 이중 진입점 토큰이 될 가능성(예: USDT)도 비현실적이지 않으므로 이 점도 고려해야 합니다.

이중 진입점 토큰의 레거시 토큰 주소를 [`AdminImpl::ownerWithdrawUnsupportedTokens`](https://github.com/feat/dolomite-margin/blob/e10f14320ece20d7492e8e68400333c5c7dec656/contracts/protocol/impl/AdminImpl.sol#L183)의 인수로 전달함으로써 관리자는 전체 토큰 잔액을 고갈시킬 수 있습니다. 레거시 토큰은 Dolomite에 추가되지 않았으므로 유효한 마켓 ID가 없으며, 따라서 [`AdminImpl::_requireNoMarket`](https://github.com/feat/dolomite-margin/blob/e10f14320ece20d7492e8e68400333c5c7dec656/contracts/protocol/impl/AdminImpl.sol#L589)을 통과할 것입니다.

```solidity
function ownerWithdrawUnsupportedTokens(
    Storage.State storage state,
    address token,
    address recipient
)
    public
    returns (uint256)
{
    _requireNoMarket(state, token);

    uint256 balance = IERC20Detailed(token).balanceOf(address(this));
    token.transfer(recipient, balance);

    emit LogWithdrawUnsupportedTokens(token, balance);

    return balance;
}
```

그러나 레거시 토큰에 대한 함수 호출은 새 버전으로 전달되므로 반환된 잔액은 프로토콜에 있는 토큰의 잔액이 되며, 이는 관리자에게 전송될 수 있습니다.

**파급력:** 이 발견 사항은 사용자의 비용으로 프로토콜을 지불 불능 상태로 만들 수 있는 치명적인 영향을 미칠 수 있지만, 외부 가정으로 인해 가능성이 낮으므로 심각도를 **중간(MEDIUM)**으로 평가합니다.

**권장 완화 방안:** 지원되는 모든 Dolomite Margin 마켓을 반복하고 지원되지 않는 토큰을 인출하기 전후의 담보 토큰 잔액이 동일한지 검증하세요.

**Dolomite:** Dolomite의 핵심 프로토콜은 수백(또는 수천) 개의 자산 상장을 지원하도록 구축되었으므로 제안된 수정 사항을 구현하고 싶지 않습니다. 첫째, 우리는 이러한 이상한 동작을 하는 토큰을 상장할 의도가 없습니다. 둘째, 제안된 수정 사항은 상장된 자산 수가 증가함에 따라 각 토큰의 잔액을 반복하는 비용이 매우 많이 들기 때문에 특정 블록에 대해 가스가 부족해질 수 있습니다.

향후 이 기능을 추가하는 소유권 어댑터를 통해 핫픽스를 제공할 수 있습니다.

**Cyfrin:** 인정함.


### `TradeImpl::buy`의 부정확한 회계 처리로 인한 사용자 자금 손실 가능성 (Inaccurate accounting in `TradeImpl::buy` could lead to loss of user funds)

**설명:** Dolomite Margin 내에서 [`TradeImpl::buy`](https://github.com/feat/dolomite-margin/blob/e10f14320ece20d7492e8e68400333c5c7dec656/contracts/protocol/impl/TradeImpl.sol#L44-L110)는 매수 거래를 수행하는 데 사용되며 매개변수 중 하나로 `Actions.BuyArgs memory args`를 받습니다. 이러한 인수는 호출자가 제공하므로 `args.exchangeWrapper`에 대해 임의의 값을 자유롭게 설정할 수 있습니다.

이 `args.exchangeWrapper` 매개변수는 컨트랙트가 받아야 하는 [`takerWei`의 양을 계산](https://github.com/feat/dolomite-margin/blob/e10f14320ece20d7492e8e68400333c5c7dec656/contracts/protocol/lib/Exchange.sol#L106-L111)하고 Dolomite 컨트랙트로 [실제로 전송되어야 하는 양을 계산](https://github.com/feat/dolomite-margin/blob/e10f14320ece20d7492e8e68400333c5c7dec656/contracts/protocol/lib/Exchange.sol#L138-L145)하는 데 사용됩니다.

이 두 값이 동일하다는 보장은 없으므로 받아야 하는 금액과 실제로 받은 금액 간에 차이가 있을 때 문제가 발생합니다. 이는 [`TradeImpl::buy`](https://github.com/feat/dolomite-margin/blob/e10f14320ece20d7492e8e68400333c5c7dec656/contracts/protocol/impl/TradeImpl.sol#L84-L89)에서 부분적으로 처리되며, 받은 금액이 예상 금액보다 크거나 같은지 확인하여 슬리피지 검사 역할을 합니다. 그러나 내부 회계는 실제로 받은 금액이 아니라 받을 것으로 예상되는 값을 기반으로 [업데이트](https://github.com/feat/dolomite-margin/blob/e10f14320ece20d7492e8e68400333c5c7dec656/contracts/protocol/impl/TradeImpl.sol#L84-L95)됩니다.

**파급력:** 부정확한 회계 처리는 사용자 자금 손실로 이어질 수 있습니다. `takerWei == tokensReceived`가 예상되지만 보장되지는 않는다는 점을 감안할 때 심각도를 **중간(MEDIUM)**으로 평가합니다.

**권장 완화 방안:** 프로토콜 회계에 대한 올바른 업데이트를 보장하기 위해 [다음 줄](https://github.com/feat/dolomite-margin/blob/e10f14320ece20d7492e8e68400333c5c7dec656/contracts/protocol/impl/TradeImpl.sol#L91-L95)을 수정하세요.

```diff
//  TradeImpl::buy
    Require.that(
        tokensReceived.value >= makerWei.value,
        FILE,
        "Buy amount less than promised",
        tokensReceived.value
    );

-   state.setPar(
-       args.account,
-       args.makerMarket,
-       makerPar
-   );
+   state.setParFromDeltaWei(
+       args.account,
+       args.makerMarket,
+       makerIndex,
+       tokensReceived
+   );
```

**Dolomite:** [6a8ae06](https://github.com/dolomite-exchange/dolomite-margin/commit/6a8ae061fa84110db7b111512f705a6cd0a472bb) 커밋으로 수정되었습니다.

**Cyfrin:** 해결됨.


### `AdminImpl::ownerWithdrawExcessTokens`는 초과 토큰 인출을 시도하기 전에 해당 마켓의 지불 능력을 확인하지 않음 (`AdminImpl::ownerWithdrawExcessTokens` does not check the solvency of given market before attempting to withdraw excess tokens)

**설명:** [`AdminImpl::ownerWithdrawExcessTokens`](https://github.com/feat/dolomite-margin/blob/e10f14320ece20d7492e8e68400333c5c7dec656/contracts/protocol/impl/AdminImpl.sol#L152-L181)를 사용하면 프로토콜 관리자가 특정 마켓에 대해 Dolomite Margin의 초과 토큰을 인출할 수 있습니다. 초과 토큰은 다음 공식을 사용하여 계산됩니다.

```
초과 토큰 = 토큰 잔액 (L) + 총 대출 (B) - 총 공급 (S)
```

여기서 `L`은 실제 유동성(Dolomite Margin의 실제 토큰 잔액)을 나타냅니다. `B`와 `S`는 가상 Dolomite 잔액입니다. 시간이 지남에 따라 대출자의 이자 수수료를 공급자에게 전달한 후 프로토콜이 수수료를 받으면서 초과 토큰이 증가합니다. 이러한 수수료의 규모는 마켓의 총 미결제 대출과 수익률에 따라 달라집니다.

그러나 특정 시나리오에서는 관리자의 `numExcessTokens` 인출로 인해 프로토콜이 일시적인 지불 불능 상태가 될 수 있습니다. 이는 인출 후 남은 토큰 잔액이 담보화를 조정 후 해당 마켓의 최대 인출 가능 가치보다 낮을 때 발생합니다. 이 상황은 유동성이 적은 마켓이나 단일 엔티티가 유동성의 상당 부분을 제공한 집중 위험이 높은 마켓에서 특히 발생할 가능성이 높습니다.

**파급력:** 이 상황에서 인출 조치는 프로토콜의 잔액 부족으로 인해 서비스 거부(DoS)를 유발할 수 있습니다. 결과는 심각할 수 있지만 수수료는 일반적으로 풀 잔액보다 훨씬 낮기 때문에 발생할 가능성은 미미합니다. 결과적으로 심각도 수준을 **중간(MEDIUM)**으로 평가합니다.

**개념 증명 (Proof of Concept):** _가정:_
- USDC 이자율은 연 10%
- 수익률 = 80% (대출 이자의 20%는 프로토콜 수수료)

아래의 단순화된 시나리오를 고려해 보세요:

| 시간  | 조치 | Alice | Bob  | Pete | USDC 잔액 | ETH 잔액 |  초과 USDC  |
|------|--------|---------|------|------| ------------| --------------| -------------- |
| T =0 | Alice 2 ETH 예치, Bob 5000 USDC 예치    | 2   | 5000  | -  | 5000 | 2 |  0 |
| T=0 | Alice 2000 USDC 대출    | 2 , -2000    | 5000  | -  | 5000 | 2 |  0 |
| T=1년 | 200 USDC 이자 발생    | 2, -2200  | 5160  | -  | 5000 | 2 |  40 |
| T=1년 | Pete 1000 USDC 예치    | 2, -2200  | 5160  | 1000  | 6000 | 2 |  40 |
| T=1년 | Bob 5160 USDC 인출    | 2, -2200  | 0  | 1000  | 840 | 2 |  40 |
| T=1년 | 프로토콜 40 USDC 인출    | 2, -2200  | 0  | 1000  | 800 | 2 |  0 |

이 단계에서 Pete는 자신의 계정에 대한 대출이 없더라도 예치금을 인출할 수 없습니다. 이는 다른 유동성 공급자가 새로운 유동성을 예치할 때까지 프로토콜이 일시적으로 지불 불능 상태이기 때문입니다.

**권장 완화 방안:** 이 문제를 해결하려면 인출을 완료하기 전에 `AdminImpl::ownerWithdrawExcessTokens` 함수에서 각 마켓에 대한 지불 능력 검사를 도입하는 것이 좋습니다. 프로토콜은 초과 토큰 인출로 인해 일시적인 지불 불능이 발생하지 않도록 해야 합니다.

**Dolomite:** 향후 이를 완화할 수 있는 방법(핵심 프로토콜에 대한 코드 변경 없이)은 관리자가 토큰을 인출하고 원자적으로 재예치하도록 하는 것입니다. 이를 통해 관리자는 수익을 "복리화"하여 프로토콜에 유동성을 유지하면서도 관리자가 초과 토큰을 "0으로" 만들 수 있습니다.

**Cyfrin:** 인정함.


### 광범위한 자산에 걸쳐 교차 자산 담보화를 지원하기 위한 시스템적 위험 통제 부족 (Inadequate systemic risk-controls to support cross-asset collateralization across a wide range of assets)

**설명:** 현재 Dolomite Margin 위험 통제는 두 가지 범주로 분류할 수 있습니다.
 - 계정 수준 위험 통제
 - 시스템적 위험 통제

Dolomite는 각 작업 종료 시 최종 상태의 유효성을 포괄적으로 검증하는 강력한 위험 모니터링 아키텍처를 통합합니다. 트랜잭션 모음을 포함하는 작업은 최종 상태의 무결성과 정확성을 보장하기 위해 철저한 시스템 검사를 거칩니다. 이 계산은 [`OperationImpl::_verifyFinalState`](https://github.com/dolomite-exchange/dolomite-margin/blob/e10f14320ece20d7492e8e68400333c5c7dec656/contracts/protocol/impl/OperationImpl.sol#L309)에서 발생하며 여기서 계정 수준 및 시스템 수준 위험이 모두 측정됩니다.

기존의 시스템적 위험 통제는 대출 한도, 공급 한도, 담보 전용 모드, 오라클 센티넬과 같은 다양한 측면을 다루지만 교차 자산 담보화로 인해 발생하는 시스템적 위험은 고려하지 못합니다.

충분한 토큰 지원 없이 가상 유동성을 생성하면 프로토콜이 유동성 압박(squeezes), 동결 및 변동하는 자산 상관관계와 관련된 위험에 노출될 수 있습니다. 이는 전통 금융의 부분 지급준비 은행업(fractional banking)과 유사합니다. 즉, 은행이 예금의 10배 이상을 대출하면 타이트한 유동성 조건으로 인해 지불 불능 위험에 노출됩니다.

Dolomite Margin의 경우 실제 토큰 잔액에 대한 가상 유동성의 비율은 레버리지로 간주될 수 있습니다. 레버리지가 높을수록 지불 불능 위험이 커집니다. 이 가상 유동성이 추가 대출을 위한 담보로 활용되면 레버리지가 더욱 증폭될 수 있습니다.

**파급력:** 더 높은 프로토콜 레버리지는 유동성이 부족한 기간 동안 지불 불능 위험을 직접적으로 증가시킬 수 있습니다. 이러한 위험은 극히 드문 블랙 스완 유형의 이벤트에 기인하므로 심각도를 **중간(MEDIUM)**으로 평가합니다.

**개념 증명 (Proof of Concept):** 다음 이벤트가 전개되는 USDT 디페깅 시나리오를 고려해 보세요:

1. 가상 USDT 보유자는 실제 USDT 유동성으로 전환하기 위해 서두르며 가능한 모든 곳에서 매도하려고 시도합니다.
2. 투기꾼들은 기회를 엿보고 USDT로 교차 담보 대출 포지션을 시작합니다. 짧은 기간 내의 높은 잠재적 수익은 더 높은 활용 수준에서도 가파른 이자율을 간과하게 만들 수 있습니다.
3. 이러한 가상 USDT 토큰은 Dolomite의 내부 풀 내에서 다른 스테이블 토큰으로 신속하게 교환될 수 있습니다.
4. 청산인은 일방적인 마켓에서 잠재적인 높은 슬리피지를 예상하여 USDT를 담보로 하는 포지션 청산을 주저할 수 있습니다.

위의 모든 요인은 연쇄 효과를 유발하여 Dolomite에 상당한 USDT 부실 채권을 축적할 수 있습니다. 대출/공급 한도가 존재하지만, 지원되지 않는 유동성 생성을 억제하기 위한 추가적인 안전장치를 도입하면 Dolomite가 잠재적인 전염 시나리오를 더 잘 관리할 수 있습니다.

**권장 완화 방안:** 시스템의 견고성을 강화하려면 `OperationImpl::_verifyFinalState` 함수에 마켓당 레버리지를 제한하는 시스템적 위험 측정을 추가하는 것을 고려하세요. 유동성이 적은 마켓에 대한 레버리지 상한선을 구현하여 사용자가 충분한 실제 유동성 없이 교차 자산 대출 포지션을 개설하는 능력을 제한하는 것을 고려하세요.

**Dolomite:** 우리는 이것이 Global Operator의 사용을 통해 더 잘 관리된다고 생각합니다. 각 상황에는 뉘앙스가 너무 많습니다. 따라서 강제 종료하거나 포지션을 수정할 수 있는 메커니즘을 추가하는 능력은 거의 모든 상황을 해결하는 데 큰 도움이 될 것입니다.

**Cyfrin:** 인정함. Global Operator는 확실히 신속한 계정 조정을 수행하는 Dolomite의 민첩성을 강화하지만 광범위한 위기 상황에서의 반응성과 효능은 모호합니다. Terra Luna 사건과 같은 과거의 사건은 투기꾼들이 디페그 상황을 자주 악용하여 공격적인 베팅을 함으로써 잠재적으로 프로토콜에 상당한 부실 채권을 안겨준다는 것을 보여주었습니다. 우리는 그러한 드문 전염 시나리오 동안 존재적 위협을 완화하기 위해 지원되지 않는 가상 유동성 생성을 제한하는 추가적인 위험 통제를 계속 권장합니다.


\clearpage
## 낮은 중요도 (Low Risk)


### `Bits::unsetBit`가 0 비트에 적용될 때의 잘못된 로직 (Incorrect logic in `Bits::unsetBit` when applied to a zero bit)

**설명:** `Bits::unsetBit`는 설정 해제할 비트가 1이 아닌 경우 예상대로 작동하지 않을 수 있습니다.

현재 다음과 같이 구현되어 있습니다:

```solidity
function unsetBit(
    uint bitmap,
    uint bit
) internal pure returns (uint) {
    return bitmap - (ONE << bit);
}
```

이 구현은 해당 비트가 이미 1로 설정된 경우에만 올바르게 설정 해제합니다. 비트가 0으로 설정된 경우 이진 뺄셈의 작동 방식으로 인해 이 연산은 대신 1로 설정하고 다른 비트도 수정합니다(의도한 동작일 가능성이 낮음).

비트를 설정 해제하는 더 적절한 방법은 비트 마스크의 보수와 비트 단위 `AND` 연산을 사용하는 것입니다. 수정된 구현은 다음과 같습니다:

```solidity
function unsetBit(
    uint bitmap,
    uint bit
) internal pure returns (uint) {
    return bitmap & ~(ONE << bit);
}
```

이 버전에서 `(ONE << bit)`는 해당 위치의 비트만 1로 설정된 마스크를 생성합니다. `~` 연산자는 이 마스크를 반전시켜 해당 위치의 비트를 0으로 설정하고 다른 모든 비트를 1로 설정합니다. 마지막으로 비트 단위 `AND` 연산은 비트맵의 모든 비트를 변경하지 않고 그대로 두지만 해당 위치의 비트만 설정 해제(0으로 설정)합니다.

**파급력:** 이 함수는 [두](https://github.com/feat/dolomite-margin/blob/e10f14320ece20d7492e8e68400333c5c7dec656/contracts/external/helpers/LiquidatorProxyBase.sol#L431-L441) [인스턴스](https://github.com/feat/dolomite-margin/blob/e10f14320ece20d7492e8e68400333c5c7dec656/contracts/protocol/lib/Storage.sol#L919-L935) 모두에서 `Bits::getLeastSignificantBit`가 먼저 호출되어 설정된 비트로만 호출되는 것으로 보입니다. 그렇지 않았다면 이 발견 사항은 훨씬 더 큰 영향을 미쳤을 것이지만 심각도를 **낮음(LOW)**으로 평가합니다.

**권장 완화 방안:** 위의 수정된 버전의 함수를 사용하세요.

**Dolomite:** [6a8ae06](https://github.com/dolomite-exchange/dolomite-margin/commit/6a8ae061fa84110db7b111512f705a6cd0a472bb) 커밋으로 수정되었습니다.

**Cyfrin:** 인정함.


### OpenZeppelin v2.5.1은 중요하지 않은 보안 패치에 대해 지원되지 않음 (OpenZeppelin v2.5.1 is not supported for non-critical security patches)

Dolomite Margin 스마트 컨트랙트는 현재 OpenZeppelin 컨트랙트 v2.5.1을 사용하고 있는 반면 최신 릴리스는 v4.9.2입니다. [OpenZeppelin 보안 정책](https://github.com/OpenZeppelin/openzeppelin-contracts/security?page=1#supported-versions)에 따라 심각한 중요도의 버그 수정만 과거 주요 릴리스로 백포트됩니다. 이에 비추어 볼 때 향후 버그 보고서나 프로토콜 변경으로 인해 취약점이 도입될 수 있으므로 보다 최신 버전을 사용하는 것이 좋습니다.

**Dolomite:** Solidity v5를 사용하는 나머지 Dolomite 스마트 컨트랙트와의 호환성을 유지하기 위해 최신 메이저 버전인 OpenZeppelin v2를 사용합니다. 스마트 컨트랙트를 업그레이드하기로 결정하면 OpenZeppelin 버전도 확실히 올릴 것입니다.

**Cyfrin:** 인정함.


### `CallImpl::call`에서 외부 호출 악용 가능성 (It may be possible to exploit external call in `CallImpl::call`)

**설명:** [`CallImpl::call`](https://github.com/feat/dolomite-margin/blob/e10f14320ece20d7492e8e68400333c5c7dec656/contracts/protocol/impl/CallImpl.sol#L39)에는 다음 로직이 포함되어 있습니다:
```solidity
state.requireIsOperator(args.account, msg.sender);
ICallee(args.callee).callFunction(
    msg.sender,
    args.account,
    args.data
);
```
발신자가 해당 계정의 운영자라고 가정할 때 `DolomiteMargin`의 컨텍스트에서 임의의 데이터를 사용하여 특정 수신자(callee)에 대한 선택기 충돌(selector clashing) 및/또는 폴백 함수를 트리거할 수 있습니다. `DolomiteMargin` 라이브러리는 독립 실행형 컨트랙트로 배포되므로 호출 컨텍스트에서 `OperationImpl`이 `msg.sender`가 될 것으로 보입니다.

다행히 충돌하는 선택기가 없고 WETH 폴백 함수는 단순히 `msg.value`를 예치하려고 시도하므로 `DepositWithdrawalProxy::initializeETHMarket`의 [무한 WETH 승인](https://github.com/feat/dolomite-margin/blob/e10f14320ece20d7492e8e68400333c5c7dec656/contracts/external/proxies/DepositWithdrawalProxy.sol#L102)을 악용하는 데 사용할 수 없습니다.

**파급력:** 이 발견 사항은 특정 상황에서 심각한 영향을 미칠 수 있지만 여러 외부 가정에 따라 달라지므로 가능성이 크게 줄어들어 심각도를 **낮음(LOW)**으로 평가합니다.

**권장 완화 방안:** `args.callee`로 사용할 수 있는 신뢰할 수 있는 컨트랙트를 화이트리스트에 추가하는 것을 고려하세요.

**Dolomite:** 우리는 구현을 가능한 한 무허가(permissionless) 상태로 유지하고 싶습니다. 우리가 시행할 수 있는 다른 가능한 확인이 있다면 기꺼이 검토하겠습니다.

**Cyfrin:** 인정함.


### `LiquidateOrVaporize::liquidate` 및 `LiquidateOrVaporize::vaporize`에서 Checks Effects Interactions 패턴 위반으로 인한 읽기 전용 재진입 취약점 (Violation of the Checks Effects Interactions Pattern in `LiquidateOrVaporize::liquidate` and `LiquidateOrVaporize::vaporize` could result in read-only reentrancy vulnerabilities)

**설명:** 담보 부족 계정을 청산하거나 소멸(vaporize)시킬 때, [SafeLiquidationCallback::callLiquidateCallbackIfNecessary](https://github.com/feat/dolomite-margin/blob/e10f14320ece20d7492e8e68400333c5c7dec656/contracts/protocol/lib/SafeLiquidationCallback.sol#L44)가 호출되어 실행을 청산 계정 소유자에게 일시적으로 [양도](https://github.com/feat/dolomite-margin/blob/e10f14320ece20d7492e8e68400333c5c7dec656/contracts/protocol/lib/SafeLiquidationCallback.sol#L53)하는 [여러](https://github.com/feat/dolomite-margin/blob/e10f14320ece20d7492e8e68400333c5c7dec656/contracts/protocol/impl/LiquidateOrVaporizeImpl.sol#L124-L130) [사례](https://github.com/feat/dolomite-margin/blob/e10f14320ece20d7492e8e68400333c5c7dec656/contracts/protocol/impl/LiquidateOrVaporizeImpl.sol#L144-L150)가 [있습니다](https://github.com/feat/dolomite-margin/blob/e10f14320ece20d7492e8e68400333c5c7dec656/contracts/protocol/impl/LiquidateOrVaporizeImpl.sol#L262-L268). [가스 그리핑(gas-griefing) 및 반환 데이터 폭탄 공격을 완화하기 위한 예방 조치](https://github.com/feat/dolomite-margin/blob/e10f14320ece20d7492e8e68400333c5c7dec656/contracts/protocol/lib/SafeLiquidationCallback.sol#L54-L55)가 마련되어 있지만, 이 로직은 Dolomite Margin의 상태에 의존할 수 있는 타사 프로토콜의 취약점을 방지하지 못합니다. 상태 잔액 업데이트는 이러한 호출 후에 수행되므로 인터페이스에서 [인정](https://github.com/feat/dolomite-margin/blob/e10f14320ece20d7492e8e68400333c5c7dec656/contracts/protocol/interfaces/ILiquidationCallback.sol#L34-L36)되고 전역 프로토콜 수준 재진입 가드로 인해 직접 악용될 수는 없지만 통합 프로토콜을 읽기 전용 재진입 공격 벡터에 노출시킬 수 있습니다.

**파급력:** Dolomite Margin에는 즉각적인 영향이 없지만 이 발견 사항은 중간 프로토콜 상태의 영향을 받을 수 있는 다른 생태계 프로토콜에 영향을 미칠 수 있으므로 심각도를 **낮음(LOW)**으로 평가합니다.

**권장 완화 방안:** 안전하지 않은 외부 호출을 하기 전에 상태 잔액을 업데이트하세요.

**Dolomite:** 사용자의 가상 잔액을 수정하지만 ERC20 전송 이벤트를 구체화하지 않는 모든 작업에 콜백 메커니즘을 추가하고 있습니다. 이를 표준화하기 위해 작업의 이벤트가 기록되기 전에 추가되었습니다. 해당 작업은 `Transfer`, `Trade`, `Liquidate`, `Vaporize`입니다. 또한 관리자가 콜백 함수에 할당할 가스 양을 설정할 수 있는 `callbackGasLimit`라는 변수를 추가했습니다. 이를 통해 가스가 동일한 방식으로 측정되지 않을 수 있는 배포에 대해 더 많은 제어가 가능합니다.

[6a8ae06](https://github.com/dolomite-exchange/dolomite-margin/commit/6a8ae061fa84110db7b111512f705a6cd0a472bb) 커밋으로 수정 사항이 추가되었습니다.

**Cyfrin:** 인정함. 지정된 커밋에서 우리는 이 감사 우려 사항과 무관한 변경 사항, 특히 콜백과 관련된 변경 사항 및 이전의 하드코딩된 가스를 대체하는 callbackGasLimit의 도입을 확인했습니다. 우리의 검토는 이러한 수정의 더 넓은 의미를 파헤치지 않고 청산 및 소멸 함수에서 CEI 패턴의 일관성에만 초점을 맞췄습니다.



### `AdminImpl::ownerSetAccountMaxNumberOfMarketsWithBalances`에서 최댓값 검증 부족 (Lack of max value validation in `AdminImpl::ownerSetAccountMaxNumberOfMarketsWithBalances`)

**설명:** 현재 관리자가 [`AdminImpl::ownerSetAccountMaxNumberOfMarketsWithBalances`](https://github.com/feat/dolomite-margin/blob/e10f14320ece20d7492e8e68400333c5c7dec656/contracts/protocol/impl/AdminImpl.sol#L403)를 호출할 때 상한보다 큰 값을 설정하지 못하도록 하는 보호 장치가 없으며, 2 이상이어야 한다는 조건만 있습니다:

```solidity
Require.that(
    accountMaxNumberOfMarketsWithBalances >= 2,
    FILE,
    "Acct MaxNumberOfMarkets too low"
);
```

이는 방지하려는 서비스 거부 시나리오로 이어질 수 있으므로 추가적인 최대 검증이 있어야 합니다.

**파급력:** 관리자 오류에 의존하므로 가능성이 낮아 심각도를 **낮음(LOW)**으로 평가합니다.

**권장 완화 방안:** 소유자가 주어진 계정에 대해 허용된 최대 잔액 보유 마켓 수를 설정할 때 상한에 대한 추가 검증을 추가하세요.

**Dolomite:** [6a8ae06](https://github.com/dolomite-exchange/dolomite-margin/commit/6a8ae061fa84110db7b111512f705a6cd0a472bb) 커밋으로 수정되었습니다. 64개 마켓의 상한을 추가했습니다.

**Cyfrin:** 인정함.


### 글로벌 운영자 설정 시 불충분한 확인 및 높은 신뢰 가정 (Inadequate checks and high trust assumptions while setting global operators)

**설명:** Dolomite는 다른 사용자를 대신하여 작업을 수행할 수 있는 권한이 부여된 계정인 `Global Operators` 개념을 도입합니다. 이 계정의 목적은 지속적인 트랜잭션 승인의 필요성을 최소화하여 사용자 경험을 향상시키는 것입니다. 그러나 특권 액세스 권한을 가진 악성 글로벌 운영자와 관련된 잠재적 위험을 인식하는 것이 중요합니다. 코드의 두 가지 인스턴스가 이러한 위험을 강조합니다:

특권 액세스로 인해 악성 글로벌 운영자는 잠재적으로 많은 피해를 입힐 수 있습니다. 코드에서 관찰된 두 가지 사례는 다음과 같습니다:

1. 글로벌 운영자는 `BorrowPositionProxyV2::openBorrowPositionWithDifferentAccounts` 함수를 사용하여 사용자를 대신하여 새로운 대출 포지션을 생성할 수 있습니다.

2. `IAutoTrader`를 구현하는 글로벌 운영자는 사용자의 마켓 메이커 역할을 하여 테이커(taker) 상대방과 거래 활동에 참여할 수 있습니다.

프로토콜 팀과의 논의를 통해 특정 컨트랙트만 Global Operator 역할이 할당되며 외부 소유 계정(EOA)을 Global Operator로 지정할 의도가 없음이 명확해졌습니다. 그러나 글로벌 운영자가 행사하는 상당한 통제력을 고려할 때 이 역할을 할당하기 위한 현재 확인 및 제어는 불충분합니다.

**파급력:** 악성 글로벌 운영자는 무단 거래에 참여하거나 사용자 계정에 대해 대출함으로써 자금을 조작할 수 있는 능력이 있습니다.

**권장 완화 방안:** 시스템 보안을 강화하려면 글로벌 운영자를 할당할 때 다음 추가 확인을 권장합니다:

- 글로벌 운영자가 외부 소유 계정(EOA)이 아님을 명시적으로 확인하세요.
- 새로운 글로벌 운영자를 등록할 때 타임락 메커니즘을 구현하세요.
- 화이트리스트 메커니즘을 구축하여 화이트리스트에 있는 유니버스의 특정 컨트랙트만 글로벌 운영자로 할당되도록 허용하세요.

**Dolomite:** Dolomite는 핵심 수준에서 일반적(generic)인 것을 극대화하며 입력 검증을 위해 가능한 한 많은 안전장치를 마련하려고 노력했습니다.

프로토콜은 현재 지연된 멀티 시그(multi sig)에 의해 관리되며 결국 타임락된 DAO(현재 구현이 완료되지 않음)에 의해 관리됩니다. 이를 통해 프로토콜은 모든 것을 시간 제어하고 소유자에 따라 프로토콜의 필요에 맞게 세부 사항을 남겨둘 수 있습니다.

EOA 확인을 추가하지 않을 것입니다. 왜냐하면 create2 주소를 글로벌 운영자로 설정하는 유효한 사용 사례가 향후에 있을 수 있으며, 제안된 확인으로는 더 이상 불가능하기 때문입니다.

어떤 이유로든 프로토콜의 제어권이 탈취되면 탈취자는 악성 운영자 컨트랙트를 생성할 수 있으며 이는 어차피 EOA만큼 나쁠 것입니다.

**Cyfrin:** 인정함.

\clearpage
## 정보성 (Informational)


### 반복되는 로직은 공유 내부 함수로 재사용 가능 (Repeated logic can be reused with a shared internal function)

`BorrowPositionProxyV1::openBorrowPosition`과 `BorrowPositionProxyV1::transferBetweenAccounts` 모두 다음과 같은 반복 코드를 포함합니다:
```solidity
AccountActionLib.transfer(
    DOLOMITE_MARGIN,
    /* _fromAccountOwner = */ msg.sender, // solium-disable-line
    _fromAccountNumber,
    /* _toAccountOwner = */ msg.sender, // solium-disable-line
    _toAccountNumber,
    _marketId,
    Types.AssetAmount({
        sign: false,
        denomination: Types.AssetDenomination.Wei,
        ref: Types.AssetReference.Delta,
        value: _amountWei
    }),
    _balanceCheckFlag
);
```
바이트코드 크기를 줄이기 위해 이를 공유 내부 함수로 이동하는 것을 고려하세요.

**Dolomite:** [6a8ae06](https://github.com/dolomite-exchange/dolomite-margin/commit/6a8ae061fa84110db7b111512f705a6cd0a472bb) 커밋으로 수정되었습니다.

**Cyfrin:** 인정함.


### `BorrowPositionProxyV2`는 승인된 글로벌 운영자가 일방적으로 포지션을 수정하고 계정 간 부채를 이전하도록 허용함 (`BorrowPositionProxyV2` allows authorized global operators to unilaterally modify positions and transfer debt between accounts)

`BorrowPositionProxyV2`에는 승인된 발신자가 호출할 때 다른 계정으로 대출 포지션을 개설/종료/이전/상환하는 함수가 포함되어 있습니다. 이것이 글로벌 운영자와 이 특정 프록시를 사용하는 프로토콜의 의도된 동작이지만, 승인된 운영자가 한 계정에서 다른 계정으로 부채를 일방적으로 이전할 수 있다는 점은 주목할 가치가 있습니다.

**Dolomite:** 우리는 이 기능을 사용하여 동일한 사용자가 소유한 계정의 경계를 넘습니다. 예를 들어 격리 모드(Isolation Mode) 볼트는 본질적으로 EOA가 소유한 스마트 컨트랙트 지갑입니다. 우리는 사용자가 자신의 계정에서 볼트로 자금을 이체할 수 있도록 BorrowPositionProxyV2를 사용합니다.

**Cyfrin:** 인정함.


### 프로젝트 README.md의 불일치 (Inconsistencies in project README.md)

프로젝트 README에는 `numberOfMarketsWithBorrow` 필드가 `Account.Storage`에 추가되었다고 명시되어 있지만 `numberOfMarketsWithDebt`여야 합니다.

또한 청산이 글로벌 운영자에게서 오도록 강제하는 require 문이 `OperationImpl`에 추가되었다고 명시되어 있지만 `LiquidateOrVaporizeImpl`이어야 합니다.

마찬가지로 만료가 글로벌 운영자에게서 오도록 강제하는 require 문은 `OperationImpl`이 아니라 `TradeImpl::trade`에 추가되었습니다.

이러한 변경 사항을 더 쉽게 탐색할 수 있도록 관련 코드 줄에 링크를 추가하는 것을 고려하세요.

**Dolomite:** [6a8ae06](https://github.com/dolomite-exchange/dolomite-margin/commit/6a8ae061fa84110db7b111512f705a6cd0a472bb) 커밋으로 수정되었습니다.

**Cyfrin:** 인정함.


### 외부 범위 구현의 로직 추상화로 인한 검증 불가능한 컨트랙트 (Unverifiable contracts resulting from logic abstraction in externally scoped implementations)

**설명:** Dolomite는 여러 토큰에 걸쳐 다양한 작업을 용이하게 하는 다목적 기본 레이어입니다. 현재 감사 범위에는 `GenericTraderProxyV1.sol` 및 `LiquidatorProxyV4WithGenericTrader.sol`과 같은 컨트랙트가 포함됩니다. 이러한 컨트랙트는 `GenericTraderProxyBase::_appendTraderActions`와 같은 주요 함수 내에서 `IIsolationModeUnwrapperTrader::createActionsForUnwrapping` 및 `IIsolationModeWrapperTrader::createActionsForWrapping`과 같은 인터페이스에 의존합니다. 마찬가지로 인터페이스 함수 `IIsolationModeToken.isTokenConverterTrusted`는 `GenericTraderProxyBase::_validateIsolationModeStatusForTraderParam`에서 활용됩니다.

감사 기간 동안 래퍼/언래퍼 트레이더 구현이 없어 래핑 및 언래핑 작업 생성 이면의 정확한 로직을 검증하는 능력이 방해를 받았습니다. 이러한 작업의 잘못된 순서 지정이나 실행은 언급된 컨트랙트의 비즈니스 로직에 영향을 미치는 취약점을 도입할 수 있습니다. 또한 감사 범위 내에 격리 모드 토큰 컨트랙트가 없어 토큰 변환기 신뢰 이면의 근거를 검토할 수 없었습니다. 결과적으로 우리는 특정 프록시 컨트랙트의 중요한 기능을 검증할 수 없습니다.

**파급력:** 이러한 인터페이스와 상호 작용하면 현재 이해할 수 없는 예상치 못한 문제가 발생할 수 있습니다.

**권장 완화 방안:** 이를 해결하려면 누락된 컨트랙트 및 함수에 대한 필요한 구현을 제공하는 것이 중요합니다.

**Dolomite:** 인정함.

**Cyfrin:** 인정함.

\clearpage

