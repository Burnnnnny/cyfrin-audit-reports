**Lead Auditors**

[0xStalin](https://x.com/0xStalin)

**Assisting Auditors**



---

# 발견 사항 (Findings)
## 중간 위험 (Medium Risk)


### 동결된 사용자의 지급금을 압류하면 보유자가 후속 분배에서 동결 해제될 경우 이중 지출로 이어질 수 있음

**설명:** `ChildToken::seizeFrozenFunds` 함수는 다음 작업을 수행하도록 설계되었습니다:
- 합법적인 보유자가 동결 이전에 발생한 모든 배당금 분배를 청구할 수 있도록 허용합니다.
- 동결 기간 동안 해당하는 모든 배당금 분배를 압류 실행 시점에 지급된 가장 최근 분배까지 지정된 관리인(custodian)에게 전송합니다.

동결된 자금이 이미 압류되어 관리인에게 리디렉션되었음에도 불구하고, 동결된 보유자가 이후에 동결 기간에 귀속되는 배당금을 청구할 수 있는 취약점이 존재합니다.

이 문제는 다음과 같은 이벤트 순서에서 발생합니다:
1. 동결 기간 동안 하나 이상의 분배가 발생한 상태에서 동결된 계정에 대해 `ChildToken::seizeFrozenFunds`가 호출됩니다. 이는 동결 기간 배당금을 관리인에게 올바르게 리디렉션하고 압류 스냅샷을 `holder.frozenIndex` 및 관련 회계 변수에 기록합니다.
2. 압류가 발생한 후 새로운 배당금 분배가 생성됩니다.
3. 계정이 여전히 동결된 상태에서 보유자(또는 누구든)가 `DividendManager::payoutBalance`를 호출합니다. 계정이 여전히 동결 상태이므로 내부 회계 변수 `holder.lastPayoutIndexCalculated`는 `holder.frozenIndex`로 강제로 재설정됩니다.
4. 이후 계정이 동결 해제됩니다.
5. 동결 해제 후 보유자가 다시 payoutBalance를 호출합니다. 이 시점에서:
- 계정은 더 이상 동결 상태가 아닙니다.
- `holder.lastPayoutIndexCalculated`는 3단계에서 이전에 강제된 값(`holder.frozenIndex`)으로 유지됩니다.
- 따라서 지급 루틴은 `holder.frozenIndex`부터 현재 최신 분배 인덱스까지의 모든 분배를 처리하고 입금합니다. 결과적으로 보유자는 이전에 압류된 동결 기간 배당금 전체를 두 번째로 받게 되어 이중 지급(관리인이 한 번 받고 보유자가 다시 받음)이 발생합니다.

**영향:** 동결 기간 동안 이미 지급금을 압류당한 동결된 사용자가 해당 지급금을 청구할 수 있는 접근 권한을 다시 얻게 되어, 사실상 다른 보유자의 지급금을 처리하기 위해 예약된 자금을 가져가게 됩니다.

**개념 증명 (Proof of Concept):** 다음 PoC를 `DividendManager.t.sol` 테스트 파일에 추가하십시오:

```solidity
    function test_PoC_DoubleSpendingPayoutsOfFrozenHolder() public {
        // distribute payout, freeze holder, distribute more payouts while holder is frozen, seizeFrozen, distribute a payout, call to payoutBalance() to reset lastIndex to frozenIndex, then unfreeze holder, call payoutBalance() and get access to all payouts since the user was frozen!
        address user = domesticUsers[0];
        address custodian = domesticUsers[1];

        uint64 tokenToMint = 5;
        uint64 payoutAmount = 100e6;

        // mint and send user the tokens
        _mintAndTransferToUser(user, tokenToMint);

        // create distribution + verify
        _fundPayoutToPaymentSettler(payoutAmount);
        assertEq(d_childTokenProxy.payoutBalance(user), payoutAmount);

        assertEq(d_childTokenProxy.payoutBalance(user), payoutAmount);

        // freeze user + 5 payouts
        d_childTokenProxy.freezeHolder(user);
        _fundPayoutToPaymentSettler(payoutAmount);
        _fundPayoutToPaymentSettler(payoutAmount);
        _fundPayoutToPaymentSettler(payoutAmount);
        _fundPayoutToPaymentSettler(payoutAmount);
        _fundPayoutToPaymentSettler(payoutAmount);

        // seize frozen payouts from user, send frozen funds to custodian
        d_childTokenProxy.seizeFrozenFunds(user, custodian, false);
        // user receives the payouts owed prior to being frozen
        assertEq(stableCoin.balanceOf(user), payoutAmount);
        // custodian receives the payouts while the user was frozen
        assertEq(stableCoin.balanceOf(custodian), payoutAmount * 5);
        assertEq(d_childTokenProxy.payoutBalance(user), 0);
        assertEq(d_childTokenProxy.isHolderFrozen(user), true);

        // One more payout - Since the user was frozen, this is the 6th payout
        _fundPayoutToPaymentSettler(payoutAmount);

        assertEq(d_childTokenProxy.payoutBalance(user), 0);

        d_childTokenProxy.unFreezeHolder(user);

        //@audit-issue => Because `frozenIndex` was not updated, bug allows the user to claim the 6 payouts since he was frozen regardless that 5 of those 6 payouts have already been paid out to the custodian via the `seizeFrozenFunds()`
        assertEq(d_childTokenProxy.payoutBalance(user), payoutAmount * 6);

    }
```

**권장 완화 조치:** 사용자를 다시 동결할 때 `holderStatus.frozenIndex`를 `$._currentPayoutIndex`로 설정하십시오.

**Remora:** 커밋 [2969545](https://github.com/remora-projects/remora-dynamic-tokens/commit/29695454e47dc9844715c9a157d90c3fcaad736d)에서 수정됨

**Cyfrin:** 확인함. 보유자가 다시 동결된 후 `holderStatus.frozenIndex`가 `$._currentPayoutIndex`로 설정됩니다.

\clearpage
## 정보 (Informational)


### `stablecoin`이 `address(0)`으로의 전송에서 되돌려지지 않을 때 사용자가 `referralData`의 `firstPurchase` 상태를 재설정할 수 있음

**설명:** 사용자는 [`ReferralManager::createReferral`](https://github.com/remora-projects/remora-dynamic-tokens/blob/final-audit-prep/contracts/CoreContracts/ReferralManager/ReferralManager.sol#L129-L145)을 호출하여 할인을 받기 위한 추천(referral)을 생성할 수 있습니다. 사용자는 할인을 받고, 추천인(referrer)은 사용자가 첫 구매를 할 때 보너스를 받습니다.

이 시스템은 사용자에게 한 번만 할인을 제공하도록 의도되었지만, 스테이블코인이 address(0)으로의 전송을 허용할 때 엣지 케이스가 있습니다. 이는 `ReferralManager::createReferral`을 호출하고 `referrer`를 `address(0)`으로 설정하는 것을 허용합니다. 이는 사실상 사용자가 이미 추천인을 설정했는지 확인하는 검사를 우회하고 `referralData.isFirstPurchase`를 true로 설정하여 다음 구매 시 사용자에게 할인을 부여합니다. 이를 통해 사용자는 다음을 수행할 수 있습니다:
1. `referrer`를 address(0)으로 설정하여 `ReferralManager::createReferral` 호출
2. 토큰 구매
3. `referrer`를 address(0)으로 설정하여 `ReferralManager::createReferral` 다시 호출

**영향:** 사용자는 `firstPurchase` 상태를 true로 재설정하여 모든 구매에 대해 할인을 받도록 추천 시스템을 조작할 수 있습니다.

**권장 완화 조치:** 추천 생성 시 `referrer` 주소가 address(0)이 아닌지 검증하십시오.
대안으로, 이 문제를 인지하고 서명자가 `referrer`가 address(0)으로 설정된 서명을 절대 생성하지 않도록 하십시오.

**Remora:** 커밋 [20eddec](https://github.com/remora-projects/remora-dynamic-tokens/commit/20eddec6e760c7c9bd3669c250e50e562312dfff)에서 수정됨

**Cyfrin:** 확인함.

\clearpage
## 가스 최적화 (Gas Optimization)


### `ReferralManager::completeFirstPurchase`에서 `nonReentrant` 수정자의 불필요한 사용

**설명:** `ReferralManager::completeFirstPurchase`에는 `ReentrancyGuardTransientUpgradeable` 라이브러리의 `nonReentrant` 수정자가 적용되어 있지만, 이 함수는 재진입(reentrancy)에 취약하지 않습니다.
- 호출자는 `TokenBank` 컨트랙트로 제한되며, 유일한 외부 호출은 스테이블코인을 추천인에게 전송하는 것입니다.

스테이블코인이 유효한 컨트랙트로 올바르게 설정되어 있는 한, `nonReentrant` 수정자를 사용할 필요가 없습니다.

**권장 완화 조치:** `nonReentrant` 수정자는 필요하지 않습니다. `ReentrancyGuardTransientUpgradeable` 라이브러리의 import를 제거하십시오.

**Remora**
커밋 [59a33a4](https://github.com/remora-projects/remora-dynamic-tokens/commit/59a33a40bfcdc585d5a24a58108fcd4f2e583a05)에서 수정됨

**Cyfrin:** 확인함.

\clearpage

