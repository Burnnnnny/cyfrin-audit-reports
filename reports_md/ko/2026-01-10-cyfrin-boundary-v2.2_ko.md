**Lead Auditors**

[Immeas](https://x.com/0ximmeas)

[BengalCatBalu](https://x.com/BengalCatBalu)


---

# 발견 사항 (Findings)
## 낮은 위험 (Low Risk)


### 모든 사용자가 출금하면 미베스팅 수량이 유실됨

**설명:** `sUSBD`는 기부 공격 방지를 위해 최초의 "소각된" 공유 잔액을 볼트 자신에게 영구적으로 민팅해 잠가 둡니다. 베스팅 기간 중 모든 사용자 공유가 상환/소각되어 잠긴 공유만 남게 되면, 남아 있는 미베스팅 보상은 계속 `totalAssets()`로 베스팅되면서 사실상 이 잠긴 공유에 귀속됩니다. 그 결과 원래 스테이커를 위한 보상이 영구적으로 청구 불가능해질 수 있습니다.

**영향:** 활성 베스팅 기간 동안 사용자가 완전히 이탈하면 미베스팅 보상의 일부 또는 전부가 되돌릴 수 없게 "유실"될 수 있습니다. 이 보상 자금은 볼트 안에 묶인 채 남고, 상환할 수 없는 잠긴 공유 잔액에 귀속됩니다.

**개념 증명 (Proof of Concept):** 다음 테스트를 `test/unit/staking/sUSDB.Withdraw.t.sol`에 추가하십시오.
```solidity
function test_VestedAmountIsLostIfAllUsersWithraw() public {
    // First set cooldown to 0 to enable direct withdraw/redeem
    vm.prank(operationalAdmin);
    isusbdMgmt.setCooldownDuration(0);

    vm.prank(rewarder);
    isusbd.transferInRewards(100e18);

    // user withdraws after half the vesting period
    vm.warp(block.timestamp + 7 days);

    vm.prank(user1);
    uint256 assets = isusbd.redeem(500e18, user1, user1);

    assertApproxEqAbs(assets, 550e18, 1e18);

    // rest of the vesting period goes
    vm.warp(block.timestamp + 7 days);

    uint256 burntAssets = isusbd.convertToAssets(isusbd.balanceOf(address(isusbd)));

    // rest of the vested amount is vested to the initial burnt shares
    assertApproxEqAbs(burntAssets, 51e18, 1e18);
}
```

**권장 완화 조치:** 다음 중 하나를 고려하십시오.

* 잠긴 공유 수량보다 충분히 큰 프로토콜 예치금을 항상 유지하여 적어도 한 명의 사용자가 항상 남도록 보장합니다.
* 잠긴 공유를 treasury로 쓸어 담는 "sweep" 메커니즘을 추가합니다. 예: `if(totalSupply() == 1e18) sweepAssets = convertToAssets(1e18) - 1e18)`


**Boundary:**
[PR#157](https://github.com/boundary-labs/boundary-protocol-ethereum/pull/157)에서 해결됨. `transferInRewards`에 활성 스테이커 확인을 추가했고, 초기 소각 예치금을 `1e12`로 낮췄습니다. 추가로 잠긴 공유보다 충분히 큰 프로토콜 예치금을 유지해 항상 최소 한 명의 스테이커가 존재하도록 할 예정입니다.

**Cyfrin:** 확인함. 이제 스테이커가 없으면 `transferInRewards`가 revert됩니다.


### `FULL` 제한을 우회할 수 있음

**설명:** `sUSBD`는 `_update()`에서 `from != address(0)` 및 `to != address(0)`인 경우에만 전송을 막아 `RestrictedStatus.FULL`을 강제합니다. 하지만 제한은 주소 단위로 적용될 뿐, 공유 자체에 "붙어 있는" 속성은 아닙니다. 어떤 계정이 곧 `FULL`로 설정될 것을 예상하면, `setRestrictedStatus(account, FULL)` 트랜잭션을 앞질러 자신의 sUSBD 공유를 새롭고 제한되지 않은 주소로 옮길 수 있습니다. 이후 제한 업데이트가 반영되면 원래 주소는 얼어붙지만, 공유는 이미 새 주소로 이동했으므로 계속 자유롭게 사용할 수 있습니다.

**영향:** `FULL` 제한을 우회할 수 있습니다.

**개념 증명 (Proof of Concept):**
1. `victim`이 `N`개의 sUSBD 공유를 보유합니다.
2. `RESTRICTION_MANAGER_ROLE`이 `setRestrictedStatus(victim, FULL)`을 mempool에 제출합니다.
3. `victim`은 이를 보고 더 높은 우선순위로 `transfer(newAddr, N)`을 제출해 선행 실행합니다.
4. `victim`은 아직 `FULL`이 아니고 `newAddr`도 제한되지 않았기 때문에 `transfer`는 성공합니다.
5. 이후 `setRestrictedStatus(victim, FULL)`이 실행되면 `victim`은 동결되지만 이미 공유는 0개이며, 실제 공유는 `newAddr`가 계속 정상적으로 출금/전송할 수 있습니다.

**권장 완화 조치:** 이 문제를 완전히 해결하려면 상당한 구조 변경이 필요합니다. 예를 들어 공유 전송 중 짧은 잠금 기간을 도입할 수 있습니다. 또 다른 방법은 사용자를 차단하는 거래를 Flashbots나 private mempool을 통해 수행하고, 이 운영 절차를 명확히 문서화하는 것입니다.

**Boundary:**
인지함. 이는 운영상 과제로 보고 있으며, 관리자 트랜잭션은 프런트런 방지를 위해 private mempool을 통해 제출할 예정입니다.


### `vestingPeriod`를 갱신하면 이미 베스팅이 끝난 보상이 다시 미베스팅으로 분류될 수 있음

**설명:** `sUSBD`는 `vestingAmount`와 `lastRewardTimestamp`를 사용해 활성 베스팅 "에포크"를 추적하고, `sUSBD::getUnvestedAmount`는 현재 `vestingPeriod`를 사용해 남은 미베스팅 보상을 계산합니다. 만약 보상이 유입된 후 `vestingPeriod`가 변경되면, 볼트는 이미 완전히 베스팅된 시점 이후에도 그 에포크에서 얼마가 여전히 "미베스팅"인지 소급해 다시 계산할 수 있습니다. `totalAssets()`는 `balanceOf(this) - getUnvestedAmount()`로 계산되므로, 이 현상은 (i) `getUnvestedAmount()`가 현재 볼트 잔액보다 커질 때 `totalAssets()` 언더플로우 및 revert를 일으키거나, (ii) revert 없이도 일시적으로 `totalAssets()`를 축소시켜 이른 시점 상환자는 적게 받고, 나중 상환자는 시간이 지나며 "부활한" 미베스팅 수량이 다시 감소함에 따라 더 많이 받는 불공정 분배를 초래할 수 있습니다.

**영향:** * **DoS / ERC4626 회계 손상:** `getUnvestedAmount() > vaultBalance`면 `totalAssets()` 및 이에 의존하는 변환/출금 호출이 충분한 시간이 지날 때까지 revert할 수 있습니다.
* **불공정한 보상 분배:** `0 < totalAssets() < vaultBalance`인 경우 공유 가격이 일시적으로 낮아져 그 시점 상환자는 적은 자산을 받고, 같은 에포크가 재베스팅되면서 남은 보유자는 더 많은 자산을 받습니다. 즉, 보상 분배가 보유 기간보다 상환 시점에 따라 달라지게 됩니다.

**개념 증명 (Proof of Concept):** 다음 테스트를 `test/unit/staking/main/sUSBD.GetUnvestedAmount.t.sol`에 추가하면, vesting period 변경이 베스팅을 다시 활성화해 `totalAssets` 호출 시 DoS를 일으키는 모습을 확인할 수 있습니다.
```solidity
function test_GetUnvestedAmount_CanResurrectAfterVestingPeriodIncrease_AndMakeTotalAssetsUnderflow()
    public
{
    // Enable direct withdraw/redeem path (so user can exit fully)
    vm.prank(operationalAdmin);
    isusbdMgmt.setCooldownDuration(0);

    vm.prank(operationalAdmin);
    isusbdMgmt.setVestingPeriod(1);

    // Start a vesting epoch
    vm.prank(rewarder);
    isusbd.transferInRewards(REWARD_AMOUNT);

    // Let rewards fully vest under the current vestingPeriod
    vm.warp(block.timestamp + _vault.vestingPeriod());
    assertEq(isusbd.getUnvestedAmount(), 0);

    // User exits, pulling out vested rewards (leaving only the permanently locked shares behind)
    vm.startPrank(user1);
    isusbd.redeem(isusbd.balanceOf(user1), user1, user1);
    vm.stopPrank();
    // Increase vesting period AFTER vesting completed. Because getUnvestedAmount() uses the *current*
    // vestingPeriod with the same vesting epoch state, this can "resurrect" a non-zero unvested amount.
    vm.prank(operationalAdmin);
    isusbdMgmt.setVestingPeriod(uint24(14 days));

    uint256 resurrectedUnvested = isusbd.getUnvestedAmount();
    uint256 vaultBal = IERC20(_vault.asset()).balanceOf(address(isusbd));

    // The problematic condition: unvested becomes larger than the vault's actual balance
    assertGt(resurrectedUnvested, vaultBal);

    // totalAssets() does (balance - unvested) -> underflow panic (0x11)
    vm.expectRevert(stdError.arithmeticError);
    isusbd.totalAssets();
}
```
다음 두 import도 함께 추가하십시오.
```diff
+ import { IERC20 } from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
+ import { stdError } from "forge-std/StdError.sol";
```

**권장 완화 조치:** vesting period를 갱신할 때 `vestingAmount = 0`으로 초기화하는 방안을 고려하십시오.
```diff
    function setVestingPeriod(
        uint24 period
    ) external onlyRole(OPERATIONAL_ADMIN_ROLE) {
        if (period > MAX_VESTING_PERIOD) revert ExceedsMax();
        if (period == 0 && cooldownDuration == 0) revert ZeroCooldownAndPeriod();
        if (getUnvestedAmount() != 0) revert VestingInProgress();

        uint24 oldPeriod = vestingPeriod;
        vestingPeriod = period;
+       vestingAmount = 0;

        emit VestingPeriodUpdated(msg.sender, oldPeriod, period);
    }
```

**Boundary:**
해결됨. [PR#181](https://github.com/boundary-labs/boundary-protocol-ethereum/pull/181)에서 권고안대로 `vestingAmount`를 0으로 재설정하도록 수정했습니다.

**Cyfrin:** 확인함. 권장 수정이 구현되었습니다.

\clearpage
## 정보 (Informational)


### 볼트 조작 엣지 케이스 완화를 위한 최소 입금액 추가

**설명:** 일반적인 ERC4626 볼트 조작 엣지 케이스에 대한 노출을 줄이기 위해 기본적인 입력값 및 반올림 검증을 추가하는 것을 고려하십시오. 특히 작은 최소 입금액(예: `>= 1e18` USBD)을 강제하면, 조작 시도나 사용자 혼란을 유발할 수 있는 dust/반올림 기반의 griefing 결과를 줄이는 데 도움이 됩니다.


**Boundary:**
인지함. 이 볼트는 이미 배포 시 `1e12` 공유를 자신에게 소각하여 inflation/donation 공격을 영구적으로 방어하고, dust 입금이 0 공유로 반올림되는 상황도 막고 있습니다. 여기에 `_deposit`와 `_withdraw` 내 기존 `ZeroAssets`, `ZeroShares` 검사가 있으므로 최소 입금액 강제는 중복이라고 판단합니다.


### 상환 문서의 정보가 잘못되어 있음

**설명:** 현재 프로토콜 문서 파일 `03-Fund-Flows`의 `B.Redeeming` 섹션은 `redeem`을 실행하려면 `benefactor`가 USDB를 `USDBMinting` 컨트랙트로 보내야 한다고 설명합니다. 그러나 코드에는 그런 동작이 구현되어 있지 않습니다. 실제로는 상환 과정에서 `USDB::burnFrom`가 호출되므로, USDB는 `benefactor`가 보유하고 있어야 하며 대신 `USDBMinting` 주소에 allowance를 부여해야 합니다.

**Boundary:**
해결됨. 문서가 [PR#162](https://github.com/boundary-labs/boundary-protocol-ethereum/pull/162)에서 수정되었습니다.

**Cyfrin:** 확인함.


### 위임받은 서명자(delegate)는 benefactor가 부여한 위임을 스스로 제거할 수 없음

**설명:** delegate는 benefactor가 부여한 위임을 제거할 수 없습니다. `acceptDelegatedSigner` 호출로 위임이 수락되면, 이후 이를 철회할 수 있는 쪽은 `removeDelegatedSigner`를 호출하는 benefactor뿐입니다.

이는 `05-Mint-and-Redeem` 문서에서 양측 모두 위임을 철회할 수 있다고 설명하는 내용과 모순됩니다. 따라서 delegate 입장에서는 기능이 부족한 상태입니다.

```
Delegated Signing - Both EOA and contract benefactors can delegate signing to an approved EOA. Delegation requires two steps: benefactor calls initiateDelegatedSigner, delegate calls acceptDelegatedSigner. Either party can remove the delegation.
```
**권장 완화 조치:** delegate도 위임을 제거할 수 있는 기능을 추가하십시오.

**Boundary:**
해결됨. 문서는 [PR#165](https://github.com/boundary-labs/boundary-protocol-ethereum/pull/165)에서 수정되었습니다. 컨트랙트 동작 자체는 올바르며, delegate 관리는 benefactor의 책임입니다.

**Cyfrin:** 확인함.


### 스마트 계정 benefactor는 ERC-1271 서명 검증과 delegated signer를 동시에 사용할 수 없음

**설명:** `USBDMinting::_verifySignature`에서 스마트 계정 benefactor는 자신의 ERC-1271 서명 로직과 delegated signer를 동시에 사용할 수 없습니다. `_delegatesPerBenefactor[benefactor]`에 delegate가 하나라도 추가되면, 해당 benefactor의 서명 검증은 ECDSA 경로로 전환되고 ERC-1271 검증은 더 이상 사용되지 않습니다. 그 결과 스마트 계정 benefactor에게는 ERC-1271 기반 검증과 delegated signing이 상호 배타적인 관계가 됩니다.

스마트 계정 benefactor가 delegate가 존재하더라도 ERC-1271 검증을 유지할 수 있게 하거나, 이 제한이 의도된 설계라면 명시적으로 문서화하는 방안을 고려하십시오.


**Boundary:**
해결됨. 의도된 설계입니다. 컨트랙트 benefactor는 하나의 서명 모드만 선택합니다. 문서는 [PR#167](https://github.com/boundary-labs/boundary-protocol-ethereum/pull/167)에서 명확히 보완되었습니다.

**Cyfrin:** 확인함.


### benefactor 제거 후에도 잔여 권한이 남음

**설명:** `removeBenefactor`로 benefactor를 제거해도, 그 benefactor에 대해 이전에 설정된 beneficiary와 delegated signer 정보는 정리되지 않습니다. 같은 주소가 나중에 benefactor로 다시 추가되면, 이전에 승인된 beneficiary와 delegate가 자동으로 다시 활성화됩니다. 즉, benefactor를 제거해도 기존 신뢰 관계가 완전히 철회되지 않아 예상과 다를 수 있습니다.

benefactor 제거 시 beneficiary 및 delegated signer 데이터를 모두 지우거나, benefactor 제거는 mint/redeem 기능만 일시 비활성화할 뿐 과거 승인 정보는 유지된다는 점을 명시적으로 문서화하는 것을 고려하십시오.

**Boundary:**
해결됨. 이는 의도된 설계입니다. benefactor는 승인 상태와 무관하게 자신의 설정을 독립적으로 관리합니다. 문서는 [PR#169](https://github.com/boundary-labs/boundary-protocol-ethereum/pull/169)에서 명확히 보완되었습니다.

**Cyfrin:** 확인함.


### 쿨다운 비활성화 시 대기 중인 쿨다운 자금이 즉시 해제됨

**설명:** 문서에는 `cooldownDuration`이 0으로 설정될 때 이미 cooldown 상태에 들어가 있는 자금이 어떻게 처리되는지 명확히 설명되어 있지 않습니다. 현재 구현에서는 cooldown을 비활성화하면 silo에 pending cooldown 잔액이 있는 사용자가 원래의 `cooldownEnd` 시점에 도달하지 않았더라도 `unstake()`를 통해 즉시 자금을 인출할 수 있습니다. 이는 문서만 신뢰하는 사용자나 통합자에게는 직관적이지 않을 수 있으며, 사실상 진행 중이던 모든 cooldown이 취소되는 효과를 냅니다.

**권장 완화 조치:** `cooldownDuration`을 0으로 설정하면 모든 pending cooldown 잔액이 즉시 해제되고 `unstake()`를 통해 즉시 인출 가능해진다는 점을 명시적으로 문서화하십시오.

**Boundary:**
해결됨. 문서가 [PR#170](https://github.com/boundary-labs/boundary-protocol-ethereum/pull/170)에서 명확히 수정되었습니다.

**Cyfrin:** 확인함.


### ERC-7702 benefactor는 현재 서명 검증 로직에서 지원되지 않음

**설명:** `USBDMinting`의 서명 검증 로직은 `address.code.length`를 사용해 EOA와 컨트랙트 계정을 구분합니다. `benefactor.code.length == 0`이면 `ECDSA` 서명을 허용하고, 그렇지 않으면 delegated signer가 없는 한 ERC-1271 검증 경로를 탑니다.

하지만 `ERC-7702`에서는 EOA가 코드(위임 표시자)를 가질 수 있으면서도 여전히 트랜잭션을 시작하는 계정처럼 동작할 수 있습니다. 이 경우 delegated signer가 없는 ERC-7702 benefactor는 컨트랙트 계정으로 간주되어 ERC-1271 검증만 강제됩니다. 만약 해당 delegated logic이 ERC-1271 서명 검증을 구현하지 않았다면, benefactor의 올바른 ECDSA 서명도 거부됩니다.

그 결과 ERC-7702 기반 benefactor는 해당 delegated 컨트랙트가 호환 가능한 서명 검증을 명시적으로 처리하지 않는 한 사실상 지원되지 않습니다. 이 동작은 암묵적이며 현재 문서화되어 있지 않습니다.

**권장 완화 조치:** 서명 처리에서 EOA/컨트랙트 구분을 위해 `address.code.length`에 의존하지 않도록 하거나, ERC-7702 benefactor는 ERC-1271 호환 서명 검증을 구현해야 한다는 점(또는 delegated signer를 사용해야 한다는 점)을 명시적으로 문서화하십시오.

**Boundary:**
해결됨. 이 사안에 대해서는 OpenZeppelin의 입장([Issue 5707](https://github.com/OpenZeppelin/openzeppelin-contracts/issues/5707))을 따릅니다. 사용자가 ERC-7702를 통해 자신의 EOA를 위임했다면, 이는 그 컨트랙트의 동작을 명시적으로 선택한 것입니다. 이를 무시하고 강제로 ECDSA 검증을 수행하는 것은 사용자 의도를 덮어쓰는 행위가 됩니다. 문서는 [PR#178](https://github.com/boundary-labs/boundary-protocol-ethereum/pull/178)에서 명확히 보완되었습니다.

**Cyfrin:** 확인함.


### 쿨다운 출금은 ERC4626의 allowance 기반 owner 흐름을 지원하지 않음

**설명:** `ERC-4626`은 withdraw 및 redeem가 `msg.sender`가 owner의 공유에 대해 충분한 ERC-20 allowance를 가질 경우, owner 주소의 공유를 소각하는 흐름을 [반드시 지원](https://eips.ethereum.org/EIPS/eip-4626)해야 한다고 규정합니다. 이 구현에서는 `cooldownDuration == 0`일 때 표준 `withdraw`와 `redeem`가 이 요구사항을 올바르게 만족합니다. 그러나 cooldown 모드가 활성화되면 이 함수들은 명시적으로 비활성화되고, 대신 `cooldownAssets` 또는 `cooldownShares`를 사용해야 합니다.

이 cooldown 출금 함수들은 owner 파라미터를 제공하지 않고 항상 `msg.sender`의 공유만 소각하므로, cooldown 모드에서는 allowance 기반 출금이 불가능합니다. 따라서 정상 모드에서는 ERC-4626 요구사항을 충족하더라도, cooldown 상태에서는 그 동작이 유지되지 않습니다.

**권장 완화 조치:** cooldown 출금 로직을 owner 파라미터와 allowance 검사까지 포함하도록 확장하거나, cooldown 모드에서는 의도적으로 share owner만 출금을 시작할 수 있다는 점을 명확히 문서화하십시오.

**Boundary:**
인지함. owner 파라미터를 생략한 이유는 griefing 공격 벡터를 방지하기 위해서입니다.


### 제한된 주소로 소유권 이전이 가능함

**설명:** `sUSBD`는 `sUSBD::setRestrictedStatus`에서 현재 `owner()`와 `treasury`를 제한 대상으로 설정하는 것을 막고, `sUSBD::setTreasury`에서도 제한 상태가 `NONE`이 아닌 주소를 treasury로 설정하지 못하게 합니다.

그러나 소유권 변경에는 동일한 보호 장치가 없습니다. 즉, owner는 이미 제한된 주소(`SOFT/HARD/FULL`)로도 소유권을 이전할 수 있습니다. 이런 일이 발생하면 새로운 owner 주소는 `setRestrictedStatus`로 제한을 해제할 수 없는데, 해당 함수가 `account == owner()`인 경우 업데이트를 명시적으로 거부하기 때문입니다. 결국 이런 owner 주소를 "비제한" 상태로 되돌리려면, 다른 비제한 주소로 다시 소유권을 넘기는 수밖에 없습니다.

프로토콜의 제한 정책과 일치하도록 `transferOwnership` 같은 소유권 이전 흐름을 오버라이드하거나, `restrictedStatuses[newOwner] == RestrictedStatus.NONE`를 강제하는 보호 장치를 추가하는 것을 고려하십시오.


**Boundary:**
인지함. 제한 상태는 관리자 작업에는 영향을 주지 않으므로 추가 보호 장치는 필요하지 않다고 판단합니다.

\clearpage
