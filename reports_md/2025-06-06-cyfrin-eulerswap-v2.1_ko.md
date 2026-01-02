**수석 감사관**

[Immeas](https://twitter.com/0ximmeas)

[BladeSec](https://x.com/BladeSec)

[ChainDefenders](https://x.com/ChDefendersEth) ([0x539](https://x.com/1337web3) & [PeterSR](https://x.com/PeterSRWeb3))


---

# 발견 사항 (Findings)
## 저위험 (Low Risk)


### 프로토콜 수수료 수령자 업데이트가 이전 EulerSwap 인스턴스에 적용되지 않음

**설명:** `protocolFeeRecipient` 값은 배포 시점에 각 `EulerSwap` 인스턴스의 매개변수에 저장됩니다. `ProtocolFee` 컨트랙트의 [`ProtocolFee::setProtocolFeeRecipient`](https://github.com/euler-xyz/euler-swap/blob/1022c0bb3c034d905005f4c5aee0932a66adf4f8/src/utils/ProtocolFee.sol#L21-L23) 함수를 호출하여 `protocolFeeRecipient`를 업데이트하면, 변경 사항은 업데이트 후 배포되는 새로운 `EulerSwap` 인스턴스에만 영향을 미칩니다. 기존 `EulerSwap` 인스턴스는 배포 중에 매개변수에 포함되어 동적으로 참조되지 않으므로 이전 `protocolFeeRecipient` 값을 유지합니다.

**파급력:** 이로 인해 이전 `EulerSwap` 인스턴스가 구식 수령자 주소로 프로토콜 수수료를 계속 전송하여 잠재적인 재정적 손실이나 자금 관리 부실로 이어질 수 있는 불일치가 발생합니다. 또한 프로토콜 소유자가 영향을 받는 `EulerSwap` 인스턴스를 수동으로 업데이트하거나 재배포하여 새로운 수령자 주소와 일치시켜야 하므로 운영상의 복잡성도 초래합니다.

**개념 증명 (Proof of Concept):** `HookFees.t.sol` 파일에 다음 테스트를 추가하십시오:
```solidity
function test_protocolFeeRecipientChange() public {
	// Define two protocol fee recipients
	address recipient1 = makeAddr("recipient1");
	address recipient2 = makeAddr("recipient2");

	// Set initial protocol fee and recipient in factory
	uint256 protocolFee = 0.1e18; // 10% of LP fee
	eulerSwapFactory.setProtocolFee(protocolFee);
	eulerSwapFactory.setProtocolFeeRecipient(recipient1);

	// Deploy pool1 with recipient1
	EulerSwap pool1 = createEulerSwapHookFull(
		60e18,
		60e18,
		0.001e18,
		1e18,
		1e18,
		0.4e18,
		0.85e18,
		protocolFee,
		recipient1
	);

	// Verify pool1's protocolFeeRecipient
	IEulerSwap.Params memory params1 = pool1.getParams();
	assertEq(params1.protocolFeeRecipient, recipient1);

	// Perform a swap in pool1
	uint256 amountIn = 1e18;
	assetTST.mint(anyone, amountIn);
	vm.startPrank(anyone);
	assetTST.approve(address(minimalRouter), amountIn);
	bool zeroForOne = address(assetTST) < address(assetTST2);
	minimalRouter.swap(pool1.poolKey(), zeroForOne, amountIn, 0, "");
	vm.stopPrank();

	// Check that fees were sent to recipient1
	uint256 feeCollected1 = assetTST.balanceOf(recipient1);
	assertGt(feeCollected1, 0);
	assertEq(assetTST.balanceOf(recipient2), 0);

	// Change protocolFeeRecipient to recipient2 in factory
	eulerSwapFactory.setProtocolFeeRecipient(recipient2);

	// Perform another swap in pool1
	assetTST.mint(anyone, amountIn);
	vm.startPrank(anyone);
	assetTST.approve(address(minimalRouter), amountIn);
	minimalRouter.swap(pool1.poolKey(), zeroForOne, amountIn, 0, "");
	vm.stopPrank();

	// Check that additional fees were sent to recipient1, not recipient2
	uint256 newFeeCollected1 = assetTST.balanceOf(recipient1);
	assertGt(newFeeCollected1, feeCollected1); // recipient1 received more fees
	assertEq(assetTST.balanceOf(recipient2), 0); // recipient2's balance unchanged
}
```

**권장 완화 방안:** `EulerSwap` 컨트랙트를 리팩토링하여 배포 매개변수에 `protocolFeeRecipient`를 저장하는 대신 `EulerSwapFactory` 컨트랙트에서 동적으로 참조하도록 하십시오. 이렇게 하면 `protocolFeeRecipient`에 대한 모든 업데이트가 이전 및 신규 `EulerSwap` 인스턴스 모두에 즉시 반영됩니다.

**Euler:** 인지함.


### 구성된 준비금(reserves)이 청산에 대한 보호를 보장하지 않을 수 있음

**설명:** [EulerSwap 백서](https://github.com/euler-xyz/euler-swap/blob/1022c0bb3c034d905005f4c5aee0932a66adf4f8/docs/whitepaper/EulerSwap_White_Paper.pdf)에 따르면:

> 가능한 준비금의 공간은 LP가 보유한 실제 유동성의 양과 운영자가 보유할 수 있는 부채의 양에 따라 결정됩니다. EulerSwap AMM은 항상 스왑 서비스를 제공하는 데 사용되는 자산을 보유하지는 않기 때문에, 엄격한 실제 준비금이 아닌 가상 준비금(virtual reserves)과 부채 한도를 기반으로 계산을 수행합니다. 각 EulerSwap LP는 독립적인 가상 준비금 수준을 구성할 수 있습니다. 이러한 준비금은 AMM이 감당할 최대 부채 노출을 정의합니다. 청산을 방지하기 위해 유효 LTV(effective LTV)는 항상 대출 볼트의 차입 LTV 미만으로 유지되어야 합니다.

이는 적절한 구성 하에서 AMM이 과도한 담보 인정 비율(LTV)로 인한 청산 위험에 처하지 않아야 함을 의미합니다. 그러나 외부 요인으로 인해 이러한 보장이 훼손될 수 있습니다.

예를 들어, 동일한 담보 볼트의 다른 포지션이 청산되어 악성 부채를 남기면, 유동성에 사용되는 공유 담보의 가치가 떨어질 수 있습니다. 이는 EulerSwap AMM이 관리하는 포지션을 포함하여 해당 볼트를 사용하는 모든 포지션의 유효 LTV에 영향을 미칩니다.

공격자는 관련 없는 악성 부채로 인해 저하된 담보 가치를 활용하여 AMM의 포지션을 청산 임계값까지 밀어붙이는 스왑을 시작함으로써 이 상황을 악용할 수 있습니다.

**권장 완화 방안:** 각 차입 작업 후 명시적인 LTV 확인을 강제하고 `EulerSwapParams`에 구성 가능한 최대 LTV 매개변수를 도입하는 것을 고려하십시오. 또한 문서에서 Euler 계정 소유자가 볼트의 상태를 모니터링할 책임이 있으며, 담보에 악성 부채가 발생하거나 가치가 하락하는 경우(이는 스왑 활동과 독립적으로 발생할 수 있음) 선제적인 조치를 취해야 함을 명확히 하십시오.

**Euler:** 인지함. 스왑이 끝날 때 상태 계산을 수행하는 것은 가스 비용이 너무 많이 듭니다.

**Cyfrin:** Euler 팀은 [PR#93](https://github.com/euler-xyz/euler-swap/pull/93)에서 청산 우려와 관련된 몇 가지 사항을 문서에 추가했습니다.

\clearpage
## 정보성 (Informational)


### 사용되지 않는 임포트 및 오류

**설명:** 다음 임포트는 사용되지 않습니다:
- [`IEVC` 및 `IEVault`, `EulerSwapPeriphery.sol#L6-L7`](https://github.com/euler-xyz/euler-swap/blob/1022c0bb3c034d905005f4c5aee0932a66adf4f8/src/EulerSwapPeriphery.sol#L6-L7)

그리고 다음 오류는 사용되지 않습니다:
- [`InvalidQuery`, `EulerSwapFactory.sol#L36`](https://github.com/euler-xyz/euler-swap/blob/1022c0bb3c034d905005f4c5aee0932a66adf4f8/src/EulerSwapFactory.sol#L36)

제거하는 것을 고려하십시오.

**Euler:** 커밋 [`6109f53`](https://github.com/euler-xyz/euler-swap/pull/96/commits/6109f53c7b49be41867d9857d6bb0b6869761a02)에서 수정됨.

**Cyfrin:** 확인함.


### 상태 변경 시 이벤트 방출 누락

**설명:** [`ProtocolFee::setProtocolFee`](https://github.com/euler-xyz/euler-swap/blob/1022c0bb3c034d905005f4c5aee0932a66adf4f8/src/utils/ProtocolFee.sol#L14-L19) 및 [`ProtocolFee::setProtocolFeeRecipient`](https://github.com/euler-xyz/euler-swap/blob/1022c0bb3c034d905005f4c5aee0932a66adf4f8/src/utils/ProtocolFee.sol#L21-L23) 모두 `EulerSwapFactory`의 상태를 변경하지만 이벤트가 방출되지 않습니다.

더 나은 오프체인 추적 및 투명성을 위해 이러한 함수에서 이벤트를 방출하는 것을 고려하십시오.

**Euler:** 커밋 [`05a9148`](https://github.com/euler-xyz/euler-swap/pull/97/commits/05a91488dcaf27633c26699e98694d99b73d20ea)에서 수정됨.

**Cyfrin:** 확인함.


### 입력 유효성 검사 부족

**설명:** `EulerSwapFactory`의 [생성자](https://github.com/euler-xyz/euler-swap/blob/1022c0bb3c034d905005f4c5aee0932a66adf4f8/src/EulerSwapFactory.sol#L44-L50)에는 `evkFactory_`, `eulerSwapImpl_`, `feeOwner_`의 주소가 `address(0)`가 아닌지 확인하는 무결성 검사(sanity check)가 없는데, 이는 흔한 실수입니다.

이들이 `address(0)`가 아닌지 검증하는 것을 고려하십시오.

예:
```solidity
require(evkFactory_ != address(0), "Zero address");
require(eulerSwapImpl_ != address(0), "Zero address");
require(feeOwner_ != address(0), "Zero address");
```

**Euler:** 인지함.

\clearpage
## 가스 최적화 (Gas Optimization)


### 스왑 시 불필요한 추가 스토리지 읽기

**설명:** EulerSwap 또는 Uniswap V4를 통해 스왑이 실행될 때 재진입 훅(reentrant hook)이 사용됩니다:

[`UniswapHook::nonReentrantHook`](https://github.com/euler-xyz/euler-swap/blob/1022c0bb3c034d905005f4c5aee0932a66adf4f8/src/UniswapHook.sol#L67-L80):

```solidity
modifier nonReentrantHook() {
    {
        CtxLib.Storage storage s = CtxLib.getStorage();
        require(s.status == 1, LockedHook());
        s.status = 2;
    }

    _;

    {
        CtxLib.Storage storage s = CtxLib.getStorage();
        s.status = 1;
    }
}
```

여기서 `CtxLib.getStorage()`는 스토리지 구조체를 로드하고 `status`를 `2`로 설정하기 위해 한 번 호출된 후, 나중에 `status`를 다시 `1`로 복원하기 위해 다시 호출됩니다.

나중에 스왑 흐름에서 동일한 스토리지 슬롯이 여러 번 다시 로드됩니다:

1. [`QuoteLib::calcLimits`](https://github.com/euler-xyz/euler-swap/blob/1022c0bb3c034d905005f4c5aee0932a66adf4f8/src/libraries/QuoteLib.sol#L70-L71)에서:

   ```solidity
   function calcLimits(IEulerSwap.Params memory p, bool asset0IsInput) internal view returns (uint256, uint256) {
       CtxLib.Storage storage s = CtxLib.getStorage();
       …
   }
   ```
2. [`QuoteLib::findCurvePoint`](https://github.com/euler-xyz/euler-swap/blob/1022c0bb3c034d905005f4c5aee0932a66adf4f8/src/libraries/QuoteLib.sol#L150-L155)에서:

   ```solidity
   function findCurvePoint(IEulerSwap.Params memory p, uint256 amount, bool exactIn, bool asset0IsInput)
       internal
       view
       returns (uint256 output)
   {
       CtxLib.Storage storage s = CtxLib.getStorage();
       …
   }
   ```
3. 그리고 [`UniswapHook::_beforeSwap`](https://github.com/euler-xyz/euler-swap/blob/1022c0bb3c034d905005f4c5aee0932a66adf4f8/src/UniswapHook.sol#L129)에서 준비금을 업데이트하기 직전에 다시:

   ```solidity
   CtxLib.Storage storage s = CtxLib.getStorage();
   ```

각 `getStorage()` 호출은 추가 SLOAD를 방출하여 스왑의 전체 가스 비용을 증가시킵니다.


재진입 방지 로직을 `_beforeSwap` 내에 직접 포함하고, 스토리지를 한 번만 가져온 다음, CurveLib 호출에서 사용할 준비금 값을 캐시하는 것을 고려하십시오. 예를 들어:

```solidity
function _beforeSwap(
    address,
    PoolKey calldata key,
    IPoolManager.SwapParams calldata params,
    bytes calldata
)
    internal
    override
    returns (bytes4, BeforeSwapDelta, uint24)
{
    IEulerSwap.Params memory p = CtxLib.getParams();

    // Single storage load and reentrancy guard
    CtxLib.Storage storage s = CtxLib.getStorage();
    require(s.status == 1, LockedHook());
    s.status = 2;

    // Cache reserves locally
    uint112 reserve0 = s.reserve0;
    uint112 reserve1 = s.reserve1;

    // … perform limit and curve-point calculations using reserve0/reserve1 …

    // Update reserves and release guard
    s.reserve0 = uint112(newReserve0);
    s.reserve1 = uint112(newReserve1);
    s.status = 1;

    return (BaseHook.beforeSwap.selector, returnDelta, 0);
}
```

이렇게 하면 중복된 스토리지 읽기를 제거하여 모든 스왑에서 가스 소비를 줄일 수 있습니다. `EulerSwap::swap` 흐름에도 동일하게 적용됩니다.


**Euler:** 인지함.


### 볼트 호출은 EVC로 직접 수행 가능

**설명:** [`FundsLib::depositAssets`](https://github.com/euler-xyz/euler-swap/blob/1022c0bb3c034d905005f4c5aee0932a66adf4f8/src/libraries/FundsLib.sol#L61-L114)에서 Euler Vault에 대한 두 번의 호출이 이루어집니다:

* [Line 92](https://github.com/euler-xyz/euler-swap/blob/1022c0bb3c034d905005f4c5aee0932a66adf4f8/src/libraries/FundsLib.sol#L92):

  ```solidity
  uint256 repaid = IEVault(vault).repay(amount > debt ? debt : amount, p.eulerAccount);
  ```

* [Line 104](https://github.com/euler-xyz/euler-swap/blob/1022c0bb3c034d905005f4c5aee0932a66adf4f8/src/libraries/FundsLib.sol#L104):

  ```solidity
  try IEVault(vault).deposit(amount, p.eulerAccount) {}
  ```

이 두 함수 호출은 모두 Vault에 정의된 `callThroughEVC` 제어자를 통해 EVC(Ethereum Vault Connector)를 거쳐 라우팅됩니다:

* [`EVault::repay`](https://github.com/euler-xyz/euler-vault-kit/blob/master/src/EVault/EVault.sol#L121):

  ```solidity
  function repay(uint256 amount, address receiver) public virtual override callThroughEVC use(MODULE_BORROWING) returns (uint256) {}
  ```

* [`EVault::deposit`](https://github.com/euler-xyz/euler-vault-kit/blob/master/src/EVault/EVault.sol#L86):

  ```solidity
  function deposit(uint256 amount, address receiver) public virtual override callThroughEVC use(MODULE_VAULT) returns (uint256) {}
  ```

각 호출은 Vault 컨트랙트를 통한 간접 호출로 인해 컨트랙트 점프 비용이 발생합니다. 이러한 오버헤드를 줄이기 위해 `FundsLib` 내의 다른 볼트 상호 작용에서 이미 수행된 것처럼 EVC에서 직접 이러한 작업을 호출할 수 있습니다.

**Euler:** 인지함.

