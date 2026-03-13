**Lead Auditors**

[Giovanni Di Siena](https://twitter.com/giovannidisiena)

# Findings
## Critical Risk


### Hypervisor collateral redemption can cause vaults to become undercollateralized due to slippage

**Description:** Hypervisor 담보를 상환할 때, `SmartVaultV4::autoRedemption`은 `SmartVaultYieldManager::quickWithdraw` 호출을 수행합니다:

```solidity
if (_hypervisor != address(0)) {
    address _yieldManager = ISmartVaultManager(manager).yieldManager();
    IERC20(_hypervisor).safeIncreaseAllowance(_yieldManager, getAssetBalance(_hypervisor));
    _withdrawn = ISmartVaultYieldManager(_yieldManager).quickWithdraw(_hypervisor, _collateralToken);
    IERC20(_hypervisor).forceApprove(_yieldManager, 0);
}
```

이는 추가적인 프로토콜 인출 수수료를 적용하지 않고 기본 담보를 인출하기 위해 `SmartVaultYieldManager::_withdrawOtherDeposit`을 호출합니다:

```solidity
function quickWithdraw(address _hypervisor, address _token) external returns (uint256 _withdrawn) {
    IERC20(_hypervisor).safeTransferFrom(msg.sender, address(this), IERC20(_hypervisor).balanceOf(msg.sender));
    _withdrawOtherDeposit(_hypervisor, _token);
    uint256 _withdrawn = _thisBalanceOf(_token);
    IERC20(_token).safeTransfer(msg.sender, _withdrawn);
}
```

자동 상환이 완료된 후, 초과 기본 담보는 `SmartVaultV4::redeposit`을 통해 Hypervisor에 다시 예치됩니다:

```solidity
function redeposit(uint256 _withdrawn, uint256 _collateralBalance, address _hypervisor, address _collateralToken)
    private
{
    uint256 _redeposit = _withdrawn > _collateralBalance ? _collateralBalance : _withdrawn;
    address _yieldManager = ISmartVaultManager(manager).yieldManager();
    IERC20(_collateralToken).safeIncreaseAllowance(_yieldManager, _redeposit);
    ISmartVaultYieldManager(_yieldManager).quickDeposit(_hypervisor, _collateralToken, _redeposit);
    IERC20(_collateralToken).forceApprove(_yieldManager, 0);
}
```

이는 `SmartVaultYieldManager::quickDeposit` 호출을 수행하며, 마찬가지로 추가적인 프로토콜 예금 수수료를 적용하지 않고 기본 담보를 예치하기 위해 `SmartVaultYieldManager::_otherDeposit_`을 호출합니다:

```solidity
function quickDeposit(address _hypervisor, address _collateralToken, uint256 _deposit) external {
    IERC20(_collateralToken).safeTransferFrom(msg.sender, address(this), _deposit);
    HypervisorData memory _hypervisorData = hypervisorData[_collateralToken];
    _otherDeposit(_collateralToken, _hypervisorData);
}
```

이 로직의 문제는 `SmartVaultYieldManager` 함수들이 호출하는 함수에 의해 슬리퍼지(slippage)가 충분히 처리되었다고 가정한다는 것입니다:

```solidity
// within _withdrawOtherDeposit()
IHypervisor(_hypervisor).withdraw(
    _thisBalanceOf(_hypervisor), address(this), address(this), [uint256(0), uint256(0), uint256(0), uint256(0)]
);

// within _swapToSingleAsset(), called from within _withdrawOtherDeposit()
// similar is present within _buy() and _sell() called from within _otherDeposit() -> _swapToRatio()
ISwapRouter(uniswapRouter).exactInputSingle(
    ISwapRouter.ExactInputSingleParams({
        tokenIn: _unwantedToken,
        tokenOut: _wantedToken,
        fee: _fee,
        recipient: address(this),
        deadline: block.timestamp + 60,
        amountIn: _balance,
        amountOutMinimum: 0,
        sqrtPriceLimitX96: 0
    })
);

// within _deposit(), called from within _otherDeposit()
IUniProxy(uniProxy).deposit(
    _thisBalanceOf(_token0),
    _thisBalanceOf(_token1),
    msg.sender,
    _hypervisor,
    [uint256(0), uint256(0), uint256(0), uint256(0)]
);
```

현재 로직은 자동 상환의 결과로 볼트가 담보 부족 상태가 될 수 없다고 가정하고 그러한 검증을 수행하지 않습니다. 그러나 공격자는 슬리퍼지 처리 부족을 이용하여 볼트가 담보 부족 상태가 되도록 할 수 있습니다.

빈 슬리퍼지 매개변수로 인해 재예치 로직 실행 직후 담보 확인을 수행해야 합니다.

**Impact:** Hypervisor 담보의 자동 상환은 볼트의 담보 부족을 초래할 수 있습니다.

**Recommended Mitigation:** 자동 상환의 결과로 볼트가 담보 부족 상태가 되는 것을 방지하기 위해 추가 유효성 검사를 추가하십시오.

**The Standard DAO:** 커밋 [b71beb0](https://github.com/the-standard/smart-vault/commit/b71beb03aef660f5f7cedcde88167e816111f283)으로 수정됨.

**Cyfrin:** 이것은 Hypervisor 예금이 있는 새로운 `SmartVaultV4`가 자동 상환으로 인해 담보 부족 상태가 되는 것을 방지하지만, 청산 임계값 바로 위까지 상당한 담보 감소를 방지하지는 못합니다. 덧붙여서, 우리는 자동 상환이 이미 담보가 부족하지만 청산되지 않은 볼트에 대해 담보 교환을 시도하고 있지 않은지 확인해야 합니다 – 대신 여기서 상당한 감소 확인을 선호합니다.

**The Standard DAO:** 커밋 [b92c98c](https://github.com/the-standard/smart-vault/commit/b92c98c4217aa566c5e2007f8fdcb9804e200ec9), [cd6bd7c](https://github.com/the-standard/smart-vault/commit/cd6bd7c612ce1476b9aa0f4b01593be0d8b32e82), 그리고 [132a013](https://github.com/the-standard/smart-vault/commit/132a0138e3e586493dcd25bfb1b56e1c761f95c2)으로 수정됨.

**Cyfrin:** 담보 비율 런타임 불변성 검사가 추가되었습니다. 그러나 `SmartVaultV4::calculateCollaralPercentage`에서 0으로 나누기를 피하기 위해 볼트 부채가 완전히 상환되는 경우를 명시적으로 처리해야 합니다.

**The Standard DAO:** 커밋 [aa4d5df](https://github.com/the-standard/smart-vault/commit/aa4d5dfb115a3962f7edcc1fa813b67c3d6950f2)로 수정됨.

**Cyfrin:** 확인됨. `minted`가 0이면 담보 비율 유효성 검사가 더 이상 실행되지 않습니다.


### Vaults can be made erroneously liquidatable due to incorrect swap path

**Description:** 자동 상환이 실행될 때, [Uniswap v3 view-only quoter](https://github.com/Uniswap/view-quoter-v3/tree/master)의 `Quoter::quoteExactOutput` 및 `Quoter::quoteExactInput` 함수가 각각 레거시 및 비레거시 볼트에 대해 호출됩니다. 이 호출들은 되돌리지(revert) 않으므로, 실행은 스왑 수행으로 진행됩니다. 그러나 `USDs`로 스왑을 시도할 때 `collateral -> USDC`의 스왑 경로가 올바르지 않습니다:

```solidity
_amountOut = ISwapRouter(_swapRouterAddress).exactInput(
    ISwapRouter.ExactInputParams({
        path: _swapPath,
        recipient: address(this),
        deadline: block.timestamp,
        amountIn: _collateralAmount,
        // minimum amount out should be at least usd value of collateral being swapped in
        amountOutMinimum: calculator.tokenToUSD(
            ITokenManager(ISmartVaultManager(manager).tokenManager()).getTokenIfExists(_collateralAddr),
            _collateralAmount
        )
    })
);
```

스왑은 성공하여 담보를 `USDC`로 변환하지만, 볼트의 `USDs` 잔액이 변경되지 않았으므로 부채가 상환되지 않습니다:

```solidity
uint256 _usdsBalance = USDs.balanceOf(address(this));
minted -= _usdsBalance;
USDs.burn(address(this), _usdsBalance);
```

`USDC`는 유효한 담보 토큰이 아니므로, 충분한 양의 담보 토큰이 스왑되면 볼트가 청산 가능하게 됩니다.

**Impact:** 자동 상환의 결과로 볼트가 잘못 청산 가능하게 될 수 있습니다.

**Recommended Mitigation:** 자동 상환을 수행할 때 스왑 경로가 올바른지 확인하고 자동 상환의 결과로 볼트 담보가 크게 감소하지 않았는지 확인하는 체크를 추가하십시오.

**The Standard DAO:** 커밋 [3c5e136](https://github.com/the-standard/smart-vault/commit/3c5e136cdab97d88e7aa79c23ee18e50d46d6276), [821bb26](https://github.com/the-standard/smart-vault/commit/132a0138e3e586493dcd25bfb1b56e1c761f95c2#diff-821bb262cfd79a7edc00f920efc390d9572957844594bc42733fc6e70f749e9a), 그리고 [132a013](https://github.com/the-standard/smart-vault/commit/132a0138e3e586493dcd25bfb1b56e1c761f95c2)으로 수정됨. 입력 및 출력 스왑 경로를 모두 저장하므로 필요한 스왑 양을 더 효율적으로 계산하는 데 사용합니다.

**Cyfrin:** 확인됨. 입력 및 출력 경로와 `USDC` 대신 `USDs`를 올바르게 참조하도록 이름이 변경된 관련 스왑 경로 및 목표 금액 변수를 포함하는 `SwapPath` 구조체의 도입으로, 경로가 전달된 대로 구성되는 한(예: 입력: `WETH -> USDC -> USDs`, 출력: `USDs -> USDC -> WETH`) 이제 올바른 것으로 보입니다.

\clearpage
## High Risk


### `AutoRedemption` mappings are not and can never be populated

**Description:** `AutoRedemption` 컨트랙트 내에 다음 매핑이 선언되어 있습니다:

```solidity
mapping(address => address) hypervisorCollaterals;
mapping(address => bytes) swapPaths;
```

그러나 이들은 채워지지 않으며 이를 수행할 수 있는 공개 함수도 없습니다.

요청이 트리거되면 `lastRequestId`에 0이 아닌 식별자가 할당됩니다:

```solidity
 lastRequestId = _sendRequest(req.encodeCBOR(), subscriptionID, MAX_REQ_GAS, donID);
```

그러나 `AutoRedemption::fulfillRequest`의 실행은 빈 스왑 경로로 인해 되돌아가므로(revert) `lastRequestId` 스토리지가 재설정되지 않습니다:

```solidity
bytes memory _collateralToUSDCPath = swapPaths[_token];
...
lastRequestId = bytes32(0);
```

새 요청을 생성할 때 기존의 미이행 요청이 없어야 한다는 `AutoRedemption::performUpkeep` 내의 조건을 고려할 때, 이는 모든 기능을 완전히 차단하고 전체 재배포를 필요로 합니다.

**Impact:** 핵심 자동 상환 기능은 모든 볼트에 대해 중단되며 재배포 없이는 수정할 수 없습니다.

**Recommended Mitigation:** 다음 중 하나를 수행하십시오:
* 매핑 미리 채우기,
* 접근 제어 설정자(setter) 함수 추가, 또는
* `SmartVaultYieldManager` 컨트랙트에서 상태 쿼리.

**The Standard DAO:** 커밋 [d72cdce](https://github.com/the-standard/smart-vault/commit/d72cdceed7d60c30abf329e1b8338bb5b13bca2b)로 수정됨.

**Cyfrin:** 확인됨. 설정자 함수가 추가되었습니다.


### Auto redemption logic can be abused by an attacker due to insufficient access control

**Description:** `AutoRedemption::performUpkeep`은 유지 관리가 필요할 때 Chainlink Automation DON에서 사용하기 위해 노출됩니다:

```solidity
function performUpkeep(bytes calldata performData) external {
    if (lastRequestId == bytes32(0)) {
        triggerRequest();
    }
}
```

그러나 주소에 의해 함수가 호출될 수 있도록 허용하는 접근 제어가 없습니다. `lastRequestId`는 `AutoRedemption::fulfillRequest` 실행이 끝날 때 `bytes32(0)`으로 재설정되므로, 이는 트리거 조건에 관계없이 이전 유지 관리가 성공한 후 반복적으로 유지 관리를 수행할 수 있음을 의미합니다.

`AutoRedemption::fulfillRequest`가 되돌아가면(revert) `lastRequestId` 상태가 재설정되지 않아 위에 표시된 `AutoRedemption::performUpkeep`의 조건으로 인해 향후 모든 자동 상환이 완전히 차단됩니다. `SmartVaultV4Legacy::autoRedemption` 및 `SmartVaultV4::autoRedemption` 모두에서 `ERC20::balanceOf`를 사용하여 상환된 `USDs` 금액을 결정하는 것과 결합하여, 공격자는 대상 볼트에 소액의 `USDs`를 직접 전송하여 이 DoS 조건을 강제할 수 있습니다:

```solidity
uint256 _usdsBalance = USDs.balanceOf(address(this));
minted -= _usdsBalance;
```

이로 인해 볼트 잔액이 예상 최대 `minted` 금액 이상으로 부풀려지고 언더플로로 인해 실행이 되돌아갑니다. Chainlink Functions DON은 실패한 이행을 재시도하지 않으므로 전체 재배포 없이 상태를 재설정하고 핵심 기능을 복구할 방법이 없습니다.

**Impact:** 공격자는 트리거 조건에 관계없이 가격 오라클 조작에 의존하지 않고 반복적으로 자동 상환을 트리거할 수 있습니다. 이는 모든 사기성 이행 시도에 대해 Chainlink 구독이 청구되므로 프로토콜에 자금 손실을 초래할 수 있습니다. 대안으로, 이행이 되돌아가도록 만들어지면 공격자는 자동 상환 메커니즘의 기능을 완전히 차단할 수 있습니다.

**Recommended Mitigation:** * `AutoRedemption::performUpkeep` 내에서 트리거 조건을 다시 확인하고 [접근 제어](https://docs.chain.link/chainlink-automation/guides/forwarder) 추가를 고려하십시오.
* 볼트 잔액을 직접 사용하는 대신 잔액 차이로 상환된 `USDs` 금액을 계산하십시오.

**The Standard DAO:** 커밋 [5ec532e](https://github.com/the-standard/smart-vault/commit/5ec532e5f3813a865102501dbb91cf13a0813930)로 수정됨.

**Cyfrin:** 트리거 조건이 다시 확인되고 TWAP가 구현되었습니다. 그러나:
* 조작을 방지하기 위해 상당히 큰 간격(최소 900초, 1800초가 아니라면)을 사용하는 것이 좋습니다. 참고: Uniswap V3 풀 오라클은 다중 블록 MEV 저항성이 없습니다.
* `USDs` 상환 금액 계산은 직접적인 `balanceOf()` 대신 잔액 차이를 사용하도록 수정되지 않았습니다.

**The Standard DAO:** 잔액 확인 대신 스왑의 출력 금액을 사용하여 커밋 [a8cdc77](https://github.com/the-standard/smart-vault/commit/a8cdc77d1fac9817128e1f3c1c8a1ab57f715513)로 수정됨.

**Cyfrin:** 확인됨. TWAP 간격이 증가했으며 상환된 금액이 스왑에서 출력된 값을 사용하도록 수정되었습니다.


### `AutoRedemption::fulfillRequest` should never be allowed to revert

**Description:** Chainlink Functions DON은 실패한 이행을 재시도하지 않으므로 `AutoRedemption::fulfillRequest`는 결코 되돌아가도록(revert) 허용되어서는 안 됩니다. 그렇지 않으면 `lastRequestId`가 `bytes32(0)`으로 재설정되지 않아 `AutoRedemption::performUpkeep`이 새 요청을 트리거할 수 없게 됩니다.

현재 인라인 주석은 Chainlink Functions DON이 오류를 반환하면 되돌리려는 의도가 있음을 시사하지만, 위에서 설명한 이유로 이는 피해야 합니다. 마찬가지로 외부 호출로 인한 잠재적인 잘못된 응답이나 되돌림은 우아하게 처리되어 다음 라인으로 넘어가야 합니다:

```solidity
lastRequestId = bytes32(0);
```

**Impact:** 자동 상환 기능의 완전한 DoS.

**Recommended Mitigation:** * 오류가 보고되면 되돌리지(revert) __마십시오__.
* 디코딩이 되돌아가지 않도록 응답의 길이를 예상 길이와 비교하여 유효성 검사하십시오.
* `try/catch` 블록을 사용하여 모든 외부 호출의 되돌림을 처리하십시오.
* `AutoRedemption::calculateUSDsToTargetPrice`가 `0`을 반환하면 단락(short-circuit)하십시오(이로 인해 스왑이 [되돌아가기](https://github.com/Uniswap/v3-core/blob/main/contracts/UniswapV3Pool.sol#L603) 때문입니다).
* 선택적으로 `lastRequestId`를 재설정하는 접근 제어 관리자 기능을 추가하십시오.

**The Standard DAO:** 커밋 [5235524](https://github.com/the-standard/smart-vault/commit/523552498edefe77aa2782d2a887bb1980cf80b9)로 수정됨.

**Cyfrin:** 확인됨. 이제 응답 길이가 예상 길이와 비교하여 유효성 검사되며, 오류가 보고되면 로직이 건너뛰어집니다. `AutoRedemption::runAutoRedemption`은 목표 `USDs` 금액이 0이 아닌 경우에만 실행되며 외부 호출의 다른 되돌림은 `try/catch` 블록을 사용하여 처리됩니다. `lastRequestId`를 강제로 재설정하는 관리자 기능은 추가되지 않았습니다.


### Auto redemption functionality broken by incorrect address passed to `SmartVaultV4::autoRedemption`

**Description:** `SmartVaultV4::autoRedemption`은 다음 서명을 가집니다:

```solidity
function autoRedemption(
    address _swapRouterAddress,
    address _quoterAddress,
    address _collateralToken,
    bytes memory _swapPath,
    uint256 _USDCTargetAmount,
    address _hypervisor
)
```

그러나 `AutoRedemption::fulfillRequest`에서의 호출은 스왑 라우터 주소 대신 볼트 주소를 잘못 전달합니다:

```solidity
IRedeemable(_smartVault).autoRedemption(
    _smartVault, quoter, _token, _collateralToUSDCPath, _USDsTargetAmount, _hypervisor
);
```

**Impact:** 레거시가 아닌 볼트의 부채를 상환하려고 할 때 자동 상환 기능이 완전히 중단됩니다.

**Recommended Mitigation:** `SmartVaultManagerV6` 컨트랙트에 저장된 올바른 스왑 라우터 주소를 전달하십시오.

**The Standard DAO:** 커밋 [4400e25](https://github.com/the-standard/smart-vault/commit/4400e25c87880b7ab1b5d19fbe520e4db5b70122)로 수정됨.

**Cyfrin:** 확인됨. 올바른 `swapRouter` 주소가 이제 전달됩니다.

\clearpage
## Medium Risk


### Chainlink Functions HTTP request is missing authentication

**Description:** `AutoRedemption` 내에 정의된 `source` 상수는 Chainlink Functions DON 내에서 해당 JavaScript 코드를 실행하는 데 사용됩니다. 그러나 대상 API 엔드포인트는 인증 없이 노출되어 있어 모든 관찰자가 요청을 보낼 수 있습니다.

**Impact:** API 엔드포인트에 대한 조정된 DDoS 공격으로 인해 서버가 중단될 수 있습니다. 이로 인해 요청이 실패하게 되며, 이는 Chainlink 구독이 청구되지만 자동 상환 페그 메커니즘이 의도한 대로 작동하지 않음을 의미합니다.

**Recommended Mitigation:** 최소한 속도 제한(rate limiting)을 구현하십시오. 가급적 [Chainlink Functions secrets](https://docs.chain.link/chainlink-functions/resources/secrets)를 사용하여 요청에 인증을 추가하십시오.

**The Standard DAO:** API에 속도 제한을 추가하여 부분적으로 수정되었습니다. 나중에 요청에 암호화된 비밀을 추가하는 것도 고려할 것입니다.

**Cyfrin:** 인지됨. 암호화된 비밀 사용이 권장됩니다.


### Automation and redemption could be artificially manipulated due to use of instantaneous `sqrtPriceX96`

**Description:** `AutoRedemption::checkUpkeep`은 Chainlink Automation DON에 의해 매 블록마다 쿼리됩니다:

```solidity
function checkUpkeep(bytes calldata checkData) external returns (bool upkeepNeeded, bytes memory performData) {
    (uint160 sqrtPriceX96,,,,,,) = pool.slot0();
    upkeepNeeded = sqrtPriceX96 <= triggerPrice;
}
```

그러나 `sqrtPriceX96`의 사용은 문제가 있습니다. 왜냐하면 이는 풀 준비금(pool reserves)의 순간적인 상태를 기반으로 하며, 이는 유지 관리가 필요하지 않은 경우에도 트리거되도록 조작될 수 있기 때문입니다. 또한 이 순간 가격은 목표 가격에 도달하기 위해 구매해야 할 `USDs` 계산에 사용됩니다:

```solidity
function calculateUSDsToTargetPrice() private view returns (uint256 _usdc) {
    int24 _spacing = pool.tickSpacing();
    (uint160 _sqrtPriceX96, int24 _tick,,,,,) = pool.slot0();
    int24 _upperTick = _tick / _spacing * _spacing;
    int24 _lowerTick = _upperTick - _spacing;
    uint128 _liquidity = pool.liquidity();
    ...
}
```

따라서 유지 관리 확인 및 이행 간의 지연 시간과 Functions 요청 이행의 추가 지연 시간에도 불구하고 `USDs` 가격은 다중 블록 조작에 의존하지 않고 조작될 수 있습니다. 공격자가 JIT(Just-In-Time) 유동성을 제공하는 것과 결합하여, `AutoRedemption::fulfillRequest` 내에서 제한된 대로 볼트의 전체 부채를 상환하도록 계산을 조작할 수 있습니다:

```solidity
if (_USDsTargetAmount > _vaultData.status.minted) _USDsTargetAmount = _vaultData.status.minted;
```

**Impact:** 볼트 소유자는 `USDs`/`USDC` 풀 준비금을 조작하여 필요하지 않은 경우에도 자동 상환을 강제할 수 있습니다. 오프체인 서비스가 반환한 볼트가 그들의 것이라면, 그들은 수수료를 우회하면서 부채를 상환하게 될 것입니다. 이 분석에서 탐색되지 않은 다른 MEV 공격도 가능할 수 있습니다.

**Recommended Mitigation:** 고립된 가격 오라클 조작의 영향을 최소화하기 위해 `AutoRedemption::performUpkeep` 및 `AutoRedemption::fulfilRequest` 내에서 유지 관리 조건을 다시 확인하십시오. 그러나 반복적인 조작이 반드시 다중 블록 조작을 필요로 하는 것은 아닙니다. 따라서 추가적으로 시간 가중 평균 가격(TWAP)을 소비하거나 Chainlink Labs와 협력하여 분산형 `USDs` 가격 오라클을 생성하는 것을 고려하십시오.

**The Standard DAO:** 커밋 [5ec532e](https://github.com/the-standard/smart-vault/commit/5ec532e5f3813a865102501dbb91cf13a0813930)로 수정됨.

**Cyfrin:** 트리거 조건이 다시 확인되고 TWAP가 구현되었습니다. 그러나 조작을 방지하기 위해 상당히 큰 간격(최소 900초, 1800초가 아니라면)을 사용하는 것이 좋습니다. 참고: Uniswap V3 풀 오라클은 다중 블록 MEV 저항성이 없습니다.

**The Standard DAO:** 커밋 [a8cdc77](https://github.com/the-standard/smart-vault/commit/a8cdc77d1fac9817128e1f3c1c8a1ab57f715513)로 수정됨.

**Cyfrin:** 확인됨. TWAP 간격이 증가했습니다.


### `USDs` redemption calculation can be manipulated due to unsafe signed-unsigned cast

**Description:** 틱 범위를 현재 틱 너머로 진행한 후 `USDs` 상환 금액을 계산할 때 다음 로직이 실행됩니다:

```solidity
} else {
    (, int128 _liquidityNet,,,,,,) = pool.ticks(_lowerTick);
    _liquidity += uint128(_liquidityNet);
    (_amount0,) = LiquidityAmounts.getAmountsForLiquidity(
        _sqrtPriceX96,
        TickMath.getSqrtRatioAtTick(_lowerTick),
        TickMath.getSqrtRatioAtTick(_upperTick),
        _liquidity
    );
}
```

`_liquidityNet` 변수는 틱 범위를 교차할 때의 순 활성 유동성 델타를 나타내므로, 이는 원래 범위의 활성 유동성과 함께 고려됩니다. 그러나 이 값은 부호를 실제로 확인하지 않고 부호 있는 정수에서 부호 없는 정수로 캐스팅됩니다. 음수 부호 있는 정수는 2의 보수를 사용하여 표현되므로, 순 음수 유동성(즉, 후속 범위의 활성 유동성이 적음)은 조용히 증가된 `_liquidity`를 매우 큰 숫자로 만듭니다. 이 시나리오는 우연히 발생하거나 공격자가 JIT(Just-In-Time) 유동성을 추가하여 강제할 수 있으며, 이로 인해 대상 볼트가 최대 `USDs`에 대해 완전히 상환될 수 있습니다:

```solidity
if (_USDsTargetAmount > _vaultData.status.minted) _USDsTargetAmount = _vaultData.status.minted;
```

**Impact:** 최상의 경우, 의도한 것보다 더 많은 부채가 상환될 수 있습니다. 최악의 경우, 이행이 되돌아가게 되어 자동 상환 기능이 영구적으로 비활성화될 수 있습니다(별도 문제에서 설명됨).

**Recommended Mitigation:** `_liquidity`를 증가시키기 전에 `_liquidityNet`의 부호를 확인하고, 음수인 경우 절대값을 빼야 합니다. `LiquidityMath` [라이브러리](https://github.com/Uniswap/v3-core/blob/main/contracts/libraries/LiquidityMath.sol) 사용을 고려하십시오.

**The Standard DAO:** 커밋 [49af007](https://github.com/the-standard/smart-vault/commit/49af007bcce5079ebfdc3de8f0c231ec4d0f121c)로 수정됨.

**Cyfrin:** 확인됨. 이제 `LiquidityMath` 라이브러리가 사용됩니다.


### Incorrect collateral quote amounts due to use of incorrect swap path

**Description:** `AutoRedemption::legacyAutoRedemption` 내에서 필요한 입력 담보의 양은 다음과 같이 계산됩니다:

```solidity
(uint256 _approxAmountInRequired,,,) =
IQuoter(quoter).quoteExactOutput(_collateralToUSDCPath, _USDsTargetAmount);
uint256 _amountIn = _approxAmountInRequired > _collateralBalance ? _collateralBalance : _approxAmountInRequired;
```

이 견적은 목표 `USDs` 금액 계산을 위해 `collateral -> USDC` 스왑 경로를 사용하므로 올바르지 않습니다. 이 로직이 트리거되려면 `USDs`가 페그 아래에 있을 것으로 예상되므로, `_USDsTargetAmount` 양의 `USDC`로 스왑하는 데 `USDs`에 비해 더 많은 담보가 필요할 것입니다. 또한 `Quoter::quoteExactOutput`은 정확한 양의 출력 토큰에 필요한 입력 토큰의 양을 계산할 때 경로가 반전될 것으로 예상합니다.

마찬가지로 `SmartVaultV4::calculateAmountIn`에서 스왑 경로는 `collateral -> USDC`이지만 출력 금액은 `USDs` 목표 금액(잘못된 이름인 `_USDCTargetAmount`로 지정됨)과 비교됩니다:

```solidity
(uint256 _quoteAmountOut,,,) = IQuoter(_quoterAddress).quoteExactInput(_swapPath, _collateralBalance);
return _quoteAmountOut > _USDCTargetAmount
    ? _collateralBalance * _USDCTargetAmount / _quoteAmountOut
    : _collateralBalance;
```

**Impact:** `_amountIn` 변수가 부풀려져 페그를 복원하는 데 필요한 것보다 더 많은 담보가 `USDs`로 스왑될 것입니다.

**Recommended Mitigation:** 반전된 `collateral -> USDs` 스왑 경로를 사용하여 필요한 입력 양을 계산하십시오.

**The Standard DAO:** 커밋 [3c5e136](https://github.com/the-standard/smart-vault/commit/3c5e136cdab97d88e7aa79c23ee18e50d46d6276)으로 수정됨.

**Cyfrin:** 확인됨. 입력 및 출력 경로와 `USDC` 대신 `USDs`를 올바르게 참조하도록 이름이 변경된 관련 스왑 경로 및 목표 금액 변수를 포함하는 `SwapPath` 구조체의 도입으로, 경로가 전달된 대로 구성되는 한(예: 입력: `WETH -> USDC -> USDs`, 출력: `USDs -> USDC -> WETH`) 이제 올바른 것으로 보입니다.


### Auto redemption will not function due to slippage misconfiguration

**Description:** 레거시 및 비레거시 볼트 모두 담보를 `USDs`로 스왑하려고 할 때 `ISwapRouter::exactInput`을 호출하며, `amountOutMinimum` 매개변수는 스왑되는 담보의 USD 가치로 지정됩니다:

```solidity
calculator.tokenToUSD(
    ITokenManager(ISmartVaultManager(manager).tokenManager()).getTokenIfExists(_collateralAddr),
    _collateralAmount
)
```

그러나 이는 스왑 중 발생하는 슬리퍼지를 고려하지 않습니다.

`USDs`가 페그 아래로 내려갈수록 주어진 담보 양에 대해 더 많은 `USDs`가 상환되므로 문제가 발생하지 않을 것입니다. 높은 수수료, 낮은 유동성 풀을 통해 스왑할 때, 특히 `USDs`가 페그에 가까우면 자동 상환이 실패할 가능성이 높습니다.

따라서 자동 상환은 대부분 성공할 가능성이 높지만, 중간 스왑의 가격 영향과 유효 `USDs` 금액의 차이가 무시할 수 없는 경우 입력 담보의 USD 가치와 동등한 `USDs` 금액 출력의 슬리퍼지 요구 사항으로 인해 자동 상환이 가끔 실패할 수 있습니다.

이는 `USDs` 가격이 하락함에 따라 성공할 가능성이 훨씬 높기 때문에 낮은 유동성 스왑 경로를 통해 라우팅해야 하는 볼트에서 상환을 시도하지 않는 한, 효과적인 페그 메커니즘으로서의 자동 상환에 미치는 영향은 미미함을 의미합니다.

**Impact:** 수수료 등급/유동성 및 `USDs` 가격 편차에 따라 자동 상환 기능이 작동하지 않을 수 있습니다.

**Recommended Mitigation:** 두 호출 모두에 덜 제한적인 슬리퍼지 매개변수를 전달하고 실행 끝에 `significantCollateralDrop()` 유효성 검사의 변형을 수행하는 것을 고려하십시오.

**The Standard DAO:** 커밋 [88b2d9f](https://github.com/the-standard/smart-vault/commit/88b2d9f8dcc4f33d1b472ad70d48a708352ee59f)로 수정됨. 꽤 임의적인 버퍼이지만 여기에는 사용자 입력의 가능성이 없습니다. 다시 말하지만, 이상적으로 구매한 `USDs`의 양은 스왑된 담보의 양보다 많아야 합니다.

**Cyfrin:** 확인됨. 낮은 유동성/높은 수수료 풀의 잠재적 문제를 완화하기 위해 버퍼가 적용되었습니다.

\clearpage
## Low Risk


### Concentrated liquidity tick logic is incorrect

**Description:** `AutoRedemption::calculateUSDsToTargetPrice`는 주어진 가격에 해당하는 현재 틱 범위를 결정하기 위해 다음의 집중 유동성 틱 계산을 수행합니다:

```solidity
int24 _spacing = pool.tickSpacing();
(uint160 _sqrtPriceX96, int24 _tick,,,,,) = pool.slot0();
int24 _upperTick = _tick / _spacing * _spacing;
int24 _lowerTick = _upperTick - _spacing;
```

그러나 Solidity의 부호 있는 정수는 다음으로 낮은 정수가 아니라 0으로 반올림되는 동작 때문에, 이 로직은 음수 틱과 `_spacing`의 정확한 배수인 양수 틱에 대해서만 정확합니다. 그렇지 않고 `tick`이 `_spacing`의 양수 비배수(non-multiple)인 경우 계산된 `_upperTick` 및 `_lowerTick`은 실제 현재 범위 바로 아래 범위에 해당합니다.

이 오류로 인해 현재 가격이 목표 가격보다 높은 경우에도 후속 `while` 루프가 실행되거나, 잘못된 조건부 분기로 들어가 현재 틱이 범위 내에 있는 것으로 간주되어야 할 때 순 유동성을 고려하게 될 수 있습니다:

```solidity
while (TickMath.getSqrtRatioAtTick(_lowerTick) < TARGET_PRICE) {
    uint256 _amount0;
    if (_tick > _lowerTick && _tick < _upperTick) {
        (_amount0,) = LiquidityAmounts.getAmountsForLiquidity(
            _sqrtPriceX96,
            TickMath.getSqrtRatioAtTick(_lowerTick),
            TickMath.getSqrtRatioAtTick(_upperTick),
            _liquidity
        );
    } else {
        (, int128 _liquidityNet,,,,,,) = pool.ticks(_lowerTick);
        _liquidity += uint128(_liquidityNet);
        (_amount0,) = LiquidityAmounts.getAmountsForLiquidity(
            _sqrtPriceX96,
            TickMath.getSqrtRatioAtTick(_lowerTick),
            TickMath.getSqrtRatioAtTick(_upperTick),
            _liquidity
        );
    }
    _usdc += _amount0;
    _lowerTick += _spacing;
    _upperTick += _spacing;
}
```

다행히도 `USDs`와 `USDC` 소수점(`18`과 `6` 각각)의 차이로 인해 이 문제가 발생할 가능성은 매우 낮습니다. 두 풀 자산(`token1`/`token0`) 비율의 제곱근을 나타내는 고정 소수점 `Q64.96` 숫자인 `sqrtPriceX96` 계산에서, `USDs`는 `token0`이고 `USDC`는 `token1`입니다. 두 풀 자산의 상대적 가치가 합리적으로 안정적일 것으로 예상되고 `TARGET_PRICE`가 `79228162514264337593543`(`1e-12` 비율에 해당)으로 정의되어 있으므로, 제곱근 가격은 현실적으로 항상 $10^{-6}$ 정도일 것입니다:

$$\text{price ratio (token1/token0)} = \frac{10^{\text{USDC decimals}}}{10^{\text{USDs decimals}}} = \frac{10^{6}}{10^{18}} = 10^{-12}$$

$$\sqrt{\text{price ratio}} = \sqrt{10^{-12}} = 10^{-6}$$

이는 틱이 가격 비율의 로그 인덱스를 나타내고 비율 $10^{-12}$이 1보다 훨씬 작으므로 로그가 음수이기 때문에 풀 틱이 현실적으로 항상 음수여야 함을 의미합니다:

$$\text{tick}=\log_{
\sqrt{(1.0001)}}(\text{price ratio})$$

이 문제가 되려면 비현실적인 규모의 디페깅(de-peg) 이벤트가 필요하지만, `USDC`는 업그레이드 가능하므로 이 주의 사항에 의존해서는 안 됩니다.

**Impact:** `USDs` 계산은 틱 범위 계산의 오류에 영향을 받을 수 있습니다.

**Recommended Mitigation:** 틱 범위를 계산할 때 `_tick`의 부호를 고려하도록 로직을 수정하십시오.

**The Standard DAO:** 인지됨. `AutoRedemption`은 음수 틱의 트리거 가격으로 배포되며 목표 가격은 음수 틱에 있습니다. 유지 관리는 트리거 가격 틱(또는 그 이하)과 목표 가격 틱 사이에서만 필요합니다. 따라서 틱은 양수가 되지 않을 것입니다.

**Cyfrin:** Circle이 `USDC`의 소수점을 업데이트하지 않는다는 가정하에 인지되었습니다.


### Additional validation should be performed on the Chainlink Functions response

**Description:** Chainlink Functions 응답을 소비할 때 `AutoRedemption::fulfillRequest` 내에서 다음 추가 유효성 검사를 수행해야 합니다:
* `_token` 주소가 `SmartVaultYieldManager` 컨트랙트에 유효한 데이터가 있는 Hypervisor(비레거시 볼트에만 유효)이거나 허용된 담보 토큰인지 확인하십시오.
* `SmartVaultManagerV6::vaultData`가 기본값이 아닌 `SmartVaultData` 구조체를 반환하도록 `_tokenID`가 실제로 발행되었는지 확인하십시오.

**The Standard DAO**
커밋 [8ee0921](https://github.com/the-standard/smart-vault/commit/8ee0921026b5a56c0d441f3e87d5530506b7e445)로 수정됨.

**Cyfrin:** 확인됨. 볼트 주소가 0이 아니고 토큰이 유효한 담보 토큰인지 확인하기 위해 `AutoRedemption::validData`가 추가되었습니다.

\clearpage
## Informational


### `AutoRedemption::calculateUSDsToTargetPrice` could be refactored to avoid repeated logic

**Description:** 목표 `USDs` 금액을 계산할 때 `LiquidityAmounts::getAmountsForLiquidity` 호출은 두 조건부 분기에 공통이며 유동성 매개변수만 유일한 차이점입니다:

```solidity
uint128 _liquidity = pool.liquidity();
while (TickMath.getSqrtRatioAtTick(_lowerTick) < TARGET_PRICE) {
    uint256 _amount0;
    if (_tick > _lowerTick && _tick < _upperTick) {
        (_amount0,) = LiquidityAmounts.getAmountsForLiquidity(
            _sqrtPriceX96,
            TickMath.getSqrtRatioAtTick(_lowerTick),
            TickMath.getSqrtRatioAtTick(_upperTick),
            _liquidity
        );
    } else {
        (, int128 _liquidityNet,,,,,,) = pool.ticks(_lowerTick);
        _liquidity += uint128(_liquidityNet);
        (_amount0,) = LiquidityAmounts.getAmountsForLiquidity(
            _sqrtPriceX96,
            TickMath.getSqrtRatioAtTick(_lowerTick),
            TickMath.getSqrtRatioAtTick(_upperTick),
            _liquidity
        );
    }
    ...
}
```

이는 유동성 계산만 조건문에 남도록 반복된 코드를 피하기 위해 리팩터링될 수 있습니다.

**The Standard DAO:** 커밋 [e48ccff](https://github.com/the-standard/smart-vault/commit/e48ccff9f33be2871cfce47312451e03a359f6cc)로 수정됨.

**Cyfrin:** 확인됨. 현재 틱이 범위보다 아래에 있을 때만 순 유동성 델타를 고려하도록 로직이 올바르게 리팩터링되었습니다.


### Incorrectly named return variable `AutoRedemption::calculateUSDsToTargetPrice`

**Description:** `AutoRedemption::calculateUSDsToTargetPrice`는 다음 서명을 가집니다:

```solidity
function calculateUSDsToTargetPrice() private view returns (uint256 _usdc)
```

그러나 명명된 반환 변수는 의미상 올바르지 않으므로 혼란을 피하기 위해 `_usds`여야 합니다.

**The Standard DAO:** 커밋 [a03f0d5](https://github.com/the-standard/smart-vault/commit/a03f0d54637195d34ed17f9a7540d6f201eef55d)로 수정됨.

**Cyfrin:** 확인됨. 반환 변수 이름이 변경되었습니다.


### Unused response parameter should be removed

**Description:** Chainlink Functions 응답에서 디코딩되어 `AutoRedemption::legacyAutoRedemption`에 인수로 전달되는 `_estimatedCollateralValueUSD` 매개변수는 사용되지 않으므로 제거할 수 있습니다:

```solidity
(uint256 _tokenID, address _token, uint256 _estimatedCollateralValueUSD) =
    abi.decode(response, (uint256, address, uint256));
...
legacyAutoRedemption(
    _smartVault, _token, _collateralToUSDCPath, _USDsTargetAmount, _estimatedCollateralValueUSD
);
```

**The Standard DAO:** 커밋 [59c4bd5](https://github.com/the-standard/smart-vault/commit/59c4bd57f1d15c8d8eeb3965368f21e78594186c)로 수정됨.

**Cyfrin:** 확인됨. 매개변수가 제거되었으며 소스/디코딩/서명이 그에 따라 업데이트되었습니다.


### Unused function argument in `SmartVaultYieldManager::quickDeposit` should be removed

**Description:** `SmartVaultYieldManager::quickDeposit` 함수는 다음 서명을 가집니다:

```solidity
function quickDeposit(address _hypervisor, address _collateralToken, uint256 _deposit)
```

그러나 `_hypervisor` 인수는 사용되지 않으므로 제거할 수 있습니다.

**The Standard DAO:** 커밋 [6f943c5](https://github.com/the-standard/smart-vault/commit/6f943c51756cd8023f45af0dee403212fdab096b)로 수정됨.

**Cyfrin:** 확인됨. 사용되지 않은 주소가 제거되었습니다.


### Incorrect `SmartVaultV4` function arguments should be renamed

**Description:** `SmartVaultV4`의 다음 함수들은 `_USDCTargetAmount` 인수를 취합니다:
* `calculateAmountIn()`
* `swapCollateral()`
* `autoRedemption()`

그러나 이는 목표 `USDs` 상환 금액을 나타내려는 의도이므로 의미상 올바르지 않으며 혼란을 피하기 위해 이름이 변경되어야 합니다.

**The Standard DAO:** 커밋 [a03f0d5](https://github.com/the-standard/smart-vault/commit/a03f0d54637195d34ed17f9a7540d6f201eef55d)로 수정됨.

**Cyfrin:** 확인됨. 함수 인수 이름이 변경되었습니다.


### The vault limit condition should not be checked for `address(0)`

**Description:** 가상 함수 `ERC721Upgradeable::_update`가 `SmartVaultManagerV6` 내에서 다음과 같이 재정의되었습니다:

```solidity
function _update(address _to, uint256 _tokenID, address _auth) internal virtual override returns (address) {
    address _from = super._update(_to, _tokenID, _auth);
    require(vaultIDs(_to).length < userVaultLimit, "err-vault-limit");
    smartVaultIndex.transferTokenId(_from, _to, _tokenID);
    if (address(_from) != address(0)) ISmartVault(smartVaultIndex.getVaultAddress(_tokenID)).setOwner(_to);
    emit VaultTransferred(_tokenID, _from, _to);
    return _from;
}
```

그러나 `userVaultLimit` 유효성 검사는 `address(0)`에 대해 수행되어서는 안 됩니다. 그렇지 않으면 이 숫자보다 많은 토큰을 소각(burn)하는 것이 불가능해집니다. 다행히도 이러한 기능은 현재 노출되지 않으며 OpenZeppelin 컨트랙트 내의 검증으로 인해 토큰을 `address(0)`으로 직접 전송하는 것은 불가능합니다. 그럼에도 불구하고 이 엣지 케이스는 사전에 피해야 합니다.

마찬가지로 `SmartVaultIndex::transferTokenId` 내의 `SmartVaultIndex::removeTokenId` 호출은 `_from`이 `address(0)`인 경우 건너뛰어야 하며, `_to`가 `address(0)`인 경우 `tokenIds` 배열에 푸시하는 것은 건너뛰어야 합니다:

```solidity
function transferTokenId(address _from, address _to, uint256 _tokenId) external onlyManager {
    removeTokenId(_from, _tokenId);
    tokenIds[_to].push(_tokenId);
}
```

**The Standard DAO:** 커밋 [ff4ef5b](https://github.com/the-standard/smart-vault/commit/ff4ef5b85561daf24f0443079a570f4bf4bd7389)로 수정됨.

**Cyfrin:** 확인됨. 유효성 검사가 이제 제로 주소에 대해 통과합니다.


### Unused return value can be removed

**Description:** `IRedeemableLegacy::autoRedemption` 함수 서명은 다음과 같습니다:

```solidity
function autoRedemption(
    address _swapRouterAddress,
    address _collateralAddr,
    bytes memory _swapPath,
    uint256 _amountIn
) external returns (uint256 _redeemed);
```

`SmartVaultV4Legact::autoRedemption` 내에서 `_amountOut` 반환 값은 `SmartVaultManagerV6::vaultAutoRedemption`에 의해 버블 업(bubbled-up)됩니다:

```solidity
function vaultAutoRedemption(
    address _smartVault,
    address _collateralAddr,
    bytes memory _swapPath,
    uint256 _collateralAmount
) external onlyAutoRedemption returns (uint256 _amountOut) {
    return IRedeemableLegacy(_smartVault).autoRedemption(swapRouter, _collateralAddr, _swapPath, _collateralAmount);
}
```

그러나 실제로는 사용되지 않으므로 두 함수 서명 모두에서 제거할 수 있습니다.

```solidity
function legacyAutoRedemption(
    address _smartVault,
    address _token,
    bytes memory _collateralToUSDCPath,
    uint256 _USDsTargetAmount,
    uint256 _estimatedCollateralValueUSD
) private {
    ...
    ISmartVaultManager(smartVaultManager).vaultAutoRedemption(_smartVault, _token, _collateralToUSDCPath, _amountIn);
}
```

**The Standard DAO:** 커밋 [2c58fa5](https://github.com/the-standard/smart-vault/commit/2c58fa5759b2d31162f31fdca7c0227a1ef08302)로 수정됨.

**Cyfrin:** 확인됨. 반환 값은 이제 이벤트를 방출하는 데 사용됩니다.


### Redemption of Hypervisor collateral can be suboptimal

**Description:** 자동 상환의 대상이 되는 `20% WBTC`, `40% WETH`, 및 `40% WBTC Hypervisor` 담보(USD 환산 기준)를 가진 볼트를 고려해 보십시오. 현재 로직에서 `WBTC Hypervisor` 주소는 상환될 담보 토큰으로 API 응답에서 수신됩니다. 그런 다음 `_token` 변수는 `hypervisorCollaterals` 매핑(`WBTC` 또는 `WETH` 중 하나이지만 둘 다는 아님)에 저장된 기본 토큰으로 재할당됩니다:

```solidity
address _hypervisor;
if (hypervisorCollaterals[_token] != address(0)) {
    _hypervisor = _token;
    _token = hypervisorCollaterals[_hypervisor];
}
IRedeemable(_smartVault).autoRedemption(
    _smartVault, quoter, _token, _collateralToUSDCPath, _USDsTargetAmount, _hypervisor
);
```

매핑이 `WBTC`가 반환되도록 구성된 경우, 가장 최적의 상환을 실행하는 것이 불가능합니다. 이는 API 응답에 담보 토큰과 선택적으로 0이 아닌 Hypervisor 토큰 주소를 모두 전달함으로써 개선될 수 있는 현재 설계의 제한 사항입니다.

**The Standard DAO:** 커밋 [fd1fe84](https://github.com/the-standard/smart-vault/commit/fd1fe846a16729d217514bb601a672f52722a611)로 수정됨.

**Cyfrin:** 응답이 담보 토큰과 Hypervisor 토큰 주소를 모두 포함하도록 수정되었습니다. 그러나 사용되기 전에 서로 올바르게 대응하는지 확인해야 합니다.

**The Standard DAO:** 커밋 [7346460](https://github.com/the-standard/smart-vault/commit/73464606afd58886ecf8fdf6372b6058566400a5)으로 수정됨.

**Cyfrin:** 확인됨. 추가 유효성 검사가 `AutoRedemption::validData`에 추가되었습니다.

\clearpage
