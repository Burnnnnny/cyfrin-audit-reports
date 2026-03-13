**Lead Auditors**

[0kage](https://twitter.com/0kage_eth)

[Giovanni Di Siena](https://twitter.com/giovannidisiena)

**Assisting Auditors**

[Hans](https://twitter.com/hansfriese)


---

# Findings
## High Risk


### Transceiver 지침이 잘못된 순서로 인코딩된 경우 대기 중인 전송이 소스 체인에 갇힐 수 있음

**설명:** 여러 Transceiver가 있는 경우, 현재 로직은 [`TransceiverStructs::parseTransceiverInstructions`](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/libraries/TransceiverStructs.sol#L326-L359)에서 검증된 대로 발신자가 Transceiver 등록 인덱스의 오름차순으로 Transceiver 지침을 인코딩할 것으로 예상합니다. 일반적인 상황에서 이 로직은 예상대로 작동하며 사용자가 Transceiver 지침을 잘못된 순서로 묶으면 트랜잭션이 실패합니다.

```solidity
/* snip */
for (uint256 i = 0; i < instructionsLength; i++) {
    TransceiverInstruction memory instruction;
    (instruction, offset) = parseTransceiverInstructionUnchecked(encoded, offset);

    uint8 instructionIndex = instruction.index;

    // The instructions passed in have to be strictly increasing in terms of transceiver index
    if (i != 0 && instructionIndex <= lastIndex) {
        revert UnorderedInstructions();
    }
    lastIndex = instructionIndex;

    instructions[instructionIndex] = instruction;
}
/* snip */
```

그러나 지연된 실행을 위해 전송이 처음 대기열에 추가될 때 Transceiver 인덱스 순서에 대한 이 요구 사항은 확인되지 않습니다. 결과적으로 이 경우인 트랜잭션은 사용자가 `NttManager::completeOutboundQueuedTransfer`를 호출하여 대기 중인 전송을 실행할 때 실패합니다.

**영향:** 메시지가 대기열에 추가되면 발신자의 자금이 NTT 관리자에게 전송됩니다. 그러나 Transceiver 인덱스의 순서가 잘못된 경우 이 대기 중인 메시지는 실행될 수 없으며, 결과적으로 사용자 자금은 NTT 관리자에 갇히게 됩니다.

**개념 증명 (Proof of Concept):** 다음 테스트를 실행하십시오:

(코드 생략 - 원본 참조)

**권장 완화 방법:** 전송 금액이 현재 아웃바운드 용량을 초과하는 경우 대기 중인 전송 목록에 메시지를 추가하기 전에 Transceiver 지침이 올바르게 정렬되었는지 확인하십시오.

**Wormhole Foundation:** [PR \#368](https://github.com/wormhole-foundation/example-native-token-transfers/pull/368)에서 수정되었습니다. 취소 로직이 추가됨에 따라 가스를 절약하기 위해 사전 검증을 하지 않기로 결정했습니다.

**Cyfrin:** 확인되었습니다. 대기 중인 전송의 취소가 올바르게 구현된 것으로 보입니다. 그러나 Transceiver 지침의 검증은 완료 단계로 지연된 상태로 유지됩니다.


### 완료 전에 새 Transceiver가 추가되거나 기존 Transceiver가 수정되면 대기 중인 전송이 소스 체인에 갇힐 수 있음

**설명:** 발신자가 현재 아웃바운드 용량을 초과하는 금액을 전송하면 이러한 전송은 [`NttManager::_transferEntrypoint`](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/NttManager/NttManager.sol#L306-L332) 내에서 지연된 실행을 위해 대기열로 보내집니다. 속도 제한 기간은 대기열 등록과 실행 사이의 시간적 지연을 결정하는 불변 변수로 정의되며, 일반적인 속도 제한 기간은 24시간입니다.

```solidity
/* snip */
// now check rate limits
bool isAmountRateLimited = _isOutboundAmountRateLimited(internalAmount);
if (!shouldQueue && isAmountRateLimited) {
    revert NotEnoughCapacity(getCurrentOutboundCapacity(), amount);
}
if (shouldQueue && isAmountRateLimited) {
    // emit an event to notify the user that the transfer is rate limited
    emit OutboundTransferRateLimited(
        msg.sender, sequence, amount, getCurrentOutboundCapacity()
    );

    // queue up and return
    _enqueueOutboundTransfer(
        sequence,
        trimmedAmount,
        recipientChain,
        recipient,
        msg.sender,
        transceiverInstructions
    );

    // refund price quote back to sender
    _refundToSender(msg.value);

    // return the sequence in the queue
    return sequence;
}
/* snip */
```

새 Transceiver가 추가되거나 기존 Transceiver가 NTT 관리자에서 제거되는 경우 속도 제한 기간 내에 대기 중인 모든 전송이 잠재적으로 되돌려질(revert) 수 있습니다. 이는 발신자가 새 구성을 기반으로 주어진 Transceiver에 대한 Transceiver 지침을 올바르게 묶지 않았을 수 있으며, 지침이 [최종적으로 파싱](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/NttManager/NttManager.sol#L357-L358)될 때 누락된 Transceiver 지침으로 인해 배달 가격을 계산하는 동안 잠재적으로 배열 인덱스 범위를 벗어난 예외가 발생할 수 있기 때문입니다. 예를 들어, 처음에 두 개의 Transceiver가 있지만 전송이 속도 제한되는 동안 추가 Transceiver가 추가되는 경우, 아래와 같이 지침 배열은 활성화된 새 Transceiver 수에 해당하는 길이 3으로 선언됩니다. 그러나 전송은 시작될 당시의 구성을 기반으로 두 개의 Transceiver 지침만 인코딩했습니다.

```solidity
function parseTransceiverInstructions(
    bytes memory encoded,
    uint256 numEnabledTransceivers
) public pure returns (TransceiverInstruction[] memory) {
    uint256 offset = 0;
    uint256 instructionsLength;
    (instructionsLength, offset) = encoded.asUint8Unchecked(offset);

    // We allocate an array with the length of the number of enabled transceivers
    // This gives us the flexibility to not have to pass instructions for transceivers that
    // don't need them
    TransceiverInstruction[] memory instructions =
        new TransceiverInstruction[](numEnabledTransceivers);

    uint256 lastIndex = 0;
    for (uint256 i = 0; i < instructionsLength; i++) {
        TransceiverInstruction memory instruction;
        (instruction, offset) = parseTransceiverInstructionUnchecked(encoded, offset);

        uint8 instructionIndex = instruction.index;

        // The instructions passed in have to be strictly increasing in terms of transceiver index
        if (i != 0 && instructionIndex <= lastIndex) {
            revert UnorderedInstructions();
        }
        lastIndex = instructionIndex;

        instructions[instructionIndex] = instruction;
    }

    encoded.checkLength(offset);

    return instructions;
}
```

**영향:** 누락된 Transceiver 지침은 해당 메시지의 총 배달 가격이 계산되는 것을 방지합니다. 이는 현재 Transceiver 목록으로 대기 중인 전송이 실행되는 것을 방지합니다. 결과적으로 기본 발신자 자금은 `NttManager` 계약에 갇히게 됩니다. 전송 중인 증명이 수신 및 실행되기 전에 대상에서 피어 NTT 관리자 계약이 업데이트되는 경우(예: 소스 체인에서 재배포된 후) 유효하지 않은 피어 오류와 함께 되돌려지는 유사한 문제가 발생합니다.

**개념 증명 (Proof of Concept):** 다음 테스트를 실행하십시오:

(코드 생략 - 원본 참조)

**권장 완화 방법:** Transceiver 인덱스가 존재하지 않을 때 배달 가격 견적에 지침을 전달하지 않는 것을 고려하십시오.

**Wormhole Foundation:** [PR \#360](https://github.com/wormhole-foundation/example-native-token-transfers/pull/360)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다. 지침 배열은 이제 활성화된 Transceiver 수가 아니라 등록된 Transceiver 수를 사용하여 할당되므로 Transceiver가 활성화/비활성화되는 경우를 수정합니다. 이는 새 Transceiver가 등록되는 경우도 수정합니다. 루프는 파싱된 지침 길이에 대해 수행되므로 새로 등록된 Transceiver에 대한 지침은 기본값으로 유지됩니다(이 로직은 표준 전송 호출에도 적용됨).

\clearpage


## Medium Risk


### `TrimmedAmount::shift`의 조용한 오버플로로 인해 속도 제한기가 우회될 수 있음

**설명:** [`TrimmedAmount::trim`](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/libraries/TrimmedAmount.sol#L136-L158) 내에는 조정된(scaled) 금액이 최대 `uint64`를 초과하지 않도록 보장하는 명시적인 확인이 있습니다:
```solidity
// NOTE: amt after trimming must fit into uint64 (that is the point of
// trimming, as Solana only supports uint64 for token amts)
if (amountScaled > type(uint64).max) {
    revert AmountTooLarge(amt);
}
```
그러나 [`TrimmedAmount::shift`](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/libraries/TrimmedAmount.sol#L121-L129) 내에는 그러한 확인이 존재하지 않으므로 여기서 `uint64`로 캐스팅할 때 조용한 오버플로가 발생할 가능성이 있습니다:
```solidity
function shift(
    TrimmedAmount memory amount,
    uint8 toDecimals
) internal pure returns (TrimmedAmount memory) {
    uint8 actualToDecimals = minUint8(TRIMMED_DECIMALS, toDecimals);
    return TrimmedAmount(
        uint64(scale(amount.amount, amount.decimals, actualToDecimals)), actualToDecimals
    );
}
```

**영향:** `TrimmedAmount::shift`의 조용한 오버플로는 [`NttManager::_transferEntryPoint`](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/NttManager/NttManager.sol#L300)에서의 사용을 고려할 때 속도 제한기가 우회되는 결과를 초래할 수 있습니다. 이 문제가 발생할 가능성과 높은 영향을 고려하여 **중간** 심각도 발견으로 분류됩니다.

**권장 완화 방법:** `TrimmedAmount::shift`에서 조정된 금액이 최대 `uint64`를 초과하지 않는지 명시적으로 확인하십시오.

**Wormhole Foundation:** [PR \#262](https://github.com/wormhole-foundation/example-native-token-transfers/pull/262)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다. OpenZeppelin `SafeCast` 라이브러리가 이제 `uint64`로 캐스팅할 때 사용됩니다.


### 64개가 등록된 후 `TransceiverRegistry::_setTransceiver`를 호출하여 비활성화된 Transceiver를 다시 활성화할 수 없음

**설명:** [`TransceiverRegistry::_setTransceiver`](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/NttManager/TransceiverRegistry.sol#L112-L153)는 Transceiver 등록을 처리하지만 다른 다운스트림 효과가 있으므로 다시 등록할 수 없으며, 이 함수는 이전에 등록되었지만 현재 비활성화된 Transceiver를 다시 활성화하는 역할도 합니다.
```solidity
function _setTransceiver(address transceiver) internal returns (uint8 index) {
    /* snip */
    if (transceiver == address(0)) {
        revert InvalidTransceiverZeroAddress();
    }

    if (_numTransceivers.registered >= MAX_TRANSCEIVERS) {
        revert TooManyTransceivers();
    }

    if (transceiverInfos[transceiver].registered) {
        transceiverInfos[transceiver].enabled = true;
    } else {
    /* snip */
}
```

이 함수는 전달된 Transceiver 주소가 `address(0)`이거나 등록된 Transceiver 수가 이미 정의된 최대값인 64개에 도달한 경우 되돌려집니다(revert). 총 64개의 등록된 Transceiver가 있고 그중 일부가 이전에 비활성화되었다고 가정할 때, 후자의 검증 위치는 활성화 상태를 `true`로 설정하는 후속 블록에 도달할 수 없으므로 비활성화된 Transceiver가 다시 활성화되는 것을 방지합니다. 결과적으로 최대 수의 Transceiver를 등록한 후에는 비활성화된 Transceiver를 다시 활성화할 수 없으므로 재배포 없이는 이 함수를 호출할 수 없게 됩니다.

**영향:** 일반적인 상황에서 이 최대 등록된 Transceiver 수에 도달해서는 안 되며, 특히 기본 Transceiver가 업그레이드 가능하기 때문에 더욱 그렇습니다. 그러나 운영 가정에 따르면 가능성은 낮지만 이 정의되지 않은 동작은 높은 영향을 미칠 수 있으므로 **중간** 심각도 발견으로 분류됩니다.

**권장 완화 방법:** 최대 Transceiver 검증 위치를 새 Transceiver 등록 처리를 담당하는 `else` 블록 내로 이동하십시오.

**Wormhole Foundation:** [PR \#253](https://github.com/wormhole-foundation/example-native-token-transfers/pull/253)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다. 이전에 등록되었지만(현재는 비활성화됨) Transceiver에 대해 검증을 건너뜁니다.


### 일시 중지된 후 NTT 관리자를 일시 중지 해제할 수 없음

**설명:** `NttManagerState::pause`는 허가된 행위자가 트리거할 수 있는 일시 중지 기능을 노출하지만 해당하는 일시 중지 해제 기능이 없습니다. 따라서 NTT 관리자가 일시 중지되면 계약 업그레이드 없이는 일시 중지를 해제할 수 없습니다.
```solidity
function pause() public onlyOwnerOrPauser {
    _pause();
}
```

**영향:** NTT 관리자를 일시 중지 해제할 수 없으면 상당한 중단이 발생할 수 있으며, 이 문제를 해결하려면 계약 업그레이드 또는 전체 재배포가 필요합니다.

**권장 완화 방법:**
```diff
+ function unpause() public onlyOwnerOrPauser {
+     _unpause();
+ }
```

**Wormhole Foundation:** [PR \#273](https://github.com/wormhole-foundation/example-native-token-transfers/pull/273)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다. 해당하는 일시 중지 해제 함수가 추가되었습니다.


### `WormholeTransceiver` 내의 불변 가스 제한으로 인해 대상 체인에서 실행 실패가 발생할 수 있음

**설명:** Wormhole 관련 Transceiver 구현은 불변 [`gasLimit`](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/Transceiver/WormholeTransceiver/WormholeTransceiverState.sol#L27) 변수를 사용하여 Relayer 배달 가격을 계산합니다. 여기서 기본 가정은 전송에 소비되는 가스가 항상 정적이라는 것입니다. 그러나 가스가 실제 L2에서 소비된 가스와 사실상 L1 가스 가격의 L2 뷰인 L1 calldata 비용의 함수로 계산되는 Arbitrum과 같은 L2 롤업의 경우에는 항상 그런 것은 아닙니다. 가스가 어떻게 추정되는지에 대한 자세한 내용은 [Arbitrum 문서](https://docs.arbitrum.io/inside-arbitrum-nitro#gas-and-fees)를 참조하십시오.

L2 가스가 L1 가스 가격에 의존하는 경우 정적 가스 제한에 의해 계산된 배달 비용이 L2에서 전송을 실행하기에 불충분한 극단적인 시나리오가 발생할 수 있습니다.

**영향:** 불변 가스 제한은 전송을 실행하는 데 필요한 L2 가스에 대해 매우 오래된 뷰를 제공할 수 있습니다. 극단적인 시나리오에서 이러한 오래된 가스 추정치는 대상 체인에서 메시지를 실행하기에 불충분할 수 있습니다. 이러한 시나리오가 발생하면 오래된 가스 추정치를 가진 모든 대기 중인 메시지가 대상 체인에 갇힐 위험이 있습니다. 가스 제한은 업그레이드를 통해 변경할 수 있지만 이 접근 방식에는 두 가지 문제가 있습니다:
1. 많은 수의 `NttManager` 계약에 걸쳐 대규모 업데이트를 동기화하는 것은 실행하기 어려울 수 있습니다.
2. 가스 제한을 변경해도 대상 체인에서 이미 실행 준비가 된 대기 중인 전송에는 영향을 미치지 않습니다.

**권장 완화 방법:** 가스 제한을 가변적으로 만드는 것을 고려하십시오. 필요한 경우 NTT 관리자는 L1 가스 가격을 추적하고 그에 따라 가스 제한을 변경할 수 있습니다.

**Wormhole Foundation:** 이 실패 사례는 더 높은 가스 제한으로 재전송을 요청하여 처리할 수 있습니다. 현재 로직은 자동 릴레이어에서 사용하는 것과 동일하며 우리는 이 설계의 현재 제한 사항에 동의합니다.

**Cyfrin:** 확인되었습니다.


### 안전하지 않은 Transceiver 업그레이드로 인해 Transceiver 불변성 및 소유권 동기화가 깨질 수 있음

**설명:** Transceiver는 NTT 토큰의 크로스 체인 메시지 처리에 필수적인 업그레이드 가능한 계약입니다. `WormholeTransceiver`는 `Transceiver` 계약의 특정 구현이지만 NTT 관리자는 모든 사용자 정의 구현의 Transceiver와 통합할 수 있습니다.

[`Transceiver::_checkImmutables`](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/Transceiver/Transceiver.sol#L72-L77)는 업그레이드 중에 불변성이 위반되지 않았는지 확인하는 내부 가상 함수입니다. 이 함수의 두 가지 확인 사항은 a) NTT 관리자 주소가 동일하게 유지되고 b) 기본 NTT 토큰 주소가 동일하게 유지되는 것입니다.

그러나 현재 로직은 통합자가 다음 중 하나를 통해 이러한 확인을 우회할 수 있도록 합니다:
1. 위의 확인 없이 `_checkImmutables()` 함수를 재정의합니다.
2. `true` 입력으로 [`Implementation::_setMigratesImmutables`](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/libraries/Implementation.sol#L101-L103)를 호출합니다. 이것은 업그레이드 중에 `_checkImmutables()` 함수 검증을 효과적으로 우회합니다.

Transceiver는 NTT 관리자 소유자 외부의 통합자에 의해 배포된다는 이해를 바탕으로, 통합자와 관련된 높은 신뢰 가정에도 불구하고, NTT 관리자가 잠재적으로 NTT 관리자 불변성을 위반할 수 있는 Transceiver 계약을 조용히 업그레이드할 수 있는 권한을 Transceiver에 위임하는 것은 위험합니다.

이에 대한 한 가지 예는 의도된 소유권 모델과 관련이 있습니다. [`Transceiver::_initialize`](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/Transceiver/Transceiver.sol#L44-L53) 내에서 Transceiver의 소유자는 `NttManager` 계약의 소유자로 설정됩니다.
```solidity
function _initialize() internal virtual override {
    // check if the owner is the deployer of this contract
    if (msg.sender != deployer) {
        revert UnexpectedDeployer(deployer, msg.sender);
    }

    __ReentrancyGuard_init();
    // owner of the transceiver is set to the owner of the nttManager
    __PausedOwnable_init(msg.sender, getNttManagerOwner());
}
```

그러나 `Transceiver::transferTransceiverOwnership`을 통한 이 소유권 이전은 NTT 관리자 자체에서만 허용됩니다:
```solidity
/// @dev transfer the ownership of the transceiver to a new address
/// the nttManager should be able to update transceiver ownership.
function transferTransceiverOwnership(address newOwner) external onlyNttManager {
    _transferOwnership(newOwner);
}
```

[`NttManagerState::transferOwnership`](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/NttManager/NttManagerState.sol#L193-L203)을 호출하여 NTT 관리자의 소유자가 변경되면 모든 Transceiver의 소유자도 함께 변경됩니다:
```solidity
/// @notice Transfer ownership of the Manager contract and all Endpoint contracts to a new owner.
function transferOwnership(address newOwner) public override onlyOwner {
    super.transferOwnership(newOwner);
    // loop through all the registered transceivers and set the new owner of each transceiver to the newOwner
    address[] storage _registeredTransceivers = _getRegisteredTransceiversStorage();
    _checkRegisteredTransceiversInvariants();

    for (uint256 i = 0; i < _registeredTransceivers.length; i++) {
        ITransceiver(_registeredTransceivers[i]).transferTransceiverOwnership(newOwner);
    }
}
```

이 설계는 NTT 관리자의 소유자가 모든 Transceiver에서 동기화되도록 보장하고 승인되지 않은 소유권 변경을 방지하기 위해 액세스 제어되도록 의도되었지만, 공개 `OwnableUpgradeable::transferOwnership` 함수가 재정의되지 않았으므로 Transceiver 소유권은 여전히 직접 이전될 수 있습니다. Transceiver 소유권이 변경되더라도 관리자는 위 함수를 통해 다시 변경할 수 있습니다.

그러나 Transceiver의 새 소유자가 불변성 확인 없이 계약 업그레이드를 수행하면 이 동작이 깨질 수 있습니다. 이런 식으로 그들은 NTT 관리자를 변경하여 올바른 관리자가 예상대로 권한을 갖지 못하게 할 수 있습니다. 결과적으로 `NttManagerState::transferOwnership`은 하나의 Transceiver라도 다른 Transceiver와 동기화되지 않으면 되돌려지며(revert), 이미 등록된 Transceiver를 제거하는 것은 불가능하므로 이 함수는 더 이상 유용하지 않게 됩니다. 대신 각 Transceiver는 수정된 Transceiver가 이전 소유자로 재설정되어 이 함수를 다시 호출할 수 있을 때까지 새 소유자로 수동 업데이트되어야 합니다.

**영향:** 이 문제로 인해 Transceiver 소유자가 오작동해야 할 수도 있지만, Transceiver가 새 NTT 관리자 또는 NTT 관리자 토큰으로 조용히 업그레이드되는 시나리오는 크로스 체인 전송에 문제가 될 수 있으므로 주목해야 합니다.

**개념 증명 (Proof of Concept):** 아래 PoC는 `true` 불리언으로 `_setMigratesImmutables()` 함수를 호출하여 `_checkImmutables()` 불변성 확인을 효과적으로 우회합니다. 결과적으로 `NttManagerState::transferOwnership`에 대한 후속 호출이 되돌려지는 것이 입증되었습니다. 이 테스트는 실행하기 전에 `Upgrades.t.sol`의 계약에 추가되어야 하며, 실제 기능을 반영하려면 `MockWormholeTransceiverContract::transferOwnership`의 되돌리기를 제거해야 합니다.

(코드 생략 - 원본 참조)

**권장 완화 방법:** `Transceiver::_checkImmutables` 및 `Implementation::_setMigratesImmutables`를 Transceiver에 대한 비공개 함수로 만드는 것을 고려하십시오. `_checkImmutables()` 함수를 재정의해야 하는 경우 다음과 같이 `_checkImmutables` 내부에서 호출되는 다른 함수를 노출하는 것을 고려하십시오:

```solidity
function _checkImmutables() private view override {
    assert(this.nttManager() == nttManager);
    assert(this.nttManagerToken() == nttManagerToken);
   _checkAdditionalImmutables();
}

function _checkAdditionalImmutables() private view virtual override {}
```

**Wormhole Foundation:** 관리자는 신뢰할 수 있는 엔터티이며 고의로 자체 업그레이드를 중단하지 않습니다. 관리자는 모든 Transceiver에 대한 소유자를 설정할 수 있는 기능이 있지만 대부분의 NTT 배포는 지원되는 모든 Transceiver에서 동일한 소유자를 공유할 가능성이 높습니다.

**Cyfrin:** 확인되었습니다.


### 동일한 체인에 대해 피어 `NttManager` 계약을 설정하면 사용자 자금 손실이 발생할 수 있음

**설명:** `NttManager::setPeer`의 현재 구현은 소유자가 NTT 관리자를 현재 체인과 동일한 체인 ID에 대한 피어로 설정할 수 있도록 합니다. NTT 관리자 소유자가 실수로(또는 그 반대로) 임의의 주소를 동일한 체인에 대한 피어 `NttManager` 주소로 설정하면, 이 구성을 통해 사용자는 소스 체인과 동일한 대상 체인으로 전송을 시작할 수 있지만 그러한 전송은 대상 체인(소스 체인과 동일)에서 실행되지 않습니다.

**영향:** 사용자에게 잠재적인 자금 손실이 있습니다. 피어가 나중에 제거되더라도 소스 체인에서 이미 전송된 메시지는 실행될 수 없습니다. 해당 메시지에 첨부된 모든 자금은 `NttManager` 계약에 갇히게 됩니다.

**개념 증명 (Proof of Concept):** 다음 테스트 케이스는 트랜잭션이 대상 체인에서 실패함을 보여줍니다.

(코드 생략 - 원본 참조)

**권장 완화 방법:** 크로스 체인 인프라를 사용하여 동일한 체인 내에서 전송하는 것은 의미가 거의 없으며 이런 방식으로 구성하면 사용자 자금 손실로 이어질 수 있습니다. NTT 관리자 소유자가 동일한 체인에 대한 피어를 설정하지 못하도록 하는 것을 고려하십시오.

**Wormhole Foundation:** [PR \#365](https://github.com/wormhole-foundation/example-native-token-transfers/pull/365)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다. 피어로서 동일한 체인을 방지하기 위한 검증이 추가되었습니다.


### 현재 설계에서 가스 환불이 없으면 사용자에게 과도한 요금이 부과되고 메시지 실행을 방해할 수 있는 릴레이어 인센티브 불일치가 발생할 수 있음

**설명:** 가스 처리를 이해하려면 현재 설계의 몇 가지 주요 측면을 강조하는 것이 중요합니다:

1. 대상 체인에서 Transceiver는 메시지를 증명(attest)하거나 메시지를 증명하고 실행할 수 있습니다. 임계값(threshold)까지의 Transceiver는 메시지를 증명하고, 임계값에 도달하게 하는 Transceiver는 메시지를 증명하고 실행합니다.

2. 각 Transceiver는 소스 체인에서 가스 견적을 제시합니다. 가격을 제시할 때 어떤 Transceiver도 대상 체인의 피어가 메시지를 단순히 증명할지 아니면 증명하고 실행할지 알지 못합니다. 즉, 모든 Transceiver는 피어가 메시지를 실행할 것이라고 가정하는 가스 견적을 제시합니다.

위의 두 가지 사실을 바탕으로 다음을 추론할 수 있습니다:

1. 임계값에 아직 도달하지 않은 경우 발신자는 이미 실행된 후 메시지를 증명하는 Transceiver를 포함하여 모든 Transceiver에 대한 배달 수수료를 지불합니다.
2. 발신자는 모든 Transceiver가 대상 체인에서 메시지 실행을 담당하는 시나리오에 대해 비용을 지불합니다. 실제로는 하나의 Transceiver만 실행하고 다른 모든 Transceiver는 증명합니다.
3. 릴레이어가 임계값에 도달하기 전에 Transceiver를 지속적으로 호출하면 릴레이어는 가스 측면에서 소비된 것보다 더 많은 수익을 얻습니다.
4. 위의 요점은 릴레이어가 항상 실행하지 않고 증명하는 쪽이 되도록 장려합니다. 영리한 릴레이어는 오프체인에서 `messageAttestations`를 쿼리하고 `messageAttestations == threshold - 1`인 경우 배달을 건너뛸 수 있습니다. 릴레이어는 임계값이 충족되기 전이나 후에 배달하는 경우 소스 체인에서 청구한 것보다 적은 가스를 소비하기 때문입니다.

**영향:**
1. 환불 메커니즘이 없기 때문에 사용자는 소스 체인에서 구제책 없이 과다 청구됩니다.
2. 릴레이어는 증명 임계값을 충족하는 메시지의 실행을 건너뛰어 메시지 실행을 방해할 수 있습니다. 현재의 경제적 인센티브는 릴레이어가 이 특정 메시지를 건너뛰면 이익을 얻습니다.

**권장 완화 방법:** 표준 릴레이어의 경우 대상 체인의 수신자 주소로 초과 가스를 환불하는 메커니즘을 고려하십시오. 핵심 Wormhole 코드베이스의 `DeliveryProvider:: quoteEvmDeliveryPrice `는 현재 사용되지 않는 `targetChainRefundPerUnitGasUnused` 매개변수를 반환합니다. 이 입력을 사용하여 발신자에게 환불할 수 있는 초과 수수료를 계산하는 것을 고려하십시오. 그렇게 하면 사용자의 비용을 절감할 수 있을 뿐만 아니라 릴레이어에 대한 잘못된 경제적 인센티브도 제거할 수 있습니다.

**Wormhole Foundation:** [PR \#326](https://github.com/wormhole-foundation/example-native-token-transfers/pull/326)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다. Transceiver는 이제 환불 주소를 수신하고 `WormholeTransceiver`에 대한 표준 릴레이는 이제 환불을 발행합니다.


### L2 시퀀서가 다운되면 액세스 제어 함수를 호출할 수 없음

**설명:** [Optimism](https://docs.optimism.io/chain/differences#address-aliasing) 및 [Arbitrum](https://docs.arbitrum.io/arbos/l1-to-l2-messaging#address-aliasing)과 같은 롤업은 강제 트랜잭션 포함 방법을 제공하므로, 시퀀서 다운타임 발생 시에도 적용되는 함수를 호출할 수 있도록 허가된 역할을 보유한 발신자를 확인할 때 액세스 제어 수정자(modifier) 내에서 앨리어싱된 발신자 주소도 [확인](https://solodit.xyz/issues/m-8-operator-is-blocked-when-sequencer-is-down-on-arbitrum-sherlock-none-index-git)하는 것이 중요합니다. 가장 적절한 예는 다음과 같습니다:

* [`PausableOwnable::_checkOwnerOrPauser`](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/libraries/PausableOwnable.sol#L17-L24): [`onlyOwnerOrPauser`](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/libraries/PausableOwnable.sol#L9-L15) 수정자에서 호출되며, 이 수정자는 [`PausableOwnable::transferPauserCapability`](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/libraries/PausableOwnable.sol#L31-L39) 및 더 중요하게는 [`NttManagerState::pause`](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/NttManager/NttManagerState.sol#L188-L191)에 적용됩니다.
* [`OwnableUpgradeable::_checkOwner`](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/libraries/external/OwnableUpgradeable.sol#L83-L90): [`onlyOwner`](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/libraries/external/OwnableUpgradeable.sol#L67-L73) 수정자에서 호출되며, 이 수정자는 [`OwnableUpgradeable::transferOwnership`](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/libraries/external/OwnableUpgradeable.sol#L92-L101), [`NttManagerState::upgrade`](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/NttManager/NttManagerState.sol#L183-L186), [`NttManagerState::transferOwnership`](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/NttManager/NttManagerState.sol#L193-L203), [`NttManagerState::setTransceiver`](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/NttManager/NttManagerState.sol#L205-L228), [`NttManagerState::removeTransceiver`](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/NttManager/NttManagerState.sol#L230-L242), [`NttManagerState::setThreshold`](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/NttManager/NttManagerState.sol#L244-L257), [`NttManagerState::setPeer`](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/NttManager/NttManagerState.sol#L259-L281), [`NttManagerState::setOutboundLimit`](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/NttManager/NttManagerState.sol#L283-L286), [`NttManagerState::setInboundLimit`](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/NttManager/NttManagerState.sol#L288-L291), 및 [`Transceiver::upgrade`](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/Transceiver/Transceiver.sol#L66-L68)에 적용됩니다.
* [`onlyRelayer`](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/Transceiver/WormholeTransceiver/WormholeTransceiverState.sol#L274-L279) 수정자는 [`WormholeTransceiver::receiveWormholeMessages`](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/Transceiver/WormholeTransceiver/WormholeTransceiver.sol#L64-L105)에 적용됩니다.

**영향:** 앨리어싱된 발신자 주소를 고려하지 못하면 중앙화된 L2 시퀀서에 의해 트랜잭션이 일괄 처리되는 체인에서 관리자 또는 기타 허가된 기능을 실행할 수 없습니다. 이 기능은 프로토콜의 긴급 일시 중지 또는 NTT 메시지의 릴레이와 같이 시간에 민감할 수 있으므로, 이 문제는 합리적인 가능성과 함께 높은 영향을 미칠 잠재력이 있습니다.

**개념 증명 (Proof of Concept):** 가능성은 낮지만 가능한 시나리오는 다음과 같습니다:

1. 공격자는 소스 체인의 NTT 관리자 프로토콜에서 익스플로잇을 식별하고 도난당한 자금을 이더리움 메인넷으로 브릿징할 계획입니다. 소스 체인은 중앙화된 시퀀서를 통해 L1 체인에 게시하기 위해 트랜잭션을 일괄 처리하는 L2 롤업이라고 가정합니다.
2. L2 시퀀서가 다운되지만, 트랜잭션은 여전히 L1 체인에 강제 포함을 통해 실행될 수 있습니다. 공격자는 이를 강제했거나 발생할 때까지 기다렸을 수 있습니다.
3. 공격자는 프로토콜을 악용하고 기본 토큰 전송을 시작하여 24시간 속도 제한 기간을 시작합니다. L2에서 아웃바운드 속도 제한이 초과되었지만 이더리움 메인넷에서 인바운드 제한은 초과되지 않았다고 가정합니다.
4. 액세스 제어 함수는 앨리어싱된 발신자 주소를 확인하지 않으므로 호출할 수 없으므로 관리자 트랜잭션(예: 프로토콜 일시 중지)을 L1에서 강제로 포함할 수 없습니다.
5. 속도 제한 기간이 지나고 시퀀서는 여전히 다운 상태입니다. 공격자는 아웃바운드 전송을 완료하고(증명이 이루어졌다고 가정) 이더리움 메인넷에서 메시지를 릴레이합니다.

**권장 완화 방법:** 허가된 소유자/일시 중지자/릴레이어 역할에 대한 발신자 주소 검증은 앨리어싱된 등가물도 고려하여 강제 포함을 통해 액세스 제어 기능이 실행될 수 있도록 해야 합니다. 위에서 설명한 익스플로잇 사례에 대한 또 다른 관련 예방 조치는 영향을 받는 체인의 인바운드 속도 제한을 0으로 줄이는 것입니다. 이는 트랜잭션이 대상에서 성공적으로 실행될 수 있는 한(즉, 동시에 시퀀서 다운타임을 겪고 있는 L2 롤업이 아님) 이 문제를 완화하는 데 효과적일 것입니다.

**Wormhole Foundation:** 몇 시간 이상 지속되는 광범위한 L2 시퀀서 다운타임 이벤트는 없었습니다. 우리는 공격자가 다운타임 이벤트와 일치시키기 위해 취약점을 붙잡고 있을 가능성이 낮다고 생각하며, 다운타임이 몇 시간 이상 지속되지 않는다는 가정하에 영향을 제한하는 속도 제한기가 있습니다.

**Cyfrin:** 확인되었습니다.

\clearpage

## Low Risk


### ERC-7201 네임스페이스 스토리지 위치의 일관성 없는 구현

**설명:** 네임스페이스 스토리지 위치 구현과 관련하여, `libraries/external`의 수정된 OpenZeppelin 라이브러리에서 올바르게 수행되는 것처럼 EIP 사양을 엄격하게 준수하려면 EIP-7201 공식을 따라야 합니다. 다른 곳에서는 사용이 잘못되었습니다.

또한 위치 문자열이 일관되지 않은 인스턴스가 있습니다. 예를 들어 `Pause.pauseRole`은 `Pauser.pauserRole`이어야 합니다.

**영향:** 네임스페이스는 다른 네임스페이스 또는 표준 Solidity 스토리지 레이아웃과의 충돌을 피하기 위해 구현됩니다. EIP-7201에 정의된 공식은 keccak256 충돌 저항 가정하에 임의의 네임스페이스 ID에 대해 이 속성을 보장합니다. 따라서 이러한 보증을 최대한 활용하려면 구현이 일관성 있는지 확인하는 것이 중요합니다.

**권장 완화 방법:** [EIP](https://eips.ethereum.org/EIPS/eip-7201)에 설명된 공식을 사용하여 `src/*`의 다른 모든 인스턴스를 업데이트하십시오:
```solidity
keccak256(abi.encode(uint256(keccak256("example.location")) - 1)) & ~bytes32(uint256(0xff));
```

**Wormhole Foundation:** 우리는 EIP를 준수하지 않지만 스토리지 슬롯은 여전히 충돌 방지가 된다고 믿습니다.

**Cyfrin:** 확인되었습니다.


### Transceiver 일시 중지 기능의 비대칭성

**설명:** 일시 중지 기능은 `Transceiver::_pauseTransceiver`를 통해 노출되지만 해당하는 일시 중지 해제 기능을 노출하는 함수는 없습니다:
```solidity
/// @dev pause the transceiver.
function _pauseTransceiver() internal {
    _pause();
}
```

**영향:** 위 함수는 현재 어디에서도 사용되지 않으므로 즉각적인 문제는 아니지만, Transceiver가 영구적으로 일시 중지될 수 있는 경우를 피하기 위해 해결해야 합니다.

**권장 완화 방법:**
```diff
+ /// @dev unpause the transceiver.
+ function _unpauseTransceiver() internal {
+     _unpause();
+ }
```

**Wormhole Foundation:** [PR \#273](https://github.com/wormhole-foundation/example-native-token-transfers/pull/273)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다. `Transceiver::_pauseTransceiver`가 제거되었습니다.


### 잘못된 Transceiver 페이로드 접두사 정의

**설명:** `WormholeTransceiverState.sol`의 `WH_TRANSCEIVER_PAYLOAD_PREFIX` 상수에 유효하지 않은 ASCII 바이트가 포함되어 있어 인라인 개발자 문서에 작성된 내용과 일치하지 않습니다:
```solidity
/// @dev Prefix for all TransceiverMessage payloads
///      This is 0x99'E''W''H'
/// @notice Magic string (constant value set by messaging provider) that idenfies the payload as an transceiver-emitted payload.
///         Note that this is not a security critical field. It's meant to be used by messaging providers to identify which messages are Transceiver-related.
bytes4 constant WH_TRANSCEIVER_PAYLOAD_PREFIX = 0x9945FF10;
```
올바른 페이로드 접두사는 `0x99455748`이며, 다음 명령을 실행할 때 출력됩니다:
```bash
cast --from-utf8 "EWH"
```

**영향:** 식별 목적으로만 사용되는 유효한 4바이트 16진수 접두사이지만, 잘못된 접두사는 다운스트림 혼란을 야기할 수 있으며 그렇지 않으면 유효한 Transceiver 페이로드가 잘못 접두사로 지정될 수 있습니다.

**권장 완화 방법:** 문서화된 문자열에 해당하는 올바른 접두사를 사용하도록 상수 정의를 업데이트하십시오:
```diff
+ bytes4 constant WH_TRANSCEIVER_PAYLOAD_PREFIX = 0x99455748;
```

**Wormhole Foundation:** 이 접두사를 변경하는 것은 여전히 유효한 `bytes4` 상수이므로 실질적인 영향이 없습니다. 다운스트림 종속성으로 인해 변경하지 않기로 결정했습니다.

**Cyfrin:** 확인되었습니다.


### `WormholeTransceiver` EVM 체인 ID 스토리지를 업데이트할 수 없음

업그레이드 결과로 등록된 EVM 호환 체인이 분기되는 드물지만 가능한 시나리오에서 [`WormholeTransceiverState::setIsWormholeEvmChain`](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/Transceiver/WormholeTransceiver/WormholeTransceiverState.sol#L228-L236)의 기존 구현은 해당 스토리지를 업데이트할 수 없음을 의미합니다. 어떤 이유로든 Optimism과 같은 체인이 EVM에서 멀어지기로 결정한 경우(실제로 일어난 일의 반대, OVM에서 EVM으로 이동), 해당 체인 ID는 이제 비 EVM 체인에 해당합니다. 따라서 이 함수는 아래 정의된 [다른 함수](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/Transceiver/WormholeTransceiver/WormholeTransceiverState.sol#L238-L256)와 유사하게 부울 인수를 취해야 합니다:
```diff
- function setIsWormholeEvmChain(uint16 chainId) external onlyOwner {
+ function setIsWormholeEvmChain(uint16 chainId, bool isEvm) external onlyOwner {
    if (chainId == 0) {
        revert InvalidWormholeChainIdZero();
    }
-     _getWormholeEvmChainIdsStorage()[chainId] = TRUE;
+     _getWormholeEvmChainIdsStorage()[chainId] = toWord(isEvm);

-     emit SetIsWormholeEvmChain(chainId);
+     emit SetIsWormholeEvmChain(chainId, isEvm);
}
```
`SetIsWormholeEvmChain` 이벤트도 이 추가 부울 필드를 취하도록 수정해야 합니다.

**Wormhole Foundation:** [PR \#252](https://github.com/wormhole-foundation/example-native-token-transfers/pull/252)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다. `isEvm` 부울 인수가 추가되어 스토리지를 수정할 수 있습니다.


### Transceiver 추가/제거 시 누락된 임계값 불변성 확인

**설명:** [`NttManagerState::_checkThresholdInvariants`](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/NttManager/NttManagerState.sol#L371-L385)는 임계값 스토리지와 관련된 불변 속성을 확인하기 위한 것이며 초기화, 마이그레이션 및 임계값 스토리지의 액세스 제어 설정 시 직접 호출됩니다. 그러나 이 함수는 현재 관련 상태에 액세스하고 수정하는 [`setTransceiver()`](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/NttManager/NttManagerState.sol#L205-L228) 및 [removeTransceiver()](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/NttManager/NttManagerState.sol#L230-L242) 함수가 실행될 때 호출되지 않습니다.

**영향:** 수정되었음에도 불구하고 여기서 임계값 스토리지에 대한 인라인 불변성 확인이 수행되지 않으므로 Transceiver 추가/제거의 버그가 눈에 띄지 않을 수 있습니다.

**권장 완화 방법:** `NttManagerState::setTransceiver` 및 `NttManagerState::removeTransceiver` 내에서 `NttManagerState::_checkThresholdInvariants`가 호출되도록 하십시오.

**Wormhole Foundation:** [PR \#381](https://github.com/wormhole-foundation/example-native-token-transfers/pull/381)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다. 이제 Transceiver가 추가 및 제거될 때 임계값 불변성이 확인됩니다. 언급했듯이 이 변경으로 인해 관리자가 NTT 관리자에서 모든 Transceiver를 비활성화하는 것을 방지할 수 있습니다. 즉, `numRegisteredTransceivers > 0`일 때 항상 적어도 1개의 활성화된 Transceiver가 있어야 합니다.

\clearpage
## Informational


### 연결되지 않은 초기화(Unchained initializers)는 `PausableOwnable::__PausedOwnable_init`에서 대신 호출되어야 함

현재 구현에서는 즉각적인 문제가 아니지만 연결되지 않은 해당 기능이 아닌 초기화 함수를 직접 사용하는 것은 피해야 합니다. 예를 들어 [`__PausedOwnable_init`](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/libraries/PausableOwnable.sol#L26-L29)은 향후 [잠재적인 중복 초기화](https://docs.openzeppelin.com/contracts/5.x/upgradeable#multiple-inheritance)를 피하기 위해 수정되어야 합니다.

```solidity
function __PausedOwnable_init(address initialPauser, address owner) internal onlyInitializing {
    __Paused_init(initialPauser);
    __Ownable_init(owner);
}
```

**Wormhole Foundation:** 이것은 검토되었으며 현재 계약에 영향을 미칩니다. 라이브러리 변경을 포함하여 향후 모든 계약 변경 사항을 검토할 것입니다.

**Cyfrin:** 확인되었습니다.


### `INTTManagerEvents` 및 `IRateLimiterEvents`에 문서화된 잘못된 topics[0]

**설명:** 인라인 NatSpec 문서는 다음 이벤트에 대해 topics[0]을 잘못 지정합니다:

1. [`INTTManagerEvents::TransceiverAdded`](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/interfaces/INttManagerEvents.sol#L51-L52) –
문서화됨: `0xc6289e62021fd0421276d06677862d6b328d9764cdd4490ca5ac78b173f25883`;
올바름: `0xf05962b5774c658e85ed80c91a75af9d66d2af2253dda480f90bce78aff5eda5`.
2. [`INTTManagerEvents::TransceiverRemoved`](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/interfaces/INttManagerEvents.sol#L59-L60) –
문서화됨: `0x638e631f34d9501a3ff0295873b29f50d0207b5400bf0e48b9b34719e6b1a39e`;
올바름: `0x697a3853515b88013ad432f29f53d406debc9509ed6d9313dcfe115250fcd18f`.
3. [`IRateLimiterEvents::OutboundTransferRateLimited`](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/interfaces/IRateLimiterEvents.sol#L20-L21) –
문서화됨: `0x754d657d1363ee47d967b415652b739bfe96d5729ccf2f26625dcdbc147db68b`;
올바름: `0xf33512b84e24a49905c26c6991942fc5a9652411769fc1e448f967cdb049f08a`.

**Wormhole Foundation:** 인라인 NatSpec 문서는 오류가 발생하기 쉽습니다. 선택자/토픽에 대한 진실의 원천으로 Foundry를 사용하는 것에 대해 생각하고 있습니다.

**Cyfrin:** 확인되었습니다.


### 함수/이벤트 서명과 일치하지 않는 NatSpec 문서

인라인 NatSpec 문서는 여러 경우에 실제 서명을 반영하지 않습니다:

1. `ITransceiver::sendMessage` [NatSpec](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/interfaces/ITransceiver.sol#L27-L32)에서 `recipientNttManagerAddress`에 대한 정의 누락:
```solidity
/// @dev Send a message to another chain.
/// @param recipientChain The Wormhole chain ID of the recipient.
/// @param instruction An additional Instruction provided by the Transceiver to be
/// executed on the recipient chain.
/// @param nttManagerMessage A message to be sent to the nttManager on the recipient chain.
function sendMessage(
    uint16 recipientChain, //@note wormhole chain id
    TransceiverStructs.TransceiverInstruction memory instruction, //@note same as above
    bytes memory nttManagerMessage,
    bytes32 recipientNttManagerAddress
) external payable;
```

2. `IRateLimiterEvents::OutboundTransferRateLimited` [NatSpec](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/interfaces/IRateLimiterEvents.sol#L19-L25)에서 `sequence`에 대한 정의 누락:
```solidity
/// @notice Emitted when an outbound transfer is rate limited.
/// @dev Topic0
///      0x754d657d1363ee47d967b415652b739bfe96d5729ccf2f26625dcdbc147db68b.
/// @param sender The initial sender of the transfer.
/// @param amount The amount to be transferred.
/// @param currentCapacity The capacity left for transfers within the 24-hour window.
event OutboundTransferRateLimited(
    address indexed sender, uint64 sequence, uint256 amount, uint256 currentCapacity
);
```

**Wormhole Foundation:** 인라인 NatSpec 문서는 오류가 발생하기 쉽습니다. 선택자/토픽에 대한 진실의 원천으로 Foundry를 사용하는 것에 대해 생각하고 있습니다.

**Cyfrin:** 확인되었습니다.


### 사용되지 않는 `PausableUpgradeable::CannotRenounceWhilePaused` 오류는 제거되어야 함

[`PausableUpgradeable::CannotRenounceWhilePaused`](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/libraries/PausableUpgradeable.sol#L50-L53)는 다음과 같이 정의된 사용자 정의 오류입니다:
```solidity
/**
 * @dev Cannot renounce the pauser capability when the contract is in the `PAUSED` state
 */
error CannotRenounceWhilePaused(address account);
```
위의 오류 및 인라인 주석은 계약이 `PAUSED` 상태일 때 일시 중지 기능을 포기할 수 없음을 의미합니다. 그러나 `PausableOwnable::transferPauserCapability`에서는 그러한 확인이 수행되지 않습니다:

```solidity
/**
 * @dev Transfers the ability to pause to a new account (`newPauser`).
 */
function transferPauserCapability(address newPauser) public virtual onlyOwnerOrPauser {
    PauserStorage storage $ = _getPauserStorage();
    address oldPauser = $._pauser;
    $._pauser = newPauser;
    emit PauserTransferred(oldPauser, newPauser);
}
```

명시된 기능이 함수에 구현되지 않은 누락 오류가 아니라 사용되지 않는 사용자 정의 오류라는 점을 감안할 때 정의를 제거하는 것이 좋습니다.

**Wormhole Foundation:** [PR \#244](https://github.com/wormhole-foundation/example-native-token-transfers/pull/244)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다. 사용되지 않는 오류가 제거되었습니다.


### 여러 `staticcall` 성공 부울이 확인되지 않음

주어진 ERC20 토큰에서 `staticcall`을 수행할 때 반환되는 성공 부울은 현재 어떤 인스턴스에 대해서도 확인되지 않습니다. 이는 합법적인 구현이 항상 `IERC20::decimals`를 구현해야 하고 `IERC20::balanceOf`를 호출할 때 되돌려져서는 안 된다는 신뢰 가정이지만, 다른 것을 디코딩할 것으로 예상할 때 되돌리기(revert) 데이터를 잘못 사용할 가능성을 보장하기 위해 이 반환 값을 확인하는 것이 좋습니다.

```solidity
function _getTokenBalanceOf(
    address tokenAddr,
    address accountAddr
) internal view returns (uint256) {
    (, bytes memory queriedBalance) =
        tokenAddr.staticcall(abi.encodeWithSelector(IERC20.balanceOf.selector, accountAddr));
    return abi.decode(queriedBalance, (uint256));
}
```

**Wormhole Foundation:** [PR \#375](https://github.dev/wormhole-foundation/example-native-token-transfers/pull/375)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다. 성공 부울이 이제 확인됩니다.


### Canonical NTT 체인 ID는 Wormhole에서 직접 가져오거나 그에 따라 매핑되어야 함

현재 불변 `chainId` 변수는 배포자가 `NttManager` [생성자](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/NttManager/NttManager.sol#L26-L32)에 전달한 값으로 할당됩니다:

```solidity
constructor(
    address _token,
    Mode _mode,
    uint16 _chainId,
    uint64 _rateLimitDuration
) NttManagerState(_token, _mode, _chainId, _rateLimitDuration) {}
```

그러나 제공된 체인 식별자가 예상한 것과 같은지 확인하기 위한 검증은 수행되지 않습니다. 이것은 Wormhole 체인 ID로 의도되었으므로 Wormhole 계약에서 반환된 값에 대해 검증되어야 합니다. 그렇지 않고 체인 식별자가 Wormhole 체인 식별자와 일치하지 않는 경우(ID가 반드시 Wormhole 체인 ID일 필요는 없다는 이해를 바탕으로), VAA에 포함된 방출기 체인과 다르기 때문에 이 상태는 Wormhole에 [게시된](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/Transceiver/WormholeTransceiver/WormholeTransceiverState.sol#L80) `WormholeTransceiver` 페이로드에 포함되어야 합니다.

Wormhole 체인 ID를 정식(canonical) 체인 ID로 만들기로 결정한 경우, 다른 Transceiver 구현은 해당 로직 내에서 해당 체인 ID 표현에 매핑해야 합니다. 따라서 위에서 설명한 것처럼 생성자에서 임의의 값을 취하는 대신 Wormhole 주소를 NTT 관리자의 생성자에 전달하여 Wormhole Core Bridge에서 직접 체인 ID를 쿼리해야 합니다.

**Wormhole Foundation:** 우리는 덜 독선적으로 만들기 위해 코어 브릿지에서 직접 체인 ID를 가져오지 않기로 결정했습니다. 이것은 문서화될 것이며 체인 ID 매핑은 필요할 때 구현할 수 있습니다.

**Cyfrin:** 확인되었습니다.

\clearpage
## Gas Optimization


### 프록시 스토리지 슬롯의 `bytes32`에서 `uint256`으로의 불필요한 유형 캐스트

프록시 스토리지 슬롯이 `bytes32` 유형에서 `uint256`으로 캐스트되는 여러 인스턴스가 있습니다. 그러나 EVM은 32바이트 워드를 사용하므로 후속 인라인 어셈블리 할당을 고려할 때 이러한 유형은 서로 교환 가능하므로 이는 불필요합니다. 한 가지 예는 [`Governance::_getConsumedGovernanceActionsStorage`](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/wormhole/Governance.sol#L34-L43)에서 찾을 수 있습니다:

```solidity
function _getConsumedGovernanceActionsStorage()
    private
    pure
    returns (mapping(bytes32 => bool) storage $)
{
    uint256 slot = uint256(CONSUMED_GOVERNANCE_ACTIONS_SLOT);
    assembly ("memory-safe") {
        $.slot := slot
    }
}
```

가스를 절약하기 위해 [이 유형 캐스트 제거](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/789ba4f167cc94088e305d78e4ae6f3c1ec2e6f1/contracts/utils/PausableUpgradeable.sol#L25-L31)를 고려하십시오.

**Wormhole Foundation:** 이를 수행하려면 상수를 미리 계산해야 합니다. 그렇지 않으면 컴파일러 오류가 발생합니다.

**Cyfrin:** 확인되었습니다.


### `Governance::encodeGeneralPurposeGovernanceMessage`의 불필요한 스택 변수

[`Governance::encodeGeneralPurposeGovernanceMessage`](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/wormhole/Governance.sol#L126-L144)는 현재 스택 변수 `callDataLength`를 할당합니다. 그러나 (이미 확인된) 다운캐스트는 후속 패킹 인코딩 내에서 직접 수행될 수 있으므로 이는 불필요합니다.

```solidity
function encodeGeneralPurposeGovernanceMessage(GeneralPurposeGovernanceMessage memory m)
    public
    pure
    returns (bytes memory encoded)
{
    if (m.callData.length > type(uint16).max) {
        revert PayloadTooLong(m.callData.length);
    }
    uint16 callDataLength = uint16(m.callData.length);
    return abi.encodePacked(
        MODULE,
        m.action,
        m.chain,
        m.governanceContract,
        m.governedContract,
        callDataLength,
        m.callData
    );
}
```

가스를 절약하기 위해 이 스택 변수를 제거하는 것을 고려하십시오.

**Wormhole Foundation:** 이것은 뷰(view)로 사용되며 내부적으로 호출되지 않으므로 가스는 문제가 되지 않습니다.

**Cyfrin:** 확인되었습니다.


### `TrimmedAmount` 구조체를 비트 패킹된 `uint72` 사용자 정의 유형으로 최적화

기존 [`TrimmedAmount`](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/libraries/TrimmedAmount.sol#L5-L8) 추상화는 구조체 할당으로 인해 [런타임 가스 오버헤드](https://soliditylang.org/blog/2021/09/27/user-defined-value-types/)가 있습니다:
```solidity
struct TrimmedAmount {
    uint64 amount;
    uint8 decimals;
}
```

이는 대신 토큰 금액과 소수점 자릿수의 비트 패킹된 `uint72` 표현인 사용자 정의 유형을 구현하여 완화할 수 있습니다. `uint72` 유형을 사용하면 기존 9바이트 너비가 유지되어 `RateLimitParams` 구조체와 같이 스토리지의 다른 곳에서 꽉 찬 패킹(tight packing)이 가능합니다:
```solidity
struct RateLimitParams {
        TrimmedAmount limit;
        TrimmedAmount currentCapacity;
        uint64 lastTxTimestamp;
    }
```
따라서 이 정의를 단일 단어(word) 내에 포함시킵니다.

상당한 리팩토링 노력이 필요하지만 사용자 정의 값 유형은 트리밍된 금액을 스택에 할당하고 산술 연산자를 오버로드할 수 있으므로 여기서 유리합니다.

**Wormhole Foundation:** [PR \#248](https://github.com/wormhole-foundation/example-native-token-transfers/pull/248)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다. 일부 주석이 올바르지 않고 추가적인 작은 최적화가 이루어질 수 있지만 그 외에는 올바른 것으로 보입니다.


### `TransceiverRegistry::_checkTransceiversInvariants`의 불변성 확인 반복 최적화

[`TransceiverRegistry::_checkTransceiversInvariants`](https://github.com/wormhole-foundation/example-native-token-transfers/blob/f4e2277b358349dbfb8a654d19a925628d48a8af/evm/src/NttManager/TransceiverRegistry.sol#L211-L234)의 모든 활성화된 Transceiver에 대한 첫 번째 루프는 여러 번의 반복을 절약하고 가스를 절약하기 위해 바로 아래 중첩 루프의 최상위 수준에 포함될 수 있습니다.

```diff
function _checkTransceiversInvariants() internal view {
    _NumTransceivers storage _numTransceivers = _getNumTransceiversStorage();
    address[] storage _enabledTransceivers = _getEnabledTransceiversStorage();

    uint256 numTransceiversEnabled = _numTransceivers.enabled;
    assert(numTransceiversEnabled == _enabledTransceivers.length);

-   for (uint256 i = 0; i < numTransceiversEnabled; i++) {
-       _checkTransceiverInvariants(_enabledTransceivers[i]);
-   }
-
    // invariant: each transceiver is only enabled once
    for (uint256 i = 0; i < numTransceiversEnabled; i++) {
+       _checkTransceiverInvariants(_enabledTransceivers[i]);
        for (uint256 j = i + 1; j < numTransceiversEnabled; j++) {
            assert(_enabledTransceivers[i] != _enabledTransceivers[j]);
        }
    }

    // invariant: numRegisteredTransceivers <= MAX_TRANSCEIVERS
    assert(_numTransceivers.registered <= MAX_TRANSCEIVERS);
}
```

**Wormhole Foundation:** 관리자 전용 메서드이므로 가스 사용량은 중요하지 않습니다.

**Cyfrin:** 확인되었습니다.

\clearpage

