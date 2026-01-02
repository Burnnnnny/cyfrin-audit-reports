## 중간 중요도 (Medium Risk)


### `LimitOrderRegistry::newOrder` 호출이 오버플로우로 인해 되돌려질(revert) 수 있음

**설명:** `uint256` 대신 `uint128`로 정의된 변수에 대해 큰 수 곱셈이 수행되므로 합리적인 입력이 새로운 주문을 개설할 때 산술 오버플로우를 일으킬 수 있습니다. 구체적으로 `LimitOrderRegistry::_mintPosition` 및 `LimitOrderRegistry::_addToPosition`에서 다음 라인(두 함수 모두에 나타남)이 문제가 됩니다:

```solidity
uint128 amount0Min = amount0 == 0 ? 0 : (amount0 * 0.9999e18) / 1e18;
uint128 amount1Min = amount1 == 0 ? 0 : (amount1 * 0.9999e18) / 1e18;
```

**파급력 (Impact):** 사용자가 `341e18`을 초과하는 예치 금액으로 새로운 주문을 생성할 수 없으므로, 프로토콜이 비교적 작은 주문으로만 작동하도록 제한됩니다.

**개념 증명 (Proof of Concept):** `test/LimitOrderRegistry.t.sol`에 다음 테스트를 붙여넣으세요:

```solidity
function test_OverflowingNewOrder() public {
    uint96 amount = 340_316_398_560_794_542_918;
    address msgSender = 0xE0b906ae06BfB1b54fad61E222b2E324D51e1da6;
    deal(address(USDC), msgSender, amount);
    vm.startPrank(msgSender);
    USDC.approve(address(registry), amount);

    registry.newOrder(USDC_WETH_05_POOL, 204900, amount, true, 0);
}
```

**권장 완화 조치 (Recommended Mitigation):** 곱셈을 수행하기 전에 `uint128` 값을 `uint256`으로 캐스팅하세요:

```solidity
uint128 amount0Min = amount0 == 0 ? 0 : uint128((uint256(amount0) * 0.9999e18) / 1e18);
uint128 amount1Min = amount1 == 0 ? 0 : uint128((uint256(amount1) * 0.9999e18) / 1e18);
```

**GFX Labs:** [f9934fe](https://github.com/crispymangoes/uniswap-v3-limit-orders/commit/f9934fe5d5eaaf061f4dab110a7b99efda7efb20) 및 [c731cd4](https://github.com/crispymangoes/uniswap-v3-limit-orders/commit/c731cd4d579af3aebd7a75c00bc9aa4ddb45f112) 커밋에서 승수를 18 소수점 자릿수에서 4 소수점 자릿수로 변경하여 수정했습니다.

**Cyfrin:** 확인했습니다.


### 이전 `BatchOrder`가 오더북에서 제거될 때까지 0 또는 1 틱(tick) 공간으로 구분된 반대 방향의 특정 풀에 대한 새 주문이 불가능함

**설명:** 방향이 현재 틱 가격에서 기존 주문과 반대되는 경우 새 주문 생성은 `LimitOrderRegistry__DirectionMisMatch()`와 함께 되돌려집니다. 하한/상한 틱 계산으로 인해 이는 1 틱 공간으로 구분된 주문에도 적용됩니다. 가격 편차로 인해 기존 주문이 ITM(In-The-Money)이 되는 경우, 기존 주문에 대한 유지 관리(upkeep)가 수행되어 오더북에서 완전히 제거될 때까지 반대 방향으로 새 주문을 하는 것은 불가능합니다.

**파급력 (Impact):** 이 엣지 케이스는 다음과 같은 상황에서만 문제가 됩니다:
* 임의의 수의 사용자가 주어진 틱 가격에 주문을 합니다.
* 가격이 편차를 보여 이 `BatchOrder`(및 잠재적으로 다른 주문)가 ITM이 됩니다.
* DoS, 오라클 다운타임 또는 유지 관리당 최대 주문 수 초과(매우 큰 가격 편차의 경우)로 인해 유지 관리가 아직 수행되지 않은 경우 원래 `BatchOrder`는 오더북에 남습니다(이중 연결 리스트로 표시됨).
* 원래 주문이 리스트에 남아 있는 한, 0 또는 1 틱 공간으로 구분된 반대 방향의 특정 풀에 대한 새 주문은 할 수 없습니다.

**개념 증명 (Proof of Concept):**
```solidity
function testOppositeOrders() external {
    uint256 amount = 1_000e6;
    deal(address(USDC), address(this), amount);

    // Current tick 204332
    // 204367
    // Current block 16371089
    USDC.approve(address(registry), amount);
    registry.newOrder(USDC_WETH_05_POOL, 204910, uint96(amount), true, 0);

    // Make a large swap to move the pool tick.
    address[] memory path = new address[](2);
    path[0] = address(WETH);
    path[1] = address(USDC);

    uint24[] memory poolFees = new uint24[](1);
    poolFees[0] = 500;

    (bool upkeepNeeded, bytes memory performData) = registry.checkUpkeep(abi.encode(USDC_WETH_05_POOL));

    uint256 swapAmount = 2_000e18;
    deal(address(WETH), address(this), swapAmount);
    _swap(path, poolFees, swapAmount);

    (upkeepNeeded, performData) = registry.checkUpkeep(abi.encode(USDC_WETH_05_POOL));

    assertTrue(upkeepNeeded, "Upkeep should be needed.");

    // registry.performUpkeep(performData);

    amount = 2_000e17;
    deal(address(WETH), address(this), amount);
    WETH.approve(address(registry), amount);
    int24 target = 204910 - USDC_WETH_05_POOL.tickSpacing();
    vm.expectRevert(
        abi.encodeWithSelector(LimitOrderRegistry.LimitOrderRegistry__DirectionMisMatch.selector)
    );
    registry.newOrder(USDC_WETH_05_POOL, target, uint96(amount), false, 0);
}
```

**권장 완화 조치 (Recommended Mitigation):** 논의된 대로 반대 방향의 주문에 대해 두 번째 오더북을 구현하세요.

**GFX Labs:** 오더북을 두 개의 리스트로 분리하고 반대 방향의 주문이 완전히 다른 LP 포지션을 사용하도록 [7b65915](https://github.com/crispymangoes/uniswap-v3-limit-orders/commit/7b6591508cc0f412edd7f105f012a0cbb7f4b1fe) 및 [9e9ceda](https://github.com/crispymangoes/uniswap-v3-limit-orders/commit/9e9cedad083ff9812a9782a88ac9db5cd713b743) 커밋에서 수정했습니다.

**Cyfrin:** 확인했습니다. '방향' 키를 포함하도록 `getPositionFromTicks` 매핑에 대한 주석을 업데이트해야 합니다.

**GFX Labs:** [f303a50](https://github.com/crispymangoes/uniswap-v3-limit-orders/commit/f303a502492aaf31aeb180640861f326039dd962) 커밋에서 수정되었습니다.

**Cyfrin:** 확인했습니다.


### `LimitOrderRegistry::cancelOrder` 호출이 오버플로우로 인해 되돌려질(revert) 수 있음

**설명:** 주문을 취소할 때 `uint256` 대신 `uint128`로 정의된 변수에 대해 큰 수 곱셈이 수행되므로 합리적인 입력이 산술 오버플로우를 일으킬 수 있습니다. 구체적으로 `LimitOrderRegistry::cancelOrder`의 포지션에서 가져올 유동성 비율을 계산할 때, `depositAmount`에 `1e18`을 곱하면 결과가 `uint128` 범위를 초과하므로 `depositAmount >= 341e18`일 때 오버플로우가 발생합니다.

```solidity
uint128 depositAmount = batchIdToUserDepositAmount[batchId][sender];
if (depositAmount == 0) revert LimitOrderRegistry__UserNotFound(sender, batchId);

// Remove one from the userCount.
order.userCount--;

// Zero out user balance.
delete batchIdToUserDepositAmount[batchId][sender];

uint128 orderAmount;
if (order.direction) {
    orderAmount = order.token0Amount;
    if (orderAmount == depositAmount) {
        liquidityPercentToTake = 1e18;
        // Update order tokenAmount.
        order.token0Amount = 0;
    } else {
        liquidityPercentToTake = (1e18 * depositAmount) / orderAmount; // @audit - overflow
        // Update order tokenAmount.
        order.token0Amount = orderAmount - depositAmount;
    }
} else {
    orderAmount = order.token1Amount;
    if (orderAmount == depositAmount) {
        liquidityPercentToTake = 1e18;
        // Update order tokenAmount.
        order.token1Amount = 0;
    } else {
        liquidityPercentToTake = (1e18 * depositAmount) / orderAmount; // @audit - overflow
        // Update order tokenAmount.
        order.token1Amount = orderAmount - depositAmount;
    }
}
```

**파급력 (Impact):** 풀의 마지막 예금자가 아닌 한, 예치 금액이 `341e18`과 같거나 초과하는 경우 사용자가 주문을 취소하는 것이 불가능할 수 있습니다.

**권장 완화 조치 (Recommended Mitigation):**
```solidity
liquidityPercentToTake = (1e18 * uint256(depositAmount)) / orderAmount;
```

**GFX Labs:** [d67b293](https://github.com/crispymangoes/uniswap-v3-limit-orders/commit/d67b293afbe3590ddb835fc2f9ec617d056cb3c2) 커밋에서 `depositAmount`를 `uint256`으로 캐스팅하여 수정했습니다.

**Cyfrin:** 확인했습니다.


### 악의적인 사용자가 1 틱 공간으로 구분된 반대 방향으로 `LimitOrderRegistry::cancelOrder`를 호출하여 특정 목표 틱에서 ITM 주문을 취소할 수 있음

**설명:** 사용자는 `LimitOrderRegistry::cancelOrder`를 호출하여 지정가 주문을 취소할 수 있습니다. 내부적으로 `_getOrderStatus`를 호출하고 반환 값을 검증함으로써 OTM(Out-Of-The-Money) 주문만 취소할 수 있도록 의도되었습니다.

```solidity
    function cancelOrder(
        UniswapV3Pool pool,
        int24 targetTick,
        bool direction //@audit don't validate order.direction == direction
    ) external returns (uint128 amount0, uint128 amount1, uint128 batchId) {
        uint256 positionId;
        {
            // Make sure order is OTM.
            (, int24 tick, , , , , ) = pool.slot0();

            // Determine upper and lower ticks.
            int24 upper;
            int24 lower;
            {
                int24 tickSpacing = pool.tickSpacing();
                // Make sure targetTick is divisible by spacing.
                if (targetTick % tickSpacing != 0)
                    revert LimitOrderRegistry__InvalidTargetTick(targetTick, tickSpacing);
                if (direction) {
                    upper = targetTick;
                    lower = targetTick - tickSpacing;
                } else {
                    upper = targetTick + tickSpacing;
                    lower = targetTick;
                }
            }
            // Validate lower, upper,and direction. Make sure order is not ITM or MIXED
            {
                OrderStatus status = _getOrderStatus(tick, lower, upper, direction);
                if (status != OrderStatus.OTM) revert LimitOrderRegistry__OrderITM(tick, targetTick, direction);
            }

            // Get the position id.
            positionId = getPositionFromTicks[pool][lower][upper];

            if (positionId == 0) revert LimitOrderRegistry__InvalidPositionId();
        }
        ...
```

주요 결함은 사용자가 사용자 정의 `direction`을 매개변수로 제공할 수 있지만 `direction == order.direction`인지에 대한 검증이 없다는 것입니다.

따라서 악의적인 사용자는 OTM 조건을 만족시키기 위해 실제 목표 틱과 1 틱 공간으로 구분된 목표 틱으로 반대 방향을 전달할 수 있습니다.

`positionId`에서 배치 주문이 검색되면 해당 함수는 거기서부터 `order.direction`을 사용하므로 그 이후에는 올바르게 계속 작동합니다.

**파급력 (Impact):** 악의적인 사용자는 1 틱 공간으로 구분된 틱과 반대 방향을 제공하여 특정 목표 틱에서 주문을 취소할 수 있습니다. 이로 인해 ITM 상태여서 취소할 수 없어야 하는 경우에도 마음대로 취소할 수 있는 무위험 거래를 할 수 있습니다.

**개념 증명 (Proof of Concept):**
(코드 생략) [원문 참고]

참고: 할당된 시간 내에 PoC로 이 발견 사항을 완전히 검증할 수 없었습니다. 현재 `order.direction = true`일 때 `amount0 == 0`이므로 `LimitOrderRegistry__NoLiquidityInOrder()`가 출력됩니다.

이 발견 사항을 증명하기 위해 `LimitOrderRegistry::cancelOrder`에서 아래 검증 섹션을 일시적으로 제거하면 ITM 주문을 취소할 수 있습니다.

(코드 생략) [원문 참고]

**권장 완화 조치 (Recommended Mitigation):** `LimitOrderRegistry::cancelOrder`에 `direction == order.direction` 검증을 추가해야 합니다.

**GFX Labs:** 오더북을 두 개의 리스트로 분리하고 반대 방향의 주문이 완전히 다른 LP 포지션을 사용하도록 [7b65915](https://github.com/crispymangoes/uniswap-v3-limit-orders/commit/7b6591508cc0f412edd7f105f012a0cbb7f4b1fe) 및 [9e9ceda](https://github.com/crispymangoes/uniswap-v3-limit-orders/commit/9e9cedad083ff9812a9782a88ac9db5cd713b743) 커밋에서 수정했습니다.

**Cyfrin:** 확인했습니다.


### 악의적인 검증자가 주문의 생성 또는 취소를 막을 수 있음

**설명:** [`MintParams`](https://github.com/crispymangoes/uniswap-v3-limit-orders/blob/83f5db9cb90926c11ee6ce872dc165b7f600f3d8/src/LimitOrderRegistry.sol#L1056), [`IncreaseLiquidityParams`](https://github.com/crispymangoes/uniswap-v3-limit-orders/blob/83f5db9cb90926c11ee6ce872dc165b7f600f3d8/src/LimitOrderRegistry.sol#L1099) 및 [`DecreaseLiquidityParams`](https://github.com/crispymangoes/uniswap-v3-limit-orders/blob/83f5db9cb90926c11ee6ce872dc165b7f600f3d8/src/LimitOrderRegistry.sol#L1207)에 대한 마감 기한(deadline)으로 `block.timestamp`를 사용하는 것은 Uniswap v3와 인터페이싱하는 주어진 트랜잭션이 검증자가 포함하기로 결정할 때마다 유효함을 의미합니다. 악의적인 검증자가 해당 풀의 틱 가격이 이동하여 마감 기한은 유효하지만 주문 상태가 더 이상 유효하지 않게 된 후까지 이러한 트랜잭션을 보류하면 주문이 생성되거나 취소되지 않을 수 있습니다.

**파급력 (Impact):** 검증자가 트랜잭션을 블록에 포함하기로 결정할 때마다 `block.timestamp`가 현재 타임스탬프가 되므로 그 시점에 유효하게 됩니다. 이로 인해 ITM 주문은 취소할 수 없으므로 가격이 목표 틱을 초과할 때까지 보류하여 취소하려는 의도였던 주문이 강제로 체결되거나, 즉시 ITM인 주문 생성이 허용되지 않는 것과 유사한 이유로 주문이 전혀 생성되지 않을 수 있습니다.

**권장 완화 조치 (Recommended Mitigation):** `NonFungiblePositionManager`를 통해 Uniswap v3와 상호 작용하는 모든 함수에 deadline 인수를 추가하고 관련 호출에 전달하세요.

**GFX Labs:** [f05cdfc](https://github.com/crispymangoes/uniswap-v3-limit-orders/commit/f05cdfce4762d980235fe4d726800d2b0a112d2d) 커밋에서 모든 포지션 관리자 호출에 deadline 인수를 추가하여 수정했습니다.

**Cyfrin:** 확인했습니다.


### `performUpkeep`에 대한 가스 그리핑(Gas griefing) 서비스 거부

**설명:** 엉킨(tangled) 리스트로 인해 발생하는 문제를 완화하기 위해 `LimitOrderRegistry::performUpkeep`은 워크(walk) 방향과 일치하는 다음 주문을 찾을 때까지 리스트를 헤드/테일 쪽으로 계속 워크하도록 수정되었습니다. 이는 [while 루프](https://github.com/crispymangoes/uniswap-v3-limit-orders/blob/83f5db9cb90926c11ee6ce872dc165b7f600f3d8/src/LimitOrderRegistry.sol#L847)를 사용하여 달성됩니다. 그러나 현재 주문과 다음 주문 사이에 반대 방향의 주문이 다수 있는 경우 가스 그리핑 공격에 취약합니다. 주문 스팸은 `minimumAssets` 요구 사항에 의해 어느 정도 완화되지만, 합리적으로 자금이 충분하거나 무정부적인 공격자는 일련의 작은 주문을 배치하여 이 루프가 유지 관리당 지정된 최대 가스보다 더 많은 가스를 소비하게 하여 합법적인 ITM 주문이 체결되는 것을 막을 수 있습니다.

**파급력 (Impact):** 악의적인 사용자가 유지 관리에 대한 서비스 거부 공격을 수행하여 ITM 주문이 체결되는 것을 막을 수 있습니다.

**권장 완화 조치 (Recommended Mitigation):** 논의된 대로 반대 방향의 주문에 대해 두 번째 오더북을 구현하여 워크 방향과 일치하는 다음 주문을 찾을 때까지 리스트를 워크할 필요를 없애세요.

**GFX Labs:** 오더북을 두 개의 리스트로 분리하고 반대 방향의 주문이 완전히 다른 LP 포지션을 사용하도록 [7b65915](https://github.com/crispymangoes/uniswap-v3-limit-orders/commit/7b6591508cc0f412edd7f105f012a0cbb7f4b1fe) 및 [9e9ceda](https://github.com/crispymangoes/uniswap-v3-limit-orders/commit/9e9cedad083ff9812a9782a88ac9db5cd713b743) 커밋에서 수정했습니다.

**Cyfrin:** 확인했습니다.

\clearpage
## 낮은 중요도 (Low Risk)


### Chainlink 패스트 가스 피드에 대한 추가 검증 수행

**설명:** Chainlink에서 제공하는 데이터 피드를 사용할 때는 여러 임계값과 반환 값을 검증하는 것이 중요합니다. 이것이 없으면 소비 계약이 오래되거나 부정확/무효한 데이터를 사용할 수 있습니다. `LimitOrderRegistry`는 현재 소유자가 지정한 하트비트 값에 대해 마지막 업데이트 이후 시간을 올바르게 검증합니다. 그러나 반환된 가스 가격이 실제로 유효한지(예: 상한/하한 범위 내에 있는지, 전체 보고 라운드의 결과인지) 확인하기 위해 추가 검증을 수행할 수 있습니다.

**파급력 (Impact):** `LimitOrderRegistry::performUpkeep`은 `LimitOrderRegistry::getGasPrice` 내의 이 기능에 의존하므로 올바르게 작동하지 않으면 주문 체결 시 DoS가 발생할 수 있습니다. 그러나 소유자가 피드 주소를 `address(0)`으로 설정하여 소유자가 지정한 대체 값을 사용하게 하는 이스케이프 해치가 있습니다.

**권장 완화 조치 (Recommended Mitigation):** `try/catch` 블록 내에서 `IChainlinkAggregator::latestRoundData`를 호출하고 다음 추가 검증을 포함하는 것을 고려하십시오:

```solidity
require(signedGasPrice > 0, "Negative gas price");
require(signedGasPrice < maxGasPrice, "Upper gas price bound breached");
require(signedGasPrice > minGasPrice, "Lower gas price bound breached");
require(answeredInRound >= roundID, "Round incomplete");
```

**GFX Labs:** 확인했습니다. Chainlink는 데이터 피드 정확성과 신뢰성에 대해 뛰어난 기록을 보유하고 있으며, Chainlink 노드가 부정확한 값을 악의적으로 보고하기 시작하면 다른 곳에 훨씬 더 수익성 있는 기회가 있습니다.

또한 패스트 가스 피드가 불안정해지는 경우 소유자가 수동으로 가스 가격을 설정하거나 GFX Labs 봇에 의해 업데이트되는 사용자 정의 Fast Gas Feed 계약을 만드는 것이 간단합니다.

**Cyfrin:** 확인했습니다.


### 래핑된 네이티브(wrapped native) 잔액이 0이면 네이티브 자산 인출이 되돌려질(revert) 수 있음

**설명:** `LimitOrderRegistry::withdrawNative` 함수를 사용하면 소유자가 계약에서 네이티브 및 래핑된 네이티브 자산을 인출할 수 있습니다. 둘 다 잔액이 0이면 인출할 것이 없으므로 이 함수에 대한 호출이 되돌려집니다.

```solidity
/**
 * @notice Allows owner to withdraw wrapped native and native assets from this contract.
 */
function withdrawNative() external onlyOwner {
    uint256 wrappedNativeBalance = WRAPPED_NATIVE.balanceOf(address(this));
    uint256 nativeBalance = address(this).balance;
    // Make sure there is something to withdraw.
    if (wrappedNativeBalance == 0 && nativeBalance == 0) revert LimitOrderRegistry__ZeroNativeBalance();
    WRAPPED_NATIVE.safeTransfer(owner, WRAPPED_NATIVE.balanceOf(address(this)));
    payable(owner).transfer(address(this).balance);
}
```

래핑된 네이티브 토큰 잔액이 0이 아닌 전송을 포함한 0 값 호출은 잘 성공하지만 반대의 경우에는 호출이 되돌려질 수 있습니다. `safeTransfer`가 래핑된 네이티브 토큰의 `transfer` 함수를 호출하므로, 래핑된 네이티브 토큰이 0 값 전송 시 되돌려지는 경우 래핑된 네이티브 토큰 잔액이 0인 0이 아닌 값 호출에 대한 전체 트랜잭션이 되돌려질 수 있습니다.

**파급력 (Impact):** 이는 래핑된 네이티브 토큰 잔액도 0이 아닐 때까지 네이티브 토큰 잔액의 인출을 일시적으로 방지할 수 있지만, 현재 의도된 대상 체인의 래핑된 네이티브 토큰 중 어느 것도 0 값 전송 시 되돌려지지 않는 것으로 보입니다.

**권장 완화 조치 (Recommended Mitigation):** 전송을 시도하기 전에 래핑된 네이티브 토큰 잔액을 별도로 검증하세요.

**GFX Labs:** [d2dd99c](https://github.com/crispymangoes/uniswap-v3-limit-orders/commit/d2dd99ccc30fd8534373be9ee0566603623e433d) 커밋에서 인출을 시도하기 전에 금액이 0이 아닌지 확인하는 `if` 문을 추가하여 수정했습니다.

**Cyfrin:** 확인했습니다. 되돌릴 필요가 없습니다. 이 줄을 제거할 수 있습니다:
```solidity
if (wrappedNativeBalance == 0 && nativeBalance == 0) revert LimitOrderRegistry__ZeroNativeBalance();
```

**GFX Labs:** 확인했습니다. Revert가 반드시 필요한 것은 아니지만 해가 되지는 않습니다.

**Cyfrin:** 확인했습니다.


### address payable에서 `transfer()` 대신 `call()`을 사용해야 함

**설명:** `transfer()` 및 `send()` 함수는 고정된 2300 가스를 전달합니다. 역사적으로 재진입 공격을 방지하기 위해 가치 전송에 이러한 함수를 사용하는 것이 종종 권장되었습니다. 그러나 EVM 명령어의 가스 비용은 하드 포크 중에 크게 변경될 수 있으며, 이는 가스 비용에 대해 고정된 가정을 하는 이미 배포된 계약 시스템을 망가뜨릴 수 있습니다. 예를 들어. EIP 1884는 `SLOAD` 명령어의 비용 증가로 인해 기존 스마트 계약 여러 개를 망가뜨렸습니다. 따라서 이제 address payable에서 이러한 함수를 사용하는 것은 [더 이상 권장되지 않습니다](https://swcregistry.io/docs/SWC-134).

현재 `transfer()` 사용을 해결해야 하는 [여러](https://github.com/crispymangoes/uniswap-v3-limit-orders/blob/83f5db9cb90926c11ee6ce872dc165b7f600f3d8/src/LimitOrderRegistry.sol#L442) [인스턴스](https://github.com/crispymangoes/uniswap-v3-limit-orders/blob/83f5db9cb90926c11ee6ce872dc165b7f600f3d8/src/LimitOrderRegistry.sol#L642)가 [있습니다](https://github.com/crispymangoes/uniswap-v3-limit-orders/blob/83f5db9cb90926c11ee6ce872dc165b7f600f3d8/src/LimitOrderRegistry.sol#L646).

**파급력 (Impact):** 더 이상 사용되지 않는 `transfer()` 함수를 사용하면 트랜잭션이 실패합니다:

* `LimitOrderRegistry::withdrawNative`를 호출하는 `owner`/`LimitOrderRegistry::claimOrder`를 호출하는 `sender`가 payable 함수를 구현하지 않는 스마트 계약인 경우.
* `LimitOrderRegistry::withdrawNative`를 호출하는 `owner`/`LimitOrderRegistry::claimOrder`를 호출하는 `sender`가 2300 가스 유닛 이상을 사용하는 payable fallback을 구현하는 스마트 계약인 경우.
* `LimitOrderRegistry::claimOrder`를 호출하는 `sender`가 2300 미만의 가스 유닛을 필요로 하지만 프록시를 통해 호출되어 호출의 가스 사용량이 2300을 초과하는 payable fallback 함수를 구현하는 스마트 계약인 경우.
* 또한 일부 다중 서명 지갑의 경우 2300보다 높은 가스를 사용하는 것이 필수적일 수 있습니다.

**권장 완화 조치 (Recommended Mitigation):** `transfer()` 대신 `call()` 또는 solmate의 `safeTransferETH`를 사용하되, [Checks Effects Interactions (CEI) 패턴](https://fravoll.github.io/solidity-patterns/checks_effects_interactions.html)을 준수하도록 하십시오.

**GFX Labs:** [6c486c9](https://github.com/crispymangoes/uniswap-v3-limit-orders/commit/6c486c92598fda3582087d2ecd36f12238115eb1) 커밋에서 solmate `safeTransferETH`를 사용하여 수정했습니다.

**Cyfrin:** 확인했습니다.


### Fee-on-transfer/디플레이션 토큰은 지원되지 않음

**설명:** 새 주문을 생성할 때 `LimitOrderRegistry::newOrder`는 예치된 토큰의 양이 함수 매개변수에 예치 전 잔액을 더한 것과 같다고 가정합니다. 모든 전송에 수수료를 부과하는 토큰의 경우 이 가정이 성립하지 않으므로 실제로 전송된 양은 지정된 양보다 적습니다. 결과적으로 이 값을 사용하여 계산된 `_mintPosition` 및 `_addToPosition`의 타이트한 슬리피지 매개변수로 인해 트랜잭션이 되돌려질 가능성이 높습니다:
```solidity
uint128 amount0Min = amount0 == 0 ? 0 : (amount0 * 0.9999e18) / 1e18;
```
또한 `batchIdToUserDepositAmount`에 저장된 금액은 특정 사용자가 예치한 실제 잔액보다 커서 잘못된 회계가 발생합니다.

**파급력 (Impact):** 프로토콜 내에서 fee-on-transfer/디플레이션 토큰을 사용할 수 없습니다.

**권장 완화 조치 (Recommended Mitigation):** 문제가 되는 토큰을 허용 목록에 추가하지 마십시오. 이러한 토큰을 지원하려는 경우 전송 전후의 토큰 잔액을 캐시하고, 받은 금액으로 입력 금액이 아닌 두 잔액 간의 차이를 사용하는 것이 좋습니다.

**GFX Labs:** 확인했습니다. 이 계약은 fee-on-transfer 또는 디플레이션 토큰을 사용하지 않습니다.

**Cyfrin:** 확인했습니다.


### 체결 가능한 ITM 주문이 항상 체결되지는 않을 수 있음

**설명:** 현재 주문은 `LimitOrderRegistry::performUpkeep`을 통해서만 체결할 수 있으며, 호출당 `maxFillsPerUpkeep` 주문으로 제한되어 이중 연결 리스트를 통해 처리됩니다. 극단적이거나 적대적인 시장 상황을 고려할 때 모든 합법적인 주문이 결국 처리될 것이라는 보장은 없습니다. 큰 가격 변동의 경우와 같이 풀 틱이 크게 빠르게 변하고 장부에 `maxFillsPerUpkeep`보다 큰 다수의 배치 주문이 존재하는 경우, 주어진 블록에서 주문의 하위 집합만 체결될 수 있습니다. 가격 편차가 적은 수의 블록 동안만 지속된다고 가정하면 리스트의 체결 가능한 ITM 주문이 다시 OTM이 되기 전에 청산되지 않을 가능성이 있습니다.

**파급력 (Impact):** 유효하고 체결 가능한 ITM 주문은 포지션 내 유동성이 기술적으로 풀 틱이 목표 틱 가격을 초과하는 기간 동안 사용되고 있음에도 불구하고 체결되지 않을 수 있습니다.

**권장 완화 조치 (Recommended Mitigation):** 주문이 체결될 수 있도록 보장하기 위해 리스트의 포지션과 관계없이 특정 주문을 독립적으로 체결하기 위해 모든 사용자가 호출할 수 있는 공개 `fulfillOrder` 함수를 추가하는 것을 고려하십시오. 주문은 유지 관리 절차에 영향을 주지 않고 단순히 체결되어 리스트에서 제거될 수 있습니다.

**GFX Labs:** 확인했습니다. 전통적인 시장 환경에서도 가격이 일부 지정가 주문 트리거를 지나갈 수 있지만 볼륨이 충분하지 않으면 해당 주문은 여전히 체결되지 않을 수 있습니다.

사용자가 특정 주문을 체결할 수 있도록 하는 사용자 정의 함수를 추가하는 것은 가능한 해결책이지만, 사용자가 Chainlink Automation보다 TX 관리에 더 능숙한 봇을 만드는 경우에만 해결책이 될 수 있으며, 이는 사용자에게 큰 요구 사항입니다. 이 때문에 계약 크기를 줄이기 위해 함수가 추가되지 않았습니다.

**Cyfrin:** 확인했습니다.

\clearpage
## 정보성 (Informational)


### 철자 오류 및 잘못된 NatSpec

다음과 같은 철자 오류 및 잘못된 NatSpec이 확인되었습니다:
* `cancelOrder`는 포지션의 모든 스왑 수수료를 주문을 취소하는 마지막 사람에게 보내는 것으로 [문서화](https://github.com/crispymangoes/uniswap-v3-limit-orders/blob/83f5db9cb90926c11ee6ce872dc165b7f600f3d8/src/LimitOrderRegistry.sol#L657)되어 있습니다. 그러나 실제 동작은 수수료가 [`LimitOrderRegistry::_takeFromPosition`](https://github.com/crispymangoes/uniswap-v3-limit-orders/blob/83f5db9cb90926c11ee6ce872dc165b7f600f3d8/src/LimitOrderRegistry.sol#L1250-L1255)에 대한 내부 호출 내의 [`tokenToSwapFees`](https://github.com/crispymangoes/uniswap-v3-limit-orders/blob/83f5db9cb90926c11ee6ce872dc165b7f600f3d8/src/LimitOrderRegistry.sol#L118-L121) 매핑에 저장되고 소유자가 [`LimitOrderRegistry::withdrawSwapFees`](https://github.com/crispymangoes/uniswap-v3-limit-orders/blob/83f5db9cb90926c11ee6ce872dc165b7f600f3d8/src/LimitOrderRegistry.sol#L423)에서 인출할 수 있는 것이므로 이 주석은 제거되어야 합니다.
* '[Cellar](https://github.com/crispymangoes/uniswap-v3-limit-orders/blob/83f5db9cb90926c11ee6ce872dc165b7f600f3d8/src/LimitOrderRegistry.sol#L455)'에 대한 [참조](https://github.com/crispymangoes/uniswap-v3-limit-orders/blob/83f5db9cb90926c11ee6ce872dc165b7f600f3d8/src/LimitOrderRegistry.sol#L446)는 업데이트하거나 제거해야 합니다.
* [이 주석](https://github.com/crispymangoes/uniswap-v3-limit-orders/blob/83f5db9cb90926c11ee6ce872dc165b7f600f3d8/src/LimitOrderRegistry.sol#L482)은 새 주문이 ITM이거나 MIXED 상태(즉, OTM이 아님)인 경우 되돌려진다는 것을 명확하게 명시해야 합니다.
* '[doubley linked list](https://github.com/crispymangoes/uniswap-v3-limit-orders/blob/83f5db9cb90926c11ee6ce872dc165b7f600f3d8/src/LimitOrderRegistry.sol#L176)'는 'doubly linked list'로 철자를 써야 합니다.
* '[effectedOrder](https://github.com/crispymangoes/uniswap-v3-limit-orders/blob/83f5db9cb90926c11ee6ce872dc165b7f600f3d8/src/LimitOrderRegistry.sol#L240-L242)'는 'affectedOrder'로 철자를 써야 합니다.
* [`LimitOrderRegistry::getGasPrice`](https://github.com/crispymangoes/uniswap-v3-limit-orders/blob/83f5db9cb90926c11ee6ce872dc165b7f600f3d8/src/LimitOrderRegistry.sol#L1284-L1305)는 뷰 함수이므로 'view logic' [섹션 헤더](https://github.com/crispymangoes/uniswap-v3-limit-orders/blob/83f5db9cb90926c11ee6ce872dc165b7f600f3d8/src/LimitOrderRegistry.sol#L1307-L1309) 내 아래로 이동해야 합니다.

**GFX Labs:** [4cff05a](https://github.com/crispymangoes/uniswap-v3-limit-orders/commit/4cff05a1e6acd5f03bac527f0121eb3d3385fc2b) 커밋에서 수정되었습니다.

**Cyfrin:** 확인했습니다.


### 계약이 여러 체인에 배포될 예정일 때 하드코딩된 주소 피하기

[`LimitOrderRegistry::fastGasFeed`](https://github.com/crispymangoes/uniswap-v3-limit-orders/blob/83f5db9cb90926c11ee6ce872dc165b7f600f3d8/src/LimitOrderRegistry.sol#L190-L193) 상태 변수는 생성 중에 [덮어쓰기](https://github.com/crispymangoes/uniswap-v3-limit-orders/blob/83f5db9cb90926c11ee6ce872dc165b7f600f3d8/src/LimitOrderRegistry.sol#L308)되지만, 이 계약은 여러 체인에 배포될 예정이고 주소가 항상 동일할 것이라는 보장이 없으므로 이 변수를 하드코딩된 주소로 초기화하지 않는 것이 좋습니다. 대신 [다른 유사한 변수](https://github.com/crispymangoes/uniswap-v3-limit-orders/blob/83f5db9cb90926c11ee6ce872dc165b7f600f3d8/src/LimitOrderRegistry.sol#L180-L183)의 경우처럼 변수를 선언하고 메인넷 주소를 나타내는 주석을 추가하십시오.

**GFX Labs:** [4cff05a](https://github.com/crispymangoes/uniswap-v3-limit-orders/commit/4cff05a1e6acd5f03bac527f0121eb3d3385fc2b) 커밋에서 수정되었습니다.

**Cyfrin:** 확인했습니다.


### `onlyOwner` 함수에 대한 입력 검증

유효한 주소를 선택하는 것이 소유자에게 가장 이익이 되므로 `LimitOrderRegistry::setRegistrar` 및 `LimitOrderRegistry::setFastGasFeed`에 대한 입력에 대해 입력 검증이 수행되지 않는다는 주석이 있지만, 여전히 몇 가지 가드레일을 구현하는 것이 좋습니다.

예를 들어 등록 기관(registrar)을 검증하기 위해 계약은 `IKeeperRegistrar::typeAndVersion`을 호출하고 반환 값이 저장소에 설정하기 전에 `LimitOrderRegistry::setupLimitOrder`에서 예상되는 값과 일치하는지 확인할 수 있습니다. `IKeeperRegistrar::register` 선택자는 `try/catch` 블록 내에서 이 함수에 호출을 수행하고 `adminAddress`에 대해 `address(0)`을 전달하고 예상되는 동작을 나타내는 `InvalidAdminAddress()` 사용자 정의 오류를 포착하여 느슨하게 검증할 수도 있습니다.

패스트 가스 피드를 검증하기 위해 계약은 먼저 데이터 피드에 호출을 수행하고 값을 저장소에 설정하기 전에 예상 범위 내에서 올바르게 보고되고 있는지 확인할 수 있습니다.

**GFX Labs:** 확인했습니다. 이는 타당한 우려 사항이지만, 계약 크기를 줄이기 위한 노력의 일환으로 이러한 검사는 추가되지 않을 것입니다. 또한 소유자가 실수로 비논리적인 값을 입력한 경우 올바른 값으로 함수를 다시 호출하면 됩니다.

**Cyfrin:** 확인했습니다.


### 토큰 중 하나에 대해 `minimumAssets`가 설정되지 않은 경우 Uniswap v3 풀의 한쪽 측면에 대해 지정가 주문이 동결될 수 있음

낮은 유동성 주문 스팸을 방지하려면 지정가 주문을 배치하기 전에 주어진 자산에 대해 `LimitOrderRegistry::minimumAssets`를 설정해야 합니다. Uniswap v3 풀에는 두 개의 자산이 포함되지만 `LimitOrderRegistry::setMinimumAssets`는 전체 계약에 걸쳐 특정 자산에 대해 호출되며 풀별로 지정되지 않습니다. 주어진 풀의 두 자산 모두에 대해 이 함수가 호출되지 않으면 최소값이 설정되지 않은 자산에 따라 주어진 방향의 지정가 주문이 허용되지 않습니다.

**GFX Labs:** 확인했습니다. 풀의 두 자산이 모두 지원되도록 하여 지정가 주문이 더 많아지고 스왑 수수료가 더 많이 발생하도록 하는 것이 소유자에게 가장 이익이 됩니다.

**Cyfrin:** 확인했습니다.


### 삼항 연산자(ternary operator) 사용 개선

직접 평가되는 부울 표현식으로 단순화할 수 있는 삼항 연산자 [인스턴스](https://github.com/crispymangoes/uniswap-v3-limit-orders/blob/83f5db9cb90926c11ee6ce872dc165b7f600f3d8/src/LimitOrderRegistry.sol#L903)가 하나 있습니다:
```solidity
bool direction = targetTick > node.tickUpper ? true : false;
```
다음과 같이 변경해야 합니다:
```solidity
bool direction = targetTick > node.tickUpper;
```

또한 삼항 연산자 사용이 권장되는 [또 다른 조건문](https://github.com/crispymangoes/uniswap-v3-limit-orders/blob/83f5db9cb90926c11ee6ce872dc165b7f600f3d8/src/LimitOrderRegistry.sol#L524-L525)이 있습니다:
```solidity
if (direction)
    assetIn = poolToData[pool].token0;
else assetIn = poolToData[pool].token1;
```
이 조건문은 다음과 같이 대체할 수 있습니다:
```solidity
assetIn = direction ? poolToData[pool].token0 : poolToData[pool].token1;
```

**GFX Labs:** [4cff05a](https://github.com/crispymangoes/uniswap-v3-limit-orders/commit/4cff05a1e6acd5f03bac527f0121eb3d3385fc2b) 커밋에서 수정되었습니다.

**Cyfrin:** 확인했습니다.


### 공유 로직을 수정자(modifier) 내에서 호출되는 내부 함수로 이동

```solidity
...
if (direction) data.token0.safeApprove(address(POSITION_MANAGER), amount0);
else data.token1.safeApprove(address(POSITION_MANAGER), amount1);

    // 0.9999e18 accounts for rounding errors in the Uniswap V3 protocol.
    uint128 amount0Min = amount0 == 0 ? 0 : (amount0 * 0.9999e18) / 1e18;
    uint128 amount1Min = amount1 == 0 ? 0 : (amount1 * 0.9999e18) / 1e18;
...
// If position manager still has allowance, zero it out.
    if (direction && data.token0.allowance(address(this), address(POSITION_MANAGER)) > 0)
        data.token0.safeApprove(address(POSITION_MANAGER), 0);
    if (!direction && data.token1.allowance(address(this), address(POSITION_MANAGER)) > 0)
        data.token1.safeApprove(address(POSITION_MANAGER), 0);
```

[`_mintPosition`](https://github.com/crispymangoes/uniswap-v3-limit-orders/tree/83f5db9cb90926c11ee6ce872dc165b7f600f3d8/src/LimitOrderRegistry.sol#L1029) 및 [`_addToPosition`](https://github.com/crispymangoes/uniswap-v3-limit-orders/tree/83f5db9cb90926c11ee6ce872dc165b7f600f3d8/src/LimitOrderRegistry.sol#L1078)의 위 로직은 두 함수 모두에서 공유되고 반복됩니다. 이 코드를 다음과 같이 수정자 내에서 호출되는 내부 함수로 이동하는 것을 고려하십시오:
```solidity
modifier approvePositionManager() {
    _approveBefore();
    _;
    _approveAfter();
}
```

**GFX Labs:** 확인했습니다. 이는 타당한 우려 사항이지만, 계약 크기를 줄이고 코드 변경을 최소화하기 위해 구현되지 않을 것입니다.

**Cyfrin:** 확인했습니다.


### `LimitOrderRegistry::newOrder`의 재진입은 전송 후크가 있는 토큰이 실행을 탈취하여 가격을 즉시 ITM이 되도록 조작하고 검증을 우회할 수 있음을 의미함

공격자에게 직접적인 이점은 없어 보이지만, 전송 후크(transfer hooks)가 있는 토큰이 `LimitOrderRegistry::newOrder`에 대한 진행 중인 호출에서 실행을 탈취하는 것이 가능합니다. 이 함수는 호출자로부터 `assetIn` [전송](https://github.com/crispymangoes/uniswap-v3-limit-orders/blob/83f5db9cb90926c11ee6ce872dc165b7f600f3d8/src/LimitOrderRegistry.sol#L527)을 수행하기 전에 새 주문 상태가 OTM인지 [검증](https://github.com/crispymangoes/uniswap-v3-limit-orders/blob/83f5db9cb90926c11ee6ce872dc165b7f600f3d8/src/LimitOrderRegistry.sol#L518)합니다. 따라서 ERC-777과 같이 전송 후크가 있는 입력 토큰을 사용하여 새 주문이 이미 검증을 통과한 후 풀 틱을 왜곡하여 실제로 MIXED 또는 ITM이 되도록 할 수 있습니다. 이를 완화하려면 자산 전송 블록을 주문이 OTM으로 검증되기 전으로 이동하고, 허용 목록 토큰을 신중하게 선택하거나 `newOrder`를 재진입 불가능으로 만드는 것을 고려하십시오.

**GFX Labs:** [4cff05a](https://github.com/crispymangoes/uniswap-v3-limit-orders/commit/4cff05a1e6acd5f03bac527f0121eb3d3385fc2b) 커밋에서 수정되었습니다.

**Cyfrin:** 확인했습니다.

\clearpage
## 가스 최적화 (Gas Optimization)


### 반복적인 외부 호출 대신 스택 변수 사용

`LimitOrderRegistry::withdrawNative` 내에서 네이티브/래핑된 네이티브 자산의 계약 잔액을 결정하는 호출은 [스택 변수](https://github.com/crispymangoes/uniswap-v3-limit-orders/blob/83f5db9cb90926c11ee6ce872dc165b7f600f3d8/src/LimitOrderRegistry.sol#L437-L438)에 캐시되지만 사용되지 않습니다. 후속 전송을 수행할 때 보낼 금액을 결정하기 위해 각 함수가 [다시 호출](https://github.com/crispymangoes/uniswap-v3-limit-orders/blob/83f5db9cb90926c11ee6ce872dc165b7f600f3d8/src/LimitOrderRegistry.sol#L441-L442)되어 반복적인 외부 호출로 가스를 낭비합니다. 대신 스택 변수를 간단히 사용할 수 있습니다.

**GFX Labs:** [4cff05a](https://github.com/crispymangoes/uniswap-v3-limit-orders/commit/4cff05a1e6acd5f03bac527f0121eb3d3385fc2b) 커밋에서 수정되었습니다.

**Cyfrin:** 확인했습니다.


### 스택 변수에 할당하는 것보다 결과를 직접 반환

`LimitOrderRegistry::newOrder`에서 반환 값은 [return 문](https://github.com/crispymangoes/uniswap-v3-limit-orders/blob/83f5db9cb90926c11ee6ce872dc165b7f600f3d8/src/LimitOrderRegistry.sol#L599) 외에는 사용되지 않는 [스택 변수](https://github.com/crispymangoes/uniswap-v3-limit-orders/blob/83f5db9cb90926c11ee6ce872dc165b7f600f3d8/src/LimitOrderRegistry.sol#L597) `batchId`에 할당됩니다. 반환은 한 줄로 수행할 수 있으므로 이 할당은 필요하지 않습니다:
```solidity
return orderBook[details.positionId].batchId;
```

**GFX Labs:** [4cff05a](https://github.com/crispymangoes/uniswap-v3-limit-orders/commit/4cff05a1e6acd5f03bac527f0121eb3d3385fc2b) 커밋에서 수정되었습니다.

**Cyfrin:** 확인했습니다.


### 한 줄로 사후 증가(post-increment) 수행

`LimitOrderRegistry::_setupOrder`의 `batchCount` 변수 사후 증가는 `order.batchId`에 할당될 때 한 줄로 수행할 수 있습니다:
```diff
  function _setupOrder(bool direction, uint256 position) internal {
     BatchOrder storage order = orderBook[position];
-    order.batchId = batchCount;
+    order.batchId = batchCount++;
     order.direction = direction;
-    batchCount++;
  }
```

**GFX Labs:** [4cff05a](https://github.com/crispymangoes/uniswap-v3-limit-orders/commit/4cff05a1e6acd5f03bac527f0121eb3d3385fc2b) 커밋에서 수정되었습니다.

**Cyfrin:** 확인했습니다.
