**수석 감사관**

[Immeas](https://twitter.com/0ximmeas)

[holydevoti0n](https://x.com/HolyDevoti0n)

---

# 발견 사항 (Findings)
## 저위험 (Low Risk)


### `LINKMigrator`에서 최소 입금액 값이 강제되지 않음

**설명:** `LINKMigrator` 컨트랙트에는 최소 입금액을 정의하기 위한 `queueDepositMin` 필드가 포함되어 있습니다. 그러나 이 값은 현재 사용되지 않고 있어, 사용자가 1 juel(1e-18 LINK)만큼 적은 금액을 입금할 수 있습니다.

**파급력:** 강제성이 없으면 사용자는 프로토콜이 의도한 것보다 적은 입금액을 제출할 수 있어, 잠재적으로 오버헤드를 증가시키거나 예상된 입금 동작을 방해할 수 있습니다.

**권장 완화 방안:** 사용되지 않는 `queueDepositMin` 필드를 제거하거나 `initiateMigration` 함수에서 이를 강제하는 것을 고려하십시오:

```diff
  function initiateMigration(uint256 _amount) external {
-     if (_amount == 0) revert InvalidAmount();
+     if (_amount < queueDepositMin) revert InvalidAmount();
```

**stake.link:**
[`0abd4f8`](https://github.com/stakedotlink/contracts/commit/0abd4f86e3c5ed28f9ea444fdc9db5394ad5b5ed)에서 수정됨.

**Cyfrin:** 확인함. `queueDepositMin`이 제거되었습니다. `minStakeAmount`는 이제 커뮤니티 풀에서 가져와 `_amount`와 비교하여 확인합니다:
```solidity
(uint256 minStakeAmount, ) = communityPool.getStakerLimits();
if (_amount < minStakeAmount) revert InvalidAmount();
```


### `bypassQueue`에서 `poolStatus` 확인 누락

**설명:** `PriorityPool.sol`의 `bypassQueue` 함수는 토큰을 스테이킹 풀에 직접 입금하기 전에 풀의 상태를 확인하지 않습니다.
```solidity
function bypassQueue(
    address _account,
    uint256 _amount,
    bytes[] calldata _data
) external onlyQueueBypassController {
    token.safeTransferFrom(msg.sender, address(this), _amount);

    uint256 canDeposit = stakingPool.canDeposit();
    if (canDeposit < _amount) revert InsufficientDepositRoom();

    stakingPool.deposit(_account, _amount, _data);
}
```
풀 상태 확인은 프로토콜의 비상 대응 시스템의 일부입니다. [RebaseController](https://github.com/stakedotlink/contracts/blob/4b6b0811835bafa4c8379a39512bfe99bc6c6ebf/contracts/contracts/core/RebaseController.sol#L120-L140)는 전략이 자금 손실로 이어지는 경우와 같은 비상 상황에서 풀 상태를 `CLOSED`로 설정할 수 있습니다. 이러한 이유는 풀을 다시 열 때 `RebaseController`에서 볼 수 있습니다:
```solidity
@>     * @notice Reopens the priority pool and security pool after they were paused as a result
@>     * of a loss and updates strategy rewards in the staking pool
     * @param _data encoded data to pass to strategies
     */
    function reopenPool(bytes calldata _data) external onlyOwner {
        if (priorityPool.poolStatus() == IPriorityPool.PoolStatus.OPEN) revert PoolOpen();


        priorityPool.setPoolStatus(IPriorityPool.PoolStatus.OPEN);
        if (address(securityPool) != address(0) && securityPool.claimInProgress()) {
            securityPool.resolveClaim();
        }
        _updateRewards(_data);
    }
```
`bypassQueue` 함수의 이러한 누락된 확인은 풀이 `CLOSED` 상태일 때 사용자 자금이 입금되도록 허용하여, 잠재적으로 프로토콜 비상 종료 중에 입금된 토큰이 손실될 수 있습니다.

**파급력:** `PriorityPool`이 `CLOSED` 또는 `DRAINING` 상태일 때에도 `LINKMigrator`를 통해 LINK 토큰을 입금할 수 있어, 사실상 보안 사고 중에 사용하도록 의도된 비상 일시 중지 메커니즘을 우회합니다. 이는 잠재적으로 사용자의 자금 손실로 이어질 수 있습니다. 그러나 손실에 대한 내재적 메커니즘이 없고 슬래싱(slashing)이 운영자 풀에서만 발생하므로 Chainlink 커뮤니티 풀의 맥락에서 이 위험은 낮은 것으로 간주됩니다. 따라서 이 시나리오는 관련된 컨트랙트 중 하나가 손상되었고 사용자가 여전히 해당 컨트랙트로 마이그레이션하는 경우에만 위협이 됩니다.

**권장 완화 방안:** bypassQueue 함수에 풀 상태 확인을 추가하십시오:
```diff
function bypassQueue(
    address _account,
    uint256 _amount,
    bytes[] calldata _data
) external onlyQueueBypassController {
+    if (poolStatus != PoolStatus.OPEN) revert DepositsDisabled();
    ...
}
```

**stake.link:**
[`c595886`](https://github.com/stakedotlink/contracts/commit/c595886faef706a74bc11815b103af6670d7ed4d)에서 수정됨.

**Cyfrin:** 확인함. 권장된 완화 조치가 구현되었습니다.


### 기존 Chainlink 스테이커가 마이그레이션 요구 사항을 우회하여 대기열을 건너뛸 수 있음

**설명:** `LINKMigrator`의 목적은 사용자가 Chainlink 커뮤니티 풀의 기존 포지션을 stake.link 볼트로 마이그레이션할 수 있도록 하는 것이며, 커뮤니티 풀이 완전히 사용 중(full utilization)일 때도 포지션을 비우면 공간이 확보되므로 가능합니다. 기존 포지션이 없는 사용자의 경우 대기열 시스템(`PriorityQueue`)을 사용하여 커뮤니티 풀의 빈 슬롯을 기다립니다.

그러나 기존 포지션이 있는 사용자는 마이그레이션을 위조하여 이 메커니즘을 악용할 수 있습니다. 포지션을 다른 주소(예: 자신이 제어하는 작은 컨트랙트)로 이동함으로써 대기열을 우회하고 공간이 있다면 stake.link에서 새 포지션을 열 수 있습니다.

마이그레이션은 [`LINKMigrator::initiateMigration`](https://github.com/stakedotlink/contracts/blob/0bd5e1eecd866b2077d6887e922c4c5940a6b452/contracts/linkStaking/LINKMigrator.sol#L79-L92) 호출로 시작됩니다:

```solidity
function initiateMigration(uint256 _amount) external {
    if (_amount == 0) revert InvalidAmount();

    uint256 principal = communityPool.getStakerPrincipal(msg.sender);

    if (principal < _amount) revert InsufficientAmountStaked();
    if (!_isUnbonded(msg.sender)) revert TokensNotUnbonded();

    migrations[msg.sender] = Migration(
        uint128(principal),
        uint128(_amount),
        uint64(block.timestamp)
    );
}
```

여기서 사용자의 원금(principal)이 기록됩니다. 나중에 마이그레이션은 `transferAndCall`을 통해 완료되며, 이는 [`LINKMigrator::onTokenTransfer`](https://github.com/stakedotlink/contracts/blob/0bd5e1eecd866b2077d6887e922c4c5940a6b452/contracts/linkStaking/LINKMigrator.sol#L100-L117)를 트리거합니다:

```solidity
uint256 amountWithdrawn = migration.principalAmount -
    communityPool.getStakerPrincipal(_sender);
if (amountWithdrawn < _value) revert InsufficientTokensWithdrawn();
```

이것은 기록된 원금과 현재 원금을 비교하여 인출을 확인합니다. 그러나 커뮤니티 풀의 총 스테이킹 금액이 감소했는지는 검증하지 않습니다. 결과적으로 사용자는 자신의 포지션을 인출하여 자신이 제어하는 컨트랙트로 전송한 다음 확인을 통과하여 stake.link에 직접 입금하고 대기열을 우회할 수 있습니다.

**파급력:** Chainlink 커뮤니티 볼트에 기존 포지션이 있는 사용자는 대기열 시스템을 우회하여 stake.link 스테이킹에 직접 액세스할 수 있습니다. 이를 위해서는 청구 기간(claim period) 내에 있어야 하며, 다시 스테이킹할 충분한 LINK가 있어야 하고, Chainlink 커뮤니티 볼트에 사용 가능한 공간이 있어야 합니다. 또한 이는 본딩 기간(bonding period)을 재설정하므로, 사용자는 새 포지션과 상호 작용하기 전에 또 다른 28일(작성 시점 기준 Chainlink 본딩 기간)을 기다려야 합니다. 그럼에도 불구하고 이러한 행동은 불공정한 대기열 건너뛰기로 이어져 프로토콜의 공정성을 훼손할 수 있습니다.

**개념 증명 (Proof of Concept):** 제3의 컨트랙트를 통한 마이그레이션 및 재스테이킹을 시뮬레이션하여 대기열 우회를 보여주는 다음 테스트를 `link-migrator.ts`에 추가하십시오:
```javascript
it('can bypass queue using existing position', async () => {
  const { migrator, communityPool, accounts, token, stakingPool } = await loadFixture(
    deployFixture
  )

  // increase max pool size so we have space for the extra position
  await communityPool.setMaxPoolSize(toEther(3000))

  // deploy our small contract to hold the existing position
  const chainlinkPosition = (await deploy('ChainlinkPosition', [
    communityPool.target,
    token.target,
  ])) as ChainlinkPosition

  // get to claim period
  await communityPool.unbond()
  await time.increase(unbondingPeriod)

  // start batch transaction
  await ethers.provider.send('evm_setAutomine', [false])

  // 1. call initiate migration
  await migrator.initiateMigration(toEther(1000))

  // 2. unstake
  await communityPool.unstake(toEther(1000))

  // 3. transfer the existing position to a contract you control
  await token.transfer(chainlinkPosition.target, toEther(1000))
  await chainlinkPosition.deposit()

  // 4. transferAndCall a new position bypassing the queue
  await token.transferAndCall(
    migrator.target,
    toEther(1000),
    ethers.AbiCoder.defaultAbiCoder().encode(['bytes[]'], [[encodeVaults([])]])
  )
  await ethers.provider.send('evm_mine')
  await ethers.provider.send('evm_setAutomine', [true])

  // user has both a 1000 LINK position in stake.link StakingPool and chainlink community pool
  assert.equal(fromEther(await communityPool.getStakerPrincipal(accounts[0])), 0)
  assert.equal(fromEther(await stakingPool.balanceOf(accounts[0])), 1000)
  assert.equal(fromEther(await communityPool.getStakerPrincipal(chainlinkPosition.target)), 1000)

  // community pool is full again
  assert.equal(fromEther(await communityPool.getTotalPrincipal()), 3000)
  assert.equal(fromEther(await stakingPool.totalStaked()), 2000)
  assert.deepEqual(await migrator.migrations(accounts[0]), [0n, 0n, 0n])
})
```
`ChainlinkPosition.sol`과 함께:
```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity 0.8.15;

import "./interfaces/IStaking.sol";
import "../core/interfaces/IERC677.sol";

contract ChainlinkPosition {

    IStaking communityPool;
    IERC677 link;

    constructor(address _communityPool, address _link) {
        communityPool = IStaking(_communityPool);
        link = IERC677(_link);
    }

    function deposit() public {
        link.transferAndCall(address(communityPool), link.balanceOf(address(this)), "");
    }
}
```

**권장 완화 방안:** `LINKMigrator::onTokenTransfer`에서 커뮤니티 풀의 총 원금이 최소 `_value`만큼 감소했는지 검증하여 마이그레이션이 커뮤니티 풀에서의 실제 퇴장을 반영하는지 확인하는 것을 고려하십시오.

**stake.link:**
[`de672a7`](https://github.com/stakedotlink/contracts/commit/de672a77813d507896502c20241618230af1bd85)에서 수정됨.

**Cyfrin:** 확인함. 권장된 완화 조치가 구현되었습니다. 커뮤니티 풀 총 원금은 이제 `initiateMigration`에서 기록된 다음 `onTokenTransfer`에서 새 풀 총 원금과 비교됩니다.

\clearpage
## 정보성 (Informational)


### 사용되지 않는 오류

**설명:** [`LINKMigrator.sol#L47`](https://github.com/stakedotlink/contracts/blob/0bd5e1eecd866b2077d6887e922c4c5940a6b452/contracts/linkStaking/LINKMigrator.sol#L47)에서 `InvalidPPState` 오류는 사용되지 않습니다. 사용하거나 제거하는 것을 고려하십시오.

**stake.link:**
[`6827d9d`](https://github.com/stakedotlink/contracts/commit/6827d9df664de5132d81570f03b55ed4a482dff5)에서 수정됨.

**Cyfrin:** 확인함.


### 중요한 상태 변경 시 이벤트 방출 누락

**설명:** [`LINKMigrator::setQueueDepositMin`](https://github.com/stakedotlink/contracts/blob/0bd5e1eecd866b2077d6887e922c4c5940a6b452/contracts/linkStaking/LINKMigrator.sol#L119-L125) 및 [`PriorityPool::setQueueBypassController`](https://github.com/stakedotlink/contracts/blob/0bd5e1eecd866b2077d6887e922c4c5940a6b452/contracts/core/priorityPool/PriorityPool.sol#L678-L685)는 이벤트를 방출하지 않고 내부 상태를 변경합니다. 이벤트는 오프체인 추적 및 투명성에 중요합니다. 이러한 함수에서 이벤트를 방출하는 것을 고려하십시오.

**stake.link:**
인지함.


### 명확성을 위해 `LINKMigrator::_isUnbonded` 이름 변경 고려

**설명:** `LINKMigrator` 컨트랙트에서 [`_isUnbonded`](https://github.com/stakedotlink/contracts/blob/0bd5e1eecd866b2077d6887e922c4c5940a6b452/contracts/linkStaking/LINKMigrator.sol#L132-L137) 함수는 사용자가 현재 Chainlink 스테이킹에 대한 청구 기간 내에 있는지 확인합니다:

```solidity
function _isUnbonded(address _account) private view returns (bool) {
    uint256 unbondingPeriodEndsAt = communityPool.getUnbondingEndsAt(_account);
    if (unbondingPeriodEndsAt == 0 || block.timestamp < unbondingPeriodEndsAt) return false;

    return block.timestamp <= communityPool.getClaimPeriodEndsAt(_account);
}
```

기능적으로는 정확하지만, `_isUnbonded`라는 이름은 청구 기간에 있는지 여부를 구체적으로 확인하므로 그 목적을 명확하게 전달하지 못할 수 있습니다. 명확성을 높이고 [`StakingPoolBase::_inClaimPeriod`](https://etherscan.io/address/0xbc10f2e862ed4502144c7d632a3459f49dfcdb5e#code)와 같은 Chainlink의 명명 규칙과의 일관성을 위해 이름을 변경하면 의도를 더 즉각적으로 명확하게 할 수 있습니다:

```solidity
function _inClaimPeriod(Staker storage staker) private view returns (bool) {
  if (staker.unbondingPeriodEndsAt == 0 || block.timestamp < staker.unbondingPeriodEndsAt) {
    return false;
  }

  return block.timestamp <= staker.claimPeriodEndsAt;
}
```

**권장 완화 방안:** 로직을 더 잘 반영하고 코드 가독성을 높이기 위해 `_isUnbonded`를 `_inClaimPeriod`로 이름을 변경하는 것을 고려하십시오.

**stake.link:**
[`9d710bf`](https://github.com/stakedotlink/contracts/commit/9d710bf35304e9b45ed1ad8468714915817904a1)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage
## 가스 최적화 (Gas Optimization)


### 변경되지 않는 상태 변수는 불변(immutable)일 수 있음

**설명:** 다음 중 어느 것도 생성자 외부에서 변경되지 않습니다:
* [`LINKMigrator.linkToken`](https://github.com/stakedotlink/contracts/blob/0bd5e1eecd866b2077d6887e922c4c5940a6b452/contracts/linkStaking/LINKMigrator.sol#L22)
* [`LINKMigrator.communityPool`](https://github.com/stakedotlink/contracts/blob/0bd5e1eecd866b2077d6887e922c4c5940a6b452/contracts/linkStaking/LINKMigrator.sol#L24)
* [`LINKMigrator.priorityPool`](https://github.com/stakedotlink/contracts/blob/0bd5e1eecd866b2077d6887e922c4c5940a6b452/contracts/linkStaking/LINKMigrator.sol#L27)

접근 시 가스를 절약하기 위해 `immutable`로 만드는 것을 고려하십시오.

**stake.link:**
[`6f9d9b7`](https://github.com/stakedotlink/contracts/commit/6f9d9b77201184d2dfc8f4c06f3430f2d360db24)에서 수정됨.

**Cyfrin:** 확인함.


### `LINKMigrator.Migration`의 비효율적인 스토리지 레이아웃

**설명:** 다음은 [`LINKMigrator.Migration`](https://github.com/stakedotlink/contracts/blob/0bd5e1eecd866b2077d6887e922c4c5940a6b452/contracts/linkStaking/LINKMigrator.sol#L31-L38) 구조체입니다:

```solidity
struct Migration {
    // amount of principal staked in Chainlink community pool
    uint128 principalAmount;
    // amount to migrate
    uint128 amount;
    // timestamp when migration was initiated
    uint64 timestamp;
}
```

이 구조체는 256비트 이상을 차지하므로 두 개의 스토리지 슬롯에 걸쳐 있습니다. 그러나 총 LINK 공급량이 10억 개(10^9 \* 10^18)이므로 모든 LINK 금액은 `uint96`에 안전하게 저장할 수 있으며, 이는 `uint96`으로 표현할 수 있는 최대값(\~7.9 \* 10^28)보다 훨씬 낮습니다. 두 금액 필드를 모두 `uint96`으로 변경하면 구조체는 `uint96 + uint96 + uint64`로 구성되어 단일 256비트 스토리지 슬롯에 깔끔하게 맞습니다.

**Cyfrin:** [`de672a7`](https://github.com/stakedotlink/contracts/commit/de672a77813d507896502c20241618230af1bd85)의 수정 후 해당 사항 없음.

