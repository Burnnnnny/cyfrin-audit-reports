**Lead Auditors**

[Dacian](https://x.com/DevDacian)

[MrPotatoMagic](https://x.com/MrPotatoMagic)

**Assisting Auditors**


---

# 발견 사항 (Findings)
## 중간 위험 (Medium Risk)


### 초기화되지 않은 CCTP 도메인 매핑이 잘못된 블록체인으로 USDC를 전송할 수 있음

**설명:** `USDCBridgeV2::chainIdToCCTPDomain`은 Wormhole 체인 ID를 Circle의 CCTP 도메인 ID에 매핑합니다.
```solidity
mapping(uint16 => uint32) public chainIdToCCTPDomain;

function getCCTPDomain(uint16 _chain) internal view returns (uint32) {
    return chainIdToCCTPDomain[_chain];  // @audit returns 0 if not set!
}
```

**영향:** Wormhole 체인 ID에 대해 이 매핑이 초기화되지 않은 경우 기본적으로 0을 반환합니다(Solidity의 `uint32` 기본값). 그러나 [Circle의 CCTP 도메인 0은 이더리움 메인넷](https://developers.circle.com/cctp/cctp-supported-blockchains#cctp-v2-supported-domains)입니다. 따라서 주어진 [Wormhole 체인 ID](https://wormhole.com/docs/products/reference/chain-ids/)(예: Avalanche의 경우 6)에 대해 매핑이 구성되지 않은 경우, `USDCBridgeV2::_transferUSDC`는 Avalanche 대신 이더리움으로 USDC를 기꺼이 전송할 것입니다.
```solidity
        circleTokenMessenger.depositForBurn(
            _amount,
            getCCTPDomain(_targetChain), // @audit 0 by default = Ethereum mainnet
            targetAddressBytes32,        // mintRecipient on destination
            USDC,          // burnToken
            destinationCallerBytes32,        // destinationCaller (restrict who can mint)
            0,
            1000
        );
```

**권장 완화 조치:** 가장 간단한 옵션은 Wormhole의 이더리움 체인 ID에 대해서만 도메인 0을 허용하도록 `USDCBridgeV2::getCCTPDomain`을 변경하는 것입니다.
```solidity
function getCCTPDomain(uint16 _chain) internal view returns (uint32 domain) {
    domain = chainIdToCCTPDomain[_chain];
    // Wormhole ChainID 2 = Ethereum https://wormhole.com/docs/products/reference/chain-ids/
    // Only allow CCTP Domain 0 for Ethereum https://developers.circle.com/cctp/cctp-supported-blockchains#cctp-v2-supported-domains
    require(domain != 0 || _chain == 2, "CCTP domain not configured");
}
```

또 다른 잠재적인 해결책은 `setBridgeAddress`를 변경하여 CCTP 도메인도 항상 설정하도록 하는 것입니다. 예:
```solidity
    function setBridgeAddress(uint16 _chainId, address _bridgeAddress, uint32 _cctpDomain) external override onlyRole(DEFAULT_ADMIN_ROLE) {
        bridgeAddresses[_chainId] = _bridgeAddress;
        chainIdToCCTPDomain[_chain] = _cctpDomain;
        emit BridgeAddressAdd(_chainId, _bridgeAddress, _cctpDomain);
    }
```

또한 `chainIdToCCTPDomain`에서 삭제하도록 `removeBridgeAddress`를 변경하는 것을 고려하십시오. 예:
```solidity
    function removeBridgeAddress(uint16 _chainId) external override onlyRole(DEFAULT_ADMIN_ROLE) {
        delete bridgeAddresses[_chainId];
        delete chainIdToCCTPDomain[_chainId];
        emit BridgeAddressRemove(_chainId);
    }
```

**Securitize:** `setBridgeAddress`를 제거하고 동일한 Wormhole 체인 ID에 대해 대상 브리지 주소와 동시에 CCTP 도메인이 구성되도록 강제하는 새로운 함수 `setCCTPBridgeAddress`를 추가하여 커밋 [d750854](https://github.com/securitize-io/bc-securitize-bridge-sc/commit/d750854aadd6873ad2be3aa95fd5abe80fa01bd3)에서 수정됨. 또한 두 매핑을 함께 지우도록 `removeBridgeAddress`도 변경되었습니다.

**Cyfrin:** 확인함.


### 빠른 완결성(Fast Finality)에 0 최대 수수료를 하드코딩하는 것은 일반적으로 최소 수수료가 1이기 때문에 호환되지 않음

**설명:** Circle의 [CCTPv2 기술 가이드](https://developers.circle.com/cctp/technical-guide)는 다음과 같은 관련 정보를 제공합니다.
* `minFinalityThreshold`가 1000 이하인 메시지는 Fast 메시지로 간주됩니다.
* `minFinalityThreshold`가 2000인 메시지는 Standard 메시지로 간주됩니다(실제로는 1000 초과인 모든 것이 Standard로 간주됨).
* 적용 가능한 수수료는 매번 트랜잭션을 실행하기 전에 이 [API](https://developers.circle.com/api-reference/cctp/all/get-burn-usdc-fees)를 사용하여 검색해야 합니다.

제공된 API는 CCTP 입력 및 출력 [도메인](https://developers.circle.com/cctp/cctp-supported-blockchains#cctp-v2-supported-domains)을 지정해야 합니다. API의 `wget` 형식을 사용하면 빠른 완결성(1000)에 대한 최소 수수료는 일반적으로 1입니다.

* 이더리움 -> 아발란체
```console
$ wget --quiet \
  --method GET \
  --header 'Content-Type: application/json' \
  --output-document \
  - https://iris-api-sandbox.circle.com/v2/burn/USDC/fees/0/1
[{"finalityThreshold":1000,"minimumFee":1},{"finalityThreshold":2000,"minimumFee":0}]%
```

* 이더리움 -> 솔라나
```console
$ wget --quiet \
  --method GET \
  --header 'Content-Type: application/json' \
  --output-document \
  - https://iris-api-sandbox.circle.com/v2/burn/USDC/fees/0/5
[{"finalityThreshold":1000,"minimumFee":1},{"finalityThreshold":2000,"minimumFee":0}]%
```

**영향:** 많은 CCTP 크로스 도메인 전송은 빠른 완결성에 대해 최소 수수료가 1이지만, `USDCBridgeV2::_transferUSDC`는 빠른 완결성 1000에 대해 최대 수수료 0을 하드코딩합니다.
```solidity
circleTokenMessenger.depositForBurn(
    _amount,
    getCCTPDomain(_targetChain),
    targetAddressBytes32,        // mintRecipient on destination
    USDC,          // burnToken
    destinationCallerBytes32,        // destinationCaller (restrict who can mint)
    0,   // @audit maximum fee
    1000 // @audit fast finality
);
```

이 조합은 호환되지 않으며 많은 크로스 도메인 전송이 빠른 완결성을 사용하지 못하고 표준 완결성으로 되돌아갑니다. 표준 완결성에 대한 최소 수수료가 0보다 커지면 표준 완결성으로의 자동 다운그레이드가 더 이상 불가능하므로 시도된 모든 크로스 도메인 전송이 되돌려질(revert) 것입니다.

**권장 완화 조치:** 이상적으로는 최대 수수료와 완결성을 입력으로 제공해야 합니다.
* 현재 수수료 bps는 원하는 도메인 조합에 대해 제공된 API를 사용하여 오프체인에서 검색해야 합니다.
* 전송할 금액에 수수료 bps를 곱하여 최대 수수료를 계산합니다.
* `circleTokenMessenger.depositForBurn`을 호출할 때 최대 수수료와 원하는 완결성을 입력으로 전달합니다.

적어도 최대 수수료를 변경할 수 있는 방법이 있어야 하며, 표준 완결성 수수료가 0이 아닌 경우 프로토콜을 사용할 수 없게 되므로 0으로 하드코딩해서는 안 됩니다.

**Securitize:** 커밋 [0d3e50d](https://github.com/securitize-io/bc-securitize-bridge-sc/commit/0d3e50daadb37b29266a86d76a9c060eeed5805d)에서 다음을 통해 수정됨:
* 항상 표준 완결성 사용
* 최대 수수료는 이제 변수이므로 Circle이 향후 표준 완결성 수수료를 인상하면 변경할 수 있습니다.

**Cyfrin:** 확인함.


### 스마트 컨트랙트 지갑을 사용하는 투자자는 자신이 제어하지 않는 주소로 대상 체인 토큰이 발행될 수 있음

**설명:** `SecuritizeBridge::bridgeDSTokens`는 페이로드 메시지에 `_msgSender()`를 인코딩하여 대상 체인에 전달된 브리지된 토큰의 대상 주소가 되도록 합니다. 이는 투자자의 지갑 주소를 나타낼 것으로 예상됩니다.
```solidity
 // Send Relayer message
        wormholeRelayer.sendPayloadToEvm{value: msg.value} (
            targetChain,
            targetAddress,
            abi.encode(
                investorDetail.investorId,
                value,
                _msgSender(), // @audit destination address of bridged tokens
                investorDetail.country,
                investorDetail.attributeValues,
                investorDetail.attributeExpirations
            ), // payload
            0, // no receiver value needed since we"re just passing a message
            gasLimit,
            whChainId,
            _msgSender()
        );
```

대상 체인에서 실행되는 `SecuritizeBridge::receiveWormholeMessages`는 해당 주소를 투자자의 계정에 추가하고 토큰을 발행합니다.
```solidity
        address[] memory investorWallets = new address[](1);
        investorWallets[0] = investorWallet;

        // @audit assumes investor controls same wallet address
        // on destination chain - not necessarily true for smart contract wallets
        registryService.updateInvestor(investorId, investorId, country, investorWallets, attributeIds, attributeValues, attributeExpirations);
        dsToken.issueTokens(investorWallet, value);
```

**영향:** 투자자의 지갑이 스마트 컨트랙트 지갑(다중 서명/AA 지갑 - 체인마다 다른 주소를 가질 수 있음)인 경우 투자자가 대상 체인의 동일한 주소에 스마트 지갑 컨트랙트를 배포한다는 보장이 없습니다.

가능성은 낮지만 최악의 시나리오에서는 대상 체인의 주소가 투자자가 아닌 다른 EOA 사용자가 소유합니다. 이로 인해 `SecuritizeBridge::receiveWormholeMessages`는 외부 사용자 주소를 대상 체인의 투자자 소유로 등록하기 때문에 외부 사용자에게 원래 투자자 계정에 대한 제어권을 부여합니다.
* Alice는 이더리움의 0xAAA...에서 Gnosis Safe를 사용합니다.
* Alice는 자신의 DSToken을 Arbitrum으로 브리지합니다.
* Bob(악의적)은 Arbitrum의 EOA 0xAAA...를 제어합니다(동일한 주소, 다른 컨트롤러).
* 브리지는 Arbitrum의 0xAAA...를 Alice의 지갑으로 등록합니다.
* Bob은 이제 Arbitrum에 있는 Alice의 모든 토큰을 제어합니다.

**권장 완화 조치:** 간단한 해결책은 투자자가 유효한 대상 체인 지갑을 제공하고 이를 페이로드에 인코딩하도록 허용하는 것이지만, 이는 투자자가 실제로 제어하지 않는 지갑을 주장하도록 허용하여 악용될 소지도 있습니다.

우리가 감사한 다른 크로스 체인 규제 TradFi 프로토콜에서 프로토콜은 투자자 주소와 자격 증명을 크로스 체인으로 브리지하는 기능을 가지고 있었으며, 이는 토큰 브리징과는 별도로 발생했습니다.

크로스 체인 토큰 브리징은 대상 주소와 관련 자격 증명이 이미 브리지된 경우에만 작동할 수 있습니다. 이를 통해 대상 주소가 항상 유효함을 보장했습니다.

**Securitize:** 인지함; 이전 감사에서 제기되었으므로 이 엣지 케이스를 알고 있습니다. 대상 주소가 EOA가 아니고 투자자가 대상 체인에서 토큰을 제어할 수 없는 경우 토큰은 락업 기간으로 인해 잠긴 상태로 유지되며 Securitize는 토큰을 압류/소각할 충분한 시간을 갖습니다.


### 체인 간에 `DSToken`을 앞뒤로 브리징하면 `totalIssuance` 상한선에 도달하여 추가 발행 및 크로스 체인 전송이 방지됨

**설명:** `StandardToken::totalIssuance`는 소각에 의해 감소되지 않지만 최대 상한선을 강제하는 데 사용됩니다. `totalIssuance`는 현재 "공급량"이 아니라 지금까지 발행된 총 토큰 수를 추적하기 위한 것이기 때문입니다.

그러나 `SecuritizeBridge`를 통한 크로스 체인 브리징을 고려할 때 흥미로운 결과가 있습니다. `receiveWormholeMessages`는 대상 체인에서 `DSToken::issueTokens`를 호출하여 `StandardToken::totalIssuance`를 증가시킵니다.

**영향:** 다음 시나리오를 고려해 보십시오.
* Alice는 1000 `DSToken`으로 이더리움 -> 아르비트럼으로 브리지합니다.
* Alice는 "동일한" 1000 `DSToken`으로 아르비트럼 -> 이더리움으로 다시 브리지합니다.
* Alice는 이 작업을 계속 반복합니다.

이 과정은 앞뒤로 이동하는 동일한 토큰일지라도 양쪽 체인에서 `totalIssuance`를 지속적으로 증가시킵니다. 어느 시점에서는 체인 중 하나에서 상한선에 도달하게 됩니다. 악의적인 투자자가 없어도 앞뒤로 자주 브리지하는 투자자만 있어도 발생합니다.

상한선에 도달하면 해당 체인에서 추가 발행 및 크로스 체인 전송이 되돌려집니다.

**권장 완화 조치:** 잠재적인 완화 조치는 다음과 같습니다.
* 브리징이 실제로 소스 체인에서 `totalIssuance`를 감소시키도록 함
* `SecuritizeBridge::receiveWormholeMessages`가 `DSToken::issueTokensCustom`을 호출하여 `reason == "BRIDGING"`을 전달하고, `TokenLibrary::issueTokensCustom`에서 `"BRIDGING"` 이유에 대해 `totalIssuance`를 증가시키지 않음
* 브리지된 토큰 수를 별도로 추적하고 이를 설명하도록 상한선 확인을 수정

**Securitize:** 커밋 [c2e62c9](https://github.com/securitize-io/dstoken/commit/c2e62c9c1137bb7c6f548b72f960d864c42445fc)에서 수정됨; 상한선은 더 이상 사용되지 않으며 관련 확인이 제거되었습니다. `totalSupply`를 사용하여 소각을 올바르게 설명하는 유사한 규정 준수 관련 확인이 있습니다.

**Cyfrin:** 확인함.

\clearpage
## 낮은 위험 (Low Risk)


### 상속되는 업그레이드 가능한 컨트랙트는 스토리지 충돌을 방지하기 위해 ERC7201 네임스페이스 스토리지 레이아웃 또는 스토리지 갭을 사용해야 함

**설명:** 프로토콜에는 다른 컨트랙트가 상속하는 업그레이드 가능한 컨트랙트가 있습니다. 이러한 컨트랙트는 다음 중 하나를 사용해야 합니다.
* [ERC7201](https://eips.ethereum.org/EIPS/eip-7201) 네임스페이스 스토리지 레이아웃 - [예시](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/master/contracts/access/AccessControlUpgradeable.sol#L60-L72)
* 스토리지 갭 (이것은 [오래되었으며 더 이상 선호되지 않는](https://blog.openzeppelin.com/introducing-openzeppelin-contracts-5.0#Namespaced) 방법이지만)

이상적인 완화책은 모든 업그레이드 가능한 컨트랙트가 ERC7201 네임스페이스 스토리지 레이아웃을 사용하는 것입니다. 위의 두 가지 기술 중 하나를 사용하지 않으면 업그레이드 중에 스토리지 충돌이 발생할 수 있습니다. 영향을 받는 컨트랙트는 다음과 같습니다.

* `CCTPBase`
* `CCTPBase`

**Securitize:** 인지함; `CCTPBase`와 `CCTPBase`는 더 이상 사용되지 않으므로 나중에 제거될 예정입니다.


### `SecuritizeBridge::receiveWormholeMessages` 호출과 함께 전송된 ETH를 회수할 방법이 없음

**설명:** `SecuritizeBridge::receiveWormholeMessages`는 `payable`로 표시되어 있지만:
* `msg.value`로 아무것도 하지 않음
* `SecuritizeBridge`에 ETH를 인출하는 함수가 없음

**영향:** `SecuritizeBridge::receiveWormholeMessages` 호출과 함께 ETH가 전송되어야 하는 경우, 컨트랙트에 갇혀 회수할 수 없게 됩니다.

**권장 완화 조치:** 컨트랙트 소유자가 컨트랙트의 ETH 잔액을 인출할 수 있는 `withdrawETH` 함수를 추가하십시오.

**Securitize:** 소유자가 호출할 수 있는 `withdrawETH` 함수를 추가하여 커밋 [923e50e](https://github.com/securitize-io/bc-securitize-bridge-sc/commit/923e50e41dc859fa9516dd370988d01d685759e6), [2b18646](https://github.com/securitize-io/bc-securitize-bridge-sc/commit/2b18646e6344fcebe4f32107cd56812877ddadea#diff-3f58493270011157ff7c863627332c733405a46f8b6524660d25b33ef16f9f74R171)에서 수정됨.

**Cyfrin:** 확인함.


### ETH를 보내기 위해 `transfer`를 사용하지 마십시오

**설명:** 일부 작업의 가스 비용을 증가시킨 2019년 12월 이스탄불 하드 포크 이후 ETH를 보내기 위해 `transfer`를 사용하는 것은 권장되지 않았습니다. `transfer`는 가스를 2300으로 하드코딩하여 수신 함수가 되돌려지게 할 수 있으므로 미래 지향적이지 않습니다.

[ETH를 보내는 권장 방법](https://www.securitize-io.io/glossary/sending-ether-transfer-send-call-solidity-code-example)은 `call`을 사용하는 것이며 Solady에는 [SafeTransferLib::safeTransferETH](https://github.com/Vectorized/solady/blob/main/src/utils/SafeTransferLib.sol#L95-L103)에 이를 수행하는 최적화된 방법이 있습니다.

`transfer`는 L2에서도 예상대로 작동하지 않을 수 있습니다. 예를 들어 스마트 컨트랙트가 eth를 보내기 위해 transfer를 사용하여 zksync Era에 921 ETH가 갇히는 [사건](https://thedefiant.io/news/defi/zksync-rescues-gemholic)이 있었지만, 결국 zksync는 갇힌 eth를 구출하기 위한 [솔루션](https://www.theblock.co/post/225364/zksync-unfreeze-millions-stuck)을 개발했습니다.

`USDCBridgeV2::withdrawETH`의 영향을 받는 코드:
```solidity
bridge/USDCBridgeV2.sol
202:        _to.transfer(amount);
```

**Securitize:** 커밋 [2b18646](https://github.com/securitize-io/bc-securitize-bridge-sc/commit/2b18646e6344fcebe4f32107cd56812877ddadea)에서 수정됨.

**Cyfrin:** 확인함.


### `SecuritizeBridge` 및 `USDCBridgeV2`에서 브리징 이행을 위한 `whenNotPaused` 수정자의 일관성 없는 사용

**설명:** `USDCBridgeV2`는 `BaseRBACContract`에서 일시 중지 기능을 상속합니다. 현재 `USDCBridgeV2::receiveWormholeMessages`는 `whenNotPaused` 수정자를 적용하지 않아 일시 중지된 대상 체인 컨트랙트에서 토큰 수령이 발생할 수 있습니다.

반대로 `SecuritizeBridge::receiveWormholeMessages`에는 `whenNotPaused`가 있어 일시 중지된 대상 체인 컨트랙트에서 토큰 수령을 방지합니다.

**권장 완화 조치:** 브리징 이행을 위한 `whenNotPaused` 수정자의 사용이 일관되지 않습니다. 이 차이에 대한 타당한 이유가 없다면 일관성을 유지해야 합니다. 다음 중 하나를 선택하십시오.
* 대상 컨트랙트가 일시 중지된 경우 브리징 이행을 허용하지 않음
* 대상 컨트랙트가 일시 중지된 경우 브리징 이행을 허용하지만 일시 중지된 경우 새 브리징 요청을 허용하지 않음

**Securitize:** 인지함; 당분간은 수정자를 변경하지 않고 유지하는 것을 선호합니다. dsToken의 경우 우리가 제어하고 발행, 소각 등을 할 수 있으므로 플라잉 브리지에서의 발행을 방지할 수 있습니다. usdc 서클 스테이블 코인에 대한 제어권이 없으므로 USDC 플라잉 브리지를 일시 중지하고 싶지 않습니다.

\clearpage
## 정보 (Informational)


### 명명된 가져오기(named imports) 사용

**설명:** 코드베이스 전체에서 명명된 가져오기를 일관되게 사용하십시오.

**Securitize:** 커밋 [85ca7bb](https://github.com/securitize-io/bc-securitize-bridge-sc/commit/85ca7bb455cd78921b4f9ad7c48b0cf9eb0470e4)에서 수정됨.

**Cyfrin:** 확인함.


### 키와 값의 목적을 명시하기 위해 명명된 매핑 매개변수 사용

**설명:** 키와 값의 목적을 명시하기 위해 명명된 매핑 매개변수를 사용하십시오.
```solidity
wormhole/WormholeCCTPUpgradeable.sol
79:    mapping(uint16 => uint32) public chainIdToCCTPDomain;

bridge/USDCBridgeV2.sol
63:    mapping(uint16 => address) public bridgeAddresses;
64:    mapping(uint16 => uint32) public chainIdToCCTPDomain;

bridge/SecuritizeBridge.sol
40:    mapping(uint16 => address) public bridgeAddresses;
```

**Securitize:** 커밋 [40f4db0](https://github.com/securitize-io/bc-securitize-bridge-sc/commit/40f4db07a0351b3aebb3547a49236a1ca54a99d3)에서 수정됨.

**Cyfrin:** 확인함.


### 중요한 매개변수 변경 시 누락된 이벤트 방출

**설명:** 중요한 매개변수 변경 시 누락된 이벤트를 방출하십시오.
* `CCTPSender::setCCTPDomain`
* `USDCBridgeV2::setCCTPDomain`

**Securitize:** `USDCBridgeV2`에 대해 커밋 [cd8c8ad](https://github.com/securitize-io/bc-securitize-bridge-sc/commit/cd8c8ada1240c862458138cc1db9372aaf970573)에서 수정되었으며, 다른 하나는 더 이상 사용되지 않으므로 그대로 둡니다.

**Cyfrin:** 확인함.


### 표준 IERC20 함수 대신 `SafeERC20` 승인 및 전송 함수 사용

**설명:** 표준 IERC20 함수 대신 [SafeERC20::forceApprove](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/utils/SafeERC20.sol#L105-L110) 및 `safeTransfer` 함수를 사용하십시오.
```solidity
wormhole/WormholeCCTPUpgradeable.sol
121:        IERC20(USDC).approve(address(circleTokenMessenger), amount);

bridge/USDCBridgeV2.sol
150:        IERC20(USDC).transferFrom(_msgSender(), address(this), _amount);
264:        IERC20(USDC).approve(address(circleTokenMessenger), _amount);
```

**Securitize:** 커밋 [d75ac6f](https://github.com/securitize-io/bc-securitize-bridge-sc/commit/d75ac6fb219f2f536172e4c7b146cece27ca175e)에서 수정됨.

**Cyfrin:** 확인함.


### `BaseContract`에서 `OwnableUpgradeable` 대신 `Ownable2StepUpgradeable` 선호

**설명:** `BaseContract`에서 `OwnableUpgradeable` 대신 [Ownable2StepUpgradeable](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/master/contracts/access/Ownable2StepUpgradeable.sol)을 선호하십시오.

**Securitize:** 인지함.


### `SecuritizeBridge::whChainId` 이름을 `whRefundChainId`로 변경

**설명:** `SecuritizeBridge::whChainId`의 유일한 목적은 Wormhole 환불 체인 ID가 되는 것입니다. 목적을 정확하게 설명하는 `whRefundChainId`와 같은 이름으로 변경하십시오.

또한 이 값을 변경하는 함수가 없습니다. 필요할 수 있는 경우 함수를 추가하는 것을 고려하십시오.

**Securitize:** 인지함.


### 업그레이드 가능한 컨트랙트는 생성자에서 `_disableInitializers`를 호출해야 함

**설명:** 업그레이드 가능한 컨트랙트는 생성자에서 `_disableInitializers`를 [호출](https://docs.openzeppelin.com/upgrades-plugins/writing-upgradeable#initializing_the_implementation_contract)해야 합니다.
```solidity
/// @custom:oz-upgrades-unsafe-allow constructor
constructor() {
    _disableInitializers();
}
```

영향을 받는 컨트랙트:
* `SecuritizeBridge`
* `USDCBridgeV2`

**Securitize:** 커밋 [4b7f654](https://github.com/securitize-io/bc-securitize-bridge-sc/commit/4b7f654e98b0a2b4d953101372b360c906459d9b)에서 수정됨.

**Cyfrin:** 확인함.


### `USDCBridgeV2::setBridgeAddress`에 `addressNotZero` 수정자 사용

**설명:** `USDCBridgeV2::setBridgeAddress`에 `addressNotZero` 수정자를 사용하십시오.
```diff
-   function setBridgeAddress(uint16 _chainId, address _bridgeAddress) external override onlyRole(DEFAULT_ADMIN_ROLE) {
+   function setBridgeAddress(uint16 _chainId, address _bridgeAddress) external override addressNotZero(_bridgeAddress) onlyRole(DEFAULT_ADMIN_ROLE) {
```

**Securitize:** 커밋 [f51d885](https://github.com/securitize-io/bc-securitize-bridge-sc/commit/f51d885e50218167a7fb16d0152337bf8e8445d6)에서 수정됨.

**Cyfrin:** 확인함.


### 역할이 부여되거나 취소되지 않았을 때 `USDCBridgeV2::addBridgeCaller, removeBridgeCaller`의 오해의 소지가 있는 이벤트 방출

**설명:** `AccessControlUpgradeable::_grantRole,_revokeRole`은 역할이 부여되었는지 또는 취소되었는지를 나타내는 `bool`을 [반환](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/master/contracts/access/AccessControlUpgradeable.sol#L204-L230)합니다.

그러나 `USDCBridgeV2::removeBridgeCaller, addBridgeCaller`는 `public` 함수를 호출하므로 반환된 `bool`을 무시하고 역할이 부여되거나 취소되지 않았더라도 항상 이벤트를 방출합니다.

**권장 완화 조치:**
```diff
    function addBridgeCaller(address _account) external override addressNotZero(_account) onlyRole(DEFAULT_ADMIN_ROLE) {
-       grantRole(BRIDGE_CALLER, _account);
-       emit BridgeCallerAdded(_account);
+       if(_grantRole(BRIDGE_CALLER, _account)) emit BridgeCallerAdded(_account);
    }

    function removeBridgeCaller(address _account) external override addressNotZero(_account) onlyRole(DEFAULT_ADMIN_ROLE) {
-       revokeRole(BRIDGE_CALLER, _account);
-       emit BridgeCallerRemoved(_account);
+       if(_revokeRole(BRIDGE_CALLER, _account)) emit BridgeCallerRemoved(_account);
    }
```

**Securitize:** 인지함.


### 사용되지 않는 함수 `USDCBridgeV2::_redeemUSDC` 제거

**설명:** `private` 함수 `USDCBridgeV2::_redeemUSDC`는 어디에서도 사용되지 않습니다. 제거하십시오.

**Securitize:** 커밋 [07a872e](https://github.com/securitize-io/bc-securitize-bridge-sc/commit/07a872e8266411a440b736e328a86528b75cbdb0)에서 수정됨.

**Cyfrin:** 확인함.


### 유효한 투자자 지갑에 대해 초기화되지 않은 국가는 미국 규정 준수 락업 기간을 우회할 수 있음

**설명:** 프로토콜에 지갑을 등록할 수 있는 한 가지 방법은 `RegistryService.sol::addWallet`을 통하는 것이며, 등록된 지갑에 대한 국가 필드는 별도의 함수 `setCountry`를 사용하여 별도로 설정할 수 있습니다.

따라서 이 두 트랜잭션 사이에 유효한 지갑에 대해 국가 필드가 빈 문자열(기본값)로 남는 짧은 기간이 있을 수 있습니다.

**영향:** DS 토큰을 브리징하는 동안 `country`는 빈 문자열이므로 `region` 메모리 변수도 기본값인 0이 됩니다. 이는 `lockPeriod`가 비미국 락 기간을 저장하고 사용함을 의미합니다.
```solidity
string memory country = registryService.getCountry(investorId);
uint256 region = complianceConfigurationService.getCountryCompliance(country);

uint256 lockPeriod = (region == US) ? complianceConfigurationService.getUSLockPeriod() : complianceConfigurationService.getNonUSLockPeriod();
        uint256 availableBalanceForTransfer = complianceService.getComplianceTransferableTokens(_msgSender(), block.timestamp, uint64(lockPeriod));
```

이것은 다음과 같은 이유로 문제가 됩니다.
1. 유효한 지갑에 대한 초기화되지 않은 국가 필드가 브리징을 수행하도록 허용됨
2. 의도한 국가 = 미국인 경우 락 기간이 대신 비미국 락 기간을 사용함

이 짧은 기간은 악의적인 지갑 소유자가 더 낮은 `lockPeriod`(비미국 락 시간이 미국 락 시간보다 작은 경우)를 사용하기 위해 사용할 수 있습니다.

**권장 완화 조치:** `validateLockedTokens`에서 국가가 빈 문자열인지 확인하고 참이면 되돌리십시오(revert).

**Securitize:** 인지함; 투자자가 토큰을 브리징하는 경우 이미 발행/지급된 것입니다. 따라서 토큰 발행/지급 시점에 국가 및 지역을 포함한 규정 준수 규칙이 검증되었습니다.


### Wormhole과 Circle CCTP가 이를 지원하더라도 `USDCBridgeV2`는 비 EVM 체인으로 브리지할 수 없음

**설명:** Wormhole과 Circle CCTP는 모두 EVM과 비 EVM 체인 간의 브리징을 지원하지만, `USCBridgeV2`는 다음과 같은 이유로 비 EVM 체인으로의 브리징을 방지합니다.

1) `_sendUSDCWithPayloadToEvm`은 항상 `wormholeRelayer.sendToEvm`을 호출합니다.
2) `bridgeAddresses[_targetChain]`은 `address`를 사용하여 대상 브리지를 저장하지만 이는 Solana와 같은 비 EVM 체인의 대상 브리지와 호환되지 않습니다.

**영향:** 비 EVM 체인으로의 브리징은 지원되지 않습니다.

**권장 완화 조치:** 비 EVM 체인으로의 브리징이 지원되어야 하는 경우:
* `sendToEvm` 대신 `wormholeRelayer.send`를 사용하십시오.
* `bridgeAddresses[_targetChain]`은 대상 브리지를 `bytes32`로 저장한 다음 EVM 체인으로 브리징할 때 `address`로 캐스트해야 합니다.
* 일반적으로 원격 체인의 주소는 `address`가 아닌 `bytes32`를 사용하여 입력으로 전달되고 저장 및 사용되어야 합니다.

**Securitize:** 인지함; 당분간 의도된 설계입니다.


### 사용되지 않는 가져오기 제거

**설명:** 사용되지 않는 가져오기를 제거하십시오.
* `USDCBridgeV2.sol`
```solidity
28:import {IBridge} from "./IBridge.sol";
```

**Securitize:** 커밋 [a45cb7e](https://github.com/securitize-io/bc-securitize-bridge-sc/commit/a45cb7edcc9ec79c5e1cc30420c826e0566af827)에서 수정됨.

**Cyfrin:** 확인함.


### `BaseContract`에서 함수 선언 솔리디티 스타일 가이드 따르기

**설명:** 함수 `pause` 및 `unpause`는 수정자 `onlyOwner` 뒤에 가시성 `public`을 정의합니다. [Solidity Style Guide](https://docs.soliditylang.org/en/latest/style-guide.html#function-declaration)에 따르면 수정자는 가시성 선언 뒤에 배치해야 합니다.
```solidity
function pause() onlyOwner external {
        _pause();
    }

    function unpause() onlyOwner external {
        _unpause();
    }
```

**권장 완화 조치:** 다음과 같이 코드를 업데이트하십시오.
```solidity
function pause() external onlyOwner {
        _pause();
    }

    function unpause() external onlyOwner {
        _unpause();
    }
```

**Securitize:** 커밋 [61fb6a6](https://github.com/securitize-io/bc-securitize-bridge-sc/commit/61fb6a6e4cb0830d152aa3a436a718fb0e0795ae)에서 수정됨.

**Cyfrin:** 확인함.


### 브리지 주소가 업데이트되면 이전 브리지 주소에서 소싱된 보류 중/재실행 가능 메시지를 실행할 수 없음

**설명:** SecuritizeBridge 및 USDCBridgeV2는 소유자에게 브리지 주소를 업데이트할 수 있는 기능을 제공합니다. 이는 새 브리지 주소가 사용될 것으로 예상되고 이전 주소가 더 이상 사용되지 않는 경우 수행될 수 있습니다.

```solidity
function setBridgeAddress(uint16 chainId, address bridgeAddress) external override onlyOwner {
        bridgeAddresses[chainId] = bridgeAddress;
        emit BridgeAddressAdd(chainId, bridgeAddress);
    }
```

**영향:** 여기서 알아야 할 중요한 동작 중 하나는 전달되기를 기다리는 보류 중이거나 실패한 대상 메시지가 있을 수 있다는 것입니다. 이것들이 실행되기 전에 브리지 주소가 업데이트되면 브리지 주소가 이전 주소로 업데이트되지 않는 한 다시는 실행할 수 없게 될 수 있습니다.

**권장 완화 조치:** 체인에 대한 브리지 주소를 업데이트하기 전에 보류 중인 메시지의 전달을 기다리고 실패한 대상 메시지를 실행하는 것을 고려하십시오.

**Securitize:** 인지함.

\clearpage
## 가스 최적화 (Gas Optimization)


### 동일한 스토리지 읽기를 방지하기 위해 스토리지 캐시

**설명:** 스토리지에서 읽는 것은 비용이 많이 듭니다. 동일한 스토리지 읽기를 방지하기 위해 스토리지를 캐시하십시오.
* `contracts/wormhole/WormholeCCTPUpgradeable.sol`
```solidity
// cache `USDC` in `redeemUSDC`
65:        uint256 beforeBalance = IERC20(USDC).balanceOf(address(this));
67:        return IERC20(USDC).balanceOf(address(this)) - beforeBalance;
```

* `contracts/bridge/SecuritizeBridge.sol`
```solidity
// cache `dsToken` in `bridgeDSTokens`
73:        require(dsToken.balanceOf(_msgSender()) >= value, "Not enough balance in source chain to bridge");
88:        dsToken.burn(_msgSender(), value, BRIDGE_REASON);
108:        emit DSTokenBridgeSend(targetChain, address(dsToken), _msgSender(), value);

// cache `dsToken` in `receiveWormholeMessages`
144:        dsToken.issueTokens(investorWallet, value);
146:        emit DSTokenBridgeReceive(sourceChain, address(dsToken), investorWallet, value);
```

* `contracts/bridge/USDCBridgeV2.sol`
```solidity
// cache `USDC` in `sendUSDCCrossChainDeposit`
143:        if (IERC20(USDC).balanceOf(_msgSender()) < _amount) {
150:        IERC20(USDC).transferFrom(_msgSender(), address(this), _amount);
// also change `_transferUSDC` to take cached `USDC` as parameter to save
// 2 storage reads inside `_transferUSDC`;
// cache `USDC, `circleTokenMessenger` in `_transferUSDC`
264:        IERC20(USDC).approve(address(circleTokenMessenger), _amount);
268:        circleTokenMessenger.depositForBurn(
272:            USDC,          // burnToken
```

**Securitize:** 인지함.


### 불필요한 작업 없이 빠르게 실패

**설명:** 트랜잭션이 되돌려질 예정이라면 불필요한 작업을 수행하지 말고 가능한 한 빨리 되돌리십시오. 이를 달성하기 위한 전략은 다음과 같습니다.
* 모든 입력 관련 유효성 검사를 먼저 수행
* 다음 유효성 검사 단계를 수행하기에 충분한 스토리지 만 읽거나 충분한 외부 호출 만 수행

예를 들어 `SecuritizeBridge::bridgeDSTokens`에서:
```solidity
    function bridgeDSTokens(uint16 targetChain, uint256 value) external override payable whenNotPaused {
        // @audit why do all this work...
        uint256 cost = quoteBridge(targetChain);
        require(msg.value >= cost, "Transaction value should be equal or greater than quoteBridge response");
        require(dsToken.balanceOf(_msgSender()) >= value, "Not enough balance in source chain to bridge");
        address targetAddress = bridgeAddresses[targetChain];
        require(bridgeAddresses[targetChain] != address(0), "No bridge address available");

        IDSRegistryService registryService = IDSRegistryService(dsServiceConsumer.getDSService(dsServiceConsumer.REGISTRY_SERVICE()));
        require(registryService.isWallet(_msgSender()), "Investor not registered");

        // @audit ...if txn will revert here due to invalid input?
        require(value > 0, "DSToken value must be greater than 0");
```

그리고 `USDCBridgeV2::sendUSDCCrossChainDeposit`에서
```solidity
    function sendUSDCCrossChainDeposit(
        uint16 _targetChain,
        address _recipient,
        uint256 _amount
    ) external override whenNotPaused nonReentrant onlyRole(BRIDGE_CALLER) {
        uint256 deliveryCost = quoteBridge(_targetChain);
        // @audit why perform this storage read...
        address targetBridge = bridgeAddresses[_targetChain];
        // @audit ...if this check will just revert? Perform this check immediately
        // after `uint256 deliveryCost = quoteBridge(_targetChain);`
        if (address(this).balance < deliveryCost) {
            revert InsufficientContractBalance();
        }
        // @audit why perform this check here?
        if (IERC20(USDC).balanceOf(_msgSender()) < _amount) {
            revert NotEnoughBalance();
        }
        // @audit if it is going to revert from this? Perform this check immediately
        // after ` address targetBridge = bridgeAddresses[_targetChain];`
        if (targetBridge == address(0)) {
            revert BridgeAddressUndefined();
        }
```

**Securitize:** 커밋 [2ad89cf](https://github.com/securitize-io/bc-securitize-bridge-sc/commit/2ad89cf65ea61c999f140fa6754fb1077a3674b1)에서 수정됨.

**Cyfrin:** 확인함.


### `SecuritizeBridge::bridgeDSTokens`의 확인에 `bridgeAddresses[targetChain]` 대신 `targetAddress` 사용

**설명:** `SecuritizeBridge::bridgeDSTokens`의 확인에 `bridgeAddresses[targetChain]` 대신 `targetAddress`를 사용하십시오.
```diff
        address targetAddress = bridgeAddresses[targetChain];
-       require(bridgeAddresses[targetChain] != address(0), "No bridge address available");
+       require(targetAddress != address(0), "No bridge address available");
```

**Securitize:** 커밋 [9081b85](https://github.com/securitize-io/bc-securitize-bridge-sc/commit/9081b858ffddcf1b9b6a3eafcbee3b6a2da192e8)에서 수정됨.

**Cyfrin:** 확인함.


### `SecuritizeBridge::bridgeDSTokens` 및 `quoteBridge`를 리팩터링하여 `internal` 함수를 사용하여 브리징 트랜잭션당 2개의 스토리지 읽기 절약

**설명:** `SecuritizeBridge::bridgeDSTokens`:
* L71은 `quoteBridge`를 호출하여 스토리지에서 `wormholeRelayer`와 `gasLimit`을 읽습니다.
* L91은 `wormholeRelayer.sendPayloadToEvm`을 호출하여 스토리지에서 `wormholeRelayer`를 다시 읽습니다.
* L108은 스토리지에서 `gasLimit`을 다시 읽습니다.

스토리지 읽기는 비용이 많이 듭니다. 여기서 동일한 스토리지 읽기를 방지하려면 다음과 같이 리팩터링하십시오.
```solidity
// new internal function
    function _quoteBridge(IWormholeRelayer relayer, uint256 _gasLimit, uint16 targetChain) internal view returns (uint256 cost) {
        (cost, ) = relayer.quoteEVMDeliveryPrice(targetChain, 0, _gasLimit);
    }

// modify `quoteBridge` to use new internal function
    function quoteBridge(uint16 targetChain) public override view returns (uint256 cost) {
        (cost, ) = _quoteBridge(wormholeRelayer, gasLimit, targetChain);
    }

// in `bridgeDSTokens` to cache `wormholeRelayer` and `gasLimit`
// then pass them to `_quoteBridge` and use them at L91 & L108
```

동일한 최적화가 `USDCBridgeV2::sendUSDCCrossChainDeposit`, `quoteBridge` 및 `_sendUSDCWithPayloadToEvm`에도 적용되어야 합니다.

**Securitize:** 커밋 [47c1ad0](https://github.com/securitize-io/bc-securitize-bridge-sc/commit/47c1ad0de51887344785cebb7f5668b769b9d092)에서 수정됨.

**Cyfrin:** 확인함.


### 지역 변수를 제거할 수 있는 경우 명명된 반환 값 사용

**설명:** 지역 변수를 제거할 수 있는 경우 명명된 반환 값을 사용하십시오.
* `SecuritizeBridge::getInvestorData`

**Securitize:** 인지함.


### `dsServiceConsumer`를 입력 매개변수로 받도록 `SecuritizeBridge::validateLockedTokens` 리팩터링

**설명:** `SecuritizeBridge::bridgeDSTokens`는 L77에서 스토리지의 `dsServiceConsumer`를 읽은 다음 L83에서 `validateLockedTokens`를 호출합니다.

내부 함수 `validateLockedTokens` 자체는 스토리지에서 `dsServiceConsumer`를 여러 번 다시 읽습니다.

스토리지에서 읽는 것은 비용이 많이 듭니다. 대신:
* `bridgeDSTokens`에서 `dsServiceConsumer`를 한 번 캐시합니다.
* `dsServiceConsumer`를 입력 매개변수로 받도록 `validateLockedTokens`를 리팩터링합니다.
* `bridgeDSTokens`에서 `validateLockedTokens`를 호출할 때 캐시된 `dsServiceConsumer`를 입력 매개변수로 전달합니다.

**Securitize:** 커밋 [bcc83e0](https://github.com/securitize-io/bc-securitize-bridge-sc/commit/bcc83e04b66320b07fcc621352de72e59985bf8a), [0cf5dd9](https://github.com/securitize-io/bc-securitize-bridge-sc/commit/0cf5dd91a4ff6ba1d2a1d5c9b55c395e415536e1)에서 수정됨.

**Cyfrin:** 확인함.


### 더 빠른 `nonReentrant` 수정자를 위해 `ReentrancyGuardTransientUpgradeable` 사용

**설명:** 더 빠른 `nonReentrant` 수정자를 위해 [ReentrancyGuardTransientUpgradeable](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/master/contracts/utils/ReentrancyGuardTransientUpgradeable.sol)을 사용하십시오.
```solidity
bridge/USDCBridgeV2.sol
27:import {ReentrancyGuardUpgradeable} from "@openzeppelin/contracts-upgradeable/utils/ReentrancyGuardUpgradeable.sol";
51:contract USDCBridgeV2 is IUSDCBridge, IWormholeReceiver, BaseRBACContract, ReentrancyGuardUpgradeable {
86:        __ReentrancyGuard_init();
```

**Securitize:** 인지함.


### `USDCBridgeV2:_sendUSDCWithPayloadToEvm`에서 사용되지 않는 반환 값 제거

**설명:** `USDCBridgeV2::_sendUSDCWithPayloadToEvm()` 함수는 `wormholeRelayer.sendToEvm` 외부 호출에서 시퀀스 번호를 반환합니다. 그러나 이 값은 이후 전혀 활용되지 않습니다.

**권장 완화 조치:** 이 변수를 제거하는 것을 고려하십시오.

**Securitize:** 인지함.


### `USDCBridgeV2::_buildCCTPKey`에 입력으로 `_refundChain`을 전달하면 1개의 스토리지 읽기 및 외부 호출 절약

**설명:** `USDCBridgeV2::sendUSDCCrossChainDeposit`은 끝에서 세 번째 매개변수로 `wormhole.chainId()`를 전달합니다.
```solidity
        _sendUSDCWithPayloadToEvm(
            _targetChain,
            targetBridge, // address (on targetChain) to send token and payload to
            payload,
            0, // receiver value
            gasLimit,
            _amount,
            wormhole.chainId(), // @audit `_refundChain`
            address(this),
            deliveryCost
        );
```

그러나 `_sendUSDCWithPayloadToEvm`은 동일한 작업을 다시 수행하는 `_buildCCTPKey`를 호출합니다.
```solidity
    function _buildCCTPKey() private view returns (MessageKey memory) {
        return MessageKey(CCTP_KEY_TYPE, abi.encodePacked(getCCTPDomain(wormhole.chainId()), uint64(0)));
    }
```

`uint16 _whSourceChain`을 입력 매개변수로 받도록 `_buildCCTPKey`를 리팩터링하고 다음과 같이 사용하십시오.
```solidity
    function _buildCCTPKey(uint16 _whSourceChain) private view returns (MessageKey memory) {
        return MessageKey(CCTP_KEY_TYPE, abi.encodePacked(getCCTPDomain(_whSourceChain), uint64(0)));
    }
```

**Securitize:** 인지함.

\clearpage

