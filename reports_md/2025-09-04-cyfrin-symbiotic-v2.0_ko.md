**수석 감사**

[0kage](https://twitter.com/0kage_eth)

[Farouk](https://x.com/Ubermensh3dot0)

[ChainDefenders](https://x.com/DefendersAudits) ([@1337web3](https://x.com/1337web3) & [@PeterSRWeb3](https://x.com/PeterSRWeb3))

**보조 감사**

[Aleph-v](https://x.com/alpeh_v) (암호학)

---

# 발견 사항

## 중간 위험 (Medium Risk)

### 오라클 장애 시 투표권이 0으로 반환되어 불공정한 투표 결과 초래

**설명:** `PricedTokensChainlinkVPCalc::stakeToVotingPowerAt`은 체인링크(Chainlink) 오라클이 실패하거나 오래된 데이터(stale data)를 반환할 때 조용히(silently) 0의 투표권을 반환합니다. 운영자는 알림 없이 모든 투표권을 잃게 되어 의도하지 않은 투표 결과가 발생합니다.

핵심 문제는 `ChainlinkPriceFeed.getPriceAt()` 함수의 에러 처리에 있습니다:

```solidity
// ChainlinkPriceFeed.sol
function getPriceAt(address aggregator, uint48 timestamp, bool invert, uint48 stalenessDuration)
    public view returns (uint256) {
    (bool success, RoundData memory roundData) = getPriceDataAt(aggregator, timestamp, invert, stalenessDuration);
    return success ? roundData.answer : 0; // @audit 조용히 0 반환
}
```

이 0 값은 투표권 계산 체인을 통해 전파됩니다:

```solidity
// PricedTokensChainlinkVPCalc.sol
function stakeToVotingPowerAt(address vault, uint256 stake, bytes memory extraData, uint48 timestamp)
	public view virtual override returns (uint256) {
	return super.stakeToVotingPowerAt(vault, stake, extraData, timestamp) * getTokenPriceAt(_getCollateral(vault), timestamp); // @audit 오라클이 오래된 경우 0 반환
}
```

**파급력:** 투표 시점에 오라클이 오래된 데이터를 반환하면, 운영자는 활성 지분이 있음에도 불구하고 모든 투표권을 잃을 수 있습니다.

**개념 증명 (Proof of Concept):** test->modules->common->voting-power-calc에 다음 PoC를 추가하십시오:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.25;

import "forge-std/Test.sol";

import {VotingPowerProvider} from "../../../../../src/contracts/modules/voting-power/VotingPowerProvider.sol";
import {VotingPowerProviderLogic} from
    "../../../../../src/contracts/modules/voting-power/logic/VotingPowerProviderLogic.sol";
import {MultiToken} from "../../../../../src/contracts/modules/voting-power/extensions/MultiToken.sol";
import {IVotingPowerProvider} from "../../../../../src/interfaces/modules/voting-power/IVotingPowerProvider.sol";
import {INetworkManager} from "../../../../../src/interfaces/modules/base/INetworkManager.sol";
import {IOzEIP712} from "../../../../../src/interfaces/modules/base/IOzEIP712.sol";
import {NoPermissionManager} from "../../../../../test/mocks/NoPermissionManager.sol";
import {PricedTokensChainlinkVPCalc} from
    "../../../../../src/contracts/modules/voting-power/common/voting-power-calc/PricedTokensChainlinkVPCalc.sol";
import {OperatorVaults} from "../../../../../src/contracts/modules/voting-power/extensions/OperatorVaults.sol";
import {ChainlinkPriceFeed} from
    "../../../../../src/contracts/modules/voting-power/common/voting-power-calc/libraries/ChainlinkPriceFeed.sol";

import {BN254} from "../../../../../src/contracts/libraries/utils/BN254.sol";
import {AggregatorV3Interface} from "@chainlink/contracts/src/v0.8/shared/interfaces/AggregatorV3Interface.sol";
import "../../../../InitSetup.sol";

// 오래된 데이터 조건을 시뮬레이션하는 모의 Chainlink Aggregator
contract MockChainlinkAggregator is AggregatorV3Interface {
    uint80 private _roundId;
    int256 private _answer;
    uint256 private _startedAt;
    uint256 private _updatedAt;
    uint80 private _answeredInRound;
    uint8 private _decimals;

    bool public isStale;
    bool public shouldRevert;

    constructor(uint8 decimals_) {
        _decimals = decimals_;
        _roundId = 1;
        _answer = 2000e8; // $2000 가격
        _startedAt = block.timestamp;
        _updatedAt = block.timestamp;
        _answeredInRound = 1;
        isStale = false;
        shouldRevert = false;
    }

    function decimals() external view override returns (uint8) {
        return _decimals;
    }

    function description() external pure override returns (string memory) {
        return "Mock Chainlink Aggregator";
    }

    function version() external pure override returns (uint256) {
        return 1;
    }

    function getRoundData(uint80 roundId) external view override
        returns (uint80, int256, uint256, uint256, uint80) {
        if (shouldRevert) {
            revert("Oracle failure");
        }
        return (_roundId, _answer, _startedAt, _updatedAt, _answeredInRound);
    }

    function latestRoundData() external view override
        returns (uint80, int256, uint256, uint256, uint80) {
        if (shouldRevert) {
            revert("Oracle failure");
        }

        uint256 updatedAt = _updatedAt;
        if (isStale) {
            // 매우 오래된 타임스탬프로 설정하여 데이터가 오래된 것처럼 보이게 함
            updatedAt = block.timestamp - 1 days;
        }

        return (_roundId, _answer, _startedAt, updatedAt, _answeredInRound);
    }

    // 오라클 조건을 시뮬레이션하기 위한 테스트 도우미
    function setStale(bool _isStale) external {
        isStale = _isStale;
    }

    function setShouldRevert(bool _shouldRevert) external {
        shouldRevert = _shouldRevert;
    }

    function setPrice(int256 newPrice) external {
        _answer = newPrice;
        _roundId++;
        _updatedAt = block.timestamp;
        _answeredInRound = _roundId;
    }

    function setUpdatedAt(uint256 newUpdatedAt) external {
        _updatedAt = newUpdatedAt;
    }
}

// PricedTokensChainlinkVPCalc를 사용하는 테스트 투표권 제공자
contract TestVotingPowerProvider is VotingPowerProvider, PricedTokensChainlinkVPCalc, NoPermissionManager {
    constructor(address operatorRegistry, address vaultFactory) VotingPowerProvider(operatorRegistry, vaultFactory) {}

    function initialize(
        IVotingPowerProvider.VotingPowerProviderInitParams memory votingPowerProviderInit
    ) external initializer {
        __VotingPowerProvider_init(votingPowerProviderInit);
    }

    function registerOperator(address operator) external {
        _registerOperator(operator);
    }

    function unregisterOperator(address operator) external {
        _unregisterOperator(operator);
    }

    function setSlashingWindow(uint48 sw) external {
        _setSlashingWindow(sw);
    }

    function registerToken(address token) external {
        _registerToken(token);
    }

    function unregisterToken(address token) external {
        _unregisterToken(token);
    }

    function registerSharedVault(address vault) external {
        _registerSharedVault(vault);
    }

    function unregisterSharedVault(address vault) external {
        _unregisterSharedVault(vault);
    }

    function registerOperatorVault(address operator, address vault) external {
        _registerOperatorVault(operator, vault);
    }

    function unregisterOperatorVault(address operator, address vault) external {
        _unregisterOperatorVault(operator, vault);
    }

    // 테스트를 위해 setTokenHops 함수 노출
    function setTokenHops(
        address token,
        address[2] memory aggregators,
        bool[2] memory inverts,
        uint48[2] memory stalenessDurations
    ) public override {
        _setTokenHops(token, aggregators, inverts, stalenessDurations);
    }
}

contract PricedTokensChainlinkVPCalcTest is InitSetupTest {
    TestVotingPowerProvider private votingPowerProvider;
    MockChainlinkAggregator private mockAggregator;
    address private vault;

    address operator1 = address(0xAAA1);
    address operator2 = address(0xAAA2);

    uint256 private constant STAKE_AMOUNT = 1000e18; // 1000 tokens
    uint48 private constant STALENESS_DURATION = 1 hours;

     function setUp() public override {
        InitSetupTest.setUp();

        address token = initSetupParams.masterChain.tokens[0];
        vault = initSetupParams.masterChain.vaults[0];

        // 모의 aggregator 배포 (대부분의 Chainlink 피드처럼 8 소수점)
        mockAggregator = new MockChainlinkAggregator(8);

        // 테스트용 모의 볼트 생성
        vault = makeAddr("vault");

        // 투표권 제공자 배포
        votingPowerProvider = new TestVotingPowerProvider(
            address(symbioticCore.operatorRegistry),
            address(symbioticCore.vaultFactory)
        );

          // 투표권 제공자 초기화
        INetworkManager.NetworkManagerInitParams memory netInit =
            INetworkManager.NetworkManagerInitParams({network: vars.network.addr, subnetworkID: IDENTIFIER});
        IVotingPowerProvider.VotingPowerProviderInitParams memory initParams = IVotingPowerProvider
            .VotingPowerProviderInitParams({
                networkManagerInitParams: netInit,
                ozEip712InitParams: IOzEIP712.OzEIP712InitParams({name: "MyVotingPowerProvider", version: "1"}),
                slashingWindow: 100,
                token: token
        });

        votingPowerProvider.initialize(initParams);
        _networkSetMiddleware_SymbioticCore(vars.network.addr, address(votingPowerProvider));


        _registerOperator_SymbioticCore(symbioticCore, operator1);
        _registerOperator_SymbioticCore(symbioticCore, operator2);

         vm.startPrank(operator1);
        votingPowerProvider.registerOperator(operator1);
        vm.stopPrank();

        vm.startPrank(operator2);
        votingPowerProvider.registerOperator(operator2);
        vm.stopPrank();

          // 올바른 매개변수로 operator1을 위한 적절한 볼트 생성
        vault = _getVault_SymbioticCore(
            VaultParams({
                owner: operator1,
                collateral: token,
                burner: 0x000000000000000000000000000000000000dEaD,
                epochDuration: votingPowerProvider.getSlashingWindow() * 2,
                whitelistedDepositors: new address[](0),
                depositLimit: 0,
                delegatorIndex: 2, // OPERATOR_SPECIFIC delegator
                hook: address(0),
                network: address(0),
                withSlasher: true,
                slasherIndex: 0,
                vetoDuration: 1
            })
        );

         _operatorOptIn_SymbioticCore(operator1, vault);

          _networkSetMaxNetworkLimit_SymbioticCore(
            votingPowerProvider.NETWORK(),
            vault,
            votingPowerProvider.SUBNETWORK_IDENTIFIER(),
            type(uint256).max
        );

        _curatorSetNetworkLimit_SymbioticCore(
            operator1, vault, votingPowerProvider.SUBNETWORK(), type(uint256).max
        );

        _deal_Symbiotic(token, getStaker(0).addr, STAKE_AMOUNT *10, true); // some big number
        _stakerDeposit_SymbioticCore(getStaker(0).addr, vault, STAKE_AMOUNT);

        vm.startPrank(vars.network.addr);
        votingPowerProvider.registerOperatorVault(operator1, vault);
        vm.stopPrank();

          // 모의 aggregator로 토큰 가격 hops 설정
        address[2] memory aggregators = [address(mockAggregator), address(0)];
        bool[2] memory inverts = [false, false];
        uint48[2] memory stalenessDurations = [STALENESS_DURATION, 0];

        votingPowerProvider.setTokenHops(token, aggregators, inverts, stalenessDurations);

     }

     function test_NormalPriceReturnsCorrectVotingPower() public {
        // 오라클 가격 설정 = 2000 (8 소수점에서 2000e8)
        mockAggregator.setPrice(2000e8);

        // 투표권 계산
        uint256 votingPower = votingPowerProvider.stakeToVotingPower(vault, STAKE_AMOUNT, "");

        assertEq(votingPower, (STAKE_AMOUNT * 10**(24-18)) * 2000*10**8*10**(18-8), "Voting power should be stake multiplied by price"); // 10^24로 정규화 스케일링

        // 2000*10**8*10**(18-8) -> chainlink 가격은 18 소수점으로 스케일링됨
        // 투표권은 24 소수점으로 정규화됨
     }

    function test_StaleOracleReturnsZeroVotingPower() public {
        // 오라클을 오래된 상태(stale)로 설정
        mockAggregator.setStale(true);

        // 투표권 계산
        uint256 votingPower = votingPowerProvider.stakeToVotingPower(vault, STAKE_AMOUNT, "");

        // 오래된 데이터로 인해 투표권은 0임 - 투표 결과를 조용히 왜곡함
        assertEq(votingPower, 0, "Voting power should be zero when oracle data is stale");
    }

}
```

**권장 완화 방법:** 0을 반환하는 대신 마지막으로 알려진 양호한 가격을 반환하는 것을 고려하십시오. 마지막으로 알려진 양호한 가격으로 인한 투표권 부정확성은 투표권을 0으로 만드는 것보다 훨씬 낫습니다.

**Symbiotic:** 인지함.

**Cyfrin:** 인지함.

### 정밀도 손실로 인해 소수점 자릿수가 많은 토큰의 투표권이 체계적으로 과소 보고됨

**설명:** `NormalizedTokenDecimalsVPCalc::_normalizeVaultTokenDecimals()`는 소수점 자릿수가 많은 토큰(24 소수점 초과)을 사용할 때 합법적인 이해 관계자의 투표권 손실을 유발합니다. 이는 `Scaler.scale()` 함수에서의 정수 나눗셈으로 인해 발생하며, 작은 결과는 0으로 내림됩니다.

```solidity
// NormalizedTokenDecimalsVPCalc.sol
function _normalizeVaultTokenDecimals(address vault, uint256 votingPower) internal view virtual returns (uint256) {
    return votingPower.scale(IERC20Metadata(_getCollateral(vault)).decimals(), BASE_DECIMALS); //@audit BASE_DECIMALS = 24
}

//Scaler.sol
function scale(uint256 value, uint8 decimals, uint8 targetDecimals) internal pure returns (uint256) {
    if (decimals > targetDecimals) {
        uint256 decimalsDiff;
        unchecked {
            decimalsDiff = decimals - targetDecimals;
        }
        return value / 10 ** decimalsDiff;  // @audit 정밀도 손실
    }
    // ...
}
```
decimals > BASE_DECIMALS (24)일 때, 함수는 10^(decimals-24)로 나누어, 이 제수(divisor)보다 작은 지분은 0의 투표권이 됩니다.

```text
// 제수 = 10^(30-24) = 10^6 = 1,000,000
지분: 999,999 단위 → 투표권: 0
지분: 500,000 단위 → 투표권: 0
지분: 1,000,000 단위 → 투표권: 1
```

`NormalizedTokenDecimalsVPCalc`를 확장하는 모든 투표권 계산기가 영향을 받습니다.

**파급력:** 의미 있는 경제적 지분을 가진 합법적인 이해 관계자가 투표권을 잃을 수 있습니다.

**개념 증명 (Proof of Concept):** `NormalizedTokenDecimalsVPCalc.t.sol`에 다음을 추가하십시오:

```solidity
function test_PrecisionLossPoC_30Decimals() public {
    // 30 소수점으로 더 나쁜 케이스
    votingPowerProvider =
        new TestVotingPowerProvider(address(symbioticCore.operatorRegistry), address(symbioticCore.vaultFactory));

    INetworkManager.NetworkManagerInitParams memory netInit =
        INetworkManager.NetworkManagerInitParams({network: vars.network.addr, subnetworkID: IDENTIFIER});

    // 30 소수점 토큰 생성 (BASE_DECIMALS=24보다 6개 더 많음)
    MockToken mockToken = new MockToken("MockToken", "MTK", 30);

    IVotingPowerProvider.VotingPowerProviderInitParams memory votingPowerProviderInit = IVotingPowerProvider
        .VotingPowerProviderInitParams({
        networkManagerInitParams: netInit,
        ozEip712InitParams: IOzEIP712.OzEIP712InitParams({name: "MyVotingPowerProvider", version: "1"}),
        slashingWindow: 100,
        token: address(mockToken)
    });

    votingPowerProvider.initialize(votingPowerProviderInit);
    _networkSetMiddleware_SymbioticCore(vars.network.addr, address(votingPowerProvider));

    // 극심한 정밀도 손실 시연
    // 제수 = 10^(30-24) = 10^6 = 1,000,000

    Vm.Wallet memory operator = getOperator(0);
    vm.startPrank(operator.addr);
    votingPowerProvider.registerOperator(operator.addr);
    vm.stopPrank();

    address operatorVault = _getVault_SymbioticCore(
        VaultParams({
            owner: operator.addr,
            collateral: address(mockToken),
            burner: 0x000000000000000000000000000000000000dEaD,
            epochDuration: votingPowerProvider.getSlashingWindow() * 2,
            whitelistedDepositors: new address[](0),
            depositLimit: 0,
            delegatorIndex: 2,
            hook: address(0),
            network: address(0),
            withSlasher: true,
            slasherIndex: 0,
            vetoDuration: 1
        })
    );

    _operatorOptIn_SymbioticCore(operator.addr, operatorVault);
    _networkSetMaxNetworkLimit_SymbioticCore(
        votingPowerProvider.NETWORK(),
        operatorVault,
        votingPowerProvider.SUBNETWORK_IDENTIFIER(),
        type(uint256).max
    );
    _curatorSetNetworkLimit_SymbioticCore(
        operator.addr, operatorVault, votingPowerProvider.SUBNETWORK(), type(uint256).max
    );

    _deal_Symbiotic(address(mockToken), getStaker(0).addr, type(uint128).max, true);

    // 999,999 단위 예금 - 상당한 경제적 가치이지만 1,000,000 제수보다 작음
    _stakerDeposit_SymbioticCore(getStaker(0).addr, operatorVault, 999_999);

    vm.startPrank(vars.network.addr);
    votingPowerProvider.registerOperatorVault(operator.addr, operatorVault);
    vm.stopPrank();

    address[] memory operatorVaults = votingPowerProvider.getOperatorVaults(operator.addr);
    uint256 actualVotingPower = votingPowerProvider.getOperatorVotingPower(operator.addr, operatorVaults[0], "");

    console.log("30-decimal token test:");
    console.log("  Stake: 999,999 units (substantial economic value)");
    console.log("  Divisor: 1,000,000 (10^6)");
    console.log("  Actual Voting Power: %d", actualVotingPower);
    console.log("  This demonstrates COMPLETE voting power loss for legitimate stakeholders!");

    // 이것은 실패하여 정밀도 손실을 보여줄 것입니다.
    assertEq(actualVotingPower, 0, "Voting power is 0 due to precision loss - this is the BUG!");
}
```

**권장 완화 방법:** 24 소수점을 초과하는 토큰을 제한하는 것을 고려하십시오. 대안으로, `_normalizeVaultTokenDecimals`를 수정하여 0이 아닌 지분에 대해 최소 1의 투표권을 보장하십시오:

```solidity
function _normalizeVaultTokenDecimals(address vault, uint256 votingPower) internal view virtual returns (uint256) {
    uint8 tokenDecimals = IERC20Metadata(_getCollateral(vault)).decimals();

    if (tokenDecimals > BASE_DECIMALS) {
        uint256 scaled = votingPower.scale(tokenDecimals, BASE_DECIMALS);
        return (scaled == 0 && votingPower > 0) ? 1 : scaled; //@audit 최소 투표권 부여
    }

    return votingPower.scale(tokenDecimals, BASE_DECIMALS);
}
```

**Symbiotic:** 인지함.

**Cyfrin:** 인지함.

### `OpNetVaultAutoDeploy`의 불완전한 검증으로 인해 유효하지 않은 버너(burner) 구성 허용, 자동 볼트 배포로 운영자 등록 시 문제 발생

**설명:** `OpNetVaultAutoDeploy._validateConfig()`는 배포 실패로 이어질 수 있는 버너 주소 검증이 누락되었습니다. 이 함수는 `isBurnerHook`이 활성화되어 있으나 버너가 `address(0)`인 구성 시나리오를 검증하지 못합니다.

이러한 검증 공백으로 인해 관리자는 검증은 통과하지만 슬래셔(slasher) 초기화 중 모든 후속 볼트 배포를 실패하게 만드는 잘못된 구성을 설정할 수 있습니다. 검증 로직은 슬래셔 없이 버너 훅이 활성화된 경우만 확인하고, 슬래셔가 활성화되고 버너 훅도 활성화되었으나 버너 주소가 0인 경우는 검증하지 못합니다.

```solidity
//OpNetVaultAutoDeployLogic

 function _validateConfig(
        IOpNetVaultAutoDeploy.AutoDeployConfig memory config
    ) public view {
        if (config.collateral == address(0)) {
            revert IOpNetVaultAutoDeploy.OpNetVaultAutoDeploy_InvalidCollateral();
        }
        if (config.epochDuration == 0) {
            revert IOpNetVaultAutoDeploy.OpNetVaultAutoDeploy_InvalidEpochDuration();
        }
        uint48 slashingWindow = IVotingPowerProvider(address(this)).getSlashingWindow();
        if (config.epochDuration < slashingWindow) {
            revert IOpNetVaultAutoDeploy.OpNetVaultAutoDeploy_InvalidEpochDuration();
        }
        if (!config.withSlasher && slashingWindow > 0) {
            revert IOpNetVaultAutoDeploy.OpNetVaultAutoDeploy_InvalidWithSlasher();
        }
        if (!config.withSlasher && config.isBurnerHook) {
            revert IOpNetVaultAutoDeploy.OpNetVaultAutoDeploy_InvalidBurnerHook();
        } //@audit 버너 주소에 대한 확인 누락
    }

```

다음은 Symbiotic 코어의 `BaseSlasher` 초기화 로직입니다:

```solidity
//BaseSlasher
  function _initialize(
        bytes calldata data
    ) internal override {
        (address vault_, bytes memory data_) = abi.decode(data, (address, bytes));

        if (!IRegistry(VAULT_FACTORY).isEntity(vault_)) {
            revert NotVault();
        }

        __ReentrancyGuard_init();

        vault = vault_;

        BaseParams memory baseParams = __initialize(vault_, data_);

        if (IVault(vault_).burner() == address(0) && baseParams.isBurnerHook) {
            revert NoBurner();
        } //@audit 버너가 없으면 볼트 배포가 revert됨

        isBurnerHook = baseParams.isBurnerHook;
    }
```
결과적으로 다음과 같은 (잘못된) 구성은 구성을 설정하는 동안 감지되지 않습니다:

```solidity
AutoDeployConfig memory maliciousConfig = AutoDeployConfig({
    epochDuration: 1000,
    collateral: validToken,
    burner: address(0),     //  제로 주소 버너
    withSlasher: true,      //  슬래셔 활성화
    isBurnerHook: true      //  훅은 활성화되었으나 버너 없음!
});

// @audit 이는 잘못 검증을 통과함
deployer.setAutoDeployConfig(maliciousConfig);
```

**파급력:** 구성이 수정될 때까지 자동 배포 모드에서 모든 새로운 운영자 등록이 실패합니다.

**개념 증명 (Proof of Concept):** `OpNetVaultAutoDeploy.t.sol`에 다음 테스트를 추가하십시오:

```solidity
    function test_SetAutoDeployConfig_InvalidBurnerAddress_WithSlasherAndBurnerHook() public {
    // @audit 이 구성은 검증에 실패해야 하지만 현재는 통과합니다.
    // 슬래셔 활성화, 버너 훅 활성화, 그러나 제로 주소 버너인 구성
    IOpNetVaultAutoDeploy.AutoDeployConfig memory maliciousConfig = IOpNetVaultAutoDeploy.AutoDeployConfig({
        epochDuration: slashingWindow,
        collateral: validConfig.collateral,
        burner: address(0),     // 제로 주소 버너 - 이것이 문제입니다
        withSlasher: true,      // 슬래셔 활성화
        isBurnerHook: true      // 훅 활성화되었으나 유효한 버너 없음!
    });

    // 현재 이는 잘못 검증을 통과합니다 - revert되어야 합니다.
    deployer.setAutoDeployConfig(maliciousConfig);

    // 잘못된 구성이 설정되었는지 확인
    IOpNetVaultAutoDeploy.AutoDeployConfig memory setConfig = deployer.getAutoDeployConfig();
    assertEq(setConfig.burner, address(0));
    assertTrue(setConfig.withSlasher);
    assertTrue(setConfig.isBurnerHook);
}



function test_AutoDeployFailure_WithInvalidBurnerConfig() public {
    // 검증을 통과하지만 배포 실패를 유발하는 잘못된 구성을 설정합니다.
    IOpNetVaultAutoDeploy.AutoDeployConfig memory maliciousConfig = IOpNetVaultAutoDeploy.AutoDeployConfig({
        epochDuration: slashingWindow,
        collateral: validConfig.collateral,
        burner: address(0),     // 제로 주소 버너
        withSlasher: true,      // 슬래셔 활성화
        isBurnerHook: true      // 훅 활성화
    });

    // 이는 검증을 통과해야 합니다.
    deployer.setAutoDeployConfig(maliciousConfig);

    // 자동 배포 활성화
    deployer.setAutoDeployStatus(true);

    // 이제 운영자 등록을 시도합니다 - 이는 볼트 배포 중 실패해야 합니다.
    // 왜냐하면 BaseSlasher._initialize()가 burner가 address(0)이지만 isBurnerHook이 true임을 발견하면 NoBurner()로 revert되기 때문입니다.
    vm.startPrank(operator1);

    // @audit 이 등록은 슬래셔 초기화로 인해 자동 배포 중 실패해야 합니다.
    vm.expectRevert();
    deployer.registerOperator();

    vm.stopPrank();

    // 볼트가 배포되지 않았는지 확인
    address deployedVault = deployer.getAutoDeployedVault(operator1);
    assertEq(deployedVault, address(0));

    // 운영자가 볼트를 가지고 있지 않은지 확인
    address[] memory vaults = deployer.getOperatorVaults(operator1);
    assertEq(vaults.length, 0);
}
```

**권장 완화 방법:** _validateConfig에 다음 확인을 추가하는 것을 고려하십시오:

```solidity
function _validateConfig(
    IOpNetVaultAutoDeploy.AutoDeployConfig memory config
) public view {
    // ... 기존 검증들 ...

    // @audit 버너 주소 검증
    if (config.withSlasher && config.isBurnerHook && config.burner == address(0)) {
        revert IOpNetVaultAutoDeploy.OpNetVaultAutoDeploy_InvalidBurnerAddress();
    }
}
```

**Symbiotic:** [625ea22](https://github.com/symbioticfi/relay-contracts/commit/625ea22865b7b03b61d4bbc6d3cf35a1a1cd81fc)에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
## 낮은 위험 (Low Risk)

### `WeightedTokensVPCalc` 및 `WeightedVaultsVPCalc`의 가중치 값에 대한 범위 확인 누락

**설명:** `WeightedTokensVPCalc` 및 `WeightedVaultsVPCalc` 컨트랙트는 가중치 값을 설정할 때 적절한 범위 확인이 부족하여, 투표권 계산 중 정수 오버플로우가 발생하거나 0 가중치로 인한 투표권의 완전한 소멸이 발생할 수 있습니다.

두 컨트랙트의 가중치 설정 함수는 검증 없이 모든 `uint208` 값을 허용합니다:

```solidity
// WeightedTokensVPCalc.sol
function setTokenWeight(address token, uint208 weight) public virtual checkPermission {
    _setTokenWeight(token, weight);  // 범위 확인 없음
}

// WeightedVaultsVPCalc.sol
function setVaultWeight(address vault, uint208 weight) public virtual checkPermission {
    _setVaultWeight(vault, weight);  // 범위 확인 없음
}
```

투표권 계산은 오버플로우 방지 없이 지분 금액에 이 가중치를 곱합니다:

```solidity
// WeightedTokensVPCalc.sol
function stakeToVotingPower(address vault, uint256 stake, bytes memory extraData)
    public view virtual override returns (uint256) {
    return super.stakeToVotingPower(vault, stake, extraData) * getTokenWeight(_getCollateral(vault)); //@audit 설정된 가중치에 따라 0이 되거나 오버플로우될 수 있음
 }
```

**파급력:** 정수 오버플로우(극도로 큰 가중치)를 유발하거나 한 토큰이 나머지를 완전히 지배하거나 투표권이 완전히 제거(0 가중치)될 수 있습니다.

가중치 설정 함수는 `checkPermission` 제어자로 보호되며 프로덕션 배포에서 네트워크 거버넌스에 의해 제어되지만, 기술적 안전장치는 여전히 중요합니다.

**권장 완화 방법:** 상수 또는 불변(immutable)인 최소 및 최대 가중치를 구현하는 것을 고려하십시오.

```solidity
contract WeightedTokensVPCalc is NormalizedTokenDecimalsVPCalc, PermissionManager {
    uint208 public constant MIN_WEIGHT = 1e6;     // 0이 아닌 최소 가중치
    uint208 public constant MAX_WEIGHT = 1e18;    // 합리적인 최대 가중치

    function setTokenWeight(address token, uint208 weight) public virtual checkPermission {
        require(weight <= MAX_WEIGHT, "Weight exceeds maximum");
        require(weight <= MAX_SAFE_WEIGHT, "Weight risks overflow");

        _setTokenWeight(token, weight);
    }
}
```

**Symbiotic:** [2a8b18d](https://github.com/symbioticfi/relay-contracts/pull/36/commits/2a8b18d8d6bb8b487b8eecf4485758f1d6fe93a5)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### `unwhitelistOperator`는 화이트리스트가 비활성화된 경우에도 상태 변경을 허용하여 일관성 없는 운영자 상태를 유발함

**설명:** `unwhitelistOperator` 함수는 화이트리스트 기능이 **비활성화**된 경우에도 상태 변경(운영자를 화이트리스트에서 제거하고 잠재적으로 등록 해제)을 수행합니다. 이는 **화이트리스트 강제는 명시적으로 활성화되었을 때만 활성화되어야 한다**는 예상 동작을 위반합니다.

화이트리스트가 비활성화된 동안 운영자를 화이트리스트에서 제거하면 컨트랙트의 상태가 조용히 변경됩니다. 나중에 화이트리스트가 다시 활성화되면, 해당 운영자는 화이트리스트에서 제거될 때 화이트리스트 관련 로직이 활성화되지 않았음에도 불구하고 예기치 않게 더 이상 화이트리스트에 포함되지 않게 됩니다.

**파급력:** 화이트리스트 기능이 비활성화되었을 때 `unwhitelistOperator`가 조용히 변경 사항을 처리하여 일관성 없는 상태를 초래합니다. 화이트리스트가 다시 활성화되면, 화이트리스트에서 제거된 운영자가 여전히 등록된 상태로 남아 일관성 없는 상태가 됩니다.

**권장 완화 방법:** 화이트리스트가 비활성화된 경우 `unwhitelistOperator` 실행을 방지하는 것을 고려하십시오. 명시적으로 되돌리거나(revert) 실행을 건너뛰십시오:

```solidity
function unwhitelistOperator(address operator) public virtual checkPermission {
    if (!isWhitelistEnabled()) {
        revert OperatorsWhitelist_WhitelistDisabled( );
    }

    _getOperatorsWhitelistStorage()._whitelisted[operator] = false;

    if (isOperatorRegistered(operator)) {
        _unregisterOperator(operator);
    }

    emit UnwhitelistOperator(operator);
}
```

**Symbiotic:** [32bea5e](https://github.com/symbioticfi/relay-contracts/pull/36/commits/32bea5ee73a1085f68645ef460f8c411c96cfcbe)에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
## 정보성 (Informational)

### `OpNetVaultAutoDeployLogic::getVetoSlasherParams`는 절대 사용되지 않음

**설명:** `OpNetVaultAutoDeployLogic::getVetoSlasherParams`는 절대 사용되지 않는 죽은 코드(dead code)입니다. 또한 `OpNetVaultAutoDeploy`는 `InstantSlasher` 전용으로 설계되었으며 `VetoSlasher` 기능을 활용할 메커니즘이 없다는 점에 주목할 가치가 있습니다.

**권장 완화 방법:** `OpNetVaultAutoDeployLogic` 라이브러리에서 `getVetoSlasherParams` 함수를 제거하는 것을 고려하십시오.

**Symbiotic:** 인지함. 외부 커스터마이징을 단순화하기 위해 변경하지 않음.

**Cyfrin:** 인지함.

### `__NoncesUpgradeable_init`이 호출되지 않음

**설명:** `NoncesUpgradeable` 컨트랙트의 `__NoncesUpgradeable_init` 함수가 `VotingPowerProvider` 추상 컨트랙트 초기화 중에 호출되지 않고 있습니다. `VotingPowerProvider`는 `NoncesUpgradeable`을 상속하므로, 넌스 관련 상태를 초기화하지 못하면 잘못된 넌스 관리로 이어질 수 있으며, 잠재적으로 서명 검증이나 기타 관련 기능에서 재생 방지(replay protection) 문제를 일으킬 수 있습니다.

**권장 완화 방법:** `__NoncesUpgradeable_init()` 호출을 포함하도록 `__VotingPowerProvider_init` 함수를 업데이트하십시오.

```solidity
function __VotingPowerProvider_init(
VotingPowerProviderInitParams memory votingPowerProviderInitParams
) internal virtual onlyInitializing {
__NetworkManager_init(votingPowerProviderInitParams.networkManagerInitParams);
VotingPowerProviderLogic.initialize(votingPowerProviderInitParams);
__OzEIP712_init(votingPowerProviderInitParams.ozEip712InitParams);
__NoncesUpgradeable_init(); // /@audit 넌스 상태를 적절히 초기화하기 위해 이 줄을 추가
}
```

**Symbiotic:** 인지함. 바이트코드 크기를 최적화하기 위해 호출하지 않음. 내부 상태를 초기화하지 않으므로 안전할 것으로 판단됨.

**Cyfrin:** 인지함.

### 중복된 수동 접근 제어 확인

**설명:** `BaseRewards` 및 `BaseSlashing` 컨트랙트 모두에서 함수들이 이미 정의된 `onlyRewarder` 및 `onlySlasher` 제어자를 사용하는 대신 수동 접근 제어 확인(`_checkRewarder` 및 `_checkSlasher`)을 수행합니다. 구체적으로:

* `BaseRewards`의 `distributeStakerRewards` 및 `distributeOperatorRewards`
* `BaseSlashing`의 `slashVault` 및 `executeSlashVault`

이러한 불일치는 코드 가독성을 떨어뜨리고 향후 접근 제어 로직을 누락하거나 중복할 위험을 증가시킵니다.

**권장 완화 방법:** 더 명확하고 일관된 접근 제어 시행을 위해 수동 `_checkRewarder` 및 `_checkSlasher` 호출을 해당 제어자로 교체하십시오:

```solidity
function distributeStakerRewards(...) public virtual onlyRewarder { ... }

function distributeOperatorRewards(...) public virtual onlyRewarder { ... }

function slashVault(...) public virtual onlySlasher returns (...) { ... }

function executeSlashVault(...) public virtual onlySlasher returns (...) { ... }
```

이를 통해 접근 확인이 선언적이고 표준화되며 유지 관리가 쉬워집니다.

**Symbiotic:** 인지함. 바이트코드 크기를 줄이기 위한 의도임.

**Cyfrin:** 인지함.

### `KeyRegistry`의 사용되지 않는 함수

**설명:** `KeyRegistry` 컨트랙트에는 64바이트 키 작업을 처리하도록 설계된 세 가지 내부 메서드가 있습니다:

* `_setKey64(address operator, uint8 tag, bytes memory key)`
* `_getKey64At(address operator, uint8 tag, uint48 timestamp)`
* `_getKey64(address operator, uint8 tag)`

그러나 이러한 함수 중 어느 것도 현재 컨트랙트 구현에서 호출되지 않습니다.

**권장 완화 방법:** 사용되지 않는 메서드(`_setKey64`, `_getKey64`, `_getKey64At`)를 제거하여 코드 명확성을 높이고, 감사 표면(audit surface area)을 줄이며, 잠재적인 죽은 코드를 제거하십시오.

**Symbiotic:** 인지함. 향후 커스터마이징을 위해 의도됨.

**Cyfrin:** 인지함.

### 빈 데이터에 대한 일관성 없는 에러 처리

**설명:** `_getSelector()`에서 데이터가 비어 있을 때 표준 관행이 아닌 `0xEEEEEEEE`를 반환합니다.

```solidity
function _getSelector(bytes memory data) internal pure returns (bytes4 selector) {
    if (data.length == 0) {
        return 0xEEEEEEEE;
    }
    // ...
}
```

**파급력:** `_getSelector` 함수와 상호 작용하는 함수를 혼란스럽게 할 수 있는 예기치 않은 반환 값.

**권장 완화 방법:** 개발자와 향후 감사자가 이것이 예상된 동작인지 알 수 있도록 선택한 동작을 문서화하는 것을 고려하십시오.

**Symbiotic:** [8667780](https://github.com/symbioticfi/relay-contracts/pull/36/commits/8667780b0653a83f68a2cdb1c630ad8582eaa787)에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
## 가스 최적화 (Gas Optimization)

### `PersistentSet` 라이브러리 스토리지 접근

**설명:** `_add`, `_remove`, `_containsAt`, `_contains` 등 여러 함수에서 동일한 `set._statuses[value]`에 대해 여러 번의 비싼 스토리지 읽기(SLOAD ~2100 가스 각각)가 발생합니다. 동일한 스토리지 슬롯에 대한 반복적인 접근은 가스를 낭비합니다.

**권장 완화 방법:** 영향을 받는 모든 함수에서 중복된 SLOAD를 피하기 위해 스토리지 포인터를 캐시하십시오:

```solidity
function _add(Set storage set, uint48 key, bytes32 value) private returns (bool) {
    unchecked {
        Status storage status = set._statuses[value]; // 한 번 캐시
        if (status.isAdded) {
            if (status.isRemoved.latest() == 0) {
                return false;
            }
            status.isRemoved.push(key, 0);
        } else {
            set._elements.push(value);
            status.isAdded = true;
            status.addedAt = key;
        }
        set._length += 1;
        return true;
    }
}

function _containsAt(Set storage set, uint48 key, bytes32 value, bytes memory hint) private view returns (bool) {
    Status storage status = set._statuses[value]; // 한 번 캐시
    return status.isAdded && key >= status.addedAt
        && status.isRemoved.upperLookupRecent(key, hint) == 0;
}

function _contains(Set storage set, bytes32 value) private view returns (bool) {
    Status storage status = set._statuses[value]; // 한 번 캐시
    return status.isAdded && status.isRemoved.latest() == 0;
}
```

이는 중복 스토리지 읽기를 제거하여 함수 호출당 ~2100-4200 가스를 절약합니다.

**Symbiotic:** [dcb1d5a](https://github.com/symbioticfi/relay-contracts/pull/36/commits/dcb1d5a294129ca9eacffd16b41ee4d451ae8519#diff-3e26ce80002242995f03115edf52736f8a38a0b4a1a83a82efef003383b68252)에서 수정되었습니다.

**Cyfrin:** 확인됨.

### `upperLookupRecentCheckpoint`에서의 스토리지 읽기 캐시

**설명:** `Checkpoints`의 `upperLookupRecentCheckpoint`와 같은 함수는 `self._trace._checkpoints.length`에 대해 여러 번의 비싼 스토리지 읽기와 반복적인 `_unsafeAccess` 호출을 수행합니다.

**권장 완화 방법:** 체크포인트 배열 참조와 길이를 캐시하십시오:

```solidity
function upperLookupRecentCheckpoint(
    Trace208 storage self,
    uint48 key
) internal view returns (bool, uint48, uint208, uint32) {
    OZCheckpoints.Checkpoint208[] storage checkpoints = self._trace._checkpoints; // 캐시
    uint256 len = checkpoints.length; // 길이 캐시

    uint256 low = 0;
    uint256 high = len;

    if (len > 5) {
        uint256 mid = len - Math.sqrt(len);
        if (key < _unsafeAccess(checkpoints, mid)._key) { // 캐시된 참조 사용
            high = mid;
        } else {
            low = mid + 1;
        }
    }

    uint256 pos = _upperBinaryLookup(checkpoints, key, low, high);
    // ... 함수의 나머지 부분
}
```

**Symbiotic:** 인지함.

**Cyfrin:** 인지함.

### 루프에서 unchecked 산술 사용 및 배열 길이 캐시 가능

**설명:** `Network::scheduleBatch()` 함수에는 unchecked 산술과 캐시된 배열 길이로 최적화할 수 있는 여러 루프가 있습니다.

**권장 완화 방법:**
```solidity
function scheduleBatch(/*...*/) public virtual override onlyRole(PROPOSER_ROLE) {
    uint256 targetsLength = targets.length;
    if (targetsLength != values.length || targetsLength != payloads.length) {
        revert TimelockInvalidOperationLength(targetsLength, payloads.length, values.length);
    }

    unchecked {
        for (uint256 i; i < targetsLength; ++i) {
            uint256 minDelay = getMinDelay(targets[i], payloads[i]);
            if (delay < minDelay) {
                revert TimelockInsufficientDelay(delay, minDelay);
            }
        }
    }

    bytes32 id = hashOperationBatch(targets, values, payloads, predecessor, salt);
    _scheduleOverriden(id, delay);

    unchecked {
        for (uint256 i; i < targetsLength; ++i) {
            emit CallScheduled(id, i, targets[i], values[i], payloads[i], predecessor, delay);
        }
    }
    // ...
}
```

**Symbiotic:** 인지함.

**Cyfrin:** 인지함.

\clearpage
