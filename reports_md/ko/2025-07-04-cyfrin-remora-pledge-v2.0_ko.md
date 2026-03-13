**Lead Auditors**

[Dacian](https://x.com/DevDacian)

[Stalin](https://x.com/Stalin_eth)

**Assisting Auditors**

 

---

# Findings
## Critical Risk


### `PledgeManager::pledge`, `refundTokens` will revert due to overflow when `pricePerToken * numTokens > type(uint32).max`

**Description:** `PledgeManager::pledge`는 두 개의 `uint32` 변수를 곱하고 그 결과를 `uint256`에 저장하여, 곱셈 결과가 `type(uint32).max`보다 클 때를 처리하려고 시도합니다:
```solidity
uint256 stablecoinAmount = pricePerToken * numTokens; // account for overflow
```

`PledgeManager::refundTokens`도 같은 방식을 사용합니다:
```solidity
uint256 refundAmount = numTokens * pricePerToken; //TOOD: overflow check
```

하지만 곱셈 결과가 `type(uint32).max`보다 크면 함수가 revert되기 때문에 이는 제대로 작동하지 않습니다.

**Impact:** `uint32`의 최대값은 4294967295입니다. `pricePerToken`이 6자리 소수를 사용하므로, 가능한 최대 `stablecoinAmount`는 $4294.96로 매우 낮습니다. 사용자들이 원하는 많은 합리적인 금액에 대해 서약(pledging)이 revert될 것입니다.

또한 컨트랙트는 업그레이드할 수 없으므로 업그레이드를 통해 수정할 수 없습니다.

**Proof of Concept:** [chisel](https://getfoundry.sh/chisel/overview)을 사용하여 이 동작을 쉽게 확인할 수 있습니다:
```solidity
$ chisel
Welcome to Chisel! Type `!help` to show available commands.
➜ uint32 a = type(uint32).max;
➜ uint32 b = 10;
➜ uint256 c = a * b;
Traces:
  [401] 0xBd770416a3345F91E4B34576cb804a576fa48EB1::run()
    └─ ← [Revert] panic: arithmetic underflow or overflow (0x11)

Error: Failed to execute REPL contract!
```

**Recommended Mitigation:** 첫째, `pricePerToken`과 `numTokens`의 크기를 늘리는 것을 고려하세요. `uint32`의 최대값은 4,294,967,295이므로 다음과 같은 의미가 있습니다:
* 6자리 소수 가격의 경우 최대 `pricePerToken`은 $4294로 너무 작을 수 있습니다.
* 최대 토큰 수량은 4.29B로 적당할 수도 있고 너무 작을 수도 있습니다.
* 간단한 해결책: 모든 프로토콜 토큰 수량을 `uint128`로 표준화하세요.

둘째, `uint32`와 같은 작은 타입을 두 개 곱하는 대신, 그 중 하나를 `uint256`으로 캐스팅하세요:
```diff
- uint256 stablecoinAmount = pricePerToken * numTokens; // account for overflow
+ uint256 stablecoinAmount = uint256(pricePerToken) * numTokens;
```

chisel을 통해 수정 사항을 확인하세요:
```solidity
$ chisel
Welcome to Chisel! Type `!help` to show available commands.
➜ uint32 a = type(uint32).max;
➜ uint32 b = 10;
➜ uint256 c = uint256(a) * b;
➜ c
Type: uint256
├ Hex: 0x9fffffff6
├ Hex (full word): 0x00000000000000000000000000000000000000000000000000000009fffffff6
└ Decimal: 42949672950
```

`TokenBank::buyToken`의 다음 라인들도 비슷한 수정이 필요한지 고려하세요:
```solidity
// @audit can `amount * curData.pricePerToken * curData.saleFee > type(uint64).max`? If so then
// consider making a similar fix here to prevent overflow revert
        uint64 stablecoinValue = amount * curData.pricePerToken;
        uint64 feeValue = (stablecoinValue * curData.saleFee) / 1e6;
```

**Remora:** 커밋 [a0b277f](https://github.com/remora-projects/remora-smart-contracts/commit/a0b277fe4a59354f3b3783c4b8c06eb60f5157610), [ced21ba](https://github.com/remora-projects/remora-smart-contracts/commit/ced21ba9758b814eb48a09a5e792aa89cc87e8f5)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Distribution of payouts will revert due to overflow when payment is made using a stablecoin with high decimals

**Description:** 지불금(Payouts)은 스테이블코인으로 지불되도록 의도디었으며, 원래 6자리 소수의 스테이블코인(USDC)이었습니다. 그러나 시스템은 지불에 사용되는 스테이블코인을 변경할 수 있는 기능을 가지고 있습니다. USDT(8자리 소수)나 USDS(18자리 소수)가 될 수 있습니다.

PaymentSettler를 도입하기 위한 변경의 일환으로, [`calculatedPayout`의 데이터 타입이 uint256에서 uint64로 변경되었습니다](https://github.com/remora-projects/remora-smart-contracts/blob/audit/Dacian/contracts/RWAToken/DividendManager.sol#L42). 이 변경은 되돌릴 수 없는 DoS를 유발하여 사용자가 지불금을 수령하지 못하게 할 수 있는 치명적인 취약점을 도입합니다.

18자리 소수의 스테이블코인을 사용하여 20 USD의 지불금을 분배할 때 uint64는 revert됩니다.
- chisel에서 볼 수 있듯이 20e18은 uint64가 담을 수 있는 최대값보다 큽니다.
```
➜ bool a = type(uint64).max > 20e18;
➜ a
Type: bool
└ Value: false
```

예를 들어, 계산해야 할 분배가 5건 남아 있는 사용자가 있고, 가장 최근 분배가 18자리 소수의 스테이블코인으로 지불된다고 가정해 봅시다. (사용자가 각 분배에서 50USD를 번다고 가정)
- 사용자가 지불금을 계산하려고 시도할 때, 마지막 분배가 사용자의 지불금을 uint64에 담을 수 있는 값보다 크게 만들기 때문에 트랜잭션이 revert됩니다. 따라서 [지불금을 uint64로 safeCasting할 때](https://github.com/remora-projects/remora-smart-contracts/blob/audit/Dacian/contracts/RWAToken/DividendManager.sol#L435-L440) 오버플로우가 발생하고 트랜잭션이 실패하여, 이 사용자는 가장 최근 지불금뿐만 아니라 아직 계산되지 않은 이전의 모든 지불금에 대해서도 수령이 불가능한 DoS 상태가 됩니다.

```solidity
    function payoutBalance(address holder) public returns (uint256) {
        ...
        for (uint16 i = payRangeStart; i >= payRangeEnd; --i) {
            ...

            PayoutInfo memory pInfo = $._payouts[i];
//@audit => `pInfo.amount` set using a stablecoin with high decimals will bring up the payoutAmount beyond the limit of what can fit in a uint64
            payoutAmount +=
                (curEntry.tokenBalance * pInfo.amount) /
                pInfo.totalSupply;
            if (i == 0) break; // to prevent potential overflow
        }
        ...
        if (payoutForwardAddr == address(0)) {
//@audit-issue => overflow will blow up the tx
            holderStatus.calculatedPayout += SafeCast.toUint64(payoutAmount);
        } else {
//@audit-issue => overflow will blow up the tx
            $._holderStatus[payoutForwardAddr].calculatedPayout += SafeCast
                .toUint64(payoutAmount);
        }
    }
```

**Impact:** 홀더들의 지불금 분배에 대한 되돌릴 수 없는 DoS.

**Recommended Mitigation:** 이 문제를 해결하기 위한 가장 간단한 수정 방법은 `calculatePayout`의 데이터 타입을 최소 `uint128`로 변경하고 모든 토큰 수량을 `uint128`로 표준화하는 것을 고려하는 것입니다.

하지만 이번에는 한 단계 더 나아가 시스템의 내부 회계를 고정된 소수점 자릿수로 정규화하여, 지불 처리에 사용되는 실제 스테이블코인의 소수점 자릿수에 영향을 받지 않도록 하는 것이 좋습니다.

이 변경의 일환으로 `PaymentSettler` 컨트랙트는 RemoraToken에서 주고받은 값을 현재 설정된 스테이블코인의 실제 소수점 자릿수로 변환하는 책임을 맡아야 합니다.

**Remora:** 커밋 [a0b277f](https://github.com/remora-projects/remora-smart-contracts/commit/a0b277fe4a59354f3b3783c4b8c06eb60f5157610), [ced21ba](https://github.com/remora-projects/remora-smart-contracts/commit/ced21ba9758b814eb48a09a5e792aa89cc87e8f5)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### A single holder can grief the payouts of all holders forwarding their payouts to the same forwarder

**Description:** 이 그리핑(griefing) 공격은 [이슈 [*Forwarders can lose payouts of the holders forwarding to them*](#forwarders-can-lose-payouts-of-the-holders-forwarding-to-them)](https://github.com/remora-projects/remora-smart-contracts/issues/49)와 유사합니다. 주요 차이점은 이 공격은 포워더(forwarder)가 홀더 상태를 얻고 동일한 `distributionIndex`에서 잔액을 0으로 만들 필요가 없다는 것입니다. 이 그리핑 공격은 포워더가 잔액이 없는 상태에서 어떤 인덱스에서도 실행될 수 있습니다.

그리핑 공격이 발생할 수 있는 단계는 다음과 같습니다:
1. 포워더가 잔액을 가지고 있으며, 홀더입니다.
2. 여러 홀더들이 동일한 주소를 지정된 포워더로 설정합니다.
3. 홀더들에 대한 지불금이 계산되어 포워더에게 입금됩니다.
4. 포워더가 지불금을 청구(claim)하고, 보류 중인 모든 지불금이 계산됩니다.
    - 이 시점에서 포워더의 `payoutBalance`는 0이 됩니다.
5. 포워더가 잔액을 0으로 만들고, [`isHolder` 상태가 제거됩니다(더 이상 홀더가 아님)](https://github.com/remora-projects/remora-smart-contracts/blob/main/contracts/RWAToken/DividendManager.sol#L596-L600).
6. 분배가 지나갑니다.
7. 홀더 중 한 명이 자신의 지정된 포워더에서 [포워더를 제거합니다](https://github.com/remora-projects/remora-smart-contracts/blob/main/contracts/RWAToken/DividendManager.sol#L215-L222).
    - [포워더가 잔액이 없고, 홀더가 아니기 때문에 포워더의 데이터가 삭제됩니다](https://github.com/remora-projects/remora-smart-contracts/blob/main/contracts/RWAToken/DividendManager.sol#L362-L365). 여기에는 해당 포워더를 포워더로 설정한 홀더들을 위해 누적된 미지급 `calculatedPayout`이 포함됩니다.
8. 7단계의 결과로, 홀더가 번 미청구 지불금이 소실됩니다.

**Impact:**
- 동일한 포워더에게 포워딩하는 홀더들의 지불금이 단일 홀더에 의해 그리핑될 수 있습니다.
- 비-홀더(non-holder) 계정으로 지불금을 포워딩하는 홀더들은, 포워더가 비-홀더인 상태에서 포워더를 제거하면 지불금을 잃게 됩니다.

**Proof of Concept:** Description 섹션에 설명된 시나리오를 재현하려면 다음 테스트를 실행하세요.
```solidity
    function test_holderForcesForwarderToLosePayouts() public {
        address user1 = users[0];
        address user2 = users[1];
        address forwarder = users[2];

        uint256 amountToMint = 1;

        _whitelistAndMintTokensToUser(user1, amountToMint * 8);
        _whitelistAndMintTokensToUser(user2, amountToMint);
        _whitelistAndMintTokensToUser(forwarder, amountToMint);

        // both users sets the same forwarder as their forwardAddress
        remoraTokenProxy.setPayoutForwardAddress(user1, forwarder);
        remoraTokenProxy.setPayoutForwardAddress(user2, forwarder);

        // fund total payout amount to funding wallet
        uint64 payoutDistributionAmount = 100e6;

        // Distribute payouts for the first 5 distributions
        for(uint i = 1; i <= 5; i++) {
            _fundPayoutToPaymentSettler(payoutDistributionAmount);
        }

        // user1 must have 0 payout because it is forwarding to `forwarder`
        uint256 user1PayoutBalance = remoraTokenProxy.payoutBalance(user1);
        assertEq(user1PayoutBalance, 0, "Forwarding payout is not working as expected");

        // user2 must have 0 payout because it is forwarding to `forwarder`
        uint256 user2PayoutBalance = remoraTokenProxy.payoutBalance(user2);
        assertEq(user2PayoutBalance, 0, "Forwarding payout is not working as expected");

        //forwarder must have the full payout for the 5 distributions because both users are forwarding to him
        uint256 forwarderPayoutBalance = remoraTokenProxy.payoutBalance(forwarder);
        assertEq(forwarderPayoutBalance, payoutDistributionAmount * 5, "Forwarding payout is not working as expected");

        // forwarder claims all the outstanding payout
        vm.startPrank(forwarder);
        remoraTokenProxy.claimPayout();
        assertEq(stableCoin.balanceOf(forwarder), forwarderPayoutBalance);

        // forwarder zeros out his PropertyToken's balance
        remoraTokenProxy.transfer(user2, remoraTokenProxy.balanceOf(forwarder));
        vm.stopPrank();

        assertEq(remoraTokenProxy.balanceOf(forwarder), 0);

        (bool isHolder) = remoraTokenProxy.getHolderStatus(forwarder).isHolder;
        assertEq(isHolder, false);

        // Distribute payouts for distributions 5 - 10
        for(uint i = 1; i <= 5; i++) {
            _fundPayoutToPaymentSettler(payoutDistributionAmount);
        }

        // user1 must have 0 payout because it is forwarding to `forwarder`
        user1PayoutBalance = remoraTokenProxy.payoutBalance(user1);
        assertEq(user1PayoutBalance, 0, "Forwarding payout is not working as expected");

        // user2 must have 0 payout because it is forwarding to `forwarder`
        user2PayoutBalance = remoraTokenProxy.payoutBalance(user2);
        assertEq(user2PayoutBalance, 0, "Forwarding payout is not working as expected");

        (uint64 calculatedPayout) = remoraTokenProxy.getHolderStatus(forwarder).calculatedPayout;
        assertEq(calculatedPayout, payoutDistributionAmount * 5, "Forwarder did not receive payout for holder forwarding to him");

        // user2 gets forwarder removed as its forwardedAddress
        remoraTokenProxy.removePayoutForwardAddress(user2);

        //@audit => When this vulnerability is fixed, we expect finalCalculatedPayout to be equals than calculatedPayout!
        (uint64 finalCalculatedPayout) = remoraTokenProxy.getHolderStatus(forwarder).calculatedPayout;
        //@audit-issue => user2 causes the payout of user1 to be lost, which is 4x the payout lose by him
        assertEq(finalCalculatedPayout, 0, "Forwarder did not lose payout of holder");
    }
```

Description 섹션에 설명된 것과 유사한 두 번째 시나리오가 있습니다. 이 다른 시나리오에서는 포워더가 비-홀더이며, 몇 번의 분배 후 홀더가 현재 포워더를 제거하거나 다른 주소로 변경하기로 결정하면, 미청구 지불금이 소실됩니다.
- 이전 시나리오를 입증하기 위해 다음 테스트를 실행하세요.
```solidity
    function test_HolderLosesPayout_HolderRemovesForwarderWhoWasNeverAHolder() public {
        address user1 = users[0];
        address forwarder = users[1];

        uint256 amountToMint = 1;

        _whitelistAndMintTokensToUser(user1, amountToMint);

        remoraTokenProxy.setPayoutForwardAddress(user1, forwarder);

        // fund total payout amount to funding wallet
        uint64 payoutDistributionAmount = 100e6;

        // Distribute payouts for the first 5 distributions
        for(uint i = 1; i <= 5; i++) {
            _fundPayoutToPaymentSettler(payoutDistributionAmount);
        }

        // user1 must have 0 payout because it is forwarding to `forwarder`
        uint256 user1PayoutBalance = remoraTokenProxy.payoutBalance(user1);
        assertEq(user1PayoutBalance, 0, "Forwarding payout is not working as expected");

        (uint64 forwarderPayoutBalance) = remoraTokenProxy.getHolderStatus(forwarder).calculatedPayout;
        assertEq(forwarderPayoutBalance, payoutDistributionAmount * 5, "Forwarding payout is not working as expected");

        // forwarder attempts to claim all his payout while he is not a holder
        (bool isHolder) = remoraTokenProxy.getHolderStatus(forwarder).isHolder;
        assertEq(isHolder, false);
        // claiming reverts because forwarder is not a holder
        vm.prank(forwarder);
        vm.expectRevert();
        remoraTokenProxy.claimPayout();

        // user1 gets forwarder removed as its forwardedAddress
        remoraTokenProxy.removePayoutForwardAddress(user1);

        // validate forwarder and holder have lost the payouts for the past 5 distributions
        (forwarderPayoutBalance) = remoraTokenProxy.getHolderStatus(forwarder).calculatedPayout;
        assertEq(forwarderPayoutBalance, 0, "Forwarding payout is not working as expected");

        (uint256 finalForwarderPayoutBalance) = remoraTokenProxy.payoutBalance(forwarder);
        assertEq(finalForwarderPayoutBalance, 0, "Forwarding payout is not working as expected");

        // user1 must have 0 payout because it is forwarding to `forwarder`
        user1PayoutBalance = remoraTokenProxy.payoutBalance(user1);
        assertEq(user1PayoutBalance, 0, "Forwarding payout is not working as expected");
    }
```

**Recommended Mitigation:** `_removePayoutForwardAddress()`에서 `forwardedAddress`의 `holderStatus`가 0인지 확인하고, 그렇지 않으면 `deleteUser()`를 호출하지 마세요.

```diff
function _removePayoutForwardAddress(
        HolderManagementStorage storage $,
        address holder,
        address forwardedAddress
    ) internal {
        if (forwardedAddress != address(0)) {
            ...
            if (
                balanceOf(forwardedAddress) == 0 &&
                payoutBalance(forwardedAddress) == 0
+              && $._holderStatus[forwardedAddress].calculatedPayout == 0
            ) deleteUser(forwardedHolder);
        }
```

**Remora:** 커밋 [7bd2691](https://github.com/remora-projects/remora-smart-contracts/commit/7bd269128ebeac7f2cae0e30d55ee666e8fa21d7)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.

\clearpage
## High Risk


### Attacker can make pledge on behalf of users if those users have approved `PledgeManager` to spend their tokens

**Description:** `PledgeManager`는 사용자가 토큰을 지출하도록 승인해야 서약(pledges)을 할 수 있습니다. 사용자는 다음 두 가지 방법으로 이를 수행할 수 있습니다:
1) `IERC20Permit::permit` 사용: 서명자의 nonce, deadline, domain separator를 강제합니다.
2) 수동으로 `IERC20::approve` 호출

사용자가 수동 방법 2)를 사용하고 토큰 승인을 열어두면, 공격자가 `PledgeManager::pledge`를 호출하여 그들을 대신해 서약을 할 수 있습니다. 왜냐하면 이 함수는 `msg.sender == data.signer`인지 확인하지 않기 때문입니다.

**Impact:** 공격자가 무고한 사용자들을 대신해 서약을 하여 해당 사용자의 토큰을 지출할 수 있습니다. 사용자가 모든 토큰을 해당 프로토콜에서 사용할 의도가 없음에도 불구하고 자주 사용하는 프로토콜에 대해 최대 승인을 해두는 것은 일반적입니다.

**Recommended Mitigation:** `PledgeManager::pledge`에서 `IERC20Permit::permit`을 사용하지 않을 때 `msg.sender == data.signer`임을 강제하세요:
```diff
        if (data.usePermit) {
            IERC20Permit(stablecoin).permit(
                signer,
                address(this),
                finalStablecoinAmount,
                block.timestamp + 300,
                data.permitV,
                data.permitR,
                data.permitS
            );
        }
+       else if(msg.sender != signer) revert MsgSenderNotSigner();
```

대안으로 `PledgeManager::refundTokens`가 작동하는 방식과 유사하게 항상 `msg.sender`를 사용하세요.

**Remora:** 커밋 [e3bda7c](https://github.com/remora-projects/remora-smart-contracts/commit/e3bda7c78321febb0e2f37b29912ba24c9e04343)에서 항상 `msg.sender`를 사용하도록 하고 permit 메서드를 제거하여 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Accounting on `PaymentSettler` will be corrupted when changing `stablecoin` that is used to process payments

**Description:** `PaymentSettler`의 회계(accounting)는 시스템 초기에 사용된 초기 스테이블코인의 소수점 자릿수를 기반으로 초기화됩니다.
시스템은 [지불에 사용되는 스테이블코인을 변경할 수 있는 기능](https://github.com/remora-projects/remora-smart-contracts/blob/audit/Dacian/contracts/PaymentSettler.sol#L195-L198)이 있으며, 스테이블코인이 다른 소수점 자릿수를 가진 스테이블코인으로 변경되면, 새로운 금액이 시스템의 기존 값과 달라지기 때문에 모든 기존 회계가 엉망이 됩니다.

이 문제는 `PaymentSettler`가 시스템에 도입된 마지막 변경에서 발생했습니다. 이전 버전에서는 시스템이 내부 회계의 소수점 자릿수를 지불에 사용되는 활성 스테이블코인의 소수점 자릿수로 올바르게 처리했습니다.

예를 들어, 6자리 소수의 스테이블코인을 사용할 때 생성된 100 USD의 수수료는 스테이블코인이 8자리 소수의 스테이블코인으로 변경되면 1 USD밖에 되지 않게 됩니다.

**Impact:** `stablecoin`을 다른 소수점 자릿수로 변경할 때 `PaymentSettler`의 회계가 손상됩니다.

**Recommended Mitigation:** C-2에 대한 권장 사항을 참조하세요.

**Remora:** 커밋 [a0b277f](https://github.com/remora-projects/remora-smart-contracts/commit/a0b277fe4a59354f3b3783c4b8c06eb60f5157610), [ced21ba](https://github.com/remora-projects/remora-smart-contracts/commit/ced21ba9758b814eb48a09a5e792aa89cc87e8f5)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### `PaymentSettler` can change `stablecoin` but `RemoraToken` can't resulting in corrupted state with DoS for core functions

**Description:** `RemoraToken`에는 `stablecoin` 멤버가 있으며, 주석에 `PaymentSettler`와 일치해야 한다고 표시되어 있습니다:
```solidity
address public stablecoin; //make sure same stablecoin is used here that is used in payment settler
```

하지만 업데이트된 코드에서 `RemoraToken::stablecoin`을 업데이트할 방법이 없습니다; 이전에 `RemoraToken`이 상속받은 `DividendManager`에는 `changeStablecoin` 함수가 있었지만 `PaymentSettler`의 도입으로 주석 처리되었습니다.

`PaymentSettler`는 `stablecoin` 멤버와 이를 변경할 수 있는 함수를 가지고 있습니다:
```solidity
address public stablecoin;

function changeStablecoin(address newStablecoin) external restricted {
    if (newStablecoin == address(0)) revert InvalidAddress();
    stablecoin = newStablecoin;
}
```

**Impact:** `PaymentSettler`가 `stablecoin`을 변경하면 변경할 수 없는 `RemoraToken::stablecoin`과 달라져 상태가 손상되고 주요 함수가 revert되는 원인이 됩니다.

**Proof Of Concept:**
```solidity
function test_changeStablecoin_inconsistentState() external {
    address newStableCoin = address(new Stablecoin("USDC", "USDC", 0, 6));

    // change stablecoin on PaymentSettler
    paySettlerProxy.changeStablecoin(newStableCoin);
    assertEq(paySettlerProxy.stablecoin(), newStableCoin);

    // now inconsistent with RemoraToken
    assertEq(remoraTokenProxy.stablecoin(), address(stableCoin));
    assertNotEq(paySettlerProxy.stablecoin(), remoraTokenProxy.stablecoin());

    // no way to update RemoraToken::stablecoin
}
```

**Recommended Mitigation:** `RemoraToken`과 `PaymentSettler`가 항상 동일한 `stablecoin`을 참조하도록 강제하세요. 이를 구현할 때, 소수점 자릿수가 다른 `stablecoin`으로 변경하면 프로토콜 회계가 손상된다는 다른 발견 사항을 고려하세요.

가장 간단한 해결책은 `RemoraToken`에서 `stablecoin`을 완전히 제거하고 `PaymentSettler`가 모든 필요한 전송을 수행하도록 하는 것일 수 있습니다.

**Remora:** 커밋 [ced21ba](https://github.com/remora-projects/remora-smart-contracts/commit/ced21ba9758b814eb48a09a5e792aa89cc87e8f5)에서 `RemoraToken`에서 `stablecoin`을 제거하고, 전송 수수료 로직을 `PaymentSettler`로 이동시키고 `RemoraToken`이 `PaymentSettler::settleTransferFee`를 호출하도록 하여 수정되었습니다.

**Cyfrin:** 검증되었습니다.

\clearpage
## Medium Risk


### `PledgeManager::refundTokens` doesn't decrement `tokensSold` when pledge hasn't concluded, preventing pledge from reaching its funding goal

**Description:** `PledgeManager::refundTokens`는 서약이 종료되지 않았을 때 `tokensSold`를 감소시키지 않습니다.

**Impact:** 환불된 토큰을 다른 사용자가 구매할 수 없으므로 서약이 자금 목표에 도달하는 것이 차단됩니다.

**Recommended Mitigation:** `PledgeManager::refundTokens`는 항상 `tokensSold`를 감소시켜야 합니다:
```diff
        if (
            !pledgeRoundConcluded &&
            SafeCast.toUint32(block.timestamp) < deadline
        ) {
            refundAmount -= (refundAmount * earlySellPenalty) / 1e6;
            _propertyToken.adminTransferFrom(
                signer,
                holderWallet,
                numTokens,
                false,
                false
            );
            emit TokensUnPledged(signer, numTokens);
        } else {
            fee = (userPay.fee / userPay.tokensBought) * numTokens;
            _propertyToken.burnFrom(signer, numTokens);
            emit TokensRefunded(signer, numTokens);
-           tokensSold -= numTokens;
        }

+      tokensSold -= numTokens;
```

**Remora:** 커밋 [6be4660](https://github.com/remora-projects/remora-smart-contracts/commit/6be4660990ebafbb7200425978f078a0865732fe)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### `TokenBank::withdrawFunds` resets `memory` not `storage` fee and sale amounts allowing multiple withdraws for the same token

**Description:** `TokenBank::withdrawFunds`는 `storage`가 아닌 `memory`의 fee 및 sale 금액을 초기화하여 동일한 토큰에 대해 여러 번 출금할 수 있게 합니다:
```solidity
    function withdrawFunds(
        address tokenAddress,
        bool fee
    ) public nonReentrant restricted {
        TokenData memory curData = tokenData[tokenAddress];
        address to;
        uint64 amount;
        if (fee) {
            to = custodialWallet;
            amount = curData.feeAmount;
            curData.feeAmount = 0; // @audit resets memory not storage
        } else {
            to = curData.withdrawTo;
            amount = curData.saleAmount;
            curData.saleAmount = 0; // @audit resets memory not storage
        }
        if (amount != 0) IERC20(stablecoin).transfer(to, amount);

        if (fee) emit FeesClaimed(tokenAddress, amount);
        else emit FundsClaimed(tokenAddress, amount);
    }
```

**Impact:** 관리자는 동일한 토큰 주소에 대해 여러 번 fee 및 sale 금액을 출금할 수 있습니다. 이는 다른 판매에서 충분한 fee 및 sale 토큰이 있는 한 작동합니다.

**Recommended Mitigation:** `memory`가 아닌 `storage`를 초기화하세요:
```diff
    function withdrawFunds(
        address tokenAddress,
        bool fee
    ) public nonReentrant restricted {
        TokenData memory curData = tokenData[tokenAddress];
        address to;
        uint64 amount;
        if (fee) {
            to = custodialWallet;
            amount = curData.feeAmount;
-           curData.feeAmount = 0;
+           tokenData[tokenAddress].feeAmount = 0;
        } else {
            to = curData.withdrawTo;
            amount = curData.saleAmount;
-           curData.saleAmount = 0;
+           tokenData[tokenAddress].saleAmount = 0;
        }
        if (amount != 0) IERC20(stablecoin).transfer(to, amount);

        if (fee) emit FeesClaimed(tokenAddress, amount);
        else emit FundsClaimed(tokenAddress, amount);
    }
```

**Remora:** 커밋 [571bfe4](https://github.com/remora-projects/remora-smart-contracts/commit/571bfe4b3129d1acaee62e323a3165c7b1c0f3d1)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Fee should be calculated after first purchase discount is applied in `TokenBank::buy` to prevent over-charging users

**Description:** `TokenBank::buy`는 총 구매 금액을 다음과 같이 구성된 것으로 계산합니다:
* `stablecoinValue` : 토큰의 가치
* `feeValue` : `stablecoinValue`에서 계산된 수수료
```solidity
        uint64 stablecoinValue = amount * curData.pricePerToken;
        uint64 feeValue = (stablecoinValue * curData.saleFee) / 1e6;
```

그 후, 이것이 사용자의 첫 구매인 경우 `stablecoinValue`에 대해 할인을 받습니다:
```solidity
        IReferralManager refManager = IReferralManager(referralManager);
        bool firstPurchase = refManager.isFirstPurchase(to);
        if (firstPurchase) stablecoinValue -= refManager.referDiscount();
```

그러나 수수료는 업데이트되지 않으므로 초기 더 높은 `stablecoinValue` 금액에서 계산된 상태로 유지됩니다.

**Impact:** 사용자는 첫 구매 할인을 받을 때 내야 할 것보다 더 높은 수수료를 지불하게 됩니다.

**Recommended Mitigation:** 할인이 적용된 후에만 `feeValue`를 계산하세요:
```diff
        address to = msg.sender;
        uint64 stablecoinValue = amount * curData.pricePerToken;
-       uint64 feeValue = (stablecoinValue * curData.saleFee) / 1e6;

        IReferralManager refManager = IReferralManager(referralManager);
        bool firstPurchase = refManager.isFirstPurchase(to);
        if (firstPurchase) stablecoinValue -= refManager.referDiscount();

+       uint64 feeValue = (stablecoinValue * curData.saleFee) / 1e6;
        curData.saleAmount += stablecoinValue;
        curData.feeAmount += feeValue;
```

**Remora:** 커밋 [4aea246](https://github.com/remora-projects/remora-smart-contracts/commit/4aea246c8de4dcd03bc11a5cd87ca617e787bdaf), [5510920](https://github.com/remora-projects/remora-smart-contracts/commit/55109201b0b592abb94a3c73a5f45c9c24b3d440)에서 수정되었습니다 - 규제상의 이유로 수수료 계산 방식이 변경되었습니다.

**Cyfrin:** 검증되었습니다.


### Buyers can pledge for tokens without having signed all documents that are required to be signed

**Description:** 구매자가 서약 라운드(pledgeRound) 동안 pledge()를 할 때, 서명해야 하는 [모든 문서에 서명했는지](https://github.com/remora-projects/remora-smart-contracts/blob/main/contracts/PledgeManager.sol#L316) 검증합니다.
적어도 하나의 문서라도 서명되지 않은 경우, 트랜잭션을 revert하는 대신 실행은 [서명자를 대신하여 서명을 검증하고](https://github.com/remora-projects/remora-smart-contracts/blob/main/contracts/PledgeManager.sol#L317-L322), 이 서명이 합법적이면 실행을 계속합니다.

```solidity
    function _verifyDocumentSignature(
        PledgeData calldata data,
        address signer
    ) internal {
        (bool res, ) = IRemoraRWAToken(propertyToken).hasSignedDocs(signer);
        if (!res)
            IRemoraRWAToken(propertyToken).verifySignature(
                signer,
                data.docHash,
                data.signature
            );
    }
```

**문제는** 이 구현이 구매자가 단 하나의 문서에만 서명함으로써 모든 문서에 서명해야 하는 요구 사항을 우회할 수 있게 한다는 점입니다. 예를 들어:
서명해야 할 3개의 문서가 있고, 사용자는 그 중 아무것도 서명하지 않았습니다. 사용자는 pledge()를 호출하고 1개의 문서에 서명하기 위한 서명 데이터를 제공합니다. 다음과 같은 일이 발생합니다:

[PropertyToken::hasSignedDocs() ](https://github.com/remora-projects/remora-smart-contracts/blob/main/contracts/RWAToken/DocumentManager.sol#L114-L127)는 사용자가 3개의 문서 중 아무것도 서명하지 않았으므로 false를 반환합니다.
```solidity
    function hasSignedDocs(address signer) public view returns (bool, bytes32) {
        ...

        for (uint256 i = 0; i < numDocs; ++i) {
            bytes32 docHash = $._docHashes[i];
//@audit => If one document that needs signature is not signed, returns false
            if (
                $._documents[docHash].needSignature &&
                $._signatureRecords[signer][docHash] == 0
            ) return (false, docHash);
        }

//@audit => returns true only if all documents that requires signature are signed
        return (true, 0x0);
    }
```

[PropertyToken::verifySignature()](https://github.com/remora-projects/remora-smart-contracts/blob/main/contracts/RWAToken/DocumentManager.sol#L148-L184)는 3개의 문서 중 하나를 서명하므로 revert되지 않습니다.
```solidity
    function verifySignature(
        address signer,
        bytes32 docHash,
        bytes memory signature
    ) external returns (bool result) {
        ...
        if (signer.code.length == 0) {
            //signer is EOA
            (address returnedSigner, , ) = ECDSA.tryRecover(digest, signature);
            result = returnedSigner == signer;
        } else {
            //signer is SCA
            (bool success, bytes memory ret) = signer.staticcall(
                ...
            );
            result = (success && ret.length == 32 && bytes4(ret) == MAGICVALUE);
        }

        if (!result) revert InvalidSignature();
        if ($._signatureRecords[signer][docHash] == 0) {
           ...
        }
//@audit => if the verification of the provided signature succeeds, execution continues
    }
```

**문제는** 사용자가 서명해야 할 3개의 문서 중 1개만 서명했음에도 불구하고 실행이 계속된다는 것입니다. 왜냐하면 (앞서 설명했듯이) [_verifyDocumentSignature()](https://github.com/remora-projects/remora-smart-contracts/blob/main/contracts/PledgeManager.sol#L312-L323)가 단 하나의 서명만 강제하도록 우회되고, holderWallet에서 서명자로 전송할 때 [`checkTC`가 false로 설정되기 때문입니다](https://github.com/remora-projects/remora-smart-contracts/blob/main/contracts/PledgeManager.sol#L207).

[PledgeManager::pledge()](https://github.com/remora-projects/remora-smart-contracts/blob/main/contracts/PledgeManager.sol#L167-L221)
```solidity
    function pledge(PledgeData calldata data) external nonReentrant {
        ...

        _verifyDocumentSignature(data, signer);

       ...

//@audit => checkTC is set as false
        //this address should be whitelisted in property token
        IRemoraRWAToken(propertyToken).adminTransferFrom(
            holderWallet,
            signer,
            numTokens,
            false,  // <====> checkTC //
            true
        );
       ...
    }
```

[RemoraToken::adminTransferFrom()](https://github.com/remora-projects/remora-smart-contracts/blob/main/contracts/RWAToken/RemoraToken.sol#L228-L252)
```solidity
    function adminTransferFrom(
        address from,
        address to,
        uint256 value,
        bool checkTC,
        bool enforceLock
    ) external restricted returns (bool) {
       ...
//@audit => checkTC as false effectively bypass the verification of TC to be signed
        (bool res, ) = hasSignedDocs(to);
        if (checkTC && !res) revert TermsAndConditionsNotSigned(to);

       ...
    }

```


**Impact:** 구매자는 서명해야 할 모든 문서에 서명하지 않았음에도 불구하고 토큰을 구매할 수 있습니다.

**Recommended Mitigation:** `PropertyToken.hasSignedDocs()` 호출이 false를 반환하면 실행을 revert하세요.

```diff
function _verifyDocumentSignature(
        PledgeData calldata data,
        address signer
    ) internal {

        (bool res, ) = IRemoraRWAToken(propertyToken).hasSignedDocs(signer);

-      if (!res)
-          IRemoraRWAToken(propertyToken).verifySignature(
-             signer,
-             data.docHash,
-              data.signature
-         );

+     if (!res) revert NotAllDocumentsAreSigned();

    }
```

**Remora:** 커밋 [5510920](https://github.com/remora-projects/remora-smart-contracts/commit/55109201b0b592abb94a3c73a5f45c9c24b3d440#diff-7ce9b56302de46e809d6a2bc534817c3a4ea314d0e16d299fae932f057245486L193-R191)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Hardcoding deadline for `permit()` will mess up the `structHash` leading to an invalid signature

**Description:** Permit을 통해 ERC20 승인을 허용하는 함수들이 permit()에 전달되는 deadline을 잘못 하드코딩합니다.
```solidity
function pledge(PledgeData calldata data) external nonReentrant {
        ...

        //5 minute deadline
        if (data.usePermit) {
            IERC20Permit(stablecoin).permit(
                signer,
                address(this),
                finalStablecoinAmount,
//@audit-issue => hardcoded deadline
                block.timestamp + 300,
                data.permitV,
                data.permitR,
                data.permitS
            );
        }
        ...
}
```
문제는 복구된 서명이 유효하지 않게 된다는 것입니다. deadline은 structHash의 매개변수이며, 서명된 deadline이 1초라도 다르면, 해시된 structHash가 사용자가 실제로 서명한 것과 다르게 되기 때문입니다.

```solidity
function permit(
        address owner,
        address spender,
        uint256 value,
        uint256 deadline,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) public virtual {
        if (block.timestamp > deadline) {
            revert ERC2612ExpiredSignature(deadline);
        }

//@audit-issue => A hardcoded deadline will mess up the structHash
        bytes32 structHash = keccak256(abi.encode(PERMIT_TYPEHASH, owner, spender, value, _useNonce(owner), deadline));

        bytes32 hash = _hashTypedDataV4(structHash);

//@audit-issue => A different hash than the actual hash signed by the signer will recover a != signer (even address(0)
        address signer = ECDSA.recover(hash, v, r, s);
        if (signer != owner) {
            revert ERC2612InvalidSigner(signer, owner);
        }

        _approve(owner, spender, value);
    }
```

**Impact:** permit을 통해 ERC20Tokens를 승인하도록 허용하는 `pledge()`, `refundToken()`과 같은 함수들이 작동하지 않을 것입니다. 왜냐하면 structHash가 사용자가 실제로 서명한 해시와 다를 것이기 때문입니다.

**Recommended Mitigation:** deadline을 하드코딩하는 대신 매개변수로 받으세요.

```diff
function pledge(PledgeData calldata data) external nonReentrant {
        ...

        //5 minute deadline
        if (data.usePermit) {
            IERC20Permit(stablecoin).permit(
                signer,
                address(this),
                finalStablecoinAmount,
-              block.timestamp + 300,
+              data.deadline,
                data.permitV,
                data.permitR,
                data.permitS
            );
        }
        ...
}
```

**Remora:** [5510920](https://github.com/remora-projects/remora-smart-contracts/commit/55109201b0b592abb94a3c73a5f45c9c24b3d440)에서 permit 기능을 제거하여 해결되었습니다.

**Cyfrin:** 검증되었습니다.


### `DocumentManager::hasSignedDocs` incorrectly returns `true` when there are no documents to sign

**Description:** `DocumentManager::hasSignedDocs`는 서명할 문서가 없을 때 잘못되게 `true`를 반환합니다:
```solidity
function hasSignedDocs(address signer) public view returns (bool, bytes32) {
    DocumentStorage storage $ = _getDocumentStorage();
    uint256 numDocs = $._docHashes.length;

    // @audit when numDocs = 0, the `for` loop is bypassed
    // skipping to the `return (true, 0x0);` statement
    for (uint256 i = 0; i < numDocs; ++i) {
        bytes32 docHash = $._docHashes[i];
        if (
            $._documents[docHash].needSignature &&
            $._signatureRecords[signer][docHash] == 0
        ) return (false, docHash);
    }

    return (true, 0x0);
}
```

**Impact:** 업스트림 컨트랙트는 사용자가 문서에 서명했다고 잘못 가정하고 금지되어야 할 사용자 작업을 허용합니다.

**Proof Of Concept:**
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.22;

import {DocumentManager} from "../../../contracts/RWAToken/DocumentManager.sol";

import {UnitTestBase} from "../UnitTestBase.sol";

contract DocumentManagerTest is UnitTestBase, DocumentManager {
    function setUp() public override {
        UnitTestBase.setUp();
        initialize();
    }

    function initialize() public initializer {
        __RemoraDocuments_init("NAME", "VERSION");
    }

    // this is incorrect and should be changed once bug is fixed
    function test_hasSignedDocs_TrueWhenNoDocs() external {
        // verify no docs
        assertEq(_getDocumentStorage()._docHashes.length, 0);

        // hasSignedDocs returns true even though no docs to sign
        (bool hasSigned, ) = hasSignedDocs(address(0x1337));
        assertTrue(hasSigned);
    }
}
```

**Recommended Mitigation:** 문서가 존재하지 않을 때, 사용자가 문서에 서명했을 리가 없습니다. 따라서 이 경우 `DocumentManager::hasSignedDocs`는 `EmptyDocument`와 같은 특정 오류로 revert되거나 `return (false, 0x0)`해야 합니다.

**Remora:** 커밋 [7454e55](https://github.com/remora-projects/remora-smart-contracts/commit/7454e55e4017aab9d286637fbe5ccf3c705324ba)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Don't add duplicate `documentHash` to `DocumentManager::DocumentStorage::_docHashes` when overwriting via `_setDocument` as this causes panic revert when calling `_removeDocument`

**Description:** `DocumentManager::_setDocument`는 의도적으로 덮어쓰기를 허용하지만, 덮어쓸 때 `_docHashes`에 중복된 `documentHash`를 추가합니다:
```solidity
function _setDocument(
    bytes32 documentName,
    string calldata uri,
    bytes32 documentHash,
    bool needSignature
) internal {
    DocumentStorage storage $ = _getDocumentStorage();
    $._documents[documentHash] = DocData({
        needSignature: needSignature,
        docURI: uri,
        docName: documentName,
        timestamp: SafeCast.toUint32(block.timestamp)
    });

    // @audit duplicate if overwriting
    $._docHashes.push(documentHash);
    emit DocumentUpdated(documentName, uri, documentHash);
}
```

**Impact:** 해시가 `_docHashes`에 중복되면, `_removeDocument` 호출 시 패닉 발생으로 revert되어 문서를 제거할 수 없습니다.

**Proof of Concept:**
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.22;

import {DocumentManager} from "../../../contracts/RWAToken/DocumentManager.sol";

import {UnitTestBase} from "../UnitTestBase.sol";

contract DocumentManagerTest is UnitTestBase, DocumentManager {
    function setUp() public override {
        UnitTestBase.setUp();
        initialize();
    }

    function initialize() public initializer {
        __RemoraDocuments_init("NAME", "VERSION");
    }

    // this is incorrect and should be changed once bug is fixed
    function test_setDocumentOverwrite(string calldata uri) external {
        bytes32 docName = "0x01234";
        bytes32 docHash = "0x5555";
        bool needSignature = true;

        // add the document
        _setDocument(docName, uri, docHash, needSignature);

        // verify its hash has been added to `_docHashes`
        DocumentStorage storage $ = _getDocumentStorage();
        assertEq($._docHashes.length, 1);
        assertEq($._docHashes[0], docHash);

        // ovewrite it
        _setDocument(docName, uri, docHash, needSignature);
        // this duplicates the hash in `_docHashes`
        assertEq($._docHashes.length, 2);
        assertEq($._docHashes[0], docHash);
        assertEq($._docHashes[1], docHash);

        // now attempt to remove it, reverts with
        // panic: array out-of-bounds access
        _removeDocument(docHash);
    }
}
```

**Recommended Mitigation:** `DocumentManager::_setDocument`에서 `documentHash`가 이미 존재하는지 확인하고, 그렇다면 `_docHashes`에 추가하지 마세요.

대안으로 중복을 허용하지 않는 `DocumentManager::DocumentStorage::_docHashes`를 위해 [`EnumerableSet`](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/structs/EnumerableSet.sol)을 사용하세요.

`_removeDocument`에서 요소가 삭제되면 루프를 빠져나오세요:
```diff
        uint256 dHLen = $._docHashes.length;
        for (uint i = 0; i < dHLen; ++i) {
            if ($._docHashes[i] == documentHash) {
                $._docHashes[i] = $._docHashes[dHLen - 1];
                $._docHashes.pop();
+               break;
            }
        }
```

**Remora:** 커밋 [1218d18](https://github.com/remora-projects/remora-smart-contracts/commit/1218d1818e1748cd7d9b71e84485f8059d135ab5)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Check return value when calling `Allowlist::exchangeAllowed` and `RemoraToken::_exchangeAllowed` to prevent unauthorized transfers

**Description:** `Allowlist::exchangeAllowed`는 사용자가 허용되지 않은 경우 revert되지만, 사용자가 둘 다 허용된 경우에는 두 사용자의 `domestic` 필드가 일치하는지에 따라 결정되는 boolean을 반환합니다:
```solidity
function exchangeAllowed(
    address from,
    address to
) external view returns (bool) {
    HolderInfo memory fromUser = _allowed[from];
    HolderInfo memory toUser = _allowed[to];
    if (from != address(0) && !fromUser.allowed)
        revert UserNotRegistered(from);
    if (to != address(0) && !toUser.allowed) revert UserNotRegistered(to);
    return fromUser.domestic == toUser.domestic; //logic to be edited later on
}
```

하지만 `RemoraToken::adminTransferFrom`은 `Allowlist::exchangeAllowed`를 호출할 때 boolean 반환 값을 확인하지 않습니다:
```solidity
function adminTransferFrom(
    address from,
    address to,
    uint256 value,
    bool checkTC,
    bool enforceLock
) external restricted returns (bool) {
    // @audit boolean return not checked
    IAllowlist(allowlist).exchangeAllowed(from, to);
```

유사하게 `RemoraToken::_exchangeAllowed`는 `Allowlist::exchangeAllowed`의 boolean 출력을 반환하지만, 이는 `RemoraToken::transfer`, `transferFrom`에서 확인되지 않습니다.

**Impact:** `Allowlist::exchangeAllowed`가 `false`를 반환하더라도 전송이 허용됩니다.

**Recommended Mitigation:** `Allowlist::exchangeAllowed`, `RemoraToken::_exchangeAllowed`의 boolean 반환 값을 확인하고 `true`를 반환할 때만 전송을 허용하세요.

대안으로 `Allowlist::exchangeAllowed`와 `RemoraToken::_exchangeAllowed`가 아무것도 반환하지 않고 항상 revert되도록 변경하세요.

**Remora:** 커밋 [13cf261](https://github.com/remora-projects/remora-smart-contracts/commit/13cf261d37f5756cac480aa1e0c8ecf756fd3af5)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### `DividendManager::distributePayout` will always revert after 255 payouts, preventing any future payout distributions

**Description:** `DividendManager::HolderManagementStorage::_currentPayoutIndex`는 `uint8`로 선언되어 있습니다:
```solidity
/// @dev The current index that is yet to be paid out.
uint8 _currentPayoutIndex;
```

`_currentPayoutIndex`는 `DividendManager::distributePayout`이 호출될 때마다 증가합니다:
```solidity
$._payouts[$._currentPayoutIndex++] = PayoutInfo({
    amount: payoutAmount,
    totalSupply: SafeCast.toUint128(totalSupply())
});
```

**Impact:** `uint8`의 최대값은 255이므로 `DividendManager::distributePayout`은 255번만 호출될 수 있습니다; 이후의 호출은 항상 revert되어 더 이상의 지불금 분배가 불가능해집니다.

**Recommended Mitigation:** 255번 이상의 지불금 분배가 필요한 경우:
* `DividendManager::HolderManagementStorage::_currentPayoutIndex`를 저장하기 위해 더 큰 크기를 사용하세요.
* 이 매핑의 `uint8` 키를 더 큰 크기와 일치하도록 변경하세요:
```solidity
mapping(address => mapping(uint8 => TokenBalanceChange)) _balanceHistory;
```
* 이 매핑의 `uint256` 키도 일치하도록 변경하세요:
```solidity
mapping(uint256 => PayoutInfo) _payouts;
```

명명된 매핑(named mappings)을 사용하여 이 인덱스들이 모두 동일한 엔티티, 즉 지불 인덱스를 참조함을 명시적으로 보여주는 것을 고려하세요.

Solidity에서 스토리지 슬롯은 256비트이고 주소는 160비트를 사용합니다. `struct HolderManagementStorage`의 관련 스토리지 레이아웃을 조사해보면 `_currentPayoutIndex`는 추가 스토리지 슬롯을 사용하지 않고도 `uint56`만큼 크게 선언될 수 있음을 알 수 있습니다:
```solidity
IERC20 _stablecoin; // 160 bits
uint8 _stablecoinDecimals; // 8 bits
uint32 _payoutFee; // 32 bits
// 200 bits have been used so 56 bits available in the current storage slot
// _currentPayoutIndex could be declared as large as `uint56` with no
// extra storage requirements
uint8 _currentPayoutIndex;
```

**Remora:** 커밋 [f929ff1](https://github.com/remora-projects/remora-smart-contracts/commit/f929ff115f82e76c8eb497ce769792e8de99602b#diff-b6e3759e2288f06f4db11f44b35e7a6398f0301035472704a0479aab4afd9b48R62)에서 `_currentPayoutIndex`를 `uint16`으로 증가시켜 수정되었습니다. 이는 충분할 것입니다.

**Cyfrin:** 검증되었습니다.


### Payout distributions to Remora token holders are diluted by initial token owner mint

**Description:** `RemoraToken::initialize`는 `_initialSupply` 매개변수를 받아 `tokenOwner`에게 발행(mint)합니다:
```solidity
_mint(tokenOwner, _initialSupply * 10 ** decimals());
```

`DividendManager::distributePayout`은 현재 총 공급량과 함께 지불금 금액을 기록합니다:
```solidity
$._payouts[$._currentPayoutIndex++] = PayoutInfo({
    amount: payoutAmount,
    totalSupply: SafeCast.toUint128(totalSupply())
});
```

`DividendManager::payoutBalance`는 기록된 총 공급량으로 나누어 사용자 지불금 금액을 계산합니다:
```solidity
payoutAmount +=
    (curEntry.tokenBalance * pInfo.amount) /
    pInfo.totalSupply;
```

**Impact:** Remora 토큰 홀더들에 대한 지불금 분배는 초기 토큰 소유자(token owner) 배포에 의해 희석됩니다. 토큰 소유자가 공급량의 상당 부분을 발행받으면, 일반 사용자의 보상은 상당히 희석될 것입니다.

**Recommended Mitigation:** 총 공급량에서 토큰 소유자의 토큰을 차감하여 지불금 분배에서 제외하세요. 하지만 토큰 소유자는 자신이 제어하는 다른 주소로 토큰을 전송하여 이를 우회할 수 있습니다.

다른 방법은 `RemoraToken`에 관리자만 설정할 수 있는 변수를 두어 소유자가 보유한 토큰 양을 기록하고, 지불금 분배 목적을 위해 총 공급량에서 이를 차감하는 것입니다. 소유자가 이 변수를 업데이트해야 하므로 여전히 소유자에 대한 신뢰가 필요합니다.

대안으로 프로토콜이 사용자의 보상이 소유자의 보유량에 의해 희석된다는 것을 인지하고 있다면 이 발견 사항을 인정(acknowledge)할 수 있습니다.

**Remora:** 계획은 초기 토큰 발행량 전부를 `TokenBank`로 보내는 것입니다; 의도된 `tokenOwner`는 사실 토큰 은행입니다. 그런 다음 이 토큰들은 투자자들을 위해 판매되므로 프로토콜 관리자는 토큰을 직접 보유하지 않습니다; 토큰은 판매를 기다리는 토큰 은행에 있거나 이를 구매한 투자자에게 있을 것입니다.

`TokenBank`가 보유한 미판매 토큰은 프로토콜 소유입니다; 우리는 토큰 은행이 여전히 보유하고 있는 미판매 토큰에 대한 지불금 분배 몫을 청구하기 위해 포워딩 메커니즘을 사용할 것입니다.


### Burning ALL PropertyTokens of a frozen holder results in the holder losing the payouts distribution while he was frozen

**Description:** 얼어붙은(frozen) 홀더로부터 PropertyToken을 소각(burning)할 때, 얼어붙은 홀더가 소유한 모든 토큰이 소각되는 경우 엣지 케이스가 발생합니다. PropertyToken 소각이 얼어붙은 홀더의 지불금 분배에 미치는 영향은 다음과 같습니다:
- 얼어붙은 홀더가 소유한 모든 PropertyToken이 소각되지 않으면, 홀더는 여전히 얼어있는 동안의 분배에 대한 지불금에 접근할 수 있습니다.
- 얼어붙은 홀더가 소유한 모든 PropertyToken이 소각되면, 홀더는 얼어있는 동안의 분배에 대한 지불금을 잃게 됩니다.

얼어붙은 홀더의 PropertyToken을 소각할 때의 동작 불일치는 얼어붙은 홀더가 지불금을 잃게 만들 수 있는 엣지 케이스를 보여줍니다.

예를 들어 - (홀더들이 동일한 인덱스에서 얼어붙었고 동일한 인덱스에서 토큰이 소각되었다고 가정):
- UserA는 1 PropertyToken을 가지고 있고 얼어붙어 있습니다.
- UserB는 2 PropertyToken을 가지고 있고 얼어붙어 있습니다.
- UserA와 B는 각각 1 PropertyToken씩 소각됩니다.
  - UserA는 얼어붙은 동안의 지불금 분배를 잃는 반면, UserB는 그 분배 동안 소유했던 2 PropertyToken에 대한 지불금에 여전히 접근할 수 있습니다.

**Impact:** 얼어붙은 홀더의 PropertyToken을 '모두' 소각하면 홀더는 얼어있는 동안의 분배에 대한 지불금을 잃게 됩니다.

**Proof of Concept:** Description 섹션에 설명된 이슈를 재현하려면 다음 테스트를 실행하세요.
```solidity
    function test_frozenHolderLosesPayoutsBecauseItsTokensGotBurnt() public {
        address user1 = users[0];
        uint256 amountToMint = 2;

        // fund total payout amount to funding wallet
        uint64 payoutDistributionAmount = 100e6;

        // Distribute payouts for the first 5 distributions
        for(uint i = 1; i <= 5; i++) {
            _fundPayoutToPaymentSettler(payoutDistributionAmount);
        }

        _whitelistAndMintTokensToUser(user1, amountToMint);

        vm.prank(user1);
        remoraTokenProxy.approve(address(this), amountToMint);

        paySettlerProxy.initiateBurning(address(remoraTokenProxy), address(this), 0);

        // verify increased current payout index twice
        assertEq(remoraTokenProxy.getCurrentPayoutIndex(), 5);

        // Distribute payouts for distributions 5 - 10
        for(uint i = 1; i <= 5; i++) {
            _fundPayoutToPaymentSettler(payoutDistributionAmount);
        }

        // Freze user 2 at distributionIndex 10
        remoraTokenProxy.freezeHolder(user1);

        uint256 user1HolderBalanceAfterBeingFrozen = remoraTokenProxy.payoutBalance(user1);

        // Distribute payouts for distributions 10 - 15
        for(uint i = 1; i <= 5; i++) {
            _fundPayoutToPaymentSettler(payoutDistributionAmount);
        }

        // verify increased current payout index twice
        assertEq(remoraTokenProxy.getCurrentPayoutIndex(), 15);

        assertEq(user1HolderBalanceAfterBeingFrozen, remoraTokenProxy.payoutBalance(user1), "Frozen holder earned payout while being frozen");

        uint256 user1PayoutBalance = remoraTokenProxy.payoutBalance(user1);
        // 5 distributions of 100e6 all for user1
        assertEq(user1PayoutBalance, 500e6, "User1 Payout is incorrect");

        vm.prank(user1);
        remoraTokenProxy.claimPayout();
        assertEq(stableCoin.balanceOf(user1), user1PayoutBalance, "Error while claiming payout");
        assertEq(remoraTokenProxy.payoutBalance(user1), 0);

        //@audit-info => When the holder is unfrozen, he gets access to the payouts for al the distributions while he was frozen
        uint256 snapshotBeforeUnfreezing = vm.snapshotState();
        // unfreeze user1 and validate it gets access to all the distributions while it was frozen
        remoraTokenProxy.unFreezeHolder(user1);
        // After being unfrozen there were pending only 5 distributions of 100e6 all for user1
        assertEq(remoraTokenProxy.payoutBalance(user1), 500e6, "User1 Payout is incorrect");
        vm.revertToState(snapshotBeforeUnfreezing);

        uint256 snapshotBeforeBurningAllFrozenHolderTokens = vm.snapshotState();
        //@audit-info => Burning ALL PropertyTokens from a holder while is frozen results in the holder
        // NOT being able to access the payouts of the distributions while he was frozen
        remoraTokenProxy.burnFrom(user1, 2, false);
        assertEq(remoraTokenProxy.balanceOf(user1), 0);

        assertEq(remoraTokenProxy.payoutBalance(user1), 0);
        // unfreeze user1 and validate it loses the payouts of the distributions while it was frozen
        remoraTokenProxy.unFreezeHolder(user1);
        assertEq(remoraTokenProxy.payoutBalance(user1), 0);
        vm.revertToState(snapshotBeforeBurningAllFrozenHolderTokens);

        //@audit-info => Burning NOT ALL PropertyTokens from a holder while is frozen results in the holder
        // being able to access the payouts of the distributions while he was frozen
        assertEq(remoraTokenProxy.payoutBalance(user1), 0);
        remoraTokenProxy.burnFrom(user1, 1, false);
        assertEq(remoraTokenProxy.balanceOf(user1), 1);
        remoraTokenProxy.unFreezeHolder(user1);
        assertEq(remoraTokenProxy.payoutBalance(user1), 500e6);
    }
```

**Recommended Mitigation:** 이 문제를 방지하기 위한 가장 덜 파괴적인 완화 방법은 `burnFrom`에 체크를 추가하여 계정이 더 이상 propertyToken 없이 남겨지고 계정이 얼어있는 경우 tx를 revert하는 것입니다.
```diff
function burnFrom(
        address account,
        uint256 value
    ) external restricted whenBurnable {
        _spendAllowance(account, _msgSender(), value);
        _burn(account, value);
+      if(balanceOf(account) == 0 && isHolderFrozen(account)) revert UserIsFrozen(account);
    }
```
대안으로, burn()과 유사하게, 계정이 얼어있으면 revert하세요.
```diff
    function burnFrom(
        address account,
        uint256 value
    ) external restricted whenBurnable {
+       if (isHolderFrozen(account)) revert UserIsFrozen(account);
        _spendAllowance(account, _msgSender(), value);
        _burn(account, value);
    }
```

**Remora:** 커밋 [6008aec](https://github.com/remora-projects/remora-smart-contracts/commit/6008aecffdd83311e4552e8efee42eae59b3cd30)에서 얼어붙은 사용자에 대한 소각을 방지하여 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Overriding fees can't be switched back once set

**Description:** TokenBank를 통해 PropertyToken을 구매할 때 수수료가 부과됩니다. 시스템은 토큰별(per-token) 수수료 또는 모든 토큰에 부과되는 기본 수수료(baseFee) 부과를 허용합니다.
`overrideFees` 변수가 true로 설정되면, TokenBank는 설정된 토큰별 수수료 대신 모든 토큰에 부과되는 기본 수수료를 부과합니다.

문제는 `overrideFees` 변수가 일단 true로 설정되면, 다시 false로 설정하여 토큰별 수수료 부과를 허용할 수 없다는 것입니다.

```solidity
    function setBaseFee(
        bool updateFee,
        bool overrideFee,
        uint32 newFee
    ) external restricted {
        ...
//@audit => enters only when it is true.
//@audit => So, once set to true, it can't be changed back to false
        if (overrideFee) {
            overrideFees = overrideFee;
            emit FeeOverride(overrideFee);
        }
    }

```

**Impact:** 일단 기본 수수료를 부과하도록 구성되면 토큰별 수수료를 부과할 수 없습니다.

**Recommended Mitigation:** `overrideFees`를 `overrideFee` 매개변수의 값으로 직접 업데이트하세요.
```diff
    function setBaseFee(
        bool updateFee,
        bool overrideFee,
        uint32 newFee
    ) external restricted {
        ...
-       if (overrideFee) {
            overrideFees = overrideFee;
            emit FeeOverride(overrideFee);
-       }
    }
```

**Remora:** 커밋 [c38787f](https://github.com/remora-projects/remora-smart-contracts/commit/c38787f4cfd8272868bc939e73ee3d3d50889740)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Forwarders can lose payouts of the holders forwarding to them

**Description:** 홀더는 자신이 번 지불금을 누적할 지정된 포워더를 가질 수 있습니다.
이 리포트에서 식별된 취약점은 지정된 포워더가 청구할 보류 중인 지불금이 없고 PropertyToken 잔액을 0으로 만든 후, 시간인 흘러(포워더가 여전히 홀더의 지불금을 누적하는 동안) 다시 홀더가 되었지만, 동일한 `distributionIndex`에서 포워더가 다시 잔액을 0으로 만드는 경우입니다.
- 이러한 단계의 조합은 컨트랙트 상태를 일관성 없는 상태로 이끌어 결국 미청구 누적 지불금이 소실되는 결과를 초래합니다.

컨트랙트를 일관성 없는 상태로 이끄는 단계는 다음과 같습니다:
1. 포워더가 잔액을 가지고 있으며, 홀더입니다.
2. 홀더가 포워더를 설정합니다.
3. 홀더에 대한 지불금이 계산되어 포워더에게 입금됩니다.
4. 포워더가 지불금을 청구하고, 모든 보류 중인 지불금이 계산됩니다.
    - 이 시점에서 포워더의 `payoutBalance`는 0이어야 합니다.
5. 포워더가 잔액을 0으로 만들고, [`isHolder` 상태가 제거됩니다(더 이상 홀더가 아님)](https://github.com/remora-projects/remora-smart-contracts/blob/main/contracts/RWAToken/DividendManager.sol#L596-L600).
6. 분배가 지나갑니다.
7. 홀더에 대한 지불금이 계산되어 포워더에게 입금됩니다.
8. 포워더가 잔액을 얻고 홀더 상태를 다시 얻습니다.
    - [lastPayoutIndexCalculated = currentPayoutIndex](https://github.com/remora-projects/remora-smart-contracts/blob/main/contracts/RWAToken/DividendManager.sol#L523-L525)
10. 포워더가 잔액을 다시 0으로 만듭니다.
    - payoutBalance(forwarder)가 호출되지만 [`lastPayoutIndexCalculated == currentPayoutIndex`](https://github.com/remora-projects/remora-smart-contracts/blob/main/contracts/RWAToken/DividendManager.sol#L450-L454)이기 때문에 0을 반환합니다.
    - 따라서, [balance == 0 이고 payoutBalance()가 0을 반환하므로, deleteUser(forwarder)가 호출됩니다](https://github.com/remora-projects/remora-smart-contracts/blob/main/contracts/RWAToken/DividendManager.sol#L547-L550).
11. 10단계의 결과로, 홀더가 번 지불금이 소실됩니다.

**Impact:** 포워더가 홀더의 지불금을 잃을 수 있습니다.

**Proof of Concept:** Description 섹션에 설명된 시나리오를 재현하려면 다음 테스트를 실행하세요.

```solidity
    function test_forwarderLosesHolderPayouts() public {
        address user1 = users[0];
        address forwarder = users[1];
        address anotherUser = users[2];

        uint256 amountToMint = 1;

        _whitelistAndMintTokensToUser(user1, amountToMint);
        _whitelistAndMintTokensToUser(forwarder, amountToMint);
        // Only whitelist and allow anotherUser
        _whitelistAndMintTokensToUser(anotherUser, 0);

        remoraTokenProxy.setPayoutForwardAddress(user1, forwarder);

        // fund total payout amount to funding wallet
        uint64 payoutDistributionAmount = 100e6;

        // Distribute payouts for the first 5 distributions
        for(uint i = 1; i <= 5; i++) {
            _fundPayoutToPaymentSettler(payoutDistributionAmount);
        }

        // user1 must have 0 payout because it is forwarding to `forwarder`
        uint256 user1PayoutBalance = remoraTokenProxy.payoutBalance(user1);
        assertEq(user1PayoutBalance, 0, "Forwarding payout is not working as expected");

        //forwarder must have the full payout for the 5 distributions because user1 is forwarding to him
        uint256 forwarderPayoutBalance = remoraTokenProxy.payoutBalance(forwarder);
        assertEq(forwarderPayoutBalance, payoutDistributionAmount * 5, "Forwarding payout is not working as expected");

        // forwarder claims all his payout
        vm.startPrank(forwarder);
        remoraTokenProxy.claimPayout();
        assertEq(stableCoin.balanceOf(forwarder), forwarderPayoutBalance);

        // forwarder zeros out his PropertyToken's balance
        remoraTokenProxy.transfer(anotherUser, remoraTokenProxy.balanceOf(forwarder));
        vm.stopPrank();

        assertEq(remoraTokenProxy.balanceOf(forwarder), 0);

        (bool isHolder) = remoraTokenProxy.getHolderStatus(forwarder).isHolder;
        assertEq(isHolder, false);

        // Distribute payouts for distributions 5 - 10
        for(uint i = 1; i <= 5; i++) {
            _fundPayoutToPaymentSettler(payoutDistributionAmount);
        }

        // user1 must have 0 payout because it is forwarding to `forwarder`
        user1PayoutBalance = remoraTokenProxy.payoutBalance(user1);
        assertEq(user1PayoutBalance, 0, "Forwarding payout is not working as expected");

        (uint64 calculatedPayout) = remoraTokenProxy.getHolderStatus(forwarder).calculatedPayout;
        assertEq(calculatedPayout, (payoutDistributionAmount * 5) / 2, "Forwarder did not receive payout for holder forwarding to him");

        vm.startPrank(anotherUser);
        // forwarder becomes a holder again
        remoraTokenProxy.transfer(forwarder, remoraTokenProxy.balanceOf(anotherUser));
        vm.stopPrank();

        vm.startPrank(forwarder);
        // forwarder zeroues out his balance again
        remoraTokenProxy.transfer(anotherUser, remoraTokenProxy.balanceOf(forwarder));
        vm.stopPrank();

        //@audit => When this vulnerability is fixed, we expect finalCalculatedPayout to be equals than calculatedPayout!
        (uint64 finalCalculatedPayout) = remoraTokenProxy.getHolderStatus(forwarder).calculatedPayout;
        assertEq(finalCalculatedPayout, 0, "Forwarder did not lose payout of holder");
    }
```

**Recommended Mitigation:** `_updateHolders()`에서 `holderStatus.calculatedPayout`이 0인지 확인하고, 0이 아니면 `deleteUser()`를 호출하지 마세요.
```diff
function _updateHolders(address from, address to) internal {
        ...

        if (from != address(0)) {
            ...
            HolderStatus storage fromHolderStatus = $._holderStatus[from];
-           if (fromBalance == 0 && payoutBalance(from) == 0) {
+           if (fromBalance == 0 && payoutBalance(from) == 0 && fromHolderStatus.calculatedPayout == 0) {
                deleteUser(fromHolderStatus);
                return;
            }
            ...
            }
        }
    }
```

**Remora:** 커밋 [d379e89](https://github.com/remora-projects/remora-smart-contracts/commit/d379e89ac7ae503f6eec775660983c7017a4a513)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Pledge can't successfully complete unless `RemoraToken` is paused

**Description:** 자금 목표에 도달하면 `PledgeManager::checkPledgeStatus`는 `RemoraToken::unpause`를 호출합니다:
```solidity
function checkPledgeStatus() public returns(bool pledgeNowConcluded) {
    if (pledgeRoundConcluded) return true;

    uint32 curTime = SafeCast.toUint32(block.timestamp);
    if (tokensSold >= fundingGoal) {
        pledgeRoundConcluded = true;
        IRemoraRWAToken(propertyToken).unpause();
        emit PledgeHasConcluded(curTime);
        return true;
```

하지만 `RemoraToken`이 일시 중지(paused)되지 않은 경우, `PausableUpgradeable::_unpause`에 `whenPaused` modifier가 있으므로 [revert됩니다](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/master/contracts/utils/PausableUpgradeable.sol#L128).

**Impact:** `RemoraToken`이 일시 중지되지 않으면 서약이 성공적으로 완료될 수 없습니다.

**Proof of Concept:**
```solidity
function test_pledge(uint256 userIndex, uint32 numTokensToBuy) external {
    address user = _getRandomUser(userIndex);
    numTokensToBuy = uint32(bound(numTokensToBuy, 1, DEFAULT_FUNDING_GOAL));

    // fund buyer with stablecoin
    (uint256 finalStablecoinAmount, uint256 fee)
        = pledgeManager.getCost(numTokensToBuy);
    stableCoin.transfer(user, finalStablecoinAmount);

    // fund this with remora tokens
    remoraTokenProxy.mint(address(this), numTokensToBuy);
    assertEq(remoraTokenProxy.balanceOf(address(this)), numTokensToBuy);

    // allow PledgeManager to spend our remora tokens
    remoraTokenProxy.approve(address(pledgeManager), numTokensToBuy);

    PledgeManagerState memory pre = _getState(address(this), user);

    remoraTokenProxy.pause();

    vm.startPrank(user);
    stableCoin.approve(address(pledgeManager), finalStablecoinAmount);
    pledgeManager.pledge(numTokensToBuy, bytes32(0x0), bytes(""));
    vm.stopPrank();

    PledgeManagerState memory post = _getState(address(this), user);

    // verify remora token balances
    assertEq(post.holderRemoraBal, pre.holderRemoraBal - numTokensToBuy);
    assertEq(post.buyerRemoraBal, pre.buyerRemoraBal + numTokensToBuy);

    // verify stablecoin balances
    assertEq(post.pledgeMgrStableBal, pre.pledgeMgrStableBal + finalStablecoinAmount);
    assertEq(post.buyerStableBal, pre.buyerStableBal - finalStablecoinAmount);

    // verify PledgeManager storage
    assertEq(post.pledgeMgrTokensSold, pre.pledgeMgrTokensSold + numTokensToBuy);
    assertEq(post.pledgeMgrTotalFee, pre.pledgeMgrTotalFee + fee);
    assertEq(post.pledgeMgrBuyerFee, pre.pledgeMgrBuyerFee + fee);
    assertEq(post.pledgeMgrTokensBought, pre.pledgeMgrTokensBought + numTokensToBuy);
}
```

**Recommended Mitigation:** `RemoraToken` 컨트랙트는 서약이 완료될 때까지 `paused` 상태로 유지되어야 하지만, 이는 불편할 수 있습니다. 대안으로 `PledgeManager::checkPledgeStatus`가 `RemoraToken`이 일시 중지된 경우에만 unpause하도록 변경하세요.

**Remora:** 커밋 [dddde02](https://github.com/remora-projects/remora-smart-contracts/commit/dddde029b97f33bcb91d0353632a3aa5f028c684)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### `RemoraToken` transfers are bricked when `from` is not whitelisted, has sufficient tokens to transfer but no tokens locked

**Description:** `RemoraToken::adminTransferFrom`, `transfer` 및 `transferFrom`은 항상 `from`이 화이트리스트에 있는지 확인하고, 그렇지 않으면 `_unlockTokens`를 호출합니다:
```solidity
    bool fromWL = whitelist[from];
    bool toWL = whitelist[to];

    if (!fromWL) _unlockTokens(from, value, false);
    if (!toWL) _lockTokens(to, value);
```

그러나 다음과 같은 시나리오가 발생할 수 있습니다:
1) `from`이 잠금 해제(unlock)할 것이 없음
2) `from`이 전송을 수행할 충분한 토큰을 가지고 있음

이 경우 전송이 차단(bricked)됩니다. 고려해야 할 또 다른 시나리오는 다음과 같습니다:

1) `from`이 10개의 토큰을 가지고 있음
2) 그 중 5개만 잠겨 있음
3) `from`이 10개의 토큰을 전송하려고 시도함

이 경우 `from`은 총 10개를 보내기 위해 5개만 잠금 해제하면 되므로 `_unlockTokens` 호출은 `amount = 5`가 되어야 합니다.

하지만 현재 코드에서는 `_unlockTokens` 호출이 `amount = 10`(전송 금액을 그대로 사용하므로)이 되어 말이 되지 않고 revert를 유발합니다.

**Impact:** `from`이 화이트리스트에 없고, 전송할 충분한 토큰이 있지만 잠긴 토큰이 없을 때 `RemoraToken` 전송이 차단(bricked)됩니다. 왜냐하면 `_unlockTokens` 호출이 revert되기 때문입니다.

**Proof Of Concept:**
```solidity
function test_transferBricked_fromNotWhitelisted_ButHasTokensToTransfer() external {
    address from = users[0];
    address to = users[1];
    uint256 amountToTransfer = 1;

    // fund `from` with remora tokens
    remoraTokenProxy.mint(from, amountToTransfer);
    assertEq(remoraTokenProxy.balanceOf(from), amountToTransfer);

    // remove `from` from whitelist
    remoraTokenProxy.removeFromWhitelist(from);

    // set lock time
    remoraTokenProxy.setLockUpTime(3600);

    vm.expectRevert(); // reverts with InsufficientTokensUnlockable
    vm.prank(from);
    remoraTokenProxy.transfer(to, amountToTransfer);
}
```

이것은 또한 화이트리스트에 있을 때 토큰을 받았다가 화이트리스트에서 제거된 사용자의 전송을 완전히 차단합니다:
```solidity
    function test_transferBricked_whitelistedHolderIsRemovedFromWhitelist() external {
        address from = users[0];
        address to = users[1];
        uint256 amountToTransfer = 10;

        _whitelistAndMintTokensToUser(from, amountToTransfer);

        // set lock time
        remoraTokenProxy.setLockUpTime(3600);

        // remove `from` from whitelist
        remoraTokenProxy.removeFromWhitelist(from);

        // fund `from` with remora tokens once it is not whitelisted
        remoraTokenProxy.mint(from, amountToTransfer);

        // 10 when was whitelisted and 10 when from was not whitelisted
        assertEq(remoraTokenProxy.balanceOf(from), amountToTransfer * 2);

        // forward beyond the lockup time to demonstrate the removed whitelisted holder can't do transfers
        vm.warp(3600 + 1);

        // reverts because from has only 10 tokens locked
        vm.expectRevert(); // reverts with InsufficientTokensUnlockable
        vm.prank(from);
        remoraTokenProxy.transfer(to, 11);

        // verify from can only transfer the 10 tokens that he received after he was removed from whitelist
        vm.prank(from);
        remoraTokenProxy.transfer(to, 10);
    }
```

**Recommended Mitigation:** 간단하고 우아한 해결책은 다음과 같을 수 있습니다:
1) `from` 잔액 확인; 전송에 필요한 `amount`보다 작으면 revert
2) 전송 함수에서 사용자가 화이트리스트에 없으면 `uint256 unlockedBalanceToSend = balance - getTokensLocked(sender);`를 계산한 다음 `unlockedBalanceToSend < amount`이면 `_unlockTokens(sender, value - unlockedBalanceToSend..)`를 호출합니다;

이 솔루션은 전송을 수행하는 데 필요한 정확한 양만 잠금 해제하려고 시도하며, 잠금 해제할 것이 없거나 사용자가 전송을 수행할 만큼 충분한 잠금 해제된 토큰을 가지고 있어 필요하지 않은 경우 잠금 해제를 시도하지 않습니다.

**Remora:** 커밋 [67c5e8e](https://github.com/remora-projects/remora-smart-contracts/commit/67c5e8e3262d45f38c18d295a0983dd51f5fd612), [5db7f11](https://github.com/remora-projects/remora-smart-contracts/commit/5db7f11427f6e767a6042c443d552ee9c024494b)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Tokens that were locked when `lockUpTime > 0` will be impossible to unlock if `lockUpTime` is set to zero

**Description:** `LockUpManager::_unlockTokens`는 `lockUpTime == 0`이면 반환합니다:
```solidity
function _unlockTokens(
    address holder,
    uint256 amount,
    bool disregardTime
) internal {
    LockUpStorage storage $ = _getLockUpStorage();
    uint32 lockUpTime = $._lockUpTime;
    // @audit returns if `lockUpTime == 0`
    if (lockUpTime == 0 || amount == 0) return;
```

**Impact:** `lockUpTime > 0`일 때 잠긴 토큰은 `lockUpTime`이 나중에 0으로 설정되면 잠금 해제가 불가능해집니다. 처음에는 문제가 발생하지 않고 사용자가 정상적으로 토큰을 전송할 수 있지만, `lockUpTime`이 0보다 크게 변경되면 사용자가 잠금한 토큰 양이 사용자 토큰 잔액보다 작거나 같아야 한다는 프로토콜 불변성 중 하나를 위반하게 되어 회계 관련 문제가 발생하기 시작합니다.

잠금은 여전히 존재하지만 사용자는 토큰을 전송했을 수 있으므로, 전송 시 잠금 해제된 잔액을 결정할 때 언더플로우 revert가 발생하여 이 불변성이 위반됩니다: `uint256 unlockedBalanceToSend = balance - getTokensLocked(sender);`

**Recommended Mitigation:** `lockUpTime == 0`이라도, 모든 토큰 잠금을 반복하는 `for` 루프를 진행하여 잠금을 해제하세요. 이는 사용자가 잠금한 토큰 양이 사용자 토큰 잔액보다 작거나 같다는 프로토콜 불변성을 유지합니다.

**Remora:** 커밋 [5db7f11](https://github.com/remora-projects/remora-smart-contracts/commit/5db7f11427f6e767a6042c443d552ee9c024494b)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Forwarders who aren't also holders are unable to claim forwarded payouts

**Description:** 또한 홀더가 아닌 포워더는 `DividendManager::payoutBalance`의 이 체크 때문에 포워딩된 지불금을 청구할 수 없습니다:
```solidity
    function payoutBalance(address holder) public returns (uint256) {
        HolderManagementStorage storage $ = _getHolderManagementStorage();
        HolderStatus memory rHolderStatus = $._holderStatus[holder];
        uint16 currentPayoutIndex = $._currentPayoutIndex;

        if (
             // @audit must be a holder to claim payouts, prevents forwarders who aren't
             // also holders from claiming their forwarded payouts
            (!rHolderStatus.isHolder) || //non-holder calling the function
            (rHolderStatus.isFrozen && rHolderStatus.frozenIndex == 0) || //user has been frozen from the start, thus no payout
            rHolderStatus.lastPayoutIndexCalculated == currentPayoutIndex // user has already been paid out up to current payout index
        ) return 0;
```

**Impact:** 또한 홀더가 아닌 포워더는 포워딩된 지불금을 청구할 수 없습니다.

**Recommended Mitigation:** `DividendManager::payoutBalance`에서 `(!rHolderStatus.isHolder)` 체크를 제거하세요.

**Remora:** 커밋 [82fd5d1](https://github.com/remora-projects/remora-smart-contracts/commit/82fd5d1a7d2c6c54790638d63d748eeb8efc870e)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.

\clearpage
## Low Risk


### Use `SafeERC20` functions instead of standard `ERC20` transfer functions

**Description:** 표준 `ERC20` 전송 함수 대신 [`SafeERC20`](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/utils/SafeERC20.sol) 함수를 사용하세요:

```solidity
$ rg "transferFrom" && rg "transfer\("
RWAToken/DividendManager.sol
317:        $._stablecoin.transferFrom(

RWAToken/RemoraToken.sol
220:     * @dev Calls OpenZeppelin ERC20Upgradeable transferFrom function.
251:        return super.transferFrom(from, to, value);
344:            $._stablecoin.transferFrom(sender, $._wallet, _transferFee);
352:     * @dev Calls OpenZeppelin ERC20Upgradeable transferFrom function.
358:    function transferFrom(
376:            $._stablecoin.transferFrom(sender, $._wallet, _transferFee);
379:        return super.transferFrom(from, to, value);

TokenBank.sol
261:        IERC20(stablecoin).transferFrom(

PledgeManager.sol
196:        IERC20(stablecoin).transferFrom(

RemoraIntermediary.sol
172:        IERC20(data.assetReceived).transferFrom(
177:        IERC20(data.assetSold).transferFrom(
197:        IERC20(data.assetReceived).transferFrom(
238:        IERC20(data.paymentToken).transferFrom(
269:            IERC20(data.paymentToken).transferFrom(
296:        IERC20(data.paymentToken).transferFrom(
331:        IERC20(token).transferFrom(payer, recipient, amount);
RWAToken/DividendManager.sol
409:            $._stablecoin.transfer(holder, payoutAmount);
429:        stablecoin.transfer($._wallet, valueToClaim);

RWAToken/RemoraToken.sol
401:        $._stablecoin.transfer(account, burnPayout);

TokenBank.sol
185:        IERC20(tokenAddress).transfer(to, amount);
206:        if (amount != 0) IERC20(stablecoin).transfer(to, amount);
237:        IERC20(stablecoin).transfer(custodialWallet, totalValue);
266:        IERC20(tokenAddress).transfer(to, amount);

PledgeManager.sol
237:                _stablecoin.transfer(feeWallet, feeValue);
239:            _stablecoin.transfer(destinationWallet, amount);
299:        IERC20(stablecoin).transfer(signer, _fixDecimals(refundAmount + fee));
```

**Remora:** 커밋 [f2f3f7e](https://github.com/remora-projects/remora-smart-contracts/commit/f2f3f7e8d51a018417615207152d9fbadf8484eb)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Fee refund can lose precision

**Description:** `PledgeManager::refundTokens`는 사용자 수수료 환불을 다음과 같이 계산합니다:
```solidity
fee = (userPay.fee / userPay.tokensBought) * numTokens;
```

**Impact:** [곱셈 전 나눗셈(division before multiplication)](https://dacian.me/precision-loss-errors#heading-division-before-multiplication)으로 인해 수수료 환불이 원래보다 적을 수 있습니다.

**Recommended Mitigation:** 나눗셈 전에 곱셈을 수행하세요:
```solidity
fee = userPay.fee * numTokens / userPay.tokensBought;
```

**Remora:** 커밋 [b69836f](https://github.com/remora-projects/remora-smart-contracts/commit/b69836f4e7effd2f1cc209608b9149671c79bc18)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### `TokenBank::addToken` should revert if token has already been added

**Description:** `TokenBank::addToken`은 토큰이 이미 추가된 경우 revert되어야 합니다.

**Impact:** `saleAmount` 및 `feeAmount`와 같은 토큰 데이터가 0으로 초기화되어 다른 핵심 기능이 오작동할 수 있습니다.

**Remora:** 2025/07/02 최신 커밋 시점에서 수정되었으나 정확한 커밋은 알 수 없습니다.

**Cyfrin:** 검증되었습니다.


### Changing stablecoin on TokenBank can mess up fees collection

**Description:** 각 토큰의 feeAmount는 현재 스테이블코인(초기에는 6자리 소수의 스테이블코인)의 소수점 자릿수로 계산됩니다. 스테이블코인이 6이 아닌 다른 소수점 자릿수를 사용하는 스테이블코인으로 변경되면, 스테이블코인을 변경하기 전에 보류 중인 수수료가 있다면 그 보류 중인 수수료는 새 스테이블코인으로 지불되어, 실제로 징수된 금액이 예상과 다르게 됩니다.

정상적인 작동 중에 수수료가 청구된 후 스테이블코인이 변경되는 사이에 buyTokens 트랜잭션이 실행될 수 있습니다. 새 토큰 구매가 수수료를 발생시키면, 그 새로운 수수료는 현재 스테이블코인을 기준으로 계산되지만, 새 스테이블코인으로 지불될 것입니다.
예를 들어: 10USDC (10e6)가 보류 중인 수수료이고 새 스테이블코인이 USDT (10e8)라면, 해당 수수료가 징수될 때 0.1USDT가 됩니다.

```solidity
    function buyToken(
        address tokenAddress,
        uint32 amount
    ) external nonReentrant {
        ...
        uint64 feeValue = (stablecoinValue * curData.saleFee) / 1e6;

        ...
        curData.feeAmount += feeValue;

        IERC20(stablecoin).transferFrom(
            to,
            address(this),
            stablecoinValue + feeValue
        );
        ...
    }

    function claimAllFees() external nonReentrant restricted {
       ...
        for (uint i = 0; i < developments.length; ++i) {
//@audit => Pending fees before stablecoin was changed were computed with the decimals of the old stablecoin
            totalValue += tokenData[developments[i]].feeAmount;
            tokenData[developments[i]].feeAmount = 0;
        }
        IERC20(stablecoin).transfer(custodialWallet, totalValue);
    }
```


**Impact:** 스테이블코인이 6이 아닌 다른 소수점 자릿수를 가진 스테이블코인으로 변경되면 징수된 수수료가 예상과 다를 수 있습니다.

**Recommended Mitigation:** PledgeManager에서 값을 현재 스테이블코인의 소수점 자릿수로 정규화하는 방식과 유사하게, TokenBank에도 동일한 로직을 구현하세요.
- 실제 전송을 수행하기 전에 스테이블코인 금액을 정규화하세요.

```diff
    function claimAllFees() external nonReentrant restricted {
        ...
-       IERC20(stablecoin).transfer(custodialWallet, totalValue);
+       IERC20(stablecoin).transfer(custodialWallet, _fixDecimals(totalValue));
        ...
    }

//@audit => Add this function to normalize values to decimals of the current stablecoin
    function _fixDecimals(uint256 value) internal view returns (uint256) {
        return
            stablecoinDecimals < 6
                ? value / (10 ** (6 - stablecoinDecimals))
                : value * (10 ** (stablecoinDecimals - 6));
    }
```

**Remora:** 커밋 [afd07fb](https://github.com/remora-projects/remora-smart-contracts/commit/afd07fb419c354dc223d0105b2fd0c5d565f465f)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### `TokenBank::removeToken` reverts when token balance is zero, making it impossible to remove tokens from the `developments` array

**Description:** TokenBank에서 토큰을 제거할 때, TokenBank 잔액이 0이면 제거 시도가 revert됩니다. 시간이 지남에 따라 `developments` 배열은 제거할 수 없는 불필요한 토큰들로 커질 것입니다.

**Impact:** `developments` 배열이 계속 커져 수수료를 청구할 때 불필요한 가스 낭비를 초래합니다.

**Recommended Mitigation:** `withdrawTokens()`를 호출하기 전에 tokenAddress의 잔액이 0인지 확인하고, 그렇다면 호출을 건너뛰세요(어차피 revert될 것이므로):
```diff
    function removeToken(address tokenAddress) external restricted {
        ...
-        withdrawTokens(tokenAddress, custodialWallet, 0);
+       if(IERC20(tokenAddress).balanceOf(address(this)) > 0) withdrawTokens(tokenAddress, custodialWallet, 0);
        ...
    }
```

**Remora:** 커밋 [2e797fc](https://github.com/remora-projects/remora-smart-contracts/commit/2e797fc45245e3cc731e7f2f70401e8df831f1f3)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Zero token transfers record receiving user as a holder in `DividendManager::HolderStatus` even if they have zero token balance

**Description:** 0 토큰 전송은 받는 사용자가 토큰 잔액이 0이더라도 `DividendManager::HolderStatus`에 홀더로 기록되게 합니다.

**Proof of Concept:** 먼저 `DividendManager`에 이 두 함수를 추가하세요:
```solidity
function getCurrentPayoutIndex() view external returns(uint8 currentPayoutIndex) {
    currentPayoutIndex = _getHolderManagementStorage()._currentPayoutIndex;
}

function getHolderStatus(address holder) view external returns(HolderStatus memory status) {
    status = _getHolderManagementStorage()._holderStatus[holder];
}
```

그 다음 PoC:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.22;

import {RemoraToken, DividendManager} from "../../../contracts/RWAToken/RemoraToken.sol";
import {Stablecoin} from "../../../contracts/ForTestingOnly/Stablecoin.sol";
import {AccessManager} from "../../../contracts/AccessManager.sol";
import {Allowlist} from "../../../contracts/Allowlist.sol";

import {UnitTestBase} from "../UnitTestBase.sol";

import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";

contract RemoraTokenTest is UnitTestBase {
    // contract being tested
    RemoraToken remoraTokenProxy;
    RemoraToken remoraTokenImpl;

    // required support contracts & variables
    Stablecoin internal stableCoin;
    AccessManager internal accessMgrProxy;
    AccessManager internal accessMgrImpl;
    Allowlist internal allowListProxy;
    Allowlist internal allowListImpl;
    address internal withdrawalWallet;
    uint32 internal constant DEFAULT_PAYOUT_FEE = 100_000; // 10%

    function setUp() public override {
        // test harness setup
        UnitTestBase.setUp();

        // support contracts / variables setup
        stableCoin = new Stablecoin("USDC", "USDC", type(uint256).max/1e6, 6);
        assertEq(stableCoin.balanceOf(address(this)), type(uint256).max/1e6*1e6);

        // contract being tested setup
        accessMgrImpl = new AccessManager();
        ERC1967Proxy proxy1 = new ERC1967Proxy(address(accessMgrImpl), "");
        accessMgrProxy = AccessManager(address(proxy1));
        accessMgrProxy.initialize(address(this));

        allowListImpl = new Allowlist();
        ERC1967Proxy proxy2 = new ERC1967Proxy(address(allowListImpl), "");
        allowListProxy = Allowlist(address(proxy2));
        allowListProxy.initialize(address(accessMgrProxy), address(this));

        withdrawalWallet = makeAddr("WITHDRAWAL_WALLET");

        // contract being tested setup
        remoraTokenImpl = new RemoraToken();
        ERC1967Proxy proxy3 = new ERC1967Proxy(address(remoraTokenImpl), "");
        remoraTokenProxy = RemoraToken(address(proxy3));
        remoraTokenProxy.initialize(
            address(this), // tokenOwner
            address(accessMgrProxy), // initialAuthority
            address(stableCoin),
            withdrawalWallet,
            address(allowListProxy),
            "REMORA",
            "REMORA",
            0
        );

        assertEq(remoraTokenProxy.authority(), address(accessMgrProxy));
    }

    function test_transferZeroTokens_RegistersHolderWithDividendManager() external {
        address user1 = users[0];
        address user2 = users[1];

        uint256 user1RemoraTokens = 1;

        // whitelist both users
        remoraTokenProxy.addToWhitelist(user1);
        assertTrue(remoraTokenProxy.isWhitelisted(user1));
        remoraTokenProxy.addToWhitelist(user2);
        assertTrue(remoraTokenProxy.isWhitelisted(user2));

        // mint user1 their tokens
        remoraTokenProxy.mint(user1, user1RemoraTokens);
        assertEq(remoraTokenProxy.balanceOf(user1), user1RemoraTokens);

        // allowlist both users
        allowListProxy.allowUser(user1, true, true, false);
        assertTrue(allowListProxy.allowed(user1));
        allowListProxy.allowUser(user2, true, true, false);
        assertTrue(allowListProxy.allowed(user2));
        assertTrue(allowListProxy.exchangeAllowed(user1, user2));

        // user1 transfers zero tokens to user2
        vm.prank(user1);
        remoraTokenProxy.transfer(user2, 0);

        // fetch user2's HoldStatus from DividendManager
        DividendManager.HolderStatus memory user2Status = remoraTokenProxy.getHolderStatus(user2);

        // user2 is listed as a holder even though they have no tokens!
        assertEq(user2Status.isHolder, true);
        assertEq(remoraTokenProxy.balanceOf(user2), 0);
    }
}
```

**Recommended Mitigation:** `RemoraToken::_update` 내의 0 토큰 전송에서 revert하거나 잔액이 0인 경우 `to`를 홀더로 설정하지 않도록 `DividendManager::_updateHolders`를 변경하세요.

**Remora:** 커밋 [0a2dea2](https://github.com/remora-projects/remora-smart-contracts/commit/0a2dea21b8dec5fa63dd2402b987b4f77c3e60b1)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### `DividendManager::distributePayout` records a new payout record increasing the current payout index for zero `payoutAmount`

**Description:** `DividendManager::distributePayout`은 0 `payoutAmount`에 대해 현재 지불 인덱스를 증가시키는 새로운 지불금 기록을 남깁니다.

**Proof of Concept:**
```solidity
function test_distributePayout_ZeroPayout() external {
    // setup one user
    address user = users[0];
    uint256 userRemoraTokens = 1;

    // whitelist user
    remoraTokenProxy.addToWhitelist(user);
    assertTrue(remoraTokenProxy.isWhitelisted(user));

    // mint user their tokens
    remoraTokenProxy.mint(user, userRemoraTokens);
    assertEq(remoraTokenProxy.balanceOf(user), userRemoraTokens);

    // allowlist user
    allowListProxy.allowUser(user, true, true, false);
    assertTrue(allowListProxy.allowed(user));

    // zero payout distribution - should revert here
    remoraTokenProxy.distributePayout(0);
    // didn't revert but increased current payout index
    assertEq(remoraTokenProxy.getCurrentPayoutIndex(), 1);
}
```

**Recommended Mitigation:** `DividendManager::distributePayout`은 `payoutAmount == 0`일 때 revert되어야 합니다.

**Remora:** 커밋 [c2002fb](https://github.com/remora-projects/remora-smart-contracts/commit/c2002fb751c22d6f4009eaa6f3dd651b5b4ca474)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### `PaymentSettler::claimAllPayouts` doesn't validate input `tokens` addresses are legitimate contracts before calling `adminClaimPayout` on them

**Description:** `PaymentSettler::initiateBurning` 및 `distributePayment`에서는 입력 `token` 주소에서 함수를 호출하기 전에 합법적인 주소인지 확인하는 다음 체크가 수행됩니다:
```solidity
if (!tokenData[token].active) revert InvalidTokenAddress();
```

하지만 `PaymentSettler::claimAllPayouts`에서는 이 체크가 수행되지 않습니다:
```solidity
function claimAllPayouts(address[] calldata tokens) external nonReentrant {
    address investor = msg.sender;
    uint256 totalPayout = 0;
    for (uint i = 0; i < tokens.length; ++i) {
        TokenData storage curToken = tokenData[tokens[i]];
        uint256 amount = IRemoraToken(tokens[i]).adminClaimPayout(
            investor,
            true
        );
```

**Impact:** 공격자가 `adminClaimPayout` 함수 인터페이스를 구현하는 자신의 컨트랙트를 배포할 수 있지만, 이 함수는 임의의 코드를 포함할 수 있습니다; 실행 흐름이 공격자의 컨트랙트로 전달됩니다. 이를 더 악용할 방법을 찾지는 못했지만 공격자가 자신의 커스텀 컨트랙트로 실행 흐름을 가로채도록 허용하는 것은 좋은 관행이 아닙니다.

**Recommended Mitigation:** 함수를 호출하기 전에 입력 `tokens`가 유효한 `RemoraToken` 컨트랙트인지 검증하세요.

**Remora:** 커밋 [4ba903e](https://github.com/remora-projects/remora-smart-contracts/commit/4ba903e52438e9570940c11ae8c39acf07256b6a)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Minting new PropertyTokens close to the end of the distribution period will dilute rewards for holders who were holding for the full period

**Description:** 홀더의 balanceHistory는 토큰 전송(발행 또는 소각 포함)이 발생할 때마다 업데이트됩니다. [회계는 특정 인덱스 동안의 홀더 잔액을 저장하며](https://github.com/remora-projects/remora-smart-contracts/blob/main/contracts/RWAToken/DividendManager.sol#L533-L540), 이는 해당 인덱스에 대해 분배된 스테이블코인 금액에서 얻은 지불금을 결정하는 데 사용됩니다.

```solidity
    function _updateHolders(address from, address to) internal {
        ...
            } else {
                // else update status data and create new entry
                tHolderStatus.mostRecentEntry = payoutIndex;
//@audit => saves the holder balance for the current payout, regardless of how much time is left for the distribution to end
                $._balanceHistory[to][payoutIndex] = TokenBalanceChange({
                    isValid: true,
                    tokenBalance: toBalance
                });
            }
        }

```

새로운 PropertyToken 발행은 모든 홀더에게 분배된 동일한 양의 지불금이 각 PropertyToken에 더 적은 지불금을 주게 됨을 의미합니다. 문제는 분배 기간이 끝나갈 무렵에 발생하는 새로운 발행이 전체 기간 동안 PropertyToken을 보유했던 홀더들의 지불금을 희석시킨다는 것입니다.

**Impact:** 전체 분배 기간 동안 PropertyToken을 보유했던 홀더들은 기간 종료에 근접하여 발행된 새로운 PropertyToken에 의해 보상이 희석될 것입니다.

**Recommended Mitigation:** [mint()](https://github.com/remora-projects/remora-smart-contracts/blob/main/contracts/RWAToken/RemoraToken.sol#L268-L270)에서 현재 분배 기간 동안 얼마나 많은 시간이 지났는지 계산하고, 특정 임계값(아마도 75-80%)이 지났다면 다음 분배 기간까지 새로운 발행을 허용하지 마세요.

**Remora:** 인정함(Acknowledged).


### Forwarder can be frozen and still receive and claim payouts while frozen

**Description:** 첫 번째 지불금 전에 포워더가 얼어붙으면(frozen), 얼어있는 동안 어떤 지불금도 받지 못하고 청구할 수 없습니다.

하지만 첫 번째 지불금 후에 포워더가 얼어붙으면, 얼어있는 동안에도 지불금을 받고 청구할 수 있습니다.

**Proof of Concept:**
```solidity
function test_forwarderFrozenBeforeFirstPayout_noPayoutBalanceWhileFrozen() external {
    address user1 = users[0];
    address forwarder = users[1];
    uint256 amountToMint = 1;

    _whitelistAndMintTokensToUser(user1, amountToMint);
    _whitelistAndMintTokensToUser(forwarder, amountToMint);

    remoraTokenProxy.setPayoutForwardAddress(user1, forwarder);

    // forwarder frozen before first distribution payout
    remoraTokenProxy.freezeHolder(forwarder);

    uint64 payoutDistributionAmount = 100e6;
    _fundPayoutToPaymentSettler(payoutDistributionAmount);

    // user1 must have 0 payout because it is forwarding to `forwarder`
    uint256 user1PayoutBalance = remoraTokenProxy.payoutBalance(user1);
    assertEq(user1PayoutBalance, 0);

    // forwarder is frozen yet so receives no forwarded payout
    uint256 forwarderPayoutBalancePreUnfreeze = remoraTokenProxy.payoutBalance(forwarder);
    assertEq(forwarderPayoutBalancePreUnfreeze, 0);
}

function test_forwarderFrozenAfterFirstPayout_validPayoutBalanceWhileFrozen_claimPayoutWhileFrozen() external {
    _fundPayoutToPaymentSettler(1);

    address user1 = users[0];
    address forwarder = users[1];
    uint256 amountToMint = 1;

    _whitelistAndMintTokensToUser(user1, amountToMint);
    _whitelistAndMintTokensToUser(forwarder, amountToMint);

    remoraTokenProxy.setPayoutForwardAddress(user1, forwarder);

    remoraTokenProxy.freezeHolder(forwarder);

    uint64 payoutDistributionAmount = 100e6;
    _fundPayoutToPaymentSettler(payoutDistributionAmount);

    // user1 must have 0 payout because it is forwarding to `forwarder`
    uint256 user1PayoutBalance = remoraTokenProxy.payoutBalance(user1);
    assertEq(user1PayoutBalance, 0);

    // forwarder is frozen yet still receives forwarded payout
    uint256 forwarderPayoutBalance = remoraTokenProxy.payoutBalance(forwarder);
    assertEq(forwarderPayoutBalance, payoutDistributionAmount/2);

    // forwarder claims all their payout
    vm.prank(forwarder);
    remoraTokenProxy.claimPayout();
    assertEq(stableCoin.balanceOf(forwarder), forwarderPayoutBalance);
}
```

**Recommended Mitigation:** 완화 방법은 이 경우 프로토콜이 무엇을 원하는지, 그리고 L-11을 완화할 계획인지 여부에 따라 다릅니다. 프로토콜이 얼어붙은 주소도 포워딩 주소 역할을 하고 지불금 잔액을 갖도록 허용하려면 다음과 같이 달성할 수 있습니다:
```diff
    function payoutBalance(address holder) public returns (uint256) {
        HolderManagementStorage storage $ = _getHolderManagementStorage();
        HolderStatus memory rHolderStatus = $._holderStatus[holder];
        uint16 currentPayoutIndex = $._currentPayoutIndex;

+       if ((rHolderStatus.isFrozen && rHolderStatus.frozenIndex == 0) && rHolderStatus.calculatedPayout > 0) return rHolderStatus.calculatedPayout;

        if (
            (!rHolderStatus.isHolder) || //non-holder calling the function
            (rHolderStatus.isFrozen && rHolderStatus.frozenIndex == 0) || //user has been frozen from the start, thus no payout
            rHolderStatus.lastPayoutIndexCalculated == currentPayoutIndex // user has already been paid out up to current payout index
        ) return 0;
```

**Remora:** 인정함; 포워딩 메커니즘은 `TokenBank`와 유동성 풀이 보유한 토큰의 지불금 분배를 포워딩하기 위해 Remora 프로토콜 소유 주소에서만 사용하도록 의도되었습니다. 따라서 동결 및 포워딩 메커니즘이 상호 작용할 가능성은 거의 없습니다.


### Forwarder can be set to frozen address

**Description:** 포워더를 얼어붙은 주소로 설정할 수 있습니다; 이는 이상적이지 않으며 최악의 시나리오에서는 토큰 손실을 초래할 수 있습니다.

**Proof of Concept:**
```solidity
    function test_freezeHolder_setPayoutForwardAddress_toFrozenForwarder() external {
        address user1 = users[0];
        address forwarder = users[1];
        uint256 amountToMint = 1;

        _whitelistAndMintTokensToUser(user1, amountToMint);
        _whitelistAndMintTokensToUser(forwarder, amountToMint);

        // freeze forwarder
        remoraTokenProxy.freezeHolder(forwarder);

        // forward user1 payouts to frozen address
        remoraTokenProxy.setPayoutForwardAddress(user1, forwarder);

        uint64 payoutDistributionAmount = 100e6;
        _fundPayoutToPaymentSettler(payoutDistributionAmount);

        // user1 must have 0 payout because it is forwarding to `forwarder`
        uint256 user1PayoutBalance = remoraTokenProxy.payoutBalance(user1);
        assertEq(user1PayoutBalance, 0);

        // forwarder is frozen  so receives no forwarded payout
        uint256 forwarderPayoutBalance = remoraTokenProxy.payoutBalance(forwarder);
        assertEq(forwarderPayoutBalance, 0);

        // remove the forwarding while forwarder still frozen
        remoraTokenProxy.removePayoutForwardAddress(user1);

        // user1 can't claim any tokens - user1 has lost their payouts
        user1PayoutBalance = remoraTokenProxy.payoutBalance(user1);
        assertEq(user1PayoutBalance, 0);
    }
```

**Recommended Mitigation:** `forwardingAddress`가 얼어붙은 경우 `DividendManager::setPayoutForwardAddress`는 revert되어야 합니다. 주소가 동결될 때 해당 주소로 포워딩하는 사용자가 있다면 모든 포워딩을 취소하는 것을 고려하세요.

**Remora:** 인정함; 포워딩 메커니즘은 `TokenBank`와 유동성 풀이 보유한 토큰의 지불금 분배를 포워딩하기 위해 Remora 프로토콜 소유 주소에서만 사용하도록 의도되었습니다. 따라서 동결 및 포워딩 메커니즘이 상호 작용할 가능성은 거의 없습니다.


### Impossible to remove a document added with zero uri length

**Description:** uri 길이가 0으로 추가된 문서는 제거할 수 없습니다.

**Proof of Concept:** "Don't add duplicate documentHash to DocumentManager::DocumentStorage::_docHashes when overwriting via _setDocument as this causes panic revert when calling _removeDocument" 버그를 수정한 후 다음 퍼즈 테스트를 실행하세요:
```solidity
    function test_setDocumentOverwrite(string calldata uri) external {
        bytes32 docName = "0x01234";
        bytes32 docHash = "0x5555";
        bool needSignature = true;

        // add the document
        _setDocument(docName, uri, docHash, needSignature);

        // verify its hash has been added to `_docHashes`
        DocumentStorage storage $ = _getDocumentStorage();
        assertEq($._docHashes.length, 1);
        assertEq($._docHashes[0], docHash);

        // ovewrite it
        _setDocument(docName, uri, docHash, needSignature);
        // verify overwriting doesn't duplicate the hash in `_docHashes`
        assertEq($._docHashes.length, 1);
        assertEq($._docHashes[0], docHash);

        // now attempt to remove it
        _removeDocument(docHash);
    }
```

마지막에 `_removeDocument`를 호출할 때 `[FAIL: EmptyDocument();`로 revert됩니다.

**Recommended Mitigation:** 빈 uri로 문서를 추가하는 것을 허용하지 마세요.

**Remora:** 커밋 [1218d18](https://github.com/remora-projects/remora-smart-contracts/commit/1218d1818e1748cd7d9b71e84485f8059d135ab5)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### `PledgeManager::pricePerToken` can only support a maximum price of `$4294`

**Description:** `PledgeManager::pricePerToken`은 `uint32`를 사용하며 주석에 6자리 소수 정밀도의 USD 가격을 나타낸다고 명시되어 있습니다:
```solidity
    uint32 public pricePerToken; //in usd (6 decimals)
```

**Impact:** `uint32`의 최대값은 4294967295이므로, 6자리 소수 정밀도로 최대 USD `pricePerToken`은 `$4294`로 제한됩니다.

부동산을 토큰화하는 것이 목표이므로 이 금액은 수백만 달러의 가치가 있을 수 있어 불충분할 수 있습니다.

**Recommended Mitigation:** 큰 USD 가격을 지원해야 하는 경우 `pricePerToken`을 저장하기 위해 더 큰 크기를 사용하세요.

**Remora:** 인정함; 우리는 가격을 토큰당 약 $50 정도로 낮게 유지할 계획입니다.

\clearpage
## Informational


### Emit missing events for storage changes

**Description:** 스토리지 변경에 대해 누락된 이벤트를 발생시키세요:
* `Allowlist::changeUserAccreditation`, `changeAdminStatus`
* `Allowlist::UserAllowed`는 `HolderInfo` boolean 플래그를 포함하고 발생시키도록 확장되어야 함
* `ReferralManager::addReferral`
* `RemoraIntermediary::setFundingWallet`, `setFeeRecipient`
* `TokenBank::changeReferralManager`, `changeStablecoin`, `changeCustodialWallet`
* `TokenBank::TokensWithdrawn`은 출금된 `amount`를 포함해야 함
* `DividendManager::setPayoutForwardAddress`, `changeWallet`
* `RemoraToken::addToWhitelist`, `removeFromWhitelist`, `updateAllowList`
* `PaymentSettler::withdraw`, `withdrawAllFees`는 출금된 금액을 발생시켜야 함, `addToken`, `changeCustodian`,  `changeStablecoin`

**Remora:** 커밋 [9051af8](https://github.com/remora-projects/remora-smart-contracts/commit/9051af840f92c7aee37b95a4d8f206d0f16abc93)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Rename `isAllowed` to `wasAllowed` in `Allowlist::allowUser`, `disallowUser`

**Description:** `Allowlist::allowUser`, `disallowUser`는 기존 `allowed` 상태를 `isAllowed`라는 명명된 반환 변수로 반환한 후, 잠재적으로 `allowed` 상태를 수정합니다.

이 함수들은 `allowed` 상태를 수정할 수 있으므로, 반환된 상태가 현재 상태가 아닐 수 있음을 명시적으로 나타내기 위해 명명된 반환 변수의 이름을 `wasAllowed`로 변경해야 합니다.

**Remora:** 커밋 [06d17a6](https://github.com/remora-projects/remora-smart-contracts/commit/06d17a68320472e44a46c59ff3623af3741f691a)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Use constants instead of magic numbers

**Description:** 매직 넘버 대신 상수를 사용하세요; 1000000, 1e6 및 10 ** 6은 모두 동일하며 다양한 파일로 import되는 상수로 선언되어야 합니다.
```solidity
TokenBank.sol
111:        if (newFee > 1000000) revert InvalidValuePassed();
252:        uint64 feeValue = (stablecoinValue * curData.saleFee) / 1e6;

PledgeManager.sol
151:        require(newPenalty <= 1000000);
157:        require(newFee <= 1000000);
180:        uint256 fee = (stablecoinAmount * pledgeFee) / 1e6;
283:            refundAmount -= (refundAmount * earlySellPenalty) / 1e6;

RWAToken/DividendManager.sol
230:        require(newFee <= 1e6);
416:        payoutAmount -= (payoutAmount * fee) / (10 ** 6);

RWAToken/RemoraToken.sol
306:            require(newBurnFee <= 1e6);
398:        if (burnFee != 0) burnPayout -= (burnPayout * burnFee) / 1e6;
```

**Remora:** 커밋 [aaaab45](https://github.com/remora-projects/remora-smart-contracts/commit/aaaab4558bd370173de2d6f11697ecfd5f097072)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Using explicit unsigned integer sizing instead of `uint`

**Description:** Solidity에서 `uint`는 자동으로 `uint256`에 매핑되지만 변수를 선언할 때 정확한 크기를 지정하는 것이 좋은 관행으로 간주됩니다:
```solidity
PaymentSettler.sol
124:        for (uint i = 0; i < len; ++i) {
174:        for (uint i = 0; i < tokens.length; ++i) {
227:        for (uint i = 0; i < tokenList.length; ++i) {

RWAToken/DocumentManager.sol
224:        for (uint i = 0; i < dHLen; ++i) {

TokenBank.sol
185:        for (uint i = 0; i < len; ++i) {
260:        for (uint i = 0; i < developments.length; ++i)
267:        for (uint i = 0; i < developments.length; ++i) {
```

**Remora:** 커밋 [6602423](https://github.com/remora-projects/remora-smart-contracts/commit/66024232cfea24b69bd055086f8088d40f3d1d4a)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Retrieve and enforce token decimal precision

**Description:** [`IERC20Metadata`](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/extensions/IERC20Metadata.sol)를 사용하여 토큰 소수점 정밀도를 검색하고 강제하세요. 예를 들어:

1) `PledgeManager::initialize`
```diff
    constructor(
        address authority,
        address _holderWallet,
        address _propertyToken,
        address _stablecoin,
-       uint16 _stablecoinDecimals,
        uint32 _fundingGoal,
        uint32 _deadline,
        uint32 _withdrawDuration,
        uint32 _pledgeFee,
        uint32 _earlySellPenalty,
        uint32 _pricePerToken
    ) AccessManaged(authority) ReentrancyGuardTransient() {
        holderWallet = _holderWallet;
        propertyToken = _propertyToken;
        stablecoin = _stablecoin;
-       stablecoinDecimals = _stablecoinDecimals;
+       stablecoinDecimals = IERC20Metadata(_stablecoin).decimals();
        fundingGoal = _fundingGoal;
        deadline = _deadline;
        postDeadlineWithdrawPeriod = _withdrawDuration;
        pledgeFee = _pledgeFee;
        earlySellPenalty = _earlySellPenalty;
        pricePerToken = _pricePerToken;
        tokensSold = 0;
    }
```

2) `TokenBank::initialize`
```diff
        stablecoin = _stablecoin; //must be 6 decimal stablecoin
+       require(IERC20Metadata(stablecoin).decimals() == 6, "Wrong decimals");
```

**Remora:** 이는 내부 Remora 정밀도에서 외부 스테이블코인 정밀도로 항상 변환하는 `remoraToNativeDecimals`를 추가하여 해결되었으므로, 프로토콜은 이제 다른 소수점 스테이블코인과 함께 작동할 수 있습니다.


### `LockUpManager::LockUpStorage::_regLockUpTime` is never used

**Description:** `LockUpManager::LockUpStorage::_regLockUpTime`은 절대 사용되지 않습니다:
```solidity
$ rg "_regLockUpTime"
RWAToken/LockUpManager.sol
36:        uint32 _regLockUpTime; //lock up time for foreign to domestic trades
```

제거하거나, 나중에 사용될 예정이지만 현재는 사용되지 않는다는 주석을 추가하세요.

**Remora:** 커밋 [91aed23](https://github.com/remora-projects/remora-smart-contracts/commit/91aed23b0a372a0aa3a7eac6e8e4a98563cea615)에서 나중에 사용될 예정임을 알리는 주석을 추가하여 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Use `SignatureChecker` library and optionally support `EIP7702` accounts which use their private key to sign

**Description:** `DocumentManager::verifySignature`에서 [SignatureChecker](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/cryptography/SignatureChecker.sol) 라이브러리를 사용하세요:
```diff
-        if (signer.code.length == 0) {
-            //signer is EOA
-            (address returnedSigner, , ) = ECDSA.tryRecover(digest, signature);
-            result = returnedSigner == signer;
-        } else {
-            //signer is SCA
-            (bool success, bytes memory ret) = signer.staticcall(
-                abi.encodeWithSelector(
-                    bytes4(keccak256("isValidSignature(bytes32,bytes)")),
-                    digest,
-                    signature
-                )
-            );
-            result = (success && ret.length == 32 && bytes4(ret) == MAGICVALUE);
-        }

-        if (!result) revert InvalidSignature();
+        if(!SignatureChecker.isValidSignatureNow(signer, digest, signature)) revert InvalidSignature();
```

또한 [EIP7702](https://eip7702.io/)를 사용하면 주소의 `code.length > 0`이지만 여전히 개인 키를 사용하여 서명하는 것이 가능해졌으므로, 현재 코드나 위의 권장 사항으로는 이 시나리오가 지원되지 않습니다.

이 시나리오를 지원하려면, [LCDSA.tryRecover](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/cryptography/ECDSA.sol#L56)를 먼저 호출한 다음 작동하지 않으면 `SignatureChecker::isValidERC1271SignatureNow`를 백업 옵션으로 호출하여 이 시나리오를 지원하는 최근 감사의 [이 발견 사항](https://solodit.remora-projects.io/issues/verifysignature-is-not-compatible-with-smart-contract-wallets-or-other-smart-accounts-remora-projects-none-evo-soulboundtoken-markdown)을 확인하세요:
```diff
+        (address recovered, ECDSA.RecoverError error,) = ECDSA.tryRecover(digest, signature);
​
+        if (error == ECDSA.RecoverError.NoError && recovered == signer) result = true;
+        else result = SignatureChecker.isValidERC1271SignatureNow(signer, digest, signature);

+        if (!result) revert InvalidSignature();
```

**Remora:** 커밋 [b545498](https://github.com/remora-projects/remora-smart-contracts/commit/b545498ed931eb63ae0ec7f6fb3297ce25886281)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Use `EIP712Upgradeable` library to simplify `DocumentManager`

**Description:** `DocumentManager`를 단순화하기 위해 [`EIP712Upgradeable`](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/master/contracts/utils/cryptography/EIP712Upgradeable.sol) 라이브러리를 사용하세요. 이 라이브러리는 도메인 분리자(domain separator)와 유용한 함수 `_hashTypedDataV4`를 제공합니다.

`EIP712Upgradeable`을 상속하고, 제공되는 모든 중복 코드를 제거한 다음 `verifySignature`에서 다음과 같이 하세요:
```diff
-        bytes32 digest = keccak256(
-            abi.encodePacked("\x19\x01", _DOMAIN_SEPARATOR, structHash)
-        );
+        bytes32 digest = _hashTypedDataV4(structHash);
```

**Remora:** 커밋 [b545498](https://github.com/remora-projects/remora-smart-contracts/commit/b545498ed931eb63ae0ec7f6fb3297ce25886281)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Remove unnecessary imports and inheritance

**Description:** 불필요한 import 및 상속을 제거하세요:
* `BurnStateManager`는 `Initializable`만 import하고 상속해야 합니다.

**Remora:** 커밋 [8419903](https://github.com/remora-projects/remora-smart-contracts/commit/84199034d7255cfc90cd6eef502616a874f26908)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.

\clearpage
## Gas Optimization


### Fail fast without performing unnecessary storage reads

**Description:** 불필요한 스토리지 읽기를 수행하지 말고 빠르게 실패하세요:

* `Allowlist::exchangeAllowed` - `from`이 허용되지 않아 트랜잭션이 실패할 경우 스토리지에서 `_allowed[to]`를 읽지 마세요:
```solidity
    function exchangeAllowed(
        address from,
        address to
    ) external view returns (bool) {
        HolderInfo memory fromUser = _allowed[from];
        if (from != address(0) && !fromUser.allowed) revert UserNotRegistered(from);

        HolderInfo memory toUser = _allowed[to];
        if (to != address(0) && !toUser.allowed) revert UserNotRegistered(to);

        return fromUser.domestic == toUser.domestic; //logic to be edited later on
    }
```

* `Allowlist::hasTradeRestriction` - `_allowed[user1]`이 허용되지 않아 트랜잭션이 실패할 경우 스토리지에서 `_allowed[user2]`를 읽지 마세요:
```solidity
    function hasTradeRestriction(
        address user1,
        address user2
    ) public returns (bool) {
        HolderInfo memory u1Data = _allowed[user1];
        if (!u1Data.allowed) revert UserNotRegistered(user1);

        HolderInfo memory u2Data = _allowed[user2];
        if (!u2Data.allowed) revert UserNotRegistered(user2);
```

**Remora:** 커밋 [81faadb](https://github.com/remora-projects/remora-smart-contracts/commit/81faadbd3baa1deec950db08abf635c5a277a7c5), [760469f](https://github.com/remora-projects/remora-smart-contracts/commit/760469fb5a0263ab5527d7c35e7887349f1a2368)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Don't initialize to default values

**Description:** 기본값으로 초기화하지 마세요:
```solidity
RWAToken/DividendManager.sol
186:      $._currentPayoutIndex = 0;

RWAToken/DocumentManager.sol
118:        for (uint256 i = 0; i < numDocs; ++i) {
222:        for (uint i = 0; i < dHLen; ++i) {

RWAToken/DividendManager.sol
349:                for (uint256 i = 0; i < len; ++i) {
463:        for (uint256 i = 0; i < rHolderStatus.forwardedPayouts.length; ++i) {

TokenBank.sol
152:        for (uint i = 0; i < developments.length; ++i) {
225:        uint64 totalValue = 0;
226:        for (uint i = 0; i < developments.length; ++i)
232:        uint64 totalValue = 0;
233:        for (uint i = 0; i < developments.length; ++i) {

PledgeManager.sol
119:        tokensSold = 0;
278:        uint256 fee = 0;

RemoraIntermediary.sol
258:        for (uint256 i = 0; i < len; ++i) {

PaymentSettler.sol
124:        for (uint i = 0; i < len; ++i) {
174:        for (uint i = 0; i < tokens.length; ++i) {
226:        uint256 totalFees = 0;
227:        for (uint i = 0; i < tokenList.length; ++i) {
```

**Remora:** 커밋 [6602423](https://github.com/remora-projects/remora-smart-contracts/commit/66024232cfea24b69bd055086f8088d40f3d1d4a)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Cache identical storage reads

**Description:** 스토리지 읽기는 비용이 많이 듭니다. 동일한 스토리지 값을 여러 번 다시 읽지 않도록 동일한 스토리지 읽기를 캐시하세요.

`PledgeManager.sol`:
```solidity
// cache `stablecoinDecimals` in `_fixDecimals`
307:            stablecoinDecimals < 6
308:                ? value / (10 ** (6 - stablecoinDecimals))
309:                : value * (10 ** (stablecoinDecimals - 6));

// cache `propertyToken` in `_verifyDocumentSignature`
316:        (bool res, ) = IRemoraRWAToken(propertyToken).hasSignedDocs(signer);
318:            IRemoraRWAToken(propertyToken).verifySignature(
```

`TokenBank.sol`:
```solidity
// cache `developments.length` in `removeToken`
152:        for (uint i = 0; i < developments.length; ++i) {
154:            address end = developments[developments.length - 1];

// cache `developments.length` in `viewAllFees`, claimAllFees
226:        for (uint i = 0; i < developments.length; ++i)
233:        for (uint i = 0; i < developments.length; ++i) {
```

`LockUpManager.sol`:
```solidity
// cache `userData.endInd` in `availableTokens`, `_unlockTokens`
117:        for (uint16 i = userData.startInd; i < userData.endInd; ++i) {
146:        for (uint16 i = userData.startInd; i < userData.endInd; ++i) {
```

**Remora:** 커밋 [6602423](https://github.com/remora-projects/remora-smart-contracts/commit/66024232cfea24b69bd055086f8088d40f3d1d4a)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Remove `< 0` comparison for unsigned integers

**Description:** 부호 없는 정수는 `< 0`일 수 없으므로 이 비교를 제거해야 합니다:
`PledgeManager.sol`:
```diff
-        if (numTokens <= 0 || _propertyToken.balanceOf(signer) < numTokens)
+        if (numTokens == 0 || _propertyToken.balanceOf(signer) < numTokens)
```

**Remora:** 커밋 [6be4660](https://github.com/remora-projects/remora-smart-contracts/commit/6be4660990ebafbb7200425978f078a0865732fe)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Variables in non-upgradeable contracts which are only set once in `constructor` should be declared `immutable`

**Description:** `constructor`에서 한 번만 설정되는 업그레이드 불가능한 컨트랙트의 변수는 `immutable`로 선언되어야 합니다:
* `PledgeManager::holderWallet`, `propertyToken`, `stablecoin`, `stablecoinDecimals`, `deadline`, `postDeadlineWithdrawPeriod`, `pricePerToken`

**Remora:** 커밋 [afd07fb](https://github.com/remora-projects/remora-smart-contracts/commit/afd07fb419c354dc223d0105b2fd0c5d565f465f)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Break out of loop once element has been deleted in `TokenBank::removeToken`

**Description:** `TokenBank::removeToken`에서 요소가 삭제되면 루프를 빠져나오세요:
```diff
    function removeToken(address tokenAddress) external restricted {
        for (uint i = 0; i < developments.length; ++i) {
            if (developments[i] == tokenAddress) {
                address end = developments[developments.length - 1];
                developments[i] = end;
                developments.pop();
+               break;
            }
        }
```

**Remora:** 2025/07/02 최신 커밋 시점에서 수정되었으나 정확한 커밋은 알 수 없습니다.

**Cyfrin:** 검증되었습니다.


### Use named returns where this eliminates a local variable and especially for `memory` returns

**Description:** 로컬 변수를 제거하는 경우, 특히 `memory` 반환의 경우 명명된 반환(named returns)을 사용하세요:
* `TokenBank::viewAllFees`

**Remora:** 커밋 [6602423](https://github.com/remora-projects/remora-smart-contracts/commit/66024232cfea24b69bd055086f8088d40f3d1d4a)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### In `RemoraToken::adminClaimPayout`, `adminTransferFrom` don't call `hasSignedDocs` when `checkTC == false`

**Description:** `RemoraToken::adminClaimPayout`에서 `checkTC == false`일 때 불필요한 작업을 방지하기 위해 `hasSignedDocs`를 호출하지 마세요:
```solidity
function adminClaimPayout(
    address investor,
    bool useStablecoin,
    bool useCustomFee,
    bool checkTC,
    uint256 feeValue
) external nonReentrant restricted {
    if(checkTC) {
        (bool res, ) = hasSignedDocs(investor);
        if (!res) revert TermsAndConditionsNotSigned(investor);
    }

    _claimPayout(investor, useStablecoin, useCustomFee, feeValue);
}
```

`adminTransferFrom`에도 유사한 수정을 적용하세요.

**Remora:** 커밋 [a5c03d4](https://github.com/remora-projects/remora-smart-contracts/commit/a5c03d4d22a783e3ab7966a7e573724648cad0a9)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### In `RemoraToken::transfer`, `transferFrom` and `_exchangeAllowed` perform all checks for each user together in order to prevent unnecessary work

**Description:** `RemoraToken::transfer`, `transferFrom` 및 `_exchangeAllowed`는 불필요한 작업을 방지하기 위해 각 사용자에 대한 모든 검사를 함께 수행해야 합니다.

`transfer`:
```solidity
        bool fromWL = _whitelist[sender];
        if (!fromWL) _unlockTokens(sender, value, false);

        bool toWL = _whitelist[to];
        if (!toWL) _lockTokens(to, value);
```

`transferFrom`:
```solidity
        bool fromWL = _whitelist[from];
        if (!fromWL) _unlockTokens(from, value, false);

        bool toWL = _whitelist[to];
        if (!toWL) _lockTokens(to, value);
```

`_exchangeAllowed`:
```solidity
        (bool resFrom, ) = hasSignedDocs(from);
        if (!resFrom && !_whitelist[from]) revert TermsAndConditionsNotSigned(from);

        (bool resTo, ) = hasSignedDocs(to);
        if (!resTo && !_whitelist[to]) revert TermsAndConditionsNotSigned(to);
```

**Remora:** 커밋 [59cf2eb](https://github.com/remora-projects/remora-smart-contracts/commit/59cf2eb26742de48988623fdd029d17b2993ba2f)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### `AllowList::hasTradeRestriction` mutability should be set to `view`

**Description:** `AllowList::hasTradeRestriction`은 스토리지에 변경을 가하지 않고 단순히 일부 변수를 읽고 확인만 하므로 함수의 mutability를 view로 표시해야 합니다.

**Recommended Mitigation:** mutability를 view로 변경하세요.

**Remora:** 커밋 [81faadb](https://github.com/remora-projects/remora-smart-contracts/commit/81faadbd3baa1deec950db08abf635c5a277a7c5)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Remove `decimals` from initial `RemoraToken` mint

**Description:** `RemoraToken`은 토큰이 비-분수형이므로 `decimals` 함수를 오버라이드하여 0을 반환합니다:
```solidity
    /**
     * @notice Defines the number of decimal places for the token.
     * RWA tokens are non-fractional and operate in whole units only.
     * @return The number of decimals (always `0`).
     */
    function decimals() public pure override returns (uint8) {
        return 0;
    }
```

그러므로 `RemoraToken::initialize`에서 `10 ** decimals()`는 항상 1로 평가되므로 `decimals` 호출을 제거하세요:
```diff
    function initialize(
        address tokenOwner,
        address initialAuthority,
        address stablecoin,
        address wallet,
        address _allowList,
        string memory _name,
        string memory _symbol,
        uint256 _initialSupply
    ) public initializer {
        // does this order matter?, open zeppelin upgrades giving errors here
        __ERC20_init(_name, _symbol);
        __ERC20Permit_init(_name);
        __Pausable_init();
        __RemoraBurnable_init();
        __RemoraLockUp_init(0); //start with 0 lock up time
        __RemoraDocuments_init(_name, "1");
        __RemoraHolderManagement_init(
            initialAuthority,
            stablecoin,
            wallet,
            0 //starts at zero, will need to update it later
        );
        __UUPSUpgradeable_init();

        allowlist = _allowList;
        _whitelist[tokenOwner] = true; //whitelist owner to be able to send tokens freely
-       _mint(tokenOwner, _initialSupply * 10 ** decimals());
+       _mint(tokenOwner, _initialSupply);
    }
```

**Remora:** 커밋 [462d2de](https://github.com/remora-projects/remora-smart-contracts/commit/462d2def8f09453435e4df433228cfaa565d2e37)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Use timestamp instead of uri length to test of existing document in `DocumentManager`

**Description:** `DocumentManager`에서 기존 문서를 테스트하기 위해 uri 길이 대신 타임스탬프를 사용하세요:
```diff
-       if (bytes($._documents[docHash].docURI).length == 0)
-           revert EmptyDocument();
+       if ($._documents[docHash].timestamp == 0) revert EmptyDocument();
```

**Remora:** 커밋 [77af634](https://github.com/remora-projects/remora-smart-contracts/commit/77af634d953ce3549a33e7b34db924233dc7689a)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.

\clearpage
