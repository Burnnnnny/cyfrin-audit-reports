**수석 감사**

[Immeas](https://twitter.com/0ximmeas)

[InAllHonesty](https://x.com/0xInAllHonesty)

---

# 발견 사항

## 중간 위험 (Medium Risk)

### `SDLVesting::stakeReleasableTokens`의 접근 제어 누락

**설명:** [`SDLVesting::stakeReleasableTokens`](https://github.com/stakedotlink/contracts/blob/3462c0d04ff92a23843adf0be8ea969b91b9bf0c/contracts/vesting/SDLVesting.sol#L108) 함수는 Stake.Link 팀이 운영하는 신뢰할 수 있는 스테이킹 봇에 의해 주기적으로 실행되어 새로 베스팅된 SDL을 수혜자가 선택한 기간 동안 SDLPool에 잠그도록 의도되었습니다. 그러나 현재 접근 제어가 없습니다.

따라서 누구나 `SDLVesting::stakeReleasableTokens`를 호출할 수 있습니다. 이를 이용하여 공격자는 수혜자의 토큰을 잠금으로써 `SDLVesting::release` 호출을 프론트러닝(front run)할 수 있습니다.

**파급력:** 공격자는 수혜자의 `release()` 트랜잭션을 프론트러닝하여 수혜자가 베스팅된 토큰에 접근하는 것을 반복적으로 거부할 수 있습니다.

**개념 증명 (Proof of Concept):**
```javascript
  it('should prevent griefing attacks', async () => {
      const { signers, accounts, start, vesting, sdlPool, sdlToken } = await loadFixture(deployFixture)

      await vesting.connect(signers[1]).setLockTime(4) // 4년 잠금
      await time.increase(DAY)

      const releasableAmount = await vesting.releasable()
      console.log("Tokens victim wanted as liquid:", fromEther(releasableAmount), "SDL")

      await vesting.stakeReleasableTokens() // 누군가 프론트러닝하여 스테이킹 강제
      const lockIdAfter = await sdlPool.lastLockId()
      console.log("Frontrun with `stakeReleasableTokens`, position created with ID:", lockIdAfter.toString())
      console.log("Position owner:", await sdlPool.ownerOf(lockIdAfter), "(vesting contract)")

      // 스테이킹 포지션 세부 정보 가져오기
      const locks = await sdlPool.getLocks([lockIdAfter])
      const lock = locks[0]

      const currentTime = await time.latest()
      const lockDuration = Number(lock.duration)
      const unlockInitiationTime = Number(lock.startTime) + lockDuration / 2
      const fullUnlockTime = Number(lock.startTime) + lockDuration

      console.log("Base SDL staked:", fromEther(lock.amount), "SDL")
      console.log("Boost received:", fromEther(lock.boostAmount), "SDL")
      console.log("Total effective staking power:", fromEther(lock.amount + lock.boostAmount), "SDL")
      console.log("Lock duration:", lockDuration / (365 * 86400), "years")

      console.log("Years until unlock initiation allowed:", (unlockInitiationTime - currentTime) / (365 * 86400))
      console.log("Years until full withdrawal possible:", (fullUnlockTime - currentTime) / (365 * 86400))

      // 피해자는 유동 토큰이 거의 없음
      await vesting.connect(signers[1]).release() // 아주 작은 나머지 받기
      const victimLiquidBalance = await sdlToken.balanceOf(accounts[1])
      console.log("Victim's liquid balance:", fromEther(victimLiquidBalance), "SDL (instead of", fromEther(releasableAmount), "SDL)")

      // 피해자가 포지션을 자신에게 전송하지만 여전히 잠겨 있음
      console.log("Transferring position to beneficiary...")
      await vesting.connect(signers[1]).withdrawRESDLPositions([4])
      console.log("Position now owned by:", await sdlPool.ownerOf(lockIdAfter))

      try {
        await sdlPool.connect(signers[1]).initiateUnlock(lockIdAfter)
        console.log("UNEXPECTED: Immediate unlock initiation succeeded!")
      } catch (error) {
        console.log("EXPECTED: Beneficiary cannot initiate unlock yet")
        console.log("Must wait 2 years before unlock can even be INITIATED, and another 2 years for full withdrawal")
      }
    })
```

**권장 완화 방법:**
1. 스테이킹 함수를 호출할 수 있는 유일한 비수혜자인 `stakingBot` 주소를 도입하십시오:

   ```diff
   +    /// @notice Address of the trusted bot allowed to call stakeReleasableTokens
   +    address public stakingBot;
   ```

2. 생성자에서 `_owner` 및 `_beneficiary`와 함께 설정하십시오:

   ```diff
       constructor(
           address _sdlToken,
           address _sdlPool,
           address _owner,
           address _beneficiary,
   +       address _stakingBot,
           uint64 _start,
           uint64 _duration,
           uint64 _lockTime
       ) {
           _transferOwnership(_owner);
   +       stakingBot = _stakingBot;
           …
       }
   ```

3. 수혜자 *또는* 스테이킹 봇만 허용하는 결합된 접근 제어 수정자를 만드십시오:

   ```diff
   +    modifier onlyBeneficiaryOrBot() {
   +        if (msg.sender != beneficiary && msg.sender != stakingBot) {
   +            revert SenderNotAuthorized();
   +        }
   +        _;
   +    }
   ```

4. 해당 수정자를 `stakeReleasableTokens()`에 적용하십시오:

   ```diff
   -    function stakeReleasableTokens() external {
   +    function stakeReleasableTokens() external onlyBeneficiaryOrBot {
           uint256 amount = releasable();
           if (amount == 0) revert NoTokensReleasable();
           …
       }
   ```

5. 봇의 키가 교체되거나 Stake.Link 팀이 자동화 주소를 변경해야 할 경우를 대비하여 소유자 전용 설정자를 추가하십시오:

   ```diff
   +    /// @notice Update the trusted staking bot address
   +    function setStakingBot(address _stakingBot) external onlyOwner {
   +        require(_stakingBot != address(0), "Invalid bot address");
   +        stakingBot = _stakingBot;
   +    }
   ```

이러한 변경을 통해 지정된 봇(및 원하는 경우 수혜자 자신)만이 주기적인 스테이킹을 트리거할 수 있게 되어 토큰을 무기한 잠글 수 있는 괴롭힘(griefing) 벡터를 제거합니다.

**Stake.Link:** 커밋 [`565b043`](https://github.com/stakedotlink/contracts/commit/565b043b98f6b0a61a9eda9b7f2ca20ecdac8598)에서 수정되었습니다.

**Cyfrin:** 확인됨. 주소 `staker`가 이제 생성자에 전달됩니다. 그리고 `onlyBeneficiaryOrStaker` 수정자가 `stakeReleasableTokens`에 적용됩니다.

\clearpage
## 정보성 (Informational)

### 기간이 0인 베스팅 엣지 케이스 (Zero‑Duration vesting edge case)

**설명:** `duration == 0`일 때, `vestedAmount(start)`를 호출하면 `else if (_timestamp > start + duration)` 검사(`start > start`가 거짓이므로)를 통과하여 선형 베스팅 분기로 떨어져 다음을 실행합니다.

```solidity
(totalAllocation * (start - start)) / duration
```

이는 정확히 `start`에서 짧은 1초 되돌리기(revert) 창을 생성합니다. 이후의 타임스탬프(`> start`)는 전체 할당을 올바르게 반환하기 때문입니다.

비교를 `>=`로 변경하는 것을 고려하십시오:
```diff
- else if (_timestamp > start + duration) {
+ else if (_timestamp >= start + duration) {
    return totalAllocation;
}
```

이렇게 하면 `start + duration`(0인 경우에도)이 즉시 "완전 베스팅됨" 분기를 산출합니다.

**Stake.Link:** 커밋 [`e458512`](https://github.com/stakedotlink/contracts/commit/e4585124c05137848196d4ca759c3e9d28b963e1)에서 수정되었습니다.

**Cyfrin:** 확인됨. 비교가 `>=`가 아닙니다.

### `constructor`에서 `_lockTime` 검증 누락

**설명:** 생성자는 `_lockTime <= MAX_LOCK_TIME`인지 확인하지 않고 `lockTime = _lockTime`을 할당합니다. 범위를 벗어난 값이 제공되면 `reSDLTokenIds[lockTime]`을 인덱싱하는 후속 호출(예: `stakeReleasableTokens` 또는 `withdrawRESDLPositions`)은 배열 경계 오류와 함께 되돌려집니다.

UX를 개선하고 빠르게 실패하도록 생성자에 명시적 검사를 추가하는 것을 고려하십시오:

```solidity
require(_lockTime <= MAX_LOCK_TIME, "Invalid lock time");
```

**Stake.Link:** 커밋 [`e458512`](https://github.com/stakedotlink/contracts/commit/e4585124c05137848196d4ca759c3e9d28b963e1)에서 수정되었습니다.

**Cyfrin:** 확인됨. `_lockTime`이 이제 `MAX_LOCK_TIME`보다 크지 않아야 합니다.

### `Ownable2Step` 사용 고려

**설명:** 이 계약은 OpenZeppelin의 단일 단계 `Ownable`을 사용하며, 여기서 `transferOwnership` 시 즉시 소유권이 이전되어 우발적하거나 원치 않는 이전의 위험이 있습니다. 새 소유자가 명시적으로 `acceptOwnership`을 호출해야 하는 OZ의 `Ownable2Step`으로 전환하는 것을 고려하십시오. 이 2단계 패턴은 잘못 전송되거나 확인되지 않은 소유권 이전을 방지하고 모범 사례인 "확인되지 않은 2단계 소유권 이전" 보호 조치와 일치합니다.

**Stake.Link:** 인지함.

### 소유자의 소유권 포기 비활성화 고려

**설명:** `Ownable`의 기본 `renounceOwnership()`은 소유자가 소유자를 `address(0)`으로 설정하여 제어를 완전히 포기하도록 허용하며, 이는 의도치 않게 호출될 수 있습니다. 특권 기능의 영구적인 손실을 방지하려면 `renounceOwnership`을 재정의하여 비활성화하거나 제한하는 것을 고려하십시오. 예를 들어:

```solidity
function renounceOwnership() public view override onlyOwner {
    revert("Renouncing ownership is disabled");
}
```

이는 소유권이 의도적인 `transferOwnership`(또는 `Ownable2Step`)을 통해서만 변경될 수 있도록 하여 우발적이거나 되돌릴 수 없는 포기를 방지합니다.

**Stake.Link:** 인지함.

### `SDLVesting::withdrawRESDLPositions` 개선 사항

**설명:** `SDLVesting::withdrawRESDLPositions()` 함수는 상태를 업데이트하기 전에 외부 호출을 수행하여 Checks-Effects-Interactions (CEI) 패턴을 위반합니다. 또한 이 함수는 잠금 시간에 대한 입력 유효성 검사가 부족하여 존재하지 않는 토큰 ID를 전송하려고 시도하여 트랜잭션 되돌림(revert) 및 열악한 사용자 경험을 유발할 수 있습니다.

SDLPool의 소유권 확인으로 인해 악용 가능성은 없지만, 함수는 외부 호출 후 상태를 업데이트하여 잠재적인 재진입(reentrancy) 벡터와 잘못된 입력의 경우 나쁜 사용자 경험을 생성합니다.

`_lockTimes`에 대한 검증을 추가하고 외부 호출 전에 상태 변경을 이동하는 것을 고려하십시오:

```diff
    function withdrawRESDLPositions(uint256[] calldata _lockTimes) external onlyBeneficiary {
        for (uint256 i = 0; i < _lockTimes.length; ++i) {

-            sdlPool.safeTransferFrom(address(this), beneficiary, reSDLTokenIds[_lockTimes[i]]);
-            delete reSDLTokenIds[_lockTimes[i]];

+            if (_lockTimes[i] > MAX_LOCK_TIME) revert InvalidLockTime();
+            uint256 tokenId = reSDLTokenIds[_lockTimes[i]]; // Cache to facilitate the deletion before transfer
+            if (tokenId == 0) continue; // Skip if no reSDL position exists so we don't break execution but also don't attempt to transfer 0.
+            delete reSDLTokenIds[_lockTimes[i]];
+            sdlPool.safeTransferFrom(address(this), beneficiary, tokenId);
        }
    }
```

**Stake.Link:** 커밋 [`e458512`](https://github.com/stakedotlink/contracts/commit/e4585124c05137848196d4ca759c3e9d28b963e1)에서 수정되었습니다.

**Cyfrin:** 확인됨. `_lockTime[i]`가 이제 `MAX_LOCK_TIME`보다 크지 않은지 확인하고 `safeTransfer` 호출 전에 삭제가 수행됩니다.

### `SDLVesting::claimRESDLRewards()`가 엣지 케이스에서 전체 베스팅 계약 잔액을 소진하는 데 사용될 수 있음

**설명:** `SDLVesting::claimRESDLRewards` 함수는 지정된 각 보상 토큰에 대해 전체 토큰 잔액을 수혜자에게 전송합니다. 관리자가 실수로 `RewardsPoolController`에 SDL 토큰을 보상 토큰으로 추가하면, 수혜자는 `SDLVesting::claimRESDLRewards([sdlToken])`을 호출하여 베스팅된 SDL 토큰과 베스팅되지 않은 SDL 토큰을 모두 즉시 소진하여 베스팅 일정을 완전히 우회할 수 있습니다.

```solidity
    function claimRESDLRewards(address[] calldata _tokens) external onlyBeneficiary {
        sdlPool.withdrawRewards(_tokens);

        for (uint256 i = 0; i < _tokens.length; ++i) {
            IERC20 token = IERC20(_tokens[i]);
            uint256 balance = token.balanceOf(address(this));

            if (balance != 0) {
                token.safeTransfer(beneficiary, balance);
            }
        }
    }
```
이는 전체 `token.balanceOf(address(this));`가 수혜자에게 전송되기 때문에 발생합니다.

SDL 토큰이 보상으로 청구되는 것을 방지하기 위한 확인을 추가하는 것을 고려하십시오:

```diff
function claimRESDLRewards(address[] calldata _tokens) external onlyBeneficiary {
    sdlPool.withdrawRewards(_tokens);

    for (uint256 i = 0; i < _tokens.length; ++i) {
+       if (_tokens[i] == address(sdlToken)) continue; // Skip SDL token

        IERC20 token = IERC20(_tokens[i]);
        uint256 balance = token.balanceOf(address(this));
        if (balance != 0) {
            token.safeTransfer(beneficiary, balance);
        }
    }
}
```

**Stake.Link:** 커밋 [`e458512`](https://github.com/stakedotlink/contracts/commit/e4585124c05137848196d4ca759c3e9d28b963e1)에서 수정되었습니다.

**Cyfrin:** 확인됨. `_tokens[i]`가 이제 `sdlToken`과 같지 않은지 확인합니다.

### NatSpec 개선 사항

**설명:** * [`SDLVesting#L11`](https://github.com/stakedotlink/contracts/blob/3462c0d04ff92a23843adf0be8ea969b91b9bf0c/contracts/vesting/SDLVesting.sol#L11): `beneficary`는 `beneficiary`여야 합니다.
* [`SDLVesting::onlyBeneficiary#L90`](https://github.com/stakedotlink/contracts/blob/3462c0d04ff92a23843adf0be8ea969b91b9bf0c/contracts/vesting/SDLVesting.sol#L90): `not  beneficiary`에 여분의 공백이 있습니다.
* [`SDLVesting::setLockTime#L187`](https://github.com/stakedotlink/contracts/blob/3462c0d04ff92a23843adf0be8ea969b91b9bf0c/contracts/vesting/SDLVesting.sol#L187): `_lockTime lock time in seconds`는 `_lockTime lock time in years`여야 합니다.
* [`SDLVesting::vestedAmount`](https://github.com/stakedotlink/contracts/blob/3462c0d04ff92a23843adf0be8ea969b91b9bf0c/contracts/vesting/SDLVesting.sol#L195-L198): 반환을 문서화하지 않습니다: `@return amount of tokens vested at the given timestamp. Returns full allocation if terminated.`

**Stake.Link:** 커밋 [`e458512`](https://github.com/stakedotlink/contracts/commit/e4585124c05137848196d4ca759c3e9d28b963e1) 및 [`565b043`](https://github.com/stakedotlink/contracts/commit/565b043b98f6b0a61a9eda9b7f2ca20ecdac8598)에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
## 가스 최적화 (Gas Optimization)

### 스토리지 변수 레이아웃 최적화 가능

**설명:** 현재 스토리지 레이아웃은 4개의 슬롯을 사용합니다.

```solidity
    // maximum lock time in years
    uint256 public constant MAX_LOCK_TIME = 4;

    // address of SDL token
    IERC677 public immutable sdlToken;
    // address of SDL pool
    ISDLPool public immutable sdlPool;

    // whether vesting has been terminated
    bool public vestingTerminated;

    // amount of tokens claimed by the beneficiary
    uint256 public released;
    // address to receive vested SDL
    address public immutable beneficiary;
    // start time of vesting in seconds
    uint64 public immutable start;
    // duration of vesting in seconds
    uint64 public immutable duration;

    // lock time in years to use for staking vested SDL
    uint64 public lockTime;
    // list of reSDL token ids for each lock time
    uint256[] private reSDLTokenIds;
```
```
╭-------------------+-----------+------+--------+-------+---------------------------------------------╮
| Name              | Type      | Slot | Offset | Bytes | Contract                                    |
+=====================================================================================================+
| _owner            | address   | 0    | 0      | 20    | contracts/vesting/SDLVesting.sol:SDLVesting |
|-------------------+-----------+------+--------+-------+---------------------------------------------|
| vestingTerminated | bool      | 0    | 20     | 1     | contracts/vesting/SDLVesting.sol:SDLVesting |
|-------------------+-----------+------+--------+-------+---------------------------------------------|
| released          | uint256   | 1    | 0      | 32    | contracts/vesting/SDLVesting.sol:SDLVesting |
|-------------------+-----------+------+--------+-------+---------------------------------------------|
| lockTime          | uint64    | 2    | 0      | 8     | contracts/vesting/SDLVesting.sol:SDLVesting |
|-------------------+-----------+------+--------+-------+---------------------------------------------|
| reSDLTokenIds     | uint256[] | 3    | 0      | 32    | contracts/vesting/SDLVesting.sol:SDLVesting |
╰-------------------+-----------+------+--------+-------+---------------------------------------------╯
```
`lockTime`을 `vestingTerminated ` 뒤로 이동하면 3개의 슬롯으로 줄일 수 있습니다:

```
╭-------------------+-----------+------+--------+-------+---------------------------------------------╮
| Name              | Type      | Slot | Offset | Bytes | Contract                                    |
+=====================================================================================================+
| _owner            | address   | 0    | 0      | 20    | contracts/vesting/SDLVesting.sol:SDLVesting |
|-------------------+-----------+------+--------+-------+---------------------------------------------|
| vestingTerminated | bool      | 0    | 20     | 1     | contracts/vesting/SDLVesting.sol:SDLVesting |
|-------------------+-----------+------+--------+-------+---------------------------------------------|
| lockTime          | uint64    | 0    | 21     | 8     | contracts/vesting/SDLVesting.sol:SDLVesting |
|-------------------+-----------+------+--------+-------+---------------------------------------------|
| released          | uint256   | 1    | 0      | 32    | contracts/vesting/SDLVesting.sol:SDLVesting |
|-------------------+-----------+------+--------+-------+---------------------------------------------|
| reSDLTokenIds     | uint256[] | 2    | 0      | 32    | contracts/vesting/SDLVesting.sol:SDLVesting |
╰-------------------+-----------+------+--------+-------+---------------------------------------------╯
```

```diff
    // maximum lock time in years
    uint256 public constant MAX_LOCK_TIME = 4;

    // address of SDL token
    IERC677 public immutable sdlToken;
    // address of SDL pool
    ISDLPool public immutable sdlPool;

    // whether vesting has been terminated
    bool public vestingTerminated;

+   // lock time in years to use for staking vested SDL
+   uint64 public lockTime;

    // amount of tokens claimed by the beneficiary
    uint256 public released;
    // address to receive vested SDL
    address public immutable beneficiary;
    // start time of vesting in seconds
    uint64 public immutable start;
    // duration of vesting in seconds
    uint64 public immutable duration;

-   // lock time in years to use for staking vested SDL
-   uint64 public lockTime;
    // list of reSDL token ids for each lock time
    uint256[] private reSDLTokenIds;
```

**Stake.Link:** 인지함.

### `SDLVesting::vestedAmount` 가스 최적화

**설명:** `SDLVesting::vestedAmount` 함수는 `if (_timestamp < start) return 0;` 검사 전에 `totalAllocation`을 계산합니다. 검사가 참을 반환하고 함수가 0을 반환하는 경우 해당 계산에 소비된 가스는 낭비됩니다.
또한 `_timestamp < start` 조건이 확인되었으므로 최종 반환 `return (totalAllocation * (_timestamp - start)) / duration;`은 unchecked 블록에서 수행하여 가스를 절약할 수 있습니다:

```diff
    function vestedAmount(uint64 _timestamp) public view returns (uint256) {
+       if (_timestamp < start) {
+            return 0;

        uint256 totalAllocation = sdlToken.balanceOf(address(this)) + released;

-      if (_timestamp < start) {
-           return 0;
-       } else if (_timestamp > start + duration) {
-           return totalAllocation;
-       } else if (vestingTerminated) {
-           return totalAllocation;
-       } else {
-           return (totalAllocation * (_timestamp - start)) / duration;
+       if (_timestamp > start + duration) {
+           return totalAllocation;
+      } else if (vestingTerminated) {
+           return totalAllocation;
+      } else {
+           unchecked {
+               return (totalAllocation * (_timestamp - start)) / duration;
+           }
        }
    }
```

**Stake.Link:** 인지함.

### 변수 캐싱을 통한 `SDLVesting::stakeReleasableTokens` 가스 최적화

**설명:**
```solidity
    function stakeReleasableTokens() external {
        uint256 amount = releasable();
        if (amount == 0) revert NoTokensReleasable();

        released += amount;
        sdlToken.transferAndCall(
            address(sdlPool),
            amount,
            abi.encode(reSDLTokenIds[lockTime], lockTime * (365 days))
        );

        if (reSDLTokenIds[lockTime] == 0) {
            reSDLTokenIds[lockTime] = sdlPool.lastLockId();
        }

        emit Staked(amount);
    }
```

현재 `lockTime`과 `reSDLTokenIds[lockTime]`은 스토리지에서 여러 번 읽힙니다. 두 변수 모두 캐시하여 가스를 절약해야 합니다:

```diff
    function stakeReleasableTokens() external {
        uint256 amount = releasable();
        if (amount == 0) revert NoTokensReleasable();

        released += amount;
-       sdlToken.transferAndCall(
-           address(sdlPool),
-           amount,
-           abi.encode(reSDLTokenIds[lockTime], lockTime * (365 days))
-       );
-
-       if (reSDLTokenIds[lockTime] == 0) {
-           reSDLTokenIds[lockTime] = sdlPool.lastLockId();
-       }

+       uint64 cachedLockTime = lockTime;
+       uint256 cachedTokenId = reSDLTokenIds[cachedLockTime];
+
+       sdlToken.transferAndCall(
+          address(sdlPool),
+          amount,
+         abi.encode(cachedTokenId, cachedLockTime * (365 days))
+      );
+
+      if (cachedTokenId == 0) {
+          reSDLTokenIds[cachedLockTime] = sdlPool.lastLockId();
+      }

       emit Staked(amount);
    }
```

**Stake.Link:** 커밋 [`128c335`](https://github.com/stakedotlink/contracts/commit/128c33560d8f43057c5d10d822b4904d0762d0fd)에서 수정되었습니다.

**Cyfrin:** 확인됨. `tokenId`가 이제 캐시됩니다.

### `reSDLTokenIds`에 고정 길이 배열 사용

**설명:** 계약은 현재 다음과 같이 선언합니다.

```solidity
uint256[] private reSDLTokenIds;
```

그리고 생성자에서 `.push(0)`와 함께 `for`-루프를 사용하여 길이를 `MAX_LOCK_TIME + 1`로 초기화합니다. 이로 인해 다음이 발생합니다:

* 스토리지의 동적 배열 길이 슬롯
* 배열 데이터에 대한 포인터 슬롯
* 여러 스토리지 쓰기 (`.push`당 하나)

배열의 길이는 항상 정확히 `MAX_LOCK_TIME + 1`(5)이므로 정적 배열:

```solidity
uint256[MAX_LOCK_TIME + 1] private reSDLTokenIds;
```

은 동적 배열 오버헤드를 제거하고 초기화 루프를 제거합니다.

동적 배열을 고정 길이 배열로 교체하고 생성자 루프를 제거하는 것을 고려하십시오:

```diff
-   // list of reSDL token ids for each lock time
-   uint256[] private reSDLTokenIds;

+   // list of reSDL token ids for each lock time (0–4 years)
+   uint256[MAX_LOCK_TIME + 1] private reSDLTokenIds;

    constructor(…) {
        ...
-       for (uint256 i = 0; i <= MAX_LOCK_TIME; ++i) {
-           reSDLTokenIds.push(0);
-       }
     }
```

이 변경은 두 개의 스토리지 슬롯(길이 + 데이터 포인터)을 하나로 축소하고 비용이 많이 드는 초기화 루프를 제거하여 배포 및 읽기당 가스 비용을 모두 줄입니다.

**Stake.Link:** 커밋 [`128c335`](https://github.com/stakedotlink/contracts/commit/128c33560d8f43057c5d10d822b4904d0762d0fd)에서 수정되었습니다.

**Cyfrin:** 확인됨. `reSDLTokenIds`가 이제 정적입니다.

\clearpage
