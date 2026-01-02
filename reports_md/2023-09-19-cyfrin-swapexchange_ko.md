**수석 감사자 (Lead Auditors)**

[Hans](https://twitter.com/hansfriese)

**보조 감사자 (Assisting Auditors)**

[0kage](https://twitter.com/0kage_eth)


---

# 발견 사항 (Findings)
## 중간 중요도 (Medium Risk)

### ERC20 토큰에 대해 안전한 전송 사용 (Use safe transfer for ERC20 tokens)

**심각도:** 중간 (Medium)

**설명:** 프로토콜은 모든 ERC20 토큰을 지원하려고 하지만 구현에서는 원래의 transfer 함수를 사용합니다.
일부 토큰(예: USDT)은 EIP20 표준을 올바르게 구현하지 않으며 transfer/transferFrom 함수가 성공 불리언 대신 void를 반환합니다. 올바른 EIP20 함수 서명으로 이러한 함수를 호출하면 되돌려집니다(revert).

```solidity
TransferUtils.sol
34:     function _transferERC20(address token, address to, uint256 amount) internal {
35:         IERC20 erc20 = IERC20(token);
36:         require(erc20 != IERC20(address(0)), "Token Address is not an ERC20");
37:         uint256 initialBalance = erc20.balanceOf(to);
38:         require(erc20.transfer(to, amount), "ERC20 Transfer failed");//@audit-issue will revert for USDT
39:         uint256 balance = erc20.balanceOf(to);
40:         require(balance >= (initialBalance + amount), "ERC20 Balance check failed");
41:     }
```

**파급력:** USDT와 같이 EIP20을 올바르게 구현하지 않는 토큰은 반환 값이 누락되어 트랜잭션을 되돌리기 때문에 프로토콜에서 사용할 수 없습니다.

**권장 완화 방안:** 반환 값 확인을 처리하고 비표준 준수 토큰도 처리하는 safeTransfer 및 safeTransferFrom 함수가 있는 OpenZeppelin의 SafeERC20 버전을 사용하는 것이 좋습니다.

**프로토콜:** [564f711](https://github.com/SwapExchangeio/Contracts/commit/564f711c6f915f5a7696739266a1f8059ee9a172) 커밋에서 수정되었습니다.

**Cyfrin:** 검증됨.

### 전송 수수료 토큰이 지원되지 않음 (Fee-on-transfer tokens are not supported)

**심각도:** 중간 (Medium)

**설명:** 프로토콜은 모든 ERC20 토큰을 지원하려고 하지만 전송 수수료 토큰(fee-on-transfer tokens)은 지원하지 않습니다.
프로토콜은 `TransferUtils::_transferERC20()` 및 `TransferUtils::_transferFromERC20()` 함수를 사용하여 ERC20 토큰을 전송합니다.

```solidity
TransferUtils.sol
51:     function _transferERC20(address token, address to, uint256 amount) internal {
52:         IERC20 erc20 = IERC20(token);
53:         require(erc20 != IERC20(address(0)), "Token Address is not an ERC20");
54:         uint256 initialBalance = erc20.balanceOf(to);
55:         require(erc20.transfer(to, amount), "ERC20 Transfer failed");
56:         uint256 balance = erc20.balanceOf(to);
57:         require(balance >= (initialBalance + amount), "ERC20 Balance check failed");//@audit-issue reverts for fee on transfer token
58:     }
```

구현은 수신자의 잔액이 초기 잔액에 전송된 금액을 더한 것보다 크거나 같은지 확인하여 전송이 성공했는지 검증합니다. 전송 수수료 토큰의 경우 실제 수신 금액이 입력 금액보다 적기 때문에 이 확인은 실패합니다. (전송 수수료 토큰에 대해서는 [여기](https://github.com/d-xo/weird-erc20#fee-on-transfer)를 읽어보세요)

전송 수수료 토큰은 매우 적지만, 이러한 이상한 ERC20 토큰을 지원하지 않는다면 프로토콜이 모든 ERC20 토큰을 지원한다고 말할 수 없습니다.

**파급력:** 전송 수수료 토큰은 프로토콜에 사용할 수 없습니다.
이러한 토큰의 희귀성을 감안하여 이 발견 사항을 중간 위험으로 평가합니다.

**권장 완화 방안:** 전송 유틸리티 함수를 업데이트하여 실제로 수신된 금액을 반환하도록 할 수 있습니다.
또는 표준 ERC20 토큰만 지원됨을 명확하게 문서화하세요.

**프로토콜:** 이 단계에서는 이를 구현하지 않기로 결정했습니다.

**Cyfrin:** 인정함. 권장된 대로 사용자 문서에 이를 언급하십시오.

### 중앙화 위험 (Centralization risk)

**심각도:** 중간 (Medium)

**설명:** 프로토콜에는 사용자에게 영향을 미칠 수 있는 관리 작업을 수행할 수 있는 권한을 가진 소유자가 있습니다.
특히 소유자는 수수료 설정 및 보상 처리자(reward handler) 주소를 변경할 수 있습니다.

1. 관리자 수수료 설정자(setter) 함수에 대한 검증이 누락되었습니다.

```solidity
FeeData.sol
31:     function setFeeValue(uint256 feeValue) external onlyOwner {
32:         require(feeValue < _feeDenominator, "Fee percentage must be less than 1");
33:         _feeValue = feeValue;
34:     }
90:
91:43:
92:44:     function setFixedFee(uint256 fixedFee) external onlyOwner {//@audit-issue validate min/max
93:45:         _fixedFee = fixedFee;
94:46:     }
```

2. 관리자가 시작한 중요한 변경 사항은 이벤트를 통해 기록되어야 합니다.

```solidity
File: helpers/FeeData.sol

31:     function setFeeValue(uint256 feeValue) external onlyOwner {

36:     function setMaxHops(uint256 maxHops) external onlyOwner {

40:     function setMaxSwaps(uint256 maxSwaps) external onlyOwner {

44:     function setFixedFee(uint256 fixedFee) external onlyOwner {

48:     function setFeeToken(address feeTokenAddress) public onlyOwner {

53:     function setFeeTokens(address[] memory feeTokenAddresses) public onlyOwner {

60:     function clearFeeTokens() public onlyOwner {

```

```solidity
File: helpers/TransferHelper.sol

86:     function setRewardHandler(address rewardAddress) external onlyOwner {

92:     function setRewardsActive(bool _rewardsActive) external onlyOwner {

```

**파급력:** 프로토콜 소유자는 신뢰할 수 있는 당사자로 간주되지만, 소유자는 검증이나 로깅 없이 수수료 설정 및 보상 처리자 주소를 변경할 수 있습니다. 이는 예상치 못한 결과로 이어질 수 있으며 사용자에게 영향을 줄 수 있습니다.

**권장 완화 방안:**
- 문서에 소유자의 권한과 책임을 명시하십시오.
- 수수료 설정의 최소값 및 최대값으로 사용할 수 있는 상수 상태 변수를 추가하십시오.
- 관리자 함수에 적절한 검증을 추가하십시오.
- 중요한 상태 변수의 변경 사항을 이벤트를 통해 기록하십시오.

**프로토콜:**

- setFeeNumerator 변경 사항이 [f8f07c5](https://github.com/SwapExchangeio/Contracts/commit/f8f07c5e72c052d11c1b4c4dfd8b849a99694e03) 커밋에서 수정됨
- setFeeNumerator 최대값이 [7874a8f](https://github.com/SwapExchangeio/Contracts/commit/7874a8f677b1e06e9b3e0289a91ab33e46806ff2) 커밋에서 감소됨
- setFixedFee 변경 사항이 [af760b4](https://github.com/SwapExchangeio/Contracts/commit/af760b484ff6da0ad9a8492a96a5e7ef19056df1) 커밋에서 수정됨
- setFeeToken/clearFeeToken/setFeeTokens에 대한 이벤트 및 배포 시 힙(heap)의 이벤트 방출을 저장하기 위한 초기화 설정자(initialization setter) 분리가 [927c102](https://github.com/SwapExchangeio/Contracts/commit/[927c1020a71c017d3a17e653be84996ba86ff8ed]) 커밋에서 수정됨
- 배열 인수 유형을 memory가 아닌 calldata로 변경하는 것이 [e615cb2](https://github.com/SwapExchangeio/Contracts/commit/e615cb21ac67d8a31656c622d3b2e3ddc1dc8891) 커밋에서 수정됨
- RewardHandler 및 RewardsActive에 대한 이벤트가 [3068a2e](https://github.com/SwapExchangeio/Contracts/commit/3068a2e88621f991a54be80ef16d868e4fb10d25) 커밋에서 수정됨
- MaxHops 및 MaxSwaps에 대한 이벤트가 [077577b](https://github.com/SwapExchangeio/Contracts/commit/077577ba26b66cbb4b977b3738b59eb727c5d87d) 커밋에서 수정됨.

**Cyfrin:** 검증됨.

## 낮은 중요도 (Low Risk)

### `SwapExchange::calculateMultiSwap()`에서 tokenA에 대한 검증 누락 (Validation is missing for tokenA in `SwapExchange::calculateMultiSwap()`)

**심각도:** 낮음 (Low)

**설명:** 프로토콜은 스왑 체인 청구를 지원하며 `SwapExchange::calculateMultiSwap()` 함수는 주어진 tokenB 양에 대해 받을 수 있는 tokenA 양을 포함한 일부 계산을 수행하는 데 사용됩니다.
구현을 살펴보면 프로토콜은 체인의 마지막 스왑의 tokenA가 실제로 `multiClaimInput`의 tokenA와 동일한지 검증하지 않습니다.
이 뷰 함수는 프론트엔드에서 `MultiSwap` 결과를 '미리 보기' 위해 사용되기 때문에 직접적인 보안 위험을 내포하지는 않지만 예상치 못한 결과를 초래할 수 있습니다. (실제 스왑 함수 `SwapExchange::_claimMultiSwap()`은 적절한 검증을 구현했다는 점에 주목할 필요가 있습니다.)

```solidity
SwapExchange.sol
150:     function calculateMultiSwap(SwapUtils.MultiClaimInput calldata multiClaimInput) external view returns (SwapUtils.SwapCalculation memory) {
151:         uint256 swapIdCount = multiClaimInput.swapIds.length;
152:         if (swapIdCount == 0 || swapIdCount > _maxHops) revert Errors.InvalidMultiClaimSwapCount(_maxHops, swapIdCount);
153:         if (swapIdCount == 1) {
154:             SwapUtils.Swap memory swap = swaps[multiClaimInput.swapIds[0]];
155:             return SwapUtils._calculateSwapNetB(swap, multiClaimInput.amountB, _feeValue, _feeDenominator, _fixedFee);
156:         }
157:         uint256 matchAmount = multiClaimInput.amountB;
158:         address matchToken = multiClaimInput.tokenB;
159:         uint256 swapId;
160:         bool complete = true;
161:         for (uint256 i = 0; i < swapIdCount; i++) {
162:             swapId = multiClaimInput.swapIds[i];
163:             SwapUtils.Swap memory swap = swaps[swapId];
164:             if (swap.tokenB != matchToken) revert Errors.NonMatchingToken();
165:             if (swap.amountB < matchAmount) revert Errors.NonMatchingAmount();
166:             if (matchAmount < swap.amountB) {
167:                 if (!swap.isPartial) revert Errors.NotPartialSwap();
168:                 matchAmount = MathUtils._mulDiv(swap.amountA, matchAmount, swap.amountB);
169:                 complete = complete && false;
170:             }
171:             else {
172:                 matchAmount = swap.amountA;
173:             }
174:             matchToken = swap.tokenA;
175:         }
176:         (uint8 feeType,) = _calculateFeeType(multiClaimInput.tokenA, multiClaimInput.tokenB);//@audit-issue no validation matchToken == multiClaimInput.tokenA
177:         uint256 fee = FeeUtils._calculateFees(matchAmount, multiClaimInput.amountB, feeType, swapIdCount, _feeValue, _feeDenominator, _fixedFee);
178:         SwapUtils.SwapCalculation memory calculation;
179:         calculation.amountA = matchAmount;
180:         calculation.amountB = multiClaimInput.amountB;
181:         calculation.fee = fee;
182:         calculation.feeType = feeType;
183:         calculation.isTokenBNative = multiClaimInput.tokenB == Constants.NATIVE_ADDRESS;
184:         calculation.isComplete = complete;
185:         calculation.nativeSendAmount = SwapUtils._calculateNativeSendAmount(calculation.amountB, calculation.fee, calculation.feeType, calculation.isTokenBNative);
186:         return calculation;
187:     }
```

**파급력:** 체인의 마지막 스왑이 `multiClaimInput`의 tokenA와 다른 tokenA를 갖는 경우 함수는 잘못된 스왑 계산 결과를 반환하며 이는 예상치 못한 결과를 초래할 수 있습니다.

**권장 완화 방안:** 체인의 마지막 스왑의 tokenA가 `multiClaimInput`의 tokenA와 동일하다는 검증을 추가하십시오.

**프로토콜:** [d3c758e](https://github.com/SwapExchangeio/Contracts/commit/d3c758e6c08f6be75bd420ffd8bf4de71a407897) 커밋에서 수정되었습니다.

**Cyfrin:** 검증됨.

## 정보성 발견 사항 (Informational Findings)

### 적절하지 않은 변수 명명 (Not proper variable naming)

**심각도:** 정보성 (Informational)

**설명:** `FeeData` 컨트랙트에는 수수료를 계산하는 데 사용되는 내부 변수 `_feeValue`가 있습니다.
이 변수의 사용 전반에 걸쳐 수수료 백분율을 계산하는 동안 분자로 사용됩니다.
혼동을 피하기 위해 이 변수의 이름을 `feeNumerator`로 변경하는 것이 좋습니다.

```solidity
FeeUtils.sol
16:     function _calculateFees(uint256 amountA, uint256 amountB, uint8 feeType,  uint256 hops, uint256 feeValue, uint256 feeDenominator, uint256 fixedFee)
17:     internal pure returns (uint256) {
18:         if (feeType == Constants.FEE_TYPE_TOKEN_B) {
19:             return MathUtils._mulDiv(amountB, feeValue, feeDenominator) * hops;
20:         }
21:         if (feeType == Constants.FEE_TYPE_TOKEN_A) {
22:             return MathUtils._mulDiv(amountA, feeValue, feeDenominator) * hops;
23:         }
24:         if (feeType == Constants.FEE_TYPE_ETH_FIXED) {
25:             return fixedFee * hops;
26:         }
27:         revert Errors.UnknownFeeType(feeType);
28:     }
```

**프로토콜:** [f6154c9](https://github.com/SwapExchangeio/Contracts/commit/f6154c99edabe7b62d956935a94567c88ee89b3d) 커밋에서 수정되었습니다.

**Cyfrin:** 검증됨.

### 불필요한 논리 연산 (Unnecessary logical operation)

**심각도:** 정보성 (Informational)

**설명:** `SwapExchange::calculateMultiSwap()` 함수에는 for 루프 내에 필요하지 않은 논리 연산이 있습니다.

```solidity
SwapExchange.sol
161:         for (uint256 i = 0; i < swapIdCount; i++) {
162:             swapId = multiClaimInput.swapIds[i];
163:             SwapUtils.Swap memory swap = swaps[swapId];
164:             if (swap.tokenB != matchToken) revert Errors.NonMatchingToken();
165:             if (swap.amountB < matchAmount) revert Errors.NonMatchingAmount();
166:             if (matchAmount < swap.amountB) {
167:                 if (!swap.isPartial) revert Errors.NotPartialSwap();
168:                 matchAmount = MathUtils._mulDiv(swap.amountA, matchAmount, swap.amountB);
169:                 complete = complete && false;//@audit-issue INFO unnecessary operation, just set complete=false
170:             }
171:             else {
172:                 matchAmount = swap.amountA;
173:             }
174:             matchToken = swap.tokenA;
175:         }
```

**프로토콜:** [a079c11](https://github.com/SwapExchangeio/Contracts/commit/a079c11cc3bc044c61493040dab1f94de4a0f14a) 커밋에서 수정되었습니다.

**Cyfrin:** 검증됨.

## 가스 최적화 (Gas Optimizations)

### 스토리지에 bool을 사용하면 오버헤드 발생 (Using bools for storage incurs overhead)

true/false에 대해 uint256(1) 및 uint256(2)를 사용하여 Gwarmaccess(100 가스)를 피하고, 과거에 `true`였던 후 `false`에서 `true`로 변경할 때 Gsset(20000 가스)을 피하십시오. 자세한 내용은 [여기](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/58f635312aa21f947cae5f8578638a85aa2519f5/contracts/security/ReentrancyGuard.sol#L23-L27)에서 확인하세요.

```solidity
File: helpers/FeeData.sol

18:     mapping(address => bool) public feeTokenMap;

```

```solidity
File: helpers/TransferHelper.sol

16:     bool public rewardsActive;

```

**프로토콜:** [2d48a4b](https://github.com/SwapExchangeio/Contracts/commit/2d48a4b7c371054207866f90b2cd98c98fb34d5a), [84dbd3a](https://github.com/SwapExchangeio/Contracts/commit/84dbd3aafa28ddad88ab1406d0946525f910a261) 커밋에서 수정되었습니다.

**Cyfrin:** 검증됨.

### 루프 외부에서 배열 길이 캐시 (Cache array length outside of loop)

캐시되지 않으면 solidity 컴파일러는 각 반복 중에 항상 배열의 길이를 읽습니다. 즉, 스토리지 배열인 경우 이는 추가 sload 작업(첫 번째를 제외한 각 반복에 대해 100 추가 가스)이고 메모리 배열인 경우 추가 mload 작업(첫 번째를 제외한 각 반복에 대해 3 추가 가스)입니다.

```solidity
File: SwapExchange.sol

403:         for (uint i = 0; i < swapIds.length; i++) {
```

```solidity
File: helpers/FeeData.sol

55:         for (uint i = 0; i < feeTokenAddresses.length; i++) {
61:         for (uint i = 0; i < feeTokenKeys.length; i++) {

```

**프로토콜:** [9ba2a93](https://github.com/SwapExchangeio/Contracts/commit/9ba2a93bdad911f541056ced4661bb2fce8db8e0) 커밋에서 수정되었습니다.

**Cyfrin:** 검증됨.

### 오버플로우가 발생하지 않는 연산의 경우 unchecked 사용 가능 (For Operations that will not overflow, you could use unchecked)

```solidity
File: SwapExchange.sol

209:    ++recordCount;

254:    totalNativeSendAmount += calculation.nativeSendAmount;

407:    total += swap.amountA;

```

```solidity
File: libraries/SwapUtils.sol

93:             uint256 expectedValue = amount + fee;

```

**프로토콜:** [f7a5dac](https://github.com/SwapExchangeio/Contracts/commit/f7a5dac3141f71b06a6d4ad3e44d657bc55ee441) 커밋에서 수정되었습니다.

**Cyfrin:** 검증됨.

### 사용자 정의 오류 사용 (Use Custom Errors)

오류 문자열을 사용하는 대신 배포 및 런타임 비용을 줄이려면 사용자 정의 오류(Custom Errors)를 사용해야 합니다. 이렇게 하면 배포 및 런타임 비용을 모두 절약할 수 있습니다. [여기](https://blog.soliditylang.org/2021/04/21/custom-errors/)에서 자세히 읽어보세요.

```solidity
File: helpers/FeeData.sol

32:         require(feeValue < _feeDenominator, "Fee percentage must be less than 1");

```

```solidity
File: helpers/TransferHelper.sol

20:         require(rewardAddress != address(0), "Reward Address is Invalid");

87:         require(rewardAddress != address(0), "Reward Address is Invalid");

98:         require(rewardHandler.logTokenFee(token, fee), "LogTokenFee failed");

103:         require(rewardHandler.logNativeFee(fee), "LogTNativeFee failed");

```

```solidity
File: libraries/TransferUtils.sol

36:         require(erc20 != IERC20(address(0)), "Token Address is not an ERC20");

38:         require(erc20.transfer(to, amount), "ERC20 Transfer failed");

40:         require(balance >= (initialBalance + amount), "ERC20 Balance check failed");

45:         require(erc20 != IERC20(address(0)), "Token Address is not an ERC20");

47:         require(erc20.transferFrom(from, to, amount), "ERC20 Transfer failed");

49:         require(balance >= (initialBalance + amount), "ERC20 Balance check failed");

54:         require(flag == true, "ETH transfer failed");

```

**프로토콜:** [ffe50aa](https://github.com/SwapExchangeio/Contracts/commit/ffe50aa913d373724ec12bc704387dfaefaab7a5) 및 [9ba796f](https://github.com/SwapExchangeio/Contracts/commit/9ba796f6ff0161376b254d80db55d3bc33414a30) 커밋에서 수정되었습니다.

**Cyfrin:** 검증됨.

### 기본값으로 변수 초기화하지 않기 (Don't initialize variables with default value)

```solidity
File: SwapExchange.sol

130:         for (uint256 i = 0; i < length; i++) {

161:         for (uint256 i = 0; i < swapIdCount; i++) {

247:         for (uint256 i = 0; i < length; i++) {

346:         for (uint256 i = 0; i < swapIdCount; i++) {

403:         for (uint i = 0; i < swapIds.length; i++) {

```

```solidity
File: helpers/FeeData.sol

55:         for (uint i = 0; i < feeTokenAddresses.length; i++) {

61:         for (uint i = 0; i < feeTokenKeys.length; i++) {

```

```solidity
File: libraries/Constants.sol

6:     uint8 public constant FEE_TYPE_ETH_FIXED = 0;

```

**프로토콜:** 83ffebcc099da256ec53644afc733eab2586c636 커밋에서 for 루프에 대해 수정되었습니다.

**Cyfrin:** 검증됨.

### `i++`보다 `++i`가 가스 비용이 적음 (`++i` costs less gas than `i++`)

특히 `for`-루프에서 사용될 때 그렇습니다 (`--i`/`i--` 도 마찬가지).

```solidity
File: SwapExchange.sol

130:         for (uint256 i = 0; i < length; i++) {

161:         for (uint256 i = 0; i < swapIdCount; i++) {

247:         for (uint256 i = 0; i < length; i++) {

346:         for (uint256 i = 0; i < swapIdCount; i++) {

403:         for (uint i = 0; i < swapIds.length; i++) {

```

```solidity
File: helpers/FeeData.sol

55:         for (uint i = 0; i < feeTokenAddresses.length; i++) {

61:         for (uint i = 0; i < feeTokenKeys.length; i++) {

```

**프로토콜:** [5662786](https://github.com/SwapExchangeio/Contracts/commit/5662786421251a32444427d4ede74100a4c772f0) 커밋에서 수정되었습니다.

**Cyfrin:** 검증됨.

### 부호 없는 정수 비교에 > 0 대신 != 0 사용 (Use != 0 instead of > 0 for unsigned integer comparison)

```solidity
File: helpers/FeeData.sol

64:         while (feeTokenKeys.length > 0) {

```

```solidity
File: libraries/SwapUtils.sol

96:         else if (sentAmount > 0) {

```

**프로토콜:** [4679de1](https://github.com/SwapExchangeio/Contracts/commit/4679de1c3f8adfc4122698fc82a529322ba4b5ed) 커밋에서 수정되었습니다.

**Cyfrin:** 검증됨.

