**Lead Auditors**

[Dacian](https://x.com/DevDacian)

[Hans](https://x.com/hansfriese)

**Assisting Auditors**



---

# Findings
## Low Risk


### 프로토콜 수수료는 프로토콜에 유리하게 올림 처리되어야 함 (Protocol fee should round up in favor of the protocol)

**설명:** `onMorphoFlashLoan`에서 프로토콜 수수료는 프로토콜에 유리하게 올림 처리되어야 합니다:
```solidity
uint256 rockoFeeBP = ROCKO_FEE_BP;
if (rockoFeeBP > 0) {
    unchecked {
        feeAmount = (flashBorrowAmount * rockoFeeBP) / BASIS_POINTS_DIVISOR;
        borrowAmountWithFee += feeAmount;
    }
}
```

올림 매개변수와 함께 OpenZeppelin [`Math::mulDiv`](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/fda6b85f2c65d146b86d513a604554d15abd6679/contracts/utils/math/Math.sol#L280)를 [사용](https://x.com/DevDacian/status/1892529633104396479)하거나 Solady [`FixedPointMathLib::fullMulDivUp`](https://github.com/Vectorized/solady/blob/c9e079c0ca836dcc52777a1fa7227ef28e3537b3/src/utils/FixedPointMathLib.sol#L548)을 사용하는 것을 고려하세요.

이러한 라이브러리를 사용하는 또 다른 이점은 `flashBorrowAmount * rockoFeeBP`의 곱셈에서 발생하는 중간 오버플로우를 방지할 수 있다는 것입니다.

**Rocko:** 커밋 [a59ba0e](https://github.com/getrocko/onchain/commit/a59ba0e7958c544ad95788ce29923a342a2ea35a)에서 수정됨.

**Cyfrin:** 검증됨.


### `USDT` 부채 토큰에 대해 리파이낸싱이 revert됨 (Refinancing reverts for `USDT` debt token)

**설명:** 프로토콜이 표준 `IERC20::approve` 및 `transfer` 함수를 사용하는 방식 때문에 `USDT` 부채 토큰에 대한 리파이낸싱이 revert됩니다.

**영향:** `USDT` 부채 토큰에 대한 리파이낸싱이 작동하지 않습니다(bricked). 현재 공식적으로 `USDC`만 지원되므로 낮은 심각도로 표시되었습니다. `USDT`의 구현은 체인마다 다릅니다. "있는 그대로(as-is)"의 프로토콜은 Base에서는 `USDT`와 작동하지만 이더리움 메인넷에서는 작동하지 않습니다.

**개념 증명 (Proof of Concept):** 감사의 일환으로 포크 퍼즈 테스팅(fork fuzz testing) 모음을 제공했습니다. 다음 명령을 실행하세요: `forge test --fork-url ETH_RPC_URL --fork-block-number 22000000 --match-test test_FuzzRefinance_AaveToCompound_DepWeth_BorUsdt -vvv`

**권장 완화 방법:** `IERC20::approve`의 모든 사용을 [`SafeERC20::forceApprove`](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/utils/SafeERC20.sol#L101-L108)로 대체하고, L738에서 `IERC20::transfer`를 `SafeERC20::safeTransfer`로 대체한 다음 PoC 테스트를 다시 실행하면 통과합니다.

기존 승인 변경에 대한 프론트 러닝을 방지하기 위한 추가 안전을 위해, 적절한 경우(예: 이전 허용량을 알고 있는 경우 `_revokeTokenSpendApprovals`에서 `safeDecreaseAllowance` 사용) [`SafeERC20::safeIncreaseAllowance`](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/utils/SafeERC20.sol#L68-L71) 및 [`safeDecreaseAllowance`](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/utils/SafeERC20.sol#L82-L90)를 사용하는 것이 이상적입니다.

**Rocko:** 커밋 [751e906](https://github.com/getrocko/onchain/commit/751e906b7c2df6cb587e709b12de25593eb02c75)에서 수정됨.

**Cyfrin:** 검증됨.

\clearpage
## Informational


### 오류 메시지가 `USDC`를 하드코딩하지만 다른 부채 토큰이 사용될 수 있음 (Error messages hardcode `USDC` but other debt tokens may be used)

**설명:** 오류 메시지가 `USDC`를 하드코딩하지만 다른 토큰이 사용될 수 있습니다. 예:
```solidity
function _closeLoanPositionAndReturnCollateralBalance(
    // @audit 부채 토큰은 USDC 외에 다른 토큰일 수 있지만 오류 메시지는 USDC를 하드코딩함
    require(
        debtBalance <= IERC20(debtTokenAddress).balanceOf(FLASH_LOAN_CONTRACT),
        "Insufficient USDC available in the flash contract"
    );
```

`onMorphoFlashLoan`의 이 코드도 부채 토큰이 USDC일 것이라고 가정합니다:
```solidity
uint256 usdcBalance = IERC20(ctx.debtTokenAddress).balanceOf(FLASH_LOAN_CONTRACT);
bool feeAmountAvailable = usdcBalance >= feeAmount;
```

**Rocko:** 커밋 [ec9f5be](https://github.com/getrocko/onchain/commit/ec9f5be20f9249cd20fcc1e173192361ecd97ef5)에서 수정됨.

**Cyfrin:** 검증됨.


### 이벤트에 indexed 매개변수가 누락됨 (Events missing indexed parameters)

**설명:** Solidity의 이벤트는 최대 3개의 indexed 매개변수를 가질 수 있으며, 이는 이벤트 로그의 토픽으로 저장됩니다. Indexed 매개변수를 사용하면 오프체인 서비스에서 이벤트를 효율적으로 필터링하고 검색할 수 있습니다. Indexed 매개변수가 없으면 애플리케이션이 특정 이벤트를 필터링하는 것이 더 어렵고 리소스 집약적이 됩니다.

개선될 수 있는 indexed 매개변수가 누락된 이벤트 사례가 있습니다.
```solidity
    event LogRefinanceLoanCall(
        string logType,
        address rockoWallet,
        string from,
        string to,
        uint256 debtBalance,
        address debtTokenAddress,
        address collateralTokenAddress,
        address aCollateralTokenAddress,
        Id morphoMarketId
    );
    event LogFlashLoanCallback(
        string logType,
        address rockoWallet,
        string from,
        string to,
        address debtTokenAddress,
        address collateralTokenAddress,
        address aCollateralTokenAddress,
        uint256 flashBorrowAmount,
        bytes data,
        Id morphoMarketId
    );
```

**권장 완화 방법:** `rockoWallet`, `debtTokenAddress`, `collateralTokenAddress`와 같이 필터링에 일반적으로 사용되는 중요한 매개변수에 `indexed` 키워드를 추가하세요.

**Rocko:** 커밋 [f5c9c80](https://github.com/getrocko/onchain/commit/f5c9c8051d5ba1bf04774ae6e8aa407aeddbcde1)에서 수정됨.

**Cyfrin:** 검증됨.


### 구성 값이 변경되지 않을 때 불필요한 이벤트 방출 (Unnecessary event emission when configuration values do not change)

**설명:** `RockoFlashRefinance::updateFee`는 `ROCKO_FEE_BP` 변수를 업데이트하고 새로운 수수료 값이 현재 값과 다른지 여부에 관계없이 `FeeUpdated` 이벤트를 방출합니다. 이는 소유자가 이미 설정된 것과 동일한 수수료 값으로 함수를 호출할 때 불필요한 이벤트 방출로 이어집니다.
`pauseContract` 함수도 비슷하게 개선될 수 있습니다.

**Rocko:** 커밋 [99a73dc](https://github.com/getrocko/onchain/commit/99a73dc20fce34811c224fdd46b7e173748bfeb8)에서 수정됨.

**Cyfrin:** 검증됨.


### Morpho에서 담보 잔액을 검색하는 구현 방식이 일관되지 않음 (Inconsistent implementation approach for retrieving collateral balance from Morpho)

**설명:** `RockoFlashRefinance::_collateralBalanceOfMorpho`는 Morpho에서 사용자의 담보 잔액을 검색하기 위해 직접 스토리지 슬롯 액세스를 사용하는 반면, 부채 검색을 위한 유사한 기능은 `MorphoLib`을 사용하여 구현됩니다. 이러한 구현 방식의 불일치는 코드를 덜 읽기 쉽고 유지 관리하기 어렵게 만듭니다.
```solidity
    function _collateralBalanceOfMorpho(
        Id morphoMarketId,
        address rockoWallet
    ) private view returns (uint256 totalCollateralAssets) {//@audit-issue 대신 MorphoLib::collateral 사용
        bytes32[] memory slots = new bytes32[](1);
        slots[0] = MorphoStorageLib.positionBorrowSharesAndCollateralSlot(morphoMarketId, rockoWallet);
        bytes32[] memory values = MORPHO.extSloads(slots);
        totalCollateralAssets = uint256(values[0] >> 128);
    }

    function _getMorphoDebtAndShares(Id marketId, address rockoWallet) private returns (uint256 debt, uint256 shares) {
        MarketParams memory marketParams = MORPHO.idToMarketParams(marketId);
        MORPHO.accrueInterest(marketParams);

        uint256 totalBorrowAssets = MORPHO.totalBorrowAssets(marketId);
        uint256 totalBorrowShares = MORPHO.totalBorrowShares(marketId);
        shares = MORPHO.borrowShares(marketId, rockoWallet);
        debt = shares.toAssetsUp(totalBorrowAssets, totalBorrowShares);
    }
```

**권장 완화 방법:** 코드베이스의 다른 부분과의 일관성을 위해 `MorphoLib::collateral`을 사용하도록 `_collateralBalanceOfMorpho`를 리팩토링하세요:

```diff
function _collateralBalanceOfMorpho(
    Id morphoMarketId,
    address rockoWallet
) private view returns (uint256 totalCollateralAssets) {
-    bytes32[] memory slots = new bytes32[](1);
-    slots[0] = MorphoStorageLib.positionBorrowSharesAndCollateralSlot(morphoMarketId, rockoWallet);
-    bytes32[] memory values = MORPHO.extSloads(slots);
-    totalCollateralAssets = uint256(values[0] >> 128);
+    totalCollateralAssets = MorphoLib.collateral(MORPHO, morphoMarketId, rockoWallet);
}
```

**Rocko:** 커밋 [5ef86b4](https://github.com/getrocko/onchain/commit/5ef86b44063a988afed93fe3f69074be757768bd)에서 수정됨.

**Cyfrin:** 검증됨.


### `onMorphoFlashLoan`에서 데이터 길이 유효성 검사가 불충분함 (Insufficient data length validation in `onMorphoFlashLoan`)

**설명:** `RockoFlashRefinance::onMorphoFlashLoan`은 `data` 매개변수의 길이에 대해 최소 20바이트를 요구하는 기본 확인을 수행합니다. 그러나 전송되는 실제 데이터가 여러 주소, 문자열 및 Id 매개변수를 포함하여 훨씬 더 크기 때문에 이 확인은 불충분합니다. 예상되는 최소 데이터 길이는 최소 256바이트에 동적 문자열 데이터에 대한 추가 바이트를 더한 것이어야 합니다.

**권장 완화 방법:**
```diff
-        require(data.length >= 20, "Invalid data");
+        require(data.length >= 256, "Invalid data");
```

**Rocko:** 커밋 [1da67d7](https://github.com/getrocko/onchain/commit/1da67d7f3ae8076a6cc135ef9a6e5595ad6e29a2)에서 수정됨.

**Cyfrin:** 검증됨.


### `_withdrawAaveCollateral`에서 `aTokenAddress`를 `refinance`의 입력으로 받아 morpho로 전달하고 다시 받는 대신 Aave에서 직접 가져와야 함 (In `_withdrawAaveCollateral` fetch `aTokenAddress` from Aave instead of receiving as input in `refinance` as passing it to morpho and back again)

**설명:** Aave의 `aTokenAddress`는 `_withdrawAaveCollateral`에서 담보를 인출할 때만 필요하지만, 현재는 다음과 같습니다:
* `refinance`에 입력으로 전달됨
* 일부 유효성 검사가 수행됨
* 다른 데이터와 함께 인코딩되어 `Morpho::flashLoan`으로 전송됨
* 그런 다음 Morpho가 `onMorphoFlashLoan`을 호출할 때 다시 전달함
* 여기서 다시 디코딩되어 여기저기 전달됨

이 모든 과정 대신, `aTokenAddress`가 필요한 `_withdrawAaveCollateral` 내부에서 Aave의 API 함수 [`IPool::getReserveData`](https://github.com/aave/aave-v3-core/blob/782f51917056a53a2c228701058a6c3fb233684a/contracts/interfaces/IPool.sol#L582)를 사용하여 올바른 `aTokenAddress`를 가져오세요:
```solidity
    function _withdrawAaveCollateral(
        address collateralAddress,
        uint256 collateralBalance,
        address rockoWallet
    ) private {
        DataTypes.ReserveData memory reserveData = AAVE.getReserveData(collateralAddress);

        // Rocko Wallet needs to send aToken here after debt is paid off
        // Be sure that Rocko Wallet has approved this contract to spend aTokens for > `collateralBalance` tokens
        _pullTokensFromCallerWallet(reserveData.aTokenAddress, rockoWallet, collateralBalance);

        // function withdraw(address asset, uint256 amount, address to)
        AAVE.withdraw(collateralAddress, collateralBalance, FLASH_LOAN_CONTRACT);
    }
```

Aave의 API를 통해 이 매개변수를 가져오면 불필요한 코드/유효성 검사가 제거되고 공격 표면도 줄어듭니다.

**Rocko:** 커밋 [d793f96](https://github.com/getrocko/onchain/commit/d793f960598240fa3eacdc6a4ec67d55dcfa2a75)에서 수정됨.

**Cyfrin:** 검증됨.


### 사용자가 모든 승인을 취소할 수 있는 방법 제공 (Provide a way for users to revoke all approvals)

**설명:** `RockoFlashRefinance`는 기존 대출 포지션을 한 대출 프로토콜에서 다른 대출 프로토콜로 이동하도록 설계되었습니다. 계약은 사용자를 대신하여 다음을 수행할 수 있어야 합니다:
* 이전 대출 제공자로부터 대출 종료
* 새로운 대출 제공자에게 새로운 대출 개설

Aave의 경우, 사용자는 리파이낸싱 계약이 포지션을 닫기 위해 [AToken](https://github.com/aave/aave-v3-core/blob/master/contracts/protocol/tokenization/AToken.sol)을 사용하고 새로운 포지션을 열기 위해 [VariableDebtToken](https://github.com/aave/aave-v3-core/blob/master/contracts/protocol/tokenization/VariableDebtToken.sol)을 사용할 수 있도록 허용해야 합니다.

Compound(Comet)의 경우, 사용자는 [allow 함수](https://github.com/compound-finance/comet/blob/68cd639c67626c86e890e5aac775ad4b6405d923/contracts/CometExt.sol#L162C14-L162C19)를 호출하여 리파이낸싱 계약을 허용해야 합니다.

Morpho의 경우, 사용자는 [setAuthorization](https://github.com/morpho-org/morpho-blue/blob/9e2b0755b47bbe5b09bf1be8f00e060d4eab6f1c/src/Morpho.sol#L437C14-L437C30) 함수를 호출하여 리파이낸싱 프로토콜을 승인해야 합니다.

프로토콜 팀은 이러한 승인과 관련된 프론트엔드 소스를 제공했으며 "승인" 지원만 있고 취소는 없었습니다. 사용자가 이러한 모든 승인을 쉽게 취소할 수 있는 방법을 제공하는 것이 좋습니다.

**Rocko:** Rocko 앱에서 호출할 때 사용자 취소가 배치 트랜잭션에 포함될 것입니다.


### `AAVE_DATA_PROVIDER` 업데이트 허용 고려 (Consider allowing update to `AAVE_DATA_PROVIDER`)

**설명:** `RockoFlashRefinance::AAVE_DATA_PROVIDER`는 `AaveProtocolDataProvider`의 주소를 불변(immutable)으로 저장합니다. 그러나 `AaveProtocolDataProvider`는 업그레이드할 수 없으며 메인넷의 "현재" 것은 43일 전에 주소 0x497a1994c46d4f6C864904A9f1fac6328Cb7C8a6에 배포되었습니다.

따라서 `AAVE_DATA_PROVIDER`가 `immutable`이 아니어야 하는지, 그리고 나중에 업데이트할 수 있도록 `onlyOwner` 함수가 존재해야 하는지 고려하세요.

`RockoFlashRefinance`에는 관련된 내부 상태가 없으므로 다시 배포할 수 있습니다. 트레이드오프는 `immutable` `AAVE_DATA_PROVIDER`를 사용하면 관련된 사용자 트랜잭션의 가스 비용이 약간 줄어들지만 업데이트하려면 계약을 다시 배포해야 한다는 것입니다.

**Rocko:** 인지함; 더 낮은 사용자 가스 비용을 위해 현재 설정을 선호합니다.

\clearpage
## Gas Optimization


### `ReentrancyGuard` 대신 `ReentrancyGuardTransient` 또는 더 가스 효율적인 `nonReentrant` modifier 사용 (Use `ReentrancyGuardTransient` instead of `ReentrancyGuard` or more gas-efficient `nonReentrant` modifiers)

**설명:** `ReentrancyGuard` 대신 더 가스 효율적인 `nonReentrant` modifier를 위해 [`ReentrancyGuardTransient`](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/ReentrancyGuardTransient.sol)를 사용하세요. OpenZeppelin 버전을 5.1로 올려야 합니다.

**Rocko:** 커밋 [675f4b2](https://github.com/getrocko/onchain/commit/675f4b2e59cddf6c3f9f2e866f4401564bd0a006)에서 수정됨.

**Cyfrin:** 검증됨.


### `updateFee`에서 구식 확인 제거 (Remove obsolete check in `updateFee`)

**설명:** `updateFee`에서 구식 확인을 제거하세요:
```diff
-        require(newFee >= 0, "Fee must not be negative");
```

`newFee`가 `uint256`으로 선언되어 음수가 될 수 없으므로 이 확인은 불필요합니다.

**Rocko:** 커밋 [2c50838](https://github.com/getrocko/onchain/commit/2c50838a5cc5e498a14874a5d3348da20087e6bc)에서 수정됨.

**Cyfrin:** 검증됨.


### `onlyOwner` 함수 내부에서 `owner()` 대신 `msg.sender` 사용 (Use `msg.sender` instead of `owner()` inside `onlyOwner` functions)

**설명:** `onlyOwner` 함수 내부에서 `owner()` 대신 `msg.sender`를 사용하면 스토리지 읽기가 제거되어 더 효율적입니다. `onlyOwner` modifier가 `msg.sender`가 소유자임을 보장하므로 안전합니다:
```solidity
757:        IERC20(tokenAddress).safeTransfer(owner(), amount);
766:        (bool success, ) = owner().call{ value: amount }("");
```

**Rocko:** 커밋 [751e906](https://github.com/getrocko/onchain/commit/751e906b7c2df6cb587e709b12de25593eb02c75)에서 수정됨.

**Cyfrin:** 검증됨.


### 동일한 문자열의 반복적인 해싱 방지 (Prevent repetitive hashing of identical strings)

**설명:** `RockoFlashRefinance::_compareStrings`는 동일한 값으로 자주 호출되어 중복된 불필요한 작업이 발생합니다. 이를 방지하는 간단하고 효율적인 방법은 `from`/`to` 입력 모두에 대해 `_parseProtocol`을 사용하여 먼저 변환한 다음 `refinance` 및 `_revokeTokenSpendApprovals`와 같은 함수에서 필요에 따라 열거형을 비교하는 것입니다.

문자열 비교가 필요한 경우:
* "aave", "morpho", "compound"와 같은 일반적인 예상 문자열에 대해 해시 결과를 `bytes32` 상수로 하드코딩하고 `_parseProtocol` 및 기타 함수 내에서 이러한 하드코딩된 상수를 사용합니다.
* `RockoFlashRefinance::refinance`와 같은 함수에서 `from`/`to` 입력의 해시를 로컬 `bytes32` 변수에 캐시하고 비교에 캐시된 해시와 하드코딩된 상수를 사용합니다.

이를 달성하는 간단한 방법 중 하나는 다음과 같습니다:
* 문자열의 해시를 반환하는 함수 정의:
```solidity
    function _hashString(string calldata input) private pure returns (bytes32 output) {
        output = keccak256(bytes(input));
    }
```
* 두 개의 `bytes32`를 입력으로 받도록 `_compareStrings` 변경:
```solidity
    function _compareStrings(bytes32 a, bytes32 b) private pure returns (bool) {
        return a == b;
    }
```

OpenZeppelin의 문자열 동등성 [구현](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/Strings.sol#L134-L136)도 고려하세요.

**Rocko:** 커밋 [a59ba0e](https://github.com/getrocko/onchain/commit/a59ba0e7958c544ad95788ce29923a342a2ea35a)에서 수정됨.

**Cyfrin:** 검증됨.


### 기본값으로 초기화하지 말 것 (Don't initialize to default values)

**설명:** Solidity가 이미 수행하므로 기본값으로 초기화하지 마세요:
```solidity
78:        ROCKO_FEE_BP = 0;
597:        uint256 debtBalance = 0;
598:        uint256 morphoDebtShares = 0;
```

**Rocko:** 커밋 [751e906](https://github.com/getrocko/onchain/commit/751e906b7c2df6cb587e709b12de25593eb02c75)에서 수정됨.

**Cyfrin:** 검증됨.


### 중복 지역 변수 및 `return` 문을 제거하기 위해 명명된 반환 변수 사용 (Use named return variables to eliminate redundant local variables and `return` statements)

**설명:** 중복 지역 변수 및 `return` 문을 제거하기 위해 명명된 반환 변수를 사용하세요:
```diff
// _closeLoanPositionAndReturnCollateralBalance L457
-    ) private returns (uint256) {
+    ) private returns (uint256 collateralBalance) {

// L464
-        uint256 collateralBalance;

// L480
-        return collateralBalance;
```

`_collateralBalanceOfAave`, `_getDebtBalanceOfAave`에도 동일한 아이디어를 적용할 수 있습니다.

**Rocko:** 커밋 [751e906](https://github.com/getrocko/onchain/commit/751e906b7c2df6cb587e709b12de25593eb02c75)에서 수정됨.

**Cyfrin:** 검증됨.


### 중복된 `onBehalfOf` 변수 제거 (Remove redundant `onBehalfOf` variables)

**설명:** 중복된 `onBehalfOf` 변수를 제거하세요:
```diff
    function _supplyToAave(address collateralAddress, uint256 collateralBalance, address rockoWallet) private {
-       address onBehalfOf = rockoWallet;
-       AAVE.supply(collateralAddress, collateralBalance, onBehalfOf, AAVE_REFERRAL_CODE);
+       AAVE.supply(collateralAddress, collateralBalance, rockoWallet, AAVE_REFERRAL_CODE);
    }
    function _borrowFromAave(address rockoWallet, address token, uint256 borrowAmount) private {
-       address onBehalfOf = rockoWallet;
-       AAVE.borrow(token, borrowAmount, AAVE_INTERESTE_RATE_MODE, AAVE_REFERRAL_CODE, onBehalfOf);
+       AAVE.borrow(token, borrowAmount, AAVE_INTERESTE_RATE_MODE, AAVE_REFERRAL_CODE, rockoWallet);
    }
```

**Rocko:** 커밋 [751e906](https://github.com/getrocko/onchain/commit/751e906b7c2df6cb587e709b12de25593eb02c75)에서 수정됨.

**Cyfrin:** 검증됨.


### `_closeLoanMorphoWithShares` 및 `_openLoanPosition`에서 중복된 `morphoMarketId` 유효성 검사 제거 (Remove redundant `morphoMarketId` validation checks in `_closeLoanMorphoWithShares` and `_openLoanPosition`)

**설명:** `RockoFlashRefinance::_closeLoanMorphoWithShares` 및 `_openLoanPosition`에는 중복된 `morphoMarketId` 유효성 검사가 포함되어 있습니다. 이 유효성 검사가 중복되는 이유는 다음과 같습니다:

* `RockoFlashRefinance::refinance`는 이미 입력 `morphoMarketId`를 검증하고 `data` 페이로드에 인코딩한 다음 `data` 페이로드와 함께 `Morpho::flashLoan`을 호출합니다:
```solidity
if (_compareStrings(to, "morpho") || _compareStrings(from, "morpho")) {
    require(_isValidId(morphoMarketId), "Morpho Market ID required for Morpho refinance");
}

bytes memory data = abi.encode(
    rockoWallet,
    from,
    to,
    debtTokenAddress,
    collateralTokenAddress,
    aCollateralTokenAddress,
    morphoMarketId,
    morphoDebtShares
);

MORPHO.flashLoan(debtTokenAddress, debtBalance, data);
```

* `Morpho::flashLoan`은 항상 수정되지 않은 `data` 페이로드를 `RockoFlashRefinance::onMorphoFlashLoan`에 전달합니다:
```solidity
function flashLoan(address token, uint256 assets, bytes calldata data) external {
    require(assets != 0, ErrorsLib.ZERO_ASSETS);

    emit EventsLib.FlashLoan(msg.sender, token, assets);

    IERC20(token).safeTransfer(msg.sender, assets);

    // @audit 수정되지 않은 `data` 페이로드를 `onMorphoFlashLoan`에 전달
    IMorphoFlashLoanCallback(msg.sender).onMorphoFlashLoan(assets, data);

    IERC20(token).safeTransferFrom(msg.sender, address(this), assets);
}
```

* `RockoFlashRefinance::onMorphoFlashLoan`은 수정되지 않은 `data` 페이로드를 디코딩하고 `RockoFlashRefinance::refinance`에서 이미 검증된 디코딩된 `morphoMarketId`를 사용하여 `_closeLoanMorphoWithShares` 및 `_openLoanPosition`을 호출합니다.

**권장 완화 방법:** 다음 위치에서 중복된 `morphoMarketId` 유효성 검사를 제거하세요:
```solidity
325: require(_isValidId(morphoMarketId), "Invalid Morpho Market ID");

503: require(_isValidId(morphoMarketId), "Morpho Market ID required for Morpho refinance");
```

**Rocko:** 커밋 [5a9aa7d](https://github.com/getrocko/onchain/commit/5a9aa7d3cfb80150448608854440c285ea08fa53)에서 수정됨.

**Cyfrin:** 검증됨.


### `_openLoanMorpho`에서 중복된 담보 잔액 확인 (Redundant collateral balance check in `_openLoanMorpho`)

**설명:** `RockoFlashRefinance::_openLoanMorpho`에는 담보 잔액 가용성에 대한 중복 확인이 포함되어 있습니다. 이 함수는 플래시 론 계약에 충분한 담보 잔액이 있는지 확인하지만, 이 확인은 이미 호출하는 `_openLoanPosition` 함수에서 수행됩니다.

**권장 완화 방법:**
```diff
    function _openLoanMorpho(
        Id morphoMarketId,
        uint256 borrowAmount,
        address collateralAddress,
        uint256 collateralBalance,
        address rockoWallet
    ) private {
        _checkAllowanceAndApproveContract(address(MORPHO), collateralAddress, collateralBalance);
        MarketParams memory marketParams = MORPHO.idToMarketParams(morphoMarketId);
-       uint256 flashLoanContractBalance = IERC20(collateralAddress).balanceOf(FLASH_LOAN_CONTRACT);
-       // emit LogBalance("Flash Loan Contract Balance", flashLoanContractBalance);
-       require(
-           flashLoanContractBalance >= collateralBalance,
-           "Insufficient collateral available in the flash contract"
-       );
```

**Rocko:** 커밋 [a8efb43](https://github.com/getrocko/onchain/commit/a8efb43b10673508e1fd184aec23a5373f73cd5d)에서 수정됨.

**Cyfrin:** 검증됨.

\clearpage
