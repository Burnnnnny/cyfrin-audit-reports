**Lead Auditors**

[Stalin](https://x.com/0xStalin)

[Arno](https://x.com/_0xarno_)

[InAllHonesty](https://x.com/0xInAllHonesty)

**Assisting Auditors**


---

# 발견 사항 (Findings)
## 치명적 위험 (Critical Risk)


### `Tranche::_withdraw`에서 `CDO::withdraw`로 전달되는 매개변수가 뒤바뀌어 `sUSDe` 인출자는 항상 손실을 입음

**설명:** 사용자는 `sUSDe` 또는 `USDe` 중 하나를 선택하여 인출할 수 있습니다. 시스템은 요청된 자산과 해당 자산의 `tokenAmount`를 기반으로 인출될 `USDe` 가치를 결정하기 위해 적절한 계산을 수행해야 합니다. 계산된 `USDe` 가치를 기반으로 시스템은 시스템에서 인출되는 `USDe` 양에 대해 필요한 `TrancheShares`를 소각합니다.

이 문제는 `Tranche::_withdraw`가 `CDO::withdraw`에 두 개의 매개변수를 역순으로 전달하는 문제를 보고합니다. 이러한 매개변수는 `baseAssets`와 `tokenAssets`입니다.
CDO는 `tokenAssets`를 먼저 받고 그 다음에 `baseAssets`를 받을 것으로 예상하지만, Tranche는 `baseAssets`를 먼저 전달한 다음 `tokenAssets`를 전달합니다.
  - 역순의 매개변수는 시스템이 잘못된 금액으로 계산을 수행하게 하여(`Strategy` 컨트랙트에서) 사용자에게 릴리스할 `sUSDe` 양이 원래보다 훨씬 적어지게 합니다(특히 `sUSDe <=> USDe` 비율이 높을 때).

```solidity
// Tranche::_withdraw() //
    function _withdraw(
        address token,
        address caller,
        address receiver,
        address owner,
        uint256 baseAssets,
        uint256 tokenAssets,
        uint256 shares
    ) internal virtual {
        ...
//@audit => Burn Trancheshares for the full requested sUSDe
@>      _burn(owner, shares);

//@audit-issue => Sends baseAssets first and then tokenAssets
@>      cdo.withdraw(address(this), token, baseAssets, tokenAssets, receiver);
        ...
    }

// StrataCDO::withdraw() //
    function withdraw(address tranche, address token, uint256 tokenAmount, uint256 baseAssets, address receiver) external onlyTranche nonReentrant {
        ...
//@audit => Because of the inverted parameters `tokenAmount` is actually `baseAssets`, and `baseAssets` is actually `tokenAssets`
 @>     strategy.withdraw(tranche, token, tokenAmount, baseAssets, receiver);
        ...
    }

// Strategy::withdraw() //
    function withdraw (address tranche, address token, uint256 tokenAmount, uint256 baseAssets, address receiver) external onlyCDO returns (uint256) {
//@audit => `baseAssets` should represent amount of `USDe` being withdrawn, but, because of the inverted parameters, here represents the actual requested amount of `sUSDe` to withdraw
        uint256 shares = sUSDe.previewWithdraw(baseAssets);
        if (token == address(sUSDe)) {
            uint256 cooldownSeconds = cdo.isJrt (tranche) ? sUSDeCooldownJrt : sUSDeCooldownSrt;
//@audit => transfers calculates `shares` of `sUSDe` to Cooldown to be sent to the user after cooldown.
            erc20Cooldown.transfer(sUSDe, receiver, shares, cooldownSeconds);
            return shares;
        }
        ...
    }
```

**영향:** `sUSDe` 인출자는 `USDe`로 받은 가치에 비해 더 많은 `TrancheShares`가 소각되기 때문에 항상 `USDe` 가치 손실을 입게 됩니다.

**개념 증명 (Proof of Concept):** 다음 PoC에서는 `sUSDe`를 인출하는 사용자가 요청한 양보다 적은 `sSUDe`를 받고 그 과정에서 `TrancheShares`가 소각되어 어떻게 손실을 입는지 보여줍니다.

다음 PoC에서 증명된 바와 같이, `sUSDe => USDe` 비율이 1:1.5라고 가정합니다.
사용자가 100 `sUSDe`(150 `USDe` 가치)를 인출 요청합니다.
- 150 JRTranche가 소각됩니다.
- ERC20Cooldown은 요청된 100 `sUSDe`를 받아야 합니다.

인출자는 약 66 `sUSDe`만 받게 되며, 이는 150 `USDe`를 인출할 수 있는 100 `sUSDe`를 전송하는 대신 100 `USDe`만 인출할 수 있습니다.

`CDO.t.sol` 테스트 파일에 다음 PoC를 추가하십시오.

```solidity
    function test_WithdrawingsUSDECausesLosesForUsers() public {
        address bob = makeAddr("Bob");
        USDe.mint(bob, 1000 ether);
        vm.startPrank(bob);
            //@audit => Bob initializes the exchange rate on sUSDe
            USDe.approve(address(sUSDe), type(uint256).max);
            sUSDe.deposit(1000 ether, bob);
        vm.stopPrank();

        address alice = makeAddr("Alice");
        uint256 initialDeposit = 150 ether;
        USDe.mint(alice, initialDeposit);

        //@audit-info => There are 1k sUSDe in circulation and 1k USDe deposited on the sUSDe contract
        //@audit-info => Exchange Rate is 1:1
        assertEq(USDe.balanceOf(address(sUSDe)), 1000 ether);
        assertEq(sUSDe.totalSupply(), 1000 ether);
        assertEq(sUSDe.convertToAssets(1e18), 1e18);

        // Simulate yield by adding USDe directly to sUSDe contract
        //@audit-info => Set sUSDe exchange rate to USDe (1:1.5)
        USDe.mint(address(sUSDe), 500 ether); // 50% yield
        assertApproxEqAbs(sUSDe.convertToAssets(1e18), 1.5e18, 1e6);

        //@audit-info => Bob deposits sUSDe when sUSDE rate to USDe is 1:1.5
        vm.startPrank(bob);
            sUSDe.approve(address(jrtVault), type(uint256).max);
            jrtVault.deposit(address(sUSDe), 100e18, bob);
            assertApproxEqAbs(jrtVault.balanceOf(bob), 150e18, 1e6);
        vm.stopPrank();

        vm.startPrank(alice);
            //@audit => Alice gets 100 sUSDe by staking 150 USDe
            USDe.approve(address(sUSDe), type(uint256).max);
            sUSDe.deposit(150e18, alice);
            assertApproxEqAbs(sUSDe.balanceOf(alice), 100e18, 1e6);

            //@audit => Alice deposits 100 sUSDe on the JRTranche and gets 150 JRTrancheShares
            sUSDe.approve(address(jrtVault), type(uint256).max);
            jrtVault.deposit(address(sUSDe), 100e18, alice);
            assertApproxEqAbs(jrtVault.balanceOf(alice), 150e18, 1e6);
            assertEq(sUSDe.balanceOf(alice), 0);

            //@audit-info => Requests to withdraw 100e18 sUSDe which are worth 150 USDe
            uint256 expected_sUSDeWithdrawn = 100e18;
            uint256 expected_USDe_valueWithdrawn = sUSDe.convertToAssets(expected_sUSDeWithdrawn);

            //@audit-issue => Alice withdraws 100 sUSDe but gets only ~66.6 sUSDe, all her JRTrancheShares are burnt
            jrtVault.withdraw(address(sUSDe), expected_sUSDeWithdrawn, alice, alice);
            uint256 alice_actual_sUSDeBalance = sUSDe.balanceOf(alice);
            uint256 alice_USDe_actualWithdrawn = sUSDe.convertToAssets(alice_actual_sUSDeBalance);

            assertEq(jrtVault.balanceOf(alice), 0);
            assertApproxEqAbs(alice_actual_sUSDeBalance, 66.5e18, 1e18);

            console2.log("Alice expected withdrawn sUSDe: ", expected_sUSDeWithdrawn);
            console2.log("Alice actual withdrawn sUSDe: ", alice_actual_sUSDeBalance);
            console2.log("====");
            console2.log("Alice expected withdrawn USDe value: ", expected_USDe_valueWithdrawn);
            console2.log("Alice actual withdrawn USDe value: ", alice_USDe_actualWithdrawn);
        vm.stopPrank();
    }
```

**권장 완화 조치:** `Tranche::_withdraw`에서 `CDO::withdraw`를 호출할 때 매개변수를 올바른 순서로 전달하도록 하십시오.

```diff
    function _withdraw(
        address token,
        address caller,
        address receiver,
        address owner,
        uint256 baseAssets,
        uint256 tokenAssets,
        uint256 shares
    ) internal virtual {
        ...
-       cdo.withdraw(address(this), token, baseAssets, tokenAssets, receiver);
+       cdo.withdraw(address(this), token, tokenAssets, baseAssets, receiver);

    }

```

**Strata:**
커밋 [31d9b72](https://github.com/Strata-Money/contracts-tranches/commit/31d9b7248073652ce28d579d4511d5b93414c6be)에서 매개변수를 올바른 순서로 전달하여 수정됨.

**Cyfrin:** 확인함.


## 높은 위험 (High Risk)


### 악의적인 사용자에 의해 사용자의 활성 인출 요청이 DoS될 수 있음

**설명:** 사용자는 `USDe` 또는 `sUSDe` 중 하나를 선택하여 인출할 수 있습니다.
- `sUSDe`를 인출할 때 자산은 인출이 요청된 트랜치에 따라 쿨다운 기간에 들어갈 수 있습니다.
- `USDe`를 인출할 때 인출이 요청된 트랜치에 관계없이 `USDe`를 받으려면 Ethena 컨트랙트에서 `sUSDe`를 언스테이킹해야 하므로 `UnstakeCooldown` 컨트랙트에 새로운 인출 요청이 생성됩니다.

이 문제가 보고하는 문제는 악의적인 사용자가 다른 사용자의 활성 인출 요청에 DoS를 유발하여 사실상 자산이 시스템에 갇히게 만드는 슬픔(grief) 공격입니다.

이 슬픔 공격은 1 wei만큼 적은 금액을 인출하고 `receiver`를 공격자가 피해를 입히고자 하는 피해자 사용자로 설정하여 달성됩니다.
`Tranche`, `CDO`, `Strategy` 컨트랙트 모두 인출자가 `receiver`에 의해 자신을 대신하여 인출을 요청하도록 승인되었는지 확인하지 않으므로, 누구나 누구를 대신하여 새로운 인출 요청을 할 수 있습니다.
- 각각의 새로운 인출 요청은 `UnstakingContract`의 배열에 푸시되며, 쿨다운 기간이 끝나면 `UnstakingContract.finalize()`가 해당 배열을 반복하여 모든 준비된 요청을 처리합니다. 공격은 이 배열을 부풀려(공격자에 의해 부풀려짐) 수천 개의 활성 요청을 반복함으로써 `out of gas error`를 유발합니다.

언스테이킹 컨트랙트에는 접근 제어가 없으므로, 대안적인 접근 방식은 언스테이킹 컨트랙트의 `request()` 메서드를 직접 호출하는 것입니다. 이를 통해 시스템의 핵심 컨트랙트를 우회하고 언스테이킹 컨트랙트에서 사용자의 인출 요청을 직접 부풀릴 수 있습니다.

```solidity
    function transfer(IERC20 token, address to, uint256 amount) external {
@>      address from = msg.sender;
        ...
        SafeERC20.safeTransferFrom(token, from, address(proxy), amount);
        ...

@>      requests.push(TRequest(uint64(unlockAt), proxy));
        emit Requested(address(token), to, amount, unlockAt);
    }

```

**영향:** 악의적인 사용자는 다른 사용자가 UnstakeCooldown 컨트랙트를 통해 USDe를 인출할 때 자산 인출을 완료하지 못하도록 DoS를 유발할 수 있습니다.

**개념 증명 (Proof of Concept):** `CDO.t.sol` 파일에 다음 PoC를 추가하십시오.
이 PoC는 악의적인 사용자가 1 wei만큼 적은 금액에 대해 막대한 양의 인출을 요청하여 다른 사용자의 활성 요청 인출을 완전히 DoS할 수 있는 방법을 보여줍니다. 결과적으로 악의적인 사용자는 가스 비용을 충당하는 적은 양의 리소스로 활성 인출을 완전히 DoS할 수 있습니다.

```solidity
    function test_DoSUserActiveRequests() public {
        address alice = makeAddr("Alice");
        address bob = makeAddr("Bob");

        uint256 initialDeposit = 1000 ether;
        USDe.mint(alice, initialDeposit);
        USDe.mint(bob, initialDeposit);

        vm.startPrank(alice);
        USDe.approve(address(jrtVault), type(uint256).max);
        jrtVault.deposit(initialDeposit, alice);
        vm.stopPrank();

        vm.startPrank(bob);
        USDe.approve(address(jrtVault), type(uint256).max);
        jrtVault.deposit(initialDeposit, bob);
        vm.stopPrank();

        //@audit => Alice requests to withdraw its full USDe balance
        uint256 totalWithdrawableAssets = jrtVault.maxWithdraw(alice);
        vm.prank(alice);
        jrtVault.withdraw(totalWithdrawableAssets, alice, alice);

        //@audit => Bob does a huge amount of tiny withdrawals to inflate the activeRequests array of Alice
        vm.pauseGasMetering();
            for(uint i = 0; i < 35_000; i++) {
                vm.prank(bob);
                jrtVault.withdraw(1, alice, bob);
            }
        vm.resumeGasMetering();

        //@audit-info => Skip till a timestamp where the requests can be finalized
        skip(1_000_000);

        //@audit-issue => Alice gets DoS her tx to finalize the withdrawal of her USDe balance
        vm.prank(alice);
        unstakeCooldown.finalize(sUSDe, alice);
    }
```

**권장 완화 조치:** 최소 인출 금액과 사용자가 자신을 대신하여 새로운 인출을 요청할 수 있는 사람을 지정할 수 있는 권한 메커니즘을 결합하십시오.

위의 완화 조치 외에도, `UnstakeCooldown` 및 `ERC20Cooldown` 컨트랙트에서 `transfer()`를 호출할 수 있는 사람을 제한하십시오. 누구나 `transfer()` 함수를 호출할 수 있는 경우, 두 Cooldown 컨트랙트 중 하나에서 `transfer()`를 직접 호출하여 이전에 언급한 완화 조치를 우회할 수 있습니다.

**Strata:**
커밋 ea7371d, da327dc, 9b5ac62에서 다음을 통해 수정됨:
1. `UnstakeCooldown::transfer` 및 `ERC20Cooldown::transfer`에 접근 제어 추가.
2. 수신자가 아닌 계정이 생성한 요청에 대한 소프트 제한과 실제 수신자가 생성한 요청에 대한 하드 제한 설정. 하드 제한에 도달하면 후속 요청은 목록의 마지막 요청에 추가됩니다.

**Cyfrin:** 확인함. 직접 함수를 호출하여 발생하는 슬픔(grief)을 방지하기 위해 접근 제어를 구현했습니다. 권한 없는 사용자가 가짜 요청을 스팸하고 인출자의 요청 대기열을 채우는 것을 방지하기 위해 제한을 구현했습니다.


### 기부 공격을 방지하는 메커니즘이 게임화되어 인출이 되돌려지고 자산이 전략(Strategy)에 갇히게 될 수 있음

**설명:** 인출이 발생할 때마다 `TrancheShares` `totalSupply()`가 미리 정의된 경계(`MIN_SHARES`) 아래로 떨어지지 않도록 확인이 수행됩니다. 그러나 이 메커니즘을 게임화하여 `shares<=>assets ratio`가 조작되게 함으로써 모든 인출이 되돌려지게 하는 방법이 있습니다. 이로 인해 Tranche가 소량의 지분을 발행하게 되어 `totalSupply()`가 `MIN_SHARES` 경계를 초과하지 않게 되고, 결과적으로 모든 인출이 되돌려집니다.

```solidity

    function _withdraw(
        ...
    ) internal override {
       ...
        _onAfterWithdrawalChecks();
        emit Withdraw(caller, receiver, owner, assets, shares);
    }

    function _onAfterWithdrawalChecks () internal view {
@>      if (totalSupply() < MIN_SHARES) {
            revert MinSharesViolation();
        }
    }
```

공격은 공격자가 `sUSDe`를 `Strategy`에 직접 기부하는 첫 번째 기부 공격입니다. 이는 시스템의 `totalAssets`를 부풀리고, 그런 다음 공격자는 `1.1USDe`를 예치하여 share<=>assets 비율이 조작되게 합니다. 즉, 1.1e18 USDe 예치에 대해 Tranche는 1 wei의 지분을 발행합니다. 결과적으로 후속 예치는 조작된 비율로 지분을 발행하게 되며, `_onAfterWithdrawalChecks()`로 인해 남은 `totalSupply()`가 `MIN_SHARES`를 초과하지 않기 때문에 인출을 시도할 때 문제가 발생합니다.

**영향:** 인출 메커니즘이 손상되어 사용자가 예치한 자산이 `Strategy` 컨트랙트에 갇히게 될 수 있습니다.

**개념 증명 (Proof of Concept):** `CDO.t.sol` 테스트 파일에 다음 PoC를 추가하고 import 섹션에서 `IErrors` 인터페이스를 가져옵니다.
`import { IErrors } from "../contracts/tranches/interfaces/IErrors.sol";`

```solidity
    function test_GameDonationAttackProtectionToTrapAssets() public {
        address alice = makeAddr("Alice");
        address bob = makeAddr("Bob");
        // Same value as in the Tranche contract
        uint256 MIN_SHARES = 0.1 ether;

        USDe.mint(bob, 1000 ether);
        vm.startPrank(bob);
            //@audit => Bob initializes the exchange rate on sUSDe
            USDe.approve(address(sUSDe), type(uint256).max);
            sUSDe.deposit(1000 ether, bob);
        vm.stopPrank();

        uint256 initialDeposit = 1000 ether;
        USDe.mint(alice, initialDeposit);
        USDe.mint(bob, initialDeposit);

        vm.startPrank(alice);
            USDe.approve(address(sUSDe), type(uint256).max);
            sUSDe.deposit(1e18, alice);
            // Step 1 => Alice transfers 1 sUSDe directly to the strategy to inflate the exchange rate and deposits 1.1e18 USDe that results in minting 1 TrancheShare
            sUSDe.transfer(address(sUSDeStrategy), 1e18);
            USDe.approve(address(jrtVault), type(uint256).max);
            jrtVault.deposit(1.1e18, alice);
        vm.stopPrank();

        assertEq(jrtVault.totalSupply(), 1);

        USDe.mint(bob, 1_000_000e18);
        vm.startPrank(bob);
            USDe.approve(address(jrtVault), type(uint256).max);
            //Step 2 => Now Bob deposits 1million USDe into the Tranche
            //Because of the manipulated exchange rate, the total minted TrancheShares for such a big deposits won't even be enough to reach the MIN_SHARES
            jrtVault.deposit(1_000_000e18, bob);
            assertLt(jrtVault.totalSupply(), MIN_SHARES);

            //Step 3 => Bob attempts to make a withdrawal, but the withdrawal reverts because the total shares on the Tranche don't reach MIN_SHARES
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
```

**권장 완화 조치:**
1. Tranche가 배포될 때 동일한 tx에서 최소 1 full USDe를 예치하는 것을 고려하십시오. 이상적으로는 예치 수신자를 `address(1)`로 설정하십시오. 이 지분은 사용할 수 없는 것으로 간주되어야 하므로, 발행된 TrancheShares는 MIN_SHARES 아래로 떨어지지 않도록 하는 하한선으로 효과적으로 사용됩니다.

2. 배포 스크립트에 특별히 주의를 기울이십시오. 전략이 Tranche에서 예치된 자산을 가져올 수 있도록 승인을 부여하려면 `Tranche::configure`를 호출해야 합니다(이 함수는 2개의 Tranche와 Strategy가 배포되어야 하는 `StrataCDO::configure`를 호출하는 데 필요합니다).

**Strata:**
준비금(reserve)에 첫 입금을 하기 전에 전략에 대한 모든 기부를 전송함으로써 커밋 [f344885](https://github.com/Strata-Money/contracts-tranches/commit/f344885d35f2acfd93b80a9c8d37df7e89ae2f08)에서 수정됨.

**Cyfrin:** 확인함. 동일한 트랜잭션에서 예금을 활성화하고 첫 번째 예금을 할 때, 이 시점 전에는 예금이 활성화되지 않았으므로 전략에 대한 기부를 청소하기 위해 첫 번째 예금을 하기 전에 `cdo::reduceReserve`를 호출해야 합니다.

\clearpage
## 중간 위험 (Medium Risk)


### `Tranche::redeem`이 `super.redeem` 대신 `super.withdraw`를 호출하여 사용자가 더 적은 자산을 받게 됨

**설명:** `Tranche::redeem` 함수에는 `super.redeem(shares, receiver, owner)` 대신 `super.withdraw(shares, receiver, owner)`를 호출하는 버그가 있습니다. 이로 인해 함수가 shares 매개변수를 assets 매개변수로 잘못 처리하여 잘못된 계산이 발생합니다.

```solidity
    function redeem(uint256 shares, address receiver, address owner) public override returns (uint256) {
        cdo.updateAccounting();
        uint256 assets = super.withdraw(shares, receiver, owner);
        return assets;
    }
```

[EIP4626](https://eips.ethereum.org/EIPS/eip-4626) 표준에 따르면:

- redeem(uint256 shares, ...)은 정확히 shares 양을 소각하고 해당 assets를 반환해야 합니다.
- withdraw(uint256 assets, ...)은 정확히 assets 양을 인출하고 소각된 shares를 반환해야 합니다.

이 버그로 인해 redeem(100 shares)은 내부적으로 withdraw(100 assets)를 호출하여 100을 지분이 아닌 자산으로 처리합니다.


**영향:** 사용자는 예상보다 훨씬 적은 자산을 받습니다.
사용자는 의도한 정확한 지분 금액을 상환할 수 없습니다.
ERC4626 표준 위반: 함수는 다음을 통해 ERC4626 사양을 위반합니다.

1. 지정된 정확한 수의 지분을 소각하지 않음
2. 잘못된 값 반환(전송된 자산 대신 소각된 지분)
3. 기본적인 redeem/withdraw 구분을 깨뜨림

**개념 증명 (Proof of Concept):**

```solidity
function test_RedeemBug() public {

    alice = makeAddr("alice");
    USDe.mint(alice, 10000 ether);

    vm.startPrank(alice);
    USDe.approve(address(jrtVault), type(uint256).max);


    uint256 initialDeposit = 1000 ether;
    uint256 sharesReceived = jrtVault.deposit(initialDeposit, alice);

    vm.stopPrank();

    // Simulate yield by adding USDe directly to sUSDe contract
    // This increases the asset-to-share ratio in sUSDe, creating yield
    USDe.mint(address(sUSDe), 200 ether); // 20% yield

    vm.startPrank(alice);

    console2.log("After yield simulation");
    console2.log("------------------------");
    console2.log("Alice shares balance:", jrtVault.balanceOf(alice));
    console2.log("Vault totalAssets:", jrtVault.totalAssets());
    console2.log("Alice convertToAssets:", jrtVault.convertToAssets(jrtVault.balanceOf(alice)));

    // Calculate exchange rate
    uint256 totalSupplyShares = jrtVault.totalSupply();
    uint256 totalAssetsVault = jrtVault.totalAssets();
    console2.log("Exchange rate (assets per share * 1e18):", totalSupplyShares > 0 ? totalAssetsVault * 1e18 / totalSupplyShares : 1e18);

    uint256 sharesToRedeem = 100 ether;
    uint256 expectedAssets = jrtVault.previewRedeem(sharesToRedeem);
    uint256 sharesBeforeRedeem = jrtVault.balanceOf(alice);

    console2.log("Before redeem");
    console2.log("------------------------");
    console2.log("Shares to redeem:", sharesToRedeem);
    console2.log("Expected assets from previewRedeem:", expectedAssets);
    console2.log("Alice shares before:", sharesBeforeRedeem);

    // Call redeem - this calls super.withdraw instead of super.redeem
    uint256 actualAssetsReturned = jrtVault.redeem(sharesToRedeem, alice, alice);
    uint256 sharesAfterRedeem = jrtVault.balanceOf(alice);
    uint256 sharesBurned = sharesBeforeRedeem - sharesAfterRedeem;

    console2.log("After redeem");
    console2.log("------------------------");
    console2.log("Actual assets returned:", actualAssetsReturned);
    console2.log("Alice shares after:", sharesAfterRedeem);
    console2.log("Shares actually burned:", sharesBurned);

    // For comparison: what would happen if we called withdraw with same numerical value
    uint256 assetsToWithdraw = sharesToRedeem; // Same number as shares
    uint256 expectedSharesForWithdraw = jrtVault.previewWithdraw(assetsToWithdraw);

    console2.log("Comparison: withdraw behavior");
    console2.log("------------------------");
    console2.log("If we called withdraw(", assetsToWithdraw, "):");
    console2.log("Expected shares burned by withdraw:", expectedSharesForWithdraw);
    console2.log("Expected assets returned by withdraw:", assetsToWithdraw);

    console2.log("------------------------");
    console2.log("Expected assets from proper redeem:", expectedAssets);
    console2.log("Actual assets returned:", actualAssetsReturned);
    console2.log("Difference:", expectedAssets > actualAssetsReturned ? expectedAssets - actualAssetsReturned : 0);

    assertTrue(expectedAssets != actualAssetsReturned, "redeem returned wrong asset amount");

    vm.stopPrank();
}
```

**권장 완화 조치:** 함수 호출을 `super.withdraw`에서 `super.redeem`으로 변경하십시오.

```diff
function redeem(uint256 shares, address receiver, address owner) public override returns (uint256) {
    cdo.updateAccounting();
-   uint256 assets = super.withdraw(shares, receiver, owner);
+   uint256 assets = super.redeem(shares, receiver, owner);
    return assets;
}
```

**Strata:**
올바른 함수를 호출하여 커밋 [31d9b72](https://github.com/Strata-Money/contracts-tranches/commit/31d9b7248073652ce28d579d4511d5b93414c6be)에서 수정됨.

**Cyfrin:** 확인함.


### 받을 자산으로 `USDe`를 요청하여 준비금을 줄이면 전략이 필요한 것보다 더 많은 `sUSDe`를 릴리스함

**설명:** 준비금을 줄이고 `USDe`를 요청할 때, Strategy는 요청된 `USDe`의 `tokenAmount`를 커버하는 데 필요한 `sUSDe`만 전송하는 대신 실제 요청된 `USDe`(`tokenAmount`) 양만큼 `sUSDe`를 잘못 전송합니다.

```solidity
//sUSDeStrategy::reduceReserve()//
    function reduceReserve (address token, uint256 tokenAmount, address receiver) external onlyCDO {
        ...
        if (token == address(USDe)) {
            //@audit-issue => transfer sUSDe for `tokenAmount` which is in USDe.
            unstakeCooldown.transfer(sUSDe, receiver, tokenAmount);
            return;
        }
        revert UnsupportedToken(token);
    }
```

**영향:** 전략은 USDe를 요청하여 준비금을 줄일 때 인출 요청된 USDe를 커버하는 데 필요한 것보다 더 많은 sUSDe를 릴리스합니다. 이는 전략이 원래 가져야 할 것보다 더 적은 USDe를 남기게 되므로 예금자가 손실을 입게 됨을 의미합니다.

**개념 증명 (Proof of Concept):** 예를 들어, `sUSDe` <=> `USDe` 비율이 1:1.5이고 150 USDe에 대한 준비금을 줄이도록 요청된 경우입니다.
- 전략은 `100 sUSDe`(150 USDe 가치)만 보내는 **대신** `150 sUSDe`(225 USDe 가치)를 `treasury`로 보냅니다.


**권장 완화 조치:** USDe를 요청하여 준비금을 줄일 때 `sUSDeStrategy::reduceReserve`에서 `sUSDe::previewWithdraw`를 호출하여 요청된 `USDe`의 `tokenAmount`를 얻는 데 필요한 sUSDe 양을 얻고 해당 양의 `sUSDe`를 `UnstakeCooldown` 컨트랙트로 전송하십시오.

```solidity
    function reduceReserve (address token, uint256 tokenAmount, address receiver) external onlyCDO {
       ...
        if (token == address(USDe)) {
+           uint256 shares = sUSDe.previewWithdraw(tokenAmount);
+           unstakeCooldown.transfer(sUSDe, receiver, shares);
-           unstakeCooldown.transfer(sUSDe, receiver, tokenAmount);
            return;
        }
        revert UnsupportedToken(token);
    }
```

**Strata:**
`tokenAmount`를 `sUSDe` 단위의 `shares`로 변환하여 커밋 [953c3bc](https://github.com/Strata-Money/contracts-tranches/commit/953c3bc8ee4b5aaca955eabb07ab1c5e62c28166)에서 수정됨.

**Cyfrin:** 확인함.


### 특정 작업에서 오래된 회계를 사용하여 NAV 또는 APR이 업데이트됨

**설명:** 다음 함수 목록은 NAV 또는 APR 또는 둘 다를 변경하지만, 업데이트가 스토리지에 반영되기 전에 해당 변수 중 어느 것도 업데이트되지 않습니다. 즉, 현재 가치를 반영하지 않는 오래된 값을 사용하여 회계가 업데이트됩니다.

- `Accounting::setReserveBps()`
- `Accounting::setRiskParameters()`
- `Accounting::reduceReserve()`
- `Accounting::updateAprs()`
- `Accounting::onAprChanged()`

예를 들어, 준비금을 줄일 때 `reserveNav`와 `nav`가 업데이트되지만 해당 변수는 이전에 업데이트되지 않았습니다. 즉, `reserveNav`와 `nav`에 대한 업데이트는 해당 nav가 마지막으로 업데이트된 이후의 오래된 값에 대해 수행됩니다.
- 마지막 업데이트 이후 `sUSDe <=> USDe 비율`이 변경되었을 수 있지만, 준비금이 감소된 금액을 뺴기 위해 업데이트하기 전에 해당 차이가 nav에 반영되지 않습니다.

또 다른 예는 APR을 업데이트할 때입니다:
- APR이 낮아지면 낮은 APR이 더 높은 기간에 적용되므로 SR Tranche는 보상 손실을 입게 됩니다.
- APR이 증가하면 더 높은 APR이 더 높은 기간에 적용되어 JR Tranche를 희생하면서 SR Tranche에 이익이 되므로 JR Tranche는 이익 손실을 입게 됩니다.

**영향:** 오래된 데이터로 회계를 업데이트하면 예상치 못한 결과가 발생하고 잠재적으로 이후 작업에 대한 회계가 엉망이 될 수 있습니다.

내부 회계를 업데이트하지 않고 APR을 업데이트하면 새 APR이 업데이트가 발생할 때까지 계산에 적용됩니다.
- APR이 낮아지면 낮은 APR이 더 높은 기간에 적용되므로 SR Tranche는 보상 손실을 입게 됩니다.
- APR이 증가하면 더 높은 APR이 더 높은 기간에 적용되어 JR Tranche를 희생하면서 SR Tranche에 이익이 되므로 JR Tranche는 이익 손실을 입게 됩니다.

**권장 완화 조치:** 설명 섹션에 나열된 함수에서 변경하기 전에 내부 NAV 및 APR을 업데이트하는 것을 고려하십시오.

**Strata:**
인덱스 및 APR 업데이트를 두 개의 별도 작업으로 리팩터링하고 NAV를 사용하기 전에 회계 업데이트를 트리거하며 `jrtNav` 또는 `srtNav` 변경 시 APR을 다시 계산하여 커밋 [7b2354](https://github.com/Strata-Money/contracts-tranches/commit/7b235498d15274867f74bd84495f227b4e20fa94), [c3c927](https://github.com/Strata-Money/contracts-tranches/commit/c3c927784d6f87582895a9d14ad022d5f5a5d6b8), [cc1172a](https://github.com/Strata-Money/contracts-tranches/commit/cc1172ac20c42f3c4624754a9d4c08dd1ce1634e), [90a7ad](https://github.com/Strata-Money/contracts-tranches/commit/90a7ad3106f44ee988067145aa8737fa0622e9c6), [2cfe69](https://github.com/Strata-Money/contracts-tranches/commit/2cfe69946b21f55a349fff6e67296c8fa6fcecef) 및 [674505](https://github.com/Strata-Money/contracts-tranches/commit/674505f27505f940ec5eefe868ecb389ae8c517f)에서 수정됨.

**Cyfrin:** 확인함.
시스템은 이제 다음과 같은 작업 전반에 걸쳐 대칭적인 관계를 갖습니다.
- 내부 회계는 실행 최상단에서 업데이트됩니다.
- `jrtNav` 또는 `srtNav`가 변경될 때마다 APR은 tx 종료 시 다시 계산됩니다.


### `UnstakeCooldown` 내부에서 구현 확인 없이 프록시 재사용으로 인해 오래된/취약한 로직에서 실행됨

**설명:** 컨트랙트는 해당 프록시가 현재 구현에서 생성되었는지 확인하지 않고 사용자의 `UnstakeCooldown::proxiesPool`에서 오래된 프록시를 재사용합니다. 클론의 대상 구현은 바이트코드에 영구적으로 내장되어 있으므로 소유자가 사용 가능한 `UnstakeCooldown::setImplementations`를 사용하여 `implementations[token]`을 업데이트하면 사용자의 풀에 이미 있는 모든 프록시는 여전히 이전 구현에 위임합니다.

**영향:** 사용자는 소유자가 `implementations`를 업데이트한 후에도 오래되거나 취약한 구현을 통해 계속 운영할 수 있습니다. 이로 인해 다음과 같은 문제가 발생할 수 있습니다:
- 요청 간 일관되지 않은 동작(일부 프록시는 새 구현을 사용하고 다른 프록시는 이전 구현을 사용).
- 이전 구현에 버그나 취약점이 포함된 경우 보안 위험.
- 이전 구현과 새 구현이 호환되지 않는 경우 회계 또는 로직 불일치.

**개념 증명 (Proof of Concept):**
1. 소유자가 `implementations[token] = ImplV1`을 설정합니다.
2. Alice가 두 번의 전송을 수행하여 `ImplV1`을 가리키는 두 개의 프록시를 생성합니다.
3. 소유자가 나중에 `setImplementations(token, ImplV2)`를 호출합니다.
4. 시간이 지나고 `ImplV1`을 가리키는 두 개의 프록시를 사용할 수 있게 됩니다.
5. Alice가 또 다른 전송을 수행합니다. 컨트랙트는 그녀의 풀에서 프록시를 꺼내 재사용합니다.
6. `implementations[token]`이 이제 `ImplV2`임에도 불구하고 해당 프록시는 여전히 `ImplV1`에 위임합니다.

**권장 완화 조치:** 프록시를 재사용할 때 프록시의 구현이 현재 `implementations[token]`과 일치하는지 확인하십시오. 그렇지 않으면 이전 프록시를 폐기하고 새 프록시를 생성하십시오.

**Strata:**
토큰에 대한 현재 구현이 재사용되는 프록시를 생성하는 데 사용된 구현과 다른지 확인하는 유효성 검사를 구현하여 커밋 [ffbedf48d](https://github.com/Strata-Money/contracts-tranches/commit/ffbedf48d268f2617e189cddc1daa167220082b3)에서 수정됨. 그렇다면 새 구현으로 새 프록시가 만들어지고 이전 프록시는 폐기됩니다.

**Cyfrin:** 확인함.


### 시니어의 TargetGain이 음수일 때, 시니어 손실이 주니어 트랜치에 이익으로 회계 처리되지 않아 nav 합계가 현재 nav와 일치하지 않게 되어 tx가 되돌려짐

**설명:** `srtGainTarget < 0`일 때 코드는 `jrtNavT1 = jrtNavT0 + gain_dTAbs`를 초기화한 *후* 시니어 손실을 `jrtNavT0`으로 전송합니다. 이것은 주니어 "이익"을 `jrtNavT1`로 전파하지 못해 변경되지 않은 상태로 둡니다.

```solidity
 if (srtGainTarget < 0) {
            // Should never happen, jic: transfer the loss to Juniors as profit
            uint256 loss = uint256(-srtGainTarget);
            uint256 srtLoss = Math.min(srtNavT0, loss);

            srtNavT0 -= srtLoss;
//@audit-issue => Updates jrtNavT0 instead of T1
@>          jrtNavT0 += srtLoss;
            srtGainTarget = 0;
        }
        uint256 srtGainTargetAbs = Math.min(
            uint256(srtGainTarget),
            Math.saturatingSub(jrtNavT1, 1e18)
        );

          // [*Users can get their withdrawal active requests DoSed by malicious users*](#users-can-get-their-withdrawal-active-requests-dosed-by-malicious-users) Final new Jrt
        jrtNavT1 = jrtNavT1 - srtGainTargetAbs;
        // [*Withdrawers of `sUSDe` always incur a loss because parameters passed from `Tranche::_withdraw` to `CDO::withdraw` are inverted*](#withdrawers-of-susde-always-incur-a-loss-because-parameters-passed-from-tranchewithdraw-to-cdowithdraw-are-inverted) Final new Srt
        srtNavT1 = srtNavT0 + srtGainTargetAbs;

//@audit-issue => sum of navs won't match because the senior loss is missing on jrtNavT1
        if (navT1 != (jrtNavT1 + srtNavT1 + reserveNavT1)) {
            revert InvalidNavSpit(navT1, jrtNavT1, srtNavT1, reserveNavT1);
        }
```
**영향:** `srtGainTarget`이 `srtNavT0`에서 할인되었지만 `jrtNavT1`에 회계 처리되지 않았기 때문에 Tx는 오류 `InvalidNavSpit()`과 함께 되돌려집니다.

**개념 증명 (Proof of Concept):** **권장 완화 조치:**
T0 대신 `jrtNavT1`을 업데이트하도록 하십시오.
```diff
 if (srtGainTarget < 0) {
            // Should never happen, jic: transfer the loss to Juniors as profit
            uint256 loss = uint256(-srtGainTarget);
            uint256 srtLoss = Math.min(srtNavT0, loss);

            srtNavT0 -= srtLoss;
-           jrtNavT0 += srtLoss;
+           jrtNavT1 += srtLoss;
            srtGainTarget = 0;
        }
        ...
```

**Strata:**
`jrtNavT0` 대신 올바른 변수 `jrtNavT1`을 업데이트하여 커밋 [5332b3](https://github.com/Strata-Money/contracts-tranches/commit/5332b383b10d1762d0413d98c3b62c1e720ac051)에서 수정됨.

**Cyfrin:** 확인함.


### 주니어 트랜치에 대한 `Tranche::maxMint`는 `jrNav`가 `JR_Shares`에 대해 `1:1` 비율 아래로 떨어질 때 오버플로우 위험이 있음

**설명:** 시스템이 두 개의 트랜치(Senior 및 Junior)로 구성되어 있고 두 트랜치 간에 예치된 모든 자산이 전략에서 함께 폴링된다는 점을 감안할 때, 각 트랜치에 대한 실제 `totalAssets()`는 해당 NAV(`srtNav` 및 `jrtNav`)를 사용하여 계산됩니다.
주니어 트랜치는 다음과 같은 용도로 사용될 수 있는 특이점이 있습니다.
1. 생성된 APR이 충분하지 않을 때 시니어의 목표 APR에 자금을 지원합니다.
2. 타격을 먼저 받고 가능한 한 많이 커버하여 시니어의 손실을 제한/감소시켜 손실을 커버합니다.

위의 두 가지 이벤트 중 하나가 발생하면 `jrtNav`가 감소하며, 이는 주니어 트랜치에 대한 `totalAssets()`가 감소하는 것으로 변환됩니다. 결과적으로 JR Shares 대 자산이 감소하고 `share<=>assets` 비율이 1:1 아래로 떨어질 수 있습니다.

`JR_Shares<=>assets`가 1:1 아래로 떨어지면, 지분을 자산으로 변환하기 위해 구현된 기본 수학 메서드와 변환될 `assets`가 `type(uint256).max`로 설정된다는 사실 때문에 `maxMint()`가 오버플로우를 발생시킵니다.

**영향:** 지분을 자산으로 변환할 때 오버플로우로 인해 `Tranche::maxMint`가 되돌려지므로 `Tranche::mint`의 DoS가 발생합니다.

**개념 증명 (Proof of Concept):** 다음 PoC에서 증명된 바와 같이, JR_Shares<=>assets 비율이 1:1 아래로 떨어지면 `Tranche::maxMint`에 대한 모든 호출이 오버플로우를 발생시켜 사실상 tx를 되돌립니다.

`CDO.t.sol` 테스트 파일에 다음 PoC를 추가하십시오.

```solidity
    function test_MaxMintOverflowsInJrTranche() public {
        address alice = makeAddr("Alice");

        uint256 initialDeposit = 1000 ether;
        USDe.mint(alice, initialDeposit);

        vm.startPrank(alice);
        USDe.approve(address(jrtVault), type(uint256).max);
        jrtVault.deposit(initialDeposit, alice);
        vm.stopPrank();

//@audit-info => Simulate 10% losses on the Jr Strategy
//@audit => This would be akin to JR Tranche covering losses or making Senior's APR whole.
        vm.prank(address(sUSDeStrategy));
        sUSDe.transfer(alice, initialDeposit / 10);

        vm.expectRevert();
        jrtVault.maxMint(alice);
    }
```

**권장 완화 조치:** Jr 트랜치에 대한 최대 예치금이 무제한이라는 점을 감안할 때 무제한 최대 지분을 반환하는 것도 괜찮습니다.
- `CDO::maxDeposit`이 `type(uint256).max`를 반환할 때 자산을 지분으로 변환하는 것을 건너뛰고 대신 동일한 값을 반환하십시오.

```diff
// Tranche::maxMint //

    function maxMint(address owner) public view override returns (uint256) {
        uint256 assets = cdo.maxDeposit(address(this));
+       if (assets == type(uint256).max) {
+          return type(uint256).max;
+       }
        return convertToShares(assets);
    }


```

**Strata:**
`type(uint256).max`를 지분으로 변환하지 않고 대신 해당 값을 JR Tranche에 대해 발행할 최대 지분으로 반환하여 커밋 [5748b2f](https://github.com/Strata-Money/contracts-tranches/commit/5748b2f292ae3f56335361633d77c9bb30e4d7fa)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage
## 낮은 위험 (Low Risk)


### `Accounting::setMinimumJrtSrtRatio`가 `minimumJrtSrtRatio` 대신 `reserveBps`를 설정하여 비율 구성을 불가능하게 함

**설명:** `Accounting::setMinimumJrtSrtRatio` 함수에는 잘못된 상태 변수를 수정하는 구현 오류가 포함되어 있습니다. `minimumJrtSrtRatio`를 설정하는 대신 함수는 잘못하여 `reserveBps`를 설정하여 최소 주니어 대 시니어 트랜치 비율을 구성할 수 없게 만듭니다.

```solidity
    function setMinimumJrtSrtRatio (uint256 bps) external onlyOwner {
        require(bps <= RESERVE_BPS_MAX, "ReserveBpsMax");
        reserveBps = bps;
        emit ReservePercentageChanged(reserveBps);
    }
```

**영향:** 불가능한 위험 매개변수 구성: `minimumJrtSrtRatio` 변수는 초기화 중에만 설정할 수 있으며(현재 5%로 하드코딩됨) 그 후에는 업데이트할 수 없어 적절한 위험 관리 조정을 방지합니다.

우발적인 준비금 구성: 트랜치 비율을 조정할 생각으로 `setMinimumJrtSrtRatio`를 호출하면 대신 준비금 비율이 수정되어 의도하지 않은 준비금 할당 변경이 발생합니다.

**권장 완화 조치:** `Accounting` 컨트랙트 내에서 다음 변경 사항을 수행하십시오.

```diff

++event MinimumJrtSrtRatioChanged(uint256 minimumJrtSrtRatio);

function setMinimumJrtSrtRatio (uint256 bps) external onlyOwner {
-   require(bps <= RESERVE_BPS_MAX, "ReserveBpsMax");
-   reserveBps = bps;
-   emit ReservePercentageChanged(reserveBps);
+   require(bps <= 1e18, "InvalidRatio"); // Max 100%
+   minimumJrtSrtRatio = bps;
+   emit MinimumJrtSrtRatioChanged(minimumJrtSrtRatio);
}
```
**Strata:**
올바른 변수 `minimumJrtSrtRatio`를 업데이트하여 커밋 [657bde](https://github.com/Strata-Money/contracts-tranches/commit/657bdef3fbb1caa0e90c1d28455523c3f5c94bbf)에서 수정됨.

**Cyfrin:** 확인함.


### `AprPairFeed`와 `Accounting` 간의 일관성 없는 APR 경계 검증

**설명:** `AprPairFeed`와 `Accounting` 컨트랙트의 APR 경계 검증 상수 간에 불일치가 있습니다. `AprPairFeed`는 -50%까지의 음수 APR을 허용하지만, `Accounting` 컨트랙트는 모든 음수 APR 값을 거부하므로 오라클에서 유효하다고 간주된 데이터가 정규화 중에 거부됩니다.

```solidity

// AprPairFeed
int64 private constant APR_BOUNDARY_MAX =    2e12; // 200%
int64 private constant APR_BOUNDARY_MIN = -0.5e12; // -50%

     /// @dev Validates that the given APR is within acceptable bounds
    function ensureValid(int64 answer) internal pure {
        require(
            APR_BOUNDARY_MIN <= answer && answer <= APR_BOUNDARY_MAX,
            "INVALID_APR"
        );


// Accounting
int64   private constant APR_BOUNDARY_MAX = 200e12;
int64   private constant APR_BOUNDARY_MIN = 0;

    function normalizeAprFromFeed (/* SD7x12 */ int64 apr) internal pure returns (UD60x18) {
        require(
            APR_BOUNDARY_MIN <= apr && apr <= APR_BOUNDARY_MAX,
            "invalid apr"
        );
```

**영향:** 프로토콜 DoS: 피드에 유효한 음수 APR 데이터(-50%에서 0% 사이)가 포함된 경우 Accounting.normalizeAprFromFeed() 함수가 되돌려져 다음을 방지합니다.

- `updateAprs()`를 통한 APR 업데이트;
- `updateIndexes()`에서의 인덱스 계산;
- 예금/인출 흐름 중 적절한 회계 업데이트;

**권장 완화 조치:** 두 컨트랙트를 정렬하십시오.

```diff
// Accounting.sol
- int64   private constant APR_BOUNDARY_MIN = 0;
+ int64   private constant APR_BOUNDARY_MIN = -0.5e12; // -50%
```

**Strata:**
음수가 되지 않도록 `Accounting`에 APR 범위를 적용하는 대신, 피드에서 보고된 음수 APR을 고려하고 시니어는 음수 APR을 갖지 않도록 하여 커밋 [c80308](https://github.com/Strata-Money/contracts-tranches/commit/c803089861f92533468ee5a31852e7cebaf49e8f)에서 수정됨.

**Cyfrin:** 확인함.


### `Accounting`의 일관성 없는 위험 프리미엄 검증으로 인해 미래 언더플로우 또는 0 APR 허용

**설명:** `Accounting::calculateRiskPremium`은 `risk = riskX + riskY * pow(tvlRatio, riskK)`를 계산합니다.
`Accounting::setRiskParameters`는 현재 TVL을 사용하여 매개변수를 업데이트한 직후 `risk < 1e18`(즉, 100% 미만)인지 확인합니다. 그러나 TVL은 나중에 변경될 수 있으므로 `risk`는 `>= 1e18`이 될 수 있습니다. `Accounting::updateIndexes`에서 `UD60x18.wrap(1e18) - risk` 식은 `risk == 1e18`인 경우 `0`을 산출하고( `aprSrt1`을 0으로 만듦) `risk > 1e18`인 경우 언더플로우로 인해 되돌려집니다. 이는 함수 간의 불일치를 생성하고 런타임에 의도하지 않은 0 또는 되돌림을 생성할 수 있습니다.


**영향:**
- TVL 변경 후 `risk > 1e18`이면 `Accounting::updateIndexes`가 언더플로우로 되돌려져 회계 업데이트, 트랜치 예금/인출 및 NAV 계산을 차단합니다.
- `risk == 1e18`이면 `aprSrt1`이 0이 되어 잠재적으로 시니어 APR(`aprSrt`)을 낮은 값으로 설정하여 잘못된 NAV 분할 및 시니어에 대한 수익 없음을 초래합니다.

**권장 완화 조치:** `Accounting::updateIndexes`에서 `risk`를 제한하거나 `risk >= 1e18`인 경우 명시적으로 되돌리십시오.


**Strata:**
현재 TVLsrt를 사용하는 대신 최대 TVLsrt 비율: 1로 위험을 계산하여 커밋 [151661](https://github.com/Strata-Money/contracts-tranches/commit/15166175a98837a26cb2d7fa818504fe21a2e788#diff-a2568622a4f3086b894ddad2f673e1f98ab5cf2f6ab7110ac3bb75fa0331b1f4R376-R379)에서 수정됨.

**Cyfrin:** 확인함.


### 주니어 트랜치 인출을 차단하기 위한 프론트러닝

**설명:** `Accounting.sol`에서 주니어 트랜치의 `maxWithdraw`는 인출 후 주니어 NAV(`jrtNav`)가 최소 `srtNav * minimumJrtSrtRatio / 1e18`(기본값 5%)을 유지하도록 제한됩니다. 이 확인은 스토리지의 현재(업데이트 후) NAV를 사용하며 `Tranche.sol`의 인출(`withdraw` 함수를 통해) 중에 호출됩니다.

공격자는 멤풀에서 피해자의 주니어 인출 트랜잭션을 프론트런할 수 있습니다.
- 공격자는 계산된 금액을 시니어 트랜치(SRT)에 예치하여 `updateBalanceFlow`(예치 중 내부적으로 호출됨)를 통해 `srtNav`를 부풀립니다.
- 이로 인해 `minJrt` 임계값이 올라가 피해자의 인출이 `maxWithdraw` 확인에 실패하고 되돌려집니다.

최대 금액을 예치할 필요는 없으며 이 조건을 되돌리기에 충분한 양만 예치하면 됩니다.
```solidity
uint256 maxAssets = maxWithdraw(owner);
if (baseAssets > maxAssets) {
            revert ERC4626ExceededMaxWithdraw(owner, baseAssets, maxAssets);
        }
```
**영향:**
- **인출 DoS**: 피해자의 합법적인 주니어 인출이 차단되어 유동성이 갇힙니다. PoC에서 공격이 `srtNav`를 부풀려 최대값을 0으로 설정한 후 40e18 인출이 실패합니다.

**개념 증명 (Proof of Concept):** 다음 Foundry 테스트(`test/CDO.t.sol`의 `test_FrontrunJRTWithdrawal`)

```solidity
function test_FrontrunJRTWithdrawal() public {
    // Setup initial state: Mint and deposit to JRT and SRT
    address victim = address(0x1234);
    address attacker = address(0x5678);
    address initialDepositor = address(0x9999);

    uint256 initialJRTDeposit = 100 ether; // jrtNav ≈ 100 ether
    uint256 initialSRTDeposit = 1000 ether; // srtNav ≈ 1000 ether
    uint256 victimWithdrawalAmount = 40 ether; // Should be valid pre-attack
    uint256 attackDepositAmount = 1000 ether; // Enough to push JRT maxWithdraw to 0

    // Victim deposits to JRT
    vm.startPrank(victim);
    USDe.mint(victim, initialJRTDeposit);
    USDe.approve(address(jrtVault), initialJRTDeposit);
    jrtVault.deposit(initialJRTDeposit, victim);
    vm.stopPrank();

    // Initial depositor to SRT (could be anyone)
    vm.startPrank(initialDepositor);
    USDe.mint(initialDepositor, initialSRTDeposit);
    USDe.approve(address(srtVault), initialSRTDeposit);
    srtVault.deposit(initialSRTDeposit, initialDepositor);
    vm.stopPrank();

    // Verify initial state: JRT maxWithdraw should allow the withdrawal
    uint256 preMaxWithdraw = accounting.maxWithdraw(true); // isJrt=true
    assertGt(preMaxWithdraw, victimWithdrawalAmount, "Initial maxWithdraw too low");

    // Simulate attacker frontrunning with large SRT deposit
    vm.startPrank(attacker);
    USDe.mint(attacker, attackDepositAmount);
    USDe.approve(address(srtVault), attackDepositAmount);
    srtVault.deposit(attackDepositAmount, attacker);
    vm.stopPrank();

    // Now simulate victim's withdrawal attempt (should fail due to updated cap)
    vm.startPrank(victim);
    vm.expectRevert(
        abi.encodeWithSelector(
            ERC4626ExceededMaxWithdraw.selector,
            victim,
            victimWithdrawalAmount,
            0 // Post-attack maxWithdraw should be 0
        )
    );
    jrtVault.withdraw(victimWithdrawalAmount, victim, victim);
    vm.stopPrank();

    // Verify post-attack state
    uint256 postMaxWithdraw = accounting.maxWithdraw(true);
    assertEq(postMaxWithdraw, 0, "Post-attack maxWithdraw not zeroed");
}
```

**재현 단계**:
1. Foundry에서 테스트 실행: `forge test --match-test test_FrontrunJRTWithdrawal`.

**Strata:**
SR 예치금을 증가시켜 `minimumJrtSrtRatio`에 도달하는 것을 방지하기 위해 SRTranche 예치금에 대한 소프트 제한을 구현하여 커밋 [23777d](https://github.com/Strata-Money/contracts-tranches/commit/23777d58ff5aa3c2dcb640d21902aa53e5d29212)에서 수정됨.

**Cyfrin:** 확인함.


### 모든 컨트랙트에서 `fallback`을 호출할 수 있는 권한을 부여할 때 계정에 실수로 `DEFAULT_ADMIN_ROLE`이 부여될 수 있음

**설명:** `AccessControlManager`는 계정에 대한 역할과 권한을 관리하여 사용자가 어떤 컨트랙트에서 무엇을 호출할 수 있는지 결정합니다.

권한은 선택기 수준에서 부여되며 한 번에 하나의 컨트랙트에만 권한을 부여하거나 계정이 모든 컨트랙트에서 동일한 선택기를 호출하도록 권한을 부여할 수 있습니다.

모든 계정(address(0))에서 선택기가 `bytes4(0)`인 `fallback()` 함수를 호출하도록 계정을 허용할 때 엣지 케이스가 있습니다.
- 이 두 입력의 조합은 DEFAULT_ADMIN_ROLE에 할당된 정확한 값인 bytes(0)으로 역할을 계산하는 결과를 낳습니다.

```solidity
    function grantCall(address contractAddress, bytes4 sel, address accountToPermit) public {
//@audit-issue => The calculated role for `address(0)` and `bytes4(0)` is bytes32(0)`
        bytes32 role = roleFor(contractAddress, sel);
//@audit-issue => Granting bytes32(0) to the account results in granting the DEFAULT_ADMIN_ROLE
        grantRole(role, accountToPermit);
        emit PermissionGranted(accountToPermit, contractAddress, sel);
    }

    function roleFor(address contractAddress, bytes4 sel) internal pure returns (bytes32 role) {
//@audit-issue => The calculated role for `address(0)` and `bytes4(0)` is bytes32(0)`
        role = (bytes32(uint256(uint160(contractAddress))) << 96) | bytes32(uint256(uint32(sel)));
    }

```

**영향:** 사용자는 실수로 DEFAULT_ADMIN_ROLE을 부여받을 수 있으며, 이를 사용하여 다른 사용자가 제한된 함수를 호출하도록 승인할 수 있습니다.

**개념 증명 (Proof of Concept):** `CDO.t.sol` 테스트 파일에 다음 PoC를 추가하십시오.
1. Alice는 모든 컨트랙트에서 fallback()을 호출할 수 있는 권한을 받으려다 실수로 DEFAULT_ADMIN_ROLE을 받습니다.
2. Alice가 DEFAULT_ADMIN_ROLE을 갖게 되면 다른 컨트랙트에서 다른 함수를 호출하도록 Bob을 승인합니다.

```solidity
    function test_grantsDefaultAdminByMisstake() public {
        bytes32 DEFAULT_ADMIN_ROLE = acm.DEFAULT_ADMIN_ROLE();

        address contractAddress = address(0);
        bytes4 selector = bytes4(0);

        address alice = makeAddr("Alice");

        assertFalse(acm.hasRole(DEFAULT_ADMIN_ROLE, alice));

        //@audit-issue => Granting permission to alice to call fallback function on any contract results on granting Alice the DEFAULT_ADMIN_ROLE
        acm.grantCall(contractAddress, selector, alice);
        assertTrue(acm.hasRole(DEFAULT_ADMIN_ROLE, alice));

        address contractA = makeAddr("contractA");
        address bob = makeAddr("bob");
        bytes4 withdrawSelector = bytes4(keccak256(bytes("withdraw(address,uint256)")));

        assertFalse(acm.hasPermission(bob, contractA, withdrawSelector));

        //@audit-info => Alice w/ DEFAULT_ADMIN can grant permissions to other accounts
        vm.startPrank(alice);
        acm.grantCall(contractA, withdrawSelector, bob);
        assertTrue(acm.hasPermission(bob, contractA, withdrawSelector));
    }

```

**권장 완화 조치:** 계산된 역할이 DEFAULT_ADMIN_ROLE이 아닌지 확인하십시오. 그렇지 않으면 tx를 되돌리십시오.

```diff
    function grantCall(address contractAddress, bytes4 sel, address accountToPermit) public {
        bytes32 role = roleFor(contractAddress, sel);
+       require(role != DEFAULT_ADMIN_ROLE, "Granting DEFAULT_ADMIN_ROLE");
        grantRole(role, accountToPermit);
        emit PermissionGranted(accountToPermit, contractAddress, sel);
    }
```

**Strata:**
`contractAddress`가 `address(0)`이거나 `selector`가 `bytes(0)`일 때 되돌리는 확인을 추가하여 커밋 [e6ad2d](https://github.com/Strata-Money/contracts-tranches/commit/e6ad2d59f1ad9abab0a9af685aeea8d510e9169d)에서 수정됨.

**Cyfrin:** 확인함. 새로운 변경으로 인해 실수로 DEFAULT_ADMIN_ROLE을 할당하는 것을 방지합니다.
권한은 컨트랙트별 선택기 기준으로 부여됩니다.


\clearpage
## 정보 (Informational)


### `Accounting`에서 `minimumJrtSrtRatio`에 대한 잘못된 주석 및 누락된 하한

**설명:**
```solidity
/// @dev minimum TVL ratio: TVLjrt/TVLsrt, e.g. >= 0.05%
```
```solidity
minimumJrtSrtRatio = 0.05e18;
```
주석에는 "0.05%"(0.0005e18)라고 되어 있지만 값은 5%(0.05e18)입니다. 그리고 값이 0.05% 미만으로 설정되는 것을 방지하는 확인이 없습니다.
5% 비율로 시작하려는 의도일 수 있습니다.



**권장 완화 조치:**
- 의도된 5%를 반영하도록 주석을 업데이트하십시오(예: ">= 5%").
- `Accounting::setMinimumJrtSrtRatio`에서 최소 경계에 대해 `require(bps >= 0.0005e18, "RatioTooLow");`를 추가하십시오.

**Strata:**
커밋 [eefd73](https://github.com/Strata-Money/contracts-tranches/commit/eefd73cfee7783cb45b19e0763d83ba2fb0084af) 및 [c1afee2](https://github.com/Strata-Money/contracts-tranches/commit/c1afee2f0c14531ddbe88f81d4aa4f3325e87fd1)에서 수정됨. 주석을 업데이트하고 `minimumJrtSrtRatio`에 대한 하한을 검증하는 확인을 추가했습니다.

**Cyfrin:** 확인함.


### `Accounting.sol`에서 사용되지 않는 `OwnableUpgradeable` 가져오기


**설명:** `OwnableUpgradeable`은 가져오지만 이 파일에서 직접 사용되지 않습니다.

**권장 완화 조치:** 코드 청결성을 개선하기 위해 사용되지 않는 가져오기를 제거하십시오.

```diff
- import { OwnableUpgradeable} from "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";
```


**Strata:**
커밋 [cfc5117b](https://github.com/Strata-Money/contracts-tranches/commit/cfc5117b07d639d95319153b82c299972dfdedd9)에서 수정됨.

**Cyfrin:** 확인함.


### `AprPairFeed::latestRoundData`에서 폴백 APR 값 검증 누락

**설명:** `AprPairFeed::latestRoundData`는 선호하는 소스(피드 또는 전략 제공자)에서 APR을 가져옵니다. 피드가 오래된 경우 `provider.getAprPair()`를 통해 전략 제공자로 폴백하지만 피드 경로의 잠재적 유효성 검사와 달리 반환된 값(예: `ensureValidAprs` 또는 경계 검사를 통해)을 유효성 검사하지 않습니다. 이는 잠재적으로 유효하지 않은 값이 사용될 수 있게 합니다.

```solidity
function latestRoundData() external view returns (TRound memory) {
        TRound memory round = latestRound;

        if (sourcePref == ESourcePref.Feed) {
            uint256 deltaT = block.timestamp - uint256(round.updatedAt);
            if (deltaT < roundStaleAfter) {
                return round;
            }
            // falls back to strategy ↓
        }

        (int64 aprTarget, int64 aprBase, uint64 t1) = provider.getAprPair();
        return TRound({
            aprTarget: aprTarget,
            aprBase: aprBase,
            updatedAt: t1,
            answeredInRound: latestRoundId + 1
        });
    }
```


**권장 완화 조치:** 피드 경계와 유사하게 폴백 가져오기 후 유효성 검사를 추가하십시오.

```diff
        (int64 aprTarget, int64 aprBase, uint64 t1) = provider.getAprPair();
+       // Add validation, e.g.:
+       ensureValid(aprTarget);
+       ensureValid(aprBase);
        return TRound({
            aprTarget: aprTarget,
            aprBase: aprBase,
            updatedAt: t1,
            answeredInRound: latestRoundId + 1
        });
```

**Strata:**
`aprTarget` 및 `aprBase`를 검증하여 커밋 [1c4009a](https://github.com/Strata-Money/contracts-tranches/commit/1c4009a61f6aa1802b0a1541c6e63b096d601d1b)에서 수정됨.

**Cyfrin:** 확인함.


### `UnstakeCooldown::transfer`에서 즉시 언스테이크에 대한 `Unstaked` 이벤트 누락

**설명:** `UnstakeCooldown::transfer`가 즉시 언스테이크를 트리거할 때(즉, 핸들러의 `proxy.request()` 호출이 `unlockAt <= block.timestamp`를 반환함), 함수는 프록시를 풀에 반환한 후 일찍 반환하지만 즉시 완료에 대한 `Unstaked` 이벤트를 방출하지 않습니다. 결과적으로 상환이 즉시 발생한 흐름(쿨다운 없음)에 대해 온체인 이벤트가 누락됩니다.

**영향:** 인출 추적을 위해 이벤트에 의존하는 오프체인 시스템은 이러한 즉시 언스테이크 작업을 놓칠 수 있습니다. 이는 온체인 잔액이나 보안에 영향을 미치지 않으며 사용자는 여전히 자금을 올바르게 받습니다.

**권장 완화 조치:** `transfer()`에 전달된 `amount` 매개변수를 사용하여 `transfer()` 즉시 분기에서 `Unstaked`(또는 새 `ImmediateUnstake`) 이벤트를 방출하십시오.

**Strata:**
두 쿨다운 컨트랙트 모두에서 방출되는 이벤트를 `Finalized`로 통합하여 커밋 [c08784](https://github.com/Strata-Money/contracts-tranches/commit/c087849442937e8465f342f9ec6dc82ac4897d0a)에서 수정됨.

**Cyfrin:** 확인함.


### `uint` 대신 명시적 부호 없는 정수 크기 조정 사용

**설명:** Solidity에서 `uint`는 자동으로 `uint256`에 매핑되지만 변수를 선언할 때 정확한 크기를 지정하는 것이 좋은 관행으로 간주됩니다.

```
Accounting.sol
200:    ) public view returns (uint jrtNavT1, uint srtNavT1, uint reserveNavT1) {

StrataCDO.sol
86:     uint totalAssetsOverall = strategy.totalAssets();
128:    uint jrtAssetsIn = isJrt_ ? baseAssets : 0;
129:    uint srtAssetsIn = isJrt_ ? 0          : baseAssets;
143:    uint jrtAssetsOut = isJrt_ ? baseAssets : 0;
144:    uint srtAssetsOut = isJrt_ ? 0          : baseAssets;
```


**Strata:**
커밋 [506c4c](https://github.com/Strata-Money/contracts-tranches/commit/506c4c744dc6dea538bb8b69dade114bee1aeb5e)에서 수정됨.

**Cyfrin:** 확인함.


### `sUSDeStrategy::reduceReserve` 내부의 도달할 수 없는 코드

**설명:** `sUSDeStrategy::reduceReserve` 내부의 마지막 줄: `revert UnsupportedToken(token);`은 도달할 수 없습니다.

`sUSDeStrategy::reduceReserve`는 `StrataCDO::reduceReserve`를 통해서만 호출될 수 있으며, 이는 `strategy.reduceReserve(token, tokenAmount, treasury);`를 호출하기 전에 `uint256 baseAssets = strategy.convertToAssets(token, tokenAmount, Math.Rounding.Floor);`를 호출합니다.

`sUSDeStrategy::convertToAssets`는 지원되지 않는 토큰에 대해 이미 되돌립니다.

```
    function convertToAssets (address token, uint256 tokenAmount, Math.Rounding rounding) external view returns (uint256) {
        if (token == address(sUSDe)) { // if sUSDe, use previewRedeem or previewMint
            return rounding == Math.Rounding.Floor
                ? sUSDe.previewRedeem(tokenAmount) // aka convertToAssets(tokenAmount)
                : sUSDe.previewMint(tokenAmount); // aka convertToAssetsCeil(tokenAmount)
        }
        if (token == address(USDe)) { // if USDe, return the input amount
            return tokenAmount;
        }
        revert UnsupportedToken(token);
    }
```

**Strata:**
인지함; 현재 동작을 유지하는 것을 선호합니다. 현재 조건에서 이 revert가 도달할 수 없다는 것은 전적으로 옳지만, 다른 메서드와 일관성을 유지하기 위해 Strategy는 토큰에 대한 모든 작업이 지원되는지 확인해야 합니다.

**Cyfrin:** 인지함.


### `Tranche`에 대한 `asset`을 설정하기 위한 오해의 소지가 있는 변수 이름

**설명:** `Tranche`가 작업할 `asset`을 설정하는 데 사용되는 변수의 이름은 `stakedAsset`입니다.
시스템이 작동하는 두 자산은 `USDe`와 `sUSDe`이기 때문에 이는 오해의 소지가 있습니다. ' sUSDe는 `USDe`의 스테이킹된 버전이므로 변수 이름은 사람들이 `Tranche`의 `asset`이 `USDe` 대신 `sUSDe`가 될 것으로 생각하도록 오도할 수 있습니다.

**권장 완화 조치:** 변수 이름을 `baseAsset` 또는 이름에 `stake`라는 단어가 없는 다른 이름으로 변경하십시오.

**Strata:**
커밋 [2d7f5a17](https://github.com/Strata-Money/contracts-tranches/commit/2d7f5a17e0ac1545506530e8ed85cad828f392f6)에서 수정됨.

**Cyfrin:** 확인함.


### 오타 및 나쁜 NATSPEC

**설명:** 다음 오타/NATSPEC 문제를 확인했습니다.

```
AccessControlled.sol
29: `NewAccessControlManager` should be `NewAccessControllManager`

Accounting.sol:
66: `InvalidNavSpit` should be `InvalidNavSplit`

ERC20Cooldown.sol:
10: `allows to store` should be `allows storing`

StrataCDO.sol:
42: `Sinior` should be `Senior`
62: `@notice` is empty

UnstakeCooldown.sol:
144: `for a tokens` should be `for tokens`
```

**Strata:**
오타를 수정하고 Natspec을 업데이트하여 커밋 [506c4c](https://github.com/Strata-Money/contracts-tranches/commit/506c4c744dc6dea538bb8b69dade114bee1aeb5e)에서 수정됨.

**Cyfrin:** 확인함.


### `AprPairFeed::getRoundData`가 지정된 것과 다른 라운드의 데이터를 반환할 수 있음

**설명:** `AprPairFeed`에서 라운드 데이터를 업데이트할 때 데이터는 `rounds` 매핑에 저장되며, 이는 계산된 `roundIdx`를 통해 액세스됩니다. `roundIdx`가 파생되는 방식으로 인해 0에서 20까지 매 20번의 업데이트마다 반복됩니다. 이는 오래된 `roundIdx`가 결국 새로운 것과 동일한 `roundIdx`를 계산하게 됨을 의미합니다. 이로 인해 특정 roundId에 대한 데이터를 검색할 때 `AprPairFeed::getRoundData`가 더 새로운 roundId에 대한 데이터를 반환하는 문제가 발생할 수 있습니다.

```solidity
    function updateRoundDataInner(int64 aprTarget, int64 aprBase, uint64 t) internal {
        ...
        uint64 roundId = (latestRoundId + 1);
@>      uint64 roundIdx = roundId % roundsCap;

        latestRoundId = roundId;
        latestRound = TRound({
            aprTarget: aprTarget,
            aprBase: aprBase,
            updatedAt: t,
            answeredInRound: roundId
        });
@>      rounds[roundIdx] = latestRound;

        emit AnswerUpdated(aprTarget, aprBase, roundId, t);
    }

    function getRoundData(uint64 roundId) public view returns (TRound memory) {
@>      uint64 roundIdx = roundId % roundsCap;
        TRound memory round = rounds[roundIdx];
        require(round.updatedAt > 0, "No data present");
@>      return round;
    }

```

**권장 완화 조치:** `rounds` 매핑에서 읽은 데이터가 지정된 roundId와 일치하는지 확인하는 것을 고려하십시오.

```diff
    function getRoundData(uint64 roundId) public view returns (TRound memory) {
        uint64 roundIdx = roundId % roundsCap;
        TRound memory round = rounds[roundIdx];
        require(round.updatedAt > 0, "No data present");
+       require(round.answeredInRound == roundId, "old round");
        return round;
    }
```

**Strata:**
쿼리된 데이터가 요청된 동일한 `roundId`에 해당하는지 확인하여 커밋 [233e3d](https://github.com/Strata-Money/contracts-tranches/commit/233e3d398b9bb52929170572fed69d1083ee1ce1)에서 수정됨.

**Cyfrin:** 확인함.


### 쿨다운 컨트랙트는 쿨다운 기간이 만료된 요청의 잔액만 고려하기 때문에 사용자의 실제 잔액을 과소 보고함

**설명:** 쿨다운 컨트랙트는 모든 활성 요청이 고려되지 않기 때문에 사용자의 실제 잔액을 과소 보고합니다. 쿨다운 기간이 만료된 요청만 사용자의 잔액의 일부로 간주됩니다.

이 구현은 항상 사용자의 잔액에 대한 실제 정보를 정확하게 표시하지 않으며 요청이 완료될 때까지(쿨다운 기간이 끝날 때까지)만 표시합니다.

**권장 완화 조치:** 사용 가능한 잔액과 잠긴 잔액을 모두 포함하여 모든 잔액을 반환하도록 `balanceOf()` 메서드를 리팩터링하는 것을 고려하십시오.

**Strata:**
잠금 기간을 기반으로 `pending` 및 `claimable` 금액과 같은 활성 요청에 대한 더 자세한 데이터를 반환하도록 커밋 [949cb4](https://github.com/Strata-Money/contracts-tranches/commit/949cb474579036655fc3da066d8c35e77443ffd4) 및 [1f82c6](https://github.com/Strata-Money/contracts-tranches/commit/1f82c6a456272fd40afcb8792b7b4b3d9c13da20)에서 수정됨.

**Cyfrin:** 확인함.


### `UnstakeCooldown::balance`는 잔액을 보고하는 실제 토큰과 다른 토큰 컨트랙트를 필요로 함

**설명:** `UnstakeCooldown` 컨트랙트는 `USDe`를 인출할 때 사용됩니다. 이는 시스템이 Ethena 컨트랙트에서 `sUSDe`의 쿨다운을 요청한 다음 사용자에게 전송될 기본 `USDe`를 `언스테이크`하도록 허용하기 위한 것입니다.
`sUSDe` 컨트랙트에서 쿨다운이 요청되면 사용자에게 주어질 `USDe`의 기본 양은 `cooldown.underlyingAmount`에서 추적됩니다.

따라서 `sUSDe`의 `cooldowns`는 `USDe`의 양을 반환합니다. 즉, `proxy.getPendingAmount()`를 쿼리하면 `sUSDe`가 아닌 `USDe` 양을 반환합니다.
- 즉, `UnstakeCooldown::balanceOf`는 `user`에 대한 `USDe` 양을 쿼리하는 것이지 `sUSDe`를 쿼리하는 것이 아니지만, `activeRequests` 매핑의 요청은 `USDe` 대신 `sUSDe`와 연관되었습니다. 이로 인해 사용자는 실제로는 `USDe` 양을 쿼리하고 있음에도 불구하고 토큰으로 `sUSDe`를 지정하여 `balanceOf()`를 호출해야 합니다.

```solidity
//sUSDeStrategy::withdraw//
    function withdraw (address tranche, address token, uint256 tokenAmount, uint256 baseAssets, address receiver) external onlyCDO returns (uint256) {
        ...
        if (token == address(USDe)) {
@>          unstakeCooldown.transfer(sUSDe, receiver, shares);
            return baseAssets;
        }
        revert UnsupportedToken(token);
    }

//UnstakeCooldown::transfer//
function transfer(IERC20 token, address to, uint256 amount) external {
    ...

@>  TRequest[] storage requests = activeRequests[address(token)][to];
    IUnstakeHandler[] storage proxies = proxiesPool[address(token)][to];

    ...

    requests.push(TRequest(uint64(unlockAt), proxy));
    emit Requested(address(token), to, amount, unlockAt);
}

function balanceOf (IERC20 token, address user, uint256 at) public view returns (uint256) {
@>  TRequest[] storage requests = activeRequests[address(token)][user];
    uint256 l = requests.length;
    uint256 balance = 0;
    for (uint256 i = 0; i < l; i++) {
        TRequest memory req = requests[i];
        if (req.unlockAt <= at) {
@>          balance += req.proxy.getPendingAmount();
        }
    }
    return balance;
}

//sUSDeCooldownRequestImpl//

    function getPendingAmount () external view returns (uint256 amount) {
@>      amount = sUSDe.cooldowns(address(this)).underlyingAmount;
        return amount;
    }

```


**권장 완화 조치:** 새 요청을 등록할 때 `sUSDe` 대신 `USDe`와 연결하십시오. 나머지 코드는 정상적으로 작동하며 이제 사용자는 `USDe`를 토큰으로 입력하고 `USDe` 금액을 얻습니다. 또한 보고된 잔액이 `USDe` 단위에 해당한다는 것을 문서화하는 것이 가치가 있을 수 있습니다.

**Strata:**
인지함; balanceOf에서 허용되는 토큰을 기본 토큰으로 변경하면 기본 토큰이 나중에 다른 스테이킹된 토큰에서 사용될 수 있으므로 확정 로직이 깨질 수 있습니다. 여기서는 특정 스테이킹된 자산에 집중하고 싶습니다.


### `sUSDe`에 대한 `maxDeposit`, `maxMint`, `maxRedeem`, `maxWithdraw`에 대한 헬퍼 함수 없음

**설명:** Tranche는 `USDe` 또는 `sUSDe`를 사용하여 예금을 허용하지만 작업할 수 있는 `sUSDe` 양을 계산하는 헬퍼 함수가 없습니다. 구현된 함수는 `USDe` 금액에 대해서만 작동합니다.

특히 `sUSDe` 인출/상환의 경우 사용자가 인출을 요청할 수 있는 `sUSDe` 양을 결정하는 것은 간단한 프로세스가 아닙니다.

`USDe` 또는 `sUSDe`에 대한 인출 흐름이 상당히 다르다는 사실은 주로 USDe 인출을 위한 언스테이킹 시간 창 때문입니다.

**권장 완화 조치:** 사용자가 `sUSDe`를 사용하여 작업할 때 최대값을 계산할 수 있도록 헬퍼 함수를 추가하는 것을 고려하십시오.
- 참고: 이 함수들은 코드 어디에서나 호출될 필요가 없습니다. 사용자가 오프체인에서 호출할 수 있도록 사용할 수 있어야 합니다.

**Strata:**
인지함; 사용자는 `Strategy::convertToTokens`를 사용할 수 있습니다. sUSDe에 대한 "maxWithdraw"를 얻으려면 사용자는 기본 `maxWithdraw`를 호출하여 `USDe` 금액을 얻고 `convertToTokens`를 사용하여 값을 `sUSDe`로 변환합니다. 나중에 우리는 이러한 헬퍼 함수를 추가 "Lens" 컨트랙트로 추출할 것입니다.

\clearpage
## 가스 최적화 (Gas Optimization)


### 중복 확인 제거

**설명:** 아래 함수는 해당 확인이 있든 없든 동일하게 동작합니다.

* `StrataCDO.sol`
```solidity
// isJrt
154:        if (tranche == address(0)) {
155:            revert InvalidTranche(tranche);
156:        }
```

**Strata:**
커밋 [934be5](https://github.com/Strata-Money/contracts-tranches/commit/934be516bd0e3643d7fe37f19ce62f141281d704)에서 중복 확인이 제거됨.

**Cyfrin:** 확인함.


### 호출 간에 결과가 변경될 수 없고 여러 번 사용되는 경우 외부 호출 결과 캐시

**설명:** * `Tranche.sol`
```solidity
// configure
212:        IERC20[] memory tokens = cdo.strategy().getSupportedTokens();
213:        uint256 len = tokens.length;
214:        address strategy = address(cdo.strategy());
```

**권장 완화 조치:**
```diff
++import { IStrategy } from "./interfaces/IStrategy.sol";
[…]
    function configure () external onlyCDO {
--      IERC20[] memory tokens = cdo.strategy().getSupportedTokens();
--      uint256 len = tokens.length;
--      address strategy = address(cdo.strategy());
++      address strategy = address(cdo.strategy());
++      IERC20[] memory tokens = IStrategy(strategy).getSupportedTokens();
++      uint256 len = tokens.length;
```

**Strata:**
`cdo::strategy` 호출 결과를 캐싱하고 재사용하여 커밋 [732b1a8](https://github.com/Strata-Money/contracts-tranches/commit/732b1a8ee5ae0bda763f74556f89bbb28b63f784)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage

