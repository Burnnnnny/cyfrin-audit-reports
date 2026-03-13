**수석 감사자**

[0kage](https://x.com/0kage_eth)

[holydevoti0n](https://github.com/h0lydev)


**보조 감사자**


---

# 결과 (Findings)
## 중간 위험 (Medium Risk)


### `PolygonStrategy::unbond`에서 부적절한 검증자 출금 인덱스 진행으로 인한 언본딩의 불공정한 분배

**설명:** `PolygonStrategy::unbond`는 `validatorWithdrawalIndex`를 추적하여 여러 볼트(vault)에 걸쳐 언본딩 요청을 균등하게 분배합니다. 그러나 인덱스는 실제 언본딩 작업이 아닌 루프 진행에 따라 진행되므로 검증자 간에 언본딩이 고르지 않게 분배될 수 있습니다.

현재 언본딩 중 볼트를 처리할 때:

 - 함수는 `validatorWithdrawalIndex`에서 시작하여 볼트를 순회합니다.
 - 볼트에 언본딩 금액을 충당할 충분한 보상이 있는 경우, 원금의 실제 언본딩 없이 보상만 인출됩니다.
 - 원금이 언본딩되지 않았더라도 `validatorWithdrawalIndex`는 순서상 다음 볼트로 업데이트됩니다.
 - 인덱스가 볼트 배열의 끝에 도달하면 0으로 래핑됩니다.

이 로직은 높은 보상을 가진 검증자가 자신의 원금 예치금을 언본딩으로부터 효과적으로 "방어"하게 하여, 이미 언본딩을 했을 수도 있는 다른 검증자에게 언본딩 부담을 불공정하게 전가합니다.

구체적으로 `unbond` 함수에서:

```solidity
while (toUnbondRemaining != 0) {
    // Process vault[i]...
    ++i;
    if (i >= vaults.length) i = 0;
}
validatorWithdrawalIndex = i; // @audit 마지막으로 처리된 볼트에서 언본딩이 발생했는지 여부와 관계없이 발생
```
보상이 `unbond` 요청을 충족하기에 충분하더라도 `vaultWithdrawalIndex`는 여전히 진행됩니다.


**파급력:** 이 문제는 검증자 간의 언본딩 작업 분배에 잠재적인 공정성 문제를 야기합니다. 더 많은 보상을 생성하는 검증자는 원금 예치금이 언본딩될 가능성이 낮은 반면, 보상이 적은 검증자는 불균형적인 언본딩 부담을 지게 됩니다.

사실상 일부 볼트는 반복적으로 언본딩되는 반면 다른 볼트는 차례를 건너뛰게 됩니다.


**개념 증명 (Proof of Concept):** `polygon-strategy.test.ts`의 라인 304에 있는 `unbond should work correctly` 테스트에서, vault[2]에 대한 언본딩이 없었음에도 불구하고 `validatorWithdrawalIndex`가 0으로 재설정되는 것을 확인하십시오. vault[0]은 두 번 언본딩되는 반면 vault[2]는 현재 라운드에서 언본딩을 건너뜁니다.


**권장되는 완화 방법:** `unbond` 함수를 수정하여 보상만 인출될 때가 아니라 실제 원금 예치금의 언본딩이 발생할 때만 `validatorWithdrawalIndex`를 진행하도록 하십시오:

```diff solidity
function unbond(uint256 _toUnbond) external onlyFundFlowController {
    if (numVaultsUnbonding != 0) revert UnbondingInProgress();
    if (_toUnbond == 0) revert InvalidAmount();

    uint256 toUnbondRemaining = _toUnbond;
    uint256 i = validatorWithdrawalIndex;
    uint256 skipIndex = validatorRemoval.isActive
        ? validatorRemoval.validatorId
        : type(uint256).max;
    uint256 numVaultsUnbonded;
    uint256 preBalance = token.balanceOf(address(this));
++ uint256 nextIndex = i; // Track the next index separately

    while (toUnbondRemaining != 0) {
        if (i != skipIndex) {
            IPolygonVault vault = vaults[i];
            uint256 deposits = vault.getTotalDeposits();

            if (deposits != 0) {
                uint256 principalDeposits = vault.getPrincipalDeposits();
                uint256 rewards = deposits - principalDeposits;

                if (rewards >= toUnbondRemaining) {
                    vault.withdrawRewards();
                    toUnbondRemaining = 0;
                } else {
                    toUnbondRemaining -= rewards;
                    uint256 vaultToUnbond = principalDeposits >= toUnbondRemaining
                        ? toUnbondRemaining
                        : principalDeposits;

                    vault.unbond(vaultToUnbond);
++               nextIndex = (i + 1) % vaults.length; // Update next index only when unbonding principal
                    toUnbondRemaining -= vaultToUnbond;
                    ++numVaultsUnbonded;
                }
            }
        }

        ++i;
        if (i >= vaults.length) i = 0;
    }
--    validatorWithdrawalIndex = i;
++    // Only update the index if we actually unbonded principal from at least one vault
++    if (numVaultsUnbonded > 0) {
++       validatorWithdrawalIndex = nextIndex;
++   }

    numVaultsUnbonding = numVaultsUnbonded;

    uint256 rewardsClaimed = token.balanceOf(address(this)) - preBalance;
    if (rewardsClaimed != 0) totalQueued += rewardsClaimed;

    emit Unbond(_toUnbond);
}

```

**Stake.link:**
[PR 151](https://github.com/stakedotlink/contracts/pull/151/commits/9fd654b46a0b9008a134fe67145d446f7e04d8ce)에서 해결되었습니다.

**Cyfrin:** 해결되었습니다.


### `PolygonStrategy.sol`에서 검증자 및 프로토콜 수수료가 보상의 100%를 초과할 수 있음

**설명:** `PolygonStrategy.sol`의 `setValidatorMEVRewardsPercentage` 함수는 검증자 MEV 보상 비율을 최대 100%(10,000 베이시스 포인트)까지 설정할 수 있도록 허용하지만, 전체 프로토콜 수수료(`_totalFeesBasisPoints`)를 고려하지 않습니다. 이는 모든 수수료의 합계가 보상의 100%를 초과하여 보상의 과다 분배로 이어질 수 있음을 의미합니다. `setValidatorMEVRewardsPercentage`는 자체 한도만 확인합니다:
```solidity
function setValidatorMEVRewardsPercentage(
    uint256 _validatorMEVRewardsPercentage
) external onlyOwner {
    // @audit should consider the totalFeesBasisPoints.
    // updateDeposits will report more than it can pay in fees.
    if (_validatorMEVRewardsPercentage > 10000) revert FeesTooLarge();

    validatorMEVRewardsPercentage = _validatorMEVRewardsPercentage;
    emit SetValidatorMEVRewardsPercentage(_validatorMEVRewardsPercentage);
}
```
다른 수수료의 한도는 베이시스 포인트로 30%임을 확인하십시오:
```solidity
// updateFee
if (_totalFeesBasisPoints() > 3000) revert FeesTooLarge();
```
이러한 분리는 `validatorMEVRewardsPercentage`와 프로토콜 수수료의 합이 보상의 100%를 초과하도록 허용하여 `StakingPool.sol`에서 과다 발행 시나리오로 이어질 수 있습니다:
```solidity
function updateDeposits(
        bytes calldata
    )
        external
        onlyStakingPool
        returns (int256 depositChange, address[] memory receivers, uint256[] memory amounts)
    {
    ...
@>            uint256 validatorMEVRewards = ((balance - totalQueued) *
                validatorMEVRewardsPercentage) / 10000;

            receivers = new address[](fees.length + (validatorMEVRewards != 0 ? 1 : 0));
            amounts = new uint256[](receivers.length);

            for (uint256 i = 0; i < fees.length; ++i) {
                receivers[i] = fees[i].receiver;
@>                amounts[i] = (uint256(depositChange) * fees[i].basisPoints) / 10000;
            }

            if (validatorMEVRewards != 0) {
@>                receivers[receivers.length - 1] = address(validatorMEVRewardsPool);
@>                amounts[amounts.length - 1] = validatorMEVRewards;
            }
           ...
}

// StakingPool._updateStrategyRewards
uint256 sharesToMint = (totalFeeAmounts * totalShares) /
    (totalStaked - totalFeeAmounts);
_mintShares(address(this), sharesToMint);
```

**예시**
1. 소유자가 `validatorMEVRewardsPercentage`를 100%(10,000 베이시스 포인트)로 설정합니다.
2. `_totalFeesBasisPoints`를 통한 프로토콜 수수료가 20%(2,000 베이시스 포인트)입니다.
3. 이제 컨트랙트의 총 수수료율은 120%(10,000 + 2,000 베이시스 포인트)입니다.
4. 보상으로 10개의 토큰을 사용할 수 있는 경우, `updateDeposits`는 `StakingPool` 로직에 12개의 토큰을 발행하도록 보고합니다.
5. 이로 인해 시스템은 실제로 보유한 것보다 더 많은 토큰을 발행하게 되어 풀의 지분이 과다 발행됩니다.

**파급력:** 프로토콜 수수료가 보상의 100%를 초과할 수 있습니다.

**개념 증명 (Proof of Concept):** `polygon-strategy.test.ts`에서 전략 초기화를 수정하여 20%의 수수료를 포함하십시오:
```diff
const strategy = (await deployUpgradeable('PolygonStrategy', [
      token.target,
      stakingPool.target,
      stakeManager.target,
      vaultImp,
      2500,
-    []
+      [
+        {
+          receiver: ethers.Wallet.createRandom().address,
+          basisPoints: 1000 // 10%
+        },
+        {
+          receiver: ethers.Wallet.createRandom().address,
+          basisPoints: 1000 // 10%
+        }
+      ],
    ])) as PolygonStrategy
```
그런 다음 다음 테스트를 추가하고 `npx hardhat test test/polygonStaking/polygon-strategy.test.ts`를 실행하십시오:
```javascript
 describe.only('updateDeposits mint above 100%', () => {
    it('should cause stakingPool to overmint rewards', async () => {
      const { strategy, token, vaults, accounts, stakingPool, validatorShare, validatorShare2 } =
        await loadFixture(deployFixture)

    await strategy.setValidatorMEVRewardsPercentage(10000) // 100%
    // current fees is 20% for receivers + 100% for validator rewards

    await stakingPool.deposit(accounts[1], toEther(1000), ['0x'])
    await strategy.depositQueuedTokens([0, 1, 2], [toEther(10), toEther(20), toEther(30)])

    assert.equal(fromEther(await strategy.getTotalDeposits()), 1000)
    assert.equal(fromEther(await strategy.totalQueued()), 940)
    assert.equal(fromEther(await strategy.getDepositChange()), 0)

    // send 10 MATIC to strategy
    await token.transfer(strategy.target, toEther(10))
    assert.equal(fromEther(await strategy.getDepositChange()), 10)

  // @audit we will mint 120% of 10 instead of 100%
  // notice totalRewards is 10(depositChange),
  await expect(stakingPool.updateStrategyRewards([0], '0x'))
  .to.emit(stakingPool, "UpdateStrategyRewards")
  // totalRewards = strategy.depositChange
  //       msg.sender,  totalStaked,   totalRewards, totalFeeAmounts
  .withArgs(accounts[0], toEther(1010), toEther(10), toEther(12));
    })
  });
```
출력:
```javascript
PolygonStrategy
    updateDeposits mint above 100%
      ✔ should cause stakingPool to overmint rewards (608ms)

  1 passing (611ms)
```


**권장되는 완화 방법:** 수수료를 업데이트할 때 총 수수료(mevRewards + fees)가 100%를 초과하지 않는지 확인하십시오:
```diff
    function setValidatorMEVRewardsPercentage(
        uint256 _validatorMEVRewardsPercentage
    ) external onlyOwner {
-         if (_validatorMEVRewardsPercentage > 10000) revert FeesTooLarge();
+        if (_validatorMEVRewardsPercentage + _totalFeesBasisPoints() > 10000) revert FeesTooLarge();

        validatorMEVRewardsPercentage = _validatorMEVRewardsPercentage;
        emit SetValidatorMEVRewardsPercentage(_validatorMEVRewardsPercentage);
    }
    function addFee(address _receiver, uint256 _feeBasisPoints) external onlyOwner {
        _updateStrategyRewards();
        fees.push(Fee(_receiver, _feeBasisPoints));
-        if (_totalFeesBasisPoints() > 3000) revert FeesTooLarge();
+       uint256 totalFees = _totalFeesBasisPoints();
+       if (totalFees > 3000 || totalFees + validatorMEVRewardsPercentage > 10000) revert FeesTooLarge();
        emit AddFee(_receiver, _feeBasisPoints);
    }

       function updateFee(
        uint256 _index,
        address _receiver,
        uint256 _feeBasisPoints
    ) external onlyOwner {
        _updateStrategyRewards();

        if (_feeBasisPoints == 0) {
            fees[_index] = fees[fees.length - 1];
            fees.pop();
        } else {
            fees[_index].receiver = _receiver;
            fees[_index].basisPoints = _feeBasisPoints;
        }

-        if (_totalFeesBasisPoints() > 3000) revert FeesTooLarge();
+       uint256 totalFees = _totalFeesBasisPoints();
+       if (totalFees > 3000 || totalFees + validatorMEVRewardsPercentage > 10000) revert FeesTooLarge();
        emit UpdateFee(_index, _receiver, _feeBasisPoints);
    }
```


**Stake.link:**
[PR 151](https://github.com/stakedotlink/contracts/pull/151/commits/3c5d2526185e4911eca5826ac136f3d21d32eab2)에서 해결되었습니다.

**Cyfrin:** 해결되었습니다.

\clearpage
## 낮은 위험 (Low Risk)


### `PolygonStrategy::upgradeVaults`가 업그레이드할 볼트가 전략에 저장된 것과 동일한지 확인하지 않음

**설명:** `PolygonStrategy::upgradeVaults`는 `_vaults` 배열을 입력으로 받아 다음과 같이 최신 `vaultImplementation`으로 업그레이드합니다:

```solidity
 function upgradeVaults(address[] calldata _vaults, bytes[] memory _data) external onlyOwner {

        for (uint256 i = 0; i < _vaults.length; ++i) { //@audit no check if the _vault address is actually one locally stored
            if (_data.length == 0 || _data[i].length == 0) {
                IPolygonVault(_vaults[i]).upgradeTo(vaultImplementation);
            } else {
                IPolygonVault(_vaults[i]).upgradeToAndCall(vaultImplementation, _data[i]);
            }
        }
        emit UpgradedVaults(); //@audit does not emit the list of vaults actually updated
    }
```

현재 구현에는 몇 가지 문제가 있습니다:

1. 입력된 주소가 실제로 컨트랙트에 로컬로 저장된 주소와 일치하는지 확인하는 절차가 없습니다. 소유자가 전혀 다른 주소를 보내더라도 `UpgradedVaults` 이벤트를 트리거할 수 있습니다.
2. `UpgradedVaults` 이벤트는 실제로 업데이트된 볼트 목록을 포함하지 않습니다. 이로 인해 오프체인 리스너가 어떤 볼트가 실제로 업그레이드되었는지 감지할 수 없습니다.
3. 특정 볼트를 입력으로 전달하면 일부 볼트는 최신 구현에서 실행되고 다른 배치는 이전 볼트 구현에 해당하는 시나리오가 발생할 수 있습니다. 이는 프로토콜 관리자에게 혼란을 줄 뿐만 아니라, 특히 볼트 구현 컨트랙트에서 버그가 발견된 경우 업그레이드 프로세스를 위험하게 만들 수 있습니다.


**파급력:** 볼트 업그레이드에 대한 투명성 부족은 컨트랙트 업그레이드와 관련된 인적 오류로 이어질 수 있습니다.

**권장되는 완화 방법:** 다음과 같은 변경을 고려하십시오:
1. 전략 내의 모든 볼트를 한 번에 새 구현으로 업그레이드합니다.
2. 볼트가 너무 많아 1번이 불가능한 경우, 업그레이드된 볼트 인덱스를 내보내십시오. 또한 볼트 주소를 입력으로 전달하는 대신 볼트 인덱스를 `upgradeVaults` 함수의 입력으로 전달하십시오.

**Stake.Link:** [PR 151](https://github.com/stakedotlink/contracts/pull/151/commits/c62e94ca962f7ba732f1a74396e5319dd5014697)에서 해결되었습니다.

**Cyfrin:** 해결되었습니다.


### 보상 재예치(Restaking)가 엣지 케이스에서 불필요한 언본딩을 유발함

**설명:** `PolygonStrategy::restakeRewards()` 함수는 누구나 호출할 수 있으며, `PolygonStrategy::unbond()` 함수가 호출되기 직전에 보상이 즉시 재예치될 수 있도록 합니다.

특정 시나리오에서, 이는 보상이 출금 요청을 충족하기에 충분했을 때 프로토콜이 불필요하게 원금 자금을 언본딩하게 할 수 있습니다.

이 문제는 `unbond()` 함수에서 보상이 계산되는 방식에서 비롯됩니다:

```solidity
// In PolygonStrategy::unbond()
uint256 deposits = vault.getTotalDeposits();
uint256 principalDeposits = vault.getPrincipalDeposits();
uint256 rewards = deposits - principalDeposits; //@audit this would be 0 if restakeRewards is called just before

if (rewards >= toUnbondRemaining) {
    vault.withdrawRewards();
    toUnbondRemaining = 0;
} else {
    toUnbondRemaining -= rewards;
    uint256 vaultToUnbond = principalDeposits >= toUnbondRemaining
        ? toUnbondRemaining
        : principalDeposits;

    vault.unbond(vaultToUnbond);

    toUnbondRemaining -= vaultToUnbond;
    ++numVaultsUnbonded; //@audit this will be 1 more than necessary
}

```

반면 `PolygonStrategy::restakeRewards()` 함수는 공개적으로 접근 가능합니다:

```solidity
// In PolygonStrategy::restakeRewards()
function restakeRewards(uint256[] calldata _vaultIds) external {
    for (uint256 i = 0; i < _vaultIds.length; ++i) {
        vaults[_vaultIds[i]].restakeRewards();
    }

    emit RestakeRewards();
}
```

**파급력:** 구체적인 시나리오를 고려해 보십시오:

- 첫 번째 적격 볼트가 언본딩 요청을 완전히 커버할 수 있는 충분한 보상을 가지고 있습니다.
- `unbond()`가 호출되기 직전에 앨리스가 `restakeRewards()`를 호출합니다.
- `unbond()`가 실행될 때 보상이 없음을 확인하고 대신 원금을 언본딩해야 합니다.
- 이는 `numVaultsUnbonding`을 불필요하게 증가시킵니다.

이 시나리오에서 대기 중인 토큰 예금은 서비스 거부(DoS)에 직면하게 됩니다.

**개념 증명 (Proof of Concept):** 다음을 추가하십시오:

```javascript
 describe.only('restakeRewards forces protocol to wait for withdrawal delay', async () => {
    it('should force protocol to wait for withdrawal delay', async () => {
      const { token, strategy, vaults, fundFlowController, withdrawalPool, validatorShare } =
        await loadFixture(deployFixture)

      await withdrawalPool.setTotalQueuedWithdrawals(toEther(960))
      assert.equal(await fundFlowController.shouldUnbondVaults(), true);

      // 1. Pre-condition: one validator can cover the amount to unbond with his rewards.
      await validatorShare.addReward(vaults[0].target, toEther(1000));

      // 2. User front-runs the unbondVaults transaction and restakes rewards for validators.
      await strategy.restakeRewards([0]);

      const totalQueuedBefore = await strategy.totalQueued();
      // 3. Protocol is forced to unbond and wait for withdrawal delay even though
      // it could obtain the funds after calling vault.withdrawRewards.
      await fundFlowController.unbondVaults()
      const totalQueuedAfter = await strategy.totalQueued();

      assert.equal(await fundFlowController.shouldUnbondVaults(), false)

      // 4. TotalQueued did not increase, meaning strategy could not withdraw any funds
      // and vault is unbending(it wouldn't be if it cover the funds with the rewards).
      assert.equal(totalQueuedBefore, totalQueuedAfter)
      assert.equal(await vaults[0].isUnbonding(), true)
    })
  })
```

출력은 다음과 같습니다:

```javascript
  PolygonFundFlowController
    restakeRewards forces protocol to wait for withdrawal delay
      ✔ should force protocol to wait for withdrawal delay (626ms)


  1 passing (628ms)
```

**권장되는 완화 방법:** 전략 및 볼트 컨트랙트 모두에서 `restakeRewards`에 대한 접근을 제한하는 것을 고려하십시오.


**Stake.Link:** 인지함. 정상적인 상황에서는 입금/언본딩이 있을 때마다 보상이 청구되므로 상당한 양의 미청구 보상이 있어서는 안 됩니다.

**Cyfrin:** 인지함.


### 검증자 보상이 최소 금액 임계값 아래로 떨어질 때 언본딩 시 DoS 발생

**설명:** `PolygonStrategy`의 [unbond](https://github.com/stakedotlink/contracts/blob/1d3aa1ed6c2fb7920b3dc3d87ece5d1645e1a628/contracts/polygonStaking/PolygonStrategy.sol#L213-L263) 함수는 남은 출금 금액을 충당하기 위해 적은 보상 금액이 사용될 때 DoS에 취약합니다.

[PolygonFundFlowController.unbondVaults](https://github.com/stakedotlink/contracts/blob/1d3aa1ed6c2fb7920b3dc3d87ece5d1645e1a628/contracts/polygonStaking/PolygonFundFlowController.sol#L121)가 호출되면 출금/언본딩할 금액을 `PolygonStrategy.unbond`로 전달합니다:
```solidity
    function unbondVaults() external {
       ...
        uint256 toWithdraw = queuedWithdrawals - (queuedDeposits + validatorRemovalDeposits);
@>        strategy.unbond(toWithdraw);
        timeOfLastUnbond = uint64(block.timestamp);
    }
```

`unbond` 함수는 검증자를 순회하며 그들의 보상을 사용하여 출금 금액을 충당합니다. 보상이 불충분한 경우 검증자의 원금 예치금에서 언본딩합니다.
```solidity
function unbond(uint256 _toUnbond) external onlyFundFlowController {
   ...
    if (rewards >= toUnbondRemaining) {
        // @audit withdrawRewards could be below the threshold from ValidatorShares
@>        vault.withdrawRewards();
        toUnbondRemaining = 0;
    } else {
        toUnbondRemaining -= rewards;
        uint256 vaultToUnbond = principalDeposits >= toUnbondRemaining
            ? toUnbondRemaining
            : principalDeposits;
@>        vault.unbond(vaultToUnbond);
        toUnbondRemaining -= vaultToUnbond;
        ++numVaultsUnbonded;
    }
   ...
}
```

드물지만 일반적인 시나리오는 반복 후 `toUnbondRemaining`에 다음 검증자를 위한 소액이 남는 경우입니다. 이 금액은 검증자의 보상으로 충당할 수 있지만, 해당 보상이 [ValidatorShares](https://etherscan.io/address/0x7e94d6cAbb20114b22a088d828772645f68CC67B#code#F1#L225) 컨트랙트의 임계값 미만인 경우 트랜잭션이 revert되어 `unbond` 프로세스에서 DoS가 발생합니다.

**예시:**
1. 전략에 3명의 검증자가 있습니다.
2. 검증자 3은 보상으로 0.9 토큰을 축적했습니다.
3. 시스템은 총 30.5 토큰을 언본딩해야 합니다.
4. 검증자 1과 2는 언본딩으로 30 토큰을 충당합니다.
5. 검증자 3은 0.9 보상으로 남은 0.5 토큰을 충당할 것으로 예상됩니다 (`withdrawRewards`가 트리거됨).
6. 0.9 토큰이 ValidatorShares의 최소 금액보다 작기 때문에 트랜잭션이 revert됩니다.

현재 로직은 또한 악의적인 사용자가 `unbond` 함수의 보상 계산 방식으로 인해 `unbond` 트랜잭션을 DoS할 수 있도록 허용합니다:

```solidity
uint256 deposits = vault.getTotalDeposits();
uint256 principalDeposits = vault.getPrincipalDeposits();
uint256 rewards = deposits - principalDeposits;
```
`getTotalDeposits`는 `PolygonVault` 컨트랙트의 현재 토큰 잔액을 고려합니다. `toUnbondRemaining`과 검증자의 보상이 볼트의 최소 보상보다 작으면, 공격자는 트랜잭션을 프론트러닝하여 현재 시나리오에서 `unbond`를 revert 시킬 수 있습니다:

1. 공격자가 `PolygonVault`에 먼지(dust) 금액을 보냅니다. `toUnbondRemaining`을 충당하기에 충분하지만 1e18(최소 보상) 미만입니다.
2. `rewards = deposits - principalDeposits` >= `toUnbondRemaining`.
3. 볼트는 `withdrawRewards`를 호출하지만 인출될 현재 청구 금액 < `1e18` ([ValidatorShares](https://etherscan.io/address/0x7e94d6cAbb20114b22a088d828772645f68CC67B#code#F1#L225)에서 revert 트리거).
4. 트랜잭션이 revert됩니다.

**파급력:** 출금을 완료하기 위해 적은 보상(< 1e18)이 필요할 때 언본딩 프로세스에서 DoS가 발생하여 프로토콜의 출금 흐름을 차단합니다.

**개념 증명 (Proof of Concept):**
1. 먼저 보상 인출 시 최소 금액 요구 사항을 추가하여 `PolygonValidatorShareMock`을 `ValidatorShares`와 동일한 동작을 반영하도록 조정합니다.
```diff
// PolygonValidatorShareMock
   function withdrawRewardsPOL() external {
        uint256 rewards = liquidRewards[msg.sender];
        if (rewards == 0) revert NoRewards();
+        require(rewards >= 1e18, "Too small rewards amount");

        delete liquidRewards[msg.sender];
        stakeManager.withdraw(msg.sender, rewards);
    }
```

2. `polygon-fund-flow-controller.test.ts`에 다음 테스트를 붙여넣고 `npx hardhat test test/polygonStaking/polygon-fund-flow-controller.test.ts`를 실행합니다:
```solidity
describe.only('DoS', async () => {
    it('will revert when withdrawRewards < 1e18', async () => {
      const { token, strategy, vaults, fundFlowController, withdrawalPool, validatorShare, validatorShare2, validatorShare3 } =
      await loadFixture(deployFixture)
      console.log("will print some stuff")

      // validator funds: [10, 20, 30]
      await withdrawalPool.setTotalQueuedWithdrawals(toEther(970.5));
      assert.equal(await fundFlowController.shouldUnbondVaults(), true);

      // 1. Pre-condition: validator acumulated dust rewards since last unbonding.
      await validatorShare3.addReward(vaults[2].target, toEther(0.9));

      // expect that vault 2 has rewards
      assert.equal(await vaults[2].getRewards(), toEther(0.9));

      // 2. Unbond:
      // Validator A will cover 10 with unbond
      // Validator B will cover 20 with unbond
      // Validator C will cover the remaining 0.5 with his 0.9 rewards.
      await expect(fundFlowController.unbondVaults()).to.be.revertedWith("Too small rewards amount");
    })
  })
```
출력:
```javascript
PolygonFundFlowController
    DoS
      ✔ will revert when withdrawRewards < 1e18 (800ms)

  1 passing (800ms)
```

**권장되는 완화 방법:** `PolygonStrategy.unbond`에서 `withdrawRewards`를 호출하기 전에 `ValidatorShares`에서 인출할 수 있는 **실제** 보상이 청구 최소 금액(1e18)보다 큰지 확인하십시오.
```diff
-if (rewards >= toUnbondRemaining) {
-    vault.withdrawRewards();
-    toUnbondRemaining = 0;
+if (rewards >= toUnbondRemaining && vault.getRewards() >= vault.minAmount()) {
+    vault.withdrawRewards();
+    toUnbondRemaining = 0;
+} else {
+    if (toUnbondRemaining > rewards) {
+        toUnbondRemaining -= rewards;
+    }
-    toUnbondRemaining -= rewards;
     uint256 vaultToUnbond = principalDeposits >= toUnbondRemaining
         ? toUnbondRemaining
         : principalDeposits;
```


**Stake.Link:** [PR 151](https://github.com/stakedotlink/contracts/pull/151/commits/fec5777db57e3649b27f8cff7ab2fd34b3fb9857)에서 해결되었습니다.

**Cyfrin:** 해결되었습니다.


### `disableInitializers`가 초기화되지 않은 컨트랙트를 방지하는 데 사용되지 않음

**설명:** `PolygonVault` 및 `PolygonStrategy` 컨트랙트는 업그레이드 가능하도록 설계되었습니다.

둘 다 `Initializable`을 상속하지만 `_disableInitializers()`를 호출하는 생성자가 없습니다.

초기화되지 않은 컨트랙트는 공격자에 의해 탈취될 수 있습니다. 이는 프록시와 구현 컨트랙트 모두에 적용되며, 프록시에 영향을 미칠 수 있습니다. 구현 컨트랙트가 사용되는 것을 방지하려면 배포 시 자동으로 잠기도록 생성자에서 `_disableInitializers()`를 호출해야 합니다.

**파급력:** `PolygonVault` 및 `PolygonStrategy`에 `_disableInitializers()`가 있는 생성자가 누락되면 초기화되지 않은 업그레이드 가능한 컨트랙트가 무단으로 탈취될 위험이 있습니다.

**권장되는 완화 방법:** `PolygonVault` 및 `PolygonStrategy`에 `_disableInitializers()`를 명시적으로 호출하는 생성자를 추가하는 것을 고려하십시오.

**Stake.Link:** [PR 151](https://github.com/stakedotlink/contracts/pull/151/commits/7278d0babbf495fb593d1c1e9d827d3d26d94d53)에서 해결되었습니다.

**Cyfrin:** 해결되었습니다.

\clearpage
## 정보성 (Informational)


### PolygonStrategy::setFundFlowController에 대한 이벤트 방출 누락

**설명:** 중요한 전략 매개변수, 즉 `fundFlowController`는 소유자에 의해 업데이트될 수 있습니다. 그러나 그러한 업데이트에 대한 이벤트 방출이 존재하지 않습니다.

**권장되는 완화 방법:** `setFundFlowController` 함수에 대한 이벤트를 방출하는 것을 고려하십시오.

**Stake.Link:** 인지함. `fundFlowController`는 컨트랙트 배포 시 한 번만 설정됩니다.

**Cyfrin:** 인지함.


### `PolygonStrategy::initialize`에서 `validatorMEVRewardsPercentage`에 대한 확인 누락

**설명:** `validatorMEVRewardsPercentage`는 0에서 10000 (100%) 범위의 유효한 값을 가진 보상 비율입니다. `PolygonStrategy::setValidatorMEVRewardsPercentage`는 보상 비율이 10000을 초과하지 않는지 올바르게 확인합니다. 그러나 이 유효성 검사는 초기화 중에 누락되었습니다.

```solidity
    function initialize(
        address _token,
        address _stakingPool,
        address _stakeManager,
        address _vaultImplementation,
        uint256 _validatorMEVRewardsPercentage,
        Fee[] memory _fees
    ) public initializer {
        __Strategy_init(_token, _stakingPool);

        stakeManager = _stakeManager;
        vaultImplementation = _vaultImplementation;
        validatorMEVRewardsPercentage = _validatorMEVRewardsPercentage; //@audit missing check on validatorMEVRewardsPercentage

```

**권장되는 완화 방법:** `validatorMEVRewardsPercentage`가 업데이트되는 모든 곳에서 유효성 검사를 일관되게 만드는 것을 고려하십시오.

**Stake.Link:** [PR 151](https://github.com/stakedotlink/contracts/pull/151/commits/2ef53d4f0dac80c6ba1dc31ec516a28d7ed02851)에서 해결되었습니다.

**Cyfrin:** 해결되었습니다.


### 누락된 0 주소 확인

**설명:** 컨트롤러, 전략 및 볼트 컨트랙트의 중요한 주소에 초기화 시점과 나중에 업데이트 시점에 0 주소 확인이 누락되었습니다.

`PolygonStrategy.sol`
```solidity
    function setValidatorMEVRewardsPool(address _validatorMEVRewardsPool) external onlyOwner {
        validatorMEVRewardsPool = IRewardsPool(_validatorMEVRewardsPool); //@audit missing zero address check
    }

   function setVaultImplementation(address _vaultImplementation) external onlyOwner { //@audit missing zero address check
        vaultImplementation = _vaultImplementation;
        emit SetVaultImplementation(_vaultImplementation);
    }
```

`PolygonFundFlowController.sol`
```solidity
   function setDepositController(address _depositController) external onlyOwner {
        depositController = _depositController; // @audit missing zero address check
    }
```

**권장되는 완화 방법:** 위에 강조 표시된 모든 곳에 0 주소 확인을 도입하는 것을 고려하십시오.

**Stake.Link:** [PR 151](https://github.com/stakedotlink/contracts/pull/151/commits/9889b7b628050dbc791f028aeb8b9ff8b21cd93a)에서 해결되었습니다.

**Cyfrin:** 해결되었습니다.


### `PolygonStrategy::updateFee`를 통해 수수료를 제거할 때 잘못된 이벤트 방출

**설명:** 주어진 인덱스에 대해 `_feeBasisPoints = 0`을 전달하여 수수료를 제거할 수 있습니다.

현재 로직은 인덱스의 수수료를 `fees` 배열의 마지막 인덱스 수수료와 교체한 다음 `fees` 배열의 마지막 요소를 팝(pop)하는 것입니다.

그러나 이 작업에 대해 방출된 이벤트는 인덱스의 수수료가 0으로 업데이트되었음을 시사합니다. 이는 인덱스의 수수료가 원래 `fees` 배열의 마지막 인덱스에 해당하는 수수료이기 때문에 부정확합니다. 또한, 이 시나리오에서 `UpdateFee` 이벤트는 이 수신자가 실제로 제거된 수수료 인덱스의 수신자와 일치하는지 확인하지 않고 함수에 입력으로 전달된 `receiver`를 방출합니다.

```solidity
function updateFee(
        uint256 _index,
        address _receiver,
        uint256 _feeBasisPoints
    ) external onlyOwner {

        if (_feeBasisPoints == 0) {
            fees[_index] = fees[fees.length - 1];
            fees.pop();

        } else {
              // ... code
        }
        emit UpdateFee(_index, _receiver, _feeBasisPoints); //@audit if _feeBasisPoints == 0, the fee for the index is now the fee in the last index (for old array)
    }
```

**권장되는 완화 방법:** `_feeBasisPoints == 0`인 특별한 경우에 대해 `AddFee`와 유사한 `RemoveFee` 이벤트를 추가하는 것을 고려하십시오. 이렇게 하면 인덱스의 수수료가 단순히 업데이트되는 경우와 완전히 제거되는 경우를 명확하게 구분할 수 있습니다.


**Stake.Link:** [PR 151](https://github.com/stakedotlink/contracts/pull/151/commits/bd29a3eee47f41ad9418418d7c959e7372b1f755)에서 해결되었습니다.

**Cyfrin:** 해결되었습니다.


### `PolygonStrategy::addFee`에서 `_feeBasisPoints > 0`에 대한 확인 누락

**설명:** `PolygonStrategy::addFee`는 `fees` 배열에 새 수수료 요소를 추가할 때 `_feeBasisPoints > 0`인지 확인하지 않습니다. 이는 `_feeBasisPoints = 0`이 `fees` 배열에서 수수료 요소를 제거하도록 트리거하는 `PolygonStrategy::updateFee` 로직과 일치하지 않습니다.

**권장되는 완화 방법:** `_feeBasisPoints`에 0이 아닌지 확인하는 로직을 도입하는 것을 고려하십시오.

**Stake.Link:** 인지함. 소유자가 올바른 값이 설정되도록 할 것입니다.

**Cyfrin:** 인지함.


### `ValidatorShares`와 상호 작용할 때 슬리피지 보호 없음

**설명:** `PolygonVault.sol` 컨트랙트에서 `buyVoucherPOL` 및 `sellVoucherPOL` 함수는 슬리피지 제한을 설정하지 않습니다.

```solidity
// PolygonVault
    function deposit(uint256 _amount) external onlyVaultController {
        token.safeTransferFrom(msg.sender, address(this), _amount);

        // @audit-issue - no slippage limit.
@>        validatorPool.buyVoucherPOL(_amount, 0);

        uint256 balance = token.balanceOf(address(this));
        if (balance != 0) token.safeTransfer(msg.sender, balance);
    }

   function unbond(uint256 _amount) external onlyVaultController {
         // @audit-issue no slippage limit
@>         validatorPool.sellVoucherPOL(_amount, type(uint256).max);

        uint256 balance = token.balanceOf(address(this));
        if (balance != 0) token.safeTransfer(msg.sender, balance);
    }
```

현재 `ValidatorShares`에는 활성 슬래싱이 없으므로 자금 손실이 발생할 수 없습니다.

**권장되는 완화 방법:** Polygon PoS 거버넌스의 슬래싱 업데이트를 계속 모니터링하십시오. 구현되면 이에 따라 슬래싱을 지원하도록 컨트랙트를 업그레이드하십시오.


**[Project]:**
인지함. 슬래싱이 도입되면 슬리피지 보호를 구현할 것입니다.

**Cyfrin:** 인지함.


### `PolygonFundFlowController.setMinTimeBetweenUnbonding`에서 입력 유효성 검사 누락

**설명:** `PolygonFundFlowController`의 `setMinTimeBetweenUnbonding` 함수는 `_minTimeBetweenUnbonding` 매개변수에 대한 입력 유효성 검사가 부족하여 0이나 극도로 높은 값을 포함한 모든 값으로 설정할 수 있습니다.
```solidity
    function setMinTimeBetweenUnbonding(uint64 _minTimeBetweenUnbonding) external onlyOwner {
        minTimeBetweenUnbonding = _minTimeBetweenUnbonding;  // @audit missing input validation
        emit SetMinTimeBetweenUnbonding(_minTimeBetweenUnbonding);
    }
```

**권장되는 완화 방법:** `minTimeBetweenUnbonding`을 설정하기 전에 최소/최대 값 확인을 추가하십시오. 예:
```diff
// declare MIN_VALUE and MAX_VALUE with the proper values.
function setMinTimeBetweenUnbonding(uint64 _minTimeBetweenUnbonding) external onlyOwner {
+    require(_minTimeBetweenUnbonding >= MIN_VALUE, "Time between unbonding too low");
+    require(_minTimeBetweenUnbonding <= MAX_VALUE, "Time between unbonding too high");

    minTimeBetweenUnbonding = _minTimeBetweenUnbonding;
    emit SetMinTimeBetweenUnbonding(_minTimeBetweenUnbonding);
}
```

**Stake.Link:** 인지함. 소유자가 올바른 값이 설정되도록 할 것입니다.

**Cyfrin:** 인지함.


### 불충분한 잔액 확인으로 인한 `PolygonStrategy::unbond`의 잠재적 무한 루프

**설명:** `PolygonStrategy::unbond` 함수는 전체 요청된 언본딩 금액(`toUnbondRemaining`)이 처리될 때까지 계속되는 while 루프를 포함합니다. 그러나 모든 볼트에서 사용 가능한 총 잔액이 언본딩 요청을 충족하기에 충분하지 않은 시나리오를 처리하기 위한 보호 장치가 없습니다.

```solidity
function unbond(uint256 _toUnbond) external onlyFundFlowController {
    // ...
    uint256 toUnbondRemaining = _toUnbond;

    // ...
    while (toUnbondRemaining != 0) {
        // Process vaults...
        ++i;
        if (i >= vaults.length) i = 0;
        // No check for complete loop iteration without progress
    }
    // ...
}
```
이 문제는 함수가 모든 볼트에 걸친 총 스테이킹 금액이 항상 언본딩 금액을 충당할 것이라고 가정하기 때문에 발생합니다. 이 가정은 현재 단일 전략 설정에서는 유효할 수 있지만, 특히 다중 전략 환경에서는 보장되지 않습니다.

`unbonding`의 규모를 결정하는 `queuedWithdrawals`를 추적하는 출금 풀은 모든 전략에 걸쳐 글로벌 수준에서 작동한다는 점에 주목할 필요가 있습니다.

**파급력:** 현재 단일 전략 설정에서는 가능성이 낮은 시나리오이지만, 무한 while 루프는 사용 가능한 모든 가스를 소비합니다.

**권장되는 완화 방법:** 루프가 진행 없이 모든 볼트를 반복했는지 감지하는 안전 메커니즘을 추가하여 언본딩 요청을 충족하기에 자금이 불충분함을 나타내는 것을 고려하십시오.

```diff solidity

function unbond(uint256 _toUnbond) external onlyFundFlowController {

        while (toUnbondRemaining != 0) {
             // ... code
             if (i >= vaults.length) i = 0;

++       // Add safety check to prevent infinite loop
++        if (i == startingIndex) {
++            // We've gone through all vaults and still have amount to unbond
++            // Process partial unbonding with what we've got so far OR revert
++           break; // or revert if that's more appropriate
++        }

       }
}

```
**Stake.Link:** [PR 151](https://github.com/stakedotlink/contracts/pull/151/commits/e138c991bc1f41ec1f33ea62b3459870a29c333e)에서 해결되었습니다.

**Cyfrin:** 해결되었습니다.

\clearpage
## 가스 최적화 (Gas Optimization)


### `_feeBasisPoints > 0`인 경우에만 `PolygonStrategy::updateFee`에서 총 수수료 확인

**설명:** `PolygonStrategy::updateFee`에서 수수료를 업데이트하는 동안 `_feeBasisPoints == 0`이면 인덱스의 수수료가 배열의 마지막 인덱스 수수료로 대체되고 마지막 요소가 배열에서 팝(pop)됩니다.

수수료는 음수가 아닌 값이므로 이 시나리오에서는 총 수수료 확인을 수행할 필요가 없습니다. 수수료 배열을 루핑하는 것을 포함하므로 확인이 가스를 많이 소모한다는 점을 강조할 가치가 있습니다.

```solidity
if (_totalFeesBasisPoints() > 3000) revert FeesTooLarge();
```

**권장되는 완화 방법:** 확인을 `else` 블록으로 이동하는 것을 고려하십시오.

**Stake.Link:** [PR 151](https://github.com/stakedotlink/contracts/pull/151/commits/44a8f032e5ae9697440f3b7d83a10538ab88e877)에서 해결되었습니다.

**Cyfrin:** 해결되었습니다.


### `PolygonStrategy::unbond()` 함수의 비효율적인 보상 수집 패턴으로 인한 불필요한 가스 소비

**설명:** `PolygonStrategy::unbond()` 함수의 현재 구현은 볼트를 순차적으로 처리하고 볼트별로 즉각적인 언본딩 결정을 내립니다.

이 접근 방식은 모든 볼트의 총 보상이 출금 금액을 충당하기에 충분할 때 불필요한 언본딩 작업으로 이어질 수 있습니다.

현재 구현에서는:

1. 함수가 각 볼트를 반복합니다.
2. 각 볼트에 대해 가능한 경우 보상을 추출합니다.
3. 단일 볼트의 보상이 불충분한 경우 해당 볼트에서 즉시 원금을 언본딩합니다.
4. 전체 `_toUnbond` 금액이 충족될 때까지 이 프로세스를 계속합니다.

이 구현은 언본딩 결정을 내리기 전에 모든 볼트에서 사용 가능한 총 보상을 고려하지 않아 상당한 가스를 소비하는 잠재적인 불필요한 언본딩 작업을 초래합니다.

**파급력:**
- 더 높은 가스 비용: 각 불필요한 언본딩 작업은 추가 가스를 소비하며, 특히 나중에 출금을 완료하기 위해 별도의 `unstakeClaim` 호출이 필요하므로 비용이 더 많이 듭니다.

- 자본 비효율성: 보상이 충분했을 때 원금을 언본딩하면 프로토콜에서 수익을 얻는 자본의 양이 줄어듭니다.

- 더 긴 대기 기간: 보상과 달리 언본딩된 원금은 재사용하거나 인출하기 전에 대기 기간이 적용됩니다.

**권장되는 완화 방법:** `unbond()` 함수를 2단계 접근 방식을 사용하도록 리팩터링하십시오:
1. 누적 보상이 언본딩 요청을 충당하기에 충분할 때까지 모든 보상을 수집합니다.
2. 모든 보상이 수집된 후에만 볼트 언본딩을 시작합니다.

샘플 구현은 다음과 같습니다:

```solidity
function unbond(uint256 _toUnbond) external onlyFundFlowController {
    if (numVaultsUnbonding != 0) revert UnbondingInProgress();
    if (_toUnbond == 0) revert InvalidAmount();

    uint256 toUnbondRemaining = _toUnbond;
    uint256 preBalance = token.balanceOf(address(this));
    uint256 skipIndex = validatorRemoval.isActive ? validatorRemoval.validatorId : type(uint256).max;
    uint256 numVaultsUnbonded;

    // @audit Phase 1 -> Withdraw all rewards first
    for (uint256 i = 0; i < vaults.length; i++) {
        if (i != skipIndex) {
            IPolygonVault vault = vaults[i];
            uint256 rewards = vault.getRewards();

            if (rewards > 0) {
                vault.withdrawRewards();

                // @audit Check if we've collected enough rewards
                uint256 currentBalance = token.balanceOf(address(this));
                uint256 collectedRewards = currentBalance - preBalance;
                if (collectedRewards >= toUnbondRemaining) {
                    // @audit We've collected enough rewards, no need to unbond principal
                    totalQueued += collectedRewards;
                    emit Unbond(_toUnbond);
                    return;
                }
            }
        }
    }

    // @audit Phase 2: If rewards weren't enough, unbond principal
    uint256 rewardsCollected = token.balanceOf(address(this)) - preBalance;
    toUnbondRemaining -= rewardsCollected;

    uint256 i = validatorWithdrawalIndex;
    while (toUnbondRemaining != 0) {
        if (i != skipIndex) {
            IPolygonVault vault = vaults[i];
            uint256 principalDeposits = vault.getPrincipalDeposits();

            if (principalDeposits != 0) {
                uint256 vaultToUnbond = principalDeposits >= toUnbondRemaining
                    ? toUnbondRemaining
                    : principalDeposits;

                vault.unbond(vaultToUnbond);
                toUnbondRemaining -= vaultToUnbond;
                ++numVaultsUnbonded;
            }
        }

        ++i;
        if (i >= vaults.length) i = 0;
    }

    validatorWithdrawalIndex = i;
    numVaultsUnbonding = numVaultsUnbonded;
    totalQueued += token.balanceOf(address(this)) - preBalance;

    emit Unbond(_toUnbond);
}
```

**Stake.Link:** 인지함.

**Cyfrin:** 인지함.

