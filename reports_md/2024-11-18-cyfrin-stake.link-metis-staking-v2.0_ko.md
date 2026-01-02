**Lead Auditors**

[0kage](https://twitter.com/0kage_eth)

[Hans](https://twitter.com/hansfriese)

**Assisting Auditors**



---

# Findings
## High Risk


### L1 CCIP messages use incorrect `tokensInTransitToL1` value leading to overvalued LST on Metis

**Description:** L1과 L2 체인 간의 인출 회계 불일치로 인해 스테이킹 시스템의 주식 가격(share price)이 인위적으로 부풀려질 수 있습니다.

`L1Transmitter::_executeUpdate` 함수에서 `L1Strategy`로부터 실제 인출된 금액은 `toWithdraw`이지만, CCIP 메시지는 전체 `queuedWithdrawals`가 인출되었다고 가정합니다. `queuedWithdrawals > canWithdraw`인 시나리오에서 CCIP 메시지는 `tokensInTransitToL1`에 대해 부풀려진 값을 보냅니다.

```solidity
//L1Transmitter.sol
function _executeUpdate() private {
    // ...
    uint256 toWithdraw = queuedWithdrawals > canWithdraw ? canWithdraw : queuedWithdrawals;
    if (toWithdraw > minWithdrawalThreshold) { //@audit only when amount > min withdrawal
        l1Strategy.withdraw(toWithdraw); //@audit actual amount withdrawn is toWithdraw
        // ... (withdrawal logic)
    }

    Client.EVM2AnyMessage memory evm2AnyMessage = _buildCCIPUpdateMessage(
        totalDeposits,
        claimedRewards + queuedWithdrawals,  // @audit This uses full queuedWithdrawals, not actual withdrawn amount
        depositsSinceLastUpdate,
        opRewardReceivers,
        opRewardAmounts
    );
    // ...
}
```


L2Transmitter.sol에서, 부풀려진 인출 금액(`tokensInTransitFromL1`)이 검증 없이 수락됩니다:

```solidity
//L2Transmitter.sol
function _ccipReceive(Client.Any2EVMMessage memory _message) internal override {
    // ...
    (
        uint256 totalDeposits,
        uint256 tokensInTransitFromL1,
        uint256 tokensReceivedAtL1,
        address[] memory opRewardReceivers,
        uint256[] memory opRewardAmounts
    ) = abi.decode(_message.data, (uint256, uint256, uint256, address[], uint256[]));

    l2Strategy.handleUpdateFromL1(
        totalDeposits,
        tokensInTransitFromL1,  //@audit This value could be inflated
        tokensReceivedAtL1,
        opRewardReceivers,
        opRewardAmounts
    );
    // ...
}
```

3. L2Strategy.sol에서, 부풀려진 tokensInTransitFromL1은 예금 변경 계산을 부풀립니다:
```solidity
function getDepositChange() public view returns (int) {
    return
        int256(
            l1TotalDeposits +
                tokensInTransitToL1 +
                tokensInTransitFromL1 +  //@audit This value could be inflated
                token.balanceOf(address(this))
        ) - int256(totalDeposits);
}
```
4. 마지막으로 StakingPool.sol에서, 부풀려진 예금 변경은 totalRewards와 주식 가격의 인위적인 증가로 이어집니다:
```solidity
function _updateStrategyRewards(uint256[] memory _strategyIdxs, bytes memory _data) private {
    int256 totalRewards;
    // ...
    for (uint256 i = 0; i < _strategyIdxs.length; ++i) {
        IStrategy strategy = IStrategy(strategies[_strategyIdxs[i]]);
        (int256 depositChange, , ) = strategy.updateDeposits(_data);
        totalRewards += depositChange;
    }

    if (totalRewards != 0) {
        totalStaked = uint256(int256(totalStaked) + totalRewards);
    }
    // ...
}
```


**Impact:** 이 취약점은 유동성 스테이킹 토큰(liquid staking token)의 주식 가격을 부풀리게 합니다.

**Recommended Mitigation:** L1에서 실제 인출된 금액을 정확하게 추적하십시오:

```solidity
uint256 actualWithdrawn = 0;
if (toWithdraw > minWithdrawalThreshold) {
    l1Strategy.withdraw(toWithdraw);
    actualWithdrawn = toWithdraw;
    // ... (rest of withdrawal logic)
}
```
L2로 보내는 CCIP 메시지에 실제 인출된 금액을 사용하십시오:
```solidity
Client.EVM2AnyMessage memory evm2AnyMessage = _buildCCIPUpdateMessage(
    totalDeposits,
    claimedRewards + actualWithdrawn,
    depositsSinceLastUpdate,
    opRewardReceivers,
    opRewardAmounts
);
```

**Stake.Link:** [6a42a8c](https://github.com/stakedotlink/contracts/commit/6a42a8c5d461422333eba1eed8cd57c560603ab7)에서 수정됨.

**Cyfrin:** 확인됨.


### Slashing loss redistribution vulnerability allows existing depositors to avoid losses at new depositors' expense

**Description:** 프로토콜의 스테이킹 시스템은 L1(이더리움)과 L2(메티스)에 걸쳐 있으며 상태 업데이트는 Chainlink CCIP를 통해 전파됩니다. 현재 설계는 메티스에서의 슬래싱(slashing) 이벤트가 L2의 주식 가격 계산에 즉시 반영되지 않는 타이밍 취약점에 노출되어 있어, 초기 예금자가 신규 예금자의 비용으로 부풀려진 주식 가격에 퇴장할 수 있습니다.

핵심 문제는 L2Strategy의 총 예금 계산이 CCIP 업데이트가 수신될 때까지 오래된 L1 상태에 의존한다는 것입니다:

```solidity
// L2Strategy.sol
function getTotalDeposits() public view override returns (uint256) {
    return l1TotalDeposits + tokensInTransitToL1 + tokensInTransitFromL1 + token.balanceOf(address(this));
}
```
메티스에서 슬래싱이 발생하면 L1Strategy.getDepositChange()에서는 보이지만 CCIP를 통해 updateDeposits가 호출될 때까지 totalDeposits에는 반영되지 않습니다:

```solidity
// L1Strategy.sol
function getDepositChange() public view returns (int) {
    uint256 totalBalance = token.balanceOf(address(this));
    for (uint256 i = 0; i < vaults.length; ++i) {
        totalBalance += vaults[i].getTotalDeposits();
    }
    return int(totalBalance) - int(totalDeposits);
}
```
이는 다음과 같은 잠재적 시나리오로 이어질 수 있습니다:

-> 메티스에서 슬래싱 발생 및 L1Strategy.getDepositChange()에서 확인 가능
-> L2의 신규 예금은 부풀려진 가격으로 주식을 받음(오래된 L1 상태 사용)
-> 초기 예금자는 슬래싱 이전 가치로 이 새로운 유동성을 사용하여 인출 가능
-> 슬래싱이 CCIP를 통해 최종적으로 반영되면, 마지막 예금자가 손실의 대부분을 부담함

**Impact:**
- 신규 예금자는 L1에서 슬래싱 후 입금했더라도 마땅히 받아야 할 것보다 적은 주식을 받을 수 있습니다.
- 프론트 러닝(front-running) 슬래싱은 기존 예금자가 신규 예금자가 생성한 유동성을 사용하여 퇴장할 수 있게 합니다. 결과적으로 불균형적인 슬래싱 영향이 최근 예금자에게 떨어집니다.

**Proof of Concept:** 다음 테스트를 실행하십시오. 이 테스트를 위해 PriorityPool에서 `allowInstantWithdrawals`를 `true`로 설정하십시오.

```solidity
  it('front running on metis', async () => {
    const {
      signers,
      accounts,
      l2Strategy,
      l2Metistoken,
      l2Transmitter,
      stakingPool,
      priorityPool,
      metisLockingInfo,
      metisLockingPool,
      l1Metistoken,
      l1Transmitter,
      l1Strategy,
      vaults,
      offRamp,
    } = await loadFixture(deployFixture)

    // // setup Bob and Alice
    const bob = signers[7]
    const bobAddress = accounts[7]
    const alice = signers[8]
    const aliceAddress = accounts[8]
    const initialDeposit = toEther(1000)

    await l2Metistoken.transfer(bobAddress, initialDeposit)
    await l2Metistoken.connect(bob).approve(priorityPool.target, ethers.MaxUint256)
    await stakingPool.connect(bob).approve(priorityPool.target, ethers.MaxUint256)

    // Bob makes initial deposit
    await priorityPool.connect(bob).deposit(initialDeposit, false, ['0x'])
    assert.equal(await fromEther(await l2Strategy.getTotalDeposits()), fromEther(initialDeposit))
    assert.equal(
      await fromEther(await l2Strategy.getTotalQueuedTokens()),
      fromEther(initialDeposit)
    )

    // executeUpdate to send message and tokens to L1
    await l2Transmitter.executeUpdate({ value: toEther(10) })
    assert.equal(await fromEther(await l2Strategy.getTotalDeposits()), fromEther(initialDeposit))
    assert.equal(await fromEther(await l2Strategy.getTotalQueuedTokens()), 0)
    assert.equal(await fromEther(await l2Metistoken.balanceOf(l2Strategy.target)), 0)
    assert.equal(await fromEther(await l2Strategy.tokensInTransitToL1()), fromEther(initialDeposit))

    // deposit the tokens received form L2 in L1
    await l1Metistoken.transfer(l1Transmitter.target, initialDeposit) // mock the deposit via L2 Bridge
    await l1Transmitter.depositTokensFromL2() // deposits balance into L1 Strategy
    await l1Transmitter.depositQueuedTokens([1], [initialDeposit]) // deposits balance into vault 1

    let vault1 = await ethers.getContractAt('SequencerVault', (await l1Strategy.getVaults())[1])
    assert.equal(await fromEther(await vault1.getPrincipalDeposits()), fromEther(initialDeposit))

    // Confirm initial deposit via CCIP back to L2
    await offRamp
      .connect(signers[5])
      .executeSingleMessage(
        ethers.encodeBytes32String('messageId'),
        777,
        ethers.AbiCoder.defaultAbiCoder().encode(
          ['uint256', 'uint256', 'uint256', 'address[]', 'uint256[]'],
          [initialDeposit, 0, initialDeposit, [], []]
        ),
        l2Transmitter.target,
        []
      )

    assert.equal(fromEther(await l2Strategy.l1TotalDeposits()), fromEther(initialDeposit))
    assert.equal(fromEther(await l2Strategy.tokensInTransitFromL1()), 0)
    assert.equal(fromEther(await l2Strategy.tokensInTransitToL1()), 0)
    assert.equal(fromEther(await l2Strategy.getTotalDeposits()), fromEther(initialDeposit))
    assert.equal(fromEther(await stakingPool.totalStaked()), fromEther(initialDeposit))

    assert.equal(fromEther(await stakingPool.balanceOf(bobAddress)), fromEther(initialDeposit))

    // simulate 20% slashing on Metis
    const slashAmount = toEther(200) // 20% of 1000
    await metisLockingPool.slashPrincipal(1, slashAmount)

    assert.equal(fromEther(await l1Strategy.getDepositChange()), -200)

    // At this point share price should still reflect 1000 tokens since slash isn't reflected
    // Alice deposits after slash but before it's reflected
    const aliceDeposit = toEther(1000)
    await l2Metistoken.transfer(alice.address, aliceDeposit)
    await l2Metistoken.connect(alice).approve(priorityPool.target, ethers.MaxUint256)
    await stakingPool.connect(alice).approve(priorityPool.target, ethers.MaxUint256)

    // Alice deposits at inflated share price
    await priorityPool.connect(alice).deposit(aliceDeposit, false, ['0x'])

    assert.equal(
      fromEther(await stakingPool.totalStaked()),
      fromEther(initialDeposit) + fromEther(aliceDeposit)
    )

    assert.equal(
      fromEther(await l2Strategy.getTotalDeposits()),
      fromEther(initialDeposit) + fromEther(aliceDeposit)
    )
    assert.equal(fromEther(await stakingPool.balanceOf(aliceAddress)), fromEther(aliceDeposit)) //1:1 conversion even though there is slashing
    assert.equal(fromEther(await l2Strategy.getTotalQueuedTokens()), fromEther(aliceDeposit))
    assert.equal(
      fromEther(await l2Metistoken.balanceOf(l2Strategy.target)),
      fromEther(aliceDeposit)
    )

    // Bob sees the slashing on L2 and places a withdrawal request
    const bobBalanceBefore = await l2Metistoken.balanceOf(bobAddress)
    const bobWithdrawAmount = await stakingPool.balanceOf(bobAddress) // try to redeem all of Bob's shares for metis token

    await priorityPool.connect(bob).withdraw(
      bobWithdrawAmount,
      0,
      0,
      [],
      false,
      false, // Don't queue, withdraw instantly from available liquidity
      ['0x']
    )

    // Process Bob's withdrawal using Alice's new deposits
    const bobBalanceAfter = await l2Metistoken.balanceOf(bob.address)
    assert.equal(fromEther(bobBalanceAfter - bobBalanceBefore), fromEther(bobWithdrawAmount)) //full withdrawal even after slashing

    // Finally CCIP message arrives reflecting the slash
    await offRamp.connect(signers[5]).executeSingleMessage(
      ethers.encodeBytes32String('messageId2'),
      777,
      ethers.AbiCoder.defaultAbiCoder().encode(
        ['uint256', 'uint256', 'uint256', 'address[]', 'uint256[]'],
        [toEther(800), 0, 0, [], []] // Now reflects the slash
      ),
      l2Transmitter.target,
      []
    )

    // Calculate Alice's actual value vs expected
    const aliceShareBalance = fromEther(await stakingPool.sharesOf(alice.address))
    const aliceCurrentValueOfShares = fromEther(await stakingPool.balanceOf(alice.address))
    assert.equal(aliceShareBalance, 1000)
    assert.equal(aliceCurrentValueOfShares, 800)
  })
```

**Recommended Mitigation:** 다음 권장 사항을 고려하십시오:
- L2Strategy에 대해 PriorityPool에서의 즉시 인출을 금지하십시오.
- `slashed`라는 새로운 소유자 제어 부울 상태 변수를 추가하십시오. 소유자는 슬래싱 이벤트가 발생하자마자 이 변수를 true로 업데이트하고 L2에서 CCIP 업데이트가 수신되면 `false`로 재설정합니다. `slashed = true`인 경우, `L2Strategy`의 `canDeposit` 및 `canWithdraw` 함수를 다음과 같이 재정의하여 CCIP 업데이트가 있을 때까지 전략에 대한 모든 입금 및 출금을 방지하십시오:

```solidity
  // @audit override in L2Strategy.sol
  function canWithdraw() public view override returns (uint256) {
        if(slashed) return 0; //@audit prevent withdrawals until slashing is updated on L2
        super.canWithdraw()
    }

    function canDeposit() public view override returns (uint256) {
        if(slashed) return 0; //@audit prevent deposits until slashing is updated on L2
        super.canDeposit()
    }
```

**Stake.Link**
인지됨. 슬래싱이 감지되는 즉시 PriorityPool.sol 일시 중지 로직을 사용할 것입니다.

**Cyfrin**
인지됨. 우리의 권장 사항은 "전략(strategy)" 수준이었던 반면 제안된 완화 조치는 "풀(pool)" 수준입니다. 여러 전략이 있는 경우 제안된 완화 조치는 모든 전략에 걸쳐 예금을 일시 중지한다는 점에 주목할 가치가 있습니다.


### Attacker can manipulate queued withdrawal execution timing on withdrawal pool to prevent withdrawals on L1 indefinitely

**Description:** 프로토콜의 `L2Transmitter::executeUpdate` 함수는 인출 타이밍 제어 조작을 통해 차단될 수 있습니다. 이 취약점은 `WithdrawalPool::performUpkeep`에 대한 접근 제어 부족과 인출 및 L2 transmitter 업데이트 간의 공유된 타이밍 제한으로 인해 존재합니다.

핵심 문제는 다음 함수들 간의 상호 작용에서 발생합니다:
```solidity
// L2Transmitter.sol
function executeQueuedWithdrawals() public {
        uint256 queuedTokens = l2Strategy.getTotalQueuedTokens();
        uint256 queuedWithdrawals = withdrawalPool.getTotalQueuedWithdrawals();
        uint256 toWithdraw = MathUpgradeable.min(queuedTokens, queuedWithdrawals);

        if (toWithdraw == 0) revert CannotExecuteWithdrawals();

        bytes[] memory args = new bytes[](1);
        args[0] = "0x";
        withdrawalPool.performUpkeep(abi.encode(args)); //@audit attacker can front-run this to block execute update
    }

function executeUpdate() external payable {
    if (block.timestamp < timeOfLastUpdate + minTimeBetweenUpdates) { //@audit can only run this once every minTimeBetweenUpdates
        revert InsufficientTimeElapsed();
    }

    if (queuedTokens != 0 && queuedWithdrawals != 0) {
        executeQueuedWithdrawals();  // This calls WithdrawalPool.performUpkeep()
        queuedTokens = l2Strategy.getTotalQueuedTokens();
        queuedWithdrawals = withdrawalPool.getTotalQueuedWithdrawals();
    }
    // ... CCIP processing ...
}

// WithdrawalPool.sol
function performUpkeep(bytes calldata _performData) external {
    uint256 canWithdraw = priorityPool.canWithdraw(address(this), 0);
    uint256 totalQueued = _getStakeByShares(totalQueuedShareWithdrawals);
    if (
        totalQueued == 0 ||
        canWithdraw == 0 ||
        block.timestamp <= timeOfLastWithdrawal + minTimeBetweenWithdrawals
    ) revert NoUpkeepNeeded();

    timeOfLastWithdrawal = uint64(block.timestamp);
    // ... withdrawal processing ...
}
```

공격자는 다음을 수행하여 `L2Transmitter::executeUpdate`를 차단할 수 있습니다:

-> `minTimeBetweenWithdrawals`가 경과할 때까지 대기
-> 시간이 경과한 직후 `L2Transmitter::executeQueuedWithdrawals()` 호출
-> `queuedWithdrawals`가 0이 되면: `queuedWithdrawals > 0`을 복원하기 위해 최소 인출을 대기열에 추가

최종적으로 `L2Transmitter::executeUpdate`가 호출될 때, `queuedTokens`와 `queuedWithdrawals`가 모두 0이 아니므로 `executeQueuedWithdrawals`를 통해 인출을 처리하려고 시도합니다. 이는 `minTimeBetweenWithdrawals`가 경과하지 않았기 때문에 실패하며 전체 `executeUpdate`를 되돌립니다(revert).


**Impact:** 이 공격은 CCIP 업데이트 처리를 지속적으로 방지하기 위해 반복될 수 있습니다. 이는 잠재적으로 L1에서의 인출을 무기한 방지할 수 있습니다.

**Proof of Concept:** 인출 풀에 대한 `minTimeBetweenWithdrawals`를 50,000으로, L2Transmitter에 대한 `minTimeBetweenUpdates`를 86400으로 설정하여 다음 POC를 실행하십시오. 또한 PriorityPool에서 `allowInstantWithdrawals`를 false로 설정하십시오.

```solidity
  it('DOS executeUpdate on L1Transmitter', async () => {
    const {
      signers,
      accounts,
      l2Strategy,
      l2Metistoken,
      l2Transmitter,
      stakingPool,
      priorityPool,
      withdrawalPool,
      offRamp,
    } = await loadFixture(deployFixture)

    // // setup Bob and Alice
    const bob = signers[7]
    const bobAddress = accounts[7]
    const alice = signers[8]
    const aliceAddress = accounts[8]

    // 1.0 Fund Alice and Bob token and give necessary approvals
    await l2Metistoken.transfer(bobAddress, toEther(100))
    await l2Metistoken.connect(bob).approve(priorityPool.target, ethers.MaxUint256)
    await stakingPool.connect(bob).approve(priorityPool.target, ethers.MaxUint256)

    await l2Metistoken.transfer(aliceAddress, toEther(100))
    await l2Metistoken.connect(alice).approve(priorityPool.target, ethers.MaxUint256)
    await stakingPool.connect(alice).approve(priorityPool.target, ethers.MaxUint256)

    // 2.0 Bob deposits 50 METIS and calls executeUpdate on L2Transmitter
    await priorityPool.connect(bob).deposit(toEther(50), false, ['0x'])
    assert.equal(fromEther(await l2Strategy.getTotalDeposits()), 50)
    assert.equal(fromEther(await l2Strategy.getTotalQueuedTokens()), 50)

    await l2Transmitter.executeUpdate({ value: toEther(10) })

    assert.equal(fromEther(await l2Strategy.getTotalDeposits()), 50)
    assert.equal(fromEther(await l2Strategy.tokensInTransitToL1()), 50)

    // 3.0 Assume tokens are invested in sequencer vaults on L1 and a message is returned
    await offRamp
      .connect(signers[5]) //@note executes message on L2
      .executeSingleMessage(
        ethers.encodeBytes32String('messageId'),
        777,
        ethers.AbiCoder.defaultAbiCoder().encode(
          ['uint256', 'uint256', 'uint256', 'address[]', 'uint256[]'],
          [toEther(50), 0, toEther(50), [], []]
        ),
        l2Transmitter.target,
        []
      )
    assert.equal(fromEther(await l2Strategy.getTotalDeposits()), 50)
    assert.equal(fromEther(await l2Strategy.tokensInTransitToL1()), 0)
    assert.equal(fromEther(await l2Strategy.tokensInTransitFromL1()), 0)
    assert.equal(fromEther(await l2Strategy.l1TotalDeposits()), 50)

    // 4.0 At this point Alice deposits 25 metis
    await priorityPool.connect(alice).deposit(toEther(25), false, ['0x'])

    assert.equal(fromEther(await l2Strategy.getTotalDeposits()), 75)
    assert.equal(fromEther(await l2Strategy.getTotalQueuedTokens()), 25)
    assert.equal(fromEther(await l2Metistoken.balanceOf(l2Strategy.target)), 25)

    // 5.0 Right before executeUpdate is called on L2Transmitter, Bob withdraws 10 metis
    await priorityPool.connect(bob).withdraw(
      toEther(10),
      0,
      0,
      [],
      false,
      true, // queue withdrawal
      ['0x']
    )

    assert.equal(fromEther(await withdrawalPool.getTotalQueuedWithdrawals()), 10)

    // 6.0 Bob then calls executeQueuedWithdrawals
    await time.increase(50001) // 50000 seconds is min time between withdrawals
    await l2Transmitter.executeQueuedWithdrawals()

    // At this point queued withdrawals is 5, queued tokens is 0
    assert.equal(fromEther(await l2Strategy.getTotalQueuedTokens()), 15)
    assert.equal(fromEther(await withdrawalPool.getTotalQueuedWithdrawals()), 0)

    // // 7.0 Bob again places a 10 metis withdrawal
    await priorityPool.connect(bob).withdraw(
      toEther(10),
      0,
      0,
      [],
      false,
      true, // queue withdrawal
      ['0x']
    )
    assert.equal(fromEther(await l2Strategy.getTotalQueuedTokens()), 15)
    assert.equal(fromEther(await withdrawalPool.getTotalQueuedWithdrawals()), 10)

    await time.increase(36401) // 50001 + 36401 > 86400. So from the initial time, we are now 86401 seconds
    // we should be able to again run execute update

    await expect(l2Transmitter.executeUpdate({ value: toEther(10) })).to.be.revertedWithCustomError(
      withdrawalPool,
      'NoUpkeepNeeded'
    )
  })
```

**Recommended Mitigation:** `L2Transmitter::executeQueuedWithdrawals`에서, `WithdrawalPool::checkUpkeep`이 `true`일 때만 `WithdrawalPool::performUpkeep`을 호출하는 것을 고려하십시오.

**Stake.Link**
[835ed4e](https://github.com/stakedotlink/contracts/commit/835ed4e2d1695ef2e926b71caa724af28fc8f60c)에서 수정됨.

**Cyfrin**
확인됨.

\clearpage
## Medium Risk


### Updating `operatorRewardPercentage` incorrectly redistributes already accrued rewards

**Description:** L1Strategy의 `operatorRewardPercentage`는 보상이 운영자와 다른 수령인 간에 어떻게 분할되는지를 결정합니다. 이 비율이 변경되면, 이전에 생성되었지만 배포되지 않은 보상은 생성될 당시의 비율이 아니라 새로운 비율을 사용하여 계산됩니다.

보상이 계산되는 `SequencerVault.sol`의 주요 코드:

```solidity
// SequencerVault.sol
function updateDeposits(
    uint256 _minRewards,
    uint32 _l2Gas
) external payable onlyVaultController returns (uint256, uint256, uint256) {
    // Calculate total deposits and changes
    uint256 principal = getPrincipalDeposits();
    uint256 rewards = getRewards();
    uint256 totalDeposits = principal + rewards;
    int256 depositChange = int256(totalDeposits) - int256(uint256(trackedTotalDeposits));

    uint256 opRewards;
    if (depositChange > 0) {
        // Uses current operatorRewardPercentage for all accrued rewards
        opRewards = (uint256(depositChange) * vaultController.operatorRewardPercentage()) / 10000;
        trackedTotalDeposits = SafeCastUpgradeable.toUint128(totalDeposits);
    }
    //...
}
```
다음 시나리오를 고려해 보십시오:

- operatorRewardPercentage = 10%
- 100 토큰의 보상 발생 (운영자에게 10, 타인에게 90)
- operatorRewardPercentage가 25%로 변경됨
- 다음 업데이트에서 동일한 100 토큰이 분할됨: 운영자에게 25, 타인에게 75
- 수령인은 효과적으로 벌어들인 15 토큰을 잃음


**Impact:** _비율 상향 조정_: 다른 수령인들이 이미 벌어들인 보상을 잃음
_비율 하향 조정_: 운영자들이 이미 벌어들인 보상을 잃음


**Recommended Mitigation:** `operatorRewardPercentage`를 새로운 값으로 업데이트하기 전에 현재 비율을 사용하여 이미 벌어들인 보상(있는 경우)을 분배하는 것을 고려하십시오. 이는 Strategy의 예금을 업데이트(결과적으로 SequencerVaults의 예금 업데이트)하고, 벌어들인 보상에 해당하는 주식을 민팅하기 위해 L2에 업데이트를 보내는 것을 포함합니다.

**Stake.Link**
[7c5d0aa](https://github.com/stakedotlink/contracts/commit/7c5d0aa923cecd705ca8e52888859ef9a71287d9)에서 수정됨.


**Cyfrin**
확인됨. 운영자 보상 비율을 업데이트하기 전에 `executeUpdate`를 실행하도록 명시적인 인라인 주석이 추가되었습니다.


### Incorrect logic causes that SequencerVaults to not behave as expected when relocking or claiming rewards.

**Description:** SequencerVault.updateDeposits()에 논리 오류가 있어 보상이 함수의 문서화된 동작에 따라 처리되지 않습니다. 아래의 함수 인라인 주석에 따르면 주석 `(set 0 to skip reward claiming)`은 minRewards=0일 때 함수가 청구를 건너뛰어야 하지만 여전히 보상의 재잠금(re-locking)을 처리해야 함을 나타냅니다.

```solidity
//SequencerVault.sol
/**
 * @notice Updates the deposit and reward accounting for this vault
 * @dev will only pay out rewards if the vault is net positive when accounting for lost deposits
 * @param _minRewards min amount of rewards to relock/claim (set 0 to skip reward claiming) --> @audit
 * @param _l2Gas L2 gasLimit for bridging rewards
 * @return the current total deposits in this vault
 * @return the operator rewards earned by this vault since the last update
 * @return the rewards that were claimed in this update
 */
 function updateDeposits(
        uint256 _minRewards,
        uint32 _l2Gas
    ) external payable onlyVaultController returns (uint256, uint256, uint256) {
 // logic
}
```
그러나 실제 구현은:

```solidity
//SequencerVault.sol
function updateDeposits(
    uint256 _minRewards,
    uint32 _l2Gas
) external payable onlyVaultController returns (uint256, uint256, uint256) {
    uint256 principal = getPrincipalDeposits();
    uint256 rewards = getRewards();
    uint256 totalDeposits = principal + rewards;

    // ... code

    // Incorrectly skips all reward processing if minRewards=0
    if (_minRewards != 0 && rewards >= _minRewards) {
        if (
            (principal + rewards) <= vaultController.getVaultDepositMax() &&
            exitDelayEndTime == 0
        ) {
            lockingPool.relock(seqId, 0, true);
        } else {
            lockingPool.withdrawRewards{value: msg.value}(seqId, _l2Gas);
            trackedTotalDeposits -= SafeCastUpgradeable.toUint128(rewards);
            totalDeposits -= rewards;
            claimedRewards = rewards;
        }
    }
    // @audit No else block to handle relocking when minRewards=0

   // code..
    return (totalDeposits, opRewards, claimedRewards);
}
```

구현이 의도된 동작과 다릅니다:

`minRewards=0`일 때:
예상: 청구 건너뛰기, 재잠금 처리
실제: 모든 보상 처리 건너뛰기

**Impact:** `minRewards == 0`일 때 보상은 재잠금되거나 청구되지 않습니다. 이는 시퀀서가 활성화될 때까지 효과적으로 `MetisLockingPool`에 보상이 갇히게 합니다.

**Proof of Concept:** 다음을 `l1-strategy.test.ts`에 추가하십시오

```solidity
 const setupSequencerWithRewards = async () => {
    const { strategy, metisLockingPool, token, accounts } = await loadFixture(deployFixture)

    // Setup vault and deposit initial amount
    await strategy.addVault('0x5555', accounts[1], accounts[2])
    const vaults = await strategy.getVaults()
    const vault0 = await ethers.getContractAt('SequencerVault', vaults[0])

    await strategy.deposit(toEther(500))

    await strategy.depositQueuedTokens([0], [toEther(500)])
    assert.equal(fromEther(await vault0.getPrincipalDeposits()), 500)
    assert.equal(fromEther(await vault0.getTotalDeposits()), 500)

    // Add rewards to sequencer
    await metisLockingPool.addReward(1, toEther(50))
    assert.equal(fromEther(await vault0.getTotalDeposits()), 550)

    return { strategy, vault0, metisLockingPool }
  }

  it('rewards not processed when minRewardsToClaim is 0', async () => {
    const { strategy, vault0, metisLockingPool } = await setupSequencerWithRewards()

    // Set minRewardsToClaim to 0
    await strategy.setMinRewardsToClaim(0)

    const rewardsBefore = await vault0.getRewards()
    assert.equal(fromEther(rewardsBefore), 50)

    // Update deposits
    await strategy.updateDeposits(0, 0)

    // Rewards should be relocked but aren't
    const rewardsAfter = await vault0.getRewards()
    assert.equal(fromEther(rewardsAfter), 50)
  })
```


**Recommended Mitigation:** 보상 관리와 관련된 현재 로직 업데이트를 고려하십시오:
- `minRewards > 0` && 보상이 최소 임계값을 초과할 때 보상이 청구되어야 합니다!
- `minRewards == 0`일 때 보상은 재잠금되어야 합니다.

**Stake.Link**
커밋 [0ec7685](https://github.com/stakedotlink/contracts/commit/0ec7685bad34e28c680518c5fea6b46d2af24e7d)에서 수정됨.


**Cyfrin**
확인됨. 구현된 로직과의 일관성을 보장하기 위해 인라인 주석이 업데이트되었습니다.

\clearpage
## Low Risk


### Transmitter address update can cause DoS for In-flight CCIP Messages

**Description:** L1Transmitter 및 L2Transmitter 컨트랙트는 다른 체인의 상대방 트랜스미터 주소를 업데이트할 수 있습니다. 그러나 이 업데이트 메커니즘은 전송 중인 CCIP 메시지를 고려하지 않으므로 도착 시 거부될 가능성이 있습니다.

`L1Transmitter.sol`에서:

```solidity
//L1Transmitter.sol
function setL2Transmitter(address _l2Transmitter) external onlyOwner {
    l2Transmitter = _l2Transmitter;
}

function _ccipReceive(Client.Any2EVMMessage memory _message) internal override {
    if (_message.sourceChainSelector != l2ChainSelector) revert InvalidSourceChain();
    if (abi.decode(_message.sender, (address)) != l2Transmitter) revert InvalidSender();
    // ... rest of the function
}
```

마찬가지로 `L2Transmitter.sol`에서:

```solidity
function setL1Transmitter(address _l1Transmitter) external onlyOwner {
    l1Transmitter = _l1Transmitter;
}

function _ccipReceive(Client.Any2EVMMessage memory _message) internal override {
    if (_message.sourceChainSelector != l1ChainSelector) revert InvalidSourceChain();
    if (abi.decode(_message.sender, (address)) != l1Transmitter) revert InvalidSender();
    // ... rest of the function
}
```

이 문제는 두 컨트랙트의 `_ccipReceive` 함수가 저장된 트랜스미터 주소에 대해 발신자를 확인하기 때문에 발생합니다. 메시지가 전송되는 동안 이 주소가 업데이트되면, 발신자가 더 이상 업데이트된 트랜스미터 주소와 일치하지 않기 때문에 해당 메시지는 도착 시 거부됩니다.

**Impact:** 크로스체인 통신에 대한 서비스 거부(DoS)로 이어질 수 있습니다. 트랜스미터 주소 업데이트 시 전송 중인 모든 메시지는 거부되어 잠재적으로 다음을 유발할 수 있습니다:

- 중요한 업데이트 또는 인출 손실
- L1과 L2 간 상태 비동기화

**Recommended Mitigation:** 두 트랜스미터 모두 업그레이드 가능하므로 이 함수를 제거하는 것을 고려하십시오. 필요한 경우, 주소를 업데이트할 필요 없이 트랜스미터 구현을 변경할 수 있습니다. 경로는 소유자가 새 주소를 설정하기 위해 이 함수들을 호출할 때 전송 중인 메시지가 없는지 확인하십시오.

**Stake.Link**
인지됨. 소유자는 이 함수들을 호출할 때 전송 중인 메시지가 없는지 확인할 것입니다.

**Cyfrin**
인지됨.

\clearpage
