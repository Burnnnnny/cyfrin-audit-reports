**Lead Auditors**

[Immeas](https://twitter.com/0ximmeas)

[ChainDefenders](https://x.com/DefendersAudits) ([0x539](https://x.com/1337web3) & [PeterSR](https://x.com/PeterSRWeb3))

**Assisting Auditors**



---

# 발견 사항 (Findings)
## 낮은 위험 (Low Risk)


### 빈 볼트에서 `StakingVault::notifyRewardAmount` 호출 시 ILV가 묶임

**설명:** `StakingVault::notifyRewardAmount`는 스테이커가 나중에 ILV를 청구할 수 있도록 주당 보상 누적기(`accIlvPerShare`)를 업데이트하는 볼트의 보상 후크입니다. 이 함수는 [`L2RevenueDistributorV3::_applyAllocation`](https://github.com/0xKaizenLabs/staking-contracts-v3/blob/c78653ed5f2e5a6d5ace13c303a8765fe30679b0/src/L2RevenueDistributorV3.sol#L500-L504)이 이미 ILV를 볼트로 전송한 후에 호출됩니다.
```solidity
if (pool.kind == PoolKind.Vault) {
    // Transfer ILV to the vault and notify
    ilv.safeTransfer(pool.recipient, amount);
    IStakingVaultMinimal(pool.recipient).notifyRewardAmount(amount);
} else {
```

`totalStaked == 0`인 동안 `notifyRewardAmount`가 호출되면, [`StakingVault::notifyRewardAmount`](https://github.com/0xKaizenLabs/staking-contracts-v3/blob/c78653ed5f2e5a6d5ace13c303a8765fe30679b0/src/StakingVault.sol#L330-L338)는 조기에 반환되고 전송된 ILV는 `accIlvPerShare`에 추가되지 않습니다.
```solidity
function notifyRewardAmount(uint256 ilvAmount)
    external
    override
    nonReentrant
    whenNotPaused
    onlyRole(DISTRIBUTOR_ROLE)
{
    // If no rewards are provided, return
    if (ilvAmount == 0) return;
```

결과적으로 해당 토큰은 볼트에 남아 있으며 정상적인 흐름을 통해 청구할 수 없습니다. 나중에 복구할 수 있는 내장된 스윕(sweep)/버퍼가 없습니다.

**영향:** 자금은 사실상 묶이게 됩니다. 사용자나 관리자 모두 기존 코드 경로를 사용하여 묶인 ILV를 재분배하거나 인출할 수 없습니다. 실제로 유일한 복구 방법은 컨트랙트를 업그레이드하는 것입니다. 그때까지 ILV는 볼트에서 유휴 상태로 유지됩니다.

**권장 완화 조치:**
* 볼트 측 버퍼: `unallocatedRewards`를 추적하고 들어오는 금액을 항상 여기에 추가합니다. `totalStaked > 0`일 때 전체 버퍼를 `accIlvPerShare`에 포함시킵니다.
* 배포자(Distributor) 측 스킵/이월(carry): 볼트로 전송하기 전에 `totalStaked() > 0`인지 확인합니다. 0이면 전송을 건너뛰고 나중에 다시 시도하도록 풀별 이월을 기록합니다(또는 청구 가능한 버킷으로 전환).
* 빈 볼트에서 하드 페일(Hard fail): `notifyRewardAmount`에서 `totalStaked == 0`일 때 되돌려 우발적인 묶임을 방지합니다(예: `require(totalStaked > 0, "No stakers")`). 전체 tx 실패를 방지하기 위해 배포자에서 풀별 try/catch 또는 건너뛰기 로직과 함께 사용합니다.

**Illuvium:** 커밋 [5f273bc](https://github.com/0xKaizenLabs/staking-contracts-v3/commit/5f273bc8a196170162400c33a43efe2fb84f0013)에서 수정됨.

**Cyfrin:** 확인함. 볼트 측 버퍼 완화 조치가 구현되었습니다.


### 배포 스크립트에 암호화되지 않은 개인 키 필요

**설명:** [Makefile](https://github.com/0xKaizenLabs/staking-contracts-v3/blob/c78653ed5f2e5a6d5ace13c303a8765fe30679b0/Makefile)의 배포 타겟은 환경 변수를 통해 원시(raw) 개인 키를 제공받아 명령줄에서 `forge`에 직접 전달해야 합니다.

```make
# Guard (fails if PRIVATE_KEY not provided)
@if [ -z "$(PRIVATE_KEY)" ]; then \
    echo "$(RED)ERROR: PRIVATE_KEY not set$(NC)"; \
    exit 1; \
fi

# Usage (Sepolia)
forge script script/Deployer.s.sol:Deployer \
  --rpc-url $(BASE_SEPOLIA_RPC) \
  --private-key $(PRIVATE_KEY) \
  --broadcast \
  --verify \
  --etherscan-api-key $(BASESCAN_API_KEY) \
  -vvvv

# Usage (Mainnet)
forge script script/Deployer.s.sol:Deployer \
  --rpc-url $(BASE_RPC) \
  --private-key $(PRIVATE_KEY) \
  --broadcast \
  --verify \
  --etherscan-api-key $(BASESCAN_API_KEY) \
  --slow \
  -vvvv
```

이런 식으로 비밀을 제공하면 일반 텍스트 처리(예: `.env` 파일, 쉘 기록)를 장려하고 프로세스 인수에서 키를 노출시킵니다. 개인 키를 일반 텍스트로 저장하는 것은 버전 관리, 잘못 구성된 백업 또는 손상된 개발자 머신을 통한 우발적인 노출 가능성을 높이므로 운영 보안 위험을 나타냅니다.

더 안전한 접근 방식은 암호화된 키 저장을 허용하는 Foundry의 [지갑 관리 기능](https://getfoundry.sh/forge/reference/script/)을 사용하는 것입니다. 예를 들어, [`cast`](https://getfoundry.sh/cast/reference/wallet/import/)를 사용하여 개인 키를 로컬 키 저장소로 가져올 수 있습니다.

```bash
cast wallet import deployerKey --interactive
```

그런 다음 배포 중에 이 키를 안전하게 참조할 수 있습니다.

```make
DEPLOYER_ACCOUNT ?= deployerKey
DEPLOYER_SENDER  ?= $(shell cast wallet address $(DEPLOYER_ACCOUNT) 2>/dev/null)

forge script script/Deployer.s.sol:Deployer \
  --rpc-url $(BASE_SEPOLIA_RPC) \
  --account $(DEPLOYER_ACCOUNT) \
  --sender $(DEPLOYER_SENDER) \
  --broadcast \
  --verify \
  --etherscan-api-key $(BASESCAN_API_KEY) \
  -vvvv
```
그리고
```make
cast wallet list | grep -q "$(DEPLOYER_ACCOUNT)" || { \
  echo "$(RED)ERROR: Keystore account '$(DEPLOYER_ACCOUNT)' not found$(NC)"; exit 1; }
```

추가 지침은 Patrick의 [이 설명 비디오](https://www.youtube.com/watch?v=VQe7cIpaE54)를 참조하십시오.

**Illuvium:** 커밋 [5f273bc](https://github.com/0xKaizenLabs/staking-contracts-v3/commit/5f273bc8a196170162400c33a43efe2fb84f0013)에서 수정됨.

**Cyfrin:** 확인함.


### 아주 적은 양(Dust amounts)의 보상이 볼트에 묶임

**설명:** `StakingVault`에서 보상은 `notifyRewardAmount` 함수를 통해 분배됩니다. 그러나 현재 구현으로 인해 일부 보상은 반올림 오류로 인해 컨트랙트에 묶인 상태로 남을 수 있습니다.

예를 들면 다음과 같습니다.

* User1이 `1e18`을 스테이킹함
* User2가 `7e18`을 스테이킹함
* 총 보상 금액은 `1e18`임

반올림으로 인해 실제 분배된 보상은 `1e18`보다 적은 `999999999999000000`이 됩니다. 이로 인해 보상의 일부가 분배되지 않고 컨트랙트에 영구적으로 묶이게 됩니다.

```solidity
function notifyRewardAmount(uint256 ilvAmount)
        external
        override
        nonReentrant
        whenNotPaused
        onlyRole(DISTRIBUTOR_ROLE)
    {
        // If no rewards are provided, return
        if (ilvAmount == 0) return;

        // Cache total staked for gas efficiency
        uint256 _totalStaked = totalStaked;

        // When no one is staking, return (prevents division by zero)
        if (_totalStaked == 0) {
            return;
        }

        // Update rewards-per-share accumulator
        accIlvPerShare += (ilvAmount * Constants.ACC_PRECISION) / _totalStaked;

        emit RewardsNotified(ilvAmount, accIlvPerShare);
    }
```

**영향:** 보상 토큰의 일부가 컨트랙트에 묶여 사용자가 청구할 수 없게 되어 보상이 비효율적으로 분배됩니다.

**개념 증명 (Proof of Concept):** 다음 테스트는 문제를 보여주며 `StakingVault.t.sol`에 추가할 수 있습니다.

```solidity
function test_twoUsersWhoStakeAreEligibleForAllRewards() public {
    address user = makeAddr("User1");
    address user2 = makeAddr("User2");
    uint96 amount = 1e18;
    uint32 duration = 31 days;
    uint96 notifyAmt = 1e18;

    // User 1 and User 2 deposit
    approveAndDeposit(user, amount, duration);
    approveAndDeposit(user2, amount * 6, duration);

    // Fund and notify rewards
    ilv.mint(address(vault), notifyAmt);
    vm.prank(admin);
    vault.notifyRewardAmount(notifyAmt);

    // Check if total pending rewards equal notifyAmt
    uint256 pending = vault.pendingRewards(user);
    uint256 pending2 = vault.pendingRewards(user2);
    assertEq(pending + pending2, notifyAmt);
}
```

총 대기 보상이 통지된 금액과 같지 않기 때문에 assertion이 실패합니다.

**권장 완화 조치:** `ACC_PRECISION`을 `1e12`에서 `1e18`로 변경하여 보상 계산의 정밀도를 높이십시오.
이 조정은 반올림 오류를 최대 **1 wei**로 줄여 거의 모든 보상이 올바르게 분배되도록 보장합니다.

**Illuvium:** 커밋 [5f273bc](https://github.com/0xKaizenLabs/staking-contracts-v3/commit/5f273bc8a196170162400c33a43efe2fb84f0013)에서 수정됨.

**Cyfrin:** 확인함.


### 배포자가 청구 가능 금액을 초과 할당하여 일시적인 청구 DoS 유발 가능

**설명:** `L2RevenueDistributorV3::distribute`는 `pendingIlv`를 증가시켜 청구 가능한 풀에 크레딧을 제공하지만 해당 미래 청구를 위해 토큰을 예약하지 않습니다. ILV를 볼트로 전송하는 후속 배포는 청구 가능 항목이 의존하는 동일한 온체인 잔액을 소비할 수 있습니다. 이로 인해 `sum(pendingIlv)`가 컨트랙트의 ILV 잔액보다 커질 수 있으므로 나중에 배포자가 채워질 때까지 `L2RevenueDistributorV3::claimPool`이 되돌립니다.

**영향:** 시스템은 운영자가 배포자에 자금을 계속 공급하는 것에 의존합니다. 그렇지 않은 경우, 청구 가능한 풀은 채워질 때까지 일시적으로 DoS될 수 있습니다. 자금의 영구적 손실은 없지만 가용성에 영향을 미치고 회계가 오해의 소지가 있게 됩니다.

**개념 증명 (Proof of Concept):** `L2RevenueDistributorV3.t.sol`에 이 테스트를 추가하십시오.
```solidity
    /// PoC: Overcommit claimables via two distributions -> claim reverts (insufficient balance)
    function test_PoC_Overcommit_BricksClaimables_TwoDistributes() public {
        // 1) Seed distributor
        uint256 amount1 = 1_000e18; // clean number to avoid rounding noise
        ilv.mint(address(distributor), amount1);

        // 2) First distribution. Pool 2 (claimable, 2000 bps) accrues 20% pending; 80% sent to vaults.
        vm.prank(operator);
        distributor.distribute(amount1);

        uint256 expectedPendingAfter1 = (amount1 * 2000) / Constants.BPS_DENOMINATOR;
        assertEq(pendingOf(2), expectedPendingAfter1, "pending after first distribute");
        assertEq(ilv.balanceOf(address(distributor)), expectedPendingAfter1, "residual ILV equals claimable pending");

        // 3) Naive second distribution using the entire remaining balance.
        uint256 amount2 = ilv.balanceOf(address(distributor)); // == expectedPendingAfter1
        vm.prank(operator);
        distributor.distribute(amount2);

        // Now: balance < pending (overcommitted)
        uint256 pendingNow = pendingOf(2); // 20% of amount1 + 20% of amount2
        uint256 balNow = ilv.balanceOf(address(distributor));
        assertLt(balNow, pendingNow, "distributor balance < claimable pending");

        vm.prank(admin);
        distributor.setClaimAllowlist(2, address(this), true);

        // 4) Claims are bricked until top-up
        vm.expectRevert("balance"); // SafeERC20 transfer fails due to insufficient ILV
        distributor.claimPool(2, address(this));
    }
```

**권장 완화 조치:** 청구 가능 항목에 이미 약속된 내용을 추적하고 예약되지 않은 잔액만 배포하도록 보장하십시오.

* 전역 누적기 추가:

  ```solidity
  uint256 public reservedClaimables;
  ```

* 청구 가능한 풀에 할당할 때 준비금을 늘리고, 청구할 때 줄입니다.

  ```solidity
  function _applyAllocation(Pool storage pool, uint256 amount) internal {
      if (pool.kind == PoolKind.Vault) {
          ilv.safeTransfer(pool.recipient, amount);
          IStakingVaultMinimal(pool.recipient).notifyRewardAmount(amount);
      } else {
          pool.pendingIlv += amount;
          reservedClaimables += amount;
      }
  }

  function claimPool(uint256 id, address to) external nonReentrant whenNotPaused {
      // ...existing checks...
      uint256 amount = pool.pendingIlv;
      if (amount == 0) revert NothingToClaim(id);
      pool.pendingIlv = 0;
      reservedClaimables -= amount;     // reduce reserved on successful claim
      ilv.safeTransfer(to, amount);
      emit PoolClaimed(id, to, amount);
  }
  ```

* 배포를 수행하기 전에 요청이 컨트랙트의 예약되지 않은 ILV를 초과하지 않는지 확인하십시오.

  ```solidity
  error InsufficientUnreservedBalance();

  function distribute(uint256 ilvAmount) external nonReentrant whenNotPaused onlyRole(OPERATOR_ROLE) {
      if (ilvAmount == 0) revert DistributeAmountZero();

      if (ilvAmount > ilv.balanceOf(address(this) - reservedClaimables) revert InsufficientUnreservedBalance();

      // proceed with existing allocation logic...
  }
  ```

이것은 `distribute()` 시작 시 `balance - claimable >= amount`를 강제하여 청구 가능한 약속이 항상 컨트랙트상의 토큰으로 뒷받침되고 `claimPool()`의 과도한 약속/DoS를 방지합니다.


**Illuvium:** 커밋 [5f273bc](https://github.com/0xKaizenLabs/staking-contracts-v3/commit/5f273bc8a196170162400c33a43efe2fb84f0013)에서 수정됨.

**Cyfrin:** 확인함.


### 준비금 가중치 할당이 현물(spot) 준비금을 사용함

**발견 사항 (낮음): 준비금 가중치 할당이 현물 준비금을 사용함 → 가격 조작 가능**

**설명:** [`L2RevenueDistributorV3::_computeReserveUnits`](https://github.com/0xKaizenLabs/staking-contracts-v3/blob/c78653ed5f2e5a6d5ace13c303a8765fe30679b0/src/L2RevenueDistributorV3.sol#L521-L525)는 현재 쌍(pair) 준비금(`getReserves()` 및 `totalSupply()`) 즉, 현물 스냅샷에서 LP "단위"를 도출합니다.
```solidity
IAerodromePair pair = IAerodromePair(pairAddr);
(uint256 r0, uint256 r1,) = pair.getReserves();
address t0 = pair.token0();
address t1 = pair.token1();
uint256 ilvReserve = t0 == address(ilv) ? r0 : (t1 == address(ilv) ? r1 : 0);
```
이는 준비금 조작을 허용합니다.

**영향:** 호출자는 `distribute` 바로 전에 LP 준비금을 잠시 왜곡하여 해당 호출의 보상을 선택한 LP 볼트로 유도할 수 있습니다. 비용은 일시적인 거래 + 수수료입니다. 전술을 반복하면 이익을 복리화할 수 있습니다. Base에 공개 멤풀이 없다고 해서 공개 MEV는 줄어들지만 조작이 완전히 제거되는 것은 아닙니다.

**권장 완화 조치:** 쌍의 `observations`에서 고정된 창에 대한 준비금 TWAP를 사용하는 것을 고려하십시오. 오라클 기록이 요청된 창을 충족할 수 없으면 되돌립니다. 충분한 관찰 길이는 풀을 배포할 때 프로토콜에 의해 보장될 수 있습니다.

```solidity
// Extend the pair interface as needed for your Aerodrome build.
interface IAerodromePairOracle is IAerodromePair {
    function observationLength() external view returns (uint256);
    function lastObservation()
        external
        view
        returns (uint256 timestamp, uint256 reserve0Cumulative, uint256 reserve1Cumulative);
    function observations(uint256 index)
        external
        view
        returns (uint256 timestamp, uint256 reserve0Cumulative, uint256 reserve1Cumulative);
}

library OracleLib {
    error OracleInsufficientHistory(); // fewer than 2 observations
    error OracleWindowNotSatisfied(uint256 availableWindow, uint256 requiredWindow);

    /// @dev Returns average reserves over `windowSecs` using cumulative reserve observations.
    function averageReserves(address pairAddr, uint32 windowSecs)
        internal
        view
        returns (uint256 avgR0, uint256 avgR1)
    {
        IAerodromePairOracle pair = IAerodromePairOracle(pairAddr);
        uint256 len = pair.observationLength();
        if (len < 2) revert OracleInsufficientHistory();

        (uint256 tNew, uint256 r0New, uint256 r1New) = pair.lastObservation();
        uint256 tTarget = tNew - uint256(windowSecs);

        // Walk back to an observation at or before tTarget (use binary search if ring buffer is large).
        uint256 idx = len - 2;
        (uint256 tOld, uint256 r0Old, uint256 r1Old) = pair.observations(idx);
        while (tOld > tTarget) {
            if (idx == 0) {
                revert OracleWindowNotSatisfied(tNew - tOld, windowSecs);
            }
            idx--;
            (tOld, r0Old, r1Old) = pair.observations(idx);
        }

        uint256 dt = tNew - tOld;
        if (dt == 0) revert OracleWindowNotSatisfied(0, windowSecs);

        avgR0 = (r0New - r0Old) / dt;
        avgR1 = (r1New - r1Old) / dt;
    }
}

contract L2RevenueDistributorV3 {
    // ...
    uint32 internal constant TWAP_WINDOW = 15 minutes;

    function _computeReserveUnits(Pool storage pool) internal view returns (uint256) {
        if (pool.kind != PoolKind.Vault) return 0;

        // ILV single-asset vault unchanged
        if (pool.underlyingToken == address(ilv)) {
            return IStakingVaultMinimal(pool.recipient).totalStaked();
        }

        // LP vault: use TWAP average reserves; revert if oracle history is insufficient
        address pairAddr = pool.underlyingToken;
        if (pairAddr == address(0)) return 0;

        (uint256 avgR0, uint256 avgR1) = OracleLib.averageReserves(pairAddr, TWAP_WINDOW);

        IAerodromePair p = IAerodromePair(pairAddr);
        address t0 = p.token0();
        address t1 = p.token1();
        uint256 ilvReserveAvg = t0 == address(ilv) ? avgR0 : (t1 == address(ilv) ? avgR1 : 0);
        if (ilvReserveAvg == 0) return 0;

        uint256 lpSupply = p.totalSupply();
        if (lpSupply == 0) return 0;

        uint256 vaultLp = IStakingVaultMinimal(pool.recipient).totalStaked();
        return (ilvReserveAvg * vaultLp) / lpSupply;
    }
}
```

**Illuvium:** 인지함; 준비금 가중치 풀 할당은 Aerodrome 쌍의 현물 준비금을 계속 사용합니다. 우리는 프론트러닝이 수익성이 없거나 불가능하도록 합리적인 슬리피지 보호 매개변수를 사용할 것이며 Base의 신뢰할 수 있는 시퀀서를 믿습니다.


### 아주 적은 양의 보상이 영구적으로 잠길 수 있음

**설명**
`distribute()` 중에 남은 금액("dust")이 나머지로 남을 수 있습니다. 이러한 나머지는 첫 번째 자격 있는 참가자에게 할당하려고 시도합니다. 그러나 나머지가 **`totalStaked / 1e18`보다 작으면** 보상이 분배되지 않고 대신 컨트랙트에 영구적으로 묶이게 됩니다.

```solidity
if (rwDistributed < remainder) {
    uint256 dust = remainder - rwDistributed;
    rwDistributed += _allocateFirstEligible(dust, poolCount, units);
}
```

**영향**
시간이 지남에 따라 이렇게 작은 나머지가 있는 반복적인 배포가 누적되어 점점 더 많은 양의 ILV가 컨트랙트에 잠기고 스테이커에게 도달하지 못하게 됩니다.

**권장 완화 조치**
의미 있는 금액만 분배되도록 **`_applyAllocation` 내부**에 임계값 확인을 적용하십시오.

```solidity
function _applyAllocation(Pool storage pool, uint256 amount) internal {
    // skip if dust is too small to be distributed
    if (amount * Constants.ACC_PRECISION <= pool.totalStaked) {
        return;
    }

    if (pool.kind == PoolKind.Vault) {
        ilv.safeTransfer(pool.recipient, amount);
        IStakingVaultMinimal(pool.recipient).notifyRewardAmount(amount);
    } else {
        pool.pendingIlv += amount;
    }
}
```

**Illuvium:** 커밋 [5f273bc](https://github.com/0xKaizenLabs/staking-contracts-v3/commit/5f273bc8a196170162400c33a43efe2fb84f0013)에서 수정됨.

**Cyfrin:** 확인함.


### 균등 분배 대체(fallback) 로직 오류

**설명:** `L2RevenueDistributorV3::_allocateEqual()` 함수에서 총 준비금 단위가 0이면 컨트랙트는 남은 ILV를 활성 준비금 가중치 풀에 균등하게 분배하는 것으로 대체합니다. 그러나 Vault 풀에 대한 자격 확인에는 `units[i] > 0` 조건이 포함되지만 이 시나리오에서는 `units[i]`가 항상 0입니다(`totalUnits`가 0이므로). 이로 인해 Vault 풀이 균등 분배에서 잘못 제외되어 청구 가능한(Claimable) 풀만 자금을 받을 수 있게 됩니다.

**영향:** 준비금 계산 결과 단위가 0이 되면 Vault 풀이 의도한 ILV 분배 몫을 받지 못해 보상이 불공정하게 할당될 수 있습니다. 이로 인해 Vault 풀에 스테이킹하는 사용자의 예상 수익이 손실되어 프로토콜의 인센티브 메커니즘이 약화될 수 있습니다.

**권장 완화 조치:** 균등 분배 모드일 때 Vault 풀에 대해 `units[i]`에 의존하지 않도록 `_allocateEqual()`의 자격 확인을 수정하십시오. 대신 풀이 활성 상태이고 준비금 가중치가 적용되는지 확인하십시오. 예를 들면 다음과 같습니다.

```solidity
bool isEligible = (pool.kind == PoolKind.Claimable) || (pool.kind == PoolKind.Vault);
```

이렇게 하면 준비금이 0일 때 청구 가능 풀과 Vault 풀 모두 균등 분배에 포함됩니다. 또한 이 동작을 명확히 하기 위해 주석을 추가하는 것을 고려하십시오.

**Illuvium:** 커밋 [5f273bc](https://github.com/0xKaizenLabs/staking-contracts-v3/commit/5f273bc8a196170162400c33a43efe2fb84f0013)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage
## 정보 (Informational)


### `L2RevenueDistributorV3`에서 사용되지 않는 `factory` 스토리지

**설명:** `L2RevenueDistributorV3`는 [`IAerodromeFactory public factory`](https://github.com/0xKaizenLabs/staking-contracts-v3/blob/c78653ed5f2e5a6d5ace13c303a8765fe30679b0/src/L2RevenueDistributorV3.sol#L54)를 저장하고 `initialize`에서 이를 수락하지만 절대 읽거나 사용하지 않습니다. 스왑은 `routes[i].factory`에 의존하고 준비금 수학은 LP 쌍 주소를 직접 사용합니다. 어떤 로직도 `factory` 변수를 참조하지 않습니다.

* 필요하지 않은 경우: `initialize`에 전달하는 것을 중지하고 public 변수를 제거하십시오.
* 안전을 위한 것이라면: 스왑 함수에서 `routes[i].factory == address(factory)`를 확인하여 스왑을 승인된 팩토리에 고정하여 이를 강제하십시오.

**Illuvium:** 커밋 [5f273bc](https://github.com/0xKaizenLabs/staking-contracts-v3/commit/5f273bc8a196170162400c33a43efe2fb84f0013)에서 수정됨.

**Cyfrin:** 확인함.


### 중요한 상태 변경에 대한 이벤트 방출 부족

**설명:** * [`StakingVault::setLockDurations`](https://github.com/0xKaizenLabs/staking-contracts-v3/blob/c78653ed5f2e5a6d5ace13c303a8765fe30679b0/src/StakingVault.sol#L388-L396)
* [`L2RevenueDistributorV3::setClaimAllowlist`](https://github.com/0xKaizenLabs/staking-contracts-v3/blob/c78653ed5f2e5a6d5ace13c303a8765fe30679b0/src/L2RevenueDistributorV3.sol#L248-L255)

이벤트가 없으면 온체인 투명성이 감소하고 오프체인 모니터링이 약해집니다. 인덱서/경보 시스템은 매개변수 변경을 감지할 수 없으므로 감사, 사고 대응 및 UX 디버깅이 더 어려워집니다.

**Illuvium:** 커밋 [5f273bc](https://github.com/0xKaizenLabs/staking-contracts-v3/commit/5f273bc8a196170162400c33a43efe2fb84f0013)에서 수정됨.

**Cyfrin:** 확인함.


### `StakingVault::_updateRewards`에서 누적된 보상의 불필요한 재계산

**설명:** [`StakingVault::_updateRewards`](https://github.com/0xKaizenLabs/staking-contracts-v3/blob/c78653ed5f2e5a6d5ace13c303a8765fe30679b0/src/StakingVault.sol#L291-L295)에서 `accrued`가 계산된 다음 `totalRewardDebt`에 대해 정확히 동일한 표현식이 재계산됩니다. 두 번째 계산은 불필요합니다.

```solidity
uint256 accrued = (staked * _acc) / _scale;
if (accrued > userInfo.totalRewardDebt) {
    userInfo.storedPendingRewards += accrued - userInfo.totalRewardDebt;
}
// @audit Recomputed unnecessarily
userInfo.totalRewardDebt = (staked * _acc) / _scale;
```

이미 계산된 값을 재사용하는 것을 고려하십시오.

```solidity
uint256 accrued = (staked * _acc) / _scale;
if (accrued > userInfo.totalRewardDebt) {
    userInfo.storedPendingRewards += accrued - userInfo.totalRewardDebt;
}
userInfo.totalRewardDebt = accrued;
```

**Illuvium:** 커밋 [5f273bc](https://github.com/0xKaizenLabs/staking-contracts-v3/commit/5f273bc8a196170162400c33a43efe2fb84f0013)에서 수정됨.

**Cyfrin:** 확인함.


### `StakingVault`에 최소 예치 금액 추가 고려

**설명:** [`StakingVault.deposit`](https://github.com/0xKaizenLabs/staking-contracts-v3/blob/c78653ed5f2e5a6d5ace13c303a8765fe30679b0/src/StakingVault.sol#L140-L141)은 1 wei를 포함하여 모든 양의 `amount`를 수락합니다. 극히 적은 지분은 경제적으로 의미가 없으며 주당 보상 모델에서 정수 반올림 엣지 케이스를 증폭시킬 수 있습니다. 이러한 스타일의 예금은 블랙햇이 볼트를 조작하는 데만 사용됩니다.

구성 가능한 최소 예치 임계값을 추가하고 `deposit`에서 이를 강제하는 것을 고려하십시오.

```solidity
error DepositAmountTooSmall(uint256 amount, uint256 minDeposit);

uint256 public minDeposit; // set by admin (e.g., during initialize), adjustable via setter + event

function setMinDeposit(uint256 newMin) external onlyRole(ADMIN_ROLE) {
    minDeposit = newMin;
    emit MinDepositUpdated(newMin);
}

function deposit(uint256 amount, uint64 lockStakeDuration) external whenNotPaused nonReentrant returns (uint256) {
    if (amount < minDeposit) revert DepositAmountTooSmall(amount, minDeposit);
    // ...existing logic...
}
```
**Illuvium:** 커밋 [5f273bc](https://github.com/0xKaizenLabs/staking-contracts-v3/commit/5f273bc8a196170162400c33a43efe2fb84f0013)에서 수정됨.

**Cyfrin:** 확인함.


### 합리적인 기간 값에 대한 온전성 확인(sanity check) 누락

**설명:** `StakingVault::initialize` 및 `StakingVaultFactory::deployVault`는 잠금 기간이 합리적인지(0이 아니고 너무 크지 않음) 확인하지 못합니다.

관리자는 `minLockDuration`이 `maxLockDuration`보다 작거나 같기만 하면 실수로 `minLockDuration = 0` 또는 `minLockDuration`과 `maxLockDuration` 모두를 매우 높은 값으로 설정할 수 있습니다.

**권장 완화 조치:** 합리적이지 않은 기간을 방지하기 위해 온전성 확인을 추가하십시오. 허용 가능한 값 범위를 결정하려면 상수를 사용하십시오.

**Illuvium:** 커밋 [5f273bc](https://github.com/0xKaizenLabs/staking-contracts-v3/commit/5f273bc8a196170162400c33a43efe2fb84f0013)에서 수정됨.

**Cyfrin:** 확인함.


### `setDistributor`에서 `address(0)` 확인 누락

**설명:** `StakingVault::setDistributor` 함수를 사용하면 관리자가 지정된 계정에 DISTRIBUTOR_ROLE을 부여하거나 취소할 수 있지만, 이 중요한 역할을 영(zero) 주소(address(0))에 할당하는 것을 방지하는 검증이 부족합니다. 즉각적인 보안 취약점을 생성하지는 않지만, 이것이 주어진 역할을 소유한 유일한 주소인 경우 보상 분배 메커니즘을 일시적으로 중단할 수 있는 운영 위험을 나타냅니다.

**권장 완화 조치:** 함수 시작 부분에 영 주소 유효성 검사 확인을 추가하십시오.

```solidity
function setDistributor(address account, bool enabled) external onlyRole(ADMIN_ROLE) {
    if (account == address(0)) revert AddressZero("account");

    if (enabled) {
        _grantRole(DISTRIBUTOR_ROLE, account);
    } else {
        _revokeRole(DISTRIBUTOR_ROLE, account);
    }
}
```

**Illuvium:** 커밋 [5f273bc](https://github.com/0xKaizenLabs/staking-contracts-v3/commit/5f273bc8a196170162400c33a43efe2fb84f0013)에서 수정됨.

**Cyfrin:** 확인함.


### `Pool.underlyingToken == StakingVault.stakingAsset`의 검증 부족

**설명:** `L2RevenueDistributorV3::_computeReserveUnits`는 `pool.underlyingToken`을 기반으로 Vault 풀이 ILV 볼트인지 LP 볼트인지 결정합니다. 그러나 이 값이 볼트의 실제 스테이킹 자산(`StakingVault.stakingAsset`)과 실제로 일치하는지 확인하는 검사가 없습니다. 불일치(구성 오류 또는 업그레이드에 의해)로 인해 배포자가 잘못된 토큰/쌍에 대해 준비금 단위를 계산하여 준비금 가중치 보상을 왜곡할 수 있습니다.

풀 생성/업데이트 시 `PoolKind.Vault`에 대해 `pool.underlyingToken == StakingVault(recipient).stakingAsset()`인지 확인하는 것을 고려하십시오. 또는 Vault 풀에 대한 구성에서 `underlyingToken`을 제거하고 항상 볼트에서 자산을 읽으십시오.

**Illuvium:** 커밋 [5f273bc](https://github.com/0xKaizenLabs/staking-contracts-v3/commit/5f273bc8a196170162400c33a43efe2fb84f0013)에서 수정됨.

**Cyfrin:** 확인함.


### 청구 가능한 풀에 대한 일관성 없는 `recipient` 처리

**설명:** [`L2RevenueDistributorV3.Pool.recipient`](https://github.com/0xKaizenLabs/staking-contracts-v3/blob/c78653ed5f2e5a6d5ace13c303a8765fe30679b0/src/L2RevenueDistributorV3.sol#L73)는 "볼트 주소 또는 청구 가능한 수신자 키"로 문서화되어 있습니다.
```solidity
address recipient; // vault address or claimable recipient key
```

그러나 `L2RevenueDistributorV3::claimPool(id, to)`는 이를 무시하고 호출자가 제공한 `to`로 ILV를 전송합니다. 한편, [배포 스크립트](https://github.com/0xKaizenLabs/staking-contracts-v3/blob/c78653ed5f2e5a6d5ace13c303a8765fe30679b0/script/Deployer.s.sol#L219)는 다른 곳에서 `helperConfig.getL1PoolAdmin()`을 허용 목록에 추가하는 동안 청구 가능한 풀에 대해 더미 `pool.recipient = address(0x1111)`을 설정합니다. 이 불일치는 혼란스럽고 미래 로직이 `recipient`를 사용하기 시작하면 취약해집니다.

`recipient`가 청구 가능한 풀에 사용되지 않는다는 문서를 명확히 하거나 `claimPool`을 `recipient`로 보내도록 변경하고 배포 스크립트에서 자리 표시자를 제거하는 것을 고려하십시오.

**Illuvium:** 커밋 [5f273bc](https://github.com/0xKaizenLabs/staking-contracts-v3/commit/5f273bc8a196170162400c33a43efe2fb84f0013)에서 수정됨.

**Cyfrin:** 확인함; recipient에 관한 문서가 더 명확해졌습니다.

\clearpage
## 가스 최적화 (Gas Optimization)


### `_assertFixedBpsCap`의 최적화되지 않은 오버플로우 확인

**설명:** `_assertFixedBpsCap()` 함수에서 오버플로우 확인 `if (totalBps > Constants.BPS_DENOMINATOR)`은 각 풀 반복에 대해 루프 내부에서 수행됩니다. 이로 인해 모든 반복에서 불필요한 조건부 평가가 발생하여 가스 사용량이 증가합니다. 확인은 모든 관련 `bps` 값을 합산한 후 한 번만 수행하면 됩니다.

**권장 완화 조치:** 루프 내부에서 `totalBps`를 누적한 다음 루프 후에 오버플로우 확인을 한 번 수행하십시오.

```solidity
function _assertFixedBpsCap() internal view {
    uint256 totalBps;
    for (uint256 i; i < pools.length; i++) {
        Pool storage pool = pools[i];
        if (!pool.active || pool.mode != PoolMode.FixedBps) continue;
        totalBps += pool.bps;
    }
    if (totalBps > Constants.BPS_DENOMINATOR) {
        revert FixedBpsOverflow(totalBps);
    }
}
```

이렇게 하면 루프 내부의 불필요한 조건부 확인을 제거하여 가스 비용을 줄일 수 있습니다.

**Illuvium**:
커밋 [5f273bc](https://github.com/0xKaizenLabs/staking-contracts-v3/commit/5f273bc8a196170162400c33a43efe2fb84f0013)에서 수정됨.

**Cyfrin**:
확인함.

\clearpage

