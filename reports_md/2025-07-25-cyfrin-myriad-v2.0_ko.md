**수석 감사**

[Immeas](https://twitter.com/0ximmeas)

[Chinmay](https://x.com/dev_chinmayf)

[InAllHonesty](https://x.com/0xInAllHonesty)

---

# 발견 사항

## 낮은 위험 (Low Risk)

### `PredictionMarketV3_4::getMarketResolvedOutcome`에서 모호한 `-1` 반환 값

**설명:** [`PredictionMarketV3_4::getMarketResolvedOutcome`](https://github.com/Polkamarkets/polkamarkets-js/blob/24f1394be94d27433d2e3a7370442126e1c1e5ba/contracts/PredictionMarketV3_4.sol#L1336-L1345)에는 시장 해결이 아직 보류 중임을 나타내는 로직이 있습니다:

```solidity
function getMarketResolvedOutcome(uint256 marketId) public view returns (int256) {
  Market storage market = markets[marketId];

  // 시장이 아직 해결되지 않은 경우 -1 반환
  if (market.state != MarketState.resolved) {
    return -1;
  }

  return int256(market.resolution.outcomeId);
}
```

이 함수는 시장이 아직 `resolved` 상태에 도달하지 않은 경우 `-1`을 반환합니다. 그러나 질문이 Reality.eth(Realitio)를 통해 해결되면 반환된 결과는 Reality.eth가 유효하지 않은 답변을 알리는 데 사용하는 `0xffff...ffff`(`type(uint256).max`)일 수 있습니다. 이는 그들의 [문서](https://reality.eth.limo/app/docs/html/contracts.html#fetching-the-answer-to-a-particular-question)에 언급되어 있습니다:

> 관례상 지원되는 모든 유형은 0xff...ff(일명 bytes32(type(uint256).max))를 사용하여 "유효하지 않음"을 의미합니다.

`int256(type(uint256).max)`는 `-1`과 같으므로 유효하지 않은 답변으로 해결된 시장도 `getMarketResolvedOutcome()`에서 `-1`을 반환합니다. 이는 모호성을 생성합니다: `-1`을 "미해결"로 해석하는 서비스는 유효하지 않은 해결을 시장이 아직 보류 중인 것처럼 실수로 처리할 수 있습니다.

**파급력:** `getMarketResolvedOutcome()`이 시장이 미해결일 때만 `-1`을 반환한다고 의존하는 소비자는 상태를 오해할 수 있습니다. 시장은 Reality.eth로부터 유효하지 않은 답변으로 해결되었지만 재사용된 센티널 값으로 인해 여전히 미해결 상태로 보일 수 있습니다.

**권장 완화 방법:** 미해결 상태를 나타내기 위해 다른 센티널 값을 반환하는 것을 고려하십시오. 결과의 수는 [32개로 제한](https://github.com/Polkamarkets/polkamarkets-js/blob/24f1394be94d27433d2e3a7370442126e1c1e5ba/contracts/PredictionMarketV3_4.sol#L102)되어 있으므로 32 이상의 숫자는 별개의 "미해결" 표시기 역할을 할 수 있습니다. 그러나 Reality.eth가 미해결 답변을 나타내는 데 사용하는 또 다른 [특수 값](https://github.com/RealityETH/reality-eth-monorepo/blob/0d2dc425f700aaf514f1927384dd6fe0015e8dba/packages/contracts/development/contracts/RealityETH_ERC20-3.0.sol#L26-L28)인 `-2`는 피하십시오.

**Myriad:** [PR](https://github.com/Polkamarkets/polkamarkets-js/pull/84), 커밋 [`27ccc51`](https://github.com/Polkamarkets/polkamarkets-js/pull/84/commits/27ccc51250696ae5ef379397653d7dd20019529f)에서 수정되었습니다.

**Cyfrin:** 확인됨. 이제 `-3`이 미해결 결과로 사용됩니다.

### `block.timestamp > markets[marketId].closesAtTimestamp == true`인 경우에도 시장이 'Open' 상태에 갇힘

**설명:** `_buy`, `_sell`, `_addLiquidity` 또는 `_removeLiquidity` 함수는 `timeTransitions(marketId)` 수정자 다음에 `atState(marketId, MarketState.open)`을 사용합니다.

누군가 `block.timestamp > markets[marketId].closesAtTimestamp && markets[marketId].state == MarketState.open` 컨텍스트에서 이러한 작업 중 하나를 수행하려고 하면 코드는 `_nextState(marketId)`를 호출하여 상태를 `open`에서 `closed`로 변경합니다: `market.state = MarketState(uint256(market.state) + 1)`. 그러나 다음 수정자 `atState(marketId, MarketState.open)`이 실행되면 모든 것이 되돌려집니다(revert). 시장 상태 변경은 지속되지 않아 `markets[marketId].state = MarketState.open` 상태로 남습니다.

게다가 `timeTransitions`가 해당 컨텍스트에서 `_nextState` 전환을 성공적으로 실행하는 유일한 때는 `resolveMarketOutcome` 내부입니다. 이로 인해 `closed` 상태는 `resolveMarketOutcome` 함수의 수정자 실행 사이에 일어나는 `open`과 `resolved` 사이의 일시적인 단계일 뿐입니다.

**파급력:** 시장은 종료되어야 할 때 `block.timestamp > markets[marketId].closesAtTimestamp == true`인 경우에도 `open` 상태로 유지되어 잠재적으로 UX 문제를 일으킬 수 있습니다.

`_buy`, `_sell`, `_addLiquidity` 또는 `_removeLiquidity`를 시도하는 모든 tx는 `atState(marketId, MarketState.open)` 수정자에 의해 되돌려져 호출자의 가스를 낭비하게 됩니다.

**개념 증명 (Proof of Concept):** `PredictionMarket.t.sol` 내부에 다음 테스트를 추가하십시오:
```solidity
    function test_MarketStuckInOpenState() public {
        // Create market that closes in 1 hour
        uint32 closeTime = uint32(block.timestamp + 3600);

        PredictionMarketV3_4.CreateMarketDescription memory desc = PredictionMarketV3_4.CreateMarketDescription({
            value: VALUE,
            closesAt: closeTime,
            outcomes: 2,
            token: IERC20(address(tokenERC20)),
            distribution: new uint256[](0),
            question: "PoC Market State Issue",
            image: "test",
            arbitrator: address(0x1),
            buyFees: PredictionMarketV3_4.Fees({fee: 0, treasuryFee: 0, distributorFee: 0}),
            sellFees: PredictionMarketV3_4.Fees({fee: 0, treasuryFee: 0, distributorFee: 0}),
            treasury: treasury,
            distributor: distributor,
            realitioTimeout: 3600,
            manager: IPredictionMarketV3Manager(address(manager))
        });

        uint256 marketId = predictionMarket.createMarket(desc);

        // Verify market is initially open
        (PredictionMarketV3_4.MarketState state, uint256 closesAt,,,,) = predictionMarket.getMarketData(marketId);
        assertEq(uint256(state), uint256(PredictionMarketV3_4.MarketState.open));
        assertEq(closesAt, closeTime);

        console.log("BEFORE TIME WARP");
        console.log("Market state (0=open, 1=closed, 2=resolved):", uint256(state));
        console.log("Time until close:", closesAt - block.timestamp);

        // TIME WARP: Move 2 hours into the future (1 hour past close time)
        vm.warp(block.timestamp + 7200);

        console.log("AFTER TIME WARP");
        console.log("Market closes at:", closesAt);
        console.log("Time past close:", block.timestamp - closesAt);

        (state,,,,,) = predictionMarket.getMarketData(marketId);
        console.log("Market state (should be 1=closed, but shows):", uint256(state));

        // Prove the market is logically closed but state is wrong
        assertTrue(block.timestamp > closesAt, "We are past close time");
        assertEq(uint256(state), uint256(PredictionMarketV3_4.MarketState.open), "Market incorrectly shows as open!");

        // ATTEMPT TO TRADE: This should fail due to modifier ordering issue
        console.log("ATTEMPTING TRADE (should fail)");

        address trader = makeAddr("trader");
        deal(address(tokenERC20), trader, 1 ether);

        vm.startPrank(trader);
        tokenERC20.approve(address(predictionMarket), type(uint256).max);
        vm.expectRevert(bytes("!ms")); // Market state error
        predictionMarket.buy(marketId, 0, 0, 0.1 ether);
        vm.stopPrank();

        console.log("Trade failed as expected due to modifier");
    }
```

**권장 완화 방법:** `closed` 상태를 제거하고 `closesAtDate`를 사용하여 시장이 열려 있는지/닫혀 있는지 나타내거나 전환이 작동하도록 만드는 것을 고려하십시오.

**Myriad:** [PR#85](https://github.com/Polkamarkets/polkamarkets-js/pull/85), 커밋 [`627d5be`](https://github.com/Polkamarkets/polkamarkets-js/pull/85/commits/627d5be4f706f6d96bdfad924d26ed83c322317b)에서 수정되었습니다.

**Cyfrin:** 확인됨. `getMarketData`는 이제 `market.state == MarketState.open && block.timestamp >= market.closesAtTimestamp`인 경우 `closed`를 반환하는 새 함수 `getMarketState`를 호출합니다.

### `closesAtTimestamp`에 바로 수행되는 작업

**설명:** 프로토콜은 다음 수정자를 통해 모든 사용자 대면 메서드의 시간 유효성을 확인합니다:

```solidity
  modifier timeTransitions(uint256 marketId) {
    if (block.timestamp > markets[marketId].closesAtTimestamp && markets[marketId].state == MarketState.open) {
      _nextState(marketId);
    }
    _;
  }
```

이 조건 `block.timestamp > markets[marketId].closesAtTimestamp`는 `block.timestamp = markets[marketId].closesAtTimestamp`인 경우를 제외하므로, 시장 종료 시 바로 buy, sell, addLiquidity 및 removeLiquidity 작업을 허용합니다.

**파급력:** 시장 상태 및 정답에 따라 위험 없는 수익 가능성.

**개념 증명 (Proof of Concept):** `PredictionMarket.t.sol`에 다음을 추가하십시오

```solidity
    function testTradeRightAtMarketClose() public {
        uint256 marketId = _createTestMarket();

        // Get market close time
        (,uint256 closesAt,,,,) = predictionMarket.getMarketData(marketId);

        // Set timestamp to exact close time
        vm.warp(closesAt);

        // This should fail - market transitions to closed due to timeTransitions modifier // NB it doesn't
        // vm.expectRevert(bytes("!ms")); // Market state error
        predictionMarket.buy(marketId, 0, 0, VALUE);
    }


    function testLiquidityOperationsAtMarketClose() public {
        uint256 marketId = _createTestMarket();

        // Get market close time
        (,uint256 closesAt,,,,) = predictionMarket.getMarketData(marketId);

        // Add more liquidity before close
        vm.warp(closesAt - 1);
        predictionMarket.addLiquidity(marketId, VALUE);

        // Try to add liquidity at exact close time - should fail // NB it doesn't
        vm.warp(closesAt);
        // vm.expectRevert(bytes("!ms"));
        predictionMarket.addLiquidity(marketId, VALUE);
    }
```

**권장 완화 방법:**
```diff
  modifier timeTransitions(uint256 marketId) {
--    if (block.timestamp > markets[marketId].closesAtTimestamp && markets[marketId].state == MarketState.open) {
++    if (block.timestamp >= markets[marketId].closesAtTimestamp && markets[marketId].state == MarketState.open) {
      _nextState(marketId);
    }
    _;
  }
```

**Myriad:** [PR#79](https://github.com/Polkamarkets/polkamarkets-js/pull/79), 커밋 [`774161e`](https://github.com/Polkamarkets/polkamarkets-js/pull/79/commits/774161e4d0a1ab16a38d6b7eda97534bce9e1f94)에서 수정되었습니다.

**Cyfrin:** 확인됨. 확인이 이제 `>=`입니다.

### 유동성 함수에 슬리피지 보호 부족

**설명:** `addLiquidity` 및 `removeLiquidity` 함수에는 `minSharesOut` 또는 `minTokensOut` 매개변수와 같은 슬리피지 보호 형태가 포함되어 있지 않습니다. 즉, 사용자는 견적과 실행 사이의 풀 상태 변경으로 인해 예상보다 훨씬 적은 주식이나 토큰을 받는 것에 대해 보호할 수 없습니다.

**파급력:** 슬리피지 경계가 없으면 사용자는 오프체인에서 예상 결과를 계산한 시점과 트랜잭션이 실행되는 시점 사이에 가격이 변동할 경우 불리한 결과에 노출됩니다. 이 위험은 공개 멤풀이 없는 체인(현재 프로토콜이 목표로 함)에서는 완화되지만, 트랜잭션 가시성이 공개된 다른 EVM 체인에서는 MEV 행위자나 프론트러너가 이러한 격차를 악용할 수 있으므로 관련이 있습니다.

**개념 증명 (Proof of Concept):** `PredictionMarket.t.sol`에 다음 테스트를 추가하십시오:
```solidity
   function testFrontRunAddLiquidity() public {
        uint256 marketId = _createTestMarket();

        address alice = makeAddr("Alice");
        address bob = makeAddr("Bob");
        deal(address(tokenERC20), alice, 1 ether);
        deal(address(tokenERC20), bob, 1 ether);

        // bob sees that alice will add liquidity and buys shares to manipulate the price
        vm.startPrank(bob);
        tokenERC20.approve(address(predictionMarket), type(uint256).max);
        predictionMarket.buy(marketId, 1, 0, 10e16);
        vm.stopPrank();

        // alice adds liquidity
        vm.startPrank(alice);
        tokenERC20.approve(address(predictionMarket), type(uint256).max);
        predictionMarket.addLiquidity(marketId, 1e16);
        vm.stopPrank();

        // bob can now sell for more gaining ~30% of Alice deposit
        vm.prank(bob);
        predictionMarket.sell(marketId, 1, 10.3e16, type(uint256).max);
    }
```

**권장 완화 방법:** `addLiquidity` 및 `removeLiquidity` 모두에 선택적 슬리피지 매개변수를 추가하는 것을 고려하십시오. 예:

```solidity
function addLiquidity(uint256 marketId, uint256 amount, uint256 minSharesOut) external;
function removeLiquidity(uint256 marketId, uint256 shares, uint256 minTokensOut) external;
```

이러한 안전 장치를 통해 사용자는 예상 결과에 따라 실행을 제한하고 프로토콜이 다른 체인으로 확장될 경우 안전성을 향상시킬 수 있습니다.

**Myriad:** [PR#88](https://github.com/Polkamarkets/polkamarkets-js/pull/88), 커밋 [`37e9e89`](https://github.com/Polkamarkets/polkamarkets-js/pull/88/commits/37e9e8912b6a8aee670bcc0002609df099cd7685)에서 수정되었습니다.

**Cyfrin:** 확인됨. `addLiquidity`는 이제 `minSharesIn` 매개변수를 받고 `removeLiquidity`는 `minValue` 매개변수를 받습니다.

### 사용자 대면 호출에 `deadline` 매개변수 추가 고려

**설명:** `buy`, `sell`, `addLiquidity`, `removeLiquidity` 함수는 현재 트랜잭션이 실행되어야 하는 시점을 제한하는 `deadline` 매개변수를 허용하지 않습니다. 마감일이 없으면 특히 제출이나 릴레이에 지연이 있는 경우 트랜잭션이 예상치 못한 시간에 채굴되어 예상치 못한 가격 책정이나 시장 상태 변경으로 이어질 수 있습니다.

**파급력:** 의도한 것보다 늦게 실행되는 트랜잭션은 특히 변동성이 크거나 얇은 시장에서 사용자에게 불리한 결과를 초래할 수 있습니다. 마감일을 포함하면 작업이 예상 시간 창 내에서만 실행되도록 하여 예측 가능성과 사용자 신뢰를 향상시킵니다.

**권장 완화 방법:** 사용자 대면 함수에 `uint256 deadline` 매개변수를 추가하고 수정자를 사용하여 강제하십시오:

```solidity
modifier onlyBefore(uint256 deadline) {
    require(block.timestamp <= deadline, "!d");
    _;
}
```
이렇게 하면 사용자가 지정한 마감일 전에 채굴되지 않으면 트랜잭션이 되돌려집니다.

**Myriad:** 인지함. 이 추가 검증 계층이 도움이 되겠지만, 언급된 사용 사례를 피하기 위한 다른 메커니즘이 이미 마련되어 있다고 믿습니다:

- 슬리피지는 이미 다음을 통해 구성 가능합니다:
  - `buy` - `minOutcomeSharesToBuy`
  - `sell` -  `maxOutcomeSharesToSell`
  - `addLiquidity` - `minSharesIn` ([#88](https://github.com/Polkamarkets/polkamarkets-js/pull/88)에서 구현됨)
  - `removeLiquidity` - `minValueOut` ([#88](https://github.com/Polkamarkets/polkamarkets-js/pull/88)에서 구현됨)

이러한 설정은 변동성/예상치 못한 가격 변화로부터 사용자를 보호합니다.

시장 상태 변경으로 인한 문제를 방지하기 위해 `timeTransitions` 수정자가 작동하여 `deadline` 없이도 트랜잭션을 우아하게 되돌립니다.

제 주된 우려는 언급된 문제를 방지하는 메커니즘이 함수 인수에 이미 있을 때 모든 함수에 추가 인수가 필요한 복잡성을 추가하는 것입니다.

### 먼지 주식을 피하기 위해 주식 기반 매도 기능 추가 고려

**설명:** 현재 인터페이스는 사용자가 주식을 매도할 때 (매도할 주식이 아닌) 원하는 출력 금액(토큰)을 지정하도록 요구합니다. 이로 인해 사용자가나 통합자가 모든 주식을 정확하게 매도하기 어렵게 만듭니다. 특히 실행 중 가격이 변동할 때 더욱 그렇습니다.

많은 경우 사용자는 먼저 `calcSellAmount()`와 같은 뷰 함수를 사용하여 전체 주식 잔액에 대해 받을 토큰 수를 계산해야 합니다. 그러나 가격 민감도와 동적 풀 상태로 인해 이 오프체인 계산은 트랜잭션이 제출될 때쯤에는 구식이 될 수 있습니다.

**파급력:** 사용자가 정확한 수의 주식을 매도할 수 없기 때문에 계정에 소량의 잔여 먼지 잔액이 남게 될 가능성이 높습니다. 이 먼지는 종종 가스를 들여 매도하거나 청구할 가치가 없으므로 사실상 회수할 수 없으며 시간이 지남에 따라 사용자 잔액을 어지럽힙니다.

**권장 완화 방법:** 다음과 같은 새 함수를 추가하는 것을 고려하십시오:

```solidity
function sellShares(uint256 marketId, uint256 outcome, uint256 shareAmount, uint256 minAmountOut) external;
```

이를 통해 사용자는 지정된 수의 주식을 직접 매도하여 오프체인 추정 오류를 제거하고 잔여 먼지를 방지할 수 있습니다.

**Myriad:** 인지함. 이것은 전적으로 타당하며 이 옵션을 갖고 싶지만 안타깝게도 스마트 계약에서 구현하기에는 너무 복잡합니다.

고정 곱 시장 조성자(fixed product market maker) 공식 `(L^n = O₁ * O₂ * ... * On)`(여기서 `L`은 유동성, `n`은 결과의 수, `Ox`는 결과 `x`의 주식 수)을 보존하기 위해, 이진 결과에서 매도할 금액을 계산하는 공식은 다음과 같습니다:

```
// x = 사용자가 받을 토큰 금액
// shares = 매도된 주식
// S = 매도 토큰 풀 잔액
// O = 다른 결과 풀 잔액

x² - x * (O + S + shares) + shares * O = 0
=> x = 1/2 (-sqrt(O^2 - 2 O (shares + S) + 4 * O * S + (shares + S)^2) + O + shares + S)
```

삼항 결과 시장의 경우 3차 공식이 됩니다:
```
x³ - x²(shares + S + O₁ + O₂) + x((shares + S)(O₁ + O₂) + O₁O₂) - shares × O₁ × O₂ = 0
```

등등. 결과의 수가 많을수록 산술 표현식이 더 복잡해집니다. 이를 계산하는 것은 스마트 계약에서 계산하기에 가스 비용이 매우 많이 들고 부정확합니다.

대안은 이미 시행 중인 방법입니다. Newton-Raphson 방법을 사용하여 사용자의 토큰 가치가 얼마인지 오프체인 추정을 수행하고 해당 메서드에서 반환된 금액으로 `sell`을 호출하는 것입니다.

\clearpage
## 정보성 (Informational)

### 테스트에서 더 이상 사용되지 않는 `testFail` 사용

**설명:** `PredictionMarket.t.sol` 및 `PredictionMarketManager.t.sol`의 여러 테스트([1](https://github.com/Polkamarkets/polkamarkets-js/blob/24f1394be94d27433d2e3a7370442126e1c1e5ba/test/PredictionMarket.t.sol#L691-L698), [2](https://github.com/Polkamarkets/polkamarkets-js/blob/24f1394be94d27433d2e3a7370442126e1c1e5ba/test/PredictionMarketManager.t.sol#L236-L248), [3](https://github.com/Polkamarkets/polkamarkets-js/blob/24f1394be94d27433d2e3a7370442126e1c1e5ba/test/PredictionMarketManager.t.sol#L284-L296), [4](https://github.com/Polkamarkets/polkamarkets-js/blob/24f1394be94d27433d2e3a7370442126e1c1e5ba/test/PredictionMarketManager.t.sol#L324-L338), [5](https://github.com/Polkamarkets/polkamarkets-js/blob/24f1394be94d27433d2e3a7370442126e1c1e5ba/test/PredictionMarketManager.t.sol#L369-L382), [6](https://github.com/Polkamarkets/polkamarkets-js/blob/24f1394be94d27433d2e3a7370442126e1c1e5ba/test/PredictionMarketManager.t.sol#L384-L399), [7](https://github.com/Polkamarkets/polkamarkets-js/blob/24f1394be94d27433d2e3a7370442126e1c1e5ba/test/PredictionMarketManager.t.sol#L425-L458), [8](https://github.com/Polkamarkets/polkamarkets-js/blob/24f1394be94d27433d2e3a7370442126e1c1e5ba/test/PredictionMarketManager.t.sol#L596-L633), [9](https://github.com/Polkamarkets/polkamarkets-js/blob/24f1394be94d27433d2e3a7370442126e1c1e5ba/test/PredictionMarketManager.t.sol#L680-L713), [10](https://github.com/Polkamarkets/polkamarkets-js/blob/24f1394be94d27433d2e3a7370442126e1c1e5ba/test/PredictionMarketManager.t.sol#L879-L892))는 Foundry의 `testFail` 패턴을 사용합니다. 이 패턴은 [더 이상 사용되지 않으며(deprecated)](https://github.com/foundry-rs/foundry/issues/4437) 더 명시적인 되돌림(revert) 기대치를 위해 피해야 합니다.

대신 테스트는 되돌릴 것으로 예상되는 줄 바로 앞에서 `vm.expectRevert()`를 사용해야 합니다. 명확성과 정확성을 높이려면 특정 오류 메시지를 포함하는 것이 이상적입니다: `vm.expectRevert("expected error")`.

또한 되돌림 조건을 명확하게 전달하기 위해 `test_Revert[When|If]...` 형식을 사용하여 [Foundry의 모범 사례](https://getfoundry.sh/guides/best-practices/writing-tests#organizing-and-naming-tests)에 맞게 이러한 테스트의 이름을 변경하는 것을 고려하십시오.

**Myriad:** [PR#89](https://github.com/Polkamarkets/polkamarkets-js/pull/89), 커밋 [`dbcfdfa`](https://github.com/Polkamarkets/polkamarkets-js/pull/89/commits/dbcfdfabe1300fa9629fd227bfac7d02428c231e)에서 수정되었습니다.

**Cyfrin:** 확인됨. 테스트가 이제 예상된 되돌림을 포착합니다.

### `Ownable2StepUpgradeable` 사용 고려

**설명:** 계약은 현재 접근 제어를 위해 `OwnableUpgradeable`을 사용합니다. 안전성을 높이려면 소유권 이전에 대한 명시적인 수락 단계를 강제하고 제어권의 우발적 손실을 방지하는 데 도움이 되는 `Ownable2StepUpgradeable`로 업그레이드하는 것을 고려하십시오.

`OwnableUpgradeable`을 `Ownable2StepUpgradeable`로 교체하여 2단계 전송 패턴을 강제하는 것을 고려하십시오.

또한 `renounceOwnership()` 함수를 재정의하여 항상 되돌리도록 하는 것을 고려하십시오. 이는 소유권의 우발적 또는 의도하지 않은 포기를 방지하여 계약에 승인된 관리자가 영구적으로 없는 상태로 남는 것을 방지합니다.

**Myriad:** [PR#83](https://github.com/Polkamarkets/polkamarkets-js/pull/83), 커밋 [`8efe6e4`](https://github.com/Polkamarkets/polkamarkets-js/pull/83/commits/8efe6e4b6fc031275e5485259e61d4f5b1fe3f2c)에서 수정되었습니다.

**Cyfrin:** 확인됨. `Ownable2StepUpgradeable`이 이제 사용됩니다.

### 라운딩 악용을 방지하기 위해 `minAmount` 강제 고려

**설명:** 잠재적인 정밀도 기반 악용이나 라운딩 오류로부터 보호하기 위해 프로토콜은 거래, 유동성 제공 및 청구와 같은 사용자 대면 작업 전반에 걸쳐 `minAmount` 임계값(예: 토큰 단위로 $1에 해당)을 강제하는 것을 고려해야 합니다.

매우 적은 금액이 허용되면 특히 고정 소수점 수학을 사용하는 시장에서 라운딩 로직, 수수료 계산 또는 불변량 집행에서 엣지 케이스가 트리거될 수 있습니다. 현재 악용 가능하지 않더라도 "먼지" 금액을 허용하지 않는 것이 더 안전한 기준을 제공합니다.

`buy`, `sell`, `addLiquidity` 및 `claim*` 함수와 같은 사용자 대면 작업에 프로토콜 전체 `minAmount` 확인을 적용하는 것을 고려하십시오.

**Myriad:** 인지함. 권장 사항은 타당하지만 다중 토큰 환경에서 구현하기가 매우 복잡합니다. 다음과 같은 몇 가지 고려 사항이 있습니다:

- ERC20 토큰 소수점 - 표준은 18이지만 USDC 또는 USDT와 같은 스테이블코인은 6입니다.
- ERC20 토큰 가격 - DAI와 WETH를 고려하십시오 - 둘 다 18 소수점을 갖지만 1 DAI = 1$이고 1 WETH > $3700입니다.

위의 점을 고려할 때, 다른 시장에 대해 다른 소수점과 가격을 다룰 수 있으므로 프로토콜 전체 `minAmount`는 적절하지 않습니다. 이를 구현하려면 `PredictionMarketManager` 측에서 강제해야 하는데, 제 생각에는 이 시점에서 피하고 싶은 로직의 오버헤드입니다.

### 단일 결과 시장 허용 안 함

**설명:** [`PredictionMarketV3_4::_createMarket`](https://github.com/Polkamarkets/polkamarkets-js/blob/24f1394be94d27433d2e3a7370442126e1c1e5ba/contracts/PredictionMarketV3_4.sol#L299-L356)에서 시장을 생성할 때 다음 [확인](https://github.com/Polkamarkets/polkamarkets-js/blob/24f1394be94d27433d2e3a7370442126e1c1e5ba/contracts/PredictionMarketV3_4.sol#L311)은 결과 수가 범위 내에 있는지 확인합니다:

```solidity
require(desc.outcomes > 0 && desc.outcomes <= MAX_OUTCOMES, "!oc");
```

그러나 `desc.outcomes == 1`을 허용하면 가능한 결과가 하나뿐인 예측 시장이 되어 시장을 갖는 목적을 훼손합니다.

확인을 다음과 같이 변경하는 것을 고려하십시오:

```solidity
require(desc.outcomes > 1 && desc.outcomes <= MAX_OUTCOMES, "!oc");
```

생성된 모든 시장에 적어도 두 가지 결과가 포함되도록 합니다.

**Myriad:** [PR#78](https://github.com/Polkamarkets/polkamarkets-js/pull/78), 커밋 [`9d0090d`](https://github.com/Polkamarkets/polkamarkets-js/pull/78/commits/9d0090d43085cca20b955cbddc33d928619c8f32)에서 수정되었습니다.

**Cyfrin:** 확인됨. 결과 확인은 `>= 2`가 아닙니다.

### `PredictionMarketV3_4::calcSellAmount`에서 언더플로 되돌림을 방지하기 위한 명시적 확인 추가

**설명:** [`PredictionMarketV3_4::calcSellAmount`](https://github.com/Polkamarkets/polkamarkets-js/blob/24f1394be94d27433d2e3a7370442126e1c1e5ba/contracts/PredictionMarketV3_4.sol#L434)의 다음 줄은 사용자가 너무 많은 주식을 매도하려고 하면 언더플로로 인해 되돌려질 수 있습니다:

```solidity
endingOutcomeBalance = (endingOutcomeBalance * outcomeShares).ceilDiv(outcomeShares - amountPlusFees);
```

`amountPlusFees >= outcomeShares`이면 분모가 0 또는 음수가 되어 되돌림이 발생합니다. 이는 수학적으로 예상되는 일이지만, 무엇이 잘못되었는지에 대한 명확한 표시를 받지 못하는 사용자에게는 혼란스러울 수 있습니다.

다음과 같은 명시적 확인을 추가하는 것을 고려하십시오:

```solidity
require(amountPlusFees < outcomeShares, "a>s");
```

이는 더 명확한 오류 메시지를 제공하고 계산 중 언더플로로 인한 되돌림을 방지합니다.

**Myriad:** [PR#80](https://github.com/Polkamarkets/polkamarkets-js/pull/80), 커밋 [`90a2742`](https://github.com/Polkamarkets/polkamarkets-js/pull/80/commits/90a274240da4b4a020098ff4d018b6df908e816b)에서 수정되었습니다.

**Cyfrin:** 확인됨. 금액보다 더 많은 결과 주식이 있는지 확인하는 검사가 이제 제자리에 있습니다.

### 사용되지 않는 필드 `MarketResolution.resolved`

**설명:** `MarketResolution` 구조체에는 [`resolved`](http://github.com/Polkamarkets/polkamarkets-js/blob/24f1394be94d27433d2e3a7370442126e1c1e5ba/contracts/PredictionMarketV3_4.sol#L160) 필드가 포함되어 있지만 현재 계약 로직의 어디에서도 사용되지 않습니다. 사용되지 않는 상태 변수를 유지하면 코드 복잡성이 증가하고 향후 유지 관리자나 감사자에게 혼란을 줄 수 있습니다. 또한 불완전하거나 오래된 로직을 암시할 수도 있습니다.

불필요한 경우 `resolved` 필드를 제거하거나 기능적 목적을 수행하도록 의도된 경우 해결 로직에 의미 있게 통합하는 것을 고려하십시오.

**Myriad:** [PR#81](https://github.com/Polkamarkets/polkamarkets-js/pull/81), 커밋 [`6060137`](https://github.com/Polkamarkets/polkamarkets-js/pull/81/commits/60601378ab213819e79836c4cec3533fe3c649b3)에서 수정되었습니다.

**Cyfrin:** 확인됨. `resolved`가 `MarketResolution` 구조체에서 제거되었습니다.

### "Weight" 및 `poolWeight`는 코드/문서 전체에서 다른 의미를 가짐

**설명:** 개념 가중치/변수 `poolWeight`는 계약 전체의 여러 위치에서 사용되며, 세 가지 다른 목적을 수행하므로 혼란을 초래할 수 있습니다:

1. [`PredictionMarketV3_4::addLiquidity`](https://github.com/Polkamarkets/polkamarkets-js/blob/24f1394be94d27433d2e3a7370442126e1c1e5ba/contracts/PredictionMarketV3_4.sol#L630) 및 [`PredictionMarketV3_4::removeLiquidity`](https://github.com/Polkamarkets/polkamarkets-js/blob/24f1394be94d27433d2e3a7370442126e1c1e5ba/contracts/PredictionMarketV3_4.sol#L751)에서는 풀의 최소 또는 최대 결과 주식 수를 나타냅니다.
2. [`MarketFees.poolWeight`](https://github.com/Polkamarkets/polkamarkets-js/blob/24f1394be94d27433d2e3a7370442126e1c1e5ba/contracts/PredictionMarketV3_4.sol#L151)에서는 수수료 누산기로 사용됩니다.
3. [문서](https://help.polkamarkets.com/how-polkamarkets-works/trading-and-price-calculation#6ec9ef9b0b8d4b61991c86d35936c4c3)에서 "가중치(weight)"는 공유 결과의 곱을 나타내는 데 사용됩니다.

의미상 다른 여러 역할에 대해 단일 변수 이름(`poolWeight`)을 사용하면 개발자, 감사자 및 통합자에게 오해를 불러일으킬 수 있습니다. 또한 코드 컨텍스트에서 "가중치"라는 단어의 의미를 혼합하면 로직 뒤의 의도가 모호해지고 수수료 동작, 주식 분배 또는 가격 계산과 같은 핵심 메커니즘을 잘못 해석할 수 있습니다.

각 사용 사례에 대해 더 구체적이고 설명적인 변수 이름을 사용하여 이러한 우려 사항을 분리하는 것을 고려하십시오. 예를 들어:

* 유동성 운영에 `minOutcomeShares` 또는 `baseShares` 사용
* 수수료 추적에 `feeAccumulator` 사용
* 가격 관련 수학에 `priceWeightProduct` 또는 유사한 것 사용

이는 가독성을 향상시키고 계약의 유지 관리자와 사용자의 인지 부하를 줄일 것입니다.

**Myriad:** [PR#82](https://github.com/Polkamarkets/polkamarkets-js/pull/82), 커밋 [`d30be85`](https://github.com/Polkamarkets/polkamarkets-js/pull/82/commits/d30be8566ea49885582f3caa1a2669c948cc4962)에서 수정되었습니다.

**Cyfrin:** 확인됨. `MarketFees`의 `poolWeight`가 `feeAccumulator`로 이름이 변경되었고 변수 `poolWeight`가 `baseShares`로 이름이 변경되었습니다.

### `PredictionMarketV3_4::mintAndCreateMarket`에서 불필요한 전송

**설명:** [`PredictionMarketV3_4::mintAndCreateMarket`](https://github.com/Polkamarkets/polkamarkets-js/blob/24f1394be94d27433d2e3a7370442126e1c1e5ba/contracts/PredictionMarketV3_4.sol#L381-L390)은 사용자에게 `FantasyERC20`을 발행(mint)한 다음 계약으로 전송합니다:
```solidity
function mintAndCreateMarket(CreateMarketDescription calldata desc) external nonReentrant returns (uint256 marketId) {
  // mint the amount of tokens to the user
  IFantasyERC20(address(desc.token)).mint(msg.sender, desc.value);

  marketId = _createMarket(desc);
  // transferring funds
  desc.token.safeTransferFrom(msg.sender, address(this), desc.value);

  return marketId;
}
```
문제는 먼저 `desc.value`가 사용자에게 발행된 다음 `desc.value`가 계약으로 전송되므로 두 번째 전송이 불필요하다는 것입니다. 이를 다음과 같이 단순화하는 것을 고려하십시오:
```solidity
function mintAndCreateMarket(CreateMarketDescription calldata desc) external nonReentrant returns (uint256 marketId) {
  // mint the amount of tokens to the user
  IFantasyERC20(address(desc.token)).mint(address(this), desc.value);

  return _createMarket(desc);
}
```
이렇게 하면 전송을 절약하고 사용자가 계약이 `FantasyERC20` 토큰을 사용하도록 승인할 필요가 없습니다.

**Myriad:** [PR#86](https://github.com/Polkamarkets/polkamarkets-js/pull/86), 커밋 [`ba955d8`](https://github.com/Polkamarkets/polkamarkets-js/pull/86/commits/ba955d84cfc73bd41743c9e21ffea151f629c012)에서 수정되었습니다.

**Cyfrin:** 확인됨. 발행은 이제 계약으로 직접 수행되며 전송이 제거되었습니다.

### 중요한 상태 변경에 대한 이벤트 누락

**설명:** 다음 호출은 이벤트를 방출하지 않습니다:
* [`LandFactory::updateLockAmount`](https://github.com/Polkamarkets/polkamarkets-js/blob/24f1394be94d27433d2e3a7370442126e1c1e5ba/contracts/LandFactory.sol#L205-L210)
* [`PredictionMarketV3_4::withdraw`](https://github.com/Polkamarkets/polkamarkets-js/blob/24f1394be94d27433d2e3a7370442126e1c1e5ba/contracts/PredictionMarketV3_4.sol#L1458-L1460)

이벤트는 오프체인 추적, 감사 및 투명성을 가능하게 합니다. 위의 호출에서 이벤트를 방출하는 것을 고려하십시오.

**Myriad:** [PR#87](https://github.com/Polkamarkets/polkamarkets-js/pull/87), 커밋 [`b40b740`](https://github.com/Polkamarkets/polkamarkets-js/pull/87/commits/b40b7403b227f6e40de984f4d0ff4c3170d6129b) 및 [`52bdf82`](https://github.com/Polkamarkets/polkamarkets-js/pull/87/commits/52bdf82088a71f721f31c9eeb1da0a79407af5d9)에서 수정되었습니다.

**Cyfrin:** 확인됨. 두 호출 모두 이제 이벤트를 방출합니다.

\clearpage
