**수석 감사자**

[Immeas](https://twitter.com/0ximmeas)

[Jorge](https://x.com/TamayoNft)

---

# 결과 (Findings)
## 치명적 위험 (Critical Risk)


### CCIP 메시지 처리에서 소스(source) 유효성 검사 누락

**설명:** YieldFi는 Chainlink CCIP를 통합하여 수익 토큰(`YToken`)의 크로스 체인 전송을 용이하게 합니다. 이 기능은 `BridgeCCIP` 컨트랙트에서 처리하며, 이러한 전송에 대한 토큰 회계를 관리합니다.

그러나 [`BridgeCCIP::_ccipReceive`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/bridge/ccip/BridgeCCIP.sol#L160-L181) 함수에는 소스 체인에서 메시지를 보낸 사람에 대한 유효성 검사가 없습니다:
```solidity
/// handle a received message
function _ccipReceive(Client.Any2EVMMessage memory any2EvmMessage) internal override {
    bytes memory message = abi.decode(any2EvmMessage.data, (bytes)); // abi-decoding of the sent text
    BridgeSendPayload memory payload = Codec.decodeBridgeSendPayload(message);
    bytes32 _hash = keccak256(abi.encode(message, any2EvmMessage.messageId));
    require(!processedMessages[_hash], "processed");

    processedMessages[_hash] = true;

    require(payload.amount > 0, "!amount");

    ...
}
```

결과적으로 공격자는 유효한 데이터를 포함한 악의적인 `Any2EVMMessage`를 생성하고 CCIP를 통해 `BridgeCCIP` 컨트랙트로 전송하여 임의의 토큰을 발행(minting)하거나 잠금 해제(unlocking)할 수 있습니다.


**파급력:** 공격자는 L1의 브리지에서 토큰을 고갈시키거나 L2에서 무제한의 토큰을 발행할 수 있습니다. 2단계 상환(redeem) 프로세스가 어느 정도 완화를 제공하지만, 이러한 악용은 여전히 프로토콜의 회계를 심각하게 방해하고 수익을 청구할 때 악용될 수 있습니다.

**권장되는 완화 방법:** 소스 체인의 신뢰할 수 있는 피어(peer)로부터만 메시지를 수락하도록 유효성 검사를 구현하는 것을 고려하십시오:
```solidity
mapping(uint64 sourceChain => mapping(address peer => bool allowed)) public allowedPeers;
...
function _ccipReceive(
    Client.Any2EVMMessage memory any2EvmMessage
) internal override {
    address sender = abi.decode(any2EvmMessage.sender, (address));
    require(allowedPeers[any2EvmMessage.sourceChainSelector][sender],"allowed");
    ...
```

**YieldFi:** 커밋 [`a03341d`](https://github.com/YieldFiLabs/contracts/commit/a03341d8103ba08473ea1cd39e64192608692aca)에서 수정되었습니다.

**Cyfrin:** 확인됨. `sender`가 이제 신뢰할 수 있는 발신자인지 확인됩니다.


### 디코딩 시 모든 CCIP 메시지가 revert됨

**설명:** YieldFi는 여러 메시징 프로토콜을 사용하여 크로스 체인 토큰 전송을 가능하게 하기 위해 기존 LayerZero 지원과 함께 Chainlink CCIP를 통합했습니다. 이를 지원하기 위해 사용자 정의 메시지 페이로드가 토큰 전송을 나타내는 데 사용됩니다. 이 페이로드는 [`Codec::decodeBridgeSendPayload`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/libs/Codec.sol#L22-L51)에서 다음과 같이 디코딩됩니다:
```solidity
(uint32 dstId, address to, address token, uint256 amount, bytes32 trxnType) = abi.decode(_data, (uint32, address, address, uint256, bytes32));
```
이 동일한 디코딩 로직이 CCIP 메시지 처리에 재사용됩니다.

그러나 Chainlink는 `dstId`에 `uint64`를 사용하며, 체인 ID(예: [Ethereum mainnet](https://docs.chain.link/ccip/directory/mainnet/chain/mainnet))는 모두 `uint32` 범위를 초과합니다. 예를 들어, 이더리움의 CCIP 체인 ID는 `5009297550715157269`이며 이는 `uint32`의 한계를 훨씬 넘습니다.

**파급력:** `uint64` 값을 `uint32`로 캐스팅할 때 오버플로로 인해 디코딩 중에 모든 CCIP 메시지가 revert됩니다. 컨트랙트를 업그레이드할 수 없으므로 실패한 메시지는 재시도할 수 없으며, 전송 로직에 따라 토큰이 잠기거나 소각되어 영구적인 자금 손실이 발생합니다.

**개념 증명 (Proof of Concept):** `CCIP Receive: Should handle received message successfully` 테스트에서 `dstId = 5009297550715157269`인 메시지를 처리하려고 시도하면 트랜잭션이 조용히 revert됩니다. Remix를 사용하여 64비트 값을 32비트 정수로 수동으로 디코딩할 때도 동일한 동작이 관찰됩니다.

**권장되는 완화 방법:** Chainlink 형식과 일치하도록 `dstId`의 타입을 `uint64`로 업데이트하는 것을 고려하십시오. 현재 LayerZero 통합에서 디코딩 후 `dstId`가 사용되지 않으므로 이 변경은 안전해야 합니다.

**YieldFi:** 커밋 [`14fc17a`](https://github.com/YieldFiLabs/contracts/commit/14fc17a46702bf0db0efb199c48e52530221612b)에서 수정되었습니다.

**Cyfrin:** 확인됨. `dstId`는 이제 `Codec.BridgeSendPayload`에서 `uint64`입니다.

\clearpage
## 높은 위험 (High Risk)


### YToken 출금 흐름에서 `Manager::redeem`에 잘못된 `owner` 전달됨

**설명:** YieldFi의 수익 토큰(`YTokens`)은 표준 ERC-4626 볼트보다 더 복잡한 출금 메커니즘을 구현합니다. 출금을 즉시 실행하는 대신 중앙 `Manager` 컨트랙트로 연기하여 오프체인 처리를 위해 대기열에 넣고 나중에 온체인에서 실행합니다.

모든 ERC-4626 볼트와 마찬가지로 제3자가 적절한 승인(allowance)이 있는 경우 사용자를 대신하여 출금 또는 상환을 시작할 수 있습니다.

그러나 [`YToken::_withdraw`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/tokens/YToken.sol#L161-L172)에서 잘못된 주소가 `manager.redeem` 함수에 전달됩니다. 동일한 문제가 [`YTokenL2::_withdraw`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/tokens/YTokenL2.sol#L170-L180)에도 존재합니다:
```solidity
// Override _withdraw to request funds from manager
function _withdraw(address caller, address receiver, address owner, uint256 assets, uint256 shares) internal override nonReentrant notPaused {
    require(receiver != address(0) && owner != address(0) && assets > 0 && shares > 0, "!valid");
    require(!IBlackList(administrator).isBlackListed(caller) && !IBlackList(administrator).isBlackListed(receiver), "blacklisted");
    if (caller != owner) {
        _spendAllowance(owner, caller, shares);
    }
    // Instead of burning shares here, just redirect to Manager
    // The share burning will happen during order execution
    // Don't update totAssets here either, as the assets haven't left the system yet
    // @audit-issue `msg.sender` passed as owner
    IManager(manager).redeem(msg.sender, address(this), asset(), shares, receiver, address(0), "");
}
```

이 호출에서 올바른 `owner`가 이미 `_withdraw`에 전달되었음에도 불구하고 `msg.sender`가 `owner`로 `manager.redeem`에 전달됩니다. 이는 `msg.sender == owner`일 때 예상대로 작동하지만 제3자가 소유자를 대신하여 행동하는 위임된 출금 시나리오에서는 실패합니다. 이러한 경우 `manager.redeem` 호출이 revert되거나, 더 나쁜 경우 `msg.sender`가 지분을 가지고 있는 경우 잘못된 사용자의 토큰이 소각될 수 있습니다.


**파급력:** 제3자가 다른 사용자를 대신하여 출금을 시작할 때(`caller != owner`), 잘못된 소유자가 `manager.redeem`에 전달됩니다. 이로 인해 호출이 revert되어 출금이 차단될 수 있습니다. 최악의 경우, `msg.sender`(호출자)도 지분을 보유하고 있다면 의도한 소유자 대신 그들의 토큰이 의도치 않게 소각될 수 있습니다.

**개념 증명 (Proof of Concept):** 다음 테스트를 `yToken.ts`의 `describe("Withdraw and Redeem")` 아래에 배치하십시오. 통과해야 하지만 `"!balance"` 오류와 함께 실패합니다:
```javascript
it("Should handle redeem request through third party", async function () {
  // Grant manager role to deployer for manager operations
  await administrator.grantRoles(MINTER_AND_REDEEMER, [deployer.address]);

  const sharesToRedeem = toN(50, 18); // 18 decimals for shares

  await ytoken.connect(user).approve(u1.address, sharesToRedeem);

  // Spy on manager.redeem call
  const redeemTx = await ytoken.connect(u1).redeem(sharesToRedeem, user.address, user.address);

  // Wait for transaction
  await redeemTx.wait();

  // to check if manager.redeem was called we can check the event of manager contract
  const events = await manager.queryFilter("OrderRequest");
  expect(events.length).to.be.greaterThan(0);
  expect(events[0].args[0]).to.equal(user.address); // owner, who's tokens should be burnt
  expect(events[0].args[1]).to.equal(ytoken.target); // yToken
  expect(events[0].args[2]).to.equal(usdc.target); // Asset
  expect(events[0].args[4]).to.equal(sharesToRedeem); // Amount
  expect(events[0].args[3]).to.equal(user.address); // Receiver
  expect(events[0].args[5]).to.equal(false); // isDeposit (false for redeem)
});
```

**권장되는 완화 방법:** `msg.sender`를 사용하는 대신 `YToken::_withdraw` 및 `YTokenL2::_withdraw` 모두에서 올바른 `owner`를 `manager.redeem`에 전달하십시오.

**YieldFi:** 커밋 [`adbb6fb`](https://github.com/YieldFiLabs/contracts/commit/adbb6fb27bd23cdedccdaf9c1f484f7780cb354c)에서 수정되었습니다.

**Cyfrin:** 확인됨. 이제 `owner`가 `manager.redeem`에 전달됩니다.

\clearpage
## 중간 위험 (Medium Risk)


### 주석 처리된 블랙리스트 확인으로 인해 제한된 전송 허용

**설명:** [`PerpetualBond::_update`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/PerpetualBond.sol#L508-L510)에서 블랙리스트에 없는 사용자 간의 전송을 제한하기 위한 라인이 현재 주석 처리되어 있습니다:

```solidity
function _update(address from, address to, uint256 amount) internal virtual override {
    // Placeholder for Blacklist check
    // require(!IBlackList(administrator).isBlackListed(from) && !IBlackList(administrator).isBlackListed(to), "blacklisted");
```

이는 `PerpetualBond` 토큰 전송에 대한 블랙리스트 집행을 효과적으로 비활성화합니다.

**파급력:** 블랙리스트에 포함된 주소는 의도된 액세스 제어 또는 규정 준수 제한을 우회하여 `PerpetualBond` 토큰을 자유롭게 보유하고 전송할 수 있습니다.

**권장되는 완화 방법:** `_update`에서 블랙리스트 확인의 주석 처리를 제거하여 블랙리스트 사용자에 대한 전송 제한을 시행하십시오.

**YieldFi:** 커밋 [`a820743`](https://github.com/YieldFiLabs/contracts/commit/a82074332cc1f57eba398100c3a43e8a70a4c8ce)에서 수정되었습니다.

**Cyfrin:** 확인됨. 블랙리스트 확인을 수행하는 라인의 주석 처리가 제거되었습니다.


### `Manager::_transferFee`가 `fee`가 0일 때 유효하지 않은 `feeShares`를 반환함

**설명:** 사용자가 `Manager::deposit`에 직접 입금할 때 프로토콜 수수료는 [`Manager::_transferFee`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/Manager.sol#L226-L242) 함수를 통해 계산됩니다:

```solidity
function _transferFee(address _yToken, uint256 _shares, uint256 _fee) internal returns (uint256) {
    if (_fee == 0) {
        return _shares;
    }
    uint256 feeShares = (_shares * _fee) / Constants.HUNDRED_PERCENT;

    IERC20(_yToken).safeTransfer(treasury, feeShares);

    return feeShares;
}
```

문제는 `_fee == 0`일 때 함수가 `0`을 반환하는 대신 전체 `_shares` 금액을 반환한다는 것입니다. 이는 [`Manager::_deposit`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/Manager.sol#L286-L296)의 다운스트림에서 잘못된 로직으로 이어지며, 여기서 결과가 전체 지분에서 차감됩니다:

```solidity
// transfer fee to treasury, already applied on adjustedShares
uint256 adjustedFeeShares = _transferFee(order.yToken, adjustedShares, _fee);

// Calculate adjusted gas fee shares
uint256 adjustedGasFeeShares = (_gasFeeShares * order.exchangeRateInUnderlying) / currentExchangeRate;

// transfer gas to caller
IERC20(order.yToken).safeTransfer(_caller, adjustedGasFeeShares);

// remaining shares after gas fee
uint256 sharesAfterAllFee = adjustedShares - adjustedFeeShares - adjustedGasFeeShares;
```

`_fee == 0`인 경우 `adjustedFeeShares` 값은 `adjustedShares`와 동일하게 되어 `adjustedGasFeeShares`가 0이 아니라고 가정할 때 `sharesAfterAllFee`가 언더플로(revert)됩니다.

**파급력:** 수수료가 0인 `Manager` 컨트랙트로의 입금은 가스 수수료도 차감되는 경우 revert됩니다. 가장 좋은 시나리오에서는 입금이 실패합니다. 최악의 경우(뺄셈이 확인 없이 통과되는 경우) 사용자에게 0의 지분이 적립될 수 있습니다.

**권장되는 완화 방법:** 다운스트림 계산이 올바르게 작동하도록 `_fee == 0`일 때 `0`을 반환하도록 `_transferFee`를 업데이트하십시오:

```diff
  if (_fee == 0) {
-     return _shares;
+     return 0;
  }
```

**YieldFi:** 커밋 [`6e76d5b`](https://github.com/YieldFiLabs/contracts/commit/6e76d5beee3ba7a49af6becc58a596a4b67841c3)에서 수정되었습니다.

**Cyfrin:** 확인됨. `_transferFee`는 이제 `_fee = 0`일 때 `0`을 반환합니다.


### `YtokenL2::previewMint` 및 `YTokenL2::previewWithdraw`가 사용자에게 유리하게 반올림함

**설명:** L2 `YToken` 컨트랙트의 경우 자산이 직접 관리되지 않습니다. 대신 볼트의 환율은 오라클에 의해 제공되며, L1의 환율을 신뢰할 수 있는 소스로 사용합니다.

이 아키텍처 선택은 `previewMint`, `previewDeposit`, `previewRedeem`, `previewWithdraw`와 내부 `_convertToShares` 및 `_convertToAssets`와 같은 함수의 사용자 정의 구현이 필요합니다. 이들은 로컬 회계 대신 오라클 제공 환율에 의존하도록 다시 구현되었습니다.

그러나 `previewMint`와 `previewWithdraw` 모두 현재 사용자에게 유리하게 반올림을 수행합니다:

- [`YTokenL2::previewMint`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/tokens/YTokenL2.sol#L249-L250):
  ```solidity
  // Calculate assets based on exchange rate
  return (grossShares * exchangeRate()) / Constants.PINT;
  ```
- [`YTokenL2::previewWithdraw`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/tokens/YTokenL2.sol#L261-L262):
  ```solidity
  // Calculate shares needed for requested assets based on exchange rate
  uint256 sharesWithoutFee = (assets * Constants.PINT) / exchangeRate();
  ```

이러한 동작은 가치 유출을 방지하기 위해 볼트에 유리하게 반올림할 것을 조언하는 [EIP-4626의 보안 권장 사항](https://eips.ethereum.org/EIPS/eip-4626#security-considerations)과 모순됩니다.

**파급력:** 사용자에게 유리하게 반올림함으로써 이러한 함수는 사용자가 받아야 할 것보다 약간 더 많은 지분이나 자산을 받을 수 있게 합니다. 2단계 출금 프로세스가 즉각적인 악용 가능성을 제한하지만, 이 반올림 오류는 특히 많은 트랜잭션이 있거나 자동화가 있는 경우 볼트에서 느리고 지속적인 가치 유출을 초래할 수 있습니다.

**권장되는 완화 방법:** `previewMint` 및 `previewWithdraw`를 볼트에 유리하게 반올림하도록 업데이트하십시오. 이는 [OpenZeppelin ERC-4626 구현](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/master/contracts/token/ERC20/extensions/ERC4626Upgradeable.sol#L177-L185)에서 사용된 접근 방식과 유사하게 명시적 반올림 방향을 가진 수정된 `_convertToShares` 및 `_convertToAssets` 함수를 채택하여 수행할 수 있습니다.

**YieldFi:** 커밋 [`a820743`](https://github.com/YieldFiLabs/contracts/commit/a82074332cc1f57eba398100c3a43e8a70a4c8ce)에서 수정되었습니다.

**Cyfrin:** 확인됨. 미리보기 함수는 이제 올바른 반올림 방향으로 `_convertToShares` 및 `_convertToAssets`를 활용합니다.


### `OracleAdapter`에서 L2 시퀀서 가동 시간 확인 누락

**설명:** L2에서 `YToken` 환율은 사용자 정의 Chainlink 오라클에 의해 제공됩니다. 환율은 [`OracleAdapter::fetchExchangeRate`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/OracleAdapter.sol#L52-L77)에서 쿼리됩니다:

```solidity
function fetchExchangeRate(address token) external view override returns (uint256) {
    address oracle = oracles[token];
    require(oracle != address(0), "Oracle not set");

    (, /* uint80 roundId */ int256 answer, , /* uint256 startedAt */ uint256 updatedAt /* uint80 answeredInRound */, ) = IOracle(oracle).latestRoundData();

    require(answer > 0, "Invalid price");
    require(updatedAt > 0, "Round not complete");
    require(block.timestamp - updatedAt < staleThreshold, "Stale price");

    // Get decimals and normalize to 1e18 (PINT)
    uint8 decimals = IOracle(oracle).decimals();

    if (decimals < 18) {
        return uint256(answer) * (10 ** (18 - decimals));
    } else if (decimals > 18) {
        return uint256(answer) / (10 ** (decimals - 18));
    } else {
        return uint256(answer);
    }
}
```

그러나 이 프로토콜은 [시퀀서가 가동 중](https://docs.chain.link/data-feeds/l2-sequencer-feeds)인지 확인하는 것이 중요한 Arbitrum 및 Optimism과 같은 L2 네트워크에 배포될 예정입니다. 이 확인이 없으면 시퀀서가 다운될 경우 L1에서 트랜잭션을 제출하는 고급 사용자에게 최신 라운드 데이터가 신선한 것처럼 보일 수 있지만 실제로는 오래된 데이터일 수 있습니다.

**파급력:** L2 시퀀서가 다운되면 오라클 데이터 업데이트가 중지됩니다. 실제로 오래된 가격이 신선한 것처럼 보여 잘못 의존될 수 있습니다. 이는 다운타임 동안 상당한 가격 변동이 발생할 경우 악용될 수 있습니다.

**권장되는 완화 방법:** 시퀀서 다운타임 동안 오래된 오라클 데이터 사용을 방지하기 위해 [Chainlink 예제](https://docs.chain.link/data-feeds/l2-sequencer-feeds#example-consumer-contract)에 표시된 대로 시퀀서 가동 시간 확인을 구현하는 것을 고려하십시오.

**YieldFi:** 커밋 [`bb26a71`](https://github.com/YieldFiLabs/contracts/commit/bb26a71e9c57685996f6c853af6df6ed961c2f98) 및 [`e9c160f`](https://github.com/YieldFiLabs/contracts/commit/e9c160fdfd6dd90650c9537fba73c17cb3c53ea5)에서 수정되었습니다.

**Cyfrin:** 확인됨. 시퀀서 가동 시간이 이제 L2에서 확인됩니다.


### 직접 YToken 입금은 최소 출금 임계값 미만의 자금을 잠글 수 있음

**설명:** [`Manager::deposit`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/Manager.sol#L134-L155)에는 [`Manager::_validate`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/Manager.sol#L125-L126) 내에서 최소 입금 금액을 시행하는 확인이 있습니다:

```solidity
uint256 normalizedAmount = _normalizeAmount(_yToken, _asset, _amount);
require(IERC4626(_yToken).convertToShares(normalizedAmount) >= minSharesInYToken[_yToken], "!minShares");
```

유사한 확인이 [상환 흐름](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/Manager.sol#L157-L197)에도 존재하며, 다시 [`Manager::_validate`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/Manager.sol#L130)를 통합니다:

```solidity
require(_amount >= minSharesInYToken[_yToken], "!minShares");
```

그러나 `YToken`에 직접 입금할 때는 그러한 최소값이 시행되지 않습니다. [`YToken::_deposit`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/tokens/YToken.sol#L140) 및 [`YTokenL2::_deposit`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/tokens/YTokenL2.sol#L150) 모두에서 유일한 요구 사항은 다음과 같습니다:

```solidity
require(receiver != address(0) && assets > 0 && shares > 0, "!valid");
```

결과적으로 사용자는 `minSharesInYToken[_yToken]`보다 적은 지분을 초래하는 금액을 입금할 수 있으며, 이는 `Manager`의 최소 출금 확인으로 인해 `Manager`를 통해 인출할 수 없어 사실상 자금이 잠기게 됩니다.

**파급력:** 사용자는 `YToken`에 직접 입금하여 최소 지분 임계값을 우회할 수 있습니다. 결과 지분 금액이 `Manager`를 통한 출금에 허용된 최소값 미만인 경우, 사용자는 포지션을 종료할 수 없습니다. 이는 의도치 않게 자금이 잠기고 사용자 경험을 저하시킬 수 있습니다.

**권장되는 완화 방법:** `YToken::_deposit` 및 `YTokenL2::_deposit`에서 `minSharesInYToken[_yToken]` 임계값을 시행하여 너무 작아서 인출할 수 없는 입금을 방지하는 것을 고려하십시오. 또한, 사용자가 인출할 수 없는 "먼지"(dust)를 남기지 않도록 출금 후 잔액을 검증하는 것을 고려하십시오(즉, 남은 지분이 `0`이거나 `> minSharesInYToken[_yToken]`이어야 함).

**YieldFi:** 커밋 [`221c7d0`](https://github.com/YieldFiLabs/contracts/commit/221c7d0644af8fcb4d229d3e95e45323dc6f99a6)에서 수정되었습니다.

**Cyfrin:** 확인됨. YToken 컨트랙트에서 최소 지분이 이제 확인됩니다. Manager는 또한 상환 후 먼지가 남지 않는지 확인합니다.

\clearpage
## 낮은 위험 (Low Risk)


### 하드코딩된 `extraArgs`는 CCIP 모범 사례를 위반함

**설명:** CCIP를 통해 크로스 체인 메시지를 보낼 때, Chainlink는 [모범 사례](https://docs.chain.link/ccip/best-practices#using-extraargs)에 설명된 대로 향후 업그레이드나 구성 변경을 허용하기 위해 `extraArgs` 매개변수를 변경 가능하게 유지할 것을 권장합니다.

그러나 이 권장 사항은 `extraArgs`가 하드코딩된 [`BridgeCCIP::send`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/bridge/ccip/BridgeCCIP.sol#L126-L133)에서 지켜지지 않았습니다:
```solidity
// Sends the message to the destination endpoint
Client.EVM2AnyMessage memory evm2AnyMessage = Client.EVM2AnyMessage({
    receiver: abi.encode(_receiver), // ABI-encoded receiver address
    data: abi.encode(_encodedMessage), // ABI-encoded string
    tokenAmounts: new Client.EVMTokenAmount[](0), // Empty array indicating no tokens are being sent
    // @audit-issue `extraArgs` hardcoded
    extraArgs: Client._argsToBytes(Client.EVMExtraArgsV2({ gasLimit: 200_000, allowOutOfOrderExecution: true })),
    feeToken: address(0) // For msg.value
});
```

**파급력:** `extraArgs`가 하드코딩되어 있으므로 향후 변경 사항이 발생하면 브리지 컨트랙트의 새 버전을 배포해야 합니다.

**권장되는 완화 방법:** `send` 함수에 매개변수로 전달하거나 구성 가능한 컨트랙트 스토리지에서 파생하여 `extraArgs`를 변경 가능하게 만드는 것을 고려하십시오.

**YieldFi:** 커밋 [`3cc0b23`](https://github.com/YieldFiLabs/contracts/commit/3cc0b2331c35327a43e95176ce6c5578f145c0ee) 및 [`fd4b7ab5`](https://github.com/YieldFiLabs/contracts/commit/fd4b7ab57a5ae2ac366b4d9d086eb372defc7f8c)에서 수정되었습니다.

**Cyfrin:** 확인됨. `extraArgs`는 이제 호출에 매개변수로 전달됩니다.


### 정적 `gasLimit`은 과다 지불을 초래할 것임

**설명:** [사용되지 않은 가스는 환불되지 않으므로](https://docs.chain.link/ccip/best-practices#setting-gaslimit) Chainlink는 실행 비용 과다 지불을 피하기 위해 `extraArgs` 매개변수 내에서 `gasLimit`을 신중하게 설정할 것을 권장합니다.

[`BridgeCCIP::send`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/bridge/ccip/BridgeCCIP.sol#L131)에서 `gasLimit`은 Chainlink의 기본값이기도 한 `200_000`으로 하드코딩되어 있습니다:

```solidity
extraArgs: Client._argsToBytes(Client.EVMExtraArgsV2({ gasLimit: 200_000, allowOutOfOrderExecution: true })),
```

이 하드코딩된 값은 토큰을 브리징하는 모든 사용자에게 직접적인 영향을 미치며, 목적지 체인에서의 실행 비용에 대해 지속적으로 과다 지불하게 됩니다.

**권장되는 완화 방법:** 더 효율적인 접근 방식은 Hardhat 또는 Foundry와 같은 도구를 사용하여 `_ccipReceive` 함수의 가스 사용량을 측정하고 그에 따라 `gasLimit`을 설정하는 것입니다(안전을 위해 여유분 추가). 이를 통해 프로토콜은 모든 크로스 체인 메시지에서 가스 비용을 과다 지불하는 것을 방지할 수 있습니다.

이 문제는 또한 `extraArgs`를 변경 가능하게 만드는 것의 중요성을 강화하므로, 실행 비용이 시간이 지남에 따라 변경되는 경우(예: [EIP-1884](https://eips.ethereum.org/EIPS/eip-1884)와 같은 프로토콜 업그레이드로 인해) 가스 한도 및 기타 매개변수를 조정할 수 있습니다.

**YieldFi:** 커밋 [`3cc0b23`](https://github.com/YieldFiLabs/contracts/commit/3cc0b2331c35327a43e95176ce6c5578f145c0ee)에서 수정되었습니다.

**Cyfrin:** 확인됨. `extraArgs`는 이제 호출에 매개변수로 전달됩니다.


### 확인되지 않은 `_receiver`는 복구할 수 없는 토큰 손실을 초래할 수 있음

**설명:** 사용자가 CCIP를 사용하여 YToken을 브리징할 때 [`BridgeCCIP::send`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/bridge/ccip/BridgeCCIP.sol#L117-L158)를 호출합니다. 이 함수에 전달되는 매개변수 중 하나는 `_receiver`이며, 이는 수신 체인의 대상 컨트랙트여야 합니다:

```solidity
function send(address _yToken, uint64 _dstChain, address _to, uint256 _amount, address _receiver) external payable notBlacklisted(msg.sender) notBlacklisted(_to) notPaused {
    require(_amount > 0, "!amount");
    require(lockboxes[_yToken] != address(0), "!token !lockbox");
    require(IERC20(_yToken).balanceOf(msg.sender) >= _amount, "!balance");
    require(_to != address(0), "!receiver");
    require(tokens[_yToken][_dstChain] != address(0), "!destination");

    bytes memory _encodedMessage = abi.encode(_dstChain, _to, tokens[_yToken][_dstChain], _amount, Constants.BRIDGE_SEND_HASH);

    // Sends the message to the destination endpoint
    Client.EVM2AnyMessage memory evm2AnyMessage = Client.EVM2AnyMessage({
        // @audit-issue `_receiver` not verified
        receiver: abi.encode(_receiver), // ABI-encoded receiver address
        data: abi.encode(_encodedMessage), // ABI-encoded string
        tokenAmounts: new Client.EVMTokenAmount[](0), // Empty array indicating no tokens are being sent
        extraArgs: Client._argsToBytes(Client.EVMExtraArgsV2({ gasLimit: 200_000, allowOutOfOrderExecution: true })),
        feeToken: address(0) // For msg.value
    });
```

그러나 `_receiver` 매개변수는 검증되지 않습니다. 사용자가 잘못되거나 악의적인 주소를 제공하면 메시지가 이를 처리할 수 없는 컨트랙트로 전달되어 브리징된 토큰의 복구할 수 없는 손실이 발생할 수 있습니다.

**권장되는 완화 방법:** 이전 결과에서 언급한 `peers` 매핑과 같은 신뢰할 수 있는 매핑에 대해 `_receiver` 주소를 검증하여 목적지 체인의 합법적인 컨트랙트에 해당하는지 확인하십시오.

**YieldFi:** 커밋 [`a03341d`](https://github.com/YieldFiLabs/contracts/commit/a03341d8103ba08473ea1cd39e64192608692aca)에서 수정되었습니다.

**Cyfrin:** 확인됨. `_receiver`는 이제 신뢰할 수 있는 피어인지 확인됩니다.


### 하드코딩된 CCIP `feeToken`이 LINK 할인 사용을 방지함

**설명:** `BridgeCCIP::send`에서 [`feeToken`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/bridge/ccip/BridgeCCIP.sol#L126-L133) 매개변수는 하드코딩되어 있습니다:
```solidity
// Sends the message to the destination endpoint
Client.EVM2AnyMessage memory evm2AnyMessage = Client.EVM2AnyMessage({
    receiver: abi.encode(_receiver), // ABI-encoded receiver address
    data: abi.encode(_encodedMessage), // ABI-encoded string
    tokenAmounts: new Client.EVMTokenAmount[](0), // Empty array indicating no tokens are being sent
    extraArgs: Client._argsToBytes(Client.EVMExtraArgsV2({ gasLimit: 200_000, allowOutOfOrderExecution: true })),
    // @audit-issue hardcoded fee token
    feeToken: address(0) // For msg.value
});
```

Chainlink CCIP는 기본 가스 토큰 또는 `LINK`를 사용하여 수수료를 지불하는 것을 지원합니다. `feeToken = address(0)`으로 하드코딩함으로써 프로토콜은 모든 사용자가 기본 가스 토큰으로 지불하도록 강제하여 유연성을 제거합니다.

이 설계 선택은 구현을 단순화하지만 비용에 영향을 미칩니다: CCIP는 `LINK` 사용 시 [10% 수수료 할인](https://docs.chain.link/ccip/billing#network-fee-table)을 제공하므로 `LINK`를 보유한 사용자는 이러한 할인된 수수료를 활용할 수 없습니다.

**권장되는 완화 방법:** 개별 비용 및 편의성 선호도에 따라 사용자가 선호하는 지불 토큰( `LINK` 또는 기본 가스)을 선택할 수 있도록 허용하는 것을 고려하십시오.

**YieldFi:** 커밋 [`3cc0b23`](https://github.com/YieldFiLabs/contracts/commit/3cc0b2331c35327a43e95176ce6c5578f145c0ee) 및 [`e9c160f`](https://github.com/YieldFiLabs/contracts/commit/e9c160fdfd6dd90650c9537fba73c17cb3c53ea5)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### Chainlink 라우터가 두 번 구성됨

**설명:** `BridgeCCIP`에는 CCIP 라우터 주소에 대한 전용 스토리지 슬롯 [`router`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/bridge/ccip/BridgeCCIP.sol#L32-L33)가 있습니다:

```solidity
contract BridgeCCIP is CCIPReceiver, Ownable {
    address public router;
```

이 값은 [`BridgeCCIP::setRouter`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/bridge/ccip/BridgeCCIP.sol#L69-L73)를 통해 관리자가 업데이트할 수 있습니다:

```solidity
function setRouter(address _router) external onlyAdmin {
    require(_router != address(0), "!router");
    router = _router;
    emit SetRouter(msg.sender, _router);
}
```

`router`는 [`BridgeCCIP::send`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/bridge/ccip/BridgeCCIP.sol#L157)에서 CCIP를 통해 메시지를 보내는 데 사용됩니다:

```solidity
IRouterClient(router).ccipSend{ value: msg.value }(_dstChain, evm2AnyMessage);
```

그러나 상속된 `CCIPReceiver` 컨트랙트는 이미 불변 라우터 주소(`i_ccipRouter`)를 정의하고 있으며, 이는 들어오는 CCIP 메시지가 올바른 라우터에서 시작되었는지 확인하는 데 사용됩니다.

이로 인해 불일치가 발생합니다: `BridgeCCIP.router`가 변경되면 컨트랙트는 새 라우터를 통해 메시지를 *보내기* 계속하지만, 원래의 불변 `i_ccipRouter`에서만 메시지를 *수신*합니다. 이 불일치는 크로스 체인 통신을 중단시키거나 메시지 전달을 작동하지 않게 만들 수 있습니다.

**권장되는 완화 방법:** `CCIPReceiver`의 라우터 주소는 불변이므로 라우터에 대한 향후 변경은 이미 `BridgeCCIP` 컨트랙트의 재배포를 필요로 합니다. 따라서 `BridgeCCIP`의 `router` 스토리지 슬롯과 `setRouter` 함수는 중복되며 오해의 소지가 있습니다. 둘 다 제거하고 `CCIPReceiver`에서 상속된 `i_ccipRouter` 값에만 의존하는 것이 좋습니다.

**YieldFi:** 커밋 [`3cc0b23`](https://github.com/YieldFiLabs/contracts/commit/3cc0b2331c35327a43e95176ce6c5578f145c0ee)에서 수정되었습니다.

**Cyfrin:** 확인됨. `router`가 제거되었고 상속된 컨트랙트의 `i_ccipRouter`가 사용됩니다.


### `PerpetualBond::setVestingPeriod`에서 베스팅 확인 누락

**설명:** `YToken`과 `PerpetualBond` 모두 구성 가능한 베스팅 기간을 통해 보상 베스팅을 지원합니다. 관리자는 `setVestingPeriod` 함수를 통해 이 기간을 업데이트할 수 있습니다. 그러나 두 컨트랙트가 베스팅 기간 변경을 검증하는 방식에는 불일치가 있습니다:

- [`YToken::setVestingPeriod`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/tokens/YToken.sol#L52-L56)에는 현재 베스팅 중인 보상이 없는지 확인하는 체크가 포함되어 있습니다:
  ```solidity
  function setVestingPeriod(uint256 _vestingPeriod) external onlyAdmin {
      require(getUnvestedAmount() == 0, "!vesting");
      require(_vestingPeriod > 0, "!vestingPeriod");
      vestingPeriod = _vestingPeriod;
  }
  ```

- [`PerpetualBond::setVestingPeriod`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/PerpetualBond.sol#L184-L188)에는 이 체크가 없습니다:
  ```solidity
  function setVestingPeriod(uint256 _vestingPeriod) external onlyAdmin {
      // @audit-issue no check for `getUnvestedAmount() == 0`
      require(_vestingPeriod > 0, "!vestingPeriod");
      vestingPeriod = _vestingPeriod;
      emit VestingPeriodUpdated(_vestingPeriod);
  }
  ```

이는 토큰이 여전히 베스팅 중인 동안에도 `PerpetualBond`의 베스팅 기간을 수정할 수 있음을 의미하며, 이는 일관되지 않거나 예상치 못한 베스팅 동작으로 이어질 수 있습니다.

**권장되는 완화 방법:** `YToken` 구현과 일치시키고 일관성을 보장하기 위해 베스팅 기간 업데이트를 허용하기 전에 `getUnvestedAmount() == 0`인지 확인하는 체크를 `PerpetualBond::setVestingPeriod`에 추가하십시오.

**YieldFi:** 커밋 [`f0bf88c`](https://github.com/YieldFiLabs/contracts/commit/f0bf88cb51a92a119cdde896c4b0118be1d1a031)에서 수정되었습니다.

**Cyfrin:** 확인됨. `unvestedAmount`가 이제 확인됩니다.


### `PerpetualBond::_validate`의 수익 청구에 대한 잔액 확인은 쉽게 우회될 수 있음

**설명:** [`PerpetualBond::_validate`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/PerpetualBond.sol#L312-L314)에는 사용자가 수익을 청구하기 전에 0이 아닌 잔액을 가지고 있는지 확인하는 체크가 있습니다:

```solidity
// Yield claim
require(balanceOf(_caller) > 0, "!bond balance"); // Caller must hold bonds to claim yield
require(accruedRewardAtCheckpoint[_caller] > 0, "!claimable yield"); // Must have claimable yield
```

그러나 이 확인은 `PerpetualBond` 토큰의 1 wei와 같은 사소한 양을 보유함으로써 우회될 수 있습니다. 더 의미 있는 확인은 컨트랙트의 다른 부분에서 가치 기반 임계값을 시행하는 방식과 유사하게 사용자의 잔액이 `minimumTxnThreshold`를 초과하는지 확인하는 것입니다.

본드로 변환된 값을 사용하여 `minimumTxnThreshold`와 비교하도록 잔액 확인을 업데이트하는 것을 고려하십시오:

```diff
- require(balanceOf(_caller) > 0, "!bond balance");
+ require(_convertToBond(balanceOf(_caller)) > minimumTxnThreshold, "!bond balance");
```

또한 [`PerpetualBond::requestYieldClaim`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/PerpetualBond.sol#L374-L378)이 이미 가치 기반 임계값 확인을 수행하므로 `accruedRewardAtCheckpoint[_caller]`에 대한 두 번째 확인은 중복됩니다:

```solidity
// Convert yield amount to bond tokens for threshold comparison
uint256 yieldInBondTokens = _convertToBond(claimableYieldAmount);

// Check if the yield claim is worth executing
require(yieldInBondTokens >= minimumTxnThreshold, "!min txn threshold");
```

이로 인해 `_validate`의 `accruedRewardAtCheckpoint` 확인은 불필요합니다.

**YieldFi:** 커밋 [`f0bf88c`](https://github.com/YieldFiLabs/contracts/commit/f0bf88cb51a92a119cdde896c4b0118be1d1a031)에서 수정되었습니다.

**Cyfrin:** 확인됨. 사용자가 토큰이 없어도(판매/전송) 수익을 가질 수 있으므로 잔액 확인이 제거되었습니다. 중복되므로 `_validate`의 수익 확인도 제거되었습니다.

\clearpage
## 정보성 (Informational)


### 수익 분배 후 `PerpetualBond.epoch`가 업데이트되지 않음

**설명:** [`PerpetualBond::distributeBondYield`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/PerpetualBond.sol#L215-L241)에서 호출자는 [`epoch + 1`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/PerpetualBond.sol#L220-L221)과 일치하는 `nonce`를 제공해야 합니다:
```solidity
function distributeBondYield(uint256 _yieldAmount, uint256 nonce) external notPaused onlyRewarder {
    require(nonce == epoch + 1, "!epoch");
```
그러나 그 후 `epoch`는 증가되지 않습니다. `epoch`를 증가시키는 것을 고려하십시오.

**YieldFi:** 커밋 [`5c1f0e7`](https://github.com/YieldFiLabs/contracts/commit/5c1f0e7a805caf1d0fddbc5a15c8b6797a424467)에서 수정되었습니다.

**Cyfrin:** 확인됨. `epoch`는 이제 새 `nonce`로 증가합니다.


### 주문이 `eligibleAt`에서 자격이 없음

**설명:** [`PerpetualBond::executeOrder`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/PerpetualBond.sol#L411) 및 [`Manager::executeOrder`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/Manager.sol#L208) 모두에서 실행된 주문이 여전히 자격이 있는지 확인하는 체크가 있습니다:
```solidity
require(block.timestamp > order.eligibleAt, "!waitingPeriod");
```
`eligibleAt`은 주문이 이 타임스탬프에 자격이 있어야 함을 나타내지만 확인은 이를 검증하지 않습니다. `>`를 `>=`로 변경하는 것을 고려하십시오:
```diff
- require(block.timestamp > order.eligibleAt, "!waitingPeriod");
+ require(block.timestamp >= order.eligibleAt, "!waitingPeriod");
```

**YieldFi:** 커밋 [`e9c160f`](https://github.com/YieldFiLabs/contracts/commit/e9c160fdfd6dd90650c9537fba73c17cb3c53ea5)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `_receiverGas` 확인이 최소 허용 값을 제외함

**설명:** LayerZero 브리지 컨트랙트 [`BridgeLR::send`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/bridge/BridgeLR.sol#L76) 및 [`BridgeMB::send`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/bridge/BridgeMB.sol#L66)에서 사용자가 충분한 `_receiverGas`를 제공했는지 확인하는 체크가 있습니다:

```solidity
require(_receiverGas > MIN_RECEIVER_GAS, "!gas");
```

변수 이름 `MIN_RECEIVER_GAS`는 지정된 금액이 *포함*되어야 함을 암시합니다. 즉, 최소 허용 값이 유효합니다. 그러나 현재 `>` 확인은 `MIN_RECEIVER_GAS` 자체를 제외합니다. 의미론적 기대와 일치시키기 위해 비교를 `>=`로 변경하는 것을 고려하십시오:

```diff
- require(_receiverGas > MIN_RECEIVER_GAS, "!gas");
+ require(_receiverGas >= MIN_RECEIVER_GAS, "!gas");
```

동일한 내용이 [`Bridge::setMIN_RECEIVER_GAS`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/bridge/Bridge.sol#L53) 호출 및 [`Bridge::quote`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/bridge/Bridge.sol#L85) 확인에도 적용됩니다.

**YieldFi:** 커밋 [`9aa242b`](https://github.com/YieldFiLabs/contracts/commit/9aa242b7351314fe07160e98699d8da14a1b9bc2)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 사용되지 않는 오류

**설명:** [`Common`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/libs/Common.sol#L5-L6) 라이브러리에 사용되지 않는 두 가지 오류가 있습니다:
```solidity
error SignatureVerificationFailed();
error BadSignature();
```
이들을 제거하는 것을 고려하십시오.

**YieldFi:** 커밋 [`9aa242b`](https://github.com/YieldFiLabs/contracts/commit/9aa242b7351314fe07160e98699d8da14a1b9bc2)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 향후 콜백 로직이 활성화될 경우의 잠재적 위험

**설명:** `Manager` 및 `PerpetualBond` 컨트랙트 모두 사용자 상호 작용을 위한 2단계 프로세스를 구현합니다. 이러한 호출의 일부로 사용자는 `_callback` 주소와 수반되는 `_callbackData`를 제공할 수 있습니다. 예를 들어, [`Manager::deposit`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/Manager.sol#L144)의 매개변수는 다음과 같습니다:

```solidity
function deposit(..., address _callback, bytes calldata _callbackData) external notPaused nonReentrant {
```

그러나 이러한 매개변수는 현재 요청이 저장될 때 전달되지 않습니다. [`Manager::deposit`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/Manager.sol#L153)의 뒷부분에 나와 있는 것처럼:

```solidity
uint256 receiptId = IReceipt(receipt).mint(msg.sender, Order(..., address(0), ""));
```

여기서 사용자가 제공한 값 대신 `address(0)`와 빈 `""`가 하드코딩됩니다.

나중에 `executeOrder` 흐름(예: [`Manager::executeOrder`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/Manager.sol#L219-L223))에서 콜백은 조건부로 실행됩니다:

```solidity
// Execute the callback
if (order.callback != address(0)) {
    (bool success, ) = order.callback.call(order.callbackData);
    require(success, "callback failed");
}
```

원래 사용자가 제공한 `_callback` 및 `_callbackData`가 전달되어 여기에서 사용된다면 심각한 보안 위험이 발생할 수 있습니다. 악의적인 사용자는 이를 악용하여 임의의 외부 호출을 실행하고 잠재적으로 `Manager` 또는 `PerpetualBond` 컨트랙트에 승인된 토큰을 훔칠 수 있습니다.

콜백 기능이 현재 의도되지 않은 경우, 향후 활성화될 위험을 피하기 위해 `_callback` 및 `_callbackData` 매개변수를 완전히 제거하거나 비활성화하는 것을 고려하십시오. 대안으로 나중에 콜백 지원이 추가될 경우 엄격한 유효성 검사 및 액세스 제어를 보장하십시오.


**YieldFi:** 인지함.


### 업그레이드 가능한 컨트랙트에서 `_disableInitializers` 누락

**설명:** YieldFi는 업그레이드 가능한 컨트랙트를 활용합니다. 구현 컨트랙트를 초기화하는 기능을 비활성화하는 것은 [모범 사례](https://docs.openzeppelin.com/upgrades-plugins/writing-upgradeable#initializing_the_implementation_contract)입니다.

모든 업그레이드 가능한 컨트랙트에 OpenZeppelin `_disableInitializers`가 있는 생성자를 추가하는 것을 고려하십시오:
```solidity
constructor() {
    _disableInitializers();
}
```

**YieldFi:** 커밋 [`584b268`](https://github.com/YieldFiLabs/contracts/commit/584b268a75a8f7c7f10eda46efaaa3ebbe4f0159)에서 수정되었습니다.

**Cyfrin:** 확인됨. 모든 업그레이드 가능한 컨트랙트에 `_disableInitializers`가 있는 생성자가 추가되었습니다.


### 사용되지 않는 임포트

**설명:** 다음 사용되지 않는 임포트를 제거하는 것을 고려하십시오:

- contracts/bridge/Bridge.sol [Line: 7](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/bridge/Bridge.sol#L7)
- contracts/bridge/Bridge.sol [Line: 9](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/bridge/Bridge.sol#L9)
- contracts/bridge/Bridge.sol [Line: 13](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/bridge/Bridge.sol#L13)
- contracts/bridge/Bridge.sol [Line: 15](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/bridge/Bridge.sol#L15)
- contracts/bridge/Bridge.sol [Line: 18](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/bridge/Bridge.sol#L18)
- contracts/bridge/Bridge.sol [Line: 20](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/bridge/Bridge.sol#L20)
- contracts/bridge/BridgeMB.sol [Line: 17](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/bridge/BridgeMB.sol#L17)
- contracts/bridge/ccip/BridgeCCIP.sol [Line: 4](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/bridge/ccip/BridgeCCIP.sol#L4)
- contracts/bridge/ccip/BridgeCCIP.sol [Line: 13](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/bridge/ccip/BridgeCCIP.sol#L13)
- contracts/core/Manager.sol [Line: 6](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/Manager.sol#L6)
- contracts/core/Manager.sol [Line: 15](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/Manager.sol#L15)
- contracts/core/Manager.sol [Line: 17](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/Manager.sol#L17)
- contracts/core/OracleAdapter.sol [Line: 6](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/OracleAdapter.sol#L6)
- contracts/core/PerpetualBond.sol [Line: 7](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/PerpetualBond.sol#L7)
- contracts/core/PerpetualBond.sol [Line: 13](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/PerpetualBond.sol#L13)
- contracts/core/interface/IPerpetualBond.sol [Line: 4](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/interface/IPerpetualBond.sol#L4)
- contracts/core/l1/LockBox.sol [Line: 10](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/l1/LockBox.sol#L10)
- contracts/core/l1/LockBox.sol [Line: 13](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/l1/LockBox.sol#L13)
- contracts/core/l1/Yield.sol [Line: 5](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/l1/Yield.sol#L5)
- contracts/core/l1/Yield.sol [Line: 10](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/l1/Yield.sol#L10)
- contracts/core/l1/Yield.sol [Line: 11](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/l1/Yield.sol#L11)
- contracts/core/l1/Yield.sol [Line: 12](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/l1/Yield.sol#L12)
- contracts/core/l1/Yield.sol [Line: 13](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/l1/Yield.sol#L13)
- contracts/core/l1/Yield.sol [Line: 14](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/l1/Yield.sol#L14)
- contracts/core/l1/Yield.sol [Line: 16](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/l1/Yield.sol#L16)
- contracts/core/tokens/YToken.sol [Line: 8](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/tokens/YToken.sol#L8)
- contracts/core/tokens/YToken.sol [Line: 14](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/tokens/YToken.sol#L14)
- contracts/core/tokens/YTokenL2.sol [Line: 12](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/tokens/YTokenL2.sol#L12)

**YieldFi:** 커밋 [`8264429`](https://github.com/YieldFiLabs/contracts/commit/826442914cb9829aa302dbaef0741659cc5a1a67)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 사용되지 않는 상수

**설명:** `Constants.sol`에 일부 사용되지 않는 상수가 있습니다. 다음을 제거하는 것을 고려하십시오:
* [#L21: `SIGNER_ROLE`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/libs/Constants.sol#L21)
* [#L38: `VESTING_PERIOD`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/libs/Constants.sol#L38)
* [#L41 `MAX_COOLDOWN_PERIOD`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/libs/Constants.sol#L41)
* [#L44: `MIN_COOLDOWN_PERIOD`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/libs/Constants.sol#L44)
* [#L47 `ETH_SIGNED_MESSAGE_PREFIX`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/libs/Constants.sol#L47)
* [#L50`REWARD_HASH`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/libs/Constants.sol#L50)
* [#L56-L59 `DEPOSIT`, `WITHDRAW`, `DEPOSIT_L2`, `WITHDRAW_L2`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/libs/Constants.sol#L56-L59)

**YieldFi:** 커밋 [`125ec4a`](https://github.com/YieldFiLabs/contracts/commit/125ec4a944c436e587d7380b8c4bf6232d3264aa)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 중요한 상태 변경 시 이벤트 방출 누락

**설명:** 다음 함수는 상태를 변경하지만 이벤트를 방출하지 않습니다. 다음에서 이벤트를 방출하는 것을 고려하십시오:


- [`Access::setAdministrator`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/administrator/Access.sol#L76)
- [`Administrator::cancelAdminRole`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/administrator/Administrator.sol#L109)
- [`Administrator::cancelTimeLockUpdate`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/administrator/Administrator.sol#L148)
- [`Bridge::setMIN_RECEIVER_GAS`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/bridge/Bridge.sol#L52)
- [`BridgeMB::setManager`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/bridge/BridgeMB.sol#L42)
- [`BridgeCCIP::setManager`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/bridge/ccip/BridgeCCIP.sol#L82)
- [`Manager::setTreasury`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/Manager.sol#L52)
- [`Manager::setReceipt`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/Manager.sol#L61)
- [`Manager::setCustodyWallet`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/Manager.sol#L71)
- [`Manager::setMinSharesInYToken`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/Manager.sol#L81)
- [`OracleAdapter::setStaleThreshold`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/OracleAdapter.sol#L48)
- [`LockBox::setManager`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/l1/LockBox.sol#L31)
- [`YToken::setManager`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/tokens/YToken.sol#L42)
- [`YToken::setYield`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/tokens/YToken.sol#L47)
- [`YToken::setVestingPeriod`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/tokens/YToken.sol#L52)
- [`YToken::setFee`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/tokens/YToken.sol#L62)
- [`YToken::setGasFee`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/tokens/YToken.sol#L72)
- [`YToken::updateTotalAssets`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/tokens/YToken.sol#L179)
- [`YTokenL2::setManager`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/tokens/YTokenL2.sol#L83)
- [`YTokenL2::setFee`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/tokens/YTokenL2.sol#L92)
- [`YTokenL2::setGasFee`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/tokens/YTokenL2.sol#L102)


**YieldFi:** 커밋 [`b978ddf`](https://github.com/YieldFiLabs/contracts/commit/b978ddfc6ba8299a6045fde5e065f5fc276c02f7)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `LockBox::unlock`에 대한 액세스가 최소 권한 원칙을 따르지 않음

**설명:** [`LockBox::unlock`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/l1/LockBox.sol#L97) 함수에는 [`onlyBridgeOrLockBox`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/administrator/Access.sol#L37-L40) 수정자가 있어 `BRIDGE_ROLE` 또는 `LOCKBOX_ROLE` 역할을 가진 호출자가 호출에 액세스할 수 있습니다.

그러나 이 함수는 브리지 컨트랙트에서만 호출됩니다. 최소 권한 원칙을 따르기 위해 `LOCKBOX_ROLE`에서 액세스 권한을 제거하는 것을 고려하십시오.

**YieldFi:** 커밋 [`f0c751a`](https://github.com/YieldFiLabs/contracts/commit/f0c751a25d3cf8d46661f7508b72193c88e6fc91)에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
## 가스 최적화 (Gas Optimization)


### `BridgeCCIP.isL1`은 불변(immutable)일 수 있음

**설명:** [`BridgeCCIP.isL1`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/bridge/ccip/BridgeCCIP.sol#L34)은 생성자에서만 [할당](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/bridge/ccip/BridgeCCIP.sol#L44)됩니다. 따라서 불변 값은 읽는 비용이 더 저렴하므로 불변으로 만들 수 있습니다.

`BridgeCCIP.isL1`을 불변으로 만드는 것을 고려하십시오.

**YieldFi:** 커밋 [`823b010`](https://github.com/YieldFiLabs/contracts/commit/823b010d74fd55fb88b31619c1a94dac2ef65ad3)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `PerpetualBond::_convertToBond`에서 읽은 `bondFaceValue`를 캐시할 수 있음

**설명:** 스토리지 값 `bondFaceValue`는 [`PerpetualBond::__convertToBond`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/PerpetualBond.sol#L291-L294)에서 두 번 읽힙니다:
```solidity
function _convertToBond(uint256 assetAmount) internal view returns (uint256) {
    if (bondFaceValue == 0) return 0; // Prevent division by zero
    return (assetAmount * 1e18) / bondFaceValue;
}
```
값을 캐시하여 한 번만 읽을 수 있습니다:
```solidity
function _convertToBond(uint256 assetAmount) internal view returns (uint256) {
    // cache read
    uint256 _bondFaceValue = bondFaceValue;
    if (_bondFaceValue == 0) return 0; // Prevent division by zero
    return (assetAmount * 1e18) / _bondFaceValue;
}
```

**YieldFi:** 커밋 [`823b010`](https://github.com/YieldFiLabs/contracts/commit/823b010d74fd55fb88b31619c1a94dac2ef65ad3)에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `YToken::_decimalsOffset` 및 `YTokenL2::_decimalsOffset`에서 불필요한 외부 호출

**설명:** [`YToken::_decimalsOffset`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/tokens/YToken.sol#L314-L316) 및 [`YTokenL2::_decimalsOffset`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/tokens/YTokenL2.sol#L314-L316)에서 기본 토큰의 소수점 자릿수가 쿼리됩니다:
```solidity
function _decimalsOffset() internal view virtual override returns (uint8) {
    return 18 - IERC20Metadata(asset()).decimals();
}
```
그러나 이 값은 이미 OpenZeppelin 기본 컨트랙트 `ERC4626Upgradeable`에 저장되어 있으므로 외부 호출 대신 사용할 수 있습니다.

**YieldFi:** 인지함.


### `Manager::executeOrder`에서 주문을 두 번 읽음

**설명:** [`Manager::executeOrder`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/Manager.sol#L207-L214)에서 주문 데이터는 영수증에서 가져옵니다:
```solidity
Order memory order = IReceipt(receipt).readOrder(_receiptId);
require(block.timestamp > order.eligibleAt, "!waitingPeriod");
require(_fee <= Constants.ONE_PERCENT, "!fee");
if (order.orderType) {
    _deposit(msg.sender, _receiptId, _amount, _fee, _gas);
} else {
    _withdraw(msg.sender, _receiptId, _amount, _fee, _gas);
}
```
그런 다음 주문은 [`Manager::_deposit`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/Manager.sol#L252-L253)에서 다시 읽힙니다:
```solidity
function _deposit(address _caller, uint256 _receiptId, uint256 _shares, uint256 _fee, uint256 _gasFeeShares) internal {
    Order memory order = IReceipt(receipt).readOrder(_receiptId);
```

그리고 [`Manager::_withdraw`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/Manager.sol#L327-L328)에서도:
```solidity
function _withdraw(address _caller, uint256 _receiptId, uint256 _assetAmountOut, uint256 _fee, uint256 _gasFeeShares) internal {
    Order memory order = IReceipt(receipt).readOrder(_receiptId);
```

이 추가 읽기는 불필요합니다. 대신 `Order memory order`를 `Manager::_deposit` 및 `Manager::_withdraw`에 매개변수로 전달하는 것을 고려하십시오. 이렇게 하면 영수증에서 데이터를 다시 읽는 것을 절약할 수 있습니다:
```solidity
function _deposit(..., Order memory order) internal {

function _withdraw(..., Order memory order) internal {
```

**YieldFi:** 커밋 [`823b010`](https://github.com/YieldFiLabs/contracts/commit/823b010d74fd55fb88b31619c1a94dac2ef65ad3)에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
