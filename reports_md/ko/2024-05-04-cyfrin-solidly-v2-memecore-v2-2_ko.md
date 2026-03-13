**수석 감사 (Lead Auditors)**

[Giovanni Di Siena](https://twitter.com/giovannidisiena)

**보조 감사 (Assisting Auditors)**

[Hans](https://twitter.com/hansfriese)


---

# 발견 사항 (Findings)
## 중간 위험 (Medium Risk)


### 과거 만료일로 잠금(Locks)을 생성할 수 있음

**설명:** [`SolidityV2ERC42069::lockToken`](https://github.com/SolidlyLabs/v2-core/blob/757b18ad05780d2af22018b0c2c9d59422dc59d3/contracts/SolidlyV2-memecore.sol#L387)의 조용한 오버플로로 인해 기간(duration)이 충분히 길면 잠금 해제 날짜가 과거가 될 수 있습니다. 이를 통해 호출자는 사기성 잠금을 생성하여 최대 기간 동안 잠갔다고 광고할 수 있지만 실제로는 즉시 인출할 수 있습니다. 그러나 소유자의 주소로 [`SolidityV2ERC42069::getLock(s)`](https://github.com/SolidlyLabs/v2-core/blob/757b18ad05780d2af22018b0c2c9d59422dc59d3/contracts/SolidlyV2-memecore.sol#L459-L471)를 호출하는 사람(아마도 UI에서)에게는 이것이 표시되므로 영향은 다소 제한적입니다. 공격자의 관점에서 확장 기능을 사용하여 마음대로 래핑(wrap around)하여 이러한 악의적인 의도를 숨길 수 있으면 이상적이지만, 날짜를 증가시킬 때 확인된 수학(checked math) 되돌리기(revert)로 인해 불가능합니다.

**영향:** 이 버그는 악용될 가능성이 높지만 영향은 제한적이므로 중간 심각도 발견으로 분류됩니다.

**개념 증명 (Proof of Concept):** `MultiTest.js`에 다음 테스트를 추가하십시오:
```javascript
it("lock can be created in the past", async function () {
  const { user1, test0, test1, router, pair } = await loadFixture(deploySolidlyV2Fixture);

  let token0 = test0;
  let token1 = test1;

  // Approve tokens for liquidity provision
  await token0.connect(user1).approve(router.address, ethers.constants.MaxUint256);
  await token1.connect(user1).approve(router.address, ethers.constants.MaxUint256);

  // Provide liquidity
  await router.connect(user1).addLiquidity(
    token0.address,
    token1.address,
    ethers.utils.parseUnits("100", 18),
    ethers.utils.parseUnits("100", 18),
    0,
    0,
    user1.address,
    ethers.constants.MaxUint256
  );

  const liquidityBalance = await pair.balanceOf(user1.address);

  let blockTimestamp = (await ethers.provider.getBlock('latest')).timestamp;

  let maxUint128 = ethers.BigNumber.from("340282366920938463463374607431768211455");

  // Lock LP tokens
  await pair.connect(user1).lockToken(user1.address, liquidityBalance, maxUint128.sub(blockTimestamp));


  let ret = await pair.getLock(user1.address, 0);
  expect(ret.date).to.be.eq(0);
});
  ```
**권장 완화 방법:** 덧셈 결과를 안전하지 않게 다운캐스팅하는 대신 덧셈을 수행하기 전에 타임스탬프를 `uint128`로 캐스팅하십시오:

```diff
locks[from].push(LockData({
    amount: uint128(amount - fee),
-     date: uint128(block.timestamp + duration)
+     date: uint128(block.timestamp) + duration
}));
```

**Solidly Labs:** 커밋 [14533e7](https://github.com/SolidlyLabs/v2-core/tree/14533e758f42009ca1d2cf4e98e2e7f33bcd4538)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다. 덧셈을 수행하기 전에 타임스탬프가 먼저 `uint128`로 캐스팅됩니다.

\clearpage
## 낮은 위험 (Low Risk)


### 토큰에 의해 차단된 계정은 수수료를 청구할 수 없음

**설명:** `SolidlyV2`에서 유동성 공급자는 풀에서 이루어진 거래에서 수수료를 받습니다. 이러한 수수료는 전송 시 동기화되므로 거래 당시 유동성 포지션을 보유한 계정에 귀속됩니다.

유동성 공급자가 수수료를 청구하고자 할 때 `SolidlyV2Pair::claimFees`를 호출합니다:
```solidity
if (feeState.feeOwed0 > 1) {
    amount0 = feeState.feeOwed0 - 1;
    _safeTransfer(token0, msg.sender, uint256(amount0));
    feesClaimedLP0 += amount0;
    feesClaimedTotal0 += amount0;
    feeState.feeOwed0 = 1;
}
```
여기서 토큰은 `msg.sender`에게 전송됩니다.

문제는 `USDC`와 같은 일부 토큰에는 일부 계정이 전송을 받거나 수행하는 것을 차단하는 차단 목록이 있다는 것입니다. `msg.sender`에게 이런 일이 발생하면 수수료를 절대 청구할 수 없으며 이 자금은 풀에 무기한 잠기게 됩니다.

**영향:** 계정이 예를 들어 `USDC`에 의해 차단되면 사용자는 수수료를 청구할 수 없습니다.

**권장 완화 방법:** `SolidlyV2Pair::claimFees`에 `address to` 매개변수를 추가하여 수수료를 다른 계정으로 전송할 수 있도록 하십시오.

**Solidly Labs:** 확인됨.

**Cyfrin:** 확인됨.


### 2단계 소유권 이전 고려

**설명:** `SolidtlyV2-memecore` 계약에서 소유자는 배포 시 `SolidlyV2Factory`에서 `msg.sender`로 설정됩니다.

또한 `SolidlyV2Factory::setOwner`를 사용하여 변경할 수 있습니다:
```solidity
    function setOwner(address _newOwner) external {
        require(msg.sender == owner);
        owner = _newOwner;
    }
```

여기서 새 주소가 올바르지 않으면 소유자를 잃을 위험이 있습니다. 이로 인해 새 copilot 할당 및 수수료 조정과 같은 일부 기능을 사용할 수 없게 됩니다.

**영향:** `owner`가 실수로 프로토콜의 통제 범위를 벗어난 계정에 부여될 수 있습니다.

**권장 완화 방법:** OpenZeppelin [`Ownable2Step`](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/access/Ownable2Step.sol)과 같이 `owner`가 먼저 `pendingOwner`를 설정한 다음 `pendingOwner`가 새 소유권을 수락할 수 있는 2단계 소유권 이전을 구현하는 것을 고려하십시오.

**Solidly Labs:** 확인됨.

**Cyfrin:** 확인됨.


### `SolidlyV2Pair::setPoolFee`는 `SolidlyV2Factory`에 정의된 수수료 설정자(fee setters)를 적절하게 고려하지 못함

**설명:** 새 `feeSetter`가 `SolidlyV2Factory`의 소유자에 의해 등록되면 풀 수수료를 팩토리에서 지정한 최소값 미만으로 설정할 수 있습니다. 이는 프로토콜이 팩토리 자체에서 명시적으로 수정해서만 이 값에 영향을 미칠 수 있는 반면, 수수료 설정자는 팩토리와 동기화되지 않도록 풀에서 직접 수정할 수 있기 때문에 풀 수수료가 항상 정의된 최소/최대 값 사이에 있어야 한다는 불변성을 깨뜨립니다.

```solidity
function setPoolFee(uint16 _poolFee) external {
    require(ISolidlyV2Factory(factory).isFeeSetter(msg.sender) || msg.sender == copilot, 'UA');
    if (msg.sender == copilot) {
        require(_poolFee >= ISolidlyV2Factory(factory).minFee() && !copilotRevoked); // minimum fee enforced for copilot
    } else {
        require(!protocolRevoked);
    }
    require(_poolFee <= 1000); // pool fee capped at 10%
    uint16 feeOld = poolFee;
    poolFee = _poolFee;
    emit SetPoolFee(feeOld, _poolFee);
}
```

또한 `SolidlyV2Factory`의 소유자가 `SolidlyV2Pair::revokeFeeRole`을 호출하면 등록된 수수료 설정자는 수수료 설정자에 대한 명시적 고려가 없음에도 불구하고 더 이상 `SolidlyV2Pair::setPoolFee`를 호출할 수 없습니다.

**영향:**
- 풀 수수료가 `SolidlyV2Factory::minFee`와 동기화되지 않을 수 있습니다.
- 프로토콜이 수수료 역할을 취소하면 수수료 설정자는 수수료를 설정할 수 없습니다.

**개념 증명 (Proof of Concept):**
```javascript
it("fee below min", async function () {
  const { user1, factory, pair } = await loadFixture(deploySolidlyV2Fixture);

  await factory.setFeeSetter(user1.address, true);
  pair.connect(user1).setPoolFee(0);
  await factory.minFee().then(minFee => console.log(`SolidlyV2Factory::minFee: ${minFee}`));
  await pair.poolFee().then(poolFee => console.log(`SolidlyV2Pair::poolFee: ${poolFee}`));

  await pair.revokeFeeRole();
  const minFee = await factory.minFee();
  await expect(pair.connect(user1).setPoolFee(minFee)).to.revertedWithoutReason();
});
```

**권장 완화 방법:** `SolidlyV2Pair::setPoolFee`에서 수수료 설정자의 호출을 팩토리 소유자와 구별하여 명시적으로 처리하고, 풀 수수료가 정의된 글로벌 최소값 미만으로 설정될 수 없도록 강제하십시오.

**Solidly Labs:** 확인됨. 이 두 가지는 의도된 설계입니다. `feeSetters`는 `minFee`의 제약을 받지 않으며, 취소(revoke)에 대해 프로토콜과 `feeSetter`는 동일하게 간주됩니다.

**Cyfrin:** 확인됨.

\clearpage
## 정보성 (Informational)


### 플래시 론(Flash loans)을 무료로 이용할 수 있음

**설명:** 다른 AMM 구현과 달리 `SolidlyV2Pair`는 각 기본 풀 토큰에 대해 독립적으로 수수료를 계산합니다. `amount0Out` 계산만 보여주면:
```solidity
if (amount0Out > 0) {
    uint256 amount1In = balance1 - (_reserve1 - amount1Out);
    uint256 fee1 = _updateFees(1, amount1In, _poolFee, _protocolRatio);
    _k(balance0, balance1 - fee1, uint256(_reserve0), uint256(_reserve1));
    _updateReserves(balance0, (balance1 - fee1));
    _emitSwap(0, amount1In, amount0Out, amount1Out, to);
}
```

단일 토큰(예: `token0`)만 교환하는 것이 가능하므로 `token0Out`만 0이 아닙니다. `amount1Out`이 `0`이므로 `uint256 amount1In = balance1 - (_reserve1 - amount1Out);` 계산은 `amount1In = 0`을 산출합니다. 따라서 수수료가 부과되지 않습니다.

**영향:** 이것은 발생 가능성이 높지만 플래시 스왑을 무료로 이용할 수 있다는 것의 영향은 프로젝트에서 결정할 문제입니다.

**개념 증명 (Proof of Concept):** `MultiTest.js`에 다음 테스트를 추가하십시오:
```javascript
it("charges no fee for flashloans", async function () {
const { user1, test0, test1, router, pair } = await loadFixture(deploySolidlyV2Fixture);

let token0 = test0;
let token1 = test1;


// Approve tokens for liquidity provision
await token0.connect(user1).approve(router.address, ethers.constants.MaxUint256);
await token1.connect(user1).approve(router.address, ethers.constants.MaxUint256);

// Provide liquidity
await router.connect(user1).addLiquidity(
  token0.address,
  token1.address,
  ethers.utils.parseUnits("100", 18),
  ethers.utils.parseUnits("100", 18),
  0,
  0,
  user1.address,
  ethers.constants.MaxUint256
);

const FlashRecipient = await ethers.getContractFactory("FlashRecipient");
const flashRecipient = await FlashRecipient.deploy(token0.address, pair.address);

// doesn't revert
await flashRecipient.takeFlashloan();
});
```

`contracts/test/FlashRecipient.sol`에 다음 파일 포함:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import {SolidlyV2Pair} from "../SolidlyV2Pair.sol";
import {IERC20} from "./IERC20.sol";

contract FlashRecipient {

    IERC20 public token0;
    SolidlyV2Pair public pair;

    constructor(address _token0, address _pair) {
        token0 = IERC20(_token0);
        pair = SolidlyV2Pair(_pair);
    }

    function takeFlashloan() external {
        pair.swap(1e18, 0, address(this), "data");
    }

    function solidlyV2Call(address , uint256 amount0Out, uint256, bytes memory) external returns (bool) {
        // no fee being paid
        token0.transfer(msg.sender, amount0Out);
        return true;
    }
}
```


**권장 완화 방법:** 양쪽 모두에 대한 수수료를 계산하되 변경된 경우에만 `_updateFees()`를 호출하십시오.

**Solidly Labs:** 확인됨. 플래시 론은 큰 시장이 아니며, 특히 밈 코인의 경우 무료로 유지하는 것의 잠재적 이점이 더 많다고 봅니다.

**Cyfrin:** 확인됨.


### 상태 변경에 대해 방출되는 이벤트 부족

**설명:** 이벤트는 오프체인에서 계약 변경을 추적하는 데 유용합니다. 따라서 상태 변경에 대한 이벤트를 방출하는 것은 온체인 상태의 오프체인 모니터링 및 추적에 매우 중요합니다.

**영향:** 모니터링 및 오프체인 추적에서 중요한 상태 변경을 놓칠 수 있습니다.

**권장 완화 방법:** 다음에 대한 이벤트 방출을 고려하십시오:
* `SolidlyV2Factory::setOwner`
* `SolidlyV2Factory::setFeeCollector`
* `SolidlyV2Factory::setFeeSetter`
* `SolidlyV2Pair::setCopilot`
* `SolidlyV2Pair::revokeFeeRole`

또한 토큰이 소각될 때(그러나 수수료 발생을 위해 추적됨) 특별한 이벤트를 방출하는 것을 고려하십시오:
* `SolidlyV2ERC42069::transferZero`
* `SolidlyV2ERC42069::transferZeroFrom`

**Solidly Labs:** 부분적으로 수정됨. 가장 중요한 이벤트는 다루었으며 팩토리에 `OwnerChanged`를 추가했습니다. 우리는 바이트코드 공간에 매우 제약을 받고 있습니다.

**Cyfrin:** 확인됨.


### LP 토큰의 2차 시장은 문제가 있음

**설명:** 수수료가 LP 토큰에 누적되는 다른 AMM 구현과 달리 Solidly V2는 수수료가 LP 토큰당 `feeGrowthGlobal0`/`1`에서 추적되는 발생 시스템을 구현합니다. 각 유동성 공급자는 `SolidlyV2Pair::claimFees`를 호출하여 발생한 수수료를 수집할 수 있으며, 유동성 공급자가 발생한 수수료에 액세스하기 위해 포지션을 소각할 필요가 없다는 이점이 있습니다.

수수료 발생은 잔액을 변경하는 모든 작업(수수료 청구, 발행, 잠금, 그리고 가장 중요한 LP 토큰 또는 잠기거나 소각된 포지션의 전송 시)과 동기화됩니다. 즉, 발생한 수수료는 그 당시 유동성 포지션을 보유한 계정에 귀속됩니다.

따라서 이러한 토큰이 2차 시장에서 거래되거나 담보로 사용되는 경우 에스크로에 있는 동안 발생한 수수료는 손실됩니다. 이러한 계약은 수정 및 Solidly V2 Memecore 시스템에 대한 지식 없이는 수수료를 청구할 수 없기 때문입니다.

**영향:** Solidly V2 자체 내 또는 다른 에스크로/담보 스타일 계약과 같은 AMM에서 Solidly V2 Memecore LP 토큰을 사용하면 수수료가 손실됩니다.

**권장 완화 방법:** 문서에서 이 동작을 명확히 하십시오. LP 토큰이 모든 종류의 풀에서 안전하게 사용되려면 ERC-4626 스타일 래퍼 볼트가 개발되어야 합니다. 잠기거나/소각된 포지션은 어차피 거래하려면 맞춤형 구현이 필요하며 개발 시 이를 처리할 것이므로 크게 우려되지 않습니다.

**Solidly Labs:** 확인됨. 어쨌든 밈 코인 LP에 대한 2차 시장은 거의 존재하지 않습니다. 밈 LP 토큰의 가장 중요한 기능은 핵심 계약에서 직접 사용할 수 있습니다.

**Cyfrin:** 확인됨.


### 사용자는 짧은 기간 잠금으로 유동성을 숨길 수 있음

**설명:** SolidlyV2-memecore의 주요 기능 중 하나는 일정 기간 동안 유동성을 잠그는 기능입니다. 이것은 유동성을 `LockBox` 계약으로 전송하며 만료(잠금 시 지정됨)될 때까지 잠깁니다.

이 유동성은 보유자에 대해 여전히 수수료가 발생하지만 기술적으로는 `LockBox` 계약이 보유합니다. 따라서 `balanceOf`는 잠금이 있는 계정에 대해 `0`을 표시합니다.

사용자가 `0` 기간으로 잠금을 설정한 다음 유동성이 필요할 때 잠금을 인출할 수 있으므로 유동성을 "숨기는" 데 사용될 수 있습니다.

**영향:** 일반적인 ERC20 `balanceOf`에는 표시되지 않으므로 사용자는 잠금을 사용하여 유동성을 "숨길" 수 있습니다. 그러나 이를 악용할 명확한 방법이 없는 것으로 보이므로 이는 정보 제공용입니다.

**권장 완화 방법:** 이것이 우려되는 경우 `balanceOf`의 일부로 잠긴 유동성을 포함하는 것을 고려하십시오. 그러나 토큰이 `LockBox` 잔액과 계정 잔액 모두에서 이중으로 계산되므로 ERC20 회계가 깨질 것입니다.

**Solidly Labs:** 확인됨. 사용자는 만료되었거나 만료에 가까운지 확인할 수 있습니다.

**Cyfrin:** 확인됨.


### 잠금 생성은 선행 매매(front-run)될 수 있음

**설명:** `SolidlyV2ERC42069::lockToken`은 먼지(dust) 잠금의 전송으로 선행 매매될 수 있습니다. 이로 인해 잠금이 생성되는 인덱스가 발신자가 트랜잭션을 보낼 때 예상했던 것과 다를 수 있습니다.

최악의 경우 잘못된 잠금이 인출, 연장 또는 분할될 수 있습니다. 발신자가 주의를 기울이면 쉽게 해결할 수 있지만 한 번의 TX에 대한 가스 비용을 낭비할 수 있습니다.

**영향:** 잠금이 생성/분할/전송되는 인덱스가 예상과 다를 수 있습니다.

**권장 완화 방법:** 사용자가 방출된 이벤트를 사용하여 잠금이 생성된 인덱스를 확인해야 함을 문서에 언급하는 것을 고려하십시오.

**Solidly Labs:** 확인됨.

**Cyfrin:** 확인됨.

\clearpage
## 가스 최적화 (Gas Optimizations)


### 도메인 구분자를 캐시하고 체인 ID가 변경된 경우에만 다시 계산

가스 효율성을 높이기 위해 계약 생성 시 현재 체인 ID를 캐시하고 체인 ID 변경이 감지된 경우(즉, `block.chainid` != 캐시된 체인 ID)에만 도메인 구분자를 다시 계산하는 것이 좋습니다. [`Solmate::ERC20`](https://github.com/transmissions11/solmate/blob/c892309933b25c03d32b1b0d674df7ae292ba925/src/tokens/ERC20.sol) 구현에서 예를 볼 수 있습니다.

**Solidly Labs:** 확인됨.

**Cyfrin:** 확인됨.

