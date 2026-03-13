**Lead Auditors**

[Giovanni Di Siena](https://twitter.com/giovannidisiena)

[0kage](https://twitter.com/0kage_eth)

**Assisting Auditors**

[Hans](https://twitter.com/hansfriese)


---

# Findings
## Medium Risk


### L2 시퀀서가 다운되면 상환(Redemptions)이 차단됨

**설명:** [Optimism](https://docs.optimism.io/chain/differences#address-aliasing) 및 [Arbitrum](https://docs.arbitrum.io/arbos/l1-to-l2-messaging#address-aliasing)과 같은 롤업은 강제 트랜잭션 포함 방법을 제공하므로, 시퀀서 다운타임 발생 시 최대 가동 시간을 허용하기 위해 `mintRecipient`가 지정된 것인지 확인할 때 [`Logic::redeemTokensWithPayload`](https://github.com/wormhole-foundation/wormhole-circle-integration/blob/f7df33b159a71b163b8b5c7e7381c0d8f193da99/evm/src/contracts/CircleIntegration/Logic.sol#L88-L91) 내에서 앨리어싱된 발신자 주소도 [확인](https://solodit.xyz/issues/m-8-operator-is-blocked-when-sequencer-is-down-on-arbitrum-sherlock-none-index-git)하는 것이 중요합니다.

```solidity
// Confirm that the caller is the `mintRecipient` to ensure atomic execution.
require(
    msg.sender.toUniversalAddress() == deposit.mintRecipient, "caller must be mintRecipient"
);
```

**영향:** 앨리어싱된 `mintRecipient` 주소를 고려하지 못하면 중앙화된 L2 시퀀서에 의해 트랜잭션이 일괄 처리되는 대상 CCTP 도메인에서 유효한 VAA의 실행을 방지합니다. 이 VAA는 프로토콜에 대한 긴급한 크로스 체인 유동성 주입과 같은 시간에 민감한 페이로드를 전달할 수 있으므로, 이 문제는 합리적인 가능성과 함께 높은 영향을 미칠 잠재력이 있습니다.

**개념 증명 (Proof of Concept):**
1. 프로토콜 X가 CCTP 도메인 A에서 CCTP 도메인 B로 10,000 USDC를 전송하려고 시도합니다.
2. CCTP 도메인 B는 중앙화된 시퀀서를 통해 L1 체인에 게시하기 위해 트랜잭션을 일괄 처리하는 L2 롤업입니다.
3. L2 시퀀서가 다운되지만, 트랜잭션은 여전히 L1 체인에 강제 포함을 통해 실행될 수 있습니다.
4. 프로토콜 X는 관련 기능을 구현하고 강제 포함을 통해 10,000 USDC 상환을 시도합니다.
5. Wormhole CCTP 통합은 `mintRecipient`를 검증할 때 계약의 앨리어싱된 주소를 고려하지 않으므로 상환이 실패합니다.
6. 이 유동성의 크로스 체인 전송은 시퀀서가 다운된 동안 차단된 상태로 유지됩니다.

**권장 완화 방법:** 강제 포함을 통해 [`Logic::redeemTokensWithPayload`](https://github.com/wormhole-foundation/wormhole-circle-integration/blob/f7df33b159a71b163b8b5c7e7381c0d8f193da99/evm/src/contracts/CircleIntegration/Logic.sol#L88-L91)가 호출될 때 최대 가동 시간을 허용하기 위해 `mintRecipient`에 대한 발신자 주소 검증은 앨리어싱된 `mintRecipient` 주소도 고려해야 합니다.

**Wormhole Foundation:** CCTP가 [이 앨리어싱을 다루지 않기](https://github.com/circlefin/evm-cctp-contracts/blob/adb2a382b09ea574f4d18d8af5b6706e8ed9b8f2/src/MessageTransmitter.sol#L270-L277) 때문에, 우리도 그래야 한다고 강력하게 느끼지 않습니다.

**Cyfrin:** 확인되었습니다.


### CCTP 메시지가 전송 중일 때 `mintRecipient`를 Circle 블랙리스트에 악의적으로 강제 추가하여 자금 손실 발생

**설명:** 유효한 CCTP 메시지가 전송 중인 동안 악의적인 행위자의 행동으로 인해 `mintRecipient`가 대상 도메인에서 상환을 실행할 수 없는 시나리오가 식별되었습니다. `mintRecipient`를 올바르게 구성하는 것은 표면적으로 사용자의 책임이지만, 공격자가 최근 공격에서 훔친 자금(외부 프로토콜에 입금되었다가 인출된 자금)이나 TORN과 같은 OFAC 제재 토큰으로 `mintRecipient` 주소를 더스팅(dusting)하여 메시지가 전송 중인 동안 해당 주소가 대상 도메인에서 Circle에 의해 블랙리스트에 오르도록 강제하여 원래 발신자와 의도한 대상 수신자 모두 토큰에 접근할 수 없게 만드는 경우를 합리적으로 가정할 수 있습니다.

현재 설계에서는 VAA의 멀티캐스트 특성으로 인해 주어진 예금에 대해 `mintRecipient`를 업데이트할 수 없습니다. CCTP는 원래 소스 호출자가 주어진 메시지 및 해당 증명에 대한 대상 호출자를 업데이트할 수 있는 [`MessageTransmitter::replaceMessage`](https://github.com/circlefin/evm-cctp-contracts/blob/1662356f9e60bb3f18cb6d09f95f628f0cc3637f/src/MessageTransmitter.sol#L129-L181)를 노출합니다. 그러나 Wormhole CCTP 통합은 현재 이 함수에 대한 접근을 제공하지 않으며 VAA의 대상 `mintRecipient`에 대한 업데이트를 허용하는 유사한 자체 기능도 없습니다. 잠재적으로 영향을 받는 VAA를 업데이트된 `mintRecipient`를 지정하는 새 VAA로 교체할 방법이 없으면, 이는 대상 도메인에서 토큰을 받는 `mintRecipient`에 대한 영구적인 서비스 거부를 초래할 수 있습니다. 소스 USDC/EURC는 소각되지만, 합법적인 수신자가 대상 도메인에서 자금을 발행할 수 있을 가능성은 매우 낮으며, 일단 토큰이 소각되면 소스 도메인에서 복구할 경로가 없습니다.

이러한 유형의 시나리오는 주로 악의적인 행위자가 소스 호출자가 성공할 것으로 예상하는 크로스 체인 자금 이체를 의도적으로 방해하려고 시도하는 경우에 발생할 가능성이 높습니다. 합리적인 행위자는 알려진 블랙리스트 주소로 크로스 체인 이체를 시도하지 않을 것입니다. 특히 의도한 수신자가 널리 사용되는 프로토콜이 아닌 독립적인 EOA인 경우 더욱 그렇습니다(널리 사용되는 프로토콜은 알려진 공격자로부터 자금을 받더라도 제재에서 면제되는 경향이 있음). 이 경우 [`Logic::redeemTokensWithPayload`](https://github.com/wormhole-foundation/wormhole-circle-integration/blob/f7df33b159a71b163b8b5c7e7381c0d8f193da99/evm/src/contracts/CircleIntegration/Logic.sol#L61-L108)에 대한 대상 호출은 CCTP 계약이 토큰 발행을 시도할 때 실패하며, `mintRecipient` 주소가 어떻게든 Circle 블랙리스트에서 해제되는 경우에만 재시도할 수 있습니다. [그 메커니즘](https://www.circle.com/hubfs/Blog%20Posts/Circle%20Stablecoin%20Access%20Denial%20Policy_pdf.pdf)은 명확하지 않습니다. 또한 법 집행 기관이 전체 프로토콜 X를 대상 도메인 Y의 발행 수신자로 블랙리스트에 올리도록 요청하여 무고한 사용자도 브릿징된 자금에 대한 접근 권한을 잃게 될 가능성도 있습니다.

메시지 교체 기능을 제한하는 동기는 이 엣지 케이스를 처리하는 데 따른 추가적인 복잡성과 추가적인 공격 표면을 감안할 때 원래 메시지의 VAA가 교체된 CCTP 증명으로 상환될 수 없도록 보장해야 하기 때문인 것으로 이해됩니다. Circle 블랙리스트 정책이 이 경우에 어떻게 적용될지 완전히 명확하지 않으므로, 관련 맥락을 가진 사람이 이 비용/편익 분석을 기반으로 결정을 내리는 데 도움을 주는 것이 가장 좋습니다. 피해자가 해결 경로 없이 블랙리스트에 강제로 포함될 수 있다면 이는 분명히 이상적이지 않습니다. 결국 이 문제를 해결할 수 있더라도 일부 리밸런싱/청산 기능을 수행해야 할 수 있는 크로스 체인 작업의 맥락에서 볼 때 영향은 시간에 민감할 수 있으며, 충분한 동기를 가진 공격자는 잠재적으로 대상 도메인에서 후속 발행 시도를 반복적으로 프런트 런(front-run)할 수 있습니다. 메시지가 더 이상 전송 중이 아니고 단순히 대상에서 실행 준비가 된 후에는 블랙리스트가 그렇게 빨리 업데이트될 가능성이 낮다고 가정하므로 이 마지막 요점이 실제로 발생할 가능성이 얼마나 되는지는 완전히 명확하지 않습니다. 어쨌든 메시지 교체를 허용하면 적지 않은 복잡성이 추가되며 앞서 확인된 바와 같이 공격 표면이 실제로 증가한다는 점에는 동의합니다. 따라서 블랙리스트가 어떻게 작동하도록 의도되었는지에 따라 메시지 교체를 허용할 가치가 있을 수 있지만, 이 문제를 해결할 가치가 있는지 확실하게 말할 수는 없습니다.

**영향:** 대상 도메인에서 주어진 VAA를 실행할 수 있는 주소는 단 하나뿐입니다. 그러나 이 `mintReceipient`가 Circle 블랙리스트에 악의적으로 추가되어 영구적으로 상환을 수행할 수 없는 시나리오가 존재합니다. 이 경우 합리적인 가능성과 함께 실질적인 자금 손실이 발생합니다.

**개념 증명 (Proof of Concept):**
1. Alice는 CCTP 도메인 A에서 10,000 USDC를 소각하여 CCTP 도메인 B의 자신의 EOA로 전송합니다.
2. 이 CCTP 메시지가 전송 중인 동안 공격자는 최근 공격에서 얻은 적지 않은 양의 USDC를 프로토콜 X에서 CCTP 도메인 B의 Alice EOA로 인출합니다.
3. 법 집행 기관은 훔친 자금을 보유하고 있는 Alice의 EOA를 블랙리스트에 올리도록 Circle에 통보합니다.
4. Alice는 CCTP 도메인 B에서 10,000 USDC를 상환하려고 시도하지만, 그녀의 EOA가 이제 USDC 계약에서 블랙리스트에 올라 있으므로 발행이 실패합니다.
5. 증명된 CCTP 메시지가 포함된 VAA는 USDC 발행이 되돌려지지 않고는 실행될 수 없으므로 10,000 USDC는 소각된 상태로 유지되며 대상 도메인에서 발행할 수 없습니다.

**권장 완화 방법:** 주어진 CCTP 메시지 및 해당 증명에 대해 VAA가 대상 도메인에서 아직 소비되지 않은 경우 새 VAA로 교체할 수 있도록 허용하는 것을 고려하십시오. 대안으로, 악의적인 블랙리스트로 인해 대상 도메인에서 아직 소비되지 않은 VAA에 의해 소각된 USDC를 복구하기 위한 추가 거버넌스 작업을 추가하는 것을 고려하십시오.

**Wormhole Foundation:** CCTP에는 메시지 교체 기능이 있지만 원래 메시지 수신자를 [변경할 수 없기](https://github.com/circlefin/evm-cctp-contracts/blob/adb2a382b09ea574f4d18d8af5b6706e8ed9b8f2/src/MessageTransmitter.sol#L170-L175) 때문에 이와 동일한 문제가 발생합니다.

**Cyfrin:** 확인되었습니다.

\clearpage
## Low Risk


### `BytesParsing::sliceUnchecked`에서 잠재적으로 위험한 범위를 벗어난 메모리 접근

**설명:** [`BytesParsing::sliceUnchecked`](https://github.com/wormhole-foundation/wormhole-circle-integration/blob/f7df33b159a71b163b8b5c7e7381c0d8f193da99/evm/src/libraries/BytesParsing.sol#L16-L57)는 현재 슬라이스 길이가 0인 퇴화된(degenerate) 경우에 대해 [일찍 종료(bail early)](https://github.com/wormhole-foundation/wormhole-circle-integration/blob/f7df33b159a71b163b8b5c7e7381c0d8f193da99/evm/src/libraries/BytesParsing.sol#L21-L24)합니다. 그러나 인코딩된 바이트 매개변수 `encoded` 자체의 길이에 대한 검증은 없습니다. `encoded`의 길이가 슬라이스 `length`보다 작으면 범위를 벗어난 메모리에 접근할 수 있습니다.

```solidity
function sliceUnchecked(bytes memory encoded, uint256 offset, uint256 length)
    internal
    pure
    returns (bytes memory ret, uint256 nextOffset)
{
    //bail early for degenerate case
    if (length == 0) {
        return (new bytes(0), offset);
    }

    assembly ("memory-safe") {
        nextOffset := add(offset, length)
        ret := mload(freeMemoryPtr)

        /* snip: inline dev comments */

        let shift := and(length, 31) //equivalent to `mod(length, 32)` but 2 gas cheaper
        if iszero(shift) { shift := wordSize }

        let dest := add(ret, shift)
        let end := add(dest, length)
        for { let src := add(add(encoded, shift), offset) } lt(dest, end) {
            src := add(src, wordSize)
            dest := add(dest, wordSize)
        } { mstore(dest, mload(src)) }

        mstore(ret, length)
        //When compiling with --via-ir then normally allocated memory (i.e. via new) will have 32 byte
        //  memory alignment and so we enforce the same memory alignment here.
        mstore(freeMemoryPtr, and(add(dest, 31), not(31)))
    }
}
```

`for` 루프는 메모리의 `encoded` 오프셋에서 시작하여 제공된 `length`에 따른 길이와 동반된 `shift` 계산을 고려하고 `dest`가 `end`보다 작은 동안 실행이 계속되므로, 더 큰 `length` 값을 전달하기만 하면 범위를 벗어난 추가 워드를 계속 로드할 수 있습니다. 따라서 원래 바이트의 길이와 관계없이 출력 슬라이스는 항상 `length` 매개변수로 정의된 크기를 갖습니다.

이는 이 함수의 확인되지 않은(unchecked) 특성과 `nextOffset` 반환 값을 인코딩된 바이트의 길이와 비교하여 검증을 수행하는 동반된 확인된(checked) 버전으로 인해 알려진 동작이라는 점은 이해됩니다.

```solidity
function slice(bytes memory encoded, uint256 offset, uint256 length)
    internal
    pure
    returns (bytes memory ret, uint256 nextOffset)
{
    (ret, nextOffset) = sliceUnchecked(encoded, offset, length);
    checkBound(nextOffset, encoded.length);
}
```

이 검토의 제약 내에서 악의적인 calldata가 이 동작을 사용하여 성공적인 공격을 시작할 수 있는 유효한 시나리오를 식별하는 것은 불가능했습니다. 그러나 이것이 이 라이브러리 함수의 사용에 버그가 없다는 보장은 아닙니다. calldata 로드와 관련된 [특정](https://solodit.xyz/issues/m-2-high-risk-checks-can-be-bypassed-with-extra-calldata-padding-sherlock-olympus-on-chain-governance-git) [특이점(quirks)](https://solodit.xyz/issues/opcalldataload-opcalldatacopy-reading-position-out-of-calldata-bounds-spearbit-none-polygon-zkevm-pdf)이 [존재](https://solodit.xyz/issues/h-04-incorrect-implementation-of-access-control-in-mimoproxyexecute-code4rena-mimo-defi-mimo-august-2022-contest-git)하기 때문입니다.

**영향:** 영향은 이 검토 범위 내의 라이브러리 함수 사용 컨텍스트에서는 제한적입니다. 그러나 이 동작이 무기화될 수 없도록 향후 다른 곳에서의 사용도 확인하는 것이 좋습니다. `BytesParsing::sliceUnchecked`는 현재 [`WormholeCctpMessages::_decodeBytes`](https://github.com/wormhole-foundation/wormhole-circle-integration/blob/f7df33b159a71b163b8b5c7e7381c0d8f193da99/evm/src/libraries/WormholeCctpMessages.sol#L227-L235)에서만 사용되며, 이 함수는 [`WormholeCctpMessages::decodeDeposit`](https://github.com/wormhole-foundation/wormhole-circle-integration/blob/f7df33b159a71b163b8b5c7e7381c0d8f193da99/evm/src/libraries/WormholeCctpMessages.sol#L196-L223)에서 호출됩니다. 후자의 함수는 두 곳에서 사용됩니다:
1. [`Logic::decodeDepositWithPayload`](https://github.com/wormhole-foundation/wormhole-circle-integration/blob/f7df33b159a71b163b8b5c7e7381c0d8f193da99/evm/src/contracts/CircleIntegration/Logic.sol#L126-L148): 여기서 인코딩된 바이트를 슬라이스하는 데 문제가 있으면 사용자가 페이로드를 디코딩하는 능력에 영향을 미쳐 상환에 필요한 정보를 올바르게 검색하지 못하게 될 수 있습니다.
2. [`WormholeCctpTokenMessenger::verifyVaaAndMint`](https://github.com/wormhole-foundation/wormhole-circle-integration/blob/f7df33b159a71b163b8b5c7e7381c0d8f193da99/evm/src/contracts/WormholeCctpTokenMessenger.sol#L144-L197)/[`WormholeCctpTokenMessenger::verifyVaaAndMintLegacy`](https://github.com/wormhole-foundation/wormhole-circle-integration/blob/f7df33b159a71b163b8b5c7e7381c0d8f193da99/evm/src/contracts/WormholeCctpTokenMessenger.sol#L199-L253): 이 함수들은 인코딩된 발행 수신자를 위해 토큰을 발행하기 위해 CCTP 및 Wormhole 메시지를 확인하고 조정합니다. 다행히 악의적인 calldata 페이로드의 경우 Wormhole 자체는 [`WormholeCctpTokenMessenger::_parseAndVerifyVaa`](https://github.com/wormhole-foundation/wormhole-circle-integration/blob/f7df33b159a71b163b8b5c7e7381c0d8f193da99/evm/src/contracts/WormholeCctpTokenMessenger.sol#L295-L311)를 통해 [`IWormhole::parseAndVerifyVM`](https://github.com/wormhole-foundation/wormhole/blob/eee4641f55954d2d0db47831688a2e97eb20f7ee/ethereum/contracts/Messages.sol#L15-L20)이 호출될 때 되돌려집니다. 왜냐하면 `uint8`로 [캐스팅](https://github.com/wormhole-foundation/wormhole/blob/main/ethereum/contracts/libraries/external/BytesLib.sol#L309)할 때 [유효한 버전 번호를 검색](https://github.com/wormhole-foundation/wormhole/blob/main/ethereum/contracts/Messages.sol#L150)할 수 없기 때문입니다.

**개념 증명 (Proof of Concept):** Python 구현에 대한 차분 테스트를 위해 다음 git diff를 적용하십시오:

(코드 생략 - 원본 참조)

**권장 완화 방법:** 슬라이스를 구성할 바이트의 길이가 0인 경우 일찍 종료하고, 확인되지 않은 버전의 함수를 사용할 때는 항상 결과 오프셋이 길이에 대해 올바르게 검증되었는지 확인하는 것을 고려하십시오.

**Wormhole Foundation:** [slice 메서드](https://github.com/wormhole-foundation/wormhole-circle-integration/blob/7599cbe984ce17dd9e87c81fb0b6ea12ff1635ba/evm/src/libraries/BytesParsing.sol#L59)가 우리를 위해 이 확인을 수행합니다. 우리는 와이어 형식에 지정된 길이를 제어하고 있으므로 확인되지 않은 변형을 안전하게 사용할 수 있습니다.

**Cyfrin:** 확인되었습니다.


### `Governance::registerEmitterAndDomain`의 불충분한 검증으로 인해 주어진 CCTP 도메인이 여러 외국 체인에 등록될 수 있음

**설명:** [`Governance::registerEmitterAndDomain`](https://github.com/wormhole-foundation/wormhole-circle-integration/blob/f7df33b159a71b163b8b5c7e7381c0d8f193da99/evm/src/contracts/CircleIntegration/Governance.sol#L48-L84)은 주어진 외국 체인에 대한 방출기(emitter) 주소와 해당 CCTP 도메인을 등록하는 데 사용되는 거버넌스 작업입니다. 현재 외국 체인의 등록된 CCTP 도메인이 로컬 체인의 도메인과 같지 않은지 확인하기 위한 검증이 수행되지만, 주어진 CCTP 도메인이 다른 외국 체인에 대해 이미 등록되지 않았는지 확인하는 검사는 없습니다. 기존 외국 체인의 CCTP 도메인이 실수로 새 외국 체인 등록에 사용되는 경우, 기존 CCTP 도메인의 [`getDomainToChain`](https://github.com/wormhole-foundation/wormhole-circle-integration/blob/f7df33b159a71b163b8b5c7e7381c0d8f193da99/evm/src/contracts/CircleIntegration/Governance.sol#L83) 매핑은 가장 최근에 등록된 외국 체인으로 덮어쓰여집니다. 이미 등록된 방출기를 업데이트하는 방법 없이 외국 체인이 다시 등록되는 것을 방지하는 검증을 감안할 때, 이 상태 손상을 수정하는 것은 불가능합니다.

```solidity
function registerEmitterAndDomain(bytes memory encodedVaa) public {
    /* snip: parsing of Governance VAA payload */

    // For now, ensure that we cannot register the same foreign chain again.
    require(registeredEmitters[foreignChain] == 0, "chain already registered");

    /* snip: additional parsing of Governance VAA payload */

    // Set the registeredEmitters state variable.
    registeredEmitters[foreignChain] = foreignAddress;

    // update the chainId to domain (and domain to chainId) mappings
    getChainToDomain()[foreignChain] = cctpDomain;
    getDomainToChain()[cctpDomain] = foreignChain;
}
```

**영향:** 손상된 상태는 공개 뷰 함수에서만 조회되므로 현재 범위에서 이 문제의 영향은 제한적입니다. 그러나 타사 통합자에게 중요한 경우 다운스트림 문제가 발생할 가능성이 있습니다.

**개념 증명 (Proof of Concept):**
1. CCTP 도메인 A는 외국 체인 식별자 X에 대해 등록됩니다.
2. CCTP 도메인 A는 외국 체인 식별자 Y에 대해 다시 등록됩니다.
3. CCTP 도메인 A에 대한 `getDomainToChain` 매핑은 이제 외국 체인 식별자 Y를 가리키는 반면, X와 Y 모두에 대한 `getChainToDomain` 매핑은 이제 CCTP 도메인 A를 가리킵니다.

**권장 완화 방법:** 외국 체인에 대한 CCTP 도메인을 등록할 때 다음 검증을 추가하는 것을 고려하십시오:

```diff
+ require (getDomainToChain()[cctpDomain] == 0, "CCTP domain already registered for a different foreign chain");
```

**Wormhole Foundation:** 우리는 거버넌스 메시지가 가디언에 의해 서명되고 온체인에 제출되기 전에 충분히 검증되었다고 생각합니다.

**Cyfrin:** 확인되었습니다.


### 등록된 방출기를 업데이트하기 위한 거버넌스 작업 부족

**설명:** Wormhole CCTP 통합 계약은 현재 주어진 외국 체인에 방출기 주소와 해당 CCTP 도메인을 등록하기 위해 [`Governance::registerEmitterAndDomain`](https://github.com/wormhole-foundation/wormhole-circle-integration/blob/f7df33b159a71b163b8b5c7e7381c0d8f193da99/evm/src/contracts/CircleIntegration/Governance.sol#L48-L84) 함수를 노출합니다. 그러나 이 상태를 업데이트하는 함수는 현재 존재하지 않습니다. 방출기 및 CCTP 도메인을 등록할 때 실수한 경우 통합 계약 전체에 대한 업그레이드를 수행하지 않는 한 되돌릴 수 없습니다. 프로토콜 업그레이드 배포에는 고유한 위험이 따르며 사소한 인적 오류에 대한 필수 수정 사항으로 수행되어서는 안 됩니다. 방출기 주소, 외국 체인 식별자 및 CCTP 도메인을 업데이트하기 위한 별도의 거버넌스 작업을 갖는 것은 잠재적인 인적 오류에 대한 바람직한 선제적 조치입니다.

```solidity
function registerEmitterAndDomain(bytes memory encodedVaa) public {
    /* snip: parsing of Governance VAA payload */

    // Set the registeredEmitters state variable.
    registeredEmitters[foreignChain] = foreignAddress;

    // update the chainId to domain (and domain to chainId) mappings
    getChainToDomain()[foreignChain] = cctpDomain;
    getDomainToChain()[cctpDomain] = foreignChain;
}
```

**영향:** 방출기가 잘못된 외국 체인 식별자 또는 CCTP 도메인으로 등록된 경우 이 문제를 완화하려면 프로토콜 업그레이드가 필요합니다. 따라서 프로토콜 업그레이드 배포와 관련된 위험과 이 문제의 잠재적인 시간 민감성을 고려할 때 낮은 심각도 문제입니다.

**개념 증명 (Proof of Concept):**
1. 거버넌스 VAA가 잘못된 외국 체인 식별자로 방출기를 잘못 등록합니다.
2. 올바른 외국 체인 식별자가 주어진 방출기 주소와 연관될 수 있도록 이 상태를 다시 초기화하려면 이제 거버넌스 업그레이드가 필요합니다.

**권장 완화 방법:** 거버넌스가 등록된 방출기 상태의 문제에 더 쉽게 대응할 수 있도록 `Governance::updateEmitterAndDomain` 함수를 추가하는 것이 좋습니다.

**Wormhole Foundation:** 기존 방출기를 업데이트할 수 있도록 허용하면 관리자 실수와 유사한 영향이 발생합니다. 그러나 업데이트를 허용하는 것이 전체 계약 업그레이드를 조정하는 것보다 실제로 쉽습니다. 하지만 이러한 업데이트를 수행하기 위한 거버넌스 메시지가 순서대로 재생되도록 쉽게 강제할 수 없으므로 이를 변경하지 않을 것입니다.

**Cyfrin:** 확인되었습니다.

\clearpage
## Informational


### `WormholeCctpTokenMessenger::setTokenMessengerApproval`에서 `IERC20::approve` 대신 `SafeERC20::safeIncreaseAllowance` 사용

`SafeERC20` 라이브러리가 `IERC20` 인터페이스에 사용되는 것으로 [선언](https://github.com/wormhole-foundation/wormhole-circle-integration/blob/bbc593d7f4caf2b59bf9de18a870e2df37ed6fd4/evm/src/contracts/WormholeCctpTokenMessenger.sol#L26)되었지만, [`WormholeCctpTokenMessenger::setTokenMessengerApproval`](https://github.com/wormhole-foundation/wormhole-circle-integration/blob/bbc593d7f4caf2b59bf9de18a870e2df37ed6fd4/evm/src/contracts/WormholeCctpTokenMessenger.sol#L92-L98)은 `SafeERC20::safeApprove` 대신 `IERC20::approve`를 직접 사용합니다. `FiatTokenV2_2`의 `IERC20::approve` 구현은 `true` 불리언을 반환하고 그렇지 않으면 되돌려지지만(revert), 일부 토큰은 이 함수가 호출될 때 조용히 실패할 수 있습니다. 따라서 프로토콜이 다른 ERC20 토큰과 작업하려는 경우 이 호출의 반환 값을 확인해야 할 수 있습니다. 또한 OpenZeppelin은 `SafeERC20::safeApprove`(v5에서 사용 중단됨)의 사용을 권장하지 않으며 대신 `safeERC20::safeIncreaseAllowance`의 사용을 권장합니다.

**Wormhole Foundation:** [PR \#52](https://github.com/wormhole-foundation/wormhole-circle-integration/pull/52)에서 수정되었습니다.

**Cyfrin:** 확인됨. `ERC20::approve`의 직접 사용이 `safeERC20::safeIncreaseAllowance`를 대신 사용하도록 수정되었습니다.


### CCTP 도메인 간에 브릿징된 자산의 소수점 자릿수(decimals)가 다를 때 발생할 수 있는 잠재적 회계 오류

대상 CCTP 도메인에 배포된 `FiatTokenV2_2` 계약은 일반적으로 6개의 소수점 자릿수를 갖지만, BNB 스마트 체인과 같은 일부 체인에서는 18의 소수점 값이 사용됩니다. Wormhole CCTP 통합 계약과 핵심 CCTP 계약 자체는 소스/대상 토큰 소수점 자릿수의 차이를 조정하지 않습니다. 이는 이러한 계약이 Wormhole x-자산(이 문제가 충분히 완화된 경우)이 아니라 각 체인의 기본 USDC/EURC와 함께 작동하기 때문에 대상 도메인에서 발행될 금액에 치명적인 문제를 일으킬 수 있습니다.

예를 들어, 두 CCTP 도메인이 모두 지원된다고 가정할 때, USDC가 18 소수점 자릿수를 갖는 BNB 스마트 체인에서 20 토큰(`20e18`로 인코딩됨)을 소각한 다음, USDC가 6 소수점 자릿수를 갖는 대상 체인(예: 이더리움)에서 이 금액을 발행하려고 하면 수신자가 20이 아니라 `20e12` 토큰을 발행하지 않았기 때문에 문제가 발생합니다.

BNB 스마트 체인은 현재 지원되는 도메인 중 하나가 아니며 현재 지원되는 모든 CCTP 도메인은 6 소수점 자릿수의 `FiatTokenV2_2` 계약 버전을 사용하므로 현재는 문제가 되지 않습니다. 비표준 도메인이 크로스 체인 전송을 위해 지원될 예정이라면 토큰 소수점 자릿수의 차이를 올바르게 조정하는 것이 중요합니다.

**Wormhole Foundation:** 지금은 변경할 필요가 없지만 CCTP가 다른 체인을 도입하는 경우 변경해야 합니다. 앞으로 주의 깊게 살펴봐야 할 사항입니다.

**Cyfrin:** 확인되었습니다.


### `Setup`이 불필요하게 OpenZeppelin `Context`를 상속함

`Setup` 계약은 현재 OpenZeppelin `Context`를 상속하지만, 해당 기능 중 어느 것도 로직 내에서 사용되지 않으므로 불필요합니다.

**Wormhole Foundation:** [PR \#52](https://github.com/wormhole-foundation/wormhole-circle-integration/pull/52)에서 수정되었습니다.

**Cyfrin:** 확인되었습니다.


### Wormhole 페이로드를 실행하는 상속 애플리케이션에 대한 잠재적 위험

Wormhole CCTP 계약은 구성(composition)과 상속(inheritance)을 통해 통합할 수 있도록 작성되었습니다. `Logic::transferTokensWithPayload`를 호출할 때 사용자는 대상 체인에서 [VAA에서 파싱](https://github.com/wormhole-foundation/wormhole-circle-integration/blob/bbc593d7f4caf2b59bf9de18a870e2df37ed6fd4/evm/src/contracts/CircleIntegration/Logic.sol#L83)되는 임의의 [Wormhole 페이로드](https://github.com/wormhole-foundation/wormhole-circle-integration/blob/bbc593d7f4caf2b59bf9de18a870e2df37ed6fd4/evm/src/contracts/CircleIntegration/Logic.sol#L34)를 전달할 수 있습니다. 필요한 경우 이 페이로드의 실행은 통합 애플리케이션의 책임인 것으로 이해합니다. 따라서 페이로드 실행의 동작은 테스트되지 않았지만, Wormhole 페이로드는 상속 계약에 유용한 정보를 포함할 수 있으므로 반드시 외부 호출로 실행될 필요는 없습니다.

페이로드가 임의의 외부 호출에 대한 입력으로 사용되는 경우 통합자에게 위험이 있습니다. Wormhole CCTP 계약을 상속하는 애플리케이션의 경우 페이로드 실행은 이러한 계약의 컨텍스트에서 발생하며, 이는 잠재적으로 위험할 수 있습니다. 따라서 페이로드에 대해 충분한 애플리케이션별 검증을 수행하는 것은 통합자의 책임입니다. 이는 명확하게 문서화되어야 합니다.

**Wormhole Foundation:** 기존 기능은 의도한 대로입니다.

**Cyfrin:** 확인되었습니다.


### 시퀀싱 고려 사항을 명확하게 문서화하고 통합자에게 전달해야 함

Wormhole CCTP 통합 계약은 기본적으로 시퀀스 내(in-sequence) 메시지 실행을 강제하지 않습니다. 이는 한 메시지가 후속 메시지를 차단하는 것을 방지하기 위한 설계 선택이며, 대신 통합자가 필요한 경우 트랜잭션을 주문할 수 있는 기능을 제공합니다. 통합 계약에 의해 전송된 Wormhole 페이로드를 실행하거나 소비하는 것은 통합 프로토콜의 책임이므로, 메시지 실행의 순서가 올바르게 처리되지 않으면 순서가 맞지 않는 실행으로 인해 높은 심각도와 높은 가능성을 가진 문제가 발생할 수 있습니다. Wormhole VAA는 주문할 필요가 없으며 효과적으로 멀티캐스트되므로, 이 감사 범위 내의 계약에 관한 한 통합에 영향을 미치지 않습니다.

서로 다른 체인 간에 토큰 전송과 함께 일반 페이로드를 처리할 때 의도된 순서가 손상되면, USDC가 DeFi 전체에 얼마나 깊이 자리 잡고 있는지를 감안할 때 대출이나 파생 상품과 같이 순서나 타이밍에 민감한 작업에 적지 않은 결과를 초래할 수 있습니다. 다음 시나리오를 고려하십시오:
1. Alice는 CCTP 도메인 A에서 CCTP 도메인 B의 Perp X로 1000 USDC를 전송합니다.
2. Alice는 5X 레버리지로 5000 USDC 포지션을 개설하기 위한 페이로드와 함께 Perp X에 100 USDC를 추가로 보냅니다.
3. Alice의 메시지는 CCTP 도메인 B에서 실행됩니다:
    1. 첫 번째 메시지가 두 번째 메시지보다 먼저 실행되면 Alice는 1100의 마진을 가지며 거래는 X에서 올바르게 생성됩니다.
    2. 두 번째 메시지가 첫 번째 메시지보다 먼저 실행되면 마진 부족으로 거래를 개설할 수 없습니다. 청산, 경매 등을 고려할 때 순서가 맞지 않는 실행은 수많은 의도하지 않은 결과를 초래할 수 있습니다.

위에서 언급했듯이 발신자는 Wormhole nonce를 지정할 수 있으며 자동 증가되는 Wormhole 시퀀스 번호도 있습니다. 이들은 모두 VAA의 대상 체인에서 수신되므로 순서를 강제하려는 통합자는 소스 도메인에서 Wormhole nonce를 자동 증가시키거나 Wormhole 시퀀스 번호를 사용한 다음 소스 체인, 발신자 주소 및 nonce/sequence를 확인하여 대상 도메인에서 순서를 강제할 수 있습니다. 이는 사용자에게 명확하게 문서화되고 전달되어야 합니다.

**Wormhole Foundation:** 정렬된 트랜잭션이 필요한 통합자는 이를 직접 강제해야 하며 이는 의도된 동작입니다.

**Cyfrin:** 확인되었습니다.


### Wormhole 페이로드에 대한 Calldata 제한을 수정해서는 안 됨

Arbitrum과 Avalanche C-Chain(15M 블록 가스 제한) 간에 작성된 종단 간 포크 테스트에 따르면, `type(uint16).max`의 최대 허용 페이로드 길이를 사용하여 ~2.5M 단위의 가스 사용량이 관찰되었습니다. 이 calldata 제한을 수정하지 않는 것이 중요합니다. 그렇지 않으면 [지나치게 큰 Wormhole 페이로드](https://solodit.xyz/issues/h-2-malicious-user-can-use-an-excessively-large-_toaddress-in-oftcoresendfrom-to-break-layerzero-communication-sherlock-uxd-uxd-protocol-git)로 인한 가스 부족(out-of-gas) 오류로 인해 `mintRecipient`가 대상 도메인에서 상환을 실행할 수 없는 시나리오가 존재할 수 있습니다. 현재 상태에서도 통합자는 `Logic::redeemTokensWithPayload`에 대한 호출을 래핑하는 추가 호출이 이 문제에 취약하지 않도록 주의해야 합니다.

**Wormhole Foundation:** 확인되었습니다.

**Cyfrin:** 확인되었습니다.


### 사용 중단된 Wormhole 가디언 세트가 만료되기 전에 전송 중인 메시지가 실행되지 않을 때 발생하는 일시적인 서비스 거부

**설명:** Wormhole은 거버넌스 VAA를 통해 가디언 세트를 업데이트하기 위해 [`Governance::submitNewGuardianSet`](https://github.com/wormhole-foundation/wormhole/blob/eee4641f55954d2d0db47831688a2e97eb20f7ee/ethereum/contracts/Governance.sol#L76-L112)의 거버넌스 작업을 노출합니다.

```solidity
function submitNewGuardianSet(bytes memory _vm) public {
    ...

    // Trigger a time-based expiry of current guardianSet
    expireGuardianSet(getCurrentGuardianSetIndex());

    // Add the new guardianSet to guardianSets
    storeGuardianSet(upgrade.newGuardianSet, upgrade.newGuardianSetIndex);

    // Makes the new guardianSet effective
    updateGuardianSetIndex(upgrade.newGuardianSetIndex);
}
```

이 함수가 호출되면 [`Setters:: expireGuardianSet`](https://github.com/wormhole-foundation/wormhole/blob/main/ethereum/contracts/Setters.sol#L13-L15)은 현재 가디언 세트가 만료되는 24시간의 기간을 시작합니다.

```solidity
function expireGuardianSet(uint32 index) internal {
    _state.guardianSets[index].expirationTime = uint32(block.timestamp) + 86400;
}
```

따라서 사용 중단된 가디언 세트 인덱스를 사용하는 전송 중인 VAA는 [`Messages::verifyVMInternal`](https://github.com/wormhole-foundation/wormhole/blob/main/ethereum/contracts/Messages.sol)에 존재하는 검증으로 인해 실행에 실패합니다.

```solidity
/// @dev Checks if VM guardian set index matches the current index (unless the current set is expired).
if(vm.guardianSetIndex != getCurrentGuardianSetIndex() && guardianSet.expirationTime < block.timestamp){
    return (false, "guardian set has expired");
}
```

[문서](https://docs.wormhole.com/wormhole/quick-start/tutorials/cctp)에 명시된 것과 달리 Wormhole CCTP 메시지의 자동 릴레이가 없으므로(통합자가 자체 릴레이를 구현하지 않는 한), 이전 가디언 세트 인덱스를 사용하는 전송 중인 메시지가 24시간 만료 기간 내에 대상 도메인의 `mintRecipient`에 의해 실행된다는 보장은 없습니다. 이는 다음과 같은 경우에 발생할 수 있습니다:
1. 통합자 메시지가 Wormhole nonce/sequence 번호 사용으로 인해 차단됨.
2. 대상 도메인에서 CCTP 계약이 일시 중지되어 모든 상환이 되돌려짐(revert).
3. Wormhole CCTP 통합 계약이 강제 포함을 위해 앨리어싱된 주소를 고려하지 않기 때문에 발생하는 L2 시퀀서 다운타임.
4. `mintRecipient`가 악용 후 일시 중지되어 모든 수신 및 발신 전송을 일시적으로 제한하는 계약임.

현재 설계에서는 VAA의 멀티캐스트 특성으로 인해 주어진 예금에 대해 `mintRecipient`를 업데이트할 수 없습니다. CCTP는 원래 소스 호출자가 주어진 메시지 및 해당 증명에 대한 대상 호출자를 업데이트할 수 있는 [`MessageTransmitter::replaceMessage`](https://github.com/circlefin/evm-cctp-contracts/blob/1662356f9e60bb3f18cb6d09f95f628f0cc3637f/src/MessageTransmitter.sol#L129-L181)를 노출합니다. 그러나 Wormhole CCTP 통합은 현재 이 함수에 대한 접근을 제공하지 않으며 VAA의 대상 `mintRecipient`에 대한 업데이트를 허용하는 유사한 자체 기능도 없습니다.

또한 대상 도메인에서 VAA를 실행하도록 허용된 유일한 주소인 `mintRecipient`에게 USDC/EURC의 상환을 강제로 실행하는 방법이 없습니다. 이는 [`Logic::redeemTokensWithPayload`](https://github.com/wormhole-foundation/wormhole-circle-integration/blob/f7df33b159a71b163b8b5c7e7381c0d8f193da99/evm/src/contracts/CircleIntegration/Logic.sol#L61-L108)에서 검증됩니다.

```solidity
// Confirm that the caller is the `mintRecipient` to ensure atomic execution.
require(
    msg.sender.toUniversalAddress() == deposit.mintRecipient, "caller must be mintRecipient"
);
```

만료된 VAA를 업데이트된 가디언 세트가 서명한 새 VAA로 대체하는 프로그래밍 방식이 없으면 소스 USDC/EURC는 소각되지만 만료된 VAA는 실행할 수 없어 대상 도메인에서 토큰을 받는 `mintRecipient`에 대한 서비스 거부(DoS)가 발생합니다. 그러나 Wormhole CCTP 통합은 [Wormhole 백서](https://github.com/wormhole-foundation/wormhole/blob/eee4641f55954d2d0db47831688a2e97eb20f7ee/whitepapers/0003_token_bridge.md#caveats)에 설명된 대로 가디언 세트가 업데이트되는 이러한 유형의 시나리오에 대해 이미 마련된 일부 완화 조치를 상속합니다. 즉, 새 가디언 세트의 서명을 사용하여 실행을 위해 만료된 VAA를 복구하거나 교체할 수 있습니다. 모든 경우에 새로운 VAA 가디언 서명은 이미 방출된 이벤트를 참조하므로 원래 VAA 메타데이터는 그대로 유지되므로 가디언 세트 인덱스 및 관련 서명을 제외한 VAA 페이로드의 내용은 재관찰(re-observation) 시 변경되지 않습니다. 이는 새 VAA가 원래 `mintRecipient`에 의한 대상 도메인에서의 실행을 위해 기존 Circle 증명과 안전하게 페어링될 수 있음을 의미합니다.

**영향:** 대상 도메인에서 주어진 VAA를 실행할 수 있는 주소는 단 하나뿐입니다. 그러나 VAA가 전송 중인 동안 가디언 세트 업데이트 후 24시간을 초과하는 기간 동안 이 `mintReceipient`가 상환을 수행할 수 없는 여러 시나리오가 식별되었습니다. 다행히 Wormhole 거버넌스에는 잘 정의된 해결 경로가 있으므로 영향은 제한적입니다.

**개념 증명 (Proof of Concept):**
1. Alice는 100 USDC를 소각하여 CCTP 도메인 A에서 CCTP 도메인 B의 dApp X로 전송합니다.
2. Wormhole은 가디언 세트를 업데이트하기 위해 거버넌스 VAA를 실행합니다.
3. 24시간이 지나 이전 가디언 세트가 만료됩니다.
4. dApp X는 CCTP 도메인 B에서 100 USDC를 상환하려고 시도하지만 메시지가 만료된 가디언 세트를 사용하여 서명되었기 때문에 VAA 검증이 실패합니다.
5. 만료된 VAA가 새 가디언 세트의 구성원에 의해 재관찰될 때까지 증명된 CCTP 메시지를 실행하여 대상 도메인에서 발행할 수 없으므로 100 USDC는 소각된 상태로 유지됩니다.

**권장 완화 방법:** 더 넓은 DeFi 생태계 내에서 USDC가 얼마나 확고하게 자리 잡고 있는지를 감안할 때 제안된 거버넌스 완화 조치를 대규모로 실행하는 것의 실용성을 신중하게 고려해야 합니다. 일시적인 광범위한 고영향 DoS가 발생할 가능성이 높지만, 지금까지 Wormhole 수명 동안 업데이트가 3번만 있었으므로 가디언 세트 업데이트가 비교적 드물게 발생할 것으로 예상된다는 점을 감안할 때 이는 다소 제한적입니다. 또한 서명된 CCTP 메시지와 새 VAA의 재결합을 처리하고 이러한 고려 사항을 통합자에게 명확하게 전달해야 하는 상세한 VAA 재관찰 시나리오를 위한 도구가 불충분할 수 있습니다.

**Wormhole Foundation:** 이것은 Wormhole 토큰 브릿지가 작동하는 방식과 동일합니다.

**Cyfrin:** 확인되었습니다.


### 잠재적인 자금 손실을 방지하기 위해 `mintRecipient` 주소는 인터페이스 지원을 표시해야 함

대상 `mintRecipient`가 스마트 계약인 경우 `IERC165` 및 다른 Wormhole/CCTP 관련 인터페이스를 구현하여 USDC/EURC 토큰을 전송/승인하는 데 필요한 기능이 있는지 확인해야 합니다. 궁극적으로 토큰 수령을 올바르게 처리하는 것은 통합자의 책임이지만, 이 권장 사항은 `Logic::redeemTokenWithPayload`를 호출한 후 토큰이 되돌릴 수 없게 갇히는 상황을 방지하는 데 도움이 될 것입니다.

**Wormhole Foundation:** 코드가 `CircleIntegration` 로직과 함께 작동하도록 보장하는 것은 통합자의 책임입니다.

**Cyfrin:** 확인되었습니다.

\clearpage

