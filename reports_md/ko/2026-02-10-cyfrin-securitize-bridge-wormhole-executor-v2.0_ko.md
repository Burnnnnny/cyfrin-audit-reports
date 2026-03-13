**Lead Auditors**

[Kage](https://x.com/0kage_eth)

[MrPotatoMagic](https://x.com/MrPotatoMagic)

**Assisting Auditors**



---

# 발견 사항 (Findings)
## 중간 위험 (Medium Risk)


### 설정되지 않은 CCTP 도메인 매핑이 0으로 기본 설정되어, 의도한 목적지 대신 Ethereum으로 USDC가 라우팅될 수 있음

**설명:** `USDCBridgeV2` 컨트랙트는 `chainIdToCCTPDomain`을 통해 Wormhole 체인 ID를 Circle CCTP 도메인 ID에 매핑합니다.

브리지 작업이 시작되면 컨트랙트는 `USDCBridgeV2::getCCTPDomain`으로 CCTP 도메인을 조회합니다.

```solidity
  function getCCTPDomain(uint16 _chain) internal view returns (uint32) {
      return chainIdToCCTPDomain[_chain];  // Returns 0 if not configured
  }
```

Solidity의 매핑은 초기화되지 않은 키에 대해 기본값 `0`을 반환합니다. 이 함수는 반환된 도메인이 명시적으로 설정된 값인지, 아니면 단순한 기본 0값인지 검증하지 않습니다. 참고로 CCTP는 [도메인 0을 Ethereum](https://developers.circle.com/cctp/references/contract-addresses#tokenmessengerv2)에 할당합니다.

다음 시나리오를 생각해 볼 수 있습니다.

```text
- 관리자가 USDCBridgeV2를 배포하고 일부 체인에 대한 CCTP 도메인을 설정하지만 특정 체인(예: Arbitrum)은 깜빡하고 설정하지 않음
- BRIDGE_CALLER가 _targetChain을 Arbitrum의 Wormhole ID로 설정해 sendUSDCCrossChainDeposit() 호출
- getCCTPDomain()이 0을 반환함 (기본 매핑 값)
- _transferUSDC()가 destinationDomain = 0으로 circleTokenMessenger.depositForBurn() 호출
- Circle의 CCTP에서 domain 0은 Ethereum mainnet에 해당함
- 소스 체인의 TokenMessenger가 Ethereum을 유효한 원격 체인으로 설정해 두었다면 트랜잭션은 성공함
- USDC는 소스 체인에서 소각되고 의도한 목적지가 아니라 Ethereum에서 민팅됨

```

**영향:** 체인 X의 수신자를 위해 보내진 USDC가 대신 Ethereum에서 민팅될 수 있습니다.

**권장 완화 조치:** 전송 전에 CCTP 도메인이 실제로 설정되어 있는지 명시적으로 검증하는 로직을 추가하는 것을 고려하십시오.

```diff
function getCCTPDomain(uint16 _chain) internal view returns (uint32) {
      uint32 domain = chainIdToCCTPDomain[_chain];

++      if (domain == 0 && _chain != 2) {
          revert CCTPDomainNotConfigured();
      }

      return domain;
  }
```


**Securitize:** [65369a1](https://github.com/securitize-io/bc-securitize-bridge-sc/commit/65369a1f98d6367090cfaa416ef318e98779fac6), [f92422b](https://github.com/securitize-io/bc-securitize-bridge-sc/commit/f92422b9e72fd8475c3f44eb9a46ede8beea7371)에서 수정함.

**Cyfrin:** 확인함.

\clearpage
## 낮은 위험 (Low Risk)


### `SecuritizeBridge`가 일시 중지되면 guardian set이 만료될 수 있음

**설명:** 임의의 호출자는 `executeVAAv1` 함수를 호출해 대기 중인 크로스체인 메시지를 실행할 수 있습니다. 하지만 `SecuritizeBridge`가 일시 중지되면 메시지를 실행할 수 없습니다.

```solidity
    function executeVAAv1(bytes calldata _encodedVM) external payable whenNotPaused {
        IWormhole _wormholeCore = wormholeCore;
        (IWormhole.VM memory vm, bool valid, ) = _wormholeCore.parseAndVerifyVM(_encodedVM);
        if (!valid) revert InvalidWormholeMessage();
```

이 일시 중지 상태에서는, [Wormhole Core](https://etherscan.io/address/0x3c3d457f1522d3540ab3325aa5f1864e34cba9d0#code) 컨트랙트에서 볼 수 있듯 guardian set에 만료 시간이 있기 때문에 전송 중인 `encodedVM`들이 만료될 수 있습니다.

```solidity
/// @dev Checks if VM guardian set index matches the current index (unless the current set is expired).
        if(vm.guardianSetIndex != getCurrentGuardianSetIndex() && guardianSet.expirationTime < block.timestamp){
            return (false, "guardian set has expired");
        }
```

**영향:** 이로 인해 투자자의 DS 토큰은 소스 체인에서 소각되었지만 목적지 체인에서는 한 번도 발행되지 않는 상황이 발생할 수 있습니다. DS 토큰을 수동 발행하는 것은 가능하겠지만, 전송 중인 메시지가 많다면 상당한 운영 부담이 생길 수 있습니다.

**권장 완화 조치:** 이를 수용 가능한 위험으로 본다면, 문제를 명시적으로 인지하고 만료된 guardian set에 속한 대기 메시지에 대해서는 DS 토큰을 수동으로 발행하는 절차를 마련하십시오.

또는 이런 상황을 피하려면 다음을 권장합니다.
 - 모든 소스 체인에서 브리지 주소를 제거합니다.
 - 대상 체인에서 전송 중인 메시지가 모두 이행/실행되도록 허용합니다.
 - 대상 체인을 일시 중지하여 남은 메시지가 만료되지 않도록 합니다.

**Securitize:** 인지함.

\clearpage
## 정보 (Informational)


### `USDCBridgeV2::quoteBridge`는 CCTP v2 흐름에서 실제로 지불되지 않는 wormhole core fee를 포함해 과대 비용 추정값을 반환함

**설명:** `USDCBridgeV2::quoteBridge`는 크로스체인 USDC 전송 비용을 추정합니다. 그러나 이 함수는 실제 브리지 작업 중에는 지불되지 않는 Wormhole Core 메시지 수수료(`coreFee`)까지 포함해 과대 추정된 값을 반환합니다.

```solidity
function quoteBridge(uint16 _targetChain) public override view returns (uint256 cost) {
      (uint256 coreFee, uint256 execFee) = _quoteBridge(_targetChain);
      cost = execFee + coreFee;  // @audit coreFee added but is not used in _quoteBridge
  }
```

`USDCBridgeV2::_quoteBridge`는 `coreFee`(Wormhole 메시지 수수료)와 `execFee`(Executor 수수료)를 모두 계산하지만, CCTP v2 흐름은 Wormhole VAA를 게시하지 않으므로 `coreFee`가 필요하지 않습니다.

```solidity
function _quoteBridge(uint16 _targetChain) private view returns (uint256 coreFee, uint256 execFee) {
      IWormhole _wormholeCore = wormholeCore;

      coreFee = _wormholeCore.messageFee();  // @audit calculated but never used in CCTP v2
      bytes memory request = ExecutorMessages.makeCCTPv2Request();
      bytes memory relayInstructions = RelayInstructions.encodeGas(gasLimit, 0);
      execFee = executorQuoterRouter.quoteExecution(
          _targetChain,
          bytes32(0),
          _msgSender(),
          quoterAddr,
          request,
          relayInstructions
      );
  }
```

실제로 `USDCBridgeV2::sendUSDCCrossChainDeposit`를 보면 `coreFee`가 계산되지만 사용되지 않습니다.

```solidity
function sendUSDCCrossChainDeposit(...) external ... {
      (, uint256 execFee) = _quoteBridge(_targetChain);  // coreFee discarded
      if (address(this).balance < execFee) revert InsufficientContractBalance();
      // ...
      executorQuoterRouter.requestExecution{value: execFee}(...);  // @audit only execFee paid
  }
```

CCTP v2 흐름은 Wormhole VAA가 아니라 Circle의 네이티브 메시징 인프라를 사용합니다. 따라서 이 경로에서는 `wormholeCore::publishMessage` 호출이 없고 Wormhole Core 메시지 수수료도 필요하지 않습니다.


**영향:** `USDCBridgeV2::quoteBridge`를 조회하는 외부 통합체(프런트엔드, 다른 컨트랙트, 오프체인 시스템)가 과대 추정된 비용 값을 받게 됩니다.

**권장 완화 조치:** `quoteBridge` 계산에서 `coreFee`를 제거하는 것을 고려하십시오.

**Securitize:** [73a0a3c](https://github.com/securitize-io/bc-securitize-bridge-sc/commit/73a0a3ccd5554c49a62f374c8e44c2696d309f85)에서 수정함.

**Cyfrin:** 확인함.


### 함수 `executeVAAv1`에 `nonReentrant` 수정자가 빠져 있음

**설명:** `SecuritizeBridge::executeVAAv1`에는 `nonReentrant` 수정자가 없습니다. 반면 `SecuritizeBridge::bridgeDSTokens`에는 이 수정자가 존재합니다. 현재로서는 직접적인 위험이 없지만, 일관성을 위해 도입하는 것이 좋습니다.

```solidity
function executeVAAv1(bytes calldata _encodedVM) external payable whenNotPaused {
```

**권장 완화 조치:** 함수 `executeVAAv1`에 `nonReentrant` 수정자를 추가하는 것을 고려하십시오.

**Securitize:** [a6b287c](https://github.com/securitize-io/bc-securitize-bridge-sc/commit/a6b287c23d901d6d0f6c02b6802158a529b1ec26)에서 수정함.

**Cyfrin:** 확인함.


### quote 요청에서 `_msgSender()` 대신 `address(this)`를 사용해야 함

**설명:** `USDCBridgeV2` 컨트랙트의 `_quoteBridge` 함수는 `quoteExecution` 호출에서 환불 주소로 `_msgSender()`를 사용합니다. 그러나 `sendUSDCCrossChainDeposit` 함수에서 외부 `requestExecution` 호출 시 사용되는 실제 환불 주소는 `address(this)`입니다.

`requestExecution` 함수:
```solidity
executorQuoterRouter.requestExecution{value: execFee}(
            _targetChain,
            bytes32(0),
            address(this), // << refund address
            quoterAddr,
            ExecutorMessages.makeCCTPv2Request(),
            RelayInstructions.encodeGas(gasLimit, 0)
        );
```

`_quoteBridge` 함수:

```solidity
function _quoteBridge(uint16 _targetChain) private view returns (uint256 coreFee, uint256 execFee) {
        IWormhole _wormholeCore = wormholeCore; // cache storage

        coreFee = _wormholeCore.messageFee();
        bytes memory request = ExecutorMessages.makeCCTPv2Request();
        bytes memory relayInstructions = RelayInstructions.encodeGas(gasLimit, 0);
        execFee = executorQuoterRouter.quoteExecution(
            _targetChain,
            bytes32(0),
            _msgSender(), // << refund address used as msg.sender
            quoterAddr,
            request,
            relayInstructions
        );
    }
```

현재 [ExecutorQuoterRouter](https://github.com/wormholelabs-xyz/example-messaging-executor/blob/14ecd59e2e9774a0e6a3b38f28896bc2d4369cd0/evm/src/ExecutorQuoterRouter.sol#L95C4-L107C31)의 quote 계산에는 영향을 주지 않지만, 일관성과 정확성을 위해 실제 환불 주소를 사용하는 것이 권장됩니다.

**권장 완화 조치:** 실제 환불 주소가 `USDCBridgeV2` 컨트랙트 자신이므로, `_quoteBridge` 함수에서 `_msgSender()` 대신 `address(this)`를 사용하십시오.

**Securitize:** [c618dee](https://github.com/securitize-io/bc-securitize-bridge-sc/commit/c618deec81d657ff63c8c975281b160f8f3f4c0c)에서 수정함.

**Cyfrin:** 확인함.


### 표준 `IERC20 approve` 대신 `SafeERC20` 승인을 사용해야 함

**설명:** `USDCBridgeV2._transferUSDC`에서 표준 `IERC20 approve` 대신 `SafeERC20::forceApprove` 함수를 사용하는 것이 좋습니다.

```solidity
IERC20(_USDC).approve(address(circleTokenMessenger), _amount);
```

**권장 완화 조치:** 위 권고를 따르는 것을 고려하십시오.

**Securitize:** [d138209](https://github.com/securitize-io/bc-securitize-bridge-sc/commit/d13820907c4bad24ab03273b03c7a55b11a69037)에서 수정함.

**Cyfrin:** 확인함.

\clearpage
## 가스 최적화 (Gas Optimization)


### `SecuritizeBridge::_quoteBridge`에서 불필요한 sequence 조회와 request 생성이 수행됨

**설명:** `SecuritizeBridge::_quoteBridge`는 이후 `ExecutorQuoter` 컨트랙트에서 무시되는 request 파라미터를 만들기 위해 불필요한 작업을 여러 번 수행합니다.

```solidity
 function _quoteBridge(uint16 _targetChain) private view returns (uint256 coreFee, uint256 execFee) {
        address targetAddress = bridgeAddresses[_targetChain];
        if (targetAddress == address(0)) revert BridgeAddressNotConfigured();

        IWormhole _wormholeCore = wormholeCore; // cache storage
        uint64 sequence = _wormholeCore.nextSequence(address(this)); //@audit sequence generated

        coreFee = _wormholeCore.messageFee();
        bytes memory request = ExecutorMessages.makeVAAv1Request(_wormholeCore.chainId(), _addressToBytes32(address(this)), sequence);
         //@audit request is computed but this is not used while
        bytes memory relayInstructions = RelayInstructions.encodeGas(gasLimit, 0);
        execFee = executorQuoterRouter.quoteExecution(
            _targetChain,
            _addressToBytes32(targetAddress),
            _msgSender(),
            quoterAddr,
            request,
            relayInstructions
        );
    }
```

[`ExecuteQuoteRouter::quoteExecution`](https://github.com/wormholelabs-xyz/example-messaging-executor/blob/14ecd59e2e9774a0e6a3b38f28896bc2d4369cd0/evm/src/ExecutorQuoterRouter.sol#L95)는 [`ExecuteQuote::requestQuote`](https://github.com/wormholelabs-xyz/example-messaging-executor/blob/14ecd59e2e9774a0e6a3b38f28896bc2d4369cd0/evm/src/ExecutorQuoter.sol#L191)를 호출하는데, 이 함수는 `request` 파라미터를 사용하지 않습니다.

```solidity
   //ExecuteQuoteRouter.sol
    function quoteExecution(
        uint16 dstChain,
        bytes32 dstAddr,
        address refundAddr,
        address quoterAddr,
        bytes calldata requestBytes,
        bytes calldata relayInstructions
    ) external view returns (uint256 requiredPayment) {
        requiredPayment =
            quoterContract[quoterAddr].requestQuote(dstChain, dstAddr, refundAddr, requestBytes, relayInstructions);
    }
```

```solidity
    //ExecutorQuoter.sol
    function requestQuote(
        uint16 dstChain,
        bytes32, //dstAddr,
        address, //refundAddr,
        bytes calldata, //requestBytes, //@audit request bytes are unused
        bytes calldata relayInstructions
    ) external view returns (uint256 requiredPayment) {
        ChainInfo storage dstChainInfo = chainInfos[dstChain];
        if (!dstChainInfo.enabled) {
            revert ChainDisabled(dstChain);
        }
        (uint256 gasLimit, uint256 msgValue) = totalGasLimitAndMsgValue(relayInstructions);
        // NOTE: this does not include any maxGasLimit or maxMsgValue checks
        requiredPayment = estimateQuote(quoteByDstChain[dstChain], dstChainInfo, gasLimit, msgValue);

        return requiredPayment;
    }
```

**권장 완화 조치:** 불필요한 sequence 조회와 request 생성을 제거하고, request 대신 빈 bytes를 사용하는 것을 고려하십시오.

**Securitize:** 인지함.


### 추가 함수와 제거 함수를 하나로 합칠 수 있음

**설명:** `USDCBridgeV2`의 `addBridgeCaller`와 `removeBridgeCaller` 함수는 하나의 함수로 합쳐 배포 가스를 절약할 수 있습니다.

```solidity
function addBridgeCaller(address _account) external override addressNotZero(_account) onlyRole(DEFAULT_ADMIN_ROLE) {
        grantRole(BRIDGE_CALLER, _account);
        emit BridgeCallerAdded(_account);
    }

    function removeBridgeCaller(address _account) external override addressNotZero(_account) onlyRole(DEFAULT_ADMIN_ROLE) {
        revokeRole(BRIDGE_CALLER, _account);
        emit BridgeCallerRemoved(_account);
    }
```


마찬가지로 `SecuritizeBridge`의 `setBridgeAddress`와 `removeBridgeAddress` 함수도 하나로 합칠 수 있습니다.
```solidity
function setBridgeAddress(uint16 _chainId, address _bridgeAddress) external override onlyOwner addressNotZero(_bridgeAddress) {
        bridgeAddresses[_chainId] = _bridgeAddress;
        emit BridgeAddressAdded(_chainId, _bridgeAddress);
    }


    function removeBridgeAddress(uint16 _chainId) external override onlyOwner {
        delete bridgeAddresses[_chainId];
        emit BridgeAddressRemoved(_chainId);
    }
```

**권장 완화 조치:** 다음과 같이 함수들을 통합하고 이벤트 이름도 갱신하는 방안을 고려하십시오.

```solidity
function addBridgeCaller(address _account, bool _status) external override addressNotZero(_account) onlyRole(DEFAULT_ADMIN_ROLE) {
        if (_status) grantRole(BRIDGE_CALLER, _account);
        else revokeRole(BRIDGE_CALLER, _account);
        emit BridgeCallerUpdated(_account);
    }
```

```solidity
function setBridgeAddress(uint16 _chainId, address _bridgeAddress) external override onlyOwner {
        bridgeAddresses[_chainId] = _bridgeAddress;
        emit BridgeAddressUpdated(_chainId, _bridgeAddress);
    }
```

**Securitize:** 인지함.

\clearpage
