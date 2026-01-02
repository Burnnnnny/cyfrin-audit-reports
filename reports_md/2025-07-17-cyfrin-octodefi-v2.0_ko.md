**수석 감사**

[Farouk](https://x.com/ubermensh3dot0)

[Giovanni Di Siena](https://x.com/giovannidisiena)

**보조 감사**

---

# 발견 사항

## 높은 위험 (High Risk)

### 중요한 `FeeController` 설정자에 대한 접근 제어 누락

**설명:** `FeeController.setFunctionFeeConfig()`, `setTokenGetter()` 및 `setGlobalTokenGetter()`는 `external`로 선언되었지만 호출자를 제한하는 **수정자(modifier)가 없습니다**. 모든 계정은 수수료율, 토큰 게터(getter) 주소를 자유롭게 (재)구성하거나 악의적인 계약으로 바꿀 수도 있습니다.

```solidity
/// @inheritdoc IFeeController
function setFunctionFeeConfig(bytes4 _selector, FeeType _feeType, uint256 _feePercentage) external {
    if (_feePercentage > maxFeeLimits[_feeType]) {
        revert FeePercentageExceedLimit();
    }

    functionFeeConfigs[_selector] = FeeConfig(_feeType, _feePercentage);

    emit FeeConfigSet(_selector, _feeType, _feePercentage);
}

/// @inheritdoc IFeeController
function setTokenGetter(bytes4 _selector, address _tokenGetter, address _target) external {
    if (_target == address(0) || _tokenGetter == address(0)) {
        revert ZeroAddressNotValid();
    }

    tokenGetters[_target][_selector] = _tokenGetter;
    emit TokenGetterSet(_target, _selector, _tokenGetter);
}

/// @inheritdoc IFeeController
function setGlobalTokenGetter(bytes4 _selector, address _tokenGetter) external {
    if (_tokenGetter == address(0)) {
        revert ZeroAddressNotValid();
    }

    globalTokenGetters[_selector] = _tokenGetter;
    emit GlobalTokenGetterSet(_selector, _tokenGetter);
}
```

**파급력:** 공격자는 다음을 수행할 수 있습니다:
* 수수료율을 0으로 설정하여 실행자의 수익을 빼돌리거나 플러그인을 사용하는 스마트 계정을 괴롭히기 위해 최대화합니다.
* `tokenGetters` 및 `globalTokenGetters`를 모든 호출에서 되돌리는(revert) 계약으로 설정하여 DoS를 유발합니다.

**권장 완화 방법:** `FeeController` 내부의 모든 상태 변경 관리자 설정자에 `onlyOwner` 수정자를 추가하십시오.

**OctoDeFi:** PR [\#12](https://github.com/octodefi/strategy-builder-plugin/pull/12)에서 수정되었습니다.

**Cyfrin:** 확인됨. `onlyOwner` 수정자가 `setFunctionFeeConfig()`, `setTokenGetter()` 및 `setGlobalTokenGetter()` 함수에 적용되었습니다.

\clearpage
## 중간 위험 (Medium Risk)

### 토큰 소수점이 18이 아닐 때 수수료 계산이 깨짐

**설명:** `FeeController.calculateFee()` 및 `calculateTokenAmount()`는 모든 ERC-20이 **18 소수점**을 갖는 것처럼 암시적으로 처리하지만, 많은 인기 있는 자산(USDC/USDT = 6 등)은 그렇지 않습니다.
`calculateFee` 내부의 다음 줄은

```solidity
uint256 _feeInUSD = _feeAmount * _tokenPrice / 10**18;
```

**오라클의** 18-dec 스케일링을 취소하기 위해 `1e18`로 나누지만 토큰 자체의 소수점은 완전히 무시합니다. 토큰의 소수점이 적으면 결과 "USD 값"은 `10**(18-decimals)`만큼 줄어들고, 소수점이 더 많으면 부풀려집니다.

`_executeStep()`은 `_executeAction()`에 의해 반환된 개별 작업 수수료를 **더하기** 때문에, 다른 정밀도의 토큰을 혼합하는 모든 단계는 동일한 단위로 표현되지 않은 숫자를 합산합니다:

| 토큰 | 소수점 | 1 %에서 $100 볼륨에 대한 수수료 (예상 $1 = 1e18) | 실제로 반환된 수수료 |
|-------|----------|-----------------------------------------------|-----------------------|
| USDT  | 6        | `1 × 10¹⁸`                                    | `1 × 10⁶` (1 e12 × 너무 낮음) |
| DAI   | 18       | `1 × 10¹⁸`                                    | `1 × 10¹⁸` (정확함) |

전략 단계가 USDT로 작업을 실행한 다음 DAI로 작업을 실행하면 총 수수료는 다음과 같습니다:

```
total = 1e6  (USDT)  + 1e18 (DAI)  = 1000000000001000000  wei-USD
```

그러나 **의미적으로 올바른** 금액은 `2 × 10¹⁸`이어야 합니다. 다운스트림 효과:

**파급력:** * `StrategyBuilderPlugin.executeAutomation()`은 집계된 수수료를 사용자가 제공한 `maxFeeInUSD`와 비교합니다. 사용자는 `maxFeeInUSD = 1.1e18`을 설정하고 잘못 스케일링된 USDT 수수료가 총계를 거의 이동시키지 않기 때문에 두 작업(DAI 부분만 지불)을 모두 실행할 수 있습니다.
* 실행자는 모든 6/8/12-dec 토큰에 대해 수익을 놓칩니다. 반대로 이국적인 고정밀 토큰 사용자는 과다 청구되어 자동화가 `FeeExceedMaxFee()`로 되돌려질 수 있습니다.

**권장 완화 방법:**
1. **18 대신 token.decimals()를 사용하십시오.**
   ```solidity
   uint8 decimals = IERC20Metadata(token).decimals();   // OZ ERC20Metadata
   uint256 scale = 10 ** decimals;                      // token’s native unit

   // feeAmount is already in token-units
   uint256 _feeInUSD = _feeAmount * _tokenPrice / scale;  // always 18-dec USD
   ```

2. **`calculateTokenAmount()`에서도 마찬가지로**
   ```solidity
   uint8 decimals = IERC20Metadata(token).decimals();
   uint256 scale = 10 ** decimals;
   return feeInUSD * scale / tokenPrice;
   ```

**OctoDeFi:** PR [\#13](https://github.com/octodefi/strategy-builder-plugin/pull/13)에서 수정되었습니다.

**Cyfrin:** USD 수수료를 오라클 소수점(18)으로 정규화하려면 두 인스턴스 모두에서 토큰 소수점을 나누어야 합니다. 또한 `staticcall`을 사용하여 토큰 소수점을 쿼리하고 실패 시 18로 폴백하는 것을 고려하십시오.

**OctoDeFi:** PR [\#27](https://github.com/octodefi/strategy-builder-plugin/pull/27)에서 수정되었습니다.

**Cyfrin:** 확인됨. 수수료 금액은 먼저 18 소수점으로 정규화된 다음 오라클 소수점으로 나뉩니다. 오라클 가격이 이미 18 소수점이라는 점을 감안할 때 필요 이상으로 다소 복잡하지만, 수수료를 먼저 18 소수점으로 정규화하지 않고 오라클 소수점 대신 토큰 소수점으로 나눌 수 있었다는 점을 감안하면 이제 정확합니다.

### `PriceOracle::getTokenPrice`에서 최신성 확인 누락

**설명:** `PriceOracle.getTokenPrice()`는 `pythOracle.getPriceUnsafe()`를 중계하지만 가격 **게시 시간**이나 신뢰 구간을 절대 확인하지 않습니다. 몇 시간 된 가격(또는 의도적으로 동결된 가격)도 최신 가격으로 허용됩니다.

```solidity
function getTokenPrice(address _token) external view returns (uint256) {
    bytes32 _oracleID = oracleIDs[_token];

    if (_oracleID == bytes32(0)) {
        revert OracleNotExist(_token);
    }

    PythStructs.Price memory price = pythOracle.getPriceUnsafe(_oracleID);

    return _scalePythPrice(price.price, price.expo);
}
```

**파급력:** * 자동화 수수료가 오래된 데이터로 계산되어 공격자가 초과 지불하거나 적게 지불할 수 있습니다.

**권장 완화 방법:** [getPriceUnsafe](https://api-reference.pyth.network/price-feeds/evm/getPriceUnsafe) 대신 [getPriceNoOlderThan](https://api-reference.pyth.network/price-feeds/evm/getPriceNoOlderThan) 함수를 사용하는 것을 고려하십시오.

**OctoDeFi:** PR [\#24](https://github.com/octodefi/strategy-builder-plugin/pull/24)에서 수정되었습니다.

**Cyfrin:** 확인됨. 오라클이 120초보다 오래된 가격을 보고하면 `PriceOracle`은 최소 수수료가 사용되도록 0을 반환합니다. 이는 좋은 해결책이지만 0 가격은 `calculateTokenAmount()`에서 0으로 나누기로 인한 패닉 되돌림(revert)을 유발합니다.

**OctoDeFi:** PR [\#27](https://github.com/octodefi/strategy-builder-plugin/pull/27)에서 수정되었습니다.

**Cyfrin:** 확인됨. 0 가격 케이스는 되돌림을 피하기 위해 명시적으로 처리되었으며 임계값은 60초로 줄어들었습니다. 이 임계값은 특정 피드에 대해 너무 제한적일 수 있으며 피드별로 구성하는 것이 좋습니다.

**OctoDeFi:** 임계값과 관련하여 우리는 특히 이것이 담보가 위험에 처한 볼트가 아니라 수수료 계산이라는 점을 고려할 때 60초가 상당히 제한적이라고 생각합니다.

대부분의 Pyth 오라클은 주요 토큰에 대해 약 1-2초마다 업데이트됩니다. 그러나 이들은 풀(pull) 기반 오라클이므로 더 작거나 유동성이 적은 토큰은 업데이트 속도가 훨씬 느릴 수 있다는 점에 유의하는 것이 중요합니다.

단순히 오래된 것이 아니라 오라클이 더 이상 작동하지 않는 시점을 실제로 감지하기 위해 임계값을 훨씬 더 높게 설정하는 것이 실제로 타당할 수 있습니다. 어쨌든 단일 임계값만으로는 가격 조작을 완전히 방지할 수 없습니다.

**Cyfrin:** 인지함.

### 블랙리스트에 있거나 되돌리는(reverting) 수수료 수신자를 통한 자동화 DoS

**설명:** `FeeHandler.handleFee()`는 `safeTransferFrom()`을 사용하여 ERC-20 토큰을 `beneficiary`, `creator`, `vault` 및 `burnerAddress`로 전달합니다. 해당 주소 중 하나라도 USDT/USDC에 의해 **블랙리스트**에 있으면(전송이 false 반환) 호출이 되돌려집니다.
`handleFeeETH()`는 2300 가스 유닛의 제한이 있는 `transfer()`를 사용합니다. 예를 들어 가스 제한으로 커버할 수 있는 것보다 더 많은 로직을 실행하는 스마트 계약 지갑과 같이 대상 계약의 `receive()`가 되돌리면 전체 자동화가 실패합니다.
```solidity
IERC20(token).safeTransferFrom(msg.sender, beneficiary, beneficiaryAmount);

if (creator != address(0)) {
    IERC20(token).safeTransferFrom(msg.sender, creator, creatorAmount);
    IERC20(token).safeTransferFrom(msg.sender, vault, vaultAmount);
} else {
    IERC20(token).safeTransferFrom(msg.sender, vault, vaultAmount + creatorAmount);
}

if (burnAmount > 0) {
    IERC20(token).safeTransferFrom(msg.sender, burnerAddress, burnAmount);
}
```

```solidity
payable(beneficiary).transfer(beneficiaryAmount);

if (creator != address(0)) {
    payable(creator).transfer(creatorAmount);
    payable(vault).transfer(vaultAmount);
} else {
    payable(vault).transfer(vaultAmount + creatorAmount);
}

if (burnAmount > 0) {
    payable(burnerAddress).transfer(burnAmount);
}
```
**파급력:** 블랙리스트에 있거나 되돌리는 `creator`가 있는 전략에 대한 모든 향후 `executeAutomation()` 호출에 대한 서비스 거부(DoS).

**권장 완화 방법:** 푸시 자금 모델 대신 각 엔터티가 자체적으로 수수료를 청구할 수 있는 **풀(pull)** 패턴을 사용하거나, 단일 실패 대상이 실행을 차단할 수 없도록 전송 주위에 `try/catch`를 사용하십시오. 또한 `transfer()`를 사용하여 네이티브 토큰 전송을 수행하지 말고 안전한 네이티브 토큰 전송을 구현하는 일부 라이브러리를 활용하십시오.

**OctoDeFi:** PR [\#14](https://github.com/octodefi/strategy-builder-plugin/pull/14)에서 수정되었습니다.

**Cyfrin:** 확인됨. 사용자가 누적된 수수료 잔액을 청구할 수 있도록 인출 방법이 구현되었습니다. 네이티브 토큰 전송은 여전히 `transfer()` 메서드에 의존하므로 이 또한 업데이트해야 합니다. `nonReentrant()` 수정자의 적용 또한 필요하지 않습니다.

**OctoDeFi:** 커밋 [7c48784](https://github.com/octodefi/strategy-builder-plugin/commit/7c48784640163998a9265f580d3b18aa46bc36a6)에서 수정되었습니다.

**Cyfrin:** 확인됨. Solady `SafeTransferLib`가 이제 모든 토큰 전송에 사용됩니다.

### 배열 인덱스 범위를 벗어남으로 인한 전략 실행 DoS

**설명:** `StrategyBuilderPlugin._validateStep()`은 조건 결과가 최대 단계 인덱스를 초과하지 않는 새 인덱스를 참조하도록 강제하려고 합니다:

```solidity
function _validateStep(StrategyStep memory step, uint256 maxStepIndex) internal pure {
    if (step.condition.result0 > maxStepIndex || step.condition.result1 > maxStepIndex) {
        revert InvalidNextStepIndex();
    }
}
```

그러나 `maxStepIndex`가 배열 길이로 전달되므로 이는 0 인덱싱을 고려하지 못하며, 1만큼 차이 나는(off-by-one) 인덱스가 지정되면 배열 인덱스 범위를 벗어나 실행 중 되돌리는 전략을 추가할 수 있습니다.

**파급력:** 실행 DoS를 유발하는 전략을 생성할 수 있습니다.

**개념 증명 (Proof of Concept):** 다음 테스트를 `StrategyBuilderPlugin.t.sol`에 추가할 수 있습니다:

```solidity
function test_executeStrategy_OOB() external {
    uint256 numSteps = 2;
    IStrategyBuilderPlugin.StrategyStep[] memory steps = _createStrategySteps(numSteps);
    steps[0].condition.result0 = 2;
    steps[0].condition.result1 = 2;
    uint32 strategyID = 222;

    deal(address(account1), 100 ether);

    //Mocks
    vm.mockCall(
        feeController,
        abi.encodeWithSelector(IFeeController.getTokenForAction.selector),
        abi.encode(address(0), false)
    );

    vm.mockCall(
        feeController,
        abi.encodeWithSelector(IFeeController.functionFeeConfig.selector),
        abi.encode(IFeeController.FeeConfig({feeType: IFeeController.FeeType.Deposit, feePercentage: 0}))
    );
    vm.mockCall(feeController, abi.encodeWithSelector(IFeeController.minFeeInUSD.selector), abi.encode(0));

    //Act
    vm.startPrank(address(account1));
    strategyBuilderPlugin.createStrategy(strategyID, creator, steps);

    strategyBuilderPlugin.executeStrategy(strategyID);
    vm.stopPrank();

    //Assert
    assertEq(tokenReceiver.balance, numSteps * 2 * TOKEN_SEND_AMOUNT);
}
```

**권장 완화 방법:** 0 인덱싱을 고려하려면 >=이어야 합니다.

```diff
    function _validateStep(StrategyStep memory step, uint256 maxStepIndex) internal pure {
--      if (step.condition.result0 > maxStepIndex || step.condition.result1 > maxStepIndex) {
++      if (step.condition.result0 >= maxStepIndex || step.condition.result1 >= maxStepIndex) {
            revert InvalidNextStepIndex();
        }
    }
```

**OctoDeFi:** PR [\#15](https://github.com/octodefi/strategy-builder-plugin/pull/15)에서 수정되었습니다.

**Cyfrin:** 확인됨. 유효성 검사가 이제 올바른 연산자를 사용합니다.

### 래퍼 계약을 통해 거래량 기반 수수료 우회 가능

**설명:** `_executeAction()`은 `FeeController.getTokenForAction()`이 `(target, selector)` 쌍을 추적된 토큰에 매핑할 수 있을 때만 백분율 수수료를 부과합니다. 매핑이 없으면 `minFeeInUSD`로 폴백합니다.
따라서 공격자는 `FeeController`가 알지 못하는 도우미 계약 내에 고가치 호출을 래핑할 수 있습니다.
*예 – Aave 입금:*
```text
User → StrategyBuilder → AAVEHandler.supplyFor(100 000 USDC) → Aave Pool.supply()
```
`AAVEHandler`는 사용자의 100 000 USDC를 받고 Aave 풀을 승인한 다음 사용자를 대신하여 공급하여 aUSDC를 반환합니다. `(AAVEHandler, supplyFor)`가 등록되지 않았기 때문에 `getTokenForAction()`은 `(address(0), false)`를 반환하므로 전략은 ~ 1 000 USDC (1 %) 대신 최소 수수료만 지불합니다. 동일한 트릭이 인출, 스왑 또는 모든 거래량 기반 선택자(selector)에 대해 작동합니다.

```solidity
function _executeAction(address _wallet, Action memory _action) internal returns (uint256 feeInUSD) {
    (address tokenToTrack, bool exist) =
        feeController.getTokenForAction(_action.target, _action.selector, _action.parameter);
    // If the volume token exist track the volume before and after the execution, else get the min fee

    uint256 preExecBalance = exist ? IERC20(tokenToTrack).balanceOf(_wallet) : 0;

    _execute(_wallet, _action);

    IFeeController.FeeType feeType = feeController.functionFeeConfig(_action.selector).feeType;

    if (exist) {
        uint256 postExecBalance = IERC20(tokenToTrack).balanceOf(_wallet);
        uint256 volume = feeType == IFeeController.FeeType.Deposit
            ? preExecBalance - postExecBalance
            : postExecBalance - preExecBalance;

        feeInUSD = feeController.calculateFee(tokenToTrack, _action.selector, volume);
    } else {
        feeInUSD = feeController.minFeeInUSD(feeType);
    }

    emit ActionExecuted(_wallet, _action);
}
```
**파급력:** 대규모 트랜잭션이 프로토콜의 최소 고정 수수료만 지불하면서 실행될 수 있어 실행자의 예상 수익이 심각하게 감소하거나 제거됩니다.

**권장 완화 방법:** 이는 단순한 코딩 버그가 아닌 설계 선택에서 비롯된 것이므로 온체인에서 해결하는 것은 쉽지 않습니다. 동작을 문서화하고 위험을 완화할 수 있는 설계 조정을 논의하는 것입니다.

**OctoDeFi:** PR [\#25](https://github.com/octodefi/strategy-builder-plugin/pull/25)에서 수정되었습니다.

**Cyfrin:** 확인됨. `StrategyBuilderPlugin`에 통합되도록 허용된 작업 계약을 검증하기 위해 `ActionRegistry` 계약이 추가되었습니다.

\clearpage
## 낮은 위험 (Low Risk)

### `_tokenDistribution()`의 반올림으로 인한 먼지(dust) 금액 축적

**설명:** `_tokenDistribution()`은 정수 나눗셈으로 각 지분을 독립적으로 계산합니다:

```solidity
function _tokenDistribution(uint256 amount) internal view returns (uint256, uint256, uint256) {
    uint256 beneficiaryAmount = (amount * beneficiaryPercentage) / PERCENTAGE_DIVISOR;
    uint256 creatorAmount = (amount * creatorPercentage) / PERCENTAGE_DIVISOR;
    uint256 vaultAmount = (amount * vaultPercentage) / PERCENTAGE_DIVISOR;
    return (beneficiaryAmount, creatorAmount, vaultAmount);
}
```

`beneficiaryPercentage + creatorPercentage + vaultPercentage == 10000`이지만 나눗셈이 잘리면(truncates) `beneficiaryAmount + creatorAmount + vaultAmount < amount`가 되어 1–2 wei의 "먼지"가 남습니다.

**파급력:** 먼지가 점차 축적되어 손실되고 의도한 수령인에게 도달하지 않습니다.

**권장 완화 방법:** 위와 같이 `beneficiaryAmount` 및 `creatorAmount`를 계산한 다음
```solidity
vaultAmount = amount - beneficiaryAmount - creatorAmount;
```
로 설정하여 먼지 없이 완전한 분배를 보장하십시오.

**OctoDeFi:** PR [\#16](https://github.com/octodefi/strategy-builder-plugin/pull/16)에서 수정되었습니다.

**Cyfrin:** 확인됨. `vaultAmount`는 이제 수혜자와 창작자 금액을 공제한 후 나머지로 계산됩니다.

### 자동화 삭제 시 `automationsToIndex` 스토리지가 올바르게 재설정되지 않음

**설명:** 자동화를 삭제할 때 로직은 `_usedInAutomations` 배열에서 마지막 요소를 팝(pop)하지만 `automationsToIndex` 매핑 스토리지를 올바르게 재설정하지 못합니다. if 문이 실행되는지 여부에 관계없이 삭제가 있을 때마다 `automationsToIndex[automationSID]`를 0으로 재설정해야 합니다:

```solidity
    function _deleteAutomation(address wallet, uint32 id) internal {
        bytes32 automationSID = getStorageId(wallet, id);
        Automation memory _automation = automations[automationSID];

        uint32[] storage _usedInAutomations = strategiesUsed[getStorageId(wallet, _automation.strategyId)];

@>      uint32 _actualAutomationIndex = automationsToIndex[automationSID];
        uint256 _lastAutomationIndex = _usedInAutomations.length - 1;
        if (_actualAutomationIndex != _lastAutomationIndex) {
            uint32 _lastAutomation = _usedInAutomations[_lastAutomationIndex];
            _usedInAutomations[_actualAutomationIndex] = _lastAutomation;
@>          automationsToIndex[getStorageId(wallet, _lastAutomation)] = _actualAutomationIndex;
        }
        _usedInAutomations.pop();

        _changeAutomationInCondition(
            wallet, _automation.condition.conditionAddress, _automation.condition.id, id, false
        );

        delete automations[automationSID];

        emit AutomationDeleted(wallet, id);
    }
```

**파급력:** 동일한 ID로 자동화가 다시 추가되면 매핑이 덮어씌워질 것으로 보이며 `_usedInAutomations`가 올바르게 팝되므로 항목이 잘못 참조되지 않을 것으로 보이므로 영향은 제한적입니다.

**권장 완화 방법:** 해당 자동화를 삭제할 때 주어진 자동화 스토리지 ID에 대한 매핑 값을 재설정하십시오.

**OctoDeFi:** PR [#17](https://github.com/octodefi/strategy-builder-plugin/pull/17)에서 수정되었습니다.

**Cyfrin:** 확인됨. 관련 `automationsToIndex` 매핑 키가 이제 `automations`와 함께 지워집니다.

### 잠재적인 0 값 전송 되돌림으로 인한 자동화 DoS

**설명:** `primaryTokenBurn` 및 `tokenBurn`에 의해 지정된 토큰 소각 비율은 단순히 `(0, 100]` 범위로 제한됩니다. 소각 비율이 100%로 구성되면 `_feeCalculation()`에 의해 반환된 `FeeHandler.handleFee()`의 소각 금액은 0이 아니지만 수혜자/창작자/볼트 사이에 분배될 총 수수료는 0인 시나리오가 발생할 수 있습니다. 표준 OpenZeppelin ERC-20 구현은 0 값 전송 시 되돌리지 않으며 소각 비율이 그렇게 높게 설정될 가능성은 거의 없지만, 가능하므로 0 값 전송 시 되돌리는 토큰 구현에 대한 DoS를 피하기 위해 분배 금액 중 하나라도 0인 경우 이 경우 전송을 건너뛰는 것이 좋습니다.

**파급력:** 소각 비율이 100%로 구성된 경우 결제를 시도할 때 자동화가 되돌려질 수 있습니다.

**권장 완화 방법:** `burnAmount`에 대한 유효성 검사와 유사하게, 각 수혜자/창작자/볼트 금액이 0이 아닌 경우에만 토큰 전송을 수행하도록 시도하십시오.

**OctoDeFi:** PR [\#14](https://github.com/octodefi/strategy-builder-plugin/pull/14)에서 수정되었습니다. M-3에서 도입된 풀(pull) 기반 방법으로 인해 더 이상 토큰이 직접 전송되지 않으므로 이 문제도 해결되었습니다.

**Cyfrin:** 확인됨. 푸시 전송 패턴이 이 문제를 해결하는 풀 패턴으로 업데이트되었습니다.

\clearpage
## 정보성 (Informational)

### 가격 오라클에 0 가격 유효성 검사가 포함될 수 있음

**설명:** `FeeController.calculateTokenAmount()`는 오라클이 반환한 가격이 0이 아닌지 확인합니다:

```solidity
    function calculateTokenAmount(address token, uint256 feeInUSD) external view returns (uint256) {
        bytes32 oracleID = oracle.oracleID(token);

        if (oracleID == bytes32(0)) {
            revert NoOracleExist();
        }

        uint256 tokenPrice = oracle.getTokenPrice(token);

@>      if (tokenPrice == 0) {
            revert InvalidTokenWithPriceOfZero();
        }

        return feeInUSD * 10 ** 18 / tokenPrice;
    }
```

대신 가격이 음수가 아닌지 이미 확인하는 `PriceOracle._scalePythPrice()`에 이를 포함할 수 있습니다:

```solidity
    function _scalePythPrice(int256 _price, int32 _expo) internal pure returns (uint256) {
@>      if (_price < 0) {
            revert NegativePriceNotAllowed();
        }
        ...
    }
```

**권장 완화 방법:**
```diff
    function _scalePythPrice(int256 _price, int32 _expo) internal pure returns (uint256) {
--      if (_price < 0) {
++      if (_price <= 0) {
            revert NegativePriceNotAllowed();
        }
        ...
    }

    function calculateTokenAmount(address token, uint256 feeInUSD) external view returns (uint256) {
        ...
--      if (tokenPrice == 0) {
--          revert InvalidTokenWithPriceOfZero();
--      }

        return feeInUSD * 10 ** 18 / tokenPrice;
    }
```

**OctoDeFi:** PR [\#18](https://github.com/octodefi/strategy-builder-plugin/pull/18)에서 수정되었습니다.

**Cyfrin:** 확인됨. 유효성 검사는 이제 가격 오라클 내에서만 수행됩니다.

### 양수 Pyth 오라클 지수는 명시적으로 처리되어야 함

**설명:** `PriceOracle._scalePythPrice()`는 지수 `_expo`가 항상 음수라고 가정합니다:

```solidity
    function _scalePythPrice(int256 _price, int32 _expo) internal pure returns (uint256) {
        if (_price < 0) {
            revert NegativePriceNotAllowed();
        }

@>      uint256 _absExpo = uint32(-_expo);

        if (_expo <= -18) {
            return uint256(_price) * (10 ** (_absExpo - 18));
        }

        return uint256(_price) * 10 ** (18 - _absExpo);
    }
```

이 가정은 [위반될 가능성이 없지만](https://x.com/abarbatei/status/1901327645373030711), 서명된 유형 및 [다른 라이브러리](https://github.com/pyth-network/pyth-crosschain/blob/main/target_chains/ethereum/sdk/solidity/PythUtils.sol)에서의 사용법에 따라 지수가 양수 값으로 구성될 수 있습니다.

프로토콜이 양수 지수를 가진 Pyth 오라클에 의존하게 되면 `uint32(-expo)`가 조용히 언더플로되어 거대한 절대값을 초래하고 최종 스케일링 중에 실행이 되돌려질 수 있습니다.

**권장 완화 방법:** 지수가 양수인 경우를 명시적으로 처리하는 것을 고려하십시오.

**OctoDeFi:** PR [\#19](https://github.com/octodefi/strategy-builder-plugin/pull/19)에서 수정되었습니다.

**Cyfrin:** 확인됨. 양수 지수 케이스가 이제 명시적으로 처리됩니다.

### 길이 0 확인 누락으로 빈 전략이 생성될 수 있음

**설명:** `StrategyBuilderPlugin.createStrategy()`는 현재 `steps` 배열 길이가 0인 경우 되돌리지 않습니다. 이는 `strategies[getStorageId(wallet, id)].steps.length`가 0이 아닌지 확인하는 `StrategyBuilderPlugin.deleteStrategy()`에 적용된 `strategyExist` 수정자 로직에 따라 삭제할 수 없는 전략을 생성할 수 있음을 의미합니다:

```solidity
modifier strategyExist(address wallet, uint32 id) {
    if (strategies[getStorageId(wallet, id)].steps.length == 0) {
        revert StrategyDoesNotExist();
    }
    _;
}
```

**파급력:** 전략은 다른 지갑을 대신하여 생성할 수 없으므로 영향은 제한적이지만 이 동작은 바람직하지 않을 가능성이 큽니다.

**개념 증명 (Proof of Concept):** 다음 테스트를 `StrategyBuilderPlugin.t.sol`에 추가할 수 있습니다:

```solidity
function test_createStrategy_Empty() external {
    uint256 numSteps;
    IStrategyBuilderPlugin.StrategyStep[] memory steps = _createStrategySteps(numSteps);

    uint32 strategyID = 222;
    vm.prank(address(account1));
    strategyBuilderPlugin.createStrategy(strategyID, creator, steps);

    //Assert
    IStrategyBuilderPlugin.Strategy memory strategy = strategyBuilderPlugin.strategy(address(account1), strategyID);

    assertEq(strategy.creator, creator);
    assertEq(strategy.steps.length, numSteps);

    vm.prank(address(account1));
    vm.expectRevert(IStrategyBuilderPlugin.StrategyDoesNotExist.selector);
    strategyBuilderPlugin.deleteStrategy(strategyID);
}

function test_createStrategy_EmptyNonZeroLength() external {
    uint256 numSteps = 2;
    IStrategyBuilderPlugin.StrategyStep[] memory steps = new IStrategyBuilderPlugin.StrategyStep[](numSteps);

    uint32 strategyID = 222;
    vm.prank(address(account1));
    strategyBuilderPlugin.createStrategy(strategyID, creator, steps);

    //Assert
    IStrategyBuilderPlugin.Strategy memory strategy = strategyBuilderPlugin.strategy(address(account1), strategyID);

    assertEq(strategy.creator, creator);
    assertEq(strategy.steps.length, numSteps);

    vm.prank(address(account1));
    strategyBuilderPlugin.executeStrategy(strategyID);

    vm.prank(address(account1));
    strategyBuilderPlugin.deleteStrategy(strategyID);
}
```

**권장 완화 방법:** `steps` 길이가 0인 경우 되돌리는 것을 고려하십시오. 두 번째 PoC에서와 같이 기본 조건 주소를 0 주소로 활용하여 0이 아닌 배열 길이로 사실상 빈 전략을 생성하는 것은 여전히 가능합니다.

**OctoDeFi:** PR [\#20](https://github.com/octodefi/strategy-builder-plugin/pull/20)에서 수정되었습니다.

**Cyfrin:** 확인됨. 조건이나 작업 없이 빈 전략 단계를 방지하기 위해 추가 검증이 구현되었습니다.

### 조건 주소가 `StrategyBuilderPlugin`에 재진입할 수 있음

**설명:** 조건 주소는 `_changeStrategyInCondition()` 및 `_changeAutomationInCondition()`의 외부 호출에서 `StrategyBuilderPlugin`에 재진입할 수 있습니다. 후자의 경우 여기서 발생할 수 있는 최악의 상황은 `strategiesUsed`에 푸시할 때 배열 항목을 복제하여 `automationsToIndex`를 손상시키고 중복 항목이 제거되지 않도록 하는 것(삭제 중에도 재진입이 발생하지 않는 한)으로 보입니다. 따라서 영향은 제한적이지만 향후 수정 시 이를 인지하는 것이 중요합니다.

**OctoDeFi:** 인지함. 재진입 문제를 기록했습니다.

**Cyfrin:** 인지함.

\clearpage
## 가스 최적화 (Gas Optimization)

### `getStorageId(msg.sender, id)`의 사용되지 않는 캐시된 값

**설명:** `createAutomation()`은 `getStorageId(msg.sender, id)`를 두 번 호출합니다. 한 번은 `automationSID`에 대해, 다시 한 번은 `_newAutomation`을 할당할 때입니다. 이는 동일한 keccak 256 계산과 추가 스택 쓰기를 중복합니다.

**권장 완화 방법:** 첫 번째 반환 값을 로컬 변수에 저장하고 재사용하십시오:

```solidity
bytes32 automationSID = getStorageId(msg.sender, id);
Automation storage _newAutomation = automations[automationSID];
```

이렇게 하면 호출당 하나의 `STATICCALL`/`KECCAK256` 연산과 몇 가지 스택 연산을 절약할 수 있습니다.

**OctoDeFi:** PR [\#21](https://github.com/octodefi/strategy-builder-plugin/pull/21)에서 수정되었습니다.

**Cyfrin:** 확인됨. 이제 캐시된 값이 사용됩니다.

### 명명된 반환 변수 사용

**설명:** 불필요한 스택 변수 할당을 피하기 위해 명명된 반환 변수를 사용할 수 있는 경우가 많이 있습니다. 예를 들어 `FeeHandler._tokenDistribution()`에서 그렇습니다. 가스를 절약하기 위해 이 함수와 기타 관련 함수를 수정하는 것을 고려하십시오.

**OctoDeFi:** PR [\#22](https://github.com/octodefi/strategy-builder-plugin/pull/22)에서 수정되었습니다.

**Cyfrin:** 확인됨. 명명된 반환 변수가 이제 사용됩니다.

### 여러 루프 반복에서 사용되는 배열 길이를 캐시할 수 있음

**설명:** 전략 단계를 검증할 때 길이를 캐시하여 가스를 절약할 수 있는데도 각 루프 반복마다 검색됩니다:

```solidity
function _validateSteps(StrategyStep[] memory steps) internal pure {
    for (uint256 i = 0; i < steps.length; i++) {
        _validateStep(steps[i], steps.length);
    }
}
```

**OctoDeFi:** PR [\#23](https://github.com/octodefi/strategy-builder-plugin/pull/23)에서 수정되었습니다.

**Cyfrin:** 확인됨. 길이가 이제 캐시됩니다.

\clearpage
