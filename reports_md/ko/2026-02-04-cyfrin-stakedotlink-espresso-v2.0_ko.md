**Lead Auditors**

[Kage](https://x.com/0kage_eth)

[Al Qa-qa](https://x.com/al_qa_qa)

**Assisting Auditors**



---

# 발견 사항 (Findings)
## 중간 위험 (Medium Risk)


### 처리되기 전의 출금 대기 토큰이 다시 예치될 수 있음

**설명:** `EspressoFundFlowController::shouldDepositQueuedTokens`는 토큰을 예치할지 판단할 때 보류 중인 출금 요청을 고려하지 않습니다.

`StakeTableV2.exitEscrowPeriod`는 언본딩 후 사용자 자금이 잠기는 시간을 제어하며, 컨트랙트상 이 값은 2일에서 14일 사이가 될 수 있습니다. `WithdrawalPool.minTimeBetweenWithdrawals`는 대기 중인 출금이 실행되는 두 `performUpkeep` 호출 사이의 최소 간격입니다(현재 배포 스크립트에서는 3일로 설정됨).

`exitEscrowPeriod < minTimeBetweenWithdrawals`인 경우 다음과 같은 공백 구간이 생깁니다.
- 토큰은 에스크로를 완료하고 `EspressoStrategy::claimUnbond`를 통해 청구됩니다.
- 하지만 시간 제한 때문에 `WithdrawalPool::performUpkeep`는 아직 실행될 수 없습니다.
- 이 구간 동안 토큰은 `totalQueued`에 머무릅니다.
- `EspressoFundController::shouldDepositQueuedTokens`는 `true`를 반환해 `depositController`가 다시 예치하도록 신호를 보냅니다.

```solidity
//EspressoFundController.sol
function shouldDepositQueuedTokens() external view returns (bool, uint256) {
      uint256 queuedTokens = strategy.totalQueued();
      return (queuedTokens > 0, queuedTokens);  // No check for pending withdrawals
  }
```

`EspressoFundFlowController.sol::withdrawVaults`는 청구 직후 출금을 즉시 처리하려고 시도합니다.

```solidity
 // EspressoFundFlowController.sol:155-163
  function withdrawVaults(uint256[] calldata _vaultIds) external {
      strategy.claimUnbond(_vaultIds);  // Tokens → totalQueued

      (bool upkeepNeeded, ) = withdrawalPool.checkUpkeep("");

      if (upkeepNeeded) {
          withdrawalPool.performUpkeep("");  // @audit May not execute if minTime not passed
      }
  }

```

`minTimeBetweenWithdrawals`가 아직 지나지 않아 `WithdrawalPool::checkUpkeep`가 `false`를 반환하면, 토큰은 `totalQueued`에 남아 있게 되고 다시 예치될 수 있습니다.

**영향:** 출금을 위해 명시적으로 언본딩한 토큰이 다시 예치되어, 언본딩의 목적이 무력화됩니다.


**권장 완화 조치:** `EspressoFundFlowController::shouldDepositQueuedTokens`를 수정해 보류 중인 출금을 위해 토큰을 남겨 두는 방안을 고려하십시오.

```diff
 function shouldDepositQueuedTokens() external view returns (bool, uint256) {
      uint256 queuedTokens = strategy.totalQueued();
--       return (queuedTokens > 0, queuedTokens);
++      uint256 queuedWithdrawals = withdrawalPool.getTotalQueuedWithdrawals();

++      // Reserve tokens for pending withdrawals
++      if (queuedTokens <= queuedWithdrawals) {
++         return (false, 0);
++      }

++      // Only deposit excess beyond withdrawal needs
++      uint256 excessTokens = queuedTokens - queuedWithdrawals;
++      return (excessTokens > 0, excessTokens);
  }
```

**Stake.Link:** [50768a6](https://github.com/stakedotlink/contracts/commit/50768a60b7abe52d123bf71126b58e968b829916)에서 수정됨.

**Cyfrin:** 확인함.


### 강제적인 보상 업데이트로 인해 수수료 관리가 서비스 거부 상태가 될 수 있음

**설명**

`EspressoStaking`에서 소유자는 수수료를 추가, 제거, 수정할 수 있습니다. 여기에는 수취인 주소와 연동된 수수료 비율 변경이 포함됩니다. 수수료가 갱신될 때마다 내부 함수 `_updateStrategyRewards`가 호출됩니다.

> contracts/espressoStaking/EspressoStrategy.sol#addFee/updateFee
```solidity
    function addFee(address _receiver, uint256 _feeBasisPoints) external onlyOwner {
        _updateStrategyRewards();
        fees.push(Fee(_receiver, _feeBasisPoints));
        if (_totalFeesBasisPoints() > 3000) revert FeesTooLarge();
        emit AddFee(_receiver, _feeBasisPoints);
    }

    function updateFee( ... ) external onlyOwner {
        _updateStrategyRewards();

        if (_feeBasisPoints == 0) { ... } else { ... }
    }

```

`_updateStrategyRewards` 함수는 `StakingPool.updateStrategyRewards`를 호출하고, 이는 다시 `EspressoStaking::updateDeposits`를 실행합니다. 누적된 수수료가 있으면 시스템은 이를 등록된 수수료 수취인에게 즉시 분배하려고 시도합니다.

수수료를 분배할 때 수취인이 컨트랙트이면, 시스템은 이를 `ERC677Receiver`로 취급하고 `onTokenTransfer` 훅을 호출합니다.

> contracts/core/StakingPool.sol
```solidity
    function _updateStrategyRewards(uint256[] memory _strategyIdxs, bytes memory _data) private {
        ...
        // distribute fees to receivers if there are any
        if (totalFeeAmounts > 0) {
            ...
            for (uint256 i = 0; i < receivers.length; i++) {
                for (uint256 j = 0; j < receivers[i].length; j++) {
                    if (feesPaidCount == totalFeeCount - 1) {
                        transferAndCallFrom(
                            address(this),
                            receivers[i][j],
                            balanceOf(address(this)),
                            "0x"
                        );
                    } else {
>>                      transferAndCallFrom(address(this), receivers[i][j], feeAmounts[i][j], "0x");
                        feesPaidCount++;
                    }
                }
            }
        }
        emit UpdateStrategyRewards(msg.sender, totalStaked, totalRewards, totalFeeAmounts);
    }

    function transferAndCallFrom( ... ) internal returns (bool) {
        _transfer(_sender, _to, _value);
        if (isContract(_to)) {
            contractFallback(_sender, _to, _value, _data);
        }
        return true;
    }

    function contractFallback( ... ) internal {
        IERC677Receiver receiver = IERC677Receiver(_to);
        receiver.onTokenTransfer(_sender, _value, _data);
    }

```

핵심 문제는 수수료를 변경하거나 제거할 때 누적된 수수료를 분배하는 업데이트가 **강제로** 수행된다는 점입니다. 만약 수수료 수취인이 악성 컨트랙트이거나(또는 EIP-7702를 통해 컨트랙트로 업그레이드된 손상된 EOA라면), `onTokenTransfer` 수신 시 revert하도록 만들어져 있으면 전체 트랜잭션이 실패합니다.

`addFee`와 `updateFee`는 모두 `_updateStrategyRewards`가 성공해야 하므로, 단 하나의 손상된 수취인만으로도 자기 자신을 제거하지 못하게 만들고 이후의 모든 수수료 설정 변경을 막을 수 있습니다. 이 상태는 `totalFeeAmounts`가 0일 때만 우회할 수 있는데, 이는 보통 "음수" 리베이스(슬래싱) 상황에서만 발생하는 드문 비정상 조건입니다.

0 주소를 수수료 수취인으로 사용하는 경우에도 동일한 문제가 발생한다는 점에 유의해야 합니다.

**영향**

- 하나의 수취인이 revert하면 전략 전체의 수수료가 묶일 수 있습니다.
- 소유자는 악성 수취인을 제거하거나 비율을 조정할 수 없습니다. 강제 분배가 항상 revert되기 때문입니다.
- 로직에 유연성이 없다면 전체 컨트랙트 업그레이드로만 완화할 수 있습니다.

**개념 증명 (Proof of Concept)**

1. `EspressoStaking`에는 여러 수수료 수취인이 있습니다.
2. 그중 하나는 `onTokenTransfer`가 호출되면 `revert()`하도록 작성된 컨트랙트(또는 공격자가 제어하는 계정)입니다.
3. 전략에 수수료가 누적됩니다.
4. 소유자가 악성 수취인을 인식하고 해당 basis points를 0으로 설정하는 `updateFee`를 호출해 제거하려고 시도합니다.
5. `updateFee`는 `_updateStrategyRewards`를 호출하며, 이 과정에서 누적된 수수료를 악성 수취인에게 전송하려고 시도합니다.
6. 악성 수취인이 revert하여 소유자의 트랜잭션이 실패하고, 악성 수취인은 시스템에 무기한 남게 됩니다.


**권장 완화 조치**
- 수수료 수취인을 추가/갱신할 때 0 주소 검증을 추가하는 방안을 고려하십시오.
- `_updateStrategyRewards` 호출을 **건너뛰는** 전용 수수료 수취인 제거 함수를 추가해, 분배가 실패하는 상황에서도 소유자가 악성 수취인을 갱신할 수 있게 하는 방안을 고려하십시오. 이는 `StakingPool`이 immutable인 경우에 적합합니다.

`StakingPool`을 변경할 수 있다면:
- 누적 수수료를 매핑에 저장하고 각 수취인이 개별적으로 청구하도록 `StakingPool`을 수정하는 방안을 고려하십시오(Pull 패턴). 이렇게 하면 한 수취인이 다른 수취인의 실행 흐름에 영향을 주지 못합니다.
- 또는 `contractFallback` 호출을 `try/catch`로 감싸고, 수취인이 revert할 경우 전체 글로벌 업데이트를 되돌리는 대신 해당 몫을 "failed transfers" 매핑에 저장해 나중에 수동 복구하도록 하는 방법도 고려할 수 있습니다.


**참고: 영향 범위**

> [!IMPORTANT]
> **이 취약점은 `EspressoStaking`에만 국한되지 않습니다. 문제는 코어 `StakingPool` 구현 자체에 존재합니다.**
> * **라이브 프로토콜:** 이 문제는 현재 운영 중인 **Link Staking**, **ETH Staking**, **Polygon Staking** 배포에도 직접 영향을 줍니다. 이 컨트랙트들이 라이브 상태이므로, 손상되었거나 악성인 수수료 수취인 하나만으로도 수수료 관리 기능이 즉시 잠길 수 있습니다.
> * **OperatorVCS 통합:** `OperatorVCS` 컨트랙트 역시 취약합니다. **vault 제거**나 **수수료 비율 조정** 같은 관리 작업은 모두 `updateStrategyRewards`를 트리거합니다. 따라서 하나의 revert하는 수수료 수취인만으로도 이러한 관리 작업이 사실상 DoS되어, 프로토콜이 vault를 제거하거나 경제 파라미터를 갱신하지 못하게 됩니다.
>

**Stake.Link:** 0 주소 검증은 [6cf8d4b](https://github.com/stakedotlink/contracts/commit/6cf8d4bdc16eabea13a7cdd7fe5bf63050205876)에서 수정됨. 수수료 수취인은 신뢰되는 주체이며, 신뢰된 수취인이 revert할 위험은 인지된 상태입니다.

**Cyfrin:** 확인함.

\clearpage
## 낮은 위험 (Low Risk)


### 비활성 vault에서 출금된 금액을 추적하지 않아 이미 언본딩된 자산을 다시 사용할 수 있음

**설명:** 언본딩은 누구나 처리할 수 있으며, `WithdrawalPool`에 대기 금액이 존재할 때 누구나 `EspressoFundFlowController::unbondVaults`를 호출해 출금에 필요한 자산을 언본딩할 수 있습니다.

`EspressoStrategy`에서 출금할 때는 모든 Vault를 순회합니다. Vault가 `inActive` 상태이면 `vault totalDeposits`를 사용하고, 그렇지 않으면 원금을 언본딩합니다.

> EspressoStrategy::unbond
```solidity
    function unbond(uint256 _toUnbond) external onlyFundFlowController {
        ...
        while (toUnbondRemaining != 0) {
            IEspressoVault vault = vaults[i];
            uint256 principalDeposits = vault.getPrincipalDeposits();

            if (!vault.isActive()) {
                uint256 deposits = vault.getTotalDeposits();
>>              toUnbondRemaining = toUnbondRemaining <= deposits
                    ? 0
                    : toUnbondRemaining - deposits;
            } else if (principalDeposits != 0) {
                ...
>>              vault.unbond(vaultToUnbond);

                toUnbondRemaining -= vaultToUnbond;
                ++numVaultsUnbonded;
            }

            i = (i + 1) % vaults.length;
            if (i == vaultWithdrawalIndex) break;
        }

        ...
    }
```

특정 `EspressoVault`에서 `unbond`를 호출하면 `EspressoStaking::undelegate`가 호출되고, 이는 위임량을 감소시킵니다(즉 `PrincipleDeposit`을 줄임). 그러나 vault가 `inActive` 상태일 경우에는 그 비활성 vault의 값을 단순히 차감하기만 할 뿐, 그 비활성 vault에서 사용한 값을 저장하거나 기록하지 않습니다. 따라서 다시 `unbond`를 호출해 이 vault에 도달하면 이미 언본딩된 자산을 또 언본딩하는 것처럼 계산될 수 있고, 그 결과 `WithdrawalPool` 큐를 채우기 위해 요청한 언본딩 자산을 실제로 인출하지 못하게 됩니다.

**영향:**
- 같은 `inActive` vault 자산을 두 번 언본딩해 비활성 vault 자산이 중복 계산됩니다.

**개념 증명 (Proof of Concept):** 다음 동작을 수행하는 스크립트를 작성했습니다.
- 100 Ether가 들어 있는 Vault 2개가 있습니다.
- 두 번째 vault는 `inActive` 상태입니다.
- 150 Ether를 언본딩했습니다.
- 새 Vault가 추가되고 여기에 100 Eth가 예치되었습니다.
- `inActive` vault를 확정하기 전에 다시 100 ETH 언본딩을 요청했습니다.
- `100 ETH`를 채우기 위해 새 vault에서 50 ETH를 가져와야 하지만(이전 `inActive` vault에서 50, 새 vault에서 50), 실제로는 비활성 vault에서 100 ETH를 언본딩한 것으로 처리되어 새 Vault 예치금은 그대로 남습니다.

다음 테스트 스크립트를 `test/espressoStaking/espresso-strategy.test.ts`에 추가하면 이 동작을 확인할 수 있습니다.
```ts
  it('Cyfrin Audit: unbond uncorrectly unbounds', async () => {
    const { signers, accounts, stakingPool, strategy, vaults, espressoStaking, validators } =
      await loadFixture(deployFixture)

    // Removing The Third Vault, We only Have two Vaults Now
    await strategy.removeVaults([2])


    // Test unbond counts inactive vault deposits toward unbond amount
    await stakingPool.deposit(accounts[0], toEther(200), ['0x'])
    await strategy.depositQueuedTokens([0, 1], [toEther(100), toEther(100)])

    // Exit validator 1
    await espressoStaking.exitValidator(validators[1])
    assert.equal(await vaults[1].isActive(), false)

    // Unbond 150: vault 0 has 100, vault 1 is inactive with 100 (counted but not unbonded)
    // So we need 150, vault 0 gives 100 (unbonded), vault 1 gives 100 (counted), done at 200 >= 150
    await strategy.unbond(toEther(150))

    assert.equal(fromEther(await vaults[0].getQueuedWithdrawals()), 100)
    assert.equal(fromEther(await vaults[1].getQueuedWithdrawals()), 0) // As it is inactive


    // Register a new validator
    const newValidator = accounts[8]
    await espressoStaking.registerValidator(newValidator)
    await strategy.addVault(newValidator)

    const vaultsAfterAdd = await strategy.getVaults()
    assert.equal(vaultsAfterAdd.length, 3)
    const newVaultAddress = vaultsAfterAdd[2]
    const newVault: EspressoVault = await ethers.getContractAt('EspressoVault', newVaultAddress)
    // --------

    // Stake 100 Ether to the new Vault
    await stakingPool.deposit(accounts[0], toEther(100), ['0x'])
    await strategy.depositQueuedTokens([2], [toEther(100)])
    assert.equal(fromEther(await newVault.getPrincipalDeposits()), 100)

    // Complete the unbond cycle
    await time.increase(exitEscrowPeriod)
    await strategy.claimUnbond([0])

    // Current State (Funds Left):
    // Vault 0: 0
    // Vault 1: 50 (It is Inactive and we unbounded 50 from it before)
    // newVault: 100 (We just deposited 100 into it)

    // Validation, the current Index is `v0` which has `0` amount
    assert.equal(Number(await strategy.vaultWithdrawalIndex()), 0)
    assert.equal(fromEther(await vaults[0].getPrincipalDeposits()), 0)
    assert.equal(fromEther(await newVault.getPrincipalDeposits()), 100)

    // Unbond 100: Vault 0 is empty so we skip it, vault 1 has 50 so we take 50 from it
    // and vault 2 has 100 so we take 50 from it
    await strategy.unbond(toEther(100))

    // Expected State After Unbond:
    // Vault 0: 0
    // Vault 1: 0 (We took 50 from it)
    // Vault 2: 50 (We took 50 from it)

    assert.equal(Number(await strategy.numVaultsUnbonding()), 0)
    assert.equal(fromEther(await vaults[0].getQueuedWithdrawals()), 0)
    // The Third Vault has 100 ether left in it instead of `50` as we double accumilate the `50` from inActive vault (v1)
    assert.equal(fromEther(await newVault.getQueuedWithdrawals()), 0)
    assert.equal(fromEther(await newVault.getPrincipalDeposits()), 100)

  })
```

**권장 완화 조치:** 이미 사용한 비활성 vault 자산을 다시 쓰지 않도록 `inActive` vault 잔액을 추적해야 합니다.

**Stake.Link:** 인지함.


### vault 제거 시 청구되지 않은 보상이 손실됨

**설명:** `EspressoStrategy::removeVaults`는 소유자가 전략에서 vault를 제거할 수 있게 합니다. 이 함수는 원금 예치금이 인출되었는지는 확인하지만, 제거 전에 보상이 청구되었는지는 확인하지 않습니다.

NatSpec 문서도 이를 명시하고 있습니다.

```solidity
  /**
   * @dev ... Will not check for unclaimed rewards so rewards must be claimed
   * before removing a vault, otherwise they will be lost.
   */
```

하지만 이 내용은 코드에서 강제되지 않습니다.


**영향:** 소유자가 먼저 보상을 청구하지 않고 vault를 제거하면, 그 보상을 회수할 방법이 없습니다.

**권장 완화 조치:** `EspressoVault::getRewards`를 통해 미청구 보상을 확인하고, 0이 아니면 revert하도록 고려하십시오.


**Stake.link:**
[a299ca7](https://github.com/stakedotlink/contracts/commit/a299ca747b0466248c741e08864161b0d2a930ea)에서 수정됨.

**Cyfrin:** 확인함.


### `withdrawRewards`와 `restakeRewards`에서 `maxRewardChangeBPS` 검증이 없어 보상 제한을 우회할 수 있음

**설명:** Reward Oracle이 Vault 보상을 업데이트할 때는 보상 증가 폭이 특정 임계값을 넘지 않는지 검증합니다. 증가량이 너무 크면 업데이트는 revert됩니다. 이 안전장치는 `StakingPool`이 `EspressoStrategy::updateDeposits`를 호출해 수수료 분배를 정산하고 `totalDeposits`를 갱신하기 전에, 과도한 보상 급증이 반영되지 않도록 막아 줍니다.

> contracts/espressoStaking/EspressoStrategy.sol#updateLifetimeRewards
```solidity
    function updateLifetimeRewards( ... ) external onlyRewardsOracle {
        if (_vaultIds.length != _lifetimeRewards.length) revert InvalidParamLengths();

        for (uint256 i = 0; i < _vaultIds.length; ++i) {
            vaults[_vaultIds[i]].updateLifetimeRewards(_lifetimeRewards[i]);
        }

        int256 rewards = getDepositChange();
>>      if (rewards > 0 && uint256(rewards) > (totalDeposits * maxRewardChangeBPS) / BASIS_POINTS)
            revert RewardsTooHigh();

        emit UpdateLifetimeRewards();
    }

```

문제는 보상을 갱신하는 경로가 `updateLifetimeRewards` 하나만이 아니라는 점입니다. Espresso Network의 `RewardClaim` 컨트랙트에서는 Light Client Root(Merkle Tree Proof)에 대해 `authData`를 검증해 보상 업데이트를 처리합니다. Espresso 문서에 따르면 보상은 최소 epoch당 한 번(대략 24시간) 업데이트되어야 하며, 누구나 이를 청구할 수 있습니다.

https://docs.espressosys.com/network/releases/testnets/decaf-testnet/running-a-node#staking
> With the initial release of Proof-of-stake, participation is limited to a dynamic, permissionless set of 100 nodes. **In each epoch (period of roughly 24 hours)** the 100 nodes with the most delegated stake form the active participation set.

https://docs.espressosys.com/network/concepts/the-espresso-network/internal-functionality/light-client#updating-and-verifying-lightclientstate
> Replica nodes update the snapshot of the stake table at the beginning of an epoch and this snapshot is used to define the set of stakers for the next epoch. **The light client state must be updated at least once per epoch**.

따라서 매일 새로운 proof를 이용해 `EspressoVault` 보상을 청구할 수 있습니다. 어떤 사용자든 `EspressoFundFlowController::restakeRewards`나 `EspressoFundFlowController::withdrawRewards`에 proof를 제공해 청구를 트리거할 수 있습니다. 이 함수들은 `claimRewards`와 `updateLifetimeRewards`를 직접 호출하지만, 누적 보상이 프로토콜이 설정한 `maxRewardChangeBPS` 한도를 넘는지는 검사하지 않습니다.

```solidity
    // contracts/espressoStaking/EspressoFundFlowController.sol#withdrawRewards
    function withdrawRewards( ... ) external {
        strategy.withdrawRewards(_vaultIds, _lifetimeRewards, _authData);
    }

    //contracts/espressoStaking/EspressoStrategy.sol#withdrawRewards
    function withdrawRewards( ... ) external onlyFundFlowController {
        ...
        for (uint256 i = 0; i < _vaultIds.length; ++i) {
>>          vaults[_vaultIds[i]].withdrawRewards(_lifetimeRewards[i], _authData[i]);
        }

        totalQueued += token.balanceOf(address(this)) - preBalance;

        emit WithdrawRewards();
    }

    // contracts/espressoStaking/EspressoVault.sol#withdrawRewards
    function withdrawRewards( ...  ) external onlyVaultController {
>>      _updateLifetimeRewards(_lifetimeRewards);

        if (getRewards() != 0) {
            espressoRewards.claimRewards(_lifetimeRewards, _authData);

            uint256 balance = token.balanceOf(address(this));
            token.safeTransfer(msg.sender, balance);
        }
    }

```

Reward oracle가 장기간 보상을 게시하지 못할 경우, 사용자는 Espresso API에서 최신 Merkle Root를 가져와 `restakeRewards` 또는 `withdrawRewards`를 호출할 수 있습니다. 누적 보상이 `maxRewardChangeBPS`를 크게 초과하더라도 이 경로에서는 트랜잭션이 revert되지 않고 처리됩니다.

https://docs.espressosys.com/network/api-reference/espresso-api/state-api#get-reward-state
> Get a Merkle proof proving the balance of a certain reward account in a given snapshot of the state.

**영향:** 하나의 트랜잭션에서 deposit change 값이 매우 크게 증가할 수 있어, `updateDeposits`에서 적용되는 `StakingPool` 제한을 우회하게 됩니다. 그 결과 `updateDeposits`가 지나치게 큰 값 또는 부정확한 간격에 대해 호출되어, 잘못된 수수료 누적과 보상 분배가 발생할 수 있습니다.

**개념 증명 (Proof of Concept):**
1. `maxRewardChangeBPS`가 `1%`로 설정되어 있습니다.
2. Vault들에 상당한 수수료가 누적되어 수익이 `2%`에 도달했지만, Oracle은 아직 업데이트하지 않았습니다.
3. 정상적인 Oracle 업데이트라면 `StakingPool`이 `updateDeposits`를 호출하기 전에 `1%` 증가까지만 허용됩니다.
4. 대신 공격자(또는 사용자)가 최신 Merkle Root를 가져와 `EspressoFundFlowController::withdrawRewards`를 호출합니다.
5. 모든 보상이 인출되고, `DepositChange`가 즉시 `2%` 증가합니다.
6. 이로써 `1%` `maxRewardChangeBPS` 제한을 우회하고 프로토콜 회계가 부정확해집니다.

**권장 완화 조치:** `EspressoFundFlowController::withdrawRewards`나 `EspressoFundFlowController::restakeRewards` 실행 후, `DepositChange` 증가량이 `maxRewardChangeBPS` 한도를 넘지 않았는지 검증해야 합니다.


**Stake.Link:** 인지함.


### 토큰 기부로 인해 일부 시나리오에서 보상 업데이트가 차단될 수 있음

**설명:** `EspressoStrategy::updateLifetimeRewards`는 `EspressoStrategy::getDepositChange`를 이용해 업데이트당 최대 보상 증가량을 강제합니다. 그러나 `EspressoStrategy::getDepositChange`에는 직접 전송으로 조작 가능한 토큰 잔액이 포함됩니다.

공격자는 전략 또는 vault에 소량의 토큰을 기부해 deposit change를 인위적으로 부풀릴 수 있으며, 그 결과 정당한 보상 업데이트가 revert되게 만들 수 있습니다. 자연 발생 보상이 `maxRewardChangeBPS` 한계에 가까운 상황이라면 공격 비용은 매우 낮을 수 있습니다.

 보상 업데이트 검사는 다음과 같습니다.

```solidity
// EspressoStrategy.sol
  function updateLifetimeRewards(
      uint256[] calldata _vaultIds,
      uint256[] calldata _lifetimeRewards
  ) external onlyRewardsOracle {
      for (uint256 i = 0; i < _vaultIds.length; ++i) {
          vaults[_vaultIds[i]].updateLifetimeRewards(_lifetimeRewards[i]);
      }

      int256 rewards = getDepositChange();  //@audit this can be manipulated by donation
      if (rewards > 0 && uint256(rewards) > (totalDeposits * maxRewardChangeBPS) / BASIS_POINTS)      //@audit causing this to revert
          revert RewardsTooHigh();
  }
```

`EspressoStrategy::getDepositChange`의 출력은 전략 컨트랙트에 토큰을 기부함으로써 조작될 수 있습니다.

```solidity
  function getDepositChange() public view returns (int) {
      uint256 totalBalance = token.balanceOf(address(this));  // @audit can inflate this by donation
      for (uint256 i = 0; i < vaults.length; ++i) {
          totalBalance += vaults[i].getTotalDeposits();
      }
      return int(totalBalance) - int(totalDeposits);  //@note can increase this
  }
```

필요한 최소 기부량은 자연 발생 보상이 임계값에 얼마나 근접했는지에 따라 달라집니다. 자연 보상이 한도에 매우 가까운 경우, 소량의 기부만으로도 보상 업데이트 흐름을 멈출 수 있습니다.


**영향:** rewardsOracle이 lifetime rewards를 갱신하지 못하게 되어 회계 시스템에서 보상이 인식되지 않습니다. 다만 이 공격은 일시적이며, 전략 보상이 업데이트되기 전까지만 지속된다는 점은 참고해야 합니다.

**권장 완화 조치:** 총 deposit change를 통해 추론하는 대신 실제 보상 증가량을 직접 계산하는 방안을 고려하십시오.

```diff
 function updateLifetimeRewards(
      uint256[] calldata _vaultIds,
      uint256[] calldata _lifetimeRewards
  ) external onlyRewardsOracle {
      if (_vaultIds.length != _lifetimeRewards.length) revert InvalidParamLengths();

++      uint256 rewardsBefore;
++      for (uint256 i = 0; i < _vaultIds.length; ++i) {
++         rewardsBefore += vaults[_vaultIds[i]].getRewards();
++      }


--   int256 rewards = getDepositChange();
      for (uint256 i = 0; i < _vaultIds.length; ++i) {
          vaults[_vaultIds[i]].updateLifetimeRewards(_lifetimeRewards[i]);
      }

++      uint256 rewardsAfter;
++     for (uint256 i = 0; i < _vaultIds.length; ++i) {
++          rewardsAfter += vaults[_vaultIds[i]].getRewards();
++      }

++      uint256 rewards = rewardsAfter - rewardsBefore;
      if (rewards > (totalDeposits * maxRewardChangeBPS) / BASIS_POINTS)
          revert RewardsTooHigh();

      emit UpdateLifetimeRewards();
  }

 ```

**Stake.link:**
인지함.


### `EspressoVault::exitIsWithdrawable`가 자산이 이미 인출되었는지 확인하지 않음

**설명:** 특정 validator가 `EspressoStaking`에서 종료되면, 잠금 기간이 지난 뒤 위임자의 자금은 인출 가능해집니다.

`EspressoVault`에는 종료된 validator의 자금이 인출 가능한지 판단하는 `view` 함수가 있으며, 이는 `EspressoFundFlowController`에서도 사용됩니다. 현재 이 함수는 `validatorExits::unlockAt` 값만 확인합니다.

> contracts/espressoStaking/EspressoVault.sol#exitIsWithdrawable
```solidity
    function exitIsWithdrawable() external view returns (bool) {
        uint256 unlocksAt = espressoStaking.validatorExits(validator);

        return unlocksAt != 0 && block.timestamp >= unlocksAt;
    }
```

`EspressoStaking` 컨트랙트에서 `validatorExits` 매핑은 출금 청구 후에도 변경되지 않습니다. 따라서 자금이 이미 인출된 뒤에도 이 view 함수는 계속 `true`를 반환합니다. 하지만 이후 `claimValidatorExit`를 다시 호출하면 위임 금액이 0으로 재설정되어 있기 때문에 revert됩니다.

https://github.com/EspressoSystems/espresso-network/blob/main/contracts/src/StakeTableV2.sol#L461-L486
```solidity
    function claimValidatorExit(address validator) public virtual override whenNotPaused {
        address delegator = msg.sender;
        uint256 unlocksAt = validatorExits[validator];
        if (unlocksAt == 0) {
            revert ValidatorNotExited();
        }

        if (block.timestamp < unlocksAt) {
            revert PrematureWithdrawal();
        }

        uint256 amount = delegations[validator][delegator];
        if (amount == 0) {
            revert NothingToWithdraw();
        }

        // Mark funds as spent
        delegations[validator][delegator] = 0;
        // the delegatedAmount is updated here (instead of during deregistration) in v2,
        // it's only decremented during withdrawal
        validators[validator].delegatedAmount -= amount;

        SafeTransferLib.safeTransfer(token, delegator, amount);

        emit ValidatorExitClaimed(delegator, validator, amount);
    }
```

**영향:**
- Stake.Link 오프체인 시스템이 잘못된 결과를 가져와, 이미 출금이 끝난 vault도 여전히 출금 가능한 것으로 식별하게 됩니다.

**권장 완화 조치:** 이 함수가 vault/delegator에 남아 있는 delegation balance도 함께 확인하도록 업데이트해야 합니다.

```diff
     function exitIsWithdrawable() external view returns (bool) {
         uint256 unlocksAt = espressoStaking.validatorExits(validator);
+        uint256 amount = espressoStaking.delegations(validator, address(this));

-        return unlocksAt != 0 && block.timestamp >= unlocksAt;
+        return unlocksAt != 0 && block.timestamp >= unlocksAt && amount > 0;
     }
```


**Stake.Link:** [e37f39f](https://github.com/stakedotlink/contracts/commit/e37f39f64e354e9d4356d16d672c1e436e062566)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage
## 정보성 (Informational)


### `FundFlowController::unbondVaults`는 여러 전략을 가진 Espresso Staking Pool에서 영구적인 DoS 상태가 될 수 있음

**설명:** `ESP` 토큰을 대상으로 하는 Staking pool은 하나 이상의 전략을 지원하도록 설계되어 있습니다.

> contracts/core/StakingPool.sol
```solidity
contract StakingPool is StakingRewardsPool {
    ...

    // list of all strategies controlled by pool
    address[] private strategies;
    ...
}
```

출금이 시작되면 `WithdrawalPool`에 큐잉됩니다. 이는 원자산을 되찾기 위해 스테이킹된 토큰의 출금을 요청할 때 발생합니다. 이 과정은 `PriorityPool::_withdraw` 호출로 트리거됩니다(대기 중인 예치가 없고 즉시 출금이 비활성화된 경우를 가정).

> contracts/core/priorityPool/PriorityPool.sol::_withdraw
```solidity
    function _withdraw( ... ) internal returns (uint256) {
        ...
        if (totalQueued != 0) { ... }

        if (
            toWithdraw != 0 &&
            allowInstantWithdrawals &&
            withdrawalPool.getTotalQueuedWithdrawals() == 0
        ) { ... }

        if (toWithdraw != 0) {
            if (!_shouldQueueWithdrawal) revert InsufficientLiquidity();

            if (toWithdraw >= withdrawalPool.minWithdrawalAmount()) {
>>              withdrawalPool.queueWithdrawal(_account, toWithdraw);
                queued = toWithdraw;
            } else {
                IERC20Upgradeable(address(stakingPool)).safeTransfer(_account, toWithdraw);
            }
        }
        ...
    }
```

`PriorityPool`에서 대기 중인 출금을 실행할 때는 `StakingPool::withdraw`가 호출됩니다. 이 함수는 `WithdrawalPool`의 대기 출금을 마무리하기 위해 여러 전략에서 유동성을 끌어옵니다.

> contracts/core/priorityPool/PriorityPool.sol#executeQueuedWithdrawals
```solidity
    function executeQueuedWithdrawals( ... ) external onlyWithdrawalPool {
        IERC20Upgradeable(address(stakingPool)).safeTransferFrom( ... );
>>      stakingPool.withdraw(address(this), address(this), _amount, _data);
        token.safeTransfer(msg.sender, _amount);
    }
```

컨트랙트에 메인 토큰이 충분하지 않으면 전략들에서 유동성을 인출합니다. `StakingPool`은 여러 전략을 가질 수 있으므로, 대기 출금 금액을 충족하기 위해 각 전략을 순회합니다.

> contracts/core/StakingPool.sol#withdraw
```solidity
    function withdraw( ... ) external onlyPriorityPool {
        uint256 toWithdraw = _amount;
        if (_amount == type(uint256).max) {
            toWithdraw = balanceOf(_account);
        }

        uint256 balance = token.balanceOf(address(this));
        if (toWithdraw > balance) {
>>          _withdrawLiquidity(toWithdraw - balance, _data);
        }
        require(
            token.balanceOf(address(this)) >= toWithdraw,
            "Not enough liquidity available to withdraw"
        );

        _burn(_account, toWithdraw);
        totalStaked -= toWithdraw;
        token.safeTransfer(_receiver, toWithdraw);
    }
// --------
    function _withdrawLiquidity(uint256 _amount, bytes[] calldata _data) private {
        ...
        for (uint256 i = strategies.length; i > 0; i--) {
            ...
            if (strategyCanWithdrawdraw >= toWithdraw) {
>>              strategy.withdraw(toWithdraw, strategyData);
                break;
            } else if (strategyCanWithdrawdraw > 0) {
>>              strategy.withdraw(strategyCanWithdrawdraw, strategyData);
                ...
            }
        }
    }
```

`EspressoStrategy`에서 출금하려면 컨트랙트가 충분한 `ESP` 토큰을 보유해야 합니다. 이는 언본딩 과정에서만 가능해집니다. 그러나 단일 전략에서 언본딩할 때 현재 로직은 **전역 queued withdrawal 총액 전체를 해당 전략 하나에서 언본딩하도록 강제**합니다.

> contracts/espressoStaking/EspressoFundFlowController.sol#unbondVaults
```solidity
    function unbondVaults() external {
>>      uint256 queuedWithdrawals = withdrawalPool.getTotalQueuedWithdrawals();
        uint256 queuedDeposits = strategy.totalQueued();

        if ( ... ) revert NoUnbondingNeeded();

        uint256 toWithdraw = queuedWithdrawals - queuedDeposits;
>>      strategy.unbond(toWithdraw);
        timeOfLastUnbond = uint64(block.timestamp);
    }
```

전략이 둘 이상이고 총 queued withdrawal 금액이 개별 전략의 예치금을 초과하는 경우, `EspressoFundFlowController#unbondVaults` 호출은 항상 revert됩니다. 개별 전략 하나만으로는 전역 출금 요구량을 충족할 수 없기 때문입니다.

> contracts/espressoStaking/EspressoStrategy.sol#unbond
```solidity
    function unbond(uint256 _toUnbond) external onlyFundFlowController {
        ...
        while (toUnbondRemaining != 0) {
            ...
            if (i == vaultWithdrawalIndex) break;
        }

>>      if (toUnbondRemaining > 0) revert InsufficientDeposits();

        vaultWithdrawalIndex = i;
        numVaultsUnbonding = numVaultsUnbonded;

        emit Unbond(_toUnbond);
    }
```

**영향:**
- 총 queued withdrawal 금액이 처리 중인 특정 전략의 총 예치금을 초과하면, 사용자는 `EspressoStrategy`에서 자산을 언본딩할 수 없습니다(즉 토큰 출금 요청을 처리할 수 없습니다).

**개념 증명 (Proof of Concept):**
- Espresso Staking Pool은 `Strategy A`, `Strategy B` 두 전략을 사용합니다.
- 각 전략에는 `10,000` 토큰이 예치되어 있습니다.
- 새로운 예치는 없고 즉시 출금은 비활성화되어 있습니다.
- 사용자가 출금을 요청해 `WithdrawalPool` 총 큐가 `15,000` 토큰이 됩니다.
- 사용자가 `EspressoFundFlowController::unbondVaults`를 통해 `Strategy A`에서 자산을 언본딩하려고 시도합니다.
- 이 호출은 `toWithdraw = 15,000`을 계산합니다.
- `Strategy A`는 10,000 토큰만 보유하고 있으므로 `InsufficientDeposits`로 revert되지만, 시스템 전체 유동성(20,000)은 충분합니다.

**권장 완화 조치:** `EspressoStrategy`에 모든 Vault를 순회해 사용 가능한 예치 자산을 추적하는 함수를 도입하십시오.

- 활성 Vault는 `getPrincipalDeposits`를 사용합니다.
- 비활성 Vault는 `getTotalDeposits`를 사용합니다.

요청된 언본딩 금액은 해당 전략의 개별 총 예치금으로 상한 처리되어야 합니다. 이렇게 하면 한 전략은 자신이 담당 가능한 만큼만 완전히 언본딩하고, 다른 전략들이 남은 금액을 처리하여 전체 사용자 출금 요청을 함께 충족할 수 있습니다.

**Stake.Link:** 인지함.


### `vaultImplementation` 변경이 `EspressoStrategy::addVault` 함수를 DoS시킬 수 있음


**설명**
`EspressoStrategy`에서 소유자는 `addVault`를 호출해 새 vault를 배포할 수 있습니다. 이 함수는 현재 `vaultImplementation`을 가리키는 `ERC1967Proxy`를 생성합니다.

배포 과정에서는 정확히 다섯 개의 파라미터를 갖는 하드코딩된 시그니처로 `initialize` 함수를 호출합니다.

> contracts/espressoStaking/EspressoStrategy.sol#addVault
```solidity
    function addVault(address _validator) external onlyOwner {
        address vault = address(
            new ERC1967Proxy(
                vaultImplementation,
                abi.encodeWithSignature(
                    "initialize(address,address,address,address,address)",
                    address(token),
                    address(this),
                    address(espressoStaking),
                    address(espressoRewards),
                    _validator
                )
            )
        );
        token.safeApprove(vault, type(uint256).max);
        vaults.push(IEspressoVault(vault));

        emit AddVault(_validator);
    }
// -------
    function setVaultImplementation(address _vaultImplementation) external onlyOwner {
        if (_vaultImplementation == address(0)) revert InvalidAddress();
        vaultImplementation = _vaultImplementation;
        emit SetVaultImplementation(_vaultImplementation);
    }

```

`vaultImplementation`은 관리자가 언제든지 변경할 수 있습니다. 또한 `upgradeVaults` 함수는 업그레이드 호출용 임의의 데이터를 허용하면서 기존 vault를 새 구현으로 업그레이드할 수 있게 합니다.

> contracts/espressoStaking/EspressoStrategy.sol#upgradeVaults

```solidity
    function upgradeVaults(address[] calldata _vaults, bytes[] memory _data) external onlyOwner {
        for (uint256 i = 0; i < _vaults.length; ++i) {
            if (_data.length == 0 || _data[i].length == 0) {
                IEspressoVault(_vaults[i]).upgradeTo(vaultImplementation);
            } else {
                IEspressoVault(_vaults[i]).upgradeToAndCall(vaultImplementation, _data[i]);
            }
        }
        emit UpgradedVaults(_vaults);
    }

```

새 `vaultImplementation`이 다른 `initialize` 시그니처를 요구한다면(예: 파라미터 수가 다르거나 타입이 바뀐다면), `addVault` 함수는 깨지게 됩니다. `addVault`는 기존 5-파라미터 시그니처를 강제하므로, 호출은 시그니처 불일치로 revert되거나 잘못된 데이터로 초기화될 수 있습니다.

**영향**
- 초기화 스키마가 다른 버전으로 `vaultImplementation`을 변경하면 `addVault` 함수가 영구적으로 DoS됩니다.
- 시그니처는 같더라도 파라미터 타입이 변경된 경우, 배포된 vault가 비일관적이거나 깨진 상태가 될 수 있습니다.

**개념 증명 (Proof of Concept)**

1. 관리자가 추가 파라미터를 요구하는 새 버전(예: `initialize(address,address,address,address,address,uint256)`)으로 `vaultImplementation`을 변경합니다.
2. 관리자가 `addVault`를 호출하려고 시도합니다.
3. `ERC1967Proxy` 생성자가 새 구현에서 하드코딩된 5-파라미터 `initialize` 호출을 실행하지 못해 트랜잭션이 revert됩니다.

**권장 완화 조치**

임의의 초기화 데이터를 받을 수 있는 오버로드된 `addVault` 함수(또는 기존 함수 수정)를 도입하십시오. 이는 `upgradeVaults`가 이미 제공하는 유연한 업그레이드 로직과 배포 로직을 일치시킵니다.

**Stake.Link:** 인지함.
