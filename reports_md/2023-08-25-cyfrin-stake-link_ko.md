## 중간 중요도 (Medium Risk)
### 오프체인 메커니즘이 엄격하게 올바른 순서로 작동하도록 보장되어야 함 (The off-chain mechanism must be ensured to work in a correct order strictly)

**심각도:** 중간 (Medium)

**설명:** `PriorityPool` 컨트랙트는 회계를 위해 배포 오라클(distribution oracle)에 의존하며 회계 계산은 오프체인에서 수행됩니다.

프로토콜 팀과의 소통에 따르면 대기된 예치금(queued deposits)에 대한 올바른 워크플로우는 다음과 같습니다:
- 스테이킹 풀에 새로운 예치 공간이 생길 때마다 `depositQueuedTokens` 함수가 호출됩니다.
- `pauseForUpdate()`를 호출하여 `PriorityPool` 컨트랙트가 일시 중지됩니다.
- 회계 계산은 `getAccountData()` 함수와 `getDepositsSinceLastUpdate()`(`depositsSinceLastUpdate`) 변수를 사용하여 오프체인에서 발생하여 최신 머클 트리를 구성합니다.
- 배포 오라클은 `updateDistribution()` 함수를 호출하고 이로 인해 `PriorityPool`이 다시 시작됩니다.

큐 컨트랙트를 일시 중지하는 유일한 목적은 회계 상태가 업데이트될 때까지 대기 취소(unqueue)를 방지하는 것입니다.
분석을 통해 오프체인 메커니즘이 순서를 매우 엄격하게 따라야 하며 그렇지 않으면 사용자 자금이 도난당할 수 있음을 발견했습니다.
프로토콜 팀이 이를 보장할 것임을 인정하지만 오프체인 메커니즘을 검증할 수 없으므로 이 발견 사항을 중간 위험으로 유지하기로 결정했습니다.

**파급력:** 만약 오프체인 메커니즘이 잘못된 순서로 발생하면 사용자 자금이 도난당할 수 있습니다.
가능성이 낮으므로 파급력을 중간(Medium)으로 평가합니다.

**개념 증명 (Proof of Concept):** 아래 테스트 케이스는 공격 시나리오를 보여줍니다.
```javascript
  it('Cyfrin: off-chain mechanism in an incorrect order can lead to user funds being stolen', async () => {
    // try deposit 1500 while the capacity is 1000
    await strategy.setMaxDeposits(toEther(1000))
    await sq.connect(signers[1]).deposit(toEther(1500), true)

    // 500 ether is queued for accounts[1]
    assert.equal(fromEther(await stakingPool.balanceOf(accounts[1])), 1000)
    assert.equal(fromEther(await sq.getQueuedTokens(accounts[1], 0)), 500)
    assert.equal(fromEther(await token.balanceOf(accounts[1])), 8500)

    // unqueue 500 ether should work while no updateDistribution was called
    await sq.connect(signers[1]).unqueueTokens(0, 0, [], toEther(500))
    assert.equal(fromEther(await sq.getQueuedTokens(accounts[1], 0)), 0)
    assert.equal(fromEther(await token.balanceOf(accounts[1])), 9000)

    // deposit again
    await sq.connect(signers[1]).deposit(toEther(500), true)
    assert.equal(fromEther(await token.balanceOf(accounts[1])), 8500)

    // victim deposits 500 ether and it will be queued
    await sq.connect(signers[2]).deposit(toEther(500), true)
    assert.equal(fromEther(await sq.totalQueued()), 1000)

    // max deposit has increased to 1500
    await strategy.setMaxDeposits(toEther(1500))

    // user sees that his queued tokens 500 can be deposited and call depositQueuedTokens
    // this will deposit the 500 ether in the queue
    await sq.connect(signers[1]).depositQueuedTokens()

    // Correct off-chain mechanism: pauseForUpdate -> getAccountData -> updateDistribution
    // Let us see what happens if getAccountData is called before pauseForUpdate

    // await sq.pauseForUpdate()

    // check account data
    var a_data = await sq.getAccountData()
    assert.equal(ethers.utils.formatEther(a_data[2][1]), "500.0")
    assert.equal(ethers.utils.formatEther(a_data[2][2]), "500.0")

    // user calls unqueueTokens to get his 500 ether back
    // this is possible because the queue contract is not paused
    await sq.connect(signers[1]).unqueueTokens(0, 0, [], toEther(500))

    // pauseForUpdate is called at a wrong order
    await sq.pauseForUpdate()

    // at this point user has 1000 ether staked and 9000 ether in his wallet
    assert.equal(fromEther(await token.balanceOf(accounts[1])), 9000)
    assert.equal(fromEther(await stakingPool.balanceOf(accounts[1])), 1000)

    // now updateDistribution is called with the wrong data
    let data = [
      [ethers.constants.AddressZero, toEther(0), toEther(0)],
      [accounts[1], toEther(500), toEther(500)],
    ]
    let tree = StandardMerkleTree.of(data, ['address', 'uint256', 'uint256'])

    await sq.updateDistribution(
      tree.root,
      ethers.utils.formatBytes32String('ipfs'),
      toEther(500),
      toEther(500)
    )

    // at this point user claims his LSD tokens
    await sq.connect(signers[1]).claimLSDTokens(toEther(500), toEther(500), tree.getProof(1))

    // at this point user has 1500 ether staked and 9000 ether in his wallet
    assert.equal(fromEther(await token.balanceOf(accounts[1])), 9000)
    assert.equal(fromEther(await stakingPool.balanceOf(accounts[1])), 1500)
    assert.equal(fromEther(await stakingPool.balanceOf(accounts[1])), 1500)
  })
```
**권장 완화 방안:** `_depositQueuedTokens` 함수의 끝에서 컨트랙트를 강제로 일시 중지하는 것을 고려하십시오.

**클라이언트:**
인정함. 프로토콜 팀은 오프체인 메커니즘의 올바른 순서를 보장할 것입니다.

**Cyfrin:** 인정함.

### 사용자의 자금이 PriorityPool 컨트랙트에 일시적으로 잠김 (User's funds are locked temporarily in the PriorityPool contract)

**심각도:** 중간 (Medium)

**설명:** 프로토콜은 스테이킹/언스테이킹 상호 작용을 최소화하기 위해 인출에 예치금 큐(deposit queue)를 활용하려고 했습니다.
사용자가 인출을 원할 때, 원하는 금액을 매개변수로 하여 `PriorityPool::withdraw()` 함수를 호출해야 합니다.
```solidity
function withdraw(uint256 _amount) external {//@audit-info LSD token
    if (_amount == 0) revert InvalidAmount();
    IERC20Upgradeable(address(stakingPool)).safeTransferFrom(msg.sender, address(this), _amount);//@audit-info get LSD token from the user
    _withdraw(msg.sender, _amount);
}
```
구현에서 볼 수 있듯이 프로토콜은 먼저 사용자로부터 `_amount`의 LSD 토큰을 가져온 다음 큐를 활용한 실제 인출이 처리되는 `_withdraw()`를 호출합니다.
```solidity
function _withdraw(address _account, uint256 _amount) internal {
    if (poolStatus == PoolStatus.CLOSED) revert WithdrawalsDisabled();

    uint256 toWithdrawFromQueue = _amount <= totalQueued ? _amount : totalQueued;//@audit-info if the queue is not empty, we use that first
    uint256 toWithdrawFromPool = _amount - toWithdrawFromQueue;

    if (toWithdrawFromQueue != 0) {
        totalQueued -= toWithdrawFromQueue;
        depositsSinceLastUpdate += toWithdrawFromQueue;//@audit-info regard this as a deposit via the queue
    }

    if (toWithdrawFromPool != 0) {
        stakingPool.withdraw(address(this), address(this), toWithdrawFromPool);//@audit-info withdraw from pool into this contract
    }

    //@audit-warning at this point, toWithdrawFromQueue of LSD tokens remain in this contract!

    token.safeTransfer(_account, _amount);//@audit-info
    emit Withdraw(_account, toWithdrawFromPool, toWithdrawFromQueue);
}
```
그러나 `_withdraw()` 함수를 살펴보면 `toWithdrawFromPool` 만큼의 LSD 토큰만 스테이킹 풀에서 인출(소각)되고 `toWithdrawFromQueue` 만큼의 LSD 토큰은 `PriorityPool` 컨트랙트에 남아 있습니다.
반면, 컨트랙트는 `accountQueuedTokens` 매핑을 통해 사용자의 대기 금액을 추적하며, 이는 회계 불일치로 이어질 수 있습니다.
이러한 불일치로 인해 사용자의 대기 금액(`getQueuedTokens()`)이 양수로 보이는 동안 사용자의 LSD 토큰이 `PriorityPool` 컨트랙트에 잠길 수 있습니다.
사용자는 `updateDistribution` 함수가 호출되면 잠긴 LSD 토큰을 청구할 수 있습니다.
프로토콜 팀과의 소통을 통해 `updateDistribution`은 _스테이킹 풀에 새로운 예치금이 없는 한 아마도 1-2일마다_ 호출될 것으로 예상된다는 점을 이해했습니다.
즉, 사용자의 자금이 일시적으로 컨트랙트에 잠길 수 있으며 이는 사용자에게 불공평합니다.

**파급력:** 사용자의 LSD 토큰이 PriorityPool 컨트랙트에 일시적으로 잠길 수 있습니다.

**개념 증명 (Proof of Concept):**
```javascript
 it('Cyfrin: user funds can be locked temporarily', async () => {
    // try deposit 1500 while the capacity is 1000
    await strategy.setMaxDeposits(toEther(1000))
    await sq.connect(signers[1]).deposit(toEther(1500), true)

    // 500 ether is queued for accounts[1]
    assert.equal(fromEther(await stakingPool.balanceOf(accounts[1])), 1000)
    assert.equal(fromEther(await sq.getQueuedTokens(accounts[1], 0)), 500)
    assert.equal(fromEther(await token.balanceOf(accounts[1])), 8500)
    assert.equal(fromEther(await sq.totalQueued()), 500)
    assert.equal(fromEther(await stakingPool.balanceOf(sq.address)), 0)

    // at this point user calls withdraw (maybe by mistake?)
    // withdraw swipes from the queue and the deposit room stays at zero
    await stakingPool.connect(signers[1]).approve(sq.address, toEther(500))
    await sq.connect(signers[1]).withdraw(toEther(500))

    // at this point getQueueTokens[accounts[1]] does not change but the queue is empty
    // user will think his queue position did not change and he can simply unqueue
    assert.equal(fromEther(await stakingPool.balanceOf(accounts[1])), 500)
    assert.equal(fromEther(await sq.getQueuedTokens(accounts[1], 0)), 500)
    assert.equal(fromEther(await token.balanceOf(accounts[1])), 9000)
    assert.equal(fromEther(await sq.totalQueued()), 0)
    // NOTE: at this point 500 ethers of LSD tokens are locked in the queue contract
    assert.equal(fromEther(await stakingPool.balanceOf(sq.address)), 500)

    // but unqueueTokens fails because actual totalQueued is zero
    await expect(sq.connect(signers[1]).unqueueTokens(0, 0, [], toEther(500))).to.be.revertedWith(
      'InsufficientQueuedTokens()'
    )

    // user's LSD tokens are still locked in the queue contract
    await stakingPool.connect(signers[1]).approve(sq.address, toEther(500))
    await sq.connect(signers[1]).withdraw(toEther(500))
    assert.equal(fromEther(await stakingPool.balanceOf(accounts[1])), 0)
    assert.equal(fromEther(await sq.getQueuedTokens(accounts[1], 0)), 500)
    assert.equal(fromEther(await token.balanceOf(accounts[1])), 9500)
    assert.equal(fromEther(await sq.totalQueued()), 0)
    assert.equal(fromEther(await stakingPool.balanceOf(sq.address)), 500)

    // user might try withdraw again but it will revert because user does not have any LSD tokens
    await stakingPool.connect(signers[1]).approve(sq.address, toEther(500))
    await expect(sq.connect(signers[1]).withdraw(toEther(500))).to.be.revertedWith(
      'Transfer amount exceeds balance'
    )

    // in conclusion, user's LSD tokens are locked in the queue contract and he cannot withdraw them
    // it is worth noting that the locked LSD tokens are credited once updateDistribution is called
    // so the lock is temporary
  })
```
**권장 완화 방안:** 사용자가 컨트랙트에서 직접 LSD 토큰을 인출할 수 있는 기능을 추가하는 것을 고려하십시오.

**클라이언트:**
이 [PR](https://github.com/stakedotlink/contracts/pull/32)에서 수정되었습니다.

**Cyfrin:** 검증됨.

## 낮은 중요도 (Low Risk)
### 사용되지 않는 라이브러리 함수를 사용하지 마십시오 (Do not use deprecated library functions)
```solidity
File: PriorityPool.sol

103:         token.safeApprove(_stakingPool, type(uint256).max);

```


## 정보성 (Informational)
### 불필요한 이벤트 방출 (Unnecessary event emissions)
`PriorityPool::setPoolStatusClosed`는 풀 상태가 이미 `CLOSED`인지 확인하지 않고 `SetPoolStatus` 이벤트를 방출합니다. 풀 상태가 이미 닫혀 있다면 이벤트 방출을 피하세요. 이를 피하십시오. 동일한 내용이 `setPoolStatus` 함수에도 적용됩니다.

### 주소 상태 변수에 값을 할당할 때 `address(0)` 확인 누락 (Missing checks for `address(0)` when assigning values to address state variables)

```solidity
File: PriorityPool.sol

399:         distributionOracle = _distributionOracle;

```

### 내부적으로 사용되지 않는 함수는 external로 표시될 수 있음 (Functions not used internally could be marked external)
```solidity
File: PriorityPool.sol

89:     function initialize(

278:     function depositQueuedTokens() public {

```

