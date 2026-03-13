# 높은 중요도 (High Risk)


### 프로토콜의 불변성이 깨질 수 있음 (Protocol's invariants can be broken)

**설명:** 이 프로토콜은 상수 함수(constant-function) AMM 유동성 풀을 위한 일반화된 프레임워크를 제공하는 것을 목표로 합니다.
우리는 언제나 유지되어야 하는 몇 가지 불변성(invariants)을 확인했습니다. 그중 하나는 `totalSupply() == calcLpTokenSupply(reserves)`이며, 이는 풀의 총 LP 공급량이 현재 `reserves` 상태 값으로부터 계산된 LP와 일치해야 한다는 것으로 해석할 수 있습니다.

현재 구현에서는 유효한 트랜잭션으로 인해 이 불변성이 깨질 수 있으며, 이는 여러 문제로 이어질 수 있습니다. 예를 들어, 아래의 개념 증명(PoC) 테스트에서 보여지는 것처럼 유효한 유동성 제거가 되돌려질(revert) 수 있습니다.

**파급력:** 이 불일치는 프로토콜의 지불 불능(유효한 트랜잭션의 revert)으로 이어지므로 파급력은 **높음(HIGH)**입니다.

**개념 증명 (Proof of Concept):**
```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

import {IERC20} from "test/TestHelper.sol";
import {IWellFunction} from "src/functions/ConstantProduct2.sol";
import {LiquidityHelper} from "test/LiquidityHelper.sol";
import "forge-std/Test.sol";
import {MockToken} from "mocks/tokens/MockToken.sol";

contract GetRemoveLiquidityOneTokenOutArithmeticFail is LiquidityHelper {
    function setUp() public {
        setupWell(2);
    }

    address internal constant ADDRESS_0 = 0x0000000000000000000000000000000000001908;
    address internal constant ADDRESS_1 = 0xffffFFFfFFffffffffffffffFfFFFfffFFFfFFfE;
    address internal constant ADDRESS_2 = 0x0000000000000000000000000000000000000001;
    address internal constant ADDRESS_3 = 0x00000011EF7b76c418fC426B8707dC60c8404f4a;

    // Failing unit tests from invariant results
    function test_getRemoveLiquidityOneTokenOutArithmeticFail() public {
        showStatus();

        address msgSender = ADDRESS_0;
        _addLiquidity(msgSender, 77_470_052_844_788_801_811_950_156_551, 17_435);
        _removeLiquidityOneTokenOut(msgSender, 5267, 0);
        showStatus();

        msgSender = ADDRESS_1;
        _addLiquidity(msgSender, 79_228_162_514_264_337_593_543_950_335, 0);
        _removeLiquidity(msgSender, 2_025_932_259_663_320_959_193_637_370_794);
        showStatus();

        msgSender = ADDRESS_2;
        _addLiquidity(msgSender, 69_069_904_726_099_247_337_000_262_288, 3);
        showStatus();

        msgSender = ADDRESS_1;
        _transferLp(msgSender, ADDRESS_3, 1_690_276_116_468_540_706_301_324_542_928);
        emit log_named_uint("LP Balance (1)", well.balanceOf(ADDRESS_1));
        emit log_named_uint("LP Balance (3)", well.balanceOf(ADDRESS_3));
        showStatus();

        msgSender = ADDRESS_0;
        emit log_string("removeLiquidity");
        _removeLiquidity(msgSender, 122_797_404_990_851_137_316_041_024_188);
        showStatus();

        msgSender = ADDRESS_3;
        emit log_string("removeLiquidityOneToken");
        _removeLiquidityOneTokenOut(msgSender, 1_690_276_116_468_540_706_301_000_000_000, 1);
        emit log_named_uint("LP Balance (3)", well.balanceOf(ADDRESS_3));
        showStatus();
        // The next line fails with an under/overflow error
        //
        // CONTEXT: In the previous operation, ADDRESS_3 removes the vast majority of his LP position
        // for token[1]. At this point the token balances of the well are as follows:
        //  * token[0].balanceOf(well) = 198508852592404865716451834587
        //  * token[1].balanceOf(well) =          625986797429655048967
        // The next operation ADDRESS_3 calls is removeLiquidityOneTokenOut() for token[0] using his
        // remaining LP position. The amount of LP tokens he has left is 3. Line 526 reverts with underflow,
        // despite all operations being completely valid. How severe is this?
        _removeLiquidityOneTokenOut(msgSender, 324_542_928, 0);
        showStatus();
    }

    function test_audit() public {
        address msgSender = address(this);
        _removeLiquidityOneTokenOut(msgSender, 1e16, 0);
        _removeLiquidity(msgSender, well.balanceOf(msgSender) - 1);
        _removeLiquidity(msgSender, 1);
    }

    function test_totalSupplyInvariantSwapFail() public {
        address msgSender = 0x576024f76bd1640d7399a5B5F61530f997Ae06f2;
        changePrank(msgSender);
        IERC20[] memory mockTokens = well.tokens();
        uint tokenInIndex = 0;
        uint tokenOutIndex = 1;
        uint tokenInAmount = 52_900_000_000_000_000_000;
        MockToken(address(mockTokens[tokenInIndex])).mint(msgSender, tokenInAmount);
        mockTokens[tokenInIndex].approve(address(well), tokenInAmount);
        uint minAmountOut = well.getSwapOut(mockTokens[tokenInIndex], mockTokens[tokenOutIndex], tokenInAmount);
        well.swapFrom(
            mockTokens[tokenInIndex], mockTokens[tokenOutIndex], tokenInAmount, minAmountOut, msgSender, block.timestamp
        );
        // check the total supply
        uint functionCalc =
            IWellFunction(well.wellFunction().target).calcLpTokenSupply(well.getReserves(), well.wellFunction().data);
        // assertEq(well.totalSupply(), functionCalc);
        showStatus();
    }

    function _transferLp(address msgSender, address to, uint amount) private {
        changePrank(msgSender);
        well.transfer(to, amount);
    }

    function _removeLiquidityOneTokenOut(
        address msgSender,
        uint lpAmountIn,
        uint tokenIndex
    ) private returns (uint tokenAmountOut) {
        changePrank(msgSender);
        IERC20[] memory mockTokens = well.tokens();
        uint minTokenAmountOut = well.getRemoveLiquidityOneTokenOut(lpAmountIn, mockTokens[tokenIndex]);
        tokenAmountOut = well.removeLiquidityOneToken(
            lpAmountIn, mockTokens[tokenIndex], minTokenAmountOut, msgSender, block.timestamp
        );
    }

    function _removeLiquidity(address msgSender, uint lpAmountIn) private returns (uint[] memory tokenAmountsOut) {
        changePrank(msgSender);
        uint[] memory minTokenAmountsOut = well.getRemoveLiquidityOut(lpAmountIn);
        tokenAmountsOut = well.removeLiquidity(lpAmountIn, minTokenAmountsOut, msgSender, block.timestamp);
    }

    function _addLiquidity(
        address msgSender,
        uint token0Amount,
        uint token1Amount
    ) private returns (uint lpAmountOut) {
        changePrank(msgSender);
        uint[] memory tokenAmountsIn = _mintToSender(msgSender, token0Amount, token1Amount);
        uint minLpAmountOut = well.getAddLiquidityOut(tokenAmountsIn);

        lpAmountOut = well.addLiquidity(tokenAmountsIn, minLpAmountOut, msgSender, block.timestamp);
    }

    function _mintToSender(
        address msgSender,
        uint token0Amount,
        uint token1Amount
    ) private returns (uint[] memory tokenAmountsIn) {
        changePrank(msgSender);
        IERC20[] memory mockTokens = well.tokens();
        MockToken(address(mockTokens[0])).mint(msgSender, token0Amount);
        MockToken(address(mockTokens[1])).mint(msgSender, token1Amount);

        tokenAmountsIn = new uint[](2);
        tokenAmountsIn[0] = token0Amount;
        tokenAmountsIn[1] = token1Amount;

        mockTokens[0].approve(address(well), token0Amount);
        mockTokens[1].approve(address(well), token1Amount);
    }

    function showStatus() public {
        IERC20[] memory mockTokens = well.tokens();
        uint[] memory reserves = well.getReserves();

        uint calcedSupply = IWellFunction(well.wellFunction().target).calcLpTokenSupply(well.getReserves(), well.wellFunction().data);
        emit log_named_uint("Total  LP Supply", well.totalSupply());
        emit log_named_uint("Calced LP Supply", calcedSupply);
    }
}
```

테스트 결과는 아래와 같습니다.

```
Running 1 test for test/invariant/RemoveOneLiquidity.t.sol:GetRemoveLiquidityOneTokenOutArithmeticFail
[FAIL. Reason: Arithmetic over/underflow] test_getRemoveLiquidityOneTokenOutArithmeticFail() (gas: 1029158)
Logs:
  Total  LP Supply: 1000000000000000000000000000
  Calced LP Supply: 1000000000000000000000000000
  Total  LP Supply: 8801707439172742871919189288925
  Calced LP Supply: 8801707439172742871919189288909
  Total  LP Supply: 10491983555641283578220513831854
  Calced LP Supply: 10491983555641283578222260432821
  Total  LP Supply: 12960446346289240477619201802843
  Calced LP Supply: 12960446346289240477619201802843
  LP Balance (1): 1
  LP Balance (3): 1690276116468540706301324542928
  Total  LP Supply: 12960446346289240477619201802843
  Calced LP Supply: 12960446346289240477619201802843
  removeLiquidity
  Total  LP Supply: 12837648941298389340303160778655
  Calced LP Supply: 12837648941298389340310429464350
  removeLiquidityOneToken
  LP Balance (3): 324542928
  Total  LP Supply: 11147372824829848634002160778655
  Calced LP Supply: 11147372824829848633998681214926

Test result: FAILED. 0 passed; 1 failed; finished in 3.70ms

Failing tests:
Encountered 1 failing test in test/invariant/RemoveOneLiquidity.t.sol:GetRemoveLiquidityOneTokenOutArithmeticFail
[FAIL. Reason: Arithmetic over/underflow] test_getRemoveLiquidityOneTokenOutArithmeticFail() (gas: 1029158)

Encountered a total of 1 failing tests, 0 tests succeeded
```

`ConstantProduct2`를 살펴보면 세 가지 문제가 확인됩니다:

1. `calcReserve` 및 `calcLpTokenSupply` 함수에서 반올림 방향이 지정되지 않았습니다. 이러한 Well 함수는 Well의 상태(reserve, LP)가 변경되는 중요한 곳에서 사용됩니다. 수학적 불일치는 프로토콜에 유리하게 반올림되어야 하지만, 현재 구현에서는 반올림 방향이 지정되지 않아 불리한 트랜잭션이 허용됩니다.

   예를 들어, [Well::getSwapOut](https://github.com/BeanstalkFarms/Wells/blob/e5441fc78f0fd4b77a898812d0fd22cb43a0af55/src/Well.sol#L231)과 [Well::\_getRemoveLiquidityOneTokenOut](https://github.com/BeanstalkFarms/Wells/blob/e5441fc78f0fd4b77a898812d0fd22cb43a0af55/src/Well.sol#L519)에서 출력 토큰 금액 계산은 올림(rounded up)되지만(하향 반올림된 값을 뺌), 내림(rounded down)되어야 합니다. 이러한 불리한 트랜잭션은 누적되어 프로토콜의 건전성에 영향을 미칩니다.

2. 이 불변성은 `Well::removeLiquidity` 이후에 보장되지 않습니다. 현재 구현은 `totalSupply`에 대한 `lpAmountIn`의 비율로 토큰 출력 금액을 계산하며, `calcLpTokenSupply([r0 * delta, r1 * delta]) = totalSupply() * delta`라고 가정합니다(여기서 `delta`는 `1 - lpAmountIn/lpTokenSupply` 비율을 나타냄). 이는 다른 이슈에서 별도로 언급되었지만, 트랜잭션 후에도 불변성이 유지되도록 보장하는 것이 중요함을 다시 한번 강조합니다. `Well::removeLiquidityOneToken`과 유사하게 마지막 토큰을 별도로 처리하는 것이 권장됩니다.

3. `totalSupply() == calcLpTokenSupply(reserves)` 불변성을 엄격하게 유지하는 것은 거의 불가능합니다. 이 예시에서는 가장 간단하고 합리적인 유동성 인출 방법인 `Well::removeLiquidityOneToken` 함수에 초점을 맞추고 `ConstantProduct2`가 Well 함수라고 가정하겠습니다. 메커니즘은 직관적입니다. 프로토콜은 요청된 `lpAmountIn`을 소각하고 감소된 LP 토큰 공급량을 사용하여 요청된 `tokenOut`(`tokens[0]`)에 대한 새로운 예비(reserve) 값을 계산합니다.

   트랜잭션 전에 `totalSupply() == calcLpTokenSupply(reserves) == T0`라고 가정해 봅시다. 트랜잭션 후 총 공급량은 `T0 - lpAmountIn`이 됩니다. 출력 토큰 금액은 `getRemoveLiquidityOneTokenOut(lpAmountIn, 0, reserves) = reserves[0] - calcReserve(reserves, 0, T0 - lpAmountIn)`으로 계산됩니다. 트랜잭션 후 계산된 총 공급량은 `calcLpTokenSupply([calcReserve(reserves, 0, T0 - lpAmountIn), reserves[1]])`가 됩니다. 트랜잭션 후 불변성이 유지되려면 `ConstantProduct2::calcLpTokenSupply`와 `ConstantProduct2::calcReserve` 함수가 정확한 역관계(`calcLpTokenSupply(calcReserves(reserves, LP)) == LP`)를 보여야 합니다. 실제로 모든 계산에는 어느 정도의 반올림이 수반되며, 두 함수가 별도로 구현되는 한 이러한 관계는 불가능합니다.

**권장 완화 방안:**
1. `IWellFunction::calcReserve` 및 `IWellFunction::calcLpTokenSupply` 함수에 반올림 방향 플래그 매개변수를 추가하세요. 이러한 함수의 모든 호출에서 올바른 반올림 방향을 적용하여 트랜잭션이 프로토콜에 유리하게 처리되도록 하세요.

2. `Well::removeLiquidity`에서 마지막 토큰을 별도로 처리하세요. `Well::getRemoveLiquidityOut`에서 처음 `n-1`개 토큰에 대한 출력 토큰 금액은 평소대로 계산하되, 마지막 토큰에 대해서는 `Well::getRemoveLiquidityOneTokenOut`과 유사한 방식을 사용하세요. 이렇게 하면 불변성을 유지하는 데 도움이 됩니다. 다른 발견 사항에서 `IWellFunction` 수준에서 출력 토큰 금액을 계산하는 전용 함수를 추가할 것을 제안했음을 알고 있습니다. 문제가 그런 방식으로 완화된다면 불변성이 유지되는지 확인하는 또 다른 검사를 추가하는 것을 고려하세요.

3. 위에서 설명했듯이 `totalSupply() == calcLpTokenSupply(reserves)` 불변성을 엄격하게 유지하는 것은 거의 불가능합니다. 우리는 `totalSupply() >= calcLpTokenSupply(reserves)`라는 약간 더 느슨한 불변성을 권장합니다. 이는 "Well 함수가 LP 공급량을 과소평가한다"고 해석할 수 있습니다. 그렇지 않다면 현재 `reserves`에서 계산된 LP 공급량을 항상 반영하는 또 다른 변수 `totalCalcSupply`를 추가할 것을 제안합니다. 이 새로운 변수는 출력 토큰 금액을 결정할 때 `totalSupply()`와 함께 사용할 수 있습니다. 예를 들어, `Well::_getRemoveLiquidityOneTokenOut`에서 `lpAmountInCalc = lpAmountIn * totalCalcSupply / totalSupply()`를 계산하고 이를 `_calcReserve` 함수에서 사용할 수 있습니다. 이렇게 하면 불일치가 계산에 반영되므로 도움이 될 것입니다.

참고로 `well.getReserves()[i] == wellFunction.calcReserve(i)`도 유지되어야 합니다.
계산 시 반올림으로 인해 이 불변성이 너무 엄격할 수 있음을 이해하며, 유사한 완화 조치가 권장됩니다.
아래는 위의 _불변성_이 깨질 수 있음을 보여주는 테스트입니다.
```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

import {IERC20} from "test/TestHelper.sol";
import {IWellFunction} from "src/functions/ConstantProduct2.sol";
import {LiquidityHelper} from "test/LiquidityHelper.sol";
import "forge-std/Test.sol";
import {MockToken} from "mocks/tokens/MockToken.sol";

contract ReservesMathFunctionCalcReservesFail is LiquidityHelper {
    function setUp() public {
        setupWell(2);
    }

    function testReservesMismatchFail() public {
        // perform initial check
        uint[] memory reserves = well.getReserves();

        uint reserve0 =
            IWellFunction(wellFunction.target).calcReserve(reserves, 0, well.totalSupply(), wellFunction.data);
        uint reserve1 =
            IWellFunction(wellFunction.target).calcReserve(reserves, 1, well.totalSupply(), wellFunction.data);

        assertEq(reserves[0], reserve0);
        assertEq(reserves[1], reserve1);

        // swap
        address msgSender = address(2);
        changePrank(msgSender);
        uint tokenInIndex = 0;
        uint tokenOutIndex = 1;
        uint amountIn = 59_534_739_918_926_377_591_512_171_513;

        IERC20[] memory mockTokens = well.tokens();
        MockToken(address(mockTokens[tokenInIndex])).mint(msgSender, amountIn);
        // approve the well
        mockTokens[tokenInIndex].approve(address(well), amountIn);
        uint minAmountOut = well.getSwapOut(mockTokens[tokenInIndex], mockTokens[tokenOutIndex], amountIn);
        well.swapFrom(
            mockTokens[tokenInIndex], mockTokens[tokenOutIndex], amountIn, minAmountOut, msgSender, block.timestamp
        );

        // perform check again
        reserves = well.getReserves();

        reserve0 = IWellFunction(wellFunction.target).calcReserve(reserves, 0, well.totalSupply(), wellFunction.data);
        reserve1 = IWellFunction(wellFunction.target).calcReserve(reserves, 1, well.totalSupply(), wellFunction.data);

        assertEq(reserves[0], reserve0);
        assertEq(reserves[1], reserve1);
    }
}

```

**Beanstalk:**
1. `ConstantProduct2::_calcReserve`는 이제 가장 가까운 숫자가 아닌 올림(round up)을 수행합니다. `ConstantProduct2::_calcLpTokenSupply`는 여전히 내림(round down)을 수행합니다. 이유는 아래와 같습니다. [876da9a](https://github.com/BeanstalkFarms/Basin/pull/84/commits/876da9a8f0df4118a701299397b36065c8a92530) 커밋에서 수정되었습니다.

    `calcReserve`와 `calcLpTokenSupply`는 모든 경우에 같은 방향으로 반올림해야 하므로 반올림 모드가 필요하지 않습니다:

    1. `calcReserve`의 모든 인스턴스는 올림해야 합니다:
        * 모든 출력 `amountIn` 값은 `_calcReserve(...) - reserveBefore`를 빼서 계산됩니다. Well은 `amountIn`을 과대평가하여 마진에서 더 많은 토큰이 Well로 전송되도록 보장하기를 원합니다. 따라서 `_calcReserve`는 올림해야 합니다. (`swapTo` 참조)
        * 모든 출력 `amountOut` 값은 `reserveBefore - _calcReserve(...)`를 빼서 계산됩니다. Well은 `amountOut`을 과소평가하여 마진에서 Well로부터 제거되는 토큰이 적도록 하기를 원합니다. 따라서 `_calcReserve`는 다시 올림해야 합니다. (`swapFrom`, `shift`, `removeLiquidityOneToken` 참조)
    2. `calcLpTokenSupply`의 모든 인스턴스는 내림해야 합니다:
        * 모든 `lpAmountIn` 값은 `totalSupply() - _calcLpTokenSupply(...)`를 빼서 계산됩니다. `lpAmountIn`은 과대평가되어야 하므로 `_calcLpTokenSupply`는 내림해야 합니다. (`removeLiquidityImbalanced` 참조)
        * 모든 `lpAmountOut` 값은 `_calcLpTokenSupply(...) - totalSupply()`를 빼서 계산됩니다. `lpAmountOut` 값은 과소평가되어야 하므로 `_calcLpTokenSupply`는 내림해야 합니다.

    반올림 모드가 필요하지 않다는 것은 사실 해당 Well Function이 반올림 방법을 결정하는 데 달려 있습니다. 위의 반올림 방법은 제안일 뿐이며 각 Well Function은 원하는 대로 반올림할 수 있습니다.

2. [H-03]에 대한 수정 조치로 LP 토큰 제거 시 각 금액 중 얼마를 제거할지 결정하기 위해 `IWellFunction`에 `calcLPTokenUnderlying` 함수를 추가했습니다. 결과적으로 Well 대신 Well Function 자체가 이제 LP가 제거될 때 제거할 토큰 수를 결정합니다. 따라서 Well Function은 `removeLiquidity` 계산을 처리하는 방법을 자유롭게 정의할 수 있으며 자체 정의된 불변성을 깨지 않도록 보장할 수 있습니다.

    [여기](https://github.com/BeanstalkFarms/Basin/commit/39ea396b0e78762cbeeee4e3fe8e343c2e114259)에 `calcLPTokenUnderlying`의 유효한 구현이 Well의 불변성을 절대 깨지 않아야 함을 나타내는 문서가 추가되었습니다.

    또한 `test_removeLiquidity_fuzz` 테스트를 업데이트하여 `removeLiquidity` 호출 후 불변성이 유지되는지 테스트하는 확인 절차를 [여기](https://github.com/BeanstalkFarms/Basin/commit/f5d14eba5c9c7a02ed15eba675ed869916ea469d)에 포함했습니다.

    사용자가 Well과 상호 작용하기 전에 Well에 유효한 Well Function이 있는지 확인해야 하고 Well을 사용하는 프로토콜이 검증된 Well Function 목록을 유지해야 한다는 점을 감안할 때, 검사를 추가하는 것은 유효하지 않은 Well Function이 검증된 경우에 대비한 추가 예방 조치일 뿐입니다.

    `removeLiquidity`에 다음 검사를 추가하면:

    ```solidity
    require(lpTokenSupply - lpAmountIn <= _calcLpTokenSupply(wellFunction(), reserves), "Well Function Invalid");
    ```

    가스 비용이 3,940 증가합니다. 이 검사가 추가 예방 조치일 뿐이라는 점을 감안할 때, 추가 가스 비용을 들여 추가할 가치는 없어 보입니다. 따라서 검사는 추가되지 않았습니다.

3. Well이 유지하는 불변성은 `_calcLpTokenSupply(...) >= totalSupply()`입니다. 즉, Well은 마진에서 더 적은 Well LP 토큰을 발행합니다. 이는 Well LP 토큰이 최소한 기본 가치를 유지하고 희석되지 않도록 하기 위함입니다.

    이를 검증하는 데 도움을 주기 위해 `checkInvariant` 함수가 `TestHelper`에 추가되었으며, 위의 불변성이 유지되지 않으면 되돌려집니다(revert). `checkInvariant` 함수는 `swapFrom`, `swapTo`, `removeLiquidity`, `removeLiquidityImbalanced`, `removeLiquidityOneToken`, `addLiquidity`를 포함한 대다수의 테스트에 추가되었습니다. [876da9a](https://github.com/BeanstalkFarms/Basin/pull/84/commits/876da9a8f0df4118a701299397b36065c8a92530) 커밋에서 추가되었습니다.

**Cyfrin:** 인정함. 변경 사항이 원래 문제를 완화함을 확인했습니다.


### 각 Well은 reserve가 0인 상태에서 `update` 호출이 이루어지지 않도록 보장할 책임이 있음 (Each Well is responsible for ensuring that an `update` call cannot be made with a reserve of 0)

**설명:** `GeoEmaAndCumSmaPump`의 현재 구현은 파일 시작 부분의 주석에서 언급된 대로 각 Well이 0이 아닌 reserve로 `update()`를 호출할 것이라고 가정합니다:

```solidity
/**
 * @title GeoEmaAndCumSmaPump
 * @author Publius
 * @notice Stores a geometric EMA and cumulative geometric SMA for each reserve.
 * @dev A Pump designed for use in Beanstalk with 2 tokens.
 *
 * This Pump has 3 main features:
 *  1. Multi-block MEV resistence reserves
 *  2. MEV-resistant Geometric EMA intended for instantaneous reserve queries
 *  3. MEV-resistant Cumulative Geometric intended for SMA reserve queries
 *
 * Note: If an `update` call is made with a reserve of 0, the Geometric mean oracles will be set to 0.
 * Each Well is responsible for ensuring that an `update` call cannot be made with a reserve of 0.
 */
 ```

그러나 `Well`에는 유효한 reserve 값으로 펌프 업데이트를 강제하는 실제 요구 사항이 없습니다. `GeoEmaAndCumSmaPump`는 기하 평균 문제를 방지하기 위해 값을 최소 1로 제한하지만, TWA 값이 Well의 reserve를 진정으로 대표하지 않는다는 점을 감안할 때, `ConstantProduct2` Well이 유효한 트랜잭션을 통해 두 토큰 중 하나에 대해 0의 reserve를 가질 수 있음에도 불구하고, 이 경우 revert하는 것보다 더 나쁘다고 생각합니다.

```solidity
GeoEmaAndCumSmaPump.sol
103:         for (uint i; i < length; ++i) {
104:             // Use a minimum of 1 for reserve. Geometric means will be set to 0 if a reserve is 0.
105:             b.lastReserves[i] =
106:                 _capReserve(b.lastReserves[i], (reserves[i] > 0 ? reserves[i] : 1).fromUIntToLog2(), blocksPassed);
107:             b.emaReserves[i] = b.lastReserves[i].mul((ABDKMathQuad.ONE.sub(aN))).add(b.emaReserves[i].mul(aN));
108:             b.cumulativeReserves[i] = b.cumulativeReserves[i].add(b.lastReserves[i].mul(deltaTimestampBytes));
109:         }
```

**파급력:** reserve 값이 0인 상태에서 펌프를 업데이트하면 가격 오라클에 사용될 가능성이 있는 중요한 상태가 왜곡될 수 있습니다. 유효한 트랜잭션을 통해 이 문제를 악용할 수 있다는 점을 고려하여 심각도를 **높음(HIGH)**으로 평가합니다. 공격자가 이 취약점을 악용하여 가격 오라클을 조작할 수 있다는 점에 유의하는 것이 중요합니다.

**개념 증명 (Proof of Concept):** 아래 테스트는 유효한 트랜잭션을 통해 reserve가 0이 될 수 있으며 펌프 업데이트가 되돌려지지 않음을 보여줍니다.

```solidity
function testUpdateCalledWithZero() public {
    address msgSender = 0x83a740c22a319FBEe5F2FaD0E8Cd0053dC711a1A;
    changePrank(msgSender);
    IERC20[] memory mockTokens = well.tokens();

    // add liquidity 1 on each side
    uint amount = 1;
    MockToken(address(mockTokens[0])).mint(msgSender, 1);
    MockToken(address(mockTokens[1])).mint(msgSender, 1);
    MockToken(address(mockTokens[0])).approve(address(well), amount);
    MockToken(address(mockTokens[1])).approve(address(well), amount);
    uint[] memory tokenAmountsIn = new uint[](2);
    tokenAmountsIn[0] = amount;
    tokenAmountsIn[1] = amount;
    uint minLpAmountOut = well.getAddLiquidityOut(tokenAmountsIn);
    well.addLiquidity(
        tokenAmountsIn,
        minLpAmountOut,
        msgSender,
        block.timestamp
    );

    // swaFromFeeOnTransfer from token1 to token0
    msgSender = 0xfFfFFffFffffFFffFffFFFFFFfFFFfFfFFfFfFfD;
    changePrank(msgSender);
    amount = 79_228_162_514_264_337_593_543_950_334;
    MockToken(address(mockTokens[1])).mint(msgSender, amount);
    MockToken(address(mockTokens[1])).approve(address(well), amount);
    uint minAmountOut = well.getSwapOut(
        mockTokens[1],
        mockTokens[0],
        amount
    );

    well.swapFromFeeOnTransfer(
        mockTokens[1],
        mockTokens[0],
        amount,
        minAmountOut,
        msgSender,
        block.timestamp
    );
    increaseTime(120);

    // remove liquidity one token
    msgSender = address(this);
    changePrank(msgSender);
    amount = 999_999_999_999_999_999_999_999_999;
    uint minTokenAmountOut = well.getRemoveLiquidityOneTokenOut(
        amount,
        mockTokens[1]
    );
    well.removeLiquidityOneToken(
        amount,
        mockTokens[1],
        minTokenAmountOut,
        msgSender,
        block.timestamp
    );

    msgSender = address(12_345_678);
    changePrank(msgSender);

    vm.warp(block.timestamp + 1);
    amount = 1;
    MockToken(address(mockTokens[0])).mint(msgSender, amount);
    MockToken(address(mockTokens[0])).approve(address(well), amount);
    uint amountOut = well.getSwapOut(mockTokens[0], mockTokens[1], amount);

    uint[] memory reserves = well.getReserves();
    assertEq(reserves[1], 0);

    // we are calling `_update` with reserves of 0, this should fail
    well.swapFrom(
        mockTokens[0],
        mockTokens[1],
        amount,
        amountOut,
        msgSender,
        block.timestamp
    );
}
```

**권장 완화 방안:** reserve 값이 0인 상태에서 호출되면 펌프 업데이트를 되돌리세요(revert).

**Beanstalk:** 권장된 수정 조치를 구현하지 않기로 결정했습니다. 이는 여러 요인 때문입니다. 먼저, 펌프 구조에 두 가지 변경 사항이 있었습니다:
1) Pump 실패 시 Well이 되돌려지지 않도록 했습니다 ([2459058](https://github.com/BeanstalkFarms/Basin/commit/2459058979a1aeb5e146adf399c68358d2857dd6) 커밋). 이는 Pump 실패로부터 Well을 보호하기 위해 구현되었습니다. 이 변경이 없으면 어떤 이유로 Pump가 고장날 경우 모든 Well 작업이 실패하므로 유동성이 영구적으로 잠길 수 있습니다.
2) reserve 중 하나라도 0이면 Pump를 초기화하지 않습니다. 이는 EMA와 마지막 잔액이 0으로 초기화되는 것을 방지하기 위함입니다. 대신 Pump는 반환되고 초기화하기 위해 0이 아닌 잔액을 받을 때까지 기다립니다.

1)로 인해 권장된 수정 조치가 구현된 경우 잔액이 0으로 전달되면 Well은 계속 작동하지만 Pump는 업데이트되지 않습니다. Well의 잔액이 1e6이고, 한 시간 동안 0으로 설정되었다가 다시 1e6으로 설정된다고 가정해 봅시다. Well이 0 잔액으로 Pump를 업데이트하려고 할 때 Pump가 실패했기 때문에 Pump는 전혀 업데이트되지 않았습니다. 따라서 Pump는 잔액이 내내 1e6이었고 0으로 설정되지 않았다고 가정하게 되며, 이는 Pump가 과거 잔액의 정확한 척도가 아님을 의미하고 조작될 수 있습니다(Well Function이 0 잔액을 사용할 수 있다고 가정할 때).

또한 잔액 1과 0의 차이는 꽤 작습니다. 첫째, 어느 경우에도 Well은 가격 발견의 신뢰할 수 있는 소스 역할을 하지 않습니다. Well에 ERC-20 토큰의 마이크로 단위가 1개만 있는 경우, ERC-20 토큰 1 마이크로 단위의 가치는 극히 작으므로 해당 자산을 더 이상 판매하려는 입찰은 본질적으로 없습니다. 이러한 이유로 Pump를 사용하는 모든 프로토콜은 Well에 두 ERC-20 토큰의 충분한 잔액이 있는지 확인하여 정확한 가격 발견 소스인지 확인해야 합니다. 따라서 Pump가 0의 잔액을 섭취할 때 1의 잔액을 사용하는 것의 결과는 미미할 것입니다.

**Cyfrin:** 인정함.


### `removeLiquidity` 로직이 ConstantProduct 이외의 일반화된 Well 함수에 대해 올바르지 않음 (`removeLiquidity` logic is not correct for generalized Well functions other than ConstantProduct)

**설명:** 이 프로토콜은 다양한 Well 함수를 사용할 수 있는 상수 함수 AMM 유동성 풀을 위한 일반화된 프레임워크를 제공하는 것을 목표로 합니다. 현재는 상수 곱(constant-product) 유형의 Well 함수만 정의되어 있지만, 더 일반적인 Well 함수도 지원할 의도가 있는 것으로 이해하고 있습니다.

현재의 `Well::removeLiquidity` 및 `Well::getRemoveLiquidityOut` 구현은 인출할 LP 토큰 금액에서 출력 토큰 금액을 얻을 때 선형성(linearity)을 가정합니다. 이는 아래에서 볼 수 있듯이 상수 곱 유형의 경우 잘 성립합니다. `ConstantProduct2`에 대해 LP 토큰의 총 공급량을 $L$, 두 토큰의 reserve 값을 $x, y$라고 하고 불변성을 $L^2=4xy$라고 가정해 봅시다.
$l$만큼의 유동성을 제거할 때 출력 금액은 $\Delta x=\frac{l}{L}x, \Delta y=\frac{l}{L}y$로 계산됩니다. 인출 후에도 불변성이 유지됨을 확인하는 것은 간단합니다. 즉, $(L-l)^2=(x-\Delta x)(y-\Delta y)$입니다.

그러나 일반적으로 이러한 종류의 _선형성_이 유지된다는 보장은 없습니다.

최근 일부 새로운 프로토콜에 의해 비선형(2차) 함수 AMM이 도입되었습니다([Numoen](https://numoen.gitbook.io/numoen/) 참조). 이러한 종류의 Well 함수를 사용하는 경우 현재의 `tokenAmountsOut` 계산은 Well의 불변성을 깨뜨릴 것입니다.

참고로 Numoen 프로토콜은 모든 트랜잭션 후 프로토콜의 불변성(상수 함수 자체)을 확인합니다.

**파급력:** 현재 `Well::removeLiquidity` 로직은 Well 함수에 대한 특정 조건(어떤 의미에서의 선형성)을 가정합니다. 이는 원래 목적과 달리 프로토콜의 일반화를 제한합니다. 이것이 일반적인 Well 함수에 대해 유동성 공급자의 자금 손실로 이어질 것이라는 점을 감안할 때 심각도를 **높음(HIGH)**으로 평가합니다.

**개념 증명 (Proof of Concept):** Numoen에서 사용하는 2차 Well 함수로 테스트 케이스를 작성했습니다.

```solidity
// QuadraticWell.sol

/**
 * SPDX-License-Identifier: MIT
 **/

pragma solidity ^0.8.17;

import "src/interfaces/IWellFunction.sol";
import "src/libraries/LibMath.sol";

contract QuadraticWell is IWellFunction {
    using LibMath for uint;

    uint constant PRECISION = 1e18; //@audit-info assume 1:1 upperbound for this well
    uint constant PRICE_BOUND = 1e18;

    /// @dev s = b_0 - (p_1^2 - b_1/2)^2
    function calcLpTokenSupply(
        uint[] calldata reserves,
        bytes calldata
    ) external override pure returns (uint lpTokenSupply) {
        uint delta = PRICE_BOUND - reserves[1] / 2;
        lpTokenSupply = reserves[0] - delta*delta/PRECISION ;
    }

    /// @dev b_0 = s + (p_1^2 - b_1/2)^2
    /// @dev b_1 = (p_1^2 - (b_0 - s)^(1/2))*2
    function calcReserve(
        uint[] calldata reserves,
        uint j,
        uint lpTokenSupply,
        bytes calldata
    ) external override pure returns (uint reserve) {

        if(j == 0)
        {
            uint delta = PRICE_BOUND*PRICE_BOUND - PRECISION*reserves[1]/2;
            return lpTokenSupply + delta*delta /PRECISION/PRECISION/PRECISION;
        }
        else {
            uint delta = (reserves[0] - lpTokenSupply)*PRECISION;
            return (PRICE_BOUND*PRICE_BOUND - delta.sqrt()*PRECISION)*2/PRECISION;
        }
    }

    function name() external override pure returns (string memory) {
        return "QuadraticWell";
    }

    function symbol() external override pure returns (string memory) {
        return "QW";
    }
}

// NOTE: Put in Exploit.t.sol
function test_exploitQuadraticWellAddRemoveLiquidity() public {
    MockQuadraticWell quadraticWell = new MockQuadraticWell();
    Call memory _wellFunction = Call(address(quadraticWell), "");
    Well well2 = Well(auger.bore("Well2", "WELL2", tokens, _wellFunction, pumps));

    approveMaxTokens(user, address(well2));
    uint[] memory amounts = new uint[](tokens.length);
    changePrank(user);

    // initial status 1:1
    amounts[0] = 1e18;
    amounts[1] = 1e18;
    well2.addLiquidity(amounts, 0, user); // state: [1 ether, 1 ether, 0.75 ether]

    Balances memory userBalances1 = getBalances(user, well2);
    uint[] memory userBalances = new uint[](3);
    userBalances[0] = userBalances1.tokens[0];
    userBalances[1] = userBalances1.tokens[1];
    userBalances[2] = userBalances1.lp;
    Balances memory wellBalances1 = getBalances(address(well2), well2);
    uint[] memory wellBalances = new uint[](3);
    wellBalances[0] = wellBalances1.tokens[0];
    wellBalances[1] = wellBalances1.tokens[1];
    wellBalances[2] = wellBalances1.lpSupply;
    amounts[0] = wellBalances[0];
    amounts[1] = wellBalances[1];

    emit log_named_array("userBalances1", userBalances);
    emit log_named_array("wellBalances1", wellBalances);
    emit log_named_int("invariant", quadraticWell.wellInvariant(wellBalances[2], amounts));

    // addLiquidity
    amounts[0] = 2e18;
    amounts[1] = 1e18;
    well2.addLiquidity(amounts, 0, user); // state: [3 ether, 2 ether, 3 ether]

    Balances memory userBalances2 = getBalances(user, well2);
    userBalances[0] = userBalances2.tokens[0];
    userBalances[1] = userBalances2.tokens[1];
    userBalances[2] = userBalances2.lp;
    Balances memory wellBalances2 = getBalances(address(well2), well2);
    wellBalances[0] = wellBalances2.tokens[0];
    wellBalances[1] = wellBalances2.tokens[1];
    wellBalances[2] = wellBalances2.lpSupply;
    amounts[0] = wellBalances[0];
    amounts[1] = wellBalances[1];

    emit log_named_array("userBalances2", userBalances);
    emit log_named_array("wellBalances2", wellBalances);
    emit log_named_int("invariant", quadraticWell.wellInvariant(wellBalances[2], amounts));

    // removeLiquidity
    amounts[0] = 0;
    amounts[1] = 0;
    well2.removeLiquidity(userBalances[2], amounts, user);

    Balances memory userBalances3 = getBalances(user, well2);
    userBalances[0] = userBalances3.tokens[0];
    userBalances[1] = userBalances3.tokens[1];
    userBalances[2] = userBalances3.lp;
    Balances memory wellBalances3 = getBalances(address(well2), well2);
    wellBalances[0] = wellBalances3.tokens[0];
    wellBalances[1] = wellBalances3.tokens[1];
    wellBalances[2] = wellBalances3.lpSupply;
    amounts[0] = wellBalances[0];
    amounts[1] = wellBalances[1];

    emit log_named_array("userBalances3", userBalances);
    emit log_named_array("wellBalances3", wellBalances);
    emit log_named_int("invariant", quadraticWell.wellInvariant(wellBalances[2], amounts)); // @audit-info well's invariant is broken via normal removeLiquidity
}
```

출력은 아래와 같습니다. 트랜잭션 후 Well의 `invariant`를 계산했는데, 0으로 유지되어야 하지만 유동성을 제거한 후 깨졌습니다. 프로토콜이 Well 함수를 사용하여 결과 유동성 토큰 공급량을 명시적으로 계산하기 때문에 유동성을 추가할 때는 불변성이 0으로 유지되었습니다. 그러나 유동성을 제거할 때는 Well 함수를 사용하지 않고 고정된 방식으로 출력 금액을 계산하여 불변성이 깨집니다.

```
forge test -vv --match-test test_exploitQuadraticWellAddRemoveLiquidity

[PASS] test_exploitQuadraticWellAddRemoveLiquidity() (gas: 4462244)
Logs:
  userBalances1: [999000000000000000000, 999000000000000000000, 750000000000000000]
  wellBalances1: [1000000000000000000, 1000000000000000000, 750000000000000000]
  invariant: 0
  userBalances2: [997000000000000000000, 998000000000000000000, 3000000000000000000]
  wellBalances2: [3000000000000000000, 2000000000000000000, 3000000000000000000]
  invariant: 0
  userBalances3: [1000000000000000000000, 1000000000000000000000, 0]
  wellBalances3: [0, 0, 0]
  invariant: 1000000000000000000

Test result: ok. 1 passed; 0 failed; finished in 5.14ms
```

**권장 완화 방안:** `IWellFunction` 인터페이스에 추가 함수를 추가하지 않고는 모든 종류의 Well 함수를 커버하는 것이 불가능하다고 생각합니다. `IWellFunction` 인터페이스에 `function calcWithdrawFromLp(uint lpTokenToBurn) returns (uint reserve)` 형태의 새로운 함수를 추가할 것을 권장합니다. 출력 토큰 금액은 새로 추가된 함수를 사용하여 계산할 수 있습니다.

**Beanstalk:** `calcLPTokenUnderlying` 함수를 `IWellFunction`에 추가했습니다. 이 함수는 주어진 LP 토큰 금액에 해당하는 각 reserve 토큰의 양을 반환합니다. 사용자가 `removeLiquidity` 함수를 사용하여 유동성을 제거할 때 사용자에게 보낼 기본 토큰 수를 결정하는 데 사용됩니다. [5271e9a](https://github.com/BeanstalkFarms/Basin/pull/57/commits/5271e9a454d1dd04848c3a7ce85f7d735a5858a0) 커밋에서 수정되었습니다.

**Cyfrin:** 인정함.


### 읽기 전용 재진입 (Read-only reentrancy)

**설명:** 현재 구현은 읽기 전용 재진입에 취약하며, 특히 [Wells::removeLiquidity](https://github.com/BeanstalkFarms/Wells/blob/e5441fc78f0fd4b77a898812d0fd22cb43a0af55/src/Well.sol#L440)에서 그렇습니다.
구현은 토큰을 전송한 후 새로운 reserve 값을 설정하므로 [Checks-Effects-Interactions (CEI) 패턴](https://fravoll.github.io/solidity-patterns/checks_effects_interactions.html)을 엄격하게 따르지 않습니다. 이는 `nonReentrant` 수정자로 인해 프로토콜 자체에 즉각적인 위험은 아니지만, 여전히 [읽기 전용 재진입](https://chainsecurity.com/curve-lp-oracle-manipulation-post-mortem/)에 취약합니다.

악의적인 공격자와 의심하지 않는 생태계 참가자는 ERC-777 토큰(제어권을 가져갈 수 있는 콜백이 있음)으로 Well을 배포하고 이 취약점을 악용할 수 있습니다. 이는 Well이 펌프에 의해 정의된 가격 함수로 확장될 예정이므로 중요한 취약점으로 이어질 것입니다. 이러한 온체인 오라클을 통합하는 타사 프로토콜이 위험에 처할 것입니다.

펌프는 토큰 전송 전에 업데이트되지만, reserve는 그 후에만 설정됩니다. 따라서 `IWell(well).getReserves()`가 호출되지만 reserve가 올바르게 업데이트되지 않은 경우 재진입 읽기 전용 호출에서 펌프 함수가 올바르지 않을 가능성이 높습니다. `GeoEmaAndCumSmaPump`의 구현은 취약하지 않은 것으로 보이지만, 각 펌프가 시간 경과에 따른 Well의 reserve 기록 방식을 선택할 수 있다는 점을 감안할 때 이는 여전히 가능한 공격 벡터로 남아 있습니다.

**파급력:** 프로토콜 자체에 즉각적인 위험은 아니지만 읽기 전용 재진입은 심각한 문제로 이어질 수 있으므로 심각도를 **높음(HIGH)**으로 평가합니다.

**개념 증명 (Proof of Concept):** 기존의 읽기 전용 재진입을 보여주는 테스트 케이스를 작성했습니다.

```solidity
// MockCallbackRecipient.sol

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

import {console} from "forge-std/Test.sol";

contract MockCallbackRecipient {
    fallback() external payable {
        console.log("here");
        (bool success, bytes memory result) = msg.sender.call(abi.encodeWithSignature("getReserves()"));
        if (success) {
            uint256[] memory reserves = abi.decode(result, (uint256[]));
            console.log("read-only-reentrancy beforeTokenTransfer reserves[0]: %s", reserves[0]);
            console.log("read-only-reentrancy beforeTokenTransfer reserves[1]: %s", reserves[1]);
        }
    }
}

// NOTE: Put in Exploit.t.sol
function test_exploitReadOnlyReentrancyRemoveLiquidityCallbackToken() public {
    IERC20 callbackToken = IERC20(new MockCallbackToken("CallbackToken", "CBTKN", 18));
    MockToken(address(callbackToken)).mint(user, 1000e18);
    IERC20[] memory _tokens = new IERC20[](2);
    _tokens[0] = callbackToken;
    _tokens[1] = tokens[1];

    vm.stopPrank();
    Well well2 = Well(auger.bore("Well2", "WELL2", _tokens, wellFunction, pumps));
    approveMaxTokens(user, address(well2));

    uint[] memory amounts = new uint[](2);
    amounts[0] = 100 * 1e18;
    amounts[1] = 100 * 1e18;

    changePrank(user);
    callbackToken.approve(address(well2), type(uint).max);
    uint256 lpAmountOut = well2.addLiquidity(amounts, 0, user);

    well2.removeLiquidity(lpAmountOut, amounts, user);
}
```

출력은 아래와 같습니다.

```
forge test -vv --match-test test_exploitReadOnlyReentrancyRemoveLiquidityCallbackToken

[PASS] test_exploitReadOnlyReentrancyRemoveLiquidityCallbackToken() (gas: 5290876)
Logs:
  read-only-reentrancy beforeTokenTransfer reserves[0]: 0
  read-only-reentrancy beforeTokenTransfer reserves[1]: 0
  read-only-reentrancy afterTokenTransfer reserves[0]: 0
  read-only-reentrancy afterTokenTransfer reserves[1]: 0
  read-only-reentrancy beforeTokenTransfer reserves[0]: 100000000000000000000
  read-only-reentrancy beforeTokenTransfer reserves[1]: 100000000000000000000
  read-only-reentrancy afterTokenTransfer reserves[0]: 100000000000000000000
  read-only-reentrancy afterTokenTransfer reserves[1]: 100000000000000000000

Test result: ok. 1 passed; 0 failed; finished in 3.66ms
```

**권장 완화 방안:** 외부 호출을 하기 전에 reserve를 업데이트하여 관련 함수에서 CEI 패턴을 구현하세요. 예를 들어 `Well::removeLiquidity` 함수는 아래와 같이 수정할 수 있습니다.

```solidity
function removeLiquidity(
    uint lpAmountIn,
    uint[] calldata minTokenAmountsOut,
    address recipient
) external nonReentrant returns (uint[] memory tokenAmountsOut) {
    IERC20[] memory _tokens = tokens();
    uint[] memory reserves = _updatePumps(_tokens.length);
    uint lpTokenSupply = totalSupply();

    tokenAmountsOut = new uint[](_tokens.length);
    _burn(msg.sender, lpAmountIn);

    _setReserves(reserves); // @audit CEI pattern

    for (uint i; i < _tokens.length; ++i) {
        tokenAmountsOut[i] = (lpAmountIn * reserves[i]) / lpTokenSupply;
        require(
            tokenAmountsOut[i] >= minTokenAmountsOut[i],
            "Well: slippage"
        );
        _tokens[i].safeTransfer(recipient, tokenAmountsOut[i]);
        reserves[i] = reserves[i] - tokenAmountsOut[i];
    }

    emit RemoveLiquidity(lpAmountIn, tokenAmountsOut);
}
```

**Beanstalk:** 재진입 가드에 진입한 경우 되돌리는 검사를 `getReserves()` 함수에 추가했습니다. 이를 통해 누구도 Well에서 함수를 실행하는 동안 `getReserves()`를 호출할 수 없습니다. [fcbf04a](https://github.com/BeanstalkFarms/Basin/pull/85/commits/fcbf04a99b00807891fb2a9791ba18ed425479ab) 커밋에서 수정되었습니다.

**Cyfrin:** 인정함.

\clearpage
## 중간 중요도 (Medium Risk)


### Well의 토큰 고유성을 보장해야 함 (Should ensure uniqueness of the tokens of Wells)

**설명:** 현재 구현에서는 Well의 토큰에서 고유성을 강제하지 않습니다.
누구나 악성 `Well` 구현으로 `Aquifer::boreWell()`을 호출하여 피해자를 위한 함정을 설정할 수 있습니다.
프로토콜 팀과의 소통을 통해 모든 Well은 _무죄가 입증될 때까지 유죄(guilty until proven innocent)_ 로 간주된다는 점을 이해했습니다.
그러나 여전히 `Aquifier::boreWell`에서 Well을 검증하고 악성 Well의 배포를 방지하는 것이 바람직합니다.
또한 배포 후 `tokens()` 변경을 금지하는 것도 강력히 권장됩니다.

Well에 중복 토큰이 있는 경우 아래와 같은 공격 경로가 존재하며 더 많을 수 있습니다.

**파급력:** 사용자에게 악성 Well에 대해 명시적으로 경고하고 유효하지 않은 Well과 상호 작용할 가능성이 낮다고 가정하지만 심각도를 **중간(MEDIUM)**으로 평가합니다.

**개념 증명 (Proof of Concept):** tokens[0]=tokens[1]이라고 가정해 봅시다.
정직한 LP가 `addLiquidity([1 ether,1 ether], 200 ether, address)`를 호출하면 reserve는 (1 ether, 1 ether)가 됩니다. 그러나 누구나 `skim()`을 호출하여 `1 ether`를 꺼낼 수 있습니다. 이는 `skimAmounts`가 `balanceOf()`에 의존하기 때문이며, 첫 번째 루프에서 `2 ether`를 반환할 것입니다.

```solidity
function skim(address recipient) external nonReentrant returns (uint[] memory skimAmounts) {
    IERC20[] memory _tokens = tokens();
    uint[] memory reserves = _getReserves(_tokens.length);
    skimAmounts = new uint[](_tokens.length);
    for (uint i; i < _tokens.length; ++i) {
        skimAmounts[i] = _tokens[i].balanceOf(address(this)) - reserves[i];
        if (skimAmounts[i] > 0) {
            _tokens[i].safeTransfer(recipient, skimAmounts[i]);
        }
    }
}
```

**권장 완화 방안:**
- 배포된 후에는 Well 토큰을 변경할 수 없도록 하세요.
- `boreWell()` 동안 토큰의 고유성을 요구하세요.

**Beanstalk:** [f10e05a](https://github.com/BeanstalkFarms/Basin/pull/81/commits/f10e05a32f60ec288ef6064e665cae797b800b39) 커밋에서 수정되었습니다.

**Cyfrin:** 인정함.


### `LibLastReserveBytes::storeLastReserves`에 너무 큰 reserve에 대한 확인이 없음 (`LibLastReserveBytes::storeLastReserves` has no check for reserves being too large)

**설명:** 모든 유동성 이벤트 및 스왑 후에 `IPump::update`()`가 호출됩니다.
펌프를 업데이트하기 위해 `LibLastReserveBytes::storeLastReserves` 함수가 사용됩니다. 이는 reserve 데이터를 스토리지의 `bytes32` 슬롯에 패킹합니다.
그런 다음 슬롯은 다음 구성 요소로 나뉩니다:
- reserve 배열 길이를 위한 1 바이트
- `timestamp`를 위한 5 바이트
- 각 reserve 잔액을 위한 16 바이트

이것은 총 22바이트를 추가하지만, 함수는 또한 두 번째 reserve 잔액을 `bytes32` 객체에 패킹하려고 시도합니다.
즉, `bytes32`에는 총 38바이트가 필요합니다:

`1(길이) + 5(타임스탬프) + 16(reserve 잔액 1) + 16(reserve 잔액 2) = 38 바이트`

이 모든 데이터를 `bytes32`에 맞추기 위해 함수는 아래와 같이 시프트를 사용하여 reserve 잔액의 마지막 몇 바이트를 잘라냅니다.

```solidity
src\libraries\LibLastReserveBytes.sol
21:    uint8 n = uint8(reserves.length);
22:    if (n == 1) {
23:        assembly {
24:            sstore(slot, or(or(shl(208, lastTimestamp), shl(248, n)), shl(104, shr(152, mload(add(reserves, 32))))))
25:        }
26:        return;
27:    }
28:    assembly {
29:        sstore(
30:            slot,
31:            or(
32:                or(shl(208, lastTimestamp), shl(248, n)),
33:                or(shl(104, shr(152, mload(add(reserves, 32)))), shr(152, mload(add(reserves, 64))))
34:            )
35:        )
36:        // slot := add(slot, 32)
37:    }
```

따라서 저장되는 금액이 너무 크면 실제 저장된 값은 저장될 것으로 예상했던 값과 달라집니다.

반면, `LibBytes.sol`에는 확인이 있는 것으로 보입니다:
```solidity
require(reserves[0] <= type(uint128).max, "ByteStorage: too large");
```
`_setReserves` 함수는 Well의 모든 reserve 업데이트 후에 이 라이브러리를 호출합니다.
따라서 실제로 현재 구현된 Well 및 Pump에서는 이 확인으로 인해 revert가 발생합니다.

그러나 이 확인 없이 구현된 Well은 추가적으로 펌프가 reserve 데이터를 잘라내도록 유발할 수 있으며, 이는 가격이 올바르지 않음을 의미합니다.

**파급력:** 사용자에게 악성 Well에 대해 명시적으로 경고하고 유효하지 않은 Well과 상호 작용할 가능성이 낮다고 가정하지만 심각도를 **중간(MEDIUM)**으로 평가합니다.

**개념 증명 (Proof of Concept):**
```solidity
function testStoreAndReadTwo() public {
    uint40 lastTimeStamp = 12345363;
    bytes16[] memory reserves = new bytes16[](2);
    reserves[0] = 0xffffffffffffffffffffffffffffffff; // This is too big!
    reserves[1] = 0x11111111111111111111111100000000;
    RESERVES_STORAGE_SLOT.storeLastReserves(lastTimeStamp, reserves);
    (
        uint8 n,
        uint40 _lastTimeStamp,
        bytes16[] memory _reserves
    ) = RESERVES_STORAGE_SLOT.readLastReserves();
    assertEq(2, n);
    assertEq(lastTimeStamp, _lastTimeStamp);
    assertEq(reserves[0], _reserves[0]); // This will fail
    assertEq(reserves[1], _reserves[1]);
    assertEq(reserves.length, _reserves.length);
}
```

**권장 완화 방안:** `LibLastReseveBytes`에서 reserve의 크기에 대한 확인을 추가할 것을 권장합니다.

또한 `LibLastReseveBytes`에 주석을 추가하여 사용자에게 시스템의 불변성에 대해 알리고 reserve의 최대 크기가 `uint256`이 아닌 `bytes16`의 최대 크기와 같아야 함을 알리는 것이 좋습니다.

**Beanstalk:** Pump가 2개의 마지막 reserve 값을 `bytes16` 4배 정밀도 부동 소수점 값과 `uint40` 타임스탬프 형태로 동일한 슬롯에 패킹하기 때문에 마지막 reserve 값에 정밀도 손실이 있습니다. 각 마지막 reserve 값은 예상되는 ~34자리 대신 ~27자리의 정밀도만 가집니다.

마지막 reserve가 reserve 업데이트의 상한을 결정하는 데만 사용되고 보존되는 27자리가 가장 중요한 자리수라는 점을 감안할 때 이로 인한 영향은 미미합니다. 상한은 조작의 영향을 방지하기 위해서만 사용되며 임의로 설정됩니다. 또한 외부 프로토콜에 의해 평가되지 않습니다. 마지막으로 27자리 정밀도는 여전히 매우 중요합니다. 1의 오류가 발생하려면 풀에 약 1,000,000,000,000,000,000,000,000,000개의 토큰이 있어야 합니다.

**Cyfrin:** 인정함.

\clearpage
## 낮은 중요도 (Low Risk)


### 업데이트가 1번만 발생한 경우 TWAP가 부정확함 (TWAP is incorrect when only 1 update has occurred)

**설명:** 펌프에 대한 업데이트가 단 한 번만 있었다면 `GeoEmaAndCumSmaPump::readTwaReserves`는 잘못된 값을 반환합니다.

**파급력:** 이것은 펌프가 아직 램프업 중일 때 하나의 오라클 업데이트에만 영향을 미치므로 심각도를 **낮음(LOW)**으로 평가합니다.

**개념 증명 (Proof of Concept):**
```solidity
// NOTE: place in `Pump.Update.t.sol`
function testTWAReservesIsWrong() public {
    increaseTime(12); // increase 12 seconds

    bytes memory startCumulativeReserves = pump.readCumulativeReserves(
        address(mWell)
    );
    uint256 lastTimestamp = block.timestamp;
    increaseTime(120); // increase 120 seconds aka 10 blocks

    (uint[] memory twaReserves, ) = pump.readTwaReserves(
        address(mWell),
        startCumulativeReserves,
        lastTimestamp
    );

    assertApproxEqAbs(twaReserves[0], 1e6, 1);
    assertApproxEqAbs(twaReserves[1], 2e6, 1);

    vm.prank(user);
    // gonna double it
    // so our TWAP should now be ~1.5 & 3
    b[0] = 2e6;
    b[1] = 4e6;
    mWell.update(address(pump), b, new bytes(0));

    increaseTime(120); // increase 12 seconds aka 10 blocks
    (twaReserves, ) = pump.readTwaReserves(
        address(mWell),
        startCumulativeReserves,
        lastTimestamp
    );

    // the reserves of b[0]:
    // - 1e6 * 10 blocks
    // - 2e6 * 10 blocks
    // average reserves over 20 blocks:
    // - 15e5
    // assertApproxEqAbs(twaReserves[0], 1.5e5, 1);
    // instead, we get:
    // b[0] = 2070529

    // the reserves of b[1]:
    // - 2e6 * 10 blocks
    // - 4e6 * 10 blocks
    // average reserves over 20 blocks:
    // - 3e6
    assertApproxEqAbs(twaReserves[1], 3e6, 1);
    // instead, we get:
    // b[1] = 4141059
}
```

**권장 완화 방안:** 업데이트가 2회 미만인 경우 펌프에서 읽는 것을 허용하지 마세요.

**Beanstalk:** 이는 `MockReserveWell`의 버그 때문입니다. 이전에는 Well이 새로운 reserve로 Pump 업데이트 호출을 수행했습니다. 그러나 Well은 이전 reserve로 Pump를 업데이트해야 합니다.

`MockReserveWell`의 이 문제는 [여기](https://github.com/BeanstalkFarms/Basin/commit/6f55fb4835dba6f7b4a69c22687c83039cac2af1#diff-e982575c648d2eaa2de1e69b9ea2f4e91b7ffdeb9455ddacd86c70239972dc84)에서 수정되었습니다.

지금 테스트를 실행하면 `twaReserve[0] = 1414213`임을 알 수 있습니다. `sqrt(1e6 * 2e6) = 1414213`이므로 예상된 결과입니다. 또한 `twaReserve[1] = 2828427`입니다. `sqrt(2e6 * 4e6) = 2828427`이므로 이 역시 예상된 결과입니다.

**Cyfrin:** 인정함.


### `GeoEmaAndCumSmaPump::constructor`에서 `A`에 대한 유효성 검사 부족 (Lack of validation for `A` in `GeoEmaAndCumSmaPump::constructor`)

**설명:** `GeoEmaAndCumSmaPump::constructor`에서 EMA 매개변수($\alpha$) `A`는 `_A`로 초기화됩니다. 이 매개변수는 1보다 작아야 합니다. 그렇지 않으면 `GeoEmaAndCumSmaPump::update`가 언더플로우로 인해 되돌려집니다(revert).

**파급력:** 이 초기화는 배포 시 배포자에 의해 수행되므로 심각도를 **낮음(LOW)**으로 평가합니다.

**권장 완화 방안:** 초기화 값 `_A`가 `ABDKMathQuad.ONE`보다 작아야 한다고 요구하세요.

**Beanstalk:** [e834f9f](https://github.com/BeanstalkFarms/Basin/commit/e834f9f6114815fbfef1a406cb0bb773449a6bc2) 커밋에서 수정되었습니다.

**Cyfrin:** 인정함.


### LibBytes의 잘못된 sload (Incorrect sload in LibBytes)

**설명:** `LibBytes`의 `storeUint128` 함수는 주어진 슬롯에서 시작하는 uint128 `reserves`를 패킹하려고 하지만 [홀수 개의 reserve를 저장](https://github.com/BeanstalkFarms/Wells/blob/e5441fc78f0fd4b77a898812d0fd22cb43a0af55/src/libraries/LibBytes.sol#L70)하는 경우 마지막 슬롯을 덮어씁니다. 현재는 `Well::_updatePumps`의 결과를 입력으로 받는 [`Well::_setReserves`](https://github.com/BeanstalkFarms/Wells/blob/e5441fc78f0fd4b77a898812d0fd22cb43a0af55/src/Well.sol#L609)에서만 호출되며, `Well::_updatePumps` 자체는 항상 `_tokens.length`를 인수로 받습니다. 따라서 홀수 개의 토큰인 경우 오류에 관계없이 슬롯의 마지막 128비트는 액세스되지 않습니다. 그러나 토큰의 길이에 따라 항상 동작하는 것이 아니라 한 번에 가변적인 수의 reserve를 설정하는 다른 구현에서 라이브러리를 사용하는 경우가 있을 수 있으며, 이는 실수로 마지막 reserve를 0으로 덮어쓸 수 있습니다.

**파급력:** 자산이 직접적인 위험에 처하지 않으므로 심각도를 **낮음(LOW)**으로 평가합니다.

**개념 증명 (Proof of Concept):** 다음 테스트 케이스는 이 문제를 더 명확하게 보여줍니다:

```solidity
// NOTE: Add to LibBytes.t.sol
function test_exploitStoreAndRead() public {
    // Write to storage slot to demonstrate overwriting existing values
    // In this case, 420 will be stored in the lower 128 bits of the last slot
    bytes32 slot = RESERVES_STORAGE_SLOT;
    uint256 maxI = (NUM_RESERVES_MAX - 1) / 2;
    uint256 storeValue = 420;
    assembly {
        sstore(add(slot, mul(maxI, 32)), storeValue)
    }

    // Read reserves and assert the final reserve is 420
    uint[] memory reservesBefore = LibBytes.readUint128(RESERVES_STORAGE_SLOT, NUM_RESERVES_MAX);
    emit log_named_array("reservesBefore", reservesBefore);

    // Set up reserves to store, but only up to NUM_RESERVES_MAX - 1 as we have already stored a value in the last 128 bits of the last slot
    uint[] memory reserves = new uint[](NUM_RESERVES_MAX - 1);
    for (uint i = 1; i < NUM_RESERVES_MAX; i++) {
        reserves[i-1] = i;
    }

    // Log the last reserve before the store, perhaps from other implementations which don't always act on the entire reserves length
    uint256 t;
    assembly {
        t := shr(128, shl(128, sload(add(slot, mul(maxI, 32)))))
    }
    emit log_named_uint("final slot, lower 128 bits before", t);

    // Store reserves
    LibBytes.storeUint128(RESERVES_STORAGE_SLOT, reserves);

    // Re-read reserves and compare
    uint[] memory reserves2 = LibBytes.readUint128(RESERVES_STORAGE_SLOT, NUM_RESERVES_MAX);

    emit log_named_array("reserves", reserves);
    emit log_named_array("reserves2", reserves2);

    // But wait, what about the last reserve
    assembly {
        t := shr(128, shl(128, sload(add(slot, mul(maxI, 32)))))
    }

    // Turns out it was overwritten by the last store as it calculates the sload incorrectly
    emit log_named_uint("final slot, lower 128 bits after", t);
}
```

완화 전 출력:

```
reservesBefore: [0, 0, 0, 0, 0, 0, 0, 420]
final slot, lower 128 bits before: 420
reserves: [1, 2, 3, 4, 5, 6, 7]
reserves2: [1, 2, 3, 4, 5, 6, 7, 0]
final slot, lower 128 bits after: 0
```

**권장 완화 방안:** 스토리지에서 기존 값을 로드하고 하위 비트에 패킹하도록 다음 수정 사항을 구현하세요:

```solidity
	sload(add(slot, mul(maxI, 32)))
```

완화 후 출력:

```
reservesBefore: [0, 0, 0, 0, 0, 0, 0, 420]
final slot, lower 128 bits before: 420
reserves: [1, 2, 3, 4, 5, 6, 7]
reserves2: [1, 2, 3, 4, 5, 6, 7, 420]
final slot, lower 128 bits after: 420
```

**Beanstalk:** [5e61420](https://github.com/BeanstalkFarms/Basin/pull/71/commits/5e614205a49f1388c36b2a02ce38cf0df317dfde) 커밋에서 수정되었습니다.

**Cyfrin:** 인정함.

\clearpage
## 정보성 (Informational)


### 비표준 스토리지 패킹 (Non-standard storage packing)

[Solidity 문서](https://docs.soliditylang.org/en/v0.8.17/internals/layout_in_storage.html)에 따르면 패킹된 스토리지 슬롯의 첫 번째 항목은 하위 정렬(lower-order aligned)되어 저장됩니다. 그러나 `LibBytes`의 [수동 패킹](https://github.com/BeanstalkFarms/Wells/blob/e5441fc78f0fd4b77a898812d0fd22cb43a0af55/src/libraries/LibBytes.sol#L37)은 이 관례를 따르지 않습니다. 첫 번째 패킹된 값을 하위 정렬 위치에 저장하도록 `storeUint128` 함수를 수정하세요.

**Beanstalk:** [5e61420](https://github.com/BeanstalkFarms/Basin/pull/71/commits/5e614205a49f1388c36b2a02ce38cf0df317dfde) 커밋에서 수정되었습니다.

**Cyfrin:** 인정함.


### EIP-1967 두 번째 선이미지 모범 사례 (EIP-1967 second pre-image best practice)

[Well.sol::RESERVES_STORAGE_SLOT](https://github.com/BeanstalkFarms/Wells/blob/e5441fc78f0fd4b77a898812d0fd22cb43a0af55/src/Well.sol#L25)과 같이 사용자 지정 [EIP-1967](https://eips.ethereum.org/EIPS/eip-1967) 스토리지 슬롯을 계산할 때, 두 번째 선이미지(second pre-image) 공격의 가능성을 더욱 줄이기 위해 해시된 값에 `-1` 오프셋을 추가하는 것이 [모범 사례](https://ethereum-magicians.org/t/eip-1967-standard-proxy-storage-slots/3185?u=frangio)입니다.

**Beanstalk:** [dc0fe45](https://github.com/BeanstalkFarms/Basin/commit/dc0fe45ae0b85bc230093acc8b35939cbc8f1844) 커밋에서 수정되었습니다.

**Cyfrin:** 인정함.


### 실험적 ABIEncoderV2 pragma 제거 (Remove experimental ABIEncoderV2 pragma)

ABIEncoderV2는 Solidity 0.8에서 기본적으로 활성화되어 있으므로 [두](https://github.com/BeanstalkFarms/Wells/blob/e5441fc78f0fd4b77a898812d0fd22cb43a0af55/src/interfaces/IWellFunction.sol#L4) [인스턴스](https://github.com/BeanstalkFarms/Wells/blob/e5441fc78f0fd4b77a898812d0fd22cb43a0af55/src/interfaces/pumps/IPump.sol#L4)를 제거할 수 있습니다.

**Beanstalk:** ABIEncoderV2 pragma의 모든 인스턴스가 제거되었습니다. [3416a2d](https://github.com/BeanstalkFarms/Basin/commit/3416a2d33876102158ab98a03f409903fe504278) 커밋에서 수정되었습니다.

**Cyfrin:** 인정함.


### 인라인 어셈블리에서 십진수/16진수 표기법의 일관성 없는 사용 (Inconsistent use of decimal/hex notation in inline assembly)

가독성을 높이고 인라인 어셈블리로 작업할 때 오류를 방지하려면 정수 상수에는 십진수 표기법을 사용하고 메모리 오프셋에는 16진수 표기법을 사용해야 합니다.

**Beanstalk:** Basin을 위해 작성된 모든 새 코드는 십진수 표기법을 사용합니다. 가독성이 더 좋기 때문에 모든 새 코드에 십진수 표기법이 선택되었습니다.

일부 외부 라이브러리는 16진수 표기법을 사용합니다(예: `ABDKMathQuad.sol`). 복잡성을 방지하기 위해 이러한 라이브러리를 수정하는 대신 그대로 두는 것이 가장 좋다고 결정했습니다.

**Cyfrin:** 인정함.


### 사용되지 않는 import 및 오류 (Unused imports and errors)

`LibMath`에서:
- OpenZeppelin SafeMath가 가져와졌지만 사용되지 않음
- `PRBMath_MulDiv_Overflow` 오류가 선언되었지만 사용되지 않음

**Beanstalk:** [여기](https://github.com/BeanstalkFarms/Basin/pull/65)에서 수정되었습니다.

**Cyfrin:** 인정함.


### LibMath 주석의 불일치 (Inconsistency in LibMath comments)

`LibMath`의 `nthRoot` 및 `sqrt` [함수](https://github.com/BeanstalkFarms/Wells/blob/e5441fc78f0fd4b77a898812d0fd22cb43a0af55/src/libraries/LibMath.sol#L41-L138) 내에서 주석의 `x`와 코드의 `a` 사용이 일관되지 않습니다.

**Beanstalk:** [여기](https://github.com/BeanstalkFarms/Basin/pull/65)에서 수정되었습니다.

**Cyfrin:** 인정함.


### FIXME 및 TODO 주석 (FIXME and TODO comments)

해결해야 할 여러 [FIXME](https://github.com/BeanstalkFarms/Wells/blob/e5441fc78f0fd4b77a898812d0fd22cb43a0af55/src/interfaces/IWell.sol#L351) 및 [TODO](https://github.com/BeanstalkFarms/Wells/blob/e5441fc78f0fd4b77a898812d0fd22cb43a0af55/src/libraries/LibMath.sol#L33) 주석이 있습니다.

**Beanstalk:** [여기](https://github.com/BeanstalkFarms/Basin/pull/65)에서 수정되었습니다.

**Cyfrin:** 인정함.


### 올바른 NatSpec 태그 사용 (Use correct NatSpec tags)

`@dev See {IWell.fn}`의 사용은 인터페이스에서 NatSpec 문서를 상속하기 위해 `@inheritdoc IWell`로 대체되어야 합니다.

**Beanstalk:** `@dev See {IWell.fn}`의 인스턴스가 보이지 않습니다. `See: {IAquifer.fn}`의 유사한 인스턴스를 제거했습니다. natspec [문서](https://docs.soliditylang.org/en/v0.7.6/natspec-format.html#inheritance-notes)를 보면 "NatSpec이 없는 함수는 기본 함수의 문서를 자동으로 상속합니다"라고 나와 있습니다. 따라서 @inheritdoc 태그를 추가하는 것은 불필요한 것 같습니다.

**Cyfrin:** 인정함.


### `GeoEmaAndCumSmaPump`의 설명이 부족한 변수 및 함수 이름은 읽기 어려움 (Poorly descriptive variable and function names in `GeoEmaAndCumSmaPump` are difficult to read)

예를 들어 `update`에서:
* `b`는 `returnedReserves`로 이름을 바꿀 수 있습니다.
* `aN`은 `alphaN` 또는 `alphaRaisedToTheDeltaTimeStamp`로 이름을 바꿀 수 있습니다.

또한 `A`/`_A`는 `ALPHA`로, `readN`은 `readNumberOfReserves`로 이름을 바꿀 수 있습니다.

**Beanstalk:** [3bf0080](https://github.com/BeanstalkFarms/Basin/pull/78/commits/3bf0080f7597eb3dfc276ece6207f4f98dcc3db3) 커밋에서 수정되었습니다.

* `Reserves` 구조체 이름을 `PumpState`로 변경.
* `b`를 `pumpState`로 변경.
* `readN`을 `readNumberOfReserves`로 변경.
* `n`을 `numberOfReserves`로 변경.
* `_A`를 `_alpha`로 변경.
* `A`를 `ALPHA`로 변경.
* `aN`을 `alphaN`으로 변경.

**Cyfrin:** 인정함.


### 바이트 시프트가 필요한지 확인하는 TODO 제거 (Remove TODO Check if bytes shift is necessary)

`LibBytes16::readBytes16`에서 다음 줄에 `TODO`가 있습니다:

```mstore(add(reserves, 64), shl(128, sload(slot))) // TODO: Check if byte shift is necessary```

단일 슬롯에 두 개의 reserve 요소 데이터가 저장되므로 왼쪽 시프트가 실제로 필요합니다. 다음 테스트는 이것이 어떻게 다른지 보여줍니다:

```solidity
function testNeedLeftShift() public {
    uint256 reservesSize = 2;
    uint256 slotNumber = 12345;
    bytes32 slot = bytes32(slotNumber);

    bytes16[] memory leftShiftreserves = new bytes16[](reservesSize);
    bytes16[] memory noShiftreserves = new bytes16[](reservesSize);

    // store some data in the slot
    assembly {
        sstore(
            slot,
            0x0000000000000000000000000000007b00000000000000000000000000000011
        )
    }

    // left shift
    assembly {
        mstore(add(leftShiftreserves, 32), sload(slot))
        mstore(add(leftShiftreserves, 64), shl(128, sload(slot)))
    }

    // no shift
    assembly {
        mstore(add(noShiftreserves, 32), sload(slot))
        mstore(add(noShiftreserves, 64), sload(slot))
    }
    assert(noShiftreserves[1] != leftShiftreserves[1]);
}
```

**Beanstalk:** [여기](https://github.com/BeanstalkFarms/Basin/pull/66)에서 수정되었습니다.

**Cyfrin:** 인정함.


### 내부 함수에 밑줄 접두사 사용 (Use underscore prefix for internal functions)

`getSlotForAddress`와 같은 함수의 경우 독자가 내부 함수임을 알 수 있도록 이름을 `_getSlotForAddress`로 지정하는 것이 더 읽기 쉽습니다. 마찬가지로 스토리지 변수에는 `s_`, 불변 변수에는 `i_`를 사용하는 것이 좋습니다.

**Beanstalk:** 내부 함수에 `_` 접두사를 추가하기로 결정했지만 `s_` 또는 `i_` 접두사는 추가하지 않기로 했습니다. [86c471f](https://github.com/BeanstalkFarms/Basin/pull/76/commits/86c471f9767ccea4bfa49a34fd6344ae77e74f24) 커밋에서 수정되었습니다.

**Cyfrin:** 인정함.


### 다수 함수에 대한 테스트 커버리지 누락 (Missing test coverage for a number of functions)

`GeoEmaAndCumSmaPump::getSlotsOffset`, `GeoEmaAndCumSmaPump::getDeltaTimestamp`, `_getImmutableArgsOffset`에 대한 테스트를 추가하여 테스트 커버리지와 예상대로 작동한다는 확신을 높이는 것을 고려하세요.

**Beanstalk:** [여기](https://github.com/BeanstalkFarms/Basin/pull/67)에서 수정되었습니다.

**Cyfrin:** 인정함.


### `uint` 대신 `uint256` 사용 (Use `uint256` over `uint`)

`uint`는 `uint256`의 별칭이며 사용이 권장되지 않습니다. 별칭이 실수로 서명 문자열 내에서 사용되는 경우 선택기로 데이터를 인코딩할 때 문제가 발생할 수 있으므로 변수 크기를 명확히 해야 합니다.

**Beanstalk:** [7ca7d64](https://github.com/BeanstalkFarms/Basin/commits/7ca7d64866ee8235641c6cc4bb71df511d1f61a5) 커밋에서 수정되었습니다.

**Cyfrin:** 인정함.


### 인라인 매직 넘버 대신 상수 변수 사용 (Use constant variables in place of inline magic numbers)

숫자를 사용할 때는 상수 변수로 저장하여 숫자가 나타내는 바를 명확히 해야 합니다.

예를 들어 `Well.sol`에서 펌프의 calldata 위치는 다음과 같이 주어집니다:

```solidity
uint dataLoc = LOC_VARIABLE + numberOfTokens() * 32 + wellFunctionDataLength();
```

추가 지식 없이는 읽기 어려울 수 있으므로 다음과 같이 변수를 할당하는 것이 좋습니다:

```solidity
uint256 constant ONE_WORD = 32;
uint256 constant PACKED_ADDRESS = 20;
...
uint dataLoc = LOC_VARIABLE + numberOfTokens() * ONE_WORD + wellFunctionDataLength();
```

`248` 및 `208`과 같은 숫자가 장황한 의미를 갖도록 시프트를 수행하는 인라인 어셈블리 블록에도 동일한 권장 사항을 적용할 수 있습니다.

또한 불변 데이터/스토리지에 값을 패킹할 때(예: 펌프) 코드가 명시적으로 이를 명시하는 주석의 이점을 누릴 수 있습니다.

**Beanstalk:** `32` 및 `20` 인스턴스를 `ONE_WORD` 및 `PACKED_ADDRESS`로 변환했습니다. [여기](https://github.com/BeanstalkFarms/Basin/pull/66)에서 수정되었습니다.

**Cyfrin:** 인정함.


### NatSpec 사용 불충분 및 복잡한 코드 블록에 대한 주석 부족 (Insufficient use of NatSpec and comments on complex code blocks)

`WellDeployer::encodeAndBoreWell`과 같은 많은 저수준 함수에 NatSpec 문서가 누락되었습니다. 또한 많은 수학 집약적인 컨트랙트 및 라이브러리는 NatSpec 및 지원 주석이 있어야만 이해하기 쉽습니다.

**Beanstalk:** `encodeAndBoreWell`에 natspec을 추가했습니다. [여기](https://github.com/BeanstalkFarms/Basin/pull/65)에서 수정되었습니다.

**Cyfrin:** 인정함.


### log2 스케일과 일반 스케일 간 변환 시 큰 값의 정밀도 손실 (Precision loss on large values transformed between log2 scale and the normal scale)

`GeoEmaAndCumSmaPump.sol::_init`에서 reserve 값은 log2 스케일로 변환됩니다:

```solidity
byteReserves[i] = reserves[i].fromUIntToLog2();
```

이 변환은 다음 테스트에서 보여지는 것처럼 특히 큰 `uint256` 값에 대해 정밀도 손실을 의미합니다:

```solidity
function testUIntMaxToLog2() public {
uint x = type(uint).max;
bytes16 y = ABDKMathQuad.fromUIntToLog2(x);
console.log(x);
console.logBytes16(y);
assertEq(ABDKMathQuad.fromUInt(x).log_2(), ABDKMathQuad.fromUIntToLog2(x));
uint x_recover = ABDKMathQuad.pow_2ToUInt(y);
console.log(ABDKMathQuad.toUInt(y));
console.log(x_recover);
}
```

정밀도 손실을 방지하기 위해 reserve 값을 명시적으로 제한하는 것을 고려하세요.

**Beanstalk:** 이는 예상된 것입니다. uint256을 bytes16으로 압축하면 256비트를 128비트로 압축하므로 정밀도를 잃지 않을 수 없습니다. 예: log 연산에는 113비트의 십진수 정밀도만 있습니다. [여기](https://github.com/abdk-consulting/abdk-libraries-solidity/blob/master/ABDKMathQuad.md#ieee-754-quadruple-precision-floating-point-numbers)를 참조하세요.

**Cyfrin:** 인정함.


### 외부 상호 작용 전에 이벤트 방출 (Emit events prior to external interactions)

[Checks Effects Interactions 패턴](https://fravoll.github.io/solidity-patterns/checks_effects_interactions.html)을 엄격하게 준수하려면 외부 상호 작용 전에 이벤트를 방출하는 것이 좋습니다. 이 패턴을 구현하는 것은 일반적으로 상태 재구성을 통한 올바른 마이그레이션을 보장하기 위해 권장되며, 이 경우 `Well.sol`의 모든 인스턴스가 `nonReentrant` 수정자에 의해 보호되므로 영향을 받지 않아야 하지만 여전히 좋은 관행입니다.

**Beanstalk:** 변경의 복잡성과 선택 사항이므로 구현하지 않기로 결정했습니다.

**Cyfrin:** 인정함.


### 시간 가중 평균 가격(TWAP) 오라클은 조작에 취약함 (Time Weighted Average Price oracles are susceptible to manipulation)

온체인 TWAP 오라클은 [조작에 취약](https://eprint.iacr.org/2022/445.pdf)하다는 점에 유의해야 합니다. 이를 사용하여 온체인 프로토콜의 중요한 부분을 구동하는 것은 [잠재적으로 위험](https://blog.openzeppelin.com/secure-smart-contract-guidelines-the-dangers-of-price-oracles/)합니다.

**Beanstalk:** 이는 조작 저항 오라클에 대한 우리의 최선의 시도입니다. 조작은 항상 가능하지만 우리 구현에는 오라클 조작에 대한 상당한 보호 기능이 있다고 믿습니다.

**Cyfrin:** 인정함.

\clearpage
## 가스 최적화 (Gas Optimization)


### 모듈로 연산 단순화 (Simplify modulo operations)

`LibBytes::storeUint128` 및 `LibBytes::readUint128`에서 `reserves.lenth % 2 == 1` 및 `i % 2 == 1`은 `reserves.length & 1 == 1` 및 `i & 1 == 1`로 단순화할 수 있습니다.

**Beanstalk:** [9db714a](https://github.com/BeanstalkFarms/Basin/commits/9db714a35a8674852e4c9ac41e1b9e548e21b38f) 커밋에서 수정되었습니다.

**Cyfrin:** 인정함.


### 분기 없는 최적화 (Branchless optimization)

`MathLib`의 `sqrt` 함수와 [관련 주석](https://github.com/BeanstalkFarms/Wells/blob/e5441fc78f0fd4b77a898812d0fd22cb43a0af55/src/libraries/LibMath.sol#L129-L136)은 이제 [분기 없는 최적화](https://github.com/transmissions11/solmate/blob/1b3adf677e7e383cc684b5d5bd441da86bf4bf1c/src/utils/FixedPointMathLib.sol#L220-L225) `z := sub(z, lt(div(x, z), z))`를 포함하는 Solmate의 `FixedPointMathLib` 변경 사항을 반영하도록 업데이트되어야 합니다.

**Beanstalk:** [46b24ac](https://github.com/BeanstalkFarms/Basin/commit/46b24aca0c65793604e3c24a07debfe163c5322a) 커밋에서 수정되었습니다.

**Cyfrin:** 인정함.

\clearpage

