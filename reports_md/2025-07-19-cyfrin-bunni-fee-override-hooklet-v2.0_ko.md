**수석 감사**

[Giovanni Di Siena](https://x.com/giovannidisiena)

[Pontifex](https://x.com/pontifex73)

**보조 감사**

---

# 발견 사항

## 중간 위험 (Medium Risk)

### 후크 및 큐레이터 수수료의 잘못된 적용으로 인해 `BunniQuoter::quoteSwap`과 `BunniHookLogic::beforeSwap` 간의 유효 스왑 수수료 계산 불일치

**설명:** `BunniQuoter::quoteSwap`과 `BunniHookLogic::beforeSwap`은 동일한 수수료 로직을 포함하도록 의도되었지만, 누락되었거나 회귀(regression)로 인해 다시 도입된 것으로 보이는 몇 가지 불일치가 남아 있습니다.

첫째, `BunniQuoter::quoteSwap`은 `useAmAmmFee`가 활성화될 때 스왑 수수료에 추가 조정을 적용하여 후크 수수료를 추가함으로써 스와퍼에게 부과되는 유효 수수료를 증가시킵니다:

```solidity
swapFee += uint24(hookFeesBaseSwapFee.mulDivUp(hookFeeModifier, MODIFIER_BASE));
```

이 조정은 후크 수수료를 포함한 스왑의 실제 비용을 설명하기 위한 것입니다. 그러나 스왑의 실제 실행을 수행하는 `BunniHookLogic::beforeSwap`에서는 이 조정이 실제로 적용되지 않습니다. 기본 수수료는 후속 이벤트 방출에 사용하기 위해 계산되며 후크 수수료를 포함하지 않고 `HookletLib::hookletAfterSwap` 호출에 전달되므로 실행 중에 잘못된 유효 스왑 수수료가 사용됩니다.

둘째, `BunniQuoter::quoteSwap`은 시뮬레이션된 스왑에 대한 가격 및 수수료 추정치를 제공하려고 하지만, 실제 `BunniHookLogic::beforeSwap` 로직에서 프로토콜 및 am-AMM 수수료와 함께 공제되는 큐레이터 수수료의 적용을 생략합니다:

```solidity
curatorFeeAmount = baseSwapFeeAmount.mulDivUp(curatorFees.feeRate, CURATOR_FEE_BASE);
```

따라서 이는 견적이 실제 실행과 다르게 되는 두 구현 간의 추가적인 불일치를 나타내며, 이 경우 견적보다 더 많이 청구됩니다. 이 불일치의 일부는 `HookStorage`의 `curatorFees`에 직접 액세스할 수 없는 뷰 함수로서의 `BunniQuoter::quoteSwap` 설계에서 비롯됩니다.

**파급력:** `Swap` 이벤트에 의존하는 오프체인 구성 요소는 잘못된 스왑 수수료 값을 참조할 수 있습니다. `BunniQuoter::quoteSwap`에 의존하여 스왑 수수료/출력을 추정하는 인터페이스는 총 수수료를 과소평가하고 사용자 출력을 과대평가하게 됩니다. 회계 또는 추가 수수료 계산을 위해 `swapFee`에 의존하는 후클릿(Hooklet)은 잘못된 금액을 청구할 수 있습니다. 후크 수수료 수정자 및 큐레이터 수수료율이 0이 아닌 경우 실제 공제에 사용되고 이벤트에서 방출되는 스왑 수수료는 견적에 의해 계산된 것보다 작습니다.

**개념 증명 (Proof of Concept):** 다음 퍼즈 테스트를 `BunniHook.t.sol`에 추가하여 동등성을 주장할 수 있으며, 현재는 그렇지 않음을 보여줍니다:

```solidity
function test_fuzz_quoter_quoteSwap(
    uint256 swapAmount,
    bool zeroForOne,
    bool amAmmEnabled
) external {
    swapAmount = bound(swapAmount, 1e6, 1e36);

    // deploy mock hooklet with all flags
    bytes32 salt;
    unchecked {
        bytes memory creationCode = type(HookletMock).creationCode;
        for (uint256 offset; offset < 100000; offset++) {
            salt = bytes32(offset);
            address deployed = computeAddress(address(this), salt, creationCode);
            if (
                uint160(bytes20(deployed)) & HookletLib.ALL_FLAGS_MASK == HookletLib.ALL_FLAGS_MASK
                    && deployed.code.length == 0
            ) {
                break;
            }
        }
    }
    HookletMock hooklet = new HookletMock{salt: salt}();

    ILiquidityDensityFunction ldf_ =
        new UniformDistribution(address(hub), address(bunniHook), address(quoter));
    bytes32 ldfParams = bytes32(abi.encodePacked(ShiftMode.STATIC, int24(-5) * TICK_SPACING, int24(5) * TICK_SPACING));

    (IBunniToken bunniToken, PoolKey memory key) = hub.deployBunniToken(
        IBunniHub.DeployBunniTokenParams({
            currency0: Currency.wrap(address(token0)),
            currency1: Currency.wrap(address(token1)),
            tickSpacing: TICK_SPACING,
            twapSecondsAgo: 0,
            liquidityDensityFunction: ldf_,
            hooklet: hooklet,
            ldfType: LDFType.STATIC,
            ldfParams: ldfParams,
            hooks: bunniHook,
            hookParams: abi.encodePacked(
                FEE_MIN,
                FEE_MAX,
                FEE_QUADRATIC_MULTIPLIER,
                FEE_TWAP_SECONDS_AGO,
                POOL_MAX_AMAMM_FEE,
                SURGE_HALFLIFE,
                SURGE_AUTOSTART_TIME,
                VAULT_SURGE_THRESHOLD_0,
                VAULT_SURGE_THRESHOLD_1,
                REBALANCE_THRESHOLD,
                REBALANCE_MAX_SLIPPAGE,
                REBALANCE_TWAP_SECONDS_AGO,
                REBALANCE_ORDER_TTL,
                amAmmEnabled,
                ORACLE_MIN_INTERVAL,
                uint48(1)
            ),
            vault0: ERC4626(address(0)),
            vault1: ERC4626(address(0)),
            minRawTokenRatio0: 0.08e6,
            targetRawTokenRatio0: 0.1e6,
            maxRawTokenRatio0: 0.12e6,
            minRawTokenRatio1: 0.08e6,
            targetRawTokenRatio1: 0.1e6,
            maxRawTokenRatio1: 0.12e6,
            sqrtPriceX96: TickMath.getSqrtPriceAtTick(4),
            name: bytes32("BunniToken"),
            symbol: bytes32("BUNNI-LP"),
            owner: address(this),
            metadataURI: "metadataURI",
            salt: ""
        })
    );

    // make initial deposit to avoid accounting for MIN_INITIAL_SHARES
    uint256 depositAmount0 = PRECISION;
    uint256 depositAmount1 = PRECISION;
    vm.startPrank(address(0x6969));
    token0.approve(address(PERMIT2), type(uint256).max);
    token1.approve(address(PERMIT2), type(uint256).max);
    weth.approve(address(PERMIT2), type(uint256).max);
    PERMIT2.approve(address(token0), address(hub), type(uint160).max, type(uint48).max);
    PERMIT2.approve(address(token1), address(hub), type(uint160).max, type(uint48).max);
    PERMIT2.approve(address(weth), address(hub), type(uint160).max, type(uint48).max);
    vm.stopPrank();

    _makeDepositWithFee(key, depositAmount0, depositAmount1, address(0x6969), 0, 0, "");

    vm.prank(HOOK_FEE_RECIPIENT_CONTROLLER);
    bunniHook.setHookFeeRecipient(HOOK_FEE_RECIPIENT);
    bunniHook.setHookFeeModifier(HOOK_FEE_MODIFIER);
    bunniHook.curatorSetFeeRate(key.toId(), uint16(MAX_CURATOR_FEE));

    if (amAmmEnabled) {
        _makeDeposit(key, 1_000_000_000 * depositAmount0, 1_000_000_000 * depositAmount1, address(this), "");

        bunniToken.approve(address(bunniHook), type(uint256).max);
        bunniHook.bid({
            id: key.toId(),
            manager: address(this),
            payload: bytes6(bytes3(POOL_MAX_AMAMM_FEE)),
            rent: 1e18,
            deposit: uint128(K) * 1e18
        });

        skipBlocks(K);

        IAmAmm.Bid memory bid = bunniHook.getBid(key.toId(), true);
        assertEq(bid.manager, address(this), "manager incorrect");
    }

    (Currency inputToken, Currency outputToken) =
        zeroForOne ? (key.currency0, key.currency1) : (key.currency1, key.currency0);
    _mint(inputToken, address(this), swapAmount * 2);
    IPoolManager.SwapParams memory params = IPoolManager.SwapParams({
        zeroForOne: zeroForOne,
        amountSpecified: -int256(swapAmount),
        sqrtPriceLimitX96: zeroForOne ? TickMath.MIN_SQRT_PRICE + 1 : TickMath.MAX_SQRT_PRICE - 1
    });

    // quote swap
    (bool success,,, uint256 inputAmount, uint256 outputAmount, uint24 swapFee,) = quoter.quoteSwap(address(this), key, params);
    assertTrue(success, "quoteSwap failed");

    // execute swap
    uint256 actualInputAmount;
    uint256 actualOutputAmount;
    {
        uint256 beforeInputTokenBalance = inputToken.balanceOfSelf();
        uint256 beforeOutputTokenBalance = outputToken.balanceOfSelf();
        vm.recordLogs();
        swapper.swap(key, params, type(uint256).max, 0);
        actualInputAmount = beforeInputTokenBalance - inputToken.balanceOfSelf();
        actualOutputAmount = outputToken.balanceOfSelf() - beforeOutputTokenBalance;
    }

    // check if swapFee matches
    Vm.Log[] memory logs = vm.getRecordedLogs();
    Vm.Log memory swapLog;
    for (uint256 i = 0; i < logs.length; i++) {
        if (logs[i].emitter == address(bunniHook) && logs[i].topics[0] == keccak256("Swap(bytes32,address,bool,bool,uint256,uint256,uint160,int24,uint24,uint256)")) {
            swapLog = logs[i];
            break;
        }
    }

    // parse log and extract swapFee from calldata
    (,,,,,,uint24 fee,) = abi.decode(swapLog.data, (
        bool,
        bool,
        uint256,
        uint256,
        uint160,
        int24,
        uint24,
        uint256
    ));

    assertEq(swapFee, fee, "swapFee doesn't match quoted swapFee");

    // check if actual amounts match quoted amounts
    assertEq(actualInputAmount, inputAmount, "actual input amount doesn't match quoted input amount");
    assertEq(actualOutputAmount, outputAmount, "actual output amount doesn't match quoted output amount");
}
```

**권장 완화 방법:** 후크 수수료 수정자 및 큐레이터 수수료 로직이 `BunniQuoter::quoteSwap` 및 `BunniHookLogic::beforeSwap` 모두에서 일관되게 적용되는지 확인하십시오. 이를 위해 풀당 큐레이터 수수료율을 조회하는 뷰 함수를 노출해야 할 수도 있습니다.

`outputAmount`/`swapFeeAmount` 조정은 로직이 약간 일관성이 없어 비교하기 어렵기 때문에 혼란스럽습니다. 가능하면 이 공통 공유 로직을 별도의 라이브러리 함수로 분리하여 추가적인 불일치가 발생할 가능성을 최소화하는 것을 고려하십시오.

**Bacon Labs:** 커밋 [2a54265](https://github.com/Bunniapp/bunni-v2/pull/135/commits/2a542654f0f13425a20306df4f7c82bb05685ae5) 및 [99dd8d4](https://github.com/Bunniapp/bunni-v2/pull/135/commits/99dd8d485b342a298eec028db91b54b90f63208d)에서 수정되었습니다.

**Cyfrin:** 확인됨. 쿼터(quoter) 및 후크 구현이 이제 일관됩니다.

\clearpage
## 낮은 위험 (Low Risk)

### `HookletLib::hookletBeforeSwap`의 불충분한 클램핑으로 인해 예상치 못한 되돌림(revert)이 발생할 수 있음

**설명:** `HookletLib::hookletBeforeSwap` 및 `HookletLib::hookletBeforeSwapView`는 외부 후클릿 호출에서 반환된 수수료 및 가격 재정의 데이터를 디코딩하고 이러한 값에 클램핑 작업을 적용하여 지정된 범위 내에 있는지 확인합니다.

```solidity
// clamp the override values to the valid range
fee = feeOverridden ? uint24(fee.clamp(0, SWAP_FEE_BASE)) : 0;
sqrtPriceX96 =
    priceOverridden ? uint160(sqrtPriceX96.clamp(TickMath.MIN_SQRT_PRICE, TickMath.MAX_SQRT_PRICE)) : 0;
```

그러나 다음 엣지 케이스는 고려되지 않았습니다:
* `SWAP_FEE_BASE`를 포괄적인 상한선으로 사용하는 것은 `BunniHookLogic::isValidParams` 및 `FeeOverrideHooklet::setFeeOverride`와 같이 `SWAP_FEE_BASE`와 동일한 수수료가 명시적으로 방지되는 코드베이스 전체의 다른 곳의 유효성 검사와 모순됩니다. 이 값을 허용하면 `SWAP_FEE_BASE - fee`가 0이 아니어야 하는 `BunniHookLogic::beforeSwap`과 같은 다운스트림 로직에서 0으로 나누기 오류가 발생할 수 있습니다.
* 재정의된 `sqrtPriceX96`을 `[TickMath.MIN_SQRT_PRICE, TickMath.MAX_SQRT_PRICE]` 범위로 제한하면 백서에 정의된 대로 사용 가능한 틱 범위 $r_{\min} = \left\lfloor \frac{t_{\min}}{w} \right\rfloor\cdot w$ 및 $r_{\max} = \bigl(\lfloor \tfrac{t_{\max}}{w} \rfloor - 1\bigr) \cdot w$에 해당하는 sqrt 가격을 고려하지 못합니다. 이 범위 밖의 가격을 허용하면 `getSqrtPriceAtTick()`이 호출될 때 잘못된 틱 사용으로 인해 되돌림(revert)이 발생할 수 있습니다.

**파급력:** 수수료 재정의 값이 `SWAP_FEE_BASE`와 같으면 `BunniHookLogic::beforeSwap`에서 추가 스왑 실행 중 0으로 나누기로 인해 되돌림이 발생할 수 있습니다. sqrt 가격이 해당 틱이 사용 가능한 틱 범위 밖에 있는 경우 이 호출 내에서 실행이 유사하게 되돌려집니다.

**개념 증명 (Proof of Concept):** 다음 패치를 적용하십시오:

```diff
---
 test/BunniHook.t.sol | 110 +++++++++++++++++--------------------------
 1 file changed, 41 insertions(+), 65 deletions(-)

diff --git a/test/BunniHook.t.sol b/test/BunniHook.t.sol
index 0a7978bd..25f3eb08 100644
--- a/test/BunniHook.t.sol
+++ b/test/BunniHook.t.sol
@@ -8,6 +8,9 @@ import {IAmAmm} from "biddog/interfaces/IAmAmm.sol";
 import "./BaseTest.sol";
 import {BunniStateLibrary} from "../src/lib/BunniStateLibrary.sol";

+import {SWAP_FEE_BASE} from "src/base/Constants.sol";
+import {CustomRevert} from "@uniswap/v4-core/src/libraries/CustomRevert.sol";
+
 contract BunniHookTest is BaseTest {
     using TickMath for *;
     using FullMathX96 for *;
@@ -1388,12 +1391,6 @@ contract BunniHookTest is BaseTest {
         ldf_.setMinTick(-30);

         // deploy pool with hooklet
-        // this should trigger:
-        // - before/afterInitialize
-        // - before/afterDeposit
-        uint24 feeMin = 0.3e6;
-        uint24 feeMax = 0.5e6;
-        uint24 feeQuadraticMultiplier = 1e6;
         (Currency currency0, Currency currency1) = (Currency.wrap(address(token0)), Currency.wrap(address(token1)));
         (IBunniToken bunniToken, PoolKey memory key) = _deployPoolAndInitLiquidity(
             currency0,
@@ -1401,11 +1398,12 @@ contract BunniHookTest is BaseTest {
             ERC4626(address(0)),
             ERC4626(address(0)),
             ldf_,
+            IHooklet(hooklet),
             ldfParams,
             abi.encodePacked(
-                feeMin,
-                feeMax,
-                feeQuadraticMultiplier,
+                uint24(0.3e6),
+                uint24(0.5e6),
+                uint24(1e6),
                 FEE_TWAP_SECONDS_AGO,
                 POOL_MAX_AMAMM_FEE,
                 SURGE_HALFLIFE,
@@ -1419,68 +1417,46 @@ contract BunniHookTest is BaseTest {
                 true, // amAmmEnabled
                 ORACLE_MIN_INTERVAL,
                 MIN_RENT_MULTIPLIER
-            )
+            ),
+            bytes32("")
         );
-        address depositor = address(0x6969);
-
-        // transfer bunniToken
-        // this should trigger:
-        // - before/afterTransfer
-        address recipient = address(0x8008);
-        vm.startPrank(depositor);
-        bunniToken.transfer(recipient, bunniToken.balanceOf(depositor));
-        vm.stopPrank();
-        vm.startPrank(recipient);
-        bunniToken.transfer(depositor, bunniToken.balanceOf(recipient));
-        vm.stopPrank();

-        // withdraw liquidity
-        // this should trigger:
-        // - before/afterWithdraw
-        vm.startPrank(depositor);
-        hub.withdraw(
-            IBunniHub.WithdrawParams({
-                poolKey: key,
-                recipient: depositor,
-                shares: bunniToken.balanceOf(depositor),
-                amount0Min: 0,
-                amount1Min: 0,
-                deadline: block.timestamp,
-                useQueuedWithdrawal: false
-            })
-        );
-        vm.stopPrank();
-
-        // shift LDF to trigger rebalance during the next swap
-        ldf_.setMinTick(-20);
-
-        // make swap
-        // this should trigger:
-        // - before/afterSwap
+        // make swap to trigger beforeSwap
         _mint(currency0, address(this), 1e6);
         IPoolManager.SwapParams memory params = IPoolManager.SwapParams({
-            zeroForOne: true,
-            amountSpecified: -int256(1e6),
-            sqrtPriceLimitX96: TickMath.MIN_SQRT_PRICE + 1
+            zeroForOne: false,
+            amountSpecified: int256(1e6),
+            sqrtPriceLimitX96: TickMath.MAX_SQRT_PRICE - 1
         });
-        vm.recordLogs();
-        _swap(key, params, 0, "");

-        // fill rebalance order
-        // this should trigger:
-        // - afterRebalance
-        Vm.Log[] memory logs = vm.getRecordedLogs();
-        Vm.Log memory orderEtchedLog;
-        for (uint256 i = 0; i < logs.length; i++) {
-            if (logs[i].emitter == address(floodPlain) && logs[i].topics[0] == IOnChainOrders.OrderEtched.selector) {
-                orderEtchedLog = logs[i];
-                break;
-            }
-        }
-        IFloodPlain.SignedOrder memory signedOrder = abi.decode(orderEtchedLog.data, (IFloodPlain.SignedOrder));
-        IFloodPlain.Order memory order = signedOrder.order;
-        _mint(key.currency0, address(this), order.consideration.amount);
-        floodPlain.fulfillOrder(signedOrder);
+        hooklet.setBeforeSwapOverride(true, uint24(SWAP_FEE_BASE), false, uint24(0));
+        vm.expectRevert(
+            abi.encodeWithSelector(
+                CustomRevert.WrappedError.selector,
+                key.hooks,
+                BunniHook.beforeSwap.selector,
+                abi.encodePacked(bytes4(keccak256("MulDivFailed()"))),
+                abi.encodePacked(bytes4(keccak256("HookCallFailed()")))
+            )
+        );
+        _swap(key, params, 0, "");
+
+        int24 tickAtPrice = TickMath.MIN_TICK;
+        uint160 priceOverride = TickMath.getSqrtPriceAtTick(tickAtPrice);
+        hooklet.setBeforeSwapOverride(false, uint24(0), true, priceOverride);
+        vm.expectRevert(
+            abi.encodeWithSelector(
+                CustomRevert.WrappedError.selector,
+                key.hooks,
+                BunniHook.beforeSwap.selector,
+                abi.encodeWithSelector(
+                    TickMath.InvalidTick.selector,
+                    tickAtPrice - ((tickAtPrice % key.tickSpacing + key.tickSpacing) % key.tickSpacing)
+                ),
+                abi.encodePacked(bytes4(keccak256("HookCallFailed()")))
+            )
+        );
+        _swap(key, params, 0, "");
     }
--
2.40.0

```

**권장 완화 방법:** `HookletLib::hookletBeforeSwap` 및 `HookletLib::hookletBeforeSwapView`의 클램핑 로직을 업데이트하여 `SWAP_FEE_BASE`에서 1을 뺌으로써 이것이 유효한 상한으로 허용되는 것을 방지하고 유사한 검증이 이미 수행된 다른 인스턴스와 동작을 일치시키십시오:

```diff
- fee = feeOverridden ? uint24(fee.clamp(0, SWAP_FEE_BASE)) : 0;
+ fee = feeOverridden ? uint24(fee.clamp(0, SWAP_FEE_BASE - 1)) : 0;
```

또한 재정의된 가격이 사용 가능한 틱 범위를 벗어난 틱에 해당하는 것으로 지정되는 것을 방지하십시오:

```diff
- sqrtPriceX96 =
-   priceOverridden ? uint160(sqrtPriceX96.clamp(TickMath.MIN_SQRT_PRICE, TickMath.MAX_SQRT_PRICE)) : 0;
+ sqrtPriceX96 =
+   priceOverridden ? uint160(sqrtPriceX96.clamp(TickMath.getSqrtPriceAtTick((TickMath.MIN_TICK / key.tickSpacing) * key.tickSpacing), TickMath.getSqrtPriceAtTick(((TickMath.MAX_TICK / key.tickSpacing) - 1) * key.tickSpacing))) : 0;
```

**Bacon Labs:** 커밋 [45643ef](https://github.com/Bunniapp/bunni-v2/pull/135/commits/45643eff21be8d3671bea025aae5ffd11a0f6467) 및 [0b5a708](https://github.com/Bunniapp/bunni-v2/pull/135/commits/0b5a708e5bafb12308dbab1c8db478ac991bae2f)에서 수정되었습니다.

**Cyfrin:** 확인됨. 수수료 클램핑 로직은 `SWAP_FEE_BASE`가 사용되는 것을 방지하도록 수정되었습니다. 가격 클램핑 로직 또한 최소 및 최대 사용 가능 틱에 해당하는 sqrt 가격 사이의 재정의를 제한하도록 수정되었습니다.

\clearpage
## 정보성 (Informational)

### `FeeOverrideHooklet::setFeeOverride`에 대한 멀티콜 지원 부족

**설명:** `FeeOverrideHooklet::setFeeOverride`는 해당 `BunniToken`의 소유권을 확인할 때 `msg.sender`에 의존합니다. 그러나 이는 멀티콜러(multicaller) 계약을 통해 호출될 때 부정확하며, 원래 호출자가 실제 소유자이더라도 검증에 실패하게 합니다.

모든 계정이 자유롭게 Bunni 풀을 배포할 수 있고 `LibMulticaller`가 핵심 Bunni 계약 전체에서 많이 사용된다는 점을 감안할 때, 풀 소유자가 후클릿(hooklet)과 Bunni 자체 모두에서 일괄 작업을 수행하도록 허용하는 것이 바람직할 수 있습니다.

**파급력:** 멀티콜러 계약을 통해 `FeeOverrideHooklet`과 상호 작용하는 합법적인 풀 소유자는 그렇게 하지 못하도록 잘못 차단되어 멀티콜 지원에 의존하는 외부 호환성이 손상될 수 있습니다.

**권장 완화 방법:** `msg.sender`의 직접 사용을 `LibMulticaller::senderOrSigner`로 교체하여 핵심 Bunni 계약과 일관성을 유지하십시오. 이 접근 방식은 직접 및 일괄 호출과의 호환성을 모두 유지하여 멀티콜 인프라를 지원하는 동시에 적절한 액세스 제어를 보장합니다.

**Bacon Labs:** 커밋 [9cf16a8](https://github.com/Bunniapp/hooklets/commit/9cf16a8400f25f5f9eeb2837915c76adc6dc4f54)에서 수정되었습니다.

**Cyfrin:** 확인됨. `FeeOverrideHooklet::setFeeOverride`는 이제 멀티콜 호출과 호환됩니다.

### `IHooklet::afterSwap`의 오래된 재조정 참조는 제거되어야 함

**설명:** `IHooklet::afterRebalance`가 그 이후 추가되었으므로 제거되어야 하는 `IHooklet::afterSwap` 개발 주석에 재조정에 대한 오래된 참조가 있습니다:

```solidity
    /// @notice Called after a swap operation.
@>  /// @dev Also called after a rebalance order execution, in which case returnData will only have
@>  /// inputAmount and outputAmount filled out.
    /// @param sender The address of the account that initiated the swap.
    /// @param key The Uniswap v4 pool's key.
    /// @param params The swap's input parameters.
    /// @param returnData The swap operation's return data.
    function afterSwap(
        address sender,
        PoolKey calldata key,
        IPoolManager.SwapParams calldata params,
        SwapReturnData calldata returnData
    ) external returns (bytes4 selector);
```

**Bacon Labs:** 커밋 [0189567](https://github.com/Bunniapp/bunni-v2/pull/135/commits/018956782d344d0d5e27a0fc193872dce430e1a4)에서 수정되었습니다.

**Cyfrin:** 확인됨. 참조가 제거되었습니다.

### `BeforeSwapFeeOverride` 구조체는 `FeeOverride` 구조체에서 사용될 수 있음

**설명:** `IHooklet`에 정의된 `BeforeSwapFeeOverride` 및 `BeforeSwapPriceOverride` 구조체는 현재 사용되지 않습니다:

```solidity
/// @notice Overrides the swap fee of a pool before the swap is executed.
/// Ignored if the pool has an am-AMM manager.
/// @member overridden If true, the swap fee is overridden.
/// @member fee The swap fee to use for the swap. 6 decimals.
struct BeforeSwapFeeOverride {
    bool overridden;
    uint24 fee;
}

/// @notice Overrides the pool's spot price before the swap is executed.
/// @member overridden If true, the pool's spot price is overridden.
/// @member sqrtPriceX96 The spot price to use for the swap. Q96 value.
struct BeforeSwapPriceOverride {
    bool overridden;
    uint160 sqrtPriceX96;
}
```

`FeeOverrideHooklet`에 정의된 `FeeOverride` 구조체는 대신 0/1에 대해 두 개의 `BeforeSwapFeeOverride` 멤버로 구성될 수 있습니다:

```solidity
struct FeeOverride {
    bool overrideZeroToOne;
    uint24 feeZeroToOne;
    bool overrideOneToZero;
    uint24 feeOneToZero;
}
```

**권장 완화 방법:**
```diff
    struct FeeOverride {
-       bool overrideZeroToOne;
-       uint24 feeZeroToOne;
-       bool overrideOneToZero;
-       uint24 feeOneToZero;
+       BeforeSwapFeeOverride overrideZeroToOne;
+       BeforeSwapFeeOverride overrideOneToZero;
    }
```

**Bacon Labs:** 구조체 정의를 사용하는 대신 커밋 [fc18b04](https://github.com/Bunniapp/bunni-v2/pull/135/commits/fc18b0434c3fb487cc91f6c8dd3cc7b29549b137)에서 사용되지 않으므로 핵심 계약에서 완전히 제거하기로 결정했습니다. 따라서 제안된 수정 사항을 구현하지 않습니다.

**Cyfrin:** 인지함. 업스트림의 사용되지 않는 구조체가 제거되었습니다.

### `FeeOverrideHooklet::setFeeOverride`에서 수수료 재정의를 제한할 필요가 없음. `HookletLib::beforeSwap`에서 클램핑되기 때문임

**설명:** `FeeOverrideHooklet::setFeeOverride`는 재정의된 수수료가 `SWAP_FEE_BASE`를 초과하지 않도록 제한합니다:

```solidity
function setFeeOverride(
    PoolId id,
    bool overrideZeroToOne,
    uint24 feeZeroToOne,
    bool overrideOneToZero,
    uint24 feeOneToZero
) public {
    if (feeZeroToOne >= SWAP_FEE_BASE || feeOneToZero >= SWAP_FEE_BASE) {
        revert FeeOverrideHooklet__InvalidSwapFee();
    }

    ...
}
```

L-01이 제안된 대로 올바르게 완화되었다고 가정하면, `HookletLib::beforeSwap` 로직에 의해 수수료가 클램핑되므로 후클릿에서 이 검증을 수행할 필요가 없습니다.

**권장 완화 방법:** 여러 인스턴스에서 동일한 검증을 수행하지 말고 대신 핵심 Bunni 계약의 동작에 의존하십시오.

```diff
    function setFeeOverride(
        PoolId id,
        bool overrideZeroToOne,
        uint24 feeZeroToOne,
        bool overrideOneToZero,
        uint24 feeOneToZero
    ) public {
-       if (feeZeroToOne >= SWAP_FEE_BASE || feeOneToZero >= SWAP_FEE_BASE) {
-           revert FeeOverrideHooklet__InvalidSwapFee();
-       }

        ...
    }
```

**Bacon Labs:** 인지함. 우리는 핵심 계약의 클램핑 동작에만 의존하는 것보다 이 검증을 유지하려고 합니다. 이 검증이 없으면 풀 큐레이터가 허용 가능한 최대값보다 높은 수수료를 설정할 수 있는 시나리오가 있으며, 이는 핵심 계약에서 최대값으로 클램핑되어 큐레이터가 설정한 것과 다른 스왑 수수료 값으로 이어집니다. 이 검증을 유지하면 혼란의 가능성을 제거하고 가스 비용의 약간의 증가 외에는 다른 해가 되지 않습니다.

**Cyfrin:** 인지함.

\clearpage
