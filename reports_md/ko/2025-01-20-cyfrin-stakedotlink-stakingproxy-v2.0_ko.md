**Lead Auditors**

[0kage](https://twitter.com/0kage_eth)

**Assisting Auditors**



---

# Findings
## Critical Risk


### 우선순위 풀의 즉시 인출로 인해 `StakingProxy` 계약의 자금 손실이 발생할 수 있음 (Instant withdrawals in priority pool can result in loss of funds for StakingProxy contract)

**설명:** 우선순위 풀(priority pool)에서 즉시 인출이 활성화된 경우, `staker`는 `StakingProxy` 계약을 통해 인출할 때 자금을 영구적으로 잃을 수 있습니다. 이 문제는 즉시 인출 중 우선순위 풀의 `_withdraw` 함수에서 인출된 금액이 적절하게 업데이트되지 않아 토큰이 우선순위 풀에 갇히고 사용자가 LST를 잃기 때문에 발생합니다.

`PriorityPool::_withdraw`

```solidity
function _withdraw(
    address _account,
    uint256 _amount,
    bool _shouldQueueWithdrawal,
    bool _shouldRevertOnZero,
    bytes[] memory _data
) internal returns (uint256) {
    if (poolStatus == PoolStatus.CLOSED) revert WithdrawalsDisabled();

    uint256 toWithdraw = _amount;
    uint256 withdrawn;
    uint256 queued;

    if (totalQueued != 0) {
        uint256 toWithdrawFromQueue = toWithdraw <= totalQueued ? toWithdraw : totalQueued;

        totalQueued -= toWithdrawFromQueue;
        depositsSinceLastUpdate += toWithdrawFromQueue;
        sharesSinceLastUpdate += stakingPool.getSharesByStake(toWithdrawFromQueue);
        toWithdraw -= toWithdrawFromQueue;
        withdrawn = toWithdrawFromQueue; // -----> @audit withdrawn is set here
    }

    if (
        toWithdraw != 0 &&
        allowInstantWithdrawals &&
        withdrawalPool.getTotalQueuedWithdrawals() == 0
    ) {
        uint256 toWithdrawFromPool = MathUpgradeable.min(stakingPool.canWithdraw(), toWithdraw);
        if (toWithdrawFromPool != 0) {
            stakingPool.withdraw(address(this), address(this), toWithdrawFromPool, _data);
            toWithdraw -= toWithdrawFromPool; // -----> @audit BUG withdrawn is not updated here
        }
    }
    // ... rest of the function
}
```

즉시 인출을 처리할 때, 함수는 스테이킹 풀에서 토큰을 성공적으로 인출한 후 withdrawn 변수를 업데이트하지 못합니다. 이로 인해 다음과 같은 순서가 발생합니다:

1. 스테이커가 `StakingProxy::withdraw`를 통해 인출을 시작합니다.
2. `StakingProxy`가 LST를 소각합니다.
3. `PriorityPool`이 스테이킹 풀에서 기본 토큰을 받습니다.
4. 그러나 `withdrawn`이 업데이트되지 않았으므로 `PriorityPool`은 토큰을 전송하지 않습니다.
5. `StakingProxy`가 유동적 스테이킹 토큰에 대한 액세스 권한을 잃는 동안 토큰은 `PriorityPool`에 갇혀 있습니다.

**영향:** `StakingProxy`는 즉시 인출을 시도할 때 유동적 스테이킹 토큰에 대한 액세스 권한을 영구적으로 잃습니다.

**개념 증명 (Proof of Concept):** 다음 테스트를 `staking-proxy.test.ts`에 복사하세요.

```typescript
 it('instant withdrawals from staking pool are not transferred to staker', async () => {
    const { stakingProxy, stakingPool, priorityPool, signers, token, strategy, accounts } = await loadFixture(deployFixture)

    // Enable instant withdrawals
    await priorityPool.setAllowInstantWithdrawals(true)

    // Deposit initial amount
    await token.approve(stakingProxy.target, toEther(1000))
    await stakingProxy.deposit(toEther(1000), ['0x'])

    // Setup for withdrawals
    await strategy.setMaxDeposits(toEther(2000))
    await strategy.setMinDeposits(0)

    // Track all relevant balances before withdrawal
    const preTokenBalance = await token.balanceOf(stakingProxy.target)
    const preLSTBalance = await stakingPool.balanceOf(stakingProxy.target)

    const prePPBalance = await token.balanceOf(priorityPool.target)

    const withdrawAmount = toEther(500)

    console.log('=== Before Withdrawal ===')
    console.log('Initial LST Balance - Proxy contract:', fromEther(preLSTBalance))
    console.log('Initial Token Balance - Poxy contract:', fromEther(preTokenBalance))


    // Perform withdrawal
    await stakingProxy.withdraw(
        withdrawAmount,
        0,
        0,
        [],
        [],
        [],
        ['0x']
    )

    // Check all balances after withdrawal
    const postTokenBalance = await token.balanceOf(stakingProxy.target)
    const postPPBalance = await token.balanceOf(priorityPool.target)
    const postLSTBalance = await stakingPool.balanceOf(stakingProxy.target)

    console.log('=== After Withdrawal ===')
    console.log('Priority Pool - token balance change:', fromEther(postPPBalance - prePPBalance))
    console.log('Staking Proxy - token balance change:', fromEther(postTokenBalance - preTokenBalance))
    console.log('Staking Proxy - LST balance change:', fromEther(postLSTBalance - preLSTBalance))

    const lstsRedeemed = fromEther(preLSTBalance - postLSTBalance)

    // Assertions

    // 1. Staking Proxy has redeeemed all his LSTs
    assert.equal(
      lstsRedeemed,
        500,
        "Staker redeemed 500 LSTs"
    )

    // 2. But staking proxy doesn't receive underlying tokens
    assert.equal(
      fromEther(postTokenBalance - preTokenBalance),
        0,
        "Staking Proxy didn't receive any tokens despite losing LSTs"
    )

    // 3. The tokens are stuck in Priority Pool
    assert.equal(
        fromEther(postPPBalance- prePPBalance),
        500,
        "Priority Pool is holding the withdrawn tokens"
    )
  })
```

**권장 완화 방법:** `PriorityPool::_withdraw`에서 즉시 인출을 처리할 때 `withdrawn` 변수를 업데이트하는 것을 고려하세요.

**Stake.Link:** 커밋 [5f3d282](https://github.com/stakedotlink/contracts/commit/5f3d2829f86bc74d6b9e805d7e61d9392d6b21b1)에서 수정됨.

**Cyfrin:** 검증됨.

\clearpage
## Low Risk


### 특권 인출과 함께 제한 없는 reSDL 토큰 입금은 실수로 reSDL 토큰 손실을 초래할 수 있음 (Unrestricted reSDL token deposits with privileged withdrawals can lead to accidental loss of reSDL tokens)

**설명:** `StakingProxy` 계약은 모든 주소에서 reSDL 토큰(ERC721)을 받을 수 있도록 허용하는 ERC721 수신자 기능을 구현합니다. 그러나 계약 소유자만 이러한 토큰을 인출할 수 있는 능력이 있습니다.

이는 사용자 소유의 reSDL 토큰이 실수로 또는 인출 제한을 이해하지 못한 채 프록시로 전송되는 경우 갇힐 수 있는 위험을 만듭니다.

`StakingProxy.sol`

```solidity

// ----> @audit Anyone can transfer reSDL tokens to the proxy
function onERC721Received(address, address, uint256, bytes calldata) external returns (bytes4) {
    return this.onERC721Received.selector;
}

// ----> @audit Only owner can withdraw reSDL tokens
function withdrawRESDLToken(uint256 _tokenId, address _receiver) external onlyOwner {
    if (sdlPool.ownerOf(_tokenId) != address(this)) revert InvalidTokenId();
    IERC721(address(sdlPool)).safeTransferFrom(address(this), _receiver, _tokenId);
}
```
reSDL 토큰은 보상을 얻는 시간 잠금 SDL 스테이킹 포지션을 나타냅니다. 프록시가 reSDL 토큰을 보유할 수 있는 기능은 보상 획득 목적을 위한 의도된 기능이지만, 특권 인출과 결합된 전송의 무제한 수용은 불필요한 위험을 초래합니다.

**영향:** 실수로 reSDL 토큰을 프록시로 전송한 사용자는 프로토콜 팀의 수동 개입 없이는 직접 복구할 방법이 없습니다.


**권장 완화 방법:** `StakingProxy` 계약에서 구성할 수 있는 승인된 주소의 전송만 수락하도록 `onERC721Received` 함수를 게이팅(gating)하는 것을 고려하세요.

**Stake.link:**
인지함.

**Cyfrin:** 인지함.


### 스토리지 갭 누락으로 인한 UUPS 업그레이드 가능 `StakingProxy`의 스토리지 충돌 위험 (Storage collision risk in UUPS upgradeable `StakingProxy` due to missing storage gap)

**설명:** `StakingProxy` 계약은 `UUPSUpgradeable` 및 `OwnableUpgradeable`을 상속하지만 업그레이드 중 스토리지 충돌을 방지하기 위한 스토리지 갭(storage gap)을 구현하지 않습니다.

`StakingProxy`는 제3자/DAO가 사용하도록 의도되었습니다. 이 계약이 자체 스토리지 변수가 있는 외부 계약에 의해 상속될 가능성이 있습니다. 이러한 시나리오에서 업그레이드 중에 `StakingProxy`에 새 스토리지 변수를 추가하면 스토리지 슬롯이 이동하고 심각한 스토리지 충돌 위험이 발생할 수 있습니다.


`StakingProxy.sol`
```solidity
contract StakingProxy is UUPSUpgradeable, OwnableUpgradeable {
    // address of asset token
    IERC20Upgradeable public token;
    // address of liquid staking token
    IStakingPool public lst;
    // address of priority pool
    IPriorityPool public priorityPool;
    // address of withdrawal pool
    IWithdrawalPool public withdrawalPool;
    // address of SDL pool
    ISDLPool public sdlPool;
    // address authorized to deposit/withdraw asset tokens
    address public staker; // ---> @audit missing storage slots
}
```

**영향:** 잠재적인 스토리지 충돌로 인해 데이터가 손상되고 계약이 오작동할 수 있습니다.

**권장 완화 방법:** 미래의 상속된 계약 변수를 위해 슬롯을 예약하도록 계약 끝에 스토리지 갭을 추가하는 것을 고려하세요. 50의 슬롯 크기는 업그레이드 가능한 계약에 대한 [OpenZeppelin의 권장 패턴](https://docs.openzeppelin.com/contracts/3.x/upgradeable#:~:text=Storage%20Gaps,with%20existing%20deployments.)입니다.

**Stake.link:**
인지함.

**Cyfrin:** 인지함.

\clearpage
