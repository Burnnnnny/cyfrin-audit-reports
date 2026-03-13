**Lead Auditors**

[Hans](https://twitter.com/hansfriese)

**Assisting Auditors**



---

# Findings
## Medium Risk


### Allow message value to be more than the quote cost

**Description:** `SecuritizeBridge` 컨트랙트의 `bridgeDSTokens()` 함수는 사용자가 `quoteBridge()`에서 얻은 견적과 일치하는 정확한 값을 제공하도록 요구합니다. 이러한 엄격한 일치 요구 사항은 사용자가 견적을 확인하는 시점과 트랜잭션을 제출하는 시점 사이에 실제 비용이 변경될 수 있기 때문에 문제를 발생시킵니다.
```solidity
    function bridgeDSTokens(uint16 targetChain, uint256 value) public override payable whenNotPaused {
        uint256 cost = quoteBridge(targetChain);
        require(msg.value == cost, "Transaction value should be equal to quoteBridge response");
...
    }
```
비용 계산은 웜홀(Wormhole)의 `DeliveryProvider` 컨트랙트([여기](https://github.com/wormhole-foundation/wormhole/blob/abd0b330efa0a1bc86f0914396cbd570c99cdf1a/relayer/ethereum/contracts/relayer/deliveryProvider/DeliveryProvider.sol#L28))에서 보여지듯이 대상 체인의 가스 가격 및 자산 변환율을 포함한 여러 요인에 따라 달라집니다. 이러한 값들은 네트워크 상태에 따라 자주 변동될 수 있습니다.

```solidity
    function quoteEvmDeliveryPrice(
        uint16 targetChain,
        Gas gasLimit,
        TargetNative receiverValue
    )
        public
        view
        returns (LocalNative nativePriceQuote, GasPrice targetChainRefundPerUnitGasUnused)
    {
        // Calculates the amount to refund user on the target chain, for each unit of target chain gas unused
        // by multiplying the price of that amount of gas (in target chain currency)
        // by a target-chain-specific constant 'denominator'/('denominator' + 'buffer'), which will be close to 1

        (uint16 buffer, uint16 denominator) = assetConversionBuffer(targetChain);
        targetChainRefundPerUnitGasUnused = GasPrice.wrap(gasPrice(targetChain).unwrap() * (denominator) / (uint256(denominator) + buffer));

        // Calculates the cost of performing a delivery with 'gasLimit' units of gas and 'receiverValue' wei delivered to the target contract

        LocalNative gasLimitCostInSourceCurrency = quoteGasCost(targetChain, gasLimit);
        LocalNative receiverValueCostInSourceCurrency = quoteAssetCost(targetChain, receiverValue);
        nativePriceQuote = quoteDeliveryOverhead(targetChain) + gasLimitCostInSourceCurrency + receiverValueCostInSourceCurrency;

        // Checks that the amount of wei that needs to be sent into the target chain is <= the 'maximum budget' for the target chain

        TargetNative gasLimitCost = gasLimit.toWei(gasPrice(targetChain)).asTargetNative();
        if(receiverValue.asNative() + gasLimitCost.asNative() > maximumBudget(targetChain).asNative()) {
            revert ExceedsMaximumBudget(targetChain, receiverValue.unwrap() + gasLimitCost.unwrap(), maximumBudget(targetChain).unwrap());
        }
    }
```
견적 확인과 트랜잭션 제출 사이에 비용이 조금이라도 변경되면 트랜잭션이 실패합니다. 이는 사용자가 정확한 금액을 지불하려고 시도함에도 불구하고 트랜잭션이 빈번하게 복귀(revert)되어 열악한 사용자 경험을 초래합니다.

악의적인 행위자는 네트워크 상태를 조작하여 가격 변동을 유발함으로써 이러한 문제를 악화시킬 수 있으며, 사실상 다른 사용자들이 자산을 성공적으로 브리징하는 것을 방해할 수 있습니다.

**Impact:** 사용자는 자산 브리징을 시도할 때 트랜잭션 실패에 직면하여 좌절감을 느낍니다. 극단적인 경우 공격자는 조건을 조작하여 가격 변동을 유발함으로써 특정 사용자의 자산 브리징을 일시적으로 막을 수 있습니다.

**Recommended Mitigation:** 현재 견적을 초과하는 값을 허용하고 초과 금액을 자동으로 사용자에게 환불하도록 함수를 수정하십시오. 이 접근 방식은 사소한 가격 변동을 처리할 수 있는 유연성을 제공하면서도 사용자가 과도하게 지불하지 않도록 보장합니다.

**Securitize:** 커밋 [d3b97a](https://bitbucket.org/securitize_dev/bc-securitize-bridge-sc/commits/d3b97a76f93fd80ed6401372eadf206e1fb5d864) 및 [221759](https://bitbucket.org/securitize_dev/bc-securitize-bridge-sc/commits/2217591277f5a52913e0cd82136de13607608123)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.

\clearpage
## Low Risk


### Make the gas limit configurable

**Description:** `SecuritizeBridge` 컨트랙트는 현재 웜홀 프로토콜을 통한 모든 크로스체인 메시지 트랜잭션에 대해 2,500,000이라는 고정된(하드코딩된) 가스 한도를 사용합니다. 이 값은 대상 체인에서 트랜잭션을 실행하는 데 허용되는 최대 계산 단위(가스)를 나타냅니다.

이 값은 현재 구현에서는 작동하지만, 하드코딩된 상수로 두면 향후 업그레이드나 컨트랙트 기능 변경으로 인해 다른 가스 소비량이 필요할 경우 조정하기 어렵습니다. 예를 들어, 컨트랙트 로직이 업그레이드되어 더 많은 계산 단계가 필요한 경우, 현재 가스 한도가 불충분해질 수 있으며, 이 값을 조정하기 위해 전체 컨트랙트를 재배포해야 할 수도 있습니다.

**Recommended Mitigation:** 값을 업데이트할 수 있는 소유자 제어(owner-controlled) 함수를 추가하여 가스 한도를 구성 가능하게 만드십시오. 이를 통해 향후 컨트랙트 업그레이드로 인해 다른 가스 소비량이 필요한 경우, 전체 컨트랙트를 재배포할 필요 없이 프로토콜 관리자가 가스 한도를 조정할 수 있습니다.

Replace:
```solidity
uint256 public constant GAS_LIMIT = 2500_000;
```
with:
```solidity
uint256 public gasLimit;

function setGasLimit(uint256 _gasLimit) external onlyOwner {
    gasLimit = _gasLimit;
}
```

**Securitize:** 커밋 [525d86](https://bitbucket.org/securitize_dev/bc-securitize-bridge-sc/commits/525d8626ac53ab6ab38689e36d9d598c0626c90e)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.

\clearpage
## Informational


### Add a validation to check the message sender and the token value

**Description:** `SecuritizeBridge` 컨트랙트의 토큰 브리징 기능에 잠재적인 우려 사항이 있습니다.
이 컨트랙트는 규정 준수가 확인된 투자자와 함께 작동하도록 설계되었지만, 유효성 검사 과정에 공백이 있습니다:

현재 구현에서:
- 컨트랙트는 사용자가 브리징할 충분한 토큰을 보유하고 있는지 확인합니다 (`balanceOf` 체크).
- 토큰이 잠겨 있지 않은지 확인합니다 (`validateLockedTokens` 체크).
그러나 발신자가 유효한 투자자인지는 명시적으로 확인하지 않습니다.

결과적으로 사용자가 0개의 토큰을 브리징하려고 시도하면 두 유효성 검사가 모두 통과됩니다.
이는 검증되지 않은 투자자가 0개의 토큰으로 브리지 트랜잭션을 성공적으로 실행할 수 있음을 의미합니다.
토큰 전송은 발생하지 않지만, 불필요한 크로스체인 메시지를 생성하고 시스템 모니터링 및 이벤트 로그에 노이즈를 발생시킬 수 있습니다.

**Securitize:** 커밋 [6529fe](https://bitbucket.org/securitize_dev/bc-securitize-bridge-sc/commits/6529fe67789adab2266590f8581ef594e162aec5)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.


\clearpage
