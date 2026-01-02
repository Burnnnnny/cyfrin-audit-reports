**수석 감사 (Lead Auditors)**

[Dacian](https://twitter.com/DevDacian)

**보조 감사 (Assisting Auditors)**



---

# 발견 사항 (Findings)
## 낮은 위험 (Low Risk)


### `InvestorBasedRateLimiter::setInvestorMintLimit` 및 `setInvestorRedemptionLimit`은 언더플로로 인해 `checkAndUpdateMintLimit` 및 `checkAndUpdateRedemptionLimit`에 대한 후속 호출을 되돌릴 수 있음

**설명:** `InvestorBasedRateLimiter::_checkAndUpdateRateLimitState` [L211-213](https://github.com/ondoprotocol/rwa-internal/blob/6747ebada1c867a668a8da917aaaa7a0639a5b7a/contracts/ousg/InvestorBasedRateLimiter.sol#L211-L213)은 해당 제한에서 현재 발행/상환 금액을 뺍니다:
```solidity
if (amount > rateLimit.limit - rateLimit.currentAmount) {
  revert RateLimitExceeded();
}
```

`setInvestorMintLimit` 또는 `setInvestorRedemptionLimit`을 사용하여 발행 또는 상환 한도 금액을 현재 발행/상환 금액보다 작게 설정하면, 이 함수에 대한 호출은 언더플로로 인해 되돌려집니다(revert).

**영향:** `InvestorBasedRateLimiter::setInvestorMintLimit` 및 `setInvestorRedemptionLimit`은 언더플로로 인해 `checkAndUpdateMintLimit` 및 `checkAndUpdateRedemptionLimit`에 대한 후속 호출을 되돌릴 수 있습니다.

**권장 완화 방법:** 제한이 현재 발행/상환 금액보다 작은 경우를 명시적으로 처리하십시오:
```solidity
if (rateLimit.limit <= rateLimit.currentAmount || amount > rateLimit.limit - rateLimit.currentAmount) {
  revert RateLimitExceeded();
}
```

**Ondo:**
커밋 [fb8ecff](https://github.com/ondoprotocol/rwa-internal/commit/fb8ecff80960c8c891ddc206c6f6f27a620e42d6)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.


### 0 주소(zero address)와 연결된 투자자 기록 생성 방지

**설명:** `InvestorBasedRateLimiter::checkAndUpdateMintLimit` 및 `checkAndUpdateRedemptionLimit`은 새 투자자 기록을 생성하고 이를 0 주소와 연결할 수 있습니다.

**영향:** 0 주소와 연결된 투자자 기록이 생성될 수 있습니다. 이는 `InvestorBasedRateLimiter` 계약의 다음 불변성을 깨뜨립니다:

> 새 `investorId`가 생성되면 하나 이상의 유효한 주소와 연결되어야 함

**권장 완화 방법:** `_setAddressToInvestorId`에서 0 주소에 대해 되돌리십시오(revert):
```solidity
function _setAddressToInvestorId(
    address investorAddress,
    uint256 newInvestorId
) internal {
    if(investorAddress == address(0)) revert NoZeroAddress();
```

**Ondo:**
커밋 [bac99d0](https://github.com/ondoprotocol/rwa-internal/commit/bac99d03d75e84ea5541297b3aa0751283c1272e)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.


### 주소와 연결되지 않은 투자자 기록 생성 방지

**설명:** `InvestorBasedRateLimiter::initializeInvestorStateDefault`는 새로 생성된 투자자를 하나 이상의 주소와 연결해야 하지만, 이를 수행하는 `for` [루프](https://github.com/ondoprotocol/rwa-internal/blob/6747ebada1c867a668a8da917aaaa7a0639a5b7a/contracts/ousg/InvestorBasedRateLimiter.sol#L253-L260)는 빈 배열로 함수를 호출하여 우회할 수 있습니다.

```solidity
function initializeInvestorStateDefault(
    address[] memory addresses
    ) external onlyRole(CONFIGURER_ROLE) {
    _initializeInvestorState(
      addresses,
      defaultMintLimit,
      defaultRedemptionLimit,
      defaultMintLimitDuration,
      defaultRedemptionLimitDuration
    );
}

function _initializeInvestorState(
    address[] memory addresses,
    uint256 mintLimit,
    uint256 redemptionLimit,
    uint256 mintLimitDuration,
    uint256 redemptionLimitDuration
    ) internal {
    uint256 investorId = ++investorIdCounter;

    // @audit this `for` loop can by bypassed by calling
    // `initializeInvestorStateDefault` with an empty array
    for (uint256 i = 0; i < addresses.length; ++i) {
      // Safety check to ensure the address is not already associated with an investor
      // before associating it with a new investor
      if (addressToInvestorId[addresses[i]] != 0) {
        revert AddressAlreadyAssociated();
      }
      _setAddressToInvestorId(addresses[i], investorId);
    }
...
}
```

**영향:** 연결된 주소 없이 투자자 기록을 생성할 수 있습니다. 이는 `InvestorBasedRateLimiter` 계약의 다음 불변성을 깨뜨립니다:

> 새 `investorId`가 생성되면 하나 이상의 유효한 주소와 연결되어야 함

**권장 완화 방법:** `_initializeInvestorState`에서 입력 주소 배열이 비어 있으면 되돌리십시오(revert):
```solidity
uint256 addressesLength = addresses.length;

if(addressesLength == 0) revert EmptyAddressArray();
```

**Ondo:**
커밋 [bac99d0](https://github.com/ondoprotocol/rwa-internal/commit/bac99d03d75e84ea5541297b3aa0751283c1272e)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.


### `InstantMintTimeBasedRateLimiter::_setInstantMintLimit` 및 `_setInstantRedemptionLimit`은 언더플로로 인해 `_checkAndUpdateInstantMintLimit` 및 `_checkAndUpdateInstantRedemptionLimit`에 대한 후속 호출을 되돌릴 수 있음

**설명:** `InstantMintTimeBasedRateLimiter::_checkAndUpdateInstantMintLimit` [L103-106](https://github.com/ondoprotocol/rwa-internal/blob/6747ebada1c867a668a8da917aaaa7a0639a5b7a/contracts/InstantMintTimeBasedRateLimiter.sol#L103-L106)은 발행 한도에서 현재 발행된 금액을 뺍니다:
```solidity
require(
  amount <= instantMintLimit - currentInstantMintAmount,
  "RateLimit: Mint exceeds rate limit"
);
```

`_setInstantMintLimit`을 사용하여 `instantMintLimit < currentInstantMintAmount`로 설정하면, 이 함수에 대한 후속 호출은 언더플로로 인해 되돌려집니다. `_setInstantRedemptionLimit` 및 `_checkAndUpdateInstantRedemptionLimit`의 경우도 마찬가지입니다.

**영향:** `InstantMintTimeBasedRateLimiter::_setInstantMintLimit` 및 `_setInstantRedemptionLimit`은 언더플로로 인해 `_checkAndUpdateInstantMintLimit` 및 `_checkAndUpdateInstantRedemptionLimit`에 대한 후속 호출을 되돌릴 수 있습니다.

**권장 완화 방법:** 제한이 현재 발행/상환 금액보다 작은 경우를 명시적으로 처리하십시오:
```solidity
function _checkAndUpdateInstantMintLimit(uint256 amount) internal {
    require(
      instantMintLimit > currentInstantMintAmount && amount <= instantMintLimit - currentInstantMintAmount,
      "RateLimit: Mint exceeds rate limit"
    );
}

function _checkAndUpdateInstantRedemptionLimit(uint256 amount) internal {
    require(
      instantRedemptionLimit > currentInstantRedemptionAmount && amount <= instantRedemptionLimit - currentInstantRedemptionAmount,
      "RateLimit: Redemption exceeds rate limit"
    );
}
```

**Ondo:**
커밋 [fb8ecff](https://github.com/ondoprotocol/rwa-internal/commit/fb8ecff80960c8c891ddc206c6f6f27a620e42d6)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.


### BlackRock이 새 `BUIDLRedeemer` 계약을 배포하고 기존 계약을 중단하면 `OUSGInstantManager` 상환이 막힘

**설명:** `BUIDLRedeemer` 계약은 매우 새로운 계약입니다. 미래에 계약의 새 버전이 배포되고 현재 버전이 작동을 중지할 가능성이 매우 높습니다.

이러한 상황에서 `OUSGInstantManager`가 계속 작동하도록 보장하려면 `buidlRedeemer` 정의에서 `immutable` 키워드를 제거하고 나중에 업데이트할 수 있는 설정자(setter) 함수를 추가하십시오.

**Ondo:**
새 `BUIDLRedeemer` 계약이 배포되면 새 `OUSGInstantManager`를 배포할 계획입니다. 우리는 변경 사항에 대한 적절한 실사가 이루어지도록 `buidlRedeemer` 주소를 변경하기 어렵게 만드는 것을 선호합니다.


### `ROUSG::unwrap`은 사용자가 원래 래핑한 것보다 약간 적은 `OUSG` 토큰을 불필요하게 반환할 수 있음

**설명:** `ROUSG` 토큰의 불변성 중 하나는 다음과 같습니다:

> 래핑을 해제할 때 사용자는 가격에 관계없이 래핑했을 때 제공한 것과 동일한 양의 OUSG 입력 토큰을 받아야 함

그러나 `ROUSG::unwrap`이 사용자가 원래 래핑한 것보다 약간 적은 `OUSG` 토큰을 불필요하게 반환할 수 있으므로 항상 그런 것은 아닙니다.

**영향:** 사용자는 원래 래핑한 것보다 약간 적은 토큰을 불필요하게 받게 되어 `ROUSG` 계약의 불변성을 깨뜨립니다.

**권장 완화 방법:** `ROUSG::unwrap`, `burn` 및 `OUSGInstantManager::redeemRebasingOUSG`를 호출할 때 호출자는 `ROUSG` 토큰 금액을 전달하는 대신 `ROUSG::sharesOf`를 통해 검색할 수 있는 지분(share) 금액을 전달해야 합니다. 그러면 출력 토큰 계산을 `shares / OUSG_TO_ROUSG_SHARES_MULTIPLIER`로 수행할 수 있으며, 이는 항상 올바른 양의 토큰을 반환합니다.

기존 함수를 반드시 제거할 필요는 없지만 사용자가 지분 금액을 입력할 수 있도록 추가 함수를 생성해야 합니다.

**Ondo:**
커밋 [df0e491](https://github.com/ondoprotocol/rwa-internal/commit/df0e491fb081f4b7cd0d7329f8763e644ea77c18), [2aa437a](https://github.com/ondoprotocol/rwa-internal/commit/2aa437aa78435fc4533c3a9d223460da34e71647)에서 수정되었습니다. 필요한 코드 변경 양으로 인해 `OUSGInstantManager`를 변경하지 않기로 결정했습니다.

**Cyfrin:** 확인되었습니다.


### USDC 디페깅 이벤트 중 `BuidlRedeemer`에 의해 프로토콜이 손해를 볼 수 있음

**설명:** `OUSGInstantManager::_redeemBUIDL`은 상환하는 1 BUIDL마다 1 USDC를 받도록 [강제](https://github.com/ondoprotocol/rwa-internal/blob/6747ebada1c867a668a8da917aaaa7a0639a5b7a/contracts/ousg/ousgInstantManager.sol#L453-L459)하므로 1 BUIDL = 1 USDC라고 가정합니다:
```solidity
uint256 usdcBalanceBefore = usdc.balanceOf(address(this));
buidl.approve(address(buidlRedeemer), buidlAmountToRedeem);
buidlRedeemer.redeem(buidlAmountToRedeem);
require(
  usdc.balanceOf(address(this)) == usdcBalanceBefore + buidlAmountToRedeem,
  "OUSGInstantManager::_redeemBUIDL: BUIDL:USDC not 1:1"
);
```
USDC 디페그(특히 디페그가 지속되는 경우)가 발생하면 1 USDC는 1달러의 가치가 없으므로 `BUIDLRedeemer`는 1:1 비율보다 더 큰 값을 반환해야 합니다. 따라서 1 BUIDL != 1 USDC이며 이는 프로토콜의 BUIDL 가치가 더 많은 USDC 가치가 있음을 의미합니다. 그러나 `BUIDLReceiver`는 이를 수행하지 않고 항상 1:1로만 [반환](https://etherscan.io/address/0x9ba14Ce55d7a508A9bB7D50224f0EB91745744b7#code)합니다.

**영향:** USDC 디페그가 발생하면 USDC 디페그로 인해 1 BUIDL의 가치가 1 USDC의 가치보다 큼에도 불구하고 상환된 1 BUIDL마다 1 USDC만 받기 때문에 프로토콜은 `BuidlRedeemer`에 의해 손해를 보게 됩니다.

**권장 완화 방법:** 이 상황을 방지하려면 프로토콜은 오라클을 사용하여 USDC가 디페깅되었는지 확인하고, 그렇다면 BUIDL 대가로 받아야 할 USDC 금액을 계산해야 합니다. 손해를 보게 된다면 상환을 방지하거나 상환을 허용하되 손해 본 금액을 스토리지에 저장한 다음 BlackRock과 오프체인 프로세스를 구현하여 손해 본 금액을 받아야 합니다.

대안으로 프로토콜은 상환을 계속 허용하기 위해 USDC 디페그 중에 기꺼이 손해를 보는 것을 프로토콜의 위험으로 단순히 받아들일 수 있습니다.

**Ondo:**
커밋 [408bff1](https://github.com/ondoprotocol/rwa-internal/commit/408bff112c39f393f67dde6c30a6addf3b221ee9), [8a9cae9](https://github.com/ondoprotocol/rwa-internal/commit/8a9cae9af5787f06db42b4224b147d60493e0133)에서 수정되었습니다. 이제 Chainlink USDC/USD 오라클을 사용하며 USDC가 허용된 최소값 아래로 디페깅되면 발행 및 상환이 모두 중지됩니다.

**Cyfrin:** 확인되었습니다.

\clearpage
## 정보성 (Informational)


### `rOUSG` 토큰에 대한 무제한 승인 구현 고려

**설명:** ERC20 토큰은 일반적으로 사용자가 지출자(spender)를 `type(uint256).max`로 승인할 수 있도록 하여 무제한 승인을 구현합니다. 이 일반적인 기능을 구현하는 것을 고려하십시오. OpenZeppelin의 [예시](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/ERC20.sol#L301-L311)를 참조하십시오.

**Ondo:**
확인됨.


### `rOUSG::transferFrom`에서 토큰을 전송하기 전에 승인 감소

**설명:** `rOUSG::transferFrom` [L286-289](https://github.com/ondoprotocol/rwa-internal/blob/6747ebada1c867a668a8da917aaaa7a0639a5b7a/contracts/ousg/rOUSG.sol#L286-L289)는 현재 승인을 확인하고 토큰을 전송한 다음 승인을 줄입니다.

더 안전한 코딩 패턴은 OpenZeppelin의 [구현](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/ERC20.sol#L151-L152)과 유사하게 승인을 먼저 줄인 다음 토큰을 전송하는 것입니다.

**Ondo:**
확인됨.


### `rOUSG::wrap`에서 지분을 발행하기 전에 토큰 전송

**설명:** `rOUSG::wrap` [L411-413](https://github.com/ondoprotocol/rwa-internal/blob/6747ebada1c867a668a8da917aaaa7a0639a5b7a/contracts/ousg/rOUSG.sol#L411-L413)은 현재 해당 지분을 발행하는 데 사용된 토큰을 전송하기 전에 지분을 발행합니다.

더 안전한 코딩 패턴은 토큰을 먼저 전송한 다음 지분을 발행하는 것입니다.

**Ondo:**
확인됨.


### 프로토콜에 유리하도록 `OUSGInstantManager::_getInstantMintFees` 및 `_getInstantRedemptionFees`에서 수수료 올림(round up)

**설명:** Solidity는 기본적으로 내림(round down)하므로 `OUSGInstantManager::_getInstantMintFees` 및 `_getInstantRedemptionFees`에서 프로토콜에 유리하도록 수수료를 명시적으로 올림하는 것을 고려하십시오.

**Ondo:**
확인됨.


### rOUSG 지분의 먼지(dust) 양을 전송할 때 오해의 소지가 있는 이벤트가 발생함

**설명:** `ROUSG.transferShares`를 호출하면 두 가지 이벤트가 발생합니다:

`TransferShares`: 얼마나 많은 rOUSG 지분이 전송되었는지
`Transfer`: 얼마나 많은 rOUSG 토큰이 전송되었는지

먼지 양으로 이 함수를 호출하면 0이 아닌 양의 지분이 전송되었다는 이벤트와 함께 `getROUSGByShares`가 0으로 반올림되므로 0 토큰이 전송되었다는 이벤트가 발생합니다.

**Ondo:**
확인됨.


### `ROUSG::burn`이 먼지 양을 태울 수 있도록 허용하는 것 고려

**설명:** `ROUSG::burn`은 관리자가 규제상의 이유로 모든 계정에서 `rOUSG` 토큰을 태우는 데 사용됩니다.

이것은 1 wei의 `OUSG`보다 작기 때문에 1e4보다 작은 지분 양을 태우는 것을 허용하지 않습니다.

```solidity
if (ousgSharesAmount < OUSG_TO_ROUSG_SHARES_MULTIPLIER)
      revert UnwrapTooSmall();
```

현재 및 미래의 규제 상황에 따라 사용자의 모든 지분을 항상 태울 수 있어야 할 수도 있습니다.

**권장 완화 방법:** 최소 금액 미만이라도 `burn` 함수가 남은 모든 지분을 태울 수 있도록 허용하는 것을 고려하십시오.

**Ondo:**
커밋 [2aa437a](https://github.com/ondoprotocol/rwa-internal/commit/2aa437aa78435fc4533c3a9d223460da34e71647)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.


### `_assertUSDCPrice`는 solidity 스타일 가이드를 위반함

**설명:** `_assertUSDCPrice` 함수는 public이며 밑줄로 시작합니다. [solidity 스타일 가이드](https://docs.soliditylang.org/en/latest/style-guide.html)에 따르면 이 규칙은 비외부(non-external) 함수 및 상태 변수(private 또는 internal)에 제안됩니다.

**권장 완화 방법:** `_`를 제거하거나 함수의 가시성(visibility)을 변경하십시오.

**Ondo:**
커밋 [fc1c8fb](https://github.com/ondoprotocol/rwa-internal/commit/fc1c8fbd9efb77d4307611d83d7350d869a23e22)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.

\clearpage
## 가스 최적화 (Gas Optimization)


### 루프 외부에서 배열 길이를 캐시하고 unchecked 루프 증가 고려

**설명:** 루프 외부에서 배열 길이를 캐시하고 `solc --ir-optimized --optimize`로 컴파일하지 않는 경우 `unchecked {++i;}` 사용을 고려하십시오.


### 변경되지 않고 여러 번 읽을 때 스택에 스토리지 변수 캐시

**설명:** 스토리지에서 읽는 것은 스택에서 읽는 것보다 훨씬 비싸므로 변경되지 않고 여러 번 읽을 때 스토리지 변수를 캐시하십시오.

```solidity
File: contracts/ousg/InvestorBasedRateLimiter.sol

// @audit cache these then use cache values when emitting event to save 2 storage reads
324:      --investorAddressCount[previousInvestorId];
335:      ++investorAddressCount[newInvestorId];

// @audit cache and use cached value for check in L470 to save 1 storage read
462:    if (mintState.lastResetTime == 0) {

// @audit cache and use cached value for check in L506 to save 1 storage read
498:    if (redemptionState.lastResetTime == 0) {
```

**Ondo:**
확인됨.


### 0으로의 불필요한 초기화 방지

**설명:** 0으로의 불필요한 초기화를 피하십시오.


### `InvestorBasedRateLimiter::_initializeInvestorState`는 스토리지에서 다시 읽지 않도록 새로 생성된 `investorId`를 반환해야 함

**설명:** `InvestorBasedRateLimiter::_initializeInvestorState`는 새로 생성된 `investorId`를 반환해야 합니다. 그러면 `checkAndUpdateMintLimit` 및 `checkAndUpdateRedemptionLimit` 내에서 사용하여 각 함수에서 1번의 스토리지 읽기를 절약할 수 있습니다.

**Ondo:**
커밋 [192c7ca](https://github.com/ondoprotocol/rwa-internal/commit/192c7ca26e4aeab4c322ef6c4be0f39b5be5d34d)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.


### 새 투자자를 생성할 때 불필요한 작업을 수행하지 않도록 `InvestorBasedRateLimiter::checkAndUpdateMintLimit` 및 `checkAndUpdateRedemptionLimit` 리팩토링

**설명:** `InvestorBasedRateLimiter::checkAndUpdateMintLimit` 및 `checkAndUpdateRedemptionLimit` 내에서 새 투자자를 생성할 때 두 번째 `if` 문 이후에 발생하는 현재 처리의 대부분을 수행할 필요가 없습니다.

**Ondo:**
확인됨.


### `InvestorBasedRateLimiter::_setAddressToInvestorId`에서 먼저 `addressToInvestorId[investorAddress]`를 읽은 다음 `if` 문 확인에서 사용

**설명:** `InvestorBasedRateLimiter::_setAddressToInvestorId`에서 먼저 `addressToInvestorId[investorAddress]`를 읽은 다음 `if` 문 확인에서 사용하여 1번의 스토리지 읽기를 절약하십시오.

**Ondo:**
확인됨.


### `InvestorBasedRateLimiter::_setAddressToInvestorId`에서 가스 환불을 위해 0으로 설정할 때 `delete` 사용

**설명:** `InvestorBasedRateLimiter::_setAddressToInvestorId`에서 0으로 설정할 때 `delete`를 사용하십시오.

**Ondo:**
확인됨.


### 읽히지 않으므로 `rOUSG::_mintShares` 및 `_burnShares`에서 반환 매개변수 제거

**설명:** 읽히지 않으므로 `rOUSG::_mintShares` 및 `_burnShares`에서 반환 매개변수를 제거하십시오. 이렇게 하면 각 함수에서 1번의 스토리지 읽기와 반환 매개변수 비용을 절약할 수 있습니다.

**Ondo:**
커밋 [dc91728](https://github.com/ondoprotocol/rwa-internal/commit/dc91728630a47ba351150287e48547a405a1282e)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.


### `OUSGInstantManager::_mint` 및 `_redeem`에서 `feeReceiver`를 캐시하고 수수료가 공제되는 경우에만 수수료 이벤트 발생

**설명:** `OUSGInstantManager::_mint`에서 `feeReceiver`를 캐시하고 수수료가 공제되는 경우에만 수수료 이벤트를 발생시켜 1번의 스토리지 읽기를 절약하십시오.

**Ondo:**
확인됨.


### `ROUSG::unwrap`을 변경하여 `OUSG` 출력 토큰 양을 반환한 다음 `OUSGInstantManager::redeemRebasingOUSG`에서 `_redeem`을 호출할 때 입력으로 사용

**설명:** `ROUSG::unwrap`을 변경하여 `OUSG` 출력 토큰 양을 반환한 다음 `OUSGInstantManager::redeemRebasingOUSG`에서 `_redeem`을 호출할 때 입력으로 사용하십시오.

**Ondo:**
확인됨.

\clearpage

