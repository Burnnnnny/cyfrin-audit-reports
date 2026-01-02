**Lead Auditors**

[Dacian](https://x.com/DevDacian)

[Immeas](https://x.com/0ximmeas)

**Assisting Auditors**

 

---

# 발견 사항 (Findings)
## 중간 위험 (Medium Risk)


### 활성 및 대기 중인 베팅은 누구나 취소할 수 있음

**설명:** [`Bet::cancel`](https://github.com/gskril/wannabet-v2/blob/d7333369548874cb99e9648ea2424ac1e67cc23f/contracts/src/Bet.sol#L163-L176) 함수는 메이커(maker)를 제외한 모든 주소가 `ACTIVE` 또는 `PENDING` 상태의 베팅을 취소할 수 있게 합니다:

```solidity
/// @dev Anybody can cancel an expired bet and send funds back to each party. The maker can cancel a pending bet.
function cancel() external {
    IBet.Bet memory b = _bet;

    // Can't cancel a bet that's already completed
    if (b.status >= IBet.Status.RESOLVED) {
        revert InvalidStatus();
    } else {
        // Pending or active bets at this point
        // The maker can cancel a pending bet, so block them from cancelling an active bet
        // @audit-issue this only prevents `maker` from cancelling, anyone including the taker can cancel an active bet
        if (b.maker == msg.sender && b.status != IBet.Status.PENDING) {
            revert InvalidStatus();
        }
    }
```

이 로직은 메이커가 `ACTIVE` 베팅을 취소하는 것만 막거나, 베팅이 최종 상태에 도달한 후 누구나 취소하는 것을 막을 뿐입니다. 실제로 이는 임의의 주소(테이커(`taker`) 포함)가 언제든지 `PENDING` 및 `ACTIVE` 베팅을 모두 취소할 수 있음을 의미합니다.

**영향:** 누구나 베팅을 취소할 수 있게 하면 그리핑(griefing)이 가능해지며, 더 중요하게는 테이커가 이미 수락한 베팅을 일방적으로 철회할 수 있게 됩니다. 예를 들어 스포츠 베팅에서 테이커가 자신의 쪽이 질 가능성이 높다고 판단되면, 단순히 `cancel()`을 호출하여 베팅을 되돌리고 판돈을 회수함으로써 베팅 메커니즘의 무결성을 훼손할 수 있습니다.

**개념 증명 (Proof of Concept):** 다음 테스트를 `BetFactory.t.sol`에 추가하여 `taker`가 만료되지 않은 `ACTIVE` 베팅을 취소할 수 있음을 보여줍니다:

```solidity
// Anyone can cancel a non-expired ACTIVE bet
function test_ActiveBetCanBeCancelledByAnyone() public {
    // 1. Have the taker accept the bet so it becomes ACTIVE
    vm.startPrank(taker);
    usdc.approve(address(betNoPool), 1000);
    betNoPool.accept();
    vm.stopPrank();

    // Sanity check: bet is ACTIVE
    assertEq(uint(betNoPool.bet().status), uint(IBet.Status.ACTIVE));

    // 2. Warp to a time before resolveBy so the bet is still non-expired
    IBet.Bet memory state = betNoPool.bet();
    vm.warp(uint256(state.resolveBy) - 1);
    assertEq(uint(betNoPool.bet().status), uint(IBet.Status.ACTIVE));

    uint256 makerBalanceBefore = usdc.balanceOf(maker);
    uint256 takerBalanceBefore = usdc.balanceOf(taker);

    // 3. taker cancels the bet
    vm.prank(taker);
    betNoPool.cancel();

    // 4. After cancellation, the bet is marked CANCELLED and funds are refunded
    assertEq(uint(betNoPool.bet().status), uint(IBet.Status.CANCELLED));
    assertEq(usdc.balanceOf(maker) - makerBalanceBefore, 1000);
    assertEq(usdc.balanceOf(taker) - takerBalanceBefore, 1000);
}
```

**권장 완화 조치:** 베팅이 활성화(active)되면 아무도 취소할 수 없도록 하거나, 대안으로 심판(judge)만이 활성 베팅을 취소할 수 있도록 하는 것을 고려하십시오.

**WannaBet:** 커밋 [cdf3d64](https://github.com/gskril/wannabet-v2/commit/cdf3d649ab818b67e26b82eb241013adbbcdf473)에서 수정됨.

**Cyfrin:** 확인함.


### `Bet::cancel`에서 한 전송은 되돌려지지만 다른 전송은 성공할 경우, 한 사용자의 토큰이 `Bet` 컨트랙트에 영구적으로 잠김

**설명:** `Bet::cancel`은 토큰 전송을 환불할 때 다음과 같이 수행합니다:
```solidity
try IERC20(b.asset).transfer(b.maker, makerRefund) {} catch {}
try IERC20(b.asset).transfer(b.taker, takerRefund) {} catch {}
```

하지만 `transfer` 호출 중 하나는 되돌려지(revert)는데 다른 하나는 성공하면, 트랜잭션은 여전히 성공적으로 실행됩니다. 이는 다음과 같은 결과를 초래합니다:
* 베팅을 `CANCELLED` 상태로 변경
* 한 사용자의 토큰을 `Bet` 컨트랙트 안에 유지

**영향:** 취소 중에 전송이 되돌려진 사용자는 `Bet::cancel`을 다시 호출할 수 없으므로 토큰을 인출할 방법이 없습니다. `Bet` 컨트랙트는 불변(immutable)이며 토큰을 구출할 수 있는 다른 함수가 없기 때문입니다.

**권장 완화 조치:** 가장 간단한 옵션은 조용한 `catch`를 제거하고 `transfer` 호출 중 하나라도 되돌려지면 전체 트랜잭션을 되돌리는 것입니다. 두 사용자 모두 환불받거나 아무도 환불받지 못하게 해야 합니다.

더 복잡한 옵션은 베팅 취소 시 다음과 같이 하는 것입니다:
* 전송이 성공한 경우 스토리지 `Bet::makerStake,takerStake`를 업데이트하여 환불된 금액을 차감
* 베팅이 취소되었지만 전송 실패로 인해 환불받지 못했다는 기록이 `Bet`에 남아있는 경우, 해당 베팅의 `maker` 또는 `taker`가 자신의 토큰을 인출할 수 있는 새로운 함수 추가

**WannaBet:** 커밋 [f1750a9](https://github.com/gskril/wannabet-v2/commit/f1750a975346b1472ef3306db31f6fc7bd2db3b5)에서 수정됨.

**Cyfrin:** 확인함.


### `BetFactory`의 소유자가 `BetFactory::createBet` 호출자를 미묘하게 러그풀(rug-pull)하여 토큰을 훔칠 수 있음

**설명:** `BetFactory`의 소유자는 Aave `supply` 함수와 동일한 인터페이스를 구현하고 단순히 토큰을 자신에게 전송하는 악의적인 컨트랙트 주소로 `BetFactory::setPool`을 호출하여 프론트러닝함으로써, `createBet` 호출자를 러그풀하고 토큰을 훔칠 수 있습니다.

이는 `Bet::initialize`가 풀에 최대 승인을 한 다음 그 위에서 `supply` 함수를 호출하기 때문에 작동합니다:
```solidity
// If the pool is set, approve and supply the funds to the pool
if (pool != address(0)) {
    IERC20(initialBet.asset).approve(pool, type(uint256).max);

    _aavePool.supply(
        initialBet.asset,
        initialBet.makerStake,
        address(this),
        0
    );
}
```

**권장 완화 조치:** `BetFactory::createBet,predictBetAddress`에서 `Clones.cloneDeterministic`에 전달되는 솔트(salt)에 `tokenToPool[asset]`을 포함하십시오. 이렇게 하면 풀이 변경되었을 때 새로운 `Bet` 컨트랙트가 호출자가 승인하지 않은 다른 주소에 생성되므로, `Bet::initialize`에서의 첫 번째 토큰 전송이 되돌려져 전체 트랜잭션이 되돌려지게 됩니다.

I-9에 대한 권장 수정 사항도 이 문제를 해결하는데, 이는 소유자가 토큰과 관련된 Aave 풀을 불법적인 풀로 설정하는 것을 방지하기 때문입니다.

**WannaBet:** 처음에 커밋 [fbd8016](https://github.com/gskril/wannabet-v2/commit/fbd80169eebe70ce96a0ddbbe61035014ee0b018)에서 솔트에 aave 풀을 포함하여 수정했습니다. 그러나 커밋 [70e1565](https://github.com/gskril/wannabet-v2/commit/70e1565b391992b7ea8b11f2cc59195478a69212)에서 I-9(풀 검증)에 대한 수정 사항을 구현한 후에는 풀이 유효한 것으로만 설정될 수 있으므로 솔트에 풀을 더 이상 포함하지 않았습니다.

**Cyfrin:** 확인함.

\clearpage
## 낮은 위험 (Low Risk)


### 표준 ERC20 함수 대신 `SafeERC20` 함수 사용

**설명:** 비표준 동작을 하는 ERC20 토큰을 지원하기 위해 표준 ERC20 함수 대신 [SafeERC20](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/utils/SafeERC20.sol) 함수인 `safeTransfer`, `safeTransferFrom`, `forceApprove` 등을 사용하십시오:
```solidity
Bet.sol
63:        IERC20(initialBet.asset).transferFrom(
71:            IERC20(initialBet.asset).approve(pool, type(uint256).max);
114:        IERC20(b.asset).transferFrom(msg.sender, address(this), b.takerStake);
150:        IERC20(b.asset).transfer(winner, totalWinnings);
155:            IERC20(b.asset).transfer(_treasury, remainder);
193:        try IERC20(b.asset).transfer(b.maker, makerRefund) {} catch {}
194:        try IERC20(b.asset).transfer(b.taker, takerRefund) {} catch {}
```

**WannaBet:** 커밋 [b571b26](https://github.com/gskril/wannabet-v2/commit/b571b26b093d20ab5d876fa0f8845671e1b6e80b)에서 수정됨.

**Cyfrin:** 확인함.


### 심판(Judge)이 메이커도 테이커도 아닌 임의의 승자를 지정할 수 있음

**설명:** `Bet::resolve`를 호출할 때, 지명된 심판은 상금을 받을 임의의 승자를 지정할 수 있습니다.

**영향:** 심판은 베팅이 `maker`와 `taker` 사이의 계약임에도 불구하고, 자신이나 베팅에 참여하지 않은 다른 관련 없는 당사자에게 상금을 보낼 수 있습니다.

**권장 완화 조치:** `Bet::resolve`는 승자가 `maker` 또는 `taker`인지 강제해야 합니다.

**WannaBet:** 커밋 [74654b5](https://github.com/gskril/wannabet-v2/commit/74654b59cc63c95c5ea8dd31f1e561bf7bd66285)에서 수정됨.

**Cyfrin:** 확인함.


### `BetFactory::createBet`에서 사용하는 솔트(Salt)가 중요한 매개변수를 제외하여 동일한 엔터티 간에 동일한 타임스탬프를 가진 다중 베팅을 방지함

**설명:** 여러 엔터티(특히 프로그램)는 동일한 타임스탬프를 가진 여러 베팅을 자신들 간에 "일괄 생성(batch create)"하고 싶을 수 있습니다.

`BetFactory::createBet`은 사용되는 솔트가 `maker, taker, acceptBy, resolveBy`로 구성되어 있지만 `asset`, `makerStake`, `takerStake` 또는 `betCount`와 같은 다른 필드를 통합하지 않기 때문에 이것이 불가능하며 되돌려(revert)질 것입니다.

**권장 완화 조치:** 가장 간단한 해결책은 솔트에 `betCount`를 추가하는 것이지만, 이는 프론트러닝을 통한 잠재적인 서비스 거부(DoS) 벡터를 도입합니다. 그러나 공격자가 새로운 컨트랙트를 배포해야 하므로 상당한 가스 비용이 들고 일시적인 DoS 외에는 얻을 것이 없기 때문에 이 DoS 벡터는 발생할 가능성이 낮습니다.

또 다른 해결책은 솔트에 `asset`, `makerStake`, `takerStake`를 추가하는 것입니다.

**WannaBet:** 인지함.


### 청구할 방법이 없어 Aave 보상 인센티브가 손실됨

**설명:** 베팅이 Aave 풀을 사용할 때, 자금은 Aave에 공급되어 [인센티브 보상](https://aave.com/docs/aave-v3/aptos/smart-contracts/incentives)을 축적할 수 있지만, 컨트랙트는 이러한 보상을 청구하거나 전달할 방법을 제공하지 않습니다(Aave 인센티브 컨트롤러와의 통합이 없으며 보상 토큰의 일반적인 스윕도 없음).

**영향:** 베팅 포지션에서 얻은 모든 Aave 인센티브 보상은 사실상 손실/고립되어 시간이 지남에 따라 수익을 놓치고 잠재적으로 보상 토큰 잔액을 복구할 수 없게 됩니다.

**권장 완화 조치:**
Aave의 인센티브 컨트롤러를 통해 인센티브를 청구하고 프로토콜의 경제학(재무부)에 따라 라우팅할 수 있는 보상 처리 메커니즘을 추가하십시오:
```solidity
function aave_claimRewards(address reward) external {
    address[] memory assets = new address[](1);
    assets[0] = _bet.asset;
    rewardsController.claimRewards(
        assets,
        type(uint256).max,
        _treasury,
        reward
    );
}
```

**WannaBet:** 인지함.

\clearpage
## 정보 (Informational)


### 키와 값의 목적을 명시적으로 나타내기 위해 명명된 매핑(named mappings) 사용

**설명:** 키와 값의 목적을 명시적으로 나타내기 위해 명명된 매핑을 사용하십시오:
```
BetFactory.sol
22:    mapping(address => address) public tokenToPool;
```

**WannaBet:** 커밋 [a20d1e7](https://github.com/gskril/wannabet-v2/commit/a20d1e7acfbeee00cc891324b022fcf0afdd721b)에서 수정됨.

**Cyfrin:** 확인함.


### `Ownable` 대신 `Ownable2Step` 사용

**설명:** `Ownable` 대신 [Ownable2Step](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/access/Ownable2Step.sol)을 사용하십시오.

**WannaBet:** 커밋 [4fb5f42](https://github.com/gskril/wannabet-v2/commit/4fb5f42f2ffc2ee07706eb47640d137c45d99b63)에서 수정됨.

**Cyfrin:** 확인함.


### 이미 명명된 반환 변수를 사용할 때 불필요한 `return` 문 제거

**설명:** 이미 명명된 반환 변수를 사용할 때 불필요한 `return` 문을 제거하십시오:
* `Bet::bet`

**WannaBet:** 커밋 [c116182](https://github.com/gskril/wannabet-v2/commit/c11618221162a1010109ece02f2e42bede3350ac)에서 수정됨.

**Cyfrin:** 확인함.


### 전송 수수료(Fee-on-transfer) 및 리베이스(rebasing) 토큰이 회계를 망가뜨림

**설명:** 베팅을 기록할 때 입력 데이터가 스토리지에 직접 저장됩니다:
```solidity
function initialize(
    IBet.Bet calldata initialBet,
    string calldata description,
    address pool,
    address treasury
) external initializer {

    _bet = initialBet;
```

이는 기록된 베팅 금액이 실제로 받은 금액이 아니라 "전체" 금액임을 의미합니다.

**영향:** 잘못된 회계로 인해 베팅 취소 및 해결이 되돌려(revert)져 토큰이 컨트랙트에 잠기게 됩니다.

**권장 완화 조치:** 가장 쉬운 해결책은 전송 수수료 또는 리베이스 토큰을 지원하지 않는 것입니다. 그렇지 않고 지원하려면 토큰을 먼저 전송한 다음, 수수료가 공제된 후 실제로 받은 금액을 확인하기 위해 전송된 금액에서 실제 잔액을 뺀 값을 스토리지에 저장해야 합니다.

리베이스 토큰의 경우 메커니즘은 Aave 잔액이 처리되는 방식과 유사할 것입니다. 컨트랙트에 총액을 기록하고 `cancel` 및 `resolve`를 위해 메이커와 테이커 간에 분할합니다. 남은 잔액은 `_treasury`로 보낼 수 있습니다.

**WannaBet:** 인지함; 우리는 이를 지원하지 않기로 결정했고 코드에 [주석](https://github.com/gskril/wannabet-v2/commit/1fa6da4d2acc96670ad136c733b14c3d2d09e558)을 명시적으로 추가했습니다.


### `Bet::accept,resolve,cancel`에서 외부 호출 전에 `Bet` 상태 업데이트

**설명:** Checks-Effects-Interactions [[1](https://fravoll.github.io/solidity-patterns/checks_effects_interactions.html), [2](https://docs.soliditylang.org/en/v0.6.11/security-considerations.html)] 패턴을 따르는 것이 현명합니다. `Bet::accept,resolve,cancel`에서 외부 호출을 하기 전에 다음 스토리지 업데이트가 발생해야 합니다:
```solidity
// `Bet::accept`
_bet.status = IBet.Status.ACTIVE;

// `Bet::resolve`
_bet.winner = winner;
_bet.status = IBet.Status.RESOLVED;

// `Bet::cancel`
_bet.status = IBet.Status.CANCELLED;
```

**WannaBet:** 커밋 [5cda880](https://github.com/gskril/wannabet-v2/commit/5cda88027d6081007642caf56dae49af5d753f41)에서 수정됨.

**Cyfrin:** 확인함.


### 남용을 방지하고 베팅이 해결될 수 있도록 `BetFactory::createBet`은 여러 시나리오에서 되돌려져야(revert) 함

**설명:** `BetFactory::createBet`은 다음과 같은 경우 되돌려져야 합니다:
* `treasury`가 `address(0)`인 경우
* `msg.sender`와 `taker`가 같은 경우
* `judge`가 `msg.sender` 또는 `taker`와 같은 경우

이 모든 시나리오는 초기화 또는 베팅 생성의 오류를 나타내므로 이러한 트랜잭션은 되돌려져야 합니다. `treasury`가 `address(0)`인 첫 번째 시나리오는 `address(0)`으로의 전송 시 되돌려지는 토큰에 대한 베팅 해결을 방지할 수 있습니다(취소는 여전히 작동함).

**WannaBet:** 메이커/테이커/심판을 동일한 주소로 허용합니다. 이는 테스트에 유용하며 기술적인 문제를 일으키지 않기 때문입니다.

커밋 [5ec090f](https://github.com/gskril/wannabet-v2/commit/5ec090f3eb73899dc191bcdf81306e9ef2cb1ab1), [f1750a9](https://github.com/gskril/wannabet-v2/commit/f1750a975346b1472ef3306db31f6fc7bd2db3b5)에서 재무부(treasury)가 설정되지 않은 경우 생성된 수익이 승자에게 가도록 수정 사항을 구현했습니다.

**Cyfrin:** 확인함.


### Aave 풀이 사용될 때 `Bet::resolve`의 `BetResolved` 이벤트가 잘못된 `totalWinnings`를 가짐

**설명:** `Bet::resolve`에서 `BetResolved` 이벤트는 Aave의 실제 자금을 확인하고 `totalWinnings`를 컨트랙트의 `aTokenBalance`로 제한하기 전에 `totalWinnings = makerStake + takerStake`로 이벤트를 발생시킵니다:
```solidity
uint256 totalWinnings = b.makerStake + b.takerStake;
emit BetResolved(winner, totalWinnings);

// If the funds are in Aave, withdraw them
if (address(_aavePool) != address(0)) {
    uint256 aTokenBalance = IERC20(_aavePool.getReserveAToken(b.asset)).balanceOf(address(this));

    // @audit totalWinnings changed
    totalWinnings = _min(totalWinnings, aTokenBalance);
    _aavePool.withdraw(b.asset, aTokenBalance, address(this));
}
```

Aave 풀이 사용될 때(수익 또는 손실 포함), 이벤트의 `amount` 필드는 `winner`에게 전송된 실제 상금과 일치하지 않을 수 있어 인덱서 및 오프체인 소비자에게 오해를 불러일으킬 수 있습니다.

Aave 회계가 적용된 후 이벤트를 발생시키는 것을 고려하십시오.

**WannaBet:** 커밋 [c18aae0](https://github.com/gskril/wannabet-v2/commit/c18aae04ef570ef877f7d06aa25ca6f10a379fc1)부터 수정됨.

**Cyfrin:** 확인함.


### 취소된 대기 중인 베팅에 대해 테이커가 Aave 수익을 받음

**설명:** 베팅이 Aave 풀을 사용하지만 `ACTIVE` 상태가 되지 않는 경우(테이커가 수락하지 않음), 메이커의 판돈만 Aave에 공급됩니다. `Bet::cancel`에서 회수된 Aave 잔액은 여전히 메이커/테이커 로직을 사용하여 분할됩니다:
```solidity
uint256 aTokenBalance = IERC20(_aavePool.getReserveAToken(b.asset))
    .balanceOf(address(this));
_aavePool.withdraw(b.asset, aTokenBalance, address(this));

makerRefund = _min(makerRefund, aTokenBalance);
takerRefund = _min(takerRefund, aTokenBalance - makerRefund);
```

따라서 테이커는 입금한 적이 없음에도 불구하고 메이커의 누적 수익(또는 회수된 원금)의 일부를 받을 수 있습니다. 하지만 이는 Aave에서 인출된 총액이 입금된 금액보다 적은 "마이너스 수익" 이벤트가 발생한 경우 메이커가 환불 우선순위를 갖는 것으로 균형이 맞춰집니다.

다음을 고려하십시오:
1) 베팅 취소 시 재무부가 Aave 생성 수익을 받아야 하는지 여부
2) 테이커가 수락하지 않은 경우에도 Aave 수익을 받아야 하는지 여부

**WannaBet:** 커밋 [f1750a9](https://github.com/gskril/wannabet-v2/commit/f1750a975346b1472ef3306db31f6fc7bd2db3b5)에서 다음과 같이 수정됨:
* 테이커는 입금한 경우에만 환불을 받음
* 수익은 존재하는 경우 재무부로, 그렇지 않으면 메이커에게 전송됨

**Cyfrin:** 확인함.


### `BetFactory::setPool`은 입력 풀이 합법적인 AaveV3 풀이고 입력 토큰을 지원하는지 검증해야 함

**설명:** `BetFactory::setPool`은 입력 풀이 합법적인 AaveV3 풀이고 입력 토큰을 지원하는지 검증해야 합니다. 이는 다음과 같이 수행할 수 있습니다:

1) `BetFactory::constructor`는 AaveV3 배포된 `PoolAddressesProvider` 컨트랙트의 주소를 입력으로 받아 불변 변수 `AAVE_ADDRESSES_PROVIDER`에 저장해야 합니다; Base 메인넷에서는 `0xe20fCBdBfFC4Dd138cE8b2E6FBb6CB49777ad64D`입니다.

2) `BetFactory::setPool`은 입력 `_pool`이 `PoolAddressesProvider::getPool`과 일치하는지 확인해야 합니다.

3) `BetFactory::setPool`은 `Pool::getReserveAToken`을 호출하고 반환 값이 `address(0)`이 아닌지 확인하여 입력 `_token`이 풀의 연관된 기본 토큰인지 확인해야 합니다.

4) 선택적으로 `IAToken::UNDERLYING_ASSET_ADDRESS`를 통해서도 검증합니다.

**권장 완화 조치:** 기존 테스트 스위트와 작동하고 M-3 발견 사항도 해결하는 잠재적인 솔루션:

1) 먼저 `BetFactory.sol`에 다음 추가 include를 추가합니다:
```solidity
import {IPool} from "@aave-dao/aave-v3-origin/src/contracts/interfaces/IPool.sol";
import {IAToken} from "@aave-dao/aave-v3-origin/src/contracts/interfaces/IAToken.sol";
import {IPoolAddressesProvider} from "@aave-dao/aave-v3-origin/src/contracts/interfaces/IPoolAddressesProvider.sol";
```

2) `BetFactory` 내부에서 이 상수와 세 가지 새로운 오류를 정의합니다:
```solidity
contract BetFactory is Ownable {
    // Base mainnet Aave V3 PoolAddressesProvider
    IPoolAddressesProvider public constant AAVE_ADDRESSES_PROVIDER =
        IPoolAddressesProvider(0xe20fCBdBfFC4Dd138cE8b2E6FBb6CB49777ad64D);

    error InvalidPool();
    error TokenNotSupported();
    error ATokenMismatch();
```

3) `BetFactory::setPool`에 대해 이 새 버전을 사용합니다:
```solidity
    function setPool(address _token, address _pool) external onlyOwner {
        // Allow setting to zero (disable Aave for this token)
        if (_pool == address(0)) {
            tokenToPool[_token] = address(0);
            emit PoolConfigured(_token, address(0));
            return;
        }

        // Validate against canonical registry
        if (_pool != AAVE_ADDRESSES_PROVIDER.getPool()) {
            revert InvalidPool();
        }

        // Verify token is listed
        address aToken = IPool(_pool).getReserveAToken(_token);
        if (aToken == address(0)) {
            revert TokenNotSupported();
        }

        // Verify bidirectional relationship
        if (IAToken(aToken).UNDERLYING_ASSET_ADDRESS() != _token) {
            revert ATokenMismatch();
        }

        tokenToPool[_token] = _pool;
        emit PoolConfigured(_token, _pool);
    }
```

**WannaBet:** 커밋 [70e1565](https://github.com/gskril/wannabet-v2/commit/70e1565b391992b7ea8b11f2cc59195478a69212)에서 수정됨.

**Cyfrin:** 확인함.


### 사용되지 않는 오류 `IBet::InvalidAmount`

**설명:** 사용자 정의 오류 `InvalidAmount`는 `IBet`에 선언되어 있지만 구현 어디에서도 사용되지 않습니다. 제거하거나 검증하려는 금액 확인에 사용하는 것을 고려하십시오.

**WannaBet:** 커밋 [6bddf7f](https://github.com/gskril/wannabet-v2/commit/6bddf7fc0be929fd10ac731ef87cebba9f7ee686)에서 수정됨.

**Cyfrin:** 확인함.


### 해결되지 않은 개발자 주석

**설명:** 제거하거나 해결해야 할 남은 개발자 주석이 있습니다:

`Bet::initialize#L62`:
```solidity
// Maybe can skip this and send it striaght to Aave ?
```

그리고

`IBbet.Bet#L29`
```solidity
// TODO: I feel like there's a more efficient way to represent the maker:taker ratio
```

첫 번째는 오해의 소지가 있으며(컨트랙트는 자금을 Aave로 직접 보내는 것이 아니라 자체적으로 보관해야 함, Aave 풀이 없을 수 있기 때문), 두 번째는 해결되지 않은 `TODO`입니다. 이를 제거하거나 명확히 하는 것을 고려하십시오.

**WannaBet:** 커밋 [6bddf7f](https://github.com/gskril/wannabet-v2/commit/6bddf7fc0be929fd10ac731ef87cebba9f7ee686)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage
## 가스 최적화 (Gas Optimization)


### 지역 변수를 제거할 수 있는 경우 명명된 반환 변수 사용

**설명:** 지역 변수를 제거할 수 있는 경우 명명된 반환 변수를 사용하십시오:
* `BetFactory::createBet`
* `Bet::_status`

**WannaBet:** 커밋 [1c5fb9d](https://github.com/gskril/wannabet-v2/commit/1c5fb9db92808e9165ba0f13fb600ae4bd9f08f3)에서 수정됨.

**Cyfrin:** 확인함.


### 몇 개의 슬롯만 필요할 때 전체 구조체를 `storage`에서 `memory`로 복사하지 않음

**설명:** 몇 개의 슬롯만 필요할 때 전체 구조체를 `storage`에서 `memory`로 복사하지 마십시오:
* `Bet::cancel`은 스토리지에서 `status,maker,makerStake,takerStake,asset` 5개의 슬롯만 읽으면 되므로 2개의 스토리지 읽기를 절약할 수 있습니다.

되돌리기 확인이 트리거될 경우 코드가 사용되지 않는 스토리지 슬롯을 읽느라 많은 가스를 낭비하지 않도록, 필요한 스토리지 슬롯을 캐싱하는 것을 고려하십시오.

**WannaBet:** 인지함.


### 동일한 스토리지 읽기를 방지하기 위해 스토리지 슬롯 캐시

**설명:** 실행 중 값이 변경되지 않는 경우 동일한 스토리지 읽기를 방지하기 위해 스토리지 슬롯을 캐시하십시오.

예를 들어, `Bet::accept,cancel,resolve`는 `_aavePool`이 0이 아닐 가능성이 높으므로 이를 캐시해야 합니다. 이렇게 하면 `accept`에서 1개의 스토리지 읽기를, `resolve,cancel`에서 2개의 스토리지 읽기를 절약할 수 있습니다.

**WannaBet:** 커밋 [b8b4863](https://github.com/gskril/wannabet-v2/commit/b8b4863960cc3eeee3ccf017e4ac3d26f65959fe)에서 수정됨.

**Cyfrin:** 확인함.


### Aave에서 인출할 때 `type(uint256).max` 사용

**설명:** `Bet::resolve` 및 `cancel`에서 Aave 포지션을 청산할 때, 컨트랙트는 `amount` 매개변수로 현재 aToken 잔액을 사용하여 인출합니다:
```solidity
uint256 aTokenBalance = IERC20(_aavePool.getReserveAToken(b.asset))
    .balanceOf(address(this));
_aavePool.withdraw(b.asset, aTokenBalance, address(this));
```

포지션을 완전히 닫기 위한 Aave의 [권장 패턴](https://aave.com/docs/aave-v3/smart-contracts/pool?utm_source=chatgpt.com#write-methods-withdraw)은 `type(uint256).max`를 전달하는 것입니다. 이는 라운딩/인덱싱 엣지 케이스에 대해 더 강력합니다. 또한 `withdraw`가 인출된 금액을 반환하므로 외부 호출 하나를 제거합니다:
```solidity
uint256 aTokenBalance = _aavePool.withdraw(b.asset, type(uint256).max, address(this));
```

**WannaBet:** 커밋 [b060cf6](https://github.com/gskril/wannabet-v2/commit/b060cf65724fc00d54b6440454261f63d662cc57)에서 수정됨.

**Cyfrin:** 확인함.


### `Bet::resolve`에서 중복 타임스탬프 확인 제거

**설명:** `Bet::resolve`에는 다음과 같은 되돌리기 확인이 있습니다:
```solidity
// Make sure the bet is active
if (_status(b) != IBet.Status.ACTIVE || block.timestamp > b.resolveBy) {
    revert InvalidStatus();
}
```

하지만 `_status(b)` 호출은 이미 `block.timestamp > b.resolveBy`를 확인하고 되돌리기를 트리거하는 `EXPIRED` 상태를 반환하므로, 여기서 동일한 타임스탬프 확인을 다시 하는 것은 중복입니다.

**WannaBet:** 커밋 [45afa44](https://github.com/gskril/wannabet-v2/commit/45afa44a0adf423a2c2775c22d9f99e0ce555bbc)에서 수정됨.

**Cyfrin:** 확인함.

\clearpage

