**수석 감사자**

[Dacian](https://twitter.com/DevDacian)

[Carlitox477](https://twitter.com/carlitox477)

**보조 감사자**



---

# 발견 사항 (Findings)
## 중급 위험 (Medium Risk)


### 공격자가 `RewardsDistributor::triggerRoot`를 악용하여 보상 청구를 차단하고 일시 중지된 상태를 해제할 수 있음

**설명:** [`RewardsDistributor::triggerRoot`](https://github.com/SolidlyV3/v3-rewards/blob/6dfb435392ffa64652c8f88c98698756ca80cf28/contracts/RewardsDistributor.sol#L511-L516)의 코드를 고려해 보십시오:
```solidity
    function triggerRoot() external {
        bytes32 rootCandidateAValue = rootCandidateA.value;
        if (rootCandidateAValue != rootCandidateB.value || rootCandidateAValue == bytes32(0)) revert RootCandidatesInvalid();
        root = Root({value: rootCandidateAValue, lastUpdatedAt: block.timestamp});
        emit RootChanged(msg.sender, rootCandidateAValue);
    }
```

이 함수는:
* 누구나 호출할 수 있습니다.
* 성공하면 `root.value`를 `rootCandidateA.value`로, `root.lastUpdatedAt`을 `block.timestamp`로 설정합니다.
* `rootCandidateA` 또는 `rootCandidateB`를 재설정하지 않으므로 계속 반복해서 호출하여 `root.lastUpdatedAt`을 지속적으로 업데이트하거나 `root.value`를 `rootCandidateA.value`로 설정할 수 있습니다.

**영향:** 공격자는 이 함수를 2가지 방식으로 악용할 수 있습니다:
* 반복적으로 호출하여 `root.lastUpdatedAt`을 지속적으로 증가시켜 `RewardsDistributor::claimAll`에서 [청구 지연 revert](https://github.com/SolidlyV3/v3-rewards/blob/6dfb435392ffa64652c8f88c98698756ca80cf28/contracts/RewardsDistributor.sol#L190-L191)를 트리거하여 효과적으로 보상 청구를 차단할 수 있습니다.
* 보상 청구가 일시 중지된 후 호출하여 일시 중지된 상태를 효과적으로 해제할 수 있습니다. 이는 `root.value`가 `rootCandidateA.value`의 유효한 값으로 덮어써지기 때문이며, 청구 일시 중지는 `root.value == zeroRoot`로 설정하여 [작동](https://github.com/SolidlyV3/v3-rewards/blob/6dfb435392ffa64652c8f88c98698756ca80cf28/contracts/RewardsDistributor.sol#L547)하기 때문입니다.

**권장 완화 조치:** 두 가지 가능한 옵션:
* `RewardsDistributor::triggerRoot`를 허가된(permissioned) 함수로 만들어 공격자가 호출할 수 없도록 합니다.
* `RewardsDistributor::triggerRoot`를 변경하여 `rootCandidateA.value = zeroRoot`로 재설정하여 반복적으로 성공적으로 호출될 수 없도록 합니다.

**Solidly:**
커밋 [653c196](https://github.com/SolidlyV3/v3-rewards/commit/653c19659474c93ef0958479191d8103bc7b7e82) & [1170eac](https://github.com/SolidlyV3/v3-rewards/commit/1170eacc9b08bed9453a34fdf498f8bb10457f17)에서 수정되었습니다.

**Cyfrin:**
확인됨. 업데이트된 구현의 한 가지 결과는 계약이 "일시 중지됨" 상태로 시작하고 루트 후보를 설정할 수 없다는 것입니다. 즉, 배포 후 초기 상태에서 "일시 중지 해제"하려면 관리자가 `setRoot`를 통해 첫 번째 유효한 루트를 설정해야 합니다.


### `RewardsDistributor`가 전송 수수료(Fee-On-Transfer) 인센티브 토큰의 예치금을 올바르게 처리하지 않음

**설명:** `the kenneth`는 텔레그램에서 전송 수수료 토큰을 `RewardsDistributor`의 인센티브 토큰으로 사용해도 괜찮다고 말했지만, 전송 수수료 토큰을 받고 보상 금액을 저장할 때 회계 처리가 전송 중 공제된 수수료를 고려하지 않습니다. [예를 들어](https://github.com/SolidlyV3/v3-rewards/blob/6dfb435392ffa64652c8f88c98698756ca80cf28/contracts/RewardsDistributor.sol#L348-L359):

```solidity
function _depositLPIncentive(
    StoredReward memory reward,
    uint256 amount,
    uint256 periodReceived
) private {
    IERC20(reward.token).safeTransferFrom(
        msg.sender,
        address(this),
        amount
    );

    // @audit 여기에 저장된 `amount`는 전송 수수료가 전송 중 
    // 공제된 후 실제로 받은 금액을 고려하지 않으므로 부정확합니다.
    _storeReward(periodReceived, reward, amount);
}
```

**영향:** 실제 보상 계산은 오프체인에서 수행되며 감사 범위 밖이거나 해당 코드를 볼 수 없습니다. 그러나 `RewardsDistributor`에서 방출된 이벤트와 `RewardsDistributor::periodRewards`에 저장된 인센티브 토큰 예치금은 전송 수수료 인센티브 토큰 예치금에 대해 잘못된 금액을 사용합니다.

**권장 완화 조치:** `RewardsDistributor::_depositLPIncentive` & `depositVoteIncentive`에서:
* `RewardsDistributor` 계약의 전송 `전(before)` 토큰 잔액을 읽습니다.
* 토큰 전송을 수행합니다.
* `RewardsDistributor` 계약의 전송 `후(after)` 토큰 잔액을 읽습니다.
* `후` 잔액과 `전` 잔액의 차이를 계산하여 전송 중 공제된 수수료를 고려하여 `RewardsDistributor` 계약이 받은 실제 금액을 얻습니다.
* 실제 수령 금액을 사용하여 이벤트를 생성하고 `RewardsDistributor::periodRewards`에 수령된 인센티브 토큰 금액을 기록합니다.

또한 `RewardsDistributor::periodRewards`는 계약 내에서 읽지 않고 쓰기만 합니다. 오프체인 처리에서 사용되지 않는 경우 제거하는 것을 고려하십시오.

**Solidly:**
커밋 [be54da1](https://github.com/SolidlyV3/v3-rewards/commit/be54da1fea0f1f6f3e4c6ee20464b962cbe2077f)에서 수정되었습니다.

**Cyfrin:**
확인됨.


### 공격자가 `RewardsDistributor` 내부 회계를 손상시켜 `cUSDCv3`와 같은 토큰에 대한 LP 토큰 인센티브 예치금을 revert 시킬 수 있음

**설명:** [cUSDCv3](https://etherscan.io/address/0x9e4dde024f7ea5cf13d5f09c51a1555305c99f0c#code#F1#L930)와 같은 일부 토큰은 `amount == type(uint256).max`에 대한 특수 케이스를 포함하고 있어 전송 함수에서 사용자의 잔액만 전송됩니다.

이러한 토큰의 경우 `depositLPTokenIncentive`를 통한 인센티브 예치금은 예상보다 적은 토큰을 전송합니다. 이 결과로 Compound와 같은 프로토콜이 `cUSDCv3`와 같은 토큰으로 풀에 인센티브를 제공하려는 경우, 공격자가 트랜잭션을 프론트런(front-run)하여 내부 회계를 손상시키고 강제로 revert 시킬 수 있습니다.

**영향:** `cUSDCv3`와 같은 토큰에 대한 인센티브 보상 예치금의 손상된 회계는 동일한 토큰을 사용하는 향후 인센티브 보상 예치금을 거부하는 데 악용될 수 있습니다.

**POC:**
다음 함수를 고려하십시오:
```solidity
function _validateIncentive(
    address token,
    uint256 amount,
    uint256 distributionStart,
    uint256 numDistributionPeriods
) private view {
    // 배포는 미래의 에포크 전환 시점에 시작되어야 하며 [1, max] 기간 동안 지속되어야 합니다.
    if (
        numDistributionPeriods == 0                  || // 0 기간 배포는 유효하지 않음
        numDistributionPeriods > maxIncentivePeriods || // 최대 기간을 초과하는 배포는 유효하지 않음
        distributionStart % EPOCH_DURATION != 0      || // 배포는 주(week)의 시작에 시작되어야 함
        distributionStart < block.timestamp             // 배포는 미래에 시작되어야 함
    ) revert InvalidIncentiveDistributionPeriod();

    // approvedIncentiveAmounts는 화이트리스트 토큰에 대해
    // 기간당 배포할 최소 토큰 양을 나타냅니다.
    uint256 minAmount = approvedIncentiveAmounts[token] * numDistributionPeriods;

    // @audit `amount == type(uint256).max`에 대해 유효성 검사 통과
    if (minAmount == 0 || amount < minAmount)
        revert InvalidIncentiveAmount();
}

function _depositLPIncentive(
    StoredReward memory reward,
    uint256 amount,
    uint256 periodReceived
) private {
    // @audit `amount == type(uint256).max`인 경우 `amount`가
    // 전송된다는 것을 보장하지 않음
    IERC20(reward.token).safeTransferFrom(msg.sender, address(this), amount);

    // @audit 이 경우 잘못된 `amount`가 저장됨
    _storeReward(periodReceived, reward, amount);
}
```
Compound와 같은 프로토콜이 2 기간 동안 `cUSDCv3`와 같은 토큰으로 풀에 인센티브를 제공하려는 경우:
1. Bob은 멤풀에서 이것을 보고 `RewardsDistributor.depositLPTokenIncentive(compound가 인센티브를 주려는 풀, cUSDCv3, type(uint256).max, DOS를 위한 배포 시작, 유효한 numDistributionPeriods)`를 호출합니다.
2. Compound가 유효한 호출을 하려고 할 때, Bob의 프론트런 트랜잭션으로 인해 금액 값이 `type(uint256).max`이므로 `periodRewards[period][rewardKey] += amount`가 오버플로되어 `_storeReward`가 revert 됩니다.


**권장 완화 조치:**
한 가지 가능한 해결책:

1) `_validateIncentive`를 2개의 함수로 나눕니다:

```solidity
function _validateDistributionPeriod(
    uint256 distributionStart,
    uint256 numDistributionPeriods
) private view {
    // 배포는 미래의 에포크 전환 시점에 시작되어야 하며 [1, max] 기간 동안 지속되어야 합니다.
    if (
        numDistributionPeriods == 0                  || // 0 기간 배포는 유효하지 않음
        numDistributionPeriods > maxIncentivePeriods || // 최대 기간을 초과하는 배포는 유효하지 않음
        distributionStart % EPOCH_DURATION != 0      || // 배포는 주(week)의 시작에 시작되어야 함
        distributionStart < block.timestamp             // 배포는 미래에 시작되어야 함
    ) revert InvalidIncentiveDistributionPeriod();
}

// 이 함수를 호출하기 전에 _validateDistributionPeriod를 호출해야 함
function _validateIncentive(
    address token,
    uint256 amount,
    uint256 numDistributionPeriods
) private view {
    uint256 minAmount = approvedIncentiveAmounts[token] * numDistributionPeriods;

    if (minAmount == 0 || amount < minAmount)
        revert InvalidIncentiveAmount();
}
```

2) 실제 받은 금액을 반환하고 `_validateIncentive`를 호출하도록 `_depositLPIncentive`를 변경합니다:

```diff
function _depositLPIncentive(
    StoredReward memory reward,
+   uint256 numDistributionPeriods
    uint256 amount,
    uint256 periodReceived
-) private {
+) private returns(uint256 actualDeposited) {
+   uint256 tokenBalanceBeforeTransfer = IERC20(reward.token).balanceOf(address(this));
    IERC20(reward.token).safeTransferFrom(
        msg.sender,
        address(this),
        amount
    );
-   _storeReward(periodReceived, reward, amount);
+   actualDeposited = IERC20(reward.token).balanceOf(address(this)) - tokenBalanceBeforeTransfer;
+   _validateIncentive(reward.token, actualDeposited, numDistributionPeriods);
+   _storeReward(periodReceived, reward, actualDeposited);
}
```

3) 새 함수를 사용하고, 실제 반환된 금액을 읽고 이벤트 방출에 사용하도록 `depositLPTokenIncentive`를 변경합니다:

```diff
function depositLPTokenIncentive(
    address pool,
    address token,
    uint256 amount,
    uint256 distributionStart,
    uint256 numDistributionPeriods
) external {
-   _validateIncentive(
-       token,
-       amount,
-       distributionStart,
-       numDistributionPeriods
-   );
+   // Verify that number of period is and start time is valid
+   _validateDistributionPeriod(
+       uint256 distributionStart,
+       uint256 numDistributionPeriods
+   );
    StoredReward memory reward = StoredReward({
        _type: StoredRewardType.LP_TOKEN_INCENTIVE,
        pool: pool,
        token: token
    });
    uint256 periodReceived = _syncActivePeriod();
-   _depositLPIncentive(reward, amount, periodReceived);
+   uint256 actualDeposited = _depositLPIncentive(reward, amount, periodReceived);

    emit LPTokenIncentiveDeposited(
        msg.sender,
        pool,
        token,
-       amount,
+       actualDeposited
        periodReceived,
        distributionStart,
        distributionStart + (EPOCH_DURATION * numDistributionPeriods)
    );
}
```

이 완화 조치는 전송 수수료 토큰에 대한 잘못된 회계 관련 문제도 해결합니다.

**Solidly:**
커밋 [be54da1](https://github.com/SolidlyV3/v3-rewards/commit/be54da1fea0f1f6f3e4c6ee20464b962cbe2077f)에서 수정되었습니다.

**Cyfrin:**
확인됨.

\clearpage
## 저위험 (Low Risk)


### 반환 데이터가 필요하지 않을 때 가스 그리핑(gas griefing) 공격을 방지하기 위해 저수준 `call()` 사용

**설명:** 반환 데이터가 필요하지 않을 때 `call()`을 사용하면 반환된 거대한 데이터 페이로드로 인한 가스 그리핑 공격에 불필요하게 노출됩니다. [예](https://github.com/SolidlyV3/v3-rewards/blob/6dfb435392ffa64652c8f88c98698756ca80cf28/contracts/RewardsDistributor.sol#L563-L564):

```solidity
(bool sent, ) = _to.call{value: _amount}("");
require(sent);
```

는 다음과 동일합니다:

```solidity
(bool sent, bytes memory data) = _to.call{value: _amount}("");
require(sent);
```

두 경우 모두 반환 데이터가 전혀 사용되지 않음에도 불구하고 반환 데이터가 메모리에 복사되어야 하므로 계약이 가스 그리핑 공격에 노출됩니다.

**영향:** 계약이 불필요하게 가스 그리핑 공격에 노출됩니다.

**권장 완화 조치:** 반환 데이터가 필요하지 않은 경우 저수준 호출을 사용하십시오. 예:

```solidity
bool sent;
assembly {
    sent := call(gas(), _to, _amount, 0, 0, 0, 0)
}
if (!sent) revert FailedToSendEther();
```

**Solidly:**
커밋 [be54da1](https://github.com/SolidlyV3/v3-rewards/commit/be54da1fea0f1f6f3e4c6ee20464b962cbe2077f)에서 수정되었습니다.

**Cyfrin:**
확인됨.


### `RewardsDistributor::depositLPSolidEmissions`, `depositLPTokenIncentive` 및 `_collectPoolFees`에서 유효한 풀 확인

**설명:** `RewardsDistributor::depositLPSolidEmissions` 및 `depositLPTokenIncentive`는 `pool`이 유효한 풀 주소인지에 대한 유효성 검사가 없는 반면, `depositVoteIncentive`는 pool 매개변수에 대한 일부 유효성 검사를 수행합니다. LP 배출/인센티브가 유효한 `pool` 매개변수에 대해 기록되도록 유효성 검사를 추가하는 것을 고려하십시오.

마찬가지로 `RewardsDistributor::_collectPoolFees`는 풀이 합법적인지 확인하지 않으며 누구나 상위 함수인 `collectPoolFees`를 호출할 수 있습니다. 공격자는 `ISolidlyV3PoolMinimal::collectProtocol`을 구현하지만 토큰을 전송하지 않고 큰 출력 금액만 반환하며, `token0`과 `token1`에 대해 인기 있는 유명 토큰의 주소를 반환하는 자체 가짜 풀을 만들 수 있습니다.

이로 인해 잘못된 정보로 이벤트 로그와 `periodRewards` 저장소 위치를 손상시켜 `RewardsDistributor`가 실제보다 훨씬 더 많은 보상을 받은 것처럼 보이게 할 수 있습니다. `RewardsDistributor::_collectPoolFees`에서 풀을 확인하고 잠재적으로 `RewardsDistributor`가 실제로 토큰을 받았는지 확인하는 것을 고려하십시오.

또한 `RewardsDistributor::periodRewards`는 계약 내에서 읽지 않고 쓰기만 합니다. 오프체인 처리에서 사용되지 않는 경우 제거하는 것을 고려하십시오.

**Solidly:**
인지함. 오프체인 프로세서는 팩토리를 통해 검증된 풀만 계산합니다.


### `SolidlyV3Pool::_mint` 및 `_swap`은 토큰이 실제로 풀에 수신되었는지 확인하지 않음

**설명:** [`SolidlyV3Pool::_mint`](https://github.com/SolidlyV3/v3-core/blob/main/contracts/SolidlyV3Pool.sol#L288-L291) 및 [`_swap`](https://github.com/SolidlyV3/v3-core/blob/callbacks/contracts/SolidlyV3Pool.sol#L644-L650)의 일부 버전은 토큰이 실제로 풀에 수신되었는지 확인하지 않습니다. 대조적으로 UniswapV3의 동등한 [`mint`](https://github.com/Uniswap/v3-core/blob/main/contracts/UniswapV3Pool.sol#L483-L484) 및 [`swap`](https://github.com/Uniswap/v3-core/blob/main/contracts/UniswapV3Pool.sol#L777-L783) 함수는 항상 토큰이 풀에 수신되었는지 확인합니다.

**영향:** Solidly는 악성 토큰이나 비표준 동작을 하는 토큰에 더 취약할 것입니다. 가능한 공격 경로 중 하나는 블랙리스트에 오른 계정의 전송을 처리하지 않지만 revert 하지 않고 단순히 `true`를 반환하는 블랙리스트가 있는 토큰입니다. 토큰 소유자는 다음을 통해 더 교묘한 러그풀을 실행할 수 있습니다:
* 풀이 충분한 크기로 성장하도록 허용
* 자신을 블랙리스트에 추가
* 악성 토큰을 실제로 전송하지 않고 `swap`을 호출하여 다른 토큰을 고갈시켜 유동성 풀을 고갈시킴.

**권장 완화 조치:** `_mint` 및 `_swap` 함수는 예상 토큰 금액이 풀로 전송되었는지 확인해야 합니다.

**Solidly:**
v3-core에서 이국적인 ERC20을 지원하지 않기 때문에 가스 절약을 위해 의도적으로 생략했습니다. 사용자가 원한다면 무허가형이므로 그러한 풀을 만들 수 있지만, 우리가 명시적으로 공식적으로 지원하지 않는 것입니다.


### 이전 버전에 Merkle Multi Proof 보안 취약점이 있었으므로 `v3-rewards/package.json`을 변경하여 최소 OpenZeppelin v4.9.2를 요구하도록 함

**설명:** `v3-rewards/package.json`은 현재 최소 OpenZeppelin 버전을 4.5.0으로 [지정](https://github.com/SolidlyV3/v3-rewards/blob/6dfb435392ffa64652c8f88c98698756ca80cf28/package.json#L7)합니다. 그러나 일부 이전 OZ 버전에는 Merkle Multi Proof에 보안 [취약점](https://github.com/OpenZeppelin/openzeppelin-contracts/security/advisories/GHSA-wprv-93r4-jj2p)이 포함되어 있었으며 이는 4.9.2에서 수정되었습니다.

**권장 완화 조치:** 최소 OpenZeppelin v4.9.2를 요구하도록 `v3-rewards/package.json`을 변경하십시오:
```solidity
"@openzeppelin/contracts": "^4.9.2",
```

**Solidly:**
커밋 [6481747](https://github.com/SolidlyV3/v3-rewards/commit/6481747737b98c8650a36f87b1aeace815505ba9)에서 수정되었습니다.

**Cyfrin:**
확인됨.

\clearpage
## 정보성 (Informational)


### 여러 곳에서 사용되므로 하드코딩된 최대 풀 수수료를 상수로 리팩터링

**설명:** `100000`은 하드코딩된 최대 풀 수수료입니다. `SolidlyV3Factory::enableFeeAmount` [L90](https://github.com/SolidlyV3/v3-core/blob/main/contracts/SolidlyV3Factory.sol#L90) 및 `SolidlyV3Pool::setFee` [L794](https://github.com/SolidlyV3/v3-core/blob/main/contracts/SolidlyV3Pool.sol#L794)의 두 곳에서 이 하드코딩된 값을 시행하는 require 문이 있습니다.

코드 전체의 여러 곳에서 동일한 하드코딩된 값을 사용하는 것은 오류가 발생하기 쉽습니다. 향후 코드 업데이트 시 개발자가 한 곳은 업데이트하지만 다른 곳은 업데이트하는 것을 잊을 수 있기 때문입니다. 하드코딩 대신 참조할 수 있는 상수를 사용하도록 리팩터링하는 것을 권장합니다.

**Solidly:**
인지함.


### 소유권 포기 및 2단계 소유권 이전을 위한 명시적 기능 선호

**설명:** [`RewardsDistributor::setOwner`](https://github.com/SolidlyV3/v3-rewards/blob/6dfb435392ffa64652c8f88c98698756ca80cf28/contracts/RewardsDistributor.sol#L459-L462) 및 [`SolidlyV3Factory::setOwner`](https://github.com/SolidlyV3/v3-core/blob/callbacks/contracts/SolidlyV3Factory.sol#L60-L64)는 현재 소유자가 `owner = address(0)`으로 설정하여 소유권을 차단할 수 있게 하며, 이는 향후 관리자 기능에 대한 액세스를 방지합니다. 실수로 이러한 일이 발생하는 것을 방지하기 위해 소유권 포기를 위한 명시적 기능을 선호하고 2단계 소유권 이전 메커니즘을 선호하십시오. 이 두 기능 모두 OZ [Ownable2Step](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/access/Ownable2Step.sol)에서 사용할 수 있습니다.

**Solidly:**
인지함.


### `require` 및 `revert` 문에는 설명적인 이유 문자열이 있어야 함

**설명:** `require` 및 `revert` 문에는 설명적인 이유 문자열이 있어야 합니다:

```solidity
File: SolidlyV3Factory.sol

46:         require(tokenA != tokenB);

48:         require(token0 != address(0));

50:         require(tickSpacing != 0);

51:         require(getPool[token0][token1][tickSpacing] == address(0));

61:         require(msg.sender == owner);

68:         require(msg.sender == owner);

75:         require(msg.sender == owner);

88:         require(msg.sender == owner);

89:         require(fee <= 100000);

94:         require(tickSpacing > 0 && tickSpacing < 16384);

95:         require(feeAmountTickSpacing[fee] == 0);

```

```solidity
File: SolidlyV3Pool.sol

116:         require(success && data.length >= 32);

127:         require(success && data.length >= 32);

302:         require(amount > 0);

327:         require(amount > 0);

947:         require(fee <= 100000);

```

```solidity
File: libraries/FullMath.sol

34:             require(denominator > 0);

43:         require(denominator > prod1);

120:             require(result < type(uint256).max);

```

```solidity
File: RewardsDistributor.sol

564:        require(sent);

595:        require(success && data.length >= 32);

```

**Solidly:**
인지함.


### 내부적으로 사용되지 않는 함수를 external로 표시할 수 있음

**설명:** 내부적으로 사용되지 않는 함수를 external로 표시할 수 있습니다:

```solidity
File: SolidlyV3Factory.sol

87:     function enableFeeAmount(uint24 fee, int24 tickSpacing) public override {

```

**Solidly:**
인지함.


### 여러 함수에 선언된 `zeroRoot`를 private 상수로 리팩터링

**설명:** `zeroRoot`는 `RewardsDistributor::pauseClaimsGovernance` [L546](https://github.com/SolidlyV3/v3-rewards/blob/6dfb435392ffa64652c8f88c98698756ca80cf28/contracts/RewardsDistributor.sol#L546) 및 `pauseClaimsPublic` [L554](https://github.com/SolidlyV3/v3-rewards/blob/6dfb435392ffa64652c8f88c98698756ca80cf28/contracts/RewardsDistributor.sol#L554)에서 선언되고 사용됩니다. 여러 함수에서 선언하지 않도록 private 상수로 리팩터링하는 것을 고려하십시오.

**Solidly:**
커밋 [653c196](https://github.com/SolidlyV3/v3-rewards/commit/653c19659474c93ef0958479191d8103bc7b7e82)에서 수정되었습니다.

**Cyfrin:**
확인됨.


### 하드코딩된 일시 중지 담보 수수료는 멀티체인 사용에 적합하지 않음

**설명:** Solidly가 향후 멀티체인을 목표로 하므로, `RewardsDistributor::pauseClaimsPublic`에 5 ether의 일시 중지 담보 수수료를 [하드코딩](https://github.com/SolidlyV3/v3-rewards/blob/6dfb435392ffa64652c8f88c98698756ca80cf28/contracts/RewardsDistributor.sol#L553)하는 것은 다른 체인에서 적합하지 않을 수 있습니다. 이 금액이 매우 적은 가치를 나타낼 수 있기 때문입니다. 일시 중지 담보 수수료에 대한 `public` 저장소 변수와 이를 설정하는 `onlyOwner` 함수를 갖는 것을 고려하십시오.

**Solidly:**
커밋 [653c196](https://github.com/SolidlyV3/v3-rewards/commit/653c19659474c93ef0958479191d8103bc7b7e82)에서 수정되었습니다.

**Cyfrin:**
확인됨.


### `RewardsDistributor::_claimSingle`은 `amountDelta`를 사용하여 `RewardClaimed`를 방출해야 함

**설명:** `RewardsDistributor::_claimSingle`에서 `amount` 매개변수는 `previouslyClaimed` 매개변수에서 차감됩니다. 사용자가 10개의 보상 토큰을 받을 자격이 있는 경우를 고려하십시오.

사용자가 10개의 토큰을 청구합니다.

그런 다음 나중에 사용자가 동일한 풀/토큰/유형(`rewardKey`)에 대해 10개의 토큰을 더 받을 자격이 생깁니다. 사용자가 `amount = 10`으로 청구하려고 하면 이전에 청구한 금액의 차감으로 인해 실패합니다; 사용자는 `amount = 20`으로 청구해야 통과합니다.

이 설계는 다소 혼란스러워 보입니다; 사용자는 자신이 청구한 총 금액을 추적한 다음, 청구할 수 있는 새 금액을 더하고, 그 총 금액으로 청구를 호출해야 합니다.

차이인 `amountDelta`만 사용자에게 전송되지만, `RewardClaimed` 이벤트는 `amount`로 방출됩니다. 따라서 위의 시나리오에서 사용자가 총 20개의 보상 토큰만 받았음에도 불구하고 `amount (10)`과 `amount (20)`으로 두 개의 `RewardClaimed` 이벤트가 방출됩니다.

사용자가 단순히 청구할 자격이 있는 금액으로 호출할 수 있도록 이 함수를 리팩터링하거나, 최소한 `amount` 대신 `amountDelta`를 사용하도록 이벤트 방출을 변경하는 것을 고려하십시오.

**Solidly:**
인지함.


### `CollateralWithdrawn` 및 `CollateralDeposited` 이벤트에 관련 금액 포함해야 함

**설명:** `RewardsDistributor::withdrawCollateral`에서 `CollateralWithdrawn` 이벤트를 방출할 때 `_amount` 매개변수를 추가하십시오. 전송된 금액이 예치된 금액과 동일할 필요는 없기 때문에 필요합니다.

필수 담보 금액이 변경되어 현재 가치가 발생한 모든 담보 예치금에 대해 사실이 아닐 수 있으므로 `CollateralDeposited` 이벤트에도 예치된 금액을 추가하는 것을 고려하십시오.

**Solidly:**
커밋 [6481747](https://github.com/SolidlyV3/v3-rewards/commit/6481747737b98c8650a36f87b1aeace815505ba9)에서 수정되었습니다.

**Cyfrin:**
확인됨.

\clearpage
## 가스 최적화 (Gas Optimization)


### 루프 외부에서 배열 길이 캐시

**설명:** 루프 외부에서 배열 길이 캐시:

```solidity
File: contracts/RewardsDistributor.sol
// @audit use `numLeaves` from L263 instead of `earners.length`
265:         for (uint256 i; i < earners.length; ) {
```

**Solidly:**
커밋 [6481747](https://github.com/SolidlyV3/v3-rewards/commit/6481747737b98c8650a36f87b1aeace815505ba9)에서 수정되었습니다.

**Cyfrin:**
확인됨.


### 변수를 기본값으로 초기화하지 말 것

**설명:** 변수를 기본값으로 초기화하지 마십시오:

```solidity
File: contracts/RewardsDistributor.sol

184:         for (uint256 i = 0; i < numClaims; ) {

```

```solidity
File: libraries/TickMath.sol

67:         uint256 msb = 0;

```

**Solidly:**
`RewardsDistributor`에 대해 커밋 [6481747](https://github.com/SolidlyV3/v3-rewards/commit/6481747737b98c8650a36f87b1aeace815505ba9)에서 수정되었습니다; v3-core는 이미 배포되었으며 업그레이드할 수 없습니다.

**Cyfrin:**
확인됨.


### `x++`보다 `++x` 선호

**설명:** `x++`보다 `++x`를 선호하십시오:

File: `TickBitmap.sol`
```solidity
48:        if (tick < 0 && tick % tickSpacing != 0) compressed--; // round towards negative infinity
```

File: `SolidlyV3Pool.sol`
```solidity
965:            if (amount0 == poolFees.token0) amount0--; // ensure that the slot is not cleared, for gas savings
```

File: `SolidlyV3Pool.sol`
```solidity
970:            if (amount1 == poolFees.token1) amount1--; // ensure that the slot is not cleared, for gas savings
```

File: `FullMath.sol`
```solidity
120:            result++;
```

**Solidly:**
인지함.


### 변경되지 않고 여러 번 읽힐 때 저장소 변수를 메모리에 캐시

**설명:** 변경되지 않고 여러 번 읽힐 때 저장소 변수를 메모리에 캐시하십시오:

File: `SolidlyV3Pool.sol`
```solidity
// @audit `slot0.fee`는 변경되지 않으므로 저장소에서 두 번 로드할 필요 없음;
// 저장소에서 메모리로 한 번 로드한 다음 메모리 내 복사본 사용
913:        uint256 fee0 = FullMath.mulDivRoundingUp(amount0, slot0.fee, 1e6);
914:        uint256 fee1 = FullMath.mulDivRoundingUp(amount1, slot0.fee, 1e6);


// @audit `poolFees.token0` 및 `poolFees.token1`은 저장소에서 여러 번 읽히지만
// L966 & L971까지 변경되지 않음. 둘 다 저장소에서 메모리로 한 번 로드한 다음
// 저장소에서 같은 값을 반복해서 읽는 대신 메모리 내 복사본 사용
961:        amount0 = amount0Requested > poolFees.token0 ? poolFees.token0 : amount0Requested;
962:        amount1 = amount1Requested > poolFees.token1 ? poolFees.token1 : amount1Requested;

964:        if (amount0 > 0) {
965:            if (amount0 == poolFees.token0) amount0--; // ensure that the slot is not cleared, for gas savings
966:            poolFees.token0 -= amount0;
967:            TransferHelper.safeTransfer(token0, recipient, amount0);
968:        }
969:        if (amount1 > 0) {
970:            if (amount1 == poolFees.token1) amount1--; // ensure that the slot is not cleared, for gas savings
971:            poolFees.token1 -= amount1;
972:            TransferHelper.safeTransfer(token1, recipient, amount1);
973:        }
```

File: `SolidlyV3Factory.sol`
```solidity
// @audit `owner`는 저장소에서 두 번 읽히며 매번 같은 값을 반환함. 저장소에서
// 메모리로 한 번 읽은 다음, 두 번 모두 메모리 내 복사본 사용
61:        require(msg.sender == owner);
62:        emit OwnerChanged(owner, _owner);
```

**Solidly:**
인지함.


### 여러 문이 있는 단일 require 대신 다중 require를 사용하는 것이 가스 소비에 더 좋음

**설명:** 여러 `&&`가 있는 단일 require 대신 다중 require를 사용하는 것이 가스 소비에 더 좋습니다. 그 이유는 `require`가 revert 될 경우 [가스를 소비하지 않는 revert](https://ethereum-org-fork.netlify.app/developers/docs/evm/opcodes)로 변환되기 때문입니다. 그러나 `&&`는 가스를 소비합니다. 따라서 여러 require를 선택하는 것이 참이어야 하는 여러 문이 있는 단일 require를 선택하는 것보다 가스 효율적입니다.

```solidity
// SolidlyV3Pool.sol
116:    require(success && data.length >= 32);
127:    require(success && data.length >= 32);
278:    require(amount0 >= amount0Min && amount1 >= amount1Min, 'AL');
293:    require(amount0 >= amount0Min && amount1 >= amount1Min, 'AL');
391:     require(amount0FromBurn >= amount0FromBurnMin && amount1FromBurn >= amount1FromBurnMin, 'AL');
456:    require(amount0 >= amount0Min && amount1 >= amount1Min, 'AL');

// RewardsDistributor.sol
607:    require(success && data.length >= 32);
```

**Solidly:**
인지함.


### `RewardsDistributor::generateLeaves`에서 두 개의 메모리 변수 최적화

**설명:** 명명된 반환 변수를 사용하고 임시 `leaf` 변수를 제거하여 `RewardsDistributor::generateLeaves`에서 두 개의 메모리 변수를 최적화하십시오:

```solidity
function _generateLeaves(
    address[] calldata earners,
    EarnedRewardType[] calldata types,
    address[] calldata pools,
    address[] calldata tokens,
    uint256[] calldata amounts
) private pure returns (bytes32[] memory leaves) {
    uint256 numLeaves = earners.length;
    // @audit 명명된 반환 변수 사용
    leaves            = new bytes32[](numLeaves);

    // @audit 루프에서 캐시된 배열 길이 사용
    for (uint256 i; i < numLeaves; ) {
        // @audit 반환 변수에 바로 할당
        leaves[i] = keccak256(
            bytes.concat(
                keccak256(
                    abi.encode(
                        earners[i],
                        types[i],
                        pools[i],
                        tokens[i],
                        amounts[i]
                    )
                )
            )
        );
        unchecked {
            ++i;
        }
    }
    return leaves;
}
```

**Solidly:**
인지함.

\clearpage
