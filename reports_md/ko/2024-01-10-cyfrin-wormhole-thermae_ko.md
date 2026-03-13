**수석 감사자**

[Dacian](https://twitter.com/DevDacian)

**보조 감사자**



---

# 발견 사항 (Findings)
## 고위험 (High Risk)


### `pool.slot0`에서 파생된 환율을 사용하는 온체인 슬리피지 계산은 쉽게 조작될 수 있음

**설명:** [`pool.slot0`의 가격을 사용하는 온체인 슬리피지 계산](https://dacian.me/defi-slippage-attacks#heading-on-chain-slippage-calculation-can-be-manipulated)은 [쉽게 조작될 수 있어](https://solodit.xyz/issues/h-4-no-slippage-protection-during-repayment-due-to-dynamic-slippage-params-and-easily-influenced-slot0-sherlock-real-wagmi-2-git) 사용자가 의도한 것보다 적은 토큰을 받게 할 수 있습니다.

**영향:** 스왑 결과로 사용자가 의도한 것보다 적은 토큰을 받을 수 있습니다.

**개념 증명(Proof of Concept):** `Portico::calcMinAmount`는 스왑이 반환해야 하는 최소 토큰 양을 온체인에서 계산하려고 시도합니다. 이는 다음을 사용하여 수행됩니다:
1) L85: 사용자가 2가지 가능한 스왑에 대해 지정할 수 있는 `maxSlippageStart` 또는 `maxSlippageFinish` 매개변수를 입력으로 받음,
2) L135: `pool.slot0`에서 가격 정보를 읽어 현재 환율을 온체인에서 가져옴

문제는 [`pool.slot0`이 플래시 론을 사용하여 조작하기 쉽기 때문에](https://solodit.xyz/issues/h-02-use-of-slot0-to-get-sqrtpricelimitx96-can-lead-to-price-manipulation-code4rena-maia-dao-ecosystem-maia-dao-ecosystem-git) 슬리피지 계산에 사용되는 실제 환율이 사용자가 기대하는 것보다 훨씬 나쁠 수 있다는 것입니다; 사용자는 스왑에 대한 샌드위치 공격을 통해 지속적으로 착취당할 가능성이 매우 높습니다.

**권장 완화 조치:**
1. 온체인에서 가격 정보가 필요한 경우, 조작에 더 강한 가격 정보를 위해 `pool.slot0` 대신 [Uniswap V3 TWAP](https://docs.uniswap.org/concepts/protocol/oracle)을 사용하십시오 (참고: 이는 [Optimism에서는 동일한 수준의 보호를 제공하지 않습니다](https://docs.uniswap.org/concepts/protocol/oracle#oracles-integrations-on-layer-2-rollups)).
2. `maxSlippageStart` 및 `maxSlippageFinish` 대신 `minAmountReceivedStart` 및 `minAmountReceivedFinish` 매개변수를 사용하고 온체인 슬리피지 계산을 제거하십시오. 온체인에서 슬리피지를 계산하는 "안전한" 방법은 없습니다. 사용자가 % 슬리피지 매개변수를 지정하면 오프체인에서 정확한 최소 금액을 계산하고 이를 입력으로 전달하십시오.

**Wormhole:**
커밋 af089d6에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
## 중급 위험 (Medium Risk)


### ERC20 `approve` 및 `transfer`의 `bool` 반환을 확인하면 성공해도 true를 반환하지 않는 메인넷 USDT 및 유사 토큰에 대해 프로토콜이 중단됨

**설명:** ERC20 `approve` 및 `transfer`의 `bool` 반환을 확인하면 호출이 성공했음에도 불구하고 [true를 반환하지 않는](https://etherscan.io/token/0xdac17f958d2ee523a2206206994597c13d831ec7#code) 메인넷 USDT 및 유사 토큰에 대해 프로토콜이 중단됩니다.

**영향:** 프로토콜이 메인넷 USDT 및 유사 토큰과 작동하지 않습니다.

**개념 증명(Proof of Concept):** Portico.sol L58, 61, 205, 320, 395, 399.

**권장 완화 조치:** [SafeERC20](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/utils/SafeERC20.sol) 또는 [SafeTransferLib](https://github.com/transmissions11/solmate/blob/main/src/utils/SafeTransferLib.sol)를 사용하십시오.

**Wormhole:**
커밋 3f08be9 및 55f93e2에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `relayerFeeAmount`를 뺄 때 정밀도 스케일링이나 최소 수령 금액 확인이 없으면 언더플로로 인해 revert 되거나 지정된 것보다 적은 토큰이 사용자에게 반환될 수 있음

**설명:** `PorticoFinish::payOut` L376은 최종 브리지 후 및 스왑 후 토큰 잔액에서 `relayerFeeAmount`를 빼려고 시도합니다:
```solidity
finalUserAmount = finalToken.balanceOf(address(this)) - relayerFeeAmount;
```

`PorticoFinish`의 토큰 계약 잔액과 `relayerFeeAmount`가 동일한 소수점 정밀도를 갖도록 보장하는 [정밀도 스케일링이 없습니다](https://dacian.me/precision-loss-errors#heading-no-precision-scaling). `relayerFeeAmount`가 18자리 소수점을 가지고 있지만 토큰이 6자리 소수점만 있는 USDC인 경우, 이는 언더플로로 인해 쉽게 revert 되어 브리지된 토큰이 갇힐 수 있습니다.

지나치게 높은 `relayerFeeAmount`는 또한 `relayerFeeAmount`를 차감한 후 사용자가 받을 최소 토큰 양에 대한 확인이 없기 때문에 브리지 후 및 스왑 후 받은 토큰 양을 크게 줄일 수 있습니다. 이 현재 구성은 ["최종 금액이 아닌 중간 금액에 대한 MinTokensOut"](https://dacian.me/defi-slippage-attacks#heading-mintokensout-for-intermediate-not-final-amount) 취약점 클래스의 예입니다. 최소 수령 토큰 확인이 `relayerFeeAmount` 차감 전에 있으므로 `relayerFeeAmount > 0`인 경우 사용자는 항상 지정된 최소값보다 적은 토큰을 받게 됩니다.

**영향:** 브리지된 토큰이 갇히거나 사용자가 지정된 최소값보다 적은 토큰을 받습니다.

**권장 완화 조치:** 결합하기 전에 토큰 잔액과 `relayerFeeAmount`가 동일한 소수점 정밀도를 갖도록 하십시오. 또는 언더플로를 확인하고 이 경우 수수료를 부과하지 마십시오. `relayerFeeAmount`를 차감할 때 사용자 지정 최소 출력 토큰 확인을 다시 시행하고, 이것이 실패하면 사용자가 적어도 최소 지정 토큰 금액을 받도록 `relayerFeeAmount`를 줄이는 것을 고려하십시오.

또 다른 옵션은 언더플로가 발생하지 않더라도 `relayerFeeAmount`를 뺀 후 남은 금액이 브리지된 금액의 높은 비율인지 확인하는 것입니다. 이는 `relayerFeeAmount`가 브리지된 금액의 큰 부분을 차지하는 시나리오를 방지하여 `relayerFeeAmount`를 브리지 후 및 스왑 후 자금의 아주 작은 %로 효과적으로 제한합니다. 그러나 이 시나리오에서도 사용자가 지정된 최소값보다 적은 토큰을 받을 수 있습니다.

스마트 계약의 관점에서 볼 때, 토큰 금액과 `relayerFeeAmount`의 소수점이 다르거나 `relayerFeeAmount`가 너무 높을 가능성으로부터 스스로를 보호해야 합니다. 예를 들어 `payOut` 내부의 L376이 브리지 보고 금액을 신뢰하지 않고 실제 토큰 잔액을 확인하는 것과 유사합니다.

**Wormhole:**
커밋 05ba84d에서 언더플로 확인을 추가하여 수정되었습니다. 모든 오동작은 잘못된 사용자 입력으로 인한 것이며 오프체인에서 수정되어야 합니다. 사용자만이 입력 매개변수에서 릴레이어 수수료를 설정할 수 있습니다.

**Cyfrin:** 릴레이어 수수료와 토큰 금액 간의 정밀도 불일치로 인한 잠재적 언더플로가 이제 처리되는지 확인했습니다. 구현은 이제 릴레이어를 선호하지만, 사용자만이 릴레이어 수수료를 설정할 수 있다는 사실로 균형이 맞춰지므로 공격 표면은 자해로 제한됩니다. 향후 릴레이어와 같은 다른 엔터티가 릴레이어 수수료를 설정할 수 있다면 브리지된 토큰을 고갈시키는 데 사용될 수 있지만, 현재 구현에서는 사용자가 잘못된 큰 릴레이어 수수료를 설정하지 않는 한 불가능합니다.

\clearpage
## 저위험 (Low Risk)


### 반환 데이터가 필요하지 않을 때 가스 그리핑(gas griefing) 공격을 방지하기 위해 저수준 `call()` 사용

**설명:** 반환 데이터가 필요하지 않을 때 `call()`을 사용하면 반환된 거대한 데이터 페이로드로 인한 가스 그리핑 공격에 불필요하게 노출됩니다. 예를 들어:
```solidity
(bool sentToUser, ) = recipient.call{ value: finalUserAmount }("");
require(sentToUser, "Failed to send Ether");
```

는 다음과 동일합니다:
```solidity
(bool sentToUser, bytes memory data) = recipient.call{ value: finalUserAmount }("");
require(sentToUser, "Failed to send Ether");
```

두 경우 모두 반환 데이터가 전혀 사용되지 않음에도 불구하고 반환 데이터가 메모리에 복사되어야 하므로 계약이 가스 그리핑 공격에 노출됩니다.

**영향:** 계약이 불필요하게 가스 그리핑 공격에 노출됩니다.

**권장 완화 조치:** 반환 데이터가 필요하지 않은 경우 저수준 호출을 사용하십시오. 예:
```solidity
bool sent;
assembly {
    sent := call(gas(), recipient, finalUserAmount, 0, 0, 0, 0)
}
if (!sent) revert Unauthorized();
```

[ExcessivelySafeCall](https://github.com/nomad-xyz/ExcessivelySafeCall) 사용을 고려하십시오.

**Wormhole:**
커밋 5f3926b에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
## 정보성 (Informational)


### `PorticoBase::unpadAddress`에서 주소 유효성에 대한 온전성 검사(sanity check) 누락

**설명:** `PorticoBase::unpadAddress`는 Wormhole Solidity SDK의 [`Utils::fromWormholeFormat`](https://github.com/wormhole-foundation/wormhole-solidity-sdk/blob/main/src/Utils.sol#L10-L15)을 재구현한 것이지만, SDK 구현에 있는 주소 유효성에 대한 온전성 검사가 누락되었습니다.

**권장 완화 조치:** `PorticoBase::unpadAddress`에 주소 유효성 온전성 검사를 추가하는 것을 고려하십시오.

**Wormhole:**
커밋 6208dd1에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `PorticoBase`의 지불 가능한 `receive()` 함수를 `PorticoFinish`로 이동

**설명:** `PorticoFinish`가 `WETH.withdraw()`를 호출할 때 ETH를 받아야 하는 유일한 계약이므로 `PorticoBase`의 지불 가능한 `receive()` 함수를 `PorticoFinish`로 이동하십시오.

`PorticoBase`를 상속하는 `PorticoStart`는 지불 가능한 `start` 함수 외에는 ETH를 받을 필요가 없으므로 지불 가능한 `receive()` 함수를 갖거나 상속할 필요가 없습니다.

**Wormhole:**
커밋 6208dd1에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 내부적으로 사용되지 않는 `Portico::start`를 external로 표시할 수 있음

**설명:** 내부적으로 사용되지 않는 `Portico::start`를 external로 표시할 수 있습니다.

**Wormhole:**
커밋 6208dd1에서 수정되었습니다.

**Cyfrin:** 확인됨.


### `TokenBridge::isDeployed`를 pure로 선언할 수 있음

**설명:** `TokenBridge::isDeployed`를 pure로 선언할 수 있습니다. 또한 이 계약의 요점이 무엇인지 확실하지 않습니다. 테스트에 사용되는 경우 `mocks` 디렉터리로 이동하십시오.

**Wormhole:**
이 계약을 제거했습니다.

**Cyfrin:** 확인됨.


### 사용되지 않는 코드 제거

**설명:**
```solidity
File: PorticoStructs.sol L67-79:
  //16 + 32 + 24 + 24 + 16 + 16 + 8 + 8 == 144
  struct packedData {
    uint16 recipientChain;
    uint32 bridgeNonce;
    uint24 startFee;
    uint24 endFee;
    int16 slipStart;
    int16 slipEnd;
    bool wrap;
    bool unwrap;
  }
```

**Wormhole:**
커밋 6208dd1에서 수정되었습니다.

**Cyfrin:** 확인됨.

\clearpage
## 가스 최적화 (Gas Optimization)


### `_completeTransfer`에서 `TOKENBRIDGE.parseTransferWithPayload`를 호출한 직후 잘못된 주소/체인 ID를 확인하여 빠른 실패(Fail fast) 유도

**설명:** [예제 코드](https://docs.wormhole.com/wormhole/quick-start/tutorials/hello-token#receiving-a-token)에 따라 `TOKENBRIDGE.parseTransferWithPayload`를 호출한 직후 잘못된 주소/체인 ID를 확인하여 `_completeTransfer`에서 빠르게 실패하도록 하십시오.

**영향:** 가스 최적화; 불필요한 작업을 수행한 후 나중에 실패하는 대신 빠르게 실패하기를 원합니다.

**개념 증명(Proof of Concept):** Portico.sol L278-300.

**권장 완화 조치:** L278 직후에 L300 확인을 수행하십시오.

**Wormhole:**
커밋 5f3926b에서 수정되었습니다.

**Cyfrin:** 확인됨.


### 변수를 기본값으로 초기화하지 말 것

**설명:** 변수를 기본값으로 초기화하지 마십시오. 예: `TickMath::getTickAtSqrtRatio()`:

```solidity
uint256 msb = 0;
```

**영향:** 가스 최적화.

**Wormhole:**
`TickMath`는 더 이상 온체인 슬리피지 계산이 수행되지 않으므로 사용되지 않습니다.


### revert 오류 문자열 대신 사용자 정의 오류 사용

**설명:** 배포 및 런타임 비용을 줄이기 위해 revert 오류 문자열 대신 사용자 정의 오류를 사용하십시오:

```solidity
File: Portico.sol

64:         require(token.approve(spender, 0), "approval reset failed");

67:       require(token.approve(spender, 2 ** 256 - 1), "infinite approval failed");

185:     require(poolExists, "Pool does not exist");

215:       require(value == params.amountSpecified + whMessageFee, "msg.value incorrect");

225:       require(value == whMessageFee, "msg.value incorrect");

232:       require(params.startTokenAddress.transferFrom(_msgSender(), address(this), params.amountSpecified), "transfer fail");

240:       require(amount >= params.amountSpecified, "transfer insufficient");

333:     require(unpadAddress(transfer.to) == address(this) && transfer.toChain == wormholeChainId, "Token was not sent to this address");

420:         require(sentToUser, "Failed to send Ether");

425:         require(sentToRelayer, "Failed to send Ether");

432:         require(finalToken.transfer(recipient, finalUserAmount), "STF");

436:         require(finalToken.transfer(feeRecipient, relayerFeeAmount), "STF");
```

**Wormhole:**
오류 문자열은 모두 길이가 32 미만으로 확인되었으며, 이는 이 계약의 목적에 충분합니다.

\clearpage
