**수석 감사자**

[0kage](https://twitter.com/0kage_eth)

[Hans](https://twitter.com/hansfriese)

**보조 감사자**



---

# 발견 사항 (Findings)
## 중대 위험 (Critical Risk)


### `CommunityVCS::getDepositChange`에서 볼트를 순차적으로 채운다는 암시적 가정으로 인해 스테이커에게 직접적인 손실 발생

**설명:** `CommunityVCS` 계약의 `getDepositChange` 함수는 프로토콜의 총 잔액을 상당히 적게 보고하게 만들 수 있습니다. 이 함수는 볼트가 순차적으로 채워진다는 가정하에 설계되었지만, 인출 시 이 가정은 쉽게 위반될 수 있습니다.

주어진 `curUnbondedVaultGroup`의 볼트 ID는 순차적이지 않고 `numVaultGroups`(Community Vault Controller Strategy의 경우 5로 설정됨)만큼 떨어져 있습니다. 인덱스에서 전체 인출이 발생하면 `getDepositChange`는 모든 후속 볼트에 대한 예치금을 포함하는 것을 중단합니다.

CommunityVCS의 취약한 코드:

```solidity
function getDepositChange() public view override returns (int) {
    uint256 totalBalance = token.balanceOf(address(this));
    for (uint256 i = 0; i < vaults.length; ++i) {
        uint256 vaultDeposits = vaults[i].getTotalDeposits();
        if (vaultDeposits == 0) break;
        totalBalance += vaultDeposits;
    }
    return int(totalBalance) - int(totalDeposits);
}
```

이 함수는 빈 볼트를 처음 만나면 계산을 중단합니다. `StakingPool`이 `updateDeposits`를 호출할 때 잘못된 변경 사항이 `totalStaked` 값을 업데이트하는 데 사용됩니다. 이는 슬래싱(slashing) 중에 발생하는 것과 유사하게 `totalStaked`가 인위적으로 감소하여 사용자에게 회계상 손실을 초래합니다.

**파급력:** 더 낮은 인덱스를 가진 빈 볼트가 있는 경우 비순차적 볼트의 총 예치금이 `totalBalance`에 포함되지 않습니다. 이는 슬래싱과 유사한 효과를 가집니다. 주요 효과는 다음과 같습니다:
- Staking Pool의 `TotalStaked`가 감소하여 스테이커에게 직접적인 손실 발생
- 수수료 수취인이 수수료를 적립하지 못함

**개념 증명 (PoC):** 아래 PoC는 첫 번째 볼트가 완전히 인출되면 `getDepositChange`가 모든 후속 볼트에 대한 볼트 예치금을 포함하는 것을 중단함을 보여줍니다.

```javascript
it(`sequential deposits assumption in Community VCS can be attacked`, async () => {
    const {
      accounts,
      adrs,
      token,
      rewardsController,
      stakingController,
      strategy,
      stakingPool,
      updateVaultGroups,
    } = await deployCommunityVaultFixture()

    await strategy.deposit(toEther(1200), encodeVaults([]))

    await updateVaultGroups([0, 5, 10], 0)
    await time.increase(claimPeriod)
    await updateVaultGroups([1, 6, 11], 0)
    await time.increase(claimPeriod)
    await updateVaultGroups([2, 7], 0)
    await time.increase(claimPeriod)
    await updateVaultGroups([3, 8], 0)
    await time.increase(claimPeriod)
    await updateVaultGroups([4, 9], 300)
    await time.increase(claimPeriod)
    await updateVaultGroups([0, 5, 10], 300)

    //@audit withdraw from the first vault
    await strategy.withdraw(toEther(100), encodeVaults([1]))

    let vaults = await strategy.getVaults()

    let totalDeposits = 0

    for (let i = 0; i < 15; i++) {
      let vault = await ethers.getContractAt('CommunityVault', vaults[i])
      totalDeposits += fromEther(await vault.getTotalDeposits())
    }

    console.log(`total deposits`, totalDeposits)
    assert.equal(totalDeposits, 1100) //@audit 1200 (deposit) - 100 (withdrawn)

    // const strategyTokenBalance = fromEther(await token.balanceOf(adrs.strategy))
    // console.log(`strategy token balance`, totalDepositsCalculated)
    const depositChange = fromEther(await strategy.getDepositChange())

    assert.equal(depositChange, -1000) //@audit This is equivalent to slashing
    //@audit -> deposit Change should be 0 at this point, instead it is -1000
    //@audit only the 0'th vault deposit is considered in the deposit change calculation
  })
```

**권장 완화 조치:** `getDepositChange`에 대한 기본 구현을 사용하는 것을 고려하십시오. 현재 인출 로직에서는 순차적 예치 가정이 쉽게 위반될 수 있습니다.

**Stake.link**
[e349a7d](https://github.com/stakedotlink/contracts/pull/108/commits/e349a7d20b5bd45400652f76e38019d6df764327)에서 수정되었습니다.

**Cyfrin**
확인됨. `CommunityVCS`에서 `getDepositChange` 재정의가 제거되었습니다.


### 업그레이드 중 스토리지 간격 불일치로 인해 볼트 컨트롤러 전략에서 스토리지 충돌 발생

**설명:** [`CommunityVCS`](https://etherscan.io/address/0x96418d70832d08cf683be81ee9890e1337fad41b#code)와 [`OperatorVCS`](https://etherscan.io/address/0x584338dabae9e5429c334fc1ad41c46ac007bc29#code)는 모두 이미 메인넷에 배포된 업그레이드 가능한 계약입니다.

감사된 계약은 이를 업그레이드하기 위한 것입니다. 그러나 상속된 계약 `VaultControllerStrategy`의 스토리지 간격(gap)에 문제가 있습니다:

배포된 계약에서 기본으로 사용되는 `VaultControllerStrategy`의 스토리지 설정은 다음과 같습니다:
```solidity
    IStaking public stakeController;       // 1
    Fee[] internal fees;                   // 2

    address public vaultImplementation;    // 3

    IVault[] internal vaults;              // 4
    uint256 internal totalDeposits;        // 5
    uint256 public totalPrincipalDeposits; // 6
    uint256 public indexOfLastFullVault;   // 7

    uint256 public maxDepositSizeBP;       // 8

    uint256[9] private __gap;              // 8 + 9 = 17
```

그리고 업그레이드된 `VaultControllerStrategy` 코드에서:
```solidity
    IStaking public stakeController;               // 1
    Fee[] internal fees;                           // 2

    address public vaultImplementation;            // 3

    IVault[] internal vaults;                      // 4
    uint256 internal totalDeposits;                // 5
    uint256 public totalPrincipalDeposits;         // 6

    uint256 public maxDepositSizeBP;               // 7

    IFundFlowController public fundFlowController; // 8
    uint256 internal totalUnbonded;                // 9

    VaultGroup[] public vaultGroups;               // 10
    GlobalVaultState public globalVaultState;      // 11
    uint256 internal vaultMaxDeposits;             // 12

    uint256[6] private __gap;                      // 12 + 6 = 18!
```

`totalUnbonded`, `vaultGroups`, `globalVaultState` 및 `vaultMaxDeposits`라는 4개의 새 슬롯이 추가되었지만 간격은 3개(`9` -> `6`)만 감소했습니다. 따라서 업그레이드된 `VaultControllerStrategy`에서 사용하는 총 스토리지 슬롯은 `1`만큼 증가하여 `Community` 및 `OperatorVCS` 구현의 스토리지를 침범하게 됩니다.

참고로 일부 변수가 이동하고 이름이 변경되었지만(`maxDepositSizeBP` 및 `indexOfLastFullVault`) 이는 `initializer`에서 처리됩니다.

**파급력:** `CommunityVCS`의 경우 영향은 상당히 적습니다. `CommunityVCS`의 스토리지는 다음과 같습니다:
```solidity
contract CommunityVCS is VaultControllerStrategy {
    uint128 public vaultDeploymentThreshold;
    uint128 public vaultDeploymentAmount;
```
추가 간격으로 인해 `vaultDeploymentThreshold`는 `vaultDeploymentAmount`(작성 당시 온체인에서 `6`)의 값을 얻게 됩니다. 그리고 `vaultDeploymentAmount`는 `0`이 됩니다.

이들은 `CommunityVCS::performUpkeep`에서 사용됩니다:
```solidity
    function performUpkeep(bytes calldata) external {
        if ((vaults.length - globalVaultState.depositIndex) >= vaultDeploymentThreshold)
            revert VaultsAboveThreshold();
        _deployVaults(vaultDeploymentAmount);
    }
```
`vaultDeploymentAmount`가 `0`이므로 문제가 발견되고 값들이 업데이트될 때까지(`setVaultDeploymentParams` 사용) 새로운 볼트가 배포되지 않습니다.

이로 인해 기껏해야 새로운 볼트가 배포되지 않아 예치금이 잠시 차단될 수 있습니다.

그러나 `OperatorVCS`의 경우 영향은 더 심각합니다:
다음은 `OperatorVCS`의 스토리지 레이아웃입니다:
```solidity
contract OperatorVCS is VaultControllerStrategy {
    using SafeERC20Upgradeable for IERC20Upgradeable;

    uint256 public operatorRewardPercentage;
    uint256 private unclaimedOperatorRewards;
```
`operatorRewardPercentage`는 `unclaimedOperatorRewards`(작성 당시 `4914838471043033862842`, `4.9e21`)의 값을 얻게 됩니다.

이것은 `OperatorVault`에서 획득한 보상 중 운영자에게 돌아갈 부분을 계산하는 데 사용됩니다:
```solidity
            opRewards =
                (uint256(depositChange) *
                    IOperatorVCS(vaultController).operatorRewardPercentage()) /
                10000;
```
이것은 궁극적으로 프로토콜에 의해 주기적으로 호출되는 `StakingPool::updateStrategyRewards`를 호출하여 트리거됩니다. `updateStrategyRewards`(`OperatorVault::updateDeposits`를 차례로 호출함)가 스토리지 충돌이 발견되기 전에 호출되면, `OperatorVaults`는 엄청난 양의 보상 지분을 운영자에게 나눠주게 됩니다. 이는 다른 사람이 보유한 각 지분의 가치를 크게 떨어뜨리고, 이를 인출하는 운영자는 풀에서 많은 `LINK`를 가져갈 수 있습니다.

`StakingPool::updateStrategyRewards`는 누구나 호출할 수 있으므로 악의적인 운영자가 이를 파악하고 스스로 호출하여 업그레이드를 백런(backrun)할 수 있습니다.

**권장 완화 조치:** 업그레이드된 `VaultControllerStrategy`의 스토리지 간격을 `5`로 줄이는 것을 고려하십시오.

**Stake.link**
[3107a31](https://github.com/stakedotlink/contracts/pull/108/commits/3107a31485a84d699e127648e85a43573de31390)에서 수정되었습니다.

**Cyfrin**
확인됨. 스토리지 간격이 올바르게 할당되었습니다.

\clearpage
## 고위험 (High Risk)


### `FundController::performUpkeep`의 접근 제어 부족으로 공격자가 인출을 차단할 수 있음

**설명:** 누구나 `FundFlowController`에서 `performUpkeep`을 수행할 수 있습니다. 이것은 `VaultControllerStrategy::updateVaultGroups`를 통해 볼트 컨트롤러의 비결합(unbonded) 볼트 그룹을 회전시킵니다.

`FundFlowController::performUpkeep`을 호출하는 사용자는 Chainlink 스테이킹 풀에서 결합 해제(unbond)할 볼트와 새로운 `totalUnbonded` 금액과 같은 `VaultControllerStrategy::updateVaultGroups`에 대한 세부 정보도 제공합니다:
```solidity
    function updateVaultGroups(
        uint256[] calldata _curGroupVaultsToUnbond,
        uint256 _nextGroup,
        uint256 _nextGroupTotalUnbonded
    ) external onlyFundFlowController {
        for (uint256 i = 0; i < _curGroupVaultsToUnbond.length; ++i) {
            vaults[_curGroupVaultsToUnbond[i]].unbond();
        }

        globalVaultState.curUnbondedVaultGroup = uint64(_nextGroup);
        totalUnbonded = _nextGroupTotalUnbonded;
    }
```
이 두 매개변수는 검증되지 않습니다. 따라서 사용자는 결합 해제할 볼트나 새로운 `totalUnbonded` 금액을 임의로 제공할 수 있습니다.

**파급력:** `_nextGroupTotalUnbonded`로 `0`을 제공하면 `VaultControllerStrategy::withdraw`의 이 확인으로 인해 인출이 효과적으로 차단됩니다:
```solidity
    function withdraw(uint256 _amount, bytes calldata _data) external onlyStakingPool {
        if (!fundFlowController.claimPeriodActive() || _amount > totalUnbonded)
            revert InsufficientTokensUnbonded();
```

`FundFlowController::performUpkeep`은 특정 시간(새 볼트를 결합 해제할 수 있을 때)에만 호출할 수 있으므로 다음 호출이 가능할 때까지 인출을 차단할 수 있습니다. 그때 공격자는 인출을 계속 차단하기 위해 이것을 다시 호출할 수 있습니다.

`totalUnbonded`를 `0`으로 설정하면 영향은 덜하지만 `_depositToVaults`에서 활성 비결합이 있는 볼트에 대한 예치도 중단됩니다:
```solidity
                if (vault.unbondingActive()) {
                    totalRebonded += deposits;
                }
...

        if (totalRebonded != 0) totalUnbonded -= totalRebonded;
```

잘못된 볼트를 제공하면 인출 가능한 볼트의 순서를 변경하여 인출에 혼란을 줄 수 있습니다. `FundFlowController`와 `VaultControllerStrategy`는 그룹이 활성 상태일 때 그룹의 예치금이 있는 모든 볼트를 사용할 수 있다고 가정하기 때문입니다.

**개념 증명 (PoC):** `vault-controller-strategy.test.ts`에 추가할 수 있는 두 가지 테스트(`totalUnbonded`용 및 비결합 볼트용):
```javascript
  it('performUpkeep, updateVaultGroups can be called by anyone and lacks validation for totalUnbonded', async () => {
    const { adrs, strategy, token, stakingController, vaults, updateVaultGroups } =
      await loadFixture(deployFixture)

    // Deposit into vaults
    await strategy.deposit(toEther(500), encodeVaults([]))
    assert.equal(fromEther(await token.balanceOf(adrs.stakingController)), 500)
    for (let i = 0; i < 5; i++) {
      assert.equal(fromEther(await stakingController.getStakerPrincipal(vaults[i])), 100)
    }
    assert.equal(Number((await strategy.globalVaultState())[3]), 5)

    // do the initial rotation to get a vault into claim period
    await updateVaultGroups([0, 5, 10], 0)
    await time.increase(claimPeriod)
    await updateVaultGroups([1, 6, 11], 0)
    await time.increase(claimPeriod)
    await updateVaultGroups([2, 7], 0)
    await time.increase(claimPeriod)
    await updateVaultGroups([3, 8], 0)
    await time.increase(claimPeriod)

    // user calls `fundFlowController.performUpkeep` with erroneous `totalUnbonded`
    await updateVaultGroups([4, 9], 0)

    // causing withdrawals to stop working
    await expect(
      strategy.withdraw(toEther(50), encodeVaults([0, 5]))
    ).to.be.revertedWithCustomError(strategy, 'InsufficientTokensUnbonded()')
  })

  it('performUpkeep, updateVaultGroups can be called by anyone and lacks validation for unbonded vaults', async () => {
    const { adrs, strategy, token, stakingController, vaults, updateVaultGroups } =
      await loadFixture(deployFixture)

    // Deposit into vaults
    await strategy.deposit(toEther(500), encodeVaults([]))
    assert.equal(fromEther(await token.balanceOf(adrs.stakingController)), 500)
    for (let i = 0; i < 5; i++) {
      assert.equal(fromEther(await stakingController.getStakerPrincipal(vaults[i])), 100)
    }
    assert.equal(Number((await strategy.globalVaultState())[3]), 5)

    // calling `fundFlowController.performUpkeep` with vaults in other groups
    await updateVaultGroups([0, 1, 2, 3, 4], 0)
    await time.increase(claimPeriod)
    await expect(
      updateVaultGroups([1, 6, 11], 0)
    ).to.be.revertedWithCustomError(stakingController, 'UnbondingPeriodActive()')
  })
```

**권장 완화 조치:** 사용자가 볼트나 금액을 전혀 제공하지 못하도록 하는 것을 고려하십시오. `FundFlowController`에는 이 모든 데이터를 수집하는 `view` 함수 `checkUpkeep`이 있습니다.

`checkUpkeep`과 `performUpkeep`을 연결하여 호출자가 데이터를 제공하지 않도록 하는 것이 좋습니다. 이것은 또한 문제 7.3.6 [*`FundFlowController::performUpkeep`의 잠재적인 오래된 데이터로 인해 잘못된 상태 업데이트가 발생할 수 있음*](#potential-stale-data-in-fundflowcontrollerperformupkeep-can-lead-to-incorrect-state-updates)도 해결합니다.

**Stake.link**
[53d6090](https://github.com/stakedotlink/contracts/pull/108/commits/53d60901c171f4b436dcb4b22fe6549ba5dcef05)에서 수정되었습니다.

**Cyfrin**
확인됨. `performUpkeep`은 더 이상 인코딩된 입력을 받지 않습니다.


### 공격자가 볼트 그룹의 총 예치 공간을 거의 무한대로 부풀릴 수 있음

**설명:** `VaultControllerStrategy::deposit` 함수는 `maxDeposits`(Chainlink에서 가져옴)와 `vaultMaxDeposits` 간의 차이를 기반으로 `totalDepositRoom`을 늘리지만, 이후에 `vaultMaxDeposits`를 업데이트하지 못합니다. 결과적으로 예치할 때마다 실제 예치 금액에 관계없이 볼트 그룹 예치 공간이 `maxDeposits - vaultMaxDeposits`만큼 증가합니다.

또한 `VaultControllerStrategy`와 상호 작용하는 `StakingPool` 계약에는 모든 사용자가 호출할 수 있는 공개 `depositLiquidity` 함수가 있습니다. 이 함수는 예치에 `StakingPool` 계약의 현재 잔액을 사용하며, 이는 소량의 토큰을 계약으로 직접 전송하여 조작할 수 있습니다.

이러한 문제의 조합으로 악의적인 행위자는 다음을 수행할 수 있습니다:
- 소량의 토큰을 계약으로 전송한 후 신중하게 조작된 데이터로 StakingPool에서 depositLiquidity를 호출하여 예치 프로세스를 조작합니다.
- 예치 함수를 반복적으로 트리거하여 VaultControllerStrategy에서 `totalDepositRoom`을 인위적으로 부풀립니다.

이 공격 벡터는 Chainlink가 볼트 한도를 늘릴 때마다 가능합니다.

**파급력:** 공격자가 예치를 반복적으로 호출하여 볼트 그룹 총 예치 공간을 거의 무한대로 늘릴 수 있습니다. 이는 2가지 심각한 부작용을 가집니다:

- 사용되지 않은 볼트는 예치 인덱스보다 낮은 ID를 가진 볼트가 가득 찼더라도 예치를 받지 못합니다.
- 인출은 예치 공간을 지속적으로 증가시키므로, 반복적인 공격은 극단적인 경우 예치 공간 오버플로를 일으켜 모든 인출을 차단할 수 있습니다.

**개념 증명 (PoC):**
```typescript
  it('inflate deposit room in vault groups to infinity', async () => {
    const {
      token,
      accounts,
      signers,
      adrs,
      strategy,
      priorityPool,
      stakingPool,
      stakingController,
      rewardsController,
      vaults,
    } = await loadFixture(deployFixture)

    const [, totalDepRoomBefore] = await strategy.vaultGroups(1)
    await stakingController.setDepositLimits(toEther(10), toEther(120))
    await stakingPool.deposit(accounts[0], 1 /*dust amount*/, [encodeVaults([0, 1, 2, 3, 4])])
    const [, totalDepRoomAfterFirstDeposit] = await strategy.vaultGroups(0)

    //@audit after first deposit, deposit room increases to 20 ether
    assert.equal(fromEther(totalDepRoomAfterFirstDeposit), 20)
    await stakingPool.deposit(accounts[0], 1 /*dust amount*/, [encodeVaults([4])])

    //@audit after first deposit, deposit room increases to 40 ether
    const [, totalDepRoomAfterSecondDeposit] = await strategy.vaultGroups(0)
    assert.equal(fromEther(totalDepRoomAfterSecondDeposit), 40)

    //@audit this can theoretically increase to infinity - disrupting the deposit logic
  })
```

**권장 완화 조치:** 모든 볼트 그룹 총 예치 공간이 업데이트되면 `vaultMaxDeposits`를 업데이트하십시오.

**Stake.link**
[04c2f10](https://github.com/stakedotlink/contracts/pull/108/commits/04c2f105cde784cb8b0f89f3fb835090213fa13d)에서 수정되었습니다.

**Cyfrin**
확인됨. `vaultMaxDeposits`가 올바르게 업데이트됩니다.


### FundFlowController에서 예치 데이터의 잘못된 처리

**설명:** `FundFlowController` 계약의 `getDepositData` 함수는 운영자 볼트가 모든 예치금을 수용할 수 있는 경우를 잘못 처리합니다. 이로 인해 빈 예치 데이터가 운영자 볼트로 전달되어 예치가 실패하거나 잘못 할당될 수 있습니다.

`FundFlowController`의 관련 코드 스니펫:

```solidity
function getDepositData(uint256 _toDeposit) external view returns (bytes[] memory) {
    uint256 toDeposit = 2 * _toDeposit;
    bytes[] memory depositData = new bytes[](2);

    (
        uint64[] memory opVaultDepositOrder,
        uint256 opVaultsTotalToDeposit
    ) = _getVaultDepositOrder(operatorVCS, toDeposit);
    depositData[0] = abi.encode(opVaultDepositOrder);

    if (opVaultsTotalToDeposit < toDeposit) {
        (uint64[] memory comVaultDepositOrder, ) = _getVaultDepositOrder(
            communityVCS,
            toDeposit - opVaultsTotalToDeposit
        );
        depositData[1] = abi.encode(comVaultDepositOrder);
    } else {
        depositData[0] = abi.encode(new uint64[](0)); // @audit: Should be depositData[1]
    }

    return depositData;
}
```

운영자 볼트가 전체 예치금을 처리할 수 있을 때 볼트 ID 순서가 포함된 depositData[0]이 삭제됩니다. 이 처리는 `if-else` 로직 흐름 모두에서 `withdrawalData[0]`만 올바르게 인코딩하는 `getWithdrawalData` 함수와도 일관성이 없습니다.

```solidity
function getWithdrawalData(uint256 _toWithdraw) external view returns (bytes[] memory) {
    uint256 toWithdraw = 2 * _toWithdraw;
    bytes[] memory withdrawalData = new bytes[](2);

    (
        uint64[] memory comVaultWithdrawalOrder,
        uint256 comVaultsTotalToWithdraw
    ) = _getVaultWithdrawalOrder(communityVCS, toWithdraw);
    withdrawalData[1] = abi.encode(comVaultWithdrawalOrder);

    if (comVaultsTotalToWithdraw < toWithdraw) {
        (uint64[] memory opVaultWithdrawalOrder, ) = _getVaultWithdrawalOrder(
            operatorVCS,
            toWithdraw - comVaultsTotalToWithdraw
        );
        withdrawalData[0] = abi.encode(opVaultWithdrawalOrder);
    } else {
        withdrawalData[0] = abi.encode(new uint64[](0)); //@audit correctly encoding this instead of withdrawalData[1]
    }

    return withdrawalData;
}
```

**파급력:** `getDepositData`는 대기 중인 토큰을 예치하기 위해 우선순위 풀(priority pool)에 전달되는 vaultId를 계산하기 위해 `PPKeeper` 계약에 의해 호출됩니다. 이 버그는 운영자 볼트에 모든 예치금을 수용할 수 있는 충분한 용량이 있을 때 예치가 실패하거나 잘못 할당되게 할 수 있습니다.

**권장 완화 조치:** 운영자 볼트가 모든 예치금을 수용할 수 있는 경우를 올바르게 처리하도록 `getDepositData` 함수를 수정하는 것을 고려하십시오. 수정된 코드는 다음과 같아야 합니다:

```diff
if (opVaultsTotalToDeposit < toDeposit) {
    (uint64[] memory comVaultDepositOrder, ) = _getVaultDepositOrder(
        communityVCS,
        toDeposit - opVaultsTotalToDeposit
    );
    depositData[1] = abi.encode(comVaultDepositOrder);
} else {
-    depositData[0] = abi.encode(new uint64[](0));
+   depositData[1] = abi.encode(new uint64[](0));
}
```

**Stake.link**
[7af3ec0](https://github.com/stakedotlink/contracts/pull/108/commits/7af3ec0b13303291b4c85975e2e353ca89ec51a6)에서 수정되었습니다.

**Cyfrin**
확인됨. 권장된 대로 수정되었습니다.


### 호출 선택자(selector) 불일치로 인해 언스테이킹(unstaking)이 revert 됨

**설명:** Chainlink 스테이킹에서 `StakingPoolBase::unstake` 호출은 다음과 같습니다:
```solidity
  function unstake(uint256 amount) external {
```
그러나 `Vault::withdraw`에서는 다음과 같이 호출됩니다:
```solidity
        stakeController.unstake(_amount, false);
```

**파급력:** 모든 볼트가 업데이트될 때까지 언스테이킹이 작동하지 않습니다.

**권장 완화 조치:** `unstake`에서 `false`를 제거하는 것을 고려하십시오:
```diff
-       stakeController.unstake(_amount, false);
+       stakeController.unstake(_amount);
```

**Stake.link**
[c563a62](https://github.com/stakedotlink/contracts/pull/108/commits/c563a6273d5fe586914c34c910bdef404458342b)에서 수정되었습니다.

**Cyfrin**
확인됨. 권장된 대로 수정되었습니다.


### 플래시 론 공격으로 CommunityVCS에 관리할 수 없는 수의 유령(ghost) 볼트가 생성될 수 있음

**설명:** `CommunityVCS`의 현재 구현은 공격자가 볼트의 수를 임의의 큰 숫자로 인위적으로 부풀릴 수 있는 잠재적인 공격 벡터를 허용합니다. 이 공격은 `globalState.depositIndex`가 단조 증가하며 이전 볼트가 비어도 재설정되지 않는다는 사실을 악용합니다. 결과적으로 이 공격을 반복할 때마다 `depositIndex`가 더 밀려나고 절대 사용되지 않을 유령 볼트가 생성됩니다.

다음 공격 벡터를 고려하십시오:
1. 다량의 토큰에 대한 플래시 론을 받습니다. `vaultDeploymentThreshold` 조건을 트리거하도록 토큰 금액을 선택합니다.
2. 볼트 ID를 지정하지 않고 Priority Pool에 이 토큰을 예치합니다.
3. 이렇게 하면 Staking Pool에 예치가 트리거되고 CommunityVCS의 예치 함수가 호출됩니다.
4. 볼트 ID가 제공되지 않았으므로 예치는 `globalState.depositIndex`에서 시작하여 예치가 완전히 할당되거나 모든 볼트가 채워질 때까지 계속됩니다. 이것은 `globalState.depositIndex`를 더 큰 숫자로 업데이트합니다.
5. CommunityVCS에서 performUpkeep을 호출하여 새 볼트를 배포합니다.
6. 1-5단계를 반복합니다.
7. StakingPool의 모든 전략에서 충분한 종료 유동성이 있다고 가정하면 공격자는 예치된 모든 토큰을 인출하여 플래시 론을 상환할 수 있습니다.

**파급력:** 불필요하게 많은 수의 볼트를 관리하면 프로토콜 운영 및 유지 관리가 심각하게 복잡해집니다. 정확한 볼트 회계를 유지하는 데 필수적인 `updateDeposits`와 같은 중요한 기능을 실행하는 데 비용이 매우 많이 들게 됩니다. 이는 모든 볼트의 최신 총 예치금을 추적하는 `getDepositChange` 함수가 사용 가능한 모든 볼트를 반복해야 하기 때문입니다. 극단적인 경우 모든 볼트를 반복해야 하는 기능에 대한 서비스 거부(DoS) 조건이 발생할 수 있습니다.

**개념 증명 (PoC):**
```typescript
  it(`flash loan attack to max out community vaults`, async () => {
    const {
      token,
      accounts,
      signers,
      adrs,
      strategy,
      priorityPool,
      stakingPool,
      stakingController,
      rewardsController,
      updateVaultGroups,
    } = await loadFixture(deployCommunityVaultFixtureWithStrategyMock)
    //@note this fixture uses a strategy mock to simulate a strategy with a max deposit of 900 tokens
    //@note by now, that strategy is already full -> so any further deposits go into the Community VCS

    console.log(`strategies length ${(await stakingPool.getStrategies()).length}`)
    assert.equal((await strategy.getVaults()).length, 20)
    await stakingPool.deposit(accounts[0], toEther(1500), [encodeVaults([]), encodeVaults([])])

    // get initial deposit index
    const [, , , depositIndex] = await strategy.globalVaultState()
    console.log(`deposit index`, depositIndex.toString())

    await strategy.performUpkeep(encodeVaults([]))
    assert.equal((await strategy.getVaults()).length, 40) //@audit 20 new vaults are created

    await updateVaultGroups([0, 5, 10], 0)
    await time.increase(claimPeriod)
    await updateVaultGroups([1, 6, 11], 0)
    await time.increase(claimPeriod)
    await updateVaultGroups([2, 7], 0)
    await time.increase(claimPeriod)
    await updateVaultGroups([3, 8], 0)
    await time.increase(claimPeriod)
    await updateVaultGroups([4, 9], 300)

    //@note a user can flashloan 1100 tokens -> deposit them with no vaultIds -> deposit index moves to 26
    await stakingPool.setPriorityPool(adrs.priorityPool)
    await token.approve(adrs.strategy, ethers.MaxUint256)
    await token.connect(signers[2]).approve(priorityPool.target, ethers.MaxUint256)
    await priorityPool
      .connect(signers[2])
      .deposit(toEther(1100), false, [encodeVaults([]), encodeVaults([])])

    const [, , , newDepositIndex] = await strategy.globalVaultState()

    //@note user can then performUpkeep (or wait for keeper to do this) to further increase vaults to 60
    await strategy.performUpkeep(encodeVaults([]))
    assert.equal((await strategy.getVaults()).length, 60) //@audit 20 new vaults are created again

    //@note 300 is now available for withdrawal
    //@note 800 is withdrawable from first strategy
    //@note creating a mock staking pool to simulate this

    //@note 800 + 300 -> total withdrawable is 1100
    //@note user can then withdraw complete amount to repay flash without queueing

    await stakingPool.connect(signers[2]).approve(adrs.priorityPool, ethers.MaxUint256)
    await priorityPool
      .connect(signers[2])
      .withdraw(toEther(1100), toEther(1100), toEther(1100), [], false, false, [
        encodeVaults([]),
        encodeVaults([0, 5, 10]),
      ])
  })
```

**권장 완화 조치:** 항상 새 볼트를 만드는 대신 빈 볼트를 재활용하는 메커니즘을 구현하는 것을 고려하십시오. 또한 플래시 론 벡터를 무효화하려면 인출에 시간 지연을 추가하는 것을 고려하십시오.

**Stake.link**
[9dcac24](https://github.com/stakedotlink/contracts/pull/108/commits/9dcac2435db6d3a1e7d0d222e91b6ffadf99484d)에서 수정되었습니다.

**Cyfrin**
확인됨. 즉각적인 스테이킹 풀 인출이 허용되지 않습니다. 또한 볼트에 예치하는 동안 현재 볼트 그룹 예치 공간도 고려됩니다. 두 가지 변경 사항이 결합되어 강조 표시된 위험을 완화합니다.

\clearpage
## 중급 위험 (Medium Risk)


### `WithdrawalPool::withdrawalBatches` 배열의 크기가 계속 증가하여 잠재적으로 서비스 거부(DoS)를 유발할 수 있음

**설명:** 인출이 확정될 때마다 새 batchId가 `withdrawalBatches` 배열에 추가됩니다. 시간이 지남에 따라 이 배열은 계속 커지고 크기가 매우 커질 수 있습니다. 사용자가 직접 인출 대기열을 트리거할 수 있다는 점에 유의하는 것이 중요합니다.

**파급력:** `getBatchIds` 및 `getFinalizedWithdrawalIdsByOwner`의 잠재적인 서비스 거부

**권장 완화 조치:** 배치의 `indexOfLastWithdrawal`이 첫 번째 대기 중인 인출의 ID(`queuedWithdrawals[0]`)보다 작은 경우, 해당 배치 및 이전 배치의 모든 인출이 완전히 처리되었음을 의미합니다. 이것을 사용하여 주기적으로 컷오프(cutoff) 배치 `id`를 계산할 수 있습니다.

다음 구현을 고려하십시오:
- 컷오프 배치 찾기: indexOfLastWithdrawal < queuedWithdrawals[0]인 마지막 배치를 찾을 때까지 처음부터 withdrawalBatches를 반복합니다.
- 삭제 또는 보관: 이 컷오프 배치까지의 모든 배치를 안전하게 삭제하거나 보관할 수 있습니다.

**Stake.link**
[7889f75](https://github.com/stakedotlink/contracts/pull/108/commits/7889f75917d9481b5ca8fc66ceba2ac3f7f25371)에서 수정되었습니다.

**Cyfrin**
확인됨. 권장된 대로 수정되었습니다.


### 공격자가 StakingPool::depositLiquidity() 호출을 선행 실행(front-running)하여 사용자가 우선순위 풀에 예치하는 것을 거부할 수 있음

**설명:** `VaultControllerStrategy` 계약은 우선순위 풀이 사용자의 자금을 전략의 특정 볼트에 예치하는 것을 방지할 수 있는 선행 실행 공격에 취약합니다. 취약점은 현재 예치 상태를 추적하기 위해 `globalState.groupDepositIndex`를 사용하는 `deposit` 함수에 있습니다. 공격자는 사용되지 않은 예치금을 전략에 예치하는 `depositLiquidity`를 호출하여 합법적인 예치를 선행 실행함으로써 이를 악용할 수 있습니다.

그리핑(griefing) 공격은 다음과 같이 작동합니다:

1. 우선순위 풀은 볼트 ID 목록을 지정하여 자금을 예치할 트랜잭션을 준비합니다.
2. 공격자는 멤풀(mempool)에서 이 대기 중인 트랜잭션을 관찰합니다.
3. 공격자는 동일한 볼트 ID를 사용하지만 더 높은 가스비로 사용되지 않은 스테이킹 풀 예치금을 사용하는 자신의 트랜잭션을 신속하게 제출합니다.
4. 공격자의 트랜잭션이 먼저 처리되어 `globalState.groupDepositIndex`가 업데이트됩니다.
5. 우선순위 풀 트랜잭션이 처리되면 `globalState.groupDepositIndex`가 더 이상 예상 값과 일치하지 않으므로 `InvalidVaultIds` 오류로 인해 실패합니다.

**파급력:** 특정 시나리오에서 우선순위 풀은 사용자 자금을 예치하지 못하여 예치 기능에 대한 서비스 거부 조건이 효과적으로 생성될 수 있습니다. 공격자가 공격을 실행하기 위해 자신의 자금을 사용할 필요는 없지만 다른 예금자를 괴롭히기 위해 사용되지 않은 스테이킹 풀 예치금(사용 가능한 경우)을 사용한다는 점이 주목할 만합니다.

**개념 증명 (PoC):**
```typescript
it('front-running deposit to change deposit index', async () => {
    const {
      token,
      accounts,
      signers,
      adrs,
      strategy,
      priorityPool,
      stakingPool,
      stakingController,
      rewardsController,
      vaults,
    } = await loadFixture(deployFixture)

    await token.transfer(await stakingPool.getAddress(), toEther(100)) //depositing to represent unused liquidity in staking pool
    //@audit front-run priority pool deposit
    await stakingPool.connect(signers[2]).depositLiquidity([encodeVaults([0, 1, 2, 3, 4])]) // anyone can deposit this

    //@audit actual deposit is DOSed as the group deposit index has changed
    await expect(
      priorityPool.connect(signers[2]).deposit(toEther(100), false, [encodeVaults([0, 1, 3, 4])])
    ).to.be.revertedWithCustomError(strategy, 'InvalidVaultIds()')
  })
```

**권장 완화 조치:** 다음 대안 중 하나를 고려하십시오:
- 승인되지 않은 액세스를 방지하기 위해 `StakingPool::depositLiquidity` 함수 제어
- 순차적 볼트 ID 또는 전역 예치 인덱스에 의존하지 않도록 예치 메커니즘을 재설계하십시오. 대신 주문 조작에 강한 예치 관리 방법을 사용하십시오.

**Stake.link**
[27b3008](https://github.com/stakedotlink/contracts/pull/108/commits/27b300875914134238f1d556fbb41244ba931625)에서 수정되었습니다.

**Cyfrin**
확인됨. `depositLiquidity`는 이제 비공개 함수입니다.


### 운영자가 Chainlink에 의해 제거되면 원금을 회수할 방법이 없음

**설명:** Chainlink는 [OperatorStakingPool](https://etherscan.io/address/0xa1d76a7ca72128541e9fcacafbda3a92ef94fdc5#code)에서 운영자를 제거할 수 있습니다. 이렇게 하면 원금이 제거되어 운영자가 더 이상 보상을 적립하지 못하게 됩니다. 그러나 원금은 손실되지 않으며 `OperatorStakingPool::unstakeRemovedPrincipal`을 호출하여 계속 사용할 수 있습니다.

`OperatorVault`에는 이와 같은 호출이 없습니다. OperatorVault가 Chainlink 스테이킹 풀에서 운영자로 제거되면 풀 원금은 잠기게 됩니다.

**파급력:** 자금은 결국 볼트로 업그레이드하여 회수할 수 있지만 긴 과정이며 그때까지 제거된 원금이 볼트 원금에 포함되므로 볼트 동작이 불완전할 것입니다:
```solidity
114:    function getPrincipalDeposits() public view override returns (uint256) {
115:        return
116:            super.getPrincipalDeposits() +
117:            IOperatorStaking(address(stakeController)).getRemovedPrincipal(address(this));
118:    }
```
따라서 볼트는 원금이 있는 것처럼 보이지만 인출할 수 없습니다.

**권장 완화 조치:** 운영자 또는 소유자가 `unstakeRemovedPrincipal`을 수행할 수 있는 호출을 추가하는 것을 고려하십시오.

**Stake.link**
[b19b58c](https://github.com/stakedotlink/contracts/pull/108/commits/b19b58c9dc97ee2caba4bf91f74271af5ea9a793)에서 수정되었습니다.

**Cyfrin**
확인됨. 제거된 원금 언스테이킹에 대한 지원이 추가되었습니다.


### `Vault::unbondingActive`가 Chainlink 및 `FundFlowController` 본딩과 동기화되지 않을 수 있음

**설명:** 각 볼트에는 현재 비결합(unbonding) 기간인지 쿼리하는 호출 `Vault::unbondingActive`가 있습니다:
```solidity
    function unbondingActive() external view returns (bool) {
        return block.timestamp < stakeController.getClaimPeriodEndsAt(address(this));
    }
```
여기서 비교는 `<`입니다.

그러나 Chainlink [볼트](https://etherscan.io/address/0xbc10f2e862ed4502144c7d632a3459f49dfcdb5e)에서:
```solidity
  function _inClaimPeriod(Staker storage staker) private view returns (bool) {
    if (staker.unbondingPeriodEndsAt == 0 || block.timestamp < staker.unbondingPeriodEndsAt) {
      return false;
    }

    return block.timestamp <= staker.claimPeriodEndsAt;
  }

```

그리고 `FundFlowController::claimPeriodActive`에서 비교는 `<=`입니다:
```solidity
    function claimPeriodActive() external view returns (bool) {
        uint256 claimPeriodStart = timeOfLastUpdateByGroup[curUnbondedVaultGroup] + unbondingPeriod;
        uint256 claimPeriodEnd = claimPeriodStart + claimPeriod;

        return block.timestamp >= claimPeriodStart && block.timestamp <= claimPeriodEnd;
    }
```

따라서 `block.timestamp == staker.claimPeriodEndsAt`일 때 `unbondingActive` 호출은 잘못된 답을 제공합니다.

**파급력:** `Vault::unbondingActive`는 `FundFlowController`에서 유지 관리할 총 비결합 및 인출 주문을 결정하는 데 사용됩니다. 그리고 `VaultControllerStrategy::withdraw`에서도 사용됩니다:
```solidity
    function withdraw(uint256 _amount, bytes calldata _data) external onlyStakingPool {
        // @audit claimPeriodActive() does `<= claimPeriodEnd`
        if (!fundFlowController.claimPeriodActive() || _amount > totalUnbonded)
            revert InsufficientTokensUnbonded();

        // ...

        for (uint256 i = 0; i < vaultIds.length; ++i) {
            // ...

            // @audit unbondingActive() does `< claimPeriodEnd`
            if (deposits != 0 && vault.unbondingActive()) {
                if (toWithdraw > deposits) {
                    vault.withdraw(deposits);
                    unbondedRemaining -= deposits;
                    toWithdraw -= deposits;
                } else if (deposits - toWithdraw > 0 && deposits - toWithdraw < minDeposits) {
                    vault.withdraw(deposits);
                    unbondedRemaining -= deposits;
                    break;
                } else {
                    vault.withdraw(toWithdraw);
                    unbondedRemaining -= toWithdraw;
                    break;
                }
            }
        }
```
따라서 `block.timestamp == staker.claimPeriodEndsAt`일 때 인출이 잘못되어 아무것도 인출되지 않습니다.

이것은 `VaultControllerStrategy::_depositToVaults`에도 적용됩니다:
```solidity
                if (vault.unbondingActive()) {
                    totalRebonded += deposits;
                }
```
여기서 `totalRebonded`, 그리고 결과적으로 `totalUnbonded`가 틀릴 것입니다. 그러나 볼트가 청구 기간을 벗어나려고 하고 따라서 `FundFlowController::performUpkeep`이 호출되어 `totalUnbonded`가 재설정될 때까지 무효이므로 영향은 훨씬 적습니다.

**개념 증명 (PoC):** `vault-controller-strategy.test.ts`에 추가할 수 있는 테스트:
```javascript
  it('should perform withdrawal at claim period end' , async () => {
    const {accounts, adrs, strategy, token, stakingController, vaults, fundFlowController, updateVaultGroups } =
      await loadFixture(deployFixture)

    // Deposit into vaults
    await strategy.deposit(toEther(500), encodeVaults([]))
    assert.equal(fromEther(await token.balanceOf(adrs.stakingController)), 500)
    for (let i = 0; i < 5; i++) {
      assert.equal(fromEther(await stakingController.getStakerPrincipal(vaults[i])), 100)
    }
    assert.equal(Number((await strategy.globalVaultState())[3]), 5)

    await updateVaultGroups([0, 5, 10], 0)
    await time.increase(claimPeriod)
    await updateVaultGroups([1, 6, 11], 0)
    await time.increase(claimPeriod)
    await updateVaultGroups([2, 7], 0)
    await time.increase(claimPeriod)
    await updateVaultGroups([3, 8], 0)
    await time.increase(claimPeriod)
    await updateVaultGroups([4, 9], 100)

    const claimPeriodEnd = Number(await fundFlowController.timeOfLastUpdateByGroup(0)) + unbondingPeriod + claimPeriod

    const balanceBefore = await token.balanceOf(accounts[0])

    // set to claim period end
    await time.setNextBlockTimestamp(claimPeriodEnd)
    await strategy.withdraw(toEther(50), encodeVaults([0, 5]))


    const balanceAfter = await token.balanceOf(accounts[0])

    // nothing was withdrawn
    assert.equal(balanceAfter - balanceBefore, 0n)
  })
```

**권장 완화 조치:** `<`를 `<=`로 변경하는 것을 고려하십시오:
```diff
    function unbondingActive() external view returns (bool) {
-       return block.timestamp < stakeController.getClaimPeriodEndsAt(address(this));
+       return block.timestamp <= stakeController.getClaimPeriodEndsAt(address(this));
    }
```

**Stake.link**
[b19b58c](https://github.com/stakedotlink/contracts/pull/108/commits/b19b58c9dc97ee2caba4bf91f74271af5ea9a793)에서 수정되었습니다.

**Cyfrin**
확인됨. 청구 기간 시작이 올바르게 계산됩니다.


### `FundFlowController::performUpkeep`의 잠재적인 오래된 데이터로 인해 잘못된 상태 업데이트가 발생할 수 있음

**설명:** `FundFlowController::checkUpkeep` 함수는 볼트를 반복하고 `unbondingActive` 상태를 확인하여 `nextGroupOpVaultsTotalUnbonded` 및 `nextGroupComVaultsTotalUnbonded`를 계산합니다. 이 데이터는 인코딩되어 키퍼(keeper)에 의해 `performUpkeep`으로 전달됩니다.

그러나 `checkUpkeep`과 `performUpkeep` 실행 사이에 잠재적인 시간 지연이 있어 오래된 데이터가 시스템 상태 업데이트에 사용될 수 있습니다. 볼트의 `unbondingActive` 상태는 현재 블록 타임스탬프와 각 볼트의 청구 기간 종료 시간을 기반으로 하므로 시간에 민감합니다. 네트워크 혼잡이나 기타 요인으로 인해 `performUpkeep` 트랜잭션이 지연되면 업데이트에 사용되는 `totalUnbonded` 값이 더 이상 볼트의 현재 상태를 정확하게 반영하지 않을 수 있습니다.

`FundFlowController.sol`
```solidity
function checkUpkeep(bytes calldata) external view returns (bool, bytes memory) {
    // ... (other code)
    (
        uint256[] memory curGroupOpVaultsToUnbond,
        uint256 nextGroupOpVaultsTotalUnbonded
    ) = _getVaultUpdateData(operatorVCS, nextUnbondedVaultGroup);
    // ... (similar for community vaults)
    return (
        true,
        abi.encode(
            curGroupOpVaultsToUnbond,
            nextGroupOpVaultsTotalUnbonded,
            curGroupComVaultsToUnbond,
            nextGroupComVaultsTotalUnbonded
        )
    );
}

function performUpkeep(bytes calldata _data) external {
    // ... (decoding and using potentially stale data)
}
```

**파급력:** performUpkeep에서 오래된 데이터를 사용하면 잘못된 상태 업데이트가 발생할 수 있습니다. 시스템은 의도한 것보다 더 많거나 적은 언본딩(unbonding)을 허용하여 기록된 상태와 볼트의 실제 상태 간에 불일치가 발생할 수 있습니다. 최악의 시나리오에서는 언본딩되어야 할 자금이 잠겨 있거나 여전히 잠겨 있어야 할 자금이 조기에 해제될 수 있습니다.

**권장 완화 조치:** `performUpkeep`이 checkUpkeep의 잠재적으로 오래된 데이터에 의존하는 대신 실행 시점에 `totalUnbonded` 값을 다시 계산하도록 수정하는 것을 고려하십시오.

**Stake.link**
[53d6090](https://github.com/stakedotlink/contracts/pull/108/commits/53d60901c171f4b436dcb4b22fe6549ba5dcef05)에서 수정되었습니다.

**Cyfrin**
확인됨. 권장된 대로 수정되었습니다.


### 보상 배수 조작에 취약한 스테이킹 보상

**설명:** Chainlink Operator 및 Community 스테이킹 풀의 스테이킹의 경우 스테이커는 Chainlink [RewardVault](https://etherscan.io/address/0x996913c8c08472f584ab8834e925b06D0eb1D813)를 통해 보상을 받습니다. 여기서 보상은 스테이킹된 시간당 발생합니다.

그러나 표준 보상 설계와 비교하여 보상 작동 방식에 차이가 있습니다. 장기간 포지션을 유지하도록 장려하기 위해 Chainlink는 `배수(multiplier)`를 사용합니다. 이것은 90일(작성 당시) 동안 0에서 1로 진행됩니다. 스테이킹을 시작하면 0의 배수를 가지며 90일 동안 선형적으로 1로 증가합니다.

이 기간 동안 언스테이킹(unstake)하면 배수가 재설정되고 획득했어야 할 몰수된 보상은 다른 스테이커에게 분배됩니다.

이것은 LinkPool 스테이커를 괴롭혀(grief) 보상을 줄이고 LinkPool 외부의 스테이커에게 보상을 제공하는 데 사용될 수 있습니다.

사용자는 전체 볼트(+ 1 LINK)를 예치하고 전체 볼트를 인출하는 과정을 반복하여 전체 현재 볼트 그룹을 반복할 수 있습니다. 따라서 모든 볼트의 배수를 재설정합니다.

**파급력:** 커뮤니티 풀의 현재 수치를 사용한 영향에 대한 간략한 개요:

커뮤니티 스테이커의 현재 방출 속도는 `~0.05 LINK/s`입니다. 커뮤니티 볼트는 현재 총 `40875000 LINK`가 스테이킹되어 가득 찼습니다. 이는 토큰당 초당 이득이 `0.05 / 40875000 =~ 1.2e-09`임을 의미합니다.

LinkPool은 풀의 ~3%인 총 `1324452 LINK` 스테이크를 가지고 있습니다.

위에서 언급했듯이 배수 기간은 `90일`이고 언본딩 기간은 `4주`로, 배수를 재설정할 수 있는 가장 빠른 기간입니다.

이것은 다음과 같은 보상 계산을 제공합니다:
```
perTokenPerS*(staked*0.2)*4 weeks =~ 783 LINK
```
총 몰수 금액을 얻으려면 `1-(staked time/multiplier period)`를 곱해야 합니다:
```
perTokenPerS*(staked*0.2)*4 weeks*(1 - (4 weeks/90 days)) =~ 540 LINK
```
따라서 이 경우 총 540 LINK가 손실됩니다. 이것은 볼트가 0 배수에서 시작할 때만 적용됩니다. 볼트가 이전에 `>90일`에 있었다면 몰수가 없으므로 볼트 그룹에 대해 두 번째로 수행할 때만 이득을 얻을 수 있습니다. 그때만 배수가 완전히 재설정되기 때문입니다.

이것의 수익성을 확인하기 위해 획득한 `540 LINK`가 풀로 나뉩니다:
```
540/40875000 = 1.3e-5 per held link
```
사용자가 100개의 볼트(각각 최대 예치 15000 LINK)를 가지고 있다면 총 19 LINK를 얻게 되며, 이는 아마도 가스 비용이 들 것입니다.

따라서 공격 벡터는 매우 작습니다. 그러나 볼트의 배수가 불필요하게 재설정되어 괴롭힘이나 의도하지 않은 보상 손실이 발생할 가능성이 여전히 있습니다.

이것은 현재 커뮤니티 스테이킹 풀 분포로 취한 것입니다. LinkPool 지분이 클수록 공격의 수익성이 커집니다.

**권장 완화 조치:** 모든 볼트 배수를 재설정하는 공격 벡터의 경우 예치 + 인출로 볼트를 쉽게 반복할 수 있어야 합니다. 이것은 작은 인출 시간 잠금(timelock)을 구현하여 완화할 수 있습니다.

<90일 동안 스테이킹한 볼트에서 인출할 경우 발생할 수 있는 의도하지 않은 보상 손실의 경우 `withdrawalIndex`의 회전을 강제하는 것을 고려하십시오. 지금은 무엇이든 선택할 수 있지만 항상 증가하면(다시 루프될 때까지) 90일 이상 볼트를 "혼자" 유지할 수 있기를 바랍니다.

**Stake.link**
[9dcac24](https://github.com/stakedotlink/contracts/pull/108/commits/9dcac2435db6d3a1e7d0d222e91b6ffadf99484d)에서 수정되었습니다.

**Cyfrin**
확인됨. 즉각적인 스테이킹 풀 인출이 허용되지 않습니다.

\clearpage
## 저위험 (Low Risk)


### `PriorityPool::depositQueuedTokens`의 유효성 검사 누락으로 대기 중인 예치가 일시적으로 지연될 수 있음

**설명:** `PriorityPool::depositQueuedTokens`에는 `_queueDepositMin < _queueDepositMax`를 확인하는 유효성 검사가 없습니다. 이 간과로 인해 사용자가 `max(strategyDepositRoom, canDeposit)`로 `_queueDepositMin`을 호출하고 `_queueDepositMax=0`으로 `depositQueuedTokens`를 호출할 수 있는 그리핑 공격 벡터가 허용됩니다.

이것은 스테이킹 풀이 항상 사용되지 않은 예치금을 전략에 예치하고 우선순위 풀의 대기 금액을 우회하도록 강제합니다. 총 예치 가능 금액, 즉 총 대기 금액과 사용되지 않은 예치금이 `queuedDepositMin`을 초과하지만 총 대기 금액만 `queuedDepositMin` 미만인 경우, 위의 조작은 대기 예치가 불필요하게 지연됨을 의미합니다.

**파급력:** 대기 중인 금액이 가능한 한 빨리 전략에 예치되기를 원하는 사용자에 대한 괴롭힘(Griefing).

**권장 완화 조치:** `depositQueuedTokens` 함수에 `_queueDepositMin < _queueDepositMax` 확인을 추가하는 것을 고려하십시오.

**Stake.link**
인지함.

**Cyfrin**
인지함.


### Chainlink가 언본딩 및 청구 기간을 변경할 때 FundflowController가 동기화되지 않음

**설명:** `FundFlowController`에는 `unbondingPeriod` 및 `claimPeriod`라는 두 가지 필드가 있으며 Chainlink 계약의 해당 상태와 동기화되어야 합니다:

```solidity
    uint64 public unbondingPeriod;
    uint64 public claimPeriod;
```

그러나 Chainlink는 `StakingPoolBase::setUnbondingPeriod` 및 `StakingPoolBase::setClaimPeriod`를 사용하여 이를 변경할 수 있습니다. `FundFlowController`에는 상태를 업데이트할 호출이 없으므로 이러한 변경 사항은 반영되지 않습니다.

또한 `performUpkeep`의 설계는 `FundFlowController`의 기간이 변경되면 잘 작동하지 않습니다:
```solidity
        if (
            timeOfLastUpdateByGroup[nextUnbondedVaultGroup] != 0 &&
            block.timestamp <=
            timeOfLastUpdateByGroup[curUnbondedVaultGroup] + unbondingPeriod + claimPeriod // @audit new periods used
        ) revert NoUpdateNeeded();

        if (
            curUnbondedVaultGroup != 0 &&
            timeOfLastUpdateByGroup[curUnbondedVaultGroup] == 0 &&
            block.timestamp <= timeOfLastUpdateByGroup[curUnbondedVaultGroup - 1] + claimPeriod // @audit new period used
        ) revert NoUpdateNeeded();

        if (block.timestamp < timeOfLastUpdateByGroup[nextUnbondedVaultGroup] + unbondingPeriod) // @audit new period used
            revert NoUpdateNeeded();

...
        // @audit time of unbonding stored, not the resulting unbonding timestamp and claim timestamp
        timeOfLastUpdateByGroup[curUnbondedVaultGroup] = uint64(block.timestamp);
```
여기서 언본딩 시간이 저장되고 언본딩 기간의 현재 상태가 사용됩니다. Chainlink 계약은 언본딩할 때 지연을 적용합니다:
```solidity
    staker.unbondingPeriodEndsAt = (block.timestamp + s_pool.configs.unbondingPeriod).toUint128();
    staker.claimPeriodEndsAt = staker.unbondingPeriodEndsAt + s_pool.configs.claimPeriod;
```
따라서 변경 시점에 `FundFlowController`가 동기화되지 않습니다.

**파급력:** Chainlink에서 `unbonding` 및 `claim` 기간이 변경되면 `FundFlowController`에 유지되는 상태가 오래된 상태가 됩니다.

`FundFlowController`가 업그레이드되거나 새 것이 배포되더라도 볼트 그룹 언본딩 및 청구 기간의 상태는 모든 기간이 만료될 때까지 사실상 오래된 상태가 됩니다.

이로 인해 프로토콜의 인출에 상당한 혼란이 발생하여 사용자 자금이 필요 이상으로 오래 잠기게 됩니다.

**권장 완화 조치:** 사용자가 Chainlink에서 언본딩할 때 Chainlink는 언본딩 타임스탬프와 청구 타임스탬프를 모두 저장합니다. LinkPool도 동일하게 수행하는 것이 좋습니다. 언본딩 시 Chainlink에 저장된 대로 언본딩 및 청구 타임스탬프를 반환하는 것을 고려하십시오.

그런 다음 `VaultControllerStrategy`는 이것을(모두 _동일해야_ 하므로 마지막 것만 반환할 수 있음) `FundFlowController`로 전파하여 동일한 언본딩 및 청구 타임스탬프를 저장할 수 있습니다. 그런 다음 회전 로직에 사용하십시오.

**Stake.link**
인지함.

**Cyfrin**
인지함.


### Chainlink는 운영자 및 커뮤니티 스테이킹 풀에 대해 다른 본딩 및 청구 기간을 가질 수 있음

**설명:** `FundFlowController`는 운영자 및 커뮤니티 풀 모두에 대해 동일한 `unbondingPeriod` 및 `claimPeriod`를 사용합니다.

Chainlink [Operator](https://etherscan.io/address/0xa1d76a7ca72128541e9fcacafbda3a92ef94fdc5) 및 [Community](https://etherscan.io/address/0xbc10f2e862ed4502144c7d632a3459f49dfcdb5e) 풀은 서로 다른 계약이 배포되므로 Chainlink는 언본딩 및 청구 기간을 변경할 수 있으므로 항상 동일하다는 보장은 없습니다.

**파급력:** Chainlink가 운영자 및 커뮤니티 볼트에서 서로 다른 언본딩 및 청구 기간을 설정하면 LinkPool에서 인출하는 능력에 영향을 미칩니다.

`FundFlowController`의 상태는 그중 하나만 정확하게 추적할 수 있기 때문입니다. 청구 기간이 여전히 활성 상태일 때 Chainlink 볼트에서 revert가 발생하므로 `checkUpkeep` 실행에 revert 및 합병증이 발생할 수도 있습니다. `checkUpkeep`의 컨트롤이 더 이상 최신 상태가 아니므로 이는 필연적으로 가끔 발생합니다.

**권장 완화 조치:** 커뮤니티 및 운영자 풀 모두에 대해 하나의 `FundFlowController`를 갖는 대신 각각 하나씩 갖는 것을 고려하십시오. 그러면 다른 기간을 개별적으로 추적할 수 있는 유연성이 제공됩니다.

**Stake.link**
인지함.

**Cyfrin**
인지함.


### 사용자가 풀에서 인출하여 운영자 슬래싱/보상 업데이트를 선행 실행(frontrun)할 수 있음

**설명:** 운영자가 슬래싱되면 스테이킹된 원금의 일부가 제거되어 슬래셔에게 전송됩니다. 이것은 Chainlink [`PriceFeedAlertsController`](https://etherscan.io/address/0x27484ba119d12649be2a9854e4d3b44cc3fdbad7)에 구성된 모든 슬래시 가능한 운영자에게 발생합니다. 이 슬래싱은 스테이킹된 LINK의 양을 줄여 `stLINK` 토큰의 가치를 낮춥니다.

참고로 기술적으로 슬래싱이 발생할 때 가치(LINK의 환율)가 변경되는 것이 아니라 `totalStaked`가 업데이트될 때 변경되며, 이는 해당 전략에 대한 `StakingPool::updateStrategyRewards` 호출 시 수행됩니다.

따라서 대규모 `stLINK` 보유자는 슬래싱 이벤트를 본 다음 `updateStrategyRewards` 호출을 인출로 선행 실행하여 여전히 보유하고 있는 사용자보다 더 높은 `LINK` 비율을 얻을 수 있습니다.

**파급력:** [`StakingVault`](https://etherscan.io/address/0xb8b295df2cd735b15be5eb419517aa626fc43cd5) 및 [`OperatorStakingPool`](https://etherscan.io/address/0xa1d76a7ca72128541e9fcacafbda3a92ef94fdc5)의 수치를 사용하면:
```
staked=2438937.970114142763711705
shares=2193487.121097554919462043

staked/shares = 1.1118998359533434
```
[`PriceFeedAlertsController`](https://etherscan.io/address/0x27484ba119d12649be2a9854e4d3b44cc3fdbad7)에 현재 슬래시 가능한 9개의 볼트가 있다는 점을 감안하면 다음과 같은 새로운 fx 비율이 제공됩니다:
```
slashAmount=700

(staked-9*slashedAmout)/shares=1.109027696910718
```
이는 주당 `0.002` 링크의 차이입니다. 대략 하나의 완전한 볼트인 `~15000 stLINK`의 포지션을 취하면 `~40 LINK`($400)를 절약할 수 있습니다. 이는 가스 비용보다 많지만 엄청난 금액은 아닙니다.

**개념 증명 (PoC):**
```solidity
it('user can withdrawal before updateStrategyRewards instant', async () => {
    const { adrs, vaults, signers, token, pp, stakingController, stakingPool, updateVaultGroup}
      = await loadFixture(deployFixture)

    const alice = signers[2]

    await pp.deposit(toEther(475), false, [encodeVaults([])])

    await token.transfer(alice, toEther(25))
    await pp.connect(alice).deposit(toEther(100), false, [encodeVaults([])])

    assert.equal(vaults.length, 5)
    for(let i = 0; i < 5; i++) {
      assert.equal(fromEther(await stakingController.getStakerPrincipal(vaults[i])), 100)
    }

    // unbond all groups
    for(let i = 0; i < 5; i++) {
      await updateVaultGroup(i)

      // don't increase after last iteration to keep the current group in claim period
      if(i < 4) {
        await time.increase(claimPeriod + 1)
      }
    }

    // slashing happens
    await stakingController.slashStaker(vaults[0], toEther(50))
    assert.equal(fromEther(await stakingController.getStakerPrincipal(vaults[0])),50)


    const balanceBefore = await token.balanceOf(alice.address)
    await stakingPool.connect(alice).approve(adrs.pp, toEther(25))
    await pp.connect(alice).withdraw(
      toEther(25),
      toEther(25),
      toEther(25),
      [ethers.ZeroHash],
      false,
      false,
      [encodeVaults([0])]
    )
    const balanceAfter = await token.balanceOf(alice.address)
    assert.equal(balanceAfter - balanceBefore, toEther(25))

    // -1000 because of initial burnt amount
    assert.equal(Number(await stakingPool.balanceOf(signers[0].address)), Number(toEther(475))-1000)

    // when strategy reward are updated it will reduce the values of the shares held by the rest of the stakers
    await stakingPool.updateStrategyRewards([0],'0x')

    assert.equal(Number(await stakingPool.balanceOf(signers[0].address)), Number(toEther(425))-1000)
  })

```

**권장 완화 조치:** 인출에 일종의 지연을 구현하는 것을 고려하십시오. 한 번의 tx에서 수행할 수 없도록 Chainlink만큼 길 필요는 없습니다.

**Stake.link**
인지함. 이 문제는 `RebaseController`가 배포될 때 대부분 완화되어야 합니다.

**Cyfrin**
인지함.


### Chainlink 풀이 일시 중지된 경우 인출이 필요 이상으로 오래 걸림

**설명:** chainlink 문서는 다음 상황에서 인출이 발생할 수 있다고 설명합니다:
```solidity
  /// @dev precondition The caller must be staked in the pool.
  /// @dev precondition The caller must be in the claim period or the pool must be closed or paused.
```
여기서 그들은 풀이 닫히거나 일시 중지될 때 항상 인출이 발생하도록 허용합니다.

`FundFlowController`에는 이것이 반영되지 않았으므로 인출이 발생하려면 풀이 올바른 볼트 그룹에 있어야 합니다. 따라서 일시 중지 후 볼트를 "비우는" 데 최소 4*청구 기간이 소요됩니다.

**파급력:** chainlink 풀이 닫히면 LinkPool이 보유한 포지션을 청산하는 데 필요한 시간보다 최대 4배 더 오래 걸립니다.

**권장 완화 조치:** 스테이킹 컨트롤러가 일시 중지된 경우 현재 볼트 그룹 및 청구 기간 외부의 볼트에서 인출을 허용하는 것을 고려하십시오.

**Stake.link**
인지함.

**Cyfrin**
인지함.

\clearpage
## 가스 최적화 (Gas Optimization)


### `FundFlowController::_getVaultDepositOrder`의 가스 최적화

**설명:** `FundFlowController::_getVaultDepositOrder`에서 첫 번째 볼트는 아래에서 볼 수 있듯이 항상 `groupDepositIndex`에 해당하는 볼트입니다.

```solidity
      if (groupDepositIndex < maxVaultIndex) {
            vaultDepositOrder[0] = groupDepositIndex;
            ++totalVaultsAdded;
            (uint256 withdrawalIndex, ) = _vcs.vaultGroups(groupDepositIndex % numVaultGroups);
            uint256 deposits = IVault(vaults[groupDepositIndex]).getPrincipalDeposits();
            if (
                deposits != maxDeposits && (groupDepositIndex != withdrawalIndex || deposits == 0)
            ) {
                totalDepositsAdded += maxDeposits - deposits;//@audit missing check to see if _toDeposits is already accomodated
            }
        }
```
`toDeposit`이 이 볼트의 예치 공간만으로 채워질 수 있을 가능성이 있습니다. 이 경우 `totalDepositsAdded >= _toDeposit`인지 확인하고 볼트 순서에 단일 볼트 ID만 사용하여 종료하는 것이 현명합니다.

또한 이 경우 값비싼 `_sortIndexesDescending`은 두 번째 볼트부터만 적용되므로 필요하지 않습니다.

**권장 완화 조치:** 다음을 고려하십시오:

1. 볼트 0 바로 뒤에 `totalDepositsAdded >= _toDeposit` 확인을 추가하고 조건이 충족되면 단일 볼트만 반환합니다.
2. `_sortIndexesDescending`을 볼트 0을 포함한 후로 이동합니다.

**Stake.link**
인지함.

**Cyfrin**
인지함.


### `FundFlowController::performUpkeep`에서 스토리지 읽기 캐시

**설명:** `FundFlowController::performUpkeep`에는 `curUnbondedVaultGroup`, `unbondingPeriod`, `claimPeriod`, `timeOfLastUpdateByGroup[curUnbondedVaultGroup]` 및 `timeOfLastUpdateByGroup[nextUnbondedVaultGroup]`과 같은 스토리지 변수에 대한 반복적인 읽기가 많이 포함되어 있습니다.

**권장 완화 조치:** 불필요한 스토리지 읽기를 절약하기 위해 이들을 캐시하는 것을 고려하십시오.

**Stake.link**
인지함.

**Cyfrin**
인지함.


### 비트와이즈 연산을 사용하여 VaultGroup 및 Fee 구조체 최적화

**설명:** `VaultControllerStrategy` 계약의 현재 `VaultGroup` 및 `Fee` 구조체는 다음과 같이 정의됩니다:

```solidity
struct VaultGroup {
    uint64 withdrawalIndex;
    uint128 totalDepositRoom;
}

struct Fee {
    address receiver;
    uint256 basisPoints;
}
```
이러한 구조체는 단일 uint192로 압축하고 액세스 및 수정에 비트와이즈 연산을 사용하여 더 최적화할 수 있습니다. 이는 이러한 구조체가 배열에서 사용되어 잠재적으로 상당한 가스 절감으로 이어질 수 있으므로 특히 유익합니다. 최대 basisPoints는 10000(100%)이므로 32비트 슬롯에 맞을 수 있습니다.

**권장 완화 조치:** `VaultGroup` 구조체를 다음과 같이 단일 uint192(withdrawalIndex의 경우 64비트, totalDepositRoom의 경우 128비트)로 압축하는 것을 고려하십시오:

```solidity
uint192[] public vaultGroups;

function getVaultGroup(uint256 index) public view returns (uint64 withdrawalIndex, uint128 totalDepositRoom) {
    uint192 group = vaultGroups[index];
    withdrawalIndex = uint64(group);
    totalDepositRoom = uint128(group >> 64);
}

function setVaultGroup(uint256 index, uint64 withdrawalIndex, uint128 totalDepositRoom) internal {
    vaultGroups[index] = uint192(withdrawalIndex) | (uint192(totalDepositRoom) << 64);
}
```
`Fee` 구조체를 다음과 같이 단일 uint192(주소의 경우 160비트, basisPoints의 경우 32비트)로 압축하는 것을 고려하십시오:

```solidity
uint192[] public fees;

function getFee(uint256 index) public view returns (address receiver, uint32 basisPoints) {
    uint256 fee = fees[index];
    receiver = address(uint160(fee));
    basisPoints = uint32(fee >> 160);
}

function setFee(uint256 index, address receiver, uint32 basisPoints) internal {
    require(basisPoints <= 10000, "Basis points cannot exceed 10000");
    fees[index] = uint192(uint160(receiver)) | (uint192(basisPoints) << 160);
}
```

**Stake.link**
인지함.

**Cyfrin**
인지함.

\clearpage
