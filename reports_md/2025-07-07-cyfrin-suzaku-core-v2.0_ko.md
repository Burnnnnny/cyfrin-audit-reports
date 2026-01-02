**Lead Auditors**

[0kage](https://twitter.com/0kage_eth)

[Farouk](https://x.com/Ubermensh3dot0)

ChainDefenders ([@1337web3](https://x.com/1337web3) & [@PeterSRWeb3](https://x.com/PeterSRWeb3))

**Assisting Auditors**



---

# Findings
## Critical Risk


### Dust limit attack on `forceUpdateNodes` allows DoS of rebalancing and potential vault insolvency

**Description:** 공격자는 `forceUpdateNodes()` 함수를 악용하여 최소한의 `limitStake` 값으로 모든 검증 노드를 업데이트 대기 상태로 만들 수 있습니다. 공격자는 지분 대 가중치 변환 시의 정밀도 손실을 악용하여 최소한의 limitStake(예: 1 wei)로 `forceUpdateNodes`를 호출할 수 있으며, 이는 실제 지분을 줄이지 않고 리밸런싱 플래그를 설정하여 전체 에포크 동안 합법적인 리밸런싱을 효과적으로 차단합니다.

다음 시나리오를 고려하십시오:
- 대규모 볼트 언델리게이션(undelegation) 요청으로 인해 볼트 활성 상태가 감소함 -> 이로 인해 리밸런싱해야 할 초과 지분이 생성됨
- 공격자가 `dustAmount`(1 wei)로 `forceUpdateNodes(operator, dustAmount)`를 호출함
- 최소 지분(≤ 1 wei)이 모든 사용 가능한 노드에서 "제거"됨
- WEIGHT_SCALE_FACTOR 정밀도로 인해 P-Chain으로 전송된 실제 가중치는 변경되지 않지만 P-Chain은 여전히 업데이트를 처리함. 사실상 비-업데이트(non-update)를 처리하고 있음.
- 해당 노드는 `nodePendingUpdate[validationID] = true`로 표시됨
- 동일한 에포크 내의 모든 후속 합법적 리밸런싱 시도는 되돌려짐(revert)

`forceUpdateNodes()`는 각 에포크의 마지막 기간 동안 누구나 호출할 수 있으며 `limitStake`에는 최소 경계 확인이 없습니다.

**Impact:**
- 초과 지분을 보유한 운영자는 이를 악용하여 자격이 있는 것보다 더 많은 지분을 유지할 수 있습니다.
- 공격자는 모든 에포크에서 모든 운영자에 대한 리밸런싱을 체계적으로 방지할 수 있습니다.
- 차단된 리밸런싱은 운영자 노드가 볼트에서 유동적이어야 할 지분을 유지함을 의미합니다. 여러 대규모 인출이 발생하면 `vault liquid assets < pending withdrawal requests`일 때 프로토콜 지급 불능 상태에 빠질 수 있습니다.

**Proof of Concept:** `AvalancheMiddlewarTest.t.sol`에 테스트를 복사하십시오.

```solidity
    function test_DustLimitStakeCausesFakeRebalancing() public {
        address attacker = makeAddr("attacker");
        address delegatedStaker = makeAddr("delegatedStaker");

         uint48 epoch0 = _calcAndWarpOneEpoch();

        // Step 1. First, give Alice a large allocation and create nodes
        uint256 initialDeposit = 1000 ether;
        (uint256 depositAmount, uint256 initialShares) = _deposit(delegatedStaker, initialDeposit);
        console2.log("Initial deposit:", depositAmount);
        console2.log("Initial shares:", initialShares);

         // Set large L1 limit and give Alice all the shares initially
        _setL1Limit(bob, validatorManagerAddress, assetClassId, depositAmount, delegator);
        _setOperatorL1Shares(bob, validatorManagerAddress, assetClassId, alice, initialShares, delegator);

        // Step 2. Create nodes that will use this stake
        // move to next epoch
        uint48 epoch1 = _calcAndWarpOneEpoch();
        (bytes32[] memory nodeIds, bytes32[] memory validationIDs,) =
            _createAndConfirmNodes(alice, 2, 0, true);

        uint48 epoch2 = _calcAndWarpOneEpoch();

        // Verify nodes have the stake
        uint256 totalNodeStake = 0;
        for (uint i = 0; i < validationIDs.length; i++) {
            uint256 nodeStake = middleware.getNodeStake(epoch2, validationIDs[i]);
            totalNodeStake += nodeStake;
            console2.log("Node", i, "stake:", nodeStake);
        }
        console2.log("Total stake in nodes:", totalNodeStake);

        uint256 operatorTotalStake = middleware.getOperatorStake(alice, epoch2, assetClassId);
        uint256 operatorUsedStake = middleware.getOperatorUsedStakeCached(alice);
        console2.log("Operator total stake (from delegation):", operatorTotalStake);
        console2.log("Operator used stake (in nodes):", operatorUsedStake);

        // Step 3. Delegated staker withdraws, reducing Alice's available stake
        console2.log("\n--- Delegated staker withdrawing 60% ---");
        uint256 withdrawAmount = (initialDeposit * 60) / 100; // 600 ether

        vm.startPrank(delegatedStaker);
        (uint256 burnedShares, uint256 withdrawalShares) = vault.withdraw(delegatedStaker, withdrawAmount);
        vm.stopPrank();

        console2.log("Withdrawn amount:", withdrawAmount);
        console2.log("Burned shares:", burnedShares);
        console2.log("Remaining shares for Alice:", initialShares - burnedShares);

         // Step 4. Reduce Alice's operator shares to reflect the withdrawal
        uint256 newOperatorShares = initialShares - burnedShares;
        _setOperatorL1Shares(bob, validatorManagerAddress, assetClassId, alice, newOperatorShares, delegator);


        console2.log("Updated Alice's operator shares to:", newOperatorShares);

        // Step 5. Move to next epoch - this creates the imbalance
        uint48 epoch3  = _calcAndWarpOneEpoch();

        uint256 newOperatorTotalStake = middleware.getOperatorStake(alice, epoch3, assetClassId);
        uint256 currentUsedStake = middleware.getOperatorUsedStakeCached(alice);

        console2.log("\n--- After withdrawal (imbalance created) ---");
        console2.log("Alice's new total stake (reduced):", newOperatorTotalStake);
        console2.log("Alice's used stake (still in nodes):", currentUsedStake);

        // Step 6. Attacker prevents legitimate rebalancing
        console2.log("\n--- ATTACKER PREVENTS REBALANCING ---");

        // Move to final window where forceUpdateNodes can be called
        _warpToLastHourOfCurrentEpoch();

        // Attacker front-runs with dust limitStake attack
        console2.log("Attacker executing dust forceUpdateNodes...");
        vm.prank(attacker);
        middleware.forceUpdateNodes(alice, 1); // 1 wei - minimal removal

        // Check if any meaningful stake was actually removed
        uint256 stakeAfterDustAttack = middleware.getOperatorUsedStakeCached(alice);
        console2.log("Used stake after dust attack:", stakeAfterDustAttack);

        uint256 actualRemoved = currentUsedStake > stakeAfterDustAttack ?
            currentUsedStake - stakeAfterDustAttack : 0;
        console2.log("Stake actually removed by dust attack:", actualRemoved);

       // The key issue: minimal stake removed, but still excess remains
        uint256 remainingExcess = stakeAfterDustAttack > newOperatorTotalStake ?
            stakeAfterDustAttack - newOperatorTotalStake : 0;
        console2.log("REMAINING EXCESS after dust attack:", remainingExcess);

        // 7. Try legitimate rebalancing - should be blocked
        console2.log("\n--- Attempting legitimate rebalancing ---");
        vm.expectRevert(); // Should revert with AvalancheL1Middleware__AlreadyRebalanced
        middleware.forceUpdateNodes(alice, 0); // Proper rebalancing with no limit
        console2.log(" Legitimate rebalancing blocked by AlreadyRebalanced");
    }
```

**Recommended Mitigation:**
- 결과적인 P-Chain 가중치가 동일할 때 업데이트를 방지하는 것을 고려하십시오.
- 남은 모든 지분이 나머지 활성 노드에 흡수되도록 `limitStake`에 최소값을 설정하는 것을 고려하십시오. 예를 들어:

```text
제거할 남은 지분: 100 ETH
축소할 수 있는 활성 노드: 10개 노드
최소 필요 limitStake: 100 ETH ÷ 10개 노드 = 노드당 10 ETH
```
이 최소값보다 작은 값은 운영자가 마땅히 그래야 하는 것보다 더 많은 지분을 유지할 수 있음을 의미합니다.

**Suzaku:**
커밋 [ee2bdd5](https://github.com/suzaku-network/suzaku-core/pull/155/commits/ee2bdd544a2705e9f10bd250ad40555f115b11cb)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Future epoch cache manipulation via `calcAndCacheStakes` allows reward manipulation

**Description:** `AvalancheL1Middleware::calcAndCacheStakes` 함수는 에포크 유효성 검사가 부족하여 공격자가 미래 에포크에 대한 지분 값을 캐시할 수 있습니다. 이를 통해 현재 지분 값을 잠그는(lock-in) 방식으로 보상 계산을 영구적으로 조작할 수 있으며, 해당 에포크가 도래했을 때는 이 값이 오래된 값이 될 수 있습니다.

`calcAndCacheStakes` 함수는 제공된 에포크가 미래가 아닌지 검증하지 않습니다:

```solidity
function calcAndCacheStakes(uint48 epoch, uint96 assetClassId) public returns (uint256 totalStake) {
    uint48 epochStartTs = getEpochStartTs(epoch); // 에포크 타이밍에 대한 유효성 검사 없음
    // ... 미래 에포크를 포함한 모든 에포크에 대한 값을 캐시하는 함수의 나머지 부분
}
```

`totalStakeCached` 플래그가 설정되면 해당 에포크 및 자산 클래스에 대한 `getOperatorStake` 호출은 아래와 같이 잘못된 `operatorStakeCache` 값을 반환합니다:

```solidity
function getOperatorStake(
    address operator,
    uint48 epoch,
    uint96 assetClassId
) public view returns (uint256 stake) {
    if (totalStakeCached[epoch][assetClassId]) {
        uint256 cachedStake = operatorStakeCache[epoch][assetClassId][operator];
        return cachedStake;
    }
    ...
}
```

미래 에포크로 호출될 때, 함수는 미래 타임스탬프에 대해 사용 가능한 최신 값을 반환하는 체크포인트 시스템(upperLookupRecent)을 사용하여 현재 지분 값을 쿼리합니다.

**Impact:** 여기에는 여러 문제가 있지만 두 가지 주요 문제는 다음과 같습니다:
- 공격자는 실제 지분이 감소하기 전에 높은 지분 값을 잠금으로써 보상 지분을 부풀릴 수 있습니다. 이후의 모든 입금/출금은 주어진 에포크에 대해 캐시된 지분이 업데이트되면 영향을 미치지 않습니다.
- `forceUpdateNodes` 메커니즘이 손상될 수 있습니다. 중요한 노드 리밸런싱 작업을 잘못 건너뛰어 시스템을 불일치 상태로 만들 수 있습니다.

**Proof of Concept:** 다음 테스트를 추가하고 실행하십시오:

```solidity
function test_operatorStakeOfTwoEpochsShouldBeEqual() public {
      uint256 operatorStake = middleware.getOperatorStake(alice, 1, assetClassId);
      console2.log("Operator stake (epoch", 1, "):", operatorStake);

      middleware.calcAndCacheStakes(5, assetClassId);
      uint256 newStake = middleware.getOperatorStake(alice, 2, assetClassId);
      console2.log("New epoch operator stake:", newStake);
      assertGe(newStake, operatorStake);

      uint256 depositAmount = 100_000_000_000_000_000_000;

      collateral.transfer(staker, depositAmount);

      vm.startPrank(staker);
      collateral.approve(address(vault), depositAmount);
      vault.deposit(staker, depositAmount);
      vm.stopPrank();

      vm.warp((5) * middleware.EPOCH_DURATION());

      middleware.calcAndCacheStakes(5, assetClassId);

      assertEq(
          middleware.getOperatorStake(alice, 4, assetClassId), middleware.getOperatorStake(alice, 5, assetClassId)
      );
  }
```

**Recommended Mitigation:** 미래 에포크 캐싱을 방지하기 위해 에포크 유효성 검사를 추가하는 것을 고려하십시오:
```solidity
function calcAndCacheStakes(uint48 epoch, uint96 assetClassId) public returns (uint256 totalStake) {
    uint48 currentEpoch = getCurrentEpoch();
    require(epoch <= currentEpoch, "Cannot cache future epochs"); //@audit 추가됨

    uint48 epochStartTs = getEpochStartTs(epoch);
    // ... 함수의 나머지 부분은 변경되지 않음
}
```
**Suzaku:**
커밋 [32b1a6c](https://github.com/suzaku-network/suzaku-core/pull/155/commits/32b1a6c55c1ab436c557114939afb3163cc9ec8f)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.

\clearpage
## High Risk


### Blacklisted implementation versions are accessible through migrations

**Description:** `VaultFactory` 계약은 취약하거나 사용되지 않는 계약 버전의 사용을 방지하기 위해 블랙리스트 메커니즘을 구현합니다. 그러나 블랙리스트 확인은 create 함수를 통해 새 볼트를 생성할 때만 적용되지만 migrate 함수에는 전혀 없습니다.

이를 통해 볼트 소유자는 보안 취약성이나 기타 중요한 문제로 인해 명시적으로 블랙리스트에 올린 버전으로 기존 볼트를 마이그레이션하여 블랙리스트 제한을 우회할 수 있습니다.

다음 시나리오를 고려하십시오:
- 버전 1로 볼트가 생성됨
- 버전 2가 팩토리에서 블랙리스트에 등록됨
- 블랙리스트에 등록되었음에도 불구하고 볼트는 버전 2로 성공적으로 마이그레이션할 수 있음
- 마이그레이션된 볼트는 블랙리스트에 등록된 구현의 기능을 완전히 상속함

블랙리스트가 제공하는 의도된 보안 장벽은 완전히 우회될 수 있어, 취약점이 식별된 후에도 잠재적인 공격 벡터를 열어줍니다.

**Impact:** 보안 취약성으로 인해 구현이 블랙리스트에 등록된 경우, 볼트 소유자는 블랙리스트에 등록된 버전으로 마이그레이션하여 여전히 해당 취약성에 자신을 노출시킬 수 있습니다.

**Proof of Concept:** 참고: 이 테스트를 실행하기 위해 제약 조건이 더 적은 `_migrate` 함수를 가진 `MockVaultTokenizedV3.sol`을 생성했습니다(아래 참조).

```solidity
  // MockTokenizedVaultV3.sol
  function _migrate(uint64 oldVersion, uint64 newVersion, bytes calldata data) internal virtual onlyInitializing {
        // if (newVersion - oldVersion > 1) {
        //    revert();
        // }
        uint256 b_ = abi.decode(data, (uint256));
        b = b_;
    }

```

```solidity
    function testBlacklistDoesNotBlockMigration() public {
        // First, create a vault with version 1
        vault1 = vaultFactory.create(
            1, // version
            alice,
            abi.encode(
                IVaultTokenized.InitParams({
                    collateral: address(collateral),
                    burner: address(0xdEaD),
                    epochDuration: 7 days,
                    depositWhitelist: false,
                    isDepositLimit: false,
                    depositLimit: 0,
                    defaultAdminRoleHolder: alice,
                    depositWhitelistSetRoleHolder: alice,
                    depositorWhitelistRoleHolder: alice,
                    isDepositLimitSetRoleHolder: alice,
                    depositLimitSetRoleHolder: alice,
                    name: "Test",
                    symbol: "TEST"
                })
            ),
            address(delegatorFactory),
            address(slasherFactory)

        );

        // Verify initial version
        assertEq(IVaultTokenized(vault1).version(), 1);

        // Blacklist version 2
        vaultFactory.blacklist(2);

        // Despite version 2 being blacklisted, we can still migrate to it!
        vm.prank(alice);
        // This should revert if blacklist was properly enforced, but it won't
        vaultFactory.migrate(vault1, 2, abi.encode(20));

        // Verify the vault is now at version 2, despite it being blacklisted
        assertEq(IVaultTokenized(vault1).version(), 2);

        // set the value of b inside MockVaultTokenizedV3 as 20
        assertEq(
            MockVaultTokenizedV3(vault1).version2State(),
            20);
    }
```

**Recommended Mitigation:** `create` 함수와의 일관성을 보장하기 위해 migrate 함수에 블랙리스트 확인을 추가하는 것을 고려하십시오.

```diff solidity
function migrate(address entity_, uint64 newVersion, bytes calldata data) external checkEntity(entity_) {
    if (msg.sender != Ownable(entity_).owner()) {
        revert MigratableFactory__NotOwner();
    }

    if (newVersion <= IVaultTokenized(entity_).version()) {
        revert MigratableFactory__OldVersion();
    }

++    // Add this missing check
++    if (blacklisted[newVersion]) {
++       revert MigratableFactory__VersionBlacklisted();
++    }

    IMigratableEntityProxy(entity_).upgradeToAndCall(
        implementation(newVersion),
        abi.encodeCall(IVaultTokenized.migrate, (newVersion, data))
    );

    emit Migrate(entity_, newVersion);
}
```

**Suzaku:**
[d3f98b8](https://github.com/suzaku-network/suzaku-core/pull/155/commits/d3f98b82653ec3fa1f6f4b26049e9508cd9cda07)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### In `DelegatorFactory` new entity can be created for a blacklisted implementation

**Description:** `DelegatorFactory::create`의 현재 구현은 지정된 구현 유형이 명시적으로 블랙리스트에 등록되었는지 확인하지 못합니다. 이는 블랙리스트에 등록된 계약 유형이 여전히 배포될 수 있도록 하여 보안 및 일관성 위험을 초래하며, 잠재적으로 거버넌스 또는 보안 제한을 우회할 수 있습니다.

**Impact:** create 함수에 블랙리스트 확인이 없으면 안전하지 않거나, 사용되지 않거나, 달리 제한된 것으로 간주되었을 수 있는 구현 유형을 기반으로 계약을 생성할 수 있습니다.

**Proof of Concept:** 새 테스트 클래스 `DelegatorFactoryTest.t.sol`을 생성하십시오:

```solidity
// SPDX-License-Identifier: MIT
// SPDX-FileCopyrightText: Copyright 2024 ADDPHO

pragma solidity 0.8.25;

import {Test, console2} from "forge-std/Test.sol";
import {DelegatorFactory} from "../src/contracts/DelegatorFactory.sol";
import {IDelegatorFactory} from "../src/interfaces/IDelegatorFactory.sol";
import {Strings} from "@openzeppelin/contracts/utils/Strings.sol";
import {IEntity} from "../../src/interfaces/common/IEntity.sol";
import {ERC165} from "@openzeppelin/contracts/utils/introspection/ERC165.sol";
import "@openzeppelin/contracts/utils/introspection/IERC165.sol";

contract DelegatorFactoryTest is Test {
    address owner;
    address operator1;
    address operator2;

    DelegatorFactory factory;
    MockEntity mockImpl;

    function setUp() public {
        owner = address(this);
        operator1 = makeAddr("operator1");
        operator2 = makeAddr("operator2");

        factory = new DelegatorFactory(owner);

        // Deploy a mock implementation that conforms to IEntity
        mockImpl = new MockEntity(address(factory), 0);

        // Whitelist the implementation
        factory.whitelist(address(mockImpl));
    }

    function testCreateBeforeBlacklist() public {
        bytes memory initData = abi.encode("test");

        address created = factory.create(0, initData);

        assertTrue(factory.isEntity(created), "Entity should be created and registered");
    }

    function testCreateFailsAfterBlacklist() public {
        bytes memory initData = abi.encode("test");

        factory.blacklist(0);

        factory.create(0, initData); //@note no revert although blacklisted
    }
}

contract MockEntity is IEntity, ERC165 {
    address public immutable FACTORY;
    uint64 public immutable TYPE;

    string public data;

    constructor(address factory_, uint64 type_) {
        FACTORY = factory_;
        TYPE = type_;
    }

    function initialize(
        bytes calldata initData
    ) external {
        data = abi.decode(initData, (string));
    }

    function supportsInterface(
        bytes4 interfaceId
    ) public view virtual override(ERC165, IERC165) returns (bool) {
        return interfaceId == type(IEntity).interfaceId || super.supportsInterface(interfaceId);
    }
}

```

**Recommended Mitigation:** `DelagatorFactory::create`에 다음 확인을 추가하는 것을 고려하십시오.

```diff solidity
function create(uint64 type_, bytes calldata data) external returns (address entity_) {
++    if (blacklisted[type_]) {
++       revert DelagatorFactory__TypeBlacklisted();
++    }
        entity_ = implementation(type_).cloneDeterministic(keccak256(abi.encode(totalEntities(), type_, data)));

        _addDelegatorEntity(entity_);

        IEntity(entity_).initialize(data);
    }
```

**Suzaku:**
커밋 [292d5b7](https://github.com/suzaku-network/suzaku-core/pull/155/commits/292d5b71ac3a377351d66f239405c4d38af53830)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Incorrect summation of curator shares in `claimUndistributedRewards` leads to deficit in claimed undistributed rewards

**Description:** `claimUndistributedRewards` 함수는 `REWARDS_DISTRIBUTOR_ROLE`이 스테이커, 운영자 또는 큐레이터가 청구하지 않은 특정 `epoch`에 대한 보상을 수집할 수 있도록 설계되었습니다. 이를 위해 다양한 참가자에게 할당된 모든 지분 비율(베이시스 포인트)의 합계를 나타내는 `totalDistributedShares`를 계산합니다.

```solidity
// Calculate total distributed shares for the epoch
uint256 totalDistributedShares = 0;

// Sum operator shares
address[] memory operators = l1Middleware.getAllOperators();
for (uint256 i = 0; i < operators.length; i++) {
    totalDistributedShares += operatorShares[epoch][operators[i]];
}

// Sum vault shares
address[] memory vaults = middlewareVaultManager.getVaults(epoch);
for (uint256 i = 0; i < vaults.length; i++) {
    totalDistributedShares += vaultShares[epoch][vaults[i]];
}

// Sum curator shares
for (uint256 i = 0; i < vaults.length; i++) {
    address curator = VaultTokenized(vaults[i]).owner();
    totalDistributedShares += curatorShares[epoch][curator];
}
```

코드는 에포크에서 활성 상태인 모든 `vaults`를 반복합니다. 각 `vault`에 대해 `curator`(소유자)를 검색하고 `curatorShares[epoch][curator]`를 `totalDistributedShares`에 추가합니다. 그러나 `curatorShares[epoch][curator]` 매핑은 이미 해당 `epoch`에 대한 특정 `curator`의 *총 누적 지분*을 저장하며, 이는 그들이 소유한 모든 볼트와 해당 볼트가 위임한 모든 운영자로부터 집계된 것입니다.

```solidity
// First pass: calculate raw shares and total
for (uint256 i = 0; i < vaults.length; i++) {
    address vault = vaults[i];
    uint96 vaultAssetClass = middlewareVaultManager.getVaultAssetClass(vault);

    uint256 vaultStake = BaseDelegator(IVaultTokenized(vault).delegator()).stakeAt(
        l1Middleware.L1_VALIDATOR_MANAGER(), vaultAssetClass, operator, epochTs, new bytes(0)
    );

    if (vaultStake > 0) {
        uint256 operatorActiveStake =
            l1Middleware.getOperatorUsedStakeCachedPerEpoch(epoch, operator, vaultAssetClass);

        uint256 vaultShare = Math.mulDiv(vaultStake, BASIS_POINTS_DENOMINATOR, operatorActiveStake);
        vaultShare =
            Math.mulDiv(vaultShare, rewardsSharePerAssetClass[vaultAssetClass], BASIS_POINTS_DENOMINATOR);
        vaultShare = Math.mulDiv(vaultShare, operatorShare, BASIS_POINTS_DENOMINATOR);

        uint256 operatorTotalStake = l1Middleware.getOperatorStake(operator, epoch, vaultAssetClass);

        if (operatorTotalStake > 0) {
            uint256 operatorStakeRatio =
                Math.mulDiv(operatorActiveStake, BASIS_POINTS_DENOMINATOR, operatorTotalStake);
            vaultShare = Math.mulDiv(vaultShare, operatorStakeRatio, BASIS_POINTS_DENOMINATOR);
        }

        // Calculate curator share
        uint256 curatorShare = Math.mulDiv(vaultShare, curatorFee, BASIS_POINTS_DENOMINATOR);
        curatorShares[epoch][VaultTokenized(vault).owner()] += curatorShare;

        // Store vault share after removing curator share
        vaultShares[epoch][vault] += vaultShare - curatorShare;
    }
}
```

단일 큐레이터가 에포크에서 활성화된 여러 볼트를 소유한 경우, 그들의 *총* 지분(`curatorShares[epoch][curator]`에서)이 `totalDistributedShares`에 여러 번(소유한 각 볼트마다 한 번씩) 추가됩니다. 이는 `totalDistributedShares` 값을 인위적으로 부풀립니다.

**Impact:** 부풀려진 `totalDistributedShares`는 실제 `undistributedRewards`의 과소평가로 이어집니다. 공식 `undistributedRewards = totalRewardsForEpoch - Math.mulDiv(totalRewardsForEpoch, totalDistributedShares, BASIS_POINTS_DENOMINATOR)`는 실제로 배포되지 않은 금액보다 더 적은 금액을 산출합니다.

결과적으로:
1.  `REWARDS_DISTRIBUTOR_ROLE`은 권한이 있는 것보다 적은 양의 미분배 토큰을 청구하게 됩니다.
2.  실제 미분배 금액과 잘못 계산된(더 작은) 금액 간의 차이는 계약에 잠긴 상태로 유지됩니다(업그레이드되지 않는 한).

**Proof of Concept:**
```solidity
// ─────────────────────────────────────────────────────────────────────────────
// PoC – Incorrect Sum-of-Shares
// Shows that the sum of operator + vault + curator shares can exceed 10 000 bp
// (100 %), proving that `claimUndistributedRewards` will mis-count.
// ─────────────────────────────────────────────────────────────────────────────
import {AvalancheL1MiddlewareTest} from "./AvalancheL1MiddlewareTest.t.sol";

import {Rewards}           from "src/contracts/rewards/Rewards.sol";
import {MockUptimeTracker} from "../mocks/MockUptimeTracker.sol";
import {ERC20Mock}         from "@openzeppelin/contracts/mocks/token/ERC20Mock.sol";

import {VaultTokenized}    from "src/contracts/vault/VaultTokenized.sol";

import {console2}          from "forge-std/console2.sol";
import {stdError}          from "forge-std/Test.sol";

contract PoCIncorrectSumOfShares is AvalancheL1MiddlewareTest {
    // ── helpers & globals ────────────────────────────────────────────────────
    MockUptimeTracker internal uptimeTracker;
    Rewards          internal rewards;
    ERC20Mock        internal rewardsToken;

    address internal REWARDS_MANAGER_ROLE     = makeAddr("REWARDS_MANAGER_ROLE");
    address internal REWARDS_DISTRIBUTOR_ROLE = makeAddr("REWARDS_DISTRIBUTOR_ROLE");

    uint96 secondaryAssetClassId = 2;          // activate a 2nd asset-class (40 % of rewards)

    // -----------------------------------------------------------------------
    //                      MAIN TEST ROUTINE
    // -----------------------------------------------------------------------
    function test_PoCIncorrectSumOfShares() public {
        _setupRewardsAndSecondaryAssetClass();          // 1. deploy + fund rewards

        address[] memory operators = middleware.getAllOperators();

        // 2. Alice creates *two* nodes (same stake reused) -------------------
        console2.log("Creating nodes for Alice");
        _createAndConfirmNodes(alice, 2, 100_000_000_001_000, true);

        // 3. Charlie is honest – single big node -----------------------------
        console2.log("Creating node for Charlie");
        _createAndConfirmNodes(charlie, 1, 150_000_000_000_000, true);

        // 4. Roll over so stakes are cached at epoch T ------------------------
        uint48 epoch = _calcAndWarpOneEpoch();
        console2.log("Moved to one epoch ");

        // Cache total stakes for primary & secondary classes
        middleware.calcAndCacheStakes(epoch, assetClassId);
        middleware.calcAndCacheStakes(epoch, secondaryAssetClassId);

        // 5. Give everyone perfect uptime so shares are fully counted --------
        for (uint i = 0; i < operators.length; i++) {
            uptimeTracker.setOperatorUptimePerEpoch(epoch,   operators[i], 4 hours);
            uptimeTracker.setOperatorUptimePerEpoch(epoch+1, operators[i], 4 hours);
        }

        // 6. Warp forward 3 epochs (rewards are distributable @ T-2) ---------
        _calcAndWarpOneEpoch(3);
        console2.log("Warped forward for rewards distribution");

        // 7. Distribute rewards ---------------------------------------------
        vm.prank(REWARDS_DISTRIBUTOR_ROLE);
        rewards.distributeRewards(epoch, uint48(operators.length));
        console2.log("Rewards distributed");

        // 8. Sum all shares and show bug (>10 000 bp) ------------------------
        uint256 totalShares = 0;

        // operator shares
        for (uint i = 0; i < operators.length; i++) {
            uint256 s = rewards.operatorShares(epoch, operators[i]);
            totalShares += s;
        }
        // vault shares
        address[] memory vaults = vaultManager.getVaults(epoch);
        for (uint i = 0; i < vaults.length; i++) {
            uint256 s = rewards.vaultShares(epoch, vaults[i]);
            totalShares += s;
        }
        // curator shares (may be double-counted!)
        for (uint i = 0; i < vaults.length; i++) {
            address curator = VaultTokenized(vaults[i]).owner();
            uint256 s = rewards.curatorShares(epoch, curator);
            totalShares += s;
        }
        console2.log("Total shares is greater than 10000 bp");

        assertGt(totalShares, 10_000);
    }

    // -----------------------------------------------------------------------
    //                      SET-UP HELPER
    // -----------------------------------------------------------------------
    function _setupRewardsAndSecondaryAssetClass() internal {
        uptimeTracker = new MockUptimeTracker();
        rewards       = new Rewards();

        // Initialise Rewards contract ---------------------------------------
        rewards.initialize(
            owner,                                  // admin
            owner,                                  // protocol fee recipient
            payable(address(middleware)),           // middleware
            address(uptimeTracker),                 // uptime oracle
            1000,                                   // protocol fee 10%
            2000,                                   // operator fee 20%
            1000,                                   // curator  fee 10%
            11_520                                  // min uptime (s)
        );

        // Assign roles -------------------------------------------------------
        vm.prank(owner);
        rewards.setRewardsManagerRole(REWARDS_MANAGER_ROLE);
        vm.prank(REWARDS_MANAGER_ROLE);
        rewards.setRewardsDistributorRole(REWARDS_DISTRIBUTOR_ROLE);

        // Mint & approve mock reward token ----------------------------------
        rewardsToken = new ERC20Mock();
        rewardsToken.mint(REWARDS_DISTRIBUTOR_ROLE, 1_000_000 ether);
        vm.prank(REWARDS_DISTRIBUTOR_ROLE);
        rewardsToken.approve(address(rewards), 1_000_000 ether);

        // Fund 10 epochs of rewards -----------------------------------------
        vm.startPrank(REWARDS_DISTRIBUTOR_ROLE);
        rewards.setRewardsAmountForEpochs(1, 10, address(rewardsToken), 100_000 * 1e18);
        vm.stopPrank();
        console2.log("Reward pool funded");

        // Configure 60 % primary / 40 % secondary split ---------------------
        vm.startPrank(REWARDS_MANAGER_ROLE);
        rewards.setRewardsShareForAssetClass(1,                     6000); // 60 %
        rewards.setRewardsShareForAssetClass(secondaryAssetClassId, 4000); // 40 %
        vm.stopPrank();
        console2.log("Reward share split set: 60/40");

        // Create a secondary asset-class + vault so split is in effect
        _setupAssetClassAndRegisterVault(
            secondaryAssetClassId, 0,
            collateral2, vault3,
            type(uint256).max, type(uint256).max, delegator3
        );
        console2.log("Secondary asset-class & vault registered\n");
    }
}
```
**Output:**
```bash
Ran 1 test for test/middleware/PoCIncorrectSumOfShares.t.sol:PoCIncorrectSumOfShares
[PASS] test_PoCIncorrectSumOfShares() (gas: 8309923)
Logs:
  Reward pool funded
  Reward share split set: 60/40
  Secondary asset-class & vault registered

  Creating nodes for Alice
  Creating node for Charlie
  Moved to one epoch
  Warped forward for rewards distribution
  Rewards distributed
  Total shares is greater than 10000 bp

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 6.20ms (2.58ms CPU time)

Ran 1 test suite in 129.02ms (6.20ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

**Recommended Mitigation:** 큐레이터 지분을 올바르게 합산하려면 각 큐레이터의 해당 에포크에 대한 총 지분이 한 번만 계산되도록 하십시오. 이는 다음을 통해 달성할 수 있습니다:

1.  **고유 큐레이터 추적:** 초기 지분 계산 중(예: `_calculateAndStoreVaultShares`에서), 해당 에포크에서 지분을 얻은 큐레이터의 고유 주소를 저장하는 데이터 구조(예: `EnumerableSet.AddressSet`)를 유지 관리합니다.
    ```solidity
     // Example: Add to state variables
     mapping(uint48 epoch => EnumerableSet.AddressSet) private _epochUniqueCurators;

     // In _calculateAndStoreVaultShares, after calculating curatorShare for a vault's owner:
     if (curatorShare > 0) {
         _epochUniqueCurators[epoch].add(VaultTokenized(vault).owner());
     }
    ```

2.  **`claimUndistributedRewards`에서 고유 큐레이터 반복:** 고유 큐레이터 세트를 반복하도록 큐레이터 지분 합계 루프를 수정합니다.
    ```solidity
     // In claimUndistributedRewards:
     ...
     // Sum curator shares
     EnumerableSet.AddressSet storage uniqueCurators = _epochUniqueCurators[epoch];
     for (uint256 i = 0; i < uniqueCurators.length(); i++) {
         address curator = uniqueCurators.at(i);
         totalDistributedShares += curatorShares[epoch][curator];
     }
     ...
    ```
이렇게 하면 에포크에서 보상을 받은 각 고유 큐레이터에 대해 `curatorShares[epoch][curatorAddress]`가 `totalDistributedShares`에 정확히 한 번 추가됩니다.

**Suzaku:**
커밋 [8f4adaa](https://github.com/suzaku-network/suzaku-core/pull/155/commits/8f4adaaca91402b1bde166f2d02c3ac9d72fc96c)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.




### Incorrect reward claim logic causes loss of access to intermediate epoch rewards

**Description:** `Rewards::distributeRewards`의 현재 구현에서 모든 지분은 현재 에포크와 분배되는 에포크 사이의 3-에포크 지연 후에 참가자에 대해 계산됩니다. 그러나 **청구 로직**에서 문제가 발생합니다.

과거 에포크에 대한 보상을 청구할 때, `lastEpochClaimedOperator`는 무조건 `currentEpoch - 1`로 업데이트됩니다. 이는 첫 번째 청구 시점에 아직 분배되지 않은 **중간 에포크**에 대한 청구를 차단할 수 있습니다.

**Problem Scenario**

다음 순서를 고려하십시오:

1. **Epoch 4**: **epoch 1**에 대한 보상이 분배됨
2. **Epoch 5**: Operator1이 보상 청구 → `lastEpochClaimedOperator = 4`
3. **Epoch 5**: **epoch 2**에 대한 보상이 이제 분배됨
4. **Epoch 5**: Operator1이 **epoch 2**에 대한 보상을 청구하려고 시도하지만, `lastEpochClaimedOperator > 2`이기 때문에 **차단됨**

결과적으로 Operator는 에포크 2에서 청구 가능한 보상에 대한 **접근 권한을 잃게 됩니다**.

**Problematic Code**

```solidity
if (totalRewards == 0) revert NoRewardsToClaim(msg.sender);
IERC20(rewardsToken).safeTransfer(recipient, totalRewards);
lastEpochClaimedOperator[msg.sender] = currentEpoch - 1; // <-- 잘못하여 중간 에포크를 건너뜀
```

**Impact:** **자금 손실** — 나중에 청구하여 `lastEpochClaimedOperator`를 이미 진행시킨 후 중간 에포크가 분배되는 경우, 사용자(운영자)는 정당한 보상을 청구하는 것이 영구적으로 방지됩니다.

**Proof of Concept:** 문제를 재현하려면 `RewardTest.t.sol`에 이 테스트 케이스를 추가하십시오:

```solidity
function test_distributeRewards_claimFee(uint256 uptime) public {
    uint48 epoch = 1;
    uptime = bound(uptime, 0, 4 hours);

    _setupStakes(epoch, uptime);
    _setupStakes(epoch + 2, uptime);

    address[] memory operators = middleware.getAllOperators();
    uint256 batchSize = 3;
    uint256 remainingOperators = operators.length;

    vm.warp((epoch + 3) * middleware.EPOCH_DURATION());
    while (remainingOperators > 0) {
        vm.prank(REWARDS_DISTRIBUTOR_ROLE);
        rewards.distributeRewards(epoch, uint48(batchSize));
        remainingOperators = remainingOperators > batchSize ? remainingOperators - batchSize : 0;
    }

    vm.warp((epoch + 4) * middleware.EPOCH_DURATION());

    for (uint256 i = 0; i < operators.length; i++) {
        uint256 operatorShare = rewards.operatorShares(epoch, operators[i]);
        if (operatorShare > 0) {
            vm.prank(operators[i]);
            rewards.claimOperatorFee(address(rewardsToken), operators[i]);
            assertGt(rewardsToken.balanceOf(operators[i]), 0, "Operator should receive rewards ");
            vm.stopPrank();
            break;
        }
    }

    vm.warp((epoch + 5) * middleware.EPOCH_DURATION());
    remainingOperators = operators.length;
    while (remainingOperators > 0) {
```solidity
        vm.prank(REWARDS_DISTRIBUTOR_ROLE);
        rewards.distributeRewards(epoch + 2, uint48(batchSize));
        remainingOperators = remainingOperators > batchSize ? remainingOperators - batchSize : 0;
    }

    vm.warp((epoch + 9) * middleware.EPOCH_DURATION());
    for (uint256 i = 0; i < operators.length; i++) {
        uint256 operatorShare = rewards.operatorShares(epoch + 2, operators[i]);
        if (operatorShare > 0) {
            vm.prank(operators[i]);
            rewards.claimOperatorFee(address(rewardsToken), operators[i]);
            vm.stopPrank();
            break;
        }
    }
}
```

**Recommended Mitigation:** `lastEpochClaimedOperator`가 항상 `currentEpoch - 1`로 할당되는 대신, **사용자가 보상을 성공적으로 청구한 최대 에포크**로만 **업데이트**되도록 `claimOperatorFee` 로직을 업데이트하십시오.

**Suzaku:**
커밋 [6a0cbb1](https://github.com/suzaku-network/suzaku-core/pull/155/commits/6a0cbb1faa796e8925decad1ce9860eb20f184e7)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.




### Timestamp boundary condition causes reward dilution for active operators

**Description:** `AvalancheL1Middleware::_wasActiveAt()` 함수는 운영자 및 볼트가 비활성화된 정확한 타임스탬프에 "활성" 상태로 잘못 처리하는 경계 조건 버그를 포함하고 있습니다. 이로 인해 비활성화된 운영자의 지분이 보상 분배를 위한 총 지분 계산에 포함되어 활성 운영자의 보상이 크게 희석됩니다.

`_wasActiveAt()` 함수는 비활성화 시간 비교에 `>` 대신 `>=`를 사용합니다:

```solidity
function _wasActiveAt(uint48 enabledTime, uint48 disabledTime, uint48 timestamp) private pure returns (bool) {
    return enabledTime != 0 && enabledTime <= timestamp && (disabledTime == 0 || disabledTime >= timestamp); //@audit disabledTime >= timestamp means an operator is active at a timestamp when he was disabled
 }
```

`disabledTime == timestamp`일 때, 운영자는 그 순간에 비활성화되었음에도 불구하고 부정확하게 활성 상태로 간주됩니다.


이 버그는 보상 계산에 사용되는 `calcAndCacheStakes()` 함수에 영향을 미칩니다. 운영자가 에포크 시작 시점에 정확히 비활성화된 경우, `totalStake`는 비활성화된 운영자의 지분을 포함하므로 부풀려진 값을 반환합니다.

```solidity
function calcAndCacheStakes(uint48 epoch, uint96 assetClassId) public returns (uint256 totalStake) {
    uint48 epochStartTs = getEpochStartTs(epoch);
    uint256 length = operators.length();

    for (uint256 i; i < length; ++i) {
        (address operator, uint48 enabledTime, uint48 disabledTime) = operators.atWithTimes(i);
        if (!_wasActiveAt(enabledTime, disabledTime, epochStartTs)) { // @audit: this gets skipped
            continue;
        }
        uint256 operatorStake = getOperatorStake(operator, epoch, assetClassId);
        operatorStakeCache[epoch][assetClassId][operator] = operatorStake;
        totalStake += operatorStake; // @audit -> this is inflated
    }
    totalStakeCache[epoch][assetClassId] = totalStake;
    totalStakeCached[epoch][assetClassId] = true;
}
```

동일한 `_wasActiveAt()` 함수가 볼트를 반복할 때 `getOperatorStake()`에서 사용됩니다. 볼트가 에포크 시작 시점에 정확히 비활성화된 경우, 운영자의 지분 계산에 잘못 포함됩니다:

```solidity
function getOperatorStake(address operator, uint48 epoch, uint96 assetClassId) public view returns (uint256 stake) {
    uint48 epochStartTs = getEpochStartTs(epoch);
    uint256 totalVaults = vaultManager.getVaultCount();

    for (uint256 i; i < totalVaults; ++i) {
        (address vault, uint48 enabledTime, uint48 disabledTime) = vaultManager.getVaultAtWithTimes(i);

        // Skip if vault not active in the target epoch
        if (!_wasActiveAt(enabledTime, disabledTime, epochStartTs)) { // @audit: same boundary bug for vaults
            continue;
        }

        // Skip if vault asset not in AssetClassID
        if (vaultManager.getVaultAssetClass(vault) != assetClassId) {
            continue;
        }

        uint256 vaultStake = BaseDelegator(IVaultTokenized(vault).delegator()).stakeAt(
            L1_VALIDATOR_MANAGER, assetClassId, operator, epochStartTs, new bytes(0)
        );

        stake += vaultStake; // @audit -> inflated when disabled vaults are included
    }
}
```

`Rewards::_calculateOperatorShare`에서, 이 부풀려진 `totalStake`는 해당 에포크의 운영자 보상 지분을 계산하는 데 사용됩니다.

```solidity
// In Rewards.sol - _calculateOperatorShare()
function _calculateOperatorShare(uint48 epoch, address operator) internal {
    // ...
    for (uint256 i = 0; i < assetClasses.length; i++) {
        uint256 operatorStake = l1Middleware.getOperatorUsedStakeCachedPerEpoch(epoch, operator, assetClasses[i]);
        uint256 totalStake = l1Middleware.totalStakeCache(epoch, assetClasses[i]); // @audit inflated value

        uint256 shareForClass = Math.mulDiv(
            Math.mulDiv(operatorStake, BASIS_POINTS_DENOMINATOR, totalStake), // @audit shares are diluted due to inflated total stake
            assetClassShare,
            BASIS_POINTS_DENOMINATOR
        );
        totalShare += shareForClass;
    }
    // ...
}
```

**Impact:** 활성 운영자에게 분배되어야 할 보상이 Rewards 계약에 갇히게 됩니다. 이는 활성 운영자 및 볼트에 대한 보상의 상당한 희석으로 이어집니다.


**Proof of Concept:**
```solidity
   function test_wasActiveAtBoundaryBug() public {
        // create nodes for alice and charlie
        // Add nodes for Alice
        (bytes32[] memory aliceNodeIds,,) = _createAndConfirmNodes(alice, 1, 0, true);
        console2.log("Created", aliceNodeIds.length, "nodes for Alice");

        // Add nodes for Charlie
        (bytes32[] memory charlieNodeIds,,) = _createAndConfirmNodes(charlie, 1, 0, true);
        console2.log("Created", charlieNodeIds.length, "nodes for Charlie");

        // move to current epoch so that nodes are active
        // record next epoch start time stamp and epoch number
        uint48 currentEpoch = _calcAndWarpOneEpoch();
        uint48 nextEpoch = currentEpoch + 1;
        uint48 nextEpochStartTs = middleware.getEpochStartTs(nextEpoch);


        // Setup rewards contract (simplified version)
        address admin = makeAddr("admin");
        address protocolOwner = makeAddr("protocolOwner");
        address rewardsDistributor = makeAddr("rewardsDistributor");

        // Deploy rewards contract
        Rewards rewards = new Rewards();
        MockUptimeTracker uptimeTracker = new MockUptimeTracker();

        // Initialize rewards contract
        rewards.initialize(
            admin,
            protocolOwner,
            payable(address(middleware)),
            address(uptimeTracker),
            1000, // 10% protocol fee
            2000, // 20% operator fee
            1000, // 10% curator fee
            11520 // min required uptime
        );

        // Setup roles
        vm.prank(admin);
        rewards.setRewardsDistributorRole(rewardsDistributor);

        // Create rewards token and set rewards
        ERC20Mock rewardsToken = new ERC20Mock();
        uint256 totalRewards = 100 ether;
        rewardsToken.mint(rewardsDistributor, totalRewards);

         vm.startPrank(rewardsDistributor);
        rewardsToken.approve(address(rewards), totalRewards);
        rewards.setRewardsAmountForEpochs(nextEpoch, 1, address(rewardsToken), totalRewards);
        vm.stopPrank();

        // Set rewards share for primary asset class
        vm.prank(admin);
        rewards.setRewardsShareForAssetClass(1, 10000); // 100% for primary asset class

        // Record initial stake
        uint256 aliceInitialStake = middleware.getOperatorStake(alice, currentEpoch, assetClassId);
        uint256 charlieInitialStake = middleware.getOperatorStake(charlie, currentEpoch, assetClassId);

        // Verify they have nodes
        bytes32[] memory aliceCurrentNodes = middleware.getActiveNodesForEpoch(alice, currentEpoch);
        bytes32[] memory charlieCurrentNodes = middleware.getActiveNodesForEpoch(charlie, currentEpoch);
        console2.log("Alice current nodes:", aliceCurrentNodes.length);
        console2.log("Charlie current nodes:", charlieCurrentNodes.length);

        console2.log("=== INITIAL STATE ===");
        console2.log("Alice initial stake:", aliceInitialStake);
        console2.log("Charlie initial stake:", charlieInitialStake);
        console2.log("Total initial:", aliceInitialStake + charlieInitialStake);


        // Move to exact epoch boundary and disable Alice
        vm.warp(nextEpochStartTs);
        vm.prank(validatorManagerAddress);
        middleware.disableOperator(alice);

        // Set uptime alice - 0, charlie - full
        uptimeTracker.setOperatorUptimePerEpoch(nextEpoch, alice, 0 hours);
        uptimeTracker.setOperatorUptimePerEpoch(nextEpoch, charlie, 4 hours);

        // Calculate stakes for the boundary epoch
        uint256 aliceBoundaryStake = middleware.getOperatorStake(alice, nextEpoch, assetClassId);
        uint256 charlieBoundaryStake = middleware.getOperatorStake(charlie, nextEpoch, assetClassId);
        uint256 totalBoundaryStake = middleware.calcAndCacheStakes(nextEpoch, assetClassId);

        console2.log("=== BOUNDARY EPOCH ===");
        console2.log("Epoch start timestamp:", nextEpochStartTs);
        console2.log("Alice disabled at timestamp:", nextEpochStartTs);
        console2.log("Alice boundary stake:", aliceBoundaryStake);
        console2.log("Charlie boundary stake:", charlieBoundaryStake);
        console2.log("Total boundary stake:", totalBoundaryStake);


       // Distribute rewards using actual Rewards contract
        vm.warp(nextEpochStartTs + 3 * middleware.EPOCH_DURATION()); // Move past distribution window

        vm.prank(rewardsDistributor);
        rewards.distributeRewards(nextEpoch, 10); // Process all operators

        // Move to claiming period
        vm.warp(nextEpochStartTs + 4 * middleware.EPOCH_DURATION());

        // Record balances before claiming
        uint256 rewardsContractBalance = rewardsToken.balanceOf(address(rewards));
        uint256 charlieBalanceBefore = rewardsToken.balanceOf(charlie);

        console2.log("=== UNDISTRIBUTED REWARDS TEST ===");
        console2.log("Total rewards in contract:", rewardsContractBalance);

        // Charlie claims his rewards
        vm.prank(charlie);
        rewards.claimOperatorFee(address(rewardsToken), charlie);

        uint256 charlieRewards = rewardsToken.balanceOf(charlie) - charlieBalanceBefore;
        console2.log("Charlie claimed:", charlieRewards);

        // Alice cannot claim (disabled/no uptime)
        vm.expectRevert();
        vm.prank(alice);
        rewards.claimOperatorFee(address(rewardsToken), alice);

        // Charlie should get 100% of operator rewards
        // Deduct protocol share - and calculate operator fees
        uint256 charliExpectedRewards = totalRewards * 9000 * 2000 / 100_000_000; // (total rewards - protocol share) * operator fee
        assertGt(charliExpectedRewards, charlieRewards);
    }
```

**Recommended Mitigation:** 정확한 타임스탬프에서 비활성화된 운영자를 제외하도록 `_wasActiveAt()`의 경계 조건을 변경하는 것을 고려하십시오.

**Suzaku:**
커밋 [9bbbcfc](https://github.com/suzaku-network/suzaku-core/pull/155/commits/9bbbcfce7bedd1dd4e60fdf55bb5f13ba8ab4847)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


###  Immediate stake cache updates enable reward distribution without P-Chain confirmation

**Description:** 미들웨어는 운영자가 `initializeValidatorStakeUpdate()`를 통해 지분 변경을 시작할 때, 이러한 변경이 P-Chain에 의해 확인되지 않은 상태임에도 불구하고 보상 계산을 위한 지분 캐시를 즉시 업데이트합니다.

이는 보상 계산이 실제 검증된 P-Chain 상태와 달라지는 시간적 창을 생성하여, 잠재적으로 운영자가 확인되지 않은 지분 증가를 기반으로 보상을 받을 수 있게 합니다.

운영자가 `initializeValidatorStakeUpdate()`를 호출하여 검증자의 지분을 수정하면, 미들웨어는 다음 에포크에 대한 지분 캐시를 즉시 업데이트합니다:

```solidity
// In initializeValidatorStakeUpdate():
function _initializeValidatorStakeUpdate(address operator, bytes32 validationID, uint256 newStake) internal {
    uint48 currentEpoch = getCurrentEpoch();

    nodeStakeCache[currentEpoch + 1][validationID] = newStake;
    nodePendingUpdate[validationID] = true;

    // @audit P-Chain operation initiated but NOT confirmed
    balancerValidatorManager.initializeValidatorWeightUpdate(validationID, scaledWeight);
}
```
그러나 보상 계산은 P-Chain 확인을 검증하지 않고 이 캐시된 지분을 즉시 사용합니다:

```solidity
function getOperatorUsedStakeCachedPerEpoch(uint48 epoch, address operator, uint96 assetClass) external view returns (uint256) {
    // Uses cached stake regardless of P-Chain confirmation status
    bytes32[] memory nodesArr = this.getActiveNodesForEpoch(operator, epoch);
    for (uint256 i = 0; i < nodesArr.length; i++) {
        bytes32 nodeId = nodesArr[i];
        bytes32 validationID = balancerValidatorManager.registeredValidators(abi.encodePacked(uint160(uint256(nodeId))));
        registeredStake += getEffectiveNodeStake(epoch, validationID); // @audit Uses unconfirmed stake
    }
```

미들웨어가 `forceUpdateNodes`에서 업데이트 대기 중인 검증자를 명시적으로 건너뛴다는 점은 주목할 가치가 있습니다:

```solidity
function forceUpdateNodes(address operator, uint256 limitStake) external {
    // ...
    for (uint256 i = length; i > 0 && leftoverStake > 0;) {
        bytes32 valID = balancerValidatorManager.registeredValidators(abi.encodePacked(uint160(uint256(nodeId))));
        if (balancerValidatorManager.isValidatorPendingWeightUpdate(valID)) {
            continue; // @audit No correction possible for pending validators
        }
        // ... stake adjustment logic
    }
}
```
이는 일관성 없는 접근 방식을 생성합니다. 노드 리밸런싱 중에는 로직이 P-chain 상태를 확인하는 반면, 보상 추정에는 동일한 확인이 누락되어 있습니다.


**Impact:**
- 운영자는 P-Chain 검증 전에 지분 증가를 제출하는 즉시 보상 부스트를 받습니다.
- P-chain은 작업을 거부하거나 확인하는 데 여러 에포크가 걸릴 수 있어 L1 Middleware 상태와 P-chain 상태의 불일치를 초래할 수 있습니다.

**Proof of Concept:** 현재 POC는 보상 계산이 확인되지 않은 지분 업데이트를 사용함을 보여줍니다. `AvalancheMiddlewareTest.t.sol`에 추가하십시오

```solidity
    function test_UnconfirmedStakeImmediateRewards() public {
        // Setup: Alice has 100 ETH equivalent stake
        uint48 epoch = _calcAndWarpOneEpoch();

        // increasuing vaults total stake
        (, uint256 additionalMinted) = _deposit(staker, 500 ether);

        // Now allocate more of this deposited stake to Alice (the operator)
        uint256 totalAliceShares = mintedShares + additionalMinted;
        _setL1Limit(bob, validatorManagerAddress, assetClassId, 3000 ether, delegator);
        _setOperatorL1Shares(bob, validatorManagerAddress, assetClassId, alice, totalAliceShares, delegator);

        // Move to next epoch to make the new stake available
        epoch = _calcAndWarpOneEpoch();

        // Verify Alice now has sufficient available stake
        uint256 aliceAvailableStake = middleware.getOperatorAvailableStake(alice);
        console2.log("Alice available stake: %s ETH", aliceAvailableStake / 1 ether);

        // Alice adds a node with 10 ETH stake
        (bytes32[] memory nodeIds, bytes32[] memory validationIDs,) =
            _createAndConfirmNodes(alice, 1, 10 ether, true);
        bytes32 nodeId = nodeIds[0];
        bytes32 validationID = validationIDs[0];

        // Move to next epoch and confirm initial state
        epoch = _calcAndWarpOneEpoch();
        uint256 initialStake = middleware.getNodeStake(epoch, validationID);
        assertEq(initialStake, 10 ether, "Initial stake should be 10 ETH");

        // Alice increases stake to 1000 ETH (10x increase)
        uint256 modifiedStake = 50 ether;
        vm.prank(alice);
        middleware.initializeValidatorStakeUpdate(nodeId, modifiedStake);

        // Check: Stake cache immediately updated for next epoch (unconfirmed!)
        uint48 nextEpoch = middleware.getCurrentEpoch() + 1;
        uint256 unconfirmedStake = middleware.nodeStakeCache(nextEpoch, validationID);
        assertEq(unconfirmedStake, modifiedStake, "Unconfirmed stake should be immediately set");

        // Verify: P-Chain operation is still pending
        assertTrue(
            mockValidatorManager.isValidatorPendingWeightUpdate(validationID),
            "P-Chain operation should still be pending"
        );

        // Move to next epoch (when unconfirmed stake takes effect)
        epoch = _calcAndWarpOneEpoch();

        // Reward calculations now use unconfirmed 1000 ETH stake
        uint256 operatorStakeForRewards = middleware.getOperatorUsedStakeCachedPerEpoch(
            epoch, alice, middleware.PRIMARY_ASSET_CLASS()
        );
        assertEq(
            operatorStakeForRewards,
            modifiedStake,
            "Reward calculations should use unconfirmed 500 ETH stake"
        );
        console2.log("Stake used for rewards: %s ETH", operatorStakeForRewards / 1 ether);
    }
```

**Recommended Mitigation:** 초기화 중이 아니라 P-Chain 확인 후에만 지분 캐시를 업데이트하는 것을 고려하십시오:

```solidity
function completeStakeUpdate(bytes32 nodeId, uint32 messageIndex) external {
    // ... existing logic ...

    // Update cache only after P-Chain confirms
    uint48 currentEpoch = getCurrentEpoch();
    nodeStakeCache[currentEpoch + 1][validationID] = validator.weight;
}
```
이 변경은 `_calcAndCacheNodeStakeForOperatorAtEpoch`의 변경도 필요로 합니다. 현재는 보류 중인 업데이트가 없을 때만 현재 에포크의 `nodeStakeCache`가 이전 에포크의 것으로 업데이트됩니다. 위의 변경 사항이 구현되면, 현재 에포크의 `nodeStakeCache`는 **항상** 이전 에포크에서 이월된 것이어야 합니다.

**Suzaku:**
커밋 [5157351](https://github.com/suzaku-network/suzaku-core/pull/155/commits/5157351d0e9a799679a74c33c6b69aa87d58ab51)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Vault rewards incorrectly scaled by cross-asset-class operator totals instead of asset class specific shares causing rewards leakage

**Description:** 현재 볼트 보상 분배 로직은 볼트 스테이커에게 보상이 체계적으로 과소 분배되도록 합니다. 문제는 자산 클래스별 운영자 보상 대신 교차 자산 클래스 운영자 수익자 지분 총계를 사용하여 개별 볼트 보상을 조정함으로써 발생합니다. 이는 운영자가 다른 자산 클래스 ID에 걸쳐 비대칭 지분을 보유한 시나리오에서 보상 누출을 유발합니다.

`Rewards::_calculateAndStoreVaultShares` 함수는 특정 자산 클래스에 속한 개별 볼트에 대한 보상을 조정하기 위해 `operatorBeneficiariesShares[operator]`(모든 자산 클래스에 걸친 운영자의 총 보상을 나타냄)를 잘못 사용합니다. 이는 볼트 보상이 운영자의 다른 관련 없는 자산 클래스 참여에 의해 축소되는 부적절한 희석 효과를 생성합니다.

먼저 `_calculateOperatorShare()`에서 운영자의 총 지분을 계산할 때:

```solidity
function _calculateOperatorShare(uint48 epoch, address operator) internal {
    // ... uptime calculations ...

    uint96[] memory assetClasses = l1Middleware.getAssetClassIds();
    for (uint256 i = 0; i < assetClasses.length; i++) {
        uint256 operatorStake = l1Middleware.getOperatorUsedStakeCachedPerEpoch(epoch, operator, assetClasses[i]);
        uint256 totalStake = l1Middleware.totalStakeCache(epoch, assetClasses[i]);
        uint16 assetClassShare = rewardsSharePerAssetClass[assetClasses[i]];

        uint256 shareForClass = Math.mulDiv(
            Math.mulDiv(operatorStake, BASIS_POINTS_DENOMINATOR, totalStake),
            assetClassShare, // @audit asset class share applied here
            BASIS_POINTS_DENOMINATOR
        );
        totalShare += shareForClass;
    }
    // ... rest of function
    operatorBeneficiariesShares[epoch][operator] = totalShare; //@audit this is storing cross-asset total
```


다시 `_calculateAndStoreVaultShares()`에서 교차 자산 운영자 지분인 `operatorBeneficiariesShares`가 특정 볼트의 볼트 지분을 계산하는 데 사용됩니다:

```solidity
function _calculateAndStoreVaultShares(uint48 epoch, address operator) internal {
    uint256 operatorShare = operatorBeneficiariesShares[epoch][operator]; // @audit already includes asset class weighting

    for (uint256 i = 0; i < vaults.length; i++) {
        address vault = vaults[i];
        uint96 vaultAssetClass = middlewareVaultManager.getVaultAssetClass(vault);

        // ... vault stake calculation ...

        uint256 vaultShare = Math.mulDiv(vaultStake, BASIS_POINTS_DENOMINATOR, operatorActiveStake);
        vaultShare = Math.mulDiv(vaultShare, rewardsSharePerAssetClass[vaultAssetClass], BASIS_POINTS_DENOMINATOR);
        vaultShare = Math.mulDiv(vaultShare, operatorShare, BASIS_POINTS_DENOMINATOR);
        //@audit Uses cross-asset operator total instead of asset-class-specific share
        // This scales vault rewards by operator's participation in OTHER asset classes
        // ... rest of function
    }
}
```

순 효과는 특정 운영자가 볼트 지분의 100%를 기여하더라도 글로벌 운영자 지분에 의해 축소되어 해당 특정 자산 클래스에 대한 보상 누출을 유발한다는 것입니다.

**Impact:** 볼트 지분이 받아야 할 것보다 적게 받는 모든 에포크에서 체계적인 보상 과소 지급이 발생하며, 이 초과분은 미분배 보상으로 다시 회수됩니다. 사실상 실제 보상 분배는 의도한 자산 클래스 할당과 일치하지 않습니다.

**Proof of Concept:** 테스트는 모든 수수료가 0인 경우에도 볼트 지분이 에포크 보상의 92.5%에 불과함을 보여줍니다. 7.5%는 미분배 보상의 일부가 되는 보상 누출에 기인합니다.

```text
  In the POC below,
- Asset Class 1: 50% share (5000 bp), 1000 total stake
- Asset Class 2: 20% share (2000 bp), 100 total stake
- Asset Class 3: 30% share (3000 bp), 100 total stake
- Operator B: 700/100/100 stake in classes 1/2/3 respectively
- Operator B's cross-asset total: (700/1000×5000) + (100/100×2000) + (100/100×3000) = 8500 bp
- Vault Share of Asset ID 2:
    - vaultShare = (100/100) × 2000 × 8500 / (10000 × 10000)
    - vaultShare = 1 × 2000 × 8500 / 100,000,000 = 1700 bp

Vault 2 should get Operator B's Asset Class 2 rewards only (2000 bp)
Instead, it gets scaled by operator's total across all classes (8500 bp)

This causes a dilution of 300 bp

```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.25;

import {Test} from "forge-std/Test.sol";
import {console2} from "forge-std/console2.sol";

import {MockAvalancheL1Middleware} from "../mocks/MockAvalancheL1Middleware.sol";
import {MockUptimeTracker} from "../mocks/MockUptimeTracker.sol";
import {MockVaultManager} from "../mocks/MockVaultManager.sol";
import {MockDelegator} from "../mocks/MockDelegator.sol";
import {MockVault} from "../mocks/MockVault.sol";
import {ERC20Mock} from "@openzeppelin/contracts/mocks/token/ERC20Mock.sol";

import {Rewards} from "../../src/contracts/rewards/Rewards.sol";
import {IRewards} from "../../src/interfaces/rewards/IRewards.sol";
import {BaseDelegator} from "../../src/contracts/delegator/BaseDelegator.sol";
import {IVaultTokenized} from "../../src/interfaces/vault/IVaultTokenized.sol";

contract RewardsAssetShareTest is Test {
    // Contracts
    MockAvalancheL1Middleware public middleware;
    MockUptimeTracker public uptimeTracker;
    MockVaultManager public vaultManager;
    Rewards public rewards;
    ERC20Mock public rewardsToken;

    // Test addresses
    address constant ADMIN = address(0x1);
    address constant PROTOCOL_OWNER = address(0x2);
    address constant REWARDS_MANAGER = address(0x3);
    address constant REWARDS_DISTRIBUTOR = address(0x4);
    address constant OPERATOR_A = address(0x1000);
    address constant OPERATOR_B = address(uint160(0x1000 + 1));

    function setUp() public {
        // Deploy mock contracts - simplified setup for our POC
        vaultManager = new MockVaultManager();

        //Set up 2 operators
        uint256[] memory nodesPerOperator = new uint256[](2);
        nodesPerOperator[0] = 1; // Operator 0x1000 has 1 node
        nodesPerOperator[1] = 1; // Operator A has 1 node

        middleware = new MockAvalancheL1Middleware(
            2,
            nodesPerOperator,
            address(0),
            address(vaultManager)
        );

        uptimeTracker = new MockUptimeTracker();

        // Deploy Rewards contract
        rewards = new Rewards();
        rewardsToken = new ERC20Mock();

        // Initialize with no fees to match our simplified example
        rewards.initialize(
            ADMIN,
            PROTOCOL_OWNER,
            payable(address(middleware)),
            address(uptimeTracker),
            0, // protocolFee = 0%
            0, // operatorFee = 0%
            0, // curatorFee = 0%
            0  // minRequiredUptime = 0
        );

        // Set up roles
        vm.prank(ADMIN);
        rewards.setRewardsManagerRole(REWARDS_MANAGER);

        vm.prank(REWARDS_MANAGER);
        rewards.setRewardsDistributorRole(REWARDS_DISTRIBUTOR);

        // Set up rewards token
        rewardsToken.mint(REWARDS_DISTRIBUTOR, 1_000_000 * 10**18);
        vm.prank(REWARDS_DISTRIBUTOR);
        rewardsToken.approve(address(rewards), 1_000_000 * 10**18);
    }

    function test_AssetShareFormula() public {
        uint48 epoch = 1;

        // Set Asset Class 1 to 50% rewards share (5000 basis points)
        vm.prank(REWARDS_MANAGER);
        rewards.setRewardsShareForAssetClass(1, 5000); // 50%

        vm.prank(REWARDS_MANAGER);
        rewards.setRewardsShareForAssetClass(2, 2000); // 20%

        vm.prank(REWARDS_MANAGER);
        rewards.setRewardsShareForAssetClass(3, 3000); // 30%

        // Set total stake in Asset Class 1 = 1000 tokens across network
        middleware.setTotalStakeCache(epoch, 1, 1000);
        middleware.setTotalStakeCache(epoch, 2, 100);  // Asset Class 2: 0 tokens
        middleware.setTotalStakeCache(epoch, 3, 100);  // Asset Class 3: 0 tokens

        // Set Operator A stake = 300 tokens (30% of network)
        middleware.setOperatorStake(epoch, OPERATOR_A, 1, 300);

        // Set operator A node stake (for primary asset class calculation)
        bytes32[] memory operatorNodes = middleware.getOperatorNodes(OPERATOR_A);
        middleware.setNodeStake(epoch, operatorNodes[0], 300);

        // No stake in other asset classes for Operator A
        middleware.setOperatorStake(epoch, OPERATOR_A, 2, 0);
        middleware.setOperatorStake(epoch, OPERATOR_A, 3, 0);

        bytes32[] memory operatorBNodes = middleware.getOperatorNodes(OPERATOR_B);
        middleware.setNodeStake(epoch, operatorBNodes[0], 700); // Remaining Asset Class 1 stake
        middleware.setOperatorStake(epoch, OPERATOR_B, 1, 700);
        middleware.setOperatorStake(epoch, OPERATOR_B, 2, 100);
        middleware.setOperatorStake(epoch, OPERATOR_B, 3, 100);

        // Set 100% uptime for Operator A & B
        uptimeTracker.setOperatorUptimePerEpoch(epoch, OPERATOR_A, 4 hours);
        uptimeTracker.setOperatorUptimePerEpoch(epoch, OPERATOR_B, 4 hours);


        // Create a vault for Asset Class 1 with 300 tokens staked (100% of operator's stake)
        address vault1Owner = address(0x500);
        (address vault1, address delegator1) = vaultManager.deployAndAddVault(
            address(0x123), // collateral
            vault1Owner
        );
        middleware.setAssetInAssetClass(1, vault1);
        vaultManager.setVaultAssetClass(vault1, 1);

          // Create vault for Asset Class 2
        address vault2Owner = address(0x600);
        (address vault2, address delegator2) = vaultManager.deployAndAddVault(address(0x123), vault2Owner);
        middleware.setAssetInAssetClass(2, vault2);
        vaultManager.setVaultAssetClass(vault2, 2);

        // Create vault for Asset Class 3
        address vault3Owner = address(0x700);
        (address vault3, address delegator3) = vaultManager.deployAndAddVault(
            address(0x125), // different collateral
            vault3Owner
        );
        middleware.setAssetInAssetClass(3, vault3);
        vaultManager.setVaultAssetClass(vault3, 3);


        // Set vault delegation: 300 tokens staked to Operator A
        uint256 epochTs = middleware.getEpochStartTs(epoch);
        MockDelegator(delegator1).setStake(
            middleware.L1_VALIDATOR_MANAGER(),
            1, // asset class
            OPERATOR_A,
            uint48(epochTs),
            300 // stake amount
        );

        MockDelegator(delegator1).setStake(middleware.L1_VALIDATOR_MANAGER(),
                                            1,
                                            OPERATOR_B,
                                            uint48(epochTs),
                                            700);

        MockDelegator(delegator2).setStake(
            middleware.L1_VALIDATOR_MANAGER(), 2, OPERATOR_B, uint48(epochTs), 100
        );

        MockDelegator(delegator3).setStake(
            middleware.L1_VALIDATOR_MANAGER(), 3, OPERATOR_B, uint48(epochTs), 100
        );

        // Set rewards for the epoch: 100,000 tokens
        vm.prank(REWARDS_DISTRIBUTOR);
        rewards.setRewardsAmountForEpochs(epoch, 1, address(rewardsToken), 100_000);


        // Wait 3 epochs as required by contract
        vm.warp((epoch + 3) * middleware.EPOCH_DURATION());

        // Distribute rewards
        vm.prank(REWARDS_DISTRIBUTOR);
        rewards.distributeRewards(epoch, 2);

        // Get calculated shares
        uint256 operatorABeneficiariesShare = rewards.operatorBeneficiariesShares(epoch, OPERATOR_A);
        uint256 operatorBBeneficiariesShare = rewards.operatorBeneficiariesShares(epoch, OPERATOR_B);

        uint256 vault1Share = rewards.vaultShares(epoch, vault1);
        uint256 vault2Share = rewards.vaultShares(epoch, vault2);
        uint256 vault3Share = rewards.vaultShares(epoch, vault3);

        console2.log("=== RESULTS ===");
        console2.log("operatorBeneficiariesShares[OPERATOR_A] =", operatorABeneficiariesShare, "basis points");
        console2.log("vaultShares[vault_1] =", vault1Share, "basis points");
        console2.log("operatorBeneficiariesShares[OPERATOR_B] =", operatorBBeneficiariesShare, "basis points");
        console2.log("vaultShares[vault_2] =", vault2Share, "basis points");
        console2.log("vaultShares[vault_3] =", vault3Share, "basis points");

        // Expected: 30% stake * 50% asset class = 15% = 1500 basis points
        assertEq(operatorABeneficiariesShare, 1500,
            "Operator share should be 1500 basis points (15%)");
        assertEq(vault1Share, 5000,
            "Vault share should be 5000 basis points (7.5% + 42.5)");

        assertEq(operatorBBeneficiariesShare, 8500,
            "Operator share should be 9500 basis points (85%)"); //  (700/1000) × 50%  +  (100/100) × 20%  + (100/100) × 30% = 85%
        assertEq(vault2Share, 1700,
            "Vault share should be 1700 basis points (19%)");
            // vaultShare = (100 / 100) × 10,000 = 10,000 bp
            // vaultShare = 10,000 × 2,000 / 10,000 = 2,000 bp (vaultShare * assetClassShare / 10000)
            // vaultShare = 2,000 × 8,500 / 10,000 = 1,700 bp (vaultShare * operatorShare / 10000)

         assertEq(vault3Share, 2550,
            "Vault share should be 2550 basis points (28.5%)");
            // vaultShare = (100 / 100) × 10,000 = 10,000 bp
            // vaultShare = 10,000 × 3,000/10,000 = 3,000 bp (vaultShare * assetClassShare / 10000)
            // vaultShare = 3,000 × 8,500/10,000 = 2,550 bp (vaultShare * operatorShare / 10000)


    }
}
```

**Recommended Mitigation:** 총 운영자 지분 대신 자산 클래스별 운영자 지분을 구현하는 것을 고려하십시오. `operatorBeneficiariesShare`는 이러한 누출을 방지하기 위해 더 세분화되고 자산 ID에 고유해야 합니다. `_calculateOperatorShare` 및 `_calculateAndStoreVaultShares`의 해당 로직은 각 assetID에 맞게 조정되어야 합니다.

```solidity
// Add per-asset-class tracking
mapping(uint48 epoch => mapping(address operator => mapping(uint96 assetClass => uint256 share)))
    public operatorBeneficiariesSharesPerAssetClass;
```

**Suzaku:**
커밋 [f9bfdf7](https://github.com/suzaku-network/suzaku-core/pull/155/commits/f9bfdf7faa7023a0e662280a34cb41be145ba7ab)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Not all reward token rewards are claimable

**Description:** `Rewards` 계약의 `lastEpochClaimedStaker`, `lastEpochClaimedCurator` 및 `lastEpochClaimedOperator` 매핑은 스테이커/큐레이터/운영자가 보상을 청구한 마지막 에포크를 추적하지만, 보상 토큰이 아닌 스테이커/큐레이터/운영자의 주소로만 키가 지정됩니다. 즉, 스테이커/큐레이터/운영자가 특정 에포크 및 보상 토큰에 대한 보상을 청구하면, 계약은 청구된 토큰뿐만 아니라 모든 토큰에 대한 매핑을 업데이트합니다. 결과적으로 스테이커/큐레이터/운영자가 동일한 에포크에 대해 여러 토큰에서 보상을 받을 자격이 있는 경우, 한 토큰에 대한 보상을 청구하면 해당 에포크에 대한 다른 토큰의 보상을 청구할 수 없게 됩니다.

**Impact:** 동일한 에포크에 대해 여러 토큰에서 보상을 받을 자격이 있는 스테이커/큐레이터/운영자는 하나의 토큰에 대해서만 보상을 청구할 수 있습니다. 한 토큰에 대해 청구하면 계약은 모든 해당 에포크를 청구된 것으로 표시하여 다른 토큰에 대한 보상을 청구할 수 없게 만듭니다. 이는 스테이커/큐레이터/운영자의 보상 손실로 이어지며 다중 토큰 보상 분배의 예상 동작을 깨뜨립니다.

**Proof of Concept:**
1. `RewardsTest.t.sol` 파일의 `_setupStakes` 함수를 변경하십시오:
```solidity
// Sets up stakes for all operators in a given epoch
function _setupStakes(uint48 epoch, uint256 uptime) internal {
	address[] memory operators = middleware.getAllOperators();
	uint256 timestamp = middleware.getEpochStartTs(epoch);

	// Define operator stake percentages (must sum to 100%)
	uint256[] memory operatorPercentages = new uint256[](10);
	operatorPercentages[0] = 10;
	operatorPercentages[1] = 10;
	operatorPercentages[2] = 10;
	operatorPercentages[3] = 10;
	operatorPercentages[4] = 10;
	operatorPercentages[5] = 10;
	operatorPercentages[6] = 10;
	operatorPercentages[7] = 10;
	operatorPercentages[8] = 10;
	operatorPercentages[9] = 10;

	uint256 totalStakePerClass = 3_000_000 ether;

	// Track total stakes for each asset class
	uint256[] memory totalStakes = new uint256[](3); // [primary, secondary1, secondary2]

	for (uint256 i = 0; i < operators.length; i++) {
		address operator = operators[i];
		uint256 operatorStake = (totalStakePerClass * operatorPercentages[i]) / 100;
		uint256 stakePerNode = operatorStake / middleware.getOperatorNodes(operator).length;

		_setupOperatorStakes(epoch, operator, operatorStake, stakePerNode, totalStakes);
		_setupVaultDelegations(epoch, operator, operatorStake, timestamp);
		uptimeTracker.setOperatorUptimePerEpoch(epoch, operator, uptime);
	}

	// Set total stakes in L1 middleware
	middleware.setTotalStakeCache(epoch, 1, totalStakes[0]);
	middleware.setTotalStakeCache(epoch, 2, totalStakes[1]);
	middleware.setTotalStakeCache(epoch, 3, totalStakes[2]);
}

// Sets up stakes for a single operator's nodes and asset classes
function _setupOperatorStakes(
	uint48 epoch,
	address operator,
	uint256 operatorStake,
	uint256 stakePerNode,
	uint256[] memory totalStakes
) internal {
	bytes32[] memory operatorNodes = middleware.getOperatorNodes(operator);
	for (uint256 j = 0; j < operatorNodes.length; j++) {
		middleware.setNodeStake(epoch, operatorNodes[j], stakePerNode);
		totalStakes[0] += stakePerNode; // Primary stake
	}
	middleware.setOperatorStake(epoch, operator, 2, operatorStake);
	middleware.setOperatorStake(epoch, operator, 3, operatorStake);
	totalStakes[1] += operatorStake; // Secondary stake 1
	totalStakes[2] += operatorStake; // Secondary stake 2
}

// Sets up vault delegations for a single operator
function _setupVaultDelegations(
	uint48 epoch,
	address operator,
	uint256 operatorStake,
	uint256 timestamp
) internal {
	for (uint256 j = 0; j < delegators.length; j++) {
		delegators[j].setStake(
			middleware.L1_VALIDATOR_MANAGER(),
			uint96(j + 1),
			operator,
			uint48(timestamp),
			operatorStake
		);
	}
}
```

2. `RewardsTest.t.sol` 파일에 다음 테스트를 추가하십시오:
```solidity
function test_claimRewards_multipleTokens_staker() public {
	// Deploy a second reward token
	ERC20Mock rewardsToken2 = new ERC20Mock();
	rewardsToken2.mint(REWARDS_DISTRIBUTOR_ROLE, 1_000_000 * 10 ** 18);
	vm.prank(REWARDS_DISTRIBUTOR_ROLE);
```solidity
	rewardsToken2.approve(address(rewards), 1_000_000 * 10 ** 18);
	uint48 startEpoch = 1;
	uint48 numberOfEpochs = 3;
	uint256 rewardsAmount = 100_000 * 10 ** 18;

	// Set rewards for both tokens
	vm.startPrank(REWARDS_DISTRIBUTOR_ROLE);
	rewards.setRewardsAmountForEpochs(startEpoch, numberOfEpochs, address(rewardsToken2), rewardsAmount);
	vm.stopPrank();

	// Setup staker
	address staker = makeAddr("Staker");
	address vault = vaultManager.vaults(0);
	uint256 epochTs = middleware.getEpochStartTs(startEpoch);
	MockVault(vault).setActiveBalance(staker, 300_000 * 1e18);
	MockVault(vault).setTotalActiveShares(uint48(epochTs), 400_000 * 1e18);

	// Distribute rewards for epochs 1 to 3
	for (uint48 epoch = startEpoch; epoch < startEpoch + numberOfEpochs; epoch++) {
		_setupStakes(epoch, 4 hours);
		vm.warp((epoch + 3) * middleware.EPOCH_DURATION());
		address[] memory operators = middleware.getAllOperators();
		vm.prank(REWARDS_DISTRIBUTOR_ROLE);
		rewards.distributeRewards(epoch, uint48(operators.length));
	}

	// Warp to epoch 4
	vm.warp((startEpoch + numberOfEpochs) * middleware.EPOCH_DURATION());

	// Claim for rewardsToken (should succeed)
	vm.prank(staker);
	rewards.claimRewards(address(rewardsToken), staker);
	assertGt(rewardsToken.balanceOf(staker), 0, "Staker should receive rewardsToken");

	// Try to claim for rewardsToken2 (should revert)
	vm.prank(staker);
	vm.expectRevert(abi.encodeWithSelector(IRewards.AlreadyClaimedForLatestEpoch.selector, staker, numberOfEpochs));
	rewards.claimRewards(address(rewardsToken2), staker);
}

function test_claimOperatorFee_multipleTokens_operator() public {
	// Deploy a second reward token
	ERC20Mock rewardsToken2 = new ERC20Mock();
	rewardsToken2.mint(REWARDS_DISTRIBUTOR_ROLE, 1_000_000 * 10 ** 18);
	vm.prank(REWARDS_DISTRIBUTOR_ROLE);
	rewardsToken2.approve(address(rewards), 1_000_000 * 10 ** 18);

	uint48 startEpoch = 1;
	uint48 numberOfEpochs = 3;
	uint256 rewardsAmount = 100_000 * 10 ** 18;

	// Set rewards for both tokens
	vm.startPrank(REWARDS_DISTRIBUTOR_ROLE);
	rewards.setRewardsAmountForEpochs(startEpoch, numberOfEpochs, address(rewardsToken2), rewardsAmount);
	vm.stopPrank();

	// Distribute rewards for epochs 1 to 3
	for (uint48 epoch = startEpoch; epoch < startEpoch + numberOfEpochs; epoch++) {
		_setupStakes(epoch, 4 hours);
		vm.warp((epoch + 3) * middleware.EPOCH_DURATION());
		address[] memory operators = middleware.getAllOperators();
		vm.prank(REWARDS_DISTRIBUTOR_ROLE);
		rewards.distributeRewards(epoch, uint48(operators.length));
	}

	// Warp to epoch 4
	vm.warp((startEpoch + numberOfEpochs) * middleware.EPOCH_DURATION());

	address operator = middleware.getAllOperators()[0];

	// Claim for rewardsToken (should succeed)
	vm.prank(operator);
	rewards.claimOperatorFee(address(rewardsToken), operator);
	assertGt(rewardsToken.balanceOf(operator), 0, "Operator should receive rewardsToken");

	// Try to claim for rewardsToken2 (should revert)
	vm.prank(operator);
	vm.expectRevert(abi.encodeWithSelector(IRewards.AlreadyClaimedForLatestEpoch.selector, operator, numberOfEpochs));
	rewards.claimOperatorFee(address(rewardsToken2), operator);
}

function test_claimCuratorFee_multipleTokens_curator() public {
	// Deploy a second reward token
	ERC20Mock rewardsToken2 = new ERC20Mock();
	rewardsToken2.mint(REWARDS_DISTRIBUTOR_ROLE, 1_000_000 * 10 ** 18);
	vm.prank(REWARDS_DISTRIBUTOR_ROLE);
	rewardsToken2.approve(address(rewards), 1_000_000 * 10 ** 18);

	uint48 startEpoch = 1;
	uint48 numberOfEpochs = 3;
	uint256 rewardsAmount = 100_000 * 10 ** 18;

	// Set rewards for both tokens
	vm.startPrank(REWARDS_DISTRIBUTOR_ROLE);
	rewards.setRewardsAmountForEpochs(startEpoch, numberOfEpochs, address(rewardsToken2), rewardsAmount);
	vm.stopPrank();

	// Distribute rewards for epochs 1 to 3
	for (uint48 epoch = startEpoch; epoch < startEpoch + numberOfEpochs; epoch++) {
		_setupStakes(epoch, 4 hours);
		vm.warp((epoch + 3) * middleware.EPOCH_DURATION());
		address[] memory operators = middleware.getAllOperators();
		vm.prank(REWARDS_DISTRIBUTOR_ROLE);
		rewards.distributeRewards(epoch, uint48(operators.length));
	}

	// Warp to epoch 4
	vm.warp((startEpoch + numberOfEpochs) * middleware.EPOCH_DURATION());

	address vault = vaultManager.vaults(0);
	address curator = MockVault(vault).owner();

	// Claim for rewardsToken (should succeed)
	vm.prank(curator);
	rewards.claimCuratorFee(address(rewardsToken), curator);
	assertGt(rewardsToken.balanceOf(curator), 0, "Curator should receive rewardsToken");

	// Try to claim for rewardsToken2 (should revert)
	vm.prank(curator);
	vm.expectRevert(abi.encodeWithSelector(IRewards.AlreadyClaimedForLatestEpoch.selector, curator, numberOfEpochs));
	rewards.claimCuratorFee(address(rewardsToken2), curator);
}
```

**Recommended Mitigation:** `lastEpochClaimedStaker`, `lastEpochClaimedCurator` 및 `lastEpochClaimedOperator` 매핑이 사용자 주소와 보상 토큰 모두에 의해 키가 지정되도록 변경하십시오. 예를 들면 다음과 같습니다:

```solidity
mapping(address staker => mapping(address rewardToken => uint48 epoch)) public lastEpochClaimedStaker;
mapping(address curator => mapping(address rewardToken => uint48 epoch)) public lastEpochClaimedCurator;
mapping(address operator => mapping(address rewardToken => uint48 epoch)) public lastEpochClaimedOperator;
```

`claimRewards` 함수 및 기타 모든 관련 로직을 업데이트하여 이 새로운 매핑 구조를 사용하도록 하여 각 보상 토큰에 대해 청구가 별도로 추적되도록 하십시오.

**Suzaku:**
커밋 [43e09e6](https://github.com/suzaku-network/suzaku-core/pull/155/commits/43e09e66272b72b89e329403b10b0160938ad3b0)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Division by zero in rewards distribution can cause permanent lock of epoch rewards

**Description:** `Rewards::_calculateOperatorShare()` 함수에서 시스템은 현재 자산 클래스 목록을 가져오지만 과거 에포크에 대한 보상을 계산하려고 시도합니다:

```solidity
function _calculateOperatorShare(uint48 epoch, address operator) internal {
    // ... uptime checks ...

    uint96[] memory assetClasses = l1Middleware.getAssetClassIds(); // @audit Gets CURRENT asset classes
    for (uint256 i = 0; i < assetClasses.length; i++) {
        uint256 operatorStake = l1Middleware.getOperatorUsedStakeCachedPerEpoch(epoch, operator, assetClasses[i]);
        uint256 totalStake = l1Middleware.totalStakeCache(epoch, assetClasses[i]); // @audit past epoch
        uint16 assetClassShare = rewardsSharePerAssetClass[assetClasses[i]];

        uint256 shareForClass = Math.mulDiv(
            Math.mulDiv(operatorStake, BASIS_POINTS_DENOMINATOR, totalStake), // ← DIVISION BY ZERO
            assetClassShare,
            BASIS_POINTS_DENOMINATOR
        );
        totalShare += shareForClass;
    }
}
```

 이로 인해 다음과 같은 불일치가 발생합니다:

- 비활성화된 자산 클래스는 반환된 배열에 유지되지만 총 지분은 0입니다.
- 새로 추가된 자산 클래스는 포함되지만 과거 에포크에 대한 기록된 지분 데이터가 없습니다.
- 어떤 이유로든 지분이 0인 자산 클래스는 0으로 나누기를 유발합니다.

다음 시나리오를 고려하십시오:

```text
- Epoch N: 정상 운영, 운영자 보상 획득, 보상 분배 대기 중
- Epoch N+1: 프로토콜 관리자가 미래 성장을 위해 합법적으로 새 자산 클래스 추가
- Epoch N+2: 새 자산 클래스가 활성화되고 보상 지분으로 구성됨
- Epoch N+3: Epoch N에 대한 보상을 분배하려고 시도할 때 시스템은:

- 현재 자산 클래스 가져옴 (새 클래스 포함)
- totalStakeCache[N][newAssetClass]를 가져오려고 시도하지만 0임
- Math.mulDiv()에서 0으로 나누기 트리거, 트랜잭션 되돌리기(revert) 발생
```

`operatorActiveStake == 0`일 때 `_calculateAndStoreVaultShares`에서도 유사한 0으로 나누기 확인이 누락되었습니다.


**Impact:** 새 자산 클래스 ID를 추가하거나, 기존 자산 클래스 ID를 비활성화/마이그레이션하거나, 단순히 특정 assetClassId에 대해 지분이 0인 경우(최소 지분 요구 사항으로 인해 가능성은 낮지만 적극적으로 시행되지는 않음)는 모두 특정 에포크에 대한 보상 분배가 영구적으로 DOS(서비스 거부)될 수 있는 인스턴스입니다.


**Proof of Concept:** 참고: 테스트를 실행하려면 `MockAvalancheL1Middleware.sol`에 다음 변경 사항이 필요합니다:

```solidity
 uint96[] private assetClassIds = [1, 2, 3]; // Initialize with default asset classes

    function setAssetClassIds(uint96[] memory newAssetClassIds) external {
        // Clear existing array
        delete assetClassIds;

        // Copy new asset class IDs
        for (uint256 i = 0; i < newAssetClassIds.length; i++) {
            assetClassIds.push(newAssetClassIds[i]);
        }
    }

    function getAssetClassIds() external view returns (uint96[] memory) {
        return assetClassIds;
    }  //@audit this function is overwritten
```

다음 테스트를 `RewardsTest.t.sol`에 복사하십시오:

```solidity
    function test_RewardsDistribution_DivisionByZero_NewAssetClass() public {
    uint48 epoch = 1;
    _setupStakes(epoch, 4 hours);

    vm.warp((epoch + 1) * middleware.EPOCH_DURATION());

    // Add a new asset class (4) after epoch 1 has passed
    uint96 newAssetClass = 4;
    uint96[] memory currentAssetClasses = middleware.getAssetClassIds();
    uint96[] memory newAssetClasses = new uint96[](currentAssetClasses.length + 1);
    for (uint256 i = 0; i < currentAssetClasses.length; i++) {
        newAssetClasses[i] = currentAssetClasses[i];
    }
    newAssetClasses[currentAssetClasses.length] = newAssetClass;

    // Update the middleware to return the new asset class list
    middleware.setAssetClassIds(newAssetClasses);

    // Set rewards share for the new asset class
    vm.prank(REWARDS_MANAGER_ROLE);
    rewards.setRewardsShareForAssetClass(newAssetClass, 1000); // 10%

     // distribute rewards
     vm.warp((epoch + 2) * middleware.EPOCH_DURATION());
    assertEq(middleware.totalStakeCache(epoch, newAssetClass), 0, "New asset class should have zero stake for historical epoch 1");

    vm.prank(REWARDS_DISTRIBUTOR_ROLE);
    vm.expectRevert(); // Division by zero in Math.mulDiv when totalStake = 0
    rewards.distributeRewards(epoch, 1);
}
```

**Recommended Mitigation:** 0으로 나누기 확인을 추가하고 주어진 assetClassId에 대해 `total stake == 0`인 경우 단순히 다음 자산으로 이동하는 것을 고려하십시오. 또한 `vaultShare`를 계산할 때 `operatorActiveStake == 0`에 대한 0 확인을 추가하십시오.

**Suzaku:**
커밋 [9ac7bf0](https://github.com/suzaku-network/suzaku-core/pull/155/commits/9ac7bf0dc8071b42e4621d453d52227cfc27a03f)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Inaccurate uptime distribution in `UptimeTracker::computeValidatorUptime` leads to reward discrepancies

**Description:** `UptimeTracker::computeValidatorUptime` 함수는 마지막 체크포인트와 현재 에포크 사이의 모든 에포크에 총 기록된 가동 시간을 균등하게 분배하여 검증자의 가동 시간을 계산합니다. 그러나 해당 에포크 동안 검증자가 실제로 활성 상태였는지는 확인하지 않습니다. 결과적으로 검증자가 오프라인이거나 비활성 상태였던 에포크에 가동 시간이 잘못 귀속될 수 있습니다.

이 결함은 가동 시간 데이터가 검증자 보상을 결정하기 위해 `Rewards` 계약에서 사용될 때 중요해집니다. 보상 값은 에포크마다 크게 다를 수 있으므로 가동 시간을 잘못된 에포크에 귀속하면 검증자의 실제 기여도가 잘못 표시될 수 있습니다. 예를 들어, 검증자가 에포크 2(보상 비율이 더 높음)에서 4시간 동안 활성 상태였지만 가동 시간이 에포크 1(보상 비율이 더 낮음)과 에포크 2 사이에 균등하게 분할되면, 검증자의 보상 계산은 실제 활동을 반영하지 않아 부정확한 보상 분배로 이어집니다.

**Impact:**
- **검증자의 재정적 손실:** 검증자는 가동 시간이 실제로 활성 상태였던 에포크 대신 보상 비율이 낮은 에포크에 크레딧되면 마땅히 받아야 할 것보다 더 적은 보상을 받을 수 있습니다.
- **불공정한 보상 분배:** 시스템은 검증자가 활성 상태가 아니었던 에포크에 대해 실수로 보상을 제공하여 잠재적으로 악의적인 행동을 장려하거나 정직한 참가자에게 불이익을 줄 수 있습니다.
- **시스템 무결성 감소:** 부정확한 가동 시간 추적은 검증자와 사용자가 플랫폼의 공정성과 신뢰성에 의문을 제기할 수 있으므로 보상 메커니즘에 대한 신뢰를 약화시킵니다.

**Proof of Concept:**
1. `MockWarpMessenger.sol` 파일에 다음 줄을 추가하십시오:
```diff
// SPDX-License-Identifier: BUSL-1.1
pragma solidity 0.8.25;

import {
    IWarpMessenger,
    WarpMessage,
    WarpBlockHash
} from "@avalabs/subnet-evm-contracts@1.2.0/contracts/interfaces/IWarpMessenger.sol";
import {ValidatorMessages} from "@avalabs/icm-contracts/validator-manager/ValidatorMessages.sol";

contract MockWarpMessenger is IWarpMessenger {
    // Constants for uptime values from tests
    uint64 constant TWO_HOURS = 2 * 60 * 60;
    uint64 constant THREE_HOURS = 3 * 60 * 60;
    uint64 constant ONE_HOUR = 1 * 60 * 60;
    uint64 constant FOUR_HOURS = 4 * 60 * 60;
    uint64 constant FIVE_HOURS = 5 * 60 * 60;
    uint64 constant SEVEN_HOURS = 7 * 60 * 60;
    uint64 constant SIX_HOURS = 6 * 60 * 60;
    uint64 constant TWELVE_HOURS = 12 * 60 * 60;
    uint64 constant ZERO_HOURS = 0;

    // Hardcoded full node IDs based on previous test traces/deterministic generation
    // These values MUST match what operatorNodes would be in UptimeTrackerTest.setUp()
    // from your MockAvalancheL1Middleware.
    // If MockAvalancheL1Middleware changes its node generation, these must be updated.
    bytes32 constant OP_NODE_0_FULL = 0xe917244df122a1996142a1cd6c7269c136c20f47acd1ff079ee7247cae2f45c5;
    bytes32 constant OP_NODE_1_FULL = 0x69e183f32216866f48b0c092f70d99378e18023f7185e52eeee2f5bbd5255293;
    bytes32 constant OP_NODE_2_FULL = 0xfcc09d5775472c6fa988b216f5ce189894c14e093527f732b9b65da0880b5f81;

    // Constructor is now empty as we are not storing operatorNodes passed from test.
    // constructor() {} // Can be omitted for an empty constructor

    function getDerivedValidationID(bytes32 fullNodeID) internal pure returns (bytes32) {
        // Corrected conversion: bytes32 -> uint256 -> uint160 -> uint256 -> bytes32
        return bytes32(uint256(uint160(uint256(fullNodeID))));
    }

    function getVerifiedWarpMessage(
        uint32 messageIndex
    ) external view override returns (WarpMessage memory, bool) {
        // The 'require' for _operatorNodes.length is removed.

        bytes32 derivedNode0ID = getDerivedValidationID(OP_NODE_0_FULL);
        bytes32 derivedNode1ID = getDerivedValidationID(OP_NODE_1_FULL);
        bytes32 derivedNode2ID = getDerivedValidationID(OP_NODE_2_FULL);
        bytes memory payload;

        // test_ComputeValidatorUptime & test_ValidatorUptimeEvent
        if (messageIndex == 0) {
            payload = ValidatorMessages.packValidationUptimeMessage(derivedNode0ID, TWO_HOURS);
        } else if (messageIndex == 1) {
            payload = ValidatorMessages.packValidationUptimeMessage(derivedNode0ID, THREE_HOURS);
        }
        // test_ComputeOperatorUptime - first epoch (0) & test_OperatorUptimeEvent
        else if (messageIndex == 2) {
            payload = ValidatorMessages.packValidationUptimeMessage(derivedNode0ID, TWO_HOURS);
        } else if (messageIndex == 3) {
            payload = ValidatorMessages.packValidationUptimeMessage(derivedNode1ID, THREE_HOURS);
        } else if (messageIndex == 4) {
            payload = ValidatorMessages.packValidationUptimeMessage(derivedNode2ID, ONE_HOUR);
        }
        // test_ComputeOperatorUptime - second epoch (1)
        else if (messageIndex == 5) {
            payload = ValidatorMessages.packValidationUptimeMessage(derivedNode0ID, FOUR_HOURS);
        } else if (messageIndex == 6) {
            payload = ValidatorMessages.packValidationUptimeMessage(derivedNode1ID, FOUR_HOURS);
        } else if (messageIndex == 7) {
            payload = ValidatorMessages.packValidationUptimeMessage(derivedNode2ID, FOUR_HOURS);
        }
        // test_ComputeOperatorUptime - third epoch (2)
        else if (messageIndex == 8) {
            payload = ValidatorMessages.packValidationUptimeMessage(derivedNode0ID, FIVE_HOURS);
        } else if (messageIndex == 9) {
            payload = ValidatorMessages.packValidationUptimeMessage(derivedNode1ID, SEVEN_HOURS);
        } else if (messageIndex == 10) {
            payload = ValidatorMessages.packValidationUptimeMessage(derivedNode2ID, SIX_HOURS);
        }
        // test_EdgeCases
        else if (messageIndex == 11) {
            payload = ValidatorMessages.packValidationUptimeMessage(derivedNode0ID, FOUR_HOURS); // EPOCH_DURATION
        } else if (messageIndex == 12) {
            payload = ValidatorMessages.packValidationUptimeMessage(derivedNode1ID, ZERO_HOURS);
        } else if (messageIndex == 13) {
            payload = ValidatorMessages.packValidationUptimeMessage(derivedNode0ID, TWELVE_HOURS); // 3 * EPOCH_DURATION
        } else if (messageIndex == 14) {
            payload = ValidatorMessages.packValidationUptimeMessage(derivedNode0ID, ZERO_HOURS);
        } else if (messageIndex == 15) {
            payload = ValidatorMessages.packValidationUptimeMessage(derivedNode2ID, ZERO_HOURS);
        } else if (messageIndex == 16) {
            payload = ValidatorMessages.packValidationUptimeMessage(derivedNode0ID, FOUR_HOURS);
        } else if (messageIndex == 17) {
            payload = ValidatorMessages.packValidationUptimeMessage(derivedNode2ID, FOUR_HOURS);
        } else {
            return (WarpMessage({sourceChainID: bytes32(uint256(1)), originSenderAddress: address(0), payload: new bytes(0)}), false);
        }

        return (
            WarpMessage({
                sourceChainID: bytes32(uint256(1)),
                originSenderAddress: address(0),
                payload: payload
            }),
            true
        );
    }

    function sendWarpMessage(
        bytes memory // message
    ) external pure override returns (bytes32) { // messageID
        return bytes32(0);
    }

    function getBlockchainID() external pure override returns (bytes32) {
        return bytes32(uint256(1));
    }

    function getVerifiedWarpBlockHash(
        uint32 // messageIndex
    ) external pure override returns (WarpBlockHash memory warpBlockHash, bool valid) {
        warpBlockHash = WarpBlockHash({sourceChainID: bytes32(uint256(1)), blockHash: bytes32(0)});
        valid = true;
    }
}
```

2. `UptimeTrackerTest.t.sol` 파일을 업데이트하십시오:
```diff
// SPDX-License-Identifier: MIT
// SPDX-FileCopyrightText: Copyright 2024 ADDPHO
pragma solidity 0.8.25;

import {Test} from "forge-std/Test.sol";
import {console2} from "forge-std/console2.sol";
import {UptimeTracker} from "../../src/contracts/rewards/UptimeTracker.sol";
import {IUptimeTracker, LastUptimeCheckpoint} from "../../src/interfaces/rewards/IUptimeTracker.sol";
import {ValidatorMessages} from "@avalabs/icm-contracts/validator-manager/ValidatorMessages.sol";
import {MockAvalancheL1Middleware} from "../mocks/MockAvalancheL1Middleware.sol";
import {MockBalancerValidatorManager} from "../mocks/MockBalancerValidatorManager2.sol";
import {MockWarpMessenger} from "../mocks/MockWarpMessenger.sol";
import {WarpMessage, IWarpMessenger} from "@avalabs/subnet-evm-contracts@1.2.0/contracts/interfaces/IWarpMessenger.sol";
import {Rewards} from "../../src/contracts/rewards/Rewards.sol";
import {ERC20Mock} from "@openzeppelin/contracts/mocks/token/ERC20Mock.sol";

contract UptimeTrackerTest is Test {
    UptimeTracker public uptimeTracker;
    MockBalancerValidatorManager public validatorManager;
    MockAvalancheL1Middleware public middleware;
    MockWarpMessenger public warpMessenger;
+    Rewards public rewards;
+    ERC20Mock public rewardsToken;

    address public operator;
    bytes32[] public operatorNodes;
    uint48 constant EPOCH_DURATION = 4 hours;
    address constant WARP_MESSENGER_ADDR = 0x0200000000000000000000000000000000000005;
    bytes32 constant L1_CHAIN_ID = bytes32(uint256(1));
+    address constant ADMIN = address(0x1);
+    address constant REWARDS_DISTRIBUTOR = address(0x2);

    event ValidatorUptimeComputed(bytes32 indexed validationID, uint48 indexed firstEpoch, uint256 uptimeSecondsAdded, uint256 numberOfEpochs);
    event OperatorUptimeComputed(address indexed operator, uint48 indexed epoch, uint256 uptime);

    function getDerivedValidationID(bytes32 fullNodeID) internal pure returns (bytes32) {
        return bytes32(uint256(uint160(uint256(fullNodeID))));
    }

    function setUp() public {
        uint256[] memory nodesPerOperator = new uint256[](1);
        nodesPerOperator[0] = 3;

        validatorManager = new MockBalancerValidatorManager();
        middleware = new MockAvalancheL1Middleware(1, nodesPerOperator, address(validatorManager), address(0));
        uptimeTracker = new UptimeTracker(payable(address(middleware)), L1_CHAIN_ID);

        operator = middleware.getAllOperators()[0];
        operatorNodes = middleware.getActiveNodesForEpoch(operator, 0);

        warpMessenger = new MockWarpMessenger();
        vm.etch(WARP_MESSENGER_ADDR, address(warpMessenger).code);

+        rewards = new Rewards();
+        rewards.initialize(
+            ADMIN,
+            ADMIN,
+            payable(address(middleware)),
+            address(uptimeTracker),
+            1000, // protocolFee
+            2000, // operatorFee
+            1000, // curatorFee
+            11_520 // minRequiredUptime
+        );
+        vm.prank(ADMIN);
+        rewards.setRewardsDistributorRole(REWARDS_DISTRIBUTOR);

+        rewardsToken = new ERC20Mock();
+        rewardsToken.mint(REWARDS_DISTRIBUTOR, 1_000_000 * 10**18);
+        vm.prank(REWARDS_DISTRIBUTOR);
+        rewardsToken.approve(address(rewards), 1_000_000 * 10**18);
    }

	...
}
```

3. `UptimeTrackerTest.t.sol` 파일에 다음 테스트를 추가하십시오:
```solidity
function test_IncorrectUptimeDistributionWithRewards() public {
	bytes32 derivedNode1ID = getDerivedValidationID(operatorNodes[1]);

	// Warp to epoch 1
	vm.warp(EPOCH_DURATION + 1);
	uptimeTracker.computeValidatorUptime(14); // uptime = 0 hours, node 0
	uptimeTracker.computeValidatorUptime(12); // uptime = 0 hours, node 1
	uptimeTracker.computeValidatorUptime(15); // uptime = 0 hours, node 2

	// Warp to epoch 3
	vm.warp(3 * EPOCH_DURATION + 1);
	uptimeTracker.computeValidatorUptime(16); // uptime = 4 hours, node 0
	uptimeTracker.computeValidatorUptime(6); // uptime = 4 hours, node 1
	uptimeTracker.computeValidatorUptime(17); // uptime = 4 hours, node 2

	// Check uptime distribution
	uint256 uptimeEpoch1 = uptimeTracker.validatorUptimePerEpoch(1, derivedNode1ID);
	uint256 uptimeEpoch2 = uptimeTracker.validatorUptimePerEpoch(2, derivedNode1ID);
	assertEq(uptimeEpoch1, 2 * 3600, "Epoch 1 uptime should be 2 hours");
	assertEq(uptimeEpoch2, 2 * 3600, "Epoch 2 uptime should be 2 hours");

	// Set different rewards for epochs 1 and 2
	uint48 startEpoch = 1;
	uint256 rewardsAmountEpoch1 = 10000 * 10**18;
	uint256 rewardsAmountEpoch2 = 20000 * 10**18;

	vm.startPrank(REWARDS_DISTRIBUTOR);
	rewards.setRewardsAmountForEpochs(startEpoch, 1, address(rewardsToken), rewardsAmountEpoch1);
	rewards.setRewardsAmountForEpochs(startEpoch + 1, 1, address(rewardsToken), rewardsAmountEpoch2);

	// Compute operator uptime
	uptimeTracker.computeOperatorUptimeAt(operator, 1);
	uptimeTracker.computeOperatorUptimeAt(operator, 2);

	// Distribute rewards
	vm.warp(5 * EPOCH_DURATION + 1);
	rewards.distributeRewards(1, 10);
	rewards.distributeRewards(2, 10);
	vm.stopPrank();

	// Set stakes for simplicity (assume operator has full stake)
	uint256 totalStake = 1000 * 10**18;
	middleware.setTotalStakeCache(1, 1, totalStake);
	middleware.setTotalStakeCache(2, 1, totalStake);
	middleware.setOperatorStake(1, operator, 1, totalStake);
	middleware.setOperatorStake(2, operator, 1, totalStake);

	vm.prank(ADMIN);
	rewards.setRewardsShareForAssetClass(1, 10000); // 100%
	vm.stopPrank();

	// Calculate expected rewards (active only in epoch 2)
	uint256 expectedUptimeEpoch1 = 0;
	uint256 expectedUptimeEpoch2 = 4 * 3600;
	uint256 totalUptimePerEpoch = 4 * 3600;
	uint256 expectedRewardsEpoch1 = (expectedUptimeEpoch1 * rewardsAmountEpoch1) / totalUptimePerEpoch;
	uint256 expectedRewardsEpoch2 = (expectedUptimeEpoch2 * rewardsAmountEpoch2) / totalUptimePerEpoch;
	uint256 totalExpectedRewards = expectedRewardsEpoch1 + expectedRewardsEpoch2;

	// Calculate actual rewards
	uint256 actualUptimeEpoch1 = uptimeEpoch1;
	uint256 actualUptimeEpoch2 = uptimeEpoch2;
	uint256 totalActualUptimePerEpoch = actualUptimeEpoch1 + actualUptimeEpoch2;
	uint256 actualRewardsEpoch1 = (actualUptimeEpoch1 * rewardsAmountEpoch1) / totalActualUptimePerEpoch;
	uint256 actualRewardsEpoch2 = (actualUptimeEpoch2 * rewardsAmountEpoch2) / totalActualUptimePerEpoch;
	uint256 totalActualRewards = actualRewardsEpoch1 + actualRewardsEpoch2;

	// Assert the discrepancy
	assertEq(totalActualUptimePerEpoch, totalUptimePerEpoch, "Total uptime in both cases is the same");
	assertLt(totalActualRewards, totalExpectedRewards, "Validator receives fewer rewards due to incorrect uptime distribution");
}
```

**Recommended Mitigation:** 이 취약점을 해결하고 정확한 가동 시간 추적 및 보상 분배를 보장하려면 다음 조치를 권장합니다:

1. **가동 시간 귀속 로직 개선:**
   검증자가 실제로 활성 상태로 확인된 에포크에만 가동 시간을 귀속하도록 `computeValidatorUptime` 함수를 업데이트하십시오. 여기에는 각 에포크의 참여를 검증하기 위해 검증자 활동 로그 또는 상태 플래그를 상호 참조하는 작업이 포함될 수 있습니다.

2. **에포크별 가동 시간 추적 구현:**
   해당 기간 동안 검증자의 실제 활동을 기반으로 각 에포크에 대해 개별적으로 가동 시간을 기록하고 계산하도록 계약을 수정하십시오. 이를 위해서는 더 자세한 데이터 저장이 필요할 수 있지만 가동 시간 귀속의 정확성을 보장합니다.

3. **검증자 활동 데이터와 통합:**
   가능하다면 검증자 상태에 대한 실시간 또는 과거 데이터를 제공하는 외부 소스(예: 오라클 또는 온체인 활동 기록)에 계약을 연결하십시오. 이를 통해 시스템은 검증자가 언제 활성 상태였는지 정확하게 판단할 수 있습니다.

이러한 완화 조치를 적용함으로써 플랫폼은 정확한 가동 시간 추적을 달성하고 공정한 보상 분배를 보장하며 검증자와 사용자 간의 신뢰를 유지할 수 있습니다.

**Suzaku:**
인지됨(Bounded).

**Cyfrin:** 인지됨(Acknowledged).

\clearpage
## Medium Risk


### Vault initialization allows deposit whitelist with no management capability

**Description:** `VaultTokenized` 계약은 화이트리스트에 주소를 추가할 수 있는 기능 없이 입금 화이트리스트가 활성화된 상태로 초기화될 수 있습니다. 이로 인해 입금은 화이트리스트에 있는 주소로 제한되지만 주소를 화이트리스트에 추가할 수 없어 사실상 모든 입금이 차단되는 상태가 발생합니다.

이 문제는 역할 일관성을 확인하는 초기화 유효성 검사 로직에서 발생합니다:

```solidity
// File: VaultTokenized.sol
function _initialize(uint64, /* initialVersion */ address, /* owner */ bytes memory data) internal onlyInitializing {
    VaultStorageStruct storage vs = _vaultStorage();
    (InitParams memory params) = abi.decode(data, (InitParams));
    // [...truncated for brevity...]

    if (params.defaultAdminRoleHolder == address(0)) {
        if (params.depositWhitelistSetRoleHolder == address(0)) {
            if (params.depositWhitelist) {
                if (params.depositorWhitelistRoleHolder == address(0)) {
                    revert Vault__MissingRoles();
                }
            } else if (params.depositorWhitelistRoleHolder != address(0)) {
                revert Vault__InconsistentRoles();
            }
        }
        // [...code...]
    }
    // [...code...]
}
```
취약점은 일관된 화이트리스트 관리를 보장하기 위한 유효성 검사가 `params.depositWhitelistSetRoleHolder == address(0)`일 때만 발생하기 때문에 존재합니다. 호출자가 `depositWhitelistSetRoleHolder`(화이트리스트를 켜거나 끌 수 있음)를 설정했지만 `depositorWhitelistRoleHolder`(화이트리스트에 주소를 추가할 수 있음)를 설정하지 않으면 유효성 검사가 완전히 우회됩니다.

이 허점으로 인해 다음과 같은 상태로 볼트를 생성할 수 있습니다:

- `depositWhitelist = true` (화이트리스트 활성화됨)
- `depositWhitelistSetRoleHolder = someAddress` (누군가 화이트리스트를 토글할 수 있음)
- `depositorWhitelistRoleHolder = address(0)` (아무도 화이트리스트에 주소를 추가할 수 없음)


**Impact:** 볼트가 이 상태로 초기화되면:

- 입금은 화이트리스트에 있는 주소로만 제한됩니다.
- 아무도 화이트리스트에 주소를 추가할 수 있는 권한이 없습니다.
- 화이트리스트가 완전히 비활성화될 때까지 입금을 할 수 없습니다.
- 유일한 해결책은 depositWhitelistSetRoleHolder를 사용하여 화이트리스트를 완전히 끄는 것입니다.

**Proof of Concept:** `vaultTokenizedTest.t.sol`에 다음을 추가하십시오

```solidity
     // This demonstrates that when the vault is created with depositWhitelist=true
     // and depositWhitelistSetRoleHolder set but depositorWhitelistRoleHolder NOT set,
     // no deposits can be made until whitelist is turned off, because no one can add
     // addresses to the whitelist.
    function test_WhitelistInconsistency() public {
        // Create a vault with whitelisting enabled but no way to add addresses to the whitelist
        uint64 lastVersion = vaultFactory.lastVersion();

        // configuration:
        // 1. depositWhitelist = true (whitelist is enabled)
        // 2. depositWhitelistSetRoleHolder = alice (someone can toggle whitelist)
        // 3. depositorWhitelistRoleHolder = address(0) (no one can add to whitelist)
        address vaultAddress = vaultFactory.create(
            lastVersion,
            alice,
            abi.encode(
                IVaultTokenized.InitParams({
                    collateral: address(collateral),
                    burner: address(0xdEaD),
                    epochDuration: 7 days,
                    depositWhitelist: true, // Whitelist ENABLED
                    isDepositLimit: false,
                    depositLimit: 0,
                    defaultAdminRoleHolder: address(0), // No default admin
                    depositWhitelistSetRoleHolder: alice, // Alice can toggle whitelist
                    depositorWhitelistRoleHolder: address(0), // No one can add to whitelist
                    isDepositLimitSetRoleHolder: alice,
                    depositLimitSetRoleHolder: alice,
                    name: "Test",
                    symbol: "TEST"
                })
            ),
            address(delegatorFactory),
            address(slasherFactory)
        );

        vault = VaultTokenized(vaultAddress);

        assertEq(vault.depositWhitelist(), true);
        assertEq(vault.hasRole(vault.DEPOSIT_WHITELIST_SET_ROLE(), alice), true);
        assertEq(vault.hasRole(vault.DEPOSITOR_WHITELIST_ROLE(), address(0)), false);
        assertEq(vault.isDepositorWhitelisted(alice), false);
        assertEq(vault.isDepositorWhitelisted(bob), false);

        // Step 1: Try to make a deposit as bob - should fail because whitelist is on
        // and bob is not whitelisted
        collateral.transfer(bob, 100 ether);
        vm.startPrank(bob);
        collateral.approve(address(vault), 100 ether);
        vm.expectRevert(IVaultTokenized.Vault__NotWhitelistedDepositor.selector);
        vault.deposit(bob, 100 ether);
        vm.stopPrank();

        // Step 2: Alice tries to add bob to the whitelist - should fail because
        // she has the role to toggle whitelist but not to add addresses to it
        vm.startPrank(alice);
        vm.expectRevert(); // Access control error (alice doesn't have DEPOSITOR_WHITELIST_ROLE)
        vault.setDepositorWhitelistStatus(bob, true);
        vm.stopPrank();

        // Step 3: Alice tries to turn off whitelist (which she can do)
        vm.startPrank(alice);
        vault.setDepositWhitelist(false);
        vm.stopPrank();

        // Step 4: Now bob should be able to deposit
        vm.startPrank(bob);
        vault.deposit(bob, 100 ether);
        vm.stopPrank();

        // Verify final state
        assertEq(vault.activeBalanceOf(bob), 100 ether);
    }
```

**Recommended Mitigation:** `depositWhitelistSetRoleHolder` 설정 여부와 관계없이 화이트리스트 구성의 일관성을 확인하도록 초기화 유효성 검사 로직을 수정하는 것을 고려하십시오.

```solidity
if (params.defaultAdminRoleHolder == address(0)) {
    if (params.depositWhitelist && params.depositorWhitelistRoleHolder == address(0)) {
        revert Vault__MissingRoles();
    }

    if (!params.depositWhitelist && params.depositorWhitelistRoleHolder != address(0)) {
        revert Vault__InconsistentRoles();
    }
     // [...code...]

}
```


**Suzaku:**
커밋 [6b7f870](https://github.com/suzaku-network/suzaku-core/pull/155/commits/6b7f87075ae366f95fb2ebad4875f2802961799c)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Vault initialization allows zero deposit limit with no ability to modify causing denial of service

**Description:** `VaultTokenized` 계약의 초기화 절차는 입금 한도 기능을 0 값으로 활성화한 상태로 볼트를 생성할 수 있도록 허용하지만, 이 한도를 변경할 수 있는 기능은 제공하지 않습니다. 이로 인해 한도가 0으로 설정되어 있고 한도를 수정할 역할이 존재하지 않기 때문에 모든 입금이 사실상 차단되는 상태가 발생합니다.

이 문제는 역할 일관성을 확인하는 초기화 유효성 검사 로직에서 발생합니다:

```solidity
// File: VaultTokenized.sol
function _initialize(uint64, /* initialVersion */ address, /* owner */ bytes memory data) internal onlyInitializing {
    VaultStorageStruct storage vs = _vaultStorage();
    (InitParams memory params) = abi.decode(data, (InitParams));
    // [...code...]

    if (params.defaultAdminRoleHolder == address(0)) {
        // [...code...]

        if (params.isDepositLimitSetRoleHolder == address(0)) { //@audit check only happens when deposit limit set holder is zero address
            if (params.isDepositLimit) {
                if (params.depositLimit == 0 && params.depositLimitSetRoleHolder == address(0)) {
                    revert Vault__MissingRoles();
                }
            } else if (params.depositLimit != 0 || params.depositLimitSetRoleHolder != address(0)) {
                revert Vault__InconsistentRoles();
            }
        }
    }
    // [...code...]
}
```
취약점은 일관된 입금 한도 구성을 보장하기 위한 유효성 검사가 `params.isDepositLimitSetRoleHolder == address(0)`일 때만 발생하기 때문에 존재합니다. 누군가가 `isDepositLimitSetRoleHolder`(입금 한도 기능을 켜거나 끌 수 있음)를 설정했지만 `depositLimit = 0`으로 설정하면서 `depositLimitSetRoleHolder`(한도 값을 수정할 수 있음)를 설정하지 않으면 유효성 검사가 완전히 우회됩니다.

이 허점으로 인해 다음과 같은 상태로 볼트를 생성할 수 있습니다:

- `isDepositLimit = true` (입금 한도 활성화됨)
- `depositLimit = 0` (입금 허용 안 됨)
- `isDepositLimitSetRoleHolder = someAddress` (누군가 한도 기능을 토글할 수 있음)
- `depositLimitSetRoleHolder = address(0)` (아무도 한도 값을 수정할 수 없음)

**Impact:** 볼트가 이 상태로 초기화되면:

- 입금은 최대 0으로 제한됩니다(사실상 모든 입금 차단).
- 아무도 입금 한도를 변경할 수 있는 권한이 없습니다.
- 입금 한도 기능이 완전히 비활성화될 때까지 입금을 할 수 없습니다.
- 유일한 해결책은 isDepositLimitSetRoleHolder를 사용하여 입금 한도 기능을 완전히 끄는 것입니다.

이는 입금 한도 기능이 활성화될 때 한도 관리가 가능할 것이라고 볼트 설계가 가정하는 경우, 특히 볼트 입금에 대한 서비스 거부로 이어질 수 있습니다.

**Proof of Concept:** `vaultTokenizedTest.t.sol`에 다음을 추가하십시오
```solidity
  // This demonstrates that when the vault is created with isDepositLimitSetRoleHolder
    // set but depositLimitSetRoleHolder NOT set,
    // deposit limit is enabled but no one can set the limit.
    function test_DepositLimitInconsistency() public {
        // Create a vault with deposit limit enabled but no way to change the limit
        uint64 lastVersion = vaultFactory.lastVersion();

        // configuration:
        // 1. isDepositLimit = true (deposit limit is enabled)
        // 2. depositLimit = 0 (zero limit)
        // 3. isDepositLimitSetRoleHolder = alice (alice can toggle the feature)
        // 4. depositLimitSetRoleHolder = address(0) (no one can set the limit)
        address vaultAddress = vaultFactory.create(
            lastVersion,
            alice,
            abi.encode(
                IVaultTokenized.InitParams({
                    collateral: address(collateral),
                    burner: address(0xdEaD),
                    epochDuration: 7 days,
                    depositWhitelist: false,
                    isDepositLimit: true, // Deposit limit ENABLED
                    depositLimit: 0, // Zero limit
                    defaultAdminRoleHolder: address(0), // No default admin
                    depositWhitelistSetRoleHolder: alice,
                    depositorWhitelistRoleHolder: alice,
                    isDepositLimitSetRoleHolder: alice, // Alice can toggle limit feature
                    depositLimitSetRoleHolder: address(0), // No one can set the limit
                    name: "Test",
                    symbol: "TEST"
                })
            ),
            address(delegatorFactory),
            address(slasherFactory)
        );

        vault = VaultTokenized(vaultAddress);

```solidity
        // Verify initial state
        assertEq(vault.isDepositLimit(), true);
        assertEq(vault.depositLimit(), 0);
        assertEq(vault.hasRole(vault.IS_DEPOSIT_LIMIT_SET_ROLE(), alice), true);
        assertEq(vault.hasRole(vault.DEPOSIT_LIMIT_SET_ROLE(), address(0)), false);

        // Step 1: Try to make a deposit - should fail because limit is 0
        collateral.transfer(bob, 100 ether);
        vm.startPrank(bob);
        collateral.approve(address(vault), 100 ether);
        vm.expectRevert(IVaultTokenized.Vault__DepositLimitReached.selector);
        vault.deposit(bob, 100 ether);
        vm.stopPrank();

        // Step 2: Alice tries to set a deposit limit - should fail because
        // she can toggle the feature but not set the limit
        vm.startPrank(alice);
        vm.expectRevert(); // Access control error
        vault.setDepositLimit(1000 ether);
        vm.stopPrank();

        // Step 3: Alice turns off the deposit limit feature
        vm.startPrank(alice);
        vault.setIsDepositLimit(false);
        vm.stopPrank();

        // Step 4: Now bob should be able to deposit
        vm.startPrank(bob);
        vault.deposit(bob, 100 ether);
        vm.stopPrank();

        // Verify final state
        assertEq(vault.activeBalanceOf(bob), 100 ether);
    }
```

**Recommended Mitigation:** `isDepositLimitSetRoleHolder` 설정 여부와 관계없이 입금 한도 구성의 일관성을 확인하도록 초기화 유효성 검사 로직을 수정하는 것을 고려하십시오.

```solidity
if (params.defaultAdminRoleHolder == address(0)) {
    // [...whitelist code...]

    if (params.isDepositLimit && params.depositLimit == 0 && params.depositLimitSetRoleHolder == address(0)) {
        revert Vault__MissingRoles();
    }

    if (!params.isDepositLimit && (params.depositLimit != 0 || params.depositLimitSetRoleHolder != address(0))) {
        revert Vault__InconsistentRoles();
    }
}
```

**Suzaku:**
커밋 [6b7f870](https://github.com/suzaku-network/suzaku-core/pull/155/commits/6b7f87075ae366f95fb2ebad4875f2802961799c)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.



### Potential underflow in slashing logic

**Description:** `VaultTokenized::onSlash`는 계산된 `withdrawalsSlashed`가 사용 가능한 `withdrawals_`를 초과하는 시나리오를 처리할 때 연쇄적인 슬래싱 로직을 사용합니다. 이러한 경우, 초과 금액은 `nextWithdrawals`가 이 추가 슬래싱 금액을 흡수할 수 있는지 확인하지 않고 `nextWithdrawalsSlashed`에 추가됩니다.

아래 코드 스니펫에서

```solidity
// In the slashing logic for the previous epoch case
if (withdrawals_ < withdrawalsSlashed) {
    nextWithdrawalsSlashed += withdrawalsSlashed - withdrawals_;
    withdrawalsSlashed = withdrawals_;
}

// Later, this could underflow if nextWithdrawalsSlashed > nextWithdrawals
vs.withdrawals[currentEpoch_ + 1] = nextWithdrawals - nextWithdrawalsSlashed; //@audit this is adjusted without checking if nextWithdrawalsSlashed <= nextWithdrawals
```
정수 산술의 반올림과 `withdrawalsSlashed`가 나머지 `(slashedAmount - activeSlashed - nextWithdrawalsSlashed)`로 계산된다는 사실로 인해, `withdrawalsSlashed`가 일반적인 비례 분배에서 `withdrawals_`를 초과할 수 있습니다. 이로 인해 초과 슬래싱이 `nextWithdrawalsSlashed`로 연쇄됩니다.

`nextWithdrawals`가 0이거나 조정된 `nextWithdrawalsSlashed`보다 작으면, `nextWithdrawals - nextWithdrawalsSlashed` 연산은 언더플로우를 일으켜 트랜잭션이 되돌아가고(revert), 사실상 슬래싱 메커니즘에서 서비스 거부(DoS)가 발생합니다.

**Impact:** 슬래싱 트랜잭션이 되돌아가 영향을 받는 시나리오에서 슬래싱이 발생하지 않도록 합니다. 특정 시나리오에서는 악의적인 행위자가 이 시나리오를 설계하여 슬래싱을 방지할 수 있습니다.

이러한 현상이 자연스럽게 발생할 가능성은 다음과 같은 경우에 증가합니다:

- 미래 인출이 아주 적거나 0인 경우
- 슬래싱 금액이 총 사용 가능한 지분에 가까운 경우
- 정수 산술의 반올림 효과가 커지는 경우

**Proof of Concept:** 다음 시나리오를 고려하십시오:

```text
activeStake_ = 99
withdrawals_ = 3
nextWithdrawals = 0
slashableStake = 102
slashedAmount = 102

Calculation with rounding down:

activeSlashed = floor(102 * 99 / 102) = 98 (rounding down from 98.97)
nextWithdrawalsSlashed = 0
withdrawalsSlashed = 102 - 98 - 0 = 4
withdrawalsSlashed (4) > withdrawals_ (3)

The final operation nextWithdrawals (0) - nextWithdrawalsSlashed (1) causes underflow.
```

**Recommended Mitigation:** `nextWithdrawalsSlashed`가 `nextWithdrawals`를 초과하는 경우를 처리하기 위해 명시적인 확인을 추가하는 것을 고려하십시오. 이 변경은 언더플로우 조건을 방지하고 모든 상황에서 슬래싱 메커니즘이 작동하도록 보장합니다.


**Suzaku:**
커밋 [98bd130](https://github.com/suzaku-network/suzaku-core/pull/155/commits/98bd13087f37a85a1e563b9ca8e12c4fab090615)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Wrong value is returned in `upperLookupRecentCheckpoint`

**Description:** `Checkpoint::upperLookupRecentCheckpoint` 함수는 구조체에 제공된 검색 키보다 작거나 같은 키를 가진 체크포인트가 존재하는지(즉, 구조체가 비어 있지 않은지) 확인하도록 설계되었습니다. 그러한 체크포인트가 존재하면 키, 값 및 트레이스(trace) 내 체크포인트의 위치를 반환합니다.
그러나 유효한 힌트(hint)가 제공되면 의도된 목적과 다르게 잘못 동작합니다.

이는 `at`이 올바른 값을 반환하고 있다는 사실에서 비롯됩니다.

```solidity
function at(Trace256 storage self, uint32 pos) internal view returns (Checkpoint256 memory) {
        OZCheckpoints.Checkpoint208 memory checkpoint = self._trace.at(pos);
        return Checkpoint256({_key: checkpoint._key, _value: self._values[checkpoint._value]});
    }
```

**Impact:** `upperLookupRecentCheckpoint`에서 체크포인트 값을 잘못 처리하면 함수가 되돌아가거나(revert) 잘못된 값을 반환하여 체크포인트 조회 메커니즘의 신뢰성을 약화시킬 수 있습니다.

**Proof of Concept:** 다음 테스트를 실행하십시오

```solidity
contract CheckpointsBugTest is Test {
    using ExtendedCheckpoints for ExtendedCheckpoints.Trace256;

    ExtendedCheckpoints.Trace256 internal trace;

    function setUp() public {
        // Initialize the trace with some checkpoints
        trace.push(100, 1000); // timestamp 100, value 1000
        trace.push(200, 2000); // timestamp 200, value 2000
        trace.push(300, 3000); // timestamp 300, value 3000
    }

    function test_upperLookupRecentCheckpoint_withoutHint_works() public view {
        // Test without hint - this should work correctly
        (bool exists, uint48 key, uint256 value, uint32 pos) = trace.upperLookupRecentCheckpoint(150);

        assertTrue(exists, "Checkpoint should exist");
        assertEq(key, 100, "Key should be 100");
        assertEq(value, 1000, "Value should be 1000");
        assertEq(pos, 0, "Position should be 0");
    }

    // This test demonstrates the bug when using a valid hint
    function test_upperLookupRecentCheckpoint_withValidHint_demonstratesBug() public {

        // First, let's get the correct hint (position 0)
        uint32 validHint = 0;
        bytes memory hintBytes = abi.encode(validHint);

        // Call with hint - this will fail due to the bug
        vm.expectRevert(); // Expecting a revert due to array bounds error
        trace.upperLookupRecentCheckpoint(150, hintBytes);
    }

}
```

**Recommended Mitigation:** `self._values[checkpoint._value]`를 참조하는 대신 `at` 함수에서 반환된 `checkpoint._value`를 직접 사용하도록 `upperLookupRecentCheckpoint` 함수를 수정하십시오. 제안된 변경 사항은 다음과 같습니다:

```diff
function upperLookupRecentCheckpoint(
        Trace256 storage self,
        uint48 key,
        bytes memory hint_
    ) internal view returns (bool, uint48, uint256, uint32) {
        if (hint_.length == 0) {
            return upperLookupRecentCheckpoint(self, key);
        }
        uint32 hint = abi.decode(hint_, (uint32));
        Checkpoint256 memory checkpoint = at(self, hint);
        if (checkpoint._key == key) {
-           return (true, checkpoint._key, self._values[checkpoint._value], hint);
+           return (true, checkpoint._key, checkpoint._value, hint);
        }
        if (checkpoint._key < key && (hint == length(self) - 1 || at(self, hint + 1)._key > key)) {
-            return (true, checkpoint._key, self._values[checkpoint._value], hint);
+            return (true, checkpoint._key, checkpoint._value, hint);
        }
        return upperLookupRecentCheckpoint(self, key);
```

**Suzaku:**
커밋 [d198969](https://github.com/suzaku-network/suzaku-core/pull/155/commits/d198969910088d087cd52d2a6bad15fe2530df9c)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.




### Inconsistent stake calculation due to mutable `vaultManager` reference in `AvalancheL1Middleware`

**Description**

`AvalancheL1Middleware` 계약은 `vaultManager` 참조 업데이트를 허용합니다. 그러나 그렇게 하면 원본 `vaultManager`와 연결된 상태 저장 또는 과거 데이터에 의존하는 로직에 **중대한 불일치**가 발생할 수 있습니다. 주요 문제는 다음과 같습니다:

* **원본 관리자에 등록된 볼트가 새 관리자로 마이그레이션되지 않습니다**.
* 재등록 시 `enabledTime` 및 `disabledTime`과 같은 **시간 기반 메타데이터**가 재설정되어 과거 활동이 잘못 정렬됩니다.
* `getOperatorStake()`의 핵심 로직은 `_wasActiveAt()`에 따라 달라지며, 이는 볼트가 특정 에포크 동안 활성 상태였는지 확인합니다.
* `vaultManager`를 교체하면 이 확인이 중단되어 다음이 발생합니다:

  * 무시된 과거 지분
  * 잘못 계산되거나 누락된 볼트
  * 잘못된 지분 귀속

이는 미들웨어 전반에 걸쳐 주요 프로토콜 보장을 깨뜨리고 스테이킹(노드 생성) 및 보상과 같은 시스템의 정확성을 손상시킵니다.

**Illustrative Flow**

1. 원본 `vaultManager`에 볼트 `V1`을 등록합니다.
2. `setVaultManager()`를 통해 `vaultManagerV2`로 교체합니다.
3. `vaultManagerV2`에 `V1`을 재등록합니다 — 참고: `enabledTime`이 재설정됩니다.
4. 재등록 전 에포크에 대해 `getOperatorStake()`를 쿼리합니다.
5. `_wasActiveAt()`은 `false`를 반환하여 지분을 제외합니다.

**Impact**

* **데이터 불일치**: `getOperatorStake()`가 잘못된 값을 반환할 수 있습니다.
* **손상된 에포크 추적**: 볼트 상태(`_wasActiveAt`과 같은)에 의존하는 에포크 기반 로직은 신뢰할 수 없게 됩니다.

**Proof of Concept**

```solidity
    function test_changeVaultManager() public {
        // Move forward to let the vault roll epochs
        uint48 epoch = _calcAndWarpOneEpoch();

        uint256 operatorStake = middleware.getOperatorStake(alice, epoch, assetClassId);
        console2.log("Operator stake (epoch", epoch, "):", operatorStake);
        assertGt(operatorStake, 0);

        MiddlewareVaultManager vaultManager2 = new MiddlewareVaultManager(address(vaultFactory), owner, address(middleware));

        vm.startPrank(validatorManagerAddress);
        middleware.setVaultManager(address(vaultManager2));
        vm.stopPrank();

        uint256 operatorStake2 = middleware.getOperatorStake(alice, epoch, assetClassId);
        console2.log("Operator stake (epoch", epoch, "):", operatorStake2);
        assertEq(operatorStake2, 0);
    }

```

**Recommended Mitigation**

미들웨어가 초기화된 후 `vaultManager`를 임의로 업데이트하는 기능을 제거하는 것을 고려하십시오. 이 변수를 업데이트할 수 있는 유연성은 공격 표면을 확장할 가능성이 있는 의도하지 않은 부작용을 초래합니다.

**Suzaku:**
커밋 [35f6e56](https://github.com/suzaku-network/suzaku-core/pull/155/commits/35f6e5604c9d3ea77ad38424bb7587f4977f2146)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.



### Premature zeroing of epoch rewards in `claimUndistributedRewards` can block legitimate claims

**Description:** `claimUndistributedRewards` 함수는 `REWARDS_DISTRIBUTOR_ROLE`이 청구되지 않은 `epoch`에 대한 보상을 수집할 수 있도록 합니다. 여기에는 `if (currentEpoch < epoch + 2) revert EpochStillClaimable(epoch)` 확인이 포함됩니다. 즉, `currentEpoch >= epoch + 2`일 때 호출할 수 있습니다.

```solidity
/// @inheritdoc IRewards
function claimUndistributedRewards(
    uint48 epoch,
    address rewardsToken,
    address recipient
) external onlyRole(REWARDS_DISTRIBUTOR_ROLE) {
    if (recipient == address(0)) revert InvalidRecipient(recipient);

    // Check if epoch distribution is complete
    DistributionBatch storage batch = distributionBatches[epoch];
    if (!batch.isComplete) revert DistributionNotComplete(epoch);

    // Check if current epoch is at least 2 epochs ahead (to ensure all claims are done)
    uint48 currentEpoch = l1Middleware.getCurrentEpoch();
    if (currentEpoch < epoch + 2) revert EpochStillClaimable(epoch);
```

동시에 일반 사용자(스테이커, 운영자, 큐레이터)는 `epoch < currentEpoch - 1`(즉, `epoch <= currentEpoch - 2`와 동일)인 한 해당 `epoch`에 대한 보상을 청구할 수 있습니다.

```solidity
// Claiming functions
/// @inheritdoc IRewards
function claimRewards(address rewardsToken, address recipient) external {
    if (recipient == address(0)) revert InvalidRecipient(recipient);

    uint48 lastClaimedEpoch = lastEpochClaimedStaker[msg.sender];
    uint48 currentEpoch = l1Middleware.getCurrentEpoch();

    if (currentEpoch > 0 && lastClaimedEpoch >= currentEpoch - 1) {
        revert AlreadyClaimedForLatestEpoch(msg.sender, lastClaimedEpoch);
    }
```

```solidity
/// @inheritdoc IRewards
function claimOperatorFee(address rewardsToken, address recipient) external {
    if (recipient == address(0)) revert InvalidRecipient(recipient);

    uint48 currentEpoch = l1Middleware.getCurrentEpoch();
    uint48 lastClaimedEpoch = lastEpochClaimedOperator[msg.sender];

    if (currentEpoch > 0 && lastClaimedEpoch >= currentEpoch - 1) {
        revert AlreadyClaimedForLatestEpoch(msg.sender, lastClaimedEpoch);
    }
```

이는 심각한 겹침을 생성합니다. `currentEpoch == epoch + 2`일 때, `epoch`에 대한 일반 청구도 여전히 허용되며 동일한 `epoch`에 대한 `claimUndistributedRewards`도 실행될 수 있습니다.

`claimUndistributedRewards` 함수는 미분배 금액을 계산한 후 전송하기 *전에* `rewardsAmountPerTokenFromEpoch[epoch].set(rewardsToken, 0);`을 실행합니다. 이 작업은 해당 `epoch` 및 `rewardsToken`에 대한 사용 가능한 보상 기록을 즉시 0으로 만듭니다.

**Impact:** `REWARDS_DISTRIBUTOR_ROLE`이 가능한 가장 빠른 시점(즉, `currentEpoch == epoch + 2`일 때)에 `claimUndistributedRewards`를 호출하면:

1. 지정된 토큰에 대한 `rewardsAmountPerTokenFromEpoch[epoch]`가 0으로 설정됩니다.
2. 해당 `epoch`에 대한 보상을 아직 청구하지 않은 스테이커, 운영자 또는 큐레이터(유효한 청구 기간 내에 있음에도 불구하고)는 이후 각각의 청구 함수(`claimRewards`, `claimOperatorFee`, `claimCuratorFee`)가 `rewardsAmountPerTokenFromEpoch[epoch]`에서 사용 가능한 보상 0을 읽는 것을 발견하게 됩니다.
3. 이로 인해 정당한 청구자는 0 보상을 받거나 청구 트랜잭션이 되돌아가 사실상 획득한 보상을 거부당하게 됩니다.
4. 결정적으로, `REWARDS_DISTRIBUTOR_ROLE`이 청구한 "미분배" 금액은 이제 부풀려질 것입니다. 해당 사용자에게 가야 했지만 청구가 차단된 보상이 포함되기 때문입니다. 이는 특권 역할이 사용자에게 고통을 주고 자금을 유용할 수 있는 메커니즘을 구성합니다.

**Recommended Mitigation:** 정기 청구 기간이 닫힌 후 미분배 보상을 청소(sweep)할 수 있기 전에 별도의 기간이 있는지 확인하십시오. 이렇게 하면 두 작업이 모두 허용되는 겹침을 방지할 수 있습니다.

`claimUndistributedRewards`의 타이밍 확인을 수정하십시오:
조건을 다음과 같이 변경하십시오:
```solidity
if (currentEpoch < epoch + 2) revert EpochStillClaimable(epoch);
```

**Suzaku:**
커밋 [71e9093](https://github.com/suzaku-network/suzaku-core/pull/155/commits/71e9093a53160b0b641e170429a7dd56d36f272c)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.





### Unclaimable rewards for removed vaults in `Rewards::claimRewards`

**Description:** `Rewards::claimRewards` 함수에서 스테이커는 이전 에포크 동안 활성 상태였던 모든 볼트에서 보상을 청구합니다. 이러한 볼트는 `middlewareVaultManager.getVaults(epoch)`를 통해 볼트를 검색하는 `_getStakerVaults` 함수에 의해 결정됩니다.

```solidity
function _getStakerVaults(address staker, uint48 epoch) internal view returns (address[] memory) {
        address[] memory vaults = middlewareVaultManager.getVaults(epoch);
        uint48 epochStart = l1Middleware.getEpochStartTs(epoch);

        uint256 count = 0;

        // First pass: Count non-zero balance vaults
        for (uint256 i = 0; i < vaults.length; i++) {
            uint256 balance = IVaultTokenized(vaults[i]).activeBalanceOfAt(staker, epochStart, new bytes(0));
            if (balance > 0) {
                count++;
            }
        }
```

취약점은 볼트가 **보상이 분배된 후**이지만 **사용자가 청구하기 전**에 제거될 때 발생합니다. `getVaults(epoch)`는 더 이상 제거된 볼트를 포함하지 않으므로 `_getStakerVaults`는 목록에서 이를 생략하고 해당 볼트에 대한 보상은 **절대 청구되지 않아** **영구적으로 잠긴 보상**이 됩니다.

1. **Epoch 5**: 볼트 1과 볼트 2에 대한 보상이 분배됩니다.
2. 스테이커 1은 볼트 1에 스테이킹하고 보상을 받았습니다.
3. **청구하기 전**, 볼트 1이 시스템에서 제거됩니다.
4. **Epoch 6**에서, 스테이커 1이 `claimRewards`를 호출합니다.
5. `_getStakerVaults`는 내부적으로 `getVaults(epoch)`를 호출하는데, 여기에는 더 이상 볼트 1이 포함되지 않습니다.
6. 볼트 1은 건너뛰고 Epoch 5에 대한 보상은 **청구되지 않은 상태로 남습니다**.
7. 이 보상은 이제 계약에 **영구적으로 갇히게 됩니다**.


```solidity

function claimRewards(address rewardsToken, address recipient) external {
        if (recipient == address(0)) revert InvalidRecipient(recipient);

        uint48 lastClaimedEpoch = lastEpochClaimedStaker[msg.sender];
        uint48 currentEpoch = l1Middleware.getCurrentEpoch();

        if (currentEpoch > 0 && lastClaimedEpoch >= currentEpoch - 1) {
            revert AlreadyClaimedForLatestEpoch(msg.sender, lastClaimedEpoch);
        }

        uint256 totalRewards = 0;

        for (uint48 epoch = lastClaimedEpoch + 1; epoch < currentEpoch; epoch++) {
            address[] memory vaults = _getStakerVaults(msg.sender, epoch);
            uint48 epochTs = l1Middleware.getEpochStartTs(epoch);
            uint256 epochRewards = rewardsAmountPerTokenFromEpoch[epoch].get(rewardsToken);
```
**Impact:** 보상이 영구적으로 청구 불가능하게 되어 계약 내에 잠깁니다.

**Proof of Concept:**
```solidity
function test_distributeRewards_andRemoveVault(
        uint256 uptime
    ) public {
        uint48 epoch = 1;
        uptime = bound(uptime, 0, 4 hours);

        address staker = makeAddr("Staker");
        address staker1 = makeAddr("Staker1");

        // Set staker balance in vault
        address vault = vaultManager.vaults(0);
        MockVault(vault).setActiveBalance(staker, 300_000 * 1e18);
        MockVault(vault).setActiveBalance(staker1, 300_000 * 1e18);

        // Set up stakes for operators, nodes, delegators and l1 middleware
        _setupStakes(epoch, uptime);

        vm.warp((epoch + 3) * middleware.EPOCH_DURATION());
        uint256 epochTs = middleware.getEpochStartTs(epoch);
        MockVault(vault).setTotalActiveShares(uint48(epochTs), 400_000 * 1e18);

        // Distribute rewards
        test_distributeRewards(4 hours);

        vm.warp((epoch + 4) * middleware.EPOCH_DURATION());

        uint256 stakerBalanceBefore = rewardsToken.balanceOf(staker);

        vm.prank(staker);
        rewards.claimRewards(address(rewardsToken), staker);

        uint256 stakerBalanceAfter = rewardsToken.balanceOf(staker);

        uint256 stakerRewards = stakerBalanceAfter - stakerBalanceBefore;

        assertGt(stakerRewards, 0, "Staker should receive rewards");

        vaultManager.removeVault(vaultManager.vaults(0));

        uint256 stakerBalanceBefore1 = rewardsToken.balanceOf(staker1);

        vm.prank(staker1);
        rewards.claimRewards(address(rewardsToken), staker1);

        uint256 stakerBalanceAfter1 = rewardsToken.balanceOf(staker1);

        uint256 stakerRewards1 = stakerBalanceAfter1 - stakerBalanceBefore1;

        assertGt(stakerRewards1, 0, "Staker should receive rewards");
    }
```

**Recommended Mitigation:** 문제 방지를 위해 `MiddlewareVaultManager::removeVault` 함수를 제거하거나 많은 에포크가 경과한 후에만 제거를 허용하는 것을 고려하십시오.

**Suzaku:**
커밋 [b94d488](https://github.com/suzaku-network/suzaku-core/pull/155/commits/b94d4880af05185a972178aeceb2877ab260b59b)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Insufficient validation in `AvalancheL1Middleware::removeOperator` can create permanent validator lockup

**Description:** 활성 노드가 있는 운영자를 의도적이든 실수로든 제거하면 운영자 노드가 영구적으로 잠기거나 프로토콜 노드 리밸런싱 프로세스가 중단될 수 있습니다.

`AvalancheL1Middleware::disableOperator` 및 `AvalancheL1Middleware::removeOperator()`에는 제거 전 운영자에게 활성 노드가 없는지 확인하는 유효성 검사가 부족합니다.

```solidity
// AvalancheL1Middleware.sol
  function disableOperator(
        address operator
    ) external onlyOwner updateGlobalNodeStakeOncePerEpoch {
        operators.disable(operator); //@note disable an operator - this only works if operator exists
    }
function removeOperator(
    address operator
) external onlyOwner updateGlobalNodeStakeOncePerEpoch {
    (, uint48 disabledTime) = operators.getTimes(operator);
    if (disabledTime == 0 || disabledTime + SLASHING_WINDOW > Time.timestamp()) {
        revert AvalancheL1Middleware__OperatorGracePeriodNotPassed(disabledTime, SLASHING_WINDOW);
    }
    operators.remove(operator); // @audit no check
}
```

운영자가 제거되면 대부분의 노드 관리 기능은 액세스 제어 제한으로 인해 영구적으로 액세스할 수 없게 됩니다:

```solidity
modifier onlyRegisteredOperatorNode(address operator, bytes32 nodeId) {
    if (!operators.contains(operator)) {
        revert AvalancheL1Middleware__OperatorNotRegistered(operator); // @audit Always fails for removed operators
    }
    if (!operatorNodes[operator].contains(nodeId)) {
        revert AvalancheL1Middleware__NodeNotFound(nodeId);
    }
    _;
}

// Force updates also blocked
function forceUpdateNodes(address operator, uint256 limitStake) external {
    if (!operators.contains(operator)) {
        revert AvalancheL1Middleware__OperatorNotRegistered(operator); // @audit prevents any force updates
    }
    // ... rest of function never executes
}

// Individual node operations blocked
function removeNode(bytes32 nodeId) external
    onlyRegisteredOperatorNode(msg.sender, nodeId) // @audit modifier blocks removed operators
{
    _removeNode(msg.sender, nodeId);
}
```


**Impact:**
1. 운영자가 P-Chain을 종료할 수 없는 영구적인 검증자 락업
2. 언델리게이션(undelegation) 중 남은 운영자에 대한 불균형적인 지분 감소
3. 제거된 운영자는 리밸런싱할 수 없음

**Proof of Concept:** AvalancheL1MiddlewareTest.t.sol에서 테스트 실행

```solidity
function test_POC_RemoveOperatorWithActiveNodes() public {
    uint48 epoch = _calcAndWarpOneEpoch();

    // Add nodes for alice
    (bytes32[] memory nodeIds, bytes32[] memory validationIDs,) = _createAndConfirmNodes(alice, 3, 0, true);

    // Move to next epoch to ensure nodes are active
    epoch = _calcAndWarpOneEpoch();

    // Verify alice has active nodes and stake
    uint256 nodeCount = middleware.getOperatorNodesLength(alice);
    uint256 aliceStake = middleware.getOperatorStake(alice, epoch, assetClassId);
    assertGt(nodeCount, 0, "Alice should have active nodes");
    assertGt(aliceStake, 0, "Alice should have stake");

    console2.log("Before removal:");
    console2.log("  Active nodes:", nodeCount);
    console2.log("  Operator stake:", aliceStake);

    // First disable the operator (required for removal)
    vm.prank(validatorManagerAddress);
    middleware.disableOperator(alice);

    // Warp past the slashing window to allow removal
    uint48 slashingWindow = middleware.SLASHING_WINDOW();
    vm.warp(block.timestamp + slashingWindow + 1);

    // @audit Admin can remove operator with active nodes (NO VALIDATION!)
    vm.prank(validatorManagerAddress);
    middleware.removeOperator(alice);

    // Verify alice is removed from operators mapping
      address[] memory currentOperators = middleware.getAllOperators();
        bool aliceFound = false;
        for (uint256 i = 0; i < currentOperators.length; i++) {
            if (currentOperators[i] == alice) {
                aliceFound = true;
                break;
            }
        }
        console2.log("Alice found:", aliceFound);
        assertFalse(aliceFound, "Alice should not be in current operators list");

    // Verify alice's nodes still exist in storage
    assertEq(middleware.getOperatorNodesLength(alice), nodeCount, "Alice's nodes should still exist in storage");

    // Verify alice's nodes still have stake cached
    for (uint256 i = 0; i < nodeIds.length; i++) {
        uint256 nodeStake = middleware.nodeStakeCache(epoch, validationIDs[i]);
        assertGt(nodeStake, 0, "Node should still have cached stake");
    }

    // Verify stake calculations still work
    uint256 stakeAfterRemoval = middleware.getOperatorStake(alice, epoch, assetClassId);
    assertEq(stakeAfterRemoval, aliceStake, "Stake calculation should still work");

}
```

**Recommended Mitigation:** 해당 운영자의 모든 활성 노드가 제거된 경우에만 운영자 제거를 허용하는 것을 고려하십시오.


**Suzaku:**
커밋 [f0a6a49](https://github.com/suzaku-network/suzaku-core/pull/155/commits/f0a6a49313f9a9789a8fb1bcaadeabf4aa63a3f8)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Historical reward loss due to `NodeId` reuse in `AvalancheL1Middleware`

**Description:** `AvalancheL1Middleware` 계약은 신규, 담합 또는 조정된 운영자(운영자 B)가 운영자 A가 이전에 사용한 *정확히 동일한 `bytes32 nodeId`*를 사용하여 노드를 의도적으로 재등록하는 경우, 지분을 이전 운영자(운영자 A)에게 잘못 귀속시키는 취약점이 있습니다. 이 시나리오는 운영자 B가 운영자 A의 과거 `nodeId`를 알고 있으며 기본 P-Chain NodeID(`P_X`, 공유된 `bytes32 nodeId`에서 파생됨)가 운영자 A의 노드가 완전히 폐기된 후 L1 `BalancerValidatorManager`에서 재등록이 가능해졌다고 가정합니다.

이 문제는 `getOperatorUsedStakeCachedPerEpoch`가 지분 계산에 사용하는 `getActiveNodesForEpoch` 함수에서 비롯됩니다. 이 함수는 운영자 A의 과거 `nodeId`들(`operatorNodes[A]`에 영구 저장됨)을 반복합니다. 재사용된 `bytes32 nodeId_X`를 처리할 때 P-Chain NodeID 형식(`P_X`)으로 변환합니다. 그런 다음 `balancerValidatorManager.registeredValidators(P_X)`를 쿼리하여 현재 L1 `validationID`를 가져옵니다. 운영자 B가 이제 L1에 `P_X`를 재등록했기 때문에 이 쿼리는 운영자 B의 새 `validationID_B2`를 반환합니다.

그 후, `getActiveNodesForEpoch`는 L1 검증자 인스턴스 `validationID_B2`(운영자 B의 노드)가 쿼리된 에포크 동안 활성 상태였는지 확인합니다. 만약 참이라면, `validationID_B2`와 관련된 지분(`nodeStakeCache`에서 읽은 운영자 B의 지분)은 해당 에포크에 대한 운영자 A의 "사용된 지분" 계산에 잘못 포함됩니다.

**Impact:**
- 운영자 A의 "사용된 지분"은 `nodeId_X`의 악의적 또는 조정된 재사용으로 인해 운영자 B의 지분만큼 인위적으로 증가합니다. 이로 인해 운영자 A는 해당 에포크 동안 실제로 보유한 것보다 더 많은 활성 담보를 보유한 것처럼 보일 수 있습니다.
- 운영자 A는 운영자 B의 자본 및 운영 노력에 따라 귀속되어야 할 보상을 부당하게 받을 수 있으며, 이는 보상의 직접적인 오도(misallocation)로 이어집니다.

**Proof of Concept:**
1.  **Epoch E0:** 운영자 A가 `bytes32 nodeId_X`를 사용하여 노드 `N1`을 등록합니다. 이 등록은 `BalancerValidatorManager`에 의해 처리되어 P-Chain NodeID `P_X`(`nodeId_X`에서 파생됨)와 관련된 L1 `validationID_A1`이 생성됩니다. 운영자 A는 `stake_A`를 가집니다. `nodeId_X`는 `operatorNodes[A]`에 영구적으로 기록됩니다.
2.  **Epoch E1:** 노드 `N1`(`validationID_A1`)이 `BalancerValidatorManager`에서 완전히 제거됩니다. P-Chain NodeID `P_X`는 새 L1 등록을 위해 사용할 수 있게 됩니다. `nodeId_X`는 운영자 A의 과거 기록(`operatorNodes[A]`)에 남아 있습니다.
3.  **Epoch E2:**
    *   운영자 B는 운영자 A의 `nodeId_X` 이전 사용 및 L1에서 `P_X`의 가용성에 대해 알고 있거나 조정하여, *정확히 동일한 `bytes32 nodeId_X`*를 제공하여 `AvalancheL1Middleware.addNode()`를 호출합니다.
    *   운영자 B는 자신의 유효한 BLS 키를 제공합니다. 노드 소프트웨어는 L1 등록 중 P-Chain NodeID `P_X`와 연관될 수 있는 자체 유효한 TLS 키를 사용합니다(`BalancerValidatorManager`가 입력 `P_X`를 기본 식별자로 사용하거나, 엄격한 일치가 수행되는 경우 운영자 B의 TLS 키가 우연히 `P_X`에도 해당한다고 가정).
    *   `BalancerValidatorManager`는 P-Chain NodeID `P_X`에 대해 이 새로운 L1 인스턴스를 성공적으로 등록하고 새 L1 `validationID_B2`를 할당합니다. 운영자 B는 `stake_B`를 스테이킹합니다. `nodeId_X`는 이제 `operatorNodes[B]`에도 기록됩니다.
4.  **Epoch E2에서 운영자 A의 지분 쿼리:**
    *   `l1Middleware.getOperatorUsedStakeCachedPerEpoch(E2, A, PRIMARY_ASSET_CLASS)`가 호출됩니다.
    *   `getActiveNodesForEpoch(A, E2)`가 호출됩니다. `operatorNodes[A]`에서 과거의 `nodeId_X`를 찾습니다.
    *   `nodeId_X`를 `P_X`로 변환합니다.
    *   `balancerValidatorManager.registeredValidators(P_X)` 호출은 이제 `validationID_B2`(`P_X`에 대한 운영자 B의 현재 활성 L1 인스턴스)를 반환합니다.
    *   함수는 `validationID_B2`를 사용하여 L1 검증자 세부 정보를 가져온 다음 `nodeStakeCache[E2][validationID_B2]`를 사용하여 지분을 가져옵니다.
    *   **결과:** `stake_B`(운영자 B의 지분)가 Epoch E2에 대한 운영자 A의 총 "사용된 지분"에 잘못 추가됩니다.

다음 코드로 작성된 PoC는 `AvalancheL1MiddlewareTest`의 설정으로 실행할 수 있습니다:
```solidity
    function test_POC_MisattributedStake_NodeIdReused() public {
        console2.log("--- POC: Misattributed Stake due to NodeID Reuse ---");

        address operatorA = alice;
        address operatorB = charlie; // Using charlie as Operator B

        // Use a specific, predictable nodeId for the test
        bytes32 sharedNodeId_X = keccak256(abi.encodePacked("REUSED_NODE_ID_XYZ"));
        bytes memory blsKey_A = hex"A1A1A1";
        bytes memory blsKey_B = hex"B2B2B2"; // Operator B uses a different BLS key
        uint64 registrationExpiry = uint64(block.timestamp + 2 days);
        address[] memory ownerArr = new address[](1);
        ownerArr[0] = operatorA; // For simplicity, operator owns the PChainOwner
        PChainOwner memory pchainOwner_A = PChainOwner({threshold: 1, addresses: ownerArr});
        ownerArr[0] = operatorB;
        PChainOwner memory pchainOwner_B = PChainOwner({threshold: 1, addresses: ownerArr});


        // Ensure operators have some stake in the vault
        uint256 stakeAmountOpA = 20_000_000_000_000; // e.g., 20k tokens
        uint256 stakeAmountOpB = 30_000_000_000_000; // e.g., 30k tokens

        // Operator A deposits and sets shares
        collateral.transfer(staker, stakeAmountOpA);
        vm.startPrank(staker);
        collateral.approve(address(vault), stakeAmountOpA);
        (,uint256 sharesA) = vault.deposit(operatorA, stakeAmountOpA);
        vm.stopPrank();
        _setOperatorL1Shares(bob, validatorManagerAddress, assetClassId, operatorA, sharesA, delegator);

        // Operator B deposits and sets shares (can use the same vault or a different one)
        collateral.transfer(staker, stakeAmountOpB);
        vm.startPrank(staker);
        collateral.approve(address(vault), stakeAmountOpB);
        (,uint256 sharesB) = vault.deposit(operatorB, stakeAmountOpB);
        vm.stopPrank();
        _setOperatorL1Shares(bob, validatorManagerAddress, assetClassId, operatorB, sharesB, delegator);

        _calcAndWarpOneEpoch(); // Ensure stakes are recognized

        // --- Epoch E0: Operator A registers node N1 using sharedNodeId_X ---
        console2.log("Epoch E0: Operator A registers node with sharedNodeId_X");
        uint48 epochE0 = middleware.getCurrentEpoch();
        vm.prank(operatorA);
        middleware.addNode(sharedNodeId_X, blsKey_A, registrationExpiry, pchainOwner_A, pchainOwner_A, 0);
        uint32 msgIdx_A1_add = mockValidatorManager.nextMessageIndex() - 1;

        // Get the L1 validationID for Operator A's node
        bytes memory pchainNodeId_P_X_bytes = abi.encodePacked(uint160(uint256(sharedNodeId_X)));
        bytes32 validationID_A1 = mockValidatorManager.registeredValidators(pchainNodeId_P_X_bytes);
        console2.log("Operator A's L1 validationID_A1:", vm.toString(validationID_A1));

        vm.prank(operatorA);
        middleware.completeValidatorRegistration(operatorA, sharedNodeId_X, msgIdx_A1_add);

        _calcAndWarpOneEpoch(); // Move to E0 + 1 for N1 to be active
        epochE0 = middleware.getCurrentEpoch(); // Update epochE0 to where node is active

        uint256 stake_A_on_N1 = middleware.getNodeStake(epochE0, validationID_A1);
        assertGt(stake_A_on_N1, 0, "Operator A's node N1 should have stake in Epoch E0");
        console2.log("Stake of Operator A on node N1 (validationID_A1) in Epoch E0:", vm.toString(stake_A_on_N1));

        bytes32[] memory activeNodes_A_E0 = middleware.getActiveNodesForEpoch(operatorA, epochE0);
        assertEq(activeNodes_A_E0.length, 1, "Operator A should have 1 active node in E0");
        assertEq(activeNodes_A_E0[0], sharedNodeId_X, "Active node for A in E0 should be sharedNodeId_X");

        // --- Epoch E1: Node N1 (validationID_A1) is fully removed ---
        console2.log("Epoch E1: Operator A removes node N1 (validationID_A1)");
        _calcAndWarpOneEpoch();
        uint48 epochE1 = middleware.getCurrentEpoch();

        vm.prank(operatorA);
        middleware.removeNode(sharedNodeId_X);
        uint32 msgIdx_A1_remove = mockValidatorManager.nextMessageIndex() - 1;

        _calcAndWarpOneEpoch(); // To process removal in cache
        epochE1 = middleware.getCurrentEpoch(); // Update E1 to where removal is cached

        assertEq(middleware.getNodeStake(epochE1, validationID_A1), 0, "Stake for validationID_A1 should be 0 after removal in cache");

        vm.prank(operatorA);
        middleware.completeValidatorRemoval(msgIdx_A1_remove); // L1 confirms removal

        console2.log("P-Chain NodeID P_X (derived from sharedNodeId_X) is now considered available on L1.");

        activeNodes_A_E0 = middleware.getActiveNodesForEpoch(operatorA, epochE1); // Check active nodes for A in E1
        assertEq(activeNodes_A_E0.length, 0, "Operator A should have 0 active nodes in E1 after removal");

        // --- Epoch E2: Operator B re-registers a node N2 using the *exact same sharedNodeId_X* ---
        console2.log("Epoch E2: Operator B registers a new node N2 using the same sharedNodeId_X");
        _calcAndWarpOneEpoch();
        uint48 epochE2 = middleware.getCurrentEpoch();

        vm.prank(operatorB);
        middleware.addNode(sharedNodeId_X, blsKey_B, registrationExpiry, pchainOwner_B, pchainOwner_B, 0);
        uint32 msgIdx_B2_add = mockValidatorManager.nextMessageIndex() - 1;

        // Get the L1 validationID for Operator B's new node (N2)
        bytes32 validationID_B2 = mockValidatorManager.registeredValidators(pchainNodeId_P_X_bytes);
        console2.log("Operator B's new L1 validationID_B2 for sharedNodeId_X:", vm.toString(validationID_B2));
        assertNotEq(validationID_A1, validationID_B2, "L1 validationID for B's node should be different from A's old one");

        vm.prank(operatorB);
        middleware.completeValidatorRegistration(operatorB, sharedNodeId_X, msgIdx_B2_add);

        _calcAndWarpOneEpoch(); // Move to E2 + 1 for N2 to be active
        epochE2 = middleware.getCurrentEpoch(); // Update epochE2 to where node is active

        uint256 stake_B_on_N2 = middleware.getNodeStake(epochE2, validationID_B2);
        assertGt(stake_B_on_N2, 0, "Operator B's node N2 should have stake in Epoch E2");
        console2.log("Stake of Operator B on node N2 (validationID_B2) in Epoch E2:", vm.toString(stake_B_on_N2));

        bytes32[] memory activeNodes_B_E2 = middleware.getActiveNodesForEpoch(operatorB, epochE2);
        assertEq(activeNodes_B_E2.length, 1, "Operator B should have 1 active node in E2");
        assertEq(activeNodes_B_E2[0], sharedNodeId_X);


        // --- Querying for Operator A's Stake in Epoch E2 (THE VULNERABILITY) ---
        console2.log("Querying Operator A's used stake in Epoch E2 (where B's node is active with sharedNodeId_X)");

        // Ensure caches are up-to-date for Operator A for epoch E2
        middleware.calcAndCacheStakes(epochE2, middleware.PRIMARY_ASSET_CLASS());

        uint256 usedStake_A_E2 = middleware.getOperatorUsedStakeCachedPerEpoch(epochE2, operatorA, middleware.PRIMARY_ASSET_CLASS());
        console2.log("Calculated 'used stake' for Operator A in Epoch E2: ", vm.toString(usedStake_A_E2));
        // ASSERTION: Operator A's used stake should be 0 in epoch E2, as their node was removed in E1.
        // However, due to the issue, it will pick up Operator B's stake.
        assertEq(usedStake_A_E2, stake_B_on_N2, "FAIL: Operator A's used stake in E2 is misattributed with Operator B's stake!");

        // Let's ensure B's node is indeed seen as active by the mock in E2
        Validator memory validator_B2_details = mockValidatorManager.getValidator(validationID_B2);
        uint48 epochE2_startTs = middleware.getEpochStartTs(epochE2);
        bool b_node_active_in_e2 = uint48(validator_B2_details.startedAt) <= epochE2_startTs &&
                                   (validator_B2_details.endedAt == 0 || uint48(validator_B2_details.endedAt) >= epochE2_startTs);
        assertTrue(b_node_active_in_e2, "Operator B's node (validationID_B2) should be active in Epoch E2");

        console2.log("--- PoC End ---");
    }
```
Output:
```bash
Ran 1 test for test/middleware/AvalancheL1MiddlewareTest.t.sol:AvalancheL1MiddlewareTest
[PASS] test_POC_MisattributedStake_NodeIdReused() (gas: 2012990)
Logs:
  --- POC: Misattributed Stake due to NodeID Reuse ---
  Epoch E0: Operator A registers node with sharedNodeId_X
  Operator A's L1 validationID_A1: 0x2f034f048644fc181bae4bb9cab7d7c67065f4763bd63c7a694231d82397709d
  Stake of Operator A on node N1 (validationID_A1) in Epoch E0: 160000000000800
  Epoch E1: Operator A removes node N1 (validationID_A1)
  P-Chain NodeID P_X (derived from sharedNodeId_X) is now considered available on L1.
  Epoch E2: Operator B registers a new node N2 using the same sharedNodeId_X
  Operator B's new L1 validationID_B2 for sharedNodeId_X: 0xe42be7d4d8b89ec6045a7938c29cb3ad84e0852269c9ce43f370002f92894cde
  Stake of Operator B on node N2 (validationID_B2) in Epoch E2: 240000000001200
  Querying Operator A's used stake in Epoch E2 (where B's node is active with sharedNodeId_X)
  Calculated 'used stake' for Operator A in Epoch E2:  240000000001200
  --- PoC End ---

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 4.58ms (1.30ms CPU time)

Ran 1 test suite in 142.55ms (4.58ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

**Recommended Mitigation:** `AvalancheL1Middleware`는 운영자의 과거 지분을 계산할 때 활동을 *해당 운영자가 원래 등록한* L1 검증자 인스턴스와 엄격하게 연결해야 합니다.

1.  **원본 등록과 함께 L1 `validationID` 저장:** 운영자(예: 운영자 A)가 `middlewareNodeId`(예: `nodeId_X`)를 등록할 때, `BalancerValidatorManager`가 반환하는 고유한 L1 `validationID`(예: `validationID_A1`)는 미들웨어 내의 해당 특정 등록 수명 주기 동안 운영자 A 및 `nodeId_X`와 영구적으로 연결되어야 합니다.
2.  **`getActiveNodesForEpoch` 로직 수정:**
    *   `getActiveNodesForEpoch(A, epoch)`가 호출되면 운영자 A가 역사적으로 등록한 `(middlewareNodeId, original_l1_validationID)` 쌍을 반복해야 합니다.
    *   각 `original_l1_validationID`(예: `validationID_A1`)에 대해:
        *   `balancerValidatorManager.getValidator(validationID_A1)`를 쿼리하여 과거의 `startedAt` 및 `endedAt` 시간을 가져옵니다.
        *   이 특정 인스턴스 `validationID_A1`이 쿼리된 `epoch` 동안 활성 상태였다면, `validationID_A1`을 사용하여 `nodeStakeCache`에서 지분을 조회합니다.
    *   이렇게 하면 조회가 L1에서 재사용된 P-Chain NodeID `P_X`와 현재 연관되어 있지만 운영자 A가 관리하지 않는 최신 `validationID`(예: `validationID_B2`)로 "미끄러지는" 것을 방지할 수 있습니다.

**Suzaku:**
커밋 [2a88616](https://github.com/suzaku-network/suzaku-core/pull/155/commits/2a886168f4d4e63a6344c4de45d57bd8d9d851b6)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.
### Incorrect inclusion of removed nodes in `_requireMinSecondaryAssetClasses` during `forceUpdateNodes`

**Description:** `_requireMinSecondaryAssetClasses` 함수는 `forceUpdateNodes` 프로세스 내에서 사용됩니다. 실행 중에 첫 번째 반복에서 노드가 제거되면, 전체 제거 프로세스가 완료될 때까지 노드가 목록에 남아 있기 때문에 후속 반복에서도 제거된 노드가 `_requireMinSecondaryAssetClasses` 계산에 여전히 포함됩니다.

```solidity
if ((newStake < assetClasses[PRIMARY_ASSET_CLASS].minValidatorStake)
                    || !_requireMinSecondaryAssetClasses(0, operator)) {
      newStake = 0;
      _initializeEndValidationAndFlag(operator, valID, nodeId);
}
```
**Example Scenario:**

1. 5개의 노드를 관리하는 운영자에 대해 `forceUpdateNodes`가 호출되고 운영자 지분이 마지막 에포크에서 떨어졌습니다.
2. 첫 번째 반복에서 `_requireMinSecondaryAssetClasses` 값의 하락으로 인해 첫 번째 노드가 제거됩니다.
3. 두 번째 반복에서 두 번째 노드가 제거되어야 하는지 확인하는 동안, `_requireMinSecondaryAssetClasses`는 이미 제거 예정인 첫 번째 노드를 계산에 잘못 포함합니다. 결과적으로 5개의 노드가 모두 제거됩니다.

이 문제는 마지막 에포크의 운영자 지분이 떨어지고 `limitStake`가 작은 값인 경우에만 발생합니다.

**Impact:** 이 동작은 노드 제거 결정 중 부정확한 평가로 이어질 수 있습니다. `_requireMinSecondaryAssetClasses` 계산 후 제거 대상으로 표시된 노드가 존재하면 시스템이 보조 자산 클래스에 대한 최소 요구 사항이 충족되었는지 잘못 판단할 수 있습니다. 이로 인해 다음이 발생할 수 있습니다:

* 필요한 노드 제거를 방지하여 제거해야 할 노드 유지, 또는
* 노드 상태 관리에 불일치를 초래하여 잠재적으로 운영자 성능 및 시스템 무결성에 영향을 미침.

**Proof of concept:**

 ```solidity
function test_AddNodes_AndThenForceUpdate() public {
        // Move to the next epoch so we have a clean slate
        uint48 epoch = _calcAndWarpOneEpoch();

        // Prepare node data
        bytes32 nodeId = 0x00000000000000000000000039a662260f928d2d98ab5ad93aa7af8e0ee4d426;
        bytes memory blsKey = hex"1234";
        uint64 registrationExpiry = uint64(block.timestamp + 2 days);
        bytes32 nodeId1 = 0x00000000000000000000000039a662260f928d2d98ab5ad93aa7af8e0ee4d626;
        bytes memory blsKey1 = hex"1235";
        bytes32 nodeId2 = 0x00000000000000000000000039a662260f928d2d98ab5ad93aa7af8e0ee4d526;
        bytes memory blsKey2 = hex"1236";
        address[] memory ownerArr = new address[](1);
        ownerArr[0] = alice;
        PChainOwner memory ownerStruct = PChainOwner({threshold: 1, addresses: ownerArr});

        // Add node
        vm.prank(alice);
        middleware.addNode(nodeId, blsKey, registrationExpiry, ownerStruct, ownerStruct, 0);
        bytes32 validationID = mockValidatorManager.registeredValidators(abi.encodePacked(uint160(uint256(nodeId))));

        vm.prank(alice);

        middleware.addNode(nodeId1, blsKey1, registrationExpiry, ownerStruct, ownerStruct, 0);
        bytes32 validationID1 = mockValidatorManager.registeredValidators(abi.encodePacked(uint160(uint256(nodeId1))));

        vm.prank(alice);

        middleware.addNode(nodeId2, blsKey2, registrationExpiry, ownerStruct, ownerStruct, 0);
        bytes32 validationID2 = mockValidatorManager.registeredValidators(abi.encodePacked(uint160(uint256(nodeId2))));

        // Check node stake from the public getter
        uint256 nodeStake = middleware.getNodeStake(epoch, validationID);
        assertGt(nodeStake, 0, "Node stake should be >0 right after add");

        bytes32[] memory activeNodesBeforeConfirm = middleware.getActiveNodesForEpoch(alice, epoch);
        assertEq(activeNodesBeforeConfirm.length, 0, "Node shouldn't appear active before confirmation");

        vm.prank(alice);
        // messageIndex = 0 in this scenario
        middleware.completeValidatorRegistration(alice, nodeId, 0);
        middleware.completeValidatorRegistration(alice, nodeId1, 1);

        middleware.completeValidatorRegistration(alice, nodeId2, 2);

        vm.startPrank(staker);
        (uint256 burnedShares, uint256 mintedShares_) = vault.withdraw(staker, 10_000_000);
        vm.stopPrank();

        _calcAndWarpOneEpoch();

        _setupAssetClassAndRegisterVault(2, 5, collateral2, vault3, 3000 ether, 2500 ether, delegator3);
        collateral2.transfer(staker, 10);
        vm.startPrank(staker);
        collateral2.approve(address(vault3), 10);
        (uint256 depositUsedA, uint256 mintedSharesA) = vault3.deposit(staker, 10);
        vm.stopPrank();

        _warpToLastHourOfCurrentEpoch();

        middleware.forceUpdateNodes(alice, 0);
        assertEq(middleware.nodePendingRemoval(validationID), false);
    }
```

**Recommended Mitigation:** `_requireMinSecondaryAssetClasses`의 구현을 `uint256` 대신 `int256` 매개변수를 받도록 변경하는 것을 고려하십시오. `forceUpdateNodes` 메서드에서 노드가 제거되면 카운터를 증가시키고 제거된 노드 수와 동일한 음수 값을 전달하십시오.
노드 제거가 여러 에포크에 걸쳐 발생하는 경우, 제거 진행 중인 노드 카운터를 생성하는 등 솔루션을 더 견고하게 만드는 것을 고려하십시오.

```diff
- function _requireMinSecondaryAssetClasses(uint256 extraNode, address operator) internal view returns (bool) {
+ function _requireMinSecondaryAssetClasses(int256 extraNode, address operator) internal view returns (bool) {
        uint48 epoch = getCurrentEpoch();
        uint256 nodeCount = operatorNodesArray[operator].length; // existing nodes

        uint256 secCount = secondaryAssetClasses.length();
        if (secCount == 0) {
            return true;
        }
        for (uint256 i = 0; i < secCount; i++) {
            uint256 classId = secondaryAssetClasses.at(i);
            uint256 stake = getOperatorStake(operator, epoch, uint96(classId));
            // Check ratio vs. class's min stake, could add an emit here to debug
            if (stake / (nodeCount + extraNode) < assetClasses[classId].minValidatorStake) {
                return false;
            }
        }
        return true;
    }
```

**Suzaku:**
커밋 [91ae0e3](https://github.com/suzaku-network/suzaku-core/pull/155/commits/91ae0e331f4400522b071c0d7093704f1c1b2dbe)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Rewards system DOS due to unchecked asset class share and fee allocations

**Description:** REWARDS_MANAGER_ROLE은 총 할당이 100%를 초과하지 않는지 확인하지 않고 자산 클래스 보상 지분을 설정할 수 있습니다. 이는 보상 초과 할당을 가능하게 하여 잠재적인 지급 불능 및 나중 청구자에 대한 서비스 거부로 이어집니다.

프로토콜, 운영자 및 큐레이터에 대한 수수료 %를 할당할 때도 유사한 문제가 존재합니다. 이러한 각 수수료를 설정할 때 현재 로직은 수수료가 100% 미만인지 확인하지만 누적 수수료가 100% 미만인지 확인하지 못합니다.

이 문제에서는 자산 클래스 지분 문제에 중점을 둡니다. 이는 특히 여러 자산이 작용할 때 발생할 가능성이 더 높기 때문입니다.

`Rewards::setRewardsShareForAssetClass()` 함수에는 총 자산 클래스 지분이 100%를 초과하지 않도록 하는 유효성 검사가 없습니다:

```solidity
function setRewardsShareForAssetClass(uint96 assetClass, uint16 share) external onlyRole(REWARDS_MANAGER_ROLE) {
    if (share > BASIS_POINTS_DENOMINATOR) revert InvalidShare(share);
    rewardsSharePerAssetClass[assetClass] = share;  // @audit No total validation
    emit RewardsShareUpdated(assetClass, share);
}
```
`_calculateOperatorShare()` 함수는 경계 확인 없이 이러한 지분을 합산하여 잠재적으로 부풀려진 숫자를 초래합니다:

```solidity
for (uint256 i = 0; i < assetClasses.length; i++) {
    uint16 assetClassShare = rewardsSharePerAssetClass[assetClasses[i]];
    uint256 shareForClass = Math.mulDiv(operatorStake * BASIS_POINTS_DENOMINATOR / totalStake, assetClassShare, BASIS_POINTS_DENOMINATOR);
    totalShare += shareForClass; // @audit Can exceed 100%
}
```
마찬가지로 `claimOperatorFee`는 `operatorShare`가 100% 미만이라고 가정하는데, 이는 보상 지분 유효성 검사가 존재하는 경우에만 참입니다.

```solidity
 function claimOperatorFee(address rewardsToken, address recipient) external {
   // code..

    for (uint48 epoch = lastClaimedEpoch + 1; epoch < currentEpoch; epoch++) {
            uint256 operatorShare = operatorShares[epoch][msg.sender];
            if (operatorShare == 0) continue;

            // get rewards amount per token for epoch
            uint256 rewardsAmount = rewardsAmountPerTokenFromEpoch[epoch].get(rewardsToken);
            if (rewardsAmount == 0) continue;

            uint256 operatorRewards = Math.mulDiv(rewardsAmount, operatorShare, BASIS_POINTS_DENOMINATOR); //@audit this can exceed reward amount - no check here
            totalRewards += operatorRewards;
        }

}
```


**Impact:**
- 초과 할당: 관리자가 총 100%를 초과하는 자산 클래스 지분을 설정
- 극단적인 경우 마지막 청구자 배치에 대해 지급 불능이 발생할 수 있습니다. 초기 사용자가 모든 보상을 청구하여 나중 사용자가 청구할 것이 남지 않게 됩니다.

**Proof of Concept:** `RewardsTest.t.sol`에서 테스트를 실행하십시오. 이 테스트를 시연하려면 `setup()`에 다음 변경 사항이 필요합니다:

```solidity

        // mint only 100000 tokens instead of 1 million
        rewardsToken = new ERC20Mock();
        rewardsToken.mint(REWARDS_DISTRIBUTOR_ROLE, 100_000 * 10 ** 18);
        vm.prank(REWARDS_DISTRIBUTOR_ROLE);
        rewardsToken.approve(address(rewards), 100_000 * 10 ** 18);

        // disribute only to 1 epoch instead of 10
       console2.log("Setting up rewards distribution per epoch...");
        uint48 startEpoch = 1;
        uint48 numberOfEpochs = 1;
        uint256 rewardsAmount = 100_000 * 10 ** 18;

```

```solidity
    function test_DOS_RewardShareSumGreaterThan100Pct() public {
        console2.log("=== TEST BEGINS ===");


        // 1: Modify fee structure to make operators get 100% of rewards
        // this is done just to demonstrate insolvency
        vm.startPrank(REWARDS_MANAGER_ROLE);
        rewards.updateProtocolFee(0);     // 0% - no protocol fee
        rewards.updateOperatorFee(10000); // 100% - operators get everything
        rewards.updateCuratorFee(0);      // 0% - no curator fee
        vm.stopPrank();

        // 2: Set asset class shares > 100%
        vm.startPrank(REWARDS_MANAGER_ROLE);
        rewards.setRewardsShareForAssetClass(1, 8000); // 80%
        rewards.setRewardsShareForAssetClass(2, 7000); // 70%
        rewards.setRewardsShareForAssetClass(3, 5000); // 50%
        // Total: 200%
        vm.stopPrank();

        // 3: Use existing working setup for stakes
        uint48 epoch = 1;
        _setupStakes(epoch, 4 hours);

        // 4: Distribute rewards
        vm.warp((epoch + 3) * middleware.EPOCH_DURATION());
        vm.prank(REWARDS_DISTRIBUTOR_ROLE);
        rewards.distributeRewards(epoch, 10);

        //5: Check operator shares (should be inflated due to 200% asset class shares)
        address[] memory operators = middleware.getAllOperators();
        uint256 totalOperatorShares = 0;

        for (uint256 i = 0; i < operators.length; i++) {
            uint256 opShare = rewards.operatorShares(epoch, operators[i]);
            totalOperatorShares += opShare;
        }
        console2.log("Total operator shares: ", totalOperatorShares);
        assertGt(totalOperatorShares, rewards.BASIS_POINTS_DENOMINATOR(),
                "VULNERABILITY: Total operator shares exceed 100%");

        //DOS when 6'th operator tries to claim rewards
        vm.warp((epoch + 1) * middleware.EPOCH_DURATION());
        for (uint256 i = 0; i < 5; i++) {
             vm.prank(operators[i]);
            rewards.claimOperatorFee(address(rewardsToken), operators[i]);
        }

        vm.expectRevert();
        vm.prank(operators[5]);
        rewards.claimOperatorFee(address(rewardsToken), operators[5]);

    }
```
**Recommended Mitigation:** 모든 자산에 걸친 총 지분이 100%를 초과하지 않도록 하는 유효성 검사를 `setRewardsShareForAssetClass`에 추가하는 것을 고려하십시오.

```solidity
function setRewardsShareForAssetClass(uint96 assetClass, uint16 share) external onlyRole(REWARDS_MANAGER_ROLE) {
    if (share > BASIS_POINTS_DENOMINATOR) revert InvalidShare(share);

    // Calculate total shares including the new one
    uint96[] memory allAssetClasses = l1Middleware.getAssetClassIds();
    uint256 totalShares = share;

    for (uint256 i = 0; i < allAssetClasses.length; i++) {
        if (allAssetClasses[i] != assetClass) {
            totalShares += rewardsSharePerAssetClass[allAssetClasses[i]];
        }
    }

    if (totalShares > BASIS_POINTS_DENOMINATOR) {
        revert TotalAssetClassSharesExceed100Percent(totalShares); //@audit this check ensures proper distribution
    }

    rewardsSharePerAssetClass[assetClass] = share;
    emit RewardsShareUpdated(assetClass, share);
}
```

`updateProtocolFee`, `updateOperatorFee`, `updateCuratorFee`와 같은 함수에도 유사한 유효성 검사를 추가하는 것을 고려하십시오.


**Suzaku:**
커밋 [001cf04](https://github.com/suzaku-network/suzaku-core/pull/155/commits/001cf049c654d363fbd87d8f2b7c8c2aa6ba6079)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Operators can lose their reward share

**Description:** `Rewards::distributeRewards` 함수는 현재 에포크보다 최소 2 에포크 이전인 에포크에 대해 보상을 분배합니다. 그러나 이러한 보상을 계산하고 분배할 때 계약은 현재 운영자 세트를 반환하는 `l1Middleware.getAllOperators()`를 사용하여 운영자 목록을 가져옵니다. 운영자가 대상 에포크 동안 활성 상태였지만 보상 분배 전에 비활성화되고 제거된 경우 현재 운영자 목록에 포함되지 않습니다. 이는 `AvalancheL1Middleware`의 `SLASHING_WINDOW`가 `2 * epochDuration`보다 짧은 경우 발생할 수 있습니다.

**Impact:** 특정 에포크에 합법적으로 활성 상태였고 보상을 받을 자격이 있는 운영자는 보상 분배가 발생하기 전에 비활성화되고 제거되면 보상을 잃을 수 있습니다. 이를 통해 계약 소유자(또는 운영자를 제거할 권한이 있는 엔티티)는 운영자 세트를 조작하고 의도적이든 아니든 여전히 활성화되었던 에포크에 대한 보상을 받지 못하도록 운영자를 제외할 수 있어 부당한 보상 손실과 프로토콜의 신뢰 문제를 초래할 수 있습니다.

**Proof of Concept:**
1. `MockAvalancheL1Middleware.sol`을 다음과 같이 변경하십시오:
```solidity
// SPDX-License-Identifier: MIT
// SPDX-FileCopyrightText: Copyright 2024 ADDPHO

pragma solidity 0.8.25;

contract MockAvalancheL1Middleware {
    uint48 public constant EPOCH_DURATION = 4 hours;
    uint48 public constant SLASHING_WINDOW = 5 hours;
    address public immutable L1_VALIDATOR_MANAGER;
    address public immutable VAULT_MANAGER;

    mapping(uint48 => mapping(bytes32 => uint256)) public nodeStake;
    mapping(uint48 => mapping(uint96 => uint256)) public totalStakeCache;
    mapping(uint48 => mapping(address => mapping(uint96 => uint256))) public operatorStake;
    mapping(address asset => uint96 assetClass) public assetClassAsset;

    // Replace constant arrays with state variables
    address[] private OPERATORS;
    bytes32[] private VALIDATION_ID_ARRAY;

    // Add mapping from operator to their node IDs
    mapping(address => bytes32[]) private operatorToNodes;

    // Track operator status
    mapping(address => bool) public isEnabled;
    mapping(address => uint256) public disabledTime;

    uint96 primaryAssetClass = 1;
    uint96[] secondaryAssetClasses = [2, 3];

    constructor(
        uint256 operatorCount,
        uint256[] memory nodesPerOperator,
        address balancerValidatorManager,
        address vaultManager
    ) {
        require(operatorCount > 0, "At least one operator required");
        require(operatorCount == nodesPerOperator.length, "Arrays length mismatch");

        L1_VALIDATOR_MANAGER = balancerValidatorManager;
        VAULT_MANAGER = vaultManager;

        // Generate operators
        for (uint256 i = 0; i < operatorCount; i++) {
            address operator = address(uint160(0x1000 + i));
            OPERATORS.push(operator);
            isEnabled[operator] = true; // Initialize as enabled

            uint256 nodeCount = nodesPerOperator[i];
            require(nodeCount > 0, "Each operator must have at least one node");

            bytes32[] memory operatorNodes = new bytes32[](nodeCount);

            for (uint256 j = 0; j < nodeCount; j++) {
                bytes32 nodeId = keccak256(abi.encode(operator, j));
                operatorNodes[j] = nodeId;
                VALIDATION_ID_ARRAY.push(nodeId);
            }

            operatorToNodes[operator] = operatorNodes;
        }
    }

    function disableOperator(address operator) external {
        require(isEnabled[operator], "Operator not enabled");
        disabledTime[operator] = block.timestamp;
        isEnabled[operator] = false;
    }

    function removeOperator(address operator) external {
        require(!isEnabled[operator], "Operator is still enabled");
        require(block.timestamp >= disabledTime[operator] + SLASHING_WINDOW, "Slashing window not passed");

        // Remove operator from OPERATORS array
        for (uint256 i = 0; i < OPERATORS.length; i++) {
            if (OPERATORS[i] == operator) {
                OPERATORS[i] = OPERATORS[OPERATORS.length - 1];
                OPERATORS.pop();
                break;
            }
        }
    }

    function setTotalStakeCache(uint48 epoch, uint96 assetClass, uint256 stake) external {
        totalStakeCache[epoch][assetClass] = stake;
    }

    function setOperatorStake(uint48 epoch, address operator, uint96 assetClass, uint256 stake) external {
        operatorStake[epoch][operator][assetClass] = stake;
    }

    function setNodeStake(uint48 epoch, bytes32 nodeId, uint256 stake) external {
        nodeStake[epoch][nodeId] = stake;
    }

    function getNodeStake(uint48 epoch, bytes32 nodeId) external view returns (uint256) {
        return nodeStake[epoch][nodeId];
    }

    function getCurrentEpoch() external view returns (uint48) {
        return getEpochAtTs(uint48(block.timestamp));
    }

    function getAllOperators() external view returns (address[] memory) {
        return OPERATORS;
    }

    function getOperatorUsedStakeCachedPerEpoch(
        uint48 epoch,
        address operator,
        uint96 assetClass
    ) external view returns (uint256) {
        if (assetClass == 1) {
            bytes32[] storage nodesArr = operatorToNodes[operator];
            uint256 stake = 0;

            for (uint256 i = 0; i < nodesArr.length; i++) {
                bytes32 nodeId = nodesArr[i];
                stake += this.getNodeStake(epoch, nodeId);
            }
            return stake;
        } else {
            return this.getOperatorStake(operator, epoch, assetClass);
        }
    }

    function getOperatorStake(address operator, uint48 epoch, uint96 assetClass) external view returns (uint256) {
        return operatorStake[epoch][operator][assetClass];
    }

    function getEpochAtTs(uint48 timestamp) public pure returns (uint48) {
        return timestamp / EPOCH_DURATION;
    }

    function getEpochStartTs(uint48 epoch) external pure returns (uint256) {
        return epoch * EPOCH_DURATION + 1;
    }

    function getActiveAssetClasses() external view returns (uint96, uint96[] memory) {
        return (primaryAssetClass, secondaryAssetClasses);
    }

    function getAssetClassIds() external view returns (uint96[] memory) {
        uint96[] memory assetClasses = new uint96[](3);
        assetClasses[0] = primaryAssetClass;
        assetClasses[1] = secondaryAssetClasses[0];
        assetClasses[2] = secondaryAssetClasses[1];
        return assetClasses;
    }

    function getActiveNodesForEpoch(address operator, uint48) external view returns (bytes32[] memory) {
        return operatorToNodes[operator];
    }

    function getOperatorNodes(address operator) external view returns (bytes32[] memory) {
        return operatorToNodes[operator];
    }

    function getAllValidationIds() external view returns (bytes32[] memory) {
        return VALIDATION_ID_ARRAY;
    }

    function isAssetInClass(uint256 assetClass, address asset) external view returns (bool) {
        uint96 assetClassRegistered = assetClassAsset[asset];
        if (assetClassRegistered == assetClass) {
            return true;
        }
        return false;
    }

    function setAssetInAssetClass(uint96 assetClass, address asset) external {
        assetClassAsset[asset] = assetClass;
    }

    function getVaultManager() external view returns (address) {
        return VAULT_MANAGER;
    }
}
```

2. `RewardsTest.t.sol`에 다음 테스트를 추가하십시오:
```solidity
function test_distributeRewards_removedOperator() public {
	uint48 epoch = 1;
	uint256 uptime = 4 hours;

	// Set up stakes for operators in epoch 1
	_setupStakes(epoch, uptime);

	// Get the list of operators
	address[] memory operators = middleware.getAllOperators();
	address removedOperator = operators[0]; // Operator to be removed
	address activeOperator = operators[1]; // Operator to remain active

	// Disable operator[0] at the start of epoch 2
	uint256 epoch2Start = middleware.getEpochStartTs(epoch + 1); // T = 8h
	vm.warp(epoch2Start);
	middleware.disableOperator(removedOperator);

	// Warp to after the slashing window to allow removal
	uint256 removalTime = epoch2Start + middleware.SLASHING_WINDOW(); // T = 13h (8h + 5h)
	vm.warp(removalTime);
	middleware.removeOperator(removedOperator);

	// Warp to epoch 4 to distribute rewards for epoch 1
	uint256 distributionTime = middleware.getEpochStartTs(epoch + 3); // T = 16h
	vm.warp(distributionTime);

	// Distribute rewards in batches
	uint256 batchSize = 3;
	uint256 remainingOperators = middleware.getAllOperators().length; // Now 9 operators
	while (remainingOperators > 0) {
		vm.prank(REWARDS_DISTRIBUTOR_ROLE);
		rewards.distributeRewards(epoch, uint48(batchSize));
		remainingOperators = remainingOperators > batchSize ? remainingOperators - batchSize : 0;
	}

	// Verify that the removed operator has zero shares
	assertEq(
		rewards.operatorShares(epoch, removedOperator),
		0,
		"Removed operator should have zero shares despite being active in epoch 1"
	);

	// Verify that an active operator has non-zero shares
	assertGt(
		rewards.operatorShares(epoch, activeOperator),
		0,
		"Active operator should have non-zero shares"
	);
}
```

**Recommended Mitigation:** 과거 에포크에 대한 보상을 분배할 때, 현재 운영자 목록에 의존하기보다는 해당 특정 에포크 동안 활성 상태였던 운영자 목록을 가져오십시오. 이는 에포크별 운영자 상태 기록을 유지하거나 대상 에포크에 존재했던 운영자 세트를 쿼리하여 달성할 수 있습니다. 또한 보상 분배 전에 운영자가 조기에 제거되는 것을 방지하기 위해 `SLASHING_WINDOW`가 최소한 보상 분배 지연만큼 길게 되도록(즉, `SLASHING_WINDOW >= 2 * epochDuration`) 하십시오.

**Suzaku:**
커밋 [dc63daa](https://github.com/suzaku-network/suzaku-core/pull/155/commits/dc63daa5082d17ce4025eee2361fb5d36dee520d)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.



### Curators will lose reward for an epoch if they lose ownership of vault after epoch but before distribution

**Description:** `Rewards` 계약은 각 에포크에 대해 큐레이터 점유율을 계산하고 할당하기 위해 `VaultTokenized(vault).owner()`를 호출하여 보상을 받아야 하는 큐레이터(볼트 소유자)를 결정합니다. 그러나 이는 보상 분배 시점의 현재 볼트 소유자를 가져오는 것이며, 보상이 분배되는 에포크 동안의 소유자가 아닙니다. `VaultTokenized`는 `OwnableUpgradeable`을 상속하므로 `transferOwnership`을 사용하여 언제든지 소유권을 이전할 수 있습니다. 즉, 에포크 종료 후 해당 에포크에 대한 보상이 분배되기 전에 볼트 소유권이 변경되면, 새 소유자가 해당 에포크 동안 소유자가 아니었더라도 과거 에포크에 대한 큐레이터 보상을 받게 됩니다.

**Impact:** 주어진 에포크에 대한 큐레이터 보상은 해당 에포크 동안 누가 볼트를 소유했는지와 관계없이 볼트의 현재 소유자가 청구할 수 있습니다. 이를 통해 행위자는 에포크가 끝난 후 보상이 분배되기 전에 볼트를 획득하여 기여하지 않은 기간에 대한 큐레이터 보상을 소급하여 청구할 수 있습니다. 에포크 동안의 활동으로 보상을 받을 자격이 있는 원래 소유자는 정당한 보상을 잃게 됩니다. 이는 프로토콜의 공정성과 의도된 인센티브 구조를 훼손합니다.

**Proof of Concept:**
1. `MockVault.sol`을 다음과 같이 변경하십시오:
```solidity
// SPDX-License-Identifier: BUSL-1.1
// SPDX-FileCopyrightText: Copyright 2024 ADDPHO
pragma solidity 0.8.25;

interface IVaultTokenized {
    function collateral() external view returns (address);
    function delegator() external view returns (address);
    function activeBalanceOfAt(address, uint48, bytes calldata) external view returns (uint256);
    function activeSharesOfAt(address, uint48, bytes calldata) external view returns (uint256);
    function activeSharesAt(uint48, bytes calldata) external view returns (uint256);
    function owner() external view returns (address);
}

contract MockVault is IVaultTokenized {
    address private _collateral;
    address private _delegator;
    address private _owner;
    mapping(address => uint256) public activeBalance;
    mapping(uint48 => uint256) private _totalActiveShares;

    constructor(address collateralAddress, address delegatorAddress, address owner_) {
        _collateral = collateralAddress;
        _delegator = delegatorAddress;
        _owner = owner_;
    }

    function collateral() external view override returns (address) {
        return _collateral;
    }

    function delegator() external view override returns (address) {
        return _delegator;
    }

    function activeBalanceOfAt(address account, uint48, bytes calldata) public view override returns (uint256) {
        return activeBalance[account];
    }

    function setActiveBalance(address account, uint256 balance) public {
        activeBalance[account] = balance;
    }

    function owner() public view override returns (address) {
        return _owner;
    }

    function activeSharesOfAt(address account, uint48, bytes calldata) public view override returns (uint256) {
        return 100;
    }

    function activeSharesAt(uint48 timestamp, bytes calldata) public view override returns (uint256) {
        uint256 totalShares = _totalActiveShares[timestamp];
        return totalShares > 0 ? totalShares : 200; // Default to 200 if not explicitly set
    }

    function setTotalActiveShares(uint48 timestamp, uint256 totalShares) public {
        _totalActiveShares[timestamp] = totalShares;
    }

    // Added function to transfer ownership
    function transferOwnership(address newOwner) public {
        require(msg.sender == _owner, "Only owner can transfer ownership");
        require(newOwner != address(0), "New owner cannot be zero address");
        _owner = newOwner;
    }
}
```

2. `RewardsTest.t.sol`에 다음 테스트를 추가하십시오:
```solidity
function test_curatorSharesAssignedToCurrentOwnerInsteadOfHistorical() public {
	uint48 epoch = 1;
	uint256 uptime = 4 hours;

	// Step 1: Setup stakes for epoch 1 to generate curator shares
	_setupStakes(epoch, uptime);

	// Step 2: Get a vault and its initial owner (Owner A)
	address vault = vaultManager.vaults(0);
	address ownerA = MockVault(vault).owner();
	address ownerB = makeAddr("OwnerB");

	// Step 3: Warp to after epoch 1 ends
	uint256 epoch2Start = middleware.getEpochStartTs(epoch + 1);
	vm.warp(epoch2Start);

	// Step 4: Transfer ownership from Owner A to Owner B
	vm.prank(ownerA);
	MockVault(vault).transferOwnership(ownerB);

	// Step 5: Warp to a time when rewards can be distributed (after 2 epochs)
	uint256 distributionTime = middleware.getEpochStartTs(epoch + 3);
	vm.warp(distributionTime);

	// Step 6: Distribute rewards for epoch 1
	address[] memory operators = middleware.getAllOperators();
	vm.prank(REWARDS_DISTRIBUTOR_ROLE);
	rewards.distributeRewards(epoch, uint48(operators.length));

	// Step 7: Verify curator shares assignment
	uint256 curatorShareA = rewards.curatorShares(epoch, ownerA);
	uint256 curatorShareB = rewards.curatorShares(epoch, ownerB);

	assertEq(curatorShareA, 0, "Owner A should have no curator shares for epoch 1");
	assertGt(curatorShareB, 0, "Owner B should have curator shares for epoch 1");

	// Step 8: Owner B claims curator rewards
	vm.prank(ownerB);
	rewards.claimCuratorFee(address(rewardsToken), ownerB);
	uint256 ownerBBalance = rewardsToken.balanceOf(ownerB);
	assertGt(ownerBBalance, 0, "Owner B should have received curator rewards");

	// Step 9: Owner A attempts to claim and reverts
	vm.prank(ownerA);
	vm.expectRevert(abi.encodeWithSelector(IRewards.NoRewardsToClaim.selector, ownerA));
	rewards.claimCuratorFee(address(rewardsToken), ownerA);
}
```

**Recommended Mitigation:** 매 에포크가 끝날 때마다 각 볼트의 소유자를 추적하여 저장하고, 큐레이터 점유율을 할당하고 보상을 분배할 때 이 과거 소유권 데이터를 사용하십시오. 이를 통해 후속 소유권 이전에 관계없이 큐레이터 보상이 항상 각 에포크의 올바른 소유자에게 입금되도록 할 수 있습니다. 에포크 경계 또는 소유권 변경 시 소유권 스냅샷 메커니즘을 구현하면 이를 달성하는 데 도움이 될 수 있습니다.

**Suzaku:**
인지됨(Acknowledged).

**Cyfrin:** 인지됨(Acknowledged).


### Inaccurate stake calculation due to decimal mismatch across multitoken asset classes

**Description:** 현재 운영자 지분을 계산할 때 시스템은 특정 자산 클래스 ID와 관련된 모든 볼트를 반복하고 모든 스테이킹된 금액을 합산합니다. 그러나 자산 클래스는 다른 소수점 정밀도(예: 소수점 6자리인 토큰과 18자리인 토큰)를 사용하는 여러 토큰 유형(담보)으로 구성될 수 있습니다. 합산 로직이 이러한 값을 공통 소수점 기준으로 정규화하지 않기 때문에 소수점이 적은 자산(예: 소수점 6자리인 USDC)은 해당 토큰에 상당한 가치가 스테이킹되어 있더라도 최종 지분 계산에서 무시할 수 있거나 0인 값으로 기여할 수 있습니다.

1. 자산 클래스 ID 100은 다음으로 구성됩니다:

   * 소수점 6자리인 토큰 A(예: USDC).
   * 소수점 18자리인 토큰 B(예: DAI).
2. 다음과 같은 볼트가 생성됩니다:

   * 10,000 토큰 A(USDC)가 스테이킹됨 (실제 값 = 10,000 \* 10⁶).
   * 10 토큰 B(DAI)가 스테이킹됨 (실제 값 = 10 \* 10¹⁸).
3. 현재 지분 합산 로직은 단순히 볼트 전체의 원시 금액을 더합니다.
4. 소수점의 차이로 인해 10,000 토큰 A(USDC)는 토큰 B의 값과 직접 비교할 때 "0" 또는 미미한 금액으로 취급될 수 있습니다.
5. 결과적인 총 지분이 잘못 계산됩니다.

```solidity
function getOperatorStake(
        address operator,
        uint48 epoch,
        uint96 assetClassId
    ) public view returns (uint256 stake) {
        if (totalStakeCached[epoch][assetClassId]) {
            uint256 cachedStake = operatorStakeCache[epoch][assetClassId][operator];

            return cachedStake;
        }

        uint48 epochStartTs = getEpochStartTs(epoch);

        uint256 totalVaults = vaultManager.getVaultCount();

        for (uint256 i; i < totalVaults; ++i) {
            (address vault, uint48 enabledTime, uint48 disabledTime) = vaultManager.getVaultAtWithTimes(i);

            // Skip if vault not active in the target epoch
            if (!_wasActiveAt(enabledTime, disabledTime, epochStartTs)) {
                continue;
            }

            // Skip if vault asset not in AssetClassID
            if (vaultManager.getVaultAssetClass(vault) != assetClassId) {
                continue;
            }

            uint256 vaultStake = BaseDelegator(IVaultTokenized(vault).delegator()).stakeAt(
                L1_VALIDATOR_MANAGER, assetClassId, operator, epochStartTs, new bytes(0)
            );

            stake += vaultStake;
        }
```

**Impact:**
- 지분 과소 보고: 소수점이 적은 토큰은 지분 계산에서 과소 대표되거나 완전히 무시됩니다.

**Proof of Concept:** `AvalancheL1MiddlewareTest.t.sol`에 이 테스트를 추가하십시오:

```solidity
function test_operatorStakeWithoutNormalization() public {
        uint48 epoch = 1;
        uint256 uptime = 4 hours;

        // Deploy tokens with different decimals
        ERC20WithDecimals tokenA = new ERC20WithDecimals("TokenA", "TKA", 6);  // e.g., USDC
        ERC20WithDecimals tokenB = new ERC20WithDecimals("TokenB", "TKB", 18); // e.g., DAI

        // Deploy vaults and associate with asset class 1
        vm.startPrank(validatorManagerAddress);
        address vaultAddress1 = vaultFactory.create(
            1,
            bob,
            abi.encode(
                IVaultTokenized.InitParams({
                    collateral: address(tokenA),
                    burner: address(0xdEaD),
                    epochDuration: 8 hours,
                    depositWhitelist: false,
                    isDepositLimit: false,
                    depositLimit: 0,
                    defaultAdminRoleHolder: bob,
                    depositWhitelistSetRoleHolder: bob,
                    depositorWhitelistRoleHolder: bob,
                    isDepositLimitSetRoleHolder: bob,
                    depositLimitSetRoleHolder: bob,
                    name: "Test",
                    symbol: "TEST"
                })
            ),
            address(delegatorFactory),
            address(slasherFactory)
        );
        address vaultAddress2 = vaultFactory.create(
            1,
            bob,
            abi.encode(
                IVaultTokenized.InitParams({
                    collateral: address(tokenB),
                    burner: address(0xdEaD),
                    epochDuration: 8 hours,
                    depositWhitelist: false,
                    isDepositLimit: false,
                    depositLimit: 0,
                    defaultAdminRoleHolder: bob,
                    depositWhitelistSetRoleHolder: bob,
                    depositorWhitelistRoleHolder: bob,
                    isDepositLimitSetRoleHolder: bob,
                    depositLimitSetRoleHolder: bob,
                    name: "Test",
                    symbol: "TEST"
                })
            ),
            address(delegatorFactory),
            address(slasherFactory)
        );
        VaultTokenized vaultTokenA = VaultTokenized(vaultAddress1);
        VaultTokenized vaultTokenB = VaultTokenized(vaultAddress2);
        vm.startPrank(validatorManagerAddress);
        middleware.addAssetClass(2, 0, 100, address(tokenA));
        middleware.activateSecondaryAssetClass(2);
        middleware.addAssetToClass(2, address(tokenB));
        vm.stopPrank();

        address[] memory l1LimitSetRoleHolders = new address[](1);
        l1LimitSetRoleHolders[0] = bob;
        address[] memory operatorL1SharesSetRoleHolders = new address[](1);
        operatorL1SharesSetRoleHolders[0] = bob;

        address delegatorAddress2 = delegatorFactory.create(
            0,
            abi.encode(
```solidity
                address(vaultTokenA),
                abi.encode(
                    IL1RestakeDelegator.InitParams({
                        baseParams: IBaseDelegator.BaseParams({
                            defaultAdminRoleHolder: bob,
                            hook: address(0),
                            hookSetRoleHolder: bob
                        }),
                        l1LimitSetRoleHolders: l1LimitSetRoleHolders,
                        operatorL1SharesSetRoleHolders: operatorL1SharesSetRoleHolders
                    })
                )
            )
        );

        L1RestakeDelegator delegator2 = L1RestakeDelegator(delegatorAddress2);

        address delegatorAddress3 = delegatorFactory.create(
            0,
            abi.encode(
                address(vaultTokenB),
                abi.encode(
                    IL1RestakeDelegator.InitParams({
                        baseParams: IBaseDelegator.BaseParams({
                            defaultAdminRoleHolder: bob,
                            hook: address(0),
                            hookSetRoleHolder: bob
                        }),
                        l1LimitSetRoleHolders: l1LimitSetRoleHolders,
                        operatorL1SharesSetRoleHolders: operatorL1SharesSetRoleHolders
                    })
                )
            )
        );
        L1RestakeDelegator delegator3 = L1RestakeDelegator(delegatorAddress3);

        vm.prank(bob);
        vaultTokenA.setDelegator(delegatorAddress2);

        // Set the delegator in vault3
        vm.prank(bob);
        vaultTokenB.setDelegator(delegatorAddress3);

        _setOperatorL1Shares(bob, validatorManagerAddress, 2, alice, 100, delegator2);
        _setOperatorL1Shares(bob, validatorManagerAddress, 2, alice, 100, delegator3);

        vm.startPrank(validatorManagerAddress);
        vaultManager.registerVault(address(vaultTokenA), 2, 3000 ether);
        vaultManager.registerVault(address(vaultTokenB), 2, 3000 ether);
        vm.stopPrank();

        _optInOperatorVault(alice, address(vaultTokenA));
        _optInOperatorVault(alice, address(vaultTokenB));
        //_optInOperatorL1(alice, validatorManagerAddress);

        _setL1Limit(bob, validatorManagerAddress, 2, 10000 * 10**6, delegator2);
        _setL1Limit(bob, validatorManagerAddress, 2, 10 * 10**18, delegator3);

        // Define stakes without normalization
        uint256 stakeA = 10000 * 10**6; // 10,000 TokenA (6 decimals)
        uint256 stakeB = 10 * 10**18;   // 10 TokenB (18 decimals)

        tokenA.transfer(staker, stakeA);
        vm.startPrank(staker);
        tokenA.approve(address(vaultTokenA), stakeA);
        vaultTokenA.deposit(staker, stakeA);
        vm.stopPrank();

        tokenB.transfer(staker, stakeB);
        vm.startPrank(staker);
        tokenB.approve(address(vaultTokenB), stakeB);
        vaultTokenB.deposit(staker, stakeB);
        vm.stopPrank();

        vm.warp((epoch + 3) * middleware.EPOCH_DURATION());


        assertNotEq(middleware.getOperatorStake(alice, 2, 2), stakeA + stakeB);

    }
```

**Recommended Mitigation:** 합산하기 전에 모든 토큰 금액을 공통 단위(예: 18자리 소수점)로 정규화하십시오.

```diff
 function getOperatorStake(
     address operator,
     uint48 epoch,
     uint96 assetClassId
 ) public view returns (uint256 stake) {
     if (totalStakeCached[epoch][assetClassId]) {
         return operatorStakeCache[epoch][assetClassId][operator];
     }

     uint48 epochStartTs = getEpochStartTs(epoch);
     uint256 totalVaults = vaultManager.getVaultCount();

     for (uint256 i; i < totalVaults; ++i) {
         (address vault, uint48 enabledTime, uint48 disabledTime) = vaultManager.getVaultAtWithTimes(i);

         // Skip if vault not active in the target epoch
         if (!_wasActiveAt(enabledTime, disabledTime, epochStartTs)) {
             continue;
         }

         // Skip if vault asset not in AssetClassID
         if (vaultManager.getVaultAssetClass(vault) != assetClassId) {
             continue;
         }

-        uint256 vaultStake = BaseDelegator(IVaultTokenized(vault).delegator()).stakeAt(
+        uint256 rawStake = BaseDelegator(IVaultTokenized(vault).delegator()).stakeAt(
             L1_VALIDATOR_MANAGER, assetClassId, operator, epochStartTs, new bytes(0)
         );

+        // Normalize stake to 18 decimals
+        address token = IVaultTokenized(vault).underlyingToken();
+        uint8 tokenDecimals = IERC20Metadata(token).decimals();
+
+        if (tokenDecimals < 18) {
+            rawStake *= 10 ** (18 - tokenDecimals);
+        } else if (tokenDecimals > 18) {
+            rawStake /= 10 ** (tokenDecimals - 18);
+        }

-        stake += vaultStake;
+        stake += rawStake;
     }
 }

```

**Suzaku:**
커밋 [ccd5e7d](https://github.com/suzaku-network/suzaku-core/pull/155/commits/ccd5e7dd376933fd3f31acf31602ff38ee93654a)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Accumulative reward setting to prevent overwrite and support incremental updates

**Description:** `Rewards::setRewardsAmountForEpochs` 함수는 승인된 배포자가 특정 미래 에포크 수에 대해 고정 보상 금액을 설정할 수 있도록 합니다. 이는 일반적으로 보상 일정을 정의하기 위해 호출됩니다(예: 에포크 1에서 5까지 에포크당 20 토큰 할당).

그러나 현재 구현은 대상 에포크에 대해 **보상이 이미 설정되었는지 확인하지 않습니다**. 결과적으로 동일한 에포크에 대해 이 함수를 다시 호출하면 **기존 보상을 조용히 덮어쓰게 되어(overwrite)** 의도하지 않은 결과가 발생합니다:

1. **이전에 할당된 보상이 덮어씌워집니다.**
2. **새 보상이 저장소에 기록되지만, 환불하거나 재할당할 메커니즘이 없기 때문에 이전 호출의 토큰은 계약에 잠긴 상태로 남습니다.**

다음과 같이 가정해 보겠습니다:

1. 배포자가 `setRewardsAmountForEpochs(1, 1, USDC, 100)`를 호출
   → 에포크 1에 100 USDC 할당
   → 100 USDC가 계약으로 전송됨

2. 나중에 두 번째 호출이 수행됨:
   `setRewardsAmountForEpochs(1, 1, USDC, 50)`
   → 에포크 1은 이제 50 USDC로 재할당됨
   → **원래 100 USDC는 여전히 계약에 남아 있지만**, 50만 분배됨
   → 차액(100 USDC)은 갇히게 됨

**Impact:** **자금 손실:** 덮어씌워진 보상은 복구하거나 재배포할 메커니즘 없이 계약에 사실상 갇히게 됩니다.
**배포자에 대한 예상치 못한 동작:** 동일한 에포크에 대해 `setRewardsAmountForEpochs`를 두 번 호출하면 경고 없이 원래 의도를 조용히 무시합니다.

**Proof of Concept:** `Rewards`에 이 테스트를 추가하십시오:

```solidity
function test_setRewardsAmountForEpochs() public {
        uint256 rewardsAmount = 1_000_000 * 10 ** 18;
        ERC20Mock rewardsToken1 = new ERC20Mock();
        rewardsToken1.mint(REWARDS_DISTRIBUTOR_ROLE, 2 * 1_000_000 * 10 ** 18);
        vm.prank(REWARDS_DISTRIBUTOR_ROLE);
        rewardsToken1.approve(address(rewards), 2 * 1_000_000 * 10 ** 18);
        vm.prank(REWARDS_DISTRIBUTOR_ROLE);
        rewards.setRewardsAmountForEpochs(5, 1, address(rewardsToken1), rewardsAmount);
        assertEq(rewards.getRewardsAmountPerTokenFromEpoch(5, address(rewardsToken1)), rewardsAmount - Math.mulDiv(rewardsAmount, 1000, 10000));
        assertEq(rewardsToken1.balanceOf(address(rewards)), rewardsAmount);
        vm.prank(REWARDS_DISTRIBUTOR_ROLE);
        rewards.setRewardsAmountForEpochs(5, 1, address(rewardsToken1), rewardsAmount);
        assertEq(rewardsToken1.balanceOf(address(rewards)), rewardsAmount * 2 );

        assertEq(rewards.getRewardsAmountPerTokenFromEpoch(5, address(rewardsToken1)), (rewardsAmount - Math.mulDiv(rewardsAmount, 1000, 10000)) * 2);
    }
```

 **Recommended Mitigation:**

이미 값이 설정된 에포크에 대한 **보상 덮어쓰기를 방지**하려면 `setRewardsAmountForEpochs` 함수에 **가드 절(guard clause)**을 추가하십시오:

```diff
for (uint48 i = 0; i < numberOfEpochs; i++) {

-    rewardsAmountPerTokenFromEpoch[startEpoch + i].set(rewardsToken, rewardsAmount);
+    uint256 existingAmount = rewardsAmountPerTokenFromEpoch[targetEpoch].get(rewardsToken);
+   rewardsAmountPerTokenFromEpoch[targetEpoch].set(rewardsToken, existingAmount + rewardsAmount);
}
```

**Suzaku:**
커밋 [a5c4913](https://github.com/suzaku-network/suzaku-core/pull/155/commits/a5c4913f9f73aa9f87e0026f4fac1cade95b8e64)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Rewards distribution DoS due to uncached secondary asset classes

**Description:** 보상 계산은 적절한 폴백 로직이 있는 getTotalStake() 함수를 사용하는 대신 totalStakeCache 매핑에 직접 액세스합니다:

```solidity
function _calculateOperatorShare(uint48 epoch, address operator) internal {
  // code..

  uint96[] memory assetClasses = l1Middleware.getAssetClassIds();
  for (uint256 i = 0; i < assetClasses.length; i++) {
       uint256 totalStake = l1Middleware.totalStakeCache(epoch, assetClasses[i]); //@audit directly accesses totalStakeCache
  }
}
```
특정 작업만이 보조 자산 클래스에 대한 캐싱을 트리거합니다:
```solidity
// @audit following only cache PRIMARY_ASSET_CLASS (asset class 1)
addNode(...) updateStakeCache(getCurrentEpoch(), PRIMARY_ASSET_CLASS)
forceUpdateNodes(...) updateStakeCache(getCurrentEpoch(), PRIMARY_ASSET_CLASS)

// @audit only caches the specific asset class being slashed
slash(epoch, operator, amount, assetClassId) updateStakeCache(epoch, assetClassId)
```
보조 자산 클래스(2, 3 등)는 다음과 같은 경우에만 캐시됩니다:

- 해당 특정 자산 클래스에 대해 슬래싱이 발생하는 경우 (드뭄)
- 수동인 calcAndCacheStakes() 호출 (개입 필요)

결과적으로 보상 배포자가 `distributeRewards`를 호출할 때, 스테이크가 캐시되지 않은 특정 자산 클래스 ID의 경우 `_calculateOperatorShare`에서 0으로 나누기 오류가 발생합니다.

**Impact:** 영향을 받는 에포크에 대한 보상 분배가 실패합니다. DoS는 일시적이며, 특정 자산 클래스 ID에 대해 `calcAndCacheStakes`를 호출하여 수동으로 개입하면 DoS 오류를 해결할 수 있다는 점을 주목할 가치가 있습니다.

**Proof of Concept:** `RewardsTest.t.sol`에 다음 테스트를 추가하십시오

```solidity
function test_RewardsDistributionDOS_With_UncachedSecondaryAssetClasses() public {
    uint48 epoch = 1;
    uint256 uptime = 4 hours;

    // Setup stakes for operators normally
    _setupStakes(epoch, uptime);

    // Set totalStakeCache to 0 for secondary asset classes to simulate uncached state
    middleware.setTotalStakeCache(epoch, 2, 0); // Secondary asset class 2
    middleware.setTotalStakeCache(epoch, 3, 0); // Secondary asset class 3

    // Keep primary asset class cached (this would be cached by addNode/forceUpdateNodes)
    middleware.setTotalStakeCache(epoch, 1, 100000); // This stays cached


    // Move to epoch where distribution is allowed (must be at least 2 epochs ahead)
    vm.warp((epoch + 3) * middleware.EPOCH_DURATION());

    // Attempt to distribute rewards - this should fail due to division by zero
    // when _calculateOperatorShare tries to calculate rewards for uncached secondary asset classes
    vm.expectRevert(); // This should revert due to division by zero in share calculation

    vm.prank(REWARDS_DISTRIBUTOR_ROLE);
    rewards.distributeRewards(epoch, 3);
}

```

**Recommended Mitigation:** assetId에 대한 지분이 존재하지 않는 경우 확인하고 캐싱하는 것을 고려하십시오.

```diff solidity
function _calculateOperatorShare(uint48 epoch, address operator) internal {
  // code..

  uint96[] memory assetClasses = l1Middleware.getAssetClassIds();
  for (uint256 i = 0; i < assetClasses.length; i++) {
++       uint256 totalStake = l1Middleware.totalStakeCache(epoch, assetClasses[i]);
++       if (totalStake == 0) {
++            l1Middleware.calcAndCacheStakes(epoch, assetClasses[i]);
++             totalStake = l1Middleware.totalStakeCache(epoch, assetClasses[i]);
++       }
       // code
  }
}
```

**Suzaku:**
커밋 [f76d1f4](https://github.com/suzaku-network/suzaku-core/pull/155/commits/f76d1f44208e9e882047713a8c49d16cccc69e36)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Uptime loss due to integer division in `UptimeTracker::computeValidatorUptime` can make validator lose entire rewards for an epoch

**Description:** `UptimeTracker::computeValidatorUptime`은 여러 에포크에 걸쳐 가동 시간을 분배할 때 정수 나눗셈 정리(truncation)로 인해 검증자 가동 시간을 잃습니다. 이로 인해 검증자는 합법적으로 획득한 가동 시간에 대한 보상 자격을 잃게 됩니다.

```solidity
// UptimeTracker::computeValidatorUptime
// Distribute the recorded uptime across multiple epochs
if (elapsedEpochs >= 1) {
    uint256 uptimePerEpoch = uptimeToDistribute / elapsedEpochs; // @audit integer division
    for (uint48 i = 0; i < elapsedEpochs; i++) {
        uint48 epoch = lastUptimeEpoch + i;
        if (isValidatorUptimeSet[epoch][validationID] == true) {
            break;
        }
        validatorUptimePerEpoch[epoch][validationID] = uptimePerEpoch; // @audit time loss due to precision
        isValidatorUptimeSet[epoch][validationID] = true;
    }
}
```

Solidity의 정수 나눗셈은 나머지를 버립니다:
- `uptimeToDistribute / elapsedEpochs`는 `uptimeToDistribute % elapsedEpochs`를 잃습니다.
- 잃어버린 나머지는 향후 계산에서 결코 복구되지 않습니다.
- `computeValidatorUptime`을 호출할 때마다 최대 `elapsedEpochs - 1`초를 잃을 수 있습니다.

이 정리는 가동 시간이 보상을 받기 위한 최소 가동 시간 임계값에 가까운 엣지 케이스에서 심각한 문제가 될 수 있습니다. 잘린 가동 시간이 `minRequiredTime`보다 1초라도 적으면 해당 에포크에 대한 검증자의 전체 보상은 0이 됩니다.

```solidity
//Rewards.sol
function _calculateOperatorShare(uint48 epoch, address operator) internal {
    uint256 uptime = uptimeTracker.operatorUptimePerEpoch(epoch, operator);
    if (uptime < minRequiredUptime) {
        operatorBeneficiariesShares[epoch][operator] = 0;  // @audit no rewards
        operatorShares[epoch][operator] = 0;
        return;
    }
    // ... calculate rewards normally
}
```
**Impact:** 정수 나눗셈으로 인한 정밀도 손실로 인해 특정 엣지 케이스에서 검증자가 에포크 전체 보상을 잃을 수 있습니다.

**Proof of Concept:** `UptimeTrackerTest.t.sol`에 테스트를 추가하십시오

```solidity
 function test_UptimeTruncationCausesRewardLoss() public {


        uint256 MIN_REQUIRED_UPTIME = 11_520;

        console2.log("Minimum required uptime per epoch:", MIN_REQUIRED_UPTIME, "seconds");
        console2.log("Epoch duration:", EPOCH_DURATION, "seconds");

        // Demonstrate how small time lost can have big impact
        uint256 totalUptime = (MIN_REQUIRED_UPTIME * 3) - 2; // 34,558 seconds across 3 epochs
        uint256 elapsedEpochs = 3;
        uint256 uptimePerEpoch = totalUptime / elapsedEpochs; // 11,519 per epoch
        uint256 remainder = totalUptime % elapsedEpochs; // 2 seconds lost

        console2.log("3 epochs scenario:");
        console2.log("  Total uptime:", totalUptime, "seconds (9.6 hours!)");
        console2.log("  Epochs:", elapsedEpochs);
        console2.log("  Per epoch after division:", uptimePerEpoch, "seconds");
        console2.log("  Lost to truncation:", remainder, "seconds");
        console2.log("  Result: ALL 3 epochs FAIL threshold!");


        // Verify
        assertFalse(uptimePerEpoch >= MIN_REQUIRED_UPTIME, "Fails threshold due to truncation");
    }
```

**Recommended Mitigation:** 남은 가동 시간을 가능한 한 많은 에포크에 균등하게 분배하거나 단순히 최신 에포크에 분배하는 것을 고려하십시오.

**Suzaku:**
커밋 [6c37d1c](https://github.com/suzaku-network/suzaku-core/pull/155/commits/6c37d1c3791565fcdf6e097e0587d956ac68f676)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Operator can over allocate the same stake to unlimited nodes within one epoch causing weight inflation and reward theft

**Description:** `AvalancheL1Middleware::addNode()` 함수는 운영자가 새 P-chain 검증자를 등록하기 위해 호출하는 진입점입니다.
요청을 수락하기 전에 함수는 `_getOperatorAvailableStake()`에 운영자의 담보 중 여유분이 얼마나 되는지 묻습니다. 해당 도우미 함수는 `totalStake`에서 `operatorLockedStake[operator]`만 뺍니다.

```solidity
    function addNode(
        bytes32 nodeId,
        bytes calldata blsKey,
        uint64 registrationExpiry,
        PChainOwner calldata remainingBalanceOwner,
        PChainOwner calldata disableOwner,
        uint256 stakeAmount // optional
    ) external updateStakeCache(getCurrentEpoch(), PRIMARY_ASSET_CLASS) updateGlobalNodeStakeOncePerEpoch {
        ...
        ...

        bytes32 valId = balancerValidatorManager.registeredValidators(abi.encodePacked(uint160(uint256(nodeId))));
        uint256 available = _getOperatorAvailableStake(operator);
        ...
        ...
    }
```


```solidity
    function _getOperatorAvailableStake(
        address operator
    ) internal view returns (uint256) {
        uint48 epoch = getCurrentEpoch();
        uint256 totalStake = getOperatorStake(operator, epoch, PRIMARY_ASSET_CLASS);

        ...
        ...

        uint256 lockedStake = operatorLockedStake[operator];
        if (totalStake <= lockedStake) {
            return 0;
        }
        return totalStake - lockedStake;
    }
```

그러나 `AvalancheL1Middleware::addNode()`는 `newStake`를 사용하기로 결정한 후에도 `operatorLockedStake`를 **증가시키지 않습니다**.
동일한 에포크에서 호출이 발생하는 한 `lockedStake`는 0으로 유지되므로, `addNode()`에 대한 모든 후속 호출은 *전체* 담보를 여전히 "무료"라고 간주하고 최대 가중치의 또 다른 검증자를 등록할 수 있습니다.
따라서 에포크가 진행되는 동안 운영자별 지분이 이중으로 계산됩니다.

초과 노드 제거는 `AvalancheL1Middleware::forceUpdateNodes()`를 통해서만 가능한데, 이는 `onlyDuringFinalWindowOfEpoch`에 의해 통제되며 에포크의 `UPDATE_WINDOW`가 경과한 **후에만** 실행할 수 있습니다.
보상 회계(`getOperatorUsedStakeCachedPerEpoch() → getActiveNodesForEpoch()`)는 에포크 **시작** 시 검증자를 스냅샷으로 찍기 때문에, 에포크 초기에 생성된 모든 추가 노드는 전체 보상 기간 동안 완전히 활성 상태인 것으로 간주됩니다.
따라서 공격자는 자신의 가중치를 부풀리고 에포크 보상 풀의 불균형적인 몫을 차지할 수 있습니다.

```solidity
function getActiveNodesForEpoch(
    address operator,
    uint48 epoch
) external view returns (bytes32[] memory activeNodeIds) {
    uint48 epochStartTs = getEpochStartTs(epoch);

    // Gather all nodes from the never-removed set
    bytes32[] memory allNodeIds = operatorNodes[operator].values();

    bytes32[] memory temp = new bytes32[](allNodeIds.length);
    uint256 activeCount;

    for (uint256 i = 0; i < allNodeIds.length; i++) {
        bytes32 nodeId = allNodeIds[i];
        bytes32 validationID =
            balancerValidatorManager.registeredValidators(abi.encodePacked(uint160(uint256(nodeId))));
        Validator memory validator = balancerValidatorManager.getValidator(validationID);

        if (_wasActiveAt(uint48(validator.startedAt), uint48(validator.endedAt), epochStartTs)) {
            temp[activeCount++] = nodeId;
        }
    }

    activeNodeIds = new bytes32[](activeCount);
    for (uint256 j = 0; j < activeCount; j++) {
        activeNodeIds[j] = temp[j];
    }
}
```
**Impact:** * 악의적인 운영자는 추가 담보 없이 무제한의 검증자를 실행하여 의도된 운영자별 제한을 초과할 수 있습니다.
* 보상 분배가 왜곡됩니다: 운영자, 볼트 및 큐레이터 지분의 합계가 100%를 초과하고 정직한 참가자가 희석됩니다.

**Proof of Concept:**
```solidity
// ─────────────────────────────────────────────────────────────────────────────
// PoC: Exploiting the missing stake-locking in addNode()
// ─────────────────────────────────────────────────────────────────────────────
import {AvalancheL1MiddlewareTest} from "./AvalancheL1MiddlewareTest.t.sol";

import {Rewards}            from "src/contracts/rewards/Rewards.sol";
import {MockUptimeTracker}  from "../mocks/MockUptimeTracker.sol";
import {ERC20Mock}          from "@openzeppelin/contracts/mocks/token/ERC20Mock.sol";

import {VaultTokenized}     from "src/contracts/vault/VaultTokenized.sol";
import {PChainOwner}        from "@avalabs/teleporter/validator-manager/interfaces/IValidatorManager.sol";

import {console2}           from "forge-std/console2.sol";

contract PoCMissingLockingRewards is AvalancheL1MiddlewareTest {
    // ── helpers & globals ────────────────────────────────────────────────────
    MockUptimeTracker internal uptimeTracker;   // Simulates uptime records
    Rewards          internal rewards;          // Rewards contract under test
    ERC20Mock        internal rewardsToken;     // Dummy ERC-20 for payouts

    address internal REWARDS_MANAGER_ROLE    = makeAddr("REWARDS_MANAGER_ROLE");
    address internal REWARDS_DISTRIBUTOR_ROLE = makeAddr("REWARDS_DISTRIBUTOR_ROLE");

    // Main exploit routine ----------------------------------------------------
    function test_PoCRewardsManipulated() public {
        _setupRewards();                                      // 1. deploy & fund rewards system
        address[] memory operators = middleware.getAllOperators();

        // --- STEP 1: move to a fresh epoch ----------------------------------
        console2.log("Warping to a fresh epoch");
        vm.warp(middleware.getEpochStartTs(middleware.getCurrentEpoch() + 1));
        uint48 epoch = middleware.getCurrentEpoch();          // snapshot for later

        // --- STEP 2: create *too many* nodes for Alice ----------------------
        console2.log("Creating 4 nodes for Alice with the same stake");
        uint256 stake1 = 200_000_000_002_000;          // Alice's full stake
        _createAndConfirmNodes(alice, 4, stake1, true);     // ← re-uses the same stake 4 times

        // Charlie behaves honestly – one node, fully staked
        console2.log("Creating 1 node for Charlie with the full stake");
        uint256 stake2 = 150_000_000_000_000;
        _createAndConfirmNodes(charlie, 1, stake2, true);

        // --- STEP 3: Remove Alice's unbacked nodes at the earliest possible moment------
        console2.log("Removing Alice's unbacked nodes at the earliest possible moment");
        uint48 nextEpoch = middleware.getCurrentEpoch() + 1;
        uint256 afterUpdateWindow =
            middleware.getEpochStartTs(nextEpoch) + middleware.UPDATE_WINDOW() + 1;
        vm.warp(afterUpdateWindow);
        middleware.forceUpdateNodes(alice, type(uint256).max);

        // --- STEP 4: advance to the rewards epoch ---------------------------
        console2.log("Advancing and caching stakes");
        _calcAndWarpOneEpoch();                               // epoch rollover, stakes cached
        middleware.calcAndCacheStakes(epoch, assetClassId);   // ensure operator stakes cached

        // --- STEP 5: mark everyone as fully up for the epoch ----------------
        console2.log("Marking everyone as fully up for the epoch");
        for (uint i = 0; i < operators.length; i++) {
            uptimeTracker.setOperatorUptimePerEpoch(epoch, operators[i], 4 hours);
        }

        // --- STEP 6: advance a few epochs so rewards can be distributed -------
        console2.log("Advancing 3 epochs so rewards can be distributed ");
        _calcAndWarpOneEpoch(3);

        // --- STEP 7: distribute rewards (attacker gets oversized share) -----
        console2.log("Distributing rewards");
        vm.prank(REWARDS_DISTRIBUTOR_ROLE);
        rewards.distributeRewards(epoch, uint48(operators.length));

        // --- STEP 8: verify that the share accounting exceeds 100 % ---------
        console2.log("Verifying that the share accounting exceeds 100 %");
        uint256 totalShares = 0;
        // operator shares
        for (uint i = 0; i < operators.length; i++) {
            totalShares += rewards.operatorShares(epoch, operators[i]);
        }
        // vault shares
        address[] memory vaults = vaultManager.getVaults(epoch);
        for (uint i = 0; i < vaults.length; i++) {
            totalShares += rewards.vaultShares(epoch, vaults[i]);
        }
        // curator shares
        for (uint i = 0; i < vaults.length; i++) {
            totalShares += rewards.curatorShares(epoch, VaultTokenized(vaults[i]).owner());
        }
        assertGt(totalShares, 10000); // > 100 % allocated

        // --- STEP 9: attacker & others claim their rewards ---------
        console2.log("Claiming rewards");
        _claimRewards(epoch);
    }

    // Claim helper – each stakeholder pulls what the Rewards contract thinks
    // they earned (spoiler: the attacker earns too much)
    function _claimRewards(uint48 epoch) internal {
        address[] memory operators = middleware.getAllOperators();
        // claim as operators --------------------------------------------------
        for (uint i = 0; i < operators.length; i++) {
            address op = operators[i];
            vm.startPrank(op);
            if (rewards.operatorShares(epoch, op) > 0) {
                rewards.claimOperatorFee(address(rewardsToken), op);
            }
            vm.stopPrank();
        }
        // claim as vaults / stakers ------------------------------------------
        address[] memory vaults = vaultManager.getVaults(epoch);
        for (uint i = 0; i < vaults.length; i++) {
            vm.startPrank(staker);
            rewards.claimRewards(address(rewardsToken), vaults[i]);
            vm.stopPrank();

            vm.startPrank(VaultTokenized(vaults[i]).owner());
            rewards.claimCuratorFee(address(rewardsToken), VaultTokenized(vaults[i]).owner());
            vm.stopPrank();
        }
        // protocol fee --------------------------------------------------------
        vm.startPrank(owner);
        rewards.claimProtocolFee(address(rewardsToken), owner);
        vm.stopPrank();
    }

    // Deploy rewards contracts, mint tokens, assign roles, fund epochs -------
    function _setupRewards() internal {
        uptimeTracker = new MockUptimeTracker();
        rewards       = new Rewards();

        // initialise with fee splits & uptime threshold
        rewards.initialize(
            owner,                          // admin
            owner,                          // protocol fee recipient
            payable(address(middleware)),   // middleware (oracle)
            address(uptimeTracker),         // uptime oracle
            1000,                           // protocol   10%
            2000,                           // operators  20%
            1000,                           // curators   10%
            11_520                          // min uptime (seconds)
        );

        // set up roles --------------------------------------------------------
        vm.prank(owner);
        rewards.setRewardsManagerRole(REWARDS_MANAGER_ROLE);

        vm.prank(REWARDS_MANAGER_ROLE);
        rewards.setRewardsDistributorRole(REWARDS_DISTRIBUTOR_ROLE);

        // create & fund mock reward token ------------------------------------
        rewardsToken = new ERC20Mock();
        rewardsToken.mint(REWARDS_DISTRIBUTOR_ROLE, 1_000_000 * 1e18);
        vm.prank(REWARDS_DISTRIBUTOR_ROLE);
        rewardsToken.approve(address(rewards), 1_000_000 * 1e18);

        // schedule 10 epochs of 100 000 tokens each ---------------------------
        vm.startPrank(REWARDS_DISTRIBUTOR_ROLE);
        rewards.setRewardsAmountForEpochs(1, 10, address(rewardsToken), 100_000 * 1e18);

        // 100 % of rewards go to the primary asset-class (id 1) ---------------
        vm.startPrank(REWARDS_MANAGER_ROLE);
        rewards.setRewardsShareForAssetClass(1, 10000); // 10 000 bp == 100 %
        vm.stopPrank();
    }
}
```
**Output:**
```bash
Ran 1 test for test/middleware/PoCMissingLockingRewards.t.sol:PoCMissingLockingRewards
[PASS] test_PoCRewardsManipulated() (gas: 8408423)
Logs:
  Warping to a fresh epoch
  Creating 4 nodes for Alice with the same stake
  Creating 1 node for Charlie with the full stake
  Removing Alice's unbacked nodes at the earliest possible moment
  Advancing and caching stakes
  Marking everyone as fully up for the epoch
  Advancing 3 epochs so rewards can be distributed
  Distributing rewards
  Verifying that the share accounting exceeds 100 %
  Claiming rewards

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 5.83ms (2.29ms CPU time)

Ran 1 test suite in 133.94ms (5.83ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

**Recommended Mitigation:** * **노드 생성 즉시 지분 잠금**

  ```solidity
  // inside addNode(), after newStake is finalised
  operatorLockedStake[operator] += newStake;
  ```

  `_initializeEndValidationAndFlag()`에서, 그리고 `_initializeValidatorStakeUpdate()`가 노드의 지분을 낮출 때마다 잠금을 해제(차감)하십시오.

**Suzaku:**
커밋 [d3f80d9](https://github.com/suzaku-network/suzaku-core/pull/155/commits/d3f80d9d3830deda5012eca6f4356b02ad768868)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### DoS on stake accounting functions by bloating `operatorNodesArray` with irremovable nodes

**Description:** 운영자가 노드를 제거할 때 의도된 흐름은 다음과 같습니다:

1. `removeNode()` (미들웨어)
2. `calcAndCacheNodeStakeForAllOperators()` (즉시 호출되거나 `updateGlobalNodeStakeOncePerEpoch` 수정자에 의해 호출됨)
   * 분기 ✔ 실행:
     ```solidity
     if (nodePendingRemoval[valID] && …) {
         _removeNodeFromArray(operator,nodeId);
         nodePendingRemoval[valID] = false;
     }
     ```
   * 노드가 `operatorNodesArray`에서 제거(pop)되고 `nodePendingRemoval` 플래그가 지워집니다.
3. P-Chain 확인이 도착한 후(워프 메시지) `completeValidatorRemoval()`

만약 1단계와 3단계가 해당 에포크의 `calcAndCacheNodeStakeForAllOperators()` 호출 *후*, **동일한 에포크 내**에서 발생하면 일관성 없는 상태에 진입합니다:

* `completeValidatorRemoval()`은 **BalancerValidatorManager**에서 `_completeEndValidation()`을 실행하여 `._registeredValidators`의 매핑 항목을 삭제합니다:
  ```solidity
  delete $._registeredValidators[validator.nodeID];
  ```
* 이제부터 다음 에포크 호출 시:
  ```solidity
  bytes32 valID =
      balancerValidatorManager.registeredValidators(abi.encodePacked(uint160(uint256(nodeId))));
  ```
  `bytes32(0)`을 반환합니다.

`_calcAndCacheNodeStakeForOperatorAtEpoch()` (에포크 E + 1) 동안:

* `valID == bytes32(0)`이므로 `nodePendingRemoval[valID]`와 `nodePendingUpdate[valID]`
  **모두** `false`입니다.
* 따라서 특별 제거 분기를 건너뛰며, `operatorNodesArray`는 **오래된 `nodeId`를 영원히 포함**하게 됩니다.
* 센티넬(sentinel) `valID`를 더 이상 재구성할 수 없으므로 다른 관리 단계에서도 이를 제거하지 못합니다.

이제 노드를 제거하거나 업데이트하는 것은 불가능하며, 운영자는 시퀀스를 반복하여 *무제한*의 유령 노드를 추가하고 `operatorNodesArray`를 부풀릴 수 있습니다. 해당 배열에 대한 모든 O(n) 루프(예: `forceUpdateNodes`, `_calcAndCacheNodeStakeForAllOperators`, 많은 뷰 도우미)는 제한 없이 증가하여 결국 블록 가스를 소진하거나 해당 운영자 및 간접적으로 프로토콜 전체 유지 관리 기능에 대해 영구적인 **DoS**를 유발합니다.


**Impact:** 크기가 커진 배열로 인해 에포크 유지 관리 및 지분 재조정 기능이 가스 부족으로 되돌아가게(revert) 됩니다. 이에 의존하는 지분 업데이트, 슬래싱, 보상 분배 및 비상 인출이 동결될 수 있습니다.

**Proof of Concept:**
```solidity
// ─────────────────────────────────────────────────────────────────────────────
// PoC – “Phantom” / Irremovable Node
// Shows how a node can be removed *logically* on the P-Chain yet remain stuck
// inside `operatorNodesArray`, blowing up storage & breaking future logic.
// ─────────────────────────────────────────────────────────────────────────────
import {AvalancheL1MiddlewareTest} from "./AvalancheL1MiddlewareTest.t.sol";
import {PChainOwner}              from "@avalabs/teleporter/validator-manager/interfaces/IValidatorManager.sol";
import {StakeConversion}          from "src/contracts/middleware/libraries/StakeConversion.sol";
import {console2}                 from "forge-std/console2.sol";

contract PoCIrremovableNode is AvalancheL1MiddlewareTest {

    /// Demonstrates *expected* vs *buggy* behaviour side-by-side
    function test_PoCIrremovableNode() public {

        // ─────────────────────────────────────────────────────────────────────
        // 1)  NORMAL FLOW – node can be removed
        // ─────────────────────────────────────────────────────────────────────
        console2.log("=== NORMAL FLOW ===");
        vm.startPrank(alice);

        // Create a fresh nodeId so it is unique for Alice
        bytes32 nodeId = keccak256(abi.encodePacked(alice, "node-A", block.timestamp));

        console2.log("Registering nodeA");
        middleware.addNode(
            nodeId,
            hex"ABABABAB",                        // dummy BLS key
            uint64(block.timestamp + 2 days),     // expiry
            PChainOwner({threshold: 1, addresses: new address[](0)}),
            PChainOwner({threshold: 1, addresses: new address[](0)}),
            100_000_000_000_000                   // stake
        );

        // Complete registration on the mock validator manager
        uint32 regMsgIdx = mockValidatorManager.nextMessageIndex() - 1;
        middleware.completeValidatorRegistration(alice, nodeId, regMsgIdx);
        console2.log("nodeA registered");

        // Length should now be 1
        assertEq(middleware.getOperatorNodesLength(alice), 1);

        // Initiate removal
        console2.log("Removing nodeA");
        middleware.removeNode(nodeId);

        vm.stopPrank();

        // Advance 1 epoch so stake caches roll over
        _calcAndWarpOneEpoch();

        // Confirm removal from P-Chain and complete it on L1
        uint32 rmMsgIdx = mockValidatorManager.nextMessageIndex() - 1;
        vm.prank(alice);
        middleware.completeValidatorRemoval(rmMsgIdx);
        console2.log("nodeA removal completed");

        // Now node array should be empty
        assertEq(middleware.getOperatorNodesLength(alice), 0);
        console2.log("NORMAL FLOW success: array length = 0\n");

        // ─────────────────────────────────────────────────────────────────────
        // 2)  BUGGY FLOW – removal inside same epoch ⇒ phantom entry
        // ─────────────────────────────────────────────────────────────────────
        console2.log("=== BUGGY FLOW (same epoch) ===");
        vm.startPrank(alice);

        // Re-use *same* nodeId to simulate quick re-registration
        console2.log("Registering nodeA in the SAME epoch");
        middleware.addNode(
            nodeId,                               // same id!
            hex"ABABABAB",
            uint64(block.timestamp + 2 days),
            PChainOwner({threshold: 1, addresses: new address[](0)}),
            PChainOwner({threshold: 1, addresses: new address[](0)}),
            100_000_000_000_000
        );
        uint32 regMsgIdx2 = mockValidatorManager.nextMessageIndex() - 1;
        middleware.completeValidatorRegistration(alice, nodeId, regMsgIdx2);
        console2.log("nodeA (second time) registered");

        // Expect length == 1 again
        assertEq(middleware.getOperatorNodesLength(alice), 1);

        // Remove immediately
        console2.log("Immediately removing nodeA again");
        middleware.removeNode(nodeId);

        // Complete removal *still inside the same epoch* (simulating fast warp msg)
        uint32 rmMsgIdx2 = mockValidatorManager.nextMessageIndex() - 1;
        middleware.completeValidatorRemoval(rmMsgIdx2);
        console2.log("nodeA (second time) removal completed");

        vm.stopPrank();
```solidity
        // Advance to next epoch
        _calcAndWarpOneEpoch();

        // BUG: array length is STILL 1 → phantom node stuck forever
        uint256 lenAfter = middleware.getOperatorNodesLength(alice);
        assertEq(lenAfter, 1, "Phantom node should remain");

        console2.log("BUGGY FLOW reproduced: node is irremovable.");
    }
}
```

**Output**
```bash
Ran 1 test for test/middleware/PoCIrremovableNode.t.sol:PoCIrremovableNode
[PASS] test_PoCIrremovableNode() (gas: 1138657)
Logs:
  === NORMAL FLOW ===
  Registering nodeA
  nodeA registered
  Removing nodeA
  nodeA removal completed
  NORMAL FLOW success: array length = 0

  === BUGGY FLOW (same epoch) ===
  Registering nodeA in the SAME epoch
  nodeA (second time) registered
  Immediately removing nodeA again
  nodeA (second time) removal completed
  BUGGY FLOW reproduced: node is irremovable.

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 4.31ms (789.96µs CPU time)

Ran 1 test suite in 158.79ms (4.31ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
PoC를 실행하기 전에 `MockBalancerValidatorManager::completeEndValidation`이 `BalancerValidatorManager` 구현에 존재하므로 다음 줄을 추가해야 합니다:
```diff
    function completeEndValidation(
        uint32 messageIndex
    ) external override {
        ...
        ...
        // Clean up
        delete pendingRegistrationMessages[messageIndex];
        delete pendingTermination[validationID];
+       delete _registeredValidators[validator.nodeID];
    }
```
- [completeValidatorRemoval](https://github.com/ava-labs/icm-contracts/blob/bd61626c67a7736119c6571776b85db6ce105992/contracts/validator-manager/ValidatorManager.sol#L604-L605)



**Recommended Mitigation:** **제거 대기 항목을 `validationID`가 아닌 `nodeId`로 추적**하거나, `_registeredValidators`가 지워진 후에도 미들웨어가 상관관계를 맺을 수 있도록 삭제 전에 보조 매핑 `nodeId ⇒ validationID`를 저장하십시오.

**Suzaku:**
커밋 [d4d2df7](https://github.com/suzaku-network/suzaku-core/pull/155/commits/d4d2df784273bc8d4de51e8aabc6aaf06cea6203)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


\clearpage
## Low Risk


### Missing zero-address validation for burner address during initialization can break slashing

**Description:** VaultTokenized 계약의 초기화 절차는 버너(burner) 매개변수가 0이 아닌 주소인지 검증하지 못합니다. 그러나 `onSlash` 함수는 SafeERC20의 `safeTransfer`를 사용하여 이 주소로 토큰을 전송하는데, 수신자가 address(0)인 경우 대부분의 ERC20 구현에서 되돌아갑니다(revert).

```solidity
// In _initialize:
vs.burner = params.burner; // No validation that params.burner != address(0)

// In onSlash:
if (slashedAmount > 0) {
    IERC20(vs.collateral).safeTransfer(vs.burner, slashedAmount); // Will revert if vs.burner is address(0)
}
```
담보와 같은 다른 중요한 매개변수는 0 주소에 대해 검증되지만, 버너 매개변수는 슬래싱 흐름에서의 중요성에도 불구하고 이 확인이 부족합니다.

**Impact:** 버너를 `address(0)`으로 설정하면 핵심 보안 기능(슬래싱)이 중단됩니다.

**Recommended Mitigation:** 초기화 중에 버너 매개변수에 대한 0 주소 유효성 검사를 추가하는 것을 고려하십시오.

**Suzaku:**
커밋 [4683ab8](https://github.com/suzaku-network/suzaku-core/pull/155/commits/4683ab82c40103fd6af5a7d2447e5be211e93f20)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### `BaseDelegator` is not using upgradeable version of `ERC165`

**Description:** `BaseDelegator` 계약은 현재 업그레이드할 수 없는 버전의 ERC165를 상속합니다. 이는 향후 ERC165 인터페이스의 업그레이드 또는 수정에 적응하는 계약의 능력을 제한하여 계약의 업그레이드 가능성 및 다른 업그레이드 가능한 계약과의 호환성에 잠재적으로 영향을 미칠 수 있습니다. 업그레이드할 수 없는 버전에는 업그레이드 가능한 계약을 위해 예약된 필요한 초기화 프로그램 및 저장소 간격(storage gap)이 없으므로 문제가 발생할 수 있습니다.

**Impact:** 업그레이드 가능한 계약(`BaseDelegator`)에서 업그레이드할 수 없는 `ERC165`를 사용하면 향후 업그레이드와의 비호환성 위험이 발생합니다.

**Recommended Mitigation:** `BaseDelagator`를 다음과 같이 변경하십시오:
```diff
- abstract contract BaseDelegator is AccessControlUpgradeable, ReentrancyGuardUpgradeable, IBaseDelegator, ERC165 {
+ abstract contract BaseDelegator is AccessControlUpgradeable, ReentrancyGuardUpgradeable, IBaseDelegator, ERC165Upgradeable {
    using ERC165Checker for address;
```

**Suzaku:**
인지됨(Acknowledged).

**Cyfrin:** 인지됨(Acknowledged).



### Vault limit cannot be modified if vault Is already enabled

**Description**

`updateVaultMaxL1Limit` 함수는 볼트가 이미 활성화된 경우 볼트 한도를 늘리거나 줄이려고 시도할 때 `MapWithTimeData__AlreadyEnabled`와 함께 되돌아갑니다(revert). 이 동작은 함수 내에서 `vaults.enable(vault)`를 호출하기 때문에 발생하는데, 이는 볼트가 이미 활성화된 상태일 때 실패합니다.

```solidity
function updateVaultMaxL1Limit(address vault, uint96 assetClassId, uint256 vaultMaxL1Limit) external onlyOwner {
    if (!vaults.contains(vault)) {
        revert AvalancheL1Middleware__NotVault(vault);
    }
    if (vaultToAssetClass[vault] != assetClassId) {
        revert AvalancheL1Middleware__WrongVaultAssetClass();
    }

    _setVaultMaxL1Limit(vault, assetClassId, vaultMaxL1Limit);

    if (vaultMaxL1Limit == 0) {
        vaults.disable(vault);
    } else {
        vaults.enable(vault);
    }
}
```

**Impact**

이미 활성화된 볼트에 대해 0이 아닌 `vaultMaxL1Limit`로 `updateVaultMaxL1Limit`를 호출하면 되돌아가서 예상되는 동작을 깨뜨립니다. 현재 설계에서는 0이 아닌 새 한도를 설정하고 다시 활성화하기 전에 볼트를 명시적으로 비활성화해야 합니다. 이 워크플로는 직관적이지 않으며 계약 소유자나 관리자에게 불필요한 마찰을 초래합니다.

**Proof of Concept**

설명된 동작으로 인해 다음 테스트가 실패합니다. `AvalancheL1MiddlewareTest`에 추가하십시오:

```solidity
function testUpdateVaultMaxL1Limit() public {
    vm.startPrank(validatorManagerAddress);

    // Attempt to update to a new non-zero limit while vault is already enabled
    vaultManager.updateVaultMaxL1Limit(address(vault), 1, 500 ether);

    vm.stopPrank();
}
```

**Recommended Mitigation**

활성화 또는 비활성화를 시도하기 전에 현재 활성화된 상태를 확인하도록 `updateVaultMaxL1Limit` 내부의 로직을 수정하는 것을 고려하십시오. 실제 상태 전환이 있는 경우에만 `vaults.enable(vault)` 또는 `vaults.disable(vault)`를 호출하십시오.

**Suzaku:**
커밋 [a9f6aaa](https://github.com/suzaku-network/suzaku-core/pull/155/commits/a9f6aaa92bd3800335c4d8085225a23a38c58b34)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.



### Incorrect vault status determination in `MiddlewareVaultManager`

**Description:** `MiddlewareVaultManager::_wasActiveAt()`은 볼트가 특정 타임스탬프에서 활성 상태였는지 여부를 결정합니다. 이 함수는 `getVaults()` 메서드에서 주어진 에포크에 대해 활성 볼트를 필터링하는 데 사용됩니다.

`_wasActiveAt()`의 현재 구현은 볼트가 비활성화된 정확한 타임스탬프에서 볼트가 활성 상태라고 잘못 간주합니다. 이 함수는 다음과 같은 경우 true를 반환합니다:

- 볼트가 활성화됨 (enabledTime != 0)
- 볼트가 타임스탬프 또는 그 이전에 활성화됨 (enabledTime <= timestamp)
- AND:
       - 볼트가 비활성화된 적이 없음 (disabledTime == 0) OR
       - 볼트의 비활성화된 타임스탬프가 쿼리 타임스탬프보다 크거나 같음 (disabledTime >= timestamp)


```solidity
// Current implementation
function _wasActiveAt(uint48 enabledTime, uint48 disabledTime, uint48 timestamp) private pure returns (bool) {
    return enabledTime != 0 && enabledTime <= timestamp && (disabledTime == 0 || disabledTime >= timestamp);
}
```

문제는 세 번째 조건(`disabledTime >= timestamp`)에 있습니다. 이 로직은 쿼리되는 타임스탬프(예: 에포크 시작)에 정확히 비활성화된 볼트가 여전히 해당 에포크에 대해 활성 상태로 간주됨을 의미하는데, 이는 직관적이지 않습니다. 일반적으로 엔티티가 특정 타임스탬프에 비활성화되면 해당 타임스탬프부터 비활성 상태로 간주되어야 합니다.


**Impact:** 에포크 경계에서 정확히 비활성화된 볼트가 해당 에포크에서 활성 상태로 잘못 포함됩니다.

**Recommended Mitigation:** 비활성화 확인에 대해 엄격한 부등식(strict inequality)을 사용하도록 `_wasActiveAt()` 함수를 수정하는 것을 고려하십시오.

**Suzaku:**
커밋 [9bbbcfc](https://github.com/suzaku-network/suzaku-core/pull/155/commits/9bbbcfce7bedd1dd4e60fdf55bb5f13ba8ab4847)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Missing validation for zero `epochDuration` in AvalancheL1Middleware can break epoch based accounting

**Description:** `AvalancheL1Middleware::EPOCH_DURATION`은 생성자에서 설정되는 불변 매개변수입니다. 현재 구현은 `slashingWindow`가 `epochDuration`보다 작지 않은지만 확인하고 `epochDuration` 자체가 0보다 큰지는 확인하지 않습니다.

```solidity
constructor(
    AvalancheL1MiddlewareSettings memory settings,
    address owner,
    address primaryAsset,
    uint256 primaryAssetMaxStake,
    uint256 primaryAssetMinStake,
    uint256 primaryAssetWeightScaleFactor
) AssetClassRegistry(owner) {
    // Other validations...

    if (settings.slashingWindow < settings.epochDuration) {
        revert AvalancheL1Middleware__SlashingWindowTooShort(settings.slashingWindow, settings.epochDuration);
    }

    //@audit No check for zero epochDuration!

    START_TIME = Time.timestamp();
    EPOCH_DURATION = settings.epochDuration;
    // Other assignments...
}
```

`settings.slashingWindow < settings.epochDuration` 확인은 slashingWindow도 0인 한 통과합니다.

**Impact:** `getEpochAtTs`와 같은 계약 함수는 EPOCH_DURATION으로 나누기에 의존하므로 0으로 나누기 오류가 발생할 수 있습니다.


**Recommended Mitigation:** 생성자에 `epochDuration` 매개변수에 대한 명시적인 유효성 검사 확인을 추가하는 것을 고려하십시오.

**Suzaku:**
인지됨(Acknowledged).

**Cyfrin:** 인지됨(Acknowledged).


### Insufficient update window validation can cause denial of service in `forceUpdateNodes`

**Description:** `AvalancheL1Middleware` 생성자는 `UPDATE_WINDOW` 매개변수가 `EPOCH_DURATION`보다 작은지 확인하지 못합니다. 지분 관리 기능에 필수적인 `onlyDuringFinalWindowOfEpoch` 수정자는 `UPDATE_WINDOW`가 `EPOCH_DURATION`보다 크거나 같으면 영구적으로 되돌아가기(revert) 때문에 이 유효성 검사는 매우 중요합니다.

`onlyDuringFinalWindowOfEpoch` 수정자는 에포크 종료 시 특정 시간 창 동안에만 함수를 호출할 수 있도록 강제하여 작동합니다:

```solidity
modifier onlyDuringFinalWindowOfEpoch() {
    uint48 currentEpoch = getCurrentEpoch();
    uint48 epochStartTs = getEpochStartTs(currentEpoch);
    uint48 timeNow = Time.timestamp();
    uint48 epochUpdatePeriod = epochStartTs + UPDATE_WINDOW;

    if (timeNow < epochUpdatePeriod || timeNow > epochStartTs + EPOCH_DURATION) { //@audit always reverts if UPDATE_WINDOW >= EPOCH_DURATION
        revert AvalancheL1Middleware__NotEpochUpdatePeriod(timeNow, epochUpdatePeriod);
    }
    _;
}
```

수정자는 다음과 같은 경우에만 유효한 실행 창을 생성합니다:

- `timeNow >= epochStartTs + UPDATE_WINDOW` (업데이트 창이 시작된 후)
- `timeNow <= epochStartTs + EPOCH_DURATION` (에포크가 끝나기 전)

이 창이 존재하려면 `UPDATE_WINDOW`는 `EPOCH_DURATION`보다 작아야 합니다.

생성자는 현재 `slashingWindow`가 `epochDuration`보다 작지 않은지만 확인하고 `UPDATE_WINDOW`에 대한 확인은 없습니다:

```solidity
constructor(
    AvalancheL1MiddlewareSettings memory settings,
    // other parameters...
) AssetClassRegistry(owner) {
    // other checks...

    if (settings.slashingWindow < settings.epochDuration) {
        revert AvalancheL1Middleware__SlashingWindowTooShort(settings.slashingWindow, settings.epochDuration);
    }

    // @audit No validation for UPDATE_WINDOW relation to EPOCH_DURATION

    // Initializations...
    EPOCH_DURATION = settings.epochDuration;
    UPDATE_WINDOW = settings.stakeUpdateWindow;
    // other initializations...
}
```

`EPOCH_DURATION`과 `UPDATE_WINDOW`는 모두 불변 변수로 설정되므로 배포 후에는 이 문제를 수정할 수 없습니다.

**Impact:** `forceUpdateNodes()` 함수는 `onlyDuringFinalWindowOfEpoch` 수정자에 의해 보호되므로 영구적으로 사용할 수 없게 됩니다.

**Recommended Mitigation:** 생성자에 `UPDATE_WINDOW > 0 && UPDATE_WINDOW < EPOCH_DURATION`인지 확인하는 명시적인 유효성 검사를 추가하는 것을 고려하십시오. 또한 구성 오류를 방지하는 데 도움이 되도록 이러한 시간 매개변수 간의 관계를 명확하게 설명하는 주석을 추가하는 것을 고려하십시오:

```solidity
/**
 * @notice Required relationship between time parameters:
 * 0 < UPDATE_WINDOW < EPOCH_DURATION <= SLASHING_WINDOW
 */
```

**Suzaku:**
커밋 [4f9d52a](https://github.com/suzaku-network/suzaku-core/pull/155/commits/4f9d52ac520312cfc0877a355f3d064229725fa2)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Disabled operators can register new validator nodes

**Description:** `AvalancheL1Middleware::addNode` 함수는 msg.sender가 운영자 목록에 포함되어 있으면 운영자가 새 노드를 등록할 수 있도록 합니다. 그러나 운영자 수명 주기를 처리하는 방식에 잠재적인 문제가 있습니다: removeOperator를 통해 운영자가 영구적으로 제거되기 전에 먼저 disableOperator를 사용하여 "비활성화" 상태로 전환해야 합니다.

문제는 addNode 함수가 운영자가 비활성화 상태인지 확인하지 않고 운영자 세트에 존재하는지만 확인하기 때문에 발생합니다. 결과적으로 비활성화된 운영자는 이 기간 동안 비활성 상태일 것으로 예상됨에도 불구하고 여전히 addNode를 호출할 수 있습니다.

**Impact:** 비활성화된 운영자는 그러한 작업을 수행하지 못하도록 하는 상태임에도 불구하고 addNode를 통해 새 검증자 노드를 계속 등록할 수 있습니다.

**Recommended Mitigation:** 등록 여부뿐만 아니라 운영자가 활성화되었는지도 확인하도록 addNode 함수를 업데이트하십시오:

```diff
(, uint48 disabledTime) = operators.getTimes(operator);
+if (!operators.contains(operator) || disabledTime > 0 ) {
-if (!operators.contains(operator)) {
    revert AvalancheL1Middleware__OperatorNotActive(operator);
}
```
**Suzaku:**
커밋 [0e0d4ae](https://github.com/suzaku-network/suzaku-core/pull/155/commits/0e0d4aee6394b8acbda391e107db8fb9f49f7102)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Hardcoded gas limit for hook in `onSlash` may cause reverts

**Description:** `BaseDelegator` 계약의 `onSlash` 함수는 후크 주소가 설정된 경우 낮은 수준의 `call`을 통해 후크를 조건부로 호출합니다:

```solidity
assembly ("memory-safe") {
    pop(call(HOOK_GAS_LIMIT, hook_, 0, add(calldata_, 0x20), mload(calldata_), 0, 0))
}
```

이 호출은 **하드코딩된 가스 제한(`HOOK_GAS_LIMIT`)**을 사용하며, 함수는 진행하기 전에 최소 `HOOK_RESERVE + HOOK_GAS_LIMIT * 64 / 63` 가스가 사용 가능한지 강제합니다. 이 요구 사항이 충족되지 않으면 함수는 `BaseDelegator__InsufficientHookGas`로 되돌아갑니다.

이 엄격한 가스 강제는 취약성을 도입합니다: 후크 실행에 `HOOK_GAS_LIMIT`에 할당된 것보다 더 많은 가스가 필요한 경우 `call`은 조용히 실패하거나 트랜잭션이 완전히 되돌아갈 수 있습니다.

**Impact:** 하드코딩된 제한보다 더 많은 가스가 필요한 후크는 지속적으로 실패하여 잠재적으로 통합을 깨뜨릴 수 있습니다.

프로토콜 복잡성이 증가함에 따라 하드코딩된 가스 제한은 불안정해지고 구성 가능성이나 향후 확장을 방해할 수 있습니다.

**Recommendation:**

계약 소유자가 후크 가스 제한을 구성할 수 있는 메커니즘 도입을 고려하십시오.

**Suzaku:**
인지됨(Acknowledged).

**Cyfrin:** 인지됨(Acknowledged).


### `NodeId` truncation can potentially cause validator registration denial of service

**Description:** `AvalancheL1Middleware::addNode` 함수는 검증자 등록 상태를 확인할 때 32바이트 `nodeId`를 20바이트로 자릅니다(truncate). 이 잘라내기는 `BalancerValidatorManager`와 상호 작용할 때 발생합니다.

```solidity
// AvalancheL1Middleware.sol
bytes32 valId = balancerValidatorManager.registeredValidators(
    abi.encodePacked(uint160(uint256(nodeId)))  // Truncates 32 bytes to 20 bytes
);
```

`uint160(uint256(nodeId))`는 `BalancerValidatorManager::registeredValidators()`에 전달하기 전에 `nodeId`의 처음 12바이트를 삭제합니다. 그러나 `ValidatorManager.registeredValidators()` 함수는 잘라내기 없이 전체 바이트 nodeId와 작동하도록 설계되었습니다.

**Impact:** 충돌하는 잘린 nodeId를 가진 다른 검증자가 이미 존재하는 경우 운영자는 검증자를 등록할 수 없습니다.

**Proof of Concept:** `AvalancheL1MiddlewareTest.t.sol`에서 다음 테스트를 실행하십시오

```solidity
function test_NodeIdCollisionVulnerability() public {
    uint48 epoch = _calcAndWarpOneEpoch();

    // Create two different nodeIds that have the same first 20 bytes
    bytes32 nodeId1 = 0x0000000000000000000000001234567890abcdef1234567890abcdef12345678;
    bytes32 nodeId2 = 0xFFFFFFFFFFFFFFFFFFFFFFFF1234567890abcdef1234567890abcdef12345678;

    // share same first 20 bytes when truncated
    bytes memory truncated1 = abi.encodePacked(uint160(uint256(nodeId1)));
    bytes memory truncated2 = abi.encodePacked(uint160(uint256(nodeId2)));
    assertEq(keccak256(truncated1), keccak256(truncated2), "Truncated nodeIds should be identical");


    // Alice adds the first node
    vm.prank(alice);
    middleware.addNode(
                nodeId1,
                hex"ABABABAB", // dummy BLS
                uint64(block.timestamp + 2 days),
                PChainOwner({threshold: 1, addresses: new address[](1)}),
                PChainOwner({threshold: 1, addresses: new address[](1)}),
                100_000_000_001_000
            );

    // Verify first node was registered
    bytes32 validationId1 = mockValidatorManager.registeredValidators(truncated1);
    assertNotEq(validationId1, bytes32(0), "First node should be registered");

    // Alice tries to add the second node with different nodeId but same truncated bytes
    // This should fail due to collision
    vm.prank(alice);
    vm.expectRevert();
    middleware.addNode(
                nodeId2,
                hex"ABABABAB", // dummy BLS
                uint64(block.timestamp + 2 days),
                PChainOwner({threshold: 1, addresses: new address[](1)}),
                PChainOwner({threshold: 1, addresses: new address[](1)}),
                100_000_000_001_000
            );
}

```

**Recommended Mitigation:** 20바이트로 강제 잘라내기를 제거하는 것을 고려하십시오.


**Suzaku:**
인지됨(Acknowledged).

**Cyfrin:** 인지됨(Acknowledged).


### Unbounded weight scale factor causes precision loss in stake conversion, potentially leading to loss of operator funds

**Description:** `AvalancheL1Middleware`는 256비트 지분 금액과 P-Chain용 64비트 검증자 가중치 간의 변환을 위해 `WEIGHT_SCALE_FACTOR`를 사용합니다. 그러나 이 스케일 팩터에는 경계가 없으며, 부적절하게 높은 값은 정밀도 손실을 유발할 수 있습니다.

변환 프로세스는 다음과 같이 작동합니다:

`stakeToWeight(): weight = stakeAmount / scaleFactor`
`weightToStake(): recoveredStake = weight * scaleFactor`

`WEIGHT_SCALE_FACTOR`가 지분 금액에 비해 너무 높으면 `stakeToWeight()`에서의 나눗셈이 0으로 잘려서 지분을 사실상 사용할 수 없게 만듭니다.


**Impact:** 정밀도 손실로 인해 검증자에 기록된 가중치가 0이 될 수 있습니다.

**Recommended Mitigation:** 생성자에서 합리적인 최대 경계를 구현하는 것을 고려하십시오.

**Suzaku:**
커밋 [b38dfed](https://github.com/suzaku-network/suzaku-core/pull/155/commits/b38dfed1d21a628582b11d217f5112290b973034)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### `remainingBalanceOwner` should be set by the protocol owner

**Description:** `addNode` 함수 호출 중 전달되는 `remainingBalanceOwner` 매개변수는 검증자가 검증자 세트에서 제거될 때 검증자 잔액에서 남은 `$AVAX`를 받을 P-Chain 소유자 주소를 나타내도록 의도되었습니다. 현재 이 값은 `addNode`를 호출하는 운영자가 제공합니다.

그러나 프로토콜 보안 및 정확성 관점에서 `remainingBalanceOwner`의 할당은 운영자에게 맡겨져서는 안 됩니다. 대신 남은 자금을 잘못된 당사자가 받는 것을 방지하기 위해 프로토콜 자체에서 결정하고 구성해야 합니다.

함수 서명의 예시 스니펫:

```solidity
function addNode(
        bytes32 nodeId,
        bytes calldata blsKey,
        uint64 registrationExpiry,
        PChainOwner calldata remainingBalanceOwner, // should be passed by the protocol
        PChainOwner calldata disableOwner,
        uint256 stakeAmount // optional
    ) external updateStakeCache(getCurrentEpoch(), PRIMARY_ASSET_CLASS) updateGlobalNodeStakeOncePerEpoch {
```

**Impact:** 검증자 제거 시 남은 $AVAX 자금의 잘못된 라우팅으로 인해 스테이커의 자금 손실이 발생할 수 있습니다.

**Recommended Mitigation:** `addNode` 작업 중 프로토콜이 내부적으로 `remainingBalanceOwner`를 할당하도록 수정하여 운영자 입력에서 이 매개변수를 제거하십시오.

```diff
function addNode(
        bytes32 nodeId,
        bytes calldata blsKey,
        uint64 registrationExpiry,
-        PChainOwner calldata remainingBalanceOwner,
        PChainOwner calldata disableOwner,
        uint256 stakeAmount // optional
    ) external updateStakeCache(getCurrentEpoch(), PRIMARY_ASSET_CLASS) updateGlobalNodeStakeOncePerEpoch {

}

```

**Suzaku:**
인지됨(Acknowledged).

**Cyfrin:** 인지됨(Acknowledged).


### `forceUpdateNodes` potentially enables mass validator removal when new asset classes are added

**Description:** `_requireMinSecondaryAssetClasses`가 false인 경우, 즉 운영자가 보조 자산의 최소 가치를 가지고 있지 않은 경우, 기본 자산 지분이 최소 지분 이상인지 여부와 관계없이 검증자가 강제로 제거됩니다.

```solidity
function forceUpdateNodes(
    address operator,
    uint256 limitStake
) external updateStakeCache(getCurrentEpoch(), PRIMARY_ASSET_CLASS) onlyDuringFinalWindowOfEpoch updateGlobalNodeStakeOncePerEpoch {
    // ... validation and setup code ...

    // ... stake calculation logic ...

        uint256 newStake = previousStake - stakeToRemove;
        leftoverStake -= stakeToRemove;

        if (
            (newStake < assetClasses[PRIMARY_ASSET_CLASS].minValidatorStake)
            || !_requireMinSecondaryAssetClasses(0, operator)  // @audit || operator used here
        ) {
            newStake = 0;
            _initializeEndValidationAndFlag(operator, valID, nodeId);  // Node removed
        } else {
            _initializeValidatorStakeUpdate(operator, valID, newStake);
            emit NodeStakeUpdated(operator, nodeId, newStake, valID);
        }
    }
}
```

소유자가 `activateSecondaryAssetClass()`를 통해 새 보조 자산 클래스를 활성화하면, 기존 운영자가 해당 자산 클래스에 대한 지분이 0일 가능성이 높으므로 `_requireMinSecondaryAssetClasses`가 false를 반환합니다.

다음 시나리오를 고려하십시오:

- 모든 운영자가 새 자산 클래스에 대한 최소 지분 요구 사항을 충족하는지 확인하지 않고 소유자가 `activateSecondaryAssetClass(newAssetClassId)`를 호출합니다.
- 공격자가 다음 즉시 업데이트 창 동안 모든 운영자에 대해 `forceUpdateNodes(operator, 0)`를 호출합니다.
- 재조정 시나리오를 가정하면, `_requireMinSecondaryAssetClasses`가 새 자산 클래스에 대해 false를 반환하므로 규정을 준수하지 않는 운영자의 모든 검증자 노드가 제거됩니다.

**Impact:** 주어진 운영자에 대한 검증자의 대량 제거는 불필요한 보상 손실과 노드 재등록을 위한 비용이 많이 드는 프로세스로 이어집니다.

**Proof of Concept:** `AvalancheL!MiddlewareTest.t.sol`에 추가하십시오

```solidity

function test_massNodeRemovalAttack() public {
    // Alice already has substantial stake from setup: 200_000_000_002_000
    // Let's verify the initial state
    uint48 epoch = _calcAndWarpOneEpoch();

    uint256 aliceInitialStake = middleware.getOperatorStake(alice, epoch, assetClassId);
    console2.log("Alice's initial stake:", aliceInitialStake);
    assertGt(aliceInitialStake, 0, "Alice should have initial stake from setup");

    // Create 3 nodes with Alice's existing stake
    (bytes32[] memory nodeIds,,) = _createAndConfirmNodes(alice, 3, 0, true);
    epoch = _calcAndWarpOneEpoch();

    // Verify nodes are active
    assertEq(middleware.getOperatorNodesLength(alice), 3, "Should have 3 active nodes");
    uint256 usedStake = middleware.getOperatorUsedStakeCached(alice);
    console2.log("Used stake after creating nodes:", usedStake);


    // 1: Owner adds new secondary asset class
    ERC20Mock newAsset = new ERC20Mock();
    vm.startPrank(validatorManagerAddress);
    middleware.addAssetClass(5, 1000 ether, 10000 ether, address(newAsset));
    middleware.activateSecondaryAssetClass(5);
    vm.stopPrank();

    // 2: Reduce Alice's available stake to trigger rebalancing
    // Use the existing mintedShares from setup
    uint256 originalShares = mintedShares; // From setup
    uint256 reducedShares = originalShares / 3; // Drastically reduce

    _setOperatorL1Shares(bob, validatorManagerAddress, assetClassId, alice, reducedShares, delegator);
    epoch = _calcAndWarpOneEpoch();

    // Verify rebalancing condition: newStake < usedStake
    uint256 newStake = middleware.getOperatorStake(alice, epoch, assetClassId);
    uint256 currentUsedStake = middleware.getOperatorUsedStakeCached(alice);
    console2.log("New available stake:", newStake);
    console2.log("Current used stake:", currentUsedStake);
    assertTrue(newStake < currentUsedStake, "Should trigger rebalancing");

    // 3: Exploit during force update window
    _warpToLastHourOfCurrentEpoch();

    uint256 nodesBefore = middleware.getOperatorNodesLength(alice);

    // The vulnerability: OR logic causes removal even for nodes with adequate individual stake
    middleware.forceUpdateNodes(alice, 1); // using 1 wei as limit to trigger the issue

    epoch = _calcAndWarpOneEpoch();
    uint256 nodesAfter = middleware.getOperatorNodesLength(alice);

    console2.log("Nodes before attack:", nodesBefore);
    console2.log("Nodes after attack:", nodesAfter);

    // All nodes removed due to OR logic
    assertEq(nodesAfter, 0, "All nodes should be removed due to mass removal attack");
}

```

**Recommended Mitigation:** 기존 운영자에게 최소 지분 요구 사항이 적용되기 전에 최소 1 에포크의 지연을 추가하는 것을 고려하십시오.

`forceUpdateNodes`는 검증자의 대량 제거를 트리거할 수 있는 공개 함수이므로, 새 보조 자산을 추가하기 전에 모든 기존 운영자가 규정을 준수하는지 확인하는 것을 관리자/소유자에게만 의존하는 것은 위험합니다.

**Suzaku:**
인지됨(Acknowledged).

**Cyfrin:** 인지됨(Acknowledged).


### Overpayment vulnerability in `registerL1`

**Description:** `L1Registry` 계약의 `registerL1` 함수는 사용자가 보낸 초과 이더(Ether)를 처리하지 않습니다. `registerFee`가 0이 아닌 값으로 설정되고 사용자가 등록에 필요한 것보다 더 많은 이더를 보내면, 계약은 초과분을 환불하는 대신 전체 금액을 유지합니다. `registerFee`가 0으로 설정되고 사용자가 이더를 보내면, 임의의 발신자에 대한 환불 로직이나 인출 경로가 없기 때문에 해당 이더는 사용자가 복구할 방법 없이 계약에 갇히게 됩니다.

**Impact:** 사용자는 등록에 필요한 것보다 더 많은 이더를 보내 자금을 잃을 수 있습니다. `registerFee`가 0인 경우, 수수료 징수원만이 누적된 수수료를 인출할 수 있고 그것도 미청구 수수료로 추적된 경우에만 가능하므로, 전송된 모든 이더는 계약에 영구적으로 잠깁니다. 이는 단순한 실수나 필수 지불에 대한 오해로 인해 사용자 불만과 자금 손실로 이어질 수 있습니다.

**Recommended Mitigation:** 필요한 `registerFee` 이상으로 전송된 초과 이더를 환불하는 로직을 `registerL1` 함수에 구현하십시오. 예를 들어, 필요한 수수료를 징수원에게 이체한 후 발신자에 대한 남은 잔액을 추적하십시오. 또한 `registerFee`가 0인 경우 이더가 전송되면 트랜잭션을 되돌려 우발적인 자금 손실을 방지하십시오.

**Suzaku:**
커밋 [1c4cfe6](https://github.com/suzaku-network/suzaku-core/pull/155/commits/1c4cfe6a785287263b83f6c877678ba779abb3fb)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


\clearpage
## Informational


### Wrong revert reason In `onSlash` functionality

**Description:** `VaultTokenized`의 `onSlash` 함수에서 `captureEpoch` 매개변수를 검증하기 위해 다음 확인이 사용됩니다:
```solidity
if ((currentEpoch_ > 0 && captureEpoch < currentEpoch_ - 1) || captureEpoch > currentEpoch_) {
    revert Vault__InvalidCaptureEpoch();
}
```
`currentEpoch_`가 0이면 `currentEpoch_ - 1`이 0보다 작아지므로 `captureEpoch < currentEpoch_ - 1` 표현식에서 언더플로우가 발생합니다. 이로 인해 확인이 의도된 사용자 지정 오류 대신 일반 산술 오류로 되돌아갑니다.

**Impact:** `currentEpoch_`가 0이면 `onSlash`를 호출할 때 비교에서 언더플로우가 발생하여 의도된 `Vault__InvalidCaptureEpoch` 오류가 아닌 일반 산술 오류로 되돌아갑니다. 이는 디버깅을 더 어렵게 만들 수 있으며 호출자에게 예상치 못한 동작을 초래할 수 있습니다.

**Recommended Mitigation:** 다음을 확인하여 언더플로우를 방지하도록 조건을 업데이트하십시오:
```solidity
if ((currentEpoch_ > 0 && captureEpoch + 1 < currentEpoch_) || captureEpoch > currentEpoch_) {
    revert Vault__InvalidCaptureEpoch();
}
```
이렇게 하면 확인이 모든 `currentEpoch_` 값에 대해 안전하고 입력이 유효하지 않을 때 항상 올바른 사용자 지정 오류로 되돌아갑니다.

**Suzaku:**
커밋 [b654dfb](https://github.com/suzaku-network/suzaku-core/commit/b654dfbb31dd6e840f2f7dfcda0f55dda3ff37b2)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


\clearpage
## Gas Optimization


### Gas optimization for `getVaults` function

**Description:** `MiddlewareVaultManager::getVaults`는 볼트의 전체 목록을 두 번 반복하는 비효율적인 패턴을 사용합니다. 첫 번째 반복은 활성 볼트 수를 계산하고, 두 번째 반복은 활성 볼트 배열을 구축합니다.

이 구현은:

- 동일한 `vaults.atWithTimes(i)` 호출을 두 번 수행합니다.
- 각 볼트에 대해 동일한 `_wasActiveAt()` 계산을 두 번 수행합니다.
- 특히 볼트 수가 증가함에 따라 불필요한 가스 소비를 초래합니다.

**Recommended Mitigation:** 첫 번째 루프에서 활성 볼트를 캐싱하여 중복 반복을 제거하는 단일 패스 접근 방식을 사용하도록 함수를 리팩터링하는 것을 고려하십시오.

```diff solidity
function getVaults(
    uint48 epoch
) external view returns (address[] memory) {
    uint256 vaultCount = vaults.length();
    uint48 epochStart = middleware.getEpochStartTs(epoch);

    // Early return for empty vaults
    if (vaultCount == 0) {
        return new address[](0);
    }

++    address[] memory tempVaults = new address[](vaultCount);
    uint256 activeCount = 0;

    // Single pass through the vaults
    for (uint256 i = 0; i < vaultCount; i++) {
        (address vault, uint48 enabledTime, uint48 disabledTime) = vaults.atWithTimes(i);
        if (_wasActiveAt(enabledTime, disabledTime, epochStart)) {
++          tempVaults[activeCount] = vault;
            activeCount++;
        }
    }

    // Create the final result array with correct size
    address[] memory activeVaults = new address[](activeCount);
-- uint256 activeIndex = 0;
-- for (uint256 i = 0; i < vaultCount; i++) {
--      (address vault, uint48 enabledTime, uint48 disabledTime) = vaults.atWithTimes(i);
--      if (_wasActiveAt(enabledTime, disabledTime, epochStart)) {
--          activeVaults[activeIndex] = vault;
--          activeIndex++;
--       }
--    }
++    for (uint256 i = 0; i < activeCount; i++) {
++       activeVaults[i] = tempVaults[i];
++    }

    return activeVaults;
}
```

**Suzaku:**
커밋 [59a0109](https://github.com/suzaku-network/suzaku-core/commit/59a01095a1940aae4e75a87580695ca3e4d99712)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Unnecessary `onlyRegisteredOperatorNode` on `completeStakeUpdate` function

**Description:** `completeStakeUpdate`는 동일한 수정자가 적용된 내부 함수 `_completeStakeUpdate`를 호출합니다. 현재 수정자 `onlyRegisteredOperatorNode`가 두 번 확인됩니다.


**Recommended Mitigation:** `_completeStakeUpdate`에서 `onlyRegisteredOperatorNode`를 제거하는 것을 고려하십시오.

**Suzaku:**
커밋 [f9946ef](https://github.com/suzaku-network/suzaku-core/commit/f9946ef8f6c7d7ab946e01d906f411352004ee41)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Optimisation of elapsed epoch calculation

**Description:** `UptimeTracker::computeValidatorUptime` 함수에서 경과된 에포크 수를 계산하는 방법을 최적화할 기회가 있습니다.

```solidity
Currently, the code redundantly retrieves both epoch indices and their corresponding start timestamps to compute the elapsed time between epochs:

uint48 lastUptimeEpoch = l1Middleware.getEpochAtTs(uint48(lastUptimeCheckpoint.timestamp));
uint256 lastUptimeEpochStart = l1Middleware.getEpochStartTs(lastUptimeEpoch);

uint48 currentEpoch = l1Middleware.getEpochAtTs(uint48(block.timestamp));
uint256 currentEpochStart = l1Middleware.getEpochStartTs(currentEpoch);

uint256 elapsedTime = currentEpochStart - lastUptimeEpochStart;
uint256 elapsedEpochs = elapsedTime / epochDuration;
```

그러나 에포크 번호 자체를 이미 알고 있으므로(lastUptimeEpoch 및 currentEpoch), 에포크 인덱스를 빼서 전체 경과 에포크 수를 직접 계산할 수 있어 getEpochStartTs에 대한 불필요한 호출을 피할 수 있습니다.

**Recommended Mitigation:** 타임스탬프 기반 에포크 기간 계산을 더 간단하고 효율적인 버전으로 바꾸십시오:

```diff
- uint256 elapsedTime = currentEpochStart - lastUptimeEpochStart;
- uint256 elapsedEpochs = elapsedTime / epochDuration;
+ uint256 elapsedEpochs = currentEpoch - lastUptimeEpoch;
```

이 변경은 동일한 결과를 달성하면서 계산 오버헤드를 줄이고 로직을 단순화합니다.

**Suzaku:**
커밋 [f9946ef](https://github.com/suzaku-network/suzaku-core/commit/f9946ef8f6c7d7ab946e01d906f411352004ee41)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Use unchecked block for increment operations in `distributeRewards`

**Description**

`Rewards::distributeRewards`에서 `unchecked` 블록을 사용하여 루프 내부의 증분 연산에 대한 가스 소비를 최적화할 수 있습니다.

```solidity
function distributeRewards(uint48 epoch, uint48 batchSize) external onlyRole(REWARDS_DISTRIBUTOR_ROLE) {
    DistributionBatch storage batch = distributionBatches[epoch];
    uint48 currentEpoch = l1Middleware.getCurrentEpoch();

    if (batch.isComplete) revert AlreadyCompleted(epoch);
    // Rewards can only be distributed after a 2-epoch delay
    if (epoch >= currentEpoch - 2) revert RewardsDistributionTooEarly(epoch, currentEpoch - 2);

    address[] memory operators = l1Middleware.getAllOperators();
    uint256 operatorCount = 0;

    for (uint256 i = batch.lastProcessedOperator; i < operators.length && operatorCount < batchSize; i++) {
        // Calculate operator's total share based on stake and uptime
        _calculateOperatorShare(epoch, operators[i]);

        // Calculate and store vault shares
        _calculateAndStoreVaultShares(epoch, operators[i]);

        batch.lastProcessedOperator = i + 1;
        operatorCount++;
    }

    if (batch.lastProcessedOperator >= operators.length) {
        batch.isComplete = true;
    }
}
```

**Recommended Mitigation**

가스 사용량을 최적화하기 위해 증분 연산에 대한 `unchecked` 블록을 도입하십시오.

```diff
function distributeRewards(uint48 epoch, uint48 batchSize) external onlyRole(REWARDS_DISTRIBUTOR_ROLE) {
    DistributionBatch storage batch = distributionBatches[epoch];
    uint48 currentEpoch = l1Middleware.getCurrentEpoch();

    if (batch.isComplete) revert AlreadyCompleted(epoch);
    // Rewards can only be distributed after a 2-epoch delay
    if (epoch >= currentEpoch - 2) revert RewardsDistributionTooEarly(epoch, currentEpoch - 2);

    address[] memory operators = l1Middleware.getAllOperators();
    uint256 operatorCount = 0;

-   for (uint256 i = batch.lastProcessedOperator; i < operators.length && operatorCount < batchSize; i++) {
+   for (uint256 i = batch.lastProcessedOperator; i < operators.length && operatorCount < batchSize;) {
        // Calculate operator's total share based on stake and uptime
        _calculateOperatorShare(epoch, operators[i]);

        // Calculate and store vault shares
        _calculateAndStoreVaultShares(epoch, operators[i]);

+       unchecked {
            batch.lastProcessedOperator = i + 1;
            operatorCount++;
+          i++;
+       }
    }

    if (batch.lastProcessedOperator >= operators.length) {
        batch.isComplete = true;
    }
}
```

**Suzaku:**
커밋 [2fb0daf](https://github.com/suzaku-network/suzaku-core/commit/2fb0dafd684eeaf11b177602c5047d1e6ce2d715)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Optimize `_getStakerVaults` to Avoid Redundant External Calls to `activeBalanceOfAt`

**Description:** `Rewards::_getStakerVaults` 함수는 0이 아닌 잔액을 가진 볼트를 필터링하기 위해 볼트 배열을 두 번 통과하므로 `IVaultTokenized.activeBalanceOfAt`에 대한 외부 호출 횟수가 두 배로 늘어납니다.

```solidity
function _getStakerVaults(address staker, uint48 epoch) internal view returns (address[] memory) {
        address[] memory vaults = middlewareVaultManager.getVaults(epoch);
        uint48 epochStart = l1Middleware.getEpochStartTs(epoch);

        uint256 count = 0;

        // First pass: Count non-zero balance vaults
        for (uint256 i = 0; i < vaults.length; i++) {
            uint256 balance = IVaultTokenized(vaults[i]).activeBalanceOfAt(staker, epochStart, new bytes(0));
            if (balance > 0) {
                count++;
            }
        }

        // Create a new array with the exact number of valid vaults
        address[] memory validVaults = new address[](count);
        uint256 index = 0;

        // Second pass: Populate the new array
        for (uint256 i = 0; i < vaults.length; i++) {
            uint256 balance = IVaultTokenized(vaults[i]).activeBalanceOfAt(staker, epochStart, new bytes(0));
            if (balance > 0) {
                validVaults[index] = vaults[i];
                index++;
            }
        }

        return validVaults;
    }
```

**Recommended Mitigation:** `_getStakerVaults`를 다음과 같이 변경하십시오:

```diff
function _getStakerVaults(address staker, uint48 epoch) internal view returns (address[] memory) {
    address[] memory vaults = middlewareVaultManager.getVaults(epoch);
    uint48 epochStart = l1Middleware.getEpochStartTs(epoch);
+   address[] memory tempVaults = new address[](vaults.length); // Temporary oversized array
    uint256 count = 0;

-   // First pass: Count non-zero balance vaults
-   for (uint256 i = 0; i < vaults.length; i++) {
-       uint256 balance = IVaultTokenized(vaults[i]).activeBalanceOfAt(staker, epochStart, new bytes(0));
-       if (balance > 0) {
-           count++;
-       }
-   }
-
-   // Create a new array with the exact number of valid vaults
+   // Single pass: Collect valid vaults
+   for (uint256 i = 0; i < vaults.length;) {
+       if (IVaultTokenized(vaults[i]).activeBalanceOfAt(staker, epochStart, new bytes(0)) > 0) {
+           tempVaults[count] = vaults[i];
+           count++;
+       }
+       unchecked { i++; }
+   }
+
+   // Copy to correctly sized array
    address[] memory validVaults = new address[](count);
-   uint256 index = 0;
-
-   // Second pass: Populate the new array
-   for (uint256 i = 0; i < vaults.length; i++) {
-       uint256 balance = IVaultTokenized(vaults[i]).activeBalanceOfAt(staker, epochStart, new bytes(0));
-       if (balance > 0) {
-           validVaults[index] = vaults[i];
-           index++;
-       }
+   for (uint256 i = 0; i < count;) {
+       validVaults[i] = tempVaults[i];
+       unchecked { i++; }
    }

    return validVaults;
}
```

**Suzaku:**
커밋 [2fb0daf](https://github.com/suzaku-network/suzaku-core/commit/2fb0dafd684eeaf11b177602c5047d1e6ce2d715)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Redundant overflow checks in safe arithmetic operations

**Description:** `Solidity 0.8.25`에는 산술 연산에 대한 내장 오버플로우 검사가 포함되어 있어 연산당 약 20-30 가스가 추가됩니다. `UptimeTracker::computeValidatorUptime`에서는 루프 증가(i++) 및 덧셈(lastUptimeEpoch + i)과 같은 연산이 uint48 및 제어된 입력을 사용하므로 오버플로우하지 않음이 보장됩니다.

**Recommended Mitigation:** 안전한 산술 연산을 위해 unchecked 블록을 사용하십시오:
```solidity

for (uint48 i = 0; i < elapsedEpochs;) {
    uint48 epoch;
    unchecked {
        epoch = lastUptimeEpoch + i;
        i++;
    }
    // ...
}
```

**Suzaku:**
커밋 [2fb0daf](https://github.com/suzaku-network/suzaku-core/commit/2fb0dafd684eeaf11b177602c5047d1e6ce2d715)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.

\clearpage
