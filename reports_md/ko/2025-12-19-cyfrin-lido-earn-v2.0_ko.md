**Lead Auditors**

[Stalin](https://x.com/0xStalin)

[Immeas](https://x.com/0ximmeas)

**Assisting Auditors**



---

# 발견 사항 (Findings)
## 낮은 위험 (Low Risk)


### `ERC4626Adapter::maxMint`는 상한이 없는 대상 볼트에서 revert됨

**설명:** `ERC4626Adapter::maxMint`는 `TARGET_VAULT.maxDeposit(address(this))` 결과를 `_convertToShares`에 전달합니다.
```solidity
function maxMint(address /* user */ ) public view override returns (uint256) {
    if (paused() || emergencyMode) return 0;
    uint256 maxAssets = TARGET_VAULT.maxDeposit(address(this));
    return _convertToShares(maxAssets, Math.Rounding.Floor);
}
```
[`maxMint`/`maxDeposit`에 대한 EIP-4626 표준](https://eips.ethereum.org/EIPS/eip-4626#maxdeposit)은 다음과 같이 규정합니다.
> MUST return `2 ** 256 - 1` if there is no limit on the maximum amount of assets that may be deposited.

따라서 표준 ERC4626 구현(OpenZeppelin 등)은 `maxDeposit` / `maxMint`의 기본값으로 `type(uint256).max`를 반환합니다. 그런데 이 값을 `_convertToShares`에 넘기면 `Math.mulDiv`가 오버플로우되어 revert하므로, `maxMint`는 유효한 상한을 반환하지 못하고 스스로 revert하게 됩니다.

**영향:** `ERC4626Adapter::maxMint`는 상한이 없는 볼트에서 revert하며, 이는 [EIP-4626 표준](https://eips.ethereum.org/EIPS/eip-4626#maxmint)의 "`maxMint` MUST NOT revert" 요구사항과 어긋납니다.

**개념 증명 (Proof of Concept):** 다음 테스트를 `ERC4626Adapter.MaxDeposit.t.sol`에 추가하십시오.
```solidity
function test_MaxMintReverts() public {
    vm.expectRevert(stdError.arithmeticError);
    vault.maxMint(alice);
}
```

**권장 완화 조치:** 대상 볼트가 "무제한"을 반환한 경우를 별도로 처리하고, `type(uint256).max`를 `_convertToShares`에 넘기지 않도록 하십시오. 예를 들면:

```solidity
function maxMint(address /* user */ ) public view override returns (uint256) {
    if (paused() || emergencyMode) return 0;

    uint256 maxAssets = TARGET_VAULT.maxDeposit(address(this));
    if (maxAssets == type(uint256).max) {
        // Underlying vault is effectively uncapped: propagate this instead of converting
        return type(uint256).max;
    }

    return _convertToShares(maxAssets, Math.Rounding.Floor);
}
```

**Lido:** 커밋 [`af57eb5`](https://github.com/lidofinance/defi-interface/commit/af57eb5e85911976b76318d9a527319812fb3130)에서 수정됨

**Cyfrin:** 확인함. 제안된 수정이 구현되었습니다.


### 볼트 입출금 시 비표준 이벤트가 발생함

**설명:** `Vault`는 `Vault::deposit`/`mint` 및 `withdraw`/`redeem`에서 커스텀 이벤트 `Deposited`, `Withdrawn`를 발생시키지만, [EIP-4626](https://eips.ethereum.org/EIPS/eip-4626#events)는 표준 이벤트 이름인 `Deposit`, `Withdraw`를 명시합니다.

**영향:** 이벤트 수준에서 구현이 표준을 따르지 않아, 표준 ERC-4626 이벤트를 기준으로 인덱싱이나 회계를 수행하는 툴링/통합체가 깨질 수 있습니다.

**권장 완화 조치:** 커스텀 `Deposited` / `Withdrawn` 대신 EIP-4626이 규정한 정확한 시그니처의 `Deposit`, `Withdraw` 이벤트를 발생시키십시오.

**Lido:** 커밋 [`52217ad`](https://github.com/lidofinance/defi-interface/commit/52217ad4ad48f0f8fc8534e78ad1032af66c4152)에서 수정됨

**Cyfrin:** 확인함. 올바른 EIP-4626 이벤트가 이제 발생합니다.


### `emergencyMode`와 `recoveryMode` 사이에 `TARGET_VAULT` 공유를 기부해 `recoveryMode` 동안 예치자의 환율을 조작하는 griefing 공격 가능

**설명:** recovery mode를 활성화할 때 수수료 수확(`harvest fees`) 호출이 실행되는데, 이 과정에서 마지막 `lastTotalAssets` 갱신 이후 이익이 감지되면 더 많은 공유를 민팅합니다.
하지만 시스템 설계상 recoveryMode가 활성화되면:
- 더 이상 `TARGET_VAULT`에서 출금할 수 없고
- fee harvesting은 `TARGET_VAULT` 내 LidoVault 보유분을 `totalAssets`의 일부로 계산하지만, 그 시점부터 남은 `TARGET_VAULT` 공유는 더 이상 상환 가능한 자산이 아닙니다.

recoveryMode 전환 이후 볼트의 환율은, recovery mode 활성화 시점에 LidoVault가 실제로 보유한 underlying token 잔액과 totalSupply를 기준으로 계산됩니다.

이 구조는 recovery mode 활성화 트랜잭션이 프런트런되고, 그 사이에 `TARGET_VAULT` 공유가 LidoVault로 기부되는 griefing 공격을 허용합니다. 이 기부는 `totalAssets`를 증가시켜 시스템이 이익이 발생했다고 오인하게 만들고, 그 결과 수수료를 부과하기 위해 새 공유를 민팅하게 됩니다. 이는 결국 LidoVault의 실제 underlying token 잔액 대비 환율을 희석시킵니다.
```solidity
    function activateRecovery() external virtual onlyRole(EMERGENCY_ROLE) nonReentrant {
        if (recoveryMode) revert RecoveryModeAlreadyActive();
        if (!emergencyMode) revert EmergencyModeNotActive();

        //@audit => The donation of TARGET_VAULT shares causes more shares to be minted
        _harvestFees();

        uint256 actualBalance = IERC20(asset()).balanceOf(address(this));
        if (actualBalance == 0) revert InvalidRecoveryAssets(actualBalance);

        uint256 supply = totalSupply();
       ...

        recoveryAssets = actualBalance;
        recoverySupply = supply;
        recoveryMode = true;

        emit RecoveryModeActivated(actualBalance, supply, protocolBalance, implicitLoss);
    }

    function convertToAssets(uint256 shares) public view virtual override returns (uint256) {
        //@audit => exchange rate during recovery mode no longer considers the TARGET_VAULT's shares worth in underlying token.
        if (recoveryMode) {
            return shares.mulDiv(recoveryAssets, recoverySupply, Math.Rounding.Floor);
        }
        return super.convertToAssets(shares);
    }
```

**영향:** recovery 환율이 조작되어 예치자들이 원래 받을 수 있었던 것보다 적은 토큰만 회수하게 될 수 있습니다.

이 griefing 공격은 공격자가 스스로 손실을 감수해야 하므로 가능성은 낮지만, 일단 발생하면 예치자 자산 손실을 유발할 수 있어 영향은 큽니다.

**개념 증명 (Proof of Concept):**
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.30;

import {Vault} from "src/Vault.sol";
import "./ERC4626AdapterTestBase.sol";

contract ERC4626AdapterPoCs is ERC4626AdapterTestBase {

    function test_PoC_manipulateShareRatioOnRecoveryMode() public {
        //@audit-info => The mitigation would be to swap `_harvestFees()` to `emergencyWithdraw()` and add a function to allow Governance withdrawing from vault once recoveryMode is enabled!
        uint256 depositAmount = 100e6;
        vault.setRewardFee(2000);

        vm.prank(alice);
        vault.deposit(depositAmount, alice);

        vault.emergencyWithdraw();
        assertEq(vault.totalAssets(),depositAmount);

        uint256 aliceAssetsDuringEmergency = vault.convertToAssets(vault.balanceOf(alice));
        assertEq(aliceAssetsDuringEmergency, depositAmount);

        uint256 snapshot = vm.snapshot();
        {
            //@audit-info => A donation to the LidoVault of TARGET_VAULT's shares
            vm.startPrank(bob);
            usdc.approve(address(targetVault), depositAmount);
            targetVault.deposit(depositAmount, address(vault));
            vm.stopPrank();

            vault.activateRecovery();
            assertEq(depositAmount, vault.recoveryAssets());

            //@audit => Manipulation -> depositor gets less assets that could've otherwise got
            uint256 aliceAssetsOnRecoveryMode = vault.convertToAssets(vault.balanceOf(alice));
            assertTrue(aliceAssetsDuringEmergency > aliceAssetsOnRecoveryMode);

            emit log_named_uint("aliceAssetsDuringEmergency: ", aliceAssetsDuringEmergency);
            emit log_named_uint("aliceAssetsOnRecoveryMode: ", aliceAssetsOnRecoveryMode);
        }

        vm.revertTo(snapshot);

        vault.activateRecovery();

        //@audit => No manipulation -> depositor gets the correct exchange rate during recoveryMode
        uint256 aliceAssetsOnRecoveryMode = vault.convertToAssets(vault.balanceOf(alice));
        assertEq(aliceAssetsOnRecoveryMode, aliceAssetsDuringEmergency);
    }
}
```

**권장 완화 조치:**
1. recovery mode 활성화 시점이 아니라 `emergencyWithdraw` 과정에서 수수료를 수확하는 것을 고려하십시오.
2. `recoveryMode` 활성화 후에는 `ERC4626Adapter::recoverERC20`가 남은 `TARGET_VAULT` 토큰을 쓸어올 수 있도록 허용하는 것을 고려하십시오.

이 접근은 LidoVault의 실제 underlying token 잔액을 기준으로 기대되는 환율을 보존하는 것을 우선시하고, 남은 `TARGET_VAULT` 공유를 sweeping하여 잠재적인 수수료 이익을 나중 문제로 미루는 보다 방어적인 전략입니다.

**Lido**
커밋 [4fd0eb7](https://github.com/lidofinance/defi-interface/commit/4fd0eb7207f607179bf470c77877baafd79dfd53), [ee29862](https://github.com/lidofinance/defi-interface/commit/ee298626d62e4d18f44b10a5cd6cbcbd3cae7188)에서 수정됨

**Cyfrin:** 확인함. 이제 `_harvestFees`는 `emergencyMode` 활성화 시점에 호출됩니다. 또한 `recoveryMode`가 활성화되면 남은 `TARGET_VAULT` 공유는 거버넌스가 `recoverERC20`을 통해 회수할 수 있습니다.


### `EmergencyVault::activateRecovery`는 revert하는 `TARGET_VAULT` 호출에 의해 DoS될 수 있음

**설명:** `EmergencyVault::activateRecovery`는 `_harvestFees()`(내부에서 `totalAssets()`와 `_getProtocolBalance()` 호출)와 `_getProtocolBalance()`를 직접 호출하면서 `TARGET_VAULT`에 외부 호출을 수행합니다. `_getProtocolBalance()`는 `TARGET_VAULT.balanceOf(address(this))`, `TARGET_VAULT.convertToAssets(...)` 같은 외부 view 호출을 수행하는데, 대상 볼트가 손상되었거나 업그레이드 가능하거나 비정상 동작하는 경우 revert할 수 있습니다. 그 결과, 어댑터가 이미 로컬에 회수 가능한 자산을 보유하고 있어도 recovery 활성화 경로 전체가 DoS될 수 있습니다.

**영향:** 사고 상황에서 `TARGET_VAULT`가 더 이상 신뢰할 수 없고 그 view 함수들이 revert한다면, vault는 `recoveryMode`를 활성화하지 못할 수 있습니다. 그러면 vault 컨트랙트가 이미 회수한 자산도 recovery 흐름이 열리기 전까지 사용자들이 상환할 수 없어 묶일 수 있습니다.

**권장 완화 조치:** `activateRecovery()`가 사고 상황에서 revert할 수 있는 외부 target vault 호출에 의존하지 않도록 리팩터링하십시오. `_getProtocolBalance()`는 이벤트 정보용에 불과하므로 `activateRecovery()`에서 제거하고, fee harvesting은 `emergencyWithdraw` 호출 이후 수동으로 처리하도록 하십시오(`emergencyMode` 제한을 제거하거나 `EMERGENCY_ROLE`에만 제한하는 방식 포함). 또는 `_harvestFees()`를 `emergencyWithdraw()` 끝부분으로 옮길 수 있습니다.

**Lido:** 커밋 [`ee29862`](https://github.com/lidofinance/defi-interface/commit/ee298626d62e4d18f44b10a5cd6cbcbd3cae7188)에서 수정됨

**Cyfrin:** 확인함. `_harvestFees()`는 `emergencyWithdraw()`로 이동했고, `_getProtocolBalance` 호출에는 `try/catch`가 추가되었습니다.


### `ERC4626Adapter::maxMint`는 수확되지 않은 대기 수수료를 고려하지 않아 실제 민팅 가능한 공유 수를 과소 계산함

**설명:** `ERC4626Adapter::maxMint`는 대상 `TARGET_VAULT`에 예치 가능한 최대 자산량을 대응하는 볼트 공유 수로 변환해 Vault가 발행할 수 있는 최대 공유 수를 계산합니다. 그러나 이 변환 과정은 대기 중인 수수료를 고려하지 않습니다.

그 결과, Vault가 보고하는 최대 공유 수는 실제보다 적게 나옵니다. 대기 수수료를 harvest하면 추가 공유가 민팅되어 total supply와 자산 대비 공유 환산 비율이 증가하기 때문입니다. 따라서 같은 자산량으로도 `maxMint`가 초기에 보고한 것보다 더 많은 공유를 발행할 수 있게 됩니다.

**영향:** `maxMint`가 실제 민팅 가능한 최대 공유 수를 정확히 보고하지 못합니다.

**개념 증명 (Proof of Concept):** 다음 테스트를 `ERC4626Adapter.MaxDeposit.t.sol`에 추가하십시오.
```solidity
    function test_PoC_MaxMint_DoesNotConsiderPendingYield() public {
        uint256 yield = 50_00e6;
        targetVault.setLiquidityCap(500_000e6);

        _seedVaults(yield);

        //@audit-info => Vault has pending yield

        uint256 snapshot = vm.snapshot();
            uint256 maxDeposit = vault.maxDeposit(alice);
            vm.prank(alice);
            vault.deposit(maxDeposit, alice);
            assertEq(vault.maxDeposit(alice), 0);
            assertEq(vault.maxMint(alice), 0);
        vm.revertTo(snapshot);

        //@audit-info => Given that Vault has pending yield, maxMint() is not accurate and will mint less shares than the actual maxMint post harvesting fees
        uint256 maxMintShares = vault.maxMint(alice);
        vm.prank(alice);
        vault.mint(maxMintShares, alice);
        assertGt(vault.maxDeposit(alice), 0);
        assertGt(vault.maxMint(alice), 0);

        //@audit-info => After attempting to mint the maxShares reported by the vault (and fees have been harvested during the mint), a second mint is possible when it shouldn't be because the previous mint was supposed to mint the max
        maxMintShares = vault.maxMint(alice);
        vm.prank(alice);
        vault.mint(maxMintShares, alice);
        assertEq(vault.maxDeposit(alice), 0);
        assertEq(vault.maxMint(alice), 0);
    }

    function _seedVaults(uint256 yield) internal {
        vm.prank(alice);
        vault.deposit(100_000e6, alice);

        vm.startPrank(bob);
        usdc.approve(address(targetVault), 100_000e6);
        targetVault.deposit(100_000e6, bob);
        vault.deposit(100_000e6, alice);
        vm.stopPrank();

        // mint yield to targetVault
        usdc.mint(address(targetVault), yield);
    }

```

**권장 완화 조치:** `previewMint`, `previewDeposit`처럼 대기 중인 수수료 harvest를 고려해 공유 수를 계산하는 방식을 `maxMint`에도 적용하는 것을 고려하십시오.

**Lido:** 커밋 [fc15b10](https://github.com/lidofinance/defi-interface/commit/fc15b104d3859955ed341e2785059e2806c6aa36)에서 수정됨.

**Cyfrin:** 확인함. `maxMint`는 이제 `TARGET_VAULT`에 예치 가능한 `maxAssets`를 `previewDeposit`에 전달합니다. `previewDeposit`는 대기 중인 수수료를 올바르게 반영하므로, 계산된 최대 공유 수는 수수료 수확 이후 실제로 발행 가능한 최대 공유 수와 일치합니다.

\clearpage
## 정보 (Informational)


### 키와 값의 목적을 더 명확히 드러내기 위해 named mapping 사용

**설명:** 키와 값의 역할이 명확히 드러나도록 named mapping을 사용하는 것을 고려하십시오.
```solidity
RewardDistributor.sol
52:    mapping(address => bool) private recipientExists;
```

**Lido:** 커밋 [4898c26](https://github.com/lidofinance/defi-interface/commit/4898c26cd0abc8426ad9e2220a8d7cac487ab9b8)에서 수정됨.

**Cyfrin:** 확인함.


### Solidity에서는 기본값으로의 초기화를 피할 것

**설명:** Solidity에서는 기본값으로의 초기화는 불필요합니다.
```solidity
RewardDistributor.sol
143:        uint256 totalBps = 0;
145:        for (uint256 i = 0; i < recipients_.length; i++) {
230:        for (uint256 i = 0; i < recipientsLength; i++) {
```

**Lido:** 커밋 [4898c26](https://github.com/lidofinance/defi-interface/commit/4898c26cd0abc8426ad9e2220a8d7cac487ab9b8)에서 `totalBps`에 대해 수정함.

**Cyfrin:** 확인함.


### `OFFSET`이 사용될 때 `Vault::decimals`는 올바른 decimals를 반영하지 않음

**설명:** [`Vault::decimals`](https://github.com/lidofinance/defi-interface/blob/99fd2b2c64c345a3c14b023dca4cb6393ffce5aa/src/Vault.sol#L543-L545)는 단순히 asset decimals를 반환합니다.
```solidity
function decimals() public view virtual override(ERC20, ERC4626) returns (uint8) {
    return IERC20Metadata(asset()).decimals();
}
```
이는 decimal offset을 고려하는 OpenZeppelin의 [ERC4626 구현](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/extensions/ERC4626.sol#L129-L131)과 다릅니다.
```solidity
function decimals() public view virtual override(IERC20Metadata, ERC20) returns (uint8) {
    return _underlyingDecimals + _decimalsOffset();
}
```

**영향:** 계산 오류를 직접 일으키지는 않지만, 환율과 공유 토큰의 decimals 표기가 어색하게 보일 수 있습니다.

**권장 완화 조치:** OpenZeppelin 구현을 따라 `OFFSET`을 decimals에 반영하는 것을 고려하십시오.
```diff
- return IERC20Metadata(asset()).decimals();
+ return IERC20Metadata(asset()).decimals(); + OFFSET;
```

**Lido:** 커밋 [`9ce9c0a`](https://github.com/lidofinance/defi-interface/commit/9ce9c0a5bef423933ec357ab3900d899069f2107)에서 수정됨

**Cyfrin:** 확인함. 이제 decimals에 `OFFSET`이 더해집니다.


### `EmergencyVault::activateRecovery` NatSpec이 잘못된 이벤트 이름을 참조함

**설명:** `EmergencyVault::activateRecovery`의 NatSpec은 `RecoveryActivated`를 발생시킨다고 적고 있습니다.
```solidity
*      Emits RecoveryActivated(actualBalance, totalSupply, protocolBalance, implicitLoss)
```
하지만 실제로 발생시키는 이벤트는 `RecoveryModeActivated`입니다.
```solidity
emit RecoveryModeActivated(actualBalance, supply, protocolBalance, implicitLoss);
```
NatSpec이 올바른 이벤트 이름을 참조하도록 수정하는 것을 고려하십시오.

**Lido:** 커밋 [`3d89267`](https://github.com/lidofinance/defi-interface/commit/3d89267ee409eb0857abb9302dcc42337429e4f9)에서 수정됨

**Cyfrin:** 확인함.


### `nonReentrant`가 첫 번째 수정자가 아님

**설명:** `EmergencyVault::emergencyWithdraw`와 `EmergencyVault::activateRecovery`는 `nonReentrant`를 첫 번째가 아니라 두 번째 수정자로 두고 있습니다. 다른 수정자 내부에서의 재진입도 막으려면, `nonReentrant`는 수정자 목록의 첫 번째에 오는 것이 바람직합니다.

**Lido:** 커밋 [`3d89267`](https://github.com/lidofinance/defi-interface/commit/3d89267ee409eb0857abb9302dcc42337429e4f9)에서 수정됨

**Cyfrin:** 확인함.


### 수수료 공유 계산 시 `_decimalsOffset`이 반영되지 않음

**설명:** 코드베이스 전반에서 assets를 shares로 바꾸는 공식은 환율 조작 방지를 위해 virtual supply를 사용합니다. 하지만 누적된 수수료에 대해 발행할 공유 수를 계산할 때는 `_calculateFeeShares()`를 호출하지 않고 `_decimalsOffset()`도 반영하지 않습니다.

다음 PoC를 `ERC4626Adapter.Fees.t.sol`에 추가하십시오.

```solidity
    function test_PoC_InflateRatioViaFees() public {
        vm.prank(alice);
        uint256 alice_receivedShares = vault.deposit(1, alice);

        emit log_named_uint("totalAssets", vault.totalAssets());
        emit log_named_uint("totalSupply", vault.totalSupply());

        emit log_named_uint("assets per wei of share", vault.convertToAssets(alice_receivedShares));

        usdc.mint(address(targetVault), 1_000e18); //
        vault.harvestFees();

        emit log_named_uint("totalSupply", vault.totalSupply());
        emit log_named_uint("assets per wei of share", vault.convertToAssets(alice_receivedShares));

        vm.prank(bob);
        uint256 bob_receivedShares = vault.deposit(100e18, bob);

        uint256 bobAssetsBeforeRedeem = usdc.balanceOf(bob);

        vm.prank(bob);
        vault.redeem(bob_receivedShares, bob, bob);

        uint256 bobAssetsAfterRedeem = usdc.balanceOf(bob);

        emit log_named_uint("bob assets withdrawn", bobAssetsAfterRedeem
         - bobAssetsBeforeRedeem);
        assertTrue(bobAssetsAfterRedeem - bobAssetsBeforeRedeem > 99e18);

        // @audit //
        // With current formula, bob withdraws: 99999880924647933225 [9.999e19])

        // With formula using _decimalsOffset(): val: 99999803787934739453 [9.999e19])

        // @audit => Difference is neglegible for the required amount to donate to inflate the ratio //

    }

```

**Lido:** 인지함. 정밀도 손실이 1 wei를 넘지 않으며, fee receiver가 받는 실제 가치에는 영향이 없습니다.

\clearpage
## 가스 최적화 (Gas Optimization)


### 더 빠른 `nonReentrant` 수식자를 위해 `ReentrancyGuardTransient` 사용

**설명:** 더 빠른 `nonReentrant` 수식자를 위해 [ReentrancyGuardTransient](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/ReentrancyGuardTransient.sol) 사용을 고려하십시오.
```solidity
Vault.sol
10:import {ReentrancyGuard} from "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
32:abstract contract Vault is ERC4626, ERC20Permit, AccessControl, ReentrancyGuard, Pausable {
```

**Lido:** 커밋 [3d89267](https://github.com/lidofinance/defi-interface/commit/3d89267ee409eb0857abb9302dcc42337429e4f9)에서 수정됨.

**Cyfrin:** 확인함.


### recipient 계정과 basis points를 더 효율적으로 읽는 방법 사용

**설명:** `RewardsDistributor::getRecipient`의 가스 비용을 791 -> 716으로 줄이려면 다음과 같이 하십시오.
```solidity
function getRecipient(uint256 index) external view returns (address account, uint256 basisPoints) {
    Recipient storage recipient = recipients[index];
    (account, basisPoints) = (recipient.account, recipient.basisPoints);
}
```

`RewardsDistributor::distribute`의 가스 비용을 59254 -> 59172로 줄이려면:
```solidity
for (uint256 i = 0; i < recipientsLength; i++) {
    Recipient storage recipient = recipients[i];
    (address account, uint256 basisPoints) = (recipient.account, recipient.basisPoints);

    uint256 amount = (balance * basisPoints) / MAX_BASIS_POINTS;

    if (amount > 0) {
        tokenContract.safeTransfer(account, amount);
        emit RecipientPaid(account, token, amount);
    }

    totalAmount += amount;
}
```

**개념 증명 (Proof of Concept):** `RewardDistributor.t.sol`에서 확인하려면:
1. `test_ReplaceRecipient_Succeeds` 함수의 마지막 호출 뒤에 snapshot을 추가합니다.
```diff
function test_ReplaceRecipient_Succeeds() public {
    RewardDistributor distributor = _deployDefaultDistributor();
    address newRecipient = makeAddr("newRecipient");

    (address oldRecipient,) = distributor.getRecipient(0);

    vm.expectEmit(true, true, true, true);
    emit RewardDistributor.RecipientReplaced(0, oldRecipient, newRecipient);

    vm.prank(admin);
    distributor.replaceRecipient(0, newRecipient);

    (address updatedRecipient,) = distributor.getRecipient(0);
+   vm.snapshotGasLastCall("RewardsDistributor", "getRecipient");
    assertEq(updatedRecipient, newRecipient);
}
```

2. `test_Distribute_DistributesAccordingToBps` 함수의 첫 호출 뒤에도 snapshot을 추가합니다.
```diff
function test_Distribute_DistributesAccordingToBps() public {
    RewardDistributor distributor = _deployDefaultDistributor();
    uint256 amount = 10_000e6;
    asset.mint(address(distributor), amount);

    address[] memory recipients = new address[](2);
    recipients[0] = recipientA;
    recipients[1] = recipientB;

    uint256[] memory expectedAmounts = new uint256[](2);
    expectedAmounts[0] = (amount * 4_000) / MAX_BPS;
    expectedAmounts[1] = (amount * 6_000) / MAX_BPS;

    vm.recordLogs();
    vm.prank(admin);
    distributor.distribute(address(asset));
+   vm.snapshotGasLastCall("RewardsDistributor", "distribute");

    Vm.Log[] memory entries = vm.getRecordedLogs();
    // snip remaining code...
```

3. 테스트 컨트랙트를 실행합니다: `forge test --match-contract RewardDistributorTest`

4. 가스 스냅샷을 확인합니다: `more snapshots/RewardsDistributor.json`

5. 권장 변경을 적용한 뒤 3), 4)를 다시 실행합니다.

**Lido:** 커밋 [4898c26](https://github.com/lidofinance/defi-interface/commit/4898c26cd0abc8426ad9e2220a8d7cac487ab9b8)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage
