**Lead Auditors**

[Dacian](https://x.com/DevDacian)

[Hans](https://x.com/hansfriese)

**Assisting Auditors**



---

# Findings
## Low Risk


### CCIP 네이티브 수수료 지불을 강제하면 `LINK` 보유자에게 10% 더 높은 비용이 발생함 (Forcing CCIP native fee payment results in 10 percent higher costs for `LINK` holders)

**설명:** CCIP는 사용자가 `LINK` 또는 네이티브 가스 토큰을 사용하여 지불할 수 있도록 허용합니다. `EVM2AnyMessage::feeToken = address(0)`를 하드코딩함으로써 프로토콜은 모든 사용자가 네이티브 가스 토큰을 사용하여 지불하도록 강제합니다.

CCIP는 `LINK`를 사용하여 지불할 경우 10% 할인을 제공하므로, 이는 프로토콜 구현을 단순화하지만 `LINK` 보유자에게 [더 높은 비용](https://docs.chain.link/ccip/billing#network-fee-table)을 초래합니다.

**Matrixdock:** 인지함.


### 사용자가 전송 및 브리징을 사용하여 차단 목록을 통한 토큰 동결을 회피할 수 있음 (Users can use transfer and bridging to evade having their tokens frozen via the blocklist)

**설명:** 일반적인 전송이나 CCIP / LayerZero 브리징을 통한 크로스 체인 전송의 한 가지 비전형적인 응용은 차단 목록을 회피하는 것입니다:
* 사용자가 멤풀에서 자신의 주소를 차단하는 `MToken::addToBlockedList`에 대한 운영자(operator) 호출을 봅니다.
* 사용자는 일반 전송 또는 CCIP / LayerZero 크로스 체인 전송을 통해 토큰을 다른 체인의 새로운 `receiver` 주소로 브리징하여 이 트랜잭션을 프론트런(front-run)합니다.
* 운영자가 새로운 `receiver` 주소에 대해 다른 체인에서 `MToken::addToBlockedList`를 호출하려고 시도하면, 사용자는 다시 또 다른 새로운 주소로 다시 브리징할 수 있습니다.

이를 방지하기 위해 운영자는 다음을 수행할 수 있습니다:
* `MToken::addToBlockedList`를 호출하기 전에 브리징을 일시 중지합니다(LayerZero에 대해서는 일시 중지가 구현되었지만 CCIP에 대해서는 구현되지 않음).
* `MToken::addToBlockedList`를 호출할 때 [flashbots](https://www.flashbots.net/)와 같은 서비스를 사용하여 트랜잭션이 공개 멤풀에 노출되지 않도록 합니다.

**Matrixdock:** 인지함.


### 메신저 계약에서 직접적인 ETH 전송을 거부하는 `receive` 함수 누락 (Missing `receive` function to reject direct ETH transfers in messager contracts)

**설명:** 메신저 계약들(`MTokenMessager`, `MTokenMessagerLZ`, `MTokenMessagerV2`)은 네이티브 토큰으로 브리징 수수료를 받도록 설계되었지만, 직접적인 ETH 전송을 처리하기 위한 `receive()` 함수를 구현하지 않았습니다. 이 함수가 없으면 사용자가 실수로 계약 주소로 ETH를 보낼 수 있으며, 이를 인출할 메커니즘이 없으므로 영구적으로 잠기게 됩니다.

**권장 완화 방법:** 계약으로의 직접적인 ETH 전송을 명시적으로 거부하도록 revert하는 `receive()` 함수를 추가하세요:

```diff
3 contract MTokenMessagerBase {
4
5     address public ccipClient;//@audit-info MToken
6
7     constructor(address _ccipClient){
8         ccipClient = _ccipClient;
9     }
+
+     receive() external payable {
+         revert("ETH transfers not accepted");
+     }
10 }
```

**Matrixdock:** 인지함.



### 크로스 체인 차단된 수신자가 제대로 처리되지 않음 (Cross-chain blocked recipients aren't properly handled)

**설명:** `MToken` 계약은 특정 주소가 토큰과 상호 작용하는 것을 방지하기 위해 차단 메커니즘을 구현합니다. 그러나 크로스 체인 기능은 차단된 주소를 제대로 처리하지 않습니다.

두 가지 주요 문제가 있습니다:

1. `MToken::msgOfCcSendToken`에서 계약은 `receiver`가 소스 체인에서 차단되었는지 확인하지만, 수신자가 대상 체인에 존재하므로 이 확인은 유효하지 않습니다.
```solidity
369:    function msgOfCcSendToken(
370:        address sender,
371:        address receiver,
372:        uint256 value
373:    ) public view returns (bytes memory message) {
374:        _checkBlocked(sender);
375:        _checkBlocked(receiver);//@audit-issue receiver가 동일한 체인에 있지 않으므로 이 확인은 의미가 없음
376:        return abi.encode(TagSendToken, abi.encode(sender, receiver, value));
377:    }
```

2. `MToken::ccReceiveToken`에서 토큰을 발행(mint)하기 전에 `receiver`가 현재(대상) 체인에서 차단되었는지 확인하는 절차가 없습니다.
```solidity
415:    function ccReceiveToken(bytes memory message) internal {
416:        (address sender, address receiver, uint value) = abi.decode(
417:            message,
418:            (address, address, uint)
419:        );
420:        _mint(receiver, value);//@audit-issue receiver가 차단되었는지 확인해야 하며, 차단된 주소로 전송된 자금을 관리해야 할 수도 있음
421:        emit CCReceiveToken(sender, receiver, value);
422:    }
```
이러한 문제로 인해 차단된 주소가 크로스 체인 전송을 통해 토큰을 수신하여 프로토콜이 의도한 보안 제어를 우회할 수 있습니다.

**영향:** 크로스 체인 전송을 사용하여 차단 메커니즘을 우회할 수 있습니다. 한 체인에서 차단된 악의적이거나 제재된 주소가 여전히 크로스 체인 전송을 통해 토큰을 수신할 수 있어 프로토콜의 보안 기능을 훼손합니다.

**개념 증명 (Proof Of Concept):**
```solidity
    // 차단된 주소로의 크로스 체인 전송 테스트
    function testCrossChainSendToken_ToBlockedAddress() public {
        // user1에게 일부 토큰 발행
        uint256 amount = 100 * 10**18;
        mintTokens(user1, amount);

        // 대상 체인에서 user2 차단
        vm.prank(operator);
        remoteChainMToken.addToBlockedList(user2);

        // user1이 차단된 user2에게 크로스 체인으로 토큰 전송 시도
        vm.startPrank(user1);
        mtoken.approve(address(mockMessager), amount);

        // 차단된 주소로 보낼 때 전송은 성공할 수 있지만 토큰은 대상에 도달하지 않아야 함
        mockMessager.sendTokenToChain{value: 0.01 ether}(
            CHAIN_SELECTOR_2,
            address(remoteChainMToken),
            user2,
            amount,
            ""
        );
        vm.stopPrank();

        // user1의 토큰이 사라졌는지 확인 (전송 과정에서 소각됨)
        assertEq(mtoken.balanceOf(user1), 0, "Tokens should be burned on source chain");

        // 차단된 사용자는 토큰을 받지 않아야 함
        // assertEq(remoteChainMToken.balanceOf(user2), 0, "Blocked user should not receive tokens");
    }
```

**권장 완화 방법:**
1. `msgOfCcSendToken`의 receiver 확인은 소스 체인과 관련이 없으므로 제거하세요:

```diff
function msgOfCcSendToken(
    address sender,
    address receiver,
    uint256 value
) public view returns (bytes memory message) {
    _checkBlocked(sender);
-   _checkBlocked(receiver);
    return abi.encode(TagSendToken, abi.encode(sender, receiver, value));
}
```

2. `ccReceiveToken`에 차단된 주소 확인을 추가하고 차단된 주소로 전송된 토큰을 처리하는 메커니즘을 구현하세요:

```diff
function ccReceiveToken(bytes memory message) internal {
    (address sender, address receiver, uint value) = abi.decode(
        message,
        (address, address, uint)
    );
+   if (isBlocked[receiver]) {
+       // 옵션 1: 복구 주소로 전송
+       _mint(operator, value);
+       emit CCReceiveBlockedAddress(sender, receiver, value);
+   } else {
        _mint(receiver, value);
+   }
    emit CCReceiveToken(sender, receiver, value);
}
```

**Matrixdock:** 인지함.


\clearpage
## Informational


### 상태가 실제로 변경될 때만 이벤트 방출 (Only emit events when state actually changes)

**설명:** 상태가 실제로 변경될 때만 이벤트를 방출하세요. 예를 들어 `MTokenMessager::setAllowedPeer`에서:
```diff
    function setAllowedPeer(
        uint64 chainSelector,
        address messager,
        bool allowed
    ) external onlyOwner {
+      require(chainSelector][messager] != allowed, "No state change");
       allowedPeer[chainSelector][messager] = allowed;
       emit AllowedPeer(chainSelector, messager, allowed);
    }
```

다음에도 영향을 미칩니다:
* `MTokenMessagerV2::setAllowedPeer`

**Matrixdock:** 인지함.


### 명명된 매핑 사용 (Use named mappings)

**설명:** index => value의 목적을 명시적으로 나타내기 위해 명명된 매핑을 사용하세요:
```solidity
MTokenMessager.sol
16:    mapping(uint64 => mapping(address => bool)) public allowedPeer;
//     mapping(uint64 chainSelector => mapping(address messager => bool allowed)) public allowedPeer;

MTokenMessagerV2.sol
28:    mapping(uint64 => mapping(address => bool)) public allowedPeer;
//     mapping(uint64 chainSelector => mapping(address messager => bool allowed)) public allowedPeer;
```

**Matrixdock:** `MTokenMessagerV2`에 대해 커밋 [f3fbe97](https://github.com/Matrixdock-RWA/RWA-Contracts/commit/f3fbe97bd20ad514b76aa422a7dfc1f8a66cd66b#diff-f1dbc2c2c340ac285844595cba6f20040bb8b33c2ae726867955370039433c6aR11-R28)에서 수정됨.

**Cyfrin:** 해결됨.


### 중요한 상태 변경에 대해 누락된 이벤트 방출 (Emit missing events for important state changes)

**설명:** 중요한 상태 변경에 대해 누락된 이벤트를 방출하세요:
* `MTokenMessagerLZ::setLZPaused`

**Matrixdock:** 커밋 [f3fbe97](https://github.com/Matrixdock-RWA/RWA-Contracts/commit/f3fbe97bd20ad514b76aa422a7dfc1f8a66cd66b#diff-591d4d35e5121caa982af913bb68ff10a5555b9462a19650bfd5b844ecedee43R31)에서 수정됨.

**Cyfrin:** 검증됨.


### LayerZero 통합은 일시 중지할 수 있지만 CCIP 통합은 일시 중지할 수 없음 (LayerZero integration can be paused but CCIP integration can't be paused)

**설명:** `MTokenMessagerLZ`는 `bool lzPaused` 스토리지 슬롯을 가지고 있으며 `onlyLZNotPaused` modifier를 사용하여 일시 중지되었을 때 LayerZero 전송/수신을 revert시킵니다.

반면 `MTokenMessager` 및 `MTokenMessagerV2`는 CCIP 전송/수신에 대해 유사한 일시 중지 기능이 없습니다.

이 비대칭이 의도적인지 아니면 CCIP 전송/수신도 유사하게 일시 중지할 수 있어야 하는지 고려하세요.

**Matrixdock:** 인지함.


### LayerZero 수신에 대한 일시 중지를 허용하지 않고, 전송에 대해서만 허용 (Don't allow pausing for LayerZero receive, only send)

**설명:** `MTokenMessagerLZ`는 수신 함수 `_lzReceive`와 두 전송 함수 `lzSendTokenToChain` / `lzSendMintBudgetToChain` 모두에 `onlyLZNotPaused` modifier를 가지고 있습니다.

발신자가 이미 토큰을 소각하고 전송했으므로 이 경우 수신이 revert되는 것을 원하지 않기 때문에 `_lzReceive`에서 `onlyLZNotPaused` modifier를 제거하는 것을 고려하세요.

**Matrixdock:** 커밋 [f3fbe97](https://github.com/Matrixdock-RWA/RWA-Contracts/commit/f3fbe97bd20ad514b76aa422a7dfc1f8a66cd66b#diff-591d4d35e5121caa982af913bb68ff10a5555b9462a19650bfd5b844ecedee43L46)에서 수정됨.

**Cyfrin:** 검증됨.


### `internal` 함수 이름에 일관된 접두사 사용 (Use consistent prefix for `internal` function names)

**설명:** 일부 `internal` 함수는 `_` 접두사 문자를 사용하지만 다른 함수는 사용하지 않습니다. 모든 `internal` 함수 이름에 일관된 접두사로 `_`를 사용하세요:

* `MTokenMessager::sendDataToChain`
* `MTokenMessagerLZ::sendThroughLZ`
* `MTokenMessagerV2::sendDataToChain`

**Matrixdock:** 인지함.


### 명명된 import 사용 (Use named imports)

**설명:** 계약들은 대부분 명명된 import를 사용하지만 이상하게도 일부 import 문은 그렇지 않습니다. 모든 곳에서 명명된 import를 사용하세요:

`MTokenMessager`:
```solidity
import "./interfaces/ICCIPClient.sol";
```

`MTokenMessagerLZ`:
```solidity
import "./MTokenMessagerBase.sol";
import "./interfaces/ICCIPClient.sol";
```

`MTokenMessagerV2`:
```solidity
import "./interfaces/ICCIPClient.sol";
import "./MTokenMessagerLZ.sol";
```

**Matrixdock:** `MTokenMessagerLZ` 및 `MTokenMessagerV2`에 대해 커밋 [f3fbe97](https://github.com/Matrixdock-RWA/RWA-Contracts/commit/f3fbe97bd20ad514b76aa422a7dfc1f8a66cd66b)에서 수정됨.

**Cyfrin:** 검증됨.


### LayerZero 통합에서 사용되며 실제로는 `MToken`을 참조하므로 `MTokenMessagerBase::ccipClient` 이름 변경 고려 (Consider renaming `MTokenMessagerBase::ccipClient` as it is used by LayerZero integration and actually refers to `MToken`)

**설명:** `MTokenMessager::ccipClient` 및 `MTokenMessagerBase::ccipClient`는 LayerZero(`MTokenMessagerLZ`)와 CCIP(`MTokenMessagerV2`) 모두에서 사용됩니다.

그러나 이들은 실제로는 단순히 `MToken` 계약을 참조합니다. 이들을 `ccipClient`라고 부르는 것은 특히 LayerZero 통합을 읽을 때 왜 `ccipClient`를 호출하는지 의아해하게 만들어 혼란스럽습니다.

`MTokenMessager::ccipClient` 및 `MTokenMessagerBase::ccipClient`의 이름을 `mToken`으로 변경하고 `IMToken`에 추가 함수를 더한 다음 `ICCIPClient`를 삭제하는 것을 고려하세요.

**Matrixdock:** `MTokenMessagerBase`에 대해 커밋 [f3fbe97](https://github.com/Matrixdock-RWA/RWA-Contracts/commit/f3fbe97bd20ad514b76aa422a7dfc1f8a66cd66b#diff-dab651c3b43b10cc975bd594f600387ed27d1bffd16250f576c85820925fab9aR6-R9)에서 수정됨.

**Cyfrin:** 검증됨.


### `MTokenMessagerLZ`에서 사용되지 않는 이벤트 `OwnershipTransferRequested` (Unused event `OwnershipTransferRequested` in `MTokenMessagerLZ`)

**설명:** `MTokenMessagerLZ` 계약은 `OwnershipTransferRequested` 이벤트를 선언하지만 계약 내 어디에서도 방출하지 않습니다. 이는 소유권 이전을 위한 타임락 메커니즘을 구현할 계획이 있었으나 완료되지 않았음을 시사합니다. 이벤트가 정의되어 있지만 사용되지 않은 상태로 남아 있어 기능이 불완전함을 나타낼 수 있습니다.

```solidity
18:     event OwnershipTransferRequested(address indexed from, address indexed to);
```

**Matrixdock:** 커밋 [f3fbe97](https://github.com/Matrixdock-RWA/RWA-Contracts/commit/f3fbe97bd20ad514b76aa422a7dfc1f8a66cd66b#diff-591d4d35e5121caa982af913bb68ff10a5555b9462a19650bfd5b844ecedee43L18-R30)에서 제거됨.

**Cyfrin:** 검증됨.



### `MTokenMessager::sendDataToChain`에서 불필요한 코드 중복 (Unnecessary code duplication in `MTokenMessager::sendDataToChain`)

**설명:** `sendDataToChain` 함수는 메시지 객체를 생성하고 수수료를 계산하는데, 이는 이미 `getFeeAndMessage` 함수에 존재하는 로직을 복제하는 것입니다. 이는 코드베이스에 중복성을 생성하여 향후 업데이트 중 불일치를 초래하고 가스 비용을 증가시킬 수 있습니다.

**권장 완화 방법:** 기존의 `getFeeAndMessage` 함수를 사용하도록 `sendDataToChain` 함수를 리팩토링하세요:

```diff
    function sendDataToChain(
        uint64 destinationChainSelector,
        address messageReceiver,
        bytes calldata extraArgs,
        bytes memory data
    ) internal returns (bytes32 messageId) {
-        Client.EVM2AnyMessage memory evm2AnyMessage = Client.EVM2AnyMessage({
-            receiver: abi.encode(messageReceiver),
-            data: data,
-            tokenAmounts: new Client.EVMTokenAmount[](0),
-            extraArgs: extraArgs,
-            feeToken: address(0)
-        });
-        uint256 fee = IRouterClient(getRouter()).getFee(
-            destinationChainSelector,
-            evm2AnyMessage
-        );
+        (uint256 fee, Client.EVM2AnyMessage memory evm2AnyMessage) = getFeeAndMessage(
+            destinationChainSelector,
+            messageReceiver,
+            extraArgs,
+            data
+        );
        if (msg.value < fee) {
            revert InsufficientFee(fee, msg.value);
        }
        messageId = IRouterClient(getRouter()).ccipSend{value: fee}(
            destinationChainSelector,
            evm2AnyMessage
        );
        if (msg.value - fee > 0) {
            payable(msg.sender).sendValue(msg.value - fee);
        }
        return messageId;
    }
```

동일한 문제가 `MTokenMessagerV2::sendDataToChain`에도 존재합니다.

**Matrixdock:** `MTokenMessagerV2`에 대해 커밋 [f3fbe97](https://github.com/Matrixdock-RWA/RWA-Contracts/commit/f3fbe97bd20ad514b76aa422a7dfc1f8a66cd66b#diff-f1dbc2c2c340ac285844595cba6f20040bb8b33c2ae726867955370039433c6aR183)에서 수정됨.

**Cyfrin:** 검증됨.



\clearpage
## Gas Optimization


### 비업그레이드 가능 계약의 생성자에서 한 번만 설정되는 스토리지 슬롯에 `immutable` 사용 (Use `immutable` for storage slots only set once in the constructor of non-upgradeable contracts)

**설명:** 생성자에서 한 번만 설정되는 스토리지 슬롯에 `immutable`을 사용하세요:
* `MTokenMessager::ccipClient`
* `MTokenMessagerBase::ccipClient`

**Matrixdock:** `MTokenMessagerBase`에 대해 커밋 [f3fbe97](https://github.com/Matrixdock-RWA/RWA-Contracts/commit/f3fbe97bd20ad514b76aa422a7dfc1f8a66cd66b#diff-dab651c3b43b10cc975bd594f600387ed27d1bffd16250f576c85820925fab9aR6-L8)에서 수정됨.

**Cyfrin:** 검증됨.


### 특히 `memory` 출력에 대해 명명된 반환 사용 (Use named returns especially for `memory` outputs)

**설명:** 특히 `memory` 출력에 대해 명명된 반환을 사용하세요. 예를 들어 `MTokenMessager::calculateCCSendTokenFeeAndMessage`에서:
```diff
    function calculateCCSendTokenFeeAndMessage(
        uint64 destinationChainSelector,
        address messageReceiver,
        address sender,
        address recipient,
        uint value,
        bytes calldata extraArgs
    )
        public
        view
        returns (uint256 fee, Client.EVM2AnyMessage memory evm2AnyMessage)
    {
        bytes memory data = ccipClient.msgOfCcSendToken(
            sender,
            recipient,
            value
        );
-       return
+       (fee, evm2AnyMessage) =
            getFeeAndMessage(
                destinationChainSelector,
                messageReceiver,
                extraArgs,
                data
            );
    }
```

다음에도 적용됩니다:
* `MTokenMessager::calculateCcSendMintBudgetFeeAndMessage`
* 구식 `return`을 제거할 수 있는 `MTokenMessager::sendDataToChain`
* `MTokenMessagerV2`의 동일한 함수들

**Matrixdock:** `MTokenMessagerV2`에 대해 커밋 [f3fbe97](https://github.com/Matrixdock-RWA/RWA-Contracts/commit/f3fbe97bd20ad514b76aa422a7dfc1f8a66cd66b#diff-f1dbc2c2c340ac285844595cba6f20040bb8b33c2ae726867955370039433c6aR82-R102)에서 수정됨.

**Cyfrin:** 검증됨.


### 초과 수수료 환불 시 금액을 캐시하고 Solady `SafeTransferLib::safeTransferETH` 사용 (Cache amount and use Solady `SafeTransferLib::safeTransferETH` when refunding excess fee)

**설명:** `MTokenMessager::sendDataToChain` 및 `MTokenMessagerV2::sendDataToChain`에서 초과 수수료를 환불할 때 금액을 캐시하고 [Solady](https://github.com/devdacian/solidity-gas-optimization?tab=readme-ov-file#10-use-safetransferlibsafetransfereth-instead-of-solidity-call-effective-035-cheaper) `SafeTransferLib::safeTransferETH`를 사용하세요:
```diff
+ import {SafeTransferLib} from "@solady/utils/SafeTransferLib.sol";

-       if (msg.value - fee > 0) {
-           payable(msg.sender).sendValue(msg.value - fee);
-       }
+       uint256 excessFee = msg.value - fee;
+       if(excessFee > 0) {
+           SafeTransferLib.safeTransferETH(msg.sender, excessFee);
+       }
```

**Matrixdock:** 인지함.

\clearpage
