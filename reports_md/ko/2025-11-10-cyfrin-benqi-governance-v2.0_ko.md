**Lead Auditors**

[Giovanni Di Siena](https://x.com/giovannidisiena)

[ChainDefenders](https://x.com/defendersaudits) ([0x539](https://x.com/1337web3) & [PeterSR](https://x.com/PeterSRWeb3))

[Jorge](https://x.com/TamayoNft)

**Assisting Auditors**



---

# 발견 사항 (Findings)
## 낮은 위험 (Low Risk)


### DAO가 `enableUpdateVotingPowerHook` 구성을 수정하는 로직을 추가하기 위해 `GaugeRegistrar`를 업그레이드하면 분배 로직이 깨질 수 있음

**설명:** `DistributionManager` 컨트랙트는 `_collectVotes()` 내의 `AddressGaugeVoter` 컨트랙트에서 게이지 투표를 읽을 때 에포크 0을 하드코딩합니다:

```solidity
function _collectVotes() internal view returns (...) {
    ...
    for (uint256 j = 0; j < gaugeAddresses.length; j++) {
        address gaugeAddress = gaugeAddresses[j];
        IGaugeRegistrar.RegisteredGauge memory gaugeInfo = gaugeRegistrar.getGaugeInfo(gaugeAddress);

@>      uint256 votes = gaugeVoter.epochGaugeVotes(0, gaugeAddress);

        gaugesByController[i][j] = IDistributionGaugeVote.GaugeVote({
            gaugeAddress: gaugeAddress,
            qiToken: gaugeInfo.qiToken,
            incentive: gaugeInfo.incentive,
            votes: votes
        });
        voteTotals[i] += votes;
    }
}
```

그러나 `AddressGaugeVoter`는 `enableUpdateVotingPowerHook` 구성 플래그에 따라 조건부로 에포크 0 또는 현재 에포크에 투표를 저장합니다:

```solidity
function _vote(address _account, GaugeVote[] memory _votes) internal {
    ...
    uint256 epoch = getWriteEpochId(); // ← Uses conditional epoch
    AddressVoteData storage voteData = epochVoteData[epoch][_account];
    ...
}

function getWriteEpochId() public view returns (uint256) {
    return enableUpdateVotingPowerHook ? 0 : epochId();
}
```

시스템은 `enableUpdateVotingPowerHook`이 항상 true라고 가정하여 에포크 0에 투표를 저장하지만:
1. 이 설정은 `GAUGE_ADMIN_ROLE`에 의해 `setEnableUpdateVotingPowerHook()` 호출을 통해 변경될 수 있습니다.
2. 배포 또는 런타임 중에 이 가정을 강제하는 유효성 검사가 없습니다.
3. false로 설정되면 투표는 현재 에포크(1, 2, 3, ...)에 저장되지만 `DistributionManager`는 항상 에포크 0에서 읽습니다.

`GAUGE_ADMIN_ROLE`에 의한 `AddressGaugeVoter::setEnableUpdateVotingPowerHook` 호출과 관련하여, `GaugeRegistrar`에는 이 권한이 부여되지만 기능은 현재 구현되지 않았습니다. 그러나 `GAUGE_REGISTRAR_ROLE`은 DAO에 부여되므로 구현을 업그레이드할 수 있으며, 잠재적으로 이 구성이 BENQI에 대해 변경될 수 없다는 가정을 위반하는 일부 로직을 추가할 수 있습니다.

**영향:** DAO에 의해 `enableUpdateVotingPowerHook` 구성이 변경되면 `distribute()`에 대한 모든 호출은 0 에포크에서 투표를 잘못 읽어 보상 계산이 잘못됩니다.

**권장 완화 조치:** 이 구성이 수정되는 것을 명시적으로 금지하거나 `enableUpdateVotingPowerHook`가 false인 경우 조건부로 현재 에포크를 검색하도록 투표 수집 로직을 수정하는 것을 고려하십시오.

**BENQI:** `AddressGaugeVoter`는 `setEnableVotingPower`를 활성화된 상태로 두어야 합니다. 이는 Aragon Escrow 시스템의 기본 설정이기 때문입니다. DAO는 주소 게이지 투표자와 게이지 등록자 모두의 공유 소유자가 될 것이며, 둘 다 UUPS입니다. 이는 등록자가 업그레이드될 수 있고 관리자로서 게이지 등록 범위를 벗어나 투표자에게 관리 호출을 할 수 있음을 의미하는 것이 맞습니다. 실제로 BENQI의 경우 DAO는 동일한 엔터티이므로 한 곳에서 침해되면 다른 곳에서도 어쨌든 침해됩니다. 이 로직은 `DistributionManagerSetup`에서 확인되며, 투표자와 `DistributionManager` 간에 DAO가 일치하지 않으면 되돌립니다(revert). 커밋 [3ab793e](https://github.com/aragon/benqi-governance/pull/25/commits/3ab793e0115d3595f95400b55e43d9e1693da0a8) 및 PR [\#20](https://github.com/aragon/benqi-governance/pull/20)에서 수정됨.

**Cyfrin:** 확인함. 운영 및 관리 `GaugeRegistrar` 역할에 대한 분리가 추가되었습니다. `DistributionManager::_validateCanDistribute`는 이제 `GaugeVoter::enableUpdateVotingPowerHook`가 비활성화된 경우에도 되돌립니다.


### `DistributorManager`에서 인터페이스 준수 검증 누락

**설명:** `DistributionManager` 컨트랙트는 초기화 중 또는 `setBudgetAllocator()` 및 `setModuleForController()`를 통해 설정될 때 `budgetAllocator` 및 컨트롤러 모듈 컨트랙트가 필요한 인터페이스를 구현하는지 현재 검증하지 않습니다. `UnifiedBudgetAllocator` 및 `BenqiCoreModule` 컨트랙트는 모두 `ERC165::supportsInterface`를 구현하여 인터페이스 준수를 선언하지만, 설정자(setter) 함수는 컨트랙트를 유효한 것으로 수락하기 전에 이를 확인하지 않습니다.

```solidity
// DistributionManager.sol
function _setBudgetAllocator(IBudgetAllocator _budgetAllocator) internal {
    if (address(_budgetAllocator) == address(0)) revert InvalidAddress("budgetAllocator");

    // @audit -  missing interface validation check

    address oldAllocator = address(budgetAllocator);
    budgetAllocator = _budgetAllocator;

    emit BudgetAllocatorSet(oldAllocator, address(_budgetAllocator));
}

function _setModuleForController(
    address _rewardController,
    IDistributionModule _module
) internal {
    if (_rewardController == address(0)) revert InvalidAddress("rewardController");
    if (address(_module) == address(0)) revert InvalidAddress("module");

    // @audit -  missing interface validation check

    if (address(rewardControllerToModule[_rewardController]) != address(0)) {
        revert ControllerAlreadyRegistered(_rewardController);
    }

    _activeRewardControllers.add(_rewardController);
    rewardControllerToModule[_rewardController] = _module;

    emit ModuleForControllerSet(_rewardController, address(_module));
}
```

**영향:** 호환되지 않는 컨트랙트를 설정하면 배포 시스템이 완전히 서비스 거부(DoS)될 수 있습니다.

**권장 완화 조치:** 두 설정자 함수 모두에 ERC-165 인터페이스 검증 확인을 추가하십시오:

```soldity
if (!IERC165(address(_budgetAllocator)).supportsInterface(type(IBudgetAllocator).interfaceId)) {
      revert InvalidInterface("budgetAllocator");
}
```

**BENQI:** 인지함.

**Cyfrin:** 인지함.


### `BenqiEcosystemModule` 및 `UnifiedBudgetAllocator`가 완전히 ERC-165를 준수하지 않음

**설명:** `BenqiEcosystemModule` 컨트랙트는 `IDistributionModule`과 `IBenqiEcosystemModuleErrors`를 모두 상속합니다. 인터페이스 감지를 위한 ERC-165 사양에 따르면 `supportsInterface()` 함수는 컨트랙트가 구현하는 모든 인터페이스에 대해 `true`를 반환해야 합니다. 그러나 현재 구현은 `type(IDistributionModule).interfaceId`와 부모 `ERC165` 컨트랙트가 지원하는 인터페이스만 확인하며 `type(IBenqiEcosystemModuleErrors).interfaceId`에 대한 확인을 포함하지 않습니다.

`UnifiedBudgetAllocator` 컨트랙트는 유사하게 `IBudgetAllocator`, `IUnifiedBudgetAllocatorEventsAndErrors`, 그리고 `IERC1822Proxiable`을 구현하는 `UUPSUpgradeable`을 포함한 여러 인터페이스를 상속합니다. `supportsInterface()` 함수는 다시 불완전하여 `IBudgetAllocator` 및 `IERC165`에 대해서만 `true`를 반환하고 `IUnifiedBudgetAllocatorEventsAndErrors` 및 `IERC1822Proxiable`에 대한 지원을 보고하지 않습니다.

**영향:** 인터페이스 감지를 위해 ERC-165를 사용하는 외부 컨트랙트 또는 오프체인 도구는 `BenqiEcosystemModule` 컨트랙트가 `IBenqiEcosystemModuleErrors` 인터페이스를 지원하지 않고 `UnifiedBudgetAllocator` 컨트랙트가 `IUnifiedBudgetAllocatorEventsAndErrors` 또는 `IERC1822Proxiable` 인터페이스를 지원하지 않는다고 잘못 결론 내릴 것입니다. 이는 적절한 인터페이스 검색에 의존하는 다른 시스템과의 상호 작용 실패 또는 오작동을 초래할 수 있습니다. 핵심 컨트랙트 로직은 영향을 받지 않지만 이는 표준 준수를 위반하며 구성 가능성 및 검색 가능성에 문제를 일으킬 수 있습니다.

**권장 완화 조치:** 구현된 모든 인터페이스에 대한 확인을 포함하도록 `supportsInterface()` 함수를 수정하십시오.

```solidity
// BenqiEcosystemModule.sol
function supportsInterface(bytes4 _interfaceId) public view virtual override returns (bool) {
    return
        _interfaceId == type(IDistributionModule).interfaceId ||
        _interfaceId == type(IBenqiEcosystemModuleErrors).interfaceId ||
        super.supportsInterface(_interfaceId);
}

// UnifiedBudgetAllocator.sol
function supportsInterface(bytes4 interfaceId) public view virtual override(DaoAuthorizableUpgradeable, IERC165) returns (bool) {
    return
        interfaceId == type(IBudgetAllocator).interfaceId ||
        interfaceId == type(IUnifiedBudgetAllocatorEventsAndErrors).interfaceId ||
        interfaceId == type(IERC1822Proxiable).interfaceId ||
        interfaceId == type(IERC165).interfaceId ||
}
```

**BENQI:** 인지함.

**Cyfrin:** 인지함.


### `DistributorManager` 내에서 Aragon DAO 작업의 최대 수가 확인되지 않음

**설명:** `DistributionManager`는 보상 분배를 위한 작업을 생성하지만 `dao.execute()`를 호출하기 전에 DAO의 `MAX_ACTIONS` 제한에 대해 현재 검증하지 않습니다. Aragon DAO는 실행당 256개 작업의 하드 제한이 있습니다:

```solidity
// lib/conditions/lib/osx/packages/contracts/src/core/dao/DAO.sol
uint256 internal constant MAX_ACTIONS = 256;

function execute(
        bytes32 _callId,
        Action[] calldata _actions,
        uint256 _allowFailureMap
    )
        external
        override
        nonReentrant
        auth(EXECUTE_PERMISSION_ID)
        returns (bytes[] memory execResults, uint256 failureMap)
    {
        // Check that the action array length is within bounds.
@>      if (_actions.length > MAX_ACTIONS) {
            revert TooManyActions();
        }
        ...
}
```

그러나 `DistributionManager`는 이 제한에 대해 검증하지 않고 작업 수를 계산합니다:

```
 function distribute() external auth(DISTRIBUTOR_ROLE) {
    ...
    IExecutor(address(dao())).execute({
        _callId: _generateCallId(),
        _actions: actions, // @audit - no length validation
        _allowFailureMap: 0
    });
    emit RewardsDistributed(currentEpochId);
}
```

시스템에서 허용되는 게이지 및 컨트롤러의 수에 대해서도 유사하게 제한이 없습니다.

**영향:** 작업 제한에 도달하면 일시적인 DoS가 발생할 수 있습니다.

**권장 완화 조치:** 이 시나리오가 발생할 가능성은 낮지만 `DistributionManager`에서 작업 수를 명시적으로 제한하는 것을 고려하십시오.

**BENQI:** 인지함.

**Cyfrin:** 인지함.


### 실제 분배 타이밍에 관계없이 예산은 항상 전체 `epochDuration`을 사용하여 계산됨

**설명:** `DistributionManager`는 전체 에포크 기간을 기준으로 예산 할당을 계산하고 각 에포크가 끝날 때 보상 컨트롤러에 토큰을 전송합니다. `setRewardSpeedInternal()` 호출을 통해 보상 속도가 업데이트될 때 예산 계산은 전체 에포크 기간 동안 보상이 분배될 것이라고 가정합니다. 그러나 분배 타이밍에 따라 이는 전송된 토큰과 실제로 보상으로 분배된 토큰 간의 불일치를 초래할 수 있습니다.

```solidity
// BenqiCoreModule::calculateMarketBudgetUsed
function calculateMarketBudgetUsed(
    MarketVotes memory _market,
    uint256 _supplySpeed,
    uint256 _borrowSpeed,
    uint256 _epochDuration
) public pure returns (uint256) {
    uint256 budget = 0;

    // Only count budget for registered gauges
    if (_market.hasRegisteredSupplyGauge) {
@>      budget += _supplySpeed * _epochDuration; // @audit - uses full epoch duration
    }

    if (_market.hasRegisteredBorrowGauge) {
@>      budget += _borrowSpeed * _epochDuration; // @audit - uses full epoch duration
    }

    return budget;
}

// DistributionManager::_buildActions
uint256 amountWithBuffer = _applyBuffer(budgetUsed);
if (amountWithBuffer != 0) {
    actions[actionCount++] = Action({
        to: address(rewardToken),
        value: 0,
        data: abi.encodeWithSelector(
            IERC20.transfer.selector,
            controller,
@>          amountWithBuffer // @audit - full epoch budget transferred
        )
    });
}
```

Comptroller에서 속도가 설정되면 이전 속도는 덮어쓰여지고 분배되지 않은 예산은 고려되지 않는 동안 즉각적인 타임스탬프부터 보상이 발생합니다:

```solidity
    function setRewardSpeedInternal(uint8 rewardType, QiToken qiToken, uint newSupplyRewardSpeed, uint newBorrowRewardSpeed) internal {
        uint currentSupplyRewardSpeed = supplyRewardSpeeds[rewardType][address(qiToken)];
        uint currentBorrowRewardSpeed = borrowRewardSpeeds[rewardType][address(qiToken)];

        if (currentSupplyRewardSpeed != 0) {
@>          updateRewardSupplyIndex(rewardType, address(qiToken));
        } else if (newSupplyRewardSpeed != 0) {
            Market storage market = markets[address(qiToken)];
            require(market.isListed, "Market is not listed");

            ...
         }

        ...

        if (currentSupplyRewardSpeed != newSupplyRewardSpeed) {
@>          supplyRewardSpeeds[rewardType][address(qiToken)] = newSupplyRewardSpeed;
            emit SupplyRewardSpeedUpdated(rewardType, qiToken, newSupplyRewardSpeed);
        }

       ...
    }

    // lib/BENQI-Smart-Contracts/lending/Comptroller.sol:1112-1130
    function updateRewardSupplyIndex(uint8 rewardType, address qiToken) internal {
        require(rewardType <= 1, "rewardType is invalid");
        RewardMarketState storage supplyState = rewardSupplyState[rewardType][qiToken];
        uint supplySpeed = supplyRewardSpeeds[rewardType][qiToken];
        uint blockTimestamp = getBlockTimestamp();
@>      uint deltaTimestamps = sub_(blockTimestamp, uint(supplyState.timestamp)); // @audit - accrues from last update
        if (deltaTimestamps > 0 && supplySpeed > 0) {
            uint supplyTokens = QiToken(qiToken).totalSupply();
@>          uint qiAccrued = mul_(deltaTimestamps, supplySpeed); // @audit - deltaTimestamps < epochDuration
            Double memory ratio = supplyTokens > 0 ? fraction(qiAccrued, supplyTokens) : Double({mantissa: 0});
            Double memory index = add_(Double({mantissa: supplyState.index}), ratio);
            rewardSupplyState[rewardType][qiToken] = RewardMarketState({
                index: safe224(index.mantissa, "new index exceeds 224 bits"),
                timestamp: safe32(blockTimestamp, "block timestamp exceeds 32 bits")
            });
        } else if (deltaTimestamps > 0) {
            supplyState.timestamp = safe32(blockTimestamp, "block timestamp exceeds 32 bits");
        }
    }
```

보상 컨트롤러로 전송되는 추가 버퍼 금액도 있고 이러한 불일치가 모든 에포크에서 발생할 수 있다는 사실을 고려할 때, 이러한 무시할 수 없는 손실은 시간이 지남에 따라 누적될 것입니다.

지연 발생 시 소급 분배가 불가능하며, 운영자가 대략 동일한 간격으로 분배를 트리거한다고 가정하지만 현재 완전히 자동화되지 않아 보장되지 않는다는 사실에 근거하여 잠재적인 지연을 설명하기 위해 투표 버퍼가 추가된 것으로 이해됩니다. 그러나 운영자가 정확히 언제 에포크가 끝나기 전에 대략 같은 시간에 분배를 실행할 것으로 예상되는지는 아직 명확하지 않습니다. 그렇다면 이는 예산의 일부가 항상 낭비되고 버퍼 금액 추가로 인해 악화된다는 것을 의미합니다.

이는 아마도 분배가 에포크 종료 후, 예를 들어 다음 에포크에서 투표가 다시 시작되기 전 시간 창에서 발생할 수 없는 한 피할 수 없습니다. 이 경우 운영자는 최적으로 에포크 종료 또는 그 이후를 목표로 해야 합니다. 불행히도 별도로 문서화된 바와 같이, 이 기간 동안 시도된 분배는 투표가 아직 시작되지 않았더라도 다음 분배를 완료된 것으로 잘못 표시하므로 현재 로직에 따르면 불가능합니다. 이상적으로 이 로직은 에포크 N+1의 초기 버퍼 기간 동안 에포크 N에 대한 분배를 지원하도록 수정되어야 합니다.

**영향:** `distribute()`가 호출되는 시점에 따라 전송된 예산의 일부가 절대 분배되지 않을 수 있습니다.

**개념 증명 (Proof of Concept):** 다음 예시를 고려하십시오:

* 0-7일: 투표 활성
* 7일: 투표 종료
* 10일: `distribute()` 호출됨 (투표 종료 3일 후)
    * 계산된 예산: 속도 * 2주 = 1000 토큰
    * Comptroller로 전송된 토큰: 1000 토큰 (추가 버퍼 미포함)
    * 새 속도 설정: 500 토큰/주 (71.43 토큰/일)
* 10-14일: 보상은 71.43 토큰/일로 발생 = ~286 토큰
* 14일: 에포크 N 종료 (그러나 속도는 변경 없이 계속됨)
* 14-21일 (에포크 N+1): 투표 활성, 보상은 여전히 71.43 토큰/일로 발생 = ~500 토큰
* 21일: 투표 종료
* 22일: 다음 `distribute()` 호출됨

* 이전 속도에서의 총 시간: 10일부터 22일 = 12일
* 에포크 N 예산에서 분배된 총액: 71.43 * 12 = ~857 토큰
* 에포크 N에서 낭비됨: 1000 - 857 = ~143 토큰

**권장 완화 조치:** 에포크 N에 대한 분배는 이상적으로 에포크 N+1의 초기 버퍼 기간 동안 이루어져야 합니다.

통합 테스트는 특히 시장 참여자의 청구도 테스트하는 것이 아니라 배포자에게 예상되는 토큰 전송 이상의 분배를 다루지 않습니다. 위에서 설명한 시나리오를 효과적으로 입증하려면 청구 가능 금액 대 전송된 금액에 대한 어설션(assertion)을 구현해야 합니다.

**BENQI:** 배포자의 특성상 실제로 버퍼로 인해 컨트랙트의 일부 자금이 청구되지 않을 수 있음을 인정합니다.

또한 각 보상 컨트롤러 구현은 관리자가 필요한 경우 자금을 회수할 수 있도록 스윕/구조(sweep/rescue) 기능을 정의한다는 점에 유의하십시오. 이는 Comptroller와 배포자 각각에 대해 `_grantQi` 및 `_rescueFunds`입니다.

우리의 생각은 부족한 것보다 약간의 초과가 있는 것이 낫다는 것입니다. 자금은 사용자에게 영향을 주지 않고 비동기적으로 회수할 수 있는 반면, 그 반대는 사실이 아니기 때문입니다.

또한 관리자는 초과분을 줄이기 위해 버퍼를 자유롭게 줄일 수 있습니다.

**Cyfrin:** 인지함.


### 단일 `rewardToken` 설계는 다중 보상 방출 지원과 호환되지 않음

**설명:** `BenqiEcosystemModule`은 QI를 단일 불변 `rewardToken`으로 가정하며 `_processGauge()` 내에서 항상 직접 참조합니다:

```solidity
 function _processGauge(
    IMultiRewardDistributor _distributor,
    GaugeVote memory _gauge,
    uint256 _totalVotes,
    uint256 _budget,
    uint256 _epochDuration,
    uint256 _emissionCap
) internal view returns (ProcessGaugeReturnValue memory returnData) {
    ...

    if (_gauge.incentive == IGaugeRegistrar.Incentive.Supply) {
        returnData.actionData = abi.encodeCall(
            IMultiRewardDistributor._updateSupplySpeed,
@>          (QiToken(_gauge.qiToken), rewardToken, newSpeed)
        );
    } else {
        returnData.actionData = abi.encodeCall(
            IMultiRewardDistributor._updateBorrowSpeed,
@>          (QiToken(_gauge.qiToken), rewardToken, newSpeed)
        );
    }
}
```

그러나 `MultiRewardDistributor`는 `qiToken` 시장당 여러 보상 토큰을 지원합니다:

```solidity
/// @notice The main data storage for this contract, holds a mapping of qiToken to array
//          of market configs
mapping(address => MarketEmissionConfig[]) public marketConfigs;
```

> 각 시장에는 고유한 방출 토큰을 가진 구성 배열이 있으며, 각 토큰은 특정 팀/사용자가 소유합니다. 해당 소유자는 공급 및 대출 방출, 종료 시간 등을 조정할 수 있습니다...

이는 `BenqiEcosystemModule` 내에 인코딩된 호출인 `_updateSupplySpeed()` 및 `_updateBorrowSpeed()`의 로직에서도 관찰할 수 있습니다:

```solidity
    function _updateSupplySpeed(
        QiToken _qiToken,
        address _emissionToken,
        uint256 _newSupplySpeed
    ) external onlyEmissionConfigOwnerOrAdmin(_qiToken, _emissionToken) {
        MarketEmissionConfig storage emissionConfig = fetchConfigByEmissionToken(
@>          _qiToken,
@>          _emissionToken
        );
        ...
    }
```

```solidity
function fetchConfigByEmissionToken(
    QiToken _qiToken,
    address _emissionToken
) internal view returns (MarketEmissionConfig storage) {
    MarketEmissionConfig[] storage configs = marketConfigs[address(_qiToken)];
    for (uint256 index = 0; index < configs.length; index++) {
        MarketEmissionConfig storage emissionConfig = configs[index];
        if (emissionConfig.config.emissionToken == _emissionToken) {
            return emissionConfig;
        }
    }

    revert("Unable to find emission token in qiToken configs");
}
```

**영향:** `BenqiEcosystemModule`은 생성자에서 설정된 단일 토큰에 대해서만 속도를 조정할 수 있습니다. `MultiRewardDistributor`에 존재하고 활성 상태일 수 있더라도 다른 방출 토큰을 구별하거나 제어할 수 없습니다.

**권장 완화 조치:** `BenqiEcosystemModule`과 `Distributor`가 다른 보상 토큰을 전달할 수 있도록 허용하는 것을 고려하십시오.

**BENQI:** 인지함, 이것은 순수하게 QI 배포자입니다.

**Cyfrin:** 인지함.


### 새 에포크의 투표 창 버퍼 내에서 실행될 때 보상 분배가 건너뛰어질 수 있음

**설명:** `DistributionManager::distribute`는 투표가 활성 상태가 아닐 때만 호출할 수 있도록 검증됩니다. 이는 `_validateCanDistribute()`를 호출하여 달성되지만, 각 에포크 시작 시의 `VOTE_WINDOW_BUFFER` 기간을 고려하지 못합니다.

Clock 컨트랙트는 다음과 같은 구조로 에포크를 정의합니다:

* `EPOCH_DURATION`: 2주
* `VOTE_DURATION`: 1주
* `VOTE_WINDOW_BUFFER`: 1시간

```solidity
    function _validateCanDistribute(IClock _clock) internal view {
        if (_activeRewardControllers.length() == 0) revert NoModulesConfigured();
@>      if (_clock.votingActive()) revert VotingStillActive(); // @audit - returns false in buffer period

@>      uint256 currentEpochId = _clock.currentEpoch(); // @audit - returns N+1
@>      if (_isEpochDistributed(currentEpochId)) { // @audit - checks if N+1 distributed, not N
            revert EpochAlreadyDistributed(currentEpochId);
        }
    }
```

각 에포크에 대해 투표는 `T = 1시간`부터 `T = 1주 - 1시간`까지만 활성화됩니다. 그러나 새 에포크가 시작될 때 다음과 같은 1시간 버퍼 기간이 있습니다:

* `votingActive()`가 false 반환
* `currentEpoch()`가 새 에포크 N+1 반환

이는 에포크 N+1의 첫 시간 동안 분배가 발생하도록 잘못 허용하여 해당 에포크에 대한 후속 분배를 방지합니다.

**영향:** 에포크 N+1 시작 시 1시간 창 동안 `distribute()`가 호출되면(예: 에포크 N의 늦은 분배의 경우), 에포크 N과 N+1 모두 절대 분배되지 않습니다. 시스템은 투표가 아직 이루어지기 전에 에포크 N+1을 분배된 것으로 표시합니다.

**개념 증명 (Proof of Concept):** 다음 테스트를 `DistributorManager.distribute.t.sol`에 추가해야 합니다:

```solidity
 function testPoC_bufferWindowSkipsEpoch() public {
    // Setup gauge and votes for epoch 0
    vm.prank(ADMIN);
    address gauge = gaugeRegistrar.registerGauge(
        QI_TOKEN_1, IGaugeRegistrar.Incentive.Supply, address(coreComptroller), "Gauge"
    );
    coreComptroller.setMarketListed(QI_TOKEN_1, true);
    addressGaugeVoter.setEpochGaugeVotes(0, gauge, 1000 ether);

    // Warp to 30 minutes into epoch 1 (in VOTE_WINDOW_BUFFER)
    vm.warp(2 weeks + 30 minutes);

    // Verify preconditions: epoch 1, voting inactive, in buffer period
    assertEq(clock.currentEpoch(), 1);
    assertFalse(clock.votingActive());
    assertLt(clock.elapsedInEpoch(), 1 hours);

    // Call distribute() - should fail but doesn't
    vm.prank(DISTRIBUTOR);
    distributionManager.distribute();

    // BUG: Epoch 1 marked distributed, Epoch 0 skipped forever
    assertFalse(distributionManager.isDistributed(0), "Epoch 0 never distributed!");
    assertTrue(distributionManager.isDistributed(1), "Epoch 1 wrongly marked distributed");
}
```

설명된 대로 에포크 0 보상은 절대 분배되지 않았고 에포크 1은 0 투표와 0 예산으로 분배되었습니다.

**권장 완화 조치:** 투표 창 버퍼 동안 분배를 명시적으로 방지하거나, 가급적이면 이 기간 내에 이전 에포크에 대한 분배를 허용하십시오.

**BENQI:** PR [\#13](https://github.com/aragon/benqi-governance/pull/13)에서 수정됨.

**Cyfrin:** 확인함. 사전 투표 버퍼 기간 동안 더 이상 분배할 수 없습니다.


### 비대칭 게이지 등록 취소로 인해 토큰이 잘못 할당될 수 있음

**설명:** `BenqiCoreModule` 컨트랙트의 보상 속도 보존 로직은 비대칭적으로 등록 취소되거나 만료된 공급/대출 게이지와 관련된 시장에 대해 무제한 QI 토큰 분배를 가능하게 합니다. 구체적으로, `GaugeRegistrar::unregisterGauge`를 통해 게이지가 등록 취소되면 연결된 `AddressGaugeVoter`에서 비활성화되기 전에 내부 추적 세트 `_allGauges` 및 `_gaugesByRewardController`에서 제거되어 향후 투표를 방지합니다. 그러나 `BenqiCoreModule::generateActions`에서 `calculateSpeeds()` 함수는 `hasRegisteredSupplyGauge` 및 `hasRegisteredBorrowGauge` 플래그를 확인합니다. 플래그가 false인 경우 기존 공급 또는 대출 보상 속도는 0으로 설정되는 대신 `CoreComptroller`에서 보존됩니다. 이 의도적인 설계는 보상 속도의 급격한 변화를 피하지만 자동 제로화 메커니즘이 부족하여 거버넌스가 `setRewardSpeed()`를 직접 호출하여 수동으로 개입하지 않는 한 속도가 마지막 0이 아닌 값으로 유지되도록 허용합니다.

**영향:** 비대칭적으로 등록 취소된 게이지가 있는 시장은 QI 보상을 무기한 계속 분배하여 시스템의 예산 에포크 밖에서 토큰을 잘못 할당할 수 있습니다. 이는 시간이 지남에 따라 QI 보상 풀을 고갈시키거나 계획되지 않은 보충을 필요로 할 수 있습니다. 경제적 손실은 시장 활동(예: 공급/대출 규모 트리거 청구), 보존된 속도 수준 및 감지 전 기간에 따라 확장됩니다. 이는 주의 깊은 거버넌스에 의존하는 운영 위험을 도입합니다. 후속 조치 없이 등록 취소가 발생하는 경우(예: DAO 프로세스의 실수로 인해), 외부 행위자가 직접 악용할 수는 없지만 상당한 누적 토큰 유출이 발생할 수 있습니다.

**권장 완화 조치:** 속도 보존 로직 제거를 고려하십시오. 이는 현재 comptroller 속도를 보존하는 대신 `hasRegisteredSupplyGauge` 또는 `hasRegisteredBorrowGauge`가 false일 때 0 또는 `MIN_SPEED`로 설정하여 현재 투표를 기반으로 모든 게이지에 대해 속도를 항상 다시 계산하도록 `BenqiCoreModule::calculateSpeeds`를 수정하여 달성할 수 있습니다.

**BENQI:** 이것은 예상보다 큰 PR로 판명되었습니다. 우리는 먼저 다음 패턴을 통해 등록 취소 후크를 구현했습니다:

등록자는 등록자에 대해 승인된 전용 엔드포인트를 통해 배포 관리자를 호출합니다. 배포 관리자는 관련 모듈을 쿼리하여 등록 취소 작업을 가져옵니다. 배포 관리자는 속도 조정에 대한 DAO 권한이 있으므로 실행합니다.

하지만 이것은 문제를 일으켰습니다. 실제로 존재하는지 확인하지 않고 게이지를 등록할 수 있었습니다. 이는 게이지가 등록될 수 있지만 등록 취소될 수는 없음을 의미하며, 두 경우 모두 목록에 없는 게이지에서 읽으려고 시도하므로 분배가 영구적으로 차단됩니다.

따라서 2가지 안전 조치를 추가했습니다:

1. 등록 중 게이지가 존재하는지 확인합니다. 이는 케이스의 95%를 커버해야 합니다.
2. 등록 취소가 실패할 경우 전용 이벤트가 발생하는 try/catch입니다. 이를 위해서는 관리자가 문제를 이해하기 위해 추가 조치가 필요하지만 기본 사례와 동일합니다.

PR [\#24](https://github.com/aragon/benqi-governance/pull/24)에서 수정됨.

**Cyfrin:** 확인함. 유효한 게이지의 등록 취소는 먼저 `Comptroller`의 현재 속도를 등록 취소 작업의 `MIN_SPEED`로 업데이트하여 향후 할당을 최소화합니다.


### 투표 가중치의 잘못된 손실을 방지하기 위해 보상 분배 전 게이지 제거를 명시적으로 방지해야 함

**설명:** 보상 분배는 투표 기간이 끝난 후 발생하며, Miles 토큰 보유자는 선택한 게이지에 투표권을 할당합니다. 그러나 투표 기간이 끝난 후 분배 단계가 시작되기 전에 게이지가 제거되면 해당 게이지에 대한 보상이 분배되지 않습니다. 결과적으로 제거된 게이지에 대해 던진 모든 투표는 쓸모없게 됩니다.

**영향:** 영향을 받는 에포크 동안 제거된 게이지에 대해 던진 모든 투표는 무효화되어 투표자의 투표 가중치 및 해당 보상이 완전히 손실됩니다.

**권장 완화 조치:** `DistributionManager::_isEpochDistributed`는 에포크가 이미 분배되었는지 여부를 결정합니다. 이 확인은 `DistributionManager` 내에 새로운 메서드를 생성하기 위해 투표가 아직 시작되지 않았다는 조건과 결합될 수 있습니다. 그런 다음 이 새로운 메서드를 `GaugeRegistrar` 내에서 사용하여 이 중요한 간격 동안 게이지가 제거되지 않도록 하여 사용자 투표 손실을 방지하고 적절한 보상 분배를 보장할 수 있습니다.

**BENQI:** 인지함. 이에 대한 해결책은 매우 복잡하고 가치가 없습니다.

한 가지 방법은 시간이 에포크 시작과 투표 시작 사이일 때만 게이지 등록 취소를 허용하는 것이지만, 이는 관리자가 게이지를 제거하려면 그렇게 할 수 있는 매우 구체적인 시간을 기다려야 함을 의미합니다.

**Cyfrin:** 인지함.


### `DistributionManagerSetup::canDistribute`는 되돌리는(revert) 대신 false를 반환해야 함

**설명:** `DistributionManagerSetup::canDistribute`는 현재 분배가 진행될 수 있는 경우 true를 반환하지만 불가능한 경우 `_validateCanDistribute()` 내에서 되돌립니다:

```solidity
    function canDistribute() public view returns (bool) {
        IClock clock = IClock(gaugeVoter.clock());
@>      _validateCanDistribute(clock);
        return true;
    }

    function _validateCanDistribute(IClock _clock) internal view {
@>      if (_activeRewardControllers.length() == 0) revert NoModulesConfigured();
@>      if (_clock.votingActive()) revert VotingStillActive();

        uint256 currentEpochId = _clock.currentEpoch();
        if (_isEpochDistributed(currentEpochId)) {
@>          revert EpochAlreadyDistributed(currentEpochId);
        }
    }
```

되돌리는 대신 이 뷰(view) 함수는 소비자가 신뢰할 수 있게 호출할 수 있도록 false를 반환해야 합니다.

**BENQI:** PR [\#20](https://github.com/aragon/benqi-governance/pull/20)에서 수정됨.

**Cyfrin:** 확인함. 이 함수에 대한 호출은 이제 되돌리거나 빈 반환 데이터로 성공합니다.


### 해당 게이지 주소의 검증 없이 임의의 컨트롤러 주소를 `DistributionManager::setModuleForController`에 전달할 수 있음

**설명:** DAO는 현재 `DistributionManager::setModuleForController` 호출에서 임의의 컨트롤러 주소를 전달할 수 있습니다. 그러나 이 로직은 `_activeRewardControllers` 세트에 잘못된 컨트롤러를 추가하지 않도록 컨트롤러의 게이지가 `GaugeRegistrar` 및 `AddressGaugeVoter`에 등록되어 있는지 추가로 검증해야 합니다.

DAO가 이것을 고의로 잘못 구성할 가능성은 낮으므로 영향은 적습니다. 그럼에도 불구하고 `_validateCanDistribute()`는 해당 컨트롤러에 해당하는 게이지가 아니라 활성 보상 컨트롤러의 길이가 0이 아닌지만 확인하므로 이 시나리오에서 true를 반환할 수 있습니다.

`_collectVotes()` 내에서 `GaugeRegistrar`로부터 게이지 주소를 쿼리하려고 시도하므로 잘못된 컨트롤러 주소의 경우 빈 배열이 항상 반환됩니다.

또한 실행은 활성 컨트롤러 중 유효한 등록된 게이지가 없는 경우 총 수집된 투표가 0이기 때문에 `UnifiedBudgetAllocator::allocateBudget`에서만 되돌립니다.

따라서 DAO가 `GaugeRegistrar`에 등록된 게이지와 이미 연결되지 않은 컨트롤러로 `setModuleForController()`를 호출할 때 실수를 하면 분배가 건너뛰어지고 예상 게이지로 보상이 전송되지 않습니다.

**권장 완화 조치:** 주어진 컨트롤러와 관련된 게이지 주소가 `GaugeRegistrar`에 이미 등록되어 있는지 확인하는 것을 고려하십시오.

**BENQI:** 인지함.

**Cyfrin:** 인지함.

\clearpage
## 정보 (Informational)


### `MIN_SPEED` 및 `DEFAULT_MIN_SPEED`를 공유 상수 라이브러리에 병합해야 함

**설명:** `DistributionManager`, `BenqiCoreModule` 및 `BenqiEcosystemModule` 컨트랙트는 각각 최소 보상 속도 `DEFAULT_MIN_SPEED` 및 `MIN_SPEED`에 대한 상수를 독립적으로 정의합니다. 현재 게이지 투표가 0으로 떨어질 때 최소 보상 흐름을 보장하기 위해 1 wei purpose의 동일한 값을 공유하지만 연결되어 있지는 않습니다.

`DistributionManager`는 `DEFAULT_MIN_SPEED`를 사용하여 허용 가능한 최대 예산 초과를 계산합니다. 반면 모듈은 자체 비공개 `MIN_SPEED` 상수를 사용하여 필요한 실제 예산을 결정하며, 여기에는 이 최소 속도 하한 적용이 포함됩니다.

**영향:** 이 중복은 추가적인 오버헤드를 추가하고 향후 개발 중에 값이 일치하지 않을 위험을 초래합니다. 모듈의 `MIN_SPEED`가 `DistributionManager`의 `DEFAULT_MIN_SPEED`와 다른 값으로 변경되면 값이 더 이상 같지 않게 됩니다.

**권장 완화 조치:** 일관성을 보장하고 유지 관리를 단순화하기 위해 최소 속도 상수는 단일 위치에 정의되어야 합니다:

1.  공유 상수를 보관할 새 라이브러리(예: `Constants.sol`)를 생성합니다.
2.  이 새 라이브러리에 단일 `MIN_SPEED` 상수를 정의합니다.
3.  `DistributionManager`, `BenqiCoreModule` 및 `BenqiEcosystemModule`에서 별도의 `DEFAULT_MIN_SPEED` 및 `MIN_SPEED` 정의를 제거합니다.
4.  세 컨트랙트 모두 새 상수 라이브러리를 가져오고 단일 `MIN_SPEED` 값을 참조하도록 합니다. 이렇게 하면 시스템의 모든 구성 요소가 항상 동일한 매개변수로 작동하여 잠재적인 불일치를 방지할 수 있습니다.

**BENQI:** PR [\#29](https://github.com/aragon/benqi-governance/pull/29)에서 수정됨.

**Cyfrin:** 확인함.


### 비표준 ERC-20 토큰의 안전하지 않은 전송

**설명:** `DistributionManager`는 보상 토큰을 컨트롤러에 분배할 때 안전한 전송(safe transfer) 등가물 대신 표준 ERC-20 `transfer()` 함수를 사용합니다. 보상 토큰(QI)은 ERC-20 사양을 준수하는 알려진 신뢰할 수 있는 컨트랙트이지만, 그렇지 않고 토큰이 표준을 올바르게 준수하지 않는 경우 `transfer()`를 직접 사용하면 조용한 실패(silent failures)로 이어질 수 있습니다.

```solidity
// DistributionManager.sol - Lines 416-424
actions[actionCount++] = Action({
    to: address(rewardToken),
    value: 0,
    data: abi.encodeWithSelector(
        IERC20.transfer.selector, // @audit - uses transfer, not safeTransfer
        controller,
        amountWithBuffer
    )
});
```

표준 `IERC20::transfer`는 다음을 수행할 수 있습니다:
* 되돌리지 않고 실패 시 false 반환 (예: USDT).
* 값을 전혀 반환하지 않음 (예: BNB)

두 경우 모두 트랜잭션은 계속 실행되고 에포크는 분배된 것으로 표시되지만 컨트롤러는 토큰을 받지 못합니다. 이는 분배가 성공한 것처럼 보이지만 토큰이 전송되지 않은 일관성 없는 상태를 생성합니다.

**권장 완화 조치:** 직접적인 ERC-20 전송 대신 확인된 안전한 전송 등가물을 사용하는 것을 고려하십시오.

**BENQI:** 인지함.

**Cyfrin:** 인지함.


### `UniquenessTests`는 호출 식별자 계산에 호출자가 포함되는 것을 잘못 참조함

**설명:** `UniquenessTests::test_CallIdIncludesCaller`는 호출자가 계산에 포함되므로 두 개의 다른 주소에서 호출할 때 호출 식별자가 다를 것이라는 가정을 기반으로 구현되었습니다. 그러나 이는 잘못된 것이며 `generateCallId()` 호출 사이에 nonce가 증가하기 때문에 식별자가 다를 뿐입니다.

**권장 완화 조치:** 이 테스트를 완전히 제거하거나 스냅샷을 사용하여 동일한 nonce에 대해 서로 다른 주소에서 호출될 때 반환된 식별자가 동일한지 검증하도록 수정하십시오.

**BENQI:** PR [\#28](https://github.com/aragon/benqi-governance/pull/28)에서 수정됨.

**Cyfrin:** 확인함.


### 이전 투표자가 재설정하지 않으면 비활성화된 게이지에 대해 투표가 자동으로 다시 전송됨

**설명:** `BenqiCoreModule.calculateSpeeds.t.sol`의 `test_calculateSpeeds_positiveVotesNoGauge()` 함수에는 존재하지 않는 게이지가 0이 아닌 투표를 갖는 시나리오가 실제로는 발생하지 않아야 한다는 가정이 포함되어 있습니다.

그러나 이것은 엄격하게 사실이 아니며 이전 투표자가 다른 게이지에 투표할 때 재설정이 없으면 투표가 자동으로 다시 전송되기 때문에 비활성화된 게이지에 대해 발생할 수 있습니다. 예를 들어 게이지가 `GaugeRegistrar`에서 등록 취소되지 않고 `AddressGaugeVoter`에서 직접 비활성화된 경우, 컨트롤러의 게이지는 `GaugeRegistrar`에서 검색되므로 이 비활성화가 반영되지 않아 등록되지 않은/비활성화된 게이지가 유령 투표권을 가질 수 있습니다. 자동 재전송된 투표는 재설정되지 않으므로, 이러한 방식으로 게이지가 비활성화된 컨트롤러는 계속해서 보상 지분을 받게 됩니다.

**영향:** 배포는 `AddressGaugeVoter`의 게이지 비활성화에 영향을 받지 않습니다.

**개념 증명 (Proof of Concept):** `BenqiIntegrationSingleEpoch.t.sol`에 테스트를 추가해야 합니다.

**권장 완화 조치:** 등록자에게 알리지 않고 투표자에서 게이지가 비활성화되는 것을 방지하는 것을 고려하십시오.

**BENQI:** 인지함.

**Cyfrin:** 인지함.


### 속도 계산 정밀도 처리가 비효율적임

**설명:** `SpeedCalculator::calcSpeed`는 `SPEED_PRECISION` 상수를 적용하여 정밀도 손실을 해결하려고 합니다. 그러나 분자와 분모 모두에 존재하면 이 요소가 즉시 다시 나뉘기 때문에 정밀도 손실에 아무런 도움이 되지 않습니다.

```solidity
function calcSpeed(
    uint256 _votes,
    uint256 _totalVotes,
    uint256 _moduleBudget,
    uint256 _epochDuration
) public pure returns (uint256) {
    if (_totalVotes == 0 || _epochDuration == 0 || _votes == 0) return 0;

    return
        (_moduleBudget * SPEED_PRECISION * _votes) /
        (_epochDuration * _totalVotes * SPEED_PRECISION);
}
```

시장 예산 사용량을 계산할 때 이 스케일링을 반대로 하는 것이 좋습니다. 즉, 먼저 속도와 기간의 곱셈을 수행합니다. 그런 다음에만 `BenqiCoreModule`과 `BenqiEcosystemModule` 간에 로직이 약간 다르게 구현된다는 점을 감안하여 `MIN_SPEED` 및 comptroller/multi-distributor의 현재 속도와 비교할 때와 같이 다른 모든 사용이 현재와 동일하도록 주의하면서 결과를 다시 나누어야 합니다.

**BENQI:** PR [\#16](https://github.com/aragon/benqi-governance/pull/16)에서 수정됨.

**Cyfrin:** 확인함. 권장된 대로 후속 사용에서 스케일링하는 대신 `SPEED_PRECISION` 사용이 완전히 제거되었습니다.

\clearpage
## 가스 최적화 (Gas Optimization)


### 변수를 기본값으로 초기화하는 것을 피하기

**설명:** 범위 내 컨트랙트 전체에 변수가 불필요하게 기본값으로 초기화되는 경우가 많이 있습니다.

**권장 완화 조치:** 가스를 절약하기 위해 변수를 기본값으로 초기화하지 마십시오.

**BENQI:** 인지함.

**Cyfrin:** 인지함.


### `_moduleBudget`이 0이면 `SpeedCalculator::calcSpeed`가 일찍 반환해야 함

**설명:** `SpeedCalculator::calcSpeed`는 현재 `_totalVotes`, `_epochDuration` 또는 `_votes` 매개변수 중 하나라도 0이면 단락(short circuit)됩니다. 이는 불필요한 계산에 가스를 낭비하는 것을 방지하므로 유익하지만, 0을 곱하면 마찬가지로 0이 반환되므로 이 검증에는 `_moduleBudget`도 포함되어야 합니다.

**권장 완화 조치:**
```diff
function calcSpeed(
    uint256 _votes,
    uint256 _totalVotes,
    uint256 _moduleBudget,
    uint256 _epochDuration
) public pure returns (uint256) {
-   if (_totalVotes == 0 || _epochDuration == 0 || _votes == 0) return 0;
+   if (_totalVotes == 0 || _epochDuration == 0 || _votes == 0 || _moduleBudget == 0) return 0;

    return
        (_moduleBudget * SPEED_PRECISION * _votes) /
        (_epochDuration * _totalVotes * SPEED_PRECISION);
}
```

**BENQI:** PR [\#15](https://github.com/aragon/benqi-governance/pull/15)에서 수정됨.

**Cyfrin:** 확인함.


### 명명된 반환 변수와 함께 return 문 사용 피하기

**설명:** `GaugeRegistrar::registerGauge`와 같이 명명된 반환 변수를 선언할 때, 명시적으로 return 문을 실행할 필요가 없으며 이를 제거하여 가스를 절약할 수 있습니다.

**권장 완화 조치:** return 문 제거.

**BENQI:** 인지함.

**Cyfrin:** 인지함.


### `DistributionManager::initialize`의 중복 검증 제거 가능

**설명:** `DistributionManager::initialize`는 `_initialBudgetAllocator`가 `address(0)`과 같으면 되돌립니다. 그러나 이 로직은 중복되며 `_setBudgetAllocator()`에 이미 존재하므로 필요하지 않습니다.

**BENQI:** PR [\#38](https://github.com/aragon/benqi-governance/pull/38)에서 수정됨.

**Cyfrin:** 확인함.


### `BenqiEcosystemModule::_buildActions`의 작업 수 증가 단순화 가능

**설명:** `BenqiEcosystemModule::_buildActions`는 처리에 따라 현재 게이지를 건너뛰지 않아야 함을 나타낼 때 작업 수를 증가시킵니다.

그러나 이 로직은 한 줄로 단순화할 수 있습니다:

```solidity
if (!prgv.shouldSkip) [actionCount++] = Action(_target, 0, prgv.actionData);
```

**BENQI:** 인지함.

**Cyfrin:** 인지함.

\clearpage

